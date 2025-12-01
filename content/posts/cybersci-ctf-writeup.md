---
title: "CyberSCI CTF Write-up: Water Treatment (Part 1)"
date: 2023-10-27T10:00:00Z
draft: false
categories: ["CTF", "Reverse Engineering"]
tags: ["Ghidra", "Python", "Crypto", "DOS", "Assembly"]
description: "Reverse engineering a legacy 16-bit DOS executable to bypass a custom implementation of the MD2 hash algorithm."
---

I recently tackled the **Water Treatment** reverse engineering challenge from the [CyberSCI Regionals 2025-26](https://github.com/CyberSCI/PastChallenges/tree/main/challenges/regionals-2025-26/re/water-treatment). The challenge involves a legacy 16-bit MS-DOS executable (`wt.exe`) and consists of three flags.

I managed to solve the first flag during the hackathon and the second one afterwards. This post details how I cracked the password to get the "Access Granted" message.

Here is how I went from raw assembly to a successful crack using Ghidra and Python.

<!--more-->

## 1. Initial Reconnaissance

Before diving into Ghidra, I started with some basic file analysis. I used the `file` command to check the file type and metadata:

```bash
$ file wt.exe
wt.exe: MS-DOS executable, MZ for MS-DOS
```

This confirmed it was a legacy 16-bit MS-DOS executable. To run it, I installed **DOSBox**. Upon running the executable in DOSBox, I was presented with the interface which revealed the system name `neptune2000` (Flag 1).

After this initial check, I loaded the binary into Ghidra and ran the auto-analysis (making sure to select the 16-bit x86 Real Mode processor). I started by looking for "low hanging fruit"—strings.

Opening the Defined Strings window, I immediately found a prompt:

```text
13dc:0073: "Enter password: "
```

Using Ghidra's "Show References" feature (XREF), I traced this string back to a function named `FUN_1000_01ee`. This is our main entry point.

## 2. Analyzing the Logic

The authentication logic in `FUN_1000_01ee` was relatively straightforward once the noise was filtered out. Here is the breakdown of the assembly flow:

```asm
1000:01fc MOV  AX, DAT_13dc_0073   ; Load "Enter password" string
1000:01ff CALL FUN_print_string    ; Print it
1000:0210 CALL FUN_1000_0010       ; <--- Interesting Function A
1000:0213 MOV  BX, 0x10            ; Length = 16 bytes
1000:0216 MOV  DX, 0x644           ; Load Target Address
1000:0219 LEA  AX, [BP + -0x10]    ; Load User Input Buffer
1000:021c CALL FUN_memcmp          ; <--- Interesting Function B
```

### The Comparison

The function ending at `021c` (`FUN_memcmp`) takes the user input and compares it against 16 bytes stored at `0x644`. Naturally, I checked the memory at `0x644`.

```hex
13dc:0644  9E 46 24 8D 91 FD CB EC
13dc:064c  B3 F4 BA 7C AA E3 8B 0D
```

These bytes are not ASCII characters. This confirms that the program isn't comparing our input against a plain-text password. It is hashing (or encrypting) our input first, then comparing the result.

This means `FUN_1000_0010` (called right before the comparison) is our hashing function.

## 3. Identifying the Algorithm

I dove into `FUN_1000_0010` to understand how the input was being transformed. The decompiled C code looked messy, but two specific patterns stood out.

### The S-Box

The code heavily referenced a lookup table at address `0x43a`. I inspected this memory region:

```hex
13dc:043a  29 2E 43 C9 A2 D8 7C 01 ...
```

A quick Google search for these hex values revealed they are the digits of Pi, which is the signature S-Box for the **MD2 (Message Digest 2)** algorithm.

### The Checksum Loop

Further confirming this, I found a loop in the decompilation that matched the MD2 checksum formula: $L = S[m[i] \oplus L]$.

```c
// Ghidra Decompilation
*(byte *)((*(byte *)(uVar6 + iVar4 + iStack_e) ^ bStack_a) + 0x43a);
```

*   `0x43a` is the S-Box.
*   The `^` operator is the XOR.
*   The loop structure was iterating over a 16-byte block.

**Conclusion:** The program hashes the user input using MD2 and compares it to the hardcoded hash at `0x644`.

## 4. Challenges Encountered

During the analysis, I faced a couple of interesting hurdles:

### MS-DOS Segmentation & Ghidra
Ghidra did not automatically link the data and the functions referencing that data. This is a common pain point with 16-bit MS-DOS executables due to segmented memory models. I had to manually follow segment:offset pairs to understand where data was being read from.

### Recognizing the Hash Function
I am not deeply familiar with legacy hash functions, so I didn't recognize the algorithm during the hackathon. The S-Box looked like random data at first. It wasn't until I searched for the specific byte sequence that I realized it was MD2.

## 5. The Solution

Since MD2 is a one-way hash function, I couldn't just "decrypt" the target bytes. However, MD2 is an obsolete algorithm (from 1989), and the password was likely simple. This made it a perfect candidate for a dictionary attack.

I wrote a Python script using `pycryptodome` to hash passwords from the `rockyou.txt` wordlist and compare them to our target.

### Solver Script

```python
from Crypto.Hash import MD2
import sys

# The target hash extracted from 0x644
TARGET = bytes.fromhex("9E 46 24 8D 91 FD CB EC B3 F4 BA 7C AA E3 8B 0D")

def main():
    print(f"[*] Brute-forcing MD2 target...")
    
    # Standard rockyou.txt location on Kali
    wordlist_path = "/usr/share/wordlists/rockyou.txt"
    
    try:
        with open(wordlist_path, "r", encoding="latin-1") as f:
            for count, line in enumerate(f):
                password = line.strip()
                
                # Calculate MD2
                h = MD2.new()
                h.update(password.encode())
                
                if h.digest() == TARGET:
                    print(f"\n[!!!] PASSWORD FOUND: {password}")
                    return
                    
                if count % 100000 == 0:
                    print(f"[*] Checked {count}...", end="\r")

    except FileNotFoundError:
        print("Error: rockyou.txt not found.")

if __name__ == "__main__":
    main()
```

## 6. Result

I ran the script, and within a few seconds, it hit a match:

```text
[*] Checked 2800000...
[!!!] PASSWORD FOUND: waterworkz
```

I fired up the executable in DOSBox, typed `waterworkz`, and bypassed the check!

It turns out that `waterworkz` is the password, which corresponds to the second flag. The system name (Flag 1) is `neptune2000`.

**Flag 1 (System Name):** `neptune2000`
**Flag 2 (Password):** `waterworkz`
