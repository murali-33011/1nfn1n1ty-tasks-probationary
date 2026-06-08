# Wireshark YT Notes

## What is Wireshark

Wireshark is a packet capture and analysis tool. It grabs all network communications passing across your network interface, not just the ones meant for you. Once data hits the wire, ethernet or wireless, it cannot be altered. That is what makes this tool reliable. Applications can lie, logs can be manipulated, but raw traffic on the wire is what it is.

It used to be called Ethereal. It does the binary to English translation for you so you are not staring at raw hex trying to figure out what is happening. It also does deep protocol decoding which means it pulls apart every layer and explains it in a readable way. Open source so the source code is available.



## Switch vs Hub and Why It Matters for Captures

This is important to understand before you even start capturing.

On a hub, when laptop A sends a message to laptop B, the hub sends it out every single interface. Laptop C gets it too. Everyone does. That makes capturing easy because everything is already coming to you.

On a switch, traffic is only sent to the intended recipient. So if A talks to B, C gets nothing. This is the problem. If you are on a switched network and you want to see traffic that is not meant for you, you have two options.

**Port span / port mirroring** is where you configure the switch so all traffic gets copied to a specific port that you are listening on. You need admin access to the switch for this. Home switches usually do not support it at all.

**ARP spoofing** is the other way. You pretend to be other devices so traffic routes to you instead. Tools for this are Ettercap and arpspoof (comes with dsniff package by Doug Song). Ettercap has a GUI, you scan for hosts, add them to target groups, then run ARP poisoning from the man in the middle menu. You want your gateway in a different target group from your hosts so you capture both ends of the conversation.

If you are only trying to capture your own traffic, you do not need any of this. ARP spoofing is only needed when you are on a switched network, you have permission to be doing this, and you cannot set up a port span.



## Frames vs Packets

This tripped me up. What Wireshark shows you are technically frames, not packets.

A frame is at Layer 2 (Ethernet). Frames are what your network interface actually sees and captures. At Layer 3 you have packets (IP). At Layer 4 you have segments (TCP) or datagrams (UDP). So technically you are capturing frames and those frames can contain packets which contain segments and so on. Wireshark uses "packet" loosely to refer to everything, but technically each row in the capture list is a frame.



## Capture Options

Go to capture options before starting if you want to configure things. You can find it in the toolbar or the capture menu.

What you can configure:

**Interfaces** : you can show or hide specific interfaces. If you have a bunch of virtual interfaces you probably want to hide the ones you do not care about. Go to manage interfaces to check/uncheck them.

**Output** : you can choose between pcap and pcapng format. pcap is the older format but has wider compatibility across different tools. pcapng is the Wireshark next generation format. If you are sharing the file with other tools, stick with pcap.

**Options** : you can resolve network names (IP to hostname via DNS), resolve transport names (port numbers to protocol names like 22 becomes SSH), and resolve MAC addresses (shows vendor name). These all slow down the capture because lookups take time, especially DNS. You can leave them off during capture and turn them on when you review the file later.

**Promiscuous mode** : enabled by default on all interfaces. This is what lets you capture traffic not specifically addressed to you.



## Capturing Wireless Traffic

By default you only get the ethernet headers even on WiFi because WiFi still uses ethernet at Layer 2 for MAC addresses. But if you also want the radio headers (the 802.11 stuff), you need to enable monitor mode.

In capture options, find your Wi-Fi interface. Set monitor mode to enabled. Set the link layer header to per packet information. Now you will see 802.11 frames, beacon frames from nearby access points, association requests and responses, all the low level wireless management frames.



## Capture Filters

Set these before you start capturing. They limit what gets captured in the first place.

How to set one: in the capture options page or in the main window before starting, type your filter in the capture filter box. The box turns green when the filter is valid. If you have no interface selected it stays red even if the syntax is right.

Examples:

```
host 172.30.42.13        captures anything with this IP as source or destination
port 80                  only traffic on port 80
```

