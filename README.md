Cloud Honeypot — SSH/Telnet Threat Intelligence Lab

An attack simulation using the production style honeypot set up on the AWS EC2 environment. It was designed as a self-assigned security portfolio project.


What Is This?

A Honeypot refers to an intentionally created trap server used for attracting attackers while recording everything they do but not giving them any real access. This project will use Cowrie, which is an open-source medium interaction honeypot that simulates SSH and Telnet services, installed in an AWS EC2 environment.

In case an attacker or bot connects to it
Cowrie:
- Pretends to log them in with a convincing login
- Provides them with a safe shell interface for interaction
- Stealthily records their IP, the username/password pairs that were tried, each command that was entered, and the downloading of any malware

Everything happens in a virtual environment; nothing affects an actual system.


Architecture

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


## 🔍 Real Attack Data (First Week Live)

Within minutes of going live, bots began scanning and connecting. Here's what was captured:

| Total sessions logged | 17+ (first log chunk only) |
| Unique attacker IPs | 14+ |
| Login attempts | 6 |
| Commands executed | 65 |
| Malware download attempts | Multiple (all blocked by Cowrie's sandbox) |

Notable Attack Patterns

Credential brute-forcing:
Bots tried common default credentials including `root/default`, `admin/admin`, `root/5up`, and `root/solokey` — standard IoT/router default passwords, suggesting these are botnet campaigns targeting poorly secured devices.

Malware deployment attempt (104.157.47.144):
After logging in as `admin/admin` via Telnet, this attacker ran a scripted payload attempting to:
1. Gain elevated shell access (`enable`, `shell`, `sh`, `linuxshell`)
2. Find a writable directory across the filesystem (`/tmp`, `/var`, `/dev/shm`, `/root`, etc.)
3. Download and execute a malware binary from `185.93.89.72` via wget, tftp, and ftp (all failed — Cowrie blocked the downloads)

The same exact script was run minutes later from `200.59.88.194` — confirming this is an automated botnet campaign hitting multiple targets simultaneously.

HTTP scanner on Telnet port (65.49.1.172):
An HTTP GET request was automatically issued by the web scanner against the Telnet port, showing how mass scanning applications tend to check all open ports irrespective of the protocol used.

Port scanners (no login attempt):
Some IPs accessed the server briefly before disconnecting within a few milliseconds; they are automatic scanning IPs that will try to brute-force access only if the port is open.

Security Design Decisions

| Cowrie runs as a non-root `cowrie` user | Limits blast radius if something goes wrong |
| Real SSH access moved to port 2200 | Separates management from honeypot traffic; port 2200 restricted to my IP only |
| iptables PREROUTING redirects (22→2222, 23→2223) | Attackers hit standard ports; Cowrie handles them safely |
| AWS Security Group firewall | Ports 22/23 open to Anywhere; port 2200 locked to trusted IP only |
| Separate isolated EC2 instance | Honeypot has no connection to any other personal or production systems |



Breach Journal Dashboard

This repo includes an interactive HTML dashboard (`breach_journal.html`) for visualizing Cowrie log data.

Features:
- Paste raw `cowrie.json` log output and parse it instantly
- Stats overview: sessions, unique IPs, login attempts, commands run
- Top username/password pairs (bar chart)
- Top attacker countries (via IP geolocation)
- Expandable per-session cards showing full attack transcript
- Search and filter by IP, credential, command, or protocol

How to use:
1. Download `breach_journal.html` and open it in any browser
2. SSH into your server: `ssh -i your-key.pem -p 2200 ubuntu@YOUR_IP`
3. Run: `sudo -u cowrie cat /home/cowrie/cowrie/var/log/cowrie/cowrie.json`
4. Copy the output, paste it into the dashboard, click Parse log

---

Tech Stack

| Cloud platform | AWS EC2 |
| Operating system | Ubuntu Server 24.04 LTS |
| Honeypot software | Cowrie (open-source, Python-based) |
| Network isolation | AWS Security Groups, iptables/NAT |
| Remote access | SSH key-based authentication |
| Python environment | venv, pip |
| Version control | Git/GitHub |
| Log format | Structured JSON (cowrie.json) |
| Dashboard | Vanilla HTML/CSS/JavaScript |



Repository Structure

```
cowrie-honeypot/
├── README.md               ← This file
├── breach_journal.html     ← Interactive log analysis dashboard
└── sample_logs/
    └── sample.json         ← Sanitized example log data for testing the dashboard
```


Setup Overview

Full step-by-step notes are kept in my private project notebook. This is a high-level summary.

1. Deployed an AWS EC2 instance with Ubuntu 24.04 OS (t3.micro), configured a security group with limited permissions
2. Connected to the server using key-based SSH authentication, created a non-root `cowrie` user
3. Cloned the Cowrie repository, created a Python virtual environment and installed dependencies
4. Activated Telnet module in the file `etc/cowrie.cfg`
5. Installed Cowrie CLI using `pip install -e .` command and launched the tool using `cowrie start` command
6. Verified that both fakes are up (2222/2223 ports) before starting the service
7. Moved real SSH service to port 2200 (fixed `systemd ssh.socket` issue)
8. Configured iptables redirection and opened ports 22/23 in the AWS security group


Key Learnings

- The automated bots start targeting servers once the machine is live as the internet is always being scanned
- Most of the bots employ a limited dictionary of default credentials (`admin/admin`, `root/default`) — these are directed towards IoT devices and routers, not only servers
- Botnet attack scripts remain identical to one another for several IP addresses at once — the two scripts on `104.157.47.144` and `200.59.88.194` were identical minutes apart
- Socket activation using `systemd` in Ubuntu 24.04 overrides `sshd_config` port configurations — needs to disable `ssh.socket` separately
- File permissions on Windows ACL in OneDrive folders prevent SSH key authentication — keep `.pem` files in simple directories without OneDrive


Legal & Ethical Notes

- The honeypot was set up using my own AWS resources for research/education
- None of the traffic was redirected, used or forwarded to third parties
- IP addresses of the attackers are not revealed here; only aggregated data is shown
- All download attempts were thwarted using Cowrie's sandbox; no malware was actually run/downloaded


Built by Daman — Ontario Tech University 
