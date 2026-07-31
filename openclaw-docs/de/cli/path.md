---
read_when:
    - Sie möchten über das Terminal ein Blatt innerhalb einer Workspace-Datei lesen oder schreiben
    - Sie schreiben Skripte für den Workspace-Status und benötigen ein stabiles, typunabhängiges Adressierungsschema.
    - Sie debuggen einen `oc://`-Pfad (prüfen Sie die Syntax und sehen Sie nach, wohin er aufgelöst wird).
summary: CLI-Referenz für `openclaw path` (Arbeitsbereichsdateien über das Adressierungsschema `oc://` prüfen und bearbeiten)
title: Pfad
x-i18n:
    generated_at: "2026-07-26T18:22:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7afe5bd1c3a5fca8dd22c7d807e390e751ae7e895c54bf0e10e2734f3889436c
    source_path: cli/path.md
    workflow: 16
---

# `openclaw path`

Shell-Zugriff auf das Adressierungsschema `oc://`: eine nach Art dispatchte Pfadsyntax
zum Prüfen und Bearbeiten adressierbarer Workspace-Dateien (Markdown, JSONC,
JSONL, YAML/YML/Lobster). Self-Hoster, Plugin-Autoren und Editor-Erweiterungen
verwenden sie, um einen eng begrenzten Speicherort zu lesen, zu finden oder zu aktualisieren, ohne
für jeden Dateityp einen eigenen Parser zu erstellen.

`path` wird vom gebündelten optionalen Plugin `oc-path` bereitgestellt. Aktivieren Sie es vor
der ersten Verwendung:

```bash
openclaw plugins enable oc-path
```

Die CLI-Verben entsprechen dem Adressierungsmodell:

- `resolve` ist konkret und liefert genau einen Treffer.
- `find` ist das Verb für mehrere Treffer bei Platzhaltern, Vereinigungen, Prädikaten und
  positionaler Erweiterung.
- `set` akzeptiert nur konkrete Pfade oder Einfügemarkierungen; Platzhaltermuster
  werden vor dem Schreiben abgelehnt.
- `validate` parst einen Pfad ohne Dateisystemzugriff.
- `emit` durchläuft für eine Datei Parsen und Ausgeben vollständig (Diagnose der Byte-Treue).

## Gründe für die Verwendung

Der OpenClaw-Zustand ist über manuell bearbeitetes Markdown, kommentierte JSONC-
Konfigurationen, nur anhängbare JSONL-Protokolle und YAML-Workflow-/Spezifikationsdateien verteilt. Skripte, Hooks
und Agenten benötigen aus diesen Dateien häufig nur einen kleinen Wert: einen Frontmatter-Schlüssel, eine
Plugin-Einstellung, ein Feld eines Protokolleintrags, einen YAML-Schritt oder einen Aufzählungspunkt unter einem
benannten Abschnitt.

`openclaw path` stellt diesen Aufrufern eine stabile Adresse bereit, statt für jeden Dateityp
eine einmalige grep-Suche, einen regulären Ausdruck oder einen Parser zu verwenden. Derselbe Pfad `oc://` kann im
Terminal validiert, aufgelöst, durchsucht, als Probelauf ausgeführt und geschrieben werden, wodurch eng begrenzte
Automatisierungen überprüfbar und wiederholbar bleiben. Der Rest der Datei bleibt erhalten, sodass
das Schreiben eines einzelnen Blatts dessen Kommentare, Zeilenenden oder benachbarte
Formatierung nicht verändert.

Verwenden Sie es, wenn das gewünschte Element eine logische Adresse besitzt, die Dateistruktur
jedoch variiert:

- Ein Hook liest eine Einstellung aus kommentiertem JSONC, ohne Kommentare zu verlieren, wenn
  er den Wert zurückschreibt.
- Ein Wartungsskript findet jedes übereinstimmende Ereignisfeld in einem JSONL-Protokoll,
  ohne das gesamte Protokoll in einen eigenen Parser zu laden.
- Ein Editor springt anhand des Slugs zu einem Markdown-Abschnitt oder Aufzählungspunkt und rendert anschließend
  genau die aufgelöste Zeile.
- Ein Agent führt vor der Anwendung einer kleinen Workspace-Bearbeitung einen Probelauf aus, wobei die
  geänderten Bytes bei der Überprüfung sichtbar sind.

