# Domino Upgrade Runbook — From Zero to Executing It Yourself
### Every phase, every step, and the WHY behind each — including the Kubernetes (EKS) upgrade underneath

---

## PART 0: The Big Picture First

**What a Domino upgrade actually is:** Domino is ~50–70 microservices deployed on Kubernetes via Helm charts, driven by an installer (**fleetcommand-agent**, operated through the **ddlctl** CLI in modern versions). "Upgrading Domino" = pointing the installer at a new version tag and letting it reconcile every chart to the new release — plus database schema migrations along the way. That's why the risky parts are: version compatibility, the databases (MongoDB especially), and anything someone changed by hand outside the installer.

**The layer cake, and the order it must be upgraded in (memorize this):**
```
   Layer 4: Domino application  <- upgraded LAST
   Layer 3: Kubernetes add-ons (CNI, CSI, CoreDNS, autoscaler)
   Layer 2: Kubernetes (EKS control plane + node groups)
   Layer 1: AWS infrastructure (instance types, AMIs)
```
**Rule: infrastructure first, Domino last.** Each Domino version supports specific Kubernetes versions (the compatibility table). If the target Domino needs a newer Kubernetes than you run, you upgrade EKS *first*, validate, and only then upgrade Domino. Never both in one window — if something breaks, you must know which layer did it.

**The three golden principles:**
1. **The installer config file is sacred.** You upgrade by re-running the installer with (an upgraded copy of) the SAME configuration you installed with, only the version changed. Never hand-edit cluster resources; the config in Git IS the deployment.
2. **Rehearse in a lower environment first.** Same config shape, same versions, same steps. The staging run is where surprises are allowed.
3. **You can't roll forward out of every hole.** Backups of the metadata databases taken immediately before, plus a written rollback plan, are what make the upgrade a controlled risk instead of a gamble.

---

## PHASE 1: Planning & Reading (1–2 weeks before; zero technical risk, highest leverage)

### Step 1.1 — Establish current and target versions
Record what you run now: Domino version (admin UI or ask the platform), Kubernetes version (`kubectl version`), EKS add-on versions, and the fleetcommand-agent/installer version last used.

**Why:** Upgrades are path-dependent. Some components only step one version at a time — MongoDB, for example, must go 4.0 -> 4.2 -> 4.4 sequentially, which means some Domino version jumps require an **intermediate Domino upgrade** first. You cannot plan a route without knowing the starting point.

### Step 1.2 — Read the release notes and compatibility matrix for EVERY version between current and target
From Domino's releases page, get the **$FLEETCOMMAND_AGENT_TAG** for the target release. Read: breaking changes, required Kubernetes versions, configuration key changes, and any "action required" notes in the fleetcommand-agent release notes for each intermediate installer version.

**Why:** The release notes are where Domino tells you, in advance, exactly what will bite you. Historic examples of the kind of thing they contain: nginx-ingress needing config changes to survive an upgrade, Apps being stopped by an upgrade and needing a restart call afterward, new required config fields. Every "surprise" in a failed upgrade was usually printed in a release note nobody read.

### Step 1.3 — Decide: does Kubernetes need upgrading too?
Compare the target Domino's supported Kubernetes range with your current EKS version.

**Why this decision gates everything:** If yes, you now have TWO change windows (EKS first, Domino later — separate windows), and critically: **user executions can keep running through a Domino-only upgrade (5.3.0+), but MUST be stopped for a Kubernetes upgrade** — because node replacement kills every pod, including 6-hour training jobs. This changes your user communications completely.

### Step 1.4 — Raise the change request(s)
Normal change in ServiceNow: description, risk assessment, the maintenance window(s), the full implementation plan (this runbook, versioned), the rollback plan (Phase 6), validation checklist (Phase 5), and — in a GxP shop — the validation impact assessment, with QA in the approval chain.

**Why:** Beyond compliance: writing the plan forces the thinking. The change record's rollback criteria ("if not validated by hour 3 of 4, roll back") is a decision made calmly in advance instead of at 2am under pressure.

### Step 1.5 — Communicate to users
Announce the window(s) ~1 week out and again the day before: what's unavailable, for how long, what happens to running work (paused vs must-stop), and where status updates will appear.

**Why:** A platform team's reputation is made less by uptime than by predictability. And for a K8s upgrade, users need enough notice to checkpoint long-running jobs.

