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

### 5.1 Create the VM

```bash
VBoxManage createvm --name "win10-victim" --ostype Windows10_64 --register

VBoxManage modifyvm "win10-victim" \
  --memory 4096 \
  --cpus 2 \
  --nic1 hostonly --hostonlyadapter1 vboxnet0 \
  --graphicscontroller vboxsvga \
  --vram 128 \
  --clipboard bidirectional

VBoxManage createhd --filename ~/VMs/win10-victim.vdi --size 61440
VBoxManage storagectl "win10-victim" --name "SATA" --add sata
VBoxManage storageattach "win10-victim" --storagectl "SATA" \
  --port 0 --device 0 --type hdd --medium ~/VMs/win10-victim.vdi

VBoxManage storageattach "win10-victim" --storagectl "SATA" \
  --port 1 --device 0 --type dvddrive \
  --medium /path/to/Win10_22H2_English_x64.iso
```

### 5.2 Post-install Windows configuration

Run these in an **elevated PowerShell** after installation:

```powershell
# Set static IP
New-NetIPAddress -InterfaceAlias "Ethernet" `
  -IPAddress 192.168.56.20 `
  -PrefixLength 24

# Enable PowerShell script execution (for Atomic Red Team)
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope LocalMachine -Force

# Enable WinRM (required for remote management testing)
Enable-PSRemoting -Force

# Enable SMB for lateral movement simulation
Set-SmbServerConfiguration -EnableSMB2Protocol $true -Force

# Disable Windows Defender real-time protection (lab only — re-enable after)
# WARNING: Only do this in an isolated lab VM
Set-MpPreference -DisableRealtimeMonitoring $true

# Enable PowerShell Script Block Logging (feeds into Wazuh detections)
$regPath = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging"
New-Item -Path $regPath -Force | Out-Null
Set-ItemProperty -Path $regPath -Name "EnableScriptBlockLogging" -Value 1

# Enable Command Process Auditing
auditpol /set /subcategory:"Process Creation" /success:enable /failure:enable
```

---

##   hydra \
  crackmapexec \
  impacket-scripts \
  smbclient \
  nmap \
  metasploit-framework

# Install Mimikatz (via Wine for Windows binary testing, or use pypykatz)
pip3 install pypykatz
```

---

## 7. Wazuh Deployment

Wazuh is deployed on `siem-server` using the official all-in-one installer, which installs the Wazuh manager, indexer (OpenSearch), and dashboard in a single script.

```bash
# On siem-server (192.168.56.30)

# Download installer
curl -sO https://packages.wazuh.com/4.x/wazuh-install.sh
curl -sO https://packages.wazuh.com/4.x/config.yml

# Edit config.yml — set node IPs to 192.168.56.30
sed -i 's/<node_ip>/192.168.56.30/g' config.yml

# Run installer (takes 5–10 minutes)
sudo bash wazuh-install.sh -a

# Save the admin password printed at the end of install — you'll need it for the dashboard
```

### 7.1 Verify services

```bash
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard

# All three should show: active (running)
```

### 7.2 Access the dashboard

Open a browser on your host machine and navigate to:

```
https://192.168.56.30
```

Login with `admin` and the password saved during install.

> **[SCREENSHOT]** — Wazuh dashboard home page showing 0 agents enrolled, system health green.

---

## 8. Agent Enrollment

### 8.1 Install the Wazuh agent on Windows

Run in **elevated PowerShell** on `win10-victim`:

```powershell
# Download agent installer
Invoke-WebRequest -Uri "https://packages.wazuh.com/4.x/windows/wazuh-agent-4.x.x-1.msi" `
  -OutFile "C:\wazuh-agent.msi"

# Install and point to manager
msiexec /i C:\wazuh-agent.msi `
  WAZUH_MANAGER="192.168.56.30" `
  WAZUH_AGENT_NAME="win10-victim" /quiet

# Start the agent service
NET START WazuhSvc
```

### 8.2 Verify enrollment on the manager

```bash
# On siem-server
sudo /var/ossec/bin/agent_control -l
# Should list win10-victim with status: Active
```

> **[SCREENSHOT]** — Wazuh dashboard Agents page showing `win10-victim` enrolled and Active.

---

## 9. Log Source Verification

Before building detections, confirm the expected log sources are flowing into Wazuh.

### 9.1 Windows Event Logs

In the Wazuh dashboard, navigate to **Security Events** and filter by agent `win10-victim`. Confirm you see:

| Event ID | Description |
|---|---|
| 4624 | Successful logon |
| 4625 | Failed logon |
| 4688 | Process creation |
| 4698 | Scheduled task created |
| 4104 | PowerShell Script Block |

### 9.2 Trigger a test event

```powershell
# On win10-victim — run a benign PowerShell command to generate Event ID 4104
powershell.exe -Command "Write-Host 'Wazuh test event'"
```

Check the Wazuh dashboard for the resulting alert within 30 seconds.

> **[SCREENSHOT]** — Wazuh Security Events showing PowerShell Event ID 4104 from win10-victim.

### 9.3 Confirm Wazuh ossec.conf includes Windows log channels

```bash
# On siem-server — verify agent config is pushed correctly
sudo cat /var/ossec/etc/shared/default/agent.conf
```

The config should include these `<localfile>` entries (add them if missing):

```xml
<ossec_config>
  <localfile>
    <location>Security</location>
    <log_format>eventchannel</log_format>
  </localfile>
  <localfile>
    <location>Microsoft-Windows-PowerShell/Operational</location>
    <log_format>eventchannel</log_format>
  </localfile>
  <localfile>
    <location>Microsoft-Windows-Sysmon/Operational</location>
    <log_format>eventchannel</log_format>
  </localfile>
</ossec_config>
```

---

## 10. What's Next

With the lab environment running and logs verified, the next steps are:

| Step | Location |
|---|---|
| Custom Sigma detection rules | [`detections/`](./detections/) |
| Per-rule detection walkthroughs | [`detections/walkthroughs/`](./detections/walkthroughs/) |
| Atomic Red Team simulations | [`simulations/`](./simulations/) |
| Hardening recommendations | [`hardening-recommendations.md`](./hardening-recommendations.md) |
