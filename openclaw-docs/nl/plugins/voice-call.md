---
read_when:
    - Je wilt vanuit OpenClaw een uitgaand spraakgesprek starten
    - Je configureert of ontwikkelt de spraakoproepplugin
    - Je hebt realtime spraak of streamingtranscriptie voor telefonie nodig
sidebarTitle: Voice call
summary: Plaats uitgaande en accepteer inkomende spraakoproepen via Twilio, Telnyx of Plivo, met optionele realtime spraak en streamingtranscriptie
title: Spraakoproepplugin
x-i18n:
    generated_at: "2026-07-27T05:43:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 79f09f7b5cb99aace0960e283723d4f4408afa5f5dacd71f3c527fa62859f56f
    source_path: plugins/voice-call.md
    workflow: 16
---

Spraakoproepen voor OpenClaw via een plugin: uitgaande meldingen, gesprekken
met meerdere beurten, full-duplex realtime spraak, streaming transcriptie en
inkomende oproepen met beleid op basis van toelatingslijsten.

**Providers:** `mock` (ontwikkeling, geen netwerk), `plivo` (Voice API + XML-overdracht +
GetInput-spraak), `telnyx` (Call Control v2), `twilio` (Programmable Voice +
Media Streams).

<Note>
De Voice Call-plugin draait **binnen het Gateway-proces**. Als je een externe
Gateway gebruikt, installeer en configureer je de plugin op de machine waarop
de Gateway draait en start je de Gateway vervolgens opnieuw om de plugin te laden.
</Note>

## Snel aan de slag

