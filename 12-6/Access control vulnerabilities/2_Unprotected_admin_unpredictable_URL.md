## Lab 02: Unprotected Admin Functionality with Unpredictable URL
 
**Technique:** Information disclosure via client-side JavaScript
**Target:** Delete user carlos
**Status:** Solved
 
### Context
 
This lab is a variation on Lab 01. The admin panel was at an unpredictable URL rather than something guessable like `/admin` or `/administrator-panel`. The intent is that without knowing the URL, an attacker cannot reach it. The flaw is that the URL was embedded in JavaScript sent to every user's browser.
 
### What I Did
 
Opened the browser developer tools and looked through the page source. The JavaScript included a conditional that revealed the admin URL:
 
```javascript
var isAdmin = false;
if (isAdmin) {
    var adminPanelTag = document.createElement('a');
    adminPanelTag.setAttribute('href', '/admin-fulsg9');
    ...
}
```
 
The URL `/admin-fulsg9` was sitting there in the client-side code even though the link was never rendered for non-admin users. I appended it to the lab URL, the admin panel loaded, and I deleted carlos.
 
### Why It Works
 
The application tried to hide the admin link from the UI by only rendering it conditionally. But hiding something from the rendered page is not the same as not sending it to the browser. The browser receives all the JavaScript regardless of what ends up visible. Anything embedded in client-side code is accessible to anyone who opens the developer tools.
 
