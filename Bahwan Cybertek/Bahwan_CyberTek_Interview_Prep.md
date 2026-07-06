# Bahwan CyberTek — DevOps Engineer Interview Prep (30-min interview, 8-hour plan)

---

## PART 0: YOUR STRATEGY (READ THIS FIRST)

### The one rule
A 30-minute interview has time for ~8–10 questions. The interviewer will pick them from the Job Description. Your job is NOT to know everything — it's to sound solid on the high-probability topics and handle the rest gracefully.

### Your positioning statement (memorize this framing)
Your real, defensible strengths: **Linux administration, AWS infrastructure (VPC/EC2/EKS), Terraform provisioning, Docker, Kubernetes platform operations (Domino on EKS), monitoring with Prometheus/Grafana**.

Your gaps vs. this JD: **Splunk→Google Observability migration, SCP changes, Zscaler, Checkmarx, Snowflake, TGW production changes, AWS Org-to-Org migration**.

For gaps, use this template — it works because it shows you understand the concept even without production experience:

> "I haven't executed that specific migration in production, but I understand the mechanics of it — [give 2 sentences of concept]. In my current role I've done [closest real thing], so the pattern of planning, dependency mapping, and rollback is familiar to me."

This is 10x more credible than a fake story, because fake stories die on the second follow-up question.

### 8-hour study plan
- **Hours 1–3:** Tier 1 questions (below) — these are near-guaranteed. Read each answer twice, then explain it out loud to yourself without looking. Speaking it out loud is what makes it stick.
- **Hours 4–5:** Tier 2 questions — read once carefully, note the analogies.
- **Hour 6:** Tier 3 (behavioral + gap-handling scripts) — practice your "tell me about yourself" out loud 3 times.
- **Hour 7:** Rapid revision — read only the **bolded key lines** in every answer.
- **Hour 8:** Rest. Do NOT cram new material in the last hour. A calm brain retrieves better than a stuffed one.

---

# TIER 1 — ALMOST GUARANTEED QUESTIONS

---

## Q1. "Tell me about yourself / walk me through your resume."

**Model answer (60–90 seconds, practice out loud):**

> "I have about 7 years of experience across DevOps and platform engineering. I started at TCS supporting a pharmaceutical client — Linux administration, AWS EC2/VPC/IAM, and I automated IBM MQ installation across 80+ servers using Bash and Ansible, which saved the team roughly 300 hours.
>
> Then at Accenture with JP Morgan Chase, I worked on containerizing a microservices Rewards application with Docker, migrating workloads to Amazon EKS, and building CI/CD pipelines — that's where I got deep into Kubernetes operations and centralized logging with Fluent Bit and Splunk.
>
> Currently I'm with Accenture on the Disney account as a Senior Cloud Engineer, where I work on the Domino Data Science platform running on Amazon EKS — Terraform-driven AWS provisioning, Prometheus/Grafana monitoring, and Kubernetes platform operations.
>
> What attracts me to this role is that it's hands-on AWS, Terraform, and migration work — which lines up with the infrastructure and automation side I enjoy most."

**Why this works:** It's chronological, each job has ONE memorable achievement, and it ends by connecting to THEIR job. Never ramble past 90 seconds.

---

## Q2. "Explain Terraform. What is the state file and why does it matter?"

**What Terraform is (understand it like this):**
Terraform is **Infrastructure as Code** — instead of clicking around the AWS console to create a VPC, an EC2 instance, or an EKS cluster, you *describe* what you want in text files (`.tf` files, written in HCL language), and Terraform makes AWS match that description.

**The mental model:** Terraform is like a *thermostat*. You declare the desired temperature (desired infrastructure), and it figures out what actions are needed to get there. You don't tell it *how* — you tell it *what*. That's why it's called **declarative**.

**The core workflow — memorize these 4 commands:**
1. `terraform init` — downloads the providers (plugins, e.g., the AWS provider) and sets up the backend. Run once per project/when providers change.
2. `terraform plan` — a **dry run**. Compares your code vs. the real world and shows what it *would* create/change/destroy. Nothing is touched yet.
3. `terraform apply` — actually makes the changes.
4. `terraform destroy` — tears everything down.

