# Subnets, Security Groups & Route Tables — In-depth Reference

This document provides comprehensive definitions, practical usage, CLI examples, configuration snippets (CloudFormation / Terraform), common architectures, troubleshooting notes, and exam tips for **Subnets**, **Security Groups (SGs)**, and **Route Tables** in Amazon VPC.

---

# 1. Subnets

## Definition

A **subnet** is a contiguous range of IP addresses in your VPC. Subnets partition a VPC CIDR into smaller IP ranges and map directly to a single Availability Zone (AZ). Each subnet is associated with a route table which determines where network traffic from the subnet is directed.

## Key Concepts

* **VPC CIDR**: The overall IP address block for a VPC (e.g. `10.0.0.0/16`).
* **Subnet CIDR**: A subset (e.g. `10.0.1.0/24`) that lives inside the VPC.
* **AZ binding**: Each subnet exists in exactly one Availability Zone.
* **Public vs Private vs Isolated**:

  * **Public subnet** — has a route to an Internet Gateway (IGW). Instances with a public IP in this subnet can be reached from the internet (if SG allows).
  * **Private subnet** — no direct route to IGW. Instances can access the internet via NAT (NAT Gateway or NAT Instance).
  * **Isolated subnet** — no route to internet or NAT. Used for sensitive resources that should not have internet connectivity.

## Use cases

* Distribute resources across AZs for high availability (create subnets in multiple AZs).
* Isolate layers (web, app, data) into separate subnets.
* Place public-facing components (ALB, NAT GW) in public subnets.
* Place databases, caches, and internal services in private/isolated subnets.

## Best Practices

* Allocate CIDR blocks with future growth in mind (avoid too small ranges).
* Use /24 subnets (256 IPs) for AZ-level resources as a sensible default.
* Spread resources across at least two AZs for HA.
* Use separate route tables for public and private subnets.
* Tag subnets with `Name`, `Environment`, `Tier`, `AZ` for clarity.

## CLI Examples (AWS CLI)

Create a subnet:

```bash
aws ec2 create-subnet --vpc-id vpc-0123456789abcdef0 --cidr-block 10.0.1.0/24 --availability-zone us-east-1a
```

Modify to enable Auto-assign Public IP on a subnet (so instances get public IPs by default):

```bash
aws ec2 modify-subnet-attribute --subnet-id subnet-0123456789abcdef0 --map-public-ip-on-launch
```

## CloudFormation snippet

```yaml
MySubnetPublic:
  Type: AWS::EC2::Subnet
  Properties:
    VpcId: !Ref VPC
    CidrBlock: 10.0.1.0/24
    AvailabilityZone: us-east-1a
    MapPublicIpOnLaunch: true
    Tags:
      - Key: Name
        Value: my-vpc-public-subnet-1
```

## Terraform snippet

```hcl
resource "aws_subnet" "public_1" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"
  map_public_ip_on_launch = true
  tags = { Name = "public-1" }
}
```

## Troubleshooting

* **No public access to instance**: Check subnet `mapPublicIpOnLaunch`, instance public IP, SG inbound rules, NACLs, and route table to IGW.
* **Instance cannot reach internet**: Ensure route table has `0.0.0.0/0` → NAT GW (private) or IGW (public); if using NAT, the NAT must be in a public subnet.

---

# 2. Security Groups (SGs)

## Definition

A **Security Group** is a virtual firewall that controls inbound and outbound traffic for AWS resources (EC2, RDS, ENIs, Lambda in VPC, etc.). Security groups are attached to network interfaces and evaluate traffic by matching rules. They are **stateful** — if an incoming packet is allowed, the response is automatically allowed.

## Key Properties

* **Stateful**: Return traffic allowed automatically.
* **Allow-only rules**: Security Groups cannot contain explicit deny rules.
* **Applied to ENIs**: You attach one or more SGs to an Elastic Network Interface (ENI) or resource.
* **Rule evaluation**: All rules are evaluated; if any rule allows the traffic, it is allowed.
* **Default security group**: Automatically created for a VPC. Its default behavior allows inbound traffic from other members of the same default SG and allows all outbound traffic.

## Typical Usage Patterns

