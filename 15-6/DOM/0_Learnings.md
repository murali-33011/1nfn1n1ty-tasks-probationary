# DOM-Based Vulnerabilities : My Learnings

DOM-based vulnerabilities are a category I find genuinely interesting because they exist entirely in the client side. The server is not involved in the attack at all. The vulnerable code runs in the victim's browser, processes attacker-controlled data, and causes harm without a single malicious byte ever touching the server. This makes them harder to detect with server-side tools and WAFs, and they require a different way of thinking compared to reflected or stored XSS.



## What Are DOM-Based Vulnerabilities

The Document Object Model is the browser's internal representation of a web page. JavaScript can read and manipulate the DOM in real time. DOM-based vulnerabilities happen when JavaScript takes data from a source that the attacker controls and passes it to a sink that does something dangerous with it, without properly validating or sanitising it along the way.

The key difference from server-side vulnerabilities: the data never goes to the server. The attack lives entirely within the JavaScript running on the page. The server sent out perfectly fine HTML. The browser's own JavaScript then poisoned it.



## Sources and Sinks

Understanding DOM vulnerabilities comes down to understanding sources and sinks and tracing the data flow between them.

**Source** is where attacker-controlled data enters the JavaScript.

Common sources:
```
location.search         URL query string (?param=value)
location.href           the entire URL
location.hash           the fragment (#value)
document.referrer       the referring page URL
document.cookie         cookie values
postMessage data        messages received from other windows or iframes
localStorage / sessionStorage
```

**Sink** is where the data ends up in a way that can cause harm.

Common sinks and what they can lead to:
```
innerHTML               XSS via injected HTML
outerHTML               XSS via injected HTML
document.write()        XSS via injected HTML
location.href           XSS via javascript: URL, or open redirection
location.assign()       open redirection or XSS
iframe.src              XSS via javascript: URL
eval()                  direct JS execution
setTimeout()            JS execution if passed a string
jQuery .html()          XSS via injected HTML
element.setAttribute()  XSS depending on which attribute
```

A DOM vulnerability exists wherever data flows from a source to a sink without sufficient validation in between.



## DOM XSS via postMessage

The `postMessage()` API allows different windows, tabs, and iframes to send messages to each other across origins. A page registers a listener for incoming messages:

```javascript
window.addEventListener('message', function(e) {
    document.getElementById('output').innerHTML = e.data;
});
```

Two things make this vulnerable. The page does not check `e.origin` to verify the sender is trusted. And it uses `innerHTML` as the sink. Anyone can open the target page in an iframe, send it a message with a malicious HTML payload, and the page will render it.

The exploit structure is always an iframe that sends the postMessage after it finishes loading:

```html
<iframe src="https://target.com/"
onload="this.contentWindow.postMessage('<img src=1 onerror=print()>','*')">
</iframe>
```

The `'*'` in the postMessage call means send to any origin. If the target page also accepts from any origin there is no barrier at all.

### Sinks postMessage Data Can Reach

**innerHTML** : classic HTML injection. Cannot directly execute `<script>` tags but event-based payloads work:
```
<img src=1 onerror=alert(1)>
<svg onload=alert(1)>
```

**location.href** : if the received message gets assigned to location.href, supply a `javascript:` URL. But often there will be a filter checking for `http:` or `https:`. The bypass is to include those strings in a comment after the payload:
```
javascript:print()//http:
```
The filter finds `http:` and passes the value through. The browser executes `javascript:print()`. The comment makes `http:` irrelevant to execution.

**iframe.src** : if the message controls a property that gets set as an iframe src, the same `javascript:` URL technique applies:
```json
{"type":"load-channel","url":"javascript:print()"}
```

### Origin Validation

The correct defence is checking `e.origin` in the message handler before doing anything with `e.data`:

```javascript
window.addEventListener('message', function(e) {
    if (e.origin !== 'https://trusted-site.com') return;
    // safe to proceed
});
```

Without this check any page can send messages and have them processed.



## DOM-Based Open Redirection

Open redirection happens when JavaScript reads a URL from a user-controlled source and redirects the browser to it without validating that the destination is safe.

```javascript
var returnUrl = /url=(https?:\/\/.+)/.exec(location);
if (returnUrl)
    location.href = returnUrl[1];
```

This regex extracts a URL from the query string and assigns it to `location.href`. There is no check that the destination is on the same domain. Supplying any external URL as the `url` parameter redirects the victim there.

```
https://target.com/post?postId=1&url=https://attacker.com
```

Open redirection is often rated low severity in isolation but it is a useful tool in attack chains. Phishing campaigns use trusted domain names as the starting URL knowing the redirect will take victims to a malicious page. It also helps bypass SSRF filters and open redirection chains can be used to deliver XSS in some configurations.

The fix is to validate that the destination URL is on an allowed domain before redirecting, or to use relative paths only.



