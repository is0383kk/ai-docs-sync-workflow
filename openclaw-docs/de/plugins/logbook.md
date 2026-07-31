---
read_when:
    - Sie möchten eine Zeitleiste Ihres Tages im Dayflow-Stil in der Control UI.
    - Sie aktivieren oder konfigurieren das gebündelte Logbook-Plugin
    - Sie möchten Stand-up-Zusammenfassungen oder Tagesrückblicke auf Grundlage der Bildschirmaktivität.
summary: Optionales automatisches Arbeitsjournal, das aus regelmäßigen Bildschirmaufnahmen erstellt wird
title: Logbuch-Plugin
x-i18n:
    generated_at: "2026-07-26T18:36:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 19197e580421dfe81f82f8599578e4c68a15004813bb2b6c3de761c14f426b08
    source_path: plugins/logbook.md
    workflow: 16
---

Das Logbook-Plugin verwandelt Bildschirmaktivitäten in ein automatisches Arbeitstagebuch. Es
erstellt regelmäßig Bildschirmaufnahmen von einem gekoppelten Node, fasst sie zu
mit Zeitstempeln versehenen Beobachtungen zusammen und erstellt Zeitleistenkarten in der
[Control UI](/de/web/control-ui). Es kann außerdem tägliche Standup-Notizen erstellen und
Fragen zu einem erfassten Tag beantworten.

Der von OpenClaw verwaltete Zustand verbleibt auf dem Gateway unter `<state-dir>/logbook/`, die
Modellverarbeitung erfolgt jedoch nicht zwangsläufig lokal. Ausgewählte Screenshots werden an die
konfigurierte Vision-Route gesendet; Beobachtungen und Zeitleistentext werden an das standardmäßige
Agentenmodell gesendet. Verwenden Sie für beide Phasen lokale Modellrouten, wenn Bildschirminhalte und
daraus abgeleitete Aktivitätstexte auf dem Rechner verbleiben müssen.

Logbook ist im Lieferumfang enthalten und standardmäßig deaktiviert. Durch Aktivieren des Plugins wird die
Bildschirmerfassung für das Gateway aktiviert, da `captureEnabled` standardmäßig `true` ist.

## Bevor Sie beginnen

Sie benötigen:

- Einen verbundenen Node, der `screen.snapshot` oder `logbook.snapshot` bereitstellt. Der
  Node der macOS-App benötigt die Berechtigung zur Bildschirmaufnahme. Ein headless betriebener macOS-Node-Host
  (`openclaw node host run`) erhält den vom Plugin bereitgestellten Befehl `logbook.snapshot`,
  der auf dem Systemwerkzeug `screencapture` basiert.
- Das gebündelte Codex-Plugin muss aktiviert und authentifiziert sein. Codex stellt derzeit
  den strukturierten Bildextraktionsvertrag bereit, den Logbook benötigt. Melden Sie sich mit
  `openclaw models auth login --provider openai` an; weitere Authentifizierungswege finden Sie unter
  [Codex-Harness](/de/plugins/codex-harness).
- Ein funktionsfähiges standardmäßiges Agentenmodell. Logbook verwendet es nach der Vision-Verarbeitung, um Karten, Standup-
  Notizen und Fragen und Antworten zum Tag zu synthetisieren.

## Schnellstart

Aktivieren Sie die Plugins Codex und Logbook:

```bash
openclaw plugins enable codex
openclaw plugins enable logbook
```

