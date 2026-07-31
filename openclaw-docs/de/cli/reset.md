---
read_when:
    - Sie möchten den lokalen Zustand löschen, während die CLI installiert bleibt
    - Sie möchten einen Probelauf durchführen, um zu sehen, was entfernt würde
summary: CLI-Referenz für `openclaw reset` (lokalen Zustand/lokale Konfiguration zurücksetzen)
title: Zurücksetzen
x-i18n:
    generated_at: "2026-07-26T18:22:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 54f1d320ee368dae4a4bfb32dea73d19eb35f9f30edd12d9c2580ab7e6a26fa6
    source_path: cli/reset.md
    workflow: 16
---

# `openclaw reset`

Lokale Konfiguration und lokalen Zustand zurücksetzen (die CLI bleibt installiert).

```bash
openclaw reset
openclaw reset --dry-run
openclaw reset --scope config --yes --non-interactive
openclaw reset --scope config+creds+sessions --yes --non-interactive
openclaw reset --scope full --yes --non-interactive
```

## Optionen

- `--scope <scope>`: `config`, `config+creds+sessions` oder `full`
- `--yes`: Bestätigungsabfragen überspringen
- `--non-interactive`: Abfragen deaktivieren; erfordert `--scope` und `--yes`
- `--dry-run`: Aktionen ausgeben, ohne Dateien zu entfernen

## Bereiche

| Bereich                   | Entfernt                                                                     | Stoppt zuerst den Gateway |
| ------------------------- | ---------------------------------------------------------------------------- | -------------------------- |
| `config`                | nur die Konfigurationsdatei                                                  | nein                       |
| `config+creds+sessions` | Konfigurationsdatei, OAuth-/Anmeldedatenverzeichnis und sitzungsspezifische Verzeichnisse pro Agent | ja |
| `full`                  | Zustandsverzeichnis (einschließlich der gemeinsam genutzten SQLite-Datenbank) sowie Arbeitsbereichsverzeichnisse | ja |

`config+creds+sessions` und `full` stoppen einen laufenden verwalteten Gateway-Dienst, bevor der Zustand gelöscht wird.

## Hinweise

- Führen Sie zuerst `openclaw backup create` aus, um vor dem Entfernen des lokalen Zustands einen wiederherstellbaren Snapshot zu erstellen.
- Der Einrichtungszustand und die Attestierungen des Arbeitsbereichs werden als Zeilen in der gemeinsam genutzten SQLite-Datenbank gespeichert. Daher entfernt `full` sie zusammen mit dem Zustandsverzeichnis; derzeit gibt es keine separaten Attestierungs-Sidecar-Dateien, die entfernt werden müssten.
- Ohne `--scope` fordert `openclaw reset` interaktiv zur Auswahl des zu entfernenden Bereichs auf.
- `--non-interactive` ist nur gültig, wenn sowohl `--scope` als auch `--yes` gesetzt sind.
- `config+creds+sessions` und `full` geben nach Abschluss `Next: openclaw onboard --install-daemon` aus.

## Verwandte Themen

- [CLI-Referenz](/de/cli)
