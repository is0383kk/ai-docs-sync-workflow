---
read_when:
    - Toewijzingen van apparaatmodel-ID's of NOTICE-/licentiebestanden bijwerken
    - Wijzigen hoe de Instances-UI apparaatnamen weergeeft
summary: Hoe OpenClaw model-ID's van Apple-apparaten opneemt om gebruiksvriendelijke namen in de macOS-app weer te geven.
title: Apparaatmodeldatabase
x-i18n:
    generated_at: "2026-07-27T05:20:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 930cd330594072d9c986b8c85c5a68e02dd096e5f0c015e3ee86b767073b93e6
    source_path: reference/device-models.md
    workflow: 16
---

De **Instances**-interface van de macOS-begeleidende app koppelt Apple-modelidentificaties aan herkenbare namen (`iPad16,6` -> "iPad Pro 13-inch (M4)", `Mac16,6` -> "MacBook Pro (14-inch, 2024)"). `DeviceModelCatalog` gebruikt ook het identificatievoorvoegsel (met het apparaatstype als terugvaloptie) om per apparaat een SF Symbol te kiezen.

Bestanden in `apps/macos/Sources/OpenClaw/Resources/DeviceModels/`:

| Bestand                                  | Doel                                        |
| ---------------------------------------- | ------------------------------------------- |
| `ios-device-identifiers.json`                       | iOS/iPadOS-identificatie -> naamtoewijzing  |
| `mac-device-identifiers.json`                       | Mac-identificatie -> naamtoewijzing         |
| `NOTICE.md`                       | Vastgezette upstream-commit-SHA's           |
| `LICENSE.apple-device-identifiers.txt`                       | Upstream-MIT-licentie                       |

## Gegevensbron

Overgenomen uit de onder de MIT-licentie uitgegeven GitHub-repository `kyle-seongwoo-jun/apple-device-identifiers`. JSON-bestanden zijn vastgezet op de commit-SHA's die zijn vastgelegd in `NOTICE.md`, zodat builds deterministisch blijven.

## De database bijwerken

1. Kies de upstream-commit-SHA's om vast te zetten (één voor iOS en één voor macOS).
2. Werk `apps/macos/Sources/OpenClaw/Resources/DeviceModels/NOTICE.md` bij met de nieuwe SHA's.
3. Download de JSON-bestanden die aan deze commits zijn gekoppeld opnieuw:

```bash
IOS_COMMIT="<commit sha for ios-device-identifiers.json>"
MAC_COMMIT="<commit sha for mac-device-identifiers.json>"

curl -fsSL "https://raw.githubusercontent.com/kyle-seongwoo-jun/apple-device-identifiers/${IOS_COMMIT}/ios-device-identifiers.json" \
  -o apps/macos/Sources/OpenClaw/Resources/DeviceModels/ios-device-identifiers.json

curl -fsSL "https://raw.githubusercontent.com/kyle-seongwoo-jun/apple-device-identifiers/${MAC_COMMIT}/mac-device-identifiers.json" \
  -o apps/macos/Sources/OpenClaw/Resources/DeviceModels/mac-device-identifiers.json
```

4. Controleer of `LICENSE.apple-device-identifiers.txt` nog steeds overeenkomt met upstream; vervang het bestand als de upstream-licentie is gewijzigd.
5. Controleer of de macOS-app zonder fouten wordt gebouwd:

```bash
swift build --package-path apps/macos
```

## Gerelateerd

- [Nodes](/nl/nodes)
- [Probleemoplossing voor Nodes](/nl/nodes/troubleshooting)
