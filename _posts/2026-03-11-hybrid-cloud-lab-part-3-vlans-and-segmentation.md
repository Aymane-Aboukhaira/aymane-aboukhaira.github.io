---
title: "Hybrid Cloud Lab Part 3: VLANs & Zero-Trust Segmentation"
date: 2026-03-11 12:00:00 +0000
categories: [IT Infrastructure, Networking]
tags: [pfsense, vlans, security, zero-trust]
---


# Hybrid Cloud Lab — Part 3: Zero-Trust VLAN Segmentation

A "flat" network where every device can communicate with every other device is a massive security vulnerability. In this post, I detail the transition from a flat `192.168.1.0/24` subnet to a segmented **Zero-Trust** architecture using VLANs and pfSense.

---

## 1. The Vulnerability of Flat Networks

In my initial setup, a compromise of `centos-vm1` (web application) would have granted an attacker direct Layer 3 access to:
- **DC01**: The domain controller and all Active Directory credentials.
- **OMV-NAS**: All centralized storage data.
- **Monitoring**: The ability to disable alerts and hide their tracks.

### Comparison: Before vs. After
| Scenario | Flat Network | Zero-Trust Segmentation |
|---|---|---|
| DMZ Compromised → DC01 | Direct Access | **BLOCKED** — DMZ-to-MGMT rule |
| DMZ Compromised → NAS | Direct Access | **BLOCKED** — DMZ-to-STORAGE rule |
| DMZ Compromised → Monitoring | Direct Access | **BLOCKED** — DMZ-to-SERVICES rule |
| DMZ → Internet | Allowed | **ALLOWED** — DMZ-to-WAN rule |

---

## 2. VLAN Strategy — Segmentation Plan

| VLAN | Name | Subnet | pfSense Gateway | DNS | Access Allowed From |
|---|---|---|---|---|---|
| 10 | MGMT | 192.168.10.0/24 | 192.168.10.1 | 192.168.10.10 | Admins Only |
| 20 | SERVICES | 192.168.20.0/24 | 192.168.20.1 | 192.168.10.10 | MGMT + specified VLANs |
| 30 | DMZ | 192.168.30.0/24 | 192.168.30.1 | 8.8.8.8 | Internet Only |
| 40 | STORAGE | 192.168.40.0/24 | 192.168.40.1 | 192.168.10.10 | MGMT + SERVICES |

---

## 3. pfSense Firewall Rules

I adopted a **deny-by-default** philosophy. Unless explicitly permitted, all traffic is blocked.

### DMZ Interface (VLAN 30) Isolation
1.  **BLOCK** | Src: DMZ net | Dst: MGMT (192.168.10.0/24) | *Prevents lateral movement to AD.*
2.  **BLOCK** | Src: DMZ net | Dst: SERVICES (192.168.20.0/24) | *Protects the monitoring stack.*
3.  **BLOCK** | Src: DMZ net | Dst: STORAGE (192.168.40.0/24) | *Impossible to exfiltrate data from NAS.*
4.  **PASS**  | Src: DMZ net | Dst: !RFC1918 | *Allows only public Internet access.*

### IPsec Interface Rules
One of the most critical steps is permitting traffic from the AWS VPC.
- **PASS** | Src: 10.0.0.0/16 | Dst: VLAN subnets | *Enables Guacamole-to-LDAP and RDP/SSH sessions.*

---

## 4. Proxmox Workaround: The Dummy Interface

In Proxmox 9.1.1, a `bridge-vlan-aware` bridge requires at least one port to be initialized. Since my internal bridge `vmbr1` was virtual, I had to use a kernel **dummy interface** as a placeholder:

```bash
# /etc/network/interfaces excerpt
auto dummy0
iface dummy0 inet manual
    pre-up ip link add dummy0 type dummy || true

auto vmbr1
iface vmbr1 inet manual
    bridge-ports dummy0
    bridge-vlan-aware yes
```

Without this workaround, every VM with a VLAN tag failed to boot with a `QEMU exit code 1`.

---

## Next Part

**[Part 4 → Enterprise Identity & Monitoring](/posts/hybrid-cloud-lab-part-4-identity-and-monitoring/)** — Configuring Active Directory and the Prometheus stack.