* **Web tier**: SG allowing HTTP/HTTPS from `0.0.0.0/0` (or specific CIDR), allowing SSH from admin IP.
* **App tier**: SG allowing traffic from Web SG (reference by SG ID) on application port (e.g., 8080).
* **DB tier**: SG allowing traffic only from App SG on DB port (e.g., 3306).

## Examples

### Security Group rules conceptual example

* `sg-web` inbound: TCP 80,443 from 0.0.0.0/0
* `sg-web` inbound: TCP 22 from 203.0.113.10/32
* `sg-app` inbound: TCP 8080 from sg-web (source = sg-web id)
* `sg-db` inbound: TCP 3306 from sg-app

### AWS CLI: create security group

```bash
aws ec2 create-security-group --group-name my-web-sg --description "Web SG" --vpc-id vpc-0123456789abcdef0
```

Add inbound rule for HTTP:

```bash
aws ec2 authorize-security-group-ingress --group-id sg-0123456789abcdef0 --protocol tcp --port 80 --cidr 0.0.0.0/0
```

## CloudFormation snippet

```yaml
WebSecurityGroup:
  Type: AWS::EC2::SecurityGroup
  Properties:
    GroupDescription: Allow HTTP and SSH
    VpcId: !Ref VPC
    SecurityGroupIngress:
      - IpProtocol: tcp
        FromPort: 80
        ToPort: 80
        CidrIp: 0.0.0.0/0
      - IpProtocol: tcp
        FromPort: 22
        ToPort: 22
        CidrIp: 203.0.113.10/32
```

## Terraform snippet

```hcl
resource "aws_security_group" "web" {
  name        = "sg-web"
  description = "Allow HTTP and SSH"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["203.0.113.10/32"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

## Security Group Best Practices

* **Least privilege**: Only open required ports and sources.
* **Use SG references** (allow traffic from another SG) rather than opening broad CIDRs when possible.
* **Use separate SGs per tier** (web, app, db) to simplify rules.
* **Avoid 0.0.0.0/0 for SSH** — restrict to admin IPs.
* **Document SG rules** with names and tags.

## Common Gotchas & Troubleshooting

* **SGs do not block return traffic** — remember they are stateful.
* **Order of rules doesn't matter** — any rule that allows traffic will permit it.
* **SG by ID for internal communication**: When specifying another SG as source, use the SG ID rather than CIDR.
* **Changing SG rules is immediate** — but check instance-level OS firewalls (iptables) or application-level blocks if connections still fail.

---

# 3. Route Tables

## Definition

A **Route Table** is a set of rules, called routes, that determine where network traffic from your subnet or gateway is directed.

## Components

* **Route**: Maps a destination CIDR (like `0.0.0.0/0` or `10.0.0.0/16`) to a target (Internet Gateway, NAT Gateway, VPC Peering Connection, Transit Gateway, Virtual Private Gateway, Network Interface, or NAT instance).
* **Main route table**: Each VPC has a main route table that automatically applies to subnets that are not explicitly associated with another route table.
* **Subnet association**: Each subnet must be associated with a single route table. Many subnets can share the same route table.

## Common Targets

* **local** — internal VPC routing (auto-created)
* **igw-xxxxx** — Internet Gateway
* **nat-xxxxx** — NAT Gateway
* **vgw-xxxxx** — Virtual Private Gateway (for Site-to-Site VPN)
* **pcx-xxxxx** — VPC Peering Connection
* **tgw-xxxxx** — Transit Gateway

## Typical Route Table Configurations

* **Public route table** (for public subnets):

  * `10.0.0.0/16` → `local`
  * `0.0.0.0/0` → `igw-xxxxx` (Internet Gateway)
* **Private route table** (for private subnets):

  * `10.0.0.0/16` → `local`
  * `0.0.0.0/0` → `nat-xxxxx` (NAT Gateway)

## CLI Examples

Create a route table and associate a subnet:

```bash
# create
aws ec2 create-route-table --vpc-id vpc-0123456789abcdef0

# associate
aws ec2 associate-route-table --route-table-id rtb-0123456789abcdef0 --subnet-id subnet-0123456789abcdef0

