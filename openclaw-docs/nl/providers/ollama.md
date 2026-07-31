---
read_when:
    - Je wilt OpenClaw uitvoeren met cloudmodellen of lokale modellen via Ollama
    - Je hebt hulp nodig bij het instellen en configureren van Ollama
    - Je wilt Ollama-visionmodellen voor beeldbegrip
summary: Voer OpenClaw uit met Ollama (cloud- en lokale modellen)
title: Ollama
x-i18n:
    generated_at: "2026-07-27T05:47:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 80ae833d006ce307406fac11fe3457809165035a38b7e0a970777baf126cc9cb
    source_path: providers/ollama.md
    workflow: 16
---

OpenClaw communiceert met de native API van Ollama (`/api/chat`), niet met het OpenAI-compatibele
`/v1`-eindpunt. Er worden drie modi ondersteund:

| Modus         | Wat deze gebruikt                                                                    |
| ------------- | ------------------------------------------------------------------------------------ |
| Cloud + lokaal | Een bereikbare Ollama-host die lokale modellen en (indien aangemeld) `:cloud`-modellen aanbiedt |
| Alleen cloud  | Rechtstreeks `https://ollama.com`, zonder lokale daemon                                |
| Alleen lokaal | Een bereikbare Ollama-host, uitsluitend lokale modellen                              |

Zie voor een configuratie met alleen de cloud en de specifieke provider-id `ollama-cloud`
[Ollama Cloud](/nl/providers/ollama-cloud). Gebruik `ollama-cloud/<model>`-verwijzingen wanneer
je cloudroutering gescheiden wilt houden van een lokale `ollama`-provider.

<Warning>
Gebruik niet de OpenAI-compatibele URL `/v1` (`http://host:11434/v1`). Hierdoor werken toolaanroepen niet goed en kunnen modellen onbewerkte JSON voor toolaanroepen als platte tekst uitvoeren. Gebruik de native URL: `baseUrl: "http://host:11434"` (zonder `/v1`).
</Warning>

De canonieke configuratiesleutel is `baseUrl`. `baseURL` wordt ook geaccepteerd voor
voorbeelden in OpenAI-SDK-stijl, maar nieuwe configuraties moeten `baseUrl` gebruiken.

## Authenticatieregels

<AccordionGroup>
  <Accordion title="Lokale en LAN-hosts">
    Loopback-, privénetwerk-, `.local`- en Ollama-URL's met alleen een hostnaam hebben geen echt bearer-token nodig. OpenClaw gebruikt hiervoor de markering `ollama-local`.
  </Accordion>
  <Accordion title="Externe hosts en Ollama Cloud-hosts">
    Openbare externe hosts en `https://ollama.com` vereisen echte aanmeldgegevens: `OLLAMA_API_KEY`, een authenticatieprofiel of de `apiKey` van de provider. Geef voor rechtstreeks gehost gebruik de voorkeur aan de provider `ollama-cloud`.
  </Accordion>
  <Accordion title="Aangepaste provider-id's">
    Voor een aangepaste provider met `api: "ollama"` gelden dezelfde regels. Een `ollama-remote`-provider die bijvoorbeeld naar een privéhost op het LAN verwijst, kan `apiKey: "ollama-local"` gebruiken; subagents verwerken die markering via de providerhook van Ollama in plaats van deze als ontbrekende aanmeldgegevens te behandelen. `memory.search.provider` kan ook naar een aangepaste provider-id verwijzen, zodat embeddings dat Ollama-eindpunt gebruiken.
  </Accordion>
  <Accordion title="Authenticatieprofielen">
    `auth-profiles.json` bewaart de aanmeldgegevens voor een provider-id; plaats eindpuntinstellingen (`baseUrl`, `api`, modellen, headers en time-outs) in `models.providers.<id>`. Oudere platte bestanden zoals `{ "ollama-windows": { "apiKey": "ollama-local" } }` zijn geen runtime-indeling; `openclaw doctor --fix` herschrijft ze met een back-up naar een canoniek API-sleutelprofiel van `ollama-windows:default`. Een waarde voor `baseUrl` in dat verouderde bestand is ruis en moet naar de providerconfiguratie worden verplaatst.
  </Accordion>
  <Accordion title="Bereik van authenticatie voor geheugenembeddings">
    Bearer-authenticatie voor Ollama-geheugenembeddings is beperkt tot de host waarvoor deze is opgegeven:

    - Een sleutel op providerniveau wordt alleen naar de host van die provider verzonden.
    - `memory.search.remote.apiKey` en overrides per agent worden alleen naar hun externe embeddinghost verzonden.
    - Een zuivere `OLLAMA_API_KEY`-omgevingsvariabele wordt behandeld als de conventie van Ollama Cloud en wordt standaard niet naar lokale/zelfgehoste hosts verzonden.

  </Accordion>
</AccordionGroup>

## Aan de slag

