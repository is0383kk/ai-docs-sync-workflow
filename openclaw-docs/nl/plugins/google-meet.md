---
read_when:
    - Je wilt dat een OpenClaw-agent deelneemt aan een Google Meet-gesprek
    - Je wilt dat een OpenClaw-agent een nieuw Google Meet-gesprek maakt
    - Je configureert Chrome, Chrome node of Twilio als transport voor Google Meet
summary: 'Google Meet-plugin: neem via Chrome of Twilio deel aan expliciete Meet-URL''s met standaardinstellingen waarmee de agent kan terugpraten'
title: Google Meet-plugin
x-i18n:
    generated_at: "2026-07-27T05:12:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8a611e283fe900984a29b563969936a641c7af430b05933eb03b98dc93b5d0c8
    source_path: plugins/google-meet.md
    workflow: 16
---

De Plugin `google-meet` neemt namens een OpenClaw-agent deel via expliciete Meet-URL's. De functie is bewust beperkt:

- De Plugin neemt alleen deel via `https://meet.google.com/...`-URL's; de Plugin belt nooit in bij een vergadering via een telefoonnummer dat de Plugin zelf vindt.
- `googlemeet create` kan via de Google Meet-API (of een browserfallback) een nieuwe Meet-URL genereren en neemt er standaard aan deel.
- Voor deelname via Chrome wordt een aangemeld Chrome-profiel gebruikt, eventueel op een gekoppelde Node. Voor deelname via Twilio wordt via de [Plugin voor spraakoproepen](/nl/plugins/voice-call) een telefoonnummer plus pincode/DTMF gebeld; hiermee kan niet rechtstreeks naar een Meet-URL worden gebeld.
- `mode: "agent"` (standaard) transcribeert de spraak van deelnemers met een realtimeprovider, stuurt deze naar de geconfigureerde OpenClaw-agent en spreekt het antwoord uit met de reguliere OpenClaw-TTS. Met `mode: "bidi"` kan een realtime spraakmodel rechtstreeks antwoorden. Met `mode: "transcribe"` wordt alleen als waarnemer deelgenomen, zonder terugspraak.
- Er wordt niet automatisch een toestemmingsmelding afgespeeld wanneer de Plugin aan een gesprek deelneemt.
- De CLI-opdracht is `googlemeet`; `meet` is gereserveerd voor bredere workflows voor telefonische vergaderingen van agents.

## Snel aan de slag

Installeer de Plugin en de lokale audioafhankelijkheden en stel vervolgens een sleutel voor een realtimeprovider in. OpenAI is de standaardprovider voor transcriptie in de modus `agent`; Google Gemini Live is beschikbaar als spraakprovider voor de modus `bidi`:

```bash
openclaw plugins install npm:@openclaw/google-meet
brew install blackhole-2ch sox
export OPENAI_API_KEY=sk-...
# alleen nodig wanneer realtime.voiceProvider "google" is voor de bidirectionele modus
export GEMINI_API_KEY=...
```

`blackhole-2ch` installeert het virtuele audioapparaat `BlackHole 2ch` waar Chrome audio doorheen stuurt. Het installatieprogramma van Homebrew vereist een herstart voordat macOS het apparaat beschikbaar stelt:

```bash
sudo reboot
```

Controleer na de herstart beide onderdelen:

```bash
system_profiler SPAudioDataType | grep -i BlackHole
command -v sox
```

De Plugin wordt na installatie standaard ingeschakeld. Voeg alleen een vermelding toe om de configuratie aan te passen:

```json5
{
  plugins: {
    entries: {
      "google-meet": {
        config: {},
      },
    },
  },
}
```

Voer `openclaw plugins disable google-meet` uit als je de Plugin niet actief wilt hebben.

Controleer de configuratie en neem vervolgens deel:

```bash
openclaw googlemeet setup
openclaw googlemeet join https://meet.google.com/abc-defg-hij
```

De uitvoer van `setup` is leesbaar voor agents en houdt rekening met de modus en het transport: de uitvoer vermeldt het Chrome-profiel, de vastgezette Node en, voor realtime deelname via Chrome, de BlackHole/SoX-audiobrug en de controle voor de vertraagde introductie. Bij deelname alleen als waarnemer worden realtimevereisten overgeslagen:

```bash
openclaw googlemeet setup --transport chrome-node --mode transcribe
```

Wanneer delegatie aan Twilio is geconfigureerd, meldt `setup` ook of `voice-call`, de Twilio-inloggegevens en de openbare beschikbaarheid van de Webhook gereed zijn. Beschouw elke controle met `ok: false` als een blokkade voor die combinatie van transport en modus voordat een agent deelneemt. Gebruik `--json` voor machineleesbare uitvoer en `--transport chrome|chrome-node|twilio` om een specifiek transport vooraf te controleren:

```bash
openclaw googlemeet setup --transport twilio
```

Of laat een agent deelnemen via de tool `google_meet`:

```json
{
  "action": "join",
  "url": "https://meet.google.com/abc-defg-hij",
  "transport": "chrome-node",
  "mode": "agent"
}
```

Op Gateway-hosts zonder macOS blijft `google_meet` zichtbaar voor acties voor artefacten, agenda's, configuratie, transcriptie, Twilio en `chrome-node`, maar lokale terugspraak via Chrome (`transport: "chrome"` met `mode: "agent"` of `"bidi"`) wordt geblokkeerd voordat de audiobrug wordt bereikt, omdat dat pad momenteel afhankelijk is van `BlackHole 2ch` van macOS. Gebruik in plaats daarvan `mode: "transcribe"`, inbellen via Twilio of een macOS-host met `chrome-node`.

### Een vergadering maken

```bash
openclaw googlemeet create --transport chrome-node --mode agent
openclaw googlemeet create --no-join
```

`create` heeft twee paden, die worden vermeld in het veld `source` van het resultaat:

- **`api`**: wordt gebruikt wanneer OAuth-inloggegevens voor Google Meet zijn geconfigureerd. Deterministisch; niet afhankelijk van de status van de browserinterface.
- **`browser`**: wordt gebruikt zonder OAuth-inloggegevens. OpenClaw opent `https://meet.google.com/new` op de vastgezette Chrome-Node en wacht totdat Google doorstuurt naar een echte URL met een vergaderingscode; het OpenClaw Chrome-profiel op die Node moet al bij Google zijn aangemeld. Zowel deelnemen als maken hergebruiken een bestaand Meet-tabblad (of een tabblad met een lopende `.../new`- of Google-accountprompt) voordat een nieuw tabblad wordt geopend; bij het vergelijken van tabbladen worden onschuldige queryreeksen zoals `authuser` genegeerd.

`create` neemt standaard deel en retourneert `joined: true` plus de deelnamesessie. Geef `--no-join` (CLI) of `"join": false` (tool) door om alleen de URL te genereren.

Stel voor kamers die via de API worden gemaakt een expliciet toegangsbeleid in, in plaats van de standaardinstelling van het Google-account over te nemen:

```bash
openclaw googlemeet create --access-type OPEN --transport chrome-node --mode agent
```

| `--access-type` | Wie zonder toelatingsverzoek kan deelnemen                              |
| --------------- | ----------------------------------------------------------------------- |
| `OPEN`          | Iedereen met de Meet-URL                                                |
| `TRUSTED`       | Vertrouwde gebruikers van de organisatie van de host, uitgenodigde externe gebruikers en inbelgebruikers |
| `RESTRICTED`    | Alleen genodigden                                                       |

Dit is alleen van toepassing op kamers die via de API worden gemaakt, dus OAuth moet zijn geconfigureerd. Als je je hebt geauthenticeerd voordat deze optie bestond, voer je `openclaw googlemeet auth login --json` opnieuw uit nadat je het bereik `meetings.space.settings` aan je OAuth-toestemmingsscherm hebt toegevoegd.

Als de browserfallback wordt geblokkeerd door een Google-aanmelding of Meet-machtiging, retourneert de tool `manualActionRequired: true` met `manualActionReason`, `manualActionMessage` en de `browser.nodeId`/`browser.targetId`/`browserUrl`. Meld dat bericht en open geen nieuwe Meet-tabbladen totdat de beheerder de browserstap heeft voltooid.

### Alleen als waarnemer deelnemen

Stel `"mode": "transcribe"` in om de bidirectionele realtimebrug over te slaan (geen BlackHole/SoX vereist en geen terugspraak). Bij deelname via Chrome in de transcriptiemodus worden ook de machtiging voor de microfoon/camera van OpenClaw en het Meet-pad **Use microphone** overgeslagen; als Meet het tussenscherm voor audiokeuze toont, probeert de automatisering eerst **Continue without microphone**. Beheerde Chrome-transporten installeren in elke modus naar beste vermogen een waarnemer voor Meet-ondertiteling, zodat duurzame notities beschikbaar zijn zonder het pad voor live raadpleging van de agent te wijzigen. `googlemeet status --json` en `googlemeet doctor` melden `captioning`, `captionsEnabledAttempted`, `transcriptLines`, `lastCaptionAt`, `lastCaptionSpeaker`, `lastCaptionText` en een `recentTranscript`-staart.

Lees voor het begrensde sessietranscript het exact bijgehouden Meet-tabblad:

```bash
openclaw googlemeet transcript <session-id>
openclaw googlemeet transcript <session-id> --since <next-index> --json
```

De waarnemer bewaart maximaal 2.000 voltooide ondertitelregels op de Meet-pagina. Zichtbare voortschrijdende tekst blijft in de gezondheidsstaart van de status totdat de ondertitelregel is voltooid, zodat het opslaan van `nextIndex` een latere tekstuitbreiding niet kan overslaan; bij het verlaten worden zichtbare regels voltooid voordat de momentopname wordt gemaakt. `droppedLines` meldt regels die aan het begin verloren zijn gegaan wanneer de limiet wordt overschreden. De begrensde `googlemeet transcript`-staart bewaart nog steeds alleen de vier laatst beëindigde sessies en wordt opnieuw ingesteld met de Gateway. Daarnaast voegt OpenClaw gedurende de vergadering voltooide ondertitelregels toe aan de gedeelde statusdatabase en schrijft het bij vertrek een afgeleide samenvatting. Gebruik [`openclaw transcripts`](/nl/cli/transcripts) om deze duurzame notities te bekijken of te exporteren.

