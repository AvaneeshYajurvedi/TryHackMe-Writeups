#  Neighbour

> **Platform:** TryHackMe  
> **Difficulty:** Easy  
> **Category:** Web / IDOR  
> **Date Completed:** 2026-04-18  
> **Room Link:** [https://tryhackme.com/room/neighbour](https://tryhackme.com/room/neighbour)

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

A simple web challenge built around an **IDOR (Insecure Direct Object Reference)** vulnerability. The goal is to access the admin's profile by manipulating a URL parameter after gaining initial access through credentials leaked in the page source.

**Goal:**  
Access the admin's profile page and capture the flag.

**Skills Practiced:**  
Source code review, credential harvesting, IDOR exploitation

---

##  Reconnaissance

### Network Scanning

```
nmap 10.49.186.65 -sV -v
```

**Findings:**

| Port | State | Service | Version |
|------|-------|---------|---------|
| 22   | open  | SSH     | OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 |
| 80   | open  | HTTP    | Apache httpd 2.4.53 (Debian) |

Two ports open — SSH and HTTP. The interesting one is port 80.

---

##  Enumeration

### Web — Login Page

Navigating to `http://10.49.186.65` revealed a standard login page prompting for a username and password, with a note about using the guest account via `Ctrl+U`.

Pressing `Ctrl+U` (or right-click → View Page Source) to inspect the HTML source exposed a comment left by the developer:

```html
<!-- use guest:guest credentials until registration is fixed -->
```

**Credentials found:** `guest:guest`

---

##  Exploitation

### Vulnerability

**Type:** IDOR — Insecure Direct Object Reference  
**Description:**  
The application uses a `user` GET parameter in the URL (`profile.php?user=guest`) to determine which profile to display. There is no server-side authorization check verifying that the logged-in user can only view their own profile. Changing the parameter value directly grants access to any user's profile.

### Steps

**Step 1 — Log in with guest credentials**

Used the credentials found in the page source to log in:

- Username: `guest`
- Password: `guest`

After login, the app displayed a welcome message and the URL changed to:

```
http://10.49.186.65/profile.php?user=guest
```

**Step 2 — Identify the IDOR**

The `user` parameter in the URL directly controls which profile is loaded — a classic IDOR pattern. The app even hints at it with the message:

> *"Try not to peep your neighbor's profile."*

**Step 3 — Manipulate the parameter**

Changed `user=guest` to `user=admin` in the URL:

```
http://10.49.186.65/profile.php?user=admin
```

No additional authentication was required. The server responded with the admin's profile page, revealing the flag.

---

##  Flags

| Flag | Value |
|------|-------|
| Flag | `flag{66be95c478473d91a5358f2440c7af1f}` |

---

##  Key Takeaways

- **View source first** — developers often leave sensitive info in HTML comments during testing and forget to remove them before deployment.
- **IDOR is deceptively simple** — just changing a URL parameter can be enough to access unauthorized resources if there's no server-side access control.
- **Authentication ≠ Authorization** — the app correctly authenticated the guest user, but completely failed to authorize what resources that user could access.
- Always look at URL parameters after logging in — `?user=`, `?id=`, `?account=` etc. are classic IDOR targets.

---

##  Tools Used

| Tool | Purpose |
|------|---------|
| Nmap | Port & service scanning |
| Browser DevTools / Ctrl+U | Page source review |

---

##  References

- [OWASP — IDOR](https://owasp.org/www-chapter-ghana/ctf/writeups/2022/idor/)
- [PortSwigger — IDOR](https://portswigger.net/web-security/access-control/idor)

---

*Writeup by avaneesh*
