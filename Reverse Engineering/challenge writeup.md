# pwn.college — cIMG Magic Number Challenges Writeup

**Author:** Ramji R  
**Category:** Reverse Engineering  
**Platform:** pwn.college  
**Challenges:** 3 (Python, C x86-64, C x86-64 variant)  

---

## Overview

These three challenges are part of the cIMG dojo on pwn.college. The goal of each level is to craft a valid `.cimg` file that passes a magic number check inside the binary, triggering a `win()` function that prints the flag. The challenges progressively move from a readable Python script to compiled C binaries, requiring increasingly lower-level analysis techniques.

---

## Challenge 1 — Python Source (cIMG Magic Number)

### Approach

The binary was a Python script with a shebang (`#!/usr/bin/exec-suid -- /usr/bin/python3 -I`), meaning the source was directly readable. No reversing tools needed — just `cat /challenge/cimg`.

### Key Code

```python
header = file.read1(4)
assert len(header) == 4, "ERROR: Failed to read header!"
assert header[:4] == b"cn~R", "ERROR: Invalid magic number!"
```

### Magic Number

```
cn~R  →  0x63 0x6e 0x7e 0x52
```

### Solution

```bash
python3 -c "import sys; sys.stdout.buffer.write(b'cn~R')" > solution.cimg
/challenge/cimg solution.cimg
```

---

## Challenge 2 — Compiled C, x86-64 (Magic Number: `(~m6`)

### Approach

The binary was a compiled ELF64. Source was not available, so static analysis was required.

**Step 1 — `strings`**: Revealed error messages (`ERROR: Invalid magic number!`, `.cimg`) but not the magic bytes themselves, indicating they were stored as integer constants rather than a string literal.

**Step 2 — `objdump -s -j .rodata`**: Confirmed no magic string in rodata.

**Step 3 — `objdump -d`**: Disassembling `main` revealed four consecutive byte comparisons immediately after the `read_exact` call:

```asm
cmp    $0x28,%al   ; '('
cmp    $0x7e,%al   ; '~'
cmp    $0x6d,%al   ; 'm'
cmp    $0x36,%al   ; '6'
```

If all four pass, execution jumps to `<win>`.

### Magic Number

```
(~m6  →  0x28 0x7e 0x6d 0x36
```

### Solution

```bash
python3 -c "import sys; sys.stdout.buffer.write(b'\x28\x7e\x6d\x36')" > solution.cimg
/challenge/cimg solution.cimg
```

---

## Challenge 3 — Compiled C, x86-64 variant (Magic Number: `cn~R`)

### Approach

Identical technique to Challenge 2. Despite the challenge being labeled "x86", the binary was still ELF64 (`endbr64`, 64-bit registers). The same `objdump -d` approach revealed four byte comparisons after `read_exact`:

```asm
cmp    $0x63,%al   ; 'c'
cmp    $0x6e,%al   ; 'n'
cmp    $0x7e,%al   ; '~'
cmp    $0x52,%al   ; 'R'
```

### Magic Number

```
cn~R  →  0x63 0x6e 0x7e 0x52
```

Note: This is the same magic number as Challenge 1 — the Python source was effectively documenting the real cIMG format spec.

### Solution

```bash
python3 -c "import sys; sys.stdout.buffer.write(b'cn~R')" > solution.cimg
/challenge/cimg solution.cimg
```

---
