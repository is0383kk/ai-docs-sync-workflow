---
read_when:
    - Een client ziet `rate limit exceeded for <method>`, `AUTH_RATE_LIMITED` of vergrendelingsfouten
    - Je wilt `gateway.auth.rateLimit` afstellen
    - Je denkt na over bescherming tegen brute-forceaanvallen op een blootgestelde Gateway
    - Je moet weten welke Gateway-oppervlakken worden beperkt en welke limieten daarvoor gelden
summary: 'Naslagwerk voor elke Gateway-snelheidslimiet: blokkeringen vóór authenticatie, throttling voor browsers en webhooks, de vangnetlimiet voor schrijfbewerkingen in het besturingsvlak, ACP-sessielimieten en de afkoelperiode voor opnieuw opstarten'
title: Snelheidsbeperking
x-i18n:
    generated_at: "2026-07-27T05:14:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7aa37b65347610bedfb1db8f661e7ba75ef3cdfed0ba73c4ce53d80acace1e48
    source_path: gateway/security/rate-limiting.md
    workflow: 16
---

De Gateway hanteert verschillende onafhankelijke snelheidslimieten. Deze beschermen verschillende
grenzen, gebruiken verschillende identiteiten als sleutel en geven verschillende foutvormen terug.
Deze pagina dient als naslagwerk voor al deze limieten.

In één oogopslag:

| Oppervlak                          | Limiet (standaard)                | Sleutel                           | Configureerbaar          |
| ---------------------------------- | --------------------------------- | --------------------------------- | ------------------------ |
| Mislukte authenticatie (token/wachtwoord/apparaat) | 10 mislukkingen / 60s, 5 min. blokkering | IP + bereik van aanmeldgegevens | `gateway.auth.rateLimit` |
| WS-authenticatiemislukkingen vanuit browsers | hetzelfde, loopback **niet** vrijgesteld | IP, of paginaoorsprong vanaf loopback | `gateway.auth.rateLimit` |
| Authenticatiemislukkingen voor Webhook (`/hooks`) | 20 mislukkingen / 60s, 60s blokkering | IP | nee |
| Schrijf-RPC's van het besturingsvlak | 30 verzoeken / 60s per methode | methode + apparaat + IP | nee |
| ACP-sessieaanmaak                  | 120 sessies / 10s                 | translator-instantie              | intern                   |
| Gateway-herstartcycli              | 30s afkoelperiode tussen herstarts | proces                            | nee                      |

## Authenticatiepogingen (vóór authenticatie)

Mislukte authenticatiepogingen worden per IP-adres van de client beperkt, voordat
verzoeken worden verwerkt. Dit vormt de beveiliging tegen brute-forceaanvallen voor openbaar toegankelijke Gateways.

- Alleen _onjuiste_ aanmeldgegevens tellen mee. Ontbrekende aanmeldgegevens (een client die nooit
  een token heeft verzonden) en geslaagde authenticaties verbruiken geen budget; een
  geslaagde authenticatie stelt de teller voor dat IP-adres opnieuw in.
- Standaardwaarden: 10 mislukkingen per 60 seconden, gevolgd door een blokkering van 5 minuten voor dat IP-adres.
- Loopback (`127.0.0.1` / `::1`) is standaard vrijgesteld, zodat lokale CLI-sessies
  niet kunnen worden geblokkeerd.
- Tellers zijn per klasse van aanmeldgegevens afgebakend, zodat een stortvloed tegen het ene oppervlak
  een ander oppervlak niet verdringt. De bereiken omvatten het gedeelde Gateway-
  token/wachtwoord, apparaattokens, Node-koppeling, hernieuwde goedkeuring van gekoppelde Nodes,
  bootstrap-tokens voor apparaten en de uitgifte van watchOS-challenges.

Tijdens een blokkering mislukken verbindingspogingen met:

```json
{
  "code": "INVALID_REQUEST",
  "message": "niet geautoriseerd: te veel mislukte authenticatiepogingen (probeer het later opnieuw)",
  "retryable": true,
  "retryAfterMs": 297000,
  "details": {
    "code": "AUTH_RATE_LIMITED",
    "authReason": "rate_limited",
    "recommendedNextStep": "wait_then_retry"
  }
}
```

Pogingen vanaf andere IP-adressen (inclusief loopback) worden tijdens een blokkering niet beïnvloed.

Pas dit aan onder `gateway.auth.rateLimit` in `openclaw.json`:

```json
{
  "gateway": {
    "auth": {
      "rateLimit": {
        "maxAttempts": 10,
        "windowMs": 60000,
        "lockoutMs": 300000,
        "exemptLoopback": true
      }
    }
  }
}
```

Herhaalde `AUTH_RATE_LIMITED`-vermeldingen in het Gateway-logboek betekenen dat iemand
aanmeldgegevens probeert te raden; zie het [draaiboek voor blootstelling](/nl/gateway/security/exposure-runbook).

### Verbindingen vanuit browsers

WebSocket-verbindingen die een browserheader `Origin` bevatten, gebruiken dezelfde
limieten, maar daarbij staat de loopbackvrijstelling **altijd uit** — een schadelijke pagina in
een lokale browser blijft een niet-vertrouwde client, dus localhost krijgt langs
dit pad geen vrijstelling. Wanneer zo'n verbinding _vanaf_ een loopbackadres binnenkomt, worden de
mislukkingen gekoppeld aan de genormaliseerde paginaoorsprong (bijvoorbeeld
`browser-origin:https://evil.example`) in plaats van aan het gedeelde loopback-IP-adres,
zodat elke oorsprong een eigen bucket krijgt; vanaf niet-loopbackadressen blijft de sleutel
het IP-adres van de client. Dit is niet configureerbaar.