<Steps>
  <Step title="Installeer de plugin">
    <Tabs>
      <Tab title="Van npm">
        ```bash
        openclaw plugins install @openclaw/voice-call
        ```
      </Tab>
      <Tab title="Vanuit een lokale map (ontwikkeling)">
        ```bash
        PLUGIN_SRC=./path/to/local/voice-call-plugin
        openclaw plugins install "$PLUGIN_SRC"
        cd "$PLUGIN_SRC" && pnpm install
        ```
      </Tab>
    </Tabs>

    Gebruik het kale pakket om de huidige releasetag te volgen. Zet alleen een
    exacte versie vast wanneer je een reproduceerbare installatie nodig hebt. Start
    daarna de Gateway opnieuw zodat de plugin wordt geladen.

  </Step>
  <Step title="Configureer provider en webhook">
    Stel de configuratie in onder `plugins.entries.voice-call.config` (zie
    [Configuratie](#configuration) hieronder). Minimaal vereist: `provider`,
    providerreferenties, `fromNumber` en een openbaar bereikbare webhook-URL.
  </Step>
  <Step title="Controleer de configuratie">
    ```bash
    openclaw voicecall setup
    openclaw voicecall setup --json
    ```

    Controleert of de plugin is ingeschakeld, de providerreferenties, de
    bereikbaarheid van de webhook en of slechts één audiomodus
    (`streaming` of `realtime`) actief is.

  </Step>
  <Step title="Voer een rooktest uit">
    ```bash
    openclaw voicecall smoke
    openclaw voicecall smoke --to "+15555550123"
    ```

    Beide zijn standaard proefruns. Voeg `--yes` toe om een korte
    uitgaande meldingsoproep te plaatsen:

    ```bash
    openclaw voicecall smoke --to "+15555550123" --yes
    ```

  </Step>
</Steps>

<Warning>
Voor Twilio, Telnyx en Plivo moet de configuratie resulteren in een **openbare webhook-URL**.
Als `publicUrl`, de tunnel-URL, de Tailscale-URL of de serve-terugvaloptie
naar loopback- of particuliere netwerkruimte verwijst, mislukt de configuratie
in plaats van een provider te starten die geen carrierwebhooks kan ontvangen.
</Warning>

## Configuratie

Als `enabled: true` maar de geselecteerde provider geen referenties heeft,
registreert het opstarten van de Gateway een waarschuwing dat de configuratie
onvolledig is, met de ontbrekende sleutels, en wordt de runtime niet gestart.
Opdrachten, RPC-aanroepen en agenttools retourneren bij gebruik nog steeds de
exact ontbrekende configuratie.

<Note>
Voice Call-referenties ondersteunen SecretRefs. `plugins.entries.voice-call.config.twilio.authToken`, `plugins.entries.voice-call.config.realtime.providers.*.apiKey`, `plugins.entries.voice-call.config.streaming.providers.*.apiKey` en `plugins.entries.voice-call.config.tts.providers.*.apiKey` worden via het standaard SecretRef-oppervlak omgezet; zie [SecretRef-oppervlak voor referenties](/nl/reference/secretref-credential-surface).
</Note>

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        enabled: true,
        config: {
          provider: "twilio", // of "telnyx" | "plivo" | "mock"
          fromNumber: "+15550001234", // of TWILIO_FROM_NUMBER voor Twilio
          toNumber: "+15550005678",
          sessionScope: "per-phone", // per-phone | per-call
          numbers: {
            "+15550009999": {
              inboundGreeting: "Silver Fox Cards, hoe kan ik helpen?",
              responseSystemPrompt: "Je bent een beknopte specialist in honkbalkaarten.",
              tts: {
                providers: {
                  openai: { speakerVoice: "alloy" },
                },
              },
            },
          },

          twilio: {
            accountSid: "ACxxxxxxxx",
            authToken: "...",
            // region: "ie1", // optioneel: us1 | ie1 | au1; standaard us1
          },
          telnyx: {
            apiKey: "...",
            connectionId: "...",
            // Openbare sleutel voor Telnyx-webhooks uit het Mission Control Portal
            // (Base64; kan ook via TELNYX_PUBLIC_KEY worden ingesteld).
            publicKey: "...",
          },
          plivo: {
            authId: "MAxxxxxxxxxxxxxxxxxxxx",
            authToken: "...",
          },

          // Webhookserver
          serve: {
            port: 3334,
            path: "/voice/webhook",
          },

          // Webhookbeveiliging (aanbevolen voor tunnels/proxy's)
          webhookSecurity: {
            allowedHosts: ["voice.example.com"],
            trustedProxyIPs: ["100.64.0.1"],
          },

          // Openbare beschikbaarstelling (kies er één)
          // publicUrl: "https://example.ngrok.app/voice/webhook",
          // tunnel: { provider: "ngrok" },
          // tailscale: { mode: "funnel", path: "/voice/webhook" },

          outbound: {
            defaultMode: "notify", // notify | conversation
          },

          streaming: { enabled: true /* alleen Twilio; zie Streaming transcriptie */ },
          realtime: { enabled: false /* zie Realtime spraakgesprekken */ },
        },
      },
    },
  },
}
```

### Configuratiereferentie

Sleutels op het hoogste niveau onder `plugins.entries.voice-call.config` die hierboven niet worden weergegeven:

| Sleutel                          | Standaard    | Opmerkingen                                                                                        |
| ------------------------------- | ------------ | -------------------------------------------------------------------------------------------------- |
| `enabled`                       | `false`      | Hoofdschakelaar voor aan/uit.                                                                      |
| `inboundPolicy`                 | `"disabled"` | `disabled` \| `allowlist` \| `pairing` \| `open`. Zie [Inkomende oproepen](#inbound-calls).       |
| `allowFrom`                     | `[]`         | E.164-toelatingslijst voor `inboundPolicy: "allowlist"`.                                           |
| `maxDurationSeconds`            | `300`        | Harde maximale duur per oproep, ongeacht of de oproep is beantwoord.                                |
| `staleCallReaperSeconds`        | `120`        | Zie [Opruimer voor verouderde oproepen](#stale-call-reaper). `0` schakelt deze uit.  |
| `silenceTimeoutMs`              | `800`        | Detectie van stilte aan het einde van spraak voor de klassieke (niet-realtime) flow.                |
| `transcriptTimeoutMs`           | `180000`     | Maximale wachttijd op een transcript van de beller voordat een beurt wordt opgegeven.               |
| `ringTimeoutMs`                 | `30000`      | Time-out voor overgaan bij uitgaande oproepen.                                                      |
| `maxConcurrentCalls`            | `1`          | Uitgaande oproepen boven deze limiet worden geweigerd.                                              |
| `outbound.notifyHangupDelaySec` | `3`          | Aantal seconden na TTS voordat in meldingsmodus automatisch wordt opgehangen.                       |
| `skipSignatureVerification`     | `false`      | Alleen voor lokaal testen; nooit inschakelen in productie.                                         |
| `store`                         | niet ingesteld | Overschrijft het standaardpad `$OPENCLAW_STATE_DIR/voice-calls` (normaal `~/.openclaw/voice-calls`). |
| `agentId`                       | `"main"`     | Agent die wordt gebruikt voor het genereren van antwoorden en de opslag van sessies.                |
| `responseModel`                 | niet ingesteld | Overschrijft het standaardmodel voor klassieke (niet-realtime) antwoorden.                          |
| `responseSystemPrompt`          | gegenereerd  | Aangepaste systeemprompt voor klassieke antwoorden.                                                 |
| `responseTimeoutMs`             | `30000`      | Time-out voor het genereren van klassieke antwoorden (ms).                                         |

Twilio gebruikt standaard het Amerikaanse US1 REST-eindpunt. Om oproepen in
een ondersteunde niet-Amerikaanse regio te verwerken, stel je `twilio.region`
in op `ie1` of `au1` en gebruik je referenties uit die
regio. Zie
[Twilio's handleiding voor de niet-Amerikaanse REST API](https://www.twilio.com/docs/global-infrastructure/using-the-twilio-rest-api-in-a-non-us-region).

<AccordionGroup>
  <Accordion title="Opmerkingen over beschikbaarstelling en beveiliging van providers">
    - Twilio, Telnyx en Plivo vereisen allemaal een **openbaar bereikbare** webhook-URL.
    - `mock` is een lokale ontwikkelprovider (geen netwerkaanroepen).
    - Telnyx vereist `telnyx.publicKey` (of `TELNYX_PUBLIC_KEY`), tenzij `skipSignatureVerification` waar is.
    - `skipSignatureVerification` is alleen bedoeld voor lokaal testen.
    - Stel bij de gratis laag van ngrok `publicUrl` in op de exacte ngrok-URL; handtekeningverificatie wordt altijd afgedwongen.
    - `tunnel.allowNgrokFreeTierLoopbackBypass: true` staat Twilio-webhooks met ongeldige handtekeningen **alleen** toe wanneer `tunnel.provider="ngrok"` en `serve.bind` loopback is (lokale ngrok-agent). Alleen voor lokale ontwikkeling.
    - URL's van de gratis ngrok-laag kunnen wijzigen of interstitialgedrag toevoegen; als `publicUrl` afwijkt, mislukken Twilio-handtekeningen. Productie: geef de voorkeur aan een stabiel domein of een Tailscale-funnel.

  </Accordion>
  <Accordion title="Limieten voor streamingverbindingen">
    - `streaming.preStartTimeoutMs` (standaard `5000`) sluit sockets die nooit een geldig `start`-frame verzenden.
    - `streaming.maxPendingConnections` (standaard `32`) beperkt het totale aantal niet-geverifieerde sockets vóór de start.
    - `streaming.maxPendingConnectionsPerIp` (standaard `4`) beperkt niet-geverifieerde sockets vóór de start per bron-IP.
    - `streaming.maxConnections` (standaard `128`) beperkt alle open mediastreamsockets (wachtend + actief).

  </Accordion>
  <Accordion title="Migraties van verouderde configuratie">
    Bij het verwerken van de configuratie worden deze verouderde sleutels
    automatisch genormaliseerd en wordt een waarschuwing met het vervangende
    pad geregistreerd; de compatibiliteitslaag wordt in een toekomstige
    release verwijderd (`2026.6.0`), dus voer `openclaw doctor --fix` uit om
    vastgelegde configuratie naar de canonieke vorm te herschrijven:

    - `provider: "log"` → `provider: "mock"`
    - `twilio.from` → `fromNumber`
    - `streaming.sttProvider` → `streaming.provider`
    - `streaming.openaiApiKey` → `streaming.providers.openai.apiKey`
    - `streaming.sttModel` → `streaming.providers.openai.model`
    - `streaming.silenceDurationMs` → `streaming.providers.openai.silenceDurationMs`
    - `streaming.vadThreshold` → `streaming.providers.openai.vadThreshold`
    - `realtime.agentContext.includeSystemPrompt` is verwijderd (realtimecontext gebruikt nu de gegenereerde agentprompt)

  </Accordion>
</AccordionGroup>

## Sessiebereik

Voice Call gebruikt standaard `sessionScope: "per-phone"`, zodat herhaalde oproepen
van dezelfde beller het gespreksgeheugen behouden. Stel `sessionScope: "per-call"` in
wanneer elke carrieroproep met nieuwe context moet beginnen, bijvoorbeeld voor
receptie-, boekings-, IVR- of Google Meet-bridgeflows waarbij hetzelfde
telefoonnummer verschillende vergaderingen kan vertegenwoordigen.

Voice Call slaat gegenereerde sessiesleutels op onder de geconfigureerde
agentnaamruimte (`agent:<agentId>:voice:*`). Expliciete onbewerkte integratiesleutels
worden naar dezelfde naamruimte omgezet: een canonieke `agent:<configuredAgentId>:*`-sleutel
behoudt die eigenaar en respecteert de kernaliasing voor
`session.mainKey`/globaal bereik; vreemde of onjuist gevormde
`agent:*`-invoer krijgt als opake sleutel een bereik onder de
geconfigureerde agent; `global` en `unknown` blijven globale
sentinels.

## Realtime spraakgesprekken

`realtime` selecteert een full-duplex realtime spraakprovider voor live
oproepaudio. Dit staat los van `streaming`, dat audio alleen doorstuurt
naar providers voor realtime transcriptie.

<Warning>
`realtime.enabled` kan niet worden gecombineerd met `streaming.enabled`. Kies
één audiomodus per oproep.
</Warning>

Huidig runtimegedrag:

- `realtime.enabled` wordt ondersteund voor Twilio en Telnyx.
- `realtime.provider` is optioneel. Als dit niet is ingesteld, gebruikt Voice Call de eerste geregistreerde realtime-spraakprovider.
- Gebundelde realtime-spraakproviders: Google Gemini Live (`google`) en OpenAI (`openai`), geregistreerd door hun providerplugins.
- Door de provider beheerde onbewerkte configuratie staat onder `realtime.providers.<providerId>`.
- Voice Call stelt standaard de gedeelde realtime-tool `openclaw_agent_consult` beschikbaar. Het realtime-model kan deze aanroepen wanneer de beller om diepgaandere redenering, actuele informatie of normale OpenClaw-tools vraagt.
- `realtime.consultPolicy` voegt optioneel richtlijnen toe voor wanneer het realtime-model `openclaw_agent_consult` moet aanroepen.
- `realtime.agentContext.enabled` is standaard uitgeschakeld. Wanneer dit is ingeschakeld, voegt Voice Call tijdens het instellen van de sessie een begrensde agentidentiteit en een geselecteerde capsule met werkruimtebestanden toe aan de instructies voor de realtime-provider.
- `realtime.fastContext.enabled` is standaard uitgeschakeld. Wanneer dit is ingeschakeld, doorzoekt Voice Call eerst de geïndexeerde geheugen-/sessiecontext voor de consultatievraag en retourneert die fragmenten binnen `realtime.fastContext.timeoutMs` aan het realtime-model, voordat alleen op de volledige consultatieagent wordt teruggevallen als `realtime.fastContext.fallbackToConsult` waar is.
- Als `realtime.provider` naar een niet-geregistreerde provider verwijst, of er helemaal geen realtime-spraakprovider is geregistreerd, registreert Voice Call een waarschuwing en slaat het realtime-media over in plaats van de hele plugin te laten mislukken.
- `inboundPolicy` mag niet `"disabled"` zijn wanneer `realtime.enabled` waar is; `validateProviderConfig` wijst die combinatie af.
- Consultatiesessiesleutels gebruiken waar mogelijk de opgeslagen oproepsessie opnieuw en vallen daarna terug op de geconfigureerde `sessionScope` (standaard `per-phone`, of `per-call` voor geïsoleerde oproepen).

### Toolbeleid

`realtime.toolPolicy` beheert de consultatierun:

| Beleid           | Gedrag                                                                                                                                 |
| ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `safe-read-only` | Stel de consultatietool beschikbaar en beperk de reguliere agent tot `read`, `web_search`, `web_fetch`, `x_search`, `memory_search` en `memory_get`. |
| `owner`          | Stel de consultatietool beschikbaar en laat de reguliere agent het normale agenttoolbeleid gebruiken.                                                      |
| `none`           | Stel de consultatietool niet beschikbaar. Aangepaste `realtime.tools` worden nog steeds doorgegeven aan de realtime-provider.                               |

`realtime.consultPolicy` beheert alleen de instructies voor het realtime-model:

| Beleid        | Richtlijn                                                                                        |
| ------------- | ----------------------------------------------------------------------------------------------- |
| `auto`        | Behoud de standaardprompt en laat de provider bepalen wanneer de consultatietool wordt aangeroepen.              |
| `substantive` | Beantwoord eenvoudige verbindende gesprekszinnen rechtstreeks en consulteer vóór feiten, geheugen, tools of context. |
| `always`      | Consulteer vóór elk inhoudelijk antwoord.                                                        |

### Spraakcontext van de agent

Schakel `realtime.agentContext` in wanneer de spraakbrug moet klinken als de
geconfigureerde OpenClaw-agent zonder voor gewone beurten een volledige
heen-en-terugoproep naar de consultatieagent uit te voeren. De contextcapsule
wordt eenmaal toegevoegd wanneer de realtime-sessie wordt gemaakt en voegt
dus geen latentie per beurt toe. Aanroepen van `openclaw_agent_consult` voeren nog
steeds de volledige OpenClaw-agent uit en moeten worden gebruikt voor
toolwerk, actuele informatie, geheugenzoekopdrachten of de werkruimtestatus.

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          agentId: "main",
          realtime: {
            enabled: true,
            provider: "google",
            toolPolicy: "safe-read-only",
            consultPolicy: "substantive",
            agentContext: {
              enabled: true,
              maxChars: 6000,
              includeIdentity: true,
              includeWorkspaceFiles: true,
              files: ["SOUL.md", "IDENTITY.md", "USER.md"],
            },
          },
        },
      },
    },
  },
}
```

