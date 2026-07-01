# Terraform Interview Guide — From Zero to Confident Answers
### Learn the concepts AND the answers in one read-through

---

## PART 0: The Mental Model (read this first — everything else depends on it)

**What problem does Terraform solve?**
Before Infrastructure as Code, you built cloud infrastructure by clicking in the AWS console. Problems: no record of what you did, no way to reproduce it, no review before risky actions, and environments drift apart ("staging worked, prod didn't — turns out someone hand-edited a security group in March").

**Terraform's answer:** you *describe* your desired infrastructure in text files, and Terraform makes reality match the description.

**The three-piece mental model — memorize this:**

> **Terraform = Desired State (your .tf files) + Actual State (real AWS) + a State File (Terraform's memory of what it created).**
> Every `terraform apply` is Terraform answering one question: *"What's the difference between what you asked for and what I know exists — and what must I create, change, or destroy to close that gap?"*

**Declarative vs imperative (a guaranteed question, so get the intuition now):**
- *Imperative* = a recipe: "run these commands in this order" (bash scripts).
- *Declarative* = a destination: "I want 3 servers and a bucket" — the tool figures out the steps. If 2 servers already exist, Terraform creates only 1 more. A script would blindly create 3 more.

**The core workflow — 4 commands (memorize the sequence):**
> **`init` → `plan` → `apply` → `destroy`**
- `terraform init` — set up the working directory: downloads providers (plugins), configures the state backend, downloads modules. Run once per directory (and again when providers/backends change).
- `terraform plan` — the dry run: compares desired vs state vs reality and prints what it *would* do (`+` create, `-` destroy, `~` change). Nothing is touched.
- `terraform apply` — executes the plan (asks for confirmation).
- `terraform destroy` — tears everything managed by this configuration down.

---

## PART 1: Core Concept Questions & Answers

**Q1. What is Terraform and why use it over manual provisioning or CloudFormation?**
> "Terraform is an Infrastructure-as-Code tool by HashiCorp: you declare desired infrastructure in HCL files, and Terraform creates, updates, or deletes real resources to match. Over manual clicking: it's reviewable (Git pull requests for infrastructure changes), repeatable (spin up an identical staging environment from the same code), and auditable — which matters in a GxP context where infrastructure changes need change control just like application changes. Over CloudFormation: Terraform is cloud-agnostic — the same tool and language manage AWS, Azure, Kubernetes, Datadog, even Domino resources via providers — and its plan output and module ecosystem are widely considered stronger. CloudFormation's advantage is being AWS-native with no state file to manage yourself."

**Q2. What is a provider?**
> "A provider is the plugin that teaches Terraform how to talk to a specific platform's API — the AWS provider translates my HCL into AWS API calls; there are providers for Azure, Kubernetes, Helm, GitHub, and hundreds more. You declare required providers with version constraints, and `terraform init` downloads them. Pinning provider versions matters: a major provider upgrade can change resource behavior, so uncontrolled provider updates are a real source of surprise plans."

**Q3. What is a resource? Show me basic HCL.**
> "A resource is one infrastructure object Terraform manages — an EC2 instance, an S3 bucket, a security group. The syntax:
> ```hcl
> resource "aws_instance" "web" {
>   ami           = "ami-0abc123"
>   instance_type = "t3.medium"
>   tags = { Name = "web-server" }
> }
> ```
> `aws_instance` is the resource *type* (defined by the provider), `web` is my local *name* for referencing it elsewhere as `aws_instance.web.id`. That referencing is how Terraform builds its dependency graph."

**Q4. What is the state file? (THE most important Terraform question — expect it)**
> "The state file (`terraform.tfstate`) is Terraform's memory: a JSON record mapping my configuration to the real resource IDs it created — 'resource aws_instance.web is instance i-0abc123 with these attributes.' It's essential because the desired configuration alone isn't enough: Terraform needs to know what it *already* made to compute the diff. Without state, Terraform would look at my config, see nothing in its memory, and try to create everything again — duplicating infrastructure.
> Three critical properties: **(1) it can contain secrets in plain text** (database passwords, etc.), so it must be stored encrypted and access-controlled; **(2) it must be shared** for team work — hence remote backends; **(3) losing it doesn't delete infrastructure, but Terraform loses track of everything it managed**, which is a painful recovery (Q-T4)."

**Q5. What is a remote backend and why do teams need one?**
> "By default state is a local file — fine solo, broken for teams: two engineers with separate local states will fight each other, and laptop state can be lost. A remote backend stores state centrally. The classic AWS setup: **S3 bucket for the state file** (versioned + encrypted) **plus a DynamoDB table for state locking**. Locking means when I run `apply`, Terraform writes a lock entry; a colleague running apply simultaneously is blocked until I finish — preventing two applies from corrupting state or racing to change the same resources. This S3+DynamoDB pattern is nearly universal on AWS." *(Note: newer Terraform versions can do S3-native locking, but S3+DynamoDB is the answer interviewers expect.)*

**Q6. Variables, outputs, and locals — what are they?**
> "**Variables** are inputs — parameterize the config so the same code deploys dev and prod with different values (`var.instance_type`), supplied via `.tfvars` files, CLI flags, or `TF_VAR_` environment variables. **Outputs** are the return values — expose useful results like a load balancer DNS name, both for humans and for *other* Terraform configurations to consume. **Locals** are internal named expressions — computed once, reused, never supplied from outside. Shorthand: variables = function parameters, outputs = return values, locals = local variables inside the function."

**Q7. What is a module?**
> "A module is a reusable package of Terraform code — the function of the Terraform world. You write a 'vpc' module once (subnets, route tables, NAT), then instantiate it for dev, staging, and prod with different variables. Benefits: DRY code, enforced standards (the module bakes in encryption and tagging so teams can't forget), and reviewable versioned units — modules are typically sourced from Git tags or a registry with pinned versions so environments don't silently change. Any directory of .tf files is technically a module; the one you run commands in is the 'root module.'"

**Q8. How does Terraform know what order to create resources in? (the DAG question)**
> "Terraform builds a **dependency graph** — a DAG, directed acyclic graph — from references between resources. If my instance references `aws_subnet.private.id`, Terraform knows: subnet first, then instance; and on destroy, the reverse. Independent resources are created **in parallel**, which is why Terraform is fast. When there's a real dependency Terraform can't see from references — rare — `depends_on` declares it explicitly. Nice interview flex: `terraform graph` outputs the DAG, and this is also why circular references are an error — 'acyclic' means no loops allowed."

**Q9. What happens on `terraform plan`, in detail?**
> "Three-way comparison. **(1) Refresh:** Terraform queries the real provider APIs to update its picture of actual resource attributes. **(2) Diff:** compares desired config vs (refreshed) state. **(3) Output:** the execution plan — create (+), destroy (–), update in-place (~), or **replace (–/+)** for attributes that can't be changed in place (changing an EC2 AMI forces destroy-and-recreate). Reading plans carefully is *the* core operational skill: an unexpected 'destroy' in a plan is your last checkpoint before an outage. In CI/CD, a saved plan file (`terraform plan -out=plan.tfplan`) is what gets reviewed and then applied, guaranteeing you apply exactly what was reviewed."

**Q10. What is drift and how do you handle it?**
> "Drift is when reality no longer matches state/config because something changed *outside* Terraform — someone hand-edited a security group in the console, or an emergency fix bypassed IaC. Terraform detects it during plan's refresh: the next plan will propose reverting the manual change back to the declared config. Handling it is a judgment call: if the manual change was wrong, apply and revert it; if it was right, **update the code to match** and then plan should show no changes. Preventing drift is process, not tooling: restrict console write access, make Terraform-the-only-path easy, and run scheduled `terraform plan` in CI as a drift detector that alerts on any diff — which is exactly the 'automation and monitoring improvement' mindset the platform role wants."

**Q11. `count` vs `for_each`?**
> "Both create multiple instances of a resource. `count` gives numbered copies (`resource[0]`, `resource[1]`) — fine for identical clones, but fragile: removing an item from the middle of the list shifts every index after it, and Terraform will destroy/recreate the shifted resources. `for_each` iterates over a map/set and keys instances by name (`resource[\"alice\"]`) — removing one item touches only that item. Rule of thumb: `for_each` for anything with identity, `count` for 'give me N identical things' or conditional creation (`count = var.enabled ? 1 : 0`)."

**Q12. What are `data` sources?**
> "A `data` block *reads* existing infrastructure that Terraform doesn't manage — look up the newest AMI ID, an existing VPC created by another team, or another configuration's outputs via `terraform_remote_state`. Data = read-only; resource = managed. It's how configurations reference things without owning them."

**Q13. What are workspaces? And how do most teams actually separate environments?**
> "Terraform workspaces let one configuration hold multiple independent state files — `terraform workspace new staging` — same code, separate states. Honest senior answer: CLI workspaces are fine for lightweight variants, but most teams separate environments with **separate directories or separate state backends per environment** (dev/staging/prod each with own state, own variables, often own AWS account), because it gives stronger isolation — a mistake in dev physically cannot touch the prod state file — and clearer code review. I'd mention both and say directory-per-environment is what I'd choose for production infrastructure."

**Q14. How do you manage secrets in Terraform?**
> "Layered answer: **never hardcode secrets in .tf files** — they end up in Git. Options, in increasing preference: sensitive variables (`sensitive = true` — hides from CLI output but **still lands in the state file in plain text**, an important honest caveat), pulling secrets at apply time from **AWS Secrets Manager/SSM Parameter Store via data sources**, or best, designing so Terraform never touches the secret value at all — e.g., create the database and let the secret be generated and stored directly in Secrets Manager, with applications reading it at runtime. And because state can contain secrets regardless, the state backend itself must be encrypted and access-restricted."

**Q15. What is `terraform import`?**
> "Brings *existing*, manually-created infrastructure under Terraform management: you write the resource block, then `terraform import aws_instance.web i-0abc123` records the mapping in state — then iterate until `plan` shows no changes, meaning your code accurately describes the real thing. This is the migration path for a company adopting IaC with a pile of hand-built infrastructure — which is realistically how I'd expect to encounter it."

**Q16. Explain the lifecycle meta-arguments.**
> "`lifecycle` blocks tune Terraform's behavior per resource: **`prevent_destroy = true`** makes any plan that would destroy the resource *fail* — insurance on databases and state-critical resources. **`create_before_destroy`** flips replacement order so the new resource exists before the old one dies — avoiding downtime on replacements. **`ignore_changes`** tells Terraform to stop reconciling specific attributes — used when an external system legitimately manages that attribute, like an autoscaler changing instance counts."

**Q17. What are provisioners, and why are they discouraged?**
> "Provisioners run scripts on resources at create/destroy time (`remote-exec`, `local-exec`). HashiCorp themselves call them a last resort: they break the declarative model — Terraform can't 'diff' a script's effects — they fail non-deterministically, and there's almost always a better tool: cloud-init/user_data for boot config, Ansible/Packer for configuration management, or baking images. Saying 'I avoid provisioners and here's why' is itself a senior signal."

---

## PART 2: Troubleshooting Questions & Answers

**T1. "`Error acquiring the state lock` — what happened and what do you do?"**
> "Someone or something is (or was) mid-operation on the same state. First, *find out who* — the lock info shows an ID, who, and when. If a colleague's apply is genuinely running: wait. If it's stale — a CI job was killed, someone's laptop died mid-apply, leaving the lock behind — then `terraform force-unlock <LOCK_ID>` releases it. The critical discipline: **confirm nothing is actually running before force-unlocking** — unlocking under a live apply is how state gets corrupted. If stale locks recur from CI, the RCA is the pipeline not handling cancellation cleanly."

**T2. "Plan shows changes you didn't make / wants to destroy something unexpectedly. What now?"**
> "Stop — never apply a plan you don't understand. Three usual suspects: **(1) Drift** — someone changed it manually; the plan is Terraform proposing to revert reality to code (decide which is correct, per Q10). **(2) Provider version change** — a provider upgrade altered defaults or attribute handling; check whether versions are pinned and what changed. **(3) A forced replacement** — I changed an attribute that can't update in place, and the plan shows destroy-and-recreate; the plan output marks which attribute *forces replacement*. If replacement of a stateful resource (database!) is unacceptable, options include `create_before_destroy`, rethinking the change, or in some cases `terraform state mv`/targeted strategies. The meta-answer interviewers want: I read plans line by line and treat an unexpected destroy as a stop-the-line event."

**T3. "Apply failed halfway. Is your infrastructure broken? What do you do?"**
> "Not broken — *partially applied*, and Terraform state accurately records whatever completed (Terraform updates state per-resource as it goes). The approach: read the actual error — commonly an IAM permission missing, a quota/limit hit, a naming conflict, or an eventual-consistency timing issue. Fix the cause, run `plan` again — it shows only the remaining work — and `apply` to converge. Terraform is idempotent by design: re-running is the normal recovery, not a special procedure. If a resource ended half-created and confused (rare), `terraform taint` (or `-replace=` in modern versions) marks it for recreation on the next apply."

**T4. "The state file was deleted/corrupted. Walk me through recovery."**
> "First: infrastructure is still running — state loss doesn't touch real resources; Terraform has amnesia, not the cloud. Recovery ladder: **(1) Backend versioning** — if state was in S3 with versioning enabled (it must be), restore the previous version — minutes, done; this is exactly why S3 versioning on state buckets is non-negotiable. **(2) Local backup** — Terraform keeps `terraform.tfstate.backup`; CI logs may also hold recent state. **(3) Worst case: rebuild state by importing** every resource (`terraform import` one by one, or tooling to assist) until plan is clean — tedious but fully recoverable. And the RCA/prevention half of the answer: versioned + encrypted backend, restricted deletion permissions, and never hand-editing state — use `terraform state` subcommands."

**T5. "Two engineers ran apply at the same time and things look inconsistent."**
> "This means state locking wasn't in place or was bypassed — the root problem to fix. Immediate cleanup: run `terraform plan` and audit the diff against reality; use `terraform state list` and targeted refresh to rebuild an accurate picture; reconcile any duplicated or orphaned resources (import or delete). Permanent fix: remote backend with locking (S3+DynamoDB), and better — remove local applies from humans entirely: **all applies flow through a CI/CD pipeline**, serialized, from reviewed plans. Then concurrent-apply becomes structurally impossible rather than procedurally discouraged."

**T6. "`terraform init` fails — what are the usual causes?"**
> "Init does three things, so failures map to three areas: **provider download** (no network/proxy to the registry, or the version constraint is impossible — e.g., two modules demanding incompatible provider versions), **backend initialization** (S3 bucket doesn't exist, no permissions, wrong region, or backend config changed — needing `init -migrate-state` or `-reconfigure`), and **module fetch** (bad Git ref/tag, no access to the repo). The error message says which; in a corporate network, proxy/firewall to registry.terraform.io is a classic."

**T7. "A resource keeps showing changes on every single plan, even after applying." (perpetual diff)**
> "A perpetual diff — usually one of: the provider API normalizes a value differently than written (lowercase vs uppercase, JSON field ordering in IAM policies), an external system rewrites the attribute after apply (an autoscaler, or AWS adding default tags), or a provider bug. Fixes in order: write the value in the API's canonical form; use `lifecycle { ignore_changes = [...] }` for attributes legitimately owned by another system; check the provider's GitHub issues — perpetual-diff bugs are common and often already documented with the workaround."

---

## PART 3: Scenario / Design Questions

**S1. "Design the Terraform setup for our organization managing this Domino/EKS platform across dev, staging, prod."**
> "Structure: a **modules repo** (versioned, tagged) holding reusable modules — vpc, eks-cluster, eks-nodegroup, s3, efs, iam — and an **environments layout** with a directory per environment (dev/staging/prod), each with its own remote state (S3+DynamoDB, versioned, encrypted, ideally per-environment AWS accounts), composing the same modules with different variables. That makes staging a true replica of prod — the same module versions — which is exactly what makes upgrade testing meaningful. Workflow: nobody applies from a laptop; pull request → CI runs fmt/validate/plan → the plan output is posted for review → after approval and merge, the pipeline applies the reviewed plan. Prod applies map to ServiceNow change requests — the PR + plan output *is* the test evidence attached to the CHG record, which ties IaC directly into the GxP change-control story. Guardrails: pinned provider and module versions, `prevent_destroy` on state-critical resources, scheduled drift-detection plans."

**S2. "How would you introduce Terraform into an environment where everything was built by hand?"**
> "Incrementally — never big-bang. **(1)** Establish the foundation first: remote state backend, repo structure, CI pipeline, standards. **(2)** New infrastructure goes through Terraform from day one — stop the bleeding. **(3)** Import existing resources in priority order — start with things that change often (that's where IaC pays off fastest) and low-risk resources to build confidence; `terraform import` each, iterate until plan is clean. **(4)** Lock the console down gradually as coverage grows, so drift can't regenerate. **(5)** Accept that some legacy corners may never be worth importing — that's a pragmatic judgment, not a failure. Throughout: every import is itself reviewable in Git, so the migration has an audit trail."

**S3. "How do you safely make a risky change — say, modifying the EKS node group Terraform manages — without downtime?"**
> "First, `plan` tells me the blast radius — is it update-in-place or a forced replacement? For a replacement of capacity-bearing resources: `create_before_destroy` so new nodes join before old ones die, combined with Kubernetes-level draining (cordon/drain old nodes so pods reschedule gracefully). Test the exact change in staging built from the same modules. Apply in a maintenance window via the pipeline, watch the cluster during, and have the rollback ready — which in Git-based IaC is beautifully simple: **revert the commit and apply again**; the previous state of the code *is* the rollback plan. That Git-revert-as-rollback point connects IaC to everything change management wants."

**S4. "Terraform vs Ansible vs Helm — when would you use which?"**
> "Terraform: *provisioning* infrastructure — cloud resources, the EKS cluster itself, IAM, networking. Ansible: *configuration management* — installing and configuring software on existing machines; mostly displaced in container land but alive for VMs and appliances. Helm: *deploying applications onto Kubernetes* — templated Kubernetes manifests. They compose: Terraform builds the EKS cluster, Helm (possibly invoked via Terraform's Helm provider) installs applications onto it. The trap answer to avoid is treating them as competitors — they're layers."

**S5. "What does a good Terraform CI/CD pipeline look like?"**
> "On PR: `terraform fmt -check` (formatting), `terraform validate` (syntax/internal consistency), a linter like tflint, security scanning (tfsec/checkov — catches public S3 buckets and open security groups before humans review), then `terraform plan -out=plan.tfplan` with the plan posted to the PR as the reviewable artifact. On merge/approval: apply *that saved plan file* — guaranteeing what was reviewed is exactly what runs — serialized per environment, with prod gated behind manual approval (and in our world, a change record). Plus a scheduled nightly plan against each environment as drift detection. This pipeline is the honest answer to 'how do you prevent' half the failure modes in Part 2."

**S6. "How do you handle a resource Terraform manages that someone needs changed RIGHT NOW during an incident?"**
> "Reality-honest answer: in a true emergency, the manual fix may happen first — service restoration wins. The discipline is what happens *next*: the manual change is now drift, so before the incident is closed, the code gets updated to match reality (or the change reverted properly through the pipeline), an emergency change record documents it, and the next scheduled drift-detection plan confirms clean. What you must never allow is the silent version — a manual prod change nobody backports, which detonates weeks later when an unrelated apply reverts it and nobody knows why the incident from March just came back."

---

## PART 4: Rapid-Fire One-Liners (skim right before the interview)

- **Workflow:** `init` (setup) → `plan` (preview) → `apply` (execute) → `destroy` (teardown)
- **State file** = Terraform's memory mapping config → real resource IDs; may contain secrets; encrypt it, version it, lock it
- **Remote backend on AWS** = S3 (versioned, encrypted) + DynamoDB (locking)
- **Plan symbols:** `+` create, `-` destroy, `~` update in-place, `-/+` replace (destroy & recreate)
- **HCL** = HashiCorp Configuration Language (the .tf syntax)
- **DAG** = dependency graph from references; independent resources apply in parallel; no cycles allowed
- **Drift** = reality changed outside Terraform; detected at plan-refresh; fix code or revert reality
- **`for_each` > `count`** for anything with identity (index-shift problem)
- **`terraform import`** = adopt existing infra into state
- **`taint` / `-replace=`** = force recreation of one resource next apply
- **`terraform state mv/rm/list`** = surgical state operations (never hand-edit the JSON)
- **`force-unlock`** = release a stale lock — only after confirming nothing is running
- **Lifecycle:** `prevent_destroy` (safety), `create_before_destroy` (zero-downtime replace), `ignore_changes` (shared ownership)
- **Provisioners** = last resort; prefer user_data/Packer/Ansible/Helm
- **Rollback in IaC** = `git revert` + apply — the code history is the rollback plan
- **Secrets:** never in .tf files; prefer Secrets Manager/SSM via data sources; remember `sensitive=true` still stores plaintext in state

---

## PART 5: If Asked "What's Your Terraform Experience?" (your honest positioning)

> "I'm earlier in my Terraform journey than in Kubernetes or AWS — I've been actively studying it, working through the core workflow, state management, and modules, and I understand it deeply at the conceptual level: the declarative model, the state file as Terraform's memory, the dependency DAG, remote backends with locking. What accelerates me is that I already know intimately what Terraform is *describing* — I've built and operated the VPCs, EKS clusters, IAM roles, and storage it provisions. Learning HCL syntax for infrastructure I already understand is a matter of weeks; I'd expect to be contributing reviewed Terraform changes within my first month."

That answer is honest, specific, and turns the gap into a timeline — which is what a hiring manager actually needs to hear.

---

## FINAL RECALL STORY (read 3 times — it chains every concept)

> *We keep our infrastructure as code: modules for vpc and eks, a directory per environment, state in a versioned encrypted S3 bucket with DynamoDB locking. I open a PR changing the node group instance type; CI runs fmt, validate, tfsec, and posts the **plan** — I read it carefully and spot a `-/+`: this attribute **forces replacement**. Since these are capacity-bearing nodes, I add **create_before_destroy** and plan Kubernetes draining. Staging (same modules, same versions) tests clean, the plan output goes onto the ServiceNow change record as evidence, and the pipeline **applies the exact saved plan** in the window. A week later the nightly drift-detection plan flags a security group someone hand-edited during an incident — we backport the fix into code so reality and Git agree again. And when a CI job died mid-apply and left a **stale lock**, I checked nothing was running, **force-unlocked**, re-ran plan — it showed only the remaining work, because a failed apply just means partially applied, and re-running converges.*

If you can retell that story, you can handle 90% of what they'll ask on Terraform.