If you start typing, Wireshark will suggest filters. You do not have to memorise all the syntax.

If you forget to set one, no big deal. You can still use display filters after the fact to narrow down what you are looking at. Capture filters just reduce the raw file size.



## Sorting and Searching

Once you have a capture you can sort by clicking any column header. Sort by protocol and you can scroll through and see everything grouped. Protocol colours make it easier: HTTP is green, DNS is blue, ARP you can see grouped together.

To search: go to Edit then Find. You can search the packet list or the packet details. You can search by string or hex value. Narrow (8 bit, standard ASCII) or wide (16 bit Unicode) character sets. Type what you want and Wireshark jumps to the first match.



## Viewing Frame Data

When you click on a frame in the top pane you get two more things:

The middle pane shows the frame broken down layer by layer. This is the important part. Wireshark pulls every header apart and labels everything for you. From top to bottom you typically see:

- Frame info (size in bytes and bits)
- Ethernet header (Layer 2, source and destination MAC, type)
- IP header (Layer 3, source and destination IP, TTL, protocol, fragmentation flags)
- Transport header (Layer 4, TCP or UDP, source and destination port, length)
- Application layer (HTTP, DHCP, whatever it is)

Each layer tells you what the next layer is going to be. So the Ethernet header says the type is IPv4, meaning the next header is IP. The IP header says the protocol is UDP, so the next header is UDP. You just follow the chain down.

The bottom pane is the raw hex on the left side and ASCII on the right. Not useful for most things since Wireshark already decoded it above.

If you click on a field in the middle pane, the corresponding bytes in the bottom pane get highlighted.



## Changing the View

View menu gives you all the layout options.

You can toggle the main toolbar, the filter toolbar, the packet list pane, the packet details pane, and the packet bytes pane. Hiding panes gives you more screen space.

You can drag column headers to reorder them. Put destination before source, move length, whatever makes sense for your workflow. To get back to original order just click the frame number column header to re-sort by frame number.

You can zoom in if you need bigger text.

Coloring rules can be changed. You can also colorize specific conversations to make them stand out in a busy capture.

The info column on the far right is a quick summary Wireshark generates for each frame. It is not from the frame itself, Wireshark creates it by pulling out the most relevant bits. Making this column wider is often worth doing because it has a lot of useful at-a-glance information.



## Streams

Whenever two systems communicate back and forth they form a conversation. Wireshark tracks these as streams and assigns each one a stream number.

To follow a stream: right click on any frame in the conversation and select Follow TCP Stream (or UDP Stream). A new window opens showing the entire conversation in one place. The red text is what you sent, blue is what the server sent back.

As you click through the conversation in the stream window, Wireshark highlights the corresponding frame in the capture list behind it.

When you close the stream window, Wireshark automatically applies a display filter like `tcp.stream eq 520` where 520 is the stream number it assigned to that conversation. You can clear this by clicking the X next to the filter box to go back to the full capture.

This is really useful for web traffic because instead of stitching together individual TCP segments manually, you just see the whole HTTP conversation laid out cleanly.



## Using Dissectors

Wireshark identifies protocols in the protocol column. Sometimes it gets it wrong, especially when traffic is on a non-standard port. For example if an HTTP server is running on port 50360 instead of 80, Wireshark might not recognise it.

To fix this: right click on the frame and select Decode As. A dialog appears where you can tell Wireshark what protocol to use for decoding that traffic. You can set rules for specific port numbers or IP protocols.

You can also write custom dissectors if you have completely custom packet formats. This is advanced but the capability is there.



## Name Resolution

By default you see raw IP addresses and port numbers. Wireshark can look these up for you.

Go to View menu. You can toggle:

- Resolve Network Addresses: converts IPs to hostnames via DNS
- Resolve Transport Addresses: converts port numbers to protocol names (80 becomes HTTP, 22 becomes SSH)
- Resolve MAC Addresses: converts MAC prefixes to vendor names