### Voorbeelden van realtime-providers

<Tabs>
  <Tab title="Google Gemini Live">
    Standaardwaarden: API-sleutel uit `realtime.providers.google.apiKey`, `GEMINI_API_KEY`
    of `GOOGLE_API_KEY`; model `gemini-3.1-flash-live-preview`;
    stem `Kore`. `sessionResumption` en `contextWindowCompression` zijn standaard
    ingeschakeld voor langere oproepen waarmee opnieuw verbinding kan worden gemaakt. Gebruik `silenceDurationMs`,
    `startSensitivity` en `endSensitivity` om snellere beurtwisseling af te stemmen
    voor telefonieaudio.

    ```json5
    {
      plugins: {
        entries: {
          "voice-call": {
            config: {
              provider: "twilio",
              inboundPolicy: "allowlist",
              allowFrom: ["+15550005678"],
              realtime: {
                enabled: true,
                provider: "google",
                instructions: "Spreek kort. Roep openclaw_agent_consult aan voordat je diepgaandere tools gebruikt.",
                toolPolicy: "safe-read-only",
                consultPolicy: "substantive",
                consultThinkingLevel: "low",
                consultFastMode: true,
                agentContext: { enabled: true },
                providers: {
                  google: {
                    apiKey: "${GEMINI_API_KEY}",
                    model: "gemini-3.1-flash-live-preview",
                    speakerVoice: "Kore",
                    silenceDurationMs: 500,
                    startSensitivity: "high",
                  },
                },
              },
            },
          },
        },
      },
    }
    ```

  </Tab>
  <Tab title="OpenAI">
    ```json5
    {
      plugins: {
        entries: {
          "voice-call": {
            config: {
              realtime: {
                enabled: true,
                provider: "openai",
                providers: {
                  openai: { apiKey: "${OPENAI_API_KEY}" },
                },
              },
            },
          },
        },
      },
    }
    ```
  </Tab>