Verwenden Sie `openclaw path` nicht für gewöhnliche Bearbeitungen vollständiger Dateien, umfangreiche Konfigurationsmigrationen oder
speicherspezifische Schreibvorgänge; dafür sollte der zuständige Befehl oder das zuständige Plugin verwendet werden. `path`
ist für kleine, adressierbare Dateioperationen vorgesehen, bei denen ein wiederholbarer Terminalbefehl
einem weiteren maßgeschneiderten Parser überlegen ist.

## Verwendung

Einen Wert aus einer manuell bearbeiteten Konfigurationsdatei lesen:

```bash
openclaw path resolve 'oc://config.jsonc/plugins/github/enabled'
```

Einen Schreibvorgang anzeigen, ohne die Festplatte zu verändern:

```bash
openclaw path set 'oc://config.jsonc/plugins/github/enabled' 'true' --dry-run
```

Übereinstimmende Einträge in einem nur anhängbaren JSONL-Protokoll finden:

```bash
openclaw path find 'oc://session.jsonl/[event=tool_call]/name'
```

Eine Anweisung in Markdown anhand von Abschnitt und Element statt anhand der Zeilen-
nummer adressieren:

```bash
openclaw path resolve 'oc://AGENTS.md/runtime-safety/openclaw-gateway'
```

Einen Pfad in der CI oder einem Vorprüfungsskript validieren, bevor das Skript liest oder
schreibt:

```bash
openclaw path validate 'oc://AGENTS.md/tools/$last/risk'
```

Diese Befehle sind zum Kopieren in Shell-Skripte vorgesehen. Verwenden Sie `--json`, wenn
ein Aufrufer strukturierte Ausgaben benötigt, und `--human`, wenn eine Person das Ergebnis
prüft.

## Funktionsweise

1. Parst die Adresse `oc://` in Slots: Datei, Abschnitt, Element, Feld und eine
   optionale Sitzungsabfrage.
2. Wählt den Dateitypadapter anhand der Erweiterung des Ziels aus (`.md`, `.jsonc`,
   `.json`, `.jsonl`, `.ndjson`, `.yaml`, `.yml`, `.lobster`).
3. Löst die Slots anhand der Struktur dieses Dateityps auf: Markdown-
   Überschriften/-Elemente, JSONC-Objektschlüssel/-Arrayindizes, JSONL-Zeileneinträge oder
   YAML-Zuordnungs-/Sequenzknoten.
4. Gibt für `set` bearbeitete Bytes über denselben Adapter aus, sodass unveränderte Teile
   der Datei ihre Kommentare, Zeilenenden und benachbarte Formatierung beibehalten, sofern
   der Dateityp dies unterstützt.

`resolve` und `set` erfordern ein einzelnes konkretes Ziel. `find` ist das explorative
Verb: Es erweitert Platzhalter, Vereinigungen, Prädikate und Ordnungsangaben zu den konkreten
Treffern, die Sie prüfen können, bevor Sie einen zum Schreiben auswählen.

## Unterbefehle

| Unterbefehl              | Zweck                                                                     |
| ----------------------- | --------------------------------------------------------------------------- |
| `resolve <oc-path>`     | Den konkreten Treffer am Pfad ausgeben (oder „nicht gefunden“).                      |
| `find <pattern>`        | Treffer für einen Platzhalter-/Vereinigungs-/Prädikatpfad aufzählen.                  |
| `set <oc-path> <value>` | Ein Blatt oder Einfügeziel an einem konkreten Pfad schreiben. Unterstützt `--dry-run`.  |
| `validate <oc-path>`    | Nur parsen; die strukturelle Aufgliederung ausgeben (Datei/Abschnitt/Element/Feld). |
| `emit <file>`           | Eine Datei durch Parsen und Ausgeben vollständig durchlaufen (Diagnose der Byte-Treue).          |

## Globale Flags

| Flag            | Gilt für                       | Zweck                                                                  |
| --------------- | -------------------------------- | ------------------------------------------------------------------------ |
| `--cwd <dir>`   | `resolve`, `find`, `set`, `emit` | Den Datei-Slot relativ zu diesem Verzeichnis auflösen (Standard: `process.cwd()`). |
| `--file <path>` | `resolve`, `find`, `set`, `emit` | Den aufgelösten Pfad des Datei-Slots überschreiben (absoluter Zugriff).                |
| `--json`        | alle                              | JSON-Ausgabe erzwingen (Standard, wenn stdout kein TTY ist).                    |
| `--human`       | alle                              | Für Menschen lesbare Ausgabe erzwingen (Standard, wenn stdout ein TTY ist).                       |
| `--value-json`  | `set`                            | `<value>` für die Blattersetzung in JSON/JSONC/JSONL als JSON parsen.           |
| `--dry-run`     | `set`                            | Die Bytes ausgeben, die geschrieben würden, ohne sie zu schreiben.                   |
| `--diff`        | `set` (erfordert `--dry-run`)     | Statt der vollständigen Bytes einen vereinheitlichten Diff ausgeben.                          |

