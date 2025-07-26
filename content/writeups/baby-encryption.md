---
title: "HTB Challenge - Baby Encryption"
date: 2025-07-26
author: "ChaosSec"
tags: ["HTB", "Crypto", "Python", "Reverse Engineering", "Challenge"]
summary: "Reverse engineered a custom modular encryption function to recover the original plaintext and extract the flag."
---

## 🧩 Challenge Overview

We are given two files:

- `msg.enc` — a file containing a hex-encoded encrypted message.
- `chall.py` — the encryption script used to generate that file.

## 🔐 Understanding the Encryption Logic

In `chall.py`, the encryption function looks like this:

```python
def encryption(msg):
    ct = []
    for char in msg:
        ct.append((123 * char + 18) % 256)
    return bytes(ct)
```

This line:

```python
(123 * char + 18) % 256
```

means that each character in the original message (represented as an integer) is **multiplied by 123, incremented by 18, and then reduced modulo 256**.

This kind of transformation is a **linear congruential encryption**, and it can be broken if we can **invert the operation**. However, since modular arithmetic does not always have clean inverses, and the modulus here is small (256), we can just **brute-force it**.

## 🔁 Reversing the Equation

We want to reverse:

```
encrypted_char = (123 * char + 18) % 256
```

To find the original `char`, one would ideally compute:

```
char = ((encrypted_char - 18) * invmod(123, 256)) % 256
```

But instead of calculating the modular inverse, we **brute-force all 1-128 ASCII values**, applying the encryption formula until the result matches the encrypted byte.

## 🔓 Decryption Script

Here is the logic we used:

```python
def encryptar(char):
    return (123 * char + 18) % 256

def msgdecrypt(msg):
    carac = []
    for byte in msg:
        for i in range(1, 129):  # limiting to printable ASCII
            if encryptar(i) == byte:
                carac.append(chr(i))
                break
    return ''.join(carac)
```

Load the file and decrypt it:

```python
with open("msg.enc") as f:
    msg = bytes.fromhex(f.read())

msgdecrypted = msgdecrypt(msg)
print(f"Decrypted message: {msgdecrypted}")
```

## 📬 Why This Works

This approach works because:

- The encryption function is **deterministic** and **one-to-one** (injective) over a small domain (printable characters).
- The modulus is small (`256`), allowing brute-force to run quickly.
- Each encrypted byte can only result from **one valid character input** in the original function, making it feasible to reverse by comparison.

## 🏁 Final Output

Running the script outputs:

```
Th3nucl34rw1ll4rr1v30nfr1d4y.HTB{l00k_47_y0u_r3v3rs1ng_3qu4710n5_c0ngr475}
```

🎯 **Flag:**
```
HTB{l00k_47_y0u_r3v3rs1ng_3qu4710n5_c0ngr475}
```

