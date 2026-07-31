---
read_when:
    - Je wilt dat een OpenClaw-agent deelneemt aan een Zoom-vergadering
    - Je configureert Chrome, BlackHole of SoX voor terugspreken tijdens Zoom-vergaderingen
summary: 'Zoom-vergaderingenplugin: neem als gast deel aan vergaderingen via de Chrome-browser'
title: Plugin voor Zoom-vergaderingen
x-i18n:
    generated_at: "2026-07-27T05:11:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d91e57cccb163f634c6eaee71dd3832fc7b9e783fc5cd02601572b302d0d25e8
    source_path: plugins/zoom-meetings.md
    workflow: 16
---

De plugin `zoom-meetings` neemt als gast deel via Zoom-vergaderlinks met de Zoom Web App in het OpenClaw Chrome-profiel. De plugin accepteert vergaderlinks onder `zoom.us/j/...` en accountsubdomeinen zoals `example.zoom.us/j/...`. De plugin maakt geen vergaderingen aan, belt niet in, gebruikt de Zoom Meeting SDK niet en legt geen audio- of video-opnamen vast.

## Installatie

Terugspreken gebruikt dezelfde lokale audiovereisten als de [Google Meet-plugin](/nl/plugins/google-meet): macOS, het virtuele audioapparaat `BlackHole 2ch` en SoX.

```bash
brew install blackhole-2ch sox
sudo reboot
system_profiler SPAudioDataType | grep -i BlackHole
command -v sox
```

De plugin is inbegrepen en standaard ingeschakeld. Voeg alleen een vermelding toe om de plugin aan te passen en controleer daarna de installatie:

```json5
{
  plugins: {
    entries: {
      "zoom-meetings": {
        config: {
          defaultMode: "agent",
          chrome: { guestName: "OpenClaw Agent" },
        },
      },
    },
  },
}
```

Voer `openclaw plugins disable zoom-meetings` uit als je de plugin niet actief wilt hebben.

```bash
openclaw zoommeetings setup
openclaw zoommeetings join 'https://zoom.us/j/1234567890'
```

Gebruik `chromeNode.node` om Chrome, BlackHole en SoX op een gekoppelde macOS-Node uit te voeren. De Node moet `zoommeetings.chrome` en `browser.proxy` toestaan.

## Modi

| Modus         | Gedrag                                                                    |
| ------------ | --------------------------------------------------------------------------- |
| `agent`      | Realtime transcriptie raadpleegt de geconfigureerde OpenClaw-agent; TTS antwoordt. |
| `bidi`       | Een realtime spraakmodel luistert en antwoordt rechtstreeks.                        |
| `transcribe` | Alleen observeren met momentopnamen van het live-ondertitelingstranscript.                   |

Live-ondertiteling van Zoom wordt na toelating in elke modus ingeschakeld, zodat OpenClaw
vergadernotities kan bewaren. De actie `transcript` retourneert nog steeds alleen
de begrensde livebuffer voor `transcribe`-sessies. Bij het verlaten slaat OpenClaw
het duurzame transcript en de afgeleide samenvatting op in de gedeelde statusdatabase;
bekijk of exporteer deze met [`openclaw transcripts`](/nl/cli/transcripts).

Automatische notities zijn standaard ingeschakeld. Stel `transcripts.enabled: false` in om
duurzame notities globaal uit te schakelen; de expliciete modus `transcribe` stelt nog steeds alleen
het begrensde live-einde beschikbaar.

## Beperkingen voor deelname als gast

De browseradapter kiest **Join from browser**, vult de gastnaam in, schakelt de camera uit, configureert de microfoon voor de geselecteerde modus en klikt op **Join**. Zoom Web App wordt uitgevoerd onder `app.zoom.us`; de plugin verleent die oorsprong vóór de navigatie machtigingen voor de microfoon en luidsprekerselectie. De status tijdens het gesprek gebruikt de Leave-bediening van Zoom. Wachtruimte-, aanmeldings-, toegangscode-, CAPTCHA- en apparaatmachtigingsstatussen retourneren expliciete redenen waarom handmatige actie nodig is.

Het beleid van de Zoom-host en het account kan deelname via de browser uitschakelen, authenticatie of e-mailverificatie vereisen, een CAPTCHA tonen of toelating door de host vereisen. Voltooi die stap in het OpenClaw Chrome-profiel en probeer daarna de status of spraak opnieuw. De plugin omzeilt het Zoom-beleid niet.

De Zoom Web App is live gevalideerd met een officiële Zoom-testvergadering voor het tussenscherm van de app, het invoeren van de gastnaam in een iframe, microfoon- en camerabediening vóór deelname, deelname, mediarechten van de browser en macOS, detectie tijdens het gesprek, inschakeling van live-ondertiteling en detectie van beëindiging door de host. Wachtruimte- en authenticatiestatussen zijn afhankelijk van het hostbeleid en behouden tekstuele terugvalopties wanneer geen stabiele DOM-id beschikbaar is.

## Tool- en Gateway-oppervlak

De agenttool `zoom_meetings` ondersteunt `join`, `leave`, `status`, `transcript` en `speak`. Gateway-methoden gebruiken het voorvoegsel `zoommeetings.*`. De Node-opdracht is `zoommeetings.chrome`.

## Gerelateerd

- [Overzicht van vergaderplugins](/plugins/meeting-plugins)
