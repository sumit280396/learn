# Domino Data Lab Administration — Deep-Dive Interview Guide
### Keycloak, AWS credential propagation, and day-to-day admin operations, from zero

---

## IMPORTANT NOTE ON "KEYSTONE" (read before the interview)

There is **no standard Domino component called Keystone**. Keystone is OpenStack's identity service. If the interviewer's team says "Keystone for AWS roles," it is almost certainly one of:
1. **Their internal/company identity or role-brokering service** (custom, named Keystone), sitting between the corporate IdP and AWS, OR
2. Loose shorthand for **Domino's AWS Credential Propagation** mechanism (covered in depth below), OR
3. An actual OpenStack Keystone if part of their stack is on-prem/private cloud.

**How to handle it in the interview (this is a strength, not a weakness):** if they mention Keystone, ask a smart clarifying question: *"Just to make sure I map it correctly — is Keystone your internal role-brokering service that federates identities to AWS, or are you referring to Domino's credential propagation layer?"* Asking precise clarifying questions about a company-specific component is exactly what a senior engineer does. The underlying pattern is identical either way: **an identity broker exchanges a user's authenticated identity for short-lived AWS STS credentials** — and that pattern is what this guide teaches you cold.

---

## PART 0: The Admin Mental Model

A Domino administrator owns five areas. Everything you'll be asked fits into one of them:

1. **Identity & Access** — Keycloak, SSO, users, organizations, roles, and how users get AWS permissions (credential propagation). *The badge office.*
2. **Compute & Environments** — hardware tiers, node pools, compute environments (Docker images), workspace/job lifecycle. *The rooms and equipment.*
3. **Data & Storage** — data sources, Domino Datasets, blob storage, quotas. *The filing system.*
4. **Platform Health** — monitoring, logs, upgrades, backups, disaster recovery. *Building maintenance.*
5. **Governance** — audit trails, project permissions, cost controls, GxP evidence. *The compliance office.*

The admin UI lives under the gear icon; the deeper controls live in **Central Config** (key-value platform settings), **Keycloak's own console** (at `https://<domino-domain>/auth/`), and ultimately **kubectl** — because everything Domino does is Kubernetes underneath.

---

## PART 1: Identity — Keycloak Deep Dive (their stack, so know this best)

### The concepts, from zero

**What Keycloak is:** an open-source identity and access management server. Domino ships with it and delegates ALL authentication to it — Domino itself never checks passwords. Keycloak handles login, SSO federation, token issuance, and session management.

**The mental picture:** Keycloak stands at the gate deciding who gets in and issuing them a badge (token); Domino runs the lab behind the gate and reads the badge to know who you are and what groups you're in.

**Key Keycloak vocabulary:**
- **Realm** — a tenant/isolation boundary. Domino auto-creates **DominoRealm** where all Domino users live; the **Master realm** holds the Keycloak admin. Admin rule #1: when changing anything, confirm you're in DominoRealm.
- **Client** — an application registered with Keycloak that users log into *through* Keycloak. Domino registers itself as a client (e.g., `domino-play`).
- **Identity Provider (IdP) federation** — Keycloak can delegate authentication upstream to the corporate IdP (Okta, Azure AD, ADFS, PingFederate) via **SAML or OIDC**. So the chain is: User → Domino → Keycloak → corporate IdP → back with an assertion → Keycloak issues Domino's token.
- **LDAP federation** — alternatively/additionally, Keycloak can sync users and groups directly from Active Directory via LDAP.
- **Mappers** — the translation rules: they copy attributes from the upstream IdP's assertion (groups, email, AWS role entitlements) into the tokens/session Domino sees. The `domino-play` client ships with mappers including `domino-group-mapper`, `aws-roles-mapper`, `aws-session-duration-mapper`, `aws-role-session-name-mapper` — names worth recognizing out loud in an interview.
- **Tokens** — Keycloak issues JWTs (JSON Web Tokens): short-lived, signed proof of identity that Domino services and even user workloads use (Domino injects a refreshed user JWT into runs for API auth).

### Q&A

