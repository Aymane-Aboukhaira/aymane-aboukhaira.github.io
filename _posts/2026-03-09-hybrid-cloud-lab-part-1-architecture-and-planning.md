---
title: "Hybrid Cloud Lab Part 1: The Vision & Architecture"
date: 2026-03-09 10:00:00 +0000
categories: [IT Infrastructure, Cloud Computing]
tags: [architecture, planning, virtualization, hybrid-cloud]
---

# Hybrid Cloud Lab — Part 1: Global Architecture

Welcome to the first part of my Hybrid Cloud Infrastructure Lab series. As a Networks & Systems engineering student at ISMONTIC (Tangier, Morocco), I’ve spent the last four months designing and deploying a complete hybrid cloud infrastructure on a legacy Dell Latitude E5470 with 8GB of RAM, coupled with an AWS Free Tier account.

This wasn't just a step-by-step tutorial walkthrough; it was about building a real system that broke in interesting ways and required deep infrastructure-level debugging.

---

## 1. Component Overview

| Component | Role | Location | IP | Status |
|---|---|---|---|---|
| Proxmox VE 8 | KVM Hypervisor | Dell E5470 | 192.168.11.50 | Active |
| pfSense CE | Firewall, NAT, IPsec, VLAN Routing | Proxmox VM | WAN: Dynamic | Active |
| DC01 — WinSrv 2022 | Active Directory, DNS, DHCP | VLAN 10 | 192.168.10.10 | Active |
| centos-vm2 | Prometheus, Grafana, Backup Master | VLAN 20 | 192.168.20.30 | Active |
| centos-vm1 | node-exporter, Future DMZ App | VLAN 30 | 192.168.30.20 | Active |
| omv-nas | OpenMediaVault — SMB + NFS | VLAN 40 | 192.168.40.40 | Active |
| EC2 lab-secure-edge | Guacamole, NPM, PostgreSQL, Fail2ban | AWS eu-north-1 | 51.20.237.223 | Active |
| IPsec IKEv2 Tunnel | pfSense ↔ EC2, AES-256/SHA-256 | Network Overlay | — | Active |

---

## 2. Network Addressing Plan

### 2.1 On-Premise VLANs

| VLAN | Name | Subnet | Gateway | DNS | Primary Hosts |
|---|---|---|---|---|---|
| 10 | MGMT | 192.168.10.0/24 | 192.168.10.1 | 192.168.10.10 (DC01) | DC01 (.10) |
| 20 | SERVICES | 192.168.20.0/24 | 192.168.20.1 | 192.168.10.10 | centos-vm2 (.30) |
| 30 | DMZ | 192.168.30.0/24 | 192.168.30.1 | 8.8.8.8 | centos-vm1 (.20) |
| 40 | STORAGE | 192.168.40.0/24 | 192.168.40.1 | 192.168.10.10 | omv-nas (.40) |

> **Note**: DMZ DNS points to 8.8.8.8 intentionally for DNS isolation; the DMZ cannot resolve `dc01.lab.local`.

### 2.2 AWS VPC

| Resource | Value | Notes |
|---|---|---|
| VPC CIDR | 10.0.0.0/16 | lab-vpc |
| Public Subnet | 10.0.1.0/24 | eu-north-1a |
| EC2 Private IP | 10.0.1.99 | Fixed in subnet |
| Elastic IP | 51.20.237.223 | Static — required for IPsec |
| Internet Gateway | lab-igw | 0.0.0.0/0 |

---

## 3. Key Architectural Decisions

### 3.1 Proxmox VE 8 — Hypervisor Choice
Proxmox VE 8 is based on KVM—the same hypervisor used by AWS EC2 under the hood. The skills are directly transferable. It's open-source, license-free, and features a mature REST API and a comprehensive web interface.

> **Hardware Constraint**: Dell Latitude E5470 with 8GB RAM. Strict allocation: never more than 6GB allocated simultaneously. The Intel I219-LM NIC requires a permanent auto-negotiation fix (`ethtool`, forced 100 Mbps full-duplex).

### 3.2 "Isolated DMZ" Architecture
Segmentation into 4 isolated VLANs eliminates the risk of lateral movement. A compromised web application in VLAN 30 cannot reach DC01, the NAS, or the monitoring stack—enforced by pfSense deny-by-default rules.

### 3.3 EC2 as the Single Entry Point
No on-premise services are exposed directly to the Internet. Access is exclusively through Guacamole on EC2 (HTTPS + TOTP + AD LDAP), with RDP/SSH traffic tunneled via IPsec AES-256. The attack surface is reduced to a single controlled point.

### 3.4 IAM Least Privilege
Two dedicated IAM users:
- `lab-sg-updater`: for modifying a single specific Security Group.
- `lab-s3-backup-bot`: `PutObject` + `ListBucket` only—never `DeleteObject`.

---

## 4. Complete Technology Stack

| Layer | Technology | Usage |
|---|---|---|
| Hypervisor | Proxmox VE 8 | KVM VM Orchestration |
| Firewall / VPN | pfSense CE | Firewall, NAT, IPsec, VLAN |
| VPN | IPsec IKEv2 / strongSwan (AES-256) | Encrypted site-to-site tunnel |
| Directory | Active Directory (WinSrv 2022) | LDAP Auth, DNS |
| Remote Access | Apache Guacamole | Clientless RDP/SSH/SFTP |
| Reverse Proxy | Nginx Proxy Manager | SSL termination, Let's Encrypt |
| Database | PostgreSQL 15 | Guacamole storage + TOTP secrets |
| Monitoring | Prometheus + Grafana | Multi-OS Metrics |
| Security | Fail2ban | Docker brute-force protection |
| Storage | OpenMediaVault 7.x | SMB + NFS NAS |
| Cloud | AWS EC2 / VPC / S3 / IAM | Cloud infrastructure |

---

## 5. Why Build This?

Most courses teach what Active Directory is. They don't teach what happens when AD authentication traverses an IPsec tunnel and fails silently at 3 AM. This gap is what this project was designed to bridge.

---

## Next Part

**[Part 2 → Hypervisor & Virtualization Foundation](/posts/hybrid-cloud-lab-part-2-hypervisor-and-foundation/)** — Dive into the core setup and initial challenges.
