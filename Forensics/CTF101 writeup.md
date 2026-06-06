# CTF Forensics Handbook Notes

**Author:** Ramji R  
**Platform:** CTF 101 Forensics Handbook  
**Category:** Forensics  
**Date:** June 2026

---

## Overview

Forensics in CTF challenges involves recovering, analyzing, and interpreting data from files, memory, disk images, and network captures. The core skill is knowing what to look for and which tool to reach for. These notes cover the fundamentals I worked through from the CTF 101 handbook.

---

## 1. File Formats and File Signatures

### What are File Signatures?

File extensions are not reliable identifiers for file types. The real identifier is the **file signature**, also called the **magic number**. These are specific bytes located at the very beginning of a file (usually the first 2-4 bytes) that tell programs what format the data is in.

For example, a JPEG file always starts with the bytes `FF D8 FF E0` in hex, or `ÿØÿà` in ASCII representation.

### Why This Matters in CTF

Challenge authors will often give you a file with a wrong or missing extension, or a file whose header has been deliberately corrupted. Before running any forensics tool on an unknown file, the first step is always to verify its actual type by checking the magic bytes.

### How to Find the File Signature

Open the file in a hex editor or use `xxd` or `hexdump` in the terminal and look at the first few bytes. Cross-reference against a file signature repository like Gary Kessler's database.

```bash
xxd file | head
file file-a.jpg
```

The `file` utility in Linux uses magic number detection internally, so it is a quick sanity check. If the extension and `file` output disagree, trust the hex.

---

## 2. Metadata

### What is Metadata?

Metadata is data embedded inside a file that describes the file itself rather than the main content. This includes things like the author of a document, GPS coordinates embedded in a photo, creation and modification timestamps, camera model, software used, and more.

### Why This Matters in CTF

Flags or clues are frequently hidden inside metadata fields. A challenge image that looks completely normal might contain GPS coordinates pointing to a location, or a comment field with an encoded flag.

### Tools

`exiftool` is the go-to utility for reading metadata from almost any file type.

```bash
exiftool challenge.jpg
exiftool -GPS* challenge.jpg   # filter for GPS fields only
```

For documents, `strings` can sometimes surface embedded metadata that other tools miss.

---

## 3. Steganography

### What is Steganography?

Steganography is the practice of hiding data inside other data. Unlike encryption, which makes data unreadable, steganography hides the fact that a message exists at all. Common carriers are images, audio files, and video.

### Detecting Steganography

Visual inspection rarely works. The usual approach is to compare two visually identical files and check whether they were modified at different times. If `FileA` and `FileB` look the same but `FileB` was modified after it was supposedly created, it is a candidate for containing hidden data.

### LSB Steganography

The most common technique in CTF image steganography is **Least Significant Bit (LSB)** encoding.

Every byte in an image has 8 bits. The most significant bit (MSB) has the highest impact on the value, the least significant bit (LSB) has the lowest. Flipping the LSB of a pixel value changes it by only 1 out of 255, which is visually imperceptible.

```
10101100  =  172
10101101  =  173   (LSB flipped, 1-unit change, invisible to eye)
```

By replacing the LSB of each pixel's colour channel with one bit of a secret message, you can encode a full message across an image with no visible distortion.

**Decoding:** Collect the LSB from each pixel value in order, group them into bytes, and convert to ASCII.

### Tools

```bash
steghide extract -sf challenge.jpg    # extract with passphrase
zsteg challenge.png                   # LSB analysis for PNGs
stegsolve                             # visual bit-plane analysis
strings challenge.jpg                 # quick check for embedded text
```

---

## 4. Disk Imaging

### What is a Disk Image?

A disk image is a bit-for-bit copy of a storage device. It captures everything: the filesystem, files, deleted file remnants, slack space, partition tables, and unallocated regions. In CTF forensics challenges, you are usually given a `.dd`, `.img`, or `.raw` file representing such an image.

### Basic Workflow

The first step with any disk image is mounting it and understanding its structure.

