# Wireshark Full Course Notes (0:00 to 2:05:31)

**Author:** Ramji R  
**Source:** [Alpha Brains Courses - Wireshark Full Course 2023](https://www.youtube.com/watch?v=n4muxtqLhN4)  
**Published:** November 15, 2023  
**Coverage:** Timestamps 0:00:00 to 2:05:31  

---

## 1. Introduction and Overview 

- Welcome and course overview
- What will be covered throughout the course
- Getting Wireshark -- download and install

---

## 2. Getting Traffic 

### Switches vs Hubs
- Hubs broadcast traffic to all ports -- easy to capture
- Switches only send traffic to the intended port -- need extra steps to capture others' traffic

### Spoofing to Obtain Traffic
- ARP spoofing / MAC flooding techniques to intercept traffic on switched networks
- Positioning your machine as a man-in-the-middle

---

## 3. Capturing and Viewing Traffic 

### Starting a Packet Capture
- Selecting the right network interface
- Starting and stopping captures

### Capture Options 
- Promiscuous mode
- Capture filters (set before capture begins)
- Ring buffer settings

### Capturing Wireless Traffic 
- Monitor mode for Wi-Fi
- Capturing 802.11 frames

### Using Filters 
- Display filters vs capture filters
- Common filter syntax: `ip.addr`, `tcp.port`, `http`, `dns`, etc.

### Sorting and Searching 
- Sorting columns (time, source, destination, protocol, length)
- Using Find Packet (Ctrl+F) to search by string, hex, regex

### Viewing Frame Data 
- Packet list pane, packet details pane, packet bytes pane
- Expanding protocol layers: Ethernet > IP > TCP > HTTP

### Changing the View 
- Customizing columns
- Coloring rules for quick visual identification
- Time display format options

---

## 4. Streams 

- Following TCP stream reassembly -- see full conversation between client and server
- UDP stream, HTTP stream, TLS stream
- Useful for extracting credentials sent over plaintext (HTTP, Telnet, FTP)
- Stream dialog box and saving stream data

---

## 5. Using Dissectors 

- Dissectors decode protocol-specific data inside packets
- Wireshark auto-detects protocols using port numbers and packet structure
- Manual dissector assignment: right-click > "Decode As" for non-standard ports
- Custom dissectors for proprietary protocols

---

## 6. Name Resolution 

- Resolving IP addresses to hostnames (DNS reverse lookup)
- MAC address resolution to manufacturer names (OUI lookup)
- Port number resolution to service names
- Enabling/disabling name resolution to reduce noise or improve readability
- Caveat: name resolution adds DNS traffic -- can skew captures

---

## 7. Saving Captures 

- Saving as `.pcap` or `.pcapng` format
- `.pcapng` is newer, supports multiple interfaces and metadata
- Saving only filtered/displayed packets vs all captured packets
- File compression options

---

## 8. Capturing From Other Sources 

- Importing captures from remote machines
- Using `tcpdump` on a remote host and piping output to Wireshark locally
- Capturing from virtual interfaces, loopback, VPNs
- USB traffic capture

---

## 9. Opening Saved Captures 

- Opening `.pcap` / `.pcapng` files
- Working with pre-captured traffic for offline analysis
- CTF relevance: analyzing provided pcap files in forensics/network challenges

---

## 10. Using Ring Buffers in Capturing 

- Ring buffer = automatically rotate capture files after a set size or time
- Prevents disk overflow during long captures
- Configure number of files and file size/duration limit
- Useful for continuous monitoring scenarios

---

## 11. Expert Analysis 

### Expert Information Panel
- Wireshark auto-flags anomalies: errors, warnings, notes, chats
- Categories: Checksum errors, retransmissions, resets, window issues

### Locating Errors 
- TCP retransmissions, duplicate ACKs, RST packets
- Malformed packets
- Zero window / window full conditions

---

## 12. Applying Dynamic Filters 

- Right-click any field in packet details > "Apply as Filter" or "Prepare as Filter"
- Building filters on the fly without memorizing syntax
- AND/OR chaining filters dynamically

---

## 13. Filtering Conversations 

- Isolating traffic between two specific endpoints
- `ip.addr == x.x.x.x && ip.addr == y.y.y.y`
- Conversation filter from right-click context menu
- Filtering by stream index

---

## 14. Investigating Latency 

- Identifying high RTT (Round Trip Time) between client and server
- Using TCP delta times to find slow segments
- Distinguishing network latency vs server processing delay
- SYN > SYN-ACK timing analysis

---

## 15. Time Deltas 

- Adding a "Time Delta from previous displayed packet" column
- Spotting gaps/spikes in packet timing
- Useful for diagnosing intermittent connectivity or application hangs

---

## 16. Detailed Display Filters 

- Deep dive into filter syntax:
  - Comparison operators: `==`, `!=`, `>`, `<`, `contains`, `matches`
  - Combining: `&&`, `||`, `!`
- Filtering by protocol field values e.g. `http.response.code == 200`
- Saving frequently used filters as bookmarks

---

## 17. Locating Response Codes 

- Filtering HTTP response codes: 200, 301, 403, 404, 500
- Identifying failed requests and server errors
- `http.response.code == 404` style filters
- Spotting unusual response patterns that may indicate attacks

---

## 18. Using Expressions in Filters 

- Wireshark's Expression builder GUI
- Browsing all available protocol fields
- Building complex filters without manually knowing field names
- Regex matching with `matches` operator

---

## 19. Locating Suspicious Traffic 

- Identifying port scans (many SYN packets, no follow-up)
- Unusual protocols or ports
- Large data exfiltration transfers
- Beaconing patterns (regular interval connections -- C2 indicator)
- DNS tunneling signatures

---

## 20. Expert Information Errors 

- Revisiting the Expert Information window in depth
- Filtering by severity: Error > Warning > Note > Chat
- TCP issues: retransmissions, out-of-order, fast retransmit, spurious retransmit
- Using expert info to quickly triage a large capture file

---

## 21. Obtaining Files 

- Reassembling files transferred over the network from packet captures
- HTTP object extraction
- Carving files from raw TCP stream data
- Relevance to malware analysis: extracting dropped payloads from pcaps

---

## 22. Exporting Captured Objects 

- File > Export Objects menu
  - Supported protocols: HTTP, SMB, TFTP, IMF, DICOM
- Extracting files (images, executables, documents) transferred over HTTP
- Saving exported objects to disk for further analysis
- Key forensics technique -- used heavily in CTF network forensics challenges

---

*Notes end at 2:05:31*
