## Eavesdrop
 
**File:** `capture_flag.pcap`
**Flag:** `picoCTF{nc_73115_411_dd54ab67}`
 
### Overview
 
This challenge involved a packet capture described as containing both a chat conversation and a file transfer. The goal was to eavesdrop on the conversation, extract whatever was being transferred, and retrieve the flag.
 
### Analysis
 
I opened `capture_flag.pcap` in Wireshark and looked at the protocol breakdown. The capture had TCP traffic on two different ports: 9001 and 9002. I started with port 9001.
 
I right-clicked a packet on port 9001 and selected "Follow TCP Stream." The stream showed a plaintext chat between two people:
 
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
 
This gave me everything I needed: the encryption algorithm (DES3), the command format, and the password (`supersecretpassword123`). The file was sent over port 9002.
 
I closed that stream and followed the TCP stream on port 9002 instead. This one was binary, not text. I could see the raw data starting with `Salted__`, which is the standard OpenSSL magic header that appears at the beginning of any file encrypted with a salted symmetric cipher.
 
To extract the encrypted file, I used tshark to pull the raw payload bytes from the port 9002 stream:
 
```bash
tshark -r capture_flag.pcap -T fields -e data 2>/dev/null
```
 
I wrote a small Python script to parse the hex output and save any blob starting with the `Salted__` magic bytes to disk:
 
```python
import binascii
 
hex_data = "<hex output from tshark>"
raw = binascii.unhexlify(hex_data)
if raw.startswith(b'Salted__'):
    with open('file.des3', 'wb') as f:
        f.write(raw)
```
 
With `file.des3` saved, I ran the exact OpenSSL command I had found in the chat:
 
```bash
openssl des3 -d -salt -in file.des3 -out file.txt -k supersecretpassword123
```
 
Then I read the output:
 
```bash
cat file.txt
```
 
Output:
 
```
picoCTF{nc_73115_411_dd54ab67}
```
 
### Summary
 
The chat session on port 9001 leaked the decryption key and the exact command needed to use it. The encrypted file transferred on port 9002 was then trivial to extract and decrypt. This challenge is a good demonstration of why sending encryption keys in plaintext over the same medium as the encrypted data completely defeats the purpose of encryption.
 
