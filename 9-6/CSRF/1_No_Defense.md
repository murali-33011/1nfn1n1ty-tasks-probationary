## Lab 01: CSRF Vulnerability with No Defenses
 
**Technique:** Basic CSRF
**Target:** Email change functionality
**Status:** Solved
 
### Context
 
CSRF (Cross-Site Request Forgery) is when an attacker tricks a victim's browser into sending a request to a site the victim is already authenticated on. The browser automatically attaches the session cookie to every request it makes to that domain. So if the victim visits an attacker controlled page while logged into a vulnerable site, the attacker can silently fire off requests as that victim without them knowing.
 
This lab had no CSRF protection at all. No token, no origin check, nothing.
 
### What I Did
 
Logged in as wiener, changed the email, and caught the request in Burp Proxy history. The request looked like this:
 
```
POST /my-account/change-email
 
email=test@example.com
```
 
No CSRF token anywhere. No special headers required. Just a POST with an email parameter.
 
From there I opened the exploit server, built the following HTML payload, and stored it:
 
```html
<form method="POST" action="https://LAB-ID.web-security-academy.net/my-account/change-email">
    <input type="hidden" name="email" value="csrf12345@attacker.net">
</form>
 
<script>
document.forms[0].submit();
</script>
```
 
The form targets the vulnerable endpoint with the forged email value. The script auto-submits it the moment the page loads. I verified it worked with View Exploit first, confirmed the email changed, then delivered to victim. Lab solved.
 
### Why This Works
 
The server receives a POST to `/my-account/change-email` with a valid session cookie and an email parameter. It has no way of knowing whether the request came from the real account page or from a random page the victim visited. It just processes it.
 
The browser did exactly what it is designed to do: send cookies with every request to the matching domain. That behaviour is what CSRF exploits.
 
---
 
