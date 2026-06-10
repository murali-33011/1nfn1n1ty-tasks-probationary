## Lab 05: Multistep Clickjacking
 
**Technique:** Multistep Clickjacking across two clicks
**Target:** Delete account button and subsequent confirmation dialog
**Status:** Solved
 
### Context
 
Some actions require multiple user confirmations deliberately as a defence. In this lab, deleting an account shows a confirmation dialog after the initial button click. The idea is that even if the first click was accidental or hijacked, the victim would notice something is wrong before confirming.
 
Clickjacking handles this by hijacking both clicks. You place two decoy elements: one over the initial delete button, one over the confirmation yes button. The victim clicks the first, sees another prompt on the decoy page, clicks the second, and the account is deleted.
 
This requires the attacker to know the layout of both the initial page and the confirmation dialog in advance, which is easy to determine by just visiting the page normally.
 
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
 
.firstClick, .secondClick {
    position: absolute;
    z-index: 1;
}
 
.firstClick {
    top: 510px;
    left: 50px;
}
 
.secondClick {
    top: 310px;
    left: 200px;
}
</style>
 
<div class="firstClick">Click me first</div>
<div class="secondClick">Click me next</div>
 
<iframe src="https://LAB-ID.web-security-academy.net/my-account"></iframe>
```
 
### Aligning Both Steps
 
Same approach as the earlier labs but done twice. First set the iframe to partially visible and align the first decoy div over the Delete account button. Then also check where the confirmation dialog's yes button appears. The confirmation dialog position is relative to the page, so it can be measured and aligned independently.
 
The decoy text changes between the two elements to give the victim a reason to click again. "Click me first" and "Click me next" creates a sense of a two-step process on the attacker's fake page, which is enough to get both clicks without arousing suspicion.
 
### Why Confirmation Dialogs Alone Are Not Enough
 
A confirmation dialog only protects against accidental clicks if the user can see what they are confirming. Inside a transparent iframe they cannot. They see whatever the attacker has written on the decoy page as context for why they are clicking. The confirmation dialog is invisible and the victim has no idea it exists.
 
### Result
 
Victim clicked the first decoy, triggering the Delete account button inside the iframe. Then clicked the second decoy, confirming the deletion through the hidden dialog. Account deleted. Lab solved.
 
