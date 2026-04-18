#  Lo-Fi

> **Platform:** TryHackMe  
> **Difficulty:** Easy  
> **Category:** Web / LFI  
> **Date Completed:** 2026-04-18 
> **Room Link:** [https://tryhackme.com/room/lofi](https://tryhackme.com/room/lofi)

---

##  Table of Contents

1. [Room Overview](#room-overview)
2. [Reconnaissance](#reconnaissance)
3. [Enumeration](#enumeration)
4. [Exploitation](#exploitation)
5. [Flags](#flags)
6. [Key Takeaways](#key-takeaways)

---

##  Room Overview

The Lo-Fi room presents a visually relaxing lofi-themed website with no login forms or obvious attack surface — just serene content and smooth background beats. Beneath the calm aesthetic, the app uses a vulnerable `?page=` parameter to include PHP files, making it susceptible to **Local File Inclusion (LFI)** via directory traversal.

**Goal:**  
Exploit the LFI vulnerability to read arbitrary files on the server and retrieve the flag.

**Skills Practiced:**  
Nmap scanning, LFI, directory traversal, source code review.

---

##  Reconnaissance

### Network Scanning

```
nmap -sC -sV 10.49.187.66
```

**Findings:**

| Port | Service | Version |
|------|---------|---------|
| 22   | SSH     | OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 |
| 80   | HTTP    | Apache httpd 2.2.22 (Ubuntu) |

Two ports open — SSH and a web server. Web is the primary attack surface.

---

##  Enumeration

### Web Application

Visiting `http://10.49.187.66/` reveals a lofi-themed website. No login page, no upload forms — just ambient content organized into sections.

Inspecting the **page source** exposes how the navigation is structured:

```html
<li><a href="/?page=relax.php">Relax</a></li>
<li><a href="/?page=sleep.php">Sleep</a></li>
<li><a href="/?page=chill.php">Chill</a></li>
<li><a href="/?page=coffee.php">Coffee</a></li>
<li><a href="/?page=vibe.php">Vibe</a></li>
<li><a href="/?page=game.php">Game</a></li>
```

The `?page=` parameter is used to dynamically include PHP files. This is a textbook LFI setup — the server is likely calling something like `include($_GET['page'])` without sanitising the input.

---

##  Exploitation

### Vulnerability

**Type:** Local File Inclusion (LFI) via Directory Traversal  
**CVE:** N/A  
**Description:**  
The `?page=` GET parameter is passed directly into a PHP file inclusion function without input validation. By using `../` sequences to traverse up the directory tree, an attacker can include arbitrary files from the server filesystem — anything readable by the Apache process.

---

### Steps

**Step 1 — Confirm LFI with `/etc/passwd`**

Using classic directory traversal to escape the web root and read a known system file:

```
http://10.49.187.66/?page=../../../../etc/passwd
```

The contents of `/etc/passwd` are returned in the browser, confirming the server is vulnerable to LFI and that we can read files outside the web root.

**Step 2 — Read the flag directly**

Guessing the flag lives at the filesystem root (`/flag.txt`) — a common CTF placement:

```
http://10.49.187.66/?page=../../../../flag.txt
```

The flag is returned directly in the browser response. 

---

##  Flags

| Flag | Value |
|------|-------|
| Room Flag | `flag{e4478e0eab69bd642b8238765dcb7d18}` |

---

##  Key Takeaways

- **Never pass user input directly into file inclusion functions.** `include($_GET['page'])` with no sanitisation is an immediate LFI. Use an allowlist of permitted page names instead.
- **Directory traversal (`../`) is simple but powerful.** Enough `../` sequences will always reach the filesystem root regardless of where the web root is.
- **Source code review pays off.** The vulnerable parameter wasn't obvious from the UI — the `?page=` pattern was only visible in the page source.
- **Sensitive files at predictable paths are low-hanging fruit.** Always try `/etc/passwd` to confirm LFI, then make educated guesses on flag/config file locations (`/flag.txt`, `/var/www/html/config.php`, etc.).

---

##  Tools Used

| Tool | Purpose |
|------|---------|
| Nmap | Port & service scanning |
| Browser DevTools / View Source | Identifying the vulnerable `?page=` parameter |
| Browser URL bar | Manual LFI payload delivery |

---

##  References

- [OWASP — LFI](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/07-Input_Validation_Testing/11.1-Testing_for_Local_File_Inclusion)
- [HackTricks — LFI](https://book.hacktricks.xyz/pentesting-web/file-inclusion)
- [PayloadsAllTheThings — File Inclusion](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/File%20Inclusion)

---

*Writeup by Avaneesh*
