# 🛡️ Live Security Operations Lab

> **A fully operational, end-to-end Security Operations Center (SOC) environment built from free and open-source tools - simulating real-world threat detection, vulnerability management, and incident response aligned to CompTIA CySA+ CS0-003/CS0-004 and MITRE ATT&CK.**

---

## 📌 What This Project Is

This is not a notes dump. This is not a cheat sheet.

This is a **working SOC lab** you can clone, deploy, and operate - where real attacks are simulated, real alerts fire, real playbooks execute, and real reports get generated. Every component maps directly to a CySA+ domain and to how actual security teams operate.

Built entirely with **free and open-source tools**. Runs on modest hardware or a free-tier cloud VM. Documented so thoroughly that a junior analyst can follow it and a senior engineer will respect the architecture.

---

## 🎯 Who This Is For

| Person | How This Helps |
|---|---|
| **CySA+ candidates** | See every exam domain working in a live environment |
| **Junior SOC analysts** | Steal the playbooks, detection rules, and runbooks for real work |
| **Homelab builders** | Fork and spin up a complete detection stack in hours |
| **Hiring managers** | A working lab that demonstrates operational security thinking |
| **Blue teamers** | Wazuh + Suricata + Zeek configs you can adapt directly |

---

## 🗺️ Project Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     ATTACK SIMULATION LAYER                     │
│          Kali Linux VM  ──►  Metasploitable / DVWA              │
└───────────────────────────────┬─────────────────────────────────┘
                                │ (malicious traffic)
┌───────────────────────────────▼─────────────────────────────────┐
│                      DETECTION LAYER                            │
│   Suricata (IDS/IPS)  +  Zeek (network analysis)                │
│   Wazuh Agent (endpoint telemetry on all VMs)                   │
└───────────────────────────────┬─────────────────────────────────┘
                                │ (logs + alerts)
┌───────────────────────────────▼─────────────────────────────────┐
│                   SIEM / ANALYSIS LAYER                         │
│        Wazuh Manager  +  OpenSearch / Kibana Dashboard          │
│        TheHive (case management) + Cortex (auto-enrichment)     │
└───────────────────────────────┬─────────────────────────────────┘
                                │ (enriched incidents)
┌───────────────────────────────▼─────────────────────────────────┐
│               VULNERABILITY MANAGEMENT LAYER                    │
│         OpenVAS / Greenbone  +  CVSS scoring scripts            │
│         Dependency-Track (SCA/SBOM)                             │
└───────────────────────────────┬─────────────────────────────────┘
                                │ (findings)
┌───────────────────────────────▼─────────────────────────────────┐
│              INCIDENT RESPONSE & REPORTING LAYER                │
│     Playbooks (this repo)  +  Velociraptor (DFIR)               │
│     Automated IR reports generated per incident                 │
└─────────────────────────────────────────────────────────────────┘
```

---


