## 6. Hex Editor
 
A hex editor lets you view and edit the raw bytes of any file. No interpretation, no rendering, just the actual data. Every byte is shown in hexadecimal on one side and its ASCII character on the other.
 
This is useful when:
 
- You need to manually check or fix a file signature
- You want to see what is actually in a file beyond what a normal viewer shows
- You are looking for hidden strings or embedded files
- You want to modify specific bytes (like fixing a corrupted file header)
```bash
xxd file.jpg | head -20       # quick look at first 20 lines
hexedit file.jpg               # interactive hex editor in terminal
```
 
GUI tools like **010 Editor** and **HxD** are common in forensics work. They let you jump to offsets, search for byte patterns, and apply templates to known file formats so the structure is parsed visually.
 
The combination of a hex editor and known file signatures lets you manually carve files out of a disk image even without dedicated recovery software. You find the magic bytes, identify where the file ends, and copy that slice of bytes out.
 
---
