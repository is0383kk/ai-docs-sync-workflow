---
read_when:
    - Sie ändern, wie Zeitstempel dem Modell oder den Benutzern angezeigt werden
    - Sie debuggen die Zeitformatierung in Nachrichten oder in der Ausgabe des System-Prompts
summary: Datums- und Zeitverarbeitung in Envelopes, Prompts, Tools und Konnektoren
title: Datum und Uhrzeit
x-i18n:
    generated_at: "2026-07-26T18:56:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e6f923022c021c1cf18ba306cd7b9a4873f5df947bb9a8fae9c737a89f64cbf2
    source_path: date-time.md
    workflow: 16
---

OpenClaw verwendet **die lokale Zeit des Hosts für Transportzeitstempel** und fügt **nur die Zeitzone** in den System-Prompt ein.
Provider-Zeitstempel bleiben erhalten, damit Tools ihre nativen Semantiken beibehalten. Wenn der Agent die aktuelle
Uhrzeit benötigt, führt er das Tool `session_status` aus.

## Nachrichtenumschläge (standardmäßig lokal)

Eingehende Nachrichten werden mit einem Wochentag und einem sekundengenauen Zeitstempel umschlossen:

```
[WhatsApp +1555 Mon 2026-01-05 16:26:34 PST] Nachrichtentext
```

Der Zeitstempel des Umschlags ist **standardmäßig hostlokal**, unabhängig von der Zeitzone des Providers.
Überschreiben Sie dies unter `agents.defaults`:

```json5
{
  agents: {
    defaults: {
      envelopeTimezone: "local", // "utc" | "local" | "user" | IANA-Zeitzone
      envelopeTimestamp: "on", // "on" | "off"
      envelopeElapsed: "on", // "on" | "off"
    },
  },
}
```

| Schlüssel             | Werte                                                | Verhalten                                                                                                                                                                       |
| --------------------- | ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `envelopeTimezone`  | `local` (Standard), `utc`, `user`, expliziter IANA-Name | `user` verwendet `agents.defaults.userTimezone` (Zeitzone des Hosts, wenn nicht festgelegt). Ein expliziter IANA-Name (z. B. `"America/Chicago"`) legt eine feste Zone fest; nicht erkannte Namen fallen auf UTC zurück. |
| `envelopeTimestamp` | `on` (Standard), `off`                                | `off` entfernt absolute Zeitstempel aus Umschlag-Headern, direkten Präfixen des Agent-Prompts und eingebetteten Präfixen der Modelleingabe.                                                       |
| `envelopeElapsed`   | `on` (Standard), `off`                                | `off` entfernt das Suffix für die verstrichene Zeit (im Stil von `+30s` / `+2m`), das die Zeit seit der vorherigen Nachricht in der Sitzung angibt.                                                               |

### Beispiele

**Lokal (Standard):**

```
[WhatsApp +1555 Sun 2026-01-18 00:19:42 PST] Hallo
```

**Zeitzone des Benutzers:**

```
[WhatsApp +1555 Sun 2026-01-18 00:19:42 CST] Hallo
```

**Verstrichene Zeit mit `envelopeTimezone: "utc"`:**

```
[WhatsApp +1555 +30s Sun 2026-01-18T05:19:00Z] Folgenachricht
```

## System-Prompt: aktuelles Datum und aktuelle Uhrzeit

Der System-Prompt enthält einen Abschnitt **Aktuelles Datum und aktuelle Uhrzeit**, der **nur die Zeitzone**
(keine Uhrzeit und kein Zeitformat) enthält, damit das Prompt-Caching stabil bleibt:

```
Zeitzone: America/Chicago
```

Die Zone ist `agents.defaults.userTimezone`, wenn sie konfiguriert ist, andernfalls die Zeitzone des Hosts.
Der Prompt weist den Agenten außerdem an, das Tool `session_status` auszuführen, wenn er das
aktuelle Datum, die aktuelle Uhrzeit oder den aktuellen Wochentag benötigt.

## Systemereigniszeilen (standardmäßig lokal)

Systemereignisse in der Warteschlange, die in den Agentenkontext eingefügt werden, erhalten einen Zeitstempel als Präfix, der dieselbe
Auswahl über `envelopeTimezone` wie Nachrichtenumschläge verwendet (Standard: hostlokal).

```
System: [2026-01-12 12:19:17 PST] Modell gewechselt.
```

### Zeitzone und Format des Benutzers konfigurieren

```json5
{
  agents: {
    defaults: {
      userTimezone: "America/Chicago",
      timeFormat: "auto", // auto | 12 | 24
    },
  },
}
```

- `userTimezone` legt die **lokale Zeitzone des Benutzers** für den Prompt-Kontext (und für `envelopeTimezone: "user"`) fest.
- `timeFormat` steuert die **12-/24-Stunden-Anzeige** für Uhrzeiten im Prompt. `auto` folgt den Betriebssystemeinstellungen.

## Erkennung des Zeitformats (automatisch)

Bei `timeFormat: "auto"` prüft OpenClaw die Betriebssystemeinstellung (macOS und Windows)
und greift andernfalls auf die Formatierung des Gebietsschemas zurück. Der erkannte Wert wird **prozessbezogen zwischengespeichert**,
um wiederholte Systemaufrufe zu vermeiden.

## Tool-Nutzdaten und Konnektoren (unverarbeitete Provider-Zeit + normalisierte Felder)

Kanal-Tools geben **Provider-native Zeitstempel** zurück und fügen zur Konsistenz normalisierte Felder hinzu:

- `timestampMs`: Epoch-Millisekunden (UTC)
- `timestampUtc`: ISO-8601-UTC-Zeichenfolge

Die unverarbeiteten Provider-Felder bleiben erhalten, damit nichts verloren geht.

- Discord: ISO-Zeitstempel in UTC
- Slack: Epoch-ähnliche Zeichenfolgen aus der API
- Telegram/WhatsApp: providerspezifische numerische/ISO-Zeitstempel

Wenn Sie die lokale Zeit benötigen, konvertieren Sie sie nachgelagert mithilfe der bekannten Zeitzone.

## Zugehörige Dokumentation

- [System-Prompt](/de/concepts/system-prompt)
- [Zeitzonen](/de/concepts/timezone)
- [Nachrichten](/de/concepts/messages)
