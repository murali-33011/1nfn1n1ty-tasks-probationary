## Lab 01: Reflected XSS into HTML Context with Nothing Encoded
 
**Technique:** Reflected XSS
**Injection Point:** Search bar
**Status:** Solved
 
### Context
 
Reflected XSS is when user input is taken from the request, immediately reflected back into the HTML response, and executed. The server takes whatever you typed, drops it into the page, and sends it back. No storage, no persistence. It fires once, in that moment, for whoever made the request.
 
### What I Did
 
The page had a search bar. I typed directly into it:
 
```html
<script>alert("youre hacked");</script>
```
 
The server put that into the HTML response without sanitising it. The browser received it, saw a `<script>` tag, and executed it. Alert fired. Lab solved.
 
### Why This Works
 
The application is not encoding or stripping anything. So when my input lands in the HTML it is treated as markup, not as text. The browser parses `<script>` as a real script tag and runs whatever is inside. Nothing is stopping it.
 
The fix would be to HTML-encode the output so `<` becomes `&lt;` and `>` becomes `&gt;`, which would make the browser render it as visible text instead of executing it.
 
---
 
