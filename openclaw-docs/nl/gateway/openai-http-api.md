---
read_when:
    - Tools integreren die OpenAI Chat Completions verwachten
summary: Stel vanuit de Gateway een OpenAI-compatibel HTTP-eindpunt `/v1/chat/completions` beschikbaar
title: OpenAI-chatvoltooiingen
x-i18n:
    generated_at: "2026-07-27T05:51:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4cc5a1a56972bb9070da0f8f60d6efd673cc1d1d516b730c55bc9d171fc7a5b3
    source_path: gateway/openai-http-api.md
    workflow: 16
---

De Gateway kan een klein OpenAI-compatibel Chat Completions-oppervlak aanbieden. Dit is **standaard uitgeschakeld**.

Na inschakeling worden al deze endpoints aangeboden op dezelfde poort als de Gateway (WS + HTTP-multiplexing):

| Methode | Pad                    |
| ------- | ---------------------- |
| POST    | `/v1/chat/completions` |
| GET     | `/v1/models`           |
| GET     | `/v1/models/{id}`      |
| POST    | `/v1/embeddings`       |
| POST    | `/v1/responses`        |

Verzoeken worden uitgevoerd als een normale Gateway-agentuitvoering (via hetzelfde codepad als `openclaw agent`), zodat routering, machtigingen en configuratie overeenkomen met jouw Gateway.

## Het endpoint inschakelen

```json5
{
  gateway: {
    http: {
      endpoints: {
        chatCompletions: { enabled: true },
      },
    },
  },
}
```

Stel `enabled: false` in (of laat dit weg) om het uit te schakelen.

## Beveiligingsgrens (belangrijk)

Behandel dit endpoint als **volledige operatortoegang** tot de Gateway-instantie:

- Een geldig Gateway-token/wachtwoord voor dit endpoint is gelijkwaardig aan een referentie voor een eigenaar/operator, niet aan een beperkt bereik per gebruiker.
- Verzoeken doorlopen hetzelfde agentpad in het besturingsvlak als vertrouwde operatoracties. Als het beleid van de doelagent gevoelige tools toestaat, kan dit endpoint deze dus gebruiken.
- Houd het uitsluitend op loopback/tailnet/privé-ingress. Stel het niet bloot aan het openbare internet.

Authenticatiematrix:

| Authenticatiepad                                                                                    | Gedrag                                                                                                                                                                                                                                                                                                    |
| ---------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `gateway.auth.mode="token"` of `"password"` + `Authorization: Bearer ...`                            | Bewijst het bezit van het gedeelde Gateway-geheim. Negeert elke `x-openclaw-scopes`-header en herstelt de volledige standaardset operatorbereiken: `operator.admin`, `operator.approvals`, `operator.pairing`, `operator.read`, `operator.talk.secrets`, `operator.write`. Behandelt chatbeurten als beurten van de eigenaar-afzender. |
| Vertrouwde identiteitsdragende HTTP (trusted-proxy-authenticatie of `gateway.auth.mode="none"` op privé-ingress) | Respecteert `x-openclaw-scopes` indien aanwezig; valt bij afwezigheid terug op de standaardset operatorbereiken. Verliest eigenaarsemantiek alleen wanneer de aanroeper de bereiken expliciet beperkt en `operator.admin` weglaat. Vereist `operator.admin` voor besturingselementen op eigenaarsniveau, zoals `x-openclaw-model`.                        |

Zie [Operatorbereiken](/nl/gateway/operator-scopes), [Beveiliging](/nl/gateway/security) en [Externe toegang](/nl/gateway/remote).

## Authenticatie

Gebruikt de authenticatieconfiguratie van de Gateway (zie [Trusted-proxy-authenticatie](/nl/gateway/trusted-proxy-auth) voor details over die modus):

