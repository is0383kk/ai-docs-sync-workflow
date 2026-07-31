---
read_when:
    - Eigenaarschap van Canvas-host, tools, opdrachten, documentatie of protocol verplaatsen
    - Controleren of Canvas nog steeds onder het beheer van de kern valt
    - De experimentele Canvas-Plugin-PR voorbereiden of beoordelen
summary: Plan- en auditchecklist voor het verplaatsen van Canvas uit de core naar een gebundelde experimentele plugin.
title: Refactor van de Canvas-plugin
x-i18n:
    generated_at: "2026-07-27T05:14:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ead3f865ea80acb1e47f45a5ab07acf19a6470035c00c81006b2b1230bedd71e
    source_path: refactor/canvas.md
    workflow: 16
---

# Refactor van de Canvas-plugin

Canvas wordt weinig gebruikt en is experimenteel. Behandel het als een gebundelde plugin, niet als een kernfunctie. Core mag generieke infrastructuur voor de Gateway, Node, HTTP, authenticatie, configuratie en native clients behouden, maar Canvas-specifiek gedrag hoort onder `extensions/canvas` te staan.

## Doel

Verplaats het eigenaarschap van Canvas naar `extensions/canvas` en behoud daarbij het huidige gedrag met gekoppelde Nodes:

- de agentgerichte tool `canvas` wordt door de Canvas-plugin geregistreerd
- Canvas-Node-opdrachten zijn alleen toegestaan wanneer de Canvas-plugin ze registreert
- A2UI-host-/bronbestanden staan onder de Canvas-plugin
- materialisatie van Canvas-documenten staat onder de Canvas-plugin
- de implementatie van de CLI-opdracht staat onder de Canvas-plugin of delegeert via een runtime-barrel waarvan de plugin eigenaar is
- documentatie en plugininventaris beschrijven Canvas als experimenteel en door een plugin ondersteund

## Geen doelen

- Ontwerp in deze refactor de Canvas-UI van de native app niet opnieuw.
- Verwijder Canvas-protocol-/clientondersteuning niet uit iOS, Android of macOS, tenzij een afzonderlijke productbeslissing bepaalt dat Canvas moet worden verwijderd.
- Bouw niet alleen voor Canvas een breed framework voor pluginservices, tenzij minstens één andere gebundelde plugin dezelfde koppeling nodig heeft.

## Huidige branchstatus

Voltooid:

- Gebundeld pluginpakket toegevoegd in `extensions/canvas`.
- `extensions/canvas/openclaw.plugin.json` toegevoegd.
- De agenttool `canvas` verplaatst van `src/agents/tools/canvas-tool.ts` naar `extensions/canvas/src/tool.ts`.
- Core-registratie van `createCanvasTool` uit `src/agents/openclaw-tools.ts` verwijderd.
- De implementatie van de Canvas-host verplaatst van `src/canvas-host` naar `extensions/canvas/src/host`.
- `extensions/canvas/runtime-api.ts` behouden als compatibiliteitsbarrel waarvan de plugin eigenaar is, voor tests, packaging en externe openbare Canvas-helpers.
- Materialisatie van Canvas-documenten verplaatst van `src/gateway/canvas-documents.ts` naar `extensions/canvas/src/documents.ts`.
- Implementatie van de Canvas-CLI en A2UI-JSONL-helpers verplaatst naar `extensions/canvas/src/cli.ts`.
- Helpers voor de Canvas-host-URL en bereikgebonden capabilities verplaatst naar `extensions/canvas/src`.
- Standaardwaarden voor Canvas-Node-opdrachten uit hardgecodeerde Core-lijsten verplaatst naar plugin `nodeInvokePolicies`.
- Canvas-hostconfiguratie waarvan de plugin eigenaar is toegevoegd in `plugins.entries.canvas.config.host`.
- HTTP-serving van Canvas en A2UI achter registratie van HTTP-routes door de Canvas-plugin geplaatst.
- Generieke dispatch voor WebSocket-upgrades toegevoegd voor HTTP-routes waarvan plugins eigenaar zijn.
- Canvas-specifieke Gateway-host-URL en authenticatie van Node-capabilities vervangen door generieke helpers voor gehoste pluginoppervlakken en Node-capabilities.
- Mediaresolvers waarvan plugins eigenaar zijn toegevoegd, zodat URL's van Canvas-documenten via de Canvas-plugin worden omgezet in plaats van dat Core interne Canvas-documentonderdelen importeert.
- `api.registerNodeCliFeature(...)` toegevoegd, zodat Canvas `openclaw nodes canvas` als een Node-functie waarvan de plugin eigenaar is kan declareren zonder het pad van de bovenliggende opdracht handmatig uit te schrijven.
- Productie-imports van `extensions/canvas/runtime-api.js` uit `src/**` verwijderd.
- De bron van de A2UI-bundel verplaatst van `apps/shared/OpenClawKit/Tools/CanvasA2UI` naar `extensions/canvas/src/host/a2ui-app`.
- Implementatie voor het bouwen/kopiëren van A2UI onder `extensions/canvas/scripts` geplaatst en root-buildkoppelingen vervangen door generieke asset-hooks voor gebundelde plugins.
- De verouderde runtime-alias voor de configuratie op het hoogste niveau, `canvasHost`, verwijderd.
- De Canvas-migratie voor doctor behouden, zodat `openclaw doctor --fix` oude `canvasHost`-configuraties herschrijft naar `plugins.entries.canvas.config.host`.
- Canvas-protocolcompatibiliteit met oude agents achter Gateway-protocol v4 verwijderd. Native clients en Gateways gebruiken nu alleen `pluginSurfaceUrls.canvas` plus `node.pluginSurface.refresh`; het verouderde pad met `canvasHostUrl`, `canvasCapability` en `node.canvas.capability.refresh` wordt in deze experimentele refactor bewust niet ondersteund.
- De gegenereerde plugininventaris bijgewerkt met Canvas.
- Referentiedocumentatie voor de plugin toegevoegd in `docs/plugins/reference/canvas.md`.

