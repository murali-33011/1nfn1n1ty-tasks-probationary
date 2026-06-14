 ## Lab 01: Exploiting XXE Using External Entities to Retrieve Files

 
### Context
 
XXE (XML External Entity) injection happens when an application parses XML input and the XML parser has external entity processing enabled. External entities are a feature of the XML specification that let you define a reference to an external resource inside the DTD (Document Type Definition). If the parser resolves these and the application is not configured to prevent it, you can point an entity at any file on the server and have its contents returned in the response.
 
This lab's stock checking functionality accepted XML, parsed it server-side, and reflected values back in the response. That reflected output is what made this a classic rather than blind XXE.
 
### What I Did
 
Navigated to a product page and clicked Check Stock. Intercepted the POST request in Burp and sent it to Repeater. The original XML body looked something like:
 
```xml
<?xml version="1.0" encoding="UTF-8"?>
<stockCheck>
    <productId>1</productId>
    <storeId>1</storeId>
</stockCheck>
```
 
I added a DOCTYPE declaration defining an external entity that pointed to `/etc/passwd`, then referenced that entity in the `productId` field:
 
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<stockCheck>
    <productId>&xxe;</productId>
    <storeId>1</storeId>
</stockCheck>
```
 
Sent the request. The response contained the full contents of `/etc/passwd`.
 
### Why It Works
 
The parser saw `&xxe;` in the XML body and resolved it by reading the file at the path specified in the entity declaration. The result was substituted into the element value and the application reflected it back. External entity processing in XML parsers is enabled by default in many environments and needs to be explicitly disabled. If it is not, any file the server process has read access to is readable by anyone who can send XML to the application.
 
