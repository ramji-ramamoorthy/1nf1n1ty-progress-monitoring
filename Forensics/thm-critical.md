# TryHackMe: Critical

**Author** Ramji R  
**Category:** Forensics     
**Difficulty:** Easy    
**Tool:** Volatility 3    

---

## Environment Setup

The memory dump `memdump.mem` was located at `/home/analyst/`. Volatility 3 was pre-installed with an alias `vol` pointing to it.

To list available Windows plugins:

```bash
vol windows --help
```

To get help:

```bash
vol -h
```

---

## Task 2: Memory Forensics Concepts

Memory forensics involves analyzing volatile memory (RAM) from a compromised machine. Unlike disk forensics, it captures running processes, active network connections, and in-memory artifacts that may not exist on disk.

Two main phases:

1. **Memory Acquisition** -- copying live RAM to a dump file (`.mem`, `.dmp`, `.raw`)
2. **Memory Analysis** -- examining the dump with tools like Volatility

The dump here was acquired using FTK Imager on the Windows target, then transferred to a Linux machine for analysis.

**Q: What type of memory is analyzed?**  
`RAM`

**Q: In which phase do you create a memory dump?**  
`Memory Acquisition`

---

## Task 3: Environment and Setup

**Q: Which plugin gets OS info?**  
`windows.info`

**Q: Which tool takes a memory dump on Linux?**  
`LiME`

**Q: Command to display the help menu?**  
`vol -h`

---

## Task 4: Gathering Target Information

Ran `windows.info` to get basic system details:

```bash
vol -f memdump.mem windows.info
```

Key output fields:

| Field | Value |
|---|---|
| Kernel Base | `0xf9066161c000` |
| Is64Bit | `False` |
| NtMajorVersion | `10` |
| SystemTime | `2024-02-24 22:52:52` |
| NtSystemRoot | `C:\Windows` |

The machine is running **Windows 10 (build 19041)**, 32-bit mode, with 2 processors.

**Q: Is the architecture x64?**  
`N`

**Q: Windows OS version?**  
`10`

**Q: Kernel base address?**  
`0xf9066161c000`

---

## Task 5: Searching for Suspicious Activity

### Network Activity

```bash
vol -f memdump.mem windows.netscan
```

Found a connection established on port `3389` from `192.168.182.139` at `2024-02-24 22:47:52`, suggesting possible RDP-based initial access.

Also found an outbound connection on port `80` to `77.91.124.20`, owned by `updater.exe`. That is not a standard Windows process and immediately looks suspicious.

**Q: Destination IP on port 80?**  
`77.91.124.20`

**Q: Program accessing port 80?**  
`updater.exe`

### Process Analysis

```bash
vol -f memdump.mem windows.pstree
```

Spotted a process with a truncated name `critical_updat` that is not part of standard Windows processes. It is the parent of `updater.exe`.

```
critical_updat  PID: XXXX   Started: 2024-02-24 22:24:40
  updater.exe   PID: 1612
```

Neither of these appear in the normal Windows process tree (explorer.exe, svchost.exe, lsass.exe, etc.), so both are flagged as suspicious.

**Q: PID of the child process of critical_updat?**  
`1612`

**Q: Timestamp for critical_updat?**  
`2024-02-24 22:24:40`

---

## Task 6: Finding Interesting Data

### Locating Files on Disk

```bash
vol -f memdump.mem windows.filescan > filescan_out
cat filescan_out | grep -i critical_updat
```

Found the full path:

```
\Users\user01\Documents\critical_update.exe
```

**Q: Full path and name for critical_updat?**  
`\Users\user01\Documents\critical_update.exe`

### MFT Timestamps for important_document.pdf

```bash
vol -f memdump.mem windows.mftscan.MFTScan > mftscan_out
cat mftscan_out | grep important_document
```

MFTScan returns four timestamps per file: Created, Modified, Updated (MFT), Accessed.

**Q: Created timestamp for important_document.pdf?**  
`2024-02-24 22:09:48`

### Dumping and Analyzing updater.exe Memory

```bash
vol -f memdump.mem -o . windows.memmap --dump --pid 1612
```

This produced `pid.1612.dmp`. Ran strings on it and searched for patterns:

```bash
strings pid.1612.dmp | less
strings pid.1612.dmp | grep -B 10 -A 10 "http://key.critical-update.com/encKEY.txt"
```

Found a hardcoded URL in memory:

```
http://key.critical-update.com/encKEY.txt
```

The HTTP response in memory contained the encryption key value: `cafebabe`

Also found `important_document.pdf` referenced in the strings output, confirming `updater.exe` interacted with the file and fetched the key from a remote C2-like server.

The HTTP response headers revealed the server software used by the attacker:

**Q: Server used by the attacker?**  
`Apache`

---
