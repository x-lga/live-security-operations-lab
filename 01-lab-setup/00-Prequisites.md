# 00 - Prerequisites

> **Before you touch a single install command**, read this entire document. Skipping prerequisites is the #1 reason lab builds fail halfway through. Ten minutes here saves hours of debugging later.

---

## 📋 What You Need Before Starting

### Hardware (Physical Host Machine)

| Resource | Minimum | Recommended | Notes |
|---|---|---|---|
| **RAM** | 16 GB | 32 GB | Wazuh alone wants 4 GB; the full stack needs ~20 GB active |
| **CPU** | 4 cores / 8 threads | 8 cores / 16 threads | Enable VT-x / AMD-V in BIOS |
| **Disk** | 100 GB free | 200 GB SSD | HDD works but Wazuh indexing on spinning disk is painful |
| **Network** | 1 NIC | 2 NICs | A second NIC for dedicated monitoring is optional but clean |
| **OS** | Windows 10/11 | Ubuntu 22.04 LTS host | Linux host = fewer hypervisor quirks |

> **⚠️ RAM is the hard constraint.** If you only have 16 GB, you will need to be strategic about which VMs are running simultaneously. The "Minimal RAM" section at the bottom of this document covers that.

---

### No Powerful Machine? Cloud Options

If your physical machine can't handle this, use a cloud provider. All of these have free tiers sufficient for the lab:

| Provider | Free Offering | Notes |
|---|---|---|
| **Oracle Cloud Free Tier** | 4 OCPUs + 24 GB RAM (always free) | Best option - generous specs |
| **Google Cloud** | $300 free credit (90 days) | Good for quick exploration |
| **AWS Free Tier** | t2.micro / t3.micro (1 year) | Too small for full stack; use for agent-only nodes |


**Oracle Cloud is the recommended cloud option.** The always-free ARM instance (4 OCPU, 24 GB RAM) is enough to run Wazuh Manager + Suricata + one target VM.

Sign up: https://www.oracle.com/cloud/free/

---

## 💿 Software Downloads

Download these before starting. Trying to download large ISOs mid-setup is frustrating.

### Hypervisor (pick one)

**Option A: VirtualBox (recommended for beginners)**
- Download: https://www.virtualbox.org/wiki/Downloads
- Also download the **VirtualBox Extension Pack** (same page) - needed for USB passthrough and improved networking
- Version: 7.x or latest stable

**Option B: VMware Workstation Player (free for personal use)**
- Download: https://www.vmware.com/products/workstation-player.html
- Version: 17.x or latest
- Note: VMware Player has better performance than VirtualBox but fewer features

**Option C: KVM/QEMU (Linux hosts only, best performance)**
```bash
sudo apt install qemu-kvm libvirt-daemon-system virt-manager bridge-utils
```
- Use if your host is Linux and you want native performance
- `virt-manager` gives you a GUI

---

### Virtual Machine Images

Download these ISOs/OVAs and store them in a dedicated folder:

```
~/lab-isos/
├── ubuntu-22.04.x-live-server-amd64.iso    (~1.4 GB)
├── kali-linux-2024.x-installer-amd64.iso   (~3.5 GB)
├── metasploitable-linux-2.0.0.zip          (~850 MB)
└── (DVWA runs inside Ubuntu - no separate download)
```

| Image | Download URL | Used For |
|---|---|---|
| **Ubuntu Server 22.04 LTS** | https://ubuntu.com/download/server | Wazuh Manager, Suricata sensor, TheHive, OpenVAS nodes |
| **Kali Linux** | https://www.kali.org/get-kali/#kali-installer-images | Attack simulation |
| **Metasploitable 2** | https://sourceforge.net/projects/metasploitable/ | Vulnerable target (download the `.zip`, extract the `.vmdk`) |


> 💡 **Verify your downloads.** Always check the SHA256 hash against what the download page provides.
> ```bash
> sha256sum ubuntu-22.04.x-live-server-amd64.iso
> ```

---

## 🔐 Accounts to Create (All Free)

Create these accounts now - some require email verification that takes time:

