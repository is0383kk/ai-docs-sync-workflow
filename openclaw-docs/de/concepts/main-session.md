---
read_when:
    - Sie möchten verstehen, wo Ihr Agent „lebt“
    - Sie erwarten denselben Kontext, unabhängig davon, ob Sie über Telegram, WhatsApp oder das Web schreiben.
    - Sie möchten, dass Ihr Agent weiß, was in Gruppen und Nebenthreads geschieht
summary: 'Eine fortlaufende Unterhaltung über alle Ihre Kanäle hinweg: die Standardeinstellung für persönliche Agenten'
title: Die Hauptsitzung
x-i18n:
    generated_at: "2026-07-26T18:20:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fb77382ebdce269a05a03ab6fa39b44b1e9f1856166f1d9cb79111dccb547f69
    source_path: concepts/main-session.md
    workflow: 16
---

OpenClaw ist in erster Linie ein persönlicher Agent. Standardmäßig landet jede
Direktnachricht, die Sie ihm senden — über Telegram, WhatsApp, iMessage, Slack-DMs,
die Web-App oder von überall sonst — in **einer fortlaufenden Unterhaltung**:
der Hauptsitzung. Fragen Sie etwas auf Ihrem Smartphone und haken Sie auf Ihrem
Laptop nach; der Agent verfügt an beiden Orten über denselben Kontext. Es gibt
ein Gehirn, und hier denkt es.

Unter der Haube ist die Hauptsitzung eine gewöhnliche Sitzung mit dem Schlüssel
`agent:<agentId>:main` (zum Beispiel `agent:main:main`). Besonders wird sie
dadurch, dass der standardmäßige DM-Geltungsbereich alle Direktnachrichten darin
zusammenführt und der Rest des Systems sie als Wurzel des Agenten behandelt:
Heartbeats aktivieren sie, Hintergrundarbeiten melden ihre Ergebnisse an sie
zurück und Aktivitäten an anderen Stellen fließen zu ihr hinauf.

## Startseite

In der Web-App ist die Hauptsitzung die Seite **Startseite** — der erste Eintrag
in der Seitenleiste. Die Identitätszeile oben stellt Ihren Agenten dar (klicken
Sie darauf, um das Agentenmenü zu öffnen); auf der Startseite sprechen Sie mit
ihm. Sitzungen, die von der Hauptunterhaltung abzweigen, erscheinen unter
**Threads**, Gruppenchats unter **Gruppen** und Coding-/CLI-Sitzungen unter
**Coding**.

## Was in die Hauptsitzung einfließt

Die Hauptsitzung ist nicht nur ein Chatprotokoll; sie ist der Ort, an dem die
Welt Ihres Agenten zusammenläuft:

- **Gruppenaktivität.** Gruppen- und Raumsitzungen bleiben isoliert (siehe unten),
  aber im standardmäßigen DM-Geltungsbereich überwacht die Hauptsitzung sie
  automatisch. Aktivitäten werden als kompakte Hinweise in eine Warteschlange
  gestellt — pro Unterhaltung zusammengeführt, niemals eine Aktivierung pro
  Nachricht — und der Agent sieht sie bei seiner nächsten Ausführung: bei
  Ihrer nächsten Nachricht oder bei einem geplanten Heartbeat. Der Agent kann
  auch die von ihm überwachten Sitzungen lesen, sodass „Was habe ich in der
  Familiengruppe verpasst?“ funktioniert.
- **Hintergrundarbeiten.** Unteragenten und erzeugte Sitzungen melden ihre
  Ergebnisse an die Sitzung zurück, die sie gestartet hat. Arbeiten, die der
  Agent von der Startseite aus angestoßen hat, werden daher an die Startseite
  zurückgemeldet.
- **Heartbeats.** Geplante Heartbeats richten sich an die Hauptsitzung. Dadurch
  werden Hinweise in der Warteschlange wahrgenommen, auch wenn Sie nichts
  geschrieben haben.

## Speicher über Zurücksetzungen und Unterhaltungen hinweg

Die fortlaufende Unterhaltung ist durch das Kontextfenster des Modells begrenzt,
daher entsteht Kontinuität durch die sie umgebenden Ebenen:

- `MEMORY.md`, der kuratierte Langzeitspeicher des Agenten, wird in jede
  neue Sitzung geladen. Tagesnotizen (`memory/YYYY-MM-DD.md`) können bei Bedarf
  durchsucht werden, und die neuesten werden nach einem `/new` oder
  `/reset` erneut als Ausgangskontext geladen. Vor der Compaction
  schreibt der Agent dauerhaft relevante Fakten in die Tagesnotizen, damit
  sie bei langen Unterhaltungen nicht unbemerkt verloren gehen.
- **Speicherabruf über Unterhaltungen hinweg** ermöglicht dem Agenten, Inhalte
  aus seinen anderen privaten Sitzungen abzurufen. In persönlichen
  Konfigurationen — wenn das globale `session.dmScope` zu
  `main` aufgelöst wird und keine DM-Überschreibungen pro Bindung
  vorhanden sind — ist dies standardmäßig aktiviert; jede konfigurierte
  DM-Isolierung deaktiviert es, sofern Sie es nicht ausdrücklich aktivieren.
  Siehe [Speicherkonfiguration](/de/reference/memory-config).

## Eine fortlaufende Sitzung mit dauerhaftem Verlauf

Die Hauptsitzung wird über Zurücksetzungen und Compaction hinweg fortgeführt,
statt das Modell ihren gesamten Verlauf auf einmal tragen zu lassen:

- Standardmäßig erfolgt keine automatische Zurücksetzung; Compaction hält den
  aktiven Kontext begrenzt und bewahrt zugleich die fortlaufende Sitzung.
  Tägliche und inaktivitätsbedingte Zurücksetzungen müssen ausdrücklich
  aktiviert werden (siehe [Sitzungsverwaltung](/de/concepts/session)). Bei
  `/new` und `/reset` wird das Ende der auslaufenden
  Unterhaltung in den täglichen Speichernotizen gesichert, und die nächste
  Sitzung lädt die neuesten Notizen erneut als Ausgangskontext. Eine
  Zurücksetzung weist eine neue aktive Sitzungs-ID zu, lässt aber das vorherige
  SQLite-Transkript weiterhin unter demselben Hauptsitzungsschlüssel
  durchsuchbar.
- Wenn sich die Unterhaltung der Grenze des Kontextfensters nähert, fasst
  Compaction sie zusammen und setzt sie an derselben Stelle fort — der
  Transkriptverlauf verbleibt im Sitzungsspeicher.
- Sitzungslisten zeigen die aktuelle aktive Unterhaltung und nicht jede
  historische Sitzungs-ID, die dahintersteht.
- Wenn die physische Datenbank, das WAL und die Sitzungsartefakte des
  agentenspezifischen Speichers das Speicherplatzbudget überschreiten
  (standardmäßig 10 GB), extrahiert OpenClaw den ältesten nicht referenzierten
  Verlauf in ein verifiziertes komprimiertes Archiv, bevor die zugehörigen
  Datenbankzeilen entfernt werden. Aktive, weitergeleitete und laufende
  Sitzungen fallen niemals dem Budget zum Opfer.

## Wenn Sie stattdessen Isolierung wünschen

Die gemeinsam genutzte Hauptsitzung ist die richtige Standardeinstellung für
einen Agenten, mit dem nur Sie kommunizieren. Wenn mehrere Personen Ihrem
Agenten Nachrichten senden können, isolieren Sie die DMs:

```json5
{
  session: {
    dmScope: "per-channel-peer",
  },
}
```

Bei einem isolierenden Geltungsbereich erhält jeder Absender eine eigene
Sitzung, die Gruppenüberwachung durch die Hauptsitzung ist deaktiviert und der
sitzungsübergreifende Speicherabruf ist standardmäßig ausgeschaltet.
`openclaw security audit` empfiehlt Isolierung, wenn mehrere DM-Absender erkannt
werden. Die vollständige Geltungsbereichsmatrix, die Identitätsverknüpfung und
routenspezifische Überschreibungen werden unter
[Sitzungsverwaltung](/de/concepts/session) und
[Channel-Routing](/de/channels/channel-routing) behandelt.

## Verwandte Themen

- [Sitzungsverwaltung](/de/concepts/session) — Routing, Geltungsbereiche, Zurücksetzungen
- [Channel-Routing](/de/channels/channel-routing) — Auswahl von Agenten und Sitzungen
- [Speicher](/de/concepts/memory) — dauerhafte Speicherebenen
- [Multi-Agent](/de/concepts/multi-agent) — mehrere isolierte Agenten ausführen