`validate` akzeptiert nur `--json`/`--human`; es erfolgt kein Dateisystemzugriff, daher
gelten `--cwd` und `--file` nicht.

## Syntax von `oc://`

```text
oc://FILE/SECTION/ITEM/FIELD?session=SCOPE
```

Slot-Regeln: `field` erfordert `item`, und `item` erfordert `section`. Für
alle vier Slots gilt:

- **Segmente in Anführungszeichen** — `"a/b.c"` bleibt über die Trennzeichen `/` und `.` hinweg erhalten. Der Inhalt ist
  byte-literal; `"` und `\` sind innerhalb von Anführungszeichen nicht zulässig. Auch der Datei-Slot
  berücksichtigt Anführungszeichen: `oc://"skills/email-drafter"/Tools/$last` behandelt
  `skills/email-drafter` als einzelnen Dateipfad.
- **Prädikate** — `[k=v]`, `[k!=v]`, `[k<v]`, `[k<=v]`, `[k>v]`, `[k>=v]`.
  Numerische Operatoren erfordern, dass beide Seiten in endliche Zahlen umgewandelt werden können.
- **Vereinigungen** — `{a,b,c}` stimmt mit jeder der Alternativen überein.
- **Platzhalter** — `*` (ein einzelnes Untersegment) und `**` (null oder mehr,
  rekursiv). `find` akzeptiert diese; `resolve` und `set` lehnen sie als
  mehrdeutig ab.
- **Positional** — `$first`/`$last` werden zum ersten/letzten Index oder
  deklarierten Schlüssel aufgelöst.
- **Ordinal** — `#N` für den N-ten Treffer in Dokumentreihenfolge.
- **Einfügemarkierungen** — `+`, `+key`, `+nnn` für schlüssel-/indexbasierte Einfügungen
  (mit `set` verwenden).
- **Sitzungsbereich** — `?session=cron-daily` usw. Unabhängig von der Slot-Verschachtelung.
  Sitzungswerte sind unverarbeitet und werden nicht prozentdekodiert; sie dürfen keine Steuerzeichen
  oder reservierten Abfragetrennzeichen enthalten (`?`, `&`, `%`).

Reservierte Zeichen (`?`, `&`, `%`) außerhalb von Segmenten in Anführungszeichen, Prädikaten oder Vereinigungen
werden abgelehnt. Steuerzeichen (U+0000–U+001F, U+007F) werden
überall abgelehnt, einschließlich des Abfragewerts `session`.

`formatOcPath(parseOcPath(path)) === path` ist für kanonische Pfade garantiert.
Nicht kanonische Abfrageparameter werden mit Ausnahme des ersten nicht leeren
Werts `session=` ignoriert.

Feste Grenzwerte: Ein Pfad ist auf 4096 Bytes, höchstens 4 Slots (Datei/Abschnitt/Element/
Feld), höchstens 64 durch Punkte getrennte Untersegmente pro Slot und höchstens 256 verschachtelte
Traversal-Ebenen für tiefe JSON-Pfade begrenzt. Unabhängig davon wird jede JSONC-/JSON-Dateieingabe
über 16 MiB mit einer Parse-Diagnose abgelehnt, statt geparst zu werden, und zwar
bei jedem Verb, das diese Datei lädt.

## Adressierung nach Dateityp

| Typ          | Dateierweiterungen             | Adressierungsmodell                                                                                    |
| ------------- | --------------------------- | --------------------------------------------------------------------------------------------------- |
| Markdown      | `.md`                       | H2-Abschnitte nach Slug, Aufzählungspunkte nach Slug oder `#N`, Frontmatter über `[frontmatter]`.                 |
| JSONC/JSON    | `.jsonc`, `.json`           | Objektschlüssel und Arrayindizes; Punkte trennen verschachtelte Untersegmente, sofern sie nicht in Anführungszeichen stehen.                        |
| JSONL         | `.jsonl`, `.ndjson`         | Zeilenadressen der obersten Ebene (`L1`, `L2`, `$first`, `$last`), anschließend Abstieg im JSONC-Stil innerhalb der Zeile. |
| YAML/.lobster | `.yaml`, `.yml`, `.lobster` | Zuordnungsschlüssel und Sequenzindizes; Kommentare und Flussstil werden von der YAML-Dokument-API verarbeitet.        |