Automatische notities zijn standaard ingeschakeld. Stel `transcripts.enabled: false` in om
duurzame notities globaal uit te schakelen; de expliciete modus `transcribe` stelt nog steeds alleen
de begrensde live-staart beschikbaar. Twilio-deelnames hebben geen browserstroom voor ondertiteling en
worden via dit pad niet vastgelegd.

Voor een ja/nee-luistertest:

```bash
openclaw googlemeet test-listen <meet-url> --transport chrome-node
```

Deze neemt deel in de transcriptiemodus, wacht op nieuwe beweging in ondertiteling/transcriptie en retourneert `listenVerified`, `listenTimedOut`, velden voor handmatige acties en de huidige status van de ondertiteling.

### Status van de realtimesessie

Tijdens sessies met terugspraak meldt de status van `google_meet` de status van Chrome/de audiobrug: `inCall`, `manualActionRequired`, `providerConnected`, `realtimeReady`, `audioInputActive`, `audioOutputActive`, tijdstempels van de laatste invoer/uitvoer, bytetellers en de gesloten status van de brug. Beheerde Chrome-sessies spreken de introductie-/testzin pas uit nadat de status `inCall: true` meldt; anders `speechReady: false` en wordt de spraakpoging geblokkeerd in plaats van stilzwijgend niets te doen.

Lokale deelname via Chrome verloopt via het aangemelde OpenClaw-browserprofiel en vereist `BlackHole 2ch` voor het microfoon-/luidsprekerpad. Eén BlackHole-apparaat is voldoende voor een eerste rooktest, maar kan echo veroorzaken; gebruik afzonderlijke virtuele apparaten of een Loopback-achtige graaf voor zuivere bidirectionele audio.

## Lokale Gateway + Chrome in Parallels

Een volledige Gateway of API-sleutel voor een model is niet vereist in een macOS-VM als deze alleen Chrome beschikbaar hoeft te stellen. Voer de Gateway en agent lokaal uit; voer een Node-host uit in de VM.

| Wordt uitgevoerd op  | Wat                                                                                             |
| -------------------- | ----------------------------------------------------------------------------------------------- |
| Gateway-host         | OpenClaw Gateway, agentwerkruimte, model-/API-sleutels, realtimeprovider, configuratie van de Google Meet-Plugin |
| Parallels-macOS-VM   | OpenClaw CLI/Node-host, Chrome, SoX, BlackHole 2ch, een bij Google aangemeld Chrome-profiel      |
| Niet nodig in de VM  | Gateway-service, agentconfiguratie, configuratie van modelprovider                              |

Installeer de VM-afhankelijkheden, start opnieuw op en controleer:

```bash
brew install blackhole-2ch sox
sudo reboot
system_profiler SPAudioDataType | grep -i BlackHole
command -v sox
```

Installeer de Plugin in de VM, waar deze standaard wordt ingeschakeld, en start de Node-host:

```bash
openclaw plugins install npm:@openclaw/google-meet
openclaw node run --host <gateway-host> --port 18789 --display-name parallels-macos
```

Als `<gateway-host>` een LAN-IP-adres zonder TLS is, meld je dan expliciet aan voor dat vertrouwde privénetwerk:

```bash
OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1 \
  openclaw node run --host <gateway-lan-ip> --port 18789 --display-name parallels-macos
```

Gebruik dezelfde vlag bij installatie als LaunchAgent (het is een procesomgevingsvariabele die in de LaunchAgent-omgeving wordt opgeslagen wanneer deze bij de installatieopdracht aanwezig is, geen instelling van `openclaw.json`):

```bash
OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1 \
  openclaw node install --host <gateway-lan-ip> --port 18789 --display-name parallels-macos --force
openclaw node restart
```

Keur de Node goed vanaf de Gateway-host en controleer vervolgens of deze zowel `googlemeet.chrome` als browserfunctionaliteit/`browser.proxy` aanbiedt:

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw nodes status
```

Leid Meet via die Node:

```json5
{
  gateway: {
    nodes: {
      commands: { allow: ["googlemeet.chrome", "browser.proxy"] },
    },
  },
  plugins: {
    entries: {
      "google-meet": {
        enabled: true,
        config: {
          defaultTransport: "chrome-node",
          chrome: {
            guestName: "OpenClaw Agent",
            autoJoin: true,
            reuseExistingTab: true,
          },
          chromeNode: {
            node: "parallels-macos",
          },
        },
      },
    },
  },
}
```

Neem nu op de normale manier deel vanaf de Gateway-host:

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij
```

Voor een rooktest met één opdracht die een sessie maakt of hergebruikt, een bekende zin uitspreekt en de sessiestatus afdrukt:

```bash
openclaw googlemeet test-speech https://meet.google.com/abc-defg-hij
```

Tijdens realtime deelname vult browserautomatisering de gastnaam in, klikt op Join/Ask to join en accepteert de prompt 'Use microphone' van Meet bij de eerste uitvoering wanneer deze verschijnt (of 'Continue without microphone' tijdens deelname in alleen-observerenmodus en het uitsluitend via de browser maken van vergaderingen). Als het profiel is afgemeld, Meet wacht op toelating door de host, Chrome toestemming voor microfoon/camera nodig heeft of Meet vastzit bij een onopgeloste prompt, rapporteert het resultaat `manualActionRequired: true` met `manualActionReason` en `manualActionMessage`. Stop met opnieuw proberen, rapporteer dat bericht plus `browserUrl`/`browserTitle` en probeer het pas opnieuw nadat de handmatige actie is voltooid.

Als `chromeNode.node` is weggelaten, selecteert OpenClaw alleen automatisch wanneer precies één verbonden Node zowel `googlemeet.chrome` als browserbesturing aanbiedt; leg `chromeNode.node` vast (Node-id, weergavenaam of extern IP-adres) wanneer meerdere geschikte Nodes zijn verbonden.

### Controles voor veelvoorkomende fouten

| Symptoom                                                 | Oplossing                                                                                                                                                                                                                                                                             |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Configured Google Meet node ... is not usable: offline` | De vastgelegde Node is bekend maar niet beschikbaar. Rapporteer de blokkade in de installatie; val niet stilzwijgend terug op een ander transport, tenzij daarom wordt gevraagd.                                                                                                       |
| `No connected Google Meet-capable node`                  | Installeer `npm:@openclaw/google-meet` in de VM, voer `openclaw plugins enable browser` uit, start `openclaw node run` en keur de koppeling goed. Als Google Meet expliciet was uitgeschakeld, schakel je het ook in. Controleer of `gateway.nodes.commands.allow` `googlemeet.chrome` en `browser.proxy` bevat. |
| `BlackHole 2ch audio device not found`                   | Installeer `blackhole-2ch` op de host die wordt gecontroleerd en start deze opnieuw op.                                                                                                                                                                                             |
| `BlackHole 2ch audio device not found on the node`       | Installeer `blackhole-2ch` in de VM en start de VM opnieuw op.                                                                                                                                                                                                                       |
| Chrome wordt geopend maar kan niet deelnemen             | Meld je aan bij het browserprofiel in de VM of laat `chrome.guestName` ingesteld. Automatische deelname als gast gebruikt OpenClaw-browserautomatisering via de browserproxy van de Node; laat `browser.defaultProfile` van de Node (of een benoemd profiel voor een bestaande sessie) verwijzen naar het gewenste profiel. |
| Dubbele Meet-tabbladen                                   | Laat `chrome.reuseExistingTab: true`. OpenClaw activeert een bestaand tabblad voor dezelfde URL en bij het maken wordt een lopend `.../new`- of Google-accountprompttabblad hergebruikt voordat een ander tabblad wordt geopend.                                                              |
| Geen audio                                               | Leid de microfoon/luidspreker van Meet via het virtuele audiopad dat OpenClaw gebruikt; gebruik afzonderlijke virtuele apparaten of Loopback-achtige routering voor zuivere duplexaudio.                                                                                                |

## Installatie-opmerkingen

De standaardinstelling voor terugspreken via Chrome gebruikt twee externe tools die OpenClaw niet bundelt of distribueert; installeer ze als hostafhankelijkheden via Homebrew:

- `sox`: audiohulpprogramma voor de opdrachtregel. De Plugin geeft expliciete CoreAudio-apparaatopdrachten voor de standaard 24 kHz PCM16-audiobrug.
- `blackhole-2ch`: virtueel audiostuurprogramma voor macOS dat het `BlackHole 2ch`-apparaat levert waar Chrome/Meet via wordt gerouteerd.

SoX heeft de licentie `LGPL-2.0-only AND GPL-2.0-only`; BlackHole is GPL-3.0. Als je een installatieprogramma of appliance bouwt waarin BlackHole met OpenClaw wordt gebundeld, controleer dan de upstreamlicentie van BlackHole of verkrijg een afzonderlijke licentie van Existential Audio.

## Transporten

| Transport     | Gebruiken wanneer                                                                            |
| ------------- | -------------------------------------------------------------------------------------------- |
| `chrome`      | Chrome/audio op de Gateway-host draaien                                                      |
| `chrome-node` | Chrome/audio op een gekoppelde Node draaien (bijvoorbeeld een Parallels-macOS-VM)             |
| `twilio`      | Telefonische inbelterugval via de Voice Call-Plugin, wanneer deelname via Chrome niet beschikbaar is |

### Chrome

Opent de Meet-URL via OpenClaw-browserbesturing en neemt deel als het aangemelde OpenClaw-browserprofiel. Op macOS controleert de Plugin vóór het starten op `BlackHole 2ch` en voert deze, indien geconfigureerd, een opdracht voor statuscontrole/het starten van de audiobrug uit voordat Chrome wordt geopend. Kies voor lokale Chrome het profiel met `browser.defaultProfile`; `chrome.browserProfile` wordt in plaats daarvan doorgegeven aan `chrome-node`-hosts.

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij --transport chrome
openclaw googlemeet join https://meet.google.com/abc-defg-hij --transport chrome-node
```

