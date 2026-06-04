# 01 - Network Design

> **Security lab networks must be isolated.** A misconfigured virtual network that bridges to your home network and then gets hit by Metasploit payloads is a bad day. This document sets up a clean, segmented virtual network that keeps attack traffic contained.

---

## 🗺️ Network Topology Overview

```
YOUR HOST MACHINE
├── Physical NIC (your home/office network) - 192.168.1.0/24 or similar
│
├── VirtualBox Host-Only Adapter: vboxnet0
│   └── SOC-Management Network: 192.168.56.0/24
│       ├── Wazuh Manager:        192.168.56.10
│       ├── Suricata/Zeek Sensor: 192.168.56.11
│       ├── TheHive + Cortex:     192.168.56.12
│       ├── OpenVAS:              192.168.56.13
│       ├── Velociraptor:         192.168.56.14
│       ├── Kali Linux:           192.168.56.20
│       ├── Metasploitable 2:     192.168.56.30
│       └── DVWA (Ubuntu):        192.168.56.31
│
└── VirtualBox Internal Network: SOC-Lab-Net (intnet)
    └── Attack/Target Network: 10.10.10.0/24
        ├── Kali Linux:          10.10.10.1   (attacker)
        ├── Metasploitable 2:    10.10.10.10  (target)
        └── DVWA:                10.10.10.11  (target)
```

### Why Two Networks?

| Network | Type | Purpose | Internet? |
|---|---|---|---|
| **SOC-Management** (192.168.56.0/24) | Host-Only | You SSH from your host into VMs; Wazuh agents report here | Host only, no internet to VMs |
| **SOC-Lab-Net** (10.10.10.0/24) | Internal | Attack traffic between Kali and targets; Suricata monitors this | No internet at all - completely isolated |

The Suricata/Zeek sensor sits with one interface on `SOC-Lab-Net` (in promiscuous mode to capture all traffic) and one on `SOC-Management` (to forward alerts to Wazuh).

---

## 🔧 VirtualBox Network Configuration

### Step 1 - Create the Host-Only Network

```
VirtualBox → File → Host Network Manager (or Tools → Network)
```

1. Click **Create** - this makes `vboxnet0`
2. Set IPv4 Address: `192.168.56.1`
3. Set IPv4 Network Mask: `255.255.255.0`
4. **Disable** DHCP Server (we assign static IPs manually)
5. Click **Apply**

Verify it was created:
```bash
# On Linux host
ip addr show vboxnet0
# Should show 192.168.56.1/24
```

### Step 2 - The Internal Network (SOC-Lab-Net)

The internal network (`intnet`) is created automatically when you add it to a VM's network adapter. No pre-configuration needed in VirtualBox. It's a completely isolated virtual switch - only VMs assigned to it can communicate, and there's zero path to your host or the internet.

---

## 🖥️ Per-VM Network Adapter Configuration

Each VM needs specific adapter assignments. Here's the complete map:

### Wazuh Manager (192.168.56.10)

| Adapter | Type | Network | IP |
|---|---|---|---|
| Adapter 1 | Host-Only | vboxnet0 | 192.168.56.10/24 |

Wazuh only needs to be reachable from your host. It doesn't need to be on the attack network.

### Suricata/Zeek Sensor (192.168.56.11)

| Adapter | Type | Network | IP |
|---|---|---|---|
| Adapter 1 | Host-Only | vboxnet0 | 192.168.56.11/24 |
| Adapter 2 | Internal | SOC-Lab-Net | 10.10.10.50/24 (or promiscuous, no IP) |

> Adapter 2 is placed in **promiscuous mode** so Suricata/Zeek captures all traffic on `SOC-Lab-Net` without needing an IP address on that interface.

### TheHive + Cortex (192.168.56.12)

| Adapter | Type | Network | IP |
|---|---|---|---|
| Adapter 1 | Host-Only | vboxnet0 | 192.168.56.12/24 |

### OpenVAS (192.168.56.13)

| Adapter | Type | Network | IP |
|---|---|---|---|
| Adapter 1 | Host-Only | vboxnet0 | 192.168.56.13/24 |
| Adapter 2 | Internal | SOC-Lab-Net | 10.10.10.51/24 |

OpenVAS needs a leg on the attack network to scan Metasploitable and DVWA.

### Velociraptor (192.168.56.14)

