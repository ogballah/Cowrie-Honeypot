Cloud Honeypot Project — My Notes

Introduction: What This Project Is

I'm building a cloud honeypot, a fake server that pretends to be a real Linux
machine with SSH (and Telnet) open. It sits on the internet and attracts bots and
attackers who are constantly scanning for weak servers to break into.

Instead of a real server, they hit Cowrie, software that fakes a realistic
login and shell, lets them "in," and quietly logs everything: their IP address,
the passwords they tried, and every command they typed. Nothing they do actually
touches a real system — it's all sandboxed.

This is a legitimate, well-known defensive / blue-team security project. The
goal is to collect real-world attack data, analyze it, and use it as a portfolio
piece for cybersecurity internship applications.

The plan, phase by phase:
1. Set up an isolated AWS server (locked down, not exposed yet)
2. Install Cowrie on it
3. Test Cowrie safely from the inside (still not exposed)
4. Deliberately open it to the internet ("go live")
5. Let it collect data, then analyze and write it up
6. Later: add a VPN for safer management access, and build a
   "breach journal" web app that turns the raw logs into a readable dashboard



Part 1: AWS Account & EC2 Instance

**What AWS/EC2 is:** AWS is Amazon's cloud platform. EC2 lets you rent a small
virtual server ("instance") instead of buying physical hardware — often free at
small sizes.

