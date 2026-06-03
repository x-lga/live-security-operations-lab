# 01 - Network Design

> **Security lab networks must be isolated.** A misconfigured virtual network that bridges to your home network and then gets hit by Metasploit payloads is a bad day. This document sets up a clean, segmented virtual network that keeps attack traffic contained.

---

## 🗺️ Network Topology Overview

```
YOUR HOST MACHINE
├── Physical NIC (your home/office network) - 192.168.1.0/24 or similar
│
├── VirtualBox Host-Only Adapter: vboxnet0
│   └── SOC-Management Network: 192.168.56.0/24
│       ├── Wazuh Manager:        192.168.56.10
│       ├── Suricata/Zeek Sensor: 192.168.56.11
│       ├── TheHive + Cortex:     192.168.56.12
│       ├── OpenVAS:              192.168.56.13
│       ├── Velociraptor:         192.168.56.14
│       ├── Kali Linux:           192.168.56.20
│       ├── Metasploitable 2:     192.168.56.30
│       └── DVWA (Ubuntu):        192.168.56.31
│
└── VirtualBox Internal Network: SOC-Lab-Net (intnet)
    └── Attack/Target Network: 10.10.10.0/24
        ├── Kali Linux:          10.10.10.1   (attacker)
        ├── Metasploitable 2:    10.10.10.10  (target)
        └── DVWA:                10.10.10.11  (target)
```