`resolve` gibt einen strukturierten Treffer zurück: `root`, `node`, `leaf` oder
`insertion-point`, mit einer 1-basierten Zeilennummer. Blattwerte werden als
Text zusammen mit einem `leafType` bereitgestellt, sodass Plugin-Autoren Vorschauen rendern können, ohne
von der AST-Struktur des jeweiligen Dateityps abhängig zu sein.

## Mutationsvertrag

`set` schreibt ein konkretes Ziel:

- Markdown-Frontmatter-Werte und `- key: value`-Elementfelder sind
  String-Blätter. Markdown-Einfügungen hängen Abschnitte, Frontmatter-Schlüssel oder
  Abschnittselemente an und rendern eine kanonische Markdown-Form für die geänderte
  Datei. Abschnittsinhalte können nicht als Ganzes über `set` geschrieben werden.
- Bei JSONC-Blattschreibvorgängen wird der String-Wert in den vorhandenen Blatttyp
  umgewandelt (`string`, endliches `number`, `true`/`false` oder `null`). Verwenden Sie `--value-json`,
  wenn bei einer JSONC-/JSON-/JSONL-Blattersetzung `<value>` als JSON geparst werden
  und sich die Struktur ändern darf, etwa wenn eine String-Kurzform für eine Secret-Referenz
  durch ein Objekt ersetzt wird. Bei Einfügungen in JSONC-Objekte und -Arrays wird
  `<value>` als JSON geparst und für gewöhnliche Blattschreibvorgänge der
  `jsonc-parser`-Bearbeitungspfad verwendet, wobei Kommentare und die umgebende
  Formatierung erhalten bleiben.
- JSONL-Blattschreibvorgänge führen innerhalb einer Zeile dieselbe Umwandlung wie JSONC
  durch. Beim Ersetzen und Anhängen ganzer Zeilen wird `<value>` als JSON geparst.
  Gerendertes JSONL behält die vorherrschende LF-/CRLF-Zeilenendekonvention der Datei bei
  (Mehrheitsentscheidung anhand der Zeilenumbrüche in der gesamten Datei, sodass eine
  überwiegend mit CRLF formatierte Datei auch bei einigen vereinzelten LFs CRLF beibehält).
- Bei YAML-Blattschreibvorgängen wird in den vorhandenen Skalartyp umgewandelt
  (`string`, endliches `number`, `true`/`false` oder `null`). YAML-Einfügungen verwenden
  die Dokument-API des mitgelieferten Pakets `yaml` für Aktualisierungen von
  Mappings und Sequenzen. Fehlerhafte YAML-Dokumente mit Parserfehlern werden vor einer
  Änderung mit `parse-error` abgelehnt.

Verwenden Sie `--dry-run` vor benutzersichtbaren Schreibvorgängen, wenn die exakten Bytes
entscheidend sind. JSONC- und YAML-Bearbeitungen ändern das vorhandene Dokument direkt
(über `jsonc-parser` beziehungsweise die Dokument-API von `yaml`), sodass unveränderte
Bytes normalerweise erhalten bleiben; Markdown erstellt die Datei bei jeder Bearbeitung
aus ihrer geparsten Struktur neu, wodurch beiläufige Formatierungen außerhalb des
geänderten Blatts normalisiert werden können. Fügen Sie `--diff` hinzu, wenn Sie die
Vorschau als fokussierten Vorher-/Nachher-Patch statt als vollständige gerenderte Datei
anzeigen möchten.

## Beispiele

```bash
# Einen Pfad validieren (kein Dateisystemzugriff)
openclaw path validate 'oc://AGENTS.md/Tools/$last/risk'

# Ein Blatt lesen
openclaw path resolve 'oc://gateway.jsonc/version'

# Platzhaltersuche
openclaw path find 'oc://session.jsonl/*/event' --file ./logs/session.jsonl

# Einen Schreibvorgang im Probelauf ausführen
openclaw path set 'oc://gateway.jsonc/version' '2.0' --dry-run

# Einen Schreibvorgang im Probelauf als Unified Diff ausführen
openclaw path set 'oc://gateway.jsonc/version' '2.0' --dry-run --diff

# Den Schreibvorgang anwenden
openclaw path set 'oc://gateway.jsonc/version' '2.0'

# Bytegetreue Rundreise (Diagnose)
openclaw path emit ./AGENTS.md
```

