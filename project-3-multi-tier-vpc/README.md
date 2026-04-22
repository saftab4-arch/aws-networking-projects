# 🏗️ AWS Multi-Tier VPC Network Architecture

## Project Overview

This project demonstrates a production-grade AWS network architecture featuring a multi-tier VPC with public, private, and isolated subnets across two Availability Zones. It includes a NAT Gateway for private subnet internet access, a Bastion Host for secure SSH access, and a VPC Endpoint for private S3 connectivity — all tested end-to-end.

> **Skills demonstrated:** VPC design, subnet segmentation, routing, security groups, IAM roles, NAT Gateway, VPC Endpoints, EC2 jump-box pattern, and defense-in-depth security principles.

---

## Architecture Diagram

```
                        Internet
                           │
                    ┌──────▼──────┐
                    │     IGW     │
                    └──────┬──────┘
                           │
          ┌────────────────▼─────────────────┐
          │         network-lab-vpc           │
          │           10.0.0.0/16             │
          │                                   │
          │  ┌─────────────────────────────┐  │
          │  │      PUBLIC SUBNETS          │  │
          │  │  AZ1: 10.0.1.0/24           │  │
          │  │  AZ2: 10.0.4.0/24           │  │
          │  │  [Bastion Host] [NAT GW]    │  │
          │  └─────────────────────────────┘  │
          │                │                  │
          │  ┌─────────────▼───────────────┐  │
          │  │      PRIVATE SUBNETS         │  │
          │  │  AZ1: 10.0.2.0/24           │  │
          │  │  AZ2: 10.0.5.0/24           │  │
          │  │  [App Server]               │  │
          │  └─────────────────────────────┘  │
          │                │                  │
          │  ┌─────────────▼───────────────┐  │
          │  │      ISOLATED SUBNETS        │  │
          │  │  AZ1: 10.0.3.0/24           │  │
          │  │  AZ2: 10.0.6.0/24           │  │
          │  │  [DB Server]                │  │
          │  └─────────────────────────────┘  │
          └───────────────────────────────────┘
                           │
                    ┌──────▼──────┐
                    │  S3 (via    │
                    │  VPC Endpt) │
                    └─────────────┘
```

---

## What We Built

| Component | Resource | Purpose |
|---|---|---|
| VPC | `network-lab-vpc` 10.0.0.0/16 | Private network container |
| Public Subnets | AZ1: 10.0.1.0/24, AZ2: 10.0.4.0/24 | Internet-facing resources |
| Private Subnets | AZ1: 10.0.2.0/24, AZ2: 10.0.5.0/24 | App servers with outbound internet |
| Isolated Subnets | AZ1: 10.0.3.0/24, AZ2: 10.0.6.0/24 | Database tier, zero internet |
| Internet Gateway | `network-lab-igw` | Entry/exit door for public internet |
| NAT Gateway | `network-lab-nat` | Outbound-only internet for private subnet |
| Bastion Host | EC2 in public subnet | Secure SSH jump box |
| App Server | EC2 in private subnet | Application tier |
| DB Server | EC2 in isolated subnet | Database tier |
| VPC Endpoint | S3 Gateway Endpoint | Private S3 access without NAT |

---

## Step-by-Step Build Guide

### Step 1: Create the VPC

**Go to:** AWS Console → VPC → Create VPC

| Field | Value |
|---|---|
| Resources to create | VPC only |
| Name tag | `network-lab-vpc` |
| IPv4 CIDR | `10.0.0.0/16` |
| IPv6 | No IPv6 |
| Tenancy | Default |

After creation → **Actions → Edit VPC Settings:**
- ✅ Enable DNS hostnames
- ✅ Enable DNS resolution

**Why:** A VPC is your private network container in AWS. The `/16` gives 65,536 IPs to carve into subnets. DNS settings are required for EC2 hostnames and VPC Endpoint functionality.

---

### Step 2: Create Subnets (6 Total)

**Go to:** VPC Console → Subnets → Create Subnet → Select `network-lab-vpc`

Add all 6 subnets in one shot:

| Name | AZ | CIDR |
|---|---|---|
| `public-subnet-az1` | us-east-1a | 10.0.1.0/24 |
| `private-subnet-az1` | us-east-1a | 10.0.2.0/24 |
| `isolated-subnet-az1` | us-east-1a | 10.0.3.0/24 |
| `public-subnet-az2` | us-east-1b | 10.0.4.0/24 |
| `private-subnet-az2` | us-east-1b | 10.0.5.0/24 |
| `isolated-subnet-az2` | us-east-1b | 10.0.6.0/24 |

