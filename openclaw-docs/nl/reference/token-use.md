---
read_when:
    - Uitleg over tokengebruik, kosten of contextvensters
    - Contextgroei of Compaction-gedrag debuggen
summary: Hoe OpenClaw promptcontext opbouwt en tokengebruik en kosten rapporteert
title: Tokengebruik en kosten
x-i18n:
    generated_at: "2026-07-27T05:16:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6624bceb0bcbca769c9d569389b73b82f1ea73133e09f0ae9859833196d85911
    source_path: reference/token-use.md
    workflow: 16
---

OpenClaw houdt **tokens** bij, geen tekens. Tokens zijn modelspecifiek, maar de meeste
OpenAI-achtige modellen gebruiken gemiddeld ~4 tekens per token voor Engelse tekst.

## Hoe de systeemprompt wordt opgebouwd

OpenClaw stelt bij elke uitvoering zijn eigen systeemprompt samen. Deze bevat:

- Lijst met tools + korte beschrijvingen
- Lijst met Skills (alleen metadata; instructies worden op aanvraag geladen met `read`). Native
  Codex-beurten krijgen het compacte Skills-blok als samenwerkingsinstructies
  voor ontwikkelaars die alleen voor die beurt gelden; andere harnassen krijgen het in het normale promptoppervlak.
  Begrensd door `skills.limits.maxSkillsPromptChars`, met een optionele overschrijving per agent
  bij `agents.entries.*.skillsLimits.maxSkillsPromptChars`.
- Instructies voor zelfupdates
- Werkruimte + bootstrapbestanden (`AGENTS.md`, `SOUL.md`, `TOOLS.md`,
  `IDENTITY.md`, `USER.md`, `HEARTBEAT.md`, `BOOTSTRAP.md` wanneer nieuw, plus
  `MEMORY.md` indien aanwezig). Grote geïnjecteerde bestanden worden afgekapt door
  `agents.defaults.bootstrapMaxChars` (standaard: `20000`); de totale bootstrapinjectie
  wordt begrensd door `agents.defaults.bootstrapTotalMaxChars` (standaard:
  `60000`).
  - Native Codex-beurten plakken geen onbewerkte `MEMORY.md` wanneer geheugentools
    voor die werkruimte beschikbaar zijn; in plaats daarvan krijgen ze een kleine geheugenverwijzing in
    samenwerkingsinstructies voor ontwikkelaars die alleen voor die beurt gelden en gebruiken ze geheugentools
    op aanvraag. Als tools zijn uitgeschakeld, zoeken in het geheugen niet beschikbaar is of
    de actieve werkruimte verschilt van de geheugenwerkruimte van de agent, valt `MEMORY.md`
    terug op het normale begrensde pad voor beurtcontext.
  - De rootversie van `memory.md` in kleine letters wordt nooit geïnjecteerd. Het is verouderde reparatie-invoer
    voor `openclaw doctor --fix`, dat deze naar `MEMORY.md` migreert.
  - Dagbestanden van `memory/*.md` maken geen deel uit van de normale bootstrapprompt;
    ze blijven bij gewone beurten op aanvraag beschikbaar via geheugentools. Modeluitvoeringen bij resetten/opstarten
    kunnen voor die eerste beurt een eenmalig opstartcontextblok met recent
    dagelijks geheugen voorvoegen, aangestuurd door
    `agents.defaults.startupContext`. Kale chatberichten `/new` en `/reset` worden
    bevestigd zonder het model aan te roepen.
  - Fragmenten van `AGENTS.md` na Compaction vereisen expliciete
    aanmelding via `agents.defaults.compaction.postCompactionSections`; plugins kunnen
    andere context toevoegen via `before_prompt_build`.
- Tijd (UTC + tijdzone van de gebruiker)
- Antwoordtags + Heartbeat-gedrag
- Runtimemetadata (host/besturingssysteem/model/denken)

Bekijk de volledige uitsplitsing in [Systeemprompt](/nl/concepts/system-prompt).

Gebruik bij het documenteren van referenties of authenticatiefragmenten de
[Conventies voor geheime placeholders](/nl/reference/secret-placeholder-conventions) om
foutpositieven van geheime-scanners bij uitsluitend documentatiewijzigingen te voorkomen.

## Wat meetelt in het contextvenster

Alles wat het model ontvangt, telt mee voor de contextlimiet:

- Systeemprompt (alle bovenstaande secties)
- Gespreksgeschiedenis (berichten van gebruiker + assistent)
- Toolaanroepen en toolresultaten
- Bijlagen/transcripties (afbeeldingen, audio, bestanden)
- Compaction-samenvattingen en snoeiartefacten
- Providerwrappers of veiligheidsheaders (niet zichtbaar, maar tellen wel mee)

Runtime-intensieve oppervlakken hebben hun eigen expliciete limieten onder
`agents.defaults.contextLimits` (overschrijvingen per agent onder
`agents.entries.*.contextLimits`):

| Sleutel                  | Doel                                                                     |
| ------------------------ | ------------------------------------------------------------------------ |
| `memoryGetMaxChars`      | Maximaal aantal tekens dat `memory_get` retourneert vóór afkapping.     |
| `postCompactionMaxChars` | Maximaal aantal tekens dat uit `AGENTS.md` behouden blijft tijdens vernieuwing na Compaction. |

Dit zijn begrensde runtimefragmenten en geïnjecteerde blokken die door de runtime worden beheerd,
los van bootstraplimieten, opstartcontextlimieten en limieten voor Skills-prompts.

OpenClaw leidt de actuele limiet voor toolresultaten af uit het effectieve contextvenster
van het model: `16000` tekens onder
100K tokens, `32000` tekens bij 100K+ tokens, `64000` tekens bij 200K+ tokens.
De bewaking van het runtimecontextaandeel begrenst één toolresultaat ook tot 30% van het
contextvenster.