</Tabs>

Zie [Google-provider](/nl/providers/google) en
[OpenAI-provider](/nl/providers/openai) voor providerspecifieke opties voor
realtime-spraak.

## Streamingtranscriptie

`streaming` verbindt Twilio Media Streams met een realtime-transcriptieprovider.
Het klassieke streamingpad vereist `provider: "twilio"`; configuratie met
Telnyx, Plivo of mock wordt afgewezen. Live-audio van Telnyx gebruikt in plaats
daarvan het afzonderlijk geauthenticeerde pad `realtime.enabled`.

Huidig runtimegedrag:

- `streaming.provider` is optioneel. Als dit niet is ingesteld, gebruikt Voice Call de eerste geregistreerde realtime-transcriptieprovider.
- Gebundelde realtime-transcriptieproviders: Deepgram (`deepgram`), ElevenLabs (`elevenlabs`), Mistral (`mistral`), OpenAI (`openai`) en xAI (`xai`), geregistreerd door hun providerplugins.
- Door de provider beheerde onbewerkte configuratie staat onder `streaming.providers.<providerId>`.
- Nadat Twilio een geaccepteerd streambericht `start` verzendt, registreert Voice Call de stream onmiddellijk, plaatst het binnenkomende media via de transcriptieprovider in de wachtrij terwijl de provider verbinding maakt en start het de eerste begroeting pas wanneer realtime-transcriptie gereed is.
- Als `streaming.provider` naar een niet-geregistreerde provider verwijst, of er geen is geregistreerd, registreert Voice Call een waarschuwing en slaat het mediastreaming over in plaats van de hele plugin te laten mislukken.

### Voorbeelden van streamingproviders

<Tabs>
  <Tab title="OpenAI">
    Standaardwaarden: API-sleutel `streaming.providers.openai.apiKey` of
    `OPENAI_API_KEY`; model `gpt-4o-transcribe`; `silenceDurationMs: 800`;
    `vadThreshold: 0.5`.

    ```json5
    {
      plugins: {
        entries: {
          "voice-call": {
            config: {
              streaming: {
                enabled: true,
                provider: "openai",
                streamPath: "/voice/stream",
                providers: {
                  openai: {
                    apiKey: "sk-...", // optioneel als OPENAI_API_KEY is ingesteld
                    model: "gpt-4o-transcribe",
                    silenceDurationMs: 800,
                    vadThreshold: 0.5,
                  },
                },
              },
            },
          },
        },
      },
    }
    ```

  </Tab>
  <Tab title="xAI">
    Standaardwaarden: API-sleutel `streaming.providers.xai.apiKey` of `XAI_API_KEY` (valt
    terug op een xAI OAuth-authenticatieprofiel als geen van beide is ingesteld); eindpunt
    `wss://api.x.ai/v1/stt`; codering `mulaw`; samplefrequentie `8000`;
    `endpointingMs: 800`; `interimResults: true`.

    ```json5
    {
      plugins: {
        entries: {
          "voice-call": {
            config: {
              streaming: {
                enabled: true,
                provider: "xai",
                streamPath: "/voice/stream",
                providers: {
                  xai: {
                    apiKey: "${XAI_API_KEY}", // optioneel als XAI_API_KEY is ingesteld
                    endpointingMs: 800,
                    language: "en",
                  },
                },
              },
            },
          },
        },
      },
    }
    ```

  </Tab>
