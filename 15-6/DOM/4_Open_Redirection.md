
## Lab 04: DOM-Based Open Redirection
 
**Technique:** DOM-based open redirection via `url` parameter

**Sink:** `location.href`
 
### Context
 
Open redirection is when an application redirects a user to a URL that is controlled by an attacker. DOM-based open redirection specifically means the redirect happens through client-side JavaScript rather than a server-side Location header. The application read the `url` parameter from the current page URL using a regex and assigned the extracted value to `location.href`.
 
The regex was:
 
```javascript
var returnUrl = /url=(https?:\/\/.+)/.exec(location);
if (returnUrl)
    location.href = returnUrl[1];
```
 
This accepted any URL starting with `http://` or `https://` and redirected to it. There was no check that the destination was on the same domain.
 
### What I Did
 
The blog post page had a "Back to Blog" link that used this JavaScript. I crafted a URL that included my exploit server as the `url` parameter value:
 
```
https://LAB-ID.web-security-academy.net/post?postId=4&url=https://EXPLOIT-SERVER-ID.web-security-academy.net
```
 
When this page loaded and the Back to Blog link was clicked, the JavaScript extracted my exploit server URL from the `url` parameter and set `location.href` to it. The victim's browser was redirected to the exploit server.
 
