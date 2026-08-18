# SentinelLab

> A Blue Team home lab for detection engineering and incident response — SOC Tier 1

![Status](https://img.shields.io/badge/status-active-brightgreen)
![Hypervisor](https://img.shields.io/badge/hypervisor-Proxmox%20VE%209.2-orange)
![SIEM](https://img.shields.io/badge/SIEM-Wazuh%204.9-blue)
![Target](https://img.shields.io/badge/target-SOC%20Tier%201-lightgrey)

---

## Purpose

SentinelLab is a personal lab built to reproduce the day-to-day work of a **Tier 1 SOC analyst**: log collection and correlation, intrusion detection, alert investigation, and incident response.

The project is part of a career change into cybersecurity (École 42 Lyon, France), with the goal of landing a Blue Team internship.

**Guiding principle:** everything is built by hand, without all-in-one scripts, in order to understand each layer — networking, hypervisor, agents, and detection rules.

---

## Architecture

```
                        Router / Gateway — 192.168.1.254
                                    │
                        ┌───────────┴───────────┐
                        │                       │
                  Powerline link          Mac M3 (admin
                  (TP-Link AV1000)         workstation)
                        │                  192.168.1.31
                        │
          ┌─────────────┴──────────────────────────────┐
          │        Dell OptiPlex 3080 Micro            │
          │        Proxmox VE 9.2 — 192.168.1.200      │
          │                                            │
          │   ┌──────────────────────────────────┐     │
          │   │  VM 100 — ubuntu-wazuh           │     │
          │   │  Ubuntu Server 24.04 LTS         │     │
          │   │  Wazuh Manager + Indexer + Dash  │     │
          │   │  192.168.1.87                    │     │
          │   │  4 vCPU / 8 GB RAM / 80 GB       │     │
          │   └──────────────────────────────────┘     │
          │                                            │
          │   [ VM 101 — Windows 10 (endpoint)  ]      │
          │   [ VM 102 — Kali Linux (attacker)  ]      │
          │   [ VM 103 — pfSense (firewall)     ]      │
          └────────────────────────────────────────────┘
```

### Hardware

| Component | Role | Specifications |
|---|---|---|
| **Dell OptiPlex 3080 Micro** | Hypervisor | i5-10500T (6c/12t), 32 GB DDR4, 256 GB M.2 SSD + free SATA bay |
| **Mac M3 16 GB** | Admin workstation | SSH access + Proxmox web UI |
| **Beelink EQ12** | Secondary server | Reserved for upcoming services (TheHive) |
| **TP-Link AV1000** | Network link | Gigabit powerline — the Dell has no WiFi card |

> **Design decision:** a powerline (HomePlug) link was chosen over a USB WiFi dongle. USB WiFi adapters suffer from driver issues under Proxmox and don't bridge cleanly (`vmbr0`), which VMs depend on for network access.

---

## Stack

| Layer | Tool | Status |
|---|---|---|
| Hypervisor | Proxmox VE 9.2 | Deployed |
| SIEM | Wazuh 4.9 (Manager, Indexer, Dashboard) | Deployed |
| Server OS | Ubuntu Server 24.04 LTS | Deployed |
| Windows endpoint | Windows 10 + Sysmon + Wazuh agent | Planned |
| Attack simulation | Kali Linux (Nmap, Hydra, Metasploit) | Planned |
| Firewall / segmentation | pfSense | Planned |
| Case management | TheHive | Planned |

---

## Repository layout

```
SentinelLab/
├── README.md
├── docs/
│   ├── 01-proxmox-installation.md
│   ├── 02-wazuh-deployment.md
│   └── troubleshooting.md
├── architecture/
│   └── network-diagram.png
├── scenarios/
│   ├── 01-ssh-bruteforce/
│   ├── 02-network-recon/
│   └── 03-windows-persistence/
├── playbooks/
│   ├── PICERL-template.md
│   ├── bruteforce-response.md
│   └── malware-triage.md
├── wazuh/
│   ├── custom-rules.xml
│   └── ossec.conf.example
└── notes/
    └── resources.md
```

---

## Detection scenarios

Every scenario follows the same structure: **context → attack execution → logs generated → Wazuh rule triggered → investigation → response**.

### 1. SSH brute force — *in preparation*
Dictionary attack from Kali against the Ubuntu server. Goal: trigger Wazuh rule 5763, analyse `auth.log`, document the investigation workflow, and test active response (IP blocking).

### 2. Network reconnaissance — *in preparation*
Nmap sweep from a compromised host. Goal: detect internal reconnaissance, identify the source, and assess scope.

### 3. Windows persistence — *in preparation*
Malicious scheduled task created on the Windows endpoint. Goal: Sysmon + Wazuh correlation, MITRE ATT&CK mapping (T1053).

---

## Response playbooks

Playbooks follow the **PICERL** model (Preparation, Identification, Containment, Eradication, Recovery, Lessons Learned), aligned with NIST SP 800-61.

---

## Issues encountered & resolutions

This section documents the real problems hit during deployment — usually where the most useful learning happens.

| Issue | Diagnosis | Resolution |
|---|---|---|
| `No Hard Disk found` during Proxmox install | NVMe SSD set to Intel RST/RAID mode, not recognised by the Linux kernel | Switched BIOS SATA operation to **AHCI** |
| Proxmox UI unreachable after install | IP entered during setup (`192.168.100.2`) was outside the actual subnet (`192.168.1.0/24`) | Corrected `/etc/network/interfaces` and `/etc/hosts` |
| `Temporary failure in name resolution` when downloading ISOs | DNS server pointed at a non-existent gateway | Reconfigured DNS to the local router + public resolver |
| noVNC console and web Shell timing out | WebSocket never reaches `pveproxy`; `termproxy` and `pvedaemon` verified healthy server-side | Worked around with an **SSH tunnel + native VNC client** (below) — root cause still to be isolated |

### Console workaround: SSH tunnel to VNC

When the web console is unavailable, QEMU's VNC socket is still reachable locally on the hypervisor:

```bash
# On the hypervisor — expose the Unix socket on a local port
socat TCP-LISTEN:5901,bind=127.0.0.1,reuseaddr,fork \
      UNIX-CONNECT:/var/run/qemu-server/100.vnc &

# On the admin workstation — SSH tunnel
ssh -L 5901:127.0.0.1:5901 root@192.168.1.200
```

Then connect to `vnc://localhost:5901` with any VNC client.

> A VNC password must be set beforehand via `qm monitor <vmid>` followed by `set_password vnc <password>`.

---

## Reproducing this lab

### Requirements
- Host machine: 4 cores minimum, 16 GB RAM (32 GB recommended)
- Wired or powerline connection to the router
- 8 GB USB stick for the Proxmox installer

### Installing Proxmox VE

```bash
# From macOS — write the ISO to a USB stick
diskutil list                              # identify the target disk
diskutil unmountDisk /dev/diskN
sudo dd if=proxmox-ve_9.2-1.iso of=/dev/rdiskN bs=1m status=progress
```

Boot from the stick and follow the installer. Double-check that the IP addressing matches your local network **before** confirming — getting this wrong leaves the web UI unreachable.

### Deploying Wazuh

```bash
curl -sO https://packages.wazuh.com/4.9/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

The script installs the indexer, server, and dashboard. Generated credentials are printed at the end of the run — **they cannot be retrieved afterwards**.

Dashboard available at `https://<server-ip>`.

---
