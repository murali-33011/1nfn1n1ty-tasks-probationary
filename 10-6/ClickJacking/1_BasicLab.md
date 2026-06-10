## Lab 01: Basic Clickjacking with CSRF Token Protection
 
**Technique:** Basic Clickjacking
**Target:** Delete account button
**Status:** Solved
 
### Context
 
Clickjacking is when you load a legitimate page inside a transparent iframe and layer a fake decoy element underneath it at exactly the right position so when the victim clicks what they think is your decoy, they are actually clicking something on the real page inside the iframe. The browser processes the click on the iframe content, not on the visible text.
 
The interesting part of this lab is that the action was protected by a CSRF token and it still did not matter. CSRF tokens only defend against requests being forged from a different origin. Clickjacking is not forging a request, it is tricking the user's own browser into making a legitimate one. The CSRF token gets included automatically because the form submission happens on the real page inside the iframe. The protection is irrelevant here.
 
The application had no `X-Frame-Options` header and no `Content-Security-Policy` restricting framing. That is what makes this possible.
 
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
    top: 510px;
    left: 50px;
    z-index: 1;
}
</style>
 
<div>Click me</div>
 
<iframe src="https://LAB-ID.web-security-academy.net/my-account"></iframe>
```
 
### How to Align It
 
Before delivering to the victim, you first test with the opacity set to something visible like 0.5 so you can see both the decoy text and the iframe content at the same time. You adjust the `top` and `left` values on the div until the decoy text sits directly over the Delete account button. Then you drop the opacity back down to near zero (0.0001 rather than 0 because some browsers ignore `opacity: 0` for click events).
 
The div has `z-index: 1` and the iframe has `z-index: 2`. The iframe sits on top visually but because it is transparent the victim sees the div. The click lands on the iframe.
 
### Result
 
Victim clicked the decoy. Click passed through to the Delete account button inside the iframe. Account deleted. Lab solved.
 
