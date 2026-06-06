# pwn.college Notes: Reverse Engineering and Program Interaction

---

## 1. Reverse Engineering: Introduction

- RE = figuring out what a program does **without** source code
- You work with compiled binaries (ELF on Linux)
- Tools you'll use: `objdump`, `ghidra`, `gdb`
- Compiled C gets turned into assembly and you read that assembly to understand the logic
- Key idea: every program is just instructions and reversing is just reading them
- Common goal: find flags, bypass checks, understand behavior
- Start by running the binary, see what it does, then dig into the disassembly

---

## 2. Program Interaction: Binary Files

- Everything in Linux is a file and binaries are just files with executable permissions
- ELF format = how Linux structures executables
  - Header tells the OS how to load the file
  - Sections: `.text` (code), `.data` (initialized data), `.bss` (uninitialized), `.rodata` (read-only strings)
- `file` command tells you what kind of binary it is
- `xxd`, `readelf`, `objdump` let you inspect the raw binary
- Static vs Dynamic linking: dynamic means the binary needs shared libs (`.so` files) at runtime
- `ldd` shows what shared libraries a binary depends on
- `strings` extracts readable text embedded inside a binary

---

## 3. Program Interaction: Linux Process Loading

- When you run a binary, the kernel loads it into memory
- The loader maps ELF sections into virtual memory
- Memory layout (low to high):
  - Text segment (code)
  - Data / BSS
  - Heap (grows up)
  - Stack (grows down)
  - Kernel space (top)
- Dynamic linker (`ld.so`) runs first and resolves shared library symbols before `main()` is called
- `argc`, `argv`, and `envp` are passed onto the stack when the process starts
- Entry point is not `main()` but `_start`, which sets things up then calls `main()`
- `strace` lets you watch all system calls a process makes in real time
- `procfs` (`/proc/<pid>/`) gives a live view of a running process's memory, file descriptors, and memory maps

---

## Summary

These three modules form a pipeline:

1. What is reverse engineering
2. What is a binary
3. How a binary comes alive as a process
