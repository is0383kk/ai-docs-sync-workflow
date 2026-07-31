---
read_when:
    - Tools aanroepen zonder een volledige agentbeurt uit te voeren
    - Automatiseringen bouwen waarvoor handhaving van toolbeleid nodig is
summary: Roep één tool rechtstreeks aan via het HTTP-eindpunt van de Gateway
title: Tools roepen de API aan
x-i18n:
    generated_at: "2026-07-27T06:18:03Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6d07f765d63255e718d5e558b662589e77b2992538f43288cd83e6e3f2a06dda
    source_path: gateway/tools-invoke-http-api.md
    workflow: 16
---

OpenClaw's Gateway biedt een HTTP-eindpunt om één tool rechtstreeks aan te roepen. Het is altijd ingeschakeld en gebruikt Gateway-authenticatie plus toolbeleid. Net als bij het OpenAI-compatibele `/v1/*`-oppervlak wordt bearer-authenticatie met een gedeeld geheim beschouwd als vertrouwde operatortoegang voor de volledige Gateway.

- `POST /tools/invoke`
- Dezelfde poort als de Gateway (WS + HTTP-multiplexing): `http://<gateway-host>:<port>/tools/invoke`
- Standaard maximale grootte van de aanvraagbody: 2 MB

## Authenticatie

Gebruikt de authenticatieconfiguratie van de Gateway.

Veelgebruikte HTTP-authenticatiepaden:

- authenticatie met gedeeld geheim (`gateway.auth.mode="token"` of `"password"`): `Authorization: Bearer <token-or-password>`
- vertrouwde HTTP-authenticatie met identiteit (`gateway.auth.mode="trusted-proxy"`): routeer via de geconfigureerde identiteitsbewuste proxy en laat deze de vereiste identiteitsheaders invoegen
- open authenticatie voor private ingress (`gateway.auth.mode="none"`): geen authenticatieheader vereist

Opmerkingen:

- `mode="token"` gebruikt `gateway.auth.token` (of `OPENCLAW_GATEWAY_TOKEN`).
- `mode="password"` gebruikt `gateway.auth.password` (of `OPENCLAW_GATEWAY_PASSWORD`).
- `mode="trusted-proxy"` vereist dat de HTTP-aanvraag afkomstig is van een geconfigureerde vertrouwde proxybron; loopbackproxy's op dezelfde host vereisen expliciet `gateway.auth.trustedProxy.allowLoopback = true`.
- Interne aanroepers op dezelfde host die de proxy omzeilen, kunnen `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD` gebruiken als lokale directe fallback. Bewijs in een `Forwarded`-, `X-Forwarded-*`- of `X-Real-IP`-header houdt de aanvraag in plaats daarvan op het pad voor vertrouwde proxy's.
- Als `gateway.auth.rateLimit` is geconfigureerd en er te veel authenticatiefouten optreden, retourneert het eindpunt `429` met `Retry-After`.

## Beveiligingsgrens (belangrijk)

Beschouw dit eindpunt als een oppervlak met **volledige operatortoegang** tot de Gateway-instantie.

- HTTP-bearer-authenticatie is hier geen model met een beperkte scope per gebruiker.
- Een geldig Gateway-token/wachtwoord voor dit eindpunt moet worden beschouwd als een referentie van een eigenaar/operator.
- Voor authenticatiemodi met een gedeeld geheim (`token` en `password`) herstelt het eindpunt de normale volledige operatorstandaardwaarden, zelfs als de aanroeper een beperktere `x-openclaw-scopes`-header verzendt.
- Authenticatie met een gedeeld geheim behandelt rechtstreekse toolaanroepen op dit eindpunt ook als beurten van de eigenaar-afzender.
- Vertrouwde HTTP-modi met identiteit (authenticatie via een vertrouwde proxy, of `gateway.auth.mode="none"` op een private ingress) respecteren `x-openclaw-scopes` wanneer aanwezig en vallen anders terug op de normale set standaardscopes voor operators.
- Houd dit eindpunt uitsluitend op loopback/tailnet/private ingress; stel het niet rechtstreeks bloot aan het openbare internet.

Authenticatiematrix:

| Authenticatiemodus                                                                       | Gedrag                                                                                                                                                                                                                                                                                                                                 |
| --------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `token` of `password` + `Authorization: Bearer ...`                                     | Bewijst het bezit van het gedeelde operatorgeheim van de Gateway. Negeert een beperktere `x-openclaw-scopes`. Herstelt de volledige set standaardscopes voor operators: `operator.admin`, `operator.approvals`, `operator.pairing`, `operator.read`, `operator.talk.secrets`, `operator.write`. Behandelt rechtstreekse toolaanroepen als beurten van de eigenaar-afzender. |
| Vertrouwde HTTP met identiteit (authenticatie via een vertrouwde proxy, of `mode="none"` op private ingress) | Verifieert een externe vertrouwde identiteit of implementatiegrens. Respecteert `x-openclaw-scopes` wanneer aanwezig. Valt terug op de normale set standaardscopes voor operators wanneer de header ontbreekt. Verliest eigenaarssemantiek alleen wanneer de aanroeper scopes expliciet beperkt en `operator.admin` weglaat.                               |

## Aanvraagbody

```json
{
  "tool": "sessions_list",
  "action": "json",
  "args": {},
  "sessionKey": "main",
  "dryRun": false
}
```

Velden:

- `tool` / `name` (tekenreeks, vereist): naam van de aan te roepen tool. `name` heeft voorrang als beide worden verzonden.
- `action` (tekenreeks, optioneel): wordt samengevoegd in `args.action` als het toolschema een `action`-eigenschap ondersteunt en `args` er nog geen heeft ingesteld.
- `args` (object, optioneel): toolspecifieke argumenten.
- `sessionKey` (tekenreeks, optioneel): sleutel van de doelsessie. Indien weggelaten of `"main"`, gebruikt de Gateway de geconfigureerde sleutel van de hoofdsessie (respecteert `session.mainKey` en de standaardagent, of `global` binnen de globale sessiescope).
- `agentId` (tekenreeks, optioneel): bepaalt de sessiesleutel voor die agent. Geeft een fout met `400` als deze conflicteert met een expliciete `sessionKey` die al aan een andere agent is gekoppeld.
- `idempotencyKey` (tekenreeks, optioneel): wordt gebruikt om een stabiele toolaanroep-id voor de aanroep af te leiden.
- `dryRun` (booleaans, optioneel): gereserveerd voor toekomstig gebruik; wordt momenteel genegeerd.

## Beleids- en routeringsgedrag

De beschikbaarheid van tools wordt gefilterd via dezelfde beleidsketen die Gateway-agents gebruiken:

- `tools.profile` / `tools.byProvider.profile`
- `tools.allow` / `tools.byProvider.allow`
- `agents.<id>.tools.allow` / `agents.<id>.tools.byProvider.allow`
- groepsbeleid (als de sessiesleutel aan een groep of kanaal is gekoppeld)
- subagentbeleid (bij aanroepen met de sessiesleutel van een subagent)

Als een tool niet door het beleid is toegestaan, retourneert het eindpunt **404**.

Belangrijke opmerkingen over grenzen:

- Uitvoeringsgoedkeuringen zijn veiligheidsmaatregelen voor operators, geen afzonderlijke autorisatiegrens voor dit HTTP-eindpunt. Als een tool hier bereikbaar is via Gateway-authenticatie + toolbeleid, voegt `/tools/invoke` geen extra goedkeuringsprompt per aanroep toe.
- Als `exec` hier bereikbaar is, beschouw deze dan als een muterend shell-oppervlak. Het weigeren van `write`, `edit`, `apply_patch` of HTTP-tools die naar het bestandssysteem schrijven, maakt shell-uitvoering niet alleen-lezen.
- Deel Gateway-bearer-referenties niet met niet-vertrouwde aanroepers. Als scheiding tussen vertrouwensgrenzen nodig is, voer dan afzonderlijke Gateways uit (bij voorkeur onder afzonderlijke OS-gebruikers/op hosts).

Gateway HTTP past standaard ook een harde blokkeerlijst toe (zelfs als het sessiebeleid de tool toestaat):