| Modus                               | Authenticatiemethode                                                                                                                                                                    |
| ----------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `gateway.auth.mode="token"`         | `Authorization: Bearer <token>`. Stel dit in via `gateway.auth.token` of `OPENCLAW_GATEWAY_TOKEN`.                                                                                              |
| `gateway.auth.mode="password"`      | `Authorization: Bearer <password>`. Stel dit in via `gateway.auth.password` of `OPENCLAW_GATEWAY_PASSWORD`.                                                                                     |
| `gateway.auth.mode="trusted-proxy"` | Routeer via de geconfigureerde identiteitsbewuste proxy; deze injecteert de vereiste identiteitsheaders. Loopbackproxy's op dezelfde host vereisen expliciet `gateway.auth.trustedProxy.allowLoopback = true`. |
| `gateway.auth.mode="none"`          | Geen authenticatieheader vereist (alleen privé-ingress).                                                                                                                               |

Opmerkingen:

- Aanroepers op dezelfde host die de proxy op een `trusted-proxy`-Gateway omzeilen, kunnen rechtstreeks terugvallen op `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD`. Bewijs in een `Forwarded`-, `X-Forwarded-*`- of `X-Real-IP`-header houdt het verzoek in plaats daarvan op het trusted-proxy-pad.
- Als `gateway.auth.rateLimit` is geconfigureerd en te veel authenticatiepogingen mislukken, retourneert het endpoint `429` met een `Retry-After`-header.

## Wanneer je dit endpoint gebruikt

- Geef hieraan de voorkeur boven het toevoegen van een nieuw ingebouwd kanaal wanneer je integratie slechts een ander operator-/clientoppervlak voor dezelfde Gateway is.
- Geef voor native mobiele clients die rechtstreeks met een externe Gateway verbinden de voorkeur aan [WebChat](/nl/web/webchat) of het [Gateway-protocol](/nl/gateway/protocol) met de bootstrap-/apparaat-tokenflow voor gekoppelde apparaten, zodat het apparaat geen gedeeld HTTP-token/wachtwoord nodig heeft.
- Bouw in plaats daarvan een kanaalplugin wanneer je een extern berichtenplatform integreert met eigen gebruikers, ruimtes, Webhook-bezorging of uitgaand transport. Zie [Plugins bouwen](/nl/plugins/building-plugins).

## Agent-eerst-modelcontract

OpenClaw behandelt het OpenAI-veld `model` als een **agentdoel**, niet als een onbewerkte model-id van een provider.

| Waarde van `model`                    | Routeert naar                                                                                                                  |
| -------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `openclaw`                                   | Geconfigureerde standaardagent                                                                                                 |
| `openclaw/default`                           | Geconfigureerde standaardagent (stabiele alias; kan veilig hardgecodeerd worden, zelfs als de werkelijke id van de standaardagent tussen omgevingen verandert) |
| `openclaw/<agentId>` of `openclaw:<agentId>` | Specifieke agent                                                                                                           |
| `agent:<agentId>`                            | Specifieke agent (compatibiliteitsalias)                                                                                     |

Optionele verzoekheaders:

| Header                                          | Effect                                                                                                                                                                                                                                                                      |
| ----------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `x-openclaw-model: <provider/model-or-bare-id>` | Overschrijft het backendmodel voor de geselecteerde agent. Bearer-aanroepers met een gedeeld geheim kunnen dit rechtstreeks gebruiken; identiteitsdragende aanroepers (trusted-proxy of privé-ingress zonder authenticatie met `x-openclaw-scopes`) hebben `operator.admin` nodig, anders `403 missing scope: operator.admin`. |
| `x-openclaw-agent-id: <agentId>`                | Compatibiliteitsoverschrijving voor agentselectie.                                                                                                                                                                                                                                 |
| `x-openclaw-session-key: <sessionKey>`          | Expliciete sessieroutering. Wordt geweigerd met `400 invalid_request_error` als deze een gereserveerde interne naamruimte gebruikt (`subagent:`, `cron:`, `acp:`).                                                                                                                                |
| `x-openclaw-message-channel: <channel>`         | Stelt de synthetische context van het ingresskanaal in voor kanaalbewuste prompts/beleidsregels.                                                                                                                                                                                              |