Warning: if you enable network name resolution while capturing, Wireshark generates DNS lookups for every IP it sees. This creates extra DNS traffic in your capture that was not originally there. Better to leave it off during capture and enable it afterwards when you are reviewing the file.



## Saving Captures

File menu, then Save. You get a choice of format.

**pcap** (Wireshark/tcpdump format) is the safest choice if you want the file to work across different tools. Wide compatibility.

**pcapng** is the newer Wireshark format with more features but some tools do not support it.

To save only a subset, use Export Specified Packets. You can export:

- All captured packets
- Only displayed packets (whatever is showing through your current display filter)
- Selected packets only

This is useful if you have filtered down to just the frames you care about and want to save a smaller file.

You can also export packet dissections as plain text, CSV, or C arrays if you want the decoded data in a different format.



## Capturing From Other Sources (tcpdump)

Wireshark can open capture files created by other tools, not just its own captures. tcpdump is a common one.

Basic tcpdump command to save a capture:

```bash
tcpdump -w output.pcap -i en0
```

`-w` writes to file, `-i` specifies the interface.

By default tcpdump may not grab the full packet. Add `-s 0` to set the snap length to unlimited and capture the entire frame:

```bash
tcpdump -w output.pcap -i en0 -s 0
```

Without `-s 0` you might only get headers and miss the data portion. If you are trying to reconstruct web traffic or look at application data, you need the full packet.

Once saved, just open the pcap in Wireshark and analyse it there. tcpdump's command line output is hard to read, Wireshark is much better for this.



## Opening Saved Captures

File menu, then Open. Or click the open folder icon in the toolbar. Wireshark lists recently opened files on the start screen as well.

On Windows the pcap file extension creates a file association so you can double click the file in Explorer to open it. On Mac and Linux the extension matters less.

Wireshark can open pcap files from many different sources including tcpdump, Wireshark itself, and various other network tools that output the pcap format.



## Ring Buffers

If you want to run a long term capture without filling up your disk, use the ring buffer feature.

Go to Capture Options then Output. Check "create a new file automatically after X megabytes". Then enable the ring buffer and set how many files you want to cycle through.

For example: 4 files of 10 megabytes each gives you 40 megabytes of rolling capture. Once you fill the 4th file, it loops back and overwrites the first one. So you always have the most recent 40 megabytes of traffic.

Why this is useful: if something happens on your network, like an intrusion detection alert fires, you stop Wireshark and you have a window of recent traffic to look back through to see what led up to the event. You get historical context without an ever-growing capture file eating your disk.



## Analysis and Expert Information

Wireshark does automated analysis as you capture and flags problems. Look at the bottom left corner of the window. You will see a coloured circle there.

The colours indicate the highest severity issue found in the capture:

- Red circle: error, something serious
- Yellow circle: warning
- Cyan: note
- Blue: chat (informational)

Click the circle to open the Expert Information panel. This lists all the flagged issues grouped by severity. You can click on any entry and it jumps you to that specific frame in the capture.

Common things Wireshark flags:

**Out of order segments**  a TCP segment arrived out of sequence. Might indicate network issues.

**Missing acknowledgement**  an ACK that should exist is not in the capture. Could just mean it was not captured, not necessarily a real problem.

**Retransmissions**  completely normal in TCP. TCP retransmits if it does not get an ACK within a certain time window. You will see these under notes, not warnings, because they are expected.

**Duplicate ACKs**  can happen when packets are delayed in transit, you retransmit, and both end up arriving. Not always a big deal but worth noting if you see a lot of them.

**Malformed packets**  a packet that does not follow the expected protocol rules. This is worth paying attention to. Malformed packets can crash applications and may indicate something intentional.

**Checksum errors** : the checksum calculated on receipt does not match the one in the header. Means something got corrupted or altered in transit.



