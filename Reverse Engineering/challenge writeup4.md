# pwn.college Writeup: cIMG Input Restrictions 

**Author** Ramji R  
**Category:** Reverse Engineering  
**Tools:** Python, objdump  

---

## Overview

All three levels share the same core idea: craft a valid `.cimg` file that satisfies the binary's header and data constraints to trigger the `win` function and print the flag. The pixel data check is identical across all levels: every byte must be printable ASCII (`0x20` to `0x7e`). What changes between levels is the **header layout**, specifically the width and height of the image field sizes.

---

## challenge 1 - Input restrictions (Python)

### Analysis

The source was provided directly. Reading it top to bottom gives all constraints immediately.

### Constraints

| Field | Size | Value |
|---|---|---|
| Magic | 4 bytes | `cIMG` |
| Version | 2 bytes (little-endian) | `0x0001` |
| Width | 1 byte | `59` |
| Height | 1 byte | `21` |
| Data | `59 * 21 = 1239` bytes | printable ASCII only |

### Solve Script

```python
python3 << 'EOF'
import subprocess, tempfile, os

header  = b"cIMG"
header += (1).to_bytes(2, "little")
header += (59).to_bytes(1, "little")
header += (21).to_bytes(1, "little")

data = b"A" * (59 * 21)

payload = header + data

with tempfile.NamedTemporaryFile(suffix=".cimg", delete=False) as f:
    f.write(payload)
    tmppath = f.name

result = subprocess.run(["/challenge/cimg", tmppath], capture_output=True, text=True)
print(result.stdout)
print(result.stderr)
os.unlink(tmppath)
EOF
```

---

## challenge 2 - Input restrictions (C)

### Analysis

Disassembled with:

```bash
objdump -d /challenge/cimg | awk '/^[0-9a-f]* <main>:/,/^[0-9a-f]* <[^>]*>:/'
```

Key instructions in `main`:

```asm
4012ec: mov $0x9, %edx          ; read 9 bytes header
401302: cmpb $0x63, [0]         ; 'c'
401304: cmpb $0x49, [1]         ; 'I'
40130b: cmpb $0x4d, [2]         ; 'M'
401312: cmpb $0x47, [3]         ; 'G'
40132d: cmpb $0x01, [4]         ; version = 1 (1 byte)
40133b: cmpw $0x47, [5]         ; width  = 71 (2-byte word)
401348: cmpw $0x15, [7]         ; height = 21 (2-byte word)
401372: mov  $0x5d3, %edx       ; read 0x5d3 = 1491 bytes data
```

Check: `71 * 21 = 1491` confirming the width/height.

### Constraints

| Field | Size | Value |
|---|---|---|
| Magic | 4 bytes | `cIMG` |
| Version | 1 byte | `0x01` |
| Width | 2 bytes (little-endian word) | `71` |
| Height | 2 bytes (little-endian word) | `21` |
| Data | `71 * 21 = 1491` bytes | printable ASCII only |

### Solve Script

```python
python3 << 'EOF'
import subprocess, tempfile, os

header  = b"cIMG"
header += (1).to_bytes(1, "little")
header += (71).to_bytes(2, "little")
header += (21).to_bytes(2, "little")

data = b"A" * (71 * 21)

payload = header + data

with tempfile.NamedTemporaryFile(suffix=".cimg", delete=False) as f:
    f.write(payload)
    tmppath = f.name

result = subprocess.run(["/challenge/cimg", tmppath], capture_output=True, text=True)
print(result.stdout)
print(result.stderr)
os.unlink(tmppath)
EOF
```

---

## challenge 3 - Input restrictons (x86)

### Analysis

Same approach as Level 2. The header size jumped to 17 bytes:

```asm
40126c: mov $0x11, %ecx         ; read 0x11 = 17 bytes header
401302: cmpb $0x63, [0]         ; 'c'
401304: cmpb $0x49, [1]         ; 'I'
40130b: cmpb $0x4d, [2]         ; 'M'
401312: cmpb $0x47, [3]         ; 'G'
40132d: cmpb $0x01, [4]         ; version = 1 (1 byte)
40133b: cmpq $0x32,  [5]        ; width  = 50 (8-byte qword)
40134a: cmpl $0x0b,  [13]       ; height = 11 (4-byte dword)
401358: mov  $0x226, %edi       ; malloc/read 0x226 = 550 bytes data
```

Check: `50 * 11 = 550` confirming the dimensions.

### Constraints

| Field | Size | Value |
|---|---|---|
| Magic | 4 bytes | `cIMG` |
| Version | 1 byte | `0x01` |
| Width | 8 bytes (little-endian qword) | `50` |
| Height | 4 bytes (little-endian dword) | `11` |
| Data | `50 * 11 = 550` bytes | printable ASCII only |

### Solve Script

```python
python3 << 'EOF'
import subprocess, tempfile, os

header  = b"cIMG"
header += (1).to_bytes(1, "little")
header += (50).to_bytes(8, "little")
header += (11).to_bytes(4, "little")

data = b"A" * (50 * 11)

payload = header + data

with tempfile.NamedTemporaryFile(suffix=".cimg", delete=False) as f:
    f.write(payload)
    tmppath = f.name

result = subprocess.run(["/challenge/cimg", tmppath], capture_output=True, text=True)
print(result.stdout)
print(result.stderr)
os.unlink(tmppath)
EOF
```

---

| Level | Version Size | Width Size | Height Size | Data Size |
|---|---|---|---|---|
| 1 (Python) | 2 bytes | 1 byte | 1 byte | 1239 |
| 2 (C) | 1 byte | 2 bytes | 2 bytes | 1491 |
| 3 (x86) | 1 byte | 8 bytes | 4 bytes | 550 |
