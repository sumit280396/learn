# Elite Scenario-Based Interview Guide
## AWS · Containers · Orchestration · Data Platform Operations · Domino · Git · Spark ML · MLOps

**How to use this guide:** Every answer is written concept-first — it explains *the problem the technology solves* before the solution, so you can learn the topic from zero while also memorizing the interview answer. Scenario questions are marked **[S]**, troubleshooting questions **[T]**. Read a section end-to-end once, then re-read only the bolded "Interview answer core" lines for revision.

---

# SECTION 1 — CONTAINERS & DOCKER

## The one-paragraph foundation (read this first)

Before containers, deploying software meant installing it on a server and hoping the server had the right OS version, libraries, and configs. "It works on my machine" was the classic failure. A **container** packages the application *plus everything it needs* (libraries, runtime, config) into one portable unit called an **image**. The container runs as an isolated process on the host — it shares the host's Linux kernel (unlike a VM, which boots a whole OS). That's why containers start in seconds and are lightweight. **Docker** is the most popular tool for building and running containers.

Key vocabulary:
- **Image** = the frozen, read-only package (like a class in programming).
- **Container** = a running instance of an image (like an object).
- **Dockerfile** = the recipe that builds an image, layer by layer.
- **Registry** = a warehouse for images (Docker Hub, AWS ECR).
- **Layer** = each Dockerfile instruction creates a cached layer; unchanged layers are reused, making builds fast.

---

### Q1. [S] "Explain to a non-technical manager why we moved from VMs to containers."

**The problem:** A VM carries an entire guest operating system — gigabytes of disk, minutes to boot, and you pay for all that duplicated OS overhead on every VM. If you run 10 apps in 10 VMs, you're running 10 copies of Linux.

**The solution:** Containers share the host's kernel. Only the app and its libraries are packaged. Result: megabytes instead of gigabytes, sub-second startup, and 5–10x more density on the same hardware.

**Interview answer core:** "VMs virtualize hardware; containers virtualize the operating system. Containers give us consistency (same image runs identically in dev, test, prod), speed (start in seconds, so we can scale and recover fast), and density (more apps per server, lower cost). VMs still matter for strong isolation or different OS kernels — containers and VMs are complementary, and in the cloud our containers actually run *on* VMs (EC2 nodes)."

---

### Q2. [T] "A developer says: 'My container exits immediately after `docker run`.' How do you troubleshoot?"

**The concept you need:** A container lives only as long as its main process (PID 1). If that process finishes or crashes, the container stops. This surprises beginners — a container is not a mini-server that "stays on"; it's a wrapper around one process.

**Troubleshooting sequence (memorize this order):**
1. `docker ps -a` — see the container's **exit code**.
   - Exit 0 = the process finished normally (e.g., someone ran `ubuntu` with no long-running command — it ran a shell, the shell exited, done).
   - Exit 1 / non-zero = the app crashed (bad config, missing env var).
   - Exit 137 = killed by SIGKILL, almost always **OOM (out of memory)** — the container exceeded its memory limit.
   - Exit 126/127 = the command wasn't executable / not found (typo in CMD/ENTRYPOINT).
2. `docker logs <container>` — read the app's stdout/stderr. 90% of answers are here.
3. `docker inspect <container>` — check `State.OOMKilled: true`, the actual command run, env vars, mounts.
4. Reproduce interactively: `docker run -it --entrypoint /bin/sh <image>` — get a shell *inside* the image and run the command manually to see the real error.

**Interview answer core:** "Exit code first, logs second, inspect third, interactive shell last. Exit 137 means OOM-killed; exit 0 usually means the image has no long-running foreground process — a common Dockerfile mistake where the app was started in the background, so PID 1 exited."

---

### Q3. [T] "The container runs fine locally but crashes in production. Walk me through your diagnosis."

**Why this happens even with containers:** The image is identical, but the *environment around it* differs: environment variables, mounted volumes, network access, resource limits, secrets, and the data itself.

**Diagnosis checklist:**
1. **Environment variables** — `docker inspect` in both places; a missing `DB_HOST` in prod is the classic culprit.
2. **Resource limits** — prod often enforces memory/CPU limits that a laptop doesn't. OOM kills (exit 137) appear only in prod.
3. **Network/firewall** — locally the container can reach the internet; in prod, security groups or proxies may block the database or an external API.
4. **Volumes & permissions** — prod may run the container as a non-root user (a security best practice) that can't write to a path the app assumes is writable.
5. **Architecture mismatch** — image built on an M1/ARM laptop, deployed to x86 servers → "exec format error."
6. **Data differences** — prod data volume or malformed records trigger code paths dev never hit.

**Interview answer core:** "The image is the same, so I diff the *context*: env vars, limits, network, permissions, CPU architecture, and data. I check logs and exit codes, then reproduce prod's constraints locally — e.g., `docker run --memory=512m --user 1000` — to force the failure on my machine."

---

### Q4. [S] "Our Docker image is 3.5 GB and builds take 20 minutes. Fix it."

**Concepts needed:** (a) images are built in **layers**, and each layer is cached — if a layer's inputs didn't change, Docker reuses it; (b) everything you copy into an image stays in it forever unless you use **multi-stage builds**.

**The fixes, in order of impact:**
1. **Multi-stage build** — build the app in a fat "builder" image (with compilers, dev headers), then copy *only the final artifact* into a slim runtime image:
   ```dockerfile
   FROM python:3.11 AS builder
   RUN pip install --prefix=/install -r requirements.txt
   FROM python:3.11-slim
   COPY --from=builder /install /usr/local
   COPY app/ /app
   ```
   Compilers and build caches never reach production. Often cuts 60–80% of size.
2. **Smaller base image** — `python:3.11-slim` or `alpine` instead of full `python:3.11` (1 GB → ~150 MB). Caution: alpine uses musl libc and can break Python wheels — slim is the safer default.
3. **Layer-cache ordering** — copy `requirements.txt` and install dependencies *before* copying source code. Then editing code doesn't invalidate the (slow) dependency layer:
   ```dockerfile
   COPY requirements.txt .
   RUN pip install -r requirements.txt   # cached unless requirements change
   COPY . .                              # changes often, but it's fast
   ```
4. **.dockerignore** — exclude `.git`, datasets, `node_modules`, logs from the build context.
5. **Combine RUN commands & clean up in the same layer** — `RUN apt-get update && apt-get install -y x && rm -rf /var/lib/apt/lists/*`. Deleting files in a *later* layer does NOT shrink the image, because earlier layers are immutable.

**Interview answer core:** "Multi-stage builds for size, dependency-first COPY ordering for cache hits, slim base images, .dockerignore, and cleanup within the same RUN layer because layers are immutable."

---

### Q5. [T] "A container can't connect to another container on the same host. What do you check?"

**The concept:** Docker gives each container its own network namespace (own IP, own ports). Containers on the same **user-defined bridge network** can reach each other *by name* (Docker runs an internal DNS). Containers on the **default bridge** cannot resolve names — a classic gotcha.

**Checklist:**
1. Are both containers on the same network? `docker network inspect <net>`.
2. Are they using the *container name* as hostname (right) or `localhost` (wrong — `localhost` inside a container is the container itself, not the host, not other containers)?
3. Is the target app listening on `0.0.0.0` inside its container, not `127.0.0.1`? An app bound to 127.0.0.1 is unreachable from outside its own namespace.
4. Right port? Note: container-to-container traffic uses the *container's* port, not the host-published `-p` port.
5. Test: `docker exec -it app1 sh` then `nc -zv app2 5432` or `curl app2:8080/health`.

**Interview answer core:** "Same user-defined network, use container names not localhost, app must bind 0.0.0.0, and use the internal port. `localhost` inside a container means the container itself — that single fact explains half of Docker networking tickets."

---

### Q6. [S] "How do you handle secrets (DB passwords) with containers? What's wrong with baking them into the image?"

**Why baked-in secrets are dangerous:** Image layers are permanent and shareable. Anyone who can pull the image can extract the secret (`docker history`, or just running the image and reading env/files). Registries get compromised; images get pushed to the wrong place.

**The hierarchy of approaches (worst → best):**
1. ❌ Hardcoded in Dockerfile/code — permanent, visible in history.
2. ⚠️ Environment variables at runtime (`-e DB_PASS=...`) — better, but visible in `docker inspect` and process listings.
3. ✅ Mounted secret files (Docker/Kubernetes Secrets) — injected at runtime as files or env, never in the image.
4. ✅✅ External secret managers (AWS Secrets Manager, Vault) — the app or an init sidecar fetches the secret at startup using its **IAM role**, so no static credential exists anywhere; supports rotation and audit logs.

**Interview answer core:** "Never in the image — layers are forever. Runtime injection at minimum; in AWS I'd use Secrets Manager with IAM roles (IRSA on EKS) so the pod fetches secrets with a short-lived identity and no static credentials ever exist."

---

### Q7. [T] "Disk on a Docker host is 95% full. What's eating it and how do you clean safely?"

