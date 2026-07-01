# Domino Data Lab Platform Support — Interview Prep Guide
### Senior Platform Support / SRE Role · Interview Tomorrow

---

## PART 0: Your Strategy (Read This First)

**Your honest position:** ~7 years with Linux, Docker, Kubernetes, and AWS. No hands-on Domino Data Lab. No SageMaker/Azure ML depth. No pharma/GxP background.

**Why you're still a strong candidate:** Domino Data Lab is a Kubernetes-native application. Every Domino workspace is a Kubernetes pod. Every Domino job is a Kubernetes job. Every Domino environment is a Docker image. When Domino breaks, you troubleshoot it with `kubectl`, container logs, and Linux fundamentals — exactly your skill set. The interviewer is really hiring a Kubernetes/platform person who can learn Domino's layer on top.

**The framing sentence to use (memorize this):**
> "I haven't administered Domino directly, but Domino is a Kubernetes-native platform — workspaces are pods, jobs are Kubernetes jobs, environments are Docker images. My 7 years with Kubernetes, Docker, and AWS means I already troubleshoot the layer Domino runs on. The Domino-specific concepts — environments, workspaces, model APIs — I've studied, and they map directly onto primitives I use daily."

**Rules for tomorrow:**
1. Never bluff Domino specifics. Say "I haven't done that in Domino specifically, but here's how I'd approach it based on how Domino works on Kubernetes..." then reason out loud.
2. Anchor every answer in what you HAVE done. Translate their question into K8s/Docker/AWS terms.
3. For scenario questions, use the troubleshooting framework in Part 4. Interviewers at senior level grade your *method*, not memorized fixes.
4. Quantify where possible ("cluster of X nodes", "reduced MTTR from X to Y", "handled X incidents/month").

---

## PART 1: Domino Data Lab Crash Course (Zero to Interview-Ready)

### What is Domino?
Domino Data Lab is an **enterprise MLOps / data science platform**. Think of it as a self-service portal where data scientists get on-demand compute, reproducible environments, and ML lifecycle tools — without touching Kubernetes themselves. Heavily used in **pharma/life sciences** (hence the GxP requirement in this JD) because it provides **reproducibility and audit trails** that regulators demand.

### Core Concepts (learn these terms cold)

| Domino Term | What it actually is | Your mental model |
|---|---|---|
| **Workspace** | Interactive dev session (JupyterLab, RStudio, VS Code) | A Kubernetes pod running an IDE container |
| **Job** | Batch/non-interactive script execution | A Kubernetes job/pod that runs to completion |
| **Environment (Compute Environment)** | The runtime definition: base image + packages | A Docker image built from a Dockerfile |
| **Project** | Container for code, data, environments, results | A Git repo + metadata + collaborators |
| **Model API** | A trained model deployed as a REST endpoint | A long-running pod behind a service/ingress |
| **App** | Hosted Dash/Shiny/Streamlit web app | Another long-running pod with a route |
| **Hardware Tier** | Named CPU/GPU/memory size (small/large/GPU) | Pod resource requests/limits + node selector |
| **Data Source** | Managed connection to S3, Snowflake, Redshift, etc. | Stored credentials + client libraries injected into pods |
| **Domino Datasets** | Shared, versioned network storage | NFS/EFS-backed PersistentVolumes mounted into pods |
| **Launcher** | Parameterized self-service job template | A form that kicks off a job with arguments |

### Architecture (this is where you shine)
Domino runs **on Kubernetes** (EKS on AWS, AKS on Azure, or on-prem). Two logical "planes":

- **Control plane (Domino Platform):** frontend/API server, Keycloak (authentication/SSO), MongoDB (metadata), Postgres, RabbitMQ (queuing), the dispatcher/orchestrator that turns user requests into Kubernetes objects, Docker registry for environment images, Prometheus/Grafana for monitoring. These run as pods in a platform namespace (commonly `domino-platform`).
- **Compute plane:** the nodes where user workloads (workspaces, jobs, model APIs) run as pods, typically in a `domino-compute` namespace, often on separate node pools (including GPU node pools). Autoscaling via cluster autoscaler.

