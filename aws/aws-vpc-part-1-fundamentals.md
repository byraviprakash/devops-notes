# AWS VPC — Part 1: Fundamentals

A VPC is your own isolated network inside an AWS Region. You choose its IP range, split it into public and private subnets, and control every path in and out with route tables, gateways and firewalls.

This part covers why a VPC exists, its building blocks, CIDR math, a hands-on build and common misconceptions. [Part 2](./aws-vpc-part-2-production.md) covers running VPCs in production.

## 1. Why a VPC exists

A VPC (Virtual Private Cloud) is a logically isolated network in one AWS Region. Your resources get private IPs from a range you choose, and nothing reaches them unless you open a path.

The three reasons it exists:

- **Isolation:** other AWS customers' workloads can't reach yours. Each VPC is its own routing domain.
- **Control:** you own the IP plan, the routing, and the firewalls (security groups and NACLs).
- **Segmentation:** you split one network into tiers (public web tier, private app and DB tiers) and into environments (dev, stage, prod).

**A common misunderstanding:** a VPC is not there to save public IPv4 addresses. Inside a VPC you use **private** ranges (RFC 1918: 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16), and thousands of customers use 10.0.0.0/16 at the same time with no conflict. Public IPv4 scarcity (about 4.3 billion addresses in total) is handled by private addressing plus NAT, and by IPv6.

**How it's done in real companies:** usually one VPC per environment, often in separate AWS accounts (accounts are the strongest isolation boundary). The IP plan is decided up front so VPCs never overlap. Overlapping ranges block peering and VPN connections later.

## 2. Core building blocks

