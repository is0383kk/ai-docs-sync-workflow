---
read_when:
    - Eindpunten toevoegen/wijzigen
    - CLI-↔-registerverzoeken debuggen
summary: HTTP-API-referentie (openbare + CLI-eindpunten + authenticatie).
x-i18n:
    generated_at: "2026-07-27T04:49:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5b180bbd56d20a3d88c1fe74ccab0fd0ecbe0e8c9624cd1afd2070a2ca1f7fb3
    source_path: clawhub/http-api.md
    workflow: 16
---

# HTTP-API

Basis-URL: `https://clawhub.ai` (standaard).

Alle v1-paden vallen onder `/api/v1/...`.
Verouderde `/api/...` en `/api/cli/...` blijven beschikbaar voor compatibiliteit (zie `DEPRECATIONS.md`).
OpenAPI: `/api/v1/openapi.json`.

## Hergebruik van de openbare catalogus

Externe directory's mogen de openbare leesendpoints gebruiken om ClawHub-Skills weer te geven of te doorzoeken. Cache de resultaten, respecteer `429`/`Retry-After`, verwijs gebruikers terug naar de canonieke ClawHub-vermelding (`https://clawhub.ai/<owner>/skills/<slug>`) en wek niet de indruk dat ClawHub de externe site onderschrijft. Probeer geen verborgen, privé- of door moderatie geblokkeerde inhoud buiten het openbare API-oppervlak te spiegelen.

Verkorte webslugs worden tussen registerfamilies omgezet, maar API-clients moeten
de canonieke URL's gebruiken die leesendpoints retourneren, in plaats van zelf de
routeprioriteit te reconstrueren.

## Frequentielimieten

Handhavingsmodel:

- Anonieme verzoeken: gehandhaafd per IP-adres.
- Geverifieerde verzoeken (geldig Bearer-token): gehandhaafd per gebruikersbucket.
- Als het token ontbreekt of ongeldig is, valt het gedrag terug op handhaving per IP-adres.
- Geverifieerde schrijfendpoints mogen niet alleen een kale `Unauthorized` retourneren wanneer
  de server de reden kent. Ontbrekende tokens, ongeldige/ingetrokken tokens en
  verwijderde/verbannen/uitgeschakelde accounts moeten elk bruikbare tekst opleveren, zodat CLI-
  clients gebruikers kunnen vertellen wat hen heeft geblokkeerd.

- Lezen: 3000/min per IP-adres, 12000/min per sleutel
- Schrijven: 300/min per IP-adres, 3000/min per sleutel
- Downloaden: 1200/min per IP-adres, 6000/min per sleutel (downloadendpoints)

Headers:

- Compatibiliteit met verouderde versies: `X-RateLimit-Limit`, `X-RateLimit-Reset`
- Gestandaardiseerd: `RateLimit-Limit`, `RateLimit-Reset`
- Bij `429`: `X-RateLimit-Remaining: 0` en `RateLimit-Remaining: 0`
- Bij `429`: `Retry-After`

Betekenis van headers:

- `X-RateLimit-Reset`: absolute Unix-epochtijd in seconden
- `RateLimit-Reset`: aantal seconden tot reset (vertraging)
- `X-RateLimit-Remaining` / `RateLimit-Remaining`: exact resterend budget, indien aanwezig.
  Geslaagde gesharde verzoeken laten deze header weg in plaats van een geschatte globale waarde te retourneren.
- `Retry-After`: aantal seconden dat moet worden gewacht voordat opnieuw wordt geprobeerd (vertraging) bij `429`

Voorbeeld van een `429`-respons:

```http
HTTP/2 429
content-type: text/plain; charset=utf-8
x-ratelimit-limit: 20
x-ratelimit-remaining: 0
x-ratelimit-reset: 1771404540
ratelimit-limit: 20
ratelimit-remaining: 0
ratelimit-reset: 34
retry-after: 34

Frequentielimiet overschreden
```

Richtlijnen voor clients:

- Als `Retry-After` bestaat, wacht dan dat aantal seconden voordat je het opnieuw probeert.
- Gebruik back-off met jitter om gesynchroniseerde nieuwe pogingen te voorkomen.
- Als `Retry-After` ontbreekt, val dan terug op `RateLimit-Reset` (of bereken dit op basis van `X-RateLimit-Reset`).

IP-bron:

- Gebruikt vertrouwde headers voor client-IP-adressen, waaronder `cf-connecting-ip`, alleen wanneer de
  implementatie vertrouwde doorgestuurde headers expliciet inschakelt.
- ClawHub gebruikt vertrouwde doorgestuurde headers om client-IP-adressen aan de rand te identificeren.
- Als er geen vertrouwd client-IP-adres beschikbaar is, gebruiken anonieme verzoeken fallback-buckets
  die alleen door het type frequentielimiet zijn afgebakend. Deze fallback-buckets bevatten geen
  door de aanroeper aangeleverde paden, slugs, pakketnamen, versies, querystrings of andere
  artefactparameters.

## Foutresponsen

Openbare v1-foutresponsen zijn platte tekst met `content-type: text/plain; charset=utf-8`.
Dit omvat validatiefouten (`400`), ontbrekende openbare bronnen (`404`), verificatie- en
machtigingsfouten (`401`/`403`), frequentielimieten (`429`) en geblokkeerde downloads. Clients
moeten de responsbody als een voor mensen leesbare tekenreeks verwerken. Onbekende queryparameters worden
voor compatibiliteit genegeerd, maar herkende queryparameters met ongeldige waarden retourneren
`400`.

## Openbare endpoints (geen verificatie)

### `GET /api/v1/search`

Queryparameters:

- `q` (vereist): querytekenreeks
- `limit` (optioneel): geheel getal
- `highlightedOnly` (optioneel): `true` om te filteren op uitgelichte Skills
- `nonSuspiciousOnly` (optioneel): `true` om verdachte (`flagged.suspicious`) Skills te verbergen
- `nonSuspicious` (optioneel): verouderde alias voor `nonSuspiciousOnly`

Respons:

```json
{
  "results": [
    {
      "score": 0.123,
      "slug": "gifgrep",
      "displayName": "GifGrep",
      "summary": "…",
      "version": "1.2.3",
      "updatedAt": 1730000000000,
      "ownerHandle": "openclaw",
      "owner": {
        "handle": "openclaw",
        "displayName": "OpenClaw",
        "image": "https://example.com/avatar.png"
      }
    }
  ]
}
```

Opmerkingen:

- Resultaten worden geretourneerd in volgorde van relevantie (overeenkomst van embeddings + boosts voor exacte slug-/naamstokens + een kleine voorafgaande populariteitsweging).
- Relevantie weegt zwaarder dan populariteit. Een exacte overeenkomst met een slug- of weergavenaamtoken kan hoger eindigen dan een ruimere overeenkomst met veel sterkere betrokkenheid.
- ASCII-tekst wordt getokeniseerd op woord- en interpunctiegrenzen. `personal-map` bevat bijvoorbeeld een zelfstandig `map`-token, terwijl `amap-jsapi-skill` `amap`, `jsapi` en `skill` bevat; zoeken naar `map` geeft `personal-map` daarom een sterkere lexicale overeenkomst dan `amap-jsapi-skill`.
- Populariteit wordt logaritmisch geschaald en begrensd. Skills met veel betrokkenheid kunnen lager eindigen wanneer de querytekst minder goed overeenkomt.
- Een verdachte of verborgen moderatiestatus kan een Skill uit openbare zoekresultaten verwijderen, afhankelijk van de filters van de aanroeper en de huidige moderatiestatus.

Richtlijnen voor vindbaarheid van uitgevers:

- Plaats de termen waarop gebruikers letterlijk zullen zoeken in de weergavenaam, samenvatting en tags. Gebruik alleen een zelfstandig slugtoken wanneer dit ook een stabiele identiteit is die je wilt behouden.
- Wijzig een slug niet alleen om op één query in te spelen, tenzij de nieuwe slug op lange termijn een betere canonieke naam is. Oude slugs worden omleidingsaliassen, maar de canonieke URL, weergegeven slug en toekomstige zoeksamenvattingen gebruiken de nieuwe slug.
- Hernoemingsaliassen behouden de resolutie voor oude URL's en installaties die via het register worden omgezet, maar de zoekrangschikking is gebaseerd op de canonieke Skill-metadata nadat de hernoeming is geïndexeerd. Bestaande statistieken blijven aan de Skill gekoppeld.
- Als een Skill onverwacht onzichtbaar is, controleer dan eerst de moderatiestatus met `clawhub inspect @owner/slug` terwijl je bent ingelogd, voordat je metadata wijzigt die verband houdt met de rangschikking.

### `GET /api/v1/skills`

Queryparameters:

