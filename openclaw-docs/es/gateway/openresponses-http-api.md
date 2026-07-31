---
read_when:
    - Integración de clientes compatibles con la API OpenResponses
    - Quieres entradas basadas en elementos, llamadas a herramientas del cliente o eventos SSE
summary: Expón un endpoint HTTP `/v1/responses` compatible con OpenResponses desde el Gateway
title: API de OpenResponses
x-i18n:
    generated_at: "2026-07-26T05:40:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5bfd6ca3bf0cecd761fde865b41a95cff3fc5681f74f31b3adae5cd2e0b0be95
    source_path: gateway/openresponses-http-api.md
    workflow: 16
---

El Gateway puede proporcionar un endpoint `POST /v1/responses` compatible con OpenResponses. Está **deshabilitado de forma predeterminada** y comparte su puerto con el Gateway (multiplexación de WS + HTTP): `http://<gateway-host>:<port>/v1/responses`.

Las solicitudes se ejecutan como una ejecución normal de agente del Gateway (la misma ruta de código que `openclaw agent`), por lo que el enrutamiento, los permisos y la configuración coinciden con los del Gateway.

Habilítelo o deshabilítelo con `gateway.http.endpoints.responses.enabled`. Cuando está habilitado, la misma superficie de compatibilidad también proporciona `GET /v1/models`, `GET /v1/models/{id}`, `POST /v1/embeddings` y `POST /v1/chat/completions`.

## Autenticación, seguridad y enrutamiento

El comportamiento operativo coincide con [OpenAI Chat Completions](/es/gateway/openai-http-api):

- La ruta de autenticación coincide con `gateway.auth.mode`: el secreto compartido (`token`/`password`) usa `Authorization: Bearer <token-or-password>`; el proxy de confianza usa encabezados de proxy con reconocimiento de identidad (los proxies de bucle invertido del mismo host necesitan `gateway.auth.trustedProxy.allowLoopback = true`, con una alternativa directa en el mismo host mediante `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD` cuando no hay ningún encabezado `Forwarded`/`X-Forwarded-*`/`X-Real-IP` presente); `none` en una entrada privada no necesita encabezado de autenticación. Consulte [Autenticación mediante proxy de confianza](/es/gateway/trusted-proxy-auth).
- Trate el endpoint como acceso completo de operador a la instancia del Gateway.
- Los modos de autenticación con secreto compartido ignoran un `x-openclaw-scopes` más restringido declarado por el portador y restauran el conjunto completo de ámbitos predeterminados del operador: `operator.admin`, `operator.approvals`, `operator.pairing`, `operator.read`, `operator.talk.secrets`, `operator.write`. Los turnos de chat en este endpoint se tratan como turnos de remitente propietario.
- Los modos HTTP de confianza que incluyen identidad (proxy de confianza o `gateway.auth.mode="none"`) respetan `x-openclaw-scopes` cuando está presente; de lo contrario, recurren al conjunto predeterminado de ámbitos del operador. La semántica de propietario solo se pierde cuando el llamador restringe explícitamente los ámbitos y omite `operator.admin`.
- Seleccione agentes con `model: "openclaw"`, `"openclaw/default"`, `"openclaw/<agentId>"` o el encabezado `x-openclaw-agent-id`.
- Use `x-openclaw-model` para sustituir el modelo de backend del agente seleccionado (requiere `operator.admin` en las rutas de autenticación que incluyen identidad).
- Use `x-openclaw-session-key` para el enrutamiento explícito de sesiones (se rechaza con `400 invalid_request_error` si usa un espacio de nombres reservado: `subagent:`, `cron:`, `acp:`).
- Use `x-openclaw-message-channel` para un contexto de canal de entrada sintético no predeterminado.

