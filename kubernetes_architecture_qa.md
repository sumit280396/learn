# Kubernetes Architecture — Deep-Dive Interview Guide
### Every component, what it does, how it fails, and how to talk about it

---

## PART 0: The Architecture in One Picture (build this in your head first)

```
                    CONTROL PLANE (the brain — EKS manages this for you)
   ┌──────────────────────────────────────────────────────────────────┐
   │   kubectl ──► API SERVER ◄──── everything talks ONLY to this     │
   │                  │  ▲                                            │
   │                  ▼  │                                            │
   │                ETCD  │        SCHEDULER (assigns pods → nodes)   │
   │           (all state)│        CONTROLLER MANAGER (the fix-it loops)│
   │                      │        CLOUD CONTROLLER (talks to AWS)    │
   └──────────────────────┼───────────────────────────────────────────┘
                          │  (kubelets watch the API server for work)
   ┌──────────────────────┼───────────────────────────────────────────┐
   │  WORKER NODE 1       │            WORKER NODE 2                  │
   │  ┌─────────┐         ▼            ┌─────────┐                    │
   │  │ KUBELET │ ──► CONTAINER        │ KUBELET │ ...                │
   │  └─────────┘     RUNTIME          └─────────┘                    │
   │  KUBE-PROXY      (containerd)     KUBE-PROXY                     │
   │  [pods] [pods]                    [pods] [pods]                  │
   └──────────────────────────────────────────────────────────────────┘
```

**The three rules that explain the whole design:**
1. **Everything goes through the API server.** No component talks to etcd directly except the API server; components don't talk to each other directly — they all watch and write via the API. One front door = one place for auth, validation, and audit.
2. **The hub-and-spoke is pull-based.** The API server doesn't push commands to nodes; kubelets *watch* for pods assigned to them. This makes the system resilient — a briefly disconnected node catches up when it reconnects.
3. **State lives in exactly one place: etcd.** Every other component is stateless and replaceable — kill the scheduler and restart it; it re-reads state and continues. Only etcd is precious.

---

## PART 1: Control Plane Components — One Deep Answer Each

**Q1. What is the API server? Why is it the most important component?**
> "kube-apiserver is the front door and the ONLY component that reads/writes etcd. Every interaction — kubectl, controllers, kubelets, Domino's dispatcher creating workspace pods — is a REST call to it. Per request it does three things in order: **authentication** (who are you — certs, tokens, on EKS your IAM identity via the aws-auth mapping/access entries), **authorization** (may you do this — RBAC), **admission control** (should this object be allowed/modified — webhooks that validate or mutate, e.g., injecting defaults, enforcing policies like 'every pod must have resource limits'). Then validate and persist to etcd. It's stateless and horizontally scaled — EKS runs multiple replicas behind an endpoint. Why it matters most: if the API server is down, nothing NEW can happen — no scheduling, no scaling, no kubectl — though and this is the crucial nuance: **already-running pods keep running**, because kubelets and containers don't need the API server to keep existing workloads alive. The cluster goes 'read-only frozen', not dark."

**Q2. What is etcd? What happens if it's lost?**
> "A distributed, consistent key-value store — the cluster's single source of truth: every object you've ever applied lives there. It uses the **Raft consensus algorithm** across an odd number of replicas (3 or 5): writes commit only when a majority (**quorum** — 2 of 3) agree, which is what makes the state consistent even if a member dies. Lose quorum → the cluster can't accept writes (frozen); lose etcd entirely without backup → the cluster has amnesia — running pods continue, but Kubernetes no longer knows anything about them, and recovery means restoring from an etcd snapshot or rebuilding. Two practical notes: etcd is latency-sensitive (slow disks = slow API = sick cluster — why it needs fast SSDs), and on **EKS, AWS owns etcd entirely** — backups, scaling, quorum — one of the strongest arguments for managed control planes. The state hierarchy to say out loud: *the cluster is cattle, etcd is the herd registry.*"

