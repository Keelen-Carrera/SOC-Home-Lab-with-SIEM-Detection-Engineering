# 01 — SOC Home Lab with SIEM Detection Engineering

> **Goal:** Deploy a three-VM home SOC environment, ingest logs into Wazuh, and validate detection engineering from attack simulation through alert triage.

---

## Table of Contents

1. [Lab Overview](#1-lab-overview)
2. [Network Diagram](#2-network-diagram)
3. [Prerequisites](#3-prerequisites)
4. [VM Build — Ubuntu SIEM Server](#4-vm-build--ubuntu-siem-server)
5. [VM Build — Windows 10 Victim](#5-vm-build--windows-10-victim)
6. [VM Build — Kali Linux Attacker](#6-vm-build--kali-linux-attacker)
7. [Wazuh Deployment](#7-wazuh-deployment)
8. [Agent Enrollment](#8-agent-enrollment)
9. [Log Source Verification](#9-log-source-verification)
10. [What's Next](#10-whats-next)

---

## 1. Lab Overview

| VM | OS | Role | RAM | Disk |
|---|---|---|---|---|
| `siem-server` | Ubuntu Server 22.04 LTS | Wazuh manager + dashboard | 4 GB | 50 GB |
| `win10-victim` | Windows 10 Pro (22H2) | Monitored endpoint | 4 GB | 60 GB |
| `kali-attacker` | Kali Linux 2024.x | Attack simulation | 2 GB | 40 GB |

All three VMs share a **host-only network** (`192.168.56.0/24`) so traffic is isolated from your home network. An optional NAT adapter on the SIEM and Kali VMs allows internet access for package installs.

---

## 2. Network Diagram

```
                        HOST MACHINE
        ┌───────────────────────────────────────────┐
        │                                           │
        │   ┌─────────────────┐                     │
        │   │  kali-attacker  │                     │
        │   │  192.168.56.10  │  (Kali Linux)       │
        │   └────────┬────────┘                     │
        │            │  attack traffic               │
        │            │  (SSH brute, SMB, PS exec)    │
        │            ▼                               │
        │   ┌─────────────────┐    Wazuh agent      │
        │   │  win10-victim   │◄───────────────┐    │
        │   │  192.168.56.20  │  (Windows 10)  │    │
        │   └─────────────────┘                │    │
        │                                      │    │
        │                             agent    │    │
        │                             logs     │    │
        │                                      ▼    │
        │   ┌──────────────────────────────────────┐│
        │   │           siem-server                ││
        │   │         192.168.56.30                ││
        │   │                                      ││
        │   │  ┌────────────────┐  ┌────────────┐  ││
        │   │  │ Wazuh Manager  │  │  Wazuh     │  ││
        │   │  │  (port 1514)   │  │ Dashboard  │  ││
        │   │  │                │  │ (port 443) │  ││
        │   │  └────────────────┘  └────────────┘  ││
        │   │         Ubuntu Server 22.04           ││
        │   └──────────────────────────────────────┘│
        │                                           │
        │         Host-Only Network: 192.168.56.0/24│
        └───────────────────────────────────────────┘
```

**Traffic flows:**
- Kali → Windows: simulated attacks (brute force, lateral movement, persistence)
- Windows → Wazuh Manager: agent heartbeat + log shipping (port 1514/UDP)
- Browser → Wazuh Dashboard: analyst alert review (HTTPS port 443)

---

## 3. Prerequisites

- **Hypervisor:** VirtualBox 7.x (free) or VMware Workstation Pro/Player
- **Host RAM:** 12 GB minimum (10 GB allocated to VMs + 2 GB for host)
- **Host Disk:** 200 GB free
- **ISOs needed:**
  - Ubuntu Server 22.04 LTS — `ubuntu-22.04-live-server-amd64.iso`
  - Windows 10 Pro — MSDN/evaluation ISO from Microsoft
  - Kali Linux — `kali-linux-2024.x-installer-amd64.iso`

### VirtualBox: create the host-only network

```bash
# Run from your host terminal
VBoxManage hostonlyif create
VBoxManage hostonlyif ipconfig vboxnet0 --ip 192.168.56.1 --netmask 255.255.255.0
VBoxManage dhcpserver modify --ifname vboxnet0 --disable
```

---

## 4. VM Build — Ubuntu SIEM Server

### 4.1 Create the VM

```bash
VBoxManage createvm --name "siem-server" --ostype Ubuntu_64 --register

VBoxManage modifyvm "siem-server" \
  --memory 4096 \
  --cpus 2 \
  --nic1 hostonly --hostonlyadapter1 vboxnet0 \
  --nic2 nat \
  --graphicscontroller vmsvga \
  --vram 16

VBoxManage createhd --filename ~/VMs/siem-server.vdi --size 51200
VBoxManage storagectl "siem-server" --name "SATA" --add sata
VBoxManage storageattach "siem-server" --storagectl "SATA" \
  --port 0 --device 0 --type hdd --medium ~/VMs/siem-server.vdi

VBoxManage storageattach "siem-server" --storagectl "SATA" \
  --port 1 --device 0 --type dvddrive \
  --medium /path/to/ubuntu-22.04-live-server-amd64.iso
```

### 4.2 Post-install OS configuration

After Ubuntu installation, set the static IP and update packages:

```bash
# /etc/netplan/00-installer-config.yaml
sudo tee /etc/netplan/00-installer-config.yaml <<'EOF'
network:
  version: 2
  ethernets:
    enp0s3:
      addresses: [192.168.56.30/24]
      nameservers:
        addresses: [8.8.8.8]
    enp0s8:
      dhcp4: true
EOF

sudo netplan apply
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git net-tools
```

---

## 5. VM Build — Windows 10 Victim
