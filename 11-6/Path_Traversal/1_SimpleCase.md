 
## Lab 01: File Path Traversal, Simple Case
 
**Technique:** Basic directory traversal
**Target:** `/etc/passwd`
**Status:** Solved
 
### Context
 
Path traversal happens when an application takes a filename or file path from user input and uses it directly to read a file from the server without properly validating or restricting what can be accessed. If the application serves files from a specific directory but does not confine requests to that directory, an attacker can use `../` sequences to walk up the directory tree and reach files anywhere on the filesystem.
 
This was the simplest possible case. No filters, no encoding, no validation. The application took the `filename` parameter and used it as-is.
 
### What I Did
 
Intercepted a product image request in Burp Suite:
 
```
GET /image?filename=8.jpg HTTP/2
```
 
Modified the `filename` parameter to traverse three directories up and read `/etc/passwd`:
 
```
GET /image?filename=../../../etc/passwd HTTP/2
```
 
Sent through Repeater. Server returned the contents of `/etc/passwd`.
 
### Why It Works
 
Each `../` moves one directory up from the current working directory. The application was serving images from something like `/var/www/images/`. Three levels up lands at the filesystem root, and from there `/etc/passwd` is a direct path. The server resolved it and read the file without any checks.
 
