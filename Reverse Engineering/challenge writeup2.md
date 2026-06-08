# pwn.college - Reverse Engineering: cIMG Challenges Writeup

**Author:** Ramji R  
**Platform:** pwn.college  
**Dojo:** Reverse Engineering  
**Category:** Magic Numbers and Version Information  

---

## Overview

This writeup covers 6 challenges from the pwn.college reverse engineering dojo, split into two categories:

- **Endianness (Magic Number)** - 3 challenges (Python, C, x86)
- **Version Information** - 3 challenges (Python, C, x86)

The goal in each challenge is to reverse engineer a `/challenge/cimg` binary (or script) and craft a `.cimg` file with the correct header bytes to pass all checks and retrieve the flag.

---

## Category 1: Endianness / Magic Number

### Challenge 1 - Magic Number (Python)

**Concept:** The binary is a Python script. Reading the source directly reveals the magic number check.

**Approach:**

```bash
cat /challenge/cimg
```

**Relevant source:**

```python
assert int.from_bytes(header[:4], "little") == 0x366D7E28, "ERROR: Invalid magic number!"
```

The script reads 4 bytes and interprets them as a little-endian integer, comparing against `0x366D7E28`.

**Key insight:** Packing `0x366D7E28` as little-endian gives bytes `\x28\x7e\x6d\x36`.

**Solution:**

```bash
python3 -c "import sys; sys.stdout.buffer.write((0x366D7E28).to_bytes(4, 'little'))" > magic.cimg
/challenge/cimg magic.cimg
```

---

### Challenge 2 - Magic Number (C)

**Concept:** A compiled C binary. Used `objdump` to find the `cmpl` instruction that performs the magic number check.

**Approach:**

```bash
objdump -d /challenge/cimg | grep -A 5 "cmp"
```

**Relevant disassembly:**

```
4012d4:  81 7c 24 04 5b 25 4d 67   cmpl   $0x674d255b, 0x4(%rsp)
```

The binary compares a 4-byte little-endian value against `0x674d255b`.

**Key insight:** `cmpl` operates on a 32-bit value already in memory (little-endian), so we just pack `0x674d255b` as little-endian bytes.

**Solution:**

```bash
python3 -c "import sys; sys.stdout.buffer.write((0x674d255b).to_bytes(4, 'little'))" > magic.cimg
/challenge/cimg magic.cimg
```

---

### Challenge 3 - Magic Number (x86)

**Concept:** Same as Challenge 2 but a different binary instance with a different magic number.

**Approach:**

```bash
objdump -d /challenge/cimg | grep -A 5 "cmp"
```

**Relevant disassembly:**

```
4012d4:  81 7c 24 04 28 7e 6d 36   cmpl   $0x366d7e28, 0x4(%rsp)
```

Magic number is `0x366d7e28` (same value as Challenge 1, different binary).

**Solution:**

```bash
python3 -c "import sys; sys.stdout.buffer.write((0x366d7e28).to_bytes(4, 'little'))" > magic.cimg
/challenge/cimg magic.cimg
```

---

## Category 2: Version Information

### Challenge 4 - Version Information (Python)

**Concept:** Beyond the magic number, the header now includes a version field. The Python source is readable directly.

**Approach:**

```bash
cat /challenge/cimg
```

**Relevant source:**

```python
header = file.read1(12)
assert header[:4] == b"(Nmg", "ERROR: Invalid magic number!"
assert int.from_bytes(header[4:12], "little") == 116, "ERROR: Invalid version!"
```

**Structure:**

| Offset | Size    | Value  | Notes                          |
|--------|---------|--------|--------------------------------|
| 0-3    | 4 bytes | `(Nmg` | Magic number (literal bytes)   |
| 4-11   | 8 bytes | `116`  | Version as 64-bit little-endian|

**Key insight:** The magic is a raw byte string `b"(Nmg"`, not an integer. The version `116` must be packed as an 8-byte (`<Q`) little-endian integer using `struct`.

**Solution:**

```bash
python3 - << 'EOF'
import struct
with open("magic.cimg", "wb") as f:
    f.write(b"(Nmg")
    f.write(struct.pack("<Q", 116))
EOF
/challenge/cimg magic.cimg
```

---

### Challenge 5 - Version Information (C)

**Concept:** A compiled C binary checks 5 bytes one at a time using `cmpb` instructions.

**Approach:**

```bash
objdump -d /challenge/cimg | grep -A 5 "cmp"
```

**Relevant disassembly:**

```
4012dc:  cmpb  $0x43, 0x3(%rsp)   # 'C'
4012e3:  cmpb  $0x4d, 0x4(%rsp)   # 'M'
4012ea:  cmpb  $0x61, 0x5(%rsp)   # 'a'
4012f1:  cmpb  $0x67, 0x6(%rsp)   # 'g'
40130c:  cmpb  $0x64, 0x7(%rsp)   # 'd'
```

**Key insight:** Each byte is checked individually via `cmpb`. Converting hex to ASCII: `0x43 0x4d 0x61 0x67 0x64` = `CMagd`. The binary reads exactly 5 bytes (`mov $0x5, %edx` before `read_exact`).

**Solution:**

```bash
python3 - << 'EOF'
with open("magic.cimg", "wb") as f:
    f.write(b"CMagd")
EOF
/challenge/cimg magic.cimg
```

---

### Challenge 6 - Version Information (x86)

**Concept:** Same byte-by-byte `cmpb` approach as Challenge 5 but with different magic bytes.

**Approach:**

```bash
objdump -d /challenge/cimg | grep -A 5 "cmp"
```

**Relevant disassembly:**

```
4012dc:  cmpb  $0x63, 0x3(%rsp)   # 'c'
4012e3:  cmpb  $0x6e, 0x4(%rsp)   # 'n'
4012ea:  cmpb  $0x6d, 0x5(%rsp)   # 'm'
4012f1:  cmpb  $0x67, 0x6(%rsp)   # 'g'
40130c:  cmpb  $0x56, 0x7(%rsp)   # 'V'
```

Converting: `0x63 0x6e 0x6d 0x67 0x56` = `cnmgV`.

**Solution:**

```bash
python3 - << 'EOF'
with open("magic.cimg", "wb") as f:
    f.write(b"cnmgV")
EOF
/challenge/cimg magic.cimg
```

---

## Tools Used:

- `cat` - reading Python scripts directly
- `objdump -d` - disassembling C/x86 binaries
- `grep -A 5 "cmp"` - filtering disassembly for comparison instructions
- `python3` heredocs (`<< 'EOF'`) - crafting binary files without saving intermediate scripts
- `struct.pack` / `.to_bytes()` - packing integers into little-endian bytes
