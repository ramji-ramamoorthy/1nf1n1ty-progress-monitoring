# Memory Forensics - Quick Notes

## What is Memory Forensics?
- Branch of digital forensics dealing with **acquisition and analysis of volatile memory (RAM)**
- Also called **Volatile Memory Forensics**

## Why it Matters
- Gives insight into the state of a compromised system
- Often the **only** source of evidence for memory-resident malware
- Example: **Stuxnet** -- lived only in memory, dormant until a target was found
- Modern malware trend: fileless/memory-only attacks

---

## Prerequisites (nice to have, not mandatory)
- Basic programming -- C, Python, Java
- Data structures -- Arrays, Stack, Heap, Linked Lists, Trees
- OS internals knowledge (helpful for deep analysis)

---

## Two High-Level Steps

### 1. Evidence Acquisition

**Windows Tools:**
| Tool | Notes |
|------|-------|
| **DumpIt** | Lightweight, low memory footprint, free (Comae) |
| **FTK Imager** | GUI-based, also does disk imaging (AccessData) |
| **Magnet Acquire** | Free, supports smartphones + computers |

**Linux Tools:**
| Tool | Command | Notes |
|------|---------|-------|
| **AVML** | `sudo ./avml output.lime` | By Microsoft, written in Rust. Don't use `--compress` |
| **LiME** | `sudo insmod lime.ko path=output.lime format=lime` | Kernel module based |

> Do NOT compress AVML output -- Volatility can't analyze compressed dumps directly.

---

### 2. Evidence Analysis -- Volatility

- Industry-standard tool, **free and open source**
- Volatility 2 (Python 2) -- covered here; Volatility 3 is newer

#### Determining the Profile (Windows only)
```bash
volatility -f memory.dmp imageinfo       # High-level summary + suggested profiles
volatility -f memory.dmp kdbgscan        # Reads _KDDEBUGGER_DATA64 block for build info
```

**Profile issues can occur when:**
- Malware tampers with `_KDDEBUGGER_DATA64`
- Multiple kernel debugger blocks exist (e.g., after hot patches without reboot)

#### Common Plugins
```bash
# Environment variables
volatility -f memory.dmp --profile=WinXPSP2x86 envars

# Active process list
volatility -f memory.dmp --profile=WinXPSP2x86 pslist

# List all plugins
volatility -f memory.dmp --profile=WinXPSP2x86 -h
```

#### What Volatility Can Do
- Detect active network connections
- Detect malware in memory
- List/extract open files
- Dump registry hives
- Extract password hashes
- Extract browser history + command prompt history
- List loaded drivers

> Plugin list varies depending on the profile used.

---

## Practice Resources
- **MemLabs** -- CTF-style memory forensics labs (Windows dumps): [github.com/stuxnet999/MemLabs](https://github.com/stuxnet999/MemLabs)
- **CTFtime** -- Find upcoming CTFs
- **Digital Forensics Discord** -- Active DFIR community (maintained by Andrew Rathbun)

---
