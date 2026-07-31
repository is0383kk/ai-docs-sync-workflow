---
read_when:
    - Externe CLI-Integrationen hinzufügen oder ändern
    - RPC-Adapter debuggen (signal-cli, imsg)
summary: RPC-Adapter für externe CLIs (signal-cli, imsg) und Gateway-Muster
title: RPC-Adapteren
x-i18n:
    generated_at: "2026-07-26T18:37:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7deee8154dc824db4eccca9a26381711693972ba2606aec47d657e3724b3a5dd
    source_path: reference/rpc.md
    workflow: 16
---

OpenClaw integriert externe CLIs über JSON-RPC. Derzeit werden zwei Muster verwendet.

## Muster A: HTTP-Daemon (signal-cli)

- `signal-cli` wird als Daemon mit JSON-RPC über HTTP ausgeführt.
- Der Ereignisstream verwendet SSE (`/api/v1/events`).
- Integritätsprüfung: `/api/v1/check`.
- OpenClaw verwaltet den Lebenszyklus, wenn `channels.signal.transport.kind="managed-native"` (Standard).

Einrichtung und Endpunkte finden Sie unter [Signal](/de/channels/signal).

## Muster B: untergeordneter stdio-Prozess (imsg)

- OpenClaw startet `imsg rpc` als untergeordneten Prozess für [iMessage](/de/channels/imessage).
- JSON-RPC wird zeilenweise über stdin/stdout übertragen (ein JSON-Objekt pro Zeile).
- Kein TCP-Port und kein Daemon erforderlich.

Verwendete Kernmethoden:

- `watch.subscribe` → Benachrichtigungen (`method: "message"`)
- `watch.unsubscribe`
- `send`
- `chats.list` (Prüfung/Diagnose)

Einrichtung und Adressierung finden Sie unter [iMessage](/de/channels/imessage) (`chat_id` wird gegenüber Anzeigezeichenfolgen bevorzugt).

## Adapter-Richtlinien

- Der Gateway verwaltet den Prozess (Start/Stopp sind an den Lebenszyklus des Providers gekoppelt).
- RPC-Clients müssen robust bleiben: Zeitüberschreitungen verwenden und nach dem Beenden neu starten.
- Stabile IDs (z. B. `chat_id`) sind Anzeigezeichenfolgen vorzuziehen.

## Verwandte Themen

- [Gateway-Protokoll](/de/gateway/protocol)
