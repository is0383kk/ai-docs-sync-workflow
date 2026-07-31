---
read_when:
    - Die Übernahme in den Speicher soll automatisch erfolgen
    - Sie möchten verstehen, was jede Dreaming-Phase bewirkt
    - Sie möchten die Konsolidierung optimieren, ohne MEMORY.md zu überfrachten
sidebarTitle: Dreaming
summary: Hintergrundkonsolidierung des Gedächtnisses mit Leicht-, Tief- und REM-Phasen sowie einem Traumtagebuch
title: Dreaming
x-i18n:
    generated_at: "2026-07-26T17:47:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 501ab42cfdfa0216c308896aa8c1719b06b49d64a62afdb004e097102a376eac
    source_path: concepts/dreaming.md
    workflow: 16
---

Dreaming ist das System zur Hintergrundkonsolidierung des Gedächtnisses in `memory-core`. Es überführt starke Kurzzeitsignale in dauerhafte Erinnerungen und hält den Prozess dabei erklärbar und überprüfbar.

<Note>
Dreaming ist **optional aktivierbar** und standardmäßig deaktiviert.
</Note>

## Was Dreaming schreibt

- **Maschinenzustand** in `memory/.dreams/` (Abrufspeicher, Phasensignale, Aufnahmeprüfpunkte, Sperren).
- **Für Menschen lesbare Ausgabe** in `DREAMS.md` (oder einer vorhandenen `dreams.md`) und optionale Phasenberichtsdateien unter `memory/dreaming/<phase>/YYYY-MM-DD.md`.

Die langfristige Übernahme schreibt weiterhin ausschließlich nach `MEMORY.md`.

## Phasenmodell

Dreaming führt pro Durchlauf drei kooperative Phasen in dieser Reihenfolge aus: leicht -> REM -> tief. Dies sind interne Implementierungsphasen, keine separat von Benutzern konfigurierten Modi.

| Phase  | Zweck                                           | Dauerhafter Schreibvorgang |
| ------ | ----------------------------------------------- | -------------------------- |
| Leicht | Aktuelles Kurzzeitmaterial sortieren und bündeln | Nein                       |
| REM    | Themen und wiederkehrende Ideen reflektieren    | Nein                       |
| Tief   | Dauerhafte Kandidaten bewerten und übernehmen   | Ja (`MEMORY.md`)    |

<AccordionGroup>
  <Accordion title="Leichtphase">
    - Liest den aktuellen Zustand des Kurzzeit-Abrufs, tägliche Gedächtnisdateien und, sofern verfügbar, redigierte Sitzungstranskripte.
    - Dedupliziert Signale und bündelt Kandidatenzeilen.
    - Schreibt einen verwalteten `## Light Sleep`-Block, wenn der Speicher Inline-Ausgaben umfasst.
    - Zeichnet Verstärkungssignale für die spätere Rangfolge in der Tiefphase auf.
    - Schreibt niemals nach `MEMORY.md`.

  </Accordion>
  <Accordion title="REM-Phase">
    - Erstellt Themen- und Reflexionszusammenfassungen aus aktuellen Kurzzeitspuren.
    - Schreibt einen verwalteten `## REM Sleep`-Block, wenn der Speicher Inline-Ausgaben umfasst.
    - Zeichnet REM-Verstärkungssignale auf, die von der Rangfolge in der Tiefphase verwendet werden.
    - Schreibt niemals nach `MEMORY.md`.

  </Accordion>
  <Accordion title="Tiefphase">
    - Ordnet Kandidaten anhand gewichteter Bewertungen und Schwellenwertprüfungen ein (`minScore`, `minRecallCount`, `minUniqueQueries` müssen alle bestanden werden).
    - Rehydriert Ausschnitte vor dem Schreiben aus aktuellen Tagesdateien, sodass veraltete oder gelöschte Ausschnitte übersprungen werden.
    - Hängt übernommene Einträge an `MEMORY.md` an.
    - Schreibt eine `## Deep Sleep`-Zusammenfassung nach `DREAMS.md` und optional nach `memory/dreaming/deep/YYYY-MM-DD.md`.

  </Accordion>
