# Terraform Interview Prep: Modules, Reusable/Compliant IaC & Troubleshooting

> A zero-to-interview-ready guide. Read Part 1 fully even if you think you know the basics — every later explanation builds on these exact mental pictures.

---

## How to use this guide

Each question below has four parts:
- **Why they ask this** — what the interviewer is actually testing
- **Simple explanation** — the plain-English mental model, assuming no prior knowledge
- **Deeper detail** — the technical meat, so you can handle follow-ups
- **What to say** — a tight, spoken-out-loud style answer you can adapt

Troubleshooting scenarios additionally give you a **step-by-step approach**, the **exact commands**, and the **mental model** to reuse on any new problem you've never seen before — because interviewers often invent a scenario on the spot to see if you *think* correctly, not whether you memorized an answer.

---

## PART 1: Foundations (skip only if these are 100% second nature)

### 1.1 What problem does Terraform solve?

Before Terraform, infrastructure (servers, networks, databases, load balancers) was created by:
- Clicking around a cloud console (AWS/Azure/GCP UI), or
- Running one-off scripts (bash/Python) that called cloud APIs

Both approaches don't scale: you can't easily review changes, replicate an environment, track what exists, or roll back safely. Nobody has a reliable record of "what infrastructure actually exists and why."

**Terraform's idea:** describe your infrastructure as *code* (text files), and let Terraform figure out what API calls are needed to make reality match that code. This is called **Infrastructure as Code (IaC)**.

