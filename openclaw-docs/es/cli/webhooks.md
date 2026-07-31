---
read_when:
    - Quieres conectar los eventos de Pub/Sub de Gmail con OpenClaw
    - Necesita la lista completa de opciones y los valores predeterminados
summary: Referencia de la CLI para `openclaw webhooks` (configuración y ejecutor de Pub/Sub de Gmail)
title: Webhooks
x-i18n:
    generated_at: "2026-07-26T05:09:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 83fff0ac2ce247402f45523eda0b5cdd551bd65212636118698e45cb8740236c
    source_path: cli/webhooks.md
    workflow: 16
---

# `openclaw webhooks`

Ayudantes e integraciones de Webhook. Actualmente, esta superficie se limita a los flujos de Gmail Pub/Sub basados en el observador `gog` incluido.

## Subcomandos

```bash
openclaw webhooks gmail setup --account <email> [...]
openclaw webhooks gmail run   [--account <email>] [...]
```

| Subcomando    | Descripción                                                                           |
| ------------- | ------------------------------------------------------------------------------------- |
| `gmail setup` | Asistente de ejecución única: observación de Gmail, tema/suscripción de Pub/Sub y entrega al hook de OpenClaw. |
| `gmail run`   | Ejecuta `gog watch serve` junto con el bucle de renovación automática de la observación en primer plano.               |

