## Lab 05: User ID Controlled by Request Parameter
 
**Technique:** Horizontal privilege escalation via IDOR
**Target:** Retrieve API key for carlos
**Status:** Solved
 
### Context
 
IDOR (Insecure Direct Object Reference) is when an application uses a user-controllable value to look up a resource without checking whether the requesting user is authorised to access that specific resource. In this case the account page used the username directly in a URL parameter to determine whose account data to display.
 
### What I Did
 
Logged in as `wiener:peter` and navigated to the account page. The URL contained my username:
 
```
/my-account?id=wiener
```
 
Sent the request to Burp Repeater. Changed the `id` parameter from `wiener` to `carlos`:
 
```
/my-account?id=carlos
```
 
The response returned carlos's account page including his API key in the HTML body. Copied the API key and submitted it.
 
### Why It Works
 
The application fetched account data based solely on the `id` parameter without checking whether the logged-in session matched the requested user. Any authenticated user could access any other user's account page by changing the parameter. The session was used to confirm you were logged in but not to verify you were the correct user for that particular resource.
 
This is horizontal privilege escalation: moving sideways to access data belonging to another user at the same privilege level, as opposed to vertical escalation where you gain a higher role.
 