**What accumulates:** stopped containers, dangling images (untagged old builds), unused volumes, build cache, and — the silent killer — **container logs** (Docker's default json-file log driver grows without bound).

**Diagnosis and cleanup:**
1. `docker system df` — shows space by images/containers/volumes/cache.
2. `du -sh /var/lib/docker/containers/*/` — find giant `-json.log` files.
3. Clean progressively:
   - `docker container prune` (stopped containers)
   - `docker image prune` (dangling), `docker image prune -a` (all unused — careful)
   - `docker builder prune` (build cache)
   - `docker volume prune` — **most dangerous**: volumes hold data; confirm nothing needed lives there.
4. Prevent recurrence: configure log rotation in `/etc/docker/daemon.json`:
   ```json
   {"log-driver":"json-file","log-opts":{"max-size":"50m","max-file":"3"}}
   ```

**Interview answer core:** "`docker system df` to find the culprit, prune in order of safety (containers → images → cache → volumes last), and fix the root cause with log rotation — unbounded container logs are the #1 cause."

---

### Q8. [S] "What's the difference between CMD and ENTRYPOINT, and when does it matter operationally?"

**The mental model:** ENTRYPOINT is the fixed executable; CMD is the default arguments. `docker run image extra-args` *replaces CMD* but *appends to ENTRYPOINT*.

- `ENTRYPOINT ["python","train.py"]` + `CMD ["--epochs","10"]` → running `docker run img --epochs 50` cleanly overrides just the arguments.
- Operationally it matters for debugging: if ENTRYPOINT is set, `docker run img bash` will NOT give you a shell (it runs `python train.py bash`). You need `docker run --entrypoint bash img`.

**Interview answer core:** "ENTRYPOINT = what always runs, CMD = default args users can override. Know `--entrypoint` for debugging containers that have a fixed entrypoint."

---

### Q9. [T] "Container is running but the application inside is unresponsive. Debug it without restarting."

1. `docker exec -it <c> sh` — get inside. Check the process is alive (`ps aux`), check it's listening (`netstat -tlnp` or `ss -tlnp`).
2. `docker stats <c>` — is it pegged at its CPU limit (throttled) or at its memory ceiling (thrashing/near-OOM)?
3. `docker logs --tail 200 <c>` — deadlock messages, GC storms, connection-pool exhaustion.
4. From inside: `curl localhost:8080/health` — distinguishes "app broken" from "network path broken."
5. If no shell in the image (distroless), use `docker debug` or run a sidecar sharing the namespace: `docker run -it --pid=container:<c> --net=container:<c> nicolaka/netshoot`.

**Interview answer core:** "Exec in, verify the process and listener, check `docker stats` for CPU throttling or memory pressure, test from inside with curl to isolate app vs network, and know the netshoot-sidecar trick for shell-less images."

---

### Q10. [S] "Why do we say 'containers should be stateless'? Where does state go?"

**The problem:** A container's writable filesystem dies with the container. Orchestrators (Kubernetes) kill and recreate containers routinely — for upgrades, scaling, node failures. Any data written inside the container evaporates.

**The rule:** treat containers as **cattle, not pets** — disposable and replaceable. All state goes to external systems:
- Databases (RDS), object storage (S3), caches (Redis/ElastiCache)
- **Volumes** (Docker volumes / Kubernetes PersistentVolumes) when the app truly needs a filesystem — the volume outlives the container.

**Interview answer core:** "Container filesystems are ephemeral by design so containers stay replaceable. State lives in databases, S3, or persistent volumes. This is what makes rolling updates, autoscaling, and self-healing possible — you can kill any container at any time and lose nothing."

---

# SECTION 2 — CONTAINER ORCHESTRATION (KUBERNETES / EKS)

## The one-paragraph foundation

Docker runs containers on one machine. But production needs containers across *dozens* of machines, with automatic restart on failure, load balancing, scaling, and zero-downtime deployments. Doing this by hand is impossible. **Kubernetes (K8s)** is the orchestrator that does it: you *declare* the desired state ("run 5 replicas of this image, expose port 80") in YAML, and Kubernetes continuously works to make reality match your declaration. This **reconciliation loop** is the single most important idea — K8s doesn't run commands, it *converges toward desired state*, forever. **EKS** is AWS's managed Kubernetes: AWS runs the control plane (the brain); you run the worker nodes (EC2 or Fargate).

Vocabulary:
- **Pod** — smallest deployable unit; one or more containers sharing network/storage. Usually 1 app container per pod.
- **Node** — a worker machine (EC2 instance in EKS).
- **Deployment** — manages a set of identical pods (replicas), handles rolling updates.
- **Service** — a stable virtual IP + DNS name in front of ever-changing pods (pods die and get new IPs; Services don't).
- **kubelet** — agent on each node that starts/stops containers as told.
- **Control plane** — API server (front door), etcd (the database of desired state), scheduler (decides which node runs each pod), controllers (the reconciliation loops).
- **Namespace** — a virtual partition of the cluster for isolation (teams, environments).

---

### Q11. [S] "Explain what happens, end to end, when you run `kubectl apply -f deployment.yaml`."

This question tests whether you understand the architecture. The answer:

1. `kubectl` sends the YAML to the **API server** (authenticated via your kubeconfig, authorized via RBAC).
2. API server validates it and stores the desired state in **etcd**.
3. The **Deployment controller** notices a new Deployment, creates a **ReplicaSet**; the ReplicaSet controller creates **Pod** objects (still just database records — nothing is running yet).
4. The **scheduler** sees pods with no assigned node, scores nodes (enough CPU/memory? matching labels? taints?) and binds each pod to a node.
5. The **kubelet** on that node sees "I've been assigned a pod," pulls the image, and starts the container via the container runtime (containerd).
6. Kubelet reports status back; controllers keep watching forever. If a pod dies, the ReplicaSet controller sees actual (4) < desired (5) and creates a replacement.

**Interview answer core:** "Declarative flow: API server → etcd → controllers create pods → scheduler places them → kubelet runs them — and the loop never stops watching. That's why K8s self-heals: it's not executing a script, it's enforcing a state."

---

### Q12. [T] "A pod is stuck in `Pending`. Diagnose it."

**Meaning of Pending:** the pod exists in etcd but the **scheduler cannot place it on any node** (or it's placed but volumes can't attach). It's a scheduling problem, not an app problem.

**Steps:**
1. `kubectl describe pod <p>` → read **Events** at the bottom. This almost always names the cause:
   - `Insufficient cpu/memory` — no node has room for the pod's **requests**. Fix: lower requests, add nodes, or check cluster autoscaler.
   - `node(s) had taint ... that the pod didn't tolerate` — nodes are reserved (e.g., GPU nodes); pod lacks a toleration.
   - `didn't match node selector/affinity` — pod demands a label no node has.
   - `pod has unbound immediate PersistentVolumeClaims` — storage can't be provisioned (wrong StorageClass, EBS in a different AZ than the node — EBS is AZ-locked!).
2. `kubectl get nodes` — are nodes Ready at all? Is the cluster autoscaler failing to add nodes (check its logs / EC2 quotas)?
3. On EKS specifically: **IP exhaustion** — the VPC CNI gives each pod a real VPC IP; small subnets run out of IPs and pods stay Pending.

**Interview answer core:** "Pending = unschedulable. `kubectl describe pod` Events tell you which of the four classic causes it is: insufficient resources, taints, affinity/selector mismatch, or PVC/zone issues. On EKS add subnet IP exhaustion to the list."

---

### Q13. [T] "A pod is in `CrashLoopBackOff`. What does that mean and what do you do?"

**Meaning:** The container **starts, then crashes, repeatedly**; kubelet restarts it with exponentially increasing back-off delays (10s, 20s, 40s… max 5m). The scheduling worked — the *application* is failing.

**Steps:**
1. `kubectl logs <p>` — the current attempt's logs.
2. `kubectl logs <p> --previous` — **the killer flag**: logs from the *crashed* attempt, which usually contain the actual error (stack trace, "connection refused to DB," missing env var).
3. `kubectl describe pod <p>` — exit code:
   - 1 = app error; 137 = OOMKilled (raise memory limit or fix a leak); 139 = segfault.
   - Also check: is it the **liveness probe** killing a healthy-but-slow app? Events will say "Liveness probe failed." A too-aggressive probe (short timeout, app needs 60s to warm up) causes crash loops of perfectly good apps — fix with a **startupProbe** or longer `initialDelaySeconds`.
4. Common root causes: bad config/secret reference (also shows as `CreateContainerConfigError`), DB not reachable at startup, migrations failing, wrong command.
5. To debug interactively when it dies too fast: temporarily override the command to `sleep 3600` (`kubectl debug` or edit the Deployment), then exec in and run the app by hand.

**Interview answer core:** "CrashLoopBackOff = starts then dies, repeatedly. `logs --previous` for the real error, exit code 137 = OOM, and always suspect the liveness probe — probes killing slow-starting apps is one of the most common self-inflicted crash loops."

---

### Q14. [T] "Pod shows `ImagePullBackOff`. Causes and fixes?"

**Meaning:** kubelet can't download the image.

Four causes cover ~99%:
1. **Typo / wrong tag** — image or tag doesn't exist. `kubectl describe pod` shows "manifest not found."
2. **Private registry auth** — "unauthorized." Need an `imagePullSecret`; on EKS pulling from **ECR**, the node's IAM role needs ECR read permissions (usually there by default with managed node groups — check if someone changed the role).
3. **Network** — nodes in private subnets can't reach the registry: NAT gateway missing/broken, or no VPC endpoints for ECR (`com.amazonaws.region.ecr.dkr`, `.ecr.api`, plus **S3 gateway endpoint** — ECR stores layers in S3; forgetting the S3 endpoint is a famous EKS gotcha).
4. **Rate limiting** — Docker Hub anonymous pull limits. Fix: authenticate or mirror images into ECR.

**Interview answer core:** "Describe the pod and read the pull error: not-found → typo, unauthorized → pullSecret/IAM, timeout → NAT or missing ECR+S3 VPC endpoints, 429 → Docker Hub rate limits."

---

### Q15. [T] "Pods are Running and Ready, but users get errors / the Service doesn't route traffic. Debug the networking path."

**The concept:** A **Service** selects pods by **labels** and forwards to the pod's **targetPort**. The most common failure is a silent mismatch — the Service selector matches *zero* pods, and nothing errors loudly.

**Steps (this sequence is gold in interviews):**
1. `kubectl get endpoints <svc>` — **the single most useful command**. Empty endpoints = selector doesn't match any pod labels, or pods aren't Ready (failing readiness probe ⇒ removed from endpoints).
2. Compare `kubectl get pods --show-labels` with the Service's `selector`.
3. Check ports: Service `targetPort` must equal the port the app actually listens on inside the container.
4. Test each hop from a debug pod (`kubectl run tmp --rm -it --image=nicolaka/netshoot -- sh`):
   - `curl <pod-ip>:<port>` — app itself OK?
   - `curl <service-name>:<port>` — Service/DNS OK?
   - If pod-IP works but service-name fails → DNS (CoreDNS) or kube-proxy issue.
5. If in-cluster works but external doesn't → Ingress/ALB layer: check Ingress annotations, ALB target group health checks, and security groups (on EKS, node/pod security groups must allow the ALB's traffic).

**Interview answer core:** "Work the chain: endpoints → labels/selector → targetPort → pod IP → service DNS → ingress/ALB. `kubectl get endpoints` empty means label mismatch or failing readiness probes — that's the answer half the time."

---

### Q16. [S] "Explain requests vs limits, and what happens when a node runs out of memory."

**The problem:** Many pods share a node. Without rules, one greedy pod starves the rest.

- **Request** = guaranteed reservation, used by the *scheduler* to decide placement. Sum of requests on a node ≤ node capacity.
- **Limit** = hard ceiling, enforced at *runtime*.
  - CPU over limit → **throttled** (slowed, not killed). Compressible resource.
  - Memory over limit → **OOMKilled** (exit 137). Incompressible — you can't "slow down" memory.

**Node memory pressure:** if the node itself runs low, kubelet **evicts** pods — lowest QoS class first: BestEffort (no requests/limits) → Burstable (requests < limits) → Guaranteed (requests = limits) last. Setting requests=limits for critical workloads protects them.

**Interview answer core:** "Requests are for scheduling, limits for runtime enforcement. CPU throttles, memory kills. QoS classes decide eviction order under node pressure — Guaranteed pods survive longest. In a data-science platform I'd set generous memory requests for training workloads because OOM kills mid-training waste hours of compute."

---

### Q17. [S] "How does a zero-downtime deployment work in Kubernetes? What can silently break it?"

**Mechanism — rolling update:** The Deployment creates a new ReplicaSet with the new image and scales it up while scaling the old one down, governed by `maxSurge` (extra pods allowed) and `maxUnavailable`. Traffic only shifts because of **readiness probes**: a new pod receives traffic *only* when its readiness probe passes; old pods are removed from Service endpoints before being killed.

**What silently breaks zero-downtime:**
1. **No readiness probe** → K8s sends traffic to pods the instant the container starts, before the app is listening → 502s during every deploy.
2. **No graceful shutdown** → on termination K8s sends SIGTERM, waits `terminationGracePeriodSeconds` (default 30s), then SIGKILL. If the app ignores SIGTERM, in-flight requests die. Apps must catch SIGTERM and drain.
3. **Not backward-compatible DB migrations** → old and new pods run *simultaneously* during rollout; a schema change that breaks the old version causes errors mid-deploy.
4. Rollback: `kubectl rollout undo deployment/<d>` — K8s keeps old ReplicaSets for exactly this.

**Interview answer core:** "Rolling update + readiness probes + SIGTERM handling = zero downtime. Miss any one of the three and users see errors. And schema changes must be compatible with both versions because both run at once."

---

### Q18. [T] "A worker node goes `NotReady`. What happens to its pods, and how do you respond?"

**What NotReady means:** the node's **kubelet stopped heartbeating** to the API server (node crashed, network partition, kubelet crashed, or resource exhaustion on the node — full disk is a classic).

**What Kubernetes does automatically:** after ~40s the node is marked NotReady; after the eviction timeout (~5 min default) the pods on it are marked for deletion and **Deployments recreate them on healthy nodes**. Pods with local state or no controller (bare pods) are lost — another reason for statelessness.

**Your response:**
1. `kubectl describe node <n>` — Conditions: `MemoryPressure`, `DiskPressure`, `PIDPressure`? DiskPressure from container logs/images is extremely common.
2. Can you SSH/SSM to the node? Check `systemctl status kubelet`, `journalctl -u kubelet`, disk (`df -h`), and whether containerd is alive.
3. On EKS: check the EC2 instance status checks, ASG activity (was it being replaced?), and whether the node's IAM role/security groups changed.
4. If unrecoverable: `kubectl cordon` (no new pods) → `kubectl drain --ignore-daemonsets` (evict gracefully) → terminate the instance and let the node group replace it. **Cattle, not pets** applies to nodes too.

**Interview answer core:** "NotReady = kubelet heartbeat lost. K8s reschedules pods after ~5 minutes automatically. I check node conditions (disk pressure is the usual villain), kubelet logs, and if it's sick, cordon-drain-replace rather than nurse it."

---

### Q19. [S] "Explain PersistentVolumes, PVCs and StorageClasses — and the classic EBS multi-AZ trap on EKS."

**The problem:** pods are ephemeral, but some workloads need durable disk.

- **PersistentVolume (PV)** — a piece of real storage (an EBS volume, an EFS filesystem) represented as a cluster object.
- **PersistentVolumeClaim (PVC)** — a pod's *request* for storage ("give me 50Gi, ReadWriteOnce"). The pod references the PVC, never the PV directly — this decouples app from infrastructure.
- **StorageClass** — the template for *dynamic provisioning*: when a PVC references StorageClass `gp3`, the EBS CSI driver automatically creates a real EBS volume and a PV for it. No human involved.
- **Access modes:** `ReadWriteOnce` (one node — EBS), `ReadWriteMany` (many nodes simultaneously — EFS/NFS). EBS can never be RWX.
- **Reclaim policy:** `Delete` (volume destroyed with PVC — default, dangerous for prod data) vs `Retain` (volume survives).

**The EBS multi-AZ trap:** EBS volumes live in **one availability zone**. If a pod with an EBS-backed PVC is rescheduled and the scheduler picks a node in a *different AZ*, the volume can't attach → pod Pending with "volume node affinity conflict." Mitigations: `volumeBindingMode: WaitForFirstConsumer` on the StorageClass (provision the volume only after the pod is placed, in the same AZ), or use EFS for cross-AZ needs.

**Interview answer core:** "PVC = request, PV = actual storage, StorageClass = automatic provisioning template. EBS is single-AZ and RWO; EFS is multi-AZ and RWX. WaitForFirstConsumer prevents the volume-in-wrong-AZ Pending trap."

---

### Q20. [S] "What is IRSA on EKS and why is it better than node IAM roles?"

**The problem:** pods need AWS permissions (read S3, fetch secrets). The lazy way is granting permissions to the **node's** IAM role — but then *every pod on that node* inherits *all* those permissions. A compromised pod can read everything. Violates least privilege.

**IRSA (IAM Roles for Service Accounts):** binds an IAM role to a *Kubernetes service account*. Under the hood: EKS runs an OIDC identity provider; the pod gets a projected service-account token; AWS STS trusts that OIDC token and exchanges it for short-lived credentials for *that specific role*. Pod A can read only its S3 bucket; pod B can only fetch its secret. No static keys anywhere, per-pod least privilege, full CloudTrail auditability. (The newer alternative is **EKS Pod Identity** — same goal, simpler setup.)

**Interview answer core:** "IRSA gives each pod its own IAM identity via OIDC federation instead of inheriting the node's role. Least privilege, no static credentials, per-workload auditing. Any time an interviewer asks 'how does your pod securely access S3/Secrets Manager,' IRSA is the answer."

---

### Q21. [T] "Cluster performance degrades every day around 9 am. How do you investigate a recurring resource crunch?"

1. **Correlate:** 9 am = users logging in / cron jobs / batch schedules. Check CronJobs (`kubectl get cronjobs -A`) and business patterns (on a data platform: everyone starts their notebooks at 9).
2. **Measure:** metrics (Prometheus/CloudWatch Container Insights): node CPU/memory, pod restarts, pending-pod count, API-server latency at that time.
3. Typical findings and fixes:
   - Morning surge of workloads exceeds capacity → **Cluster Autoscaler/Karpenter** scaling too slowly; pre-scale on a schedule or tune it.
   - Pods with no requests set land badly and overcommit nodes → enforce requests via LimitRanges/policy.
   - One namespace hogging the cluster → **ResourceQuotas** per team/namespace.
   - Image pull storms at scale-up → smaller images, pre-pulled node images (custom AMI).

**Interview answer core:** "Recurring = scheduled. Correlate with cron and human patterns, confirm with metrics, then fix structurally: autoscaling tuned for the surge, enforced requests, and per-namespace quotas so one team can't starve the platform."

---

### Q22. [S] "When would you choose ECS or Fargate over EKS?"

- **ECS** — AWS's own, simpler orchestrator. Less to learn/operate, deeply integrated with AWS, but AWS-only and a smaller ecosystem. Great for straightforward microservices when the team is small.
- **EKS** — full Kubernetes: portable, huge ecosystem (Helm, operators, Domino runs on it), but more operational complexity.
- **Fargate** — a *serverless compute layer* for either: no nodes to manage, you pay per pod/task. Ideal for spiky or low-ops workloads; trade-offs: no daemonsets, no GPUs (a dealbreaker for ML training), higher per-unit cost at scale.

**Interview answer core:** "ECS for simplicity in an all-AWS shop, EKS when you need the Kubernetes ecosystem or portability — platforms like Domino require it — and Fargate to eliminate node management for spiky, non-GPU workloads."

---

# SECTION 3 — AWS (CORE SERVICES + TROUBLESHOOTING)

## The one-paragraph foundation

AWS rents you infrastructure by the hour/second. The pieces you must speak fluently: **EC2** (virtual machines), **VPC** (your private network: subnets, route tables, gateways), **S3** (infinitely scalable object storage), **IAM** (who can do what), **EBS/EFS** (block/file storage), **RDS** (managed databases), **CloudWatch** (metrics/logs/alarms), **ELB/ALB** (load balancers). The two mental models that unlock everything: (1) **networking is deny-by-default** — traffic flows only if route tables, security groups and NACLs all permit it; (2) **IAM is deny-by-default** — an action succeeds only if a policy explicitly allows it and nothing explicitly denies it.

Networking vocabulary:
- **VPC** — your isolated network with a CIDR range (e.g., 10.0.0.0/16).
- **Subnet** — a slice of the VPC in one AZ. *Public* subnet = its route table has a route to an **Internet Gateway**; *private* = it doesn't.
- **NAT Gateway** — lets private-subnet resources initiate outbound internet (pull packages, call APIs) without being reachable from outside.
- **Security Group (SG)** — a stateful firewall on the instance/ENI. Stateful = return traffic is auto-allowed. Only allow rules.
- **NACL** — stateless firewall at the subnet boundary; needs explicit rules both directions; rarely touched, but a classic exam/interview trap.
- **Route 53** — DNS. **VPC endpoints** — private paths to AWS services (S3, ECR) without traversing the internet/NAT.

---

### Q23. [T] "An EC2 instance in a private subnet can't reach the internet to download packages. Walk the path."

**Trace the packet — this structure impresses interviewers:**
1. **Route table** of the subnet: is there a route `0.0.0.0/0 → nat-xxxx`? If it points to an IGW instead, that's wrong for a private subnet (no public IP ⇒ IGW can't help).
2. **NAT Gateway health & placement**: the NAT GW itself must sit in a **public** subnet (one that routes 0.0.0.0/0 → IGW) and have an Elastic IP. NAT in a private subnet is a classic misconfiguration.
3. **Security group** outbound: default allows all outbound, but hardened environments restrict it — is 443 out allowed?
4. **NACLs**: outbound 443 AND inbound **ephemeral ports 1024–65535** (stateless — the response has to be explicitly allowed back in).
5. **DNS**: can it resolve at all? `nslookup amazon.com`. VPC needs `enableDnsSupport/enableDnsHostnames`; test `curl https://1.1.1.1` vs by-name to split DNS vs routing.
6. **Test each layer**: `curl -v https://registry.example.com` — DNS resolution, TCP connect, TLS — where exactly does it hang?