- `limit` (optioneel): geheel getal (1–200)
- `cursor` (optioneel): pagineringscursor voor elke sortering anders dan `trending`
- `sort` (optioneel): `updated` (standaard), `recommended` (alias: `default`), `createdAt` (alias: `newest`), `downloads`, `stars` (alias: `rating`), verouderde installatiealiassen `installsCurrent`/`installs`/`installsAllTime` verwijzen naar `downloads`, `trending`
- `nonSuspiciousOnly` (optioneel): `true` om verdachte (`flagged.suspicious`) Skills te verbergen
- `nonSuspicious` (optioneel): verouderde alias voor `nonSuspiciousOnly`

Ongeldige waarden voor `sort` retourneren `400`.

Opmerkingen:

- `recommended` gebruikt signalen voor betrokkenheid en recentheid.
- `trending` rangschikt op basis van installaties in de afgelopen 7 dagen (op basis van telemetrie).
- `createdAt` is stabiel voor crawls van nieuwe Skills; `updated` verandert wanneer bestaande Skills opnieuw worden gepubliceerd.
- Bij `nonSuspiciousOnly=true` kunnen sorteringen op basis van een cursor minder dan `limit` items op een pagina retourneren, omdat verdachte Skills na het ophalen van de pagina worden uitgefilterd.
- Gebruik `nextCursor` om de paginering voort te zetten wanneer deze aanwezig is. Een korte pagina betekent op zichzelf niet dat het einde van de resultaten is bereikt.

Respons:

```json
{
  "items": [
    {
      "slug": "gifgrep",
      "displayName": "GifGrep",
      "summary": "…",
      "topics": ["Productivity"],
      "tags": { "latest": "1.2.3" },
      "stats": {},
      "createdAt": 0,
      "updatedAt": 0,
      "latestVersion": { "version": "1.2.3", "createdAt": 0, "changelog": "…" },
      "metadata": { "os": ["macos"], "systems": ["aarch64-darwin"] }
    }
  ],
  "nextCursor": null
}
```

### `GET /api/v1/skills/{slug}`

Respons:

```json
{
  "skill": {
    "slug": "gifgrep",
    "displayName": "GifGrep",
    "summary": "…",
    "topics": ["Productivity"],
    "tags": { "latest": "1.2.3" },
    "stats": {},
    "createdAt": 0,
    "updatedAt": 0
  },
  "latestVersion": { "version": "1.2.3", "createdAt": 0, "changelog": "…" },
  "metadata": { "os": ["macos"], "systems": ["aarch64-darwin"] },
  "owner": { "handle": "steipete", "displayName": "Peter", "image": null },
  "moderation": {
    "isSuspicious": false,
    "isMalwareBlocked": false,
    "verdict": "clean",
    "reasonCodes": [],
    "summary": null,
    "engineVersion": "v2.0.0",
    "updatedAt": 0
  }
}
```

Opmerkingen:

- Oude slugs die zijn aangemaakt door hernoemings-/samenvoegingsflows van eigenaren worden omgezet naar de canonieke Skill.
- `metadata.os`: OS-beperkingen die in de frontmatter van de Skill zijn gedeclareerd (bijv. `["macos"]`, `["linux"]`). `null` indien niet gedeclareerd.
- `metadata.systems`: Nix-systeemdoelen (bijv. `["aarch64-darwin", "x86_64-linux"]`). `null` indien niet gedeclareerd.
- `metadata` is `null` als de Skill geen platformmetadata heeft.
- `moderation` wordt alleen opgenomen wanneer de Skill is gemarkeerd of de eigenaar deze bekijkt.

### `GET /api/v1/skills/{slug}/moderation`

Retourneert gestructureerde moderatiestatus.

Respons:

```json
{
  "moderation": {
    "isSuspicious": true,
    "isMalwareBlocked": false,
    "verdict": "suspicious",
    "reasonCodes": ["suspicious.dynamic_code_execution"],
    "summary": "Detected: suspicious.dynamic_code_execution",
    "engineVersion": "v2.0.0",
    "updatedAt": 0,
    "legacyReason": null,
    "evidence": [
      {
        "code": "suspicious.dynamic_code_execution",
        "severity": "critical",
        "file": "index.ts",
        "line": 3,
        "message": "Dynamic code execution detected.",
        "evidence": ""
      }
    ]
  }
}
```

Opmerkingen:

- Eigenaren en moderators hebben toegang tot moderatiedetails voor verborgen Skills.
- Openbare aanroepers krijgen `200` alleen voor reeds gemarkeerde zichtbare Skills.
- Bewijs wordt voor openbare aanroepers geredigeerd en bevat alleen ruwe fragmenten voor eigenaren/moderators.

### `POST /api/v1/skills/{slug}/report`

Rapporteer een Skill ter beoordeling door een moderator. Rapporten gelden voor de hele Skill, zijn optioneel gekoppeld
aan een versie en worden toegevoegd aan de wachtrij met Skill-rapporten.

Verificatie:

- Vereist een API-token.

Verzoek:

```json
{ "reason": "Suspicious install step", "version": "1.2.3" }
```

Respons:

```json
{
  "ok": true,
  "reported": true,
  "alreadyReported": false,
  "reportId": "skillReports:...",
  "skillId": "skills:...",
  "reportCount": 1
}
```

### `GET /api/v1/skills/-/reports`

Endpoint voor moderators/beheerders voor de intake van Skill-rapporten.

Queryparameters:

- `status` (optioneel): `open` (standaard), `confirmed`, `dismissed` of `all`
- `limit` (optioneel): geheel getal (1-200)
- `cursor` (optioneel): pagineringscursor

Respons:

```json
{
  "items": [
    {
      "reportId": "skillReports:...",
      "skillId": "skills:...",
      "skillVersionId": "skillVersions:...",
      "slug": "gifgrep",
      "displayName": "GifGrep",
      "version": "1.2.3",
      "reason": "Verdachte installatiestap",
      "status": "open",
      "createdAt": 1730000000000,
      "reporter": {
        "userId": "users:...",
        "handle": "reporter",
        "displayName": "Melder"
      },
      "triagedAt": null,
      "triagedBy": null,
      "triageNote": null
    }
  ],
  "nextCursor": null,
  "done": true
}
```

### `POST /api/v1/skills/-/reports/{reportId}/triage`

Endpoint voor moderators/beheerders om skillmeldingen af te handelen of te heropenen.

Verzoek:

```json
{ "status": "confirmed", "note": "Beoordeeld en betreffende versie verborgen.", "finalAction": "hide" }
```

`note` is vereist voor `confirmed` en `dismissed`; deze mag worden weggelaten wanneer
`status` weer wordt ingesteld op `open`. Geef `finalAction: "hide"` door met een getriageerde
melding om de skill binnen dezelfde controleerbare workflow te verbergen.

### `GET /api/v1/skills/{slug}/versions`

Queryparameters:

- `limit` (optioneel): geheel getal
- `cursor` (optioneel): pagineringscursor

### `GET /api/v1/skills/{slug}/versions/{version}`

Retourneert versiemetadata en een bestandenlijst.

- `version.security` bevat, indien beschikbaar, de genormaliseerde verificatiestatus van de scan en scannergegevens
  (VirusTotal + LLM).

### `GET /api/v1/skills/{slug}/scan`

Retourneert details van de beveiligingsscanverificatie voor een skillversie.

Queryparameters:

- `version` (optioneel): specifieke versietekenreeks.
- `tag` (optioneel): een getagde versie omzetten (bijvoorbeeld `latest`).

Opmerkingen:

- Als noch `version` noch `tag` is opgegeven, wordt de nieuwste versie gebruikt.
- Bevat de genormaliseerde verificatiestatus plus scannerspecifieke details.
- `security.hasScanResult` is alleen `true` wanneer een scanner een definitief oordeel heeft opgeleverd (`clean`, `suspicious` of `malicious`).
- `moderation` is een actuele moderatiemomentopname op skillniveau, afgeleid van de nieuwste versie.
- Controleer bij het opvragen van een historische versie `moderation.matchesRequestedVersion` en `moderation.sourceVersion` voordat je `moderation` en `security` als dezelfde versiecontext behandelt.

### `POST /api/v1/skills/-/scan`

Geverifieerd indieningsendpoint voor nieuwe ClawScan-taken.

Scans van lokale uploads worden niet meer ondersteund. Verzoeken die
`multipart/form-data` of `{ "source": { "kind": "upload" } }` gebruiken, retourneren `410`.

Gepubliceerde scans gebruiken JSON:

```json
{
  "source": { "kind": "published", "slug": "gifgrep", "version": "1.2.3" },
  "update": false
}
```

Opmerkingen:

- Payloads van scanverzoeken en downloadbare rapporten verlopen in de opslag voor scanverzoeken na het bewaarvenster.
- Gepubliceerde scans vereisen beheerstoegang als eigenaar/uitgever of bevoegdheid als platformmoderator/-beheerder.
- Gepubliceerde scans schrijven alleen terug wanneer `update: true` en de scan met succes wordt voltooid.
- Het antwoord is `202` met `{ "ok": true, "scanId": "...", "jobId": "...", "status": "queued", "sourceKind": "published", "update": false, "queue": { "queuedAhead": 0, "queuedAheadIsEstimate": false, "position": 1, "running": 0, "runningIsEstimate": false, "note": "Scans are asynchronous and may take time to complete." } }`.
- Scantaken zijn asynchroon. Handmatige scanverzoeken krijgen voorrang boven normaal publicatie-/backfillwerk, maar voltooiing blijft afhankelijk van de beschikbaarheid van workers.

### `GET /api/v1/skills/-/scan/{scanId}`

Geverifieerd pollingendpoint voor een ingediende scan.

- Retourneert de status in wachtrij/actief/geslaagd/mislukt.
- Retourneert `queue.queuedAhead` en `queue.position` zolang het verzoek in de wachtrij staat, zodat clients kunnen tonen hoeveel handmatige scans met prioriteit vóór het verzoek staan. Zeer grote wachtrijen worden begrensd en gerapporteerd met `queuedAheadIsEstimate: true`.
- Indien beschikbaar bevat `report` de secties `clawscan`, `skillspector`, `staticAnalysis` en `virustotal`.
- Mislukte scantaken retourneren `status: "failed"` met `lastError`.

### `GET /api/v1/skills/-/scan/{scanId}/download`

Geverifieerd endpoint voor rapportarchieven.

- Vereist een geslaagde scan; niet-afgesloten scans retourneren `409`.
- Retourneert een ZIP-bestand met `manifest.json`, `clawscan.json`, `skillspector.json`, `static-analysis.json`, `virustotal.json` en `README.md`.

### `GET /api/v1/skills/-/scan/download/{name}?version=<version>&kind=skill|plugin`

Geverifieerd endpoint voor opgeslagen rapportarchieven van ingediende versies.

- Vereist beheerstoegang als eigenaar/uitgever tot de skill of Plugin, of bevoegdheid als platformmoderator/-beheerder.
- Retourneert opgeslagen scanresultaten voor exact de ingediende versie, inclusief geblokkeerde of verborgen versies.
- `kind` is standaard `skill`; gebruik `kind=plugin` voor scans van plugins/pakketten.
- Retourneert dezelfde ZIP-structuur als downloads van scanverzoeken.

### `POST /api/v1/skills/-/scan/batch`

Canonieke batchroute voor opnieuw scannen, alleen voor beheerders. Deze accepteert dezelfde payloadstructuur als de verouderde `POST /api/v1/skills/-/rescan-batch`.

### `POST /api/v1/skills/-/scan/batch/status`

Canonieke batchstatusroute, alleen voor beheerders. Deze accepteert `{ "jobIds": ["..."] }` en retourneert dezelfde geaggregeerde tellers als de verouderde `POST /api/v1/skills/-/rescan-batch/status`.

### `GET /api/v1/skills/{slug}/verify`

Retourneert de verificatie-envelop van de Skill Card die door `clawhub skill verify` wordt gebruikt.

Queryparameters:

- `version` (optioneel): specifieke versietekenreeks.
- `tag` (optioneel): een getagde versie omzetten (bijvoorbeeld `latest`).

Opmerkingen:

- `ok` is alleen `true` wanneer voor de geselecteerde versie een Skill Card is gegenereerd, deze niet door moderatie als malware is geblokkeerd en de ClawScan-verificatie schoon is.
- De skillidentiteit, uitgeversidentiteit en metadata van de geselecteerde versie zijn envelopvelden op het hoogste niveau (`slug`, `displayName`, `publisherHandle`, `version`, `resolvedFrom`, `tag`, `createdAt`), zodat shellautomatisering deze kan lezen zonder geneste wrappers uit te pakken.
- `security` is het ClawScan-/beveiligingsoordeel op het hoogste niveau. Automatisering moet zich baseren op `ok`, `decision`, `reasons` en `security.status`.
- `security.signals` bevat ondersteunend scannerbewijs, zoals `staticScan`, `virusTotal` en `skillSpector`.
- `security.signals.dependencyRegistry` blijft behouden voor compatibiliteit met v1-antwoorden, maar de scanner voor het bestaan van afhankelijkheden in het register is buiten gebruik gesteld en deze sleutel is altijd `null`.
- `provenance` is alleen `server-resolved-github-import` wanneer ClawHub tijdens publicatie of import een GitHub-repo/ref/commit/pad heeft omgezet en opgeslagen; anders is deze `unavailable`.

### `POST /api/v1/skills/-/security-verdicts`

Retourneert actuele compacte beveiligingsoordelen voor exacte skillversies. Dit
collectie-endpoint is bedoeld voor clients die al weten welke geïnstalleerde
ClawHub-skillversies ze moeten weergeven, zoals de OpenClaw Control UI.

Verzoek:

```json
{
  "items": [{ "slug": "gifgrep", "version": "1.2.3" }]
}
```

Opmerkingen:

- `items` moet 1-100 unieke `{ slug, version }`-paren bevatten.
- Resultaten worden per item geretourneerd; één ontbrekende skill of versie laat niet het hele antwoord mislukken.
- Het antwoord bevat alleen beveiligingsgegevens. Het bevat geen Skill Card-gegevens, status van gegenereerde kaarten, lijsten met artefactbestanden of gedetailleerde scannerpayloads.
- `security.signals` bevat alleen ondersteunend bewijs op statusniveau; gebruik `/scan` of de beveiligingsauditpagina van ClawHub voor volledige scannergegevens.
- `security.signals.dependencyRegistry` blijft behouden voor compatibiliteit met v1-antwoorden, maar de scanner voor het bestaan van afhankelijkheden in het register is buiten gebruik gesteld en deze sleutel is altijd `null`.
- Het ontbreken van een Skill Card heeft geen invloed op `ok`, `decision` of `reasons` van dit endpoint; clients moeten de geïnstalleerde `skill-card.md` lokaal lezen wanneer ze kaartinhoud nodig hebben.
- Gebruik `/verify` wanneer je de Skill Card-verificatie-envelop voor één skill nodig hebt, `/card` wanneer je gegenereerde kaart-Markdown nodig hebt en `/scan` wanneer je gedetailleerde scannergegevens nodig hebt.

Antwoord:

```json
{
  "schema": "clawhub.skill.security-verdicts.v1",
  "items": [
    {
      "ok": true,
      "decision": "pass",
      "reasons": [],
      "requestedSlug": "gifgrep",
      "slug": "gifgrep",
      "displayName": "GifGrep",
      "publisherHandle": "steipete",
      "publisherDisplayName": "Peter",
      "requestedVersion": "1.2.3",
      "version": "1.2.3",
      "createdAt": 0,
      "checkedAt": 0,
      "skillUrl": "https://clawhub.ai/steipete/skills/gifgrep",
      "securityAuditUrl": "https://clawhub.ai/steipete/skills/gifgrep/security-audit?version=1.2.3",
      "security": {
        "status": "clean",
        "passed": true,
        "signals": {
          "staticScan": { "status": "clean", "reasonCodes": [] },
          "virusTotal": null,
          "skillSpector": null,
          "dependencyRegistry": null
        }
      }
    },
    {
      "ok": false,
      "decision": "fail",
      "reasons": ["version.not_found"],
      "requestedSlug": "missing-version",
      "requestedVersion": "1.0.0",
      "error": { "code": "version_not_found", "message": "Versie niet gevonden" },
      "security": null
    }
  ]
}
```

### `GET /api/v1/skills/{slug}/file`

Retourneert exact de opgeslagen bestandsbytes als download. Voeg `preview=1` toe om een begrensd voorbeeld met geëscapete tekst
op te vragen; elk bestand met geldige UTF-8-bytes kan als voorbeeld worden weergegeven, ongeacht de extensie of MIME-
metadata.

Queryparameters:

- `path` (vereist)
- `version` (optioneel)
- `tag` (optioneel)
- `preview=1` (optioneel; retourneert `text/plain` of `415` wanneer de bytes geen geldige UTF-8 vormen)

Opmerkingen:

- Gebruikt standaard de nieuwste versie.
- Limiet voor onbewerkte downloads: 10MB.
- Limiet voor tekstvoorbeelden: 200KB.

### `GET /api/v1/packages`

Uniform catalogusendpoint voor:

- skills
- codeplugins
- bundelplugins

Queryparameters:

- `limit` (optioneel): geheel getal (1–100)
- `cursor` (optioneel): pagineringscursor
- `family` (optioneel): `skill`, `code-plugin` of `bundle-plugin`
- `channel` (optioneel): `official`, `community` of `private`
- `isOfficial` (optioneel): `true` of `false`
- `sort` (optioneel): `updated` (standaard), `recommended`, `trending`, `downloads`, verouderde alias `installs`
- `category` (optioneel): filter voor plugincategorieën. Wordt alleen ondersteund wanneer het
  verzoek is beperkt tot pluginpakketten (`/api/v1/plugins`,
  `/api/v1/code-plugins`, `/api/v1/bundle-plugins` of pakketendpoints met
  `family=code-plugin`/`family=bundle-plugin`). Beheerde categorieën en
  verouderde v1-filteraliassen worden gedocumenteerd onder `GET /api/v1/plugins`.

