# 📘 NAT-Gateway.md
# AWS NAT Gateway – In-Depth Guide

## 1. What is a NAT Gateway?
A **NAT Gateway (Network Address Translation Gateway)** enables outbound Internet traffic **from private subnet instances**, while blocking inbound traffic from the Internet.

### ✔ Purpose
- Allows outbound Internet access for private EC2s (updates, packages).
- Blocks inbound traffic, ensuring security.
- Enables private → public Internet communication safely.

---

## 2. Why NAT Gateway Is Needed?
Private EC2 instances **cannot access the Internet** because:
- They **do NOT have public IPs**
- Their route tables **do not route to an IGW**

Thus commands like:

```
yum update
apt update
npm install
docker pull
curl https://example.com
```

❌ Fail without a NAT Gateway.

---

## 3. How NAT Gateway Works (Deep Dive)

### 📌 Architecture Diagram

```
                    +------------------------+
                    |       Internet         |
                    +-----------+------------+
                                |
                        +-------v--------+
                        |      IGW       |
                        +-------+--------+
                                |
                   Public Subnet (has IGW route)
                                |
                +---------------+----------------+
                |                                |
      +---------v----------+          +-----------v----------+
      |   NAT Gateway      |          | Bastion / Load Bal   |
      +---------+----------+          +----------------------+
                |
         Private Subnet (NO Public IPs)
                |
      +---------v-------------+
      | Private EC2 Instances |
      +------------------------+
```

---

## 4. NAT Gateway vs Internet Gateway

| Feature | NAT Gateway | Internet Gateway |
|--------|-------------|------------------|
| Direction | Outbound only | Inbound + Outbound |
| Works for | Private Subnet | Public Subnet |
| Requires Public IP? | ✔ Yes | ✔ Yes |
| Allows inbound? | ❌ No | ✔ Yes |
| Security | High | Medium |
| Use Case | Package installs, updates | Hosting public apps, APIs |

---

## 5. Types of NAT in AWS

### 1️⃣ NAT Gateway (Recommended)
- AWS-managed  
- Scalable & highly available  
- No admin overhead  
- COST: Higher  

### 2️⃣ NAT Instance
- EC2 instance acting as NAT  
- Must update, scale, manage  
- Cheaper but high maintenance  
- Only for learning/testing  

---

## 6. Private Subnet Routing Through NAT

### ✔ Private Route Table
```
Destination        Target
-------------------------------------------
10.0.0.0/16        local
0.0.0.0/0          nat-xxxxxxxx
```

### ✔ Public Subnet Route Table (Where NAT Gateway lives)
```
Destination        Target
-------------------------------------------
10.0.0.0/16        local
0.0.0.0/0          igw-xxxxxxxx
```

---

## 7. How NAT Gateway Performs Translation

Example:

Private EC2 runs: `apt update`

| Step | Action |
|------|--------|
| 1 | EC2 uses private IP 10.0.2.15 |
| 2 | Traffic → Private Route Table → NAT Gateway |
| 3 | NAT replaces private IP with its Elastic IP |
| 4 | Sends request to Internet through IGW |
| 5 | Response returns to NAT |
| 6 | NAT maps response back to EC2 |
| 7 | EC2 receives data |

---

## 8. Setup NAT Gateway – Step-by-Step

### ✔ Prerequisites
- VPC (10.0.0.0/16)
- Public Subnet
- Private Subnet
- IGW attached to VPC  

---

### 🔧 STEP 1: Create an Elastic IP (EIP)
VPC Console → **Elastic IPs** → Allocate → Save EIP.

---

### 🔧 STEP 2: Create NAT Gateway
VPC → NAT Gateways → Create  
- Subnet: Public Subnet  
- Elastic IP: Choose EIP  

---

### 🔧 STEP 3: Update Private Route Table
Add route:
```
Destination: 0.0.0.0/0
Target: NAT Gateway (nat-xxxxx)
```

---

### 🔧 STEP 4: Test from Private EC2
SSH into private EC2 via Bastion:

```
curl https://google.com
sudo yum update -y
```

If working → NAT Gateway is correctly configured.

---

## 9. NAT Gateway High Availability

NAT Gateway is **AZ-specific**, meaning:

✔ Private Subnet in AZ-a → NAT in AZ-a = Works  
❌ Private Subnet in AZ-b → NAT in AZ-a = Does NOT Work  

### ✔ AWS Best Practice  
Deploy **one NAT Gateway per AZ** for high availability.

---

## 10. NAT Gateway Pricing

You pay for:
- **Per hour cost** (₹ 32–45/hr)
- **Data processed cost** (₹ 6–10 per GB)

### ⚠ NAT Gateway becomes expensive for:
- Docker pulls  
- Large downloads  
- S3 sync  
- System updates  

Use **VPC Endpoints** instead when possible.

---

## 11. NAT Gateway vs VPC Endpoints

| Feature | NAT Gateway | VPC Endpoint |
|---------|-------------|--------------|
| Cost | High | Very low |
| Performance | Medium | High |
| Security | Medium | Very Secure |
| For AWS Services (S3/DynamoDB) | ❌ Not ideal | ✔ Best option |

---

## 12. Troubleshooting NAT Gateway

### ✔ 1️⃣ Check Private Route Table  
Must contain:
```
0.0.0.0/0 → NAT Gateway
```

### ✔ 2️⃣ NAT Gateway Status  
Should be **Available**.

### ✔ 3️⃣ NAT Gateway is in PUBLIC Subnet  
Public subnet must have:
```
0.0.0.0/0 → IGW
```

### ✔ 4️⃣ Security Groups  
Outbound MUST allow:
```
0.0.0.0/0
```

### ✔ 5️⃣ NACL Rules  
Allow ephemeral ports:
```
1024-65535
```

---

## 13. When NOT to Use NAT Gateway

Avoid NAT gateway if:
- You only need S3 or DynamoDB access  
- You want to reduce cost  
- You use serverless (Lambda doesn’t need NAT for S3)

### Use Instead:
- **S3 Gateway Endpoint**  
- **DynamoDB Gateway Endpoint**  

---

## 14. Summary

NAT Gateway provides:
✔ Secure outbound internet for private EC2  
✔ No inbound access  
✔ Fully managed  
✔ Highly scalable  

But:
⚠ Costly  
⚠ One per AZ needed  

---
