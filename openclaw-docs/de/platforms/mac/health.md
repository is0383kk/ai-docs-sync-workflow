---
read_when:
    - Fehlerbehebung bei Statusanzeigen der Mac-App
summary: Wie die macOS-App den Zustand von Gateway und Kanälen meldet
title: Systemprüfungen (macOS)
x-i18n:
    generated_at: "2026-07-26T17:55:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 095abdbefa7db7c0d14435e2c5db7d1ebc03afa0c539555a7abdd9170d015fb8
    source_path: platforms/mac/health.md
    workflow: 16
---

# Systemdiagnosen unter macOS

So lesen Sie den Systemzustand verknüpfter Kanäle in der Menüleisten-App ab.

## Menüleiste

Statuspunkt:

- Grün: verknüpft und Diagnose erfolgreich.
- Orange: verknüpft, aber eine Kanaldiagnose meldet einen eingeschränkten Zustand oder keine Verbindung.
- Rot: noch nicht verknüpft.

Die zweite Zeile lautet „verknüpft · Authentifizierung vor 12 Min.“ oder zeigt den Fehlergrund an.
„Run Health Check Now“ im Menü löst eine bedarfsgesteuerte Diagnose aus.

## Einstellungen

- Die Registerkarte „Allgemein“ zeigt eine Karte zum Systemzustand: Statuspunkt, Zusammenfassungszeile (Verknüpfungsstatus +
  Alter der Authentifizierung) und optional eine Zeile mit Fehlerdetails sowie die Schaltflächen **Jetzt erneut versuchen** und
  **Protokolle öffnen**.
- Die **Registerkarte „Kanäle“** zeigt für WhatsApp und Telegram den Status und die Steuerelemente der einzelnen Kanäle an (Anmelde-QR-Code,
  Abmelden, Diagnose, letzte Trennung/letzter Fehler).

## Funktionsweise der Diagnose

Die App ruft alle ~60 Sekunden und bei Bedarf über ihre bestehende WebSocket-
Verbindung (nicht durch Aufruf einer CLI-Shell) die RPC `health` des Gateways auf. Die RPC lädt
Anmeldedaten und meldet den Status, ohne Nachrichten zu senden. Die App speichert den letzten
erfolgreichen Snapshot und den letzten Fehler getrennt zwischen, sodass die Benutzeroberfläche sofort geladen wird und
im Offlinebetrieb nicht flackert.

## Im Zweifelsfall

Verwenden Sie den CLI-Ablauf unter [Gateway-Systemzustand](/de/gateway/health) (`openclaw status`,
`openclaw status --deep`, `openclaw health --json`) und führen Sie
`openclaw logs --follow` aus, gefiltert nach `web-heartbeat` / `web-reconnect`.

## Verwandte Themen

- [Gateway-Systemzustand](/de/gateway/health)
- [macOS-App](/de/platforms/macos)
