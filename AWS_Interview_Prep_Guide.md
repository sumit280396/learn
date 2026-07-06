# AWS Interview Prep Guide (Zero to Deep Understanding)

> How to use this guide: Each topic starts with a **plain-English explanation** (assume you know nothing), followed by **"why does this exist"**, then **interview questions with detailed answers**, and finally **troubleshooting tips** you can use on the job. Read the explanation before the Q&A — the Q&A will make much more sense once you have the mental model.

---

# 1. VPC (Virtual Private Cloud)

## The Mental Model
Imagine AWS is a giant city, and every company that uses AWS needs its own private, gated neighborhood inside that city — with its own streets, its own gate rules, and its own security guards. That gated neighborhood is a **VPC**. It's an isolated network inside AWS where you launch your servers (EC2), databases, containers, etc. Nobody outside your VPC can just wander into it unless you allow it.

Key building blocks inside a VPC:
- **CIDR block**: The range of IP addresses your neighborhood is allowed to use (e.g., `10.0.0.0/16` = about 65,536 addresses).
- **Subnets**: Smaller streets within the neighborhood. A subnet lives inside ONE Availability Zone (AZ = a physical data center).
  - **Public subnet**: Has a route to the internet (via Internet Gateway). Think "houses facing the main road."
  - **Private subnet**: No direct route to internet. Think "houses in a cul-de-sac, no direct road out."
- **Internet Gateway (IGW)**: The neighborhood's main gate to the outside world/internet. Attached to the VPC, used by public subnets.
- **Route Table**: The "signboard" at every street corner telling traffic where to go (covered in detail in its own section).
- **Security Group (SG)**: A virtual firewall attached to individual resources (like a bodyguard for one house) — stateful (if you allow traffic in, the reply is automatically allowed out).
- **Network ACL (NACL)**: A firewall at the subnet level (like a checkpoint at the entrance of a street) — stateless (you must explicitly allow both inbound AND outbound).

## Why It Exists
Without a VPC, every AWS customer's servers would just be floating in a shared pool — a security nightmare. VPC gives you **isolation, control over IP addressing, and control over traffic flow**, which is essential for security and compliance.

## Interview Questions & Answers

**Q1: What is a VPC and why do we need it?**
A: A VPC (Virtual Private Cloud) is a logically isolated section of AWS where you control your own IP address range, subnets, route tables, and gateways. We need it because it lets us build a private network in the cloud with the same level of control we'd have in an on-premise data center — deciding what's public-facing, what's private, and how traffic flows — while still getting cloud scalability.

**Q2: What's the difference between a public subnet and a private subnet?**
A: The actual technical difference isn't a label AWS assigns — it comes down to the **route table** attached to the subnet. A subnet is "public" if its route table has a route sending `0.0.0.0/0` (all internet-bound traffic) to an Internet Gateway. It's "private" if that route is missing (or instead points to a NAT Gateway for outbound-only access). This trips up a lot of beginners — there's no checkbox called "make this public," it's purely about routing.

**Q3: What is CIDR notation and how do you calculate the number of usable IPs?**
A: CIDR (Classless Inter-Domain Routing) notation like `10.0.0.0/16` specifies a network address and a "prefix length" (16) that tells you how many bits are fixed for the network vs. flexible for hosts. Fewer bits fixed (smaller number after the slash) = more available IPs. A `/16` gives 65,536 IPs, a `/24` gives 256 IPs. AWS reserves 5 IPs in every subnet (network address, VPC router, DNS, future use, and broadcast address), so a `/24` subnet actually gives you 251 usable IPs.

**Q4: What's the difference between a Security Group and a Network ACL?**
A: 
- Security Group (SG): operates at the instance/ENI level, is **stateful** (return traffic is automatically allowed), and can only have "allow" rules (no explicit deny).
- Network ACL (NACL): operates at the subnet level, is **stateless** (you must write rules for both directions), and can have explicit "allow" and "deny" rules, evaluated in numbered order.
Think of the SG as personal security for a specific person, and the NACL as security at the building's front gate — everyone in that subnet is subject to it regardless of individual permissions.

**Q5: Can two VPCs communicate with each other by default?**
A: No. VPCs are isolated by default, even within the same AWS account. To connect them you need **VPC Peering**, a **Transit Gateway**, or **PrivateLink**, depending on the use case and scale.

**Q6: What is an Elastic Network Interface (ENI)?**
A: An ENI is a virtual network card that gets attached to an EC2 instance. It holds the private IP, MAC address, and security groups. Understanding ENIs matters for troubleshooting — e.g., when Lambda functions run inside a VPC, AWS attaches ENIs to give them network access to private resources.

**Q7: What happens if you don't have an Internet Gateway attached to your VPC?**
A: No resource in that VPC can reach the public internet directly, and nothing on the internet can reach in — full isolation. This is common for sensitive internal-only workloads (e.g., databases) that should never be reachable from the internet.

**Q8: How do resources in a private subnet access the internet (e.g., to download a security patch)?**
A: They can't directly. You route their outbound traffic through a **NAT Gateway** (or NAT instance) sitting in a public subnet. The private resource never gets a public IP, so nothing on the internet can initiate a connection back to it — outbound-only. (Full detail in the NAT Gateway section.)

**Q9: What is VPC Peering and what are its limitations?**
A: VPC Peering directly connects two VPCs so resources can talk using private IPs, as if they were on the same network. Limitations: it's **non-transitive** (if A is peered with B, and B is peered with C, A cannot talk to C through B — you'd need a direct A-C peering), CIDR ranges must not overlap, and it doesn't scale well beyond a handful of VPCs (becomes a mesh nightmare) — which is exactly why Transit Gateway exists.