## Locating Errors Visually

In the packet list pane, Wireshark colour codes errors:

Black background with red text is the one to pay attention to. It stands out from everything else and indicates something that needs investigation. Duplicate ACKs, checksum failures, and similar issues show up this way.

You can spot these just by scrolling through the capture without needing to open Expert Information. When you see a black/red row, click it and look at the info column and the TCP analysis flags in the frame details.



## Display Filters

These are different from capture filters. Capture filters decide what gets recorded. Display filters decide what you see out of what was already recorded. You can change display filters as many times as you want without affecting the actual capture.

Type in the filter bar at the top. It turns green when the filter is valid, red/pink when it is not.

Basic examples:

```
ip.addr == 172.30.42.13          any frame with this IP as source or destination
ip.src == 172.30.42.13           only frames where this is the source
http                             only HTTP frames
dns                              only DNS frames
tcp                              only TCP
```

You can combine conditions with `and`, `or`, and `not`:

```
ip.addr == 172.30.42.13 and http
http or dns
not arp
```

The displayed count at the bottom shows how many frames match. The total captured count never changes.



## Filtering Conversations

Once you find a frame of interest you can filter down to just that conversation in several ways.

**Right click approach**: right click the frame, go to Apply As Filter, and choose Selected (include this), or Not Selected (exclude this). You can also do And Selected or Or Selected to add to an existing filter.

**Follow stream**: right click and Follow TCP Stream (or UDP Stream). Wireshark applies a filter like `tcp.stream eq 20` automatically so you are only looking at that conversation's frames.

**Colorize Conversation**: right click, Colorize Conversation, choose a layer (IPv4 for example). This gives all frames in that conversation a distinct colour so they stand out in the full capture without hiding everything else.



## Investigating Latency

Timestamps in Wireshark are all relative to the start of the capture. There is no absolute clock in IP or TCP headers. There is no field that says this packet was sent at 14:32:07.

The time to live field is not about actual time, it is a hop count. Do not confuse it with latency measurement.

To measure latency you calculate the time between a request and its response, which is essentially round trip time.

To make this easier:

Go to Edit then Preferences (on Mac it is Wireshark menu then Preferences). Go into Protocols, find TCP. Enable "Calculate conversation timestamps". This tells Wireshark to track the time since the first frame and the time since the previous frame within each TCP stream.

Once enabled, if you click on a TCP frame and expand the TCP header in the details pane, you will see two new fields:

- Time since first frame in this TCP stream
- Time since previous frame in this TCP stream

These show you how long individual steps in a conversation are taking.



## Time Deltas

Related to the latency section but more useful for analysis across the whole capture.

After enabling conversation timestamps (as above), find a TCP frame, expand the TCP header, right click on "Time since previous frame in this TCP stream" and select Apply As Column. This adds a column to the packet list.

Now you can sort by this column. Frames with very high time delta values are candidates for investigation because they represent significant gaps in a conversation. This is how you spot network congestion or slow server responses.

Sort descending, find the outliers, then right click and Follow TCP Stream to see the full conversation around that delay.



## Detailed Display Filters

Going deeper than the basics.

Wireshark knows about relative sequence numbers. It takes the actual random starting sequence number and calls it zero, then shows incremental values from there. So when you filter for `tcp.seq == 0` you are finding the first frame of a TCP three-way handshake (the SYN).

To find just SYN frames and not the SYN-ACK frames you can add:

```
tcp.seq == 0 and tcp.ack == 1
```

For ICMP echo requests (pings):

```
icmp.type == 8
```

For cases where you sent a ping and got no response:

```
icmp.type == 8 and !icmp.resp_in
```

For HTTP stuff:

```
http.request.method == "GET"
http.response.code == 200
http.response.code == 404
http.response.code == 302
```

To look for basic authentication (credentials in clear text):

```
http.authbasic
```