<Tabs>
  <Tab title="Onboarding (aanbevolen)">
    <Steps>
      <Step title="Onboarding uitvoeren">
        ```bash
        openclaw onboard
        ```

        Selecteer **Ollama** en kies vervolgens een modus: **Cloud + lokaal**, **Alleen cloud** of **Alleen lokaal**.

        Bij een nieuwe begeleide configuratie controleert OpenClaw eerst de standaard of geconfigureerde
        Ollama-host. Een geïnstalleerd model wordt alleen automatisch aangeboden wanneer
        `/api/show` ondersteuning voor tools en een contextvenster van ten minste 16K bevestigt;
        ontbrekende metadata of metadata voor een kleiner contextvenster blijft in het handmatige configuratiepad. De
        gedeelde configuratieladder voor CLI/macOS verifieert de geselecteerde route nog steeds met een
        echte voltooiing voordat deze wordt opgeslagen. Deze automatische controle haalt nooit een
        model op; als er geen geschikt geïnstalleerd model bestaat, gaat de onboarding verder naar de
        normale Ollama-kiezer.
      </Step>
      <Step title="Een model selecteren">
        `Cloud only` vraagt om `OLLAMA_API_KEY` en stelt gehoste cloudstandaarden voor. `Cloud + Local` en `Local only` vragen om een Ollama-basis-URL, detecteren beschikbare modellen en halen het geselecteerde lokale model automatisch op als het ontbreekt. Een geïnstalleerde `:latest`-tag zoals `gemma4:latest` wordt eenmaal weergegeven in plaats van `gemma4` te dupliceren. `Cloud + Local` controleert ook of de host is aangemeld voor cloudtoegang.
      </Step>
      <Step title="Verifiëren">
        ```bash
        openclaw models list --provider ollama
        ```
      </Step>
    </Steps>

    Niet-interactief:

    ```bash
    openclaw onboard --non-interactive \
      --auth-choice ollama \
      --custom-base-url "http://ollama-host:11434" \
      --custom-model-id "qwen3.5:27b" \
      --accept-risk
    ```

    `--custom-base-url` en `--custom-model-id` zijn optioneel; als je ze weglaat, worden de lokale standaardhost en het voorgestelde model `gemma4` gebruikt.

  </Tab>

  <Tab title="Handmatige configuratie">
    <Steps>
      <Step title="Ollama installeren en starten">
        Download het via [ollama.com/download](https://ollama.com/download) en haal vervolgens een model op:

        ```bash
        ollama pull gemma4
        ```

        Voer voor hybride cloudtoegang `ollama signin` uit op dezelfde host.
      </Step>
      <Step title="Aanmeldgegevens instellen">
        ```bash
        export OLLAMA_API_KEY="ollama-local"    # lokale/LAN-host, elke waarde werkt
        export OLLAMA_API_KEY="your-real-key"   # alleen https://ollama.com
        ```

        Of in de configuratie: `openclaw config set models.providers.ollama.apiKey "OLLAMA_API_KEY"`.
      </Step>
      <Step title="Het model selecteren">
        ```bash
        openclaw models list
        openclaw models set ollama/gemma4
        ```

        Of in de configuratie:

        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "ollama/gemma4" },
            },
          },
        }
        ```
      </Step>
    </Steps>

  </Tab>
</Tabs>

## Cloudmodellen via een lokale host

`Cloud + Local` routeert zowel lokale als `:cloud`-modellen via één bereikbare
Ollama-host — dit is de hybride flow van Ollama en de modus die je tijdens de configuratie moet kiezen
wanneer je beide wilt.

OpenClaw vraagt om de basis-URL, detecteert lokale modellen en controleert de
`ollama signin`-status. Wanneer je bent aangemeld, stelt het gehoste standaarden voor
(`kimi-k2.5:cloud`, `minimax-m2.7:cloud`, `glm-5.1:cloud`, `glm-5.2:cloud`). Als je
niet bent aangemeld, blijft de configuratie alleen lokaal totdat je `ollama signin` uitvoert.

Gebruik voor toegang tot uitsluitend de cloud zonder lokale daemon `openclaw onboard --auth-choice ollama-cloud` en raadpleeg [Ollama Cloud](/nl/providers/ollama-cloud) — voor dat pad zijn `ollama signin` en een actieve server niet nodig:

```bash
openclaw onboard --auth-choice ollama-cloud
openclaw models set ollama-cloud/kimi-k2.5:cloud
```

De lijst met cloudmodellen die tijdens `openclaw onboard` wordt weergegeven, wordt live gevuld vanuit
`https://ollama.com/api/tags` en is beperkt tot 500 vermeldingen, zodat de kiezer de
huidige gehoste catalogus weergeeft. Als `ollama.com` niet bereikbaar is of tijdens de
configuratie geen modellen retourneert, valt OpenClaw terug op de hardgecodeerde lijst met suggesties, zodat
de onboarding toch wordt voltooid.

## Modeldetectie (impliciete provider)

Wanneer `OLLAMA_API_KEY` (of een authenticatieprofiel) is ingesteld en noch
`models.providers.ollama`, noch een andere aangepaste provider met `api: "ollama"` is
gedefinieerd, detecteert OpenClaw modellen via `http://127.0.0.1:11434`:

| Gedrag               | Details                                                                                                                                                                                                                                                                                       |
| -------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Catalogusquery       | `/api/tags`                                                                                                                                                                                                                                                                            |
| Mogelijkhedendetectie | Best-effort `/api/show` leest `contextWindow`, `num_ctx`-Modelfile-parameters en mogelijkheden (beeld/tools/redeneren)                                                                                                                                                       |
| Beeldmodellen        | Een `vision`-mogelijkheid uit `/api/show` markeert het model als geschikt voor afbeeldingen (`input: ["text", "image"]`)                                                                                                                                                              |
| Redeneerdetectie     | Gebruikt indien beschikbaar de `thinking`-mogelijkheid uit `/api/show`; valt terug op een heuristiek op basis van de naam (`r1`, `reason`, `reasoning`, `think`) wanneer Ollama mogelijkheden weglaat. `glm-5.2:cloud` en `deepseek-v4-flash\|pro:cloud` worden altijd als redeneermodellen behandeld, ongeacht de gerapporteerde mogelijkheden. |
| Tokenlimieten        | `maxTokens` gebruikt standaard de maximale tokenlimiet van OpenClaw voor Ollama                                                                                                                                                                                                         |
| Kosten               | Alle kosten zijn `0`                                                                                                                                                                                                                                                           |

```bash
ollama list
openclaw models list
```