**Q1. How does authentication work in Domino?**
> "Domino delegates all authentication to its bundled Keycloak service. In the simplest setup Keycloak stores local usernames/passwords itself; in any enterprise deployment Keycloak federates to the corporate identity provider via SAML or OIDC — so users log in with corporate SSO, MFA policy is enforced upstream where it belongs, and joiner/leaver lifecycle is owned by corporate IT. Keycloak maps attributes from the IdP assertion — group memberships, email, and in our case AWS role entitlements — through client mappers into the session Domino consumes. Domino admins reach Keycloak's own console at the `/auth/` path; the initial admin credential lives in a Kubernetes secret (`keycloak-http`) in the platform namespace — a nice detail showing everything here is still Kubernetes underneath."

**Q2. A new user can't log in. Walk me through your triage.**
> "Split the chain: **(1) Does the user exist and is the federation healthy?** In Keycloak (DominoRealm!), search the user — if federated from the IdP/LDAP, did they get provisioned; are they in the required group that grants Domino access? **(2) Does login fail at the IdP or after?** If they never see the corporate login page, it's the Keycloak↔IdP federation config; if they authenticate upstream but bounce back with an error, it's usually the classics: **mismatched redirect URI, stale client secret, or clock skew breaking token validation**, plus certificate expiry on the SAML signing cert. A SAML-tracer browser extension shows the actual assertions flowing, which localizes it fast. **(3) One user vs everyone?** Everyone = expired IdP certificate or a config change (check what changed); one user = their account, group membership, or attributes."

**Q3. How do users get authorization *inside* Domino (roles/permissions)?**
> "Two layers: **platform roles** — Domino admin, support, or standard user — and **project-level permissions** — owner, contributor, results consumer, etc., controlling who can see or execute what per project. Groups synced from the IdP through Keycloak map to Domino **Organizations**, so access follows corporate group membership automatically rather than manual per-user grants — which is also the auditable, GxP-friendly way: access reviews happen against AD groups, one source of truth."

---

## PART 2: AWS Credential Propagation (the "Keystone" territory — study hardest)

### The problem it solves
Data scientists need AWS access (S3 buckets, Redshift) *from inside their workspaces*, with **their own** permissions — not a shared platform-wide credential (audit nightmare, blast radius) and not long-lived personal access keys pasted into notebooks (security nightmare). The answer: **derive short-lived AWS credentials from the user's SSO identity, automatically, per workload.**

### How it works — the flow to narrate in the interview
1. SSO must be enabled: user logs into Domino through Keycloak federated to the corporate IdP.
2. The IdP's SAML assertion carries **AWS federation attributes** — the list of IAM role ARNs this user is entitled to assume (managed by the IdP/AD group membership, not by Domino).
3. Keycloak's mappers (`aws-roles-mapper` and friends) pass those attributes through into the user's Domino session.
4. When the user starts a workspace or job, Domino calls **AWS STS `AssumeRoleWithSAML`** using the user's assertion, receiving **temporary credentials** for the entitled role(s).
5. Domino writes them into the running pod as a standard **AWS credentials file** at `/var/lib/domino/home/.aws/credentials`, with the `AWS_SHARED_CREDENTIALS_FILE` environment variable pointing at it — so boto3, the AWS CLI, and R packages all just work with zero code changes.
6. Domino **refreshes the credentials periodically** during the session so they don't expire mid-work — which is why each propagated IAM role needs an **assume-self policy** (the role must be allowed to re-assume itself for refresh).

**The one-sentence summary to memorize:** *"User identity flows IdP → Keycloak → Domino, which exchanges the SAML assertion for temporary STS credentials per entitled role and injects a periodically-refreshed AWS credentials file into every run — so users get their own least-privilege AWS access with no long-lived keys anywhere."*

