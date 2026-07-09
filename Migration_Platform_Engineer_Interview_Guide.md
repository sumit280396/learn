# Complete Interview Prep Guide — Platform Engineer (Migration-Heavy Role)
## Scenario & Troubleshooting Questions with Zero-Assumed-Knowledge Answers

**How to read this guide:** Each section starts with a "Mental Model" — the core idea that makes everything else make sense. Read that first. Then every question follows the pattern: *the concept explained simply → the answer you'd give in the interview*. Answers are written so you can speak them naturally, not memorize them.

---

# PART 1: MIGRATION EXPERIENCE (The Heart of This Role)

This JD is screaming one thing: **they are mid-migration or about to start one**. AWS Org-to-Org migration + Splunk-to-Google observability migration + network change execution. Expect 40-50% of the interview here.

## 1.1 AWS Organization / Tenant Migration

### Mental Model
An **AWS Organization** is a container for many AWS accounts. One account is the **management account** (pays the bills, sets policies); the rest are **member accounts**. Big companies use one account per team/environment (prod, dev, security, networking). "Tenant migration from one Organization to another" means: a company (often after an acquisition, divestiture, or restructure) needs to move member accounts out of Org A and into Org B — while workloads keep running.

The critical insight: **moving an account between Orgs doesn't move any workloads** — the EC2 instances, VPCs, and data all stay exactly where they are. What changes is everything *attached to the Org*: consolidated billing, SCPs (guardrail policies), centralized services (SSO, GuardDuty, Config), cross-account trust, and shared networking (RAM shares like Transit Gateway attachments).

### Q1. "Walk me through how you would plan an AWS account migration from one Organization to another."

**Answer structure (say it in phases):**

**Phase 1 — Discovery & inventory.** Before touching anything, build a dependency map of everything the account inherits from the old Org:
- **SCPs** applied to the account (what guardrails will disappear or change?)
- **Identity**: Is access via AWS IAM Identity Center (SSO) from the old Org's management account? That breaks the moment you leave.
- **Shared resources via AWS RAM** (Resource Access Manager): Transit Gateway attachments, shared subnets, Route 53 Resolver rules shared from a central networking account. RAM shares are Org-scoped — leaving the Org can invalidate them.
- **Centralized security services**: GuardDuty, Security Hub, AWS Config aggregators, CloudTrail org trails — all administered from the old Org.
- **Billing**: Reserved Instances and Savings Plans are shared across the old Org's consolidated billing; the account loses that benefit on exit.
- **Networking**: Does traffic flow through a central egress/inspection VPC owned by the old Org?

**Phase 2 — Design the target state.** Map every dependency to its equivalent in the new Org: which OU (organizational unit) the account lands in, which SCPs apply there, new SSO permission sets, new RAM shares for TGW, re-enrolling in the new Org's security tooling.

**Phase 3 — Pre-migration prep.** Create break-glass IAM users with long-lived credentials *inside the account itself* (because SSO access will break during the move). Convert Org-dependent configurations to account-local ones temporarily. Communicate change windows.

