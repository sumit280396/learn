# Setting Up Domino Data Lab on AWS EKS — A Novice-Friendly Walkthrough
### Every step explained with the WHY behind it

---

## The Big Picture Analogy (read this first)

Think of it like opening a **co-working office building** for data scientists:

- **AWS** = the city (land, electricity, water)
- **EKS (Kubernetes)** = the building itself — floors, rooms, elevators that decide where people work
- **Domino** = the property management company you install inside the building — reception desk, room booking system, security badges, cleaning staff
- **Workspaces/Jobs** = the actual offices people book and work in

You can't install the management company before the building exists, and you can't build the building before the land is ready. That's why the steps below happen in this exact order: **land → building → utilities → management company → open the doors.**

---

## PHASE 0: Planning & Prerequisites (before touching anything)

### Step 0.1 — Get the Domino license and installer access
Domino is commercial enterprise software, not open source. You contact Domino Data Lab, sign a license, and they give you access to their installer tooling and container images (or credentials to pull them from their registry).

**Why:** Without license credentials, you literally cannot download Domino's images. This is also when Domino's team tells you which Kubernetes versions the current Domino release supports — write that number down, it controls Step 2.

### Step 0.2 — Check the compatibility matrix
Every Domino version supports specific Kubernetes versions (e.g., "Domino X.Y supports EKS 1.28–1.30").

**Why:** If you build a cluster on a Kubernetes version Domino doesn't support, the install fails or behaves unpredictably. This mismatch is one of the most common real-world install failures. Rule: **read the compatibility matrix before creating anything.**

### Step 0.3 — Request AWS service quota increases
In the AWS console, request quota increases for the EC2 instance types you'll need — especially GPU instances (g4dn, g5, p3) which default to very low or zero quota in new accounts.

**Why:** Quotas take hours-to-days for AWS to approve. If you skip this, everything works until the first data scientist requests a GPU workspace and the autoscaler silently fails because AWS refuses to give you the instance. Do it now so it's approved by the time you need it.

### Step 0.4 — Size the deployment
Decide roughly: how many concurrent users, how many need GPUs, how big is the data. This determines node instance types and counts.

**Why:** The platform's own services (databases, queues, web frontend) need dedicated capacity — typically a handful of medium-large nodes just for Domino itself — *before* a single user logs in. Undersizing the platform nodes makes everything feel broken later.

---

## PHASE 1: Build the Network (the land and roads)

### Step 1.1 — Create a VPC with public and private subnets across 3 Availability Zones
Use Terraform or eksctl to create a VPC: private subnets (where all nodes live) and small public subnets (only for load balancers and NAT gateways), spread across 3 AZs.

**Why private subnets:** Your cluster nodes should never be directly reachable from the internet — that's basic security. Users reach Domino only through a load balancer.

**Why 3 AZs:** An Availability Zone is a physically separate data center. If one AZ has an outage and everything you own is in it, your platform is down. Spreading across 3 gives high availability.

**Why NAT gateways:** Nodes in private subnets still need *outbound* internet (to pull Docker images, download Python packages). A NAT gateway allows outbound-only traffic — nodes can call out, nothing can call in.

---

## PHASE 2: Create the EKS Cluster (the building)

### Step 2.1 — Create the EKS control plane
Using Terraform or `eksctl`, create the EKS cluster on the Kubernetes version from your compatibility matrix, placed in the VPC from Phase 1.

**Why EKS instead of running Kubernetes yourself:** AWS manages the Kubernetes control plane (API server, etcd, scheduler) for you — patching it, keeping it highly available. You only manage worker nodes. Managing your own control plane is a full-time job you don't want.

### Step 2.2 — Create THREE separate node groups
This is the most important architectural decision, so understand it well:

**Node group 1 — "platform" nodes** (e.g., 3–4 × m5.2xlarge)
For Domino's own services: web frontend, MongoDB, Keycloak, RabbitMQ, monitoring. **Apply a Kubernetes taint** to these nodes (e.g., `domino-platform=true:NoSchedule`).

**Why the taint:** A taint is a "no entry unless invited" sign. It guarantees user workloads can never land on these nodes. Without it, one data scientist's memory-hungry notebook could get scheduled next to your database, starve it, and take the whole platform down for everyone. Isolation of control plane from workloads is a core reliability principle.

**Node group 2 — "compute" nodes** (e.g., m5.2xlarge/m5.4xlarge, autoscaling 1→N)
Where user workspaces and jobs actually run. Configured to autoscale.

**Why autoscaling:** Data science load is bursty — heavy 10am–6pm, idle at night. Autoscaling means you pay for nodes only when workloads need them.

**Node group 3 — "GPU" nodes** (e.g., g4dn.xlarge, autoscaling 0→N, tainted `nvidia.com/gpu:NoSchedule`)
For GPU workloads only.

