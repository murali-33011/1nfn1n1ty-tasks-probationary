
## Lab 1: Basic SSRF Against the Local Server
 
**Technique:** SSRF via stockApi parameter to localhost
**Target:** Delete user carlos via internal admin interface
**Status:** Solved
 
### Context
 
SSRF (Server-Side Request Forgery) is when an application makes an HTTP request to a URL that comes from user input. The server is the one making the request, not the browser. This matters because the server often has access to internal resources that are not reachable from the outside. Admin panels, internal APIs, metadata services, and other backend systems that are firewalled from the internet are all accessible to the server itself.
 
This lab had the most direct version of the vulnerability. The stock check functionality accepted a full URL in the `stockApi` parameter and passed it directly to an HTTP request with no validation.
 
### What I Did
 
I visited a product page and triggered the stock check. Intercepted the POST request in Burp and sent it to Repeater. The parameter in the body was:
 
```
stockApi=https://target-store.net/product/stock
```
 
I changed it to:
 
```
stockApi=http://localhost/admin
```
 
The response came back with the full HTML of the internal admin panel. The server fetched `localhost/admin` on my behalf and returned its contents. I read through the HTML and found the deletion endpoint:
 
```
http://localhost/admin/delete?username=carlos
```
 
Updated the `stockApi` value to that URL and sent the request. Carlos was deleted.
 
### Why It Works
 
The application took the URL from the request body and made a server-side HTTP request to it without checking whether the destination was internal or external. From the server's perspective, `localhost` is just another address. It has no reason not to fetch it. The admin interface was only protected by network-level access control, meaning it was only supposed to be reachable from the server itself. The SSRF made the server reach it for me.
 