**Enable Auto-assign Public IP** on both public subnets:
- Select subnet → Actions → Edit subnet settings → ✅ Auto-assign public IPv4

**Why:** Three tiers create defense in depth. Public = internet-facing, Private = app layer with outbound only, Isolated = database layer with no internet at all. Two AZs provide high availability — if one AZ fails, the other keeps running.

> **Note:** AZ2 subnets are provisioned and ready. In production, App and DB servers would be deployed across both AZs behind an Application Load Balancer with two NAT Gateways for full HA.

---

### Step 3: Internet Gateway

**Go to:** VPC Console → Internet Gateways → Create Internet Gateway

| Field | Value |
|---|---|
| Name tag | `network-lab-igw` |

After creation → **Actions → Attach to VPC → `network-lab-vpc`**

**Why:** The IGW is the highway entrance ramp for your VPC. It enables two-way traffic between public subnets and the internet. Attaching it to the VPC does NOT automatically give all subnets internet access — that requires route table configuration in the next step.

---

### Step 4: Route Tables

**Go to:** VPC Console → Route Tables → Create Route Table

Create 3 route tables:

#### Public Route Table
| Field | Value |
|---|---|
| Name | `public-rt` |
| VPC | `network-lab-vpc` |

Add route: `0.0.0.0/0 → network-lab-igw`

Associate subnets: `public-subnet-az1`, `public-subnet-az2`

#### Private Route Table
| Field | Value |
|---|---|
| Name | `private-rt` |
| VPC | `network-lab-vpc` |

Route will be added in Step 5 after NAT Gateway is created.

Associate subnets: `private-subnet-az1`, `private-subnet-az2`

#### Isolated Route Table
| Field | Value |
|---|---|
| Name | `isolated-rt` |
| VPC | `network-lab-vpc` |

No internet route — local route only.

Associate subnets: `isolated-subnet-az1`, `isolated-subnet-az2`

**Why:** Route tables are the GPS for your network. `0.0.0.0/0` is the default catch-all route. Every route table automatically gets a `10.0.0.0/16 → local` route, which allows all subnets to communicate within the VPC — that's how App servers talk to DB servers.

---

### Step 5: NAT Gateway

**Go to:** VPC Console → NAT Gateways → Create NAT Gateway

| Field | Value |
|---|---|
| Name | `network-lab-nat` |
| Subnet | `public-subnet-az1` |
| Connectivity type | Public |
| Elastic IP | Click "Allocate Elastic IP" |

Wait for status to show **Available** (~2 minutes).

Then go back to `private-rt` → Routes → Edit Routes → Add Route:

`0.0.0.0/0 → network-lab-nat`

**Why:** NAT Gateway must live in a public subnet because it needs IGW access to reach the internet on behalf of private servers. The Elastic IP gives it a fixed public address so return traffic knows where to go. Private servers stay invisible to the internet — only outbound traffic is allowed.

> **Production note:** For full AZ redundancy, deploy a second NAT Gateway in `public-subnet-az2` and create separate private route tables per AZ, each pointing to their local NAT.

> **Cost warning:** NAT Gateway charges ~$0.045/hour + data transfer. Delete it when done with the lab.

---

### Step 6: Security Groups

**Go to:** EC2 Console → Security Groups → Create Security Group

#### Bastion SG (`bastion-sg`)
| Type | Port | Source |
|---|---|---|
| SSH | 22 | My IP only |

#### App SG (`app-sg`)
| Type | Port | Source |
|---|---|---|
| SSH | 22 | `bastion-sg` |

#### DB SG (`db-sg`)
| Type | Port | Source |
|---|---|---|
| MySQL/Aurora | 3306 | `app-sg` |

**Why:** This is the chained security model — each tier only trusts the layer directly above it. By referencing security groups instead of IPs, the rules scale automatically as you add more servers. This is called Security Group Referencing and is a production best practice.

Security groups are **stateful** — return traffic is automatically allowed without needing explicit outbound rules. This differs from NACLs which are **stateless** and require explicit rules in both directions.

---

### Step 7: EC2 Instances

**Go to:** EC2 Console → Instances → Launch Instance

#### Bastion Host
| Field | Value |
|---|---|
| Name | `bastion-host` |
| AMI | Amazon Linux 2023 |
| Instance type | t2.micro |
| Subnet | `public-subnet-az1` |
| Auto-assign public IP | Enable |
| Security Group | `bastion-sg` |