<Note>
El Gateway también inicia automáticamente `gog gmail watch serve` durante el arranque una vez configurados `hooks.enabled=true` y `hooks.gmail.account` (los configura `gmail setup`). `gmail run` aplica la misma lógica en primer plano, lo que resulta útil para depurar o cuando el observador del Gateway está deshabilitado. Consulte [Integración de Gmail Pub/Sub](/es/automation/cron-jobs#gmail-pubsub-integration) para conocer los detalles del inicio automático y la opción de exclusión `OPENCLAW_SKIP_GMAIL_WATCHER`.
</Note>

## `webhooks gmail setup`

```bash
openclaw webhooks gmail setup --account you@example.com
openclaw webhooks gmail setup --account you@example.com --project my-gcp-project --json
openclaw webhooks gmail setup --account you@example.com --hook-url https://gateway.example.com/hooks/gmail
```

Instala `gcloud` y `gog` si no están presentes, autentica `gcloud`, crea el tema y la suscripción de Pub/Sub, inicia la observación de Gmail y escribe la configuración de `hooks.gmail` con `hooks.enabled=true`. Muestra `Next: openclaw webhooks gmail run`.

### Obligatorio

| Indicador                | Descripción             |
| ------------------- | ----------------------- |
| `--account <email>` | Cuenta de Gmail que se observará. |

### Opciones de Pub/Sub

| Indicador                    | Valor predeterminado                | Descripción                                                                                                                             |
| ----------------------- | ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `--project <id>`        | (ninguno)                 | ID del proyecto de GCP (el propietario del cliente OAuth). Si no está disponible, se usa el ID del proyecto del propio tema y, después, el proyecto resuelto a partir de las credenciales de `gog`. |
| `--topic <name>`        | `gog-gmail-watch`      | Nombre del tema de Pub/Sub.                                                                                                                     |
| `--subscription <name>` | `gog-gmail-watch-push` | Nombre de la suscripción de Pub/Sub.                                                                                                              |
| `--label <label>`       | `INBOX`                | Etiqueta de Gmail que se observará.                                                                                                                   |
| `--push-endpoint <url>` | (ninguno)                 | Endpoint push explícito de Pub/Sub. Anula Tailscale.                                                                                    |

### Opciones de entrega de OpenClaw

| Indicador                   | Valor predeterminado                                      | Descripción                                |
| ---------------------- | -------------------------------------------- | ------------------------------------------ |
| `--hook-url <url>`     | Creada a partir de `hooks.path` y el puerto del Gateway | URL del webhook de OpenClaw.                      |
| `--hook-token <token>` | `hooks.token` o un token generado          | Token del webhook de OpenClaw.                    |
| `--push-token <token>` | Token generado                              | Token push reenviado a `gog watch serve`. |

### Opciones de `gog watch serve`

| Indicador                  | Valor predeterminado         | Descripción                                                                                                                                  |
| --------------------- | --------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `--bind <host>`       | `127.0.0.1`     | Host de enlace de `gog watch serve`.                                                                                                                 |
| `--port <port>`       | `8788`          | Puerto de `gog watch serve`.                                                                                                                      |
| `--path <path>`       | `/gmail-pubsub` | Ruta de `gog watch serve`. Se fuerza a `/` cuando Tailscale está habilitado sin un destino explícito, ya que Tailscale elimina la ruta antes de pasarla por el proxy. |
| `--include-body`      | `true`          | Incluye fragmentos del cuerpo de los correos electrónicos. No existe ningún indicador de la CLI para desactivar esta opción; configure `hooks.gmail.includeBody: false` en la configuración.                  |
| `--max-bytes <n>`     | `20000`         | Máximo de bytes por fragmento del cuerpo.                                                                                                                  |
| `--renew-minutes <n>` | `720` (12h)     | Renueva la observación de Gmail cada N minutos.                                                                                                           |

### Exposición mediante Tailscale

| Indicador                      | Valor predeterminado  | Descripción                                                      |
| ------------------------- | -------- | ---------------------------------------------------------------- |
| `--tailscale <mode>`      | `funnel` | Expone el endpoint push mediante tailscale: `funnel`, `serve` o `off`. |
| `--tailscale-path <path>` | (ninguno)   | Ruta para tailscale serve/funnel.                                 |
| `--tailscale-target <t>`  | (ninguno)   | Destino de Tailscale serve/funnel (puerto, `host:port` o URL).       |

### Salida

| Indicador     | Descripción                                       |
| -------- | ------------------------------------------------- |
| `--json` | Muestra un resumen legible por máquina en lugar de texto. |

## `webhooks gmail run`

```bash
openclaw webhooks gmail run --account you@example.com
```

Ejecuta `gog watch serve` junto con el bucle de renovación automática de la observación en primer plano y reinicia `gog watch serve` tras un retraso de 2s si termina inesperadamente.

`run` acepta los mismos indicadores de Pub/Sub, entrega de OpenClaw, `gog watch serve` y Tailscale que `setup`, excepto por lo siguiente:

- `--account` es **opcional** en `run`; si no se especifica, se usa `hooks.gmail.account`.
- `run` **no** acepta `--project`, `--push-endpoint` ni `--json`.
- Cada indicador recurre al valor de configuración correspondiente de `hooks.gmail.*` (escrito por `setup`) y, después, al mismo valor predeterminado integrado que utiliza `setup`, con una excepción: `--tailscale` tiene como valor predeterminado `off` en `run` (no `funnel`) cuando no se han configurado ni el indicador ni `hooks.gmail.tailscale.mode`.

| Categoría          | Indicadores                                                                            |
| ----------------- | -------------------------------------------------------------------------------- |
| Pub/Sub           | `--account`, `--topic`, `--subscription`, `--label`                              |
| Entrega de OpenClaw | `--hook-url`, `--hook-token`, `--push-token`                                     |
| `gog watch serve` | `--bind`, `--port`, `--path`, `--include-body`, `--max-bytes`, `--renew-minutes` |
| Tailscale         | `--tailscale`, `--tailscale-path`, `--tailscale-target`                          |

<Note>
Para `run`, el valor de `--topic` es la ruta completa del tema de Pub/Sub (`projects/.../topics/...`), no solo el nombre corto del tema.
</Note>

## Contenido relacionado

- [Referencia de la CLI](/es/cli)
- [Automatización de Webhook](/es/automation/cron-jobs)
- [Integración de Gmail Pub/Sub](/es/automation/cron-jobs#gmail-pubsub-integration)
