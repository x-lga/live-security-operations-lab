# 03 - Suricata IDS/IPS Setup

> **Suricata is your eyes on the network.** While Wazuh watches endpoints, Suricata watches packets - every TCP handshake, every DNS lookup, every HTTP request crossing your lab network gets inspected against a rule set. This is where network-based attacks get caught.

---

## What Suricata Does in This Lab

| Capability | Description |
|---|---|
| **Signature-based detection** | Matches traffic against rules (like Snort, but multithreaded) |
| **Protocol analysis** | Deep packet inspection for HTTP, DNS, TLS, SMB, and 50+ protocols |
| **Flow tracking** | Maintains state of TCP sessions, tracks connection patterns |
| **File extraction** | Can extract files from HTTP/FTP/SMB for further analysis |
| **EVE JSON output** | Structured log output consumed by Wazuh and ELK |
| **IPS mode** | Can drop packets inline (we'll run in IDS mode for the lab) |

Suricata runs on the **Sensor VM** (192.168.56.11), listening on `enp0s8` (the promiscuous interface on `SOC-Lab-Net`).

---

