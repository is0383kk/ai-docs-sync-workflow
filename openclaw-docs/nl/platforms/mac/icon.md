---
read_when:
    - Gedrag van het menubalkpictogram wijzigen
summary: Statussen en animaties van het menubalkpictogram voor OpenClaw op macOS
title: Menubalkpictogram
x-i18n:
    generated_at: "2026-07-27T05:11:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8a38f1253f0c376ef2ce6c0ae339b67084c472c764964bcc7ad21e10133e2b47
    source_path: platforms/mac/icon.md
    workflow: 16
---

# Statussen van het menubalkpictogram

Bereik: macOS-app (`apps/macos`). Rendering: `CritterIconRenderer.makeIcon(...)`. Koppeling van animatie/status: `CritterStatusLabel` + `CritterStatusLabel+Behavior.swift`.

## Statussen

| Status                | Trigger                                   | Weergave                                                                                              |
| --------------------- | ----------------------------------------- | --------------------------------------------------------------------------------------------------- |
| Inactief              | Standaard                                 | Normale knipper-/wiebelfanimatie; open ogen behouden een glanzende lichtreflectie                    |
| Gepauzeerd            | `isPaused=true`                           | Voelsprieten hangen omlaag ("buiten dienst") met open ogen; geen beweging                           |
| Slapend               | Gateway niet verbonden/niet geconfigureerd | Voelsprieten hangen omlaag en ogen sluiten tot `⌣ ⌣`-oogleden; geen beweging            |
| Vieren                | Bericht verzonden (`sendCelebrationTick`)      | Ogen tonen ~0,9 s vrolijke, oplichtende `∩ ∩`-boogjes, plus een schopbeweging met een poot |
| Spraakactivatie (grote oren) | Activeringswoord gehoord             | Voelsprieten gaan rechtop staan en worden langer (`earScale=1.9`); zakken na stilte weer omlaag |
| Bezig                 | `isWorking=true` of een actieve `IconState` | Sneller gewiebel met de poten (`legWiggle` tot `1.0`) plus een kleine horizontale verschuiving; boven op het inactieve gewiebel |

Een badge voor toolactiviteit (een schijfje met een SF Symbol, bijvoorbeeld `chevron.left.slash.chevron.right` voor uitvoering) kan boven op hetzelfde beestjespictogram worden weergegeven wanneer een sessie een actieve taak of tool heeft. Die badge is afkomstig van `IconState`/`ActivityKind`; zie [Menubalk](/nl/platforms/mac/menu-bar) voor het volledige statusmodel.

## Oren bij spraakactivatie

- Trigger: `AppStateStore.shared.triggerVoiceEars(ttl: nil)`, aangeroepen vanuit de opnamepijplijn voor spraakactivatie (`VoiceWakeRuntime`) en vanuit debug-/testtools voor spraakactivatie (`VoiceWakeTester`, `VoiceWakeOverlayController`).
- Stoppen: `stopVoiceEars()`, aangeroepen wanneer de opname wordt afgerond.
- Stilteperiode vóór afronding: normaal `2.0s`, of `5.0s` als alleen het activeringswoord is gehoord en er geen verdere spraak volgde (`VoiceWakeRuntime.silenceWindow` / `triggerOnlySilenceWindow`).
- Tijdens de versterking worden de timers voor inactief knipperen en wiebelen en voor poot- en oorbewegingen onderbroken (`earBoostActive` blokkeert de animatietaak in `CritterStatusLabel+Behavior`).

## Vormen en afmetingen

- Canvas: sjabloonafbeelding van 18x18 pt, gerenderd naar een bitmapbuffer van 36x36 px (2x), zodat het pictogram scherp blijft op Retina-schermen.
- De oorschaal is standaard `1.0`; spraakversterking stelt `earScale=1.9` in zonder het totale kader te wijzigen.
- `antennaDroop` (0-1) klapt de voelsprieten omlaag voor de gepauzeerde en slapende houdingen.
- Het rennen met de poten gebruikt `legWiggle` tot `1.0`, met een kleine horizontale schudbeweging.

## Gedragsnotities

- Er is geen externe CLI-/broker-schakelaar voor de oren of de werkstatus; beide worden intern aangestuurd door app-signalen (`AppState.setWorking`, `AppState.triggerVoiceEars`) om onbedoeld heen-en-weer bewegen te voorkomen.
- Houd elke nieuwe TTL kort (ruim onder 10 s), zodat het pictogram snel terugkeert naar de basisstatus als een taak blijft hangen.

## Gerelateerd

- [Menubalk](/nl/platforms/mac/menu-bar)
- [macOS-app](/nl/platforms/macos)