### The alternative/complementary mechanisms (senior differentiator)
- **IRSA (IAM Roles for Service Accounts):** Kubernetes-native — the pod's service account is trusted by an IAM role via the EKS cluster's OIDC provider. Used for *platform services* and can be used for workloads. Operational gotcha worth quoting: the OIDC provider is unique per EKS cluster, so **blue-green cluster upgrades break every IRSA trust policy** until they're updated with the new cluster's OIDC endpoint — painful when the IAM roles are owned by a different team.
- **EKS Pod Identity:** AWS's newer mechanism solving that — role association is by cluster/namespace/service-account name through an EKS API, no per-cluster OIDC trust policy edits. Better for service accounts that assume one fixed role.
- **These coexist:** SAML credential propagation for *human users* (they can hold multiple roles and switch), Pod Identity/IRSA for *service accounts and automation*. Domino service accounts can't use user credential propagation — automation jobs need the pod-identity path. Knowing that boundary is genuinely advanced.
- **IAM auth for Domino Data Sources:** admins can enable `AWSIAMRole` as an auth type for S3/Redshift/MySQL/Postgres data sources in Central Config, so data source connections ride the propagated role instead of stored passwords.

### Q&A

**Q4. "How would users in Domino access S3 securely?" (they WILL ask this)**
> Give the flow above, then close with: "The security wins: least privilege per user, no long-lived keys to leak or rotate, every S3 action in CloudTrail attributable to the individual's assumed role session — which is exactly the attributability GxP data integrity expects. And role entitlement is governed in AD/the IdP, so access reviews stay in the corporate identity process."

**Q5. "A user says AWS access stopped working inside their workspace. Triage it."**
> "Localize in this order:
> **(1) Is the credentials file there?** In the workspace terminal: `echo $AWS_SHARED_CREDENTIALS_FILE` and check `/var/lib/domino/home/.aws/credentials` exists and lists the expected role profile. Missing entirely → propagation isn't configured/enabled for them → check their IdP attributes (are they in the AD group that grants the role?).
> **(2) File exists but calls fail →** expired vs denied. `aws sts get-caller-identity` tells you *who AWS thinks they are*. Expired mid-session means the **refresh** is failing — check the workspace/pod logs (`kubectl logs <pod> -n <compute-namespace> -c <container>`), where refresh attempts and their errors are visible. A classic real error: `AccessDenied ... not authorized to perform: sts:TagSession` — the role's trust policy is missing an action Domino's assume flow needs; another classic is the missing **assume-self** permission breaking refresh.
> **(3) Credentials valid but the specific action denied →** not a Domino problem: the IAM role's *permission policy* doesn't allow that S3 action/bucket — route to whoever owns the role.
> **(4) Everyone broken at once →** something systemic: IdP SAML certificate rotated/expired, a trust policy changed, or an upstream IdP config change — check change records first.
> The framework: **identity present? → assumable? → refreshable? → permitted?** Four layers, each with a distinct owner."

**Q6. "What breaks with AWS roles during a Domino blue-green upgrade?" (advanced — huge if it lands)**
> "If any workloads or services use IRSA, the new cluster has a **new OIDC provider**, so every IAM role trust policy pointing at the old cluster's OIDC endpoint stops working until updated — and trust policy updates are eventually consistent, up to ~20 minutes under load. Mitigations: inventory role↔service-account mappings before the upgrade, script the trust policy updates into the cutover plan, or move service-account roles to EKS Pod Identity, which doesn't embed the per-cluster OIDC endpoint. SAML-based user credential propagation is unaffected by cluster identity — different trust chain — which is another reason the human/service split of mechanisms is wise."

---

## PART 3: Day-to-Day Administration Q&A

**Q7. What does a Domino admin actually do day to day?**
> "Ticket triage tiers into: workspace/job failures (mostly Kubernetes-level diagnosis), environment problems (image builds, package conflicts), access issues (Keycloak/permissions/credential propagation), and data connectivity. Beyond tickets: curating the compute environment catalog, managing hardware tiers against capacity and cost, user/org lifecycle, monitoring dashboards and alert response, executing upgrades and patches through change control, backup verification, and driving the automation that shrinks all of the above."

**Q8. How do you manage compute environments well at org scale?**
> "Treat environments as a governed product, not a free-for-all: a small set of **blessed base environments** (Python, R, GPU variants) that the platform team owns, versioned and pinned; teams extend from those rather than from random internet images. Every environment revision is immutable and recorded per run — the reproducibility story. Admin disciplines: pre-publish smoke tests (start a workspace from the new revision before users get it), vulnerability scanning of images, pruning unused revisions (registry bloat → node disk pressure), and keeping CUDA/driver alignment with GPU nodes. Most 'platform is broken' Mondays trace back to an ungoverned Friday environment edit."

