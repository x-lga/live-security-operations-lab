# 04 - Zeek Network Analysis Setup

> **Zeek and Suricata are complementary, not redundant.** Suricata fires alerts when traffic matches a bad signature. Zeek creates a *behavioral record* of every network connection - who talked to whom, for how long, how many bytes, using which protocols. When Suricata says "something bad happened," Zeek tells you *exactly what the full conversation looked like.*

---

## What Zeek Does That Suricata Doesn't

| Capability | Suricata | Zeek |
|---|---|---|
| Signature-based alerts | ✅ | ❌ (by default) |
| Full connection logs | Partial | ✅ (conn.log) |
| DNS query history | Alert only | ✅ Complete history |
| HTTP full transaction logs | Partial | ✅ Every request/response |
| File extraction | ✅ | ✅ With hashing |
| SSL/TLS certificate logging | Partial | ✅ Complete cert details |
| Protocol detection | ✅ | ✅ (more flexible) |
| Scripting / custom analytics | Limited | ✅ Full scripting language |
| Threat hunting pivot data | Limited | ✅ Ideal |

Think of it this way: Suricata is your burglar alarm; Zeek is your security camera system.

---

## Zeek Log Files Reference

Zeek produces structured tab-separated log files. These are the most important:

| Log File | Contains |
|---|---|
| `conn.log` | Every TCP/UDP/ICMP connection: IPs, ports, duration, bytes, state |
| `dns.log` | Every DNS query and response |
| `http.log` | Every HTTP request: method, URI, user-agent, status, response size |
| `ssl.log` | Every TLS session: certificate details, cipher suites, validation |
| `files.log` | Every file transferred with MD5/SHA1/SHA256 hashes |
| `weird.log` | Protocol anomalies and unexpected behaviors |
| `notice.log` | Zeek's own alert-like notices (policy violations, etc.) |
| `x509.log` | Full SSL certificate details |
| `ssh.log` | SSH connection details (direction, auth outcome, algorithms) |
| `smtp.log` | SMTP email metadata |

---

## Prerequisites

- Sensor VM (same as Suricata) - Ubuntu Server 22.04
- enp0s8 in promiscuous mode
- Suricata already installed and running

---

## Step 1 - Install Zeek

```bash
# Add the Zeek repository
echo 'deb http://download.opensuse.org/repositories/security:/zeek/xUbuntu_22.04/ /' | \
  sudo tee /etc/apt/sources.list.d/security:zeek.list

curl -fsSL https://download.opensuse.org/repositories/security:zeek/xUbuntu_22.04/Release.key | \
  gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/security_zeek.gpg > /dev/null

sudo apt update
sudo apt install zeek-lts -y

# Zeek installs to /opt/zeek — add it to your PATH
echo 'export PATH=/opt/zeek/bin:$PATH' >> ~/.bashrc
source ~/.bashrc

# Verify installation
zeek --version
# Zeek version 6.x.x
```

---

## Step 2 - Configure Zeek

Zeek's primary configuration is in `/opt/zeek/etc/`.

### 2.1 - Set the Monitoring Interface

```bash
sudo nano /opt/zeek/etc/node.cfg
```

```ini
[zeek]
type=standalone
host=localhost
interface=enp0s8
```

For a multi-core setup (better performance):
```ini
[manager]
type=manager
host=localhost

[proxy-1]
type=proxy
host=localhost

[worker-1]
type=worker
host=localhost
interface=enp0s8
lb_method=pf_ring
lb_procs=2
```

### 2.2 - Configure Network Ranges

```bash
sudo nano /opt/zeek/etc/networks.cfg
```

```
# Define your local networks
# Zeek uses this to determine "local" vs "remote" traffic
10.10.10.0/24     Attack Lab Network
192.168.56.0/24   SOC Management Network
```

### 2.3 - Configure Global Settings

```bash
sudo nano /opt/zeek/etc/zeekctl.cfg
```

