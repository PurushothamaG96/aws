# Amazon VPC (Virtual Private Cloud)

Amazon VPC lets you provision a logically isolated section of the AWS Cloud where you define and control your virtual network. This includes IP address ranges, subnets, route tables, gateways, and security configurations.

---

# 🧩 **1. What is VPC?**

A **Virtual Private Cloud (VPC)** is your own isolated network inside AWS. You control:

* IP ranges
* Subnets (public/private)
* Route tables
* Gateways
* Network ACLs & Security Groups

VPC behaves like your private data center inside AWS.

---

# 🧱 **2. VPC Key Components**

## **2.1 CIDR Block**

Defines the IP address range of your VPC.

* Example: `10.0.0.0/16`
* Allows 65,536 IP addresses

## **2.2 Subnets**

A subnet is a subset of IP addresses within the VPC.

* **Public Subnet:** Has route to Internet Gateway
* **Private Subnet:** No direct internet access

Example:

* Public: `10.0.1.0/24`
* Private: `10.0.2.0/24`

## **2.3 Route Tables**

Control where traffic flows.

* Public subnet route table → IGW
* Private subnet route table → NAT Gateway / Local routing

## **2.4 Internet Gateway (IGW)**

Allows internet access for public subnets.

## **2.5 NAT Gateway / NAT Instance**

Allows outbound internet access for resources in private subnets.

* NAT Gateway = Highly available + managed
* NAT Instance = Older, manual scaling

## **2.6 DHCP Options Set**

Defines DNS servers, NTP servers, etc.

* Default DNS: `AmazonProvidedDNS`

## **2.7 Security Groups**

Firewall at instance level

* Stateful (return traffic automatically allowed)

## **2.8 Network ACLs (NACLs)**

Firewall at subnet level

* Stateless (return traffic rules must be added separately)

## **2.9 VPC Peering**

Connects two VPCs privately.

* No transitive peering

## **2.10 Transit Gateway**

Centrally connects multiple VPCs and on-prem networks.

## **2.11 VPC Endpoints**

Private access to AWS services without using the internet.

* **Interface Endpoint** → Elastic Network Interface
* **Gateway Endpoint** → S3, DynamoDB

---

# 🌐 **3. VPC Architecture Diagram (Conceptual)**

```
                Internet
                   |
              [Internet Gateway]
                   |
         -----------------------------
         |           VPC             |
         | CIDR: 10.0.0.0/16         |
         |                           |
   ----------------      -------------------
   | Public Subnet |     | Private Subnet  |
   | 10.0.1.0/24   |     | 10.0.2.0/24     |
   ----------------      -------------------
         |                           |
  [EC2 / Load Balancer]        [EC2 / RDS]
         |                           |
   Route to IGW            Route to NAT Gateway
```

---

# 🔒 **4. Security in VPC**

## **4.1 Security Groups (SG)**

* Instance-level firewall
* **Stateful**
* Only allow rules
* Auto return traffic allowed

Good for:

* EC2
* RDS
* Lambda inside VPC

## **4.2 Network ACLs (NACLs)**

* Subnet-level firewall
* **Stateless**
* Allow and deny rules

Use cases:

* Deny specific IP ranges
* Additional subnet-level filtering

---

# 🛰️ **5. Routing in VPC**

Each subnet must be associated with a Route Table.

### Public Subnet Route Table

```
Destination       Target
10.0.0.0/16       local
0.0.0.0/0         Internet Gateway
```

### Private Subnet Route Table

```
Destination       Target
10.0.0.0/16       local
0.0.0.0/0         NAT Gateway
```

---

# 🔗 **6. VPC Connectivity Options**

## **6.1 VPC Peering**

* One-to-one connection
* No transitive routing

## **6.2 Transit Gateway (TGW)**

* Hub-and-spoke model
* Connects multiple VPCs and VPN

## **6.3 Site-to-Site VPN**

* Encrypted connection to on-prem network

## **6.4 Direct Connect**

* Private, dedicated physical link to AWS

---

# 🏗️ **7. VPC Best Practices**

* Use multiple AZs for high availability
* Keep private subnets for RDS, ECS, Batch
* Use NAT Gateway (not NAT instance)
* Restrict inbound SG rules as much as possible
* Enable VPC Flow Logs
* Use VPC Endpoints for private access to S3/DynamoDB

---

# 🧪 **8. Exam Tips**

* IGW = public subnet access
* NAT = private subnet outbound internet
* SG = stateful, NACL = stateless
* Gateway endpoints only support S3 & DynamoDB
* VPC peering does NOT support transitive routing
* Use Transit Gateway for large network topologies

---

# 📚 **References**

* Amazon VPC Documentation
* AWS Certified Solutions Architect Exam Guide
* Hands-on VPC labs in AWS Console

---

**End of vpc.md**
