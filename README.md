# 🔐 Red Team vs Blue Team — Attack & Defense Simulation

> **Solo lab project** where I played both the attacker and the defender simultaneously on 3 live virtual machines — generating 19,369 real HTTP requests and blocking them with a single firewall rule, while keeping a legitimate user completely unaffected.

---

## 📌 Project Summary

| Field | Detail |
|---|---|
| **Student** | Saidinesh Andekar |
| **Batch** | 13 |
| **Date** | 17 March 2026 |
| **Platform** | Oracle VirtualBox — Host-Only Isolated Network |
| **Server** | Apache HTTP Server 2.4.52 (Ubuntu) |
| **Attack Tools** | Nmap 7.98 · Dirb v2.22 · Curl 8.18.0 |
| **Defense Tools** | UFW 0.36.1 · Apache Access Logs · AWK/Sort/Uniq |
| **Attack Volume** | **19,369 HTTP requests** from attacker IP |
| **Final Result** | ✅ Attacker blocked — Legitimate user unaffected |

---

## 🖥️ Lab Architecture

```
┌─────────────────────────────────────────────┐
│         Host-Only Network 192.168.152.0/24  │
│                                             │
│  🔴 Kali Linux        🔵 Linux Lite 6.6     │
│  192.168.152.3  ───►  192.168.152.5         │
│  (Attacker)           (Apache Server)       │
│                            ▲                │
│  🟢 Linux Mint 22.3        │                │
│  192.168.152.6  ───────────┘                │
│  (Legitimate User)                          │
└─────────────────────────────────────────────┘
```

| VM | OS | IP Address | Role |
|---|---|---|---|
| 🔴 Kali Linux | Kali Rolling | `192.168.152.3` | Attacker |
| 🔵 Linux Lite 6.6 | Ubuntu 22.04 base | `192.168.152.5` | Apache Server + Defender |
| 🟢 Linux Mint 22.3 | Ubuntu base | `192.168.152.6` | Legitimate User |

---

## ⚔️ Red Team — What I Did as the Attacker

### Attack 1 — Port Scanning (Nmap)
```bash
nmap -sV 192.168.152.5
```
**Output:**
```
PORT   STATE SERVICE  VERSION
80/tcp open  http     Apache httpd 2.4.52 ((Ubuntu))
Nmap done: 1 IP address scanned in 14.02 seconds
```
**Finding:** Exact Apache version exposed → enables CVE database lookups.

---

### Attack 2 — Directory Enumeration (Dirb)
```bash
dirb http://192.168.152.5
```
**Output:**
```
GENERATED WORDS: 4612
+ http://192.168.152.5/index.html    (CODE:200|SIZE:857)
+ http://192.168.152.5/server-status  (CODE:403|SIZE:278)
DOWNLOADED: 4612 - FOUND: 2
```
**Finding:** `/server-status` exists even though access is denied — endpoint location leaked.

---

### Attack 3 — Traffic Flood (Curl)
```bash
for i in {1..200}; do curl -s http://192.168.152.5 > /dev/null; done
```
**Result:** Generated **19,369 total requests** from `192.168.152.3` vs only **5** from the legitimate user.

---

## 🛡️ Blue Team — How I Detected and Stopped It

### Step 1 — Real-Time Log Monitoring
```bash
tail -f /var/log/apache2/access.log
```
**Live output showed:**
```
192.168.152.3 - - [17/Mar/2026:19:50:56 +0530] "GET / HTTP/1.1" 200 1109 "-" "curl/8.18.0"
192.168.152.3 - - [17/Mar/2026:19:50:56 +0530] "GET / HTTP/1.1" 200 1109 "-" "curl/8.18.0"
192.168.152.3 - - [17/Mar/2026:19:50:56 +0530] "GET / HTTP/1.1" 200 1109 "-" "curl/8.18.0"
```
**Red flag:** Same IP, same timestamp, `curl` user-agent → automated, not human.

---

### Step 2 — Attacker Identification (AWK Pipeline)
```bash
awk '{print $1}' /var/log/apache2/access.log | sort | uniq -c | sort -rn
```
**Output:**
```
19369  192.168.152.3    ← ATTACKER
  200  192.168.152.5    ← Server self
    5  192.168.152.6    ← Legitimate user
```

---

### Step 3 — Block with UFW
```bash
sudo ufw deny from 192.168.152.3
sudo ufw allow 80
sudo ufw reload
```
**Final firewall rules:**
```
[ 1] Anywhere    DENY IN     192.168.152.3   ← evaluated FIRST
[ 2] 80          ALLOW IN    Anywhere
[ 3] 80 (v6)     ALLOW IN    Anywhere (v6)
```

---

## ✅ Verification Results

| # | Test | Result |
|---|---|---|
| 1 | Kali curl blocked | `curl: (28) Failed to connect after 133480 ms` ✅ |
| 2 | Kali Firefox blocked | "The connection has timed out" ✅ |
| 3 | Linux Mint curl works | Full HTML returned (HTTP 200) ✅ |
| 4 | Linux Mint Firefox works | Full styled page loaded ✅ |
| 5 | Attacker confirmed in logs | 19,369 requests from `192.168.152.3` ✅ |
| 6 | UFW DENY rule active | Rule [1] DENY IN `192.168.152.3` confirmed ✅ |

**6 / 6 tests passed.**

---

## 🧠 Key Learnings