Opmerkingen:

- Ongeldige waarden voor `family`, `channel`, `isOfficial`, `featured`,
  `highlightedOnly` of `sort` retourneren `400`. Onbekende queryparameters worden genegeerd.
- `GET /api/v1/code-plugins` en `GET /api/v1/bundle-plugins` blijven aliassen voor vaste families.
- Skillvermeldingen blijven gebaseerd op het skillregister en kunnen nog steeds alleen via `POST /api/v1/skills` worden gepubliceerd.
- `POST /api/v1/packages` is nog steeds alleen bedoeld voor releases van codeplugins en bundelplugins.
- Anonieme aanroepers zien alleen openbare pakketkanalen.
- Geverifieerde aanroepers kunnen in lijst-/zoekresultaten privépakketten zien van uitgevers waartoe ze behoren.
- `channel=private` retourneert alleen pakketten die de geverifieerde aanroeper kan lezen.

### `GET /api/v1/packages/search`

Uniform zoeken in de catalogus voor skills en pluginpakketten.

Queryparameters:

- `q` (verplicht): querytekenreeks
- `limit` (optioneel): geheel getal (1–100)
- `family` (optioneel): `skill`, `code-plugin` of `bundle-plugin`
- `channel` (optioneel): `official`, `community` of `private`
- `isOfficial` (optioneel): `true` of `false`
- `category` (optioneel): filter voor Plugin-categorieën. Wordt alleen ondersteund wanneer het
  verzoek is beperkt tot Plugin-pakketten. Beheerde categorieën en verouderde v1-
  filteraliassen zijn gedocumenteerd onder `GET /api/v1/plugins`.

Opmerkingen:

- Ongeldige waarden voor `family`, `channel`, `isOfficial`, `featured` of
  `highlightedOnly` retourneren `400`. Onbekende queryparameters worden genegeerd.
- Anonieme aanroepers zien alleen openbare pakketkanalen.
- Geverifieerde aanroepers kunnen zoeken in privépakketten van uitgevers waartoe ze behoren.
- `channel=private` retourneert alleen pakketten die de geverifieerde aanroeper kan lezen.

### `GET /api/v1/plugins`

Bladeren door de catalogus, uitsluitend voor Plugins, in code-Plugin- en bundel-Plugin-pakketten.

Queryparameters:

- `limit` (optioneel): geheel getal (1-100)
- `cursor` (optioneel): pagineringscursor
- `isOfficial` (optioneel): `true` of `false`
- `sort` (optioneel): `recommended` (standaard), `trending`, `downloads`, `updated`, verouderde alias `installs`
- `category` (optioneel): filter voor Plugin-categorieën. Huidige waarden:
  `channels`, `models`, `memory`, `context`, `voice`, `media`, `web`,
  `tools`, `runtime`, `gateway`, `security`, `other`.

Verouderde v1-filteraliassen blijven geaccepteerd op leeseindpunten:

- `mcp-tooling`, `data` en `automation` worden herleid tot `tools`.
- `observability` en `deployment` worden herleid tot `gateway`.
- `dev-tools` wordt herleid tot `runtime`.

`trending` is een ranglijst voor installaties/downloads over zeven dagen en gebruikt geen totalen over de gehele periode.
Op het uniforme `/api/v1/packages`-eindpunt geldt dit alleen voor Plugins; gebruik
`/api/v1/skills?sort=trending` voor de Skills-catalogus.

Verouderde aliassen worden niet geaccepteerd als opgeslagen of door auteurs gedeclareerde categoriewaarden.

### `GET /api/v1/skills/export`

Bulkexport van de nieuwste openbare Skills voor offlineanalyse.

Authenticatie:

- API-token vereist.

Queryparameters:

- `startDate` (verplicht): ondergrens in Unix-milliseconden voor `updatedAt` van de Skill.
- `endDate` (verplicht): bovengrens in Unix-milliseconden voor `updatedAt` van de Skill.
- `limit` (optioneel): geheel getal (1-250), standaard `250`.
- `cursor` (optioneel): pagineringscursor uit het vorige antwoord.

Antwoord:

- Inhoud: ZIP-archief.
- Elke geëxporteerde Skill heeft `{publisher}/{slug}/` als hoofdmap.
- Gehoste Skills bevatten de bestanden van de laatst opgeslagen versie en worden vermeld in
  `_manifest.json` met `sourceRef: "public-clawhub"`.
- Huidige door GitHub ondersteunde Skills met een `clean`- of `suspicious`-scan bevatten
  `_source_handoff.json` met `sourceRef: "public-github"`, repository, commit, pad,
  inhoudshash en archief-URL. Ze bevatten geen door ClawHub gehoste bronbestanden.
- Elke Skill bevat `_export_skill_meta.json`.
- `_manifest.json` wordt altijd opgenomen in de hoofdmap van het ZIP-bestand.
- `_errors.json` wordt opgenomen wanneer afzonderlijke Skills of bestanden niet konden worden
  geëxporteerd.

Headers:

- `X-Next-Cursor`
- `X-Has-More`
- `X-Total-Returned`
- `X-Date-Range`
- `X-Export-Errors`

### `GET /api/v1/plugins/export`

Bulkexport van de nieuwste openbare Plugin-releases voor offlineanalyse.

Authenticatie:

- API-token vereist.

Queryparameters:

- `startDate` (verplicht): ondergrens in Unix-milliseconden voor `updatedAt` van de Plugin.
- `endDate` (verplicht): bovengrens in Unix-milliseconden voor `updatedAt` van de Plugin.
- `limit` (optioneel): geheel getal (1-250), standaard `250`.
- `cursor` (optioneel): pagineringscursor uit het vorige antwoord.
- `family` (optioneel): `code-plugin` of `bundle-plugin`. Weglaten betekent beide
  Plugin-families.

Antwoord:

- Inhoud: ZIP-archief.
- Elke geëxporteerde Plugin heeft `{family}/{packageName}/` als hoofdmap.
- Elke geëxporteerde Plugin bevat de opgeslagen bestanden van de nieuwste release.
- Exportmetadata per Plugin wordt opgeslagen in
  `__clawhub_export/{family}/{packageName}/plugin_meta.json`.
- `_manifest.json` wordt altijd opgenomen in de hoofdmap van het ZIP-bestand.
- `_errors.json` wordt opgenomen wanneer afzonderlijke Plugins of bestanden niet konden worden
  geëxporteerd.

Headers:

- `X-Next-Cursor`
- `X-Has-More`
- `X-Total-Returned`
- `X-Date-Range`
- `X-Export-Errors`

### `GET /api/v1/plugins/search`

Zoeken, uitsluitend naar Plugins, in code-Plugin- en bundel-Plugin-pakketten.

Queryparameters:

- `q` (verplicht): querytekenreeks
- `limit` (optioneel): geheel getal (1-100)
- `isOfficial` (optioneel): `true` of `false`
- `category` (optioneel): filter voor Plugin-categorieën. Huidige waarden:
  `channels`, `models`, `memory`, `context`, `voice`, `media`, `web`,
  `tools`, `runtime`, `gateway`, `security`, `other`.

Opmerkingen:

- De verouderde v1-filteraliassen die onder `GET /api/v1/plugins` zijn gedocumenteerd, worden ook
  geaccepteerd.
- Categoriefiltering is een echt API-filter dat wordt ondersteund door digest-
  rijen voor Plugin-categorieën, niet door een herschrijving van de zoekquery.
- Resultaten worden in volgorde van relevantie geretourneerd en worden momenteel niet gepagineerd.
- Sorteerknoppen in de browserinterface voor het zoeken naar Plugins herschikken de geladen relevantieresultaten,
  overeenkomstig het huidige bladergedrag van `/skills`.

### `GET /api/v1/packages/{name}`

Retourneert detailmetadata van het pakket.

Opmerkingen:

- Skills kunnen in de uniforme catalogus ook via deze route worden herleid.
- Privépakketten retourneren `404`, tenzij de aanroeper de eigenaar-uitgever kan lezen.

### `DELETE /api/v1/packages/{name}`

Verwijdert een pakket en alle releases via een zachte verwijdering.

Opmerkingen:

- Vereist een API-token voor de pakketeigenaar, een eigenaar/beheerder van de organisatie-uitgever,
  platformmoderator of platformbeheerder.

### `GET /api/v1/packages/{name}/versions`

Retourneert de versiegeschiedenis.

Queryparameters:

- `limit` (optioneel): geheel getal (1–100)
- `cursor` (optioneel): pagineringscursor

Opmerkingen:

- Privépakketten retourneren `404`, tenzij de aanroeper de eigenaar-uitgever kan lezen.