Weitere Grammatikbeispiele:

```bash
# Schlüssel in Anführungszeichen setzen, die / oder . enthalten
openclaw path resolve 'oc://config.jsonc/agents.defaults.models/"anthropic/claude-opus-4-7"/alias'

# Tiefe JSON-/JSONC-Pfade können Schrägstrichsegmente verwenden; diese werden zu gepunkteten Untersegmenten normalisiert
openclaw path set 'oc://openclaw.json/agents/list/0/tools/exec/security' 'allowlist' --dry-run

# Ein JSONC-Blatt durch ein geparstes Objekt ersetzen
openclaw path set 'oc://openclaw.json/gateway/auth/token' '{"source":"file","provider":"secrets","id":"/test"}' --value-json --dry-run

# Prädikatssuche über untergeordnete JSONC-Elemente
openclaw path find 'oc://config.jsonc/plugins/[enabled=true]/id'

# In ein JSONC-Array einfügen
openclaw path set 'oc://config.jsonc/items/+1' '{"id":"new","enabled":true}' --dry-run

# Einen JSONC-Objektschlüssel einfügen
openclaw path set 'oc://config.jsonc/plugins/+github' '{"enabled":true}' --dry-run

# Ein JSONL-Ereignis anhängen
openclaw path set 'oc://session.jsonl/+' '{"event":"checkpoint","ok":true}' --file ./logs/session.jsonl

# Die letzte JSONL-Wertzeile auflösen
openclaw path resolve 'oc://session.jsonl/$last/event' --file ./logs/session.jsonl

# Einen YAML-Workflow-Schritt auflösen
openclaw path resolve 'oc://workflow.yaml/steps/0/id'

# Einen YAML-Skalar aktualisieren
openclaw path set 'oc://workflow.yaml/steps/$last/id' 'classify-renamed' --dry-run

# Markdown-Frontmatter adressieren
openclaw path resolve 'oc://AGENTS.md/[frontmatter]/name'

# Markdown-Frontmatter einfügen
openclaw path set 'oc://AGENTS.md/[frontmatter]/+description' 'Agent instructions' --dry-run

# Markdown-Elementfelder suchen
openclaw path find 'oc://SKILL.md/Tools/*/send_email'

# Einen sitzungsbezogenen Pfad validieren
openclaw path validate 'oc://AGENTS.md/Tools/$last/risk?session=cron-daily'
```

## Rezepte nach Dateiart

Dieselben fünf Verben funktionieren für alle Arten; das Adressierungsschema
entscheidet anhand der Dateierweiterung.

### Markdown

```text
<!-- frontmatter.md -->
---
name: drafter
description: Agent zum Verfassen von E-Mails
tier: core
---
## Werkzeuge
- gh: GitHub-CLI
- curl: HTTP-Client
- send_email: aktiviert
```

```bash
$ openclaw path resolve 'oc://x.md/[frontmatter]/tier' --file frontmatter.md --human
Blatt @ L4: "core" (String)

$ openclaw path resolve 'oc://x.md/tools/gh/gh' --file frontmatter.md --human
Blatt @ L9: "GitHub-CLI" (String)

$ openclaw path find 'oc://x.md/tools/*' --file frontmatter.md --human
3 Treffer für oc://x.md/tools/*:
  oc://x.md/tools/gh           →  Node @ L9 [md-item]
  oc://x.md/tools/curl         →  Node @ L10 [md-item]
  oc://x.md/tools/send-email   →  Node @ L11 [md-item]
```

Das Prädikat `[frontmatter]` adressiert den YAML-Frontmatter-Block; `tools`
findet die Überschrift `## Tools` anhand ihres Slugs, und Elementblätter behalten
ihre Slug-Form bei, selbst wenn die Quelle Unterstriche verwendet
(`send_email` wird zu `send-email`).

### JSONC

```text
// config.jsonc
{
  "plugins": {
    "github": {"enabled": true, "role": "vcs"},
    "slack":  {"enabled": false, "role": "chat"}
  }
}
```

