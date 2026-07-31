---
read_when:
    - Je wilt dat een OpenClaw-agent deelneemt aan een Microsoft Teams-vergadering
    - Je configureert Chrome, BlackHole of SoX voor terugspreken in Teams-vergaderingen
summary: 'Microsoft Teams-vergaderingsplugin: neem als gast via de Chrome-browser deel aan werk- of consumentenvergaderingen'
title: Microsoft Teams-vergaderingenplugin
x-i18n:
    generated_at: "2026-07-27T05:11:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6f84e58d478185d026dd79a02a8500af48f51689ef6865d56badb0e27c6d2814
    source_path: plugins/teams-meetings.md
    workflow: 16
---

De `teams-meetings`-plugin neemt als gast deel via Microsoft Teams-links in het OpenClaw Chrome-profiel. De plugin accepteert werklinks onder `teams.microsoft.com/l/meetup-join/...` en consumentenlinks onder `teams.live.com/meet/...`. De plugin maakt geen vergaderingen aan, belt niet in, roept Microsoft Graph niet aan en legt geen audio- of video-opnamen vast.

## Installatie

Terugspreken gebruikt dezelfde lokale audiovereisten als de [Google Meet-plugin](/nl/plugins/google-meet): macOS, het virtuele audioapparaat `BlackHole 2ch` en SoX.

```bash
brew install blackhole-2ch sox
sudo reboot
system_profiler SPAudioDataType | grep -i BlackHole
command -v sox
```

De plugin is standaard inbegrepen en ingeschakeld. Voeg alleen een vermelding toe om de plugin aan te passen en controleer daarna de installatie:

```json5
{
  plugins: {
    entries: {
      "teams-meetings": {
        config: {
          defaultMode: "agent",
          chrome: { guestName: "OpenClaw Agent" },
        },
      },
    },
  },
}
```

Voer `openclaw plugins disable teams-meetings` uit als je de plugin niet actief wilt hebben.

```bash
openclaw teamsmeetings setup
openclaw teamsmeetings join 'https://teams.microsoft.com/l/meetup-join/...'
```

Gebruik `chromeNode.node` om Chrome, BlackHole en SoX op een gekoppelde macOS-node uit te voeren. De node moet `teamsmeetings.chrome` en `browser.proxy` toestaan.

## Modi

| Modus         | Gedrag                                                                    |
| ------------ | --------------------------------------------------------------------------- |
| `agent`      | Realtime transcriptie raadpleegt de geconfigureerde OpenClaw-agent; TTS antwoordt. |
| `bidi`       | Een realtime spraakmodel luistert en antwoordt rechtstreeks.                        |
| `transcribe` | Alleen observerend deelnemen met momentopnamen van het live-ondertitelingstranscript.                   |

Liveondertiteling van Teams wordt na toelating in elke modus ingeschakeld, zodat OpenClaw
notities met sprekerstoewijzing kan bewaren. De actie `transcript` retourneert nog steeds
alleen de begrensde livebuffer voor `transcribe`-sessies. Bij het verlaten slaat OpenClaw
het duurzame transcript en de afgeleide samenvatting op in de gedeelde statusdatabase; geef
ze weer of exporteer ze met [`openclaw transcripts`](/nl/cli/transcripts).

Automatische notities zijn standaard ingeschakeld. Stel `transcripts.enabled: false` in om
duurzame notities globaal uit te schakelen; de expliciete modus `transcribe` stelt nog steeds alleen
het begrensde live-uiteinde beschikbaar.

## Beperkingen voor deelname als gast

De browseradapter sluit het tussenscherm van de app, vult de gastnaam in, schakelt de camera uit, configureert de microfoon voor de geselecteerde modus en klikt op de knop om deel te nemen. Tijdens een gesprek wordt het besturingselement voor ophangen gebruikt; statussen voor de lobby, aanmelding bij de tenant en apparaatmachtigingen retourneren expliciete redenen voor handmatige actie. Omleidingen van de consumentenlauncher voor vergaderingen en de door Chrome weergegeven `BlackHole 2ch (Virtual)`-labels worden ondersteund.

Teams-tenantbeleid kan aanmelding, e-mailverificatie of toelating door de organisator vereisen. Voltooi die stap in het OpenClaw Chrome-profiel en probeer daarna opnieuw de status op te vragen of spraak te gebruiken. De plugin omzeilt het tenantbeleid niet.

De Teams-webclient voor consumenten is live gevalideerd voor het tussenscherm van de app, invoer van de gastnaam, microfoon-/cameraschakelaars vóór deelname, deelname, toelating vanuit de lobby, mediamachtigingen, detectie van een actief gesprek, liveondertiteling, routering van BlackHole-invoer/-uitvoer, verlaten en detectie na het gesprek. Werktenants kunnen ander beleid opleggen voor aanmelding, e-mailverificatie, toelating en bevestiging bij het verlaten; voltooi elke gemelde handmatige actie in het OpenClaw Chrome-profiel.

## Tool- en Gateway-oppervlak

De agenttool `teams_meetings` ondersteunt `join`, `leave`, `status`, `transcript` en `speak`. Gateway-methoden gebruiken het voorvoegsel `teamsmeetings.*`. De nodeopdracht is `teamsmeetings.chrome`.

## Gerelateerd

- [Overzicht van vergaderplugins](/plugins/meeting-plugins)
- [Microsoft Teams-kanaal](/nl/channels/msteams)