**Q3. What does the scheduler actually do — in detail?**
> "kube-scheduler watches for pods with no node assigned and runs a two-phase decision per pod: **Filtering** — eliminate nodes that CAN'T host it: insufficient unreserved CPU/memory versus the pod's *requests*, untolerated taints, unsatisfied node selectors/affinity, volume/AZ constraints, port conflicts. **Scoring** — rank surviving nodes: spread across zones, resource balance, affinity preferences, image locality (a node that already has the image starts faster). Highest score wins; the scheduler writes a 'binding' (pod → node) through the API server — that's ALL it does; it never touches the node. Two things worth volunteering: scheduling is based on **requests, not real usage** — a node can be 'full' by requests while idle in reality, which is why request right-sizing is a cost lever; and if filtering eliminates every node, the pod stays **Pending** with the reason in its events — which makes the scheduler's logic the literal checklist for diagnosing Pending pods."

**Q4. What is the controller manager? Explain the reconciliation loop precisely.**
> "kube-controller-manager runs dozens of control loops in one process, each owning an object type and running the same algorithm forever: **watch desired state (API) → observe actual state → compute the diff → act to converge → repeat.** Examples: the ReplicaSet controller counts pods matching the template — fewer than desired, create; more, delete. The node controller watches kubelet heartbeats — silent too long, mark NotReady, then evict pods after the timeout. The job controller creates pods until completions are met. Deployment, endpoint, namespace, PV controllers — same pattern. This loop IS Kubernetes' famous self-healing: nobody 'restarts your crashed pod' — a controller notices actual ≠ desired and closes the gap. It's also why deleting a managed pod is futile as a fix and why the platform keeps working through disruptions without human action. Level-triggered, not edge-triggered — it reconciles against the current state each pass, so missed events don't matter."

**Q5. What is the cloud controller manager?**
> "The bridge between Kubernetes abstractions and the real cloud: when you create a `type: LoadBalancer` service, this is what calls AWS to provision the NLB/ALB; it also handles node lifecycle (deregistering terminated EC2 instances) and cloud routes. It exists as a separate component so Kubernetes core stays cloud-neutral — provider-specific logic is pluggable. On EKS, its work is shared with in-cluster controllers like the **AWS Load Balancer Controller** (for ALBs/ingress). Interview relevance: 'created a LoadBalancer service and no AWS LB appeared' → this layer — its logs, its IAM permissions."

---

## PART 2: Node Components

**Q6. What is the kubelet — and what does it NOT do?**
> "The node agent — the only Kubernetes component that actually makes containers exist. It watches the API server for pods bound to its node and drives them to reality: instruct the runtime to pull images and start containers, mount volumes, run liveness/readiness probes, restart failed containers per policy, and report pod + node status (heartbeats) back. What it does NOT do — worth saying explicitly: it doesn't decide what runs where (scheduler's job) and it doesn't restart pods on OTHER nodes when its node dies (controllers' job). If a kubelet dies: containers on the node keep running (the runtime holds them), but the node stops reporting → NotReady → controllers evict and reschedule the pods elsewhere. The kubelet is also the node-level enforcer: cgroup limits (OOM kills at memory limits), and **eviction under node pressure** — disk or memory pressure makes the kubelet proactively evict pods (lowest QoS class first) to save the node, which is where 'Guaranteed vs Burstable vs BestEffort' QoS becomes operationally real."

**Q7. What is kube-proxy? Explain how a Service actually works underneath.**
> "A Service's ClusterIP is a *virtual* IP — no interface owns it. kube-proxy runs on every node and programs the Linux dataplane (**iptables** rules, or **IPVS** at scale) so that traffic addressed to the ClusterIP is intercepted and DNAT'd to one of the current healthy pod IPs from the Service's Endpoints — connection-level load balancing done by the kernel, not by a proxy process in the data path. It watches Services/Endpoints via the API and keeps the rules current as pods come and go. Failure intuition: kube-proxy broken on a node → pods ON THAT NODE can't reach services properly while everything else looks fine — the 'it only fails from node 7' pattern. And to pre-empt the follow-up: **DNS is separate** — CoreDNS maps service *names* to ClusterIPs; kube-proxy makes the ClusterIP *work*. Name resolution problem vs connection problem — different components."

