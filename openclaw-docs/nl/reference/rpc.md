---
read_when:
    - Externe CLI-integraties toevoegen of wijzigen
    - RPC-adapters debuggen (signal-cli, imsg)
summary: RPC-adapters voor externe CLI's (signal-cli, imsg) en Gateway-patronen
title: RPC-adapters
x-i18n:
    generated_at: "2026-07-27T05:50:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7deee8154dc824db4eccca9a26381711693972ba2606aec47d657e3724b3a5dd
    source_path: reference/rpc.md
    workflow: 16
---

OpenClaw integreert externe CLI's via JSON-RPC. Momenteel worden twee patronen gebruikt.

## Patroon A: HTTP-daemon (signal-cli)

- `signal-cli` wordt uitgevoerd als daemon met JSON-RPC via HTTP.
- De gebeurtenisstroom is SSE (`/api/v1/events`).
- Statuscontrole: `/api/v1/check`.
- OpenClaw beheert de levenscyclus bij `channels.signal.transport.kind="managed-native"` (de standaardinstelling).

Zie [Signal](/nl/channels/signal) voor de configuratie en eindpunten.

## Patroon B: onderliggend stdio-proces (imsg)

- OpenClaw start `imsg rpc` als onderliggend proces voor [iMessage](/nl/channels/imessage).
- JSON-RPC wordt regelgescheiden via stdin/stdout verzonden (één JSON-object per regel).
- Geen TCP-poort en geen daemon vereist.

Gebruikte kernmethoden:

- `watch.subscribe` → meldingen (`method: "message"`)
- `watch.unsubscribe`
- `send`
- `chats.list` (controle/diagnostiek)

Zie [iMessage](/nl/channels/imessage) voor de configuratie en adressering (`chat_id` heeft de voorkeur boven weergaveteksten).

## Richtlijnen voor adapters

- De Gateway beheert het proces (starten/stoppen is gekoppeld aan de levenscyclus van de provider).
- Maak RPC-clients robuust: gebruik time-outs en start opnieuw na afsluiting.
- Geef de voorkeur aan stabiele ID's (bijvoorbeeld `chat_id`) boven weergaveteksten.

## Gerelateerd

- [Gateway-protocol](/nl/gateway/protocol)