| Tool             | Reden                                                     |
| ---------------- | --------------------------------------------------------- |
| `exec`           | Rechtstreekse opdrachtuitvoering (RCE-oppervlak)           |
| `spawn`          | Willekeurig aanmaken van onderliggende processen (RCE-oppervlak) |
| `shell`          | Uitvoering van shell-opdrachten (RCE-oppervlak)            |
| `fs_write`       | Willekeurige bestandswijziging op de host                  |
| `fs_delete`      | Willekeurige bestandsverwijdering op de host               |
| `fs_move`        | Willekeurig verplaatsen/hernoemen van bestanden op de host |
| `apply_patch`    | Het toepassen van patches kan willekeurige bestanden herschrijven |
| `sessions_spawn` | Sessieregie; agents op afstand starten is RCE              |
| `sessions_send`  | Injectie van berichten tussen sessies                      |
| `cron`           | Besturingsvlak voor permanente automatisering              |
| `gateway`        | Gateway-besturingsvlak; voorkomt herconfiguratie via HTTP  |
| `nodes`          | Node-opdrachtdoorgifte kan `system.run` bereiken op gekoppelde hosts |

`cron`, `gateway` en `nodes` zijn ook uitsluitend voor eigenaren: zelfs buiten deze standaardblokkeerlijst kunnen aanroepers die geen eigenaar zijn ze niet via dit oppervlak aanroepen.

Pas de algemene blokkeerlijst aan via `gateway.tools`:

```json5
{
  gateway: {
    tools: {
      // Extra tools om via HTTP /tools/invoke te blokkeren
      deny: ["browser"],
      // Tools voor aanroepers met eigenaar-/beheerdersrechten uit de standaardblokkeerlijst verwijderen
      allow: ["gateway"],
    },
  },
}
```

`gateway.tools.allow` is een overschrijving van de blootstelling, geen scope-upgrade. In HTTP-modi met identiteit blijven `cron`, `gateway` en `nodes` niet beschikbaar voor aanroepers zonder eigenaar-/beheerdersidentiteit (`operator.admin`), zelfs als ze in `gateway.tools.allow` staan. Bearer-authenticatie met een gedeeld geheim volgt nog steeds de bovenstaande regel voor volledig vertrouwde operators.

Om groepsbeleid te helpen bij het bepalen van de context, kun je optioneel het volgende instellen:

- `x-openclaw-message-channel: <channel>` (voorbeeld: `slack`, `telegram`)
- `x-openclaw-account-id: <accountId>` (wanneer er meerdere accounts bestaan)
- `x-openclaw-message-to: <target>` (afleverdoel voor beleid van de berichtentool)
- `x-openclaw-thread-id: <threadId>` (threadcontext voor beleid van de berichtentool)

## Antwoorden

| Status | Betekenis                                                                                      |
| ------ | ---------------------------------------------------------------------------------------------- |
| `200`  | `{ ok: true, result }`                                                                         |
| `400`  | `{ ok: false, error: { type, message } }` (ongeldige aanvraag of fout in toolinvoer)                |
| `401`  | Niet geautoriseerd                                                                             |
| `403`  | `{ ok: false, error: { type, message, requiresApproval? } }` (toolaanroep geblokkeerd door beleid)     |
| `404`  | Tool niet beschikbaar (niet gevonden of niet op de toestemmingslijst)                          |
| `405`  | Methode niet toegestaan                                                                        |
| `408`  | Time-out bij het lezen van de aanvraagbody                                                     |
| `413`  | Aanvraagbody overschreed de maximale payloadgrootte                                            |
| `429`  | Authenticatiesnelheid beperkt (`Retry-After` ingesteld)                                  |
| `500`  | `{ ok: false, error: { type, message } }` (onverwachte fout bij tooluitvoering; opgeschoond bericht) |

## Voorbeeld

```bash
curl -sS http://127.0.0.1:18789/tools/invoke \
  -H 'Authorization: Bearer secret' \
  -H 'Content-Type: application/json' \
  -d '{
    "tool": "sessions_list",
    "action": "json",
    "args": {}
  }'
```

## Gerelateerd

- [Gateway-protocol](/nl/gateway/protocol)
- [Tools en plugins](/nl/tools)