## DOM-Based Cookie Manipulation

This one is less obvious. Some applications write the current page URL into a cookie client-side and then read that cookie back later to render something in the DOM. The URL is user-controlled. The cookie inherits whatever is in the URL. The DOM reads the cookie without sanitisation.

```javascript
// sets cookie from current URL
document.cookie = 'lastViewedProduct=' + window.location + '; SameSite=None; Secure';

// later on another page reads it into the DOM
document.getElementById('link').innerHTML = '<a href="' + getCookieValue('lastViewedProduct') + '">Last viewed</a>';
```

The attack is in two stages because the cookie needs to be set before it can be read.

Stage one: send the victim to a crafted URL that includes the XSS payload so it gets written into the cookie.

Stage two: send the victim to the page that reads the cookie and renders it.

An iframe handles both stages automatically:

```html
<iframe src="https://target.com/product?productId=1&'><script>print()</script>"
onload="if(!window.x)this.src='https://target.com';window.x=1;">
</iframe>
```

The `window.x` flag prevents an infinite redirect loop. On first load (product page): `window.x` is undefined, condition is true, src is changed to the homepage and `window.x` is set to 1. On second load (homepage): `window.x` is already 1, condition is false, no redirect. The homepage reads the poisoned cookie and renders the payload.

Cookies are an overlooked source in DOM analysis. When an application writes client-side cookies from URL parameters and reads them back into the DOM, that entire chain is an attack surface.



## How to Find DOM-Based Vulnerabilities

### Manual Review

Open the browser developer tools and look through the JavaScript source files. Search for sink function calls:

```
innerHTML
outerHTML
document.write
location.href
location.assign
location.replace
eval
setTimeout
setInterval
iframe.src
```

For each one, trace backwards to find where the data being passed to it came from. If you can trace a line from `location.search`, `document.cookie`, `postMessage`, or any other user-controlled source to that sink, you have a candidate.

### postMessage Specifically

In the browser console you can monitor all messages being received on a page:

```javascript
window.addEventListener('message', function(e) {
    console.log(e.origin, e.data);
});
```

Or look for existing `message` event listeners in the source. Check what the handler does with `e.data`. If it goes into `innerHTML`, `location.href`, or another sink without validation, test it.

### DOM Invader

Burp Suite's DOM Invader browser extension automates a lot of this. It instruments the browser's JavaScript engine to track taint flows from sources to sinks and highlights them in the UI. Useful for quickly identifying which parameters are reaching which sinks without having to read all the code manually.



## Sink Behaviour Reference

Not all sinks execute the same payloads. Knowing which payload works in which sink saves a lot of trial and error.

| Sink | Executes `<script>` | Event handlers | `javascript:` URL |
|||||
| `innerHTML` | No | Yes (`onerror`, `onload`, etc.) | No |
| `outerHTML` | No | Yes | No |
| `document.write()` | Yes | Yes | Yes |
| `location.href` | No | No | Yes (on assignment) |
| `iframe.src` | No | No | Yes (browser loads it) |
| `eval()` | Yes | Yes | Yes |
| `jQuery .html()` | No | Yes | No |

When the sink is `innerHTML` you need event-based payloads because script tags are blocked:
```html
<img src=1 onerror=payload>
<svg onload=payload>
```

When the sink is `location.href` or `iframe.src` you use the `javascript:` protocol:
```
javascript:payload
```



## Key Patterns to Remember

**Filter bypass with comments in `javascript:` URLs**

If a check looks for `http:` or `https:` anywhere in the string:
```
javascript:print()//http:
```
The comment makes the substring check pass while the browser only executes what comes before the comment.

**Two-stage attacks with iframes**

When the exploit requires visiting two different pages in sequence, an iframe with a redirect on the `onload` event handles it. The `window.x` flag prevents infinite loops.

**JSON.parse does not sanitise**

If a postMessage handler parses the received data with `JSON.parse()` before processing it, you send a JSON string rather than plain text. The parsing step does not make the data safe. Whatever properties the parsed object has can still reach dangerous sinks.

**Data through transformations**

Trace data through every transformation it passes through on the way to the sink. `JSON.parse()`, regex extraction, string concatenation, substring operations. At each step ask where the data came from originally and whether the transformation removed the dangerous parts or just changed the format.



## Summary

| Lab | Source | Sink | Bypass Needed |
| postMessage innerHTML | postMessage data | `innerHTML` | None, direct injection |
| postMessage location.href | postMessage data | `location.href` | `http:` substring in comment |
| postMessage iframe.src | postMessage JSON field | `iframe.src` | None, direct `javascript:` URL |
| Open redirection | URL parameter | `location.href` | None |
| Cookie manipulation | URL parameter via cookie | `innerHTML` | Two-stage iframe exploit |

