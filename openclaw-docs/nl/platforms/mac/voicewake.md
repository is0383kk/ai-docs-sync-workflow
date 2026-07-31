---
read_when:
    - Werken aan activeringsroutes via spraak of PTT
summary: Spraakactivering en push-to-talk-modi plus routeringsdetails in de Mac-app
title: Stemactivatie (macOS)
x-i18n:
    generated_at: "2026-07-27T05:39:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d3b2a01ee997b4158bf88b9ef54b1e523503722620f943d594323516619e7502
    source_path: platforms/mac/voicewake.md
    workflow: 16
---

# Voice Wake en push-to-talk

## Vereisten

Voice Wake en push-to-talk vereisen macOS 26 of nieuwer. Op oudere macOS-versies zijn de bedieningselementen verborgen op de pagina met spraakinstellingen, waar in plaats daarvan de vereiste van macOS 26 wordt weergegeven.

Voor Voice Wake moet Apple Speech herkenning op het apparaat ondersteunen voor de geselecteerde taal. De app weigert passief naar het activeringswoord te luisteren wanneer dit uitsluitend lokale contract niet beschikbaar is; er wordt nooit teruggevallen op herkenning via het netwerk. Push-to-talk, Talk Mode en dicteren met Quick Chat zijn expliciete gebruikersacties en mogen Apple Speech-netwerkdiensten gebruiken voor bredere taalondersteuning.

## Modi

- **Activeringswoordmodus** (standaard): een altijd actieve Speech-herkenner op het apparaat wacht op activeringstokens (`swabbleTriggerWords`). Bij een overeenkomst begint de opname, verschijnt de overlay met gedeeltelijke tekst en wordt het bericht na een stilte automatisch verzonden.
- **Push-to-talk (rechter Option ingedrukt houden)**: houd de rechter Option-toets ingedrukt om direct op te nemen; er is geen activering nodig. De overlay verschijnt zolang je de toets ingedrukt houdt. Als je de toets loslaat, wordt de opname afgerond en na een korte vertraging doorgestuurd, zodat je de tekst kunt bewerken.

## Runtimegedrag (activeringswoord)

- De herkenner bevindt zich in `VoiceWakeRuntime`.
- De activering vindt alleen plaats wanneer er een duidelijke pauze is tussen het activeringswoord en het volgende woord (`triggerPauseWindow` = 0.55s). De overlay/geluidstoon kan tijdens de pauze al starten, nog voordat de opdracht begint.
- Stiltevensters: 2.0s (`silenceWindow`) wanneer er doorlopend wordt gesproken, 5.0s (`triggerOnlySilenceWindow`) als alleen de activering is gehoord.
- Harde stop: 120s (`captureHardStop`) om uit de hand lopende sessies te voorkomen.
- Debounce tussen sessies: 350ms (`debounceAfterSend`) na verzending.
- De overlay wordt aangestuurd via `VoiceWakeOverlayController`, met verschillende tekstkleuren voor vastgelegde en vluchtige tekst.
- Na verzending wordt de herkenner opnieuw en zonder resterende status gestart om naar de volgende activering te luisteren.

## Levenscyclusinvarianten

- Als Voice Wake is ingeschakeld en de machtigingen zijn verleend, blijft de activeringswoordherkenner luisteren, behalve tijdens een actieve push-to-talk-opname.
- Bij het sluiten van de overlay, ook wanneer deze handmatig met de X-knop wordt gesloten, wordt de herkenner altijd hervat: `VoiceSessionCoordinator.overlayDidDismiss` roept bij elk sluitingspad `VoiceWakeRuntime.refresh(state:)` aan. Zie [Spraakoverlay](/nl/platforms/mac/voice-overlay) voor het sessie-/tokenmodel.

## Details van push-to-talk

- Voor sneltoetsdetectie wordt een globale `.flagsChanged`-monitor gebruikt voor de rechter Option-toets (`keyCode 61` + `.option`). Deze neemt gebeurtenissen alleen waar en houdt ze nooit tegen.
- De opname bevindt zich in `VoicePushToTalk`: Speech wordt onmiddellijk gestart, gedeeltelijke resultaten worden naar de overlay gestreamd en bij het loslaten wordt `VoiceWakeForwarder` aangeroepen.
- Bij het starten van push-to-talk wordt de activeringswoordruntime gepauzeerd om conflicterende audiotaps te voorkomen; deze wordt na het loslaten automatisch opnieuw gestart.
- Machtigingen: Microfoon + Spraak zijn vereist; voor het ontvangen van toetsgebeurtenissen is goedkeuring voor Accessibility/Input Monitoring nodig.
- Externe toetsenborden: sommige maken de rechter Option-toets niet beschikbaar zoals verwacht. Bied een alternatieve sneltoets aan als gebruikers melden dat invoer wordt gemist.

## Instellingen voor gebruikers

- Schakelaar **Voice Wake**: schakelt de activeringswoordruntime in.
- **Hold Right Option to talk**: schakelt de push-to-talk-monitor in.
- Als de geselecteerde taal geen herkenning op het apparaat ondersteunt op deze Mac, blijft Voice Wake uitgeschakeld terwijl push-to-talk en Talk Mode beschikbaar blijven.
- Taal- en microfoonkeuzelijsten, een live niveaumeter, een tabel met activeringswoorden en een tester (uitsluitend lokaal, stuurt nooit iets door).
- De microfoonkeuzelijst behoudt de laatste selectie als een apparaat wordt losgekoppeld, toont een melding dat het apparaat niet is verbonden en valt tijdelijk terug op de systeemstandaard totdat het apparaat terugkeert.
- **Geluiden**: geluidstonen bij detectie van de activering en bij verzending, standaard ingesteld op het macOS-systeemgeluid "Glass". Kies per gebeurtenis een bestand dat door `NSSound` kan worden geladen (bijv. MP3/WAV/AIFF), of kies **No Sound**.

## Doorstuurgedrag

- Bij het doorsturen kiest `VoiceWakeForwarder.selectedSessionOptions` de sessiesleutel van de actieve WebChat-sessie als die is ingesteld, en anders de hoofdsessiesleutel van de Gateway.
- De sessie wordt opgezocht via `sessions.list`. Het bezorgkanaal en het doel worden afgeleid uit de bezorgcontext van de sessie (waarbij wordt teruggevallen op het laatste kanaal/doel en vervolgens op een geparseerde sessiesleutel). Als niets kan worden bepaald, wordt standaard WebChat gebruikt.
- Als de bezorging mislukt, wordt de fout vastgelegd (`voicewake.forward`-categorie) en blijft de uitvoering zichtbaar via WebChat-/sessielogboeken.

## Doorstuurpayload

- `VoiceWakeForwarder.prefixedTranscript(_:)` plaatst vóór het transcript een regel met een machinehint (de bepaalde hostnaam, waarbij wordt teruggevallen op "deze Mac"), die wordt gedeeld door de activeringswoord- en push-to-talk-paden.

## Snelle verificatie

- Schakel push-to-talk in, houd de rechter Option-toets ingedrukt, spreek en laat de toets los: de overlay moet gedeeltelijke resultaten tonen en deze vervolgens verzenden.
- Tijdens het ingedrukt houden moeten de oren in de menubalk vergroot blijven (`triggerVoiceEars(ttl: nil)`); na het loslaten worden ze weer kleiner.

## Gerelateerd

- [Voice Wake](/nl/nodes/voicewake)
- [Spraakoverlay](/nl/platforms/mac/voice-overlay)
- [macOS-app](/nl/platforms/macos)
