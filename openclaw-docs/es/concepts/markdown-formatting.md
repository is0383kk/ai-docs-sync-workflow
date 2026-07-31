---
read_when:
    - Está cambiando el formato o la segmentación de Markdown para los canales de salida
    - Está añadiendo un nuevo formateador de canal o una asignación de estilo
    - Está depurando regresiones de formato en todos los canales
summary: Pipeline de formato Markdown para canales de salida
title: Formato de Markdown
x-i18n:
    generated_at: "2026-07-26T04:38:09Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f9a35fd9a6386068e1e3bec73ec6e692f49239b468f42dd737f919b1c6a88e41
    source_path: concepts/markdown-formatting.md
    workflow: 16
---

OpenClaw convierte el Markdown saliente en una representación intermedia compartida
(IR) antes de renderizar la salida específica de cada canal. La IR conserva texto sin formato más
intervalos de estilo/enlace, de modo que un único paso de análisis sirve para todos los canales y la fragmentación nunca
divide el formato en medio de un intervalo.

## Pipeline

1. **Analizar Markdown para convertirlo en IR** (`markdownToIR`) - texto sin formato más intervalos de estilo
   (negrita, cursiva, tachado, código, bloque de código, contenido oculto, cita en bloque,
   encabezado 1-6) e intervalos de enlace. Los desplazamientos son unidades de código UTF-16 para que los intervalos
   de estilo de Signal se ajusten directamente a su API. Las tablas se analizan solo cuando el canal
   habilita un modo de tabla.
2. **Fragmentar la IR** (`chunkMarkdownIR` / `renderMarkdownIRChunksWithinLimit`)
   - la división se realiza sobre el texto de la IR antes del renderizado, por lo que los estilos en línea y los
     enlaces se segmentan por fragmento en lugar de romperse al cruzar un límite.
3. **Renderizar por canal** (`renderMarkdownWithMarkers`) - un mapa de marcadores de estilo
   convierte los intervalos en el marcado nativo del canal.

| Canal                                                            | Renderizador                                                                         | Notas                                                                                         |
| ---------------------------------------------------------------- | ------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------- |
| Slack                                                            | tokens mrkdwn (`*bold*`, `_italic_`, `` `code` ``, delimitadores de código)          | Los enlaces se convierten en `<url\|label>`; el enlace automático se deshabilita durante el análisis para evitar enlaces duplicados |
| Telegram                                                         | etiquetas HTML (`<b>`, `<i>`, `<s>`, `<code>`, `<pre><code>`, `<a href>`, `<tg-spoiler>`) | También admite tablas y encabezados de mensajes enriquecidos (`<h1>`-`<h6>`) cuando `richMessages` está activado |
| Signal                                                           | texto sin formato + intervalos `text-style`                                          | Los enlaces se renderizan como `label (url)` cuando la etiqueta difiere de la URL             |
| Discord, WhatsApp, iMessage, Microsoft Teams y otros canales     | texto sin formato                                                                    | Sin estilos basados en IR; la conversión de tablas Markdown sigue ejecutándose mediante `convertMarkdownTables` |

## Ejemplo de IR

Markdown de entrada:

```markdown
Hola, **mundo**; consulta la [documentación](https://docs.openclaw.ai).
```

IR (esquemática):

```json
{
  "text": "Hola, mundo; consulta la documentación.",
  "styles": [{ "start": 6, "end": 11, "style": "bold" }],
  "links": [{ "start": 26, "end": 39, "href": "https://docs.openclaw.ai" }]
}
```

## Tratamiento de tablas

`markdown.tables` controla cómo convierte un canal las tablas Markdown, por
canal y, opcionalmente, por cuenta:

| Modo      | Comportamiento                                                                       |
| --------- | ------------------------------------------------------------------------------------ |
| `code`    | Renderizar como una tabla ASCII alineada dentro de un bloque de código (valor predeterminado de respaldo) |
| `bullets` | Convertir cada fila en viñetas `label: value`                                        |
| `block`   | Mantener las tablas nativas donde el transporte las admita; de lo contrario, recurrir a `code` |
| `off`     | Deshabilitar el análisis de tablas; el texto sin procesar de la tabla pasa sin cambios |