| Service | URL | Why You Need It |
|---|---|---|
| **GitHub** | https://github.com | Clone this repo, version-control your configs |
| **VirusTotal** | https://www.virustotal.com | IOC enrichment in TheHive/Cortex analyzers |
| **AbuseIPDB** | https://www.abuseipdb.com | IP reputation lookups (free API key) |
| **Shodan** | https://www.shodan.io | Optional: asset intelligence enrichment |
| **AlienVault OTX** | https://otx.alienvault.com | Threat intelligence feeds for MISP |

Keep your API keys in a password manager. You'll plug them into Cortex analyzers in a later step.

---

## 🛠️ Host Machine Preparation

### On Linux (Ubuntu/Debian Host)

```bash
# Update your system first — always
sudo apt update && sudo apt upgrade -y

# Install VirtualBox
sudo apt install virtualbox virtualbox-ext-pack -y

# Add your user to the vboxusers group
sudo usermod -aG vboxusers $USER

# Install useful utilities
sudo apt install curl wget git htop net-tools nmap -y

# Log out and back in for group changes to take effect
```


### On Windows Host

1. Install VirtualBox using the downloaded installer (run as Administrator)
2. Install the Extension Pack (double-click the `.vbox-extpack` file after VirtualBox is installed)
3. Enable Hyper-V compatibility if prompted (VirtualBox 7.x handles this better than earlier versions)
4. Install Git for Windows: https://gitforwindows.org/
5. Install Windows Terminal: https://aka.ms/terminal (makes your life easier)


### BIOS/UEFI Settings (Critical)

Before VMs will run, you must enable hardware virtualization in your BIOS:

- **Intel processors**: Enable **VT-x** (Intel Virtualization Technology)
- **AMD processors**: Enable **AMD-V** (AMD Virtualization)
- Also enable **VT-d** / **AMD-Vi** if available (IOMMU for better device passthrough)

How to access BIOS: Press F2, F10, F12, or Delete during boot (varies by manufacturer).

> If you see an error like `VT-x is disabled in the BIOS` when starting a VM, this is your problem.

---

## 📁 Folder Structure Convention

Create this folder structure on your host before you start. Consistency here prevents confusion later:

```bash
mkdir -p ~/cysa-lab/{isos,snapshots,configs,logs}
```

| Folder | Purpose |
|---|---|
| `~/cysa-lab/isos/` | Store all downloaded ISO and OVA files |
| `~/cysa-lab/snapshots/` | VirtualBox snapshot exports (backup clean states) |
| `~/cysa-lab/configs/` | Copies of your VM config files |
| `~/cysa-lab/logs/` | Exported logs for offline analysis |

---

## 🌐 Network Planning (Brief Intro)

You'll create **two virtual networks** in VirtualBox:

| Network | Type | Purpose |
|---|---|---|
| `SOC-Management` | Host-Only (192.168.56.0/24) | Admin access to all VMs from your host |
| `SOC-Lab-Net` | Internal Network | Isolated traffic between VMs; no internet |

Full details are in [`01-network-design.md`](01-network-design.md). Just know now that your VMs will **not** have internet access during attack simulations — this is intentional and important.

---

## 🗃️ VM Allocation Plan

Plan your VM memory allocation before you start creating machines:

| VM | RAM | CPU | Disk | Purpose |
|---|---|---|---|---|
| **Wazuh Manager** | 6 GB | 2 cores | 50 GB | SIEM brain + indexer |
| **Suricata/Zeek Sensor** | 2 GB | 2 cores | 20 GB | Network monitoring |
| **TheHive + Cortex** | 4 GB | 2 cores | 30 GB | Case management |
| **OpenVAS** | 4 GB | 2 cores | 30 GB | Vulnerability scanning |
| **Velociraptor** | 2 GB | 1 core | 20 GB | DFIR |
| **Kali Linux** | 2 GB | 2 cores | 30 GB | Attack simulation |
| **Metasploitable 2** | 512 MB | 1 core | 8 GB | Vulnerable target |
| **DVWA (Ubuntu)** | 1 GB | 1 core | 15 GB | Web app target |
| **TOTAL** | **~21.5 GB** | **13 cores** | **~203 GB** | |




