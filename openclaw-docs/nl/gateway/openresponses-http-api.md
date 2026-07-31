---
read_when:
    - Clients integreren die de OpenResponses-API gebruiken
    - Je wilt itemgebaseerde invoer, clienttoolaanroepen of SSE-gebeurtenissen
summary: Stel vanuit de Gateway een OpenResponses-compatibel HTTP-eindpunt `/v1/responses` beschikbaar
title: OpenResponses-API
x-i18n:
    generated_at: "2026-07-27T06:15:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5bfd6ca3bf0cecd761fde865b41a95cff3fc5681f74f31b3adae5cd2e0b0be95
    source_path: gateway/openresponses-http-api.md
    workflow: 16
---

De Gateway kan een met OpenResponses compatibel `POST /v1/responses`-eindpunt aanbieden. Het is **standaard uitgeschakeld** en deelt zijn poort met de Gateway (WS + HTTP-multiplexing): `http://<gateway-host>:<port>/v1/responses`.

Verzoeken worden uitgevoerd als een normale agentrun van de Gateway (hetzelfde codepad als `openclaw agent`), zodat routering, machtigingen en configuratie overeenkomen met jouw Gateway.

Schakel het in of uit met `gateway.http.endpoints.responses.enabled`. Wanneer het is ingeschakeld, biedt hetzelfde compatibiliteitsoppervlak ook `GET /v1/models`, `GET /v1/models/{id}`, `POST /v1/embeddings` en `POST /v1/chat/completions` aan.

## Authenticatie, beveiliging en routering

Het operationele gedrag komt overeen met [OpenAI Chat Completions](/nl/gateway/openai-http-api):

