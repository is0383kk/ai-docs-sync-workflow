---
read_when:
    - Je wilt auto-compaction en /compact begrijpen
    - Je debugt lange sessies die tegen contextlimieten aanlopen
summary: Hoe OpenClaw lange gesprekken samenvat om binnen de modellimieten te blijven
title: Compaction
x-i18n:
    generated_at: "2026-07-27T05:30:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: eb1f794fa60affd602378bcff8b07786bfeca55ab3fa09d5fa7214a05fa48806
    source_path: concepts/compaction.md
    workflow: 16
---

Elk model heeft een contextvenster: het maximale aantal tokens dat het kan verwerken. Wanneer een gesprek die limiet nadert, **comprimeert** OpenClaw oudere berichten tot een samenvatting, zodat de chat kan doorgaan.

## Hoe het werkt

1. Oudere gespreksbeurten worden samengevat in een compacte vermelding.
2. De samenvatting wordt opgeslagen in het sessietranscript.
3. Recente berichten blijven intact.

OpenClaw houdt toolaanroepen van de assistent gekoppeld aan de bijbehorende `toolResult`-vermeldingen wanneer het een splitsingspunt voor Compaction kiest. Als het punt binnen een toolblok valt, verplaatst OpenClaw de grens zodat het paar bij elkaar blijft en het huidige niet-samengevatte einde behouden blijft.

De volledige gespreksgeschiedenis blijft op schijf staan. Compaction verandert alleen wat het model tijdens de volgende beurt ziet.

<Note>
Nieuwe configuraties stellen `agents.defaults.compaction.mode` standaard in op `"safeguard"` (strengere beveiligingsregels en controles op de kwaliteit van samenvattingen). Stel `mode: "default"` expliciet in om dit uit te schakelen.
</Note>

## Automatische Compaction

Automatische Compaction is standaard ingeschakeld. Deze wordt uitgevoerd wanneer de sessie de contextlimiet nadert, of wanneer het model een contextoverloopfout retourneert (in dat geval voert OpenClaw Compaction uit en probeert het opnieuw).

Je ziet:

- `embedded run auto-compaction start` / `complete` in normale Gateway-logboeken.
- `🧹 Auto-compaction complete` in de uitgebreide modus.
- `/status` met `🧹 Compactions: <count>`.

<Info>
Vóór Compaction herinnert OpenClaw de agent er automatisch aan belangrijke notities op te slaan in [geheugenbestanden](/nl/concepts/memory). Dit voorkomt contextverlies.
</Info>

<AccordionGroup>
  <Accordion title="Patronen van overloopfouten die OpenClaw herkent">
    OpenClaw herkent tientallen providerspecifieke foutteksten voor contextoverloop (Anthropic, OpenAI, Bedrock, Gemini, Ollama, OpenRouter en meer). Veelvoorkomende voorbeelden:

    - `request_too_large`
    - `context length exceeded`
    - `input exceeds the maximum number of tokens`
    - `input token count exceeds the maximum number of input tokens` (Bedrock)
    - `input is too long for the model`
    - `ollama error: context length exceeded`

  </Accordion>
</AccordionGroup>

## Handmatige Compaction

Typ `/compact` in een chat om Compaction af te dwingen. Voeg instructies toe om de samenvatting te sturen:

```text
/compact Richt je op de ontwerpbeslissingen voor de API
```

Wanneer `agents.defaults.compaction.keepRecentTokens` is ingesteld (standaard: 20,000), respecteert handmatige Compaction dat afkappunt en blijft het recente einde in de opnieuw opgebouwde context behouden. Zonder een expliciet behoudbudget werkt handmatige Compaction als een strikt controlepunt en gaat deze uitsluitend verder vanaf de nieuwe samenvatting.

## Configuratie

Configureer Compaction onder `agents.defaults.compaction` in je `openclaw.json`. De meest gebruikte instellingen staan hieronder; zie [Uitgebreide informatie over sessiebeheer](/nl/reference/session-management-compaction) voor de volledige referentie.