**Interview answer core:** "Route table → NAT placement/EIP → SG outbound → NACL both directions (ephemeral ports!) → DNS. Trace the packet hop by hop; never guess."

---

### Q24. [T] "Two EC2 instances in the same VPC can't talk to each other. Diagnose."

1. **Security groups** — 90% of cases. Does the target's SG allow inbound on that port *from the source's SG or CIDR*? Pro pattern: reference SGs by ID ("allow inbound 5432 from sg-app") instead of IPs.
2. **Same VPC ⇒ routing is automatic** (the implicit local route). If they're in *different* VPCs, you need peering/Transit Gateway and correct routes both directions.
3. **NACLs** — if the subnets differ, check both NACLs, both directions.
4. **OS-level firewall** — iptables/firewalld on the instance, or the app listening on 127.0.0.1 only (`ss -tlnp`).
5. Tools: **VPC Reachability Analyzer** (analyzes the config path and names the blocking component) and **VPC Flow Logs** (see REJECT records to identify whether SG or NACL dropped it).

**Interview answer core:** "SG first (statistically it's that), then NACLs, then the instance OS. Same-VPC routing is automatic. Reachability Analyzer gives a definitive answer when eyeballing fails."

---

### Q25. [T] "Application gets `Access Denied` when reading an S3 bucket. Systematic IAM debugging."

