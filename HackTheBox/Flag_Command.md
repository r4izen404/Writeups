# Writeup - Dimensional Escape Quest (HackTheBox)

**Máquina:** Dimensional Escape Quest  
**Plataforma:** HackTheBox  
**IP:** 154.57.164.74:32322  
**Autor del challenge:** Xclow3n  
**Clasificación:** Oficial  

## Resumen

Dimensional Escape Quest es un challenge de tipo web/juego de la dificultad Very Easy. El camino consiste en inspeccionar las peticiones de red que realiza la aplicación para descubrir un **comando secreto** devuelto por la API, e introducirlo en la terminal del juego para obtener la flag.

---

## 1. Reconocimiento

Al visitar la página principal nos encontramos con una terminal interactiva con estilo de juego de aventura medieval ("forest maze"). Podemos ejecutar comandos como `help`, `start`, `clear`, `info`, `restart` y `audio`.

### Análisis de red

Al abrir las **Developer Tools** del navegador en la pestaña **Network** y recargar la página, observamos una petición a la API:

```
GET /api/options
```

### Respuesta de `/api/options`

Inspeccionando la respuesta JSON:

```json
{
  "allPossibleCommands": {
    "1": ["HEAD NORTH", "FOLLOW A MYSTERIOUS PATH", "SET UP CAMP"],
    "2": ["...", "...", "..."],
    "3": ["...", "...", "..."],
    "4": ["...", "...", "..."],
    "secret": "Blip-blop, in a pickle with a hiccup! Shmiggity-shmack"
  }
}
```

El campo `secret` contiene el comando oculto que no se muestra en la interfaz del juego.

---

## 2. Ejecución del comando secreto

1. Escribimos `start` en la terminal para iniciar la partida.
2. Introducimos el comando secreto:

```
Blip-blop, in a pickle with a hiccup! Shmiggity-shmack
```

3. El servidor responde con la flag directamente en el mensaje de la terminal.

---

## Resumen de la cadena de ataque

| Paso | Técnica | Resultado |
|-------|---------|-----------|
| 1 | Inspeccionar peticiones de red (DevTools) | Descubrimos endpoint `/api/options` |
| 2 | Leer respuesta JSON de la API | Comando secreto en el campo `secret` |
| 3 | Introducir comando secreto en la terminal | Obtenemos la flag `HTB{...}` |
