---
read_when:
    - Je wilt dat een OpenClaw-agent deelneemt aan een videovergadering
    - Je kiest tussen de plugins voor Google Meet-, Microsoft Teams- en Zoom-vergaderingen
    - Je hebt de gedeelde Chrome-, BlackHole-, SoX- of vergadermodusconfiguratie nodig
summary: Kies en configureer deelname aan vergaderingen via Google Meet, Microsoft Teams of Zoom
title: Vergaderplugins
x-i18n:
    generated_at: "2026-07-27T05:12:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f41488de018402e3d5cfd01fa5351cdb6107412477d5d54e2d9e186e0fc8ee94
    source_path: plugins/meeting-plugins.md
    workflow: 16
---

OpenClaw heeft afzonderlijke plugins voor Google Meet, Microsoft Teams-vergaderingen en Zoom. Alle drie kunnen via Chrome deelnemen, gebruiken dezelfde deelnamemodi en kunnen Chrome uitvoeren op de Gateway-host of op een gekoppelde node. Hun platform-URL's, installatiemodel en extra mogelijkheden verschillen.

Deze plugins nemen deel aan vergaderingen. Ze staan los van berichtkanalen zoals het [Microsoft Teams-kanaal](/nl/channels/msteams) en van de [Plugin voor spraakoproepen](/nl/plugins/voice-call).

## Kies een plugin

| Platform        | Plugin                                      | Geaccepteerde vergaderlinks                                                                                  | Installatie                                     | Deelnamepaden                                             | Platformspecifieke mogelijkheden                                                                                         |
| --------------- | ------------------------------------------- | ----------------------------------------------------------------------------------------------------------- | ------------------------------------------------ | --------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| Google Meet     | [`google-meet`](/nl/plugins/google-meet)       | `meet.google.com/...`                                                                                       | Installeren vanaf npm of ClawHub; standaard ingeschakeld | Lokale Chrome, Chrome op een gekoppelde node of inbellen via Twilio | Kan vergaderingen maken via de Meet-API of een aangemelde browser; kan ondersteunde Meet-artefacten lezen met OAuth |
| Microsoft Teams | [`teams-meetings`](/plugins/teams-meetings) | Werklinks onder `teams.microsoft.com/l/meetup-join/...` en consumentenlinks onder `teams.live.com/meet/...` | Inbegrepen; standaard ingeschakeld              | Lokale Chrome of Chrome op een gekoppelde node            | Deelname als gast aan werk- en consumentenvergaderingen                                                                  |
| Zoom            | [`zoom-meetings`](/plugins/zoom-meetings)   | `zoom.us/j/...` en accountsubdomeinen zoals `example.zoom.us/j/...`                                      | Inbegrepen; standaard ingeschakeld              | Lokale Chrome of Chrome op een gekoppelde node            | Deelname als gast via de Zoom Web App                                                                                    |

Kies Google Meet wanneer je vergaderingen moet maken, Google API-artefacten nodig hebt of via Twilio telefonisch wilt deelnemen. Kies Teams of Zoom voor rechtstreekse deelname als gast via de browser op die platforms. De plugins voor Teams en Zoom maken geen vergaderingen, bellen niet in, roepen de API van de leverancier niet aan en maken geen audio- of video-opnamen.

## Kies een modus

De drie plugins hebben dezelfde modi:

| Modus        | Gedrag                                                                                                  | Audiovereisten                                          |
| ------------ | ------------------------------------------------------------------------------------------------------- | ------------------------------------------------------- |
| `agent`      | Realtime transcriptie gaat naar de geconfigureerde OpenClaw-agent; reguliere OpenClaw-TTS spreekt het antwoord uit. | Terugspreken via Chrome vereist de BlackHole- en SoX-bridge. |
| `bidi`       | Een realtime spraakmodel luistert en antwoordt rechtstreeks.                                           | Terugspreken via Chrome vereist de BlackHole- en SoX-bridge. |
| `transcribe` | Neemt alleen als waarnemer deel en stelt een begrensd realtime ondertitelingstranscript beschikbaar wanneer het platform ondertiteling levert. | Geen BlackHole- of SoX-bridge voor terugspreken.        |

Gebruik `transcribe` wanneer de agent alleen de tekst van de vergadering nodig heeft. Gebruik `agent` voor normale OpenClaw-redenering en tools. Gebruik `bidi` wanneer directe spraak met lage latentie belangrijker is dan elke beurt via de reguliere agent te routeren.

Het begrensde realtime transcript blijft alleen beschikbaar in de modus `transcribe`. In alle
drie de modi slaan deelnames via de browser ook voltooide ondertitelingsregels en een afgeleide
samenvatting op in de gedeelde statusdatabase. Bij het verlaten van de vergadering worden zichtbare
ondertitels voltooid en wordt de samenvatting opgeslagen; gebruik [`openclaw transcripts`](/nl/cli/transcripts)
om deze weer te geven, te bekijken of te exporteren. Dit duurzame notitiepad verandert het realtime
transcript voor agentraadpleging niet en maakt geen audio- of video-opname.

Automatische notities zijn standaard ingeschakeld. Stel `transcripts.enabled: false` in om
duurzame notities globaal uit te schakelen. Een expliciet geselecteerde `transcribe`-sessie behoudt zijn
begrensde uiteinde van de realtime ondertiteling zonder duurzame rijen te schrijven. De beschikbaarheid van ondertiteling
hangt nog steeds af van het vergaderplatform, account, de taal en het beleid van de host.

## Chrome en audio voorbereiden

