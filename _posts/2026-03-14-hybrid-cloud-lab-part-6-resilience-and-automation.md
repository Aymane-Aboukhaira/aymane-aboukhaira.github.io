---
title: "Hybrid Cloud Lab Part 6: Resilience & Automation"
date: 2026-03-14 10:00:00 +0000
categories: [IT Infrastructure, DevOps]
tags: [storage, backups, automation, bash, disaster-recovery]
---


# Hybrid Cloud Lab — Part 6: Resilience & Automation

We’ve reached the final part of my Hybrid Cloud Infrastructure Lab series. Today, I’m detailing the "Safety Net:" how I automated the most frustrating parts of IP management and implemented a 3-tier disaster recovery (DR) strategy that guarantees data survival.

---

## 1. The Dynamic IP Deadlock

Residential ISPs assign dynamic public IPs. When your home IP changes, three things break simultaneously:
1.  **AWS Security Group**: The old IP is no longer authorized for SSH/IPsec.
2.  **Fail2ban `ignoreip`**: You risk being banned from your own server.
3.  **IPsec Tunnel**: The SG blocks the new IP, killing the VPN.

### The Self-Healing Architecture
I solved this with a two-layer synchronization system:
- **On-Premise Cron**: Every 5 minutes, a CentOS VM updates a **DuckDNS** hostname with the current home IP.
- **AWS Script (`update-sg-ip.sh`)**: Every 5 minutes, EC2 resolves the DuckDNS hostname, compares it to the SG rules, and updates them automatically if a change is detected.

```bash
# update-sg-ip.sh logic excerpt
HOME_IP=$(dig +short $DDNS_HOST | tail -1)
CURRENT_SG_IP=$(aws ec2 describe-security-groups --group-ids $SG_ID ...)

if [ "$HOME_IP" != "$CURRENT_SG_IP" ]; then
    aws ec2 authorize-security-group-ingress --cidr ${HOME_IP}/32 ...
    sed -i "s|ignoreip = .*|ignoreip = 127.0.0.1/8 $HOME_IP|" /etc/fail2ban/jail.local
    systemctl restart fail2ban
fi
```

---

## 2. 3-Tier Disaster Recovery (3-2-1 Rule)

I implemented a strategy to ensure that even a total hardware failure on-premise or a compromised cloud instance wouldn't be fatal.

| Tier | Media | Location | Frequency | Content |
|---|---|---|---|---|
| **1** | USB EXT4 (Air-gapped) | On-site | Manual | Full VM Snapshots (.vma.zst) |
| **2** | AWS S3 (Stochastic) | Cloud | Daily 2 AM | EC2 Configs: NPM, Guacamole, IPsec |
| **3** | AWS S3 (Stochastic) | Cloud | Daily 3 AM | pfSense `config.xml`, Monitoring, DBs |

### Anti-Ransomware Protection
The S3 bucket uses a **write-only IAM policy**. The backup bot can `PutObject`, but it cannot `DeleteObject`. With S3 Versioning enabled, this means that even if the backup credentials are stolen, the historical backups cannot be purged.

---

## 3. Conclusion: The Lab is Never Finished

Building this was a deep dive into the realities of modern sysadmin work. It taught me that enterprise engineering isn't just about the initial deploy; it's about the automation that keeps the system alive and the backups that give you sleep at night.

The full repository with all configurations and scripts is available on my GitHub.

Thank you for following along with this 6-part journey!

**Aymane Aboukhaira**
*Networks & Systems — ISMONTIC, Tangier*
