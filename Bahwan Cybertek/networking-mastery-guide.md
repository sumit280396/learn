# Enterprise Cloud Networking: From Zero to Senior-Level Mastery

**VPC · NAT Gateway · Route Tables · Transit Gateway · Route 53 · Resolvers · DNS Cutovers · SSL/TLS · Zscaler · WAF**

---

## How to Read This Guide

One honest note before we start: with 7 years on Linux, AWS, and Kubernetes, you know more of this than you think. You've `curl`-ed services, debugged pods that couldn't reach the internet, and configured kubeconfigs pointing at EKS endpoints — all of that *is* networking. What you're missing is the **organized mental model** and the **enterprise vocabulary** (cutover, TGW attachment, WAF rule scope-down). This guide builds both.

The single mental model that ties everything together:

> **Every network problem is a packet trying to get from A to B and back. At every hop, three questions decide its fate: (1) Where should I go next? (routing) (2) Am I allowed? (security) (3) What name/address am I even looking for? (DNS).**

Every technology in this guide answers one of those three questions. Route tables, TGW, NAT = routing. Security groups, NACLs, WAF, Zscaler = security. Route 53, Resolvers, DNS servers = naming. SSL/TLS = a fourth question layered on top: "Can I trust who answered, and can anyone eavesdrop?"

---

# PART 1: Networking Foundations (The 20% You Must Know Cold)

## 1.1 IP Addresses and CIDR

An IPv4 address is 32 bits, written as four octets: `10.0.1.25`. A **CIDR block** describes a *range* of addresses: `10.0.0.0/16`.

The `/16` means: "the first 16 bits are fixed (the network part), the remaining 16 bits are free (the host part)."

| CIDR | Fixed bits | Usable range | Total IPs |
|------|-----------|--------------|-----------|
| 10.0.0.0/8 | 8 | 10.0.0.0 – 10.255.255.255 | ~16.7M |
| 10.0.0.0/16 | 16 | 10.0.0.0 – 10.0.255.255 | 65,536 |
| 10.0.1.0/24 | 24 | 10.0.1.0 – 10.0.1.255 | 256 |
| 10.0.1.0/28 | 28 | 10.0.1.0 – 10.0.1.15 | 16 |

**Quick math trick:** every time the prefix number goes *down* by 1, the range *doubles*. /24 = 256, /23 = 512, /22 = 1024.

**Private ranges (RFC 1918)** — the only ranges you should use inside VPCs and datacenters:
- `10.0.0.0/8`
- `172.16.0.0/12` (172.16.x.x – 172.31.x.x)
- `192.168.0.0/16`

These are not routable on the public internet. This is *why NAT exists* (Section 2.5).

**AWS reserves 5 IPs in every subnet.** In `10.0.1.0/24`: `.0` (network), `.1` (VPC router), `.2` (DNS resolver — remember this one, it matters hugely later), `.3` (reserved), `.255` (broadcast). So a /24 gives you 251 usable IPs, not 254.

**Senior-level trap question:** *"Can two VPCs both use 10.0.0.0/16?"* Yes — until you try to connect them. Overlapping CIDRs cannot be peered or (cleanly) attached to the same Transit Gateway route table, because a router with a packet for 10.0.5.9 can't know which VPC you meant. This is why enterprises maintain a **central IPAM (IP Address Management)** plan and every new VPC gets a non-overlapping block.

## 1.2 How Routing Decisions Are Made: Longest Prefix Match

A route table is a list of `destination CIDR → target` rules. When a packet arrives, the router picks the rule with the **most specific (longest) prefix** that matches.

Example route table:
```
10.0.0.0/16      → local
10.1.0.0/16      → tgw-abc123
0.0.0.0/0        → nat-xyz789
```

A packet to `10.1.5.5` matches both `10.1.0.0/16` and `0.0.0.0/0`. The `/16` is more specific, so it goes to the Transit Gateway. `0.0.0.0/0` (the **default route**, "everything else") only catches traffic nothing else matched — that's how "internet-bound" traffic is identified.

This one rule — **longest prefix wins** — explains 80% of routing behavior you'll ever debug.

## 1.3 Ports, Protocols, and the Ones That Matter

An IP gets a packet to a *machine*; a port gets it to a *process* on that machine.

| Port | Protocol | What it is |
|------|----------|-----------|
| 22 | TCP | SSH |
| 53 | UDP **and** TCP | DNS (UDP normally; TCP for large responses/zone transfers) |
| 80 | TCP | HTTP |
| 443 | TCP | HTTPS (TLS) |
| 6443 | TCP | Kubernetes API server |
| 3306 / 5432 | TCP | MySQL / PostgreSQL |

**TCP vs UDP:** TCP does a 3-way handshake (SYN → SYN-ACK → ACK), guarantees delivery and order. UDP is fire-and-forget. DNS uses UDP for speed; TLS/HTTP ride on TCP.

**Troubleshooting significance:** if `telnet host 443` (or `nc -zv host 443`) hangs forever, the SYN is being *dropped* (firewall/SG/routing). If it's *refused* instantly, the packet arrived but nothing is listening — the network is fine, the app is the problem. This one distinction separates junior from senior troubleshooting instantly.

## 1.4 The Life of a Request (memorize this sequence)

When a pod/laptop/EC2 runs `curl https://api.example.com`:

1. **DNS**: resolve `api.example.com` → an IP (Sections on DNS/Route 53).
2. **Routing**: consult route table(s) hop by hop to reach that IP (VPC route tables, TGW, NAT).
3. **Security gates**: security groups, NACLs, firewalls, Zscaler, WAF must all permit it.
4. **TCP handshake**: SYN/SYN-ACK/ACK on port 443.
5. **TLS handshake**: certificates verified, encryption keys agreed (SSL/TLS section).
6. **HTTP request/response** flows inside the encrypted tunnel.

Every outage you will ever troubleshoot is a failure at exactly one of these six steps. Senior engineers debug by **bisecting this sequence**: "Does DNS resolve? Yes → routing next. Does TCP connect? No → security or routing. …"

---

# PART 2: Amazon VPC — Your Private Slice of AWS

## 2.1 What a VPC Is and Why It Exists

A **VPC (Virtual Private Cloud)** is a logically isolated virtual network inside an AWS region that *you* define: you pick its IP range (CIDR), carve it into subnets, and control every path in and out.

**Why it exists:** in a physical datacenter you'd buy switches, routers, and firewalls and cable them together. A VPC is that entire datacenter network, software-defined. Nothing can talk to your VPC unless you explicitly create a path (internet gateway, peering, TGW, VPN…). Default posture: **isolated**.

Real-world anchor for you: every EKS cluster you've supported lives inside a VPC. Worker nodes are EC2 instances in its subnets; pod IPs (with the AWS VPC CNI) are real VPC IPs taken from subnet ranges. When a Domino platform "can't pull images from ECR," you were debugging VPC networking whether you called it that or not.

## 2.2 Subnets and Availability Zones

A **subnet** is a slice of the VPC CIDR pinned to **one Availability Zone**. A subnet never spans AZs.

Typical production layout for VPC `10.0.0.0/16`:

```
AZ-a: public  10.0.0.0/24   | private 10.0.10.0/24  | data 10.0.20.0/24
AZ-b: public  10.0.1.0/24   | private 10.0.11.0/24  | data 10.0.21.0/24
AZ-c: public  10.0.2.0/24   | private 10.0.12.0/24  | data 10.0.22.0/24
```