# create route to IGW
aws ec2 create-route --route-table-id rtb-0123456789abcdef0 --destination-cidr-block 0.0.0.0/0 --gateway-id igw-0123456789abcdef0
```

## CloudFormation snippet

```yaml
PublicRouteTable:
  Type: AWS::EC2::RouteTable
  Properties:
    VpcId: !Ref VPC

PublicRoute:
  Type: AWS::EC2::Route
  DependsOn: VPCGatewayAttachment
  Properties:
    RouteTableId: !Ref PublicRouteTable
    DestinationCidrBlock: 0.0.0.0/0
    GatewayId: !Ref InternetGateway

PublicSubnetRouteTableAssociation:
  Type: AWS::EC2::SubnetRouteTableAssociation
  Properties:
    SubnetId: !Ref PublicSubnet
    RouteTableId: !Ref PublicRouteTable
```

## Terraform snippet

```hcl
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags = { Name = "public-rt" }
}

resource "aws_route_table_association" "public_assoc" {
  subnet_id      = aws_subnet.public_1.id
  route_table_id = aws_route_table.public.id
}
```

## Advanced Routing Use Cases

* **VPC Peering**: Add `pcx-xxxxx` route in both VPCs route tables to enable cross-VPC communication.
* **Transit Gateway**: Use TGW routes to centralize connectivity across dozens of VPCs and on-prem networks.
* **VPN/Direct Connect**: Routes to `vgw-xxxxx` for traffic that should traverse the VPN/Direct Connect.
* **Blackhole routes**: If the target is deleted the route becomes a blackhole — remove or update the route.

## Troubleshooting

* **No internet access**: Ensure route table for subnet has `0.0.0.0/0` → IGW (public) or NAT (private), and check IGW attached to VPC / NAT in public subnet.
* **Cross-VPC traffic not working**: Confirm peering route exists on both sides, check SGs/NACLs, and remember there is **no transitive routing** via peering.
* **Subnet still using main route table**: Associate the subnet explicitly to a custom route table to override the main route table.

---

# 4. Patterns & Examples

## Typical 3-tier architecture (multi-AZ)

* **VPC**: `10.0.0.0/16`
* **Public subnets (2 AZs)**: `10.0.1.0/24`, `10.0.3.0/24` — for ALB, NAT GW
* **Private app subnets (2 AZs)**: `10.0.2.0/24`, `10.0.4.0/24` — for EC2 app servers
* **Private DB subnets (2 AZs, isolated)**: `10.0.5.0/24`, `10.0.6.0/24` — for RDS with no IGW or NAT

Traffic flow:

* Users → ALB (public SG) → Web EC2 (private SG allowing ALB) → App DB (DB SG allowing app SG)
* App EC2 outbound internet → NAT Gateway in public subnet

## VPC Endpoint usage (S3 / DynamoDB)

* For private access to S3 from private subnets, create a **Gateway VPC Endpoint** for `com.amazonaws.<region>.s3` and add the endpoint to the route tables of your private subnets.
* Interface endpoints provide private ENIs in subnet(s) for other AWS services.

---

# 5. Exam-Focused Tips

* **Subnet**: Tied to an AZ; public vs private determined by route to IGW.
* **Security Group**: Stateful, attach to ENIs, only allow rules, can reference other SGs.
* **Route Table**: Explicit routes; `local` route cannot be removed; main RT applies to subnets not explicitly associated.
* **NAT Gateway vs NAT Instance**: NAT GW is managed, highly available (within an AZ) and preferred.
* **VPC Peering**: No transitive routing; you must add routes for both sides.
* **VPC Endpoint**: Use for private access to AWS services—gateway endpoints for S3/DynamoDB and interface endpoints for most other services.

---

# 6. Handy Cheatsheet (One-liners)

* **Subnet**: "AZ-scoped IP block in a VPC."
* **Security Group**: "Stateful instance firewall — allow only."
* **Route Table**: "Where traffic from a subnet goes (IGW/NAT/VGW/TGW/PCX)."

---

# 7. References & Further Reading

* AWS VPC Documentation: Amazon Virtual Private Cloud (VPC)
* AWS Security Group Concepts
* AWS Route Tables and Routing

---

*End of document.*