Bekende resterende Canvas-oppervlakken waarvan Core eigenaar is:

- Canvas-handlers van native apps onder `apps/` gebruiken nog steeds bewust het oppervlak van de Canvas-plugin
- Canvas-protocol-/clienthandlers van native apps onder `apps/`
- uitvoer van gepubliceerde artefacten gebruikt nog steeds `dist/canvas-host/a2ui` voor achterwaarts compatibele runtime-lookup, maar de kopieerstap valt nu onder het eigenaarschap van de plugin

## Beoogde structuur

`extensions/canvas` moet eigenaar zijn van:

- pluginmanifest en pakketmetadata
- registratie van agenttools
- beleid voor Node-aanroepopdrachten
- Canvas-host en A2UI-runtime
- bron van de Canvas-A2UI-bundel en scripts voor het bouwen/kopiëren van assets
- aanmaak van Canvas-documenten en omzetting van assets
- implementatie van de Canvas-CLI
- Canvas-documentatiepagina en vermelding in de plugininventaris

Core moet alleen eigenaar zijn van generieke koppelingen:

- detectie en registratie van plugins
- generiek register voor agenttools
- generiek beleidsregister voor Node-aanroepen
- generieke HTTP-/authenticatiefunctionaliteit van de Gateway en dispatch voor WebSocket-upgrades
- generieke URL-omzetting voor gehoste pluginoppervlakken
- generieke registratie van gehoste mediaresolvers
- generiek transport van Node-capabilities
- generieke configuratie-infrastructuur
- generieke detectie van asset-hooks voor gebundelde plugins

Native apps mogen Canvas-opdrachthandlers behouden als clients van het protocol. Ze zijn niet de eigenaar van de pluginruntime.

## Migratiestappen

1. Behandel `plugins.entries.canvas.config.host` als het configuratieoppervlak waarvan de plugin eigenaar is.
2. Werk de documentatie bij zodat Canvas wordt beschreven als een experimentele gebundelde plugin.
3. Voer gerichte Canvas-tests, controles van de plugininventaris, controles van de Plugin-SDK-API en build-/typepoorten uit die door runtimegrenzen worden beïnvloed.

## Auditchecklist

Voordat de refactor als voltooid wordt beschouwd:

- `rg "src/canvas-host|../canvas-host"` retourneert geen actieve bronimports.
- `rg "canvas-tool|createCanvasTool" src` vindt geen implementatie van een Canvas-tool waarvan Core eigenaar is.
- `rg "canvas.present|canvas.snapshot|canvas.a2ui" src/gateway` vindt geen hardgecodeerde standaardwaarden voor allowlists buiten generieke tests voor pluginbeleid.
- `rg "extensions/canvas/runtime-api" src --glob '!**/*.test.ts'` is leeg.
- `rg "canvas-documents" src` is leeg.
- `rg "registerNodesCanvasCommands|nodes-canvas" src` is leeg; de Canvas-plugin registreert `openclaw nodes canvas` via geneste CLI-metadata van de plugin.
- `rg "createCanvasHostHandler|handleA2uiHttpRequest" src/gateway` retourneert geen eigenaarschap van de Gateway-runtime.
- `rg "apps/shared/OpenClawKit/Tools/CanvasA2UI|canvas-a2ui-copy|extensions/canvas/src/host/a2ui" scripts .github package.json` vindt alleen compatibiliteitswrappers of paden waarvan de plugin eigenaar is.
- `pnpm plugins:inventory:check` slaagt.
- `pnpm plugin-sdk:api:check` slaagt, of gegenereerde API-contractrecords worden bewust bijgewerkt en beoordeeld.
- Gerichte Canvas-tests slagen.
- Tests voor gewijzigde lanes slagen voor Canvas-host-/A2UI-paden.
- De PR-beschrijving vermeldt expliciet dat Canvas experimenteel en door een plugin ondersteund is.

## Verificatieopdrachten

Gebruik tijdens het itereren gerichte lokale controles:

```sh
pnpm test extensions/canvas/src/host/server.test.ts extensions/canvas/src/host/server.state-dir.test.ts extensions/canvas/src/host/file-resolver.test.ts
pnpm test src/gateway/server.plugin-node-capability-auth.test.ts src/gateway/server-import-boundary.test.ts
pnpm test extensions/canvas/src/config-migration.test.ts src/commands/doctor-legacy-config.migrations.test.ts
pnpm test test/scripts/changed-lanes.test.ts test/scripts/build-all.test.ts extensions/canvas/scripts/bundle-a2ui.test.ts test/scripts/bundled-plugin-assets.test.ts extensions/canvas/scripts/copy-a2ui.test.ts src/infra/run-node.test.ts
pnpm tsgo:extensions
pnpm plugins:inventory:check
pnpm plugin-sdk:api:check
```

Voer `pnpm build` vóór het pushen uit als de runtime-barrel, lazy import, packaging of gepubliceerde pluginoppervlakken veranderen.
