# AWS Interview Guide — From Zero to Confident Answers
### EC2 · VPC · IAM · Security Groups · Route 53 · Load Balancers · Auto Scaling · S3

---

## PART 0: The Mental Model (the map everything fits on)

**What AWS is:** renting slices of Amazon's data centers by the hour — machines, networks, storage, and services — created and destroyed via API calls instead of purchase orders.

**Geography — Regions and AZs (foundational, asked constantly):**
- A **Region** (e.g., us-east-1, ap-south-1) is a geographic cluster of data centers. Resources live in a region; most don't leave it automatically.
- An **Availability Zone (AZ)** is one or more physically separate data centers *within* a region — own power, cooling, network. us-east-1 has AZs like us-east-1a, 1b, 1c.
- **The rule that drives all HA design:** one AZ can fail (fire, flood, power). Anything important runs in **at least 2–3 AZs**. Every "how do you make X highly available" answer starts with "spread it across AZs."

**The shared responsibility model (a guaranteed question):**
> "AWS is responsible for security **OF** the cloud — the physical data centers, hardware, and the infrastructure running the services. The customer is responsible for security **IN** the cloud — what we configure: IAM permissions, security groups, encryption settings, patching our EC2 operating systems, our S3 bucket policies. Practically: if an S3 bucket leaks data, that's on us, not AWS — misconfiguration is the customer's side of the line."

**How the ten services in this guide fit together (the one-paragraph map):**
> A **VPC** is your private network — your fenced land. **Subnets** partition it across AZs; **route tables**, an **Internet Gateway**, and **NAT gateways** control what can talk to the internet. **EC2** instances are the virtual machines living in subnets, guarded by **Security Groups** (per-instance firewalls). **IAM** decides who and what may perform which AWS actions — humans and machines alike, via **policies** attached to users and **roles**. A **Load Balancer** spreads incoming traffic across instances in multiple AZs; an **Auto Scaling Group** adds/removes those instances with demand. **Route 53** is DNS — it turns your domain name into the load balancer's address. **S3** is the bottomless object store on the side for files, backups, and data.

---

## PART 1: VPC — Your Network (learn this first; everything lives inside it)

**Q1. What is a VPC? Explain subnets, and public vs private.**
> "A VPC is a logically isolated private network within AWS — you define its IP range (a CIDR block like 10.0.0.0/16, ~65k addresses) and nothing enters or leaves except how you allow. You carve it into **subnets**, each a smaller CIDR slice **bound to one AZ** — so multi-AZ design means creating subnets in each AZ.
> The public/private distinction is purely about routing: a **public subnet**'s route table has a route to an **Internet Gateway** (IGW) — instances with public IPs there are internet-reachable. A **private subnet** has no IGW route — nothing can reach it from the internet, period. Standard pattern: load balancers in public subnets; everything else — application servers, databases, EKS nodes — in private subnets. Minimal attack surface: the only thing facing the internet is the thing designed to."

**Q2. How do private instances reach OUT to the internet (updates, package downloads)? NAT.**
> "A **NAT Gateway** — placed in a *public* subnet — with the private subnets' route tables pointing internet-bound traffic (0.0.0.0/0) at it. NAT translates their private addresses to its public one for **outbound** connections and relays replies back; but nothing on the internet can *initiate* a connection inward. Outbound yes, inbound no — exactly what private servers need for patching and pulling images. Operational notes worth volunteering: NAT gateways are per-AZ (one per AZ for HA, or a single one as a cost/risk tradeoff), they cost per-hour AND per-GB (a data-heavy platform pulling huge images through NAT runs up real money — VPC endpoints for S3/ECR bypass NAT and are the fix), and 'my private instance can't download anything' is usually a missing/broken NAT route."

**Q3. Route tables and Internet Gateway — what do they actually do?**
> "The **IGW** is the VPC's door to the internet — attach one per VPC; it does nothing by itself. **Route tables** are the rules deciding where traffic goes, attached per subnet: every table has the local VPC route built in; adding `0.0.0.0/0 → IGW` is literally what *makes* a subnet public; `0.0.0.0/0 → NAT` makes a private subnet outbound-capable. Whenever 'X can't reach Y' comes up in AWS, the route table is check #1 — connectivity in a VPC is just routing plus firewalls, evaluated in that order."

