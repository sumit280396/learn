# CI/CD Interview Guide (GitLab) — From Zero to Confident Answers
### Understand the concepts AND have the answers ready

---

## PART 0: The Mental Model (read first — everything builds on this)

**The problem CI/CD solves:**
Before CI/CD, releasing software looked like this: developers work in isolation for weeks, someone manually merges everything ("integration hell"), someone manually builds it, manually copies it to a server, manually restarts things — and every manual step is a chance for human error, done differently every time, by whoever happens to be on duty. Releases were rare, scary events.

**The idea, in one line each:**
- **CI — Continuous Integration:** every time a developer pushes code, a machine *automatically* builds it and runs tests. Broken code is caught in minutes, not weeks, while the change is still small and fresh in the author's head.
- **CD — Continuous Delivery:** every change that passes tests is automatically packaged and made *deployable* — production deployment is one button press away, always.
- **CD — Continuous Deployment (the other CD):** remove the button too — every passing change deploys to production automatically, no human in the loop.

**The assembly-line analogy:**
A pipeline is a factory assembly line for code. Code enters at one end (a `git push`), passes through stations (build → test → scan → package → deploy), and any station can reject the product and stop the line. The stations are identical every single run — that consistency, not just speed, is the real win. A human deploying at 5pm Friday is variable; a pipeline is not.

**GitLab vocabulary in 60 seconds (memorize these six words):**
- **Pipeline** — one complete run of the assembly line, triggered by a push, merge request, schedule, or manual click.
- **Stage** — a phase of the line (build, test, deploy). Stages run **in sequence**.
- **Job** — one task inside a stage (unit-tests, lint, docker-build). Jobs in the same stage run **in parallel**.
- **Runner** — the machine that actually executes jobs. GitLab (the server) orchestrates; runners do the work.
- **`.gitlab-ci.yml`** — the file in the repo root that defines all of it. Pipeline-as-code: versioned, reviewable, same as everything else.
- **Artifact** — files a job produces and passes to later jobs (a compiled binary, test reports, a Terraform plan file).

**Memory hook:** *"A pipeline has stages, stages have jobs, jobs run on runners, defined in one YAML file."*

---

## PART 1: Core Concept Questions & Answers

**Q1. What is CI/CD and why does it matter?**
> "CI is automatically building and testing every code change on push — catching integration problems in minutes while changes are small. CD is automating the path to production so every passing change is either deployable at a button press (Continuous Delivery) or deployed automatically (Continuous Deployment). The value isn't just speed — it's *consistency and small batch size*: every deployment runs the identical tested process, and shipping small changes frequently means when something breaks, the suspect list is one small diff, not six weeks of accumulated work. That directly reduces MTTR and change failure rate — and for a platform team, the pipeline itself becomes the enforcement point for quality gates, security scanning, and change-control evidence."

**Q2. Continuous Delivery vs Continuous Deployment? (classic trap question)**
> "Both automate everything up to production; the difference is the last step. **Delivery**: production deploy requires a human approval — the release is always *ready*, but a person pulls the trigger. **Deployment**: no human gate; every green pipeline goes to production automatically. Most regulated environments — certainly GxP — run Continuous Delivery, because the human approval maps to the change-management approval: the pipeline produces the evidence, the change record provides the authorization, a person clicks deploy inside the approved window."

