# Change & Release Management Interview Q&A (ServiceNow Context)
### Built for recall: read 2–3 times, and the frameworks will stick

---

## MEMORY HOOKS — learn these five lines first, everything else hangs off them

1. **"Incident restores, Problem explains, Change modifies."** (the three ticket types and their jobs)
2. **"3 change types: Standard, Normal, Emergency."** (pre-approved / CAB-approved / approved-after)
3. **"7 states: New → Assess → Authorize → Scheduled → Implement → Review → Closed."** (ServiceNow change lifecycle)
4. **"Every change needs 4 things: risk assessment, test evidence, rollback plan, validation plan."**
5. **"In GxP: no undocumented change ever touches production. The documentation IS the compliance."**

---

## SECTION A: Foundations

**Q1. What is change management and why does it exist?**
> "Change management is the controlled process for modifying production systems — the goal is to maximize successful changes while minimizing disruption to services. Statistically, a large share of outages are caused by changes we made ourselves, not by hardware failing. So the process forces every change to answer four questions before it happens: What's the risk? Has it been tested? How do we roll back? How do we verify success? In ServiceNow, this lives in the Change Management module, where every change is a record with an audit trail — which in a life-sciences environment is also a regulatory requirement, not just good hygiene."

**Q2. What's the difference between an Incident, a Problem, and a Change? (VERY commonly asked)**
> "**Incident** = something is broken right now; the goal is to restore service fast — even a workaround counts. **Problem** = the underlying root cause behind one or more incidents; the goal is to explain and permanently eliminate it — this is where RCA lives. **Change** = any controlled modification to production — often the *output* of a problem record (the permanent fix gets implemented via a change request).
> The flow ties together: users can't start workspaces (Incident, I restart the failing service — restored). Why did it fail? RCA finds a memory leak (Problem record). The vendor patch that fixes it goes to production through a Change request. In ServiceNow these three records are linked to each other, so an auditor or a new engineer can trace the whole story."
> **Memory hook: Incident restores, Problem explains, Change modifies.**

**Q3. What are the three types of changes? Give examples from platform work.**
> "**Standard change** — low-risk, routine, pre-approved with a documented procedure. No CAB needed each time; the template itself was approved once. Example: adding a compute node to the cluster, rotating a non-production credential, a routine certified environment package update.
> **Normal change** — anything non-routine with meaningful risk. Goes through the full lifecycle: risk assessment, peer/technical approval, CAB review, scheduled window. Example: a Domino version upgrade, a Kubernetes version upgrade, changing storage configuration.
> **Emergency change** — needed *now* to resolve or prevent a major incident; you get expedited approval (an emergency CAB or a designated approver) and complete full documentation retrospectively. Example: production platform is down, the fix requires a config change — you don't wait for Tuesday's CAB, but you also don't skip the record.
> The senior nuance: a healthy organization keeps making more things Standard over time — automating and templating routine work so the CAB only spends attention on genuinely risky changes."

**Q4. Walk me through the ServiceNow change lifecycle (the states).**
> "**New** — I create the change request (CHG record): description, reason, affected Configuration Items from the CMDB (e.g., 'Domino production platform'), planned dates.
> **Assess** — risk and impact analysis; ServiceNow calculates a risk score from impact × likelihood questions; peer/technical reviews happen here.
> **Authorize** — approvals: change manager, and for higher-risk changes the CAB. In GxP shops, Quality Assurance is often a mandatory approver here.
> **Scheduled** — approved and locked into a maintenance window on the change calendar; communications go out to affected users.
> **Implement** — execute during the window, following the documented implementation plan step by step; if things go wrong beyond agreed thresholds, execute the backout plan.
> **Review** — post-implementation review (PIR): did it work, was the window respected, any incidents caused? Attach validation evidence.
> **Closed** — with a closure code: successful, successful with issues, or unsuccessful/rolled back.
> **Memory hook: New → Assess → Authorize → Scheduled → Implement → Review → Closed.**"

**Q5. What is the CAB?**
> "The Change Advisory Board — a recurring meeting (often weekly) where the change manager, technical leads, service owners, and in pharma usually QA, review upcoming normal changes. They're checking: is the risk assessment honest, does the rollback plan actually work, do changes collide with each other or with business events (a release freeze during a regulatory submission, for example). The CAB doesn't do the technical review — that happens before, in Assess — the CAB does the *business risk and scheduling* judgment. There's usually also an eCAB — a small emergency CAB that can convene fast for emergency changes."

