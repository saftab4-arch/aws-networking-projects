# Project 1 — VPC to VPC via Transit Gateway

## Overview
Simulates on-premises to AWS cloud connectivity using Transit Gateway as the central hub.

## Architecture
VPC-A (AWS Cloud) ←→ TGW ←→ VPC-B (On-Prem Sim)

## CIDR Design
- VPC-A: 10.0.0.0/16 (simulates AWS cloud)
- VPC-B: 192.168.0.0/16 (simulates on-prem/home laptop)
- TGW: hub connecting both

## What I Learned

## Screenshots
![Architecture](screenshots/architecture.png)

## Cleanup
