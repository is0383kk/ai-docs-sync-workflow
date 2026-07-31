---
read_when:
    - Je wilt TaskFlows activeren of aansturen vanuit een extern systeem
    - Je configureert de meegeleverde webhooks-plugin
summary: 'Webhooks-plugin: geverifieerde TaskFlow-ingang voor vertrouwde externe automatisering'
title: Webhooks-Plugin
x-i18n:
    generated_at: "2026-07-27T06:03:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 77e455450d6183635c76a1e8002feeb287deb4ff242dbd555ef9d0f2b21ce5f6
    source_path: plugins/webhooks.md
    workflow: 16
---

De Webhooks-plugin voegt geauthenticeerde HTTP-routes toe, zodat een vertrouwd extern
systeem (Zapier, n8n, een CI-taak, een interne service) beheerde OpenClaw
TaskFlows via HTTP kan aanmaken en aansturen, zonder een aangepaste plugin te schrijven.

De plugin wordt uitgevoerd binnen het Gateway-proces. Installeer en configureer
de plugin voor een externe Gateway op die host en start vervolgens de Gateway opnieuw. De plugin wordt zonder
geconfigureerde routes geleverd en doet dus niets totdat je ten minste één route toevoegt.

## Routes configureren

Stel de configuratie in onder `plugins.entries.webhooks.config`:

```json5
{
  plugins: {
    entries: {
      webhooks: {
        enabled: true,
        config: {
          routes: {
            zapier: {
              path: "/plugins/webhooks/zapier",
              sessionKey: "agent:main:main",
              secret: {
                source: "env",
                provider: "default",
                id: "OPENCLAW_WEBHOOK_SECRET",
              },
              controllerId: "webhooks/zapier",
              description: "Zapier TaskFlow bridge",
            },
          },
        },
      },
    },
  },
}
```

Routevelden:

| Veld           | Vereist | Standaard                     | Opmerkingen                                   |
| -------------- | -------- | ----------------------------- | --------------------------------------------- |
| `enabled`      | nee      | `true`                        |                                               |
| `path`         | nee      | `/plugins/webhooks/<routeId>` | Moet uniek zijn voor alle routes.             |
| `sessionKey`   | ja       | -                             | Sessie die eigenaar is van de gekoppelde TaskFlows. |
| `secret`       | ja       | -                             | Platte tekenreeks of een SecretRef (hieronder). |
| `controllerId` | nee      | `webhooks/<routeId>`          | Wordt gebruikt als de standaardcontroller voor `create_flow`. |
| `description`  | nee      | -                             | Alleen een opmerking voor de beheerder.       |

`secret` accepteert een platte tekenreeks of een SecretRef: `{ source: "env" | "file" | "exec", provider: "default", id: "..." }`.

SecretRefs worden omgezet in de momentopname van de opstartconfiguratie van de Gateway. Wanneer het
secret van één route niet kan worden omgezet, blijft de Gateway actief en blijft precies die route
geregistreerd maar inactief: verzoeken ontvangen een algemene authenticatiefout (`401`).
Andere routes blijven beschikbaar. Herstel de SecretRef-bron en laad of start vervolgens
de Gateway opnieuw om de nieuwe momentopname te activeren. SecretRef-waarden worden nooit omgezet
in het openbare aanvraagpad.

## Beveiligingsmodel

Elke route handelt met de TaskFlow-bevoegdheid van de geconfigureerde `sessionKey`: de route
kan elke TaskFlow waarvan die sessie eigenaar is inspecteren en wijzigen. TaskFlow-toegang
verloopt altijd via `api.runtime.tasks.managedFlows.bindSession(...)`, zodat een
route nooit buiten de gekoppelde sessie kan handelen. Om de impact te beperken:

- Gebruik voor elke route een sterk, uniek secret.
- Geef de voorkeur aan een SecretRef boven een inline secret in platte tekst.
- Koppel routes aan de meest beperkte sessie die bij de workflow past.
- Stel alleen het specifieke webhookpad beschikbaar dat je nodig hebt.

Volgorde van de verwerking van verzoeken voor elk pad: controles van de HTTP-methode (alleen `POST`)
en `Content-Type: application/json`, vervolgens een frequentielimiet met een vast tijdvenster (120
verzoeken per venster van 60 seconden per sleutelcombinatie van pad en client-IP, met maximaal 4,096 bijgehouden
sleutels), vervolgens beperking van actieve verzoeken (8 gelijktijdige verzoeken per sleutel, met maximaal
4,096 bijgehouden sleutels), vervolgens authenticatie met het gedeelde secret en daarna het lezen van een JSON-body
van 256 KB met een limiet van 15 seconden. Verzoeken die niet slagen voor een eerdere controle bereiken
de latere controles nooit.