1. **Attackers always leave traces** — Every request is logged with IP, timestamp, and user-agent. No invisible attack exists at the network layer.
2. **Volume anomalies are the simplest signatures** — 19,369 vs 5 requests. One AWK command identifies the attacker instantly.
3. **Exact versions enable targeted attacks** — Nmap exposed `Apache httpd 2.4.52`. Production servers must suppress version banners.
4. **A 403 still leaks endpoint existence** — `/server-status` at 403 reveals the endpoint is there even when blocked.
5. **Firewall rules can be surgical** — One UFW DENY blocked a single IP with zero impact on all other traffic.
6. **Dual-role simulation = 360° understanding** — Playing both attacker and defender reveals how attack decisions directly produce log signatures.

---

## 🌍 Real-World Application

| Industry | How These Skills Are Used |
|---|---|
| 🏦 Banking (HDFC, SBI, Axis) | SOC teams run this exact AWK log analysis 24/7 to detect fraud |
| 🛒 E-Commerce (Amazon, Flipkart) | Request-count analysis blocks scrapers stealing pricing data |
| 🏛️ Government / Defence | CERT-In mandates Red Team exercises for critical infrastructure |
| ☁️ Cloud (AWS, Azure, GCP) | UFW-style rules power VPC security groups protecting millions of servers |
| 🔐 Cybersecurity Firms | This exact lifecycle is the standard penetration test engagement flow |

---

## 🛠️ Tools & Technologies

![Kali Linux](https://img.shields.io/badge/Kali_Linux-557C94?style=flat&logo=kalilinux&logoColor=white)
![Apache](https://img.shields.io/badge/Apache-D22128?style=flat&logo=apache&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=flat&logo=linux&logoColor=black)
![VirtualBox](https://img.shields.io/badge/VirtualBox-183A61?style=flat&logo=virtualbox&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-E95420?style=flat&logo=ubuntu&logoColor=white)
![Bash](https://img.shields.io/badge/Bash-4EAA25?style=flat&logo=gnubash&logoColor=white)

| Tool | Version | Purpose |
|---|---|---|
| Nmap | 7.98 | Network port scanning and version detection |
| Dirb | 2.22 | Web directory and file enumeration |
| Curl | 8.18.0 | HTTP request generation and traffic flooding |
| Apache2 | 2.4.52 | Web server (attack target) |
| UFW | 0.36.1 | Firewall — attacker IP blocking |
| AWK + Sort + Uniq | GNU | Log analysis pipeline |
| VirtualBox | Latest | VM orchestration |

---

## 📁 Repository Structure

```
redblue-lab/
│
├── README.md                          ← You are here
├── report/
│   └── saidinesh_redblue_report.tex   ← Full LaTeX report (Overleaf)
├── screenshots/
│   ├── 01_kali_whoami_id_ipa.png
│   ├── 02_nmap_dirb_curl_flood.png
│   ├── 03_curl_blocked_output.png
│   ├── 04_firefox_timeout.png
│   ├── 05_liteterminal_tail_logs.png
│   ├── 06_lite_ipa_webroot.png
│   ├── 07_apache_status_awk.png
│   ├── 08_ufw_install_status.png
│   ├── 09_ufw_deny_rules.png
│   ├── 10_ufw_final_rules.png
│   ├── 11_mint_whoami_curl.png
│   └── 12_mint_firefox_success.png
└── LICENSE
```

---

## 🚀 How to Reproduce This Lab

### Prerequisites
- Oracle VirtualBox installed
- At least 8 GB RAM on host machine
- ~50 GB free disk space

### Step 1 — Download ISOs
```
Kali Linux  : https://www.kali.org/get-kali/
Linux Lite  : https://www.linuxliteos.com/download.php  (version 6.6)
Linux Mint  : https://linuxmint.com/download.php  (Xfce edition)
```

### Step 2 — VirtualBox Network Setup
- Create all 3 VMs with **Adapter 1 = Host-Only Adapter**
- For Linux Lite only: temporarily add **Adapter 2 = NAT** to install Apache

### Step 3 — Install Apache on Linux Lite
```bash
sudo apt-get update
sudo apt-get install apache2 -y
sudo systemctl start apache2
sudo systemctl enable apache2
```

### Step 4 — Run the Attack (from Kali)
```bash
# Identify yourself
whoami && id && ip a

# Verify target is reachable
ping -c 4 192.168.152.5

# Attack 1: Port scan
nmap -sV 192.168.152.5

# Attack 2: Directory enum
dirb http://192.168.152.5

# Attack 3: Traffic flood
for i in {1..200}; do curl -s http://192.168.152.5 > /dev/null; done
```

### Step 5 — Defend (from Linux Lite)
```bash
# Monitor live logs
tail -f /var/log/apache2/access.log

# Identify attacker
awk '{print $1}' /var/log/apache2/access.log | sort | uniq -c | sort -rn

# Block attacker
sudo ufw deny from 192.168.152.3
sudo ufw allow 80
sudo ufw reload
sudo ufw status numbered
```

### Step 6 — Verify
```bash
# From Kali (should fail)
curl http://192.168.152.5

# From Linux Mint (should succeed)
curl http://192.168.152.5
```

---

## 📄 License

This project is licensed under the MIT License — see [LICENSE](LICENSE) for details.

---

## 🤝 Connect

**Saidinesh Andekar** — Cybersecurity Enthusiast | Batch 13

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-0077B5?style=flat&logo=linkedin)](https://linkedin.com/in/YOUR-LINKEDIN)
[![GitHub](https://img.shields.io/badge/GitHub-Follow-181717?style=flat&logo=github)](https://github.com/YOUR-GITHUB)

> *"This project demonstrates a complete SOC incident response lifecycle on real virtual machines — not a simulator, not a guided exercise. Real attack, real detection, real block."*

---

⭐ **If this project helped you, please star the repository!**
