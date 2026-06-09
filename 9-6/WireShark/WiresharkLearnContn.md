## Statistics: Endpoints

Statistics menu then Endpoints.

This shows every single endpoint that communicated in the capture. Across the top you get tabs for TCP, Ethernet, IPv4, IPv6, UDP. Each tab breaks things out differently.

The ethernet tab shows MAC addresses. In that capture there were 31 separate MAC addresses meaning 31 devices on the network at some point.

IPv4 tab showed 77 addresses. IPv6 had 51.

The TCP and UDP tabs are a bit different because those protocols care about ports. So the TCP tab showed 2086 endpoints, not because there are 2086 machines, but because one IP address on multiple ports counts as multiple endpoints. Same IP showing up multiple times just means it was communicating on different ports. Same thing with UDP, 415 endpoints.

The key point: if you only care about IP addresses, go to the IPv4 or IPv6 tab. If you are looking at port-level stuff, use TCP or UDP.

Enable name resolution at the bottom of the endpoints window to get hostnames instead of raw IPs. Otherwise you are just staring at numbers.



## Statistics: Conversations

Statistics menu then Conversations.

Endpoints just gives you a list of individual addresses. Conversations is the better view because it shows you which address is talking to which other address, bidirectional. You get address A and address B.

The TCP tab shows IP address and port combinations on both sides. So you might see the same IP address appear multiple times because each separate port pairing is its own conversation row.

If you only want to look at IP address to IP address without caring about ports, switch to the IPv4 tab. There all the separate port combinations for the same two IP addresses get consolidated into one row.

Each row also shows you packets A to B, packets B to A, total packets, bytes each way, total bytes, relative start time, and duration.

You can sort by any of these columns. Useful for finding the most active talkers on your network. Sort by total bytes or total packets and the noisiest conversations rise to the top.

The reason to look at bytes instead of just packet count is that packet count and bytes do not always tell the same story. You can have more packets but fewer bytes if packet sizes are small, or fewer packets but more bytes if they are large. Both views are useful depending on what you are investigating.

If you want name resolution to work in the conversations view you have to first turn it on in the View menu under Resolve Network Addresses. The conversations window will not do it on its own.



## Statistics: IO Graph

Statistics menu then IO Graph.

This plots packets per second on the Y axis against time on the X axis. Good for seeing spikes and patterns in traffic over the duration of the capture.

By default you get two lines: all packets (black) and TCP errors (blue). You can click each one to toggle it on or off.

You can hover over any point on the graph and it tells you which packet number that corresponds to. If you click on the graph it jumps to that frame in the packet list behind the window.

Interval controls how granular the view is. Default is 1 second. You can increase it to 10 seconds or more which gives you a smoother view but less detail. Going finer gives you more detail on exactly when events happened.

You can switch between time elapsed and time of day on the X axis.

You can add more display filters as additional lines. Just type a filter in the filter field, pick a colour, and it shows up as its own line. For example you could add a line for a specific port or a specific protocol. This lets you see how different types of traffic behave relative to each other across the same timeline.

You can change the graph style for each line individually: line chart, bar chart, stacked bar. You can also change the Y axis to bytes, bits, or statistical values like max and min.

There is also a flow graph option in the statistics menu. That one is more like a ladder diagram showing the sequence of messages in a conversation rather than a rate over time view.

You can copy the graph to use in a report.



## Identifying Active Conversations

Turn on name resolution first: View menu then Resolve Network Addresses. Otherwise conversations just shows raw IPs which are harder to interpret.

Then Statistics menu then Conversations. Sort by bytes or packets descending.

What you are looking for are the systems doing the most work. In the example from the video one system had 1177 packets totalling 719 kilobytes going to something that resolved to snugglelove.com which looked like ad traffic.

Another address via Akamai had 88 packets for 315 kilobytes, while another had fewer packets but 66 kilobytes. The difference between those two tells you something. The first had many small packets. The second had fewer but larger packets. This is why sorting by bytes and sorting by packets can sometimes give you a different ranking.

You can look at this from the IP layer, the TCP layer, or the UDP layer depending on what level of detail you need. The IP tab gives you the aggregate. The TCP tab breaks it down by port.



## GeoIP Setup