Het instellen van `models.providers.ollama` met een expliciete `models`-array, of een
aangepaste provider met `api: "ollama"` en een niet-loopback `baseUrl`, schakelt
automatische detectie uit; modellen moeten dan handmatig worden gedefinieerd (zie
[Configuratie](#configuration)). Een `models.providers.ollama`-vermelding die naar de gehoste
`https://ollama.com` verwijst, slaat detectie eveneens over, omdat Ollama Cloud-modellen
door de provider worden beheerd. Aangepaste loopback-providers zoals
`http://127.0.0.2:11434` gelden nog steeds als lokaal en behouden automatische detectie.

Je kunt een volledige verwijzing zoals `ollama/<pulled-model>:latest` gebruiken zonder een
handmatig geschreven `models.json`-vermelding; OpenClaw verwerkt deze live. Voor aangemelde
hosts valideert het selecteren van een niet-vermelde `ollama/<model>:cloud`-verwijzing dat exacte
model met `/api/show` en voegt het alleen aan de runtimecatalogus toe als Ollama
metadata bevestigt — typefouten blijven mislukken als onbekende modellen.

### Rooktests

Voor een gerichte tekstprobe die het volledige oppervlak van agenttools overslaat:

```bash
OLLAMA_API_KEY=ollama-local \
  openclaw infer model run \
    --local \
    --model ollama/llama3.2:latest \
    --prompt "Reply with exactly: pong" \
    --json
```

Voeg `--file` met een afbeelding toe voor een lichte probe van een beeldmodel (accepteert PNG/JPEG/WebP;
bestanden die geen afbeelding zijn, worden geweigerd voordat Ollama wordt aangeroepen — gebruik
`openclaw infer audio transcribe` voor audio):

```bash
OLLAMA_API_KEY=ollama-local \
  openclaw infer model run \
    --local \
    --model ollama/qwen2.5vl:7b \
    --prompt "Describe this image in one sentence." \
    --file ./photo.jpg \
    --json
```

Geen van beide paden laadt chattools, geheugen of sessiecontext. Als dit slaagt
terwijl normale agentantwoorden mislukken, ligt het probleem waarschijnlijk bij de tool-/agentcapaciteit
van het model en niet bij het eindpunt.

Een model selecteren met `/model ollama/<model>` is een exacte gebruikerskeuze: als de
geconfigureerde `baseUrl` onbereikbaar is, mislukt het volgende antwoord met de providerfout
in plaats van stilzwijgend terug te vallen op een ander geconfigureerd model.

Geïsoleerde Cron-taken voegen één lokale veiligheidscontrole toe voordat de agentbeurt begint:
als het geselecteerde model wordt omgezet naar een lokale/privénetwerk-/`.local` Ollama-
provider en `/api/tags` onbereikbaar is, registreert OpenClaw die uitvoering als
`skipped` met het model in de fouttekst. Deze eindpuntcontrole wordt
5 minuten per host in de cache opgeslagen, zodat herhaalde Cron-taken voor een gestopte daemon niet allemaal
mislukkende verzoeken starten.

Liveverificatie:

```bash
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_OLLAMA=1 OPENCLAW_LIVE_OLLAMA_WEB_SEARCH=0 \
  pnpm test:live -- extensions/ollama/ollama.live.test.ts
```

Laat voor Ollama Cloud dezelfde livetest naar het gehoste eindpunt wijzen (slaat
embeddings standaard over; forceer ze met `OPENCLAW_LIVE_OLLAMA_EMBEDDINGS=1`, omdat een
cloudsleutel mogelijk geen autorisatie geeft voor `/api/embed`):

```bash
export OLLAMA_API_KEY='<your-ollama-cloud-api-key>'
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_OLLAMA=1 \
OPENCLAW_LIVE_OLLAMA_BASE_URL=https://ollama.com \
OPENCLAW_LIVE_OLLAMA_MODEL=glm-5.1:cloud \
OPENCLAW_LIVE_OLLAMA_WEB_SEARCH=1 \
pnpm test:live -- extensions/ollama/ollama.live.test.ts
```

Haal een model op om het toe te voegen; het wordt automatisch ontdekt:

```bash
ollama pull mistral
```

## Node-lokale inferentie

Agents kunnen een korte taak delegeren aan een Ollama-model op een gekoppelde desktop- of
server-Node. De prompt en het antwoord lopen via de bestaande geauthenticeerde
Gateway/Node-verbinding; het verzoek wordt uitgevoerd via het eigen loopback-Ollama-
eindpunt van de Node (`http://127.0.0.1:11434`).

<Steps>
  <Step title="Ollama op de Node starten">
    ```bash
    ollama pull qwen3:0.6b
    ollama list
    ```
  </Step>
  <Step title="De Node-host verbinden">
    ```bash
    openclaw node run \
      --host <gateway-host> \
      --port 18789 \
      --display-name "Local inference"
    ```

    Keur het apparaat en de bijbehorende Node-opdrachten goed op de Gateway-host en verifieer vervolgens:

    ```bash
    openclaw devices list
    openclaw devices approve <deviceRequestId>
    openclaw nodes pending
    openclaw nodes approve <nodeRequestId>
    openclaw nodes status --connected
    ```

    Een eerste verbinding, of een upgrade die Ollama-opdrachten toevoegt, kan
    goedkeuring voor Node-opdrachten activeren. Als de Node verbinding maakt zonder
    `ollama.models` en `ollama.chat` aan te kondigen, controleer `openclaw nodes pending` opnieuw.

  </Step>
  <Step title="Gebruiken vanuit een agent">
    De meegeleverde Ollama-Plugin stelt de tool `node_inference` beschikbaar. Agents roepen
    eerst `action: "discover"` aan en daarna `action: "run"` met een Node en model uit
    dat resultaat (`run` kan de Node weglaten wanneer precies één geschikte Node is
    verbonden). Bijvoorbeeld: "Ontdek de Ollama-modellen op mijn Nodes en gebruik
    vervolgens het snelste geladen model om deze tekst samen te vatten."
  </Step>
</Steps>

De detectie leest `/api/tags`, controleert de mogelijkheden van `/api/show` en gebruikt
`/api/ps` indien beschikbaar om reeds geladen modellen als eerste te rangschikken. Ze retourneert alleen
lokale modellen die Ollama als chatgeschikt rapporteert (mogelijkheid `completion`) —
Ollama Cloud-vermeldingen en modellen die alleen embeddings ondersteunen, worden uitgesloten. Elke uitvoering schakelt
het denkproces van het model uit en stelt de uitvoer standaard in op 512 tokens (harde limiet 8192), tenzij de
toolaanroep een andere `maxTokens` aanvraagt; sommige modellen (bijvoorbeeld GPT-OSS)
ondersteunen het uitschakelen van het denkproces niet en kunnen nog steeds redeneertokens uitvoeren.

Ollama op een Node actief houden zonder het aan agents beschikbaar te stellen:

```bash
openclaw config set plugins.entries.ollama.config.nodeInference.enabled false
```

Start de Node opnieuw (`openclaw node restart`, of stop en voer `openclaw node run` opnieuw uit
voor een voorgrondsessie). De Node stopt met het aankondigen van `ollama.models` en
`ollama.chat`; Ollama zelf en de Ollama-provider van de Gateway blijven ongewijzigd.
Zet de waarde terug op `true` en start opnieuw om dit weer in te schakelen; een gewijzigd opdrachtenoppervlak
kan na het opnieuw verbinden opnieuw goedkeuring voor `openclaw nodes pending` vereisen.

Verifieer de Node-opdrachten rechtstreeks, zonder agentbeurt:

```bash
openclaw nodes invoke \
  --node "Local inference" \
  --command ollama.models \
  --params '{}' \
  --invoke-timeout 90000 \
  --timeout 100000

openclaw nodes invoke \
  --node "Local inference" \
  --command ollama.chat \
  --params '{"model":"qwen3:0.6b","prompt":"Reply with exactly: pong","maxTokens":32,"timeoutMs":120000}' \
  --invoke-timeout 130000 \
  --timeout 140000
```

`--invoke-timeout` begrenst hoe lang de Node de opdracht mag uitvoeren;
`--timeout` begrenst de volledige Gateway-aanroep en moet groter zijn.

Node-lokale inferentie gebruikt altijd het eigen loopback-eindpunt van de Node — ze
hergebruikt geen geconfigureerde externe/cloud-`models.providers.ollama.baseUrl`. De
Node-opdrachten zijn standaard beschikbaar op macOS-, Linux- en Windows-Node-
hosts en blijven onderworpen aan het normale beleid voor Node-koppeling en -opdrachten.

## Visie en afbeeldingsbeschrijving

De meegeleverde Ollama-Plugin registreert Ollama als een provider voor
mediabegrip met afbeeldingsondersteuning, zodat OpenClaw expliciete verzoeken om afbeeldingsbeschrijvingen
en geconfigureerde standaardinstellingen voor afbeeldingsmodellen kan routeren via lokale of gehoste
Ollama-visiemodellen.

```bash
ollama pull qwen2.5vl:7b
export OLLAMA_API_KEY="ollama-local"
openclaw infer image describe --file ./photo.jpg --model ollama/qwen2.5vl:7b --json
```

`--model` moet een volledige `<provider/model>`-referentie zijn; wanneer deze is ingesteld, probeert `infer image
describe` eerst dat model in plaats van de beschrijving over te slaan voor modellen
die al native visie ondersteunen. Als de aanroep mislukt, kan OpenClaw doorgaan
via `agents.defaults.imageModel.fallbacks`; fouten bij de voorbereiding van bestanden/URL's
mislukken voordat een fallback wordt geprobeerd. Gebruik `infer image describe` voor OpenClaws
stroom voor afbeeldingsbegrip en de geconfigureerde `imageModel`; gebruik `infer model run
--file` voor een onbewerkte multimodale test met een aangepaste prompt.

Ollama instellen als standaardprovider voor het begrijpen van inkomende media-afbeeldingen:

```json5
{
  agents: {
    defaults: {
      imageModel: {
        primary: "ollama/qwen2.5vl:7b",
      },
    },
  },
}
```

Geef de voorkeur aan de volledige `ollama/<model>`-referentie. Een kale `imageModel`-referentie zoals
`qwen2.5vl:7b` wordt alleen genormaliseerd naar `ollama/qwen2.5vl:7b` wanneer precies dat model
onder `models.providers.ollama.models` wordt vermeld met
`input: ["text", "image"]` en geen andere geconfigureerde afbeeldingsprovider
dezelfde kale id aanbiedt; gebruik anders expliciet het providerprefix.

Trage lokale visiemodellen kunnen voor afbeeldingsbegrip een langere time-out nodig hebben dan
cloudmodellen en kunnen op hardware met beperkte middelen vastlopen als Ollama probeert
de volledige geadverteerde visiecontext van het model toe te wijzen. Stel een time-out voor de mogelijkheid in
en begrens `num_ctx`:

```json5
{
  models: {
    providers: {
      ollama: {
        models: [
          {
            id: "qwen2.5vl:7b",
            name: "qwen2.5vl:7b",
            input: ["text", "image"],
            params: { num_ctx: 2048, keep_alive: "1m" },
          },
        ],
      },
    },
  },
  tools: {
    media: {
      image: {
        timeoutSeconds: 180,
        models: [{ provider: "ollama", model: "qwen2.5vl:7b", timeoutSeconds: 300 }],
      },
    },
  },
}
```

Deze time-out geldt voor het begrijpen van inkomende afbeeldingen en voor de expliciete
tool `image`. `models.providers.ollama.timeoutSeconds` regelt nog steeds de
onderliggende beveiliging voor Ollama-HTTP-verzoeken bij normale modelaanroepen.

Liveverificatie:

```bash
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_OLLAMA_IMAGE=1 \
  pnpm test:live -- src/agents/tools/image-tool.ollama.live.test.ts
```

Als je `models.providers.ollama.models` handmatig definieert, markeer visiemodellen dan
expliciet:

```json5
{
  id: "qwen2.5vl:7b",
  name: "qwen2.5vl:7b",
  input: ["text", "image"],
  contextWindow: 128000,
  maxTokens: 8192,
}
```

OpenClaw weigert verzoeken om afbeeldingsbeschrijvingen voor modellen die niet als
afbeeldingsgeschikt zijn gemarkeerd. Bij impliciete detectie is dit afkomstig van de visiemogelijkheid
van `/api/show`.

## Configuratie

<Tabs>
  <Tab title="Basis (impliciete detectie)">
    ```bash
    export OLLAMA_API_KEY="ollama-local"
    ```

    <Tip>
    Als `OLLAMA_API_KEY` is ingesteld, kun je `apiKey` weglaten in de providervermelding; OpenClaw vult deze in voor beschikbaarheidscontroles.
    </Tip>

  </Tab>

  <Tab title="Expliciet (handmatige modellen)">
    Gebruik expliciete configuratie voor een gehoste cloudconfiguratie, een niet-standaardhost/-poort, geforceerde
    contextvensters of volledig handmatige modellenlijsten:

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "https://ollama.com",
            apiKey: "OLLAMA_API_KEY",
            api: "ollama",
            models: [
              {
                id: "kimi-k2.5:cloud",
                name: "kimi-k2.5:cloud",
                reasoning: false,
                input: ["text", "image"],
                cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
                contextWindow: 128000,
                maxTokens: 8192
              }
            ]
          }
        }
      }
    }
    ```

  </Tab>

  <Tab title="Aangepaste basis-URL">
    Expliciete configuratie schakelt automatische detectie uit, dus modellen moeten worden vermeld:

    ```json5
    {
      models: {
        providers: {
          ollama: {
            apiKey: "ollama-local",
            baseUrl: "http://ollama-host:11434", // Geen /v1 - URL van de native Ollama-API
            api: "ollama", // Expliciet: garandeert native gedrag voor toolaanroepen
            timeoutSeconds: 300, // Optioneel: langer verbindings-/streambudget voor koude lokale modellen
            models: [
              {
                id: "qwen3:32b",
                name: "qwen3:32b",
                params: {
                  keep_alive: "15m", // Optioneel: houd het model tussen beurten geladen
                },
              },
            ],
          },
        },
      },
    }
    ```

    <Warning>
    Voeg `/v1` niet toe. Dat pad selecteert de OpenAI-compatibele modus, waarin toolaanroepen niet betrouwbaar zijn.
    </Warning>

  </Tab>
</Tabs>

## Veelgebruikte recepten

Vervang model-id's door exacte namen uit `ollama list` of
`openclaw models list --provider ollama`.

<AccordionGroup>
  <Accordion title="Lokaal model met automatische detectie">
    Ollama op dezelfde machine als de Gateway, automatisch gedetecteerd:

    ```bash
    ollama serve
    ollama pull gemma4
    export OLLAMA_API_KEY="ollama-local"
    openclaw models list --provider ollama
    openclaw models set ollama/gemma4
    ```

    Voeg geen `models.providers.ollama`-blok toe, tenzij je handmatige modellen nodig hebt.

  </Accordion>

  <Accordion title="Ollama-host op het LAN met handmatige modellen">
    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://gpu-box.local:11434",
            apiKey: "ollama-local",
            api: "ollama",
            timeoutSeconds: 300,
            contextWindow: 32768,
            maxTokens: 8192,
            models: [
              {
                id: "qwen3.5:9b",
                name: "qwen3.5:9b",
                reasoning: true,
                input: ["text"],
                params: {
                  num_ctx: 32768,
                  thinking: false,
                  keep_alive: "15m",
                },
              },
            ],
          },
        },
      },
      agents: {
        defaults: {
          model: { primary: "ollama/qwen3.5:9b" },
        },
      },
    }
    ```

    `contextWindow` is OpenClaws contextbudget; `params.num_ctx` wordt naar
    Ollama verzonden. Houd ze op elkaar afgestemd wanneer de hardware niet de volledige
    geadverteerde context van het model kan uitvoeren.

  </Accordion>

  <Accordion title="Alleen Ollama Cloud">
    Geen lokale daemon, rechtstreeks gehoste modellen:

    ```bash
    export OLLAMA_API_KEY="your-ollama-api-key"
    ```

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "https://ollama.com",
            apiKey: "OLLAMA_API_KEY",
            api: "ollama",
            models: [
              {
                id: "kimi-k2.5:cloud",
                name: "kimi-k2.5:cloud",
                reasoning: false,
                input: ["text", "image"],
                contextWindow: 128000,
                maxTokens: 8192,
              },
            ],
          },
        },
      },
      agents: {
        defaults: {
          model: { primary: "ollama/kimi-k2.5:cloud" },
        },
      },
    }
    ```

    Zie [Ollama Cloud](/nl/providers/ollama-cloud) voor de specifieke provider-id
    `ollama-cloud` in plaats van deze structuur.

  </Accordion>

  <Accordion title="Cloud plus lokaal via een aangemelde daemon">
    ```bash
    ollama signin
    ollama pull gemma4
    ```

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://127.0.0.1:11434",
            apiKey: "ollama-local",
            api: "ollama",
            timeoutSeconds: 300,
            models: [
              { id: "gemma4", name: "gemma4", input: ["text"] },
              { id: "kimi-k2.5:cloud", name: "kimi-k2.5:cloud", input: ["text", "image"] },
            ],
          },
        },
      },
      agents: {
        defaults: {
          model: {
            primary: "ollama/gemma4",
            fallbacks: ["ollama/kimi-k2.5:cloud"],
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Meerdere Ollama-hosts">
    Aangepaste provider-id's bij het uitvoeren van meer dan één Ollama-server;
    elke server krijgt een eigen host, modellen, authenticatie en time-out.

    ```json5
    {
      models: {
        providers: {
          "ollama-fast": {
            baseUrl: "http://mini.local:11434",
            apiKey: "ollama-local",
            api: "ollama",
            contextWindow: 32768,
            models: [{ id: "gemma4", name: "gemma4", input: ["text"] }],
          },
          "ollama-large": {
            baseUrl: "http://gpu-box.local:11434",
            apiKey: "ollama-local",
            api: "ollama",
            timeoutSeconds: 420,
            contextWindow: 131072,
            maxTokens: 16384,
            models: [{ id: "qwen3.5:27b", name: "qwen3.5:27b", input: ["text"] }],
          },
        },
      },
      agents: {
        defaults: {
          model: {
            primary: "ollama-fast/gemma4",
            fallbacks: ["ollama-large/qwen3.5:27b"],
          },
        },
      },
    }
    ```

    OpenClaw verwijdert het actieve providerprefix (met een kaal
    `ollama/`-prefix als terugvaloptie) voordat Ollama wordt aangeroepen,
    zodat `ollama-large/qwen3.5:27b` Ollama bereikt als `qwen3.5:27b`.

  </Accordion>

  <Accordion title="Slank profiel voor lokale modellen">
    Sommige lokale modellen kunnen eenvoudige prompts verwerken, maar hebben
    moeite met de volledige toolset van de agent. Beperk tools en context voordat
    je algemene runtime-instellingen wijzigt:

    ```json5
    {
      agents: {
        list: [
          {
            id: "local",
            experimental: {
              localModelLean: true,
            },
            model: { primary: "ollama/gemma4" },
          },
        ],
      },
      models: {
        providers: {
          ollama: {
            baseUrl: "http://127.0.0.1:11434",
            apiKey: "ollama-local",
            api: "ollama",
            contextWindow: 32768,
            models: [
              {
                id: "gemma4",
                name: "gemma4",
                input: ["text"],
                params: { num_ctx: 32768 },
                compat: { supportsTools: false },
              },
            ],
          },
        },
      },
    }
    ```

    Gebruik `compat.supportsTools: false` alleen wanneer het model of de server
    herhaaldelijk faalt op toolschema's — dit verruilt agentmogelijkheden voor
    stabiliteit. `localModelLean` verwijdert zware browser-, cron-, bericht-,
    mediageneratie-, spraak- en PDF-tools van het directe agentoppervlak, tenzij
    ze expliciet vereist zijn, en plaatst grotere catalogi achter Tool Search.
    Het verandert de runtimecontext of denkmodus van Ollama niet. Combineer het
    met `params.num_ctx` en `params.thinking: false` voor kleine Qwen-achtige
    denkmodellen die in een lus raken of hun budget aan verborgen redeneringen
    besteden.

  </Accordion>
</AccordionGroup>

### Modelselectie

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "ollama/gpt-oss:20b",
        fallbacks: ["ollama/llama3.3", "ollama/qwen2.5-coder:32b"],
      },
    },
  },
}
```

