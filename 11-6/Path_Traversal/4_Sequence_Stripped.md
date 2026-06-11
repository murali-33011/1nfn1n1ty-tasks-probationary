## Lab 04: Traversal Sequences Stripped with Superfluous URL Decode
 
**Technique:** Double URL encoding to bypass filter before decode
**Target:** `/etc/passwd`
**Status:** Solved
 
### Context
 
The application blocked `../` and also decoded URL-encoded input and blocked `%2f` (which is `/` URL-encoded). The order of operations is what creates the vulnerability. The filter ran on the raw input before the server performed URL decoding. So if you encode the characters in a way the filter does not recognise, they pass through validation and then get decoded into their real values before the file read happens.
 
### What I Did
 
Intercepted the image request and used a double URL-encoded payload:
 
```
GET /image?filename=..%252f..%252f..%252fetc/passwd HTTP/2
```
 
Sent through Repeater. Server returned `/etc/passwd`.
 
### Why It Works
 
`%25` is the URL encoding of the `%` character. So `%252f` is a double-encoded `/`:
 
```
%252f  
%25 decodes to %  
leaving %2f  
%2f decodes to /
```
 
When the filter saw `%252f` it did not recognise it as a slash because it had not decoded it yet. The filter saw no traversal sequences and passed the input through. The server then decoded `%252f` to `%2f` and then to `/`, reconstructing the traversal sequence just before the file read.
 
This works because the validation and the file read were not operating on the same version of the input. Validation should always happen on the fully decoded and normalised input. If the file system will see a decoded version, the filter needs to see the same decoded version.
 