```bash
$ openclaw path resolve 'oc://config.jsonc/plugins/github/enabled' --file config.jsonc --human
Blatt @ L4: "true" (Boolesch)

$ openclaw path set 'oc://config.jsonc/plugins/slack/enabled' 'true' --file config.jsonc --dry-run
--dry-run: würde 142 Bytes in /…/config.jsonc schreiben
{
  "plugins": {
    "github": {"enabled": true, "role": "vcs"},
    "slack":  {"enabled": true, "role": "chat"}
  }
}
```

JSONC-Bearbeitungen werden über `jsonc-parser` ausgeführt, sodass Kommentare und
Leerraum einen `set` überstehen. Führen Sie zunächst `--dry-run` aus,
um die Bytes vor der Übernahme zu prüfen. `.json`-Dateien verwenden denselben
Adapter und Bearbeitungspfad wie `.jsonc`.

### JSONL

```text
{"event":"start","userId":"u1","ts":1}
{"event":"action","userId":"u1","ts":2}
{"event":"end","userId":"u1","ts":3}
```

```bash
$ openclaw path find 'oc://session.jsonl/[event=action]/userId' --file session.jsonl --human
1 Treffer für oc://session.jsonl/[event=action]/userId:
  oc://session.jsonl/L2/userId  →  Blatt @ L2: "u1" (String)

$ openclaw path resolve 'oc://session.jsonl/L2/ts' --file session.jsonl --human
Blatt @ L2: "2" (Zahl)
```

Jede Zeile ist ein Datensatz. Adressieren Sie ihn über ein Prädikat
(`[event=action]`), wenn Sie die Zeilennummer nicht kennen, oder über das kanonische
Segment `LN`, wenn sie bekannt ist. `.ndjson`-Dateien verwenden
denselben Adapter wie `.jsonl`.

### YAML

```text
# workflow.yaml
name: inbox-triage
steps:
  - id: fetch
    command: gmail.search
  - id: classify
    command: openclaw.invoke
```

```bash
$ openclaw path resolve 'oc://workflow.yaml/steps/0/id' --file workflow.yaml --human
Blatt @ L3: "fetch" (String)

$ openclaw path set 'oc://workflow.yaml/steps/$last/id' 'classify-renamed' --file workflow.yaml --dry-run
--dry-run: würde 99 Bytes in /…/workflow.yaml schreiben
name: inbox-triage
steps:
  - id: fetch
    command: gmail.search
  - id: classify-renamed
    command: openclaw.invoke
```

YAML verwendet die `Document`-API des Pakets `yaml` statt eines
selbst entwickelten Parsers. Dadurch bleiben bei gewöhnlichen Parse-/Ausgaberundreisen
Kommentare und die Autorenstruktur erhalten, während aufgelöste Pfade dasselbe Modell
aus Mapping-Schlüsseln und Sequenzindizes wie JSONC verwenden. Derselbe Adapter
verarbeitet `.yaml`-, `.yml`- und `.lobster`-Dateien.

## Unterbefehlsreferenz

### `resolve <oc-path>`

Liest ein einzelnes Blatt oder einen einzelnen Node. Platzhalter werden abgelehnt –
verwenden Sie dafür `find`. Beendet sich bei einem Treffer mit
`0`, bei einem regulären Fehltreffer mit `1` und bei einem
Parsefehler oder abgelehnten Muster mit `2`.

```bash
openclaw path resolve 'oc://AGENTS.md/tools/gh/risk' --human
openclaw path resolve 'oc://gateway.jsonc/server/port' --json
```

### `find <pattern>`

Listet jeden Treffer für ein Platzhalter-, Prädikat- oder Vereinigungsmuster auf.
Beendet sich bei mindestens einem Treffer mit `0`, bei keinem Treffer
mit `1`. Platzhalter im Dateislot werden mit `OC_PATH_FILE_WILDCARD_UNSUPPORTED` abgelehnt –
geben Sie eine konkrete Datei an (Globbing über mehrere Dateien ist eine spätere Funktion).

```bash
openclaw path find 'oc://AGENTS.md/tools/**/risk'
openclaw path find 'oc://session.jsonl/[event=action]/userId'
openclaw path find 'oc://config.jsonc/plugins/{github,slack}/enabled'
```

### `set <oc-path> <value>`