Chrome kan op de Gateway-host of op een gekoppelde node worden uitgevoerd. Een externe Chrome-node moet `browser.proxy` plus de platformopdracht toestaan:

| Plugin          | Node-opdracht           |
| --------------- | ----------------------- |
| Google Meet     | `googlemeet.chrome`    |
| Microsoft Teams | `teamsmeetings.chrome` |
| Zoom            | `zoommeetings.chrome`  |

Voer voor de modus `agent` of `bidi` via Chrome Chrome uit op macOS en installeer de gedeelde audioafhankelijkheden op diezelfde host:

```bash
brew install blackhole-2ch sox
sudo reboot
system_profiler SPAudioDataType | grep -i BlackHole
command -v sox
```

De Gateway-host blijft eigenaar van de OpenClaw-agent en modelreferenties wanneer Chrome op een gekoppelde node wordt uitgevoerd. Configureer een provider voor realtime transcriptie en OpenClaw-TTS voor de modus `agent`, of een provider voor realtime spraak voor de modus `bidi`. De platformhandleidingen bevatten de opties voor providers en audio-opdrachten.

## Plugins installeren of uitschakelen

Installeer Google Meet afzonderlijk; na installatie is het standaard ingeschakeld. Microsoft Teams-vergaderingen en Zoom zijn inbegrepen bij OpenClaw en standaard ingeschakeld:

```bash
# Alleen Google Meet
openclaw plugins install npm:@openclaw/google-meet
```

Schakel elke vergaderplugin uit die je niet gebruikt:

```bash
openclaw plugins disable google-meet
openclaw plugins disable teams-meetings
openclaw plugins disable zoom-meetings
```

Start de Gateway opnieuw als het pad voor pluginbeheer dit niet automatisch doet. Voer vervolgens de installatiecontrole voor het platform uit voordat je deelneemt.

## Controleren en deelnemen

| Platform        | Installatiecontrole              | Opdracht om deel te nemen                                                    |
| --------------- | -------------------------------- | ----------------------------------------------------------------------------- |
| Google Meet     | `openclaw googlemeet setup`    | `openclaw googlemeet join 'https://meet.google.com/abc-defg-hij'`             |
| Microsoft Teams | `openclaw teamsmeetings setup` | `openclaw teamsmeetings join 'https://teams.microsoft.com/l/meetup-join/...'` |
| Zoom            | `openclaw zoommeetings setup`  | `openclaw zoommeetings join 'https://zoom.us/j/1234567890'`                   |

Beschouw elke mislukte installatiecontrole als een blokkade voor dat transport en die modus. Selecteer voor een rooktest met alleen waarnemen de modus `transcribe` en controleer of de status een actieve oproepsessie meldt voordat je ondertitelingstekst verwacht.

Voor rooktests met terugspreken vereist geverifieerde spraak meer dan alleen bytes die door de afspeelopdracht worden geaccepteerd. De gedeelde bridge voor opdrachtenparen correleert een begrensde golfvormvingerafdruk van de huidige uitvoergeneratie met audio die terugkomt via het opnamepad van de BlackHole-microfoon; Google Meet, Teams en Zoom melden `speechOutputVerified: true` niet wanneer alleen de teller voor uitvoerbytes oploopt of wanneer audio van een niet-gerelateerde deelnemer aanwezig is.

## Omgaan met prompts voor platformbeleid

Browserautomatisering verwerkt de normale bedieningselementen voor de gastnaam, camera en microfoon vóór deelname, deelname, tijdens het gesprek en verlaten. Ze omzeilt geen beleid van het platform of de organisator.

- Google Meet kan vereisen dat je je bij Google aanmeldt, door de host wordt toegelaten of een browsermachtiging kiest.
- Microsoft Teams kan vereisen dat je je bij de tenant aanmeldt, je e-mailadres verifieert of door de organisator wordt toegelaten.
- Zoom kan authenticatie, e-mailverificatie, een toegangscode, het voltooien van een CAPTCHA of toelating door de host vereisen; een account kan deelname via de browser ook uitschakelen.

Wanneer een deelname- of statusresultaat `manualActionRequired` meldt, voltooi je de gemelde stap in hetzelfde OpenClaw Chrome-profiel voordat je het opnieuw probeert. Het herhaaldelijk openen van nieuwe tabbladen lost een blokkade door een account, tenant, lobby of CAPTCHA niet op.

Neem alleen deel aan vergaderingen waarvoor de operator gemachtigd is een agent toe te voegen. Informeer deelnemers wanneer lokaal beleid of toestemmingsregels openbaarmaking van geautomatiseerde deelname, transcriptie of gesynthetiseerde spraak vereisen.

## Discord-spraakchat

[Discord-spraakkanalen](/nl/channels/discord#voice-channels) bieden systeemeigen realtime gesprekken met alleen audio, zonder automatisering van browservergaderingen. OpenClaw kan deelnemen aan een spraakkanaal, luisteren, beurten via een OpenClaw-agent of realtime spraakmodel routeren en antwoorden uitspreken. Het verzendt of ontvangt geen cameravideo of schermdeling, zelfs niet wanneer mensen video gebruiken in hetzelfde Discord-kanaal. Discord-spraak is daarom een verwant oppervlak voor livegesprekken en geen vierde plugin voor browservergaderingen.

## Platformhandleidingen

- [Google Meet-plugin](/nl/plugins/google-meet)
- [Plugin voor Microsoft Teams-vergaderingen](/plugins/teams-meetings)
- [Plugin voor Zoom-vergaderingen](/plugins/zoom-meetings)
- [Plugins beheren](/nl/plugins/manage-plugins)
- [Browserbesturing](/nl/tools/browser)