**What makes a subnet "public" or "private"? Nothing intrinsic.** It is purely determined by its route table:

- **Public subnet** = its route table has `0.0.0.0/0 → Internet Gateway`.
- **Private subnet** = no route to an IGW. Internet-bound traffic (if allowed at all) goes `0.0.0.0/0 → NAT Gateway`.

This is a classic interview question and a classic real-world misconfiguration: someone associates the wrong route table and suddenly a "private" subnet is internet-facing.

**What lives where:** load balancers and NAT gateways in public subnets; app servers, EKS nodes, most workloads in private subnets; databases in isolated "data" subnets often with *no* internet route at all.

## 2.3 Route Tables in the VPC

Every subnet is associated with exactly **one** route table (a route table can serve many subnets). Every VPC route table always contains an unremovable **`local` route** for the VPC's own CIDR — that's what lets any subnet reach any other subnet in the same VPC without extra config.

Public route table:
```
10.0.0.0/16  → local
0.0.0.0/0    → igw-0abc
```

Private route table (per-AZ, see NAT section):
```
10.0.0.0/16  → local
10.1.0.0/16  → tgw-0def        (path to other VPCs / on-prem)
0.0.0.0/0    → nat-0aaa        (internet egress via NAT)
```

**Troubleshooting habit:** when "X can't reach Y" in AWS, the first two things a senior engineer opens are (1) the route table of X's subnet, (2) the security groups. In that order if the destination is outside the VPC; reverse order if inside.

## 2.4 Internet Gateway (IGW)

The IGW is a horizontally-scaled, highly available VPC component that does two things: routes traffic between the VPC and the internet, and performs **1:1 NAT between an instance's private IP and its public/Elastic IP**.

For an instance to be reachable from the internet, ALL of these must be true (a great interview checklist):
1. VPC has an IGW attached.
2. Instance's subnet route table has `0.0.0.0/0 → igw`.
3. Instance has a public IP or Elastic IP.
4. Security group allows the inbound port.
5. NACL allows it in **and** the reply out.

Miss any one → unreachable. In practice, almost nothing except load balancers and NAT gateways should be directly internet-reachable.

## 2.5 NAT Gateway — Deep Dive

**The problem it solves:** instances in private subnets have only private IPs (10.x.x.x). Private IPs don't exist on the internet — no internet router can route a reply back to `10.0.11.37`. But those instances still need *outbound* internet: pulling container images, OS patches, calling external APIs (very relevant: EKS nodes pulling from public registries, Domino installers fetching charts).

**What NAT does:** Network Address Translation. The NAT Gateway sits in a **public subnet** with an **Elastic IP**. When a private instance sends a packet to the internet:

1. Instance `10.0.11.37` sends packet to `52.1.2.3:443`; its route table sends `0.0.0.0/0` traffic to the NAT Gateway.
2. NAT rewrites the source: `10.0.11.37:39482` becomes `<NAT-EIP>:61001`, and records the mapping in a **connection tracking table**.
3. Internet server replies to `<NAT-EIP>:61001`; NAT looks up the mapping and rewrites the destination back to `10.0.11.37:39482`.

**Crucial property:** NAT is **one-way**. Outbound connections (and their replies) work; the internet can never *initiate* a connection inward through NAT. This is why NAT is both a routing tool and, effectively, a security posture.

**Key facts a senior engineer knows:**
- NAT GW is **AZ-scoped**. Best practice: one NAT GW per AZ, and each AZ's private route table points at its **local** NAT GW. If you point all AZs at one NAT GW, its AZ failure takes out egress for the whole VPC, and you pay cross-AZ data charges.
- It scales automatically (~up to 100 Gbps) but has a hard limit of **~55,000 simultaneous connections to a single unique destination (IP+port)** per NAT GW — exceeding it causes `ErrorPortAllocation` and mysterious connection failures at scale.
- **Cost:** hourly charge + per-GB processing. A classic enterprise cost bug: workloads pulling terabytes from **S3/ECR through NAT**. Fix: **VPC Gateway Endpoint for S3** (free) and Interface Endpoints for ECR — traffic then bypasses NAT entirely.
- NAT Gateway vs NAT *Instance*: the instance is a legacy DIY EC2 approach — cheaper, but you manage HA, patching, bandwidth. Almost always answer "NAT Gateway" today.

**Debug scenario you can narrate in interviews:** "Pods in one AZ suddenly can't reach the internet, other AZs fine → immediately suspect the per-AZ NAT GW or that AZ's route table. Check route table target status; check NAT GW metrics (ErrorPortAllocation, PacketsDropCount)."

## 2.6 Security Groups vs NACLs

| | Security Group | Network ACL |
|---|---|---|
| Attaches to | ENI (instance/pod/LB) | Subnet |
| State | **Stateful** — reply traffic auto-allowed | **Stateless** — must allow both directions explicitly |
| Rules | Allow only | Allow **and** Deny |
| Evaluation | All rules evaluated together | Rules by number, first match wins |
| Default | Deny all inbound, allow all outbound | Default NACL allows all |

**Stateful vs stateless is the concept interviewers probe.** With an SG, if you allow inbound 443, the response packets go out automatically — no outbound rule needed. With a NACL, you must allow inbound 443 **and** outbound **ephemeral ports (1024–65535)**, because the client's reply goes back to a random high port. Forgetting ephemeral ports in NACLs is *the* classic NACL outage.

**Powerful SG pattern:** SGs can reference **other SGs**. Instead of "allow 5432 from 10.0.11.0/24," you say "allow 5432 from sg-app-servers." Membership-based security that survives IP changes — this is how well-run environments (and EKS: node SG, cluster SG) are built.