**Q4. Security Groups vs NACLs — the classic comparison. (guaranteed question)**
> "Both filter traffic; they differ in level and memory:
> **Security Group** — a firewall on the *instance* (technically the network interface). **Stateful**: allow the inbound request and the response is automatically allowed back — you don't write return rules. **Allow rules only** — everything not allowed is denied. Default: all inbound denied, all outbound allowed. Killer feature: rules can reference *other security groups* — 'allow port 5432 from the app tier's SG' — membership-based instead of IP-based, which is how tiered architectures stay clean.
> **NACL** — a filter on the *subnet*. **Stateless**: return traffic must be explicitly allowed (hence rules for ephemeral ports — a classic exam/gotcha detail). Supports **explicit DENY** rules, evaluated in numbered order. Default NACL allows everything.
> Practice: security groups do ~95% of the work; NACLs stay default except for coarse subnet-level blocks (e.g., deny a hostile IP range). Troubleshooting order: SG first, NACL second — and remember SGs never block responses; NACLs can."

---

## PART 2: EC2 — The Machines

**Q5. Explain EC2 fundamentals: instances, AMIs, instance types, EBS.**
> "**EC2** = virtual machines on demand. An **AMI** is the template you launch from — OS plus whatever's baked in; golden AMIs with your hardening/agents pre-installed are how fleets stay consistent (and 'bake a new AMI, replace instances' is the immutable-infrastructure pattern vs patching in place). **Instance types** encode the hardware family and size — general purpose (m5/m6), compute-heavy (c5), memory-heavy (r5), GPU (g4dn/g5/p3) — chosen per workload; on our platform, GPU types back the GPU hardware tiers. **EBS** is network-attached block storage — the instance's disks: persistent independent of the instance, snapshot-able to S3, but **AZ-locked** (a volume attaches only to instances in its own AZ — the root cause of some sneaky scheduling issues) and single-attach (one instance at a time — the reason shared storage needs EFS instead). gp3 is the default SSD type; instance-store (ephemeral, dies with the hardware) exists for scratch speed."

**Q6. Purchase models: On-Demand vs Reserved/Savings Plans vs Spot — and when?**
> "**On-Demand** — pay by the second, no commitment: the default, and right for spiky/unknown loads. **Reserved/Savings Plans** — commit 1–3 years for steep discounts (up to ~70%): right for the always-on baseline — the platform nodes running 24/7. **Spot** — AWS's spare capacity at up to ~90% off, with the catch that AWS can reclaim it on 2 minutes' notice: right for fault-tolerant, interruptible work — batch jobs, some CI runners; wrong for anything that can't die suddenly (a user's 3-hour interactive workspace). Mature cost strategy = layers: reserved for baseline, on-demand for interactive burst, spot for tolerant batch — and I'd quantify the mix from utilization data."

**Q7. What's instance metadata (IMDS) — and why do people mention IMDSv2?**
> "Every instance can query `169.254.169.254` for data about itself — instance ID, AZ, and critically **the temporary credentials of its IAM role**. That last part made IMDS the target of SSRF attacks: trick an app into fetching that URL and it hands over cloud credentials. **IMDSv2** requires a session token (a PUT first), which blunts naive SSRF — enforcing IMDSv2 is now standard hardening. This connects IAM to EC2: metadata is *how* a role's credentials physically arrive on the machine."

---

## PART 3: IAM — Who May Do What (the most interview-dense topic here)