**Storage:** Blob storage (S3) for project files/artifacts, shared filesystem (EFS/NFS) for Domino Datasets, EBS for pod scratch space.

**The flow when a user clicks "Start Workspace":**
1. Frontend request → Domino API → dispatcher
2. Dispatcher creates a Kubernetes pod spec: the environment's Docker image, resources from the hardware tier, volume mounts for project files/datasets
3. Scheduler places the pod on a compute node (autoscaler adds a node if needed)
4. Image pulled from registry → containers start → user's project files sync in → IDE becomes reachable through Domino's ingress/proxy

If you can narrate that flow, you can troubleshoot every stage of it — and that's 80% of this job.

### Why pharma loves Domino (GxP tie-in)
- **Reproducibility:** every run records the exact environment (image), code version, and data — you can re-execute an analysis from 3 years ago identically. Critical for regulatory submissions.
- **Audit trail:** who ran what, when, with which data — supports GxP/21 CFR Part 11 expectations.
- **Governed access:** SSO, RBAC on projects, controlled data source credentials.

---

## PART 2: Concept Questions & Answers

### A. Domino-Specific

**Q1. What is Domino Data Lab and why do companies use it?**
> "It's an enterprise data science platform that gives data scientists self-service, reproducible compute on top of Kubernetes. Instead of fighting for shared servers or managing their own Python/R environments, users launch workspaces or jobs with a defined environment and hardware tier. For life sciences it's especially valuable because it captures full lineage — code version, environment image, data — which supports GxP reproducibility and audit requirements."

**Q2. Explain Domino's architecture.**
> Give the control plane / compute plane explanation from Part 1. Mention: runs on Kubernetes (EKS/AKS), platform services include Keycloak for auth, MongoDB for metadata, RabbitMQ for queuing, an internal Docker registry for environments; user workloads run as pods in the compute namespace; storage split between S3 blob storage, EFS/NFS datasets, and EBS scratch.

**Q3. What is a Compute Environment in Domino?**
> "It's Domino's abstraction over a Docker image. An environment defines the base image plus a Dockerfile-like instruction set for extra packages, plus pluggable workspace IDEs. When an admin edits an environment, Domino builds and versions a new image in its registry. Every workspace or job pins the environment revision it used — that's how reproducibility is guaranteed. As a platform admin I'd own curating base environments, keeping CUDA/driver versions aligned with GPU nodes, and controlling image sprawl."

**Q4. Difference between a Workspace and a Job?**
> "A workspace is interactive — a pod running JupyterLab/RStudio/VS Code that a user works in live. A job is non-interactive batch execution of a script that runs to completion and records results. Operationally, workspaces are long-lived and idle-prone (cost concern), jobs are ephemeral. Both are pods with the environment image and hardware tier resources."

**Q5. What is a Hardware Tier?**
> "A named preset of CPU, memory, and GPU that maps to Kubernetes resource requests/limits and node selectors. Admins define tiers like 'small', 'large-memory', 'gpu-v100' so users pick capacity without knowing Kubernetes. Tuning tiers to actual node instance types matters — a tier that doesn't bin-pack well onto node sizes wastes money or causes unschedulable pods."

**Q6. How does Domino handle model deployment?**
> "A trained model can be published as a Model API — Domino wraps the code and environment into a container serving REST requests, deployed as replicated pods behind Domino's routing layer, with versioning so you can promote or roll back. There are also Apps for dashboards (Streamlit/Dash/Shiny). Domino can additionally export models to external hosts like SageMaker."

**Q7. What are Domino Datasets vs Project Files?**
> "Project files are the code/artifacts synced per project, backed by blob storage like S3. Datasets are large shared, versioned data on network storage (EFS/NFS) mounted into runs — better for big data you don't want copied per-run, and shareable across projects with permissions."

