# TryHackMe: Forensic Imaging —> Writeup

**Room:** Forensic Imaging

---

## Overview

This room covers forensic disk imaging from the ground up. It walks through why imaging is done before any investigation begins, the tools used to do it, how to verify image integrity with hashing, and how to mount images for examination without touching the original evidence. The practical tasks involve listing block devices, hashing an existing image, mounting it to read contents, creating a new forensic image from a loop device using dc3dd, and verifying the result.

---

## Task 1: Introduction

No questions. This task introduces the concept of forensic imaging as a foundational step in digital forensics. Before any analysis happens, a bit-for-bit copy of the storage device is taken. All work is done on the copy. The original is preserved untouched so it can be verified against the copy at any point and presented in court without any risk of contamination from the investigation itself.

The room also introduces the concept of write blockers, hardware or software mechanisms that allow a device to be read without any data being written back to it. Even mounting a drive normally can alter timestamps and metadata. Write blockers prevent that.

---

## Task 2: Linux Basics for Forensics

### Question 1: What command can be used to list all block devices in Linux?

**Answer:** `lsblk`

`lsblk` lists all block devices attached to the system including physical disks, partitions, and loop devices. In a forensic context this is typically the first command you run to identify what storage devices are present and what their device names are before you start imaging. It shows the device path, size, type, and mount point if one exists.

### Question 2: Which bash command displays all commands executed in a session?

**Answer:** `history`

`history` outputs the command history for the current shell session. In forensics this is relevant because command history is itself a form of evidence. During an investigation of a suspect system, the shell history can reveal what the user was doing, what tools they ran, and in what order. It is one of the first places investigators look during live forensic analysis.

The task also covers basic Linux navigation and file handling which form the operational baseline for doing any forensic work on a Linux system.

---

## Task 3: Forensic Imaging Theory

No questions. This task covers the theoretical side of imaging in more depth.

The key distinction explained here is between a logical copy and a forensic image. A logical copy is just copying files. A forensic image is a sector-by-sector clone of the entire device including unallocated space, deleted files, file system structures, and slack space. That unallocated space is where deleted data lives and it is often where the most interesting evidence is found.

The task covers common forensic image formats. The raw format (`.img` or `.dd`) is the simplest: a straight bit-for-bit copy with no compression or metadata. The EnCase format (`.E01`) and AFF (Advanced Forensic Format) are container formats that embed metadata like case notes, examiner name, acquisition timestamps, and segment hashes directly into the image file. These are preferred in professional investigations because everything is self-documented.

The room also explains imaging tools: `dd` is the classic Unix utility, widely available but with no built-in verification or progress reporting. `dc3dd` is a forensic enhancement of `dd` that adds hashing on the fly, progress reporting, and better error handling. `dcfldd` is a similar alternative. `FTK Imager` and `Guymager` are GUI tools used in professional settings. For this room dc3dd is used for the practical imaging task.

---

## Task 4: Hashing a Forensic Image

### Question: What is the MD5 hash of the image `exercise.img`?

**Command used:**

```bash
sudo md5sum exercise.img
```

**Output:**

```
1f1da616156f73083521478c334841bb  exercise.img
```

**Answer:** `1f1da616156f73083521478c334841bb`

Hashing is how forensic integrity is verified. You hash the image at the time it is created. Any time later you can hash it again and compare. If the hashes match, the image has not been modified. If they do not match, something changed and the image cannot be trusted.

MD5 is the most commonly used hash for this purpose in forensics, though SHA256 is considered more robust since MD5 has known collision vulnerabilities. For practical integrity checking in most scenarios MD5 is still widely used because speed matters when imaging large drives.

The hash also becomes part of the documented evidence chain. You record it at acquisition time and it follows the image through every step of the investigation.

---

## Task 5: Mounting a Forensic Image

### Question: What is the content of `flag.txt` inside `exercise.img`?

**Commands used:**

```bash
sudo mount /home/analyst/exercise.img /mnt
cat /mnt/flag.txt
```

**Answer:** `THM{mounttt-mounttt-me}`

Initially I tried creating a separate mount point:

```bash
sudo mkdir /mnt/exercise
sudo mount /home/analyst/exercise.img /mnt/exercise
cat /mnt/exercise/flag.txt
```

This did not produce output, so I mounted directly to `/mnt` instead and it worked.

Mounting a forensic image lets you browse the file system it contains without running any risk of writing to the original device. The image file acts as a virtual disk. Once mounted it appears in the directory tree like any other file system.

In a real investigation you would mount with the `ro` (read-only) flag as an additional safeguard:

```bash
sudo mount -o ro /home/analyst/exercise.img /mnt
```

This ensures that even if something in the system tries to write to the mounted image, it will be blocked at the mount level. The image integrity is preserved regardless of what the investigator does while examining the contents.

---

## Task 6: Creating a Forensic Image with dc3dd

### Question 1: What is the MD5 hash of the image created from the 1GB loop device?

**Commands used:**

```bash
lsblk
sudo dc3dd if=/dev/loop4 of=evidence.img
md5sum evidence.img
```

**Output:**

```
1fab86e499934dda789c9c4aaf27101d  evidence.img
```

**Answer:** `1fab86e499934dda789c9c4aaf27101d`

I first ran `lsblk` to identify which loop device was the target. The 1GB device showed up as `/dev/loop4`.

`dc3dd` is used here instead of plain `dd` because it provides progress output during imaging and can compute hashes on the fly as it reads. The syntax is similar to `dd` with `if` for input file (the source device) and `of` for output file (the destination image).

The loop device is a Linux construct that allows a file to be treated as a block device. It is commonly used in forensics labs and CTFs to simulate physical disks without needing actual hardware.

---

### Question 2: What is the content of `flag.txt` inside the imaged loop device?

**Commands used:**

```bash
sudo mkdir -p /mnt/newimage
sudo mount -o loop evidence.img /mnt/newimage
cat /mnt/newimage/flag.txt
```

**Answer:** `THM{well-done-imaginggggggg}`

The `-o loop` option tells mount that the source is an image file rather than a physical device. Without it, mount would not know how to handle a regular file as a block device. With it, the kernel sets up a loop device automatically to back the image and the file system inside it becomes accessible.

This is the full workflow: identify the device with `lsblk`, image it with `dc3dd`, hash the image to record its integrity, then mount and examine the contents. Every step documented, original device never touched after imaging, hash on record for chain of custody.

---

## Key Takeaways

Forensic imaging always comes before analysis. You never work on the original device. The forensic image is what gets examined.

Hashing is what makes an image legally defensible. The hash recorded at acquisition time can be reproduced at any later point to prove the image has not changed.

`dc3dd` over plain `dd` for forensic work because it adds progress reporting and on-the-fly hashing. `dd` works but gives you nothing during the process and no built-in verification after.

Mounting images with read-only flags is good practice even in a lab environment. It builds the right habits for working with real evidence.

`lsblk` is always step one when working with an unfamiliar Linux system in a forensic context. Know what devices are there before you do anything else.

---
