# Kubernetes Mastery: From First Principles to Interview Confidence

*A complete self-study book for Sumit — built for understanding, not memorization.*

## How to use this book

Every chapter follows the same structure: the core mental model first, then the mechanics, then what breaks in production and how to diagnose it, and finally recall questions. The recall questions are not optional decoration — they are the retention mechanism. Read a chapter, close the book, and answer the questions out loud in your own words as if an interviewer asked them. If you can't, re-read only the part you missed. Revisit each chapter's questions again after 2 days, then 7 days, then 21 days (spaced repetition). Mix questions from old chapters into new study sessions (interleaving). Pair every chapter with hands-on practice on a kind/minikube cluster or KillerCoda — typing commands builds recall under pressure in a way reading never will.

The single organizing idea of this entire book: **Kubernetes is a reconciliation engine.** You declare desired state, it is stored in etcd, and a collection of independent controllers each watch the API server and work to make actual state match desired state. Every component, every object, every failure mode in this book is an instance of that one idea. When you face an unfamiliar interview question, derive the answer from this model.

---

# Chapter 1: Architecture and the Reconciliation Loop

## The mental model

Kubernetes is not a container platform; it is a distributed system whose only job is to converge reality toward a declared intent. You never command Kubernetes to *do* something — you declare what *should exist*, and a swarm of controllers does whatever is necessary, forever, to keep it true. This is why a deleted pod comes back, why a dead node's workloads reappear elsewhere, and why the system self-heals without any central orchestrator issuing commands.

## The control plane (the brain)

**kube-apiserver** is the front door and the only door. Every component — kubectl, controllers, the scheduler, kubelets, your pods — talks exclusively to the API server. It authenticates, authorizes, validates, and admits every request, then reads/writes etcd. Nothing else touches etcd directly. This hub-and-spoke design is why Kubernetes is debuggable: all state changes pass one chokepoint.

**etcd** is a distributed, consistent key-value store (Raft consensus) holding the entire cluster state — every object you've ever applied lives here as the record of truth. Think of it like Terraform state: the declared record that reconciliation compares reality against. It typically runs as 3 or 5 members for quorum; lose quorum and the cluster becomes read-only in effect (running workloads continue, but no changes can be made).

