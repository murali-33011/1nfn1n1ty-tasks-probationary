## Lab 3: The Evil's Den
 
**Tools:** Volatility 3, steghide, python3

**Flag:** `inctf{0n3_h4lf_1s_n0t_3n0ugh}`
 
### Description of Challenge
 
The description said a malicious script encrypted something. It also told me the flag was split into two halves and that I would need the first half to get the second. That second part was interesting because it meant the first half was also a passphrase for something. The hint about steghide told me one half was hidden inside an image.
 
### Step 1: Command Lines
 
```bash
vol -f MemoryDump_Lab3.raw windows.cmdline
```
 
Two notepad instances were open: one with `evilscript.py` and one with `vip.txt`. That was the encryption script and its output. I needed to dump both.
 
### Step 2: Dumping the Files
 
```bash
vol -f MemoryDump_Lab3.raw windows.filescan | grep -E "evilscript|vip.txt"
```
 
Dumped both files using their physical offsets.
 
### Step 3: Reading the Script
 
`evilscript.py` contained:
 
```python
import sys
import string
 
def xor(s):
    a = ''.join(chr(ord(i)^3) for i in s)
    return a
 
def encoder(x):
    return x.encode("base64")
 
if __name__ == "__main__":
    f = open("C:\\Users\\hello\\Desktop\\vip.txt", "w")
    arr = sys.argv[1]
    arr = encoder(xor(arr))
    f.write(arr)
    f.close()
```
 
The script takes input, XORs each character with 3, then Base64-encodes the result and writes it to `vip.txt`. To reverse it I just had to flip those operations: decode Base64 first, then XOR with 3 again.
 
`vip.txt` contained: `am1gd2V4M20wXGs3b2U=`
 
### Step 4: Reversing the Encryption
 
```python
import base64
s = 'am1gd2V4M20wXGs3b2U='
decoded = base64.b64decode(s).decode()
result = ''.join(chr(ord(i)^3) for i in decoded)
print(result)
```
 
Output: `inctf{0n3_h4lf`
 
First half of the flag. And since the description said I would need the first half to get the second, this was also going to be the steghide passphrase.
 
### Step 5: Finding the Image
 
```bash
vol -f MemoryDump_Lab3.raw windows.filescan | grep -i ".jpeg"
```
 
Found `suspision1.jpeg` on the desktop. Dumped it.
 
### Step 6: Steghide Extraction
 
```bash
steghide extract -sf suspision1.jpeg
```
 
Entered `inctf{0n3_h4lf` as the passphrase. Extracted a file called `secret text` containing: `_1s_n0t_3n0ugh}`
 
Combined both halves: `inctf{0n3_h4lf_1s_n0t_3n0ugh}`
 
The design of this challenge was elegant. You had to get the first half before the second half was even accessible, and the first half being the passphrase meant you could not skip steps. The hint about steghide in the challenge description gave away the format for the second half but you still had to do the work to get the key for it.
 