Aangepaste provider-id's werken op dezelfde manier: voor een verwijzing met het
actieve providerprefix, zoals `ollama-spark/qwen3:32b`, verwijdert OpenClaw dat prefix
voordat Ollama wordt aangeroepen en wordt `qwen3:32b` verzonden.

Geef bij trage lokale modellen de voorkeur aan afstemming op providerniveau
voordat je de time-out van de volledige agentruntime verhoogt:

```json5
{
  models: {
    providers: {
      ollama: {
        timeoutSeconds: 300,
        models: [
          {
            id: "gemma4:26b",
            name: "gemma4:26b",
            params: { keep_alive: "15m" },
          },
        ],
      },
    },
  },
}
```

`timeoutSeconds` omvat de HTTP-aanvraag voor het model: het opzetten van de
verbinding, headers, het streamen van de body en het volledig bewaakte afbreken
van de fetch. `params.keep_alive` wordt doorgestuurd als `keep_alive` op
het hoogste niveau bij native `/api/chat`-aanvragen; stel dit per model
in wanneer de laadtijd van de eerste beurt de beperkende factor is.

### Snelle verificatie

```bash
# Ollama-daemon zichtbaar voor deze machine
curl http://127.0.0.1:11434/api/tags

# OpenClaw-catalogus en geselecteerd model
openclaw models list --provider ollama
openclaw models status

# Directe rooktest van het model
openclaw infer model run \
  --model ollama/gemma4 \
  --prompt "Antwoord exact met: ok"
```

