## Lab 04: User Role Controlled by Request Parameter (JSON Body)
 
**Technique:** Mass assignment via JSON body parameter injection
**Target:** Delete user carlos
**Status:** Solved
 
### Context
 
This lab required a role ID of 2 to access the admin panel. The role was stored server-side but the application had a mass assignment vulnerability in the profile update endpoint. Mass assignment happens when an API endpoint accepts a JSON object and binds all the fields in that object directly to a database model without filtering which fields are allowed to be updated. If the role field is part of the model and the endpoint does not explicitly exclude it, an attacker can update their own role by including it in the request body.
 
### What I Did
 
Logged in as `wiener:peter` and went to the account page. Used the email update feature and intercepted the request in Burp Suite.
 
The request body was JSON:
 
```json
{
    "email": "test@example.com"
}
```
 
The response included the current user data with a `roleid` field showing my role was 1.
 
Sent the request to Repeater and added `"roleid":2` to the JSON body:
 
```json
{
    "email": "test@example.com",
    "roleid": 2
}
```
 
Sent it. The response showed my `roleid` had updated to 2.
 
Navigated to `/admin`. Admin panel loaded. Deleted carlos.
 
### Why It Works
 
The email update endpoint was designed to update the email field. But the server was taking the entire JSON body and applying all fields to the user record without restricting which fields could be modified. By including `roleid` in the request I was able to update a field the application never intended to expose through that endpoint.
 
The fix is to explicitly whitelist only the fields that should be updatable through each endpoint. Any field not on the whitelist should be ignored regardless of whether it appears in the request body.
 
