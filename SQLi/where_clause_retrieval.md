## Lab 01: SQL Injection in WHERE Clause : Hidden Data Retrieval
 
 **Technique:** Boolean Injection
**Status:** Solved
 
### Context
 
The application filters products by a category value passed in the URL. Internally the query also applies a `released = 1` condition to hide unreleased items. The goal is to make the query return all rows including the hidden ones.
 
### Final Payload
 
```
' OR 1=1--
```
 
**Full URL:**
```
https://0a4000a90410fb4d809e2b7700df0040.web-security-academy.net/filter?category=%27+OR+1=1--
```
 
### Payload Breakdown
 
| Token | Purpose |
|---|---|
| `'` | Closes the string literal the application wraps the category value in |
| `OR 1=1` | Appends a condition that is always true, making the WHERE clause evaluate to true for every row in the table |
| `--` | SQL comment sequence, everything after this including the original `released = 1` filter is ignored by the database engine |
 
### Result
 
All product rows returned including hidden unreleased items. Lab solved.
 
---