**Q8. How would you upgrade Domino?**
> "Domino upgrades are effectively Kubernetes application upgrades. My approach: read release notes for breaking changes and prerequisite versions (including required Kubernetes version), back up the metadata stores (MongoDB, Postgres) and verify blob storage state, take a change-management window through the ITSM process since pharma requires controlled changes, run the upgrade in a lower environment first, validate with a smoke-test checklist — start workspace, run job, call a model API, test each data source type — then upgrade production, with a documented rollback plan. Post-upgrade, watch platform pod health and error rates in Prometheus/Grafana."

### B. Kubernetes (your strength — expect these to go deep)

**Q9. Walk me through what happens when a pod is created.**
> "API server validates and persists the object to etcd; scheduler assigns it to a node based on resource requests, affinity, taints/tolerations; kubelet on that node pulls the image, sets up the pod sandbox and networking via CNI, mounts volumes, and starts containers; readiness probes gate traffic. In Domino terms this is exactly what happens when a user starts a workspace."

**Q10. Pod stuck in Pending — what do you check?**
> "`kubectl describe pod` first — the events section usually says it: insufficient CPU/memory, no node matching the selector/taint, unbound PVC, or cluster autoscaler at max. In Domino this is the classic 'my workspace won't start' ticket — often a GPU tier with no GPU node available, or the autoscaler hitting an AWS instance quota."

**Q11. CrashLoopBackOff vs ImagePullBackOff?**
> "ImagePullBackOff: kubelet can't fetch the image — bad tag, registry auth, or network to the registry; in Domino, often a broken environment build or registry credential issue. CrashLoopBackOff: container starts and dies repeatedly — check `kubectl logs --previous` for the exit reason; commonly OOMKilled (exit 137), bad entrypoint, or missing config. OOMKilled in Domino means the user needs a bigger hardware tier or their code has a memory issue."

**Q12. Requests vs limits, and why they matter here.**
> "Requests drive scheduling and capacity math; limits enforce a ceiling (memory limit breach = OOMKill; CPU limit = throttling). Domino hardware tiers are basically request/limit presets. If requests are set far below real usage you get noisy-neighbor node pressure and evictions; too high and utilization craters. Tier tuning is a real cost/reliability lever."

**Q13. Explain taints/tolerations and node affinity in Domino's context.**
> "GPU nodes are tainted so ordinary pods don't land there; GPU hardware tiers add the toleration and a node selector. Platform nodes are often tainted too so user workloads can't disturb the control plane services."

**Q14. How do you debug a service that's unreachable inside the cluster?**
> "Confirm endpoints exist (`kubectl get endpoints`) — no endpoints means label selector mismatch or pods failing readiness. Then DNS resolution from a debug pod, then network policies, then the ingress/route layer. In Domino, 'workspace started but I can't connect' typically lives in the proxy/ingress path."

**Q15. How does the cluster autoscaler work?**
> "It watches for unschedulable pods and scales node groups up; scales down nodes that are underutilized and drainable. Failure modes worth mentioning: cloud quota limits, GPU instance scarcity in a region, pods with local storage or restrictive PodDisruptionBudgets blocking scale-down."

### C. Docker

**Q16. Dockerfile best practices for data science environments.**
> "Pin base image and package versions for reproducibility (critical in GxP), order layers so the slow rarely-changing installs (system libs, CUDA) come first for cache efficiency, keep images lean, one concern per image, and scan for vulnerabilities. In Domino, environment definitions are Dockerfile instructions — the same principles apply, and unpinned packages are the number one cause of 'my code worked last month' tickets."

**Q17. Image vs container? Layers?**
> "Image is the immutable, layered template; container is a running instance with a writable layer on top. Layers are content-addressed and shared, which is why base-image reuse saves pull time and disk — relevant when a Domino environment image is 10 GB and node disk pressure starts evicting pods."

### D. Git / GitHub / GitLab / Bitbucket

