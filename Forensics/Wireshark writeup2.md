# Wireshark Full Course Notes (2:05:34 to 3:34:06)

**Author:** Ramji R  
**Source:** [Alpha Brains Courses - Wireshark Full Course 2023](https://www.youtube.com/watch?v=n4muxtqLhN4)  
**Coverage:** Timestamps 2:05:34 to 3:34:06 (End)

---

## 23. Statistics -- Endpoints 

- The **Statistics menu** presents captured data in a visual and structured way
- **Endpoint Statistics** (Statistics > Endpoints) lists all endpoints found in the capture

### Tabs available across the top
- TCP, Ethernet, IPv4, IPv6, UDP -- each shows endpoints for that protocol

### What endpoint statistics show
- **Ethernet tab:** MAC addresses -- shows 31 separate MAC addresses = 31 devices on the network
- **IPv4 tab:** all IPv4 addresses that communicated -- 77 addresses in the example capture
- **IPv6 tab:** all IPv6 addresses -- 51 addresses in the example
- **TCP tab:** IP + port combinations = 2086 unique TCP endpoints
- **UDP tab:** IP + port combinations = 415 unique UDP endpoints

### Key distinction
- TCP/UDP tabs = IP address AND port combinations (one IP can appear multiple times for different ports)
- If you only care about IP addresses, use the **IPv4 or IPv6 tab** -- not TCP/UDP

### Name resolution in endpoints
- Click **Name Resolution** at the bottom to resolve IP addresses to hostnames instead of raw IPs
- Port numbers also appear as numeric values (e.g. 443 = SSL/TLS web traffic)

---

## 24. Conversations 

- **Endpoints alone are one-sided** -- they just list addresses, not who is talking to whom
- **Conversations** (Statistics > Conversations) shows bidirectional communications: Address A talking to Address B

### What conversations show
- Address A, Address B, packets A to B, packets B to A, total packets, bytes each direction, total bytes
- **Relative start time** and **duration** of the conversation
- **Bit rate and byte rate** per direction

### TCP conversations
- Shows IP + port combinations -- same IP address can appear multiple times with different source ports
  - e.g. `172.30.42.8:63815` and `172.30.42.8:63811` are separate conversations even though the IP is the same

### IPv4 conversations tab
- **Consolidates all port combinations** into a single IP-to-IP row
- Use this when you only care about which IPs are talking to each other, not which ports

### Ethernet conversations
- Shows MAC address pairs (Address A, Address B)

### Sorting conversations
- Sort by any column -- packets A to B, packets B to A, bytes, duration
- Example: 1,107 total packets between two hosts for 719,000 bytes total

---

## 25. Graphing 

- Wire shark can generate graphs directly from packet captures (Statistics > IO Graph)

### IO Graph
- **Y axis:** packets per second (default)
- **X axis:** time
- Hover over the graph -- shows which packet number you are at
- Click a point on the graph -- jumps to that frame in the packet list below

### Customizing the IO Graph
- **Interval:** change granularity (e.g. 1 second, 10 seconds) -- wider interval = less granular bar/line graph
- **Time of day:** toggle between elapsed time from capture start vs actual clock time
- **Overlays:** add multiple display filters as separate lines/bars with different colors
  - Default overlays: All Packets (black) and TCP Errors (blue)
  - Add custom filters: e.g. Source Port filter as a separate colored line
- **Graph styles:** line chart, stacked bar, and more
- **Y axis options:** packets, bytes, bits, max, min, avg
- **Smoothing:** available for line charts
- **Copy graph:** export it for use in reports

### Flow Graph
- Statistics > Flow Graph -- shows a **ladder diagram** style visualization of a conversation
- Useful for visualizing sequential message exchanges

---

## 26. Identifying Active Conversations 

- Goal: find which endpoints are generating the most traffic

### Steps
1. Enable name resolution first: View > Resolve Network Addresses (must be on or hostnames won't show in conversations)
2. Go to Statistics > Conversations
3. Sort by **packets** or **bytes** to find the most active endpoints

### Packets vs Bytes -- important distinction
- High packet count + low byte count = many small packets
- Low packet count + high byte count = fewer but larger packets
- Example: 88 packets / 315 KB vs 66 packets / 315 KB -- similar bytes but very different packet counts
- **Sort by bytes** for a more accurate picture of actual traffic volume

### Perspectives available
- IP layer: which IPs are talking (ignores ports)
- TCP layer: which IP+port pairs are talking
- UDP layer: same for UDP
- Sort by packets A to B, packets B to A, bytes, duration independently

### Use case
- Identifying a node causing network congestion -- narrow down to one system or a handful of systems

---

## 27. Using GeoIP 

- GeoIP maps IP addresses to physical locations (country, city, latitude, longitude)
- More convenient than doing a `whois` lookup manually

### Setting up GeoIP databases
1. Download the **MaxMind GeoLite Legacy** databases from maxmind.com:
   - Country database (IPv4)
   - City database
   - Network numbers (ASN) database
2. Unzip the downloaded `.gz` files
3. In Wireshark: Preferences > Name Resolution > GeoIP Database Directories
4. Click **+** and add the directory where the databases are stored

### Enabling GeoIP lookups
- Preferences > Protocols > IPv4 > **Enable GeoIP lookups**

### GeoIP display filters
```
# Show only packets where IP is in the United States
ip.geoip.country == "United States"

# Show only packets where IP is NOT in the United States
!ip.geoip.country == "United States"

# Show packets where latitude is greater than 45
ip.geoip.lat > 45

# Show packets where latitude is less than 45
ip.geoip.lat < 45
```

### GeoIP in Endpoints window
- After enabling GeoIP: Statistics > Endpoints
- New columns appear: Country, AS Number, City, Latitude, Longitude
- Private IP addresses and some multicast addresses will NOT have GeoIP entries

---

## 28. Mapping Packet Locations Using GeoIP 

- Statistics > Endpoints > **Map** button -- generates a world map in your default browser
- Each dot on the map = an endpoint IP address resolved to a physical location
- Click a dot to see: hostname, city, packet count, byte count
- Clusters visible around expected data center locations:
  - Northern Virginia, Massachusetts, San Francisco Bay Area, Southern California
  - Central US -- common for backup/DR data center facilities
- Works globally if the correct MaxMind databases are installed
- Relies on database accuracy -- unlisted IPs will not appear

---

## 29. Using Protocol Hierarchies 

- Statistics > Protocol Hierarchy -- full breakdown of all protocols in the capture

### Structure
- Hierarchical tree starting from the lowest layer:
  - Frame (100% -- everything captured is a frame)
  - Ethernet (100% -- all frames are Ethernet in this capture)
  - IPv4 / IPv6
  - TCP / UDP / ICMP
  - Application layer protocols (HTTP, DNS, SSL, IRC, etc.)
  - Presentation layer (GIF, JPEG, PNG)

### Columns available
- Protocol name, % packets, packet count, % bytes, byte count, bits/s
- Click column headers to reorder by any metric

### SSL/TLS note
- Wireshark can identify SSL/TLS frames but cannot decode them
- Certificates and encryption keys are stored on the endpoints, not in Wireshark
- SSL traffic shows up in the hierarchy but content is unreadable

---

## 30. Locating Suspicious Traffic Using Protocol Hierarchies (2:37:22 - 2:41:23)

- Use protocol hierarchy to spot unexpected protocols on your network

### Example suspicious findings
- **IRC (Internet Relay Chat):** ~279 packets -- unexpected on a corporate network, suggests someone is using an IRC client
- **"Data" entries:** UDP or TCP frames Wireshark couldn't identify -- may be orphan frames or unrecognized protocols worth investigating
- **Tor nodes in TLS traffic:** TLS 1.2 traffic originating from a Tor node -- indicates someone browsing anonymously, potentially hiding activity

### Workflow
1. Statistics > Protocol Hierarchy
2. Identify unexpected protocols
3. Right-click > Apply as Filter > Selected -- instantly filters the packet list to only that protocol
4. If a display filter is already active, the protocol hierarchy updates to show only what is currently displayed
5. Clear the filter to return to the full hierarchy view

---

## 31. Graphing Analysis Flags 

- Use IO Graph to **visualize errors over time** rather than hunting through the packet list

### Steps
1. Statistics > IO Graph
2. Enable the **TCP errors** overlay (checkbox -- uses filter `tcp.analysis.flags`)
3. Also enable **All Packets** overlay to see traffic volume alongside errors

### Reading the graph
- Spikes in errors WITH high traffic = errors caused by congestion
- Spikes in errors WITH low traffic = something specific happening (worth investigating)
- Large spike in traffic with NO errors = normal high-traffic event

### Drill down from the graph
- Click a spike on the graph -- jumps to that time window in the packet list
- Then open Expert Information (Analyze > Expert Information) to see what errors occurred in that window
- Gives a fast visual triage before diving into raw packet analysis

---

## 32. Voice Over IP (VoIP) / Telephony 

- Wireshark has a dedicated **Telephony menu** for VoIP analysis
- Two main VoIP signaling protocols:

### SIP (Session Initiation Protocol)
- Text-based protocol -- very readable, similar structure to HTTP
- Used to initiate, manage, and terminate calls
- Example message: `INVITE sip:<destination_address>` -- begins a call setup
- Filter: `sip`

### H.323
- Binary protocol -- not human-readable directly, Wireshark decodes it for you
- A suite of protocols including:
  - **H.225** -- call signaling (setup/teardown)
  - **H.225 RAS** -- registration, admission, and status
- Filter: `h225`
- Wireshark parses fields like source call signal address, port, call identifier, call type
- Example raw bytes: `0a 01 03 8f` = IP address `10.1.3.143`

### RTP (Real-Time Protocol)
- Carries the actual **media (audio/video)** in both SIP and H.323 calls
- Filter: `rtp`
- Contains PCM (Pulse Code Modulation) encoded audio

---

## 33. Locating VoIP Conversations 

- Telephony menu > **VoIP Calls** -- lists all VoIP calls found in the capture
- Shows: protocol used (SIP or H.323), packet count, start time, stop time, initial speaker IP

### RTP Streams
- Telephony > **RTP Streams** -- lists individual RTP media streams
- For a normal call there should be **two RTP streams** (one per direction)
- Shows: lost packets, source/destination, SSRC

### SIP example
- Three separate VoIP conversations in the capture -- confirmed by distinct start/stop times
- Three corresponding RTP streams

---

## 34. Using VoIP Statistics 

### SIP Statistics (Telephony > SIP Statistics)
- Shows counts of each SIP message type:
  - 200 OK, 180 Ringing, 100 Trying, REGISTER, INVITE, ACK
- Setup timing: minimum, average, and maximum call setup time
- Can create a display filter directly from SIP statistics

### H.225 Statistics (Telephony > H.323 > H.225)
- Message counts: Call Proceeding, Connect (pickup), Alerting (ringing), etc.
- Select a stream > **Analyze** to get:
  - **Jitter** -- variation in packet arrival times
  - **Delta** -- time between packets
  - **Latency**
  - **Lost/dropped packets**

### Call quality indicators
- High jitter = choppy audio
- High latency = noticeable delay
- Dropped packets = missing audio chunks (even 20ms gaps are audible)

---

## 35. Ladder Diagrams / Flow Sequences 

- Telephony > SIP Flows > **Flow Sequence** -- generates a ladder diagram

### What a ladder diagram shows
- Visual time-ordered sequence of messages between call participants
- X axis = devices/nodes involved in the call
- Y axis = time (top to bottom)
- Each "rung" = one message with an arrow showing direction
- Horizontal arrows show the flow from one device to another

### Reading a SIP ladder diagram
- INVITE sent from caller
- 100 Trying returned
- 180 Ringing returned
- 200 OK returned (party picked up)
- RTP streams begin (media flow)
- ACK sent

### Port information
- Source port shown (e.g. 5061) and destination port (e.g. 5060)
- Arrow direction confirms which is source and which is destination

### Click-through behavior
- Click any message in the ladder diagram -- Wireshark selects that frame in the packet list
- Extremely useful for VoIP troubleshooting when calls bounce through multiple proxies/gateways

---

## 36. Getting Audio from VoIP Captures 

- If RTP is present in the capture, Wireshark can **play back the audio**

### Method
1. Telephony > VoIP Calls
2. Select a call > **Play Streams**
3. An RTP player pops up -- press play to hear the audio

### Export options
- Export as **RTP dump** for playback in other tools

### Limitation
- If RTP is **encrypted (SRTP)**, Wireshark cannot play it back
- Would need the encryption keys from both endpoints to decrypt -- not practical in most scenarios
- Unencrypted RTP = full audio playback available

---

## 37. Advanced -- Command Line with tshark 

- **tshark** = command line equivalent of Wireshark
- Use when you only have SSH/terminal access (no GUI)
- Similar to `tcpdump` and `snoop`

### Basic capture
```bash
# Capture and print to terminal
tshark

# Save to file (full packet, wifi interface)
tshark -w saved.pcap -s 0 -i en1

# Open saved pcap in Wireshark afterwards
# (same as opening any pcap file in Wireshark GUI)
```

---

## 38. Splitting Capture Files with editcap 

- **editcap** -- splits, filters, or modifies existing pcap files

### Common use cases
- Split a large pcap into smaller files by packet count or time
- Remove duplicate packets
- Chop/truncate packet sizes
- Ignore a number of bytes

### Split by packet count
```bash
# Split into files of 5000 packets each
editcap -c 5000 saved.pcap split.pcap

# Output files: split_<datestamp>_001.pcap, split_<datestamp>_002.pcap, etc.
# Each file = 5000 frames but different byte sizes (packet sizes vary)
```

### Split by time interval
```bash
# Split into files of N seconds each
editcap -i <seconds> saved.pcap split.pcap
```

---

## 39. Merging Capture Files with mergecap 

- **mergecap** -- combines multiple pcap files into one

### Merge vs Concatenate
- **Merge:** interleaves packets based on timestamps (correct chronological order)
- **Concatenate (`-a`):** appends files sequentially, no timestamp reordering

### Usage
```bash
# Merge all split pcap files back into one
mergecap -w new_saved.pcap split*.pcap

# The * glob expands to all matching filenames in ASCII order
# For a small number of files, list them explicitly:
mergecap -w output.pcap file1.pcap file2.pcap file3.pcap
```

### Use cases
- Combining ring buffer output files into one for full analysis
- Merging captures from multiple sources or time windows
- Reassembling files split by editcap

---

## 40. Capture Stop Conditions with tshark 

- Automate when tshark stops capturing without manual Ctrl+C

### Stop conditions (`-c` and `-a`)
```bash
# Stop after 50,000 packets
tshark -c 50000

# Stop after 15 seconds
tshark -a duration:15

# Stop after a certain file size (KB)
tshark -a filesize:<KB>

# Stop after N files (used with ring buffer)
tshark -a files:<N>
```

### Ring buffer with auto-stop
```bash
# Ring buffer: rotate files every N seconds, keep last N files, stop after M files
tshark -b duration:<seconds> -b files:<N> -a files:<M> -w output.pcap
```

### Advantage over Wireshark GUI
- Automation -- set it and walk away, capture stops automatically
- Combine with capture filters for targeted automated captures

---

## 41. Command Line Capture Filters with tshark 

- Use `-f` flag to apply a **capture filter** -- limits what gets captured (not just display)

### Syntax
```bash
# Capture only traffic involving a specific host
tshark -f "host 172.30.42.13"

# Capture only traffic on port 80 (HTTP)
tshark -f "port 80"

# Combine filters
tshark -f "host 172.30.42.13 and port 80"
```

- `host <IP>` -- matches source OR destination
- `port <N>` -- matches source OR destination port
- Combine with `and`, `or`, `not`
- Works with auto-stop conditions:

```bash
# Capture port 80 traffic, stop after 50000 packets
tshark -f "port 80" -c 50000 -w http_capture.pcap
```

---

## 42. Extracting Specific Data Fields with tshark 

- Use `-T fields` + `-e <fieldname>` to extract only the data you care about

### Syntax
```bash
# Extract frame number, source IP, destination IP from HTTP traffic
tshark -f "src port 80 and host 172.30.42.13" \
       -T fields \
       -e frame.number \
       -e ip.src \
       -e ip.dst

# Save output to a text file
tshark -f "src port 80 and host 172.30.42.13" \
       -T fields \
       -e frame.number \
       -e ip.src \
       -e ip.dst > ip.txt
```

### Post-processing the output
- Output is tab-separated text -- easy to import into Excel or process with bash
- Example pipeline:
```bash
# Extract second column (source IPs), sort, deduplicate
awk '{print $2}' ip.txt | sort | uniq
```

### Available field names
- `ip.src`, `ip.dst`, `tcp.srcport`, `tcp.dstport`, `http.request.method`, `frame.number`, etc.
- Same field names used in Wireshark display filters

---

## 43. Getting Statistics on the Command Line 

- Use `-z` flag with tshark to get statistics output (similar to Wireshark's Statistics menu)
- Use `-q` to suppress per-packet output (quiet mode -- only show the stats summary)

### List available statistics types
```bash
# Deliberately provide an invalid stat name to get the full list
tshark -q -z io,xyz
```

### Protocol hierarchy statistics
```bash
tshark -q -z io,phs
# Output: text-based protocol hierarchy tree with indentation
# Shows: ethernet > IPv6 > UDP/TCP > application protocols
```

### Active hosts list
```bash
tshark -q -z hosts
# Output: all IP addresses seen + their resolved hostnames
```

### Other available statistics (`-z` options)
| Option | Description |
|--------|-------------|
| `io,phs` | Protocol hierarchy statistics |
| `hosts` | List of active hosts |
| `http,stat` | HTTP request/response statistics |
| `sip,stat` | SIP message statistics |
| `rpc,stat` | Remote procedure call statistics |
| `smb,stat` | Server Message Block statistics |
| `radius,stat` | RADIUS statistics |

### Advantage
- Text-based output can be redirected to a file and imported into Excel for further analysis
- Useful when GUI is unavailable (SSH sessions, remote servers)

---

*some of the content in this notes were ai generated for better clarity*
