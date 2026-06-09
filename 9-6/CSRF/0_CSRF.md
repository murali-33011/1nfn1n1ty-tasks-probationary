# My Learnings

CSRF is one of those vulnerabilities that feels simple until you start pulling at the edge cases. The core idea is that a browser automatically attaches cookies to every request it makes to a matching domain. If a user is logged into a site and visits a page the attacker controls, the attacker can silently fire off requests to that site on the user's behalf, and the server cannot tell the difference.

The server sees a valid session cookie. The server sees the right endpoint being hit. It processes the request. The user never knew it happened.

---

## What is CSRF

A website trusts requests that come with the right session cookie. The browser trusts the user's intent. CSRF sits in the gap between those two assumptions.

The attack flow is:

1. Victim is logged into a site with an active session cookie
2. Victim visits an attacker controlled page (via phishing link, malicious ad, whatever)
3. That page contains HTML that fires a request to the target site
4. The browser attaches the victim's session cookie automatically
5. The server processes the request thinking it came from the user

The attacker does not need to steal the session cookie. They never see it. The browser just uses it for them.

The most common payload is a form with hidden fields that auto-submits on page load:

```html
<form method="POST" action="https://target.com/change-email">
    <input type="hidden" name="email" value="attacker@evil.com">
</form>

<script>
document.forms[0].submit();
</script>
```

---

## What Damage Can CSRF Do

Anything a legitimate user can do, an attacker can force them to do without their knowledge:

- Change email address (account takeover vector if the attacker then does a password reset)
- Change password
- Transfer funds
- Delete account data
- Modify settings or permissions
- Make purchases

The severity depends entirely on what authenticated actions the application exposes.

---

## CSRF Defenses and How They Break

This is the important part. Most real-world CSRF vulnerabilities are not about a complete absence of protection. They are about protection that was implemented with a gap somewhere.

### No Protection at All

The simplest case. The application does not check anything beyond the session cookie. Any cross-site form submission works. You just write the form, auto-submit it, done.

This is Lab 01.

### CSRF Tokens

The standard fix for CSRF is to include a secret token in every state-changing request. The server generates it, embeds it in the form, and validates it when the form is submitted. Because the attacker cannot read the token from the victim's browser (due to same-origin policy), they cannot include a valid one in a forged request.

But tokens can be implemented incorrectly in several ways.

**Token only validated on POST, not GET (Lab 02)**

The server checks the CSRF token when the request method is POST. But the same endpoint also accepts GET requests with the parameters in the query string, and GET requests are never checked. The attacker just uses a GET-based form (which is the HTML default when no method attribute is specified) and bypasses the entire protection.

```html
<form action="https://target.com/change-email">
    <input type="hidden" name="email" value="attacker@evil.com">
</form>
```

No `method="POST"` means the form submits as GET. No token needed.

**Token only validated when present (Lab 03)**

The server validates the token if it exists in the request. If the `csrf` parameter is simply absent from the request body, the validation logic never runs and the request goes through. The attacker just omits the token from the form entirely.

The check probably looks something like "if token is present, validate it" without the accompanying "else, reject the request."

**Token not tied to user session (Lab 04)**

The server generates CSRF tokens but they come from a global pool rather than being generated per session. A valid token from user A's account can be used in a request made under user B's session. The attacker creates their own account, generates a valid token, embeds it in the exploit, and the victim's browser submits it. The victim's session cookie authenticates the request, the attacker's token satisfies the CSRF check.

This is why CSRF tokens must be bound to the specific session that generated them.

**Token tied to a non-session cookie (Lab 05)**

This is the most sophisticated variant. The application uses a `csrfKey` cookie that is paired with a `csrf` body token. They are linked to each other but neither is linked to the actual session cookie. If the `csrfKey` and `csrf` token match, the server accepts the request regardless of which session is active.

The attacker uses their own account to generate a valid `csrfKey` and `csrf` pair. The problem is getting their `csrfKey` cookie into the victim's browser. This required a second vulnerability: CRLF injection in a search parameter that was reflected into response headers. By injecting a `Set-Cookie` header via the search endpoint, the attacker plants their `csrfKey` cookie in the victim's browser before the forged form submits.

---

## How I Used Burp Suite for Each Lab

The workflow was consistent across all five labs:

