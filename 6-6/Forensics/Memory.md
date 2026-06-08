 
## 5. Memory Forensics
 
Memory forensics is analysing a RAM dump. Running processes, open network connections, decrypted data, passwords in memory, and clipboard contents all live in RAM and nowhere else. If you image memory at the right moment you can find things that were never written to disk.
 
**Volatility** is the standard tool for this.
 
### Workflow
 
```bash
# Step 1: Run strings for quick wins
strings memory.dmp | grep -i flag
 
# Step 2: Identify the OS profile
volatility -f memory.dmp imageinfo
 
# Step 3: List running processes
volatility -f memory.dmp --profile=PROFILE pslist
volatility -f memory.dmp --profile=PROFILE pstree
volatility -f memory.dmp --profile=PROFILE psscan
 
# Step 4: Dump memory of a specific process (e.g. notepad.exe with PID 1234)
volatility -f memory.dmp --profile=PROFILE memdump -p 1234 -D output/
 
# Step 5: View the dumped data in the right format
# notepad = .txt, photoshop = .psd, word = .docx
strings output/1234.dmp | less
```
 
You must identify the profile first because Volatility uses it to interpret memory structures correctly. The same raw bytes mean different things in Windows XP versus Windows 10.
 
Suspicious process signs to look for: processes with no parent, processes with names that look like legitimate system processes but have different paths, multiple instances of processes that should only run once, processes with strange memory regions.
 
---