</Tabs>

## TTS voor oproepen

Voice Call gebruikt de kernconfiguratie `tts` voor streaming-spraak tijdens
oproepen. Je kunt deze onder de pluginconfiguratie overschrijven met **dezelfde structuur** —
deze wordt diep samengevoegd met `tts`.

```json5
{
  tts: {
    provider: "elevenlabs",
    providers: {
      elevenlabs: {
        speakerVoiceId: "pMsXgVXv3BLzUgSXRplE",
        modelId: "eleven_multilingual_v2",
      },
    },
  },
}
```

<Warning>
**Microsoft-spraak wordt genegeerd voor spraakoproepen.** Telefoniesynthese vereist
een provider die uitvoer voor telefoniedoeleinden implementeert; de
Microsoft-spraakprovider doet dit niet en wordt daarom overgeslagen voor
oproepen. In plaats daarvan worden andere providers in de terugvalketen geprobeerd.
</Warning>

Opmerkingen over gedrag:

- Verouderde `tts.<provider>`-sleutels binnen de pluginconfiguratie (`openai`, `elevenlabs`, `microsoft`, `edge`) worden hersteld door `openclaw doctor --fix`; vastgelegde configuratie moet `tts.providers.<provider>` gebruiken.
- Kern-TTS wordt gebruikt wanneer Twilio-mediastreaming is ingeschakeld; anders vallen oproepen terug op providerspecifieke stemmen.
- Als er al een Twilio-mediastream actief is, valt Voice Call niet terug op TwiML `<Say>`. Als telefonie-TTS in die toestand niet beschikbaar is, mislukt het afspeelverzoek in plaats van twee afspeelpaden te combineren.
- Wanneer telefonie-TTS terugvalt op een secundaire provider, registreert Voice Call voor foutopsporing een waarschuwing met de providerketen (`from`, `to`, `attempts`).
- Wanneer Twilio-barge-in of het afbreken van de stream de wachtende TTS-wachtrij wist, worden in de wachtrij geplaatste afspeelverzoeken afgehandeld in plaats van bellers die op voltooiing van het afspelen wachten te laten hangen.

### TTS-voorbeelden

<Tabs>
  <Tab title="Alleen kern-TTS">
```json5
{
  tts: {
    provider: "openai",
    providers: {
      openai: { speakerVoice: "alloy" },
    },
  },
}
```
  </Tab>
  <Tab title="Overschrijven met ElevenLabs (alleen oproepen)">
```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          tts: {
            provider: "elevenlabs",
            providers: {
              elevenlabs: {
                apiKey: "elevenlabs_key",
                speakerVoiceId: "pMsXgVXv3BLzUgSXRplE",
                modelId: "eleven_multilingual_v2",
              },
            },
          },
        },
      },
    },
  },
}
```
  </Tab>
  <Tab title="OpenAI-model overschrijven (diep samenvoegen)">
```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          tts: {
            providers: {
              openai: {
                model: "gpt-4o-mini-tts",
                speakerVoice: "marin",
              },
            },
          },
        },
      },
    },
  },
}
```
  </Tab>
</Tabs>

## Inkomende oproepen

Het beleid voor inkomende oproepen is standaard `disabled`. Stel het volgende in om inkomende oproepen in te schakelen:

```json5
{
  inboundPolicy: "allowlist",
  allowFrom: ["+15550001234"],
  inboundGreeting: "Hallo! Hoe kan ik helpen?",
}
```

<Warning>
`inboundPolicy: "allowlist"` is een controle van de beller-ID met een laag betrouwbaarheidsniveau. De plugin
normaliseert de door de provider geleverde waarde `From` en vergelijkt deze met `allowFrom`.
Webhook-verificatie verifieert de levering door de provider en de integriteit van de payload,
maar bewijst **niet** het eigendom van het PSTN-/VoIP-nummer van de beller. Beschouw
`allowFrom` als filtering op beller-ID, niet als sterke identificatie van de beller.
</Warning>

Automatische antwoorden gebruiken het agentsysteem. Stem dit af met `responseModel`,
`responseSystemPrompt` en `responseTimeoutMs`.

### Routering per nummer

Gebruik `numbers` wanneer één Voice Call-plugin oproepen voor meerdere telefoon-
nummers ontvangt en elk nummer zich als een afzonderlijke lijn moet gedragen. Het ene
nummer kan bijvoorbeeld een informele persoonlijke assistent gebruiken, terwijl een ander een zakelijke
persona, een andere antwoordagent en een andere TTS-stem gebruikt.

Routes worden geselecteerd op basis van het door de provider geleverde, gebelde nummer `To`. Sleutels moeten
E.164-nummers zijn. Wanneer een oproep binnenkomt, bepaalt Voice Call eenmaal de overeenkomende
route, slaat de gevonden route op in de oproeprecord en hergebruikt die
effectieve configuratie voor de begroeting, het klassieke pad voor automatische antwoorden, het realtime
consultatiepad en TTS-weergave. Als geen route overeenkomt, wordt de algemene Voice Call-
configuratie gebruikt. Uitgaande oproepen gebruiken `numbers` niet; geef het uitgaande
doel, het bericht en de sessie expliciet door wanneer je de oproep start.

Routeoverschrijvingen ondersteunen momenteel:

- `inboundGreeting`
- `tts`
- `agentId`
- `responseModel`
- `responseSystemPrompt`
- `responseTimeoutMs`

