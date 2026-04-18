#  Dig Dug

> **Platform:** TryHackMe  
> **Difficulty:** Easy  
> **Category:** DNS Enumeration  
> **Date Completed:** 2026-04-18 
> **Room Link:** [https://tryhackme.com/room/digdug](https://tryhackme.com/room/digdug)

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

Dig Dug is a DNS-focused challenge that teaches how to query a specific DNS server directly rather than enumerating the server itself. The target runs a DNS service on port 53, but a standard Nmap scan reveals nothing obvious. The flag is retrieved by querying the domain `givemetheflag.com` through the target's DNS server using `dig`.

**Goal:**  
Query the custom DNS server to resolve `givemetheflag.com` and retrieve the flag embedded in the DNS response.

**Skills Practiced:**  
Nmap scanning, DNS enumeration, `dig` with custom nameserver, understanding DNS query structure.

---

##  Reconnaissance

### Network Scanning

```
nmap -Pn 10.49.142.161
```

**Findings:**

| Port | Service |
|------|---------|
| 22   | SSH     |

Port 53 (DNS) does not appear in the Nmap output — but the room confirms this machine **is** a DNS server. This is a key observation: the attack surface isn't the server's own services, it's what the DNS server **knows about**.

---

##  Enumeration

### Initial DNS Probing

Running standard `dig` against the target IP alone yields no useful answer:

```
dig 10.49.142.161
```

The response comes back as `NXDOMAIN` — the IP itself resolves to nothing meaningful.

### The Key Insight

Digging into DNS concepts reveals the correct approach: rather than enumerating the DNS server itself, we need to **query a specific domain through this server**. The room hints that `givemetheflag.com` is the domain of interest.

The `@` syntax in `dig` lets you direct a DNS query to a specific nameserver instead of your system's default resolver.

---

##  Exploitation

### Vulnerability

**Type:** Information Disclosure via DNS  
**CVE:** N/A  
**Description:**  
The target DNS server holds a custom record for `givemetheflag.com` that contains the flag. By pointing `dig` at the target nameserver directly, we bypass the normal DNS resolution chain and retrieve the flag from the server's zone data.

---

### Steps

**Step 1 — Query `givemetheflag.com` via the target DNS server**

```bash
dig givemetheflag.com @10.49.142.161
```

The response contains the flag embedded in the DNS answer section. 

---

##  Flags

| Flag | Value |
|------|-------|
| Room Flag | `flag{0767ccd06e79853318f25aeb08ff83e2}` |

---

##  Key Takeaways

- **DNS servers hold data — don't just scan them, query them.** Nmap tells you a port is open; `dig` tells you what the server actually knows. These are different things.
- **The `@` flag in `dig` is powerful.** `dig <domain> @<nameserver>` lets you direct any DNS query to any resolver — essential for CTFs, internal DNS recon, and zone transfer attempts.
- **Port 53 not showing in Nmap doesn't mean DNS isn't running.** Firewall rules, scan options, or UDP-only services can hide it. Always validate with tool-specific probes (`dig`, `nslookup`, `host`).
- **DNS is a rich recon surface.** Beyond A records, explore TXT, MX, NS, and CNAME records — and always try zone transfers (`dig axfr`) against internal nameservers.

---

##  Tools Used

| Tool | Purpose |
|------|---------|
| Nmap | Initial port scan |
| `dig` | DNS query with custom nameserver |

---

##  References

- [HackTricks — DNS Enumeration](https://book.hacktricks.xyz/network-services-pentesting/pentesting-dns)
- [dig Man Page](https://linux.die.net/man/1/dig)
- [DNS Zone Transfer — OWASP](https://owasp.org/www-community/attacks/DNS_Zone_Transfer)

---

*Writeup by Avaneesh*
