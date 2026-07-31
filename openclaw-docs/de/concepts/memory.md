---
read_when:
    - Sie möchten verstehen, wie Speicher funktioniert
    - Sie möchten wissen, welche Speicherdateien geschrieben werden sollen
summary: Wie OpenClaw sich sitzungsübergreifend an Dinge erinnert
title: Speicherübersicht
x-i18n:
    generated_at: "2026-07-26T18:24:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cdfd5276d6289a4ee38b5203eb5443312c4b040d4ea67abe4a9c579703136339
    source_path: concepts/memory.md
    workflow: 16
---

OpenClaw merkt sich Dinge, indem es einfache Markdown-Dateien in den
Arbeitsbereich Ihres Agenten schreibt (Standard: `~/.openclaw/workspace`). Das Modell erinnert sich nur an das, was
auf dem Datenträger gespeichert wird; es gibt keinen verborgenen Zustand.

## Funktionsweise

Ihr Agent verfügt über drei speicherbezogene Dateien:

- **`MEMORY.md`** — Langzeitgedächtnis. Dauerhafte Fakten, Präferenzen und
  Entscheidungen. Wird zu Beginn einer Sitzung geladen.
- **`memory/YYYY-MM-DD.md`** (oder `memory/YYYY-MM-DD-<slug>.md`) — tägliche Notizen.
  Laufender Kontext und Beobachtungen. Die datierten Notizen von heute und gestern werden
  automatisch bei einem einfachen `/new` oder `/reset` geladen; Varianten mit Slug, wie sie
  beispielsweise vom mitgelieferten Session-Memory-Hook geschrieben werden, werden zusammen mit der
  Datei erfasst, die nur das Datum enthält.
- **`DREAMS.md`** (optional) — Traumtagebuch und Zusammenfassungen von Dreaming-Durchläufen zur
  menschlichen Überprüfung, einschließlich historisch fundierter Backfill-Einträge.

<Tip>
Wenn sich Ihr Agent etwas merken soll, bitten Sie ihn einfach darum: „Merken Sie sich, dass ich
TypeScript bevorzuge.“ Er schreibt die Notiz in die entsprechende Datei.
</Tip>

## Was wohin gehört

`MEMORY.md` ist die kompakte, kuratierte Ebene: dauerhafte Fakten, Präferenzen, bestehende
Entscheidungen und kurze Zusammenfassungen, die zu Beginn einer
Sitzung verfügbar sein sollten. Sie ist kein Rohtranskript, Tagesprotokoll oder vollständiges Archiv.

`memory/YYYY-MM-DD.md`-Dateien bilden die Arbeitsebene: detaillierte tägliche Notizen,
Beobachtungen, Sitzungszusammenfassungen und Rohkontext, die später noch nützlich sein können.
Diese werden für `memory_search` und `memory_get` indexiert, aber nicht
bei jedem Durchlauf in den Bootstrap-Prompt eingefügt.

Im Laufe der Zeit extrahiert der Agent nützliches Material aus täglichen Notizen und überführt es in
`MEMORY.md`; veraltete Einträge im Langzeitgedächtnis werden entfernt. Generierte Anweisungen für den Arbeitsbereich
und der Heartbeat-Ablauf erledigen dies regelmäßig; Sie müssen
`MEMORY.md` nicht für jedes Detail manuell bearbeiten.

Wenn `MEMORY.md` das Budget für Bootstrap-Dateien überschreitet, lässt OpenClaw die Datei auf dem
Datenträger unverändert, kürzt jedoch die in den Kontext eingefügte Kopie. Verstehen Sie dies als
Signal, detailliertes Material nach `memory/*.md` zu verschieben, nur eine dauerhafte
Zusammenfassung in `MEMORY.md` zu behalten oder die Bootstrap-Grenzwerte zu erhöhen, wenn Sie mehr
Prompt-Budget aufwenden möchten. Verwenden Sie `/context list`, `/context detail` oder `openclaw doctor`, um
die Rohgröße, die eingefügte Größe und den Kürzungsstatus anzuzeigen.

## Import aus Programmierassistenten

Die Control UI kann vorhandene lokale Erinnerungen aus Codex und Claude Code importieren.
Öffnen Sie **Settings** → **Import Memory**, wählen Sie den Zielagenten aus, prüfen Sie die
erkannten Dateien und bestätigen Sie den Import. OpenClaw kopiert ausschließlich Markdown-Erinnerungen:

- Codex: die konsolidierten Dateien `MEMORY.md` und `memory_summary.md` unter
  `~/.codex/memories` (oder `CODEX_HOME/memories`). Rohe Rollout- und Transkriptdateien
  werden nicht importiert.
