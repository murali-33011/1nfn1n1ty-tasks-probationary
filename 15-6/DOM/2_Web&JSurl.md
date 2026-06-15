
## Lab 02: DOM XSS Using Web Messages and a JavaScript URL
 
**Technique:** DOM XSS via postMessage with `javascript:` URL bypass

**Sink:** `location.href`

 
### Context
 
This lab had a web message handler that took the received data and assigned it to `location.href`. Assigning a `javascript:` URL to `location.href` executes the JavaScript. The application knew this was a risk and tried to defend against it by checking whether the received string contained `http:` or `https:` before using it.
 
The flaw is that the check only looked for the presence of those substrings anywhere in the string. It did not verify that the string actually started with `http://` or `https://`. So a payload that started with `javascript:` but also contained `http:` somewhere later would pass the check.
 
JavaScript comments are the natural way to include arbitrary text after executable code without affecting execution. Anything after `//` on a line is ignored by the JS engine.
 
### The Payload
 
```html
<iframe src="https://LAB-ID.web-security-academy.net/"
onload="this.contentWindow.postMessage('javascript:print()//http:','*')">
</iframe>
```
 
The payload sent to the target page was `javascript:print()//http:`. The validation check scanned for `http:` and found it at the end inside the comment. Check passed. The value was assigned to `location.href`. The browser saw a `javascript:` URL and executed `print()`. The `//http:` part was never evaluated because it was in a comment.
 
### The Wider Point
 
This is the same class of bypass as comment-based SQL injection or URL parameter bypass. Filters that check for the presence of a string rather than validating the structure of the input can be defeated by including the expected string in a position where it has no functional effect. The correct defence is to verify the URL scheme strictly: accept only strings that begin with `https://` and reject everything else, not strings that contain `https:` somewhere.
 
