# Project 1 — VPC to VPC Connectivity via Transit Gateway

## Overview
This project demonstrates how to connect two isolated VPCs using AWS Transit 
Gateway, simulating a real-world hybrid cloud setup where an on-premises network 
communicates with AWS cloud infrastructure.

VPC-A represents an AWS cloud environment and VPC-B simulates an on-premises 
or home office network. Both VPCs are connected through a central Transit Gateway, 
allowing EC2 instances in each VPC to communicate with each other bidirectionally.

## Architecture
 ![Architecture](screenshots/architecture.svg)
 
****
VPC-A-Cloud (10.0.0.0/16)          VPC-B-OnPrem (192.168.0.0/16)
┌─────────────────────┐            ┌─────────────────────┐
│  subnet-A-public    │            │  subnet-B-onprem    │
│  10.0.1.0/24        │            │  192.168.1.0/24     │
│                     │            │                     │
│  EC2-A-Cloud        │            │  EC2-B-OnPrem       │
│  10.0.1.x           │            │  192.168.1.x        │
└────────┬────────────┘            └──────────┬──────────┘
│                                    │
│         attach-VPC-A               │
└──────────────┐                     │
▼                     │
┌─────────────┐              │
│     TGW     │◄─────────────┘
│ TGW-Project1│  attach-VPC-B
└─────────────┘




## Resources Created
- 2 VPCs (VPC-A-Cloud, VPC-B-OnPrem)
- 2 Subnets (one per VPC)
- 2 Internet Gateways (one per VPC)
- 1 Transit Gateway (TGW-Project1)
- 2 TGW Attachments (attach-VPC-A, attach-VPC-B)
- 2 Security Groups (ec2-A-sg, ec2-B-sg)
- 2 EC2 Instances (Amazon Linux 2023, t2.micro)

## CIDR Design
| Resource | CIDR | Purpose |
|----------|------|---------|
| VPC-A-Cloud | 10.0.0.0/16 | Simulates AWS cloud network |
| VPC-B-OnPrem | 192.168.0.0/16 | Simulates on-premises network |
| subnet-A-public | 10.0.1.0/24 | Public subnet in VPC-A |
| subnet-B-onprem | 192.168.1.0/24 | Public subnet in VPC-B |

## Route Table Configuration
**VPC-A Route Table:**
| Destination | Target |
|-------------|--------|
| 10.0.0.0/16 | Local |
| 192.168.0.0/16 | TGW-Project1 |
| 0.0.0.0/0 | IGW-VPC-A |

**VPC-B Route Table:**
| Destination | Target |
|-------------|--------|
| 192.168.0.0/16 | Local |
| 10.0.0.0/16 | TGW-Project1 |
| 0.0.0.0/0 | IGW-VPC-B |

## Security Groups
**ec2-A-sg (attached to EC2-A):**
| Type | Protocol | Source |
|------|----------|--------|
| All ICMP | ICMP | 192.168.0.0/16 |
| SSH | TCP 22 | My IP |

**ec2-B-sg (attached to EC2-B):**
| Type | Protocol | Source |
|------|----------|--------|
| All ICMP | ICMP | 10.0.0.0/16 |
| SSH | TCP 22 | My IP |

## What I Learned
- Transit Gateway acts as a central hub router between VPCs — similar to a 
  core switch in a data center
- VPCs are completely isolated by default — even in the same AWS account and 
  region they cannot communicate without explicit routing
- Route tables on both sides must be configured — traffic is two-way so both 
  VPCs need to know how to reach each other's CIDR range
- Internet Gateways are not created automatically when using "VPC only" mode — 
  they must be created and attached manually, with a default route added
- Security groups must explicitly allow ICMP for ping to work — AWS blocks all 
  inbound traffic by default
- TGW attachments are how you "plug in" a VPC to the Transit Gateway — one 
  attachment per VPC, selecting the subnet TGW will use in that VPC

## Proof of Connectivity
- EC2-A successfully pinged EC2-B across the Transit Gateway (10.0.1.x → 192.168.1.x)
- EC2-B successfully pinged EC2-A across the Transit Gateway (192.168.1.x → 10.0.1.x)

## Screenshots
| # | Screenshot | Description |
|---|-----------|-------------|
| 01| VPC-A created with CIDR 10.0.0.0/16 |
| 02  | Both VPCs visible in the list |
| 03  | Transit Gateway state = Available |
| 04 | VPC-A attachment Available |
| 05 | Both attachments Available |
| 06| VPC-B route table with TGW route |
| 07 | EC2-A running with private IP |
| 08 | Both EC2 instances running |
| 09  | Ping from EC2-A to EC2-B |
| 10 | Ping from EC2-B to EC2-A |

## Cleanup
Resources deleted after project completion to avoid unnecessary charges:
- EC2 instances (EC2-A-Cloud, EC2-B-OnPrem)
- Transit Gateway attachments (attach-VPC-A, attach-VPC-B)
- Transit Gateway (TGW-Project1)
- Internet Gateways (IGW-VPC-A, IGW-VPC-B)
- Subnets, Security Groups, VPCs