Schreibt ein Blatt. Kombinieren Sie den Befehl mit `--dry-run`, um die zu
schreibenden Bytes in einer Vorschau anzuzeigen, ohne die Datei zu verändern.
Fügen Sie `--diff` für eine Vorschau als Unified Diff hinzu. Beendet sich
nach einem erfolgreichen Schreibvorgang mit `0`, mit `1`,
wenn das Substrat den Vorgang ablehnt (beispielsweise beim Auslösen einer
Sentinel-Schutzbedingung), und bei Parsefehlern mit `2`.

```bash
openclaw path set 'oc://gateway.jsonc/version' '2.0' --dry-run
openclaw path set 'oc://gateway.jsonc/version' '2.0' --dry-run --diff
openclaw path set 'oc://gateway.jsonc/version' '2.0'
openclaw path set 'oc://AGENTS.md/Tools/+gh/risk' 'low'
```

Die Einfügungsmarkierung `+key` erstellt das benannte untergeordnete Element,
wenn es noch nicht vorhanden ist; `+nnn` beziehungsweise ein alleinstehendes
`+` dienen zur indizierten Einfügung beziehungsweise zum Anhängen.

### `validate <oc-path>`

Reine Parseprüfung. Kein Dateisystemzugriff. Nützlich, um zu bestätigen, dass ein
Vorlagenpfad korrekt aufgebaut ist, bevor Variablen eingesetzt werden, oder um die
strukturelle Aufschlüsselung zur Fehlerdiagnose anzuzeigen:

```bash
$ openclaw path validate 'oc://AGENTS.md/tools/gh' --human
gültig: oc://AGENTS.md/tools/gh
  Datei:     AGENTS.md
  Abschnitt: tools
  Element:   gh
```

Beendet sich bei Gültigkeit mit `0`, bei Ungültigkeit mit
`1` (mit strukturierten Angaben für `code` und
`message`) und bei Argumentfehlern mit `2`.

### `emit <file>`

Führt eine Datei durch den Parser und Emitter ihrer jeweiligen Art und wieder zurück.
Bei einer fehlerfreien Datei sollte die Ausgabe byteidentisch mit der Eingabe sein;
Abweichungen weisen auf einen Parserfehler oder das Auslösen einer Sentinel-Bedingung hin.
Nützlich zur Fehlerdiagnose des Substratverhaltens bei realen Eingaben.

```bash
openclaw path emit ./AGENTS.md
openclaw path emit ./gateway.jsonc --json
```

## Exit-Codes

| Code | Bedeutung                                                                  |
| ---- | -------------------------------------------------------------------------- |
| `0`  | Erfolg. (`resolve` / `find`: mindestens ein Treffer. `set`: Schreibvorgang erfolgreich.) |
| `1`  | Kein Treffer oder `set` wurde vom Substrat abgelehnt (kein Fehler auf Systemebene). |
| `2`  | Argument- oder Parsefehler.                                                |

## Ausgabemodus

`openclaw path` berücksichtigt TTY: menschenlesbare Ausgabe in einem Terminal, JSON,
wenn stdout über eine Pipe weitergeleitet oder umgeleitet wird. `--json` und
`--human` überschreiben die automatische Erkennung.

## Hinweise

- `set` schreibt Bytes über den Emit-Pfad des Substrats, der die
  Redaction-Sentinel-Schutzprüfung automatisch anwendet. Ein Blatt, das
  `__OPENCLAW_REDACTED__` enthält (wortgetreu oder als Teilzeichenfolge), wird zum
  Schreibzeitpunkt abgelehnt.
- Das JSONC-Parsen und Bearbeiten von Blättern verwendet die Plugin-lokale Abhängigkeit `jsonc-parser`,
  sodass Kommentare und Formatierung bei gewöhnlichen Schreibvorgängen an Blättern erhalten
  bleiben, anstatt einen selbst entwickelten Parser-/Neurendering-Pfad zu durchlaufen.
- `path` kennt weder die Verfolgung noch die Wiederherstellung der zuletzt als funktionsfähig bekannten Konfiguration (LKG);
  dieser Lebenszyklus wird an anderer Stelle verwaltet. Wenn eine Datei, die Sie über `path` bearbeiten,
  ebenfalls per LKG verfolgt wird, entscheidet der nächste Lesevorgang der Konfiguration, ob sie übernommen oder
  wiederhergestellt wird; behandeln Sie eine Bearbeitung mit `path` genauso wie jeden anderen direkten Schreibvorgang in
  diese Datei.

## Verwandte Themen

- [CLI-Referenz](/de/cli)
