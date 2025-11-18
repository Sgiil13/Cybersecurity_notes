# Índice

- [[#Explicación]]
- [[#Cómo identificar un Endpoint XSS]]
- [[#Finalidad del XSS]]
- [[#Herramientas]]
- [[#Referencias]]

# Explicación

XSS es una vulnerabilidad de seguridad que permite a un atacante inyectar código malicioso(generalmente JavaScript) en una página web. Cuando la víctima accede a la página, el código se ejecuta en su navegador, permitiendo robar cookies, sesiones, datos personales o realizar acciones no autorizadas.

Podemos diferenciar 3 tipos de ataques XSS:

- **Reflected XSS**: El código malicioso se refleja directamente en la respuesta del servidor. El atacante envía una URL maliciosa a la víctima (no es persistente).

- **Stored XSS**: El código se almacena permanentemente en la base de datos (comentarios, mensajes, foros). Se ejecuta cada vez que otros usuarios acceden (persistente).

- **DOM-based XSS**: El código se inyecta modificando el Document Object Model del navegador, sin pasar por el servidor (ejecución en el Client-Site)

A continuación se muestra un ejemplo básico de un payload XSS:

```
<script>alert('XSS')</script>
```

# Cómo identificar un Endpoint XSS

Para localizar posibles Endpoints XSS realizaremos el siguiente proceso:

1. **Identificar puntos de entrada**: Buscamos parámetros GET/POST, formularios o cualquier otra entrada de texto.

2. **Inyectamos payloads de prueba**: Probamos a introducir varios payloads básicos para testear

	- Simple: `<script>alert('XSS')</script>`
	
	- Con evento: `"><img src=x onerror="alert('XSS')">`
	
	- URL encoded: `%3cscript%3ealert('XSS')%3c/script%3e`
	
	- Sin script: `<svg onload=alert('XSS')>`

3. **Analizamos las respuestas**: Buscamos dónde se muestra la respuesta

	- Si se muestra el pop-up con el mensaje 'XSS', ya tenemos un endpoint vulnerable
	
	- En caso contrario, buscamos si la aplicación tiene filtros o validaciones de caracteres como `< > " ' ( )`

4. **Ajustamos los payloads**: En caso de que encontremos qué caracteres estén restringidos, buscamos payloads que puedan bypassear los filtros.

# Finalidad del XSS

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
# Herramientas

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

# Referencias

Lista de payloads XSS: https://github.com/payloadbox/xss-payload-list


