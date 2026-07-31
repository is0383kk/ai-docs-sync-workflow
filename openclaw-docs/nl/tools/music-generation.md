---
read_when:
    - Muziek of audio genereren via de agent
    - Muziekgeneratieproviders en -modellen configureren
    - De parameters van de tool music_generate begrijpen
sidebarTitle: Music generation
summary: Genereer muziek via music_generate in workflows voor ComfyUI, fal, Google Lyria, MiniMax en OpenRouter
title: Muziekgeneratie
x-i18n:
    generated_at: "2026-07-27T05:37:23Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3f2a8a4a36e47839c7896046a556f7bf84f6c168492e2de46736635fe2a9358e
    source_path: tools/music-generation.md
    workflow: 16
---

De tool `music_generate` maakt muziek of audio via de gedeelde
mogelijkheid voor muziekgeneratie, ondersteund door ComfyUI, fal, Google, MiniMax en
OpenRouter.

<Note>
`music_generate` verschijnt alleen wanneer ten minste één provider voor muziekgeneratie
beschikbaar is: een expliciete `agents.defaults.mediaModels.music`-configuratie of een
provider met geconfigureerde authenticatie (bijvoorbeeld een ingestelde API-sleutel).
</Note>

Voor agentuitvoeringen met een sessie start `music_generate` als achtergrondtaak,
houdt de voortgang bij in het taakregister en wekt vervolgens de agent wanneer de track
gereed is, zodat die de gebruiker kan informeren en de voltooide audio kan bijvoegen. De
voltooiingsagent volgt het contract voor zichtbare antwoorden van de sessie: automatisch
definitief antwoorden wanneer dit is geconfigureerd, of `message(action="send")` wanneer de
sessie de berichtentool vereist. Als de sessie van de aanvrager inactief is of niet kan
worden gewekt en de gegenereerde audio nog steeds in het antwoord ontbreekt, verzendt
OpenClaw een idempotente directe fallback met alleen de ontbrekende audio.

## Snel aan de slag

<Tabs>
  <Tab title="Ondersteund door gedeelde provider">
    <Steps>
      <Step title="Authenticatie configureren">
        Stel een API-sleutel in voor ten minste één provider, bijvoorbeeld
        `GEMINI_API_KEY` of `MINIMAX_API_KEY`.
      </Step>
      <Step title="Een standaardmodel kiezen (optioneel)">
        ```json5
        {
          agents: {
            defaults: {
              musicGenerationModel: {
                primary: "google/lyria-3-clip-preview",
              },
            },
          },
        }
        ```
      </Step>
      <Step title="De agent een opdracht geven">
        _"Genereer een vrolijke synthpoptrack over een nachtelijke autorit door een
        neonstad."_

        De agent roept `music_generate` automatisch aan. Er is geen
        toelatingslijst voor tools nodig.
      </Step>
    </Steps>

    Zonder een agentuitvoering met een sessie (directe/lokale contexten) wordt de tool
    inline uitgevoerd en retourneert deze het definitieve mediapad in hetzelfde toolresultaat.

  </Tab>
  <Tab title="ComfyUI-workflow">
    <Steps>
      <Step title="De workflow configureren">
        Configureer `plugins.entries.comfy.config.music` met workflow-
        JSON en prompt-/uitvoerknooppunten.
      </Step>
      <Step title="Cloudauthenticatie (optioneel)">
        Stel voor Comfy Cloud `COMFY_API_KEY` of `COMFY_CLOUD_API_KEY` in.
      </Step>
      <Step title="De tool aanroepen">
        ```text
        /tool music_generate prompt="Warme ambient-synthloop met een zachte bandtextuur"
        ```
      </Step>
    </Steps>
  </Tab>
</Tabs>

Voorbeeldprompts:

```text
Genereer een filmische pianotrack met zachte strijkers en zonder zang.
```

```text
Genereer een energieke chiptuneloop over het lanceren van een raket bij zonsopgang.
```

Gebruik `action: "list"` om beschikbare providers/modellen te bekijken en
`action: "status"` om de actieve muziektaak met een sessie te bekijken:

```text
/tool music_generate action=list
/tool music_generate action=status
```

Voorbeeld van directe generatie:

```text
/tool music_generate prompt="Dromerige lo-fi-hiphop met vinyltextuur en zachte regen" instrumental=true
```

## Ondersteunde providers