**Q10: What is a VPC Endpoint, and why would you use one?**
A: A VPC Endpoint lets your VPC privately connect to supported AWS services (like S3, DynamoDB) **without going through the public internet or a NAT Gateway**. There are two types: **Gateway Endpoints** (free, for S3/DynamoDB, added as a route table entry) and **Interface Endpoints** (powered by PrivateLink, use an ENI with a private IP, cost money per hour). Why use them? Security (traffic never leaves AWS's network) and cost (you can avoid NAT Gateway data-processing charges for things like S3 access).

## Troubleshooting Tips
- **"My EC2 instance in a public subnet still can't be reached from the internet"**: Check, in order: (1) does the subnet's route table have `0.0.0.0/0 → IGW`? (2) does the instance have a public IP or Elastic IP? (3) does the Security Group allow inbound on that port? (4) does the NACL allow inbound AND outbound on that port/ephemeral range?
- **"Two VPCs are peered but still can't talk"**: Check both sides' route tables point to the peering connection, and that Security Groups on both ends allow the traffic — peering doesn't override SGs.
- Always troubleshoot networking top-down: Route Table → Security Group → NACL → OS-level firewall (like iptables) — in that order, since each is a different layer that can silently drop traffic.

---

# 2. Route Tables

## The Mental Model
A route table is literally a list of instructions: "If traffic is going to THIS destination, send it THROUGH this path." Every subnet must be associated with exactly one route table (though one route table can serve many subnets). When you don't explicitly associate a subnet, it uses the VPC's **main route table**.

Each row (route) has:
- **Destination**: a CIDR block (e.g. `0.0.0.0/0` means "everywhere," `10.0.1.0/24` means a specific subnet)
- **Target**: where to send matching traffic (Internet Gateway, NAT Gateway, Transit Gateway, Peering Connection, "local," etc.)

AWS always picks the **most specific matching route** (longest prefix match) — similar to how you'd follow the most detailed direction available.

## Why It Exists
Without route tables, AWS wouldn't know whether traffic destined for a certain IP should go to the internet, to another VPC, to on-premises via VPN, or just stay local. It's the traffic cop of your VPC.

## Interview Questions & Answers

**Q1: What is the "local" route and can you delete it?**
A: Every route table automatically gets a route for the VPC's own CIDR block with a target of "local" — this allows all subnets within the VPC to talk to each other by default. You cannot delete or modify this route; it's baked in.

**Q2: What is the difference between the main route table and a custom route table?**
A: The main route table is the default one AWS creates with your VPC; any subnet not explicitly associated with another table uses it. A custom route table is one you create yourself to control specific subnets differently (e.g., a private subnet's table routing to a NAT Gateway instead of an IGW). Best practice: don't leave critical routes in the main table — explicitly assign route tables to every subnet so nothing is "accidentally public."

**Q3: If two routes could match the same destination, which one wins?**
A: AWS uses the **longest prefix match** rule — the most specific (smallest range) route wins. For example, a route for `10.0.1.0/24` wins over a route for `0.0.0.0/0` for traffic going to `10.0.1.5`.

**Q4: How does a route table make a subnet "public"?**
A: By having a route where destination = `0.0.0.0/0` and target = the Internet Gateway. That's the entire definition — nothing else.

**Q5: Can a single route table be attached to multiple subnets?**
A: Yes. Many subnets can share one route table. But a subnet can only have ONE route table attached at a time.

## Troubleshooting Tips
- If traffic seems to "disappear," check the route table associated with the SPECIFIC subnet the resource is in — people often check the VPC's main table and forget the resource's subnet uses a different custom table.
- Symptom: "Instance can't reach the internet" but SG/NACL look fine → 90% of the time it's a missing or wrong route table entry.
- Use the **Route Table tab → Routes tab** in the console, and cross-check against the **Subnet Associations tab**, to confirm you're editing the table that's actually attached to the relevant subnet.

---

# 3. NAT Gateway

## The Mental Model
NAT = Network Address Translation. A NAT Gateway is like a **one-way mail forwarding service**: private resources (in a private subnet, with no public IP) want to reach the internet — say, to download a package update — but should NOT be reachable FROM the internet (no one should be able to just message them). The NAT Gateway sits in a public subnet, has its own Elastic IP, and forwards outbound requests on behalf of private instances, then routes the responses back — but nobody outside can initiate a NEW connection into the private instance through the NAT Gateway.

Analogy: it's like calling out from a company's front desk — outsiders can respond to your call, but they can't dial an extension directly to reach your desk.

## Why It Exists
Private subnets are private for a reason — usually databases, internal app servers — but they still often need OUTBOUND internet access (patches, calling external APIs, etc.). NAT Gateway provides that without exposing them to inbound internet traffic.

## Interview Questions & Answers

**Q1: What is a NAT Gateway and why would you use it instead of just giving instances a public IP?**
A: A NAT Gateway lets instances in a private subnet initiate outbound internet connections (e.g., downloading updates, calling third-party APIs) without exposing them to inbound internet traffic. If you gave every instance a public IP instead, they'd be directly reachable from the internet — a huge attack surface risk for things like databases and internal servers that should never accept unsolicited inbound traffic.

**Q2: Where does a NAT Gateway live — public subnet or private subnet?**
A: It must be placed in a **public subnet** (so it can reach the Internet Gateway) and needs an Elastic IP attached. The private subnet's route table then points `0.0.0.0/0` traffic to this NAT Gateway.

**Q3: What's the difference between a NAT Gateway and a NAT Instance?**
A: NAT Gateway is a fully managed AWS service — highly available within an AZ, auto-scales bandwidth, no patching needed, but costs an hourly fee + per-GB data processing fee. A NAT Instance is just a regular EC2 instance configured to do NAT — cheaper for low-traffic use cases, but you must manage/patch/scale it yourself, and it's a single point of failure unless you build redundancy. In real-world interviews, the expected answer is "NAT Gateway is preferred for production due to being managed and highly available, NAT Instance is used for cost savings in dev/small-scale environments."

**Q4: Is a NAT Gateway highly available across Availability Zones?**
A: A single NAT Gateway lives in ONE AZ. If that AZ goes down, the NAT Gateway goes down with it. Best practice for high availability: deploy **one NAT Gateway per AZ**, and have each AZ's private subnets route through the NAT Gateway in their own AZ. This avoids cross-AZ data transfer charges too.

**Q5: Does a NAT Gateway allow inbound connections initiated from the internet?**
A: No — that is the entire point. It's strictly for translating and forwarding OUTBOUND-initiated traffic and its return responses. If you need something reachable from the internet, that resource needs to be in a public subnet with its own public IP (or sit behind a Load Balancer / API Gateway / CloudFront).

**Q6: Why is NAT Gateway sometimes a major cost driver in AWS bills, and how do you reduce that cost?**
A: NAT Gateway charges both an hourly rate AND a per-GB data processing fee for everything that passes through it — this adds up fast for high-traffic workloads (e.g., large S3 downloads routed through it). To reduce cost: use **VPC Gateway Endpoints** for S3/DynamoDB traffic (free, bypasses NAT entirely), consolidate NAT Gateways where HA isn't critical, or use NAT Instances for low-traffic dev environments.

## Troubleshooting Tips
- **"My private EC2 instance can't reach the internet"**: Check (1) private subnet's route table has `0.0.0.0/0 → NAT Gateway ID`, (2) the NAT Gateway itself is in "Available" state, (3) the NAT Gateway's own subnet (public) has a route to the IGW, (4) the NAT Gateway has an Elastic IP attached, (5) Security Groups/NACLs allow outbound.
- **"It works in one AZ but not another"**: Classic sign of only having ONE NAT Gateway and private subnets in other AZs still routing to it correctly, OR (more often) each AZ's private subnet route table pointing to a NAT Gateway that doesn't exist in that AZ / wasn't set up per-AZ.
- Check **CloudWatch metrics** for the NAT Gateway (`BytesOutToDestination`, `ErrorPortAllocation`) — port allocation errors mean you're running out of available ports due to too many concurrent connections, a sign you may need multiple NAT Gateways or connection pooling on the client side.

---

# 4. Transit Gateway

## The Mental Model
Imagine you have 10 VPCs and need them all to talk to each other, plus talk to your on-premises data center. Doing this with VPC Peering would require a "mesh" of connections — with 10 VPCs, that's 45 individual peering connections to manage (and peering isn't transitive, remember). 

A **Transit Gateway (TGW)** is like a central **airport hub**. Instead of every city having a direct flight to every other city, all cities fly through the hub, and the hub routes you onward. Each VPC (and VPN, and Direct Connect connection) attaches to the Transit Gateway once, and the TGW handles routing between all of them.

## Why It Exists
At scale, VPC Peering's mesh-of-connections approach becomes unmanageable. Transit Gateway centralizes routing, dramatically simplifying network architecture for large organizations with many VPCs/accounts/on-prem connections.

## Interview Questions & Answers

**Q1: What problem does Transit Gateway solve that VPC Peering doesn't?**
A: VPC Peering doesn't scale — connections are one-to-one and non-transitive, so N VPCs need up to N(N-1)/2 peering connections to fully mesh. Transit Gateway solves this with a hub-and-spoke model: each VPC connects once to the TGW, and the TGW routes traffic between all attached networks, including on-premises via VPN/Direct Connect. It turns an unmanageable mesh into a simple hub.

**Q2: Is traffic through a Transit Gateway transitive?**
A: Yes — that's the key differentiator from peering. If VPC A and VPC B are both attached to the same TGW, they can communicate through it (subject to TGW route tables allowing it), even without a direct peering connection.

**Q3: What is a Transit Gateway Route Table, and how is it different from a VPC route table?**
A: A TGW has its own route tables, separate from VPC route tables. VPC route tables route traffic FROM the VPC TO the TGW (usually `0.0.0.0/0 → TGW` or specific CIDRs). The TGW's own route table then decides how to route that traffic BETWEEN attachments (VPCs, VPNs, Direct Connect). You can have multiple TGW route tables to segment traffic — e.g., a "production" route table that only routes between prod VPCs, isolating them from a "dev" TGW route table.

**Q4: Can Transit Gateway connect VPCs across different AWS accounts or regions?**
A: Yes. Cross-account sharing is done via **AWS Resource Access Manager (RAM)**. Cross-region connectivity is done via **Transit Gateway Peering** between TGWs in different regions.

**Q5: What are common use cases for Transit Gateway?**
A: Connecting dozens/hundreds of VPCs in a large organization; centralizing egress to the internet through a shared VPC (reducing the number of NAT Gateways needed); connecting on-premises data centers to many VPCs via a single VPN/Direct Connect attachment instead of one per VPC; multi-account network segmentation (e.g., isolating prod from dev traffic using separate TGW route tables).

## Troubleshooting Tips
- **"VPCs attached to the same TGW still can't communicate"**: Check three separate layers: (1) VPC route table routes the destination CIDR to the TGW, (2) TGW route table actually has a route associating the two attachments, (3) Security Groups on the destination allow the traffic. All three commonly get missed simultaneously by beginners.
- Remember TGW route tables are separate per-attachment associations — an attachment is only "listening" to the TGW route table it's associated with, so double check attachment-to-route-table associations, not just the routes themselves.

---

# 5. Route 53

## The Mental Model
Route 53 is AWS's **DNS (Domain Name System) service** — think of DNS as the internet's phonebook. Humans type `www.example.com`; computers need an IP address like `192.0.2.1` to actually connect. Route 53 translates the human-friendly name into the machine-friendly address.

("53" refers to the standard port number DNS uses — a nerdy AWS naming convention.)

Route 53 does three main jobs:
1. **Domain registration** — you can buy domain names through it.
2. **DNS resolution** — answering "what's the IP for this domain?"
3. **Health checking & routing policies** — deciding WHICH IP to return based on health, location, load, etc.

## Why It Exists
Every application on the internet needs a reliable, fast way for users to find it by name instead of memorizing IP addresses, and businesses need smart routing (e.g., send European users to a European server) and automatic failover if a server goes down.

## Interview Questions & Answers

**Q1: What is DNS, in simple terms, and what role does Route 53 play?**
A: DNS converts human-readable domain names into IP addresses computers use to route traffic. Route 53 is AWS's managed DNS service — it hosts your domain's DNS records and answers lookup queries reliably and quickly from a globally distributed network of servers.

**Q2: What is a Hosted Zone?**
A: A Hosted Zone is a container for all the DNS records for one domain (e.g., `example.com`). There are two types: **Public Hosted Zones** (answer queries from the public internet) and **Private Hosted Zones** (only resolve within specified VPCs — useful for internal-only domain names).

**Q3: What are the most common DNS record types and what do they do?**
A: 
- **A record**: maps a domain name to an IPv4 address.
- **AAAA record**: maps a domain name to an IPv6 address.
- **CNAME**: maps a domain name to ANOTHER domain name (an alias), not directly to an IP.
- **Alias record**: an AWS-specific enhancement similar to CNAME but can be used at the root domain (`example.com` itself) and points directly to AWS resources (like a CloudFront distribution or Load Balancer) at no extra query cost.
- **MX record**: routes email for the domain.
- **TXT record**: holds arbitrary text, often used for domain verification (e.g., proving you own a domain to a third-party service) or SPF/DKIM email security.
- **NS record**: lists the name servers responsible for the domain.

**Q4: What's the difference between a CNAME and an Alias record, and why does AWS recommend Alias?**
A: A CNAME cannot be used on the "zone apex" (the root domain, e.g., `example.com` without `www`) due to DNS standards — it can only be used on subdomains like `www.example.com`. An Alias record is Route 53's own extension that behaves like a CNAME but is allowed at the root domain, and it directly resolves to AWS resources without an extra DNS lookup hop, plus it's free (no charge per query) — so AWS recommends it whenever pointing to an AWS resource (CloudFront, ALB, S3 static site, etc.).

**Q5: What are Route 53 routing policies, and when would you use each?**
A: 
- **Simple**: one record, one (or a static list of) resource(s) — default, no special logic.
- **Weighted**: split traffic by percentage across multiple resources — great for A/B testing or gradual rollouts (e.g., 90% old version, 10% new version).
- **Latency-based**: routes users to whichever region gives them the lowest latency — good for global apps with regional deployments.
- **Failover**: routes to a primary resource, automatically switching to a secondary if the primary fails a health check — used for disaster recovery.
- **Geolocation**: routes based on the user's geographic location (e.g., legal/content restrictions, or serving localized content).
- **Geoproximity**: like geolocation but lets you shift traffic between regions by adjusting a "bias," and works with AWS Traffic Flow.
- **Multivalue answer**: returns multiple healthy IPs randomly, providing basic load balancing/redundancy (not a substitute for a full load balancer, but lightweight DNS-level redundancy).

**Q6: How does Route 53 health checking work, and what does it actually check?**
A: Route 53 health checks periodically send requests (HTTP/HTTPS/TCP) to an endpoint from multiple global locations. If a threshold of failures is reached, the endpoint is marked unhealthy, and Route 53 stops returning its IP for queries (e.g., in a failover routing policy), automatically shifting traffic to healthy resources. This enables automated failover without manual intervention.

**Q7: What is the TTL (Time To Live) in DNS, and why does it matter?**
A: TTL tells other DNS resolvers (like your ISP's DNS server) how long to CACHE a DNS answer before asking again. A high TTL (e.g., 24 hours) reduces load on your DNS servers and speeds up repeat lookups, but means changes propagate slowly (people may see the old answer for up to that TTL). A low TTL (e.g., 60 seconds) lets you change records quickly (useful before a migration or failover event) but increases query volume. Best practice: lower your TTL BEFORE a planned change, then raise it back afterward.

## Troubleshooting Tips
- **"I updated a DNS record but nothing changed"**: It's almost always **TTL caching** — either at the resolver, ISP, browser, or OS level. Use `dig` or `nslookup` directly against Route 53's name servers to bypass cache and confirm the record itself is correct, then just wait out the TTL elsewhere.
- **"My domain isn't resolving at all"**: Check that the domain's registrar (GoDaddy, Route 53 registrar, etc.) actually points its **NS records** to Route 53's assigned name servers — a very common misconfiguration when a domain is registered elsewhere but DNS is meant to be managed in Route 53.
- **"Failover isn't happening"**: Confirm the health check is actually associated with the record, check the health check's own status (not just the endpoint), and verify the health checker's source IPs aren't being blocked by your Security Group/firewall (Route 53 health checkers come from specific AWS IP ranges that must be allowed).

---

# 6. Route 53 Resolvers

## The Mental Model
Standard Route 53 (above) answers DNS queries for the **public internet** for domains you host. **Route 53 Resolver** is different — it's the DNS resolution service that's automatically built into every VPC (it lives at the `.2` address of your VPC's CIDR, e.g., `10.0.0.2`), and by default it resolves both public domain names AND private Route 53 hosted zone names for resources INSIDE your VPC.

The interesting/advanced part is **Resolver Endpoints**, used for **hybrid DNS** — connecting your VPC's DNS resolution with an on-premises network:
- **Inbound Resolver Endpoint**: lets on-premises servers send DNS queries INTO your VPC (e.g., "resolve this private AWS resource name for me").
- **Outbound Resolver Endpoint**: lets your VPC forward specific DNS queries OUT to your on-premises DNS servers (e.g., "resolve this internal corporate domain name for me").
And **Resolver Rules** define which domains get forwarded where.

## Why It Exists
In real-world hybrid environments (part infrastructure on AWS, part in a corporate data center), you need DNS resolution to work seamlessly in both directions — someone in AWS needs to resolve `internal-app.corp.local` sitting on-prem, and someone on-prem needs to resolve `myapp.internal.aws` sitting in a VPC. Route 53 Resolver + endpoints + rules make that possible.

## Interview Questions & Answers

**Q1: What is the default DNS resolver in every VPC, and what does it resolve?**
A: Every VPC automatically gets a Route 53 Resolver reachable at the base of the VPC's CIDR range plus two (e.g., `10.0.0.2` for a `10.0.0.0/16` VPC). By default it resolves public internet domain names as well as any private hosted zones associated with that VPC, and internal AWS service endpoints.

**Q2: What's the difference between an Inbound and Outbound Resolver Endpoint?**
A: An **Inbound Endpoint** allows DNS queries to flow FROM outside the VPC (like on-premises servers) INTO the VPC, so external systems can resolve names hosted in your private Route 53 zones. An **Outbound Endpoint** allows DNS queries to flow FROM your VPC OUT to external DNS servers (like an on-prem Active Directory DNS server), via Resolver Rules that specify which domains get forwarded where.

**Q3: What is a Resolver Rule?**
A: A Resolver Rule tells the Outbound Resolver Endpoint: "for queries matching this specific domain (e.g., `corp.local`), forward them to these specific DNS server IPs" instead of resolving them normally. This is how hybrid DNS resolution across AWS and on-premises networks is achieved.

**Q4: Why would a company need Route 53 Resolver endpoints instead of just using Route 53 public/private hosted zones alone?**
A: Hosted zones alone only work within AWS's DNS system. If a company has an on-premises data center with its OWN existing DNS infrastructure (e.g., Active Directory DNS) that needs to resolve AWS-hosted private domain names, or if AWS resources need to resolve on-premises internal domain names, you need a bridge between the two DNS systems — that bridge is the Resolver endpoints + rules, working over a VPN or Direct Connect connection.

## Troubleshooting Tips
- **"On-prem servers can't resolve my private AWS hosted zone"**: Check an Inbound Resolver Endpoint exists, its Security Group allows DNS (port 53 TCP/UDP) from the on-prem network's IP range, and that the on-prem DNS server is configured to forward the relevant domain queries to the Inbound Endpoint's IPs.
- **"AWS resources can't resolve on-prem internal domains"**: Check the Outbound Resolver Endpoint exists, a Resolver Rule exists for that specific domain, the rule is associated with the correct VPC, and network connectivity (VPN/Direct Connect) between the VPC and on-prem DNS servers is healthy.
- Use **VPC DNS query logging** to see exactly what's being queried and where it's failing.

---

# 7. ACM (AWS Certificate Manager)

## The Mental Model
When you visit `https://` websites, that little padlock icon means your connection is encrypted using **TLS/SSL**, which relies on a **digital certificate** to prove the website is who it says it is and to enable encryption. Getting and renewing these certificates manually is a pain — they expire (usually yearly) and need to be reissued.

**ACM** is AWS's service that **issues, manages, and automatically renews** these TLS/SSL certificates for you, for free, when used with integrated AWS services like CloudFront, Elastic Load Balancer, and API Gateway.

## Why It Exists
Before ACM, you'd buy a certificate from a third party, manually install it, and remember to renew it before expiry (a common cause of production outages when forgotten!). ACM automates the entire lifecycle for AWS-integrated resources.

## Interview Questions & Answers

**Q1: What is ACM and what problem does it solve?**
A: ACM (AWS Certificate Manager) provisions, manages, and auto-renews SSL/TLS certificates used to enable HTTPS. It solves the pain of manually purchasing, installing, and renewing certificates — a process that used to require manual tracking and was a common cause of outages when certificates silently expired.

**Q2: How does ACM validate that you own a domain before issuing a certificate?**
A: Two methods: **DNS validation** (ACM gives you a special CNAME record to add to your domain's DNS; once ACM sees it, ownership is proven — and if this record stays in place, renewal happens automatically forever) or **Email validation** (ACM emails addresses like admin@yourdomain.com with an approval link — but this requires manual action on every renewal). DNS validation is strongly preferred because it enables fully automatic renewal.

**Q3: Can you use an ACM certificate on an EC2 instance directly?**
A: No — this is a very common trick question. ACM certificates can only be used with **integrated AWS services**: CloudFront, Elastic Load Balancers (ALB/NLB/CLB), and API Gateway. You cannot export the private key or install an ACM cert directly onto an EC2 instance's web server (e.g., Apache/Nginx) — for that you'd need a certificate from a different source, or you'd terminate TLS at a Load Balancer in front of the EC2 instance instead.

**Q4: Why do CloudFront certificates need to be requested in the us-east-1 region specifically?**
A: This is an AWS-specific quirk asked often in interviews: CloudFront is a global service, but ACM certificates used by CloudFront **must be requested in the `us-east-1` (N. Virginia) region**, regardless of where your other resources live. Certificates for regional services like an ALB must be in the SAME region as that ALB.

**Q5: Is ACM free?**
A: Yes, when the certificate is used with supported AWS integrated services (CloudFront, ELB, API Gateway) — there's no charge for the certificate itself, you only pay for the underlying AWS resources using it. (Private CA for internal certificate issuance is a separate paid feature.)

## Troubleshooting Tips
- **"My certificate is stuck in 'Pending Validation'"**: Check that the CNAME validation record was actually added to your DNS exactly as ACM specified (common mistake: adding an extra domain suffix, since Route 53 sometimes auto-appends the zone name).
- **"CloudFront won't let me select my certificate"**: 99% of the time, the cert was requested in the wrong region — it must be in `us-east-1` for CloudFront.
- **"Certificate renewed but the site still shows the old/expired cert"**: For DNS-validated certs tied to CloudFront/ALB, renewal is automatic, but check if there's a caching layer (browser cache, CDN edge cache) still serving old handshake info, or if the resource wasn't correctly re-associated with the renewed certificate ARN.

---

# 8. CloudFront

## The Mental Model
CloudFront is AWS's **Content Delivery Network (CDN)**. Imagine your website's server is in Virginia, USA, but you have a user in Mumbai — every request has to travel halfway around the world, which is slow. CloudFront solves this by caching copies of your content at **Edge Locations** — data centers spread across the globe, much closer to end users. When the Mumbai user requests your content, CloudFront serves it from a nearby edge location instead of the far-away origin server, making it much faster.

CloudFront sits in front of an **Origin** (which could be S3, an EC2/ALB-based application, or any HTTP server) and:
1. Caches content close to users (huge speed boost for static content).
2. Provides HTTPS termination easily (paired with ACM).
3. Can protect against DDoS attacks and integrate with AWS WAF.
4. Can serve dynamic content too (not just static) by passing requests through to the origin when needed.

## Why It Exists
Without a CDN, every user request travels all the way to your origin server, which is slow for distant users, and puts heavy repeated load on your origin for popular content. CloudFront improves performance, reduces origin load, and adds a security layer.

## Interview Questions & Answers

**Q1: What is CloudFront and why would you put it in front of your application?**
A: CloudFront is AWS's CDN — it caches and delivers content from edge locations physically closer to end users, dramatically reducing latency, offloading traffic from your origin server, and providing an easy way to add HTTPS, DDoS protection (via AWS Shield), and WAF-based filtering right at the edge.

**Q2: What is the difference between an Edge Location and a Region?**
A: A Region is a full AWS data center cluster where you can run compute, storage, databases, etc. An Edge Location is a much smaller, more numerous location (hundreds worldwide, vs. dozens of regions) used specifically for caching CloudFront content and running Lambda@Edge — it doesn't run general-purpose AWS services.

**Q3: What is a "cache hit" vs a "cache miss," and how does TTL affect caching?**
A: A cache hit means CloudFront already has a valid copy of the requested content at the edge and serves it immediately without contacting the origin — fast. A cache miss means the content isn't cached (or has expired) at that edge location, so CloudFront must fetch it from the origin first, cache it, then serve it — slower, but only on the FIRST request per edge location. TTL (Time To Live) controls how long an object stays cached before CloudFront re-checks with the origin; you set this via cache-control headers from the origin or CloudFront cache policies.

**Q4: How do you serve both static (cacheable) and dynamic (non-cacheable) content through the same CloudFront distribution?**
A: Using **multiple cache behaviors** based on URL path patterns. For example, `/images/*` and `/css/*` might use a caching policy with long TTLs, while `/api/*` uses a policy with caching disabled (or very short TTL) and forwards headers/cookies/query strings needed for dynamic responses straight to the origin.

**Q5: What is Origin Access Control (OAC), and why is it important when the origin is S3?**
A: If an S3 bucket is public, anyone can bypass CloudFront and hit S3 directly, defeating the purpose of caching/security controls. OAC (the modern replacement for the older Origin Access Identity/OAI) lets you keep the S3 bucket **fully private**, and only allow CloudFront (using a specific signed identity) to fetch objects from it — forcing all traffic through CloudFront where you control caching, HTTPS, and access rules.

**Q6: What is Lambda@Edge (or CloudFront Functions), and when would you use it?**
A: Both let you run small pieces of code at CloudFront edge locations to customize requests/responses in transit (e.g., URL rewrites, header manipulation, A/B testing, authentication checks) without a full round trip to the origin. **CloudFront Functions** are lighter-weight, cheaper, and faster (for simple, sub-millisecond JavaScript logic like header/URL manipulation), while **Lambda@Edge** supports heavier logic, longer execution time, and Node.js/Python with more capabilities, at higher cost/latency. Choose CloudFront Functions for simple logic, Lambda@Edge for complex logic.

**Q7: How does CloudFront handle HTTPS between the viewer and CloudFront, versus between CloudFront and the origin?**
A: These are configured independently. "Viewer Protocol Policy" controls whether end users must use HTTPS to reach CloudFront (often "Redirect HTTP to HTTPS"). "Origin Protocol Policy" controls whether CloudFront uses HTTP or HTTPS when it fetches content FROM your origin server. You can have HTTPS on the user-facing side while the origin connection uses plain HTTP (though HTTPS end-to-end is best practice).

## Troubleshooting Tips
- **"I updated my website but CloudFront is still showing old content"**: Classic caching issue — you need to **invalidate the cache** (via console/CLI, `aws cloudfront create-invalidation`) or wait out the TTL, or better, use versioned file names (e.g., `app.v2.js`) to avoid needing invalidations at all.
- **"Getting 403 Forbidden from CloudFront with an S3 origin"**: Almost always an Origin Access Control / bucket policy misconfiguration — the S3 bucket policy must explicitly allow the CloudFront distribution's OAC to `s3:GetObject`.
- **"HTTPS works on CloudFront but the app shows mixed-content warnings"**: Check if the origin is serving some resources over plain HTTP — the Origin Protocol Policy or the app's own hardcoded URLs might need updating to HTTPS.
- Check the **X-Cache header** in the response (`Hit from cloudfront` vs `Miss from cloudfront`) to instantly diagnose whether a slow response was a cache miss or a genuinely slow origin.

---

# 9. API Gateway

## The Mental Model
API Gateway is the **front door / receptionist** for your backend APIs. Instead of exposing your Lambda functions, EC2 servers, or other backend services directly to the internet, you put API Gateway in front. It receives HTTP requests from clients (web apps, mobile apps, third parties), and routes them to the correct backend, while also handling cross-cutting concerns: authentication, throttling/rate limiting, request validation, response transformation, caching, and logging — all without you writing that boilerplate code yourself.

Three types of APIs it offers:
- **REST API**: Full-featured, supports request/response transformation, API keys, usage plans, more granular control — but more expensive and slightly higher latency.
- **HTTP API**: Newer, simpler, cheaper, lower latency — a leaner subset of features, good for straightforward proxy-to-Lambda use cases.
- **WebSocket API**: For persistent, two-way real-time connections (e.g., chat apps, live dashboards).

## Why It Exists
Building authentication, rate limiting, request validation, and routing logic yourself into every backend service is repetitive and error-prone. API Gateway centralizes these concerns as a managed layer in front of your actual business logic.

## Interview Questions & Answers

**Q1: What is API Gateway and why not just expose your Lambda function or EC2 server directly?**
A: API Gateway acts as a managed front door that handles routing, authentication/authorization, throttling, input validation, caching, and monitoring for your APIs — so your backend (Lambda, EC2, or any HTTP endpoint) doesn't need to reimplement all of that. It also decouples your API's public contract from your backend implementation, letting you change the backend without changing the client-facing API.

**Q2: What's the difference between REST API and HTTP API in API Gateway?**
A: HTTP API is a newer, lighter-weight, cheaper, and lower-latency option built for simple proxy integrations (e.g., API Gateway → Lambda) but has fewer built-in features (no built-in API keys/usage plans, more limited request/response transformation). REST API is the original, full-featured option supporting request validation, API keys and usage plans, WAF integration, private APIs via VPC endpoints, and detailed response mapping — at higher cost. Choose HTTP API for simple, cost-sensitive use cases; REST API when you need its advanced feature set.

**Q3: What is a Lambda Proxy Integration versus a Lambda Custom (non-proxy) Integration?**
A: In **Proxy Integration**, API Gateway passes the ENTIRE raw request (headers, query params, body, etc.) directly to Lambda as-is, and expects Lambda to return a specifically-structured response (with statusCode, headers, body) — Lambda has full control. In **Custom Integration**, API Gateway performs mapping/transformation of the request BEFORE it reaches Lambda, and transforms Lambda's response before returning it to the client — giving API Gateway more control but requiring you to configure mapping templates (using Velocity Template Language), which is more complex. Most modern applications use Proxy Integration for simplicity.

**Q4: How does throttling work in API Gateway?**
A: API Gateway lets you set rate limits (steady-state requests per second) and burst limits (using a token bucket algorithm) at the account level, API stage level, or even per-API-key via Usage Plans. If a client exceeds the limit, API Gateway returns a `429 Too Many Requests` response — this protects your backend from being overwhelmed and can be used to enforce fair usage or tiered service plans between customers.

**Q5: What are API Gateway "stages," and why are they useful?**
A: A Stage represents a snapshot/deployment of your API configuration tied to a name — like `dev`, `staging`, `prod`. Each stage can have its own configuration (throttling limits, caching, logging level, stage variables for environment-specific values like database URLs) and its own invoke URL, letting you test changes safely before promoting to production.

**Q6: What authentication/authorization options does API Gateway support?**
A: 
- **IAM authorization**: caller must sign requests with AWS credentials (good for internal service-to-service calls).
- **Lambda authorizers (custom authorizers)**: a Lambda function you write that validates a token (e.g., a JWT) and returns an IAM policy deciding allow/deny — fully custom logic.
- **Cognito User Pool authorizers**: integrates directly with AWS Cognito for user sign-up/login flows, validating JWT tokens issued by Cognito.
- **API Keys**: simple keys tied to Usage Plans, mainly for tracking/throttling usage per client, NOT meant as strong security (should be combined with another method for real security).

**Q7: What is caching in API Gateway, and what are the risks of misusing it?**
A: API Gateway can cache responses at the stage level for a configurable TTL, reducing calls to your backend for frequently-requested, rarely-changing data. Risk: if you cache endpoints that return user-specific or rapidly-changing data without properly varying the cache key (e.g., by including auth tokens or query params in the cache key), you can accidentally leak one user's cached data to another user, or serve stale data. Always cache thoughtfully based on the actual variability of the response.

## Troubleshooting Tips
- **"Getting 403 Forbidden"**: Check the authorizer configuration (IAM policy, Lambda authorizer's returned policy, or Cognito token) — 403 usually means the request was authenticated but not authorized, whereas 401 usually means it wasn't authenticated at all.
- **"Lambda works when tested directly, but API Gateway returns a 502 Bad Gateway"**: This classic error means Lambda's response wasn't in the exact format API Gateway expects for Proxy Integration (must include `statusCode`, `body` as a string, and optionally `headers`) — check your Lambda's return statement structure.
- **"Changes aren't showing up after I updated my API"**: You likely forgot to **redeploy the stage** — editing resources/methods in API Gateway doesn't automatically push to a live stage; you must explicitly deploy.
- **"Getting CORS errors in the browser console"**: You need to enable CORS on the API Gateway resource (which auto-generates the required OPTIONS method and headers), AND your backend/Lambda's actual response also needs to include the right `Access-Control-Allow-Origin` headers — enabling CORS in the console alone doesn't fix Lambda's own response headers in Proxy Integration mode.

---

# 10. Athena

## The Mental Model
Athena is a **serverless query service** that lets you run standard SQL queries directly against data sitting in S3 — without needing to load it into a database first. Imagine you have millions of log files or CSV/Parquet files sitting in an S3 bucket; instead of setting up a database server, importing all that data, and then querying it, Athena lets you point SQL directly at the S3 files and get answers.

Under the hood, Athena uses **Presto/Trino** (a distributed SQL query engine) and relies on the **AWS Glue Data Catalog** to know the "schema" (table structure — column names/types) of your S3 data.

## Why It Exists
Loading huge datasets into a traditional database just to run occasional analytical queries is slow, expensive, and often unnecessary. Athena lets you query data "in place" in S3, paying only for the amount of data scanned per query, with zero infrastructure to manage.

## Interview Questions & Answers

**Q1: What is Athena and how is it different from a traditional database?**
A: Athena is a serverless, interactive query service that runs standard SQL directly against data stored in S3, with no need to load, transform, or provision any database infrastructure beforehand. Unlike a traditional database (like RDS or a data warehouse) where data is loaded and stored/indexed in the database engine itself, Athena queries the raw files in S3 on-demand, and you pay per query based on data scanned, not for a running server.

**Q2: How does Athena know the structure (schema) of files in S3?**
A: Through the **AWS Glue Data Catalog** (or the older Athena-native catalog) — you define a "table" that maps to a location in S3 and specifies column names, types, and the file format (CSV, JSON, Parquet, ORC, etc.). This can be done manually, or automatically via an **AWS Glue Crawler** that scans the data and infers the schema for you.

**Q3: How is Athena billed, and what does that mean for how you should design your data?**
A: Athena charges based on the **amount of data scanned per query** (a per-TB rate), regardless of how much data is actually returned. This has huge design implications: if your data is a giant CSV file, every query scans everything. If you convert to a compressed, columnar format like **Parquet**, and **partition** your data (e.g., by date, splitting into separate S3 "folders"), Athena can skip scanning irrelevant partitions/columns entirely, cutting costs dramatically — often 90%+ savings.

**Q4: What is partitioning in Athena, and why does it matter for performance and cost?**
A: Partitioning organizes your S3 data into a folder structure based on a column's value, e.g., `s3://bucket/logs/year=2026/month=07/day=06/`. When you query with a `WHERE year=2026 AND month=07` filter, Athena can skip scanning all other folders entirely (this is called "partition pruning"), drastically reducing the data scanned — meaning both faster queries AND lower cost, since Athena bills per byte scanned.

**Q5: What's the difference between row-based formats (CSV/JSON) and columnar formats (Parquet/ORC), and why does Athena perform better with the latter?**
A: In a row-based format, each row's full data is stored together, so a query that only needs 2 out of 20 columns still has to scan all 20 columns' worth of data per row. In a columnar format like Parquet, data is stored column-by-column, so a query needing only 2 columns only reads those 2 columns' data across all rows, skipping the rest — much less data scanned, which is both faster and (since Athena bills per byte scanned) cheaper. Columnar formats are also typically more compressible.

**Q6: Where do Athena query results get stored?**
A: Athena writes query results to an S3 location you configure (often called the "query result location"). This means you're also billed standard S3 storage costs for accumulated result files, and it's common practice to set up S3 lifecycle rules to auto-delete old query results.

## Troubleshooting Tips
- **"My Athena query is very slow / expensive"**: Check if data is compressed and columnar (Parquet/ORC) rather than raw CSV/JSON, and whether partitioning is being used and actually leveraged in the `WHERE` clause (using functions on partition columns, like `WHERE YEAR(date_col)=2026` instead of `WHERE year_partition=2026`, can defeat partition pruning).
- **"Query fails with 'HIVE_PARTITION_SCHEMA_MISMATCH' or schema errors"**: Usually means the Glue Data Catalog's defined schema doesn't match the actual file structure/types in S3 — often fixed by re-running a Glue Crawler or manually correcting the table DDL.
- **"New data added to S3 isn't showing up in query results"**: If using partitions, you may need to run `MSCK REPAIR TABLE` (for Hive-style partitions) or explicitly add new partitions, since Athena doesn't automatically detect new S3 "folders" unless told to refresh the catalog.

---

# 11. ECS (Elastic Container Service)

## The Mental Model
Containers (like Docker containers) package an application with everything it needs to run, so it behaves the same everywhere. But running containers at scale — deciding which physical servers to run them on, restarting them if they crash, scaling them up/down — is complex. **ECS** is AWS's service for **orchestrating containers**: it decides where your containers run, keeps the desired number running, and integrates with load balancers, logging, auto-scaling, etc.

Key concepts:
- **Task Definition**: a blueprint (JSON) describing your container(s) — image, CPU/memory, ports, environment variables. Think of it as a "recipe."
- **Task**: one running instance of a Task Definition — the actual running container(s).
- **Service**: keeps a specified number of Tasks running continuously, replacing any that fail, and can attach to a Load Balancer.
- **Cluster**: a logical grouping of the compute resources where your tasks/services run.
- **Launch Types**: 
  - **EC2 launch type**: you manage the underlying EC2 instances that host your containers (more control, more management overhead).
  - **Fargate launch type**: serverless — AWS manages the underlying compute entirely; you just specify CPU/memory needs per task (less management, generally higher per-unit cost, but no server patching/scaling to worry about).

## Why It Exists
Manually managing where containers run, restarting failed ones, and scaling them is operationally heavy. ECS automates this so you can focus on your application, not the orchestration plumbing.

## Interview Questions & Answers

**Q1: What is ECS, and what problem does it solve?**
A: ECS is a container orchestration service — it manages running, scaling, and healing Docker containers across a cluster of compute resources. It solves the operational burden of manually deciding container placement, restarting failed containers, integrating with load balancers, and scaling based on demand.

**Q2: What is the difference between a Task Definition, a Task, and a Service?**
A: A Task Definition is the blueprint/recipe (what container image, how much CPU/memory, which ports, env vars). A Task is one actual running instance created from that blueprint. A Service is a controller that ensures a specified number of Tasks are always running, replacing failed ones automatically, and can register Tasks with a Load Balancer's target group for traffic distribution.

**Q3: What's the difference between the EC2 launch type and Fargate launch type in ECS?**
A: With the **EC2 launch type**, you provision and manage the underlying EC2 instances (patching, scaling the instance fleet, choosing instance types) that host your containers — more control and potentially cheaper at large/predictable scale, but more operational overhead. With **Fargate**, AWS manages the underlying infrastructure entirely — you just declare how much CPU/memory each task needs, and AWS runs it on infrastructure you never see or manage — simpler operationally, but generally higher per-unit compute cost, and you lose some low-level control (e.g., can't SSH into the host).

**Q4: How does ECS Service Auto Scaling work?**
A: You attach a scaling policy (target tracking, step scaling, or scheduled) to an ECS Service, based on a metric like average CPU utilization or ALB request count per target. When the metric crosses your threshold, ECS automatically adjusts the desired count of running Tasks up or down within your configured min/max bounds — similar in concept to EC2 Auto Scaling Groups, but for container tasks instead of whole instances.

**Q5: How does ECS integrate with Load Balancers?**
A: An ECS Service can register its running Tasks as targets in an Application Load Balancer's (or Network Load Balancer's) Target Group. As Tasks start/stop (due to scaling, deployments, or failures), ECS automatically registers/deregisters them with the target group, and the ALB performs health checks and routes traffic only to healthy Tasks.

**Q6: What's the difference between ECS and EKS?**
A: ECS is AWS's own proprietary container orchestrator, simpler to learn and tightly integrated with AWS services. EKS (Elastic Kubernetes Service) is AWS's managed Kubernetes offering, using the open-source, industry-standard Kubernetes API — more powerful/flexible and portable across cloud providers, but with a steeper learning curve and more operational complexity. Choose ECS for AWS-native simplicity, EKS if you need Kubernetes-specific features, portability, or your team already knows Kubernetes.

**Q7: What is a "task role" versus a "task execution role" in ECS?**
A: The **Task Role** is the IAM role your APPLICATION CODE running inside the container assumes to call other AWS services (e.g., reading from S3, writing to DynamoDB). The **Task Execution Role** is used by the ECS AGENT itself (not your app) to do things like pull the container image from ECR and write logs to CloudWatch, BEFORE your application code even starts running. Confusing these two is an extremely common beginner mistake — e.g., giving S3 permissions to the execution role instead of the task role, so the app itself still can't access S3.

## Troubleshooting Tips
- **"My ECS task keeps stopping/restarting (crash loop)"**: Check the **Task's "Stopped reason"** in the console, and check CloudWatch Logs for the container's actual application error output — common causes are the container process exiting immediately (e.g., misconfigured startup command), failing health checks, or out-of-memory kills (check if the task's memory limit is too low).
- **"Task can't pull the container image"**: Usually the Task Execution Role is missing ECR pull permissions, or (for private subnets without a NAT Gateway/VPC endpoint) there's no network path to reach ECR.
- **"Load Balancer shows targets as unhealthy"**: Check the health check path/port configured on the target group actually matches what the container serves, and that the container's Security Group allows inbound traffic from the Load Balancer's Security Group on that port.
- **"Task role's application code gets 'Access Denied' calling S3/DynamoDB"**: Double check you attached permissions to the TASK ROLE, not the Task Execution Role (see Q7 above) — this is the #1 confusion point in real-world ECS troubleshooting.

---

# 12. Lambda

## The Mental Model
Lambda is **serverless compute** — you write a function (in Python, Node.js, Java, etc.), upload it, and AWS runs it **only when triggered** (by an API call, a file upload to S3, a scheduled time, a queue message, etc.), automatically scaling from zero to thousands of concurrent executions, and you pay only for the exact compute time used (measured in milliseconds), not for idle server time.

Think of it like hiring a contractor who only shows up and gets paid when there's actual work to do, versus a full-time employee (EC2 server) who you pay for 24/7 whether there's work or not.

Key concepts:
- **Trigger/Event Source**: what invokes the function (API Gateway, S3 event, EventBridge schedule, SQS, DynamoDB Streams, etc.)
- **Execution Role**: the IAM role the function assumes to access other AWS services.
- **Cold Start**: the delay when Lambda has to initialize a new execution environment (download code, start runtime) before running your function, versus a "warm" invocation reusing an already-initialized environment.
- **Concurrency**: how many instances of your function can run simultaneously; you can set Reserved Concurrency (guarantee/limit capacity) or Provisioned Concurrency (pre-warm environments to avoid cold starts).
- **Timeout**: max execution duration (up to 15 minutes) before AWS forcibly stops the function.

## Why It Exists
Running and managing servers 24/7 for workloads that are intermittent, event-driven, or spiky is wasteful and operationally heavy. Lambda removes server management entirely and aligns cost directly with actual usage.

## Interview Questions & Answers

**Q1: What is AWS Lambda and what are its core benefits?**
A: Lambda is a serverless compute service that runs your code in response to events, automatically managing all underlying infrastructure (provisioning, patching, scaling). Core benefits: no server management, automatic scaling (from zero to very high concurrency), pay-per-use billing (per millisecond of actual execution, not idle time), and fast integration with dozens of AWS event sources.

**Q2: What is a "cold start" and how can you reduce its impact?**
A: A cold start happens when Lambda needs to create a brand-new execution environment for your function — downloading your code, initializing the runtime, running any initialization code outside your handler — before it can process the actual request, adding noticeable latency (from tens of milliseconds to a few seconds depending on runtime/package size). To reduce impact: use **Provisioned Concurrency** (keeps a set number of environments pre-warmed), choose lighter-weight runtimes (e.g., avoid heavy JVM startup for latency-sensitive use cases), keep deployment packages small, and move expensive initialization code (like DB connections) outside the handler function so it's reused across warm invocations.

**Q3: What is the difference between Reserved Concurrency and Provisioned Concurrency?**
A: **Reserved Concurrency** sets both a guaranteed minimum AND a hard maximum number of concurrent executions for a specific function — useful to protect downstream systems from being overwhelmed, or to guarantee capacity is always available for a critical function (taken from the account's shared pool). **Provisioned Concurrency** pre-initializes a specified number of execution environments so they're already "warm" and ready, eliminating cold starts for that many concurrent invocations — used purely for latency, at an additional cost since you pay for the provisioned capacity even when idle.

**Q4: What is an Execution Role, and why does every Lambda function need one?**
A: The Execution Role is an IAM role that grants the Lambda function permission to interact with other AWS services (e.g., read from S3, write to DynamoDB, publish to SNS) and, at minimum, permission to write logs to CloudWatch. Every function needs one because Lambda functions have no permissions by default — following the principle of least privilege, you explicitly grant only what's needed.

**Q5: How does Lambda handle scaling, and what is a "throttle"?**
A: Lambda automatically creates new execution environments in response to concurrent incoming events, scaling up rapidly (with an initial burst capacity, then a steady increase rate per AWS account/region limits). If the number of concurrent executions would exceed your account's or function's concurrency limit, additional invocation attempts are **throttled** (rejected with a `TooManyRequestsException` for synchronous calls, or automatically retried/queued for asynchronous/event-source-mapping calls, depending on the trigger type).

**Q6: What's the difference between synchronous and asynchronous Lambda invocation, and why does it matter for error handling?**
A: **Synchronous invocation** (e.g., via API Gateway) means the caller waits for the function to finish and receives the response directly — errors must be handled by the caller in real time. **Asynchronous invocation** (e.g., via S3 event notifications, SNS) means the caller hands off the event and moves on; Lambda internally retries failed asynchronous invocations (typically twice more) and can route persistently failing events to a configured **Dead Letter Queue (DLQ)** or **On-Failure Destination** for later inspection — critical for building resilient event-driven pipelines where you don't want failures to be silently lost.

**Q7: Why would you put a Lambda function inside a VPC, and what's the tradeoff?**
A: You'd put Lambda in a VPC when it needs to access VPC-only resources, like an RDS database in a private subnet. The tradeoff historically was added cold-start latency due to ENI creation, though AWS has significantly improved this with the Hyperplane ENI architecture. Still, functions in a VPC needing internet access (e.g., calling a public API) require a NAT Gateway, since VPC-attached Lambdas lose default internet access unless explicitly routed.

**Q8: What is the Lambda execution timeout, and what happens if it's exceeded?**
A: You configure a maximum duration (up to 15 minutes) your function is allowed to run. If your code doesn't finish within that time, AWS forcibly terminates the execution and returns a timeout error. This is a common cause of failures for functions doing long-running tasks (e.g., processing large files) — the fix is usually to break the work into smaller chunks, use Step Functions to orchestrate multi-step long-running workflows, or move to a different compute option (ECS/Fargate) for genuinely long-running processes.

## Troubleshooting Tips
- **"Function times out intermittently"**: Check for cold starts combined with a tight timeout, slow downstream dependencies (e.g., a database under load), or missing connection reuse (recreating expensive clients on every invocation instead of outside the handler).
- **"Function works when tested manually but fails when triggered by S3/SQS/etc."**: Check the event source's actual payload structure — it's often different from what you assumed, and your code may be parsing the event object incorrectly. Log the raw `event` object first when debugging.
- **"Getting 'Access Denied' calling another AWS service"**: Check the Execution Role's attached policies — a very common and simple fix, always check this first.
- **"High Lambda costs"**: Check allocated memory — Lambda's CPU power scales with memory allocation, so sometimes RAISING memory (rather than lowering it) makes the function finish faster and paradoxically cost LESS overall since billing is duration × memory. Profile to find the memory "sweet spot."
- Always check **CloudWatch Logs** first (`/aws/lambda/<function-name>` log group) — Lambda automatically logs there, and it's the fastest way to see actual runtime errors and stack traces.

---

# 13. Step Functions

## The Mental Model
Step Functions lets you build **visual workflows** that coordinate multiple AWS services (especially Lambda functions) in a defined sequence, including branching logic, parallel execution, retries, and error handling — without writing all that orchestration/glue code yourself.

Analogy: if a single Lambda function is one worker doing one task, Step Functions is the **project manager** coordinating a whole team of workers (Lambda functions, ECS tasks, other AWS services) through a multi-step process, handling "what happens if this step fails," "wait until this condition is met," "do these three things in parallel," etc.

Workflows are defined using **Amazon States Language (ASL)**, a JSON-based format describing states like `Task`, `Choice`, `Parallel`, `Wait`, `Map`, `Succeed`, `Fail`.

Two workflow types:
- **Standard Workflows**: for long-running (up to 1 year), auditable, exactly-once execution workflows — you pay per state transition.
- **Express Workflows**: for high-volume, short-duration (up to 5 minutes), at-least-once execution workflows — cheaper for high-throughput event processing, you pay based on execution duration and number of executions rather than state transitions.

## Why It Exists
Coordinating multiple Lambda functions or services manually (with one Lambda calling another, tracking state, handling retries) becomes messy and hard to debug/visualize as workflows grow complex. Step Functions externalizes that coordination logic into a manageable, visual, and resilient state machine.

## Interview Questions & Answers

**Q1: What is Step Functions and what problem does it solve?**
A: Step Functions is a serverless orchestration service that lets you define multi-step workflows (state machines) coordinating AWS services like Lambda, ECS, SNS, and more — with built-in support for sequencing, branching, parallelism, retries, and error handling. It solves the problem of "orchestration code" (e.g., one Lambda invoking another and manually tracking state/retries) becoming tangled, hard to visualize, and hard to debug as workflows grow more complex.

**Q2: What is the difference between Standard and Express Workflows?**
A: **Standard Workflows** support long-running executions (up to 1 year), provide full execution history for auditing/debugging, and guarantee "exactly-once" execution — billed per state transition. **Express Workflows** are designed for high-volume, short-duration (up to 5 minutes) event-processing workloads, are cheaper at scale, but only guarantee "at-least-once" execution (meaning duplicate execution is possible, so your steps should be idempotent) and offer less detailed execution history by default. Choose Standard for critical, auditable, long-running business processes; Express for high-throughput streaming/event processing.

**Q3: What are some common "state types" in a Step Functions state machine?**
A: 
- **Task**: performs actual work, typically invoking a Lambda function or another AWS service.
- **Choice**: adds branching logic (if/else) based on the input data.
- **Parallel**: runs multiple branches of the workflow simultaneously.
- **Map**: iterates over a list of items, running the same set of steps for each item (can run concurrently) — great for batch processing.
- **Wait**: pauses the workflow for a specified time or until a specific timestamp.
- **Succeed / Fail**: terminal states marking the workflow's end result.

**Q4: How does Step Functions handle errors and retries?**
A: Each Task state can define a `Retry` block (specifying which errors to retry, how many times, and backoff intervals) and a `Catch` block (specifying what to do — e.g., transition to a cleanup/notification state — if retries are exhausted or a specific error occurs). This built-in resilience means you don't need to hand-write try/catch/retry logic inside every Lambda function for orchestration-level failures.

**Q5: What is the "Map" state used for, and give a real-world example?**
A: The Map state runs the same workflow steps for every item in an input array or list, either sequentially or concurrently up to a configurable concurrency limit. Real-world example: processing a batch of 1,000 uploaded images — a Map state can invoke a "resize image" Lambda function once per image, in parallel, without you writing a manual loop-and-invoke pattern yourself.

**Q6: How would you use Step Functions to avoid Lambda's 15-minute timeout limitation for a long-running process?**
A: Instead of trying to cram a long process into a single Lambda invocation (limited to 15 minutes), you break the process into multiple smaller Lambda functions/steps, each handling a portion of the work, and use Step Functions to orchestrate them in sequence (or use Wait states to poll a long-running external job, or Task Tokens for callback patterns where a Task pauses until an external process reports completion) — Standard Workflows can run for up to a year total, even though no single Lambda step exceeds 15 minutes.

## Troubleshooting Tips
- **"Workflow execution failed but I don't know where"**: Use the **visual execution graph** in the Step Functions console — it highlights exactly which state failed and shows the input/output/error at each step, far easier than digging through logs manually.
- **"Same execution seems to run steps twice"**: If using Express Workflows, remember they guarantee "at-least-once," not "exactly-once" — ensure your Task logic (e.g., the Lambda functions) is idempotent (safe to run more than once with the same input).
- **"Getting 'States.Timeout' errors"**: Check both the state-level `TimeoutSeconds` in your ASL definition and the underlying service's own timeout (e.g., the Lambda function's timeout) — a Step Functions state can time out even if the underlying Lambda hasn't, if you set an aggressive state-level timeout.
- **"IAM permission errors invoking a Task"**: The Step Functions state machine's own IAM role needs permission to invoke whatever resource the Task targets (e.g., `lambda:InvokeFunction`) — separate from any permissions the Lambda function itself has.

---

# 14. AWS Glue

## The Mental Model
Glue is AWS's **serverless data integration / ETL (Extract, Transform, Load) service**. Imagine you have messy data scattered across S3, RDS databases, and other sources, in different formats, and you need to clean it up, convert it, combine it, and load it somewhere usable (like a data warehouse, or back into S3 in a query-optimized format for Athena). Glue automates this process without you managing servers.

Key components:
- **Glue Data Catalog**: a central metadata repository (the "table of contents") describing where your data lives and what its schema looks like — used by Glue itself, and also by Athena, Redshift Spectrum, and EMR.
- **Glue Crawler**: a job that scans your data sources (like an S3 bucket) and automatically infers the schema, populating the Data Catalog with table definitions — so you don't need to manually define every column.
- **Glue Job**: the actual ETL script (auto-generated or custom, usually PySpark or Python) that transforms data — e.g., converting CSV to Parquet, joining multiple sources, filtering/cleaning records.
- **Glue Workflow / Trigger**: orchestrates multiple crawlers and jobs to run in sequence or on a schedule.

## Why It Exists
Building and maintaining your own ETL infrastructure (Spark clusters, scheduling, schema management) is heavy operational work. Glue provides this as a serverless, pay-per-use service, and its Data Catalog acts as the "single source of truth" for schema across multiple AWS analytics services.

## Interview Questions & Answers

**Q1: What is AWS Glue, and what are its main components?**
A: AWS Glue is a serverless data integration (ETL) service used to discover, prepare, and combine data from multiple sources for analytics, machine learning, or application development. Its main components are: the **Data Catalog** (central metadata store), **Crawlers** (auto-discover schema from data sources), **Jobs** (the actual data transformation scripts, typically Apache Spark-based), and **Workflows/Triggers** (orchestrate crawlers and jobs together, on a schedule or event basis).

**Q2: What does a Glue Crawler actually do?**
A: A Crawler connects to a data source (e.g., an S3 path, a JDBC database), scans sample data, infers the schema (column names, data types, file format, partition structure), and either creates or updates a table definition in the Glue Data Catalog — so you don't have to hand-write that metadata, and so tools like Athena can immediately query the data via SQL.

**Q3: How does Glue relate to Athena?**
A: Athena needs to know a table's schema and location to run SQL against S3 data — that schema/location info lives in the **Glue Data Catalog**. So Glue and Athena work as a pair: Glue Crawlers discover and catalog the data's structure, and Athena queries the actual data using that catalog's table definitions. (Also note: modern Athena workgroups actually use the Glue Data Catalog as their metastore by default.)

**Q4: What is the difference between a Glue Job and a Glue Crawler?**
A: A Crawler's only job is to DISCOVER schema and populate the Data Catalog — it doesn't move or transform actual data. A Glue Job actually PROCESSES data — reading from a source, applying transformations (cleaning, filtering, joining, format conversion), and writing the result to a destination. They're complementary: you might crawl raw data to catalog it, run a Job to transform/clean it into a new S3 location, then crawl that new location too so it's queryable.

**Q5: What is a DynamicFrame in Glue, and how is it different from a Spark DataFrame?**
A: A DynamicFrame is Glue's own data abstraction, similar to a Spark DataFrame but designed to handle **semi-structured, messy, or evolving schema data** more gracefully (e.g., where a field might sometimes be a string and sometimes a number across different records) without failing outright — it tracks schema more flexibly. Glue lets you convert between DynamicFrames and standard Spark DataFrames as needed, using the flexibility of DynamicFrames for cleanup and the raw performance/full feature set of Spark DataFrames for heavier transformations.

**Q6: What are Glue's pricing/billing basics, and why does it matter for job design?**
A: Glue Jobs are billed based on **Data Processing Units (DPUs)** consumed per second/hour the job runs (essentially, how much compute you allocate multiplied by how long the job runs). This means poorly optimized jobs (e.g., unnecessarily reading/scanning way more data than needed, inefficient joins, not partitioning output data) directly cost more money and take longer — same principle as Athena, efficient data formats (Parquet) and partitioning matter a lot.

**Q7: What is a Glue Workflow used for?**
A: A Workflow lets you chain together multiple Crawlers and Jobs into an orchestrated pipeline with dependencies (e.g., "Job A must finish before Crawler B starts, which must finish before Job C starts") and trigger conditions (on-demand, scheduled, or event-based) — essentially Glue's own lightweight orchestration layer for ETL-specific pipelines (as opposed to Step Functions, which is more general-purpose orchestration across any AWS service).

## Troubleshooting Tips
- **"Crawler ran but Athena still can't find the table / gets wrong results"**: Check that the crawler actually completed successfully (check its run history/logs), that it's pointed at the correct S3 path (including trailing slashes matter), and that the inferred schema (data types) actually matches reality — crawlers occasionally misinfer types from messy sample data, which you may need to manually correct in the Data Catalog afterward.
- **"Glue Job fails with out-of-memory or timeout errors"**: Usually means too much data is being processed by too few DPUs, or an inefficient transformation (e.g., a large join without partitioning/filtering first) — try increasing DPUs, filtering data earlier in the job, or converting source data to a more efficient columnar format first.
- **"Schema keeps changing every time the crawler runs, breaking downstream queries"**: This happens with genuinely inconsistent source data; consider configuring the crawler's schema update behavior (e.g., "add new columns only" vs. fully overwriting) to prevent constant schema churn, or clean/standardize data before it lands in the crawled location.

---

# Quick Cross-Topic "How Do These Fit Together" Questions

These are common in interviews to test whether you understand the AWS ecosystem holistically, not just isolated services.

**Q1: Walk me through how a request flows for a typical modern serverless web app using several of these services together.**
A: A user's browser resolves the domain via **Route 53**, which returns an Alias pointing to a **CloudFront** distribution (secured with an **ACM** certificate for HTTPS). CloudFront caches static assets at the edge and forwards dynamic/API requests to **API Gateway**. API Gateway authenticates the request and invokes a **Lambda** function (possibly orchestrated by **Step Functions** for multi-step business logic). That Lambda might run inside a **VPC** to reach a private database, using a **NAT Gateway** for any outbound internet calls it needs to make, all governed by **Route Tables** and Security Groups within the VPC. If there's an analytics component, raw event data might land in S3, get cataloged by **Glue Crawlers**, and be queried via **Athena** for reporting.

**Q2: If you have dozens of VPCs across multiple AWS accounts that all need to reach a shared set of on-premises resources, what would you use?**
A: A **Transit Gateway**, shared across accounts via AWS Resource Access Manager, acting as the central hub — each VPC attaches once, and a VPN or Direct Connect attachment from the TGW to on-premises provides the shared path, avoiding the need for a peering mesh or per-VPC VPN connections. **Route 53 Resolver** endpoints and rules would handle the hybrid DNS resolution piece so names resolve correctly in both directions.

**Q3: Why might you choose ECS with Fargate over plain Lambda for a workload?**
A: Lambda is ideal for short-lived (under 15 minutes), event-driven, bursty workloads. ECS/Fargate is better suited for long-running processes, workloads needing more control over the runtime environment (custom OS-level dependencies, larger persistent memory/CPU needs), or steady, predictable traffic where container-level control and lack of a hard timeout matter more than Lambda's finer-grained, event-triggered scaling.

---

*Tip for interviews: whenever you explain a service, always tie it back to "the problem it solves" and give a mental analogy — this signals real understanding rather than memorized definitions, and it's exactly what strong interviewers are listening for.*
