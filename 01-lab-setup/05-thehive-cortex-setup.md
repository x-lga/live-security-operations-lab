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

## Step 2 - Install Cassandra (TheHive Database)

TheHive 5.x uses Cassandra for storage.

```bash
# Install Cassandra
wget -qO - https://downloads.apache.org/cassandra/KEYS | \
  sudo gpg --dearmor -o /usr/share/keyrings/cassandra-archive.gpg

echo "deb [signed-by=/usr/share/keyrings/cassandra-archive.gpg] \
  https://debian.cassandra.apache.org 40x main" | \
  sudo tee /etc/apt/sources.list.d/cassandra.sources.list

sudo apt update
sudo apt install cassandra -y

# Start Cassandra
sudo systemctl enable cassandra
sudo systemctl start cassandra

# Wait 30 seconds for Cassandra to initialize, then verify
sleep 30
nodetool status
# Should show "UN" (Up/Normal) for your node
```

Configure Cassandra for TheHive:

```bash
sudo nano /etc/cassandra/cassandra.yaml
```

Change these values:
```yaml
cluster_name: 'thehive'
# Leave everything else as default for the lab
```

```bash
# Remove old data and restart with new cluster name
sudo systemctl stop cassandra
sudo rm -rf /var/lib/cassandra/data/system/*
sudo systemctl start cassandra
sleep 30
nodetool status
```

---

## Step 3 - Install Elasticsearch (TheHive Index)

TheHive 5.x uses Elasticsearch for indexing.

```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | \
  sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] \
  https://artifacts.elastic.co/packages/7.x/apt stable main" | \
  sudo tee /etc/apt/sources.list.d/elastic-7.x.list

sudo apt update
sudo apt install elasticsearch -y
```

Configure Elasticsearch:
```bash
sudo nano /etc/elasticsearch/elasticsearch.yml
```

```yaml
http.host: 127.0.0.1
transport.host: 127.0.0.1
cluster.name: hive
thread_pool.search.queue_size: 100000
path.logs: "/var/log/elasticsearch"
path.data: "/var/lib/elasticsearch"
xpack.security.enabled: false
script.allowed_types: "inline,stored"
```

```bash
sudo systemctl enable elasticsearch
sudo systemctl start elasticsearch

# Verify Elasticsearch is responding
curl -s http://localhost:9200/
# Should return JSON with cluster info
```

---