```bash
file challenge.img                        # identify the image type
fdisk -l challenge.img                    # view partition table
mount -o loop,offset=<bytes> challenge.img /mnt/challenge   # mount a partition
```

For deeper analysis, tools like **Autopsy** or **FTK Imager** provide a GUI to browse the filesystem, recover deleted files, and search for strings across the entire image including unallocated space.

### File Carving

When files are deleted, only the directory entry is removed. The actual data often remains in unallocated space until overwritten. **File carving** recovers these files by scanning raw bytes for known file signatures regardless of the filesystem structure.

```bash
foremost -i challenge.img -o output/
binwalk -e challenge.img                  # also extracts embedded files
```

### Useful Areas to Investigate

- Slack space (padding between end of file data and end of cluster)
- Deleted files in unallocated clusters
- Hidden partitions
- Filesystem journal entries that may retain traces of old file content

---

## 5. Memory Forensics

### What is a Memory Dump?

A memory dump (RAM image) is a snapshot of a computer's volatile memory at a point in time. It contains running processes, open network connections, decrypted data that may not exist on disk, command history, and much more. Memory forensics in CTF typically involves a `.raw` or `.mem` file.

### Volatility Workflow

**Volatility** is the standard tool for memory analysis. The general workflow is:

1. Run `strings` on the image for quick surface-level clues
2. Identify the OS profile using `imageinfo`
3. List processes and look for anything suspicious
4. Dump memory from interesting processes
5. Extract relevant artefacts (files, registry, network state, etc.)

```bash
# Step 1: identify the profile
python vol.py -f image.raw imageinfo

# Step 2: list processes
python vol.py -f image.raw --profile=Win7SP0x64 pslist
python vol.py -f image.raw --profile=Win7SP0x64 pstree

# Step 3: dump a process by PID
python vol.py -f image.raw --profile=Win7SP0x64 memdump -p 2019 -D dump/

# Step 4: other useful commands
python vol.py -f image.raw --profile=Win7SP0x64 connections   # network connections
python vol.py -f image.raw --profile=Win7SP0x64 cmdscan       # commands run in cmd.exe
```

### What to Look For

Suspicious parent-child process relationships, processes with unusual names or paths, processes running from temp directories, unexpected network connections from known benign processes, and clipboard or notepad data that might contain flag material.

---

## 6. Hex Editors

### What is a Hex Editor?

A hex editor lets you view and edit the raw binary data of any file. It shows the hexadecimal representation of each byte alongside its ASCII character equivalent. This is essential for verifying file signatures, patching corrupt headers, and spotting embedded data that would be invisible through a normal file viewer.

### What to Look For

- The first few bytes (magic number / file signature)
- Human-readable strings scattered through binary data
- Suspiciously large blocks of null bytes or repeated patterns
- Embedded files signalled by a secondary file signature appearing mid-file

### Tools

- `xxd` / `hexdump` for quick terminal inspection
- `hexcurse`, `hexedit` for terminal editing
- `hexyl` for a cleaner terminal view with colour coding
- **010 Editor** or **ImHex** for GUI-based analysis with structure templates

```bash
xxd challenge.bin | head -20           # view first 20 lines
xxd challenge.bin | grep -i "flag"     # search for flag-like strings
hexedit challenge.bin                  # interactive terminal editor
```

A common CTF task is receiving a file where the magic bytes have been replaced or zeroed out, and the fix is simply patching the first few bytes back to the correct signature using a hex editor.

---

## Summary

| Topic | Core Concept | Key Tools |
|---|---|---|
| File Formats | Magic bytes identify true file type | `file`, `xxd`, Gary Kessler DB |
| Metadata | Hidden context data inside files | `exiftool`, `strings` |
| Steganography | Data hidden inside carrier files | `steghide`, `zsteg`, `stegsolve` |
| Disk Imaging | Bit-for-bit copy of storage media | `Autopsy`, `foremost`, `binwalk` |
| Memory Forensics | RAM snapshots containing live process data | `Volatility`, `strings` |
| Hex Editors | Raw binary view and editing | `xxd`, `hexedit`, `ImHex` |
