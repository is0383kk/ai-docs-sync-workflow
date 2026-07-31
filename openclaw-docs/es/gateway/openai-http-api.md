---
read_when:
    - Integración de herramientas que requieren OpenAI Chat Completions
summary: Expón un endpoint HTTP `/v1/chat/completions` compatible con OpenAI desde el Gateway
title: Completado de chat de OpenAI
x-i18n:
    generated_at: "2026-07-26T05:13:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4cc5a1a56972bb9070da0f8f60d6efd673cc1d1d516b730c55bc9d171fc7a5b3
    source_path: gateway/openai-http-api.md
    workflow: 16
---

El Gateway puede ofrecer una pequeña interfaz de Chat Completions compatible con OpenAI. Está **deshabilitada de forma predeterminada**.

Una vez habilitada, ofrece todo lo siguiente en el mismo puerto que el Gateway (multiplexación de WS + HTTP):

| Método | Ruta                   |
| ------ | ---------------------- |
| POST   | `/v1/chat/completions` |
| GET    | `/v1/models`           |
| GET    | `/v1/models/{id}`      |
| POST   | `/v1/embeddings`       |
| POST   | `/v1/responses`        |

Las solicitudes se ejecutan como una ejecución normal de un agente del Gateway (la misma ruta de código que `openclaw agent`), por lo que el enrutamiento, los permisos y la configuración coinciden con los del Gateway.

## Habilitación del endpoint

```json5
{
  gateway: {
    http: {
      endpoints: {
        chatCompletions: { enabled: true },
      },
    },
  },
}
```

Establezca `enabled: false` (u omítalo) para deshabilitarlo.

## Límite de seguridad (importante)

Trate este endpoint como **acceso completo de operador** a la instancia del Gateway:

- Un token o una contraseña válidos del Gateway para este endpoint equivalen a una credencial de propietario u operador, no a un ámbito limitado por usuario.
- Las solicitudes pasan por la misma ruta de agente del plano de control que las acciones de operadores de confianza, por lo que, si la política del agente de destino permite herramientas sensibles, este endpoint puede utilizarlas.
- Manténgalo únicamente en la interfaz de bucle invertido, la tailnet o una entrada privada. No lo exponga a la Internet pública.

Matriz de autenticación:

| Ruta de autenticación                                                                                            | Comportamiento                                                                                                                                                                                                                                                                                                  |
| ---------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `gateway.auth.mode="token"` o `"password"` + `Authorization: Bearer ...`                            | Demuestra la posesión del secreto compartido del Gateway. Ignora cualquier encabezado `x-openclaw-scopes` y restablece el conjunto completo predeterminado de ámbitos de operador: `operator.admin`, `operator.approvals`, `operator.pairing`, `operator.read`, `operator.talk.secrets`, `operator.write`. Trata los turnos del chat como turnos enviados por el propietario. |
| HTTP de confianza con identidad (autenticación mediante proxy de confianza o `gateway.auth.mode="none"` en una entrada privada) | Respeta `x-openclaw-scopes` cuando está presente; si está ausente, utiliza el conjunto predeterminado de ámbitos de operador. Solo pierde la semántica de propietario cuando el autor de la llamada restringe explícitamente los ámbitos y omite `operator.admin`. Requiere `operator.admin` para controles de nivel de propietario como `x-openclaw-model`.                        |

Consulte [Ámbitos de operador](/es/gateway/operator-scopes), [Seguridad](/es/gateway/security) y [Acceso remoto](/es/gateway/remote).

## Autenticación

Utiliza la configuración de autenticación del Gateway (consulte [Autenticación mediante proxy de confianza](/es/gateway/trusted-proxy-auth) para obtener información detallada sobre ese modo):

| Modo                                | Cómo autenticarse                                                                                                                                                                     |
| ----------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `gateway.auth.mode="token"`         | `Authorization: Bearer <token>`. Se establece mediante `gateway.auth.token` o `OPENCLAW_GATEWAY_TOKEN`.                                                                                              |
| `gateway.auth.mode="password"`      | `Authorization: Bearer <password>`. Se establece mediante `gateway.auth.password` o `OPENCLAW_GATEWAY_PASSWORD`.                                                                                     |
| `gateway.auth.mode="trusted-proxy"` | Enrute la solicitud mediante el proxy configurado que reconoce identidades; este inyecta los encabezados de identidad requeridos. Los proxies de bucle invertido del mismo host necesitan `gateway.auth.trustedProxy.allowLoopback = true` explícito. |
| `gateway.auth.mode="none"`          | No se requiere ningún encabezado de autenticación (solo para entradas privadas).                                                                                                                                         |

Notas:

