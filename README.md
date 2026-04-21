# AWS Networking Projects

Hands-on AWS networking projects built from scratch, simulating real-world
hybrid cloud and multi-VPC architectures. Each project is fully documented
with architecture diagrams, screenshots, and detailed explanations of what
was built and learned.

> One project per week. Building in public.

---

## Projects

| # | Project | Key Services | Status |
|---|---------|-------------|--------|
| 01 | [VPC to VPC via Transit Gateway](#project-1) | VPC, TGW, EC2, IGW | ✅ Complete |
| 02 | [Bidirectional App Traffic via TGW](#project-2) | VPC, TGW, EC2, Node.js | ✅ Complete |

---

## Project 1 — VPC to VPC via Transit Gateway

**Folder:** `project-1-tgw-vpc-connectivity/`

Two completely isolated VPCs connected through a central Transit Gateway,
simulating a hybrid cloud setup where on-premises infrastructure communicates
with AWS cloud. Bidirectional ICMP (ping) traffic confirmed working through TGW.

### Architecture

VPC-A-Cloud (10.0.0.0/16)          VPC-B-OnPrem (192.168.0.0/16)
┌──────────────────┐                ┌──────────────────┐
│  EC2-A-Cloud     │                │  EC2-B-OnPrem    │
│  10.0.1.x        │                │  192.168.1.x     │
└────────┬─────────┘                └─────────┬────────┘
│                                    │
└──────────── TGW-Project1 ──────────┘
(central hub)

### What was built
- 2 VPCs with non-overlapping CIDRs
- 2 Subnets + 2 Internet Gateways
- 1 Transit Gateway with 2 VPC attachments
- Route tables configured on both sides
- Security groups allowing ICMP between VPCs
- Bidirectional ping confirmed working

### Key learning
Transit Gateway acts like a core switch — VPCs plug into it via attachments
and route tables on both sides direct traffic through the hub. VPCs are
completely isolated by default even in the same account and region.

---

## Project 2 — Bidirectional App Traffic via Transit Gateway

**Folder:** `project-2-tgw-bidirectional/`

Same TGW architecture as Project 1 but with real Node.js HTTP servers running
on both EC2 instances. Live application traffic flows in both directions through
the Transit Gateway — simulating microservice-to-microservice communication
across network boundaries.

### Architecture
VPC-A-Cloud (10.0.0.0/16)          VPC-B-OnPrem (192.168.0.0/16)
┌──────────────────────┐            ┌──────────────────────┐
│  EC2-A-Cloud         │            │  EC2-B-OnPrem        │
│  Node.js app :3000   │            │  Node.js app :3000   │
└──────────┬───────────┘            └───────────┬──────────┘
│                                    │
└──────────── TGW-Project2 ──────────┘
EC2-A curl → TGW → EC2-B:3000  ✅
EC2-B curl → TGW → EC2-A:3000  ✅
### What was built
- Same VPC/TGW/IGW setup as Project 1
- Node.js 18 HTTP server deployed on both EC2 instances
- Security groups opening port 3000 between VPCs
- Bidirectional HTTP curl confirmed working with JSON responses

### Key learning
Transit Gateway is protocol-agnostic — it routes HTTP, HTTPS, database
traffic, gRPC identically to ICMP. Security groups must explicitly open
the specific port from the remote VPC's CIDR range.

---

## Skills Demonstrated

- AWS VPC design and CIDR planning
- Transit Gateway architecture and attachment configuration
- Route table design for multi-VPC environments
- Internet Gateway setup and default routing
- Security group rules for cross-VPC traffic
- EC2 provisioning and EC2 Instance Connect
- Node.js application deployment on Amazon Linux 2023
- Hybrid cloud network simulation

## AWS Services Used

`VPC` `Transit Gateway` `EC2` `Internet Gateway` `Route Tables` `Security Groups`

## Real-World Relevance

These projects simulate architectures used in production environments where:
- Multiple teams own separate VPCs and need secure service-to-service communication
- On-premises data centers connect to AWS over Direct Connect or Site-to-Site VPN
- Microservices in different VPCs call each other's APIs without public internet exposure

The Transit Gateway routing logic in these projects is identical to what runs
in enterprise hybrid cloud environments — just with VPCs instead of real
physical networks on one side.

---

## Author

**Syed Basit Aftab**
[GitHub](https://github.com/saftab4-arch)

---

## Repo Structure


aws-networking-projects/
├── README.md
├── project-1-tgw-vpc-connectivity/
│   ├── README.md
│   └── screenshots/
│       ├── architecture.svg
│       ├── 01-vpc-a-created.png
│       └── ...
└── project-2-tgw-bidirectional/
├── README.md
└── screenshots/
├── architecture.svg
├── 01-both-vpcs-created.png
└── ...

> All resources are deleted after each project to avoid unnecessary AWS charges.
> Screenshots and documentation serve as the permanent record of each build.
