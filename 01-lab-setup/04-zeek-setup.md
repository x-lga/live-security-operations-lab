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