Grote providervensters worden niet automatisch ingeschakeld wanneer ze de
kosten of latentie wezenlijk wijzigen. Rechtstreekse OpenAI GPT-5.5- en GPT-5.6-modellen
publiceren bijvoorbeeld een totaal venster van `1050000` tokens, maar OpenClaw stelt hun actieve
runtimebudget standaard in op `272000` tokens. Het aanmeldbare invoerbudget van `922000` reserveert de
volledige uitvoerruimte van `128000`, en OpenAI past hogere prijzen voor lange context toe
op het volledige verzoek zodra de invoer meer dan `272000` tokens bedraagt. Zie
[Standaardwaarden voor OpenAI-contextvensters](/nl/providers/openai#context-window-defaults-and-long-context-opt-in).

Voor afbeeldingen schaalt OpenClaw afbeeldingspayloads van transcripties/tools vóór
provideraanroepen omlaag. Pas dit aan met `agents.defaults.imageMaxDimensionPx` (standaard:
`1200`):

- Lagere waarden verminderen het gebruik van visietokens en de payloadgrootte.
- Hogere waarden behouden meer visuele details voor screenshots met veel OCR/UI.

Gebruik voor een praktische uitsplitsing (per geïnjecteerd bestand, tools, Skills en grootte van de
systeemprompt) `/context list` of `/context detail`. Zie
[Context](/nl/concepts/context).

## Het huidige tokengebruik bekijken

In de chat:

- `/status` -> statuskaart met veel emoji met het sessiemodel, contextgebruik,
  invoer-/uitvoertokens van het laatste antwoord en geschatte kosten wanneer lokale prijzen zijn
  geconfigureerd voor het actieve model.
- `/usage off|tokens|full` -> voegt aan elk antwoord een gebruiksvoettekst per antwoord toe.
  Blijft per sessie behouden (opgeslagen als `responseUsage`).
  - `/usage reset` (aliassen: `inherit`, `clear`, `default`) wist de
    sessieoverschrijving, zodat de geconfigureerde standaard opnieuw wordt overgenomen.
  - `/usage tokens` toont details over tokens/cache van de beurt.
  - `/usage full` toont beknopte details over model/context/kosten; geschatte kosten
    verschijnen alleen wanneer OpenClaw gebruiksmetadata en lokale prijzen voor het
    actieve model heeft. Aangepaste indelingen van `messages.usageTemplate` kunnen
    token-/cachevelden bevatten.
- `/usage cost` -> lokaal kostenoverzicht uit OpenClaw-sessielogboeken.

Andere oppervlakken:

- **TUI/Web-TUI:** `/status` en `/usage` worden ondersteund.
- **CLI:** `openclaw status --usage` en `openclaw channels list` tonen
  genormaliseerde quotavensters van providers (`X% left`, geen kosten per antwoord).
  Huidige providers van gebruiksvensters: Claude (Anthropic), ClawRouter, Copilot
  (GitHub), DeepSeek, Gemini (Google Gemini CLI), MiniMax, OpenAI, Xiaomi,
  Xiaomi Token Plan en z.ai.

Gebruiksoppervlakken normaliseren algemene providerspecifieke veldaliassen vóór
weergave. Voor Responses-verkeer van de OpenAI-familie omvat dit zowel
`input_tokens`/`output_tokens` als `prompt_tokens`/`completion_tokens`, zodat
transportspecifieke veldnamen `/status`, `/usage` of sessieoverzichten
niet wijzigen. Gebruik van Gemini CLI wordt ook genormaliseerd: de standaardparser van `stream-json`
leest assistentgebeurtenissen van `message`, en `stats.cached` wordt toegewezen aan
`cacheRead`, waarbij `stats.input_tokens - stats.cached` wordt gebruikt wanneer de CLI
een expliciet veld `stats.input` weglaat. Verouderde JSON-overschrijvingen lezen antwoordtekst nog steeds
uit `response`.

Voor native Responses-verkeer van de OpenAI-familie worden WebSocket-/SSE-gebruiksaliassen
op dezelfde manier genormaliseerd, en totalen vallen terug op genormaliseerde invoer + uitvoer
wanneer `total_tokens` ontbreekt of `0` is.

Wanneer de huidige sessiemomentopname weinig gegevens bevat, kunnen `/status` en `session_status`
token-/cachetellers en het label van het actieve runtimemodel herstellen uit het
meest recente gebruikslogboek van de transcriptie. Bestaande niet-nul actuele waarden hebben nog steeds
voorrang op terugvalwaarden uit de transcriptie, en grotere promptgerichte
transcriptietotalen kunnen winnen wanneer opgeslagen totalen ontbreken of kleiner zijn.

Gebruiksauthenticatie voor quotavensters van providers komt eerst uit providerspecifieke hooks;
als een provider geen hook heeft (of de hook geen token oplevert),
valt OpenClaw terug op overeenkomende OAuth-/API-sleutelreferenties uit authenticatieprofielen,
de omgeving of configuratie.

Assistentvermeldingen in transcripties bewaren dezelfde genormaliseerde gebruiksvorm,
inclusief `usage.cost` wanneer voor het actieve model prijzen zijn geconfigureerd en de
provider gebruiksmetadata retourneert. Dit geeft `/usage cost` en
op transcripties gebaseerde sessiestatus een stabiele bron, zelfs nadat de actuele
runtimestatus verdwenen is.

OpenClaw houdt de gebruiksboekhouding van providers gescheiden van de huidige contextmomentopname.
Provider-`usage.total` kan gecachete invoer, uitvoer en
meerdere modelaanroepen in toollussen bevatten, waardoor het nuttig is voor kosten en telemetrie, maar
het actuele contextvenster te hoog kan worden weergegeven. Contextweergaven en diagnostiek gebruiken
de nieuwste promptmomentopname (`promptTokens`, of de laatste modelaanroep wanneer geen
promptmomentopname beschikbaar is) voor `context.used`.

## Kostenschatting (indien weergegeven)

Kosten worden geschat op basis van je configuratie voor modelprijzen:

```text
models.providers.<provider>.models[].cost
```

Dit zijn **USD per 1M tokens** voor `input`, `output`, `cacheRead` en
`cacheWrite`. Als prijzen ontbreken, laat `/usage full` de kosten weg; gebruik
`/usage tokens` of een aangepaste `messages.usageTemplate` wanneer je
in elk antwoord details over tokens/cache nodig hebt. Kostenweergave is niet beperkt tot authenticatie
met een API-sleutel: providers zonder API-sleutel, zoals `aws-sdk`, kunnen geschatte kosten weergeven wanneer
hun geconfigureerde modelvermelding lokale prijzen bevat en de provider
gebruiksmetadata retourneert.

Nadat sidecars en kanalen het gereedheidspad van de Gateway hebben bereikt, start OpenClaw een
optionele prijsbootstrap op de achtergrond voor geconfigureerde modelreferenties die nog geen
lokale prijzen hebben. Die bootstrap haalt externe prijscatalogi van OpenRouter en
LiteLLM op. Stel `models.pricing.enabled: false` in om het ophalen van die
catalogi op offline of beperkte netwerken over te slaan; expliciete
`models.providers.*.models[].cost`-vermeldingen blijven lokale kostenschattingen aansturen.

## Invloed van cache-TTL en snoeien

Promptcaching van providers is alleen van toepassing binnen het cache-TTL-venster. OpenClaw
kan optioneel **cache-TTL-snoeien** uitvoeren: het snoeit de sessie zodra de
cache-TTL is verlopen en stelt vervolgens het cachevenster opnieuw in, zodat volgende verzoeken
de opnieuw gecachete context hergebruiken in plaats van de volledige geschiedenis opnieuw te cachen.
Dit houdt de schrijfkosten van de cache lager wanneer een sessie langer dan de TTL inactief blijft.

Configureer dit in [Gateway-configuratie](/nl/gateway/configuration) en bekijk de
gedragsdetails in [Sessiesnoeiing](/nl/concepts/session-pruning).

Heartbeat kan de cache tijdens perioden van inactiviteit **warm** houden. Als de cache-TTL
van je model `1h` is, kan het instellen van het Heartbeat-interval net daaronder (bijv. `55m`)
voorkomen dat de volledige prompt opnieuw wordt gecachet, waardoor de schrijfkosten van de cache afnemen.

In configuraties met meerdere agents kun je één gedeelde modelconfiguratie behouden en het cachegedrag
per agent afstemmen met `agents.entries.*.params.cacheRetention`.

Bekijk [Promptcaching](/nl/reference/prompt-caching) voor een volledige handleiding per instelling.

Voor prijzen van de Anthropic API zijn cachelezingen aanzienlijk goedkoper dan invoertokens,
terwijl cacheschrijfacties tegen een hogere vermenigvuldigingsfactor worden gefactureerd. Bekijk de prijzen van Anthropic
voor promptcaching voor de nieuwste tarieven en TTL-vermenigvuldigingsfactoren:
[https://docs.anthropic.com/docs/build-with-claude/prompt-caching](https://docs.anthropic.com/docs/build-with-claude/prompt-caching)

### Voorbeeld: houd een cache van 1h warm met Heartbeat

```yaml
agents:
  defaults:
    model:
      primary: "anthropic/claude-opus-4-6"
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "long"
    heartbeat:
      every: "55m"
```

### Voorbeeld: gemengd verkeer met een cachestrategie per agent

```yaml
agents:
  defaults:
    model:
      primary: "anthropic/claude-opus-4-6"
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "long" # standaardbasis voor de meeste agents
  list:
    - id: "research"
      default: true
      heartbeat:
        every: "55m" # houd de langetermijncache warm voor diepgaande sessies
    - id: "alerts"
      params:
        cacheRetention: "none" # voorkom cacheschrijfbewerkingen voor piekgewijze meldingen
```

`agents.entries.*.params` wordt boven op de `params` van het geselecteerde model samengevoegd, zodat je
alleen `cacheRetention` kunt overschrijven en de overige modelstandaarden
ongewijzigd overneemt.

### Anthropic-context van 1M

OpenClaw schaalt Claude 4.x-modellen die algemeen beschikbaar zijn, zoals Opus 4.8, Opus 4.7, Opus
4.6 en Sonnet 4.6, met het contextvenster van 1M van Anthropic. Voor deze modellen heb je
`params.context1m: true` niet nodig.

```yaml
agents:
  defaults:
    models:
      "anthropic/claude-opus-4-6":
        alias: opus
```

Oudere configuraties kunnen `context1m: true` behouden, maar OpenClaw verstuurt
de uitgefaseerde bètaheader `context-1m-2025-08-07` van Anthropic voor deze instelling niet meer en
breidt niet-ondersteunde oudere Claude-modellen niet uit naar 1M.

Vereiste: de referentie moet in aanmerking komen voor gebruik met een lange context. Zo niet,
dan reageert Anthropic voor die aanvraag met een snelheidslimietfout aan de kant van de provider.

Als je je bij Anthropic verifieert met OAuth-/abonnementstokens
(`sk-ant-oat-*`), behoudt OpenClaw de voor OAuth vereiste Anthropic-bètaheaders,
terwijl de uitgefaseerde bèta `context-1m-*` wordt verwijderd als deze nog in een
oudere configuratie staat.

## Tips om de tokendruk te verminderen

- Gebruik `/compact` om lange sessies samen te vatten.
- Kort grote tooluitvoer in je workflows in.
- Verlaag `agents.defaults.imageMaxDimensionPx` voor sessies met veel schermafbeeldingen.
- Houd beschrijvingen van Skills kort (de lijst met Skills wordt in de prompt geïnjecteerd).
- Geef voor uitgebreid, verkennend werk de voorkeur aan kleinere modellen.

Zie [Skills](/nl/tools/skills) voor de exacte formule voor de overhead van de lijst met Skills.

## Gerelateerd

- [API-gebruik en -kosten](/nl/reference/api-usage-costs)
- [Promptcaching](/nl/reference/prompt-caching)
- [Gebruik bijhouden](/nl/concepts/usage-tracking)
