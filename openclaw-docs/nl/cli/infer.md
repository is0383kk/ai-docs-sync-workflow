---
read_when:
    - '`openclaw infer`-opdrachten toevoegen of wijzigen'
    - Stabiele headless-capability-automatisering ontwerpen
summary: Infer-first-CLI voor providergestuurde workflows voor modellen, afbeeldingen, audio, TTS, video, web en embeddings
title: Inferentie-CLI
x-i18n:
    generated_at: "2026-07-27T04:53:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3147bb516a08e12c4eacd6bd527af62049ecae25b5fde9439da6a4431c147b07
    source_path: cli/infer.md
    workflow: 16
---

`openclaw infer` is de canonieke headless-interface voor door providers ondersteunde inferentie. Deze biedt capabilityfamilies (`model`, `image`, `audio`, `tts`, `video`, `web`, `embedding`), geen onbewerkte RPC-namen van de Gateway of tool-id's van agents. `openclaw capability ...` is een alias voor dezelfde commandostructuur.

Redenen om dit te verkiezen boven een eenmalige providerwrapper:

- Hergebruikt providers en modellen die al in OpenClaw zijn geconfigureerd.
- Stabiele `--json`-envelop voor scripts en door agents aangestuurde automatisering (zie [JSON-uitvoer](#json-output)).
- Voert voor de meeste subcommando's het normale lokale pad uit zonder de Gateway.
- Voor end-to-end-providercontroles doorloopt het de meegeleverde CLI, het laden van configuratie, het bepalen van de standaardagent, het activeren van gebundelde plugins en de gedeelde capabilityruntime voordat het providerverzoek wordt verzonden.

## Maak van infer een skill

Kopieer en plak dit naar een agent:

```text
Lees https://docs.openclaw.ai/cli/infer en maak vervolgens een skill die mijn gebruikelijke workflows naar `openclaw infer` routeert.
Richt je op modeluitvoeringen, beeldgeneratie, videogeneratie, audiotranscriptie, TTS, zoeken op het web en embeddings.
```

Een goede op infer gebaseerde skill koppelt veelvoorkomende gebruikersintenties aan het juiste subcommando, bevat enkele canonieke voorbeelden per workflow, verkiest `openclaw infer ...` boven alternatieven op lager niveau en documenteert niet het volledige infer-oppervlak opnieuw in de hoofdtekst van de skill.

## Commandostructuur

```text
 openclaw infer
  list
  inspect

  model
    run
    list
    inspect
    providers
    auth login
    auth logout
    auth status

  image
    generate
    edit
    describe
    describe-many
    providers

  audio
    transcribe
    providers

  tts
    convert
    voices
    providers
    personas
    status
    enable
    disable
    set-provider
    set-persona

  video
    generate
    describe
    providers

  web
    search
    fetch
    providers

  embedding
    create
    providers
```

`infer list` / `infer inspect --name <capability>` tonen deze structuur als gegevens (capability-id, transporten, beschrijving).

## Veelvoorkomende taken

| Taak                          | Commando                                                                                       | Opmerkingen                                                 |
| ----------------------------- | --------------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| Een tekst-/modelprompt uitvoeren       | `openclaw infer model run --prompt "..." --json`                                              | Standaard lokaal                                      |
| Een modelprompt op afbeeldingen uitvoeren  | `openclaw infer model run --prompt "Describe this" --file ./image.png --model provider/model` | Herhaal `--file` voor meerdere afbeeldingen                   |
| Een afbeelding genereren             | `openclaw infer image generate --prompt "..." --json`                                         | Gebruik `image edit` wanneer je vanuit een bestaand bestand begint  |
| Een afbeeldingsbestand of URL beschrijven | `openclaw infer image describe --file ./image.png --prompt "..." --json`                      | `--model` moet een voor afbeeldingen geschikt `<provider/model>` zijn |
| Audio transcriberen              | `openclaw infer audio transcribe --file ./memo.m4a --json`                                    | `--model` moet `<provider/model>` zijn                  |
| Spraak synthetiseren             | `openclaw infer tts convert --text "..." --output ./speech.mp3 --json`                        | `tts status` wordt alleen via de Gateway uitgevoerd            |
| Een video genereren              | `openclaw infer video generate --prompt "..." --json`                                         | Ondersteunt providerhints zoals `--resolution`        |
| Een videobestand beschrijven         | `openclaw infer video describe --file ./clip.mp4 --json`                                      | `--model` moet `<provider/model>` zijn                  |
| Zoeken op het web                | `openclaw infer web search --query "..." --json`                                              |                                                       |
| Een webpagina ophalen              | `openclaw infer web fetch --url https://example.com --json`                                   |                                                       |
| Embeddings maken             | `openclaw infer embedding create --text "..." --json`                                         |                                                       |

## Gedrag

- Gebruik `--json` wanneer de uitvoer als invoer voor een ander commando of script dient; gebruik anders tekstuitvoer.
- Gebruik `--provider` of `--model provider/model` om een specifieke backend vast te leggen.
- Gebruik `model run --thinking <level>` voor een eenmalige override voor denken/redeneren: `off`, `minimal`, `low`, `medium`, `high`, `adaptive`, `xhigh` of `max`.
- Voor `image describe`, `audio transcribe` en `video describe` moet `--model` de vorm `<provider/model>` gebruiken.
- Voor `image describe` accepteert `--file` lokale paden en HTTP(S)-URL's; externe URL's doorlopen het normale SSRF-beleid voor het ophalen van media.
- Toestandsloze uitvoeringscommando's (`model run`, `image *`, `audio *`, `video *`, `web *`, `embedding *`) zijn standaard lokaal. Door de Gateway beheerde toestandscommando's (`tts status`) gebruiken standaard de Gateway.
- Voor het lokale pad hoeft de Gateway nooit actief te zijn.
- Lokale `model run` is een gestroomlijnde, eenmalige providervoltooiing: deze bepaalt het geconfigureerde agentmodel en de authenticatie, maar start geen chatagentbeurt, laadt geen tools en opent geen gebundelde MCP-servers.
- `model run --file` voegt afbeeldingsbestanden (met automatisch gedetecteerd MIME-type) aan de prompt toe; herhaal `--file` voor meerdere afbeeldingen. Niet-afbeeldingsbestanden worden geweigerd — gebruik in plaats daarvan `infer audio transcribe` of `infer video describe`.
- `model run --gateway` doorloopt Gateway-routering, opgeslagen authenticatie, providerselectie en de ingebedde runtime, maar blijft een onbewerkte modelprobe: geen eerder sessietranscript, bootstrap-/AGENTS-context, tools of gebundelde MCP-servers.
- `model run --gateway --model <provider/model>` vereist een Gateway-referentie van een vertrouwde operator, omdat het de Gateway vraagt een eenmalige provider-/modeloverride uit te voeren.

## Model

Tekstinferentie en inspectie van modellen/providers.

```bash
openclaw infer model run --prompt "Antwoord exact met: smoke-ok" --json
openclaw infer model run --prompt "Vat dit changelog-item samen" --model openai/gpt-5.4 --json
openclaw infer model run --prompt "Beschrijf deze afbeelding in één zin" --file ./photo.jpg --model google/gemini-2.5-flash --json
openclaw infer model run --prompt "Gebruik hier meer redeneervermogen" --thinking high --json
openclaw infer model providers --json
openclaw infer model inspect --model gpt-5.6-sol --json
```

Gebruik volledige `<provider/model>`-referenties met `--local` om één provider met een rooktest te controleren zonder de Gateway te starten of het tooloppervlak van de agent te laden:

```bash
openclaw infer model run --local --model anthropic/claude-sonnet-4-6 --prompt "Antwoord exact met: pong" --json
openclaw infer model run --local --model cerebras/zai-glm-4.7 --prompt "Antwoord exact met: pong" --json
openclaw infer model run --local --model google/gemini-2.5-flash --prompt "Antwoord exact met: pong" --json
openclaw infer model run --local --model groq/llama-3.1-8b-instant --prompt "Antwoord exact met: pong" --json
openclaw infer model run --local --model mistral/mistral-medium-3-5 --prompt "Antwoord exact met: pong" --json
openclaw infer model run --local --model mistral/mistral-small-latest --prompt "Antwoord exact met: pong" --json
openclaw infer model run --local --model openai/gpt-5.6-luna --prompt "Antwoord exact met: pong" --json
openclaw infer model run --local --model ollama/qwen2.5vl:7b --prompt "Beschrijf deze afbeelding." --file ./photo.jpg --json
```

Opmerkingen:

- Lokale `model run` is de meest gerichte CLI-rooktest voor de status van provider/model/authenticatie: voor providers anders dan ChatGPT-Codex wordt alleen de opgegeven prompt verzonden.
- Lokale `model run --model <provider/model>` kan exacte gebundelde rijen uit de statische catalogus bepalen (dezelfde rijen die `openclaw models list --all` toont) voordat die provider naar de configuratie is geschreven. Providerauthenticatie blijft vereist; ontbrekende referenties veroorzaken authenticatiefouten, niet `Unknown model`.
- Laat voor redeneerprobes met Mistral Medium 3.5 de temperatuur oningesteld/op de standaardwaarde. Mistral weigert `reasoning_effort="high"` met `temperature: 0`; gebruik de standaardtemperatuur of een waarde die niet nul is, zoals `0.7`.
- Lokale probes met OpenAI ChatGPT/Codex OAuth (`openai-chatgpt-responses`-API) voegen een minimale systeeminstructie toe, zodat het transport het vereiste veld `instructions` kan invullen — zonder volledige agentcontext, tools, geheugen of sessietranscript.
- `model run --file` voegt afbeeldingsinhoud rechtstreeks aan het afzonderlijke gebruikersbericht toe. Gangbare indelingen (PNG, JPEG, WebP) werken wanneer het MIME-type als `image/*` wordt gedetecteerd; niet-ondersteunde of niet-herkende bestanden mislukken voordat de provider wordt aangeroepen. Gebruik in plaats daarvan `infer image describe` wanneer je OpenClaws routering en fallbacks voor afbeeldingsmodellen wilt gebruiken in plaats van een rechtstreekse multimodale modelprobe.
- Het geselecteerde model moet afbeeldingsinvoer ondersteunen; modellen die alleen tekst ondersteunen, kunnen het verzoek op providerniveau weigeren.
- `model run --prompt` moet tekst bevatten die niet alleen uit witruimte bestaat; lege prompts worden geweigerd voordat een provider- of Gateway-aanroep plaatsvindt.
- Lokale `model run` wordt afgesloten met een andere waarde dan nul wanneer de provider geen tekstuitvoer retourneert, zodat onbereikbare providers en lege voltooiingen niet op geslaagde probes lijken.
- Gebruik `model run --gateway` om Gateway-routering of de configuratie van de agentruntime te testen terwijl de modelinvoer onbewerkt blijft. Gebruik `openclaw agent` of een chatinterface voor volledige agentcontext, tools, geheugen en sessietranscript.
- `--thinking adaptive` wordt gekoppeld aan `medium` op het niveau van de voltooiingsruntime; `--thinking max` wordt gekoppeld aan `max` voor OpenAI-modellen die de ingebouwde maximale inspanning ondersteunen, en anders aan `xhigh`.
- `model auth login`, `model auth logout` en `model auth status` beheren de opgeslagen authenticatiestatus van providers.

## Afbeelding

Generatie, bewerking en beschrijving.

```bash
openclaw infer image generate --prompt "vriendelijke illustratie van een kreeft" --json
openclaw infer image generate --prompt "filmische productfoto van een koptelefoon" --json
openclaw infer image generate --model openai/gpt-image-1.5 --output-format png --background transparent --prompt "eenvoudige sticker met een rode cirkel op een transparante achtergrond" --json
openclaw infer image generate --model openai/gpt-image-2 --quality low --openai-moderation low --prompt "goedkoop concept voor een poster" --json
openclaw infer image generate --prompt "trage backend voor afbeeldingen" --timeout-ms 180000 --json
openclaw infer image edit --file ./logo.png --model openai/gpt-image-1.5 --output-format png --background transparent --prompt "behoud het logo en verwijder de achtergrond" --json
openclaw infer image edit --file ./poster.png --prompt "maak hiervan een verticale storyadvertentie" --size 2160x3840 --aspect-ratio 9:16 --resolution 4K --json
openclaw infer image describe --file ./photo.jpg --json
openclaw infer image describe --file https://example.com/photo.png --json
openclaw infer image describe --file ./receipt.jpg --prompt "Extraheer de verkoper, datum en het totaalbedrag" --json
openclaw infer image describe-many --file ./before.png --file ./after.png --prompt "Vergelijk de schermafbeeldingen en vermeld de zichtbare wijzigingen in de gebruikersinterface" --json
openclaw infer image describe --file ./ui-screenshot.png --model openai/gpt-5.4-mini --json
openclaw infer image describe --file ./photo.jpg --model ollama/qwen2.5vl:7b --prompt "Beschrijf de afbeelding in één zin" --timeout-ms 300000 --json
```

Opmerkingen:

- Gebruik `image edit` wanneer je begint met bestaande invoerbestanden; `--size`, `--aspect-ratio` of `--resolution` voegen geometrieaanwijzingen toe voor providers/modellen die deze ondersteunen.
- `--output-format png --background transparent` met `--model openai/gpt-image-1.5` levert OpenAI PNG-uitvoer met een transparante achtergrond; `--openai-background` is een OpenAI-specifieke alias voor dezelfde aanwijzing. Providers die geen ondersteuning voor achtergronden declareren, rapporteren dit als een genegeerde overschrijving (zie `ignoredOverrides` in de [JSON-envelop](#json-output)).
- `--quality low|medium|high|auto` werkt voor providers die aanwijzingen voor beeldkwaliteit ondersteunen, waaronder OpenAI. OpenAI accepteert ook `--openai-moderation low|auto`.
- `image providers --json` vermeldt welke gebundelde afbeeldingsproviders detecteerbaar, geconfigureerd en geselecteerd zijn, en welke mogelijkheden voor genereren/bewerken elke provider biedt.
- `image generate --model <provider/model> --json` is de meest gerichte live-rooktest voor wijzigingen in het genereren van afbeeldingen:

  ```bash
  openclaw infer image providers --json
  openclaw infer image generate \
    --model google/gemini-3.1-flash-image \
    --prompt "Minimale vlakke testafbeelding: één blauw vierkant op een witte achtergrond, zonder tekst." \
    --output ./openclaw-infer-image-smoke.png \
    --json
  ```

  Het antwoord rapporteert `ok`, `provider`, `model`, `attempts` en de paden van de geschreven uitvoer. Wanneer `--output` is ingesteld, kan de uiteindelijke extensie het door de provider geretourneerde MIME-type volgen.

- Gebruik voor `image describe` en `image describe-many` `--prompt` voor een taakspecifieke instructie (OCR, vergelijking, UI-inspectie, beknopte ondertiteling).
- Gebruik `--timeout-ms` voor trage lokale visiemodellen of koude starts van Ollama.
- Voor `image describe` wordt eerst een expliciete `--model` uitgevoerd (dit moet een voor afbeeldingen geschikt `<provider/model>` zijn), waarna geconfigureerde `agents.defaults.imageModel.fallbacks` worden geprobeerd als die aanroep mislukt. Fouten bij het voorbereiden van de invoer (ontbrekend bestand, niet-ondersteunde URL) leiden tot een fout voordat een terugvalpoging plaatsvindt, en het model moet in de modelcatalogus of providerconfiguratie als geschikt voor afbeeldingen zijn aangemerkt.
- Haal voor lokale Ollama-visiemodellen eerst het model op en stel `OLLAMA_API_KEY` in op een willekeurige tijdelijke waarde, bijvoorbeeld `ollama-local`. Zie [Ollama](/nl/providers/ollama#vision-and-image-description).

## Audio

Bestandstranscriptie (geen realtime sessiebeheer).

```bash
openclaw infer audio transcribe --file ./memo.m4a --json
openclaw infer audio transcribe --file ./team-sync.m4a --language en --prompt "Concentreer je op namen en actiepunten" --json
openclaw infer audio transcribe --file ./memo.m4a --model openai/whisper-1 --json
```

`--model` moet `<provider/model>` zijn.

## TTS

Spraaksynthese en de status van TTS-providers/persona's.

```bash
openclaw infer tts convert --text "hallo van openclaw" --output ./hello.mp3 --json
openclaw infer tts convert --text "Je build is voltooid" --output ./build-complete.mp3 --json
openclaw infer tts providers --json
openclaw infer tts personas --json
openclaw infer tts status --json
```

Opmerkingen:

- `tts status` ondersteunt alleen `--gateway` (dit weerspiegelt de door de Gateway beheerde TTS-status).
- Gebruik `tts providers`, `tts voices`, `tts personas`, `tts set-provider` en `tts set-persona` om TTS-gedrag te inspecteren en configureren.

## Video

Genereren en beschrijven.

```bash
openclaw infer video generate --prompt "filmische zonsondergang boven de oceaan" --json
openclaw infer video generate --prompt "langzame droneopname boven een bosmeer" --resolution 768P --duration 6 --json
openclaw infer video describe --file ./clip.mp4 --json
openclaw infer video describe --file ./clip.mp4 --model openai/gpt-5.4-mini --json
```

Opmerkingen:

- `video generate` accepteert `--size`, `--aspect-ratio`, `--resolution`, `--duration`, `--audio`, `--watermark` en `--timeout-ms`, die worden doorgestuurd naar de runtime voor het genereren van video's.
- `--model` moet `<provider/model>` zijn voor `video describe`.

## Web

Zoeken en ophalen.

```bash
openclaw infer web search --query "OpenClaw-documentatie" --json
openclaw infer web search --query "OpenClaw infer-webproviders" --json
openclaw infer web fetch --url https://docs.openclaw.ai/cli/infer --json
openclaw infer web providers --json
```

`web providers` vermeldt beschikbare, geconfigureerde en geselecteerde providers voor zoeken en ophalen.

## Insluiting

Vectorcreatie en inspectie van insluitingsproviders.

```bash
openclaw infer embedding create --text "vriendelijke kreeft" --json
openclaw infer embedding create --text "klantenserviceticket: vertraagde verzending" --model openai/text-embedding-3-large --json
openclaw infer embedding providers --json
```

## JSON-uitvoer

Infer-opdrachten normaliseren JSON-uitvoer onder een gedeelde envelop:

```json
{
  "ok": true,
  "capability": "image.generate",
  "transport": "local",
  "provider": "openai",
  "model": "gpt-image-2",
  "attempts": [],
  "outputs": []
}
```

Stabiele velden op het hoogste niveau:

- `ok`
- `capability`
- `transport`
- `provider`
- `model`
- `attempts`
- `inputs` (afbeeldingsbijlagen die met het verzoek zijn verzonden, indien van toepassing)
- `outputs`
- `ignoredOverrides` (aanwijzingssleutels die een provider niet ondersteunt, indien van toepassing)
- `error`

Voor opdrachten voor gegenereerde media bevat `outputs` bestanden die door OpenClaw zijn geschreven. Gebruik voor automatisering de `path`, `mimeType`, `size` en eventuele mediaspecifieke afmetingen in die array, in plaats van door mensen leesbare stdout te parseren.

## Veelvoorkomende valkuilen

```bash
# Fout
openclaw infer media image generate --prompt "vriendelijke kreeft"

# Goed
openclaw infer image generate --prompt "vriendelijke kreeft"
```

```bash
# Fout
openclaw infer audio transcribe --file ./memo.m4a --model whisper-1 --json

# Goed
openclaw infer audio transcribe --file ./memo.m4a --model openai/whisper-1 --json
```

## Gerelateerd

- [CLI-referentie](/nl/cli)
- [Modellen](/nl/concepts/models)