</AccordionGroup>

## Aufnahme von Sitzungstranskripten

Dreaming kann redigierte Sitzungstranskripte in den Dreaming-Korpus aufnehmen. Sofern verfügbar, fließen Transkripte zusammen mit täglichen Gedächtnissignalen und Abrufspuren in die Leichtphase ein. Persönliche und vertrauliche Inhalte werden vor der Aufnahme redigiert.

## Traumtagebuch

Dreaming führt ein erzählerisches **Traumtagebuch** in `DREAMS.md`. Sobald jede Phase über genügend Material verfügt, führt `memory-core` nach bestem Bemühen einen Hintergrunddurchlauf eines Subagenten aus und hängt einen kurzen Tagebucheintrag an. Dabei wird das standardmäßige Laufzeitmodell verwendet, sofern `dreaming.model` nicht konfiguriert ist. Wenn das konfigurierte Modell nicht verfügbar ist, wird der Tagebuchdurchlauf einmal mit dem Standardmodell der Sitzung wiederholt. Fehler bei Vertrauensprüfung oder Zulassungsliste werden nicht erneut versucht und bleiben in den Protokollen sichtbar, statt stillschweigend auf einen generischen Tagebucheintrag zurückzufallen.

<Note>
Das Tagebuch ist für die menschliche Lektüre in der Träume-Benutzeroberfläche vorgesehen und dient nicht als Quelle für Übernahmen. Tagebuch- und Berichtsartefakte sind von der Kurzzeitübernahme ausgeschlossen; nur fundierte Gedächtnisausschnitte können nach `MEMORY.md` übernommen werden.
</Note>

Für Prüfungs- und Wiederherstellungsarbeiten gibt es außerdem einen fundierten historischen Rückfüllpfad:

<AccordionGroup>
  <Accordion title="Rückfüllbefehle">
    - `memory rem-harness --path ... --grounded` zeigt eine Vorschau fundierter Tagebuchausgaben aus historischen `YYYY-MM-DD.md`-Notizen.
    - `memory rem-backfill --path ...` schreibt rückgängig machbare fundierte Tagebucheinträge nach `DREAMS.md`.
    - `memory rem-backfill --path ... --stage-short-term` bündelt fundierte dauerhafte Kandidaten in demselben Kurzzeit-Beweisspeicher, den die normale Tiefphase verwendet.
    - `memory rem-backfill --rollback` und `--rollback-short-term` entfernen diese gebündelten Rückfüllartefakte, ohne gewöhnliche Tagebucheinträge oder den aktuellen Kurzzeit-Abruf zu verändern.

  </Accordion>
</AccordionGroup>

Die Control UI stellt denselben Ablauf zum Rückfüllen und Zurücksetzen des Tagebuchs auf dem Tab „Gedächtnis“ des Agenten (Seite „Agenten“) bereit, sodass Sie die Ergebnisse in der Traumszene prüfen können, bevor Sie entscheiden, ob fundierte Kandidaten eine Übernahme verdienen. Ein eigener fundierter Szenenpfad zeigt, welche gebündelten Kurzzeiteinträge aus der historischen Wiedergabe stammen und welche übernommenen Elemente hauptsächlich auf fundierten Daten basieren. Außerdem können Sie damit ausschließlich fundierte gebündelte Einträge löschen, ohne den aktuellen Kurzzeitzustand zu verändern.

## Signale für die Rangfolge in der Tiefphase

Die Rangfolge in der Tiefphase verwendet sechs gewichtete Basissignale sowie die Phasenverstärkung:

| Signal                | Gewichtung | Beschreibung                                               |
| --------------------- | ---------- | ---------------------------------------------------------- |
| Relevanz              | 0.30       | Durchschnittliche Abrufqualität des Eintrags               |
| Häufigkeit            | 0.24       | Anzahl der vom Eintrag gesammelten Kurzzeitsignale         |
| Abfragevielfalt       | 0.15       | Unterschiedliche Abfrage-/Tageskontexte, in denen er auftrat |
| Aktualität            | 0.15       | Zeitlich abnehmende Aktualitätsbewertung                   |
| Konsolidierung        | 0.10       | Stärke der Wiederholung über mehrere Tage                  |
| Begrifflicher Gehalt | 0.06       | Dichte der Konzept-Tags aus Ausschnitt/Pfad                |