**Q8. Explain IAM: users, groups, roles, policies — precisely.**
> "IAM governs authentication (who are you) and authorization (what may you do) for every AWS API call.
> **User** — a permanent identity for a person (or legacy service) with long-lived credentials: password/access keys.
> **Group** — a bundle of users for attaching policies once ('admins', 'developers').
> **Role** — an identity with permissions but **no permanent credentials**: it's *assumed*, yielding **temporary credentials** via STS. Anything can assume a role if the role's **trust policy** allows it — an EC2 instance, a Lambda, a Kubernetes pod, a user from another account, a federated corporate identity.
> **Policy** — a JSON document of permissions: statements of **Effect** (Allow/Deny), **Action** (`s3:GetObject`), **Resource** (which ARNs), optional **Condition**. Attached to users, groups, or roles.
> The modern posture to state plainly: **humans federate through SSO, machines use roles — long-lived access keys are the thing we're always trying to eliminate**, because a leaked key works from anywhere until rotated, while role credentials expire in hours."

**Q9. How is a policy evaluated? (the deny logic — a favorite probe)**
> "Three-step logic: **(1)** By default, everything is implicitly **denied**. **(2)** An **Allow** in any applicable policy grants the action. **(3)** An explicit **Deny** anywhere **overrides every Allow** — deny always wins. So debugging 'access denied despite an Allow' means hunting for an explicit Deny in: another attached policy, a **permissions boundary**, a **Service Control Policy** from the AWS Organization (invisible in the account itself — the classic 'the policy looks perfect but it still fails' cause in enterprises), or a **resource-based policy** like an S3 bucket policy denying from outside a VPC. Also worth naming: identity policies AND resource policies both apply — for same-account S3 access, an Allow on either side suffices, but a Deny on either side kills it."

**Q10. Why roles instead of access keys on an EC2 instance? Walk through how it works.**
> "Attach an **instance profile** (the wrapper that binds a role to EC2) and the instance's applications automatically get **temporary, auto-rotating credentials** through the metadata service — the SDK/CLI finds them with zero configuration and zero secrets on disk. Versus baking access keys into the machine or code: nothing to leak into an AMI or Git, nothing to rotate manually, and revocation is instant — detach the role. The same pattern generalizes everywhere: EKS pods get roles via IRSA/Pod Identity, humans get roles via SSO federation, cross-account access is role assumption. One design principle covers it all: **identity assumes role, receives short-lived credentials, credentials expire** — the phrase to use is 'least privilege with temporary credentials.'"

**Q11. What is STS? And cross-account access?**
> "STS — Security Token Service — is the credential vending machine behind every role assumption: `AssumeRole` (and variants like `AssumeRoleWithSAML`, which is exactly what Domino's credential propagation calls) returns a temporary key/secret/token trio with an expiry. **Cross-account**: account B creates a role whose *trust policy* names account A; a principal in A calls AssumeRole on it and operates in B with that role's permissions — no shared users, no shared keys, full CloudTrail attribution on both sides. This is the standard enterprise pattern for tooling, auditing, and CI reaching into workload accounts."

---

## PART 4: Load Balancers & Auto Scaling — Traffic and Elasticity

**Q12. ALB vs NLB — differences and when to use which. (guaranteed)**
> "**ALB — Application Load Balancer, layer 7 (HTTP/HTTPS):** understands the request — routes by host and path (`api.company.com` → one target group, `/auth/*` → another), terminates TLS, supports WebSockets, integrates auth. The default for web applications and what sits in front of a platform like Domino.
> **NLB — Network Load Balancer, layer 4 (TCP/UDP):** doesn't read the request — forwards connections at extreme scale with ultra-low latency, static IPs per AZ (firewall-whitelisting friendly), preserves source IPs. For non-HTTP protocols, extreme throughput, or when clients need fixed IPs.
> Both are multi-AZ and route only to **healthy targets in target groups** via **health checks** — and that health-check wiring is where most LB troubleshooting lives (T4). One-liner: *ALB reads the request and routes smart; NLB moves packets fast and dumb.*"

