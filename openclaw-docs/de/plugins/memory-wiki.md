---
read_when:
    - Sie möchten dauerhaftes Wissen, das über einfache MEMORY.md-Notizen hinausgeht
    - Sie konfigurieren das mitgelieferte memory-wiki-Plugin
    - Sie benötigen separate Wiki-Vaults für Agenten in einem Gateway
    - Sie möchten wiki_search, wiki_get oder den Bridge-Modus verstehen
summary: 'memory-wiki: kompilierter Wissensspeicher mit Herkunftsnachweisen, Aussagen, Dashboards und Brückenmodus'
title: Memory-Wiki
x-i18n:
    generated_at: "2026-07-26T18:29:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fda3c801ae39b529a3f1fcaf8791b6dcb1d8116ba2e73e99cca62dca6c64140a
    source_path: plugins/memory-wiki.md
    workflow: 16
---

`memory-wiki` ist ein gebündeltes Plugin, das dauerhaftes Wissen in einem
navigierbaren Wiki zusammenführt: deterministische Seiten, strukturierte Aussagen mit Belegen,
Herkunftsnachweise, Dashboards und maschinenlesbare Zusammenfassungen.

Es ersetzt nicht das Active-Memory-Plugin. Abruf, Übernahme, Indizierung und
Dreaming verbleiben bei dem jeweils konfigurierten Memory-Backend
(`memory-core`, QMD, Honcho usw.). `memory-wiki` wird parallel dazu eingesetzt und führt
Wissen in einer gepflegten Wiki-Ebene zusammen.

Aktivieren Sie das Plugin, bevor Sie seine CLI, Tools oder Laufzeitintegration verwenden:

```bash
openclaw plugins enable memory-wiki
openclaw gateway restart
```

| Ebene                | Zuständig für                                                                     |
| -------------------- | --------------------------------------------------------------------------------- |
| Active-Memory-Plugin | Abruf, semantische Suche, Übernahme, Dreaming, Memory-Laufzeit                    |
| `memory-wiki`        | Zusammengestellte Wiki-Seiten, herkunftsreiche Synthesen, Dashboards, Wiki-Suche/-Abruf/-Anwendung |

Praktische Regel:

- `memory_search` für einen umfassenden Abrufdurchlauf über alle konfigurierten Korpora
- `wiki_search` / `wiki_get`, wenn Sie Wiki-spezifische Rangordnung, Herkunftsnachweise oder eine aussagenbasierte Struktur auf Seitenebene benötigen
- `memory_search corpus=all`, um beide Ebenen in einem Aufruf abzudecken, sofern das Active-Memory-Plugin die Korpusauswahl unterstützt

