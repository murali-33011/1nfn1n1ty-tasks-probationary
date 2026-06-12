## Lab 03: User Role Controlled by Request Parameter
 
**Technique:** Cookie manipulation to forge admin role
**Target:** Delete user carlos
**Status:** Solved
 
### Context
 
The application used a cookie to track whether the current user was an admin. The cookie value was a simple boolean: `Admin=false` for regular users. The problem is that cookies are stored in the browser and can be modified freely by the user. If the server trusts the cookie value without verifying it server-side against a session or database record, anyone can give themselves whatever role they want.
 
### What I Did
 
Navigated to `/admin` directly. The page rejected me, saying the admin panel was only accessible to administrators.
 
Logged in with the provided credentials `wiener:peter`. Tried `/admin` again. Still rejected.
 
Opened the browser developer tools and went to the cookies section. Found a cookie named `Admin` with the value `false`. Changed it to `true` and refreshed `/admin`.
 
The admin panel loaded. Deleted carlos.
 
### Why It Works
 
The server was reading the `Admin` cookie value and using it directly to make access decisions. There was no server-side verification of whether the current session user actually had admin privileges. The entire access control decision lived in the browser where the user has full control over it.
 
This is a fundamental design flaw. Role and permission information should never be stored in a client-controlled location like a cookie without a cryptographic signature that prevents tampering. Either use server-side session storage or sign the cookie value so modifications are detectable.
 