`/v1/models` vermeldt agentdoelen op het hoogste niveau (`openclaw`, `openclaw/default`, `openclaw/<agentId>`), geen backendprovidermodellen en geen subagents; subagents blijven onderdeel van de interne uitvoeringstopologie. Als je `x-openclaw-model` weglaat, wordt de geselecteerde agent uitgevoerd met het normaal geconfigureerde model.

`/v1/embeddings` gebruikt dezelfde agentdoel-id's voor `model`. Stuur `x-openclaw-model` (vanaf een aanroeper met een gedeeld geheim of een identiteitsdragende aanroeper met `operator.admin`) om een specifiek embeddingmodel te kiezen; anders gebruikt het verzoek de normale embeddingconfiguratie van de geselecteerde agent.

## Sessiegedrag

Het endpoint is standaard **statusloos per verzoek** (voor elke aanroep wordt een nieuwe sessiesleutel gegenereerd).

Als het verzoek een OpenAI-tekenreeks `user` bevat, leidt de Gateway daaruit een stabiele sessiesleutel af, zodat herhaalde aanroepen een agentsessie kunnen delen. Gebruik voor aangepaste apps per gespreksthread dezelfde waarde voor `user`; vermijd identifiers op accountniveau, tenzij je wilt dat meerdere gesprekken/apparaten één OpenClaw-sessie delen. Gebruik `x-openclaw-session-key` alleen wanneer je expliciete routeringscontrole over meerdere clients/threads nodig hebt, met sleutels die door de toepassing worden beheerd en de bovenstaande gereserveerde naamruimtes vermijden.

## Verzoeklimieten

Het endpoint gebruikt ingebouwde limieten van 20 MB per verzoekbody, 8 `image_url`-onderdelen
uit het meest recente gebruikersbericht en 20 MB aan cumulatieve gedecodeerde
afbeeldingsgegevens. Het beleid voor afbeeldingsbronnen blijft configureerbaar onder
`gateway.http.endpoints.chatCompletions.images`:

```json5
{
  gateway: {
    http: {
      endpoints: {
        chatCompletions: {
          enabled: true,
          images: {
            allowUrl: false,
            urlAllowlist: ["cdn.example.com", "*.assets.example.com"],
            allowedMimes: [
              "image/jpeg",
              "image/png",
              "image/gif",
              "image/webp",
              "image/heic",
              "image/heif",
            ],
            maxBytes: 10485760,
            maxRedirects: 3,
            timeoutMs: 10000,
          },
        },
      },
    },
  },
}
```

De standaardwaarden voor afbeeldingsinstellingen zijn:

| Sleutel                | Standaardwaarde                                                     |
| --------------------- | ------------------------------------------------------------------- |
| `images.allowUrl`     | `false` (`image_url`-onderdelen afkomstig van URL's worden geweigerd, tenzij dit is ingeschakeld) |
| `images.maxBytes`     | 10MB per afbeelding                                                 |
| `images.maxRedirects` | 3                                                                   |
| `images.timeoutMs`    | 10s                                                                 |

HEIC/HEIF-bronnen voor `image_url` worden geaccepteerd en vóór levering aan de provider genormaliseerd naar JPEG via de gedeelde OpenClaw-afbeeldingsprocessor (Rastermill), die voor indelingen waarvoor externe codec-ondersteuning nodig is terugvalt op een systeemconverter (`sips`, ImageMagick, GraphicsMagick of ffmpeg).

Beveiligingsopmerking: het toestaan van een hostnaam via een allowlist omzeilt de blokkering van privé-/interne IP-adressen niet. Pas voor Gateways die aan internet zijn blootgesteld naast beveiligingen op toepassingsniveau ook netwerkcontroles voor uitgaand verkeer toe. Zie [Beveiliging](/nl/gateway/security).

## Contract voor chattools

`/v1/chat/completions` ondersteunt een subset van functietools die compatibel is met gangbare OpenAI Chat-clients.

### Ondersteunde verzoekvelden

| Veld                       | Opmerkingen                                                                                                                                   |
| -------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `tools`                    | Array van `{ "type": "function", "function": { ... } }`                                                                                     |
| `tool_choice`              | `"auto"`, `"none"`, `"required"` of `{ "type": "function", "function": { "name": "..." } }`                                                  |
| `messages[*].role: "tool"` | Vervolgbeurten                                                                                                                                |
| `messages[*].tool_call_id` | Koppelt een toolresultaat terug aan een eerdere toolaanroep                                                                                   |
| `max_completion_tokens`    | Getal; limiet per aanroep voor het totale aantal voltooiingstokens (inclusief redeneertokens). Huidige veldnaam; wordt gebruikt wanneer zowel dit veld als `max_tokens` worden verzonden. |
| `max_tokens`               | Getal; verouderde alias, genegeerd wanneer `max_completion_tokens` ook aanwezig is.                                                           |
| `temperature`              | Getal 0-2; op basis van beste inspanning doorgestuurd naar de upstreamprovider. `400 invalid_request_error` indien buiten bereik.             |
| `top_p`                    | Getal 0-1; op basis van beste inspanning. `400 invalid_request_error` indien buiten bereik.                                                    |
| `frequency_penalty`        | Getal -2.0 tot 2.0; op basis van beste inspanning. `400 invalid_request_error` indien buiten bereik.                                          |
| `presence_penalty`         | Getal -2.0 tot 2.0; op basis van beste inspanning. `400 invalid_request_error` indien buiten bereik.                                          |
| `seed`                     | Geheel getal; op basis van beste inspanning. `400 invalid_request_error` voor niet-gehele waarden.                                           |
| `stop`                     | Tekenreeks of array van maximaal 4 tekenreeksen; op basis van beste inspanning. `400 invalid_request_error` voor meer dan 4 reeksen of items die geen tekenreeks zijn of leeg zijn. |

Alle velden voor sampling en tokenlimieten gebruiken hetzelfde kanaal voor streamparameters van de agent en worden op basis van beste inspanning doorgestuurd:

- Tokenlimiet: de veldnaam op de verbinding wordt gekozen door het providertransport: `max_completion_tokens` voor eindpunten uit de OpenAI-familie, `max_tokens` voor providers die alleen de verouderde naam accepteren (Mistral, Chutes).
- `stop` wordt gekoppeld aan het stopveld van het transport: `stop` voor Chat Completions-backends, `stop_sequences` voor Anthropic. De OpenAI Responses-API heeft geen stopparameter, dus `stop` wordt niet toegepast op modellen die door Responses worden ondersteund.
- De op ChatGPT gebaseerde Codex Responses-backend gebruikt vaste sampling aan de serverzijde en verwijdert `temperature`/`top_p` (samen met `max_output_tokens`, `metadata`, `prompt_cache_retention`, `service_tier`) voordat de aanvraag die backend bereikt.

### Niet-ondersteunde varianten

Retourneert `400 invalid_request_error` voor:

- `tools` die geen array is, toolitems die geen functie zijn, of ontbrekende `tool.function.name`
- `tool_choice`-varianten zoals `allowed_tools` en `custom`
- `tool_choice.function.name`-waarden die niet overeenkomen met een opgegeven tool

Voor `tool_choice: "required"` en aan een functie vastgezette `tool_choice` beperkt het eindpunt de aan de client beschikbaar gestelde set functietools, instrueert het de runtime om een clienttool aan te roepen voordat deze antwoordt, en geeft het een fout als het antwoord van de agent geen overeenkomende gestructureerde clienttoolaanroep bevat. Dit geldt voor de door de aanroeper opgegeven HTTP-lijst `tools`, niet voor elke interne agenttool van OpenClaw.

### Vorm van niet-streamend toolantwoord

Wanneer de agent tools aanroept, gebruikt het antwoord:

- `choices[0].finish_reason = "tool_calls"`
- `choices[0].message.tool_calls[]`-items met `id`, `type: "function"`, `function.name`, `function.arguments` (JSON-tekenreeks)
- Commentaar van de assistent vóór de toolaanroep, in `choices[0].message.content` (mogelijk leeg)

### Vorm van streamend toolantwoord

Wanneer `stream: true`, komen toolaanroepen binnen als incrementele SSE-fragmenten: een initiële delta voor de assistentrol, optionele delta's met assistentcommentaar, een of meer `delta.tool_calls`-fragmenten met de toolidentiteit en argumentfragmenten, gevolgd door een laatste fragment met `finish_reason: "tool_calls"` en `data: [DONE]`.

Als `stream_options.include_usage=true`, wordt vóór `[DONE]` een afsluitend gebruiksfragment uitgezonden.

### Vervolglus voor tools

Voer na ontvangst van `tool_calls` de aangevraagde functie(s) uit en verzend een vervolgaanvraag die het eerdere bericht met de toolaanroep van de assistent bevat, plus een of meer `role: "tool"`-berichten met overeenkomende `tool_call_id`. Hierdoor wordt dezelfde redeneerlus van de agent voortgezet om het definitieve antwoord te produceren.

## Streaming (SSE)

Stel `stream: true` in om Server-Sent Events te ontvangen:

- `Content-Type: text/event-stream`
- Elke gebeurtenisregel is `data: <json>`
- De stream eindigt met `data: [DONE]`

## Snelle installatie van Open WebUI

- Basis-URL: `http://127.0.0.1:18789/v1`
- Basis-URL voor Docker op macOS: `http://host.docker.internal:18789/v1`
- API-sleutel: je bearer-token voor de Gateway
- Model: `openclaw/default`

Verwacht gedrag: `GET /v1/models` vermeldt `openclaw/default`, en Open WebUI gebruikt dit als de id van het chatmodel. Stel voor een specifieke backendprovider of een specifiek model het normale standaardmodel van de agent in, of verzend `x-openclaw-model` (aanroeper met gedeeld geheim, of aanroeper met identiteit en `operator.admin`).

Snelle rooktest:

```bash
curl -sS http://127.0.0.1:18789/v1/models \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

Als dit `openclaw/default` retourneert, kunnen de meeste Open WebUI-installaties verbinding maken met dezelfde basis-URL en hetzelfde token.

## Voorbeelden

Stabiele sessie voor één appgesprek:

```bash
curl -sS http://127.0.0.1:18789/v1/chat/completions \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "openclaw/default",
    "user": "conv:YOUR_CONVERSATION_ID",
    "messages": [{"role":"user","content":"Vat mijn taken voor vandaag samen"}]
  }'