**Phase 4 — Execution.** The mechanical steps are simple: the account leaves Org A (removed by Org A's management account or self-leaves), exists briefly as **standalone** (must have its own payment method and support plan attached — a common gotcha), then accepts an invitation from Org B's management account, and is moved into the right OU.

**Phase 5 — Post-migration validation.** Verify: SCPs from the new OU apply as expected, SSO works via the new Identity Center, TGW attachments re-shared via new RAM shares are healthy, CloudTrail/GuardDuty/Config re-enabled under the new delegated admin, billing shows in the new consolidated bill, and application health checks pass end-to-end.

**Key line to say:** "The account move itself is minutes; the real work is the dependency inventory and the re-plumbing of everything Org-scoped — identity, guardrails, shared networking, and security tooling. Workloads never move; their governance context does."

### Q2. "What breaks the moment an account leaves its old Organization?"

Fire these off confidently:
1. **SSO / Identity Center access** — permission sets from the old management account stop working. This is why break-glass IAM access must exist beforehand.
2. **SCPs stop applying** — the account is briefly *ungoverned*. If an SCP was blocking dangerous actions (e.g., disabling CloudTrail, leaving regions open), that protection is gone until the new Org's SCPs attach. Security teams care deeply about minimizing this window.
3. **RAM shares scoped to the Org** — shared Transit Gateway, shared subnets, shared Resolver rules can be invalidated. If your TGW attachment came from an Org-wide share, you need account-specific (ARN-based) shares set up first, or connectivity drops.
4. **Org CloudTrail / Config / GuardDuty / Security Hub** — the account falls out of centralized logging and threat detection. Compliance gap.
5. **Consolidated billing, RI/Savings Plan sharing, volume discounts** — cost impact.
6. **Service quotas / support plan** — a standalone account needs its own support plan; also verify quotas in the destination context.

### Q3. "How do you keep Transit Gateway connectivity alive during an Org migration?"

**Concept first:** A TGW usually lives in a central *networking account* and is shared to spoke accounts using AWS RAM. RAM lets you share with (a) the whole Organization, (b) an OU, or (c) specific account IDs. If the share was "whole Organization," an account leaving the Org loses eligibility.

**Answer:** "Before the move, I'd change or supplement the RAM share to target the **specific account ID** rather than the Org. Account-ID-based shares survive Org membership changes (the recipient just has to accept the invite since auto-accept only works inside the same Org). I'd verify the TGW attachment stays in `available` state, and test cross-VPC traffic before, during, and after the move. If the destination Org has its own TGW, the longer-term work is a planned re-attachment: create the new attachment, update route tables on both sides, shift traffic, then remove the old attachment — ideally with overlapping connectivity so there's no hard cutover."

### Q4. "How would you migrate workloads if the account itself *can't* move (e.g., the account must stay behind)?"

Then it's a **resource-level migration** — much bigger:
- **Data**: S3 (cross-account replication or `aws s3 sync` with bucket policies), EBS snapshots shared cross-account, RDS snapshot share + restore, or DMS for minimal downtime.
- **Infra**: If everything is in Terraform, re-point the provider at the new account and re-apply; import or recreate. This is where "reusable Terraform modules" pays off — same modules, new account.
- **Cutover**: DNS-based. Stand up the new environment, validate, lower Route 53 TTLs in advance, flip records (or use weighted routing for gradual shift), monitor, then decommission.
- **Identity/secrets**: Recreate IAM roles, KMS keys (data encrypted with old-account KMS keys needs re-encryption or key policy grants), secrets in Secrets Manager.

**Key line:** "Account moves relocate governance; resource migrations relocate workloads. The first is a dependency-mapping exercise; the second is a data-gravity and cutover exercise — and DNS with pre-lowered TTLs is the steering wheel for cutover."

### Q5. "What's your rollback plan for an account migration?"

"Rollback for the *account move* is symmetrical: leave the new Org, rejoin the old one — so I keep the old Org's invitation path and OU placement documented and don't decommission old-Org constructs (SCP structure, RAM shares, SSO assignments) until a defined bake period passes. For *network changes* bundled with the migration, every change gets its own rollback: DNS changes roll back fast because we pre-lowered TTLs; route table changes are captured as before/after states in Terraform so rollback is `terraform apply` of the previous state; certificate changes keep the old cert valid until the new one is proven. The principle: never make an irreversible change during the migration window itself."

---

## 1.2 Observability Migration: Splunk → Google Observability & Google SecOps

### Mental Model
**Splunk** is a log analytics platform: agents (Universal Forwarders) or HTTP endpoints (HEC — HTTP Event Collector) ship logs in; users search them with **SPL** (Search Processing Language); dashboards and alerts are built on saved searches. Enterprises use it for both **operational logs** (app errors, latency) and **security logs** (SIEM use case: auth events, firewall logs, threat detection).

**Google Cloud Observability** (formerly Stackdriver) = Cloud Logging + Cloud Monitoring + Trace. Logs land in Cloud Logging, are queried with its query language, routed with **log sinks**, and metrics/alerts live in Cloud Monitoring.

**Google SecOps** (formerly Chronicle) = Google's SIEM. It ingests security telemetry, normalizes it into **UDM (Unified Data Model)**, and runs detection rules written in **YARA-L**. So a Splunk exit typically splits into two streams: *operational* logs → Google Observability, *security* logs → Google SecOps.

Why migrate? Almost always **cost** (Splunk licenses by ingest volume and is famously expensive) plus platform consolidation onto Google.

### Q6. "How would you plan a Splunk to Google Observability/SecOps migration?"

**Answer in phases:**

**1. Inventory what Splunk is actually doing.** Three inventories:
- **Sources**: every forwarder, HEC token, syslog feed, cloud integration — what's sending data, how much volume (GB/day), which indexes.
- **Consumers**: dashboards, scheduled searches, alerts, and — critically — *who* uses them. Many are dead; usage analysis lets you migrate 20% of assets that deliver 90% of value.
- **Detections** (security side): correlation searches / SIEM rules that must be rebuilt in SecOps.

**2. Classify each source: operational vs security.** App logs, infra metrics → Cloud Logging/Monitoring. Auth logs, EDR, firewall, DNS security telemetry, CloudTrail → Google SecOps.

**3. Re-plumb ingestion.**
- On GCP: native (Ops Agent, built-in service logs).
- On AWS/on-prem: forwarders repointed, or an aggregation layer. This is where I'd highlight **using a pipeline layer** (e.g., Fluent Bit / Vector / existing syslog aggregation) so sources are decoupled from destinations — you flip the destination once instead of touching every source.
- SecOps side: Google SecOps **forwarders** and **feeds** (it has native feed integrations for common sources like CloudTrail via S3/SQS), mapping each source to the right log type so UDM parsing works.

**4. Dual-ship (run both) during transition.** Send to Splunk *and* Google in parallel for a validation window. Compare event counts, verify parsing, rebuild and A/B the critical alerts and detections. Never hard-cut observability — you'd be flying blind exactly when risk is highest.

**5. Rebuild the consumption layer.** SPL searches → Cloud Logging queries / SecOps searches; dashboards → Cloud Monitoring dashboards; Splunk alerts → Monitoring alerting policies; SIEM correlation rules → YARA-L detections in SecOps. Prioritize by usage.

**6. Cutover & decommission.** Once parity is validated per source, stop the Splunk feed for that source (reduces license cost immediately — do it source-by-source, not big-bang), keep Splunk read-only for historical search until retention requirements expire, then decommission.

**Key line:** "The trap in observability migrations is treating it as a data-plumbing problem. The plumbing is the easy half. The hard half is the consumption layer — dashboards, alerts, and detections people rely on — and the answer there is usage-driven triage plus a dual-ship validation window so we never lose visibility."

### Q7. "How do you validate that no logs are being lost during the migration?"

"Three checks. First, **volume reconciliation**: compare events-per-source-per-hour between Splunk and the Google side during dual-shipping; sustained deltas mean a broken source. Second, **canary events**: inject synthetic marker events at known intervals at each source and alert if they don't arrive in the destination within an SLA — this catches silent pipeline failures. Third, **end-to-end alert testing**: trigger conditions that should fire rebuilt alerts/detections and confirm they do. On the security side, missing logs is a compliance issue, not just an ops issue, so I'd document the reconciliation results as evidence."

### Q8. "What are the hard parts of moving SIEM detections from Splunk to Google SecOps?"

- **Language translation**: Splunk correlation searches are SPL; SecOps detections are **YARA-L** running over **UDM**-normalized events. It's not find-and-replace — the data model differs, so each rule needs its logic re-expressed against UDM fields and re-tested.
- **Parsing/normalization**: SecOps depends on parsers mapping raw logs into UDM. If a source's parser is missing or incomplete, detections silently see nothing. Validate UDM field population per source *before* trusting detections.
- **Historical context**: detections that rely on baselines or lookups need those rebuilt.
- **Testing**: replay known-bad events (or red-team style test events) through the new pipeline and verify each migrated detection fires. Detection parity must be *proven*, not assumed.

### Q9. "Splunk searches are down to be decommissioned but the security team says they still need 13 months of historical data. What do you do?"

"Retention requirements don't migrate with the pipeline. Options, in order of preference: (1) keep Splunk in a **read-only, frozen-ingest** mode until old data ages out — no new license cost for ingest, but infra cost remains; (2) export historical indexes to cheap object storage (S3/GCS) in a searchable-on-demand form and document the retrieval procedure; (3) backfill critical historical security data into SecOps if feasible. The decision is compliance-driven — I'd get the retention requirement in writing and cost the options."

---

## 1.3 Network & Security Change Execution (Route 53, Certificates, TGW, DNS, Zscaler, WAF)

### Mental Model
This bullet describes a person who can **execute risky network changes in production without breaking things**: DNS cutovers, cert rotations, TGW route changes, Zscaler policy updates, WAF rule deployments — under change management (CAB approvals, change windows, rollback plans). The interviewer wants to hear *discipline*: pre-checks, staged rollout, observation, rollback readiness.

**Zscaler quick primer** (in case it comes up cold):
- **ZIA (Zscaler Internet Access)**: cloud proxy for *outbound* internet traffic. User/server traffic is steered to Zscaler's cloud (via agent, PAC file, or GRE/IPsec tunnels from the edge), which does SSL inspection, URL filtering, malware scanning. Think "the corporate web proxy, but as a cloud service."
- **ZPA (Zscaler Private Access)**: zero-trust replacement for VPN for *inbound-to-private-apps* access. Users connect to Zscaler; lightweight **App Connectors** near the apps dial *out* to Zscaler; users are stitched to apps per-policy. No inbound firewall holes, apps are invisible to the internet.

**WAF quick primer**: A Web Application Firewall inspects HTTP(S) requests against rules (SQL injection, XSS patterns, rate limits, IP lists). AWS WAF attaches to CloudFront, ALB, or API Gateway. Rules run in **Count mode** (log matches, don't block) or **Block mode**. The professional workflow is always: **deploy in Count → analyze matches for false positives → then flip to Block**.

### Q10. "Walk me through a production DNS cutover you'd execute in Route 53."

"Say we're moving an API from an old ALB to a new one. My playbook:

1. **T-minus days: lower the TTL.** If the record's TTL is 3600s, resolvers worldwide may cache the old answer for up to an hour after I change it. So days before the window, I drop TTL to 60s and wait *at least one old-TTL period* for caches to expire. This makes the eventual cutover take effect in about a minute globally.
2. **Pre-checks in the window**: new target is healthy (direct-to-ALB curl tests against the new target, correct cert served, health checks green), monitoring dashboards open, rollback command pre-written.
3. **The change**: update the alias/CNAME. For extra safety on high-stakes services, use **weighted routing** — shift 5% → 25% → 100% while watching error rates, instead of a hard flip.
4. **Validation**: `dig` against multiple resolvers, synthetic transactions through the new path, watch 5xx/latency on both old and new targets (traffic draining from old, arriving at new).
5. **Rollback**: because TTL is 60s, reverting the record restores traffic in ~a minute. Old target stays alive until bake period ends.
6. **Cleanup**: after the bake period, raise TTL back to normal and decommission the old target.

The one-sentence version: **TTL discipline before, weighted shift during, cheap rollback after.**"

### Q11. "You changed a DNS record 30 minutes ago. Some users reach the new endpoint, others still hit the old one. Why, and is it a problem?"

"That's expected behavior, not a fault — it's **cache expiry propagation**. DNS changes aren't pushed; each resolver serves its cached answer until its TTL expires, then re-queries. Different resolvers cached at different times, so expiry is staggered. If the record's TTL was 3600s, full convergence can take up to an hour (longer for misbehaving resolvers that ignore TTLs, and for OS/browser-level caches). It's only a problem if the old endpoint was killed prematurely — which is why the runbook keeps the old target serving until beyond the max TTL window. If someone's stuck longer than TTL, I'd check: client-side OS DNS cache, corporate resolver behavior (some clamp minimum TTLs), and whether there's a pinned hosts-file or connection reuse (long-lived keep-alive connections don't re-resolve DNS)."

### Q12. "An SSL certificate expired in production at 2 AM. Users see browser errors. Walk me through response and prevention."

**Response:** "First confirm scope — which cert, on which endpoint: `openssl s_client -connect host:443 -servername host | openssl x509 -noout -dates` tells me exactly what cert is being served and its validity. Then the fastest safe fix depends on where the cert lives:
- **ACM cert on ALB/CloudFront/API Gateway**: ACM certs auto-renew *if* validation is intact — an expiry usually means DNS validation records were deleted or it's an *imported* cert (imported certs never auto-renew). Fix: re-issue/renew, and for imported certs, **re-import the new cert under the same ACM ARN** so all attachments pick it up without touching the listeners.
- **Cert on an EC2/on-prem server**: install the renewed cert + full chain, reload the web server gracefully (`nginx -s reload`), verify with `openssl s_client` that the *new* dates and the *complete chain* are served.

**Prevention** is the real answer: certificate lifecycle management. Inventory every cert (ACM, imported, on-instance, internal CA); prefer ACM-managed with DNS validation because renewal is automatic and hands-off; monitor expiry with alerts at 45/30/15 days (AWS Config rule or ACM expiry events into EventBridge → alert); protect the DNS validation records from deletion; and make cert rotation a routine automated change, not an annual panic."

### Q13. "After renewing a certificate, some clients still fail with trust errors while browsers work fine. What's going on?"

"Classic **incomplete chain** problem. A server must present the *leaf certificate plus intermediates*. Browsers are forgiving — they cache intermediates or fetch them via AIA — but strict clients (Java apps, curl, mobile SDKs, monitoring agents) fail if an intermediate is missing. Diagnose with `openssl s_client -connect host:443 -showcerts` and check whether the chain is complete and in order. Fix: deploy the full chain file (leaf + intermediates, not the root). A second variant of this symptom: the CA changed its intermediate/root and old clients don't have the new root in their trust store — then the fix is on the client trust store side or choosing a cert chain compatible with your oldest clients."

### Q14. "Describe how you'd execute a Transit Gateway route change in production."

**Concept first:** A TGW is a regional cloud router; VPCs and VPN/Direct Connect attach to it. Each attachment is *associated* with one **TGW route table** (decides where that attachment's outbound traffic can go) and *propagates* its routes into route tables (advertises itself). Two-hop rule of thumb: for VPC-A to reach VPC-B via TGW, you need (1) VPC-A's *subnet route table* pointing the destination CIDR at the TGW, (2) the *TGW route table* routing that CIDR to VPC-B's attachment, and (3) the return path configured symmetrically. Most outages come from fixing one direction and forgetting the return.

**Answer:** "Standard change discipline: document current state (export route tables), define the exact route additions/changes in Terraform so before/after is reviewable in the plan, identify blast radius (which attachments' traffic pattern changes), schedule the window, and pre-write the rollback (the inverse routes). During execution: apply, then validate with actual traffic tests in both directions — not just 'route exists' but 'packets flow' — using Reachability Analyzer or test instances, plus VPC Flow Logs if something's off. The classic failure I'd check for: **asymmetric routing** — forward path updated, return path still pointing the old way, which shows up as connections timing out despite 'the route being there'. And I'd watch for **CIDR overlaps**, because TGW route evaluation is longest-prefix-match, so a more specific stale route can silently hijack traffic."

### Q15. "Traffic between two VPCs through TGW suddenly fails after a change window. How do you troubleshoot?"

"Layered, both directions:
1. **Subnet route tables in VPC-A**: does the destination CIDR route to the TGW attachment? (Someone may have reverted or overwritten it.)
2. **TGW route table associated with A's attachment**: route to B's attachment present, in `active` state (not blackhole)?
3. **Return path**: same two checks from B back to A — the most common miss.
4. **Attachment health**: both attachments `available`? Correct subnets per AZ? (TGW attachment needs a subnet in each AZ it serves.)
5. **Security groups & NACLs**: routing can be perfect while SGs block. Remember SGs are stateful (return traffic auto-allowed), NACLs are stateless (need explicit rules both directions, including ephemeral ports 1024-65535 for return traffic).
6. **Tools**: VPC **Reachability Analyzer** gives a verdict with the exact blocking component; **VPC Flow Logs** show ACCEPT/REJECT per direction — if A→B shows ACCEPT in A but nothing arrives in B's logs, it's routing; if B logs REJECT, it's SG/NACL.
7. **Longest-prefix check**: a new more-specific route (e.g., /24 vs the expected /16) added in the window can hijack traffic to the wrong attachment."

### Q16. "Users behind Zscaler can't reach an internal app that works fine from the office network. Where do you look?"

"First, identify which Zscaler path is in play:
- If it's **ZPA** (private app access): check (1) is the app's domain/IP defined in an **App Segment**, (2) does an **Access Policy** allow this user/group, (3) are the **App Connectors** serving that segment healthy and able to reach the app (connectors dial outbound — check their egress and their DNS resolution of the app, since the *connector*, not the user, resolves internal DNS), (4) client-side: Zscaler Client Connector running, right posture profile. A very common failure: app moved IPs/hostname and the App Segment definition is stale.
- If it's **ZIA** (outbound proxy) blocking access to an app: check URL filtering/cloud app policy for a block verdict in ZIA logs, and **SSL inspection** issues — ZIA man-in-the-middles TLS, re-signing with the Zscaler root CA; if the client or app doesn't trust that CA (common with Java apps, CLI tools, containers that have their own trust stores), you get cert errors. Fix is deploying the Zscaler root cert into that trust store or an inspection bypass for that destination.
- The office-vs-remote difference itself is the clue: office traffic may bypass Zscaler entirely (direct route), so the delta is everything Zscaler adds — policy, SSL inspection, connector reachability."

### Q17. "How do you roll out a new WAF rule without breaking legitimate traffic?"

"**Count mode first, always.** Deploy the rule in Count so it logs matches without blocking. Let it soak — days for a busy app — then analyze the matched requests (WAF logs to S3/CloudWatch, or the sampled requests console): are matches genuinely malicious, or is a legitimate pattern tripping it? Classic false positives: rich-text fields matching SQLi/XSS patterns, large legitimate payloads hitting size rules, an office NAT IP tripping rate limits. Tune — add scope-down statements, exclude specific URI paths or fields, adjust thresholds — re-soak, and only when the false-positive rate is clean, flip to **Block**. Even then, flip during a monitored window with rollback ready (Block → Count is one change). For AWS Managed Rule groups, the same applies: enable the group in count-override first, then selectively un-override rules. The one-liner: *WAF changes are guilty until proven innocent — Count mode is the trial.*"

### Q18. "Legitimate users report intermittent 403s after a WAF change last week. Troubleshoot."

"403 with WAF in path = suspect a blocking rule. Steps: (1) confirm the 403 originates from WAF, not the app — WAF-blocked requests never reach the backend, so backend logs won't show them; response headers/body style also differ. (2) Pull WAF logs filtered to `action=BLOCK` around the reported timestamps, correlate by client IP/URI. The log shows exactly **which rule** in **which rule group** terminated the request. (3) Understand *why* those requests match — 'intermittent' usually means only certain payloads trip it (e.g., only when users paste content containing quotes → SQLi pattern; only heavy users → rate limit; only one office IP → IP reputation list). (4) Fix surgically: rule exception scoped to the specific path/field, not disabling the whole rule. (5) Regression-test the attack the rule exists to stop still gets blocked. And process-wise: this is why the rule should have soaked in Count mode longer."

---

## 1.4 SCP Policy Changes in Controlled Enterprise Environments

### Mental Model
**SCPs (Service Control Policies)** are Organization-level guardrails attached to OUs or accounts. Crucial mental model: **SCPs never grant permissions — they define the maximum possible permissions.** Effective permission = intersection of (SCP allows) ∩ (IAM policy allows). Even an account's root user is bound by SCPs. They're how enterprises enforce "no one can disable CloudTrail," "only these regions," "no public S3," org-wide.

Inheritance: SCPs flow down the OU tree. An action must be allowed at *every level* from root → OU chain → account. A `Deny` anywhere wins. Most orgs keep `FullAWSAccess` attached everywhere and layer explicit **Deny** statements — a deny-list strategy — because allow-list SCPs are brittle.

Why "controlled environments" matters: an SCP change can instantly break *every workload in every account under that OU*. So changes are high-ceremony: reviewed, tested, staged.

### Q19. "How would you safely roll out an SCP change in a large organization?"

"Treat it like a production deploy with a blast radius of entire accounts:
1. **Author in code** — SCPs live in Terraform/git, changes come as PRs with review from security + platform.
2. **Understand evaluation** — verify the intended effect against the inheritance chain: will this Deny at the OU level break something a child account legitimately does? Search CloudTrail across affected accounts for recent usage of the actions being denied — *data, not assumption*. If accounts actively call an API I'm about to deny, I've just found my breakage before it happens.
3. **Stage the rollout** — apply to a **sandbox/test OU** first, then a canary set of low-risk accounts, then progressively broader OUs. Never root-first.
4. **Monitor** — after each stage, watch CloudTrail for `AccessDenied` spikes attributable to the new SCP (the error mentions 'with an explicit deny in a service control policy'), and keep a fast-feedback channel with affected teams.
5. **Rollback plan** — the previous policy version is one `terraform apply` away; detaching/reverting an SCP takes effect immediately.
The key discipline: SCP mistakes don't degrade gracefully — they hard-deny — so usage analysis *before* and staged rollout *during* are non-negotiable."

### Q20. "A team says they suddenly get AccessDenied on an action their IAM role clearly allows. How do you debug?"

"IAM allows but access denied → look **above** IAM:
1. **Read the error** — modern AccessDenied messages state the source: '...with an explicit deny in a service control policy' vs a permissions boundary vs a resource policy. That usually ends the mystery.
2. If SCP: walk the account's OU chain and inspect every attached SCP for a matching Deny (or a missing Allow, if allow-list style). Check **conditions** on the deny — region conditions, tag conditions, `aws:PrincipalArn` exceptions — teams often fail only in one region or only for untagged resources.
3. Check the other permission layers in order: **permissions boundary** on the role, **resource-based policy** (bucket policy, KMS key policy — KMS is a classic: IAM allows `kms:Decrypt` but the key policy doesn't), **session policies**, and VPC endpoint policies if the call goes through an endpoint.
4. Confirm with the **IAM Policy Simulator** or CloudTrail's `errorMessage` for the exact call.
The mental model I'd state out loud: *effective access = SCP ∩ permissions boundary ∩ identity policy ∩ resource policy, with any explicit Deny winning* — so 'my IAM policy allows it' is only one vote of four."

### Q21. "Give an example of SCPs you'd expect in a regulated enterprise."

- Deny disabling/deleting security tooling: `cloudtrail:StopLogging`, `cloudtrail:DeleteTrail`, `guardduty:Delete*`, `config:Delete*`.
- **Region restriction**: deny all actions outside approved regions (with condition `aws:RequestedRegion`), carving out global services (IAM, Route 53, CloudFront, support).
- Deny leaving the Organization (`organizations:LeaveOrganization`).
- Deny making S3 public / disabling Block Public Access at account level.
- Deny root user actions (condition on `aws:PrincipalArn` = root).
- Protect network plumbing: deny deleting/modifying TGW attachments, flow logs, or the org's VPC baseline except by the platform pipeline role (exception via `aws:PrincipalArn` condition).
- Require encryption / deny creating unencrypted EBS volumes or RDS instances.

---

# PART 2: AWS CLOUD SERVICES — SCENARIO & TROUBLESHOOTING

## 2.1 VPC, NAT Gateway, Route Tables

### Mental Model
A **VPC** is your private network in AWS: a CIDR block (e.g., 10.0.0.0/16) carved into **subnets** (each in one AZ). A subnet is "public" only because its **route table** has a route `0.0.0.0/0 → Internet Gateway`; "private" subnets instead route `0.0.0.0/0 → NAT Gateway`. A **NAT Gateway** lives in a *public* subnet and lets private instances initiate outbound internet connections (it translates their private IPs to its Elastic IP) while blocking inbound-initiated traffic. Route tables are longest-prefix-match: the most specific matching route wins; the `local` route for the VPC CIDR always exists and can't be removed.

**Security groups** = stateful, instance-level, allow-rules-only (return traffic automatically allowed). **NACLs** = stateless, subnet-level, allow+deny, evaluated by rule number — need explicit rules in *both* directions including ephemeral ports.

### Q22. "An EC2 instance in a private subnet can't reach the internet. Troubleshoot."

"In order:
1. **Subnet route table**: is there `0.0.0.0/0 → nat-xxxx`? Is the route `active` (a blackholed NAT route means the NAT GW was deleted)?
2. **NAT Gateway itself**: exists, state `available`, and — the classic — **is it in a public subnet**? A NAT GW in a subnet without an IGW route is decorative. Does it have an Elastic IP?
3. **The public subnet's route table**: `0.0.0.0/0 → igw-xxxx` present?
4. **Security group** on the instance: outbound rules allow the traffic? (Default SG allows all outbound, but hardened environments often restrict it.)
5. **NACLs** on both the private and public subnets: outbound 443 allowed *and* inbound ephemeral (1024-65535) allowed for return traffic — NACLs are stateless.
6. **DNS**: if `curl` to an IP works but to a hostname doesn't, it's DNS not routing — check VPC DNS settings (`enableDnsSupport`) or the corporate resolver path.
7. If all that's clean: VPC Flow Logs on the instance ENI — REJECT records name the direction being blocked; Reachability Analyzer for a definitive verdict."

### Q23. "NAT Gateway costs are huge on your bill. What do you do?"

"NAT charges per-GB processed, so find the flows: enable VPC Flow Logs, aggregate top talkers through the NAT ENI. The usual culprit is **traffic to AWS services going out the NAT and back** — S3, ECR image pulls, CloudWatch logs. Fixes: (1) **Gateway endpoints** for S3 and DynamoDB — free, route-table-based, removes that traffic from NAT entirely; (2) **Interface endpoints (PrivateLink)** for ECR, CloudWatch, STS, etc. — hourly + per-GB cost but usually far cheaper than NAT for heavy traffic and keeps it private; (3) check for cross-AZ waste — one NAT per AZ so traffic doesn't hairpin across AZs; (4) for container-heavy environments, image pulls through ECR endpoints alone often cut the bill dramatically."

### Q24. "Two VPCs are peered but instances can't communicate. Why?"

"Peering creates the *possibility*, not the plumbing. Check: (1) **routes on both sides** — each VPC's subnet route tables need the *other* VPC's CIDR pointed at the peering connection (people add one side and forget the other); (2) **SG/NACL** — SG rules must allow the peer CIDR (you can reference peer SGs only in same-region peering); (3) **overlapping CIDRs** — peering between overlapping VPCs is invalid/unroutable, which is why enterprises use TGW + IPAM planning; (4) DNS resolution over peering must be explicitly enabled if they're resolving each other's private hostnames; (5) peering is **non-transitive** — if A↔B and B↔C, A cannot reach C through B. That's the fundamental reason enterprises graduate to Transit Gateway: hub-and-spoke instead of a full mesh of peerings."

## 2.2 Route 53 & Route 53 Resolver

### Mental Model
**Route 53** = AWS DNS. Public hosted zones answer the internet; **private hosted zones** answer only inside associated VPCs. Routing policies: simple, weighted (traffic %), latency, failover (with health checks), geolocation. **Alias records** are Route-53-special: they point at AWS resources (ALB, CloudFront) at the zone apex, resolve free, and track the target's IPs automatically.

**Route 53 Resolver** is the piece enterprises live on: every VPC has a built-in resolver at VPC-CIDR+2 (the ".2 resolver"). For **hybrid DNS**:
- **Inbound endpoints**: ENIs in your VPC that *on-prem* DNS servers forward to, so on-prem can resolve your private zones.
- **Outbound endpoints + forwarding rules**: the VPC resolver forwards queries for specific domains (e.g., `corp.internal`) *to on-prem* DNS servers. Rules can be shared org-wide via RAM from a central networking account.

### Q25. "EC2 instances can't resolve on-prem hostnames like db.corp.internal. Troubleshoot."

"That's the outbound-resolver path:
1. Is there a **Resolver rule** (forward type) for `corp.internal` targeting the on-prem DNS IPs? Is it **associated with this VPC**? (Rules shared via RAM still need per-VPC association — the number-one miss in spoke accounts.)
2. Is the **outbound endpoint** healthy, and do its ENIs' security groups allow **TCP+UDP 53 outbound** to the on-prem servers?
3. **Network path**: can the endpoint ENIs actually reach on-prem DNS — routes via TGW/VPN/DX, on-prem firewall allowing port 53 from the endpoint subnets? Flow logs on the endpoint ENIs show whether queries leave and answers return.
4. **On-prem side**: do those DNS servers answer queries sourced from AWS ranges (ACLs on the DNS servers themselves)?
5. Test precisely: `dig db.corp.internal @<VPC+2 resolver>` from the instance vs `dig @<on-prem-DNS-IP>` directly — tells you whether the break is rule matching or network path."

### Q26. "On-prem servers can't resolve your private hosted zone entries. What's the design fix?"

"Inbound path: create a **Route 53 Resolver inbound endpoint** (ENIs in two AZs), then configure on-prem DNS servers with a **conditional forwarder**: queries for the private zone's domain forward to the inbound endpoint IPs. Also ensure the private hosted zone is **associated with the VPC** hosting the endpoint (the resolver only answers for zones associated with its VPC), SGs on the endpoint allow 53 from on-prem ranges, and the network path exists. This inbound/outbound endpoint pair is the standard hybrid-DNS architecture, usually centralized in a shared networking account with rules shared via RAM."

### Q27. "Explain how you'd design DNS failover for a critical API."

"Route 53 **failover routing** with **health checks**: primary record points at the main region's endpoint with a health check (HTTPS check against a real health path, not just TCP); secondary record points at the DR region. Health check fails → Route 53 starts answering with the secondary. Key details: keep the record **TTL low (30-60s)** or failover is delayed by caching; make the health check test *actual functionality* (a /health that verifies dependencies) not just 'port open'; health checkers live *outside* your VPC so the endpoint must be publicly checkable — for private resources, use a CloudWatch-alarm-based health check instead; and rehearse the failback, which is where untested designs break."

## 2.3 ACM (Certificate Manager)

### Mental Model
ACM issues and stores TLS certs. **ACM-issued + DNS-validated = auto-renews forever** as long as the validation CNAME stays in DNS. **Imported certs never auto-renew.** ACM certs can't be exported (private key never leaves AWS) — they attach only to integrated services: ALB/NLB, CloudFront, API Gateway. CloudFront requires its cert in **us-east-1**, always — a famous gotcha.

### Q28. "An ACM certificate failed to renew and the ALB is now serving an expired cert. What happened and how do you fix and prevent it?"

"ACM DNS-validated certs renew automatically, so a failure means: (1) the **DNS validation CNAME was deleted** — someone 'cleaned up' the weird `_abc123` record in Route 53; (2) it's **email-validated** and nobody clicked the renewal mail; or (3) it's an **imported** cert, which never auto-renews. Fix: restore the validation CNAME (ACM retries validation) or re-issue and swap on the listener; for imported, re-import the renewed cert **to the same certificate ARN** so every attached listener/distribution updates in one step. Prevention: standardize on DNS validation, protect validation records (IaC-managed, so drift/deletion is visible), and alert on the ACM `days-to-expiry` metric / expiry EventBridge events at 45 days."

## 2.4 CloudFront

### Mental Model
CloudFront = CDN: edge locations worldwide cache your content and terminate TLS close to users. An origin can be S3, ALB, API Gateway, anything HTTP. **Cache behaviors** route by path pattern (`/api/*` → ALB origin, no caching; `/static/*` → S3, long TTL). Cache key = what makes a cached object unique (URL + selected headers/cookies/query strings) — the more you include, the worse your hit ratio.

### Q29. "Users report stale content after a deploy. Options?"

"Three tools, in order of preference: (1) **versioned asset URLs** (`app.3f9c.js`) — new deploy references new URLs, cache misses naturally, and old assets can stay cached forever; this is the right long-term fix. (2) **Invalidation** (`/*` or targeted paths) — immediate but costs beyond the free tier and is a blunt instrument; fine as a deploy step for HTML entry points. (3) Sensible **TTLs** via Cache-Control headers from the origin — short TTL on HTML, long on fingerprinted assets. If stale content persists *after* invalidation, remember the browser cache is a second layer — check response Cache-Control headers, because CloudFront invalidation can't reach a user's browser cache."

### Q30. "CloudFront returns 502/504 from your ALB origin. Troubleshoot."

"502 = CloudFront couldn't get a valid response from origin; 504 = origin timed out.
- **TLS to origin**: with 'HTTPS only' to the origin, the ALB's cert must be valid, unexpired, and **match the origin domain name configured in CloudFront** — a mismatched or self-signed origin cert is the classic 502.
- **Origin reachability**: ALB SG must allow CloudFront (best via the CloudFront managed prefix list), ALB healthy targets, correct origin domain.
- **Timeouts**: origin response timeout defaults to 30s — a slow backend gives 504; either fix the backend latency or adjust the timeout consciously.
- **Keep-alive mismatch**: origin closing idle connections faster than CloudFront reuses them causes intermittent 502s — set ALB idle timeout ≥ CloudFront's origin keep-alive.
- Data source: CloudFront access logs give the `x-edge-result-type` and detailed error type; ALB access logs show whether the request even arrived."

## 2.5 API Gateway

### Mental Model
Managed front door for APIs: routing, auth (IAM/Cognito/Lambda authorizers), throttling, request validation, stages. REST APIs integrate with Lambda, HTTP backends, or AWS services. For private backends in a VPC, a **VPC Link** connects API Gateway to an internal NLB/ALB. Hard limit worth knowing: **29-second integration timeout** on REST APIs (now raisable in some cases, but 29s is the famous number).

### Q31. "API Gateway returns 504 for some requests. What's your first suspicion?"

"The **29-second integration timeout**. If the backend (Lambda or HTTP) takes longer, API Gateway gives up with 504 even though the backend may finish successfully — which also causes 'the work happened twice' bugs when clients retry. Confirm by comparing backend duration logs against the 29s line. Fixes: make the backend faster, or restructure long work as **async** — API returns 202 immediately with a job ID (queue + worker, or Step Functions), client polls or gets a callback. Also check Lambda cold starts inside a VPC and Lambda's own timeout setting — a Lambda timing out at its own limit surfaces differently (502 with an error payload) than the gateway timing out (504)."

### Q32. "How does API Gateway reach services in a private VPC, and what breaks there?"

"**VPC Link**: REST API VPC links target an internal **NLB**; HTTP APIs can target ALB/NLB/Cloud Map. Failure modes: NLB target group unhealthy (health check path/port wrong), SGs on targets not allowing traffic from the NLB subnets (NLB preserves client IP by default, so target SGs may need broader allows or you toggle client-IP preservation), and the VPC link status itself. Debug order: is the NLB healthy independent of API Gateway (test from inside the VPC) → then the VPC link → then API Gateway execution logs."

## 2.6 ECS (with Fargate)

### Mental Model
ECS runs containers. **Task definition** = blueprint (image, CPU/memory, env, ports). **Service** = keeps N tasks running behind an optional load balancer, does rolling deploys. Launch types: EC2 (you manage instances) or **Fargate** (serverless — AWS runs the task, you pick CPU/mem). Fargate tasks in private subnets need a path to pull images (NAT or ECR VPC endpoints) — the single most common ECS-networking failure.

### Q33. "A Fargate task keeps failing to start with a container image pull error. Troubleshoot."

"`CannotPullContainerError` in a private subnet is almost always **network path to ECR**:
1. Route to internet via NAT, or — better — **ECR interface endpoints** (`ecr.api`, `ecr.dkr`) *plus the S3 gateway endpoint* (image layers are stored in S3 — forgetting the S3 endpoint is the classic half-fix), plus a **CloudWatch Logs endpoint** if using awslogs.
2. Endpoint SGs allow 443 from the task subnets; endpoint DNS enabled.
3. **Task execution role** (not the task role) has ECR pull permissions — `AmazonECSTaskExecutionRolePolicy`.
4. Image actually exists at that tag; repository policy allows the account.
Quick differential: `assignPublicIp` enabled in a public subnet works but private fails → network path. Everything fails everywhere → IAM or image reference."

### Q34. "ECS service is 'running' but the ALB shows unhealthy targets and users get 503. Debug."

"503 from ALB = no healthy targets. Sequence:
1. **Target group health check** vs reality: right **path** (does the app actually serve /health?), right **port** (with awsvpc networking the target port is the container port), right success codes.
2. **SG chain**: ALB SG allows inbound from users; **task SG allows inbound from the ALB SG on the container port** — the most common miss.
3. **Container truth**: `aws ecs execute-command` into a task (or check logs) — is the app listening on the port the task definition declares, bound to 0.0.0.0 not 127.0.0.1?
4. **Grace period**: slow-starting app failing checks before it's up → set health check grace period so ECS doesn't kill tasks mid-boot, creating a kill-restart loop that looks like flapping.
5. Check ECS **service events** tab — it narrates exactly why tasks are being stopped."

### Q35. "Tasks are being OOM-killed. How do you spot and fix it?"

"Stopped-task reason shows `OutOfMemoryError: Container killed due to memory usage` (exit code 137). Fargate enforces the task memory hard; container-level memory limits inside the task also kill. Fix: check Container Insights memory utilization trend — genuine need → raise task memory (Fargate CPU/mem combos are fixed pairs); a leak → fix the app, and as mitigation restart tasks on a schedule. Also distinguish 137 (OOM/SIGKILL) from 139 (segfault) and health-check kills — the stopped reason string tells you which."

## 2.7 Lambda

### Mental Model
Functions that run on demand; you pay per invocation + duration. Key mechanics: **cold start** (first invocation in a new environment loads runtime+code — slower), **concurrency** (each concurrent request = one environment; account-level limit, default 1000), **timeout** (max 15 min), and **invocation modes**: synchronous (API GW — caller gets errors), asynchronous (S3/EventBridge — Lambda retries twice, then dead-letter), and stream-based (SQS/Kinesis with batching).

### Q36. "A Lambda behind API Gateway has intermittent multi-second latency spikes. Why?"

"**Cold starts**: when a request arrives and no warm environment exists, Lambda initializes a new one — runtime, dependencies, init code — adding hundreds of ms to seconds (worst with big packages and VPC-attached functions historically, and with heavy static init). Confirm via the `Init Duration` field in the REPORT log line / X-Ray. Mitigations in order: shrink the package and lazy-load, move heavy init out of the handler path only if actually needed per-invoke, and for hard latency SLOs use **provisioned concurrency** — pre-warmed environments — accepting its cost. Also verify the spike isn't actually downstream (DB connection establishment on cold envs is a related classic — mitigated with RDS Proxy)."

### Q37. "Lambda processing SQS messages is throwing errors and the queue is backing up. Walk through it."

"SQS→Lambda: failed batches return to the queue after visibility timeout and retry; poison messages retry forever without protection. Checklist: (1) read the actual error in CloudWatch Logs; (2) **poison message** handling — configure a **DLQ with maxReceiveCount** so bad messages divert after N attempts instead of clogging; (3) **visibility timeout ≥ 6× function timeout** (AWS guidance) or messages reappear while still processing → duplicates; (4) partial failures — use **batch item failure reporting** so one bad record doesn't fail the whole batch; (5) concurrency throttling — if Lambda hits concurrency limits, throttles look like backlog; check the Throttles metric. And idempotency, always, because SQS standard is at-least-once delivery."

## 2.8 Step Functions

### Mental Model
Managed workflow orchestration: a **state machine** (defined in JSON — Amazon States Language) chains steps — Lambda calls, ECS tasks, service integrations — with **built-in retries, error catching, branching, parallelism, and human-wait patterns**. Use it when a process spans multiple steps that need reliability and visibility: order pipelines, data pipelines, the async pattern behind APIs. Standard workflows = long-running (up to a year), exactly-once; Express = high-volume short-lived.

### Q38. "When would you use Step Functions instead of chaining Lambdas directly?"

"Whenever the *coordination* has requirements: retries with backoff per step, compensation on failure, branching, parallel fan-out, waiting (for time or human approval), or simply **visibility** — Step Functions gives a visual execution history showing exactly which step failed with what input/output, which turns multi-step debugging from log archaeology into reading a diagram. Lambda-calls-Lambda couples functions, loses state on failure, double-bills (caller waits), and hits the 15-min ceiling. My rule: two steps with no failure semantics — direct call is fine; anything with failure handling or more steps — Step Functions."

### Q39. "A Step Functions execution failed midway. How do you debug and recover?"

"Open the execution: the **graph view** shows the failed state in red; its details show the input it received, the error name, and the cause — usually a downstream exception or a `States.Timeout`/`States.TaskFailed`. Fix the underlying cause, then recover: re-run idempotent flows from the start, or use **redrive** to resume a failed Standard execution from the failed state. Design-side: every Task state gets `Retry` (with exponential backoff, on transient errors) and `Catch` routing to a failure-handling path — that's the point of the tool."

## 2.9 AWS Glue & Athena

### Mental Model
This trio is the standard serverless analytics stack:
- **S3** holds raw data files (CSV/JSON/Parquet).
- **Glue Data Catalog** = the metastore: databases/tables that describe *schema and location* of data sitting in S3. **Glue Crawlers** scan S3 and infer/update table schemas. **Glue Jobs** = managed Spark for ETL (e.g., convert CSV→Parquet, clean, partition).
- **Athena** = serverless SQL that queries S3 *in place* using catalog tables. Pay per data scanned ($5/TB) — so everything about Athena optimization is "scan less": **partitioning** (folder-per-day like `dt=2026-07-09/` lets `WHERE dt=...` skip everything else), **columnar formats** (Parquet reads only needed columns), **compression**.

### Q40. "Athena queries are slow and expensive. How do you fix that?"

"Athena cost = data scanned, so reduce scan:
1. **Partition** the data by the columns queries filter on (usually date) and make sure queries filter on the partition key — an unpartitioned or unfiltered query full-scans the bucket.
2. **Convert to Parquet** (a Glue job does this): columnar + compressed typically cuts scan 90%+ vs raw CSV/JSON, and `SELECT` of a few columns reads only those columns.
3. **Compact small files** — thousands of tiny files murder performance via per-file overhead; target ~128MB+ files.
4. Check the query itself: `SELECT *` on wide tables, joins without partition pruning.
Evidence: the query stats show 'data scanned' — before/after numbers make the win concrete."

### Q41. "A Glue crawler ran but Athena queries return zero rows / wrong schema. Troubleshoot."

"Common causes: (1) **new partitions not loaded** — data landed in new S3 prefixes but the table doesn't know; re-run the crawler, run `MSCK REPAIR TABLE`, or better, use **partition projection** so Athena computes partitions without a crawler at all; (2) crawler grouped files into unexpected tables or inferred a wrong type because of mixed file formats/dirty rows in one prefix — inspect the catalog table's schema vs a sample file; (3) location mismatch — table points at the wrong prefix; (4) permissions — Athena's results bucket or Lake Formation permissions blocking reads (empty results with no error smells like Lake Formation). I'd `SELECT "$path" ... LIMIT 10` to see which files Athena is actually reading."


---

# PART 3: NETWORKING FUNDAMENTALS (DNS, Routing, LB, SSL/TLS, Troubleshooting)

## 3.1 DNS Deep Understanding

### Mental Model
DNS resolves names → IPs through a chain: client asks its **recursive resolver** (corporate DNS, VPC .2 resolver, 8.8.8.8), which walks root → TLD → **authoritative** nameservers (Route 53 for your zones), caches the answer for **TTL** seconds, and serves it. Record types you must speak fluently: **A** (name→IPv4), **AAAA** (IPv6), **CNAME** (name→name; illegal at zone apex — that's why Route 53 has Alias), **MX** (mail), **TXT** (verification/SPF), **NS** (delegation), **SOA** (zone metadata).

### Q42. "dig returns the right IP but the app team says DNS is broken. What do you check?"

"First, reproduce *from where they are*: DNS answers can differ by resolver (split-horizon: internal resolvers give private IPs, public give public; geolocation policies give per-region answers). `dig app.corp.com` from my machine vs `dig app.corp.com @their-resolver` vs from their host. Then check the full client path: OS-level DNS cache (`systemd-resolve --statistics`/flush), `/etc/hosts` overrides, `/etc/resolv.conf` search domains causing them to resolve a *different* FQDN than they think, and application-level caching — the JVM famously caches DNS forever by default (`networkaddress.cache.ttl`), so an app can hold a dead IP long after DNS moved. If the name resolves correctly everywhere, 'DNS is broken' usually means 'connection to the resolved IP fails' — different problem, move to connectivity."

### Q43. "Explain TTL trade-offs like you're advising a change window."

"TTL is the caching lease. **High TTL** (hours): fewer queries, faster resolution from cache, but changes propagate slowly — bad before planned cutovers, terrible during incidents. **Low TTL** (30-60s): near-real-time change propagation, at the cost of more resolver load and slightly more lookup latency. Operational discipline: run steady-state at a moderate TTL, and *pre-lower TTL at least one full old-TTL period before any planned change*, then restore after the bake period. For failover records, TTL must be low permanently or the failover is theatrical — DNS says 'go to DR' but clients keep cached primary answers for an hour."

## 3.2 Load Balancing

### Mental Model
**ALB** (Layer 7): understands HTTP — routes by host/path/headers, terminates TLS, WebSockets, WAF-attachable. **NLB** (Layer 4): TCP/UDP passthrough, extreme performance, static IPs per AZ, preserves source IP — used for non-HTTP, VPC Link targets, PrivateLink. Both use **target groups** with health checks; unhealthy targets are pulled from rotation. Key behaviors: **connection draining/deregistration delay** (in-flight requests finish before target removal), **cross-zone load balancing**, **sticky sessions** (cookie-based, ALB).

### Q44. "Users get intermittent 502s from an ALB. Walk through the diagnosis."

"502 = ALB got a bad/failed response from a target. Sources, in observed-frequency order:
1. **Keep-alive timeout mismatch** — the classic: backend idle timeout *shorter* than ALB's (60s default). Backend closes an idle connection just as ALB reuses it → 502. Fix: backend keep-alive timeout > ALB idle timeout.
2. **Targets restarting/deploying** — deploys without proper deregistration draining; check whether 502s correlate with deploy timestamps or autoscaling events.
3. **App crashes mid-request** or resets connections under load (check target app logs at those timestamps).
4. Response malformed / headers too large.
Evidence trail: ALB **access logs** — the `elb_status_code` vs `target_status_code` columns and target response time tell you whether the target answered at all; a 502 with `target_status_code: -` means the connection failed, pointing at network/keep-alive, not app logic."

### Q45. "Health checks pass but users get errors. How is that possible?"

"The health check is lying by being too shallow: it tests `/health` returning 200, which the app serves even while its real dependencies (DB, downstream API) are broken — so 'healthy' targets fail real requests. Fixes: make the health endpoint *meaningfully* check critical dependencies (with care not to cascade failures or hammer the DB), or add synthetic transaction monitoring for the real user path. The inverse failure also exists: too-deep health checks flap targets out during a transient dependency blip, turning a partial degradation into a full outage. Health check depth is a deliberate design decision — I'd say that out loud in the interview."

## 3.3 SSL/TLS & Certificate Lifecycle

### Mental Model
TLS gives encryption + server authentication. **Handshake** (simplified): client hello (supported versions/ciphers + **SNI** — the hostname, so one IP can serve many certs) → server sends cert chain → client verifies chain up to a trusted root CA, checks hostname match and validity dates → keys agreed → encrypted session. **Chain**: leaf cert (your domain) ← intermediate CA ← root CA (in client trust stores). Server must serve leaf + intermediates. **Lifecycle**: issue (with domain validation) → deploy → monitor expiry → renew → rotate → revoke if compromised. Public cert max lifetime is now ~13 months (and shrinking industry-wide), which forces automation.

### Q46. "Walk me through everything that can go wrong TLS-wise, and how you'd diagnose each."

"My diagnostic tool is one command: `openssl s_client -connect host:443 -servername host` (that `-servername` matters — it sets SNI; without it you may get a different/default cert and misdiagnose).
- **Expired cert**: `-dates` shows notAfter in the past. Renew/rotate.
- **Wrong cert for the name** (hostname mismatch): CN/SAN doesn't include the requested name — often SNI misconfig on the server or a cert missing a SAN entry.
- **Incomplete chain**: `-showcerts` shows only the leaf; strict clients fail while browsers work. Deploy the full chain.
- **Untrusted CA**: internal CA cert not in the client's trust store — common with Zscaler SSL inspection re-signing, Java trust stores, containers.
- **Protocol/cipher mismatch**: old client can't do TLS 1.2+ after a hardening change, or server policy dropped a cipher a legacy client needs. `nmap --script ssl-enum-ciphers` or testing with forced versions isolates it.
- **Clock skew**: client's clock wrong → 'certificate not yet valid'. Real and hilarious in incident reviews."

### Q47. "How do you manage certificate lifecycle at enterprise scale?"

"Four pillars: (1) **Inventory** — you can't renew what you don't know exists; scan and register every cert: ACM, imported, on-instance, appliance, internal CA. (2) **Automation-first issuance** — ACM DNS-validated wherever the endpoint is AWS-integrated (auto-renew forever); ACME/cert-manager patterns elsewhere; imported/manual certs are exceptions with owners and runbooks. (3) **Monitoring** — expiry alerts at 45/30/15 days wired to the owning team, plus alert on *renewal failure* not just expiry proximity. (4) **Rotation as routine** — practiced, low-ceremony rotation procedure, because shrinking cert lifetimes mean manual annual rotation doesn't scale. And protect the renewal dependencies themselves — DNS validation records under IaC so nobody deletes them."

## 3.4 General Network Troubleshooting Method

### Q48. "An application 'can't connect' to a database. Give me your generic methodology."

"Bottom-up, splitting the path in half each time:
1. **Name**: does the hostname resolve, and to the IP we expect? (`dig`) — wrong answer here explains everything downstream.
2. **Route/reach**: can we reach the IP at all — is there a route, does anything on path block? In AWS: subnet route tables, TGW tables both directions, SG/NACL. `traceroute`/Reachability Analyzer.
3. **Port**: is the service listening and reachable on the port? From client: `nc -zv host 5432` or `curl -v telnet://host:5432`. On server: `ss -tlnp | grep 5432` — listening on 0.0.0.0 or only 127.0.0.1?
4. **TLS** (if applicable): handshake succeeds? `openssl s_client`.
5. **Auth/app layer**: connection opens but app errors → credentials, DB max_connections, pool exhaustion — read the actual error, it usually names the layer.
Splitting evidence: connection **timeout** = packets dropped silently (SG, NACL, routing, firewall). Connection **refused** = reached the host, nothing listening on that port (wrong port, service down). That single distinction — timeout vs refused — is the highest-value diagnostic bit in networking."

---

# PART 4: STORAGE & DATA PLATFORMS (EBS, RDS, MongoDB, Snowflake, S3)

## 4.1 EBS

### Mental Model
Network-attached block storage for EC2 — a virtual disk. AZ-locked (volume must be in the instance's AZ; move across AZs via snapshot→restore). Types: **gp3** (general SSD; baseline 3000 IOPS, provision more independently of size — the default choice), io2 (high-IOPS databases), st1/sc1 (throughput HDD). **Snapshots** are incremental, stored in S3, cross-region-copyable. Performance vocabulary: IOPS (operations/sec), throughput (MB/s), latency.

### Q49. "A database on EC2 got slow. You suspect storage. Prove it and fix it."

"Evidence first: on the instance, `iostat -x 1` — the tell-tale columns are **%util near 100** and **await** (per-IO latency) climbing; in CloudWatch, compare `VolumeReadOps+VolumeWriteOps` against the volume's provisioned IOPS, and check throughput against its MB/s limit. Three distinct culprits: (1) **volume limits** — consuming all provisioned IOPS/throughput → raise gp3 IOPS/throughput (online, no downtime via elastic volumes); (2) **instance limits** — every EC2 instance type has its own EBS bandwidth cap; a huge volume on a small instance is throttled by the *instance* — check `EBSIOBalance%`/instance EBS limits, fix by resizing the instance; (3) legacy **gp2 burst credits** exhausted (BurstBalance → 0) — migrate to gp3. Saying 'it can be the volume OR the instance EBS cap' is a senior-level distinction."

### Q50. "How do you resize a volume with minimal downtime, and what's the part people forget?"

"`modify-volume` grows it online — no detach needed. The forgotten part: the **filesystem doesn't grow itself**. After modification: `lsblk` to confirm the new size, `growpart /dev/nvme0n1 1` to expand the partition, then `resize2fs` (ext4) or `xfs_growfs -d /mount` (XFS) — all online. Also: you can grow, never shrink; and after a modify there's a cooldown (~6h) before the next modification."

## 4.2 RDS

### Mental Model
Managed relational databases (Postgres/MySQL/etc.): AWS handles patching, backups, failover. **Multi-AZ** = synchronous standby for HA (failover flips the DNS endpoint to the standby; not for read scaling). **Read replicas** = async copies for read scaling/reporting (can lag). Backups: automated (point-in-time restore within retention) + manual snapshots. Connect via the **endpoint DNS name, never the IP** — failover works by changing what that name resolves to.

### Q51. "RDS failover happened but the app stayed broken for 20 minutes. Why?"

"Failover flips the endpoint's DNS to the standby in ~1-2 minutes; anything broken longer is on the **client side holding the old answer or old connections**: (1) **connection pools** holding TCP connections to the dead primary — pools must detect dead connections (validation query, sensible max lifetime) and reconnect; (2) **JVM DNS caching** — Java caches lookups indefinitely by default; set `networkaddress.cache.ttl` low; (3) any layer caching the resolved IP. Fixes: pool health validation, DNS-cache TTL settings, connecting via **RDS Proxy** (which pins into the failover and makes it near-transparent to apps), and rehearsing failover — `reboot with failover` in a test window proves the app's recovery behavior instead of assuming it."

### Q52. "RDS storage is filling up fast. Immediate and root-cause response?"

"Immediate: check `FreeStorageSpace` trend; enable/verify **storage autoscaling** or manually grow storage (online) before it hits full — a full RDS goes read-only/unavailable. Root cause hunt: (1) runaway **logs** — Postgres WAL buildup from a **broken replication slot** (an abandoned replica/DMS task keeps WAL retained forever — a classic silent disk-killer; drop the stale slot), or verbose general/audit logs; (2) **temp files** from bad queries; (3) genuine data growth vs bloat — Postgres table bloat needs vacuum tuning; (4) local backups/binlogs retention on MySQL. The interview gold is naming the replication-slot WAL scenario — it's a real-world pager story."

### Q53. "Read replica lag is growing. Consequences and fixes?"

"Consequence: readers serve stale data — 'I just updated my profile and it's gone' bugs — and if lag grows unbounded the replica can never catch up. Causes: write-heavy bursts on the primary exceeding replay speed, an underpowered replica instance/storage, long-running queries on the replica blocking replay, or missing indexes making replay expensive. Fixes: right-size the replica (it must be ≥ primary class for heavy write loads), tune or kill long queries on it, split reporting to its own replica, and at design level, route only lag-tolerant reads to replicas. Monitor `ReplicaLag` with alerts."

## 4.3 MongoDB

### Mental Model
Document database: JSON-like documents in collections; flexible schema. HA via **replica sets**: one **primary** takes writes, **secondaries** replicate via the oplog; primary failure → automatic **election** of a new primary (needs a majority of voting members — hence odd counts: 3, 5). Scale-out via **sharding** (split by shard key). You'll recognize this operationally from Domino — its control plane state lives in MongoDB, and 'stuck workspace = Mongo state desync' is a story you already own.

### Q54. "A MongoDB replica set lost its primary and writes are failing. What's happening?"

"Election mechanics: a new primary requires **majority of voting members**. With 3 members, losing 1 → remaining 2 elect fine (seconds of write pause). Losing 2 → the survivor can't reach majority, stays **secondary**, and the set is *read-only* until quorum returns. So my checks: `rs.status()` — member states, who's reachable, any member stuck in RECOVERING/ROLLBACK; network partitions between members (a partition can isolate the primary, which steps down — correct behavior, not a bug); and the deployment-shape mistake behind many of these incidents: an **even number of members or two members in one failure domain** (one AZ/node), so a single infrastructure failure kills majority. Fixes: restore connectivity/members; architecture fix: 3 voting members across 3 failure domains. And on the app side, writes should use retryable writes / handle the election-window errors gracefully."

### Q55. "MongoDB queries got slow. Method?"

"(1) Find the offenders: enable/read the **profiler**/slow query log; (2) `explain()` the slow query — the smoking gun is **COLLSCAN** (full collection scan) instead of IXSCAN, or `totalDocsExamined` vastly exceeding `nReturned`; fix with an index matching the query shape (equality → sort → range field order); (3) check `db.serverStatus()` memory — working set exceeding RAM turns everything into disk reads; (4) look for missing-index-induced replication lag and lock metrics; (5) infrastructure layer last: disk latency (iostat) under the data path. Same discipline as SQL tuning — measure, explain, index, verify."

## 4.4 Snowflake

### Mental Model
Cloud data warehouse (runs *on* AWS/GCP/Azure but is its own product). The architectural headline — **separation of storage and compute**: data lives once in cheap object storage; compute is **virtual warehouses** (t-shirt-sized clusters: XS, S, M...) that you spin up per workload, and they can all query the same data concurrently without contention. You pay per-second while a warehouse runs → the operational levers are **auto-suspend** (warehouse sleeps when idle) and right-sizing. Other terms: **Snowpipe** (continuous ingestion from S3), **stages** (locations for loading data, e.g., an external stage pointing at an S3 bucket), **Time Travel** (query data as of a past point — undelete!), **zero-copy cloning** (instant dev copies of prod data without duplicating storage).

### Q56. "As a platform engineer, what would you actually manage in Snowflake?"

"Not the analytics — the platform around it: (1) **ingestion plumbing**: S3 → external stage → Snowpipe or COPY jobs; that means IAM — Snowflake accesses S3 via a **storage integration** backed by an IAM role trust, the classic breakage point; (2) **cost governance**: warehouses left running are the #1 waste — enforce auto-suspend, resource monitors with credit quotas and alerts, right-size warehouses per workload; (3) **access**: RBAC roles, SCIM/SSO integration, network policies (IP allow-lists / PrivateLink so traffic doesn't cross the public internet); (4) **Terraform** — there's a Snowflake provider; warehouses, roles, grants as code fits this JD's 'compliant, standardized IaC' theme."

### Q57. "Snowpipe stopped loading new files from S3. Troubleshoot."

"Snowpipe is event-driven: S3 event notification → SQS → Snowpipe loads the file. Break points: (1) **event notification chain** — S3 bucket notification configured to the right SQS/SNS, on the right prefix? (Someone re-terraforming the bucket often clobbers notifications.) (2) **Storage integration / IAM** — the IAM role trust policy has an external ID from Snowflake; if the role or trust was recreated, access silently breaks — `SELECT SYSTEM$PIPE_STATUS(...)` and the `COPY_HISTORY` view show errors; (3) **file-level failures** — malformed files rejected; COPY_HISTORY shows per-file load errors; (4) pipe paused. Recovery: fix cause, then `ALTER PIPE ... REFRESH` to sweep missed files."

## 4.5 S3

### Mental Model
Object storage: buckets → keys (objects), 11-nines durability, effectively infinite. Access control layers: IAM policies, **bucket policies** (resource-based), Block Public Access, ACLs (legacy). Features you should wield: **versioning** (undelete/ransomware protection), **lifecycle rules** (transition to IA/Glacier, expire), **replication** (CRR for DR), encryption (SSE-S3/SSE-KMS), **presigned URLs**, event notifications.

### Q58. "An application suddenly gets AccessDenied on S3 it accessed yesterday. Nothing 'changed'. Debug."

"Something changed — find which of the *five* layers: (1) **IAM policy** on the caller's role — diff via CloudTrail's IAM change events; (2) **bucket policy** — same; explicit Deny in a bucket policy beats any Allow; look especially for new `aws:SourceVpce` / `aws:SourceIp` conditions from a security hardening change that now excludes this caller's path; (3) **SCP** newly applied to the account (the 'explicit deny in a service control policy' error text); (4) **KMS** — if objects are SSE-KMS, the caller needs `kms:Decrypt` *on the key policy* too; a key policy change breaks reads while the bucket policy looks fine — a very common one; (5) **VPC endpoint policy** if the call traverses an S3 endpoint. CloudTrail's `errorMessage` on the failed call plus recent change history across those five layers finds it fast."

### Q59. "How would you protect and recover S3 data against accidental deletion or ransomware?"

"**Versioning** — deletes become delete markers; recover by removing the marker or restoring a prior version. **MFA delete / bucket policies denying permanent deletes** for hardening. **Object Lock** (WORM) where compliance demands immutability. **Replication** to a second account/region — a different-account replica with tight access is the ransomware-grade protection, because compromise of the source account can't reach it. **Lifecycle** rules to control cost of retained versions. And least-privilege: almost nothing needs `s3:DeleteObjectVersion` — deny it broadly."


---

# PART 5: TERRAFORM (IaC — Modules, State, Enterprise Patterns)

### Mental Model
Terraform: you declare desired infrastructure in **HCL**; `terraform plan` diffs desired vs the **state file** (Terraform's record of what it manages, mapping code addresses → real resource IDs) vs reality; `apply` executes the diff via **providers** (AWS, Snowflake, etc.). **Modules** = reusable packages of resources with input variables and outputs — the unit of standardization ("our VPC module bakes in flow logs, tagging, and compliant defaults; teams consume it instead of hand-rolling"). Enterprise essentials: **remote state** (S3 backend + locking) shared safely by teams/CI, state is sensitive (contains secrets) and must be protected, and drift (reality diverging from state) is managed with plan-in-CI.

### Q60. "How do you design a Terraform module that's reusable AND compliant?"

"A module is an API — design it like one:
- **Opinionated defaults, minimal required inputs**: the module encodes the compliant path (encryption on, logging on, mandatory tags merged in, private-by-default) so the easy path is the compliant path. Consumers provide only what genuinely varies (CIDR, name, size).
- **Guarded escape hatches**: variables with `validation` blocks constraining allowed values (e.g., instance types from an approved list); don't expose knobs that let consumers turn off compliance.
- **Outputs as contract**: expose IDs/ARNs downstream modules need, nothing internal.
- **Versioned releases**: modules in their own repos (or registry), consumed by **pinned version** (`?ref=v2.3.1`) — never branch references — so a module change can't silently alter every consumer; upgrades are deliberate.
- **Tested**: example configurations + automated validation (fmt/validate/plan in CI, policy checks) per release.
The one-liner: *a good module makes the compliant thing the default thing, and version pinning makes change deliberate.*"

### Q61. "Explain Terraform state — why it exists, and what goes wrong with it."

"State maps 'this resource block' → 'that real AWS resource ID', stores attributes for interpolation, and enables diffing without scanning the whole cloud. Failure catalogue:
- **Concurrent applies** corrupting state → remote backend with **locking** (S3 backend with lockfile/DynamoDB lock) so two applies can't interleave.
- **Lost/corrupted state** → Terraform thinks nothing exists and plans to create duplicates → backend versioning (S3 versioning) for recovery; state is backed up, never hand-edited casually.
- **Drift** — someone changed infra in the console → next plan shows unexpected diffs; handle by reviewing plan output, then either codify the manual change or let apply revert it; detect proactively with scheduled plan runs.
- **Secrets in state** — state stores resource attributes including passwords → encrypt the backend, restrict access tightly, treat state as sensitive data.
- **Importing existing infra** — `terraform import` (or import blocks) to adopt console-created resources into state instead of recreating.
- Surgical tools: `terraform state mv` for refactors (renaming/moving resources without destroy/create), `state rm` to unmanage, `-refresh-only` to reconcile."

### Q62. "terraform plan wants to destroy and recreate a production database. What do you do?"

"Stop — never apply a surprising destroy. Diagnose why: the plan says which attribute forces replacement (`# forces replacement` annotation). Common causes: (1) a **refactor renamed the resource address** — Terraform sees 'old gone, new needed'; fix with `terraform state mv` old new (or a `moved` block) so state follows the rename, and plan goes clean; (2) an attribute that genuinely can't change in place — then the question is whether the change is truly wanted and how to do it safely (blue-green, snapshot-restore) instead of letting Terraform nuke prod; (3) drift or a provider upgrade changing behavior. Guardrails I'd mention: `lifecycle { prevent_destroy = true }` on stateful crown jewels, and mandatory human review of plans in the pipeline — a plan output is a change request, not a formality."

### Q63. "How do you structure Terraform across many teams and environments?"

"Separation on two axes. **Environments**: same code, different variable inputs and *different state* per environment — via directory-per-env with a shared module layer, or workspaces (with care), or TFE/Spacelift-style stacks; the invariant is prod and dev never share a state file. **Ownership**: split state by blast radius and team — networking (VPC/TGW) in its own state owned by platform, each app team's infra in theirs, connected via `terraform_remote_state` data sources or data lookups; a small state applies fast and a mistake in one can't corrupt everything. **Change flow**: all changes via PR → CI runs fmt/validate/plan + policy-as-code (OPA/Sentinel/Checkov) → human review of the plan → apply from the pipeline with its own role, humans don't hold apply credentials. That last point — pipeline-only applies — is what 'controlled enterprise environment' means in practice."

### Q64. "Terraform apply failed halfway. Is the infrastructure broken? What now?"

"Not corrupted — Terraform is not transactional but it *records what it completed*: resources created before the failure are in state; the rest were never attempted. Recovery: read the error (commonly IAM denies, quota limits, naming conflicts, eventual-consistency timing), fix the cause, re-run apply — it continues from where it stopped because state knows what exists. Edge cases: a resource created in the cloud but the write to state failed (rare — apply interrupted hard) → next plan wants to create a duplicate → import the orphan. If state got locked by the dead run: `terraform force-unlock <id>` after confirming no apply is actually running."

---

# PART 6: GITLAB CI/CD

### Mental Model
`.gitlab-ci.yml` in the repo defines the pipeline: **stages** run in order (build → test → scan → deploy), **jobs** within a stage run in parallel, executed by **runners** (shared or self-hosted). Vocabulary that signals fluency: `rules:` (when jobs run — branch, MR, tag), **artifacts** (files passed between jobs) vs **cache** (speed-up, best-effort), `needs:` (DAG — jobs start when their dependencies finish rather than waiting for the whole prior stage), **environments** + `when: manual` (deployment gates/approvals), protected branches/variables, `include:` (shared pipeline templates across repos), and GitLab's built-in security scanning templates (SAST, dependency scanning, secret detection, container scanning).

### Q65. "Design a pipeline for a containerized service with security and deployment controls, per this JD."

"Stages:
1. **Validate/build**: lint, unit tests, build the image, tag with the commit SHA (immutable — never deploy `:latest`), push to registry.
2. **Scan** (the JD's 'security scanning integrated'): SAST on code (their stack apparently uses **Checkmarx** — run the Checkmarx scan job here), dependency scanning, **secret detection**, and **container scanning** (Trivy/GitLab's scanner) on the built image. Fail the pipeline on new criticals — enforcement, not decoration.
3. **Deploy to dev/staging** automatically on main; run integration/smoke tests against it.
4. **Deploy to prod** behind controls: `when: manual` gate, protected environment (only authorized approvers can trigger), possibly a change-ticket check for regulated flows. Deploy strategy rolling/blue-green with automated post-deploy health verification and rollback (redeploy previous SHA — which is why immutable tags matter).
Cross-cutting: shared `include:` templates so every repo inherits the same scan/deploy jobs (standardization = compliance at scale), protected variables for credentials — better, OIDC to AWS instead of stored keys — and artifacts/`needs:` for fast DAG-shaped execution."

### Q66. "A pipeline that worked yesterday now fails, no code change. Your debugging order?"

"'No code change' means the change is in a dependency of the pipeline:
1. **Read the job log**, obviously — the error usually classifies itself.
2. **Runner environment**: shared-runner image updated? Base image `FROM x:latest` pulled a new version? Tool versions unpinned (`pip install foo` grabbing today's release)? — this is the #1 cause; the cure is pinning everything.
3. **External services**: registry down/rate limits (Docker Hub!), package mirrors, artifact stores.
4. **Expired credentials**: tokens, deploy keys, cloud credentials — anything with a lifetime.
5. **CI/CD variable changes**: someone edited group/project variables; protected-variable scope changed.
6. **Cache poisoning**: stale corrupt cache — clear runner cache and re-run.
Retrying once distinguishes 'flaky/external' from 'deterministic', and comparing the last green job's log environment section against the red one diff-hunts version changes."

### Q67. "Docker build inside GitLab CI fails with 'cannot connect to the Docker daemon'. Explain and fix."

"The job runs *in a container*, which doesn't automatically have a Docker daemon. Options: (1) **Docker-in-Docker (dind)**: run the `docker:dind` service alongside the job, set `DOCKER_HOST` accordingly — requires privileged runners, which is its own security conversation; (2) mount the host's Docker **socket** — fast but jobs get root-equivalent control of the host, generally frowned upon on shared runners; (3) **daemonless builders** — Kaniko or BuildKit rootless — build OCI images without a daemon or privilege, the enterprise-preferred answer for shared/regulated environments. Knowing *why* dind needs privileged mode and offering Kaniko as the compliant alternative is the senior answer."

### Q68. "How do you keep secrets safe in GitLab CI?"

"Layers: (1) never in the repo — **secret detection** in the pipeline catches accidents; (2) CI/CD variables marked **protected** (only protected branches/tags see them — so a rogue MR from a feature branch can't exfiltrate prod creds) and **masked** (redacted from logs); (3) the strongest pattern: **no static cloud secrets at all** — GitLab's OIDC/JWT federation: the job presents its short-lived identity token to AWS, assumes a role scoped to that project/branch via the trust policy condition, and gets temporary credentials. Nothing to leak, rotate, or steal. (4) External secret managers (Vault, Secrets Manager) fetched at runtime for app secrets. And audit: who can edit variables, protected environments for who can deploy."

---

# PART 7: DOCKER

### Mental Model
Image = layered, immutable filesystem built from a Dockerfile (each instruction = a layer; layers cache and are shared). Container = a running process (or few) isolated via kernel namespaces + cgroups, with a thin writable layer on top of the image. Not a VM — shares the host kernel. Key practical themes: small secure images (multi-stage builds, minimal bases, non-root user), layer-cache-aware Dockerfiles, and the debugging trio: `docker logs`, `docker exec`, `docker inspect`.

### Q69. "A container exits immediately after start. Debug it."

"(1) `docker ps -a` → **exit code**: `0` = the process finished (a container lives only as long as PID 1; a service that daemonizes itself orphans PID 1 and the container 'exits successfully' — run it in foreground mode); `1`/app-specific = crashed, read `docker logs <id>`; `125-127` = Docker/entrypoint problems ('executable not found' — wrong path, missing binary in a minimal image, or CRLF line endings in the entrypoint script from a Windows checkout — a real classic); `137` = SIGKILL, usually **OOM** (`docker inspect` → `OOMKilled: true`) → raise memory limit or fix the leak; `139` = segfault. (2) If logs are empty, override the entrypoint to get inside: `docker run -it --entrypoint sh image` and run the command by hand. (3) `docker inspect` for env/volume/command misconfig. Exit-code fluency is exactly what an interviewer is probing for here."

### Q70. "Your image is 1.8 GB. Make it small and secure."

"(1) **Multi-stage build**: build with the full toolchain in stage one, `COPY --from=build` only the artifact into a minimal runtime stage — compilers and build deps never ship; (2) **minimal base**: alpine/distroless/slim instead of full OS images — smaller *and* smaller CVE surface; (3) **layer hygiene**: combine RUN steps with cleanup in the same layer (deleting files in a *later* layer doesn't shrink the image — earlier layers are immutable), `.dockerignore` to keep junk out of the build context; (4) **cache-friendly ordering**: dependency manifests copied and installed *before* copying source, so code changes don't invalidate the dependency layer — faster CI too; (5) security: pin base versions, run as **non-root USER**, scan in the pipeline (ties back to their JD)."

### Q71. "Two containers on a host can't talk to each other. Quick triage."

"How are they networked? Default bridge: containers reach each other by IP but **no DNS by name** — user-defined bridge networks give name resolution; the classic fix is `docker network create app && --network app` for both, then reach by container name. Check: same network (`docker network inspect`), the target actually listening (exec in and `ss -tlnp`) and bound to 0.0.0.0 not localhost — 127.0.0.1 inside a container is unreachable from other containers; port publishing (`-p`) matters only for host/outside access, not container-to-container. In compose, services share a network and resolve by service name automatically."

---

# PART 8: LINUX ADMINISTRATION & PERFORMANCE

### Mental Model
For performance interviews, own the **USE method** (Brendan Gregg): for every resource — CPU, memory, disk I/O, network — check **U**tilization, **S**aturation, **E**rrors. Tool map: CPU → `top/htop`, load average, `vmstat` (run-queue r column = saturation); Memory → `free -h` (know that 'available' matters, buff/cache is reclaimable — the classic "Linux ate my RAM" misunderstanding), swap activity in `vmstat` (si/so), OOM killer in `dmesg`; Disk → `iostat -x` (%util, await), `df -h` (space) *and* `df -i` (inodes!); Network → `ss`, `ping`, retransmits. Services → `systemctl status`, `journalctl -u svc`. 

### Q72. "A Linux server is 'slow'. Give me your first 60 seconds."

"A fixed opening sequence, then branch:
1. `uptime` — load averages vs core count: load 30 on 4 cores = saturated; also *which direction* is it trending (1/5/15-min values).
2. `top` (or `htop`) — is it CPU? Which process? **us vs sy vs wa**: high user = app burning CPU; high system = kernel/syscall churn; high **iowait = it's not a CPU problem, it's disk** — pivot to iostat.
3. `free -h` — truly low *available* memory? `vmstat 1` — si/so nonzero = **swapping**, which makes everything feel broken; `dmesg -T | grep -i oom` for OOM kills.
4. `iostat -x 1` — %util pinned, await high = storage bottleneck.
5. `df -h` and `df -i` — full disk or **exhausted inodes** (disk 'full' with space free — millions of tiny files; a beautiful gotcha to mention).
6. `ss -s` / network errors if the slowness is remote-facing.
That's the USE method in practice: identify *which resource* is saturated before touching anything."

### Q73. "Disk is 100% full. df says full, but du of the whole filesystem shows 30 GB less. Explain."

"A process is holding **deleted files open**: deleting an open file removes its directory entry (so `du` can't see it) but the kernel keeps the blocks until the last file handle closes (so `df` still counts them). Typical culprit: a log file rotated/deleted while the service still writes to it. Find it: `lsof +L1` (or `lsof | grep deleted`) → shows the process and the size it's holding. Fix: restart/reload the process to release the handle — or truncate via `/proc/<pid>/fd/<n>` if a restart is unacceptable. Prevention: proper logrotate with `copytruncate` or a post-rotate signal so the app reopens its log. This question is a rite of passage — nail it."

### Q74. "A service won't start after a reboot. Systemd debugging path."

"`systemctl status svc` — state and last log lines; `journalctl -u svc -b` — full logs since boot (the -b scopes to this boot). Common causes ordered: (1) **dependency/order** — service started before its network/mount/database dependency (After=/Requires= missing in the unit); (2) config error introduced earlier but only *bites* at restart — config was changed weeks ago, never reloaded, reboot finally loaded it (why 'nothing changed today' is misleading); (3) **enabled vs started** confusion — it was never `systemctl enable`d so it doesn't start at boot at all; (4) permissions/paths — files on a mount that wasn't ready, SELinux denials (`ausearch -m avc`); (5) port already taken by something else that grabbed it first. Fix, then `systemd-analyze verify` the unit and reboot-test."

### Q75. "High load average but CPU is idle. What does that mean?"

"Load average on Linux counts runnable **plus uninterruptible-sleep (D-state)** tasks — processes stuck waiting on I/O (disk, NFS). So high load + idle CPU = **tasks blocked on I/O**, not CPU demand. Confirm: `vmstat` b column, `ps aux | awk '$8 ~ /D/'` to list D-state processes, then `iostat -x` to find the sick device — or a hung NFS mount, the notorious D-state factory (processes touching a dead NFS server hang unkillably). The distinction between 'load = CPU busy' (wrong) and 'load = demand for CPU *or* stuck on I/O' (right) is a senior-signal answer."

---

# PART 9: SHELL SCRIPTING

### Mental Model
For this JD, scripting = automation glue: safe operational scripts, log/text processing (grep/awk/sed), scheduled tasks, wrapping AWS CLI. Interviewers probe *safety habits* more than syntax: error handling, quoting, idempotency.

### Q76. "What do you put at the top of every production bash script, and why?"

"`set -euo pipefail` — and I can defend each flag: **-e** exit on any command failure instead of blundering on (a script that continues after a failed `cd` and then runs `rm -rf ./*` is the horror story); **-u** error on undefined variables (catches typos like `rm -rf $TMP_DIRR/` expanding to `rm -rf /`); **-o pipefail** a pipeline fails if *any* stage fails, not just the last (`curl ... | jq` shouldn't 'succeed' when curl failed). Plus: **quote every variable** (`"$var"` — unquoted expansion word-splits on spaces), `trap cleanup EXIT` for teardown (temp files, locks) even on failure, explicit logging with timestamps, and **idempotency** — safe to re-run, checks state before acting — because ops scripts get re-run mid-failure by stressed humans."

### Q77. "Write the approach (not perfect syntax) for: find all ERROR lines in today's logs across app servers, count by error type, alert if any type exceeds 100."

"Structure over syntax: loop over hosts (or use the aggregated log path if central logging exists — say that first, it's the senior instinct); `grep ERROR` the day's file, extract the error type field with `awk` (or `sed` if it needs pattern extraction), `sort | uniq -c | sort -rn` for counts — the single most-used pipeline in operations; compare counts to threshold in a while-read loop and fire the alert (webhook/SNS) for offenders. Mention the guardrails: handle 'no matches' (grep exits 1 on no match — interacts with `set -e`, so `|| true` deliberately), quote paths, and timestamped output for audit."

### Q78. "Cron job 'doesn't work' but runs fine manually. Classic causes?"

"The cron environment is *not* your shell: (1) **PATH is minimal** — script calls `aws`/`jq` by bare name, works in your login shell, fails in cron → use absolute paths or set PATH in the script; (2) no profile/bashrc sourced → env vars/credentials you rely on interactively don't exist; (3) **working directory** is HOME, relative paths break; (4) output goes nowhere → redirect stdout/stderr to a log (`>> /var/log/job.log 2>&1`) or you'll never see the error; (5) percent signs in crontab need escaping; (6) it ran as a different user with different permissions. Debug method: capture `env` from inside the cron run and diff against your shell. This question appears in nearly every ops interview — it's free points."

---

# PART 10: SECURITY & COMPLIANCE (Checkmarx, Google SecOps, Governance)

## 10.1 Checkmarx

### Mental Model
**Checkmarx** is an application-security platform, best known for **SAST** (Static Application Security Testing): it scans *source code* (without running it) for vulnerability patterns — SQL injection, XSS, hardcoded secrets, insecure crypto — by tracing data flow from sources (user input) to sinks (queries, output). The suite (Checkmarx One) also covers **SCA** (Software Composition Analysis — vulnerable open-source dependencies), IaC scanning (KICS), and more. As a platform engineer, you don't fix the app code — you **integrate the scan into CI/CD and operate the policy**.

### Q79. "How would you integrate Checkmarx into GitLab CI/CD, and how do you handle the noise?"

"Integration: Checkmarx provides a CLI/plugin — a pipeline job in the scan stage authenticates to the Checkmarx server/tenant, submits the code (typically incremental scans on MRs for speed, full scans nightly/on main), and pulls results; the pipeline **fails on policy breach** — e.g., any new High severity — publishing findings back to the MR so developers see them in review. Noise handling — the part that decides whether developers hate it: (1) **baseline** existing findings so day one doesn't block everything on old debt — enforce on *new* findings, burn down the backlog separately; (2) a governed **triage workflow** — marking false-positive/accepted-risk requires justification and security sign-off, audited (compliance cares who dismissed what); (3) tune rulesets per language/framework; (4) fast incremental scans in MR, thorough scans async — never make the secure path the slow path or teams route around it."

### Q80. "SAST vs DAST vs SCA vs container scanning — where does each live in the pipeline?"

"**SAST** (Checkmarx): source code patterns, runs at MR/commit — earliest, cheapest to fix. **SCA**: your dependencies' known CVEs (the 'log4shell detector'), also at build. **Container scanning** (Trivy et al.): the built image's OS packages + layers, after build, and continuously against the registry (new CVEs appear against old images). **DAST**: probes the *running* application from outside, in staging — catches config/runtime issues static analysis can't. Defense in depth: each sees a class the others don't; shift-left the cheap ones, keep runtime scanning for the rest."

## 10.2 Google SecOps Support Workflows

### Q81. "What does 'supporting Google SecOps workflows' mean for a platform engineer day-to-day?"

"The security analysts write detections and investigate; the platform engineer keeps the **telemetry pipeline healthy and complete**: (1) **feed/forwarder management** — onboarding new log sources (define the source, deploy the forwarder or configure the native feed e.g. CloudTrail via S3, pick the right log type so the UDM parser applies), fixing broken feeds; (2) **ingestion health monitoring** — alert on silent sources ('source X sent nothing for 2 hours' — silence is the dangerous failure mode in security logging), volume anomalies; (3) **parser/normalization issues** — a source flowing but not populating UDM fields breaks detections invisibly; validate field mapping on onboarding; (4) IAM and access for the SecOps tenant; (5) supporting migrations like this Splunk exit. The theme: security *outcomes* depend on pipeline *completeness* — my job is that no one finds out during an incident that a log source died in March."

## 10.3 AWS Governance & Access Policy Management

### Q82. "How do the layers of AWS access control fit together? Give the full evaluation picture."

"For any API call, effective permission is the intersection of every applicable layer, with **explicit Deny winning anywhere**:
1. **SCP** (Org guardrail — the outer boundary, applies even to root),
2. **Permissions boundary** (a cap on what a specific role can do, regardless of its policies — used to let teams create roles safely),
3. **Identity-based policy** (the role/user's IAM policies — the only layer that *grants* by itself),
4. **Resource-based policy** (bucket policy, KMS key policy, SQS policy — can grant cross-account),
5. **Session policies / conditions** (assumed-role session restrictions, condition keys like SourceVpce, MFA, tags).
Cross-account access needs an allow on *both* sides (identity side and resource side). Debugging any AccessDenied = identifying which layer said no — the error message and Policy Simulator get you there. Reciting this stack cleanly is one of the highest-signal answers in an AWS interview."

### Q83. "How do you implement least-privilege at scale without drowning in IAM tickets?"

"(1) **Permission sets via Identity Center** — humans get standardized, role-based access (ReadOnly, Developer, PlatformAdmin) per account, not bespoke IAM users; (2) **workloads use roles, never keys** — IRSA/pod identity on EKS, task roles on ECS, OIDC federation from GitLab — short-lived credentials everywhere, nothing static to leak; (3) **permissions boundaries + self-service** — teams can create their own roles inside a boundary that caps them, killing the ticket queue without losing control; (4) **guardrails over gates** — SCPs deny the catastrophic; IAM Access Analyzer flags external exposure and generates least-privilege policies from actual CloudTrail usage (policy-from-usage is the practical route to least privilege — start broad-ish, tighten from evidence); (5) everything in Terraform, reviewed, so access changes have an audit trail by construction."

---

# PART 11: TYING IT TO YOUR EXPERIENCE + QUESTIONS TO ASK

## How to answer "have you done X?" when you haven't done X exactly
Use the **adjacent-experience bridge** — honest and effective:
- Org migration: "I haven't executed an Org-to-Org account move end-to-end, but I've operated the exact dependency surface it touches daily — TGW attachments shared via RAM, Route 53 private zones and resolver rules, SCP-constrained accounts, centralized logging — supporting Domino platforms on EKS in enterprise pharma environments. The migration playbook is a dependency-mapping exercise over the estate I already run."
- Splunk→Google: "My observability migration thinking comes from platform cutovers I have done: dual-run, validate parity with data, cut source-by-source, never fly blind. The Google-specific pieces — UDM, YARA-L, feeds — I've studied and can ramp fast because the discipline transfers."
- Snowflake/Checkmarx: "I've operated the same *pattern* — [data platforms / security scanning in pipelines] — with [RDS, MongoDB / container scanning, GitLab SAST]; the tool specifics are the smallest part of the job."
Never bluff a hands-on story for a tool you haven't touched — interviewers pull threads.

## Strong questions to ask them
1. "Is the Org migration and Splunk exit already underway, or is this role helping plan it? What phase?"
2. "What's the change-management model for network changes — CAB cadence, who owns rollback authority during windows?"
3. "How is the shared networking account structured — central TGW, central egress, resolver rules via RAM?"
4. "For the SecOps migration — is detection parity with Splunk a formal exit criterion?"
5. "How much of the estate is under Terraform today, and is drift a known problem?"

## Final-hour priorities (if you read nothing else twice)
1. **Part 1 entirely** — migration is why this role exists.
2. Q20/Q82 (policy evaluation stack), Q10 (DNS cutover), Q14-15 (TGW change + troubleshoot), Q17 (WAF count-mode) — these map exactly to the JD's "network and security change execution" line, and you've already studied the underlying material.
3. Q72-75 (Linux) and Q69 (Docker exit codes) — highest-frequency screeners.
4. Skim Snowflake (Q56-57), Checkmarx (Q79), SecOps (Q81) — you only need conversational fluency on these, not depth.

Good luck. You've prepared more broadly than most candidates walking into this — the migration framing is the only genuinely new muscle, and now you have it.
