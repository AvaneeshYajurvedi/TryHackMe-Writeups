#  Ninja Skills

> **Platform:** TryHackMe  
> **Difficulty:** Easy  
> **Category:** Linux / File Enumeration  
> **Date Completed:** 2026-04-24  
> **Room Link:** [https://tryhackme.com/room/ninjaskills](https://tryhackme.com/room/ninjaskills)

---

##  Table of Contents

1. [Room Overview](#room-overview)
2. [Questions & Solutions](#questions--solutions)
3. [Answers](#answers)
4. [Key Takeaways](#key-takeaways)

---

##  Room Overview

Ninja Skills is a Linux command-line challenge focused on file enumeration using the `find` command. Given a set of 12 specific files scattered across the filesystem, each question asks you to identify the correct file based on a property — group ownership, file content, hash, line count, owner UID, or permissions. No exploitation involved — pure Linux kung-fu.

**Target Files:**
`8V2L`, `bny0`, `c4ZX`, `D8B3`, `FHl1`, `oiM0`, `PFbD`, `rmfX`, `SRSq`, `uqyw`, `v2Vb`, `X1Uy`

**Skills Practiced:**  
`find` with group/owner/permission filters, `grep` with regex, `sha1sum`, `wc -l`, `ls -ln`.

---

##  Questions & Solutions

### Question 1 — Which files are owned by the `best-group` group?

```bash
find / -group best-group 2>/dev/null
```

Filters the entire filesystem for files belonging to `best-group`. Returns two matching files.

**Answer: `D8B3 v2Vb`**

---

### Question 2 — Which file contains an IP address?

First, locate all 12 target files:

```bash
find / -type f \( \
  -name 8V2L -o -name bny0 -o -name c4ZX -o -name D8B3 -o \
  -name FHl1 -o -name oiM0 -o -name PFbD -o -name rmfX -o \
  -name SRSq -o -name uqyw -o -name v2Vb -o -name X1Uy \
\) 2>/dev/null
```

Then search their contents for an IPv4 pattern using regex:

```bash
find / -exec grep -l '[0-9][0-9][.][0-9][0-9][.][0-9][0-9][.][0-9][0-9]' {} \; 2>/dev/null
```

**Answer: `oiM0`**

---

### Question 3 — Which file has the SHA1 hash `9d54da7584015647ba052173b84d45e8007eba94`?

Run `sha1sum` across all 12 target files and compare:

```bash
find / \( \
  -name 8V2L -o -name bny0 -o -name c4ZX -o -name D8B3 -o \
  -name FHl1 -o -name oiM0 -o -name PFbD -o -name rmfX -o \
  -name SRSq -o -name uqyw -o -name v2Vb -o -name X1Uy \
\) -exec sha1sum {} \; 2>/dev/null
```

Match the output hash against `9d54da7584015647ba052173b84d45e8007eba94`.

**Answer: `c4ZX`**

---

### Question 4 — Which file contains 230 lines?

Count lines across all target files using `wc -l`:

```bash
find / \( \
  -name 8V2L -o -name bny0 -o -name c4ZX -o -name D8B3 -o \
  -name FHl1 -o -name oiM0 -o -name PFbD -o -name rmfX -o \
  -name SRSq -o -name uqyw -o -name v2Vb -o -name X1Uy \
\) -type f -exec wc -l {} \; 2>/dev/null
```

Identify which file returns 230.

**Answer: `bny0`**

---

### Question 5 — Which file's owner has a UID of 502?

Use `ls -ln` (numeric listing) via `-exec` to display the owner UID (column 3):

```bash
find / \( \
  -name 8V2L -o -name bny0 -o -name c4ZX -o -name D8B3 -o \
  -name FHl1 -o -name oiM0 -o -name PFbD -o -name rmfX -o \
  -name SRSq -o -name uqyw -o -name v2Vb -o -name X1Uy \
\) -exec ls -ln {} \; 2>/dev/null
```

Look for UID `502` in the third column of the output.

**Answer: `X1Uy`**

---

### Question 6 — Which file is executable by everyone?

Using the same `ls -ln` output from Question 5, inspect the permissions column (column 1). A file executable by everyone will have the `x` bit set in the "others" position — i.e. the permissions string ends in `x` (e.g. `-rwxr-xr-x` or `---x--x--x`).

**Answer: `8V2L`**

---

##  Answers

| Question | Answer |
|----------|--------|
| Files owned by `best-group` (alphabetical) | `D8B3 v2Vb` |
| File containing an IP address | `oiM0` |
| File with SHA1 hash `9d54da75...` | `c4ZX` |
| File containing 230 lines | `bny0` |
| File whose owner has UID 502 | `X1Uy` |
| File executable by everyone | `8V2L` |

---

##  Key Takeaways

- **`find` is the Swiss Army knife of Linux file enumeration.** With the right flags — `-group`, `-user`, `-uid`, `-perm`, `-exec` — it can answer almost any file-system query in a single command.
- **`-exec` chains are powerful.** Combining `find` with `-exec sha1sum`, `-exec wc -l`, or `-exec ls -ln` turns file discovery into instant analysis without needing intermediate scripts.
- **File permissions are critical to understand.** The `x` bit in the "others" field (`------x` / `o+x`) means any user on the system can execute that file — even without ownership. This is a privesc risk if the file runs privileged code.
- **UID 502 doesn't map to a named user here.** Using `ls -ln` instead of `ls -l` bypasses the need for `/etc/passwd` to resolve usernames — useful in restricted environments where name resolution is unavailable.
- **SHA1 is deprecated for security use** but still common for file integrity checks in CTFs and legacy systems.

---

##  Tools Used

| Tool / Command | Purpose |
|----------------|---------|
| `find` | Core enumeration — group, type, name filters |
| `grep -l` with IPv4 regex | Searching file contents for IP addresses |
| `sha1sum` via `-exec` | Hash comparison across target files |
| `wc -l` via `-exec` | Line count per file |
| `ls -ln` via `-exec` | Numeric UID display and permission inspection |

---

##  References

- [Linux `find` Man Page](https://linux.die.net/man/1/find)
- [HackTricks — Linux File Enumeration](https://book.hacktricks.xyz/linux-hardening/useful-linux-commands)
- [GTFOBins — find](https://gtfobins.github.io/gtfobins/find/)

---

*Writeup by Avaneesh*
