# The Break-Fix Lab: One Project to Understand AWS Networking

**Format:** Build a 3-tier web application from scratch in a hand-built VPC, then deliberately sabotage it 10 ways and diagnose each failure.

**Time:** 4–5 hours | **Cost:** ~$2–4 if torn down same day (biggest costs: NAT Gateway $0.045/hr, ALB $0.0225/hr, RDS db.t4g.micro ~$0.016/hr)

**The core idea:** You don't understand a component until you know what breaks without it. Every phase follows the same pattern:

1. **Why does this exist?** (the problem it solves)
2. **Build it** (console, deliberately — no Terraform this time; you need to *see* every knob)
3. **What breaks without it?** (you'll prove this in Phase 6)

---

## Architecture You're Building

```
Internet
   │
   ▼
[Route 53]  →  [ACM cert]
   │
   ▼
[ALB]  (public subnets, 2 AZs)          ── access logs ──► [S3] ──► [Glue/Athena]
   │
   ▼
[ECS Fargate task]  (private subnets)   ── outbound via ──► [NAT Gateway]
   │                                     ── S3 traffic via ──► [Gateway Endpoint]
   ▼
[RDS PostgreSQL]  (private subnets)
```

**Covered directly:** VPC, subnets, route tables, IGW, NAT Gateway, VPC endpoints, security groups, NACLs, IAM (roles/policies/trust), ECS, ALB (load balancing + health checks), RDS, EBS (via RDS storage), Route 53, Route 53 Resolver, ACM, SSL/TLS, certificate lifecycle, S3, Athena, Glue, DNS, routing, network troubleshooting.

**Covered as extensions (Appendix):** Transit Gateway, Lambda + API Gateway, CloudFront, Step Functions.

**Deliberately excluded:** MongoDB and Snowflake are not AWS services — see Appendix D for how to talk about them in interviews.

---

## Phase 0 — Setup & Cost Guardrails (15 min)

1. Use a region close to you: `ap-south-1` (Mumbai).
2. Set a billing alarm: Billing → Budgets → $5 budget with email alert. **Do this first.**
3. Bookmark the **teardown checklist** (Phase 8). NAT Gateway and ALB bill hourly whether used or not.
4. Optional but recommended: have a domain. Route 53 can register one for ~$3–14/yr, or use any domain you own. Without one, Phase 5 uses a **private hosted zone** instead — you still learn Route 53 mechanics, just not public DNS + ACM validation.

---

## Phase 1 — The Network Foundation (45 min)

### 1.1 VPC — why it exists

A VPC is your private slice of AWS's network: an isolated IP address space where *nothing* can talk to anything until you explicitly wire it. Without a VPC boundary, every resource would be on a shared network — no isolation between customers, no control over your own topology.

**Build:** VPC with CIDR `10.0.0.0/16`. Do NOT use the "VPC and more" wizard — create only the VPC. You will hand-build everything else, because the wizard is why people never learn this.

Enable **DNS hostnames** and **DNS resolution** on the VPC (Actions → Edit VPC settings). Note what these do — you'll break DNS later if you forget.

### 1.2 Subnets — why they exist

A subnet is a slice of the VPC CIDR pinned to **one Availability Zone**. Two reasons subnets exist:

1. **Blast radius / HA:** an AZ is a physical failure domain. Two subnets in two AZs = your app survives a data-center failure. ALB and RDS *refuse* to deploy without subnets in ≥2 AZs — you'll hit this wall yourself.
2. **Routing granularity:** routing decisions in AWS are made *per subnet* (via route table association). "Public" vs "private" is not a checkbox — it is purely a property of which route table the subnet uses.

**Build 4 subnets:**

| Name | CIDR | AZ |
|---|---|---|
| public-a | 10.0.0.0/24 | ap-south-1a |
| public-b | 10.0.1.0/24 | ap-south-1b |
| private-a | 10.0.10.0/24 | ap-south-1a |
| private-b | 10.0.11.0/24 | ap-south-1b |

On public-a and public-b: enable **auto-assign public IPv4**. (What happens if you forget? An instance launched there gets no public IP and is unreachable from the internet even with a perfect route table. Classic gotcha.)

### 1.3 Internet Gateway — why it exists

The IGW is the VPC's door to the internet. It's a horizontally-scaled, AWS-managed NAT between your instances' private IPs and their public IPs. Without it, no packet enters or leaves the VPC to the internet, period — no route can point anywhere useful.

**Build:** Create IGW, attach to VPC. (An unattached IGW does nothing — another quiet failure mode.)

### 1.4 Route Tables — why they exist

Every packet leaving a subnet consults exactly one route table. The table answers one question: "for this destination, where do I send this?" Longest-prefix match wins. Every route table has an unremovable `local` route for the VPC CIDR — that's why any two subnets in a VPC can always reach each other *at the routing layer* (security groups may still block).

**Build:**

- **Route table `rt-public`:** add route `0.0.0.0/0 → IGW`. Associate with public-a, public-b.
- **Route table `rt-private`:** leave it with only the `local` route for now. Associate with private-a, private-b.

**Key mental model:** the *only* difference between a public and private subnet is that one line: `0.0.0.0/0 → igw-xxx`. Nothing else.

### 1.5 NAT Gateway — why it exists

Private-subnet resources have no public IP, so the IGW can't translate for them — they can't reach the internet even with a route to the IGW (try it later: route private traffic to IGW directly and watch it blackhole). But they still need *outbound* internet: pulling container images, OS patches, calling external APIs.

The NAT Gateway sits in a **public** subnet, has an Elastic IP, and translates many private IPs to its one public IP — outbound only. Return traffic for established connections comes back; unsolicited inbound never gets in. That asymmetry *is* the security property.

**Build:** NAT Gateway in **public-a** with a new Elastic IP. Then add to `rt-private`: `0.0.0.0/0 → nat-xxx`.

**Question to sit with:** why must the NAT GW live in a public subnet? (Because *it* needs the IGW route to reach the internet. A NAT GW in a private subnet is a very expensive brick — real production incident pattern.)

### 1.6 S3 Gateway Endpoint — why it exists

Without it, ECS-to-S3 traffic goes: private subnet → NAT GW → IGW → S3's public endpoint. You pay NAT data-processing charges ($0.045/GB) for traffic that never needed to leave AWS. A **Gateway Endpoint** injects a route into your route table sending S3-destined traffic through AWS's internal network — free, faster, and no public exposure.

**Build:** VPC → Endpoints → create Gateway endpoint for S3, associate with `rt-private`. Then look at `rt-private`: you'll see a new route with a **prefix list** (`pl-xxx`) as destination — that's the set of S3 public IP ranges. This is longest-prefix-match doing real work: S3 traffic matches the prefix list route; everything else falls through to `0.0.0.0/0 → NAT`.

(Interface Endpoints — the other kind — are ENIs with private IPs for services like ECR, Secrets Manager, SSM. They cost ~$0.01/hr each. Know the difference: Gateway = route table entry, S3/DynamoDB only, free. Interface = ENI + private DNS, most services, paid.)

### 1.7 Checkpoint quiz (answer before moving on)

1. A packet leaves an ECS task in private-a headed to `142.250.183.14` (google.com). Trace every hop through your components.
2. Same task, packet headed to `10.0.0.55`. Which route matches?
3. Same task, packet headed to an S3 bucket. Which route matches, and why not the NAT?
4. You delete the `0.0.0.0/0` route from rt-public. What breaks, in order?

---

## Phase 2 — IAM: Who May Do What (30 min)

### 2.1 The model in four sentences

- A **policy** is a JSON document: Effect / Action / Resource / (Condition).
- An **identity** (user, role) has policies attached; a request is allowed only if some policy explicitly allows it and none explicitly denies it. **Default is deny.**
- A **role** is an identity with no password/keys that a trusted principal can *assume* to get temporary credentials. The **trust policy** says who may assume it; the **permission policies** say what it can do once assumed. These are two different documents and confusing them causes half of all IAM tickets.
- **Resource policies** (S3 bucket policies, etc.) live on the resource and are evaluated *together* with identity policies. Cross-account access needs the resource side to agree.

### 2.2 The distinction that matters for ECS

You will create **two** roles, and the difference is a top-tier interview answer:

| Role | Assumed by | Used for | Failure symptom if broken |
|---|---|---|---|
| **Task Execution Role** | ECS agent (the platform) | Pull image from ECR, write logs to CloudWatch, fetch secrets *to start the container* | Task never starts; stopped reason mentions ECR/CloudWatch |
| **Task Role** | Your application code inside the container | Whatever the app calls (S3, DynamoDB…) via SDK | App starts fine, then gets `AccessDenied` at runtime |

**Build:**

1. Role `ecsTaskExecutionRole`: trust policy principal `ecs-tasks.amazonaws.com`, attach AWS-managed policy `AmazonECSTaskExecutionRolePolicy`. **Open both JSON documents and read them.** Notice the trust policy is about `sts:AssumeRole`; the permission policy is about `ecr:*` and `logs:*` actions.
2. Role `appTaskRole`: same trust principal, attach an inline policy allowing `s3:GetObject`/`s3:PutObject` on a specific bucket ARN you'll create in Phase 3 (`arn:aws:s3:::your-lab-bucket/*`). Deliberately scope it narrowly — you'll widen it when it fails, and *feel* least-privilege in practice.

### 2.3 How to troubleshoot IAM — the universal method

1. Read the error. `AccessDenied` errors name the action and often the ARN.
2. **CloudTrail → Event history → filter by error code `AccessDenied`.** Every denied API call is logged with who, what action, what resource. This is the single most useful IAM debugging move and most engineers don't know it.
3. IAM console → the role → **Access Advisor** tab (what's actually used) and the **policy simulator** for "would this call succeed?"

---

## Phase 3 — Compute Tier: ECS + ALB + Security Groups (60 min)

### 3.1 Security Groups — why they exist

Route tables decide *where packets can go*; security groups decide *whether they're allowed in/out at the ENI*. SGs are **stateful**: allow inbound, the reply is automatically allowed out (and vice versa). SGs have no deny rules — anything not allowed is dropped.

The senior-level pattern: **reference SGs by SG ID, not by CIDR.** "App SG allows :5432 from *the ALB's SG*" survives IP changes, autoscaling, everything.

**Build three SGs (empty VPC-scoped shells first, then rules):**

| SG | Inbound | Outbound |
|---|---|---|
| `sg-alb` | 80, 443 from `0.0.0.0/0` | default (all) |
| `sg-app` | 8080 from **sg-alb** (by SG ID) | default (all) |
| `sg-db` | 5432 from **sg-app** (by SG ID) | default (all) |

This chain — internet → ALB → app → DB, each hop only from the previous SG — is the canonical 3-tier security model. Notice nothing in `sg-db` mentions an IP address.

**NACLs — the other layer:** NACLs sit at the *subnet* boundary, are **stateless** (return traffic needs its own rule — this is why ephemeral ports 1024–65535 matter), and support explicit DENY. Default NACL allows everything; leave it alone for now. You'll weaponize it in Phase 6. Interview one-liner: SG = stateful, ENI-level, allow-only; NACL = stateless, subnet-level, allow+deny, rule-number ordered.

### 3.2 The app

Skip writing code — use a public image so nothing distracts from infrastructure. Good choice: `public.ecr.aws/docker/library/httpd:latest` (Apache, serves on port 80 — adjust `sg-app` inbound to 80, or use an image on 8080). Even simpler for meaningful output: `ealen/echo-server` (echoes request details back — extremely useful for seeing what headers the ALB adds, like `X-Forwarded-For`).

**Build:**

1. ECS → Create cluster (Fargate).
2. Task definition: Fargate, 0.25 vCPU / 0.5 GB, the container image, port mapping 80 (or 8080), **execution role = ecsTaskExecutionRole, task role = appTaskRole**, awslogs log driver to a new CloudWatch group.
3. Don't create the service yet — ALB first.

### 3.3 ALB — why load balancers exist

Three problems, one component: (1) **distribution** — spread traffic across N replicas; (2) **health** — stop sending traffic to dead replicas, automatically; (3) **decoupling** — clients get one stable DNS name while backends churn (deploys, scaling, AZ failure). The ALB operates at L7: it terminates the HTTP(S) connection, inspects it, and opens a *new* connection to the target — which is why it can route on path/host/headers and why TLS can terminate here.

**Build:**

1. Target group: type **IP** (Fargate tasks are ENIs with IPs, not instances), protocol HTTP, port matching the container, health check path `/`. Look at the health check knobs: interval, threshold counts — these decide how fast a bad task is ejected.
2. ALB: internet-facing, **public-a + public-b** (it will refuse a single subnet — the ≥2 AZ wall from Phase 1), security group `sg-alb`. Listener HTTP:80 → target group.
3. **Enable access logs**: ALB → Attributes → access logs → new S3 bucket `<yourname>-alb-logs-lab`. The console offers to create the bucket policy — read that policy: it's a *resource policy* granting the regional ELB log-delivery account `s3:PutObject`. This is IAM resource policies in the wild, and it feeds Phase 7.

### 3.4 Wire ECS to ALB

Create the ECS **service**: 2 tasks, **private subnets**, `sg-app`, **no public IP** (they don't need one — that's what the NAT GW is for), attach to the target group.

**Watch what happens, in order:** service schedules tasks → execution role pulls the image *through the NAT Gateway* → tasks register in the target group as `initial` → health checks pass → `healthy`. Hit the ALB DNS name in your browser. If you used echo-server, look at `X-Forwarded-For` — that's how your app learns the real client IP despite the ALB terminating the connection.

**If tasks fail to start:** ECS → task → stopped reason. `CannotPullContainerError` = the task can't reach ECR/registry = almost always a NAT/route/subnet problem, not an application problem. Burn this into memory; it's the most common ECS-on-private-subnets failure.

---

## Phase 4 — Data Tier: RDS (30 min, provision it early — it takes ~10 min)

### 4.1 Why RDS / where EBS fits

RDS = a managed EC2+EBS+PostgreSQL bundle: AWS handles patching, backups, failover. The storage *is* EBS under the hood — when you pick gp3 and 20 GB, you're making an EBS decision. EBS itself: network-attached block storage, AZ-locked (a volume lives in one AZ — the reason Multi-AZ RDS is *replication*, not a shared disk), persists independently of the instance, snapshots to S3.

**Build:** RDS → PostgreSQL, `db.t4g.micro`, 20 GB gp3, **no public access**, subnet group = private-a + private-b (another ≥2 AZ wall), security group `sg-db`, single-AZ (Multi-AZ doubles cost; know that it exists: synchronous standby in the other AZ, automatic DNS failover on the *endpoint* — which is a Route 53 CNAME, connecting to Phase 5).

### 4.2 Connect from the app tier

Fastest path: ECS Exec into a running task (`aws ecs execute-command ... --interactive --command "/bin/sh"` — needs `enableExecuteCommand` on the service and SSM permissions on the task role: a deliberate small IAM+ECS side quest). From inside:

```sh
# DNS: resolve the RDS endpoint — notice it returns a PRIVATE IP (10.0.x.x)
nslookup mydb.xxxx.ap-south-1.rds.amazonaws.com
# Reachability: TCP handshake test, no psql needed
nc -zv <endpoint> 5432   # or: timeout 3 bash -c '</dev/tcp/HOST/5432' && echo open
```

That `nc -zv` is your core network troubleshooting primitive: it isolates "network path + SG" from "application/auth problem." If `nc` succeeds but the app can't connect → credentials/DB config. If `nc` times out → SG/NACL/route. If DNS fails → different problem entirely (Phase 5).

---

## Phase 5 — DNS, Route 53, TLS, ACM (30 min)

### 5.1 DNS mental model

DNS maps names → IPs via a delegation hierarchy: root → TLD (`.com`) → your zone's authoritative servers. Your resolver caches answers for the record's **TTL**. TTL discipline is a real ops skill: before a migration/cutover, drop TTL to 60s *a full old-TTL in advance*, cut over, verify, raise it back. Forget this and half the internet keeps hitting the old IP for hours.

### 5.2 Route 53 & the Resolver

Route 53 has three separable jobs: registrar, **authoritative DNS** (hosted zones), and health-checked routing policies (failover, weighted, latency).

The **Route 53 Resolver** is different: it's the *recursive resolver inside your VPC*, at `VPC_CIDR+2` (yours: `10.0.0.2`). Every instance uses it by default. It answers from private hosted zones, resolves AWS endpoints to private IPs where applicable, and forwards the rest to public DNS. **Resolver endpoints** (inbound/outbound) exist for hybrid DNS: outbound endpoints forward queries for `corp.internal` to on-prem DNS servers via rules; inbound endpoints let on-prem resolve your private zones. You know Zscaler environments — this is exactly the machinery enterprise DNS runs through.

**Build (works with no domain):** Route 53 → private hosted zone `lab.internal`, associated with your VPC. Records:
- `app.lab.internal` → **Alias** → your ALB
- `db.lab.internal` → CNAME → the RDS endpoint

From an ECS task: `nslookup app.lab.internal` — answered by 10.0.0.2, invisible to the outside world. **Alias vs CNAME** (interview staple): Alias is Route 53–proprietary, resolves at the zone apex (CNAME can't), free queries, tracks the target's IPs automatically.

**If you have a public domain:** public hosted zone, A/Alias record → ALB, and do 5.3 for real.

### 5.3 TLS, ACM, certificate lifecycle

TLS gives you three things: **encryption** (nobody reads it), **integrity** (nobody alters it), **authentication** (the cert proves the server is who it claims — signed by a CA your client trusts). Handshake in one breath: ClientHello (versions/ciphers) → server sends cert → client validates chain to a trusted root → key exchange → symmetric session keys → encrypted app data.

**Certificate lifecycle** = request → **domain validation** (ACM: you prove control by creating a specific DNS CNAME — Route 53 does it one-click) → issue → deploy (attach to ALB/CloudFront) → **renew** → revoke/replace. ACM's killer feature: public certs auto-renew *as long as the validation CNAME stays in place* — delete that record and renewal silently fails, which is the ACM-specific expiry incident. Constraint worth knowing: ACM certs are non-exportable, region-bound, and only attach to AWS-managed endpoints (ALB/NLB/CloudFront/API GW). CloudFront requires the cert in us-east-1 — classic exam/interview trap.

**Build (domain required):** ACM cert for `app.yourdomain.com` (DNS validation) → add HTTPS:443 listener on the ALB with the cert → change HTTP:80 listener to a 301 redirect to HTTPS. Inspect with:

```sh
openssl s_client -connect app.yourdomain.com:443 -servername app.yourdomain.com | openssl x509 -noout -dates -issuer -subject
```

`-servername` sets **SNI** — how one ALB serves many certs on one IP. Note **TLS terminates at the ALB**: ALB→task is plain HTTP inside the VPC (common, acceptable) unless you do end-to-end TLS (needed for strict compliance — your GxP world).

---

## Phase 6 — The Break-Fix Gauntlet (45–60 min) ★ the whole point

Rules: break one thing, **observe the symptom first**, write down your hypothesis, *then* diagnose with tools, fix, verify. The symptom→layer mapping you build here is the actual interview asset.

**Your toolkit:**
- `curl -v` (which layer failed: DNS? TCP connect? TLS? HTTP status?)
- `dig` / `nslookup` (DNS layer)
- `nc -zv host port` (pure TCP reachability)
- **VPC Reachability Analyzer** (VPC console → give it source ENI + dest ENI + port; it walks route tables, SGs, NACLs and names the exact blocking component — use it *after* forming your own hypothesis, as the answer key)
- **VPC Flow Logs** (enable on the VPC → CloudWatch Logs; ACCEPT/REJECT per flow. One REJECT = SG/NACL. Nothing at all = routing never delivered the packet)
- ECS stopped-task reasons, ALB target health reasons, CloudTrail `AccessDenied` events

| # | Sabotage | Predict the symptom, then verify | Layer |
|---|---|---|---|
| 1 | Delete `0.0.0.0/0 → NAT` from rt-private, force new ECS deployment | New tasks: `CannotPullContainerError`. **Running tasks keep serving** — inbound path via ALB doesn't need NAT. Sit with why. | Routing |
| 2 | Remove the `sg-alb → sg-app` inbound rule on sg-app | Targets drain to `unhealthy`, ALB returns **502/504**. Flow Logs show REJECTs from the ALB ENI IPs | SG |
| 3 | On sg-db, change source from sg-app to some random SG | App logs: **connection timeout** (not refused!) to :5432. `nc -zv` from task hangs. Timeout ≈ silent drop ≈ SG/NACL; "refused" ≈ reached the host, nothing listening | SG |
| 4 | Detach `AmazonECSTaskExecutionRolePolicy` from the execution role, force new deployment | Tasks fail to start; stopped reason cites ECR or CloudWatch Logs auth. CloudTrail shows the `AccessDenied` | IAM |
| 5 | Add NACL rule #90 on private subnets: DENY TCP 1024–65535 inbound | Everything breaks weirdly: outbound connections from tasks fail because **return traffic** lands on ephemeral ports and NACLs are stateless. The single best statefulness lesson available | NACL |
| 6 | In the private hosted zone, point `db.lab.internal` at a wrong IP | App fails with connection errors; `nslookup` shows the lie instantly. Lesson: **always check DNS before blaming the network** | DNS |
| 7 | Change target group health check path to `/nonexistent` | Tasks are perfectly fine, ALB marks all targets unhealthy → 503, ECS may kill/replace healthy tasks in a loop. "Healthy app, failing health check" is a real prod pattern | LB |
| 8 | Add an S3 bucket policy with explicit `Deny s3:PutObject` to the ALB logs bucket | ALB silently stops delivering logs (find the gap in Athena later). **Explicit deny beats every allow** | IAM resource policy |
| 9 | Disable "DNS resolution" on the VPC | `nslookup` of anything fails inside the VPC — you killed the Route 53 Resolver's service to your VPC. Total, dramatic, instructive | Resolver |
| 10 | (Domain users) Delete the HTTPS listener; or edit the validation CNAME and read ACM's renewal-eligibility warning | curl to https:// fails at **TLS layer** (connection refused on 443), not HTTP layer — read `curl -v` output closely to see the difference | TLS |

**After each fix, write one line:** *symptom → layer → tool that proved it.* Ten of those lines are a better interview cheat-sheet than any Q&A doc, because they're yours.

---

## Phase 7 — Athena + Glue on Your ALB Logs (20–30 min)

By now S3 has real ALB access logs. Athena = serverless SQL directly over S3 (pay per data scanned); Glue Data Catalog = the metadata layer telling Athena how those files map to table columns. No cluster, no load step — this is the "why" of both services.

Athena → set a query-results S3 location → run the AWS-documented `CREATE EXTERNAL TABLE` DDL for ALB logs (search "Athena ALB access logs" in AWS docs; it's a regex SerDe over the log format), pointing `LOCATION` at your log prefix. Then:

```sql
SELECT elb_status_code, count(*) FROM alb_logs GROUP BY 1 ORDER BY 2 DESC;
SELECT target_status_code, request_url, count(*) FROM alb_logs
WHERE elb_status_code >= 500 GROUP BY 1,2 ORDER BY 3 DESC;
```

You should literally see the 502s/503s from your Phase 6 sabotage. Infrastructure telemetry → S3 → SQL forensics: this is a genuinely senior workflow, and the DDL you ran *created a Glue Catalog table* — that's the Glue tie-in (a Glue *crawler* would have inferred the schema automatically; you did it manually).

---

## Phase 8 — Teardown (15 min, in this order)

1. ECS service → 0 tasks → delete service → delete cluster
2. ALB → target group → delete
3. RDS instance (skip final snapshot)
4. **NAT Gateway** → then **release the Elastic IP** (unattached EIPs bill!)
5. VPC endpoints, Route 53 private zone
6. VPC (sweeps subnets/RTs/IGW/SGs)
7. Keep or empty+delete the S3 buckets; drop the Athena table
8. Next day: check Billing → verify ~zero spend

---

## Appendix A — Transit Gateway Extension (~45 min, +$0.05/hr/attachment)

TGW exists because VPC peering doesn't scale: peering is non-transitive, so N VPCs need N(N−1)/2 meshes. TGW is a regional cloud router — attach each VPC once, hub-and-spoke.

**Mini-lab:** second VPC `10.1.0.0/16` with one private subnet + one EC2/task → create TGW → attach both VPCs → **two-hop routing** (the part everyone gets wrong): (1) each VPC's route table needs `10.x.0.0/16 → tgw-xxx` pointing at the *other* VPC's CIDR, AND (2) the *TGW's own route table* needs routes to each attachment (auto-propagated by default). Traffic must be routed **into** the TGW by the VPC and **across** it by the TGW table. Then `nc` between them, break one of the two hops, watch Reachability Analyzer name it. This matches the TGW material from your networking deep-dives — now with your hands on it.

## Appendix B — Lambda + API Gateway Extension (~30 min, ~free)

The serverless mirror of Phases 3: API Gateway (HTTP API) plays the ALB role, Lambda plays ECS, the **Lambda execution role** is the exact analog of the task role. Build: Lambda (Python, returns JSON) → HTTP API → route → invoke. Then break it the IAM way: API Gateway needs a **resource-based policy on the Lambda** permitting `lambda:InvokeFunction` (the console adds it silently — delete it and watch 500s). Bonus: put the Lambda *in your VPC* and discover it loses internet access until it's in a private subnet with the NAT route — the single most common Lambda networking surprise. Step Functions (orchestration of Lambdas as state machines with retries/branching) is a 15-min add-on if time remains.

## Appendix C — CloudFront Extension (~20 min, domain required)

CloudFront = CDN: edge locations cache/accelerate, TLS terminates at the edge, origin = your ALB. Key facts to internalize even if you skip the build: cert must live in **us-east-1**; cache behaviors decide what's cached vs passed through; a common architecture locks the ALB to accept traffic only from CloudFront (via custom header or managed prefix list) — SG-by-reference thinking again.

## Appendix D — MongoDB & Snowflake (talk track, no build)

Neither is AWS. **MongoDB**: document DB; on AWS you'd run Atlas (SaaS) reached via **VPC peering or PrivateLink** — so your networking answer is the same skills as this lab; AWS's native analog is DocumentDB. **Snowflake**: cloud data warehouse, separates storage from compute; connectivity again via **PrivateLink** in enterprise setups. Interview move: pivot any question about them to the connectivity/IAM/networking layer, where your depth actually is.

## Appendix E — Symptom → Layer Quick Reference

| Symptom | First suspect | Tool |
|---|---|---|
| Connection **timeout** | SG / NACL / route silently dropping | `nc -zv`, Flow Logs, Reachability Analyzer |
| Connection **refused** | Reached the host; nothing listening / wrong port | check the service, port mapping |
| DNS resolution failure | Resolver, records, VPC DNS settings | `dig` / `nslookup` |
| 502 / 504 from ALB | Backend unreachable or slow (SG between ALB and targets, dead app) | target health, sg-app rules |
| 503 from ALB | **No healthy targets** | health check config, path, thresholds |
| `CannotPullContainerError` | No route to registry (NAT/route/endpoint) or execution role | route tables, stopped reason |
| `AccessDenied` | IAM — identity or resource policy, or explicit deny | CloudTrail event history |
| Works inbound, fails outbound | NAT route missing, or stateless NACL ephemeral ports | route table, NACL rules |
| Intermittent per-AZ failures | One AZ's subnet/route/NAT misconfigured | check resources per-AZ |
| TLS handshake failure | Cert expired/mismatched/missing listener | `openssl s_client`, `curl -v` |