- Claude Code: Markdown-Dateien aus dem automatischen Speicherverzeichnis jedes Projekts unter
  `~/.claude/projects/*/memory` sowie eine benutzerkonfigurierte
  `autoMemoryDirectory`, sofern vorhanden. Projektanweisungen, Sitzungen, Einstellungen
  und Anmeldedaten sind nicht Bestandteil dieser reinen Speicheraktion.

Importierte Dateien bleiben unter `memory/imports/codex/` und
`memory/imports/claude-code/` im ausgewählten Agentenarbeitsbereich getrennt. Sie werden
für `memory_search` indexiert und sind über `memory_get` verfügbar; sie werden nicht mit der
Bootstrap-Datei `MEMORY.md` des Agenten zusammengeführt. Die Quelldateien bleiben unverändert.

Die Vorschau kennzeichnet Konflikte am Zielort. Aktivieren Sie **Replace existing imports**, um
diese Dateien zu ersetzen; beim Anwenden wird eine verifizierte Sicherung vor dem Import erstellt, und
Kopien überschriebener Dateien auf Elementebene bleiben im Migrationsbericht erhalten.

## Aktionsrelevante Erinnerungen

Die meisten Erinnerungen sind gewöhnliche Markdown-Notizen. Einige beeinflussen, was der Agent
später tun sollte; halten Sie bei diesen fest, wann aufgrund der Notiz sicher gehandelt werden kann, und nicht nur
die Tatsache selbst.

Halten Sie diese Handlungsgrenze fest, wenn eine Notiz Folgendes betrifft:

- Anforderungen an Genehmigungen oder Berechtigungen,
- vorübergehende Einschränkungen,
- Übergaben an eine andere Sitzung, einen anderen Thread oder eine andere Person,
- Ablaufbedingungen,
- den sicheren Handlungszeitpunkt,
- die Autorität der Quelle oder des Verantwortlichen,
- Anweisungen, eine naheliegende Aktion zu vermeiden.

Eine nützliche aktionsrelevante Erinnerung verdeutlicht:

- was das zukünftige Verhalten ändert,
- wann oder unter welcher Bedingung dies gilt,
- wann es abläuft oder wodurch eine Aktion freigegeben wird,
- was der Agent vermeiden sollte,
- wer die Quelle oder der Verantwortliche ist, sofern dies Vertrauen oder Autorität beeinflusst.

Der Speicher kann den Genehmigungskontext bewahren, setzt jedoch keine Richtlinien durch. Verwenden Sie
die Genehmigungseinstellungen, das Sandboxing und geplante Aufgaben von OpenClaw für verbindliche
betriebliche Kontrollen.

Beispiel:

```md
Die API-Migration wird in einer anderen Sitzung konzipiert. Zukünftige Durchläufe sollten
die API-Implementierung nicht aus diesem Thread heraus bearbeiten; verwenden Sie die Erkenntnisse hier nur als
Designgrundlage, bis der Migrationsplan vorliegt.
```

Ein weiteres Beispiel:

```md
Ein Bericht aus einer nicht vertrauenswürdigen Quelle muss vor der Übernahme geprüft werden. Zukünftige Durchläufe
sollten ihn nur als Beleg behandeln; speichern Sie ihn nicht als dauerhafte Erinnerung, bis ein
vertrauenswürdiger Prüfer den Inhalt bestätigt.
```

Dies ist kein vorgeschriebenes Schema für jede Erinnerung; einfache Fakten können knapp bleiben.
Verwenden Sie aktionsrelevante Grenzen, wenn der Verlust von Zeitangaben, Autorität, Ablaufbedingungen oder
Kontext zum sicheren Handeln dazu führen könnte, dass der Agent später falsch handelt.

Verwenden Sie [geplante Aufgaben](/de/automation/cron-jobs) für genaue Erinnerungen, zeitgesteuerte Prüfungen
und wiederkehrende Arbeiten. Der Speicher kann den dauerhaften Kontext dieser
Arbeiten weiterhin zusammenfassen.

## Eingestellte abgeleitete Verpflichtungen

Einige zukünftige Folgeaktionen sind keine dauerhaften Fakten. Wenn Sie ein Vorstellungsgespräch
für morgen erwähnen, könnte die nützliche Erinnerung „nach dem Vorstellungsgespräch nachfragen“ lauten, nicht „dies
für immer in `MEMORY.md` speichern“.

Das Experiment mit abgeleiteten Verpflichtungen wurde eingestellt. OpenClaw extrahiert oder
übermittelt diese Folgeaktionen nicht mehr. Verwenden Sie [geplante Aufgaben](/de/automation/cron-jobs) für
zukünftige Aktionen; der veraltete Befehl `openclaw commitments` bleibt verfügbar, um
vorhandene gespeicherte Zeilen einzusehen oder zu verwerfen.

