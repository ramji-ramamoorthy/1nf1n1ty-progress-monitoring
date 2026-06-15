# MemLabs - Labs 1, 2 & 3 Writeup

**Author:** Ramji R  
**Tool:** Volatility3    
**OS:** Arch Linux    
**Flag Format:** `flag{...}` (Lab 3 uses `inctf{...}`)  

> All commands use `vol` as the alias for `python /path/to/volatility3/vol.py`  
> You can set this with: `alias vol='python ~/volatility3/vol.py'`

---

## Lab 1 - Beginner's Luck

### Challenge Description
> My sister's computer crashed. We were very fortunate to recover this memory dump. Your job is to get all her important files from the system. From what we remember, we suddenly saw a black window pop up with something being executed. When the crash happened, she was trying to draw something. That's all we remember from the time of crash.  
> **Note:** This challenge is composed of 3 flags.

---

### Step 1 - Identify the OS / Profile

In Volatility3, you don't need to specify a profile manually - it auto-detects. Start by listing processes to confirm the dump loads correctly.

```bash
vol -f MemoryDump_Lab1.raw windows.info
```

Output confirms: **Windows 7 SP1 x64**

---

### Step 2 - List Running Processes

```bash
vol -f MemoryDump_Lab1.raw windows.pslist
```

Key processes spotted:
- `cmd.exe`
- `mspaint.exe` (PID 2424)
- `WinRAR.exe` (PID 1512)

---

### Flag 1 - CMD Console (Base64)

The black window in the description hints at `cmd.exe`. Use the `cmdline` and `consoles` equivalent in Volatility3:

```bash
vol -f MemoryDump_Lab1.raw windows.cmdline
```

```bash
vol -f MemoryDump_Lab1.raw windows.consoles
```

Look for a command named `St4G3$1` - its output contains a Base64 string:

```
ZmxhZ3t0aDFzXzFzX3RoM18xc3Rfc3Q0ZzMhIX0=
```

Decode it:

```bash
echo ZmxhZ3t0aDFzXzFzX3RoM18xc3Rfc3Q0ZzMhIX0= | base64 -d
```

**Output:**
```
flag{th1s_1s_th3_1st_st4g3!!}
```

> **Flag 1:** `flag{th1s_1s_th3_1st_st4g3!!}`

---

### Flag 2 - MS Paint Memory Dump (GIMP)

The description says she was "drawing something" - that's `mspaint.exe` (PID 2424).

Dump the process memory:

```bash
vol -f MemoryDump_Lab1.raw -o ./lab1_output/ windows.memmap --pid 2424 --dump
```

This creates a file like `pid.2424.dmp`. Rename it to `.data`:

```bash
mv lab1_output/pid.2424.dmp lab1_output/2424.data
```

Open it in **GIMP**:
1. Open GIMP → `File > Open As` → select `2424.data`
2. Choose **Raw image data**
3. Set **Image Type** to `RGB`
4. Adjust **Width** (try around 1200–1300) and **Offset** until an image appears
5. The image will appear flipped - go to `Image > Transform > Rotate 180°`, then `Flip Horizontally`

The flag is visible in the rendered image.

> **Flag 2:** `flag{G00d_Boy_good_girL}`

---

### Flag 3 - WinRAR + NTLM Hash

Check what WinRAR was opening:

```bash
vol -f MemoryDump_Lab1.raw windows.cmdline | grep -i winrar
```

Output shows it was opening:
```
C:\Users\Alissa Simpson\Documents\Important.rar
```

Scan for the file in memory:

```bash
vol -f MemoryDump_Lab1.raw windows.filescan | grep -i "Important.rar"
```

Note the physical offset (e.g., `0x3fa3ebc0`), then dump it:

```bash
vol -f MemoryDump_Lab1.raw -o ./lab1_output/ windows.dumpfiles --physaddr 0x3fa3ebc0
```

Rename the dumped file and try to extract it:

```bash
mv lab1_output/file.*.dat Important.rar
unrar e Important.rar
```

It asks for a password and gives the hint:
```
Password is NTLM hash (in uppercase) of Alissa's account passwd.
```

Dump password hashes:

```bash
vol -f MemoryDump_Lab1.raw windows.hashdump
```

Find Alissa's NTLM hash (the second hash after the colon):
```
Alissa Simpson:1003:...:f4ff64c8baac57d22f22edc681055ba6:::
```

Use it in **uppercase** as the password:
```
F4FF64C8BAAC57D22F22EDC681055BA6
```

```bash
unrar e Important.rar
# Enter password: F4FF64C8BAAC57D22F22EDC681055BA6
```

This extracts `flag3.png` containing the flag.

> **Flag 3:** `flag{w3ll_3rd_stage_was_easy}`

---

### Lab 1 - All Flags

