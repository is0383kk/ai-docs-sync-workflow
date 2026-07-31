---
read_when:
    - Sie müssen beantworten können, wer einen Agenten oder ein Tool ausgeführt hat, wann die Ausführung erfolgte und wie sie endete
    - Sie benötigen inhaltsfreie Metadaten zum Lebenszyklus eingehender oder ausgehender Nachrichten
    - Sie benötigen einen begrenzten, schwärzungssicheren Aktivitätsexport
summary: CLI-Referenz für reine Metadaten-Audit-Datensätze zum Lebenszyklus von Ausführungen, Tools und Nachrichten
title: Auditdatensätze
x-i18n:
    generated_at: "2026-07-26T18:16:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: da9df6f388b0a24c3b79d755fa59d047cce99262bc6d9c890be7a83da75693a8
    source_path: cli/audit.md
    workflow: 16
---

# `openclaw audit`

Fragen Sie das reine Metadaten-Audit-Ledger des Gateways nach Agent-Ausführungen, Tool-Aktionen und
optional aktivierten Datensätzen zum Nachrichtenlebenszyklus ab.

Das Ledger ist für Ausführungs- und Tool-Ereignisse standardmäßig aktiviert. Legen Sie
[`audit.enabled: false`](/de/gateway/configuration-reference#audit) fest und starten Sie das
Gateway neu, um alle neuen Ereignisdatensätze zu unterbinden. Nachrichtendatensätze sind separat
standardmäßig deaktiviert; setzen Sie `audit.messages` auf `direct` oder `all` und starten Sie das Gateway neu, um
sie aufzuzeichnen. Vorhandene Datensätze bleiben bis zu ihrem Ablauf (30 Tage) abfragbar.

Das Ledger ist von Konversationstranskripten getrennt: Es zeichnet Identität,
Reihenfolge, Herkunft, Aktion, Status und normalisierte Ergebniscodes auf, speichert jedoch niemals
Inhalte, und Nachrichtenkennungen erscheinen nur als installationslokale,
schlüsselbasierte Pseudonyme. [Audit-Verlauf](/de/gateway/audit) beschreibt das vollständige Datenmodell,
die Datenschutzsemantik, Speicher-/Aufbewahrungsgrenzen und Abdeckungsbeschränkungen; diese Seite
behandelt die Befehlsoberfläche.

```bash
openclaw audit
openclaw audit --agent main --status failed
openclaw audit --session "agent:main:main" --after 2026-07-01T00:00:00Z
openclaw audit --run 8c69f72e-8b11-4c54-98d5-1a3dd67450c3
openclaw audit --kind tool_action --limit 50 --json
openclaw audit --kind message --direction outbound --channel telegram --json
```

## Filter

- `--agent <id>`: exakte Agent-ID
- `--session <key>`: exakter Sitzungsschlüssel
- `--run <id>`: exakte Ausführungs-ID
- `--kind <kind>`: `agent_run`, `tool_action` oder `message`
- `--status <status>`: `started`, `succeeded`, `failed`, `cancelled`,
  `timed_out`, `blocked` oder `unknown`
- `--direction <direction>`: Nachrichtenrichtung, `inbound` oder `outbound`
- `--channel <channel>`: exakter Nachrichtenkanal
- `--after <timestamp>` / `--before <timestamp>`: einschließlich ISO-Zeitstempel oder
  Unix-Millisekunden
- `--limit <count>`: Seitengröße von 1 bis 500; Standardwert `100`
- `--cursor <sequence>`: eine vorherige Abfrage mit neuesten Einträgen zuerst fortsetzen
- `--json`: die begrenzte Seite als JSON ausgeben

Die CLI fragt den versionierten Aktivitäts-RPC ab, sodass ein Befehl das vollständige
konfigurierte Ledger anzeigt. Die Textausgabe zeigt Zeit, Art, Richtung, Kanal, Status,
Agent, Ausführung und Aktion. Fehlende Nachrichtenherkunft wird als `-` dargestellt; OpenClaw
erfindet keine Agent- oder Ausführungs-IDs. Tool-Aktionen zeigen außerdem den Tool-Namen. Die JSON-
Ausgabe enthält `nextCursor`, wenn eine weitere Seite vorhanden ist. Übergeben Sie diesen Wert an
`--cursor`, um fortzufahren, ohne Datensätze neu zu ordnen, die während der Seitennavigation eintreffen.

Diese Exporte bleiben sensible betriebliche Metadaten, obwohl Nachrichtentexte
und unverarbeitete Nachrichtenidentitätsfelder fehlen. Agent-, Sitzungs- und Ausführungs-IDs, Zeitangaben,
Kanäle, Ergebnisse und stabile HMAC-Referenzen können Aktivitäten korrelieren. Schützen Sie
sie mit denselben Zugriffskontrollen und Aufbewahrungspraktiken wie andere
Betreiberdatensätze.

## Aufgezeichnete Ereignisse

Das Gateway projiziert vertrauenswürdige Lebenszyklus-Datenströme auf sechs Aktionen:

- `agent.run.started`
- `agent.run.finished`
- `tool.action.started`
- `tool.action.finished`
- `message.inbound.processed`
- `message.outbound.finished`

Jeder zurückgegebene Datensatz enthält eine stabile Ereignis-ID, eine monoton steigende Ledger-
Sequenz, einen Lebenszyklus-Zeitstempel, Akteur, Aktion, Status, eine
`schemaVersion: 1`-Markierung, Quellsequenz und `redaction: "metadata_only"`.
Die Herkunft von Agent, Sitzung und Ausführung sowie ereignisspezifische Felder sind nur vorhanden, wenn
die vertrauenswürdige Quelle sie bereitstellt. Nachrichtendatensätze lassen absichtlich
`sessionKey` und `sessionId` aus, sodass `--session` nur Ausführungs- und Tool-Datensätze filtert.

Abschließende Ausführungs- und Tool-Datensätze unterscheiden Erfolg, Fehler, Abbruch,
Zeitüberschreitung und Richtlinienblockierungen anhand geschlossener Status- und Fehlercodes. `unknown` ist ein
explizites nicht erfolgreiches Ergebnis, wenn eine vorgelagerte Laufzeitumgebung kein
maßgebliches Endergebnis bereitstellt. Tool-Aufruf-IDs werden nur als stabile
Fingerabdrücke exportiert. Tool-Namen müssen dem kompakten, modellseitigen Namensvertrag
entsprechen; andere Werte werden zu `unknown`.

Nachrichtendatensätze ergänzen Richtung, Kanal, Konversationsart, Ergebnis und
optional Zustellungsart, Fehlerphase, Dauer, Ergebnisanzahl, normalisierten
Ursachencode sowie schlüsselbasierte Konto-/Konversations-/Nachrichten-/Zielpseudonyme. Die
aktuelle Eingangsgrenze umfasst akzeptierte Nachrichten, die die Kerndispatch-Verarbeitung erreichen,
einschließlich Kernduplikaten und abschließenden Verarbeitungsergebnissen. Die Ausgangsgrenze
schreibt eine Abschlusszeile pro ursprünglicher logischer Antwortnutzlast, die die
gemeinsame dauerhafte Zustellung erreicht; Segmentierung und Adapter-Auffächerung werden in
`resultCount` aggregiert. In die Warteschlange gestellte, wiederholbare oder mehrdeutige Sendevorgänge werden erst aufgezeichnet, nachdem eine
Bestätigung, ein Dead Letter oder ein Abgleich das Ergebnis endgültig gemacht hat.
Plugin-lokale und direkte Sendepfade, die diese gemeinsamen Grenzen umgehen, sind
noch nicht abgedeckt; das Fehlen einer Zeile beweist nicht, dass keine Nachricht existiert hat.

Das Audit-Ledger ersetzt weder Transkripte noch Aufgabenverlauf, Cron-Ausführungsverlauf
oder Protokolle. Es stellt einen kleinen ausführungsübergreifenden Index für Betreiberfragen bereit, ohne
Konversationsinhalte in einen weiteren Speicher zu kopieren.

Bei Eingangszeilen misst `durationMs` den Kerndispatch und `resultCount` zählt
finalisierte, in die Warteschlange gestellte Tool-, Blockierungs- und Antwortnutzlasten. Bei Ausgangszeilen
umfasst `durationMs` die Zustellungsverantwortung bis zu ihrem Abschluss (und damit
die Wartezeit in der Warteschlange), während `resultCount` identifizierte physische Sendevorgänge der Plattform
zählt. `deliveryKind` beschreibt, sofern vorhanden, die effektive Nutzlast nach Hook-
und Rendering-Verarbeitung; unterdrückte und durch Abstürze mehrdeutige Zeilen lassen sie aus.

## Gateway-RPC

`audit.activity.list` erfordert `operator.read` und akzeptiert dieselben Filter. Er
gibt die benannte V1-Union von Aktivitätsereignissen zurück, einschließlich Ausführungs-, Tool-, Eingangs-
und Ausgangsnachrichtendatensätzen.

```bash
openclaw gateway call audit.activity.list --params '{"channel":"telegram","limit":50}'
```

Das Ergebnis ist `{ "events": AuditActivityEventV1[], "nextCursor"?: string }`.
Die Ergebnisse werden mit den neuesten zuerst angezeigt und sind auf 500 Datensätze pro Anfrage begrenzt.

Der ausgelieferte RPC `audit.list` bleibt für ältere Ausführungs-/Tool-Clients unverändert. Wenn
`audit.activity.list` auf einem älteren Gateway nicht verfügbar ist, versucht die CLI
`audit.list` nur erneut, wenn jeder angeforderte Filter von dieser Legacy-Methode unterstützt wird. `--kind message`,
`--direction` und `--channel` schlagen auf einem älteren Gateway mit einer Upgrade-Meldung fehl,
anstatt stillschweigend verworfen zu werden.

## Verwandte Themen

- [Audit-Verlauf](/de/gateway/audit)
- [Gateway-Protokoll](/de/gateway/protocol#audit-ledger-rpc)
- [Sitzungen](/de/cli/sessions)
- [Aufgaben](/de/cli/tasks)
- [Cron-Aufgaben](/de/automation/cron-jobs)
