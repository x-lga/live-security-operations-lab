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

### 2.3 - Configure EVE JSON Output (for Wazuh integration)

Find the `outputs` section and configure EVE JSON:

```yaml
outputs:
  - eve-log:
      enabled: yes
      filetype: regular
      filename: /var/log/suricata/eve.json
      # Rotate logs daily
      rotate-interval: day
      # What to log
      types:
        - alert:
            payload: yes           # Include packet payload in alerts
            payload-buffer-size: 4kb
            payload-printable: yes
            packet: yes
            http-body: yes
            http-body-printable: yes
            metadata: yes
            tagged-packets: yes
        - anomaly:
            enabled: yes
            types:
              decode: yes
              stream: yes
              applayer: yes
        - http:
            extended: yes
        - dns:
            version: 2
        - tls:
            extended: yes
        - files:
            force-magic: yes
        - smtp:
        - ftp
        - rdp
        - nfs
        - smb
        - tftp
        - ikev2
        - krb5
        - bittorrent-dht
        - snmp
        - rfb
        - sip
        - dhcp:
            enabled: yes
            extended: yes
        - ssh
        - tunnel
        - mqtt:
            passwords: yes
        - flow
        - netflow
        - stats:
            totals: yes
            threads: no
            deltas: no
```


### 2.4 - Set Up the Rules Directory

```yaml
default-rule-path: /var/lib/suricata/rules

rule-files:
  - suricata.rules
  - /etc/suricata/rules/local.rules
```

---

## Step 3 - Update Suricata Rules (Emerging Threats)

```bash
# Update rules using suricata-update
sudo suricata-update

# List available rule sources
sudo suricata-update list-sources

# Enable free Emerging Threats Open rules
sudo suricata-update enable-source et/open

# Enable OISF rules
sudo suricata-update enable-source oisf/trafficid

# Run the update to download and merge rules
sudo suricata-update

# Verify rules were downloaded
ls -la /var/lib/suricata/rules/
wc -l /var/lib/suricata/rules/suricata.rules
# Should show tens of thousands of rules
```

---

## Step 4 - Install Custom Lab Rules

Copy the custom rules from the repo:

```bash
sudo mkdir -p /etc/suricata/rules/
sudo cp /path/to/cysa-lab/detection-rules/suricata/local.rules \
  /etc/suricata/rules/local.rules
```

Or create them manually (see `detection-rules/suricata/local.rules` for full content). A quick preview:

```bash
sudo nano /etc/suricata/rules/local.rules
```

---
## Step 5 - Test the Configuration

```bash
# Test the configuration file for syntax errors
sudo suricata -T -c /etc/suricata/suricata.yaml

# Expected output:
# <Date> - <Info> - Running suricata under test mode
# ...
# Configuration provided was successfully loaded. Exiting.
```

---

## Step 6 - Start Suricata as a Service

```bash
# Enable Suricata to start at boot
sudo systemctl enable suricata

# Start Suricata
sudo systemctl start suricata

# Check status
sudo systemctl status suricata

# Watch the logs for startup errors
sudo tail -f /var/log/suricata/suricata.log
```

Expected healthy output:
```
... engine started
... all threads are running
```

---

## Step 7 - Verify Detection is Working

From the **Kali VM**, run an nmap scan against Metasploitable:

```bash
nmap -sV -p 1-1000 10.10.10.10
```

On the **Sensor VM**, check the EVE JSON log:

```bash
sudo tail -f /var/log/suricata/eve.json | python3 -m json.tool | grep -A5 '"event_type":"alert"'
```

You should see alert entries appearing within seconds of the scan. If you see nothing, check:

```bash
# Is traffic actually hitting the interface?
sudo tcpdump -i enp0s8 -c 10
# If nothing comes back, your promiscuous mode or VirtualBox config is wrong
```

---

## Step 8 - Integrate Suricata Alerts with Wazuh

Tell Wazuh to read Suricata's EVE JSON log. On the **Sensor VM**, add this to `/var/ossec/etc/ossec.conf`:

```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
  <label key="@source">suricata</label>
</localfile>
```

Restart the Wazuh agent:
```bash
sudo systemctl restart wazuh-agent
```

