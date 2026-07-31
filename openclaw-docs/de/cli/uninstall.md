---
read_when:
    - Sie möchten den Gateway-Dienst und/oder den lokalen Zustand entfernen
    - Sie möchten zuerst einen Probelauf durchführen
summary: CLI-Referenz für `openclaw uninstall` (Gateway-Dienst und lokale Daten entfernen)
title: Deinstallieren
x-i18n:
    generated_at: "2026-07-26T17:44:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1e2e3996cf6d5c0fd11e5054c8fe60f7f8d25047193bb13944ca170bf77b581a
    source_path: cli/uninstall.md
    workflow: 16
---

# `openclaw uninstall`

Deinstallieren Sie den Gateway-Dienst und/oder lokale Daten. Die CLI selbst wird nicht
entfernt; deinstallieren Sie sie separat über npm/pnpm.

## Optionen

| Flag                | Standard | Beschreibung                                          |
| ------------------- | ------- | ---------------------------------------------------- |
| `--service`         | `false` | Entfernt den Gateway-Dienst.                          |
| `--state`           | `false` | Entfernt Status und Konfiguration.                             |
| `--workspace`       | `false` | Entfernt Workspace-Verzeichnisse.                        |
| `--app`             | `false` | Entfernt die macOS-App.                                |
| `--all`             | `false` | Kurzform für `--service --state --workspace --app`. |
| `--yes`             | `false` | Überspringt Bestätigungsaufforderungen.                           |
| `--non-interactive` | `false` | Deaktiviert Aufforderungen; erfordert `--yes`.                   |
| `--dry-run`         | `false` | Gibt geplante Aktionen aus, ohne Dateien zu entfernen.        |

Wenn keine Umfangs-Flags angegeben sind, fragt eine interaktive Mehrfachauswahl ab, welche Komponenten
entfernt werden sollen (standardmäßig sind Dienst, Status und Workspace vorausgewählt).

## Beispiele

```bash
openclaw backup create
openclaw uninstall
openclaw uninstall --service --yes --non-interactive
openclaw uninstall --state --workspace --yes --non-interactive
openclaw uninstall --all --yes
openclaw uninstall --dry-run
```

## Hinweise

- Führen Sie zuerst `openclaw backup create` aus, um vor dem Entfernen von
  Status oder Workspaces einen wiederherstellbaren Snapshot zu erstellen.
- `--state` behält konfigurierte Workspace-Verzeichnisse bei, sofern nicht auch `--workspace`
  ausgewählt ist.

## Verwandte Themen

- [CLI-Referenz](/de/cli)
- [Deinstallation](/de/install/uninstall)
