## Lab 2: A New World
 
**Tools:** Volatility 3, KeePassXC, sqlite3, 7zip


**Flags:**
- Flag 1: `flag{w3lc0m3_T0_$T4g3_!_Of_L4B_2}`
- Flag 2: `flag{w0w_th1s_1s_Th3*SeC0nD_ST4g3*!!}`
- Flag 3: `flag{oK_So_Now_St4g3_3_is_DoNE!!}`

  
### Description of Challenge
 
The description mentioned "environmental" again (same trick as Lab 0) and the subject using browsers and password managers. That gave me three clear leads: check environment variables, check Chrome history, and find the KeePass database.
 
### Flag 1: Environment Variables
 
Based on Lab 0 I went to environment variables early this time:
 
```bash
vol -f MemoryDump_Lab2.raw windows.envars | grep "NEW_TMP"
```
 
Found a variable called `NEW_TMP` with a value that looked like a path ending in a Base64 string:
 
```
C:\Windows\ZmxhZ3t3M2xjMG0zX1QwXyRUNGczXyFfT2ZfTDRCXzJ9
```
 
Took the Base64 part and decoded it:
 
```bash
echo ZmxhZ3t3M2xjMG0zX1QwXyRUNGczXyFfT2ZfTDRCXzJ9 | base64 -d
```
 
Output: `flag{w3lc0m3_T0_$T4g3_!_Of_L4B_2}`
 
The pattern from Lab 0 applied directly here.
 
### Flag 2: KeePass Database
 
I knew from the description that a password manager was involved. Scanned for `.kdbx` files:
 
```bash
vol -f MemoryDump_Lab2.raw windows.filescan | grep -i ".kdbx"
```
 
Found `Hidden.kdbx` at `\Device\HarddiskVolume2\Users\SmartNet\Secrets\Hidden.kdbx`. Dumped it using the physical offset.
 
Now I needed the master password. My thinking was that if they used a password manager, the master password might be stored somewhere accessible. I searched for anything named "password":
 
```bash
vol -f MemoryDump_Lab2.raw windows.filescan | grep -i "password"
```
 
Found `Password.png` in the Pictures folder of another user. Dumped that image and opened it. The master password was written visibly in the corner of the image. Used it to open the KeePass database in KeePassXC and the flag was stored as a password entry inside.
 
Flag 2: `flag{w0w_th1s_1s_Th3*SeC0nD_ST4g3*!!}`
 
### Flag 3: Chrome History and Mega Link
 
Scanned for Chrome's history SQLite file:
 
```bash
vol -f MemoryDump_Lab2.raw windows.filescan | grep -i "chrome" | grep -i "history"
```
 
Dumped the history database file and queried it:
 
```bash
sqlite3 lab2_output/file.*.dat "SELECT url FROM urls ORDER BY last_visit_time DESC;"
```
 
A Mega.nz link appeared pointing to a folder with an `Important.zip` file. Downloaded the zip. It was password protected.
 
I stared at this for a while. The challenge had given me two flags so far. My first instinct was to try the other flags as the password, but that seemed too direct. Then I noticed the zip was for Stage 3 and the challenge description mentioned the password was the SHA1 hash of the Flag 3 from Lab 1. I checked back:
 
```bash
echo -n "flag{w3ll_3rd_stage_was_easy}" | sha1sum
```
 
Got `6045dd90029719a039fd2d2ebcca718439dd100a`. Used that as the zip password:
 
```bash
7z x Important.zip
```
 
Extracted an image containing the flag.
 
Flag 3: `flag{oK_So_Now_St4g3_3_is_DoNE!!}`
 
