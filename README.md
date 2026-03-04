# 🛡️ SOC Home Lab: Splunk SIEM + Brute Force Detection

A hands-on cybersecurity home lab that demonstrates setting up a SIEM environment using **Splunk Enterprise**, deploying a **Splunk Universal Forwarder** on a Linux endpoint, and detecting a simulated **SSH brute-force attack** using **Hydra** from a Kali Linux attacker machine.

---

## 📋 Table of Contents

- [Lab Overview](#lab-overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Part 1: Hypervisor & VM Setup](#part-1-hypervisor--vm-setup)
- [Part 2: Network Configuration](#part-2-network-configuration)
- [Part 3: Splunk Enterprise (SIEM Server)](#part-3-splunk-enterprise-siem-server)
- [Part 4: Splunk Universal Forwarder (Ubuntu Endpoint)](#part-4-splunk-universal-forwarder-ubuntu-endpoint)
- [Part 5: Configure Receiving Port in Splunk](#part-5-configure-receiving-port-in-splunk)
- [Part 6: Connect Forwarder to Splunk Server](#part-6-connect-forwarder-to-splunk-server)
- [Part 7: Monitor Log Files](#part-7-monitor-log-files)
- [Part 8: Simulate SSH Brute-Force Attack](#part-8-simulate-ssh-brute-force-attack)
- [Part 9: Detect Attack in Splunk](#part-9-detect-attack-in-splunk)
- [Key Findings](#key-findings)
- [Troubleshooting](#troubleshooting)

---

## Lab Overview

This lab simulates a real-world SOC scenario involving:

- A **Splunk Enterprise** instance acting as a centralized SIEM
- An **Ubuntu VM** (`social-loon`) as the monitored endpoint with the Splunk Universal Forwarder installed
- A **Kali Linux** attacker machine simulating an SSH brute-force attack using Hydra
- Log ingestion and event analysis inside Splunk to detect the attack

---

## Architecture

```
┌─────────────────────┐        Forwards logs (port 9997)        ┌──────────────────────────┐
│  Ubuntu VM          │ ──────────────────────────────────────► │  Splunk Enterprise       │
│  (social-loon)      │                                          │  (SIEM - localhost:8000) │
│  Splunk UF installed│                                          └──────────────────────────┘
└─────────────────────┘
         ▲
         │  SSH Brute-Force Attack
         │
┌─────────────────────┐
│  Kali Linux         │
│  (Attacker)         │
│  Hydra + Nmap       │
└─────────────────────┘
```

**Machines:**

| Role            | OS         | Hostname    | IP             |
| --------------- | ---------- | ----------- | -------------- |
| SIEM Server     | Host / VM  | —           | 192.168.*.* |
| Victim Endpoint | Ubuntu     | social-loon | 10.120.115.175 |
| Attacker        | Kali Linux | kali        | 10.120.115.1   |

---

## Prerequisites

- Hypervisor: **KVM/QEMU** via `virt-manager` (recommended) or VirtualBox
- **Splunk Enterprise** account (free trial at [splunk.com](https://www.splunk.com))
- **Ubuntu** VM (22.04 LTS or similar)
- **Kali Linux** VM
- Basic Linux command-line knowledge

> ⚠️ **VirtualBox Note:** If you encounter the error `Kernel driver not installed (rc=-1908)`, run `/sbin/vboxconfig` as root. If EFI Secure Boot is enabled, you must sign the kernel modules (`vboxdrv`, `vboxnetflt`, etc.) before proceeding. Alternatively, switch to KVM/QEMU which avoids this issue.

---

## Part 1: Hypervisor & VM Setup

### Using virt-manager (KVM/QEMU)

1. Open **virt-manager** and click **File > Add Connection**
2. Select **Hypervisor: QEMU/KVM** (use `qemu:///system` — not the user session, as user sessions have very limited guest capabilities)
3. Check **Autoconnect** and click **Connect**

### Network Interface Configuration

For each VM, configure the **Virtual Network Interface**:

- **Network source:** `Bridge device...`
- **Device model:** `virtio`
- Verify the **Link state** is `active`

> Using **Bridge device** allows VMs to communicate on the same network segment as the host, enabling cross-VM connectivity needed for this lab.

---

## Part 2: Network Configuration

### Verify Network on Ubuntu VM

```bash
# Check network interfaces
ip a

# Check device connection status
nmcli device
```

Expected output shows interface `enp1s0` with state `connecting`.

If the IP address is not assigned, try requesting one via DHCP:

```bash
sudo dhclient -v enp1s0
```

> If `dhclient` continuously sends `DHCPDISCOVER` messages without receiving an offer, verify your bridge network settings in virt-manager and ensure the host bridge interface is correctly configured.

---

## Part 3: Splunk Enterprise (SIEM Server)

Splunk Enterprise runs on the host or a dedicated VM and serves as the central SIEM at `localhost:8000`.

Access the Splunk web interface at:

```
http://localhost:8000
```

Log in with your administrator credentials.

---

## Part 4: Splunk Universal Forwarder (Ubuntu Endpoint)

### Step 1: Check System Architecture

```bash
uname -a
uname -m
```

Confirm the output shows `x86_64` — this determines which package to download.

### Step 2: Install Prerequisites

```bash
sudo apt install dpkg
sudo apt install wget
```

### Step 3: Download Splunk Universal Forwarder

1. Go to [splunk.com](https://www.splunk.com) → **Trials & Downloads** → **Universal Forwarder**
2. Log in to your Splunk account (top-right corner)
3. Select **Linux** tab → **64-bit** → **.deb** format
4. Click **Copy wget link** to copy the download command

```bash
wget -O splunkforwarder-10.2.1-<hash>-linux-amd64.deb \
  "https://download.splunk.com/products/universalforwarder/releases/10.2.1/linux/splunkforwarder-10.2.1-<hash>-linux-amd64.deb"
```

Verify the download:

```bash
ls
# Should show: splunkforwarder-10.2.1-c892b66d163d-linux-amd64.deb
```

### Step 4: Install the Package

```bash
sudo dpkg -i splunkforwarder-10.2.1-*-linux-amd64.deb
```

### Step 5: Start the Forwarder and Accept License

```bash
sudo /opt/splunkforwarder/bin/splunk start --accept-license
```

When prompted:

- **Administrator username:** `ubuntu` (or your preferred username)
- **Password:** Set a strong password (minimum 8 printable ASCII characters)

### Step 6: Verify Forwarder Status

```bash
sudo /opt/splunkforwarder/bin/splunk status
```

Expected output:

```
splunkd is running (PID: 2412).
splunk helpers are running (PIDs: 2462).
```

---

## Part 5: Configure Receiving Port in Splunk

In the **Splunk Enterprise** web interface:

1. Click **Settings** (top menu)
2. Under **DATA**, select **Forwarding and receiving**
3. Click **Configure receiving** → **New Receiving Port**
4. Enter port **`9997`** and save

Verify the receiving port shows as **Enabled** in the list.

---

## Part 6: Connect Forwarder to Splunk Server

On the **Ubuntu VM**, point the forwarder at the Splunk server:

```bash
sudo /opt/splunkforwarder/bin/splunk add forward-server <SPLUNK_SERVER_IP>:9997
```

Enter the forwarder admin credentials when prompted.

Expected output:

```
Added forwarding to: 192.168.*.*:9997
```

---

## Part 7: Monitor Log Files

Add the log sources you want Splunk to collect from the Ubuntu endpoint:

```bash
# Monitor authentication logs
sudo /opt/splunkforwarder/bin/splunk add monitor /var/log/auth.log

# Monitor system logs
sudo /opt/splunkforwarder/bin/splunk add monitor /var/log/syslog
```

### Verify in Splunk

In the Splunk search bar, run:

```
index=*
```

You should now see events flowing in. The two monitored sources (`/var/log/auth.log` and `/var/log/syslog`) should appear, with syslog making up the majority (~94%) of events.

> Before adding the forwarder, `index=*` returns **0 events**. After configuring it, you should see **1,000+ events** within minutes.

---

## Part 8: Simulate SSH Brute-Force Attack

### On the Kali Linux Attacker Machine

#### Step 1: Create a Victim User on Ubuntu

On the Ubuntu VM, create a test user to be targeted:

```bash
sudo adduser victim
# Set password: 123456  (intentionally weak for lab purposes)
```

#### Step 2: Verify SSH Config on Ubuntu

```bash
sudo nano /etc/ssh/sshd_config
```

Check that the following are set (note: with `PasswordAuthentication no`, brute force with passwords is blocked at the SSH level — set to `yes` temporarily for this lab):

```
PasswordAuthentication yes   # Must be yes for Hydra to attempt passwords
PubkeyAuthentication yes
PermitRootLogin no
```

Verify the current effective setting:

```bash
sudo sshd -T | grep passwordauthentication
```

#### Step 3: Scan the Target from Kali

```bash
nmap 10.120.115.175
```

Expected output confirms SSH is accessible:

```
PORT   STATE SERVICE
22/tcp open  ssh
```

#### Step 4: Run the Brute-Force Attack with Hydra

```bash
hydra -l victim -P dummy_password.txt ssh://10.120.115.175
```

Hydra will attempt passwords from the wordlist. The attack succeeds when Hydra finds the correct password:

```
[22][ssh] host: 10.120.115.175   login: victim   password: 123456
1 of 1 target successfully completed, 1 valid password found
```

---

## Part 9: Detect Attack in Splunk

### Search for Brute-Force Events

In Splunk Search & Reporting, run:

```
index=* source=/var/log/auth.log "Failed password"
```

You will see **hundreds of failed authentication events** from the attacker IP `10.120.115.1` against the `victim` account in rapid succession — a clear indicator of a brute-force attack.

Example events:

```
sshd[3410]: Failed password for victim from 10.120.115.1 port 59466 ssh2
sshd[3407]: Failed password for victim from 10.120.115.1 port 59428 ssh2
sshd[3408]: Failed password for victim from 10.120.115.1 port 59448 ssh2
...
```

### Observe User Creation Events

Splunk also captured the `adduser` activity on the endpoint:

```
adduser[2993]: Adding user 'victim' to group 'users'
adduser[2993]: Adding new user 'victim' to supplemental / extra groups 'users'
passwd[3017]: password changed for victim
adduser[2993]: Copying files from '/etc/skel'
adduser[2993]: Creating home directory '/home/victim'
```

This demonstrates full **endpoint visibility** — every administrative action is logged and searchable.

---

## Key Findings

| Observation                 | Detail                                 |
| --------------------------- | -------------------------------------- |
| **Attack Type**             | SSH Brute-Force                        |
| **Tool Used**               | Hydra                                  |
| **Attacker IP**             | 10.120.115.1 (Kali Linux)              |
| **Target**                  | 10.120.115.175 (Ubuntu `social-loon`)  |
| **Target User**             | `victim`                               |
| **Cracked Password**        | `123456`                               |
| **Events Before Forwarder** | 0                                      |
| **Events After Forwarder**  | 1,283+ (growing)                       |
| **Log Sources**             | `/var/log/auth.log`, `/var/log/syslog` |
| **Detection Method**        | Failed password events in Splunk       |

---

## Troubleshooting

**DHCP not assigning an IP to the VM**

- Check bridge interface settings in virt-manager
- Try `sudo dhclient -v enp1s0` manually
- Verify host bridge (`br0` or similar) is active

**VirtualBox kernel driver error (rc=-1908)**

- Run `sudo /sbin/vboxconfig` as root
- If Secure Boot is enabled, sign the kernel modules or disable Secure Boot in BIOS

**Splunk forwarder shows no events in Splunk**

- Confirm receiving port 9997 is enabled in Splunk (Settings > Forwarding and receiving)
- Verify the `forward-server` was added successfully: `sudo /opt/splunkforwarder/bin/splunk list forward-server`
- Check firewall rules allow port 9997 between the two machines

**Hydra brute-force attempt fails with "connection refused"**

- Ensure `PasswordAuthentication yes` is set in `/etc/ssh/sshd_config` on Ubuntu
- Restart SSH service: `sudo systemctl restart sshd`
- Confirm port 22 is open via `nmap`

**Blocking the attacker (remediation)**

Once the brute-force attack is detected in Splunk, block the attacker IP using UFW or iptables on the Ubuntu VM:

```bash
# Using UFW
sudo ufw deny from 10.120.115.1

# Using iptables
sudo iptables -A INPUT -s 10.120.115.1 -j DROP
```

---

## ⚠️ Disclaimer

This lab is intended **strictly for educational purposes** in a controlled, isolated environment. Do not perform brute-force attacks or any unauthorized access attempts on systems you do not own or have explicit written permission to test.
