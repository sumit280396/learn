# AWS Scenario-Based & Troubleshooting Interview Guide
### From Zero Knowledge to Senior-Level Answers

**How to use this guide:** Every section opens with a "Mental Model" — the first-principles explanation of how the technology actually works. Read that first. The scenario questions then make sense on their own, because troubleshooting is just applying the mental model in reverse: something broke, so which assumption in the model was violated? Interviewers at the senior level don't test definitions; they test whether you can walk a packet, a request, or a permission-check through the system and find where it dies.

---

# PART 1 — VPC, Subnets, Route Tables, NAT Gateway

## Mental Model: What a VPC actually is

A **VPC (Virtual Private Cloud)** is your own private slice of the AWS network — a logically isolated network where you decide the IP address range, how it's carved up, and what can talk to what. Think of it as renting an empty office floor: AWS gives you the floor (the VPC), and you decide where the rooms go (subnets), where the doors are (gateways), and who has keys (security groups).

Key building blocks, in the order a packet meets them:

1. **CIDR block** — the IP range of the VPC, e.g. `10.0.0.0/16` (65,536 addresses). Everything inside the VPC gets an IP from this range.
2. **Subnets** — slices of the CIDR placed in a specific Availability Zone, e.g. `10.0.1.0/24`. A subnet is not "public" or "private" by nature — it becomes public or private purely based on its **route table**.
3. **Route tables** — the rulebook that says "traffic destined for X goes out via Y." Every subnet is associated with exactly one route table. Every route table has an implicit, undeletable `local` route for the VPC CIDR — that's why any two instances in the same VPC can reach each other by default (routing-wise).
4. **Internet Gateway (IGW)** — the VPC's door to the internet. Attached to the VPC, not to a subnet. A subnet is "public" if (and only if) its route table has `0.0.0.0/0 → igw-xxxx`.
5. **NAT Gateway** — lets instances in *private* subnets initiate outbound connections to the internet (for updates, pulling images, calling APIs) while blocking anything on the internet from initiating a connection *in*. It does this by translating the private source IP into its own Elastic (public) IP — that's the "Network Address Translation."
6. **Security Groups (SG)** — a stateful firewall attached to the network interface (ENI) of a resource. *Stateful* means: if inbound traffic is allowed, the response is automatically allowed out — you never need a matching outbound rule for replies. SGs only have **allow** rules; there's no deny.
7. **NACLs (Network ACLs)** — a *stateless* firewall at the subnet boundary. Stateless means return traffic must be *explicitly* allowed — this is why NACLs need ephemeral port ranges (1024–65535) opened for return traffic. NACLs support both allow and deny rules, evaluated in numbered order, first match wins.

**The golden troubleshooting sequence for "A can't reach B":**
Route table (is there even a path?) → Security Group on B (inbound allowed?) → Security Group on A (outbound allowed? — usually open by default) → NACLs on both subnets (including ephemeral return ports) → the OS firewall / is the service actually listening (`ss -tlnp`)?

---

## Q1. An EC2 instance in a public subnet has a public IP, but you can't SSH to it. Walk me through your troubleshooting.

**What the interviewer wants:** a layered, systematic elimination — not random guessing.

**Answer:**

"I troubleshoot from the outside in, following the packet's path.

1. **Is the subnet actually public?** Having a public IP is not enough. I check the subnet's route table for `0.0.0.0/0 → igw-xxxx`. A common mistake is launching into a subnet that *looks* public (auto-assign public IP enabled) but whose route table has no IGW route. The instance has a public IP that leads nowhere — return traffic can't get back out.
2. **Security Group:** does the SG attached to the instance allow inbound TCP 22 from my source IP? SGs are default-deny inbound, so a missing rule is the #1 cause. I also verify I'm checking the SG attached to *this instance's ENI*, not a similarly named one.
3. **NACL:** the subnet's NACL must allow inbound TCP 22 *and* outbound TCP 1024–65535 (ephemeral ports) — because NACLs are stateless, my SSH client's reply traffic goes back on a random high port. The default NACL allows all, so this only bites when someone has customized it.
4. **Instance level:** is sshd running and listening (`ss -tlnp | grep 22`)? Is the OS firewall (firewalld/ufw/iptables) blocking it? I'd check via SSM Session Manager or the EC2 serial console since SSH is down.
5. **My side:** corporate network or Zscaler blocking outbound 22, wrong key, wrong username (ec2-user vs ubuntu).

Tools that shortcut this: **VPC Reachability Analyzer** proves whether a network path exists, and **VPC Flow Logs** show whether packets are arriving and being ACCEPTed or REJECTed. A REJECT in flow logs = SG/NACL. Packets arriving with ACCEPT but no response = OS-level problem."

---

## Q2. Instances in a private subnet can't download OS updates or pull Docker images. What do you check?

**Concept first:** private subnet = no IGW route. For outbound internet, it needs a **NAT Gateway** that itself lives in a *public* subnet.

**Answer:**

"This is the classic NAT path problem. The working chain is: private instance → private subnet's route table (`0.0.0.0/0 → nat-xxxx`) → NAT Gateway sitting in a **public** subnet → that public subnet's route table (`0.0.0.0/0 → igw`) → internet. I verify each link:

1. Private subnet's route table has a default route pointing to the NAT Gateway. If someone associated the wrong route table with the subnet, this breaks silently.
2. The NAT Gateway is in a **public** subnet and that subnet's route table points to the IGW. A NAT Gateway placed in a private subnet is a very common misconfiguration — it exists, it's 'available', but it has no path to the internet itself.
3. The NAT Gateway's state is `Available` and it has an Elastic IP.
4. Security group **outbound** rules on the instance allow 443/80 (default SGs allow all outbound, but hardened environments often restrict it).
5. NACLs allow the outbound 443/80 and the inbound ephemeral return ports.
6. If it's DNS failing rather than connectivity (`curl` to an IP works, to a hostname doesn't), that's a Route 53 Resolver / DHCP option set issue, not NAT.

One senior-level addition: if the traffic is only going to AWS services like S3 or ECR, the *better* fix is **VPC endpoints** (Gateway endpoint for S3, Interface endpoints for ECR) — traffic stays on the AWS network, no NAT data-processing charges, and no internet exposure at all. In EKS/container environments, NAT Gateway data-processing costs from image pulls are a well-known bill shock; S3+ECR endpoints eliminate most of it."

---

## Q3. Your NAT Gateway bill exploded this month. How do you find the cause and reduce it?

**Answer:**

"NAT Gateway charges two ways: hourly, and **per GB processed**. Bill explosions are almost always the per-GB component. My approach:

1. **Identify the traffic:** enable VPC Flow Logs on the NAT Gateway's ENI (or the whole VPC) and aggregate by source/destination. CloudWatch metric `BytesOutToDestination` on the NAT Gateway confirms the volume trend.
2. **The usual suspects:**
   - **Traffic to S3/ECR/DynamoDB going through NAT.** This is the #1 cause. S3 and DynamoDB Gateway endpoints are *free*; Interface endpoints for ECR/CloudWatch/SSM are cheap compared to NAT per-GB. Container clusters pulling images through NAT is the classic pattern.
   - **Cross-AZ NAT usage:** instances in AZ-a using a NAT Gateway in AZ-b adds cross-AZ data charges *and* creates an availability coupling. Best practice is one NAT Gateway per AZ, with each private subnet routing to the NAT in its own AZ.
   - Chatty applications: log shipping, metrics agents, or a misconfigured service polling an external API in a tight loop.
3. **Fixes in priority order:** add Gateway endpoints for S3/DynamoDB (free, ~5-minute change), add Interface endpoints for the AWS services showing up in flow logs, fix AZ alignment, then address the application behavior itself."

---

## Q4. Two EC2 instances in the same VPC but different subnets can't communicate. Route tables look fine. Now what?

**Answer:**

"Within a VPC, routing is essentially never the problem — the implicit `local` route covers the entire VPC CIDR and can't be deleted. So if 'route tables look fine,' I go straight to the filters:

1. **Security groups.** Inbound on the destination for the specific port and protocol. Detail that trips people up: if the rule references a *security group ID* as the source, the source instance must actually have that SG attached — referencing SGs is by membership, not by name similarity. Also, ping failing while the app works is normal if ICMP isn't allowed — I test the actual port with `nc -zv <ip> <port>`, not ping.
2. **NACLs on both subnets.** Different subnets can have different NACLs. Because NACLs are stateless, I need: outbound from subnet A on the destination port, inbound on subnet B on that port, outbound from B on ephemeral ports, inbound on A on ephemeral ports. Any one missing rule breaks it, and a low-numbered DENY beats a higher-numbered ALLOW.
3. **OS firewall and listening state** on the destination: `ss -tlnp` — is the service bound to `0.0.0.0` or only to `127.0.0.1`? An app bound to localhost is unreachable from anywhere else no matter how open the cloud config is. This one is extremely common.
4. **Verify with evidence:** VPC Flow Logs show REJECT (SG/NACL) vs ACCEPT-with-no-reply (OS/app level). Reachability Analyzer gives a definitive yes/no on the AWS-config path and names the blocking component."

---

## Q5. Explain the difference between a Security Group and a NACL, with a scenario where each is the right tool.

**Answer:**

"Four differences that matter:

| | Security Group | NACL |
|---|---|---|
| Attaches to | ENI (instance/resource) | Subnet |
| State | **Stateful** — replies auto-allowed | **Stateless** — replies need explicit rules |
| Rules | Allow only | Allow **and Deny** |
| Evaluation | All rules evaluated together | Numbered order, first match wins |

**SG is the right tool** for expressing application intent: 'the app tier accepts 8080 only from the load balancer's SG.' Referencing SGs by ID means it keeps working as instances scale in and out — you're describing *who*, not *which IPs*.

**NACL is the right tool** when you need a **deny** — SGs can't deny. Classic scenario: a specific IP range is attacking or scanning your subnet. You add a low-numbered DENY rule in the NACL for that CIDR, and it's blocked at the subnet edge before it even reaches any instance's SG evaluation. NACLs are also used as a compliance guardrail: 'nothing in this subnet may ever talk on port 23,' enforced regardless of what SGs individual teams create.

In practice: SGs do 95% of the work; NACLs are left at default-allow except for explicit deny requirements. Managing fine-grained rules in stateless NACLs is error-prone because of the ephemeral port problem."

---

## Q6. You need to connect two VPCs. Compare VPC Peering vs Transit Gateway — when do you choose which?

**Answer:**

"**VPC Peering** is a direct, one-to-one private connection between two VPCs. It's simple and has no per-GB processing charge (only standard data transfer), but it has two hard limitations:

1. **Non-transitive.** If A↔B and B↔C are peered, A cannot reach C through B. Full mesh of N VPCs needs N×(N−1)/2 peerings — 10 VPCs = 45 peering connections, each with route table entries on both sides. It becomes unmanageable fast.
2. **No overlapping CIDRs**, and no transitive access to things attached to the peer (their VPN, their IGW, their other peers).

**Transit Gateway (TGW)** is a regional cloud router — a hub. Every VPC, VPN, and Direct Connect attaches to the hub once, and the hub routes between them. Transitive by design: attach 50 VPCs, they can all reach each other (subject to TGW route tables, which let you segment — e.g., prod VPCs can't reach dev VPCs, but everything reaches the shared-services VPC).

**Decision rule:** two or three VPCs with a simple, stable relationship → peering (cheaper, lower latency, no hourly cost). Anything resembling an organization — many VPCs, hybrid connectivity via VPN/Direct Connect, need for centralized egress or inspection, multiple accounts — → Transit Gateway. TGW costs an hourly attachment fee plus per-GB processing, so you're paying for manageability.

Interview bonus: peering has a hard technical edge in latency-sensitive, high-volume paths because there's no middle hop; some architectures use TGW for general connectivity plus a direct peering for one heavy data path."

---

## Q7. VPC Flow Logs show `REJECT` for traffic you believe should be allowed. How do you read flow logs and pin the cause?

**Answer:**

"A flow log record gives me: source IP, destination IP, source port, destination port, protocol, packets, bytes, and the action (ACCEPT/REJECT). How I use it:

1. **REJECT means an SG or NACL dropped it.** The record tells me the direction: if the destination is my instance and it's REJECT, either the instance's SG inbound or the subnet NACL inbound blocked it.
2. **Distinguish SG from NACL with a pattern:** a subtle but powerful trick — if I see the *initial* packet ACCEPTed inbound but the *return* traffic REJECTed outbound, it must be the NACL, because a security group is stateful and would never reject return traffic for a connection it allowed in. Stateless NACL missing the ephemeral-port outbound rule produces exactly this signature.
3. **No record at all** for the expected traffic: the packet never reached this ENI — routing problem, wrong IP, or it's being blocked before the VPC (on-prem firewall, Zscaler, peered VPC's side).
4. **ACCEPT but the app still fails:** the network delivered the packet; the problem is above layer 4 — service not listening, TLS failure, app error. Move to the host: `ss -tlnp`, app logs, `curl -v`.

One gotcha worth naming: flow logs don't capture everything — DHCP, instance metadata (169.254.169.254), and Amazon DNS traffic to the VPC resolver are excluded, so 'I don't see DNS packets' doesn't mean DNS isn't happening."

---

## Q8. A team says "the VPC ran out of IP addresses." What happened and what are your options?

**Answer:**

"Each subnet loses 5 addresses to AWS (network, broadcast, VPC router, DNS, one reserved), and every ENI consumes an IP — in Kubernetes/EKS environments this is amplified massively because **every pod gets a VPC IP** via the VPC CNI. A busy EKS cluster can drain a /24 in no time. Symptoms: instances or pods fail to launch with 'insufficient free addresses in subnet,' EKS pods stuck in `ContainerCreating` with CNI IP-allocation errors.

Options, from least to most invasive:

1. **Add a secondary CIDR block to the VPC.** VPCs support additional CIDRs; you create new, larger subnets from the secondary range and point new workloads there. No downtime, the standard fix.
2. **For EKS specifically: custom networking / prefix delegation.** VPC CNI prefix delegation assigns /28 prefixes per ENI instead of individual IPs, dramatically increasing pods-per-node efficiency; custom networking can put pods in the secondary CIDR range (even 100.64.0.0/10 carrier-grade NAT space, which doesn't collide with corporate RFC1918 allocations).
3. **Audit for waste:** unattached ENIs, oversized subnets sitting idle, load balancers in subnets that are too small (ALBs need at least a /27 with 8 free IPs per subnet and grow under load).
4. **Long-term:** IP address management (IPAM) discipline — AWS has a VPC IPAM service for planning CIDR allocation across accounts so teams stop hand-picking overlapping or undersized ranges.

The thing you *can't* do is resize an existing subnet — subnet CIDRs are immutable. You add new ones."

---

## Q9. Traffic between an EC2 instance and an RDS database in the same VPC is intermittently slow. How do you approach a *performance* (not connectivity) network problem?

**Answer:**

"Intermittent slowness is about saturation or limits, not configuration. My checklist:

1. **Is it actually the network?** First separate network latency from database latency: run a quick TCP-level check (`ping`/`hping3` for RTT, or timestamped `nc` connects) versus query execution time in the DB's own metrics. If RTT is flat and low while queries are slow, this is an RDS problem (CPU, IOPS, locks) wearing a network costume.
2. **Cross-AZ?** If the instance and RDS are in different AZs, there's inherent ~0.5–1ms extra RTT plus cross-AZ data charges. For chatty workloads (many small queries), RTT multiplies: 1,000 sequential queries × 1ms = a whole extra second. Fix is same-AZ placement or reducing round trips (batching, connection pooling).
3. **Instance network limits.** Every EC2 instance size has a network bandwidth cap, and smaller instances have *burst* bandwidth — they can spend a credit-like allowance and then get throttled hard. CloudWatch/ethtool metrics `bw_in_allowance_exceeded`, `bw_out_allowance_exceeded`, `pps_allowance_exceeded`, and `conntrack_allowance_exceeded` (visible via `ethtool -S`) show micro-throttling that classic monitoring misses. `conntrack_allowance_exceeded` in particular causes mysterious dropped connections on busy instances.
4. **Connection churn.** Apps opening a new DB connection per request pay TCP+TLS handshake costs every time and can exhaust conntrack. Evidence: high rate of new flows in flow logs, RDS `DatabaseConnections` sawtoothing. Fix: connection pooling, or RDS Proxy.
5. **DNS in the path:** if the app resolves the RDS endpoint on every connection and something slows DNS (resolver throttling — the VPC resolver caps packets-per-second per ENI), that shows up as intermittent connect delays. Caching resolution or using a local cache fixes it."

---

## Q10. Explain what happens, step by step, when an instance in a private subnet makes a request to `https://api.example.com`. (The "walk me through it" question.)

**Answer:**

"I'll trace every hop — this is the question that reveals whether someone really understands VPC networking.

1. **DNS resolution.** The instance asks its configured resolver — by default the VPC's Route 53 Resolver at the VPC base +2 address (e.g., 10.0.0.2). The resolver checks: is this a private hosted zone associated with this VPC? A resolver rule forwarding this domain elsewhere? Otherwise it recursively resolves via public DNS and returns the public IP of api.example.com.
2. **Routing decision.** The instance's OS sends a packet to that public IP. The subnet's route table is consulted: the most specific matching route wins. The VPC `local` route doesn't match a public IP, so the default route `0.0.0.0/0 → nat-gateway` matches.
3. **Security group / NACL egress.** Instance SG outbound must allow 443 (stateful, so return is automatic); subnet NACL outbound 443 and inbound ephemeral must allow (stateless).
4. **NAT translation.** The NAT Gateway rewrites the source IP from the instance's private 10.x address to the NAT's Elastic IP, records the mapping in its translation table, and forwards the packet using *its own* subnet's route table — which sends `0.0.0.0/0` to the IGW.
5. **IGW.** The Internet Gateway does a final mapping and puts the packet on the internet. api.example.com sees the NAT's Elastic IP as the client.
6. **TCP + TLS.** Three-way handshake, then TLS handshake: the server presents its certificate chain, the instance validates it against its CA trust store, they negotiate keys, and the encrypted HTTP request flows.
7. **Return path.** Response arrives at the NAT's Elastic IP; NAT looks up its translation table, rewrites the destination back to the instance's private IP, and forwards it. Stateful SG lets it in automatically; NACL needs the ephemeral inbound rule.

If asked where this breaks: step 1 fails = DNS/resolver issue; step 2 = route table; 3 = SG/NACL; 4–5 = NAT in wrong subnet or no EIP; 6 = certificate/trust/proxy interception (Zscaler-style TLS inspection re-signing certs is a classic enterprise failure here)."


---

# PART 2 — Transit Gateway (TGW)

## Mental Model

A Transit Gateway is a **regional router that AWS runs for you**. You don't see its interfaces or its OS — you see *attachments* (VPCs, VPNs, Direct Connect gateways, peered TGWs in other regions) and *TGW route tables*.

Two directions of routing, and both must be correct:

- **Getting traffic TO the TGW:** the *VPC's* subnet route table needs a route like `10.0.0.0/8 → tgw-xxxx`. The TGW being attached does nothing by itself — no VPC route, no traffic.
- **Getting traffic THROUGH the TGW:** each attachment is *associated* with exactly one **TGW route table**, which decides where traffic arriving from that attachment can go. Attachments can also *propagate* their routes into TGW route tables (auto-populating them).

**Association vs Propagation — the concept everyone fumbles:**
- **Association** = which TGW route table is used for traffic *coming from* this attachment. (One per attachment. Think: "whose rulebook do I consult when this attachment sends me a packet?")
- **Propagation** = this attachment advertises its CIDRs *into* a TGW route table so others can find it. (Can propagate to many tables.)

Segmentation falls out naturally: prod VPC attachments associate with a "prod" TGW route table that only contains routes to other prod VPCs and shared services; dev attachments associate with a "dev" table. Same TGW, isolated networks.

**The two-hop rule of thumb for debugging:** every TGW flow needs (1) a VPC route pointing at the TGW on the source side, (2) a TGW route table entry pointing at the destination attachment, and (3) a *return path* — the destination VPC's route table pointing back at the TGW and the TGW routing back to the source attachment. One-way routes are the most common TGW bug: pings fail not because the request can't get there, but because the reply can't get back.

---

## Q11. VPC A and VPC B are both attached to a Transit Gateway, but instances can't communicate. Full walkthrough.

**Answer:**

"Six checkpoints, in packet order:

1. **Source VPC route table:** does the subnet's route table in VPC A have a route for VPC B's CIDR (or a supernet like 10.0.0.0/8) pointing to the TGW attachment? Attachment without a VPC route is checkpoint-one failure and the most common one.
2. **TGW attachment health & subnets:** both attachments in `available` state. Detail: a VPC attachment places an ENI in one subnet *per AZ you selected*. If VPC A's workload is in AZ-c but the attachment only covers AZ-a and AZ-b, traffic from AZ-c has no TGW ENI to use — 'TGW works for some instances but not others' is almost always this.
3. **TGW route table (forward):** the route table *associated with VPC A's attachment* must have a route to VPC B's CIDR pointing at VPC B's attachment — via propagation or a static route. If A and B are associated with different TGW route tables (deliberate segmentation), that's by design.
4. **TGW route table (return):** the table associated with *B's* attachment must route back to A. Asymmetric TGW tables → one-way traffic → connections time out after SYN.
5. **Destination VPC route table:** VPC B's subnet route table needs a route for VPC A's CIDR back to the TGW. People remember the request path and forget the reply path constantly.
6. **SG/NACL:** VPC B's instance SG must allow inbound from VPC A's CIDR — you can't reference an SG *ID* across a TGW (SG referencing works over peering in some cases, but not TGW), so rules must be CIDR-based. NACLs on both sides including the TGW attachment ENIs' subnets.

Fastest diagnostic: **Reachability Analyzer** handles TGW paths and names the exact failing component. Flow logs on both ends distinguish 'never arrived' from 'arrived and got rejected.'"

---

## Q12. Design centralized egress: 30 VPCs, all internet-bound traffic must go through one inspected egress point. How does it work and what's the routing trap?

**Answer:**

"**Architecture:** spoke VPCs have *no* IGWs or NAT Gateways. Each spoke's route table sends `0.0.0.0/0 → TGW`. The TGW route table for spokes sends `0.0.0.0/0` to a dedicated **egress VPC** attachment. The egress VPC contains NAT Gateways (and usually inspection appliances or AWS Network Firewall) in front of an IGW. Return traffic retraces the path.

**The routing trap — appliance-mode / asymmetry:** the egress VPC's route tables must send traffic *back* to the TGW for all spoke CIDRs (e.g., `10.0.0.0/8 → TGW`), and this return route goes on the subnets holding the NAT Gateways. Miss it, and outbound works while replies die in the egress VPC.

Second trap: if you run stateful inspection appliances across multiple AZs, you must enable **appliance mode** on the egress VPC attachment. Without it, TGW may send the forward flow through AZ-a's appliance and the return flow through AZ-b's appliance; a stateful firewall that never saw the SYN drops the reply. Appliance mode keeps a flow symmetric to one AZ.

Third consideration: spokes must not be able to route to each other through the egress table if isolation is required — that's controlled by *not* propagating spoke routes into the spoke-associated TGW route table, only the default route and shared services.

**Why do this at all:** one place for egress filtering/inspection/logging, ~28 fewer NAT Gateways' hourly cost, single audit point for compliance — at the price of TGW per-GB processing on all internet traffic and a critical shared dependency you must build multi-AZ."

---

## Q13. On-prem users connected via VPN/Direct Connect into the TGW can reach VPC A but not VPC B. Both VPCs are attached and "look identical." Where do you look?

**Answer:**

"'Reaches A but not B' through the same hybrid path means the shared path works — the difference is per-VPC. In likelihood order:

1. **TGW route table associated with the VPN/DX attachment:** does it contain B's CIDR? Maybe B's attachment propagates to a different TGW route table, or B was attached later and only static routes were used and nobody added B's.
2. **Propagation from B is missing** while A's exists — check the propagations tab on the TGW route table.
3. **B's VPC route table lacks the return route** to the on-prem CIDR via TGW. A's has it, B's doesn't. Classic 'we copied the setup but missed one route table' — especially when VPC B has multiple route tables and someone updated the wrong one.
4. **Route advertisement toward on-prem:** with BGP over VPN/DX, on-prem needs to *learn* B's CIDR. If A's CIDR is advertised but B's isn't (prefix filters on the customer gateway, or DX allowed-prefixes list not updated), on-prem routers send B-bound traffic elsewhere (often to the internet or a null route). I'd check the BGP received-routes on the on-prem side.
5. **Overlapping CIDR:** if B's CIDR overlaps with an on-prem range, on-prem routers prefer the local/more-specific route and traffic never enters the tunnel. This is the ugliest cause because the fix is re-IP or NAT.
6. Only after routing is proven: SG/NACL differences in B.

The BGP and allowed-prefixes checks are what separate senior answers — people who've only done VPC-to-VPC forget that hybrid routing has an *advertisement* dimension, not just tables."

---

## Q14. What's the difference between TGW peering and just attaching everything to one TGW? When do you need TGW peering?

**Answer:**

"A Transit Gateway is **regional**. One TGW serves all VPCs in its region (across accounts, via Resource Access Manager sharing). You need **TGW peering** when connecting *regions*: TGW in us-east-1 peers with TGW in eu-west-1 over the AWS backbone (encrypted, no public internet).

Key facts that come up:

- TGW peering routes are **static only** — no propagation across a peering. Every cross-region CIDR must be added by hand (or by IaC) to both TGWs' route tables. Forgetting the return-side static route is the standard cross-region failure.
- It's non-transitive at the TGW level in the sense that careful route design is needed for three-plus regions — you typically build a mesh of TGW peerings.
- Bandwidth and latency ride the AWS global backbone; data transfer is charged at inter-region rates.

So: one region → one TGW, no peering needed regardless of VPC count. Multi-region → TGW per region + peerings + static routes, all managed via Terraform because manual static route management across regions is error-prone."


---

# PART 3 — DNS, Route 53, and Route 53 Resolver

## Mental Model

**DNS is a distributed phone book with caching.** A client asks a *resolver* "what's the IP for app.example.com?" The resolver either answers from cache or walks the hierarchy: root servers → `.com` TLD servers → example.com's *authoritative* name servers, which hold the actual records. The answer is cached everywhere along the way for **TTL** seconds.

Records you must know:
- **A / AAAA** — name → IPv4 / IPv6 address.
- **CNAME** — name → *another name* (an alias that triggers a second lookup). Rule: a CNAME cannot coexist with other records at the same name, and you can't put a CNAME at a zone apex (example.com itself).
- **Alias (Route 53-specific)** — like a CNAME but resolved *inside* Route 53 to the target's current IPs, works at the apex, is free to query, and auto-tracks AWS resources (ALB, CloudFront) whose IPs change constantly. This is why you point example.com at an ALB with an Alias, never with hard-coded A records.
- **NS** — who is authoritative for a zone. **SOA** — zone metadata. **TXT** — verification/SPF. **MX** — mail.

**Route 53** is AWS's DNS service with three distinct jobs people conflate:
1. **Public hosted zones** — authoritative DNS for your domains on the internet.
2. **Private hosted zones (PHZ)** — DNS visible *only inside VPCs you associate*, e.g. `db.internal` resolving to a private IP.
3. **Routing policies** — Simple, Weighted (traffic splitting/canary), Latency-based, Failover (with health checks), Geolocation, Geoproximity, Multivalue. DNS-level traffic steering.

**Route 53 Resolver** is the DNS server that lives inside every VPC at `VPC-CIDR-base + 2` (10.0.0.0/16 → 10.0.0.2). Instances use it automatically via DHCP. It answers from: private hosted zones associated to the VPC → resolver rules → VPC internal names → public DNS. For **hybrid DNS**:
- **Inbound endpoints** = ENIs with IPs in your VPC that *on-prem* DNS servers can forward queries to ("on-prem wants to resolve AWS private names").
- **Outbound endpoints + forwarding rules** = the VPC resolver forwards queries for specific domains (e.g. `corp.internal`) *to on-prem* DNS servers ("AWS wants to resolve on-prem names").

**TTL discipline for cutovers:** DNS changes don't propagate on your schedule — they take effect as caches expire. Before any migration, lower the TTL (e.g., 300s → 60s) *at least one full old-TTL period in advance*, do the cutover, verify, then raise TTL back. Skipping this means some clients hit the old IP for hours.

---

## Q15. You changed a DNS record two hours ago; some users reach the new server, others still hit the old one. Explain why and what you should have done.

**Answer:**

"This is TTL and caching behaving exactly as designed. When the record had, say, TTL 3600, every resolver that looked it up cached it for up to an hour — and there are *layers* of caches: the user's OS, their router, their ISP's resolver, corporate DNS, sometimes the application's own connection pool holding open connections to the old IP. Each cache expires on its own clock starting from when *it* fetched the record, so the change rolls out raggedly over the TTL window. Some enterprise resolvers also ignore very low TTLs or clamp them upward, and some applications (JVMs are notorious — `networkaddress.cache.ttl` defaults to caching forever with a security manager) cache DNS internally beyond the TTL.

What should have happened — the standard cutover runbook:
1. **T-minus (at least one old-TTL before):** lower TTL to 60s. If TTL was 24h, do this a day or more ahead — the lowered TTL only takes effect after old cached copies expire.
2. **Cutover:** change the record. Now stragglers converge within ~60s.
3. **Keep the old target alive** and serving (or redirecting) during the overlap window — never decommission the old endpoint the moment you flip DNS.
4. **Verify from multiple vantage points:** `dig @8.8.8.8`, `dig @1.1.1.1`, and directly against the authoritative server (`dig @ns-xxx.awsdns-xx.com example.com` — the authoritative answer is the truth; anything different elsewhere is caching).
5. **Restore TTL** to the normal value once traffic has fully shifted.

For zero-drama migrations, better than a hard flip: **weighted routing** — 95/5, watch errors, 50/50, 0/100. DNS-level canary."

---

## Q16. An EC2 instance can resolve public domains but not `db.corp.internal` (an on-prem name). Another instance can't resolve your private hosted zone names. Diagnose both.

**Answer:**

"Two different failure paths through the VPC resolver's decision order.

**Case 1 — on-prem name fails:** the VPC resolver has no idea `corp.internal` exists on-prem unless a **forwarding rule** tells it. Checklist:
1. Is there a Route 53 Resolver **outbound endpoint** in this VPC (or a shared one reachable from it)?
2. Is there a **forwarding rule** for `corp.internal` targeting the on-prem DNS server IPs — and is that rule **associated with this VPC**? Rules exist independently; an unassociated rule does nothing. In multi-account setups the rule is usually shared via RAM and someone forgot to associate the new VPC.
3. Network path from the outbound endpoint ENIs to on-prem DNS: port 53 **UDP and TCP** through SGs, NACLs, TGW/VPN routing, and the on-prem firewall. TCP/53 matters — large responses fall back to TCP and 'UDP-only' rules cause maddening intermittent failures.
4. Do the on-prem DNS servers accept recursive queries from the endpoint IPs (ACLs on BIND/Windows DNS)?
Test precisely: `dig db.corp.internal` from the instance (uses VPC resolver) vs `dig db.corp.internal @<onprem-dns-ip>` (bypasses forwarding). If direct works but via-resolver doesn't → rule/association problem. If direct also fails → network path problem.

**Case 2 — private hosted zone fails:** PHZs only answer inside **associated VPCs**.
1. Is the PHZ associated with *this* VPC? (Most common miss, again especially cross-account, which requires an authorize-then-associate two-step.)
2. VPC settings `enableDnsSupport` and `enableDnsHostnames` must both be true — PHZ resolution silently fails without them.
3. Is the instance actually using the VPC resolver? If someone hardcoded `8.8.8.8` in resolv.conf or a custom DHCP option set points elsewhere, private zones are invisible — Google's DNS has never heard of your PHZ. `cat /etc/resolv.conf` should show the +2 address (or systemd-resolved chaining to it).
4. Overlapping zones: if a PHZ exists for `example.com`, the resolver answers *only* from the PHZ for anything under it — a public record like `www.example.com` that isn't duplicated in the PHZ becomes unresolvable inside the VPC. That's the 'private zone shadows public zone' trap."

---

## Q17. Design DNS failover between two regions with Route 53, and explain what health checks can and cannot see.

**Answer:**

"**Setup:** primary and secondary records with a **Failover routing policy** — both are, say, Alias records to the ALB in each region. The primary record carries a **health check**; when it fails, Route 53 starts answering with the secondary. TTL kept low (30–60s) so clients converge quickly on failover.

**How health checks work:** a fleet of Route 53 checkers around the world probes your endpoint (HTTP/HTTPS/TCP) every 30s (or 10s fast mode); the endpoint is healthy if a configurable percentage of checkers succeed. You can also do **calculated health checks** (combine children with AND/OR logic) and **CloudWatch-alarm-based checks** (mark unhealthy when an alarm fires — the only way to health-check *private* resources, since Route 53 checkers live on the public internet and cannot reach into a VPC).

**Blind spots — this is what interviewers dig for:**
1. Checkers probe from *outside*. 'Healthy' means the internet-facing endpoint returned 200 — the app could still be failing for real users (DB down but health path is static, for instance). Health check paths should exercise real dependencies (`/health` that touches the DB) — but not so deeply that a slow dependency flaps you.
2. **Clients cache DNS.** Failover changes the *answer*; clients holding cached records or long-lived connections keep hitting the dead region until TTL expiry — and some resolvers/apps ignore TTLs. DNS failover is minutes-scale, never seconds-scale. If you need faster, that's Global Accelerator (anycast IPs, no DNS dependency) or client-side retry logic.
3. **Failback flapping:** when the primary recovers, traffic snaps back automatically. If the primary is 'recovering' in a fragile state (cold caches, scaling from zero), the returning flood can knock it over again. Mitigations: require longer healthy-check streaks, or use a manual gate for failback.
4. Health checking the *ALB DNS name* vs *what users hit*: if the failure is between CloudFront and the ALB, or in DNS itself, an ALB-targeted health check stays green while users see errors. Check the path users actually take."

---

## Q18. `dig` returns the right IP but the application still can't connect / connects to the wrong place. What non-obvious DNS issues do you check?

**Answer:**

"When DNS 'works' in dig but the app disagrees, the app is not doing what dig does:

1. **The app uses a different resolver path.** dig queries `/etc/resolv.conf` nameservers directly; apps go through glibc's resolution order — `/etc/nsswitch.conf`, which consults **`/etc/hosts` first**. A stale `/etc/hosts` entry overrides DNS invisibly. Always `cat /etc/hosts` early.
2. **Container vs host DNS.** Inside Kubernetes, pods use CoreDNS, which has its own config, its own cache, and `ndots:5` search-domain behavior — a name like `db.internal` gets tried as `db.internal.namespace.svc.cluster.local` etc. first. Host-level dig proves nothing about pod-level resolution; test with a debug pod.
3. **Application-internal DNS caching.** JVM caching (potentially forever), connection pools pinning resolved IPs, HTTP clients with keep-alive to the old address. Restart or use documented cache-TTL settings.
4. **IPv6/IPv4 (AAAA vs A):** app resolves AAAA, tries IPv6, and the IPv6 path is broken/blocked — while your dig checked A. `dig AAAA` and check happy-eyeballs behavior.
5. **Search domain surprises:** `hostname` resolving via a search suffix to something unexpected. `dig +search` vs plain dig to compare.
6. **systemd-resolved stub** (127.0.0.53) with its own cache and per-interface DNS settings: `resolvectl status`, `resolvectl query <name>` show what the OS *actually* uses.
7. **Split-horizon mismatch:** the name resolves differently inside vs outside the VPC (private hosted zone vs public zone), and the app is resolving in a context you didn't test — e.g., a Lambda outside the VPC getting the public answer.

Method: reproduce with the *app's* resolution path — `getent hosts <name>` uses the same NSS path as most applications and is the honest test on Linux."

---

## Q19. Weighted routing is set 90/10 between old and new stacks, but the new stack is getting almost no traffic / wildly uneven traffic. Why can weighted DNS routing be inaccurate?

**Answer:**

"Weighted routing controls the ratio of DNS *answers*, not the ratio of *traffic* — and several things sit between those two:

1. **Resolver caching concentration:** one big ISP or corporate resolver serves millions of users; it asks once, caches the answer for the TTL, and everyone behind it goes to whichever side that single answer picked. Low client diversity → high variance from the target ratio. Lower TTL improves fidelity at the cost of query volume.
2. **Long-lived connections:** clients that got the 'old' answer keep reusing keep-alive connections or gRPC channels far beyond TTL; new-answer traffic only comes from *new* connections.
3. **Health checks:** if the new stack's record has a failing health check attached, Route 53 stops serving it entirely regardless of weight — always the first thing to check when one side gets ~0%.
4. **Weight 0 vs missing records** and record-set mistakes: the two records must have the same name and type in the same zone; a typo means one 'weight' isn't in the set at all.
5. Some resolvers/clients pin or re-sort answers unexpectedly.

If a precise split matters (real canary analysis), do the split at a layer that sees every request: **ALB weighted target groups** or CloudFront/Lambda-based routing gives exact request-level ratios. DNS weighting is a coarse instrument — good for gradual region shifts, wrong tool for a precise 1% canary."


---

# PART 4 — SSL/TLS, ACM, Certificate Lifecycle

## Mental Model

**TLS solves three problems** for a connection: **encryption** (nobody can read it), **integrity** (nobody can modify it), and **authentication** (you're really talking to who you think — this is what certificates are for).

**How a certificate proves identity:** a certificate binds a *name* (CN/SAN, e.g. `api.example.com`) to a *public key*, and is *signed* by a Certificate Authority (CA). Your OS/browser ships with a **trust store** of root CAs. During the TLS handshake the server presents its certificate plus **intermediate certificates** forming a **chain**: leaf → intermediate(s) → a root your client already trusts. The client verifies each signature up the chain, checks the name matches the host it dialed, checks validity dates, and checks revocation. If any link fails, the handshake fails.

**Handshake in one breath (TLS 1.2/1.3):** ClientHello (supported versions/ciphers + **SNI**, the plaintext hostname that lets one IP serve many certs) → ServerHello + certificate chain → key exchange (ECDHE for forward secrecy) → both derive session keys → encrypted application data. TLS 1.3 shortens this to one round trip.

**The failure taxonomy — almost every TLS incident is one of six:**
1. **Expired certificate** (dates).
2. **Name mismatch** (cert says `example.com`, client dialed `api.example.com`; wildcards `*.example.com` cover one level only — not `a.b.example.com`, not the bare apex).
3. **Incomplete chain** (server didn't send intermediates; browsers often hide this by caching/fetching intermediates, `curl` and programmatic clients fail — the classic "works in Chrome, fails in the app").
4. **Untrusted issuer** (private CA or corporate TLS-inspection proxy like Zscaler re-signing traffic with a CA the client doesn't trust).
5. **Protocol/cipher mismatch** (old client can't do TLS 1.2+, or hardened server refuses old ciphers).
6. **Clock skew** (client's clock outside the cert's validity window — containers and IoT devices).

**ACM (AWS Certificate Manager):** free public certs for use on AWS *integrated* services — ALB/NLB, CloudFront, API Gateway. Two giant properties:
- **You can never export the private key** of an ACM public cert. It only lives inside AWS services. Need a cert on an EC2 instance / on-prem / in a pod? ACM public certs can't do it — use ACM Private CA (exportable), Let's Encrypt, or a commercial cert.
- **Auto-renewal** — with **DNS validation**, ACM renews automatically forever as long as the validation CNAME record stays in DNS. Email validation requires a human and is how certs get forgotten and expire. Rule: always DNS validation, never delete the validation CNAME.
- **Regionality:** ACM certs are regional. CloudFront requires the cert in **us-east-1**, full stop. An ALB in eu-west-1 needs the cert in eu-west-1.

**Where TLS terminates matters:** at an ALB, TLS ends there; ALB→target is a *new* connection (plain HTTP or re-encrypted HTTPS — and the ALB does *not* strictly validate the target's cert, so self-signed on targets is fine). NLB can pass TCP straight through (end-to-end TLS to the instance) or terminate TLS itself. CloudFront has two separate TLS legs: viewer→CloudFront and CloudFront→origin, configured independently — and CloudFront **does** validate the origin's certificate.

---

## Q20. Your site went down with certificate-expired errors. Walk through immediate response, then how you make sure it never happens again.

**Answer:**

"**Immediate:**
1. Confirm and locate: `openssl s_client -connect site.com:443 -servername site.com | openssl x509 -noout -dates -subject -issuer`. The `-servername` flag matters — it sets SNI, and without it you may inspect the wrong certificate on a multi-cert endpoint. Identify *which* cert in the path expired: edge (CloudFront), load balancer, or origin/backend — each is a separate cert. Users see whichever leg is broken; `curl -v` at each hop isolates it.
2. Replace: if ACM with DNS validation, renewal should have been automatic — its failure means the validation CNAME was deleted or the domain moved; re-validate or issue a fresh cert and swap it on the listener (ALB listener cert swap is seconds, no downtime). If it's a manually-managed cert on an instance, deploy the renewed cert and reload the web server (`nginx -s reload` — reload, not restart).
3. Verify from outside: multiple clients, and check the **full chain** is served, not just the leaf.

**Prevention — the real answer:**
1. **Move everything possible to ACM with DNS validation** — renewal becomes a non-event. Protect the validation CNAMEs (IaC-managed, alarms on deletion).
2. **Inventory + monitoring for the rest:** ACM emits `DaysToExpiry` via CloudWatch; AWS Config rule `acm-certificate-expiration-check`; for non-ACM certs, external synthetic checks (a scheduled Lambda or monitoring tool doing s_client against every public endpoint) alerting at 30/14/7 days. The monitoring must be *from the client's perspective* — checking the cert file on disk misses 'renewed but never deployed.'
3. **Kill email validation** everywhere; it's the process that depends on a human reading an inbox.
4. For instance/pod-level certs: automate with cert-manager (Kubernetes) or ACM Private CA integration — the goal is no certificate anywhere that requires a calendar reminder.

The senior framing: certificate expiry is never a certificate problem, it's a *process* problem — a manual step existed where automation should have been."

---

## Q21. `curl` from a server fails with "unable to verify the first certificate" but the site opens fine in a browser. Explain and fix.

**Answer:**

"This is the **incomplete chain** signature. The server is sending only its leaf certificate without the intermediate CA cert. Browsers mask this: they cache intermediates from other sites and can fetch missing ones via the certificate's AIA (Authority Information Access) URL. `curl`, `wget`, Java, Python, and most programmatic clients do neither — they need the server to present the full chain, and it isn't, so verification dies at the first link.

**Prove it:** `openssl s_client -connect host:443 -servername host -showcerts`. You'll see one certificate where there should be two or three, and an error like `unable to get local issuer certificate` / `verify error:num=21`. An SSL Labs scan reports 'chain issues: incomplete' explicitly.

**Fix:** configure the server to serve leaf **plus intermediates** — on nginx that means the `ssl_certificate` file must be the concatenation: leaf first, then intermediate(s) (`cat leaf.crt intermediate.crt > fullchain.crt`). CA-provided 'fullchain' files exist for exactly this. On an ALB/ACM this basically can't happen because ACM manages the chain — which is itself an argument for terminating TLS on managed services.

**Two lookalikes to rule out:**
- If curl fails on *every* site including google.com → the server's **CA trust store** is broken or outdated (`update-ca-certificates` / `update-ca-trust`), or the traffic passes a **TLS-inspecting proxy** (Zscaler) whose corporate root CA isn't installed on this box. The issuer shown by s_client gives it away — if the issuer is 'Zscaler' instead of the real CA, it's inspection.
- Root cert *too new/old* for an outdated trust store (the Let's Encrypt DST Root X3 expiry in 2021 broke old systems exactly this way)."

---

## Q22. End-to-end encryption requirement: traffic must be encrypted from client all the way to the application pod/instance. How do you architect it with an ALB, and what changes with an NLB?

**Answer:**

"**With an ALB (two TLS sessions):** client→ALB uses the ACM cert on the HTTPS listener; then ALB→target is a *second, independent* TLS connection — set the target group protocol to HTTPS. The traffic is encrypted on both legs, but it is *terminated and re-originated* at the ALB, meaning the ALB sees plaintext momentarily (in memory). Critical detail interviewers probe: **the ALB does not validate the target's certificate** — expired or self-signed certs on targets are accepted. That's operationally convenient (no cert management fire drills on backends) but means the backend leg's authentication is weaker; compensate with network controls (SG allowing only the ALB) if the threat model cares. This satisfies most 'encryption in transit' compliance requirements.

**With an NLB (true end-to-end):** run a **TCP listener** and the NLB forwards bytes untouched — the TLS session is negotiated directly between client and target. The target must hold the certificate and private key (so not ACM public certs — they're non-exportable; use Private CA/Let's Encrypt/cert-manager). You lose L7 features (path routing, WAF, HTTP header manipulation) because the LB never sees inside the stream. You also lose `X-Forwarded-For`; client IP is preserved natively by NLB in most modes, or via Proxy Protocol v2.

**Choosing:** compliance usually says 'encrypted in transit' → ALB re-encryption is standard and keeps L7 features + WAF. A hard requirement that *no intermediary can decrypt* (some regulated/zero-trust designs) → NLB passthrough or service-mesh mTLS (Istio/App Mesh) inside the cluster, where sidecars do mutual TLS pod-to-pod with automatic short-lived certs — which is the modern answer to cert distribution at scale."

---

## Q23. After a "security hardening" change, some customers report they can't connect at all — TLS handshake failures — while most are fine. What happened?

**Answer:**

"Selective breakage right after hardening = **protocol/cipher floor raised above what some clients can speak**. The change was almost certainly the security policy on the ALB/CloudFront (e.g., moving to a TLS 1.2-only or TLS 1.3-only policy, or dropping older cipher suites). The clients failing are old: Java 7, .NET 4.5-era apps, old Android, embedded devices, ancient curl/OpenSSL builds, or partner systems — anything that can't negotiate the new minimum. The handshake dies at ClientHello: server finds no mutually acceptable version/cipher and sends a fatal alert; users see 'handshake failure' with no HTTP response at all.

**Diagnose:**
1. Reproduce per-version: `openssl s_client -connect host:443 -tls1_1` vs `-tls1_2` vs `-tls1_3` — shows exactly which floor was set.
2. Identify affected clients from data: ALB access logs include the negotiated `ssl_protocol` and `ssl_cipher` per request; *before* such a change, you query these logs to quantify how much traffic still uses TLS 1.0/1.1 and old ciphers — that's the impact analysis that should have happened. CloudFront can log the same.
3. Check whether it's cipher-specific rather than version: a client that supports TLS 1.2 but only RSA key-exchange ciphers will fail against an ECDHE-only policy.

**Resolve:** business decision, not just technical — either roll back to a policy including the needed ciphers temporarily, with a deadline and comms for clients to upgrade (the standard path — PCI killed TLS 1.0 industry-wide this way), or hold the line if the affected traffic is negligible/unwanted (scanners, bots). The senior lesson to state: TLS policy changes are *breaking changes for clients* and must be preceded by negotiated-protocol analytics from access logs, announced deprecation windows, and ideally a canary (change one endpoint first)."

---

## Q24. Explain mTLS and a scenario where you'd need to configure it in AWS.

**Answer:**

"Normal TLS authenticates one direction: the *server* proves its identity; the client stays anonymous at the TLS layer (authentication happens later via API keys/tokens). **Mutual TLS adds the reverse:** the server requests a certificate from the *client* during the handshake and validates it against a CA the server trusts. Both sides prove identity cryptographically before a single application byte flows. Connections without a valid client cert never even reach the application.

**Where it shows up in real AWS work:**
1. **Partner/B2B APIs:** a bank or healthcare partner mandates mTLS to your API. **API Gateway supports mTLS natively** — you upload a truststore (the CA certs that sign valid client certs) to S3, enable mTLS on a custom domain, and API Gateway rejects handshakes lacking a valid client cert. Same on ALB, which added mTLS support (verify mode with a trust store, passing cert details to targets in headers).
2. **Service-to-service (zero trust):** service meshes give every pod a short-lived cert (SPIFFE identity) and enforce mTLS between all services — network location stops implying trust; identity is per-workload.
3. **IoT:** every device holds a unique client cert; AWS IoT Core authenticates devices via mTLS.

**Operational pain points to mention (this is what makes the answer senior):** client cert *distribution and rotation* is the hard part — expiry of a client cert breaks that client with an opaque handshake error, and revocation (CRL/OCSC handling) needs a defined process. Debugging is harder too: failures happen pre-HTTP, so there's no status code — you diagnose from LB/API Gateway connection logs and `openssl s_client -cert client.crt -key client.key` to test with a client cert."


---

# PART 5 — Load Balancers (ALB/NLB), CloudFront, API Gateway

## Mental Model

**Load balancing = one stable front door, many interchangeable backends.** AWS gives you two main types:

- **ALB (Application Load Balancer)** — Layer 7 (HTTP/HTTPS). It *terminates* the client connection, reads the request, and makes routing decisions on host, path, headers, query strings. Supports target groups of instances, IPs, Lambda; WebSockets; HTTP/2; authentication; WAF attachment. Because it proxies, the backend sees the ALB's IP as the source — the real client IP arrives in the `X-Forwarded-For` header.
- **NLB (Network Load Balancer)** — Layer 4 (TCP/UDP/TLS). Ultra-high performance, static IPs per AZ (assignable Elastic IPs — the reason firewalled partners demand NLB), preserves client source IP, can pass TLS through untouched. No HTTP awareness: no path routing, no WAF.

**Target groups & health checks:** the LB continuously probes each target (`/health`, expected 200). Unhealthy targets stop receiving *new* traffic. The single most important operational fact: **if ALL targets are unhealthy, the ALB "fails open" and sends traffic to all of them anyway** — so "everything unhealthy" often looks like intermittent errors, not a clean outage.

**Cross-zone load balancing:** ALB always distributes across AZs evenly; NLB has it configurable (historically off by default — a classic source of "one AZ's targets get hammered").

**CloudFront** = global CDN: 400+ edge locations cache your content near users. Core loop: request hits edge → cache hit? serve instantly : forward to **origin** (S3, ALB, API Gateway, anything HTTP) → cache per **Cache Policy** (what's in the cache key: path + which headers/cookies/query strings) and TTL → serve. More things in the cache key = more cache fragmentation = lower hit ratio. **Origin Access Control (OAC)** lets an S3 origin be private — only CloudFront can read it, enforced by bucket policy. Even for *uncacheable* dynamic APIs, CloudFront helps: TLS terminates at the edge and the origin fetch rides the AWS backbone, cutting latency; plus it's the natural attachment point for WAF and DDoS absorption.

**API Gateway** = managed API front end: routing, **authentication (IAM/Cognito/Lambda authorizers)**, throttling and usage plans, request validation and transformation, caching — the cross-cutting API concerns as configuration. Two HTTP flavors: **REST API** (full features: API keys, usage plans, request transformation, caching) and **HTTP API** (cheaper, faster, JWT auth, fewer features). Critical hard limit: **29-second integration timeout** on REST APIs (now adjustable in some cases, but the default that bites everyone) — long-running work must go async. Private integrations reach into a VPC via **VPC Link** (which for REST APIs targets an NLB; for HTTP APIs targets an ALB/NLB/Cloud Map).

---

## Q25. Users report intermittent 5xx errors through the ALB. How do you tell whether it's the ALB or your application, and what do the different 5xx codes mean?

**Answer:**

"The status code plus two CloudWatch metrics answer 'whose fault' immediately: **`HTTPCode_ELB_5XX_Count`** (the ALB itself generated the error) vs **`HTTPCode_Target_5XX_Count`** (the backend returned it and the ALB relayed). That split is the fork in the road.

**Target-generated 5xx** → application problem: exceptions, DB failures, overload. Go to app logs/APM.

**ALB-generated codes, each with a specific meaning:**
- **502 Bad Gateway:** the ALB couldn't get a valid response — target closed the connection mid-request, sent garbage, or crashed. The classic *hidden* cause: **keep-alive timeout mismatch**. The ALB reuses backend connections with a 60s idle timeout (default); if the app's keep-alive timeout is *shorter* (nginx default 75 is fine, but many apps use 5–30s), the app closes an idle connection at the exact moment the ALB sends a request on it → race → 502. Fix: app keep-alive timeout **greater than** the ALB idle timeout. Intermittent 502s at low traffic are this, more often than not.
- **503 Service Unavailable:** no registered targets, or no *healthy* targets in a routable state — check target group registration and health.
- **504 Gateway Timeout:** target didn't respond within the ALB idle timeout (60s default) — slow queries, deadlocks, undersized backend. Raise the timeout only as a band-aid; fix the latency.
- **500 from the ELB itself:** rare; auth actions or ALB internal errors.

**Systematic approach:** (1) split ELB vs Target 5xx metrics, per target group and per AZ — a single-AZ pattern points to one bad host or an AZ event; (2) ALB **access logs** (enable them always — they include target IP, target processing time, and the actual status pair) to see which targets and which request patterns; (3) correlate with health check flapping (`UnHealthyHostCount`), deployments, and traffic spikes; (4) for 502s, check connection-timeout configs both sides; (5) if all targets show unhealthy, remember the ALB fails open — health check misconfig (wrong path/port after a deploy) masquerades as intermittent 5xx while the app is actually fine."

---

## Q26. After a deployment, the target group shows all targets unhealthy but the application "works fine" when you curl it directly. What are the usual causes?

**Answer:**

"Health check config drifted from application reality. In order of frequency:

1. **Wrong path:** deployment changed/removed the health endpoint (`/health` → `/healthz`, or the new framework 404s the old path). The health check expects 200; a 404 or 301 fails it. Note the matcher — a redirect (301/302) fails unless the success codes include it. People curl `/` and say 'it works' while the check probes `/health`.
2. **Wrong port:** target group checks the *traffic port* by default; if the app moved ports or a sidecar/proxy pattern changed, the probe hits nothing. In ECS with dynamic port mapping, the target group must use the dynamically registered port (it does automatically) — but a hardcoded health check port breaks this.
3. **Security group:** the target's SG must allow the health check from the **ALB's security group** on the health check port. Curling from your laptop or from the instance itself tests a *different* source. New SGs after infra changes commonly miss this rule.
4. **App binding:** app listens on `127.0.0.1` — local curl works (that's *why* 'it works when I curl it'!), external probes fail. `ss -tlnp` tells the truth: it must bind `0.0.0.0` or the pod/ENI IP.
5. **Timing:** app takes 90s to boot, health check gives it (interval × unhealthy-threshold) ≈ 30s and marks it dead; in ECS this becomes a kill-loop as the scheduler replaces 'unhealthy' tasks forever. Fix: health check grace period / startup-tolerant thresholds.
6. **Slow health endpoint:** the check has a response timeout (default 5s); a `/health` that calls the database can exceed it under load — the app 'works' but answers health probes too slowly, causing flapping that tracks load.

The phrase 'works when I curl it' should trigger one reflex: *curl from where, to which port, on which path?* The health check's exact source, port, and path is the only test that counts — replicate it precisely, e.g. curl from another instance with the ALB's SG."

---

## Q27. A partner requires whitelisting your load balancer's IP addresses in their firewall. Why is this a problem with an ALB, and what's the solution?

**Answer:**

"**ALB IPs are not static.** An ALB is a fleet of nodes whose IPs change as AWS scales, replaces, or rebalances them — that's why you must always use its DNS name (or better, a Route 53 Alias to it). Any hardcoded ALB IP will eventually go stale and the integration dies mysteriously weeks later. So 'whitelist the ALB's IPs' is an impossible request as stated.

**Solutions:**
1. **NLB, which has static IPs** — one per AZ, and you can assign your own Elastic IPs. If L4 is enough, front the service with an NLB and hand the partner the EIPs; done. This is *the* textbook reason to pick NLB over ALB.
2. **Need ALB features (L7 routing, WAF) *and* static IPs:** put an **NLB in front of the ALB** — natively supported: NLB with an ALB-type target group. Partner whitelists the NLB's EIPs, ALB does its L7 job behind it.
3. **AWS Global Accelerator** in front of the ALB: gives two static anycast IPs valid globally, plus performance benefits — the managed version of the same idea, common for global partner integrations.

Flip side worth volunteering: for *outbound* whitelisting ('partner must whitelist the IPs we call *from*'), the answer is a **NAT Gateway's Elastic IP** — all egress from private subnets exits with that stable address."

---

## Q28. CloudFront is serving stale content after a release / or: your cache hit ratio is terrible. Handle both.

**Answer:**

"**Stale content:** cached objects live until TTL expiry; deploying new files doesn't touch edge caches. Immediate fix: **invalidation** (`/*` or specific paths — takes a minute or two, first 1,000 paths/month free, then paid). But invalidation-on-every-deploy is a smell. The real fix is **versioned asset names**: build pipelines emit `app.a1b2c3.js` (content-hashed filenames) referenced by an `index.html` that itself has a short TTL or `no-cache`. New release = new filenames = instant, cache-busting-free rollout, with the old assets still valid for users on the old HTML. Long TTLs on hashed assets (a year), short on HTML — that's the standard pattern.

**Bad cache hit ratio:** hit ratio dies by cache-key fragmentation — every distinct cache key stores a separate copy:
1. **Forwarding too much in the cache policy:** all cookies, all headers, or all query strings in the cache key means near-every request is 'unique' and misses. Audit the cache policy: include *only* what actually changes the response. A session cookie in the cache key ≈ per-user caching ≈ 0% hit ratio (and a data-leak risk if the response is actually shared).
2. **Query-string noise:** marketing parameters (`utm_*`, `fbclid`) fragment the key. Exclude them or allowlist only meaningful params. Same for parameter *ordering* differences.
3. **Origin Cache-Control headers:** `no-cache`, `private`, `max-age=0` from the origin defeat caching even when CloudFront would allow it; developers often set these accidentally framework-wide. Check what the origin actually sends.
4. **Low TTLs** or genuinely dynamic content: measure with the `x-cache` response header (Hit/Miss from cloudfront) and the CloudFront cache statistics report; segment by path — often one chatty API path drags the average down while static assets are fine, and the fix is separate **cache behaviors** per path pattern with appropriate policies each."

---

## Q29. Users can reach your ALB directly, bypassing CloudFront (and its WAF). Why is this bad and how do you lock it down?

**Answer:**

"The ALB has a public DNS name; anyone who discovers it (certificate transparency logs make this trivial) can hit the origin directly — skipping CloudFront's WAF rules, rate limiting, geo blocks, and DDoS absorption. Your security controls only protect the front door while the back door is open.

**Lockdown options, weakest to strongest:**
1. **SG with the CloudFront managed prefix list:** AWS publishes a managed prefix list of CloudFront origin-facing IP ranges; the ALB's security group allows 443 only from that list. Blocks the general internet — but any *other* CloudFront distribution (including an attacker's) is still allowed through, since the IPs are shared by all of CloudFront.
2. **Custom header validation (the standard):** CloudFront adds a secret custom header (e.g. `X-Origin-Verify: <random>`) on origin requests; an ALB listener rule forwards only requests bearing it, else fixed-response 403. Combined with #1, this is the common production pattern. Rotate the secret periodically (two-header overlap during rotation to avoid downtime).
3. **Strongest, newer option:** CloudFront **VPC origins** — the ALB becomes *private* (internal, no public exposure at all) and CloudFront reaches it inside the VPC. Nothing to bypass. Where available, this is the clean answer.

For S3 origins the equivalent is **OAC** with a bucket policy allowing only that distribution. And the honest senior note: same logic applies one layer down — targets should only accept traffic from the ALB's SG, so nobody bypasses the ALB either. Every layer only accepts from the layer above."

---

## Q30. An API behind API Gateway returns 504 for one specific operation that takes about a minute. What's happening and how do you redesign it?

**Answer:**

"API Gateway enforces a **~29-second integration timeout** (the historic hard cap on REST APIs; it's now raisable in some configurations, but raising it is rarely the right move). Your one-minute operation can never finish inside the synchronous window — the gateway cuts the connection at 29s and returns 504 while the backend may actually still be working (which also means the work might complete invisibly, causing 'it failed but it happened' confusion — worth checking for duplicate side effects).

**Redesign — the async pattern:**
1. **Accept-then-poll:** the API immediately validates the request, drops a job onto SQS (or starts a Step Functions execution), and returns **202 Accepted** with a job ID in milliseconds. A worker (Lambda/ECS) processes it. The client polls `GET /jobs/{id}` for status/result, or better, gets a callback/webhook/WebSocket push when done. This is the canonical answer.
2. **Step Functions integration:** API Gateway can start a Step Functions execution directly (no Lambda glue); the state machine handles the long work, retries, and error states, and its execution ARN doubles as the job ID.
3. If the operation is only slow because of a cold path or an unoptimized query, fixing the latency is option zero — async adds real complexity (idempotency, status storage, client changes) and shouldn't be reached for reflexively.

Related timeout stack worth reciting: client timeout ≥ CloudFront origin timeout ≥ API Gateway 29s ≥ Lambda timeout (max 15 min, but capped in effect at 29s when invoked synchronously through REST API Gateway). Misaligned timeouts produce ghost errors where the caller gives up while the callee succeeds."

---

## Q31. Compare where you'd put ALB vs API Gateway in front of a service — they seem to overlap.

**Answer:**

"They overlap at 'HTTP routing to backends' and diverge everywhere else:

**API Gateway earns its cost when you need API-product features:** per-client authentication and authorization (IAM, Cognito, JWT, Lambda authorizers), throttling and quotas per API key/usage plan (monetized or partner-facing APIs), request/response validation and transformation, built-in caching, and native serverless integrations (direct to Lambda, Step Functions, DynamoDB without compute in between). It's pay-per-request — brilliant at low/spiky volume, and it's the natural front for Lambda.

**ALB wins when it's 'just' traffic distribution to always-on services:** container/instance backends (ECS/EKS), WebSockets at scale, very high sustained request volumes (hourly+LCU pricing beats per-request pricing at high throughput — at hundreds of millions of requests/month, API Gateway's bill dwarfs an ALB's), gRPC, and when you don't need per-client API management. ALB also invokes Lambda if needed.

**Real-world composite:** public API products → CloudFront + WAF + API Gateway (+ VPC Link to internal NLB/ALB). Internal microservices on EKS → ALB via ingress controller, no API Gateway at all. High-volume mobile backend → often ALB for the hot data path and API Gateway only for the management/partner plane.

Rules of thumb to state: need usage plans/API keys/request validation → API Gateway. Need cheapest stable high throughput to containers → ALB. Both → CloudFront in front of either for TLS-at-edge, WAF, and DDoS posture."


---

# PART 6 — ECS, Lambda, Step Functions

## Mental Model

**ECS (Elastic Container Service)** runs Docker containers with three nouns:
- **Task definition** — the blueprint (image, CPU/memory, env vars, ports, IAM roles, logging config). Versioned as revisions.
- **Task** — a running copy of a task definition (one or more containers).
- **Service** — the supervisor that keeps N tasks running, replaces dead ones, wires them to a load balancer target group, and performs rolling deployments.

Two launch types: **EC2** (you manage the instances tasks land on) and **Fargate** (serverless — AWS runs the task, you never see a host; each task gets its own ENI in your subnet with `awsvpc` networking, meaning tasks have their own IPs and their own security groups).

**Two IAM roles per task — this distinction is a guaranteed interview question:**
- **Task execution role** — used by the *ECS agent* to set the task up: pull the image from ECR, fetch secrets from Secrets Manager/SSM, create the CloudWatch log stream. Failures here mean the task never starts.
- **Task role** — assumed by *your application code* inside the container to call AWS APIs (S3, DynamoDB…). Failures here mean the app runs but gets AccessDenied.

**Lambda** = event-driven functions: upload code, AWS runs it per event, bills per millisecond. The runtime model to internalize: an invocation runs in an **execution environment** that is *frozen and reused* between invocations. First use (or scale-up) requires a **cold start**: provision environment + load code + run initialization (everything outside the handler). Subsequent invocations on a warm environment skip all that — which is why you open DB connections and SDK clients *outside* the handler (they persist), and why global state persisting between invocations surprises people. Key limits: 15-minute max duration, memory 128MB–10GB (**CPU scales with memory** — the counterintuitive perf lever), concurrency limited per account/region (default ~1,000, raisable), `/tmp` scratch space.

**Sync vs async invocation changes everything about errors:** synchronous (API Gateway, direct invoke) — caller gets the error, caller retries. Asynchronous (S3 events, SNS, EventBridge) — Lambda retries **twice** automatically, then the event goes to a **DLQ/failure destination** if configured (or is lost if not). Stream/queue sources (SQS, Kinesis, DynamoDB Streams) — an **event source mapping** polls and invokes; failed batches return to the queue and retry until maxReceiveCount → DLQ, and (for streams) a poison record can block a shard. Because retries are everywhere, **handlers must be idempotent**.

**Step Functions** = a state machine orchestrating steps (Lambda calls, ECS tasks, direct AWS SDK integrations) with retries, catch/fallback paths, parallelism (`Map`), human-wait patterns, and full execution history. **Standard** workflows: up to a year, exactly-once, priced per state transition — orchestration/business processes. **Express**: up to 5 minutes, at-least-once, priced per request/duration — high-volume event processing. Step Functions exists so retry/timeout/error-routing logic lives *declaratively in the workflow* instead of hand-written inside application code.

---

## Q32. An ECS Fargate task starts and immediately stops, over and over. How do you troubleshoot?

**Answer:**

"First, get the reason ECS itself recorded: the **stopped task's `stoppedReason`** (Console → cluster → tasks, filter Stopped; or `aws ecs describe-tasks`). Stopped tasks are only retrievable for a short while, so grab it fast. The reason string routes you:

1. **`CannotPullContainerError`** — can't fetch the image. Sub-causes: wrong image tag; **no network path to ECR** (private-subnet task with no NAT and no ECR VPC endpoints — Fargate must pull over the network; note ECR needs *three* endpoints: `ecr.api`, `ecr.dkr`, and the **S3 gateway endpoint**, because image layers live in S3 — missing the S3 one produces maddening partial failures); or **execution role** lacks ECR permissions. If the task is in a public subnet, it also needs `assignPublicIp=ENABLED` or it has no route out.
2. **`ResourceInitializationError` mentioning secrets** — execution role can't read the referenced Secrets Manager/SSM parameters (or a KMS key). Again *execution* role, not task role.
3. **Container exits immediately (`Essential container in task exited`)** — the app itself crashes: check **CloudWatch Logs** for the container. If there are *no logs at all*, the crash is pre-logging: bad entrypoint/command, missing env var read at startup, or the log configuration itself is broken (awslogs group doesn't exist and the execution role can't create it). Run the same image locally with the same env/command to reproduce — `docker run` with the task's entrypoint reveals 'exec format error' (wrong CPU architecture — an image built on an ARM Mac deployed to x86 Fargate, an extremely common modern cause; fix with `--platform linux/amd64` builds or set the task's CPU architecture to ARM64).
4. **`OutOfMemoryError` / OOMKilled (exit code 137)** — container exceeded memory; raise task memory or fix the leak. Exit code 137 = SIGKILL = almost always OOM.
5. **Health check kill-loop** — task starts fine, target group health check fails (see the ALB section), ECS replaces it, forever. The stopped reason says failed health checks; the fix is the health check or the `healthCheckGracePeriod`.

Habit to state: `stoppedReason` first, container logs second, local reproduction third. Random guessing at task definitions is what junior engineers do."

---

## Q33. ECS service deployment is stuck — new tasks won't stabilize, old tasks won't drain. What's going on and how do rolling deployments actually work?

**Answer:**

"Rolling deployment mechanics: the service starts new-revision tasks (bounded by `maximumPercent`, default 200%), waits for them to pass **health checks** (container health check and/or target group health), registers them with the LB, then drains and stops old tasks (bounded by `minimumHealthyPercent`). 'Stuck' means new tasks never reach *healthy*, so ECS refuses to kill the old ones — which is the system protecting you.

**Diagnosis:**
1. **Service events tab** — ECS narrates exactly what it's doing: 'task failed container health checks,' 'unable to place task,' etc. Always start there.
2. New tasks failing health checks → previous question's logic (path/port/SG/slow startup/grace period).
3. **Unable to place tasks:** on EC2 launch type — insufficient CPU/memory/ports on the instances (with static port mappings only one task per port per host!); on Fargate — subnet out of IPs, or hitting Fargate service quotas.
4. **Capacity math deadlock:** desired=2, min healthy=100%, max=200% needs room for 4 tasks during deploy; if capacity only fits 2, the deploy can't start new ones without killing old ones, which min-healthy forbids → stuck. Fix the percentages or capacity.
5. **Draining that never finishes:** target group **deregistration delay** (default 300s) plus long-lived connections (WebSockets) keep old tasks in 'draining' for a long time — that part is normal and tunable.

**Safety nets to name:** enable the **deployment circuit breaker** (with rollback) so a bad revision auto-reverts instead of hanging for hours; use **CloudWatch alarms on the deployment** for metric-based rollback; consider blue/green via CodeDeploy for the high-stakes services. The interview-grade summary: a stuck ECS deployment is ECS refusing to make things worse — read the events, fix why new tasks aren't healthy, and install the circuit breaker so it fails fast next time."

---

## Q34. Your Lambda works in testing but in production intermittently fails with timeouts when calling an RDS database in a VPC. Multiple angles — walk through them.

**Answer:**

"Lambda + RDS is a minefield of four classic issues, and 'intermittent under load' suggests #2 or #3:

1. **VPC networking basics:** a Lambda that must reach a private RDS needs VPC configuration (subnets + SG). Its SG must be allowed inbound on 5432/3306 by the RDS SG. Symptom of wrong networking is *consistent* timeouts, not intermittent — but check first. Also, a VPC-attached Lambda loses internet access unless a NAT path exists (breaking calls to public APIs and even some AWS endpoints without VPC endpoints) — a companion failure people miss.
2. **Connection exhaustion — the big one.** Every concurrent Lambda execution environment opens its *own* DB connection (they can't share a pool across environments). A traffic spike to 500 concurrent Lambdas = 500 connections; RDS `max_connections` on a small instance might be ~100–200. New connections get refused/hang; existing ones slow down. Evidence: RDS `DatabaseConnections` metric pinned at max, correlating with Lambda concurrency spikes. **Fix: RDS Proxy** — a managed connection pooler between Lambda and RDS that multiplexes thousands of Lambda connections onto a small warm pool, also smoothing failovers. This question exists in interviews basically to see if you say 'RDS Proxy.'
3. **Doing connections wrong in code:** opening a connection *inside* the handler per invocation adds connect+TLS latency every time and churns connections. Correct pattern: create the client **outside** the handler so warm environments reuse it — with validation/reconnect logic because idle frozen connections get silently dropped by the DB or by conntrack timeouts, producing the 'first request after idle fails' signature. That exact signature — failures after quiet periods — is stale-connection reuse.
4. **Timeout stack misalignment:** Lambda timeout must exceed worst-case query time; the SDK/driver connect timeout must be shorter than the Lambda timeout so you get a real error ('connection refused') instead of the Lambda dying at its own timeout with a useless generic message. And if API Gateway fronts it, everything must fit in 29s.

Also worth one sentence: cold starts in VPC used to add ~10s of ENI setup — Hyperplane ENIs fixed this years ago, so if someone blames VPC cold starts today, that's outdated knowledge; modern VPC cold-start overhead is small."

---

## Q35. A Lambda processing SQS messages is causing duplicates downstream, and some messages end up in the DLQ having "failed" although their work appears done. Explain the mechanics and fixes.

**Answer:**

"Both symptoms come from the same root: **SQS is at-least-once, and a message is only 'done' when it's deleted after a *successful* Lambda return.** The event source mapping polls a batch, invokes the Lambda, and deletes the batch messages only if the invocation succeeds. Failure modes:

1. **Function succeeds in effect but errors at the end** (throws after the DB write, or times out post-work): the message isn't deleted, becomes visible again after the **visibility timeout**, is reprocessed → duplicate work; after `maxReceiveCount` failures it lands in the DLQ *even though its side effects happened*. That's exactly the 'failed but done' confusion.
2. **Visibility timeout shorter than processing time:** message reappears and a *second* Lambda starts processing while the first is still running → guaranteed duplicates. Rule: **visibility timeout ≥ 6× the Lambda timeout** (AWS's guidance for the ESM) — at minimum comfortably longer than the function timeout.
3. **Batch failure semantics:** by default, one bad message fails the *whole batch* — nine successfully processed messages get retried alongside the poison one → duplicates for the nine, and the poison message eventually drags others toward the DLQ. **Fix: `ReportBatchItemFailures`** — the function returns which items failed; only those retry.

**The fundamental fix is idempotency,** because at-least-once delivery makes retries a matter of when, not if:
- Use a natural or embedded **idempotency key** per message; before processing, check a store (DynamoDB with conditional writes is the standard) — seen it? skip. Powertools for Lambda has an idempotency utility that packages this.
- Make operations naturally idempotent where possible (upserts, `PUT` semantics, conditional updates) rather than increments/appends.
- Note that even SQS FIFO's exactly-once *delivery* dedup window doesn't remove the need for idempotent *processing* — the consumer can still crash mid-work.

DLQ hygiene completes the answer: alarm on DLQ depth, keep enough context in messages to replay them, and have a redrive process (SQS redrive-to-source is built in now)."

---

## Q36. When do you reach for Step Functions instead of chaining Lambdas via SQS/direct calls? Give a scenario and explain error handling.

**Answer:**

"**The smell that says 'Step Functions':** workflow logic — retries, branching, waiting, compensation — is being hand-written inside function code, or state about 'where are we in the process' is being smuggled through queues. Chained Lambdas hide the workflow; nobody can see where an order got stuck without archaeology through logs.

**Scenario:** order processing — validate → charge payment → reserve inventory → arrange shipping → notify. Requirements: payment retried with backoff but *never* double-charged, inventory failure must *refund* the payment (compensation), shipping can take a human day (wait for callback), and support must see exactly where any order is.

**Why Step Functions fits:**
- **Retry/Catch per state, declaratively:** each state defines `Retry` (error types, interval, backoff, max attempts) and `Catch` (route to a fallback state). The payment state retries transient errors 3× with exponential backoff, but a `CardDeclined` error routes straight to a 'notify customer' branch — no try/except pyramids in code.
- **Compensation (saga pattern):** inventory failure catches into a `RefundPayment` state. The rollback logic is *visible in the diagram*, which is the whole point for auditability.
- **`waitForTaskToken`** pauses the execution (up to a year on Standard) until an external system/human posts the token back — the shipping-callback and approval-step pattern, impossible to do cleanly with raw Lambda chaining.
- **Observability for free:** every execution shows each state's input/output and where it failed; support looks at a picture, not CloudWatch Logs spelunking.
- **Direct SDK integrations** remove glue Lambdas entirely (Step Functions can call DynamoDB, SNS, ECS RunTask natively).

**When NOT to:** simple linear pipe (S3 → Lambda → done), extreme-volume simple event processing where per-state-transition pricing on Standard adds up (that's Express workflows or plain SQS+Lambda), or sub-millisecond latency paths. The one-line decision rule: if you'd need to *draw* the process to explain it to someone, it belongs in Step Functions where the drawing is the implementation."

---

## Q37. Lambda cold starts are hurting a latency-sensitive API. What actually works to reduce them?

**Answer:**

"A cold start = provision environment + download/mount code + initialize the runtime + run *your* init code (imports, client construction). Attack each part:

1. **Measure first:** X-Ray or CloudWatch `Init Duration` in the REPORT log lines quantifies cold-start frequency and cost. If cold starts are 0.5% of invocations at +400ms, the business impact may be nil — say this; it's the senior move.
2. **Shrink initialization:** trim dependencies (huge node_modules/JARs increase load time), lazy-import rarely used libraries, and keep only necessary client construction in global scope. Language choice matters: Python/Node cold-start in tens-to-hundreds of ms; JVM/.NET historically in seconds — for Java, **SnapStart** (snapshot of the initialized VM, restored on demand) cuts it dramatically.
3. **More memory = more CPU = faster init.** Bumping 128MB → 1024MB often cuts init time enough to pay for itself. Tune with AWS Lambda Power Tuning (the well-known state machine) rather than guessing.
4. **Provisioned concurrency** — the direct answer: keep N environments pre-initialized; requests within N never cold-start. Costs money continuously; use with Application Auto Scaling on a schedule/utilization for the latency-critical alias. This is the lever for hard latency SLOs.
5. **Architecture honesty:** if the endpoint needs consistently single-digit-ms and high sustained throughput, a warm container service (ECS/Fargate behind ALB) may simply fit better — Lambda isn't a religion.

Anti-answer worth flagging if the interviewer floats it: scheduled 'pinger' invocations to keep functions warm only keeps *one* environment warm and does nothing under concurrency scale-out; provisioned concurrency replaced that hack."


---

# PART 7 — S3, EBS, RDS (Storage & Databases)

## Mental Model

**S3** = object storage: flat namespace of buckets containing objects addressed by key. Not a filesystem — no real directories (prefixes are naming convention), no partial file edits (you replace whole objects). Properties to internalize: 11-nines durability via replication across ≥3 AZs; **strong read-after-write consistency** (since Dec 2020 — anyone describing eventual consistency for new objects is out of date); virtually infinite scale; per-request pricing plus storage-class pricing (Standard → Intelligent-Tiering → Standard-IA → Glacier tiers, traded off on retrieval cost/time). **Access control layers stack:** bucket policies (resource-based), IAM policies (identity-based), Block Public Access (account/bucket-level master switch that overrides everything), ACLs (legacy — disable via Object Ownership), plus encryption (SSE-S3 default, SSE-KMS adds key-level access control and audit), versioning (protects against overwrite/delete), and lifecycle rules (auto-transition/expire).

**EBS** = network-attached block storage for EC2 — a virtual disk. Volume types: **gp3** (general SSD; baseline 3,000 IOPS + 125MB/s, independently provisionable — the default choice), **io1/io2** (provisioned high IOPS, multi-attach capable), **st1/sc1** (throughput-oriented HDD for sequential workloads). Facts that generate interview questions: an EBS volume lives in **one AZ** (move it via snapshot); it's network-attached, so instance-type network/EBS bandwidth caps can bottleneck before the volume does; **gp2 ties IOPS to size** (3 IOPS/GB with burst credits — small gp2 volumes exhausting burst credits is a legendary "mystery slowness" cause; gp3 removed this trap); snapshots are incremental to S3 and restored volumes **lazy-load** blocks (first-read latency until initialized, fixable with Fast Snapshot Restore).

**RDS** = managed relational databases (Postgres/MySQL/etc.): AWS handles the OS, backups, patching, and failover; you don't get SSH. **Two different replication concepts people mix up:**
- **Multi-AZ** = a synchronous standby in another AZ for **high availability** — it serves *no traffic*; on failure, RDS flips the DNS endpoint to the standby (~1–2 min classic, seconds with Multi-AZ *Cluster* which also adds readable standbys).
- **Read replicas** = asynchronous copies for **read scaling** (and cross-region DR) — they serve reads, they lag by design, and promoting one is a manual DR action.
One sentence to memorize: *Multi-AZ is for availability, read replicas are for scalability.* Performance triad to check on any slow RDS: CPU, memory/freeable, and **IOPS/throughput vs what the storage class provides** — plus Performance Insights to see which queries and waits dominate.

---

## Q38. An application gets `AccessDenied` on `s3:GetObject` even though the IAM policy allows `s3:*` on the bucket. Walk through every layer that could deny.

**Answer:**

"S3 authorization is the intersection of several evaluations, and **any explicit Deny anywhere wins**. My layer walk:

1. **IAM policy scope bug:** `s3:*` on `arn:aws:s3:::my-bucket` covers *bucket-level* actions only; object actions need `arn:aws:s3:::my-bucket/*`. Missing the `/*` resource is the single most common cause of exactly this symptom.
2. **Bucket policy explicit Deny:** common patterns — deny unless requests come from a specific VPC endpoint (`aws:SourceVpce`), specific IPs, or with TLS (`aws:SecureTransport`); a request from outside those conditions is denied regardless of IAM allows. Read the bucket policy for `"Effect": "Deny"` conditions your request violates.
3. **KMS:** if objects use SSE-KMS, `GetObject` also requires `kms:Decrypt` on the key — and the *key policy* must allow it too. AccessDenied on KMS-encrypted objects with a clean S3 policy = KMS permissions, near-always.
4. **Cross-account object ownership (legacy ACL trap):** in older setups, an object PUT by account B into account A's bucket could be *owned by B*, and A's bucket policy couldn't grant what A doesn't own. Modern fix: Object Ownership = Bucket owner enforced (ACLs disabled). If ACLs aren't disabled, this ghost still walks.
5. **Permissions boundary or SCP:** an Organizations Service Control Policy or an IAM permissions boundary silently caps what the identity's policy can grant — the policy says allow, the effective permission is no. Check with the **IAM Policy Simulator** or, better, the real request's failure detail.
6. **Wrong credentials in the runtime:** the code isn't using the role you think — an env-var access key overrides the instance/task role in the SDK credential chain. `aws sts get-caller-identity` from the actual runtime settles who you *are* before debugging what you *may do*.
7. **Block Public Access / requester expectations:** if the access was supposed to be public/presigned, BPA settings or an expired presigned URL explain it.

Method summary for the interviewer: identify the *actual* principal (`get-caller-identity`), then evaluate: SCP → boundary → identity policy (resource ARN with `/*`!) → bucket policy denies → KMS → ownership. CloudTrail's `errorMessage` on the denied event frequently names the failing policy type outright."

---

## Q39. You must serve private S3 content (user uploads) to your web app's users securely. Compare the approaches.

**Answer:**

"Never make the bucket public — the standard patterns:

1. **Presigned URLs (the default answer):** the backend, using its own IAM permissions, generates a time-limited signed URL for a specific object; the browser fetches directly from S3. No proxying load on your servers, per-object and per-user control (your app decides who gets a URL), works for uploads too (presigned PUT — which also offloads large uploads from your API). Caveats: the URL is bearer-token-like until expiry (keep expiries short — minutes), and the URL is only as good as the credentials that signed it (URLs signed by temporary role credentials die when the session expires, a classic 'presigned URL stopped working' gotcha).
2. **CloudFront + OAC (+ signed URLs/cookies if per-user):** bucket allows only the CloudFront distribution (Origin Access Control); public-ish shared content just works with caching and custom domains; per-user restriction uses **CloudFront signed URLs or signed cookies** (cookies shine when a user needs access to *many* objects, e.g., all segments of a video stream — you can't practically presign hundreds of URLs). This is the right answer at scale or when you want CDN performance, WAF, and a custom domain.
3. **Proxy through the app** (app reads S3, streams to user): maximum control (on-the-fly authz, transformation), but your compute pays for every byte — acceptable for small/rare files, an anti-pattern for media.

Decision line to give: one-off downloads and uploads → S3 presigned URLs; content delivery at scale with user-level auth → CloudFront + OAC + signed cookies; and in all cases Block Public Access stays on and the bucket policy enforces the single intended access path."

---

## Q40. An EC2-hosted database (self-managed, on EBS) has gotten mysteriously slow. Storage is suspected. How do you prove or disprove EBS as the bottleneck?

**Answer:**

"Storage slowness has exactly three suspects: the volume's limits, the *instance's* limits, or it's not storage at all. Evidence-based walk:

1. **What is the volume entitled to?** Type and provisioned figures: gp3 baseline 3,000 IOPS/125MBs unless raised; gp2 = 3 IOPS per GB with burst — a 100GB gp2 has a 300 IOPS baseline and a burst bucket. Check **`BurstBalance`** in CloudWatch: if it decays to zero and latency spikes track it, that's the smoking gun — the volume literally ran out of burst credits. Fix: migrate to gp3 (online, no downtime) and provision what the workload needs.
2. **Is the volume maxed?** CloudWatch `VolumeReadOps+VolumeWriteOps` vs provisioned IOPS, `VolumeThroughputPercentage`, and queue depth (`VolumeQueueLength`) — sustained high queue with ops pinned at the limit = volume saturation. On-host, `iostat -x 1`: `%util` near 100 with high `await` confirms the device view; `await` (per-IO latency) around 1–4ms is normal for SSD-class EBS, tens of ms says throttling or saturation.
3. **Is the *instance* the cap?** Every instance type has an **EBS bandwidth/IOPS ceiling** (and smaller instances have *burstable* EBS performance with their own micro-credit system). A giant io2 volume on a small instance is capped by the instance. CloudWatch/ethtool EBS-allowance-exceeded metrics expose this. Fix: bigger or storage-optimized instance, or spread across volumes only if the instance cap isn't the binding one.
4. **IO pattern mismatch:** databases do small random IO — an st1/sc1 (HDD, throughput-optimized) volume under a database collapses on random IOPS by design; likewise huge sequential scans can hit MB/s limits while IOPS look fine. Match volume type to pattern.
5. **Rule out the impostors:** memory pressure causing cache misses (suddenly the same queries do 10× the physical reads — the *workload* on disk grew, disk didn't shrink), a recently restored-from-snapshot volume **lazy-loading** blocks (first-touch latency, fixed by initialization or Fast Snapshot Restore), or a query plan regression. `iostat` showing modest load with slow queries = the problem is above the storage.

Deliverable of the analysis: either 'volume limits reached — resize/retype,' 'instance EBS cap reached — resize instance,' or 'storage is fine — look at the database.'"

---

## Q41. RDS failed over at 3 AM. Some applications recovered instantly; one was down for 45 more minutes until restarted. Explain what happened during failover and why that app stayed broken.

**Answer:**

"**What Multi-AZ failover does:** the standby is promoted and RDS updates the **DNS record** behind the endpoint name to point at the new primary. It does *not* change the endpoint name and it does not push anything to clients — recovery is entirely dependent on clients re-resolving DNS and re-establishing connections. Well-behaved apps see connections drop, reconnect, re-resolve, done in ~1–2 minutes.

**Why one app stayed down — the classic trio:**
1. **DNS caching in the runtime.** The JVM is the canonical offender: with default/legacy settings (`networkaddress.cache.ttl` = forever under a security manager, or long) it caches the *old IP* indefinitely — the app diligently reconnects to a dead primary's address for 45 minutes. Fix: set JVM DNS TTL to ~10–30s. Any runtime or hand-rolled resolver caching can do this.
2. **Connection pools holding corpses.** Pools full of TCP connections to the old primary that die *slowly* (no RST because the old host vanished; connections sit until TCP keepalive/timeouts). Without **connection validation** (test-on-borrow / validation query) and sane max-lifetime settings, the app keeps handing out dead connections. Fix: pool validation, `maxLifetime` a few minutes, aggressive socket timeouts.
3. **No reconnect/retry logic:** app treated connection failure as fatal and sat in a broken state rather than rebuilding.

**Prevention checklist:** JVM/runtime DNS TTL configured; pool validation + max connection lifetime; retry with backoff on transient DB errors; **RDS Proxy** as the systemic fix — it holds the client-side connections steady and re-points to the new primary itself, shrinking app-visible failover dramatically; and *test failover on purpose* (`aws rds reboot-db-instance --force-failover`) in non-prod, because failover behavior you've never rehearsed is behavior you don't actually have. Also verify Multi-AZ Cluster vs classic Multi-AZ — the cluster variant fails over in seconds and gives readable standbys, and know that failover triggers an alarmable RDS event."

---

## Q42. Read queries are slowing the primary RDS instance. Someone proposes "just add a read replica." What must the team understand before and after doing that?

**Answer:**

"Read replicas work — with three eyes-open realities:

1. **Replication lag is a data-consistency feature of your app now.** Replicas are **asynchronous**; they trail the primary by milliseconds normally, and by seconds-to-minutes under write bursts, long transactions, or an undersized replica. Any read-after-write flow (user saves, next page reads) served from a replica can show *stale data* — user edits profile, refresh shows the old one. The application must be partitioned: **reads that must see the latest write go to the primary; read-only/analytics/list-page traffic goes to replicas.** This is an application code change, not a checkbox — the ORM/datasource needs read/write routing, and monitoring must include `ReplicaLag` with alarms.
2. **The replica needs to be sized like it means it,** typically the same class as the primary: an undersized replica lags perpetually and can never catch up, which is worse than no replica. Replication also adds some load on the primary.
3. **A replica is not HA.** If the *primary* dies, replicas don't auto-promote (in RDS classic; Aurora differs). Multi-AZ remains the availability answer; replicas are the scalability answer; production usually wants both. (Worth saying: replica *promotion* is a valid manual DR tool, and cross-region replicas are the standard DR-region pattern.)

And the step-zero senior move: before scaling hardware, check whether the reads are slow because they're *bad* — Performance Insights top-SQL, missing indexes, N+1 query patterns, absent caching (ElastiCache for hot reads often beats a replica for cost and latency). Adding replicas to serve unindexed queries is renting a bigger room for the mess."

---

## Q43. A bucket must be protected against accidental deletion and ransomware-style overwrites, with old data moving to cheaper storage automatically. Design it.

**Answer:**

"Layered design:

1. **Versioning ON.** Every overwrite creates a new version; every delete just adds a *delete marker* — the data is recoverable by removing the marker or reading old versions. Versioning is the foundation; nothing else works without it.
2. **MFA Delete or, more practically, Object Lock** for the ransomware/insider case: versioning alone doesn't stop an attacker with credentials from deleting *versions*. **Object Lock in Compliance mode** (WORM) makes versions undeletable for a retention period *by anyone including root* — the actual ransomware-resistant control, also the answer to regulatory retention (GxP-style environments know this well). Governance mode is the softer variant with an override permission.
3. **Deny-based bucket policy guardrails:** explicit Deny on `s3:DeleteBucket`, and on `s3:PutBucketVersioning` (so nobody suspends versioning), except for a break-glass role. Block Public Access on, obviously.
4. **Lifecycle rules for cost:** current versions transition Standard → Standard-IA at 30 days → Glacier Flexible/Deep Archive per access pattern; **noncurrent versions** transition faster and *expire* after N days (bounding the cost of keeping versions); plus `AbortIncompleteMultipartUpload` cleanup (silent cost leak). If access patterns are unknown, Intelligent-Tiering does the moving automatically for a small monitoring fee.
5. **Replication as the last line:** Cross-Region (or same-region) Replication to a bucket in a *different account* with different credentials — an attacker who owns account A still can't touch the replica in account B. Different-account backup is the real ransomware answer at the architecture level.
6. **Observability:** CloudTrail data events (or at least management events) on the bucket, EventBridge alarm on unusual delete volume.

The one-line summary: versioning makes mistakes recoverable, Object Lock makes malice ineffective, lifecycle keeps it affordable, cross-account replication survives full account compromise."


---

# PART 8 — Athena & AWS Glue (Analytics)

## Mental Model

**Athena** = serverless SQL directly over files in S3. No cluster, no loading: you point it at data via a table definition and it scans the files at query time. **Pricing is per terabyte scanned** — which makes Athena the rare service where *performance optimization and cost optimization are the same activity*: scan fewer bytes, pay less, finish faster. The three levers, in order of impact:
1. **Partitioning** — organize S3 keys as `s3://bucket/data/year=2026/month=07/day=09/...`; queries filtering on partition columns skip whole directories. Unpartitioned data = full scan every query.
2. **Columnar formats** — **Parquet/ORC** store data by column with min/max statistics per block, so `SELECT two columns WHERE x > 5` reads only those columns and skips non-matching blocks. Converting CSV/JSON → Parquet routinely cuts scan (and cost) by 10–100×.
3. **Compression and file sizing** — compressed data scans cheaper; and S3 GETs per file mean **many tiny files are poison** (a million 10KB files spends the query in request overhead). Target ~128MB–1GB files; compact small files.

**Where do table definitions live? The Glue Data Catalog** — the metastore that maps "table name" → schema + format + S3 location + partitions. Athena, Glue jobs, EMR, Redshift Spectrum all share it. Key operational fact: **the catalog only knows the partitions it's been told about.** New data landing in a new `day=10/` prefix is invisible to Athena until the partition is registered — via `MSCK REPAIR TABLE` (slow, scans everything), explicit `ALTER TABLE ADD PARTITION` (fast, precise), **partition projection** (Athena computes partition locations from a pattern — no registration ever needed; the modern answer for date-partitioned data), or a Glue **crawler**.

**Glue** = serverless ETL + the catalog:
- **Crawlers** scan S3/JDBC sources, infer schemas, and create/update catalog tables (convenient; also a common source of schema surprises when inference guesses differently than you'd like).
- **Glue jobs** = managed Spark (or lightweight Python shell) for transform work — the canonical job being "read raw CSV/JSON, clean, write partitioned Parquet." Capacity in DPUs; job bookmarks track already-processed data for incremental runs.
- Typical pipeline you should be able to draw: raw zone (S3) → crawler catalogs it → Glue Spark job transforms → curated zone (S3, Parquet, partitioned) → catalog table → Athena/BI queries.

---

## Q44. An Athena query that "should" return yesterday's data returns nothing — or returns zero rows for new dates — while old dates work. What's wrong?

**Answer:**

"Old partitions work, new ones don't = **the partition metadata is stale**. The files are in S3, but the table's partition list in the Glue Catalog doesn't include the new date, so Athena never looks at that prefix. Verification: `SHOW PARTITIONS table` — yesterday won't be listed; and an `aws s3 ls` on the expected prefix confirms the data physically exists (also catching the other classic: the writer put data at a *slightly different* path — `dt=2026-07-08` vs `date=2026/07/08` — partition column name and path format must match the table definition exactly).

**Fixes, worst to best:**
1. `MSCK REPAIR TABLE` — re-scans and registers everything; works, but slow and expensive on big tables; fine as a one-off.
2. `ALTER TABLE ADD PARTITION (dt='2026-07-08') LOCATION 's3://.../dt=2026-07-08/'` — precise and fast; automatable in the pipeline (whoever writes the data registers the partition — Glue jobs can do this natively).
3. **Partition projection** — define in table properties that `dt` is a date ranging from X to NOW with a given format; Athena *computes* partition locations and never consults a partition list again. No registration step to forget, and faster query planning on tables with many thousands of partitions. This is the answer that ends this class of incident permanently for date-style partitions.
4. A scheduled Glue crawler also works but adds cost/latency and can mutate schemas unexpectedly — I'd prefer projection or explicit ADD PARTITION for pipelines I control.

If it's not partitions: zero rows can also mean schema/format mismatch (SerDe reading Parquet as CSV returns garbage or nothing), or the query's filter written so it doesn't match the partition column type (string vs date comparisons)."

---

## Q45. Your team's Athena bill has grown huge and analysts complain queries are slow. Same fix for both — what do you do?

**Answer:**

"Athena's per-TB-scanned pricing means I attack *bytes scanned*, and speed follows automatically:

1. **Audit the workload:** every query's `Data scanned` is in the console/history and CloudWatch. Rank by total bytes: typically a handful of dashboards/scheduled queries dominate. Also check for `SELECT *` habits and missing partition filters.
2. **Convert to Parquet** (via a Glue job or Athena CTAS): columnar + compressed + statistics. This alone is routinely a 10–100× scan reduction if the data is CSV/JSON today. CTAS (`CREATE TABLE AS SELECT ... STORED AS PARQUET PARTITIONED BY ...`) is the low-effort conversion path.
3. **Partition on what people filter by** (nearly always date, sometimes tenant/region) and make analysts *use* the partition column in WHERE clauses — an unpartition-filtered query on a partitioned table still scans everything. Partition projection to keep it maintenance-free.
4. **Fix the small-files problem:** streaming pipelines writing a file per minute create millions of tiny objects; compact them (Glue job or periodic CTAS rewrite) toward 128MB+ — this is often the hidden reason 'Parquet didn't help.'
5. **Guardrails:** **workgroups** with per-query and per-workgroup **data-scanned limits** (a runaway `SELECT *` gets cut off at, say, 1TB), separate workgroups per team for cost attribution, and query result reuse for repeatedly-run dashboard queries.
6. Precompute where warranted: heavy daily aggregations become a scheduled CTAS/Glue job writing a summary table; dashboards then query the small summary, not the raw events.

The framing to give the interviewer: in Athena, cost and latency are the same knob — bytes scanned — and the hierarchy is format > partitioning > file sizing > query hygiene > guardrails."

---

## Q46. A Glue ETL job that "worked for months" now fails intermittently / runs 5× longer. What do you look at?

**Answer:**

"Glue is managed Spark, so this is Spark troubleshooting with Glue-specific handles:

1. **What changed is almost always the data:** volume growth (month-end spikes), a schema drift (new/renamed column from the source — crawler picked it up, job logic didn't; or worse, inference changed a column's type when a file full of nulls arrived), or an explosion of small input files. Compare this run's input stats to a good run's.
2. **Read the right logs:** Glue job runs expose driver/executor logs in CloudWatch and (if enabled) the **Spark UI** — the difference between guessing and knowing. In the Spark UI: which stage is slow, is one task taking forever while others finish (**data skew** — one huge key in a join/groupBy lands on one executor; fix by salting keys or broadcast-joining the small side), are executors dying with OOM?
3. **OOM / `No space left`:** executor memory exceeded — bump worker type (G.1X → G.2X) or number of workers, but first check for skew and for `collect()`-style anti-patterns pulling data to the driver; also huge shuffles from unnecessary wide transformations.
4. **Intermittent failure patterns:** source contention (JDBC source database throttling the job at busy hours), S3 throttling on hot prefixes at high parallelism, **concurrent runs colliding** (two runs writing the same output path — check job concurrency settings and bookmark behavior), or spot-style capacity issues on huge parallel jobs.
5. **Job bookmarks:** if incremental processing suddenly reprocesses everything (duration blow-up) or skips data (missing rows), bookmarks were reset or the source layout changed in a way bookmarks can't track. Verify bookmark state and the job's `transformation_ctx` usage.
6. **Cost/duration guardrails after fixing:** set job timeout, alarms on duration and DPU-hours, and if input keeps growing, enable auto-scaling workers rather than a fixed count.

Senior one-liner: a Spark job that degrades without a code change degraded because the *data* changed — profile the input before touching the code."

---

## Q47. Design a pipeline: application logs land in S3 as gzipped JSON continuously; business wants SQL dashboards with data available within ~an hour, cheaply. Walk through it.

**Answer:**

"Lake pattern, right-sized:

1. **Raw zone stays as-is:** `s3://logs-raw/app=X/dt=YYYY-MM-DD/hour=HH/...`, gzipped JSON. Immutable, cheap, replayable. Lifecycle policy to IA/Glacier after N days.
2. **Hourly Glue job (or Athena CTAS/INSERT INTO on a schedule):** reads the latest raw hour (job bookmarks or event-driven trigger from S3/EventBridge), flattens/cleans the JSON, writes **Parquet, partitioned by dt/hour**, compacted to healthy file sizes, into `s3://logs-curated/...`. For modest volumes, an Athena-based `INSERT INTO curated SELECT ... FROM raw WHERE dt/hour = latest` scheduled via EventBridge + Step Functions is even cheaper and simpler than Spark — I'd start there and reach for Glue Spark only when transformation complexity or volume demands it.
3. **Catalog:** curated table defined once with **partition projection** on dt/hour — no crawler needed on a stable schema, no partition-registration failures ever. A crawler runs only on the *raw* zone if its schema genuinely drifts.
4. **Serving:** Athena workgroup for the dashboard tool (QuickSight or Grafana Athena plugin), with data-scanned limits as guardrails; heavy repeated aggregations get a small pre-aggregated summary table refreshed by the same hourly cycle.
5. **Orchestration & ops:** EventBridge schedule (or S3-event-driven) → Step Functions running transform → partition registration is unnecessary (projection) → success/failure alarms; DLQ-style handling for malformed records (bad JSON rows routed to a quarantine prefix rather than failing the whole batch).

Why this shape: object counts and scan bytes stay controlled (Parquet + partitions + compaction), latency is bounded by the schedule (~hourly meets the requirement — if they later say 'minutes,' the answer evolves toward Kinesis Data Firehose writing Parquet directly with dynamic partitioning), and the whole thing is serverless — cost scales with data, not with idle clusters."

---

# PART 9 — IAM (Identity & Access Management)

## Mental Model

IAM answers one question on every AWS API call: **is this *principal* allowed to perform this *action* on this *resource* under these *conditions*?**

Nouns:
- **Principals:** users (humans, discouraged for apps), **roles** (assumable identities with *temporary* credentials — the correct mechanism for everything non-human and most humans via SSO), services, federated identities.
- **Policies:** JSON documents of `Effect` (Allow/Deny), `Action`, `Resource`, `Condition`. **Identity-based** (attached to a principal) vs **resource-based** (attached to the resource — S3 bucket policies, KMS key policies, SQS queue policies, Lambda resource policies; these can grant access to *other accounts'* principals, which identity policies alone cannot).
- **Roles + STS:** `sts:AssumeRole` exchanges your identity for temporary credentials of a role, gated by the role's **trust policy** (who may assume it). Two policies on every role, and they answer different questions: trust policy = *who can wear this hat*, permission policy = *what the hat allows*.

**The evaluation algorithm (memorize this order):**
1. Start at implicit **Deny** (nothing allowed by default).
2. An **explicit Deny anywhere wins over everything** — no allow overrides it.
3. Otherwise, need an **Allow** from: identity policy OR (same-account) resource policy. Cross-account requires *both* sides to allow.
4. The result is further capped by **SCPs** (Organizations guardrails — they never grant, only bound), **permissions boundaries** (per-identity cap), and **session policies**.

**Credential hygiene facts:** EC2 gets credentials via the **instance profile/metadata service** (use IMDSv2); ECS via the **task role**; Lambda via the **execution role**; EKS via **IRSA/Pod Identity** — long-lived access keys in code/env are the anti-pattern every one of these mechanisms exists to eliminate. The SDK **credential chain** (env vars → config → container/instance credentials) explains many mysteries: an old env var silently outranks the role you configured.

---

## Q48. Application on EC2/ECS gets `AccessDenied` calling an AWS API "even though the role has the permission." Give your complete diagnostic method.

**Answer:**

"Two questions in strict order: *who am I actually?* then *what denies me?*

**Step 1 — establish the real identity:** from inside the workload, `aws sts get-caller-identity`. Half of these tickets end here: the process isn't using the role you think — a leftover `AWS_ACCESS_KEY_ID` env var or a `~/.aws/credentials` file outranks the instance/task role in the SDK chain; or on ECS the permission was added to the **execution role** instead of the **task role** (or vice versa — app-runtime calls need the *task* role; image pulls and secrets need the *execution* role); or the pod isn't actually getting the IRSA role (service account annotation missing).

**Step 2 — read the actual error and CloudTrail:** the denied event in CloudTrail shows the exact action attempted, the resource ARN, and often *which policy type* denied ('with an explicit deny in a service control policy'). The action name matters precisely — the code may call `s3:ListBucket` while you granted `s3:GetObject`, or `kms:Decrypt` is the hidden extra requirement on encrypted resources.

**Step 3 — walk the evaluation stack:**
1. **Explicit Deny hunt** first (it trumps everything): SCPs from the Organization, deny statements in the resource policy (VPC-endpoint-only, TLS-only, IP conditions), deny in any attached policy.
2. **Resource ARN precision:** bucket vs `bucket/*`, region/account in the ARN, wildcards not matching what you think.
3. **Condition mismatches:** the policy allows only with certain tags (ABAC), from certain VPC endpoints, or with MFA — and the request doesn't satisfy the condition.
4. **Permissions boundary** on the role capping it below the policy.
5. **Cross-account:** both the identity policy *and* the resource policy must allow; and for role-chaining, each trust policy in the chain.

**Tools to name:** IAM **Policy Simulator** for what-if evaluation, **CloudTrail** for what actually happened, IAM **Access Analyzer** for unintended access and for *generating* least-privilege policies from actual usage. Guessing at JSON is what you do only after these tools have narrowed it."

---

## Q49. Account B's application must read objects from Account A's bucket. Design the access and explain the classic failure.

**Answer:**

"**Cross-account S3 needs BOTH sides to say yes:**
1. **Account A (resource side):** bucket policy allowing the specific principal — best practice, the *role* in account B (`arn:aws:iam::B:role/app-role`) — actions `s3:GetObject` (and `s3:ListBucket` if listing) on the bucket and `bucket/*` respectively.
2. **Account B (identity side):** the app role's policy allowing the same actions on account A's bucket ARNs. Cross-account, an identity policy alone or a resource policy alone is insufficient — this dual requirement *is* the model.

Alternative pattern (often cleaner): account A publishes a **role** with the S3 permissions and a trust policy allowing account B; B's app performs `sts:AssumeRole` and acts *as* an account-A identity — then only A's side needs S3 permissions, and A retains full control/audit of what the role can do. Choose bucket-policy for simple read paths; role-assumption when access is broader or A wants the control plane.

**The classic failures:**
1. **KMS forgotten:** objects are SSE-KMS encrypted; B gets `AccessDenied` on GetObject despite perfect S3 policies because `kms:Decrypt` isn't granted — required in *both* the key policy (A side) and the role's policy (B side). This is the #1 cross-account S3 ticket in real life.
2. **Object ownership (legacy):** objects written by a third account with ACLs enabled aren't owned by A, so A's bucket policy can't share them — fixed by Object Ownership = bucket owner enforced.
3. **The confused deputy angle** for the senior mention: when granting *services or vendors* access via roles, the trust policy should require an **ExternalId** / use `aws:SourceArn`/`aws:SourceAccount` conditions so a third party can't trick the service into using its access on someone else's behalf."

---

## Q50. Security asks you to implement least privilege for a role that currently has `AdministratorAccess`, without breaking the application. Method?

**Answer:**

"Least privilege by *observation*, not by guesswork:

1. **Measure actual usage:** the app has been running — CloudTrail already recorded every API call the role made. **IAM Access Analyzer policy generation** consumes CloudTrail history and drafts a policy scoped to the actions/services actually used; **Access Advisor** on the role shows last-accessed per service to confirm what's dead. This turns 'what does it need?' from an interview of developers into a query.
2. **Draft with resource scoping:** actions from step 1, resources narrowed to the actual ARNs (the generated policy needs this tightening by hand — generation gets actions right, resources broad). Add condition keys where natural (region, tags).
3. **Deploy in observe-first fashion:** attach the new restrictive policy *alongside* admin? No — better: replace admin with the new policy in a lower environment first and exercise the app's full lifecycle (deploys, scale events, failure paths — the rare paths are what audits miss: the role only calls `sns:Publish` when something breaks). In production, be ready to diff CloudTrail `AccessDenied` events immediately after the switch; each denial names the exact missing action — tighten-iterate quickly.
4. **Guard the future:** a **permissions boundary** or SCP prevents privilege from silently re-expanding; alerting on `AccessDenied` spikes for the role catches regressions; periodic Access Advisor reviews to prune.
5. Sequence honestly: communicate a rollback plan (the old policy is one attach away) and do it in a change window — least-privilege migrations fail socially, not technically, when the first surprise denial has no fast path to fix.

The interview-worthy summary: least privilege isn't written, it's *derived* — CloudTrail is the source of truth for what an identity really needs, and AccessDenied events are the feedback loop for what you missed."

---

## Q51. Explain the difference between an IAM role's trust policy and permission policy with a failure scenario for each.

**Answer:**

"A role has two independent gates:

- **Trust policy (assume role policy):** *who is allowed to become this role* — the principals permitted to call `sts:AssumeRole`. It grants zero AWS permissions itself.
- **Permission policies:** *what the role can do once assumed*.

**Trust policy failure scenario:** you create a role for ECS tasks with perfect S3 permissions, but tasks fail to start or the app can't get credentials, and manual `aws sts assume-role` returns `AccessDenied: not authorized to perform sts:AssumeRole`. Cause: the trust policy doesn't list the right principal — for an ECS task role the trusted service must be `ecs-tasks.amazonaws.com`; for Lambda `lambda.amazonaws.com`; for EKS IRSA the OIDC provider with the right sub condition (a single-character namespace/serviceaccount typo in the IRSA condition is a legendary time sink); for cross-account, account B's role/user ARN. Error happens at *assumption time*, before any real API call — that timing is the diagnostic tell.

**Permission policy failure scenario:** assumption succeeds (`get-caller-identity` shows the role's assumed ARN), the app runs, and specific API calls fail with `AccessDenied` — the role exists, you're wearing it, but the hat doesn't allow `s3:PutObject` on that ARN. Error happens at *action time*.

So the two-question triage: does `sts get-caller-identity` show the role? **No** → trust side (or credential chain). **Yes, but actions fail** → permission side (then the full evaluation walk: explicit denies, resource ARNs, conditions, boundaries, SCPs). Keeping these two failure timings distinct instantly halves any role-related debugging session."


---

# PART 10 — Cross-Service "War Story" Scenarios
### The composite questions that separate senior candidates

These combine multiple domains — exactly the format of second-round scenario interviews. Practice narrating these out loud.

---

## Q52. THE BIG ONE: "A user reports your site is down. Nothing else is known. Walk me through your entire investigation, from their browser to your database."

**Answer (narrate as a request trace):**

"I follow the request path and bisect at every hop. The stack is: user → DNS → CloudFront → WAF → ALB → targets (ECS/EC2) → database.

**1. Scope first (2 minutes, before touching anything):** is it one user, one region, or everyone? Check the monitoring dashboards and error-rate metrics — synthetic checks, ALB 5xx counts, CloudFront error rates. 'One user' investigations and 'global outage' investigations are different jobs. Also: *what changed?* Recent deploys, DNS changes, certificate renewals, scaling events — the change log is the highest-yield diagnostic in existence.

**2. DNS layer:** `dig site.com` from outside — does the name resolve, and to what? Wrong/missing answer → Route 53 zone problem, expired domain, health-check-driven failover pointing somewhere unexpected, or a recent record change still propagating (TTL). Resolves fine → next hop.

**3. TLS/edge:** `curl -v https://site.com` — does the TLS handshake complete? Certificate errors (expired/wrong name/chain) live here. Handshake OK but error status → note WHO issued the error page: CloudFront error (`x-cache`, `server: CloudFront` headers, 502/503 from CloudFront) means the origin fetch failed; a WAF block returns 403 with a CloudFront request ID — check WAF metrics for a rule suddenly blocking legit traffic (a new managed rule update or rate limit).

**4. Origin/ALB:** hit the ALB directly (if reachable) to bisect CloudFront-vs-origin. ALB metrics tell the story: `HTTPCode_ELB_5XX` vs `HTTPCode_Target_5XX` (ALB's fault vs app's fault), `HealthyHostCount` (zero healthy = health check or app-down problem), and target response times (soaring latency → timeouts → 504s).

**5. Compute:** targets unhealthy → ECS service events / instance status. Tasks crash-looping (bad deploy — roll back first, diagnose after), OOM kills, failed image pulls, or hosts down. If a deploy correlates: **roll back now** — restoring service beats root-causing, always state this priority.

**6. Data layer:** app logs showing DB connection errors/timeouts → RDS: failover event in progress? `DatabaseConnections` maxed (connection storm)? CPU/IOPS saturated? Storage full (a classic total-outage cause — RDS goes read-only)?

**7. Communicate throughout:** status updates at each bisection, and once service is restored, the timeline feeds the postmortem.

The meta-answer the interviewer wants: bisect along the request path with evidence at each hop, prioritize restoration over diagnosis, and treat 'what changed recently' as the first question, not the last."

---

## Q53. After migrating a service into private subnets "for security," three things broke: image pulls, a third-party API integration, and CloudWatch logging. One root cause — explain all three.

**Answer:**

"Root cause: **the private subnets have no egress path** — no NAT Gateway route (or NAT exists but a route table association was missed). Everything that needed to *initiate outbound* connections died together, and the trio is diagnostic:

1. **Image pulls (ECR):** Fargate/nodes pull images over the network; no route to ECR endpoints → `CannotPullContainerError`.
2. **Third-party API:** outbound HTTPS to the internet → dead without NAT.
3. **CloudWatch logging:** the logs API is a *public AWS endpoint* — private instances reach it via NAT or via VPC endpoints; with neither, log delivery fails silently or with agent errors. People forget AWS services themselves are 'the internet' from a private subnet's perspective.

**Fix has two philosophies, and 'for security' argues for the second:**
- Quick: NAT Gateway in a public subnet + `0.0.0.0/0 → NAT` in the private route tables. Everything works, but all traffic (including AWS-bound) transits NAT — cost per GB, and the instances can reach the *entire* internet, which is a broad egress posture.
- Right for the stated goal: **VPC endpoints for the AWS services** — S3 gateway endpoint (free) + interface endpoints for `ecr.api`, `ecr.dkr`, `logs`, plus whatever else appears (STS, SSM, Secrets Manager, monitoring). AWS traffic never leaves the AWS network. Then the third-party API is the only true internet need: NAT with tight egress control, or an egress proxy/firewall allowing only that domain. Interface endpoints need **Private DNS enabled** and their **own security group allowing 443 from the VPC** — a missing endpoint-SG rule is the standard 'I added endpoints and it still fails' follow-up.

Bonus diagnostic credibility: how would I confirm? From an affected host: `curl -m 5 https://logs.<region>.amazonaws.com` times out; route table shows no 0.0.0.0/0; flow logs show outbound attempts with no return. Three symptoms, one route table."

---

## Q54. Latency for API calls doubled after "no changes." Metrics show backend processing time is flat. Where did the time go?

**Answer:**

"Backend flat + total doubled = the time is in the *path*, not the app. The suspects between client and handler:

1. **Prove the split:** ALB access logs record three durations per request — `request_processing_time`, `target_processing_time`, `response_processing_time` — and CloudFront logs record `time-taken` vs origin latency. These pin which segment grew. This is why access logs stay enabled *before* you need them.
2. **TLS/connection churn:** if keep-alive broke (config change on a proxy, client library update), every request pays fresh TCP+TLS handshakes — +1–3 RTTs each. Evidence: new-connection rate up in LB metrics, `ssl_handshake` counts up.
3. **DNS in the hot path:** resolver slowness or a TTL drop causing constant re-resolution adds latency per *connection*; the VPC resolver also rate-limits per ENI — heavy DNS chatter gets throttled and looks exactly like random added latency.
4. **Cross-AZ or path change:** an autoscaling/redeploy event redistributed targets so calls now cross AZs (adds ~0.5–1ms per hop — deadly for chatty request fans), or NLB cross-zone setting changed, or traffic silently moved from a VPC-endpoint path to a NAT path (or through an inspection appliance/Zscaler leg that's newly saturated).
5. **A dependency's latency counted outside 'backend processing':** if 'backend time' is app-measured around the handler but the connection *pool wait* happens before measurement, DB pool exhaustion shows up as invisible time. Check pool wait metrics.
6. **Retry inflation:** a downstream returning occasional errors triggers client retries — p50 flat, p95/p99 and *average* up. Always look at percentiles, not averages: 'latency doubled' as an average is often 'p99 went 10×,' which points at timeouts/retries rather than uniform slowdown.
7. **'No changes' is always false.** Deploy logs, config change history, CloudTrail for infra mutations, and AWS Health Dashboard for the platform's own events (AZ issues do happen). Something changed; the job is finding whose.

The takeaway line: flat backend time relocates the search to connections, DNS, network path, and retries — and hop-by-hop timing data (LB access logs, X-Ray traces) converts that from speculation to arithmetic."

---

## Q55. Design question: expose an internal API (running on ECS in private subnets) to (a) other internal VPCs, (b) a partner company, (c) the public — describe the layers you'd add for each audience.

**Answer:**

"Same service, three exposure levels, additive layers:

**(a) Internal VPCs:** internal (non-internet-facing) ALB in front of the ECS service; connectivity via Transit Gateway (route both directions + CIDR-based SGs) — or, if I want to share the API *without* extending network routing (no CIDR overlap concerns, no transitive access), **AWS PrivateLink**: put an NLB in front, create a VPC Endpoint Service, consumers create interface endpoints in their VPCs. PrivateLink is the answer when the consumer shouldn't get network *access*, just service access — that distinction is senior gold.

**(b) Partner company:** depends on the partnership's plumbing. Options ladder: PrivateLink cross-account (partner is on AWS — cleanest, private, no internet); site-to-site VPN/DX to the TGW (network-level integration, heavier); or public exposure hardened for one consumer — API Gateway with **mTLS** (partner certs), IP allowlisting via WAF, usage plans/throttling per key. Contracts usually push toward mTLS + allowlist on a dedicated custom domain.

**(c) Public:** full defense-in-depth stack: Route 53 → **CloudFront** (TLS at edge, DDoS absorption, caching where possible) → **WAF** (managed rules + rate limiting — deployed in count mode first, then block, to avoid breaking real users) → **API Gateway** (authn/authz — Cognito/JWT, request validation, throttling, usage plans) → **VPC Link** → internal NLB/ALB → ECS. Origin locked so it's unreachable except via the front door (custom header validation or VPC origins). Observability at every layer (access logs, WAF logs, X-Ray).

The structural point to land: the service itself never changes — audiences differ only in which *front door* you build, and each broader audience adds identity, rate control, and attack-surface layers rather than replacing the previous design."

---

## Q56. Rapid-fire lightning round — one-to-three-line answers you should be able to produce instantly.

**Q: Instance in public subnet, no public IP — can it reach the internet?**
No. IGW route exists but there's no public address to translate to. Assign a public/Elastic IP, or route via NAT.

**Q: Can a security group reference an SG in another VPC?**
Over VPC peering, yes (same region). Over Transit Gateway, no — use CIDRs.

**Q: Why does my ALB show healthy targets in only 2 of 3 AZs?**
The ALB was only given subnets in 2 AZs, or the third AZ's subnet ran out of IPs. ALBs need a subnet per AZ they serve.

**Q: S3 Gateway endpoint added, but instances still reach S3 via NAT — why?**
The endpoint attaches to specific *route tables*; the instances' route table wasn't selected. Also check the endpoint policy and that the request targets the same-region S3.

**Q: gp2 volume, everything slow every afternoon?**
Burst credit exhaustion under sustained IOPS — check BurstBalance; migrate to gp3.

**Q: Lambda can't reach the internet after adding it to a VPC?**
VPC Lambdas have no public path by design; they need private subnets with NAT (or VPC endpoints for AWS services). Never put a Lambda in a public subnet expecting internet — it has no public IP.

**Q: CNAME at the zone apex?**
Not allowed by DNS rules. Route 53's answer: Alias record.

**Q: Difference between 502 and 504 at the ALB?**
502: target sent a broken/closed response (often keep-alive mismatch). 504: target sent nothing within the idle timeout.

**Q: CloudFront cert must be in which region?**
us-east-1, regardless of where origins live.

**Q: How do targets know the real client IP behind an ALB? Behind an NLB?**
ALB: `X-Forwarded-For` header. NLB: source IP preserved natively (or Proxy Protocol v2 in some modes).

**Q: SQS message processed twice — is SQS broken?**
No — standard queues are at-least-once by contract. Consumers must be idempotent; check visibility timeout vs processing time.

**Q: ECS task can pull images but app gets AccessDenied on S3?**
Permissions are on the execution role; S3 calls need them on the **task role**.

**Q: MSCK REPAIR vs partition projection?**
MSCK scans and registers partitions (slow, repeated); projection computes them from a pattern (no registration, faster planning). Prefer projection for date-style partitions.

**Q: Explicit Allow in IAM vs explicit Deny in the bucket policy — who wins?**
Deny. Explicit deny beats everything, everywhere, always.

**Q: Multi-AZ vs read replica in one line each?**
Multi-AZ: synchronous standby for failover, serves no reads. Read replica: async copy for read scaling, lags, manual promotion.

---

# How to Study This Guide (read this last)

1. **First pass — mental models only.** Read every Part's Mental Model section back to back in one sitting. That's the skeleton; the 56 questions are just flesh on it.
2. **Second pass — active recall.** Read each *question*, close the guide, answer out loud for 2–3 minutes, then compare. The gap between what you said and what's written is your actual study list. This matters far more than a third reading.
3. **Narrate paths, not facts.** Senior interviews reward "let me trace the request" structure. Q10, Q52, and Q54 are templates — internalize their *shape* and you can improvise answers to scenarios this guide doesn't cover.
4. **Anchor to your real experience.** You've supported platforms on EKS: every SG/NACL, DNS, cert, IAM-role, and storage question here has happened on your clusters in some form. Prefacing answers with "I've seen a version of this when..." transforms textbook answers into senior answers.
5. **The universal troubleshooting spine** — if you memorize one thing:
   *Scope it → What changed? → Trace the path → Bisect with evidence (logs/metrics at each hop, never guesses) → Restore before root-causing → Prevent recurrence.*
   Every answer in this guide is that spine wearing different clothes.
