---
read_when:
    - Depuración de los indicadores de estado de la aplicación para Mac
summary: Cómo informa la aplicación para macOS sobre los estados de salud del Gateway y los canales
title: Comprobaciones de estado (macOS)
x-i18n:
    generated_at: "2026-07-26T04:45:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 095abdbefa7db7c0d14435e2c5db7d1ebc03afa0c539555a7abdd9170d015fb8
    source_path: platforms/mac/health.md
    workflow: 16
---

# Comprobaciones de estado en macOS

Cómo consultar el estado de los canales vinculados desde la aplicación de la barra de menús.

## Barra de menús

Punto de estado:

- Verde: vinculado + sondeo correcto.
- Naranja: vinculado, pero el sondeo de un canal informa que está degradado/no conectado.
- Rojo: aún no vinculado.

La línea secundaria muestra "vinculado · autenticación 12m" o el motivo del fallo.
"Run Health Check Now" en el menú inicia un sondeo bajo demanda.

## Ajustes

- La pestaña General muestra una tarjeta de estado: punto de estado, línea de resumen (estado del vínculo +
  antigüedad de la autenticación) y una línea opcional con los detalles del fallo, con los botones **Reintentar ahora** y
  **Abrir registros**.
- La **pestaña Canales** muestra el estado y los controles de cada canal (código QR de inicio de sesión,
  cierre de sesión, sondeo, última desconexión/error) para WhatsApp y Telegram.

## Cómo funciona el sondeo

La aplicación llama al RPC `health` del Gateway mediante su conexión WebSocket
existente (sin ejecutar un proceso de la CLI) cada ~60s y bajo demanda. El RPC carga
las credenciales e informa del estado sin enviar mensajes. La aplicación almacena en caché por separado la última
instantánea correcta y el último error, para que la interfaz de usuario se cargue al instante y
no parpadee mientras está sin conexión.

## En caso de duda

Utilice el flujo de la CLI descrito en [Estado del Gateway](/es/gateway/health) (`openclaw status`,
`openclaw status --deep`, `openclaw health --json`) y ejecute
`openclaw logs --follow`, filtrando por `web-heartbeat` / `web-reconnect`.

## Relacionado

- [Estado del Gateway](/es/gateway/health)
- [Aplicación para macOS](/es/platforms/macos)
