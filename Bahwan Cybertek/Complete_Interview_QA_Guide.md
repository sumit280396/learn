# Complete Interview Preparation Guide
## Platform Engineer — AWS Migration, Networking, IaC, Security
### Every Why/How Question With Mentor-Level Answers

**How to use this guide:** Each section starts with a 2-minute mental model (read this even if you skip questions), then questions ordered from foundational to senior-level. Answers are written to teach, not just to memorize. Words in *italics* are terms you should use naturally in your answers — they signal hands-on experience.

---

# SECTION 1: AWS NETWORKING CORE (VPC, NAT Gateway, Transit Gateway, Route Tables)

## Mental Model First

Think of a **VPC** as your own private building inside AWS's city. You choose the address range (CIDR block, e.g., 10.0.0.0/16 = ~65,000 addresses). Inside the building you create **subnets** — floors of the building, each in one Availability Zone. A subnet is "public" or "private" based on ONE thing only: **what its route table says**. There is no "public subnet" checkbox — a subnet is public if its route table sends 0.0.0.0/0 (all internet-bound traffic) to an **Internet Gateway (IGW)**.

**Route tables** are the building's signage: "traffic destined for X, go through door Y." Every subnet consults exactly one route table. Routes are matched by **longest prefix match** — the most specific route wins (a /24 route beats a /16 route beats 0.0.0.0/0).

A **NAT Gateway** is the one-way revolving door: things inside private subnets can go OUT to the internet (pull Docker images, hit APIs, download patches), but nothing outside can initiate a connection IN.

A **Transit Gateway (TGW)** is the highway interchange. Without it, connecting N VPCs requires N×(N-1)/2 peering connections (a mesh nightmare at scale). With TGW, every VPC attaches once to the hub, and TGW route tables decide who can talk to whom. This is how enterprises build hub-and-spoke networks and connect on-premises networks (via Direct Connect or VPN) to hundreds of VPCs.

---

### Q1. What is a VPC and why do we need it? Doesn't AWS already isolate customers?

AWS isolates customers from *each other* at the hypervisor level, but a VPC gives **you** control over **your own** network topology: your IP addressing plan, your subnetting, your routing, your firewall rules, and your connectivity to on-premises. Without a VPC you couldn't, for example, guarantee that your database has no route to the internet, or connect your AWS environment to your corporate data center with non-overlapping IP space. A VPC is a *logically isolated software-defined network* — every packet is governed by rules you define (route tables, security groups, NACLs), and that's the foundation of any compliance story (PCI, HIPAA, GxP): you can *prove* what can talk to what.

### Q2. Walk me through what happens when an EC2 instance in a private subnet makes a request to the internet.

This is a classic. Answer it as a packet journey:

1. The instance sends a packet to, say, 142.250.x.x (google.com). Its OS routing table says "not local → send to default gateway," which is the VPC router (the .1 address of the subnet).
2. The VPC router consults the **subnet's route table**. Longest prefix match: 142.250.x.x doesn't match the local VPC CIDR route, so it falls to 0.0.0.0/0 → which points to a **NAT Gateway** (because this is a private subnet).
3. The NAT Gateway (which lives in a **public** subnet) performs *source NAT*: it rewrites the packet's source IP from the instance's private IP (10.0.2.15) to the NAT Gateway's **Elastic IP**, and records the connection in its translation table.
4. The packet leaves via the public subnet's route table → 0.0.0.0/0 → **Internet Gateway** → internet.
5. The response comes back to the NAT Gateway's Elastic IP; the NAT Gateway looks up its translation table, rewrites the destination back to 10.0.2.15, and forwards it.
6. Return traffic is allowed automatically because **security groups are stateful** — the outbound connection's return packets are permitted without an inbound rule.

Key insight to mention: the NAT Gateway must be in a public subnet with a route to the IGW, and the private subnet's route table must point to the NAT Gateway. Two route tables are involved.

### Q3. Why is a NAT Gateway placed in a public subnet? What happens if you put it in a private one?

The NAT Gateway's entire job is to relay traffic to the internet, and the only path to the internet is via the Internet Gateway. A subnet only has that path if its route table has 0.0.0.0/0 → IGW — the definition of a public subnet. If you put the NAT Gateway in a private subnet, it would receive traffic from instances but have no route to the IGW itself — traffic would black-hole. This is a real production incident pattern: someone changes a route table, and suddenly all outbound traffic from private subnets silently fails (image pulls fail, package repos time out) with no error from AWS itself. The fix is checking the *route table associated with the NAT Gateway's subnet*.

### Q4. NAT Gateway vs Internet Gateway — when do you use which?

- **IGW**: bidirectional. Resources with public IPs in public subnets can receive inbound connections from the internet AND initiate outbound. One IGW per VPC. No bandwidth limits, no cost, highly available by design.
- **NAT Gateway**: outbound-only initiation. Private resources can call out; internet cannot call in. It's a managed, AZ-scoped appliance you pay for (hourly + per-GB processed).

Rule of thumb: anything that must accept inbound internet traffic (load balancers, bastions) → public subnet behind IGW. Everything else (app servers, databases, EKS worker nodes) → private subnet using NAT for outbound only. This is *defense in depth* — even if a security group is misconfigured, a private instance has no inbound internet path.

### Q5. Why deploy a NAT Gateway per AZ? What's the failure mode if you don't?

A NAT Gateway is a **zonal** resource. If you have one NAT GW in AZ-a and instances in AZ-a, b, and c all route through it:
1. **Cost/latency**: cross-AZ traffic incurs inter-AZ data charges and added latency for every outbound byte.
2. **Blast radius**: if AZ-a fails, instances in b and c lose ALL outbound internet connectivity even though their own AZ is healthy — you've turned a single-AZ failure into a multi-AZ outage.

Best practice: one NAT Gateway per AZ, and each AZ's private route table points to its **local** NAT Gateway. Mention that you'd codify this in Terraform with a `for_each` over AZs so it can't be done inconsistently.

### Q6. Explain route table evaluation. If a route table has 10.0.0.0/8 → TGW and 10.1.0.0/16 → peering connection, where does traffic to 10.1.5.20 go?