---

## PHASE 2: Pre-Upgrade Health & Backups (day of / day before)

### Step 2.1 — Run the Domino Admin Toolkit
The toolkit (bundled with Domino 5.5+, deployable via `toolkit.sh` otherwise) runs a battery of checks inside the cluster: service health, known infrastructure bugs, configuration problems.

**Why:** Domino's own documented first step. Upgrading an already-sick deployment guarantees a bad night — you can't tell upgrade damage from pre-existing damage. Fix every red finding BEFORE the window; a clean bill of health is your baseline.

### Step 2.2 — Check for configuration drift (Domino 6.0+: Helm drift check)
Verify no Helm releases have drifted from their installer-managed desired state — i.e., nobody `kubectl edit`ed platform resources by hand.

**Why:** The installer will *overwrite* out-of-band changes when it reconciles. If some hand-applied fix is currently load-bearing (it happens), the upgrade silently removes it and something breaks an hour later with no obvious cause. Find these now: either fold them properly into the config file, or note them in the plan for re-application post-upgrade.

### Step 2.3 — Back up EVERYTHING stateful, and VERIFY the backups
- **MongoDB** (projects, users, run history — Domino's brain) — manual Domino backup procedure
- **Postgres** (Keycloak — identities/SSO config)
- Confirm **S3** blob storage versioning is on (it should always be)
- **Snapshot the current installer config file** — copy `domino.yml` to `domino-orig-<date>.yml` before touching it
- Record current versions of everything (your "known good" manifest)

**Why each:** MongoDB/Postgres restore is the core of the rollback plan — an unverified backup is a hope, so test-restore or at least integrity-check it. The config copy is what "run the installer with the old version" needs if you roll back. Domino's docs are explicit: back up before upgrading; data loss is rare but catastrophic failures happen.

### Step 2.4 — Capture a validation baseline
Run your smoke-test checklist (Phase 5) against the CURRENT version and save the results.

**Why:** After the upgrade, "is this broken or was it always like that?" is a real question. A baseline answers it in seconds and keeps the war room focused on genuine regressions.

---

## PHASE 3 (ONLY IF NEEDED): Upgrade Kubernetes / EKS First — Its Own Window

> EKS upgrades move **one minor version at a time** (1.28 -> 1.29 -> 1.30 needs two passes). Control plane first, then nodes, then add-ons — always that order.

### Step 3.1 — Audit for deprecated/removed APIs
Run a deprecation scanner (`pluto`, `kubent`) against the cluster and your manifests for APIs removed in the target Kubernetes version.

**Why:** Kubernetes removes old API versions on a schedule. Anything still using a removed API stops applying/working after the control plane upgrade — this is the #1 cause of "the upgrade broke things subtly." Fix manifests BEFORE, not after.

### Step 3.2 — Stop user executions / maintenance mode
Put Domino into **maintenance mode** (the domino-maintenance-mode tooling pauses apps, model APIs, restartable workspaces, scheduled jobs) and let running jobs finish or stop them with notice.

**Why:** Node replacement evicts every pod. A paused, quiesced platform turns "we killed 40 people's work" into "the window we announced."

### Step 3.3 — Upgrade the EKS control plane
Via Terraform (change the cluster version in code — the right way, keeping IaC truthful) or the console/CLI. AWS performs a rolling, managed upgrade of the API servers; takes tens of minutes.

**Why control plane first:** Kubernetes supports kubelets being *older* than the control plane (within skew limits), never newer. Order is therefore forced: brain, then muscle.

### Step 3.4 — Upgrade node groups (the muscle)
Update the node group Kubernetes version / launch template AMI. EKS managed node groups do a **rolling replacement**: new nodes join, old nodes are **cordoned and drained** (pods evicted gracefully, respecting PodDisruptionBudgets), then terminated — batch by batch.

**Why rolling + why this is the user-visible part:** capacity stays available throughout, but every pod moves — hence maintenance mode. Watch platform pods reschedule cleanly between batches; platform StatefulSets (MongoDB) deserve special attention as their volumes reattach.

### Step 3.5 — Upgrade the add-ons to the versions matched to the new Kubernetes
VPC CNI, CoreDNS, kube-proxy, EBS/EFS CSI drivers, cluster autoscaler.

**Why (the most-forgotten step in all of EKS operations):** each Kubernetes version expects matched add-on versions; skew here causes the weird intermittent failures — DNS flakiness, volume mount errors — that surface days later and look like ghosts. AWS publishes the version mapping per EKS release; make this an explicit checklist line.

### Step 3.6 — Validate Kubernetes itself, then Domino on top, then exit maintenance mode
Nodes Ready on the new version, platform pods all Running, then the Phase 5 smoke tests.

**Why validate before proceeding to the Domino upgrade (in its later window):** if Domino misbehaves NOW, the cause is the Kubernetes layer — you want that discovered and fixed while the change is singular and the rollback story simple.

---

## PHASE 4: The Domino Upgrade Itself

### Step 4.1 — Enter maintenance mode (or confirm execution policy)
For Domino >=5.3 pure-Domino upgrades, user executions can *typically* continue; maintenance mode is still the conservative, recommended posture for production — it pauses apps, model APIs, restartable workspaces, and scheduled jobs cleanly.

**Why the conservative choice:** "typically continue" is doing a lot of work in that sentence. In a regulated production environment, a quiet platform during the window costs users little and removes a whole category of mid-upgrade edge cases.

### Step 4.2 — Set up the installer environment
On your admin workstation/bastion with cluster access:
```bash
unset HISTFILE     # don't write the registry credentials into shell history
export QUAY_USERNAME=<provided by Domino>
export QUAY_PASSWORD=<provided by Domino>
export FLEETCOMMAND_AGENT_TAG=<installer tag for the TARGET Domino release>
```

**Why:** The installer image and Domino's charts/images are pulled from Domino's registry (quay.io) with your license credentials; the TAG pins exactly which release you're deploying. `unset HISTFILE` — small habit, real security.

### Step 4.3 — Upgrade the configuration file (don't hand-port it)
Modern (ddlctl) path — generate the new-format config FROM the existing deployment or existing file:
```bash
ddlctl create config --from-domino --agent-version $FLEETCOMMAND_AGENT_TAG
# or: ddlctl create config --from-file domino.yml --agent-version $FLEETCOMMAND_AGENT_TAG
```
This writes a timestamped, upgraded config (`config-$TAG.<timestamp>.yaml`). **Diff it against your previous config** and review every change; commit it to Git.

**Why the tool does the conversion:** config schemas change between installer versions — new required fields, renamed keys. The tool ports your values into the new schema correctly; hand-editing is where typos become outages. The diff review is your chance to catch anything the conversion changed that you didn't expect. (Legacy path: copy `domino.yml`, change the `version:` field to the new Domino version, run the same installer flow.)

### Step 4.4 — DRY RUN first
Generate the Domino custom resource with the operator in dry-run mode:
```bash
ddlctl create domino --config config-$FLEETCOMMAND_AGENT_TAG.<ts>.yaml \
  --agent-version $FLEETCOMMAND_AGENT_TAG --export > domino-cluster.yaml
# edit domino-cluster.yaml: spec.agent.spec.dryRunMode: "true", apply it
```
The dry run reports every change the upgrade would make versus what's currently deployed — Domino's equivalent of `terraform plan`.

**Why this is the single most protective step:** you read the intended changes BEFORE anything happens. Unexpected deletions, a chart change you didn't anticipate, drift about to be overwritten — all visible here, while everything still works. Review it like a plan file; attach it to the change record as evidence.

### Step 4.5 — Execute
Flip `dryRunMode` to `"false"` (or remove it) — the platform operator reconciles the deployment to the new version — or on the legacy path, run the installer directly. Then watch:
```bash
kubectl get pods -n domino-platform -w
```
Charts upgrade in sequence; pods cycle; schema migrations run. Expect 30–90 minutes. Success looks like the installer's completion message: *"Deployment complete. Domino is accessible at <your FQDN>."*

**Why watch rather than wait:** a chart stuck in a CrashLoop or a migration job failing is visible in minutes via pod states and logs — catching it early keeps you inside the window's decision points. Don't panic-restart things mid-reconcile; read first (Phase 6 has the failure playbook).

### Step 4.6 — Post-upgrade actions from the release notes
Whatever the notes said: restart Apps if the version transition stops them, re-apply any documented out-of-band changes from Step 2.2, component-specific follow-ups (e.g., historically, Elasticsearch pods needed an ordered delete-to-upgrade).

**Why:** The upgrade isn't the installer finishing; it's the release notes' checklist finishing.

---

## PHASE 5: Validation (the upgrade is not done until this passes)

Run the full smoke-test checklist — the same one from your baseline, ideally executed by a second person against a printed list:

1. **Login via SSO** (Keycloak chain, token issuance)
2. **Start a workspace on each hardware tier** including GPU (scheduling -> image pull -> ingress routing)
3. **Run a batch job to completion** (dispatcher -> queue -> executor)
4. **Read from every data source type** (credentials, network paths)
5. **AWS credential propagation**: credentials file present in a fresh workspace, and refresh working
6. **Write to a Domino Dataset** (EFS mounts)
7. **Call a Model API**; restart Apps and verify
8. **Environments build**: rebuild one environment revision successfully (registry + builder health)
9. **Admin functions**: admin UI, user management, Central Config reachable
10. **Monitoring**: Grafana dashboards green; no alert storm; error rates at baseline

Then: exit maintenance mode, announce completion, and enter **hypercare** — elevated monitoring and fast-lane triage for 24–48 hours.

**Why a written checklist executed twice (baseline + post):** it converts "seems fine" into evidence — which in GxP is literally the OQ-style record attached to the change, and everywhere is the difference between confidence and hope. Hypercare exists because some regressions only surface under real Monday-morning load.

---

## PHASE 6: When It Goes Wrong — Failure & Rollback Playbook

### The decision framework (agree on it BEFORE the window)
- **Installer/operator fails mid-run:** read the failing chart/job logs. Many failures are transient (image pull timeout, a slow readiness) — the reconcile/installer is **safe to re-run**; it's idempotent like Terraform and resumes converging. One informed retry is reasonable. Unknown-cause failure + window running out = rollback.
- **Rollback trigger:** the criteria written in the change record — e.g., "not validated by hour 3 of a 4-hour window" or "any Sev-1 regression without a known fix." The point of pre-agreeing: at 2am, you execute a decision, not a debate.

### The rollback procedure (what your Step 2.3 backups were for)
1. Re-enter/confirm maintenance mode.
2. **Restore MongoDB and Postgres** from the pre-upgrade backups — mandatory if schema migrations ran, because new-schema databases + old application code = corruption; this is WHY the backups are taken seconds before, not the night before.
3. Re-run the installer/operator with the **original config and previous version** (your `domino-orig-<date>.yml` snapshot / previous Git commit).
4. Run the validation checklist against the restored version; compare to baseline.
5. Exit maintenance mode; communicate honestly: "upgrade rolled back, service restored, rescheduling after RCA."

### Afterward — the part that makes you senior
Close the change as **unsuccessful** (honestly — gamed closure codes poison the metrics that protect you). Incident/deviation record as applicable. Blameless RCA centered on one question: **"why didn't the staging rehearsal catch this?"** — and the corrective action almost always improves the *rehearsal fidelity* (staging config drift from prod is the usual answer) or adds the missing item to the validation checklist. In GxP: deviation + CAPA, tracked to closure. Then reschedule and go again — a rolled-back upgrade executed cleanly is a success of process, not a failure of nerve.

---

## THE ONE-PARAGRAPH VERSION (memorize for the interview)

> "Upgrade order is infrastructure-first, Domino-last, never both in one window. Plan: read every release note and the Kubernetes compatibility table between current and target — some paths need intermediate versions because components like MongoDB step one version at a time — and raise the change with rollback criteria decided in advance. Pre-flight: Domino Admin Toolkit health check, drift check for out-of-band changes the installer would overwrite, verified backups of MongoDB and Postgres, a copy of the installer config, and a validation baseline. If Kubernetes needs it: separate window — deprecation audit, maintenance mode because node drains kill executions, control plane one minor at a time, then node groups rolling, then the matched add-ons everyone forgets. Domino window: maintenance mode, upgrade the config file with the installer's own conversion, **dry-run and review the plan like Terraform**, execute, watch the platform namespace, then the release-notes post-steps. Validate with the same written checklist you baselined — including credential propagation — hypercare after. And if it fails: re-run once informed, otherwise restore the databases, redeploy the old version from the old config, validate, and let the RCA fix the rehearsal, not the blame."

Deliver that paragraph and you've answered the "manage platform upgrades" bullet of the JD at a level most candidates with actual Domino experience couldn't articulate.
