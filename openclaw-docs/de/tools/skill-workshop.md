---
read_when:
    - Sie möchten, dass der Agent über den Chat einen Skill erstellt oder aktualisiert.
    - Sie müssen einen generierten Skill-Entwurf prüfen, anwenden, ablehnen oder unter Quarantäne stellen
    - Sie konfigurieren Genehmigung, Autonomie, Speicher oder Beschränkungen für den Skill Workshop
    - Sie möchten verstehen, wo Vorschläge zum selbstständigen Lernen überprüft werden
sidebarTitle: Skill Workshop
summary: Workspace-Skills durch die Überprüfung in Skill Workshop erstellen und aktualisieren
title: Skills-Workshop
x-i18n:
    generated_at: "2026-07-26T18:10:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2c2590f2a1bcad3b22ef8504eac7b3a44611c3fedc0df3832660f8926ce04252
    source_path: tools/skill-workshop.md
    workflow: 16
---

Skill Workshop ist der kontrollierte Weg von OpenClaw zum Erstellen und Aktualisieren von Workspace-
Skills. Agenten und Operatoren schreiben über diesen Weg niemals direkt `SKILL.md` –
sie erstellen einen **Vorschlag** (einen ausstehenden Entwurf mit Inhalt, Ziel-
bindung, Scannerstatus, Hashes und Rollback-Metadaten), der erst durch Anwenden
zu einem aktiven Skill wird.

Skill Workshop schreibt ausschließlich Workspace-Skills. Gebündelte,
Plugin-, ClawHub-, Extra-Root-, verwaltete, persönliche Agenten- oder System-Skills werden niemals verändert.

## Funktionsweise

- **Vorschlag zuerst:** Generierte Inhalte werden als `PROPOSAL.md` gespeichert, nicht
  als `SKILL.md`.
- **Anwenden ist der einzige aktive Schreibvorgang:** Erstellen, Aktualisieren und Überarbeiten ändern niemals
  aktive Skills.
- **Auf den Workspace beschränkt:** Beim Erstellen ist das Workspace-Stammverzeichnis `skills/` das Ziel; Aktualisierungen
  sind nur für beschreibbare Workspace-Skills zulässig.
- **Kein Überschreiben:** Das Erstellen schlägt fehl, wenn der Ziel-Skill bereits vorhanden ist.
- **An Hash gebunden:** Aktualisierungsvorschläge werden an den aktuellen Ziel-Hash gebunden und werden
  zu `stale`, wenn sich der aktive Skill vor dem Anwenden ändert.
- **Durch Scanner abgesichert:** Vor dem Schreiben führt das Anwenden den Sicherheitsscanner erneut aus.
- **Wiederherstellbar:** Vor der Änderung aktiver Dateien schreibt das Anwenden Rollback-Metadaten.
- **Einheitliche Oberflächen:** Chat, CLI und Gateway rufen denselben Dienst auf.

## Lebenszyklus

```text
create/update -> ausstehend
revise        -> ausstehend
apply         -> angewendet
reject        -> abgelehnt
quarantine    -> unter Quarantäne
target change -> veraltet
```

Nur ein Vorschlag mit dem Status `pending` kann überarbeitet, angewendet, abgelehnt oder unter Quarantäne gestellt werden.

## Lebenszyklus-Kuratierung

Das Gateway erfasst die aggregierte Skill-Nutzung in der gemeinsamen Zustandsdatenbank. Einmal
täglich überprüft es Skills, die von Skill Workshop erstellt und angewendet wurden. Skills, die länger
als 30 Tage nicht verwendet wurden, werden zu `stale`; nach 90 Tagen werden sie zu `archived` und
nicht mehr in neue Skill-Snapshots von Agenten aufgenommen. Archivierte Skill-Dateien bleiben auf
dem Datenträger unverändert. Manuell erstellte Skills werden niemals kuratiert; nur Skills, die durch Skill-
Workshop-Vorschläge erstellt wurden, werden in die Lebenszyklus-Kuratierung aufgenommen.

