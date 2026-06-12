# picoCTF Network Forensics Writeups

This document covers the solutions for four picoCTF challenges: FindAndOpen, Packets Primer, Eavesdrop, and Torrent Analyze.

---

## 1. FindAndOpen

**Files provided:** `dump.pcap`, `flag.zip`

### Approach

The pcap contains a mix of real network traffic (mDNS/Chromecast discovery packets) and a bunch of fake "Ethernet frames" that are actually just raw ASCII text stuffed into the packet bytes. Reading every packet's raw bytes as text reveals several decoy strings repeated across many packets, like:

- "Flying on Ethernet secret: Is this the flag"
- "Could the flag have been splitted?"
- "Maybe try checking the other file"

These are red herrings designed to waste time.

### Finding the real clue

One packet (out of the ~69) stands out as a base64 blob:

```
AABBHHPJGTFRLKVGhpcyBpcyB0aGUgc2VjcmV0OiBwaWNvQ1RGe1IzNERJTkdfTE9LZF8=
```

The leading characters `AABBHHPJGTFRLKV` are junk padding. The actual base64 starts at the `V` right before `Ghpcy`, giving:

```
VGhpcyBpcyB0aGUgc2VjcmV0OiBwaWNvQ1RGe1IzNERJTkdfTE9LZF8=
```

Decoding this base64 string gives:

```
This is the secret: picoCTF{R34DING_LOKd_
```

### Cracking the zip

This decoded string is a hint towards the zip password, not the full flag. The actual password for `flag.zip` is the partial flag string itself:

```
picoCTF{R34DING_LOKd_
```

Using this as the zip password:

```bash
unzip -P "picoCTF{R34DING_LOKd_" flag.zip
cat flag
```

### Flag

```
picoCTF{R34DING_LOKd_fil56_succ3ss_cbf2ebf6}
```

---

## 2. Packets Primer

**Files provided:** `network-dump_flag.pcap`

### Approach

This pcap is small (only 9 packets) and mostly consists of TCP handshake/ARP traffic. Loading it with scapy and listing the packets shows there is exactly one packet carrying a payload, a `PA` (Push/Ack) packet sent to port 9000:

```python
from scapy.all import *
pkts = rdpcap("network-dump_flag.pcap")
for p in pkts:
    if p.haslayer(Raw):
        print(p[Raw].load)
```

### Extracting the flag

The raw payload contains the flag with a space inserted between every character:

```
p i c o C T F { p 4 c k 3 7 _ 5 h 4 r k _ b 9 d 5 3 7 6 5 }
```

Stripping the spaces (`"".join(payload.decode().split())`) gives the flag directly.

### Flag

```
picoCTF{p4ck37_5h4rk_b9d53765}
```

---

## 3. Eavesdrop

**Files provided:** `capture_flag.pcap`

### Approach

This pcap contains multiple TCP streams between `10.0.2.15` and `10.0.2.4` on ports 9001 and 9002, plus some unrelated DHCP/ARP/DNS/HTTP noise. Reassembling and reading the payloads on port 9001 reveals a plaintext chat conversation between two people:

```
Hey, how do you decrypt this file again?
You're serious?
Yeah, I'm serious
*sigh* openssl des3 -d -salt -in file.des3 -out file.txt -k supersecretpassword123
Ok, great, thanks.
Let's use Discord next time, it's more secure.
C'mon, no one knows we use this program like this!
Whatever.
Hey.
Yeah?
Could you transfer the file to me again?
Oh great. Ok, over 9002?
Yeah, listening.
Sent it
Got it.
You're unbelievable
```

This gives away both the decryption command and the password (`supersecretpassword123`).

### Extracting the encrypted file

Following the conversation, the actual file gets sent over port 9002. Pulling the raw payload from that stream shows it starts with `Salted__`, which is the standard header for an OpenSSL-encrypted file:

```python
from scapy.all import *
pkts = rdpcap("capture_flag.pcap")
for p in pkts:
    if p.haslayer(TCP) and p.haslayer(Raw) and p[TCP].dport == 9002:
        data = bytes(p[Raw].load)
        with open("file.des3", "wb") as f:
            f.write(data)
```

### Decrypting

Using the command and password leaked in the chat:

```bash
openssl enc -des3 -d -salt -in file.des3 -out file.txt -k supersecretpassword123
cat file.txt
```

### Flag

```
picoCTF{nc_73115_411_dd54ab67}
```

---

## 4. Torrent Analyze

**Files provided:** `torrent.pcap` (~60MB)

### Background

The challenge description hints that a colleague has been torrenting on the company network, and that the file name itself is the flag (`picoCTF{filename}`). A separate hint also confirms the filename ends in `.iso`.

### Approach

Searching for plain strings like `.iso` or `4:name` directly in the pcap does not turn up anything useful, because the relevant bencoded BitTorrent metadata fields are split across multiple packets/TCP segments and mixed in with binary data, so a simple `strings` or substring search misses it.

The reliable way to identify the file is through the **BT-DHT (Distributed Hash Table)** traffic. BitTorrent's DHT protocol passes around an `info_hash`, a 20-byte (40 hex character) SHA-1 hash that uniquely identifies a torrent's metadata (including the file name).

### Steps

1. Open the pcap in Wireshark and apply the display filter:
   ```
   bt-dht
   ```
   This isolates all the DHT protocol packets and filters out the unrelated DNS/HTTP/ARP noise.

2. Expand the BitTorrent protocol layer on these packets and look for the `info_hash` field. Multiple different hashes may show up since DHT traffic carries info about many torrents, not just the one of interest. Cross-referencing which hash appears most frequently (or appears in packets sourced from the local machine, `10.0.2.15`) narrows it down to the relevant one:
   ```
   e2467cbf021192c241367b892230dc1e05c0580e
   ```

3. Search this hash on a public torrent indexer / search engine (e.g. via Google or a site like Torrentz/Linuxtracker). The hash resolves to a well-known Ubuntu ISO torrent:
   ```
   ubuntu-19.10-desktop-amd64.iso
   ```

   This works because the `info_hash` is derived from the torrent's metadata (including the file name and size), so any indexer that has seen this torrent before will have it mapped to the corresponding filename, even without needing the original `.torrent` file.

### Flag

```
picoCTF{ubuntu-19.10-desktop-amd64.iso}
```