De routewaarde `tts` wordt diep samengevoegd boven op de algemene Voice Call-configuratie `tts`, zodat
je doorgaans alleen de providerstem hoeft te overschrijven:

```json5
{
  inboundGreeting: "Hallo vanaf de hoofdlijn.",
  responseSystemPrompt: "Je bent de standaard spraakassistent.",
  tts: {
    provider: "openai",
    providers: {
      openai: { speakerVoice: "coral" },
    },
  },
  numbers: {
    "+15550001111": {
      inboundGreeting: "Silver Fox Cards, hoe kan ik helpen?",
      responseSystemPrompt: "Je bent een beknopte specialist in honkbalkaarten.",
      tts: {
        providers: {
          openai: { speakerVoice: "alloy" },
        },
      },
    },
  },
}
```

### Contract voor gesproken uitvoer

Voor automatische antwoorden voegt Voice Call een strikt contract voor gesproken uitvoer toe aan
de systeemprompt, dat een JSON-antwoord van het type `{"spoken":"..."}` vereist. Voice Call
extraheert de gesproken tekst defensief:

- Negeert payloads die als redeneer-/foutinhoud zijn gemarkeerd.
- Parseert rechtstreekse JSON, JSON in een codeblok of inline sleutels van het type `"spoken"`.
- Valt terug op platte tekst en verwijdert waarschijnlijke inleidende alinea's met planning/metatekst.

Hierdoor blijft de gesproken weergave gericht op tekst voor de beller en wordt voorkomen dat
planningstekst in de audio terechtkomt.

### Gedrag bij het starten van een gesprek

Voor uitgaande oproepen van het type `conversation` is de verwerking van het eerste bericht gekoppeld aan de actuele
afspeelstatus:

- Het wissen van de wachtrij bij onderbreken en automatisch antwoorden worden alleen onderdrukt zolang de eerste begroeting actief wordt uitgesproken.
- Als de eerste weergave mislukt, keert de oproep terug naar `listening` en blijft het eerste bericht in de wachtrij staan om opnieuw te proberen.
- De eerste weergave voor Twilio-streaming begint zonder extra vertraging zodra de stream verbinding maakt.
- Onderbreken breekt actieve weergave af en wist Twilio TTS-items die in de wachtrij staan maar nog niet worden afgespeeld. Gewiste items worden als overgeslagen afgehandeld, zodat de vervolglogica voor antwoorden kan doorgaan zonder te wachten op audio die nooit zal worden afgespeeld.
- Realtime spraakgesprekken gebruiken de eigen openingsbeurt van de realtime stream. Voice Call plaatst **geen** verouderde TwiML-update van het type `<Say>` voor dat eerste bericht, zodat uitgaande sessies van het type `<Connect><Stream>` verbonden blijven.

### Respijtperiode bij verbreken van Twilio-stream

Wanneer de verbinding met een Twilio-mediastream wordt verbroken, wacht Voice Call **2000 ms** voordat
de oproep automatisch wordt beëindigd:

- Als de stream binnen die periode opnieuw verbinding maakt, wordt het automatisch beëindigen geannuleerd.
- Als na de respijtperiode geen stream opnieuw wordt geregistreerd, wordt de oproep beëindigd om vastgelopen actieve oproepen te voorkomen.

## Opruimer voor verouderde oproepen

Gebruik `staleCallReaperSeconds` (standaard **120**) om oproepen te beëindigen die nooit worden
beantwoord en nooit een actieve gespreksstatus bereiken, bijvoorbeeld oproepen in meldingsmodus
waarbij de provider nooit een afsluitende Webhook levert. Stel dit in op `0` om
het uit te schakelen.

De opruimer wordt elke 30 seconden uitgevoerd en beëindigt alleen oproepen zonder
tijdstempel `answeredAt` die nog niet de eindstatus of actieve
status (`speaking`/`listening`) hebben. Beantwoorde gesprekken worden dus nooit door deze timer
opgeruimd; `maxDurationSeconds` (standaard 300) is de afzonderlijke limiet die
beantwoorde oproepen beëindigt wanneer ze te lang duren.

Voor meldingsachtige flows waarbij providers traag kunnen zijn met het leveren van Webhooks
voor overgaan/beantwoorden, verhoog je `staleCallReaperSeconds` boven de standaardwaarde, zodat trage maar normale
oproepen niet voortijdig worden opgeruimd; `120`-`300` seconden is een redelijk bereik voor productie.

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          maxDurationSeconds: 300,
          staleCallReaperSeconds: 120,
        },
      },
    },
  },
}
```

## Webhook-beveiliging

Wanneer een proxy of tunnel vóór de Gateway staat, reconstrueert de plugin
de openbare URL voor handtekeningverificatie. Deze opties bepalen welke
doorgestuurde headers worden vertrouwd:

<ParamField path="webhookSecurity.allowedHosts" type="string[]">
  Hosts uit doorstuurheaders op de toelatingslijst.
</ParamField>
<ParamField path="webhookSecurity.trustForwardingHeaders" type="boolean">
  Vertrouw doorgestuurde headers zonder toelatingslijst.
</ParamField>
<ParamField path="webhookSecurity.trustedProxyIPs" type="string[]">
  Vertrouw doorgestuurde headers alleen wanneer het externe IP-adres van het verzoek overeenkomt met de lijst.
</ParamField>

Aanvullende beveiligingen:

- Webhook-**beveiliging tegen herhaling** is ingeschakeld voor Twilio, Telnyx en Plivo. Herhaalde geldige Webhook-verzoeken worden bevestigd, maar bijwerkingen worden overgeslagen.
- Twilio-gespreksbeurten bevatten een token per beurt in callbacks van het type `<Gather>`, zodat verouderde/herhaalde spraakcallbacks niet kunnen voldoen aan een nieuwere wachtende transcriptiebeurt.
- Niet-geverifieerde Webhook-verzoeken worden vóór het lezen van de body geweigerd wanneer de vereiste handtekeningheaders van de provider ontbreken.
- De voice-call-Webhook gebruikt vóór handtekeningverificatie het gedeelde profiel voor het lezen van de body vóór authenticatie (maximale body van 64 KB, leestime-out van 5 seconden), plus een limiet per sleutel voor gelijktijdig verwerkte verzoeken (standaard 8 gelijktijdige verzoeken per sleutel).

Voorbeeld met een stabiele openbare host:

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          publicUrl: "https://voice.example.com/voice/webhook",
          webhookSecurity: {
            allowedHosts: ["voice.example.com"],
          },
        },
      },
    },
  },
}
```

