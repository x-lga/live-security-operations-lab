# 05 - TheHive + Cortex Setup

> **TheHive turns alerts into cases.** When Wazuh fires a hundred alerts, TheHive is where you triage them into structured incidents, assign them to analysts, track every action taken, and close with documented resolution. Cortex sits alongside it as an automation engine - it takes an IP address from an alert and automatically queries VirusTotal, AbuseIPDB, and Shodan so you don't have to.

---

## Architecture

```
Wazuh Manager ──► (integration webhook) ──► TheHive
                                              │
                                              ├── Cases (structured incidents)
                                              ├── Observables (IOCs: IPs, hashes, domains)
                                              └── Cortex (automatic enrichment)
                                                    ├── VirusTotalAnalyzer
                                                    ├── AbuseIPDB
                                                    ├── Shodan
                                                    └── MISP (threat intel lookup)
```

- **TheHive**: The case management platform. Think of it as a ticket system purpose-built for security incidents.
- **Cortex**: The automation engine. Analyzers run against observables (IOCs) and return enrichment data automatically.
- Both run on the same Ubuntu VM: **192.168.56.12**

---

## Prerequisites

- Ubuntu Server 22.04 LTS VM
- IP: 192.168.56.12
- 4 GB RAM, 2 CPU cores, 30 GB disk
- Java 11 required for TheHive/Cortex
- VirusTotal API key (free tier, 4 lookups/min)
- AbuseIPDB API key (free tier)

---

## Step 1 - Prepare the VM

```bash
sudo apt update && sudo apt upgrade -y
sudo hostnamectl set-hostname thehive-cortex

# Install Java 11 (required by TheHive and Cortex)
sudo apt install openjdk-11-jre-headless -y
java -version
# openjdk version "11.x.x"

# Install other dependencies
sudo apt install curl wget gnupg apt-transport-https python3-pip -y
```

---
