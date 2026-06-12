 
## Lab 06: User ID Controlled by Request Parameter with Unpredictable User IDs
 
**Technique:** IDOR with GUID enumeration via information disclosure
**Target:** Retrieve API key for carlos
**Status:** Solved
 
### Context
 
This lab is the same IDOR vulnerability as Lab 05 but the application used GUIDs (Globally Unique Identifiers) instead of usernames in the `id` parameter. GUIDs are long random strings that are practically impossible to guess, which is an attempt to make IDORs unexploitable even if the access control check is missing. The flaw is that carlos's GUID was disclosed in a different part of the application.
 
### What I Did
 
Browsed the blog section of the application and found a post authored by carlos. Clicked on his name. The URL of his author profile contained his GUID:
 
```
/blogs?userId=4ab83d4e-7e2f-4811-9f1b-5f1b27234c3e
```
 
Noted the GUID.
 
Logged in as `wiener:peter` and navigated to my account page. Changed the `id` parameter to carlos's GUID:
 
```
/my-account?id=4ab83d4e-7e2f-4811-9f1b-5f1b27234c3e
```
 
The response returned carlos's account page with his API key. Submitted it.
 
### Why It Works
 
Using unpredictable IDs is a defence in depth measure but it is not a substitute for proper access control. If the GUID leaks anywhere, through blog post author links, comments, activity feeds, or any other feature that maps a visible user to their internal ID, the protection is gone. The actual fix is server-side verification that the authenticated session matches the requested user ID, not obscuring the ID format.
 
