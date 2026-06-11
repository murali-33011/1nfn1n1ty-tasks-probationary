
## Lab 02: Traversal Sequences Blocked with Absolute Path Bypass
 
**Technique:** Absolute path injection
**Target:** `/etc/passwd`
**Status:** Solved
 
### Context
 
The application attempted to defend against path traversal by blocking `../` sequences in the input. The idea is that if you cannot traverse up directories, you cannot escape the intended directory. This is a reasonable starting point but it is not sufficient on its own because it only addresses relative traversal and says nothing about absolute paths.
 
### What I Did
 
Intercepted the same image request:
 
```
GET /image?filename=8.jpg HTTP/2
```
 
Instead of using `../` sequences I supplied an absolute path directly:
 
```
GET /image?filename=/etc/passwd HTTP/2
```
 
Sent through Repeater. Server returned `/etc/passwd`.
 
### Why It Works
 
The filter was only looking for and stripping `../`. It never considered that someone could just supply a full absolute path starting from `/`. The application passed the value through to the file read function which is happy to accept absolute paths. There was nothing to bypass because the protection never covered this case.
 
The lesson here is that blocking one attack pattern is not the same as fixing the underlying vulnerability. You have to validate that the final resolved path is within the intended directory, regardless of what form the input took.
 