**The evaluation model (learn this cold):** a request is allowed only if: no **explicit Deny** anywhere, AND at least one **Allow** in the union of identity policies / resource (bucket) policies — and if the account uses **SCPs** or **permission boundaries**, those must allow it too. Explicit Deny always wins.

**Checklist:**
1. **Which identity is actually calling?** `aws sts get-caller-identity` from the app's environment. Wrong role assumed is very common (the pod using the node role instead of IRSA, a hardcoded old key in env vars overriding the role...).
2. **Identity policy** — does the role allow `s3:GetObject` on `arn:aws:s3:::bucket/*`? Careful: `GetObject` needs the `/*` object ARN; `ListBucket` needs the bucket ARN — mixing these up causes half of S3 denials.
3. **Bucket policy** — any explicit Deny (e.g., deny non-TLS, deny outside a VPC endpoint, deny other accounts)?
4. **Cross-account?** Then BOTH the identity policy (in account A) AND the bucket policy (in account B) must allow. Plus object ownership/ACL quirks in older buckets.
5. **KMS** — if objects are SSE-KMS encrypted, the role also needs `kms:Decrypt` on the key. "S3 says AccessDenied but the policy looks right" is very often KMS.
6. **SCP** — in AWS Organizations, an SCP may cap everything regardless of the role policy.
7. Tools: **IAM Policy Simulator**, CloudTrail (the event's `errorMessage` often names the failing policy type).

**Interview answer core:** "Confirm the caller with sts get-caller-identity, then walk the evaluation chain: identity policy → bucket policy → KMS key policy → SCP, remembering explicit Deny trumps everything and cross-account needs both sides. The two evergreen gotchas: ListBucket vs GetObject ARNs and missing kms:Decrypt."

---

### Q26. [S] "Design a highly available web application on AWS. What fails and how does it survive?"

**Layers of failure and their answers:**
- Instance dies → **Auto Scaling Group** across multiple AZs replaces it; **ALB** health checks stop routing to it within seconds.
- AZ dies → resources in ≥2 AZs; ALB and ASG are AZ-aware; RDS **Multi-AZ** fails over the database automatically (standby promoted, DNS flipped, ~1–2 min).
- Stateless web tier → session state in ElastiCache/DynamoDB, files in S3, so *any* instance can serve *any* user.
- Region dies (rare, but asked) → Route 53 failover/latency routing to a second region, S3 cross-region replication, RDS cross-region read replica or Aurora Global.
- Overload → ASG target-tracking scaling on CPU/requests; CloudFront to absorb read traffic.

**Interview answer core:** "Stateless compute in an ASG across AZs behind an ALB, state pushed to Multi-AZ RDS/S3/ElastiCache, health checks everywhere, and Route 53 for anything beyond one region. HA is about removing every single point of failure and letting automation handle replacement."

---

### Q27. [T] "The ALB returns 502/504 errors. What do these codes mean and how do you fix them?"

**Concept:** the ALB is a middleman. 5xx from the ALB tells you *which side* failed:
- **502 Bad Gateway** — the target sent a broken/immediately-closed response: app crashed mid-request, target's keep-alive timeout *shorter* than the ALB's idle timeout (target closes connection, ALB tries to reuse it), or TLS mismatch to the target.
- **504 Gateway Timeout** — target didn't answer within the ALB idle timeout (default 60s): app too slow, DB queries hanging, or SG blocking ALB→target so the connection never completes.
- **503** — no healthy targets at all (all failing health checks, or target group empty).

**Steps:**
1. Target group **health status** and the health-check config (path returns 200? port right? SG allows ALB's health checks?).
2. ALB **access logs** (enable to S3): the fields show target processing time and which target erred.
3. Fix the keep-alive rule: target's idle timeout **>** ALB's idle timeout.
4. For 504 with healthy targets: app-side latency — APM/logs, DB slow queries, thread-pool exhaustion.

**Interview answer core:** "502 = target answered badly (crash or keep-alive mismatch), 504 = target didn't answer in time, 503 = no healthy targets. Health checks and ALB access logs localize it in minutes."

---

### Q28. [S] "S3: how do you secure a bucket properly, and what is the disaster you're preventing?"

**The disaster:** public buckets leaking customer data — the most common self-inflicted cloud breach in history.

**Defense in depth:**
1. **Block Public Access** at account level — the master off-switch; overrides any accidental public ACL/policy.
2. **Bucket policies with least privilege** — specific principals, specific actions, conditions (`aws:SecureTransport` to force TLS, `aws:SourceVpce` to restrict to your VPC endpoint).
3. **Encryption** — SSE-S3 default or SSE-KMS for auditable, permissioned keys (KMS adds a second authorization layer: even with s3:* you can't read without kms:Decrypt).
4. **Versioning + MFA delete / Object Lock** — protects against deletion and ransomware.
5. **Access logging / CloudTrail data events** — who touched what.
6. Presigned URLs for temporary external sharing instead of opening the bucket.

**Interview answer core:** "Block Public Access on, least-privilege bucket policies with TLS/VPC conditions, KMS encryption as a second gate, versioning against deletion, and CloudTrail for audit. Never solve a sharing problem by making a bucket public — presigned URLs exist for that."

---

### Q29. [T] "EC2 instance is unreachable via SSH. Full diagnostic path."

1. **Instance state & status checks** (console): system check failed = AWS hardware issue → stop/start (migrates hosts); instance check failed = OS problem (kernel panic, full disk, network config broken inside).
2. **Network path:** public IP present? SG inbound 22 from your IP? Subnet route to IGW? NACLs?
3. **Key/user:** right private key, right username (ec2-user vs ubuntu), key permissions 400.
4. **Can't fix blind?** Use **SSM Session Manager** (no SSH needed at all — if the SSM agent runs and the instance has the SSM role, you get a shell through AWS). This is also the modern answer to "we don't open port 22 at all."
5. Last resorts: EC2 **Serial Console**, or detach the root EBS volume, attach to a rescue instance, fix (e.g., fstab typo, authorized_keys), reattach.
6. Check **CloudWatch**: CPU pegged at 100% or memory exhaustion can make SSH time out while the instance is "running."

**Interview answer core:** "Status checks split hardware vs OS. Then network path (SG/route/IP), then keys. Modern best practice: SSM Session Manager instead of SSH entirely — no open ports, IAM-audited access. Volume-swap rescue for a broken OS."

---

### Q30. [S] "Your AWS bill doubled this month. How do you find and fix it?"

1. **Cost Explorer** grouped by service, then by usage type, then by tag/account — narrow from 'what service' to 'which resource'.
2. Usual suspects: **NAT Gateway data processing** (chatty private workloads pulling from the internet or cross-AZ traffic — fix with VPC endpoints for S3/ECR), forgotten **GPU instances** left running, unattached **EBS volumes/snapshots** piling up, **data transfer** cross-AZ/region, over-provisioned RDS, S3 without lifecycle policies.
3. Structural fixes: tagging discipline + Cost Allocation Tags, budgets/alerts, rightsizing (Compute Optimizer), Savings Plans for steady load, **Spot** for interruptible/batch and ML training, S3 lifecycle → IA/Glacier, autoscaling so idle = small.
4. Set **AWS Budgets alarms** so next time it's a Slack alert, not a monthly surprise.

**Interview answer core:** "Cost Explorer to localize, then attack the classics — NAT data charges (VPC endpoints), idle GPUs, orphaned EBS, transfer costs — and institutionalize prevention: tags, budgets, Spot for batch, lifecycle policies."

---

### Q31. [S] "Explain S3 vs EBS vs EFS — when does each fit, especially for data/ML platforms?"

- **S3 (object):** infinite scale, 11-nines durability, HTTP API, cheapest. No filesystem semantics (no append, no POSIX). **The default home of datasets, models, artifacts, logs.**
- **EBS (block):** a virtual disk attached to **one instance in one AZ**. Low latency, high IOPS. For databases, boot volumes, scratch space.
- **EFS (file, NFS):** a shared POSIX filesystem mountable by **many instances/pods across AZs** simultaneously. For shared home directories, shared notebook/project storage (this is why platforms like Domino use EFS-style shared storage for datasets/projects). Pricier per GB, throughput scales with size or provisioned mode.

**Interview answer core:** "S3 for objects at scale (data lake, model artifacts), EBS for single-instance low-latency disk, EFS when many nodes need the same POSIX filesystem at once — the shared-storage layer for multi-user data-science platforms."

---

### Q32. [T] "CloudWatch alarm fired: RDS CPU at 100% and the app is timing out. What now?"

1. **Stabilize first:** identify and kill runaway queries — RDS **Performance Insights** shows top SQL by load; `SHOW PROCESSLIST`/`pg_stat_activity` to kill the offender.
2. **Diagnose:** new deployment (bad query/missing index)? A batch/report job overlapping peak hours? Connection storm (no pooling; each Lambda/pod opening fresh connections)?
3. **Fix classes:** add the missing index; add read replicas and route reporting/read traffic there; connection pooling (RDS Proxy is purpose-built, especially for Lambda); cache hot reads in ElastiCache; scale instance class as a last resort (money doesn't fix a missing index for long).
4. **Prevent:** slow-query log review in CI, Performance Insights baseline, alarms at 70% not 100%.

**Interview answer core:** "Performance Insights to find the top query, kill/fix it, then structural relief — indexes, read replicas, RDS Proxy pooling, caching — and only then vertical scaling. Alarm earlier next time."

---

# SECTION 4 — GIT

## The one-paragraph foundation

Git is a distributed version control system: every clone is a *full copy* of the project history. History is a chain of **commits** (snapshots, each pointing to its parent). A **branch** is just a movable pointer to a commit — cheap to create, which enables the modern workflow: branch per feature, review via pull/merge request, merge into main. Three "areas" you move changes between: **working directory** (your files) → `git add` → **staging area** (what the next commit will contain) → `git commit` → **repository** (permanent history). **Remote** (origin) = the shared copy on GitHub/GitLab; `push`/`pull` sync with it.

---

### Q33. [S] "Explain merge vs rebase, and your team policy for which to use when."

- **Merge** creates a *merge commit* joining two histories. Truthful (shows what really happened), but busy repos get a tangled graph.
- **Rebase** *replays* your commits on top of the latest main, as if you'd started from there — linear, clean history. But it **rewrites commits** (new hashes).

**The golden rule:** never rebase commits that others may have already pulled (anything pushed to a shared branch) — rewriting shared history forces everyone into conflict hell.

**Sane team policy:** rebase your *local/private* feature branch onto main to stay current (`git pull --rebase`), merge (or squash-merge) the PR into main. Squash-merge = one clean commit per feature.

**Interview answer core:** "Merge preserves true history; rebase rewrites it for linearity. Rebase private branches freely, never shared ones. Our flow: rebase the feature branch to stay current, squash-merge the PR."

---

### Q34. [T] "You have a merge conflict. What is Git actually telling you, and how do you resolve it safely?"

**Concept:** a conflict means both branches changed the **same lines** of the same file (or one deleted a file the other edited). Git refuses to guess; it stops and marks the file:

```
<<<<<<< HEAD
your branch's version
=======
their branch's version
>>>>>>> feature-x
```

**Resolution steps:**
1. `git status` — lists conflicted files.
2. Open each file, decide: yours, theirs, or a hand-written combination. Remove ALL marker lines.
3. **Understand both changes before choosing** — the #1 conflict mistake is mechanically keeping "mine" and silently deleting a colleague's bug fix.
4. `git add <file>` (marks resolved) → `git commit` (merge) or `git rebase --continue`.
5. Escape hatch: `git merge --abort` / `git rebase --abort` returns you to the pre-attempt state. Nothing is lost.
6. Verify: run the tests. A textually resolved conflict can still be *logically* wrong.

**Interview answer core:** "Conflicts = same lines changed on both sides; Git is asking a human to decide. Read both versions, resolve intentionally, add + continue, run tests — and --abort makes any attempt reversible."

---

### Q35. [T] "A teammate accidentally committed AWS keys and pushed. What's the FULL incident response?"

This tests security thinking, not just Git:
1. **Revoke the credential immediately.** Step one, always — the key is compromised the second it hit the remote (bots scan public GitHub in minutes). Rotate in IAM, check CloudTrail for misuse.
2. Only then clean history: deleting the file in a *new* commit is NOT enough — the secret lives in history. Rewrite with `git filter-repo` (or BFG), force-push, everyone re-clones. On GitHub/GitLab, old commits can persist in forks/caches — use their support tools.
3. **Prevent:** pre-commit secret scanners (gitleaks, git-secrets), push protection, secrets in Secrets Manager/CI variables, `.env` in `.gitignore`.

**Interview answer core:** "Rotate first, clean history second, prevent third. A pushed secret is compromised regardless of how fast you delete it — history rewriting is cleanup, not mitigation."

---

### Q36. [T] "You committed to the wrong branch / need to undo a commit. Which tool when: reset, revert, cherry-pick?"

- **`git revert <sha>`** — creates a *new* commit that undoes an old one. **The only safe undo for pushed/shared history.**
- **`git reset`** — moves the branch pointer backwards. `--soft` (keep changes staged), `--mixed` (keep unstaged), `--hard` (discard — destructive). Only for *local, unpushed* work.
- Wrong branch (not pushed): `git checkout right-branch && git cherry-pick <sha>`, then on the wrong branch `git reset --hard HEAD~1`.
- Lost something after a bad reset? **`git reflog`** — the log of everywhere HEAD has been; commits are recoverable for ~90 days. Knowing reflog signals real experience.

**Interview answer core:** "Revert for shared history, reset for local mistakes, cherry-pick to move commits, reflog as the safety net that makes almost nothing in Git truly lost."

---

### Q37. [S] "Design a Git branching strategy for a team deploying via CI/CD."

- **Trunk-based / GitHub Flow (modern default):** short-lived feature branches off `main`, PR + review + CI must pass, squash-merge, `main` always deployable, deploy from main. Small frequent merges = fewer conflicts.
- **GitFlow (legacy-ish):** long-lived `develop`, `release`, `hotfix` branches. Fits versioned software with scheduled releases; overkill and slow for continuous deployment.
- Non-negotiables either way: protected `main` (no direct pushes, required reviews + passing CI), meaningful commit messages, small PRs.

**Interview answer core:** "Trunk-based with short-lived branches and protected main for CI/CD shops; GitFlow only when you genuinely maintain multiple released versions in parallel."

---

### Q38. [T] "`git pull` says 'your local changes would be overwritten'. Options?"

1. **Commit** the local work, then pull.
2. **Stash**: `git stash` → `git pull` → `git stash pop`. Ideal for "not ready to commit."
3. Discard if unwanted: `git restore <file>`.

**Interview answer core:** "Commit, stash, or discard — Git refuses to silently destroy uncommitted work, which is a feature. Stash is the everyday answer."

---

# SECTION 5 — DOMINO DATA LAB (PLATFORM OPERATIONS)

## The one-paragraph foundation

**Domino** is an enterprise **data-science/MLOps platform** that runs *on Kubernetes* (EKS in AWS). Its job: self-service, reproducible compute — a scientist clicks "start a workspace" and Domino launches a **pod** running a Jupyter/RStudio/VS Code container with their chosen environment, attached to shared project files and datasets. Platform engineers keep this machinery healthy. Key concepts:

- **Workspace** — an interactive session (Jupyter etc.) = a pod on an EKS node.
- **Job** — a non-interactive batch run (train.py) = also a pod.
- **Compute Environment** — a *Docker image* + metadata defining tools/libraries. Environment builds = Docker image builds Domino performs for users.
- **Hardware Tier** — a named size (Small = 2 CPU/8 GB, GPU-large…) mapping to Kubernetes resource requests and specific node pools.
- **Domino Datasets / project files** — shared storage (typically EFS/NFS-backed) mounted into workspace pods.
- **Model API** — a trained model deployed as a REST endpoint (pods behind a service).
- Architecture split: the **platform plane** (Domino's own services — frontend, dispatcher, Keycloak auth, MongoDB, Git service) and the **compute plane** (nodes running user workloads). Platform nodes are usually tainted/dedicated.

The punchline: **almost every Domino incident is a Kubernetes/AWS incident wearing a Domino costume.** Workspace won't start = pod won't schedule. Environment build fails = Docker build fails. Slow files = EFS/NFS issue.

---

### Q39. [T] "A data scientist says: 'My workspace is stuck starting / queued.' Diagnose end to end."

Translate to Kubernetes: workspace = pod. "Queued/stuck" = Pending or failing pod.

1. **Find the pod**: `kubectl get pods -n domino-compute | grep <run-id>`; Domino's admin UI also shows run status and node.
2. `kubectl describe pod` → Events. Now it's the Pending playbook (Q12):
   - **Insufficient resources** — the Hardware Tier (say 4 CPU/16 GB) fits no node → autoscaler at node-group max? **EC2 vCPU quota** blocking GPU scale-ups (quota failures are silent — check ASG activity history)?
   - **Taint/nodeSelector mismatch** — hardware tiers map to node pools via labels; a tier pointing at a label no node group carries = eternal Pending.
   - **Image pull** — DS environment images are huge (10+ GB); slow pulls look like "stuck." ImagePullBackOff → registry auth/network (Q14).
   - **Volume attach** — EFS/NFS mounts failing: CSI driver pods healthy? EFS mount-target security group allows NFS 2049 from node SGs?
3. Pods fine but Domino still shows queued → Domino's dispatcher/orchestration services: check platform-namespace pods and logs.

**Interview answer core:** "A workspace is a pod, so describe-pod first: capacity/autoscaler (including GPU quotas), tier-to-node-pool label mismatches, giant image pulls, EFS mounts — in that order of likelihood. K8s clean? Then Domino's dispatcher services."

---

### Q40. [T] "A user's environment build keeps failing. What's happening under the hood and how do you fix it?"

**Under the hood:** a Compute Environment is a Dockerfile Domino builds in-cluster from a base image + the user's instructions, then pushes to the internal/ECR registry.

1. **Read the build log** — it's a Docker build; errors are Docker errors:
   - `pip`/conda dependency conflicts or stale pins → fix the requirements.
   - Network failures to PyPI/apt → NAT gateway, proxy, or the enterprise artifact mirror (Artifactory/Nexus) down or misconfigured index URL.
   - Base-image pull failures → registry auth/network.
   - Build **OOM/disk pressure** on the builder — big conda solves are memory hogs; resize the builder or clean node disk.
2. **Push failures** at the end → registry storage full, expired credentials, ECR permissions.
3. **Prevent:** curated centrally-maintained base environments, pinned versions, lean images (Q4 applies verbatim).

**Interview answer core:** "Environment builds are Docker builds — read the log like a Dockerfile failure: dependency resolution, network to package mirrors, base-image pulls, builder resources, registry push. My Docker playbook transfers 1:1."

---

### Q41. [T] "Users report project/dataset files loading extremely slowly. Approach?"

**Concept:** Domino project files/datasets sit on **shared NFS/EFS storage** mounted into pods. Slow = storage layer, network, or access pattern.

1. **Scope:** one user or everyone? One dataset or all storage? Everyone+everything → platform storage; one user → their workload's pattern.
2. **EFS specifics:** CloudWatch `PercentIOLimit`, throughput. Bursting-mode EFS earns throughput by stored size; a small-but-busy filesystem exhausts **burst credits** and crawls — the classic EFS incident. Fix: Elastic/Provisioned throughput.
3. **Access pattern:** NFS hates millions of tiny files and metadata-heavy ops (untarring 500k files, `ls` on giant dirs). Guidance: keep big data in **S3** and read directly; datasets for curated shared data; consolidate small files.
4. **Network:** mount-target AZ alignment (cross-AZ NFS adds latency), node network saturation.
5. Reproduce with `dd`/`fio` from a debug pod against the same mount — objective numbers.

**Interview answer core:** "Scope, then EFS burst credits (the usual culprit), then access patterns — NFS plus millions of small files is pathological; heavy data belongs in S3. I benchmark from a debug pod so we argue with numbers, not vibes."

---

### Q42. [S] "How do you run a Domino platform upgrade in production with minimal disruption?"

1. **Release notes & compatibility matrix** — Domino version ↔ Kubernetes version ↔ AMI/CSI drivers. Upgrades fail at incompatibilities.
2. **Backups first:** platform data (MongoDB, internal Git, config), volume snapshots. Know the rollback story *before* touching prod.
3. **Rehearse in staging** mirroring prod versions.
4. **Change management:** window, user comms ("running workspaces may be interrupted"), freeze on new runs.
5. Execute (Helm-driven), then **validate** with a smoke-test checklist: login (Keycloak), start a workspace, run a job, build an environment, hit a Model API.
6. Keep manifests/values in Git so rollback is `helm rollback` + restore, not archaeology.

**Interview answer core:** "Compatibility matrix, backups with a tested rollback, staging rehearsal, comms window, Helm upgrade, then scripted smoke tests of the golden paths."

---

### Q43. [T] "Nobody can log in to Domino this morning. Where do you look?"

1. **Blast radius:** all users (platform/auth outage) vs some (group/permission change).
2. Domino auth runs through **Keycloak**, usually federated to corporate **SSO/SAML/LDAP**. Check Keycloak pods and logs first.
3. **Certificate expiry** — SAML signing or TLS certs expiring is the #1 "sudden morning auth outage" in enterprises. Check dates.
4. **Upstream IdP** — Okta/AD incident? Someone rotated the SAML metadata/client secret?
5. **Dependencies:** Keycloak's database healthy? Ingress/ALB healthy (maybe the whole frontend is down, not auth)?

**Interview answer core:** "Scope, Keycloak pod health, then the two enterprise classics: expired SAML/TLS certificates and upstream IdP changes. Auth chains fail at the seams — certs, secrets, metadata."

---

### Q44. [S] "A team needs GPU workspaces. What must exist at every layer for one to run?"

Bottom-up — this shows systems thinking:
1. **AWS:** a GPU node group (g5/p4), and the **EC2 vCPU quota for that GPU family** approved (default is often 0!).
2. **Node:** GPU AMI with NVIDIA drivers; the **NVIDIA device plugin** DaemonSet so K8s advertises `nvidia.com/gpu`.
3. **Kubernetes:** taints on GPU nodes (so cheap pods don't squat on $30/hr machines) + tolerations; pod requests `nvidia.com/gpu: 1`.
4. **Domino:** a GPU **Hardware Tier** mapped to that node pool; an environment image with CUDA libraries matching the driver version.
5. **Cost:** GPU pool scales to zero when idle; idle-workspace auto-shutdown (scientists forget GPU workspaces overnight — the biggest GPU waste).

**Interview answer core:** "Quota → drivers + device plugin → taints/tolerations + gpu requests → Domino tier + CUDA-matched environment → scale-to-zero and idle shutdown. Most 'GPU broken' tickets are a zero quota, missing device plugin, or CUDA/driver mismatch."

---

### Q45. [T] "A deployed Model API is erroring / timing out. Debug it."

A Model API = pods serving HTTP behind a service. So:
1. **Pod health:** restarting (CrashLoop from a model failing to load)? **OOMKilled (137)** — models need RAM ≥ model size + overhead; very common with large models.
2. **Logs:** request-payload schema mismatches, missing features, model artifact path wrong after a rebuild.
3. **Timeouts:** inference slower than the gateway timeout — profile, add replicas, optimize/batch.
4. **What changed:** new model version deployed? Roll back to the previous version instantly, diagnose offline.
5. **Load:** traffic exceeding replica capacity — autoscaling.

**Interview answer core:** "It's pods + HTTP: OOM first (big models eat memory), logs for payload/artifact errors, latency vs timeout budget, and always ask 'what changed' — a new model version is the usual trigger; roll back fast."

---

# SECTION 6 — DATA PLATFORM OPERATIONS & SPARK

## The one-paragraph foundation

A "data platform" is the machinery that moves, stores, transforms and serves data at scale: ingestion (files, streams), a **data lake** (S3), transformation engines (**Spark**, Glue), catalogs (Glue Data Catalog / Hive metastore), query engines (Athena), orchestration (Airflow), and consumers (BI, ML). **Operations** means keeping pipelines running on schedule, data correct, storage tidy, and costs sane.

**Spark in three sentences:** Spark is a distributed compute engine — your dataset is split into **partitions**, and a cluster of **executors** processes partitions in parallel, coordinated by a **driver**. Transformations are **lazy**: nothing runs until an *action* (write, count, collect) triggers a job, which Spark compiles into **stages** separated by **shuffles** (the expensive step where data is redistributed across the network — e.g., for groupBy/join). Understanding shuffles, partitions, and memory is 90% of Spark troubleshooting.

---

### Q46. [T] "A Spark job that ran fine for months now fails with OOM / executor lost. Diagnose."

**First question: what changed?** Data volume grew, data *skew* appeared, or someone changed the code/cluster.

1. **Read the Spark UI** (the single most important skill): which stage failed? Task-level view — do a few tasks process 100x the data of the rest? That's **skew**.
2. **Data skew** — one key (e.g., customer_id = NULL, or one mega-customer) dominates a groupBy/join; the task handling that key gets a giant partition and OOMs while others finish in seconds. Fixes: filter/repair the hot key, **salting** (append a random suffix to the hot key to spread it, aggregate twice), broadcast join if one side is small, or Spark 3's **Adaptive Query Execution (AQE)** skew handling.
3. **Driver vs executor OOM** — driver OOM usually means someone did `collect()` on a big dataset (pulling everything to one machine — the cardinal sin) or huge broadcast; executor OOM means partitions too big → increase `spark.sql.shuffle.partitions`, repartition, or more executor memory.
4. **Growth** — same code, 5x data: partitions that used to be 100 MB are now 500 MB. Scale partitions with data.
5. **Executor lost** on Spot/preemptible nodes may just be an instance reclaim — check the node, not the code.

**Interview answer core:** "Spark UI first: find the failing stage and look at task-time distribution. Uniform-slow = undersized cluster/partitions; a-few-tasks-huge = skew, fixed by salting, broadcast joins, or AQE. Driver OOM = someone collect()ed. And always ask what changed — usually the data did."

---

### Q47. [S] "Explain shuffle, and why 'wide' transformations are expensive."

- **Narrow transformation** (map, filter): each output partition depends on one input partition — no data movement; stays local, fast, pipelined.
- **Wide transformation** (groupByKey, join, distinct, repartition): output partitions need data *from many* input partitions — all rows with the same key must meet on the same executor. Spark must **shuffle**: every executor writes sorted blocks to disk, then every executor fetches its share over the network. Disk + network + serialization = the expensive part of any job; each shuffle is a **stage boundary**.

**Practical implications:** minimize shuffles (filter/project *before* joins, avoid repeated groupBys), prefer **broadcast joins** when one table is small (ship the small table to every executor — zero shuffle of the big table), tune `spark.sql.shuffle.partitions` (default 200 is wrong for most jobs — too few = huge partitions/OOM, too many = tiny-task overhead; AQE can coalesce automatically).

**Interview answer core:** "Shuffle = redistributing rows so same-key rows co-locate; it hits disk, network, and serialization, and defines stage boundaries. Performance tuning is mostly shuffle management: filter early, broadcast small tables, size shuffle partitions to the data, let AQE help."

---

### Q48. [T] "The nightly ETL pipeline didn't finish and downstream dashboards are stale. Incident walkthrough."

1. **Triage & communicate:** which pipeline, since when, what's affected downstream. Tell stakeholders early ("dashboards show yesterday's data; ETA at 10:00") — comms is half of ops.
2. **Orchestrator first** (Airflow/Step Functions): which task failed or is hanging? Read *its* logs. Hanging ≠ failed — a stuck upstream dependency or a sensor waiting on a file that never arrived is common ("source system didn't deliver" is the #1 cause and isn't your bug).
3. **Classify:** transient (Spot loss, API timeout → retry), data problem (schema change from source, malformed records, unexpected NULLs), resource (job grew past cluster size), code (yesterday's deploy).
4. **Recover safely:** rerun from the failed step — which requires pipelines to be **idempotent** (rerunning produces the same result, no duplicates: overwrite partition-by-date rather than blind append). If not idempotent, clean partial outputs first.
5. **Postmortem prevention:** alerting on SLA (not just failure — *lateness*), data-quality checks at ingestion (row counts, schema validation, null thresholds — e.g., Great Expectations/Deequ), retries with backoff for transient classes.

**Interview answer core:** "Orchestrator logs → classify (source-data, transient, resource, code) → rerun idempotently from the failed step → communicate throughout → prevent with SLA alerts and data-quality gates. Idempotency is what makes 3 a.m. reruns safe."

---

### Q49. [S] "Why do millions of small files kill a data lake, and how do you fix it?"

**The problem:** every file in S3 means a separate request/listing/task. Spark schedules roughly a task per file; 5 million 50 KB files = astronomical listing time and task overhead. Query engines (Athena) crawl. This is the **small files problem**, and streaming/frequent micro-batch writers create it naturally.

**Fixes:**
1. **Compaction** — periodic jobs rewriting small files into ~128 MB–1 GB files (or use table formats below that automate it).
2. **Write smarter** — `coalesce/repartition` before writing so each partition emits one healthy file.
3. **Columnar formats** — **Parquet** over CSV/JSON: columnar (read only needed columns), compressed, with statistics enabling predicate pushdown (skip whole files/row-groups). Often 10–100x faster and cheaper in Athena (which bills per data scanned).
4. **Partition the data layout** — `s3://lake/events/date=2026-07-10/` so queries by date scan only relevant prefixes. But don't over-partition (partitioning by user_id creates… the small files problem again).
5. **Modern table formats** — Iceberg/Delta/Hudi add ACID transactions, schema evolution, time travel, and built-in compaction on top of the object store.

**Interview answer core:** "Small files multiply per-file overhead — listing, tasks, requests. Cure: compaction into 128 MB–1 GB Parquet files, date-based partitioning that matches query patterns, and table formats like Iceberg that manage this automatically."

---

### Q50. [T] "A source team changed a column's name/type and silently broke three downstream pipelines. How do you handle and prevent schema drift?"

**Handle:** identify the change (diff current vs expected schema; catalogs help), patch consumers or add a compatibility mapping, backfill any corrupted period.
**Prevent — the real answer:**
1. **Data contracts** — an agreed, versioned schema between producer and consumers; producers can't change it unilaterally.
2. **Schema validation at ingestion** — reject/quarantine records not matching the expected schema (fail loudly at the boundary rather than corrupt silently downstream).
3. **Schema evolution rules** — additive changes (new nullable columns) allowed; renames/type-narrowing require versioned migration. Table formats (Iceberg/Delta) enforce and track evolution.
4. **Alerting on drift** — automated schema-diff checks in the pipeline.

**Interview answer core:** "React: diff, patch, backfill. Prevent: data contracts, validation at ingestion that quarantines bad data, additive-only evolution rules, and schema-drift alerts. The principle is 'fail loudly at the boundary' — silent corruption downstream is far costlier than a stopped pipeline."

---

# SECTION 7 — SPARK ML

## The one-paragraph foundation

**Spark MLlib** brings machine learning to Spark's distributed engine — used when data doesn't fit on one machine. Its core abstraction is the **Pipeline**: an ordered chain of stages. Stages are either **Transformers** (take a DataFrame, return a modified DataFrame — e.g., a trained model adding a `prediction` column) or **Estimators** (have a `.fit(df)` that *learns* from data and returns a Transformer — e.g., `LogisticRegression.fit()` returns a `LogisticRegressionModel`). Features must be assembled into a single **vector column** (`VectorAssembler`) because MLlib algorithms consume one vector column, not many scalar columns.

---

### Q51. [S] "Why use Spark ML instead of scikit-learn? And when is Spark ML the wrong choice?"

- **Use Spark ML** when training data is too big for one machine's memory, or feature engineering itself requires distributed joins/aggregations over huge tables — the ML runs *where the data already is*, avoiding a giant data-movement step.
- **Wrong choice** when data fits in memory (sklearn/XGBoost on one beefy node is simpler, faster, and has richer algorithms), for deep learning (use PyTorch/TensorFlow — Spark can still do the data prep and orchestration), and for low-latency single-record serving (Spark is a batch engine; export the model or its logic for online serving).
- Common real-world pattern: **Spark for feature engineering at scale → sample/aggregate → train with sklearn/XGBoost**; or distributed training only when truly necessary.

**Interview answer core:** "Spark ML earns its complexity only when the data genuinely doesn't fit one machine or the features come from big distributed joins. Fits-in-RAM → sklearn. Deep learning → PyTorch. Serving → export the model. Don't distribute what a single node handles — distribution is a cost, not a feature."

---

### Q52. [S] "Explain the Pipeline API and why data leakage is the disaster it prevents."

**Data leakage:** information from validation/test data (or from the future) sneaking into training. Classic example: computing a scaler's mean/std on the *whole* dataset before splitting — the training process has now 'seen' test data statistics, and offline metrics look better than real performance. Models that leak look great in the lab and fail in production.

**How Pipelines prevent it:** you declare stages — `[StringIndexer, VectorAssembler, StandardScaler, LogisticRegression]` — and call `pipeline.fit(train)`. Every fittable stage learns **only from training data**; the fitted PipelineModel then applies identical transforms to validation/test/production data. One artifact contains preprocessing + model, so serving preprocessing can never drift from training preprocessing (**training/serving skew**, the other classic disaster, also prevented).

**Interview answer core:** "Pipelines bundle preprocessing + model into one fit/transform unit: fit learns statistics only on train data (preventing leakage), and the saved PipelineModel applies identical transforms in production (preventing training/serving skew). Both bugs are invisible offline and catastrophic online, which is why the abstraction exists."

---

### Q53. [T] "Distributed model training is painfully slow. Where do you look?"

1. **Spark UI again:** is time in data prep stages (shuffles from joins — Section 6 applies) or the training iterations?
2. **Caching:** iterative algorithms read the training set many times. Without `.cache()`/`persist()`, Spark **recomputes the entire input lineage every iteration** — the single most common Spark ML performance bug. Cache the final training DataFrame (and check the Storage tab confirms it actually fit in memory).
3. **Partitioning:** thousands of tiny partitions = scheduler overhead per iteration; too few = no parallelism and stragglers. Repartition to a few × total cores.
4. **CrossValidation blowup:** grid search multiplies training runs (5 folds × 20 combos = 100 fits). Use `parallelism` on the validator, shrink the grid, or use TrainValidationSplit.
5. **Cluster reality:** executors actually running? Dynamic allocation shrunk the job? Spot reclaims causing recomputation?

**Interview answer core:** "Cache the training DataFrame — iterative ML without caching recomputes the whole pipeline per iteration and is the #1 slow-training cause. Then partitions sized to cores, then the CV multiplication factor, then check the cluster is actually giving you the executors you think."

---

### Q54. [S] "How do you get a Spark-trained model into production serving?"

- **Batch scoring (most common):** load the saved PipelineModel in a scheduled Spark job, score millions of rows, write predictions to a table/S3. Spark serves this natively and well.
- **Online/real-time serving:** Spark is not a low-latency server. Options: export the model to a portable format (**ONNX**, or re-implement in a lightweight library), serve behind a REST API (e.g., a Domino Model API / MLflow serving); MLflow can wrap Spark models but per-request SparkSession overhead makes naive approaches slow.
- Either way: version the artifact (MLflow registry), and keep preprocessing inside the exported pipeline to avoid skew.

**Interview answer core:** "Batch scoring in Spark is the natural fit. For online serving, export to a portable format and serve from a proper API layer — and ship preprocessing with the model so serving matches training."

---

# SECTION 8 — MLOPS

## The one-paragraph foundation

**MLOps = DevOps discipline applied to machine learning.** The extra difficulty: software is code, but an ML system is **code + data + model**, and all three change independently. A model can break with *zero code changes* because the world's data drifted. So MLOps adds, on top of CI/CD: **experiment tracking** (what data/params/code produced this model?), a **model registry** (versioned, promotable model artifacts), **automated training pipelines**, **deployment strategies for models**, and — the part with no software equivalent — **monitoring for drift and model quality** plus **retraining loops**. MLflow is the common open-source backbone (tracking + registry); Domino provides these capabilities as a platform.

The lifecycle to narrate in interviews: data prep → experiment (tracked) → register model → validate → deploy (gradually) → monitor (drift/performance) → retrain → repeat.

---

### Q55. [S] "A data scientist emails you 'model_final_v2_REAL.pkl' and asks you to deploy it. What's wrong with this picture, and what does the right process look like?"

**What's wrong:** no lineage (what code/data/params produced it?), no reproducibility (can't rebuild it), no versioning, no validation gate, no rollback story, unknown dependencies (which sklearn version pickled it? — pickles are environment-sensitive), and a manual, unauditable path to prod.

**The right process:**
1. Training runs inside a **tracked experiment** (MLflow): params, metrics, data version, code commit, environment logged automatically.
2. The artifact is **registered** in a model registry with a version and stage (staging → production).
3. **Automated validation** before promotion: accuracy above baseline on a held-out set, latency budget, input/output schema checks, bias checks if applicable.
4. **CI/CD deploys from the registry** — never from an email. The serving image is built with pinned dependencies matching training.
5. Rollback = repoint to the previous registry version.

**Interview answer core:** "A pickle in an email has no lineage, no reproducibility, no gate, no rollback. The fix is registry-driven deployment: tracked experiment → registered version → automated validation → CI/CD promotion → one-click rollback. My job as platform engineer is making that path easier than emailing pickles."

---

### Q56. [S] "Explain model drift. How do you detect it and what do you do about it?"

**The concept:** models learn patterns from a snapshot of the world; the world moves.
- **Data drift (covariate shift):** input distribution changes — new user demographics, sensor recalibrations, a pandemic changing behavior. Inputs no longer resemble training data.
- **Concept drift:** the input→output relationship itself changes — fraud tactics evolve, so yesterday's fraud signature no longer predicts today's fraud even if inputs look similar.

**Detection:**
1. **Ground truth lag problem:** you often learn the true label late or never (was that loan a default? — takes months). So monitor in layers:
2. **Input distribution monitoring** — compare live feature distributions vs training baseline (PSI, KL divergence, KS tests); alert on shift. Works *without* labels — your early-warning system.
3. **Prediction distribution monitoring** — a fraud model suddenly flagging 5x more cases is a red flag even before labels arrive.
4. **Delayed performance metrics** — when labels land, compute accuracy/precision on recent windows vs launch benchmarks.

**Response:** investigate (real-world change vs data pipeline bug — a broken upstream feature *looks* like drift!), retrain on recent data, possibly re-engineer features; if severe, fall back to a simpler robust model or human review.

**Interview answer core:** "Data drift = inputs changed; concept drift = the relationship changed. Because labels lag, monitor inputs and prediction distributions as early warnings, then delayed metrics for confirmation. And always rule out a broken pipeline first — a nulled-out upstream feature is drift's most common impersonator."

---

### Q57. [S] "How do you deploy a new model version safely? Compare the strategies."

The fear: offline metrics looked better, but production behavior disappoints (leakage, skew, unseen data). So expose it gradually:
1. **Shadow deployment** — new model receives *copies* of live traffic and logs predictions, but the old model's answers are served. Zero user risk; compares behavior on real data. Best first step for high-stakes models.
2. **Canary** — route 5% of traffic to the new version, watch metrics (latency, error rate, business KPIs), ramp 5→25→50→100%. Automatic rollback on regression.
3. **Blue/green** — two full environments; instant traffic switch and instant rollback; costs double capacity briefly.
4. **A/B test** — statistically rigorous split when the question is business impact (revenue, engagement), not just correctness.

**Interview answer core:** "Shadow to compare risk-free, canary to ramp with automatic rollback, blue/green for instant switchover, A/B when measuring business impact. The shared principle: never trust offline metrics alone; production data is the only real test set."

---

### Q58. [T] "The model's production accuracy is way below its offline evaluation. Root-cause it."

The four classic causes, in the order to check:
1. **Training/serving skew** — preprocessing differs between training and serving (different code paths, library versions, a feature computed slightly differently online vs in the batch pipeline). Detect: log served features, replay through the offline pipeline, diff. Fix: share one preprocessing artifact (pipeline bundled with model — Q52).
2. **Data leakage in training** — the offline number was never real (a feature that encodes the label, future information, test contamination). Suspect this when offline results looked "too good."
3. **Broken feature pipeline in prod** — a feature silently nulled/defaulted; the model runs but on garbage. Feature-level monitoring catches it.
4. **Distribution mismatch** — evaluation set didn't represent production traffic (trained/evaluated on last year, serving this year; or evaluation on clean data, production is messy).

**Interview answer core:** "Skew, leakage, broken features, unrepresentative eval — in that order. First move: log real serving inputs and replay them offline; if offline predictions on the same inputs differ from what was served, it's skew; if they match and are still bad, the offline evaluation itself was flawed."

---

### Q59. [S] "Design the CI/CD pipeline for an ML system. How does it differ from app CI/CD?"

**Three pipelines, not one:**
1. **CI on code** (as in software): lint, unit tests on feature/training code, build training + serving images. Plus ML extras: tests on *data expectations* and a fast smoke-train on sample data.
2. **CT — Continuous Training** (the ML-specific one): an automated pipeline that, on trigger (schedule, drift alert, new data), runs data validation → feature engineering → training → evaluation against the current champion → auto-registers the candidate if it wins.
3. **CD on the model**: promotion from registry through staging (integration tests: schema, latency, load) to production via canary/shadow (Q57), with monitoring wired in as the last gate.

**Key differences from app CI/CD:** artifacts include data and models (large, versioned separately — DVC/lakeFS/registries), tests include statistical quality gates (not just pass/fail), triggers include *data events* not just commits, and "rollback" may mean re-serving an old model while retraining.

**Interview answer core:** "CI for code, CT for models, CD for serving. The novel piece is continuous training triggered by data and drift, with an automated champion/challenger gate. And version everything — code in Git, data with DVC or a table format, models in a registry — because reproducing any prod model on demand is the acid test of MLOps maturity."

---

### Q60. [S] "What is a feature store and what problems justify it?"

**Problems:** (1) every team recomputes the same features slightly differently — wasted work and inconsistent definitions; (2) **training/serving skew** — features computed one way in the batch training pipeline and another way in the online service; (3) **point-in-time correctness** — training rows must use feature values *as they were at that moment*, not today's values (using today's values leaks the future).

**What it is:** a central system with a **definitions layer** (feature computed once, one definition), an **offline store** (historical values for training, with point-in-time joins) and an **online store** (low-latency current values for serving) — both fed from the same definitions, killing skew by construction. Examples: Feast, SageMaker Feature Store, Databricks/Tecton.

**Interview answer core:** "A feature store centralizes feature definitions and serves them consistently to both training (offline, point-in-time correct) and inference (online, low-latency). It's justified when multiple models share features or online serving exists — it eliminates skew and future-leakage structurally rather than by discipline."

---

### Q61. [T] "Training pipeline succeeds but produces a garbage model (metrics collapsed vs last run). Investigate."

Pipeline green ≠ model good. Steps:
1. **Diff the runs** — this is why experiment tracking exists: compare params, code commit, data version, environment between last-good and current run. Something changed.
2. **Data first:** row counts, class balance, feature distributions vs previous training set. Upstream changes (a source dropped, a join now produces nulls, dedup logic changed) are the most common cause of silent model collapse.
3. **Label sanity:** label pipeline broken? Labels shifted/renamed? A flipped label mapping produces exactly "suddenly terrible metrics."
4. **Environment:** library version bumps changing defaults (a new sklearn/Spark version altering a solver default is a real, recurring incident).
5. **Prevent:** data-validation gates *inside* the training pipeline (expectations on inputs), and a promotion gate that refuses to register a model underperforming the current champion — so garbage can't reach prod even when it's produced.

**Interview answer core:** "Diff against the last good run via tracking metadata — data version, code, params, environment. It's usually the data: counts, distributions, and especially labels. And the registry gate means a bad model is an investigation, never an outage."

---

### Q62. [S] "As a platform engineer, what does 'supporting MLOps' actually mean day to day? (Positioning answer)"

This is the answer that ties YOUR profile together:
- Data scientists own models and metrics; **you own the paved road**: the Kubernetes/EKS platform their workloads run on, the compute environments (images), CI/CD plumbing, tracking/registry services (MLflow/Domino) kept available and backed up, GPU capacity and quotas, storage (EFS/S3) performance, secrets/IAM (IRSA), cost controls, and the monitoring/alerting substrate.
- Their incidents land on you translated: "my experiment is stuck" = Pending pod; "training is slow" = storage or scheduling; "deploy failed" = image/CI/IAM. Everything in Sections 1–5 IS MLOps support.

**Interview answer core:** "MLOps support means running the platform that makes the ML lifecycle self-service and reliable: compute orchestration, environments, tracking and registry services, storage, security, and cost. I don't need to build the model to keep the entire path from notebook to production endpoint fast and safe — that path is Kubernetes, AWS, Docker, and CI/CD underneath, which is exactly my ground."

---

# APPENDIX — HOW TO HANDLE "HAVE YOU DONE THIS HANDS-ON?"

Honest positioning scripts (adapt, don't recite):
- **Bridge from adjacent experience:** "I haven't run Spark clusters in production specifically, but I've operated the layer Spark runs on — Kubernetes and EC2 fleets — and debugged the same failure classes: OOM kills, skewed load, storage bottlenecks. The Spark UI is a new dashboard over failure modes I know well."
- **Show the learning loop:** "When I ramped on [Domino/EKS], my method was: architecture first, then break-fix labs in my own AWS account, then shadowing tickets. I'd apply the same ramp here — I'm usually productive on a new platform within weeks because the Linux/K8s/AWS foundation transfers."
- **Never bluff specifics.** If asked for a war story you don't have: "I haven't hit that exact incident; here's exactly how I'd approach it —" then give the structured diagnosis from this guide. Interviewers rate a crisp method above a vague anecdote.

---

## FINAL REVISION CHECKLIST (30 minutes before the interview)
1. Reconciliation loop, and the kubectl-apply end-to-end story (Q11).
2. Pending vs CrashLoopBackOff vs ImagePullBackOff — causes cold (Q12–14).
3. `kubectl get endpoints` as the service-debugging opener (Q15).
4. Exit code 137 = OOM. Everywhere. Docker, K8s, Domino, model APIs.
5. Packet-tracing order: route table → NAT → SG → NACL → DNS (Q23).
6. IAM evaluation: explicit deny wins; ListBucket vs GetObject; kms:Decrypt (Q25).
7. Rebase private, merge shared; rotate leaked secrets before cleaning history (Q33, Q35).
8. Domino = K8s in costume: workspace = pod, environment = Docker image, tier = requests + node pool (Section 5).
9. Spark: shuffle is the cost, skew is the villain, cache before iterating, never collect() big data (Q46–47, Q53).
10. MLOps: registry-driven deploys, drift monitoring without labels, shadow → canary, training/serving skew as the #1 prod-accuracy killer (Q55–58).