## Verzoekindeling

Verzend `POST`-verzoeken met `Content-Type: application/json` en
`Authorization: Bearer <secret>` of `x-openclaw-webhook-secret: <secret>`:

```bash
curl -X POST https://gateway.example.com/plugins/webhooks/zapier \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer YOUR_SHARED_SECRET' \
  -d '{"action":"create_flow","goal":"Review inbound queue"}'
```

## Ondersteunde acties

| Actie              | Doel                                                               |
| ------------------ | ------------------------------------------------------------------ |
| `create_flow`      | Maak een beheerde TaskFlow aan voor de sessie van de route.        |
| `get_flow`         | Haal één TaskFlow op aan de hand van de id.                         |
| `list_flows`       | Geef de TaskFlows voor de sessie van de route weer.                 |
| `find_latest_flow` | Haal de laatst bijgewerkte TaskFlow op.                             |
| `resolve_flow`     | Zoek een TaskFlow op aan de hand van een ondoorzichtig token.       |
| `get_task_summary` | Haal het taakoverzicht voor een TaskFlow op.                        |
| `set_waiting`      | Markeer een TaskFlow als wachtend, met optionele status-/wachtgegevens. |
| `resume_flow`      | Hervat een wachtende/geblokkeerde TaskFlow.                         |
| `finish_flow`      | Markeer een TaskFlow als voltooid.                                 |
| `fail_flow`        | Markeer een TaskFlow als mislukt.                                   |
| `request_cancel`   | Vraag coöperatieve annulering aan.                                 |
| `cancel_flow`      | Annuleer een TaskFlow (kan `202` retourneren als onderliggende taken nog actief zijn). |
| `run_task`         | Maak een beheerde onderliggende taak aan binnen een bestaande TaskFlow. |

Wijzigingsacties (`set_waiting`, `resume_flow`, `finish_flow`, `fail_flow`,
`request_cancel`) vereisen `flowId` en `expectedRevision` voor optimistische
gelijktijdigheidscontrole; een verouderde revisie retourneert `409 revision_conflict`.

### `create_flow`

```json
{
  "action": "create_flow",
  "goal": "Review inbound queue",
  "status": "queued",
  "notifyPolicy": "done_only"
}
```

### `run_task`

Toegestane waarden voor `runtime`: `subagent`, `acp`. `startedAt`, `lastEventAt` en
`progressSummary` zijn alleen geldig wanneer `status` gelijk is aan `"running"`; als je ze
met een andere status verzendt, wordt `400 invalid_request` geretourneerd.

```json
{
  "action": "run_task",
  "flowId": "flow_123",
  "runtime": "acp",
  "childSessionKey": "agent:main:acp:worker",
  "task": "Inspect the next message batch"
}
```

## Antwoordstructuur

```json
{
  "ok": true,
  "routeId": "zapier",
  "result": {}
}
```

```json
{
  "ok": false,
  "routeId": "zapier",
  "code": "not_found",
  "error": "TaskFlow not found.",
  "result": {}
}
```

Flow- en taakweergaven bevatten nooit metadata over de eigenaar of sessie, zodat antwoorden
de gekoppelde `sessionKey` van de route niet kunnen lekken. Waarden voor `code` zijn onder meer `not_found`,
`not_managed`, `revision_conflict`, `persist_failed`, `cancel_requested`,
`cancel_pending`, `terminal`, `invalid_request`, `request_rejected` en
actiespecifieke terugvalcodes (`mutation_rejected`, `create_rejected`,
`task_not_created`, `cancel_rejected`) wanneer een wijziging wordt geweigerd om een
reden die niet door de bovengenoemde benoemde codes wordt gedekt.

## Gerelateerd

- [Hooks](/nl/automation/hooks) - interne gebeurtenisgestuurde hooks tegenover deze HTTP-gebaseerde TaskFlow-bridge
- [Gateway-webhooks (`hooks.*`-configuratie)](/nl/automation/cron-jobs#webhooks) - afzonderlijke generieke HTTP-endpointfunctie van de Gateway; niet hetzelfde als de routes van deze plugin
- [Runtime-SDK voor plugins](/nl/plugins/sdk-runtime)
- [CLI-webhooks](/nl/cli/webhooks)