Angeheftete Skills umgehen Lebenszyklusübergänge. Ein veralteter Skill kehrt zu `active`
zurück, nachdem er verwendet und der nächste Durchlauf ausgeführt wurde. Archivierte Skills kehren nur durch eine
explizite Wiederherstellung zurück:

Lebenszyklusübergänge und Wiederherstellungen gelten für neue Sitzungen; laufende Sitzungen behalten
ihren aktuellen Skill-Snapshot.

```bash
openclaw skills curator status
openclaw skills curator pin <skill>
openclaw skills curator unpin <skill>
openclaw skills curator restore <skill>
```

Alle Kuratorbefehle akzeptieren `--json`. Der Status meldet außerdem deterministische Überschneidungs-
kandidaten ausschließlich als Vorschläge; Skills werden niemals zusammengeführt und kein Modell wird aufgerufen.

## Chat

Fordern Sie den Agenten zum gewünschten Skill auf; er ruft `skill_workshop` auf und gibt eine
Vorschlags-ID zurück.

### Aus der letzten Arbeit lernen

Verwenden Sie `/learn`, um die aktuelle Unterhaltung oder benannte Quellen in einen
standardgeleiteten Skill-Vorschlag umzuwandeln:

```text
/learn
/learn docs/runbook.md und https://example.com/guide; Schwerpunkt auf Wiederherstellung
```

Ohne Anforderung weist `/learn` den Agenten an, den wiederverwendbaren Arbeitsablauf aus
der aktuellen Unterhaltung zu extrahieren. Mit einer Anforderung behandelt der Agent Pfade, URLs, eingefügte
Notizen und Verweise auf Unterhaltungen als Quellen und berücksichtigt dabei Anforderungen an Schwerpunkt, Umfang und
Benennung. Er sammelt die Quellen mit seinen vorhandenen Werkzeugen und ruft anschließend
`skill_workshop` mit `action: "create"` auf.

Der resultierende Vorschlag bleibt `pending`; `/learn` wendet ihn niemals an. Prüfen und
wenden Sie ihn über den normalen Genehmigungsablauf oder mit `openclaw skills workshop` an.

Erstellen:

```text
Erstellen Sie einen Skill namens morning-catchup, der meine Posteingangsroutine am Montag ausführt.
```

Einen vorhandenen Workspace-Skill aktualisieren:

```text
Aktualisieren Sie trip-planning so, dass vor der Buchung auch Sitzpläne geprüft werden.
```

Einen ausstehenden Vorschlag iterativ bearbeiten:

```text
Zeigen Sie mir den Vorschlag morning-catchup.
Überarbeiten Sie ihn so, dass auch alle als dringend markierten Elemente hervorgehoben werden.
Wenden Sie den Vorschlag morning-catchup an.
```

Vom Agenten initiierte Vorgänge für `apply`, `reject` und `quarantine` werden standardmäßig ohne eine zusätzliche
Genehmigungsaufforderung ausgeführt. Setzen Sie `skills.workshop.approvalPolicy` auf `"pending"`,
um vor diesen Aktionen die Genehmigung eines Operators anzufordern.

Wenn eine Genehmigung erforderlich ist, nennt die Aufforderung die Vorschlags-ID und den Ziel-
Skill und zeigt die Vorschlagsbeschreibung, die Anzahl der unterstützenden Dateien und die Größe des Haupttexts.
Genehmigungsanfragen sind zeitlich so begrenzt, dass sie vor dem Watchdog des Agentenwerkzeugs abgeschlossen werden. Wenn vor
Ablauf der Aufforderung keine Entscheidung eingeht, wird die Lebenszyklusaktion nicht ausgeführt:
Der Vorschlag bleibt ausstehend und unverändert. Entscheiden Sie später in der Benutzeroberfläche von Skill Workshop oder führen Sie
`openclaw skills workshop apply|reject|quarantine <proposal-id>` aus. Agenten sollten
eine abgelaufene Lebenszyklusaktion nicht in einer Schleife wiederholen.