**Step 1: Identify the target request**

Log in with the provided credentials, perform the action you want to forge (email change in all these labs), then open Proxy > HTTP History in Burp and find the request.

**Step 2: Test the protection**

Send the request to Repeater. Try modifying or removing the CSRF token if one is present. Try changing the request method from POST to GET. Try removing the token parameter entirely. Each test tells you something about how the validation works and where the gap is.

**Step 3: Build the payload**

Once you understand exactly what the server checks, build the minimum HTML needed to replicate the request from a cross-origin page. The exploit server provided by the lab is where you host the payload.

**Step 4: Verify with View Exploit**

Click View Exploit on the exploit server to test the payload in your own session first. Confirm the action actually happened (email changed, etc.) before delivering to the victim.

**Step 5: Deliver to Victim**

Click Deliver to Victim. The simulated victim visits the exploit page with their session active and the forged request fires.

---

## Payload Patterns

**Basic POST (Labs 01, 03, 04)**

```html
<form method="POST" action="https://target.com/change-email">
    <input type="hidden" name="email" value="attacker@evil.com">
</form>
<script>document.forms[0].submit();</script>
```

**GET based (Lab 02)**

```html
<form action="https://target.com/change-email">
    <input type="hidden" name="email" value="attacker@evil.com">
</form>
<script>document.forms[0].submit();</script>
```

No `method` attribute means the form defaults to GET. Parameters go in the query string.

**POST with hardcoded token (Lab 04)**

```html
<form method="POST" action="https://target.com/change-email">
    <input type="hidden" name="email" value="attacker@evil.com">
    <input type="hidden" name="csrf" value="attacker_generated_valid_token">
</form>
<script>document.forms[0].submit();</script>
```

**Cookie injection then POST (Lab 05)**

```html
<form method="POST" action="https://target.com/change-email">
    <input type="hidden" name="email" value="attacker@evil.com">
    <input type="hidden" name="csrf" value="attacker_token">
</form>

<img src="https://target.com/?search=x%0d%0aSet-Cookie:%20csrfKey=attacker_key%3b%20SameSite=None"
onerror="document.forms[0].submit()">
```

The `img` tag fires first. The server responds with the injected Set-Cookie header. Cookie is planted in the victim's browser. Image fails to load. `onerror` fires the form submission. The form sends with the victim's session cookie, the attacker's `csrfKey` cookie, and the attacker's `csrf` token in the body.

---

## Why CRLF Injection Works in Lab 05

`%0d%0a` is URL-encoded `\r\n` which is the line terminator in HTTP headers. When the search term is reflected directly into a response header without sanitisation, injecting `\r\n` lets you start a new header line. Everything you write after it becomes a legitimate new response header.

```
Reflected-Search: test
Set-Cookie: csrfKey=attacker_key; SameSite=None
```

The browser processes that `Set-Cookie` header and stores the cookie. The attacker now controls what `csrfKey` value the victim's browser will send.

`SameSite=None` is required because without it browsers will not send the cookie on cross-site requests, which would defeat the whole point.

---

## Summary Table

| Lab | Protection Present | The Gap | Bypass Method |
|---|---|---|---|
| 01 | None | No protection at all | Basic form auto-submit |
| 02 | Token on POST only | GET requests not checked | Switch to GET method form |
| 03 | Token when present | Absent token skips check | Omit the token parameter |
| 04 | Token validated | Token not session-bound | Use own account's valid token |
| 05 | Token + csrfKey cookie pair | Cookie not session-bound | CRLF inject csrfKey cookie into victim's browser |

---

## What Proper CSRF Protection Looks Like

A well implemented CSRF token must satisfy all of the following:

- Tied to the specific user session that generated it
- Cannot be reused across sessions
- Present in every state-changing request
- Validated regardless of whether the parameter is present or absent (missing token should reject, not pass)
- Checked on all HTTP methods that can trigger state changes, not just POST
- If cookie-based, the cookie must also be tied to the session

SameSite cookie attribute is a separate browser-level defence that helps. Setting `SameSite=Strict` or `SameSite=Lax` on the session cookie means the browser will not send it on cross-site requests, which stops most CSRF attack vectors entirely. But it is not a complete replacement for token-based protection because there are edge cases and older browsers that do not support it.

---
