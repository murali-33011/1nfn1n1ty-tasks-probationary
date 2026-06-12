## Packets Primer
 
**File:** `network-dump_flag.pcap`
**Flag:** `picoCTF{p4ck37_5h4rk_ceccaa7f}`
 
### Overview
 
A small packet capture with a straightforward goal: find the flag inside. The hint pointed to Wireshark as the primary tool.
 
### Analysis
 
I opened `network-dump_flag.pcap` in Wireshark. The capture was tiny, only 9 packets total. The first three were the standard TCP three-way handshake between `10.0.2.15` and `10.0.2.4` on port 9000. Packets 6 through 9 were ARP exchanges. That left packet 4 as the only one carrying actual data.
 
I clicked on packet 4 in Wireshark and expanded the TCP segment in the packet detail pane. The payload was 60 bytes. I right-clicked the data layer and selected "Follow TCP Stream" to see it as readable text.
 
The stream view immediately showed something unusual: the content appeared to have spaces between every single character. I also extracted the raw hex data using tshark for confirmation:
 
```bash
tshark -r network-dump_flag.pcap -T fields -e data 2>/dev/null
```
 
Output:
 
```
7020692063206f204320542046207b207020342063206b20332037205f2035206820342072206b205f20632065206320632061206120372066207d0a
```
 
I decoded that in Python:
 
```python
data = '7020692063206f204320542046207b207020342063206b20332037205f2035206820342072206b205f20632065206320632061206120372066207d0a'
decoded = bytes.fromhex(data).decode()
print(decoded)
print(decoded.replace(' ', ''))
```
 
Output:
 
```
p i c o C T F { p 4 c k 3 7 _ 5 h 4 r k _ c e c c a a 7 f }
picoCTF{p4ck37_5h4rk_ceccaa7f}
```
 
The flag was sent over TCP in plaintext, with each character separated by a space byte (0x20). Removing the spaces revealed the flag.
 
### Summary
 
With only 9 packets in the capture, there was very little to analyze. Following the one TCP stream that carried payload data gave the flag directly, just formatted with spaces between every character.
 
