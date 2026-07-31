---
read_when:
    - Aktualisieren von Zuordnungen für Gerätemodellkennungen oder NOTICE-/Lizenzdateien
    - Ändern der Anzeige von Gerätenamen in der Instanzen-Benutzeroberfläche
summary: Wie OpenClaw Apple-Gerätemodellkennungen für benutzerfreundliche Namen in der macOS-App einbindet.
title: Gerätemodell-Datenbank
x-i18n:
    generated_at: "2026-07-26T18:03:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 930cd330594072d9c986b8c85c5a68e02dd096e5f0c015e3ee86b767073b93e6
    source_path: reference/device-models.md
    workflow: 16
---

Die Benutzeroberfläche **Instances** der macOS-Begleit-App ordnet Apple-Modellkennungen verständlichen Namen zu (`iPad16,6` -> „iPad Pro 13 Zoll (M4)“, `Mac16,6` -> „MacBook Pro (14 Zoll, 2024)“). `DeviceModelCatalog` verwendet außerdem das Kennungspräfix (mit Rückgriff auf die Gerätefamilie), um für jedes Gerät ein SF Symbol auszuwählen.

Dateien in `apps/macos/Sources/OpenClaw/Resources/DeviceModels/`:

| Datei                                  | Zweck                                  |
| -------------------------------------- | -------------------------------------- |
| `ios-device-identifiers.json`                     | Zuordnung iOS-/iPadOS-Kennung -> Name  |
| `mac-device-identifiers.json`                     | Zuordnung Mac-Kennung -> Name           |
| `NOTICE.md`                     | Angeheftete Upstream-Commit-SHAs        |
| `LICENSE.apple-device-identifiers.txt`                     | MIT-Lizenz des Upstream-Projekts        |

## Datenquelle

Aus dem unter der MIT-Lizenz stehenden GitHub-Repository `kyle-seongwoo-jun/apple-device-identifiers` eingebunden. Die JSON-Dateien sind an die in `NOTICE.md` aufgezeichneten Commit-SHAs gebunden, damit Builds deterministisch bleiben.

## Datenbank aktualisieren

1. Wählen Sie die Upstream-Commit-SHAs aus, die angeheftet werden sollen (eine für iOS, eine für macOS).
2. Aktualisieren Sie `apps/macos/Sources/OpenClaw/Resources/DeviceModels/NOTICE.md` mit den neuen SHAs.
3. Laden Sie die an diese Commits gebundenen JSON-Dateien erneut herunter:

```bash
IOS_COMMIT="<commit sha for ios-device-identifiers.json>"
MAC_COMMIT="<commit sha for mac-device-identifiers.json>"

curl -fsSL "https://raw.githubusercontent.com/kyle-seongwoo-jun/apple-device-identifiers/${IOS_COMMIT}/ios-device-identifiers.json" \
  -o apps/macos/Sources/OpenClaw/Resources/DeviceModels/ios-device-identifiers.json

curl -fsSL "https://raw.githubusercontent.com/kyle-seongwoo-jun/apple-device-identifiers/${MAC_COMMIT}/mac-device-identifiers.json" \
  -o apps/macos/Sources/OpenClaw/Resources/DeviceModels/mac-device-identifiers.json
```

4. Bestätigen Sie, dass `LICENSE.apple-device-identifiers.txt` weiterhin mit dem Upstream-Projekt übereinstimmt; ersetzen Sie die Datei, falls sich die Upstream-Lizenz geändert hat.
5. Überprüfen Sie, ob die macOS-App fehlerfrei gebaut wird:

```bash
swift build --package-path apps/macos
```

## Verwandte Themen

- [Nodes](/de/nodes)
- [Fehlerbehebung für Nodes](/de/nodes/troubleshooting)
