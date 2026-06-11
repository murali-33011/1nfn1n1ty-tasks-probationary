## Lab 05: Validation of Start of Path
 
**Technique:** Prefix bypass with traversal sequences appended after the valid prefix
**Target:** `/etc/passwd`
**Status:** Solved
 
### Context
 
The application required the filename parameter to start with a specific directory prefix such as `/var/www/images/`. This is an attempt to ensure the file path stays within the intended directory by enforcing a known starting point. The flaw is that checking only the prefix of a path tells you nothing about where the path ends up after resolution.
 
### What I Did
 
Intercepted the image request. The server was passing a full path in the filename parameter rather than just a filename:
 
```
GET /image?filename=/var/www/images/8.jpg HTTP/2
```
 
Modified it to include traversal sequences after the valid prefix:
 
```
GET /image?filename=/var/www/images/../../../etc/passwd HTTP/2
```
 
Sent through Repeater. Server resolved the path and returned `/etc/passwd`.
 
### Why It Works
 
The validation checked that the path started with `/var/www/images/` and it did. The check passed. But then the path resolution process handled the `../` sequences and the final path that the server actually opened was `/etc/passwd`, which is nowhere near `/var/www/images/`.
 
The correct fix is to resolve the canonical path first, then check whether the resolved path starts with the allowed prefix. The `realpath()` function in many languages does this. Checking the input string before resolution is checking the wrong thing.
 