Wireshark can map IP addresses to physical locations using MaxMind GeoLite databases.

To set it up:

1. Go to maxmind.com and download the GeoLite Legacy downloadable databases. You need at least the country database, the city database, and the ASN database.
2. Unzip the files once downloaded.
3. In Wireshark go to Preferences then Name Resolution. At the bottom you will see GeoIP database directories. Add the path to the folder where you unzipped the databases.
4. Go to Preferences then Protocols then find IPv4 and enable GeoIP lookups.

Private IP addresses will not have GeoIP entries. Multicast addresses will not either. For everything else you should get country, city, latitude, and longitude.



## Identifying Packets by Location

Once the GeoIP databases are loaded and enabled you can use them in display filters.

```
ip.geoip.country == "United States"
```

to only show packets going to or from US IPs.

```
!ip.geoip.country == "United States"
```

to see everything outside the US.

You can also filter by latitude:

```
ip.geoip.lat > 45
```

This would show packets where the resolved latitude is above 45 degrees. Useful if you want to quickly narrow down traffic to specific geographic regions.

In the endpoints window, once GeoIP is configured, you now see extra columns: country, AS number, city, latitude, and longitude. These are populated for any IP that exists in the MaxMind database.



## Mapping Packet Locations

In the endpoints window there is a Map button. Click it and Wireshark generates a map in your default browser with dots plotted for every IP address that had a GeoIP entry.

Each dot is clickable. Clicking a dot shows the hostname, city, packet count, and byte count for that endpoint.

The map clusters where data centers tend to be. In a US-based capture you would typically see clusters in Northern Virginia, Massachusetts, California bay area, Southern California. The middle of the country sometimes has data centers too, backup facilities and similar.

If you have international traffic it will show up globally on the map. Depends on what MaxMind databases you downloaded.



## Protocol Hierarchy

Statistics menu then Protocol Hierarchy.

This shows the entire capture broken down by protocol, starting at the frame level and working up through the stack. Every single thing captured starts as a frame, so frames are always 100%. Ethernet is also 100% because that is what Layer 2 is here. Then IPv4 underneath that, then TCP and UDP underneath IPv4, then the application protocols underneath those.

Everything is shown with percentage of packets, percentage of bytes, total packet count, total byte count.

You can sort by any column. Clicking the column header reorders by that field.

Why this is useful: it gives you a fast overview of everything in the capture without having to manually look through frames. You can see at a glance what proportion of your traffic is DNS, what proportion is HTTP, what is TCP vs UDP.

Note on SSL/TLS: it shows up in the hierarchy but Wireshark cannot decode it unless you have the encryption keys. The keys are on the endpoints, not in the capture. So SSL traffic just appears as SSL with no application layer detail underneath it.



## Locating Suspicious Traffic Using Protocol Hierarchy

In the protocol hierarchy window, right click any protocol row and select Apply As Filter to instantly filter the main capture to only those frames.

Things to look for that are unexpected:

IRC showing up on a corporate network is unusual. 279 packets of IRC traffic in the example from the video. Not necessarily malicious but worth investigating. Someone is connected to an IRC server.

Rows that just say "data" with no protocol identified above them. Those could be orphan frames or traffic Wireshark could not decode. Worth looking at.

SSL traffic going to Tor nodes. In the example there was TLS 1.2 traffic coming from a Tor node. The traffic itself is encrypted so you cannot see the content. But the fact that it is a Tor node might be significant depending on context.

The protocol hierarchy respects whatever display filter is active. So if you apply a filter and then open the protocol hierarchy, it only shows what is currently visible. Clear the filter to go back to the full picture.



## Graphing Analysis Flags

Statistics menu then IO Graph.

At the bottom of the IO graph panel there is a default entry for TCP errors with the filter `tcp.analysis.flags`. Enable that checkbox.

What this gives you is a timeline view of where errors are occurring in the capture. You can see clusters of errors at specific points in time.

Use this alongside the all packets line. Compare them:

If errors spike while traffic is also high, that is expected to some degree. Heavy traffic can cause issues.

If errors spike while traffic is low, that is more interesting. Something specific is generating errors rather than just general load.

