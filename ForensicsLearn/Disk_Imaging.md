## 4. Disk Imaging
 
A forensic image is a bit-for-bit exact copy of a storage device. Every single bit is copied in order, including deleted files, unallocated space, and slack space. It is not the same as just copying files over.
 
The reason we image the disk first rather than working on the original is to preserve evidence. Any action on the original drive (even just mounting it) can alter timestamps and overwrite data. You always work on the copy.
 
```bash
dd if=/dev/sda of=disk.img bs=4M
```
 
### Checksums
 
After imaging, you hash the image to verify its integrity. If the hash of the image matches the hash of the original drive, you have a verified exact copy. If anyone tampers with the image later, the hash will change and you will know.
 
```bash
md5sum disk.img
sha256sum disk.img
```
 
SHA256 is preferred over MD5 in real forensics because MD5 has known collision vulnerabilities. But both are fine for verifying accidental corruption.
 
---
 