## CLI

```bash
openclaw voicecall call --to "+15555550123" --message "Hallo van OpenClaw"
openclaw voicecall start --to "+15555550123"   # alias voor call
openclaw voicecall continue --call-id <id> --message "Nog vragen?"
openclaw voicecall speak --call-id <id> --message "Een ogenblik"
openclaw voicecall dtmf --call-id <id> --digits "ww123456#"
openclaw voicecall end --call-id <id>
openclaw voicecall status --call-id <id>
openclaw voicecall tail
openclaw voicecall latency                      # vat de latentie per beurt samen op basis van logboeken
openclaw voicecall expose --mode funnel
```

Wanneer de Gateway al actief is, delegeren operationele opdrachten van het type `voicecall`
naar de door de Gateway beheerde voice-call-runtime, zodat de CLI geen
tweede Webhook-server bindt. Als er geen Gateway bereikbaar is, vallen de opdrachten terug op
een zelfstandige CLI-runtime.

`latency` leest `calls.jsonl` uit het standaardopslagpad voor voice-call. Gebruik
`--file <path>` om naar een ander logboek te verwijzen en `--last <n>` om
de analyse te beperken tot de laatste N records (standaard 200). De uitvoer bevat min/max/gem,
p50 en p95 voor de latentie per beurt en de wachttijden voor luisteren.

## Agenttool

Toolnaam: `voice_call`.

| Actie           | Argumenten                                  |
| --------------- | ------------------------------------------ |
| `initiate_call` | `message`, `to?`, `mode?`, `dtmfSequence?` |
| `continue_call` | `callId`, `message`                        |
| `speak_to_user` | `callId`, `message`                        |
| `send_dtmf`     | `callId`, `digits`                         |
| `end_call`      | `callId`                                   |
| `get_status`    | `callId`                                   |

De voice-call-plugin levert een bijpassende agentskill.

## Gateway-RPC

| Methode                     | Argumenten                                                        | Opmerkingen                                                               |
| --------------------------- | ---------------------------------------------------------------- | ------------------------------------------------------------------------- |
| `voicecall.initiate`        | `to?`, `message`, `mode?`, `sessionKey?`, `requesterSessionKey?` | Valt terug op de configuratie van `toNumber` wanneer `to` is weggelaten. |
| `voicecall.start`           | `to`, `message?`, `mode?`, `dtmfSequence?`, `sessionKey?`        | Hetzelfde als `initiate`, maar accepteert ook `dtmfSequence` vóór de verbinding. |
| `voicecall.continue`        | `callId`, `message`                                              | Blokkeert totdat de beurt is afgehandeld en retourneert het transcript.   |
| `voicecall.continue.start`  | `callId`, `message`                                              | Asynchrone variant: retourneert onmiddellijk een `operationId`.      |
| `voicecall.continue.result` | `operationId`                                                    | Vraagt het resultaat van een wachtende `voicecall.continue.start`-bewerking op. |
| `voicecall.speak`           | `callId`, `message`                                              | Spreekt zonder te wachten en gebruikt de realtime-bridge wanneer `realtime.enabled`. |
| `voicecall.dtmf`            | `callId`, `digits`                                               |                                                                           |
| `voicecall.end`             | `callId`                                                         |                                                                           |
| `voicecall.status`          | `callId?`                                                        | Laat `callId` weg om alle actieve gesprekken weer te geven.     |

`dtmfSequence` is alleen geldig met `mode: "conversation"`; gesprekken in meldingsmodus
moeten `voicecall.dtmf` gebruiken nadat het gesprek bestaat als ze cijfers
na het verbinden nodig hebben.

## Probleemoplossing

### Instellen van webhooktoegang mislukt

Voer de instelling uit vanuit dezelfde omgeving waarin de Gateway wordt uitgevoerd:

```bash
openclaw voicecall setup
openclaw voicecall setup --json
```

Voor `twilio`, `telnyx` en `plivo` moet `webhook-exposure` groen zijn. Een
geconfigureerde `publicUrl` mislukt nog steeds wanneer deze naar lokale of privé-
netwerkruimte verwijst, omdat de telecomprovider die adressen niet kan terugbellen.
Gebruik `localhost`, `127.0.0.1`, `0.0.0.0`, `10.x`, `172.16.x`-`172.31.x`,
`192.168.x`, `169.254.x`, `fc00::/7`, `fd00::/8` of andere carrier-grade-NAT-
bereiken niet als `publicUrl`.

Uitgaande Twilio-gesprekken in meldingsmodus sturen hun initiële `<Say>`-TwiML rechtstreeks
mee in het verzoek om een gesprek te starten, zodat het eerste gesproken bericht niet afhankelijk is
van het ophalen van webhook-TwiML door Twilio. Een openbare webhook blijft vereist voor status-
callbacks, conversatiegesprekken, DTMF vóór het verbinden, realtime-streams en
gespreksbesturing na het verbinden.