| Adapter | Type | Network | IP |
|---|---|---|---|
| Adapter 1 | Host-Only | vboxnet0 | 192.168.56.14/24 |

### Kali Linux (Attacker)

| Adapter | Type | Network | IP |
|---|---|---|---|
| Adapter 1 | Host-Only | vboxnet0 | 192.168.56.20/24 |
| Adapter 2 | Internal | SOC-Lab-Net | 10.10.10.1/24 |

Kali needs both: management access (so you can SSH in or control it) and attack network access (to reach targets).


### Metasploitable 2 (Target)

| Adapter | Type | Network | IP |
|---|---|---|---|
| Adapter 1 | Host-Only | vboxnet0 | 192.168.56.30/24 |
| Adapter 2 | Internal | SOC-Lab-Net | 10.10.10.10/24 |


### DVWA Ubuntu (Target)

| Adapter | Type | Network | IP |
|---|---|---|---|
| Adapter 1 | Host-Only | vboxnet0 | 192.168.56.31/24 |
| Adapter 2 | Internal | SOC-Lab-Net | 10.10.10.11/24 |

---

## ⚙️ Configuring Static IPs Inside Each VM

All VMs run Ubuntu Server (except Kali). Use Netplan for static IP configuration.

### Ubuntu Server - Static IP via Netplan

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

**Example for Wazuh Manager (single NIC):**
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: no
      addresses:
        - 192.168.56.10/24
      gateway4: 192.168.56.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

**Example for Suricata Sensor (dual NIC):**
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: no
      addresses:
        - 192.168.56.11/24
      gateway4: 192.168.56.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
    enp0s8:
      dhcp4: no
      # No IP - promiscuous capture only
```

Apply changes:
```bash
sudo netplan apply
```

Verify:
```bash
ip addr
ping 192.168.56.1  # Should reach your host
```

### Kali Linux - Static IP

Kali uses NetworkManager. Set static IPs via the GUI or:

```bash
# Edit connection for management NIC
nmcli con mod "Wired connection 1" ipv4.addresses 192.168.56.20/24
nmcli con mod "Wired connection 1" ipv4.gateway 192.168.56.1
nmcli con mod "Wired connection 1" ipv4.dns "8.8.8.8"
nmcli con mod "Wired connection 1" ipv4.method manual
nmcli con up "Wired connection 1"

# Edit connection for attack NIC
nmcli con mod "Wired connection 2" ipv4.addresses 10.10.10.1/24
nmcli con mod "Wired connection 2" ipv4.method manual
nmcli con up "Wired connection 2"
```

---

## 🔍 Enabling Promiscuous Mode on the Sensor Interface

The Suricata/Zeek sensor needs to see **all** traffic on `SOC-Lab-Net`, not just traffic addressed to it.

### In VirtualBox:

1. With the Sensor VM **powered off**, open its settings
2. Go to **Network → Adapter 2** (the SOC-Lab-Net adapter)
3. Set **Promiscuous Mode** to: `Allow All`
4. Click OK


### Inside the VM (persistent):

```bash
# Enable promiscuous mode on the monitoring interface
sudo ip link set enp0s8 promisc on

# Make it persistent across reboots
sudo nano /etc/rc.local
# Add this line before 'exit 0':
# ip link set enp0s8 promisc on
```

Verify:
```bash
ip link show enp0s8
# Look for 'PROMISC' in the flags
```

---

## 🔥 Firewall Rules (UFW)

Apply these firewall rules on each server VM to restrict access to expected sources only.


### Wazuh Manager

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 192.168.56.0/24 to any port 22    # SSH from management net
sudo ufw allow from 192.168.56.0/24 to any port 1514  # Wazuh agent UDP
sudo ufw allow from 192.168.56.0/24 to any port 1515  # Wazuh agent enrollment
sudo ufw allow from 192.168.56.0/24 to any port 55000 # Wazuh API
sudo ufw allow from 192.168.56.0/24 to any port 443   # Wazuh dashboard
sudo ufw enable
```

### TheHive

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 192.168.56.0/24 to any port 22
sudo ufw allow from 192.168.56.0/24 to any port 9000  # TheHive web UI
sudo ufw allow from 192.168.56.0/24 to any port 9001  # Cortex web UI
sudo ufw enable
```

### OpenVAS

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 192.168.56.0/24 to any port 22
sudo ufw allow from 192.168.56.0/24 to any port 9392  # Greenbone web UI
sudo ufw enable
```

---