### Een ander model gebruiken

Compaction gebruikt standaard het primaire model van de agent. Stel `agents.defaults.compaction.model` in om het samenvatten over te dragen aan een krachtiger of gespecialiseerd model. De overschrijving accepteert een `provider/model-id`-tekenreeks of een kale alias die is geconfigureerd onder `agents.defaults.models`:

```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "model": "openrouter/anthropic/claude-sonnet-4-6"
      }
    }
  }
}
```

Kale geconfigureerde aliassen worden vóór het begin van Compaction omgezet naar hun canonieke provider en model. Als een kale waarde overeenkomt met zowel een alias als een geconfigureerde letterlijke model-ID, krijgt de letterlijke model-ID voorrang. Een kale waarde zonder overeenkomst blijft een model-ID bij de actieve provider.

Dit werkt ook met lokale modellen, bijvoorbeeld een tweede Ollama-model dat specifiek voor samenvattingen wordt gebruikt:

```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "model": "ollama/llama3.1:8b"
      }
    }
  }
}
```

Wanneer dit niet is ingesteld, begint Compaction met het actieve sessiemodel. Als het samenvatten mislukt door een providerfout waarbij modelterugval is toegestaan, probeert OpenClaw die Compaction-poging opnieuw via de bestaande modelterugvalketen van de sessie. De terugvalkeuze is tijdelijk en wordt niet teruggeschreven naar de sessiestatus. Een expliciete overschrijving met `agents.defaults.compaction.model` blijft exact en neemt de terugvalketen van de sessie niet over.

### Behoud van identificatoren

De samenvatting door Compaction behoudt standaard ondoorzichtige identificatoren (`identifierPolicy: "strict"`). Overschrijf dit met `identifierPolicy: "off"` om het uit te schakelen. Aangepaste richtlijnen horen thuis in de `summarize()`-implementatie van een Compaction-provider.

### Bewaking van het aantal bytes in het actieve transcript

Wanneer `agents.defaults.compaction.maxActiveTranscriptBytes` is ingesteld, activeert OpenClaw
vóór een uitvoering normale lokale Compaction als de transcriptgeschiedenis
die grootte bereikt. Dit is nuttig voor langlopende sessies waarbij contextbeheer
aan de kant van de provider de modelcontext gezond kan houden terwijl de opgeslagen
transcriptgeschiedenis blijft groeien. De onbewerkte bytes worden niet gesplitst;
de normale Compaction-pijplijn wordt gevraagd een semantische samenvatting te maken.

<Warning>
De bytebewaking is van toepassing op de actieve SQLite-transcriptgeschiedenis. Verouderde JSONL-
controlepuntartefacten zijn niet het actieve doel van Compaction.
</Warning>

### Opvolgende transcripties

Wanneer `agents.defaults.compaction.truncateAfterCompaction` is ingeschakeld, herschrijft OpenClaw het bestaande transcript niet ter plaatse. Het maakt een nieuw actief opvolgend transcript van de Compaction-samenvatting, de behouden status en het niet-samengevatte einde, en registreert vervolgens controlepuntmetagegevens die vertakkings- en herstelprocessen naar die gecomprimeerde opvolger verwijzen.
Opvolgende transcripties verwijderen ook exact dubbele lange gebruikersbeurten die
binnen een kort venster voor nieuwe pogingen binnenkomen, zodat stormen van nieuwe
kanaalpogingen na Compaction niet in het volgende actieve transcript terechtkomen.

OpenClaw schrijft voor nieuwe Compaction-bewerkingen niet langer afzonderlijke
`.checkpoint.*.jsonl`-kopieën. Bestaande verouderde controlepuntbestanden kunnen nog
worden gebruikt zolang ernaar wordt verwezen en worden door de normale
sessieopschoning verwijderd.

### Compaction-meldingen

