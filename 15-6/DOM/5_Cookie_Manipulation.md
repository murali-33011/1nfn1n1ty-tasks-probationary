## Lab 05: DOM-Based Cookie Manipulation
 
**Technique:** DOM-based XSS via cookie poisoning with a two-stage exploit

**Sink:** `innerHTML` reading from a poisoned cookie

 
### Context
 
This lab involved a chain of two separate DOM vulnerabilities that had to be exploited together. The application stored the URL of the last viewed product in a cookie called `lastViewedProduct`. It set this cookie client-side from the current page URL. On the homepage it read this cookie back and inserted it into the DOM using `innerHTML`.
 
The attack surface is: the cookie is set from a URL the user controls, and the cookie value is later read and written to the DOM without sanitisation. If you can get a malicious script into the URL, it ends up in the cookie, and when the homepage loads it, the script executes.
 
The challenge with this attack is timing. You need the victim to first visit the crafted product URL so the cookie gets set, and then visit the homepage so the poisoned cookie gets processed. A single link cannot do both. An iframe with a redirect solves this.
 
### The Payload
 
```html
<iframe src="https://LAB-ID.web-security-academy.net/product?productId=1&'><script>print()</script>"
onload="if(!window.x)this.src='https://LAB-ID.web-security-academy.net';window.x=1;">
</iframe>
```
 
The attack happens in two stages driven by the iframe:
 
Stage one: the iframe loads the crafted product URL. This URL contains `'><script>print()</script>` appended after the product ID. The application stores this full URL in the `lastViewedProduct` cookie. The cookie is now poisoned with an XSS payload.
 
Stage two: the `onload` handler fires. It checks `window.x` to know whether this is the first or second load. On the first load `window.x` is undefined, so the condition is true and the iframe src is redirected to the homepage. `window.x` is then set to 1 to prevent the redirect from happening again.
 
Stage three: the iframe loads the homepage. The homepage reads the `lastViewedProduct` cookie and inserts its value into the DOM with `innerHTML`. The stored payload `'><script>print()</script>` breaks out of whatever element it is placed in and the script executes.
 
### Why the `window.x` Check Matters
 
Without the `window.x` flag the `onload` event would trigger on both loads: the product page load and the homepage load. On the second load it would redirect back to the product page, creating an infinite loop. The flag ensures the redirect only happens once.
 
### The Cookie as an Attack Surface
 
What makes this interesting is that the cookie is the bridge between the two stages. The user-controlled URL gets written into a cookie, the cookie persists between page loads, and then it gets rendered into the page. Cookies are often overlooked as an attack surface because they are generally thought of as something the server writes and the client stores. But when the client itself sets cookies from URL parameters and then uses them as DOM input, they become an exploitable data flow.
 
