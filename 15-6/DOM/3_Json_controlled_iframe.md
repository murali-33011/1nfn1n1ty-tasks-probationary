
## Lab 03: DOM XSS Using Web Messages and JSON.parse
 
**Technique:** DOM XSS via postMessage with JSON-controlled iframe src

**Sink:** `iframe.src` attribute set to `javascript:` URL
 
### Context
 
The application listened for messages, parsed them with `JSON.parse()`, and then acted on them based on a `type` field in the parsed object. One of the available action types was `load-channel`, which took a `url` value from the message and assigned it to the `src` attribute of an iframe on the page.
 
Setting an iframe `src` to a `javascript:` URL executes that JavaScript in the context of the iframe. The application performed no origin validation on incoming messages and no scheme validation on the URL before assigning it to the src attribute.
 
### The Payload
 
```html
<iframe src="https://LAB-ID.web-security-academy.net/"
onload='this.contentWindow.postMessage("{\"type\":\"load-channel\",\"url\":\"javascript:print()\"}","*")'>
</iframe>
```
 
The message sent was a JSON string:
 
```json
{"type":"load-channel","url":"javascript:print()"}
```
 
The target application parsed this JSON, matched the `type` of `load-channel` in its switch statement, and assigned `javascript:print()` to the iframe's src attribute. The browser loaded the iframe with a `javascript:` URL and `print()` executed.
 
### The Difference from Lab 01 and 02
 
Lab 01 used `innerHTML` as the sink. Lab 02 used `location.href`. This lab used `iframe.src`. Different sinks but the same pattern: user controlled data flowing into a dangerous property without validation. The `iframe.src` sink is worth remembering because it is less obvious than `innerHTML` and often overlooked when reviewing code.
 
The use of JSON parsing in the middle also shows that sanitisation at the surface level is not enough. The data goes through a parsing step that transforms it, and the dangerous assignment happens on the parsed output. Any review of this code needs to follow the data through transformations to see where it eventually lands.
 