**Q6. What must a good change request contain? (know this cold — it's the "4 things" hook)**
> "Beyond the description and affected CIs: **(1) Risk & impact assessment** — what could go wrong, who is affected, blast radius. **(2) Test evidence** — proof it was executed successfully in a lower environment; for a Domino upgrade, that's the full smoke-test checklist run in staging. **(3) Backout/rollback plan** — not the sentence 'we will roll back,' but concrete steps: restore MongoDB from the pre-change backup, redeploy version X, validation steps to confirm rollback worked, and the *decision criteria* — at what point, or by what time in the window, do we pull the trigger. **(4) Validation plan** — how we prove success afterward: the smoke-test checklist, monitoring dashboards to watch, and for how long.
> A rollback plan that has never been tested is a hope, not a plan — for major changes I want the rollback itself rehearsed in staging."

---

## SECTION B: Release Management

**Q7. Change management vs release management — what's the difference?**
> "Change management governs *individual modifications* and their risk. Release management is the discipline of *packaging, scheduling, and deploying* sets of changes into environments — it owns the pipeline: build → test → staging → production, release calendars, release notes, and coordination when a release bundles many changes. A release typically *contains or triggers* one or more change requests. Concretely: 'Domino platform release for Q3' is a release — it bundles the Domino version upgrade, two environment updates, and a monitoring improvement; each production-touching piece rides its own change record, coordinated under the release plan."

**Q8. Describe a healthy release process for a platform like Domino.**
> "Environments: dev → staging (a scaled-down replica of production, same Domino version, same integrations) → production. Every release is tested in staging with the standard validation checklist. Releases are scheduled on a calendar visible to stakeholders, avoiding business-critical periods — in pharma, you check for freeze windows around regulatory submissions or critical study milestones. Release notes go to users before and after. Version everything — the installer config file lives in Git, so every release is reproducible and diffable. And post-release, a defined hypercare window where the team watches dashboards and prioritizes any fallout."

**Q9. What is a change freeze / blackout window?**
> "A period when non-emergency changes are prohibited — typically around year-end, major business events, or in pharma around regulatory submissions and audits. The reasoning: even a well-tested change carries residual risk, and during critical periods the cost of any disruption outweighs the benefit of the change. Emergency changes are still allowed with heightened approval. As a platform engineer I check the freeze calendar before scheduling anything."

---

## SECTION C: GxP Overlay (this is what makes THIS role different)

**Q10. How does change management differ in a GxP/pharma environment?**
> "Four big additions on top of standard ITIL:
> **(1) Validated system status** — the platform has been formally qualified (IQ/OQ/PQ: Installation, Operational, Performance Qualification). A change can *break validation*, so every change gets a **validation impact assessment**: does this change affect the validated state? If yes, re-qualification activities (re-running qualification test scripts) are part of the change plan.
> **(2) QA as an approver** — Quality Assurance signs off on GxP-impacting changes, not just IT.
> **(3) Documentation rigor** — evidence isn't optional or informal; executed test scripts, signatures, and timestamps become part of the system's validation package that auditors can inspect years later. The principle: *if it isn't documented, it didn't happen.*
> **(4) Deviations and CAPA** — if a change fails or something was done outside the approved plan, you raise a deviation record, and the fix to the *process* goes through CAPA — Corrective and Preventive Action. CAPA is essentially the quality system's version of a postmortem action item, but formally tracked to closure.
> The mindset shift: in a startup, process overhead feels like friction. Here, patient-impacting research runs on this platform — the discipline *is* the job."

**Q11. What is CAPA?**
> "Corrective and Preventive Action — the formal quality-system mechanism for fixing problems at the process level. **Corrective** = eliminate the cause of an existing nonconformity (this failure happened — fix why). **Preventive** = eliminate the cause of a *potential* one (this could happen — fix it before it does). Each CAPA has an owner, due date, effectiveness check, and is tracked to closure. In my world: a failed Domino upgrade would produce an incident, an RCA, and a CAPA like 'staging environment didn't match production configuration — implement automated config drift detection between environments.'"

---

## SECTION D: Scenario Questions (use these structures verbatim)

**S1. "Your change failed mid-implementation in production. What do you do?"**
> "The backout decision comes first, and it shouldn't be a debate at 2am — the change plan already defined the criteria: 'if not validated by hour 3 of the 4-hour window, roll back.' So: assess whether I'm within recoverable scope and time; if the failure isn't quickly diagnosable, **execute the documented backout plan** — for a Domino upgrade, that's restore metadata DB backups, redeploy prior version, run the validation checklist to confirm the rollback restored service. Communicate to stakeholders per the plan. Close the change as unsuccessful — honestly; gaming the closure code destroys your change-success metrics' meaning. Then the follow-up: RCA into why staging didn't catch it, a problem record, and in a GxP shop a deviation + CAPA. The worst possible move is improvising untested fixes in production past the window — that's how a failed change becomes a multi-day outage."

**S2. "Production is down at 2am. The fix requires a production change. Walk me through it."**
> "This is the emergency change path. First, incident management runs in parallel — comms, severity declared. For the fix: I don't skip approval, I use the *expedited* approval — page the designated emergency approver or convene the eCAB; in ServiceNow I raise an Emergency change, capturing at minimum what I'm changing and the risk, even if brief. Implement the fix, verify restoration, then complete the full documentation retrospectively — detailed record, evidence, linked to the incident. Emergency doesn't mean undocumented; it means approval is faster and paperwork finishes after. And if I find myself raising emergency changes regularly, that's a process smell — it means normal planning is failing somewhere, and that pattern itself deserves a problem record."

**S3. "A senior data scientist asks you to 'just quickly' make a production config change without a ticket. What do you do?"**
> "I don't do it, but I don't just stonewall either. I explain the why — this is a validated GxP platform, an undocumented production change breaks its validated status and creates audit findings that cost far more than the ticket takes — and then I make the right path easy: if it's genuinely urgent I'll raise the change myself right now and get it expedited; if it's routine and recurring, I'll propose we make it a pre-approved Standard change or, better, a self-service automation so they never need to ask me again. Most 'skip the process' pressure is really a complaint that the process is slow — the senior move is fixing the speed, not bypassing the control."

**S4. "How do you handle a change that caused an incident, discovered a day later?"**
> "First restore service — incident process, and the fastest mitigation is usually rolling back the change, even a day later, unless data has diverged in ways that make rollback riskier than fixing forward; that's an explicit risk decision to make, not assume. Link the incident to the change record in ServiceNow — this correlation is exactly why we record changes. Then the PIR/RCA: why did validation not catch it — was the validation checklist incomplete, was monitoring missing a signal, did the issue only manifest under real load? The corrective actions typically improve the validation plan and add the missing alert. This scenario is also my answer to why I check 'what changed recently?' as one of the first steps in *any* incident triage — most incidents trace back to a change."

**S5. "How would you improve a change process that engineers complain is too slow?"**
> "Measure first: where does time actually go — waiting for CAB, writing documentation, testing? Then three levers: **(1) Expand the Standard change catalog** — identify high-volume, historically-safe change types and template + pre-approve them; this removes the bulk of CAB queue time. **(2) Automate the evidence** — if the CI/CD pipeline automatically attaches test results and deploy logs to the ServiceNow record (via its API), documentation stops being manual toil. **(3) Risk-based CAB** — low-risk normal changes get change-manager approval only; CAB attention concentrates on genuinely high-risk items. The goal isn't less control, it's control proportional to risk — that's actually more ITIL-correct, not less."

**S6. "What change management metrics would you track?"**
> "**Change success rate** — percentage of changes closed successful without causing incidents; the headline health metric. **Emergency change ratio** — high means poor planning upstream. **Unauthorized change count** — should be zero; anything else is a serious control failure, and in GxP an audit finding. **Change-related incident rate** — incidents with a causing change linked. **Lead time per change type** — to spot process friction (feeds the S5 answer). **Rollback rate** — how often backout plans get executed. I'd trend these in ServiceNow dashboards and bring them to service reviews — metrics turn change management from bureaucracy into an improvable system."

---

## SECTION E: Quick ServiceNow Vocabulary (skim before the interview)

| Term | Meaning in one line |
|---|---|
| **CHG / INC / PRB records** | Change, Incident, Problem ticket types (the prefixes on ticket numbers) |
| **CMDB** | Configuration Management Database — inventory of systems (CIs) and their relationships; changes declare which CIs they touch |
| **CI (Configuration Item)** | Any managed component — "Domino Production Platform," "EKS prod cluster" |
| **CAB / eCAB** | Change Advisory Board / emergency version for urgent approvals |
| **PIR** | Post-Implementation Review — the "did it work?" step before closing |
| **Change calendar** | Shared schedule of all changes — used to spot collisions and respect freezes |
| **Standard change catalog** | Library of pre-approved templated routine changes |
| **SLA/OLA** | Service commitments to customers / between internal teams |
| **Knowledge Base (KB)** | Where RCA learnings and runbooks are published |
| **Deviation / CAPA** | (GxP) Record of departing from approved process / formal corrective & preventive action |
| **IQ/OQ/PQ** | (GxP) Installation / Operational / Performance Qualification — the validation evidence trio |

---

## FINAL RECALL DRILL — if you remember nothing else, remember this story

Tell yourself this story twice before bed; it contains the entire topic:

> *Users can't start workspaces → I raise an **Incident** and restore service with a restart. RCA finds a memory leak → **Problem** record. The vendor patch goes to production via a **Normal change**: I write the CHG in ServiceNow with the 4 essentials (risk assessment, staging test evidence, tested rollback plan, validation checklist), it moves **New → Assess → Authorize** (change manager + CAB + QA because we're GxP) **→ Scheduled** (window on the calendar, users notified) **→ Implement** (patch applied; had it failed, the backout criteria say when to restore backups and redeploy) **→ Review** (smoke tests pass, evidence attached) **→ Closed** successful. The routine node-scaling I do weekly is a **Standard change** — pre-approved, no CAB. The night production died at 2am was an **Emergency change** — eCAB approval fast, paperwork completed after. And the time the upgrade failed, we rolled back, raised a **deviation**, and the **CAPA** was fixing the staging/production config drift that let it slip through.*

Read that paragraph three times and you can answer every question in this document.