## CLI

```bash
# Erstellen
openclaw skills workshop propose-create \
  --name morning-catchup \
  --description "Tägliche Posteingangsaufarbeitung: sichten, archivieren, hervorheben, entwerfen, planen" \
  --proposal ./PROPOSAL.md

# Einen vorhandenen Workspace-Skill aktualisieren
openclaw skills workshop propose-update trip-planning --proposal ./PROPOSAL.md

# Auflisten und prüfen
openclaw skills workshop list
openclaw skills workshop inspect <proposal-id>

# Vor der Genehmigung überarbeiten
openclaw skills workshop revise <proposal-id> --proposal ./PROPOSAL.md

# Abschließen
openclaw skills workshop apply <proposal-id>
openclaw skills workshop reject <proposal-id> --reason "Duplikat"
openclaw skills workshop quarantine <proposal-id> --reason "Sicherheitsprüfung erforderlich"
```

Jeder Unterbefehl akzeptiert `--agent <id>` (Ziel-Workspace; standardmäßig
aus dem aktuellen Arbeitsverzeichnis abgeleitet, danach der Standardagent) und `--json` (strukturierte Ausgabe).
`propose-create`, `propose-update` und `revise` akzeptieren außerdem `--goal <text>` und
`--evidence <text>`, um den Vorschlagskontext zusammen mit `--proposal` aufzuzeichnen.

## Vorschlagsinhalt

Solange der Vorschlag ausstehend ist, wird er als `PROPOSAL.md` mit ausschließlich für Vorschläge vorgesehenem
Frontmatter gespeichert:

```markdown
---
name: "morning-catchup"
description: "Tägliche Posteingangsaufarbeitung: sichten, archivieren, hervorheben, entwerfen, planen"
status: proposal
version: "v1"
date: "2026-05-30T00:00:00.000Z"
---
```

Beim Anwenden schreibt Skill Workshop die aktive Datei `SKILL.md` und entfernt die
ausschließlich für Vorschläge vorgesehenen Felder: `status`, Vorschlags-`version` und Vorschlags-`date`.

## Unterstützende Dateien

Verwenden Sie `--proposal-dir`, wenn der vorgeschlagene Skill Dateien neben
`PROPOSAL.md` benötigt:

```bash
openclaw skills workshop propose-create \
  --name weekly-update \
  --description "Freitagsrückblick: Statistiken, Höhepunkte, die drei wichtigsten Punkte der nächsten Woche" \
  --proposal-dir ./weekly-update-proposal
```

Das Verzeichnis muss `PROPOSAL.md` enthalten. Unterstützende Dateien müssen unter
`assets/`, `examples/`, `references/`, `scripts/` oder `templates/` liegen. Skill
Workshop scannt, hasht und speichert sie mit dem Vorschlag und schreibt sie erst beim Anwenden
neben die aktive Datei `SKILL.md`.

Abgelehnte Pfade für unterstützende Dateien: absolute Pfade, verborgene Pfadsegmente, Pfad-
traversierung, sich überschneidende Pfade, ausführbare Dateien, Nicht-UTF-8-Text, Nullbytes
und Pfade außerhalb der standardmäßigen Unterstützungsordner.

## Agentenwerkzeug

Das Modell verwendet `skill_workshop` mit einem erforderlichen Parameter `action`:
`create | update | revise | list | inspect | apply | reject | quarantine`.
Weitere Parameter gelten abhängig von der Aktion:

| Parameter                  | Verwendet von                                          | Hinweise                                                              |
| -------------------------- | ------------------------------------------------------ | --------------------------------------------------------------------- |
| `name`                     | `create`, `inspect`, `revise`                        | Erforderlich für `create`; löst andernfalls einen ausstehenden Vorschlag anhand des Namens auf |
| `description`              | `create`, `update`, `revise`                         | Max. 160 Byte                                                         |
| `skill_name`               | `update`                                             | Name oder Schlüssel eines vorhandenen Skills                          |
| `proposal_content`         | `create`, `update`, `revise`                         | Als `PROPOSAL.md` gespeichert; durch `skills.workshop.maxSkillBytes` begrenzt |
| `support_files`            | `create`, `update`, `revise`                         | Array von `{ path, content }`                                          |
| `goal`, `evidence`         | `create`, `update`, `revise`                         | Freitextkontext                                                       |
| `proposal_id`              | `inspect`, `revise`, `apply`, `reject`, `quarantine` | Zielvorschlag                                                         |
| `reason`                   | `apply`, `reject`, `quarantine`                      | Optional                                                              |
| `query`, `status`, `limit` | `list`                                               | Filtern/paginieren; `limit` max. 50, Standardwert 20       |

Agenten müssen `skill_workshop` für generierte Skill-Arbeiten verwenden. Sie dürfen
Vorschlagsdateien nicht über `write`, `edit`, `exec`, Shell-
Befehle oder direkte Dateisystemoperationen erstellen oder ändern.

<Note>
`skill_workshop` ist ein integriertes Agentenwerkzeug und in
`tools.profile: "coding"` enthalten. Wenn es durch eine strengere Richtlinie ausgeblendet wird, fügen Sie
`skill_workshop` zur aktiven Liste `tools.allow` hinzu oder verwenden Sie
`tools.alsoAllow: ["skill_workshop"]`, wenn der Geltungsbereich ein Profil ohne explizites
`tools.allow` verwendet. In Sandbox-Ausführungen wird das hostseitige
Skill-Workshop-Werkzeug nicht erstellt. Führen Sie Aktionen zur Vorschlagsprüfung daher aus einer normalen hostseitigen
Agentensitzung oder über die CLI aus.
</Note>

## Vorgeschlagene Skills

OpenClaw erkennt dauerhafte Anweisungen wie „beim nächsten Mal“, „merken Sie sich“ und reaktive Korrekturen,
wenn ein interaktiver Durchlauf endet, einschließlich fehlgeschlagener Durchläufe. Beim nächsten Durchlauf bietet der Agent an, den
zuletzt erkannten Arbeitsablauf über `skill_workshop` zu speichern; der Benutzer entscheidet, ob ein
Vorschlag erstellt wird. Dieser integrierte Vorschlag erstellt oder ändert selbst keinen Skill. Aktivieren Sie
`skills.workshop.autonomous.enabled`, um stattdessen direkt ausstehende Vorschläge zu erstellen. In der Control-
UI bietet der Tab „Workshop“ dieselbe Einstellung als Umschalter **Selbstlernend** im Seitenkopf und
als Schaltfläche zum Aktivieren auf der leeren Vorschlagstafel.

### Frühere Sitzungen durchsuchen

Die Control UI kann ältere Arbeiten prüfen, ohne autonomes Selbstlernen zu aktivieren.
Öffnen Sie **Plugins → Workshop** und wählen Sie **Skill-Ideen suchen**. Der Scan beginnt mit
den neuesten geeigneten Sitzungen und prüft ein begrenztes Fenster substanzieller Arbeiten.
Cron-, Heartbeat-, Hook-, Subagenten-, ACP-, Plugin-eigene und interne Prüf-
sitzungen sowie Unterhaltungen mit weniger als sechs Modelldurchläufen werden übersprungen.

Der Prüfer verwendet das konfigurierte Modell des ausgewählten Agenten und erhält ein
von Geheimnissen bereinigtes, größenbegrenztes Transkriptpaket. Dabei gilt derselbe konservative
Maßstab wie bei der Erfahrungsprüfung: ein konkretes Wiederherstellungsmuster oder ein stabiles Verfahren, das
mindestens zwei zukünftige Modell- oder Werkzeugaufrufe einsparen würde. Routinearbeiten und einmalige
Fakten sollten keinen Vorschlag erzeugen.

