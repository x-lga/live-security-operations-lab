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