**kube-scheduler** watches for pods with no `nodeName` set. For each, it runs two phases: *filtering* (which nodes are feasible — enough CPU/memory for the pod's requests, matching nodeSelector/affinity, tolerating taints, volume constraints) and *scoring* (rank the survivors — spread, resource balance, affinity preferences). Then it does one tiny thing: it writes `nodeName` on the pod via the API server. It never contacts a node. Scheduling is a database update.

**kube-controller-manager** runs dozens of independent control loops: the Deployment controller, ReplicaSet controller, Node controller, Job controller, EndpointSlice controller, ServiceAccount controller, and more. Each loop is embarrassingly simple: watch a resource type, compare desired vs actual, act to close the gap, repeat. On managed clouds there's also a **cloud-controller-manager** that reconciles cloud resources (load balancers for Services, node lifecycle against the cloud API).

On EKS, the entire control plane is managed by AWS — you never see these as pods you administer; you consume the API server endpoint and AWS guarantees etcd, scheduler, and controllers.

## The worker node (the muscle)

**kubelet** is the node agent. It watches the API server for pods bound to *its* node, tells the container runtime to run them, executes liveness/readiness/startup probes, mounts volumes, and continuously reports pod and node status back. The kubelet is the only component that actually *starts* your workload.

**Container runtime** (containerd, CRI-O) does the low-level work: pull images, create containers via the CRI (Container Runtime Interface). Docker knowledge maps directly here — containerd is the same engine that powered Docker underneath.

**kube-proxy** programs each node's networking (iptables or IPVS rules) so that traffic to a Service's virtual IP gets load-balanced to real pod IPs. It watches Services and EndpointSlices and reconciles kernel rules — networking as a control loop.

## The story of `kubectl apply` (memorize the story, not the facts)

You apply a Deployment asking for 3 nginx replicas. (1) kubectl sends YAML to the API server, which authenticates, validates, and writes the Deployment to etcd. Nothing is running — an intention has been recorded. (2) The Deployment controller sees a Deployment with no ReplicaSet and creates one; the ReplicaSet controller sees desired 3 / actual 0 and creates 3 Pod objects. Still nothing running — just more records. (3) The scheduler sees 3 unscheduled pods, filters and scores nodes, and writes a nodeName on each. (4) Each chosen node's kubelet sees a pod bound to it, pulls the image via containerd, starts containers, runs probes, and reports status. (5) kube-proxy programs Service routing to the new pod IPs. Five actors, none commanding another, all watching the API server and reconciling their own slice.

## What breaks and how to reason about it

Because components are independent watchers, failures degrade gracefully and predictably. Scheduler down: existing pods keep running; new pods sit `Pending` with no nodeName. Controller-manager down: pods keep running; but delete a pod and nothing recreates it, deployments won't roll out. API server down: everything running keeps running (kubelets manage local pods autonomously), but no changes, no kubectl, no self-healing coordination. etcd down: same as API server down, because the API server can't read/write state. A node dies: the node controller marks it NotReady, and after the eviction timeout (~5 min by default) its pods are marked for deletion and controllers recreate them elsewhere. Derive any "what if X fails" answer from: *what does X watch, and what stops being reconciled without it?*

## Recall questions

1. Why does killing the scheduler not affect running applications? What symptom appears instead?
2. What is the only action the scheduler actually performs when it schedules a pod?
3. Trace kubectl apply for a Deployment through all five actors from memory.
4. Why is "everything watches the API server" more resilient than components commanding each other directly?

---

# Chapter 2: Pods — the Atom of Kubernetes

## The mental model

A pod is not "a container." A pod is a *shared execution environment*: one or more containers that share the same network namespace (one IP, one localhost, one port space) and can share volumes. The pod exists because some workloads are genuinely multiple processes that must live and die together — an app plus a log shipper, an app plus a proxy. Kubernetes schedules pods, never bare containers, so the unit of scheduling and the unit of shared fate are the same thing.

Crucially: **pods are mortal and disposable.** A pod is never healed — it is replaced. Its IP is ephemeral. All higher machinery (ReplicaSets, Services) exists to cope with this mortality. Internalize this and half of Kubernetes design suddenly makes sense.

## Pod lifecycle and phases

A pod moves through phases: `Pending` (accepted, but not all containers running — includes time unscheduled and time pulling images), `Running`, `Succeeded` (all containers exited 0 — Jobs), `Failed`, and `Unknown` (node unreachable). Within a running pod, individual containers have states: Waiting (with a reason like `ImagePullBackOff`), Running, Terminated.

Restart behavior is governed by `restartPolicy` (Always, OnFailure, Never) and applies to *containers within the pod on the same node* — the kubelet restarts crashed containers in place with exponential backoff (10s, 20s, 40s... capped at 5 min). That backoff loop is what `CrashLoopBackOff` means: the container starts, dies, and the kubelet is waiting increasingly long before retrying.

Termination is graceful by design: on delete, the pod gets SIGTERM (after the preStop hook, if defined), then `terminationGracePeriodSeconds` (default 30s) to exit cleanly, then SIGKILL. Simultaneously it is removed from Service endpoints — which is why apps should handle SIGTERM and finish in-flight requests.

## Probes — the kubelet's health checks

**Liveness probe**: "is this process alive or wedged?" Failure → kubelet restarts the container. **Readiness probe**: "can this container serve traffic right now?" Failure → pod is removed from Service endpoints but NOT restarted. **Startup probe**: "has this slow-starting app finished booting?" — it disables the other probes until it succeeds, protecting slow apps from liveness-kill loops.

The classic interview trap: what happens if you configure an aggressive liveness probe on a slow-starting app? The app gets killed before it finishes booting, restarts, gets killed again — a self-inflicted CrashLoopBackOff. The fix is a startup probe or a longer `initialDelaySeconds`. Also know the failure-mode difference: a bad liveness probe causes restarts; a bad readiness probe causes a running pod that mysteriously receives no traffic.

Probe types: httpGet, tcpSocket, exec, grpc. Key tuning fields: initialDelaySeconds, periodSeconds, timeoutSeconds, failureThreshold, successThreshold.

## Init containers and multi-container patterns

**Init containers** run to completion, one at a time, in order, *before* app containers start. Use them for prerequisites: wait for a database, run migrations, fetch config, fix volume permissions. If one fails, the pod restarts them per restartPolicy. In `kubectl get pods`, `Init:0/2` status tells you the pod is stuck in initialization.

**Sidecar pattern**: a helper container alongside the app — log shipper, metrics exporter, service-mesh proxy (Istio's Envoy is injected as a sidecar by a mutating webhook). Since Kubernetes 1.28+, native sidecars exist as init containers with `restartPolicy: Always`, which start before the app and keep running. **Ambassador pattern**: a local proxy the app talks to on localhost, which handles routing/TLS to the outside. **Adapter pattern**: a container that transforms the app's output (e.g., reformat logs/metrics) into a standard the platform expects.

## A pod spec worth knowing cold

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web
  labels:
    app: web
spec:
  initContainers:
  - name: wait-for-db
    image: busybox
    command: ["sh", "-c", "until nc -z db 5432; do sleep 2; done"]
  containers:
  - name: app
    image: myapp:1.4
    ports:
    - containerPort: 8080
    resources:
      requests: {cpu: 250m, memory: 256Mi}
      limits: {cpu: "1", memory: 512Mi}
    readinessProbe:
      httpGet: {path: /healthz, port: 8080}
      initialDelaySeconds: 5
      periodSeconds: 5
    livenessProbe:
      httpGet: {path: /livez, port: 8080}
      initialDelaySeconds: 15
      periodSeconds: 10
```

## What breaks and how to diagnose it

`Pending`: the scheduler can't place it. Run `kubectl describe pod` and read Events — insufficient CPU/memory, no node matches nodeSelector/affinity, taints not tolerated, or PVC unbound. `ImagePullBackOff` / `ErrImagePull`: wrong image name/tag, private registry without imagePullSecrets, or registry unreachable. `CrashLoopBackOff`: the container starts then dies — read `kubectl logs pod` and crucially `kubectl logs pod --previous` (logs of the crashed instance); common causes are app misconfiguration, failed dependency, OOMKilled, or a bad liveness probe. `OOMKilled` (exit code 137): the container exceeded its memory limit — visible in `kubectl describe pod` under Last State. `CreateContainerConfigError`: referenced ConfigMap/Secret doesn't exist. Running but no traffic: readiness probe failing — check Endpoints with `kubectl get endpoints svc-name`.

The universal triage sequence: `kubectl get pod -o wide` → `kubectl describe pod` (read Events bottom-up) → `kubectl logs [--previous]` → `kubectl exec -it pod -- sh` if you need to poke inside.

## Recall questions

1. Why do pods exist at all — why doesn't Kubernetes schedule containers directly?
2. Liveness vs readiness probe: what does each failure cause, and which one produces "running but receiving no traffic"?
3. A slow Java app keeps restarting forever right after deploy. What is happening and what are two fixes?
4. What is the difference between CrashLoopBackOff and ImagePullBackOff at the mechanism level, and which kubectl commands diagnose each?
5. Describe the graceful termination sequence, including the two signals and what happens with Service endpoints.

---

# Chapter 3: Workload Controllers

## The mental model

You almost never create pods directly, because pods are mortal. Instead you create a *controller object* that owns pods and reconciles their count and version. Each controller answers a different question about how your workload should exist: identical and interchangeable replicas (Deployment), stable identities with ordered storage (StatefulSet), one per node (DaemonSet), run to completion (Job/CronJob).

## Deployments and ReplicaSets

A ReplicaSet's entire job is: keep N pods matching this selector alive. A Deployment sits above it and manages *versions*: every time you change the pod template, the Deployment creates a NEW ReplicaSet for the new version and progressively scales it up while scaling the old one down. Old ReplicaSets are kept at 0 replicas as rollback history (`revisionHistoryLimit`, default 10). This layering is the answer to "why does a Deployment need a ReplicaSet" — separation of "keep N alive" from "manage change between versions."

Rolling update tuning lives in the strategy: `maxSurge` (how many extra pods above desired may exist during rollout) and `maxUnavailable` (how many may be missing). With replicas=10, maxSurge=2, maxUnavailable=0, you get zero-downtime rollouts that go up to 12 pods mid-roll. The alternative strategy `Recreate` kills all old pods first — needed when two versions can't coexist (e.g., exclusive lock on a RWO volume).

Rollout mechanics you should be able to type from memory:

```bash
kubectl rollout status deployment/web
kubectl rollout history deployment/web
kubectl rollout undo deployment/web            # back to previous revision
kubectl rollout undo deployment/web --to-revision=3
kubectl rollout restart deployment/web         # bounce pods without spec change
kubectl scale deployment/web --replicas=5
```

A rollout only proceeds as new pods become Ready — which means readiness probes are your deployment safety net. A new version whose readiness probe never passes will stall the rollout (visible as `ProgressDeadlineExceeded`) while old pods keep serving. That interaction — probes gating rollouts — is a favorite senior-level question.

## StatefulSets

For workloads where identity matters: databases, Kafka, Zookeeper, MongoDB. A StatefulSet gives each pod a stable ordinal name (`db-0`, `db-1`, `db-2`), a stable DNS identity via a headless Service (`db-0.db-svc.ns.svc.cluster.local`), and its own PVC created from `volumeClaimTemplates` — when `db-1` is rescheduled, it reattaches to *its own* volume. Pods are created in order (0, then 1, then 2) and terminated in reverse; updates roll from the highest ordinal down. Deleting a StatefulSet does NOT delete its PVCs — data outlives the workload by design, which is both a safety feature and a source of "why is my new StatefulSet using old data" surprises.

Deployment vs StatefulSet in one sentence: Deployments manage cattle (interchangeable, any pod can be replaced by any other), StatefulSets manage pets with name tags and their own luggage.

## DaemonSets

One pod per node (or per matching node). Used for node-level agents: log collectors (Fluent Bit), monitoring (node-exporter, Datadog agent), CNI plugins, storage drivers (the EBS CSI node driver on EKS runs as a DaemonSet). DaemonSet pods tolerate many node conditions and are scheduled even on nodes marked unschedulable for normal workloads. New node joins the cluster → the DaemonSet controller immediately reconciles a pod onto it.

## Jobs and CronJobs

A Job runs pods to successful completion (exit 0) — batch work, migrations, one-off scripts. Key knobs: `completions` (how many successes needed), `parallelism` (how many run at once), `backoffLimit` (retries before marking the Job failed, default 6), `activeDeadlineSeconds` (wall-clock kill switch), and `ttlSecondsAfterFinished` (auto-cleanup). A CronJob creates Jobs on a schedule; know `concurrencyPolicy` (Allow / Forbid / Replace — what happens if the previous run is still going) and `startingDeadlineSeconds` (how late a missed run may start). Interview flavor: "your nightly job sometimes overlaps with the previous run and corrupts data" → `concurrencyPolicy: Forbid`.

## What breaks and how to diagnose it

Deployment stuck mid-rollout: `kubectl rollout status` + `kubectl describe deployment` — usually new pods failing readiness (check the new ReplicaSet's pods) or insufficient capacity for the surge. Pods keep coming back after delete: something owns them — check `ownerReferences` in the pod YAML or `kubectl get rs,deploy`. Changed a ConfigMap but pods didn't pick it up: Deployments only roll on pod-template changes — use `kubectl rollout restart` or hash the ConfigMap into pod annotations. StatefulSet pod stuck Pending forever after node failure: its PVC (EBS = RWO, zonal) can only attach in one AZ — a scheduling/storage interaction that regularly stumps candidates. Selector immutability: you cannot change a Deployment's `spec.selector` after creation; attempting it errors.

## Recall questions

1. Why does a Deployment create ReplicaSets instead of managing pods directly? What does each layer own?
2. With replicas=4, maxSurge=1, maxUnavailable=0 — describe the rollout step by step. What gates each step?
3. Name three guarantees a StatefulSet provides that a Deployment does not, and one operational surprise around PVC deletion.
4. Your CronJob's runs sometimes overlap. Which field fixes it and what are its three values?
5. A rollout is stuck with 2 new pods not Ready and old pods still serving. Is this an outage? What protected you, and what do you check first?

---

# Chapter 4: Networking

## The mental model

Kubernetes networking rests on three flat rules: every pod gets its own IP; every pod can reach every other pod without NAT; and node agents can reach pods on their node. The CNI plugin (Calico, Cilium, flannel — on EKS, the AWS VPC CNI which assigns pods real VPC IPs from the subnet) implements these rules. Everything above — Services, DNS, Ingress — exists to solve one problem this creates: *pod IPs are ephemeral*, so nobody can depend on them directly. Networking in Kubernetes is a stack of stable names over unstable addresses.

## Services — stable virtual IPs over mortal pods

A Service is a stable virtual IP (ClusterIP) plus a label selector. The EndpointSlice controller watches pods matching the selector and maintains the list of ready pod IPs; kube-proxy programs each node's iptables/IPVS so traffic to the ClusterIP is DNAT'ed to one of those pod IPs. Two reconciliation loops, one stable front door. Only pods passing their readiness probe are listed — this is the wiring behind "readiness failure = no traffic."

Service types build on each other. **ClusterIP** (default): reachable inside the cluster only. **NodePort**: additionally opens a port (30000–32767) on every node, forwarding to the Service — crude external access. **LoadBalancer**: additionally asks the cloud controller to provision a cloud LB (on EKS: an NLB/ALB via the AWS Load Balancer Controller) pointing at the nodes/pods. **ExternalName** is odd-one-out: just a DNS CNAME to an external hostname, no proxying. And a **headless Service** (`clusterIP: None`) skips the virtual IP entirely: DNS returns the individual pod IPs — this is how StatefulSet pods get per-pod DNS names, and how clients do their own load balancing.

## DNS

CoreDNS (a Deployment in kube-system) serves cluster DNS. Every Service gets `svc-name.namespace.svc.cluster.local`; pods resolve short names via search domains, so within the same namespace `curl http://db` works, cross-namespace needs `db.other-ns`. Each pod's `/etc/resolv.conf` points to the CoreDNS ClusterIP. A huge fraction of "networking is broken" tickets are actually DNS: test with `kubectl run -it --rm dbg --image=busybox -- nslookup kubernetes.default`.

## Ingress and Gateway API

A Service exposes one thing; an **Ingress** is an L7 (HTTP) routing table: host- and path-based rules mapping to Services, plus TLS termination. Critical understanding: the Ingress *resource* is inert YAML — you need an **Ingress controller** (nginx-ingress, AWS ALB controller, Traefik) actually watching those resources and reconciling a real proxy's config. "I created an Ingress and nothing happens" almost always means no controller is installed or the `ingressClassName` doesn't match. The Gateway API is the modern successor (Gateway, HTTPRoute) with better role separation — worth name-dropping in interviews.

## NetworkPolicy

By default, all pod-to-pod traffic is allowed. A NetworkPolicy selects pods and whitelists their ingress/egress by pod selector, namespace selector, or CIDR. Two traps: policies are enforced by the CNI, and not all CNIs support them (the base AWS VPC CNI needs Calico/Cilium or newer VPC CNI features for enforcement); and policies are additive whitelists — as soon as ANY policy selects a pod, everything not explicitly allowed is denied for that direction. Default-deny is just an empty-rule policy selecting all pods. In a GxP environment, namespace isolation via default-deny + explicit allows is the expected answer to "how do you segment workloads."

## The path of a request (tell it as a story)

External user hits your app: DNS resolves to the cloud load balancer → LB forwards to the Ingress controller pods → the controller matches host/path rules and proxies to the backend Service's endpoints (often directly to pod IPs) → the pod replies. Pod calls another service internally: app resolves `payments.prod` via CoreDNS → gets ClusterIP → node's iptables (programmed by kube-proxy) DNATs to a ready pod IP → CNI routes the packet pod-to-pod.

## What breaks and how to diagnose it

Service unreachable: check `kubectl get endpoints svc` — empty endpoints means selector doesn't match pod labels (compare them character by character) or no pods are Ready. Intermittent failures to one replica: one pod failing readiness or a stale endpoint. Can reach by pod IP but not Service name: DNS (CoreDNS pods healthy? resolv.conf?) or kube-proxy on that node. Ingress 404/default backend: rule host/path mismatch or wrong ingressClassName; Ingress 502/503: backend Service has no ready endpoints. Connection works in namespace A but not from B: NetworkPolicy. The layered triage: pod → Service endpoints → DNS → Ingress → policy, in that order, halving the search space each step.

## Recall questions

1. What problem do Services solve, and what two components cooperate to make a ClusterIP actually route traffic?
2. `kubectl get endpoints mysvc` returns nothing. Give the two most likely causes and how to confirm each.
3. Explain ClusterIP vs NodePort vs LoadBalancer as layers. When is a headless Service the right tool and what does its DNS return?
4. You created an Ingress and nothing happens at all. What is the most likely missing piece and why?
5. By default, can any pod talk to any other pod? Describe how you would isolate a namespace and what component actually enforces it.

---

# Chapter 5: Storage

## The mental model

Storage in Kubernetes is deliberately split into three concerns so that teams can work independently: the *cluster/infra side* provides volumes (PersistentVolume), the *app side* requests them (PersistentVolumeClaim), and a *matchmaker* binds requests to supply (the PV controller, with StorageClasses automating supply on demand). A PVC is to storage what a pod is to compute: a declared desire that controllers reconcile into reality.

## PV, PVC, StorageClass

A **PersistentVolume** is a cluster-scoped record of an actual piece of storage (an EBS volume, an EFS filesystem, an NFS export) with capacity, access modes, and a reclaim policy. A **PersistentVolumeClaim** is a namespaced request: "I need 20Gi, ReadWriteOnce." Binding matches a claim to a compatible PV, one-to-one and exclusively.

Static provisioning (admin pre-creates PVs) is rare in cloud. **Dynamic provisioning** is the norm: the PVC names a **StorageClass**, and the class's `provisioner` (a CSI driver, e.g. `ebs.csi.aws.com`) creates the backing volume and PV on demand. Key StorageClass fields: `provisioner`, `parameters` (e.g. EBS type gp3), `reclaimPolicy`, `allowVolumeExpansion`, and `volumeBindingMode`. `volumeBindingMode: WaitForFirstConsumer` delays volume creation until a pod using the PVC is scheduled — essential for zonal storage like EBS, so the volume is created in the same AZ the scheduler picked, instead of the volume's AZ constraining or breaking scheduling.

## Access modes — and what they really mean

Access modes are capabilities of the *volume technology*, enforced per node: **RWO** (ReadWriteOnce) — mountable read-write by a single *node* (EBS); **RWX** (ReadWriteMany) — many nodes simultaneously (EFS, NFS); **ROX** — many nodes read-only; **RWOP** (ReadWriteOncePod) — a single pod. The classic pitfall: RWO is per-node, so two pods on the *same* node can share an RWO volume, which hides the bug until a pod lands on another node. On EKS the mapping to remember: EBS = RWO block storage, fast, zonal; EFS = RWX shared filesystem, cross-AZ — exactly why platforms like Domino use blob/EFS-style shared storage for datasets and project files while databases sit on EBS.

## Reclaim policy and volume lifecycle

When a PVC is deleted, the PV's `persistentVolumeReclaimPolicy` decides the fate of data: **Delete** (default for dynamic provisioning — backing volume destroyed) or **Retain** (PV becomes `Released`, data preserved, manual cleanup required; a Released PV will NOT rebind to a new claim until an admin clears `claimRef`). For production databases, Retain is the safety choice. PVC deletion is also guarded by finalizers: a PVC in use by a pod sits in `Terminating` until the pod goes away — a frequent "why won't this delete" mystery.

## The three-layer mount chain (how a file actually reaches a container)

Worth being able to narrate: (1) **Attach** — the CSI controller attaches the EBS volume to the node (it becomes a block device like /dev/nvme1n1); (2) **Node mount** — the CSI node driver (a DaemonSet) formats if needed and mounts it to a node path under /var/lib/kubelet; (3) **Container bind mount** — the kubelet bind-mounts that path into the container at the pod's `mountPath`. Stuck pods with volume errors are diagnosed by asking *which layer failed*: attach errors (volume in another AZ, attach limit reached), mount errors (filesystem corruption, permissions), or bind/config errors.

## Ephemeral volumes

`emptyDir`: scratch space sharing the pod's lifetime — dies with the pod; ideal for caches and for sharing files between containers in a pod (`medium: Memory` makes it a tmpfs). `hostPath`: mounts a node directory — powerful, dangerous, breaks portability, mostly for node agents; expect a security question about why it's restricted. ConfigMaps and Secrets also mount as volumes (next chapter), and `ephemeral` inline PVCs give per-pod dynamic volumes.

## What breaks and how to diagnose it

PVC stuck `Pending`: no StorageClass (or wrong name), provisioner/CSI driver not installed or unhealthy, or with WaitForFirstConsumer it simply waits for a pod — that's normal. Pod stuck `ContainerCreating` with volume events: read `kubectl describe pod` — attach failures (EBS in wrong AZ after node failure — the RWO-zonal StatefulSet trap from Chapter 3), "volume already attached to another node" (previous pod's node still holds it), or mount permission issues (fix with `fsGroup` in securityContext for non-root containers). `Released` PV that won't rebind: clear claimRef or accept that's by design. Volume full: expand by editing the PVC size if `allowVolumeExpansion: true`.

## Recall questions

1. Explain the division of labor between PV, PVC, and StorageClass — who creates what in dynamic provisioning?
2. Why does EBS require WaitForFirstConsumer binding mode? What goes wrong without it?
3. Two pods share an RWO volume fine for weeks, then one day the second pod is stuck ContainerCreating. What happened?
4. Walk the three-layer mount chain for an EBS-backed PVC, and name one failure at each layer.
5. A teammate deletes a PVC for a production database with reclaim policy Delete. What happens, and what two settings would have prevented data loss?

---

# Chapter 6: Configuration and Secrets

## The mental model

Twelve-factor discipline: the same image runs in every environment; configuration is injected at runtime from objects that live in the cluster. ConfigMaps hold non-sensitive config; Secrets hold sensitive material with slightly different handling. Both are just key-value objects in etcd that pods consume in two ways — as environment variables or as mounted files — and the difference between those two ways is a genuinely important operational fact.

## ConfigMaps

Create from literals, files, or directories (`kubectl create configmap app-cfg --from-file=app.properties`). Consume as env vars (`envFrom` for all keys, or `valueFrom.configMapKeyRef` per key) or as a volume, where each key becomes a file. The operational asymmetry every senior engineer must know: **env vars are frozen at container start; mounted files update in place** (within ~a minute, via kubelet sync) when the ConfigMap changes — but only if the app re-reads its files. And pods do not restart on ConfigMap change: the standard patterns are `kubectl rollout restart`, or hashing the ConfigMap into a pod-template annotation so every config change triggers a normal rolling update (Helm's checksum annotation trick).

## Secrets

Structurally a ConfigMap with `data` values base64-encoded. Say it plainly in interviews: **base64 is encoding, not encryption** — anyone who can read the Secret object has the value. Real protection comes from layers around it: RBAC restricting who can get/list secrets, encryption at rest for etcd (EKS supports KMS envelope encryption), and avoiding secrets in env vars (visible via `kubectl describe`? no — but leakable via crash dumps, `/proc`, and child processes) in favor of mounted files with tight permissions. Types: Opaque (generic), `kubernetes.io/dockerconfigjson` (registry pull credentials, referenced by `imagePullSecrets`), TLS secrets for Ingress. In serious environments, secrets live in an external manager — AWS Secrets Manager or Vault — synced in via External Secrets Operator or CSI Secrets Store driver; mentioning that is the difference between a textbook answer and a platform engineer's answer.

## Downward API and projected volumes

The Downward API injects the pod's *own* metadata into itself — pod name, namespace, node name, labels, resource requests — as env vars (`fieldRef: metadata.name`) or files. Useful for logging context and per-pod identity without external lookups. Projected volumes combine multiple sources (ConfigMap + Secret + downward API + the ServiceAccount token) into one mount — the auto-mounted SA token you saw in Chapter 8's admission story is exactly a projected, expiring, audience-bound token.

## What breaks and how to diagnose it

`CreateContainerConfigError`: the pod references a ConfigMap/Secret key that doesn't exist (typo, wrong namespace — these objects are namespaced and cannot be referenced across namespaces). App ignoring new config: env-var consumption (needs restart) vs file consumption (updates, but app must re-read). `ImagePullBackOff` on a private registry: missing/incorrect imagePullSecret. Secret value "looks like garbage": someone double-base64-encoded it, or used `data` with a plain string instead of `stringData`. Mounted ConfigMap directory hides other files: a volume mount over a non-empty path shadows the image's directory — use `subPath` to mount a single file (with the caveat that subPath mounts do NOT receive live updates).

## Recall questions

1. Env var vs volume mount for ConfigMaps: state the update-behavior difference and the standard pattern to make config changes roll pods.
2. "Kubernetes Secrets are encrypted." Correct this statement precisely, then name three real layers of secret protection.
3. What is CreateContainerConfigError and what are its two most common causes?
4. How would you give a pod its own pod name and namespace as environment variables, and why might an app need that?
5. Why does mounting a ConfigMap file with subPath change its update semantics?

---

# Chapter 7: Scheduling and Resource Management

## The mental model

Two separate systems are easy to conflate. **Scheduling** decides *where* a pod goes, once, using requests and placement rules. **Resource enforcement at runtime** decides what happens on the node as processes actually consume CPU and memory. Requests drive the first; limits drive the second. Most production incidents in this area come from misunderstanding that split.

## Requests and limits

`requests` are the scheduler's currency: a node is feasible only if the sum of requested resources of its pods plus the new pod fits its allocatable capacity. Requests are a *reservation for scheduling math*, not a runtime cap — a pod requesting 100m CPU can burst far above it if the node has slack. `limits` are runtime enforcement, and the two resources behave differently, which is a top-tier interview point: **CPU is compressible** — exceed the limit and the kernel throttles you (CFS quota), causing latency, not death; **memory is incompressible** — exceed the limit and the container is OOMKilled (exit 137). This is why "my service has mysterious latency spikes" often traces to CPU throttling and why "restarts with exit 137" means memory limits.

## QoS classes and eviction

From requests/limits Kubernetes derives a Quality of Service class per pod: **Guaranteed** (requests == limits for every container), **Burstable** (some requests set, not equal to limits), **BestEffort** (nothing set). When a *node* runs out of memory or disk, the kubelet evicts pods in roughly that reverse order — BestEffort first, then Burstable exceeding requests, Guaranteed last (interacting with pod priority). Distinguish clearly: **OOMKill** is the kernel killing a container for exceeding its own limit; **eviction** is the kubelet removing whole pods because the node is under pressure. Different mechanism, different messages (`OOMKilled` in container state vs `Evicted` pod status with a reason).

## Steering placement: selectors, affinity, taints

`nodeSelector` is the blunt tool: pod goes only to nodes with these labels. **Node affinity** is its expressive version, with `requiredDuringScheduling` (hard) and `preferredDuringScheduling` (soft) rules. **Pod affinity / anti-affinity** places pods relative to other pods — the canonical use is anti-affinity spreading replicas across nodes or zones (`topologyKey: topology.kubernetes.io/zone`) so one failure domain can't take out all replicas; `topologySpreadConstraints` is the modern, more controllable spreading mechanism.

**Taints and tolerations** invert the logic: labels/affinity are pods *choosing* nodes; taints are nodes *repelling* pods. A taint (`kubectl taint nodes gpu-1 gpu=true:NoSchedule`) blocks all pods that don't carry a matching toleration. Effects: NoSchedule, PreferNoSchedule, and NoExecute (also evicts already-running pods). Kubernetes itself uses NoExecute taints for node problems — `node.kubernetes.io/not-ready` — with default tolerations of 300s, which is the machinery behind "pods leave a dead node after ~5 minutes." The composition question interviewers love: to dedicate nodes to a team you need BOTH a taint (keep others out) AND affinity/selector on the team's pods (make them land there) — a taint alone doesn't attract anything.

## Priority and preemption; namespace-level guardrails

PriorityClasses give pods an integer priority; when a high-priority pod can't schedule, the scheduler may *preempt* (evict) lower-priority pods to make room — how system components stay alive on packed clusters. At the namespace level, **ResourceQuota** caps totals (sum of requests/limits, object counts) and **LimitRange** sets per-container defaults and bounds — the mutating-admission mechanism that injects default limits into pods that didn't specify any (connecting back to Chapter 8's gauntlet).

## What breaks and how to diagnose it

Pod Pending with `Insufficient cpu/memory`: the sum of *requests* doesn't fit anywhere — nodes can look half-idle in actual usage yet be fully *reserved*; check `kubectl describe node` (Allocated resources) and right-size requests. Pending with `didn't match node selector/affinity` or `had untolerated taint`: read the FailedScheduling event, it names the counts per reason. Latency spikes with healthy pods: CPU throttling — check container_cpu_cfs_throttled metrics; fix by raising/removing CPU limits (a widely debated practice: many platforms set CPU requests but no CPU limits). Random pod deaths on one node: node-pressure eviction — `kubectl describe node` conditions (MemoryPressure, DiskPressure). Deployment scaled but some replicas Pending only in one zone: capacity or zonal storage constraints interacting with spread rules.

## Recall questions

1. Requests vs limits: which does the scheduler use, and what happens at runtime when a container exceeds its CPU limit vs its memory limit?
2. Derive the three QoS classes and the node-pressure eviction order. How is eviction different from OOMKill?
3. You must dedicate a node pool to GPU workloads. Why is a taint alone insufficient, and what is the complete configuration?
4. Explain the ~5-minute delay before pods leave a failed node in terms of taints.
5. A cluster shows 40% actual CPU usage but new pods won't schedule for insufficient CPU. Explain to a junior engineer what is happening.

---

# Chapter 8: Security and RBAC

## The mental model

Every request — human or machine — passes a gauntlet of checkpoints inside the API server before touching etcd: **Authentication** (who are you? failure = 401), **Authorization/RBAC** (may you do this? failure = 403), then **Admission control** (mutating first — the request may be modified; validating last — final policy veto). Runtime security on the node is a separate layer governed by securityContext. Reason about any security question by locating it at the right checkpoint.

## Identity: Users and ServiceAccounts

There is no User object in Kubernetes — human identity comes from outside: client certificates (the CN is the username, the O fields are groups) or OIDC tokens; on EKS, IAM identities map to Kubernetes users/groups via access entries (or the legacy aws-auth ConfigMap). **ServiceAccounts** are for software: real namespaced objects; every pod runs as one (default SA unless specified) and receives a projected, expiring token at `/var/run/secrets/kubernetes.io/serviceaccount/token` for calling the API server. On EKS, IRSA (IAM Roles for Service Accounts) extends this: an annotation on the SA lets its pods assume an AWS IAM role via OIDC federation — the clean answer to "how do pods get AWS permissions without node-wide credentials."

## RBAC: four objects, one grammar

Roles define WHAT (rules: apiGroups + resources + verbs), bindings define WHO (subjects: users, groups, ServiceAccounts). Each comes namespaced and cluster-scoped: Role/RoleBinding within a namespace; ClusterRole/ClusterRoleBinding cluster-wide (ClusterRoles are also required for cluster-scoped resources like nodes and PVs, and for nonResourceURLs). Two load-bearing facts: RBAC is **purely additive — no deny rules exist**; and **a RoleBinding may reference a ClusterRole**, granting its permissions only inside that namespace — the pattern for defining "developer" once and binding it per namespace. Built-in ClusterRoles to know: cluster-admin, admin, edit, view.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: {name: deploy-manager, namespace: prod}
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: {name: ci-deploys, namespace: prod}
subjects:
- kind: ServiceAccount
  name: ci-bot
  namespace: prod
roleRef:
  kind: Role
  name: deploy-manager
  apiGroup: rbac.authorization.k8s.io
```

The diagnostic workhorse: `kubectl auth can-i create deployments -n prod --as=system:serviceaccount:prod:ci-bot` — test any identity's permissions instantly. 401 vs 403 tells you which checkpoint failed: 401 = the cluster doesn't know who you are (certs, tokens, kubeconfig); 403 = it knows exactly who you are and RBAC says no (hunt for the missing binding).

## Security contexts: what a running container may do

Pod-level: `runAsUser`, `runAsGroup`, `runAsNonRoot: true`, `fsGroup` (group ownership applied to mounted volumes — the fix when a non-root container can't write its PVC). Container-level adds: `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem: true`, `capabilities: {drop: [ALL], add: [NET_BIND_SERVICE]}`, `privileged` (host-level access — the answer in regulated environments is "never, and admission blocks it"), and `seccompProfile: {type: RuntimeDefault}`. This is your Linux knowledge in YAML: users, groups, capabilities, seccomp.

**Pod Security Admission** enforces baseline hygiene per namespace via labels — levels `privileged`, `baseline`, `restricted` in modes enforce/audit/warn. Label a namespace `pod-security.kubernetes.io/enforce: restricted` and non-compliant pods are rejected at admission.

## Admission control

Runs after RBAC, on writes only. **Mutating** webhooks first (they change the object): sidecar injection, default limits via LimitRange, SA token mounts, image tag rewriting. **Validating** webhooks last, judging the final post-mutation object: OPA Gatekeeper / Kyverno policies like "only images from our registry," "all pods must set limits," "no privileged containers." Frame it for pharma: admission is change control enforced in code — nothing reaches the record of truth without passing the policy gates. Operational war story to volunteer: a validating webhook whose backing service is down can block all matching writes cluster-wide; check `failurePolicy` (Fail vs Ignore) and `kubectl get validatingwebhookconfigurations` when "nothing will deploy anymore."

## What breaks and how to diagnose it

403s: `kubectl auth can-i` as the affected identity, then hunt bindings (`kubectl get rolebindings,clusterrolebindings -A -o wide | grep <subject>`). 401s: kubeconfig context, expired client cert (common on kubeadm clusters after a year), expired/absent token. Pod can't call AWS APIs on EKS: IRSA annotation missing/wrong, OIDC provider not associated. Pod rejected at creation with a policy message: read it — PSA level or a validating webhook names its reason. Everything blocked cluster-wide: broken validating webhook with failurePolicy Fail.

## Recall questions

1. Trace a kubectl request through the four checkpoints in order, naming what each can do to the request.
2. 401 vs 403 — what does each mean and where do you look next for each?
3. A RoleBinding in dev references ClusterRole admin. What exactly can the subject now do, and where?
4. Why is base64 not a security mechanism, and what three layers actually protect Secrets?
5. Why must mutating admission run before validating admission?
6. On EKS, how does a pod get AWS IAM permissions cleanly, and why is that better than node instance roles?

---

# Chapter 9: Cluster Administration

## The mental model

Administration is the care of the reconciliation engine itself: keeping the record of truth safe (etcd backups), keeping identities valid (certificates), keeping versions current (upgrades), and managing the fleet (node lifecycle). The unifying discipline is: change one layer at a time, in the right order, with a way back.

## etcd backup and restore

The whole cluster is the contents of etcd; back it up like the crown jewels. Snapshot: `ETCDCTL_API=3 etcdctl snapshot save /backup/snap.db --endpoints=https://127.0.0.1:2379 --cacert=... --cert=... --key=...` (the three TLS flags are mandatory and a favorite exam/interview detail). Restore creates a NEW data directory from the snapshot (`etcdctl snapshot restore snap.db --data-dir=/var/lib/etcd-restored`) and you repoint the etcd static pod manifest at it. On managed EKS, AWS owns etcd — your equivalent duty is backing up cluster *objects* (Velero, GitOps repos as source of truth) and data on volumes.

## Certificates

kubeadm clusters run an internal CA under /etc/kubernetes/pki; component certs expire after one year. `kubeadm certs check-expiration` shows status; `kubeadm certs renew all` renews (then restart control-plane static pods). The classic incident: exactly one year after cluster creation, kubectl starts failing with x509 errors — expired client certificates. Kubelet certs can auto-rotate if enabled. Managed clusters hide most of this, but the x509 diagnosis story is still expected knowledge.

## Cluster upgrades

Order is law: **control plane first, then nodes**; never let kubelets be newer than the API server, and version skew allows kubelets up to (historically two, now three) minor versions behind. kubeadm flow: upgrade kubeadm binary → `kubeadm upgrade plan` → `kubeadm upgrade apply v1.x.y` on the first control-plane node → upgrade kubelet/kubectl there → repeat per node with `kubeadm upgrade node`. For workers, one node at a time: `kubectl drain node --ignore-daemonsets --delete-emptydir-data` (cordons, then evicts pods respecting PodDisruptionBudgets) → upgrade kubelet → `kubectl uncordon`. On EKS: upgrade the control plane via AWS, then node groups (rolling replacement), then critical add-ons (VPC CNI, CoreDNS, kube-proxy) — add-on version compatibility is the step people forget. Upgrade one minor version at a time; no skipping.

## Node operations and disruption management

`cordon` = mark unschedulable (existing pods stay); `drain` = cordon + evict everything (except DaemonSets) for maintenance; `uncordon` = reopen. **PodDisruptionBudgets** (minAvailable / maxUnavailable) protect apps during *voluntary* disruptions: drains and rolling node replacements will pause rather than violate the budget — and a PDB of minAvailable equal to replica count will wedge a drain forever, a real-world gotcha. Cluster **autoscaling** ties in here: the Cluster Autoscaler (or Karpenter on EKS) adds nodes when pods are unschedulable and consolidates/drains underused nodes, respecting PDBs; HPA scales pods on metrics, VPA adjusts requests — know which scales what.

## Housekeeping worth naming

Namespaces as the unit of multi-tenancy (quotas, RBAC, network policy, PSA per namespace). Auditing via API server audit logs — in GxP settings, the audit trail question is coming; the answer is API audit logs shipped to durable storage plus GitOps history. Resource pruning: finished Jobs (ttlSecondsAfterFinished), old ReplicaSets (revisionHistoryLimit), orphaned PVs. Monitoring the control plane itself: API server latency, etcd fsync latency, scheduler pending pods.

## What breaks and how to diagnose it

Node NotReady: `kubectl describe node` conditions → then on the node: `systemctl status kubelet`, `journalctl -u kubelet -f` — kubelet down, runtime down, disk pressure, or network to the API server; remember the ~5-minute NoExecute taint before pods evacuate. Drain hangs: a PDB that can't be satisfied or pods with local emptyDir data (needs the flag). Post-upgrade weirdness: add-on skew (CoreDNS/CNI/kube-proxy versions). Full disk on nodes: image garbage collection thresholds, log rotation. kubectl suddenly failing cluster-wide with x509: certificate expiry.

## Recall questions

1. Write the etcdctl snapshot save command from memory, including why the TLS flags are required.
2. What is the correct upgrade order for a cluster and why must the control plane go first?
3. Explain cordon vs drain vs uncordon, and how a PodDisruptionBudget can make a drain hang forever.
4. A node goes NotReady. Give your first three commands and the timeline of what happens to its pods automatically.
5. On EKS, which admin responsibilities disappear and which remain yours?

---

# Chapter 10: Troubleshooting Mastery

## The mental model

Troubleshooting Kubernetes is not memorizing error messages; it is walking the reconciliation chain and asking, at each actor, "did you see the desired state, and did you act?" Desired state flows API server → controllers → scheduler → kubelet → runtime → network. Any symptom localizes to a link in that chain. The senior habit: narrate the chain out loud in interviews — the method impresses more than the answer.

## The universal triage kit

```bash
kubectl get pods -o wide                  # status, restarts, node, IP
kubectl describe pod <p>                  # EVENTS — read them first, bottom-up
kubectl logs <p> [-c container] --previous  # crashed instance's logs
kubectl get events --sort-by=.lastTimestamp -n <ns>
kubectl exec -it <p> -- sh                # inspect from inside
kubectl get endpoints <svc>               # is anyone behind the Service?
kubectl auth can-i <verb> <resource> --as=<subject>
kubectl describe node <n>                 # conditions, allocated resources
kubectl debug <p> -it --image=busybox --target=<c>   # ephemeral container, distroless-friendly
```

Events answer most mysteries; logs answer app crashes; describe node answers capacity and pressure.

## The decision tree by symptom

**Pending** → scheduler can't place it. Events say why: Insufficient cpu/memory (requests vs allocatable — remember reservations, not usage), untolerated taint, affinity/selector mismatch, unbound PVC, or (rarely) scheduler down.

**ImagePullBackOff / ErrImagePull** → name/tag typo, private registry needing imagePullSecrets, rate limits, or network egress. Confirm by pulling the exact image string manually.

**CrashLoopBackOff** → the app starts and dies. `logs --previous` is the truth. Causes ranked: app config error (missing env/secret), dependency unreachable, OOMKilled (exit 137 in Last State), bad liveness probe killing a healthy-but-slow app, wrong command/entrypoint (exit 1/127).

**ContainerCreating (stuck)** → infrastructure-level: volume attach/mount failures (AZ mismatch, attached-elsewhere), CNI failing to give an IP, missing ConfigMap/Secret (CreateContainerConfigError), runtime problems.

**Running but not working** → move up the stack. No traffic: readiness probe → `get endpoints` empty → label/selector mismatch. Intermittent: one bad replica. Name resolution: CoreDNS test pod (`nslookup kubernetes.default`). External access dead: Ingress class/rules → controller logs → Service → endpoints. Cross-namespace blocked: NetworkPolicy.

**Node-level trouble** → NotReady: kubelet/runtime/disk/network on the node (`journalctl -u kubelet`); pressure conditions evict BestEffort first; ~5 minutes to automatic pod evacuation via NoExecute taints.

**Cluster-wide "nothing deploys"** → validating webhook down with failurePolicy Fail, quota exhausted, or control-plane/etcd trouble.

**Terminating forever** → finalizers on the object (inspect YAML; removing finalizers is the last resort and understand why before you do), or an unreachable node holding the pod.

## Drills — do these on a real cluster

Break things deliberately and repair them: deploy an image with a wrong tag; set a liveness probe against a nonexistent path; create a Service whose selector misses by one character; set requests larger than any node; taint all nodes and watch a deployment starve; delete CoreDNS pods and watch resolution recover; fill a PVC and expand it; create a PDB equal to replica count and try draining; write a Role missing the verb you need and watch the 403, then fix it with auth can-i as your guide. Each drill welds a symptom to its mechanism — SadServers and KillerCoda have guided versions of many of these.

## Recall questions

1. From memory: the triage sequence for a pod stuck Pending vs stuck ContainerCreating — different suspects, why?
2. CrashLoopBackOff with nothing useful in kubectl logs — what command gets the real story and why?
3. A Service returns connection refused; pods are Running. Walk the exact chain you check, in order.
4. What is a finalizer and why can it hold namespaces or PVCs in Terminating forever?
5. "Nothing in the whole cluster will deploy since this morning." Name three cluster-wide suspects and how to confirm each in one command.

---

# Appendix A: Spaced Repetition Schedule

Study each chapter, then re-answer its recall questions at these intervals: same day (immediately after reading), day 2, day 7, day 21. Keep a simple log of which questions you missed; only re-read the sections behind missed questions. In every session, spend the first ten minutes answering questions from *previous* chapters before touching new material — interleaving old with new is what converts familiarity into durable recall. In the final week before an interview, switch entirely to recall: pick random questions across all chapters and answer aloud in full sentences, then explain each answer as if teaching a junior engineer (Feynman check — if you can't explain it simply, you've found a gap).

# Appendix B: Command Reference

```bash
# Inspect
kubectl get <res> -o wide | -o yaml | -A
kubectl describe <res> <name>
kubectl get events --sort-by=.lastTimestamp
kubectl logs <pod> [-c c] [--previous] [-f]
kubectl exec -it <pod> -- sh
kubectl debug <pod> -it --image=busybox

# Change
kubectl apply -f f.yaml | kubectl delete -f f.yaml
kubectl edit deploy/<d>
kubectl scale deploy/<d> --replicas=N
kubectl rollout status|history|undo|restart deploy/<d>
kubectl label|annotate <res> <name> k=v

# Nodes
kubectl cordon|drain|uncordon <node>
kubectl taint nodes <n> key=val:NoSchedule
kubectl top nodes|pods    # needs metrics-server

# Access
kubectl auth can-i <verb> <res> [--as=...] [-n ns]
kubectl config get-contexts | use-context <c>

# Fast object creation (dry-run as YAML generator)
kubectl create deploy web --image=nginx --dry-run=client -o yaml
kubectl expose deploy web --port=80 --target-port=8080 --dry-run=client -o yaml
kubectl run dbg --image=busybox -it --rm -- sh
```

*End of book. Read a chapter, close it, answer the questions aloud. Then come back and let me interrogate you like an interviewer would — that's where mastery gets proven.*
