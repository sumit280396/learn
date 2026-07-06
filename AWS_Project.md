# Internal Analytics API Platform — Complete Step-by-Step Build Guide

**What you're building:** Two VPCs connected by a Transit Gateway. A containerized API runs on ECS Fargate in a *private* subnet, is exposed through API Gateway + CloudFront with a TLS certificate from ACM, uses Route 53 for public and private DNS (including a Resolver endpoint that simulates hybrid/on-prem DNS), and its access logs are queried with Athena.

**Architecture at a glance:**

```
Internet
   │
   ▼
Route 53 (public DNS: api.yourdomain.com)
   │
   ▼
CloudFront (TLS cert from ACM, us-east-1)
   │
   ▼
API Gateway (HTTP API)
   │  VPC Link (private integration)
   ▼
Internal ALB ──► ECS Fargate task (private subnet, VPC-A)
                      │ outbound via NAT Gateway (image pulls)
                      │
              Transit Gateway
                      │
                 VPC-B (shared services)
                  ├── Route 53 Resolver inbound endpoint
                  └── test EC2 instance

ALB access logs ──► S3 ──► Athena (SQL over logs)
```

**Cost warning:** NAT Gateway (~$0.045/hr), Transit Gateway (~$0.05/hr per VPC attachment → ~$0.10/hr for two), Route 53 Resolver inbound endpoint (~$0.125/hr per ENI, minimum 2 ENIs → ~$0.25/hr), ALB (~$0.02/hr), Fargate (pennies for one tiny task). A focused build-and-teardown session costs roughly **$2–5 total**. The teardown section at the end is mandatory — do not leave this running overnight.

