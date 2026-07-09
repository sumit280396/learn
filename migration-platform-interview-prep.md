# Interview Prep: Migration-Focused Platform Engineer Role

**Read the JD's story first.** This team is executing three migrations simultaneously: (1) AWS accounts moving between Organizations, (2) observability moving Splunk → Google Cloud Observability + Google SecOps, (3) ongoing controlled network/security changes (Route 53, certs, TGW, DNS, Zscaler, WAF, SCPs). They're hiring someone to *execute changes safely in a live enterprise*. Every answer should radiate: sequencing, blast-radius thinking, parallel-run validation, rollback plans, change windows.

**Your honest-positioning scripts** (use these verbatim shapes for gap areas):
- *"I haven't executed an org-to-org migration end-to-end, but I've worked extensively with the components that break during one — RAM-shared Transit Gateways, SCPs, cross-account IAM — so I know exactly what the risk surface is. Here's how I'd sequence it..."*
- *"My observability depth is on the infrastructure side — log pipelines, agents, shipping — rather than the Splunk query layer. For a Splunk-to-Google migration my value is in the pipeline cutover and parallel-run validation."*
- *"I know Checkmarx as a category — SAST in the pipeline with quality gates — and I've integrated equivalent scanning stages in GitLab CI. The tool-specific workflow I'd pick up in days."*

---

# SECTION 1 — AWS TENANT MIGRATION (Org to Org) ★ highest priority

**Q1. Why would a company migrate AWS accounts from one Organization to another at all?**
Mergers/acquisitions and divestitures are the classic drivers: the acquired company's accounts must move into the parent's Organization for unified billing, security guardrails (SCPs), identity (IAM Identity Center), and compliance visibility. Other drivers: consolidating shadow-IT orgs, separating a business unit being sold, or restructuring after a failed landing-zone design. The alternative to moving accounts is migrating *workloads* into fresh accounts in the target org — cleaner but far more work; moving the account preserves everything inside it.

**Q2. How does an account actually move between Organizations, mechanically?**
Three steps: (1) the account **leaves the source org** — either the source management account removes it or the member account leaves itself; (2) it exists briefly as a **standalone account** — which requires it to have its own valid payment method, contact info, and support plan, because it's no longer under consolidated billing; (3) it **accepts an invitation** from the target org's management account. The account ID never changes — which is the single most important fact of the whole migration.

**Q3. Why is "the account ID doesn't change" so important?**
Because almost everything inside the account keeps working: resource ARNs, IAM roles, S3 bucket policies referencing the account, KMS key policies, existing cross-account trusts *by account ID*. What breaks is everything that depended on **org membership or org-scoped sharing** — that's the actual risk surface, and enumerating it is the senior-level answer (next question).

**Q4. What breaks when an account leaves an Organization? (the money question — know this list cold)**
- **SCPs stop applying instantly.** The account goes from "guardrailed" to "unrestricted" the moment it leaves. That's a compliance/security gap window — mitigations: minimize the window, pre-stage restrictive IAM permission boundaries, or pre-attach equivalent controls, and have the target org's OU + SCPs ready so the account lands directly into guardrails.
- **AWS RAM shares within the old org break** — shared Transit Gateway attachments, shared Route 53 Resolver rules, shared subnets. If this account consumed a TGW shared from a network account in the old org, its **connectivity dies**. This must be re-architected *before* the move (new attachments/shares in the target org, or temporary VPC peering as a bridge).
- **Consolidated billing, RI/Savings Plans sharing** — the account loses the benefit of the old org's discounts; cost per workload can jump overnight. Finance must be warned.
- **Org-level services detach:** org-wide CloudTrail no longer logs this account (audit gap — enable an account-local trail before moving), AWS Config aggregators, GuardDuty/Security Hub org membership, delegated administrator relationships, CloudFormation StackSets (service-managed), Control Tower enrollment, IAM Identity Center access (users lose SSO into the account — plan break-glass IAM access).
- **Route 53 Resolver rules / PHZ associations shared via RAM** — cross-account DNS resolution breaks with them.