De audio van de Chrome-microfoon/luidspreker wordt via de lokale OpenClaw-audiobrug geleid. Als `BlackHole 2ch` niet is geïnstalleerd, mislukt deelname met een installatiefout in plaats van zonder audiopad deel te nemen.

### Twilio

Een strikt belplan dat is gedelegeerd aan de [Voice Call-Plugin](/nl/plugins/voice-call). Het parseert Meet-pagina's niet op telefoonnummers; Google Meet moet een telefonisch inbelnummer en een pincode voor de vergadering beschikbaar stellen.

Schakel Voice Call in op de Gateway-host, niet op de Chrome-Node:

```json5
{
  plugins: {
    allow: ["google-meet", "voice-call", "google"],
    entries: {
      "google-meet": {
        enabled: true,
        config: {
          defaultTransport: "chrome-node",
          // of stel "twilio" in als Twilio de standaard moet zijn
        },
      },
      "voice-call": {
        enabled: true,
        config: {
          provider: "twilio",
          inboundPolicy: "allowlist",
          realtime: {
            enabled: true,
            provider: "google",
            instructions: "Neem als OpenClaw-agent deel aan deze Google Meet. Houd het kort.",
            toolPolicy: "safe-read-only",
            providers: {
              google: {
                silenceDurationMs: 500,
                startSensitivity: "high",
              },
            },
          },
        },
      },
      google: {
        enabled: true,
      },
    },
  },
}
```

Geef Twilio-referenties via de omgeving op om geheimen buiten `openclaw.json` te houden:

```bash
export TWILIO_ACCOUNT_SID=AC...
export TWILIO_AUTH_TOKEN=...
export TWILIO_FROM_NUMBER=+15550001234
export GEMINI_API_KEY=...
```

Gebruik in plaats daarvan `realtime.provider: "openai"` met `OPENAI_API_KEY` als OpenAI de realtime spraakprovider is.

Start of herlaad de Gateway nadat je `voice-call` hebt ingeschakeld; wijzigingen in de Plugin-configuratie worden pas na herladen van kracht. Controleer:

```bash
openclaw config validate
openclaw plugins list | grep -E 'google-meet|voice-call'
openclaw googlemeet setup
```

Wanneer Twilio-delegatie is aangesloten, bevat `googlemeet setup` controles voor `twilio-voice-call-plugin`, `twilio-voice-call-credentials` en `twilio-voice-call-webhook`.

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij \
  --transport twilio \
  --dial-in-number +15551234567 \
  --pin 123456
```

Gebruik `--dtmf-sequence` voor een aangepaste reeks, met vooraan `w` of komma's voor een pauze vóór de pincode:

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij \
  --transport twilio \
  --dial-in-number +15551234567 \
  --dtmf-sequence ww123456#
```

## OAuth en preflight

OAuth is optioneel voor het maken van een Meet-link, omdat `googlemeet create` kan terugvallen op browserautomatisering. Configureer OAuth voor het maken via de officiële API, ruimteresolutie of preflight van de Meet Media API. Deelnames via Chrome/Chrome-Node zijn nooit afhankelijk van OAuth; ze gebruiken hoe dan ook een aangemeld Chrome-profiel, BlackHole/SoX en (voor `chrome-node`) een verbonden Node.

### Google-referenties maken

In Google Cloud Console:

<Steps>
<Step title="Een project maken of selecteren">
</Step>
<Step title="De Google Meet REST API inschakelen">
</Step>
<Step title="Het OAuth-toestemmingsscherm configureren">
Internal is het eenvoudigst voor een Google Workspace-organisatie. External werkt voor persoonlijke/testinstallaties; zolang de app zich in Testing bevindt, voeg je elk Google-account dat de app zal autoriseren toe als testgebruiker.
</Step>
<Step title="De gevraagde bereiken toevoegen">
- `https://www.googleapis.com/auth/meetings.space.created`
- `https://www.googleapis.com/auth/meetings.space.readonly`
- `https://www.googleapis.com/auth/meetings.space.settings`
- `https://www.googleapis.com/auth/meetings.conference.media.readonly`
- `https://www.googleapis.com/auth/calendar.events.readonly` (agenda opzoeken)
- `https://www.googleapis.com/auth/drive.meet.readonly` (export van de hoofdtekst van transcript/slimme notitie)

</Step>
<Step title="Een OAuth-client-ID maken">
Applicatietype **Web application**. Geautoriseerde omleidings-URI:

```text
http://localhost:8085/oauth2callback
```

</Step>
<Step title="Het client-ID en clientgeheim kopiëren">
</Step>
</Steps>

`meetings.space.created` is vereist door `spaces.create`. `meetings.space.readonly` zet Meet-URL's/-codes om naar ruimtes. Met `meetings.space.settings` kan OpenClaw `SpaceConfig`-instellingen zoals `accessType` doorgeven bij het maken van een ruimte via de API. `meetings.conference.media.readonly` is voor preflight en mediawerk met de Meet Media API; Google kan inschrijving voor Developer Preview vereisen voor daadwerkelijk gebruik van de Media API. `calendar.events.readonly` is alleen nodig voor het opzoeken in de agenda via `--today`/`--event`. `drive.meet.readonly` is alleen nodig voor `--include-doc-bodies`-export. Als je alleen via de browser aan Chrome-sessies wilt deelnemen, sla OAuth dan volledig over.

### Het vernieuwingstoken aanmaken

Configureer `oauth.clientId` en optioneel `oauth.clientSecret` (of geef ze door als omgevingsvariabelen) en voer vervolgens uit:

```bash
openclaw googlemeet auth login --json
```

Dit voert een PKCE-flow uit met een localhost-callback op `http://localhost:8085/oauth2callback` en drukt een `oauth`-configuratieblok met een vernieuwingstoken af. Voeg `--manual` toe voor een kopieer-en-plakflow wanneer de browser de lokale callback niet kan bereiken:

```bash
OPENCLAW_GOOGLE_MEET_CLIENT_ID="your-client-id" \
OPENCLAW_GOOGLE_MEET_CLIENT_SECRET="your-client-secret" \
openclaw googlemeet auth login --json --manual
```

JSON-uitvoer:

```json
{
  "oauth": {
    "clientId": "your-client-id",
    "clientSecret": "your-client-secret",
    "refreshToken": "refresh-token",
    "accessToken": "access-token",
    "expiresAt": 1770000000000
  },
  "scope": "..."
}
```

Sla het `oauth`-object op onder de Plugin-configuratie:

```json5
{
  plugins: {
    entries: {
      "google-meet": {
        enabled: true,
        config: {
          oauth: {
            clientId: "your-client-id",
            clientSecret: "your-client-secret",
            refreshToken: "refresh-token",
          },
        },
      },
    },
  },
}
```

Geef de voorkeur aan omgevingsvariabelen als je het vernieuwingstoken niet in de configuratie wilt opnemen; eerst wordt de configuratie verwerkt, daarna de omgeving als terugval. Als je je hebt geauthenticeerd voordat ondersteuning voor het maken van vergaderingen, het opzoeken in de agenda of het exporteren van de documenttekst bestond, voer je `openclaw googlemeet auth login --json` opnieuw uit zodat het vernieuwingstoken de huidige set bereiken dekt.

### OAuth controleren met doctor

```bash
openclaw googlemeet doctor --oauth --json
```

Dit controleert of de OAuth-configuratie bestaat en of met de vernieuwingstoken een toegangstoken kan worden aangemaakt, zonder de Chrome-runtime te laden of een verbonden Node te vereisen. Het rapport bevat alleen statusvelden (`ok`, `configured`, `tokenSource`, `expiresAt`, controlemeldingen) en geeft nooit de toegangstoken, vernieuwingstoken of het clientgeheim weer.

| Controle             | Betekenis                                                                        |
| -------------------- | -------------------------------------------------------------------------------- |
| `oauth-config`       | `oauth.clientId` plus `oauth.refreshToken`, of een gecachte toegangstoken, is aanwezig |
| `oauth-token`        | De gecachte toegangstoken is nog geldig of met de vernieuwingstoken is een nieuwe aangemaakt |
| `meet-spaces-get`    | De optionele controle `--meeting` heeft een bestaande Meet-ruimte gevonden |
| `meet-spaces-create` | De optionele controle `--create-space` heeft een nieuwe Meet-ruimte gemaakt |

Toon met de aanmaakcontrole met neveneffecten aan dat de Meet-API is ingeschakeld en het bereik `spaces.create` beschikbaar is:

```bash
openclaw googlemeet doctor --oauth --create-space --json
```

Toon leestoegang tot een bestaande ruimte aan:

```bash
openclaw googlemeet doctor --oauth --meeting https://meet.google.com/abc-defg-hij --json
openclaw googlemeet resolve-space --meeting https://meet.google.com/abc-defg-hij
```

Een `403` bij deze controles betekent meestal dat de Meet REST-API is uitgeschakeld, dat de vernieuwingstoken het vereiste bereik mist of dat het Google-account geen toegang tot die ruimte heeft. Een fout met de vernieuwingstoken betekent dat je `openclaw googlemeet auth login --json` opnieuw moet uitvoeren en het nieuwe blok `oauth` moet opslaan.

Voor de browserfallback is geen OAuth nodig; Google-authenticatie is daar afkomstig van het aangemelde Chrome-profiel op de geselecteerde Node, niet van de OpenClaw-configuratie.

Deze omgevingsvariabelen worden als fallback geaccepteerd:

- `OPENCLAW_GOOGLE_MEET_CLIENT_ID` of `GOOGLE_MEET_CLIENT_ID`
- `OPENCLAW_GOOGLE_MEET_CLIENT_SECRET` of `GOOGLE_MEET_CLIENT_SECRET`
- `OPENCLAW_GOOGLE_MEET_REFRESH_TOKEN` of `GOOGLE_MEET_REFRESH_TOKEN`
- `OPENCLAW_GOOGLE_MEET_ACCESS_TOKEN` of `GOOGLE_MEET_ACCESS_TOKEN`
- `OPENCLAW_GOOGLE_MEET_ACCESS_TOKEN_EXPIRES_AT` of `GOOGLE_MEET_ACCESS_TOKEN_EXPIRES_AT`
- `OPENCLAW_GOOGLE_MEET_DEFAULT_MEETING` of `GOOGLE_MEET_DEFAULT_MEETING`
- `OPENCLAW_GOOGLE_MEET_PREVIEW_ACK` of `GOOGLE_MEET_PREVIEW_ACK`

### Artefacten vinden, vooraf controleren en lezen

```bash
openclaw googlemeet resolve-space --meeting https://meet.google.com/abc-defg-hij
openclaw googlemeet preflight --meeting https://meet.google.com/abc-defg-hij
```

Nadat Meet conferentierecords heeft gemaakt:

```bash
openclaw googlemeet artifacts --meeting https://meet.google.com/abc-defg-hij
openclaw googlemeet attendance --meeting https://meet.google.com/abc-defg-hij
openclaw googlemeet export --meeting https://meet.google.com/abc-defg-hij --output ./meet-export
```

Met `--meeting` gebruiken `artifacts` en `attendance` standaard het nieuwste conferentierecord; geef `--all-conference-records` door voor elk bewaard record.

Bij het opzoeken in de agenda wordt de vergaderings-URL eerst via Google Calendar gevonden voordat artefacten worden gelezen (hiervoor is een vernieuwingstoken vereist die het alleen-lezenbereik voor Calendar-gebeurtenissen bevat):

```bash
openclaw googlemeet latest --today
openclaw googlemeet calendar-events --today --json
openclaw googlemeet artifacts --event "Weekly sync"
openclaw googlemeet attendance --today --format csv --output attendance.csv
```

`--today` doorzoekt de agenda `primary` van vandaag naar een gebeurtenis met een Meet-link; `--event <query>` zoekt in overeenkomende gebeurtenistekst; `--calendar <id>` richt zich op een niet-primaire agenda. `calendar-events` toont een voorbeeld van overeenkomende gebeurtenissen en markeert welke `latest`/`artifacts`/`attendance`/`export` zal kiezen.

Als je de id van het conferentierecord al kent, kun je dit rechtstreeks aanspreken:

```bash
openclaw googlemeet latest --meeting https://meet.google.com/abc-defg-hij
openclaw googlemeet artifacts --conference-record conferenceRecords/abc123 --json
openclaw googlemeet attendance --conference-record conferenceRecords/abc123 --json
```

Sluit de ruimte voor een via de API gemaakte ruimte:

```bash
openclaw googlemeet end-active-conference https://meet.google.com/abc-defg-hij
```

Roept `spaces.endActiveConference` aan en vereist OAuth met het bereik `meetings.space.created` voor een ruimte die het geautoriseerde account kan beheren. Accepteert een Meet-URL, vergaderingscode of `spaces/{id}` en zet deze eerst om naar de ruimteresource van de API. Dit staat los van `googlemeet leave`: `leave` stopt de lokale/sessiedeelname van OpenClaw; `end-active-conference` vraagt Google Meet de actieve conferentie voor de ruimte te beëindigen.

Schrijf een leesbaar rapport:

```bash
openclaw googlemeet artifacts --conference-record conferenceRecords/abc123 \
  --format markdown --output meet-artifacts.md
openclaw googlemeet attendance --conference-record conferenceRecords/abc123 \
  --format csv --output meet-attendance.csv
openclaw googlemeet export --conference-record conferenceRecords/abc123 \
  --include-doc-bodies --zip --output meet-export
openclaw googlemeet export --conference-record conferenceRecords/abc123 \
  --include-doc-bodies --dry-run
```

`artifacts` retourneert metagegevens van het conferentierecord plus metagegevens van deelnemer-, opname-, transcript-, gestructureerde transcriptitem- en slimme-notitieresources wanneer Google deze beschikbaar stelt. `--no-transcript-entries` slaat het opzoeken van items over voor grote vergaderingen. `attendance` breidt deelnemers uit tot deelnemersessierijen met tijdstippen waarop ze voor het eerst en laatst zijn gezien, de totale sessieduur, markeringen voor te laat komen/vroeg vertrekken en dubbele deelnemersresources die op basis van de aangemelde gebruiker of weergavenaam zijn samengevoegd; `--no-merge-duplicates` houdt onbewerkte resources gescheiden, `--late-after-minutes`/`--early-before-minutes` stellen de drempelwaarden af.

`export` schrijft een map met `summary.md`, `attendance.csv`, `transcript.md`, `artifacts.json`, `attendance.json` en `manifest.json`. `manifest.json` registreert de gekozen invoer, exportopties, conferentierecords, uitvoerbestanden, aantallen, tokenbron, eventueel gebruikte Calendar-gebeurtenissen en waarschuwingen over gedeeltelijk ophalen. `--zip` schrijft ook een overdraagbaar archief naast de map. `--include-doc-bodies` exporteert de tekst van gekoppelde Google Docs voor transcripties/slimme notities via Drive `files.export` (hiervoor is het alleen-lezenbereik van Drive Meet vereist); zonder dit bereik bevatten exports alleen Meet-metagegevens en gestructureerde transcriptitems. Bij een gedeeltelijke artefactfout (een fout bij het weergeven van slimme notities, transcriptitems of de documenttekst) blijft de waarschuwing in de samenvatting/het manifest staan in plaats van dat de volledige export mislukt. `--dry-run` haalt dezelfde gegevens op en geeft de JSON van het manifest weer zonder de map of het ZIP-bestand te maken.