**Region:** Use **us-east-1 (N. Virginia)** for everything. Reason: CloudFront requires its ACM certificate to be in us-east-1 no matter where the rest of your stack lives, so putting everything there removes one source of confusion. (In real jobs you'd pick the region close to users/data and only the *cert* goes to us-east-1 — that distinction is itself an interview question.)

**How to use this guide:** every step has a **Concept** or **Why** note. Read those even when the clicking feels obvious — they are the interview answers.

---

## Part 0 — Prerequisites (15–20 min)

### 0.1 AWS account and IAM user

1. If you don't have an AWS account, create one at aws.amazon.com (needs a credit card; this project costs a few dollars).
2. **Do not work as the root user.** Console → **IAM → Users → Create user** → name `sumit-admin` → check "Provide user access to the AWS Management Console" → set a password.
3. On the permissions step, choose **Attach policies directly** → attach `AdministratorAccess` (fine for a personal lab; in a company you'd use least-privilege roles via IAM Identity Center).
4. Sign out of root, sign back in as this IAM user.

> **Concept — why not root?** Root can do irreversible things (close the account, change billing) and cannot be restricted by IAM policies. Real environments lock root away with MFA and no access keys, and use IAM roles for daily work. Interview phrasing: "root is break-glass only."

### 0.2 Billing alarm (2 min — do not skip)

**Billing and Cost Management → Budgets → Create budget** → use the **Zero spend budget** template or a $10 monthly cost budget with an email alert.

> **Why:** the #1 way self-learners get burned is a forgotten NAT Gateway, TGW, or Resolver endpoint. All three bill per hour whether or not traffic flows.

### 0.3 Your terminal: CloudShell

You'll need a shell with `curl` and `dig` for testing. Use **AWS CloudShell** — the `>_` icon in the console's top bar. Free, browser-based, AWS CLI pre-installed.

### 0.4 (Strongly recommended) A domain name

Phase 4's custom domain + ACM certificate needs a domain you control:

- Cheapest path: **Route 53 → Registered domains → Register domain** — `.click` / `.link` domains cost ~$3–5/yr, and registering in Route 53 auto-creates the **public hosted zone** for you.
- Already own a domain elsewhere (GoDaddy/Namecheap)? Create a Route 53 public hosted zone for it and update the **NS records** at your registrar to the four Route 53 name servers. Propagation can take hours — kick this off *now* if you're going this route.
- No domain? You can still build everything; you'll use the default `dxxxx.cloudfront.net` URL and skip only the custom-domain steps. Still read Phase 4's concepts — they're heavily asked.

---

## Part 1 — Networking Foundation: VPC, Subnets, IGW, NAT, Route Tables, Transit Gateway (~60 min)

### Concepts first — read before clicking

**VPC (Virtual Private Cloud):** your own logically isolated slice of the AWS network. You pick a private IP range (a CIDR block like `10.0.0.0/16`); nothing enters or leaves unless you create an explicit path.

**CIDR notation:** `10.0.0.0/16` = "first 16 bits fixed" → 10.0.0.0–10.0.255.255 → 65,536 addresses. A `/24` subnet has 256 addresses, but **AWS reserves 5 per subnet** (network address, VPC router, DNS, future use, broadcast) → **251 usable**. Interview trap: "usable IPs in a /24 in AWS?" → 251, not 254.

**Subnet:** a slice of the VPC CIDR living in exactly **one Availability Zone**. A subnet is not inherently public or private — its **route table** decides:
- **Public subnet** = route table contains `0.0.0.0/0 → Internet Gateway`.
- **Private subnet** = no route to the IGW. For *outbound-only* internet (image pulls, package updates) it routes `0.0.0.0/0 → NAT Gateway`.

**Internet Gateway (IGW):** horizontally scaled, highly available VPC component enabling two-way internet traffic; performs 1:1 NAT between an instance's private IP and its public IP. One per VPC.

**NAT Gateway:** lets private-subnet resources *initiate* outbound internet connections while blocking inbound connections from outside. Facts interviewers probe:
- Lives in a **public subnet** (it reaches the internet through the IGW on your behalf) and needs an **Elastic IP**.
- It is **zonal**. Production HA = one NAT GW per AZ, with each AZ's private route table pointing at its local NAT GW (avoids cross-AZ data charges and a single point of failure).
- NAT **Gateway** vs NAT **instance**: gateway is managed, scales to ~100 Gbps, no security groups attach to it; instance is a self-managed EC2 box (legacy; requires disabling source/dest check).

**Route table:** rules of "destination CIDR → target". Every subnet is associated with exactly one route table (explicitly, or implicitly with the VPC's *main* route table). Matching is **longest-prefix**: `10.1.0.0/16 → TGW` beats `0.0.0.0/0 → NAT` for traffic to 10.1.x.x. Every route table has an un-deletable `local` route for the VPC CIDR — that's why subnets in a VPC can always talk to each other.

**Transit Gateway (TGW):** a regional, highly available cloud **router**. VPCs, VPNs, and Direct Connect *attach* to it; it routes between them via its own **TGW route tables**. Why it exists:
- VPC **peering** is 1-to-1 and **non-transitive** (A↔B, B↔C peered ⇒ A still cannot reach C). 10 VPCs full-mesh = 45 peering connections. TGW replaces the mesh with **hub-and-spoke** — this is how every large pharma/enterprise network is built.
- **Two routing layers must both be correct:** (1) each VPC subnet route table must send the remote CIDR → TGW; (2) the TGW route table must map each CIDR → the right attachment (by default, attachments auto-*propagate* their VPC CIDR into it).
- **Association** = which TGW route table an attachment consults; **propagation** = which TGW route table an attachment advertises its routes into. Enterprises disable the defaults and build separate TGW route tables (prod / dev / shared) for segmentation — a favorite senior-level topic.
- An attachment places an **ENI in each subnet/AZ you select**; traffic from an AZ with no attachment ENI cannot use the TGW.
- Classic troubleshooting question: *"VPC-A can't reach VPC-B via TGW — what do you check?"* → VPC-A subnet route table → VPC-B subnet route table (**return path!**) → TGW route table → security groups → NACLs.

**Security Groups vs NACLs** (you'll hit these constantly):
- **Security group:** *stateful* firewall on an ENI (instance/task/ALB). Stateful = allowed inbound automatically allows the response outbound. Allow rules only. SGs can reference other SGs as source — the key pattern used later (ALB SG → task SG).
- **NACL:** *stateless* firewall at the subnet boundary; must allow both directions, including **ephemeral return ports 1024–65535**. Allow + deny rules, evaluated by rule number. The default NACL allows everything — leave it alone here, but know the ephemeral-port answer.

### 1.1 Create VPC-A (the app VPC)

1. Console → search **VPC** → **Create VPC** → choose **"VPC and more"** (the wizard builds subnets, route tables, IGW and NAT in one shot — but you must understand what it built).
2. Settings:
   - Name tag auto-generation: `vpc-a-app`
   - IPv4 CIDR: `10.0.0.0/16`
   - Availability Zones: **2**
   - Public subnets: **2**, Private subnets: **2**
   - NAT gateways: **In 1 AZ** (lab economics; production = 1 per AZ)
   - VPC endpoints: **None** (we *want* traffic through NAT so you can break it in Part 6; in a real GxP shop you'd add S3/ECR interface endpoints to keep traffic private and cut NAT cost — say exactly that in interviews)
   - DNS hostnames and DNS resolution: **both enabled** (needed later for private hosted zones and ECS)
3. Create, and study the wizard's preview diagram — it shows which route table points where.
4. **Verify (this is the learning):** VPC console → **Route tables**, filter by `vpc-a-app`:
   - Public route table: `10.0.0.0/16 → local`, `0.0.0.0/0 → igw-…`
   - Private route table(s): `10.0.0.0/16 → local`, `0.0.0.0/0 → nat-…`
   - Check the **Subnet associations** tab on each.

### 1.2 Create VPC-B (shared services)

Repeat with:
- Name: `vpc-b-shared`
- CIDR: `10.1.0.0/16` — **must not overlap with VPC-A.** TGW cannot route between overlapping CIDRs. "How do you plan CIDRs org-wide?" is a real interview question (answer: central IP address management, non-overlapping ranges per VPC/environment/region).
- AZs: 2, Public: 2, Private: 2, **NAT gateways: None** (no egress needed; saves money), DNS options enabled.

### 1.3 Create the Transit Gateway and attach both VPCs

1. VPC console → **Transit gateways → Create transit gateway** → name `tgw-lab`, keep defaults (**default route table association: enable, default propagation: enable** — one shared TGW route table, everything reaches everything; fine for a lab, see the segmentation concept above for what production does).
2. Wait ~2 min for state **Available**.
3. **Transit gateway attachments → Create transit gateway attachment**:
   - Name `tgw-att-vpc-a`, TGW `tgw-lab`, type **VPC**, VPC `vpc-a-app`, subnets: **one private subnet in each AZ**.
   - Repeat as `tgw-att-vpc-b` for `vpc-b-shared`, one private subnet per AZ.
   > **Why these subnets:** the attachment drops an ENI into each selected subnet — those ENIs are the TGW on/off-ramps per AZ. Best practice is dedicated /28 subnets for TGW ENIs; private subnets are fine here.
4. When both attachments are **Available**: **Transit gateway route tables** → the default table → **Routes** tab → you should see `10.0.0.0/16` and `10.1.0.0/16`, each *propagated* from its attachment. TGW side done.

### 1.4 Add the VPC-side routes (the step everyone forgets)

The TGW knows both VPCs; the VPCs don't yet know to send traffic *to* the TGW.

1. **Route tables** → each **private** route table of `vpc-a-app` → **Edit routes → Add route**: destination `10.1.0.0/16`, target **Transit Gateway → tgw-lab**.
2. Each **private** route table of `vpc-b-shared`: add `10.0.0.0/16 → tgw-lab`.
3. Also add the same TGW route to VPC-B's **public** route table (our VPC-B test instance will sit in a public subnet for SSM reasons — see below — and it still needs a return path to 10.0.0.0/16).

> **Interview gold:** if you add the route on only one side, packets arrive but the **return path** is missing → asymmetric routing → timeouts that look like "it's just slow". Always verify routing in both directions.

### 1.5 Test connectivity across the TGW

1. **Create an SSM role once:** IAM → Roles → Create role → AWS service → EC2 → attach `AmazonSSMManagedInstanceCore` → name `ec2-ssm-role`.
   > **Concept — Session Manager:** browser shell into instances with **no SSH keys, no port 22, no public IP required** (if the instance can reach SSM endpoints). This is how regulated (GxP) shops grant access, with full session audit logs. Mention it in interviews.
2. Launch `test-vpc-b`: **EC2 → Launch instance** → Amazon Linux 2023, `t3.micro`:
   - Network: `vpc-b-shared`, Subnet: a **public** subnet, Auto-assign public IP: **enabled**.
   - Security group `sg-test-b`: allow **All ICMP – IPv4** from `10.0.0.0/16` and **HTTP 80** from `10.0.0.0/16` and `10.1.0.0/16`.
   - Advanced details → IAM instance profile: `ec2-ssm-role`.
   > **Why public subnet here?** SSM needs the instance to reach AWS's SSM endpoints. VPC-B has no NAT. The clean fix is VPC interface endpoints for `ssm`, `ssmmessages`, `ec2messages` (know this for interviews); the fast lab fix is a public subnet + public IP. The TGW test is unaffected — you'll ping its *private* IP.
3. Launch `test-vpc-a` the same way but in a **private** subnet of `vpc-a-app`, no public IP, SG allowing ICMP from `10.1.0.0/16`, same SSM role. (It reaches SSM via the NAT Gateway — notice the asymmetry with VPC-B and make sure you can explain it.)
4. On `test-vpc-b`, start a web server for later tests: connect via **Session Manager** and run:
   ```bash
   sudo dnf install -y nginx && sudo systemctl enable --now nginx
   echo "hello from vpc-b" | sudo tee /usr/share/nginx/html/index.html
   ```
5. Connect to `test-vpc-a` via Session Manager and test:
   ```bash
   ping -c 3 <private-IP-of-test-vpc-b>      # ICMP across TGW
   curl http://<private-IP-of-test-vpc-b>/   # HTTP across TGW → "hello from vpc-b"
   ```
6. If it fails, debug in this exact order (memorize it): VPC-A route table → VPC-B route table → TGW route table → SG on target → NACLs → does the AZ have a TGW attachment ENI?

**Checkpoint:** you can now explain and *demonstrate* VPC/subnet/IGW/NAT design, longest-prefix routing, TGW attachments, propagation vs association, and asymmetric routing.

---

## Part 2 — DNS: Route 53 Private Hosted Zone + Resolver Inbound Endpoint (~30 min)

### Concepts first

**Route 53** is AWS's DNS service, and it wears three hats: domain registrar, **public hosted zones** (answer DNS queries from the internet), and **private hosted zones** (answer DNS queries only from inside VPCs you associate).

**Private hosted zone (PHZ):** a DNS zone like `internal.sumitlab.local` that resolves *only* inside associated VPCs. One PHZ can be associated with many VPCs — that's how you get consistent internal names across an entire multi-VPC network.

**The Amazon-provided resolver (".2 resolver"):** every VPC has a built-in DNS server at **VPC CIDR base + 2** (for 10.0.0.0/16 → `10.0.0.2`), also reachable at the link-local address `169.254.169.253`. Every instance uses it by default. It resolves: private hosted zones, AWS internal names (like `*.compute.internal` and VPC endpoint DNS), and forwards everything else to public DNS. Critically, **it only accepts queries from inside its own VPC** — an on-prem datacenter cannot query it. That limitation is exactly why Resolver endpoints exist.

**Route 53 Resolver endpoints:**
- **Inbound endpoint:** ENIs with IP addresses inside your VPC that *accept DNS queries from outside the VPC* (on-prem, or peered/TGW-connected networks) and answer using the VPC's resolver — meaning on-prem servers can resolve your private hosted zones and private AWS names. On-prem DNS (e.g., corporate Active Directory DNS) gets a *conditional forwarder*: "for `*.internal.sumitlab.local`, ask 10.0.x.x".
- **Outbound endpoint + resolver rules:** the reverse direction — the VPC resolver forwards queries for on-prem domains (e.g., `corp.company.com`) out to on-prem DNS servers.
- Hybrid DNS = inbound + outbound together. In a pharma company with on-prem AD and data centers, this is guaranteed to exist, which is why it's worth your lab time.

**DNS records you'll use:** `A` (name → IPv4), `CNAME` (name → another name; not allowed at a zone apex), and Route 53's **Alias** record (AWS-proprietary: name → an AWS resource like CloudFront/ALB; works at the apex, free queries, auto-tracks the target's IPs). "Alias vs CNAME" is a top-5 Route 53 interview question.

### 2.1 Create the private hosted zone

1. **Route 53 → Hosted zones → Create hosted zone**:
   - Domain name: `internal.sumitlab.local`
   - Type: **Private hosted zone**
   - Associate VPC: region us-east-1 → `vpc-a-app`. Add another association for `vpc-b-shared`.
2. Inside the zone, **Create record**:
   - Name: `web`, Type: **A**, Value: the **private IP** of `test-vpc-b`, TTL 60.

### 2.2 Test private DNS resolution from the other VPC

On `test-vpc-a` (Session Manager):

```bash
dig +short web.internal.sumitlab.local     # → 10.1.x.x
curl http://web.internal.sumitlab.local/   # → "hello from vpc-b"
```

> **What just happened:** the instance asked its VPC's .2 resolver → the resolver saw the PHZ association → answered with the private IP → the packet then rode your TGW routes. DNS told it *where*; routing carried it *there*. Keep those two layers separate in your head — half of all "DNS issues" are actually routing issues and vice versa. `dig` working but `curl` timing out = DNS fine, routing/SG problem. `dig` failing = DNS problem (PHZ association? VPC DNS settings enabled?).

### 2.3 Create a Resolver inbound endpoint (simulating hybrid DNS)

1. **Route 53 → Resolver → Inbound endpoints → Create inbound endpoint**:
   - Name `inbound-lab`, VPC `vpc-b-shared`, security group: create/pick one allowing **TCP 53 and UDP 53** from `10.0.0.0/16`.
   - Choose two subnets (different AZs) in VPC-B; let AWS pick the IPs. (Two ENIs is the minimum — that's for HA, and it's why this costs ~$0.25/hr. Create it, test it, note the IPs, and feel free to delete it early.)
2. Wait until **Operational**, note the two ENI IP addresses.

### 2.4 Test it like an on-prem server would

Pretend `test-vpc-a` is an on-prem server: instead of using its own VPC resolver, query the inbound endpoint **directly**:

```bash
dig @10.1.x.x web.internal.sumitlab.local   # @ = "send the query to this specific DNS server"
```

You get the private answer, from *outside* the endpoint's VPC — which is exactly what an on-prem DNS conditional forwarder would do across a VPN/Direct Connect (that also rides through a TGW in real designs, so your lab topology mirrors reality).

> **Interview articulation:** "On-prem resolvers can't query a VPC's .2 resolver directly, so we deploy Route 53 Resolver inbound endpoints and point on-prem conditional forwarders at their ENI IPs; outbound endpoints plus resolver rules handle the reverse direction for on-prem domains."

**Cost tip:** once this test works, you can delete the inbound endpoint immediately — nothing later depends on it.

---

## Part 3 — The API: ECR, ECS Fargate, Internal ALB, API Gateway VPC Link (~60–75 min)

### Concepts first

**ECS (Elastic Container Service):** AWS's container orchestrator. Core objects:
- **Cluster:** a logical grouping of capacity.
- **Task definition:** the versioned "recipe" — image, CPU/memory, ports, environment, IAM roles, log config. Every edit creates a new **revision** (immutable — the same philosophy as Domino's versioned Compute Environments, a parallel worth drawing in your interview).
- **Task:** one running copy of a task definition (≈ a Kubernetes pod).
- **Service:** keeps N tasks running, replaces failed ones, integrates with load balancers, does rolling deployments (≈ a k8s Deployment + Service glued together).
- **Launch types:** **EC2** (you manage container instances) vs **Fargate** (serverless — AWS runs the task on invisible infrastructure; you pay per vCPU/GB-second). We use Fargate.
- **Two IAM roles per task — a classic interview question:**
  - **Task *execution* role:** used by the ECS agent *to start* the task — pull the image from ECR, write logs to CloudWatch.
  - **Task role:** used by *your application code* to call AWS APIs (S3, DynamoDB…). Confusing these is the most common ECS IAM bug.
- **`awsvpc` network mode** (mandatory on Fargate): each task gets its **own ENI, its own private IP, and its own security group** — tasks are first-class network citizens, exactly like k8s pods with VPC-native CNI (say that in the interview; it maps ECS to your k8s knowledge).

**Why our task can pull its image at all:** the task runs in a *private* subnet; the image lives in ECR (reached over the internet or endpoints). Pull path: task ENI → private route table → **NAT Gateway** → IGW → ECR. Remove any link and you get the single most famous Fargate error: `CannotPullContainerError / i/o timeout`. You will deliberately cause it in Part 6.

**ALB (Application Load Balancer):** Layer-7 load balancer. **Internal** ALB = private IPs only, resolvable/reachable only within the VPC (and connected networks). Components: **listener** (port 80) → **target group** (where health checks live). With Fargate the target type is **ip** (tasks are ENIs/IPs, not instances).

**API Gateway (HTTP API) + VPC Link:** API Gateway is a managed front door (routing, throttling, auth). Its control plane lives *outside* your VPC — so to reach an **internal** ALB it needs a **VPC Link**: ENIs that API Gateway places inside your subnets to reach private resources. This "private integration" pattern (public managed edge → private backend) is the standard way enterprises expose internal services without making anything in the VPC public. Note: HTTP APIs are the cheaper, simpler flavor; REST APIs add usage plans/API keys/request validation — know that one-liner difference.

### 3.1 Push a container image to ECR

1. **ECR → Repositories → Create repository** → name `hello-api` (private).
2. Open **CloudShell**. CloudShell can't run Docker, so we'll do the honest lab shortcut — pull a public image and re-tag it… which *also* doesn't work without Docker. So use the simplest real path: build on your `test-vpc-a` instance? It's in a private subnet with NAT — Docker works there. Session Manager into `test-vpc-a`:

   ```bash
   sudo dnf install -y docker && sudo systemctl enable --now docker

   # A tiny API: nginx that returns JSON
   mkdir ~/api && cd ~/api
   cat > default.conf << 'EOF'
   server {
     listen 8080;
     location / { default_type application/json; return 200 '{"service":"hello-api","status":"ok"}'; }
     location /health { return 200 'healthy'; }
   }
   EOF
   cat > Dockerfile << 'EOF'
   FROM public.ecr.aws/nginx/nginx:latest
   COPY default.conf /etc/nginx/conf.d/default.conf
   EXPOSE 8080
   EOF
   sudo docker build -t hello-api .
   ```
3. Give the instance permission to push: **IAM → Roles → `ec2-ssm-role` → Add permissions** → attach `AmazonEC2ContainerRegistryPowerUser`.
4. Push (replace `ACCOUNT_ID`; find it in the console's account menu):
   ```bash
   AWS_REGION=us-east-1; ACCOUNT_ID=<your-account-id>
   aws ecr get-login-password --region $AWS_REGION | sudo docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
   sudo docker tag hello-api:latest $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/hello-api:v1
   sudo docker push $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/hello-api:v1
   ```
   > **Concept:** `get-login-password` exchanges your IAM credentials for a 12-hour Docker registry token — ECR auth is IAM auth. Tag `v1`, not `latest`: immutable, explicit versions are what change management (and GxP validation) demand.

### 3.2 Create the security groups (do these first — order matters)

**EC2 → Security groups → Create**, both in `vpc-a-app`:

1. `sg-alb-internal`: inbound **HTTP 80** from `10.0.0.0/16` (VPC Link ENIs and any VPC client).
2. `sg-ecs-task`: inbound **TCP 8080** with **source = `sg-alb-internal`** (the SG itself, not a CIDR).

> **Concept — SG chaining:** "only the ALB may talk to tasks" expressed by *reference*, not by IP. Tasks come and go with random IPs; the rule never changes. This pattern (SG references SG) is the answer to "how do you secure traffic between tiers?"

### 3.3 Create the internal ALB + target group

1. **EC2 → Load balancers → Create → Application Load Balancer**:
   - Name `alb-internal-api`, Scheme: **Internal**, VPC `vpc-a-app`, select the **two private subnets**, SG `sg-alb-internal`.
2. Under Listeners, create a target group inline: **Target type: IP** (Fargate!), name `tg-hello-api`, protocol HTTP, **port 8080**, health check path `/health`. Register no targets (ECS will do it). Listener: HTTP **80** → forward to `tg-hello-api`.

> **Why "IP" target type:** Fargate tasks are ENIs with IPs — there's no instance to register. The ECS service will register/deregister task IPs into the target group automatically as tasks start and stop.

### 3.4 Create the ECS cluster, task definition, and service

1. **ECS → Clusters → Create cluster** → name `lab-cluster`, infrastructure: **Fargate only**.
2. **Task definitions → Create new task definition**:
   - Family `hello-api`, Launch type Fargate, CPU **0.25 vCPU**, memory **0.5 GB**.
   - **Task execution role:** let the console create `ecsTaskExecutionRole` (it grants ECR pull + CloudWatch Logs).
   - Container: name `hello-api`, image `ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/hello-api:v1`, container port **8080**, log collection: CloudWatch (default `awslogs` config is fine).
3. **Clusters → lab-cluster → Services → Create**:
   - Launch type Fargate, task definition `hello-api` (latest revision), service name `hello-api-svc`, desired tasks **2** (so you see load balancing across AZs).
   - Networking: VPC `vpc-a-app`, the **two private subnets**, SG `sg-ecs-task`, **Public IP: OFF** (private subnet — a public IP would be useless and is a red flag; with public IP off, the image pull *must* go via NAT, which is the point).
   - Load balancing: Application Load Balancer → `alb-internal-api` → existing listener 80 → existing target group `tg-hello-api`.
4. Watch **Tasks** go `PROVISIONING → PENDING → RUNNING`, then **EC2 → Target groups → tg-hello-api → Targets** until both show **healthy**.
5. Test from inside the VPC — on `test-vpc-a`:
   ```bash
   curl http://<ALB-DNS-name>/        # {"service":"hello-api","status":"ok"}
   curl http://<ALB-DNS-name>/health  # healthy
   ```
   (ALB DNS name is on the load balancer's details page. Note it resolves to *private* IPs — try `dig` on it.)

**Troubleshooting map (memorize — these are interview questions):**
- Task stuck `PENDING` / `CannotPullContainerError … i/o timeout` → task can't reach ECR: NAT route missing, public IP off with no NAT, or SG blocking egress.
- `CannotPullContainerError … 403/denied` → execution role missing ECR permissions, or wrong image URI/tag.
- Task RUNNING but target **unhealthy** → health check path/port wrong, app listening on a different port than the container port, or `sg-ecs-task` doesn't allow 8080 *from the ALB SG*.
- Tasks start then get killed repeatedly → failing health checks → ECS replaces them (check the *target group* before blaming ECS).
- App can't call AWS APIs → you put permissions on the execution role instead of the **task role**.

### 3.5 Put API Gateway in front via VPC Link

1. **API Gateway → VPC links → Create** (choose the **HTTP APIs** flavor): name `vpclink-lab`, VPC `vpc-a-app`, the two private subnets, SG `sg-alb-internal` (or a dedicated SG; it just needs outbound to the ALB).
2. Wait for **Available** (a few minutes — it's provisioning ENIs into your subnets; peek at **EC2 → Network interfaces** and you'll literally see them).
3. **API Gateway → APIs → Build HTTP API**: name `hello-http-api` → skip integrations for now → create.
4. **Routes → Create**: method `ANY`, path `/{proxy+}` (greedy proxy: forward every path).
5. **Integrations → Manage integrations → Create**: type **Private resource** → target the **ALB listener** (select `alb-internal-api`, listener 80) → via VPC link `vpclink-lab`. Attach the integration to the `ANY /{proxy+}` route.
6. Under **Stages**, the `$default` stage with auto-deploy is created for you. Copy the **Invoke URL** and test *from your own laptop browser or CloudShell*:
   ```bash
   curl https://<api-id>.execute-api.us-east-1.amazonaws.com/
   ```

**Pause and appreciate what just happened:** a container in a private subnet with no public IP, behind an internal ALB that public DNS can't even reach, is now serving the public internet — because API Gateway tunnels into your VPC through VPC Link ENIs. You can now explain "how do you expose a private service publicly without making it public" — a guaranteed interview question.

---

## Part 4 — The Edge: ACM Certificate, CloudFront, Route 53 Public DNS (~30–40 min)

### Concepts first

**ACM (AWS Certificate Manager):** issues free, auto-renewing public TLS certificates for use on AWS-managed endpoints (CloudFront, ALB, API Gateway). You **cannot export** the private key of an ACM public cert — it only lives on AWS services (that's the trade for free + auto-renewal). Validation proves you control the domain:
- **DNS validation** (use this): ACM gives you a CNAME record to create; if your zone is in Route 53 there's a one-click button. Because the record stays in place, **renewals are automatic forever** — that's why DNS validation is best practice.
- **Email validation**: legacy; renewal requires a human clicking an email. Say "DNS validation for hands-off renewal" in interviews.
- **The us-east-1 rule:** CloudFront is a *global* service whose control plane lives in us-east-1, so **a cert attached to CloudFront must be requested in us-east-1** — even if your app runs in Mumbai. A cert for an ALB, by contrast, must be in the ALB's own region. Top-tier interview trivia that trips people constantly.

**CloudFront:** AWS's CDN — 400+ edge locations worldwide. Users connect to the nearest edge; the edge serves cached content or fetches from your **origin** (S3, ALB, API Gateway, anything HTTP). Key vocabulary:
- **Origin**: where content comes from. **Distribution**: your CloudFront configuration. **Behavior**: per-path rules (cache policy, allowed methods, origin).
- **Why put CloudFront in front of an API** (dynamic, barely cacheable)? (1) TLS terminates at the nearest edge and rides AWS's backbone to the origin — lower global latency; (2) AWS WAF and AWS Shield DDoS protection attach at the edge; (3) one domain can front many origins by path (`/api/*` → API GW, `/*` → S3 static site) — the classic full-stack pattern; (4) caching for whatever *is* cacheable.
- **Cache key** concept: by default CloudFront ignores headers/cookies/query strings when caching — for an API you typically use the managed **CachingDisabled** policy plus **AllViewer** origin-request policy so requests pass through intact.

**Route 53 Alias vs CNAME (asked in almost every AWS interview):** a CNAME can't exist at the zone apex (`sumitlab.click` itself) and costs a query resolution hop; an **Alias** is Route 53-internal, works at the apex, is free, and auto-follows AWS resources' changing IPs. Alias targets: CloudFront, ALB/NLB, API Gateway, S3 websites, etc.

### 4.1 Request the certificate (in us-east-1)

Skip 4.1–4.3's custom-domain parts if you have no domain — create the distribution with the default certificate instead.

1. Confirm the region selector says **N. Virginia (us-east-1)**.
2. **AWS Certificate Manager → Request certificate → Request a public certificate**:
   - Domain names: `api.sumitlab.click` (use your actual domain).
   - Validation: **DNS validation**.
3. Open the certificate → click **Create records in Route 53** (the one-click button). Wait a few minutes until status = **Issued**.
   > If it hangs in *Pending validation* for >30 min: your domain's NS records don't point at the Route 53 zone (registrar-side problem), or the CNAME wasn't created. `dig NS sumitlab.click` and compare with the hosted zone's NS record.

### 4.2 Create the CloudFront distribution

1. **CloudFront → Create distribution**:
   - Origin domain: paste your API's invoke domain `<api-id>.execute-api.us-east-1.amazonaws.com` (do **not** pick anything from the dropdown; and don't include `https://` or a path).
   - Protocol: HTTPS only.
   - Default cache behavior: Viewer protocol policy **Redirect HTTP to HTTPS**; Allowed methods **GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE**; Cache policy **CachingDisabled**; Origin request policy **AllViewer**.
     > **Why AllViewer + CachingDisabled:** APIs need the full request forwarded (headers, query strings, auth) and usually must not be cached. Caching an authenticated API response and serving it to another user is a real-world incident class — say that sentence in an interview and you sound senior.
   - Web Application Firewall: **Do not enable** (costs money; but *know* that WAF attaches here).
   - Alternate domain name (CNAME): `api.sumitlab.click`; Custom SSL certificate: pick your ACM cert (it only appears if it's in us-east-1 and Issued).
2. Create, wait for **Deployed** (5–15 min — it's pushing config to hundreds of edge locations).
3. Test with the default domain first:
   ```bash
   curl https://dxxxxxxxx.cloudfront.net/
   ```

### 4.3 Point your domain at CloudFront

1. **Route 53 → Hosted zones → your public zone → Create record**:
   - Name `api`, type **A**, **Alias: ON** → Alias to CloudFront distribution → pick yours.
2. Wait a minute, then:
   ```bash
   dig +short api.sumitlab.click     # CloudFront edge IPs
   curl https://api.sumitlab.click/  # {"service":"hello-api","status":"ok"}
   ```

**Trace the full request path out loud — this is your interview story:** browser → Route 53 alias → nearest CloudFront edge (TLS via ACM cert) → API Gateway → VPC Link ENI → internal ALB → SG-gated Fargate task ENI in a private subnet → response back out. Every hop is a service you configured and can troubleshoot.

---

## Part 5 — Athena: SQL Over Your Access Logs (~20–30 min)

### Concepts first

**Athena** is serverless, interactive SQL (based on Trino/Presto) directly over files in **S3**. Nothing to provision; you pay **per TB of data scanned** (~$5/TB).
- **Schema-on-read:** unlike a database (schema-on-write), the data just sits in S3 as text/Parquet; the table definition (in the **Glue Data Catalog**) is a lens applied at query time. Wrong schema? Redefine the table; the data never moves.
- **Cost/perf levers (the interview answer):** **partition** the data (e.g., by date) so queries scan only relevant prefixes; use **columnar formats** (Parquet/ORC) + compression; `SELECT` only needed columns; use `LIMIT` while exploring.
- Athena always needs a **query results location** in S3 (it writes every result set there).

### 5.1 Enable ALB access logs

1. **S3 → Create bucket** → name `sumit-lab-alb-logs-<accountid>` (bucket names are globally unique), region us-east-1, defaults.
2. **EC2 → Load balancers → alb-internal-api → Attributes → Edit** → enable **Access logs** → the bucket above. The console adds the required **bucket policy** allowing the regional ELB log-delivery account to write — open the bucket's Permissions tab and *read that policy*; "service X can't write to my bucket" is an everyday bucket-policy troubleshooting scenario.
3. Generate traffic so there's something to analyze:
   ```bash
   for i in $(seq 1 60); do curl -s https://api.sumitlab.click/ > /dev/null; curl -s https://api.sumitlab.click/nope > /dev/null; done
   ```
   Logs flush to S3 in ~5-minute batches — check the bucket for `.log.gz` objects under `AWSLogs/<account>/elasticloadbalancing/...` before continuing.

### 5.2 Create the Athena table and query

1. **Athena → Query editor**. First run prompts for a query result location → set `s3://sumit-lab-alb-logs-<accountid>/athena-results/`.
2. AWS documentation ships ready-made DDL for ALB logs ("Querying Application Load Balancer logs" page). It's a `CREATE EXTERNAL TABLE alb_logs (...)` statement with a `ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.RegexSerDe'` and a long regex, ending with `LOCATION 's3://sumit-lab-alb-logs-<accountid>/AWSLogs/<account-id>/elasticloadbalancing/us-east-1/'`. Copy it from the docs, fix the LOCATION, run it.
   > **What the DDL means:** `EXTERNAL` = Athena only stores metadata; the data stays (and stays owned) in S3. The SerDe (serializer/deserializer) is the parser — here a regex that slices each raw log line into columns. That's schema-on-read made concrete.
3. Run real queries:
   ```sql
   SELECT elb_status_code, count(*) AS requests
   FROM alb_logs GROUP BY elb_status_code ORDER BY requests DESC;

   SELECT client_ip, count(*) AS hits
   FROM alb_logs GROUP BY client_ip ORDER BY hits DESC LIMIT 10;

   SELECT request_url, avg(target_processing_time) AS avg_latency
   FROM alb_logs GROUP BY request_url ORDER BY avg_latency DESC LIMIT 10;
   ```
   Notice the 404s from your `/nope` requests, and that the "client IPs" are the VPC Link ENIs' private IPs — a nice teachable artifact of the private-integration architecture.
4. Look at the **Data scanned** figure under each query and connect it to the pricing model. Then say the optimization sentence: *"in production I'd partition by day and convert to Parquet to cut scan costs by ~90%."*

---

## Part 6 — Break It On Purpose (~30 min) — where the real interview prep is

For each drill: break → observe the *exact* symptom → fix → verify. The symptoms are what interviewers describe when they ask troubleshooting questions; you'll recognize them instead of guessing.

**Drill 1 — Kill the NAT path.**
VPC-A private route table → delete the `0.0.0.0/0 → nat-…` route. Then force a new task: ECS → service → **Update → Force new deployment**.
*Symptom:* new tasks stuck in PENDING, then stopped with `CannotPullContainerError: … i/o timeout` (events tab of the service). Existing healthy tasks keep serving — established flows don't need new pulls.
*Lesson:* image pulls need an egress path; and this is why prod uses ECR/S3 VPC endpoints (no internet dependency at all).
*Fix:* re-add the route, force deployment again.

**Drill 2 — Blackhole the TGW (one direction).**
Delete `10.0.0.0/16 → tgw` from VPC-B's route tables only. From `test-vpc-a`, ping/curl `test-vpc-b`.
*Symptom:* timeout — not "connection refused". The request *arrives* (VPC-A's route is fine); the *reply* has no route home. Timeouts = routing/filtering; refused = the service itself.
*Fix:* re-add.

**Drill 3 — Break private DNS.**
Route 53 → your PHZ → remove the VPC-A association. On `test-vpc-a`: `dig web.internal.sumitlab.local`.
*Symptom:* NXDOMAIN — but `curl http://10.1.x.x/` still works. Cleanly separates "DNS broken" from "network broken".
*Fix:* re-associate (takes a minute to take effect).

**Drill 4 — Break the SG chain.**
Edit `sg-ecs-task`: change the source of the 8080 rule from `sg-alb-internal` to some random SG.
*Symptom:* targets drain to **unhealthy** in the target group; ALB returns 502/504; ECS eventually kills and replaces tasks (which also go unhealthy — a replacement loop). Task logs show *nothing* because packets never arrive — "no log entries at all" is itself a diagnostic clue pointing at network/SG, not the app.
*Fix:* restore the SG reference.

**Drill 5 — Health check mismatch.**
Target group → Health checks → change the path to `/wrong`. Watch targets go unhealthy while `curl` *through the ALB* fails yet the app is perfectly fine.
*Lesson:* "the app is down" vs "the health check is wrong" look identical from outside — always check what the health checker is actually probing.
*Fix:* restore `/health`.

---

## Part 7 — Teardown (do this immediately — order matters because of dependencies)

1. **Route 53:** delete the alias record `api.…` and PHZ records, then the private hosted zone. (Keep the public zone if you keep the domain — it's $0.50/mo.)
2. **CloudFront:** *Disable* the distribution, wait for Deployed, then *Delete* (a distribution can't be deleted while enabled).
3. **API Gateway:** delete the HTTP API, then the **VPC Link**.
4. **ECS:** update service desired count to 0 → delete service → delete cluster. Deregister task definitions (optional). **ECR:** delete the repo (images cost pennies but be clean).
5. **ALB:** delete load balancer, then target group.
6. **EC2:** terminate `test-vpc-a` and `test-vpc-b`.
7. **Resolver inbound endpoint:** delete (if you didn't already).
8. **NAT Gateway:** delete (VPC console → NAT gateways). Then **release the Elastic IP** (EC2 → Elastic IPs) — an unattached EIP bills hourly, a classic leak.
9. **Transit Gateway:** delete both attachments, wait, then delete the TGW.
10. **VPCs:** delete `vpc-a-app` and `vpc-b-shared` (the console deletes their subnets/route tables/IGWs; if deletion fails, it names the dependency still holding on — usually a leftover ENI or SG reference, which is itself a good lesson).
11. **Keep (essentially free):** the S3 log bucket + Athena table, so you can revisit queries. Or empty + delete the bucket if you want zero residue.
12. Next day, glance at **Cost Explorer** to confirm spend dropped to ~zero — that habit is real FinOps hygiene.

---

## Part 8 — The Interview Cheat Sheet (rehearse these out loud)

**One-line story:** *"I built a multi-VPC platform: an ECS Fargate API in private subnets behind an internal ALB, exposed publicly via API Gateway VPC Link and CloudFront with an ACM cert and Route 53 alias records; a second shared-services VPC connected over Transit Gateway with a private hosted zone and a Resolver inbound endpoint to simulate hybrid DNS; ALB logs analyzed in Athena. Then I broke each layer deliberately and documented the failure signatures."*

Rapid-fire answers you've now *earned*:
- Usable IPs in a /24? **251** (AWS reserves 5).
- What makes a subnet public? **A route to an IGW in its route table** — nothing else.
- Why must NAT GW be in a public subnet? **It needs the IGW for its own egress**; private resources route 0.0.0.0/0 to it.
- TGW vs peering? **Peering is non-transitive 1:1 (n² mesh); TGW is a hub-and-spoke regional router with its own route tables.**
- Cross-VPC traffic fails via TGW — checklist? **Source VPC RT → destination VPC RT (return path) → TGW RT → SG → NACL → attachment ENI per AZ.**
- Alias vs CNAME? **Alias is Route 53-internal, apex-capable, free, auto-tracks AWS endpoints.**
- Why Resolver endpoints? **The .2 VPC resolver refuses queries from outside the VPC; inbound endpoints give on-prem a DNS door in, outbound + rules forward out.**
- Why is the ACM cert in us-east-1? **CloudFront is global with a us-east-1 control plane; ALB certs live in the ALB's region.**
- ECS execution role vs task role? **Execution = agent (pull image, ship logs); task = the app's AWS API calls.**
- Fargate task can't pull image? **Egress path (NAT/endpoints), public-IP setting, execution-role perms, image URI — in that order.**
- How do you expose a private service publicly? **API Gateway private integration via VPC Link (or CloudFront→public ALB); the service itself never gets a public IP.**
- Athena cost control? **Partitioning, Parquet/ORC, column pruning — pay per TB scanned.**
- Timeout vs connection refused? **Timeout = routing/SG/NACL swallowing packets; refused = reachable host, nothing listening.**

Good luck with the build — the debugging detours you hit along the way will teach you more than the happy path, so when something breaks, resist the urge to tear it down and restart; diagnose it with the checklists above.