**Practice reality:** most shops use SGs heavily and leave NACLs at default, reserving NACLs for coarse subnet-level guardrails (e.g., "data subnet may never talk to the internet") or emergency IP blocks (NACL can *deny*; SG can't).

## 2.7 VPC Endpoints (private paths to AWS services)

Without endpoints, an EC2 in a private subnet calling S3 goes: instance → NAT → IGW → S3's public endpoint. Costly and traverses "internet-ish" space.

- **Gateway Endpoints** (S3, DynamoDB only): a route-table entry (`pl-xxxx → vpce-…`). **Free.** Every serious VPC should have the S3 gateway endpoint.
- **Interface Endpoints** (PrivateLink; almost every other service — ECR, STS, EC2 API, CloudWatch, SSM…): an ENI with a private IP inside your subnet, plus a private DNS name so `ecr.api.us-east-1.amazonaws.com` resolves to that private IP. Hourly + per-GB cost.

**Why enterprises care:** compliance ("no traffic leaves the private network"), NAT cost reduction, and enabling fully-isolated VPCs (no IGW/NAT at all) that can still use AWS services. In pharma/life-sciences environments (your world), isolated VPCs with interface endpoints are the norm.

**EKS tie-in:** a private EKS cluster with no internet egress needs interface endpoints for at least ECR (api + dkr), S3 gateway, EC2, STS, and CloudWatch Logs — or nodes fail to join and images fail to pull. If you've seen a node stuck `NotReady` in a locked-down cluster, missing endpoints are a prime suspect.

## 2.8 VPC Peering (and its limits — the setup for Transit Gateway)

**VPC Peering** = a private 1:1 connection between two VPCs (same or different accounts/regions). Traffic stays on the AWS backbone. Requirements: non-overlapping CIDRs, routes added in **both** VPCs' route tables, SGs updated.

**The killer limitation: peering is NOT transitive.** If A↔B and B↔C are peered, A **cannot** reach C through B. Ever. So full connectivity between N VPCs requires N×(N−1)/2 peerings: 10 VPCs = 45 peering connections, each with route-table entries to maintain. This "mesh explosion" is precisely the problem Transit Gateway was built to solve.

---

# PART 3: Transit Gateway (TGW) — The Cloud Router

## 3.1 What It Is and the Problem It Solves

A **Transit Gateway** is a regional, managed, highly-available **router** that acts as a hub. Instead of meshing VPCs together with peering, every VPC (and VPN, and Direct Connect link) attaches to the TGW once, and the TGW routes between them.

**Hub-and-spoke:** 10 VPCs = 10 attachments instead of 45 peerings. Adding VPC #11 = 1 new attachment, not 10 new peerings.

**What can attach to a TGW:**
- **VPC attachments** (the TGW places an ENI in a subnet in each AZ of the VPC — usually small dedicated /28 "TGW subnets")
- **VPN attachments** (site-to-site IPsec to on-prem)
- **Direct Connect Gateway** (dedicated physical link to on-prem)
- **TGW peering** (connect TGWs across regions or organizations — note: *TGW peering is also non-transitive*)

**Enterprise reality (your interview context):** virtually every large enterprise runs a hub-and-spoke: a central **network/egress VPC** (with firewalls, proxies, shared NAT), a **shared-services VPC** (DNS, AD, monitoring), and dozens of workload VPCs (one per app/team/environment) — all glued together by TGW, with on-prem datacenters connected via Direct Connect through the same TGW. When your Domino EKS cluster reaches an on-prem LDAP server or an internal artifact registry, the packets are almost certainly crossing a TGW.

## 3.2 TGW Route Tables — Where the Real Power (and Complexity) Lives

A TGW has its **own route tables**, separate from VPC route tables. Two independent operations control everything:

- **Association**: which TGW route table an attachment *uses for its outbound lookups*. Each attachment is associated with exactly **one** TGW route table.
- **Propagation**: an attachment *advertising its CIDRs into* a TGW route table (its routes appear there automatically). One attachment can propagate to **many** route tables.

**Default behavior:** a default TGW route table where every attachment associates and propagates → every VPC can reach every VPC. Fine for small setups; enterprises disable this.

**Segmentation example** (the pattern to describe in interviews):

```
TGW Route Table "prod":     associated: all prod VPC attachments
                            propagated routes: prod VPCs + shared-services + on-prem
TGW Route Table "nonprod":  associated: all dev/test attachments
                            propagated routes: nonprod VPCs + shared-services
TGW Route Table "shared":   associated: shared-services + on-prem attachments
                            propagated routes: everything
```

Result: prod and nonprod can both reach shared services and shared services can reach both, but **prod and nonprod can never reach each other** — enforced at the routing layer, without a single firewall rule. This is called **network segmentation via TGW route tables**, and it's a compliance cornerstone in regulated industries (pharma, finance).

## 3.3 The Two-Hop Route Lookup (the #1 TGW troubleshooting insight)

For VPC-A (10.0.0.0/16) to reach VPC-B (10.1.0.0/16) through TGW, **TWO route tables must both cooperate, in each direction**:

1. **VPC-A's subnet route table** must send the traffic to the TGW: `10.1.0.0/16 → tgw-xxx`. **The TGW attachment does NOT add this automatically. Ever.** You (or your IaC) must add it.
2. **The TGW route table associated with VPC-A's attachment** must know where 10.1.0.0/16 lives: `10.1.0.0/16 → attachment-vpc-b` (via propagation or a static route).
3. And the **return path** needs the mirror image: VPC-B's subnet route table `10.0.0.0/16 → tgw`, and B's associated TGW route table pointing back to A's attachment.

**Asymmetric routing failures** — where A→B works at the routing level but replies die — are the classic TGW incident. Symptom: `traceroute` shows packets leaving, TCP never completes (SYN sent, no SYN-ACK). Checklist: source VPC RT → source-side TGW RT → dest VPC RT → dest-side TGW RT → SGs/NACLs at destination.

Other must-knows:
- TGW attachments live in specific subnets; an AZ with **no TGW attachment subnet cannot use the TGW** (traffic from that AZ is blackholed). Always attach in every AZ the workloads use.
- **Blackhole routes**: a TGW route can be deliberately set to `blackhole` to drop traffic to a CIDR — used to slam the door on a compromised VPC instantly. If `aws ec2 describe-transit-gateway-route-tables` shows a route state `blackhole`, that's your answer to "why is traffic silently dropped."
- **MTU**: TGW supports 8500-byte jumbo frames for VPC↔VPC but **1500 over VPN**. Symptom of MTU issues: small requests (curl a health endpoint) work, large transfers hang. Classic senior-level scenario.
- **Cost**: per-attachment hourly + per-GB processed. High-volume VPC↔VPC pairs are sometimes given a direct peering to bypass TGW data charges.

## 3.4 "TGW Changes" in Change Tickets — What They Actually Mean

When a change record says "TGW changes," it's usually one of:
1. **New VPC attachment** (new app VPC being onboarded) — risk: route conflicts, forgotten return routes.
2. **Route table modifications** (new static route, new propagation, segmentation change) — risk: longest-prefix-match surprises hijacking existing traffic; a new more-specific route can silently redirect flows.
3. **VPN/DX changes** on TGW — risk: on-prem connectivity blips, BGP route flaps.
4. **Migration to/from TGW** (e.g., replacing legacy peering mesh) — the riskiest; done VPC-by-VPC with rollback routes staged.

**Why you (platform support) care:** after any TGW change window, the first symptom of a mistake is *your platform* breaking — Domino can't reach on-prem LDAP/AD, EKS nodes can't reach an internal registry, license servers unreachable. Being able to say "let me check whether the TGW route table associated with our VPC's attachment still has the on-prem prefixes propagated" is exactly the senior-level fluency the interview wants.

---

# PART 4: DNS from First Principles

## 4.1 What DNS Is

DNS translates names (`api.example.com`) into IPs (`52.1.2.3`). It's a **globally distributed, hierarchical, cached database** — arguably the largest distributed system on Earth.

**Why names at all?** IPs change (autoscaling, failover, migrations). Names are the stable layer of indirection. Nearly every "cutover" or "migration" in enterprise IT is ultimately executed *by changing what a name points to* — which is why DNS knowledge is disproportionately valuable.

## 4.2 The Resolution Chain (walk this in interviews)

When a client resolves `app.example.com`:

1. **Local caches**: browser cache → OS cache (`/etc/hosts` checked here too!) → if hit, done.
2. **Recursive resolver**: the client asks its configured resolver (from `/etc/resolv.conf` on Linux; in a VPC this is the **`.2` resolver**; at home your ISP's or 8.8.8.8). If the resolver has it cached, done.
3. Otherwise the resolver walks the hierarchy:
   - **Root servers** (`.`): "who handles `.com`?"
   - **TLD servers** (`.com`): "who handles `example.com`?" → returns the domain's **authoritative name servers** (for Route 53 domains, names like `ns-123.awsdns-45.com`)
   - **Authoritative server**: holds the actual records; returns `app.example.com A 52.1.2.3` **with a TTL**.
4. The resolver caches the answer for TTL seconds and returns it to the client.

**Recursive vs authoritative** — a resolver *finds* answers on behalf of clients and caches them; an authoritative server *owns* the answers for its zones. Route 53 hosted zones = authoritative. The VPC `.2` resolver = recursive. Keeping these two roles straight makes every hybrid-DNS design comprehensible.

## 4.3 Record Types You Must Know

| Type | Maps | Example / notes |
|------|------|-----------------|
| **A** | name → IPv4 | `app.example.com → 52.1.2.3` |
| **AAAA** | name → IPv6 | |
| **CNAME** | name → another name | `www → app.example.com`. **Cannot exist at a zone apex** (`example.com` itself) — a rule that Route 53 Alias records exist to work around. |
| **Alias** (Route 53-specific) | name → AWS resource | Points to ALB/CloudFront/S3/another R53 record. Works at apex, resolves to A records, free queries, auto-tracks the target's changing IPs. **Default choice for anything fronting an ELB.** |
| **NS** | zone → its authoritative servers | Delegation glue |
| **SOA** | zone metadata | serial, refresh, **negative-caching TTL** |
| **MX** | mail routing | |
| **TXT** | arbitrary text | SPF/DKIM, **domain-validation for TLS certs (ACM!)** |
| **PTR** | IP → name (reverse DNS) | |
| **SRV** | service discovery (port+host) | AD/LDAP, Kubernetes headless services |

**CNAME vs Alias is a guaranteed interview question.** Answer: Alias is Route 53's internal pointer to AWS resources — apex-compatible, free, returns A records directly (one less lookup), and health-checkable. CNAME is standard DNS, works anywhere, but not at apex and each lookup is a chargeable extra hop.

## 4.4 TTL — The Concept That Rules All Cutovers

**TTL (Time To Live)** = how long resolvers worldwide may cache a record. TTL 3600 means: after you change the record, some clients may keep using the OLD value for **up to an hour** — and you cannot force remote resolvers to flush.

Consequences every senior engineer internalizes:
- **DNS changes are never instant.** "Propagation" is really *cache expiry*, happening independently at thousands of resolvers.
- Long TTL (3600–86400) = fewer queries, cheaper, more cache resilience — but slow to change.
- Short TTL (30–60s) = agile changes — but more query load and more exposure if your DNS is down.
- **Negative caching**: a NXDOMAIN ("name doesn't exist") answer is *also* cached, per the SOA's negative TTL. Query a name *before* creating it and you can poison caches with "doesn't exist" — a subtle pre-cutover landmine: don't test the new name before the record exists.

---

# PART 5: Route 53 — AWS's DNS Service

## 5.1 The Three Jobs of Route 53

1. **Domain registration** (buy/renew domains) — commodity.
2. **Authoritative DNS hosting** (hosted zones) — the core.
3. **Health checking & traffic policy routing** — the differentiator.

Fun fact for rapport: the name is Route "53" because DNS runs on port 53.

## 5.2 Public vs Private Hosted Zones

- **Public hosted zone**: answers queries from the whole internet. Creating one gives you 4 NS servers; delegation from the registrar's NS records makes it live.
- **Private hosted zone (PHZ)**: answers **only** queries originating from VPCs you associate with it. This is how `db.internal.mycompany.com` resolves inside your VPCs but doesn't exist to the world.

**Split-horizon DNS** — a senior-level pattern: the *same name* in a public zone and a private zone. `app.example.com` publicly resolves to the external ALB; inside the VPC, the PHZ shadows it and resolves to an internal ALB. Internal traffic stays private. **Gotcha:** the PHZ answers *authoritatively* for its whole zone within associated VPCs — any public record you didn't replicate into the PHZ becomes unresolvable from inside (NXDOMAIN). "Works from my laptop, NXDOMAIN from the pod" → check for a shadowing private zone. This single gotcha explains a huge share of enterprise DNS tickets.

**EKS/Domino tie-in:** CoreDNS in the cluster forwards non-cluster names to the VPC `.2` resolver, which consults PHZs. So private zone misconfigurations surface as *pod-level* DNS failures — your territory.

## 5.3 Routing Policies (know all, master four)

| Policy | Behavior | Use case |
|--------|----------|----------|
| **Simple** | Static answer | Default |
| **Weighted** | Split by weights (e.g., 90/10) | **Canary releases, gradual cutovers** |
| **Latency-based** | Answer with the region lowest-latency to the resolver | Multi-region performance |
| **Failover** | Primary while healthy; secondary when health check fails | **DR / active-passive** |
| Geolocation | By user's country/continent | Compliance, localization |
| Geoproximity | By distance, with bias dials | Traffic shifting |
| Multivalue answer | Up to 8 healthy records, client picks | Poor-man's LB |
| IP-based | By resolver's source CIDR | ISP-specific routing |

**Weighted + Failover are the two you'll use in every migration and DR design** — see the cutover playbook below.

## 5.4 Health Checks

Route 53 health checkers (a global fleet) probe an endpoint (HTTP/HTTPS/TCP, expected status/string match); a record tied to a failing health check is withdrawn from answers. Can also alarm on CloudWatch metrics or aggregate child health checks ("calculated" checks).

**Critical subtlety:** health checkers live **on the internet** — they cannot probe private IPs directly. For internal failover you use health checks based on **CloudWatch alarms** instead. Also: your firewall/WAF must allow the published Route 53 health-checker IP ranges, or you'll "fail" a perfectly healthy endpoint — a real incident pattern.

## 5.5 Route 53 Resolver — Hybrid DNS (this is where "Resolvers" in your topic list lives)

**The default:** every VPC has the **Amazon-provided resolver at VPC-CIDR+2** (`10.0.0.2` in a 10.0.0.0/16 VPC), a.k.a. AmazonProvidedDNS / the ".2 resolver". It resolves: private hosted zones, EC2 internal names, VPC endpoint private names, and public internet names. Every instance's DHCP-provided `/etc/resolv.conf` points at it. **It is only reachable from within the VPC** — on-prem servers cannot query it directly. That limitation creates the hybrid-DNS problem:

**Problem 1 — cloud needs on-prem names:** a pod must resolve `ldap.corp.internal`, which only the datacenter's DNS servers know.
**Problem 2 — on-prem needs cloud names:** a datacenter app must resolve `db.prod.aws.internal`, which lives in a private hosted zone.

**Route 53 Resolver Endpoints** solve both:

- **Outbound Resolver Endpoint** + **Forwarding Rules**: ENIs in your VPC that *forward* selected domains outward. Rule: "queries for `corp.internal` → forward to 192.168.1.10 and 192.168.1.11 (on-prem DNS), via the outbound endpoint (over TGW/DX/VPN)." Rules can be **shared across the org (AWS RAM)** and associated with many VPCs — central DNS team manages once, everyone inherits.
- **Inbound Resolver Endpoint**: ENIs with fixed private IPs *inside* the VPC that **accept** DNS queries from outside the VPC. On-prem DNS servers are configured with a conditional forwarder: "for `*.aws.internal`, ask 10.0.5.10 and 10.0.6.10 (inbound endpoint IPs)." Those queries then resolve against PHZs exactly as if they came from inside.

**The full enterprise hybrid picture** (describe this and you sound senior):

```
On-prem client → on-prem DNS → conditional forwarder (*.aws.internal)
   → over Direct Connect/TGW → Inbound Endpoint → .2 resolver → Private Hosted Zone ✔

Pod in EKS → CoreDNS → .2 resolver → matches forwarding rule (corp.internal)
   → Outbound Endpoint → over TGW → on-prem DNS ✔
Everything else → .2 resolver → public internet DNS ✔
```

Also know: **Resolver DNS Firewall** (allow/deny-lists domains at the resolver — blocks malware C2 / data exfil via DNS) and **Resolver Query Logging** (log every query per VPC — the first tool you reach for when debugging "what is this pod actually resolving?").

## 5.6 "DNS Server Changes" in Enterprise Change Tickets

When a change record says "DNS server changes," it typically means one of these — each with distinct blast radius:

1. **Changing which resolvers clients use** (DHCP option sets in AWS, DHCP scope on-prem): a new **DHCP option set** on a VPC changes `/etc/resolv.conf` for instances **on lease renewal or restart** — staggered, confusing rollout; some instances on old resolvers, some on new. Symptom: "DNS works on new pods but not old nodes."
2. **Migrating authoritative DNS** (e.g., on-prem BIND/Infoblox → Route 53, or between providers): export zone, import, verify record parity, lower TTLs, then **change NS delegation at the registrar/parent zone** — the moment of cutover. Parent NS TTLs are often 24–48h, so plan a long overlap where **both** old and new servers keep serving identical data.
3. **Changing forwarder targets** (on-prem conditional forwarders, or R53 forwarding-rule target IPs): if the old on-prem DNS IPs are decommissioned before every forwarding rule/endpoint is updated, cloud→on-prem resolution dies. Always verify rule target IPs during datacenter DNS refresh projects.
4. **CoreDNS changes in EKS** (your domain!): Corefile edits, forwarders, NodeLocal DNSCache rollout. Symptoms of trouble: cluster-wide 5s DNS latency (the infamous conntrack/ndots issues), resolution failures for external names only, etc.

**Universal DNS-change safety rules:** lower TTLs *at least one old-TTL period before* the change; keep old infrastructure serving until query logs show traffic has drained; never delete the old answer until well after cutover; verify with `dig` against **specific servers** (`dig @old-server`, `dig @new-server`, `dig @10.0.0.2`) rather than trusting your own cache.

## 5.7 DNS Cutover — The Complete Playbook

"DNS cutover" = redirecting traffic from environment A to environment B by changing DNS. This is *the* standard mechanism for migrations (datacenter→cloud, region→region, old ALB→new ALB, blue→green).

**Phase 0 — Preparation (days before):**
- Inventory every record touching the migration; find hardcoded IPs (grep configs — hardcoded IPs are the silent cutover-killers that DNS cannot fix).
- **Lower TTLs** on affected records to 60s. Do this at least one *old-TTL* before the window (if TTL was 86400, lower it 24h+ in advance) — otherwise resolvers still hold the old long-TTL answer through your window.
- Build and fully validate the target environment; test it by IP or a temporary test hostname (`app-new.example.com`).
- Pre-stage the change (Route 53 change batch / Terraform plan reviewed), define **success criteria** and an explicit **rollback trigger** ("error rate > 2% for 5 min → roll back").

**Phase 1 — Cutover (the window):**
- Option A, **hard cutover**: update the record (ideally an Alias swap to the new ALB). Atomic in Route 53 (change batches are all-or-nothing), takes effect as caches expire (≤ new 60s TTL).
- Option B, **weighted (canary) cutover** — the senior answer: create weighted records old:new at 95:5 → watch metrics → 75:25 → 50:50 → 0:100. Instant rollback at any step by resetting weights. Caveat: DNS weighting is statistical and granular only at resolver level; corporate resolvers can pin thousands of users to one answer.
- Keep the **old environment fully running** throughout.

**Phase 2 — Validation:**
- `dig app.example.com @8.8.8.8`, `@1.1.1.1`, `@<corporate resolver>`, and from inside VPCs (`@10.0.0.2`) — confirm the new answer everywhere. (Remember split-horizon: update the private zone too, or internal users are still on the old path!)
- Watch new-side dashboards (requests, latency, 4xx/5xx, TLS errors) **and old-side traffic draining to ~zero**. Old-side access logs are your ground truth for "who hasn't moved" — long-TTL stragglers, hardcoded IPs, /etc/hosts entries, embedded clients that resolve once at startup (JVMs with default cache-forever behavior are notorious: `networkaddress.cache.ttl`).

**Phase 3 — Post-cutover:**
- After stability (24–72h), raise TTLs back to normal (e.g., 300–3600).
- Decommission the old environment only after logs show zero legitimate traffic — a common practice is "leave it dark but running for a week."

**Rollback:** it's just the reverse DNS change — which is exactly why TTLs were lowered: rollback also takes effect within ~60s. If you never lowered TTLs, rollback takes hours. That single sentence demonstrates senior-level thinking in an interview.

**Certificate coupling (foreshadowing Part 6):** the new endpoint must present a valid cert for the hostname *before* cutover. Cert misses are the #1 embarrassing cutover failure: DNS flips perfectly and every browser screams `NET::ERR_CERT_COMMON_NAME_INVALID`.

---

# PART 6: SSL/TLS and Certificates

## 6.1 What TLS Actually Does

TLS (Transport Layer Security — "SSL" is the legacy name everyone still uses) gives an HTTP connection three properties:
1. **Encryption** — nobody on the path can read the traffic.
2. **Authentication** — you're really talking to `api.example.com`, not an impostor (this is what certificates are for).
3. **Integrity** — traffic can't be silently modified in transit.

HTTPS = HTTP inside a TLS tunnel, on port 443.

## 6.2 The TLS Handshake (know the shape, not every byte)

After the TCP handshake:
1. **ClientHello**: client sends supported TLS versions, cipher suites, and — critically — **SNI (Server Name Indication)**: the hostname it wants, in plaintext. SNI is how one IP/load balancer can host many HTTPS sites and pick the right certificate; it's also what proxies like Zscaler read to know what you're connecting to.
2. **ServerHello + Certificate**: server picks version/cipher and sends its **certificate chain**.
3. **Client verifies the certificate** (Section 6.4 — where everything goes wrong).
4. **Key exchange** (ephemeral Diffie–Hellman in modern TLS 1.2/1.3 → *forward secrecy*): both sides derive the same symmetric session keys.
5. Everything after is symmetric-encrypted application data.

TLS 1.3 (current standard) compresses this to one round trip and removes weak options; TLS 1.0/1.1 are deprecated/banned by compliance baselines. "Disable TLS 1.0/1.1 on the ALB listener policy" is itself a common change ticket.

## 6.3 What a Certificate Is

An X.509 certificate is a signed statement: *"This public key belongs to this hostname, vouched for by this Certificate Authority, valid from date A to date B."* Key fields:
- **Subject / SANs (Subject Alternative Names)**: the hostnames it's valid for. Modern validation uses SANs exclusively (CN alone is ignored). Wildcard `*.example.com` covers one label deep (`app.example.com` yes; `a.b.example.com` **no** — classic gotcha).
- **Issuer**: which CA signed it.
- **Validity window**: public certs are now capped at ~398 days, hence constant rotation (and hence ACM's value).
- **Key usage / EKU**: e.g., serverAuth vs clientAuth.

## 6.4 The Chain of Trust (the concept behind 90% of cert incidents)

Your server cert (**leaf**) is signed by an **intermediate CA**, which is signed by a **root CA**. Clients ship with a **trust store** of root CAs (OS/browser/JVM cacerts). Verification walks: leaf → intermediate(s) → a root the client already trusts. The client also checks: hostname matches a SAN, dates valid, not revoked, signature math checks out.

**The classic failure:** the server must send **leaf + intermediates** (the "full chain"). Servers that send only the leaf work in browsers (which cache/fetch intermediates) but **fail from curl, Java, Python** with `unable to get local issuer certificate`. "Works in Chrome, fails from the pod" ⇒ incomplete chain, almost every time. Diagnose with:

```bash
openssl s_client -connect api.example.com:443 -servername api.example.com -showcerts
# -servername sets SNI — forgetting it tests the WRONG cert on multi-tenant LBs
echo | openssl s_client -connect host:443 -servername host 2>/dev/null | openssl x509 -noout -dates -subject -issuer
```

**The other classic:** private/internal CA. Enterprises run internal CAs for internal services; clients (containers!) that don't have the internal root in their trust store fail verification. Fix: bake the internal root CA into base images / mount it / update `ca-certificates`. In Kubernetes this is the eternal "add the corp CA to the image or a trust bundle" ticket. Zscaler makes this universal — see Part 7.

## 6.5 ACM — AWS Certificate Manager

- **Free public certs** for use on AWS-managed endpoints: **ALB/NLB, CloudFront, API Gateway** ("managed" is the key: you can't export the private key or install an ACM public cert on an EC2/nginx yourself).
- **Validation**: DNS validation = create a specific CNAME ACM gives you (in Route 53, one click). Leave that CNAME in place → **automatic renewal forever**. This is the killer feature: no more expiry incidents for the edge. (Email validation exists; avoid it — manual renewals.)
- **Regionality**: certs are regional; **CloudFront requires the cert in us-east-1** — a famous gotcha.
- **ACM Private CA**: managed internal CA for private/mTLS certs (pricey, used in regulated orgs; integrates with cert-manager on EKS).

## 6.6 "SSL/TLS and Certificate Changes" in Change Tickets — and Their Failure Modes

Typical changes and what breaks:

1. **Renewal/rotation** (the most common): replace an expiring cert on ALB listener / CloudFront / API GW / ingress controller. Risks: wrong SAN set on the new cert; forgetting one of several places the cert is installed (ALB *and* CloudFront *and* the origin); with **certificate pinning** in mobile apps or B2B clients, rotating the cert breaks pinned clients — must coordinate.
2. **New SAN / new domain added**: reissue with expanded SANs, deploy everywhere.
3. **CA migration** (e.g., DigiCert → ACM, or new intermediate): old clients/partners with stale trust stores or pinned intermediates fail. Ask: "do any consumers pin?"
4. **Policy hardening**: bump ALB security policy (disable TLS 1.0/1.1, weak ciphers). Risk: ancient clients (old Java 7 agents, embedded devices, legacy Windows) can no longer connect — check access logs for TLS version distribution *before* the change.
5. **Enabling mTLS** (client certs required): huge coordination — every client needs a cert issued and configured.
6. **Internal CA root rotation**: every trust store in the company must get the new root **before** services start presenting certs from it — a months-long dual-trust program, not a change window.

**Kubernetes-specific cert surfaces you already touch:** the EKS cluster CA in kubeconfigs; kubelet/API server certs (EKS manages these); admission-webhook certs (expired webhook cert = cluster-wide deploy failures — a legendary incident class); cert-manager-issued ingress certs; Domino's own TLS at its ingress. When "everything is broken" in a cluster after months of stability, *check webhook and internal cert expiry early.*

**Monitoring (the senior answer to "how do you prevent cert outages?"):** don't rely on humans. CloudWatch `DaysToExpiry` (ACM emits it), Blackbox-exporter/Prometheus probing `probe_ssl_earliest_cert_expiry`, or a synthetic check, alerting at 30/14/7 days — covering *every* endpoint including internal ones, because the forgotten internal cert is always the one that expires.

---

# PART 7: Zscaler — Enterprise Security in the Traffic Path

## 7.1 What Zscaler Is

Zscaler is a **cloud-delivered security proxy platform** (the leading "SSE/SASE" vendor). Instead of routing all traffic through datacenter firewall/proxy appliances, enterprises route user and (sometimes) server traffic through Zscaler's global cloud, which enforces security policy. Two main products:

- **ZIA (Zscaler Internet Access)** — a **forward proxy to the internet**: URL filtering, malware scanning, DLP (data-loss prevention), sandboxing, and **TLS inspection**. Replaces the old on-prem web proxy. Traffic gets to ZIA via the **Zscaler Client Connector** agent on endpoints, GRE/IPsec tunnels from offices, or PAC files.
- **ZPA (Zscaler Private Access)** — **VPN replacement (ZTNA, zero-trust network access)**: users get brokered, per-application access to internal apps. **App Connectors** (small VMs/containers deployed *inside* your VPCs/datacenter) dial **out** to the Zscaler cloud; user connections are stitched to app connections in the broker. No inbound firewall holes, users never get "on the network," apps are invisible to scanning. Access is identity + policy based, per app — that's "zero trust."

Why enterprises adopt it: users are everywhere (remote work), apps are everywhere (multi-cloud + datacenter), so backhauling everything to a datacenter perimeter is slow and pointless; move the security perimeter into the cloud, close inbound firewall exposure, get uniform policy + logging for compliance. In pharma/finance environments (your interview context), assume Zscaler or an equivalent is in the path of *every employee's* traffic.

## 7.2 TLS Inspection — The Part That Causes All Your Tickets

To scan HTTPS content, ZIA performs **sanctioned man-in-the-middle**: it terminates your TLS session, inspects the plaintext, and re-encrypts onward — presenting the client a certificate for the destination **signed by the Zscaler (or company) intermediate CA**, not the real CA.

For this to work invisibly, **every client must trust the Zscaler root CA** — pushed by IT to managed laptops' OS trust stores. And here's the operational blast radius you've almost certainly felt:

- **CLI tools and language runtimes don't use the OS trust store**: Python (`certifi`), Node, Java (its own `cacerts`), pip, npm, git, curl-in-a-container… all fail with `SSL: CERTIFICATE_VERIFY_FAILED` / `self-signed certificate in certificate chain` on Zscaler-equipped machines. Root cause: they see the Zscaler-signed cert and don't trust its CA.
- **Fixes** (in order of correctness): add the Zscaler root CA to each tool's trust store (`REQUESTS_CA_BUNDLE`, `NODE_EXTRA_CA_CERTS`, `keytool -importcert` into JVM cacerts, `git config http.sslCAInfo`, bake into container base images); or have the Zscaler admin add a **TLS-inspection bypass** for specific trusted destinations. Never normalize `verify=false` — it silently disables authentication.
- **Containers/build agents on corporate networks** inherit the problem: docker builds pulling packages fail TLS inside the build. Standard enterprise pattern: a base image layer that installs the corp+Zscaler CA bundle.
- **Certificate pinning breaks under inspection** (banking apps, some SDKs) — those destinations go on the inspection **bypass list**.
- **Diagnostic one-liner**: `openssl s_client -connect site:443 -servername site | openssl x509 -noout -issuer` — if the issuer says Zscaler, you've *proven* inspection is in path. That's a beautiful, concrete troubleshooting move to describe in an interview.

## 7.3 "Zscaler-Related Updates" in Change Tickets

Typical changes and their platform-side symptoms:

1. **Policy changes** (URL categories, cloud-app controls, file-type blocks): suddenly `pip install` or an API to a newly-categorized SaaS fails — often with a Zscaler block page (HTML!) returned to a CLI expecting JSON, producing bizarre parse errors. *"We started getting HTML in our JSON responses"* ⇒ proxy block page, nearly always.
2. **SSL-inspection scope changes** (new categories inspected / bypass list edits): tools that worked yesterday start failing cert verification today with zero changes on your side. First question in triage: *"Was there a Zscaler change last night?"*
3. **Client Connector version rollouts**: local connectivity oddities, split-tunnel behavior changes, developer VPN conflicts.
4. **ZPA changes** (new App Segments, connector upgrades, policy edits): users lose/gain access to internal apps like the Domino UI; App Connector down in a VPC ⇒ "app unreachable via ZPA" while the app itself is perfectly healthy — check connector health *before* touching the app.
5. **Traffic-forwarding changes** (PAC file edits, GRE/IPsec tunnel work, new datacenter egress): whole-office connectivity behavior shifts; asymmetric weirdness during tunnel failovers.

**Server-side/VPC egress note:** some enterprises also force *workload* (not just user) egress through Zscaler or an inspection VPC (via TGW to a security VPC, or Zscaler Cloud/Branch Connectors). If your EKS nodes' outbound traffic transits inspection, everything in 7.2 applies to *pods* — the corp CA bundle must be in your container images and helm-deployed apps. Knowing this pattern exists is genuinely senior.

---

# PART 8: WAF — Web Application Firewall

## 8.1 What a WAF Is and Where It Sits

Network firewalls/SGs work at L3/L4 (IPs and ports) — they can't tell a legitimate login POST from a SQL-injection POST, because both are "TCP 443, allowed." A **WAF operates at Layer 7**: it parses HTTP requests (path, headers, body, cookies, query strings) and applies rules against them, protecting against the OWASP Top 10 web attacks: **SQL injection, XSS, path traversal, remote/local file inclusion**, plus bots, scrapers, credential stuffing, and L7 DDoS floods.

**AWS WAF attaches to:** CloudFront, **ALB**, API Gateway, AppSync, Cognito. (Not NLB — NLB is L4 and never sees HTTP.) Traffic order at the edge: `Client → CloudFront(+WAF) → ALB(+WAF) → targets`. Related: **AWS Shield** (DDoS; Standard free, Advanced paid) and **AWS Network Firewall** (L3/L4/IPS for VPC traffic) — know the distinction: WAF = HTTP semantics; Network Firewall = packets/flows; Shield = volumetric DDoS.

## 8.2 How AWS WAF Is Structured

- **Web ACL**: the top-level policy attached to a resource; has a **default action** (Allow or Block) and an ordered list of rules; capacity measured in WCUs.
- **Rules**: match statements + action. Actions: **Allow, Block, Count** (match but don't enforce — log only), **CAPTCHA/Challenge**.
- **Managed Rule Groups**: pre-built rule sets — AWS's (Core rule set/CRS, Known Bad Inputs, SQLi, Linux/PHP/WordPress-specific, **IP Reputation**, **Bot Control**) and third-party (F5, Fortinet…). The standard baseline: CRS + Known Bad Inputs + IP Reputation.
- **Custom rules**: match on IPs (IP sets), geo, headers, URI patterns, body, sizes, with AND/OR/NOT logic. E.g., "block requests to `/admin/*` not from the corporate IP set."
- **Rate-based rules**: "if one IP exceeds N requests per 5 minutes (optionally scoped to a path like `/login`), block it" — the L7 anti-brute-force/anti-scrape workhorse.
- **Scope-down statements**: restrict an expensive rule (e.g., Bot Control) to only part of the traffic.
- **Logging**: full request logs to Kinesis Firehose → S3/OpenSearch; CloudWatch metrics per rule. Non-negotiable in production — without logs you cannot debug false positives.

## 8.3 The Central Operational Tension: False Positives

WAF rules are pattern matchers, and legitimate traffic sometimes looks like attacks: a CMS user writes a blog post *about* SQL containing `SELECT * FROM`; a JSON payload includes `<script>` in a code-sharing field; a data-science platform (hello, Domino) POSTs notebook content full of quotes, semicolons, and shell snippets. The CRS `SQLi_BODY` or `CrossSiteScripting_BODY` rules fire → **403 for a legitimate user**.

**The professional deployment workflow (this answer alone can carry a WAF interview question):**
1. Deploy every new rule/rule group in **Count mode** first.
2. Bake for days/weeks; analyze logs/metrics for what *would have been* blocked.
3. Investigate matches: real attacks vs. legitimate-but-weird traffic.
4. Add **exceptions**: exclude specific sub-rules, or scope-down ("don't run SQLi body inspection on `/api/notebooks/*`"), or label-based overrides.
5. Only then flip to **Block** — and watch 403-rate dashboards during the change window.

**Troubleshooting "users are getting 403s after the WAF change":** WAF logs filtered by `action=BLOCK` → identify the **terminating rule** → decide false positive vs. attack → refine (exclude rule / scope-down / adjust) — or in a bleeding-emergency, flip that rule to Count. Also know the *silent* failure: WAF body inspection has size limits (ALB inspects up to ~8KB by default with configurable oversize handling) — "large uploads fail / oversized requests behave oddly" can be oversize-handling configuration.

## 8.4 "WAF Configuration Changes" in Change Tickets

1. **New managed rule group version rollouts** (AWS updates rule logic): behavior can shift without you touching anything — pin versions or watch metrics after auto-updates.
2. **Adding/tuning rules** (new custom rule, rate-limit thresholds, geo blocks): the count-first workflow above; risk = false positives.
3. **IP set updates** (allowlist a new partner/scanner/office range; blocklist attackers): trivial change, huge impact if a monitoring/scanner IP is missed (health checks blocked = "outage" that's actually the WAF).
4. **Emergency response**: mid-attack, add rate rules/IP blocks fast — WAF is one of the few places "change now, paper later" is common; make sure you know your org's emergency-change process (your ServiceNow background is relevant precisely here).
5. **Attach/detach WAF to a resource**: attaching a Web ACL with default-Block and wrong rules to a live ALB is an instant self-inflicted outage. Review default action *every time*.

---

# PART 9: Putting It All Together

## 9.1 The Full Enterprise Request Path (narrate this in interviews)

*A remote employee opens `domino.pharma-corp.com`:*

1. Laptop's Zscaler Client Connector intercepts; **ZPA** policy says this is an internal app → brokered via an App Connector in the shared-services VPC. (Or, for a public app: DNS → CloudFront → WAF → ALB.)
2. **DNS**: corporate resolver → conditional forwarder → **Route 53 inbound resolver endpoint** → **private hosted zone** → internal ALB IPs.
3. **Routing**: user traffic reaches the app VPC via **TGW** (from the connector's VPC), governed by TGW route tables enforcing prod/nonprod segmentation.
4. **Security gates**: ALB security group; possibly an internal **WAF** on the ALB.
5. **TLS**: ALB presents an **ACM** cert (auto-renewing, DNS-validated); ends at the ingress; possibly re-encrypted to pods.
6. Pod needs on-prem LDAP: CoreDNS → `.2` resolver → **outbound resolver endpoint forwarding rule** → on-prem DNS; packets route over **TGW → Direct Connect**; pod pulls a package from PyPI: egress via **NAT Gateway** (or inspected egress), S3 artifacts via the **gateway endpoint**.

Every topic in your list appears in that one request. If you can tell this story fluidly, you *are* operating at senior level.

## 9.2 Scenario Drills (senior-level Q&A)

**Q1. After a change window, pods can reach the internet but not the on-prem artifact registry. Walk me through it.**
Bisect: DNS or routing? `dig registry.corp.internal @10.0.0.2` — NXDOMAIN/timeout ⇒ DNS path: forwarding rule intact? outbound endpoint ENIs healthy? on-prem DNS target IPs changed in last night's "DNS server change"? Resolves fine ⇒ routing: VPC route table still has on-prem CIDR → TGW? TGW route table for our attachment still has on-prem propagations (DX/VPN attachment healthy? blackhole routes?)? Then SGs/on-prem firewall. State that you'd check *the change record* first — correlating symptoms to last night's change is the fastest diagnostic there is.

**Q2. TLS errors from CI containers only, browsers fine. Why?**
`openssl s_client ... | openssl x509 -noout -issuer`. Issuer = Zscaler/corp CA ⇒ TLS inspection; containers lack the corp root → add CA bundle to images / env vars / request bypass. Issuer = real CA ⇒ probably incomplete chain (server missing intermediates): browsers tolerate, OpenSSL/Java don't.

**Q3. You changed a Route 53 record 30 minutes ago; some users still hit the old target. Failure? **
No — TTL cache expiry. Check what the TTL *was* before the change; resolvers hold the old answer that long. Plus stragglers: JVM infinite DNS cache, hardcoded IPs, /etc/hosts, corporate resolvers with cache-min policies, and the private-zone copy you might have forgotten (split-horizon).

**Q4. How would you migrate an app from on-prem to AWS with near-zero downtime?**
Build+validate in AWS behind a test hostname (valid cert on the new endpoint *first*) → lower TTLs a day ahead → weighted DNS 95/5 → shift while watching error/latency dashboards on both sides → 0/100 → drain-verify old logs → raise TTLs → decommission after a dark-standby week. Rollback = reset weights (fast *because* TTLs are low).

**Q5. One AZ's workloads lost internet egress; other AZs fine.**
Per-AZ NAT GW or its route table. Check the private route table for that AZ (0.0.0.0/0 target state = blackhole?), NAT GW status/metrics (ErrorPortAllocation = port exhaustion to a hot destination), and whether someone "consolidated" NAT GWs in a cost-cutting change.

**Q6. After a WAF update, 0.5% of API POSTs return 403.**
WAF logs, filter BLOCK, find terminating rule (likely a CRS body-inspection sub-rule on payloads containing SQL-ish/script-ish content). Short-term: flip that sub-rule to Count or add a scope-down excluding the affected path. Long-term: count-mode bake for all future rule changes.

**Q7. Prod VPC A must never talk to dev VPC B, but both need shared services. Design it.**
TGW with segmented route tables: prod RT and nonprod RT each propagate shared-services (+on-prem) but not each other; shared RT propagates all. Defense in depth: SGs referencing prefix lists, and NACL guardrails on data subnets.

**Q8. What breaks if two teams' VPCs both use 10.0.0.0/16 and now must integrate?**
No clean peering/TGW routing (ambiguous longest-prefix). Options, worst→best: NAT/PrivateLink between them (PrivateLink is the standard overlap workaround — expose the service as an endpoint service, consumer gets an interface endpoint, no routing between CIDRs at all); secondary CIDRs + migration; re-IP one VPC. Then: institute org-wide IPAM.

## 9.3 30-Second Definitions (rapid-fire recall)

- **VPC**: your isolated software-defined network in a region; you own the CIDR, subnets, and every path in/out.
- **Subnet**: AZ-scoped slice of the VPC; public vs private is decided *only* by its route table.
- **Route table**: destination→target rules; longest prefix wins; every subnet has exactly one.
- **IGW**: the VPC's door to the internet; 1:1 NAT for public IPs.
- **NAT Gateway**: many-private-to-one-public source NAT for *outbound-only* internet from private subnets; per-AZ; connection-tracked.
- **VPC endpoint**: private path to AWS services; gateway (S3/DDB, free, route entry) vs interface (ENI+private DNS).
- **Peering**: private 1:1 VPC link; non-transitive; mesh pain at scale.
- **TGW**: regional hub router for VPCs/VPN/DX; its own route tables (association+propagation) enable segmentation; remember the two-hop lookup.
- **DNS**: hierarchical cached name→IP database; recursive resolvers vs authoritative servers; TTL governs change speed.
- **Route 53**: AWS authoritative DNS + registrar + health checks; public vs private hosted zones; alias records; weighted/failover policies power cutovers and DR.
- **Resolver (.2)**: the built-in VPC recursive resolver; inbound/outbound endpoints + forwarding rules = hybrid on-prem↔cloud DNS.
- **DNS cutover**: migrate traffic by changing records: lower TTLs early, canary with weights, validate with dig against specific servers, drain old logs, rollback = reverse change.
- **TLS**: encryption + authentication + integrity; certs bind hostnames to keys via a CA chain of trust; full chain and SANs are where incidents live; ACM automates the AWS edge.
- **Zscaler**: cloud security proxy — ZIA (internet, TLS inspection → trust-store tickets) and ZPA (zero-trust app access via outbound-dialing connectors).
- **WAF**: L7 HTTP firewall on CloudFront/ALB/API GW; managed rule groups + custom + rate rules; deploy in Count first; false positives are the job.

## 9.4 Hands-On Lab Path (fastest route from reading to owning)

Do these in a personal AWS account (nearly all free-tier-friendly except TGW/NAT hours — build, test, tear down same day):

1. **VPC from scratch, no wizard**: VPC 10.0.0.0/16, 2 AZ × (public+private) subnets, IGW, per-AZ NAT, route tables by hand. Prove: private EC2 can `curl` out via NAT but is unreachable inbound. Add an S3 gateway endpoint and watch S3 traffic bypass NAT (VPC Flow Logs).
2. **Two VPCs + TGW**: attach both, break connectivity three different ways on purpose (remove VPC route / remove propagation / blackhole route) and diagnose each with Route Analyzer + flow logs.
3. **Route 53 drills**: a cheap domain (~$12); hosted zone; ALB + alias record; weighted 50/50 between two targets and watch distribution; a failover pair with a health check and kill the primary. Practice `dig +trace`, `dig @server`, TTL observation.
4. **Private hosted zone + resolver**: PHZ `internal.lab` on the VPC; resolve from an instance; add a resolver query log and read it; (optional) outbound endpoint with a forwarding rule to a dummy DNS server to see the mechanics.
5. **TLS**: ACM DNS-validated cert on the ALB; then break it on purpose — request a cert without the right SAN and observe the browser vs curl errors; run the openssl one-liners until they're muscle memory.
6. **WAF**: Web ACL on the ALB with CRS in Count mode; send `curl "https://yourapp/?q=' OR 1=1--"`; read the sampled requests/logs; flip to Block; verify 403; add an exclusion.

Roughly 10–14 focused hours total, and it converts every concept in this guide into something you've *broken and fixed yourself* — which is exactly what senior interviews probe for.

---
*End of guide. Suggested next step: after the labs, revisit Part 9.2 and answer each scenario out loud without notes.*
