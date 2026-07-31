---
read_when:
    - Usa el plugin de llamadas de voz y quiere que todos los puntos de entrada de la CLI
    - Se necesitan tablas de opciones y valores predeterminados para setup, smoke, call, continue, speak, dtmf, end, status, tail, latency, expose y start
summary: Referencia de la CLI para `openclaw voicecall` (interfaz de comandos del plugin de llamadas de voz)
title: Llamada de voz
x-i18n:
    generated_at: "2026-07-26T05:07:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: aec445886cccb79c9212dd9f1f448ff9634274deb380632be786478c9bb29670
    source_path: cli/voicecall.md
    workflow: 16
---

# `openclaw voicecall`

`voicecall` es un comando proporcionado por un plugin. Solo aparece cuando el plugin
de llamadas de voz está instalado y habilitado.

Cuando el Gateway está en ejecución, los comandos operativos (`call`, `start`,
`continue`, `speak`, `dtmf`, `end`, `status`) se dirigen al entorno de ejecución
de llamadas de voz de ese Gateway. Si no se puede acceder a ningún Gateway, recurren a un entorno de ejecución
de CLI independiente.

## Subcomandos

```bash
openclaw voicecall setup    [--json]
openclaw voicecall smoke    [-t <phone>] [--message <text>] [--mode <m>] [--yes] [--json]
openclaw voicecall call     -m <text> [-t <phone>] [--mode <m>]
openclaw voicecall start    --to <phone> [--message <text>] [--mode <m>]
openclaw voicecall continue --call-id <id> --message <text>
openclaw voicecall speak    --call-id <id> --message <text>
openclaw voicecall dtmf     --call-id <id> --digits <digits>
openclaw voicecall end      --call-id <id>
openclaw voicecall status   [--call-id <id>] [--json]
openclaw voicecall tail     [--file <path>] [--since <n>] [--poll <ms>]
openclaw voicecall latency  [--file <path>] [--last <n>]
openclaw voicecall expose   [--mode <m>] [--path <p>] [--port <port>] [--serve-path <p>]
```

| Subcomando | Descripción                                                     |
| ---------- | --------------------------------------------------------------- |
| `setup`    | Muestra las comprobaciones de disponibilidad del proveedor y el webhook.                     |
| `smoke`    | Ejecuta comprobaciones de disponibilidad; realiza una llamada de prueba real solo con `--yes`. |
| `call`     | Inicia una llamada de voz saliente.                                |
| `start`    | Alias de `call` con `--to` obligatorio y `--message` opcional. |
| `continue` | Reproduce un mensaje y espera la siguiente respuesta.                 |
| `speak`    | Reproduce un mensaje sin esperar una respuesta.                 |
| `dtmf`     | Envía dígitos DTMF a una llamada activa.                             |
| `end`      | Cuelga una llamada activa.                                         |
| `status`   | Inspecciona las llamadas activas (o una mediante `--call-id`).                   |
| `tail`     | Sigue `calls.jsonl` (útil durante las pruebas del proveedor).              |
| `latency`  | Resume las métricas de latencia de turnos de `calls.jsonl`.              |
| `expose`   | Alterna Tailscale Serve/Funnel para el endpoint del webhook.         |

## Configuración y prueba rápida

### `setup`

De forma predeterminada, muestra comprobaciones de disponibilidad legibles. Pase `--json` para scripts.

```bash
openclaw voicecall setup
openclaw voicecall setup --json
```

### `smoke`

Ejecuta las mismas comprobaciones de disponibilidad. Solo realiza una llamada telefónica real cuando están presentes
tanto `--to` como `--yes`.

| Indicador          | Valor predeterminado               | Descripción                             |
| ------------------ | --------------------------------- | --------------------------------------- |
| `-t, --to <phone>` | (ninguno)                         | Número de teléfono al que llamar para una prueba rápida real.  |
| `--message <text>` | `OpenClaw voice call smoke test.` | Mensaje que se reproducirá durante la llamada de prueba rápida. |
| `--mode <mode>`    | `notify`                          | Modo de llamada: `notify` o `conversation`.  |
| `--yes`            | `false`                           | Realiza la llamada saliente real.  |
| `--json`           | `false`                           | Muestra JSON legible por máquina.            |

```bash
openclaw voicecall smoke
openclaw voicecall smoke --to "+15555550123"        # ejecución simulada
openclaw voicecall smoke --to "+15555550123" --yes  # llamada de notificación real
```

<Note>
Para los proveedores externos (`plivo`, `telnyx`, `twilio`), `setup` y `smoke` requieren una URL pública de webhook procedente de `publicUrl`, un túnel o la exposición mediante Tailscale. Se rechazan las alternativas de servicio con dirección de bucle invertido o privada porque los operadores no pueden acceder a ellas.
</Note>

## Ciclo de vida de las llamadas

### `call`

Inicia una llamada de voz saliente.

