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

## 📂 Repository Structure

```
cysa-lab/
│
├── README.md                        ← You are here
│
├── lab-setup/
│   ├── 00-prerequisites.md          ← Hardware, software, accounts needed
│   ├── 01-network-design.md         ← Virtual network topology
│   ├── 02-wazuh-setup.md            ← SIEM installation and config
│   ├── 03-suricata-setup.md         ← IDS/IPS installation and config
│   ├── 04-zeek-setup.md             ← Network analysis setup
│   ├── 05-thehive-cortex-setup.md   ← Case management setup
│   ├── 06-openvas-setup.md          ← Vulnerability scanner setup
│   ├── 07-velociraptor-setup.md     ← DFIR tool setup
│   └── 08-attack-simulation.md      ← Kali + target VM setup
│
├── detection-rules/
│   ├── README.md
│   ├── suricata/
│   │   ├── local.rules              ← Custom Suricata rules
│   │   └── rules-explained.md
│   ├── wazuh/
│   │   ├── custom-rules.xml         ← Custom Wazuh detection rules
│   │   ├── custom-decoders.xml      ← Log decoders
│   │   └── rules-explained.md
│   └── sigma/
│       ├── lateral-movement.yml
│       ├── persistence.yml
│       └── exfiltration.yml
│
├── playbooks/
│   ├── README.md
│   ├── IR-001-malware-infection.md
│   ├── IR-002-brute-force-attack.md
│   ├── IR-003-port-scan-recon.md
│   ├── IR-004-data-exfiltration.md
│   ├── IR-005-web-application-attack.md
│   ├── IR-006-insider-threat.md
│   └── IR-007-ransomware.md
│
├── vulnerability-management/
│   ├── README.md
│   ├── scan-procedures.md
│   ├── cvss-scoring-guide.md
│   ├── remediation-workflow.md
│   ├── sample-reports/
│   │   └── vuln-report-template.md
│   └── scripts/
│       ├── cvss-calculator.py
│       └── vuln-report-generator.py
│
├── threat-hunting/
│   ├── README.md
│   ├── hunting-methodology.md
│   ├── hypothesis-library.md
│   ├── hunt-001-beaconing.md
│   ├── hunt-002-lateral-movement.md
│   ├── hunt-003-credential-dumping.md
│   └── hunt-004-c2-communication.md
│
├── incident-response/
│   ├── README.md
│   ├── ir-plan.md
│   ├── communication-templates.md
│   ├── chain-of-custody.md
│   ├── evidence-collection.md
│   └── post-incident-report-template.md
│
├── scripts/
│   ├── README.md
│   ├── setup/
│   │   ├── install-wazuh-agent.sh
│   │   ├── install-suricata.sh
│   │   └── install-zeek.sh
│   ├── detection/
│   │   ├── ioc-checker.py
│   │   ├── log-analyzer.sh
│   │   └── baseline-checker.py
│   └── reporting/
│       ├── generate-ir-report.py
│       └── weekly-summary.sh
│
├── docs/
│   ├── cysa-domain-mapping.md
│   ├── mitre-attack-mapping.md
│   ├── tool-glossary.md
│   └── references.md
│
└── dashboards/
    ├── README.md
    ├── wazuh-dashboard-config.json
    └── screenshots/
        └── (add your screenshots here)
```

---

## 🔧 Tool Stack (100% Free)

| Tool | Purpose | Cost |
|---|---|---|
| **VirtualBox / VMware Workstation Player** | Hypervisor | Free |
| **Ubuntu Server 22.04 LTS** | Base OS for servers | Free |
| **Kali Linux** | Attack simulation | Free |
| **Metasploitable 2/3** | Intentionally vulnerable target | Free |
| **DVWA** | Vulnerable web app | Free |
| **Wazuh** | SIEM + EDR + log management | Free (open source) |
| **Suricata** | Network IDS/IPS | Free (open source) |
| **Zeek** | Network analysis framework | Free (open source) |
| **TheHive** | Security incident case management | Free (community) |
| **Cortex** | Alert enrichment + automation | Free (open source) |
| **OpenVAS / Greenbone** | Vulnerability scanner | Free (community) |
| **Velociraptor** | Digital forensics + IR | Free (open source) |
| **Dependency-Track** | SCA / SBOM analysis | Free (open source) |
| **MISP** | Threat intelligence platform | Free (open source) |
| **Sigma** | Portable detection rules | Free (open source) |

---

## 🚀 Quick Start

> Full setup guides are in `/lab-setup/`. This is the 5-minute orientation.

### Minimum Hardware Requirements

| Resource | Minimum | Recommended |
|---|---|---|
| RAM | 16 GB | 32 GB |
| CPU | 4 cores | 8 cores |
| Disk | 100 GB free | 200 GB SSD |
| OS | Windows 10/11 or Linux | Linux host preferred |

> 💡 **No powerful machine?** Use [Oracle Cloud Free Tier](https://www.oracle.com/cloud/free/) — it offers always-free VMs with enough resources to run this lab.

### Step 1 - Clone the Repo

```bash
git clone https://github.com/x-lga/live-security-operations-lab.git
cd cysa-lab
```

### Step 2 - Set Up the Network

Follow [`lab-setup/01-network-design.md`](lab-setup/01-network-design.md) to create your isolated virtual network.

### Step 3 - Deploy the SIEM

Follow [`lab-setup/02-wazuh-setup.md`](lab-setup/02-wazuh-setup.md) — this is your lab's brain.

### Step 4 - Deploy Detection Tools

Follow guides 03 and 04 for Suricata and Zeek.

### Step 5 - Deploy Targets and Attack

Follow [`lab-setup/08-attack-simulation.md`](lab-setup/08-attack-simulation.md) to spin up Metasploitable and run your first attack.

### Step 6 - Watch Alerts Fire

Open your Wazuh dashboard. Run an nmap scan from Kali. Watch the alerts populate.

---


