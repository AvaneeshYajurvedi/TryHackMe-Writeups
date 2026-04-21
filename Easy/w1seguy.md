#  W1seGuy

> **Platform:** TryHackMe  
> **Difficulty:** Easy  
> **Category:** Cryptography / XOR / Known-Plaintext Attack  
> **Date Completed:** 2026-04-21
  
> **Room Link:** [https://tryhackme.com/room/w1seguy](https://tryhackme.com/room/w1seguy)

---

##  Table of Contents

1. [Room Overview](#room-overview)
2. [Source Code Analysis](#source-code-analysis)
3. [Exploitation](#exploitation)
4. [Automation Script](#automation-script)
5. [Flags](#flags)
6. [Key Takeaways](#key-takeaways)

---

##  Room Overview

W1seGuy is a cryptography challenge centred on XOR encryption. A Python server generates a random 5-character key, XORs it against a flag, and sends the hex-encoded ciphertext — asking you to recover the key. The vulnerability is a classic **known-plaintext attack**: the flag format `THM{...}` is known, and XOR's reversible nature means ciphertext XOR plaintext = key.

**Goal:**  
Recover the XOR key from the hex-encoded ciphertext and use it to decrypt the real flag. Submit the key to the server to receive Flag 2.

**Skills Practiced:**  
XOR cryptography, known-plaintext attack, Python scripting, netcat socket interaction, brute-force key recovery.

---

##  Source Code Analysis

The task file provides the server's Python source:

```python
import random
import socketserver
import socket, os
import string

flag = open('flag.txt','r').read().strip()

def setup(server, key):
    flag = 'THM{thisisafakeflag}'
    xored = ""

    for i in range(0, len(flag)):
        xored += chr(ord(flag[i]) ^ ord(key[i % len(key)]))

    hex_encoded = xored.encode().hex()
    return hex_encoded

def start(server):
    res = ''.join(random.choices(string.ascii_letters + string.digits, k=5))
    key = str(res)
    hex_encoded = setup(server, key)
    send_message(server, "This XOR encoded text has flag 1: " + hex_encoded + "\n")

    send_message(server, "What is the encryption key? ")
    key_answer = server.recv(4096).decode().strip()

    if key_answer == key:
        send_message(server, "Congrats! Here is flag 2: " + flag + "\n")
```

**Key observations:**

- The server generates a **random 5-character alphanumeric key**
- It XORs that key against `'THM{thisisafakeflag}'` — a **known plaintext**
- The result is hex-encoded and sent to the client as **Flag 1**
- If you submit the correct key, the server returns **Flag 2** from `flag.txt`

---

##  Exploitation

### Vulnerability — Known-Plaintext XOR Attack

XOR has a critical property:

```
ciphertext = plaintext XOR key
key        = ciphertext XOR plaintext
```

Because the plaintext is hardcoded as `'THM{thisisafakeflag}'` and sent to us, we can recover the key directly by XORing the ciphertext bytes against the known plaintext bytes.

### Step 1 — Connect to the server

```bash
nc 10.48.153.251 1337
```

The server responds with:

```
This XOR encoded text has flag 1: 1e18240f287b31051a2c0f281d352c3e640a1f3b0b3e1b4739261c101c0d382410442d3828260625
What is the encryption key?
```

### Step 2 — Recover the first 4 key characters

XOR the first 4 bytes of the ciphertext against `THM{` (the known flag prefix):

```
ciphertext[0] XOR 'T' = key[0]
ciphertext[1] XOR 'H' = key[1]
ciphertext[2] XOR 'M' = key[2]
ciphertext[3] XOR '{' = key[3]
```

This gives us the first 4 characters of the key immediately.

### Step 3 — Brute-force the 5th character

The key is 5 alphanumeric characters. With the first 4 known, only 62 possibilities remain for the last character — trivially brute-forced.

**Validation:** a candidate key is correct if decrypting the full ciphertext produces a string starting with `THM{` and ending with `}`.

---

##  Automation Script

Since connections are live (a new key is generated each time), an automated script handles the full flow — connecting, receiving the ciphertext, recovering the key, and decrypting the flag:

```python
import binascii
import string
import itertools
from concurrent.futures import ThreadPoolExecutor, as_completed
import argparse

def find_key_start(encrypted_text, known_start):
    encrypted_bytes = binascii.unhexlify(encrypted_text)
    num_iterations = min(len(encrypted_bytes), len(known_start))
    key_start = ""
    for i in range(num_iterations):
        key_char = chr(encrypted_bytes[i] ^ ord(known_start[i]))
        key_start += key_char
    return key_start

def xor_decode(hex_encoded, key):
    xored = binascii.unhexlify(hex_encoded)
    decoded = ''.join(chr(xored[i] ^ ord(key[i % len(key)])) for i in range(len(xored)))
    return decoded

def test_key(key, hex_encoded, known_start, known_end):
    decoded_flag = xor_decode(hex_encoded, key)
    if decoded_flag.startswith(known_start) and decoded_flag.endswith(known_end):
        return key, decoded_flag
    return None

def generate_and_test_keys_with_prefix(hex_encoded, known_start, known_end, prefix):
    key_length = 5
    remaining_length = key_length - len(prefix)
    valid_chars = string.ascii_letters + string.digits
    matching_flags = []

    with ThreadPoolExecutor(max_workers=8) as executor:
        futures = []
        key_iterator = (
            prefix + ''.join(key_tuple)
            for key_tuple in itertools.product(valid_chars, repeat=remaining_length)
        )
        for key in key_iterator:
            futures.append(executor.submit(test_key, key, hex_encoded, known_start, known_end))
        for future in as_completed(futures):
            result = future.result()
            if result:
                matching_flags.append(result)

    return matching_flags

def main():
    parser = argparse.ArgumentParser(description='Find key and decode XOR encrypted data.')
    parser.add_argument('-e', '--encrypted', type=str, required=True,
                        help='Hex-encoded encrypted data')
    args = parser.parse_args()

    known_start = "THM{"
    known_end = "}"

    prefix = find_key_start(args.encrypted, known_start)
    print(f"Potential Key Start: {prefix}")

    matching_flags = generate_and_test_keys_with_prefix(
        args.encrypted, known_start, known_end, prefix
    )

    if matching_flags:
        for key, flag in matching_flags:
            print(f"Derived Key:   {key}")
            print(f"Decoded Flag:  {flag}")
    else:
        print("No valid flag found.")

if __name__ == "__main__":
    main()
```

**Usage:**

```bash
python3 solve.py -e <hex_encoded_ciphertext>
```

### Example Runs

```bash
python3 solve.py -e 64113f3a2901381e2f2d752106002d446d112a3a71370072385c150b290c422d0b712c42213d3324
# Potential Key Start: 0YrA
# Derived Key:   0YrAY
# Decoded Flag:  THM{p1alntExtAtt4ckcAnr3alLyhUrty0urxOr}

```

Submit the derived key back to the server to receive Flag 2 from `flag.txt`.

---

##  Flags

| Flag | Value |
|------|-------|
| Flag 1 (XOR decoded) | `THM{p1alntExtAtt4ckcAnr3alLyhUrty0urxOr}` |
| Flag 2 (server response) | `THM{BrUt3_ForC1nG_XOR_cAn_B3_FuN_nO?}` |

---

##  Key Takeaways

- **XOR with a known plaintext is trivially broken.** `key = ciphertext XOR plaintext` — if you know any part of the plaintext, you recover the corresponding key bytes for free. The `THM{` flag format gave us 4 of 5 key characters instantly.
- **Hardcoding the plaintext in the encryption function defeats the whole purpose.** The server XORed a *fixed, known* string — not the secret flag. An attacker always has the plaintext, making the key recoverable.
- **Short keys amplify XOR weaknesses.** A 5-character key repeating over a long message means every 5th byte is encrypted with the same key character. Pattern analysis and known-plaintext attacks become trivial.
- **Known-plaintext attacks are a real-world concern.** File headers, protocol magic bytes, and predictable data formats all give attackers the leverage they need to break weak XOR schemes. This is why XOR alone is never used in modern cryptography.

---

##  Tools Used

| Tool | Purpose |
|------|---------|
| Netcat | Connecting to the server on port 1337 |
| Python (custom script) | Known-plaintext XOR key recovery + brute-force of 5th character |

---

##  References

- [Wikipedia — Known-Plaintext Attack](https://en.wikipedia.org/wiki/Known-plaintext_attack)
- [CryptoHack — XOR Properties](https://cryptohack.org/courses/intro/xor0/)
- [HackTricks — XOR Cryptography](https://book.hacktricks.xyz/crypto-and-stego/cryptography/xor-cipher)

---

*Writeup by Avaneesh*