You can go really deep. Type `ip.` and Wireshark immediately suggests every field it knows about under the IP protocol. Same for `tcp.`, `udp.`, `http.` etc. This is how you discover what is filterable without memorising everything.

Boolean operators: `and`, `or`, `not`. Comparison operators: `==`, `!=`, `>`, `<`, `>=`, `<=`. String matching: `contains` for simple substring, `matches` for regex.



## Expression Builder

If you do not know the field name you want to filter on, the expression builder helps.

Click the Expression button next to the filter bar. A dialog opens with every field Wireshark knows about, organised by protocol. You can search in here. Click a field name to see a short description of what it is. Set your comparison operator. Set the value. Click OK and it populates the filter bar.

You can also chain filters from here using the And/Or buttons. Useful when you are exploring unfamiliar protocols and do not know the field syntax off the top of your head.



## Locating Response Codes

Specific to HTTP analysis.

Filter on `http.request.method == "GET"` to see all outgoing GET requests.

Filter on `http.response.code == 200` to see successful responses.

Filter on `http.response.code == 302` to find redirects. Open the HTTP header of a 302 and you will see where it redirected to.

Filter on `http.response.code == 404` for not found errors.

When you open an HTTP frame with a redirect or error, Wireshark often shows you in the info column which frame the request came from and where the response goes. It links them together which makes it easy to trace the full flow.



## Locating Suspicious Traffic

**Protocol hierarchy** is the go-to starting point. Statistics menu then Protocol Hierarchy.

This shows you every protocol found in the entire capture with packet counts and percentages of total traffic. Look for anything unexpected.

In a typical corporate network capture you expect to see Ethernet, IPv4, IPv6, TCP, UDP, HTTP, DNS, NetBIOS if there is file sharing, maybe some ICMP. If you see something like IRC (Internet Relay Chat) with a few hundred packets, that stands out. Someone on the network is connected to an IRC server. Not necessarily malicious but worth investigating.

Other things that might catch your eye: unusual tunnelling protocols, protocols you simply do not recognise, anything with a small packet count that sits in an unexpected part of the hierarchy.

Once you spot something, right click the protocol in the hierarchy window and select Apply As Filter. Wireshark closes the window and applies that protocol as a display filter so you see only those frames.



## Expert Information Errors in Detail

The Expert Information panel groups issues by type and lets you jump to specific frames. Worth doing a structured review here rather than just scrolling.

Go to Analyze menu then Expert Information, or click the coloured circle in the bottom left.

For each error or warning you can click the row to expand it and see the frame numbers. Click a frame number to jump to it in the capture.

Example from video: IRC frames showing errors about missing prefix ending with a space. This turned out to be truncated frames, packets that did not include the complete protocol data. The Kink protocol (Kerberized Internet Negotiation of Keys) had a malformed packet error, meaning the packet did not follow the rules the protocol specifies. Malformed packets can crash applications so these are worth looking into.

The expert info does not just find your errors. It also confirms expected behaviour. Retransmissions in TCP are normal. Seeing them under notes rather than warnings means Wireshark is not particularly concerned about them.



## Obtaining Files from a Capture

Everything that crossed the network during the capture is in Wireshark. If a file was transferred over HTTP during that time, it is in there.

**Export Objects**: File menu then Export Objects then HTTP. A list appears showing every file object found in the HTTP traffic: images, scripts, documents, whatever was transferred. You can see the filename, size, and which packet it was in. Click one and select Save to write it to disk.

Wireshark handles stitching together the multiple TCP frames that made up that file automatically. You just get the file.

Same thing works for SMB (Windows file sharing). File menu, Export Objects, SMB. If someone was transferring files across the network you can pull them out.

**Export Packet Bytes**: if you select a specific frame and go to File then Export Packet Bytes, you can dump the raw bytes of that packet to a file.

This has real forensic value. If you captured traffic during an incident you can recover files, images, documents that were sent across the network even if they were deleted from the endpoints afterwards.

