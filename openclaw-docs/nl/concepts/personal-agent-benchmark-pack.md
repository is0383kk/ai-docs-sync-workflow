---
read_when:
    - Betrouwbaarheidscontroles voor een lokale persoonlijke agent uitvoeren
    - De QA-scenariocatalogus in de repository uitbreiden
    - Herinneringen, antwoorden, geheugen, redactie, veilige opvolging van tools, taakstatus, veilig deelbare diagnostiek, door bewijs onderbouwde voltooiingsclaims en herstel na fouten verifiëren
summary: Lokale qa-channel-scenario's voor controles van privacybeschermende persoonlijke-assistentworkflows.
title: Benchmarkpakket voor persoonlijke agents
x-i18n:
    generated_at: "2026-07-27T05:03:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 35da45e4b22b1044a777fa8d6bce87f9ace377950dd0af3f2419b40cfe4d9be6
    source_path: concepts/personal-agent-benchmark-pack.md
    workflow: 16
---

Het Personal Agent Benchmark Pack is een klein, door een repository ondersteund QA-scenariopakket voor
lokale workflows voor persoonlijke assistenten. Het is geen generieke modelbenchmark en
heeft geen nieuwe runner nodig: het hergebruikt de privé-QA-stack ([QA-overzicht](/nl/concepts/qa-e2e-automation)),
het synthetische [QA-kanaal](/nl/channels/qa-channel) en de bestaande
`qa/scenarios` YAML-catalogus.

## Scenario's

Tien scenario's, gedefinieerd in `qa/scenarios/personal/*.yaml`:

| Scenario-id                                | Controles                                                                                    |
| ------------------------------------------ | -------------------------------------------------------------------------------------------- |
| `personal-reminder-roundtrip`              | Fictieve persoonlijke herinneringen via lokale Cron-bezorging                                |
| `personal-channel-thread-reply`            | Routering van fictieve DM's en antwoorden in threads via `qa-channel`                       |
| `personal-memory-preference-recall`        | Ophalen van fictieve voorkeuren uit de tijdelijke geheugenbestanden van de QA-werkruimte     |
| `personal-redaction-no-secret-leak`        | Controles die verifiëren dat fictieve geheimen niet worden herhaald                          |
| `personal-tool-safety-followthrough`       | Veilige, door leesacties ondersteunde opvolging met tools na een korte goedkeuringsdialoog    |
| `personal-approval-denial-stop`            | Stopgedrag bij geweigerde goedkeuring voor een gevoelig lokaal leesverzoek                    |
| `personal-task-followthrough-status`       | Door bewijs ondersteunde rapportage van taakstatussen die in behandeling, geblokkeerd en voltooid gescheiden houdt |
| `personal-share-safe-diagnostics-artifact` | Veilig deelbare diagnostische artefacten die nuttige statusinformatie behouden en onbewerkte persoonlijke inhoud weglaten |
| `personal-no-fake-progress`                | Door bewijs ondersteunde voltooiingsclaims die fictieve voortgang voorkomen zolang lokaal bewijs ontbreekt |
| `personal-failure-recovery`                | Herstel na fouten dat een gedeeltelijke status rapporteert en de grenzen voor nieuwe pogingen duidelijk houdt |

De machineleesbare pakketmetadata (lijst met id's, titel, beschrijving) staat in
`extensions/qa-lab/src/scenario-packs.ts` als `QA_PERSONAL_AGENT_SCENARIO_IDS`.
Voer het pakket uit met `--pack personal-agent`:

```bash
OPENCLAW_ENABLE_PRIVATE_QA_CLI=1 pnpm openclaw qa suite \
  --provider-mode mock-openai \
  --pack personal-agent \
  --concurrency 1
```

`--pack` is additief bij herhaalde `--scenario`-vlaggen. Expliciete scenario's worden
eerst uitgevoerd, waarna de pakketscenario's in de volgorde van `QA_PERSONAL_AGENT_SCENARIO_IDS` worden uitgevoerd,
waarbij duplicaten worden verwijderd.

Het pakket is gericht op `qa-channel` met `mock-openai` of een andere lokale QA-providerlane.
Richt het niet op livechatdiensten of echte persoonlijke accounts.

## Privacymodel

Scenario's gebruiken uitsluitend fictieve gebruikers, fictieve voorkeuren, fictieve geheimen en de
tijdelijke QA Gateway-werkruimte die door de suite wordt aangemaakt. Ze mogen geen echt
OpenClaw-gebruikersgeheugen, sessies, inloggegevens, launch agents, globale
configuraties of live Gateway-status lezen of schrijven.

Artefacten blijven in de bestaande artefactmap van de QA-suite en worden behandeld
als testuitvoer. Redactiecontroles gebruiken fictieve markeringen, zodat fouten veilig kunnen worden
geïnspecteerd en in issues kunnen worden vastgelegd.

## Het pakket uitbreiden

Voeg nieuwe `.yaml`-gevallen toe onder `qa/scenarios/personal/` en voeg vervolgens het scenario-id
toe aan `QA_PERSONAL_AGENT_SCENARIO_IDS`. Houd elk geval klein, lokaal en deterministisch
in `mock-openai`, en richt het op één gedrag van een persoonlijke assistent.

Goede kandidaten voor vervolgwerk: controles voor de export van geredigeerde trajecten,
controles voor uitsluitend lokale Plugin-workflows.

Voeg geen nieuwe runner, Plugin, afhankelijkheid, live transport of modelbeoordelaar toe
totdat de scenariocatalogus voldoende stabiele gevallen bevat om dat oppervlak te rechtvaardigen.