**The state file (`terraform.tfstate`) — the most asked Terraform question:**
- The state file is Terraform's **memory**. It's a JSON file recording every resource Terraform created and its real-world ID (e.g., "I created VPC vpc-0abc123").
- **Why it's needed:** when you run `plan`, Terraform compares three things: your code (desired), the state file (what it thinks exists), and the real cloud (actual). Without state, Terraform wouldn't know that the VPC in your code is the *same* VPC it already created — it would try to create a duplicate.
- **Why teams use REMOTE state (S3):** if the state file only lives on your laptop, your teammate can't work on the same infrastructure, and if your laptop dies, Terraform loses its memory. So teams store state in an **S3 bucket**.
- **State locking (DynamoDB):** if two engineers run `apply` at the same time, they'd corrupt the state. A DynamoDB table is used as a **lock** — the first person's apply locks the state; the second person must wait. 
- **Golden line to say:** *"We keep state in S3 with DynamoDB locking and versioning enabled — remote state for collaboration, locking to prevent concurrent applies, versioning so we can recover from a corrupted state."*

**Terraform modules (the JD mentions "reusable modules"):**
- A module is a **reusable package of Terraform code** — like a function in programming. Instead of copy-pasting 200 lines of VPC code into every project (violating DRY), you write a `vpc` module once, and every team calls it with different inputs (variables) like CIDR range and environment name.
- **Why the JD cares:** modules enforce **standardization** — every VPC in the company gets built the same compliant way.
- Real experience you can cite: *"In my current role I modularized our Terraform for VPC, EKS, IAM and security groups — provisioning time dropped from hours to under 45 minutes."* (This is on your resume — own it.)

**Likely follow-up: "What happens if someone changes a resource manually in the console?"**
> "That's drift. On the next `terraform plan`, Terraform compares state to reality, detects the difference, and proposes reverting it to match the code. The fix is either apply (revert the manual change) or update the code to make the manual change official. The real lesson is process: changes should only go through code."

---

## Q3. "Explain a VPC and its components." (Core AWS networking — heavily weighted in this JD)

**The mental model:** A VPC is your **private, fenced-off section of AWS's network** — like renting a floor in an office building. Inside your floor you decide the room layout (subnets), who can enter (security), and how traffic flows (route tables).

**The components, in the order traffic flows:**

1. **CIDR block** — the IP range of your VPC, e.g., `10.0.0.0/16` (≈65,000 IPs). Every resource inside gets an IP from this range.

2. **Subnets** — subdivisions of the VPC, each living in ONE Availability Zone.
   - **Public subnet** = a subnet whose route table has a route to the Internet Gateway. Things here (like load balancers) can be reached from the internet.
   - **Private subnet** = no direct internet route. Your application servers and databases live here for safety.
   - **The key definition to say:** *"A subnet is public or private purely based on its route table — public has a 0.0.0.0/0 route to the IGW, private doesn't."*

3. **Internet Gateway (IGW)** — the VPC's front door to the internet. Attached to the VPC, one per VPC. Allows two-way traffic for public subnets.

4. **NAT Gateway** — solves this problem: *servers in private subnets need to download updates/packages (outbound), but must NOT be reachable from the internet (inbound).* The NAT Gateway sits in a **public** subnet; private subnets route their outbound internet traffic through it. Traffic can go out, responses come back, but no one outside can initiate a connection in. **One-way valve** — that's the analogy.

5. **Route tables** — the GPS of each subnet: "traffic for 10.0.0.0/16 stays local, traffic for 0.0.0.0/0 (everything else) goes to the NAT/IGW."

6. **Security Groups vs NACLs** — a classic question:
   - **Security Group** = firewall on the *instance* (ENI level). **Stateful** — if you allow inbound port 443, the response traffic is automatically allowed out. Only *allow* rules.
   - **NACL** = firewall on the *subnet*. **Stateless** — you must explicitly allow both inbound AND outbound. Supports *deny* rules.
   - **Memorize:** *"Security groups are stateful and instance-level; NACLs are stateless and subnet-level. Day-to-day we control access with security groups; NACLs are a coarse subnet-level backstop."*

**Follow-up: "Why multiple Availability Zones?"**
> "High availability. An AZ is a physically separate data center. We spread subnets across at least 2–3 AZs so a data-center failure doesn't take down the platform — for example, EKS worker nodes across three AZs."

---