The core promise is **declarative** management: you say *what you want* (e.g., "I want 3 web servers and 1 load balancer"), not *how to get there* (you don't write the steps). Terraform computes the "how."

### 1.2 The core building blocks (you must be fluent in these words)

| Term | Plain-English meaning |
|---|---|
| **Provider** | A plugin that lets Terraform talk to a specific system's API (AWS, Azure, GCP, Kubernetes, GitHub, Datadog, etc.). Without a provider, Terraform doesn't know how to create anything. |
| **Resource** | One real-world object you want Terraform to manage — an EC2 instance, an S3 bucket, a database, a DNS record. Each resource block is a declaration: "this thing should exist with these settings." |
| **Data source** | A read-only lookup — "go find this existing thing and give me its details" (e.g., look up the latest AMI ID). It never creates or changes anything. |
| **Variable (input)** | A parameter you pass into your configuration so it isn't hardcoded — like a function argument. |
| **Output** | A value your configuration exposes after it runs — like a function's return value (e.g., the IP address of the server you just created). |
| **State** | Terraform's internal record (a JSON file, `terraform.tfstate`) of what it *believes* currently exists and its attributes. This is the single most important — and most misunderstood — concept in Terraform. |
| **Plan** | A dry-run: Terraform compares your code + variables against the state file (and, for safety, reality) and shows you exactly what it *would* create, change, or destroy — without doing it. |
| **Apply** | Actually executes the plan — makes the real API calls to create/update/destroy resources, and then updates the state file to match. |
| **Backend** | Where the state file is stored (locally on disk, or remotely in S3, Azure Storage, Terraform Cloud, etc.) and how it's locked during operations. |
| **HCL** | HashiCorp Configuration Language — the `.tf` file syntax. Declarative, block-based, human-readable. |

### 1.3 Why "state" is the single most important concept

Terraform is declarative, but cloud APIs are not "declarative-aware" — they just take create/update/delete calls. So Terraform needs a way to know:
- What did I create last time?
- What are its current attributes (like its ID)?
- What's different between "what I created before" and "what my code says now"?

The **state file** is that memory. Every `plan` and `apply` is fundamentally a three-way comparison:

```
Your .tf code (desired)   vs   State file (last known)   vs   Real infrastructure (actual)
```

Most real-world Terraform pain (drift, "resource already exists," locking errors, weird diffs) comes back to a mismatch somewhere in this triangle. Internalize this triangle now — you'll use it constantly in the troubleshooting section.

### 1.4 The Terraform workflow (mental sequence)

```
write/edit .tf files
      │
terraform init      → downloads providers/modules, sets up backend
      │
terraform validate  → checks syntax/internal consistency (no API calls)
      │
terraform plan      → dry-run: shows add/change/destroy, compares state vs code vs reality
      │
terraform apply     → executes the plan, calls real APIs, updates state
      │
terraform destroy   → (when needed) tears down everything Terraform manages
```

Keep this sequence in your head — nearly every interview question maps onto "which stage of this pipeline does this problem happen in, and why."

### 1.5 Idempotency (a word you should use confidently)

**Idempotency** means: running the same operation multiple times produces the same end result as running it once. Terraform `apply` is meant to be idempotent — if nothing changed in your code and nothing drifted in real infrastructure, running `apply` again should show "no changes." This is *why* Terraform is trusted for repeatable automation (e.g., in CI/CD pipelines) — you're not scared to run it twice by accident.

---

## PART 2: Terraform Modules — Concepts & Interview Q&A

### 2.1 What is a module, really?

A **module** is just a folder containing `.tf` files. That's it. There's nothing magical about the word — every Terraform configuration is technically a module (the one you run `terraform apply` from is called the **root module**). When people say "modules" in interviews, they usually mean **reusable child modules** — a folder you designed so it can be called (like a function) from multiple places with different inputs.

**Mental model: a module is a function.**

| Programming function | Terraform module |
|---|---|
| Parameters | Input variables (`variables.tf`) |
| Function body | Resource definitions (`main.tf`) |
| Return value | Output values (`outputs.tf`) |
| Calling the function | A `module "name" { source = "..." }` block |

```hcl
module "web_server" {
  source        = "./modules/ec2-instance"   # where the module code lives
  instance_type = "t3.micro"                 # this is an "argument" (input variable)
  environment   = "prod"
}
```

### 2.2 Standard module file structure

```
modules/ec2-instance/
├── main.tf         # resource definitions — the actual logic
├── variables.tf    # input variable declarations (with types, defaults, validation)
├── outputs.tf       # values exposed to whoever calls this module
├── versions.tf      # required_providers + required Terraform version
└── README.md        # usage docs (inputs, outputs, examples) — often auto-generated
```

This isn't enforced by Terraform — it's a convention. But interviewers expect you to know it, because consistency across modules is what makes a whole team able to jump into any module and immediately know where to look.

---

### Q1. What is a Terraform module and why do we use them?

**Why they ask this:** Baseline check — can you explain the "why," not just the "what."

**Simple explanation:** A module packages a piece of infrastructure logic (e.g., "a properly configured S3 bucket with encryption and logging") into a reusable unit, so any team can reuse it with different inputs instead of copy-pasting the same 50 lines of HCL every time they need an S3 bucket.

**Deeper detail:** Modules give you:
- **Reusability** — write once, use across many environments/teams/projects.
- **Encapsulation/abstraction** — consumers only need to know the inputs/outputs, not the internal resource wiring. A junior engineer can use a "VPC module" without understanding subnetting.
- **Consistency & standardization** — if every team uses the same "S3 module," every bucket automatically follows the same security/tagging/naming rules.
- **Easier testing and versioning** — you can version a module independently (like a library/package) and let consumers pin to a known-good version.
- **Reduced blast radius of mistakes** — logic is centralized and reviewed once, instead of being re-invented (and re-broken) in 20 different repos.

**What to say:** "A module is a self-contained, reusable unit of Terraform configuration — essentially a function for infrastructure. We use them so teams don't reinvent (and misconfigure) the same resources repeatedly; instead they consume a tested, standardized module and just supply inputs like environment or size."

---

### Q2. What's the difference between the root module and a child module?

**Simple explanation:** The **root module** is the directory where you actually run `terraform init/plan/apply` — the entry point. A **child module** is any module called *from* another module (root or otherwise) using a `module` block.

**Deeper detail:** Every `.tf` configuration is technically a module — Terraform doesn't have a special "root module" file type; it's defined by where you invoke Terraform CLI. Child modules can themselves call other child modules — this is called **module composition/nesting**. There's no hard limit on nesting depth, but deep nesting hurts readability and debuggability, so most style guides cap it (commonly 2-3 levels).

**What to say:** "The root module is your working directory — where you run the CLI. Anything it calls via a `module` block is a child module. Child modules can call other child modules too, but we try to keep nesting shallow for readability."

---

### Q3. How do input variables and validation work in a module?

**Simple explanation:** Input variables are how you pass configuration *into* a module, similar to function arguments. You can also add **validation rules** so bad input fails fast with a clear message, instead of silently creating something wrong or failing deep inside a cloud API call.

**Deeper detail:**
```hcl
variable "environment" {
  type        = string
  description = "Deployment environment"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment must be one of: dev, staging, prod."
  }
}
```
- `type` enforces a data shape (`string`, `number`, `bool`, `list(string)`, `map(string)`, `object({...})`). This catches type errors at `plan` time instead of deep in `apply`.
- `default` makes a variable optional.
- `sensitive = true` masks the value in CLI output and logs (doesn't encrypt it in state, though — important nuance, see troubleshooting section).
- `validation` blocks let you enforce business rules (naming conventions, allowed instance sizes, allowed regions) — this is a big part of "compliant, standardized" infrastructure, because it stops non-compliant input before any API call is made.

**What to say:** "Input variables are typed parameters into the module. I always add explicit types and validation blocks — for example restricting `environment` to an enum, or instance type to an approved list — because it turns a policy rule into a fast, local error instead of a runtime failure or, worse, a silently non-compliant resource."

---

### Q4. What are output values and why do modules need them?

**Simple explanation:** Outputs are the "return values" of a module — data the module wants to hand back to whoever called it, e.g., the ID of a VPC it just created, so a different module can plug that ID in as its own input.

**Deeper detail:** Outputs are how modules **chain together**. E.g., a `vpc` module outputs `vpc_id` and `subnet_ids`; an `ec2` module takes those as input variables. This is how large infrastructures are composed from small modules — data flows through outputs → inputs, forming a dependency graph. Outputs can also be marked `sensitive = true` to hide them from default CLI output (e.g., a generated password).

**What to say:** "Outputs expose specific values from inside a module — like the resulting resource ID or ARN — so other modules or the root config can consume them. That's the mechanism that lets you compose small modules into a full infrastructure stack."

---

### Q5. Local modules vs. remote modules — what's the difference and when do you use each?

**Simple explanation:** `source = "./modules/vpc"` is a **local module** — it lives in the same repo. `source = "git::https://..."` or `source = "app.terraform.io/org/vpc/aws"` is a **remote module** — pulled from a Git repo, the public Terraform Registry, or a private registry.

**Deeper detail:**
- Local modules are good for logic specific to one project/repo, or while you're still developing/iterating on a module.
- Remote modules (especially from a **private module registry**, e.g., Terraform Cloud's private registry, or a Git repo with tags) are how you share standardized, versioned modules *across* teams and repos — this is the backbone of "reusable, standardized infrastructure code" at an org level.
- Remote sources support **version constraints**:
```hcl
module "vpc" {
  source  = "app.terraform.io/my-org/vpc/aws"
  version = "~> 2.1.0"   # allows 2.1.x, not 2.2.0
}
```
This means teams can upgrade a shared module deliberately, on their own schedule, instead of an update breaking everyone at once.

**What to say:** "Local modules are for logic scoped to one repo. For anything meant to be shared org-wide — VPC patterns, S3 bucket standards, IAM role patterns — we publish them as versioned remote modules, usually via a private registry or tagged Git releases, so consuming teams can pin a version and upgrade intentionally."

---

### Q6. How do you version a module, and why does version pinning matter?

**Simple explanation:** Just like a software library (e.g., an npm or pip package), a module can have version numbers (usually via Git tags like `v1.2.0`, following **semantic versioning**: MAJOR.MINOR.PATCH). Consumers pin to a version range so an update to the module doesn't silently break their infrastructure.

**Deeper detail:**
- **MAJOR** version bump = breaking change (e.g., removed a variable, renamed an output, changed resource behavior in an incompatible way).
- **MINOR** = new backward-compatible feature (e.g., added an optional variable).
- **PATCH** = backward-compatible bug fix.
- Without pinning, if module authors push a breaking change, *every* consumer's next `plan` could show unexpected destroys — a serious production risk.
- Standard practice: consuming teams use `~>` (pessimistic constraint operator) to allow patch/minor upgrades automatically but require a deliberate action for major upgrades.

**What to say:** "We follow semantic versioning for modules — patch and minor releases are backward compatible, major releases can break things. Consumers pin with `~>` so they get safe updates automatically but have to explicitly opt into breaking changes. This is critical at scale — without it, one module change could silently break dozens of consuming teams."

---

### Q7. What's the difference between `count`, `for_each`, and dynamic blocks, and when would you use each inside a module?

**Simple explanation:** All three let you create multiple similar things without copy-pasting a resource block repeatedly.

- `count`: repeat a resource N times, indexed 0, 1, 2... Good for identical, unnamed copies.
- `for_each`: repeat a resource once per item in a map or set, keyed by a meaningful string. Good when items have identities (e.g., one subnet per named AZ) and you don't want reordering to accidentally destroy/recreate things.
- **dynamic block**: generate repeated *nested blocks* inside a single resource (e.g., multiple `ingress` rules inside one `aws_security_group` resource) — not multiple resources, but multiple sub-blocks within one resource.

**Deeper detail — why `for_each` is usually preferred over `count` for anything long-lived:**
With `count`, resources are tracked in state by index (`aws_instance.this[0]`, `[1]`, `[2]`). If you remove the *middle* item from your list, Terraform sees index shifts and may destroy/recreate resources that didn't actually need to change. With `for_each`, resources are tracked by a stable key (`aws_instance.this["web"]`, `["api"]`), so removing one item only affects that one resource.

```hcl
# count - fragile if list order changes
resource "aws_instance" "this" {
  count = length(var.instance_names)
  tags  = { Name = var.instance_names[count.index] }
}

# for_each - stable, keyed by name
resource "aws_instance" "this" {
  for_each = toset(var.instance_names)
  tags     = { Name = each.value }
}
```

**What to say:** "I default to `for_each` for anything where items have a natural identity, because state tracking is key-based and safer against reordering. I use `count` mainly for simple 0-or-1 conditional resource creation, or truly identical, order-independent copies. Dynamic blocks are for repeating nested configuration blocks within a single resource, like multiple ingress rules in a security group."

---

### Q8. How do you make a module genuinely reusable across environments (dev/staging/prod) instead of just reusable in theory?

**Simple explanation:** A module is only "reusable" if it doesn't hardcode anything environment-specific — names, sizes, account IDs, CIDR ranges. Everything environment-specific should come in as a variable, with the calling code (root module) supplying different `.tfvars` per environment.

**Deeper detail — practical checklist interviewers listen for:**
1. **No hardcoded values** — region, account ID, CIDR blocks, instance sizes all as variables.
2. **Sensible defaults where safe**, but never a default that could be "accidentally production" (e.g., don't default `environment = "prod"`).
3. **Naming driven by variables**, not literals — e.g., `name = "${var.project}-${var.environment}-web"`.
4. **Conditional logic for environment-specific behavior**, e.g., only enable deletion protection when `var.environment == "prod"`.
5. **Separate state per environment** (via workspaces or separate backend keys) — see Part 3.
6. **Tests / example usage** in the module repo (an `examples/` folder showing a minimal working call) so consumers don't guess.

**What to say:** "The test I apply is: could someone else, on a different team, consume this module for a completely different environment without editing the module's internals? That means no hardcoded names/CIDRs/account IDs, environment-driven conditionals for things like deletion protection, and clear input/output docs — usually enforced by keeping an `examples/` directory that's also used in automated testing."

---

## PART 3: Building Reusable, Compliant & Standardized Infrastructure Code

This section is about the "org-level" concerns: how do you stop 50 engineers from creating 50 slightly-different, slightly-insecure versions of "an S3 bucket"?

### 3.1 The core mental model: shift rules left, into code

"Compliance" traditionally meant a human manually reviewed infrastructure after it was built (or a security team ran periodic audits). The IaC approach is to **encode the rules into the pipeline** so violations are caught *before* anything is created — this is called **shifting left**.

There are three layers where you can enforce standards, from weakest to strongest://
1. **Convention / documentation** (weakest — relies on people reading and following docs)
2. **Module design** (variable validation, sensible/safe defaults, hardcoded non-negotiables like encryption always on)
3. **Automated policy enforcement in the pipeline** (strongest — code literally cannot merge or apply if it violates policy)

Good answers in this section usually show you understand layer 3 is what "true" compliance means, but layers 1-2 reduce how often you even hit a layer-3 rejection.

---

### Q9. How do you enforce standards (naming, tagging, security settings) across many teams using Terraform?

**Simple explanation:** You don't rely on people remembering the rules — you bake the rules into shared modules and automated checks, so it's *harder to do the wrong thing than the right thing*.

**Deeper detail — the toolbox:**
- **Shared/standardized modules**: e.g., an internal "S3 bucket" module that always turns on encryption and blocks public access, with no variable exposed to disable it. If a variable to disable a security control genuinely must exist, default it to the safe value.
- **Mandatory tagging via `default_tags`** (in the AWS provider block, for example) so every resource created under that provider automatically gets tags like `Environment`, `Owner`, `CostCenter` without each resource author remembering to add them.
```hcl
provider "aws" {
  default_tags {
    tags = {
      ManagedBy   = "Terraform"
      Environment = var.environment
      Owner       = var.team_name
    }
  }
}
```
- **Policy as Code**: automated tools that inspect a Terraform plan and reject it if it violates a rule, *before* apply. Main tools:
  - **Sentinel** — HashiCorp's policy-as-code framework, built into Terraform Cloud/Enterprise. Policies are written in Sentinel language and run against the plan output.
  - **OPA (Open Policy Agent) / Conftest** — general-purpose policy engine (Rego language), often used with plain Terraform + CI/CD.
  - **tfsec / Checkov / Trivy** — static analysis tools that scan `.tf` files (before even running `plan`) for known-insecure patterns (e.g., "S3 bucket without encryption," "security group open to 0.0.0.0/0 on port 22").
- **Pre-commit hooks**: run `terraform fmt`, `terraform validate`, `tflint`, and security scanners locally before a commit is even made, catching issues in seconds instead of waiting for CI.
- **CI/CD pipeline gates**: `plan` runs automatically on a pull request, output is posted as a PR comment for human review, and merge is blocked unless policy checks pass.

**What to say:** "I rely on layered enforcement: internal shared modules that bake in secure defaults so people don't need to remember rules, mandatory tagging via provider-level `default_tags`, static scanning with tools like tfsec or Checkov in CI, and for stronger guarantees, policy-as-code — Sentinel if we're on Terraform Cloud/Enterprise, or OPA/Conftest otherwise — that can actually block a non-compliant plan from being applied."

---

### Q10. What is Policy as Code, and how does Sentinel/OPA actually work in the pipeline?

**Simple explanation:** Policy as Code means writing your compliance/security rules as actual code that a tool can automatically check, instead of a PDF checklist a human reads. The tool inspects the *plan* (the list of proposed changes) and passes/fails it against your rules.

**Deeper detail:**
- Terraform generates a plan in a structured, machine-readable format (`terraform show -json plan.out`).
- A policy engine reads that structured plan and evaluates rules against it — e.g., "no resource of type `aws_s3_bucket` may have `acl = public-read`" or "every resource must have a `CostCenter` tag."
- **Sentinel** policies have enforcement levels: `advisory` (warns but doesn't block), `soft-mandatory` (blocks, but an authorized user can override), `hard-mandatory` (cannot be overridden by anyone).
- **OPA/Conftest** does the same conceptually but is open-source, cloud-agnostic, and commonly wired into GitHub Actions/GitLab CI rather than tied to Terraform Cloud.
- Either way, the check happens **between `plan` and `apply`** — that's the enforcement point. This is why understanding the plan/apply pipeline (Part 1) matters — policy as code is literally inserted as a gate in that pipeline.

**What to say:** "Policy as Code evaluates the structured plan output against rules before apply is allowed to run. Sentinel does this natively in Terraform Cloud/Enterprise with configurable enforcement levels — advisory, soft-mandatory, hard-mandatory. OPA/Conftest does the same thing in a cloud-agnostic way, usually wired into the CI pipeline as a required check on the pull request."

---

### Q11. How do you manage the same infrastructure code across multiple environments (dev/staging/prod)?

**Simple explanation:** You want the *same* module/code logic everywhere (so environments don't drift apart in behavior), but *different* input values and, critically, **separate state files** so a mistake in dev can never touch prod.

**Deeper detail — two common patterns:**

**Pattern A: Directory-per-environment (most common, most explicit)**
```
environments/
├── dev/
│   ├── main.tf        # calls the shared module
│   └── dev.tfvars
├── staging/
│   ├── main.tf
│   └── staging.tfvars
└── prod/
    ├── main.tf
    └── prod.tfvars
```
Each environment has its own backend configuration (its own state file location), so `terraform apply` in `dev/` can never touch prod's state, even by accident. This is the pattern most teams prefer because the blast radius of any mistake is contained by directory, and you can see exactly what's deployed where just by reading folders.

**Pattern B: Terraform Workspaces**
```bash
terraform workspace new dev
terraform workspace new prod
terraform workspace select prod
terraform apply
```
Workspaces let one set of `.tf` files manage multiple named state files (`terraform.tfstate.d/dev`, `.../prod`) using `terraform.workspace` inside the code to branch behavior:
```hcl
resource "aws_instance" "web" {
  instance_type = terraform.workspace == "prod" ? "m5.large" : "t3.micro"
}
```
Workspaces are lighter-weight but riskier for prod — it's easy to run `apply` while accidentally in the wrong workspace, since the code all lives in one place with no folder boundary as a visual safety net. Most production-grade setups prefer Pattern A for anything customer-facing, and reserve workspaces for lower-stakes, ephemeral environments (like per-feature-branch preview environments).

**What to say:** "I prefer separate directories per environment, each with its own backend/state, because the folder boundary itself prevents accidentally running an operation against the wrong environment — the blast radius is contained structurally, not just by remembering to switch a workspace. Workspaces are useful for something like ephemeral per-PR preview environments, where the stakes of a mix-up are low."

---

### Q12. What does "DRY" mean in the context of Terraform, and can you over-apply it?

**Simple explanation:** DRY = "Don't Repeat Yourself." In Terraform, it means factoring shared logic into modules instead of copy-pasting resource blocks across environments/projects.

**Deeper detail — yes, you can over-apply it:** A common interview trap is assuming "more DRY is always better." In practice:
- Over-abstracting a module with 40 optional variables trying to handle every possible use case makes it *harder* to understand and reason about than three duplicated but simple resource blocks.
- The standard guidance (echoing general software engineering wisdom) is: **duplicate twice, abstract on the third repetition** — don't build a generalized module speculatively before you've seen at least two or three real, similar use cases, because you'll guess the wrong abstraction.
- Good module design optimizes for **clarity of the interface** (inputs/outputs) more than minimizing lines of code.

**What to say:** "DRY is important, but I don't over-abstract early — I'll duplicate a resource block once or twice across use cases before extracting a module, because premature abstraction based on a guess about future needs tends to produce a module with too many optional flags that's harder to reason about than the duplication it replaced."

---

### Q13. How do you keep remote state secure and manage locking?

**Simple explanation:** The state file often contains sensitive data (sometimes plaintext secrets, like a generated database password) and is the "source of truth" for your infrastructure — if two people run `apply` at the same time against the same state, you can corrupt it. So you need secure, shared, locked storage for it.

**Deeper detail:**
- **Remote backend** (e.g., AWS S3, Azure Storage Account, GCS, Terraform Cloud) stores state centrally so the whole team/pipeline reads the same file, instead of everyone having their own local `terraform.tfstate` (which would immediately diverge).
- **State locking** prevents two concurrent `apply` operations from racing and corrupting the file. On AWS, this is classically done with a **DynamoDB table** alongside the S3 backend (newer Terraform versions support native S3 locking too); Terraform Cloud/Enterprise and other backends handle locking natively.
- **Encryption**: enable encryption at rest on the backend storage (S3 bucket encryption, Azure Storage encryption) and restrict IAM/access policies so only the pipeline (and break-glass admins) can read the state.
- **Never commit state to Git** — it's often excluded via `.gitignore`, precisely because of the sensitive data and merge-conflict risk (state files are JSON, not human-mergeable).

**What to say:** "State should always be in a remote, encrypted backend — like S3 with a DynamoDB lock table, or Terraform Cloud, which locks natively. Locking prevents two concurrent applies from corrupting state, and encryption plus tight IAM policies protect what's often sensitive data living inside the state file. It should never be committed to version control."

---

## PART 4: Troubleshooting — The Mental Model, Commands & Scenario Questions

This is the part interviewers weight heavily, because it separates "read a tutorial" candidates from people who've actually operated Terraform in production. Master the **general framework** first — then every scenario below (and any new one you're given live) is just an application of it.

### 4.1 The universal troubleshooting framework

Recall the triangle from Part 1:

```
Your .tf code (desired)   vs   State file (last known)   vs   Real infrastructure (actual)
```

**Every single Terraform problem is a mismatch somewhere in this triangle**, or an error in the mechanics of running the tool (auth, network, syntax, provider version). When faced with *any* Terraform problem — including ones you've never seen — walk this sequence out loud in an interview:

1. **Read the actual error message fully, top to bottom.** Terraform's errors are usually specific (resource address, attribute, reason). Resist the urge to guess before reading it.
2. **Identify which stage failed**: `init`? `validate`? `plan`? `apply` (and if apply, did it fail before or after some resources were already created)?
3. **Classify the failure type**:
   - Syntax/config error (caught by `validate`)
   - Provider/auth/network error (can't reach the API)
   - State vs. reality mismatch (drift, already-exists, not-found)
   - Dependency/ordering error (cyclic, missing `depends_on`)
   - Logic error (wrong conditional, wrong variable type)
4. **Reproduce safely**: run `terraform plan` (never jump straight to `apply` when debugging) to see current intended changes without affecting anything.
5. **Isolate**: use `-target` to scope a plan/apply to one resource, or comment out unrelated parts, to narrow down which resource/module is actually the problem.
6. **Inspect state directly** if you suspect a state/reality mismatch (`terraform state show`, `terraform show`).
7. **Fix at the right layer**: is the fix in code (`.tf` files), in state (surgical `state` commands), or in real infrastructure (fix manually then reconcile)?
8. **Validate the fix** with `plan` again before `apply`, confirming the diff is now exactly what you expect — nothing more.
9. **Prevent recurrence**: would a `validation` block, a `moved` block, CI check, or process change have caught this earlier?

That 9-step loop *is* your answer to "how do you approach troubleshooting Terraform issues" as a general interview question. Memorize the shape of it, not the exact wording.

### 4.2 Command cheat sheet (know what each one is *for*, not just its syntax)

| Command | What it's for |
|---|---|
| `terraform init` | Downloads providers/modules, configures backend. Run this after cloning a repo, adding a provider, or changing backend config. |
| `terraform init -upgrade` | Also upgrades providers/modules to the latest version allowed by constraints. |
| `terraform validate` | Checks HCL syntax and internal consistency (types, references) — no API calls, no state read. Fast first check. |
| `terraform fmt` | Auto-formats code to canonical style. Run before every commit. |
| `terraform plan` | Shows what would change. Always run before `apply`. Use `-out=plan.tfplan` to save an exact plan to apply later (avoids a race between plan and apply). |
| `terraform plan -target=<address>` | Scopes the plan to one resource/module — useful for isolating a problem or doing a careful, narrow change. Use sparingly; it can hide dependency issues. |
| `terraform apply` | Executes changes. `terraform apply plan.tfplan` applies a previously saved plan exactly (safer in automation). |
| `terraform destroy` | Tears down everything in scope. `-target` scopes it to specific resources. |
| `terraform show` | Human-readable dump of current state (or a saved plan file if given one). |
| `terraform state list` | Lists every resource address currently tracked in state. First command to run when you're lost about "what does Terraform think exists." |
| `terraform state show <address>` | Shows all attributes Terraform has recorded for one specific resource — great for comparing against real cloud console values. |
| `terraform state mv <src> <dst>` | Renames/moves a resource's tracked address in state *without* destroying/recreating it in real life. Used when refactoring code (e.g., moving a resource into a module). |
| `terraform state rm <address>` | Removes a resource from state tracking *without* deleting the real resource. Used when you want Terraform to "forget" about something (e.g., you're importing it elsewhere, or handing it off). |
| `terraform import <address> <real_id>` | Brings an already-existing real resource under Terraform's management by writing it into state. Doesn't generate the `.tf` code for you (in older Terraform) — you still write the matching resource block yourself. |
| `terraform taint <address>` (deprecated) / `terraform apply -replace=<address>` | Forces a specific resource to be destroyed and recreated on the next apply, even if Terraform doesn't otherwise think it changed. `-replace` is the modern, safer replacement for the old `taint` command. |
| `terraform refresh` (or `plan -refresh-only`) | Reconciles state with real infrastructure without changing either your code or real resources — updates state's recorded attributes to match reality. Useful to see/accept drift. |
| `terraform graph` | Outputs the dependency graph (can pipe to Graphviz) — helpful for understanding or debugging why Terraform is ordering operations a certain way, or finding cycles. |
| `terraform console` | An interactive REPL to evaluate expressions (e.g., test what a `for` expression or function actually returns) without running plan/apply. Excellent for debugging complex expressions. |
| `terraform providers` | Shows which providers are required and by which modules — useful for hunting down version conflicts. |
| `terraform output` | Prints output values from the current state. `-json` for machine-readable. |
| `terraform force-unlock <lock_id>` | Manually removes a stuck state lock. Dangerous — only use when you're certain no other operation is actually running. |
| `TF_LOG=DEBUG terraform apply` (env var) | Turns on detailed internal logging — including raw provider API requests/responses. `TRACE` is even more verbose. Set `TF_LOG_PATH` to write it to a file instead of flooding your terminal. |


### 4.3 Scenario-Based Troubleshooting Questions

For each scenario: read the **Scenario**, try to answer **before** reading the Approach — that's how you build the reflex.

---

#### Scenario 1: "Error acquiring the state lock" — apply won't run

**The error looks like:**
```
Error: Error acquiring the state lock
Lock Info:
  ID:        a1b2c3d4-...
  Path:      terraform.tfstate
  Operation: OperationTypeApply
  Who:       ci-runner@build-42
```

**Mental model:** This is a **mechanics-of-the-tool** problem, not a code or infrastructure problem. Terraform uses a lock (e.g., a DynamoDB item, or native backend locking) so two operations can't write state simultaneously and corrupt it. This error means Terraform believes a lock is currently held.

**Approach:**
1. Ask first: **is another apply actually running right now?** (Check your CI/CD pipeline for a concurrent/queued job, check if a teammate is running Terraform locally.) This is the most common real cause — don't touch the lock yet.
2. If genuinely nothing is running, the lock is likely **stale** — usually left behind because a previous run crashed, was killed (e.g., CI job timeout, laptop lost network mid-apply), or the process was force-terminated before it could release the lock.
3. Only after confirming no other real operation is in progress, remove the stale lock:
   ```bash
   terraform force-unlock <LOCK_ID>
   ```
   (The lock ID is printed in the error message.)
4. Re-run `terraform plan` to confirm state is in a sane state before applying.

**Why this matters / root cause prevention:** Locks exist specifically to prevent state corruption from concurrent writes. Force-unlocking while something *is* actually running is how people corrupt state — so the discipline is "verify, then unlock," never "unlock first."

---

#### Scenario 2: `Error: resource already exists` on apply

**The error looks like:**
```
Error: creating S3 Bucket: BucketAlreadyOwnedByYou: Your previous request
to create the named bucket succeeded and you already own it.
```

**Mental model:** This is a **state vs. reality mismatch** — specifically, the real resource exists in the cloud, but Terraform's state file doesn't know about it, so Terraform thinks it needs to *create* it (rather than manage the existing one), and the API rejects a duplicate-create call.

**Common causes:**
- The resource was created manually (console/CLI) outside Terraform.
- A previous `apply` created the resource but then failed *before* it could write the result into state (see Scenario 8).
- Someone ran `terraform state rm` on it earlier without meaning to.

**Approach:**
1. Confirm the resource genuinely already exists and matches what your code intends to create (don't blindly import something that might be a different, unrelated resource with a colliding name).
2. Bring it under management **without recreating it**:
   ```bash
   terraform import aws_s3_bucket.my_bucket my-bucket-name
   ```
3. Run `terraform plan` immediately after import — you will very likely see a diff, because the real resource's settings (encryption, versioning, tags) probably don't exactly match your `.tf` code yet. Reconcile by either editing your code to match reality, or editing real settings to match your code, until `plan` shows no changes.

**Why this matters:** `import` only creates the *state entry* — it does not generate `.tf` code for you in most Terraform versions (some newer workflows support `terraform plan -generate-config-out` to scaffold code from an existing resource, but always review generated code carefully).

---

#### Scenario 3: `plan` shows a resource will be destroyed and recreated, but you only changed one small thing

**The error/plan output looks like:**
```
  # aws_instance.web must be replaced
-/+ resource "aws_instance" "web" {
      ~ ami           = "ami-0111" -> "ami-0222" # forces replacement
```

**Mental model:** Not every cloud attribute can be changed in-place ("updated"). Some attributes are **immutable** at the API level — the only way to change them is to destroy the resource and create a new one. Terraform annotates this in the plan with `# forces replacement`.

**Approach:**
1. **Always read the `# forces replacement` annotation** — Terraform tells you exactly which attribute triggered it.
2. Decide if replacement is actually acceptable:
   - If yes (e.g., it's a dev instance, or downtime is fine), proceed with apply.
   - If no (e.g., it's a production database — replacement means data loss), you need a different approach:
     - Check if there's an alternative attribute/resource design that supports in-place update instead (e.g., some resources have separate "modify" vs "replace" attributes).
     - Use the `create_before_destroy` lifecycle rule to at least avoid downtime, if replacement is unavoidable but downtime isn't acceptable:
       ```hcl
       lifecycle {
         create_before_destroy = true
       }
       ```
     - For truly critical stateful resources (databases), plan an explicit migration strategy (snapshot/restore) rather than letting Terraform destroy/recreate.

**Why this matters:** This is one of the most common "silent production incident" causes — someone changes what looks like a harmless field and doesn't notice the plan says "replace" instead of "update in-place." Reading plan output line-by-line, especially the +/- symbols and annotations, is a core discipline.

---

#### Scenario 4: Someone changed a resource manually in the console — now `plan` shows unexpected changes ("drift")

**Mental model:** This is **drift** — reality moved away from what state (and code) say it should be. Terraform doesn't proactively watch for changes; it only notices drift the next time you run `plan`/`apply`, when it refreshes state against real infrastructure.

**Approach:**
1. Run `terraform plan` — Terraform will show a diff between the manually-changed real value and what your code says it should be.
2. Decide which side is "correct":
   - If the code is correct (the manual change was a mistake or unauthorized) → run `terraform apply` to revert reality back to match code.
   - If the manual change should actually be kept → update your `.tf` code to match the new reality, so the *next* plan shows no diff.
3. To just *see* drift without changing anything (read-only investigation):
   ```bash
   terraform plan -refresh-only
   ```
   This updates state to reflect reality (if you then `apply` the refresh-only plan) without touching your `.tf` files or real resources — useful purely for visibility/reporting on drift.

**Why this matters / prevention:** The long-term fix for drift isn't a command, it's a process: restrict console/manual access to production resources (least-privilege IAM), and route *all* changes through the Terraform pipeline (pull request → plan → review → apply) so there's no "side door" for reality to diverge from code.

---

#### Scenario 5: You need to move a resource into a new module without destroying and recreating it

**Scenario in plain words:** You refactored your code — a resource that used to live directly in the root module now needs to live inside `module "networking"`. If you just move the code, Terraform will see the old address disappear and a new address appear, and plan a destroy + create — even though nothing about the real resource needs to change.

**Mental model:** State tracks resources by **address** (e.g., `aws_vpc.main` vs. `module.networking.aws_vpc.main`). Moving code without telling Terraform about the address change looks, to Terraform, like "delete the old one, create a brand new one" — because from state's point of view, the old address vanished and an unrelated new address appeared.

**Approach — two ways, prefer the first:**

**Option A (modern, preferred): `moved` block** — declare the move directly in code so it's version-controlled and repeatable for every teammate/environment:
```hcl
moved {
  from = aws_vpc.main
  to   = module.networking.aws_vpc.main
}
```
Run `terraform plan` — Terraform detects the `moved` block and treats it as a rename, not a destroy/create. No manual CLI command needed, and it's documented in code for anyone else pulling the same change.

**Option B (manual, older approach):**
```bash
terraform state mv aws_vpc.main module.networking.aws_vpc.main
```
This directly edits the state file's tracked address. Downside: it's a one-time manual action per state file (someone has to remember to run it against every environment's state — dev, staging, prod — separately), whereas a `moved` block travels with the code automatically.

**Why this matters:** This shows you understand that "refactoring Terraform code" is fundamentally different from refactoring application code — the *state* has to be told about renames explicitly, or Terraform will interpret a rename as delete+create.

---

#### Scenario 6: `Error: Cycle: aws_instance.a, aws_instance.b`

**Mental model:** Terraform builds a **dependency graph** (a DAG — Directed Acyclic Graph) from your resource references, to figure out the correct order to create/update/destroy things. A cycle means resource A depends on B, and B (directly or indirectly) depends on A — there's no valid order to create them in.

**Approach:**
1. Look at what's creating the mutual dependency — usually one of:
   - Resource A references an attribute of B, and Resource B references an attribute of A (direct cycle).
   - An unnecessary `depends_on` was added on both sides.
   - A more subtle indirect cycle through 3+ resources (A → B → C → A).
2. Use `terraform graph` to visualize the dependency chain if it's not obvious from reading the code.
3. Break the cycle by removing the unneeded reference/`depends_on` on at least one side. Often the real fix is realizing one of the two "dependencies" isn't actually needed — e.g., two security groups referencing each other's ID for ingress rules should instead use a separate `aws_security_group_rule` resource attached after both groups already exist, rather than inline rules that create a circular reference.

**Why this matters:** Cycles usually reveal a design issue, not just a syntax fix — the resolution is often restructuring which resource "owns" a piece of configuration, not just reordering code.

---

#### Scenario 7: A resource is created before something it actually depends on, causing a runtime failure (e.g., "security group not found")

**Mental model:** Terraform normally infers ordering automatically through **implicit dependencies** — if resource B references `resource.a.id` anywhere in its config, Terraform knows to create A first. Problems happen when a dependency exists in the *real world* (e.g., "this IAM role's permissions must exist before this Lambda tries to use them") but is **not visible in the code** as an attribute reference — so Terraform doesn't know to order them.

**Approach:**
1. Identify: is the dependency expressed anywhere in the HCL as an attribute reference? If not, Terraform has no way to know about it.
2. Add an **explicit dependency** using `depends_on`:
   ```hcl
   resource "aws_lambda_function" "this" {
     # ... config ...
     depends_on = [aws_iam_role_policy_attachment.lambda_exec]
   }
   ```
3. Use `depends_on` sparingly — only when there's a real dependency not expressible through an attribute reference (e.g., eventual-consistency issues like IAM permission propagation delay, or ordering that matters for reasons outside Terraform's visibility). Overusing `depends_on` makes the dependency graph harder to reason about and can introduce accidental cycles.

**Why this matters:** This tests whether you understand implicit vs. explicit dependencies — a very common real-world gotcha, especially with IAM (permissions can take a few seconds to propagate even after the API confirms creation, causing intermittent "access denied" failures right after a role is attached).

---

#### Scenario 8: `apply` fails partway through — some resources were created, others weren't

**Mental model:** Terraform applies changes largely in dependency order but the API calls are real, individual operations — if resource #4 out of 10 fails (e.g., hit a quota limit, or a naming conflict), Terraform doesn't roll back the first three. Whatever was successfully created **is already real and is recorded in state** (Terraform updates state incrementally as each resource succeeds, not only at the very end). This is intentional — Terraform never has "transactions" the way a database does, because cloud APIs don't support that either.

**Approach:**
1. Don't panic or immediately re-run `apply` blindly — first read the error to understand exactly which resource failed and why (quota limit, naming conflict, invalid parameter, transient API error).
2. Run `terraform plan` — because state already reflects what *did* get created, the plan will now correctly show only the *remaining* work needed (it won't try to recreate the resources that already succeeded).
3. Fix the root cause of the failure (e.g., request a quota increase, fix a naming collision, correct an invalid parameter).
4. Re-run `terraform apply` — it will pick up where it left off, because state is the accurate source of "what's already done."

**Why this matters:** This question tests whether you understand that state is updated incrementally, resource-by-resource, during apply — not just once at the end. That's *why* re-running apply after a partial failure is normally safe and simply resumes, rather than duplicating already-created resources.

---

#### Scenario 9: Provider version conflict between modules

**The error looks like:**
```
Error: Failed to query available provider packages
Could not retrieve the list of available versions for provider
hashicorp/aws: no available releases match the given constraints
```
or
```
Error: Incompatible provider version
Module "vpc" requires provider aws >= 5.0
Module "ec2" requires provider aws < 4.0
```

**Mental model:** Different modules in your configuration can each declare their own `required_providers` version constraints. Terraform needs to find a *single* provider version that satisfies every constraint simultaneously across the whole configuration (Terraform doesn't run two versions of the same provider side-by-side in one configuration).

**Approach:**
1. Run `terraform providers` to see which modules require which provider versions.
2. Identify the actual conflicting constraints (as in the error above).
3. Resolve by either:
   - Upgrading the module with the older/lower constraint to a version compatible with the newer provider, or
   - Relaxing an overly strict constraint (e.g., changing `= 4.2.0` to `~> 4.2`) if that module's author was just being overly conservative and the module actually works fine on newer minor versions, or
   - As a last resort, pinning the whole configuration to a provider version that satisfies the *lowest common denominator*, and tracking a follow-up to upgrade the lagging module.

**Why this matters:** This shows understanding that provider version constraints are resolved globally across an entire configuration's module tree, not per-module in isolation — a common surprise for people newer to multi-module setups.

---

#### Scenario 10: Authentication/permission errors on apply (e.g., `AccessDenied`, `UnauthorizedOperation`)

**Mental model:** This is not a Terraform bug — Terraform is just the messenger relaying whatever the cloud provider's API returned. The credentials Terraform is using (from environment variables, a shared credentials file, an assumed IAM role, a CI/CD pipeline's identity, etc.) don't have permission to perform the specific API call being attempted.

**Approach:**
1. Read exactly which action was denied (the error usually names the specific API action, e.g., `ec2:CreateSecurityGroup`, and sometimes the resource ARN).
2. Confirm *which* identity Terraform is actually running as — this is the step people skip. Locally: `aws sts get-caller-identity` (for AWS) shows the actual authenticated identity, which is often not what you assumed (wrong profile, expired assumed-role session, wrong environment variables set).
3. Compare that identity's attached policy against the denied action — is the permission simply missing, or is there an explicit `Deny` (e.g., an SCP or permission boundary) overriding an `Allow`?
4. Fix at the right layer: add the missing IAM permission, fix the assumed-role trust policy, or refresh expired temporary credentials — never work around this by escalating to overly broad permissions (e.g., `"*"`) as a "fix," which is itself a compliance red flag interviewers listen for you to avoid.

**Why this matters:** Interviewers use this to check you don't treat every error as "a Terraform problem to fix in `.tf` files" — recognizing when an error is actually about credentials/IAM outside Terraform's config is a key diagnostic skill.

---

#### Scenario 11: `terraform destroy` (or apply removing a resource) hangs or fails with a dependency violation

**The error looks like:**
```
Error: error deleting VPC: DependencyViolation: The vpc has dependencies
and cannot be deleted.
```

**Mental model:** Terraform asked the cloud API to delete a resource (e.g., a VPC), but the *cloud provider itself* refuses because something else still references it (e.g., a network interface, a resource Terraform doesn't manage, or a resource created manually that lives inside that VPC). This is a reality-level constraint, not a Terraform state issue.

**Approach:**
1. Identify what's actually still attached — often *not* visible in your Terraform code at all, because it's a resource that exists outside Terraform's management (created manually, or by a different Terraform state/team).
2. Check the cloud console/CLI directly for the resource (e.g., `aws ec2 describe-network-interfaces --filters Name=vpc-id,Values=<vpc-id>`) to find the blocking dependency.
3. Either remove the blocking resource manually (if it's genuinely orphaned/unneeded) or bring it under Terraform management so future changes are coordinated, then retry `terraform destroy`/`apply`.

**Why this matters:** This tests whether you know to look *outside* the Terraform codebase when the cloud API itself is the one refusing the operation — not every problem is solvable by editing `.tf` files.

---

#### Scenario 12: A `variable` type error breaks `plan`, e.g., `Error: Invalid value for input variable`

**The error looks like:**
```
Error: Invalid value for variable
on variables.tf line 12:
  12: variable "subnet_count" {
Invalid value for "subnet_count": string required.
```

**Mental model:** This is caught very early — usually at `plan`/`validate` time, before any API call — because Terraform enforces the `type` you declared for a variable, and validation blocks add further business-logic checks. This is a *design feature*, not a bug: it's meant to fail loud and early rather than send bad data into a cloud API deep into `apply`.

**Approach:**
1. Read exactly which variable and what type mismatch (e.g., a `.tfvars` file passing a string where a `number` or `list` was declared).
2. Check every place that variable's value could be coming from, in Terraform's precedence order (highest to lowest priority):
   - Command-line `-var` flags
   - `-var-file` flagged files
   - `*.auto.tfvars` files (auto-loaded)
   - `terraform.tfvars` (auto-loaded)
   - Environment variables (`TF_VAR_name`)
   - The variable's own `default` in `variables.tf`
3. Fix the source providing the wrong type, or relax/correct the declared `type` if the type constraint itself was wrong.
4. Re-run `terraform validate` (fast, no state/API needed) before `plan` to confirm the fix.

**Why this matters:** Knowing the **variable precedence order** cold is a very common interview trip-up — many candidates know types exist but can't say which source wins when a variable is defined in multiple places.

---

#### Scenario 13: `plan`/`apply` becomes extremely slow as the project grows

**Mental model:** Performance problems are almost always about **scale of state and provider calls**, not the HCL code itself being "slow" (HCL evaluation is cheap). The bottleneck is usually the number of API calls Terraform's providers make to refresh/read the current state of every tracked resource before computing a plan.

**Approach:**
1. Confirm the bottleneck is really the refresh step: run with `TF_LOG=DEBUG` and look at how much time is spent on API read calls vs. actual graph computation.
2. Common fixes, in order of how often they help:
   - **Split one giant state file into multiple smaller states** (e.g., per environment, per major component like "networking" vs. "compute" vs. "data"), using **remote state data sources** (`terraform_remote_state`) to reference outputs across states where needed. Smaller state = fewer resources to refresh per operation.
   - Use `-target` only for genuinely scoped, temporary work — not as a permanent workaround for a state that's grown too large.
   - Increase `-parallelism=n` (default is 10) if the cloud API and your account's rate limits can handle more concurrent calls — this speeds up plan/apply by doing more API calls in parallel. (Careful: this can worsen Scenario 14 below if you push past provider rate limits.)
   - For very large repeated resource sets, double-check `for_each`/`count` aren't being used at a scale (thousands of items) that's simply beyond what one state file should reasonably hold.

**Why this matters:** This is a scaling/architecture question — the "real" fix is almost always splitting state boundaries sensibly (by team ownership, by blast radius, by change frequency), not a CLI flag.

---

#### Scenario 14: Cloud API throttling / rate limit errors during apply (e.g., `Throttling: Rate exceeded`)

**Mental model:** Cloud providers cap how many API requests you can make per second. When Terraform manages many resources with high `-parallelism`, it can legitimately hit these caps, especially in accounts with lower default limits or against services with historically low rate limits (e.g., IAM).

**Approach:**
1. Confirm from the error that it's genuinely a throttling response from the provider (not a permissions or config error mistaken for one).
2. Reduce concurrency: `terraform apply -parallelism=5` (or lower) so fewer simultaneous API calls are made.
3. Most providers implement automatic retry-with-backoff for throttling internally — if it's still failing, check if the provider version in use is outdated and missing retry improvements; upgrading the provider can help.
4. For a structural fix, request a service quota/rate-limit increase from the cloud provider if this is a recurring, legitimate scale issue (not just a one-off).

**Why this matters:** Distinguishing "the API is telling me I have a permissions problem" (Scenario 10) from "the API is telling me I'm going too fast" (this scenario) is a basic but important error-reading skill.


---

#### Scenario 15: Sensitive data (a password, API key) is visible in the state file or in `plan` output

**Mental model:** Marking a variable or output `sensitive = true` only **masks it from CLI/log output** — it does **not encrypt it inside the state file**. The state file stores the full, real attribute values of every resource in plain JSON, because Terraform needs the actual values to detect drift and compute diffs. This is a very common misconception to correct in an interview — pointing it out shows depth.

**Approach / how you actually protect sensitive data:**
1. Ensure the **backend itself** encrypts data at rest (e.g., S3 bucket with SSE encryption enabled, Terraform Cloud which encrypts state natively) — this is the real protection layer, not `sensitive = true`.
2. Restrict **who/what can read the state file** via IAM policy / access controls on the backend — treat read access to state as equivalent to read access to secrets.
3. Avoid putting long-lived secrets in Terraform variables/state at all where possible — prefer having Terraform provision a reference (e.g., create a secret's *name* in a secrets manager) while the actual secret value is generated/rotated and stored in a dedicated secrets manager (AWS Secrets Manager, Vault, etc.), not passed through `.tfvars` or state directly.
4. Use `sensitive = true` anyway — it's still useful for preventing accidental exposure in CI logs, PR comment plan output, and terminal scrollback, which are common real-world leak vectors even though it's not encryption.

**What to say:** "`sensitive = true` hides a value from CLI output and logs, but the state file itself still stores the real value in plaintext JSON — so the actual protection has to come from encrypting the backend at rest and tightly restricting who can read state, plus, ideally, keeping long-lived secrets out of Terraform entirely by delegating them to a secrets manager."

---

#### Scenario 16 (meta-question): "Walk me through how you'd troubleshoot an error you've genuinely never seen before."

This question is designed to see your *process*, since they can't test you on every possible error. Give them the general framework from 4.1, but say it out loud as a story:

**What to say:**
"First, I read the full error message rather than pattern-matching on the first line — Terraform errors are usually specific about which resource address and attribute is involved. Then I figure out which stage failed: init, validate, plan, or apply — that tells me if it's a config/syntax issue, a provider/auth/network issue, or a state-versus-reality mismatch. If it's unclear, I reproduce it safely with `plan` rather than `apply`, and use `-target` to isolate which resource is actually responsible. If I suspect a state issue specifically, I'll inspect it directly with `terraform state list` and `state show` rather than guessing. If I still can't tell what's happening internally, I'll turn on `TF_LOG=DEBUG` to see the actual API calls and responses underneath Terraform's abstraction — at that point I'm usually looking at a genuine cloud API error message, which I can search or reason about directly, since it's no longer a 'Terraform mystery,' it's a concrete API response. Finally, once fixed, I think about whether a validation rule, a `moved` block, or a CI check could catch this earlier next time."

---

## PART 5: Rapid-Fire Reference (last-minute review)

- **Module** = reusable folder of `.tf` files, called like a function via input variables and output values.
- **Root module** = where you run the CLI. **Child module** = anything called via a `module` block.
- **`for_each` over `count`** for anything with a natural identity — state keys are stable against reordering.
- **Semantic versioning** for modules: MAJOR = breaking, MINOR = new backward-compatible feature, PATCH = bug fix. Consumers pin with `~>`.
- **State** = Terraform's record of reality; almost every bug is a mismatch between code, state, and real infrastructure.
- **Policy as Code** (Sentinel / OPA) runs between `plan` and `apply` to block non-compliant changes automatically.
- **`default_tags`** at the provider level enforce mandatory tagging without relying on every engineer remembering.
- **Separate state per environment** (directory-per-env preferred over workspaces for production, due to the structural safety of folder boundaries).
- **`moved` block** > `terraform state mv` for refactors — it's version-controlled and applies automatically for every environment/teammate.
- **`depends_on`** only needed when a real dependency isn't visible through an attribute reference (e.g., IAM propagation delay).
- **Partial apply failures are safe to re-run** — state updates incrementally per resource, not just at the end.
- **`sensitive = true`** hides values from output/logs — it does **not** encrypt the state file. Backend encryption + access control is the real protection.
- **State locking** (e.g., DynamoDB + S3) prevents concurrent applies from corrupting state — verify nothing is actually running before `force-unlock`.
- **`# forces replacement`** in a plan means the attribute is immutable at the API level — always check if that's actually acceptable before applying.

---

## PART 6: How to carry yourself in the interview

- **Narrate your mental model out loud**, even for scenarios you instantly recognize — interviewers are grading your *reasoning process* as much as the final answer, because that's what predicts how you'll handle a *novel* production incident they haven't scripted.
- When given a scenario, it's completely fine to ask **one clarifying question** first (e.g., "is this in a shared/production state, or an isolated dev environment?") — real engineers gather context before diagnosing, and it signals maturity rather than uncertainty.
- If you genuinely don't know a specific command's exact syntax, say what you'd do conceptually and that you'd check `terraform <command> -help` or the docs — that's honest and realistic; nobody has every flag memorized.
- Bring your own real example if you have one (even a small one) — a specific war story ("we had a drift issue where someone manually resized an RDS instance, here's how we resolved and then prevented it") is more convincing than reciting theory perfectly.
