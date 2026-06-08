## 2. Metadata
 
Metadata is data about data. The file itself contains the content but metadata describes that content: when it was created, what created it, where it was created, and so on.
 
### EXIF Data
 
EXIF (Exchangeable Image File Format) is metadata specifically attached to image files. A photo taken on a phone can contain GPS coordinates of where it was taken, timestamp, device make and model, camera settings, and software used to edit it.
 
```bash
exiftool filename
```
 
This gives you file size, dimensions, file type, program used to create it, OS used to create it, and for images all the EXIF fields. In CTFs this is often where the flag is sitting.
 
### Timestamps (MAC Times)
 
Timestamps record when things happened to a file. The classic set is called MAC.
 
| Term | What it means | Where it lives |
|---|---|---|
| **Modified** `mtime` | Last time the file contents were changed | Filesystem |
| **Accessed** `atime` | Last time the file was opened or read | Filesystem |
| **Created** `ctime` or `btime` | When the file was first created | Filesystem |
| **Date Changed (MFT)** | When the MFT record itself was last changed | NTFS MFT entry |
| **Filename Date Created (MFT)** | Creation time in the MFT filename attribute | NTFS MFT filename attribute |
| **Filename Date Modified (MFT)** | Modify time in the MFT filename attribute | NTFS MFT filename attribute |
| **Filename Date Accessed (MFT)** | Access time in the MFT filename attribute | NTFS MFT filename attribute |
| **INDX Entry Date Created** | Creation time in directory index entries | NTFS `$I30` index |
| **INDX Entry Date Modified** | Modify time in directory index | NTFS `$I30` index |
| **INDX Entry Date Accessed** | Access time in directory index | NTFS `$I30` index |
| **INDX Entry Date Changed** | Record change time in directory index | NTFS `$I30` index |
 
### Why This Matters
 
Creating, copying, moving, or opening a file all affect different MAC times in different ways. If you can pull all timestamps for a set of files you can reconstruct a timeline of exactly what happened. This is how investigators figure out the sequence of events in an incident.
 
On Windows, timestamps live in the NTFS MFT (Master File Table). On Linux they are stored in the inode. Tools like `stat`, `ls -l`, `exiftool`, and `Autopsy` can pull these.
 
---