**Q8. Container runtime, CRI, CNI, CSI — decode the acronyms.**
> "Kubernetes standardized its pluggable edges into three interfaces:
> **CRI (Container Runtime Interface)** — how the kubelet talks to whatever actually runs containers; **containerd** is the standard runtime now (Docker-the-engine was removed as a runtime — 'Docker deprecation' — but Docker-built *images* run unchanged, since images are an open standard; a nice gotcha to articulate).
> **CNI (Container Network Interface)** — pluggable pod networking: assigns each pod its IP and wires connectivity. On EKS the **AWS VPC CNI** gives pods real VPC IPs — great integration, with the famous operational consequence: **pod density is limited by ENI/IP capacity per instance type, and subnets can run out of IPs** → pods stuck ContainerCreating with 'failed to assign an IP.'
> **CSI (Container Storage Interface)** — pluggable storage drivers (EBS CSI, EFS CSI) that provision/attach/mount volumes; no driver installed → PVCs never bind.
> One sentence to sound fluent: *kubelet speaks CRI to run containers, CNI to network them, CSI to give them disks.*"

**Q9. What is CoreDNS and why is it architecture-critical?**
> "The cluster's internal DNS, running as a Deployment: every Service gets a name — `service.namespace.svc.cluster.local` — and every pod's resolv.conf points at CoreDNS. Since ALL service-to-service communication starts with a name lookup, CoreDNS is a hidden single point of failure: if it's down or overwhelmed, *everything* looks broken simultaneously — apps 'can't reach' dependencies that are perfectly healthy — which is why 'sudden cluster-wide connection failures' should trigger 'check CoreDNS' as an early reflex. Operational notes: it scales as a normal Deployment (under-replicated CoreDNS on a big cluster = intermittent resolution timeouts = maddening flaky-error tickets), and the cross-namespace gotcha: short name `mysvc` resolves only within the same namespace — cross-namespace calls need `mysvc.otherns`."

---

## PART 3: Flow Questions (narrating these = sounding senior)

**Q10. "Walk me through EXACTLY what happens from `kubectl apply` to a running, reachable pod — component by component."** (THE architecture question — rehearse this out loud)
> "1. **kubectl** sends the manifest to the **API server**.
> 2. API server **authenticates** me (EKS: IAM), **authorizes** via RBAC, runs **admission** webhooks, validates, and **persists to etcd**. Response: 'created'. Nothing is running yet — only desired state exists.
> 3. The **deployment controller** (in controller-manager) sees the new Deployment via its watch → creates a **ReplicaSet**; the ReplicaSet controller creates **pod objects** — still just etcd records, `nodeName` empty.
> 4. The **scheduler** sees unbound pods → **filters** nodes (resources vs requests, taints, affinity, volumes) → **scores** survivors → writes the **binding** through the API server.
> 5. The chosen node's **kubelet** sees a pod assigned to it → **CRI**: containerd pulls the image and starts containers → **CNI**: pod gets an IP → **CSI**: volumes attach and mount → probes begin.
> 6. **Readiness passes** → the endpoints controller adds the pod IP to the Service's **Endpoints** → **kube-proxy** on every node updates iptables/IPVS so the Service's ClusterIP now balances to it → **CoreDNS** already resolves the service name.
> 7. And from now on, the **controllers reconcile forever** — the pod dying at 3am just re-runs steps 3–6 without a human.
> Bonus close: 'and each step is a distinct failure domain — which is why the pod's status tells you which component to interrogate.'"

