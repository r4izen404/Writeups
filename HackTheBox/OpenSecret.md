# Writeup - OpenSecret (HackTheBox)

**Máquina:** OpenSecret  
**Plataforma:** HackTheBox  
**IP:** 154.57.164.75:31503

## Resumen

OpenSecret es un challenge de web que consiste en un portal de soporte con tickets. Los JWT se generan desde el cliente con una **SECRET_KEY hardcodeada** en el código JavaScript, exponiendo la flag directamente en el fuente.

---

## 1. Reconocimiento

### Página principal

El portal permite enviar tickets de soporte con nombre, email y descripción.

### Código fuente

Inspeccionando el HTML de la página, en un bloque `<script>` se encuentra:

```javascript
// JWT Secret Key
const SECRET_KEY = "<REDACTED>";
```

La clave secreta usada para firmar los JWT está hardcodeada en texto plano en el lado del cliente, y además tiene formato de flag.

---

## Resumen de la cadena de ataque

| Paso | Técnica | Resultado |
|-------|---------|-----------|
| 1 | Ver código fuente de la página | `SECRET_KEY` hardcodeada en JavaScript |
| 2 | Leer el valor | Flag: `<REDACTED>` |