Konfigurieren Sie für einen deterministischen Start ein explizites Vision-Modell:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
      logbook: {
        enabled: true,
        config: {
          visionModel: "codex/gpt-5.6-sol",
        },
      },
    },
  },
}
```

Wenn Sie `plugins.allow` verwenden, schließen Sie sowohl `codex` als auch `logbook` ein. Starten Sie das
Gateway nach einer Änderung der Plugin-Konfiguration neu, prüfen Sie anschließend die Registrierungen
und öffnen Sie das Dashboard:

```bash
openclaw gateway restart
openclaw plugins inspect logbook --runtime --json
openclaw nodes status --connected
openclaw nodes describe --node <idOrNameOrIp>
openclaw dashboard
```

Die Node-Beschreibung muss `screen.snapshot` oder `logbook.snapshot` enthalten.
Headless-Nodes geben `logbook.snapshot` erst an, nachdem das Plugin aktiviert wurde.
Wenn der Befehl fehlt, lesen Sie die [Node-Fehlerbehebung](/de/nodes/troubleshooting).

Die Registerkarte Logbook wird nur bei aktiviertem Plugin und einer `operator.write`-
Control-UI-Sitzung angezeigt. Die Statuszeile sollte **Erfassung läuft** ohne Fehler anzeigen.
Eine Zeitleistenkarte erscheint, wenn sich das Analysefenster schließt. Alternativ können Sie
**Jetzt analysieren** auswählen, nachdem Aktivität erfasst wurde.

## Funktionsweise

1. **Erfassen**: Alle `captureIntervalSeconds` (standardmäßig 30s) ruft Logbook
   den Erfassungsbefehl des ausgewählten Nodes auf und speichert ein skaliertes JPEG-Bild.
   Aufeinanderfolgende identische Bilder werden als inaktiv markiert und von der Analyse ausgeschlossen.
2. **Beobachten**: Sobald ein Analysefenster (standardmäßig 15 Minuten) abgelaufen ist, wählt das
   Plugin bis zu 16 aktive Bilder aus und sendet sie an das Vision-Modell,
   das mit Zeitstempeln versehene Aktivitätsbeobachtungen zurückgibt („VS Code: Bearbeiten von
   store.ts, Beheben eines Typfehlers“). Eine Erfassungslücke von mehr als zwei Minuten oder
   die lokale Mitternacht schließt ebenfalls das aktuelle Fenster.
3. **Synthetisieren**: Beobachtungen sowie die vorhandenen Karten der letzten 45 Minuten werden
   zu Zeitleistenkarten (jeweils 10-60 Minuten) mit Titel, Zusammenfassung,
   Kategorie, Haupt-App und etwaigen kurzen Ablenkungen überarbeitet.
4. **Bereinigen**: Bilder, die älter als `retentionDays` (standardmäßig 14) sind, werden gelöscht.
   Karten, Beobachtungen und zwischengespeicherte Standups bleiben erhalten.

Tagesgrenzen und Zeitleistenuhren verwenden die lokale Zeitzone des Gateways, nicht die
Zeitzone des Browsers. Bilder und die SQLite-Zeitleistendatenbank befinden sich unter
`<state-dir>/logbook/`.

## Modell- und Datenfluss

Logbook verwendet zwei separate Modellrouten:

| Phase             | Gesendete Daten                                           | Modellroute                                                        |
| ---------------- | --------------------------------------------------------- | ----------------------------------------------------------------- |
| Beobachten       | Bis zu 16 ausgewählte JPEG-Bilder sowie deren Erfassungszeiten | `visionModel` oder ein kompatibler übernommener `tools.media`-Codex-Eintrag |
| Karten synthetisieren | Beobachtungen mit Zeitstempeln und aktuelle Zeitleistenkarten | Standardmäßiges Agentenmodell über die Plugin-LLM-Laufzeit |
| Standup erstellen | Karten für den ausgewählten und den vorherigen Tag       | Standardmäßiges Agentenmodell über die Plugin-LLM-Laufzeit |
| Fragen zum Tag stellen | Die Frage, Karten des ausgewählten Tages und aktuelle Beobachtungen | Standardmäßiges Agentenmodell über die Plugin-LLM-Laufzeit |

Die vollständige SQLite-Datenbank wird an keines der Modelle gesendet. Unverarbeitete Screenshots werden nur
an die Beobachtungsphase gesendet; Kartensynthese, Standup sowie Fragen und Antworten erhalten abgeleiteten
Text.

## Konfiguration

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
      logbook: {
        enabled: true,
        config: {
          captureEnabled: true,
          captureIntervalSeconds: 30,
          analysisIntervalMinutes: 15,
          nodeId: "my-mac",
          screenIndex: 0,
          maxWidth: 1440,
          visionModel: "codex/gpt-5.6-sol",
          retentionDays: 14,
        },
      },
    },
  },
}
```

Alle Logbook-Konfigurationsschlüssel sind optional. Numerische Werte werden auf ganze Zahlen gerundet
und auf den unterstützten Bereich begrenzt.

| Schlüssel                 | Standardwert | Bereich oder Werte       | Verhalten                                                                                    |
| ------------------------- | ------- | ----------------------- | -------------------------------------------------------------------------------------------- |
| `captureEnabled`          | `true`  | boolesch                 | Dauerhafter Hauptschalter für neue Momentaufnahmen; die Zeitleiste bleibt verfügbar, wenn `false`      |
| `captureIntervalSeconds`  | `30`    | `5`-`600`               | Verzögerung zwischen Erfassungsversuchen                                                               |
| `analysisIntervalMinutes` | `15`    | `3`-`120`               | Vorgesehenes Beobachtungsfenster; Lücken und Mitternacht können es früher schließen                            |
| `nodeId`                  | nicht festgelegt   | Node-ID oder Anzeigename | Bindet die Erfassung an einen verbundenen Node; beim Abgleich wird die Groß-/Kleinschreibung nicht berücksichtigt                             |
| `screenIndex`             | `0`     | `0`-`16`                | Nullbasierter Anzeigeindex                                                                     |
| `maxWidth`                | `1440`  | `480`-`3840`            | Angeforderte Obergrenze der Erfassungsgröße; headless betriebenes macOS wendet sie auf die größte Abmessung an               |
| `visionModel`             | nicht festgelegt   | `provider/model`        | Explizite strukturierte Route; fehlerhafte Referenzen pausieren die Analyse, nicht unterstützte Provider lassen Batches fehlschlagen |
| `retentionDays`           | `14`    | `1`-`365`               | Löscht alte Bilder; Karten, Beobachtungen und Standups bleiben erhalten                                 |