#### App Server
| Field | Value |
|---|---|
| Name | `app-server` |
| AMI | Amazon Linux 2023 |
| Instance type | t2.micro |
| Subnet | `private-subnet-az1` |
| Auto-assign public IP | Disable |
| Security Group | `app-sg` |

#### DB Server
| Field | Value |
|---|---|
| Name | `db-server` |
| AMI | Amazon Linux 2023 |
| Instance type | t2.micro |
| Subnet | `isolated-subnet-az1` |
| Auto-assign public IP | Disable |
| Security Group | `db-sg` |

**Why:** The Bastion is the only server with a public IP — it acts as the single entry point into the private network. This minimizes attack surface. App and DB servers have no public IPs and are unreachable directly from the internet.

> **Production improvement:** In real environments, replace the Bastion with **AWS Systems Manager Session Manager** — no port 22 open anywhere, no key pair management, full audit logging.

---

### Step 8: VPC Endpoint (S3 Gateway)

**Go to:** VPC Console → Endpoints → Create Endpoint

| Field | Value |
|---|---|
| Name | `s3-endpoint` |
| Service category | AWS Services |
| Service | com.amazonaws.us-east-1.s3 (Type: Gateway) |
| VPC | `network-lab-vpc` |
| Route Tables | `private-rt`, `isolated-rt` |

**Why:** Without the endpoint, S3 traffic from private subnets flows through NAT Gateway → IGW → public internet → S3. With a Gateway Endpoint, traffic stays entirely on AWS's private backbone — faster, free, and more secure. AWS automatically adds an S3 prefix list route to your selected route tables.

| | Via NAT | Via VPC Endpoint |
|---|---|---|
| Cost | Pays per GB | Free |
| Path | Public internet | AWS private network |
| Security | Traffic exposed | Never leaves AWS |

---

### Step 9: IAM Role for S3 Access

**Go to:** IAM Console → Roles → Create Role

| Field | Value |
|---|---|
| Trusted entity | AWS Service → EC2 |
| Policy | AmazonS3ReadOnlyAccess |
| Role name | `app-server-s3-role` |

Attach to App Server: EC2 → select `app-server` → Actions → Security → Modify IAM Role → select `app-server-s3-role`

**Why:** EC2 instances need IAM roles to call AWS APIs. Read-only access follows the least privilege principle — the app can read from S3 but cannot create or delete buckets.

---

## Testing

### Test Setup: Copying SSH Key Through the Chain

SSH access follows the jump-box pattern. The key must be present on each server to hop to the next.

**Step 1 — Copy key from local machine to Bastion (run from Windows PowerShell):**
```powershell
scp -i "c:\users\basit\Downloads\project1-key.pem" "c:\users\basit\Downloads\project1-key.pem" ec2-user@<bastion-public-ip>:/home/ec2-user/.ssh/
```

**Step 2 — Fix key permissions on Bastion:**
```bash
chmod 400 ~/.ssh/project1-key.pem
```

**Step 3 — Copy key from Bastion to App Server (run from Bastion):**
```bash
scp -i ~/.ssh/project1-key.pem ~/.ssh/project1-key.pem ec2-user@<app-private-ip>:/home/ec2-user/.ssh/
```

**Step 4 — Fix key permissions on App Server:**
```bash
chmod 400 ~/.ssh/project1-key.pem
```

---

### Test 1: SSH Into Bastion Host ✅

```powershell
# Run from local Windows PowerShell
ssh -i "c:\users\basit\Downloads\project1-key.pem" ec2-user@<bastion-public-ip>
```

**Expected:** Amazon Linux 2023 banner and `[ec2-user@ip-10-0-1-x ~]$` prompt

**Proves:** Bastion is reachable from the internet via public IP. Security group correctly allows SSH from your IP only.

---

### Test 2: Bastion → App Server ✅

```bash
# Run from inside Bastion session
ssh -i ~/.ssh/project1-key.pem ec2-user@10.0.2.111
```

**Expected:** `[ec2-user@ip-10-0-2-111 ~]$` — notice private IP only, no public IP

**Proves:** Jump-box pattern works. App server is only reachable through the Bastion. `app-sg` correctly allows SSH from `bastion-sg` only.

---

### Test 3: Private Internet Access via NAT Gateway ✅

```bash
# Run from inside App Server session
curl google.com
```

**Expected:** HTML response (`301 Moved`) from Google

**Proves:** Private subnet has outbound internet via NAT Gateway without any public IP on the instance.

---