**Why scale-to-zero:** GPU instances cost several dollars/hour. When nobody is training a model, you want zero of them running.
**Why tainted:** So an ordinary CPU notebook never accidentally occupies a $3/hour GPU machine. Only pods that explicitly tolerate the taint (GPU hardware tiers) land here.

### Step 2.3 — Install cluster add-ons
Install: **Cluster Autoscaler** (or Karpenter), **EBS CSI driver**, **EFS CSI driver**, **NVIDIA device plugin** (on GPU nodes), and a **metrics server**.

**Why each:**
- *Cluster Autoscaler:* watches for pods that can't be scheduled and adds nodes; removes idle nodes. This is the engine behind "user clicks Start Workspace and capacity appears."
- *CSI drivers:* CSI = Container Storage Interface. These are the plugins that let Kubernetes create and attach AWS EBS volumes and EFS filesystems to pods. No driver = no storage = pods stuck.
- *NVIDIA device plugin:* Kubernetes doesn't natively know what a GPU is. This plugin advertises GPUs as schedulable resources so a pod can request `nvidia.com/gpu: 1`.

---

## PHASE 3: Storage (the warehouse and filing cabinets)

### Step 3.1 — Create an S3 bucket (or several)
For Domino's **blob storage**: project files, job results, logs, backups, and the internal Docker registry's backend.

**Why S3:** It's effectively infinitely scalable, cheap, durable (11 nines), and Domino is built to use it natively on AWS. Enable versioning and encryption on the bucket — in a pharma/GxP context, encryption at rest is non-negotiable.

### Step 3.2 — Create an EFS filesystem
For **Domino Datasets** — the large shared data that multiple users and projects mount simultaneously.

**Why EFS and not EBS:** EBS is like a personal hard drive — it can attach to only one node at a time. EFS is a network filesystem (NFS) that hundreds of pods across many nodes can mount **at the same time**. Shared datasets require exactly that. Choose the throughput mode carefully — EFS burst credits running out is a classic "why is everything slow" root cause later.

### Step 3.3 — Define Kubernetes StorageClasses
Create StorageClasses pointing at EBS (for per-pod scratch volumes) and EFS (for shared datasets).

**Why:** A StorageClass is a named recipe — when Domino says "give this workspace 50 GB of scratch," Kubernetes looks up the recipe and provisions the right AWS storage automatically. Domino's installer config will reference these class names.

---

## PHASE 4: Identity & Permissions (the badges and keys)

### Step 4.1 — Set up IRSA (IAM Roles for Service Accounts)
Enable the OIDC provider on the EKS cluster and create IAM roles that specific Kubernetes service accounts can assume — e.g., Domino's blob-storage service gets a role allowing access to *only* its S3 bucket; the autoscaler gets a role allowing *only* EC2 scaling actions.

**Why IRSA instead of one big node role:** If you attach broad S3 permissions to the node itself, *every pod on that node* — including any user's notebook — inherits them. IRSA gives each service its own narrowly-scoped identity. This is the principle of least privilege, and in a regulated life-sciences environment, auditors will specifically look for this.

---

## PHASE 5: Domain Name & TLS (the street address and door locks)

