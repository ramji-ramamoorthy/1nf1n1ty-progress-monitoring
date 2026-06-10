# Reverse Engineering Tools
**pwn.college - Reverse Engineering Module**

---

## 1. Static vs Dynamic Tools

| | Static Tools | Dynamic Tools |
|---|---|---|
| **When** | Program at rest | Program at runtime |
| **What** | Analyzes the ELF file itself | Analyzes execution behavior |
| **Examples** | Binary Ninja, Ghidra, IDA Pro | GDB, ltrace, strace |

---

## 2. Static Reverse Engineering Tools

### 2.1 Simple/Recon Tools

**`nm`**
- Lists symbols imported/exported by a binary
- Quick way to see what functions a program uses (e.g. `printf`, `malloc`)
- Usage: `nm -d <binary>`

**`strings`**
- Dumps ASCII strings found anywhere in the file
- Useful for finding hardcoded paths, messages, keys
- Good starting point to understand program intent

**`objdump`**
- Simple disassembler, lighter than the advanced tools
- Usage: `objdump -d <binary>`

**`checksec`**
- Analyzes security mitigations on a binary
- Reports: NX, PIE, stack canaries, RELRO, ASLR, etc.
- Important to run before exploitation attempts
- Usage: `checksec <binary>`

**`kaitai struct`**
- Browser-based binary format explorer
- Supports ELF, ZIP, PDF, and many other formats
- Useful for visualizing file structure

---

### 2.2 Advanced Disassemblers (Interactive)

These tools let you build up a mental model of a program slowly, adding comments, renaming variables, and navigating the control flow graph.

**IDA Pro**
- Industry gold standard, oldest major tool
- Extremely expensive (worth it if reversing is your livelihood)
- Coined the term "interactive disassembler"

**Binary Ninja / Binary Ninja Cloud**
- Main commercial competitor to IDA
- `cloud.binary.ninja` - free browser-based version, no install needed
- Recommended tool for this module

**Ghidra**
- Created by the NSA, open sourced
- Mature, powerful, free
- Good alternative to Binary Ninja

**Cutter**
- GUI frontend for the open source `radare2` project
- Free and open source

**angr**
- Academic binary analysis framework
- Includes a GUI, open source
- Focuses on automated analysis and symbolic execution

---

### 2.3 Binary Ninja Cloud Workflow

1. Go to `cloud.binary.ninja`, create an account
2. Upload your ELF file, hit Continue and let it analyze
3. Toggle useful views:
   - **Graph view** - control flow graph per function
   - **Linear disassembly** - more like traditional objdump output
   - **Addresses on/off** - adds addresses to graph nodes
   - **Opcode bytes** - shows raw hex bytes per instruction
   - **High Level IL (HLIL)** - decompiles to near-C pseudocode

4. Key features:
   - **Comments** - right click any instruction, add notes
   - **Variable renaming** - Binary Ninja auto-identifies stack variables (e.g. `var_108`), you can rename them to something meaningful like `input_buffer`
   - **Variable tracking** - it links the same variable across reads/writes automatically
   - **Cloud saves** - all annotations saved automatically, multiple sessions supported

> **Note:** For this module, stay in the **assembly view**. HLIL is useful but can hide important details.

---

## 3. Dynamic Reverse Engineering Tools

### 3.1 ltrace and strace

**`strace`** - traces **system calls**
**`ltrace`** - traces **library calls**

Both are useful for both debugging and reversing.

**ltrace diff technique:**
```bash
ltrace ./binary <<< "AAAAAAAAAA" > trace_a.txt 2>&1
ltrace ./binary <<< "BBBBBBBBBB" > trace_b.txt 2>&1
diff trace_a.txt trace_b.txt
```
Diffing two traces with different inputs gives you a high-level picture of program behavior. Enough to solve some challenges on its own.

---

### 3.2 GDB Basics (Review)

- Break at entry: `break _start` or `break main`
- Disassemble: `disassemble main`
- Step one instruction: `si` (step instruction)
- Examine memory: `x/5i $rip` (next 5 instructions), `x/s $rsp` (string at rsp)
- Recommended: use a GDB enhancement plugin