**Q13. Explain Auto Scaling Groups end to end.**
> "An **ASG** maintains a fleet: you give it a **launch template** (AMI, instance type, SG, role — the recipe), min/desired/max counts, and subnets across AZs. It then does three jobs: **(1) Self-healing** — health checks (EC2 status or, better, the load balancer's health check — instances failing app-level health get replaced, not just dead VMs); **(2) Scaling** — policies move desired capacity: target tracking is the modern default ('keep average CPU at 60%' — AWS does the math), plus step and scheduled scaling (scale up before the 9am login wave); **(3) Balance** — spreading instances across AZs. Instances are cattle by definition: nothing precious may live on one, state goes to EBS/EFS/S3/databases. On EKS, node groups are ASGs under the hood, with the cluster autoscaler adjusting desired capacity — so ASG failure modes (max reached, quota hit, AZ imbalance) surface as 'pods stuck Pending.'"

---

## PART 5: Route 53 — DNS

**Q14. Route 53 basics: record types and Alias vs CNAME. (the classic)**
> "Route 53 is AWS's DNS: hosted zones hold your domain's records. Types to know: **A** (name → IPv4), **CNAME** (name → another name), and AWS's special **Alias** record — like a CNAME pointing at an AWS resource (ALB, CloudFront, S3 site) but with two crucial advantages: **it works at the zone apex** (`company.com` itself — plain CNAMEs are forbidden there by the DNS spec) and **it's free and resolves internally**. Rule: pointing at an AWS resource → Alias, essentially always. And the operational habit: records have **TTLs** — caching means DNS changes propagate on a delay; lowering the TTL *ahead of* a planned cutover is the pro move."

**Q15. Route 53 routing policies?**
> "Beyond simple: **weighted** (split traffic by percentage — canary and gradual migrations), **latency-based** (send users to the lowest-latency region), **geolocation** (route by user location — compliance/localization), and **failover** — primary/secondary with **health checks**: Route 53 probes the primary and flips DNS to the standby when it fails, which is the DNS layer of a disaster-recovery design. Health-checked failover + multi-region is the standard 'how would you survive a region outage' ingredient."

---

## PART 6: S3 — The Object Store

**Q16. S3 fundamentals — and what makes it different from a filesystem.**
> "S3 stores **objects** (a blob + metadata, up to 5 TB) in **buckets** with globally unique names, addressed by key — no real directories, just key prefixes that look like paths. It's not a disk: no partial-file edits, no mounting as a normal filesystem, no POSIX — you PUT and GET whole objects over HTTP. What you get in exchange: effectively infinite capacity, **11 nines of durability** (data replicated across ≥3 AZs), massive parallel throughput, and per-GB pricing. It's the backbone role in most architectures — and in ours: Domino's blob storage, Terraform state, backups, logs."

**Q17. Storage classes and lifecycle policies?**
> "Classes trade retrieval speed/cost against storage price: **Standard** (hot data), **Standard-IA** (cheaper storage, per-GB retrieval fee — backups, older artifacts), **Glacier tiers** (archive — minutes-to-hours retrieval, very cheap), and **Intelligent-Tiering** (auto-moves objects by access pattern — the 'don't want to think about it' option). **Lifecycle policies** automate transitions and expiry: 'after 30 days → IA, after 180 → Glacier, delete noncurrent versions after 90.' For a platform generating endless artifacts and logs, lifecycle rules are one of the easiest recurring cost wins available — and the kind of automation improvement worth proposing in a first 90 days."

**Q18. S3 security: how do you prevent the infamous public bucket leak?**
> "Layers: **Block Public Access** — the account- and bucket-level master switch that overrides any misconfigured policy below it; on everywhere by default posture. **Bucket policies** (resource-based JSON — who may do what to this bucket, including Deny conditions like 'only from our VPC endpoint' or 'only over TLS') plus **IAM policies** on the caller — same-account access needs an Allow on either side and no Deny on either. **Encryption** — SSE-S3 or SSE-KMS (KMS adds key-level access control and audit — the GxP-friendly choice) — on by default now, but *enforced* by policy in regulated setups. **Versioning** — protects against overwrite/delete (and is non-negotiable on state/backup buckets). **Access logging/CloudTrail data events** for audit. And **presigned URLs** for sharing single objects temporarily instead of ever loosening the bucket. The honest summary: S3 leaks are always configuration, and Block Public Access exists because bucket-policy JSON proved too easy to get wrong."