### Step 5.1 — Choose the platform hostname and create the TLS certificate
Pick the URL users will visit (e.g., `domino.yourcompany.com`), create/validate a certificate in AWS ACM (or bring your company's cert), and prepare the Route53 DNS zone.

**Why before installing:** Domino's installer config requires the hostname up front — it's baked into how Keycloak (SSO), the web frontend, and workspace routing generate URLs. Changing the hostname after install is painful. And HTTPS isn't optional: credentials and proprietary research data flow through this URL.

---

## PHASE 6: Install Domino Itself (move the management company in)

### Step 6.1 — Write the installer configuration file
Domino ships an installer (historically the `fleetcommand-agent`) driven by a single YAML configuration file. In it you declare everything you built above: Domino version, hostname, the S3 bucket names, the EFS filesystem/StorageClass names, which node pools are platform vs compute vs GPU (matched by node labels), SSL certificate details, and your license credentials.

**Why config-driven:** One reviewable, version-controlled file describes the entire deployment. In a GxP world this is gold — the config file *is* part of your installation qualification (IQ) evidence, and re-running the installer with the same file gives a reproducible result.

### Step 6.2 — Run the installer
The installer connects to your cluster and deploys Domino: it creates the `domino-platform` namespace and brings up roughly 50–70 pods — the web frontend, API server, dispatcher, Keycloak, MongoDB, Postgres, RabbitMQ, the internal Docker registry, Prometheus, Grafana, and more. This takes a while (30–60+ minutes).

**Why so many pods:** Domino is a microservices application. Each concern — auth, metadata, queuing, image building, monitoring — is its own service. Watch progress with `kubectl get pods -n domino-platform -w` and wait until everything is `Running`.

### Step 6.3 — Point DNS at the load balancer
The install creates an AWS load balancer. Create the Route53 record pointing `domino.yourcompany.com` at it.

**Why last:** The load balancer's address doesn't exist until the installer creates it. Once DNS propagates, browsing to the URL should show the Domino login page — your first "it's alive" moment.

---

## PHASE 7: Post-Install Configuration (open for business)

Do these in order, each building on the last:

### Step 7.1 — Log in as the initial admin and connect SSO
Wire Keycloak to your company identity provider (Okta / Azure AD / etc.) via SAML or OIDC.

**Why:** You do not want a separate password system for a regulated platform. SSO means joiners/leavers are handled by corporate IT, and access reviews (a GxP/audit requirement) have one source of truth.

### Step 7.2 — Create Hardware Tiers
Define tiers like `small (2 CPU / 8 GB)`, `large (8 CPU / 32 GB)`, `gpu-small (4 CPU / 16 GB / 1 GPU)` — with resource sizes that **fit cleanly into your node instance sizes**, and with GPU tiers carrying the toleration + node selector for the GPU pool.

**Why the sizing care:** If your nodes have 16 GB and you define a 12 GB tier, only one such pod fits per node and 4 GB is wasted on every node, forever. Tiers that bin-pack cleanly onto instance sizes are the difference between a 60% and 90% utilized (i.e., affordable) platform.

### Step 7.3 — Build the first Compute Environments
Create a standard Python environment and an R environment: choose base images, add your organization's common packages, **pin every version**.

**Why pinning:** Unpinned packages mean the environment silently changes every rebuild — and "my code gave different results this month" is both a support nightmare and a GxP reproducibility violation. Pinned, versioned environments are the foundation of everything Domino promises a pharma customer.

### Step 7.4 — Configure Data Sources
Set up managed connections to S3, Snowflake, Redshift, etc., with service credentials, and check network paths (security groups) from the compute nodes to those systems.

**Why now:** A platform users can log into but can't reach data from is useless. Testing connectivity from *inside a compute pod* now saves you the "data source down" tickets on day one.

### Step 7.5 — Verify monitoring and set up backups
Confirm the bundled Prometheus/Grafana dashboards work, add alerts (platform pod health, node disk, queue depth, certificate expiry), and schedule backups of MongoDB/Postgres plus S3 versioning/replication.

**Why before launch:** Monitoring installed *after* the first outage is a lesson everyone learns exactly once. And your metadata databases hold every project's history — losing MongoDB without a backup means losing the platform's memory.

---

## PHASE 8: Validation / Smoke Testing (the inspection before opening day)

Run this checklist end-to-end as a real user would:

1. Log in via SSO → **proves** auth chain works
2. Create a project, start a workspace on each tier (including GPU) → **proves** scheduling, autoscaling, image pull, ingress routing
3. Run a batch job → **proves** dispatcher + queue path
4. Read a file from each data source → **proves** credentials + network paths
5. Write to a Domino Dataset → **proves** EFS mounting
6. Publish a trivial Model API and curl it → **proves** the serving + routing layer
7. Stop everything and confirm nodes scale back down → **proves** you won't be surprised by the AWS bill

**Why a written checklist:** In a GxP environment this checklist, executed and signed off, becomes your **OQ (Operational Qualification)** evidence. And you'll reuse the exact same checklist after every future upgrade to prove nothing broke — which is precisely the upgrade discipline the job description asks about.

---

## The One-Paragraph Version (memorize for the interview)

> "You start with planning — license, Kubernetes compatibility matrix, AWS quota increases for GPU instances. Then networking: a VPC with private subnets across three AZs. Then the EKS cluster with three node groups — tainted platform nodes so Domino's own services are isolated from user workloads, autoscaling compute nodes, and scale-to-zero tainted GPU nodes — plus the cluster autoscaler, CSI drivers, and NVIDIA device plugin. Storage is S3 for blob data, EFS for shared datasets, EBS for scratch, exposed via StorageClasses. IRSA gives each Domino service least-privilege AWS access. You pre-provision the hostname and TLS cert because they're baked into the install config. Then Domino's installer, driven by one version-controlled YAML referencing all of the above, deploys the platform services into their namespace. Post-install: SSO via Keycloak, hardware tiers sized to bin-pack node instances, pinned compute environments, data sources, monitoring, and backups. Finally an end-to-end smoke test — which in a GxP shop doubles as qualification evidence and becomes the standard post-upgrade validation checklist."

If you can deliver that paragraph, you sound like someone who has done this — because you understand *why* each piece exists, which is what actually distinguishes senior candidates.