### `GET /api/v1/packages/{name}/versions/{version}`

Retourneert één pakketversie, inclusief bestandsmetadata, compatibiliteit,
verificatie, artefactmetadata en scangegevens.

Opmerkingen:

- `version.artifact.kind` is `legacy-zip` voor pakketarchieven uit het oude systeem of
  `npm-pack` voor door ClawPack ondersteunde releases.
- ClawPack-releases bevatten npm-compatibele velden `npmIntegrity`, `npmShasum` en
  `npmTarballName`.
- `version.sha256hash` is verouderde compatibiliteitsmetadata voor oude clients. Deze
  hasht exact de ZIP-bytes die door `/api/v1/packages/{name}/download` worden geretourneerd.
  Moderne clients moeten `version.artifact.sha256` gebruiken, waarmee het
  canonieke releaseartefact wordt geïdentificeerd.
- `version.vtAnalysis`, `version.llmAnalysis` en `version.staticScan` worden
  opgenomen wanneer scangegevens bestaan.
- Privépakketten retourneren `404`, tenzij de aanroeper de eigenaar-uitgever kan lezen.

### `GET /api/v1/packages/{name}/versions/{version}/security`

Retourneert het exacte beveiligings- en vertrouwensoverzicht van de pakketrelease voor installatie-
clients. Dit is het openbare OpenClaw-gebruiksoppervlak om te bepalen of een
herleide release kan worden geïnstalleerd.

Authenticatie:

- Openbaar leeseindpunt. Er is geen token van een eigenaar, uitgever, moderator of beheerder
  vereist.

Antwoord:

```json
{
  "package": {
    "name": "@openclaw/example-plugin",
    "displayName": "Voorbeeld-Plugin",
    "family": "code-plugin"
  },
  "release": {
    "releaseId": "packageReleases:...",
    "version": "1.2.3",
    "artifactKind": "npm-pack",
    "artifactSha256": "0123456789abcdef...",
    "npmIntegrity": "sha512-...",
    "npmShasum": "0123456789abcdef0123456789abcdef01234567",
    "npmTarballName": "example-plugin-1.2.3.tgz",
    "createdAt": 1730000000000
  },
  "trust": {
    "scanStatus": "malicious",
    "moderationState": "quarantined",
    "blockedFromDownload": true,
    "reasons": ["manual:quarantined", "scan:malicious"],
    "pending": false,
    "stale": false
  }
}
```

Antwoordvelden:

- `package.name`, `package.displayName` en `package.family` identificeren het
  herleide registerpakket.
- `release.releaseId`, `release.version` en `release.createdAt` identificeren de
  exacte release die is beoordeeld.
- `release.artifactKind`, `release.artifactSha256`, `release.npmIntegrity`,
  `release.npmShasum` en `release.npmTarballName` zijn aanwezig wanneer ze bekend zijn voor
  het releaseartefact.
- `trust.scanStatus` is de effectieve vertrouwensstatus die is afgeleid van scannerinvoer
  en handmatige releasemoderatie.
- `trust.moderationState` mag null zijn. De waarde is `null` wanneer er geen handmatige
  releasemoderatie bestaat.
- `trust.blockedFromDownload` is het blokkeringssignaal voor installatie. OpenClaw en andere
  installatieclients moeten de installatie blokkeren wanneer deze waarde `true` is, in plaats van
  blokkeringsregels opnieuw af te leiden uit scanner- of moderatievelden.
- `trust.reasons` is de lijst met uitleg voor gebruikers en audits. Redencodes
  zijn stabiele, compacte tekenreeksen, zoals `manual:quarantined`, `scan:malicious`
  en `package:malicious`.
- `trust.pending` betekent dat een of meer vertrouwensinvoerwaarden nog op voltooiing wachten.
- `trust.stale` betekent dat het vertrouwensoverzicht is berekend op basis van verouderde invoer en
  moet worden behandeld alsof vernieuwing vereist is voordat met hoge zekerheid een toestemmingsbesluit wordt genomen.

Opmerkingen:

- Dit eindpunt is versiespecifiek. Clients moeten het aanroepen nadat ze de
  pakketversie hebben herleid die ze willen installeren, niet alleen nadat ze de nieuwste
  pakketmetadata hebben gelezen.
- Privépakketten retourneren `404`, tenzij de aanroeper de eigenaar-uitgever kan lezen.
- Dit eindpunt is opzettelijk beperkter dan moderatie-eindpunten voor eigenaren/moderators.
  Het stelt de installatiebeslissing en openbare uitleg beschikbaar, niet
  de identiteit van melders, meldingsinhoud, privébewijs of interne beoordelings-
  tijdlijnen.

### `GET /api/v1/packages/{name}/versions/{version}/artifact`

Retourneert de expliciete metadata van de artefactresolver voor een pakketversie.

Opmerkingen:

- Verouderde pakketversies retourneren een `legacy-zip`-artefact en een verouderde ZIP-
  `downloadUrl`.
- ClawPack-versies retourneren een `npm-pack`-artefact, npm-integriteitsvelden, een
  `tarballUrl` en de verouderde ZIP-compatibiliteits-URL.
- Dit is het OpenClaw-resolveroppervlak; hiermee wordt voorkomen dat de archiefindeling wordt afgeleid uit
  een gedeelde URL.

### `GET /api/v1/packages/{name}/versions/{version}/artifact/download`

Downloadt het versieartefact via het expliciete resolverpad.

Opmerkingen:

- ClawPack-versies streamen exact de bytes van het geüploade npm-pack `.tgz`.
- Verouderde ZIP-versies leiden om naar `/api/v1/packages/{name}/download?version=`.
- Gebruikt de snelheidsbucket voor downloads.

### `GET /api/v1/packages/{name}/readiness`

Retourneert de berekende gereedheid voor toekomstig gebruik door OpenClaw.

Gereedheidscontroles omvatten:

- status van het officiële kanaal
- beschikbaarheid van de nieuwste versie
- beschikbaarheid van het ClawPack npm-pack-artefact
- artefactdigest
- herkomst van bronrepository en commit
- compatibiliteitsmetadata voor OpenClaw
- hostdoelen
- scanstatus

Respons:

```json
{
  "package": {
    "name": "@openclaw/example-plugin",
    "displayName": "Voorbeeldplugin",
    "family": "code-plugin",
    "isOfficial": true,
    "latestVersion": "1.2.3"
  },
  "ready": false,
  "checks": [
    {
      "id": "clawpack",
      "label": "ClawPack-artefact",
      "status": "fail",
      "message": "De nieuwste versie is uitsluitend beschikbaar als verouderd ZIP-bestand."
    }
  ],
  "blockers": ["clawpack"]
}
```

### `GET /api/v1/packages/migrations`

Moderatorendpoint voor het weergeven van migratieregels voor officiële OpenClaw-plugins.

Authenticatie:

- Vereist een API-token voor een moderator- of beheerdersgebruiker.

Queryparameters:

- `phase` (optioneel): `planned`, `published`, `clawpack-ready`,
  `legacy-zip-only`, `metadata-ready`, `blocked`, `ready-for-openclaw` of
  `all` (standaard).
- `limit` (optioneel): geheel getal (1-100)
- `cursor` (optioneel): pagineringscursor

Respons:

```json
{
  "items": [
    {
      "migrationId": "officialPluginMigrations:...",
      "bundledPluginId": "core.search",
      "packageName": "@openclaw/search-plugin",
      "packageId": "packages:...",
      "owner": "platform",
      "sourceRepo": "openclaw/openclaw",
      "sourcePath": "plugins/search",
      "sourceCommit": "abc123",
      "phase": "blocked",
      "blockers": ["ClawPack ontbreekt"],
      "hostTargetsComplete": true,
      "scanClean": false,
      "moderationApproved": false,
      "runtimeBundlesReady": false,
      "notes": null,
      "createdAt": 1760000000000,
      "updatedAt": 1760000000000
    }
  ],
  "nextCursor": null,
  "done": true
}
```

### `POST /api/v1/packages/migrations`

Beheerdersendpoint voor het maken of bijwerken van een migratieregel voor een officiële plugin.

Authenticatie:

- Vereist een API-token voor een beheerdersgebruiker.

Aanvraagbody:

```json
{
  "bundledPluginId": "core.search",
  "packageName": "@openclaw/search-plugin",
  "owner": "platform",
  "sourceRepo": "openclaw/openclaw",
  "sourcePath": "plugins/search",
  "sourceCommit": "abc123",
  "phase": "blocked",
  "blockers": ["ClawPack ontbreekt"],
  "hostTargetsComplete": true,
  "scanClean": false,
  "moderationApproved": false,
  "runtimeBundlesReady": false,
  "notes": "wacht op upload door uitgever"
}
```

Opmerkingen:

- `bundledPluginId` wordt genormaliseerd naar kleine letters en is de stabiele upsertsleutel.
- `packageName` wordt genormaliseerd als npm-naam; het pakket mag ontbreken voor geplande
  migraties.