**Q18. How does Git fit into a data science platform?**
> "Domino projects can be Git-based — code lives in GitHub/GitLab/Bitbucket and syncs into workspaces. Platform-side issues I'd support: service credentials/deploy keys, SSH vs HTTPS auth, webhook integrations, and users hitting merge conflicts or large-file problems (steer them to Datasets instead of committing data). I'd also speak to branching strategy and PR review as change control for analytical code."

**Q19. Merge vs rebase? Resolving conflicts?**
> "Merge preserves history with a merge commit; rebase replays commits for a linear history — never rebase shared branches. Conflicts: pull latest, resolve markers, test, commit. Keep it brief unless probed."

### E. AWS / SageMaker / Azure ML

**Q20. What AWS services underpin a Domino deployment?**
> "EKS for the cluster, EC2 node groups (including GPU instances like g4dn/p3), S3 for blob storage, EFS for datasets, EBS for volumes, ECR or Domino's internal registry for images, IAM/IRSA for pod-level AWS permissions, ALB/NLB for ingress, Route53, CloudWatch. Cost levers: autoscaling aggressiveness, spot for interruptible workloads, idle-workspace shutdown."

**Q21. What is SageMaker and how does it compare to Domino?**
> "SageMaker is AWS's managed ML platform — notebook instances/Studio, managed training jobs, hyperparameter tuning, and managed inference endpoints. Domino is cloud-agnostic and collaboration/governance-centric; SageMaker is deeply AWS-native. They coexist: Domino can export models to SageMaker endpoints for serving. I'm strongest on the AWS infrastructure under both; my SageMaker knowledge is conceptual and I'd ramp quickly hands-on." *(Honest, and shows you know the boundary of your knowledge.)*

**Q22. Azure ML equivalent concepts?**
> "Azure ML workspaces, compute instances/clusters, environments (also Docker-based), managed online endpoints for inference, model registry. Conceptually parallel to SageMaker. Same honest caveat as above."

### F. GxP / ITSM / Change Management (memorize — this is their culture)

**Q23. What is GxP and how does it affect platform operations?**
> "GxP is the family of 'good practice' regulations in life sciences — GLP, GCP, GMP — governing quality and traceability of regulated work. For a platform team it means: validated systems (documented evidence the platform does what it's specified to do — IQ/OQ/PQ qualification), strict change control (no ad-hoc production changes; everything through approved change requests), audit trails, access control, and data integrity per ALCOA+ principles (data should be Attributable, Legible, Contemporaneous, Original, Accurate). Practically: an upgrade I could do in an hour at a tech company becomes a planned, documented, approved, validated change here — and that's appropriate given patient-impacting research runs on the platform."

**Q24. Walk me through your change management process.**
> "Raise a change request in the ITSM tool (ServiceNow/Jira Service Management): description, risk assessment, rollback plan, validation/test evidence, maintenance window. CAB or approver sign-off for normal changes; standard changes pre-approved for routine low-risk work; emergency changes with retrospective approval for incident fixes. Execute in window, validate, close with evidence. I'd also distinguish incident vs problem vs change tickets — RCA output feeds problem records which drive permanent-fix changes."

**Q25. Incident severity and SLAs?**
> "P1 platform-down (all users blocked) — immediate response, incident bridge, comms cadence; P2 major degradation (a capability like model APIs down); P3 single-user issues; P4 requests. Postmortem/RCA for P1/P2, blameless, with tracked corrective actions."

---

## PART 3: The Universal Troubleshooting Framework (Use for EVERY scenario question)

When asked any "X is broken, what do you do?" — structure your answer with these steps out loud. Senior candidates are graded on method.