## Speicherwerkzeuge

Der Agent verfügt über zwei Werkzeuge zur Arbeit mit dem Speicher:

- **`memory_search`** — findet relevante Notizen mithilfe semantischer Suche, selbst wenn
  sich die Formulierung vom Original unterscheidet.
- **`memory_get`** — liest eine bestimmte Speicherdatei oder einen Zeilenbereich.

Beide Werkzeuge werden vom aktiven Speicher-Plugin bereitgestellt (Standard: `memory-core`).

## Speichersuche

Wenn ein Embedding-Provider konfiguriert ist, verwendet `memory_search` eine Hybridsuche:
Vektorähnlichkeit (semantische Bedeutung) kombiniert mit Schlüsselwortabgleich (exakte
Begriffe wie IDs und Codesymbole). Dies funktioniert sofort mit einem API-Schlüssel
für jeden unterstützten Provider.

<Info>
OpenClaw verwendet standardmäßig OpenAI-Embeddings. Legen Sie
`memory.search.provider` ausdrücklich fest, um Gemini, Voyage,
Mistral, Bedrock, DeepInfra, lokales GGUF, Ollama, LM Studio, GitHub Copilot oder
einen generischen OpenAI-kompatiblen Endpunkt zu verwenden.
</Info>

Unter [Speichersuche](/de/concepts/memory-search) finden Sie Informationen zur Funktionsweise der Suche, zu
Optimierungsoptionen und zur Einrichtung von Providern.

## Speicher-Backends

<CardGroup cols={3}>
<Card title="Integriert (Standard)" icon="database" href="/de/concepts/memory-builtin">
SQLite-basiert. Funktioniert sofort mit Schlüsselwortsuche, Vektorähnlichkeit und
Hybridsuche. Keine zusätzlichen Abhängigkeiten.
</Card>
<Card title="QMD" icon="search" href="/de/concepts/memory-qmd">
Lokal ausgerichteter Sidecar mit Neusortierung, Abfrageerweiterung und der Möglichkeit,
Verzeichnisse außerhalb des Arbeitsbereichs zu indexieren.
</Card>
<Card title="Honcho" icon="brain" href="/de/concepts/memory-honcho">
KI-nativer sitzungsübergreifender Speicher mit Benutzermodellierung, semantischer Suche und
Multi-Agenten-Bewusstsein. Plugin-Installation.
</Card>
<Card title="LanceDB" icon="layers" href="/de/plugins/memory-lancedb">
LanceDB-gestützter Speicher mit OpenAI-kompatiblen Embeddings, automatischem Abruf,
automatischer Erfassung und Unterstützung für lokale Ollama-Embeddings. Plugin-Installation.
</Card>
</CardGroup>

## Wissens-Wiki-Ebene

Wenn sich dauerhafter Speicher eher wie eine gepflegte Wissensdatenbank
als wie rohe Notizen verhalten soll, verwenden Sie das mitgelieferte Plugin `memory-wiki`. Es kompiliert dauerhaftes
Wissen in einen Wiki-Tresor mit deterministischer Seitenstruktur, strukturierten
Behauptungen und Belegen, Nachverfolgung von Widersprüchen und Aktualität, generierten
Dashboards, kompilierten Zusammenfassungen und Wiki-nativen Werkzeugen (`wiki_status`,
`wiki_search`, `wiki_get`, `wiki_apply`, `wiki_lint`).

`memory-wiki` ersetzt das aktive Speicher-Plugin nicht; das aktive Speicher-Plugin
bleibt für Abruf, Übernahme und Dreaming zuständig. `memory-wiki` fügt daneben eine
provenienzreiche Wissensebene hinzu.

<CardGroup cols={1}>
<Card title="Speicher-Wiki" icon="book" href="/de/plugins/memory-wiki">
Kompiliert dauerhaften Speicher in einen provenienzreichen Wiki-Tresor mit Behauptungen,
Dashboards, Brückenmodus und Obsidian-freundlichen Arbeitsabläufen.
</Card>
</CardGroup>

## Automatische Speicherleerung

Bevor [Compaction](/de/concepts/compaction) Ihre Unterhaltung zusammenfasst,
führt OpenClaw einen stillen Durchlauf aus, der den Agenten daran erinnert, wichtigen Kontext
in Speicherdateien zu sichern. Dies ist standardmäßig aktiviert; setzen Sie
`agents.defaults.compaction.memoryFlush.enabled: false`, um es zu deaktivieren.

Um diesen Verwaltungsdurchlauf auf einem lokalen Modell auszuführen, legen Sie eine exakte Überschreibung fest, die
nur für den Speicherleerungsdurchlauf gilt (die Modell-Fallback-Kette der aktiven
Sitzung wird nicht übernommen):

