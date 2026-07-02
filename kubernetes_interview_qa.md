# Kubernetes Interview Guide — From Zero to Confident Answers
### The most important document in your prep set. Read Part 0 twice.

---

## PART 0: The Mental Model (everything in Kubernetes follows from these four ideas)

### Idea 1 — The problem it solves
You have containers (from Docker) — great, portable, isolated. But production needs *hundreds* of them across *many* machines, restarted when they crash, scaled when load rises, connected to each other, updated without downtime. Doing that by hand is impossible. Kubernetes is the **container orchestrator**: you tell it WHAT you want running, and it continuously makes it happen across a fleet of machines.

### Idea 2 — Desired state + reconciliation loop (THE core idea)
You never tell Kubernetes *steps*; you declare *desired state* in YAML: "I want 3 replicas of this container, with this much memory, reachable on this port." Kubernetes runs an endless loop: **observe actual state → compare to desired → act to close the gap.** A container crashes at 3am? The loop notices actual (2 replicas) ≠ desired (3) and starts a replacement. Nobody is paged; that's the point. Sound familiar? It's the same declarative model as Terraform — but running continuously, every few seconds, forever.

### Idea 3 — The architecture: control plane + worker nodes
A **cluster** = machines working as one. Two kinds:

**Control plane** (the brain — managed FOR you by EKS on AWS):
- **API server** — the front door; every command, component, and kubectl call talks to it, nothing bypasses it
- **etcd** — the database storing all cluster state (every object you've declared)
- **Scheduler** — decides *which node* each new pod should run on (based on resources, constraints)
- **Controller manager** — runs the reconciliation loops (the "notice and fix" logic)

**Worker nodes** (the muscle — EC2 instances you manage):
- **kubelet** — the agent on each node; receives pod assignments from the API server and makes containers actually run
- **Container runtime** — actually runs containers (containerd)
- **kube-proxy** — handles the networking rules that make services work

**Flow to narrate:** `kubectl apply` → API server validates and stores in etcd → scheduler assigns the pod a node → that node's kubelet pulls the image and starts containers → controllers keep watching forever.

### Idea 4 — Everything is an object; objects wrap objects
Kubernetes is a small set of Lego pieces. The **Pod** is the atom; everything else manages, exposes, or configures pods. Learn the pieces in Part 1 and every question becomes "which piece does this, and what happens when it misbehaves."

---

## PART 1: The Core Objects (concept + interview answer for each)

**Q1. What is a Pod? Why not just "container"?**
> "The pod is Kubernetes' smallest deployable unit: one or more containers that share a network namespace (same IP, can talk over localhost) and can share volumes. Usually one main container per pod; extra 'sidecar' containers handle support duties — log shipping, proxies. Why the wrapper exists: it gives Kubernetes one schedulable unit with one IP and one lifecycle, however many processes cooperate inside. Key mindset: **pods are cattle, not pets** — mortal, replaceable, never repaired in place; if one is sick you kill it and let the controller make a new one. In Domino, every workspace and job is exactly one pod."

**Q2. What is a Deployment (and ReplicaSet)?**
> "You almost never create bare pods — a bare pod that dies stays dead. A **Deployment** declares 'keep N replicas of this pod template running, and here's how to roll out changes.' Under the hood it manages a **ReplicaSet** (the object that literally maintains the replica count). The Deployment's superpower is **rolling updates**: change the image tag and it gradually creates new-version pods while draining old ones — zero downtime — and `kubectl rollout undo` reverts. Deployment = stateless app workhorse; it's what runs Domino's platform services and model APIs."

**Q3. What is a Service? Explain the types. (guaranteed question)**
> "Pods are mortal and their IPs change constantly, so you never point at pods directly. A **Service** is a stable virtual IP + DNS name that load-balances across whatever pods currently match its **label selector**. Types:
> **ClusterIP** (default) — reachable only inside the cluster; how microservices find each other.
> **NodePort** — opens a fixed port on every node; crude external access, mostly for dev.
> **LoadBalancer** — asks the cloud for a real load balancer (an AWS NLB/ALB) pointing at the service; the standard external front door on EKS.
> The wiring detail interviewers probe: the Service selects pods by **labels**, and only pods passing their **readiness probe** are included in its **Endpoints** list. Half of 'service is unreachable' tickets are a label typo or failing readiness — check `kubectl get endpoints` first."

**Q4. What is Ingress?**
> "One LoadBalancer per service gets expensive and unmanageable. **Ingress** is layer-7 routing: one entry point routing by hostname and path — `domino.company.com/api` → this service, `/auth` → that one — plus TLS termination. It needs an **ingress controller** (nginx, ALB controller) actually running to enforce the rules. Domino's front door — and every workspace URL — flows through this layer; 'pod is Running but user can't connect' usually lives here."

**Q5. ConfigMaps and Secrets?**
> "Both inject configuration into pods (as environment variables or mounted files) so images stay environment-agnostic — same image, different config in dev vs prod. **ConfigMap** = non-sensitive settings. **Secret** = credentials/keys — access-controllable via RBAC and kept out of pod specs, but the honest caveat: base64-encoded, not encrypted by default, so real setups add encryption at rest and often an external secret manager (AWS Secrets Manager) synced in. Also the answer to 'app can't find its config': the ConfigMap/Secret is missing, misnamed, or mounted at the wrong path."

**Q6. What is a Namespace?**
> "A virtual partition of the cluster: names, RBAC permissions, and resource quotas scope to a namespace. It's how one physical cluster hosts multiple teams/apps without collisions — and how Domino separates its own services (`domino-platform`) from user workloads (`domino-compute`). Quotas per namespace cap how much CPU/memory a tenant can consume; RBAC per namespace controls who can touch what."

**Q7. PersistentVolumes, PVCs, StorageClasses? (the storage trio)**
> "Containers' filesystems die with them, so durable data needs external volumes, abstracted in three layers: a **PersistentVolume (PV)** is actual storage (an EBS disk, an EFS share); a **PersistentVolumeClaim (PVC)** is a pod's request for storage ('give me 50Gi, read-write'); a **StorageClass** is the recipe for creating PVs on demand — pod claims via PVC, StorageClass dynamically provisions the PV from AWS. Key distinction to volunteer: **EBS = one node at a time** (ReadWriteOnce — scratch space, databases); **EFS = many pods across nodes simultaneously** (ReadWriteMany — shared data, Domino Datasets). 'Pod stuck Pending with unbound PVC' = this layer: wrong StorageClass name, CSI driver missing, or AZ mismatch between pod and EBS volume."

**Q8. StatefulSet, DaemonSet, Job/CronJob — when does each exist?**
> "**StatefulSet** — for stateful apps needing stable identity: pods get sticky names (mongo-0, mongo-1), stable per-pod storage, and ordered startup — databases like Domino's MongoDB. **DaemonSet** — 'exactly one of this pod on every node': log collectors, node monitoring agents, the NVIDIA driver plugin on GPU nodes. **Job** — run to completion, retry on failure; **CronJob** — a Job on a schedule. Domino batch executions are Kubernetes Jobs; backups are CronJobs. Quick mapping: Deployment=stateless, StatefulSet=stateful, DaemonSet=per-node, Job=finite."

---

## PART 2: Scheduling, Resources & Health (where platform engineering lives)

**Q9. Requests vs limits — explain precisely. (guaranteed, and where seniority shows)**
> "**Request** = what the scheduler reserves for the pod — the guarantee, used to decide which node has room. **Limit** = the ceiling enforced at runtime. The two enforcement behaviors differ and this is the key detail: exceed the **memory** limit → the kernel **OOM-kills** the container (exit code 137, restart); hit the **CPU** limit → **throttled**, not killed — the app just gets slow. Consequences: requests set too low → nodes overcommit, pressure, evictions; too high → wasted capacity and money. Domino hardware tiers are literally request/limit presets, so tier tuning IS this concept applied. And the QoS detail for bonus points: requests==limits gives a pod 'Guaranteed' class — last to be evicted under node pressure."

**Q10. How does the scheduler pick a node? Taints, tolerations, affinity?**
> "Filter then score: eliminate nodes that can't fit (insufficient unreserved resources, wrong constraints), then score the rest and bind. The constraint tools:
> **Taints** (on nodes) repel pods — 'nobody schedules here unless they explicitly tolerate this'; **tolerations** (on pods) are the permission slip. Pattern: GPU nodes tainted so ordinary pods can't waste them; platform nodes tainted so user workloads can't disturb Domino's services.
> **Node affinity/selectors** attract pods to labeled nodes ('must run on gpu=true nodes'). Taints keep the wrong pods OUT; affinity steers the right pods IN — GPU tiers in Domino use both.
> **Pod anti-affinity** spreads replicas across nodes/AZs so one failure can't take all copies."

**Q11. Liveness vs readiness probes?**
> "Both are periodic health checks with different consequences. **Liveness**: 'is this container alive?' — fail it and kubelet **restarts** the container (self-healing for hung processes). **Readiness**: 'can it take traffic *right now*?' — fail it and the pod is **removed from service endpoints** (no restart) until it passes; covers startup warmup and temporary unhealth. The classic misconfiguration to name: an over-aggressive liveness probe on a slow-starting app creates a restart loop that looks like a crashing app but is really a probe problem — a startup probe or longer initial delay fixes it."

**Q12. Explain autoscaling: HPA vs Cluster Autoscaler.**
> "Two directions: **HPA (Horizontal Pod Autoscaler)** scales *pods* — more replicas when CPU/memory/custom metrics rise; right for serving workloads like model APIs. **Cluster Autoscaler** (or Karpenter) scales *nodes* — sees pods stuck Pending for lack of room, adds nodes; removes drained, underutilized ones. They chain: HPA adds pods → pods Pending → CA adds nodes. In Domino, CA is the star: workspace demand is inherently spiky, and its failure modes — AWS instance quota caps, GPU scarcity in an AZ, node-group max reached, scale-down blocked by pods with local storage — are everyday platform tickets."

**Q13. What is RBAC?**
> "Role-Based Access Control for the Kubernetes API: **Roles** (namespaced) or **ClusterRoles** (cluster-wide) define allowed verbs on resources ('get/list pods'), and **RoleBindings** grant them to users, groups, or **ServiceAccounts** (identities for pods themselves). Least privilege applies: platform admins ≠ read-only support ≠ CI pipelines, each bound to what they need. On EKS this connects to IAM — and a pod's ServiceAccount is also the hook for IRSA/Pod Identity, which is how Domino services get their AWS permissions."

**Q14. What actually happens when I run `kubectl apply -f deployment.yaml`? (the flow question — narrate it smoothly and you sound experienced)**
> "kubectl sends the object to the **API server**, which authenticates/authorizes me (RBAC), validates, and persists it to **etcd**. The **deployment controller** notices a new Deployment and creates a ReplicaSet; the ReplicaSet controller creates pod objects. The **scheduler** sees unbound pods, filters and scores nodes, and binds each pod to one. That node's **kubelet** sees its assignment, pulls the image via the container runtime, sets up networking (CNI assigns the pod IP), mounts volumes, starts containers, and begins running probes. As readiness passes, endpoints update and services route traffic. And the controllers never stop watching — that's why deleting a pod just summons a replacement."

---

## PART 3: Troubleshooting (the heart of the interview for a support role)

### The universal first moves — say these before any specific answer:
```
kubectl get pods -n <ns>              # what state is it in?
kubectl describe pod <pod> -n <ns>    # EVENTS at the bottom = usually the answer
kubectl logs <pod> [-c container] [--previous]   # app output; --previous = the crashed one
kubectl get events -n <ns> --sort-by=.lastTimestamp
```
**The state itself routes the diagnosis:** Pending = scheduling. ImagePullBackOff = image. CrashLoopBackOff = the app. Running-but-broken = networking/probes/app logic.

**T1. Pod stuck in Pending. Diagnose.**
> "Pending = no node accepted it — pure scheduling. `kubectl describe pod` events say why, almost always one of four:
> **(1) Insufficient CPU/memory** — no node has room for the *requests*; check whether the autoscaler is adding a node, and if not, why: node group at max, or AWS **instance quota** hit (the invisible ceiling — everything looks configured right and nothing scales).
> **(2) Taint/selector mismatch** — pod demands `gpu=true` nodes or doesn't tolerate a taint; e.g., a GPU tier when zero GPU nodes exist yet.
> **(3) Unbound PVC** — the storage claim can't be fulfilled: wrong StorageClass, missing CSI driver, or an EBS volume in a different AZ than any schedulable node.
> **(4) Quota** — the namespace's ResourceQuota is exhausted.
> In Domino this is THE 'my workspace won't start' ticket, and 'GPU tier + quota cap' is the single most common real cause I'd check first."

**T2. CrashLoopBackOff. Diagnose.**
> "The container starts, dies, and kubelet retries with growing backoff — the pod is fine, the *process* is failing. First: `kubectl logs --previous` (the crashed container's output — the current one may be too young to have logged). Then `describe` for the **exit code**: **137 = OOMKilled** — memory limit exceeded; fix = bigger limit/tier or fix the leak. **1/other codes** = app error — missing config or env var (is the ConfigMap/Secret mounted?), can't reach a dependency, bad entrypoint. And the sneaky one: exit 0 with restarts, or healthy-app restarts = a **misconfigured liveness probe** killing a working container — check probe settings before blaming the app. In Domino: user's workspace crashlooping usually = OOM (undersized tier) or a broken environment image."

**T3. ImagePullBackOff / ErrImagePull.**
> "Kubelet can't fetch the image. Causes in likelihood order: **typo/nonexistent tag** (someone pushed `v1.2` but the spec says `v1.2.0`), **registry authentication** — missing/expired imagePullSecret or, on EKS, the node role lacking ECR permissions, **network to the registry** — private subnets with a broken NAT path, or **the registry itself down**. `describe` shows the exact pull error; testing the pull from a node isolates auth vs network. Domino flavor: an environment build published a bad revision, or the internal registry has an issue — roll users back to the previous environment revision while fixing."

**T4. Node NotReady. What happens and what do you do?**
> "NotReady = the kubelet stopped heartbeating to the API server. After the eviction timeout (~5 min default), pods on it are rescheduled elsewhere — the system self-heals if there's capacity. Diagnosis: is the machine dead (EC2 status checks), kubelet crashed/hung, **disk pressure** (full disk is the classic silent killer — image cache and logs), memory pressure, or network partition? If the node is reachable: kubelet logs (`journalctl -u kubelet`), disk (`df -h`). Treatment follows cattle logic: `cordon` (no new pods) → `drain` (evict gracefully) → recycle the instance; investigate the *pattern* (many nodes = systemic — an AMI update? overlay network issue?), not the corpse. Prevention: disk/pressure alerts BEFORE NotReady."

**T5. Service exists, pods Running — but nothing can reach the app. Walk the path.**
> "Layer walk, inside-out:
> **(1) The app itself** — `kubectl exec` into the pod (or a debug pod) and curl `localhost:<port>`: does the process actually listen, on the port the spec claims?
> **(2) Endpoints** — `kubectl get endpoints <svc>`: **empty endpoints = the smoking gun**, meaning either the service's label selector matches no pods (typo — compare selector vs pod labels) or the pods are failing **readiness** (described as Running but not Ready — check probe).
> **(3) Port wiring** — service `port` vs `targetPort` vs what the container listens on: a mismatch fails silently.
> **(4) DNS** — from a debug pod, `nslookup <svc>.<ns>.svc.cluster.local`; cross-namespace calls need the namespace in the name — a classic gotcha.
> **(5) NetworkPolicies** — if the cluster uses them, is traffic between these namespaces allowed?
> **(6) The edge** — ingress rules, ALB target group health, TLS cert validity.
> Naming this sequence calmly is worth more in a support-role interview than any single fact."

**T6. "Everything was fine; now the whole cluster is degrading." Approach?**
> "Blast-radius triage: **what changed** (deploys, upgrades, config — check change records first, most incidents are self-inflicted), then look for the shared dependency behind a broad failure: **control-plane/API health** (EKS is managed but not magic — throttling shows as slow kubectl and controller lag), **DNS** (CoreDNS overwhelmed = *everything* looks broken since services resolve by name — restart/scale CoreDNS is a legitimate mitigation), **cascading evictions** (one node dies → pods pile onto others → pressure spreads — check node group capacity), **certificate expiry** (simultaneous TLS failures everywhere), or **IP exhaustion** on the VPC CNI (pods can't get IPs — subnets out of addresses). The discipline: mitigate the spread first — add capacity, restart the choking shared service — then root-cause."

**T7. OOMKilled keeps recurring for a workload. Beyond 'raise the limit'?**
> "'Raise the limit' repeated forever is capacity whack-a-mole, not engineering. Real approach: **measure** actual usage over time (Prometheus, `kubectl top`) — is it a slow leak (line climbs to death: an app bug to fix) or genuine working-set (load outgrew the allocation: resize with data)? Right-size request AND limit from the observed p99, not vibes. In Domino terms: recurring user OOMs = a tier-sizing conversation backed by usage graphs — maybe the user needs the bigger tier, or maybe their pandas code loads a 40 GB file it should be chunking — the graph tells us which, and 'here's your memory profile' turns a complaint into a collaboration."

---

## PART 4: Scenario / Design Questions

**S1. "Design the Kubernetes layout for a multi-team data science platform." (the JD-in-one-question)**
> "Separation of concerns at every layer: **namespaces** per concern — platform services isolated from user compute, quotas on user namespaces; **node pools** by workload class — tainted platform nodes (control-plane services protected from noisy users), general autoscaling compute, tainted scale-to-zero GPU pools; **RBAC** least-privilege per persona — admins, support (read-mostly), CI service accounts; **pod-level hygiene** — every workload with requests/limits (enforced via defaults/LimitRanges so nothing runs unbounded), readiness/liveness on services, anti-affinity spreading platform replicas across AZs; **storage** — EBS classes for scratch, EFS for shared; **observability** — Prometheus/Grafana with alerts on the leading indicators: pending-pod age, node disk, queue depth, cert expiry. Which is, not coincidentally, a description of how Domino ships."

**S2. "You need to patch/upgrade nodes with zero user disruption. Process?"**
> "Rolling, one node (or small batch) at a time: **cordon** (no new pods land) → **drain** (evict pods gracefully — controllers reschedule them; PodDisruptionBudgets throttle the pace so services never lose too many replicas at once) → patch or, better, **replace** the node with an updated image (immutable infrastructure: fresh node from a new AMI beats patching in place) → uncordon → verify → next. Wrapped in change management, watching platform dashboards between batches. Wrinkle worth volunteering: long-running user jobs don't reschedule gracefully — drains can kill a 3-hour training run — so for compute nodes you coordinate windows or let nodes drain naturally by attrition (block new work, wait out the running jobs)."

**S3. "A deployment rollout is stuck / new version is bad. Act."**
> "`kubectl rollout status` shows stuck; `describe` + new-pod states show why — usually the new pods can't become Ready (image issue, failing readiness, insufficient resources for the surge). If the version is bad in behavior: **`kubectl rollout undo`** immediately — rollback is one command *because* Deployments keep ReplicaSet history; restore first, diagnose second. Prevention layers: maxUnavailable/maxSurge tuned so a bad rollout can't take everything down, readiness probes honest enough to actually gate traffic, and post-deploy smoke tests in the pipeline that trigger the undo automatically."

**S4. "How do Kubernetes and Domino relate — why does your K8s knowledge transfer?"**
> "Domino is a Kubernetes application through and through: its control plane is Deployments/StatefulSets in a platform namespace; every user workspace and job is a pod in a compute namespace; environments are container images; hardware tiers are requests/limits plus node selectors/tolerations; GPU tiers are taints + the NVIDIA device plugin; Datasets are EFS-backed PVCs; workspace URLs flow through ingress; scale is the cluster autoscaler. So every Domino support scenario reduces to a Kubernetes diagnosis with Domino vocabulary on top — which is exactly why I've invested my preparation in the Kubernetes layer: it's the layer where the actual troubleshooting happens."

---

## PART 5: Rapid-Fire One-Liners (skim before the interview)

- **Reconciliation:** declare desired state; controllers loop observe→compare→act, forever — self-healing IS this loop
- **Control plane:** API server (front door) / etcd (state DB) / scheduler (placement) / controllers (the loops). **Node:** kubelet / runtime / kube-proxy
- **Object ladder:** Deployment → ReplicaSet → Pods; Service (stable IP over label-selected, Ready pods) → Ingress (L7 routing, one front door)
- **Service types:** ClusterIP (internal) / NodePort (crude) / LoadBalancer (cloud LB)
- **Workload types:** Deployment=stateless / StatefulSet=stable identity / DaemonSet=one-per-node / Job=run-to-completion / CronJob=scheduled
- **Storage:** PVC (ask) → StorageClass (recipe) → PV (disk); **EBS=RWO single node, EFS=RWX many pods**
- **Requests=scheduling guarantee; limits=ceiling. Memory over limit → OOMKill (exit 137); CPU over limit → throttled**
- **Taints repel (out), affinity attracts (in)** — GPU/platform nodes tainted; GPU tiers tolerate + select
- **Liveness fail → restart; readiness fail → out of endpoints (no restart)**; aggressive liveness on slow starter = fake crash loop
- **HPA scales pods; Cluster Autoscaler scales nodes; they chain**
- **Debug quartet:** `get` → `describe` (read EVENTS) → `logs --previous` → `events --sort-by`
- **State routes diagnosis:** Pending=scheduling / ImagePull=image/auth / CrashLoop=app or probe / Running-unreachable=endpoints→ports→DNS→policy→ingress
- **Empty `kubectl get endpoints` = selector mismatch or readiness failing** — the #1 service bug
- **Node NotReady:** kubelet silent → pods evicted ~5min; check disk pressure first; cordon→drain→replace
- **`rollout undo`** = instant Deployment rollback; **cordon/drain** = safe node maintenance
- **Cluster-wide weirdness:** what changed? → CoreDNS, API throttling, certs, IP exhaustion, cascading eviction

---

## FINAL RECALL STORY (read 3 times — it chains the entire system)

> *I `kubectl apply` a Deployment: the **API server** checks my RBAC and writes it to **etcd**; controllers spawn a **ReplicaSet**, which spawns pods; the **scheduler** filters nodes — my pods' **requests** don't fit anywhere, so they sit **Pending** until the **cluster autoscaler** adds a node (last month it couldn't: AWS **quota** cap — the invisible ceiling). The new node's **kubelet** pulls the image — once it couldn't, **ImagePullBackOff**, an expired registry secret — mounts the **PVC** (EBS scratch; the shared data is **EFS**, because many pods mount it at once), and starts containers. One enters **CrashLoopBackOff**; `logs --previous` and **exit 137** say OOMKilled — limit raised after checking the memory graph showed real working set, not a leak. **Readiness** passes, the pod joins the **Service's endpoints** — last week a label typo left endpoints empty and 'the app was down' while every pod ran perfectly. Traffic arrives through **Ingress** on one ALB. Tuesday a node goes **NotReady** — disk full of image cache — pods evict and reschedule (**cattle, not pets**); I cordon, drain, replace it, and add a disk alert so next time we act *before* the corpse. The bad Thursday deploy? **`rollout undo`**, service restored in seconds, diagnosis after. And through all of it, the **reconciliation loop** never sleeps: desired versus actual, observe, compare, act — the whole platform, and the whole job, is keeping that loop's world true.*

If you internalize Part 0 and can retell that story, you can reason through any Kubernetes question they ask — and reasoning out loud from the model is exactly what senior interviews reward.
