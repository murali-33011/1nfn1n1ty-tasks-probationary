
## Lab 01: DOM XSS Using Web Messages
 
**Technique:** DOM XSS via insecure postMessage handler

**Sink:** `innerHTML`

 
### Context
 
The `postMessage()` API is a browser mechanism that allows scripts from different origins to communicate with each other. A page can listen for incoming messages using a `message` event listener and then do something with the data it receives. The vulnerability here is that the page took whatever it received through `postMessage` and dropped it into the DOM using `innerHTML` without any sanitisation.
 
Because `innerHTML` parses and renders HTML, any HTML you inject gets treated as real markup by the browser. Injecting event-based payloads like `onerror` on an invalid image element is the standard approach since `innerHTML` does not execute `<script>` tags directly.
 
The other important detail is the origin validation issue. The `postMessage()` call used `'*'` as the target origin, meaning the message would be accepted from any origin. A page that accepts messages from anyone and puts them straight into `innerHTML` is the complete picture of this vulnerability.
 
### The Payload
 
```html
<iframe src="https://LAB-ID.web-security-academy.net/"
onload="this.contentWindow.postMessage('<img src=1 onerror=print()>','*')">
</iframe>
```
 
I hosted this on the exploit server. When the victim loads the exploit page, the iframe opens the target application. Once the iframe finishes loading, the `onload` handler fires and calls `postMessage` to send the malicious HTML string to the target page.
 
The target page receives the message, takes the data, and writes it into the DOM with `innerHTML`. The browser parses the injected `<img>` tag, tries to load an image from `src=1` which is not a valid URL, the load fails, and `onerror` fires calling `print()`.
 
### Why It Works
 
The vulnerability has two parts that together make it exploitable. The page accepts messages from any origin rather than validating that the sender is trusted. And the page uses `innerHTML` to process the received data rather than treating it as plain text. Either one alone would be less dangerous. Both together mean anyone can send HTML to the page and have it rendered and executed.
 
