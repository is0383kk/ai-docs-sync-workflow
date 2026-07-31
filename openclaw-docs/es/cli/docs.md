---
read_when:
    - Quieres buscar la documentación activa de OpenClaw desde la terminal
    - Necesita saber a qué API de búsqueda alojada llama la CLI de la documentación
summary: Referencia de la CLI para `openclaw docs` (buscar en el índice de la documentación en vivo)
title: Documentación
x-i18n:
    generated_at: "2026-07-26T05:34:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b0b575f0b76d40a53dd4f79c55fd65969a24eae27e27bd1c46d395f61fe89e42
    source_path: cli/docs.md
    workflow: 16
---

# `openclaw docs`

Busca en el índice de documentación en vivo de OpenClaw desde la terminal.

## Uso

```bash
openclaw docs                       # muestra el punto de entrada de la documentación y un ejemplo de búsqueda
openclaw docs <query...>            # busca en el índice de documentación en vivo
```

| Argumento     | Descripción                                                                        |
| ------------ | ---------------------------------------------------------------------------------- |
| `[query...]` | Consulta de búsqueda de formato libre. Las consultas de varias palabras se unen con espacios y se envían como una sola. |

Sin ninguna consulta, `openclaw docs` muestra la URL del punto de entrada de la documentación y un comando de búsqueda de ejemplo en lugar de ejecutar una búsqueda.

## Ejemplos

```bash
openclaw docs browser existing-session
openclaw docs sandbox allowHostControl
openclaw docs gateway token secretref
```

## Cómo funciona

`openclaw docs` llama a `https://docs.openclaw.ai/api/search` y presenta los resultados JSON. La solicitud de búsqueda utiliza un tiempo de espera fijo de 30 segundos.

## Salida

En una terminal enriquecida (TTY), los resultados se presentan como un encabezado seguido de una lista con viñetas: título de la página, URL enlazada de la documentación y un breve fragmento en la línea siguiente. Cuando no hay resultados, se muestra "No hay resultados.".

En una salida no enriquecida (redirigida mediante una canalización, `--no-color`, scripts), los mismos datos se presentan como Markdown:

```markdown
# Búsqueda en la documentación: <query>

- [Título](https://docs.openclaw.ai/...) - fragmento
- [Título](https://docs.openclaw.ai/...) - fragmento
```

## Códigos de salida

| Código | Significado                                                                  |
| ---- | ------------------------------------------------------------------------ |
| `0`  | La búsqueda se realizó correctamente, incluidas las respuestas sin resultados.                       |
| `1`  | La llamada a la API de búsqueda de la documentación alojada falló; stderr muestra el mensaje de error. |

## Contenido relacionado

- [Referencia de la CLI](/es/cli)
- [Documentación en vivo](https://docs.openclaw.ai)
