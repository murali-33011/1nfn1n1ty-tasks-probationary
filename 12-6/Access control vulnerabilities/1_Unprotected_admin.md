## Lab 01: Unprotected Admin Functionality
 
**Technique:** Information disclosure via robots.txt
**Target:** Delete user carlos
**Status:** Solved
 
### Context
 
Access control vulnerabilities are some of the most straightforward to exploit when they exist because the application is simply not checking whether you should be allowed to do something. In this case the admin panel existed at a specific URL with no authentication check on it at all. Anyone who knew the URL could access it.
 
The question is how to find the URL. This lab disclosed it through `robots.txt`, which is a file sites publish to tell search engine crawlers which paths they should not index. The irony is that listing paths in `robots.txt` does not actually restrict access to them. It just tells crawlers to ignore them. But it also tells anyone who reads the file exactly where the sensitive paths are.
 
### What I Did
 
Appended `/robots.txt` to the lab URL and read the contents. The `Disallow` directive listed `/administrator-panel` as a path crawlers should not visit.
 
Replaced `/robots.txt` with `/administrator-panel` in the URL bar. The admin panel loaded with no authentication prompt.
 
Deleted carlos.
 
### Why It Works
 
The application assumed that if a path was not linked from anywhere visible, users would not find it. This is security through obscurity and it does not work. `robots.txt` is publicly accessible by design and is one of the first places an attacker checks.
