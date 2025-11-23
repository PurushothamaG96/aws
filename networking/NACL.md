# **AWS Network ACL (NACL) – In-Depth Guide**

A **Network ACL (NACL)** is a **stateless firewall** for controlling inbound and outbound traffic at the **subnet level** within a VPC. It provides an extra layer of defense and controls what traffic is allowed to enter/leave each subnet.

---

# 📘 **1. What is a Network ACL?**

A **Network Access Control List (NACL)** is:

* A **subnet-level** firewall
* **Stateless** (does NOT remember previous requests)
* Rule-based with **allow** or **deny** statements
* Evaluates rules in **ascending order** (lowest to highest number)
* Applies rules to **both inbound and outbound traffic**
* The **default NACL allows all traffic**

---

# 🔍 **2. Key Characteristics of NACLs**

| Feature          | Description                                           |
| ---------------- | ----------------------------------------------------- |
| **Scope**        | Applies to subnets                                    |
| **Statefulness** | Stateless → Return traffic must be explicitly allowed |
| **Rules**        | Numbered list, processed in order                     |
| **Default**      | Default NACL → allows all traffic                     |
| **Custom NACL**  | Denies all traffic by default until you add rules     |
| **Association**  | One subnet → One NACL, but one NACL → Many subnets    |

---

# 🧱 **3. How NACLs Work**

NACLs process traffic in **two directions**:

### **Inbound rules** → Control traffic entering a subnet

### **Outbound rules** → Control traffic leaving a subnet

Rules are evaluated in order:

```
Rule 100 → Rule 101 → Rule 102 → ... → * → Deny
```

If a match is found, evaluation stops.
If no rule matches → *Implicit Deny*

---

# 🔥 **4. NACL vs Security Group (SG)**

| Feature          | NACL                 | Security Group                   |
| ---------------- | -------------------- | -------------------------------- |
| Level            | Subnet               | Instance/ENI                     |
| Stateful         | ❌ No                 | ✔️ Yes                           |
| Rules            | Allow + Deny         | Allow only                       |
| Process Order    | Rule number sequence | No order (all checked)           |
| Default Behavior | Deny all (custom)    | Deny all until allowed           |
| Association      | Subnet-wide          | Attached to individual resources |

**Use both together for layered security.**

---

# 🗂 **5. NACL Rule Structure**

Each rule contains:

* **Rule number** (1–32766)
* **Protocol** (TCP, UDP, ICMP, ALL)
* **Port range** (for TCP/UDP)
* **Source/Destination** (CIDR)
* **Action** (ALLOW / DENY)

### Example Rule

```
Rule #: 100
Protocol: TCP
Port range: 80
Source: 0.0.0.0/0
Action: ALLOW
```

---

# 🌐 **6. Typical NACL Configurations**

## ✔️ **Public Subnet NACL Example**

Used for resources like Internet-facing **EC2**, **ALB**, or **NAT Gateway**.

### Inbound Rules

| Rule # | Protocol  | Port       | Source    | Action          |
| ------ | --------- | ---------- | --------- | --------------- |
| 100    | TCP       | 80         | 0.0.0.0/0 | ALLOW           |
| 110    | TCP       | 443        | 0.0.0.0/0 | ALLOW           |
| 120    | Ephemeral | 1024–65535 | 0.0.0.0/0 | ALLOW           |
| *      | All       | All        | All       | DENY (implicit) |

### Outbound Rules

| Rule # | Protocol  | Port       | Destination | Action |
| ------ | --------- | ---------- | ----------- | ------ |
| 100    | TCP       | 80         | 0.0.0.0/0   | ALLOW  |
| 110    | TCP       | 443        | 0.0.0.0/0   | ALLOW  |
| 120    | Ephemeral | 1024–65535 | 0.0.0.0/0   | ALLOW  |

---

## ✔️ **Private Subnet NACL Example (for Application Tier)**

### Inbound

| Rule # | Protocol  | Port       | Source          | Action |
| ------ | --------- | ---------- | --------------- | ------ |
| 100    | TCP       | 3306       | App Subnet CIDR | ALLOW  |
| 110    | Ephemeral | 1024–65535 | NAT Gateway IP  | ALLOW  |

