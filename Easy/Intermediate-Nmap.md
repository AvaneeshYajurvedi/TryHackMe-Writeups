#  Intermediate Nmap

> **Platform:** TryHackMe  
> **Difficulty:** Easy  
> **Category:** Network Enumeration / Nmap / SSH  
> **Date Completed:** 2026-04-20

> **Room Link:** [https://tryhackme.com/room/intermediatenmap](https://tryhackme.com/room/intermediatenmap)

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

A network enumeration challenge focused on the power of full-port Nmap scanning. The target runs multiple services on non-standard ports — including two SSH instances and a mystery service on port 31337 that leaks credentials. The credentials are then used to pivot through the SSH services and find the flag on the filesystem.

**Goal:**  
Enumerate all open ports, identify the leaked credentials, use them to SSH in, and retrieve the flag.

**Skills Practiced:**  
Full-port Nmap scanning, service fingerprinting, `netcat` probing, non-standard SSH port login, Linux filesystem exploration.

---

##  Reconnaissance

### Full Port Scan

```bash
nmap -sV -sC -p- 10.49.140.3
```

Scanning all 65535 ports instead of just the default top 1000 reveals services that a standard scan would miss entirely.

**Findings:**

| Port  | Service | Notes |
|-------|---------|-------|
| 22    | SSH     | OpenSSH — standard port |
| 2222  | SSH     | OpenSSH — non-standard port, key-auth only |
| 31337 | Unknown | Nmap output includes credentials: `ubuntu` / `Dafdas!!/str0ng` |

Port 31337 is a well-known "elite" port in hacker culture — its presence here is intentional. The Nmap service scan banner leaks login credentials directly.

---

##  Enumeration

### Probing Port 31337

Connecting to the mystery port with netcat to see what it responds with:

```bash
nc 10.49.140.3 31337
```

The service responds — confirming it is active and banner-grabbing works. The credentials (`ubuntu` / `Dafdas!!/str0ng`) are visible in the Nmap output for this port.

### Testing SSH on Port 2222

With credentials in hand, the non-standard SSH port is tried first:

```bash
ssh ubuntu@10.49.140.3 -p 2222
```

**Result:** Connection refused for password auth — port 2222 requires a **public key** to log in. Dead end for now.

### SSH on Port 22

Falling back to the standard SSH port:

```bash
ssh ubuntu@10.49.140.3 -p 22
```

**Result:** Login successful with `ubuntu` / `Dafdas!!/str0ng`. 

---

##  Exploitation

### Filesystem Exploration

Once logged in, the home directory is empty — nothing of interest under `ubuntu`'s home.

Checking for other users on the system:

```bash
ls /home
```

Another user's directory is found. Navigating to it:

```bash
ls /home/<other_user>/
```

The flag file is sitting directly in that directory — readable without any lateral movement or privilege escalation needed.

```bash
cat /home/<other_user>/flag.txt
```

---

##  Flags

| Flag | Value |
|------|-------|
| Room Flag | `flag{251f309497a18888dde5222761ea88e4}` |

---

##  Key Takeaways

- **Always scan all ports with `-p-`.** Default Nmap only scans the top 1000 ports. Here, ports 2222 and 31337 — both critical — would have been completely missed without a full scan.
- **Service banners can leak credentials.** Port 31337 handed over a username and password directly in the Nmap output. Always read banner data carefully.
- **Non-standard ports are not security.** Running SSH on port 2222 or a service on 31337 provides zero security through obscurity — a full port scan finds them instantly.
- **Check `/home` for other users.** When your own directory is empty, always explore other home directories. World-readable files in another user's home can be the direct path to the flag without any privesc at all.
- **Port 31337 (eleet/leet) is a CTF staple.** Recognise it — it almost always signals something intentional.

---

##  Tools Used

| Tool | Purpose |
|------|---------|
| Nmap (`-p-`) | Full port scan with service & script detection |
| Netcat (`nc`) | Banner grabbing on port 31337 |
| SSH | Login on port 22 with leaked credentials |

---

##  References

- [Nmap Full Port Scanning](https://nmap.org/book/man-port-specification.html)
- [HackTricks — Nmap](https://book.hacktricks.xyz/generic-methodologies-and-resources/external-recon-methodology/nmap)

---

*Writeup by Avaneesh*
