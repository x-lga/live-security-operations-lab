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