| Indicador              | Obligatorio | Valor predeterminado | Descripción                                                                |
| ---------------------- | ----------- | -------------------- | -------------------------------------------------------------------------- |
| `-m, --message <text>` | sí        | (ninguno)            | Mensaje que se reproducirá cuando se conecte la llamada.                                   |
| `-t, --to <phone>`     | no       | configuración `toNumber` | Número de teléfono E.164 al que llamar.                                                |
| `--mode <mode>`        | no       | `conversation`    | Modo de llamada: `notify` (colgar después del mensaje) o `conversation` (mantener abierta). |

```bash
openclaw voicecall call --to "+15555550123" --message "Hello"
openclaw voicecall call -m "Heads up" --mode notify
```

### `start`

Alias de `call` con una estructura de indicadores predeterminada diferente.

| Indicador          | Obligatorio | Valor predeterminado | Descripción                              |
| ------------------ | ----------- | -------------------- | ---------------------------------------- |
| `--to <phone>`     | sí        | (ninguno)            | Número de teléfono al que llamar.                    |
| `--message <text>` | no       | (ninguno)            | Mensaje que se reproducirá cuando se conecte la llamada. |
| `--mode <mode>`    | no       | `conversation` | Modo de llamada: `notify` o `conversation`.   |

### `continue`

Reproduce un mensaje y espera una respuesta.

| Indicador          | Obligatorio | Descripción          |
| ------------------ | ----------- | -------------------- |
| `--call-id <id>`   | sí        | ID de la llamada.    |
| `--message <text>` | sí        | Mensaje que se reproducirá. |

### `speak`

Reproduce un mensaje sin esperar una respuesta.

| Indicador          | Obligatorio | Descripción          |
| ------------------ | ----------- | -------------------- |
| `--call-id <id>`   | sí        | ID de la llamada.    |
| `--message <text>` | sí        | Mensaje que se reproducirá. |

### `dtmf`

Envía dígitos DTMF a una llamada activa.

| Indicador           | Obligatorio | Descripción                                      |
| ------------------- | ----------- | ------------------------------------------------ |
| `--call-id <id>`    | sí        | ID de la llamada.                                |
| `--digits <digits>` | sí        | Dígitos DTMF (por ejemplo, `ww123456#` para pausas). |

### `end`

Cuelga una llamada activa.

| Indicador        | Obligatorio | Descripción       |
| ---------------- | ----------- | ----------------- |
| `--call-id <id>` | sí        | ID de la llamada. |

### `status`

Inspecciona las llamadas activas.

| Indicador        | Valor predeterminado | Descripción                         |
| ---------------- | -------------------- | ----------------------------------- |
| `--call-id <id>` | (ninguno)            | Restringe la salida a una llamada.  |
| `--json`         | `false` | Muestra JSON legible por máquina.   |

```bash
openclaw voicecall status
openclaw voicecall status --json
openclaw voicecall status --call-id <id>
```

## Registros y métricas

### `tail`

Sigue el registro JSONL de llamadas de voz. Al iniciarse, muestra las últimas `--since` líneas y,
a continuación, transmite las líneas nuevas a medida que se escriben.

| Indicador       | Valor predeterminado                | Descripción                         |
| --------------- | ----------------------------------- | ----------------------------------- |
| `--file <path>` | resuelto desde el almacén del plugin | Ruta a `calls.jsonl`.       |
| `--since <n>`   | `25`                       | Líneas que se mostrarán antes de iniciar el seguimiento. |
| `--poll <ms>`   | `250` (mínimo 50)         | Intervalo de sondeo en milisegundos. |

### `latency`

Resume las métricas de latencia de turnos y espera de escucha de `calls.jsonl`. La salida es
JSON con resúmenes de `recordsScanned`, `turnLatency` y `listenWait`.

| Indicador       | Valor predeterminado                  | Descripción                              |
| --------------- | ------------------------------------- | ---------------------------------------- |
| `--file <path>` | resuelto desde el almacén del plugin | Ruta a `calls.jsonl`.               |
| `--last <n>`    | `200` (mínimo 1)          | Número de registros recientes que se analizarán. |

## Exposición de webhooks

### `expose`

Habilita, deshabilita o cambia la configuración de Tailscale Serve/Funnel para el
webhook de voz.

| Indicador             | Valor predeterminado                     | Descripción                                     |
| --------------------- | ---------------------------------------- | ----------------------------------------------- |
| `--mode <mode>`       | `funnel`                                  | `off`, `serve` (tailnet) o `funnel` (público). |
| `--path <path>`       | configuración `tailscale.path` o `--serve-path` | Ruta de Tailscale que se expondrá.                       |
| `--port <port>`       | configuración `serve.port` o `3334`             | Puerto local del webhook.                             |
| `--serve-path <path>` | configuración `serve.path` o `/voice/webhook`   | Ruta local del webhook.                             |

```bash
openclaw voicecall expose --mode serve
openclaw voicecall expose --mode funnel
openclaw voicecall expose --mode off
```

<Warning>
Exponga el endpoint del webhook únicamente a redes de confianza. Siempre que sea posible, prefiera Tailscale Serve a Funnel.
</Warning>

## Temas relacionados

- [Referencia de la CLI](/es/cli)
- [Plugin de llamadas de voz](/es/plugins/voice-call)