1. **Scope & impact:** One user or all? One project or platform-wide? When did it start? What changed? (Recent deploy/upgrade? — check change records.)
2. **Reproduce/observe:** Get the run ID / workspace ID, look at it from the platform side.
3. **Follow the request path layer by layer:** UI/API → dispatcher → Kubernetes scheduling → image pull → container start → networking/storage → application code.
4. **Standard tooling:** `kubectl get/describe pod`, `kubectl logs` (and `--previous`), events, node status, Prometheus/Grafana dashboards, platform service logs.
5. **Mitigate first, then fix:** restore service (restart, reroute, scale, rollback), then RCA.
6. **RCA & permanent fix:** 5 Whys / timeline reconstruction; fix root cause via change management; add monitoring/automation so it's caught earlier next time; document in KB.

Ending answers with "…and then I'd add an alert/automation so we catch this before users do" repeatedly = senior signal, and matches the JD's "drive automation and monitoring improvements" line.

---

## PART 4: Scenario Questions & Model Answers

**S1. "A user's workspace is stuck in 'Starting' / won't launch. Walk me through it."**
> "First scope: just this user or many? If many, suspect platform-level — check dispatcher and platform pod health. For one user: find the workspace's pod in the compute namespace and `kubectl describe` it.
> - **Pod Pending** → scheduling: insufficient resources for the hardware tier, GPU nodes unavailable, autoscaler at max or hitting an EC2 quota, or a taint/selector mismatch. Fix: free capacity, raise node group max, request quota increase, or have the user pick an available tier.
> - **ImagePullBackOff** → the environment image: recent environment edit broke the build, registry auth, or a huge image timing out on pull. Check the environment build logs.
> - **CrashLoopBackOff** → `kubectl logs --previous`: OOMKilled means the tier is too small for their startup code; otherwise a broken environment (bad package install, entrypoint failure).
> - **Pod Running but UI never connects** → readiness probe, or the proxy/ingress route to the workspace.
> Mitigate for the user (working tier/environment), then RCA — e.g., if an environment change broke everyone's workspaces, roll the environment back to the previous revision, and add a pre-publish validation step so environment edits get smoke-tested."

**S2. "Jobs are queued and not executing."**
> "Queued means the dispatcher hasn't successfully turned them into running pods. Check: (1) compute capacity — are nodes full, is the autoscaler scaling? `kubectl get nodes`, pending pods, autoscaler logs, cloud quotas. (2) Platform side — dispatcher pod healthy? RabbitMQ (Domino's queue) backed up or down? MongoDB healthy? (3) Namespace quotas or a stuck finalizer flood of old pods. If it started after a change, correlate with the change record. Mitigation might be scaling the node group manually while fixing the autoscaler; permanent fix could be right-sizing capacity and alerting on queue depth."

**S3. "A Model API endpoint is returning errors / down. Business-critical."**
> "This is a P1/P2 — restore first. Check the model pods: crashed (logs, OOM), failing health checks, or fine but the routing layer is broken? If a new model version was just promoted, **roll back to the previous version immediately** — Domino versions models exactly for this. If pods are OOMKilled, bump resources. If it's dependency/data drift causing 500s inside the model code, engage the model owner while keeping the old version serving. Afterward: RCA, add latency/error-rate alerts and a canary or staged promotion process so bad versions never take 100% of traffic."

**S4. "Users report the platform is slow."**
> "'Slow' needs decomposition: slow UI, slow workspace startup, or slow code execution?
> - **UI slow for everyone** → platform services: frontend/API pod CPU, MongoDB performance, load balancer. Grafana first.
> - **Workspace startup slow** → usually image pull time (bloated environments) or autoscaler node spin-up latency; fixes include slimming images, image pre-pull/caching on nodes, warm capacity.
> - **User code slow** → their pod's CPU throttling (limits), disk I/O on EFS/NFS (datasets on network storage are a classic bottleneck — check EFS throughput mode/burst credits on AWS), or noisy neighbors on the node.
> I'd use the USE method — utilization, saturation, errors — across CPU, memory, disk, network at node and pod level to localize it."

