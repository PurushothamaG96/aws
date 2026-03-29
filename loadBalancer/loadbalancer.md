# AWS Load Balancer — Complete Setup Guide
> Step-by-step instructions, core concepts, and interview preparation

---

## Table of Contents

1. [What is a Load Balancer?](#1-what-is-a-load-balancer)
2. [Types of AWS Load Balancers](#2-types-of-aws-load-balancers)
3. [Key Concepts & Definitions](#3-key-concepts--definitions)
4. [Prerequisites](#4-prerequisites)
5. [Step-by-Step: Launch an Application Load Balancer (ALB)](#5-step-by-step-launch-an-application-load-balancer-alb)
6. [Step-by-Step: Launch a Network Load Balancer (NLB)](#6-step-by-step-launch-a-network-load-balancer-nlb)
7. [Configuring Health Checks](#7-configuring-health-checks)
8. [SSL/TLS Termination](#8-ssltls-termination)
9. [Auto Scaling Integration](#9-auto-scaling-integration)
10. [Monitoring & Logging](#10-monitoring--logging)
11. [Common Interview Questions & Answers](#11-common-interview-questions--answers)
12. [Troubleshooting](#12-troubleshooting)
13. [Cost Considerations](#13-cost-considerations)

---

## 1. What is a Load Balancer?

A **Load Balancer** is a managed AWS service that automatically distributes incoming application traffic across multiple targets — such as EC2 instances, containers, IP addresses, and Lambda functions — in one or more Availability Zones (AZs).

### Why Use a Load Balancer?

| Problem | How Load Balancer Solves It |
|---|---|
| Single point of failure | Routes traffic to healthy targets only |
| Traffic spikes | Distributes load across multiple instances |
| Zero-downtime deployments | Gradually shifts traffic to new instances |
| SSL management | Centralizes certificate handling |
| Geographic distribution | Routes to targets across multiple AZs |

---

## 2. Types of AWS Load Balancers

### 2.1 Application Load Balancer (ALB)
- **Layer**: OSI Layer 7 (HTTP/HTTPS)
- **Best for**: Web applications, microservices, container-based apps
- **Features**: Path-based routing, host-based routing, WebSocket support, HTTP/2, gRPC
- **Use case**: Route `/api/*` to one target group and `/static/*` to another

### 2.2 Network Load Balancer (NLB)
- **Layer**: OSI Layer 4 (TCP/UDP/TLS)
- **Best for**: Ultra-high performance, low latency, static IP requirements
- **Features**: Handles millions of requests per second, preserves source IP, supports static/elastic IPs
- **Use case**: Gaming servers, financial trading platforms, IoT

### 2.3 Gateway Load Balancer (GWLB)
- **Layer**: OSI Layer 3 (Network)
- **Best for**: Deploying, scaling, and running third-party virtual network appliances
- **Use case**: Firewalls, intrusion detection systems (IDS/IPS), deep packet inspection

### 2.4 Classic Load Balancer (CLB) — Legacy
- **Layer**: Layer 4 and Layer 7
- **Status**: ⚠️ Not recommended for new deployments. AWS recommends migrating to ALB or NLB.

---

## 3. Key Concepts & Definitions

| Term | Definition |
|---|---|
| **Target Group** | A logical grouping of targets (EC2, IPs, Lambda) that receive traffic from the load balancer |
| **Listener** | A process that checks for connection requests using a configured port and protocol (e.g., port 443, HTTPS) |
| **Listener Rule** | Conditions and actions that determine how the load balancer routes requests |
| **Health Check** | Periodic requests sent to targets to verify they can receive traffic |
| **Availability Zone (AZ)** | Isolated locations within a region; load balancers spread traffic across AZs for fault tolerance |
| **Cross-Zone Load Balancing** | Distributes traffic evenly across all registered targets in all enabled AZs |
| **Sticky Sessions** | Routes a user's requests to the same target for the duration of a session (cookie-based) |
| **SSL Termination** | The load balancer decrypts HTTPS traffic and forwards plain HTTP to backend targets |
| **Access Logs** | Detailed logs of requests sent to the load balancer, stored in S3 |
| **Connection Draining** | (Deregistration Delay) Allows in-flight requests to complete before deregistering a target |
| **Security Group** | Virtual firewall controlling inbound/outbound traffic to the load balancer |
| **VPC** | Virtual Private Cloud — the isolated network where your load balancer and targets reside |
| **Weighted Target Groups** | ALB feature to send a percentage of traffic to different target groups (useful for canary deployments) |

---

## 4. Prerequisites

Before launching a load balancer, ensure you have:

- [ ] An **AWS account** with sufficient IAM permissions (`elasticloadbalancing:*`)
- [ ] A **VPC** with at least **2 subnets in different Availability Zones** (required for high availability)
- [ ] **Internet Gateway** attached to VPC (for internet-facing load balancers)
- [ ] At least **2 running EC2 instances** (or other targets) to register
- [ ] A **Security Group** for the load balancer (allow inbound 80/443 from internet)
- [ ] A **Security Group** for EC2 instances (allow inbound from load balancer's SG only)
- [ ] (Optional) An **SSL/TLS certificate** in AWS Certificate Manager (ACM) for HTTPS

### IAM Permissions Required

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "elasticloadbalancing:*",
        "ec2:DescribeInstances",
        "ec2:DescribeSubnets",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeVpcs",
        "acm:ListCertificates",
        "acm:DescribeCertificate"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## 5. Step-by-Step: Launch an Application Load Balancer (ALB)

### Step 1 — Open the EC2 Console

1. Sign in to **AWS Management Console**
2. Navigate to **EC2** → Under "Load Balancing" in the left sidebar, click **Load Balancers**
3. Click **Create load balancer**
4. Select **Application Load Balancer** → Click **Create**

### Step 2 — Configure Basic Settings

```
Name:          my-app-alb
Scheme:        Internet-facing  (or Internal for private traffic)
IP address type: IPv4           (or Dualstack for IPv6 support)
```

> **Internet-facing**: Has a public DNS name resolved to public IPs. Use for public websites.  
> **Internal**: Routes traffic within the VPC only. Use for microservice-to-microservice communication.

### Step 3 — Configure Network Mapping

1. Select your **VPC**
2. Select **at least 2 Availability Zones** (e.g., `us-east-1a`, `us-east-1b`)
3. For each AZ, select the appropriate **public subnet**

> ⚠️ **Best Practice**: Always select subnets in a minimum of 2 AZs for high availability.

### Step 4 — Configure Security Groups

1. Remove the default security group
2. Select or create a security group with:

```
Inbound Rules:
  Type: HTTP   | Port: 80   | Source: 0.0.0.0/0
  Type: HTTPS  | Port: 443  | Source: 0.0.0.0/0

Outbound Rules:
  Type: All traffic | Destination: 0.0.0.0/0
```

### Step 5 — Configure Listeners and Routing

A **Listener** defines which port and protocol the ALB accepts traffic on.

**Add Listener:**
```
Protocol: HTTP
Port:     80
Default Action: Forward to → [Target Group] (create below)
```

**Create a Target Group:**
```
Target type:       Instances
Name:              my-app-tg
Protocol:          HTTP
Port:              80
VPC:               [Your VPC]
Protocol version:  HTTP1
```

**Health check settings:**
```
Protocol:           HTTP
Path:               /health   (or / if no health endpoint)
Healthy threshold:  2
Unhealthy threshold: 3
Timeout:            5 seconds
Interval:           30 seconds
Success codes:      200
```

### Step 6 — Register Targets

1. Select your EC2 instances from the list
2. Set port to **80** (or your app's port)
3. Click **Include as pending below**
4. Click **Register pending targets**

### Step 7 — Add HTTPS Listener (Recommended)

1. Click **Add listener**
2. Set Protocol: **HTTPS**, Port: **443**
3. Default action: **Forward to** your target group
4. Under **Secure listener settings**:
   - Security policy: `ELBSecurityPolicy-TLS13-1-2-2021-06` (recommended)
   - Certificate source: **ACM**
   - Select your certificate
5. Add a redirect rule on port 80 → port 443 (HTTP → HTTPS redirect)

### Step 8 — Add Tags (Optional but Recommended)

```
Key: Environment   Value: production
Key: Team          Value: platform
Key: Project       Value: my-app
```

### Step 9 — Review and Create

1. Review all settings
2. Click **Create load balancer**
3. Wait ~2-3 minutes for the ALB to become **Active**

### Step 10 — Verify

```bash
# Get the DNS name of your ALB
aws elbv2 describe-load-balancers \
  --names my-app-alb \
  --query 'LoadBalancers[0].DNSName' \
  --output text

# Test connectivity
curl -I http://<ALB-DNS-NAME>

# Expected response
HTTP/1.1 200 OK
```

---

## 6. Step-by-Step: Launch a Network Load Balancer (NLB)

### Step 1 — Create the NLB

1. Navigate to **EC2 → Load Balancers → Create load balancer**
2. Select **Network Load Balancer** → Click **Create**

### Step 2 — Basic Configuration

```
Name:          my-nlb
Scheme:        Internet-facing
IP address type: IPv4
```

### Step 3 — Network Mapping

1. Select your **VPC**
2. Select **Availability Zones** and subnets
3. (Optional) Assign **Elastic IP addresses** per AZ for static IPs

> 💡 NLB supports static IPs — a key advantage over ALB when clients require IP whitelisting.

### Step 4 — Listeners

```
Protocol: TCP
Port:     80
```

> NLB supports: **TCP, UDP, TLS, TCP_UDP**

### Step 5 — Create Target Group

```
Target type:       Instances
Name:              my-nlb-tg
Protocol:          TCP
Port:              80
VPC:               [Your VPC]
```

**Health check:**
```
Protocol: TCP      (or HTTP if you want HTTP-level health checks)
Interval: 10 seconds
Healthy threshold: 3
Unhealthy threshold: 3
```

### Step 6 — Register Targets and Create

Same as ALB — select instances, register, and create.

---

## 7. Configuring Health Checks

Health checks ensure the load balancer only routes traffic to **healthy targets**.

### Health Check Parameters

| Parameter | Recommended Value | Description |
|---|---|---|
| Protocol | HTTP / HTTPS | Use same protocol as your app |
| Path | `/health` or `/ping` | Dedicated health endpoint (return 200 OK) |
| Port | `traffic port` | Same port as target receives traffic |
| Healthy threshold | 2 | Consecutive checks to mark healthy |
| Unhealthy threshold | 2 | Consecutive failures to mark unhealthy |
| Timeout | 5s | Max time to wait for a response |
| Interval | 30s | Time between health checks |
| Success codes | 200 | HTTP codes considered healthy |

### Best Practice: Implement a `/health` Endpoint

```python
# Python Flask example
@app.route('/health')
def health():
    # Check DB connection, cache, etc.
    return {'status': 'healthy'}, 200
```

```javascript
// Node.js Express example
app.get('/health', (req, res) => {
  res.status(200).json({ status: 'healthy', timestamp: new Date() });
});
```

---

## 8. SSL/TLS Termination

### How SSL Termination Works

```
Client → [HTTPS: Encrypted] → ALB → [HTTP: Decrypted] → EC2 Instances
```

The ALB handles all SSL/TLS encryption and decryption, reducing CPU load on backend servers.

### Setting Up ACM Certificate

```bash
# Request a public certificate via AWS CLI
aws acm request-certificate \
  --domain-name "*.yourdomain.com" \
  --validation-method DNS \
  --region us-east-1

# List certificates
aws acm list-certificates --region us-east-1
```

### HTTP to HTTPS Redirect Rule

In your HTTP listener (port 80), set the default action to:

```
Action type: Redirect
Protocol:    HTTPS
Port:        443
Status code: 301 (Permanent redirect)
```

---

## 9. Auto Scaling Integration

Load balancers work seamlessly with **Auto Scaling Groups (ASG)** to automatically add/remove targets.

### Attaching ALB to Auto Scaling Group

```bash
# Attach a target group to an Auto Scaling Group
aws autoscaling attach-load-balancer-target-groups \
  --auto-scaling-group-name my-asg \
  --target-group-arns arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/my-app-tg/abc123

# Verify health check type is ELB
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name my-asg \
  --health-check-type ELB \
  --health-check-grace-period 300
```

> ⚠️ Set `health-check-grace-period` to give new instances time to start before health checks begin.

### Flow: Scale-Out Event

```
CloudWatch Alarm (CPU > 70%)
    ↓
Auto Scaling adds EC2 instance
    ↓
Instance registers with Target Group
    ↓
Health check passes
    ↓
ALB starts routing traffic to new instance
```

---

## 10. Monitoring & Logging

### Enable Access Logs (ALB)

```bash
# Create S3 bucket for logs
aws s3 mb s3://my-alb-logs-bucket

# Add bucket policy for load balancer to write logs
# (See AWS docs for the exact bucket policy per region)

# Enable access logs on your ALB
aws elbv2 modify-load-balancer-attributes \
  --load-balancer-arn arn:aws:elasticloadbalancing:... \
  --attributes \
    Key=access_logs.s3.enabled,Value=true \
    Key=access_logs.s3.bucket,Value=my-alb-logs-bucket \
    Key=access_logs.s3.prefix,Value=my-app-alb
```

### Key CloudWatch Metrics to Monitor

| Metric | Description | Alert Threshold |
|---|---|---|
| `RequestCount` | Total requests per period | Sudden drops/spikes |
| `TargetResponseTime` | Latency from ALB to targets | > 1s (P99) |
| `HTTPCode_ELB_5XX_Count` | Server errors from ALB | > 0 |
| `HTTPCode_Target_5XX_Count` | 5xx errors from targets | > 1% of requests |
| `HealthyHostCount` | Number of healthy targets | < minimum required |
| `UnHealthyHostCount` | Number of unhealthy targets | > 0 |
| `ActiveConnectionCount` | Active TCP connections | Baseline + 3σ |
| `NewConnectionCount` | New TCP connections per period | Sudden spikes |

### Create CloudWatch Alarm (CLI)

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "ALB-UnhealthyHosts" \
  --alarm-description "Alert when unhealthy host count > 0" \
  --metric-name UnHealthyHostCount \
  --namespace AWS/ApplicationELB \
  --statistic Average \
  --period 60 \
  --evaluation-periods 2 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --dimensions \
    Name=LoadBalancer,Value=app/my-app-alb/abc123 \
    Name=TargetGroup,Value=targetgroup/my-app-tg/def456 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:my-alert-topic
```

---

## 11. Common Interview Questions & Answers

### 🔹 Q1: What is the difference between ALB and NLB?

| Criteria | ALB | NLB |
|---|---|---|
| OSI Layer | Layer 7 (HTTP/HTTPS) | Layer 4 (TCP/UDP/TLS) |
| Routing | Path, host, query string, headers | IP + Port only |
| Static IP | ❌ No (uses DNS) | ✅ Yes (Elastic IP) |
| Latency | Higher (~ms) | Ultra-low (μs) |
| WebSockets | ✅ Yes | ✅ Yes |
| Source IP preservation | Via `X-Forwarded-For` header | ✅ Native |
| Use case | Web apps, APIs, microservices | High-perf, TCP/UDP, IoT |

---

### 🔹 Q2: What happens when a target fails a health check?

The load balancer marks the target as **unhealthy** after the configured number of consecutive failed checks. Once unhealthy, the ALB/NLB stops sending new requests to that target. It continues health checks, and when the target passes the **healthy threshold** consecutively, it is marked healthy again and traffic resumes. **In-flight requests** are not immediately terminated — this is handled by **connection draining** (deregistration delay, default 300 seconds).

---

### 🔹 Q3: What is Connection Draining / Deregistration Delay?

When a target is deregistered (e.g., during scale-in), the load balancer waits up to the **deregistration delay** period (default: 300 seconds) before forcibly closing connections. During this period, no new requests are sent to the deregistering target, but existing in-flight requests are allowed to complete. This ensures zero request drops during rolling deployments or scale-in events.

---

### 🔹 Q4: What is Cross-Zone Load Balancing?

By default, each load balancer node distributes traffic only to targets within its own AZ. With **Cross-Zone Load Balancing enabled**, each node distributes traffic evenly across all registered targets in all enabled AZs, regardless of AZ.

- **ALB**: Enabled by default, no extra charge
- **NLB/GWLB**: Disabled by default; enabling incurs data transfer charges between AZs

---

### 🔹 Q5: How do you achieve zero-downtime deployments with an ALB?

Use **rolling deployments** with Auto Scaling:
1. Deploy new instances in the ASG
2. New instances pass health checks and are registered with the target group
3. Old instances are deregistered — connection draining allows in-flight requests to complete
4. Terminate old instances

Alternatively, use **weighted target groups** in ALB for **canary deployments**:
- 90% traffic → stable target group
- 10% traffic → new version target group
- Gradually shift weight as confidence increases

---

### 🔹 Q6: How does an ALB handle HTTPS/SSL?

ALB supports **SSL termination**: the ALB decrypts incoming HTTPS traffic using a certificate stored in **AWS Certificate Manager (ACM)** or IAM. After decryption, it forwards plain HTTP to the backend targets. Optionally, ALB-to-target traffic can also be encrypted (end-to-end TLS) by configuring the target group to use HTTPS.

---

### 🔹 Q7: What is Sticky Sessions and when would you use it?

**Sticky Sessions** (Session Affinity) route a user's requests to the same target throughout a session using a cookie. ALB uses the `AWSALB` cookie by default.

**Use when**: Your application stores session state **locally on the server** (e.g., in-memory session store).

**Avoid when**: Your application is stateless, or you use a distributed session store like **Redis/ElastiCache** — which is the better architectural pattern.

---

### 🔹 Q8: Can an ALB route traffic to targets outside AWS?

Yes. ALB supports **IP-type target groups**, which allows routing to any IP address — including on-premises servers reachable via **AWS Direct Connect** or **VPN**. This enables hybrid cloud architectures.

---

### 🔹 Q9: What is the difference between an internet-facing and internal load balancer?

| | Internet-facing | Internal |
|---|---|---|
| DNS resolution | Resolves to public IPs | Resolves to private IPs |
| Accessible from | Public internet | Within VPC / connected networks |
| Use case | Public websites, APIs | Microservice-to-microservice, backend tiers |
| Requires | Internet Gateway in VPC | No Internet Gateway needed |

---

### 🔹 Q10: How would you troubleshoot HTTP 502/504 errors from an ALB?

| Error Code | Meaning | Common Causes |
|---|---|---|
| **502 Bad Gateway** | ALB received an invalid response from target | App crashed, wrong port, app not running |
| **503 Service Unavailable** | No healthy targets | All targets failed health checks |
| **504 Gateway Timeout** | Target didn't respond in time | App is too slow, health check path wrong, security group blocking |

**Troubleshooting steps:**
1. Check **CloudWatch metrics**: `HealthyHostCount`, `HTTPCode_Target_5XX`
2. Inspect **ALB Access Logs** in S3 — look for `target_status_code` field
3. SSH into a target instance and test the app directly: `curl localhost:80/health`
4. Verify **Security Groups**: Does EC2 SG allow inbound from ALB SG on the app port?
5. Check **Health Check** configuration: correct path, port, and success codes?
6. Check **Deregistration Delay** — did you check too soon after deploying?

---

### 🔹 Q11: What is the ALB Listener Rule evaluation order?

ALB evaluates listener rules in **priority order** (lowest number = highest priority). The **default rule** (priority: last) is always evaluated last and acts as a catch-all. Rules can match on: host headers, HTTP methods, path patterns, query strings, source IPs, and HTTP headers.

```
Rule Priority 1: path /api/*    → Forward to api-target-group
Rule Priority 2: path /admin/*  → Forward to admin-target-group
Rule Priority 3: host app.com   → Forward to main-target-group
Default Rule:                   → Return 404 fixed response
```

---

## 12. Troubleshooting

### Target Shows "Unhealthy" in Target Group

```bash
# Check target health details
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:...

# Common fixes:
# 1. Verify app is running on the instance
ssh ec2-user@<instance-ip>
curl localhost:80/health

# 2. Check security group allows traffic from ALB
# EC2 SG must allow inbound from ALB's SG on the app port

# 3. Verify health check path returns 200
# 4. Check if instance has enough capacity (CPU/Memory)
```

### Cannot Access Application via ALB DNS

```bash
# 1. Verify ALB state is "active"
aws elbv2 describe-load-balancers --names my-app-alb

# 2. Verify DNS resolves
nslookup my-app-alb-123456.us-east-1.elb.amazonaws.com

# 3. Check ALB security group allows inbound 80/443
# 4. Verify subnets are in correct VPC
# 5. Check Internet Gateway is attached to VPC (for internet-facing ALB)
```

---

## 13. Cost Considerations

AWS charges for load balancers based on:

| Component | ALB | NLB |
|---|---|---|
| **Hourly charge** | ~$0.008/hr per LCU | ~$0.006/hr per NLCU |
| **LCU/NLCU** | Based on connections, requests, bandwidth | Based on connections, bytes |
| **Data transfer** | Standard AWS rates | Standard + AZ cross-zone fees |
| **ACM certificates** | Free for ALB use | Free for ALB use |

### Cost Optimization Tips

- **Delete unused load balancers** — idle ALBs still incur hourly charges
- **Use one ALB for multiple apps** — use path/host-based routing instead of multiple ALBs
- **Disable cross-zone load balancing on NLB** if not needed (avoids AZ data transfer costs)
- **Set appropriate idle timeout** — default 60s; reduce for short-lived connections
- **Use target group weights** for gradual rollouts instead of maintaining multiple ALBs

---

## Quick Reference Commands

```bash
# List all load balancers
aws elbv2 describe-load-balancers

# List target groups
aws elbv2 describe-target-groups

# Check target health
aws elbv2 describe-target-health --target-group-arn <arn>

# Describe listeners
aws elbv2 describe-listeners --load-balancer-arn <arn>

# Describe listener rules
aws elbv2 describe-rules --listener-arn <arn>

# Delete a load balancer
aws elbv2 delete-load-balancer --load-balancer-arn <arn>

# Get ALB DNS name
aws elbv2 describe-load-balancers \
  --names my-app-alb \
  --query 'LoadBalancers[0].DNSName' \
  --output text
```

---

*Document version: 1.0 | AWS Region: us-east-1 (adapt as needed) | Last updated: March 2026*