Vervang voor externe hosts `127.0.0.1` door de `baseUrl`-host.
Als `curl` werkt maar OpenClaw niet, controleer dan of de Gateway op
een andere machine, in een container of onder een ander serviceaccount draait.

## Ollama Web Search

OpenClaw levert **Ollama Web Search** mee als een `web_search`-provider.

| Eigenschap   | Details                                                                                                                                                    |
| ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Host         | `models.providers.ollama.baseUrl` wanneer ingesteld, anders `http://127.0.0.1:11434`; `https://ollama.com` gebruikt rechtstreeks de gehoste API                                   |
| Authenticatie | Zonder sleutel voor een aangemelde lokale host; `OLLAMA_API_KEY` of geconfigureerde providerauthenticatie voor rechtstreeks zoeken via `https://ollama.com` of hosts met authenticatiebeveiliging |
| Vereiste     | Lokale/zelfgehoste hosts moeten actief en aangemeld zijn met `ollama signin`; rechtstreeks gehost zoeken vereist `baseUrl: "https://ollama.com"` plus een echte API-sleutel |

Kies dit tijdens `openclaw onboard` of `openclaw configure --section web`, of stel het volgende in:

```json5
{
  tools: {
    web: {
      search: {
        provider: "ollama",
      },
    },
  },
}
```