**Q3. Walk me through a typical pipeline for a containerized application.**
> "Stages in order:
> **(1) Build** — compile/package the code, build the Docker image, tag it with the commit SHA (never rebuild later — the *same* image travels through all environments; that's a core principle: build once, promote everywhere).
> **(2) Test** — unit tests, linting, in parallel jobs since they're independent.
> **(3) Scan** — SAST (static code security analysis), dependency/container vulnerability scanning, secret detection. Shift-left security: cheaper to catch here than in prod.
> **(4) Publish** — push the image to a registry (GitLab's built-in Container Registry or ECR).
> **(5) Deploy to staging** — automatically; run integration/smoke tests against it.
> **(6) Deploy to production** — a **manual job** in GitLab (`when: manual`), gated on approval, ideally deploying the *exact* image that passed staging."

**Q4. What is a GitLab Runner? Types of executors?**
> "The runner is the agent that executes jobs — GitLab server schedules, runners work. They can be **shared** (available to all projects — gitlab.com provides these) or **specific/self-hosted** (your own machines — needed when jobs must reach private networks, like deploying into our EKS cluster). Executors define *how* a runner runs jobs: **shell** (directly on the machine — simple, but jobs pollute each other), **docker** (each job in a fresh container — the standard choice: clean, reproducible, you pick the image per job), and **kubernetes** (each job as a pod in a cluster — autoscaling CI capacity; the natural fit when you already operate EKS). For our platform I'd run self-hosted runners on Kubernetes: scaling, isolation, and network access to what we deploy."

**Q5. Explain the key parts of a `.gitlab-ci.yml`.**
> ```yaml
> stages: [build, test, deploy]
>
> build-image:
>   stage: build
>   image: docker:24            # container this job runs in
>   script:
>     - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA .
>     - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
>
> unit-tests:
>   stage: test
>   image: python:3.12
>   script: [pytest --junitxml=report.xml]
>   artifacts:
>     reports: { junit: report.xml }
>
> deploy-prod:
>   stage: deploy
>   script: [./deploy.sh]
>   environment: production
>   when: manual                # human gate = Continuous Delivery
>   only: [main]
> ```
> "Talking points: `stages` defines sequence; each job names its stage; `image` picks the container; `script` is the actual work; `artifacts` pass files/reports forward; `when: manual` creates the approval gate; `rules`/`only` control which branches trigger which jobs. And predefined variables like `CI_COMMIT_SHORT_SHA` give you traceable image tags for free."

**Q6. Artifacts vs cache — what's the difference? (common and commonly-fumbled)**
> "**Artifacts** are *outputs* passed between jobs in the same pipeline and stored by GitLab — the built binary, the test report, the Terraform plan file. They're about correctness: later jobs need them. **Cache** is an *optimization* reused across pipelines — dependency directories like `.pip` or `node_modules` so every run doesn't re-download the internet. Cache can be safely lost (just slower); artifacts cannot (pipeline breaks). One-liner: artifacts flow forward within a pipeline, cache flows across pipelines."

**Q7. How do you handle secrets in GitLab CI?**
> "Never in the YAML or the repo. GitLab **CI/CD variables** (Settings → CI/CD), marked **masked** (hidden in job logs) and **protected** (only exposed to pipelines on protected branches — so a developer can't print prod credentials from a feature branch pipeline). Better still for cloud: **OIDC federation** — GitLab issues a short-lived identity token per job, AWS trusts it and grants a role, so there are *no stored long-lived AWS keys at all*. That's the modern answer interviewers like. And for application secrets, integrate an external manager (Vault, AWS Secrets Manager) rather than pushing secrets through CI variables."

**Q8. What are environments and protected branches in GitLab?**
> "**Protected branches** — main/production branches where direct pushes are blocked and merge requests with approvals are required; also what the 'protected' variable flag keys off. **Environments** — GitLab's model of deploy targets (staging, production): each deployment is recorded against the environment, giving a deployment history — what's deployed where, by whom, since when — plus per-environment protection rules (only certain people can approve a production deploy). That deployment history is audit-trail gold in a regulated shop."

**Q9. What deployment strategies do you know? (very common; know all four)**
> "**Recreate** — stop old, start new; simple, has downtime.
> **Rolling** — replace instances gradually (Kubernetes' default Deployment behavior); no downtime, but two versions briefly coexist, and rollback is a re-roll.
> **Blue-green** — run a full new environment (green) beside the old (blue), test it, then switch traffic at once; instant rollback by switching back; costs double capacity during the switch.
> **Canary** — send a small slice of real traffic (say 5%) to the new version, watch error/latency metrics, then progressively increase; catches problems with minimal blast radius; the most operationally mature, needs good monitoring to be meaningful.
> Bonus distinction that impresses: **feature flags** decouple *deploy* from *release* — code ships dark and is switched on per-user/percentage, so 'rollback' becomes flipping a flag, no redeploy at all."

**Q10. What does 'shift left' mean?**
> "Moving quality and security checks earlier in the lifecycle — into the pipeline and even pre-commit — instead of finding problems in staging or production. Concretely in GitLab: SAST, dependency scanning, container scanning, secret detection, license checks run on every merge request, and the MR can't merge with critical findings. Rationale: the cost of a defect grows enormously the later it's found — a scanner comment on an MR costs minutes; the same flaw in production is an incident."

**Q11. How does CI/CD apply to infrastructure (tie-in to Terraform)?**
> "Same discipline, different artifact. Our Terraform pipeline: on merge request — `fmt`, `validate`, tflint, tfsec, then `terraform plan -out=plan.tfplan` with the plan posted to the MR as the review artifact; on approval/merge — apply *that saved plan file*, so exactly what was reviewed is what executes; production applies behind a manual gate tied to a change record; plus a scheduled nightly plan as drift detection. The pipeline turns infrastructure change from 'someone with credentials ran commands' into a reviewed, evidenced, serialized process — which is precisely what GxP change control wants."

**Q12. What makes a *good* pipeline? (senior-signal question)**
> "**Fast** — under ~10 minutes for the MR feedback loop, via parallel jobs, caching, and running only what changed; slow pipelines get bypassed, and a bypassed pipeline protects nothing. **Reliable** — flaky tests are poison: once people expect red, they ignore red; I quarantine and fix flaky tests aggressively. **Reproducible** — pinned job images and dependency versions, no 'latest'. **Build-once** — one immutable, SHA-tagged artifact promoted through environments, never rebuilt per environment. **Observable** — clear failure messages, test reports surfaced in the MR. **Safe** — least-privilege credentials, protected variables, human gates where risk warrants. And the pipeline config itself is code-reviewed like everything else."

**Q13. What CI/CD metrics would you track? (DORA — name-drop this)**
> "The four DORA metrics: **deployment frequency** (how often we ship), **lead time for changes** (commit → production), **change failure rate** (what fraction of deploys cause incidents), and **MTTR** (how fast we recover). They balance each other — speed metrics alone reward recklessness; stability metrics alone reward paralysis; elite teams are good at *both*, and the research shows they're correlated, not traded off. Operationally I'd also watch pipeline duration and pipeline success rate as team-health metrics."

---

## PART 2: Troubleshooting Questions & Answers

**T1. "A pipeline job is stuck in 'pending' forever. What do you check?"**
> "Pending means no runner has picked the job up. Checklist: **(1)** Are runners online? (Settings → CI/CD → Runners — green?) **(2)** Tag mismatch — the classic: the job specifies `tags: [docker]` and no online runner carries that tag; GitLab waits forever for a runner that doesn't exist. **(3)** Runner capacity — all runners busy at their concurrency limit; jobs queue. **(4)** Protected/shared settings — a protected-branch job may only run on protected runners. Fix ranges from starting/registering a runner, correcting tags, to autoscaling runner capacity — and the permanent fix for recurring queueing is Kubernetes-executor runners that scale with demand."

**T2. "The job fails with 'permission denied' pushing a Docker image / accessing AWS. Approach?"**
> "Auth chain, layer by layer. For registry pushes: is the job logging into the registry (`docker login` with `$CI_REGISTRY_USER/$CI_JOB_TOKEN` for GitLab's registry), and does the token have push rights to *this* project? For AWS: are the credentials present in this context — remember **protected variables don't exist on unprotected branches**, the single most common 'works on main, fails on my branch' cause — and does the IAM role actually allow the action? Debug method: echo which identity the job has (`aws sts get-caller-identity`), then test the specific action. And note the fix direction: broaden the *role* thoughtfully, don't paste stronger keys into variables."

**T3. "Tests pass locally but fail in CI. Why? (extremely common question)"**
> "Environment differences, almost always. Suspects in order: **(1) Version drift** — local Python/Node/library versions differ from the job image; fix by pinning the job image and dependencies to match. **(2) Missing service** — locally a database is running; in CI it isn't. GitLab's `services:` keyword spins up sidecar containers (e.g., `postgres:16`) alongside the job. **(3) Hidden state** — the test depends on files or env vars that exist on the laptop; CI's clean environment exposes the hidden dependency (CI is *right*, the test was lying). **(4) Timing/resources** — CI runners are slower or parallel tests race; that's a flaky test to fix, not re-run forever. Diagnostic trick: reproduce locally by running the same container image the job uses — `docker run -it <job image>` — which almost always reveals the difference."

**T4. "The pipeline is green but production is broken after deploy. What happened, what do you do?"**
> "Immediate: **roll back first, diagnose second** — redeploy the previous image tag (or `helm rollback`, or revert the merge and let the pipeline redeploy); restore service, then investigate. Then the real question: *why did green not mean working?* Usual gaps: tests don't cover the failing path; environment config differs between staging and prod (the code was fine, the prod config broke it — config should be reviewed and diffed like code); a dependency only present in prod (data shape, scale, an integration); or the deploy succeeded but health checks were too shallow to notice the app was up-but-broken. Corrective actions: add the missing test, add post-deploy **smoke tests to the pipeline itself** so deploy jobs fail loudly, and improve health/readiness checks. This becomes an RCA with the pipeline as the system under review."

**T5. "A previously-working pipeline started failing with no code changes. Causes?"**
> "'No code changes' never means 'no changes.' Suspects: **(1) Unpinned dependencies** — the job pulls `latest` of an image or package and upstream released something breaking; the fix and the lesson is pinning. **(2) Expired credentials/tokens** — deploy tokens, cloud keys, certificates age out; distinctive symptom: everything fails at the same auth step. **(3) External service change** — registry, package index, or API the pipeline depends on changed or is down. **(4) Runner environment drift** — someone updated the runner host or a shared runner image changed. **(5) Data/state growth** — caches or artifacts hit size/time limits. Method: read the first failing line, diff this run's environment against the last green run — GitLab keeps both logs — and the answer is usually visible in the delta."

**T6. "Docker build inside a CI job fails or behaves oddly — what's Docker-in-Docker?"**
> "Building images in CI needs a Docker daemon, and the job itself runs *in* a container. Options: **Docker-in-Docker (dind)** — a `docker:dind` service container provides a daemon; needs privileged mode, which is a security consideration on shared runners. **Socket mounting** — expose the host's Docker socket to jobs; simpler, but jobs can then control the host daemon — a real security tradeoff to mention. **Daemonless builders** — kaniko or buildah build images without a Docker daemon at all; the security-preferred answer on Kubernetes runners. Knowing the tradeoff triangle here is a strong senior signal for a platform role."

**T7. "How do you debug a failing job efficiently?"**
> "Read the log from the *first* error, not the last — later errors are usually cascade noise. Reproduce locally in the job's exact image. Add temporary debug output or use GitLab's job log with timestamps to see where time/failures occur. For gnarly cases, run the job with a debug/interactive session if allowed. And once solved, the senior move: make the failure mode *legible* — better error messages, a pipeline lint stage, or a doc — so the next person doesn't spend the same hour."

---

## PART 3: Scenario / Design Questions

**S1. "Design the CI/CD setup for our Domino platform team." (the job-specific one — study this)**
> "Three pipelines for three artifact types:
> **(1) Infrastructure (Terraform):** MR → fmt/validate/tflint/tfsec → plan posted to MR → approval → apply the saved plan; prod behind a manual gate mapped to a ServiceNow change; nightly drift-detection plans.
> **(2) Container environments (the Docker images data scientists use):** changes to environment Dockerfiles → build → vulnerability scan (critical in pharma) → push to registry with immutable tags → automated smoke test that actually starts a workspace/job from the new image in a test project before it's published to users. This prevents the classic 'environment edit broke everyone's workspaces' incident.
> **(3) Platform configuration/automation scripts:** linted, tested, reviewed like any code.
> Runners: self-hosted on our EKS (Kubernetes executor) so they scale and can reach private endpoints; OIDC to AWS so no stored keys. Everything on protected branches with required approvals — and the MR + pipeline evidence attaches to change records, making compliance a byproduct of the workflow instead of extra paperwork."

**S2. "We deploy manually today, twice a month, and it's painful. How would you introduce CI/CD?"**
> "Incrementally, earning trust at each step — never a big-bang replacement. **Step 1: CI only** — automate build and test on every push; no deployment changes yet; low risk, immediate value, builds the habit of green-means-good. **Step 2: automate the artifact** — pipeline produces the versioned, scanned deployable; humans still deploy it, but everyone deploys the *same* artifact the same way. **Step 3: scripted deploy to a non-prod environment** triggered by the pipeline — prove the automation where mistakes are cheap. **Step 4: production as a manual pipeline job** — one button, full logs, wrapped in the existing change process. What changed culturally: releases went from rare scary events to boring frequent ones — and 'boring' is the goal. I'd measure the DORA metrics before and after to make the improvement visible."

**S3. "How would you make deployments safe enough that a mid-week production deploy is boring?"**
> "Layered defenses: build-once immutable artifacts; staging that genuinely mirrors prod (same images, config templated from the same source); automated post-deploy smoke tests in the pipeline that fail the deploy job loudly; progressive delivery — rolling or canary with automatic metric checks (error rate, latency) before full rollout; instant rollback path rehearsed and one command away (previous image tag / helm rollback / git revert); feature flags so risky features ship dark and release is decoupled from deploy; and alerts watching the deploy window. Each layer converts a class of surprise into a non-event — and the honest point: you don't get boring deploys from courage, you get them from rollback speed."

**S4. "GitLab CI vs Jenkins — compare them."**
> "**Jenkins**: self-hosted automation server, enormously flexible via plugins, pipelines in Groovy (Jenkinsfile); the cost is operating Jenkins itself — plugin dependency hell, upgrades, security patching are real ongoing work; still everywhere in enterprises. **GitLab CI**: built into the same platform as the code — no integration seams, pipeline config in YAML in the repo, runners are the only infrastructure you manage, MR-integrated results, built-in registry and security scanning. My take: for a team already on GitLab, GitLab CI wins on total cost of ownership and integration; Jenkins earns its keep where deep customization or existing investment dominates. Also worth naming: GitHub Actions is the same philosophy as GitLab CI on the GitHub side — concepts transfer almost one-to-one."

**S5. "A developer pushes directly to main and it deployed a bug. How do you prevent this?"**
> "Controls, in order: **protect the main branch** — direct pushes blocked, merge requests mandatory; **required approvals** (at least one reviewer, code owners for sensitive paths); **pipeline must pass** before merge is allowed (merge checks); **production deploy behind a manual gate** with environment protection rules limiting who can trigger it. And the cultural half: make the compliant path the *fast* path — if the process is quick, nobody's tempted to bypass it. This maps one-to-one onto change control: the MR is the change record's technical twin."

**S6. "How do you handle database schema changes in CI/CD?" (advanced — having any answer impresses)**
> "The hard problem, because databases have state you can't rebuild from Git. Principles: schema changes as versioned, ordered **migrations** in the repo (Flyway/Liquibase/framework migrations), applied by the pipeline, never by hand. Make migrations **backward-compatible** with the currently-running app version — expand-then-contract: add the new column first (old code ignores it), deploy code using it, remove the old column in a later release — because during rolling deploys, old and new code run simultaneously against one schema. Never edit an applied migration; add a new one. And rollback for schema is *forward* — a new migration undoing the change — since restoring backups loses data written since."

---

## PART 4: Rapid-Fire One-Liners (skim before the interview)

- **CI** = auto build+test every push; **Delivery** = prod is one approved click; **Deployment** = no click, fully automatic
- **Hierarchy:** pipeline → stages (sequential) → jobs (parallel within a stage) → runners execute
- **`.gitlab-ci.yml`** in repo root defines everything — pipeline-as-code
- **Executors:** shell (dirty), docker (standard), kubernetes (scalable) — self-hosted runners for private networks
- **Artifacts** = outputs forward within a pipeline; **cache** = speed-up across pipelines
- **Secrets:** masked + protected CI variables; best = OIDC to cloud (no stored keys); never in YAML
- **`when: manual`** = approval gate; **environments** = deploy history per target
- **Build once, promote everywhere** — same SHA-tagged image through all environments
- **Deploy strategies:** recreate (downtime) / rolling (default) / blue-green (instant switch-back) / canary (metric-gated %) — feature flags decouple deploy from release
- **Shift left** = security/quality scans on every MR (SAST, dependency, container, secret detection)
- **DORA 4:** deployment frequency, lead time, change failure rate, MTTR
- **Stuck pending job** = runner offline / tag mismatch / capacity
- **"Works locally, fails in CI"** = version drift, missing service, hidden state, or flaky timing
- **Green pipeline, broken prod** = rollback first; then close the gap (missing test, config diff, shallow health checks, add smoke tests post-deploy)
- **dind vs socket-mount vs kaniko** = the three ways to build images in CI; kaniko is the security-clean one

---

## PART 5: Honest Positioning If Asked About Your CI/CD Experience

Adapt to whatever you HAVE done (fill this in truthfully tonight):

> "My hands-on depth is strongest on the infrastructure side of CI/CD — the runners, the Kubernetes and Docker layer pipelines execute on, the AWS permissions they need — and I understand the pipeline design principles well: build-once artifacts, staged promotion, manual gates for regulated environments, shift-left scanning. Where I haven't personally built something — say, a full canary rollout system — I understand the mechanics and tradeoffs and can implement from that understanding. In this role my first automation targets would be the environment-image pipeline with vulnerability scanning and pre-publish smoke tests, and Terraform plan/apply pipelines tied into change records — both directly reduce the incident classes a Domino platform team sees most."

Naming *specific first projects* turns "I'm learning" into "here's my plan" — that's the difference between a gap and a roadmap.

---

## FINAL RECALL STORY (read 3 times — it chains every concept)

> *A developer opens a merge request. The push triggers a **pipeline** defined in **`.gitlab-ci.yml`**: the **build stage** builds a Docker image tagged with the commit SHA (**build once**) via **kaniko** on our **Kubernetes-executor self-hosted runners**; the **test stage** runs unit tests and lint **in parallel**, with a Postgres **service** container because tests need a DB (which is why they passed locally but failed in CI last month until we added it); the **scan stage** runs SAST and container scanning (**shift left**) and once caught a critical CVE before it ever reached staging. Artifacts carry the test report into the MR view. The branch is **protected** — merge needs an approval and a green pipeline. On merge, staging deploys automatically and **post-deploy smoke tests** run — added after the day a green pipeline still broke prod because health checks were shallow; we **rolled back first** by redeploying the previous image tag, diagnosed second. Production is `when: manual`, its **environment** protected, credentials via **OIDC** (no stored keys) and **protected variables** — which explains that time a feature-branch job failed with permission-denied: protected variables don't exist there. The prod deploy is **canary**: 5% traffic, error-rate check, then full rollout. And the quarter's DORA review showed deploy frequency up and change failure rate down — because those move together, not against each other.*

Retell that story and you can field 90% of any CI/CD interview.