---

## PART 7: Troubleshooting Questions & Answers

**T1. "You can't SSH/connect to an EC2 instance. Systematic diagnosis?"**
> "Layer by layer, outside-in: **(1) Is it running and healthy?** — console state and the two status checks (system = AWS's hardware problem, stop/start moves hosts; instance = OS-level — boot issue, resource exhaustion). **(2) Network path** — does it have a public IP (or am I coming through a VPN/bastion to a private one)? Is the subnet's route table right (public needs the IGW route)? **(3) Security Group** — inbound port 22 from *my* source IP? (The most common cause — someone's IP changed or the rule says a different CIDR.) **(4) NACL** — non-default NACLs are the stateless trap: inbound 22 AND outbound ephemeral ports needed. **(5) OS level** — sshd running? Host firewall? Disk full (SSH can fail weirdly on a full disk)? Right key/user? If the OS itself is the problem, **SSM Session Manager** gets you in without SSH at all — and the modern answer to volunteer: with SSM properly set up, we shouldn't be opening port 22 in the first place."

**T2. "A private-subnet instance can't reach the internet (yum/pip/docker pull all fail)."**
> "The NAT path, checked in order: route table on *that* subnet has `0.0.0.0/0 → NAT gateway`? The NAT gateway exists, is in a **public** subnet (a NAT in a private subnet is a decorative appliance — a real misconfiguration classic), and its own subnet routes to the IGW? Security group **outbound** rules allow 80/443 (default allows all out, but hardened environments restrict — check)? DNS resolving (VPC DNS enabled)? And if it's specific AWS services failing rather than the whole internet — S3, ECR — check whether the design intends **VPC endpoints** for those and the endpoint/policy is broken. Isolation trick: `curl` a public IP directly — works while names fail = DNS; both fail = routing/NAT."

**T3. "S3 access denied (403) — but 'the policy allows it.' Debug it."**
> "403 hunting order: **(1) Explicit Deny somewhere** — deny beats allow: bucket policy conditions (aws:SourceVpce, TLS-only, IP conditions), SCPs from the Organization, permissions boundaries. **(2) Which identity is actually calling?** — `aws sts get-caller-identity` first; half of these tickets are 'wrong profile/role assumed.' **(3) Both sides for cross-account** — cross-account needs the bucket policy AND the caller's IAM to allow. **(4) KMS** — if SSE-KMS encrypts the objects, the caller also needs kms:Decrypt on the key; 'S3 says yes, KMS says no' still reads as access denied — a top interview-worthy gotcha. **(5) Object ownership/ACL legacy** cases in old buckets. Tooling to name: the **IAM Policy Simulator** and CloudTrail's errorCode field show you exactly which policy said no."

**T4. "Users get 502/504 from the ALB. Diagnose."**
> "Decode the code first: **502 Bad Gateway** = the target answered *badly* — app crashed mid-request, closed the connection, TLS mismatch to the target, or responses malformed; **504 Gateway Timeout** = the target didn't answer *in time* — app hung, overloaded, or the app's own upstream (DB) is slow; ALB's idle timeout (default 60s) shorter than a legitimately slow request also manufactures 504s. Then the standard sweep: **target group health** — how many targets healthy? All unhealthy = the health-check config itself (wrong path/port, app returns 302 while the check demands 200 — a beloved classic) or the app truly down; SG chain — targets' SG must allow from the **ALB's** SG; and correlate with deploys — 5xx spikes right after a rollout point at the release, and the ALB's access logs + target response codes tell you whether the LB or the app is the liar."