The network diagram always nests the same way: AWS cloud → Region → VPC → Availability Zones → subnets, with an Internet Gateway, route tables and NAT gateways controlling traffic. Interviewers often ask you to draw it ([Part 2](./aws-vpc-part-2-production.md#7-production-reference-architecture) has the full diagram).

| Component | What it is | Key rules |
| --- | --- | --- |
| Region | A geographic area (us-east-2 = Ohio, us-east-1 = N. Virginia) | A VPC lives in exactly one Region |
| Availability Zone (AZ) | One or more isolated data centers inside a Region (us-east-2a, 2b, 2c) | AZs already exist. You don't create them, you place subnets in them |
| VPC | Your private network, defined by a CIDR block | Spans all AZs of its Region. Size /16 to /28. Secondary CIDRs can be added later |
| Subnet | A slice of the VPC CIDR | Lives in exactly **one** AZ. Subnet CIDRs can't overlap and must sit inside the VPC CIDR |
| Route table | Rules: "traffic for destination X goes to target Y" | Every subnet uses exactly one route table (the **main** route table unless you associate another). Every table has a built-in `local` route for the VPC CIDR |
| Internet Gateway (IGW) | The VPC's two-way door to the internet | One per VPC. Managed, redundant and auto-scaling, not a single physical device. Free; you pay for data transfer |
| NAT Gateway | Lets **private** instances make **outbound** connections (updates, external APIs) while blocking inbound ones | Sits in a **public** subnet with an Elastic IP. Hourly + per-GB charge. One per AZ for high availability |
| Security Group (SG) | Firewall on the instance / network interface | **Stateful**, allow rules only |
| Network ACL (NACL) | Firewall on the subnet | **Stateless**, allow and deny rules, evaluated in number order |

**What makes a subnet public?** Nothing about the subnet itself. A new subnet uses the VPC's main route table, which in a new VPC has only the local route, so the subnet starts private. It becomes public only when its route table has `0.0.0.0/0 → igw-xxxx` **and** its instances have a public IP or Elastic IP. Remove that route and the same subnet is private again. This is the most important idea in VPC networking.

**How subnets talk to each other.** Every subnet in a VPC can route to every other subnet through the built-in `local` route. No gateway is involved. Whether traffic is actually **allowed** is decided by security groups and NACLs.

**Private does not mean attack-proof.** A database in a private subnet has no direct internet path. But if a public instance is compromised, the attacker can reach the DB unless the DB's security group allows only the app tier on the DB port.

**IGW vs NAT gateway, the right way to say it:**

- **IGW** = internet access both ways for public subnets.
- **NAT gateway** = outbound-only internet access for private subnets. The NAT gateway itself lives in a public subnet. The private subnet's route table sends `0.0.0.0/0 → nat-xxxx`.
- **Public → private traffic inside the VPC** uses neither. It goes over the `local` route.

## 3. CIDR and subnet math

CIDR (Classless Inter-Domain Routing) writes an IP range as `address/prefix`. The prefix says how many of the 32 bits are fixed, and the remaining bits are free for hosts. [cidr.xyz](https://cidr.xyz) is a handy visual calculator.

```math
\text{total IPs} = 2^{(32 - \text{prefix})} \qquad \text{usable in an AWS subnet} = 2^{(32 - \text{prefix})} - 5
```

| CIDR | Total IPs | Usable in an AWS subnet | Typical use |
| --- | --- | --- | --- |
| /16 | 65,536 | 65,531 | Whole VPC (the largest AWS allows) |
| /20 | 4,096 | 4,091 | Large subnet, e.g. EKS worker nodes |
| /24 | 256 | 251 | Standard subnet |
| /25 | 128 | 123 | Half of a /24 |
| /28 | 16 | 11 | Smallest subnet AWS allows |
| /32 | 1 | not allowed as a subnet | A single host, used in security group rules ("my IP only") |

### The 5 reserved IPs (per subnet)

AWS reserves 5 addresses in **every subnet**, not once per VPC. For subnet 10.0.0.0/24:

| Address | Reserved for |
| --- | --- |
| 10.0.0.0 | Network address |
| 10.0.0.1 | VPC router |
| 10.0.0.2 | Amazon DNS resolver (VPC base + 2) |
| 10.0.0.3 | Reserved for future use |
| 10.0.0.255 | Broadcast (AWS doesn't support broadcast, but reserves it) |

So the first usable address in this subnet is **10.0.0.4**, and instances get addresses from .4 to .254.

### Pick a private range

The first octets aren't arbitrary. Use an RFC 1918 private range for every VPC:

- 10.0.0.0/8
- 172.16.0.0/12 (the default VPC uses 172.31.0.0/16)
- 192.168.0.0/16

A common mistake is typing **192.0.0.0/16** when you mean 192.168.0.0/16. That range is **not** private: it contains IETF-reserved and publicly routable addresses. AWS lets you use publicly routable ranges in a VPC, but traffic to real internet hosts in that range would be routed inside your VPC instead.

### Pitfall: running out of space for subnets

If the VPC is 10.0.0.0/**24** and the first subnet is also 10.0.0.0/24, that subnet takes all 256 addresses. Creating a second subnet then fails. AWS gives one of two errors:

- **An overlap error** means the range clashes with a subnet you already have.
- **A "not within the VPC CIDR" error** means the range falls outside the VPC (for example, 10.0.1.0/24 in a 10.0.0.0/24 VPC).

A subnet's CIDR **can't be edited**. You must delete and recreate the subnet, or add a **secondary CIDR** to the VPC. Plan the split up front instead:

| Plan | Subnet 1 | Subnet 2 |
| --- | --- | --- |
| Small VPC 10.0.0.0/24, split in half | 10.0.0.0/25 (.0 to .127) | 10.0.0.128/25 (.128 to .255) |
| Recommended: VPC 10.0.0.0/16 | 10.0.1.0/24 | 10.0.2.0/24 |
| VPC 192.168.0.0/16 | 192.168.0.0/24 (public) | 192.168.1.0/24 (private) |

With /24 subnets, step through the **third** octet (x.x.0.0/24, x.x.1.0/24, x.x.2.0/24). Changing the last octet doesn't help, because a /24 always starts at .0.

**Rule of thumb:** give the VPC a /16 and hand out /24s or /20s by stepping the third octet. Keep a written IP plan so VPCs in different environments and accounts never overlap.

## 4. Hands-on: build a VPC in the console

This lab proves one point: **an instance in a VPC is unreachable until there's a full path from the internet to it.** That path is IGW attached → route `0.0.0.0/0 → IGW` → subnet associated with that route table → public IP on the instance → security group rule. Region used: us-east-2 (Ohio).

1. **Look at the default VPC first.** In the VPC console, the unnamed VPC is the **default VPC**, which AWS creates in every Region. It uses 172.31.0.0/16, has one /20 public subnet per AZ and an IGW attached, and its main route table already has `0.0.0.0/0 → IGW`. That's why EC2 "just works" there. Use it for quick tests, not for production.
2. **Create VPC → "VPC only".** "VPC and more" is a wizard that builds the VPC, subnets, route tables, IGW and optional NAT in one go. Build it piece by piece once so you understand each part.
    - Name `test-vpc`, IPv4 CIDR `10.0.0.0/16`, no IPv6.
    - **Tenancy: Default.** Tenancy decides whether your **EC2 instances** run on shared hardware or on hardware dedicated to your account. It has nothing to do with network devices. Dedicated tenancy adds a per-Region fee (about $2 per hour whenever any Dedicated Instance runs in that Region) on top of the instance charges, so keep Default unless compliance requires it.
3. **Create subnets.** `test-public-subnet` in `us-east-2a` with `10.0.1.0/24`, and `test-private-subnet` in `us-east-2b` with `10.0.11.0/24`.
4. **Create the Internet Gateway** `test-igw`. A new IGW is *detached*. Actions → **Attach to VPC** → `test-vpc`.
5. **Create route table** `test-public-rt` in `test-vpc`.
    - Routes → Edit routes → add `0.0.0.0/0` → target `test-igw`. `0.0.0.0/0` is the default route: "any IPv4 destination not matched by a more specific route."
    - Subnet associations → associate `test-public-subnet`. The private subnet stays on the main route table, which has only the `local` route.
6. **Public subnet settings → Enable auto-assign public IPv4.** This affects only instances launched after the change.
7. **Create a security group** in `test-vpc`. Allow SSH (22) from **your IP only** (`x.x.x.x/32`), and HTTP (80) if you'll test a web server.
    - SSH from `0.0.0.0/0` is one of the most common audit findings. Don't do it.
    - Browser-based **EC2 Instance Connect** connects from an AWS-owned IP range, so allow that range (AWS publishes it as a managed prefix list per Region).
    - Better still for real projects: **SSM Session Manager** or an **EC2 Instance Connect Endpoint**, with no SSH port open at all.
8. **Launch an EC2 instance** (Ubuntu, t2.micro or t3.micro, key pair) into `test-vpc` / `test-public-subnet`. It gets a private IP in the 10.0.1.4 to 10.0.1.254 range, plus a public IP.
9. **Connect: it works.** The IGW maps the instance's public IP to its private IP (1:1 NAT), the security group allows the connection in, and the subnet's 0.0.0.0/0 → IGW route sends the replies back out.
10. **Prove the concept.** Remove the `0.0.0.0/0 → IGW` route (or launch into the private subnet) and the connection times out. The same subnet is now private.

**Troubleshooting order when you can't connect:** public IP present? → subnet's route table has `0.0.0.0/0 → IGW`? → IGW attached to this VPC? → NACL allows inbound **and** outbound (ephemeral ports 1024–65535)? → security group allows your IP? → OS firewall and service listening? VPC Reachability Analyzer checks the AWS side of this chain (route tables, gateways, NACLs, security groups) for you. It can't see the OS firewall or whether the service is running.

### Next exercise: add a NAT gateway

Give the private subnet outbound internet:

1. Allocate an Elastic IP.
2. Create a NAT gateway **in the public subnet** using that EIP.
3. Create `test-private-rt` with `0.0.0.0/0 → nat-xxxx` and associate it with the private subnet.
4. Launch an instance in the private subnet. It can run `apt update` but can't be reached from the internet.

The NAT gateway is never placed *in* the private subnet. Only the private subnet's route table points to it.

## 5. Common misconceptions, corrected

These come up constantly in beginner material and in interviews. Each one sounds close to right, which is why it sticks.

| Misconception | What's actually true |
| --- | --- |
| A NAT gateway connects the public subnet to the private subnet | Subnets reach each other over the `local` route. NAT only gives private instances **outbound** internet |
| You attach the NAT gateway to the private subnet | The NAT gateway lives **in a public subnet**. The private subnet's route table points `0.0.0.0/0` at it |
| 192.0.0.0/16 is a private range | Only 10.0.0.0/8, 172.16.0.0/12 and 192.168.0.0/16 are private (RFC 1918) |
| VPCs exist because IPv4 addresses are running out | VPCs exist for isolation and control. Private ranges repeat freely across VPCs |
| Tenancy picks shared or dedicated network hardware | Tenancy is about whether EC2 instances run on shared or single-tenant hardware |
| AWS reserves 5 IPs per VPC | 5 per **subnet**: .0, .1 router, .2 DNS, .3 future, last = broadcast |
| The first usable IP in 10.0.0.0/24 is 10.0.0.1 | In an AWS subnet it's 10.0.0.4 |
| You can resize a subnet to fix an overlap | Subnet CIDRs are fixed. Recreate the subnet or add a secondary VPC CIDR |
| A subnet named "public" is public | Public = route `0.0.0.0/0 → IGW` + a public IP. The name means nothing |
| A DB in a private subnet can't be attacked | It has no direct internet path, but a compromised public host can still reach it. Lock down the DB's security group |
| You create AZs inside your VPC | AZs already exist. The VPC spans them, and each subnet sits in one AZ |
| An IGW is a physical device | It's a managed, redundant, horizontally scaled VPC component |
| Opening SSH to 0.0.0.0/0 is fine for testing | Restrict to your IP, or use SSM Session Manager / EC2 Instance Connect Endpoint |
| A route to "0.0.0.0" | Write `0.0.0.0/0`: the default route |
| A /32 VPC or subnet holds one device | /32 is valid in SG rules, but VPCs and subnets must be /16 to /28 |

---

Next: [Part 2: VPC in production](./aws-vpc-part-2-production.md)
