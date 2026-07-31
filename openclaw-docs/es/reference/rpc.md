---
read_when:
    - Añadir o cambiar integraciones de CLI externas
    - Depuración de adaptadores RPC (signal-cli, imsg)
summary: Adaptadores RPC para CLI externas (signal-cli, imsg) y patrones de Gateway
title: Adaptadores RPC
x-i18n:
    generated_at: "2026-07-26T05:21:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7deee8154dc824db4eccca9a26381711693972ba2606aec47d657e3724b3a5dd
    source_path: reference/rpc.md
    workflow: 16
---

OpenClaw integra CLI externas mediante JSON-RPC. Actualmente se utilizan dos patrones.

## Patrón A: demonio HTTP (signal-cli)

- `signal-cli` se ejecuta como demonio con JSON-RPC sobre HTTP.
- El flujo de eventos es SSE (`/api/v1/events`).
- Sondeo de estado: `/api/v1/check`.
- OpenClaw controla el ciclo de vida cuando `channels.signal.transport.kind="managed-native"` (valor predeterminado).

Consulte [Signal](/es/channels/signal) para conocer la configuración y los endpoints.

## Patrón B: proceso secundario mediante stdio (imsg)

- OpenClaw inicia `imsg rpc` como proceso secundario para [iMessage](/es/channels/imessage).
- JSON-RPC se delimita por líneas mediante stdin/stdout (un objeto JSON por línea).
- No se requiere ningún puerto TCP ni demonio.

Métodos principales utilizados:

- `watch.subscribe` → notificaciones (`method: "message"`)
- `watch.unsubscribe`
- `send`
- `chats.list` (sondeo/diagnóstico)

Consulte [iMessage](/es/channels/imessage) para conocer la configuración y el direccionamiento (se prefiere `chat_id` frente a las cadenas de visualización).

## Directrices para adaptadores

- Gateway controla el proceso (el inicio y la detención están vinculados al ciclo de vida del proveedor).
- Mantenga la resiliencia de los clientes RPC: tiempos de espera y reinicio al finalizar.
- Prefiera identificadores estables (por ejemplo, `chat_id`) frente a las cadenas de visualización.

## Contenido relacionado

- [Protocolo de Gateway](/es/gateway/protocol)
