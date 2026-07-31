---
read_when:
    - Quiere leer los resúmenes de transcripciones almacenados desde la terminal
    - Necesita la ruta a un resumen de transcripciones en Markdown
    - Está depurando la estructura de almacenamiento de las transcripciones del núcleo
summary: Referencia de la CLI para `openclaw transcripts` (enumerar, mostrar y exportar transcripciones almacenadas)
title: CLI de transcripciones
x-i18n:
    generated_at: "2026-07-26T04:36:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c04ba637fb46ec271383b2f0d17655e18018e07f489c34dc3fd14ad926f27aa4
    source_path: cli/transcripts.md
    workflow: 16
---

# `openclaw transcripts`

Comando de inspección y exportación para transcripciones de reuniones persistentes. Los participantes en el navegador de Google Meet,
Microsoft Teams y Zoom capturan notas automáticamente;
la herramienta de agente `transcripts` también admite la captura mediante proveedores y la importación manual.

El estado canónico de las transcripciones reside en la base de datos SQLite compartida en
`$OPENCLAW_STATE_DIR/state/openclaw.sqlite`. `show` y `path` materializan explícitamente
artefactos visibles para el usuario en el directorio de estado:

```text
$OPENCLAW_STATE_DIR/transcripts/YYYY-MM-DD/<session>/
  metadata.json
  transcript.jsonl
  summary.json
  summary.md
```

Estos archivos son exportaciones, no un segundo almacén de tiempo de ejecución. OpenClaw no vuelve a leerlos
durante la captura, el resumen ni el listado. El directorio de estado predeterminado es
`~/.openclaw`; se puede sustituir con `OPENCLAW_STATE_DIR`. El directorio de fecha se obtiene
de la hora de inicio de la sesión; el directorio de sesión es un slug seguro para el sistema de archivos
derivado del id. de sesión.

## Comandos

```bash
openclaw transcripts list
openclaw transcripts show <session>
openclaw transcripts show YYYY-MM-DD/<session>
openclaw transcripts path <session>
openclaw transcripts path YYYY-MM-DD/<session>
openclaw transcripts path <session> --dir
openclaw transcripts path <session> --metadata
openclaw transcripts path <session> --transcript
openclaw transcripts list --json
openclaw transcripts show <session> --json
openclaw transcripts path <session> --json
```

| Comando                       | Descripción                                          |
| ----------------------------- | ---------------------------------------------------- |
| `list`                        | Enumera las sesiones almacenadas.                                |
| `show <session>`              | Imprime y materializa `summary.md`.                  |
| `path <session>`              | Materializa e imprime la ruta de `summary.md`.         |
| `path <session> --dir`        | Materializa todos los artefactos e imprime su directorio. |
| `path <session> --metadata`   | Materializa e imprime `metadata.json`.               |
| `path <session> --transcript` | Materializa e imprime `transcript.jsonl`.            |
| `--json`                      | Imprime una salida legible por máquinas (cualquier subcomando).      |

`<session>` acepta un id. de sesión simple o un selector con fecha
(`YYYY-MM-DD/<session>`). Utilice la forma con fecha cuando el mismo id. de sesión
aparezca en más de un día, por ejemplo, `openclaw transcripts show
2026-05-22/standup`. Los id. de sesión predeterminados incluyen una marca de tiempo y un sufijo
aleatorio; asigne un id. fijo a una sesión solo cuando ese id. sea único en el día.

## Salida

`list` imprime una línea separada por tabulaciones para cada sesión: selector, hora de inicio, título,
ruta del resumen.

```text
2026-05-22/standup  2026-05-22T09:00:00.000Z  Reunión semanal de seguimiento  /Users/user/.openclaw/transcripts/2026-05-22/standup/summary.md
```

El selector es el valor más seguro para volver a pasarlo a `show` o `path`.

