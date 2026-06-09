# pwn.college cIMG Format Reverse Engineering Writeups

**Author:** Ramji R  
**Platform:** pwn.college  
**Dojo:** Reverse Engineering   
**Series:** Metadata and Data 

---

## Challenge 1: Metadata and Data (Python)

### Analysis

The challenge binary was a Python script, readable with `cat`:

```python
header = file.read1(13)
assert header[:4] == b"(M63"          # magic bytes
assert int.from_bytes(header[4:8], "little") == 1   # version (4 bytes LE)
width  = int.from_bytes(header[8:9], "little")
assert width == 62                      # width (1 byte)
height = int.from_bytes(header[9:13], "little")
assert height == 14                     # height (4 bytes LE)
data = file.read1(width * height)
assert len(data) == width * height
```

### Header Layout

| Offset | Size | Value | Description |
|--------|------|-------|-------------|
| 0x00   | 4    | `(M63` | Magic bytes |
| 0x04   | 4    | `0x00000001` | Version (LE u32) |
| 0x08   | 1    | `0x3E` (62) | Width (u8) |
| 0x09   | 4    | `0x0000000E` (14) | Height (LE u32) |

**Total header:** 13 bytes
**Data size:** 62 × 14 = **868 bytes**

### Solve

```bash
python3 -c "
import sys
magic   = b'(M63'
version = (1).to_bytes(4, 'little')
width   = (62).to_bytes(1, 'little')
height  = (14).to_bytes(4, 'little')
data    = b'A' * (62 * 14)
sys.stdout.buffer.write(magic + version + width + height + data)
" > /tmp/solve.cimg

/challenge/cimg /tmp/solve.cimg
```

---

## Challenge 2: Metadata and Data (C)

### Analysis

The binary was a compiled ELF. Reversed with `objdump -d`. Key checks extracted from `main`:

```
401301:  cmpb $0x43, 0xf(%rsp)   # 'C'
401308:  cmpb $0x6d, 0x10(%rsp)  # 'm'
40130f:  cmpb $0x40, 0x11(%rsp)  # '@'
401316:  cmpb $0x67, 0x12(%rsp)  # 'g'  -> magic = "Cm@g"

40132c:  cmpw $0x1, 0x13(%rsp)   # version = 1 (u16 LE)
40133b:  cmpb $0x47, 0x15(%rsp)  # width = 0x47 = 71 (u8)
401349:  cmpw $0x15, 0x16(%rsp)  # height = 0x15 = 21 (u16 LE)
```

The `malloc(0x5d3)` call confirms data size: 71 × 21 = **1491 = 0x5D3**.

Header buffer size from `mov $0x9,%edx` passed to `read_exact` = **9 bytes**.

### Header Layout

| Offset | Size | Value | Description |
|--------|------|-------|-------------|
| 0x00   | 4    | `Cm@g` | Magic bytes |
| 0x04   | 2    | `0x0001` | Version (LE u16) |
| 0x06   | 1    | `0x47` (71) | Width (u8) |
| 0x07   | 2    | `0x0015` (21) | Height (LE u16) |

**Total header:** 9 bytes
**Data size:** 71 × 21 = **1491 bytes**

### Solve

```bash
python3 -c "
import sys
magic   = b'Cm@g'
version = (1).to_bytes(2, 'little')
width   = (71).to_bytes(1, 'little')
height  = (21).to_bytes(2, 'little')
data    = b'A' * (71 * 21)
sys.stdout.buffer.write(magic + version + width + height + data)
" > /tmp/solve.cimg

/challenge/cimg /tmp/solve.cimg
```

---

## Challenge 3: Metadata and Data (x86)

### Analysis

Same approach as Challenge 2 but with wider integer types for the dimension fields. Key checks from `main`:

```
4012fc:  cmpb $0x28, 0xa(%rsp)   # '('
401303:  cmpb $0x4e, 0xb(%rsp)   # 'N'
40130a:  cmpb $0x6e, 0xc(%rsp)   # 'n'
401311:  cmpb $0x72, 0xd(%rsp)   # 'r'  -> magic = "(Nnr"

40132c:  cmpw $0x1, 0xe(%rsp)    # version = 1 (u16 LE)
40133b:  cmpl $0x40, 0x10(%rsp)  # width = 0x40 = 64  (u32 LE) <-- cmpl not cmpb/cmpw
401349:  cmpl $0xc, 0x14(%rsp)   # height = 0xc = 12  (u32 LE)
```

The critical difference from Challenge 2: width and height are **4-byte dwords** (`cmpl`), expanding the header to **14 bytes**. The `malloc(0x300)` call confirms: 64 × 12 = **768 = 0x300**.

### Header Layout

| Offset | Size | Value | Description |
|--------|------|-------|-------------|
| 0x00   | 4    | `(Nnr` | Magic bytes |
| 0x04   | 2    | `0x00000001` | Version (LE u16) |
| 0x06   | 4    | `0x00000040` (64) | Width (LE u32) |
| 0x0A   | 4    | `0x0000000C` (12) | Height (LE u32) |

**Total header:** 14 bytes
**Data size:** 64 × 12 = **768 bytes**

### Solve

```bash
python3 -c "
import sys
magic   = b'(Nnr'
version = (1).to_bytes(2, 'little')
width   = (64).to_bytes(4, 'little')
height  = (12).to_bytes(4, 'little')
data    = b'A' * (64 * 12)
sys.stdout.buffer.write(magic + version + width + height + data)
" > /tmp/solve.cimg

/challenge/cimg /tmp/solve.cimg
```

---