Compaction wordt standaard stil uitgevoerd. Stel `notifyUser` in om korte statusberichten weer te geven wanneer Compaction begint en eindigt, en om een melding over verminderde werking te tonen wanneer een geheugenoverdracht vóór Compaction is uitgeput, maar het antwoord nog steeds doorgaat:

```json5
{
  agents: {
    defaults: {
      compaction: {
        notifyUser: true,
      },
    },
  },
}
```

### Geheugenoverdracht

Vóór Compaction kan OpenClaw een **stille geheugenoverdracht** uitvoeren om duurzame notities op schijf op te slaan. Stel `agents.defaults.compaction.memoryFlush.model` in wanneer deze onderhoudsbeurt een lokaal model moet gebruiken in plaats van het actieve gespreksmodel:

```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "memoryFlush": {
          "model": "ollama/qwen3:8b"
        }
      }
    }
  }
}
```

De overschrijving van het model voor geheugenoverdracht is exact en neemt de terugvalketen van de actieve sessie niet over. Zie [Geheugen](/nl/concepts/memory) voor details en configuratie.

## Inplugbare Compaction-providers

Plugins kunnen via `registerCompactionProvider()` in de Plugin-API een aangepaste Compaction-provider registreren. Wanneer een provider is geregistreerd en geconfigureerd, draagt OpenClaw het samenvatten aan deze provider over in plaats van aan de ingebouwde LLM-pijplijn.

Stel de ID van een geregistreerde provider in je configuratie in om deze te gebruiken:

```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "provider": "my-provider"
      }
    }
  }
}
```

Het instellen van een `provider` dwingt automatisch `mode: "safeguard"` af. Providers ontvangen dezelfde Compaction-instructies en hetzelfde beleid voor het behoud van identificatoren als het ingebouwde pad, en OpenClaw behoudt na de provideruitvoer nog steeds de context van recente beurten en achtervoegsels van gesplitste beurten.

<Note>
Als de provider mislukt of een leeg resultaat retourneert, valt OpenClaw terug op ingebouwde LLM-samenvatting.
</Note>

## Compaction versus opschoning

|                  | Compaction                              | Opschoning                              |
| ---------------- | --------------------------------------- | --------------------------------------- |
| **Wat het doet** | Vat oudere gesprekken samen             | Kort oude toolresultaten in             |
| **Opgeslagen?**  | Ja (in het sessietranscript)             | Nee (alleen in het geheugen, per verzoek) |
| **Bereik**       | Volledig gesprek                         | Alleen toolresultaten                    |

[Sessieopschoning](/nl/concepts/session-pruning) is een lichtere aanvulling die tooluitvoer inkort zonder deze samen te vatten.

## Probleemoplossing

**Wordt Compaction te vaak uitgevoerd?** Het contextvenster van het model is mogelijk klein, of tooluitvoer is mogelijk groot. Probeer [sessieopschoning](/nl/concepts/session-pruning) in te schakelen.

**Voelt de context na Compaction verouderd aan?** Gebruik `/compact Focus on <topic>` om de samenvatting te sturen, of schakel de [geheugenoverdracht](/nl/concepts/memory) in zodat notities behouden blijven.

**Een schone lei nodig?** `/new` start een nieuwe sessie zonder Compaction.

Zie [Uitgebreide informatie over sessiebeheer](/nl/reference/session-management-compaction) voor geavanceerde configuratie (gereserveerde tokens, behoud van identificatoren, aangepaste contextengines en Compaction aan de serverzijde van OpenAI).

## Gerelateerd

- [Sessie](/nl/concepts/session): sessiebeheer en levenscyclus.
- [Sessieopschoning](/nl/concepts/session-pruning): toolresultaten inkorten.
- [Context](/nl/concepts/context): hoe context voor agentbeurten wordt opgebouwd.
- [Hooks](/nl/automation/hooks): levenscyclushooks voor Compaction (`before_compaction`, `after_compaction`).