- Los autores de llamadas del mismo host que omitan el proxy en un Gateway `trusted-proxy` pueden recurrir directamente a `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD`. Cualquier evidencia de encabezado `Forwarded`, `X-Forwarded-*` o `X-Real-IP` mantiene la solicitud en la ruta del proxy de confianza.
- Si `gateway.auth.rateLimit` está configurado y fallan demasiados intentos de autenticación, el endpoint devuelve `429` con un encabezado `Retry-After`.

## Cuándo utilizar este endpoint

- Prefiera este endpoint en lugar de añadir un nuevo canal integrado cuando la integración sea simplemente otra interfaz de operador o cliente para el mismo Gateway.
- Para clientes móviles nativos que se conecten directamente a un Gateway remoto, prefiera [WebChat](/es/web/webchat) o el [Protocolo del Gateway](/es/gateway/protocol) con el flujo de arranque de dispositivos emparejados y tokens de dispositivo, para que el dispositivo no necesite un token o una contraseña HTTP compartidos.
- En su lugar, cree un plugin de canal cuando integre una red de mensajería externa con sus propios usuarios, salas, entrega mediante Webhook o transporte de salida. Consulte [Creación de plugins](/es/plugins/building-plugins).

## Contrato de modelo centrado en agentes

OpenClaw trata el campo `model` de OpenAI como un **destino de agente**, no como un identificador de modelo de proveedor sin procesar.

| Valor de `model`                                | Se enruta a                                                                                                                |
| -------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `openclaw`                                   | Agente predeterminado configurado                                                                                                 |
| `openclaw/default`                           | Agente predeterminado configurado (alias estable; se puede codificar de forma fija sin riesgo aunque el identificador real del agente predeterminado cambie entre entornos) |
| `openclaw/<agentId>` o `openclaw:<agentId>` | Agente específico                                                                                                           |
| `agent:<agentId>`                            | Agente específico (alias de compatibilidad)                                                                                     |

Encabezados de solicitud opcionales:

| Encabezado                                          | Efecto                                                                                                                                                                                                                                                                      |
| ----------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `x-openclaw-model: <provider/model-or-bare-id>` | Sustituye el modelo de backend del agente seleccionado. Los autores de llamadas que usan un secreto compartido como portador pueden utilizarlo directamente; los autores de llamadas con identidad (proxy de confianza o entrada privada sin autenticación con `x-openclaw-scopes`) necesitan `operator.admin`; de lo contrario, `403 missing scope: operator.admin`. |
| `x-openclaw-agent-id: <agentId>`                | Sustitución de compatibilidad para la selección del agente.                                                                                                                                                                                                                                 |
| `x-openclaw-session-key: <sessionKey>`          | Enrutamiento explícito de sesiones. Se rechaza con `400 invalid_request_error` si utiliza un espacio de nombres interno reservado (`subagent:`, `cron:`, `acp:`).                                                                                                                                |
| `x-openclaw-message-channel: <channel>`         | Establece el contexto sintético del canal de entrada para instrucciones y políticas que reconocen el canal.                                                                                                                                                                                              |

`/v1/models` enumera destinos de agente de nivel superior (`openclaw`, `openclaw/default`, `openclaw/<agentId>`), no modelos de proveedores de backend ni subagentes; los subagentes permanecen como topología interna de ejecución. Si se omite `x-openclaw-model`, el agente seleccionado se ejecuta con su modelo configurado habitual.

`/v1/embeddings` utiliza los mismos identificadores `model` de destino de agente. Envíe `x-openclaw-model` (desde un autor de llamada con secreto compartido o uno con identidad y `operator.admin`) para seleccionar un modelo de embeddings específico; de lo contrario, la solicitud utiliza la configuración habitual de embeddings del agente seleccionado.

## Comportamiento de las sesiones

De forma predeterminada, el endpoint **no conserva estado entre solicitudes** (se genera una nueva clave de sesión en cada llamada).

Si la solicitud incluye una cadena `user` de OpenAI, el Gateway deriva de ella una clave de sesión estable para que las llamadas repetidas puedan compartir una sesión de agente. En aplicaciones personalizadas, reutilice el mismo valor de `user` en cada hilo de conversación; evite los identificadores de nivel de cuenta, salvo que se desee que varias conversaciones o dispositivos compartan una misma sesión de OpenClaw. Utilice `x-openclaw-session-key` únicamente cuando necesite controlar explícitamente el enrutamiento entre varios clientes o hilos, con claves administradas por la aplicación que eviten los espacios de nombres reservados anteriores.

## Límites de las solicitudes

El endpoint utiliza límites integrados de 20 MB por cuerpo de solicitud, 8 partes de `image_url`
del último mensaje del usuario y 20 MB de datos de imagen decodificados
acumulados. La política de fuentes de imágenes sigue siendo configurable en
`gateway.http.endpoints.chatCompletions.images`:

```json5
{
  gateway: {
    http: {
      endpoints: {
        chatCompletions: {
          enabled: true,
          images: {
            allowUrl: false,
            urlAllowlist: ["cdn.example.com", "*.assets.example.com"],
            allowedMimes: [
              "image/jpeg",
              "image/png",
              "image/gif",
              "image/webp",
              "image/heic",
              "image/heif",
            ],
            maxBytes: 10485760,
            maxRedirects: 3,
            timeoutMs: 10000,
          },
        },
      },
    },
  },
}
```

Valores predeterminados de la configuración de imágenes:

| Clave                   | Valor predeterminado                                                             |
| --------------------- | ------------------------------------------------------------------- |
| `images.allowUrl`     | `false` (las partes de `image_url` procedentes de URL se rechazan a menos que estén habilitadas) |
| `images.maxBytes`     | 10MB por imagen                                                      |
| `images.maxRedirects` | 3                                                                   |
| `images.timeoutMs`    | 10s                                                                 |

Las fuentes `image_url` HEIC/HEIF se aceptan y normalizan a JPEG antes de entregarlas al proveedor mediante el procesador de imágenes compartido de OpenClaw (Rastermill), que recurre a un conversor del sistema (`sips`, ImageMagick, GraphicsMagick o ffmpeg) para los formatos que requieren compatibilidad con códecs externos.

Nota de seguridad: incluir un nombre de host en la lista de permitidos no evita el bloqueo de direcciones IP privadas o internas. Para Gateways expuestos a Internet, aplique controles de salida de red además de las protecciones de la aplicación. Consulte [Seguridad](/es/gateway/security).

## Contrato de herramientas de chat

`/v1/chat/completions` admite un subconjunto de herramientas de función compatible con clientes comunes de Chat de OpenAI.

### Campos de solicitud compatibles

| Campo                      | Notas                                                                                                                                         |
| -------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `tools`                    | Matriz de `{ "type": "function", "function": { ... } }`                                                                                        |
| `tool_choice`              | `"auto"`, `"none"`, `"required"` o `{ "type": "function", "function": { "name": "..." } }`                                                  |
| `messages[*].role: "tool"` | Turnos de seguimiento                                                                                                                               |
| `messages[*].tool_call_id` | Vincula el resultado de una herramienta con una llamada anterior a una herramienta                                                                                                 |
| `max_completion_tokens`    | Número; límite por llamada del total de tokens de finalización (incluidos los tokens de razonamiento). Nombre de campo actual; se usa cuando se envían tanto este como `max_tokens`. |
| `max_tokens`               | Número; alias heredado, se ignora cuando `max_completion_tokens` también está presente.                                                                   |
| `temperature`              | Número de 0 a 2; se reenvía al proveedor ascendente en la medida de lo posible. `400 invalid_request_error` si está fuera del intervalo.                                     |
| `top_p`                    | Número de 0 a 1; se aplica en la medida de lo posible. `400 invalid_request_error` si está fuera del intervalo.                                                                         |
| `frequency_penalty`        | Número de -2.0 a 2.0; se aplica en la medida de lo posible. `400 invalid_request_error` si está fuera del intervalo.                                                                 |
| `presence_penalty`         | Número de -2.0 a 2.0; se aplica en la medida de lo posible. `400 invalid_request_error` si está fuera del intervalo.                                                                 |
| `seed`                     | Entero; se aplica en la medida de lo posible. `400 invalid_request_error` para valores no enteros.                                                                     |
| `stop`                     | Cadena o matriz de hasta 4 cadenas; se aplica en la medida de lo posible. `400 invalid_request_error` para más de 4 secuencias o entradas que no sean cadenas o estén vacías.           |

Todos los campos de muestreo y límite de tokens utilizan el mismo canal de parámetros de flujo del agente y se reenvían en la medida de lo posible:

- Límite de tokens: el transporte del proveedor elige el nombre del campo en la transmisión: `max_completion_tokens` para los endpoints de la familia OpenAI y `max_tokens` para los proveedores que solo aceptan el nombre heredado (Mistral, Chutes).
- `stop` se asigna al campo de detención del transporte: `stop` para backends de Chat Completions y `stop_sequences` para Anthropic. La API Responses de OpenAI no tiene ningún parámetro de detención, por lo que `stop` no se aplica a los modelos respaldados por Responses.
- El backend Codex Responses basado en ChatGPT utiliza un muestreo fijo en el servidor y elimina `temperature`/`top_p` (junto con `max_output_tokens`, `metadata`, `prompt_cache_retention` y `service_tier`) antes de que la solicitud llegue a dicho backend.

### Variantes no compatibles

Devuelve `400 invalid_request_error` para:

- `tools` que no sean matrices, entradas de herramientas que no sean funciones o ausencia de `tool.function.name`
- variantes de `tool_choice`, como `allowed_tools` y `custom`
- valores de `tool_choice.function.name` que no coincidan con una herramienta proporcionada

Para `tool_choice: "required"` y `tool_choice` fijado a una función, el endpoint restringe el conjunto de herramientas de función del cliente expuesto, indica al runtime que llame a una herramienta del cliente antes de responder y genera un error si la respuesta del agente no contiene ninguna llamada estructurada coincidente a una herramienta del cliente. Esto se aplica a la lista HTTP `tools` proporcionada por quien realiza la llamada, no a todas las herramientas internas del agente de OpenClaw.

### Estructura de la respuesta de herramienta sin streaming

Cuando el agente llama a herramientas, la respuesta utiliza:

- `choices[0].finish_reason = "tool_calls"`
- entradas de `choices[0].message.tool_calls[]` con `id`, `type: "function"`, `function.name` y `function.arguments` (cadena JSON)
- Comentario del asistente anterior a la llamada a la herramienta, en `choices[0].message.content` (posiblemente vacío)

### Estructura de la respuesta de herramienta con streaming

Cuando `stream: true`, las llamadas a herramientas llegan como fragmentos SSE incrementales: un delta inicial del rol de asistente, deltas opcionales de comentarios del asistente, uno o más fragmentos de `delta.tool_calls` que contienen la identidad de la herramienta y fragmentos de argumentos y, después, un fragmento final con `finish_reason: "tool_calls"` y `data: [DONE]`.

Si `stream_options.include_usage=true`, se emite un fragmento final de uso antes de `[DONE]`.

### Bucle de seguimiento de herramientas

Después de recibir `tool_calls`, ejecute las funciones solicitadas y envíe una solicitud de seguimiento que incluya el mensaje anterior del asistente con la llamada a la herramienta y uno o más mensajes `role: "tool"` con un `tool_call_id` coincidente. Esto continúa el mismo bucle de razonamiento del agente para generar la respuesta final.

## Streaming (SSE)

Establezca `stream: true` para recibir eventos enviados por el servidor:

- `Content-Type: text/event-stream`
- Cada línea de evento es `data: <json>`
- El flujo termina con `data: [DONE]`

## Configuración rápida de Open WebUI

- URL base: `http://127.0.0.1:18789/v1`
- URL base de Docker en macOS: `http://host.docker.internal:18789/v1`
- Clave de API: el token de portador del Gateway
- Modelo: `openclaw/default`

Comportamiento esperado: `GET /v1/models` enumera `openclaw/default` y Open WebUI lo utiliza como id. del modelo de chat. Para un proveedor o modelo de backend específico, establezca el modelo predeterminado habitual del agente o envíe `x-openclaw-model` (llamante con secreto compartido o llamante con identidad y `operator.admin`).

Prueba rápida de humo:

```bash
curl -sS http://127.0.0.1:18789/v1/models \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

Si devuelve `openclaw/default`, la mayoría de las configuraciones de Open WebUI pueden conectarse con la misma URL base y el mismo token.

## Ejemplos

Sesión estable para una conversación de una aplicación:

```bash
curl -sS http://127.0.0.1:18789/v1/chat/completions \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "openclaw/default",
    "user": "conv:YOUR_CONVERSATION_ID",
    "messages": [{"role":"user","content":"Resume mis tareas para hoy"}]
  }'
```

Reutilice el mismo valor de `user` en llamadas posteriores de esa conversación para continuar la misma sesión del agente.

Sin streaming:

```bash
curl -sS http://127.0.0.1:18789/v1/chat/completions \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "openclaw/default",
    "messages": [{"role":"user","content":"hola"}]
  }'
```

Con streaming:

```bash
curl -N http://127.0.0.1:18789/v1/chat/completions \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -H 'x-openclaw-model: openai/gpt-5.4' \
  -d '{
    "model": "openclaw/research",
    "stream": true,
    "messages": [{"role":"user","content":"hola"}]
  }'
```

Enumerar modelos:

```bash
curl -sS http://127.0.0.1:18789/v1/models \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

Obtener un modelo:

```bash
curl -sS http://127.0.0.1:18789/v1/models/openclaw%2Fdefault \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

Crear embeddings:

```bash
curl -sS http://127.0.0.1:18789/v1/embeddings \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -H 'x-openclaw-model: openai/text-embedding-3-small' \
  -d '{
    "model": "openclaw/default",
    "input": ["alpha", "beta"]
  }'
```

`/v1/embeddings` admite `input` como cadena o matriz de cadenas.

## Contenido relacionado

- [Referencia de configuración](/es/gateway/configuration-reference)
- [Ámbitos del operador](/es/gateway/operator-scopes)
- [OpenAI](/es/providers/openai)