Click on a spike in the error line on the graph. Wireshark jumps to the corresponding frames in the packet list. From there you can open Expert Information to get more detail on what those errors actually are.

This is a much faster way to locate error clusters than scrolling through thousands of frames manually looking for black rows.



## Voice Over IP: Protocols

Two main protocols for VoIP:

**SIP (Session Initiation Protocol)** is text-based and HTTP-like. Easy to read in Wireshark because you can just look at the message and understand it. An invite message looks very similar to an HTTP request. SIP handles call setup and teardown, not the actual audio.

**H.323** is a binary protocol suite. Much harder to read because the fields are encoded in binary rather than text. Wireshark decodes it for you but you would not be able to read the raw hex without help. H.323 is actually a collection of protocols and H.225 is one of the main ones inside it.

**RTP (Real-time Transport Protocol)** is used by both SIP and H.323. This is the protocol that actually carries the voice audio. PCM audio is encoded inside RTP packets and transported between endpoints.

To filter for SIP just type `sip` in the display filter. To get H.225 messages use `h225`. You can also find them through the Protocol Hierarchy by selecting Apply As Filter on the relevant row.



## Locating VoIP Conversations

Telephony menu then VoIP Calls.

This gives you a list of all detected VoIP calls in the capture. Each row shows the start and stop time, the protocol used (SIP or H.323), total packet count, and the initial speaker IP.

If you have separate start/stop times for different rows it means they are genuinely separate calls. If the times overlap or are continuous it might be one call split across entries.

From the VoIP calls window you can also check for RTP streams. There should be two RTP streams per call, one in each direction.

For the RTP streams you can run an analysis and get:

- Lost packets: frames that never arrived
- Jitter: variation in arrival timing between packets
- Latency: delay between packets

These three values are what determines call quality. Jitter makes calls sound choppy. Lost packets create gaps. High latency makes conversations feel like talking over satellite. Even a small number of lost packets per second is audible because each RTP packet is typically only 20 milliseconds of audio.

For SIP there is also a SIP statistics view (Telephony menu then SIP) which shows you counts of each response code: 100 Trying, 180 Ringing, 200 OK, etc. and some timing stats if you have enough data.

H.323 has its own equivalent under Telephony then H.225 which breaks out call proceeding, connect, and alerting messages.



## Ladder Diagrams (Flow Sequences)

Telephony menu then SIP Flows. Then select a flow and click Flow Sequence.

A ladder diagram shows you the entire call as a sequence of messages on a timeline, from top to bottom, with arrows between endpoints showing which direction each message went.

The horizontal axis represents the different devices in the call path. For SIP calls you often pass through proxies and registrars before reaching the destination. So you might see three or four vertical columns in the ladder representing each hop.

Left to right across the top are the IP addresses of each device involved. Top to bottom is time. Each rung of the ladder is one message.

A typical SIP call looks like: INVITE goes out, 100 Trying comes back, 180 Ringing comes back, 200 OK comes back when the party picks up, ACK is sent, then RTP starts flowing.

Clicking on a rung in the ladder highlights the corresponding frame in the packet capture behind the window. Useful for jumping directly to the raw packet for a specific step in the call.

This is far faster than manually tracing which frame is the INVITE and which is the 200 OK by hand.



## Extracting Audio from VoIP Captures

Telephony menu then VoIP Calls. Select a call and click Play Streams.

An RTP player opens and plays back the audio from the capture. That is it. No export required, no third party tools.

You can also export as an RTP dump file if you want to play it back in a different tool.

One limitation: if the RTP stream is encrypted you cannot do this. The audio has to be unencrypted inside the capture for this to work. If someone used SRTP (Secure RTP) then the packets are there but the content is ciphertext and Wireshark has nothing to decrypt it with unless you somehow provide the keys.



## TShark (Command Line)

TShark is Wireshark on the command line. Useful when you are working over SSH or in an environment without a GUI.

Basic capture to file:

```bash
tshark -w saved.pcap -s 0 -i en1
```

`-w` writes to file, `-s 0` sets snap length to capture the full packet, `-i` specifies the interface.

Without `-s 0` you might only get the headers and miss the actual data portion. Always use it if you want the complete frame.

While capturing, TShark outputs each frame to the terminal as it arrives. If you are writing to a file you will only see the count going up.

