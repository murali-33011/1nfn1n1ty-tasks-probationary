# TryHackMe: Critical -> Writeup



## Overview

This room is a guided introduction to memory forensics using Volatility 3. The scenario involves a potentially compromised Windows machine where a memory dump has been captured and handed to the analyst. The investigation covers OS profiling, network connection analysis, process enumeration, file system scanning, MFT analysis, and string extraction from a suspicious process memory dump. The goal is to piece together what happened on the machine by reading the evidence left in RAM.



## Task 1: Introduction

No questions. The room introduces memory forensics as a discipline focused on extracting volatile evidence from RAM. Volatile evidence is data that exists only while the system is running and is permanently lost the moment power is cut. This includes running processes, open network connections, loaded drivers, decrypted data, clipboard contents, and registry hives loaded into memory. Disk-based forensics cannot recover this. A memory dump captured at the right moment can provide a snapshot of the system's live state that no other evidence source can replicate.



## Task 2: Memory Forensics Theory

### Question 1: What type of memory is analyzed during a forensic memory task?

**Answer:** `RAM`

RAM (Random Access Memory) is where a running system stores everything it is actively working with. When a process runs, its code, data, and any files it has open get loaded into RAM. This is why memory forensics is so powerful for incident response: malware that runs entirely in memory and never touches the disk is invisible to traditional file-based analysis but leaves extensive traces in a RAM dump.

### Question 2: In which phase will you create a memory dump of the target system?

**Answer:** `Memory Acquisition`

Memory acquisition is the first phase of a memory forensics investigation. It involves capturing a raw bit-for-bit copy of the contents of RAM at a specific point in time. Tools used for this include WinPmem and Magnet RAM Capture on Windows, and LiME (Linux Memory Extractor) on Linux. The output is typically a `.raw` or `.dmp` file that can then be loaded into an analysis tool like Volatility.

Acquisition has to happen before analysis. The order matters because RAM is volatile. If the system is rebooted or shut down, everything in memory is gone. Acquisition preserves the state so analysis can happen later, on a separate machine, without needing the original system to be running.



## Task 3: Volatility and Tools

### Question 1: Which plugin helps get information about the OS running on the lab machine?

**Answer:** `windows.info`

`windows.info` is typically the first plugin run against a Windows memory dump. It extracts metadata about the operating system including the OS version, architecture, build number, kernel base address, and system time. This information is necessary before running most other plugins because the data structures in memory are OS-version specific. Knowing exactly what you are looking at shapes every step of the analysis that follows.

### Question 2: Which tool can take a memory dump on a Linux OS?

**Answer:** `LIME`

LiME (Linux Memory Extractor) is a loadable kernel module for Linux. It is loaded as a kernel module on the target system and dumps RAM directly to a file or over a network connection. Because it operates at the kernel level it can capture memory cleanly without the userspace interference that would affect a less privileged tool. It outputs a raw image compatible with Volatility.

### Question 3: Which command displays the help menu using Volatility?

**Answer:** `vol -h`

Running `vol -h` prints the full list of available flags and plugins. Useful when you need to check plugin names or understand what options are available for a specific plugin. Volatility 3 uses `vol` as the base command rather than the `python vol.py` syntax of Volatility 2.



## Task 4: OS Profiling the Memory Dump

All answers were obtained by running `windows.info` against the provided memory dump.

```bash
vol -f memdump.raw windows.info
```

### Question 1: Is the architecture of the machine x64?

**Answer:** `Y`

The dump confirmed a 64-bit architecture. This is important because memory structure layouts, pointer sizes, and kernel data structure offsets differ between 32-bit and 64-bit systems. Running a 64-bit plugin set against a 32-bit dump would produce garbage output.

### Question 2: What is the version of the Windows OS?

**Answer:** `10`

The machine was running Windows 10. Combined with the build number that `windows.info` also provides, this would be enough to identify the specific update version if needed for deeper analysis.

### Question 3: What is the base address of the kernel?

**Answer:** `0xf8066161b000`

The kernel base address is where the Windows kernel (`ntoskrnl.exe`) is loaded in virtual memory. This address is significant for several reasons. Modern Windows uses KASLR (Kernel Address Space Layout Randomisation) which randomises this address at boot to make exploitation harder. In memory forensics, having the kernel base address allows tools like Volatility to correctly locate and parse kernel data structures. Without it, finding process lists, loaded modules, and other kernel-resident information would be significantly harder.



## Task 5: Network and Process Analysis

### Plugin Used: `windows.netscan`

```bash
vol -f memdump.raw windows.netscan
```

`windows.netscan` scans memory for network connection structures. It can identify established connections, listening sockets, and recently closed connections that are still present in memory. It reveals the local and remote IP addresses and ports, the state of the connection, the process ID, and the owning process name.

### Question 1: Destination IP address on port 80?

**Answer:** `192.168.182.128`