### Test 4: S3 Access via VPC Endpoint ✅

```bash
# Run from inside App Server session
aws s3 ls
```

**Expected:** List of S3 buckets in your account

**Proves:** Private EC2 can access S3 without going through NAT or the public internet. Traffic stays on AWS private backbone via the Gateway Endpoint.

---

### Test 5: App Server → DB Server ✅

```bash
# Run from inside App Server session
ssh -i ~/.ssh/project1-key.pem ec2-user@10.0.3.37
```

**Expected:** `[ec2-user@ip-10-0-3-37 ~]$`

**Proves:** App tier can reach DB tier via local VPC routing. `db-sg` correctly allows SSH from `app-sg`.

---

### Test 6: DB Server Internet Isolation ✅

```bash
# Run from inside DB Server session
curl google.com
# Hit Ctrl+C after a few seconds
```

**Expected:** Command hangs with no response — requires Ctrl+C to cancel

**Proves:** Isolated subnet has zero internet access. No NAT route exists in `isolated-rt`. Even if an attacker compromises the DB server, they cannot reach the internet or download malicious tools.

---

## Test Results Summary

| Test | Command | Result |
|---|---|---|
| ✅ Bastion SSH | `ssh ec2-user@<bastion-public-ip>` | Connected |
| ✅ Bastion → App Server | `ssh ec2-user@10.0.2.111` | Connected |
| ✅ NAT Gateway | `curl google.com` | 301 response |
| ✅ S3 VPC Endpoint | `aws s3 ls` | Buckets listed |
| ✅ App → DB | `ssh ec2-user@10.0.3.37` | Connected |
| ✅ DB Isolation | `curl google.com` | Timeout (expected) |

---

## Key Concepts

### Why Is This Called "Defense in Depth"?
Each layer adds an additional security barrier:
1. Only Bastion has a public IP — single entry point
2. App servers only accept SSH from Bastion SG
3. DB servers only accept MySQL from App SG
4. DB servers have zero internet access
5. S3 traffic never hits the public internet

An attacker would need to breach all layers sequentially — significantly harder than a flat network.

### Why NAT Gateway Lives in the Public Subnet
NAT Gateway needs IGW access to reach the internet on behalf of private servers. It acts as a middleman — private servers make outbound requests through NAT, which forwards them using its own public Elastic IP. The internet only ever sees the NAT's IP, never the private server.

### Why VPC Endpoint Is Better Than NAT for S3
For S3 and DynamoDB, a Gateway Endpoint is completely free and routes traffic privately through AWS's backbone network. Companies doing heavy S3 usage (ML training, backups, log archiving) can save thousands of dollars per month by eliminating NAT data charges for S3 traffic.

### Security Group Referencing vs IP-Based Rules
By referencing `app-sg` as the source for DB access instead of IP addresses, the rule automatically applies to any instance that joins `app-sg` — no manual updates needed when scaling. This is a production best practice that scales to hundreds of servers with zero rule changes.

---

## Cleanup (Important — Avoid Ongoing Charges)

Delete resources in this order to avoid dependency errors:

1. EC2 Instances (Bastion, App, DB)
2. NAT Gateway (`network-lab-nat`)
3. Release Elastic IP
4. VPC Endpoints
5. Security Groups
6. Route Tables
7. Subnets
8. Internet Gateway (detach first, then delete)
9. VPC
10. S3 bucket (`network-lab-test-syed-123`)
11. IAM Role (`app-server-s3-role`)

> **Note:** NAT Gateway is the only resource that incurs hourly charges (~$0.045/hr). Delete it first if you need to pause the lab.

---

## Future Improvements

- Add an **Application Load Balancer** spanning both AZs in front of the App Server
- Deploy **App and DB servers in AZ2** for true high availability
- Add a **second NAT Gateway** in AZ2 with separate private route tables per AZ
- Replace Bastion with **AWS Systems Manager Session Manager** — no SSH keys, no port 22, full audit logs
- Add **VPC Flow Logs** to CloudWatch for network traffic monitoring
- Implement **S3 bucket policy** restricting access to VPC Endpoint only
- Convert to **Terraform** for infrastructure as code

---


- **Cloud:** AWS (us-east-1)
- **Compute:** EC2 Amazon Linux 2023 (t2.micro)
- **Networking:** VPC, Subnets, IGW, NAT Gateway, Route Tables, Security Groups
- **Storage:** S3 with Gateway VPC Endpoint
- **Security:** IAM Roles, Security Group Chaining, Bastion Jump Host
