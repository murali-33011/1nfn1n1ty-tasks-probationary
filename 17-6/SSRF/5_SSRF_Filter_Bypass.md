
## Lab 5: SSRF with Filter Bypass via Open Redirection
 
**Technique:** SSRF through an open redirect on the same domain
**Target:** Access internal admin at a known IP via redirect chain and delete carlos
**Status:** Solved
 
### Context
 
This lab blocked direct requests to internal IP addresses in the `stockApi` parameter. The filter caught any URL pointing to `192.168.x.x` or similar internal ranges. But the application trusted URLs on its own domain. This is where the open redirect becomes the bypass.
 
An open redirect is when a page on the application takes a URL from a parameter and redirects the browser to it. Here the redirect was in the page's own navigation functionality and the `path` parameter was reflected into the `Location` response header. Normally this is a moderate severity finding on its own. Combined with SSRF it becomes the bypass for the IP restriction.
 
### What I Did
 
I found the open redirect while looking at the product navigation. The `nextProduct` endpoint accepted a `path` parameter:
 
```
/product/nextProduct?path=https://external-site.com
```
 
And the `Location` header in the response contained whatever I put in `path`. The server trusted this URL because it was on its own domain. So I set `stockApi` to point at this redirect endpoint and made the redirect target the internal admin:
 
```
stockApi=http://target-store.net/product/nextProduct?path=http://192.168.0.12:8080/admin
```
 
The application saw a URL on its own domain and did not block it. The server fetched that URL. The server's own application redirected it to the internal address. The server followed the redirect and fetched the admin panel. The response came back to me.
 
To delete carlos I updated the redirect target:
 
```
stockApi=http://target-store.net/product/nextProduct?path=http://192.168.0.12:8080/admin/delete?username=carlos
```
 
Sent it. Done.
 
### Why the Redirect Works as a Bypass
 
The filter checked whether the `stockApi` URL was pointing at an internal IP. My URL was pointing at the application's own domain, so it passed. The filter had no visibility into where the redirect ultimately led. The application itself was the one that redirected to the internal address. The filter was checking the first hop but not where the chain ended.
 
This is a common pattern in SSRF bypass techniques. If the filter only validates the initial URL and the application follows redirects automatically, any redirect the application trusts becomes a proxy to internal resources.
 
