---
read_when:
    - Arbeiten an Telemetrie-/Datenschutzkontrollen
    - Fragen dazu, welche Daten erhoben werden
summary: Vom ClawHub-CLI erfasste Installationstelemetrie und wie Sie deren Erfassung deaktivieren.
x-i18n:
    generated_at: "2026-07-26T18:16:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a02bb1c76fea3105255235f6314ade73f260f692d6eb1b41f8001dc84db6ded7
    source_path: clawhub/telemetry.md
    workflow: 16
---

# Telemetrie

ClawHub verwendet minimale CLI-Telemetrie, um aggregierte Installationszahlen für Skills und Plugins zu berechnen.

## Wann Telemetrie erfasst wird

Telemetrie wird nur gesendet, wenn:

- Sie in der CLI angemeldet sind.
- Sie `clawhub install <slug>` ausführen oder eine authentifizierte Installation von
  `openclaw plugins install clawhub:<package>` abschließen.
- Telemetrie **nicht deaktiviert** ist (siehe „Deaktivieren der Telemetrie“ weiter unten).

Wenn Sie nicht angemeldet sind, werden keine Daten gemeldet.

## Welche Daten wir erfassen

Nachdem ein Skill oder Plugin installiert und der lokale Installationsdatensatz gespeichert wurde, sendet die CLI
nach dem Best-Effort-Prinzip ein einzelnes Installationsereignis.

Das Ereignis enthält:

- Den Slug des installierten Skills oder den kanonischen Paketnamen des Plugins.
- `version`: die installierte Version, sofern bekannt.

### Welche Daten wir _nicht_ erfassen

- Keine Ordnerpfade oder aus Ordnern abgeleiteten Kennungen.
- Keine Dateiinhalte.
- Keine laufbezogenen Protokolle, Prompts oder sonstigen CLI-Ausgaben.

## Installationszahlen

Für Skills verwaltet ClawHub:

- `installsAllTime`: eindeutige Benutzer, die mindestens eine CLI-Installation des Skills gemeldet haben.
- `installsCurrent`: eindeutige Benutzer, die eine Installation gemeldet und ihre
  Telemetriedaten nicht gelöscht haben.

Bei Plugins zählt ClawHub die erste erfolgreiche Installation, die von jedem Benutzer für jedes Paket gemeldet wird.
Wiederholte Installationen und Aktualisierungen aktualisieren die erfasste Version, ohne die aggregierte
Installationszahl zu erhöhen.

## Transparenz und Benutzerkontrollen

Alle sehen nur **aggregierte Installationszähler**.

Wenn Sie Ihr Konto löschen, werden auch Ihre Telemetriedaten gelöscht und ihr Beitrag zu den Installationszählern
entfernt.

## Deaktivieren der Telemetrie

Setzen Sie die Umgebungsvariable:

```bash
export CLAWHUB_DISABLE_TELEMETRY=1
```

Wenn diese Variable gesetzt ist, sendet die CLI keine Installationstelemetrie.
