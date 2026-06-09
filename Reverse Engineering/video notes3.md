# pwn.college -- Reverse Engineering: Functions and Frames

**Author:** Ramji R  
**Source:** [pwn.college - Reverse Engineering - Functions and Frames (YouTube)](https://www.youtube.com/watch?v=)  
**Lecture Series:** Reverse Engineering  
**Instructor:** Yan Shoshitaishvili (Zardus), Arizona State University  

---

## 1. What Is a Program?

- A program is a collection of **modules** (libraries + the main binary)
- Each module is made up of **functions**
- Functions contain blocks of instructions that:
  - Operate on variables and data structures
  - Trigger system calls
  - Implement the actual functionality of the program

---

## 2. Modules

- Modules = collections of functions shipped in a single library
- They exist to make developers' lives easier
  - e.g. instead of writing dynamic memory allocation from scratch, you use libc (`malloc`, `free`, etc.)

### Implication for reverse engineering
- You typically do NOT need to reverse engineer library code
- Libraries are well documented -- look up their documentation to understand what a program is trying to achieve with them
- What you DO need to reverse engineer is the **main module** of the program

---

## 3. Functions -- The Atomic Unit of Reverse Engineering

- Functions are the **atomic construct** for reverse engineering
- Each function has a **well-defined goal**:
  - Getter/setter in OOP
  - Calculation
  - Input parsing/dispatching
- Functions can be reverse engineered **in isolation**
  - Open a function, understand what it does, move to the next
  - As you reverse more functions, a broader understanding of the software emerges

---

## 4. Functions -- The Control Flow Graph (CFG)

- A function is made up of **basic blocks** -- groups of instructions that always execute together
- Blocks are connected by **edges** representing conditional or unconditional control flow transfers
- The result is a **Control Flow Graph (CFG)**
- Every execution of a function = a path through this graph

### CFG edge colors (in Ghidra/Binary Ninja/IDA)
| Color | Meaning |
|-------|---------|
| Blue | Unconditional jump -- always taken |
| Green | Conditional jump -- taken if condition is TRUE |
| Red | Fall-through -- taken if condition is FALSE |

### Example -- simple cat in assembly
```
_start:
    call main
    call exit

main:
    [prologue]
    --> loop start:
        syscall read(stdin, stack_buf, 256)   ; rdi=0, rsi=rsp, rdx=256
        cmp rax, 0
        jle [return]                          ; if read <= 0, exit loop
        syscall write(stdout, stack_buf, rax) ; rdi=1, rsi=rsp, rdx=rax
        jmp [loop start]                      ; unconditional -- keep looping
    [epilogue]
    ret
```

- Following the CFG reveals: as long as read returns > 0, keep reading and writing -- this IS cat

---

## 5. The Stack

- Local variables live on the **stack** (not in ELF sections)
- ELF data sections recap:
  - `.data` -- pre-initialized global writable data
  - `.rodata` -- global read-only data
  - `.bss` -- uninitialized global writable data
- None of these handle **local variables** -- that is the stack's job

### What the stack stores
- Local variables of the current function
- Execution context (return addresses, saved registers)

### Stack direction -- grows backwards
- Due to a historical oddity, the stack **grows towards lower addresses**
- `push` instruction: **RSP decreases by 8**, data written at new RSP
- `pop` instruction: **RSP increases by 8**, data read from old RSP

```
push 0x41   --> RSP -= 8, mem[RSP] = 0x41
push 0x42   --> RSP -= 8, mem[RSP] = 0x42
push 0x43   --> RSP -= 8, mem[RSP] = 0x43

Stack (low addr) --> [0x43][0x42][0x41] (high addr)
```

- Higher memory address = bottom of stack (older data)
- Lower memory address = top of stack (newest data, where RSP points)

---

## 6. The Stack -- Initial Layout at Program Start

When a program starts, the stack already contains:

```
[argc] [argv[0]] [argv[1]] ... [argv[n]] [NULL] [envp[0]] [envp[1]] ... [envp[n]] [NULL] [strings...]
```

- **argc** -- number of arguments (at top of initial stack)
- **argv[]** -- array of pointers to argument strings, NULL-terminated
- **envp[]** -- array of pointers to environment strings, NULL-terminated
- **The actual strings** -- stored above the pointer arrays, null-terminated

### Example
```
Program run as: ./program hello world
With env vars:  VAR1=a VAR2=b VAR3=c

Stack holds:
argc = 3
argv[0] -> "./program"
argv[1] -> "hello"
argv[2] -> "world"
argv[3] = NULL
envp[0] -> "VAR1=a"
envp[1] -> "VAR2=b"
envp[2] -> "VAR3=c"
envp[3] = NULL
... actual strings follow ...
```

- `argv[0]` dereferences the pointer and gives you the string `"./program"`
- `argc` is the count (3 in this case)

---

## 7. Call and Return -- How Functions Are Invoked

### `call` instruction
- **Implicitly pushes** the return address (address of the instruction AFTER the call) onto the stack
- Then jumps to the target function

```asm
; Example at address 0x401050:
call main        ; pushes 0x401055 (next instruction) onto stack, jumps to main
; 0x401055: next instruction here
```

### `ret` instruction
- **Implicitly pops** the return address from the stack
- Jumps to that address

```asm
ret              ; pops saved return address, jumps there
```

### call vs jmp
| Instruction | Effect |
|-------------|--------|
| `call` | Pushes return address, then jumps |
| `jmp` | Just jumps -- no return address saved |
| `ret` | Pops return address, jumps there |

---

## 8. Function Prologue -- Setting Up the Stack Frame

The **prologue** runs at the start of every function. It sets up the **stack frame** -- the function's private area of the stack for local variables.

### Two key registers
- **RSP (Stack Pointer)** -- always points to the TOP (lowest address, left side) of the current stack frame
- **RBP (Base Pointer / Frame Pointer)** -- always points to the BOTTOM (highest address, right side) of the current stack frame

### Standard prologue
```asm
push rbp          ; save caller's base pointer onto stack
mov rbp, rsp      ; set base pointer = current stack pointer
                  ; (RBP and RSP now point to same location -- top of this frame)
sub rsp, 0x50     ; "allocate" space for local variables
                  ; (move RSP left to create the frame)
```

### What this achieves
- Saves the old RBP so the caller's frame can be restored later
- Establishes a new stable RBP for this function's frame
- Creates space for local variables by moving RSP leftward

### Note on "allocation"
- `sub rsp, N` does NOT create new memory -- the memory was always there
- It simply marks the region between RSP and RBP as belonging to this function's frame

---

## 9. Function Epilogue -- Tearing Down the Stack Frame

The **epilogue** runs at the end of every function. It tears down the stack frame and prepares to return to the caller.

### Standard epilogue
```asm
mov rsp, rbp      ; "deallocate" -- move RSP back to where RBP is
                  ; (effectively discards all local variables)
pop rbp           ; restore caller's base pointer
ret               ; pop return address and jump there
```

### Shorthand -- `leave` instruction
```asm
leave             ; equivalent to: mov rsp, rbp / pop rbp
ret
```

### Important security note
- Tearing down the frame does NOT zero out or destroy the data
- The data from the old frame **stays in memory** -- RSP just moved past it
- If a new stack frame is set up without overwriting this region, **old sensitive data can be leaked**
- This is the root cause of many **stack data leakage vulnerabilities**

---

## 10. Frame Pointer Optimization

- Modern compilers can optimize away RBP as a frame pointer
- With optimization, RBP is freed up as a **general purpose register**
- Stack management is done directly via RSP arithmetic instead

```asm
; Without frame pointer optimization (standard)
push rbp
mov rbp, rsp
sub rsp, 0x50

; With frame pointer optimization (optimized)
push rbp          ; still pushed due to calling convention (callee-saved)
sub rsp, 0x50     ; RSP managed directly, no RBP frame setup
```

- Don't panic if you see a function that never sets `mov rbp, rsp` -- this is normal in optimized binaries
- RSP-relative addressing replaces RBP-relative addressing

---

## 11. Callee-Saved Registers in the Prologue

- Certain registers are **callee-saved** by calling convention: `RBX`, `RBP`, `R12`, `R13`, `R14`, `R15`
- If a function uses these registers, it MUST save them at the start and restore them at the end
- This is why you often see extra `push` instructions in the prologue beyond just `push rbp`

```asm
; Prologue saving multiple callee-saved registers
push rbp
push rbx
push r12
push r13
mov rbp, rsp
sub rsp, 0x30
```

- The epilogue must `pop` them in reverse order before `ret`
- Violating this is "impolite" -- it corrupts the caller's register state

---


# pwn.college -- Reverse Engineering: Data Access

**Author:** Ramji R   
**Source:** [pwn.college - Reverse Engineering - Data Access (YouTube)](https://www.youtube.com/watch?v=)  
**Lecture Series:** Reverse Engineering  
**Instructor:** Yan Shoshitaishvili (Zardus), Arizona State University  

---

## 1. Recap -- Data Regions

Data can live in several different memory regions. Knowing which region data is in tells you how it will be accessed in assembly.

| Region | Contents | Notes |
|--------|----------|-------|
| `.data` | Pre-initialized global writable data | Stored in ELF file |
| `.rodata` | Global read-only data | Stored in ELF file |
| `.bss` | Uninitialized global writable data | NOT stored in ELF -- mapped as zeroed memory |
| Stack | Statically allocated local variables | Covered in previous video |
| Heap | Dynamically allocated data (`malloc`/`free`) | Covered in depth in later modules |

---

## 2. Stack Data Access

Data on the stack is accessed in three ways:

### Method 1 -- push / pop
```asm
push rax        ; store rax at top of stack, RSP -= 8
pop rbx         ; load top of stack into rbx, RSP += 8
```

### Method 2 -- RSP-relative addressing
- RSP points to the **top (lowest address / left side)** of the stack frame
- Offsets from RSP are **positive** (going right / forward in memory into the frame)

```asm
mov rax, [rsp]        ; access top of stack
mov rax, [rsp + 0x8]  ; access 8 bytes above top
mov rax, [rsp + 0x10] ; access 16 bytes above top
```

### Method 3 -- RBP-relative addressing
- RBP points to the **bottom (highest address / right side)** of the stack frame
- Offsets from RBP are **negative** (going left / backward in memory towards RSP)

```asm
mov rax, [rbp - 0x8]   ; first local variable
mov rax, [rbp - 0x10]  ; second local variable
mov rax, [rbp - 0x28]  ; further into the frame
```

### Same memory, two views
```
Stack frame layout:

Low addr (RSP) <----------------> High addr (RBP)
[rsp+0x00] = [rbp-0x28]
[rsp+0x08] = [rbp-0x20]
[rsp+0x10] = [rbp-0x18]
[rsp+0x18] = [rbp-0x10]
[rsp+0x20] = [rbp-0x08]
                          [rbp+0x00] = saved old RBP
                          [rbp+0x08] = return address
```

### Method 4 -- indirect (pointer in register)
```asm
mov rdx, rsp      ; copy RSP into rdx
mov rax, [rdx]    ; dereference rdx to access stack data
```

- This is tricky during reversing -- you see `[rdx]` but you have to trace back where `rdx` was set
- Could have been set in a completely different function

---

## 3. ELF Section Data Access (.data / .rodata / .bss)

- `.data`, `.rodata`, and `.bss` are all stored in the ELF file (or mapped directly after it)
- They are always at a **known offset from the program code**
- Accessed using **RIP-relative addressing**

### Why RIP-relative?
- The ELF is mapped into memory in two separate regions:
  - Code segment (text) -- where instructions live
  - Data segments (.data, .rodata, .bss) -- mapped separately at a different address
- The distance between code and data is fixed and known at compile time
- RIP (instruction pointer) always points to the current instruction, so `[rip + offset]` gives a stable reference to data

### Load (read from .data / .rodata / .bss)
```asm
mov rax, [rip + 0x2008]    ; load value from global variable
                            ; offset is usually a large number (code and data are far apart in memory)
```

### Store (write to .data / .bss)
```asm
mov [rip + 0x2008], rax    ; store value into global variable
```

### Get address of global (LEA)
```asm
lea rax, [rip + 0x2008]    ; load effective address -- rax = address of global variable
                            ; no dereference, just the address itself
```

- After `lea`, `rax` holds a **pointer** to the global variable
- Later code that uses `[rax]` is dereferencing that pointer
- During reversing: when you see `lea rax, [rip+offset]` followed later by `[rax]`, trace where `rax` goes -- it's a pointer to a global

---

## 4. Heap Data Access

- Heap data is returned by `malloc` as a pointer in `rax`
- That pointer must be stored somewhere:
  - On the stack (in a local variable)
  - In a global variable (`.data` / `.bss`)
  - Kept in a register (short-lived)

```c
// C code
char *buf = malloc(256);
buf[0] = 'A';

// Compiled to roughly:
mov rdi, 256
call malloc          ; rax = pointer to heap buffer
mov [rbp - 0x8], rax ; store heap pointer as local variable on stack

mov rax, [rbp - 0x8] ; load the pointer back
mov byte [rax], 0x41  ; dereference and write 'A' to heap
```

---

## 5. Critical Distinction -- Direct vs Indirect Access

This is one of the most important things to get right when reading disassembly.

### Direct stack access (one dereference)
```asm
mov rax, [rsp]       ; read the VALUE stored at the top of the stack
```
- RSP points to some address
- `[rsp]` reads the 8 bytes at that address
- You get whatever VALUE is sitting there

### Indirect / pointer access (two dereferences)
```asm
mov rax, [rsp]       ; rax = VALUE stored at top of stack (this value is itself a pointer)
mov rdx, [rax]       ; rdx = VALUE stored at the address rax points to
```
- The value at the top of the stack is itself a **pointer** (e.g. a heap pointer stored as a local variable)
- `[rax]` dereferences that pointer to get the actual data

### Side by side
```asm
; Example 1 -- direct stack access
mov rax, rsp
mov rdx, [rax]       ; reads data AT the stack pointer location
                     ; same as: mov rdx, [rsp]

; Example 2 -- indirect (pointer on stack pointing to heap)
mov rax, [rsp]       ; rax = pointer stored on stack (e.g. malloc return value)
mov rdx, [rax]       ; rdx = data at the heap address rax points to
```

- The only visual difference: **one set of square brackets vs two**
- Getting this wrong during reversing = completely misunderstanding what data is being accessed
- Pointer access is most commonly seen with **heap-allocated data**

---

## 6. Data Structures -- Structs and Objects in Assembly

- In C/C++ you can define structs, objects, and complex nested types
- **All of this information is lost during compilation**
- The compiler just sees a blob of bytes at a base address and accesses fields by fixed offsets

```c
// C struct
struct User {
    int id;        // offset 0
    char name[16]; // offset 4
    int age;       // offset 20
};

struct User u;
u.id = 1;
u.age = 25;

// Compiled to roughly:
mov dword [rbp - 0x20], 1       ; u.id = 1   (offset 0 from base)
mov dword [rbp - 0x0c], 25      ; u.age = 25 (offset 20 from base)
```

### How to reconstruct structs during reversing
- Watch how data is accessed relative to a base pointer or heap pointer
- Multiple accesses to `[rax]`, `[rax + 4]`, `[rax + 8]`, etc. suggest a struct
- Group fields by their offsets and infer types from how they're used (byte access = char, dword = int, qword = pointer, etc.)
- There is no shortcut -- this takes practice

---

## 7. Summary -- How to Identify Data Access Type

When you see a memory access in disassembly, ask:

| What you see | What it means |
|---|---|
| `[rsp + N]` or `[rbp - N]` | Stack local variable (direct) |
| `push` / `pop` | Stack access |
| `[rip + large_offset]` | Global variable in .data / .rodata / .bss |
| `lea reg, [rip + offset]` | Address of a global (pointer, not value) |
| `mov reg, [rsp]` then `[reg]` | Pointer stored on stack -- probably heap data |
| `call malloc` then `[rax]` | Heap data access |
| `[rax + 0]`, `[rax + 4]`, `[rax + 8]` | Struct or object field access via pointer |

---

*parts of this notes were AI generated for clarity*
