---
read_when:
    - Configuración de Synology Chat con OpenClaw
    - Depuración del enrutamiento de webhooks de Synology Chat
summary: Configuración del Webhook de Synology Chat y de OpenClaw
title: Synology Chat
x-i18n:
    generated_at: "2026-07-26T05:01:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3c03379944ee4187260a7287f6d2aed1ad8fdd1c22b5581c8a5d55515bbb6ad5
    source_path: channels/synology-chat.md
    workflow: 16
---

Synology Chat se conecta a OpenClaw mediante un par de webhooks: un webhook saliente de Synology Chat publica los mensajes directos entrantes en el Gateway, y las respuestas regresan mediante un webhook entrante de Synology Chat.

Estado: plugin oficial, instalado por separado. Solo mensajes directos; se admite el envío de texto y archivos mediante URL.

## Instalación

```bash
openclaw plugins install @openclaw/synology-chat
```

Checkout local (cuando se ejecuta desde un repositorio git):

```bash
openclaw plugins install ./path/to/local/synology-chat-plugin
```

Detalles: [Plugins](/es/tools/plugin)

## Configuración rápida

1. Instale el plugin (indicado anteriormente).
2. En las integraciones de Synology Chat:
   - Cree un webhook entrante y copie su URL.
   - Cree un webhook saliente con su token secreto.
3. Dirija la URL del webhook saliente a su Gateway de OpenClaw:
   - `https://gateway-host/webhook/synology` de forma predeterminada.
   - O su `channels.synology-chat.webhookPath` personalizado.
4. Finalice la configuración en OpenClaw. Synology Chat aparece en la misma lista de configuración de canales en ambos flujos:
   - Guiado: `openclaw onboard` o `openclaw channels add`
   - Directo: `openclaw channels add --channel synology-chat --token <token> --url <incoming-webhook-url>`
5. Reinicie el Gateway y envíe un mensaje directo al bot de Synology Chat.

Detalles de autenticación del webhook:

- OpenClaw acepta el token del webhook saliente desde `body.token`, después
  `?token=...` y, por último, las cabeceras.
- Formatos de cabecera aceptados:
  - `x-synology-token`
  - `x-webhook-token`
  - `x-openclaw-token`
  - `Authorization: Bearer <token>`
- Los tokens vacíos o ausentes provocan un rechazo seguro.
- Las cargas útiles pueden ser `application/x-www-form-urlencoded` o `application/json`; `token`, `user_id` y `text` son obligatorios.

## Durabilidad de los mensajes entrantes

Una vez superadas las comprobaciones del token, la política de remitentes y el límite de frecuencia, OpenClaw elimina el token del webhook del sobre almacenado y pone el evento en una cola duradera antes de confirmarlo. La ruta devuelve `204` únicamente después de que esa incorporación se complete correctamente; un error de persistencia devuelve `503` para que Synology Chat pueda volver a intentarlo en lugar de perder el mensaje de forma silenciosa.

Los eventos pendientes o reintentables sobreviven al reinicio del Gateway. El `post_id` estable de Synology evita entradas duplicadas en la cola mientras exista el registro correspondiente de finalización activo o retenido. La entrega sigue siendo al menos una vez en la transferencia de la cola al agente, por lo que un fallo en ese límite aún puede reproducir un turno.

Configuración mínima:

```json5
{
  channels: {
    "synology-chat": {
      enabled: true,
      token: "synology-outgoing-token",
      incomingUrl: "https://nas.example.com/webapi/entry.cgi?api=SYNO.Chat.External&method=incoming&version=2&token=...",
      webhookPath: "/webhook/synology",
      dmPolicy: "allowlist",
      allowedUserIds: ["123456"],
      rateLimitPerMinute: 30,
      allowInsecureSsl: false,
    },
  },
}
```

## Variables de entorno

Para la cuenta predeterminada, se pueden usar variables de entorno:

- `SYNOLOGY_CHAT_TOKEN`
- `SYNOLOGY_CHAT_INCOMING_URL`
- `SYNOLOGY_NAS_HOST`
- `SYNOLOGY_ALLOWED_USER_IDS` (separados por comas)
- `SYNOLOGY_RATE_LIMIT`
- `OPENCLAW_BOT_NAME`

Los valores de configuración prevalecen sobre las variables de entorno.

