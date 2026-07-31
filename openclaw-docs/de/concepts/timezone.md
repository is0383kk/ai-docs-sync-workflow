---
read_when:
    - Sie möchten sich schnell ein grundlegendes Verständnis der Zeitzonenbehandlung verschaffen
    - Sie entscheiden, wo Sie eine Zeitzone festlegen oder überschreiben möchten
summary: Wo Zeitzonen in OpenClaw erscheinen – in Umschlägen, Tool-Payloads und im System-Prompt
title: Zeitzonen
x-i18n:
    generated_at: "2026-07-26T17:46:26Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9d1620b4b2cedba89bd6ab4392018cd48d0ef92a6abc1744011d482557e2c4fc
    source_path: concepts/timezone.md
    workflow: 16
---

OpenClaw standardisiert Zeitstempel, sodass das Modell statt einer Mischung aus Provider-lokalen Uhren eine **einheitliche Referenzzeit** sieht. Drei Bereiche zeigen Zeitzonen an, jeweils für einen eigenen Zweck:

## Drei Zeitzonenbereiche

| Bereich              | Anzeige                                                                                                    | Standard                                  | Konfiguriert über                                      |
| -------------------- | ---------------------------------------------------------------------------------------------------------- | ----------------------------------------- | ------------------------------------------------------ |
| Nachrichtenumschläge | Umschließt eingehende Kanalnachrichten: `[Signal +1555 Sun 2026-01-18 00:19:42 PST] hello`                                                 | Host-lokal                                | `agents.defaults.envelopeTimezone`                                     |
| Tool-Nutzdaten       | Kanaltools im `readMessages`-Stil geben die unverarbeitete Provider-Zeit sowie normalisierte Werte für `timestampMs` / `timestampUtc` zurück | UTC-Felder sind immer vorhanden           | Nicht konfigurierbar; behält Provider-native Zeitstempel bei |
| System-Prompt        | Ein kleiner `Current Date & Time`-Block, der **nur die Zeitzone** enthält (keinen Uhrzeitwert, um die Cache-Stabilität zu gewährleisten) | Host-Zeitzone, wenn `userTimezone` nicht gesetzt ist | `agents.defaults.userTimezone`                                     |

Der System-Prompt lässt die aktuelle Uhrzeit bewusst aus, damit das Prompt-Caching über mehrere Durchläufe hinweg stabil bleibt. Wenn der Agent die aktuelle Uhrzeit benötigt, ruft er `session_status` auf.

## Benutzerzeitzone festlegen

```json5
{
  agents: {
    defaults: {
      userTimezone: "America/Chicago",
    },
  },
}
```

Wenn `userTimezone` nicht gesetzt ist, ermittelt OpenClaw die Host-Zeitzone zur Laufzeit über `Intl.DateTimeFormat().resolvedOptions().timeZone` (ohne die Konfiguration zu ändern). `agents.defaults.timeFormat` (`auto` | `12` | `24`) steuert die Darstellung im 12-/24-Stunden-Format in Umschlägen und nachgelagerten Bereichen, nicht jedoch im Abschnitt des System-Prompts.

## Werte für die Zeitzone des Umschlags

`agents.defaults.envelopeTimezone` akzeptiert:

- `"local"` (Standard) oder `"host"` – Zeitzone des Host-Rechners.
- `"utc"` oder `"gmt"` – UTC.
- `"user"` – die ermittelte `agents.defaults.userTimezone` (fällt auf die Host-Zeitzone zurück, wenn sie nicht gesetzt ist).
- Eine beliebige explizite IANA-Zeitzonenkennung, z. B. `"Europe/Vienna"`.

## Wann eine Überschreibung sinnvoll ist

- **Verwenden Sie `"utc"`** für konsistente Zeitstempel auf Hosts in unterschiedlichen Regionen oder zur Abstimmung auf UTC-basierte Diagnose-/Protokollausgaben.
- **Verwenden Sie `"user"`**, damit Umschläge unabhängig von der Zeitzone des Gateway-Hosts an der konfigurierten Benutzerzeitzone ausgerichtet bleiben.
- **Verwenden Sie eine feste IANA-Zeitzone**, wenn sich der Gateway-Host in einer Zeitzone befindet, der Umschlag aber unabhängig von einer Host-Migration immer in einer anderen Zeitzone angezeigt werden soll.
- **Setzen Sie `envelopeTimestamp: "off"`**, wenn der Zeitstempelkontext für die Unterhaltung nicht nützlich ist. Dadurch werden absolute Zeitstempel aus Umschlägen, direkten Präfixen des Agenten-Prompts und eingebetteten Präfixen der Modelleingabe entfernt.

Die vollständige Verhaltensreferenz, Beispiele für jeden Provider und die Formatierung verstrichener Zeit finden Sie unter [Datum und Uhrzeit](/de/date-time).

## Verwandte Themen

- [Datum und Uhrzeit](/de/date-time) – vollständiges Verhalten und Beispiele für Umschläge, Tools und Prompts.
- [Heartbeat](/de/gateway/heartbeat) – aktive Zeiten verwenden die Zeitzone für die Planung.
- [Cron-Aufgaben](/de/automation/cron-jobs) – Cron-Ausdrücke verwenden die Zeitzone für die Planung.