**T5. "The Auto Scaling Group keeps launching and terminating instances in a loop (churn). Why?"**
> "Instances are failing health checks shortly after launch, so the ASG dutifully replaces them forever. Root causes in likelihood order: **health check grace period too short** — the app needs 3 minutes to boot, the ELB health check condemns it at 60 seconds: instant death loop, fix the grace period; **the app genuinely failing on boot** — bad AMI/user-data, can't reach a dependency, secrets missing — check an instance's logs *before* it's culled (or detach one from the ASG to keep it alive for autopsy — a nice practical detail); **ELB health check misconfigured** (path/port/expected code); or **capacity thrash** from aggressive scaling policies with no cooldown — scale-out and scale-in fighting each other; fix with cooldowns/target tracking. The meta-point: the ASG is doing exactly what it was told — the telling was wrong."

**T6. "DNS 'not working' after you changed a Route 53 record."**
> "First suspect: **TTL caching** — the old value legitimately lives in resolvers until TTL expires; `dig` the record directly against a Route 53 name server vs a public resolver to see whether the *authority* is right and the world just hasn't caught up (and remember the lesson: drop TTL before planned changes). Then: right **hosted zone**? (Public vs *private* hosted zones — a private zone answers only inside its associated VPCs, and 'works in the VPC, fails outside' or vice versa is the private-zone signature.) Correct record type — apex needs **Alias**, not CNAME? Health-checked failover records — is a health check failing and routing you to the secondary on purpose? And if the domain is registered elsewhere: do the registrar's NS records actually delegate to this zone's name servers — the silent misconfiguration that makes a perfect zone irrelevant."

---

## PART 8: Scenario / Design Questions

**S1. "Design a highly available, secure network for a web application (or our EKS platform) from scratch."**
> "One VPC, /16. Three AZs. Per AZ: a small **public subnet** (ALB, NAT gateway) and a large **private subnet** (application/EKS nodes), plus isolated subnets for databases if used. **IGW** for the VPC; NAT per AZ for private egress; **VPC endpoints** for S3/ECR to keep bulk traffic off NAT (cost + security). **ALB** in the public subnets terminating TLS (ACM certificate), forwarding to targets in private subnets; SG chain: ALB SG allows 443 from the world; app SG allows app-port **from the ALB's SG only**; DB SG from the app SG only — membership-based, no IP lists. **ASG/node groups** span the three private subnets. **Route 53** alias record → ALB. **IAM roles** everywhere, zero access keys; CloudTrail on; S3 buckets versioned, encrypted, Block Public Access. That design is simultaneously the answer to the HA question, the security question, and 'what does Domino's AWS layer look like' — it's the same picture."

**S2. "Your AWS bill doubled. Approach?"**
> "Diagnose before dieting: **Cost Explorer grouped by service, then by tag** — which service jumped, when, correlated with what change? Usual suspects for a data platform: **NAT data processing** (image pulls/data transfers through NAT — fix with VPC endpoints), **idle compute** (workspaces/instances left running — idle-shutdown automation is the single biggest platform win), **unattached EBS volumes and old snapshots** (orphans from terminated instances), **over-provisioned instances** (right-size from utilization metrics), **S3 without lifecycle rules** (everything hot forever), and **missing reservation coverage** on the 24/7 baseline. Then make it structural: tagging enforced so cost maps to teams, budgets/alerts so the next doubling pages someone at +20% instead of at invoice time. Cost is an ops metric, not a finance surprise."

**S3. "How would you give an external partner/another account temporary access to one S3 prefix?"**
> "Never a user with keys. Options by duration: **presigned URLs** for one-off object shares (minutes/hours, generated by our identity, no principal created at all); for ongoing access, a **cross-account role**: trust policy naming their account (ideally their specific role, with an ExternalId condition against confused-deputy), permission policy scoped to `s3:GetObject` on `bucket/partner-prefix/*` only; they AssumeRole and every action is CloudTrail-attributed. Add a bucket-policy backstop and, if warranted, an expiry via policy condition. The pattern to say out loud: *scope to the prefix, use roles not users, temporary credentials, both-sides auditability.*"

