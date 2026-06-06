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

## Prerequisites

- Sensor VM running Ubuntu Server 22.04
- IP: 192.168.56.11 (management), no IP on enp0s8 (monitoring)
- enp0s8 in promiscuous mode (set in VirtualBox and with `ip link set promisc on`)
- Wazuh Agent installed on this VM (per the Wazuh setup guide)

---

## Step 1 - Install Suricata

```bash
# Add the Suricata OISF stable PPA (gives us latest stable release)
sudo add-apt-repository ppa:oisf/suricata-stable -y
sudo apt update

# Install Suricata
sudo apt install suricata -y

# Verify installation
suricata --version
# Should output: Suricata version 7.x.x RELEASE
```

---

## Step 2 - Configure Suricata

The main config is at `/etc/suricata/suricata.yaml`. It's long — we'll only touch the critical sections.

```bash
sudo nano /etc/suricata/suricata.yaml
```

### 2.1 - Set Your Network Ranges

Find the `vars` section and set your home/protected network:

```yaml
vars:
  address-groups:
    HOME_NET: "[192.168.56.0/24,10.10.10.0/24]"
    EXTERNAL_NET: "!$HOME_NET"
    HTTP_SERVERS: "$HOME_NET"
    SMTP_SERVERS: "$HOME_NET"
    SQL_SERVERS: "$HOME_NET"
    DNS_SERVERS: "$HOME_NET"
    TELNET_SERVERS: "$HOME_NET"
    AIM_SERVERS: "$EXTERNAL_NET"
    DC_SERVERS: "$HOME_NET"
    DNP3_SERVER: "$HOME_NET"
    DNP3_CLIENT: "$HOME_NET"
    MODBUS_CLIENT: "$HOME_NET"
    MODBUS_SERVER: "$HOME_NET"
    ENIP_CLIENT: "$HOME_NET"
    ENIP_SERVER: "$HOME_NET"

  port-groups:
    HTTP_PORTS: "80"
    SHELLCODE_PORTS: "!80"
    ORACLE_PORTS: 1521
    SSH_PORTS: 22
    DNP3_PORTS: 20000
    MODBUS_PORTS: 502
    FILE_DATA_PORTS: "[$HTTP_PORTS,110,143]"
    FTP_PORTS: 21
    GENEVE_PORTS: 6081
    VXLAN_PORTS: 4789
    TEREDO_PORTS: 3544
```

### 2.2 - Set the Monitoring Interface

Find the `af-packet` section (or `pcap` — af-packet is preferred on Linux):

```yaml
af-packet:
  - interface: enp0s8      # Your promiscuous monitoring interface
    cluster-id: 99
    cluster-type: cluster_flow
    defrag: yes
    use-mmap: yes
    tpacket-v3: yes
```
