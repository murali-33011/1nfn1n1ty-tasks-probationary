## Lab 0: Never Too Late Mister
 
**Tools:** Volatility 2
**Flag:** `flag{you_are_good_but1_4m_b3tt3r}`
 
### Description of Challenge
 
The description mentioned John being an "environmental" activist who hated Thanos. That was immediately suspicious because challenge descriptions in CTFs do not include random pop culture references for nothing. I made a note of both "environmental" and "Thanos" as likely hints and kept them in mind as I worked through the dump.
 
### Step 1: Profile Identification
 
First thing I always do is run `imageinfo` to figure out what OS the dump came from:
 
```bash
volatility -f Challenge.raw imageinfo
```
 
Got back `Win7SP1x86` as the suggested profile. Used that for every subsequent command.
 
### Step 2: Process Listing
 
```bash
volatility -f Challenge.raw --profile=Win7SP1x86 pslist
```
 
Scanned through the output and three processes caught my attention:
 
`cmd.exe` is always worth looking at because commands executed in the terminal can tell you exactly what the user was doing. `DumpIt.exe` made sense since that is how the memory dump itself was taken. `mspaint.exe` I noted for later but did not end up needing it here. `explorer.exe` was just normal desktop activity.
 
The combination of `cmd.exe` and a Python script executable told me someone ran something in the terminal right before the dump was taken.
 
### Step 3: Command History
 
Since `cmd.exe` was running I used `cmdscan` to pull any executed commands:
 
```bash
volatility -f Challenge.raw --profile=Win7SP1x86 cmdscan
```
 
This revealed a command: `C:\Python27\python.exe C:\Users\hello\Desktop\demon.py.txt`
 
A Python script called `demon.py.txt` was run on the desktop. The `.txt` extension on a Python file looked deliberate, like someone tried to disguise it. I wanted to know what it printed.
 
### Step 4: Console Output
 
To see what the script actually wrote to the terminal I used `consoles`:
 
```bash
volatility -f Challenge.raw --profile=Win7SP1x86 consoles
```
 
The stdout contained: `335d366f5d6031767631707f`
 
That is clearly hex. I decoded it immediately but got a garbage string. That alone was not the flag. I needed to know what to do with it. That is when I went back to the challenge description hints.
 
### Step 5: Environment Variables
 
The word "environmental" in the description made me think of environment variables specifically, not just the general theme. I ran `envars`:
 
```bash
volatility -f Challenge.raw --profile=Win7SP1x86 envars
```
 
Scrolling through the output I found a variable called `Thanos` with the value `xor` and `password`. That was the connection. The description gave away both hints: environmental for envars, and Thanos for the variable name inside it.
 
So now I had three things: a hex string decoded to garbage, an instruction to XOR it, and a second hint about a password.
 
### Step 6: XOR Brute Force
 
Decoded the hex string to raw bytes and XORed every byte with values 0 through 254 to see which one produced readable output:
 
```python
a = "335d366f5d6031767631707f".decode("hex")
for i in range(0, 255):
    b = ""
    for j in a:
        b = b + chr(ord(j) ^ i)
    print b
```
 
At key value 2 (the third output since we start at 0) the result was `1_4m_b3tt3r}`. That looked like the tail end of a flag. So now I had the second half.
 
### Step 7: Password Hash
 
The second hint from the environment variable said `password`. That pointed me toward extracting password hashes from the dump:
 
```bash
volatility -f Challenge.raw --profile=Win7SP1x86 hashdump
```
 
Got back several NTLM hashes. One of them was `101da33f44e92c27835e64322d72e8b7`. I put that into a lookup tool and it cracked to `you_are_good_but`. That gave me the first half of the flag.
 
### Putting It Together
 
First half from the NTLM hash: `you_are_good_but`
Second half from the XOR decode: `1_4m_b3tt3r}`
 
Combined with the flag prefix: `flag{you_are_good_but1_4m_b3tt3r}`
 
The pattern here was following the description as a literal guide. "Environmental" pointed to environment variables. "Thanos" was the variable name inside it. The value of that variable told me the operations to apply. The console output gave me the ciphertext. The hashdump gave me the prefix. Everything connected once I treated the description as clues rather than flavour text.
 