**S5. "A data source connection (e.g., Snowflake/S3) suddenly fails for a project."**
> "Scope: one project or all? All → shared credential expired (rotated service account/password/key), network path change (security group, firewall, proxy), or DNS. One project → that project's credentials or permissions changed on the data side. Test from inside a compute pod (same network path as user workloads) — a quick connectivity check isolates network vs auth vs data-platform-side. Classic root causes: expired credentials nobody tracked, a security-group change from another team, IAM policy change. Permanent fix: credential expiry monitoring/rotation automation, and synthetic connectivity checks per data source that alert before users notice."

**S6. "You upgraded Domino last night; this morning nothing works. Go."**
> "Rollback decision comes first: per the change plan, if impact is platform-wide and no quick fix is evident within the agreed window, execute the rollback (restore metadata DB backups, redeploy prior version). In parallel, diagnose: `kubectl get pods -n domino-platform` — which services are unhealthy? Common upgrade failures: schema migration errors, image versions incompatible with the Kubernetes version, changed configuration values, certificate issues. Whatever happened, the RCA must answer 'why didn't lower-environment testing catch this?' and the corrective action improves the pre-prod validation checklist. In a GxP shop, this also means an incident record, deviation documentation, and a formally tracked CAPA."

**S7. "Disk pressure / nodes evicting pods."**
> "Node disk fills from image bloat (many large environment revisions cached), container logs, or pod scratch usage. `kubectl describe node` shows DiskPressure; kubelet garbage-collects images and evicts pods. Fixes: bigger node volumes, image cleanup policies, limiting environment image sprawl, log rotation, and alerting on node disk before eviction thresholds."

**S8. "A user says results differ from last month with the same code."**
> "Reproducibility question — Domino's home turf. Compare the two runs: environment revision (did packages update?), code commit, data version, hardware (CPU vs GPU nondeterminism). Most often an unpinned package changed in a rebuilt environment. Fix: pin versions in environments; policy that regulated projects pin environment revisions. In GxP context this matters enormously, which is why Domino records all of this per run."

**S9. "How would you reduce platform costs?"**
> "Idle workspace auto-shutdown (biggest quick win — interactive sessions left running overnight), right-size hardware tiers against actual utilization from Prometheus, autoscaler tuning and scale-to-zero for GPU node groups, spot instances for fault-tolerant batch jobs, image slimming to cut pull time and disk, and storage lifecycle policies on S3/EFS. I'd start by measuring: utilization dashboards per tier and per team."

**S10. "How do you approach RCA? Give the format."**
> "Blameless, evidence-based. Reconstruct a timeline from logs/metrics/change records; identify trigger vs root cause vs contributing factors; 5 Whys to get below the surface cause ('pod OOMKilled' is not a root cause — 'no memory alerting and tier defaults never reviewed' is closer); define corrective actions with owners and dates — a permanent fix, plus detection improvement, plus prevention (automation, validation gates). Document in the problem record and knowledge base. The JD explicitly says 'permanent fixes' — the anti-pattern is restart-and-close."

---

## PART 5: Project / Experience Questions (STAR answers from YOUR background)

They will ask "tell me about a time…" — prepare 4–5 stories from your actual Linux/Docker/K8s/AWS work. Templates below; fill the [brackets] with your real details tonight and say them out loud twice.

**P1. "Tell me about a complex production issue you resolved." (RCA story)**
> S: "In my current role we run [N] services on Kubernetes/[EKS] serving [users/scale]."
> T: "We had [incident — e.g., pods OOMKilling under load / intermittent 5xx / node failures] impacting [who]."
> A: "I scoped impact, checked recent changes, used kubectl/describe/logs and [Grafana/CloudWatch] to trace it to [root cause]. Mitigated by [rollback/scale/config], then drove the permanent fix: [what], plus added [alert/automation]."
> R: "Restored in [time]; recurrence dropped to zero; MTTR for that class of issue improved by [X]."

**P2. "Tell me about automation you built."** (JD: "drive automation and monitoring improvements")
> Pick your best: CI/CD pipeline, autoscaling setup, scripted patching, Terraform/IaC, self-healing cron. Emphasize toil eliminated in hours/week and errors prevented.

