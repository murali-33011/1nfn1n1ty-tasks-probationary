## Lab 06: Validation of File Extension with Null Byte Bypass
 
**Technique:** Null byte injection to truncate enforced file extension
**Target:** `/etc/passwd`
**Status:** Solved
 
### Context
 
The application enforced that the filename had to end with a specific extension such as `.png`. The intent is to restrict access to image files only. The vulnerability is that the enforcement was done at the application layer using string checking, but the underlying file access was done in a context where a null byte terminates the string. These two layers disagreed about where the filename ended.
 
### What I Did
 
Intercepted the image request and crafted a payload with a URL-encoded null byte:
 
```
GET /image?filename=../../../etc/passwd%00.png HTTP/2
```
 
Sent through Repeater. Server returned `/etc/passwd`.
 
### Why It Works
 
`%00` is the URL encoding of the null byte, which is the character with ASCII value zero. In C and languages built on C runtime functions, strings are null-terminated. When the underlying file open function encountered the null byte, it treated everything after it as irrelevant. The actual path it opened was `../../../etc/passwd`.
 
The validation layer saw `../../../etc/passwd%00.png` and confirmed the extension was `.png`. Check passed. The file system saw `../../../etc/passwd` and opened it. Two different views of the same input.
 
This class of vulnerability is more common in older applications and in code that bridges between managed languages and native system calls. Modern high-level languages typically do not expose null byte issues in their string handling, but calling into native file APIs can reintroduce them.
 
The fix is to validate on the fully decoded and normalised filename after stripping null bytes, not on the raw input.
 