Ein Scan kann höchstens drei ausstehende Vorschläge erstellen oder überarbeiten. Er kann keinen aktiven Skill
anwenden, ablehnen, unter Quarantäne stellen oder bearbeiten. Workshop zeigt die kumulative Abdeckung an,
zum Beispiel **20 Sitzungen geprüft · 18. Juni–heute · 2 Ideen gefunden**. Wählen Sie
**Frühere Arbeiten scannen**, um ab dem gespeicherten Cursor der ältesten Sitzung fortzufahren. Nachdem
der verfügbare Verlauf ausgeschöpft ist, wird die Aktion zu **Neue Arbeiten scannen**.

Die historische Überprüfung erfolgt manuell, selbst wenn
`skills.workshop.autonomous.enabled` auf `false` gesetzt ist. Jeder Klick startet einen Modelldurchlauf,
daher gelten die Preisgestaltung und die Datenverarbeitungsbedingungen des Providers. Der Cursor und die Abdeckungszahlen
werden in der gemeinsam genutzten OpenClaw-Zustandsdatenbank gespeichert; Transkriptinhalte werden nicht
in den Scanstatus kopiert.

Wenn die autonome Erfassung aktiviert ist, kann OpenClaw außerdem nach erfolgreicher,
umfangreicher Arbeit und nachdem das gesamte Agentensystem inaktiv geworden ist, eine konservative Überprüfung durchführen. Diese isolierte Überprüfung kann höchstens
einen ausstehenden Vorschlag erstellen oder überarbeiten. Sie kann weder einen aktiven Skill aktualisieren noch einen
Vorschlag anwenden, ablehnen oder unter Quarantäne stellen, selbst wenn `approvalPolicy` auf `"auto"` gesetzt ist.

Einzelheiten zu Aktivierung, Eignung, Datenschutz und Kosten,
zum Vorschlagsschwellenwert und zur Fehlerbehebung finden Sie unter [Selbstlernen](/de/tools/self-learning).

## Genehmigung und Autonomie

```json5
{
  skills: {
    workshop: {
      autonomous: {
        enabled: false,
      },
      allowSymlinkTargetWrites: false,
      approvalPolicy: "auto",
      maxPending: 50,
      maxSkillBytes: 40000,
    },
  },
}
```

| Einstellung                    | Standardwert  | Auswirkung                                                                                                                                                              |
| -------------------------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `autonomous.enabled`       | `false`  | Erstellt aus ausdrücklichen Korrekturen sowie nach einer Inaktivitätsverzögerung aus umfangreicher abgeschlossener Arbeit mit wiederverwendbarer Wiederherstellung oder erheblichen Einsparungen bei Hin- und Rückläufen ausstehende Vorschläge.   |
| `allowSymlinkTargetWrites` | `false`  | Ermöglicht beim Anwenden Schreibzugriffe über Skill-Symlinks im Arbeitsbereich, deren tatsächliches Ziel in `skills.load.allowSymlinkTargets` aufgeführt ist.                                                 |
| `approvalPolicy`           | `"auto"` | `"auto"` überspringt eine zusätzliche Aufforderung für vom Agenten initiierte Aktionen vom Typ `apply`, `reject` oder `quarantine` (der Agent muss die Aktion dennoch aufrufen). `"pending"` erfordert eine Genehmigung. |
| `maxPending`               | `50`     | Begrenzt ausstehende und unter Quarantäne gestellte Vorschläge pro Arbeitsbereich (1-200).                                                                                                       |
| `maxSkillBytes`            | `40000`  | Begrenzt die Größe des Vorschlagstexts in Byte (1024-200000).                                                                                                                     |