Treffer in der Leicht- und REM-Phase fügen eine kleine, mit der Zeit abnehmende Verstärkung aus `memory/.dreams/phase-signals.json` hinzu.

Ergebnisse von Schattenversuchen können vor jedem dauerhaften Schreibvorgang als Prüfsignal auf die Basisbewertung aufgesetzt werden: Ein hilfreicher Versuch verleiht einem Kandidaten eine kleine begrenzte Verstärkung, ein neutraler Versuch hält ihn zurückgestellt und ein schädlicher Versuch kennzeichnet ihn für diesen Bewertungsdurchlauf als abgelehnt. Dieses Signal dient ausschließlich Berichten – es kann die Reihenfolge der Kandidaten oder Prüfmetadaten ändern, schreibt jedoch niemals nach `MEMORY.md` und übernimmt keinen Kandidaten selbstständig.

### Berichtsabdeckung für QA-Schattenversuche

QA Lab enthält ein ausschließlich der Berichterstellung dienendes Szenario, mit dem untersucht werden kann, wie ein zukünftiger Dreaming-Schattenversuch eine Kandidatenerinnerung vor der Übernahme prüfen könnte: Ein Agent vergleicht eine Basisantwort mit einer Antwort, die die Kandidatenerinnerung verwenden kann, und schreibt anschließend einen lokalen Bericht mit Urteil, Begründung und Risikokennzeichnungen. Diese Abdeckung ist auf QA beschränkt – sie überprüft, dass das Berichtsartefakt von `MEMORY.md` getrennt bleibt und der Agent niemals behauptet, der Kandidat sei übernommen worden. Sie fügt kein produktives Schattenversuchsverhalten hinzu und ändert die Übernahme-Engine der Tiefphase nicht.

Der Schattenversuchs-Runner `memory-core` behält denselben ausschließlich der Berichterstellung dienenden Vertrag für Codepfade bei, die ein stabiles Artefakt benötigen. Er akzeptiert den Kandidaten, die Versuchsaufforderung, das Basisergebnis, das Kandidatenergebnis, das Urteil, die Begründung, die Risikokennzeichnungen und die Belegreferenzen und schreibt anschließend mit `promotion action: report-only` einen Bericht. Hilfreiche Urteile werden einer `promote`-Empfehlung zugeordnet, neutrale Urteile `defer` und schädliche Urteile `reject` – keine dieser Aktionen schreibt nach `MEMORY.md` oder wendet eine Übernahme durch die Tiefphase an.

## Zeitplanung

Wenn aktiviert, verwaltet `memory-core` automatisch einen Cron-Job für einen vollständigen Dreaming-Durchlauf. Dieser wird über den primären Laufzeit-Workspace und alle konfigurierten Agenten-Workspaces hinweg dedupliziert, damit die Auffächerung der Subagenten-Workspaces `DREAMS.md` und den Gedächtniszustand des Hauptagenten nicht ausschließt.

| Einstellung          | Standardwert     |
| -------------------- | ---------------- |
| `dreaming.frequency` | `0 3 * * *` |
| `dreaming.model` | Standardmodell   |

## Schnellstart

<Tabs>
  <Tab title="Dreaming aktivieren">
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
  </Tab>
  <Tab title="Benutzerdefiniertes Durchlaufintervall">
    ```json
    {
      "plugins": {
        "entries": {
          "memory-core": {
            "config": {
              "dreaming": {
                "enabled": true,
                "timezone": "America/Los_Angeles",
                "frequency": "0 */6 * * *"
              }
            }
          }
        }
      }
    }
    ```
  </Tab>
</Tabs>

## Slash-Befehl

```text
/dreaming status
/dreaming on
/dreaming off
/dreaming help
```