Ohne `nodeId` bevorzugt Logbook einen verbundenen App-Node, der
`screen.snapshot` bereitstellt, und greift anschließend auf einen headless betriebenen Node zurück, der
`logbook.snapshot` bereitstellt. In einer nicht fest gebundenen Einrichtung wird ein fehlgeschlagener Node hinter andere
geeignete Nodes verschoben. Der Pause-Schalter im Dashboard gilt nur für die Sitzung und wird beim Neustart des
Gateways zurückgesetzt; verwenden Sie `captureEnabled: false` für einen dauerhaften Stopp.

### Auswahl des Vision-Modells

Logbook löst das Beobachtungsmodell in dieser Reihenfolge auf:

1. `plugins.entries.logbook.config.visionModel`
2. den ersten bildfähigen Codex-Eintrag unter `tools.media.models`

Andere Medien-Provider werden übersprungen, da sie derzeit nicht den
strukturierten Extraktionsvertrag bereitstellen, den Logbook benötigt. Das Festlegen von
`tools.media.image.enabled: false` deaktiviert übernommene Medienstandardwerte, ein
explizites Logbook-`visionModel` gilt jedoch weiterhin.

## Dashboard-Registerkarte

- **Zeitleiste**: Erweiterbare Karten pro Aktivität mit Kategoriefarben, der Haupt-
  App, Ablenkungsmarkierungen und einem Schlüsselbild.
- **Tagesüberblick**: Fokusanteil, Kategorieaufschlüsselung, meistgenutzte Apps.
- **Tägliches Standup**: Wandelt gestern und heute in eine direkt einfügbare Aktualisierung um.
- **Fragen Sie nach Ihrem Tag**: Fragen in natürlicher Sprache, die anhand der erfassten
  Zeitleiste beantwortet werden („Wann habe ich den Gateway-PR geprüft?“).
- **Jetzt analysieren**: Schließt das aktuelle Erfassungsfenster sofort, statt
  auf das Analyseintervall zu warten.

## Gateway-Methoden

Logbook registriert die folgenden Gateway-RPC-Methoden:

| Methode               | Parameter                | Umfang           | Ergebnis                                                                 |
| --------------------- | ------------------------ | ---------------- | ------------------------------------------------------------------------ |
| `logbook.status`      | keine                    | `operator.read`  | Status von Erfassung, Analyse, Modell, Node, Gateway-Tag und Gateway-Zeitzone |
| `logbook.days`        | keine                    | `operator.read`  | Tage mit Anzahlen von Zeitleistenkarten und zeitlichen Kartengrenzen     |
| `logbook.timeline`    | `{ day?: "YYYY-MM-DD" }` | `operator.read`  | Abgeleitete Karten und Tagesstatistiken; standardmäßig der aktuelle Tag des Gateways |
| `logbook.frames`      | `{ startMs, endMs }`     | `operator.write` | Bildmetadaten im angeforderten Bereich in Epochen-Millisekunden          |
| `logbook.frame`       | `{ frameId }`            | `operator.write` | Ein unverarbeitetes JPEG-Bild als base64                                 |
| `logbook.standup`     | `{ day?, refresh? }`     | `operator.write` | Zwischengespeicherter oder neu generierter Standup-Text für einen Tag    |
| `logbook.ask`         | `{ day?, question }`     | `operator.write` | Auf der Zeitleiste basierende Antwort für einen Tag                      |
| `logbook.capture.set` | `{ paused }`             | `operator.write` | Nur für die Sitzung geltender Pausenzustand und aktualisierter Status    |
| `logbook.analyze.now` | keine                    | `operator.write` | Startet die ausstehende Analyse oder gibt einen Grund zurück, warum sie nicht gestartet werden konnte |

Die Lesemethoden geben den Betriebszustand oder abgeleiteten Text zurück. Unverarbeitete Screenshot-
Pixel, Aktionen mit Modellkosten und Laufzeitänderungen erfordern
`operator.write`. Die Registerkarte der Control UI erfordert ebenfalls `operator.write`, da sie
diese Aktionen und Vorschauen unverarbeiteter Bilder bereitstellt; ein schreibgeschützter Client kann die
Methoden für abgeleiteten Text weiterhin direkt aufrufen.