- Het authenticatiepad komt overeen met `gateway.auth.mode`: gedeeld geheim (`token`/`password`) gebruikt `Authorization: Bearer <token-or-password>`; trusted-proxy gebruikt proxyheaders met identiteitsgegevens (loopbackproxy's op dezelfde host vereisen `gateway.auth.trustedProxy.allowLoopback = true`, met een directe terugval op dezelfde host via `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD` wanneer er geen `Forwarded`-/`X-Forwarded-*`-/`X-Real-IP`-header aanwezig is); `none` op privé-ingress vereist geen authenticatieheader. Zie [Authenticatie via vertrouwde proxy](/nl/gateway/trusted-proxy-auth).
- Behandel het eindpunt als volledige operatortoegang tot de Gateway-instantie.
- Authenticatiemodi met een gedeeld geheim negeren een beperktere, via bearer opgegeven `x-openclaw-scopes` en herstellen de volledige standaardset operatorbereiken: `operator.admin`, `operator.approvals`, `operator.pairing`, `operator.read`, `operator.talk.secrets`, `operator.write`. Chatbeurten op dit eindpunt worden behandeld als beurten van de eigenaar als afzender.
- Vertrouwde HTTP-modi met identiteitsgegevens (trusted-proxy of `gateway.auth.mode="none"`) respecteren `x-openclaw-scopes` indien aanwezig en vallen anders terug op de standaardset operatorbereiken. Eigenaarssemantiek gaat alleen verloren wanneer de aanroeper de bereiken expliciet beperkt en `operator.admin` weglaat.
- Selecteer agents met `model: "openclaw"`, `"openclaw/default"`, `"openclaw/<agentId>"` of de header `x-openclaw-agent-id`.
- Gebruik `x-openclaw-model` om het backendmodel van de geselecteerde agent te overschrijven (vereist `operator.admin` op authenticatiepaden met identiteitsgegevens).
- Gebruik `x-openclaw-session-key` voor expliciete sessieroutering (wordt geweigerd met `400 invalid_request_error` als het een gereserveerde naamruimte gebruikt: `subagent:`, `cron:`, `acp:`).
- Gebruik `x-openclaw-message-channel` voor een niet-standaard context van een synthetisch ingresskanaal.

Zie voor de canonieke uitleg van agentdoelmodellen, `openclaw/default`, doorvoer van embeddings en overschrijvingen van backendmodellen [OpenAI Chat Completions](/nl/gateway/openai-http-api#agent-first-model-contract).

Zie [Operatorbereiken](/nl/gateway/operator-scopes) en [Beveiliging](/nl/gateway/security).

## Sessiegedrag

Standaard is het eindpunt **toestandsloos per verzoek** (bij elke aanroep wordt een nieuwe sessiesleutel gegenereerd).

Als het verzoek een OpenResponses-tekenreeks `user` bevat, leidt de Gateway daaruit een stabiele sessiesleutel af, zodat herhaalde aanroepen een agentsessie kunnen delen.

`previous_response_id` hergebruikt de sessie van het eerdere antwoord wanneer het verzoek binnen hetzelfde bereik van agent/gebruiker/aangevraagde sessie blijft (overeenkomst op basis van authenticatieonderwerp, agent-id en `x-openclaw-session-key`).

## Verzoekstructuur

| Veld                                                             | Ondersteuning                                                                                                                    |
| ---------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `input`                                               | Tekenreeks of array van itemobjecten.                                                                                            |
| `instructions`                                               | Wordt samengevoegd met de systeemprompt.                                                                                         |
| `tools`                                               | Tooldefinities van de client (functietools).                                                                                     |
| `tool_choice`                                               | `"auto"`, `"none"`, `"required"` of `{ "type": "function", "name": "..." }` om clienttools te filteren of verplicht te stellen. |
| `stream`                                               | Schakelt SSE-streaming in.                                                                                                       |
| `max_output_tokens`                                               | Uitvoerlimiet op basis van beste inspanning (afhankelijk van provider).                                                          |
| `temperature`                                               | Samplingtemperatuur op basis van beste inspanning. Wordt genegeerd door de op ChatGPT gebaseerde Codex Responses-backend, die vaste sampling aan de serverzijde gebruikt. |
| `top_p`                                               | Nucleus-sampling op basis van beste inspanning. Dezelfde kanttekening voor Codex Responses als bij `temperature`.            |
| `user`                                               | Stabiele sessieroutering.                                                                                                        |
| `previous_response_id`                                               | Sessiecontinuïteit (zie hierboven).                                                                                              |
| `max_tool_calls`, `reasoning`, `metadata`, `store`, `truncation` | Worden geaccepteerd, maar momenteel genegeerd.                                                                                   |

## Items (invoer)

### `message`

Rollen: `system`, `developer`, `user`, `assistant`.

- `system` en `developer` worden aan de systeemprompt toegevoegd.
- Het meest recente item `user` of `function_call_output` wordt het 'huidige bericht'.
- Eerdere berichten van de gebruiker/assistent worden als geschiedenis voor context opgenomen.

### `function_call_output` (tools per beurt)

Stuur toolresultaten terug naar het model:

```json
{
  "type": "function_call_output",
  "call_id": "call_123",
  "output": "{\"temperature\": \"72F\"}"
}
```

### `reasoning` en `item_reference`

Worden geaccepteerd voor schemacompatibiliteit, maar genegeerd bij het opbouwen van de prompt.

## Tools (functietools aan de clientzijde)

Geef tools op met `tools: [{ type: "function", name, description?, parameters? }]`.

Als de agent een tool aanroept, retourneert het antwoord een uitvoeritem `function_call`. Stuur een vervolgverzoek met `function_call_output` om de beurt voort te zetten.

Voor `tool_choice: "required"` en een aan een functie vastgezette `tool_choice` beperkt het eindpunt de beschikbare set clientfunctietools, instrueert het de runtime om vóór het antwoorden een clienttool aan te roepen en weigert het de beurt als deze geen overeenkomende gestructureerde clienttoolaanroep bevat, in overeenstemming met het contract `/v1/chat/completions`. Niet-streamende verzoeken retourneren `502` met een `api_error`; streamende verzoeken zenden een gebeurtenis `response.failed` uit.

## Afbeeldingen (`input_image`)

Ondersteunt base64- of URL-bronnen:

```json
{
  "type": "input_image",
  "source": { "type": "url", "url": "https://example.com/image.png" }
}
```

Toegestane MIME-typen (standaard): `image/jpeg`, `image/png`, `image/gif`, `image/webp`, `image/heic`, `image/heif`. Maximale grootte (standaard): 10MB.

## Bestanden (`input_file`)

Ondersteunt base64- of URL-bronnen:

```json
{
  "type": "input_file",
  "source": {
    "type": "base64",
    "media_type": "text/plain",
    "data": "SGVsbG8gV29ybGQh",
    "filename": "hello.txt"
  }
}
```

Toegestane MIME-typen (standaard): `text/plain`, `text/markdown`, `text/html`, `text/csv`, `application/json`, `application/pdf`. Maximale grootte (standaard): 5MB.

Huidig gedrag:

- De bestandsinhoud wordt gedecodeerd en aan de **systeemprompt** toegevoegd, niet aan het gebruikersbericht, zodat deze tijdelijk blijft (niet opgeslagen in de sessiegeschiedenis).
- Gedecodeerde bestandstekst wordt verpakt als **niet-vertrouwde externe inhoud** voordat deze wordt toegevoegd, zodat bestandsbytes als gegevens worden behandeld en niet als vertrouwde instructies. Het ingevoegde blok gebruikt expliciete grensmarkeringen (`<<<EXTERNAL_UNTRUSTED_CONTENT id="...">>>` / `<<<END_EXTERNAL_UNTRUSTED_CONTENT id="...">>>`) en een metadataregel `Source: External`. De lange banner `SECURITY NOTICE:` wordt opzettelijk weggelaten om het promptbudget te behouden; de grensmarkeringen en metadata blijven van toepassing.
- PDF's worden eerst geparseerd om tekst te extraheren. Als er weinig tekst wordt gevonden, worden de eerste pagina's gerasterd tot afbeeldingen en aan het model doorgegeven, en gebruikt het ingevoegde bestandsblok de tijdelijke aanduiding `[PDF content rendered to images]`.

PDF-parsing wordt geleverd door de gebundelde Plugin `document-extract`, die `clawpdf` en de meegeleverde PDFium WebAssembly-runtime gebruikt voor tekstextractie en paginarendering.

Standaardinstellingen voor ophalen via URL:

- `files.allowUrl`: `true`
- `images.allowUrl`: `true`
- `maxUrlParts`: `8` (totaal aantal op URL gebaseerde onderdelen `input_file` + `input_image` per verzoek)
- Verzoeken worden beveiligd (DNS-resolutie, blokkering van privé-IP-adressen, omleidingslimieten, time-outs).
- Optionele toelatingslijsten voor hostnamen worden per invoertype ondersteund (`files.urlAllowlist`, `images.urlAllowlist`): exacte host (`"cdn.example.com"`) of subdomeinen met jokertekens (`"*.assets.example.com"`, komt niet overeen met het hoofddomein). Lege of weggelaten toelatingslijsten betekenen dat er geen beperking via een toelatingslijst voor hostnamen geldt.
- Stel `files.allowUrl: false` en/of `images.allowUrl: false` in om ophalen via URL volledig uit te schakelen.

## Limieten voor bestanden en afbeeldingen

Het eindpunt gebruikt een ingebouwde limiet van 20 MB voor de verzoekbody. Het bronbeleid voor bestanden en afbeeldingen
blijft configureerbaar onder `gateway.http.endpoints.responses`:

```json5
{
  gateway: {
    http: {
      endpoints: {
        responses: {
          enabled: true,
          maxUrlParts: 8,
          files: {
            allowUrl: true,
            urlAllowlist: ["cdn.example.com", "*.assets.example.com"],
            allowedMimes: [
              "text/plain",
              "text/markdown",
              "text/html",
              "text/csv",
              "application/json",
              "application/pdf",
            ],
            maxBytes: 5242880,
            maxChars: 60000,
            maxRedirects: 3,
            timeoutMs: 10000,
            pdf: {
              maxPages: 4,
              maxPixels: 4000000,
              minTextChars: 200,
            },
          },
          images: {
            allowUrl: true,
            urlAllowlist: ["images.example.com"],
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

Standaardwaarden indien weggelaten:

| Sleutel                   | Standaard  |
| ------------------------- | ---------- |
| `maxUrlParts`        | 8          |
| `files.maxBytes`        | 5MB        |
| `files.maxChars`        | 60k        |
| `files.maxRedirects`        | 3          |
| `files.timeoutMs`        | 10s        |
| `files.pdf.maxPages`        | 4          |
| `files.pdf.maxPixels`        | 4,000,000  |
| `files.pdf.minTextChars`        | 200        |
| `images.maxBytes`        | 10MB       |
| `images.maxRedirects`        | 3          |
| `images.timeoutMs`        | 10s        |

HEIC/HEIF `input_image`-bronnen worden genormaliseerd naar JPEG voordat ze via de gedeelde OpenClaw-beeldprocessor (Rastermill) aan de provider worden geleverd. Deze valt voor indelingen die externe codec-ondersteuning vereisen terug op een systeemconverter (`sips`, ImageMagick, GraphicsMagick of ffmpeg).

Beveiligingsopmerking: URL-toegestane lijsten worden vóór het ophalen en bij elke omleidingsstap afgedwongen. Het toestaan van een hostnaam omzeilt de blokkering van privé-/interne IP-adressen niet. Pas voor gateways die via internet bereikbaar zijn naast beveiligingen op applicatieniveau ook netwerkuitgaandverkeercontroles toe. Zie [Beveiliging](/nl/gateway/security).

## Streaming (SSE)

Stel `stream: true` in om Server-Sent Events te ontvangen:

- `Content-Type: text/event-stream`
- Elke gebeurtenisregel is `event: <type>` en `data: <json>`
- De stream eindigt met `data: [DONE]`

Gebeurtenistypen die momenteel worden verzonden: `response.created`, `response.in_progress`, `response.output_item.added`, `response.content_part.added`, `response.output_text.delta`, `response.output_text.done`, `response.content_part.done`, `response.output_item.done`, `response.completed`, `response.failed` (bij een fout).

## Gebruik

`usage` wordt ingevuld wanneer de onderliggende provider aantallen tokens rapporteert. OpenClaw normaliseert gangbare aliassen in OpenAI-stijl voordat deze tellers de onderliggende status-/sessieoppervlakken bereiken, waaronder `input_tokens` / `output_tokens` en `prompt_tokens` / `completion_tokens`.

## Fouten

Fouten gebruiken een JSON-object zoals:

```json
{ "error": { "message": "...", "type": "invalid_request_error" } }
```

Veelvoorkomende gevallen: `400` ongeldige aanvraagbody, `401` ontbrekende/ongeldige authenticatie, `403` ontbrekend operatorbereik, `405` verkeerde methode, `429` te veel mislukte authenticatiepogingen (met `Retry-After`).

## Voorbeelden

Zonder streaming:

```bash
curl -sS http://127.0.0.1:18789/v1/responses \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -H 'x-openclaw-agent-id: main' \
  -d '{
    "model": "openclaw",
    "input": "hi"
  }'
```

Met streaming:

```bash
curl -N http://127.0.0.1:18789/v1/responses \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -H 'x-openclaw-agent-id: main' \
  -d '{
    "model": "openclaw",
    "stream": true,
    "input": "hi"
  }'
```

## Gerelateerd

- [OpenAI-chatvoltooiingen](/nl/gateway/openai-http-api)
- [Operatorbereiken](/nl/gateway/operator-scopes)
- [OpenAI](/nl/providers/openai)
