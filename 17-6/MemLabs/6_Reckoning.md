
## Lab 6: The Reckoning
 
**Tools:** Volatility 2, community chromehistory plugin
**Flag:** `inctf{...}` (assembled from two separate halves)
 
### Context
 
The description said David communicated via the internet. That told me to look at browser activity. The processes in the dump included `chrome.exe`, `firefox.exe`, and `WinRAR.exe`. The WinRAR process was the obvious first stop because it meant an archive was being accessed, and the flag was split into two parts so I expected two separate trails.
 
### Part 1: WinRAR and the Password in Environment Variables
 
Checked what WinRAR was opening:
 
```bash
vol.py -f MemoryDump_Lab6.raw --profile=Win7SP1x64 cmdline | grep WinRAR.exe
```
 
It was opening `C:\Users\Jaffa\Desktop\pr0t3ct3d\flag.rar`. I found it in the file scan and dumped it:
 
```bash
vol.py -f MemoryDump_Lab6.raw --profile=Win7SP1x64 filescan | grep "flag.rar"
vol.py -f MemoryDump_Lab6.raw --profile=Win7SP1x64 dumpfiles -Q <offset> -D ./rar -n
```
 
Extracting it required a password. My thinking was that if someone was using WinRAR interactively during the dump, the password might still be floating around somewhere. Environment variables were the first place I checked because previous labs had planted passwords there:
 
```bash
vol.py -f MemoryDump_Lab6.raw --profile=Win7SP1x64 envars | grep -i WinRAR
```
 
Found the password: `easypeasyvirus`. Extracted the archive and got `flag2.png`, which despite being named flag2 contained the second half of the final flag, not a standalone flag.
 
### Part 2: Chrome History Chain
 
Recovered Chrome browsing history using the community plugin:
 
```bash
vol.py --plugins=volatility-plugins/plugins/ -f MemoryDump_Lab6.raw --profile=Win7SP1x64 chromehistory > chromehistory.txt
```
 
Scrolled through the history and found a Pastebin URL. Fetched it:
 
```bash
curl -s https://pastebin.com/raw/RSGSi1hk
```
 
The paste contained a link to a Google Drive document. I opened it and the content looked like plain filler text at first glance. I read through it carefully and found a Mega.nz link hidden inside the body of the document. That Mega link led to the first half of the flag.
 
Combining the first half from the Mega trail with the second half from `flag2.png` completed the flag.
 
The trail in this lab was intentionally long: Chrome history to Pastebin, Pastebin to Google Drive, Google Drive text to Mega, Mega to the first flag half. Each step was only visible if you followed the previous one. The WinRAR trail for the second half was much shorter but required knowing to check environment variables for the password. Both approaches relied on patterns from earlier labs which made them feel natural rather than arbitrary.
 