```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "memoryFlush": {
          "model": "ollama/qwen3:8b"
        }
      }
    }
  }
}
```

<Tip>
Die Speicherleerung verhindert Kontextverlust während der Compaction. Wenn Ihr Agent
wichtige Fakten aus der Unterhaltung noch nicht in eine Datei geschrieben hat, werden sie
automatisch gespeichert, bevor die Zusammenfassung erfolgt.
</Tip>

## Dreaming

Dreaming ist ein optionaler Konsolidierungsdurchlauf für den Speicher im Hintergrund. Es sammelt
kurzfristige Abrufsignale, bewertet Kandidaten und überführt nur qualifizierte
Elemente in das Langzeitgedächtnis (`MEMORY.md`):

- **Optional aktivierbar**: standardmäßig deaktiviert.
- **Geplant**: Wenn aktiviert, verwaltet `memory-core` automatisch einen wiederkehrenden Cron-
  Job für einen vollständigen Dreaming-Durchlauf.
- **Schwellenwertbasiert**: Übernahmen müssen die Grenzwerte für Bewertung, Abrufhäufigkeit und
  Abfragevielfalt erfüllen.
- **Überprüfbar**: Phasenzusammenfassungen und Tagebucheinträge werden zur
  menschlichen Überprüfung in `DREAMS.md` geschrieben.

Unter [Dreaming](/de/concepts/dreaming) finden Sie Details zum Phasenverhalten, zu Bewertungssignalen und zum
Traumtagebuch.

## Fundierter Backfill und Live-Übernahme

Das Dreaming-System verfügt über zwei zusammenhängende Prüfpfade:

- **Live-Dreaming** arbeitet mit dem kurzfristigen Dreaming-Speicher unter
  `memory/.dreams/` und wird von der normalen tiefen Phase verwendet, um zu entscheiden, was
  in `MEMORY.md` überführt wird.
- **Fundierter Backfill** liest historische `memory/YYYY-MM-DD.md`-Notizen als
  eigenständige Tagesdateien und schreibt strukturierte Prüfergebnisse in `DREAMS.md`.

Fundierter Backfill ist nützlich, um ältere Notizen erneut zu verarbeiten und zu prüfen, was das
System als dauerhaft betrachtet, ohne `MEMORY.md` manuell zu bearbeiten.

```bash
openclaw memory rem-backfill --path ./memory --stage-short-term
```

Das Flag `--stage-short-term` stellt fundierte dauerhafte Kandidaten in demselben
kurzfristigen Dreaming-Speicher bereit, den die normale tiefe Phase bereits verwendet; es
überführt sie nicht direkt. Daher gilt:

- `DREAMS.md` bleibt die Prüffläche für Menschen.
- Der kurzfristige Speicher bleibt die maschinenorientierte Bewertungsfläche.
- `MEMORY.md` wird weiterhin ausschließlich durch die tiefe Übernahme beschrieben.

So machen Sie eine erneute Verarbeitung rückgängig, ohne gewöhnliche Tagebucheinträge oder den normalen Abrufzustand
zu verändern:

```bash
openclaw memory rem-backfill --rollback
openclaw memory rem-backfill --rollback-short-term
```

## CLI

```bash
openclaw memory status          # Indexstatus und Provider prüfen
openclaw memory search "query"  # Über die Befehlszeile suchen
openclaw memory index --force   # Index neu erstellen
```

## Weiterführende Informationen

- [Speichersuche](/de/concepts/memory-search): Suchpipeline, Provider und Optimierung.
- [Integrierte Speicher-Engine](/de/concepts/memory-builtin): standardmäßiges SQLite-Backend.
- [QMD-Speicher-Engine](/de/concepts/memory-qmd): fortschrittlicher Local-First-Sidecar.
- [Honcho-Speicher](/de/concepts/memory-honcho): KI-nativer sitzungsübergreifender Speicher.
- [Memory LanceDB](/de/plugins/memory-lancedb): LanceDB-basiertes Plugin mit OpenAI-kompatiblen Einbettungen.
- [Speicher-Wiki](/de/plugins/memory-wiki): kompilierter Wissensspeicher und Wiki-native Werkzeuge.
- [Dreaming](/de/concepts/dreaming): Überführung von kurzfristigem Abruf in den Langzeitspeicher im Hintergrund.
- [Referenz zur Speicherkonfiguration](/de/reference/memory-config): alle Konfigurationsoptionen.
- [Compaction](/de/concepts/compaction): wie Compaction mit dem Speicher interagiert.
- [Active Memory](/de/concepts/active-memory): Subagentenspeicher für interaktive Chatsitzungen.
