# pwn.college Reverse Engineering: cIMG Challenges Writeup

**Author** Ramji R  
**Category:** Reverse Engineering  
**Platform:** pwn.college  
 
---
 
## Overview
 
All three challenges follow the same pattern. The program reads a custom binary image format called `cIMG`, validates its header and pixel data, and prints the flag only if the image contains exactly **275 non-space characters** (0x113 in hex). The goal in each challenge is to reverse the header struct and craft a valid `.cimg` file.
 
---
 
## Challenge 1: cIMG (Python)
 
### Method
 
The source code was directly readable via `cat /challenge/cimg`. The program is a Python script that reads a binary file and validates it step by step using assertions.
 
### Header Layout (20 bytes)
 
| Offset | Size | Field   | Value         |
|--------|------|---------|---------------|
| 0x00   | 4    | magic   | `cIMG`        |
| 0x04   | 4    | version | `1` (uint32)  |
| 0x08   | 4    | width   | any (uint32)  |
| 0x0C   | 8    | height  | any (uint64)  |
 
### Key Checks
 
- Magic bytes must be `cIMG`
- Version must equal `1`
- Data must be `width * height` bytes, all printable ASCII (0x20 to 0x7E)
- Exactly **275 non-space characters** required to trigger the flag
### Solve Script
 
```python
python3 -c "
import struct
 
magic   = b'cIMG'
version = struct.pack('<I', 1)   # 4 bytes
width   = struct.pack('<I', 25)  # 4 bytes
height  = struct.pack('<Q', 11)  # 8 bytes  (25 * 11 = 275)
 
data = b'A' * 275
 
with open('/tmp/flag.cimg', 'wb') as f:
    f.write(magic + version + width + height + data)
"
/challenge/cimg /tmp/flag.cimg
```
 
---
 
## Challenge 2: cIMG (C, 16-byte header)
 
### Method
 
The binary was a compiled ELF64 executable. `objdump -d` was used to disassemble it. The `main` function was read instruction by instruction to reconstruct the header struct.
 
### Key Instructions Analysed
 
```
401317: cmpb $0x63, (%rsp)       -> byte[0] == 'c'
40131d: cmpb $0x49, 0x1(%rsp)    -> byte[1] == 'I'
401324: cmpb $0x4d, 0x2(%rsp)    -> byte[2] == 'M'
40132b: cmpb $0x47, 0x3(%rsp)    -> byte[3] == 'G'
401346: cmpq $0x1, 0x4(%rsp)     -> version = uint64 at offset 4
401355: movzwl 0xc(%rsp), %r12d  -> width  = uint16 at offset 0xc
40135b: movzwl 0xe(%rsp), %edx   -> height = uint16 at offset 0xe
401402: cmp $0x113, %edx         -> non-space count must be 275
```
 
The `read_exact` call used `0x10` (16) as the size, confirming a 16-byte header.
 
### Header Layout (16 bytes)
 
| Offset | Size | Field   | Value         |
|--------|------|---------|---------------|
| 0x00   | 4    | magic   | `cIMG`        |
| 0x04   | 8    | version | `1` (uint64)  |
| 0x0C   | 2    | width   | any (uint16)  |
| 0x0E   | 2    | height  | any (uint16)  |
 
### Solve Script
 
```python
python3 -c "
import struct
 
magic   = b'cIMG'
version = struct.pack('<Q', 1)   # 8 bytes
width   = struct.pack('<H', 25)  # 2 bytes
height  = struct.pack('<H', 11)  # 2 bytes  (25 * 11 = 275)
 
data = b'A' * 275
 
with open('/tmp/flag.cimg', 'wb') as f:
    f.write(magic + version + width + height + data)
"
/challenge/cimg /tmp/flag.cimg
```
 
---
 
## Challenge 3: cIMG (C, 8-byte header)
 
### Method
 
Same approach as challenge 2. The objdump output was compared against challenge 2 to spot the differences quickly.
 
### Key Differences from Challenge 2
 
```
401304: mov $0x8, %edx           -> header is now only 8 bytes (was 0x10)
401347: cmpw $0x1, 0x4(%rsp)     -> version is uint16 at offset 4 (was uint64)
401356: movzbl 0x6(%rsp), %r12d  -> width  = uint8 at offset 6 (was uint16)
40135c: movzbl 0x7(%rsp), %edx   -> height = uint8 at offset 7 (was uint16)
```
 
### Header Layout (8 bytes)
 
| Offset | Size | Field   | Value        |
|--------|------|---------|--------------|
| 0x00   | 4    | magic   | `cIMG`       |
| 0x04   | 2    | version | `1` (uint16) |
| 0x06   | 1    | width   | any (uint8)  |
| 0x07   | 1    | height  | any (uint8)  |
 
### Solve Script
 
```python
python3 -c "
import struct
 
magic   = b'cIMG'
version = struct.pack('<H', 1)   # 2 bytes
width   = struct.pack('B', 25)   # 1 byte
height  = struct.pack('B', 11)   # 1 byte  (25 * 11 = 275)
 
data = b'A' * 275
 
with open('/tmp/flag.cimg', 'wb') as f:
    f.write(magic + version + width + height + data)
"
/challenge/cimg /tmp/flag.cimg
```
 
---
