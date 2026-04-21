# Project 2 — Bidirectional Application Traffic via Transit Gateway

## Overview
This project extends Project 1 by running real Node.js web applications on both
EC2 instances and testing bidirectional HTTP traffic through the Transit Gateway.

VPC-A simulates an AWS cloud environment and VPC-B simulates an on-premises
network. Both EC2 instances run a Node.js HTTP server on port 3000. Traffic
flows in both directions through the Transit Gateway, simulating real
microservice-to-microservice communication across network boundaries.

## ArchitectureVPC-A-Cloud (10.0.0.0/16)              VPC-B-OnPrem (192.168.0.0/16)
┌──────────────────────────┐           ┌──────────────────────────┐
│  subnet-A-public         │           │  subnet-B-onprem         │
│  10.0.1.0/24             │           │  192.168.1.0/24          │
│                          │           │                          │
│  EC2-A-Cloud             │           │  EC2-B-OnPrem            │
│  10.0.1.x                │           │  192.168.1.x             │
│  Node.js app :3000       │           │  Node.js app :3000       │
└───────────┬──────────────┘           └──────────┬───────────────┘
│                                     │
│──────────────────────────────────── │
TGW-Project2
(central hub)EC2-A curl → TGW → EC2-B app (On-Prem to Cloud direction)
EC2-B curl → TGW → EC2-A app (Cloud to On-Prem direction)


## Resources Created
- 2 VPCs (VPC-A-Cloud, VPC-B-OnPrem)
- 2 Subnets (one per VPC)
- 2 Internet Gateways (one per VPC)
- 1 Transit Gateway (TGW-Project2)
- 2 TGW Attachments (attach-VPC-A, attach-VPC-B)
- 2 Security Groups (ec2-A-sg, ec2-B-sg)
- 2 EC2 Instances (Amazon Linux 2023, t2.micro)
- 2 Node.js HTTP servers (port 3000)

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
| 192.168.0.0/16 | TGW-Project2 |
| 0.0.0.0/0 | IGW-VPC-A |

**VPC-B Route Table:**
| Destination | Target |
|-------------|--------|
| 192.168.0.0/16 | Local |
| 10.0.0.0/16 | TGW-Project2 |
| 0.0.0.0/0 | IGW-VPC-B |

## Security Groups
**ec2-A-sg (attached to EC2-A):**
| Type | Protocol | Port | Source |
|------|----------|------|--------|
| All ICMP | ICMP | - | 192.168.0.0/16 |
| Custom TCP | TCP | 3000 | 192.168.0.0/16 |
| SSH | TCP | 22 | My IP |

**ec2-B-sg (attached to EC2-B):**
| Type | Protocol | Port | Source |
|------|----------|------|--------|
| All ICMP | ICMP | - | 10.0.0.0/16 |
| Custom TCP | TCP | 3000 | 10.0.0.0/16 |
| SSH | TCP | 22 | My IP |

## Node.js App
Both EC2 instances run a simple HTTP server on port 3000 built with Node.js 18.
The app responds with a JSON payload containing server identity, IP, and timestamp.

```javascript
const http = require('http');

const server = http.createServer((req, res) => {
  res.writeHead(200, {'Content-Type': 'application/json'});
  res.end(JSON.stringify({
    message: 'Hello from VPC-B OnPrem server!',
    server: 'EC2-B-OnPrem',
    ip: '192.168.1.x',
    timestamp: new Date().toISOString()
  }));
});

server.listen(3000, '0.0.0.0', () => {
  console.log('Server running on port 3000');
});
```

## What I Learned
- Transit Gateway routes real application traffic, not just ICMP ping — any
  TCP/UDP traffic works as long as routing and security groups allow it
- Security groups must open specific ports — port 3000 had to be explicitly
  allowed from the other VPC's CIDR range
- Node.js can be installed on Amazon Linux 2023 via NodeSource repository
- This architecture mirrors real microservice communication across VPCs in
  enterprise environments — service A in one VPC calling service B in another
- The TGW is protocol-agnostic — it routes packets regardless of whether
  they are ping, HTTP, HTTPS, or any other protocol

## Proof of Connectivity
- EC2-A curl to EC2-B:3000 returned JSON response successfully
- EC2-B curl to EC2-A:3000 returned JSON response successfully
- Both directions confirmed working through Transit Gateway

## Screenshots
| # | Screenshot | Description |
|---|-----------|-------------|
| 01 | ![Both VPCs](screenshots/01-both-vpcs-created.png) | Both VPCs created |
| 02 | ![TGW Available](screenshots/02-tgw-available.png) | Transit Gateway available |
| 03 | ![Attachments](screenshots/03-both-attachments-available.png) | Both attachments available |
| 04 | ![Route Tables](screenshots/04-route-tables-configured.png) | Route tables configured |
| 05 | ![EC2s Running](screenshots/05-both-ec2-running.png) | Both EC2s running |
| 06 | ![Node EC2-B](screenshots/06-nodejs-running-ec2b.png) | Node.js running on EC2-B |
| 07 | ![Curl A to B](screenshots/07-curl-ec2a-to-ec2b.png) | EC2-A curl to EC2-B app |
| 08 | ![Node EC2-A](screenshots/08-nodejs-running-ec2a.png) | Node.js running on EC2-A |
| 09 | ![Curl B to A](screenshots/09-curl-ec2b-to-ec2a.png) | EC2-B curl to EC2-A app |
| 10 | ![Both Working](screenshots/10-both-directions-working.png) | Both directions confirmed |

## Cleanup
Resources deleted after project completion:
- EC2 instances (EC2-A-Cloud, EC2-B-OnPrem)
- Transit Gateway attachments (attach-VPC-A, attach-VPC-B)
- Transit Gateway (TGW-Project2)
- Internet Gateways (IGW-VPC-A, IGW-VPC-B)
- Subnets, Security Groups, VPCs

- 