## Q4. "Explain Route 53 and how you'd do a DNS cutover." (Directly from JD)

**What Route 53 is:** AWS's **DNS service**. DNS is the internet's phonebook — it translates a name (`app.company.com`) into an IP address. Route 53 hosts your DNS records and answers those lookups.

**Record types to know:**
- **A record** — name → IPv4 address.
- **CNAME** — name → another name (`www.app.com` → `app.com`). Can't be used at the root/apex of a domain.
- **Alias record** — AWS special: like a CNAME but works at the apex and points at AWS resources (load balancers, CloudFront) for free.

**Routing policies (Route 53's superpower — say two or three):**
- **Simple** — one answer.
- **Weighted** — split traffic by percentage (e.g., 90% old, 10% new — great for gradual migration/canary).
- **Failover** — health-checks the primary; if it's unhealthy, DNS answers with the secondary. DR pattern.
- **Latency-based** — send users to the region closest/fastest for them.

**The DNS cutover question — how migrations actually happen (JD asks for this explicitly):**

> "The key concept is **TTL — Time To Live**. Every DNS record has a TTL that tells resolvers how long to cache the answer. If TTL is 24 hours and I change the record, some users keep hitting the old IP for up to 24 hours.
>
> So a safe cutover looks like: **(1)** Days before the cutover, lower the TTL from, say, 3600s to 60s and wait for the old TTL to expire — now the world refreshes quickly. **(2)** Verify the new target works (test with its direct IP/hostname). **(3)** At the change window, update the record to the new target. **(4)** Monitor traffic shifting — watch logs/metrics on both old and new. **(5)** Keep the old environment alive for rollback — rollback is just flipping the record back, and with a 60s TTL that takes effect in about a minute. **(6)** After stabilization, raise the TTL back up and decommission the old side."

**Why this answer wins:** it shows migration thinking — TTL planning, validation, monitoring, rollback — which is exactly the "risk mitigation, rollback planning, post-migration stabilization" language in the JD.

---

## Q5. "Explain Docker. Image vs container? What's in a Dockerfile?"

**The mental model:** 
- An **image** is a **recipe + frozen meal** — a read-only package containing your app, its dependencies, and the OS libraries it needs.
- A **container** is that image **running** — a live, isolated process. One image can run as many containers.
- **Say:** *"An image is the immutable template; a container is a running instance of it. Class vs object, if you like."*

**Why Docker at all (the "why" interviewers want):** It solves *"works on my machine."* The app is shipped WITH its entire environment, so dev, test, and prod run the identical thing. Containers share the host's Linux kernel (unlike VMs which each carry a whole OS), so they start in seconds and are lightweight.

**Dockerfile essentials — be able to narrate this:**
```dockerfile
FROM python:3.11-slim        # base image to build on
WORKDIR /app                 # working directory inside the image
COPY requirements.txt .      # copy dependency list first (layer caching!)
RUN pip install -r requirements.txt   # install deps
COPY . .                     # now copy the application code
EXPOSE 8000                  # documents the port the app listens on
CMD ["python", "app.py"]     # the command the container runs at start
```

**Two "senior-level" points that impress:**
1. **Layer caching:** every Dockerfile instruction creates a layer, and layers are cached. We copy `requirements.txt` and install dependencies BEFORE copying the app code — so when only code changes, Docker reuses the cached dependency layer and builds in seconds instead of minutes.
2. **Multi-stage builds:** build the app in a heavy "builder" image (with compilers/build tools), then copy only the final artifact into a slim runtime image. Result: smaller image, faster pulls, smaller attack surface.

**Troubleshooting commands (JD says "troubleshooting" for Docker):**
- `docker ps -a` — list containers including dead ones.
- `docker logs <container>` — first stop for a crashing container.
- `docker exec -it <container> sh` — shell into a running container.
- `docker inspect <container>` — full config, IP, mounts, exit code.
- **Common issue to mention:** *"A container exiting immediately usually means the main process (CMD) crashed or finished — I check `docker logs` and the exit code from `docker inspect`."*

---

## Q6. "Explain your GitLab CI/CD pipeline." (JD requires GitLab specifically)

**The concept:** CI/CD automates the path from `git push` to production. **CI (Continuous Integration)** = every commit is automatically built and tested. **CD (Continuous Delivery/Deployment)** = passing builds are automatically deployed.

**How GitLab CI works — the pieces:**
- A file named **`.gitlab-ci.yml`** in the repo root defines the pipeline. Push code → GitLab reads this file → pipeline runs.
- The pipeline is made of **stages** (run in order) containing **jobs** (run in parallel within a stage).
- Jobs execute on **runners** — worker machines (often containers) that pick up jobs.

**A pipeline you can describe/whiteboard:**
```yaml
stages: [build, test, scan, deploy]

build-image:
  stage: build
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA

unit-tests:
  stage: test
  script: [pytest]

security-scan:
  stage: scan
  script: [trivy image $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA]

deploy-prod:
  stage: deploy
  script: [kubectl set image deploy/app app=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA]
  when: manual        # human approval gate for production
  environment: production
```

**Concepts to name-drop correctly:**
- **Artifacts** — files a job saves and passes to later jobs (e.g., a build output or test report).
- **Cache** — speeds up repeat runs (e.g., dependency downloads).
- **Variables/secrets** — credentials stored in GitLab settings (masked), never in the repo.
- **`when: manual`** — approval gate before production deploys.
- **Shift-left (JD core principle!):** *"We run tests and security scans early in the pipeline — that's shift-left: catch problems at commit time when they're cheap, not in production when they're expensive."* The JD literally lists "Shift Left" as a core principle — use the phrase.

**Honest note:** at JPMC you used **Jules** (their internal CI tool). Say: *"At JP Morgan I built pipelines on Jules, their internal CI platform — the concepts map one-to-one to GitLab: pipeline-as-code, stages, runners, artifacts — and I use GitLab CI in my current environment."*

---

## Q7. Linux troubleshooting scenarios (JD: "performance tuning and troubleshooting")

### Scenario A: "A server is slow / high CPU. Walk me through your approach."

> "I follow a structured approach rather than guessing — essentially the USE method: check **Utilization, Saturation, Errors** for each resource.
>
> **(1)** `top` or `htop` — is CPU actually high? Which process? Also check **load average** (top-right of `top`): load average is the number of processes wanting CPU; if load is 12 on a 4-core box, processes are queuing — that's saturation.
> **(2)** Check what KIND of CPU use: high `%us` (user) = application burning CPU; high `%sy` (system) = kernel/syscall heavy; high `%wa` (**iowait**) = the CPU is actually idle *waiting for disk* — the problem is storage, not CPU. This distinction is the mark of someone who's actually debugged systems.
> **(3)** If iowait: `iostat -x 1` to find the busy disk. If memory: `free -h` — and if swap is being used heavily, the box is thrashing.
> **(4)** Drill into the process: `ps aux --sort=-%cpu | head`, application logs, `strace` if needed.
> **(5)** Fix + prevent: restart/scale/fix the app, then add a Grafana alert so we catch it before users do."

### Scenario B: "Disk is full."

> "**(1)** `df -h` — which filesystem is full. **(2)** `du -sh /* 2>/dev/null | sort -h` — walk down to find the biggest directories; it's usually logs. **(3)** The classic trap: `df` says full but `du` can't find the space — that means a process is holding a **deleted file open**; the space isn't freed until the process closes it. Find it with `lsof | grep deleted` and restart that process. **(4)** Prevention: logrotate, and disk-usage alerts at 80%."

**Why mentioning the deleted-file trap matters:** it's a real-world scar that instantly signals hands-on experience.

### Scenario C: "A service won't start."

> "`systemctl status <svc>` for state and last error lines → `journalctl -u <svc> -e` for full logs → common causes: config syntax error, port already in use (`ss -tlnp | grep <port>`), missing permissions/file, failed dependency. Fix, `systemctl restart`, and `systemctl enable` so it survives reboot."

---

## Q8. "Explain Kubernetes basics / your EKS experience."

(The JD lists K8s under 'preferred' but your resume leads with it, so it WILL come up.)

**The 30-second definition:** *"Kubernetes is a container orchestrator — you tell it the desired state ('run 3 replicas of this image behind this service') in YAML, and its controllers continuously reconcile reality to match: rescheduling crashed pods, replacing failed nodes, scaling. EKS is AWS's managed Kubernetes, where AWS runs the control plane for you."*

**Objects to be fluent in (one line each):**
- **Pod** — smallest unit; one or more containers sharing network/storage.
- **Deployment** — manages replicas of pods + rolling updates.
- **Service** — stable virtual IP/DNS in front of pods (pods die and change IPs; the Service doesn't). Types: ClusterIP (internal), NodePort, LoadBalancer (provisions an AWS LB).
- **ConfigMap / Secret** — configuration / sensitive config injected into pods.
- **PV / PVC / StorageClass** — persistent storage; on EKS a PVC with the gp3 StorageClass dynamically provisions an EBS volume (EBS = single-node RWO; EFS = shared RWX).
- **Ingress** — HTTP routing rules into the cluster (host/path → service).

**Your real story (rehearse it):**
> "In my current role I support the Domino data science platform on EKS — multiple environments, Terraform-provisioned clusters, Prometheus/Grafana for node health, pod status and resource utilization. Typical issues I handle: pods stuck in Pending (usually insufficient node resources or PVC binding problems), CrashLoopBackOff (check `kubectl logs --previous` and describe events), and node pressure evictions."

**The debug flow to recite:** `kubectl get pods` → `kubectl describe pod` (events section tells you WHY it's pending/failing) → `kubectl logs` (add `--previous` for the crashed attempt) → `kubectl exec` to poke inside.

---

# TIER 2 — LIKELY QUESTIONS (this JD is migration-heavy)

---

## Q9. "What is a Transit Gateway? How is it different from VPC peering?"

**The problem it solves:** Suppose a company has 30 VPCs that all need to talk to each other. With **VPC peering** (a direct one-to-one link between two VPCs), you'd need 30×29/2 = 435 separate peering connections — an unmanageable mesh. Peering is also **non-transitive**: if A↔B and B↔C are peered, A still cannot reach C through B.

**Transit Gateway (TGW) = the central router / airport hub.** Every VPC (and on-premises VPN/Direct Connect) attaches to the TGW once, and the TGW routes between all of them. 30 VPCs = 30 attachments instead of 435 peerings. Traffic between any two attached networks flows through the hub, and TGW route tables let you control *who* can reach *whom* (e.g., prod VPCs can't reach dev VPCs).

**Memorize:** *"Peering is point-to-point and non-transitive — fine for two or three VPCs. Transit Gateway is hub-and-spoke: every VPC attaches once, routing is centralized and controllable, and it also connects on-prem via VPN or Direct Connect. At enterprise scale, TGW."*

**If asked about "TGW changes" from the JD:** *"TGW changes are high-blast-radius because a wrong route table entry can black-hole traffic for many VPCs at once — so I'd treat them like a production cutover: document current routes, make the change in a window, validate reachability from representative sources, and have the exact rollback route change ready."*

---

## Q10. "Explain SSL/TLS and certificate management. How do you handle a certificate change?" (Directly in JD)

**What TLS does — two jobs:**
1. **Encryption** — traffic between browser and server can't be read in transit.
2. **Identity** — the certificate proves the server really is `app.company.com`, because a trusted **Certificate Authority (CA)** signed it. Browsers trust the CA, so they trust the cert.

**The handshake, simply:** client connects → server presents its certificate → client verifies the CA signature, the domain name matches, and the expiry date → both sides agree on encryption keys → encrypted session begins.

**ACM (AWS Certificate Manager) — the AWS answer:**
- ACM issues free public certificates for your domains and **auto-renews** them — this eliminates the classic outage cause: an expired cert nobody was watching.
- Validation is usually **DNS validation**: ACM asks you to create a specific CNAME record in Route 53 proving you control the domain; if that record stays in place, renewals are automatic forever.
- ACM certs attach directly to **ALB/NLB, CloudFront, API Gateway** — you never handle the private key.

**"How would you execute a certificate change?" (JD language):**
> "Issue/import the new cert first and validate it, attach it to the load balancer or CloudFront distribution alongside or replacing the old one during a change window, verify with `openssl s_client -connect host:443` or a browser that the new cert is being served with the full chain, watch error rates for TLS handshake failures — clients failing usually means a missing intermediate cert in the chain — and keep the old cert available for quick re-attachment as rollback. Expiry monitoring/alerting afterwards so it's never a surprise."

**Common outage story you can reference generically:** *"Most cert incidents are either an expired cert or an incomplete chain — the server sends its cert but not the intermediate, and some clients fail. That's why validation after the change matters."*

---

## Q11. "What are SCPs (Service Control Policies)?" (Directly in JD)

**The setup:** AWS **Organizations** lets an enterprise manage many AWS accounts under one umbrella (org), grouped into **OUs (Organizational Units)** — e.g., Prod OU, Dev OU, Sandbox OU.

**What an SCP is:** an organization-level **guardrail** that sets the *maximum* permissions any identity in an account can have. Think of it as **the outer fence**: IAM decides what a user is allowed to do, but the SCP defines what is even *possible* in that account.

**The #1 gotcha (interviewers love this):** **SCPs never GRANT permissions — they only limit them.** A user needs: (allowed by IAM) AND (allowed by SCP). If either says no, it's no. Even the account's root user is constrained by SCPs.

**Concrete examples to give:**
- "Deny everything outside ap-south-1 and us-east-1" — region restriction for compliance.
- "Deny `s3:DeleteBucket` in prod accounts."
- "Deny leaving the organization or disabling CloudTrail" — governance protection.

**Why SCP *changes* are risky (JD asks about executing them):** an SCP applies to entire OUs — a wrong Deny can instantly break workloads across dozens of accounts. So: test the policy on a sandbox OU first, use IAM Access Analyzer / CloudTrail to check what would be affected, roll out to one account, then expand. That answer shows the "controlled enterprise environment" mindset the JD wants.

---

## Q12. "How would you plan a migration from Org A to Org B?" (The centerpiece of this JD)

You likely won't have done this — but they may ask *how you'd approach it*. Give a framework; frameworks are credible without faking specifics.

> "I'd structure it in phases:
>
> **1. Discovery & dependency mapping.** Inventory everything in scope — accounts, VPCs, DNS zones, certificates, IAM roles, cross-account dependencies, data stores. The migrations that fail are the ones that miss a hidden dependency, like a hardcoded IP or a cross-account role.
>
> **2. Design the target & the order.** Decide what moves vs. what gets rebuilt fresh in Org B (with Terraform, rebuilding from code is often cleaner than moving). Sequence by dependency: shared networking first, then platforms, then applications.
>
> **3. Plan each cutover like a mini-project:** change window, step-by-step runbook, validation checklist, explicit **rollback plan**, and a communication plan for stakeholders.
>
> **4. Pilot.** Migrate the lowest-risk workload first, learn, refine the runbook.
>
> **5. Execute in waves,** with things like DNS handled via the TTL-lowering pattern so rollback is a one-minute record flip.
>
> **6. Post-migration stabilization:** hypercare period, monitoring parity in the new org, then decommission the old side only after a defined bake time.
>
> The principles I'd hold throughout: service continuity, rollback-first thinking, and validating after every step rather than at the end."

**Then anchor to reality:** *"The closest I've done hands-on is migrating on-prem workloads to EKS at JP Morgan and standing up multi-environment platforms with Terraform — same discipline of dependency mapping, runbooks, and validation, at a smaller blast radius."*

---

## Q13. "Have you worked on Splunk to Google Observability migration?" (Be honest here — this is un-fakeable)

**Your honest, strong answer:**
> "I've worked extensively on the Splunk side — at JP Morgan I integrated Fluent Bit with Splunk for centralized log aggregation across Kubernetes, so I know the log pipeline: agents on the nodes, forwarding, indexing, dashboards, alerts. I haven't executed a migration to Google Observability specifically, but I understand what it involves: re-pointing the log routing — for example Fluent Bit can output to Google Cloud Logging instead of Splunk — rebuilding the critical dashboards and alerts on the Google side, running both in parallel to validate parity before cutting over, and then retiring Splunk. The agent-based collection model is the same; it's the destination, query language, and alerting that change."

**Minimum concepts so the terms don't scare you:**
- **Google Observability** (formerly Stackdriver) = Google Cloud's suite: **Cloud Logging** (logs), **Cloud Monitoring** (metrics/dashboards/alerts), **Cloud Trace** (request tracing).
- **Google SecOps** (formerly Chronicle) = Google's **SIEM** — a security analytics platform that ingests security logs at scale and lets security teams detect and investigate threats. Splunk is often used as a SIEM too, which is why enterprises migrate Splunk security use-cases to SecOps (usually to cut Splunk licensing cost).
- The migration pattern in one line: **"re-route the pipes, rebuild the views, run in parallel, validate parity, cut over."**

---

## Q14. Quick definitions you must not blank on (one confident paragraph each)

**WAF (Web Application Firewall):** protects web applications at layer 7 (HTTP). Unlike a network firewall (which filters by IP/port), a WAF inspects the *content* of HTTP requests and blocks attack patterns — **SQL injection, cross-site scripting (XSS)**, bad bots, and it can rate-limit. AWS WAF attaches to CloudFront, ALB, or API Gateway and uses rules/managed rule sets. **WAF change risk:** a too-aggressive rule blocks legitimate users — so new rules go in **Count mode** first (log what *would* be blocked), review, then switch to Block.

**Zscaler:** a cloud security proxy platform many enterprises use. Instead of routing all employee/server internet traffic through a corporate data-center firewall, traffic goes through Zscaler's cloud, which does filtering, SSL inspection, and zero-trust access (ZIA for internet access, ZPA for private app access). **Why it matters to a DevOps engineer:** Zscaler sits in the traffic path — so Zscaler changes can break server connectivity to external endpoints (package repos, APIs), TLS validation (because it re-signs traffic with its own cert), and DNS behavior. If asked: *"I haven't administered Zscaler, but I've dealt with enterprise proxy behavior — cert-chain and egress issues — which is where Zscaler changes typically bite infrastructure."*

**Checkmarx:** a **SAST** tool — Static Application Security Testing. It scans source code for vulnerabilities (injection flaws, insecure crypto, secrets) *without running the app*, typically as a stage in the CI pipeline. That's shift-left security. Contrast: SAST scans code; DAST tests the running app; tools like Trivy scan container images for vulnerable packages.

**CloudFront:** AWS's **CDN** — caches content at 400+ edge locations worldwide so users get responses from nearby, cutting latency and origin load. Also the natural attachment point for WAF and ACM certs.

**Snowflake:** a cloud data warehouse — SQL analytics at scale, with storage and compute separated and independently scalable. From a DevOps view: you manage access, network connectivity (PrivateLink), and cost.

**Athena:** serverless SQL queries directly against files in S3 — pay per query, no infrastructure. Classic use: querying logs (ALB logs, CloudTrail) stored in S3.

**Lambda / Step Functions / Glue in one breath:** *"Lambda is serverless functions — code that runs on events without servers. Step Functions orchestrates multi-step workflows across Lambdas and services with retries and branching, as a state machine. Glue is managed ETL — crawls, catalogs, and transforms data, usually feeding Athena or a warehouse."*

**ECS vs EKS:** both run containers. **ECS** is AWS's own, simpler orchestrator — less to manage, AWS-proprietary. **EKS** is managed Kubernetes — the industry-standard API, more powerful and portable, more operational complexity. *"ECS for simplicity in an all-AWS shop; EKS when you want the Kubernetes ecosystem or portability."*

---

## Q15. "How do you approach cost optimization?" (A JD 'core principle')

> "A few layers: **(1) Right-sizing** — use utilization metrics to shrink over-provisioned instances/nodes; most environments run at 15–20% utilization. **(2) Pricing models** — Savings Plans/Reserved for steady workloads, **Spot instances** for fault-tolerant ones like CI runners or batch jobs, at up to ~70–90% discount. **(3) Storage hygiene** — S3 lifecycle policies to move old data to cheaper tiers, delete unattached EBS volumes and old snapshots, gp2→gp3 migration is a free ~20% saving. **(4) Turn things off** — non-prod environments scheduled down at night/weekends. **(5) Visibility** — tagging + Cost Explorer so costs map to teams; you can't optimize what you can't attribute. In Kubernetes specifically: set proper resource requests so nodes bin-pack efficiently, and use cluster autoscaler."

---

# TIER 3 — BEHAVIORAL + GAP HANDLING

---

## Q16. "Tell me about a challenging production issue you resolved." (STAR — use a REAL one)

Use a true story from your work. If you need a structure, here's a defensible one based on your actual Domino/EKS work — adapt the details to what really happened to you:

> **S:** "On the Domino platform, data scientists reported workspaces stuck in a starting state — a platform-down situation for their work."
> **T:** "I was on point to restore service and find the root cause."
> **A:** "I started at the Kubernetes layer: `kubectl get pods` showed pods Pending. `kubectl describe` events showed the scheduler couldn't place them — insufficient resources on the nodes. Grafana confirmed node memory pressure had been climbing. Short-term I scaled the node group; then I found the real cause — workloads without proper resource requests/limits, so a few heavy jobs starved the cluster."
> **R:** "Service restored within the hour; longer-term we enforced resource requests/limits and added Grafana alerts on node pressure, so we now catch it before users notice. Troubleshooting time on similar issues dropped significantly."

**Why STAR:** Situation, Task, Action, Result — it forces a complete story with an outcome. Always end with the result and the prevention step; prevention is what separates senior from junior.

## Q17. "This role needs someone self-driven with minimal supervision. Give an example."

> "At TCS I owned the IBM MQ automation end-to-end — identified that manual installs across 80+ servers were burning the team's time, built the Bash/Ansible automation myself, tested it on a subset, then rolled it out. It saved roughly 300 hours and got team-wide recognition. Nobody assigned me that; I saw the waste and fixed it." 

(True, on your resume, verifiable — this is your best independence story.)

## Q18. The gap-handling script — when asked "Have you done X?" and you haven't

**Never say just "no." Never fake a story. Use this 3-step pattern:**
1. **Honest one-liner:** "I haven't run that in production..."
2. **Concept proof:** "...but I understand how it works: [2 sentences — you now have these for every JD topic above]."
3. **Bridge to real experience:** "...and the closest thing I've done hands-on is [real thing]."

Example for "Have you done AWS Bedrock?": *"I haven't built on Bedrock — I know it's AWS's managed service for foundation models, API access to models like Claude without managing infrastructure. I do use AI tooling daily in my workflow — Claude Code and Copilot — so I'm comfortable in that ecosystem."*

**Why this works:** interviewers expect gaps on a JD this broad (nobody has ALL of: TGW + Zscaler + SCP + Snowflake + Checkmarx + Google SecOps migrations). What they're screening for is whether you *learn fast and tell the truth* — because a contractor who bluffs is a liability in production.

## Q19. Questions YOU should ask (pick 2 — asking nothing looks disengaged)

1. "What stage is the Org A to Org B migration in — planning, pilot, or execution? What's been the hardest dependency so far?"
2. "For the Splunk to Google Observability move — is the driver primarily cost, consolidation, or capability?"
3. "What does success look like for this role in the first 90 days?"

These show you read the JD and think like the person doing the job.

---

# FINAL-HOUR CHEAT SHEET (read this in hour 7)

- Terraform: **init → plan → apply**; state = Terraform's memory; **S3 + DynamoDB lock**; modules = reusable, DRY, standardized.
- Public subnet = route to **IGW**; private outbound via **NAT GW** (one-way valve). SG = **stateful, instance**; NACL = **stateless, subnet**.
- Route 53 cutover: **lower TTL → validate target → flip record → monitor → old side = rollback**.
- Docker: image = template, container = running instance; deps layer before code layer (**cache**); multi-stage = small images; `docker logs` first.
- GitLab CI: `.gitlab-ci.yml`, **stages → jobs → runners**; scan early = **shift-left**; `when: manual` for prod.
- Linux: high iowait = disk problem not CPU; `df` vs `du` mismatch = **deleted file held open** (`lsof | grep deleted`); `journalctl -u svc`.
- K8s debug: **get → describe (events!) → logs --previous → exec**.
- TGW = **hub-and-spoke router**; peering = point-to-point, non-transitive.
- ACM: **DNS-validated, auto-renews**; cert failures = expiry or missing intermediate chain.
- SCP: **guardrail, never grants**, IAM ∩ SCP; test on sandbox OU first.
- Migration framework: **discover → map dependencies → runbook + rollback → pilot → waves → stabilize**.
- WAF = layer-7, SQLi/XSS, new rules in **Count mode** first. Checkmarx = **SAST**. Google SecOps = **SIEM**.
- Gap script: **honest → concept → bridge to real experience.**

Good luck. Speak slowly, anchor in your real work, and when you don't know — say so, then show you understand the concept. That combination gets contractors hired.