`/dreaming on` und `/dreaming off` erfordern für Kanalaufrufer den Eigentümerstatus oder `operator.admin` für Gateway-Clients. `/dreaming status` und `/dreaming help` sind schreibgeschützt.

## CLI-Arbeitsablauf

<Tabs>
  <Tab title="Übernahmevorschau/-anwendung">
    ```bash
    openclaw memory promote
    openclaw memory promote --apply
    openclaw memory promote --limit 5
    openclaw memory status --deep
    ```

    Manuelles `memory promote` verwendet standardmäßig die Schwellenwerte der Tiefphase, sofern diese nicht mit CLI-Flags überschrieben werden.

  </Tab>
  <Tab title="Übernahme erläutern">
    Erläutern Sie, warum ein bestimmter Kandidat übernommen oder nicht übernommen würde:

    ```bash
    openclaw memory promote-explain "router vlan"
    openclaw memory promote-explain "router vlan" --json
    ```

  </Tab>
  <Tab title="Vorschau des REM-Testsystems">
    Zeigen Sie eine Vorschau der REM-Reflexionen, Kandidatenwahrheiten und Ergebnisse der Tiefenübernahme an, ohne etwas zu schreiben:

    ```bash
    openclaw memory rem-harness
    openclaw memory rem-harness --json
    ```

  </Tab>
</Tabs>

## Wichtige Standardwerte

Alle Einstellungen befinden sich unter `plugins.entries.memory-core.config.dreaming`.

<ParamField path="enabled" type="boolean" default="false">
  Aktiviert oder deaktiviert den Dreaming-Durchlauf.
</ParamField>
<ParamField path="frequency" type="string" default="0 3 * * *">
  Cron-Intervall für den vollständigen Dreaming-Durchlauf.
</ParamField>
<ParamField path="model" type="string">
  Optionale Modellüberschreibung für den Subagenten des Traumtagebuchs. Verwenden Sie einen kanonischen `provider/model`-Wert, wenn Sie außerdem eine `allowedModels`-Zulassungsliste für Subagenten festlegen.
</ParamField>
<ParamField path="phases.deep.maxPromotedSnippetTokens" type="number" default="160">
  Maximale geschätzte Tokenanzahl, die aus jedem nach `MEMORY.md` übernommenen Kurzzeit-Abrufausschnitt beibehalten wird. Die Herkunft der Rangfolge bleibt sichtbar.
</ParamField>

<Warning>
`dreaming.model` erfordert `plugins.entries.memory-core.subagent.allowModelOverride: true`. Um dies einzuschränken, legen Sie außerdem `plugins.entries.memory-core.subagent.allowedModels` fest. Der automatische erneute Versuch gilt nur für Fehler aufgrund eines nicht verfügbaren Modells; Fehler bei Vertrauensprüfung oder Zulassungsliste bleiben in den Protokollen sichtbar, statt stillschweigend auszuweichen.
</Warning>

<Note>
Die meisten Phasenrichtlinien, Schwellenwerte und Speicherverhaltensweisen sind interne Implementierungsdetails. Die vollständige Liste der Schlüssel finden Sie in der [Referenz zur Gedächtniskonfiguration](/de/reference/memory-config#dreaming).
</Note>

## Träume-Benutzeroberfläche

Wenn aktiviert, zeigt der Gateway-Tab **Träume** Folgendes an:

- aktueller Aktivierungsstatus von Dreaming
- Status auf Phasenebene und Vorhandensein eines verwalteten Durchlaufs
- Anzahlen für Kurzzeit-, fundierte und heutige Signale sowie heutige Übernahmen
- Zeitpunkt des nächsten geplanten Durchlaufs
- einen eigenen fundierten Szenenpfad für gebündelte Einträge aus historischen Wiedergaben
- eine ausklappbare Traumtagebuchansicht auf Grundlage von `doctor.memory.dreamDiary`

## Verwandte Themen

- [Gedächtnis](/de/concepts/memory)
- [Gedächtnis-CLI](/de/cli/memory)
- [Referenz zur Gedächtniskonfiguration](/de/reference/memory-config)
- [Gedächtnissuche](/de/concepts/memory-search)
