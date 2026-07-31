---
read_when:
    - Sie möchten semantischen Speicher indizieren oder durchsuchen
    - Sie debuggen die Speicherverfügbarkeit oder Indizierung
    - Sie möchten abgerufene Kurzzeiterinnerungen in `MEMORY.md` überführen
summary: CLI-Referenz für `openclaw memory` (status/index/search/promote/promote-explain/rem-harness/rem-backfill)
title: Speicher
x-i18n:
    generated_at: "2026-07-26T18:17:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6354745f8622ee80345325fa6f3e7d6c5f280cb63b9cdb100a766cf9e300af59
    source_path: cli/memory.md
    workflow: 16
---

# `openclaw memory`

Verwalten Sie die semantische Speicherindizierung, die Suche und die Übernahme in `MEMORY.md`.
Bereitgestellt durch das gebündelte `memory-core`-Plugin und verfügbar, wenn
`plugins.slots.memory` den Wert `memory-core` auswählt (die Standardeinstellung). Andere Speicher-
Plugins stellen eigene CLI-Namensräume bereit.

Verwandte Themen: Konzept [Speicher](/de/concepts/memory), [Dreaming](/de/concepts/dreaming),
[Referenz zur Speicherkonfiguration](/de/reference/memory-config), [Speicher-Wiki](/de/plugins/memory-wiki),
[Wiki](/de/cli/wiki), [Plugins](/de/tools/plugin).

## `memory status`

```bash
openclaw memory status [--agent <id>] [--deep] [--index] [--fix] [--json] [--verbose]
```

Ohne `--agent` wird der Befehl für jeden Agenten in `agents.entries` ausgeführt; wenn keine Agentenliste
konfiguriert ist, wird auf den Standardagenten zurückgegriffen.

| Flag        | Wirkung                                                                                                                                                                                                                                                                                                    |
| ----------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--deep`    | Prüft die Bereitschaft des Vektorspeichers, des Embedding-Providers und der semantischen Suche (bedingt zusätzliche Provider-Aufrufe). Der einfache Aufruf `memory status` bleibt schnell und überspringt diese Prüfung; ein unbekannter Vektor-/Semantikstatus bedeutet, dass er nicht geprüft wurde. Der lexikalische QMD-Aufruf `searchMode: "search"` überspringt semantische Vektorprüfungen immer, selbst mit `--deep`. |
| `--index`   | Indiziert erneut, wenn der Speicher nicht aktuell ist. Impliziert `--deep`.                                                                                                                                                                                                                                                          |
| `--fix`     | Repariert veraltete Recall-Sperren und normalisiert Metadaten zur Übernahme.                                                                                                                                                                                                                                               |
| `--json`    | Gibt JSON aus.                                                                                                                                                                                                                                                                                               |
| `--verbose` | Gibt detaillierte Protokolle für jede Phase aus.                                                                                                                                                                                                                                                                             |

Wenn die Zeile `Dreaming` selbst mit `dreaming.enabled: true` weiterhin `off` anzeigt oder
geplante Durchläufe offenbar nie ausgeführt werden, ist der verwaltete Dreaming-Cron davon abhängig, dass der
Heartbeat des Standardagenten ausgelöst wird, um den Abgleich anzustoßen. Weitere Informationen zur Planung finden Sie unter
[Dreaming](/de/concepts/dreaming).

Der Status führt außerdem alle zusätzlichen Suchpfade aus `memory.search.extraPaths` auf.

## `memory index`

```bash
openclaw memory index [--agent <id>] [--force] [--verbose]
```

Es gilt dieselbe agentenspezifische Eingrenzung wie bei `status`. `--force` führt statt einer
inkrementellen Indizierung eine vollständige Neuindizierung aus. `--verbose` zeigt Provider, Modell, Quellen und
Details zu zusätzlichen Pfaden je Agent an, bevor der Indizierungsfortschritt ausgegeben wird.

## `memory search`

```bash
openclaw memory search [query] [--query <text>] [--agent <id>] [--max-results <n>] [--min-score <n>] [--json]
```

- Abfrage: positionsgebundenes `[query]` oder `--query <text>`. Wenn beide angegeben sind, hat `--query`
  Vorrang. Wenn keines angegeben ist, schlägt der Befehl fehl.
- `--agent <id>`: Verwendet standardmäßig den Standardagenten (nicht die vollständige Agentenliste).
- `--max-results <n>`: Begrenzt die Anzahl der Ergebnisse (positive Ganzzahl).
- `--min-score <n>`: Filtert Treffer unterhalb dieses Werts heraus.

## `memory promote`

Ordnet Kurzzeitkandidaten aus `memory/YYYY-MM-DD.md` und hängt optional die
bestplatzierten Einträge an `MEMORY.md` an.

```bash
openclaw memory promote [--agent <id>] [--limit <n>] [--min-score <n>] \
  [--min-recall-count <n>] [--min-unique-queries <n>] [--apply] [--include-promoted] [--json]