Valores predeterminados de los plugins por canal: Signal, WhatsApp y Matrix usan de forma predeterminada
`bullets`; Mattermost usa `off`; Telegram usa `block` (que
se resuelve como `code` salvo que la cuenta tenga `richMessages` habilitado). Cualquier
canal sin un valor predeterminado explícito del plugin recurre a `code`.

```yaml
channels:
  discord:
    markdown:
      tables: code
    accounts:
      work:
        markdown:
          tables: off
```

## Reglas de fragmentación

- Los límites de fragmentos proceden de los adaptadores o la configuración del canal y se aplican al texto de la IR, no
  a la salida renderizada.
- Los bloques de código delimitados se conservan como un único bloque con un salto de línea final para que
  los canales rendericen correctamente el delimitador de cierre.
- Los prefijos de listas y citas en bloque forman parte del texto de la IR, por lo que la fragmentación nunca
  los divide por la mitad.
- Los estilos en línea nunca se dividen entre fragmentos; el renderizador vuelve a abrir un estilo
  abierto al principio del siguiente fragmento.

Consulta [Streaming y fragmentación](/concepts/streaming) para conocer los límites de los fragmentos y
el comportamiento de entrega entre canales.

## Política de enlaces

- **Slack:** `[label](url)` -> `<url|label>`; las URL simples permanecen sin cambios.
- **Telegram:** `[label](url)` -> `<a href="url">label</a>` (modo de análisis HTML).
- **Signal:** `[label](url)` -> `label (url)`, salvo que la etiqueta ya
  coincida con la URL.

## Contenido oculto

Los marcadores de contenido oculto (`||spoiler||`) se analizan para Signal (se asignan a intervalos de estilo
`SPOILER`) y Telegram (se asignan a `<tg-spoiler>`). Otros canales tratan
`||...||` como texto sin formato.

## Añadir o actualizar un formateador de canal

1. **Analizar una sola vez** con `markdownToIR(...)`, pasando las opciones adecuadas para el canal
   (`autolink`, `headingStyle`, `blockquotePrefix`, `tableMode`).
2. **Renderizar** con `renderMarkdownWithMarkers(...)` y un mapa de marcadores de estilo (o
   lógica personalizada de intervalos de estilo para transportes como Signal).
3. **Fragmentar** con `chunkMarkdownIR(...)` o
   `renderMarkdownIRChunksWithinLimit(...)` antes de renderizar cada fragmento.
4. **Conectar el adaptador** para que invoque el nuevo fragmentador y renderizador desde la
   ruta de envío saliente.
5. **Probar** con pruebas de formato y una prueba de entrega saliente si el canal
   utiliza fragmentos.

## Errores habituales

- Los tokens entre paréntesis angulares de Slack (`<@U123>`, `<#C123>`, `<https://...>`) deben
  sobrevivir al escapado; el HTML sin procesar debe seguir escapándose de forma segura.
- El HTML de Telegram exige escapar el texto situado fuera de las etiquetas para evitar que se rompa el marcado.
- Los intervalos de estilo de Signal usan desplazamientos UTF-16, no desplazamientos de puntos de código.
- Conserva los saltos de línea finales en los bloques de código delimitados para que el marcador de cierre
  quede en su propia línea.

## Contenido relacionado

<CardGroup cols={2}>
  <Card title="Streaming y fragmentación" href="/es/concepts/streaming" icon="bars-staggered">
    Comportamiento del streaming saliente, límites de los fragmentos y entrega específica de cada canal.
  </Card>
  <Card title="Prompt del sistema" href="/es/concepts/system-prompt" icon="message-lines">
    Lo que ve el modelo antes de la conversación, incluidos los archivos inyectados del espacio de trabajo.
  </Card>
</CardGroup>
