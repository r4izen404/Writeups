# Writeup - SpookyPass (HackTheBox)

**Máquina:** SpookyPass  
**Plataforma:** HackTheBox  
**Autor del challenge:** clubby789  
**Clasificación:** Official  

## Resumen

SpookyPass es un challenge de reversing. El binario pide una contraseña y el uso de `strings` revela la contraseña en texto plano dentro del propio ejecutable.

---

## 1. Reconocimiento

### Ejecución inicial

Al ejecutar el binario nos pide una contraseña:

```
$ ./pass
Welcome to the SPOOKIEST party of the year.
Before we let you in, you'll need to give us the password: foo
You're not a real ghost; clear off!
```

### Strings

Usamos `strings` para inspeccionar el binario en busca de datos ocultos:

```bash
strings ./pass
```

```
Welcome to the [1;3mSPOOKIEST[0m party of the year.
Before we let you in, you'll need to give us the password:
s3cr3t_p455_f0r_gh05t5_4nd_gh0ul5
Welcome inside!
You're not a real ghost; clear off!
```

La cadena `s3cr3t_p455_f0r_gh05t5_4nd_gh0ul5` aparece justo después del mensaje de solicitud de contraseña.

---

## 2. Obtención de la flag

Introducimos la contraseña encontrada:

```bash
$ ./pass
Welcome to the SPOOKIEST party of the year.
Before we let you in, you'll need to give us the password: s3cr3t_p455_f0r_gh05t5_4nd_gh0ul5
Welcome inside!
<REDACTED>
```

---

## Resumen de la cadena de ataque

| Paso | Técnica | Resultado |
|-------|---------|-----------|
| 1 | Ejecutar el binario | Identificar que pide una contraseña |
| 2 | `strings ./pass` | Contraseña en texto plano: `s3cr3t_p455_f0r_gh05t5_4nd_gh0ul5` |
| 3 | Introducir la contraseña | Flag: `<REDACTED>` |