Voor rechtstreeks gehost zoeken via Ollama Cloud:

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "https://ollama.com",
        apiKey: "OLLAMA_API_KEY",
        api: "ollama",
        models: [{ id: "kimi-k2.5:cloud", name: "kimi-k2.5:cloud", input: ["text"] }],
      },
    },
  },
  tools: {
    web: {
      search: { provider: "ollama" },
    },
  },
}
```

Voor een zelfgehoste host probeert OpenClaw eerst de lokale
`/api/experimental/web_search`-proxy en valt vervolgens terug op het gehoste
`/api/web_search`-pad op dezelfde host; een aangemelde lokale daemon antwoordt
normaal gesproken via de lokale proxy. Rechtstreekse `https://ollama.com`-aanroepen
gebruiken altijd het gehoste `/api/web_search`-eindpunt.

<Note>
Zie [Ollama Web Search](/nl/tools/ollama-search) voor de volledige configuratie en werking.
</Note>

## Geavanceerde configuratie

<AccordionGroup>
  <Accordion title="Verouderde OpenAI-compatibele modus">
    <Warning>
    **Toolaanroepen zijn in deze modus niet betrouwbaar.** Gebruik deze modus alleen wanneer een proxy de OpenAI-indeling vereist en je niet afhankelijk bent van native toolaanroepen.
    </Warning>

    Stel `api: "openai-completions"` expliciet in voor een proxy achter
    `/v1/chat/completions`:

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://ollama-host:11434/v1",
            api: "openai-completions",
            injectNumCtxForOpenAICompat: true, // standaard: true
            apiKey: "ollama-local",
            models: [...]
          }
        }
      }
    }
    ```

    Deze modus ondersteunt mogelijk niet tegelijkertijd streaming en
    toolaanroepen; mogelijk is `params: { streaming: false }` op het model nodig.

    OpenClaw injecteert in deze modus standaard `options.num_ctx`, zodat Ollama
    niet stilzwijgend terugvalt op een context van 4096 tokens. Als je proxy
    onbekende `options`-velden weigert, schakel dit dan uit:

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://ollama-host:11434/v1",
            api: "openai-completions",
            injectNumCtxForOpenAICompat: false,
            apiKey: "ollama-local",
            models: [...]
          }
        }
      }
    }
    ```

  </Accordion>

  <Accordion title="Contextvensters">
    Voor automatisch ontdekte modellen gebruikt OpenClaw het contextvenster dat
    `/api/show` rapporteert, inclusief grotere `PARAMETER num_ctx`-waarden
    uit aangepaste Modelfiles; anders valt het terug op het standaard
    Ollama-contextvenster van OpenClaw.

    `contextWindow`, `contextTokens` en `maxTokens` op
    providerniveau stellen standaardwaarden in voor elk model onder die provider
    en kunnen per model worden overschreven. `contextWindow` is het eigen
    prompt-/Compaction-budget van OpenClaw. Native `/api/chat`-aanvragen
    laten `options.num_ctx` oningesteld, tenzij je `params.num_ctx` expliciet
    instelt, zodat Ollama zijn eigen standaardwaarde op basis van het model,
    `OLLAMA_CONTEXT_LENGTH` of VRAM toepast; ongeldige, nul-, negatieve of
    niet-eindige `params.num_ctx`-waarden worden genegeerd. Als een oudere
    configuratie alleen `contextWindow`/`maxTokens` gebruikte om de
    context van native aanvragen af te dwingen, voer dan `openclaw doctor --fix` uit
    om deze waarden naar `params.num_ctx` te kopiëren. De OpenAI-compatibele
    adapter injecteert nog steeds standaard `options.num_ctx` vanuit de
    geconfigureerde `params.num_ctx` of `contextWindow`; schakel dit uit met
    `injectNumCtxForOpenAICompat: false` als de upstream `options` weigert.

    Native modelvermeldingen accepteren ook algemene Ollama-runtimeopties onder
    `params`, die worden doorgestuurd als native `/api/chat`
    `options`: `num_keep`, `seed`,
    `num_predict`, `top_k`, `top_p`,
    `min_p`, `typical_p`, `repeat_last_n`,
    `temperature`, `repeat_penalty`, `presence_penalty`,
    `frequency_penalty`, `stop`, `num_batch`,
    `num_gpu`, `main_gpu`, `use_mmap` en
    `num_thread`. Enkele sleutels (`format`,
    `keep_alive`, `truncate`, `shift`) worden als
    aanvraagvelden op het hoogste niveau doorgestuurd in plaats van genest onder
    `options`. OpenClaw stuurt alleen deze Ollama-aanvraagsleutels door,
    zodat uitsluitend voor de runtime bestemde parameters zoals
    `streaming` nooit naar Ollama worden verzonden. Gebruik
    `params.think` (of `params.thinking`) om `think` op het
    hoogste niveau in te stellen; `false` schakelt denken op API-niveau
    uit voor Qwen-achtige denkmodellen.

    ```json5
    {
      models: {
        providers: {
          ollama: {
            contextWindow: 32768,
            models: [
              {
                id: "llama3.3",
                contextWindow: 131072,
                maxTokens: 65536,
                params: {
                  num_ctx: 32768,
                  temperature: 0.7,
                  top_p: 0.9,
                  thinking: false,
                },
              }
            ]
          }
        }
      }
    }
    ```

    `agents.defaults.models["ollama/<model>"].params.num_ctx` per model werkt ook; de expliciete modelvermelding van de provider heeft voorrang als beide zijn ingesteld.

  </Accordion>

  <Accordion title="Denkcontrole">
    OpenClaw stuurt denken door zoals Ollama het verwacht: `think` op het hoogste niveau, niet
    `options.think`. Automatisch ontdekte modellen waarvan `/api/show` een
    `thinking`-mogelijkheid rapporteert, bieden `/think low`, `/think medium`, `/think high`
    en `/think max`; modellen zonder denkfunctie bieden alleen `/think off`.

    ```bash
    openclaw agent --model ollama/gemma4 --thinking off
    openclaw agent --model ollama/gemma4 --thinking low
    ```

    Of stel een standaardwaarde voor een model in:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "ollama/gemma4": {
              thinking: "low",
            },
          },
        },
      },
    }
    ```

    `params.think`/`params.thinking` per model kan API-denken voor een specifiek
    model uitschakelen of afdwingen. OpenClaw behoudt die expliciete configuratie
    wanneer de actieve uitvoering alleen de impliciete standaardwaarde `off` heeft; een runtimeopdracht die niet op uit staat,
    zoals `/think medium`, overschrijft deze nog steeds. Een ingeschakeld
    denkverzoek wordt nooit verzonden naar een model dat expliciet is gemarkeerd als
    `reasoning: false`; een `think: false`-verzoek wordt altijd verzonden.

  </Accordion>

  <Accordion title="Redeneermodellen">
    Modellen met de naam `deepseek-r1`, `reasoning`, `reason` of `think` worden
    standaard behandeld als modellen met redeneervermogen — er is geen extra configuratie nodig:

    ```bash
    ollama pull deepseek-r1:32b
    ```

  </Accordion>

  <Accordion title="Modelkosten">
    Ollama wordt lokaal uitgevoerd en is gratis, dus alle modelkosten zijn `0` voor zowel
    automatisch ontdekte als handmatig gedefinieerde modellen.
  </Accordion>

  <Accordion title="Geheugen-embeddings">
    De meegeleverde Ollama-plugin registreert een provider voor geheugen-embeddings voor
    [zoeken in het geheugen](/nl/concepts/memory). Deze gebruikt de geconfigureerde basis-URL
    en API-sleutel van Ollama, roept `/api/embed` aan en bundelt waar mogelijk meerdere geheugenfragmenten in
    één `input`-verzoek.

    Wanneer `proxy.enabled=true` gebruiken embeddingverzoeken naar de exacte hostlokale
    loopback-oorsprong die is afgeleid van de geconfigureerde `baseUrl` het
    beveiligde directe pad van OpenClaw in plaats van de beheerde forwardproxy. De geconfigureerde
    hostnaam moet zelf `localhost` of een letterlijk loopback-IP-adres zijn — DNS-namen
    die alleen naar loopback worden omgezet, gebruiken nog steeds het beheerde proxypad. Ollama-hosts op het LAN,
    tailnet, privénetwerk en openbare netwerk blijven altijd het
    beheerde proxypad gebruiken, en omleidingen naar een andere host/poort nemen het
    vertrouwen niet over. `proxy.loopbackMode: "proxy"` leidt loopback-verkeer toch via de
    proxy; `proxy.loopbackMode: "block"` weigert het voordat verbinding wordt gemaakt —
    zie [Beheerde proxy](/nl/security/network-proxy#gateway-loopback-mode).

    | Eigenschap | Waarde |
    | --- | --- |
    | Standaardmodel | `nomic-embed-text` |
    | Automatisch ophalen | Ja, als het niet lokaal aanwezig is |
    | Standaard gelijktijdigheid voor inlineverwerking | 1 (andere providers hebben standaard een hogere waarde; verhoog deze met `nonBatchConcurrency` als de host dit aankan) |

    Embeddings tijdens zoekopdrachten gebruiken voorvoegsels voor ophalen bij modellen die deze vereisen of
    aanbevelen: `nomic-embed-text`, `qwen3-embedding` en
    `mxbai-embed-large`. Documentbatches blijven ongewijzigd, zodat bestaande indexen
    geen formaatmigratie nodig hebben.

    ```json5
    {
      memory: {
        search: {
          provider: "ollama",
          remote: {
            // Standaard voor Ollama. Verhoog dit op grotere hosts als opnieuw indexeren te langzaam gaat.
            nonBatchConcurrency: 1,
          },
        },
      },
    }
    ```

    Beperk voor een externe embeddinghost de authenticatie tot die host:

    ```json5
    {
      memory: {
        search: {
          provider: "ollama",
          model: "nomic-embed-text",
          remote: {
            baseUrl: "http://gpu-box.local:11434",
            apiKey: "ollama-local",
            nonBatchConcurrency: 2,
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Streamingconfiguratie">
    Ollama gebruikt standaard de **native API** (`/api/chat`), die
    streaming en toolaanroepen tegelijk ondersteunt — er is geen speciale configuratie nodig.

    Voor native verzoeken wordt de denkcontrole rechtstreeks doorgestuurd: `/think off`
    en `openclaw agent --thinking off` verzenden `think: false` op het hoogste niveau, tenzij
    expliciet `params.think`/`params.thinking` is geconfigureerd; `/think
    low|medium|high` verzendt de bijbehorende inspanningswaarde; `/think max` wordt gekoppeld aan
    Ollama's hoogste inspanningsniveau, `think: "high"`.

    <Tip>
    Zie voor het OpenAI-compatibele eindpunt in plaats daarvan ‘Verouderde OpenAI-compatibele modus’ hierboven — streaming en toolaanroepen werken daar mogelijk niet tegelijk.
    </Tip>

  </Accordion>
</AccordionGroup>

## Problemen oplossen

<AccordionGroup>
  <Accordion title="WSL2-crashlus (herhaaldelijk opnieuw opstarten)">
    In WSL2 met NVIDIA/CUDA maakt het officiële Ollama-installatieprogramma voor Linux een
    `ollama.service`-systemd-eenheid met `Restart=always`. Als die service
    automatisch start en tijdens het opstarten van WSL2 een GPU-model laadt, kan Ollama tijdens
    het laden hostgeheugen vastzetten; Hyper-V-geheugenterugwinning kan die
    pagina's niet altijd terugwinnen, waardoor Windows de WSL2-VM kan beëindigen, systemd
    Ollama opnieuw start en de lus zich herhaalt.

    Aanwijzingen: herhaaldelijk opnieuw opstarten/beëindigen van WSL2, hoog CPU-gebruik in `app.slice` of
    `ollama.service` direct na het starten van WSL2 en SIGTERM van systemd in plaats
    van de Linux OOM-killer.

    OpenClaw registreert bij het opstarten een waarschuwing wanneer het WSL2 detecteert, `ollama.service`
    ingeschakeld is met `Restart=always` en CUDA-markeringen zichtbaar zijn.

    Mitigatie:

    ```bash
    sudo systemctl disable ollama
    ```

    Voeg aan de Windows-kant het volgende toe aan `%USERPROFILE%\.wslconfig` en voer daarna
    `wsl --shutdown` uit:

    ```ini
    [experimental]
    autoMemoryReclaim=disabled
    ```

    Of verkort keep-alive/start Ollama alleen handmatig wanneer dat nodig is:

    ```bash
    export OLLAMA_KEEP_ALIVE=5m
    ollama serve
    ```

    Zie [ollama/ollama#11317](https://github.com/ollama/ollama/issues/11317).

  </Accordion>

  <Accordion title="Ollama niet gedetecteerd">
    Controleer of Ollama actief is, `OLLAMA_API_KEY` (of een authenticatieprofiel) is ingesteld
    en `models.providers.ollama` **niet** expliciet is gedefinieerd:

    ```bash
    ollama serve
    curl http://localhost:11434/api/tags
    ```

  </Accordion>

  <Accordion title="Geen modellen beschikbaar">
    Haal het model lokaal op of definieer het expliciet in
    `models.providers.ollama`:

    ```bash
    ollama list  # Bekijk wat er is geïnstalleerd
    ollama pull gemma4
    ollama pull gpt-oss:20b
    ollama pull llama3.3     # Of een ander model
    ```

  </Accordion>

  <Accordion title="Verbinding geweigerd">
    ```bash
    # Controleer of Ollama actief is
    ps aux | grep ollama

    # Of start Ollama opnieuw
    ollama serve
    ```

  </Accordion>

  <Accordion title="Externe host werkt met curl, maar niet met OpenClaw">
    Controleer dit vanaf dezelfde machine en runtime waarop de Gateway wordt uitgevoerd:

    ```bash
    openclaw gateway status --deep
    curl http://ollama-host:11434/api/tags
    ```

    Veelvoorkomende oorzaken:

    - `baseUrl` verwijst naar `localhost`, maar de Gateway wordt uitgevoerd in Docker of op een andere host.
    - De URL gebruikt `/v1`, waardoor OpenAI-compatibel gedrag wordt geselecteerd in plaats van native Ollama.
    - Voor de externe host moeten de firewall of LAN-binding worden aangepast.
    - Het model staat op de daemon van je laptop, maar niet op de externe daemon.

  </Accordion>

  <Accordion title="Model geeft tool-JSON als tekst weer">
    Meestal bevindt de provider zich in de OpenAI-compatibele modus of kan het model
    geen toolschema's verwerken. Geef de voorkeur aan de native modus:

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://ollama-host:11434",
            api: "ollama",
          },
        },
      },
    }
    ```

    Als een klein lokaal model nog steeds niet met toolschema's overweg kan, stel dan
    `compat.supportsTools: false` in voor die modelvermelding en test opnieuw.

  </Accordion>

  <Accordion title="Kimi of GLM retourneert onleesbare symbolen">
    Gehoste Kimi/GLM-antwoorden die bestaan uit lange reeksen niet-talige symbolen worden
    behandeld als een mislukte provideraanroep en niet als een geslaagd antwoord, zodat
    de normale afhandeling voor opnieuw proberen, fallback en fouten het overneemt in plaats van
    beschadigde tekst in de sessie op te slaan.

    Als dit opnieuw gebeurt, leg dan de modelnaam en het huidige sessiebestand vast, evenals
    of de uitvoering `Cloud + Local` of `Cloud only` gebruikte, en probeer vervolgens een nieuwe
    sessie en een fallbackmodel:

    ```bash
    openclaw infer model run --model ollama/kimi-k2.5:cloud --prompt "Antwoord exact met: ok" --json
    openclaw models set ollama/gemma4
    ```

  </Accordion>

  <Accordion title="Koud lokaal model krijgt een time-out">
    Grote lokale modellen kunnen de eerste keer veel tijd nodig hebben om te laden. Beperk de time-out tot de
    Ollama-provider en houd het model eventueel tussen beurten geladen:

    ```json5
    {
      models: {
        providers: {
          ollama: {
            timeoutSeconds: 300,
            models: [
              {
                id: "gemma4:26b",
                name: "gemma4:26b",
                params: { keep_alive: "15m" },
              },
            ],
          },
        },
      },
    }
    ```

    Als de host zelf langzaam verbindingen accepteert, verlengt `timeoutSeconds` ook
    de beveiligde verbindingstime-out voor deze provider.

  </Accordion>

  <Accordion title="Model met grote context is te traag of heeft onvoldoende geheugen">
    Veel modellen adverteren contexten die groter zijn dan je hardware
    comfortabel kan uitvoeren. Native Ollama gebruikt zijn eigen standaardwaarde voor de runtime, tenzij
    `params.num_ctx` is ingesteld. Beperk zowel het budget van OpenClaw als de aanvraagcontext van Ollama
    voor een voorspelbare latentie tot het eerste token:

    ```json5
    {
      models: {
        providers: {
          ollama: {
            contextWindow: 32768,
            maxTokens: 8192,
            models: [
              {
                id: "qwen3.5:9b",
                name: "qwen3.5:9b",
                params: { num_ctx: 32768, thinking: false },
              },
            ],
          },
        },
      },
    }
    ```

    Verlaag `contextWindow` als OpenClaw te veel prompttekst verzendt. Verlaag
    `params.num_ctx` als de runtimecontext van Ollama te groot is voor de machine.
    Verlaag `maxTokens` als het genereren te lang duurt.

  </Accordion>
</AccordionGroup>

<Note>
Meer hulp: [Problemen oplossen](/nl/help/troubleshooting) en [Veelgestelde vragen](/nl/help/faq).
</Note>

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Ollama Cloud" href="/nl/providers/ollama-cloud" icon="cloud">
    Installatie uitsluitend voor de cloud met de speciale `ollama-cloud`-provider.
  </Card>
  <Card title="Modelproviders" href="/nl/concepts/model-providers" icon="layers">
    Overzicht van alle providers, modelverwijzingen en failovergedrag.
  </Card>
  <Card title="Modelselectie" href="/nl/concepts/models" icon="brain">
    Modellen kiezen en configureren.
  </Card>
  <Card title="Ollama Web Search" href="/nl/tools/ollama-search" icon="magnifying-glass">
    Volledige installatie- en gedragsdetails voor webzoekopdrachten via Ollama.
  </Card>
  <Card title="Configuratie" href="/nl/gateway/configuration" icon="gear">
    Volledige configuratiereferentie.
  </Card>
</CardGroup>
