---
read_when:
    - Hosttooling bouwen die de Gateway WebSocket-RPC-client niet kan gebruiken
    - Gateway-beheerautomatisering beschikbaar maken achter een privé-ingang die wordt vertrouwd
    - Het beveiligingsmodel voor HTTP-toegang tot Gateway-methoden controleren
summary: Stel geselecteerde methoden van het Gateway-besturingsvlak beschikbaar via de gebundelde, optionele admin-http-rpc-plugin
title: Beheer-HTTP-RPC-plugin
x-i18n:
    generated_at: "2026-07-27T05:55:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0709081efd0ce65cef7edac54df9a71978cbad17e2b25df83ac9075de938376c
    source_path: plugins/admin-http-rpc.md
    workflow: 16
---

De gebundelde `admin-http-rpc`-Plugin stelt via HTTP een allowlist van Gateway-controlplanemethoden beschikbaar voor vertrouwde hostautomatisering die geen Gateway WebSocket-verbinding open kan houden.

De Plugin wordt met OpenClaw meegeleverd, maar is standaard uitgeschakeld; wanneer deze is uitgeschakeld, wordt de route niet geregistreerd. Wanneer deze is ingeschakeld, voegt de Plugin `POST /api/v1/admin/rpc` toe aan dezelfde listener als de Gateway (`http://<gateway-host>:<port>/api/v1/admin/rpc`).

Schakel de Plugin alleen in voor privétools op de host, automatisering binnen een tailnet of een vertrouwde interne ingress. Stel deze route nooit rechtstreeks bloot aan het openbare internet.

## Voordat je de Plugin inschakelt

Admin HTTP RPC is een volledig controlplane-oppervlak voor operators: elke aanroeper die de HTTP-authenticatie van de Gateway doorstaat, kan de onderstaande methoden uit de allowlist aanroepen. Schakel de Plugin alleen in als aan al deze voorwaarden is voldaan:

- De aanroeper wordt vertrouwd om de Gateway te beheren.
- De aanroeper kan de WebSocket RPC-client niet gebruiken.
- De route is alleen bereikbaar via loopback, een tailnet of een privé-ingress met authenticatie.
- Je hebt de toegestane methoden gecontroleerd en ze komen overeen met de automatisering die je wilt uitvoeren.

Gebruik in plaats daarvan WebSocket RPC voor OpenClaw-clients en interactieve tools die een Gateway WebSocket-verbinding open kunnen houden.

## Inschakelen

Schakel de gebundelde Plugin in:

<Tabs>
  <Tab title="CLI">
    ```bash
    openclaw plugins enable admin-http-rpc
    openclaw gateway restart
    ```
  </Tab>
  <Tab title="Configuratie">
    ```json5
    {
      plugins: {
        entries: {
          "admin-http-rpc": { enabled: true },
        },
      },
    }
    ```
  </Tab>
</Tabs>

De route wordt tijdens het opstarten van de Plugin geregistreerd. Start de Gateway daarom opnieuw nadat je de Plugin-configuratie hebt gewijzigd.

Schakel de Plugin uit wanneer je het HTTP-oppervlak niet meer nodig hebt:

```bash
openclaw plugins disable admin-http-rpc
openclaw gateway restart
```

## De route verifiëren

Gebruik `health` als het kleinste veilige verzoek:

```bash
curl -sS http://<gateway-host>:<port>/api/v1/admin/rpc \
  -H 'Authorization: Bearer <gateway-token>' \
  -H 'Content-Type: application/json' \
  -d '{"method":"health","params":{}}'
```

Een geslaagd antwoord bevat `ok: true`:

```json
{
  "id": "generated-request-id",
  "ok": true,
  "payload": {
    "status": "ok"
  }
}
```

Wanneer de Plugin is uitgeschakeld, retourneert de route `404` omdat deze niet is geregistreerd.

## Authenticatie

De route van de Plugin gebruikt de HTTP-authenticatie van de Gateway.

Veelgebruikte authenticatiemethoden:

- authenticatie met een gedeeld geheim (`gateway.auth.mode="token"` of `"password"`): `Authorization: Bearer <token-or-password>`
- vertrouwde identiteitsdragende HTTP-authenticatie (`gateway.auth.mode="trusted-proxy"`): leid het verkeer via de geconfigureerde identiteitsbewuste proxy en laat deze de vereiste identiteitsheaders invoegen
- open authenticatie via een privé-ingress (`gateway.auth.mode="none"`): geen authenticatieheader vereist

## Beveiligingsmodel

Behandel deze Plugin als een volledig Gateway-oppervlak voor operators.

- Door de Plugin in te schakelen, bied je opzettelijk toegang tot de admin-RPC-methoden uit de allowlist op `/api/v1/admin/rpc`.
- De Plugin declareert het gereserveerde `contracts.gatewayMethodDispatch: ["authenticated-request"]`-manifestcontract, waardoor de met Gateway geauthenticeerde HTTP-route controlplanemethoden binnen het proces kan doorsturen. Dit is geen sandbox: het contract voorkomt onbedoeld gebruik van gereserveerde SDK-helpers, maar vertrouwde Plugins worden nog steeds uitgevoerd in het Gateway-proces.
- Bearer-authenticatie met een gedeeld geheim (`token`/`password`-modi) bewijst het bezit van het operatorgeheim van de Gateway; specifiekere `x-openclaw-scopes`-headers worden op dat pad genegeerd en de normale volledige operatorstandaardwaarden worden hersteld.
- Vertrouwde identiteitsdragende HTTP-authenticatie (`trusted-proxy`-modus) respecteert `x-openclaw-scopes` wanneer dit aanwezig is.
- `gateway.auth.mode="none"` betekent dat deze route geen authenticatie vereist als de Plugin is ingeschakeld. Gebruik dit alleen achter een privé-ingress die je volledig vertrouwt.
- Nadat de authenticatie van de Plugin-route is geslaagd, worden verzoeken afgehandeld via dezelfde Gateway-methodhandlers en bereikcontroles als WebSocket RPC.
- De route blijft bereikbaar tijdens een voorbereide opschortingslease. Begrensde verzoekvalidatie en het lokale `commands.list`-detectieantwoord blijven beschikbaar. Van de methoden die naar de Gateway worden doorgestuurd, mogen alleen `gateway.suspend.prepare`, `gateway.suspend.status` en `gateway.suspend.resume` worden uitgevoerd wanneer toelating is gesloten; andere methoden uit de allowlist retourneren het normale opnieuw te proberen Gateway-antwoord `UNAVAILABLE`.
- Houd deze route op loopback, een tailnet of een vertrouwde privé-ingress. Stel de route niet rechtstreeks bloot aan het openbare internet. Gebruik afzonderlijke gateways wanneer aanroepers vertrouwensgrenzen overschrijden.

## Verzoek

```http
POST /api/v1/admin/rpc
Authorization: Bearer <gateway-token>
Content-Type: application/json
```

```json
{
  "id": "optional-request-id",
  "method": "health",
  "params": {}
}
```

Velden:

- `id` (tekenreeks, optioneel): wordt naar het antwoord gekopieerd. Als dit wordt weggelaten, wordt een UUID gegenereerd.
- `method` (tekenreeks, vereist): naam van een toegestane Gateway-methode.
- `params` (elk type, optioneel): parameters die specifiek zijn voor de methode.

De standaard maximale grootte van de verzoekbody is 1 MB.

## Antwoord

Geslaagde antwoorden gebruiken de RPC-structuur van de Gateway:

```json
{
  "id": "optional-request-id",
  "ok": true,
  "payload": {}
}
```

Fouten van Gateway-methoden gebruiken:

```json
{
  "id": "optional-request-id",
  "ok": false,
  "error": {
    "code": "INVALID_REQUEST",
    "message": "bad params"
  }
}
```

De HTTP-status volgt de foutcode:

| Foutcode                   | HTTP-status |
| -------------------------- | ----------- |
| `INVALID_REQUEST`          | 400         |
| `APPROVAL_NOT_FOUND`       | 404         |
| `NOT_LINKED`, `NOT_PAIRED` | 409         |
| `UNAVAILABLE`              | 503         |
| `AGENT_TIMEOUT`            | 504         |
| elke andere code           | 500         |

## Toegestane methoden

- detectie: `commands.list`
  Retourneert de namen van de HTTP RPC-methoden die door deze Plugin zijn toegestaan.
- Gateway: `health`, `status`, `logs.tail`, `usage.status`, `usage.cost`, `gateway.restart.request`, `gateway.suspend.prepare`, `gateway.suspend.status`, `gateway.suspend.resume`
- configuratie: `config.get`, `config.schema`, `config.schema.lookup`, `config.set`, `config.patch`, `config.apply`
- kanalen: `channels.status`, `channels.start`, `channels.stop`, `channels.logout`
- web: `web.login.start`, `web.login.wait`
- modellen: `models.list`, `models.authStatus`
- agents: `agents.list`, `agents.create`, `agents.update`, `agents.delete`
- goedkeuringen: `exec.approvals.get`, `exec.approvals.set`, `exec.approvals.node.get`, `exec.approvals.node.set`
- Cron: `cron.status`, `cron.list`, `cron.get`, `cron.runs`, `cron.add`, `cron.update`, `cron.remove`, `cron.run`
- apparaten: `device.pair.list`, `device.pair.approve`, `device.pair.reject`, `device.pair.remove`
- Nodes: `node.list`, `node.describe`, `node.pair.list`, `node.pair.approve`, `node.pair.reject`, `node.pair.remove`, `node.rename`
- taken: `tasks.list`, `tasks.get`, `tasks.cancel`
- diagnostiek: `doctor.memory.status`, `update.status`

Andere Gateway-methoden worden geblokkeerd totdat ze bewust worden toegevoegd.

## Vergelijking met WebSocket

Het normale Gateway WebSocket RPC-pad blijft de voorkeurs-API voor het controlplane van OpenClaw-clients. Gebruik admin HTTP RPC alleen voor hosttools die een HTTP-oppervlak voor verzoeken en antwoorden nodig hebben.

WebSocket-clients met een gedeeld token maar zonder vertrouwde apparaatidentiteit kunnen tijdens het verbinden niet zelf adminbereiken declareren. Admin HTTP RPC volgt bewust het bestaande vertrouwde HTTP-operatormodel: wanneer de Plugin is ingeschakeld, wordt bearer-authenticatie met een gedeeld geheim voor dit adminoppervlak behandeld als volledige operatortoegang.

## Problemen oplossen

`404 Not Found`

: De Plugin is uitgeschakeld, de Gateway is sinds het inschakelen niet opnieuw gestart of het verzoek wordt naar een ander Gateway-proces verzonden.

`401 Unauthorized`

: Het verzoek voldeed niet aan de HTTP-authenticatie van de Gateway. Controleer het bearertoken of de identiteitsheaders van de vertrouwde proxy.

`405 Method Not Allowed`

: Het verzoek gebruikte iets anders dan `POST`.

`413 Payload Too Large`

: De verzoekbody overschreed de limiet van 1 MB.

`400 INVALID_REQUEST`

: De verzoekbody is geen geldige JSON, het veld `method` ontbreekt, de methode staat niet in de allowlist van de Plugin of een hervattings-ID voor een opschorting komt niet overeen met de actieve lease.

`503 UNAVAILABLE`

: De Gateway-methode wordt gestart, is onderworpen aan een snelheidslimiet, is opgeschort of wacht op een concurrerende opschortings- of hervattingsbewerking. Controleer `error.details` wanneer dit aanwezig is en respecteer `error.retryAfterMs` voordat je het opnieuw probeert.

## Gerelateerd

- [Operatorbereiken](/nl/gateway/operator-scopes)
- [Gateway-beveiliging](/nl/gateway/security)
- [Externe toegang](/nl/gateway/remote)
- [Plugin-manifest](/nl/plugins/manifest#contracts-reference)
- [SDK-subpaden](/nl/plugins/sdk-subpaths)
