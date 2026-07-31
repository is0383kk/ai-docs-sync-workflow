---
read_when:
    - Spraakoverlaygedrag aanpassen
summary: Levenscyclus van de spraakoverlay wanneer het activeringswoord en push-to-talk elkaar overlappen
title: Spraakoverlay
x-i18n:
    generated_at: "2026-07-27T05:21:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: eef571c3e8d41a97779537b1b373fab25b08f63575b50e5019f6c5fbcb782c52
    source_path: platforms/mac/voice-overlay.md
    workflow: 16
---

# Levenscyclus van de spraakoverlay (macOS)

Doelgroep: bijdragers aan de macOS-app. Doel: de spraakoverlay voorspelbaar houden wanneer activering via een wekwoord en push-to-talk elkaar overlappen.

## Gedrag

- Als de overlay al zichtbaar is door het wekwoord en de gebruiker op de sneltoets drukt, neemt de sneltoetssessie de bestaande tekst over in plaats van deze opnieuw in te stellen. De overlay blijft zichtbaar zolang de sneltoets ingedrukt wordt gehouden. Bij loslaten: verzenden als er tekst overblijft na het verwijderen van witruimte, anders sluiten.
- Alleen het wekwoord verzendt nog steeds automatisch bij stilte; push-to-talk verzendt onmiddellijk bij het loslaten.

## Implementatie

- `VoiceSessionCoordinator` (`apps/macos/Sources/OpenClaw/VoiceSessionCoordinator.swift`) is de enige eigenaar van de actieve spraaksessie. Het is een `@MainActor @Observable`-singleton, geen actor. API: `startSession`, `updatePartial`, `finalize`, `sendNow`, `dismiss`, `updateLevel`, `snapshot`. Elke sessie bevat een `UUID`-token; aanroepen met een verouderd of niet-overeenkomend token worden genegeerd.
- `VoiceWakeOverlayController` (`VoiceWakeOverlayController+Session.swift`) rendert de overlay en stuurt gebruikersacties (`requestSend`, `dismiss`) via het sessietoken terug naar de coördinator. Het beheert nooit zelf de sessiestatus.
- Push-to-talk (`VoicePushToTalk.begin()`) neemt alle zichtbare overlaytekst over als `adoptedPrefix` (via `VoiceSessionCoordinator.shared.snapshot()`), zodat de tekst behouden blijft en nieuwe spraak eraan wordt toegevoegd wanneer de sneltoets wordt ingedrukt terwijl de wekwoordoverlay zichtbaar is. Bij het loslaten wacht het maximaal 1.5s op een definitief transcript voordat het terugvalt op de huidige tekst.
- Bij `dismiss` roept de overlay `VoiceSessionCoordinator.overlayDidDismiss` aan, wat `VoiceWakeRuntime.refresh(state:)` activeert, zodat luisteren naar het wekwoord wordt hervat na handmatig sluiten met X, sluiten wegens lege tekst en sluiten na verzending.
- Uniform verzendpad: als de tekst na het verwijderen van witruimte leeg is, sluiten; anders speelt `sendNow` eenmaal het verzendgeluid af, stuurt de tekst door via `VoiceWakeForwarder` en sluit vervolgens de overlay.

## Logboekregistratie

Het spraaksubsysteem is `ai.openclaw`; elk onderdeel registreert logboekberichten in zijn eigen categorie:

| Categorie                | Onderdeel                                       |
| ----------------------- | ----------------------------------------------- |
| `voicewake.coordinator` | `VoiceSessionCoordinator`                       |
| `voicewake.overlay`     | `VoiceWakeOverlayController`/`VoiceWakeOverlay` |
| `voicewake.ptt`         | Push-to-talk-sneltoets en -opname                 |
| `voicewake.runtime`     | Wekwoordruntime                               |
| `voicewake.chime`       | Afspelen van geluid                                  |
| `voicewake.sync`        | Synchronisatie van algemene instellingen                            |
| `voicewake.forward`     | Doorsturen van transcript                           |
| `voicewake.meter`       | Microfoonniveaumonitor                               |

## Checklist voor foutopsporing

- Stream de logboeken tijdens het reproduceren van een overlay die zichtbaar blijft:

  ```bash
  sudo log stream --predicate 'subsystem == "ai.openclaw" AND category CONTAINS "voicewake"' --level info --style compact
  ```

- Controleer of er slechts één actief sessietoken is; verouderde callbacks worden door de coördinator genegeerd.
- Controleer of bij het loslaten van push-to-talk altijd `end()` wordt aangeroepen met het actieve token; verwacht bij lege tekst dat de overlay wordt gesloten zonder geluid of verzending.

## Gerelateerd

- [macOS-app](/nl/platforms/macos)
- [Spraakactivering (macOS)](/nl/platforms/mac/voicewake)
- [Gespreksmodus](/nl/nodes/talk)