A connection to port 80 on `192.168.182.128` was identified in the network scan output. Port 80 is standard HTTP. An outbound connection to an internal IP on port 80 suggests the compromised machine was communicating with an attacker-controlled server on the same network, consistent with a post-exploitation C2 or file delivery scenario.

### Question 2: Program responsible for the port 80 connection?

**Answer:** `msedge.exe`

Microsoft Edge was the owning process for the port 80 connection. This is a common way for malicious activity to blend in. A browser process making HTTP connections is not suspicious on its own. If an attacker can inject into or abuse a browser process, or if the browser was used to initiate a download that triggered the infection chain, the network activity will be attributed to a legitimate system binary.

### Question 3: PID of the child process of `critical_updat`?

**Answer:** `1612`

```bash
vol -f memdump.raw windows.pstree
```

`windows.pstree` displays the process tree showing parent-child relationships. The process `critical_updat` (a truncated name, which itself is suspicious since legitimate Windows processes rarely have truncated names at 15 characters, the EPROCESS structure limit) had spawned a child process with PID 1612. Child processes spawned by suspicious executables are worth investigating immediately because they can represent command execution, persistence mechanisms, or further payload delivery.

### Question 4: Timestamp for `critical_updat`?

**Answer:** `2024-02-24 22:51:50.000000`

The process creation timestamp is recorded in the EPROCESS structure in memory. This tells us exactly when `critical_updat` was launched. Correlating this with other timestamps in the investigation, particularly the file creation time from the MFT analysis, helps build a timeline of the attack.



## Task 6: File System and String Analysis

### Question 1: Full path of `critical_updat` from `windows.filescan`?

```bash
vol -f memdump.raw windows.filescan | grep critical
```

**Answer:** `C:\Users\user01\Documents\critical_update.exe`

`windows.filescan` scans memory for FILE_OBJECT structures which Windows creates for every file that is opened. Even if a file has been deleted from disk it can still appear here if it was opened during the memory snapshot. The full path confirms the binary was placed in the Documents folder of `user01`, which is a user-writable location and a common staging ground for malware that does not want to write into system directories where it might trigger UAC or attract attention.

The truncated name `critical_updat` visible in the process list is because the Windows EPROCESS ImageFileName field is limited to 15 characters. The actual file is `critical_update.exe`.

### Question 2: Created timestamp for `important_document.pdf` from MFT scan?

```bash
vol -f memdump.raw windows.mftscan.MFTScan | grep important_document
```

**Answer:** `2024-02-24 20:39:42.000000`

The Master File Table is the NTFS structure that records metadata for every file on the volume including four timestamps: created, modified, accessed, and MFT entry changed. `windows.mftscan.MFTScan` parses MFT entries that are resident in memory. The PDF was created about two hours before `critical_update.exe` was executed, suggesting it may have been a lure document used to distract the user while the malicious executable ran in the background, or it was itself a vehicle for the initial compromise.

### Question 3: Server identified in `updater.exe` memory strings?

```bash
vol -f memdump.raw windows.memmap --pid <PID> --dump
strings pid.<PID>.dmp | grep -i server
```

**Answer:** `SimpleHTTP/0.6 Python/3.10.4`

Dumping the memory of the suspicious process and running `strings` against it revealed an HTTP server header in the captured traffic. `SimpleHTTP/0.6 Python/3.10.4` is the server string returned by Python's built-in `http.server` module. This is a standard tool used by attackers during post-exploitation to quickly serve files over HTTP, typically for payload delivery or exfiltration. The attacker running a Python SimpleHTTP server on `192.168.182.128` aligns directly with the port 80 connection identified in the network scan. The full picture is that the victim machine connected to the attacker's Python HTTP server on the internal network, likely to download additional payloads.



## Reconstructed Attack Timeline

Based on the evidence extracted from the memory dump:

`2024-02-24 20:39:42`  `important_document.pdf` created on the system, possibly used as a lure or delivered as part of the initial compromise.

`2024-02-24 22:51:50` : `critical_update.exe` executed from `C:\Users\user01\Documents\`. The process name was truncated to `critical_updat` in the EPROCESS structure.

At some point after execution, `critical_update.exe` spawned a child process (PID 1612) and established or facilitated an outbound HTTP connection to `192.168.182.128:80`, a Python SimpleHTTP server under attacker control.



## Key Takeaways

`windows.info` first, always. You need to confirm what OS and architecture you are working with before anything else makes sense.

`windows.netscan` is one of the most immediately useful plugins for identifying active or recent C2 connections. IP addresses, ports, and owning processes all in one output.

`windows.pstree` reveals parent-child relationships that are critical for understanding attack chains. Legitimate system processes have predictable parents. A suspicious executable spawning children is a strong signal.

`windows.filescan` finds file objects in memory. Useful for locating malware that may have already been deleted from disk.

MFT timestamps from `windows.mftscan.MFTScan` let you build a timeline independent of file system timestamps which can be manipulated. MFT entries in memory are harder to tamper with after the fact.

Strings analysis on a dumped process memory can reveal network traffic, C2 addresses, server banners, and command output that the process was working with at the time the dump was taken.

