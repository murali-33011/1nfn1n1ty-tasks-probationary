## Lab 1: Beginner's Luck
 
**Tools:** Volatility 3, GIMP
**Flags:**
- Flag 1: `flag{th1s_1s_th3_1st_st4g3!!}`
- Flag 2: `flag{G00d_Boy_good_girL}`

  
### Description of Challenge
 
The description mentioned a black window popping up (cmd.exe) and someone trying to draw something (mspaint). Two clear leads in the description, both pointing at specific processes.
 
### Step 1: Profile and Process List
 
```bash
vol -f MemoryDump_Lab1.raw windows.info
vol -f MemoryDump_Lab1.raw windows.pslist
```
 
Confirmed Windows 7 SP1 x64. Three processes stood out immediately:
 
`cmd.exe` because of the black window mention. `mspaint.exe` with PID 2424 because of the drawing mention. `WinRAR.exe` with PID 1512 which I noted but did not end up needing for the flags I found.
 
### Flag 1: CMD Console Output
 
Checked command line arguments first:
 
```bash
vol -f MemoryDump_Lab1.raw windows.cmdline
```
 
Saw a command referencing something called `St4G3$1`. That name looked intentionally staged (pun intended). I looked at console output next:
 
```bash
vol -f MemoryDump_Lab1.raw windows.consoles
```
 
Found a Base64 string in the output: `ZmxhZ3t0aDFzXzFzX3RoM18xc3Rfc3Q0ZzMhIX0=`
 
Decoded it immediately:
 
```bash
echo ZmxhZ3t0aDFzXzFzX3RoM18xc3Rfc3Q0ZzMhIX0= | base64 -d
```
 
Output: `flag{th1s_1s_th3_1st_st4g3!!}`
 
That was clean. Stage one done quickly.
 
### Flag 2: MS Paint Memory Recovery
 
The description said she was drawing. That pointed straight at `mspaint.exe` PID 2424. I knew from prior reading that mspaint stores the canvas contents in memory and you can recover them by dumping the process memory and opening the raw bytes as an image in GIMP.
 
Dumped the process memory:
 
```bash
vol -f MemoryDump_Lab1.raw -o ./lab1_output/ windows.memmap --pid 2424 --dump
```
 
Got a `.dmp` file. Renamed it to `.data` so GIMP would accept it as raw image data:
 
```bash
mv lab1_output/pid.2424.dmp lab1_output/2424.data
```
 
Opened it in GIMP as a raw image. The tricky part here is that you have to manually set the width to get the image to appear correctly. I started around 1200 and adjusted until the rendered output looked like an actual image rather than noise. Once the right width was found the image appeared but it was upside down and mirrored, which is normal for mspaint memory dumps.
 
Applied `Image > Transform > Rotate 180°` followed by `Flip Horizontally` and the flag became readable in the image.
 
Flag 2: `flag{G00d_Boy_good_girL}`
 
The key insight for this one was knowing that mspaint does not write to disk, it keeps the canvas in memory. The process dump is the only way to recover it.
 
