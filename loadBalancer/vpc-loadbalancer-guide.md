# AWS VPC — Complete Guide
> Simple explanations, step-by-step setup, key definitions & interview preparation

---

## Table of Contents

1. [What is a VPC?](#1-what-is-a-vpc)
2. [VPC Core Components — Simple Definitions](#2-vpc-core-components--simple-definitions)
3. [VPC Architecture Overview](#3-vpc-architecture-overview)
4. [Step-by-Step: Create a VPC from Scratch](#4-step-by-step-create-a-vpc-from-scratch)
5. [Step-by-Step: Create Subnets](#5-step-by-step-create-subnets)
6. [Step-by-Step: Internet Gateway](#6-step-by-step-internet-gateway)
7. [Step-by-Step: Route Tables](#7-step-by-step-route-tables)
8. [Step-by-Step: Security Groups](#8-step-by-step-security-groups)
9. [Step-by-Step: Network ACLs (NACLs)](#9-step-by-step-network-acls-nacls)
10. [Step-by-Step: NAT Gateway](#10-step-by-step-nat-gateway)
11. [VPC Peering](#11-vpc-peering)
12. [VPC Endpoints](#12-vpc-endpoints)
13. [VPN & Direct Connect](#13-vpn--direct-connect)
14. [Common Interview Questions & Answers](#14-common-interview-questions--answers)
15. [Troubleshooting](#15-troubleshooting)
16. [Quick Reference Commands](#16-quick-reference-commands)

---

## 1. What is a VPC?

**VPC = Virtual Private Cloud**

Think of it like this:

> 🏢 AWS is a huge apartment building.  
> Your **VPC** is your **private apartment** inside that building.  
> You decide who can enter, which rooms exist, and what connects to the outside world.

In technical terms, a VPC is a **logically isolated section of the AWS cloud** where you launch AWS resources in a virtual network that **you define and control**.

### Why do you need a VPC?

| Without VPC | With VPC |
|---|---|
| Resources exposed to internet by default | Full control over who can access what |
| No network isolation between customers | Completely isolated from other AWS customers |
| Cannot define custom IP ranges | Define your own IP address ranges |
| No control over traffic routing | Custom routing rules between subnets |

### Default VPC vs Custom VPC

| | Default VPC | Custom VPC |
|---|---|---|
| Created by | AWS automatically (per region) | You |
| CIDR block | 172.31.0.0/16 | Your choice |
| Subnets | Auto-created (public) per AZ | You create them |
| Internet access | Yes (all subnets are public) | Depends on your design |
| Best for | Quick testing/learning | Production workloads |

---

## 2. VPC Core Components — Simple Definitions

### 🔷 VPC (Virtual Private Cloud)
Your isolated private network inside AWS. Like your own data centre in the cloud. You define the IP range using a **CIDR block** (e.g., `10.0.0.0/16`).

---

### 🔷 CIDR Block (Classless Inter-Domain Routing)
The IP address range for your VPC or subnet.

```
10.0.0.0/16  →  65,536 total IP addresses  (10.0.0.0 – 10.0.255.255)
10.0.0.0/24  →  256 total IP addresses     (10.0.0.0 – 10.0.0.255)
10.0.0.0/28  →  16 total IP addresses      (smallest allowed in AWS)
```

> **Simple rule**: The smaller the number after `/`, the MORE IP addresses you get.

---

### 🔷 Subnet
A **subdivision of your VPC**. Like dividing your apartment into rooms — bedroom, kitchen, living room. Each subnet lives in one Availability Zone.

- **Public Subnet** — Has a route to the Internet Gateway. Instances here can be accessed from the internet.
- **Private Subnet** — No direct route to the internet. Instances here are hidden from the public.

---

### 🔷 Availability Zone (AZ)
A physical data centre (or cluster of data centres) within an AWS Region. Examples: `ap-south-1a`, `ap-south-1b` in Mumbai.

> Best practice: Create subnets in **at least 2 AZs** for high availability.

---

### 🔷 Internet Gateway (IGW)
The **door between your VPC and the internet**. Without it, nothing in your VPC can reach the internet and nobody from the internet can reach you. One IGW per VPC.

---

### 🔷 Route Table
A set of rules (**routes**) that tell network traffic **where to go**. Every subnet must be associated with a route table.

```
Destination: 0.0.0.0/0   →  Target: Internet Gateway  (sends all traffic to internet)
Destination: 10.0.0.0/16 →  Target: local             (traffic within VPC stays local)
```

---

### 🔷 Security Group (SG)
A **virtual firewall at the instance level**. Controls inbound and outbound traffic for individual EC2 instances, RDS databases, etc.

- **Stateful** — If you allow inbound traffic, the response is automatically allowed outbound (no need to add a rule for it).
- Rules are **ALLOW only** — you cannot explicitly deny.

---

### 🔷 Network ACL (NACL)
A **virtual firewall at the subnet level**. Controls traffic entering and leaving entire subnets.

- **Stateless** — You must explicitly allow both inbound AND outbound traffic.
- Rules are evaluated in **numbered order** (lowest first).
- Supports both **ALLOW and DENY** rules.

---

### 🔷 NAT Gateway (Network Address Translation)
Allows instances in a **private subnet** to access the internet (e.g., to download updates) **without being directly reachable from the internet**. Lives in a public subnet.

> Think of it like: Private subnet instances use NAT Gateway as a **proxy** to reach the internet.

---

### 🔷 Elastic IP (EIP)
A **static public IPv4 address** that you can attach to resources like EC2 instances or NAT Gateways. Unlike regular public IPs that change on restart, an Elastic IP stays the same.

---

### 🔷 VPC Peering
A **private connection between two VPCs** (same or different accounts/regions) using AWS's backbone network — no internet, no VPN required.

---

### 🔷 VPC Endpoint
Allows you to **privately connect your VPC to AWS services** (like S3, DynamoDB) without using the internet, a NAT Gateway, or a VPN.

---

### 🔷 Flow Logs
Captures **all network traffic metadata** going in and out of your VPC, subnets, or network interfaces. Stored in CloudWatch or S3. Used for monitoring, security, and troubleshooting.

---

## 3. VPC Architecture Overview

### 3-Tier Architecture (Recommended Pattern)

```
                          INTERNET
                              │
                    ┌─────────▼─────────┐
                    │  Internet Gateway  │
                    └─────────┬─────────┘
                              │
              ┌───────────────▼───────────────┐
              │           VPC 10.0.0.0/16      │
              │                               │
   ┌──────────▼──────────┐   ┌───────────────▼───────────┐
   │   Public Subnet      │   │     Public Subnet          │
   │   10.0.1.0/24 (AZ-a)│   │     10.0.2.0/24 (AZ-b)   │
   │  [Load Balancer]     │   │     [NAT Gateway]          │
   └──────────┬──────────┘   └───────────────┬───────────┘
              │                               │
   ┌──────────▼──────────┐   ┌───────────────▼───────────┐
   │   Private Subnet     │   │     Private Subnet         │
   │   10.0.3.0/24 (AZ-a)│   │     10.0.4.0/24 (AZ-b)   │
   │  [App Servers]       │   │     [App Servers]           │
   └──────────┬──────────┘   └───────────────┬───────────┘
              │                               │
   ┌──────────▼──────────┐   ┌───────────────▼───────────┐
   │   Private Subnet     │   │     Private Subnet         │
   │   10.0.5.0/24 (AZ-a)│   │     10.0.6.0/24 (AZ-b)   │
   │  [Databases / RDS]   │   │     [Databases / RDS]      │
   └─────────────────────┘   └───────────────────────────┘
```

**Layer 1 — Public Subnets**: Load Balancers, Bastion Hosts, NAT Gateways  
**Layer 2 — Private App Subnets**: EC2 application servers  
**Layer 3 — Private DB Subnets**: RDS, ElastiCache — no internet access at all

---

## 4. Step-by-Step: Create a VPC from Scratch

### Method 1 — AWS Console

1. Open **AWS Console** → Search for **VPC** → Click **VPC**
2. In the left menu, click **Your VPCs**
3. Click **Create VPC**

Fill in the form:

```
Resources to create:  VPC only
Name tag:             my-production-vpc
IPv4 CIDR block:      10.0.0.0/16
IPv6 CIDR block:      No IPv6 CIDR block
Tenancy:              Default
```

4. Click **Create VPC**

> ✅ Your VPC is created! But it's empty — no subnets, no internet access yet.

### Method 2 — AWS CLI

```bash
# Create VPC
aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=my-production-vpc}]'

# Output gives you the VPC ID — save it!
# e.g., vpc-0abc1234def567890

# Enable DNS hostnames (recommended)
aws ec2 modify-vpc-attribute \
  --vpc-id vpc-0abc1234def567890 \
  --enable-dns-hostnames '{"Value": true}'

# Enable DNS resolution
aws ec2 modify-vpc-attribute \
  --vpc-id vpc-0abc1234def567890 \
  --enable-dns-support '{"Value": true}'
```

### How to Choose Your CIDR Block

| CIDR | IP Range | Total IPs | Use Case |
|---|---|---|---|
| `10.0.0.0/16` | 10.0.0.0 – 10.0.255.255 | 65,536 | Large production VPC |
| `10.0.0.0/20` | 10.0.0.0 – 10.0.15.255 | 4,096 | Medium VPC |
| `10.0.0.0/24` | 10.0.0.0 – 10.0.0.255 | 256 | Small VPC / single subnet |

> ⚠️ **AWS reserves 5 IPs** in every subnet (first 4 + last 1). So a `/24` gives you 251 usable IPs, not 256.

---

## 5. Step-by-Step: Create Subnets

### Create a Public Subnet

1. Left menu → **Subnets** → **Create subnet**
2. Select your VPC: `my-production-vpc`
3. Fill in:

```
Subnet name:          public-subnet-1a
Availability Zone:    ap-south-1a   (choose your region's AZ)
IPv4 CIDR block:      10.0.1.0/24
```

4. Click **Add new subnet** to add more, then **Create subnet**

### Create a Private Subnet

```
Subnet name:          private-app-subnet-1a
Availability Zone:    ap-south-1a
IPv4 CIDR block:      10.0.3.0/24
```

### Enable Auto-Assign Public IP on Public Subnet

After creating the public subnet:
1. Select the subnet → **Actions** → **Edit subnet settings**
2. Check ✅ **Enable auto-assign public IPv4 address**
3. Save

### Subnet CIDR Planning Example

```
VPC: 10.0.0.0/16

Public Subnets:
  10.0.1.0/24  →  public-subnet-1a   (AZ: ap-south-1a)
  10.0.2.0/24  →  public-subnet-1b   (AZ: ap-south-1b)

Private App Subnets:
  10.0.3.0/24  →  private-app-1a     (AZ: ap-south-1a)
  10.0.4.0/24  →  private-app-1b     (AZ: ap-south-1b)

Private DB Subnets:
  10.0.5.0/24  →  private-db-1a      (AZ: ap-south-1a)
  10.0.6.0/24  →  private-db-1b      (AZ: ap-south-1b)
```

### CLI: Create Subnets

```bash
# Public Subnet
aws ec2 create-subnet \
  --vpc-id vpc-0abc1234def567890 \
  --cidr-block 10.0.1.0/24 \
  --availability-zone ap-south-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-subnet-1a}]'

# Private App Subnet
aws ec2 create-subnet \
  --vpc-id vpc-0abc1234def567890 \
  --cidr-block 10.0.3.0/24 \
  --availability-zone ap-south-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-app-1a}]'
```

---

## 6. Step-by-Step: Internet Gateway

The Internet Gateway is what connects your VPC to the internet.

### Create and Attach

1. Left menu → **Internet Gateways** → **Create internet gateway**

```
Name tag: my-vpc-igw
```

2. Click **Create internet gateway**
3. After creation → **Actions** → **Attach to VPC** → Select `my-production-vpc` → **Attach**

### CLI

```bash
# Create IGW
aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=my-vpc-igw}]'
# Note the InternetGatewayId: igw-0abc123...

# Attach to VPC
aws ec2 attach-internet-gateway \
  --internet-gateway-id igw-0abc123 \
  --vpc-id vpc-0abc1234def567890
```

> ⚠️ Just creating an IGW is NOT enough. You must also add a route in the Route Table pointing to it.

---

## 7. Step-by-Step: Route Tables

Every subnet must be associated with a route table that tells traffic where to go.

### Create a Public Route Table

1. Left menu → **Route Tables** → **Create route table**

```
Name:  public-route-table
VPC:   my-production-vpc
```

2. After creation → **Routes** tab → **Edit routes** → **Add route**:

```
Destination: 0.0.0.0/0
Target:      Internet Gateway → igw-0abc123
```

3. Click **Save changes**

### Associate Public Subnet with Public Route Table

1. **Subnet Associations** tab → **Edit subnet associations**
2. Select `public-subnet-1a` and `public-subnet-1b`
3. **Save associations**

### Create a Private Route Table

```
Name:  private-route-table
VPC:   my-production-vpc
```

Routes:
```
Destination: 10.0.0.0/16  → local  (auto-added, cannot remove)
Destination: 0.0.0.0/0    → NAT Gateway  (add after creating NAT GW)
```

Associate private subnets (`private-app-1a`, `private-app-1b`) with this table.

### Route Table Summary

| Route Table | Associated Subnets | Routes |
|---|---|---|
| `public-route-table` | public-subnet-1a, 1b | local + IGW |
| `private-route-table` | private-app-1a, 1b | local + NAT GW |
| `db-route-table` | private-db-1a, 1b | local only |

---

## 8. Step-by-Step: Security Groups

Security Groups are **instance-level firewalls**. Think of them as a bouncer at the door of each EC2 instance.

### Create a Security Group for the Load Balancer

1. Left menu → **Security Groups** → **Create security group**

```
Name:         alb-sg
Description:  Security group for Application Load Balancer
VPC:          my-production-vpc
```

Inbound Rules:
```
Type: HTTP   | Port: 80   | Source: 0.0.0.0/0     (from internet)
Type: HTTPS  | Port: 443  | Source: 0.0.0.0/0     (from internet)
```

Outbound Rules:
```
Type: All traffic | Destination: 0.0.0.0/0
```

### Create a Security Group for App Servers

```
Name:         app-server-sg
Description:  Security group for application servers
VPC:          my-production-vpc
```

Inbound Rules:
```
Type: HTTP  | Port: 80 | Source: alb-sg   ← only from ALB, not direct internet!
Type: SSH   | Port: 22 | Source: bastion-sg (or your office IP only)
```

### Create a Security Group for Database

```
Name:         db-sg
Description:  Security group for RDS
VPC:          my-production-vpc
```

Inbound Rules:
```
Type: MySQL/Aurora | Port: 3306 | Source: app-server-sg  ← only from app servers
```

> ✅ **Best Practice**: Reference other security groups as sources, not IP ranges. This way, if you add new app servers, they automatically get DB access.

### CLI: Create Security Group

```bash
# Create SG
aws ec2 create-security-group \
  --group-name app-server-sg \
  --description "Security group for app servers" \
  --vpc-id vpc-0abc1234def567890

# Add inbound rule
aws ec2 authorize-security-group-ingress \
  --group-id sg-0abc123 \
  --protocol tcp \
  --port 80 \
  --source-group sg-alb123
```

---

## 9. Step-by-Step: Network ACLs (NACLs)

NACLs are an **extra layer of security at the subnet level**. Like a security checkpoint at the entrance of an entire building floor.

### Default NACL Behaviour

- The **default NACL** allows ALL inbound and outbound traffic.
- A **custom NACL** DENIES everything by default until you add allow rules.

### Create a Custom NACL

1. Left menu → **Network ACLs** → **Create network ACL**

```
Name:  public-nacl
VPC:   my-production-vpc
```

2. Add Inbound Rules:

| Rule # | Type | Protocol | Port | Source | Allow/Deny |
|---|---|---|---|---|---|
| 100 | HTTP | TCP | 80 | 0.0.0.0/0 | ALLOW |
| 110 | HTTPS | TCP | 443 | 0.0.0.0/0 | ALLOW |
| 120 | Custom TCP | TCP | 1024-65535 | 0.0.0.0/0 | ALLOW |
| * | All traffic | All | All | 0.0.0.0/0 | DENY |

> Rule 120 allows **ephemeral ports** — required because responses from internet come back on random high ports.

3. Add Outbound Rules:

| Rule # | Type | Protocol | Port | Destination | Allow/Deny |
|---|---|---|---|---|---|
| 100 | HTTP | TCP | 80 | 0.0.0.0/0 | ALLOW |
| 110 | HTTPS | TCP | 443 | 0.0.0.0/0 | ALLOW |
| 120 | Custom TCP | TCP | 1024-65535 | 0.0.0.0/0 | ALLOW |
| * | All traffic | All | All | 0.0.0.0/0 | DENY |

4. Associate with public subnets.

---

## 10. Step-by-Step: NAT Gateway

Allows private subnet instances to reach the internet (e.g., to `yum update` or download packages) **without being publicly accessible**.

### Create NAT Gateway

> ⚠️ NAT Gateway must be placed in a **PUBLIC subnet** and needs an **Elastic IP**.

1. Left menu → **NAT Gateways** → **Create NAT Gateway**

```
Name:              my-nat-gateway
Subnet:            public-subnet-1a      ← must be PUBLIC
Connectivity type: Public
Elastic IP:        Allocate Elastic IP   ← click to auto-allocate
```

2. Click **Create NAT Gateway** — takes ~2 minutes to become Available.

### Update Private Route Table

Add this route to `private-route-table`:
```
Destination: 0.0.0.0/0
Target:      NAT Gateway → nat-0abc123
```

Now private instances can reach the internet, but cannot be reached from it.

### NAT Gateway vs NAT Instance

| | NAT Gateway | NAT Instance |
|---|---|---|
| Managed by | AWS (fully managed) | You (self-managed EC2) |
| Availability | Highly available within AZ | Single point of failure |
| Bandwidth | Up to 100 Gbps | Depends on instance type |
| Cost | Higher | Lower (but more operational work) |
| Recommendation | ✅ Always use for production | Only for cost-sensitive dev/test |

### CLI: Create NAT Gateway

```bash
# Allocate Elastic IP
aws ec2 allocate-address --domain vpc
# Note AllocationId: eipalloc-0abc123

# Create NAT Gateway in public subnet
aws ec2 create-nat-gateway \
  --subnet-id subnet-public1a \
  --allocation-id eipalloc-0abc123 \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=my-nat-gateway}]'

# Add route in private route table
aws ec2 create-route \
  --route-table-id rtb-private123 \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id nat-0abc123
```

---

## 11. VPC Peering

Connects two VPCs privately so resources in each can talk to each other as if they're in the same network — no internet, no VPN.

### Use Cases
- Connect dev VPC to shared-services VPC (logging, monitoring tools)
- Connect your VPC to a vendor's VPC in another AWS account
- Multi-region VPC connectivity

### How to Set Up VPC Peering

**VPC A**: `10.0.0.0/16`  
**VPC B**: `10.1.0.0/16`

> ⚠️ CIDR blocks must NOT overlap. `10.0.0.0/16` and `10.0.0.0/16` in two VPCs CANNOT be peered.

1. **VPC A side** → **Peering Connections** → **Create Peering Connection**

```
Peering connection name: vpc-a-to-vpc-b
VPC (Requester):         vpc-a-id
VPC (Accepter):          vpc-b-id  (same or different account/region)
```

2. **VPC B side** → Select the pending peering connection → **Actions** → **Accept Request**

3. **Update Route Tables on BOTH sides**:

In VPC A's route table:
```
Destination: 10.1.0.0/16  →  Target: Peering Connection (pcx-0abc123)
```

In VPC B's route table:
```
Destination: 10.0.0.0/16  →  Target: Peering Connection (pcx-0abc123)
```

4. Update **Security Groups** to allow traffic from the other VPC's CIDR.

### VPC Peering Limitations
- **Not transitive**: If A peers B and B peers C, A cannot talk to C through B.
- **No overlapping CIDRs** allowed.
- For transitive connectivity, use **AWS Transit Gateway** instead.

---

## 12. VPC Endpoints

Connect to AWS services (S3, DynamoDB, etc.) **privately** without going through the internet.

### Types of VPC Endpoints

| Type | Works With | How It Works |
|---|---|---|
| **Gateway Endpoint** | S3, DynamoDB only | Added as a route in route table (free) |
| **Interface Endpoint** | 100+ AWS services | Creates an ENI (private IP) in your subnet (paid) |

### Create a Gateway Endpoint for S3

1. Left menu → **Endpoints** → **Create Endpoint**

```
Name:              s3-gateway-endpoint
Service category:  AWS services
Service:           com.amazonaws.ap-south-1.s3  (Gateway type)
VPC:               my-production-vpc
Route tables:      Select private-route-table
```

2. Click **Create endpoint**

A route is automatically added to your private route table:
```
Destination: pl-xxxxx (S3 prefix list)  →  Target: vpce-0abc123
```

Now your private instances can access S3 without internet or NAT Gateway — **cheaper and more secure**.

---

## 13. VPN & Direct Connect

### AWS Site-to-Site VPN
Connects your **on-premises network** to your VPC over the **internet** using encrypted IPSec tunnels.

```
On-premises Office  ←──[Encrypted VPN Tunnel]──→  Virtual Private Gateway  ←──→  VPC
```

- Easy to set up (minutes)
- Uses the public internet (encrypted)
- Bandwidth limited by internet connection
- Good for: small workloads, backup connectivity

### AWS Direct Connect
A **dedicated private physical network connection** between your data centre and AWS — no internet involved.

```
On-premises DC  ←──[Private Fiber Line]──→  AWS Direct Connect Location  ←──→  VPC
```

- Very high bandwidth (1 Gbps, 10 Gbps, 100 Gbps)
- Consistent, low-latency performance
- Takes weeks to provision (physical setup required)
- Good for: large data transfers, latency-sensitive apps, hybrid cloud

---

## 14. Common Interview Questions & Answers

### 🔹 Q1: What is a VPC and why is it used?

A VPC (Virtual Private Cloud) is a logically isolated network within AWS where you launch resources. It gives you complete control over your network environment — including IP address ranges, subnets, route tables, and gateways. It's used to isolate workloads, control traffic flow, and secure resources from public access.

---

### 🔹 Q2: What is the difference between a Public and Private Subnet?

| | Public Subnet | Private Subnet |
|---|---|---|
| Route to internet | Yes (via Internet Gateway) | No direct route |
| Instances can get public IP | Yes | No |
| Accessible from internet | Yes (if SG allows) | No |
| Internet access outbound | Direct via IGW | Via NAT Gateway |
| Typical resources | Load balancers, Bastion hosts | App servers, Databases |

---

### 🔹 Q3: What is the difference between Security Groups and NACLs?

| Feature | Security Group | NACL |
|---|---|---|
| Level | Instance level | Subnet level |
| State | Stateful | Stateless |
| Rules | Allow only | Allow and Deny |
| Evaluation | All rules evaluated | Rules evaluated in order (lowest # first) |
| Default | Deny all inbound, Allow all outbound | Allow all (default NACL) |
| Applies to | ENI / Instance | Entire Subnet |

> **Interview tip**: Security Groups are stateful — if you allow port 80 inbound, the response is automatically allowed outbound. NACLs are stateless — you must explicitly allow both inbound port 80 AND outbound ephemeral ports (1024-65535).

---

### 🔹 Q4: What are ephemeral ports and why do you need them in NACLs?

When a client connects to your server on port 80, the server responds back to the **client's randomly chosen port** between 1024–65535 (called an ephemeral/dynamic port). Since NACLs are stateless, you must explicitly allow outbound traffic on ports `1024-65535` so responses can reach clients. Security Groups handle this automatically (stateful).

---

### 🔹 Q5: What is the difference between a NAT Gateway and an Internet Gateway?

| | Internet Gateway | NAT Gateway |
|---|---|---|
| Direction | Both inbound and outbound | Outbound only |
| Who uses it | Public subnet resources | Private subnet resources |
| Enables | Bidirectional internet access | Internet access for private instances without exposing them |
| Location | Attached to VPC | Lives in a public subnet |
| Type | Gateway (no IP) | Has an Elastic IP |

---

### 🔹 Q6: What is VPC Peering and what are its limitations?

VPC Peering is a private network connection between two VPCs (same/different account or region) using AWS's backbone. Traffic never traverses the public internet.

**Limitations:**
- **Not transitive** — A→B and B→C does NOT mean A→C
- **No overlapping CIDR blocks** between peered VPCs
- **No edge-to-edge routing** — cannot route through peered VPC to reach internet, VPN, or Direct Connect

**Alternative for transitive routing**: **AWS Transit Gateway**

---

### 🔹 Q7: What is AWS Transit Gateway?

Transit Gateway is a **central hub** that connects multiple VPCs and on-premises networks. Instead of creating many peering connections (which don't scale), all VPCs connect to the Transit Gateway in a hub-and-spoke model.

```
VPC-A ──┐
VPC-B ──┼──→ Transit Gateway ──→ On-premises (VPN/Direct Connect)
VPC-C ──┘
```

Supports **transitive routing** — VPC-A can reach VPC-C through the Transit Gateway.

---

### 🔹 Q8: How many IP addresses does AWS reserve in each subnet?

AWS reserves **5 IP addresses** in every subnet:

| Reserved IP | Purpose |
|---|---|
| x.x.x.0 | Network address |
| x.x.x.1 | VPC router (default gateway) |
| x.x.x.2 | AWS DNS server |
| x.x.x.3 | Reserved for future use |
| x.x.x.255 | Broadcast address |

So a `/24` subnet has 256 − 5 = **251 usable IPs**.

---

### 🔹 Q9: What is a VPC Endpoint and when would you use it?

A VPC Endpoint lets your VPC communicate with AWS services (like S3, DynamoDB, SQS) **privately without internet or NAT Gateway**.

**Use it when:**
- You want to avoid NAT Gateway costs for S3 access
- Compliance requires no data to leave the AWS network
- You want lower latency to AWS services

**Types:**
- **Gateway Endpoint**: S3 and DynamoDB — free, added to route table
- **Interface Endpoint**: All other AWS services — creates a private IP (ENI) in your subnet, costs per hour

---

### 🔹 Q10: Can two VPCs have the same CIDR block? What problems does this cause?

Yes, two separate VPCs can have the same CIDR block. But **you cannot peer them** because there would be no way to determine which VPC a packet belongs to — routes would be ambiguous.

**Best practice**: Plan your CIDR blocks ahead of time across all VPCs to avoid overlaps, especially if you anticipate peering or Transit Gateway connections in the future.

---

### 🔹 Q11: What happens to traffic if you have both a Security Group and a NACL?

Traffic must pass **both** the NACL (subnet level) and the Security Group (instance level).

**Inbound**: NACL is evaluated first → then Security Group  
**Outbound**: Security Group is evaluated first → then NACL

Both must allow the traffic for it to succeed. If either blocks it, the traffic is dropped.

---

### 🔹 Q12: What is a Bastion Host?

A Bastion Host (also called a Jump Box) is a special EC2 instance in a **public subnet** that you SSH into first, then SSH from there into instances in private subnets.

```
Your Laptop → [SSH] → Bastion Host (public subnet) → [SSH] → Private EC2 Instance
```

- The private EC2 security group only allows SSH from the Bastion's private IP (or SG)
- Bastion SG only allows SSH from your office/home IP
- **Alternative**: AWS Systems Manager Session Manager — SSH without a Bastion Host at all

---

### 🔹 Q13: What is VPC Flow Logs?

VPC Flow Logs capture **metadata** about IP traffic going to and from network interfaces in your VPC. It does **not** capture payload/content — just the "who talked to whom, when, on which port, and was it accepted or rejected."

**Stored in**: CloudWatch Logs or S3  
**Use cases**: Security analysis, troubleshooting connectivity, compliance auditing

---

### 🔹 Q14: What is the difference between VPN and Direct Connect?

| | Site-to-Site VPN | Direct Connect |
|---|---|---|
| Connection | Over public internet (encrypted) | Private dedicated fiber |
| Setup time | Minutes | Weeks |
| Bandwidth | Up to ~1.25 Gbps | 1, 10, or 100 Gbps |
| Latency | Variable (internet) | Consistent and low |
| Cost | Low | High |
| Reliability | Less reliable | Very reliable |
| Use case | Small workloads, backup link | Large data transfer, low-latency needs |

---

## 15. Troubleshooting

### Instance in private subnet cannot reach internet

```
Checklist:
✅ NAT Gateway exists and is in Active state?
✅ NAT Gateway is in a PUBLIC subnet?
✅ Public subnet has route to Internet Gateway?
✅ Private subnet route table has 0.0.0.0/0 → NAT Gateway?
✅ Instance Security Group allows outbound traffic?
✅ NACL allows outbound traffic and inbound ephemeral ports?
```

### Cannot SSH into EC2 instance

```
Checklist:
✅ Is the instance in a public subnet?
✅ Does the subnet have auto-assign public IP enabled?
✅ Is there a route 0.0.0.0/0 → Internet Gateway in the route table?
✅ Security Group allows inbound TCP port 22 from your IP?
✅ NACL allows inbound port 22 and outbound ephemeral ports?
✅ Correct key pair used for SSH?
✅ Instance is in Running state?
```

### Two instances in different subnets cannot communicate

```
Checklist:
✅ Both instances in the same VPC?
✅ Security Groups allow traffic between them (source = other SG or CIDR)?
✅ NACL allows traffic in both directions?
✅ Route table has a 'local' route for the VPC CIDR?
```

### Enable VPC Flow Logs to diagnose

```bash
# Enable flow logs for a VPC, sent to CloudWatch
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids vpc-0abc1234def567890 \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name /aws/vpc/flowlogs \
  --deliver-logs-permission-arn arn:aws:iam::123456789012:role/flowlogsRole
```

Look for `REJECT` entries — these show you exactly what traffic is being blocked and where.

---

## 16. Quick Reference Commands

```bash
# ── VPC ─────────────────────────────────────────────────────────────────────
# List all VPCs
aws ec2 describe-vpcs

# Get your VPC details
aws ec2 describe-vpcs --vpc-ids vpc-0abc1234def567890

# ── Subnets ──────────────────────────────────────────────────────────────────
# List all subnets
aws ec2 describe-subnets

# List subnets in a specific VPC
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=vpc-0abc1234def567890"

# ── Internet Gateway ─────────────────────────────────────────────────────────
# List internet gateways
aws ec2 describe-internet-gateways

# ── Route Tables ─────────────────────────────────────────────────────────────
# List route tables
aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=vpc-0abc1234def567890"

# ── Security Groups ──────────────────────────────────────────────────────────
# List security groups
aws ec2 describe-security-groups \
  --filters "Name=vpc-id,Values=vpc-0abc1234def567890"

# ── NAT Gateways ─────────────────────────────────────────────────────────────
# List NAT gateways
aws ec2 describe-nat-gateways

# ── NACLs ────────────────────────────────────────────────────────────────────
# List NACLs
aws ec2 describe-network-acls \
  --filters "Name=vpc-id,Values=vpc-0abc1234def567890"

# ── VPC Peering ──────────────────────────────────────────────────────────────
# List peering connections
aws ec2 describe-vpc-peering-connections

# ── VPC Endpoints ────────────────────────────────────────────────────────────
# List endpoints
aws ec2 describe-vpc-endpoints \
  --filters "Name=vpc-id,Values=vpc-0abc1234def567890"

# ── Flow Logs ────────────────────────────────────────────────────────────────
# List flow logs
aws ec2 describe-flow-logs
```

---

## Summary Cheat Sheet

```
VPC Creation Checklist:
  ✅ Create VPC with CIDR block
  ✅ Create public subnets (at least 2 AZs)
  ✅ Create private subnets (app layer, db layer)
  ✅ Create and attach Internet Gateway
  ✅ Create public route table → route 0.0.0.0/0 to IGW
  ✅ Associate public subnets with public route table
  ✅ Create Elastic IP → Create NAT Gateway in public subnet
  ✅ Create private route table → route 0.0.0.0/0 to NAT GW
  ✅ Associate private subnets with private route table
  ✅ Create Security Groups (ALB, App, DB)
  ✅ (Optional) Create NACLs for extra subnet-level security
  ✅ (Optional) Create VPC Endpoints for S3/DynamoDB
  ✅ Enable VPC Flow Logs for monitoring
```

---

*Document version: 1.0 | AWS Region: ap-south-1 (Mumbai) | Last updated: March 2026*