**Q9. How do you monitor a Domino deployment?**
> "Domino bundles Prometheus/Grafana. Three altitude levels: **platform services** (are the ~dozens of platform pods healthy — frontend, dispatcher, MongoDB, RabbitMQ, Keycloak; restart counts, error rates), **cluster/infrastructure** (node capacity and disk pressure, autoscaler behavior, EFS throughput/burst credits, certificate expiry), and **user experience** (workspace start latency, job queue depth, model API error rates/latency). The senior move is alerting on the *user-experience* signals — queue depth rising and workspace-start p95 degrading page you before the ticket flood — plus synthetic checks: a scheduled canary job that starts a workspace and touches each data source hourly."

**Q10. Backup and disaster recovery for Domino?**
> "What must be protected: the **metadata stores** (MongoDB — projects, users, run history; Postgres — Keycloak's data), **blob storage** (S3 — project files/artifacts; protected via versioning and cross-region replication), **Datasets** (EFS — AWS Backup), and the **installation config** (in Git). The cluster itself is cattle — rebuildable from IaC — it's the state that's sacred. DR posture: documented RPO/RTO, restores actually *rehearsed* (an untested backup is a hope, and in GxP the DR procedure is itself a validated, evidenced document), and runbooks for full-region recovery: rebuild cluster from Terraform, reinstall Domino from the versioned config, restore data stores, repoint DNS."

**Q11. Walk me through executing a Domino upgrade end to end. (ties everything together)**
> "Plan: read release notes for breaking changes and the Kubernetes compatibility matrix; raise the change request with risk assessment, rollback plan (metadata DB restore + prior version redeploy), and validation checklist. Rehearse: run the upgrade in staging built from the same config; execute the full smoke-test suite — login via SSO, workspace per tier, job, model API call, each data source, **and AWS credential propagation** (start a workspace, verify the credentials file appears and refreshes). Execute: maintenance window, comms out, backups taken and verified immediately before, upgrade run, checklist executed, hypercare monitoring after. Document: evidence attached, change closed honestly; in GxP, validation impact assessed up front and re-qualification evidence filed. And if it's a blue-green cluster upgrade, the IRSA trust-policy inventory from Q6 is on the plan."

---

## PART 4: Scenario Questions

**S1. "Monday morning: 40 tickets — nobody's workspace starts. Go."**
> "That volume means platform-level, not user-level — declare an incident, comms out, stop triaging individual tickets. First question: **what changed over the weekend?** Check change records and any automated updates. Then the workspace-creation chain top-down: dispatcher and platform pods healthy (`kubectl get pods -n domino-platform`)? RabbitMQ processing? Are workspace pods being created in the compute namespace at all — and if created, what state: Pending (capacity/autoscaler/quota — did an AWS quota or node group change?), ImagePullBackOff (registry down, or an environment revision published Friday broke the default env), CrashLoop (bad image), or Running-but-unreachable (ingress/cert — did a TLS certificate expire Sunday night? Classic Monday incident). Mitigate at the failing layer, restore, then RCA — and if it was the Friday environment edit, the corrective action is the pre-publish smoke gate from Q8."

**S2. "A data scientist insists their job is 'stuck' for 2 hours. It shows Running."**
> "Running in Domino means the pod is running — not that the code is progressing. Two branches: **platform-stuck** — check the pod: is it actually scheduled and its containers up, or is 'Running' hiding an init container wedged on a dataset mount (EFS hiccup) or an image pull that hasn't completed? **code-stuck** — pod healthy, so look at resource telemetry: CPU pinned at the limit = throttled compute, working but slow; CPU near zero = waiting on something — a database query, an external API, a lock; memory climbing toward the limit = about to OOM. I'd share the evidence with the user ('your process has been at 0% CPU waiting on a connection to X since 10:02') — turning 'the platform is broken' into a collaborative diagnosis is half this job's craft."