## Datenschutzhinweise

- Snapshots können alles enthalten, was auf dem Bildschirm angezeigt wird, einschließlich Geheimnissen. Frames verlassen das Gerät niemals, außer als ausgewählte Eingabe für das konfigurierte Beobachtungsmodell.
- Beobachtungen, aktuelle Karten und Fragen können das Gerät während der Kartensynthese, der Standup-Erstellung oder bei Fragen und Antworten über das standardmäßige Agentenmodell verlassen. Wenden Sie die Datenverarbeitungsrichtlinie des Providers auf beide Modellrouten an.
- Verwenden Sie lokale Routen sowohl für das strukturierte Beobachtungsmodell als auch für das standardmäßige Agentenmodell, wenn Sie eine vollständig lokale Pipeline benötigen.
- Frames, die Zeitleistendatenbank und temporäre Aufnahmen werden mit Dateiberechtigungen gespeichert, die ausschließlich dem Eigentümer Zugriff gewähren.
- Das Hinzufügen von `screen.snapshot` zu `gateway.nodes.commands.deny` ist der Notausschalter für Bildschirmaufnahmen: Es blockiert sowohl die Aufnahme durch App-Nodes als auch Logbooks eigenen Befehl `logbook.snapshot`.
- Das Festlegen von `tools.media.image.enabled: false` verhindert außerdem, dass Logbook die Medienbildmodelle zur Analyse verwendet; dann wird nur ein explizit in der Plugin-Konfiguration festgelegtes `visionModel` verwendet.

## Fehlerbehebung

### Der Logbook-Tab fehlt

Überprüfen Sie alle drei Voraussetzungen:

1. `openclaw plugins list --enabled` enthält `logbook`.
2. Der Gateway wurde nach der Änderung des Plugins oder der Positivliste neu gestartet.
3. Die Verbindung zur Control UI verfügt über `operator.write`; schreibgeschützte Sitzungen erhalten die interaktive Tab-Beschreibung nicht.

Wenn `plugins.allow` festgelegt ist, muss es für die empfohlene Konfiguration sowohl `logbook` als auch `codex` enthalten.

### Die Aufnahme meldet einen Fehler

```bash
openclaw nodes status --connected
openclaw nodes describe --node <idOrNameOrIp>
openclaw logs --follow
```

- Stellen Sie sicher, dass der Node `screen.snapshot` oder `logbook.snapshot` bereitstellt.
- Erteilen Sie auf dem aufnehmenden Mac die Berechtigung für Bildschirmaufnahmen.
- Wenn `nodeId` konfiguriert ist, stellen Sie sicher, dass es mit der Node-ID oder dem Anzeigenamen übereinstimmt.
- Überprüfen Sie, dass `gateway.nodes.commands.deny` nicht `screen.snapshot` enthält.

Nach drei aufeinanderfolgenden Fehlern setzt Logbook die Aufnahme für zehn Aufnahmezyklen aus und versucht es anschließend erneut. Eine nicht fest zugeordnete Einrichtung kann zu einem anderen geeigneten Node wechseln.

### Aufnahmen sind erfolgreich, aber es werden keine Karten angezeigt

- Der Status **Modell fehlt** bedeutet, dass keine kompatible strukturierte Bildverarbeitungsroute gefunden wurde. Aktivieren und authentifizieren Sie das Codex-Plugin oder legen Sie ein gültiges explizites `visionModel` fest. Aufgenommene Frames bleiben ausstehend, solange das Modell fehlt, und können analysiert werden, nachdem die Konfiguration korrigiert wurde.
- Warten Sie auf `analysisIntervalMinutes` oder wählen Sie **Jetzt analysieren**, nachdem Aktivität aufgezeichnet wurde.
- Aufeinanderfolgende identische Frames gelten als Inaktivitätsnachweis und werden nicht in Analysebatches aufgenommen. Ändern Sie vor dem Testen den sichtbaren Bildschirminhalt.
- Wenn der neueste Batch einen Fehler anzeigt, beheben Sie das Modell- oder Authentifizierungsproblem und wählen Sie **Jetzt analysieren**. Fehlgeschlagene Batches werden nur nach dieser ausdrücklichen Aktion erneut versucht, um wiederholte Modellkosten zu vermeiden.

## Verwandte Themen

- [Plugins verwalten](/de/plugins/manage-plugins)
- [Codex-Harness](/de/plugins/codex-harness)
- [Medienverständnis](/de/nodes/media-understanding)
- [Nodes](/de/nodes)
- [Node-Fehlerbehebung](/de/nodes/troubleshooting)
- [Control UI](/de/web/control-ui)