**P3. "Tell me about an upgrade or migration you led."**
> Kubernetes version upgrade, Docker base-image refresh, AWS migration — emphasize planning, testing in lower env, rollback plan, communication. This maps directly onto "manage platform upgrades and patching."

**P4. "Tell me about supporting non-infrastructure users."**
> Data scientists are your customers here. Any story of supporting developers/analysts — translating their 'it's broken' into a technical diagnosis, being patient, writing docs/runbooks — signals platform-support fit.

**P5. "A time you disagreed with a decision / pushed back."**
> Have one ready (change made without testing, risky shortcut). Show you escalated professionally with data — pairs well with the GxP change-discipline culture.

**Handling "Do you have Domino experience?" directly:**
> Use the framing sentence from Part 0. Then add: "In my first 30 days I'd get Domino administrator training material, shadow existing tickets, and build a runbook mapping each common Domino failure to its Kubernetes-level diagnosis — most of that mapping I can already draft today." (Offering a 30-day plan unprompted is a strong senior move.)

**Handling "You have 6 years 8 months; we asked for 7–10."**
> "I'm effectively at 7 years, and the depth matters more than the number: those years are concentrated in exactly the stack Domino runs on — Kubernetes, Docker, AWS, Linux. The Domino-specific layer is learnable in weeks; the Kubernetes depth under it takes years, and I already have it."

---

## PART 6: Rapid-Fire Facts & Terms (skim 30 min before interview)

- Domino platform namespace ≈ `domino-platform`; user workloads ≈ `domino-compute`
- Auth: **Keycloak** (SSO/SAML/OIDC) · Metadata: **MongoDB** · Queue: **RabbitMQ** · Monitoring: **Prometheus + Grafana**
- Storage: **S3** (blob/project files), **EFS/NFS** (Datasets), **EBS** (pod volumes)
- Runs on **EKS / AKS / GKE / on-prem Kubernetes**
- Environment = versioned **Docker image**; every run pins its environment **revision** → reproducibility
- Workspace IDEs: **JupyterLab, RStudio, VS Code**
- Model API = model served as **REST endpoint**, versioned, roll-back-able; can export to **SageMaker**
- GxP: validated systems, **IQ/OQ/PQ**, change control, audit trails, **ALCOA+** data integrity, CAPA
- ITSM: incident vs **problem** vs change; **CAB**; standard/normal/emergency changes; ServiceNow is the usual tool
- Exit code **137 = OOMKilled**; `kubectl logs --previous` for crashed containers
- USE method: **Utilization, Saturation, Errors** (Brendan Gregg) — use it for any "slow" question

## PART 7: Questions YOU Should Ask Them (pick 3)

1. "What are the most common ticket categories the team sees — is it more workspace/environment issues, data connectivity, or platform-level incidents?"
2. "Is Domino running on EKS/AKS here, and does the platform team own the Kubernetes layer too, or is that a separate team?"
3. "How mature is the monitoring/alerting today — are you mostly reactive to user tickets, or catching issues before users report them?" *(sets you up as the automation person)*
4. "How does GxP validation interact with platform upgrades — what's the typical lead time for a Domino version upgrade here?"
5. "What would success look like for this role in the first 90 days?"

---

## Tonight's Priority Order (you have limited hours)
1. **Part 1** — Domino concepts table + architecture flow. Be able to narrate "user clicks Start Workspace → …" from memory. (45 min)
2. **Part 3 + 4** — the troubleshooting framework and scenarios S1, S3, S4, S6. Say them out loud. (60 min)
3. **Part 5** — fill in your 4 STAR stories with real details. (45 min)
4. **Part 2F** — GxP/ITSM answers, since pharma culture screening is likely. (20 min)
5. Sleep. A rested brain reasoning through Kubernetes beats a tired brain reciting Domino trivia.
