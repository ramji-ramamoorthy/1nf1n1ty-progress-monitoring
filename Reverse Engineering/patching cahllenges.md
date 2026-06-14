# pwn.college -- Reverse Engineering: Patching

**Author** Ramji R  
**Platform** Pwn.college  
**Category:** Reverse Engineering    

---

## Core Concept

These challenges present a license key verifier binary. Instead of reversing the key transformation, you **patch the binary itself** -- flipping conditional jumps so the program always takes the success path.

The binary takes patch inputs interactively via stdin:
```
Offset (hex) to change: <offset>
New value (hex): <value>
```

**Key opcode:** `jne` = `0x75`, flip to `je` = `0x74`  
**2-byte jne:** `0f 85` → `0f 84` (patch the second byte: `0x85` → `0x84`)

---

## Methodology

```bash
# Find the target jump
objdump -d /challenge/binary | grep -B2 -A8 "jne\|jnz"

# Look for the jne right after memcmp, just before the win/flag call
# Then pipe offsets + dummy key directly to the binary
printf "<offset1>\n<val1>\n...\n<dummy_input>\n" | /challenge/binary
```

---

## Challenge 1 -- Patched Up (Easy)

**Patches allowed:** 5  
**Target:** Single `jne` guarding the win function

```
2911:   75 14   jne 2927   ← flip this
2918:   call    <win>
```

**Solve:**
```bash
printf "2911\n74\n1000\n90\n1001\n90\n1002\n90\n1003\n90\naaaa\n" | /challenge/patched-up-easy
```
Remaining 4 patches are harmless NOPs at unused offsets.

---

## Challenge 2 -- Patched Up (Hard)

**Patches allowed:** 5  
**Target:** Same structure, different binary. MD5 mangler on input (irrelevant since we patch the jump)

```
1e7d:   75 14   jne 1e93   ← flip this
1e84:   call    <win>
```

**Solve:**
```bash
printf "1e7d\n74\n1000\n90\n1001\n90\n1002\n90\n1003\n90\naaaa\n" | /challenge/patched-up-hard
```

---

## Challenge 3 -- Puzzle Patch (Easy)

**Patches allowed:** 5  
**Target:** Input goes through a puzzle/mangling transform. Patch the final comparison jump anyway.

```
21c0:   75 14   jne 21d6   ← flip this
21c7:   call    <win>
```

**Solve:**
```bash
printf "21c0\n74\n1000\n90\n1001\n90\n1002\n90\n1003\n90\naaaa\n" | /challenge/puzzle-patch-easy
```

---

## Challenge 4 -- Puzzle Patch (Hard)

**Patches allowed:** 5  
**Target:** Same as Easy variant, different offsets

```
17f4:   75 14   jne 180a   ← flip this
17fb:   call    <win>
```

**Solve:**
```bash
printf "17f4\n74\n1000\n90\n1001\n90\n1002\n90\n1003\n90\naaaa\n" | /challenge/puzzle-patch-hard
```

---

## Challenge 5 -- Patch Perfect (Easy)

**Patches allowed:** 2  
**New mechanic:** Binary has an **integrity check** -- checksums itself before the key check. Both jumps must be patched.

```
269c:   75 6e   jne 270c   ← INTEGRITY CHECK, flip this
292e:   75 14   jne 2944   ← KEY CHECK → win, flip this
```

Both patches use up exactly the 2 allowed bytes.

**Solve:**
```bash
printf "269c\n74\n292e\n74\naaaa\n" | /challenge/patch-perfect-easy
```

---

## Challenge 6 -- Patch Perfect (Hard)

**Patches allowed:** 2  
**New mechanic:** Integrity check uses a **2-byte jne** (`0f 85`), so the opcode to patch is `0x85` → `0x84` at offset+1

```
1a23:   0f 85 e3 00 00 00   jne 1b0c   ← INTEGRITY CHECK (2-byte), patch 1a24: 85→84
1b08:   75 2c               jne 1b36   ← KEY CHECK → win, patch 1b08: 75→74
```

**Solve:**
```bash
printf "1a24\n84\n1b08\n74\naaaa\n" | /challenge/patch-perfect-hard
```

---

## Tools Used
- `objdump -d` -- disassembly
- `grep jne\|jnz` -- finding conditional jumps
- `printf ... | /challenge/binary` -- piping patch values + input

---
