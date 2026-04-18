#  CyberHeroes

> **Platform:** TryHackMe  
> **Difficulty:** Easy  
> **Category:** Web / Client-Side Auth / Source Code Review  
> **Date Completed:** 2026-04-18  
> **Room Link:** [https://tryhackme.com/room/cyberheroes](https://tryhackme.com/room/cyberheroes)

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

CyberHeroes is a beginner-friendly web challenge where authentication is handled entirely on the **client side** via JavaScript. The login credentials are hardcoded in the page source — with the password obfuscated by simply reversing the string. Reading and decoding the source is all it takes to capture the flag.

**Goal:**  
Bypass the login page by reading the client-side JavaScript authentication logic and retrieving the flag.

**Skills Practiced:**  
Nmap scanning, Gobuster directory bruteforcing, page source review, JavaScript analysis, string reversal.

---

##  Reconnaissance

### Network Scanning

```
sudo nmap -sC -sV -Pn 10.49.137.220
```

**Findings:**

| Port | Service | Version |
|------|---------|---------|
| 22   | SSH     | OpenSSH 8.2p1 |
| 80   | HTTP    | Apache httpd 2.4.48 — "CyberHeroes" |

---

##  Enumeration

### Directory Bruteforcing

```bash
gobuster dir -u http://10.49.137.220 \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -t 50
```

Nothing of particular interest found via Gobuster. The attack surface is the web application itself.

### Web Application

Visiting `http://10.49.137.220/` shows a simple welcome page. Clicking **Login** in the sidebar leads to a login form with username and password fields.

### Page Source Review

Viewing the page source (`Ctrl+U`) and searching for `flag` reveals the authentication logic hardcoded in a `<script>` block:

```javascript
function authenticate() {
  a = document.getElementById('uname')
  b = document.getElementById('pass')
  const RevereString = str => [...str].reverse().join('');
  if (a.value == "h3ck3rBoi" & b.value == RevereString("54321@terceSrepuS")) {
    var xhttp = new XMLHttpRequest();
    xhttp.onreadystatechange = function() {
      if (this.readyState == 4 && this.status == 200) {
        document.getElementById("flag").innerHTML = this.responseText;
        document.getElementById("todel").innerHTML = "";
        document.getElementById("rm").remove();
      }
    };
    xhttp.open("GET", "RandomLo0o0o0o0o0o0o0o0o0gpath12345_Flag_"+a.value+"_"+b.value+".txt", true);
    xhttp.send();
  }
  else {
    alert("Incorrect Password, try again.. you got this hacker !")
  }
}
```

**Analysis:**
- **Username:** `h3ck3rBoi` — hardcoded in plaintext
- **Password:** The script compares the input against `RevereString("54321@terceSrepuS")` — meaning the expected password is that string **reversed**

---

##  Exploitation

### Vulnerability

**Type:** Client-Side Authentication Bypass / Hardcoded Credentials  
**CVE:** N/A  
**Description:**  
The entire authentication logic runs in the browser's JavaScript. The credentials are embedded directly in the source — the password is only weakly obfuscated by reversing the string, which is trivially decoded.

---

### Steps

**Step 1 — Decode the password**

The script reverses `"54321@terceSrepuS"` before comparing. Reverse it to get the actual password:

```bash
echo "54321@terceSrepuS" | rev
```

Output:
```
SuperSecret@12345
```

**Step 2 — Log in**

Enter the credentials on the login page:

```
Username: h3ck3rBoi
Password: SuperSecret@12345
```

The flag is rendered directly on the page. 

---

##  Flags

| Flag | Value |
|------|-------|
| Room Flag | `flag{edb0be532c540b1a150c3a7e85d2466e}` |

---

##  Key Takeaways

- **Never implement authentication on the client side.** Any logic running in the browser is fully visible and controllable by the user — it provides zero security.
- **Obfuscation is not encryption.** Reversing a string is not a security measure. If the algorithm to decode it is sitting right next to the ciphertext in the same script, it offers no protection.
- **Always check the page source before reaching for heavy tools.** Gobuster found nothing here — the vulnerability was a simple `Ctrl+U` away.
- **Hardcoded credentials in any form are a critical vulnerability**, whether in plaintext, reversed, base64-encoded, or otherwise.

---

##  Tools Used

| Tool | Purpose |
|------|---------|
| Nmap | Port & service scanning |
| Gobuster | Directory bruteforcing |
| Browser DevTools / View Source | Discovering hardcoded credentials in JS |
| Bash `rev` | Decoding the reversed password string |

---

##  References

- [OWASP — Client-Side Authentication](https://owasp.org/www-community/vulnerabilities/Insufficient_Authentication)
- [OWASP — Hardcoded Credentials](https://owasp.org/www-community/vulnerabilities/Use_of_hard-coded_credentials)

---

*Writeup by Avaneesh*