Die autonome Erfassung erkennt prospektive Regeln (beispielsweise „von nun an“) und reaktive
Korrekturen (beispielsweise „das ist nicht das, worum ich gebeten habe“). Sie gruppiert neue Anweisungen nach Thema in bis
zu drei Vorschläge pro Durchlauf, leitet Übereinstimmungen im Vokabular an vorhandene beschreibbare Skills im Arbeitsbereich weiter und
überarbeitet ihren eigenen ausstehenden Vorschlag, wenn eine weitere Korrektur denselben Skill betrifft.

Bei erfolgreicher umfangreicher Arbeit ohne ausdrückliche Korrektur entscheidet ein isolierter Durchlauf des ausgewählten
Modells, ob der abgeschlossene Verlauf die konservative Schwelle für Vorschläge überschreitet. Das
Vordergrundmodell wird nicht zum Lernen aufgefordert, bevor es antwortet. Die Hintergrundüberprüfung bewahrt den
Vordergrunddurchlauf als Herkunftsnachweis des Vorschlags, kann nicht auf allgemeine Agentenwerkzeuge zugreifen und keine Entscheidungen über den
Lebenszyklus treffen. Die Überprüfung beginnt erst, wenn die Vordergrundlaufzeit sowohl ihr exakt aufgelöstes Modell meldet
als auch bestätigt, dass `skill_workshop` tatsächlich verfügbar war. Eine restriktive oder unbekannte Werkzeugrichtlinie
schlägt daher sicher fehl und erstellt keinen Vorschlag.

Das vollständige autonome Überprüfungsverhalten und Sicherheitsmodell finden Sie unter
[Selbstlernen](/de/tools/self-learning).

Vorschlagsbeschreibungen sind unabhängig von
`maxSkillBytes` stets auf 160 Byte begrenzt.

## Gateway-Methoden

| Methode                             | Geltungsbereich            |
| ---------------------------------- | ---------------- |
| `skills.proposals.list`            | `operator.read`  |
| `skills.proposals.inspect`         | `operator.read`  |
| `skills.proposals.historyStatus`   | `operator.read`  |
| `skills.proposals.historyScan`     | `operator.admin` |
| `skills.proposals.create`          | `operator.admin` |
| `skills.proposals.update`          | `operator.admin` |
| `skills.proposals.revise`          | `operator.admin` |
| `skills.proposals.requestRevision` | `operator.admin` |
| `skills.proposals.apply`           | `operator.admin` |
| `skills.proposals.reject`          | `operator.admin` |
| `skills.proposals.quarantine`      | `operator.admin` |
| `skills.curator.status`            | `operator.read`  |
| `skills.curator.pin`               | `operator.admin` |
| `skills.curator.unpin`             | `operator.admin` |
| `skills.curator.restore`           | `operator.admin` |

`requestRevision` ist ausschließlich über das Gateway verfügbar (ohne Entsprechung in der CLI oder den Agentenwerkzeugen): Die Methode
leitet frei formulierte Überarbeitungsanweisungen an die Chatsitzung des zuständigen Agenten weiter,
anstatt `PROPOSAL.md` direkt zu ersetzen. Dies ist für Benutzeroberflächen vorgesehen, die den Agenten zur
Überarbeitung auffordern, statt wortwörtlich neue Inhalte zu übermitteln.

`historyStatus` und `historyScan` sind Unterstützungsmethoden für die Control UI. `historyScan`
akzeptiert `direction: "older" | "newer"`; die Ergebnisse bleiben dabei immer als ausstehende
Vorschläge erhalten.

## Speicherung

```text
<OPENCLAW_STATE_DIR>/skill-workshop/
  proposals.json
  proposals/<proposal-id>/
    proposal.json
    PROPOSAL.md
    rollback.json
    assets/
    examples/
    references/
    scripts/
    templates/
```

Standardmäßiges Zustandsverzeichnis: `~/.openclaw`.

- `proposal.json`: kanonischer Vorschlagsdatensatz.
- `proposals.json`: schneller Auflistungsindex, der aus den Vorschlagsordnern neu erstellt werden kann.
- `PROPOSAL.md`: ausstehender Skill-Vorschlag.
- `rollback.json`: Wiederherstellungsmetadaten, die geschrieben werden, bevor die Anwendung aktive Dateien ändert.

