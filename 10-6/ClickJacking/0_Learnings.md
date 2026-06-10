# My Learnings

Clickjacking is one of those attacks that is conceptually simple but takes a bit of thinking to fully appreciate. The idea is that you load a legitimate website inside a transparent iframe on a page you control, then place fake visible elements underneath it so when the victim clicks your decoy text, their click actually lands on whatever is hidden in the iframe behind it. The victim thinks they are clicking one thing. The browser registers a click on something completely different.

What makes it work is that the browser does not warn you when you click something inside an iframe. It just processes the click. The session cookie is already there, the CSRF token is already in the form, everything is already authenticated. The attacker does not need to steal anything or forge anything. They just need the user's finger.

---

## What is Clickjacking

Clickjacking is a UI redress attack. The attacker redraws what the user sees while keeping the real interactive elements hidden underneath. The key ingredients are:

An application that can be loaded inside an iframe. If the site has no `X-Frame-Options` header and no `Content-Security-Policy` frame-ancestors directive, it can be framed by anyone.

An action on that page that requires only a click to execute. Deleting an account, changing an email address, submitting a form, confirming a transaction.

A page the attacker controls where they can host the iframe and decoy elements.

The structural setup is always the same. The iframe gets `position: relative`, a fixed width and height, very low opacity so it is invisible, and `z-index: 2` so it sits on top in terms of click priority. The decoy element gets `position: absolute`, coordinates that align it with the target button in the iframe, and `z-index: 1` so it is visible but below the iframe in the stacking order. The victim sees the decoy. The click hits the iframe.

---

## Why CSRF Protection Does Not Help

This is the important thing I understood from Lab 01. CSRF tokens protect against forged cross-origin requests. Clickjacking is not a forged request. The victim's own browser is making a fully legitimate request on an authenticated session. The CSRF token is already in the form inside the iframe, and when the click happens the form submits normally with a valid token. The protection is bypassed not because it was broken but because it was never designed to handle this scenario.

This is also why clickjacking requires framing specifically. If you could just trigger the request without an iframe, you would use CSRF. Clickjacking is what you use when CSRF is blocked but framing is not.

---

## How to Build the Payload

The technique is identical across all labs with small variations.

Start with a visible opacity during development. Set the iframe to `opacity: 0.5` so you can see both the decoy text and the underlying page at the same time. Move the decoy div's `top` and `left` values until it sits exactly over the button you want to hijack. Then drop the opacity to `0.0001` before delivering. You do not use `0` because some browsers handle `opacity: 0` differently and it can interfere with click propagation.

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
<iframe src="https://target.com/sensitive-page"></iframe>
```

That is the skeleton for every basic clickjacking attack. Everything else is a variation on this.

---

## Types of Clickjacking I Have Covered

### Basic Clickjacking

The most straightforward case. Load the page, overlay a decoy, victim clicks it. No extra tricks needed. Works when the page has no frame protections at all. Lab 01 showed this works even when CSRF tokens are present because as explained above, they are irrelevant here.

### URL Parameter Prefill

Some applications let you prepopulate form fields via URL query parameters. This is actually useful legitimate functionality for deep linking and user experience but it becomes a clickjacking amplifier. Instead of just tricking the victim into clicking a button that does one fixed thing, you control what value gets submitted.

```
https://target.com/my-account?email=attacker@evil.com
```

Load this in the iframe and the email field is already filled in with the attacker's value before the victim does anything. The victim just has to click Update.

### Bypassing Frame Busters with Sandbox

Applications sometimes try to defend against framing using JavaScript. The typical frame buster checks if the page is the top window and redirects if it is not. Something like `if (top !== self) { top.location = self.location; }`.

The bypass is the `sandbox` attribute on the iframe element. A sandboxed iframe blocks JavaScript execution by default. The frame buster script never runs. The page stays inside the iframe.

The catch is that `sandbox` also blocks form submission by default. You need to re-enable it with `allow-forms`. This one flag selectively restores form submission while keeping JavaScript disabled. The frame buster stays dead.

```html
<iframe sandbox="allow-forms" src="https://target.com/page"></iframe>
```

That is the entire bypass. One attribute, one flag.

### Clickjacking to Trigger DOM XSS

Some XSS payloads only execute after a user clicks something. They are called click-triggered or self-XSS and are generally dismissed as unexploitable because you cannot force anyone to click the right thing. Clickjacking solves exactly that problem.

You inject the XSS payload into a URL parameter that populates a form field:

```
/feedback?name=<img src=1 onerror=print()>&email=x@x.com&subject=x&message=x
```

Load that in the iframe. Everything is pre-filled. The victim just needs to click Submit. The XSS fires on submission.

This is the combination that makes previously dismissed self-XSS findings into real vulnerabilities. If a page is frameable and has click-triggered self-XSS, the two together are exploitable.

### Multistep Clickjacking

Some actions have confirmation dialogs as a second layer of protection. The assumption is that even if the first click was hijacked, the victim will notice something is wrong before confirming.

Multistep clickjacking hijacks both clicks. You place two decoy elements: one aligned over the initial button, one aligned over the confirmation dialog's confirm button. The decoy text for the second element gives the victim a fake reason to click again so it does not feel suspicious.

```html
<div class="firstClick">Click me first</div>
<div class="secondClick">Click me next</div>
```

The victim clicks first decoy, hidden button fires. They see your fake prompt for a second click. They click second decoy, confirmation dialog confirms. Action is complete. The confirmation dialog provides no real protection because the victim never saw it was there.

---

## The Defences and Why They Work

**X-Frame-Options** is the simple solution. Setting the header to `DENY` or `SAMEORIGIN` tells the browser not to render the page inside an iframe from a different origin. Browser enforced. JavaScript frame busters are bypassable as shown in Lab 03. This header is not.

**Content-Security-Policy with frame-ancestors** is the modern version. More flexible because you can specify exactly which origins are allowed to frame the page. Better than X-Frame-Options which only has two modes.

**JavaScript frame busters** are unreliable as a standalone defence. They can be bypassed with the sandbox iframe technique. They should never be the only protection.

The pattern across all five labs was the same: the applications either had no framing protection at all, or had a JavaScript-based protection that was bypassed. Proper HTTP headers would have stopped all of them.

---

## The Alignment Problem

The hardest practical part of clickjacking is getting the decoy positioned exactly over the right button. Pixel-perfect alignment matters because if you are even slightly off the victim's click misses the target.

The workflow I used across all labs:

1. Set `opacity: 0.5` on the iframe during testing
2. View the exploit to see both layers at once
3. Adjust `top` and `left` on the decoy div until it lines up with the button
4. For multistep attacks, check the position of the confirmation button separately since it appears on a different state of the page
5. Set opacity to `0.0001` and deliver

The `top` and `left` values will be different for every application and every button depending on the page layout. There is no universal value. You have to measure it each time.

---

## Summary

| Lab | What Made It Different | Technique |
|---|---|---|
| 01 | CSRF token present, did not matter | Basic overlay |
| 02 | Email field prepopulated via URL parameter | URL param prefill |
| 03 | Frame buster script present | Sandbox iframe bypass |
| 04 | Click-triggered DOM XSS | Clickjacking + XSS chain |
| 05 | Confirmation dialog present | Two decoy elements |

The core of every single one of these was the same iframe overlay structure. What changed was either what made the page harder to frame or what the click needed to do once it landed.

---