- Dit houdt alleen de migratiegereedheid bij. Het wijzigt OpenClaw niet en genereert
  geen ClawPacks.

### `GET /api/v1/packages/moderation/queue`

Moderator-/beheerdersendpoint voor beoordelingswachtrijen van pakketreleases.

Authenticatie:

- Vereist een API-token voor een moderator- of beheerdersgebruiker.

Queryparameters:

- `status` (optioneel): `open` (standaard), `blocked`, `manual` of `all`
- `limit` (optioneel): geheel getal (1-100)
- `cursor` (optioneel): pagineringscursor

Betekenis van statussen:

- `open`: verdachte, schadelijke, in behandeling zijnde, in quarantaine geplaatste, ingetrokken of gemelde releases.
- `blocked`: in quarantaine geplaatste, ingetrokken of schadelijke releases.
- `manual`: elke release met een handmatige moderatie-override.
- `all`: elke release met een handmatige override, een niet-schone scanstatus of een pakketmelding.

Respons:

```json
{
  "items": [
    {
      "packageId": "packages:...",
      "releaseId": "packageReleases:...",
      "name": "@openclaw/example-plugin",
      "displayName": "Voorbeeldplugin",
      "family": "code-plugin",
      "channel": "community",
      "isOfficial": false,
      "version": "1.2.3",
      "createdAt": 1730000000000,
      "artifactKind": "npm-pack",
      "scanStatus": "malicious",
      "moderationState": "quarantined",
      "moderationReason": "handmatige beoordeling",
      "sourceRepo": "openclaw/example-plugin",
      "sourceCommit": "abc123",
      "reportCount": 2,
      "lastReportedAt": 1730000001000,
      "reasons": ["manual:quarantined", "scan:malicious", "reports:2"]
    }
  ],
  "nextCursor": null,
  "done": true
}
```

### `POST /api/v1/packages/{name}/report`

Meld een pakket voor beoordeling door een moderator. Meldingen gelden op pakketniveau en kunnen optioneel
aan een versie worden gekoppeld. Ze worden aan de moderatiewachtrij toegevoegd, maar verbergen of
blokkeren op zichzelf downloads niet automatisch; moderators moeten releasemoderatie gebruiken om
artefacten goed te keuren, in quarantaine te plaatsen of in te trekken.

Authenticatie:

- Vereist een API-token.

Aanvraag:

```json
{ "reason": "Verdacht systeemeigen binair bestand", "version": "1.2.3" }
```

Respons:

```json
{
  "ok": true,
  "reported": true,
  "alreadyReported": false,
  "packageId": "packages:...",
  "releaseId": "packageReleases:...",
  "reportCount": 1
}
```

### `GET /api/v1/packages/reports`

Moderator-/beheerdersendpoint voor de intake van pakketmeldingen.

Authenticatie:

- Vereist een API-token voor een moderator- of beheerdersgebruiker.

Queryparameters:

- `status` (optioneel): `open` (standaard), `confirmed`, `dismissed` of `all`
- `limit` (optioneel): geheel getal (1-100)
- `cursor` (optioneel): pagineringscursor

Respons:

```json
{
  "items": [
    {
      "reportId": "packageReports:...",
      "packageId": "packages:...",
      "releaseId": "packageReleases:...",
      "name": "@openclaw/example-plugin",
      "displayName": "Voorbeeldplugin",
      "family": "code-plugin",
      "version": "1.2.3",
      "reason": "Verdacht systeemeigen binair bestand",
      "status": "open",
      "createdAt": 1730000000000,
      "reporter": {
        "userId": "users:...",
        "handle": "reporter",
        "displayName": "Melder"
      },
      "triagedAt": null,
      "triagedBy": null,
      "triageNote": null
    }
  ],
  "nextCursor": null,
  "done": true
}
```

### `GET /api/v1/packages/{name}/moderation`

Eigenaar-/moderatorendpoint voor inzicht in pakketmoderatie.

Authenticatie:

- Vereist een API-token voor de pakketeigenaar, een lid van de uitgever, een moderator of
  een beheerdersgebruiker.

Respons:

```json
{
  "package": {
    "packageId": "packages:...",
    "name": "@openclaw/example-plugin",
    "displayName": "Voorbeeldplugin",
    "family": "code-plugin",
    "channel": "community",
    "isOfficial": false,
    "reportCount": 2,
    "lastReportedAt": 1730000001000,
    "scanStatus": "malicious"
  },
  "latestRelease": {
    "releaseId": "packageReleases:...",
    "version": "1.2.3",
    "artifactKind": "npm-pack",
    "scanStatus": "malicious",
    "moderationState": "quarantined",
    "moderationReason": "handmatige beoordeling",
    "blockedFromDownload": true,
    "reasons": ["manual:quarantined", "scan:malicious", "reports:2"],
    "createdAt": 1730000000000
  }
}
```

### `POST /api/v1/packages/reports/{reportId}/triage`

Moderator-/beheerdersendpoint voor het afhandelen of heropenen van pakketmeldingen.

Aanvraag:

```json
{
  "status": "confirmed",
  "note": "Beoordeeld en de betreffende release in quarantaine geplaatst.",
  "finalAction": "quarantine"
}
```

`note` is vereist voor `confirmed` en `dismissed`; het mag worden weggelaten wanneer
`status` wordt teruggezet naar `open`. Geef `finalAction: "quarantine"` of
`finalAction: "revoke"` door met een bevestigde melding om releasemoderatie toe te passen binnen
dezelfde controleerbare workflow.

Respons:

```json
{
  "ok": true,
  "reportId": "packageReports:...",
  "packageId": "packages:...",
  "status": "confirmed",
  "reportCount": 0
}
```

### `POST /api/v1/packages/{name}/versions/{version}/moderation`

Moderator-/beheerdersendpoint voor de beoordeling van pakketreleases.

Aanvraag:

```json
{ "state": "quarantined", "reason": "Verdachte systeemeigen payload." }
```

Ondersteunde statussen:

- `approved`: handmatig beoordeeld en toegestaan.
- `quarantined`: geblokkeerd in afwachting van vervolgactie.
- `revoked`: geblokkeerd nadat een release eerder werd vertrouwd.

In quarantaine geplaatste en ingetrokken releases retourneren `403` vanuit routes voor het downloaden van artefacten.
Elke wijziging schrijft een vermelding naar het auditlogboek.

### `GET /api/v1/packages/{name}/file`

Retourneert de exacte opgeslagen bytes van een pakketbestand als download. Voeg `preview=1` toe om dezelfde begrensde
UTF-8-tekstvoorvertoning aan te vragen die voor skillbestanden wordt gebruikt.

Queryparameters:

- `path` (vereist)
- `version` (optioneel)
- `tag` (optioneel)
- `preview=1` (optioneel; retourneert `text/plain` of `415` wanneer de bytes geen geldige UTF-8 vormen)

Opmerkingen:

- Gebruikt standaard de nieuwste release.
- Gebruikt de snelheidsbucket voor lezen, niet die voor downloads.
- Limiet voor onbewerkte downloads: 10MB.
- Limiet voor tekstvoorvertoning: 200KB; ondoorzichtige bestanden retourneren alleen bij voorvertoningsaanvragen `415`.
- Lopende VirusTotal-scans blokkeren leesacties niet; schadelijke releases kunnen elders nog steeds worden achtergehouden.
- Privépakketten retourneren `404`, tenzij de aanroeper de eigenaar-uitgever mag lezen.

### `GET /api/v1/packages/{name}/download`

Downloadt het verouderde deterministische ZIP-archief voor een pakketrelease.

Queryparameters:

- `version` (optioneel)
- `tag` (optioneel)

Opmerkingen:

- Gebruikt standaard de nieuwste release.
- Skills leiden om naar `GET /api/v1/download`.
- Plugin-/pakketarchieven zijn ZIP-bestanden met een `package/`-hoofdmap, zodat oude OpenClaw-
  clients blijven werken.
- Deze route blijft uitsluitend ZIP ondersteunen. De route streamt geen ClawPack `.tgz`-bestanden.
- Responsen bevatten de headers `ETag`, `Digest`, `X-ClawHub-Artifact-Type` en
  `X-ClawHub-Artifact-Sha256` voor integriteitscontroles door de resolver.
- Metadata die alleen voor het register bestemd is, wordt niet in het gedownloade archief geïnjecteerd.
- Lopende VirusTotal-scans blokkeren downloads niet; schadelijke releases retourneren `403`.
- Privépakketten retourneren `404`, tenzij de aanroeper de eigenaar is.

### `GET /api/npm/{package}`

Retourneert een npm-compatibele packument voor pakketversies die door ClawPack worden ondersteund.

Opmerkingen:

- Alleen versies met geüploade ClawPack npm-pack-tarballs worden vermeld.
- Verouderde versies die uitsluitend als ZIP beschikbaar zijn, worden bewust weggelaten.
- `dist.tarball`, `dist.integrity` en `dist.shasum` gebruiken npm-compatibele
  velden, zodat gebruikers npm desgewenst naar de mirror kunnen verwijzen.
- Packuments van pakketten met een scope ondersteunen zowel `/api/npm/@scope/name` als het gecodeerde
  aanvraagpad `/api/npm/@scope%2Fname` van npm.

### `GET /api/npm/{package}/-/{tarball}.tgz`

Streamt de exacte bytes van de geüploade ClawPack-tarball voor npm-mirrorclients.

Opmerkingen:

- Gebruikt de snelheidsbucket voor downloads.
- Downloadheaders bevatten de ClawHub SHA-256 plus npm-metadata voor integriteit en shasum.
- Controles voor moderatie en toegang tot privépakketten blijven van toepassing.

### `GET /api/v1/resolve`

Wordt door de CLI gebruikt om een lokale vingerafdruk aan een bekende versie te koppelen.

Queryparameters:

- `slug` (vereist)
- `hash` (vereist): hexadecimale sha256 van 64 tekens van de bundelvingerafdruk

Respons:

```json
{ "slug": "gifgrep", "match": { "version": "1.2.2" }, "latestVersion": { "version": "1.2.3" } }
```

### `GET /api/v1/download`

Downloadt een ZIP-bestand van een gehoste skillversie, of retourneert een overdracht naar GitHub-broncode voor een
huidige door GitHub ondersteunde skill met een `clean`- of `suspicious`-scan en zonder gehoste
versie.

Queryparameters:

- `slug` (verplicht)
- `version` (optioneel): semver-tekenreeks
- `tag` (optioneel): tagnaam (bijv. `latest`)

Opmerkingen:

- Als noch `version` noch `tag` is opgegeven, wordt de nieuwste versie gebruikt.
- Voor zacht verwijderde versies wordt `410` geretourneerd.
- Overdrachten voor door GitHub ondersteunde skills proxyen of spiegelen geen bytes. Het JSON-antwoord
  bevat `sourceRef: "public-github"`, `repo`, `commit`, `path`, `contentHash`
  en `archiveUrl`; de scan-/huidige status is een toegangspoort en wordt niet als metagegevens
  van de geslaagde payload opgenomen.
- Downloadstatistieken worden geteld als unieke identiteiten per UTC-dag (`userId` wanneer het API-token geldig is, anders het IP-adres).

## Authenticatie-eindpunten (Bearer-token)

Alle eindpunten vereisen:

```
Authorization: Bearer clh_...
```

### `GET /api/v1/whoami`

Valideert het token en retourneert de gebruikershandle.

### `POST /api/v1/skills`

Publiceert een nieuwe versie.

- Bij voorkeur: `multipart/form-data` met `payload`-JSON + `files[]`-blobs.
- Een JSON-body met `files` (op basis van storageId) wordt ook geaccepteerd.
- Optioneel payloadveld: `ownerHandle`. Indien aanwezig, bepaalt de API die
  uitgever aan serverzijde en moet de actor uitgeverstoegang hebben.
- Optioneel payloadveld: `migrateOwner`. Bij `true` met `ownerHandle` kan een
  bestaande skill naar die eigenaar worden verplaatst als de actor beheerder/eigenaar is bij zowel
  de huidige als de doeluitgever. Zonder deze expliciete toestemming worden eigenaarswijzigingen
  geweigerd.

### `POST /api/v1/packages`

Publiceert een release van een codeplugin of bundelplugin.

- Vereist authenticatie met een Bearer-token.
- Vereist `multipart/form-data`.
- Toegestane formuliervelden zijn `payload`, herhaalde `files`-blobs of één `clawpack`-
  tarballverwijzing. `clawpack` kan een `.tgz`-blob zijn of een opslag-ID dat door
  de upload-URL-flow is geretourneerd. Gefaseerde publicaties met een opslag-ID moeten ook
  de `clawpackUploadTicket` bevatten die met die upload-URL is geretourneerd.
- Gebruik `files` of `clawpack`, nooit beide in hetzelfde verzoek.
- JSON-bodies en door de aanroeper aangeleverde `payload.files`- / `payload.artifact`-
  metagegevens worden geweigerd.
- Rechtstreekse meerdelige publicatieverzoeken zijn beperkt tot 18MB. ClawPack-tarballs kunnen
  de upload-URL-flow gebruiken tot de tarballlimiet van 120MB.
- Optioneel payloadveld: `ownerHandle`. Indien aanwezig, mogen alleen beheerders namens die eigenaar publiceren.

Belangrijkste validaties:

- `family` moet `code-plugin` of `bundle-plugin` zijn.
- Pluginpakketten vereisen `openclaw.plugin.json`. ClawPack-uploads van `.tgz` moeten
  dit bevatten op `package/openclaw.plugin.json`.
- Codeplugins vereisen `package.json`, metagegevens van de bronrepository, metagegevens
  van de broncommit, metagegevens van het configuratieschema, `openclaw.compat.pluginApi` en
  `openclaw.build.openclawVersion`.
- `openclaw.hostTargets` en `openclaw.environment` zijn optionele metagegevens.
- Alleen de organisatie-uitgever `openclaw` en de persoonlijke uitgevers van huidige
  leden van de organisatie `openclaw` mogen naar het kanaal `official` publiceren.
- Bij publicaties namens anderen wordt de geschiktheid voor het officiële kanaal nog steeds aan de hand van het doeleigenaarsaccount gevalideerd.

### `DELETE /api/v1/skills/{slug}` / `POST /api/v1/skills/{slug}/undelete`

Een skill zacht verwijderen / herstellen (eigenaar, moderator of beheerder).

Optionele JSON-body:

```json
{ "reason": "Vastgehouden voor moderatie in afwachting van juridische beoordeling." }
```

Indien aanwezig, wordt `reason` opgeslagen als moderatienotitie van de skill en naar het auditlogboek gekopieerd.
Door de eigenaar geïnitieerde zachte verwijderingen reserveren de slug 30 dagen, waarna een
andere uitgever de slug kan claimen. Het verwijderingsantwoord bevat `slugReservedUntil` wanneer deze vervaldatum van toepassing is.
Verbergingen door moderators/beheerders en verwijderingen om veiligheidsredenen verlopen niet op deze manier.

Verwijderingsantwoord:

```json
{ "ok": true, "slugReservedUntil": 1730000000000 }
```

Statuscodes:

- `200`: geslaagd
- `401`: niet geautoriseerd
- `403`: verboden
- `404`: skill/gebruiker niet gevonden
- `500`: interne serverfout

### `POST /api/v1/users/publisher`

Alleen voor beheerders. Zorgt dat er een organisatie-uitgever voor een handle bestaat. Als de handle nog naar een
oude gedeelde gebruiker/persoonlijke uitgever verwijst, migreert het eindpunt deze eerst naar een organisatie-uitgever.
Geef voor een nieuw aangemaakte organisatie `memberHandle` op; de handelende beheerder wordt niet als lid toegevoegd.
`memberRole` is standaard `owner`.

- Body: `{ "handle": "openclaw", "displayName": "OpenClaw", "memberHandle": "alice", "memberRole": "owner", "trusted": true }`
- Antwoord: `{ "ok": true, "publisherId": "...", "handle": "openclaw", "created": true, "migrated": false, "trusted": true, "member": { "userId": "...", "handle": "alice", "role": "owner" } }`

### `POST /api/v1/publishers`

Geauthenticeerde selfservice voor het aanmaken van een organisatie-uitgever. Maakt een nieuwe organisatie-uitgever aan en voegt de
aanroeper als eigenaar toe. Dit eindpunt migreert geen bestaande gebruikers-/persoonlijke handles en
markeert de uitgever niet als vertrouwd/officieel.

- Body: `{ "handle": "opik", "displayName": "Opik" }`
- Antwoord: `{ "ok": true, "publisherId": "...", "handle": "opik", "created": true, "trusted": false }`
- Retourneert `409` wanneer de handle al door een uitgever, gebruiker of persoonlijke uitgever wordt gebruikt.

### `POST /api/v1/users/reserve`

Alleen voor beheerders. Reserveert hoofdslugs en pakketnamen voor de rechtmatige eigenaar zonder een
release te publiceren. Pakketnamen worden persoonlijke tijdelijke pakketten zonder releaseregels, zodat dezelfde
eigenaar later de echte release van de codeplugin of bundelplugin onder die naam kan publiceren.

- Body: `{ "handle": "openclaw", "slugs": ["diffs"], "packageNames": ["@openclaw/diffs"], "reason": "reserved for official OpenClaw plugin" }`
- Antwoord: `{ "ok": true, "succeeded": 2, "failed": 0, "results": [{ "kind": "slug", "name": "diffs", "ok": true, "action": "reserved" }] }`

### `POST /api/v1/users/publisher-recovery`