`SYNOLOGY_CHAT_INCOMING_URL` y `SYNOLOGY_NAS_HOST` no se pueden establecer desde un `.env` del espacio de trabajo; consulte [Archivos `.env` del espacio de trabajo](/es/gateway/security#workspace-env-files).

## Política de mensajes directos y control de acceso

- Valores admitidos para `dmPolicy`: `allowlist` (predeterminado), `open` y `disabled`. Synology Chat no tiene un flujo de emparejamiento; autorice a los remitentes añadiendo sus identificadores numéricos de usuario de Synology a `allowedUserIds`.
- `allowedUserIds` acepta una lista (o una cadena separada por comas) de identificadores de usuario de Synology.
- En el modo `allowlist`, una lista `allowedUserIds` vacía se considera una configuración incorrecta y la ruta del webhook no se iniciará.
- `dmPolicy: "open"` permite mensajes directos públicos únicamente cuando `allowedUserIds` incluye `"*"`; con entradas restrictivas, solo los usuarios coincidentes pueden conversar. `open` con una lista `allowedUserIds` vacía también impide que se inicie la ruta.
- `dmPolicy: "disabled"` bloquea los mensajes directos.
- La vinculación del destinatario de las respuestas permanece de forma predeterminada en el `user_id` numérico estable. `channels.synology-chat.dangerouslyAllowNameMatching: true` es un modo de compatibilidad de emergencia que vuelve a habilitar la búsqueda por nombre de usuario o alias modificable para entregar las respuestas.

## Entrega saliente

Use identificadores numéricos de usuario de Synology Chat como destinos. Se aceptan los prefijos `synology-chat:`, `synology_chat:` y `synology:`.

Ejemplos:

```bash
openclaw message send --channel synology-chat --target 123456 --message "Hola desde OpenClaw"
openclaw message send --channel synology-chat --target synology-chat:123456 --message "Hola de nuevo"
openclaw message send --channel synology-chat --target synology:123456 --message "Prefijo corto"
```

El texto saliente se divide en fragmentos de 2000 caracteres. Se admite el envío de contenido multimedia mediante la entrega de archivos basada en URL: el NAS descarga y adjunta el archivo (máximo de 32 MB). Las URL de archivos salientes deben usar `http` o `https`, y los destinos de red privados o bloqueados de otro modo se rechazan antes de que OpenClaw reenvíe la URL al webhook del NAS.

## Varias cuentas

Se admiten varias cuentas de Synology Chat en `channels.synology-chat.accounts`.
Cada cuenta puede sustituir el token, la URL entrante, la ruta del webhook, la política de mensajes directos y los límites.
Las sesiones de mensajes directos se aíslan por cuenta y usuario, por lo que el mismo `user_id` numérico
en dos cuentas de Synology diferentes no comparte el estado de la transcripción.
Asigne a cada cuenta habilitada un `webhookPath` distinto. OpenClaw rechaza rutas exactas duplicadas
y se niega a iniciar cuentas con nombre que solo hereden una ruta de webhook compartida en configuraciones con varias cuentas.
Si se necesita intencionadamente la herencia antigua para una cuenta con nombre, establezca
`dangerouslyAllowInheritedWebhookPath: true` en esa cuenta o en `channels.synology-chat`,
pero las rutas exactas duplicadas se siguen rechazando de forma segura. Se recomienda usar rutas explícitas para cada cuenta.

```json5
{
  channels: {
    "synology-chat": {
      enabled: true,
      accounts: {
        default: {
          token: "token-a",
          incomingUrl: "https://nas-a.example.com/...token=...",
        },
        alerts: {
          token: "token-b",
          incomingUrl: "https://nas-b.example.com/...token=...",
          webhookPath: "/webhook/synology-alerts",
          dmPolicy: "allowlist",
          allowedUserIds: ["987654"],
        },
      },
    },
  },
}
```

## Notas de seguridad

- Mantenga en secreto `token` y cámbielo si se filtra.
- Mantenga `allowInsecureSsl: false` salvo que confíe explícitamente en un certificado local autofirmado del NAS.
- Las solicitudes entrantes del webhook se verifican mediante token y se someten a un límite de frecuencia por remitente (`rateLimitPerMinute`, 30 de forma predeterminada).
- Las comprobaciones de tokens no válidos usan una comparación de secretos en tiempo constante y producen un rechazo seguro; los intentos repetidos con tokens no válidos bloquean temporalmente la dirección IP de origen.
- El texto de los mensajes entrantes se depura para eliminar patrones conocidos de inyección de instrucciones y se trunca a 4000 caracteres.
- Se recomienda `dmPolicy: "allowlist"` para producción.
- Mantenga `dangerouslyAllowNameMatching` desactivado salvo que necesite explícitamente la entrega antigua de respuestas basada en nombres de usuario.
- Mantenga `dangerouslyAllowInheritedWebhookPath` desactivado salvo que acepte explícitamente el riesgo de enrutamiento por rutas compartidas en una configuración con varias cuentas.

## Solución de problemas

- `Missing required fields (token, user_id, text)`:
  - a la carga útil del webhook saliente le falta uno de los campos obligatorios
  - si Synology envía el token en las cabeceras, asegúrese de que el gateway o proxy conserve esas cabeceras
- `Invalid token`:
  - el secreto del webhook saliente no coincide con `channels.synology-chat.token`
  - la solicitud está llegando a la cuenta o ruta de webhook incorrecta
  - un proxy inverso eliminó la cabecera del token antes de que la solicitud llegara a OpenClaw
- `Rate limit exceeded`:
  - demasiados intentos con tokens no válidos desde el mismo origen pueden bloquearlo temporalmente
  - los remitentes autenticados también tienen un límite de frecuencia de mensajes independiente por usuario
- `Allowlist is empty. Configure allowedUserIds or use dmPolicy=open with allowedUserIds=["*"].`:
  - `dmPolicy="allowlist"` está habilitado, pero no hay usuarios configurados
- `User not authorized`:
  - el `user_id` numérico del remitente no figura en `allowedUserIds`

## Contenido relacionado

- [Descripción general de los canales](/es/channels) — todos los canales admitidos
- [Grupos](/es/channels/groups) — comportamiento del chat grupal y control mediante menciones
- [Enrutamiento de canales](/es/channels/channel-routing) — enrutamiento de sesiones para mensajes
- [Seguridad](/es/gateway/security) — modelo de acceso y protección