**Q11. "What happens when a NODE dies — trace the architecture's response."**
> "T+0: the node's **kubelet stops heartbeating**. T+~40s: the **node controller** marks it NotReady. T+~5min (eviction timeout): the node controller **deletes the pod objects** on it; **ReplicaSet/Deployment controllers** instantly notice actual < desired and create replacement pods; the **scheduler** binds them to healthy nodes; kubelets there start them; **endpoints controller + kube-proxy** remove the dead pod IPs from services (readiness/endpoints actually pull traffic away earlier than the eviction). Meanwhile the **cluster autoscaler** replaces lost capacity if needed, and on EKS the node group spawns a fresh EC2 instance. Total human involvement: zero — which is the reconciliation loop as an availability strategy. Caveats worth adding: StatefulSet pods with EBS volumes recover slower (volume detach/attach across AZ constraints), and if the cluster lacks spare capacity, replacements sit Pending — self-healing needs headroom to heal into."

**Q12. "What happens if each control plane component dies? Go component by component."** (differentiator question)
> "**API server down**: cluster frozen for changes — no kubectl, no scheduling, no controller actions — but **running pods keep serving**; kubelets manage existing workloads locally. Traffic flows; nothing new happens.
> **etcd down/quorum lost**: API server can't read/write → same freeze, and if data is lost, the cluster's memory is gone — the one component whose loss isn't just an outage but potential amnesia.
> **Scheduler down**: everything works EXCEPT new pods pile up Pending. Existing workloads untouched.
> **Controller manager down**: the fix-it loops stop — crashed pods stay unreplaced, node failures unhandled, endpoints go stale. The cluster stops self-healing but keeps running — decay, not collapse.
> **kubelet down (one node)**: that node's pods keep running but unmanaged; node goes NotReady; ~5min later pods rescheduled elsewhere.
> **kube-proxy down (one node)**: service routing breaks *from that node only*.
> **CoreDNS down**: everything LOOKS down — name resolution fails cluster-wide while every workload is healthy.
> The pattern that impresses: **Kubernetes degrades gracefully — data plane (running pods) survives control plane failures**; and on EKS, AWS carries the pager for the top half of this list."

---

## PART 4: Architecture-Level Troubleshooting

**T1. "kubectl is slow / timing out for everyone. Where do you look?"**
> "That's API-server-path degradation, not a workload issue. On EKS: check the EKS console/health and **API server metrics/audit logs in CloudWatch**. Common causes: **client flooding** — a misbehaving controller/operator or CI hammering the API (rate-limit/throttling signatures — 429s — in logs; find the noisy client in audit logs); **etcd pressure** — too many objects, huge LIST calls (e.g., something listing every pod every second), oversized objects (giant ConfigMaps); or webhook drag — a slow/failing **admission webhook** stalls every write (dead webhook with `failurePolicy: Fail` can freeze all deployments — a classic). Mitigation: find and stop the abusive client, fix or bypass the sick webhook; and note the reassuring architecture fact while triaging: running workloads are unaffected."

**T2. "Pods on ONE node can't reach any services; other nodes fine."**
> "Single-node scope screams node-local dataplane: **kube-proxy** on that node (pod running? logs? iptables/IPVS rules present and current?), the **CNI agent** on that node (aws-node pod on EKS — crashed CNI = broken pod networking), conntrack table exhaustion, or that node's security group/network config drifted. Quick isolation: from a pod on the sick node, curl a *pod IP directly* (bypasses services) — works → service plumbing (kube-proxy); fails → pod networking (CNI). Fix is usually restarting the node-local agent or recycling the node — cattle logic — with the RCA asking why (OOM on the agent? version skew after a partial upgrade?)."