```

Gebruik bij latere aanroepen voor dat gesprek opnieuw dezelfde waarde voor `user` om dezelfde agentsessie voort te zetten.

Niet-streamend:

```bash
curl -sS http://127.0.0.1:18789/v1/chat/completions \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "openclaw/default",
    "messages": [{"role":"user","content":"hoi"}]
  }'
```

Streamend:

```bash
curl -N http://127.0.0.1:18789/v1/chat/completions \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -H 'x-openclaw-model: openai/gpt-5.4' \
  -d '{
    "model": "openclaw/research",
    "stream": true,
    "messages": [{"role":"user","content":"hoi"}]
  }'
```

Modellen weergeven:

```bash
curl -sS http://127.0.0.1:18789/v1/models \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

Eén model ophalen:

```bash
curl -sS http://127.0.0.1:18789/v1/models/openclaw%2Fdefault \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

Embeddings maken:

```bash
curl -sS http://127.0.0.1:18789/v1/embeddings \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -H 'x-openclaw-model: openai/text-embedding-3-small' \
  -d '{
    "model": "openclaw/default",
    "input": ["alpha", "beta"]
  }'
```

`/v1/embeddings` ondersteunt `input` als tekenreeks of array van tekenreeksen.

## Gerelateerd

- [Configuratiereferentie](/nl/gateway/configuration-reference)
- [Operatorbereiken](/nl/gateway/operator-scopes)
- [OpenAI](/nl/providers/openai)