Now check the **Wazuh dashboard** under **Security Events**. Filter by `rule.groups: suricata` — you should see Suricata alerts appearing in the SIEM.

---

## Step 9 - Tune Noisy Rules (Suppress False Positives)

In a lab environment, some rules will fire constantly on benign traffic. Create a suppression list:

```bash
sudo nano /etc/suricata/threshold.conf
```

```
# Suppress rule 2010935 (ET SCAN Nmap OS Detection) for the Kali IP to reduce noise
suppress gen_id 1, sig_id 2010935, track by_src, ip 10.10.10.1

# Rate limit repeated HTTP 404 alerts — alert max 10 per minute
rate_filter gen_id 1, sig_id 2019876, new_action alert, timeout 60, count 10, seconds 60, track by_src
```

---

## Step 10 - Set Up Rule Auto-Update (Cron)

```bash
# Update rules daily at 2 AM
echo "0 2 * * * root /usr/bin/suricata-update && systemctl reload suricata" | \
  sudo tee /etc/cron.d/suricata-update
```

---

## Suricata Operational Commands

```bash
# Reload rules without restarting (sends SIGUSR2)
sudo systemctl reload suricata

# Check Suricata statistics
sudo cat /var/log/suricata/stats.log | tail -50

# Test a specific rule against a PCAP file
sudo suricata -r /path/to/capture.pcap -c /etc/suricata/suricata.yaml \
  -l /tmp/suricata-test/

# List loaded rules
sudo suricata --list-app-layer-protos

# Check which rules fired most often (useful for tuning)
grep '"event_type":"alert"' /var/log/suricata/eve.json | \
  python3 -c "
import json,sys,collections
alerts = [json.loads(l)['alert']['signature'] for l in sys.stdin if 'alert' in l]
for sig, count in collections.Counter(alerts).most_common(20):
    print(f'{count:5d}  {sig}')
"
```

---

## Understanding Suricata Alert Structure (EVE JSON)

Every alert in `eve.json` looks like this:

```json
{
  "timestamp": "2024-01-15T14:23:01.456789+0300",
  "flow_id": 1234567890,
  "event_type": "alert",
  "src_ip": "10.10.10.1",
  "src_port": 54321,
  "dest_ip": "10.10.10.10",
  "dest_port": 80,
  "proto": "TCP",
  "alert": {
    "action": "allowed",
    "gid": 1,
    "signature_id": 2019284,
    "rev": 5,
    "signature": "ET SCAN Possible Nmap User-Agent Observed",
    "category": "Web Application Attack",
    "severity": 2,
    "metadata": {
      "affected_product": ["Any"],
      "attack_target": ["Any"],
      "created_at": ["2013_01_14"],
      "deployment": ["Perimeter"],
      "former_category": ["SCAN"],
      "mitre_tactic_id": ["TA0043"],
      "mitre_tactic_name": ["Reconnaissance"],
      "mitre_technique_id": ["T1595"],
      "mitre_technique_name": ["Active Scanning"],
      "performance_impact": ["Low"],
      "signature_severity": ["Minor"],
      "updated_at": ["2023_10_26"]
    }
  },
  "http": {
    "http_port": 0,
    "url": "/",
    "http_user_agent": "Mozilla/5.0 (compatible; Nmap Scripting Engine)"
  },
  "app_proto": "http"
}
```

Key fields for triage:
- `alert.signature` — the rule that fired
- `alert.severity` — 1 (critical) to 3 (low)
- `src_ip` / `dest_ip` — who's talking to whom
- `alert.metadata.mitre_technique_id` — maps the alert to ATT&CK

---

## ✅ Suricata Setup Verification Checklist

- [ ] Suricata service running (`systemctl status suricata`)
- [ ] No errors in `/var/log/suricata/suricata.log`
- [ ] EVE JSON file being written to (`ls -la /var/log/suricata/eve.json`)
- [ ] Nmap scan from Kali generates alerts in `eve.json`
- [ ] Wazuh dashboard showing Suricata alerts under Security Events
- [ ] Custom rules loaded (`grep "local.rules" /etc/suricata/suricata.yaml`)
- [ ] Rule update cron job configured

---

*Next: [`04-zeek-setup.md`](04-zeek-setup.md) - adding network analysis with Zeek.*
