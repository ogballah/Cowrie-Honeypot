# 🍯 Cloud Honeypot — SSH/Telnet Threat Intelligence Lab

A production-style honeypot deployed on AWS EC2 to capture and analyze real-world unauthorized access attempts. Built as a self-directed defensive security portfolio project.

---

## 📌 What Is This?

A **honeypot** is a deliberately exposed fake server designed to attract attackers and log everything they do — without giving them access to anything real. This project uses **Cowrie**, an open-source medium-interaction honeypot, running on an **AWS EC2** instance to emulate SSH and Telnet services.

When a bot or attacker connects, Cowrie:
- Fakes a realistic login and accepts almost any username/password
- Gives them a sandboxed fake shell to interact with
- Silently logs everything: their IP, credentials tried, every command typed, and any malware download attempts

None of it touches a real system — it's all a trap.

---

## 🏗️ Architecture

```
Internet (Attackers / Bots)
        │
        ▼
AWS EC2 Instance (Ubuntu 24.04 LTS, t3.micro)
        │
        ├── Port 22  (SSH)    ──iptables PREROUTING──▶ Cowrie fake SSH  (port 2222)
        ├── Port 23  (Telnet) ──iptables PREROUTING──▶ Cowrie fake Telnet (port 2223)
        └── Port 2200 (Real SSH, restricted to my IP only — management access)

Cowrie logs everything to:
  /home/cowrie/cowrie/var/log/cowrie/cowrie.json
```

---

## 🔍 Real Attack Data (First 9 Days Live)

Within minutes of going live, bots began scanning and connecting. Here's what was captured:

| Metric | Value |
|---|---|
| Total sessions logged | 17+ (first log chunk only) |
| Unique attacker IPs | 14+ |
| Login attempts | 6 |
| Commands executed | 65 |
| Malware download attempts | Multiple (all blocked by Cowrie's sandbox) |

### Notable Attack Patterns

**Credential brute-forcing:**
Bots tried common default credentials including `root/default`, `admin/admin`, `root/5up`, and `root/solokey` — standard IoT/router default passwords, suggesting these are botnet campaigns targeting poorly secured devices.

**Malware deployment attempt (104.157.47.144):**
After logging in as `admin/admin` via Telnet, this attacker ran a scripted payload attempting to:
1. Gain elevated shell access (`enable`, `shell`, `sh`, `linuxshell`)
2. Find a writable directory across the filesystem (`/tmp`, `/var`, `/dev/shm`, `/root`, etc.)
3. Download and execute a malware binary from `185.93.89.72` via wget, tftp, and ftp (all failed — Cowrie blocked the downloads)

The same exact script was run minutes later from `200.59.88.194` — confirming this is an automated botnet campaign hitting multiple targets simultaneously.

**HTTP scanner on Telnet port (65.49.1.172):**
A web scanner accidentally sent an HTTP GET request to the Telnet port — demonstrating that mass-scanning tools often probe every open port regardless of the expected protocol.

**Port scanners (no login attempt):**
Several IPs connected briefly and disconnected within milliseconds — automated scanners just checking if the port is open, not bothering to brute-force unless it is.

---

## 🛡️ Security Design Decisions

| Decision | Reason |
|---|---|
| Cowrie runs as a non-root `cowrie` user | Limits blast radius if something goes wrong |
| Real SSH access moved to port 2200 | Separates management from honeypot traffic; port 2200 restricted to my IP only |
| iptables PREROUTING redirects (22→2222, 23→2223) | Attackers hit standard ports; Cowrie handles them safely |
| AWS Security Group firewall | Ports 22/23 open to Anywhere; port 2200 locked to trusted IP only |
| Separate isolated EC2 instance | Honeypot has no connection to any other personal or production systems |

---

## 📊 Breach Journal Dashboard

This repo includes an interactive HTML dashboard (`breach_journal.html`) for visualizing Cowrie log data.

**Features:**
- Paste raw `cowrie.json` log output and parse it instantly
- Stats overview: sessions, unique IPs, login attempts, commands run
- Top username/password pairs (bar chart)
- Top attacker countries (via IP geolocation)
- Expandable per-session cards showing full attack transcript
- Search and filter by IP, credential, command, or protocol

**How to use:**
1. Download `breach_journal.html` and open it in any browser
2. SSH into your server: `ssh -i your-key.pem -p 2200 ubuntu@YOUR_IP`
3. Run: `sudo -u cowrie cat /home/cowrie/cowrie/var/log/cowrie/cowrie.json`
4. Copy the output, paste it into the dashboard, click **Parse log**

---

## 🧰 Tech Stack

| Category | Technology |
|---|---|
| Cloud platform | AWS EC2 |
| Operating system | Ubuntu Server 24.04 LTS |
| Honeypot software | Cowrie (open-source, Python-based) |
| Network isolation | AWS Security Groups, iptables/NAT |
| Remote access | SSH key-based authentication |
| Python environment | venv, pip |
| Version control | Git/GitHub |
| Log format | Structured JSON (cowrie.json) |
| Dashboard | Vanilla HTML/CSS/JavaScript |

---

## 📁 Repository Structure

```
cowrie-honeypot/
├── README.md               ← This file
├── breach_journal.html     ← Interactive log analysis dashboard
└── sample_logs/
    └── sample.json         ← Sanitized example log data for testing the dashboard
```

---

## ⚙️ Setup Overview

> Full step-by-step notes are kept in my private project notebook. This is a high-level summary.

1. Created an AWS EC2 instance (Ubuntu 24.04, t3.micro) with a locked-down security group
2. SSH'd in using key-based authentication, created a non-privileged `cowrie` user
3. Cloned Cowrie, set up a Python virtual environment, installed dependencies
4. Enabled Telnet support in `etc/cowrie.cfg`
5. Ran `pip install -e .` to install the Cowrie CLI, started it with `cowrie start`
6. Confirmed both fake ports (2222/2223) were listening before going live
7. Moved real SSH to port 2200 (resolved a `systemd ssh.socket` conflict in the process)
8. Set up iptables redirects and opened ports 22/23 in the AWS security group

---

## 📝 Key Learnings

- Automated bot traffic begins hitting exposed servers within **minutes** of going live — the internet is constantly being scanned
- Most bots use the same small dictionary of default credentials (`admin/admin`, `root/default`) — these target IoT devices and routers, not just servers
- Botnet campaigns run the **exact same scripts** across many IPs simultaneously — `104.157.47.144` and `200.59.88.194` ran identical payloads minutes apart
- `systemd` socket activation on Ubuntu 24.04 overrides `sshd_config` port settings — requires disabling `ssh.socket` separately
- File permission ACLs on Windows (especially inside OneDrive folders) can block SSH key authentication — keeping `.pem` files in plain local directories avoids this

---

## ⚠️ Legal & Ethical Notes

- This honeypot was deployed on my own AWS infrastructure for research and educational purposes
- No attacker traffic was forwarded, redirected, or used to target any third party
- Raw attacker IP addresses are kept private; only summarized/aggregated data is published here
- Malware download attempts were blocked by Cowrie's sandbox — no actual malware was executed or stored

---

*Built by Daman — Ontario Tech University | Defensive Security / Blue Team track*