**T3. "Pods stuck ContainerCreating (not Pending). What's different and what do you check?"**
> "Past Pending means *scheduled* — a node accepted it — so the failure is in the kubelet's startup pipeline, and `describe pod` events name the stage: **CNI/IP assignment** — 'failed to assign an IP': on EKS, subnet IP exhaustion or the instance's ENI limit reached — the VPC CNI classics; **volume attach/mount** — EBS volume stuck attached to the old node, AZ mismatch, EFS mount targets/security groups blocking NFS; **image pull in progress** — a 15 GB environment image just takes minutes (Domino!) — pull progress visible in events; or **secret/configmap mount missing** — referenced object doesn't exist. Pending = scheduler's problem; ContainerCreating = kubelet's problem — that one sentence shows you know the architecture."

**T4. "Intermittent 'connection refused / name resolution' errors across many apps, but everything shows healthy."**
> "Intermittent + cluster-wide + healthy workloads = shared infrastructure, and the prime suspect is **CoreDNS under-provisioned or degraded**: check its pods' CPU/restarts, error/latency metrics, and whether resolution fails under load (ndots/conntrack amplification are known aggravators). Second suspect: **endpoint churn** — pods flapping in/out of readiness (an overly tight readiness probe under load) makes services intermittently route to dying pods. Third: **conntrack exhaustion** on busy nodes. Approach: correlate error timestamps with CoreDNS metrics and endpoint-change events; the fix ranges from scaling CoreDNS (it's just a Deployment) to fixing the flapping probe. This question tests whether you know that in this architecture, 'everything is a name lookup first' — DNS is load-bearing."

**T5. "After an EKS version upgrade, some things subtly broke. What classes of breakage does the architecture predict?"**
> "Upgrades move the API server first, so: **API deprecations/removals** — manifests using removed API versions fail to apply (the notorious apps/v1beta1-era breakages; audit with pluto/kubent BEFORE upgrading); **version skew** — nodes/kubelets more than the supported skew behind the control plane misbehave — node groups must follow promptly; **add-on compatibility** — CNI/CSI/CoreDNS/kube-proxy versions are matched per K8s version and EKS expects them upgraded in step (the most commonly forgotten step); **webhook/controller compatibility** — operators built against old APIs; and from the Domino admin guide: **IRSA trust policies** if the upgrade was blue-green (new cluster = new OIDC provider). Which is why the upgrade runbook is: deprecation audit → staging rehearsal → control plane → node groups → add-ons → validation suite — under change control."

---

## PART 5: Scenario Questions

**S1. "Is the control plane a single point of failure? How is it made highly available — and what does EKS give you?"**
> "In a self-managed cluster you'd run multiple control plane nodes: stateless components (API server — active-active behind a load balancer; schedulers/controller-managers — leader election, one active, others standby) and **etcd as a 3- or 5-member Raft quorum across zones** — surviving one member/zone loss while maintaining majority. EKS packages exactly that: multi-AZ control plane, managed etcd with backups, an SLA — you consume an endpoint. And the architectural safety net beneath HA: even total control plane loss doesn't stop running pods — the data plane keeps serving while the brain is restored. My conclusion as a platform engineer: control-plane HA is a solved problem I *buy* (EKS) so my engineering time goes to the layers users actually feel — capacity, storage, networking, and the platform on top."

**S2. "We're seeing 'too many pods' scheduling failures on nodes that look half-empty. Explain, architecturally."**
> "Two ceilings besides CPU/memory: **max pods per node** — kubelet's limit, which on EKS with the VPC CNI derives from the instance type's **ENI × IPs-per-ENI** capacity (a small instance may cap at ~17 pods regardless of free CPU), and **allocatable vs capacity** — the kubelet reserves system/kube overhead, so schedulable resources are less than the machine's raw size. So 'half-empty' by CPU can be full by IP math. Fixes: larger instance types, prefix delegation mode on the VPC CNI (raises IP capacity per ENI), or right-sizing many-small-pods workloads. This is exactly the kind of invisible-ceiling issue a Domino platform hits when hundreds of small workspaces pack onto nodes."