Key settings to verify/adjust:
```ini
# Log rotation interval (in minutes)
LogRotationInterval = 3600

# Log expiration (in days) - keep 30 days of logs
LogExpireInterval = 30

# Storage directory for logs
LogDir = /opt/zeek/logs

# Enable JSON log format (for Wazuh integration)
LogFormat = json
```


### 2.4 - Enable JSON Output (Critical for Wazuh)

Create a local site policy:

```bash
sudo nano /opt/zeek/share/zeek/site/local.zeek
```

Add at the end:

```zeek
# Use JSON output format for all logs
@load policy/tuning/json-logs

# Load useful analysis scripts
@load protocols/ftp/detect-bruteforcing
@load protocols/dns/detect-external-names
@load protocols/http/detect-sqli
@load protocols/http/detect-webapps
@load protocols/ssl/validate-certs
@load protocols/ssl/log-hostcerts-only
@load frameworks/files/hash-all-files
@load frameworks/files/detect-MHR
@load misc/scan

# Custom log settings
redef LogAscii::use_json = T;
redef LogAscii::json_timestamps = JSON::TS_ISO8601;
redef Site::local_nets += { 10.10.10.0/24, 192.168.56.0/24 };
```

---

## Step 3 - Initialize and Start Zeek

```bash
# Initialize Zeekctl (creates necessary directories, checks config)
sudo /opt/zeek/bin/zeekctl deploy

# This does: install → start → check
# Expected output:
# installing ... done
# starting zeek ... done

# Check status
sudo /opt/zeek/bin/zeekctl status
# Should show "running" for your Zeek worker(s)

# Check logs are being written
ls -la /opt/zeek/logs/current/
# You should see: conn.log dns.log http.log ssl.log etc.
```

---

## Step 4 - Verify Zeek is Capturing Traffic

From **Kali**, run a quick ping and HTTP request:

```bash
ping -c 4 10.10.10.10
curl http://10.10.10.10/
```

On the **Sensor VM**:

```bash
# Watch connection log in real-time
tail -f /opt/zeek/logs/current/conn.log | python3 -m json.tool

# Check DNS log
tail -f /opt/zeek/logs/current/dns.log | python3 -m json.tool

# Check HTTP log
tail -f /opt/zeek/logs/current/http.log | python3 -m json.tool
```

You should see the connection from Kali (10.10.10.1) to Metasploitable (10.10.10.10) appear in `conn.log`.

---

## Step 5 - Integrate Zeek Logs with Wazuh

Add Zeek log sources to the Wazuh agent on the Sensor VM:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add these `<localfile>` blocks:

```xml
<!-- Zeek connection log -->
<localfile>
  <log_format>json</log_format>
  <location>/opt/zeek/logs/current/conn.log</location>
  <label key="@source">zeek-conn</label>
</localfile>

<!-- Zeek DNS log -->
<localfile>
  <log_format>json</log_format>
  <location>/opt/zeek/logs/current/dns.log</location>
  <label key="@source">zeek-dns</label>
</localfile>

<!-- Zeek HTTP log -->
<localfile>
  <log_format>json</log_format>
  <location>/opt/zeek/logs/current/http.log</location>
  <label key="@source">zeek-http</label>
</localfile>

<!-- Zeek notice log (Zeek's own alerts) -->
<localfile>
  <log_format>json</log_format>
  <location>/opt/zeek/logs/current/notice.log</location>
  <label key="@source">zeek-notice</label>
</localfile>

<!-- Zeek weird log (anomalies) -->
<localfile>
  <log_format>json</log_format>
  <location>/opt/zeek/logs/current/weird.log</location>
  <label key="@source">zeek-weird</label>
</localfile>
```

Restart the agent:
```bash
sudo systemctl restart wazuh-agent
```

---

## Step 6 - Useful Zeek Queries for Threat Hunting

Zeek logs are structured JSON. These one-liners are your starting points for hunting:

```bash
# Find all unique source IPs that made HTTP connections
cat /opt/zeek/logs/current/http.log | \
  python3 -c "import sys,json; [print(json.loads(l).get('id.orig_h','')) for l in sys.stdin]" | \
  sort -u

# Find HTTP requests with suspicious user agents
cat /opt/zeek/logs/current/http.log | \
  python3 -c "
import sys, json
for line in sys.stdin:
    try:
        r = json.loads(line)
        ua = r.get('user_agent', '')
        if any(x in ua.lower() for x in ['nmap', 'sqlmap', 'nikto', 'masscan', 'python-requests']):
            print(f\"{r.get('ts')} | {r.get('id.orig_h')} -> {r.get('id.resp_h')} | {r.get('method')} {r.get('host','')}{r.get('uri','')} | UA: {ua}\")
    except: pass
"

# Find DNS queries to suspicious TLDs
cat /opt/zeek/logs/current/dns.log | \
  python3 -c "
import sys, json
for line in sys.stdin:
    try:
        r = json.loads(line)
        q = r.get('query', '')
        if any(q.endswith(tld) for tld in ['.xyz', '.top', '.club', '.pw', '.cc']):
            print(f\"{r.get('ts')} | {r.get('id.orig_h')} queried: {q}\")
    except: pass
"

# Find large data transfers (potential exfiltration)
cat /opt/zeek/logs/current/conn.log | \
  python3 -c "
import sys, json
for line in sys.stdin:
    try:
        r = json.loads(line)
        # Alert if more than 1 MB sent outbound
        if int(r.get('orig_bytes', 0)) > 1048576:
            print(f\"LARGE UPLOAD: {r.get('id.orig_h')}:{r.get('id.orig_p')} -> {r.get('id.resp_h')}:{r.get('id.resp_p')} | {r.get('orig_bytes')} bytes | proto: {r.get('proto')} | duration: {r.get('duration')}\")
    except: pass
"

# Find all unique SSL certificates (for detecting unusual encryption)
cat /opt/zeek/logs/current/ssl.log | \
  python3 -c "
import sys, json
for line in sys.stdin:
    try:
        r = json.loads(line)
        print(f\"{r.get('server_name', 'N/A')} | Issuer: {r.get('issuer', 'N/A')} | Valid: {r.get('validation_status', 'N/A')}\")
    except: pass
" | sort -u
```

---

## Step 7 - Automate Zeekctl with Cron

Zeek needs to be periodically "cron-checked" to rotate logs and detect if it crashed:

```bash
# Add to crontab
sudo crontab -e
```

Add these lines:
```cron
# Zeek: check every 5 minutes, rotate logs hourly
*/5 * * * * /opt/zeek/bin/zeekctl cron
```

---

## Step 8 - Zeekctl Quick Reference

```bash
# Start Zeek
sudo zeekctl start

# Stop Zeek
sudo zeekctl stop

# Restart Zeek (picks up config changes)
sudo zeekctl restart

# Deploy config changes (reinstalls and restarts)
sudo zeekctl deploy

# Check Zeek status
sudo zeekctl status

# View Zeek worker logs in real-time
sudo zeekctl diag

# List all available log files
ls -la /opt/zeek/logs/current/

# Check historical logs (rotated by date)
ls /opt/zeek/logs/
# Rotated logs are in dated directories: 2024-01-15/
```

---

## Zeek + Suricata: How They Work Together

When investigating an incident:

1. **Suricata fires an alert** for signature `ET SCAN Nmap User-Agent Observed` from `10.10.10.1`
2. **You pivot to Zeek `conn.log`** — filter by `id.orig_h == 10.10.10.1`, get the time range
3. **Zeek shows you** every connection that IP made before/during/after the Suricata alert
4. **Zeek `dns.log`** shows what hostnames that IP was resolving
5. **Zeek `http.log`** shows every URI it was hitting
6. **Suricata confirms** a specific attack signature; **Zeek confirms** the attacker's entire session history

This combination is more powerful than either alone.

---
