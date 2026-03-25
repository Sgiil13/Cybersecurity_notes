
# 💡 Explicación

XSS es una vulnerabilidad de seguridad que permite a un atacante inyectar código malicioso(generalmente JavaScript) en una página web. Cuando la víctima accede a la página, el código se ejecuta en su navegador, permitiendo robar cookies, sesiones, datos personales o realizar acciones no autorizadas.

Podemos diferenciar 3 tipos de ataques XSS:

- **Reflected XSS**: El código malicioso se refleja directamente en la respuesta del servidor. El atacante envía una URL maliciosa a la víctima (no es persistente).

- **Stored XSS**: El código se almacena permanentemente en la base de datos (comentarios, mensajes, foros). Se ejecuta cada vez que otros usuarios acceden (persistente).

- **DOM-based XSS**: El código se inyecta modificando el Document Object Model del navegador, sin pasar por el servidor (ejecución en el Client-Site)

A continuación se muestra un ejemplo básico de un payload XSS:

```
<script>alert('XSS')</script>
```

## 1️⃣ Reflected XSS

🔍 **Cómo Identificarlo**: Parámetros de búsqueda, mensajes de error, nombres de usuario reflejados tras login.

1. Introducimos un texto visible (ej. `TEST`)

2. Buscamos ese string en el Código Fuente (`Ctrl + U`) de la respuesta

3. Si lo encontramos, probamos qué caracteres especiales (`< > " '`) permite el servidor sin codificar

### 💀 **Variantes de Explotación**

1. *CONTEXTO HTML SIMPLE* (el texto car entre etiquetas)

```
<script>alert(1)</script>
<img src=x onerror=alert(1)>
```

2. *CONTEXTO DE ATRIBUTO* (el texto cae dentro de un `value="..."` o similar)

```
" onfocus="alert(1)" autofocus="    (Cierras el atributo y usas un evento)
```

3. *CONTEXTO DE JAVASCRIPT* (el texto cae dentro de un bloque `<Script>`)

```
'-alert(1)-'      (Rompes la cadena de texto de JS sin romper la sintaxis)
```

## 2️⃣ Stored XSS

🔍 **Cómo Identificarlo**: Comentarios, campos de perfil de usuario, tickets de soporte, nombres de archivos subidos.

1. Inyectamos el String de prueba en formularios que se guarden

2. Navegamos por al aplicación para localizar donde se muestra ese String.

3. En ese caso, lo buscamos en el Código Fuente y analizamos en qué contexto está
### 💀 **Variantes de Explotación**

1. *CONTEXTO DE ENLACE* (`href`)

```
javascript:alert(1)
```

## 3️⃣ DOM XSS

🔍 **Cómo Identificarlo**: Buscamos localizar los siguientes puntos de entrada: `location.hash` (`#`), `location.search` (`?`), `document.referrer`.

1. No mires el código fuente (`Ctrl + U`), no verás nada. Usa las DevTools (F12).

2. Busca Sinks peligrosos en los archivos `.js`: `innerHTML`, `document.write`, `eval()`, o selectores de jQuery `$()`.

3. Rastrea el flujo de datos desde la URL hasta el Sink.

### 💀 **Variantes de Explotación**

1. *CONTEXTO DEL SINK* `innerHTML`(El navegador no ejecuta etiquetas `<script>`)

```
<img src=x onerror=alert(1)>
```

2. *CONTEXTO SINK jQuery* `$()`: (Si el hash se pasa a un selector, jQuery puede crear elementos)

```
Usa un `<iframe>` para forzar un cambio de hash (`hashchange`) en la víctima.
```


3. *CONTEXTO SINK DEL ATRIBUTO* (El JS modifica el `src` de un script o el `href` de un botón)

```
Cambiar el valor a un script externo o al protocolo `javascript:`.
```


# 🎯 Finalidad del XSS

Si conseguimos un endpoint vulnerable a XSS, podemos extraer información del usuario como por ejemplo:

1. Cookies de sesión: 
```
<script>alert(document.cookie)</script>
```

2. Credenciales y datos sensibles en almacenamiento local:
```
window.localStorage
window.sessionStorage
```

etc...

# ✋🏻 Bypasses

Para bypassear algunas medidas débiles de seguridad, podemos realizar los siguientes bypasses:

- **Mayúsculas**: `<SCRIPT>alert(1)</SCRIPT>`
- **Mixto**: `<ScRiPt>alert(1)</sCrIpT>`
- **URL encoding**: `%3Cscript%3E`
- **Hex encoding**: `\x3cscript\x3e`
- **SVG**: `<svg><script>alert(1)</script></svg>`

Hay más técnicas, pero menos comunes.

# 🔧 Herramientas

La herramienta más conocida es **XSStrike**: https://github.com/s0md3v/XSStrike
### Modo de uso

##### 1. Escaneo básico de URL

- GET request:
```
python xsstrike.py -u "http://example.com/search.php?q=query"
```

- POST request: 
```
python xsstrike.py -u "http://example.com/search.php" --data "q=query"
```


- POST con JSON: 
```
python xsstrike.py -u "http://example.com/api" --data '{"q":"query"}' --json
```

##### 2. Modo Crawling

- Crawling básico:
```
python xsstrike.py -u "http://example.com/" --crawl
```

- Con profundidad especificada (nivel 3)
```
python xsstrike.py -u "http://example.com/" --crawl -l 3
```

# 🔗 Referencias

Lista de payloads XSS: [https://github.com/payloadbox/xss-payload-list](https://github.com/payload-box/xss-payload-list)