**S4. "An instance's IAM role credentials were leaked (SSRF/IMDS). Respond."**
> "Contain: **revoke the role's active sessions** (attach the AWSRevokeOlderSessions inline deny — invalidates credentials issued before now) — faster and safer than deleting the role; then rotate/patch the vulnerable app, enforce **IMDSv2** (hop limit 1) on the fleet, and review the role's permissions — the blast radius equals what the role could do, which is why least privilege was the real pre-incident control. Investigate with **CloudTrail**: every call made with those credentials is logged with source IPs — scope what was touched. Then the prevention writeup: IMDSv2 enforced account-wide, GuardDuty (flags anomalous credential use, including 'instance credentials used from outside EC2'), and the finding feeds the problem/CAPA process. Structure: contain → investigate → harden → institutionalize."

---

## PART 9: Rapid-Fire One-Liners (skim before the interview)

- **Region = geography; AZ = independent data center; HA = ≥2 AZs, always**
- **Shared responsibility: AWS = OF the cloud; you = IN the cloud (config, IAM, patching, buckets)**
- **Public subnet = route to IGW; private = no IGW route; NAT (in a public subnet) = outbound-only for private**
- **SG: instance-level, stateful, allow-only, can reference other SGs. NACL: subnet-level, stateless, has deny, ordered rules**
- **SG chain pattern: ALB-SG ← world:443; app-SG ← ALB-SG; db-SG ← app-SG**
- **EBS = AZ-locked, single-attach block disk; EFS = multi-attach NFS; S3 = objects over HTTP**
- **Purchase layers: Reserved (baseline) + On-Demand (burst) + Spot (interruptible batch, 2-min warning)**
- **IAM eval: implicit deny → allow grants → explicit deny WINS over everything (check SCPs & boundaries)**
- **Roles > keys: temporary creds via STS; EC2 gets them through IMDS (enforce IMDSv2); humans via SSO; pods via IRSA**
- **`aws sts get-caller-identity` = the first command in ANY access-denied debug**
- **KMS-encrypted S3 needs kms:Decrypt too — the hidden second permission**
- **ALB = L7, host/path routing, TLS; NLB = L4, static IPs, raw speed. Health checks gate targets**
- **502 = target answered badly; 504 = target didn't answer in time; all-targets-unhealthy = suspect the health check config**
- **ASG churn = grace period too short, app failing boot, or bad health check — the ASG obeys, the config lies**
- **Route 53: Alias > CNAME for AWS resources (works at apex, free); TTL caching delays changes — lower TTL before cutovers**
- **S3: 11 nines durability; versioning on important buckets; Block Public Access on; lifecycle rules = free money**
- **Leak response: revoke sessions → CloudTrail scope → IMDSv2/least-privilege hardening**

---

## FINAL RECALL STORY (read 3 times — the whole map in motion)

> *A user hits `platform.company.com`: **Route 53** answers with an **Alias** to the **ALB**, which terminates TLS in the **public subnets** and — reading the path, layer 7 — forwards to healthy targets in **private subnets**, allowed in only because the app **security group** trusts *the ALB's SG*, not IP lists. The targets are an **Auto Scaling Group** across three **AZs**; last month it looped killing newborn instances — the **health-check grace period** was shorter than boot time; the ASG obeyed, the config lied. The instances hold **no keys**: an **instance profile** feeds temporary **STS** credentials through **IMDSv2**, and the day one app got SSRF'd we **revoked the role's sessions**, scoped the damage in **CloudTrail**, and hardened the fleet. Their images pull through **VPC endpoints** — the bill taught us what **NAT per-GB pricing** means. Artifacts land in **S3**: versioned, KMS-encrypted (the 403 that stumped us was the missing **kms:Decrypt**, not S3), Block Public Access on, **lifecycle rules** quietly archiving to Glacier. And the Friday everything 'broke' after a DNS change? **TTL caching** — the world's resolvers finished forgetting on schedule, and we now drop TTLs before every cutover. Ten services, one picture: DNS → LB → firewall chain → elastic fleet in private subnets → roles not keys → objects in S3 — and every outage was a config telling the truth about itself.*

Hold Part 0's map and this story, and you can place any AWS question they ask onto the picture — then answer from where it sits.
