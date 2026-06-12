 
## FindAndOpen
 
**Files:** `flag.zip`, `dump.pcap`
**Flag:** `picoCTF{R34DING_LOKd_fil56_succ3ss_cbf2ebf6}`
 
### Overview
 
I was given a password-protected zip file and a packet capture. The challenge description said the password was hidden somewhere in the trace file, and hinted that a password cracker was not the right approach. That pointed me toward simply reading the packet contents.
 
### Analysis
 
I opened `dump.pcap` in Wireshark first to get a feel for what was in it. The capture was mostly mDNS and Chromecast-related traffic. I noticed some DNS TXT record responses in the protocol list. Following a few of those streams, I could see the data field contained some unusual-looking strings.
 
Rather than manually digging through each packet, I ran `strings` against the pcap to pull out all readable text:
 
```bash
strings dump.pcap
```
 
This immediately surfaced several interesting strings. Two stood out:
 
```
Flying on Ethernet secret: Is this the flag
iBwaWNvQ1RGe1Could the flag have been splitted?
AABBHHPJGTFRLKVGhpcyBpcyB0aGUgc2VjcmV0OiBwaWNvQ1RGe1IzNERJTkdfTE9LZF8=
```
 
The last one looked like base64. The leading bytes (`AABB...`) were packet framing noise, but the portion starting from `Ghpcy` onward was clean base64. I decoded it in Python:
 
```python
import base64
s = 'dGhpcyBpcyB0aGUgc2VjcmV0OiBwaWNvQ1RGe1IzNERJTkdfTE9LZF8='
print(base64.b64decode(s).decode())
```
 
Output:
 
```
this is the secret: picoCTF{R34DING_LOKd_
```
 
The decoded string was clearly the zip password, even though it ended mid-flag with a trailing underscore. The hint had said the flag might be split, which is why the pcap only contained part of it. I used this string directly as the password:
 
```bash
unzip -P "picoCTF{R34DING_LOKd_" flag.zip
```
 
This extracted a file called `flag`, which I then read:
 
```bash
cat flag
```
 
Output:
 
```
picoCTF{R34DING_LOKd_fil56_succ3ss_cbf2ebf6}
```
 
### Summary
 
The password was sitting in a base64-encoded DNS TXT record inside the pcap. Running `strings` against the file was enough to find it without touching a password cracker at all. Decoding the base64 gave the zip password, and the extracted file contained the complete flag.
 