Alleen voor beheerders. Herstelt een persoonlijke uitgever voor een geverifieerde vervangende GitHub OAuth-principal
zonder Convex Auth-accountregels te bewerken. Het verzoek moet beide onveranderlijke GitHub-
provideraccount-ID's noemen; veranderlijke handles worden alleen gebruikt als beveiliging voor de operator.

Het eindpunt voert standaard een testuitvoering uit. Voor het toepassen van het herstel zijn `dryRun: false` en
`confirmIdentityVerified: true` vereist nadat medewerkers onafhankelijk de continuïteit tussen beide
GitHub-principals hebben geverifieerd. Het herstel mislukt veilig wanneer de huidige persoonlijke
uitgever van de doelgebruiker skills, pakketten of GitHub-skillbronnen heeft.
Het herstel migreert ook oude `ownerUserId`-velden voor de skills van de herstelde uitgever,
skill-slugaliassen, pakketten, pakketinspectiewaarschuwingen en afgeleide zoekdigestregels, zodat
paden voor directe eigenaren overeenkomen met de nieuwe uitgeversbevoegdheid. Een actieve reservering
voor de beveiligde handle van de herstelde handle wordt ook opnieuw toegewezen aan de vervangende gebruiker, zodat latere
profielsynchronisatie de concurrerende bevoegdheid van de voormalige gebruiker niet kan herstellen. Elke primaire tabel is beperkt tot
100 regels per toepassingstransactie; grotere herstelacties moeten eerst een hervatbare eigenaarsmigratie gebruiken.
GitHub-skillbronnen vallen onder de uitgever en worden als gecontroleerd gerapporteerd in plaats van herschreven.

- Body: `{ "handle": "gingiris", "nextUserHandle": "gingiris-1031", "previousGitHubProviderAccountId": "123", "nextGitHubProviderAccountId": "456", "reason": "Verified account continuity for issue #2555", "confirmIdentityVerified": true, "dryRun": false }`
- Antwoord: `{ "ok": true, "dryRun": false, "recovered": true, "publisherId": "...", "handle": "gingiris", "previousUser": { "userId": "...", "handle": "gingiris", "nextHandle": "gingiris-recovered", "githubProviderAccountId": "123", "authAccountCount": 1 }, "nextUser": { "userId": "...", "handle": "gingiris-1031", "nextHandle": "gingiris", "githubProviderAccountId": "456", "authAccountCount": 1 }, "retiredPersonalPublisher": null, "resourceOwnerMigration": { "limitPerTable": 100, "skills": 1, "skillSlugAliases": 1, "packages": 0, "packageInspectorWarnings": 0, "githubSourcesChecked": 1, "handleReservations": 1 }, "identityVerified": true, "reason": "Verified account continuity for issue #2555" }`

### Eindpunten voor beheer van eigenaars-slugs

- `POST /api/v1/skills/{slug}/rename`
  - Body: `{ "newSlug": "new-canonical-slug" }`
  - Antwoord: `{ "ok": true, "slug": "new-canonical-slug", "previousSlug": "old-slug" }`
- `POST /api/v1/skills/{slug}/merge`
  - Body: `{ "targetSlug": "canonical-target-slug" }`
  - Antwoord: `{ "ok": true, "sourceSlug": "old-slug", "targetSlug": "canonical-target-slug" }`

Opmerkingen:

- Beide eindpunten vereisen authenticatie met een API-token en werken alleen voor de eigenaar van de skill.
- `rename` behoudt de vorige slug als omleidingsalias.
- `merge` verbergt de bronvermelding en leidt de bronslug om naar de doelvermelding.

### Eindpunten voor eigendomsoverdracht

- `POST /api/v1/skills/{slug}/transfer`
  - Body: `{ "toUserHandle": "target_handle", "message": "optional" }`
  - Antwoord: `{ "ok": true, "transferId": "skillOwnershipTransfers:...", "toUserHandle": "target_handle", "expiresAt": 1730000000000 }`
- `POST /api/v1/skills/{slug}/transfer/accept`
- `POST /api/v1/skills/{slug}/transfer/reject`
- `POST /api/v1/skills/{slug}/transfer/cancel`
  - Antwoord (accepteren/weigeren/annuleren): `{ "ok": true, "skillSlug": "demo-skill?" }`
- `GET /api/v1/transfers/incoming`
- `GET /api/v1/transfers/outgoing`
  - Antwoordstructuur: `{ "transfers": [{ "_id": "...", "skill": { "slug": "demo", "displayName": "Demo" }, "fromUser"|"toUser": { "handle": "..." }, "message": "...", "requestedAt": 0, "expiresAt": 0 }] }`

### `POST /api/v1/users/ban`

Een gebruiker verbannen en skills waarvan deze eigenaar is permanent verwijderen (alleen moderator/beheerder).

Body:

```json
{ "handle": "user_handle", "reason": "optionele reden voor verbanning" }
```

of

```json
{ "userId": "users_...", "reason": "optionele reden voor verbanning" }
```

Antwoord:

```json
{ "ok": true, "alreadyBanned": false, "deletedSkills": 3 }
```

### `POST /api/v1/users/unban`

De verbanning van een gebruiker opheffen en in aanmerking komende skills herstellen (alleen beheerder).

Body:

```json
{ "handle": "user_handle", "reason": "optionele reden voor opheffing van verbanning" }
```

of

```json
{ "userId": "users_...", "reason": "optionele reden voor opheffing van verbanning" }
```

Antwoord:

```json
{ "ok": true, "alreadyUnbanned": false, "restoredSkills": 3 }
```

### `POST /api/v1/users/reclassify-ban`

De opgeslagen reden voor een bestaande verbanning wijzigen zonder de verbanning op te heffen of
inhoud te herstellen (alleen beheerder). Voert standaard een testuitvoering uit, tenzij `dryRun` gelijk is aan `false`.

Body:

```json
{ "handle": "user_handle", "reason": "spam door massapublicatie", "dryRun": true }
```

of

```json
{ "userId": "users_...", "reason": "spam door massapublicatie", "dryRun": false }
```

Antwoord:

```json
{
  "ok": true,
  "dryRun": false,
  "userId": "users_...",
  "handle": "user_handle",
  "previousReason": "automatische verbanning wegens malware",
  "nextReason": "spam door massapublicatie",
  "changed": true
}
```

### `POST /api/v1/users/role`

Een gebruikersrol wijzigen (alleen beheerder).

Body:

```json
{ "handle": "user_handle", "role": "moderator" }
```

of

```json
{ "userId": "users_...", "role": "admin" }
```

Antwoord:

```json
{ "ok": true, "role": "moderator" }
```

### `GET /api/v1/users`

Gebruikers weergeven of zoeken (alleen beheerder).

Queryparameters:

- `q` (optioneel): zoekopdracht
- `query` (optioneel): alias voor `q`
- `limit` (optioneel): maximumaantal resultaten (standaard 20, maximaal 200)

Antwoord:

```json
{
  "items": [
    {
      "userId": "users_...",
      "handle": "user_handle",
      "displayName": "Gebruiker",
      "name": "Gebruiker",
      "role": "moderator"
    }
  ],
  "total": 1
}
```

### `POST /api/v1/stars/{slug}` / `DELETE /api/v1/stars/{slug}`

Een bladwijzer toevoegen/verwijderen. De oude `stars`-route en namen van antwoordvelden blijven
beschikbaar voor compatibiliteit. Beide eindpunten zijn idempotent.

Antwoorden:

```json
{ "ok": true, "starred": true, "alreadyStarred": false }
```

```json
{ "ok": true, "unstarred": true, "alreadyUnstarred": false }
```

## Oude CLI-eindpunten (afgeschaft)

Nog steeds ondersteund voor oudere CLI-versies:

- `GET /api/cli/whoami`
- `POST /api/cli/upload-url`
- `POST /api/cli/publish`
- `POST /api/cli/telemetry/install`
- `POST /api/cli/skill/delete`
- `POST /api/cli/skill/undelete`

Zie `DEPRECATIONS.md` voor het verwijderingsplan.

`POST /api/cli/upload-url` retourneert `uploadUrl` en `uploadTicket`. Pakketpublicaties
die een ClawPack-tarball klaarzetten, moeten het resulterende opslag-ID als
`clawpack` en het geretourneerde ticket als `clawpackUploadTicket` verzenden.

## Registerdetectie (`/.well-known/clawhub.json`)

De CLI kan register-/authenticatie-instellingen via de site vinden:

- `/.well-known/clawhub.json` (JSON, bij voorkeur)
- `/.well-known/clawdhub.json` (oud)

Schema:

```json
{ "apiBase": "https://clawhub.ai", "authBase": "https://clawhub.ai", "minCliVersion": "0.0.5" }
```

Als je zelf host, bied je dit bestand aan (of stel je `CLAWHUB_REGISTRY` expliciet in; voorheen `CLAWDHUB_REGISTRY`).
