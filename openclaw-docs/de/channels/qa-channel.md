---
read_when:
    - Sie binden den synthetischen QA-Transport in einen lokalen oder CI-Testlauf ein.
    - Sie benötigen die Konfigurationsoberfläche des mitgelieferten qa-channel-Plugins.
    - Sie arbeiten iterativ an der End-to-End-QA-Automatisierung
summary: Synthetisches Plugin für einen Slack-ähnlichen Kanal für deterministische OpenClaw-QA-Szenarien
title: QA-Kanal
x-i18n:
    generated_at: "2026-07-26T18:14:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a43c35e197116a6bd44b238010eb508aed23dea99ab872d10e6fc853b5f4d4a7
    source_path: channels/qa-channel.md
    workflow: 16
---

`qa-channel` ist ein repo-lokaler synthetischer Nachrichtentransport für automatisierte OpenClaw-QA (`extensions/qa-channel`, privates Paket, von paketierten Installationen ausgeschlossen). Es handelt sich nicht um einen Produktionskanal – er dient dazu, dieselbe Channel-Plugin-Grenze zu testen, die von realen Transporten verwendet wird, während der Zustand deterministisch und vollständig einsehbar bleibt.

## Funktionsweise

- Zielgrammatik der Slack-Klasse:
  - `dm:<user>`
  - `channel:<room>`
  - `group:<room>`
  - `thread:<room>/<thread>`
- Gemeinsame `channel:`- und `group:`-Unterhaltungen werden Agenten als Gruppen-/Channel-Raum-Turns bereitgestellt, sodass sie dieselben Routing-Richtlinien für sichtbare Antworten und Nachrichtentools testen, die von Discord, Slack, Telegram und ähnlichen Transporten verwendet werden.
- HTTP-gestützter synthetischer Bus zum Einspeisen eingehender Nachrichten, Erfassen ausgehender Transkripte, Erstellen von Threads sowie für Reaktionen, Bearbeitungen, Löschungen und Such-/Leseaktionen.
- Hostseitiger Selbsttest-Runner, der einen Markdown-Bericht unter `.artifacts/qa-e2e/` schreibt.

## Konfiguration

```json
{
  "channels": {
    "qa-channel": {
      "baseUrl": "http://127.0.0.1:43123",
      "botUserId": "openclaw",
      "botDisplayName": "OpenClaw QA",
      "allowFrom": ["*"],
      "pollTimeoutMs": 1000
    }
  }
}
```

Kontoschlüssel:

- `enabled` – Hauptschalter für dieses Konto.
- `name` – optionale Anzeigebezeichnung.
- `baseUrl` – URL des synthetischen Busses. Das Konto gilt als konfiguriert, sobald diese festgelegt ist.
- `botUserId` – synthetische Bot-Benutzer-ID, die in der Zielgrammatik verwendet wird (Standard: `openclaw`).
- `botDisplayName` – Anzeigename für ausgehende Nachrichten (Standard: `OpenClaw QA`).
- `pollTimeoutMs` – Wartefenster für Long Polling. Ganzzahl zwischen 100 und 30000 (Standard: 1000).
- `allowFrom` – Absender-Zulassungsliste (Benutzer-IDs oder `"*"`; Standard: `["*"]`). Direktnachrichten verwenden
  immer die Richtlinie `open`; die Gruppenrichtlinie mit Zulassungsliste verwendet ebenfalls diese synthetischen
  Absender-IDs.
- `groupPolicy` – Richtlinie für gemeinsam genutzte Räume: `"open"` (Standard), `"allowlist"` oder
  `"disabled"`.
- `groupAllowFrom` – optionale Absender-Zulassungsliste für gemeinsam genutzte Räume. Wenn sie unter
  `"allowlist"` nicht angegeben ist, greift QA Channel auf `allowFrom` zurück.
- `groups.<room>.requireMention` – erfordert eine Bot-Erwähnung, bevor in einem
  bestimmten Gruppen-/Channel-Raum geantwortet wird (Standard: false). `groups."*"` legt den Standard fest;
  raumspezifische `tools` / `toolsBySender` legen Überschreibungen der Tool-Richtlinie fest.
- `defaultTo` – Ersatzziel, wenn keines angegeben ist.
- `actions.messages` / `actions.reactions` / `actions.search` / `actions.threads` – aktionsspezifische Tool-Zugriffssteuerung.

Kontenübergreifende Schlüssel auf oberster Ebene:

- `accounts` – Datensatz benannter kontospezifischer Überschreibungen, indiziert nach Konto-ID.
- `defaultAccount` – bevorzugte Konto-ID, wenn mehrere Konten konfiguriert sind.

## Runner

Hostseitiger Selbsttest (schreibt einen Markdown-Bericht unter `.artifacts/qa-e2e/`):

```bash
pnpm qa:e2e
```

Dieser wird über `qa-lab` geleitet, startet den repo-internen QA-Bus, bootet den Runtime-Ausschnitt `qa-channel` und führt einen deterministischen Selbsttest aus.

Vollständige repo-gestützte Szenariosuite:

```bash
pnpm openclaw qa suite
```

Führt Szenarien parallel auf der QA-Gateway-Lane aus. Informationen zu Szenarien, Profilen und Provider-Modi finden Sie in der [QA-Übersicht](/de/concepts/qa-e2e-automation).

Docker-gestützte QA-Site (Gateway und QA-Lab-Debugger-Benutzeroberfläche in einem Stack):

```bash
pnpm qa:lab:up
```

Erstellt die QA-Site, startet den Docker-gestützten Gateway- und QA-Lab-Stack und gibt die QA-Lab-URL aus. Von dort aus können Sie Szenarien auswählen, die Modell-Lane festlegen, einzelne Läufe starten und die Ergebnisse live verfolgen. Der QA-Lab-Debugger ist vom ausgelieferten Control-UI-Bundle getrennt.

## Verwandte Themen

- [QA-Übersicht](/de/concepts/qa-e2e-automation) – Gesamt-Stack, Transportadapter, Matrix-Profile und Szenarioerstellung
- [Kopplung](/de/channels/pairing)
- [Gruppen](/de/channels/groups)
- [Channel-Übersicht](/de/channels)
