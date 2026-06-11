 
## Lab 03: Traversal Sequences Stripped Non-Recursively
 
**Technique:** Nested traversal sequences to survive a single-pass filter
**Target:** `/etc/passwd`
**Status:** Solved
 
### Context
 
The application stripped `../` from the input before using it. This is a step up from Lab 02 because it addresses the relative traversal case. The problem is the filter only ran once. It found `../` and removed it, but did not check whether removing it created a new traversal sequence.
 
### What I Did
 
Intercepted the image request and modified it with a nested payload:
 
```
GET /image?filename=....//....//....//etc/passwd HTTP/2
```
 
Sent through Repeater. Server returned `/etc/passwd`.
 
### Why It Works
 
The payload `....//` is constructed so that after the single-pass filter removes `../` from the middle, what remains is `../`:
 
```
....//  
  ^^        the filter removes this ../ in the middle  
..  /       leaving this, which is ../
```
 
More precisely: `....//` contains `../` starting at position two. After the filter strips it once, you are left with `../`. The outer dots and the remaining slash reconstruct a valid traversal sequence.
 
A proper fix requires running the sanitisation repeatedly until no traversal sequences remain, or better yet validating the resolved path rather than trying to sanitise the input.
