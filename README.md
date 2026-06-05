# 🔐 Pentesting Labs & Writeups

<div align="center">

# 🛡️ Cybersecurity Learning Journey

Repositorio dedicado a la documentación de laboratorios, máquinas vulnerables y entornos de práctica enfocados en Pentesting, Red Teaming y preparación para certificaciones como CPTS y OSCP.

![CPTS](https://img.shields.io/badge/HTB-CPTS-orange?style=for-the-badge)
![OSCP](https://img.shields.io/badge/OffSec-OSCP-red?style=for-the-badge)
![Writeups](https://img.shields.io/badge/Writeups-Active-success?style=for-the-badge)
![Pentesting](https://img.shields.io/badge/Pentesting-Learning-blue?style=for-the-badge)

</div>

---

## 📖 Sobre este repositorio

Este repositorio recopila mis resoluciones, notas técnicas y metodologías utilizadas durante la explotación de laboratorios y máquinas vulnerables de distintas plataformas de formación en ciberseguridad.

El objetivo principal es consolidar conocimientos prácticos en:

* Enumeración
* Explotación de vulnerabilidades
* Escalada de privilegios
* Active Directory
* Pivoting
* Movimiento lateral
* Post-explotación
* Red Teaming

Todo el contenido ha sido realizado en entornos autorizados y destinados al aprendizaje.

> ⚠️ Uso exclusivamente educativo. No utilices estas técnicas sobre sistemas sin autorización explícita.

---

## 🎯 Objetivos

* Documentar el proceso completo de comprometer sistemas vulnerables.
* Crear una base de conocimiento reutilizable.
* Mejorar habilidades de Pentesting y Red Teaming.
* Preparar certificaciones prácticas como CPTS y OSCP.
* Mantener un registro de técnicas, herramientas y metodologías utilizadas.

---

## 🏆 Plataformas

* 🚩 Hack The Box
* 🎓 TryHackMe
* 🧠 The Hackers Labs
* 🐧 Hack My VM
* 🐳 DockerLabs
* 🖥️ VulnHub
* 🔍 PortSwigger Web Security Academy
* ⚔️ Otras plataformas y laboratorios vulnerables

---

## 📂 Estructura del repositorio

```text
.
├── HackTheBox/
│   ├── Easy/
│   ├── Medium/
│   └── Hard/
│
├── TryHackMe/
│   ├── Offensive/
│   ├── ActiveDirectory/
│   └── Web/
│
├── TheHackersLabs/
│   ├── Linux/
│   ├── Windows/
│   └── ActiveDirectory/
│
├── HackMyVM/
│   ├── Easy/
│   ├── Medium/
│   └── Hard/
│
├── DockerLabs/
│
├── VulnHub/
│
├── PortSwigger/
│
└── Notes/
    ├── ActiveDirectory/
    ├── LinuxPrivesc/
    ├── WindowsPrivesc/
    └── WebExploitation/
```

---

## 🔍 Metodología utilizada

La mayoría de los laboratorios siguen una metodología estructurada:

### 1. Reconocimiento

* Descubrimiento de hosts
* Identificación de servicios
* Detección de tecnologías
* Fingerprinting

### 2. Enumeración

* SMB
* LDAP
* Kerberos
* DNS
* SNMP
* NFS
* RPC
* Aplicaciones web
* Active Directory

### 3. Explotación

* Vulnerabilidades conocidas
* Configuraciones inseguras
* Credenciales expuestas
* Ataques web
* Remote Code Execution

### 4. Escalada de privilegios

#### Linux

* Sudo Misconfigurations
* SUID
* Capabilities
* Cron Jobs
* Docker Abuse
* Kernel Exploits

#### Windows

* Weak Service Permissions
* DLL Hijacking
* Registry Abuse
* Token Abuse
* Credential Dumping
* Scheduled Tasks

### 5. Movimiento lateral

* Pivoting
* Tunneling
* Port Forwarding
* SOCKS Proxies
* SSH Tunnels

### 6. Post-Explotación

* Enumeración avanzada
* Persistencia
* Recolección de credenciales
* Exfiltración de información
* Documentación

---

## 📚 Temáticas cubiertas

### 🌐 Enumeración y Reconocimiento

* Host Discovery
* Port Scanning
* Service Enumeration
* SMB Enumeration
* LDAP Enumeration
* DNS Enumeration
* NFS Enumeration
* SNMP Enumeration
* RPC Enumeration
* Web Enumeration
* Active Directory Enumeration

### 🌍 Explotación Web

* SQL Injection (SQLi)
* Blind SQL Injection
* Cross-Site Scripting (XSS)
* Command Injection
* Server-Side Request Forgery (SSRF)
* XML External Entity (XXE)
* File Inclusion (LFI/RFI)
* Insecure File Uploads
* SSTI
* Authentication Bypass
* IDOR
* CSRF
* Deserialization Vulnerabilities

### 🔐 Ataques a Credenciales

* Brute Force
* Password Spraying
* Credential Stuffing
* Password Cracking
* NTLM Attacks
* Pass-the-Hash
* Pass-the-Ticket
* Kerberoasting
* AS-REP Roasting

### 🐧 Linux Privilege Escalation

* Sudo Abuse
* SUID Binaries
* Linux Capabilities
* Writable Files
* Cron Jobs
* Docker Escapes
* NFS Abuse
* PATH Hijacking
* Kernel Exploitation

### 🪟 Windows Privilege Escalation

* Weak Service Permissions
* Unquoted Service Paths
* Registry Exploitation
* Scheduled Tasks
* Token Manipulation
* DLL Hijacking
* SeImpersonatePrivilege
* Credential Dumping

### 🏢 Active Directory

* BloodHound
* SharpHound
* PowerView
* Kerberoasting
* AS-REP Roasting
* ACL Abuse
* GPO Abuse
* DCSync
* RBCD
* Delegation Attacks
* Lateral Movement
* Domain Privilege Escalation

### 🔄 Pivoting y Movimiento Lateral

* Chisel
* Ligolo-ng
* SSH Tunneling
* SOCKS Proxies
* Port Forwarding
* Multi-Hop Pivoting
* Reverse Tunnels

### 🧰 Post-Explotación

* Persistence
* Credential Harvesting
* Data Exfiltration
* Looting
* OPSEC
* Reporting

---

## 🛠️ Herramientas frecuentes

### Reconocimiento

* Nmap
* Rustscan
* Netdiscover
* Masscan

### Enumeración

* Enum4Linux
* CrackMapExec
* rpcclient
* ldapsearch
* BloodHound

### Web

* Burp Suite
* Gobuster
* Feroxbuster
* ffuf
* SQLMap

### Credenciales

* Hydra
* John The Ripper
* Hashcat
* Kerbrute

### Linux

* LinPEAS
* pspy
* GTFOBins

### Windows

* WinPEAS
* PowerView
* SharpHound
* Mimikatz
* Rubeus

### Pivoting

* Chisel
* Ligolo-ng
* SSH
* Socat

---

## 📈 Progreso

| Plataforma       | Completados |
| ---------------- | ----------- |
| Hack The Box     | 0           |
| TryHackMe        | 0           |
| The Hackers Labs | 0           |
| Hack My VM       | 0           |
| DockerLabs       | 0           |
| VulnHub          | 0           |
| PortSwigger      | 0           |

---

## 🎓 Certificaciones objetivo

* HTB Certified Penetration Testing Specialist (CPTS)
* Offensive Security Certified Professional (OSCP)
* CRTO
* PNPT

---

## ⚖️ Disclaimer

Todo el contenido de este repositorio tiene fines exclusivamente educativos y de investigación en ciberseguridad.

Las técnicas descritas han sido practicadas únicamente en laboratorios, máquinas vulnerables y entornos autorizados. El autor no se responsabiliza del uso indebido de la información aquí expuesta.

---

<div align="center">

### ⭐ Si este repositorio te resulta útil, considera darle una estrella.

Happy Hacking 👨‍💻

</div>
