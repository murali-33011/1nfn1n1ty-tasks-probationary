## Lab 04: DOM XSS in innerHTML Sink Using location.search Source
 
**Technique:** DOM Based XSS
**Sink:** `innerHTML`
**Source:** `location.search`
**Status:** Solved
 
### Context
 
The vulnerable script:
 
```javascript
function doSearchQuery(query) {
    document.getElementById('searchMessage').innerHTML = query;
}
var query = (new URLSearchParams(window.location.search)).get('search');
if(query) {
    doSearchQuery(query);
}
```
 
The search term is pulled from the URL and written into the page using `innerHTML`. This is a different sink than `document.write` and it behaves differently.
 
### Why Script Tags Do Not Work Here
 
`innerHTML` does not execute `<script>` tags. This is a browser security decision. When you set `innerHTML`, any `<script>` elements that appear in the string are intentionally not executed. So the Lab 01 payload would silently do nothing here.
 
### The Payload
 
```html
<img src=1 onerror=alert(1)>
```
 
Because `innerHTML` will parse and render other HTML tags even if it blocks scripts, an `img` tag gets created. The `src` is set to `1` which is not a valid image URL. The browser tries to load it, fails, and fires the `onerror` event handler. That handler calls `alert(1)`. Lab solved.
 
### Sink Behaviour Summary
 
Understanding which sink you are dealing with matters because each one has different rules about what it will and will not execute.
 
| Sink | Executes `<script>`? | Executes event handlers? |
|---|---|---|
| `document.write` | Yes | Yes |
| `innerHTML` | No | Yes (via onerror, onload, etc.) |
| `eval()` | Yes | Yes |
| `href` attribute | No | Only javascript: protocol |
 
---
