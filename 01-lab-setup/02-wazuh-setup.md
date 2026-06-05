# 02 - Wazuh SIEM Setup

> **Wazuh is the brain of this lab.** It collects logs from every endpoint, correlates events, fires alerts, and drives the dashboard you'll spend most of your time in. Get this right before you touch anything else.

---

## What Wazuh Does in This Lab

| Component | Function |
|---|---|
| **Wazuh Manager** | Receives agent telemetry, applies detection rules, generates alerts |
| **Wazuh Indexer** | OpenSearch-based storage and search engine for all log data |
| **Wazuh Dashboard** | Kibana-based web UI - your primary analyst workstation |
| **Wazuh Agents** | Deployed on every VM; collect logs, monitor file integrity, detect anomalies |

The full stack runs on a single Ubuntu Server VM. In production these would be separate servers, but for the lab, one box handles it all.

---

## Prerequisites

- Ubuntu Server 22.04 LTS VM running
- Static IP set to `192.168.56.10`
- At least 6 GB RAM allocated
- At least 50 GB disk
- Internet access (at least for the initial install - we'll restrict it after)

---

## Step 1 - Prepare the Ubuntu Server VM

```bash
# Update and upgrade
sudo apt update && sudo apt upgrade -y

# Set the hostname
sudo hostnamectl set-hostname wazuh-manager

# Confirm your IP
ip addr show enp0s3

# Set the timezone (adjust to yours)
sudo timedatectl set-timezone Africa/Nairobi

# Install useful utilities
sudo apt install curl wget gnupg apt-transport-https lsb-release -y
```

---

## Step 2 - Install Wazuh All-in-One (Recommended for Lab)

Wazuh provides an installation assistant that handles the full stack. This is the fastest way to get a working environment.

```bash
# Download the Wazuh installation assistant
curl -sO https://packages.wazuh.com/4.x/wazuh-install.sh

# Make it executable
chmod +x ./wazuh-install.sh

# Run the all-in-one installation
# This installs Manager + Indexer + Dashboard on one node
sudo bash ./wazuh-install.sh -a
```

This will take **10–20 minutes**. The script will:
1. Configure system prerequisites
2. Install and configure Wazuh Indexer (OpenSearch)
3. Install Wazuh Manager
4. Install Wazuh Dashboard

At the end of installation, the script outputs:

```
INFO: --- Summary ---
INFO: You can access the web interface https://<wazuh-dashboard-ip>
    User: admin
    Password: <GENERATED_PASSWORD>
```

**Save this password immediately.** You can also retrieve it later:

```bash
sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
```

---

## Step 3 - Verify the Installation

```bash
# Check all Wazuh services are running
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard

# Check Wazuh Manager logs for errors
sudo tail -100 /var/ossec/logs/ossec.log

# Check the API is responding
curl -k -X GET "https://localhost:55000/" \
  -H "Authorization: Bearer $(curl -s -u admin:admin -k -X POST 'https://localhost:55000/security/user/authenticate?raw=true')"
```

---

## Step 4 - Access the Dashboard

From your **host machine**, open a browser and navigate to:

```
https://192.168.56.10
```

Accept the self-signed certificate warning and log in with:
- Username: `admin`
- Password: (the one generated during install)

You should see the Wazuh dashboard. No agents are connected yet - that's expected.

---

## Step 5 - Wazuh Manager Configuration

The main config file is at `/var/ossec/etc/ossec.conf`. We'll tune several settings for the lab.

```bash
sudo nano /var/ossec/etc/ossec.conf
```

### Key Configuration Sections

**Global Settings:**
```xml
<global>
  <jsonout_output>yes</jsonout_output>
  <alerts_log>yes</alerts_log>
  <logall>no</logall>
  <logall_json>no</logall_json>
  <email_notification>no</email_notification>
  <smtp_server>smtp.example.wpg</smtp_server>
  <email_from>wazuh@example.com</email_from>
  <email_to>recipient@example.com</email_to>
  <email_maxperhour>12</email_maxperhour>
  <email_log_source>alerts.log</email_log_source>
  <agents_disconnection_time>10m</agents_disconnection_time>
  <agents_disconnection_alert_id>504</agents_disconnection_alert_id>
</global>
```

**Alert Thresholds** (lower the threshold to see more alerts in the lab):
```xml
<alerts>
  <log_alert_level>3</log_alert_level>
  <email_alert_level>12</email_alert_level>
</alerts>
```
