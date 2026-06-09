## Lab 05: CSRF Where Token Is Tied to Non-Session Cookie
 
**Technique:** CSRF via HTTP response header injection to plant a controlled csrfKey cookie
**Target:** Email change functionality
**Status:** Solved
 
### Context
 
This one was more layered than the previous labs. The application used two separate values for CSRF protection: a `csrf` token in the request body and a `csrfKey` cookie. These two were linked to each other but neither was tied to the actual session cookie. So as long as the `csrf` token and `csrfKey` cookie matched each other, the request passed, regardless of which session was making it.
 
The challenge was that to forge a request on the victim's behalf, the attacker needed to get their own matching `csrfKey` cookie into the victim's browser. This is where the header injection came in.
 
### What I Did
 
Logged in as wiener, caught the email change request:
 
```
POST /my-account/change-email
 
Cookie:
csrfKey=Mqrj4rGeJoqVOG4ygRzMx84K1lio349J
session=iXatNIcMq3sQPGAvwv1KHqU5zPGQ7Mxm
 
email=test1@gmail.com
csrf=MGjGdfxGIgwp94Ii4lnaYgIz3XQHWjYW
```
 
In Repeater I tested each piece. Changing the session cookie caused a logout. Changing the `csrfKey` cookie caused a CSRF failure rather than a session error. This confirmed that session and csrfKey were independent of each other.
 
Then using the carlos account in a separate browser I replaced carlos's `csrfKey` and `csrf` token with wiener's values while keeping carlos's session cookie. Request went through. Confirmed that the csrfKey and csrf token pair was not bound to any specific user session.
 
So now I had a valid matching pair of `csrfKey` and `csrf` token from my own wiener account that I could use against any victim. The only remaining problem was getting the victim's browser to hold my `csrfKey` cookie so that when the forged request fires, the cookie matches the token in the form.
 
I looked at the search functionality of the site. The search term was reflected in a response header, which meant I could inject a newline into the search parameter and append an arbitrary `Set-Cookie` header to the response. This is a response header injection (also called HTTP header injection or CRLF injection).
 
The payload to inject the cookie via the search endpoint:
 
```
/?search=test%0d%0aSet-Cookie:%20csrfKey=Mqrj4rGeJoqVOG4ygRzMx84K1lio349J%3b%20SameSite=None
```
 
`%0d%0a` is a URL-encoded carriage return and line feed (CRLF). When this gets reflected into the response headers, it breaks the line and starts a new header: `Set-Cookie: csrfKey=...`. The `SameSite=None` is needed so the cookie is sent in cross-site requests.
 
The final exploit HTML:
 
```html
<form method="POST" action="https://LAB-ID.web-security-academy.net/my-account/change-email">
    <input type="hidden" name="email" value="victim@attacker.net">
    <input type="hidden" name="csrf" value="MGjGdfxGIgwp94Ii4lnaYgIz3XQHWjYW">
</form>
 
<img src="https://LAB-ID.web-security-academy.net/?search=test%0d%0aSet-Cookie:%20csrfKey=Mqrj4rGeJoqVOG4ygRzMx84K1lio349J%3b%20SameSite=None"
onerror="document.forms[0].submit()">
```
 
Here is what happens when the victim loads this page:
 
1. The browser requests the img src URL, which hits the vulnerable search endpoint with the injected Set-Cookie header in the response.
2. The victim's browser receives and stores the attacker's `csrfKey` cookie for that domain.
3. The image request fails (there is no actual image), triggering the `onerror` handler.
4. `onerror` calls `document.forms[0].submit()` which fires the forged POST.
5. The POST goes to the server with the victim's session cookie (authentication), the attacker's `csrfKey` cookie (just planted), and the attacker's `csrf` token (in the form body).
6. The server checks that the `csrfKey` and `csrf` token match each other, they do, so the request passes.
7. Email changed. Lab solved.
### Why This Works
 
The CSRF protection here was structurally broken from the start because it was not anchored to the session. Tying a CSRF token to a separate cookie rather than to the session means that the protection is only as strong as the attacker's ability to control what cookies the victim holds. The CRLF injection in the search endpoint provided exactly that control. Even without the header injection, the design would be vulnerable to anyone who could get an arbitrary cookie set on the victim's browser through any other means.
 
---
 
