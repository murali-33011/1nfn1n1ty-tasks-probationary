 
## Lab 3: Blind SSRF with Out-of-Band Detection
 
**Technique:** Blind SSRF via Referer header with Burp Collaborator callback
**Target:** Confirm SSRF exists through out-of-band DNS and HTTP interaction
**Status:** Solved
 
### Context
 
Blind SSRF is where the server makes the request but the response from that request does not come back to you in the HTTP response. You cannot see the output. To confirm the vulnerability exists you need to make the server reach a host you control and watch for the incoming connection. Burp Collaborator is exactly that.
 
The interesting thing about this lab is that the vulnerable parameter was not in the request body. It was in the `Referer` header. Some applications use the Referer header to make analytics or tracking requests in the background. That background request is an SSRF surface that is easy to miss.
 
### What I Did
 
Intercepted a product page request. The application was sending the Referer header on these requests. I generated a Burp Collaborator payload URL and replaced the Referer value with it:
 
```
Referer: https://YOUR-COLLABORATOR-PAYLOAD.burpcollaborator.net
```
 
Forwarded the request. Opened the Collaborator client and clicked Poll Now. Both a DNS lookup and an HTTP request arrived from the target application.
 
### What This Tells Me
 
The application was reading the Referer header and making an outbound request to whatever URL it found there. Because I pointed it at Collaborator I could see the interaction. If I had pointed it at an internal resource, the same mechanism would fetch it. Blind SSRF confirmed.
 