**Longest prefix match**: the most specific matching route wins, regardless of order. 10.1.5.20 matches both routes, but /16 is more specific than /8, so it goes to the **peering connection**. This is the foundation of *routing surgery* during migrations: you can carve out a more specific route to redirect a slice of traffic (e.g., one subnet) while the broader aggregate route keeps handling everything else. Also mention: the `local` route (the VPC's own CIDR) is always present and cannot be deleted — intra-VPC traffic never leaves the VPC.

### Q7. What is a Transit Gateway and why does it exist when VPC peering already exists?

VPC peering is point-to-point and **non-transitive**: if A↔B and B↔C are peered, A cannot reach C through B. So connecting N VPCs fully requires N(N−1)/2 peerings — 10 VPCs = 45 peerings, 50 VPCs = 1,225. Each with its own route table entries. Unmanageable.

A **Transit Gateway** is a regional, managed *transit hub*: each VPC creates one **attachment** to the TGW, and the TGW routes between attachments. It also terminates **VPN** and **Direct Connect** attachments, so it's the single point where on-premises meets all your VPCs. It supports **TGW route tables** (route domains): you can associate different attachments with different route tables to implement segmentation — e.g., prod VPCs can reach shared-services but not dev; every spoke can reach the inspection VPC but not each other directly.

### Q8. Explain the "two hops" of Transit Gateway routing. Why do people get this wrong?

Traffic through a TGW is routed **twice**, and both must be correct:

1. **VPC side**: the source subnet's route table must have a route for the destination CIDR → the TGW attachment (e.g., 10.2.0.0/16 → tgw-xxxx).
2. **TGW side**: the TGW route table *associated with the source VPC's attachment* must have a route for the destination CIDR → the destination VPC's attachment.

And then the reverse path needs the same two hops for return traffic. People get it wrong because they add the VPC route, see traffic leave the VPC, and assume it's done — but the TGW route table silently drops it (a *blackhole*). Debugging tool: **VPC Reachability Analyzer** traces the whole path and tells you exactly which hop lacks a route. Also mention **TGW route table propagation** — attachments can auto-propagate their CIDRs into a TGW route table, vs *static* routes for controlled environments where you want explicit approval of every path.

### Q9. How would you troubleshoot: "Instance in VPC-A cannot reach a service in VPC-B via TGW"?

Give a layered, systematic answer — this is what "senior" sounds like:

1. **Confirm scope**: is it one instance, one subnet, or all of VPC-A? One port or all traffic? (ping vs application port — ICMP is often blocked while TCP works.)
2. **Source security group**: does it allow outbound to VPC-B's CIDR? (Usually yes — default SG allows all outbound.)
3. **Source subnet route table**: is there a route for VPC-B's CIDR → TGW attachment?
4. **TGW route table** associated with VPC-A's attachment: route to VPC-B's attachment? Not blackholed?
5. **Return path**: VPC-B's TGW route table and subnet route tables must route back to VPC-A. *Asymmetric routing* — forward path works, return path missing — is the classic TGW failure.
6. **Destination security group**: allows inbound from VPC-A's CIDR on the right port?
7. **NACLs**: rarely the issue (default allows all), but NACLs are *stateless* — they need BOTH inbound and outbound rules for return traffic (including ephemeral ports 1024–65535).
8. Confirm with **VPC Reachability Analyzer** and **VPC Flow Logs** (look for REJECT records — they tell you the ENI and direction where the packet died).

### Q10. Security Groups vs NACLs — explain the difference like I'm going to be paged at 2 AM.

- **Security Group**: attached to the ENI (the instance/pod/LB network interface). **Stateful** — if you allow the request in, the response is automatically allowed out. Only *allow* rules. All rules evaluated together.
- **NACL**: attached to the **subnet**. **Stateless** — return traffic needs its own explicit rule, including the ephemeral port range. Supports *deny* rules. Rules evaluated in numeric order, first match wins.

At 2 AM: check security groups first (they're the problem 90% of the time), but if a security group looks right and traffic still dies, check whether someone added a NACL deny rule, or whether a stateless NACL is dropping return traffic on ephemeral ports. NACLs are best kept at default (allow-all) and used only as a coarse *subnet-level emergency block* (e.g., blocking a scanning IP during an incident) — fine-grained control belongs in security groups.

### Q11. What are VPC endpoints and why would you use them instead of routing to S3 over the internet via NAT?

Three reasons: cost, security, compliance.

- **Gateway endpoints** (S3, DynamoDB only): a route-table entry that sends S3-bound traffic across AWS's private network. Free. Without it, private instances reach S3 *through the NAT Gateway* — and NAT charges per GB, so heavy S3 traffic (backups, data lakes, Docker layer caches) through NAT is an enormous silent cost.
- **Interface endpoints** (PrivateLink, nearly every AWS API): an ENI in your subnet with a private IP; DNS for the service resolves to that private IP. Traffic to ECR, CloudWatch, SSM, Secrets Manager etc. never touches the internet.
- **Security/compliance**: you can attach **endpoint policies** ("this endpoint can only access these specific buckets") and prove data never traversed the public internet — vital in regulated environments. In locked-down enterprises (like ones using Zscaler and no direct internet), interface endpoints are how workloads reach AWS APIs *at all*.

### Q12. How do you decide a VPC's CIDR range in an enterprise? Why does it matter so much?

Because **IP space is the one thing you can't easily change later**, and overlapping CIDRs break everything: you cannot peer or TGW-route between VPCs with overlapping ranges (the local route always wins — traffic for an overlapping range never leaves the VPC). In an enterprise:
- IP allocation is centrally managed (an IPAM — IP Address Management — team or AWS's IPAM service) so every VPC, every on-prem site, and every acquired company gets non-overlapping space.
- Size for growth: EKS especially devours IPs (every pod gets a VPC IP with the AWS VPC CNI). Undersized VPCs cause pod scheduling failures when subnets run out of IPs — a real incident pattern I'd watch for.
- Plan subnets per AZ per tier (public / private-app / private-data), leaving headroom.
This matters directly to migrations: an Org-to-Org migration often collides with overlapping CIDRs, forcing re-IP or NAT tricks — the most painful part of any migration plan.

### Q13. Why can't two VPCs with overlapping CIDRs communicate even via TGW?

Routing is destination-based. If VPC-A is 10.0.0.0/16 and VPC-B is also 10.0.0.0/16, then from A's perspective a packet destined to 10.0.5.10 matches the **local route** — the VPC router delivers it inside VPC-A (or drops it) and it never reaches the TGW. Even at the TGW, a route table can only have one route per prefix — it can't distinguish "10.0.0.0/16 the first one" from "10.0.0.0/16 the other one." Workarounds exist (private NAT Gateway to translate ranges, or PrivateLink which doesn't require routing at all), but the real answer is *prevent overlap through IPAM discipline*.

---
# SECTION 2: DNS — ROUTE 53 AND ROUTE 53 RESOLVER

## Mental Model First

DNS is the internet's phonebook: names → IP addresses. **Route 53** has three separate jobs people conflate: (1) **domain registration**, (2) **authoritative DNS hosting** (hosted zones that answer "what is the IP for app.example.com?"), and (3) **health-checked traffic routing** (failover, weighted, latency-based answers).

Separately, **Route 53 Resolver** is the *recursive* side — the thing inside your VPC that answers DNS questions *for your instances*. Every VPC has it built in at the **VPC CIDR base + 2** address (e.g., 10.0.0.2), often called the ".2 resolver" or AmazonProvidedDNS. The hard enterprise problem it solves: **hybrid DNS** — how does an EC2 instance resolve `db.corp.internal` (on-prem name), and how does an on-prem server resolve `app.aws.internal` (private zone in AWS)? Answer: **Resolver endpoints** (inbound/outbound) + **forwarding rules**.

**TTL (time to live)** is how long resolvers cache an answer. TTL discipline is THE critical skill in migration cutovers: lower TTL days before the change, flip the record, traffic follows within seconds, raise TTL after.

---

### Q14. Walk me through a full DNS resolution — what happens when an EC2 instance looks up app.example.com?

1. The instance's OS sends the query to its configured resolver — in a VPC, that's the **.2 resolver** (Route 53 Resolver) via DHCP options.
2. The Resolver checks, in order: is this name in a **private hosted zone** associated with this VPC? Does it match a **forwarding rule** (send it to on-prem DNS servers)? Otherwise, resolve it publicly.
3. Public resolution is recursive: root servers → .com TLD servers → the domain's authoritative name servers (which, if hosted in Route 53, are the four NS servers of the public hosted zone).
4. The authoritative server returns the record (A, AAAA, CNAME, ALIAS...), the resolver **caches it for the TTL**, and returns it to the instance.

Bonus point: the .2 resolver only answers queries from within the VPC — that's why on-prem servers can't just query it directly, and why **inbound resolver endpoints** exist.

### Q15. Public vs private hosted zones — what's the difference and how does split-horizon DNS work?

A **public hosted zone** answers queries from the whole internet. A **private hosted zone** answers only from VPCs you've explicitly **associated** with it. **Split-horizon**: you can have the SAME name in both — `app.example.com` publicly resolves to a CloudFront distribution, but inside your VPCs the private zone answers with an internal load balancer IP. Internal traffic stays internal (lower latency, no internet exposure), external users get the public path. Association is per-VPC — a common gotcha is a new VPC that "can't resolve internal names" simply because nobody associated the private zone with it.

### Q16. Explain Route 53 Resolver inbound and outbound endpoints. When do you need each?

They solve hybrid DNS in the two directions:

- **Outbound endpoint + forwarding rules**: AWS → on-prem. Workloads in VPCs need to resolve `*.corp.internal` which lives on on-prem DNS servers (Active Directory, Infoblox). You create an outbound endpoint (ENIs in your VPC) and a rule: "for domain corp.internal, forward to 192.168.1.10 and .11." Rules can be shared across the whole Org via **AWS RAM**, so you define hybrid DNS once, centrally.
- **Inbound endpoint**: on-prem → AWS. On-prem servers need to resolve names in your **private hosted zones**. They can't reach the .2 resolver, so you create an inbound endpoint (ENIs with fixed private IPs in the VPC), and configure the on-prem DNS servers with a *conditional forwarder*: "for aws.internal, ask 10.0.1.53 and 10.0.2.53."

In enterprises, both endpoints usually live in a central *shared-services / networking VPC* attached to the TGW, so all spokes and on-prem use one DNS integration point.

### Q17. What is a CNAME vs an ALIAS record, and why can't you CNAME a zone apex?

- **CNAME**: "this name is an alias for that name" — the resolver must do a second lookup. DNS RFCs forbid a CNAME at the **zone apex** (example.com itself) because the apex must carry SOA and NS records, and a CNAME can't coexist with other records.
- **ALIAS** (Route 53-specific): looks like an A record to the outside world. Route 53 resolves the target (ALB, CloudFront, S3 website, API Gateway) *internally* at query time and returns the actual IPs. Works at the apex, free queries, and automatically tracks the target's changing IPs — which is essential because **ALB and CloudFront IPs change constantly**; hardcoding them in an A record is a guaranteed future outage.

Rule: pointing at an AWS resource → ALIAS. Pointing at an external name → CNAME (non-apex only).

### Q18. Explain Route 53 routing policies and when you'd use each.

- **Simple**: one answer, no health checks.
- **Weighted**: split traffic by percentage — the DNS mechanism for *canary releases* and gradual migration cutovers (10% new environment, 90% old, ramp up).
- **Latency-based**: answer with the region closest (by measured latency) to the user — multi-region active/active.
- **Failover**: primary + secondary with a **health check**; when the health check fails, Route 53 answers with the secondary — DNS-level disaster recovery.
- **Geolocation / geoproximity**: route by user location — compliance (EU users → EU region) or content localization.
- **Multivalue answer**: up to 8 healthy records returned, poor-man's client-side load balancing.

Senior nuance: DNS failover is only as fast as **TTL + client behavior**. Some clients (JVMs with default settings, connection pools) cache DNS far beyond TTL, so weighted/failover shifts are never perfectly instant — one reason cutover plans include *forcing connection recycling* or draining.

### Q19. Why is TTL discipline critical in a migration cutover? Give me the runbook.

TTL is a promise to every resolver on the planet: "you may cache this answer for N seconds." If your record has TTL 86400 (24h) and you flip it during a cutover, some users keep hitting the OLD target for up to a day — you can't force them to forget. So:

1. **Days before cutover** (at least longer than the current TTL): lower TTL to something small, e.g., 60s. You must wait out the *old* TTL before the new one is universally in effect.
2. **Cutover**: change the record. Within ~60s, resolvers worldwide expire their cache and pick up the new answer.
3. **Validate**: dig from multiple vantage points, watch traffic/metrics shift on both sides (old side draining to zero, new side ramping).
4. **Rollback plan**: because TTL is low, rolling back is also fast — flip the record back.
5. **After stability**: raise TTL back (300–3600s) to reduce query load/cost and resolver churn.

Mention the trap: lowering TTL *during* the incident doesn't help — resolvers holding the old answer won't re-ask until the old TTL expires. TTL work is *pre-work*.

### Q20. How do you debug "DNS resolution is failing for some instances but not others"?

Systematic layers:
1. **What exactly fails?** `dig app.internal` vs `dig app.internal @10.0.0.2` vs `dig @8.8.8.8` — this separates OS resolver config problems from VPC resolver problems from the record itself.
2. **VPC attributes**: `enableDnsSupport` and `enableDnsHostnames` must be true — a surprisingly common root cause in Terraform-created VPCs where someone forgot the flags.
3. **Private zone association**: is the private hosted zone associated with *this* VPC? "Works in VPC-A, fails in VPC-B" is almost always association.
4. **Forwarding rules**: is a resolver rule capturing this domain and forwarding it somewhere broken? Check outbound endpoint health, security groups on the endpoint ENIs (UDP/TCP 53), and whether the on-prem targets answer.
5. **Caching**: is a stale answer cached (TTL)? Compare authoritative answer (`dig @ns-xxx.awsdns...`) vs resolver answer.
6. **Resolver query logging**: Route 53 Resolver query logs show every query, the response code, and which rule handled it — the definitive evidence.

---

# SECTION 3: SSL/TLS, ACM, AND CERTIFICATE LIFECYCLE MANAGEMENT

## Mental Model First

TLS solves three problems: **encryption** (nobody can read the traffic), **integrity** (nobody can tamper with it), and **authentication** (you're really talking to the server you think you are). Authentication is what certificates are for: a certificate is a signed statement from a **Certificate Authority (CA)** saying "this public key belongs to app.example.com." Your browser/OS ships with a list of trusted **root CAs**; trust flows down a **chain**: root CA → intermediate CA → your leaf certificate.

**ACM (AWS Certificate Manager)** issues free public certificates that AWS **auto-renews** — but only for use on AWS-managed endpoints (ALB, CloudFront, API Gateway); you can never export the private key. Certificate **lifecycle management** = issuance → validation → deployment → monitoring expiry → renewal/rotation → revocation. Expired certificates are one of the most common self-inflicted enterprise outages, which is why this is in the JD.

---

### Q21. Walk me through a TLS handshake at a level you'd use to debug it.

For TLS 1.2 (know this; then mention 1.3 improvements):
1. **ClientHello**: client sends supported TLS versions, cipher suites, and crucially the **SNI (Server Name Indication)** — the hostname it wants, in plaintext. SNI is how one load balancer/IP serves many certificates: it picks the cert matching the SNI.
2. **ServerHello + Certificate**: server picks the cipher suite and sends its certificate **chain** (leaf + intermediates).
3. **Client validates the chain**: signature by a trusted root? Not expired? Hostname matches CN/SAN? Not revoked?
4. **Key exchange** (ECDHE for *forward secrecy* — session keys aren't derivable from the server's private key even if it later leaks), both sides derive the same **symmetric session key**.
5. All further traffic is symmetric encryption (AES-GCM) — asymmetric crypto is only used for the handshake because it's slow.

**TLS 1.3**: one round trip instead of two, only forward-secret key exchanges allowed, legacy weak ciphers removed, and most of the handshake itself is encrypted.

Debugging tools to name: `openssl s_client -connect host:443 -servername host` (shows the chain the server actually sends, verification errors, protocol negotiated), and `curl -vI https://...`.

### Q22. A client gets "certificate not trusted" but the cert is valid and not expired. Most likely cause?

**Missing intermediate certificate.** The server must send the *full chain* (leaf + intermediates); browsers often paper over a missing intermediate (via caching or AIA fetching), but programmatic clients (curl, Java, Python) fail hard. Verify with `openssl s_client` — it prints the chain sent. Other causes to list: client trust store missing/outdated root (old OS images, container base images without ca-certificates), hostname mismatch (cert for `example.com` but client hit `www.example.com` and there's no SAN for it), client clock skew (cert appears "not yet valid"), or — very relevant in Zscaler enterprises — **TLS inspection re-signing the cert with the corporate CA**, and this particular client doesn't trust the corporate root (see Zscaler section).

### Q23. What is ACM and what are its key constraints?

ACM issues and manages **free public TLS certificates** with two defining properties:
1. **Managed auto-renewal**: as long as validation remains in place, ACM renews before expiry — eliminating the #1 cause of cert outages.
2. **Non-exportable private keys**: the key never leaves AWS; certs can only be attached to **integrated services** — ALB/NLB, CloudFront, API Gateway. You cannot install an ACM cert on an EC2 instance's nginx or on-prem.

Constraints worth naming: it's **regional** — a cert must exist in the same region as the ALB using it, and CloudFront specifically requires certs in **us-east-1**. Validation is via **DNS** (preferred — a CNAME record ACM checks; leave it in place and renewals are fully automatic) or email (legacy, manual, avoid). For internal/private PKI there's **ACM Private CA** (paid) issuing certs for internal names. If you need the key elsewhere, you use a third-party or private CA and *import* into ACM — but imported certs are NOT auto-renewed, so they need expiry monitoring.

### Q24. How do you manage certificate lifecycle at enterprise scale — hundreds of certs, multiple teams?

Talk process + automation:
- **Inventory**: you can't manage what you can't see. AWS Config / Security Hub rules to detect certs nearing expiry; a central register of every cert, owner, where it's deployed, and how it renews.
- **Prefer auto-renewal paths**: ACM with DNS validation for anything on AWS edges; cert-manager with Let's Encrypt or a private CA for Kubernetes ingress (it handles issuance and rotation as CRDs — I've worked with this pattern on EKS).
- **Monitoring as backstop**: CloudWatch/ACM emits `DaysToExpiry`; alert at 45/30/14 days. External blackbox checks (probing the actual endpoint) catch the case where the cert renewed in ACM but was never *deployed* to the thing serving traffic.
- **Rotation as routine change**: renewing a cert should be a no-drama, automated or runbooked change — not a heroic event. In controlled environments, pre-approved *standard changes* in ServiceNow.
- **Chain and trust store hygiene**: when a CA rotates intermediates or a root nears expiry, coordinate trust store updates across old OS images, containers, and appliances — this is the kind of "network and security change execution" the JD refers to.

### Q25. Where can TLS terminate in an AWS architecture, and what are the trade-offs?

- **CloudFront edge**: closest to users, offloads handshakes globally.
- **ALB**: the common choice — ACM cert on the listener, ALB decrypts, inspects (enabling WAF, path routing), and forwards to targets over HTTP or re-encrypted HTTPS.
- **NLB**: TLS termination at layer 4, or pure TCP passthrough when the *backend* must terminate (end-to-end encryption requirements, client-cert auth at the app).
- **The pod/instance itself** (e.g., nginx ingress or a service mesh with mTLS): true end-to-end encryption; required in strict compliance environments where traffic must be encrypted even inside the VPC.

Trade-off summary: terminating earlier = better performance and centralized cert management, but plaintext exists behind that point; terminating later = end-to-end encryption but you manage certs on more things. Many regulated setups do **TLS termination at ALB + re-encryption to targets** as the balance.

---

# SECTION 4: EDGE AND API LAYER — CLOUDFRONT, API GATEWAY

## Mental Model First

**CloudFront** is AWS's CDN: ~600+ edge locations worldwide that cache content close to users and carry non-cacheable traffic over AWS's private backbone instead of the public internet. Everything revolves around: **origins** (where content comes from — S3, ALB, API Gateway), **cache behaviors** (path-pattern rules deciding caching/TTL/allowed methods), and the **cache key** (which parts of a request — path, headers, query strings, cookies — define "the same object").

**API Gateway** is a managed front door for APIs: it handles auth, throttling, request validation, and routing to backends (Lambda, ECS, anything HTTP) so your services don't each reimplement that. Know the difference between **REST API** (feature-rich, older) and **HTTP API** (cheaper, faster, simpler), and how **VPC Link** lets the gateway reach *private* services inside your VPC.

---

### Q26. How does CloudFront actually work when a user requests a file?

1. DNS (Route 53 ALIAS) resolves your domain to CloudFront; the user is routed to the **nearest edge location**.
2. Edge checks its cache for the object (identified by the **cache key**). **Cache hit** → served immediately from the edge.
3. **Cache miss** → the edge fetches from a **regional edge cache** (a bigger mid-tier), and on miss there, from the **origin** — over AWS's backbone, not the public internet.
4. The response is cached per the TTL rules (Cache-Control headers from origin, or the behavior's min/default/max TTL) and served.

Two things people miss: CloudFront helps **dynamic, uncacheable traffic too** (TLS terminates at the edge; the origin fetch rides the AWS backbone with persistent connections — lower latency globally), and the **cache key** design is everything: forward only the headers/query strings that genuinely change the response, or your hit ratio collapses because every variant is a different object.

### Q27. How do you secure the origin so users can't bypass CloudFront and hit the ALB/S3 directly?

If users can reach the origin directly, they bypass WAF, caching, and geo controls. Patterns:
- **S3 origin**: **Origin Access Control (OAC)** — the bucket policy allows reads only from your specific CloudFront distribution (signed by CloudFront's service principal). Bucket stays fully private. (OAC replaced legacy OAI.)
- **ALB origin**: have CloudFront inject a **secret custom header** (e.g., `X-Origin-Verify: <random>`) and a WAF/listener rule on the ALB rejecting requests without it; and/or restrict the ALB security group to the AWS-managed **CloudFront origin-facing prefix list**.
This is a favorite senior question because it tests whether you think about the *whole* attack surface, not just the happy path.

### Q28. What is cache invalidation vs versioned object names, and which do you prefer?

**Invalidation**: telling CloudFront "purge /path/*" so the next request refetches from origin. It takes seconds-to-minutes, and beyond 1,000 free paths/month it costs money. **Versioned names** (app.v2.js, or content-hashed filenames from your build): every deploy references *new* object names, so caches never need purging — old versions age out naturally, and rollback is trivial (point back at old names). Prefer versioning for static assets (it's deterministic and instant); keep invalidation for emergencies (bad content published, immediate purge required). This maps directly to CI/CD: the pipeline builds hashed assets, uploads to S3, then deploys HTML referencing the new hashes.

### Q29. API Gateway — why put it in front of services at all? And REST vs HTTP APIs?

It centralizes cross-cutting concerns so every service doesn't reimplement them: **authentication/authorization** (IAM, Cognito, Lambda authorizers, JWT validation), **throttling and usage plans** (protect backends from abuse, per-client quotas via API keys), **request validation**, **routing/versioning** across stages, caching, and uniform observability (access logs, latency metrics per route).

- **REST API**: the full-featured original — API keys, usage plans, request/response transformation, caching, WAF attachment, private APIs.
- **HTTP API**: newer, ~70% cheaper, lower latency, native JWT authorizers — but fewer features. Default choice unless you need REST-only features.
Also know **WebSocket APIs** exist for bidirectional connections.

### Q30. Your API Gateway needs to reach an ECS service running in private subnets. How?

**VPC Link.** The gateway is a managed service outside your VPC; VPC Link gives it a private path in:
- **HTTP API VPC Link**: creates ENIs in your subnets and can target an **ALB**, NLB, or Cloud Map service directly — the modern, flexible option.
- **REST API VPC Link**: requires an **NLB** in front of the service (a historical constraint worth knowing because plenty of environments still run it: API GW → VPC Link → NLB → ECS targets).
The point to land: no public exposure of the service — the security groups on the ECS tasks only allow the NLB/ALB, the load balancer is internal, and the only public entry is the gateway (with auth, throttling, and WAF).

### Q31. Users report intermittent 502/504 from your CloudFront → API Gateway → ECS stack. How do you localize the failure?

Work from the edge inward, using each layer's telemetry:
1. **Which layer generated the error?** CloudFront access logs (`x-edge-result-type`, origin response codes) tell you whether CloudFront itself erred or relayed an origin error. An `OriginTimeout` vs a relayed 502 are different investigations.
2. **API Gateway** access/execution logs: integration latency vs gateway latency separates "gateway slow" from "backend slow"; 504 at the gateway = integration timeout (max 29s on REST APIs — long-running backend calls WILL 504; that's a design smell to fix with async patterns).
3. **ALB/NLB metrics**: `HTTPCode_ELB_5XX` vs `HTTPCode_Target_5XX` — did the LB fail to get a response (no healthy targets, connection refused) or did the target return the error? Check `UnHealthyHostCount` and target group health reasons.
4. **ECS**: task restarts/OOM kills (a restarting task briefly refuses connections → intermittent 502), deployment events (rolling deploys without proper *connection draining* or with too-aggressive health checks cause exactly "intermittent" errors), and app logs.
5. **Timeout alignment**: a classic root cause — the ALB idle timeout (default 60s) longer than the app's keep-alive timeout causes the ALB to reuse a connection the app already closed → sporadic 502. Rule: each layer's timeout should be *shorter* than the layer in front of it, and the app's keep-alive *longer* than the ALB's idle timeout.

---

# SECTION 5: COMPUTE AND DATA SERVICES — ECS, LAMBDA, STEP FUNCTIONS, GLUE, ATHENA

## Mental Model First

- **ECS**: AWS's container orchestrator (the simpler sibling of Kubernetes). A **task definition** (like a pod spec: image, CPU/memory, env, ports) runs as **tasks**, kept alive and scaled by a **service**, on either **Fargate** (serverless — AWS runs the hosts) or **EC2** launch type (you manage the hosts). Your Kubernetes knowledge transfers almost 1:1 — say that in the interview.
- **Lambda**: run a function on demand; pay per invocation and per ms; AWS handles all servers and scaling. Constraints: 15-min max runtime, stateless, cold starts.
- **Step Functions**: a managed **state machine** for orchestrating multi-step workflows (Lambda calls, ECS tasks, Glue jobs) with retries, error branching, and visual execution history — the "workflow engine" so you don't write orchestration glue code.
- **Glue**: managed **ETL** (serverless Spark) plus the **Glue Data Catalog** — the central metadata store ("this S3 path is a table with these columns") that Athena depends on. **Crawlers** scan data and populate the catalog.
- **Athena**: serverless SQL directly over files in S3 (Presto/Trino engine). No cluster, pay per TB scanned. The trio S3 (storage) + Glue Catalog (schema) + Athena (query) = the standard AWS data lake.

---

### Q32. ECS vs EKS — when would you choose ECS, and how does your Kubernetes experience map?

ECS when: you're all-in on AWS, want deep native integration (IAM per task, ALB, CloudWatch) with far less operational overhead, and don't need the Kubernetes ecosystem (Helm, operators, CRDs) or portability. EKS when: you need Kubernetes APIs, multi-cloud/hybrid consistency, or the ecosystem. Mapping (use this table in your head): task definition ≈ pod spec; service ≈ deployment (+ its service); ECS cluster ≈ k8s cluster; Fargate ≈ "no node management"; capacity provider ≈ node group/autoscaler; task role ≈ IRSA/pod identity; awsvpc network mode ≈ every task gets its own ENI like a pod gets an IP. Since I run workloads on EKS daily, ECS concepts are the same reconciliation model — desired count vs running count, health checks, rolling deployments — with different names.

### Q33. Fargate vs EC2 launch type — trade-offs?

**Fargate**: no hosts to patch, scale, or right-size; per-task isolation; pay per vCPU-second/GB-second. Costs more per unit of compute, no daemonsets/GPU flexibility (limited), no host access for debugging (use ECS Exec instead of SSH). **EC2**: cheaper at steady high utilization (especially with Spot/RIs), full control of the host (GPUs, special AMIs, host-level agents), but you own patching, capacity management, and bin-packing. Default answer for a platform team: Fargate for most services (operational simplicity), EC2 for cost-optimized steady workloads or special hardware.

### Q34. How does an ECS rolling deployment work, and what protects you from deploying a broken image?

The service scheduler replaces tasks gradually governed by **minimumHealthyPercent** and **maximumPercent** (e.g., 100/200 = start new tasks before stopping old). A new task only receives traffic after passing the **target group health check**; if new tasks fail health checks, the **deployment circuit breaker** (enable it!) marks the deployment failed and **auto-rolls back** to the last working task set. Also available: **blue/green via CodeDeploy** — a parallel task set behind a second target group, test traffic on a test listener, then shift (all-at-once, linear, or canary) with instant rollback by shifting back. Mention *deregistration delay* (connection draining) so in-flight requests finish before old tasks die — skipping this causes 5xx blips during every deploy.

### Q35. What is a Lambda cold start and how do you mitigate it?

On first invocation (or scale-out), AWS must create the execution environment: provision the sandbox, download code, start the runtime, run your init code — that's the **cold start** (tens of ms to seconds; worst with big packages, VPC-attached functions historically, and JVM runtimes). Warm environments are reused, so steady traffic mostly avoids it. Mitigations: smaller deployment packages and lazy imports; move heavy init outside the handler (it runs once per environment, not per invocation); **provisioned concurrency** (pre-warmed environments — costs money, use for latency-critical paths); choose faster-initializing runtimes; SnapStart for Java. Also know: Lambda scales by running **one request per environment at a time** — concurrency = number of environments — and account-level concurrency limits can throttle you (429s) during spikes.

### Q36. Why Step Functions instead of one big Lambda (or Lambda calling Lambda)?

Because orchestration in code is where reliability goes to die: a 15-minute Lambda limit, no durable state if something crashes mid-flow, hand-rolled retries, and no visibility into where a multi-step process failed. Step Functions gives you: **durable state** (executions can run up to a year — Standard workflows), **per-step retry/backoff/catch** policies declared in the state machine, **visual execution history** (you see exactly which step failed with what input — priceless for ops), parallel branches and **Map** for fan-out, and direct **service integrations** (call ECS, Glue, SQS, DynamoDB without glue Lambdas). Also mention the **`.sync` integration pattern** — Step Functions can start an ECS task or Glue job and *wait for it to complete* — and **Standard vs Express** workflows (Express: high-volume, short-lived, cheaper, at-least-once execution semantics).

### Q37. Explain how Athena, Glue, and S3 fit together — and how you make Athena fast and cheap.

S3 holds the raw files; the **Glue Data Catalog** holds table definitions (schema, format, partition layout, S3 location); Athena reads catalog metadata and scans S3 to execute SQL, charging **per TB scanned**. So performance and cost are the same lever — scan less:
1. **Partitioning**: lay out data as `s3://bucket/table/year=2026/month=07/day=08/`; queries filtering on partition keys skip everything else (*partition pruning*).
2. **Columnar formats**: Parquet/ORC instead of CSV/JSON — Athena reads only the referenced columns, plus compression and predicate pushdown; typical 10–100x scan reduction.
3. **File sizing**: avoid millions of tiny files (S3 request overhead dominates); aim for ~128MB–1GB files — compaction jobs (Glue) fix this.
4. **Partition projection** for high-cardinality date partitions — avoids slow metadata lookups and endless `MSCK REPAIR TABLE`.
Common failure to know: **crawler-drifted schemas** or `HIVE_PARTITION_SCHEMA_MISMATCH` when files' schemas evolved inconsistently — the fix is schema discipline at write time, not crawler tweaking.

### Q38. When Glue vs Lambda vs ECS for data processing?

**Lambda**: small, event-driven transforms (< 15 min, < 10GB memory) — e.g., per-file processing on S3 upload. **Glue**: heavy, distributed ETL over large datasets — it's managed Spark; also the right tool when the output must land in the Data Catalog. **ECS/Fargate task**: long-running or custom-runtime processing that isn't Spark-shaped and exceeds Lambda limits — arbitrary containers, your own libraries. Orchestrate any mix with Step Functions. Interviewers want to hear you pick by *constraints* (duration, scale, runtime needs), not by fashion.

---

# SECTION 6: MIGRATION EXPERIENCE — THE HEART OF THIS JD

## Mental Model First

Every migration, regardless of technology, is the same five-phase discipline:
1. **Discovery** — inventory everything; the thing you didn't know existed is what causes the outage.
2. **Design** — target state + *coexistence* state (both environments running in parallel).
3. **Migration waves** — move in small, low-risk-first batches, never big-bang.
4. **Cutover** — a rehearsed, time-boxed switching event with explicit go/no-go criteria and rollback.
5. **Decommission** — only after validation; keep the old path warm during a rollback window.

The senior signals interviewers listen for: **rollback plan before change plan**, **TTL/prerequisite pre-work**, **testing in count/monitor mode before enforce mode**, and **communication/change-management discipline**.

---

### Q39. Describe how you'd plan an AWS tenant migration from one AWS Organization to another. (THE big question for this JD)

Structure your answer in phases:

**Why it happens**: mergers/divestitures, consolidating orgs after acquisitions, or moving to a new landing zone with better governance.

**Phase 1 — Discovery & assessment**: inventory every account and workload; map dependencies that *break* when an account changes Organizations:
- **Consolidated billing** and RI/Savings Plan sharing (discounts pooled at the org level — moving an account changes its economics).
- **SCPs**: the account leaves the old org's SCPs and inherits the new org's — anything the new SCPs deny that workloads rely on will break *at the moment of the move*. Diff the SCP sets first.
- **Org-level integrations**: CloudTrail org trails, AWS Config aggregators, GuardDuty/Security Hub delegated administrators, IAM Identity Center (SSO) — all sever on move and must be re-established in the new org.
- **RAM shares** (TGW! Resolver rules! subnets!): resources shared *from* the old org stop being shared to a departed account. If your VPC attaches to a TGW shared via RAM from the old org's networking account, **your network breaks** on move unless re-shared or re-architected.
- **Cross-account IAM trust**, KMS key policies, S3 bucket policies referencing `aws:PrincipalOrgID` — any policy conditioned on the old org ID silently starts denying.

**Phase 2 — The mechanics**: two fundamentally different strategies:
- **Account migration (lift the account)**: the account itself moves — remove from old org (it briefly becomes standalone: must have its own payment method, support plan), then accept an invitation from the new org's management account. Resources, resource IDs, and data don't move at all — *identity and governance context* changes. Lower effort, but you inherit the account's history and the entire dependency-severing list above.
- **Workload migration (rebuild and move data)**: new accounts in the new org, redeploy infrastructure with **Terraform** (this is where IaC pays for itself — the same modules stamp out the new environment), then migrate data (S3 replication/DataSync, RDS snapshots or DMS for minimal downtime, etc.) and cut over via DNS. Higher effort, but you land clean in the new org's standards.

**Phase 3 — Execution in waves**: dev/sandbox accounts first to discover unknowns cheaply; then low-criticality prod; then crown jewels. Per wave: pre-checks → change window → migrate → validate (network reachability, SSO login, monitoring flowing, app health) → soak period.

**Phase 4 — Cutover discipline**: for workload migrations, DNS-weighted shifting with pre-lowered TTLs; explicit rollback plan (flip DNS back; for account moves, the rollback is re-inviting to the old org — rehearse it).

**Phase 5 — Decommission** after a defined validation window; final billing reconciliation; update the CMDB and IAM Identity Center assignments.

If asked what's hardest: **the invisible org-scoped dependencies** — RAM shares and `PrincipalOrgID` conditions — because nothing warns you; things just stop working at move time. That's why discovery includes grepping every IAM/KMS/S3 policy for the old org ID.

### Q40. How would you migrate observability from Splunk to Google Cloud Observability and Google SecOps?

First, separate the two migrations hidden in this sentence:
- **Operational observability** (logs/metrics/traces for engineers) → **Google Cloud Observability** (Cloud Logging, Cloud Monitoring, formerly Stackdriver).
- **Security telemetry** (SIEM: detections, investigations, compliance retention) → **Google SecOps** (formerly Chronicle) — Google's SIEM that ingests security logs at fixed-cost scale, normalizes them into **UDM (Unified Data Model)**, and runs **YARA-L detection rules** over them.

**Plan:**
1. **Inventory the Splunk estate**: every ingestion source (forwarders, HEC endpoints, syslog, CloudTrail/VPC Flow Logs pipelines), every index, and critically **every consumer**: dashboards, saved searches, **alerts** (each is an SPL query someone depends on), and SOC **detection rules**. The searches/alerts inventory IS the migration backlog.
2. **Classify each source**: engineer-facing → Cloud Observability; security-facing (auth logs, CloudTrail, EDR, firewall, Zscaler logs) → SecOps. Some go to both.
3. **Re-plumb ingestion**: replace Splunk forwarders/HEC with the **Google Ops Agent** on VMs, native GCP service logging, and for AWS-side sources, pipelines shipping CloudTrail/VPC Flow Logs/ALB logs to Google (SecOps has native feeds/ingestion APIs for AWS sources; Bindplane/forwarders for the rest). Design this so sources are switched *per-source*, not big-bang.
4. **Translate the query layer** — the real work: SPL → Cloud Logging query language / Log Analytics (SQL) for ops content; SPL detection rules → **YARA-L** rules in SecOps. Not mechanical translation — re-validate each detection fires on known-true test events. Prioritize by usage (Splunk audit logs tell you which searches/dashboards are actually used — typically a huge fraction are dead and shouldn't be migrated at all).
5. **Dual-run**: ship to both platforms for a defined overlap window; compare event counts per source (catching silent ingestion loss), alert parity (same incidents detected by both), and dashboard equivalence. Dual-run costs double ingestion — time-box it deliberately.
6. **Retention/compliance**: security logs often carry 1–7 year retention mandates. Either bulk-migrate historical data or (commonly) keep Splunk in read-only "cold" mode until old data ages out — decommissioning ingestion but retaining search, which also slashes license cost immediately.
7. **Cutover & decommission**: point alert routing/on-call and SOC workflows to the new platforms, freeze Splunk changes, then decommission ingestion, then the platform.

**Risks to volunteer**: silent gaps in security coverage during transition (mitigate with dual-run and detection parity testing), SPL knowledge concentrated in a few people, alert fatigue from mis-tuned translated rules, and per-source parsing/normalization surprises (UDM field mapping).

### Q41. Describe how you'd execute a coordinated network/security change involving Route 53, certificates, TGW, DNS, Zscaler, and WAF.

This JD line describes a *cutover event* — e.g., moving an application behind new network entry points during a migration. Show the discipline:

**Pre-work (days/weeks ahead):**
- **DNS**: lower TTLs on every record that will change (wait out old TTLs).
- **Certificates**: issue/validate ACM certs for the new endpoints *in advance* (DNS validation records in place, CloudFront certs in us-east-1); verify full chains with `openssl s_client` against a staging endpoint.
- **TGW**: pre-build attachments and *stage* routes for the new path — where possible, add more-specific routes only at cutover so rollback = delete one route.
- **Zscaler**: this is the one teams forget — if users reach the app through Zscaler (ZIA for internet-bound, ZPA for private app access), the new endpoint/domain must be in Zscaler policy *before* cutover: SSL-inspection exemptions or cert trust for the new domain, ZPA app segments updated with new FQDNs/IP ranges, firewall policy for new destinations. Otherwise the change "works from AWS" and fails for every corporate user.
- **WAF**: deploy the WAF ACL on the new entry point in **count mode** first — rules log matches but don't block. Bake it against real traffic (or mirrored/synthetic traffic), review false positives, THEN switch to block. Never cut over onto a brand-new WAF in block mode.

**Change window:**
- Frozen, rehearsed runbook with named owners per step, explicit *validation checkpoints* and a **go/no-go decision point** with rollback criteria decided *in advance* (so you're not negotiating rollback at 2 AM under pressure).
- Execute: TGW route flip → DNS record change → verify propagation (dig from multiple resolvers, inside and outside Zscaler) → synthetic transactions through the full path (corporate user via Zscaler, external user via CloudFront/WAF) → watch WAF logs, ALB 5xx, cert handshake errors, TGW flow logs.

**Post:**
- Soak with heightened monitoring; raise TTLs after stability; move WAF from count to block per plan; document actuals vs plan; only then decommission old paths.

In a controlled enterprise, all of this rides a **change management process** (ServiceNow CR, CAB approval, stakeholder comms, peer-reviewed runbook) — say that explicitly; this JD's "controlled enterprise environments" phrase is testing for it.

### Q42. What are SCPs, and how do you safely change them in a controlled enterprise environment?

**Service Control Policies** are Organization-level guardrails attached to the org root, OUs, or accounts. Critical mental model: **SCPs never grant permissions — they set the maximum boundary.** Effective permission = (identity IAM policy allows) AND (no SCP denies). They apply to everyone in the account *including the root user*, which is why they're the enforcement layer for non-negotiables: deny leaving the org, deny disabling CloudTrail/GuardDuty, deny creating resources outside approved regions, deny deleting KMS keys, require encryption.

**Why changes are dangerous**: an SCP change instantly affects *every principal in every account underneath it* — human, CI/CD pipelines, and service roles alike. A slightly-too-broad deny can break every deployment pipeline in the org simultaneously. And SCP denials surface to teams as confusing generic `AccessDenied` errors (with "due to a Service Control Policy" in the message if you know to look).

**Safe change workflow:**
1. **SCPs live in Terraform/IaC** — version-controlled, peer-reviewed via MR, never hand-edited in the console.
2. **Impact analysis before merge**: query **CloudTrail/Athena for the last 90 days** — "who currently performs the actions this policy would deny?" If the answer isn't "nobody," you've found the breakage in advance. IAM Access Analyzer policy checks validate syntax and can compare policy versions for unintended new denials.
3. **Test the deny logic** in a sandbox OU with a canary account replicating real workload roles.
4. **Staged rollout**: apply to sandbox OU → dev OU → prod OU, with a soak period and monitoring for `AccessDenied` spikes (CloudTrail metric filter/alert on SCP-attributed denials) between stages.
5. **Change management**: CR with the policy diff, blast-radius statement, rollback plan (revert the Terraform MR — and note SCP changes take effect quickly, so rollback is fast too).
6. **Beware policy size/structure limits** (SCP document size caps, inheritance behavior: a deny anywhere in the chain wins; the default `FullAWSAccess` allow must remain or everything is implicitly denied — removing it is the classic catastrophic mistake).

### Q43. In any migration, how do you decide wave grouping and ordering?

By **risk and dependency**: (1) dependency-map first — things that must move together (chatty app+DB pairs) form a *move group*; (2) start with low-blast-radius, high-learning workloads (dev/internal tools) to burn down unknowns; (3) sequence shared infrastructure (networking, DNS, identity) *before* the workloads that depend on it; (4) schedule crown-jewel/regulated workloads last, after the runbook is battle-tested; (5) keep waves small enough that a failed wave is absorbable and rollback is credible. And between every wave: a retro that updates the runbook — the runbook version at wave 10 should look meaningfully better than at wave 1.

---

# SECTION 7: NETWORKING FUNDAMENTALS — LOAD BALANCING, ZSCALER, WAF, TROUBLESHOOTING

## Mental Model First

- **Load balancing**: distribute traffic across healthy targets. **L4 (NLB)** balances TCP/UDP connections — fast, preserves source IP, static IPs. **L7 (ALB)** understands HTTP — routes on host/path/headers, terminates TLS, integrates WAF and auth.
- **Zscaler**: a cloud security proxy enterprises route ALL user traffic through. **ZIA** (Internet Access) = secure web gateway for internet-bound traffic, including **TLS inspection** (it decrypts, inspects, re-encrypts traffic — re-signing with a corporate CA). **ZPA** (Private Access) = zero-trust access to internal apps *without* a VPN — users never get network access, only brokered per-app connections via lightweight **App Connectors** deployed near the apps (e.g., in your VPCs).
- **WAF**: a layer-7 firewall inspecting HTTP requests against rules (SQLi, XSS, bad bots, rate limits) attached to CloudFront/ALB/API Gateway. Golden operational rule: **count mode first, then block.**

---

### Q44. ALB vs NLB — how do you choose, and how do health checks/target groups work?

**ALB** when you need HTTP intelligence: host/path/header routing to multiple target groups, TLS termination with SNI (many certs on one listener), WAF attachment, OIDC/Cognito auth at the edge, WebSockets/HTTP2/gRPC. **NLB** when you need: raw TCP/UDP or extreme throughput, **static IPs / Elastic IPs** (firewall whitelisting requirements — enterprises love this), **preserved client source IP** at L4, TLS passthrough to backends, or to be a **PrivateLink/VPC Link (REST)** target. **Target groups** hold the registered targets (instances, IPs, Lambda) and the **health check** definition (path, interval, healthy/unhealthy thresholds). Only healthy targets receive traffic; a target failing checks is taken out and, in ECS/ASG contexts, replaced. Two ops gotchas: health check path must be a cheap endpoint that reflects true readiness (not `/` doing a DB call — or a DB blip marks ALL targets unhealthy and the LB fails everything); and **deregistration delay** lets in-flight requests drain before target removal.

### Q45. Explain ZIA vs ZPA, and how Zscaler changes how you troubleshoot.

**ZIA**: users' internet-bound traffic is steered (via the Client Connector agent, PAC files, or GRE/IPsec tunnels from offices) to Zscaler's cloud, where policy is enforced: URL filtering, malware scanning, DLP, cloud firewall — and **TLS inspection**: Zscaler terminates the user's TLS, inspects plaintext, and re-encrypts to the destination, presenting the client a cert signed by the **corporate root CA** (pushed to managed devices' trust stores).

**ZPA**: replaces VPN with zero-trust. The user never joins the network; the **App Connector** (deployed in your VPC/on-prem, making only *outbound* connections to Zscaler — no inbound firewall holes) and the client both dial into Zscaler brokers, which stitch per-application connections based on identity + policy. Apps are defined as **app segments** (FQDNs/IPs + ports). Least-privilege per app, apps are invisible to unauthorized users, lateral movement is structurally prevented.

**Troubleshooting impact** — this is the practical gold:
- "Works from the server, fails for corporate users" (or vice versa) → suspect the Zscaler path: policy block, SSL inspection issue, or a missing/incorrect **ZPA app segment** (new FQDN or port not added → connection has no path).
- TLS errors only on corporate devices → SSL inspection: an app or CLI tool (python requests, curl in a container, Java) that doesn't use the OS trust store won't trust the Zscaler-signed cert → cert verification failures. Fixes: add the corporate root to that tool's CA bundle, or an inspection **exemption** for that destination (common for pinned-cert apps, which break under inspection by design).
- New app/domain rollouts must include Zscaler policy work as a **prerequisite step** in the change plan — retro-fitting it after user complaints is the amateur version.
- Source IP surprises: through ZIA, apps see Zscaler egress IPs, not office IPs — breaks naive IP allowlists; through ZPA, the app sees the App Connector's IP.

### Q46. How does AWS WAF work and what's the professional way to roll out rules?

WAF evaluates each HTTP request against a **Web ACL** — an ordered set of rules: **AWS managed rule groups** (Core rule set, Known bad inputs, SQLi, IP reputation, Bot Control), **rate-based rules** (block IPs exceeding N requests per 5 min — the cheap anti-abuse/anti-scrape layer), and custom rules (match on URI, headers, geo, IP sets). Each rule action: allow, block, count, CAPTCHA/challenge. Attach points: CloudFront (global, protects at the edge), ALB, API Gateway, AppSync.

**Professional rollout — count mode first:**
1. Deploy every new rule/rule group in **count mode**: matches are logged (WAF logs → S3/CloudWatch/Firehose) but nothing is blocked.
2. Analyze matches against real production traffic for days: which rules would have blocked *legitimate* requests? (Classic false positives: APIs POSTing JSON that trips SQLi patterns, file uploads tripping size rules.)
3. Tune: add scoped-down statements/exclusions for the false-positive patterns, or label-based overrides on managed rules.
4. Flip to **block** rule-by-rule, watching a dashboard of block counts + app error rates; be ready to revert a single rule to count.
5. Keep monitoring — managed rule groups auto-update, so periodic review of what's newly matching.
Skipping count mode and going straight to block on a production app is how you cause your own outage — interviewers who've lived this will light up when you volunteer the count-first workflow.

### Q47. Give me your general network troubleshooting methodology — "users can't reach the app."

Show a layered, evidence-driven method (OSI bottom-up or path-tracing — pick path-tracing, it sounds like experience):

1. **Scope**: all users or some? (Corporate-only → Zscaler path. One region → edge/DNS. Everyone → the app/entry point.) All endpoints or one path? Since when — correlate with recent changes (deploys, DNS, cert, WAF, route changes — check the change calendar FIRST; most outages are self-inflicted).
2. **DNS**: does the name resolve, and to the *right* answer? `dig` from multiple vantage points; stale TTL? Missing private zone association?
3. **Reachability/transport**: `nc -zv host 443`, traceroute/mtr — connection refused (nothing listening / SG) vs timeout (routing/firewall black hole) are different diagnoses.
4. **TLS**: `openssl s_client` — handshake failure? Wrong/expired cert? Corporate inspection issue?
5. **HTTP layer**: `curl -v` — what status? WAF 403? LB 502/504 (check ELB vs target 5xx metrics)? App 500 (app logs)?
6. **AWS evidence tools**: VPC Flow Logs (ACCEPT/REJECT at each ENI — tells you exactly where packets die), Reachability Analyzer (config-level path verification), ALB access logs, WAF logs.
7. **Bisect the path**: test from a pod → from the node → from another VPC → from outside — each hop that works narrows the failing segment.
State the principle: *change one variable per test, and let logs/flow data — not guesses — drive the next step.*

### Q48. What tools do you actually use at the Linux level for network debugging, and what does each tell you?

- `dig` / `dig +trace`: DNS answers and the full delegation path; `@server` to compare resolvers.
- `curl -v` / `curl --resolve`: full HTTP+TLS transaction; `--resolve` tests a specific backend IP while keeping the Host/SNI right — invaluable for "is it DNS or the server" and pre-cutover testing of the NEW target before DNS flips.
- `nc -zv host port`: pure TCP connectivity — separates network from application.
- `ss -tlnp`: what's actually listening on this host, on which interface, owned by which process (the modern netstat).
- `mtr`: continuous traceroute with loss per hop — where latency/loss enters the path.
- `tcpdump -i any port 443 -w cap.pcap`: ground truth. Did the SYN arrive? Did we answer? Retransmissions (loss)? RSTs (something actively refusing)? Reading a capture settles arguments no dashboard can.
- `ip route get <dst>`: which route/interface the kernel will actually use.
- `conntrack`: NAT/connection-tracking table state — exhaustion causes mysterious drops on busy NAT/proxy nodes.

---

# SECTION 8: STORAGE AND DATA PLATFORMS — EBS, S3, RDS, MongoDB, Snowflake

## Mental Model First

Pick storage by access pattern:
- **EBS** = a network-attached disk for one EC2 instance (block storage; the "hard drive"). Zonal.
- **S3** = infinite object storage over HTTP (files as immutable objects, no filesystem). Regional, 11-nines durable.
- **RDS** = managed relational databases (Postgres/MySQL/etc.) — AWS handles patching, backups, failover.
- **MongoDB** = document database (JSON-like, flexible schema); runs as replica sets for HA and shards for scale.
- **Snowflake** = cloud data warehouse whose defining idea is **separated storage and compute**: data lives once in cloud object storage; independent **virtual warehouses** (compute clusters) spin up/down and scale independently, billed per second of use.

---

### Q49. Explain EBS volume types and the ops issues you've seen with EBS.

Types: **gp3** (general SSD — the default; you provision IOPS/throughput independently of size, unlike gp2 where IOPS scaled with GB — migrating gp2→gp3 is a free-lunch cost win), **io1/io2** (provisioned high IOPS for demanding databases), **st1/sc1** (cheap throughput/cold HDD for sequential workloads). Key properties: **zonal** (attaches only within its AZ — a "volume won't attach" mystery is often an AZ mismatch), snapshots to S3 are **incremental** and are how you move volumes across AZ/region, and resizing is online (grow the volume, then grow the filesystem — `growpart` + `resize2fs`/`xfs_growfs`; forgetting step two is a classic).

Ops issues worth citing (these ring true from EKS work): **IOPS/throughput exhaustion** — a gp2 volume's burst balance drains and the database mysteriously slows (watch `BurstBalance`, `VolumeQueueLength`); **attach/detach stuck states** on Kubernetes — a PV stuck attaching to a new node because the old node didn't cleanly release it (the EBS CSI attach → mount → bind-mount chain), forcing controlled detach; and volumes in `error` state after failed snapshots.

### Q50. S3 — what should a platform engineer know beyond "it stores files"?

- **Consistency**: strong read-after-write now (since 2020) — no more eventual-consistency dance.
- **Storage classes & lifecycle**: Standard → Standard-IA → Glacier tiers via lifecycle rules; **Intelligent-Tiering** when access patterns are unknown. Lifecycle policies are the #1 S3 cost lever; the #2 is cleaning up **incomplete multipart uploads** (invisible in the console, billable — add the lifecycle rule).
- **Security model**: block public access at the account level; bucket policies for cross-account/conditions; **encryption by default** (SSE-S3 or SSE-KMS — KMS gives auditable key usage and lets you enforce `PrincipalOrgID`-style conditions; watch KMS request costs/throttling at high request rates).
- **Versioning + replication**: versioning protects from overwrite/delete (with MFA-delete for the paranoid tier); CRR/SRR for DR, compliance locality, or org migrations (replication is a primary data-move mechanism in Org-to-Org migrations).
- **Performance**: request rates scale per prefix; multipart for large objects; S3 → NAT cost trap avoided with the free **gateway endpoint** (ties back to Section 1).
- **Event-driven patterns**: S3 events → Lambda/SQS/EventBridge — the trigger for ingestion pipelines (relevant to the Glue/Athena stack).

### Q51. How does RDS Multi-AZ differ from read replicas? And what's your backup/restore story?

**Multi-AZ** = availability: a synchronously replicated standby in another AZ; automatic failover (~60–120s) by flipping the endpoint's DNS (apps must reconnect — connection pools that cache DNS forever ride out failovers badly; ties back to TTL/client behavior). The standby serves **no traffic**. **Read replicas** = scale/read-offloading: asynchronous copies that serve reads (with replication **lag** — never read-your-own-writes from a replica), promotable to standalone (a DR and *migration* pattern: replicate → promote → cut over). **Backups**: automated daily snapshots + continuous transaction logs enable **point-in-time recovery** within the retention window (up to 35 days); restores create a **new instance** (you don't restore "in place" — plan the DNS/endpoint swap); manual snapshots persist beyond retention and are **shareable cross-account** — the standard mechanism for moving databases between accounts/orgs. Senior add-on: measure your actual **RTO** by rehearsing restores — an untested backup is a hope, not a plan.

### Q52. What does a platform engineer need to know about MongoDB?

Architecture over query syntax: a **replica set** is 1 primary (all writes) + secondaries replicating the **oplog**; an **election** promotes a secondary if the primary dies — you need an odd number of voting members (or an arbiter) to maintain quorum, and a majority to elect (a 2-node set that loses one node goes read-only — no majority). **Read/write concerns** tune consistency vs latency: `w: majority` survives failover without losing acknowledged writes; reading from secondaries risks stale reads. **Sharding** distributes data by a shard key when one replica set isn't enough — a bad shard key (low cardinality, monotonic) creates hotspots. Ops staples: connections storms from poorly pooled apps, **working set vs RAM** (when indexes+hot data exceed memory, performance falls off a cliff), missing indexes found via the slow query log/profiler, and backups via `mongodump` (small), snapshots, or Atlas/Ops Manager continuous backup. From my Domino platform work, MongoDB is the control-plane state store — I've debugged the class of issue where Mongo's view of state desyncs from Kubernetes' actual state, so I've seen its failure modes from the operator's seat.

### Q53. What makes Snowflake architecturally different from a traditional database, and what should platform/ops care about?

Three-layer architecture: (1) **storage** — all data in compressed columnar *micro-partitions* on cloud object storage (S3/GCS/Azure), paid separately; (2) **compute** — **virtual warehouses**: independent clusters that all see the same data; different teams/workloads get *their own* warehouse so the ETL job can't slow the BI dashboards (workload isolation without data copies); auto-suspend/auto-resume per-second billing; (3) **cloud services** — query parsing, optimization, metadata, transactions, access control. Killer features to name: **zero-copy cloning** (instant full clones of databases for testing — metadata-only), **Time Travel** (query/undrop data as of N days ago), **data sharing** without copying. Platform/ops concerns: **cost governance is the job** — warehouses left running or oversized are the classic bill explosion (auto-suspend everything, resource monitors with kill thresholds, right-size warehouse per workload), plus RBAC design, and network policy/PrivateLink integration into the enterprise network.

---

# SECTION 9: TERRAFORM — MODULES, STATE, ENTERPRISE PATTERNS

## Mental Model First

Terraform is **declarative**: you describe the desired end state in HCL; Terraform compares it to reality and computes the minimal set of changes. The comparison happens through the **state file** — Terraform's record of every resource it manages and the mapping between your code and real-world resource IDs. The workflow is always: `init` (download providers/modules, connect backend) → `plan` (show the diff — the safety review) → `apply` (execute). **Modules** are Terraform's reusable functions: inputs (variables) → resources → outputs. Enterprise Terraform = remote state with locking, module registries with versioning, policy-as-code gates, and plans reviewed like code.

---

### Q54. Why does Terraform need a state file at all? Why not just query AWS every time?

Three reasons: (1) **mapping** — the state binds `aws_instance.web` in your code to `i-0abc123` in AWS; without it Terraform can't know which real resource corresponds to which code block (names/tags aren't reliable or universal); (2) **performance and dependency metadata** — state caches attributes and records the dependency order needed to destroy things correctly; (3) **tracking deletions** — if you delete a resource block from code, only state tells Terraform that resource *used to be managed* and should be destroyed; pure API-scanning couldn't distinguish "not mine" from "mine, now removed from code." Follow-up trap: **state can drift** from reality (console changes) — `terraform plan` (which refreshes) detects drift, and the discipline answer is *nobody changes managed infra outside Terraform*.

### Q55. How do you manage state safely for a team?

- **Remote backend**: S3 bucket (versioned, encrypted, access-logged) — never local state in git (it contains secrets in plaintext and merge conflicts destroy it).
- **State locking**: DynamoDB lock table (or S3's native lockfile support in newer versions) so two applies can't run concurrently and corrupt state.
- **State segmentation**: many small states (per environment × per component: `prod/networking`, `prod/eks`, `dev/networking`), not one monolith — smaller blast radius, faster plans, narrower permissions. Cross-state references via `terraform_remote_state` data sources or better, data sources querying real resources by tags/names.
- **Least-privilege access**: state contains secrets → the state bucket is as sensitive as a secrets store; restrict it accordingly.
- **Surgical commands you should know**: `terraform state mv` (refactoring code without destroy/recreate), `terraform import` (adopt existing resources), `terraform state rm` (unmanage without destroying), `-refresh-only` plans (reconcile drift into state deliberately).

### Q56. Design a reusable, compliant Terraform module. What makes a module "good"?

- **Right abstraction level**: a module should encode an *opinionated pattern* ("our standard VPC", "our compliant S3 bucket"), not wrap a single resource 1:1 (worthless indirection) nor an entire application stack (inflexible monolith).
- **Interface discipline**: minimal required variables with **validation blocks** and sane defaults; typed variables; outputs exposing exactly what consumers need (IDs, ARNs) — the module's variables/outputs are its API contract.
- **Compliance baked in, not optional**: the module *always* enables encryption, versioning, required tags, logging — consumers can't opt out of the guardrails. That's how "reusable, compliant, standardized" (the JD phrase) actually happens: teams get compliance by default just by using the blessed module.
- **Versioned releases**: modules live in their own repos / a registry, consumed with **pinned versions** (`source = "...?ref=v2.3.1"`); semantic versioning so consumers upgrade deliberately. Never let live infra float on a module's main branch.
- **Tested**: `terraform validate` + fmt in CI, static policy scans (tfsec/Checkov), example configurations that CI actually plans/applies in a sandbox (Terratest or `terraform test` for the mature version).
- **Documented**: README with usage examples; terraform-docs generating the inputs/outputs table.

### Q57. count vs for_each — and why does everyone say prefer for_each?

`count` creates a **list** — resources identified by index (`aws_instance.web[0]`, `[1]`, `[2]`). Remove the *middle* item and every subsequent resource **shifts index**, so Terraform plans to destroy/recreate all of them — a catastrophic plan from an innocent edit. `for_each` creates a **map** keyed by meaningful stable keys (`aws_subnet.private["ap-south-1a"]`); adding/removing one key touches only that resource. Rule: `count` only for the boolean trick (`count = var.enabled ? 1 : 0`) or truly identical interchangeable copies; `for_each` for everything that is a collection of named things.

### Q58. How does Terraform decide the order to create resources?

It builds a **DAG (directed acyclic graph)** from references: if resource B's arguments reference `aws_vpc.main.id`, B depends on the VPC — **implicit dependencies** from expressions are the primary mechanism, and Terraform parallelizes anything without a dependency path between them. `depends_on` is the escape hatch for dependencies Terraform can't see in expressions (e.g., an IAM policy that must exist before a service can function even though nothing references it) — overusing it is a smell signaling the code isn't expressing real relationships. Cycles are errors; destroys happen in reverse dependency order.

### Q59. A `terraform apply` failed halfway. What's the situation and what do you do?

Terraform is **not transactional** — there's no rollback. Resources created before the failure are created and *recorded in state* (state is written incrementally). The situation is a partially-built environment, and the recovery is usually boring: read the actual error (quota? IAM deny — maybe an SCP! provider bug? name conflict?), fix the root cause, and **run apply again** — Terraform resumes from where it is, because plan/apply is idempotent against current state. Complications worth naming: a resource that was created but a *crash* prevented state from recording it → next apply hits "already exists" → resolve with `terraform import`; and if the process died holding the **lock**, `terraform force-unlock` after confirming no apply is actually running. The deep answer: this is exactly why plans run in CI with review, why states are small, and why applies are done from a pipeline, not laptops.

### Q60. How do you run Terraform in a pipeline (GitLab) with proper controls?

Merge-request-driven workflow: (1) on MR: `fmt -check`, `validate`, security/policy scan (tfsec/Checkov — and per this JD, Checkmarx IaC scanning), then `terraform plan -out=plan.tfplan` posted to the MR — **the plan is the review artifact**; humans approve the diff, not the intent. (2) on merge to main: apply **the saved plan file** (guaranteeing what was approved is what executes), against remote state with locking, using short-lived credentials via **OIDC federation** between GitLab and AWS (no long-lived AWS keys stored in CI variables — say this, it's a strong senior signal). (3) **protected environments / manual approval gate** in front of prod applies. (4) Policy-as-code (OPA/Sentinel-style) to hard-block noncompliant plans (public S3, missing tags, wrong regions). (5) Drift detection: scheduled `plan` jobs alerting on non-empty diffs.

---

# SECTION 10: GITLAB CI/CD — PIPELINES, SECURITY SCANNING, DEPLOYMENT CONTROLS

## Mental Model First

Everything lives in `.gitlab-ci.yml`. A **pipeline** is a set of **stages** (run sequentially) containing **jobs** (run in parallel within a stage) executed by **runners** (agents — shared SaaS runners or your own on EC2/Kubernetes). Jobs pass files forward via **artifacts**; **cache** speeds up repeated dependency downloads; `rules:` decide when jobs run (branch, MR, tag, changed paths); `needs:` breaks strict stage ordering into a DAG for speed. Security is integrated as scanning jobs (SAST, dependency, secrets, container scanning) whose results gate merges. Deployment control = **environments** + **protected branches/environments** + **manual approval jobs**.

---

### Q61. Walk me through a production-grade pipeline for a containerized service.

Stages and what each is *for*:
1. **Validate/lint** — fail fast on cheap checks (lint, fmt, commit hygiene).
2. **Test** — unit tests with coverage reports surfaced in the MR.
3. **Security scans in parallel** — SAST (code vulnerabilities — Checkmarx or GitLab SAST), **dependency/SCA scanning** (known CVEs in libraries), **secret detection** (committed credentials — blockable), IaC scanning if the repo carries Terraform.
4. **Build** — build the Docker image once, tag with the **commit SHA** (immutable, traceable — never build per-environment), push to the registry.
5. **Container scan** — scan the built image (Trivy/GitLab container scanning) for OS/package CVEs; fail on critical.
6. **Deploy to dev/staging** — automatically on merge to main; run integration/smoke tests against it.
7. **Deploy to prod** — `when: manual` job in a **protected environment** (only authorized approvers can trigger; environment-scoped secrets only available here), deploying the *same SHA-tagged image* that passed everything. Rollback = redeploy previous SHA.
Cross-cutting: `rules:` so MRs run validation/tests only; `needs:` to let independent jobs start early; artifacts carrying test/scan reports into the MR widget so reviewers see everything in one place.

### Q62. How do you keep secrets out of pipelines properly?

Layered answer: (1) **masked + protected CI/CD variables** as the baseline — masked (hidden in logs), protected (only exposed on protected branches, so a malicious MR from a fork can't exfiltrate prod secrets — know this attack); (2) better: **no static secrets at all** — GitLab issues an **OIDC ID token** per job, and cloud providers/Vault trust it: the job assumes an AWS role or pulls from Vault with a short-lived credential scoped to that project/branch/environment. Nothing long-lived to leak or rotate. (3) **secret detection scanning** to catch accidental commits, plus pre-commit hooks; (4) never `echo` secrets, beware `set -x` in scripts leaking them into logs; (5) environment-scoped variables so dev jobs physically cannot see prod values.

### Q63. What deployment controls does GitLab give you for regulated/controlled environments?

- **Protected branches**: only maintainers merge to main; force-push disabled; MR + approvals required (approval rules can mandate code owners / security team sign-off).
- **Protected environments**: deployments to `production` can only be triggered by allowed users/groups; combine with **deployment approvals** (N approvers before the job runs) — this is your CAB-in-the-pipeline.
- **Manual jobs** (`when: manual`) as explicit human gates.
- **Audit**: every pipeline, approver, variable change, and deployment is logged — the evidence trail auditors ask for. Pair with the ServiceNow change record: the CR references the pipeline URL; the deployment approval maps to the change approval. In GxP-style environments (my pharma platform background), this pipeline evidence is exactly what validation/compliance reviews consume.
- **Version-pinned includes/templates**: central CI templates (`include:` from a hardened repo) so every team inherits the compliant pipeline skeleton rather than reinventing it — the CI analog of the compliant Terraform module.

### Q64. A pipeline is suddenly slow / flaky. How do you optimize and stabilize?

Diagnose before optimizing — GitLab shows per-job durations; find the actual long pole. Common wins: **cache dependencies** properly (keyed on lockfile hash) vs re-downloading every job; **Docker layer strategy** (order Dockerfile so dependency layers cache; use `--cache-from` or BuildKit registry cache in CI where each job starts cold); **`needs:` DAG** so jobs start as soon as their true dependencies finish rather than waiting for whole stages; parallelize test suites (`parallel:`); right-size runners (CPU-starved shared runners → your own autoscaling runners on EC2/Kubernetes); `interruptible: true` so superseded pipelines cancel. Flakiness: quarantine flaky tests rather than normalizing retries; `retry:` only for known-transient infra failures (runner disconnects) — blanket retries hide real bugs; pin tool/image versions (mystery breakage from `latest` drifting is self-inflicted).

---

# SECTION 11: DOCKER — CREATION, MANAGEMENT, TROUBLESHOOTING

## Mental Model First

A container is **not a VM** — it's a normal Linux process wearing isolation: **namespaces** give it a private view (its own PID tree, network stack, mounts, hostname), **cgroups** cap its resources (CPU, memory), and a **layered image filesystem** (union/overlay FS) gives it its own root. An **image** is an immutable stack of read-only layers (one per Dockerfile instruction) plus metadata; a **container** is that stack plus a thin writable layer, running. Because a container is just a process, all your Linux debugging instincts apply directly.

---

### Q65. Explain images vs containers vs layers, and why layer ordering in a Dockerfile matters.

Image = read-only layer stack + config (entrypoint, env, ports). Container = image + writable top layer + a running process. Each Dockerfile instruction creates a layer, and builds cache per-layer: a layer rebuilds only if its instruction or its *inputs* changed — and **everything after a changed layer rebuilds too**. Hence the ordering rule: stable things first, volatile things last — `COPY package.json` + `npm install` *before* `COPY . .` — so code changes don't invalidate the dependency-install layer. Get this wrong and every build reinstalls dependencies; get it right and code-only builds take seconds. Layers are also shared on disk and in registries — ten images on the same base share the base layers once.

### Q66. What makes a production-quality Dockerfile?

- **Multi-stage builds**: build stage with compilers/dev deps; final stage copies only the artifact onto a minimal base — smaller attack surface, smaller pulls, faster pod starts.
- **Minimal, pinned base images**: `python:3.12-slim` not `latest` (reproducibility), distroless/alpine where practical; fewer packages = fewer CVEs for the container scanner to flag.
- **Non-root USER**: create and switch to an unprivileged user — required by hardened Kubernetes policies anyway.
- **Deterministic installs**: lockfiles, `--no-cache-dir`, clean apt lists in the same RUN to keep the layer small.
- **.dockerignore**: keep .git, secrets, node_modules out of the build context (context size = build upload time, plus secret-leak risk).
- **Proper ENTRYPOINT as PID 1**: exec form (`["python", "app.py"]`) so signals reach the app — otherwise SIGTERM hits a shell that ignores it, graceful shutdown never happens, and Kubernetes SIGKILLs your app after the grace period (a real cause of dropped requests during deploys). Use tini/init if the app spawns children (zombie reaping).
- **HEALTHCHECK** (or rely on orchestrator probes), and never bake secrets into layers — even deleted files persist in earlier layers (`docker history` reveals them). Secrets come at runtime.

### Q67. A container keeps restarting / exits immediately. Your debugging sequence?

1. `docker ps -a` / `kubectl get pods`: exit code first. **137** = SIGKILL — usually **OOMKilled** (confirm via `docker inspect` `.State.OOMKilled` / pod status) → memory limit vs actual usage; **139** = segfault; **1** = app error; **126/127** = permission / command-not-found (bad entrypoint or missing binary — common with alpine images missing bash).
2. `docker logs` (`--previous` on k8s for the crashed instance) — the app usually tells you (missing env var, can't reach DB, config parse error).
3. `docker inspect`: verify the actual entrypoint/cmd/env/mounts the container ran with — the config you *think* vs reality.
4. Reproduce interactively: `docker run -it --entrypoint sh image` — poke around: is the file there? does the binary run? DNS resolve?
5. CrashLoopBackOff with instant exits and no logs → entrypoint itself failing before the app initializes, or the process exiting 0 because it daemonized (containers need foreground processes).
6. If it's slow-death restarts: liveness probe too aggressive (killing a healthy-but-slow-starting app — fix with startupProbe/initialDelay) — the classic Kubernetes self-inflicted restart loop.

### Q68. How does Docker networking work, and what are common network issues?

Default **bridge** network: each container gets a veth pair into a Linux bridge (docker0), NAT (iptables masquerade) for outbound, `-p host:container` publishes ports via DNAT rules. **User-defined bridge networks** add embedded DNS — containers resolve each other by name (the default bridge doesn't). **host** network: no isolation, container shares the host stack (performance/simplicity, port conflicts). In Kubernetes, Docker's networking is replaced by the CNI (in EKS, the VPC CNI: pods get real VPC IPs — so pod connectivity issues become *VPC* issues: security groups, subnet IP exhaustion, ENI limits per node — that last one caps pods-per-node and causes scheduling failures). Common issues: container can't reach out (host iptables/firewalld mangled Docker's rules — restart docker to rebuild them), name resolution fails between containers (default bridge, no DNS — use a user-defined network), port already allocated (host port conflict), and MTU mismatches in overlay/VPN environments causing large-packet hangs while pings work.

# Complete Interview Q&A Guide — Senior Platform / Cloud Engineer
### Built from the JD: AWS Networking, Migrations, Terraform, GitLab CI/CD, Docker, Linux, Security

**How to use this guide:** Every answer has two layers — a *mental model* (so you truly understand it) and *interview framing* (how to say it out loud). Read the mental model first. If you understand the "why," the "how" becomes obvious.

---

# SECTION 1 — VPC & Core AWS Networking

### Q1. What is a VPC and why does it exist?
**Mental model:** A VPC (Virtual Private Cloud) is your own private, logically isolated slice of the AWS network. Think of AWS as a giant apartment building — a VPC is your apartment. You choose the address range (CIDR block, e.g., `10.0.0.0/16`), you decide the internal room layout (subnets), and you control the doors (gateways, security groups).

**Why it exists:** Before VPCs, cloud instances sat on shared flat networks — no isolation, no custom IP addressing, no way to replicate an enterprise network topology. VPCs let enterprises bring their network design (private subnets, DMZs, firewalls, on-prem connectivity) into the cloud.

**Interview answer:** "A VPC is a logically isolated virtual network in AWS where I define the IP address space, subnets, routing, and security boundaries. It's the foundation of everything — every EC2 instance, ECS task, RDS database lives inside a VPC, and the VPC design decides what can talk to what."

### Q2. What is a subnet, and why do we split a VPC into public and private subnets?
**Mental model:** A subnet is a sub-range of the VPC's CIDR that lives in exactly **one Availability Zone**. The split into public/private is a security posture:
- **Public subnet** = its route table has a route to an **Internet Gateway (IGW)**. Resources here can have public IPs and be reached from the internet (load balancers, bastion hosts).
- **Private subnet** = no route to the IGW. Resources here (app servers, databases) cannot be reached from the internet directly — they reach *out* via a NAT Gateway.

**The key insight interviewers want:** "public" vs "private" is not a property of the subnet itself — it's purely determined by **the route table attached to it**. A subnet is public *because* its route table sends `0.0.0.0/0` to an IGW.

### Q3. How does a route table actually work? What is route evaluation order?
**Mental model:** A route table is a list of "destination → target" rules. When a packet leaves a resource, AWS looks at the destination IP and picks the **most specific matching route** (longest prefix match) — not top-to-bottom order.

Example route table for a private subnet:
| Destination | Target | Meaning |
|---|---|---|
| 10.0.0.0/16 | local | Anything inside the VPC stays inside (this route is automatic and cannot be deleted) |
| 10.20.0.0/16 | tgw-xxxx | Traffic to another VPC goes via Transit Gateway |
| 0.0.0.0/0 | nat-xxxx | Everything else goes to the NAT Gateway |

If a packet is headed to `10.20.5.4`, both `0.0.0.0/0` and `10.20.0.0/16` match — but `/16` is more specific, so the TGW route wins. **Longest prefix match** is a phrase worth saying in the interview.

### Q4. What is a NAT Gateway and why do private subnets need it?
**Mental model:** Servers in private subnets have only private IPs (`10.x.x.x`), which are not routable on the internet. But they still need outbound access — to pull Docker images, download OS patches, call external APIs. NAT (Network Address Translation) solves this: the NAT Gateway sits in a **public subnet**, has an Elastic IP, and rewrites the source IP of outbound packets from the private IP to its own public IP. It tracks connections so return traffic gets translated back.

**Critical property:** NAT is one-directional. It allows **outbound-initiated** connections only. The internet cannot initiate a connection *into* your private subnet through a NAT Gateway. That's the whole security value.

**Common follow-ups:**
- *"Why one NAT Gateway per AZ?"* — For high availability (a NAT GW lives in one AZ; if that AZ dies, private subnets routing to it lose internet) and to avoid cross-AZ data charges.
- *"NAT Gateway vs NAT instance?"* — NAT GW is managed, scales to 100 Gbps automatically, no patching. NAT instance is a DIY EC2 box — cheaper at tiny scale, but you own its availability. Enterprises use NAT GW.
- *"Why is my NAT bill huge?"* — NAT GW charges per GB processed. Classic culprit: workloads pulling from S3/ECR through NAT. Fix: **VPC Gateway Endpoints (S3/DynamoDB, free)** and **Interface Endpoints (ECR, etc.)** so traffic stays on the AWS network and skips NAT.

### Q5. Internet Gateway vs NAT Gateway — what's the actual difference?
- **IGW**: a horizontally-scaled, VPC-attached door to the internet. Does 1:1 translation between a resource's public IP and private IP. Allows **inbound and outbound**. No bandwidth limit, free.
- **NAT GW**: allows **outbound only**, many private IPs share one public IP (many:1), lives in a specific subnet/AZ, charged per hour + per GB.

**One-liner:** "IGW is the front door with a doorbell — traffic flows both ways. NAT Gateway is a one-way exit door — insiders can leave and come back, outsiders can't enter."

### Q6. What are Security Groups vs NACLs, and why are both stateful vs stateless important?
**Security Group (SG):** a virtual firewall on the **ENI/instance level**. **Stateful** — if you allow inbound traffic, the response is automatically allowed out (and vice versa). Only *allow* rules. All rules evaluated together.

**Network ACL (NACL):** a firewall on the **subnet level**. **Stateless** — return traffic must be explicitly allowed (this is why NACLs need ephemeral port ranges `1024-65535` open for responses). Supports *allow and deny* rules, evaluated in numbered order, first match wins.

**Why stateful matters practically:** With SGs you never think about return traffic. With NACLs, a classic outage is "I allowed inbound 443 but forgot the outbound ephemeral ports — the handshake dies." NACLs are mostly used for coarse guardrails (e.g., explicitly deny a malicious IP range); SGs do the real work.

### Q7. How would you troubleshoot "Instance A can't reach Instance B" inside AWS?
Give this as a systematic checklist — interviewers love a methodical answer:
1. **Route tables** — does A's subnet have a route to B's CIDR (local, peering, TGW)?
2. **Security groups** — does B's SG allow inbound from A's SG/IP on that port? Does A's SG allow outbound (usually default-allow)?
3. **NACLs** — both directions, both subnets, including ephemeral ports.
4. **Is the service listening?** — `ss -tlnp` on B; a network path to a dead process still fails.
5. **DNS** — is A resolving B's name to the right IP? (`dig`, check private hosted zone / resolver rules)
6. **VPC Reachability Analyzer / VPC Flow Logs** — Flow Logs show ACCEPT/REJECT per ENI; REJECT tells you a SG/NACL dropped it. Reachability Analyzer statically verifies the path and names the blocking component.

Mentioning **Flow Logs + Reachability Analyzer** signals real operational experience.

---

# SECTION 2 — Transit Gateway (TGW)

### Q8. Why does Transit Gateway exist? What problem does it solve over VPC peering?
**Mental model:** VPC peering is a point-to-point cable between two VPCs. It works fine for 2–3 VPCs. But peering is **non-transitive** — if A↔B and B↔C are peered, A still cannot reach C through B. So connecting *n* VPCs fully requires *n(n−1)/2* peering connections: 10 VPCs = 45 connections, 50 VPCs = 1,225. Unmanageable.

**Transit Gateway = a cloud router / central hub.** Every VPC (and every VPN/Direct Connect to on-prem) attaches to the TGW once. The TGW routes between all of them. 50 VPCs = 50 attachments instead of 1,225 peerings. It turns a mesh into a **hub-and-spoke**.

**Interview answer:** "TGW is a regional, managed router that solves the mesh-scaling problem of VPC peering. Peering is non-transitive, so it scales quadratically; TGW gives hub-and-spoke connectivity that scales linearly, plus centralized route control, which is what enterprises need for segmentation."

### Q9. How does traffic actually flow through a TGW? (The two-hop routing model)
This is the concept most candidates get wrong. **Every packet crossing a TGW is routed twice:**

1. **Hop 1 — VPC route table:** The source VPC's subnet route table must have a route sending the destination CIDR to the TGW attachment (`10.20.0.0/16 → tgw-xxxx`). This gets the packet *to* the TGW.
2. **Hop 2 — TGW route table:** The TGW has its own route tables. It looks up the destination and forwards to the correct *attachment* (`10.20.0.0/16 → attachment-of-VPC-B`).

**And the return path needs both hops configured in reverse.** The #1 cause of "TGW connectivity works one way only" is a missing return route in either the destination VPC's route table or the TGW route table.

**Analogy:** VPC route table = the sign in your office saying "for other buildings, go to the shuttle stop." TGW route table = the shuttle dispatcher deciding which building your package goes to. Both must know the address.

### Q10. How do TGW route tables enable network segmentation?
A TGW can have **multiple route tables**, and each attachment is *associated* with exactly one (which table its incoming traffic uses) and can *propagate* its routes into any of them (advertising itself as a destination).

**Classic enterprise pattern:**
- **Prod route table:** prod VPC attachments associate here; only prod + shared-services routes are propagated in. Prod VPCs can reach each other and shared services — but there is simply *no route* to dev.
- **Dev route table:** same idea for dev.
- **Shared services / egress route table:** propagated into both, so everyone reaches DNS, monitoring, or the central egress VPC.

**Key phrase:** "Segmentation by routing — if the route doesn't exist in your TGW route table, the traffic can't flow, regardless of security groups." This is how enterprises enforce prod/dev isolation at the network layer.

### Q11. What's the difference between association and propagation on a TGW?
- **Association** = which TGW route table an attachment's *outgoing lookups* use ("which map do I read?"). One per attachment.
- **Propagation** = injecting an attachment's CIDRs *into* a TGW route table ("who learns my address?"). Can propagate into many tables.

Getting this crisp instantly signals hands-on depth.

### Q12. How does on-prem connectivity work through TGW (hybrid)?
On-prem connects via **Site-to-Site VPN** or **Direct Connect (via a Direct Connect Gateway)**, both of which become TGW *attachments*. On-prem routes are learned via **BGP** and propagate into TGW route tables; VPC CIDRs are advertised back to on-prem. So one TGW gives every attached VPC a path to the datacenter without per-VPC VPNs.

**Follow-up you may get:** *"How do you avoid overlapping CIDRs?"* — You can't route between overlapping CIDRs; enterprises maintain a central **IPAM** (IP address management) plan so every VPC and on-prem range is unique. If overlap already exists, options are painful: NAT tricks, PrivateLink, or re-IP the VPC. Say "prevention via IPAM is the real answer."

### Q13. What are TGW's limits/costs worth knowing?
- Regional service; cross-region needs **TGW peering** (which *is* transitive-safe but static-routed).
- Charged per attachment-hour + per GB processed — so hairpinning heavy traffic through TGW costs money; huge flows between two specific VPCs may justify direct peering (peering data is cheaper and lower latency).
- MTU 8500 within AWS attachments, 1500 over VPN.

---

# SECTION 3 — Route 53 & DNS (including Resolvers and Hybrid DNS)

### Q14. Explain how DNS resolution actually works, end to end.
**Mental model:** DNS is a distributed phone book resolved by delegation.
1. Your laptop asks its configured **recursive resolver** (e.g., corporate DNS, 8.8.8.8): "What's the IP of app.example.com?"
2. If not cached, the resolver asks a **root server** → which points to the **.com TLD servers** → which point to the **authoritative name servers for example.com** (e.g., Route 53) → which return the actual record.
3. The resolver caches the answer for the record's **TTL** and returns it to you.

**Key vocabulary:** *recursive resolver* (does the legwork, caches), *authoritative server* (owns the zone, gives the final answer), *TTL* (how long answers may be cached).

### Q15. What record types must you know cold?
- **A / AAAA** — name → IPv4 / IPv6 address.
- **CNAME** — name → another name. Cannot exist at a zone apex (`example.com` itself).
- **Alias (Route 53-specific)** — like a CNAME but works at the apex and points at AWS resources (ALB, CloudFront, S3 website); resolved server-side, free queries.
- **NS** — which name servers are authoritative for a zone (how delegation works).
- **SOA** — zone metadata.
- **TXT** — arbitrary text; used for domain validation (ACM!), SPF/DKIM.
- **MX** — mail routing.

*"CNAME vs Alias"* is a near-guaranteed question: Alias is Route 53's answer to the apex-CNAME restriction, and it tracks the underlying AWS resource's IPs automatically.

### Q16. What are Route 53 routing policies and when do you use each?
- **Simple** — one answer.
- **Weighted** — split traffic by percentage → canary releases, gradual migrations (e.g., 95% old stack, 5% new).
- **Latency-based** — answer with the region closest (by measured latency) to the user.
- **Failover** — primary/secondary with health checks; DNS-level DR.
- **Geolocation / Geoproximity** — route by user location (compliance, localization).
- **Multivalue** — up to 8 healthy records, poor-man's load balancing.

**Migration relevance (this JD!):** weighted routing is *the* tool for cutting traffic over from an old environment to a new one gradually, with failover as the safety net.

### Q17. What are private hosted zones?
A hosted zone that answers DNS queries **only inside VPCs you associate with it**. `db.internal.corp` can resolve to `10.0.5.20` for your workloads while being invisible to the internet. Essential for internal service naming. Same name can exist in a public and private zone (**split-horizon DNS**) — internal clients get internal IPs, external clients get public ones.

### Q18. What are Route 53 Resolver endpoints, and why do hybrid networks need them? (High-probability question for this JD)
**The problem:** Every VPC has a built-in resolver at VPC-CIDR+2 (e.g., `10.0.0.2`), but it's only reachable from *inside* the VPC. So:
- On-prem servers can't resolve your private hosted zone names.
- VPC workloads can't resolve on-prem Active Directory names like `db01.corp.local`.

**The solution — two endpoint types:**
- **Inbound Resolver Endpoint** — ENIs with IPs in your VPC that *accept* DNS queries from on-prem. On-prem DNS servers are configured to conditionally forward `*.aws.internal` queries to these IPs. Now on-prem can resolve AWS private zones.
- **Outbound Resolver Endpoint + Forwarding Rules** — lets the VPC resolver *forward* queries for specific domains (e.g., `corp.local`) to your on-prem DNS servers' IPs. Now AWS workloads resolve on-prem names.

**Interview one-liner:** "Inbound = on-prem asking AWS. Outbound = AWS asking on-prem. Rules define which domains go where, and rules can be shared across accounts with RAM so you configure hybrid DNS once for the whole org."

### Q19. Why is TTL discipline critical during migrations/cutovers?
If a record has TTL 86400 (24h), resolvers worldwide may keep serving the **old IP for up to a day after you change it**. Migration playbook:
1. **Days before cutover:** lower TTL to 60s (you must wait out the *old* TTL before the low TTL is universally in effect).
2. **Cutover:** change the record; traffic moves within ~a minute.
3. **Rollback stays instant** while TTL is low.
4. **After stability:** raise TTL back (low TTLs = more query load/cost and more DNS dependency).

Saying "you have to lower TTL *at least one old-TTL-period before* the change" shows real cutover experience.

### Q20. How do you troubleshoot DNS issues?
- `dig app.example.com` — check answer, TTL, which server answered.
- `dig @10.0.0.2 app.example.com` — query the VPC resolver directly to isolate resolver vs client config.
- `dig +trace` — walk the delegation chain from root; finds broken NS delegations.
- Check **/etc/resolv.conf** on Linux (who is the client actually asking?).
- For hybrid: verify resolver rules, endpoint SG allows TCP+UDP 53, and on-prem conditional forwarders point at the *inbound endpoint IPs*.
- Remember caching at every layer: OS cache, app-level caches (JVM caches DNS forever by default!), resolver cache.

---

# SECTION 4 — SSL/TLS, ACM & Certificate Lifecycle

### Q21. Walk me through what happens in a TLS handshake. Why do we need it?
**Why:** Plain HTTP is readable and modifiable by anyone on the path. TLS gives you **encryption** (privacy), **authentication** (you're really talking to your bank, not an imposter), and **integrity** (nothing was tampered with).

**How (TLS 1.2 simplified, conceptually):**
1. **ClientHello:** client says "here are the TLS versions and cipher suites I support" + a random value.
2. **ServerHello + Certificate:** server picks the cipher suite and sends its **certificate** (its public key, signed by a Certificate Authority).
3. **Certificate validation:** client checks — is it signed by a CA I trust (chain of trust up to a root in my trust store)? Is it expired? Does the hostname match the CN/SAN? Is it revoked?
4. **Key exchange:** client and server derive a shared **session key** (with ephemeral Diffie-Hellman in modern suites — giving *forward secrecy*: even if the server's private key leaks later, past traffic stays safe).
5. From here, all traffic is encrypted **symmetrically** with the session key (symmetric crypto is fast; asymmetric was only used to establish trust and exchange keys).

**TLS 1.3** cuts the handshake to one round trip and removes weak ciphers — worth mentioning.

**The key sentence:** "Asymmetric cryptography authenticates and bootstraps trust; symmetric cryptography does the bulk encryption because it's orders of magnitude faster."

### Q22. What is the certificate chain of trust?
Your server cert is signed by an **intermediate CA**, which is signed by a **root CA**. Browsers/OSes ship with a trust store of root CAs. Validation = follow signatures upward until you hit a trusted root. **Classic production incident:** server serves only the leaf cert without the intermediate → some clients fail with "unable to verify the first certificate." Fix: serve the full chain. Debug with `openssl s_client -connect host:443 -showcerts`.

### Q23. What is ACM and why do enterprises use it?
AWS Certificate Manager issues **free public TLS certificates** and — the killer feature — **auto-renews them** when validated via DNS. Certificates deployed on managed services (ALB, CloudFront, API Gateway) renew and rotate with zero human action.

**Constraints worth stating:**
- ACM public certs **cannot be exported** — the private key never leaves AWS. You can only use them on integrated services (ALB, NLB, CloudFront, API GW). For certs on EC2/on-prem, you need ACM Private CA or an external cert.
- **CloudFront requires the cert to be in us-east-1** (classic gotcha — CloudFront is global and its control plane lives there).
- **DNS validation > email validation**: DNS validation plants a CNAME in your zone; as long as it stays, every renewal auto-validates. Email validation requires a human every renewal — a renewal-failure time bomb.

### Q24. Describe the certificate lifecycle and how you manage it at enterprise scale.
Lifecycle: **request → validate domain ownership → issue → deploy → monitor expiry → renew → rotate → (revoke if compromised)**.

Enterprise practice:
- Inventory *all* certs (ACM, imported, on-prem, appliance certs) — outages come from the cert nobody tracked.
- Automate: ACM for AWS-native, cert-manager/Let's Encrypt for Kubernetes, Vault or Private CA for internal mTLS.
- **Monitor expiry** with alarms (AWS Config rule / Security Hub / custom check at 45/30/14 days). Expired certs are one of the most common self-inflicted enterprise outages.
- Know your **TLS termination points**: at CloudFront? ALB? Both (re-encrypt to origin)? Each termination point has its own cert to manage.

### Q25. Where can TLS terminate in an AWS architecture, and what are the trade-offs?
- **Terminate at ALB/CloudFront, HTTP to backend:** simplest; acceptable only if the VPC-internal path is trusted.
- **Terminate and re-encrypt (end-to-end TLS):** CloudFront→origin over TLS, ALB→target over TLS. Standard in regulated environments (pharma, finance — relevant to your background).
- **TCP passthrough (NLB):** the backend terminates TLS itself; needed for mTLS to the app or non-HTTP protocols.

### Q26. How does Zscaler fit into enterprise TLS, and why does it break things? (JD mentions Zscaler)
Zscaler is a cloud security proxy. **ZIA** (Internet Access) proxies user traffic to the internet; **ZPA** (Private Access) gives zero-trust access to internal apps without a classic VPN.

**The TLS angle:** ZIA does **SSL inspection** — it man-in-the-middles TLS by re-signing sites with a Zscaler CA cert that's pushed into the corporate trust store. Consequences engineers hit constantly:
- CLI tools/containers that don't use the OS trust store (Python `certifi`, Java keystores, Node) fail with certificate errors → fix by adding the Zscaler root CA to each tool's trust store, **never** by disabling verification.
- Certificate pinning breaks → those destinations need SSL-inspection bypass rules.
- During network changes, traffic paths through Zscaler must be considered: is the flow going direct, via ZIA, or via ZPA? A "DNS/network change" can silently change which path applies.

Being able to say "the fix is distributing the Zscaler root CA into every trust store — OS, JVM, Python, Docker images — not disabling TLS verification" is a strong senior signal.

---

# SECTION 5 — CloudFront, API Gateway & WAF

### Q27. What is CloudFront and why put it in front of an application?
A global **CDN**: 600+ edge locations cache content close to users. Benefits: latency (content served from the nearest edge), origin offload (cache hits never touch your servers), cost (CloudFront egress is cheaper than direct), security (TLS at edge, AWS Shield DDoS absorption, WAF attachment point), and a single global entry point.

**Key mechanics:** *origins* (S3, ALB, API GW), *behaviors* (path-pattern → policy, e.g., `/api/*` no-cache to ALB, `/static/*` long-cache to S3), *cache keys* (URL + chosen headers/cookies/query strings — the more in the key, the lower the hit ratio), *invalidations* (purge cached objects; better practice is versioned filenames like `app.v2.js`).

**S3 security pattern:** Origin Access Control (OAC) — bucket is private and only CloudFront may read it, so nobody bypasses the CDN.

### Q28. What is API Gateway and when do you choose it over an ALB?
API Gateway is a managed front door for APIs: request routing, **authentication/authorization** (IAM, Cognito, Lambda authorizers), **throttling and usage plans/API keys**, request validation and transformation, caching, and native Lambda integration.

**API GW vs ALB:** ALB is a load balancer — cheap at high volume, simple L7 routing. API GW is API *management* — choose it when you need per-client throttling, keys, authorizers, request validation, or a serverless backend. At very high request volume ALB is dramatically cheaper; that's a common real-world reason to switch.

### Q29. What is a VPC Link and why does it exist?
API Gateway is a public managed service outside your VPC. To let it call **private** resources (an internal ALB/NLB in private subnets) without exposing them, a **VPC Link** creates the private connection (via PrivateLink/ENIs) from API GW into your VPC. This is *the* pattern for "public API, private backend."

### Q30. Explain WAF and the count-mode-first workflow. (JD explicitly lists WAF)
AWS WAF filters HTTP(S) requests at CloudFront/ALB/API GW using rules: AWS **managed rule groups** (SQLi, XSS, known bad inputs, IP reputation), rate-based rules (block IPs exceeding N requests/5 min), and custom rules (match on headers, paths, geo, body).

**Count-mode-first (the operational discipline):** never enable a new rule directly in BLOCK. First run it in **COUNT** mode — the rule tags matching requests in logs/metrics but blocks nothing. Analyze what *would* have been blocked for days/weeks, find false positives (legitimate traffic matching the rule — e.g., a rich-text field triggering XSS rules), add exclusions/scope-downs, and only then flip to BLOCK.

**Why interviewers care:** turning on managed rules in block mode on day one is how you cause a self-inflicted outage on legitimate users. Count-first is the change-management-safe path — exactly the mindset a "network and security change execution" role wants.

---

# SECTION 6 — ECS, Lambda, Step Functions, Glue, Athena

### Q31. What is ECS and how do its pieces fit together?
ECS (Elastic Container Service) is AWS's native container orchestrator (the simpler sibling of EKS/Kubernetes).
- **Task definition** — the blueprint (like a Kubernetes pod spec): container image(s), CPU/memory, env vars, IAM roles, logging config.
- **Task** — a running instance of that blueprint.
- **Service** — keeps N tasks running, replaces failures, integrates with ALB target groups, handles rolling deployments (like a Kubernetes Deployment).
- **Cluster** — logical grouping of capacity.

### Q32. Fargate vs EC2 launch type — how do you choose?
- **Fargate:** serverless containers — you specify CPU/memory per task, AWS runs it, no hosts to patch or scale. Best default for most services; you trade some cost and lose GPU/daemon-level control.
- **EC2 launch type:** you manage the instances. Choose when you need GPUs, special instance types, very dense bin-packing economics at scale, or host-level agents.

**Coming from Kubernetes (your strength):** map it out loud — "task definition ≈ pod spec, service ≈ deployment + service, Fargate ≈ managed nodes taken to the extreme." Interviewers value candidates who transfer mental models.

### Q33. Two IAM roles in ECS — what's the difference?
- **Task execution role** — used by the ECS *agent* to set the task up: pull the image from ECR, fetch secrets, write logs to CloudWatch.
- **Task role** — used by *your application code* to call AWS APIs (read S3, query DynamoDB).
"Task can't pull image" → execution role. "App gets AccessDenied on S3" → task role. Knowing which is which is a classic hands-on discriminator.

### Q34. What is Lambda, and what are its key operational realities?
Lambda runs your function code on demand — no servers, scales per-request, billed per ms. Key operational facts a senior should volunteer:
- **Limits:** 15-min max execution, memory 128MB–10GB (CPU scales with memory), /tmp ephemeral storage, package size limits (use container images up to 10GB).
- **Cold starts:** first invocation on a new environment initializes the runtime + your init code → latency spike. Mitigations: provisioned concurrency, smaller packages, lighter runtimes, keeping VPC config lean.
- **Lambda in a VPC:** attaches ENIs into your subnets so it can reach private resources (RDS). Modern Lambda uses Hyperplane ENIs so the old brutal VPC cold-start penalty is largely gone — but the subnet needs routes to whatever the function calls, and NAT/endpoints for AWS APIs.
- **Concurrency:** account-level pool; one runaway function can starve others → set reserved concurrency for critical functions.

### Q35. What are Step Functions and when do you use them over "Lambda calling Lambda"?
Step Functions is a managed **workflow orchestrator**: you define a state machine (steps, branching, parallelism, retries, error catching, human-wait) in JSON (ASL), and AWS runs it with full execution history.

**Why not chain Lambdas directly?** Chaining hides state, has no built-in retry/compensation, hits the 15-min ceiling, and is undebuggable at 3am. Step Functions gives **declarative retries with backoff, catch-and-recover paths, visual execution history, and workflows that can run up to a year** (Standard) — e.g., data pipelines, order processing, approval flows. **Express workflows** handle high-volume short-lived event processing.

**One-liner:** "Business logic goes in Lambda; *coordination* logic goes in Step Functions."

### Q36. What is AWS Glue?
Managed **ETL + data catalog** service:
- **Glue Data Catalog** — a central metastore of table definitions (schemas, partitions) over data sitting in S3 and elsewhere. Athena, EMR, Redshift Spectrum all read it.
- **Glue Crawlers** — scan S3, infer schemas, populate the catalog automatically.
- **Glue ETL jobs** — serverless Spark (or Python shell) jobs to transform data (e.g., convert CSV to partitioned Parquet).

### Q37. What is Athena and how does it work with S3/Glue? What makes queries cheap vs expensive?
Athena = **serverless SQL directly on S3 data**, using the Glue catalog for schemas. No cluster, pay **per TB scanned** ($5/TB).

Since cost = bytes scanned, optimization is about scanning less:
1. **Columnar formats (Parquet/ORC)** — read only needed columns; 10–100x less scanned than CSV/JSON.
2. **Partitioning** (e.g., `s3://logs/year=2026/month=07/day=08/`) — `WHERE` clauses on partition keys prune whole directories.
3. **Compression** and avoiding millions of tiny files.

**The pipeline story that ties Glue+Athena together:** raw JSON lands in S3 → Glue job converts to partitioned Parquet → crawler/catalog exposes tables → Athena queries them for analytics, and it's a fraction of the cost of a warehouse for ad-hoc use.

---

# SECTION 7 — Load Balancing & Networking Fundamentals

### Q38. ALB vs NLB — differences and when to use which?
- **ALB (Layer 7 / HTTP):** understands HTTP — routes by path/host/header, does TLS termination, WAF attachment, auth integration, WebSockets. Use for web apps/APIs/microservices.
- **NLB (Layer 4 / TCP-UDP):** forwards connections at massive scale with ultra-low latency, static IPs / Elastic IPs per AZ, preserves source IP, supports TLS passthrough. Use for non-HTTP protocols, extreme performance, PrivateLink endpoints, or when clients must see a fixed IP.

### Q39. How do health checks and connection draining work?
The LB probes each target (e.g., GET `/health` expecting 200) at intervals; after N failures the target is marked unhealthy and receives no *new* traffic. **Deregistration delay (draining)** lets in-flight requests finish before a target is fully removed — critical for zero-downtime deploys. A good `/health` endpoint checks real dependencies shallowly (can I reach my DB?) without being so deep that a transient dependency blip marks the whole fleet unhealthy (cascading failure).

### Q40. Walk me through "what happens when you type a URL and press Enter" — the full path.
The classic. Structure it in layers:
1. **DNS:** browser/OS cache → recursive resolver → (root→TLD→authoritative) → IP.
2. **TCP:** three-way handshake (SYN, SYN-ACK, ACK) to the IP on port 443.
3. **TLS:** handshake — certificate validation, key exchange, session keys (Q21).
4. **HTTP:** request sent; hits CloudFront edge → maybe cache hit; else → WAF evaluation → ALB → target selection via health checks → app → DB → response flows back, possibly cached at edge.
5. **Render.**
Being able to name where **each JD technology sits in this path** (Route 53, ACM cert at CloudFront/ALB, WAF, TGW if the origin is in another VPC) is a superpower answer.

### Q41. What Linux tools do you use for network troubleshooting, and for what?
- `ping` — basic reachability/latency (ICMP; often blocked, so absence of ping ≠ down).
- `traceroute`/`mtr` — path and where latency/loss begins (mtr = continuous traceroute+ping).
- `dig` — DNS (Q20).
- `curl -v` — full HTTP+TLS view: resolution, connect, cert chain, headers, status. `curl -w` for timing breakdown (DNS vs connect vs TTFB).
- `ss -tlnp` — what's listening on which port, by which process (modern netstat).
- `tcpdump -i any port 443` — packet capture; the ground truth. "Do SYNs arrive? Do SYN-ACKs leave?" instantly splits network vs application problems.
- `openssl s_client` — inspect certs/TLS handshake.
- `ip route get <dst>` — which route/interface the kernel will actually use.

**Method to state:** "I work down or up the stack deliberately — DNS → TCP connectivity → TLS → HTTP → application — and at each layer I have a tool that gives ground truth."

---

# SECTION 8 — Migrations (AWS Org/Tenant, Observability, Network/Security Changes, SCPs)

This is the section that differentiates this JD. They want someone who can *execute change safely in a controlled enterprise environment*. Even without having done these exact migrations, you can speak fluently about the methodology — and methodology is what they're testing.

### Q42. What is an AWS Organization, and what does "tenant migration between Organizations" involve?
**AWS Organizations** = the enterprise umbrella over many AWS accounts: consolidated billing, a hierarchy of **OUs (organizational units)**, and centralized guardrails (**SCPs**). A "tenant migration" = moving an AWS account (or workloads) from one Organization to another — common after mergers, divestitures, or org restructures.

**Why it's hard — the account carries invisible dependencies on its old org:**
- **Consolidated billing & savings**: Reserved Instances/Savings Plans shared from the old payer stop applying → cost spike.
- **SCPs change**: guardrails from the old org vanish, the new org's SCPs suddenly apply — things that worked may now be denied (and vice versa).
- **Cross-account IAM trust**: roles trusting `aws:PrincipalOrgID` of the old org break.
- **Shared resources via RAM** (TGW attachments! Resolver rules! subnets): sharing is often org-scoped — leaving the org can *detach your networking*.
- **Centralized services**: org-level CloudTrail, Config, GuardDuty, SSO/Identity Center, delegated admin — all need re-establishment in the new org.

**Migration approach to describe:**
1. **Discovery** — inventory every org-coupled dependency (RAM shares, PrincipalOrgID conditions, SCP reliance, billing constructs, Identity Center assignments).
2. **Plan** — sequence: prepare the destination org (OU placement, SCPs, networking shares ready), pre-create replacements for org-scoped shares.
3. **Execute in a window** — remove account from old org, accept invitation to new org, move into target OU, re-share networking, validate.
4. **Validate** — connectivity, IAM access paths, logging/monitoring flowing, billing attached.
5. **Rollback plan** defined *before* the change.

### Q43. What are SCPs, and how do you change them safely in a controlled enterprise?
**Service Control Policies** are org-level **guardrails**: they define the *maximum* permissions any identity in an account can have. They never grant — they only bound. Effective permission = intersection of SCP ∩ IAM policy (∩ permission boundary ∩ session policy). Even the account root user is bound by SCPs.

Examples: deny leaving the org, deny disabling CloudTrail/GuardDuty, deny regions outside an allowlist, deny deleting KMS keys.

**Safe change process (this is what they're really asking):**
1. **Understand current effect** — SCPs are inherited down the OU tree; map what applies where.
2. **Model the change** — a new *deny* can break running automation org-wide instantly. Search CloudTrail for who currently performs the action you're about to deny.
3. **Test in a sandbox OU** — apply to a test OU with a canary account first.
4. **Prefer additive/staged rollout** — OU by OU, starting with lowest-risk.
5. **Change management** — CR with impact analysis, approval, defined rollback (SCP rollback is fast: detach/reattach — but the *blast radius while wrong* is the entire org).
6. **Monitor** — spike in AccessDenied in CloudTrail after the change = your alarm bell.

**Gotcha worth quoting:** SCP evaluation is not like IAM merge — a policy must be *allowed by every level* (root→OU→account). Attaching a narrow "allow-list" SCP at one level implicitly denies everything not listed at that level.

### Q44. Describe how you'd run an observability migration (e.g., Splunk → Google Observability / SecOps).
**Frame it as a data-pipeline + people migration, not a tool swap:**

1. **Inventory what Splunk does today:** which sources feed it (agents/forwarders on VMs, CloudWatch→Splunk, firewall logs, app logs), which dashboards/alerts/saved searches exist, who consumes them, and which are *actually used* (usage analytics — typically a large fraction is dead weight you shouldn't migrate).
2. **Map the destination:** Google Cloud Observability (Cloud Logging/Monitoring) for operational telemetry; **Google SecOps (Chronicle)** for security telemetry — it's a SIEM: normalizes logs into UDM (unified data model), applies detection rules (YARA-L), used by the SOC.
3. **Build parallel ingestion (dual-ship):** point sources at both systems — e.g., replace/augment Splunk Universal Forwarders with Google Ops Agent / BindPlane, route AWS logs (CloudTrail, VPC Flow Logs, GuardDuty) into SecOps via native feeds. Run both in parallel; never big-bang.
4. **Translate content:** SPL searches → new query language; dashboards rebuilt; **alert parity matrix** — every production alert must exist and be verified firing in the new system before the old one is muted.
5. **Validate:** compare event counts/coverage between systems; test detections end-to-end (send a synthetic event, see the alert).
6. **Cutover per source/team, then decommission** Splunk after a bake period, keeping historical data per retention/compliance requirements (regulated industries may need old data queryable for years — archive strategy matters).

**Senior signal:** "The riskiest moment in an observability migration is the gap — the outage or the security event that happens while alerts exist in neither system properly. Dual-run and alert-parity verification are how you close that gap."

### Q45. How do you execute a network/security change (DNS, certs, TGW, Zscaler, WAF) in a controlled enterprise environment?
Give the change-execution discipline — this is the heart of the JD:

1. **Pre-change:** document current state (route tables, DNS answers via `dig`, cert chains via `openssl`, WAF rule state); write the exact change steps; write the **validation plan** (what will I test to prove success?); write the **rollback plan with a decision point** ("if validation fails by T+30min, we roll back"); get CAB approval; schedule a low-traffic window; notify stakeholders.
2. **Sequencing for DNS-involving changes:** TTL lowered days in advance (Q19).
3. **Execute smallest reversible increments:** WAF in count-mode first; weighted DNS at 5% before 100%; TGW route added for a test CIDR before the full range.
4. **Validate from multiple vantage points:** inside the VPC, from on-prem (through Zscaler!), from the internet — because Zscaler/ZPA means the corporate user's path differs from yours.
5. **Post-change:** monitor error rates/latency dashboards through the bake period; update documentation/diagrams; close the CR with evidence.

**Why multi-vantage validation matters:** a DNS change can look perfect from your laptop and be broken for on-prem users whose queries go through corporate DNS → Zscaler → different resolver path. Saying this out loud is a differentiator.

### Q46. What's your general framework for any migration? (Have this memorized)
**Assess → Plan → Pilot → Parallel-run → Cutover → Validate → Decommission.**
Principles to weave in: inventory before you move; migrate in waves by risk (lowest first); prefer reversible steps; define success criteria and rollback triggers *before* executing; dual-run whenever the system is stateful or alerting; never decommission until the bake period passes.

---

# SECTION 9 — Storage & Data Platforms (S3, EBS, RDS, MongoDB, Snowflake)

### Q47. S3 — what should a platform engineer know cold?
- **Object store**: flat namespace of key→object (prefixes look like folders but aren't), 11 nines durability, regional service with global namespace.
- **Storage classes & lifecycle**: Standard → Standard-IA → Glacier tiers; Intelligent-Tiering automates it; lifecycle rules transition/expire objects (huge cost lever).
- **Security**: block public access (account-wide), bucket policies vs IAM policies, encryption (SSE-S3 default, SSE-KMS for auditable key control), VPC Gateway Endpoint so private workloads reach S3 without NAT (free + faster + private).
- **Versioning + replication** for protection/DR; **consistency**: S3 is now strongly consistent (read-after-write).
- **Presigned URLs** for temporary access without credentials.

### Q48. EBS — key concepts and the failure/performance angles?
Block storage volumes attached to instances over the network. Know:
- **Types:** gp3 (general default; provision IOPS/throughput independently of size — cheaper than gp2), io1/io2 (high, guaranteed IOPS for databases), st1/sc1 (throughput/cold HDD).
- **AZ-locked:** an EBS volume lives in one AZ; to move it, snapshot → restore in another AZ (snapshots go to S3, are incremental, and are cross-region-copyable → DR building block).
- **The attach chain (you know this from PV/PVC work):** EBS volume attaches to the node → appears as a block device (`/dev/nvme1n1`) → filesystem mounted → (in Kubernetes, bind-mounted into the pod). Troubleshooting stuck volumes = walk that chain.
- **Performance issues:** check if you're hitting the volume's IOPS/throughput cap (CloudWatch `VolumeReadOps`, queue length) vs the *instance's* EBS bandwidth cap — a common trap where upgrading the volume does nothing because the instance is the bottleneck.

### Q49. RDS — what does "managed" buy you, and what are Multi-AZ vs read replicas?
RDS runs relational engines (Postgres/MySQL/etc.) with AWS handling provisioning, patching, automated backups (point-in-time restore), and failover.
- **Multi-AZ** = synchronous standby in another AZ for **high availability** — automatic failover (DNS endpoint flips to the standby); standby serves *no* reads. It's an HA feature, not a scaling feature.
- **Read replicas** = asynchronous copies for **read scaling** (and cross-region DR); can lag; can be promoted.
**Interview trap:** "Multi-AZ is for availability, read replicas are for scalability — they solve different problems and mature setups use both."
- Ops angles: parameter groups (config), maintenance windows, storage autoscaling, connection limits (use RDS Proxy for Lambda/spiky connection storms).

### Q50. MongoDB — what should you be able to say?
Document database: JSON-like (BSON) documents, flexible schema, rich queries and secondary indexes.
- **Replica set** = 1 primary (writes) + secondaries (replication, failover via election). Gives HA. **Never run standalone in prod.**
- **Sharding** = horizontal partitioning by shard key for scale beyond one node; shard key choice is the critical design decision (bad key = hot shards).
- **Ops you've genuinely touched (Domino!):** MongoDB backs Domino's control plane state — you've debugged stuck workspaces caused by MongoDB state desync with Kubernetes reality. USE that story: "I've operated MongoDB as the state store of an ML platform — my debugging pattern was comparing the platform's recorded state in Mongo against actual Kubernetes state and reconciling."
- Know: connection strings with replica-set awareness, `rs.status()`, oplog (replication log), write concern (`w: majority` = durable ack), backups via `mongodump`/snapshots/Atlas.

### Q51. Snowflake — architecture in one minute?
Cloud data warehouse whose defining idea is **separation of storage and compute**:
- Data lives centrally in cloud object storage (columnar micro-partitions).
- **Virtual warehouses** = independent compute clusters that all query the same data — ETL, BI, and data science each get their own warehouse, sized independently, so workloads never contend. Suspend when idle → pay per-second only while running.
- Platform-engineer-relevant features: auto-suspend/auto-resume (the #1 cost control), **Time Travel** (query data as of a past point; undrop tables), zero-copy cloning (instant full env copies for testing), RBAC roles, and network policies/PrivateLink for enterprise access control.
- Ingestion from S3 via **stages + COPY INTO / Snowpipe** (continuous loading).

**Positioning line:** "As a platform engineer my Snowflake surface is access management, network connectivity (PrivateLink), cost governance via warehouse policies, and the ingestion pipelines from S3 — not writing analytics SQL."

---

# SECTION 10 — Terraform & Infrastructure as Code

### Q52. Why Infrastructure as Code? Why Terraform specifically?
**Why IaC:** infrastructure defined in versioned code gives you repeatability (identical envs), review (PRs on infra changes), auditability (git history = change history), disaster recovery (rebuild from code), and elimination of console-clicking drift.

**Why Terraform:** declarative (you state the end state, it computes the steps), **cloud-agnostic via providers** (AWS, GCP, Snowflake, GitLab — one workflow), a huge module ecosystem, and **plan-before-apply** — you see the exact diff before anything changes, which is what makes IaC safe in enterprises.

### Q53. Explain Terraform state — what it is, why it exists, and how you manage it in a team.
**What/why:** `terraform.tfstate` is Terraform's memory — a JSON mapping of "resource in my code" → "real resource ID in AWS" plus recorded attributes. Terraform needs it to know what it already created (so it updates instead of duplicating), to detect drift, and to compute deletions (a resource removed from code is found in state → planned for destruction).

**Team setup (say this fluently):**
- **Remote backend:** state in S3 — shared, durable, versioned (versioning on the bucket = state history/recovery), encrypted.
- **Locking:** DynamoDB lock table (or S3 native locking in newer versions) so two applies can't corrupt state concurrently.
- **State isolation:** separate state per environment/component (prod networking ≠ dev app) — limits blast radius; a corrupted or mis-applied state affects only its slice.
- **Never edit state by hand.** Use `terraform state mv/rm`, `terraform import`.
- **State is sensitive** — it can contain secrets in plaintext → encrypt, restrict access.

### Q54. Walk through what `terraform plan` and `apply` actually do.
1. `init` — install providers, configure backend, download modules.
2. `plan` — **refresh** (read real-world current state via provider APIs), **diff** (desired config vs state vs reality), build a **DAG** (dependency graph from references between resources), output proposed create/update/destroy actions. Crucially: some changes are in-place updates, others force **destroy-and-recreate** (marked `-/+` — e.g., changing an RDS identifier). Reading a plan for forced replacements *before* apply is the core production skill.
3. `apply` — execute the plan, walking the DAG (parallel where independent, ordered where dependent), update state.

### Q55. What is a Terraform module and what makes a *good* reusable module? (JD: "reusable, compliant, standardized")
A module is a folder of `.tf` files with defined **inputs (variables)** and **outputs** — a function for infrastructure. You write `module "vpc" { source = "..."; cidr = ... }` instead of 200 lines of resources.

**What makes it good — this is the senior answer:**
- **Sensible, secure defaults; small input surface.** Callers should get a compliant resource by default (e.g., an S3 module that enables encryption, versioning, and public-access-block *without being asked*). Compliance is baked in, not opt-in.
- **Versioned** (git tags / registry versions) — consumers pin versions (`?ref=v1.2.0`), so a module change never silently hits every environment; teams upgrade deliberately.
- **Composable, single-purpose** — a VPC module, an ALB module — not a "whole platform" mega-module.
- **Validated inputs** (`validation` blocks), documented, with examples and automated tests (at minimum `terraform validate` + plan in CI; ideally Terratest/`terraform test`).
- **Enforced standards:** required tags, naming conventions, allowed instance types — the module *is* the policy delivery mechanism. Plus org-level policy-as-code (OPA/Sentinel/Checkov) in the pipeline as the backstop.

### Q56. How does Terraform know the order to create resources?
From **references**: if a subnet's config references `aws_vpc.main.id`, Terraform infers subnet depends on VPC and builds a **DAG**. Independent branches run in parallel (default 10 concurrent). `depends_on` exists for dependencies Terraform can't see (rare; a smell if overused).

### Q57. What is drift and how do you handle it?
Drift = reality no longer matches state/code because someone changed things outside Terraform (console hotfix). `terraform plan` surfaces it (refresh reads reality). Handling: decide whether reality or code is correct → either codify the manual change or let apply revert it. Prevention: restrict console write access in managed accounts, run scheduled drift-detection plans in CI, treat "console change" as an incident-only exception with follow-up codification.

### Q58. Rapid-fire Terraform questions you should be ready for:
- **`count` vs `for_each`:** count is positional — removing item 0 *shifts indices* and destroys/recreates everything after it. `for_each` keys by name → stable. Prefer `for_each` for collections.
- **How do you adopt an existing (click-created) resource?** `terraform import` (or `import` blocks in modern TF) — brings it into state; then write matching config until plan is clean.
- **Secrets in Terraform?** Never hardcode. Feed via environment/CI variables, or better: resources read secrets at runtime from Secrets Manager/Vault; mark variables `sensitive`. Remember state still stores values → protect state.
- **Workspaces vs directories for environments?** Workspaces share one config with per-workspace state — fine for identical envs; most enterprises prefer directory-per-env (or Terragrunt) because prod usually *does* differ and needs isolation + separate pipelines.
- **What happens if apply fails midway?** Terraform is not transactional — resources created before the failure exist and are in state; you fix and re-apply (idempotent convergence). "Tainted" resources get recreated.
- **`terraform destroy` safety?** Guard prod with `prevent_destroy` lifecycle blocks, restricted pipelines, and no human credentials with delete rights.

---

# SECTION 11 — GitLab CI/CD

### Q59. Explain GitLab CI/CD's building blocks.
Everything is defined in **`.gitlab-ci.yml`** in the repo:
- **Pipeline** — a run, triggered by a commit/MR/schedule/manual.
- **Stages** — ordered phases (build → test → scan → deploy); stages run sequentially.
- **Jobs** — units of work inside a stage; jobs *within* a stage run **in parallel**.
- **Runners** — the machines that execute jobs (shared SaaS runners, or self-hosted — on EC2/Kubernetes — required in enterprises for network access to internal systems and for compliance).
- Each job typically runs in a **Docker container** you choose (`image:`), gets the repo checked out, runs `script:` lines; nonzero exit = job fails.

### Q60. How do jobs share data? (artifacts vs cache)
- **Artifacts** — files a job explicitly saves that are **passed to later stages** and downloadable (build outputs, test reports, terraform plan files). Guaranteed transfer.
- **Cache** — best-effort speed-up for *repeated* pipeline runs (dependency directories like `.npm`, `.terraform`). May be absent; never rely on it for correctness.
**Rule:** correctness → artifacts; speed → cache.

### Q61. How do you control when jobs run?
- **`rules:`** — modern conditional logic: run only on MRs, only on default branch, only when certain files changed (`changes:`), or make a job `when: manual` (approval gates for prod deploys) or `allow_failure: true`.
- **Protected branches + protected variables:** prod credentials are marked protected → only pipelines on protected branches (e.g., `main`) can access them. This is a key security control to mention.
- **Environments** — jobs declare `environment: production`, enabling deployment tracking, and GitLab environments can require approvals.

### Q62. Design a Terraform pipeline in GitLab. (Very likely question given this JD)
Describe this flow:
1. **validate stage:** `terraform fmt -check`, `terraform validate`, lint (tflint).
2. **security scan:** Checkov/tfsec/Trivy scanning the IaC for misconfigurations (public S3, open SGs) — fail the pipeline on criticals.
3. **plan stage:** `terraform plan -out=plan.tfplan`; save the **plan file as an artifact**; post the plan summary as an MR comment for review.
4. **apply stage:** `terraform apply plan.tfplan` — applies **exactly the reviewed plan** (not a fresh one — this closes the TOCTOU gap where infra changed between plan and apply); `when: manual`, protected branch only, after MR approval.
5. **Auth:** runner assumes an IAM role via **OIDC** — GitLab issues a short-lived JWT, AWS trusts it → **no long-lived AWS keys stored in CI variables**. Saying "OIDC federation instead of static keys" is a strong senior marker.
6. Per-environment: MR → plan against dev, merge → apply dev, manual gate → plan/apply prod.

### Q63. How do you integrate testing and security scanning into pipelines? (JD: "testing, security scanning, and deployment controls")
Layered scanning, each catching a different class of problem:
- **SAST** (static analysis of your code — Checkmarx, GitLab SAST, Semgrep) — insecure code patterns (SQL injection, hardcoded secrets in code paths).
- **Secret detection** — committed credentials (run on every push; a leaked key in git history is compromised forever).
- **Dependency/SCA scanning** — known CVEs in libraries.
- **Container scanning** (Trivy/Grype on the built image) — vulnerable OS packages in your Docker image.
- **IaC scanning** (Checkov/tfsec) — cloud misconfigurations before they're applied.
- **DAST** — probing the *running* app in a test environment.
**Deployment controls:** manual approval gates, protected environments, and policies that block merge/deploy on critical findings — with a triage/exception workflow so the pipeline doesn't get bypassed the moment it's inconvenient.

### Q64. What is Checkmarx specifically? (JD lists it by name)
An enterprise **SAST** platform (Checkmarx One suite also does SCA/IaC). It parses source code into a queryable representation and traces **data flow from sources (user input) to sinks (SQL query, file write)** to find injection-class vulnerabilities — without running the code.

**Platform-engineer angle (your role):** you don't fix the Java findings; you **integrate Checkmarx into GitLab CI** — a scan job calling the Checkmarx CLI/API, results feeding back to MRs, pipeline gates on severity thresholds, managing false-positive triage flow with AppSec, and tuning presets so scan time doesn't destroy pipeline speed (e.g., incremental scans on MRs, full scans nightly). Frame it exactly like that.

### Q65. How do you optimize a slow GitLab pipeline?
- Parallelize independent jobs within stages; use `needs:` to build a DAG and start jobs as soon as *their* dependencies finish rather than waiting for whole stages.
- Cache dependencies; use smaller/purpose-built images; pre-bake toolchains into a custom runner image instead of `apt-get install` every job.
- `rules: changes:` so docs-only commits don't run the world; split mono-repo pipelines per component.
- Docker layer caching for image builds (BuildKit, `--cache-from`).
- Move heavy scans (full SAST) to scheduled pipelines; keep MR pipelines incremental.

---

# SECTION 12 — Docker

### Q66. What is a container, really? How is it different from a VM?
**Mental model:** a container is **a normal Linux process** wrapped in isolation. Two kernel features do all the work:
- **Namespaces** — isolate what the process *sees*: its own PID tree, network stack, mount points, hostname, users.
- **cgroups** — limit what it can *use*: CPU, memory, IO.
Plus a layered filesystem for its root.

**VM vs container:** a VM virtualizes hardware and runs a **full guest OS with its own kernel** (minutes to boot, GBs). Containers **share the host kernel** (milliseconds to start, MBs). That's why containers are dense and fast — and why the isolation boundary is weaker than a VM's (kernel vulnerabilities cross containers).

**One-liner:** "A container is not a lightweight VM — it's an isolated process. Namespaces control visibility, cgroups control resources, and everything shares one kernel."

### Q67. Image vs container? What are image layers?
- **Image** = immutable, read-only template (filesystem + metadata). **Container** = a running instance: the image's layers plus one thin **writable layer** on top. Class vs object.
- Each Dockerfile instruction creates a **layer**; layers are content-addressed and **shared** between images (ten images on one base pull the base once). Pushes/pulls transfer only missing layers.
- **Consequence for Dockerfile design:** layer caching — a change in one instruction invalidates that layer *and everything after it*. Hence the golden ordering: rarely-changing steps first (base image, dependency install), frequently-changing steps last (`COPY . .`). Copy `package.json`/`requirements.txt` and install deps *before* copying source, so code changes don't re-run dependency installation.

### Q68. What are Dockerfile best practices you'd enforce?
1. **Pinned, minimal base images** (`python:3.12-slim`, distroless/alpine where feasible) — smaller attack surface, faster pulls; never floating `latest`.
2. **Layer ordering for cache** (above).
3. **Multi-stage builds** — build with the heavy toolchain in stage 1, `COPY --from=build` only the artifact into a slim runtime stage → small, tool-free production images.
4. **Run as non-root** (`USER app`) — container escape from root is far worse; many platforms enforce this.
5. **No secrets in images** — not in ENV, not in any layer (deleted files still exist in earlier layers!); inject at runtime or use BuildKit `--mount=type=secret`.
6. `.dockerignore` (keep `.git`, creds, junk out of build context), one process per container, meaningful `HEALTHCHECK`, scan images (Trivy) in CI.

### Q69. How do you troubleshoot a container that keeps crashing or misbehaving?
1. `docker ps -a` — status/exit code. **Exit code 137 = SIGKILL, almost always OOM-killed** (memory limit); 1/other = app error; 126/127 = bad entrypoint/command.
2. `docker logs <c>` — the app's stdout/stderr (containers should log to stdout, not files).
3. `docker inspect` — actual config: env, mounts, limits, restart policy, `OOMKilled: true` flag.
4. `docker exec -it <c> sh` — poke inside a *running* container (check files, DNS, connectivity). If it dies instantly, override the entrypoint: `docker run -it --entrypoint sh image` and run the real command manually to watch it fail.
5. `docker stats` — live CPU/memory vs limits.
6. Networking: containers on the same user-defined bridge network resolve each other **by name** via Docker's embedded DNS (127.0.0.11); "connection refused between containers" is usually wrong network, wrong port (container port vs published host port), or app bound to 127.0.0.1 instead of 0.0.0.0 — that last one is a classic.

### Q70. Explain Docker networking and volumes in brief.
- **bridge** (default): containers get private IPs on a virtual bridge; `-p 8080:80` publishes container port 80 on host 8080 via NAT. **User-defined bridges** add name-based DNS.
- **host**: no isolation, container shares host network stack (performance/edge cases).
- **Volumes** (managed by Docker) vs **bind mounts** (host path mapped in): data must outlive the container's writable layer, so databases/state go on volumes. "Containers are ephemeral; anything worth keeping lives in a volume."

---

# SECTION 13 — Linux Administration & Performance

### Q71. A Linux server is "slow." Walk me through your diagnosis. (Guaranteed question — use the USE method)
State the framework first: **USE method (Brendan Gregg) — for every resource check Utilization, Saturation, Errors.**

1. **First 60 seconds:** `uptime` (load averages — trend?), `dmesg -T | tail` (OOM kills? disk errors?), `top`/`htop` (who's eating CPU/memory? what's the `wa` iowait %?).
2. **CPU:** `mpstat -P ALL 1` (one hot core = single-threaded bottleneck), `pidstat 1` (per-process). High **load with low CPU%** → processes stuck in IO wait (D state) → look at disk.
3. **Memory:** `free -m` — remember **buff/cache is reclaimable; "available" is the real number**. Swap in/out (`vmstat 1` — si/so columns) = real memory pressure. Check `dmesg` for OOM-killer.
4. **Disk:** `iostat -xz 1` — %util, await (latency), queue size. High await = storage bottleneck (on AWS: are we throttled at EBS volume IOPS or instance EBS bandwidth?).
5. **Network:** `sar -n DEV 1`, `ss -s` (connection counts), retransmits.
6. Then go app-level: `strace`/`perf` if a specific process is the suspect.

**Load average nuance to volunteer:** Linux load counts *runnable + uninterruptible-IO* tasks — so load 30 on a 4-core box might be CPU starvation *or* 26 processes stuck on a dead NFS mount. iowait distinguishes them.

### Q72. Explain Linux memory: why does `free` show almost no free memory on a healthy box?
Linux uses idle RAM as **page cache** — caching file data to make IO fast. It's given back instantly when applications need it. So "free" being tiny is *good*; the number that matters is **available**. Related: OOM killer (kernel kills the process with the worst oom_score when truly out of memory — check `dmesg`), and swap (overflow to disk; heavy swapping = thrashing = the real "server is dying" signal).

### Q73. What happens during the Linux boot process / how do you manage services?
BIOS/UEFI → bootloader (GRUB) → kernel + initramfs → **systemd** (PID 1) → targets/units.
Daily ops verbs: `systemctl status/start/enable unit`, `journalctl -u unit -f --since "1 hour ago"` for logs, unit files in `/etc/systemd/system/` (`daemon-reload` after edits), `Restart=on-failure` for resilience. Be ready to sketch a minimal unit file: `[Unit] After=network.target`, `[Service] ExecStart=...`, `[Install] WantedBy=multi-user.target`.

### Q74. Rapid-fire Linux you must not fumble:
- **File permissions:** `rwx` for user/group/other; `chmod 750`; special bits — setuid, setgid (on dirs: inherit group), sticky bit (/tmp). Directories need `x` to enter.
- **Find big files / full disk:** `df -h` (which filesystem), `du -xh --max-depth=1 /` walk-down; **deleted-but-open files** still hold space — `lsof +L1` (classic: rotated log deleted while a process holds it open; df and du disagree).
- **Inodes:** disk "full" with space free = inode exhaustion (`df -i`) from millions of tiny files.
- **Process states:** R running, S sleeping, D uninterruptible IO (can't be killed — usually storage/NFS trouble), Z zombie (dead, parent hasn't reaped; harmless in small numbers, a leak in large).
- **Signals:** SIGTERM (15, polite, allows cleanup) vs SIGKILL (9, immediate, no cleanup — last resort); SIGHUP often = reload config.
- **`/proc`:** kernel's live view — `/proc/<pid>/limits`, `/proc/meminfo`; ulimits (open-files limit causing "too many open files").

### Q75. Shell scripting — what do you actually get asked?
- **Safety header:** `set -euo pipefail` — exit on error, on unset variables, and fail a pipeline if any stage fails. Quoting variables (`"$var"`) to survive spaces. Explain *why*: unguarded scripts continue past failures and destroy things.
- **Exit codes:** 0 = success; check `$?` or use `if command; then`. Every automation decision hangs on exit codes.
- **Core idioms:** loops over files (`for f in *.log`), `while read -r line`, functions, `trap cleanup EXIT` (guaranteed cleanup — temp files, locks), here-docs, command substitution `$(...)`.
- **Text processing trio:** `grep -E` (find), `awk '{print $2}'` (columns/aggregation), `sed 's/x/y/'` (edit streams); plus `sort | uniq -c | sort -rn` for "top N" analyses, `jq` for JSON (API/CLI output).
- **When *not* shell:** beyond ~100 lines, needing data structures/error handling → Python. Saying you know the boundary is a senior answer.
- Be ready to sketch on the spot: a log-cleanup script (`find /var/log -name "*.log" -mtime +30 -delete` with dry-run flag), a health-check loop with retry/backoff, or parsing an API response with `curl | jq`.

---

# SECTION 14 — Security, Compliance & Governance in AWS

### Q76. How does IAM policy evaluation actually work?
For any API call: start at implicit **deny** → an explicit **Allow** (identity or resource policy) permits → but any explicit **Deny** anywhere overrides everything → then the allow must survive intersection with **SCPs**, **permission boundaries**, and **session policies**. Order to recite: *explicit deny > SCP > permission boundary > allow > implicit deny.*
Adjacent must-knows: **roles vs users** (roles = assumable, temporary credentials — the enterprise standard; long-lived user keys are an anti-pattern), instance profiles / IRSA (workloads get roles, not keys), least privilege, and `Condition` keys (MFA, SourceIp, PrincipalOrgID).

### Q77. What does "governance and access policy management" look like in a mature AWS org?
- **Multi-account strategy:** accounts as the primary blast-radius boundary (per team/env), organized in OUs.
- **Preventive controls:** SCPs (guardrails — Q43), permission boundaries (cap what delegated admins can grant), IAM Identity Center (SSO, no local users).
- **Detective controls:** org-wide CloudTrail (immutable, centralized), AWS Config rules (continuous compliance evaluation — "no unencrypted volumes"), Security Hub aggregation, GuardDuty (threat detection).
- **Access hygiene:** everything through roles + SSO, credential reports, Access Analyzer (find unintended external access), periodic access reviews — which in pharma/GxP contexts is an audit requirement you can speak to from experience.

### Q78. How do you embed security into engineering processes rather than bolting it on? ("shift-left")
The pipeline answer from Q63, plus the platform framing: **make the secure path the easy path** — Terraform modules with compliant defaults, golden base images patched centrally, secrets manager integration provided out of the box, scanning gates with a fast triage/exception process. "Security controls that slow engineers down get bypassed; my job as a platform engineer is building controls into the paved road so compliance is the default."

### Q79. What is Google SecOps and what does "supporting its workflows" mean for you?
Google SecOps (formerly Chronicle) is Google's cloud **SIEM/SOAR**: ingests security telemetry at massive scale, normalizes it into **UDM**, retains it (12 months standard), runs **YARA-L detection rules**, and supports SOC investigation and automated response playbooks.
**Your role as platform engineer isn't SOC analysis — it's the plumbing:** building/maintaining ingestion (feeds from AWS CloudTrail, VPC Flow Logs, GuardDuty; forwarders/BindPlane for on-prem and app logs), ensuring parser/normalization health, monitoring feed lag and gaps (a silent dead feed = blind SOC), and supporting the Splunk→SecOps migration mechanics from Q44. Frame it exactly like that — it's honest and it's what they need.

### Q80. How do you handle secrets across the platform?
Never in code, images, or CI variables in plaintext where avoidable. Hierarchy: **AWS Secrets Manager / SSM Parameter Store** (with KMS) for AWS-native; Vault in multi-cloud shops; ECS/Lambda native secret injection; Kubernetes external-secrets. Short-lived credentials over static ones everywhere: IAM roles, **OIDC federation for CI** (Q62), IRSA for pods. Rotation automated (Secrets Manager rotation lambdas for RDS). Detection backstop: secret scanning in CI + push protection.

---

# SECTION 15 — Behavioral & Positioning (Read This Last, It Matters Most)

### Q81. "Tell me about yourself" — your 60-second version for THIS job.
"I'm a platform/infrastructure engineer with about seven years across Linux, AWS, Kubernetes, and Docker. Most recently I've supported an enterprise data-science platform (Domino) running on EKS in a regulated life-sciences environment — so my day-to-day is exactly the intersection this role describes: AWS networking and troubleshooting, Terraform-managed infrastructure, GitLab CI/CD pipelines, and executing changes under strict change control. Before that I worked through TCS and Accenture with enterprise clients like JP Morgan and Disney, which is where I learned to operate in controlled, process-heavy environments. I'm looking for a senior platform role where I own infrastructure change execution end to end."

### Q82. "Have you done an AWS org migration / Splunk-to-Google migration?" — the honest bridge answer.
Don't bluff. Bridge: "I haven't executed that exact migration, but I've executed the same *class* of change — [cutover/upgrade you've done] under change control: inventory and dependency mapping first, staged execution with validation gates, rollback criteria defined before the window, multi-vantage validation after. The tooling differs; the discipline is identical, and I ramp on tooling fast — here's what I already understand about how that migration works…" — then show Q42/Q44 knowledge. Knowing the failure modes of a migration you haven't done is *more* impressive than claiming you've done it.

### Q83. Prepare 3 STAR stories mapped to this JD before tomorrow:
1. **A hard production troubleshooting story** (networking or Kubernetes/EKS — e.g., the stuck-workspace/MongoDB state desync diagnosis) → proves systematic debugging.
2. **A change you executed under change control** (upgrade, cert rotation, DNS/infra change with CAB, rollback plan) → proves the "controlled enterprise environment" requirement.
3. **An automation win** (pipeline you built/optimized, Terraform you modularized, script that removed toil) → proves the IaC/CI/CD requirement.
For each: Situation (2 lines), Task, Action (the meat — YOUR steps), Result (quantified if possible).

### Q84. Questions to ask THEM (always ask 2–3):
- "What's the current state of the AWS org migration — planning, mid-flight, or stabilizing? What's been the hardest dependency so far?"
- "How mature is the Terraform module library — building it or maintaining it?"
- "How does change management work here — CAB cadence, and how much automation is trusted for prod changes?"
- "What does success look like for this role in the first 90 days?"

---

## Final-night priorities (if time is short, master these in order)
1. **TGW two-hop routing + segmentation (Q9–Q11)** — explicitly in the JD, high signal.
2. **Route 53 Resolver endpoints + TTL discipline (Q18–Q19)** — explicitly in the JD.
3. **Migration methodology + SCP safe-change process (Q42–Q46)** — the differentiating section.
4. **Terraform state, modules, GitLab TF pipeline with OIDC (Q53–Q55, Q62)**.
5. **TLS handshake + ACM constraints + Zscaler trust store story (Q21–Q26)**.
6. **Linux USE-method walkthrough (Q71)** — the guaranteed live question.
7. **Your bridge answers and STAR stories (Q81–Q83)**.

You already know more of this than you think — most of it is your EKS/Domino experience wearing different labels. Go in leading with your strengths.