## Grenzwerte

| Grenzwert                           | Wert                                                                |
| ------------------------------- | -------------------------------------------------------------------- |
| Beschreibung                     | 160 Byte                                                            |
| Vorschlagstext                   | `skills.workshop.maxSkillBytes` (Standardwert 40.000; feste Obergrenze 1 MiB) |
| Unterstützungsdateien                   | 64 pro Vorschlag                                                      |
| Größe der Unterstützungsdateien               | jeweils 256 KiB, insgesamt 2 MiB                                            |
| Ausstehende und unter Quarantäne gestellte Vorschläge | `skills.workshop.maxPending` pro Arbeitsbereich (Standardwert 50)              |

## Fehlerbehebung

| Problem                                        | Lösung                                                                                                                                                                                                  |
| ---------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Skill proposal description is too large`      | Kürzen Sie `description` auf höchstens 160 Byte.                                                                                                                                                                 |
| `Skill proposal content is too large`          | Kürzen Sie den Vorschlagstext oder erhöhen Sie `skills.workshop.maxSkillBytes`.                                                                                                                                         |
| `Target skill changed after proposal creation` | Überarbeiten Sie den Vorschlag anhand des aktuellen Ziels oder erstellen Sie einen neuen Vorschlag.                                                                                                                                   |
| `Proposal scan failed`                         | Prüfen Sie die Scannerbefunde und überarbeiten Sie den Vorschlag anschließend oder stellen Sie ihn unter Quarantäne.                                                                                                                                           |
| `untrusted symlink target`                     | Konfigurieren Sie `skills.load.allowSymlinkTargets` und aktivieren Sie `skills.workshop.allowSymlinkTargetWrites` nur für absichtlich gemeinsam genutzte Skill-Stammverzeichnisse.                                                                  |
| `Support file paths must be under one of...`   | Verschieben Sie Unterstützungsdateien nach `assets/`, `examples/`, `references/`, `scripts/` oder `templates/`.                                                                                                                |
| Vorschlag wird nicht in der Liste angezeigt                 | Überprüfen Sie den ausgewählten `--agent`-Arbeitsbereich und `OPENCLAW_STATE_DIR`.                                                                                                                                            |
| Agent kann `skill_workshop` nicht aufrufen             | Überprüfen Sie die aktive Werkzeugrichtlinie und den Ausführungsmodus. `coding` enthält das Werkzeug; restriktive `tools.allow`-Richtlinien müssen es ausdrücklich aufführen, und Sandbox-Ausführungen müssen eine normale hostseitige Agentensitzung oder die CLI verwenden. |

### Diagnose der Werkzeugrichtlinie

Wenn die autonome Erfassung aktiviert ist, führt `openclaw doctor` die
Prüfung `core/doctor/skill-workshop-tool-policy` für den Standardagenten aus. Wenn die Richtlinie
`skill_workshop` ausblendet, nennt die Warnung die erste ausschließende Konfigurationsebene und
die genaue erforderliche Änderung an `allow` oder `alsoAllow`. Ältere Betriebsanleitungen verwenden möglicherweise weiterhin
`openclaw plugins inspect skill-workshop`; dieser Befehl erläutert nun, dass Skill
Workshop integriert ist, und gibt gegebenenfalls denselben Richtlinienhinweis aus.

## Verwandte Themen

- [Skills](/de/tools/skills) für Ladereihenfolge, Vorrang und Sichtbarkeit
- [Selbstlernen](/de/tools/self-learning) für konservative Skill-Vorschläge nach einem Durchlauf
- [Skills erstellen](/de/tools/creating-skills) für Grundlagen zu manuell erstellten `SKILL.md`
- [Skills-Konfiguration](/de/tools/skills-config) für das vollständige `skills.workshop`-Schema
- [Skills-CLI](/de/cli/skills) für `openclaw skills`-Befehle
