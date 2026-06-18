
## Lab 2: Basic SSRF Against Another Back-End System
 
**Technique:** SSRF with IP enumeration via Burp Intruder
**Target:** Discover internal admin on 192.168.0.X and delete carlos
**Status:** Solved
 
### Context
 
In this lab the admin interface was not on localhost but on a separate internal backend server somewhere in the `192.168.0.X` range. I did not know which host it was on so I had to scan the subnet.
 
### What I Did
 
Intercepted the stock check request and sent it to Intruder. Set the `stockApi` parameter to:
 
```
stockApi=http://192.168.0.§1§:8080/admin
```
 
Marked the last octet as the payload position and configured a simple numeric payload list from 1 to 255. Launched the attack.
 
Most responses came back with a non-200 status. One host returned 200 with content that matched an admin panel. I noted that IP, sent the request to Repeater, and changed the path:
 
```
stockApi=http://192.168.0.[found-ip]:8080/admin/delete?username=carlos
```
 
Sent it. Carlos deleted.
 
### The Thinking Behind It
 
The challenge description said the admin was somewhere on that subnet. Brute forcing every possible host in a /24 range takes a few seconds in Intruder when each request either immediately connects or immediately fails. The status code difference (200 vs 404 or connection refused) makes the hit easy to spot in the results. This is exactly what SSRF with Intruder is designed for: internal network discovery that you cannot do from the outside.
 