**S3. "Explain Kubernetes architecture to a data scientist in 60 seconds." (tests communication — a support-role skill)**
> "'Think of it as a very diligent operations team made of software. You hand it a written request — I need this program, this much memory, always running. A receptionist (API server) records it in the official ledger (etcd). A dispatcher (scheduler) picks which machine has room. A site agent on that machine (kubelet) actually starts your program. And a supervisor (controllers) re-reads the ledger every few seconds and fixes any difference between what's promised and what's real — your program crashes at 3am, it's restarted before anyone notices. When you start a Domino workspace, this is what happens underneath — and when it won't start, I read that chain step by step to find which link broke.' Being able to compress the architecture like this is worth as much in a support role as knowing it deeply."

---

## PART 6: Rapid-Fire (skim before the interview)

- **Three design rules:** everything through the API server; components watch/pull (not pushed); all state in etcd — everything else stateless & replaceable
- **API server** = auth → RBAC → admission → validate → persist; down = frozen, not dark (running pods survive)
- **etcd** = Raft quorum (3/5, majority to write); the only precious component; EKS owns it
- **Scheduler** = filter (can't-host) then score (best-host); places by **requests, not usage**; its checklist = your Pending-pod diagnosis
- **Controller manager** = reconciliation loops: watch → diff → act, forever = self-healing
- **Cloud controller** = LoadBalancer services → real AWS LBs; node lifecycle
- **kubelet** = only component that touches containers; probes, cgroup enforcement, pressure evictions (QoS order); its silence = NotReady → eviction ~5min
- **kube-proxy** = makes ClusterIPs real via iptables/IPVS; per-node — broken = that node only
- **kubelet speaks CRI (containerd) to run, CNI to network, CSI to store**
- **VPC CNI on EKS** = pods get real VPC IPs; ceilings = ENI/IP per instance + subnet exhaustion → ContainerCreating 'failed to assign IP'
- **CoreDNS** = every service call starts with a name lookup → its degradation looks like total outage; it's just a Deployment — scale it
- **Pending = scheduler's problem; ContainerCreating = kubelet's problem** (CNI/CSI/image/mounts)
- **Component-death table:** API=freeze / etcd=freeze(+amnesia risk) / scheduler=Pending pileup / controllers=no self-heal / kubelet=one node evicted / kube-proxy=one node's routing / CoreDNS=everything *looks* down
- **Upgrade breakage classes:** API removals, version skew, add-on mismatch, webhook/operator compat, (blue-green: IRSA OIDC)
- **Two invisible node ceilings:** max-pods from ENI/IP math; allocatable < capacity (kubelet reserves)

---

## FINAL RECALL STORY (read 3 times)

> *`kubectl apply` walks in the one front door: the **API server** checks who I am (IAM), what I may do (RBAC), what policy says (admission), and writes desired state to **etcd** — the ledger, guarded by a **Raft quorum**. The **deployment controller** turns my intent into a ReplicaSet, then pods; the **scheduler** filters and scores nodes and binds each pod; the node's **kubelet** — speaking **CRI** to containerd, **CNI** for the pod's VPC IP, **CSI** for the volume — makes it real, and once **readiness** passes, the endpoints update and **kube-proxy**'s kernel rules on every node start steering the Service's traffic there, names resolved by **CoreDNS**. Thursday a node dies: heartbeats stop, the node controller marks it NotReady, evicts at five minutes, controllers re-create, the scheduler re-places, the autoscaler replaces the metal — nobody paged. Friday kubectl crawls for everyone: a dead **admission webhook** stalling every write — fixed, and running pods never noticed, because the **data plane outlives the brain**. The month's subtle mystery — half-empty nodes refusing pods — was the **ENI/IP ceiling**, not CPU. And the whole system is just one idea wearing many uniforms: **watch the ledger, compare to reality, close the gap, forever.***

Master Part 0's three rules and Q10's narration, and you can *derive* the answer to any architecture question they invent — which reads as far more senior than recall ever does.