Agents gebruiken dezelfde acties via de tool `google_meet` (`export`, `create` met `accessType`, `end_active_conference`, `test_listen`); zie [Tool](#tool).

### Live-rooktest

```bash
OPENCLAW_LIVE_TEST=1 \
OPENCLAW_GOOGLE_MEET_LIVE_MEETING=https://meet.google.com/abc-defg-hij \
pnpm test:live -- extensions/google-meet/google-meet.live.test.ts
```

```bash
openclaw googlemeet setup --transport chrome-node --mode transcribe
openclaw googlemeet test-listen https://meet.google.com/abc-defg-hij --transport chrome-node --timeout-ms 30000
```

| Variabele                                                                                                                 | Doel                                                                   |
| ------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| `OPENCLAW_LIVE_TEST=1`                                                                                                    | Schakelt afgeschermde livetests in                                     |
| `OPENCLAW_GOOGLE_MEET_LIVE_MEETING`                                                                                       | Bewaarde Meet-URL, code of `spaces/{id}`                              |
| `OPENCLAW_GOOGLE_MEET_CLIENT_ID` / `GOOGLE_MEET_CLIENT_ID`                                                                | OAuth-client-id                                                        |
| `OPENCLAW_GOOGLE_MEET_REFRESH_TOKEN` / `GOOGLE_MEET_REFRESH_TOKEN`                                                        | Vernieuwingstoken                                                      |
| `OPENCLAW_GOOGLE_MEET_CLIENT_SECRET`, `OPENCLAW_GOOGLE_MEET_ACCESS_TOKEN`, `OPENCLAW_GOOGLE_MEET_ACCESS_TOKEN_EXPIRES_AT` | Optioneel; dezelfde fallbacknamen zonder het voorvoegsel `OPENCLAW_` werken ook |

Voor de basisrooktest voor artefacten/aanwezigheid zijn `meetings.space.readonly` en `meetings.conference.media.readonly` nodig. Voor opzoeken in de agenda is `calendar.events.readonly` nodig. Voor het exporteren van documenttekst uit Drive is `drive.meet.readonly` nodig.

### Aanmaakvoorbeelden

```bash
openclaw googlemeet create
```

Geeft de nieuwe vergaderings-URI, bron en deelnamesessie weer. Met OAuth wordt de Meet-API gebruikt; zonder OAuth het aangemelde profiel van de vastgezette Chrome-Node. JSON van de browserfallback:

```json
{
  "source": "browser",
  "meetingUri": "https://meet.google.com/abc-defg-hij",
  "joined": true,
  "browser": {
    "nodeId": "ba0f4e4bc...",
    "targetId": "tab-1"
  },
  "join": {
    "session": {
      "id": "meet_...",
      "url": "https://meet.google.com/abc-defg-hij"
    }
  }
}
```

Als de browserfallback eerst een Google-aanmelding of blokkering vanwege Meet-machtigingen tegenkomt, retourneert `google_meet` gestructureerde details in plaats van een gewone tekenreeks:

```json
{
  "source": "browser",
  "error": "google-login-required: Meld je aan bij Google in het OpenClaw-browserprofiel en probeer daarna opnieuw een vergadering te maken.",
  "manualActionRequired": true,
  "manualActionReason": "google-login-required",
  "manualActionMessage": "Meld je aan bij Google in het OpenClaw-browserprofiel en probeer daarna opnieuw een vergadering te maken.",
  "browser": {
    "nodeId": "ba0f4e4bc...",
    "targetId": "tab-1",
    "browserUrl": "https://accounts.google.com/signin",
    "browserTitle": "Sign in - Google Accounts"
  }
}
```

JSON voor aanmaak via de API:

```json
{
  "source": "api",
  "meetingUri": "https://meet.google.com/abc-defg-hij",
  "joined": true,
  "space": {
    "name": "spaces/abc-defg-hij",
    "meetingCode": "abc-defg-hij",
    "meetingUri": "https://meet.google.com/abc-defg-hij"
  },
  "join": {
    "session": {
      "id": "meet_...",
      "url": "https://meet.google.com/abc-defg-hij"
    }
  }
}
```

Bij het maken wordt standaard deelgenomen, maar Chrome/Chrome-Node heeft nog steeds een aangemeld Google-profiel nodig om via de browser deel te nemen; als het profiel is afgemeld, rapporteert OpenClaw `manualActionRequired: true` of een browserfallbackfout en vraagt het de beheerder de Google-aanmelding te voltooien voordat deze het opnieuw probeert.

Stel `preview.enrollmentAcknowledged: true` alleen in nadat je hebt bevestigd dat je Cloud-project, OAuth-principal en deelnemers aan de vergadering zijn ingeschreven voor het Google Workspace Developer Preview Program voor Meet-media-API's.

## Configuratie

Voor het algemene pad van de Chrome-agent hoeven alleen de Plugin, BlackHole, SoX, een sleutel voor een realtimeprovider en een geconfigureerde OpenClaw-TTS-provider te zijn ingeschakeld:

```json5
{
  plugins: {
    entries: {
      "google-meet": {
        enabled: true,
        config: {},
      },
    },
  },
}
```

### Standaardinstellingen

| Sleutel                            | Standaard                                | Opmerkingen                                                                                                                                                                                                       |
| --------------------------------- | ---------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `defaultTransport`                | `"chrome"`                               |                                                                                                                                                                                                                   |
| `defaultMode`                     | `"agent"`                                | `"realtime"` wordt geaccepteerd als verouderde alias voor `"agent"`; nieuwe aanroepers moeten `"agent"` gebruiken                                                                                                                        |
| `chromeNode.node`                 | niet ingesteld                          | Node-id/-naam/-IP voor `chrome-node`; vereist wanneer er meer dan één geschikte Node verbonden kan zijn                                                                                                                      |
| `chrome.launch`                   | `true`                                   | Start Chrome om deel te nemen; stel `false` alleen in wanneer je een reeds geopende sessie hergebruikt                                                                                                                                 |
| `chrome.audioBackend`             | `"blackhole-2ch"`                        |                                                                                                                                                                                                                   |
| `chrome.guestName`                | `"OpenClaw Agent"`                       | Wordt weergegeven op het Meet-gastscherm wanneer je bent afgemeld                                                                                                                                                                         |
| `chrome.autoJoin`                 | `true`                                   | Probeert de gastnaam in te vullen en op Join Now te klikken op `chrome-node`                                                                                                                                                   |
| `chrome.reuseExistingTab`         | `true`                                   | Activeert een bestaand Meet-tabblad in plaats van dubbele tabbladen te openen                                                                                                                                                      |
| `chrome.waitForInCallMs`          | `20000`                                  | Wacht totdat het Meet-tabblad meldt dat het gesprek actief is voordat de terugspreekintro wordt gestart                                                                                                                                          |
| `chrome.audioFormat`              | `"pcm16-24khz"`                          | Audioformaat voor commandoparen; `"g711-ulaw-8khz"` is alleen bedoeld voor verouderde/aangepaste commandoparen die telefonie-audio uitvoeren                                                                                                   |
| `chrome.audioBufferBytes`         | `4096`                                   | SoX-verwerkingsbuffer voor gegenereerde audio-opdrachten van commandoparen (de helft van de standaardbuffer van SoX van 8192 bytes, waardoor de pijplijnlatentie afneemt); waarden worden begrensd op minimaal 17 bytes                                         |
| `chrome.audioInputCommand`        | gegenereerde SoX-opdracht                | Leest van CoreAudio `BlackHole 2ch` en schrijft audio in `chrome.audioFormat`                                                                                                                                        |
| `chrome.audioOutputCommand`       | gegenereerde SoX-opdracht                | Leest audio in `chrome.audioFormat` en schrijft naar CoreAudio `BlackHole 2ch`                                                                                                                                          |
| `chrome.bargeInInputCommand`      | niet ingesteld                          | Optionele lokale microfoonopdracht die ondertekende 16-bits little-endian mono-PCM schrijft om te detecteren wanneer een persoon de assistent tijdens het afspelen onderbreekt; geldt voor de door de Gateway gehoste commandopaarbridge                          |
| `chrome.bargeInRmsThreshold`      | `650`                                    | RMS-niveau dat als menselijke onderbreking wordt beschouwd                                                                                                                                                                           |
| `chrome.bargeInPeakThreshold`     | `2500`                                   | Piekniveau dat als menselijke onderbreking wordt beschouwd                                                                                                                                                                          |
| `chrome.bargeInCooldownMs`        | `900`                                    | Minimale vertraging tussen herhaalde wisacties na een onderbreking                                                                                                                                                                |
| `mode` (per aanvraag)              | `"agent"`                                | Terugspreekmodus; zie de tabel [Agent- en bidi-modi](#agent-and-bidi-modes)                                                                                                                                       |
| `realtime.provider`               | `"openai"`                               | Compatibiliteitsterugval die wordt gebruikt wanneer de onderstaande velden binnen het bereik niet zijn ingesteld                                                                                                                                                |
| `realtime.transcriptionProvider`  | `"openai"`                               | Provider-id die door de modus `agent` wordt gebruikt voor realtime transcriptie                                                                                                                                                       |
| `realtime.voiceProvider`          | niet ingesteld                          | Provider-id die door de modus `bidi` wordt gebruikt voor directe realtime spraak; stel deze in op `"google"` voor Gemini Live terwijl transcriptie in agentmodus OpenAI blijft gebruiken. Combineer met `realtime.model` om het specifieke Gemini Live-model te kiezen. |
| `realtime.toolPolicy`             | `"safe-read-only"`                       | Zie [Agent- en bidi-modi](#agent-and-bidi-modes)                                                                                                                                                                 |
| `realtime.instructions`           | korte instructies voor gesproken antwoorden          | Instrueert het model om kort te spreken en `openclaw_agent_consult` te gebruiken voor uitgebreidere antwoorden                                                                                                                              |
| `realtime.introMessage`           | `"Say exactly: I'm here and listening."` | Wordt eenmaal uitgesproken wanneer de realtime bridge verbinding maakt; stel in op `""` om stil deel te nemen                                                                                                                                       |
| `realtime.agentId`                | `"main"`                                 | OpenClaw-agent-id die wordt gebruikt voor `openclaw_agent_consult`                                                                                                                                                               |
| `voiceCall.enabled`               | `true`                                   | Delegeert het Twilio PSTN-gesprek, DTMF en de introductiebegroeting aan de Voice Call-plugin                                                                                                                                 |
| `voiceCall.dtmfDelayMs`           | `12000`                                  | Wachttijd voorafgaand aan het afspelen van een uit een pincode afgeleide DTMF-reeks via Twilio                                                                                                                                               |
| `voiceCall.postDtmfSpeechDelayMs` | `5000`                                   | Vertraging voordat de realtime introductiebegroeting wordt aangevraagd nadat Voice Call het Twilio-deel heeft gestart                                                                                                                        |

Met `chrome.audioBridgeCommand` en `chrome.audioBridgeHealthCommand` kan een externe bridge het volledige lokale audiopad beheren in plaats van `chrome.audioInputCommand`/`chrome.audioOutputCommand`; zie [Opmerkingen](#notes) voor de beperking op de modus waarin ze kunnen worden gebruikt.

Er bestaat een `openclaw doctor --fix`-migratie voor de verouderde vorm `realtime.provider: "google"`: deze verplaatst die intentie naar `realtime.voiceProvider: "google"` plus `realtime.transcriptionProvider: "openai"` wanneer die velden nog niet zijn ingesteld.

### Optionele overschrijvingen

```json5
{
  defaults: {
    meeting: "https://meet.google.com/abc-defg-hij",
  },
  browser: {
    defaultProfile: "openclaw",
  },
  chrome: {
    guestName: "OpenClaw Agent",
    waitForInCallMs: 30000,
    bargeInInputCommand: [
      "sox",
      "-q",
      "-t",
      "coreaudio",
      "External Microphone",
      "-r",
      "24000",
      "-c",
      "1",
      "-b",
      "16",
      "-e",
      "signed-integer",
      "-t",
      "raw",
      "-",
    ],
  },
  chromeNode: {
    node: "parallels-macos",
  },
  defaultMode: "agent",
  realtime: {
    provider: "openai",
    transcriptionProvider: "openai",
    voiceProvider: "google",
    model: "gemini-3.1-flash-live-preview",
    agentId: "jay",
    toolPolicy: "owner",
    introMessage: "Say exactly: I'm here.",
    providers: {
      google: {
        speakerVoice: "Kore",
      },
    },
  },
}
```

ElevenLabs voor zowel luisteren als spreken in agentmodus:

```json5
{
  tts: {
    provider: "elevenlabs",
    providers: {
      elevenlabs: {
        modelId: "eleven_v3",
        speakerVoiceId: "pMsXgVXv3BLzUgSXRplE",
      },
    },
  },
  plugins: {
    entries: {
      "google-meet": {
        config: {
          realtime: {
            transcriptionProvider: "elevenlabs",
            providers: {
              elevenlabs: {
                modelId: "scribe_v2_realtime",
                audioFormat: "ulaw_8000",
                sampleRate: 8000,
                commitStrategy: "vad",
              },
            },
          },
        },
      },
    },
  },
}
```

De blijvende Meet-stem is afkomstig van `tts.providers.elevenlabs.speakerVoiceId`. Agentantwoorden kunnen ook `[[tts:speakerVoiceId=... model=eleven_v3]]`-instructies per antwoord gebruiken wanneer overschrijvingen van het TTS-model zijn ingeschakeld, maar de configuratie is de deterministische standaard voor vergaderingen. Bij deelname tonen de logboeken `transcriptionProvider=elevenlabs`, en elk gesproken antwoord wordt vastgelegd als `provider=elevenlabs model=eleven_v3 speakerVoiceId=<voiceId>`.

Configuratie uitsluitend voor Twilio:

```json5
{
  defaultTransport: "twilio",
  twilio: {
    defaultDialInNumber: "+15551234567",
    defaultPin: "123456",
  },
  voiceCall: {
    gatewayUrl: "ws://127.0.0.1:18789",
  },
}
```

Met `voiceCall.enabled: true` (de standaard) en Twilio-transport plaatst Voice Call de DTMF-reeks voordat de realtime mediastream wordt geopend en gebruikt vervolgens de opgeslagen introductietekst als eerste realtime begroeting. Als `voice-call` niet is ingeschakeld, kan Google Meet het belplan nog steeds valideren en vastleggen, maar kan het Twilio-gesprek niet worden geplaatst.

Laat `voiceCall.gatewayUrl` niet ingesteld om de lokale vertrouwde Gateway-runtime te gebruiken, die de
aanroepende agent gedurende de volledige aanroep behoudt. Een geconfigureerde Gateway-URL blijft een expliciet WebSocket-doel en
kan de herkomst van plugins niet verifiëren; koppelingen met niet-standaardagents worden veilig geweigerd in plaats van stilzwijgend
een andere agent te gebruiken. Voer Google Meet en Voice Call uit in hetzelfde Gateway-proces wanneer routering
per agent vereist is.

## Tool

Agents gebruiken de tool `google_meet`:

```json
{
  "action": "join",
  "url": "https://meet.google.com/abc-defg-hij",
  "transport": "chrome-node",
  "mode": "agent"
}
```

| `action`                | Doel                                                                                              |
| ----------------------- | ------------------------------------------------------------------------------------------------- |
| `join`                  | Deelnemen via een expliciete Meet-URL                                                              |
| `create`                | Een ruimte maken (en standaard deelnemen); ondersteunt `accessType`/`entryPointAccess`             |
| `status`                | Actieve sessies weergeven of één sessie inspecteren via `sessionId`                                |
| `setup_status`          | Dezelfde controles uitvoeren als `googlemeet setup`                                                |
| `resolve_space`         | Een URL/code/`spaces/{id}` oplossen via `spaces.get`                                           |
| `preflight`             | Vereisten voor OAuth en het oplossen van vergaderingen valideren                                   |
| `latest`                | De nieuwste conferentierecord voor een vergadering zoeken                                          |
| `calendar_events`       | Een voorbeeld weergeven van Agenda-afspraken met Meet-links                                        |
| `artifacts`             | Conferentierecords en metadata van deelnemers/opnamen/transcripten/slimme notities weergeven        |
| `attendance`            | Deelnemers en deelnemerssessies weergeven                                                           |
| `export`                | De bundel met artefacten/aanwezigheid/transcript/manifest schrijven; stel `"dryRun": true` in voor alleen het manifest |
| `recover_current_tab`   | Een bestaand Meet-tabblad activeren/inspecteren zonder een nieuw tabblad te openen                  |
| `transcript`            | Het begrensde ondertitelingstranscript lezen; `sinceIndex` gaat verder vanaf de vorige `nextIndex` |
| `leave`                 | Een sessie beëindigen (Chrome klikt op Verlaten; sluit alleen zelf geopende tabbladen; Twilio verbreekt de verbinding) |
| `end_active_conference` | De actieve Google Meet-conferentie beëindigen voor een via de API beheerde ruimte                   |
| `speak`                 | De realtimeagent onmiddellijk laten spreken op basis van `sessionId` en `message`         |
| `test_speech`           | Een sessie maken/hergebruiken, een bekende zin activeren en de Chrome-status retourneren           |
| `test_listen`           | Een sessie maken/hergebruiken die alleen observeert en wachten op wijzigingen in ondertiteling/transcript |

`test_speech` dwingt altijd `mode: "agent"` of `"bidi"` af en mislukt als wordt gevraagd om in `mode: "transcribe"` te worden uitgevoerd, omdat sessies die alleen observeren geen spraak kunnen uitsturen. `speechOutputVerified` vereist zowel nieuwe realtime-uitvoerbytes als nieuwe, niet-stille audio die tijdens die uitvoer terugkomt via het microfoonopnamepad van de bridge. Oudere uitvoer of een loopbacksignaal van een hergebruikte sessie telt niet mee, en alleen een toename van sink-bytes wordt niet langer als geverifieerde spraak gerapporteerd.

Voor Chrome-transports houdt `leave` een hergebruikt tabblad van de gebruiker open nadat op de knop voor het verlaten van het Meet-gesprek is geklikt. Tabbladen die door OpenClaw zijn geopend, worden na vertrek gesloten.

Gebruik `transport: "chrome"` wanneer Chrome op de Gateway-host wordt uitgevoerd en `transport: "chrome-node"` wanneer het op een gekoppelde Node wordt uitgevoerd. In beide gevallen worden de modelproviders en `openclaw_agent_consult` op de Gateway-host uitgevoerd, zodat modelreferenties daar blijven. Logboeken in agentmodus bevatten bij het starten van de bridge de vastgestelde transcriptieprovider en het model, en na elk gesynthetiseerd antwoord de TTS-provider, het model, de stem, de uitvoerindeling en de samplefrequentie. Onbewerkte `mode: "realtime"` wordt nog steeds geaccepteerd als verouderde compatibiliteitsalias voor `mode: "agent"`, maar wordt niet meer vermeld in de `mode`-enum van de tool.

`create` met een ruimte die door een API wordt ondersteund en een expliciet toegangsbeleid:

```json
{
  "action": "create",
  "transport": "chrome-node",
  "mode": "agent",
  "accessType": "OPEN"
}
```

De actieve conferentie van een bekende ruimte beëindigen:

```json
{
  "action": "end_active_conference",
  "meeting": "https://meet.google.com/abc-defg-hij"
}
```

Validatie waarbij eerst wordt geluisterd, voordat wordt beweerd dat een vergadering bruikbaar is:

```json
{
  "action": "test_listen",
  "url": "https://meet.google.com/abc-defg-hij",
  "transport": "chrome-node",
  "timeoutMs": 30000
}
```

Op verzoek spreken:

```json
{
  "action": "speak",
  "sessionId": "meet_...",
  "message": "Zeg precies: ik ben hier en luister."
}
```

`status` bevat de Chrome-status wanneer die beschikbaar is:

| Veld                                                                  | Betekenis                                                                                                              |
| --------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `inCall`                                                              | Chrome lijkt zich in het Meet-gesprek te bevinden                                                                      |
| `micMuted`                                                            | Meet-microfoonstatus op basis van beste inspanning                                                                     |
| `manualActionRequired` / `manualActionReason` / `manualActionMessage` | Het browserprofiel vereist handmatig aanmelden, toelating door de Meet-host, machtigingen of herstel van browserbesturing voordat spraak kan werken |
| `speechReady` / `speechBlockedReason` / `speechBlockedMessage`        | Of beheerde Chrome-spraak nu is toegestaan; `speechReady: false` betekent dat OpenClaw de introductie-/testzin niet heeft verzonden |
| `providerConnected` / `realtimeReady`                                 | Status van de realtime-spraakbridge                                                                                    |
| `lastInputAt` / `lastOutputAt`                                        | Laatste audio die van/naar de bridge is ontvangen/verzonden                                                            |
| `audioOutputRouted` / `audioOutputDeviceLabel`                        | Of de media-uitvoer van het Meet-tabblad actief naar het BlackHole-apparaat van de bridge is gerouteerd                |
| `lastOutputLoopbackAt` / `outputLoopbackSignalBytes`                  | Nieuwe uitvoer waarvan de golfvormvingerafdruk is gecorreleerd op het BlackHole-microfoonopnamepad                    |
| `lastOutputLoopbackCorrelation`                                       | Correlatiescore die het opgenomen signaal koppelt aan de huidige generatie van assistentuitvoer                        |
| `outputGeneration` / `verifiedOutputGeneration`                       | Monotone id's; gelijkheid betekent dat de huidige uitvoer, en niet een oudere uiting, de loopback-controle heeft doorstaan |
| `lastOutputLoopbackRms` / `lastOutputLoopbackPeak`                    | Audio-energiediagnostiek voor het nieuwste geverifieerde loopback-opnamefragment                                       |
| `lastSuppressedInputAt` / `suppressedInputBytes`                      | Loopback-invoer genegeerd terwijl het afspelen van assistentaudio actief is                                            |

## Agent- en bidi-modi

| Modus   | Wie het antwoord bepaalt      | Uitvoerpad voor spraak                  | Gebruiken wanneer                                      |
| ------- | ----------------------------- | -------------------------------------- | ----------------------------------------------------- |
| `agent` | De geconfigureerde OpenClaw-agent | Normale OpenClaw TTS-runtime            | Je gedrag wilt waarbij „mijn agent in de vergadering zit” |
| `bidi`  | Het realtime-spraakmodel      | Audiorespons van de realtime-spraakprovider | Je een gesprekslus met de laagste latentie wilt    |

`agent`-modus: de realtime-transcriptieprovider hoort de vergaderaudio, definitieve transcripties van deelnemers worden via de geconfigureerde OpenClaw-agent gerouteerd en het antwoord wordt uitgesproken via de normale OpenClaw TTS. Nabijgelegen definitieve transcriptfragmenten worden vóór de raadpleging samengevoegd, zodat één gesproken beurt niet meerdere verouderde gedeeltelijke antwoorden oplevert; realtime-invoer wordt onderdrukt zolang assistentaudio in de wachtrij nog wordt afgespeeld, en recente transcriptie-echo's die op assistentuitvoer lijken, worden vóór de raadpleging genegeerd, zodat BlackHole-loopback de agent niet op zijn eigen spraak laat antwoorden.

`bidi`-modus: het realtime-spraakmodel antwoordt rechtstreeks en kan `openclaw_agent_consult` aanroepen voor diepere redenering, actuele informatie of normale OpenClaw-tools. De raadpleegtool voert achter de schermen de normale OpenClaw-agent uit met recente context uit het vergaderingstranscript en retourneert een beknopt gesproken antwoord; in de `agent`-modus stuurt OpenClaw dat antwoord rechtstreeks naar TTS, in de `bidi`-modus kan het realtime-spraakmodel het uitspreken. Hiervoor wordt hetzelfde gedeelde raadpleegmechanisme gebruikt als voor Voice Call.

Standaard worden raadplegingen uitgevoerd voor de agent `main`; stel `realtime.agentId` in om een Meet-kanaal naar een specifieke agentwerkruimte, standaardmodellen, toolbeleid, geheugen en sessiegeschiedenis te verwijzen. Raadplegingen in agentmodus gebruiken per vergadering een `agent:<id>:subagent:google-meet:<session>`-sessiesleutel, zodat vervolgvragen de vergadercontext behouden en tegelijkertijd het normale agentbeleid overnemen. Wanneer een agent in agentmodus `google_meet` aanroept, vertakt de consultantsessie het huidige transcript van de aanroeper voordat op de spraak van deelnemers wordt geantwoord; de Meet-sessie blijft afzonderlijk, zodat vervolgvragen in de vergadering het transcript van de aanroeper niet rechtstreeks wijzigen.

`realtime.toolPolicy` bepaalt de raadpleeguitvoering:

| Beleid           | Gedrag                                                                                                                            |
| ---------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `safe-read-only` | De raadpleegtool beschikbaar stellen; de normale agent beperken tot `read`, `web_search`, `web_fetch`, `x_search`, `memory_search`, `memory_get` |
| `owner`          | De raadpleegtool beschikbaar stellen; de normale agent zijn gebruikelijke toolbeleid laten gebruiken                            |
| `none`           | De raadpleegtool niet beschikbaar stellen aan het realtime-spraakmodel                                                           |

De sleutel van de raadpleegsessie heeft per Meet-sessie een eigen bereik, zodat opeenvolgende raadpleegaanroepen eerdere raadpleegcontext binnen dezelfde vergadering hergebruiken.

Dwing een gesproken gereedheidscontrole af nadat Chrome volledig heeft deelgenomen:

```bash
openclaw googlemeet speak meet_... "Zeg precies: ik ben hier en luister."
```

Volledige rooktest voor deelnemen en spreken:

```bash
openclaw googlemeet test-speech https://meet.google.com/abc-defg-hij \
  --transport chrome-node \
  --message "Zeg precies: ik ben hier en luister."
```

## Checklist voor livetests

Voordat je een vergadering overdraagt aan een onbeheerde agent:

```bash
openclaw googlemeet setup
openclaw nodes status
openclaw googlemeet test-speech https://meet.google.com/abc-defg-hij \
  --transport chrome-node \
  --message "Zeg precies: Google Meet-spraaktest voltooid."
```

Verwachte Chrome-node-status:

- `googlemeet setup` is volledig groen en bevat `chrome-node-connected` wanneer Chrome-node het standaardtransport is of een node is vastgezet.
- `nodes status` toont dat de geselecteerde node verbonden is en zowel `googlemeet.chrome` als `browser.proxy` aanbiedt.
- Het tabblad Meet neemt deel en `test-speech` retourneert de Chrome-status met `inCall: true`.

Voor een externe Chrome-host, zoals een Parallels macOS-VM, is dit de kortste veilige controle na het bijwerken van de Gateway of de VM:

```bash
openclaw googlemeet setup
openclaw nodes status --connected
openclaw nodes invoke \
  --node parallels-macos \
  --command googlemeet.chrome \
  --params '{"action":"setup"}'
```

Daarmee wordt aangetoond dat de Gateway-plugin is geladen, de VM-node met het huidige token is verbonden en de Meet-audiobrug beschikbaar is voordat een agent een echt vergadertabblad opent.

Gebruik voor een Twilio-rooktest een vergadering die telefonische inbelgegevens beschikbaar stelt:

```bash
openclaw googlemeet setup
openclaw googlemeet join https://meet.google.com/abc-defg-hij \
  --transport twilio \
  --dial-in-number +15551234567 \
  --pin 123456
```

Verwachte Twilio-status:

- `googlemeet setup` bevat groene controles voor `twilio-voice-call-plugin`, `twilio-voice-call-credentials` en `twilio-voice-call-webhook`.
- `voicecall` is na het herladen van de Gateway beschikbaar in de CLI.
- De geretourneerde sessie heeft `transport: "twilio"` en een `twilio.voiceCallId`.
- `openclaw logs --follow` toont dat DTMF-TwiML vóór realtime-TwiML is aangeboden, gevolgd door een realtimebrug met de eerste begroeting in de wachtrij.
- `googlemeet leave <sessionId>` beëindigt de gedelegeerde spraakoproep.

## Problemen oplossen

### Agent kan de Google Meet-tool niet zien

Controleer of de plugin is ingeschakeld en herlaad de Gateway; de actieve agent ziet alleen plugintools die door het huidige Gateway-proces zijn geregistreerd:

```bash
openclaw plugins list | grep google-meet
openclaw googlemeet setup
```

Op Gateway-hosts zonder macOS blijft `google_meet` zichtbaar, maar lokale Chrome-acties voor terugspreken worden geblokkeerd voordat ze de audiobrug bereiken. Gebruik `mode: "transcribe"`, telefonisch inbellen via Twilio of een macOS-host met `chrome-node` in plaats van het standaardpad voor de lokale Chrome-agent.

### Geen verbonden node met Google Meet-mogelijkheden

Op de nodehost:

```bash
openclaw plugins install npm:@openclaw/google-meet
openclaw plugins enable browser
OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1 \
  openclaw node run --host <gateway-lan-ip> --port 18789 --display-name parallels-macos
```

Op de Gateway-host:

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw nodes status
```

De node moet verbonden zijn en `googlemeet.chrome` plus `browser.proxy` vermelden; de Gateway-configuratie moet beide toestaan:

```json5
{
  gateway: {
    nodes: {
      commands: { allow: ["browser.proxy", "googlemeet.chrome"] },
    },
  },
}
```

Als `googlemeet setup` niet slaagt voor `chrome-node-connected`, of het Gateway-logboek `gateway token mismatch` meldt, installeer de node dan opnieuw of start deze opnieuw met het huidige Gateway-token:

```bash
OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1 \
  openclaw node install \
  --host <gateway-lan-ip> \
  --port 18789 \
  --display-name parallels-macos \
  --force
```

Herlaad vervolgens de nodeservice en voer dit opnieuw uit:

```bash
openclaw googlemeet setup
openclaw nodes status --connected
```

### Browser wordt geopend, maar agent kan niet deelnemen

Voer `googlemeet test-listen` uit voor deelnames waarbij alleen wordt geobserveerd, of `googlemeet test-speech` voor realtime-deelnames, en controleer vervolgens de geretourneerde Chrome-status. Als een van beide `manualActionRequired: true` meldt, toon dan `manualActionMessage` aan de operator en probeer het niet opnieuw totdat de browseractie is voltooid.

Veelvoorkomende handmatige acties: meld je aan bij het Chrome-profiel; laat de gast toe vanuit het Meet-hostaccount; verleen Chrome toestemming voor de microfoon/camera wanneer de systeemeigen prompt verschijnt; sluit of herstel een vastgelopen toestemmingsdialoogvenster van Meet.

Meld niet dat je „niet aangemeld” bent alleen omdat Meet vraagt „Do you want people to hear you in the meeting?”; dat is het tussenscherm van Meet voor de audiokeuze. OpenClaw klikt via browserautomatisering op **Use microphone** wanneer dat beschikbaar is en blijft wachten op de werkelijke vergaderstatus; voor een browserfallback die alleen een vergadering aanmaakt, kan het in plaats daarvan op **Continue without microphone** klikken, omdat voor het genereren van de URL het realtime-audiopad niet nodig is.

### Vergadering aanmaken mislukt

`googlemeet create` gebruikt de Meet-API `spaces.create` wanneer OAuth is geconfigureerd, en anders de browser van de vastgezette Chrome-node. Controleer het volgende:

- **Aanmaken via API**: `oauth.clientId` en `oauth.refreshToken` (of overeenkomende `OPENCLAW_GOOGLE_MEET_*`-omgevingsvariabelen) zijn aanwezig en het vernieuwingstoken is aangemaakt nadat ondersteuning voor aanmaken werd toegevoegd; oudere tokens beschikken mogelijk niet over `meetings.space.created`, dus voer `openclaw googlemeet auth login --json` opnieuw uit.
- **Browserfallback**: `defaultTransport: "chrome-node"` en `chromeNode.node` verwijzen naar een verbonden node met `browser.proxy` en `googlemeet.chrome`; het OpenClaw Chrome-profiel op die node is aangemeld en kan `https://meet.google.com/new` openen.
- **Nieuwe pogingen voor browserfallback**: hergebruik een bestaand tabblad met `.../new` of een Google-accountprompt voordat je een nieuw tabblad opent; probeer de toolaanroep opnieuw in plaats van handmatig nog een tabblad te openen.
- **Handmatige actie**: als de tool `manualActionRequired: true` retourneert, gebruik dan `browser.nodeId`, `browser.targetId`, `browserUrl` en `manualActionMessage` om de operator te begeleiden; blijf niet in een lus opnieuw proberen.
- **Tussenscherm voor audiokeuze**: als Meet „Do you want people to hear you in the meeting?” toont, laat het tabblad dan open. OpenClaw hoort op **Use microphone** of (alleen voor aanmaken) **Continue without microphone** te klikken en te blijven wachten op de gegenereerde URL; als dat niet lukt, moet de foutmelding `meet-audio-choice-required` vermelden en niet `google-login-required`.

### Agent neemt deel, maar praat niet

```bash
openclaw googlemeet setup
openclaw googlemeet doctor
```

Gebruik `mode: "agent"` voor het pad STT -> OpenClaw-agent -> TTS en `mode: "bidi"` voor de directe realtime-spraakfallback. `mode: "transcribe"` start opzettelijk geen brug voor terugspreken. Voer voor foutopsporing waarbij alleen wordt geobserveerd `openclaw googlemeet status --json <session-id>` uit nadat deelnemers hebben gesproken en controleer `captioning`, `transcriptLines` en `lastCaptionText`. Als `inCall` waar is, maar `transcriptLines` `0` blijft, zijn Meet-ondertitels mogelijk uitgeschakeld, heeft niemand gesproken sinds de observator is geïnstalleerd, is de Meet-interface gewijzigd of zijn live-ondertitels niet beschikbaar voor de taal of het account van de vergadering.

`googlemeet test-speech` controleert altijd het realtimepad en meldt of voor die aanroep uitvoerbytes van de brug zijn waargenomen. Als `speechOutputVerified` onwaar is en `speechOutputTimedOut` waar is, heeft de realtimeprovider de uiting mogelijk geaccepteerd, maar heeft OpenClaw geen nieuwe uitvoerbytes de Chrome-audiobrug zien bereiken.

Controleer ook het volgende: er is een sleutel voor een realtimeprovider (`OPENAI_API_KEY` of `GEMINI_API_KEY`) beschikbaar op de Gateway-host; `BlackHole 2ch` is zichtbaar op de Chrome-host; `sox` bestaat daar; de Meet-microfoon en -luidspreker worden via het virtuele audiopad geleid (`doctor` hoort `meet output routed: yes` te tonen voor realtime-deelnames met lokale Chrome).

`googlemeet doctor [session-id]` toont de sessie, node, status tijdens de oproep, reden voor handmatige actie, verbinding met de realtimeprovider, `realtimeReady`, audio-invoer-/uitvoeractiviteit, tijdstempels van de laatste audio, bytetellers en browser-URL. Gebruik `googlemeet status [session-id] --json` voor onbewerkte JSON en `googlemeet doctor --oauth` (voeg `--meeting` of `--create-space` toe) om OAuth-vernieuwing te verifiëren zonder tokens prijs te geven.

Als een agent een time-out heeft gekregen en er al een Meet-tabblad is geopend, controleer dit dan zonder nog een tabblad te openen:

```bash
openclaw googlemeet recover-tab
openclaw googlemeet recover-tab https://meet.google.com/abc-defg-hij
```

De equivalente toolactie is `recover_current_tab`: deze activeert en controleert een bestaand Meet-tabblad voor het geselecteerde transport (lokale browserbesturing voor `chrome`, de geconfigureerde node voor `chrome-node`) zonder een nieuw tabblad of een nieuwe sessie te openen, en meldt de huidige blokkade (aanmelding, toelating, toestemmingen, status van de audiokeuze). De CLI-opdracht communiceert met de geconfigureerde Gateway, die actief moet zijn; voor `chrome-node` moet de node ook verbonden zijn.

### Twilio-installatiecontroles mislukken

`twilio-voice-call-plugin` mislukt wanneer `voice-call` niet is toegestaan of niet is ingeschakeld: voeg het toe aan `plugins.allow`, schakel `plugins.entries.voice-call` in en herlaad de Gateway.

`twilio-voice-call-credentials` mislukt wanneer bij de Twilio-backend de account-SID, het authenticatietoken of het nummer van de beller ontbreekt:

```bash
export TWILIO_ACCOUNT_SID=AC...
export TWILIO_AUTH_TOKEN=...
export TWILIO_FROM_NUMBER=+15550001234
```

`twilio-voice-call-webhook` mislukt wanneer `voice-call` niet via een openbare Webhook bereikbaar is, of `publicUrl` naar een loopbackadres of privé-netwerkruimte verwijst. Gebruik `localhost`, `127.0.0.1`, `0.0.0.0`, `10.x`, `172.16.x`-`172.31.x`, `192.168.x`, `169.254.x`, `fc00::/7` of `fd00::/8` niet als `publicUrl`; callbacks van de provider kunnen die niet bereiken. Stel `plugins.entries.voice-call.config.publicUrl` in op een openbare URL of configureer bereikbaarheid via een tunnel/Tailscale:

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        enabled: true,
        config: {
          provider: "twilio",
          fromNumber: "+15550001234",
          publicUrl: "https://voice.example.com/voice/webhook",
        },
      },
    },
  },
}
```

Gebruik voor lokale ontwikkeling bereikbaarheid via een tunnel of Tailscale in plaats van een privé-host-URL:

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          tunnel: { provider: "ngrok" },
          // of
          tailscale: { mode: "funnel", path: "/voice/webhook" },
        },
      },
    },
  },
}
```

