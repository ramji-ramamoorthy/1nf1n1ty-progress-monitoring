# TryHackMe - Forensic Imaging

**Author** Ramji R  
**Room:** [Forensic Imaging](https://tryhackme.com/room/forensicimaging)  
**Category:** Forensics  

---

## Overview

This room covers the basics of forensic imaging -- creating an exact bit-by-bit copy of digital storage media. It walks through identifying a target device, setting up an audit trail, creating a raw image, verifying integrity via hashing, and mounting the image for inspection.

---

## Task 1 - Introduction

Forensic imaging is the process of creating an exact, bit-by-bit copy of digital storage media, capturing everything including deleted files, hidden files, and unallocated space.

Key goals:
- Preserve original data untouched
- Create a verifiable copy for investigation and legal proceedings
- Maintain the chain of custody throughout

---

## Task 2 - Preparation

### Environment

Linux is commonly used for forensic acquisition because:
- Open-source tools satisfy evidential reliability guidelines
- The Linux kernel supports a wide range of file systems

### Write-Blockers

A write-blocker is a hardware device that sits between the evidence drive and the forensics workstation, intercepting all write commands to prevent any modification to the original evidence.

### Audit Trail

Before starting, set up session logging so every command is recorded. Useful bash settings:

```bash
set -o history
shopt -s histappend
export HISTCONTROL=
export HISTIGNORE=
export HISTFILE=~/.bash_history
export HISTFILESIZE=-1
export HISTSIZE=-1
export HISTTIMEFORMAT="%F-%R "
```

You can also use the `script` tool to record the full terminal session.

### Identifying the Device

List mounted filesystems:
```bash
df
```

List all block devices including unmounted ones:
```bash
lsblk -a
```

Get loop device details:
```bash
sudo losetup -l /dev/loopX
```

Get UUID and filesystem type:
```bash
sudo blkid /dev/loopX
```

Identify the target device by its size (in this room, look for the 1.1 GB loop device).

---

## Task 3 - Creating a Forensic Image

### Common Imaging Tools

| Tool | Description |
|------|-------------|
| `dd` | Standard Unix copy utility, creates raw disk images |
| `dc3dd` | Enhanced `dd` with hashing and logging for forensics |
| `ddrescue` | Data recovery focused, handles damaged drives |
| `FTK Imager` | GUI-based, widely used in professional forensics |
| `Guymager` | GUI-based, supports multiple image formats |
| `ewfacquire` | Creates Expert Witness Format (EWF) images |

### Creating an Image with dc3dd

```bash
sudo dc3dd if=/dev/loopX of=output.img log=imaging_log.txt
```

- `if` = input file (the source device)
- `of` = output file (the image file to create)
- `log` = saves all output to a log file

Verify the image was created with the correct size:
```bash
ls -alh output.img
```

---

## Task 4 - Integrity Checking

After imaging, verify the copy matches the original using hash comparison. If the hashes match, the image is a perfect copy and has not been altered.

### Hashing with md5sum

Hash the image file:
```bash
sudo md5sum output.img
```

Hash the original device:
```bash
sudo md5sum /dev/loopX
```

Both outputs should be identical.

Common hash algorithms used in forensics: MD5, SHA-1, SHA-256.

`dc3dd` can also generate hashes automatically during the imaging process using the `hash` parameter.

---

## Task 5 - Other Types of Imaging

Forensic imaging is not limited to physical disks:

| Type | Description |
|------|-------------|
| Remote Imaging | Acquire data over a network without physical access |
| USB Images | Image the full contents of a USB drive |
| Docker Images | Snapshot of a container's filesystem and configuration |

### Mounting an Image

To verify an image is valid and inspect its contents:

```bash
# Create a mount point
sudo mkdir -p /mnt/mountpoint

# Mount the image
sudo mount -o loop image.img /mnt/mountpoint

# Browse the contents
ls /mnt/mountpoint
```

---

## Task 6 - Practical Exercise

SSH credentials for this task:
- **Username:** practical
- **Password:** forensics

Steps to complete:
1. Use `lsblk -a` to identify the 1 GB loop device
2. Use `dc3dd` to create an image of it
3. Run `md5sum` on the image for the hash answer
4. Mount the image and read `flag.txt` for the flag answer

---