Para obtener la explicación canónica de los modelos de destino del agente, `openclaw/default`, el paso directo de embeddings y las sustituciones del modelo de backend, consulte [OpenAI Chat Completions](/es/gateway/openai-http-api#agent-first-model-contract).

Consulte [Ámbitos del operador](/es/gateway/operator-scopes) y [Seguridad](/es/gateway/security).

## Comportamiento de las sesiones

De forma predeterminada, el endpoint **no conserva estado entre solicitudes** (se genera una nueva clave de sesión en cada llamada).

Si la solicitud incluye una cadena `user` de OpenResponses, el Gateway deriva de ella una clave de sesión estable para que las llamadas repetidas puedan compartir una sesión de agente.

`previous_response_id` reutiliza la sesión de la respuesta anterior cuando la solicitud permanece dentro del mismo ámbito de agente, usuario y sesión solicitada (determinado por el sujeto de autenticación, el identificador del agente y `x-openclaw-session-key`).

## Estructura de la solicitud

| Campo                                                            | Compatibilidad                                                                                                                        |
| ---------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `input`                                                          | Cadena o matriz de objetos de elemento.                                                                                               |
| `instructions`                                                   | Se combina con el prompt del sistema.                                                                                                 |
| `tools`                                                          | Definiciones de herramientas del cliente (herramientas de función).                                                                                      |
| `tool_choice`                                                    | `"auto"`, `"none"`, `"required"` o `{ "type": "function", "name": "..." }` para filtrar o exigir herramientas del cliente.                |
| `stream`                                                         | Habilita el streaming mediante SSE.                                                                                                         |
| `max_output_tokens`                                              | Límite aproximado de salida (depende del proveedor).                                                                                 |
| `temperature`                                                    | Temperatura de muestreo aproximada. El backend de Codex Responses basado en ChatGPT la ignora, ya que usa un muestreo fijo del lado del servidor. |
| `top_p`                                                          | Muestreo de núcleo aproximado. Se aplica la misma salvedad de Codex Responses que para `temperature`.                                                    |
| `user`                                                           | Enrutamiento estable de sesiones.                                                                                                        |
| `previous_response_id`                                           | Continuidad de la sesión (consulte más arriba).                                                                                                |
| `max_tool_calls`, `reasoning`, `metadata`, `store`, `truncation` | Se aceptan, pero actualmente se ignoran.                                                                                                |

## Elementos (entrada)

### `message`

Roles: `system`, `developer`, `user`, `assistant`.

- `system` y `developer` se añaden al prompt del sistema.
- El elemento `user` o `function_call_output` más reciente se convierte en el «mensaje actual».
- Los mensajes anteriores del usuario y del asistente se incluyen como historial para proporcionar contexto.

### `function_call_output` (herramientas basadas en turnos)

Devuelva los resultados de las herramientas al modelo:

```json
{
  "type": "function_call_output",
  "call_id": "call_123",
  "output": "{\"temperature\": \"72F\"}"
}
```

### `reasoning` y `item_reference`

Se aceptan por compatibilidad con el esquema, pero se ignoran al crear el prompt.

## Herramientas (herramientas de función del lado del cliente)

Proporcione herramientas con `tools: [{ type: "function", name, description?, parameters? }]`.

Si el agente llama a una herramienta, la respuesta devuelve un elemento de salida `function_call`. Envíe una solicitud de seguimiento con `function_call_output` para continuar el turno.

Para `tool_choice: "required"` y un `tool_choice` fijado a una función, el endpoint restringe el conjunto de herramientas de función del cliente expuestas, indica al entorno de ejecución que llame a una herramienta del cliente antes de responder y rechaza el turno si no incluye una llamada estructurada coincidente a una herramienta del cliente, de acuerdo con el contrato `/v1/chat/completions`. Las solicitudes sin streaming devuelven `502` con un `api_error`; las solicitudes con streaming emiten un evento `response.failed`.

## Imágenes (`input_image`)

Admite fuentes en base64 o mediante URL:

```json
{
  "type": "input_image",
  "source": { "type": "url", "url": "https://example.com/image.png" }
}
```

Tipos MIME permitidos (valor predeterminado): `image/jpeg`, `image/png`, `image/gif`, `image/webp`, `image/heic`, `image/heif`. Tamaño máximo (valor predeterminado): 10MB.

## Archivos (`input_file`)

Admite fuentes en base64 o mediante URL:

```json
{
  "type": "input_file",
  "source": {
    "type": "base64",
    "media_type": "text/plain",
    "data": "SGVsbG8gV29ybGQh",
    "filename": "hello.txt"
  }
}
```

Tipos MIME permitidos (valor predeterminado): `text/plain`, `text/markdown`, `text/html`, `text/csv`, `application/json`, `application/pdf`. Tamaño máximo (valor predeterminado): 5MB.

Comportamiento actual:

- El contenido del archivo se decodifica y se añade al **prompt del sistema**, no al mensaje del usuario, por lo que permanece efímero (no se conserva en el historial de la sesión).
- El texto decodificado del archivo se delimita como **contenido externo no fiable** antes de añadirlo, por lo que los bytes del archivo se tratan como datos, no como instrucciones fiables. El bloque insertado usa marcadores de límite explícitos (`<<<EXTERNAL_UNTRUSTED_CONTENT id="...">>>` / `<<<END_EXTERNAL_UNTRUSTED_CONTENT id="...">>>`) y una línea de metadatos `Source: External`. Omite intencionadamente el extenso aviso `SECURITY NOTICE:` para preservar el presupuesto del prompt; los marcadores de límite y los metadatos siguen siendo aplicables.
- Primero se analiza el texto de los PDF. Si se encuentra poco texto, las primeras páginas se rasterizan como imágenes y se pasan al modelo, y el bloque de archivo insertado usa el marcador de posición `[PDF content rendered to images]`.

El análisis de PDF lo proporciona el plugin `document-extract` incluido, que usa `clawpdf` y su entorno de ejecución WebAssembly de PDFium empaquetado para extraer texto y representar páginas.

Valores predeterminados de obtención mediante URL:

- `files.allowUrl`: `true`
- `images.allowUrl`: `true`
- `maxUrlParts`: `8` (total de partes `input_file` + `input_image` basadas en URL por solicitud)
- Las solicitudes están protegidas (resolución DNS, bloqueo de direcciones IP privadas, límites de redirecciones y tiempos de espera).
- Se admiten listas opcionales de nombres de host permitidos para cada tipo de entrada (`files.urlAllowlist`, `images.urlAllowlist`): host exacto (`"cdn.example.com"`) o subdominios con comodín (`"*.assets.example.com"`, no coincide con el dominio raíz). Las listas vacías u omitidas implican que no hay restricciones por lista de nombres de host permitidos.
- Para deshabilitar por completo las obtenciones basadas en URL, establezca `files.allowUrl: false` y/o `images.allowUrl: false`.

## Límites de archivos e imágenes

El endpoint usa un límite integrado de 20 MB para el cuerpo de la solicitud. La política de fuentes de archivos e imágenes
sigue siendo configurable en `gateway.http.endpoints.responses`:

```json5
{
  gateway: {
    http: {
      endpoints: {
        responses: {
          enabled: true,
          maxUrlParts: 8,
          files: {
            allowUrl: true,
            urlAllowlist: ["cdn.example.com", "*.assets.example.com"],
            allowedMimes: [
              "text/plain",
              "text/markdown",
              "text/html",
              "text/csv",
              "application/json",
              "application/pdf",
            ],
            maxBytes: 5242880,
            maxChars: 60000,
            maxRedirects: 3,
            timeoutMs: 10000,
            pdf: {
              maxPages: 4,
              maxPixels: 4000000,
              minTextChars: 200,
            },
          },
          images: {
            allowUrl: true,
            urlAllowlist: ["images.example.com"],
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

Valores predeterminados cuando se omiten:

| Clave                      | Valor predeterminado   |
| ------------------------ | --------- |
| `maxUrlParts`            | 8         |
| `files.maxBytes`         | 5MB       |
| `files.maxChars`         | 60k       |
| `files.maxRedirects`     | 3         |
| `files.timeoutMs`        | 10s       |
| `files.pdf.maxPages`     | 4         |
| `files.pdf.maxPixels`    | 4,000,000 |
| `files.pdf.minTextChars` | 200       |
| `images.maxBytes`        | 10MB      |
| `images.maxRedirects`    | 3         |
| `images.timeoutMs`       | 10s       |

Las fuentes HEIC/HEIF `input_image` se normalizan a JPEG antes de entregarlas al proveedor mediante el procesador de imágenes compartido de OpenClaw (Rastermill), que recurre a un conversor del sistema (`sips`, ImageMagick, GraphicsMagick o ffmpeg) para los formatos que necesitan compatibilidad con códecs externos.

Nota de seguridad: las listas de permitidos de URL se aplican antes de la descarga y en cada salto de redirección. Incluir un nombre de host en la lista de permitidos no elude el bloqueo de direcciones IP privadas o internas. Para los gateways expuestos a Internet, aplique controles de salida de red además de las protecciones a nivel de aplicación. Consulte [Seguridad](/es/gateway/security).

## Transmisión (SSE)

Establezca `stream: true` para recibir eventos enviados por el servidor:

- `Content-Type: text/event-stream`
- Cada línea de evento es `event: <type>` y `data: <json>`
- La transmisión finaliza con `data: [DONE]`

Tipos de eventos emitidos actualmente: `response.created`, `response.in_progress`, `response.output_item.added`, `response.content_part.added`, `response.output_text.delta`, `response.output_text.done`, `response.content_part.done`, `response.output_item.done`, `response.completed`, `response.failed` (en caso de error).

## Uso

`usage` se rellena cuando el proveedor subyacente informa del recuento de tokens. OpenClaw normaliza los alias habituales de estilo OpenAI antes de que esos contadores lleguen a las superficies posteriores de estado y sesión, incluidos `input_tokens` / `output_tokens` y `prompt_tokens` / `completion_tokens`.

## Errores

Los errores utilizan un objeto JSON como este:

```json
{ "error": { "message": "...", "type": "invalid_request_error" } }
```

Casos habituales: `400` cuerpo de solicitud no válido, `401` autenticación ausente o no válida, `403` ámbito de operador ausente, `405` método incorrecto, `429` demasiados intentos de autenticación fallidos (con `Retry-After`).

## Ejemplos

Sin transmisión:

```bash
curl -sS http://127.0.0.1:18789/v1/responses \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -H 'x-openclaw-agent-id: main' \
  -d '{
    "model": "openclaw",
    "input": "hi"
  }'
```

Con transmisión:

```bash
curl -N http://127.0.0.1:18789/v1/responses \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -H 'x-openclaw-agent-id: main' \
  -d '{
    "model": "openclaw",
    "stream": true,
    "input": "hi"
  }'
```

## Temas relacionados

- [Finalizaciones de chat de OpenAI](/es/gateway/openai-http-api)
- [Ámbitos de operador](/es/gateway/operator-scopes)
- [OpenAI](/es/providers/openai)