Start de Gateway opnieuw of herlaad deze en voer vervolgens het volgende uit:

```bash
openclaw googlemeet setup --transport twilio
openclaw voicecall setup
openclaw voicecall smoke
```

`voicecall smoke` controleert standaard alleen de gereedheid. Voer een droogtest uit voor een specifiek nummer:

```bash
openclaw voicecall smoke --to "+15555550123"
```

Voeg `--yes` alleen toe om opzettelijk een echte uitgaande oproep te plaatsen:

```bash
openclaw voicecall smoke --to "+15555550123" --yes
```

### Twilio-oproep start, maar neemt nooit deel aan de vergadering

Controleer of de Meet-gebeurtenis telefonische inbelgegevens beschikbaar stelt en geef het exacte inbelnummer plus de pincode of een aangepaste DTMF-reeks door:

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij \
  --transport twilio \
  --dial-in-number +15551234567 \
  --dtmf-sequence ww123456#
```

Gebruik voorafgaande `w` of komma's in `--dtmf-sequence` voor een pauze vóór de pincode.

Als de oproep is aangemaakt, maar de lijst met Meet-deelnemers de inbellende deelnemer nooit toont:

- `openclaw googlemeet doctor <session-id>`: controleer de gedelegeerde Twilio-oproep-ID, of DTMF in de wachtrij is geplaatst en of om de inleidende begroeting is gevraagd.
- `openclaw voicecall status --call-id <id>`: controleer of de oproep nog actief is.
- `openclaw voicecall tail`: controleer of Twilio-webhooks bij de Gateway aankomen.
- `openclaw logs --follow`: zoek naar de Twilio Meet-volgorde: Google Meet delegeert het deelnemen, Voice Call slaat TwiML voor DTMF vóór de verbinding op en biedt dit aan, Voice Call biedt realtime-TwiML voor de Twilio-oproep aan en vervolgens vraagt Google Meet met `voicecall.speak` om de inleidende spraak.
- Voer `openclaw googlemeet setup --transport twilio` opnieuw uit; een groene installatiecontrole is vereist, maar bewijst niet dat de pincodevolgorde van de vergadering correct is.
- Controleer of het inbelnummer bij dezelfde Meet-uitnodiging en regio hoort als de pincode.
- Verhoog `voiceCall.dtmfDelayMs` vanaf de standaardwaarde van 12 seconden als Meet langzaam opneemt of het oproeptranscript nog steeds de pincodeprompt toont nadat DTMF vóór de verbinding is verzonden.
- Als de deelnemer deelneemt, maar je de begroeting niet hoort, controleer dan `openclaw logs --follow` op het `voicecall.speak`-verzoek na DTMF en op TTS-weergave via de mediastream of de Twilio-fallback `<Say>`. Als het transcript nog steeds „enter the meeting PIN” toont, heeft de telefonische verbinding nog niet deelgenomen aan de Meet-ruimte, zodat deelnemers geen spraak zullen horen.

Als webhooks niet aankomen, debug dan eerst de Voice Call-plugin: de provider moet `plugins.entries.voice-call.config.publicUrl` of de geconfigureerde tunnel kunnen bereiken. Zie [Problemen met spraakoproepen oplossen](/nl/plugins/voice-call#troubleshooting).

## Opmerkingen

De officiële media-API van Google Meet is gericht op ontvangst, dus om in een gesprek te spreken is nog steeds een deelnemerspad nodig. Deze Plugin houdt die grens zichtbaar: Chrome verzorgt deelname via de browser en lokale audioroutering; Twilio verzorgt telefonische inbeldeelname.

De terugspreekmodi van Chrome vereisen `BlackHole 2ch` plus een van de volgende opties:

- `chrome.audioInputCommand` plus `chrome.audioOutputCommand`: OpenClaw beheert de bridge en stuurt audio in `chrome.audioFormat` door tussen deze opdrachten en de geselecteerde provider. De modus `agent` gebruikt realtime transcriptie plus reguliere TTS; de modus `bidi` gebruikt de realtime spraakprovider. Het standaardpad is 24 kHz PCM16 met `chrome.audioBufferBytes: 4096`; 8 kHz G.711 mu-law blijft beschikbaar voor verouderde opdrachtparen.
- `chrome.audioBridgeCommand`: een externe bridgeopdracht beheert het volledige lokale audiopad en moet worden afgesloten nadat de daemon is gestart of gevalideerd. Alleen geldig voor `bidi`, omdat de modus `agent` directe toegang tot het opdrachtenpaar nodig heeft voor TTS.

Met de Chrome-bridge met opdrachtenpaar kan `chrome.bargeInInputCommand` naar een afzonderlijke lokale microfoon luisteren en het afspelen door de assistent wissen wanneer een persoon begint te praten, zodat menselijke spraak voorrang houdt op de uitvoer van de assistent, zelfs wanneer de gedeelde BlackHole-loopbackinvoer tijdens het afspelen door de assistent tijdelijk wordt onderdrukt. Net als `chrome.audioInputCommand`/`chrome.audioOutputCommand` is dit een door de beheerder geconfigureerde lokale opdracht: gebruik een expliciet vertrouwd opdrachtpad of een expliciete argumentenlijst, en nooit een script vanaf een niet-vertrouwde locatie.

Routeer voor heldere duplexaudio de Meet-uitvoer en de Meet-microfoon via afzonderlijke virtuele apparaten of een virtuele apparaatgraaf in Loopback-stijl; één gedeeld BlackHole-apparaat kan andere deelnemers terug laten echoën in het gesprek.

`googlemeet speak` activeert de actieve terugspreekaudiobridge voor een Chrome-sessie; `googlemeet leave` stopt deze (en beëindigt bij Twilio-sessies die via Voice Call zijn gedelegeerd ook het onderliggende gesprek). Gebruik `googlemeet end-active-conference` om ook de actieve Google Meet-vergadering te sluiten voor een via de API beheerde ruimte.

## Gerelateerd

- [Overzicht van vergaderplugins](/nl/plugins/meeting-plugins)
- [Voice Call-plugin](/nl/plugins/voice-call)
- [Spreekmodus](/nl/nodes/talk)
- [Plugins bouwen](/nl/plugins/building-plugins)