### Webhooks

De HTTP-ingang `/hooks` heeft een eigen limiet voor mislukkingen: 20 mislukte
authenticaties per 60 seconden per IP-adres van de client, gevolgd door een blokkering van 60 seconden.
Loopback is niet vrijgesteld. Geslaagde hook-authenticatie stelt de teller opnieuw in. Beperkte
verzoeken ontvangen gewone HTTP `429 Too Many Requests` met een header `Retry-After`
(seconden). De limieten liggen vast; als een legitieme integratie deze bereikt,
corrigeer dan de aanmeldgegevens in plaats van steeds agressiever opnieuw te proberen.

## Schrijfbewerkingen van het besturingsvlak (vangnet na authenticatie)

Administratieve RPC's die schrijven (`config.apply`, `config.patch`, `plugins.install`,
`plugins.setEnabled`, `plugins.uninstall`, `update.run`, `worktrees.*`,
`gateway.restart.request`, ...) worden daarnaast **na**
autorisatie beperkt: 30 verzoeken per 60 seconden, per methode, per
`deviceId+clientIp`.

Dit is geen beveiligingsgrens — aanroepers beschikken al over `operator.admin` — maar
een vangnet dat uit de hand gelopen client- of agentlussen begrenst die kostbare
bewerkingen blijven bestoken. Interactief gebruik bereikt deze limiet nooit; elke methode heeft een eigen bucket, zodat
het in- of uitschakelen van een Plugin niet ten koste gaat van het budget voor configuratieschrijfbewerkingen.

Wanneer de limiet wordt overschreden, mislukt het verzoek met een fout die opnieuw kan worden geprobeerd:

```json
{
  "code": "UNAVAILABLE",
  "message": "snelheidslimiet voor config.patch overschreden; probeer het over 35s opnieuw",
  "retryable": true,
  "retryAfterMs": 34539,
  "details": { "method": "config.patch", "limit": "30 per 60s" }
}
```

Clients moeten `retryAfterMs` respecteren. De limiet ligt vast (niet configureerbaar);
buckets verlopen vanzelf en worden door Gateway-onderhoud opgeschoond.

## ACP-sessieaanmaak

De ACP-translator beperkt de sessieaanmaak tot 120 nieuwe sessies per venster van 10 seconden
per translator-instantie. Bij overschrijding mislukt het verzoek met een fout
waarvan het bericht de wachttijd bevat (dit pad heeft geen gestructureerd veld `retryAfterMs`):

```
Snelheidslimiet voor ACP-sessieaanmaak overschreden voor <method>; probeer het over <n>s opnieuw.
```

Dit begrenst uit de hand gelopen clients die in een lus sessies aanmaken; normaal gebruik door IDE's en
agents blijft hier ruim onder.

## Afkoelperiode voor herstarts

Gateway-herstartverzoeken worden samengevoegd en hanteren vervolgens een afkoelperiode van 30 seconden tussen
herstartcycli. Een herstart die tijdens de afkoelperiode wordt aangevraagd, wordt gepland nadat deze
is verstreken, in plaats van afgewezen. Dit staat los van de limiet voor het besturingsvlak
hierboven: `gateway.restart.request` verbruikt een budgetpositie van het besturingsvlak _en_
de resulterende herstart houdt zich aan de afkoelperiode.

## Operationele opmerkingen

- Alle begrenzers bevinden zich in het geheugen en gelden per proces; meerdere Gateways delen
  geen status. Door het Gateway-proces te vervangen, worden de tellers die eigendom zijn van de Gateway
  gewist (authenticatieblokkeringen, Webhook-beperking, buckets van het besturingsvlak). De
  afkoelperiode voor herstarts blijft bewust behouden tijdens herstartcycli binnen hetzelfde proces — dat is
  precies wat deze beperkt — en wordt alleen samen met het proces opnieuw ingesteld. De ACP-sessielimiet
  behoort toe aan de translator-instantie en wordt opnieuw ingesteld wanneer die instantie opnieuw wordt
  aangemaakt, niet wanneer de Gateway opnieuw wordt gestart.
- Bucketmaps zijn begrensd (harde limieten voor het aantal vermeldingen plus periodieke opschoning), zodat
  een stortvloed aan unieke sleutels het geheugengebruik niet onbeperkt kan laten groeien.
- Wanneer een client zich achter een reverse proxy bevindt, is het effectieve IP-adres het vastgestelde
  IP-adres van de client; zie [authenticatie via vertrouwde proxy's](/nl/gateway/trusted-proxy-auth) voor hoe
  proxyheaders worden gevalideerd voordat ze dit kunnen beïnvloeden.
- De signalering voor opnieuw proberen verschilt per oppervlak: Gateway-RPC-begrenzers retourneren
  `retryable: true` plus `retryAfterMs`, de Webhook-ingang gebruikt HTTP 429
  met een header `Retry-After`, en ACP neemt de wachttijd op in het foutbericht.
  Wacht in alle gevallen gedurende de aangegeven tijd voordat je het opnieuw probeert, in plaats van
  het onmiddellijk opnieuw te proberen.
