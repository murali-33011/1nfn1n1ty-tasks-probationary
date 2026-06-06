## Lab 04: UNION Injection — Database Version on MySQL and Microsoft
 
**Technique:** UNION SELECT + @@version
**Database:** MySQL 8.0.42
**Status:** Solved
 
### Context
 
Same UNION injection vector but targeting a MySQL or Microsoft SQL Server backend. The key challenge is that comment syntax differs across databases. MySQL requires a space after the double dash (`-- `) or a hash (`#`), while SQL Server uses plain `--`. Multiple trials were needed to find the working combination.
 
### Final Payload
 
```
' UNION SELECT @@version,NULL-- -
```
 
**Full URL:**
```
https://0ab1008703cc0d128017a87a00f0004b.web-security-academy.net/filter?category=%27%20UNION%20SELECT%20@@version,NULL--%20-
```
 
### Payload Breakdown
 
| Token | Purpose |
|---|---|
| `@@version` | A global system variable available in both MySQL and Microsoft SQL Server that returns the database engine version string, not available in Oracle or PostgreSQL |
| `NULL` | Column count placeholder, same reasoning as Lab 03 |
| `-- -` | The trailing space and dash after the double dash ensure the comment is properly terminated in MySQL, MySQL's parser requires whitespace after `--` for it to be treated as a comment, the extra `-` satisfies that requirement after URL decoding |
 
### Why Earlier Attempts Failed
 
Payloads using a bare `#` failed because the hash was URL-encoded to `%23` before reaching the server and the application did not decode it back. SQL Server style `--` without trailing whitespace was not accepted by MySQL's parser. The `-- -` pattern is the reliable cross-tool workaround.
 
### Response Received
 
```
8.0.42-0ubuntu0.20.04.1
```
 
Confirmed MySQL 8.0.42. Lab solved.
 
---
