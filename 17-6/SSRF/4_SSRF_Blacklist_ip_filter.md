
## Lab 4: SSRF with Blacklist-Based Input Filter
 
**Technique:** SSRF bypass using alternate localhost representation and double URL encoding
**Target:** Access internal admin and delete carlos despite a blacklist filter
**Status:** Solved
 
### Context
 
This lab had a filter that blocked obvious SSRF payloads. Direct attempts at `http://127.0.0.1/` and `/admin` were rejected. The application was checking the URL against a blacklist of known patterns. Blacklist-based filters are fundamentally flawed because they try to enumerate all the ways something dangerous can be expressed, and that list is never complete.
 
### What I Did
 
First attempt with `http://127.0.0.1/` came back blocked. I tried alternative representations of localhost. `http://127.1/` worked. This is a valid IP address shorthand that most HTTP clients and operating systems handle the same as `127.0.0.1`, but it does not match a simple string check for `127.0.0.1` or `localhost`.
 
That got me past the host restriction and I landed on the server's response. But trying to access `/admin` was blocked too. The filter was scanning the path for the word "admin". I needed to obfuscate it.
 
Double URL encoding was the approach. The letter `a` is `%61` URL-encoded. `%` itself is `%25` URL-encoded. So `a` double-encoded is `%2561`. When the filter ran it saw `%2561dmin` and found no match for "admin". When the server decoded the path to make the request it decoded `%25` to `%` first, giving `%61`, then decoded `%61` to `a`, giving `admin`.
 
The final payload:
 
```
stockApi=http://127.1/%2561dmin
```
 
Got the admin panel. Found the deletion URL and repeated the same approach with `delete` if needed, then submitted the request.
 
### The Filter Bypass Pattern
 
The filter decoded the URL once before checking it. The server decoded it twice before using it. One extra round of encoding was enough to make the dangerous string invisible to the check while remaining functional for the actual request. This is the same double-encoding principle from the path traversal labs.
 