**Q5. How would you plan and sequence an org-to-org account migration? (rehearse this as a monologue)**
1. **Discovery:** inventory every org-dependency of the account — RAM shares consumed/provided, SCPs currently constraining it, identity access paths, org-CloudTrail/Config/GuardDuty, RI/SP sharing, StackSets, marketplace subscriptions, support plan.
2. **Pre-stage the landing:** in the target org, prepare the destination OU, SCPs, IAM Identity Center assignments, org-CloudTrail/Config/GuardDuty auto-enrollment, and any RAM shares (TGW, resolver rules) it will need on day one.
3. **Bridge networking:** if connectivity depends on old-org RAM shares, build the replacement path first (attachment to the new org's TGW, or temporary peering) so there's no connectivity gap.
4. **Close the audit gap:** enable an account-local CloudTrail before leaving.
5. **Execute in a change window:** leave old org → standalone (payment/support pre-arranged) → accept new-org invite → move into target OU → verify SCPs apply → verify identity access → verify connectivity/DNS → verify logging.
6. **Validate & decommission:** run the workload's health checks, compare billing, then tear down bridge networking and old-org leftovers.
Rollback plan: the reverse invitation path — which is why you keep the old org's OU and shares intact until validation completes.

**Q6. What happens to IAM users, roles, and running workloads during the move?**
Nothing, directly — IAM is account-scoped, not org-scoped, and EC2/ECS/RDS keep running; the migration is a control-plane event. The outages come indirectly: severed RAM-shared networking, lost SSO access, and any application logic that calls org APIs (`organizations:Describe*`) or relies on `aws:PrincipalOrgID` conditions — **that condition is a big one**: S3/KMS policies using `aws:PrincipalOrgID` of the *old* org will start denying this account's principals the moment it leaves. Discovery must grep policies for the old org ID.

**Q7. Why might you migrate workloads to new accounts instead of moving the account?**
When the account carries baggage: unknown IAM sprawl, non-compliant history, root-email ownership problems, or the target org's landing zone (Control Tower) mandates vended accounts. Trade-off: workload migration is a much bigger project (data transfer, DNS cutover, new ARNs breaking hard-coded references) but yields a clean, compliant account. The account-move is faster but imports all technical debt. A senior answer names both and the deciding factors.

**Q8. How do you keep security posture during the "standalone window"?**
Minimize duration (minutes, not days — the leave and join are API calls done back-to-back in a runbook), pre-position IAM permission boundaries or a temporary restrictive inline policy on human roles, restrict who holds credentials during the window, monitor CloudTrail (the account-local trail) in real time, and schedule it in a low-activity window with a second engineer verifying each step. If asked "what if the invite fails" — the account is fully functional standalone; you pause, diagnose, and the rollback is re-inviting into the old org.

---

# SECTION 2 — OBSERVABILITY MIGRATION: Splunk → Google Observability & SecOps

**Q9. What are the pieces on each side? (vocabulary check)**
Splunk: **Universal Forwarders/Heavy Forwarders** ship data → **indexers** store it → **search heads** run **SPL** queries, dashboards, alerts, and correlation searches (in Splunk Enterprise Security for the SIEM use case). Google side splits into two products: **Google Cloud Observability** (formerly Stackdriver / Cloud Operations suite) = Cloud Logging, Cloud Monitoring, Cloud Trace, with the **Ops Agent** on VMs and **Log Analytics** (SQL over logs); and **Google SecOps** (formerly Chronicle) = the SIEM — massive-scale security telemetry, normalized into **UDM** (Unified Data Model), detections written in **YARA-L**. Operational logs go to Observability; security telemetry goes to SecOps.

**Q10. Why would an organization migrate off Splunk?**
Almost always cost: Splunk licenses primarily on **ingest volume**, and enterprise log volumes grow relentlessly; SecOps pricing decouples from raw ingest, and organizations already committed to Google (GCP workloads, or a Google-partnership strategy) consolidate. Secondary drivers: SecOps' retention model (long, cheap retention by default) and reducing self-managed indexer infrastructure.

**Q11. How would you plan a Splunk → Google migration? (rehearse as a monologue)**
1. **Inventory the demand side first:** every dashboard, alert, scheduled search, and detection rule in Splunk — and their *owners*. A large fraction is always dead; migrating garbage is the classic failure. Get owners to certify what they actually need.
2. **Inventory the supply side:** every log source and how it ships — forwarders, syslog, HEC (HTTP Event Collector), cloud integrations.
3. **Re-plumb ingestion per source type:** GCP-native sources flow to Cloud Logging automatically; VMs get the Ops Agent replacing the Universal Forwarder; AWS sources ship via native integrations/feeds into SecOps or via Pub/Sub pipelines; syslog devices repoint to a SecOps forwarder. For each source, decide: Observability (ops) or SecOps (security) or both.
4. **Translate the query/detection layer:** SPL doesn't port automatically. Dashboards/alerts rebuild in Cloud Monitoring / Log Analytics (SQL); Splunk ES correlation searches rebuild as **YARA-L** rules in SecOps against **UDM** fields — which requires validating that SecOps **parsers** normalize your log formats into UDM correctly (custom formats need custom parsers; this is the hidden workload of the whole migration).
5. **Parallel run:** dual-ship for a defined period (30–90 days typical). Compare event counts per source per day between systems (a simple daily reconciliation report), verify alerts fire in both, let the SOC operate in the new tool with the old as safety net.
6. **Cutover by source, not big-bang:** decommission Splunk ingestion source-by-source after each passes reconciliation; keep Splunk read-only until its retention window expires (or export cold data) for investigations/compliance.

**Q12. What's the hardest part of such a migration?**
Not the plumbing — the **semantic layer**: field mappings. Splunk's flexible schema-on-read fields vs SecOps' structured UDM means every detection rewritten must be re-validated against real data ("does `src_ip` in the old rule map to `principal.ip` in UDM for *this* log source?"). Second hardest: organizational — analysts fluent in SPL resisting a new query language. Parallel-run periods are as much retraining time as validation time.

**Q13. How do you validate nothing was lost?**
Volume reconciliation (events/source/day, both systems, automated diff with an alert on divergence >X%), alert-parity testing (replay or trigger known conditions, confirm both fire), detection coverage mapping (a spreadsheet: every old correlation search → its new YARA-L rule → test evidence), and retention verification (query a 6-month-old event in the new system). Deliverable framing: "an exit checklist per source, signed off before Splunk ingestion is cut."

**Q14. What is Google SecOps' UDM and why does it matter?**
UDM is the normalized schema every ingested log is parsed into — entities like `principal` (who acted), `target` (what was acted on), `network`, `metadata`. It matters because detections are written once against UDM and apply across all sources that parse into it — the opposite of Splunk's per-sourcetype field extractions. The cost: everything hinges on parser quality; a broken parser means silent detection blindness, which is why parser validation belongs in the migration's test plan.

---

# SECTION 3 — NETWORK & SECURITY CHANGE EXECUTION (Route 53, certs, TGW, DNS, Zscaler, WAF)

**Q15. Walk me through how you'd execute a production DNS cutover.**
TTL discipline: at least one full old-TTL *before* the window, lower the record's TTL (e.g., 3600 → 60). At the window: change the record, verify propagation with `dig` against multiple resolvers (authoritative NS directly, 8.8.8.8, the corporate resolver), watch traffic shift on the new target's metrics, keep the old target alive until traffic drains to ~zero (stragglers = resolvers ignoring TTL). After soak: raise TTL back. Rollback is trivial precisely *because* TTL is low — that's the point. If it's a Route 53 **alias**, note there's no TTL to manage on your side, but downstream cache behavior still applies to the resolved answers.

**Q16. Why is a certificate change risky, and how do you de-risk it?**
Because TLS failures are total and client-side — a bad cert doesn't degrade, it hard-fails every connection, and you can't fix clients' caches or pinned expectations. De-risk: verify the new cert's chain and SAN list *before* deployment (`openssl x509 -noout -text`), check for anything doing **certificate pinning** (mobile apps, service-to-service mTLS — the classic outage cause), deploy to one endpoint/listener first, validate with `openssl s_client` from inside and outside the network (a Zscaler-inspected path and a direct path can behave differently — see Q19), then roll out. Renewals with ACM should be non-events *if* DNS validation records were never deleted — auditing that validation CNAMEs still exist is a cheap preventive check worth mentioning.

**Q17. Describe the full certificate lifecycle and where it goes wrong.**
Request → domain validation (prove control: DNS CNAME or email) → issuance → deployment (ALB/CloudFront/API GW listeners, or servers) → monitoring → renewal → replacement/revocation. Failure modes in order of real-world frequency: (1) **silent renewal failure** — ACM can't renew because the validation CNAME was deleted, or a manually-installed cert simply expires unwatched (mitigation: expiry monitoring 30/14/7 days out, and AWS Config rule / Security Hub check for expiring ACM certs); (2) incomplete chain on manual installs (leaf without intermediate — works in some clients, fails in others: maddening); (3) SAN mismatch after a hostname change; (4) pinning breakage on rotation; (5) the CloudFront us-east-1 requirement surprising someone mid-change.

**Q18. What does a controlled TGW routing change look like in an enterprise?**
Never edit live route tables ad hoc. Process: model the change (which TGW route table, which attachment, propagated vs static, what's the longest-prefix interaction with existing routes), predict blast radius (every attachment associated with that route table is in scope), pre-write validation and rollback steps, execute in a window, validate with Reachability Analyzer + a pre-agreed matrix of `nc`/ping tests between representative endpoints, and watch flow logs for unexpected REJECTs or drops. Mention **blackhole routes** as the controlled kill switch and the **two-hop model** (VPC route table AND TGW route table) as the checklist you verify both sides of.

**Q19. What is Zscaler and how does it complicate network changes?**
Zscaler is a cloud security-edge platform: **ZIA** (Internet Access) is a cloud proxy all outbound user/server internet traffic routes through — filtering, DLP, and crucially **TLS inspection** (Zscaler terminates and re-signs TLS with its own CA, which every managed endpoint trusts); **ZPA** (Private Access) is the ZTNA VPN-replacement — users reach internal apps via App Connectors, based on identity, with no inbound network path. Complications for change work: (1) TLS inspection means servers/containers making outbound calls need the **Zscaler root CA in their trust store**, or they throw cert errors — the #1 "works on my machine, fails in prod" cause in Zscaler shops, including inside Docker images and CI runners; (2) some destinations must be **SSL-inspection-bypassed** (cert-pinned SaaS, package repos that break); (3) ZPA means "can user X reach app Y" is an *identity/policy* question, not a routing question — troubleshooting shifts from traceroute to Zscaler policy logs; (4) any new outbound dependency (new API endpoint, new repo) may need a Zscaler policy change — so enterprise change coordination includes the proxy team.

**Q20. How do you roll out a WAF rule change safely?**
**Count mode first, always.** AWS WAF rules can be set to Count instead of Block: deploy the new rule counting, watch CloudWatch metrics / WAF logs (to S3 or Firehose) for what it *would have* blocked over a representative traffic period (include a business cycle — a week if you can), review matches for false positives with app owners, tune (scope-down statements, exclusions), then flip to Block in a window with instant rollback = flip back to Count. For managed rule groups, same idea via rule-group override to Count. Never deploy straight-to-Block on production traffic; that sentence alone signals experience.

**Q21. A change window is approved for tonight touching DNS + a cert + a TGW route. How do you run it?**
Sequenced runbook with per-step validation and independent rollback: (1) pre-checks (TTLs already lowered days ago, cert chain pre-verified, current-state routing captured — `describe-route-tables` output saved as the rollback reference); (2) execute lowest-risk-first or dependency order, one change at a time, validating each before the next — never stack unvalidated changes, because when something breaks you won't know which change did it; (3) validation matrix run after each step (dig/openssl/nc probes from inside, outside, and through-Zscaler vantage points); (4) soak period; (5) comms — start/step/done in the change channel. If any validation fails: stop, roll back that step, do not proceed. Interviewers hiring for this JD are listening for exactly this discipline.

---

# SECTION 4 — SCPs & GOVERNANCE IN CONTROLLED ENVIRONMENTS

**Q22. What is an SCP, precisely?**
A Service Control Policy is an org-level policy attached to the root, an OU, or an account that defines the **maximum available permissions** for all identities in affected accounts — including the account root user. It grants nothing; it's a ceiling. Effective permission = intersection of SCP ∧ identity policy ∧ (resource policy where relevant) ∧ permission boundary, minus any explicit deny. SCPs don't affect the management account itself, and don't apply to service-linked roles — two classic trick-question facts.

**Q23. How do you change an SCP in a controlled enterprise environment without breaking things?**
SCP changes are high-blast-radius (every identity in every account under the OU) and there is **no native dry-run**. Process: (1) impact analysis — which accounts inherit this, what workloads call the affected actions (query CloudTrail/Athena across accounts for actual usage of the actions you're about to deny — data beats guessing); (2) prefer **deny-list style** additions to touching allow-list SCPs (safer diff); (3) test on a **sandbox OU first** — move a test account under an OU with the new SCP, run representative workloads; (4) roll out progressively — least-critical OU first, soak, then production OUs; (5) monitor CloudTrail for a spike in `AccessDenied` with the SCP as cause after each stage; (6) rollback = detach/revert the policy version, which is instant — but the *detection* of breakage is the slow part, hence staged rollout. Also: SCPs in Terraform/IaC with peer-reviewed MRs, never console edits — governance change control applies to the guardrails themselves.

**Q24. An application suddenly gets AccessDenied and its IAM policy is perfect. Diagnose.**
Walk the gates out loud: explicit deny anywhere? → SCP on the account's OU chain (invisible from inside the account — check from the management account) → permission boundary on the role → resource policy (bucket/key/queue) → session policy if assumed with one → VPC endpoint policy if the call went through an endpoint → for KMS specifically, the key policy. CloudTrail's event record often includes which policy type denied. The candidate who says "check the SCP and the VPC endpoint policy" gets hired; most people loop forever in IAM.

**Q25. How does governance/access management look at org scale?**
IAM Identity Center for human access (permission sets, group-based assignment, no long-lived keys), SCPs for guardrails (deny root actions, deny regions, deny disabling CloudTrail/GuardDuty, deny leaving the org), permission boundaries for delegated role-creation, org-wide CloudTrail + Config with conformance packs for detection, Access Analyzer for unintended external access, and everything as IaC through reviewed pipelines. One tight paragraph like that answers most "governance" questions.

---

# SECTION 5 — CORE AWS SERVICES (rapid but deep)

**Q26. Why does a VPC exist — what problem does it solve?**
Isolation and topology control: a private, software-defined network where nothing communicates until explicitly wired — your IP space, your subnets, your routing. Without it, no meaningful security boundary between workloads or tenants is possible.

**Q27. What makes a subnet public?**
One thing only: its associated route table has `0.0.0.0/0 → Internet Gateway`. Not a checkbox, not the name. (Plus, instances need a public IP for the IGW to translate — a public-subnet instance without a public IP is still unreachable.)

**Q28. Why do route tables matter and how do they evaluate?**
Every packet leaving a subnet consults exactly one route table; **longest prefix wins**. The immutable `local` route makes all intra-VPC subnets mutually routable — so intra-VPC connectivity problems are *never* routing, always SG/NACL. That deduction shortcut is a senior tell.

**Q29. Why NAT Gateway? What breaks without it?**
Private instances have no public IP, so they can't originate internet traffic even with an IGW route. NAT GW (in a public subnet, with an EIP) translates outbound many-to-one; unsolicited inbound can't enter. Without it: image pulls fail (`CannotPullContainerError`), patching fails, external APIs unreachable — while inbound-via-ALB traffic keeps working, which confuses people. Cost-aware add-on: S3/DynamoDB traffic should bypass NAT via gateway endpoints ($0.045/GB saved).

**Q30. Why Transit Gateway over VPC peering?**
Peering is point-to-point and non-transitive → N VPCs need N(N−1)/2 meshes and route-table sprawl. TGW is a regional hub router: attach once, route centrally, segment via multiple TGW route tables, extend to VPN/Direct Connect, share cross-account via RAM. Know the **two-hop routing model** (VPC RT routes *into* TGW; TGW RT routes *across*) and that cross-account SG-references don't work through TGW — CIDR-based rules instead.

**Q31. Route 53 vs Route 53 Resolver — distinguish them.**
Route 53 = authoritative DNS (hosted zones, records, routing policies: failover/weighted/latency) + registrar. The **Resolver** = the recursive resolver inside every VPC at CIDR+2 that instances actually query. Resolver **endpoints** (inbound/outbound) + **forwarding rules** (shareable via RAM) are how hybrid and multi-account DNS is built: outbound rules forward `corp.internal` to on-prem/other-VPC DNS; inbound endpoints let on-prem resolve your private zones. Alias vs CNAME: alias works at zone apex, free queries, AWS-target-aware.

**Q32. Why ACM?**
Managed cert issuance + **automatic renewal** (while the DNS validation record exists) + native attachment to ALB/NLB/CloudFront/API GW. Removes the expired-cert incident class. Constraints: non-exportable, region-bound, us-east-1 for CloudFront.

**Q33. Why CloudFront?**
Latency (edge caching + optimized backbone to origin even for dynamic content), origin offload, edge TLS termination, DDoS absorption (with Shield), and WAF attachment point. Enterprise pattern: lock the origin so only CloudFront can reach it (custom header or managed prefix list) — otherwise the CDN/WAF is bypassable.

**Q34. API Gateway vs ALB?**
ALB: L7 load balancer for long-lived backends in your VPC — cheap at volume, target groups, health checks. API Gateway: managed API front door — auth (IAM/Cognito/JWT), throttling, API keys, stages, request transformation — natural fit for Lambda. HTTP API vs REST API: HTTP is cheaper/simpler; REST has usage plans, API keys, request validation. Private APIs reach VPC backends via **VPC Link**.

**Q35. Why ECS, and the two-role distinction?**
Managed container orchestration without Kubernetes' operational weight; Fargate removes host management entirely. **Task execution role** = the ECS agent's permissions to *start* the container (pull from ECR, write logs, fetch secrets) — failure = task never starts. **Task role** = the app's runtime AWS permissions — failure = app runs, then AccessDenied. Troubleshooting flow: stopped-task reason → `CannotPullContainerError` = networking/NAT/endpoint or execution role; health-check kills = target group config or app.

**Q36. Why Lambda; what are its networking gotchas?**
Zero-infrastructure event-driven compute, per-ms billing, scales to zero. Gotchas: putting a Lambda **in a VPC** removes its default internet access — it now needs private subnets with a NAT route (most common Lambda networking surprise); cold starts; 15-min limit; the **resource-based policy** needed for services (API GW, S3, EventBridge) to invoke it — consoles add it silently, IaC authors forget it.

**Q37. Why Step Functions?**
Orchestration as configuration: retries with backoff, catch/branching, parallelism, and state passing between Lambdas/services *without* embedding orchestration logic in code. Debugging superpower: the execution graph shows each state's exact input/output. Standard vs Express: long-running exactly-once vs high-volume short-lived.

**Q38. Why Glue and Athena, and how do they relate?**
Glue Data Catalog = the metadata layer (tables, schemas, partitions) describing data sitting in S3; crawlers infer schema; Glue jobs do Spark ETL. Athena = serverless SQL that reads S3 *through* the catalog, priced per data scanned. Therefore the whole optimization story is: **partition** (prune scans) + **columnar formats** (Parquet) + compression. "ALB logs → S3 → Athena forensics" is a lived example you can cite from your labs.

**Q39. Why EBS; key properties?**
Network-attached block storage for EC2: persists independently of the instance, **AZ-locked** (hence: cross-AZ move = snapshot → restore; hence RDS Multi-AZ is replication, not shared disk), snapshots are incremental to S3, gp3 decouples IOPS/throughput from size. io2 for high-IOPS databases.

**Q40. Why RDS; Multi-AZ vs read replica?**
Managed relational DB: patching, automated backups/PITR, failover. **Multi-AZ** = synchronous standby, automatic failover via DNS flip of the endpoint (a Route 53 CNAME — connect the dots aloud), for availability, not read scaling. **Read replica** = async copy, for read scaling / cross-region, manual promotion. Connection troubleshooting: `nc -zv endpoint 5432` isolates network/SG from auth/app.

**Q41. S3 — what should a platform engineer emphasize?**
Object store, 11-nines durability, effectively infinite. Platform-relevant depth: **bucket policies as resource policies** (cross-account = both sides must allow), Block Public Access, encryption (SSE-S3/SSE-KMS — KMS adds key-policy as another access gate), lifecycle → storage classes, versioning + replication, **gateway endpoints** for private access, Object Ownership/bucket-owner-enforced (kills the cross-account object-ownership foot-gun), and event notifications feeding Lambda/Step Functions.

**Q42. MongoDB and Snowflake — how do you answer as an infra person?**
MongoDB: document/NoSQL; enterprise deployments use **Atlas** (SaaS) reached via VPC peering or **PrivateLink** — so your relevance is the connectivity, DNS (SRV records), and credential/secret management. Snowflake: cloud data warehouse, storage/compute separation, runs on cloud-provider storage (S3); connectivity via PrivateLink; data integration via **external stages** reading your S3 through an IAM role Snowflake assumes — a cross-account trust policy with an **ExternalId** condition (mention "confused deputy" — it lands). Pivot every question on these two toward networking, IAM, and TLS, where your depth is real.

---

# SECTION 6 — NETWORKING FUNDAMENTALS

**Q43. Explain DNS resolution end to end.**
Client asks its recursive resolver (in a VPC: the Route 53 Resolver at CIDR+2; in a Zscaler enterprise: possibly corporate DNS with conditional forwarders). Resolver walks the hierarchy — root → TLD → authoritative NS for the zone — caches the answer for its TTL, returns it. Failures cluster in three places: wrong record at the authoritative source, stale cache (TTL), or the client using a different resolver than you think (split-horizon: internal vs external answers differ by design — private hosted zones are exactly this).

**Q44. Stateful vs stateless — SG vs NACL.**
SG: ENI-level, stateful (return traffic auto-allowed), allow-only, all rules evaluated. NACL: subnet-level, stateless (return traffic needs its own rule — hence ephemeral ports 1024–65535), allow+deny, rules evaluated by number, first match wins. Symptom mapping: **timeout = silent drop = SG/NACL/route/blackhole; connection refused = reached the host, nothing listening; NXDOMAIN = stop, it's DNS.** Recite that triple in the interview — it's the distilled troubleshooting doctrine.

**Q45. Explain load balancing beyond "distributes traffic."**
Three functions: distribution across replicas, **health-based ejection** (the LB continuously probes and stops routing to failures — self-healing without human action), and decoupling (stable frontend name over churning backends: deploys, scaling, AZ loss). L7 (ALB) terminates HTTP and routes on content (path/host/header), adds `X-Forwarded-For`; L4 (NLB) forwards TCP/UDP flows, preserves source IP, static IPs, extreme throughput. Health check design is where incidents live: wrong path = healthy app marked dead (503s + restart loops); too-lenient thresholds = dead app kept in rotation.

**Q46. Explain the TLS handshake and what each part buys you.**
ClientHello (versions/ciphers/SNI) → server presents its certificate → client validates the chain up to a trusted root CA (authentication: the server is who it claims) → key exchange (ECDHE: ephemeral keys → forward secrecy) → symmetric session keys → encrypted application data (confidentiality + integrity). SNI = the hostname in the ClientHello enabling many certs on one IP. In a Zscaler shop add: ZIA re-signs with the Zscaler CA, so the cert a client sees inside the network is *not* the origin's cert — knowing this explains a whole class of "cert error only on corporate machines" tickets.

**Q47. Your systematic network troubleshooting method?**
Layer walk with the right tool per layer: DNS (`dig`/`nslookup` — and against *which* resolver), reachability (`nc -zv`, distinguishing timeout vs refused), path (route tables → both hops if TGW → Reachability Analyzer as answer key), filtering (SG → NACL → flow logs for REJECT vs absence), TLS (`openssl s_client -servername` — chain, dates, SNI), application (`curl -v`, status codes, LB access logs). Plus the two meta-questions: *what changed?* (CloudTrail/change records) and *from which vantage points does it fail?* (one caller = scoped SG/policy; everywhere = shared dependency).

---

# SECTION 7 — TERRAFORM

**Q48. Why Terraform / why IaC at all?**
Reproducibility (environments from code, not memory), review (infrastructure changes as MRs with diffs — `terraform plan` *is* the diff), drift detection, and disaster recovery as `apply`. Declarative model: you state the goal; Terraform diffs desired vs state vs reality and computes the DAG-ordered change set.

**Q49. Explain state — why it exists, and remote state discipline.**
State maps config resources to real-world IDs; without it Terraform can't know what it manages or compute diffs. Team discipline: remote backend (S3) + **locking** (DynamoDB table, or S3 native lockfile in newer versions) to prevent concurrent applies corrupting state; state contains secrets → encrypt (SSE-KMS) and restrict access as sensitive data; never hand-edit — use `terraform state mv/rm` and `import`. Per-env isolation: separate state files (directory-per-env or workspaces — prefer directories for prod/nonprod hard separation, and be ready to defend that).

**Q50. What makes a *good* Terraform module (the JD says "reusable, compliant, standardized")?**
Single responsibility with an opinionated happy path; a small variable surface with validated inputs (`validation` blocks, sensible defaults) rather than 80 pass-through variables; outputs that downstream modules actually need; **compliance baked in as non-overridable defaults** (encryption on, public access blocked, mandatory tags via `default_tags`/tag variables) — the module *is* the policy; semantic versioning published to a registry (GitLab's Terraform module registry fits this JD) so consumers pin versions and upgrades are deliberate; examples/ and tests (terraform test / Terratest) in the repo; and pre-commit hooks: fmt, validate, tflint, and a security scanner (tfsec/checkov) so non-compliant plans fail in CI before review.

**Q51. Common state incidents and fixes?**
Drift (console change): `plan` shows it; decide adopt (`import`/code change) vs revert (apply). Resource created outside TF: `import` + write matching config. Refactor moving resources between modules: `terraform state mv` (or `moved` blocks — the modern, reviewable way) to avoid destroy/recreate. Locked state after a crashed run: `force-unlock` with the lock ID, after confirming no run is live. Deleted state file: restore from S3 versioning (which you enabled, because you always version the state bucket — say this).

**Q52. How does Terraform order operations?**
It builds a dependency graph (DAG) from references (`aws_subnet.a.vpc_id` implies subnet-after-VPC) plus explicit `depends_on`, applies independent branches in parallel, and reverses the order on destroy. Circular references are a design smell — restructure, usually by extracting the shared dependency.

---

# SECTION 8 — GITLAB CI/CD

**Q53. Design a production-grade GitLab pipeline for a containerized service.**
Stages: **validate** (lint, fmt, unit tests, in parallel) → **build** (Docker image, tagged with commit SHA, pushed to registry) → **scan** (SAST via Checkmarx/GitLab SAST, dependency/SCA scan, container image scan (Trivy), secret detection, IaC scan — all as gates) → **deploy nonprod** (auto) → **integration tests** → **deploy prod** behind a **manual approval** (`when: manual` + protected environment with required approvers) → post-deploy smoke tests with automated rollback on failure. Cross-cutting: caching (dependencies) vs artifacts (build outputs passed between stages — know the difference), `rules:` to control when jobs run (MR vs main vs tag), protected variables for prod credentials, and **OIDC federation to AWS instead of stored keys** — `id_tokens` in the job assume an IAM role via a trust policy conditioned on the GitLab project/branch. That last one is a strong senior differentiator: no long-lived AWS keys in CI variables.

**Q54. How do you optimize a slow pipeline?**
Measure first (GitLab shows per-job durations). Then: parallelize independent jobs within a stage (or use `needs:` for DAG pipelines that ignore stage barriers), cache dependency directories keyed on lockfile hash, use Docker layer caching / buildkit and multi-stage builds so unchanged layers skip, split test suites with `parallel:`, use `rules:changes` to skip jobs irrelevant to the diff (monorepos), and right-size runners. Typical story arc to tell: "45 min → 12 min by DAG-ifying with needs:, caching node_modules on lockfile hash, and layer-caching the base image."

**Q55. How do security controls integrate into the pipeline (JD: "testing, security scanning, deployment controls")?**
Shift-left as gates, not reports: SAST (Checkmarx) on every MR with a policy threshold (fail on new high/critical), SCA for vulnerable dependencies, secret detection pre-merge, container scan before registry push, IaC scan before plan/apply; then **deployment controls**: protected branches (only maintainers merge to main), protected environments (prod deploys require named approvers), manual gates, and audit trail via pipeline history. Key nuance interviewers probe: **how you handle findings** — triage workflow, false-positive marking with justification, severity-based SLAs, and a break-glass exception process with expiry, rather than either "block everything" or "dashboard nobody reads."

**Q56. Terraform in GitLab CI — the standard pattern?**
MR pipeline: `fmt -check` → `validate` → tflint/checkov → `plan`, with the plan output posted to the MR for human review (the plan IS the change review). Merge to main: `apply` from the *saved plan artifact* (apply exactly what was reviewed — mention `-out=planfile`; it's a precision point most miss), remote state with locking, OIDC-assumed role scoped least-privilege, and environment-specific pipelines from directory structure.

---

# SECTION 9 — DOCKER

**Q57. Image vs container; what actually is a container?**
Image = immutable layered filesystem + metadata; container = a running process (or tree) on the host kernel, isolated by **namespaces** (pid/net/mnt/user — what it can see) and limited by **cgroups** (CPU/memory — what it can use), with an overlay filesystem stacking the image layers plus a writable layer. "It's a process, not a VM" unlocks all Docker troubleshooting: host tools (`ps`, `nsenter`, host kernel logs) apply.

**Q58. What makes a good production Dockerfile?**
Multi-stage builds (build tools in stage one, runtime-only final image), minimal base (slim/distroless/alpine — with the glibc-vs-musl caveat for alpine), layer-cache-aware ordering (copy dependency manifests → install → then copy source, so code changes don't bust the dependency layer), non-root USER, pinned base versions, no secrets in layers (they persist in history even if "deleted" later — use build secrets/BuildKit), a HEALTHCHECK, and `.dockerignore`. Enterprise addition from Q19: corporate CA certs (Zscaler) baked into images that make outbound TLS calls through inspection.

**Q59. Container troubleshooting flow?**
`docker logs` (crashed?) → `docker inspect` (exit code: **137 = OOM-killed/SIGKILL — check cgroup memory limit; 1 = app error; 126/127 = entrypoint/command problems**) → `docker exec` shell in (or `docker run --entrypoint sh` if it dies instantly) → resource limits (`docker stats`) → networking (`docker network inspect`, is the app listening on 0.0.0.0 not 127.0.0.1 — the classic "port mapped but unreachable" cause) → DNS inside the container. On ECS the same flow starts from the stopped-task reason.

---

# SECTION 10 — LINUX ADMINISTRATION & PERFORMANCE

**Q60. Your systematic approach to "the server is slow"?**
The **USE method** (Brendan Gregg): for every resource — CPU, memory, disk I/O, network — check **U**tilization, **S**aturation, **E**rrors. Concretely: `uptime` (load vs core count — load includes tasks in uninterruptible I/O wait, not just CPU), `vmstat 1` (r column = run-queue saturation; si/so = swapping = memory crisis), `free -h` (distinguish "used" from cache — available is the number that matters), `iostat -xz 1` (%util and await = disk saturation), `sar -n DEV 1` / `ss -s` (network), `top`/`pidstat` to attribute to processes, `dmesg -T` for errors (OOM-killer lines!). Saying "USE method" by name, then walking it, is a senior differentiator you already have loaded.

**Q61. High iowait — what does it mean and what next?**
CPUs idle *waiting on disk*. Next: `iostat -xz 1` — which device, what await/queue depth; `pidstat -d` — which process; then why: undersized EBS IOPS (gp2 burst-balance exhaustion is the classic AWS version — CloudWatch BurstBalance metric), a query doing full scans, swapping (memory problem masquerading as disk), or a backup job. Fix matches cause: gp3 with provisioned IOPS, query/index fix, more memory.

**Q62. A process was killed mysteriously overnight.**
`dmesg -T | grep -i oom` / `journalctl -k` — the OOM killer logs its victim and the memory state. Then: was it the process's leak or system pressure; check cgroup limits if containerized (ECS memory limit = same mechanism); mitigations: fix the leak, raise limits consciously, adjust `oom_score_adj` for critical daemons, add swap only as a deliberate choice.

**Q63. Disk full — walk through it.**
`df -h` (which filesystem) → `du -xh --max-depth=1 /` narrowing down (`-x` stays on one filesystem) → usual suspects: logs without rotation (`/var/log`, fix logrotate), package caches, Docker (`docker system df`, prune). Trick scenario: `df` says full but `du` can't find it → **deleted files still held open** by a process (`lsof +L1` / `lsof | grep deleted`) — restart or signal the holder; the space frees when the fd closes. That one earns respect.

**Q64. systemd essentials for a platform engineer?**
`systemctl status/start/enable`, `journalctl -u service -f --since`, unit files (ExecStart, Restart=on-failure, resource controls — `MemoryMax` is cgroups again), targets vs runlevels, `systemd-analyze blame` for boot issues, drop-in overrides (`systemctl edit`) instead of editing vendor units.

---

# SECTION 11 — SHELL SCRIPTING

**Q65. What separates a production script from a throwaway?**
`set -euo pipefail` (exit on error, unset-var error, pipeline failure propagation) — and knowing its sharp edges; quoting everything (`"$var"`); `trap cleanup EXIT` for temp files/locks; idempotency (safe to re-run — check-then-act, `mkdir -p`, upserts); explicit logging with timestamps to stderr; meaningful exit codes; no secrets in args (visible in `ps`) — env vars or files; `shellcheck` in CI. Then the judgment answer: beyond ~100 lines or when it needs real data structures/error handling, switch to Python — knowing when *not* to shell-script is part of scripting proficiency.

**Q66. Give a real automation example you can defend.**
Have one ready from your work — shape: "wrapper for safe operations": pre-flight checks → dry-run mode printing intended actions → confirmation gate → execution with per-item error handling that continues and reports failures at the end → summary + non-zero exit if anything failed. E.g., a script that snapshots EBS volumes by tag, verifies snapshot completion, prunes ones older than N days with a dry-run default. Be ready to sketch its skeleton on a whiteboard.

---

# SECTION 12 — SECURITY, CHECKMARX, GOOGLE SECOPS

**Q67. What is Checkmarx and how does it fit a pipeline?**
Checkmarx is an application-security platform, best known for **SAST** (static analysis of source code for vulnerability patterns: injection, XSS, path traversal, hard-coded secrets) — modern platform "Checkmarx One" also bundles SCA (dependency vulns), IaC scanning (KICS), and API security. Pipeline integration: a scan stage in GitLab CI on MRs and main, results gated by policy (e.g., fail the pipeline on new criticals — "new" matters: baseline existing debt, block regressions), findings pushed to the developer's MR. The mature-process part interviewers actually probe: **triage** — confirming true positives, marking false positives *with documented justification* (SAST FP rates are real), severity SLAs, and exception workflows with expiry dates. If asked directly about hands-on: use the honest script — you know the category and the pipeline mechanics; the tool UI is a fast learn.

**Q68. SAST vs DAST vs SCA vs container scanning — one line each.**
SAST: reads source/bytecode without running it — finds code flaws early, has false positives. DAST: attacks the running app from outside — finds real exploitable behavior, later in cycle. SCA: inventories dependencies against CVE databases — finds the log4shell class. Container scan: OS packages + layers in the image (Trivy/ECR scanning). Defense in depth = all four at different pipeline stages, plus secret detection.

**Q69. What does "supporting Google SecOps workflows" likely mean for this role?**
Not being a SOC analyst — being the platform side: keeping **telemetry pipelines** healthy (feeds/forwarders from AWS into SecOps: CloudTrail, VPC flow logs, GuardDuty findings, DNS logs), onboarding new log sources (ingestion config + parser/UDM validation), managing the infrastructure-as-code around forwarders and feeds, and responding when detection engineers say "we're blind on source X" — which is a log-pipeline troubleshooting task, squarely your skill set. Frame it that way proactively and you turn a gap into a strength.

**Q70. How do you integrate security controls into engineering processes generally (JD language)?**
Principle: controls as **paved road, enforced in the pipeline**, not as after-the-fact review. Concretely: compliant-by-default Terraform modules (Q50), scanning gates (Q55), OIDC instead of static credentials, protected environments and approvals for prod, org guardrails via SCPs (Q22–25), drift/config detection (AWS Config), and vulnerability findings routed as tickets with SLAs rather than PDFs. One sentence to land: "the goal is making the secure path the easy path — engineers shouldn't have to opt in to compliance."

---

# SECTION 13 — SCENARIO / BEHAVIORAL (they WILL ask these — prepare stories)

**Q71. "Tell me about a complex change you executed in production."** Use STAR with a real change from your EKS/Domino support work: the plan, the validation matrix, the rollback you prepared (even if unused), the communication. Emphasize *the discipline*, not the technology.

**Q72. "Tell me about a production issue you troubleshot under pressure."** Pick one with a layered diagnosis (symptom → hypothesis → tool → root cause). Your PV/PVC or stuck-workspace MongoDB-desync stories fit — end with what you changed to prevent recurrence.

**Q73. "A migration step fails midway through a change window. What do you do?"** Stop the sequence (never stack changes on a failure), execute the pre-written rollback for that step, verify rollback restored known-good state, communicate status, then diagnose offline. The answer is the *discipline*, and it's the answer this JD is screening for.

**Q74. "How do you coordinate a change that spans teams (network, security, app owners)?"** Change ticket with explicit scope/blast radius, named validation owners per dependency, a shared runbook with per-step owners, a comms channel live during the window, and pre-agreed go/no-go criteria. Reference ServiceNow-style change management from your enterprise background.

**Q75. "What would your first 30 days look like on this team?"** Map the estate (org structure, TGW/DNS topology, pipeline inventory), shadow one change end-to-end, take ownership of a small well-bounded migration task, and read the runbooks — improving one as you go. Signals: humility + bias to safe action.

---

# ONE-PAGE CRAM SHEET (read this at breakfast)

- **Org migration:** account ID never changes (ARNs survive); SCPs stop at the door (security gap window); RAM shares break (TGW! Resolver rules!); `aws:PrincipalOrgID` conditions break; pre-stage the landing OU; account-local CloudTrail before leaving; bridge networking before cutting.
- **Splunk→Google:** inventory dashboards/detections first (kill the dead ones); Ops Agent replaces forwarders; SecOps = Chronicle, UDM schema, YARA-L rules, parsers are the hidden work; parallel-run with daily volume reconciliation; cut over per-source.
- **AccessDenied gates:** explicit deny → SCP → identity policy → permission boundary → resource policy → VPC endpoint policy → (KMS key policy).
- **Symptom triple:** timeout = silent drop (SG/NACL/route/blackhole/SCP) · refused = host reached, nothing listening · NXDOMAIN = it's DNS.
- **TGW:** two hops — VPC RT into it, TGW RT across it; RAM to share; blackhole = kill switch; no cross-account SG refs → CIDR rules → CIDR planning matters.
- **Certs:** lifecycle = request→validate→issue→deploy→renew; ACM auto-renews only while the validation CNAME lives; CloudFront cert in us-east-1; pinning breaks rotations; Zscaler re-signs TLS (its CA must be in trust stores — including containers and CI runners).
- **DNS change:** lower TTL one full old-TTL early; dig against multiple resolvers; keep old target until traffic drains.
- **WAF:** count mode → analyze real traffic → tune → block. Never straight to block.
- **SCP change:** no dry-run exists → CloudTrail/Athena usage analysis → sandbox OU → staged rollout → watch AccessDenied rates → instant detach as rollback.
- **Pipeline:** validate → build → scan (gated) → deploy nonprod → test → manual-gated prod; OIDC to AWS, no stored keys; apply the saved plan artifact.
- **Terraform module:** small validated inputs, compliance as defaults, semver in a registry, tests + tfsec in pre-commit/CI.
- **Linux:** USE method by name; dmesg for OOM; lsof +L1 for phantom disk usage; load counts D-state.
- **Docker exit codes:** 137 OOM · 1 app · 126/127 entrypoint.
- **When you don't know:** "I haven't run that specific migration, but here's the risk surface I *do* know and how I'd sequence it" — then actually sequence it. Approach beats memorized facts for this role.