Eine gängige lokal ausgerichtete Einrichtung: QMD als Active-Memory-Backend für den Abruf und
`memory-wiki` im Modus `bridge` für dauerhafte synthetisierte Seiten. Siehe das
Beispiel für QMD und den Bridge-Modus unter [Konfiguration](#configuration).

Wenn der Bridge-Modus null exportierte Artefakte meldet, stellt das Active-Memory-Plugin
derzeit keine öffentlichen Bridge-Eingaben bereit. Führen Sie zunächst `openclaw wiki doctor` aus
und prüfen Sie anschließend, ob das Active-Memory-Plugin öffentliche Artefakte unterstützt.

## Vault-Modi

- `isolated` (Standard): eigener Vault, eigene Quellen, keine Abhängigkeit vom Active-Memory-Plugin. Verwenden Sie diesen Modus für einen eigenständigen, kuratierten Wissensspeicher.
- `bridge`: liest öffentliche Memory-Artefakte und Ereignisprotokolle des Active-Memory-Plugins über öffentliche Schnittstellen des Plugin-SDK. Verwenden Sie diesen Modus, um die exportierten Artefakte des Memory-Plugins zusammenzustellen, ohne auf private Plugin-Interna zuzugreifen.
- `unsafe-local`: expliziter Ausweg für lokale private Pfade auf demselben Rechner. Bewusst experimentell und nicht portabel; verwenden Sie ihn nur, wenn Sie die Vertrauensgrenze verstehen und ausdrücklich lokalen Dateisystemzugriff benötigen, den der Bridge-Modus nicht bereitstellen kann.

Vault-Modus und Vault-Geltungsbereich sind separate Entscheidungen:

- `vaultMode` bestimmt, woher die Wiki-Eingaben stammen.
- `vault.scope` bestimmt, ob alle Agenten einen Vault verwenden oder jeder Agent einen untergeordneten Vault erhält.

`vault.scope: "global"` ist der Standard und behält das bestehende Verhalten mit einem einzelnen Vault
bei. Verwenden Sie `vault.scope: "agent"` mit dem Modus `isolated` oder `bridge`, wenn
Agenten keine Wiki-Seiten, zusammengestellten Zusammenfassungen, Suchergebnisse oder Schreibvorgänge gemeinsam nutzen dürfen.
Der Agent-Geltungsbereich kann nicht mit dem Modus `unsafe-local` kombiniert werden, da diese konfigurierten
privaten Pfade keine agenteneigenen Eingaben sind. Die Konfigurationsvalidierung weist diese
Kombination zurück.

Der Bridge-Modus kann abhängig vom Konfigurationsschalter `bridge.*` Folgendes indizieren:

- exportierte Memory-Artefakte (`indexMemoryRoot`)
- tägliche Notizen (`indexDailyNotes`)
- Dreaming-Berichte (`indexDreamReports`)
- Memory-Ereignisprotokolle (`followMemoryEvents`)

Wenn der Bridge-Modus aktiv und `bridge.readMemoryArtifacts` aktiviert ist,
werden `openclaw wiki status`, `openclaw wiki doctor` und `openclaw wiki bridge
import` über den laufenden Gateway geleitet, sodass sie denselben Kontext des Active-Memory-
Plugins wie das Agenten-/Laufzeit-Memory verwenden. Wenn die Bridge deaktiviert ist oder
Artefaktlesevorgänge ausgeschaltet sind, behalten diese Befehle ihr lokales/Offline-Verhalten bei.

## Vault-Struktur

```text
<vault>/
  AGENTS.md
  WIKI.md
  index.md
  inbox.md
  entities/
  concepts/
  syntheses/
  sources/
  reports/
  _attachments/
  _views/
  .openclaw-wiki/
```

Verwaltete Inhalte verbleiben in generierten Blöcken; Blöcke mit menschlichen Notizen bleiben
bei der Neugenerierung erhalten.

- `sources/`: importiertes Rohmaterial und durch Bridge/unsichere lokale Quellen gespeiste Seiten
- `entities/`: dauerhafte Dinge, Personen, Systeme, Projekte, Objekte
- `concepts/`: Ideen, Abstraktionen, Muster, Richtlinien (auch das Ziel für OKF-Importe)
- `syntheses/`: zusammengestellte Zusammenfassungen und gepflegte Gesamtübersichten
- `reports/`: generierte Dashboards

## Importe im Open Knowledge Format

```bash
openclaw wiki okf import ./bundles/ga4
```

Importieren Sie ein entpacktes Open-Knowledge-Format-Paket in Wiki-Konzeptseiten. Dies eignet sich
gut, wenn ein Datenkatalog, Dokumentations-Crawler oder Anreicherungsagent bereits
OKF erzeugt: Behalten Sie OKF als portables Austauschartefakt bei und lassen Sie `memory-wiki`
daraus OpenClaw-native Konzeptseiten und zusammengestellte Zusammenfassungen erstellen.

- nicht reservierte `.md`-Dateien sind Konzeptdokumente
- jedes importierte Konzept benötigt ein nicht leeres Frontmatter-Feld `type`; ein fehlendes `type` erzeugt eine Warnung vom Typ `missing-type`, und die Datei wird übersprungen
- unbekannte `type`-Werte werden als generische Konzepte akzeptiert
- `index.md` und `log.md` sind reserviert und werden niemals als Konzepte importiert
- fehlerhafte oder externe Markdown-Links bleiben unverändert

Importierte Seiten werden unter `concepts/` abgelegt, sodass bestehende Abläufe für Zusammenstellung, Suche, Abruf und
Dashboards sie ohne einen zweiten Wiki-Baum erfassen. Jede Seite behält die
ursprüngliche OKF-Konzept-ID, den Quellpfad, `type`, `resource`, `tags`, den Zeitstempel
und das vollständige Frontmatter des Erzeugers. Interne OKF-Links werden auf die generierten
Wiki-Konzeptseiten umgeschrieben und erzeugen außerdem strukturierte `relationships`-Einträge mit
`kind: okf-link`.

## Strukturierte Aussagen und Belege

Seiten enthalten strukturiertes `claims`-Frontmatter, nicht nur Freitext. Jede
Aussage kann `id`, `text`, `status`, `confidence`, `evidence[]` und
`updatedAt` enthalten. Jeder Belegeintrag kann `kind`, `sourceId`, `path`,
`lines`, `weight`, `confidence`, `privacyTier`, `note` und `updatedAt` enthalten.

Dadurch verhält sich das Wiki wie eine Überzeugungsebene und nicht wie eine passive Notizablage.
Aussagen können verfolgt, bewertet, angefochten und bis zu ihren Quellen zurückverfolgt werden.

## Agentenbezogene Entitätsmetadaten

Entitätsseiten enthalten generische Routing-Metadaten, die für Personen, Teams,
Systeme, Projekte oder jeden anderen Entitätstyp verwendet werden können:

- `entityType`: zum Beispiel `person`, `team`, `system`, `project`
- `canonicalId`: stabiler Identitätsschlüssel über Aliasse und Importe hinweg
- `aliases`: Namen, Handles oder Bezeichnungen, die auf dieselbe Seite verweisen
- `privacyTier`: frei formulierbare Zeichenfolge; `public` wird als „keine Prüfung erforderlich“ behandelt, jeder andere Wert (zum Beispiel `local-private`, `sensitive`, `confirm-before-use`) wird in `reports/privacy-review.md` markiert
- `bestUsedFor` / `notEnoughFor`: kompakte Routing-Hinweise
- `lastRefreshedAt`: Zeitstempel der Quellenaktualisierung, getrennt vom Bearbeitungszeitpunkt der Seite
- `personCard`: optionale personenspezifische Routing-Karte (Handles, soziale Profile, E-Mail-Adressen, Zeitzone, Zuständigkeitsbereich, geeignete Anfragen, ungeeignete Anfragen, Konfidenz, Datenschutzstufe)
- `relationships`: typisierte Kanten zu verwandten Seiten (Ziel, Art, Gewichtung, Konfidenz, Belegart, Datenschutzstufe, Notiz)

Beginnen Sie bei einem Personen-Wiki mit `reports/person-agent-directory.md` und öffnen Sie anschließend
die Personenseite mit `wiki_get`, bevor Sie Kontaktdaten oder abgeleitete
Fakten verwenden.

<Accordion title="Beispiel für eine Entitätsseite">
```yaml
pageType: entity
entityType: person
id: entity.example-person
canonicalId: maintainer.example-person
aliases:
  - Alex
  - example-handle
privacyTier: local-private
bestUsedFor:
  - Routing im Beispiel-Ökosystem
notEnoughFor:
  - rechtliche Genehmigung
lastRefreshedAt: "2026-04-29T00:00:00.000Z"
personCard:
  handles:
    - "@example-handle"
  socials:
    - "https://x.example/example-handle"
  emails:
    - alex@example.com
  timezone: America/Chicago
  lane: Beispiel-Ökosystem
  askFor:
    - Fragen zur beispielhaften Einführung
  avoidAskingFor:
    - nicht zugehörige Abrechnungsentscheidungen
  confidence: 0.8
  privacyTier: confirm-before-use
relationships:
  - targetId: entity.other-person
    targetTitle: Andere Person
    kind: collaborates-with
    confidence: 0.7
    evidenceKind: discrawl-stat
claims:
  - id: claim.example.routing
    text: Alex ist für das Routing im Beispiel-Ökosystem hilfreich.
    status: supported
    confidence: 0.9
    evidence:
      - kind: maintainer-whois
        sourceId: source.maintainers
        privacyTier: local-private
```
</Accordion>

## Zusammenstellungspipeline

Die Zusammenstellung liest Wiki-Seiten, normalisiert Zusammenfassungen und speichert einen maschinenorientierten
Snapshot im gemeinsam genutzten SQLite-Plugin-Status von OpenClaw. Der Laufzeitcode verwendet den
lebenszykluseigenen Owner-Snapshot, um SQLite während der asynchronen Prompt-Vorbereitung zu laden;
die synchrone Prompt-Zusammenstellung durchsucht niemals Markdown und liest keine Cache-Dateien.
Die zusammengestellte Ausgabe ermöglicht außerdem die erste Wiki-Indizierung für Suche/Abruf, die
Rückauflösung von Aussage-IDs zu ihren zugehörigen Seiten, kompakte Prompt-Ergänzungen und die
Berichterstellung.

Änderungen an Quellen und Vault-Wiederherstellungen werden erst nach der nächsten
Zusammenstellung maschinenwirksam. Beim Neustarten oder Aktualisieren des Plugin-Lebenszyklus wird die kausal
verkettete Zusammenstellungsveröffentlichung des Vaults mit SQLite verglichen und ein Snapshot aus einem
neueren, zurückgesetzten Zustand abgelehnt. Ein Compiler, der vor dem Rollback gestartet wurde, kann
nicht gegenüber dem wiederhergestellten Vorgänger veröffentlichen. Die Prompt-Vorbereitung fragt den
Vault nicht regelmäßig ab und installiert keine Dateiwächter.
Nach einer Rollback-Quarantäne entfernt eine Zusammenstellung im laufenden Prozess den Owner
sofort; ein separater Compiler-Prozess erfordert eine Aktualisierung des Plugin-Lebenszyklus, damit
der Daemon die neue dauerhafte Veröffentlichung bestätigen kann.
Zusammengestellte Caches können neu erstellt werden: Cache-Zeilen aus Epochen vor der Veröffentlichung werden
als Fehltreffer behandelt und durch die nächste Zusammenstellung ersetzt; sie werden nicht migriert.

## Dashboards und Zustandsberichte

Wenn `render.createDashboards` aktiviert ist, pflegt die Zusammenstellung Dashboards unter
`reports/`:

| Bericht                             | Erfasst                                            |
| ----------------------------------- | -------------------------------------------------- |
| `reports/open-questions.md`         | Seiten mit ungeklärten Fragen                      |
| `reports/contradictions.md`         | Cluster widersprüchlicher Notizen                  |
| `reports/low-confidence.md`         | Seiten und Aussagen mit niedriger Konfidenz        |
| `reports/claim-health.md`           | Aussagen ohne strukturierte Belege                 |
| `reports/stale-pages.md`            | veraltete oder unbekannte Aktualität               |
| `reports/person-agent-directory.md` | Routing-Karten für Personen/Entitäten              |
| `reports/relationship-graph.md`     | strukturierte Beziehungskanten                     |
| `reports/provenance-coverage.md`    | Abdeckung der Belegklassen                         |
| `reports/privacy-review.md`         | nicht öffentliche Datenschutzstufen, die vor der Verwendung geprüft werden müssen |

## Suche und Abruf

Zwei Such-Backends:

- `shared`: verwendet den gemeinsamen Memory-Suchablauf, sofern verfügbar
- `local`: durchsucht das Wiki lokal

Drei Korpora: `wiki`, `memory`, `all`.

- `wiki_search` / `wiki_get` verwenden nach Möglichkeit zusammengestellte Zusammenfassungen für den ersten Durchlauf
- Aussage-IDs werden zur zugehörigen Seite zurückaufgelöst
- angefochtene/veraltete/aktuelle Aussagen beeinflussen die Rangordnung
- Herkunftsbezeichnungen bleiben in den Ergebnissen erhalten

Suchmodi (Parameter `--mode` / Tool `mode`):

| Modus             | Verstärkt                                                        |
| ----------------- | ---------------------------------------------------------------- |
| `auto`            | ausgewogene Standardeinstellung                                  |
| `find-person`     | personenähnliche Entitäten, Aliasse, Handles, soziale Profile, kanonische IDs |
| `route-question`  | Agentenkarten, Hinweise für Nachfragen und optimale Verwendung, Beziehungskontext |
| `source-evidence` | Quellseiten und strukturierte Metadaten zu Nachweisen             |
| `raw-claim`       | Abgleich strukturierter Aussagen; gibt Metadaten zu Aussagen und Nachweisen zurück |

Wenn ein Ergebnis mit einer strukturierten Aussage übereinstimmt, gibt `wiki_search`
`matchedClaimId`, `matchedClaimStatus`, `matchedClaimConfidence`,
`evidenceKinds` und `evidenceSourceIds` in seiner Detailnutzlast zurück. Die Textausgabe
enthält kompakte Zeilen für `Claim:` und `Evidence:`, sofern verfügbar.

## Agentenwerkzeuge

| Werkzeug      | Zweck                                                                                                                                                         |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `wiki_status` | aktueller Vault-Modus und -Umfang, aufgelöster Agent, Zustand, Verfügbarkeit der Obsidian-CLI                                                                  |
| `wiki_search` | Wiki-Seiten und, sofern konfiguriert, den gemeinsamen Erinnerungskorpus durchsuchen; akzeptiert `mode` für Personensuche, Fragenweiterleitung, Quellnachweise oder detaillierte Rohdatenabfragen zu Aussagen |
| `wiki_get`    | eine Wiki-Seite anhand von ID/Pfad lesen; greift auf den gemeinsamen Erinnerungskorpus zurück, wenn die gemeinsame Suche aktiviert ist und die Suche kein Ergebnis liefert |
| `wiki_apply`  | gezielte Synthese-/Metadatenänderungen ohne freie Bearbeitung von Seiten                                                                                       |
| `wiki_lint`   | Strukturprüfungen, Herkunftslücken, Widersprüche, offene Fragen                                                                                                |

Das Plugin registriert außerdem eine nicht exklusive Ergänzung des Erinnerungskorpus, sodass gemeinsame
`memory_search` und `memory_get` auf das Wiki zugreifen können, wenn das aktive Erinnerungs-
Plugin die Korpusauswahl unterstützt.

## Verhalten von Prompt und Kontext

Wenn `context.includeCompiledDigestPrompt` aktiviert ist, hängen Erinnerungsprompt-Abschnitte
einen kompakten kompilierten Schnappschuss aus dem Plugin-Zustand an: nur die wichtigsten Seiten,
nur die wichtigsten Aussagen, Anzahl der Widersprüche, Anzahl der Fragen sowie Einstufungen
zu Konfidenz und Aktualität. Dies ist optional, da es die Prompt-Struktur ändert; relevant ist es hauptsächlich
für Kontext-Engines oder die Prompt-Zusammenstellung, die ausdrücklich Erinnerungsergänzungen
verwenden.

## Konfiguration

Legen Sie die Konfiguration unter `plugins.entries.memory-wiki.config` ab:

```json5
{
  plugins: {
    entries: {
      "memory-wiki": {
        enabled: true,
        config: {
          vaultMode: "isolated",
          vault: {
            scope: "global",
            path: "~/.openclaw/wiki/main",
            renderMode: "obsidian",
          },
          obsidian: {
            enabled: true,
            useOfficialCli: true,
            vaultName: "OpenClaw Wiki",
            openAfterWrites: false,
          },
          bridge: {
            enabled: false,
            readMemoryArtifacts: true,
            indexDreamReports: true,
            indexDailyNotes: true,
            indexMemoryRoot: true,
            followMemoryEvents: true,
          },
          unsafeLocal: {
            allowPrivateMemoryCoreAccess: false,
            paths: [],
          },
          ingest: {
            autoCompile: true,
            maxConcurrentJobs: 1,
            allowUrlIngest: true,
          },
          search: {
            backend: "shared",
            corpus: "wiki",
          },
          context: {
            includeCompiledDigestPrompt: false,
          },
          render: {
            preserveHumanBlocks: true,
            createBacklinks: true,
            createDashboards: true,
          },
        },
      },
    },
  },
}
```

Wichtige Umschalter:

| Schlüssel                                  | Werte / Standard                               | Hinweise                                                                      |
| ------------------------------------------ | ---------------------------------------------- | ----------------------------------------------------------------------------- |
| `vaultMode`                                | `isolated` (Standard), `bridge`, `unsafe-local` | wählt Eingabe- und Integrationsverhalten                                      |
| `vault.scope`                              | `global` (Standard), `agent`                    | ein gemeinsamer Vault oder ein untergeordneter Vault pro Agent                |
| `vault.path`                               | globaler Standard `~/.openclaw/wiki/main`      | exakter globaler Vault; das übergeordnete Verzeichnis im Agentenumfang ist standardmäßig `~/.openclaw/wiki` |
| `vault.renderMode`                         | `native` (Standard), `obsidian`                 |                                                                               |
| `bridge.readMemoryArtifacts`               | Standard `true`                            | öffentliche Artefakte des aktiven Erinnerungs-Plugins importieren             |
| `bridge.followMemoryEvents`                | Standard `true`                            | Ereignisprotokolle im Bridge-Modus einbeziehen                                |
| `unsafeLocal.allowPrivateMemoryCoreAccess` | Standard `false`                           | erforderlich, um Importe mit `unsafe-local` auszuführen                      |
| `unsafeLocal.paths`                        | Standard `[]`                              | explizite lokale Pfade für den Import im Modus `unsafe-local`                |
| `search.backend`                           | `shared` (Standard), `local`                    |                                                                               |
| `search.corpus`                            | `wiki` (Standard), `memory`, `all`              |                                                                               |
| `context.includeCompiledDigestPrompt`      | Standard `false`                           | kompakten Digest-Schnappschuss des ausgewählten Agenten an Erinnerungsprompt-Abschnitte anhängen |
| `render.createBacklinks`                   | Standard `true`                            | deterministische Blöcke mit zugehörigen Inhalten erzeugen                     |
| `render.createDashboards`                  | Standard `true`                            | Dashboard-Seiten erzeugen                                                     |

### Vaults pro Agent

Setzen Sie `vault.scope` auf `agent`, um jedem konfigurierten Agenten ein eigenes Wiki zuzuweisen.
In diesem Umfang ist `vault.path` ein übergeordnetes Verzeichnis, und OpenClaw hängt die
normalisierte Agenten-ID an:

```json5
{
  agents: {
    list: [{ id: "support" }, { id: "marketing" }],
  },
  plugins: {
    entries: {
      "memory-wiki": {
        enabled: true,
        config: {
          vaultMode: "bridge",
          vault: {
            scope: "agent",
            path: "~/.openclaw/wiki",
          },
          bridge: {
            enabled: true,
            readMemoryArtifacts: true,
          },
        },
      },
    },
  },
}
```

Dies wird zu `~/.openclaw/wiki/support` und
`~/.openclaw/wiki/marketing` aufgelöst. Wenn `vault.path` im Agentenumfang weggelassen wird, ist das
übergeordnete Verzeichnis standardmäßig `~/.openclaw/wiki`. Der standardmäßige Agent `main` behält daher
den vorhandenen Pfad `~/.openclaw/wiki/main` bei.

Agentenwerkzeuge, kompilierte Prompt-Digests und die über
`memory_search` / `memory_get` bereitgestellte Wiki-Ergänzung lösen den Vault anhand des aktiven Agentenkontexts auf.
Geben Sie bei CLI- und Gateway-Aufrufen in einer Konfiguration mit mehreren konfigurierten Agenten
den Agenten explizit mit `openclaw wiki --agent <agentId> ...` oder über `agentId` der Gateway-
Anfrage an. Ein einzelner konfigurierter Agent bleibt die Standardeinstellung, wenn keine ID
angegeben wird.

Im Bridge-Modus akzeptieren agentenspezifische Importe ein öffentliches Erinnerungsartefakt nur, wenn
dessen `agentIds` den ausgewählten Agenten enthält. Artefakte, die einem anderen Agenten gehören,
keine Eigentumsmetadaten enthalten oder einen unbekannten Eigentümer haben, werden übersprungen. Der globale Umfang
behält das vorhandene Verhalten für gemeinsame Artefakte bei.

<Warning>
Eine Änderung von `vault.scope` kopiert oder teilt einen vorhandenen Vault nicht auf. Im Agentenumfang
wird ein explizit konfigurierter `vault.path` zu einem übergeordneten Verzeichnis. Verschieben oder
importieren Sie daher vorhandene Seiten bewusst, bevor Sie produktive Agenten umstellen. Sichern Sie
zuerst den Vault.

Vaults pro Agent bilden eine Wissensgrenze innerhalb desselben Prozesses, keine Sicherheitsgrenze
des Betriebssystems. Plugins und nicht isolierte Werkzeuge mit Zugriff auf das Host-Dateisystem können
weiterhin das Verzeichnis eines anderen Agenten lesen. Verwenden Sie [Sandboxing](/de/gateway/sandboxing) oder
[separate Gateway-Profile](/de/gateway/multiple-gateways), wenn Agenten einander nicht
vertrauen.
</Warning>

### Beispiel: QMD + Bridge-Modus

Verwenden Sie dies, wenn Sie QMD für den Abruf und `memory-wiki` als gepflegte
Wissensebene einsetzen möchten. Jede Ebene bleibt fokussiert: QMD hält Rohnotizen, Sitzungs-
exporte und zusätzliche Sammlungen durchsuchbar, während `memory-wiki`
stabile Entitäten, Aussagen, Dashboards und Quellseiten kompiliert.

```json5
{
  memory: {
    backend: "qmd",
  },
  plugins: {
    entries: {
      "memory-wiki": {
        enabled: true,
        config: {
          vaultMode: "bridge",
          bridge: {
            enabled: true,
            readMemoryArtifacts: true,
            indexDreamReports: true,
            indexDailyNotes: true,
            indexMemoryRoot: true,
            followMemoryEvents: true,
          },
          search: {
            backend: "shared",
            corpus: "all",
          },
          context: {
            includeCompiledDigestPrompt: false,
          },
        },
      },
    },
  },
}
```

Dadurch bleibt QMD für den Abruf des Active Memory zuständig, `memory-wiki` konzentriert sich auf
kompilierte Seiten und Dashboards, und die Prompt-Struktur bleibt unverändert, bis Sie
kompilierte Digest-Prompts bewusst aktivieren.

## CLI

```bash
openclaw wiki status
openclaw wiki doctor
openclaw wiki init
openclaw wiki ingest ./notes/alpha.md
openclaw wiki compile
openclaw wiki lint
openclaw wiki search "alpha"
openclaw wiki get entity.alpha
openclaw wiki apply synthesis "Alpha Summary" --body "..." --source-id source.alpha
openclaw wiki bridge import
openclaw wiki obsidian status
```

Die vollständige Befehlsreferenz einschließlich
`wiki okf import`, `wiki apply metadata`, `wiki unsafe-local import`,
`wiki chatgpt import` / `wiki chatgpt rollback` und des vollständigen `wiki obsidian`-
Unterbefehlssatzes finden Sie unter [CLI: Wiki](/de/cli/wiki).

## Obsidian-Unterstützung

Wenn `vault.renderMode` auf `obsidian` gesetzt ist, schreibt das Plugin Obsidian-freundliches
Markdown und kann optional die offizielle `obsidian`-CLI für Zustands-
prüfungen, Vault-Suchen, das Öffnen einer Seite, das Aufrufen eines Befehls und das Wechseln zur
Tagesnotiz verwenden. Dies ist optional; das Wiki funktioniert auch im nativen Modus ohne
Obsidian.

Agentenspezifische Vaults können weiterhin Obsidian-freundliches Markdown verwenden, aber die Konfigurations-
validierung lehnt `obsidian.useOfficialCli: true` mit `vault.scope: "agent"` ab.
Die aktuelle Einstellung `obsidian.vaultName` ist global und kann nicht für jeden Agenten
einen eigenen Obsidian-Vault auswählen. Verwenden Sie stattdessen die Wiki-Werkzeuge und CLI-Vorgänge,
oder belassen Sie ein von Obsidian verwaltetes Wiki im globalen Umfang.

## Empfohlener Arbeitsablauf

<Steps>
<Step title="Aktives Memory-Plugin für den Abruf beibehalten">
Abruf, Hochstufung und Dreaming bleiben dem konfigurierten Memory-Backend zugeordnet.
</Step>
<Step title="memory-wiki aktivieren">
Beginnen Sie mit dem Modus `isolated`, sofern Sie nicht ausdrücklich den Bridge-Modus verwenden möchten.
</Step>
<Step title="wiki_search / wiki_get verwenden, wenn die Provenienz wichtig ist">
Ziehen Sie diese `memory_search` vor, wenn Sie Wiki-spezifisches Ranking oder eine Glaubwürdigkeitsstruktur auf Seitenebene benötigen.
</Step>
<Step title="wiki_apply für eng begrenzte Synthesen oder Metadatenaktualisierungen verwenden">
Vermeiden Sie die manuelle Bearbeitung verwalteter generierter Blöcke.
</Step>
<Step title="wiki_lint nach wesentlichen Änderungen ausführen">
Erkennt Widersprüche, offene Fragen und Provenienzlücken.
</Step>
<Step title="Dashboards für die Sichtbarkeit veralteter Inhalte und von Widersprüchen aktivieren">
Legen Sie `render.createDashboards: true` fest (Standard).
</Step>
</Steps>

## Zugehörige Dokumentation

- [Memory-Übersicht](/de/concepts/memory)
- [CLI: Memory](/de/cli/memory)
- [CLI: Wiki](/de/cli/wiki)
- [Übersicht über das Plugin SDK](/de/plugins/sdk-overview)