### Outbound

| Rule # | Protocol  | Port       | Destination    | Action |
| ------ | --------- | ---------- | -------------- | ------ |
| 100    | Ephemeral | 1024–65535 | NAT Gateway IP | ALLOW  |
| 110    | TCP       | 3306       | DB Subnet CIDR | ALLOW  |

---

## ✔️ **DB Subnet NACL (Highly Restricted)**

### Inbound

| Rule # | Protocol | Port | Source          | Action |
| ------ | -------- | ---- | --------------- | ------ |
| 100    | TCP      | 3306 | App Subnet CIDR | ALLOW  |

### Outbound

| Rule # | Protocol  | Port       | Destination     | Action |
| ------ | --------- | ---------- | --------------- | ------ |
| 100    | Ephemeral | 1024–65535 | App Subnet CIDR | ALLOW  |

---

# 👩‍💻 **7. NACL CLI Commands**

### Create NACL

```bash
aws ec2 create-network-acl --vpc-id vpc-123456
```

### Add Rule

```bash
aws ec2 create-network-acl-entry \
  --network-acl-id acl-123456 \
  --rule-number 100 \
  --protocol tcp \
  --port-range From=80,To=80 \
  --cidr-block 0.0.0.0/0 \
  --rule-action allow \
  --ingress
```

### Associate NACL to Subnet

```bash
aws ec2 associate-network-acl \
  --subnet-id subnet-12345 \
  --network-acl-id acl-12345
```

---

# 🏗 **8. Terraform Example**

```hcl
resource "aws_network_acl" "public" {
  vpc_id = aws_vpc.main.id

  ingress {
    rule_no = 100
    action = "allow"
    protocol = "tcp"
    from_port = 80
    to_port   = 80
    cidr_block = "0.0.0.0/0"
  }

  egress {
    rule_no = 100
    action = "allow"
    protocol = "tcp"
    from_port = 80
    to_port   = 80
    cidr_block = "0.0.0.0/0"
  }
}
```

---

# 🧩 **9. Real-Time Traffic Flow Example**

### Example Scenario: EC2 in Public Subnet accessing Internet

1. User accesses EC2 → **Inbound NACL** rule must ALLOW
2. EC2 responds → **Outbound NACL** must ALLOW
3. Since NACL is stateless → You **must allow both directions**

---

# 🛡 **10. Best Practices for NACLs**

🔹 Use NACLs for **coarse-grained**, subnet-level filtering
🔹 Use Security Groups for **fine-grained**, instance-level control
🔹 Always allow **ephemeral ports**
🔹 Keep rule numbering spaced (100, 110, 120...) for future rules
🔹 For public subnets → ALLOW HTTP/HTTPS and ephemeral
🔹 For database subnets → Restrict to app subnet only
🔹 Never rely ONLY on NACLs → combine with SGs

---

# ❗ **11. Common NACL Troubleshooting Issues**

### ❌ Issue: EC2 can't connect to Internet

* Missing outbound ephemeral rules
* Missing inbound ephemeral rules
* NAT Gateway IP not allowed

### ❌ Issue: RDS unreachable

* App subnet CIDR not allowed in DB inbound rule

### ❌ Issue: ALB health checks failing

* Health check port not permitted
* Inbound 1024–65535 ephemeral ports missing

---

# 🎯 **12. Exam Tips (AWS Solutions Architect)**

* NACL = **stateless**, SG = **stateful**
* NACL applies at **subnet level**
* **Default NACL** allows everything; **Custom NACL** denies everything
* Must explicitly allow **return traffic** (ephemeral ports)
* Use NACLs to block **specific IPs** (SGs cannot deny traffic)

---

# ✅ **Summary**

NACLs are essential for subnet-level security in a VPC. They offer powerful rule-based traffic filtering, especially when combined with Security Groups, giving a layered defense model for AWS networking.

Let me know if you'd like a diagram, cheat sheet, or also want **Firewall Manager**, **AWS WAF**, or **VPC Tra
