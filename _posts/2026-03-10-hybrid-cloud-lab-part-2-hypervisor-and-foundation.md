---
title: "Hybrid Cloud Lab Part 2: Hypervisor & Virtualization Foundation"
date: 2026-03-10 10:00:00 +0000
categories: [IT Infrastructure, Virtualization]
tags: [proxmox, pfsense, networking, hardware-troubleshooting]
---

# Hybrid Cloud Lab — Part 2: Post-Mortems & Infrastructure Lessons

The most valuable part of any infrastructure project isn't the configuration files—it's the incidents. In this post, I’m sharing four critical issues I faced during the foundation setup, the Root Cause Analysis (RCA), and the permanent fixes implemented.

| # | Incident | Duration | Impacted Services | Root Cause |
|---|---|---|---|---|
| 1 | Fail2ban NAT Lockout | ~20 min | Guacamole, IPsec, SSH | Admin IP shared with pfSense |
| 2 | Silent IPsec Blackhole | ~45 min | VPN Tunnel, Guacamole | ISP IP change mismatch in SG |
| 3 | Silent pfSense DDNS Failure | ~30 min | VPN Tunnel (Repeat) | Native DDNS client failed without alert |
| 4 | Proxmox VLAN Crash | ~90 min | VLAN 30 (DMZ) | Proxmox 9.1.1 bridge validation |

---

## 1. Incident: Fail2ban NAT Lockout

### Chronology
- **T+0**: Generated 5 intentional 403 errors from an admin workstation to test Fail2ban.
- **T+30s**: Ban triggered: Public IP `102.100.226.122` was dropped in the `DOCKER-USER` chain.
- **T+31s**: pfSense sent IKEv2 keep-alive packets from the *same* IP.
- **T+45s**: DPD timeout—pfSense declared the tunnel dead; SSH timed out.

### Root Cause
The admin workstation and pfSense shared the same public IP via residential NAT. When Fail2ban banned the IP for "brute force" attempts (the test), it inadvertently killed the VPN tunnel and SSH access.

### Resolution
- **Emergency Fix**: Accessed AWS via CloudShell and ran `fail2ban-client set npm-docker unbanip 102.100.226.122`.
- **Permanent Fix**: Automated the `ignoreip` update in `jail.local` via the `update-sg-ip.sh` script to ensure the current admin IP is never banned.

---

## 2. Incident: Silent IPsec Blackhole

### Symptoms
pfSense showed "Connecting..." in an infinite loop. No IKE packets were arriving at the AWS strongSwan instance, despite the tunnel being "up" previously.

### Root Cause
A change in the home ISP's public IP caused a mismatch in the AWS Security Group. AWS SGs drop unauthorized packets silently—strongSwan never saw the connection attempts, and pfSense didn't know why it was being ignored.

### Resolution
Implemented a self-healing automation script (detailed in Part 6) that checks the current home IP against the AWS SG every 5 minutes and updates it if a mismatch is detected.

---

## 3. Incident: Proxmox VLAN Crash (QEMU Exit Code 1)

### Symptom
Attempting to start a VM with a VLAN tag resulted in: `TASK ERROR: start failed: command '/usr/sbin/qm start 101' failed: exit code 1`.

### Root Cause
Proxmox VE 9.1.1 introduced a validation rule: a `bridge-vlan-aware` bridge *must* have at least one physical or dummy port attached. My internal virtual bridge was port-less, causing the kernel to refuse to bring it up.

### Resolution
Created a kernel **dummy interface** and attached it to the bridge:
```bash
# /etc/network/interfaces
auto dummy0
iface dummy0 inet manual
    pre-up ip link add dummy0 type dummy || true

auto vmbr1
iface vmbr1 inet manual
    bridge-ports dummy0
    bridge-vlan-aware yes
```

---

## Meta-Lesson: Silent Failures are the Enemy

Each of these incidents was caused by a silent failure in an unmonitored layer. Systems that fail loudly are easy to fix; those that fail silently cost hours of misdirected debugging.

**The takeaway: Build observable systems. If a critical component fails, it must make noise.**

---

## Next Part

**[Part 3 → VLANs & Zero-Trust Segmentation](/posts/hybrid-cloud-lab-part-3-vlans-and-segmentation/)** — How I moved away from a flat network to a secure enterprise model.
