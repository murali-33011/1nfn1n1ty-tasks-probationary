
## Lab 4: Obsession
 
**Tools:** Volatility 2
**Flag:** `inctf{1_is_n0t_EQu4l_7o_2_bUt_th1s_d0s3nt_m4ke_s3ns3}`
 
### Context
 
The description said a hacker deleted an important file. That immediately told me the file would not show up through normal `dumpfiles` because the file no longer exists on the filesystem in the traditional sense. Deleted files can sometimes still be recovered from the MFT (Master File Table) if the MFT entries have not been overwritten yet.
 
### Step 1: Profile and Process List
 
```bash
vol.py -f MemoryDump_Lab4.raw imageinfo
```
 
Got `Win7SP1x64` as the suggested profile.
 
```bash
vol.py -f MemoryDump_Lab4.raw --profile=Win7SP1x64 pslist
```
 
`StikyNot.exe` showed up and looked suspicious. I spent some time on it before realising it was a deliberate distraction planted in the challenge. Nothing useful came from it.
 
### Step 2: File Scan for Context
 
```bash
vol.py -f MemoryDump_Lab4.raw --profile=Win7SP1x64 filescan > filescan.txt
grep -i "eminem" filescan.txt
grep -i "SlimShady" filescan.txt
```
 
Two usernames came up: `eminem` and `SlimShady`. There was a file called `important.txt` associated with these paths. I tried to dump it through `dumpfiles` and it failed, which confirmed it had been deleted.
 
### Step 3: MFT Recovery
 
Since the file was deleted I turned to the MFT parser:
 
```bash
vol.py -f MemoryDump_Lab4.raw --profile=Win7SP1x64 mftparser > mft.txt
grep -A 20 -i "important.txt" mft.txt
```
 
The MFT entry for the file was still in memory. Inside the `$DATA` attribute section there was a hex blob representing the file's contents. The hex was mangled with inconsistent whitespace from the raw output so I cleaned it up in CyberChef using From Hex after stripping the whitespace. The decoded output was the flag.
 
The key thing I learned here is that `dumpfiles` only works for files that are still active in memory. For deleted files, MFT entries can persist in memory even after deletion and `mftparser` is the way to pull them out. The content is stored as a resident `$DATA` attribute in the MFT record itself when the file is small enough.
 
