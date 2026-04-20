# Project 2 — Bidirectional Traffic via TGW

## Overview
EC2 in VPC-A communicates with EC2 in VPC-B in both directions through Transit Gateway.
Simulates EC2 reaching on-prem API and on-prem reaching AWS EC2.

## Architecture
EC2-A → TGW → EC2-B (AWS to On-Prem direction)
EC2-B → TGW → EC2-A (On-Prem to AWS direction)

## What I Learned

## Screenshots
![Architecture](screenshots/architecture.png)

## Cleanup
