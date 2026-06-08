## Lab 02: Stored XSS into HTML Context with Nothing Encoded
 
**Technique:** Stored XSS
**Injection Point:** Blog comment section
**Status:** Solved
 
### Context
 
Stored XSS is the more dangerous variant. Instead of the payload firing once in the attacker's own browser, it gets saved to the database and executes in the browser of every person who views that page afterwards. The attacker does not need to be there. The payload is sitting in the application waiting for victims.
 
### What I Did
 
I opened a blog post, found the comment section, and submitted:
 
```html
<script>alert("youre hacked");</script>
```
 
as the comment. After submitting I reloaded the page. The comment was fetched from the database, inserted into the HTML, and the browser executed the script. Alert fired. Lab solved.
 
### Reflected vs Stored
 
The payload was identical to Lab 01 but the mechanism is completely different.
 
| | Reflected | Stored |
|---|---|---|
| Where payload lives | URL or request parameter | Database |
| Who gets hit | Only whoever sends the crafted request | Every user who views the page |
| Persistence | None | Until the comment is deleted |
| Delivery | Requires tricking a user into clicking a link | Self-contained in the page |
 
Stored XSS is considered more severe because you do not need to socially engineer anyone into clicking anything. The payload sits there and does its job automatically.
 
---