Press Ctrl+C to stop. Then open the pcap in Wireshark for proper analysis. TShark output on the command line is workable but Wireshark is much better for actual investigation.



## Splitting Capture Files (editcap)

editcap is a command line tool that comes with Wireshark.

To split a pcap into files of 5000 packets each:

```bash
editcap -c 5000 input.pcap split.pcap
```

The output files get a date and timestamp appended to the filename automatically. So you get `split_20230101_120000.pcap`, `split_20230101_120001.pcap` and so on.

You can also split by time interval instead of packet count using `-i seconds`.

Other things editcap can do:
- Remove duplicate packets
- Chop packet sizes (trim each packet to a specified number of bytes)
- Ignore a specific number of bytes

Useful for working with very large captures where you want to isolate a specific time window, or when you need to trim packet sizes to remove sensitive payload data.



## Merging Capture Files (mergecap)

mergecap combines multiple pcap files into one.

```bash
mergecap -w output.pcap split*.pcap
```

The `*` glob expands to all files matching that pattern. You could also list them individually if needed.

Two modes:
- Merge (default): looks at timestamps and interleaves frames in correct time order
- Concatenation (`-a` flag): just appends one file after the other regardless of timestamps

Merging is what you want if the files were created from a ring buffer or from multiple simultaneous captures that you want to combine into a coherent timeline.

The resulting file might have a slightly different size than the original because merging can drop or shift frames in edge cases. Open both in Wireshark if you want to verify.



## Capture Stop Conditions in TShark

Useful when you want to run a capture and walk away, having it stop automatically.

Stop after a packet count:

```bash
tshark -c 50000
```

Stop after a duration in seconds:

```bash
tshark -a duration:15
```

Stop after a file size (in kilobytes):

```bash
tshark -a filesize:10000
```

Stop after a number of files (when using ring buffer):

```bash
tshark -a files:4
```

You can combine conditions. For ring buffer with stop conditions:

```bash
tshark -b duration:60 -b files:5 -w capture.pcap
```

This creates a new file every 60 seconds, keeps only 5 files in the ring, and stops after 5 files.

These automations are harder to replicate in the GUI. TShark is the better tool when you need to run an unattended capture.



## Command Line Capture Filters in TShark

Same syntax as Wireshark capture filters but passed with `-f`:

```bash
tshark -f "host 172.30.42.13"
```

Only captures traffic to or from that IP.

```bash
tshark -f "port 80"
```

Only captures traffic on port 80 regardless of direction.

You can combine them:

```bash
tshark -f "host 172.30.42.13 and port 80"
```

These can be combined with any of the stop conditions or output file settings.



## Extracting Specific Fields from Captures (TShark)

Instead of getting the default output format, you can tell TShark exactly what fields to extract.

```bash
tshark -T fields -e frame.number -e ip.src -e ip.dst -f "src port 80 and host 172.30.42.13"
```

`-T fields` switches to field extraction mode. `-e` specifies each field you want. You can add as many `-e` flags as you want.

Redirect to a file:

```bash
tshark -T fields -e frame.number -e ip.src -e ip.dst > ip.txt
```

Now you have a text file with just the columns you care about. From there you can sort it, deduplicate it, or import it into Excel for further analysis.

Field names follow the same convention as display filters: `ip.src`, `ip.dst`, `tcp.srcport`, `http.request.method`, etc.



## Getting Statistics on the Command Line (TShark)

`-z` flag is for statistics. `-q` flag suppresses the packet-by-packet output so you only see the stats at the end.

Protocol hierarchy:

```bash
tshark -q -z io,phs
```

Active hosts:

```bash
tshark -q -z hosts
```

This gives you all the IP addresses seen in the capture with their resolved hostnames.

To get a list of all available statistics types, pass an invalid value and TShark will print the full list of options:

```bash
tshark -q -z io,invalid
```

Some available stats:
- `io,phs` for protocol hierarchy
- `hosts` for active host list
- `http,stat` for HTTP response code counts
- `sip,stat` for SIP message counts
- `smb,srt` for SMB service response times

The output is text-based which makes it easy to pipe into other tools or import into Excel for further manipulation.

