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
