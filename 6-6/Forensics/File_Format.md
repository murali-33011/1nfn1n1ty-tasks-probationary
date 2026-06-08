## 1. File Formats and File Signatures
 
Every file format has a unique identifier baked into the very start of the file called a **file signature** (also called a magic number). It is usually 2 to 4 bytes long and sits at the beginning of the file. The OS and forensic tools use this to identify what type of file it actually is, regardless of what the file extension says.
 
This matters because someone can rename `malware.exe` to `photo.jpg` but the file signature will still say it is an executable.
 
### How to Read It
 
```bash
xxd img.jpg | head
hexdump img.jpg
```
 
`xxd` dumps the file in hex and ASCII side by side. The first few bytes are the signature. JPEG files always start with `FF D8 FF`.
 
### Using the file Command
 
```bash
file file-a.jpg
```
 
Output:
```
file-a.jpg: JPEG image data, JFIF standard 1.01, resolution (DPI), density 96x96,
segment length 16, comment: "CREATOR: gd-jpeg v1.0 (using IJG JPEG v80), quality = 90",
baseline, precision 8, 1024x576, components 3
```
 
The `file` command reads the magic bytes and tells you what the file actually is. It does not trust the extension.
 
### Common File Signatures
 
| File Type | Hex Signature | ASCII |
|---|---|---|
| JPEG | `FF D8 FF` | `ÿØÿ` |
| PNG | `89 50 4E 47` | `.PNG` |
| PDF | `25 50 44 46` | `%PDF` |
| ZIP | `50 4B 03 04` | `PK..` |
| EXE | `4D 5A` | `MZ` |
 
---
