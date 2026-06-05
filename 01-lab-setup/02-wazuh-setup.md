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