| Provider   | Standaardmodel                | Referentie-invoer | Ondersteunde instellingen                              | Authenticatie                         |
| ---------- | ----------------------------- | ----------------- | ------------------------------------------------------ | ------------------------------------- |
| ComfyUI    | `workflow`            | Maximaal 1 afbeelding | Door de workflow gedefinieerde muziek of audio     | `COMFY_API_KEY`, `COMFY_CLOUD_API_KEY` |
| fal        | `fal-ai/minimax-music/v2.6`            | Geen              | `lyrics`, `instrumental`, `durationSeconds`, `format` | `FAL_KEY` of `FAL_API_KEY` |
| Google     | `lyria-3-clip-preview`            | Maximaal 10 afbeeldingen | `lyrics`, `instrumental`, `format` | `GEMINI_API_KEY`, `GOOGLE_API_KEY` |
| MiniMax    | `music-2.6`            | Geen              | `lyrics`, `instrumental`, `format` (alleen mp3) | `MINIMAX_API_KEY` of MiniMax OAuth |
| OpenRouter | `google/lyria-3-pro-preview`            | Maximaal 1 afbeelding | `lyrics`, `instrumental`, `durationSeconds`, `format` | `OPENROUTER_API_KEY` |

MiniMax registreert twee provider-id's die dezelfde modellen delen: `minimax` voor
authenticatie met een API-sleutel en `minimax-portal` voor OAuth. Modelverwijzingen volgen
het authenticatiepad (`minimax/music-2.6` versus `minimax-portal/music-2.6`); zie
[MiniMax](/nl/providers/minimax#music-generation).

fal biedt naast het standaardmodel op basis van MiniMax ook
`fal-ai/ace-step/prompt-to-audio` (wav, geen songteksten, geen schakelaar voor
instrumentaal) en `fal-ai/stable-audio-25/text-to-audio` (wav,
alleen prompt). Het standaardmodel `lyria-3-clip-preview` van Google levert alleen mp3;
`lyria-3-pro-preview` ondersteunt ook wav. MiniMax biedt ook `music-2.6-free`,
`music-cover` en `music-cover-free`. OpenRouter biedt ook `google/lyria-3-clip-preview`.

### Mogelijkhedenmatrix

Het expliciete moduscontract dat wordt gebruikt door `music_generate`, contracttests en de
gedeelde live-controle:

| Provider   | `generate` | `edit` | Bewerkingslimiet | Gedeelde live-trajecten                                               |
| ---------- | :--------: | :----: | ----------------- | --------------------------------------------------------------------- |
| ComfyUI    |     ✓      |   ✓    | 1 afbeelding      | Niet in de gedeelde controle; gedekt door `extensions/comfy/comfy.live.test.ts`          |
| fal        |     ✓      |   —    | Geen              | `generate`                                                    |
| Google     |     ✓      |   ✓    | 10 afbeeldingen   | `generate`, `edit`                                |
| MiniMax    |     ✓      |   —    | Geen              | `generate`                                                    |
| OpenRouter |     ✓      |   ✓    | 1 afbeelding      | `generate`, `edit`                                |

## Toolparameters

<ParamField path="prompt" type="string" required>
  Prompt voor muziekgeneratie. Vereist voor `action: "generate"`.
</ParamField>
<ParamField path="action" type='"generate" | "status" | "list"' default="generate">
  `"status"` retourneert de huidige sessietaak; `"list"` bekijkt providers.
</ParamField>
<ParamField path="model" type="string">
  Overschrijving van provider/model (bijvoorbeeld `google/lyria-3-pro-preview`,
  `comfy/workflow`).
</ParamField>
<ParamField path="lyrics" type="string">
  Optionele songtekst wanneer de provider expliciete invoer van songteksten ondersteunt.
</ParamField>
<ParamField path="instrumental" type="boolean">
  Vraag uitsluitend instrumentale uitvoer aan wanneer de provider dit ondersteunt.
</ParamField>
<ParamField path="image" type="string">
  Pad of URL van één referentieafbeelding.
</ParamField>
<ParamField path="images" type="string[]">
  Meerdere referentieafbeeldingen (maximaal 10 bij providers die dit ondersteunen).
</ParamField>
<ParamField path="durationSeconds" type="number">
  Gewenste duur in seconden wanneer de provider duuraanwijzingen ondersteunt.
</ParamField>
<ParamField path="format" type='"mp3" | "wav"'>
  Aanwijzing voor de uitvoerindeling wanneer de provider dit ondersteunt.
</ParamField>
<ParamField path="filename" type="string">Aanwijzing voor de uitvoerbestandsnaam.</ParamField>

<Note>
Niet alle providers ondersteunen alle parameters. OpenClaw valideert harde
limieten, zoals aantallen invoeritems, nog steeds vóór verzending. Wanneer een provider
duur ondersteunt maar een korter maximum hanteert dan de aangevraagde waarde, beperkt
OpenClaw deze tot de dichtstbijzijnde ondersteunde duur. Echt niet-ondersteunde optionele
aanwijzingen worden met een waarschuwing genegeerd wanneer de geselecteerde provider of
het geselecteerde model deze niet kan toepassen. Toolresultaten vermelden de toegepaste
instellingen; `details.normalization` legt elke omzetting van aangevraagd naar toegepast vast.
</Note>

Time-outs voor providerverzoeken zijn uitsluitend configuratie voor beheerders. OpenClaw
gebruikt `agents.defaults.mediaModels.music.timeoutMs` wanneer dit is geconfigureerd, verhoogt
waarden onder 120000ms tot 120000ms en gebruikt anders standaard 300000ms
voor providerverzoeken.

## Asynchroon gedrag

Muziekgeneratie met een sessie wordt als achtergrondtaak uitgevoerd:

- **Achtergrondtaak:** `music_generate` maakt een achtergrondtaak, retourneert
  onmiddellijk een gestart-/taakantwoord en plaatst de voltooide track later in
  een vervolgbericht van de agent.
- **Dubbel werk voorkomen:** zolang een taak `queued` of `running` is,
  retourneren latere aanroepen van `music_generate` in dezelfde sessie de taakstatus
  in plaats van een nieuwe generatie te starten. Gebruik `action: "status"` om dit
  expliciet te controleren. Een recent voltooid, overeenkomend verzoek wordt ook
  gedurende 2 minuten gededupliceerd.
- **Status opzoeken:** `openclaw tasks list` of `openclaw tasks show <taskId>`
  bekijkt de status in de wachtrij, tijdens uitvoering en na beëindiging.
- **Wekken bij voltooiing:** OpenClaw injecteert een interne voltooiingsgebeurtenis terug
  in dezelfde sessie, zodat het model zelf het gebruikersgerichte vervolgbericht kan schrijven.
- **Promptaanwijzing:** latere gebruikers-/handmatige beurten in dezelfde sessie krijgen een
  kleine runtime-aanwijzing wanneer een muziektaak al wordt uitgevoerd, zodat het model
  `music_generate` niet blind opnieuw aanroept.
- **Fallback zonder sessie:** directe/lokale contexten zonder een echte
  agentsessie worden inline uitgevoerd en retourneren het definitieve audioresultaat in dezelfde beurt.

### Levenscyclus van taken

De muziektaak toont dezelfde statussen als het algemene taakregister (zie
[Achtergrondtaken](/nl/automation/tasks#task-lifecycle) voor de volledige
toestandsmachine, inclusief `timed_out`, `cancelled` en `lost`). De meeste muziekuitvoeringen
doorlopen:

| Status      | Betekenis                                                                                      |
| ----------- | ---------------------------------------------------------------------------------------------- |
| `queued` | Taak gemaakt; wacht totdat de provider deze accepteert.                                  |
| `running` | De provider verwerkt de taak (doorgaans 30 seconden tot 3 minuten, afhankelijk van provider en duur). |
| `succeeded` | Track gereed; de agent wordt gewekt en plaatst deze in het gesprek.                       |
| `failed` | Providerfout of time-out; de agent wordt gewekt met foutdetails.                          |

Controleer de status via de CLI:

```bash
openclaw tasks list
openclaw tasks show <taskId>
openclaw tasks cancel <taskId>
```

## Configuratie

### Modelselectie

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "google/lyria-3-clip-preview",
        fallbacks: ["fal/fal-ai/minimax-music/v2.6", "minimax/music-2.6"],
      },
    },
  },
}
```

### Selectievolgorde van providers

OpenClaw probeert providers in deze volgorde:

1. `model`-parameter uit de toolaanroep (als de agent er een opgeeft).
2. `musicGenerationModel.primary` uit de configuratie.
3. `musicGenerationModel.fallbacks` in volgorde.
4. Automatische detectie, uitsluitend met standaardproviders op basis van authenticatie:
   - eerst de huidige standaardprovider voor tekstmodellen, als deze ook
     muziekgeneratie aanbiedt;
   - de overige geregistreerde providers voor muziekgeneratie, alfabetisch op
     provider-id.

Als een provider mislukt, wordt de volgende kandidaat automatisch geprobeerd. Als alle
kandidaten mislukken, bevat de fout details van elke poging.

Automatische fallback tussen geauthenticeerde providers is altijd ingeschakeld. Een
`model` per aanroep blijft bepalend.

## Opmerkingen over providers

<AccordionGroup>
  <Accordion title="ComfyUI">
    Workflowgestuurd en afhankelijk van de geconfigureerde graaf plus de Node-toewijzing
    voor prompt-/uitvoervelden. De meegeleverde `comfy`-Plugin wordt via het
    register van providers voor muziekgeneratie gekoppeld aan de gedeelde
    `music_generate`-tool.
  </Accordion>
  <Accordion title="fal">
    Gebruikt fal-modeleindpunten via het gedeelde authenticatiepad voor providers. De
    meegeleverde provider gebruikt standaard `fal-ai/minimax-music/v2.6` en biedt ook
    `fal-ai/ace-step/prompt-to-audio` en
    `fal-ai/stable-audio-25/text-to-audio` voor prompt-naar-audioverzoeken.
    Songteksten en de instrumentale modus zijn alleen beschikbaar voor MiniMax-modellen; de andere twee
    modellen ondersteunen alleen prompts.
  </Accordion>
  <Accordion title="Google (Lyria 3)">
    Gebruikt batchgeneratie van Lyria 3. De huidige meegeleverde flow ondersteunt
    een prompt, optionele songtekst en optionele referentieafbeeldingen. Het
    standaardmodel `lyria-3-clip-preview` voert alleen mp3 uit; het
    model `lyria-3-pro-preview` ondersteunt ook wav.
  </Accordion>
  <Accordion title="MiniMax">
    Gebruikt het batch-eindpunt `music_generation`. Ondersteunt een prompt, optionele
    songtekst, instrumentale modus en mp3-uitvoer via authenticatie met een
    `minimax`-API-sleutel of `minimax-portal` OAuth. Biedt ook de modellen
    `music-2.6-free`, `music-cover` en `music-cover-free`.
  </Accordion>
  <Accordion title="OpenRouter">
    Gebruikt audio-uitvoer van OpenRouter-chatvoltooiingen met streaming ingeschakeld. De
    meegeleverde provider gebruikt standaard `google/lyria-3-pro-preview` en biedt ook
    `openrouter/google/lyria-3-clip-preview`.
  </Accordion>
</AccordionGroup>

## Het juiste pad kiezen

- **Gedeeld, ondersteund door providers** wanneer je modelselectie, provider-
  failover en de ingebouwde asynchrone taak-/statusflow wilt.
- **Pluginpad (ComfyUI)** wanneer je een aangepaste workflowgraaf nodig hebt of een
  provider die geen deel uitmaakt van de gedeelde meegeleverde muziekfunctionaliteit.

Als je ComfyUI-specifiek gedrag debugt, raadpleeg dan
[ComfyUI](/nl/providers/comfy). Als je gedeeld providergedrag debugt,
begin dan met [fal](/nl/providers/fal), [Google (Gemini)](/nl/providers/google),
[MiniMax](/nl/providers/minimax) of [OpenRouter](/nl/providers/openrouter).

## Providerfunctionaliteitsmodi

Het gedeelde contract voor muziekgeneratie ondersteunt expliciete modusdeclaraties:

- `generate` voor generatie op basis van alleen een prompt.
- `edit` wanneer het verzoek een of meer referentieafbeeldingen bevat.

Nieuwe providerimplementaties moeten bij voorkeur expliciete modusblokken gebruiken:

```typescript
capabilities: {
  generate: {
    maxTracks: 1,
    supportsLyrics: true,
    supportsFormat: true,
  },
  edit: {
    enabled: true,
    maxTracks: 1,
    maxInputImages: 1,
    supportsFormat: true,
  },
}
```

Oudere platte velden zoals `maxInputImages`, `supportsLyrics` en
`supportsFormat` zijn **niet** voldoende om bewerkingsondersteuning aan te geven. Providers
moeten `generate` en `edit` expliciet declareren, zodat livetests, contracttests
en de gedeelde `music_generate`-tool de modusondersteuning
deterministisch kunnen valideren.

## Livetests

Optionele live testdekking voor de gedeelde meegeleverde providers (fal, Google, MiniMax,
OpenRouter):

```bash
OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts
```

Gelijkwaardige repowrapper die hetzelfde testbestand uitvoert:

```bash
pnpm test:live:media:music
```

Dit livebestand gebruikt standaard reeds geëxporteerde provideromgevingsvariabelen vóór opgeslagen
authenticatieprofielen en voert zowel `generate` als de gedeclareerde `edit`-dekking uit wanneer
de provider de bewerkingsmodus inschakelt. Huidige dekking:

- `google`: `generate` plus `edit`
- `fal`: alleen `generate`
- `minimax`: alleen `generate`
- `openrouter`: `generate` plus `edit`
- `comfy`: afzonderlijke Comfy-livetestdekking, geen onderdeel van de gedeelde providerreeks

Optionele live testdekking voor het meegeleverde ComfyUI-muziekpad:

```bash
OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts
```

Het Comfy-livebestand dekt ook Comfy-workflows voor afbeeldingen en video's wanneer die
secties zijn geconfigureerd.

## Gerelateerd

- [Achtergrondtaken](/nl/automation/tasks) — taakregistratie voor losgekoppelde `music_generate`-uitvoeringen
- [ComfyUI](/nl/providers/comfy)
- [Configuratiereferentie](/nl/gateway/config-agents#agent-defaults) — `musicGenerationModel`-configuratie
- [Google (Gemini)](/nl/providers/google)
- [MiniMax](/nl/providers/minimax)
- [Modellen](/nl/concepts/models) — modelconfiguratie en failover
- [Overzicht van tools](/nl/tools)
