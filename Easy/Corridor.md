#  Corridor

> **Platform:** TryHackMe  
> **Difficulty:** Easy  
> **Category:** Web / IDOR  
> **Date Completed:** 2026-04-18 
> **Room Link:** [https://tryhackme.com/room/corridor](https://tryhackme.com/room/corridor)

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

The Corridor room presents a visual challenge — a hallway with multiple doors, each linking to a different room via a URL. The goal is to identify and exploit an **IDOR (Insecure Direct Object Reference)** vulnerability hidden in the URL path, where room identifiers are encoded as MD5 hashes.

**Goal:**  
Find the hidden flag by manipulating the MD5-hashed room identifier in the URL.

**Skills Practiced:**  
IDOR, MD5 hash recognition, hash encoding/decoding, boundary/edge case testing.

---

##  Reconnaissance

### Initial Observation

Upon launching the challenge, a corridor with multiple doors is presented:


Clicking on any door changes the URL to a path like:

```
http://10.201.48.135/8f14e45fceea167a5a36dedd4bea2543
```

The path segment is recognisable as an **MD5 hash** — 32 hex characters.

---

##  Enumeration

### Mapping the Doors

By clicking each door and decoding the MD5 hash in the URL using [CrackStation](https://crackstation.net/), the rooms map to the following values:

| Door Position | MD5 Hash (URL) | Decoded Value |
|---------------|----------------|---------------|
| 1st (leftmost) | `c4ca4238a0b923820dcc509a6f75849b` | 1 |
| 2nd | `c81e728d9d4c2f636f067f89cc14862c` | 2 |
| 3rd | `eccbc87e4b5ce2fe28308fd9f2a7baf3` | 3 |
| 4th | `a87ff679a2f3e71d9181a67b7542122c` | 4 |
| 5th | `e4da3b7fbbce2345d7772b0674a318d5` | 5 |
| 6th | `1679091c5a880faf6fb5e6087eb1b2dc` | 6 |
| 7th | `8f14e45fceea167a5a36dedd4bea2543` | 7 |
| 8th | `c9f0f895fb98ab9159f51fd0297e236d` | 8 |
| 9th | `45c48cce2e2d7fbdea1afc51c7c6ad26` | 9 |
| 10th | `d3d9446802a44259755d38e6d163e820` | 10 |
| 11th | `6512bd43d9caa6e02c990b0a82652dca` | 11 |
| 12th | `c20ad4d76fe97759aa27a0c99bff6710` | 12 |
| 13th (rightmost) | `c51ce410c124a10e0db5e4b97fc2af39` | 13 |

The doors go from **1 to 7** on the left side and **13 down to 8** on the right side — suggesting the app uses sequential integer IDs under the hood.

---

##  Exploitation

### Vulnerability

**Type:** IDOR (Insecure Direct Object Reference)  
**CVE:** N/A  
**Description:**  
The application uses MD5 hashes of sequential integer room IDs as URL path parameters. Because MD5 is a non-secret, reversible (via lookup) hash, an attacker can enumerate room IDs by simply encoding arbitrary integers and navigating to the corresponding URL.

---

### Steps

**Step 1 — Identify the hash format**

After clicking a door, the URL reveals a 32-character hex string — consistent with MD5 format. This is confirmed by decoding it on CrackStation.

```
http://10.48.155.199/8f14e45fceea167a5a36dedd4bea2543
                     └─ MD5("7") = 8f14e45fceea167a5a36dedd4bea2543
```

**Step 2 — Map all visible rooms**

Clicking each door and decoding the hashes reveals rooms `1` through `13`. The visible range is `1–13`.

**Step 3 — Test edge cases**

Testing the upper boundary: encode `14` to MD5 and navigate to that URL.

```python
import hashlib
print(hashlib.md5(b"14").hexdigest())
# a2d0c8d59b9c8a56e3d9edc7b5d4f38d
```

```
http://10.48.155.199/a2d0c8d59b9c8a56e3d9edc7b5d4f38d  →   Failed
```

Testing the lower boundary: encode `0` to MD5 and navigate to that URL.

```python
import hashlib
print(hashlib.md5(b"0").hexdigest())
# cfcd208495d565ef66e7dff9f98764da
```

```
http://10.48.155.199/cfcd208495d565ef66e7dff9f98764da  →   Success!
```

Room `0` exists but is not shown in the corridor — it's the hidden room containing the flag.

---

##  Flags

| Flag | Value |
|------|-------|
| Room Flag | `flag{2477ef02448ad9156661ac40a6b8862e}` |

---

##  Key Takeaways

- **MD5 is not a secure identifier.** Using a hash of a predictable value (like a sequential integer) provides no real protection — it's trivially reversible via rainbow tables or tools like CrackStation.
- **IDOR doesn't require bypassing authentication.** Simply guessing or computing a valid object reference can be enough to access unauthorised resources.
- **Always test boundary values.** The interesting endpoint here was `0` — below the visible range — which is a classic off-by-one / edge case miss in access control.
- When IDs are sequential integers, **brute-forcing the hash space is trivial.** A proper fix would use UUIDs or cryptographically random, non-guessable tokens — and enforce server-side authorisation on every request.

---

##  Tools Used

| Tool | Purpose |
|------|---------|
| Browser DevTools | Observing URL changes per door |
| CrackStation | Decoding MD5 hashes to plaintext |
| Python (hashlib) | Encoding edge case integers to MD5 |

---

##  References

- [OWASP — IDOR](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/05-Authorization_Testing/04-Testing_for_Insecure_Direct_Object_References)
- [CrackStation](https://crackstation.net/)
- [HackTricks — IDOR](https://book.hacktricks.xyz/pentesting-web/idor)

---

*Writeup by Avaneesh*
