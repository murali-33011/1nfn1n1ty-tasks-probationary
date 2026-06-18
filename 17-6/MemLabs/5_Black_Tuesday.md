
## Lab 5: Black Tuesday
 
**Tools:** Volatility 2
**Flags:**
- Flag 1: `flag{!!_w3LL_d0n3_St4g3-1_0f_L4B_5_D0n3_!!}`
- Flag 2: revealed inside `Stage2.png`
- Flag 3: recovered from hidden notepad process memory
### Context
 
The description gave several leads. Strange files with unreadable names made of alphabets and numbers pointed toward Base64 encoded filenames. A crashing application was probably a rabbit hole or a hint about a specific process. The three-flag structure and the note saying the second flag is not the end told me to keep digging even once I found something that looked complete.
 
### Step 1: Profile and Processes
 
```bash
vol.py -f MemoryDump_Lab5.raw imageinfo
```
 
Profile: `Win7SP1x64`
 
```bash
vol.py -f MemoryDump_Lab5.raw --profile=Win7SP1x64 pstree
```
 
`WinRAR.exe` was running and several `NOTEPAD.EXE` instances appeared. The WinRAR process was the first thing I wanted to look at because archive files in CTF memory challenges almost always contain flags.
 
### Flag 1: Browser History and Base64 Filename
 
I checked what WinRAR was opening:
 
```bash
vol.py -f MemoryDump_Lab5.raw --profile=Win7SP1x64 cmdline -p <WinRAR_PID>
```
 
It was opening a file with a name that looked like Base64: `SW1wb3J0YW50.rar`. I noted this for later but extracting it was going to need a password. The challenge description mentioned files with unreadable names, which made me think browser or explorer history. I ran `iehistory`:
 
```bash
vol.py -f MemoryDump_Lab5.raw --profile=Win7SP1x64 iehistory
```
 
This showed a `.bmp` file in the Pictures directory with a long Base64-looking filename. I decoded it:
 
```bash
echo "ZmxhZ3shIV93M0xMX2QwbjNfU3Q0ZzMtMV8wZl9MNEJfM19EMG4zXyEhfQ==" | base64 -d
```
 
Output: `flag{!!_w3LL_d0n3_St4g3-1_0f_L4B_5_D0n3_!!}`
 
The challenge had a known typo where `L4B_3` should be read as `L4B_5`. This was Flag 1 and it was also going to be the password for the RAR.
 
### Flag 2: Password-Protected RAR
 
Found the RAR file in the file scan:
 
```bash
vol.py -f MemoryDump_Lab5.raw --profile=Win7SP1x64 filescan | grep "SW1wb3J0YW50.rar"
vol.py -f MemoryDump_Lab5.raw --profile=Win7SP1x64 dumpfiles -Q <offset> -D . -n
```
 
Used Flag 1 as the password to extract it:
 
```bash
unrar e file.None.<offset>.SW1wb3J0YW50.rar.dat
```
 
Got `Stage2.png` which contained Flag 2 visibly in the image.
 
### Flag 3: Hidden Notepad Process
 
The note said the second flag was not the end. I switched to `psxview` which cross-references multiple process lists and flags any process that appears in some but not all of them. Hidden processes show up this way because malware or intentional hiding typically makes a process invisible in the standard `pslist` or `pstree` output by unlinking it from certain lists, but it cannot hide from all of them simultaneously.
 
```bash
vol.py -f MemoryDump_Lab5.raw --profile=Win7SP1x64 psxview
```
 
Found additional `notepad.exe` instances that were not visible in the earlier `pstree` output. Dumped the memory of those processes:
 
```bash
vol.py -f MemoryDump_Lab5.raw --profile=Win7SP1x64 memdump -p <PID> -D .
strings <pid>.dmp | grep -i "inctf"
```
 
Flag 3 appeared in the strings output from one of those hidden process dumps.
 
The lesson from this lab is that `psxview` is essential when a challenge description hints at hidden activity or something suspicious. Normal process listing only shows you what the OS wants you to see. Cross-referencing multiple enumeration methods reveals what was hidden.