**What I set up:**
- Region: **N. Virginia (us-east-1)**
- Instance name: `honeypot-project`
- OS image (AMI): **Ubuntu Server 24.04 LTS**, 64-bit (x86)
- Instance type: **t3.micro** (free tier eligible)
- Key pair: `honeypot-key-new.pem` — RSA, .pem format (downloaded to my laptop —
  this file proves it's really me connecting, like a digital key)
- Storage: default 8 GB (left as-is)
- Security group: `honeypot-sg`
  - Started with SSH (port 22) restricted to **My IP only**, so nothing else
    could reach the server while it was being built

**Why lock it down first:** keeping the firewall closed to everyone except me
meant the project was fully built and ready, but not actually exposed to real
attackers until I deliberately opened it later ("go live" step).

**Billing safety:** set up a billing alert/budget in AWS so I get an email if
costs go above a set amount (e.g. $5/month) — free tier covers this project.

---

## Part 2: Connecting From My Laptop via SSH (PowerShell)

**What SSH is:** a secure way to remotely control a Linux server from my
laptop's terminal — like typing directly on the server itself, but from home.

**Steps I took:**
1. Opened **PowerShell** on my laptop
2. Navigated to the folder where my `.pem` key file was saved
3. Tried to lock down the key file's permissions (Windows requires this —
   SSH refuses to use a key file that's "too open" for security reasons)
4. Ran into a **permissions error loop** — Windows/OneDrive kept resetting or
   blocking the permission changes on the key file
5. **Fixed it by:**
   - Opening PowerShell **as Administrator**
   - Taking ownership of the file: `takeown /F "honeypot-key-new.pem"`
   - Granting full control: `icacls "honeypot-key-new.pem" /grant "DAMAN\daman:F"`
   - Copying the key **out of OneDrive** to a simpler path: `C:\Users\daman\`
     (OneDrive sync was interfering with file permissions)
   - Resetting permissions cleanly and granting read-only access to just my
     exact account name: `icacls.exe "honeypot-key-new.pem" /grant:r "DAMAN\daman:R"`
6. Connected successfully:
   ```
   ssh -i "honeypot-key-new.pem" ubuntu@3.14.67.126
   ```

**Key lesson learned:** on Windows, `.pem` key files stored inside OneDrive
folders can cause permission errors — safer to keep them in a plain local
folder like `C:\Users\daman\`.

**Another lesson learned:** my home IP address can change between sessions.
When that happens, SSH just times out — the fix is going back into the AWS
security group and re-selecting "My IP" on the relevant rule to refresh it.

---

## Part 3: Installing & Starting Cowrie

**What Cowrie is:** the actual honeypot software. It's the fake server itself —
the real Ubuntu server on AWS is just the house it lives in. Cowrie fakes SSH
and Telnet logins, gives attackers a realistic-looking (but fake) shell, and
logs everything they do to a JSON log file.

**Steps I took on the server:**
1. Updated the system: `sudo apt update && sudo apt upgrade -y`
2. Installed build dependencies: `git`, `python3-venv`, `python3-dev`,
   `libssl-dev`, `libffi-dev`, `build-essential`
3. Created a dedicated, low-privilege user so Cowrie never runs as an admin:
   `sudo adduser --disabled-password cowrie`
4. Switched into that account
5. Cloned Cowrie's code: `git clone http://github.com/cowrie/cowrie`
6. Created and activated a Python virtual environment:
   ```
   python3 -m venv cowrie-env
   source cowrie-env/bin/activate
   ```
7. Installed Cowrie's required packages: `pip install --upgrade -r requirements.txt`
8. Copied the config template into place (had to search for it — this version
   keeps it in a different spot than older guides expect):
   ```
   find . -name "*.cfg.dist"
   cp src/cowrie/data/etc/cowrie.cfg.dist etc/cowrie.cfg
   ```
9. Enabled Telnet in the config (off by default, but bots attack it constantly):
   ```
   sed -i '827s/enabled = false/enabled = true/' etc/cowrie.cfg
   ```
10. This version of Cowrie doesn't ship the old `bin/cowrie` launcher script —
    had to install Cowrie itself as a command first:
    ```
    pip install -e .
    ```
11. Started it: `cowrie start`
12. Confirmed both fake ports were live and listening:
    ```
    ss -tlnp | grep -E "2222|2223"
    ```
    Result: port **2222** (fake SSH) and port **2223** (fake Telnet) both showing
    LISTEN — Cowrie running and ready.

**Tested it safely from inside the server first** (before exposing anything
publicly) by connecting to Cowrie's fake SSH port from the server itself:
```
ssh -p 2222 root@localhost
```
Typed a random password, got dropped into a fake shell, ran `whoami`/`ls`, then
confirmed the whole interaction was captured in the log file:
```
sudo -u cowrie tail -n 20 /home/cowrie/cowrie/var/log/cowrie/cowrie.json
```
The JSON entry showed the fake username/password, the session length, and the
commands typed — confirming Cowrie was logging correctly.

---

## Part 4: Going Live

**The problem I ran into:** to make the honeypot reachable on the "normal"
looking ports (22 for SSH, 23 for Telnet), I needed to redirect those ports to
Cowrie's fake ports (2222/2223). But my *real* management access also used port
22 — redirecting it would have locked me out.

**The fix — move real SSH to a different port first (2200):**
1. Edited the SSH config: `sudo nano /etc/ssh/sshd_config`, changed `#Port 22`
   to `Port 2200`
2. Discovered Ubuntu was actually using a separate `ssh.socket` file that
   overrode the config and kept forcing port 22 — had to disable that directly:
   ```
   sudo systemctl disable --now ssh.socket
   sudo systemctl enable --now ssh.service
   sudo systemctl restart ssh.service
   ```
3. Added a new AWS security group rule for port 2200 (Custom TCP, source: My IP)
4. Tested the new port worked in a **second PowerShell window** before closing
   the original one, to avoid getting locked out:
   ```
   ssh -i "honeypot-key-new.pem" -p 2200 ubuntu@3.14.67.126
   ```
   Confirmed working.

**Setting up the redirects:**
```
sudo apt install -y iptables
sudo iptables -t nat -A PREROUTING -p tcp --dport 22 -j REDIRECT --to-port 2222
sudo iptables -t nat -A PREROUTING -p tcp --dport 23 -j REDIRECT --to-port 2223
```

**Opening the firewall (the actual "go live" moment):**
In the AWS security group (`honeypot-sg`):
- Port 22 → changed source from My IP to **Anywhere-IPv4 (0.0.0.0/0)**
- Port 23 → added new rule, Anywhere-IPv4
- Port 2200 → **left as My IP only** — this stays my private, locked-down
  management access

**Result:** the honeypot is now publicly reachable and actively logging real
attack attempts, while my own access to manage the server stays fully private
on port 2200.

**How I check for activity:**
```
ssh -i "honeypot-key-new.pem" -p 2200 ubuntu@3.14.67.126
sudo -u cowrie tail -n 100 /home/cowrie/cowrie/var/log/cowrie/cowrie.json
```
Or to watch it live as it happens:
```
sudo -u cowrie tail -f /home/cowrie/cowrie/var/log/cowrie/cowrie.json
```
Or just filter for login attempts specifically:
```
sudo -u cowrie grep "login attempt" /home/cowrie/cowrie/var/log/cowrie/cowrie.json
```
(If my home IP changes between check-ins, I have to refresh the port 2200
security group rule to "My IP" again first.)

---



---

## Quick Reference

| EC2 Public IP | `3.14.67.126` |
| Key file | `honeypot-key-new.pem` (stored at `C:\Users\daman\`) |
| Real SSH management port | `2200` (locked to My IP) |
| Honeypot SSH port (public-facing) | `22` → redirected to Cowrie's `2222` |
| Honeypot Telnet port (public-facing) | `23` → redirected to Cowrie's `2223` |
| Reconnect command | `ssh -i "honeypot-key-new.pem" -p 2200 ubuntu@3.14.67.126` |
| Security group | `honeypot-sg` |
| Region | N. Virginia (us-east-1) |
| Instance type | t3.micro |
| OS | Ubuntu Server 24.04 LTS |
| Start Cowrie | `cowrie start` |
| Cowrie log file | `/home/cowrie/cowrie/var/log/cowrie/cowrie.json` |
| Watch logs live | `sudo -u cowrie tail -f /home/cowrie/cowrie/var/log/cowrie/cowrie.json` |

---