`list --json` devuelve objetos con `sessionId`, `selector`, `date`, `title`,
`startedAt`, `stoppedAt`, `source`, `path`, `summaryPath`, `hasSummary`.
Las URL de origen de reuniones almacenadas contienen únicamente el origen y la ruta; las cadenas de consulta,
los fragmentos y las credenciales incrustadas se eliminan antes de la persistencia.

`show --json` devuelve los metadatos de la sesión almacenada, el selector, el directorio de la sesión,
la ruta del resumen y el texto Markdown del resumen.

`path --json` devuelve la ruta seleccionada e indica si ese artefacto se pudo
materializar. Las exportaciones de metadatos y transcripciones siempre existen para una sesión
almacenada; una ruta de resumen indica `exists: false` hasta que la sesión tenga un resumen.

## Varias sesiones al día

Las sesiones se agrupan por fecha y, después, por id. de sesión. Diez reuniones en un día se convierten en
diez carpetas hermanas:

```text
~/.openclaw/transcripts/2026-05-22/
  transcript-2026-05-22T09-00-00-000Z-a1b2c3d4/
  transcript-2026-05-22T10-30-00-000Z-b2c3d4e5/
  standup/
```

Utilice los id. generados de forma predeterminada para la automatización. Utilice un id. fijo como `standup` solo
cuando no vaya a repetirse en la misma fecha.

## Resúmenes ausentes

Las sesiones en directo almacenan y materializan `summary.md` cuando la sesión termina;
las transcripciones importadas lo hacen inmediatamente después de la importación. Una sesión puede aparecer en
`list` sin un resumen mientras la captura siga activa, si un proveedor ha fallado
durante la detención o si los metadatos se almacenaron antes de que llegara alguna intervención.

Utilice `path <session> --transcript` para inspeccionar la transcripción sin procesar de solo anexado,
o ejecute la acción `summarize` de la herramienta `transcripts` para regenerar el resumen
en Markdown.

## Actualización del almacén de archivos heredado

Las versiones de OpenClaw anteriores al almacén SQLite escribían el estado canónico de tiempo de ejecución
directamente en `$OPENCLAW_STATE_DIR/transcripts/`. Ejecute:

```bash
openclaw doctor --fix
```

Doctor importa todo el árbol heredado en SQLite, verifica el número y el orden
de las filas, registra comprobantes de migración y mueve el árbol de origen verificado a un
archivo `transcripts.migrated-*` con marca de tiempo. Los comandos de tiempo de ejecución no recurren
a los archivos heredados. Conserve el archivo hasta haber verificado las sesiones importadas
y cualquier exportación de la que dependa.

## Configuración

La captura de transcripciones de reuniones está habilitada de forma predeterminada. Para desactivarla globalmente:

```json
{
  "transcripts": {
    "enabled": false
  }
}
```

- `enabled` (valor predeterminado: `true`): habilita las notas automáticas de reuniones, la herramienta
de transcripciones y las fuentes de inicio automático configuradas. Establézcalo en `false` cuando las notas de reuniones
no deban persistir en el host. Un modo `transcribe` de reunión solicitado explícitamente
conserva su cola acotada existente de subtítulos en directo, pero no
escribe filas persistentes mientras este ajuste sea falso.
  Configure las fuentes de inicio automático con `transcripts.autoStart`. Cada entrada se
  habilita mediante su presencia; omita una entrada para deshabilitar esa fuente. `discord-voice`
  es la fuente incluida con capacidad de inicio automático y requiere `guildId` y
  `channelId`:

```json
{
  "transcripts": {
    "enabled": true,
    "autoStart": [
      {
        "providerId": "discord-voice",
        "guildId": "1234567890",
        "channelId": "2345678901"
      }
    ]
  }
}
```

Los id. de proveedores de reuniones son `google-meet`, `teams` y `zoom`. Sus alias
son `googlemeet`/`meet`, `teams-meetings`/`microsoft-teams`/`msteams` y
`zoom-meetings`, respectivamente. Los proveedores de reuniones se conectan a una sesión de bot
de reuniones que ya esté activa; las incorporaciones normales a reuniones no necesitan una entrada `autoStart`.
