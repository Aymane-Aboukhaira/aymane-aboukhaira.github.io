---
title: "Hybrid Cloud Lab Part 4: Enterprise Identity & Monitoring"
date: 2026-03-05 13:00:00 +0000
categories: [IT Infrastructure, Administration]
tags: [active-directory, windows-server, prometheus, grafana]
---


# Hybrid Cloud Lab — Part 4: Multi-Layer Security & Identity

In this part, I detail the **Defense-in-Depth** strategy implemented on the cloud edge. This isn't just about setting a password; it's about having three independent security layers that cover different failure modes.

| Threat | Mitigation | Layer |
|---|---|---|
| Guacamole Brute Force | Fail2ban (5 attempts = 1h ban) | 2 |
| Stolen AD Credentials | TOTP MFA (Physical factor) | 3 |
| Port Scans / Vulnerabilities | AWS Security Group (Minimalist) | 1 |
| Unauthorized SSH Access | SG: Home IP only + RSA Key | 1 |
| IAM S3 Ransomware | IAM write-only policies | 1 |

---

## 1. Layer 1 — AWS Security Group

The first perimeter acts at the AWS network level, dropping packets before they even reach the OS.

| Port | Protocol | Source | Usage |
|---|---|---|---|
| 443 | TCP | 0.0.0.0/0 | HTTPS for Guacamole |
| 22 | TCP | Home IP /32 | SSH Administration |
| 500/4500 | UDP | Home IP /32 | IPsec IKE/NAT-T |

---

## 2. Layer 2 — Fail2ban in a Docker Environment

One major challenge was that Docker bypasses the standard `iptables` INPUT chain. A standard Fail2ban setup does **nothing** for containerized services.

### The Solution: DOCKER-USER Chain
I configured Fail2ban to inject rules into the `DOCKER-USER` chain, which is the only chain Docker checks before its own routing rules.

```bash
# Example NPM audit filter targeting the DOCKER-USER chain
[npm-docker]
enabled = true
filter  = npm-general
logpath = /opt/npm/data/logs/proxy-host-*_access.log
action  = iptables-multiport[
    name="npm-docker",
    port="http,https",
    protocol="tcp",
    chain="DOCKER-USER"]
```

---

## 3. Layer 3 — TOTP MFA + AD LDAP

For the final layer, I implemented a multi-factor authentication flow that traverses the IPsec tunnel to authenticate against my local Active Directory.

### Authentication Flow:
1.  **User enters credentials**: Username + Password.
2.  **LDAP Validation**: Guacamole connects to `192.168.10.10` (DC01) via the IPsec tunnel to verify.
3.  **TOTP Prompt**: If valid, the user must enter a 6-digit code from a physical device (Google Authenticator).
4.  **Access Granted**: The encrypted RDP session is established.

```yaml
# Guacamole Environment Configuration
LDAP_HOSTNAME: 192.168.10.10
LDAP_USER_BASE_DN: CN=Users,DC=lab,DC=local
TOTP_ENABLED: "true"
```

This ensures that even if an attacker has your password, they are still blocked by the physical factor and the network-level IP whitelist.

---

## Next Part

**[Part 5 → Hybrid Connectivity & Secure Access](/posts/hybrid-cloud-lab-part-5-vpn-and-remote-access/)** — Connecting it all together with the IPsec Site-to-Site tunnel.