| Stage | Flag |
|-------|------|
| 1 | `flag{th1s_1s_th3_1st_st4g3!!}` |
| 2 | `flag{G00d_Boy_good_girL}` |
| 3 | `flag{w3ll_3rd_stage_was_easy}` |

---

---

## Lab 2 - A New World

### Challenge Description
> One of the clients of our company lost access to his system due to an unknown error. He is supposedly a very popular "environmental" activist. As part of the investigation, he told us that his go-to applications are browsers, his password managers, etc. We hope that you can dig into this memory dump and find his important stuff and give it back to us.  
> **Note:** This challenge is composed of 3 flags.

---

### Step 1 - List Processes

```bash
vol -f MemoryDump_Lab2.raw windows.pslist
```

Key processes:
- `chrome.exe`
- `KeePass.exe`

---

### Flag 1 - Environment Variables

The word `"environmental"` in the description is a hint - check environment variables:

```bash
vol -f MemoryDump_Lab2.raw windows.envars | grep "NEW_TMP"
```

Every process has a variable `NEW_TMP` set to a Base64 value:
```
C:\Windows\ZmxhZ3t3M2xjMG0zX1QwXyRUNGczXyFfT2ZfTDRCXzJ9
```

Decode the Base64 part:

```bash
echo ZmxhZ3t3M2xjMG0zX1QwXyRUNGczXyFfT2ZfTDRCXzJ9 | base64 -d
```

**Output:**
```
flag{w3lc0m3_T0_$T4g3_!_Of_L4B_2}
```

> **Flag 1:** `flag{w3lc0m3_T0_$T4g3_!_Of_L4B_2}`

---

### Flag 2 - KeePass Database

KeePass stores passwords in a `.kdbx` database file. Scan for it:

```bash
vol -f MemoryDump_Lab2.raw windows.filescan | grep -i ".kdbx"
```

Found:
```
\Device\HarddiskVolume2\Users\SmartNet\Secrets\Hidden.kdbx
```

Note the offset and dump it:

```bash
vol -f MemoryDump_Lab2.raw -o ./lab2_output/ windows.dumpfiles --physaddr <offset>
```

Now find the master password - scan for anything named "password":

```bash
vol -f MemoryDump_Lab2.raw windows.filescan | grep -i "password"
```

Found:
```
\Device\HarddiskVolume2\Users\Alissa Simpson\Pictures\Password.png
```

Dump the image:

```bash
vol -f MemoryDump_Lab2.raw -o ./lab2_output/ windows.dumpfiles --physaddr <offset>
```

Open `Password.png` - the master password is visible in the bottom-right corner of the image.

Use that password to open `Hidden.kdbx` in KeePass (install via `sudo pacman -S keepassxc`):

```bash
keepassxc lab2_output/Hidden.kdbx
```

The flag is the password stored inside the database.

> **Flag 2:** `flag{w0w_th1s_1s_Th3*SeC0nD_ST4g3*!!}`

---

### Flag 3 - Chrome History + Mega Link

Install the Volatility community plugin for Chrome history:

```bash
git clone https://github.com/superponible/volatility-plugins /tmp/vol-plugins
```

> **Note:** This plugin was written for Volatility2. For Volatility3, extract Chrome's SQLite history file from memory directly:

Scan for Chrome's history file:

```bash
vol -f MemoryDump_Lab2.raw windows.filescan | grep -i "chrome" | grep -i "history"
```

Dump the History SQLite file and open it:

```bash
vol -f MemoryDump_Lab2.raw -o ./lab2_output/ windows.dumpfiles --physaddr <offset>
sqlite3 lab2_output/file.*.dat "SELECT url FROM urls ORDER BY last_visit_time DESC;"
```

A **Mega.nz** link appears pointing to a folder `MemLabs_Lab2_Stage3` with a file `Important.zip`.

Download the zip. It is password-protected. The password is the **SHA1 hash** of Flag 3 from Lab 1:

```bash
echo -n "flag{w3ll_3rd_stage_was_easy}" | sha1sum
# Output: 6045dd90029719a039fd2d2ebcca718439dd100a
```

Extract the zip:

```bash
7z x Important.zip
# Password: 6045dd90029719a039fd2d2ebcca718439dd100a
```

An image is extracted containing the flag.

> **Flag 3:** `flag{oK_So_Now_St4g3_3_is_DoNE!!}`

---

### Lab 2 - All Flags

| Stage | Flag |
|-------|------|
| 1 | `flag{w3lc0m3_T0_$T4g3_!_Of_L4B_2}` |
| 2 | `flag{w0w_th1s_1s_Th3*SeC0nD_ST4g3*!!}` |
| 3 | `flag{oK_So_Now_St4g3_3_is_DoNE!!}` |

---

---

## Lab 3 - The Evil's Den