Gebruik één openbaar toegangspad:

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          publicUrl: "https://voice.example.com/voice/webhook",
          // of
          tunnel: { provider: "ngrok" },
          // of
          tailscale: { mode: "funnel", path: "/voice/webhook" },
        },
      },
    },
  },
}
```

Start na een configuratiewijziging de Gateway opnieuw of laad deze opnieuw en voer daarna uit:

```bash
openclaw voicecall setup
openclaw voicecall smoke
```

`voicecall smoke` is een proefuitvoering, tenzij je `--yes` meegeeft.

### Providerreferenties mislukken

Controleer de geselecteerde provider en de vereiste referentievelden:

- Twilio: `twilio.accountSid`, `twilio.authToken` en `fromNumber`, of
  `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN` en `TWILIO_FROM_NUMBER`.
- Telnyx: `telnyx.apiKey`, `telnyx.connectionId`, `telnyx.publicKey` en
  `fromNumber`, of `TELNYX_API_KEY`, `TELNYX_CONNECTION_ID` en
  `TELNYX_PUBLIC_KEY`.
- Plivo: `plivo.authId`, `plivo.authToken` en `fromNumber`, of
  `PLIVO_AUTH_ID` en `PLIVO_AUTH_TOKEN`.

De referenties moeten op de Gateway-host aanwezig zijn. Het bewerken van een lokaal shellprofiel
heeft geen invloed op een Gateway die al wordt uitgevoerd, totdat deze opnieuw wordt gestart of zijn
omgeving opnieuw laadt.

### Gesprekken starten, maar providerwebhooks komen niet aan

Controleer of de providerconsole naar de exacte openbare webhook-URL verwijst:

```text
https://voice.example.com/voice/webhook
```

Inspecteer daarna de runtimestatus:

```bash
openclaw voicecall status --call-id <id>
openclaw voicecall tail
openclaw logs --follow
```

Veelvoorkomende oorzaken:

- `publicUrl` verwijst naar een ander pad dan `serve.path`.
- De tunnel-URL is gewijzigd nadat de Gateway is gestart.
- Een proxy stuurt het verzoek door, maar verwijdert of herschrijft host-/protoheaders.
- De firewall of DNS routeert de openbare hostnaam naar een andere locatie dan de Gateway.
- De Gateway is opnieuw gestart zonder dat de Voice Call-plugin was ingeschakeld.

Wanneer zich een reverse proxy of tunnel vóór de Gateway bevindt, stel je
`webhookSecurity.allowedHosts` in op de openbare hostnaam of gebruik je
`webhookSecurity.trustedProxyIPs` voor een bekend proxyadres. Gebruik
`webhookSecurity.trustForwardingHeaders` alleen wanneer je de proxygrens
zelf beheert.

### Handtekeningverificatie mislukt

Providerhandtekeningen worden gecontroleerd aan de hand van de openbare URL die OpenClaw
reconstrueert uit het binnenkomende verzoek. Als handtekeningen mislukken:

- Controleer of de webhook-URL van de provider exact overeenkomt met `publicUrl`, inclusief schema, host en pad.
- Werk bij gratis ngrok-URL's `publicUrl` bij wanneer de tunnelhostnaam verandert.
- Zorg dat de proxy de oorspronkelijke host- en protoheaders behoudt of configureer `webhookSecurity.allowedHosts`.
- Schakel `skipSignatureVerification` niet in buiten lokale tests.

### Deelname aan Google Meet via Twilio mislukt

Google Meet gebruikt deze Plugin voor deelname via Twilio-inbellen. Controleer eerst Voice
Call:

```bash
openclaw voicecall setup
openclaw voicecall smoke --to "+15555550123"
```

Controleer daarna expliciet het Google Meet-transport:

```bash
openclaw googlemeet setup --transport twilio
```

Als Voice Call groen is, maar de Meet-deelnemer nooit deelneemt, controleer dan het Meet-
inbelnummer, de pincode en `--dtmf-sequence`. Het telefoongesprek kan correct werken,
terwijl de vergadering een onjuiste DTMF-reeks weigert of negeert.

Google Meet start het Twilio-telefoongedeelte via `voicecall.start` met een
DTMF-reeks vóór het verbinden. Van pincodes afgeleide reeksen bevatten de
`voiceCall.dtmfDelayMs` van de Google Meet-plugin (standaard **12000 ms**) als voorloop-
wachtcijfers voor Twilio, omdat Meet-inbelprompts laat kunnen verschijnen. Voice Call leidt
daarna terug naar realtime-afhandeling voordat om de introductiebegroeting wordt gevraagd.

Gebruik `openclaw logs --follow` voor de live fasetracering. Een geslaagde Twilio Meet-
deelname registreert deze volgorde:

- Google Meet delegeert de Twilio-deelname aan Voice Call.
- Voice Call slaat DTMF-TwiML voor het verbinden op.
- De initiële TwiML van Twilio wordt verwerkt en aangeboden vóór de realtime-afhandeling.
- Voice Call biedt realtime-TwiML aan voor het Twilio-gesprek.
- Google Meet vraagt na de vertraging na DTMF om introductiespraak met `voicecall.speak`.

`openclaw voicecall tail` toont nog steeds opgeslagen gespreksrecords; nuttig voor
gespreksstatus en transcripties, maar niet elke webhook-/realtime-overgang
verschijnt daar.

### Realtime-gesprek heeft geen spraak

Controleer of slechts één audiomodus is ingeschakeld: `realtime.enabled` en
`streaming.enabled` kunnen niet beide waar zijn.

Controleer voor realtime Twilio-/Telnyx-gesprekken ook het volgende:

- Er is een realtime-providerplugin geladen en geregistreerd.
- `realtime.provider` is niet ingesteld of noemt een geregistreerde provider.
- De API-sleutel van de provider is beschikbaar voor het Gateway-proces.
- `openclaw logs --follow` toont dat realtime-TwiML is aangeboden, de realtime-bridge is gestart en de initiële begroeting in de wachtrij is geplaatst.

## Gerelateerd

- [Gespreksmodus](/nl/nodes/talk)
- [Tekst-naar-spraak](/nl/tools/tts)
- [Spraakactivering](/nl/nodes/voicewake)