**`.gdbinit` setup (recommended):**
```
set disassembly-flavor intel
set history save on
source /path/to/pwndbg/gdbinit.py   # or peda / gef
```

---

### 3.3 GDB Scripting

You can write a `.gdb` script file with arbitrary GDB commands and run it with:
```bash
gdb -x script.gdb ./binary
```

**Example script (`cat.gdb`):**
```gdb
break *0x401025
commands
  x/5i $rip
  x/s $rsp
  continue
end
run
```

Key uses:
- Auto-print registers/memory at every breakpoint hit
- Log data across many loop iterations without manual intervention
- Use `printf` in commands to format output for external scripts
- Modify memory to skip parts of a program you've already understood (`set {int}0xaddr = value`)

**`display` command** - runs automatically every time GDB stops:
```gdb
display/5i $rip
```

---

### 3.4 Position Independent Executables (PIE) in GDB

**Position-dependent binary** (compiled without `-pie`): loads at fixed address like `0x401000`. Breakpoints work directly with known addresses.

**PIE binary** (compiled with `-static-pie` or `-pie`): loads at a random base address. Raw addresses like `0x401025` are useless.

**Fix:** Set a convenience variable for the base address in `.gdbinit`:
```gdb
set $base = 0x7ffff7ffd000   # find this with: info proc mappings
```
Then set breakpoints as:
```gdb
break *($base + 0x1025)
```

> The exact base GDB uses varies by version. Common values: `0x555555554000` or `0x7ffff7ffd000`. Check `info proc mappings` on your system.

---

### 3.5 Timeless Debugging / Record-Replay

Run the program once, record the entire execution trace, then rewind/replay freely without re-running.

#### GDB Built-in Record/Replay

```gdb
# Start recording after breaking at entry
record

# Continue normally, do your input
continue

# Now reverse-step through execution
reverse-stepi    # rs  - one instruction backward
stepi            # si  - one instruction forward
reverse-continue # rc  - rewind to start of recording
```

To stop recording but keep program alive at current position:
```gdb
record stop
```

> **Quirk:** GDB's record replay doesn't save register state at every single instruction due to optimizations. Register values shown while reverse-stepping may be inaccurate (e.g. shows return value of a syscall instead of value before it). Workaround: rewind to where you want, do `record stop`, then step forward normally from there.

#### External Tools (Better Performance)

**`rr` (Mozilla)**
- Designed for complex programs like Firefox
- Requires specific hardware performance counters
- Not available in pwn.college environment, must run locally
- Install: `https://rr-project.org`

**`Kira`**
- Built specifically for reverse engineering / hacking
- More complex setup, but Docker container available
- Website: `akira.me`

---

## 4. Toolchain Summary

| Goal | Tool |
|---|---|
| Quick recon | `strings`, `nm`, `checksec` |
| Disassembly / static analysis | Binary Ninja Cloud, Ghidra |
| Trace syscalls/libcalls | `strace`, `ltrace` |
| Step-through debugging | GDB + pwndbg/peda/gef |
| Automate GDB behavior | GDB scripting (`-x script.gdb`) |
| Rewind execution | GDB `record`, `rr`, Kira |

---

## 5. Quick Reference: GDB Commands

```
run / r                   - start program
continue / c              - resume execution
break *0xADDR             - set breakpoint at address
break func_name           - set breakpoint at function
info breakpoints          - list breakpoints
delete N                  - delete breakpoint N
stepi / si                - step one instruction (into calls)
nexti / ni                - step one instruction (over calls)
finish                    - run until function returns
x/Ni $rip                 - print N instructions at rip
x/s $rsp                  - print string at rsp
x/gx $rsp                 - print 8-byte hex at rsp
info registers            - show all registers
info proc mappings        - show memory map (find PIE base)
set $var = value          - set convenience variable
record                    - start record/replay
reverse-stepi             - step one instruction backward
reverse-continue          - rewind to start of recording
record stop               - stop recording, keep program live
```

---

*refined using AI*
