---
title: "Hybrid Cloud Lab Part 5: Hybrid Connectivity & Secure Access"
date: 2026-03-06 14:00:00 +0000
categories: [IT Infrastructure, Cloud Computing]
tags: [aws, vpn, ipsec, guacamole, security]
---


# Hybrid Cloud Lab — Part 5: IPsec IKEv2 Site-to-Site VPN

Connecting an on-premise homelab to AWS via IPsec seems simple until you realize both ends have fundamentally different constraints. In this post, I detail the asymmetric configuration required to establish a stable tunnel between pfSense (behind a residential NAT) and a strongSwan responder on AWS.

---

## 1. Asymmetric Topology Requirements

| Constraint | pfSense (On-Premise) | EC2 strongSwan (AWS) |
|---|---|---|
| Public IP | Dynamic (Residential ISP) | Static (Elastic IP) |
| NAT | Behind Residential double NAT | Direct Public IP |
| IKEv2 Role | **Initiator** (MUST initiate) | **Responder** (waits for connection) |
| Peer Address | `51.20.237.223` (Static) | `%any` (Accepts dynamic IP) |
| DPD Action | `restart` (Auto-reconnect) | `clear` (Clears dead session) |

---

## 2. AWS Side: strongSwan Configuration

### `/etc/ipsec.conf` — Key Directives

The AWS instance acts as the passive responder. The `modp2048` group and `aes256-sha256` suites are enforced strictly.

```ini
# /etc/ipsec.conf excerpt
conn lab-tunnel
    auto=add             # Load config but do NOT initiate
    keyexchange=ikev2
    left=10.0.1.99      # EC2 Private IP
    leftid=51.20.237.223 # Elastic IP (Identity for pfSense)
    leftsubnet=10.0.0.0/16
    right=%any           # Accept any initiator IP (Dynamic)
    rightsubnet=192.168.0.0/16
    ike=aes256-sha256-modp2048! # Strict mode
    esp=aes256-sha256!
    forceencaps=yes      # Force NAT-Traversal (UDP 4500)
```

---

## 3. pfSense Side: IKEv2 Phase 1 & 2

### Phase 1 (IKEv2)
- **Encryption**: AES-256 / SHA-256 / DH Group 14 (2048-bit).
- **NAT Traversal**: Forced. This is critical as the residential ISP often uses CGNAT or restrictive state tables.
- **Keep-alive ping**: Configured to ping `10.0.1.99` every 10 seconds. This maintains the NAT mapping in the residential router.

### Phase 2 (ESP)
- **Remote Network**: `10.0.0.0/16` (The entire AWS VPC).
- **PFS**: **OFF**. One of the most common causes of Phase 2 failure is a PFS mismatch. Ensure both sides match exactly.

---

## 4. Troubleshooting & Verification

- **Status Check**: `sudo ipsec statusall` on EC2 should show `ESTABLISHED`.
- **Packet Flow**: `tcpdump -i vtnet0 'udp port 500 or udp port 4500'` on pfSense validates that IKE packets are leaving the premise.
- **Reachability**: Once up, you should be able to ping the private IP of any VM in the lab from the EC2 instance (e.g., `ping 192.168.10.10`).

---

## Next Part

**[Part 6 → Resilience & Automation](/posts/hybrid-cloud-lab-part-6-resilience-and-automation/)** — Implementing self-healing scripts and a 3-tier backup strategy.