```

| Flag                       | Standardwert      | Wirkung                                                            |
| -------------------------- | ------------ | ----------------------------------------------------------------- |
| `--limit <n>`              |              | Maximale Anzahl zurückzugebender/anzuwendender Kandidaten.                                   |
| `--min-score <n>`          | `0.75`       | Minimaler gewichteter Übernahmewert.                                 |
| `--min-recall-count <n>`   | `3`          | Erforderliche Mindestanzahl an Recalls.                                    |
| `--min-unique-queries <n>` | `2`          | Erforderliche Mindestanzahl unterschiedlicher Abfragen.                            |
| `--apply`                  | nur Vorschau | Hängt ausgewählte Kandidaten an `MEMORY.md` an und markiert sie als übernommen. |
| `--include-promoted`       |              | Schließt Kandidaten ein, die bereits in früheren Zyklen übernommen wurden.           |
| `--json`                   |              | Gibt JSON aus.                                                       |

Diese CLI-Standardwerte unterscheiden sich von den Schwellenwerten der Tiefenphase
des geplanten Dreaming-Durchlaufs (siehe [Dreaming](#dreaming) unten). Geben Sie explizite Flags an, um
bei einem einmaligen manuellen Lauf das Verhalten des Durchlaufs zu erreichen.

Bewertungssignale: Recall-Häufigkeit, Relevanz beim Abruf, Abfragevielfalt,
zeitliche Aktualität, Konsolidierung über mehrere Tage und Reichhaltigkeit abgeleiteter Konzepte. Sie stammen
sowohl aus Speicher-Recalls als auch aus täglichen Aufnahmedurchläufen und umfassen zudem eine leichte Verstärkung
in der Leicht-/REM-Phase bei wiederholten Dreaming-Durchläufen. Vor dem Schreiben
liest die Übernahme die aktuelle Tagesnotiz erneut ein, sodass Änderungen oder Löschungen an Kurzzeitfragmenten
seit der Bewertung berücksichtigt werden, anstatt Inhalte aus einem veralteten Snapshot zu übernehmen.

## `memory promote-explain`

Erläutert die Aufschlüsselung des Werts eines Übernahmekandidaten.

```bash
openclaw memory promote-explain <selector> [--agent <id>] [--include-promoted] [--json]
```

`<selector>` stimmt mit dem Schlüssel eines Kandidaten (exakt oder als Teilzeichenfolge), seinem Pfad oder dem Text
seines Fragments überein.

## `memory rem-harness`

Zeigt eine Vorschau auf REM-Reflexionen, Wahrheitskandidaten und die Ausgabe der Tiefenphase,
ohne etwas zu schreiben.

```bash
openclaw memory rem-harness [--agent <id>] [--path <file-or-dir>] [--grounded] [--include-promoted] [--json]
```

- `--path <file-or-dir>`: Initialisiert das Testsystem anhand historischer täglicher Dateien aus `YYYY-MM-DD.md`
  anstelle des aktuellen Arbeitsbereichs.
- `--grounded`: Rendert zusätzlich eine fundierte Vorschau von `What Happened` / `Reflections` /
  `Possible Lasting Updates` aus den historischen Notizen.

## `memory rem-backfill`

Schreibt fundierte historische REM-Zusammenfassungen zur Überprüfung in der Benutzeroberfläche in `DREAMS.md`.
Umkehrbar.

```bash
openclaw memory rem-backfill --path <file-or-dir> [--agent <id>] [--stage-short-term] [--json]
openclaw memory rem-backfill --rollback [--rollback-short-term] [--json]
```

- `--path <file-or-dir>`: Erforderlich, sofern `--rollback`/`--rollback-short-term`
  nicht angegeben ist. Historische tägliche Speicherdatei(en) oder das Verzeichnis, aus denen die Nachbefüllung erfolgt.
- `--stage-short-term`: Übernimmt außerdem fundierte dauerhafte Kandidaten in den aktiven
  Kurzzeit-Übernahmespeicher, damit sie von der normalen Tiefenphase bewertet werden können.
- `--rollback`: Entfernt zuvor geschriebene fundierte Tagebucheinträge aus
  `DREAMS.md`.
- `--rollback-short-term`: Entfernt zuvor bereitgestellte fundierte Kurzzeit-
  kandidaten.

## Dreaming

Dreaming ist das System zur Speicherkonsolidierung im Hintergrund mit drei zusammenwirkenden
Phasen, die nach einem gemeinsamen Zeitplan in dieser Reihenfolge ausgeführt werden: **leicht** (Kurzzeit-
material sortieren/bereitstellen), **REM** (reflektieren und Themen hervorheben), **tief** (dauerhafte
Fakten in `MEMORY.md` übernehmen). Nur die Tiefenphase schreibt in `MEMORY.md`.

- Aktivieren Sie es mit `plugins.entries.memory-core.config.dreaming.enabled: true`
  (Standardwert: `false`); `memory-core` verwaltet den Cron-Job des Durchlaufs automatisch, ein manueller
  Aufruf von `openclaw cron add` ist nicht erforderlich.
- Schalten Sie es im Chat mit `/dreaming on|off` um; prüfen Sie es mit `/dreaming status`
  (oder `/dreaming`/`/dreaming help`). `on`/`off` erfordert den Status als Kanalinhaber
  oder Gateway-`operator.admin`; `status` und die Hilfe bleiben für alle verfügbar, die
  den Befehl aufrufen können.
- Die menschenlesbare Phasenausgabe wird in `DREAMS.md` (oder eine vorhandene Datei `dreams.md`) geschrieben.
  Standardmäßig (`dreaming.storage.mode: "separate"`) schreibt jede Phase außerdem einen
  eigenständigen Bericht nach `memory/dreaming/<phase>/YYYY-MM-DD.md`; setzen Sie `mode:
"inline"`, um Berichte stattdessen in die tägliche Speicherdatei zu integrieren, oder `"both"`
  für beides.
- Geplante und manuelle Ausführungen von `memory promote` verwenden dieselben Bewertungssignale
  der Tiefenphase; nur die Standardschwellenwerte unterscheiden sich (siehe Tabelle oben und
  die geplanten Standardwerte unten).
- Geplante Ausführungen werden auf die Speicherarbeitsbereiche aller konfigurierten Agenten verteilt.

Geplante Standardwerte (`plugins.entries.memory-core.config.dreaming`):

| Schlüssel                                    | Standardwert     |
| -------------------------------------- | ----------- |
| `frequency`                            | `0 3 * * *` |
| `phases.deep.minScore`                 | `0.8`       |
| `phases.deep.minRecallCount`           | `3`         |
| `phases.deep.minUniqueQueries`         | `3`         |
| `phases.deep.recencyHalfLifeDays`      | `14`        |
| `phases.deep.maxAgeDays`               | `30`        |
| `phases.deep.maxPromotedSnippetTokens` | `160`       |

```json
{
  "plugins": {
    "entries": {
      "memory-core": {
        "config": {
          "dreaming": {
            "enabled": true
          }
        }
      }
    }
  }
}
```

Vollständige Schlüsselliste und Phasendetails: [Dreaming](/de/concepts/dreaming),
[Referenz zur Speicherkonfiguration](/de/reference/memory-config#dreaming).

## SecretRef-Gateway-Abhängigkeit

Wenn Remote-API-Schlüsselfelder von Active Memory als SecretRefs konfiguriert sind, lösen `memory`-
Befehle sie anhand des aktiven Gateway-Snapshots auf. Ist das Gateway
nicht verfügbar, schlägt der Befehl sofort fehl. Dies erfordert ein Gateway, das die Methode
`secrets.resolve` unterstützt; ältere Gateways geben einen Fehler für eine unbekannte Methode zurück.

## Verwandte Themen

- [CLI-Referenz](/de/cli)
- [Speicherübersicht](/de/concepts/memory)