### Challenge Description
> A malicious script encrypted a very secret piece of information I had on my system. Can you recover the information for me please?  
> **Note:** This challenge is composed of only 1 flag, split into 2 parts.  
> **Hint:** You'll need the first half of the flag to get the second.  
> You will need: `sudo pacman -S steghide`  
> **Flag format:** `inctf{s0me_l33t_Str1ng}`

---

### Step 1 - Check Running Process Command Lines

```bash
vol -f MemoryDump_Lab3.raw windows.cmdline
```

Two notepad instances are open:
```
notepad.exe  →  C:\Users\hello\Desktop\evilscript.py
notepad.exe  →  C:\Users\hello\Desktop\vip.txt
```

---

### Step 2 - Dump the Files

Scan for both files:

```bash
vol -f MemoryDump_Lab3.raw windows.filescan | grep -E "evilscript|vip.txt"
```

Dump both using their offsets:

```bash
vol -f MemoryDump_Lab3.raw -o ./lab3_output/ windows.dumpfiles --physaddr <offset_evilscript>
vol -f MemoryDump_Lab3.raw -o ./lab3_output/ windows.dumpfiles --physaddr <offset_vip>
```

---

### Step 3 - Understand the Evil Script

Contents of `evilscript.py`:

```python
import sys
import string

def xor(s):
    a = ''.join(chr(ord(i)^3) for i in s)
    return a

def encoder(x):
    return x.encode("base64")

if __name__ == "__main__":
    f = open("C:\\Users\\hello\\Desktop\\vip.txt", "w")
    arr = sys.argv[1]
    arr = encoder(xor(arr))
    f.write(arr)
    f.close()
```

The script **XORs** the input with `^3`, then **Base64-encodes** it, and writes the result to `vip.txt`.

Contents of `vip.txt`:
```
am1gd2V4M20wXGs3b2U=
```

---

### Flag - Part 1 (Reverse the Encryption)

Reverse the operations: Base64 decode → XOR with 3:

```python
python3 -c "
import base64
s = 'am1gd2V4M20wXGs3b2U='
decoded = base64.b64decode(s).decode()
result = ''.join(chr(ord(i)^3) for i in decoded)
print(result)
"
```

**Output:**
```
inctf{0n3_h4lf
```

> **First half:** `inctf{0n3_h4lf`

---

### Flag - Part 2 (Steganography)

The hint says steghide is needed. Scan memory for JPEG images:

```bash
vol -f MemoryDump_Lab3.raw windows.filescan | grep -i ".jpeg"
```

Found:
```
\Device\HarddiskVolume2\Users\hello\Desktop\suspision1.jpeg
```

Dump it:

```bash
vol -f MemoryDump_Lab3.raw -o ./lab3_output/ windows.dumpfiles --physaddr <offset>
mv lab3_output/file.*.dat suspision1.jpeg
```

Use steghide to extract hidden data (passphrase = first half of the flag):

```bash
steghide extract -sf suspision1.jpeg
# Enter passphrase: inctf{0n3_h4lf
```

This extracts a file called `secret text`:

```bash
cat "secret text"
```

**Output:**
```
_1s_n0t_3n0ugh}
```

Combine both halves:

```
inctf{0n3_h4lf + _1s_n0t_3n0ugh} = inctf{0n3_h4lf_1s_n0t_3n0ugh}
```

> **Flag:** `inctf{0n3_h4lf_1s_n0t_3n0ugh}`

---

### Lab 3 - Flag

| Flag | Value |
|------|-------|
| Full | `inctf{0n3_h4lf_1s_n0t_3n0ugh}` |

---

---

## Quick Reference - Volatility3 Commands Used

| Task | Command |
|------|---------|
| OS info | `vol -f dump.raw windows.info` |
| Process list | `vol -f dump.raw windows.pslist` |
| Command lines | `vol -f dump.raw windows.cmdline` |
| Console output | `vol -f dump.raw windows.consoles` |
| Environment vars | `vol -f dump.raw windows.envars` |
| File scan | `vol -f dump.raw windows.filescan` |
| Dump files | `vol -f dump.raw -o ./out/ windows.dumpfiles --physaddr <offset>` |
| Dump process mem | `vol -f dump.raw -o ./out/ windows.memmap --pid <PID> --dump` |
| Password hashes | `vol -f dump.raw windows.hashdump` |

---

## Tools Used

- **Volatility3** - Memory analysis framework (`python vol.py`)
- **GIMP** - Reconstruct mspaint image from raw memory (`sudo pacman -S gimp`)
- **steghide** - Steganography extraction (`sudo pacman -S steghide`)
- **KeePassXC** - Open KeePass `.kdbx` database (`sudo pacman -S keepassxc`)
- **7-Zip** - Extract password-protected zips (`sudo pacman -S p7zip`)
- **sqlite3** - Read Chrome's History database (`sudo pacman -S sqlite`)
- **unrar** - Extract RAR archives (`sudo pacman -S unrar`)
- **base64 / python3** - Decode encoded strings (built-in)

---