**S3. "Security asks: prove that user X could only ever access bucket Y. Can you?"**
> "Yes — and this is the payoff of the credential propagation design. Chain of evidence: the IdP/AD group membership history shows which AWS roles X was entitled to (identity team's records); Keycloak event logs show X's authentications; AWS **CloudTrail** shows every STS AssumeRoleWithSAML call and every S3 API call under X's role session — attributable to the individual because each user assumes the role with their own session identity, no shared credentials anywhere. Plus Domino's own audit trail of which runs X executed. That end-to-end attributability — identity to action — is precisely what a GxP or security audit wants, and it exists *because* we never used shared keys."

**S4. "We want to onboard a new team of 30 data scientists next month. What's your plan?"**
> "Access: create/confirm their AD groups, map to Domino organizations, and register their AWS role entitlements in the IdP so credential propagation just works on day one. Capacity: forecast their tier usage, adjust node group maxima and quotas — and check the GPU quota story early, it has lead time. Environments: confirm the blessed environments cover their stack, or build one with them *before* day one. Data: their data sources configured and connectivity tested from a compute pod. Enablement: a short onboarding doc plus office hours the first week — front-loading enablement measurably cuts the ticket wave. And instrument it: watch their queue times and failure rates the first two weeks to catch sizing misses fast."

---

## PART 5: Rapid-Fire Facts (skim before the interview)

- Keycloak console: `https://<domino-domain>/auth/` — realm **DominoRealm** (Master = Keycloak admin only); initial admin password in the `keycloak-http` Kubernetes secret
- Auth chain: **User → Domino → Keycloak → corporate IdP (SAML/OIDC) → assertion back → token to Domino**
- Login-loop classics: **mismatched redirect URI, stale client secret, expired SAML cert, clock skew**
- `domino-play` client mappers include: **aws-roles-mapper, aws-session-duration-mapper, aws-role-session-name-mapper, domino-group-mapper**
- Credential propagation: requires SSO; **AssumeRoleWithSAML** → temp creds → file at **`/var/lib/domino/home/.aws/credentials`**, pointed to by **`AWS_SHARED_CREDENTIALS_FILE`**; refreshed during session; roles need **assume-self**; watch for **`sts:TagSession`** trust-policy errors
- Triage ladder for AWS access: **present? → assumable? → refreshable? → permitted?**
- **IRSA** = per-cluster OIDC trust (breaks on blue-green cluster swaps); **EKS Pod Identity** = association by API, survives cluster swaps; humans → SAML propagation, service accounts → pod identity
- Data sources can use **AWSIAMRole** auth type (enabled via Central Config) to ride propagated roles
- Backup priorities: **MongoDB + Postgres (metadata), S3 (blob), EFS (datasets), install config in Git** — the cluster is cattle, the state is sacred
- Synthetic monitoring: scheduled canary that starts a workspace + touches each data source

---

## FINAL RECALL STORY (read 3 times)

> *A scientist logs in: Domino redirects to **Keycloak**, which federates to corporate **Azure AD via SAML**; the assertion carries her **AWS role entitlements**, which the **aws-roles-mapper** passes into her session. She starts a workspace — a pod in the compute namespace — and Domino calls **AssumeRoleWithSAML**, writing temporary credentials to **`/var/lib/domino/home/.aws/credentials`**, refreshed all session (her role has **assume-self**). Her boto3 code reads S3 with *her* permissions; **CloudTrail** attributes every call to her session — audit-ready. Tuesday she reports AWS 'broken': the file exists, `sts get-caller-identity` works, but refresh errors show **sts:TagSession denied** in the pod logs — the IAM team's trust-policy change last night; change record confirms it; reverted via emergency change. Next month's **blue-green upgrade** plan includes updating every **IRSA** trust policy for the new cluster's OIDC provider — the automation service accounts use **EKS Pod Identity** now, so they don't care. And Monday's 40-ticket storm? Friday's unreviewed environment edit — ImagePullBackOff across every default workspace; rolled back the environment revision, then made pre-publish smoke tests mandatory. The platform got boring again, which is the whole job.*

Retell that story and you own the hardest questions this interview can throw on Domino administration.
