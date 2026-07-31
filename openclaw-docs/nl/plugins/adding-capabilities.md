---
read_when:
    - Een nieuwe kernfunctionaliteit en een registratie-interface voor plugins toevoegen
    - Bepalen of code thuishoort in de kern, een leveranciersplugin of een functieplugin
    - Een nieuwe runtimehelper voor kanalen of tools aansluiten
sidebarTitle: Adding capabilities
summary: Handleiding voor bijdragers voor het toevoegen van een nieuwe gedeelde mogelijkheid aan het Plugin-systeem van OpenClaw
title: Mogelijkheden toevoegen (gids voor bijdragers)
x-i18n:
    generated_at: "2026-07-27T05:39:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 14f86c98eb10c6e92970d1b65009ac7bb103afcb6bc57bad2c39e59bc038c961
    source_path: plugins/adding-capabilities.md
    workflow: 16
---

<Info>
  Dit is een **bijdragershandleiding** voor ontwikkelaars van de OpenClaw-kern. Als je
  een externe plugin bouwt, raadpleeg dan [Plugins bouwen](/nl/plugins/building-plugins).
  Voor de diepgaande architectuurreferentie (capability-model, eigenaarschap,
  laadpijplijn, runtimehelpers) raadpleeg je [Interne werking van plugins](/nl/plugins/architecture).
</Info>

Gebruik dit wanneer OpenClaw een nieuw gedeeld domein nodig heeft, zoals embeddings, het
genereren van afbeeldingen, het genereren van video's of een toekomstig functiegebied dat door leveranciers wordt ondersteund.

De regel:

- **plugin** = eigenaarschapsgrens
- **capability** = gedeeld kerncontract

Koppel een leverancier niet rechtstreeks aan een kanaal of tool. Definieer eerst de capability.

## Wanneer je een capability maakt

Maak alleen een nieuwe capability wanneer **al** het volgende waar is:

1. Meer dan één leverancier zou deze redelijkerwijs kunnen implementeren.
2. Kanalen, tools of functieplugins moeten deze kunnen gebruiken zonder rekening te houden met de leverancier.
3. De kern moet verantwoordelijk zijn voor fallback-, beleids-, configuratie- of afleveringsgedrag.

Als het werk alleen voor een leverancier is en er nog geen gedeeld contract bestaat, definieer dan eerst het contract.

## De standaardvolgorde

1. Definieer het getypeerde kerncontract.
2. Voeg pluginregistratie voor dat contract toe.
3. Voeg een gedeelde runtimehelper toe.
4. Koppel als bewijs één echte leveranciersplugin.
5. Zet verbruikers in functies en kanalen over op de runtimehelper.
6. Voeg contracttests toe.
7. Documenteer de configuratie voor operators en het eigenaarschapsmodel.

## Wat waar thuishoort

| Laag                       | Verantwoordelijk voor                                                                                                                                                                                                                  |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Kern**                   | Aanvraag-/antwoordtypen; providerregister en -resolutie; fallbackgedrag; configuratieschema met doorgegeven `title`-/`description`-documentatiemetadata op geneste object-, jokerteken-, array-item- en compositieknooppunten; runtimehelperoppervlak. |
| **Leveranciersplugin**     | API-aanroepen naar de leverancier, afhandeling van leveranciersauthenticatie, leveranciersspecifieke normalisatie van aanvragen en registratie van de capability-implementatie.                                                        |
| **Functie-/kanaalplugin**  | Roept `api.runtime.*` of de overeenkomstige `plugin-sdk/*-runtime`-helper aan. Roept nooit rechtstreeks een leveranciersimplementatie aan.                                                                                              |

## Koppelvlakken voor providers en harnesses

Gebruik **providerhooks** wanneer het gedrag bij het modelprovidercontract hoort in plaats van bij de generieke agentlus. Voorbeelden zijn providerspecifieke aanvraagparameters na transportselectie, voorkeuren voor authenticatieprofielen, promptoverlays en vervolgroutering voor fallback nadat een model of profiel is uitgevallen.

Gebruik **agent-harnesshooks** wanneer het gedrag hoort bij de runtime die een beurt uitvoert. Harnesses kunnen expliciete protocoluitkomsten classificeren, zoals lege uitvoer, redenering zonder zichtbare uitvoer of een gestructureerd plan zonder definitief antwoord, zodat het fallbackbeleid van het buitenste model de beslissing over opnieuw proberen kan nemen.

Houd beide koppelvlakken beperkt:

- De kern is verantwoordelijk voor het beleid voor opnieuw proberen en fallback.
- Providerplugins zijn verantwoordelijk voor providerspecifieke hints voor aanvragen, authenticatie en routering.
- Harnessplugins zijn verantwoordelijk voor runtimespecifieke classificatie van pogingen.
- Plugins van derden retourneren hints en wijzigen de kernstatus niet rechtstreeks.

## Bestandschecklist

Voor een nieuwe capability moet je waarschijnlijk deze gebieden aanpassen:

- `src/<capability>/types.ts`
- `src/<capability>/...registry/runtime.ts`
- `src/plugins/types.ts`
- `src/plugins/registry.ts`
- `src/plugins/captured-registration.ts`
- `src/plugins/contracts/registry.ts`
- `src/plugins/runtime/types-core.ts`
- `src/plugins/runtime/index.ts`
- `src/plugin-sdk/<capability>.ts`
- `src/plugin-sdk/<capability>-runtime.ts`
- Een of meer gebundelde pluginpakketten.
- Configuratie, documentatie, tests.

## Uitgewerkt voorbeeld: afbeeldingen genereren

Het genereren van afbeeldingen volgt de standaardstructuur:

1. De kern definieert `ImageGenerationProvider`.
2. De kern stelt `registerImageGenerationProvider(...)` beschikbaar.
3. De kern stelt `api.runtime.imageGeneration.generate(...)` en `.listProviders(...)` beschikbaar.
4. Leveranciersplugins (`comfy`, `deepinfra`, `fal`, `google`, `litellm`, `microsoft-foundry`, `minimax`, `openai`, `openrouter`, `vydra`, `xai`) registreren door leveranciers ondersteunde implementaties.
5. Toekomstige leveranciers registreren hetzelfde contract zonder kanalen of tools te wijzigen.

De configuratiesleutel is bewust gescheiden van routering voor beeldanalyse:

- `agents.defaults.imageModel` analyseert afbeeldingen.
- `agents.defaults.mediaModels.image` genereert afbeeldingen.

Houd deze gescheiden, zodat fallback en beleid expliciet blijven.

## Embeddingproviders

Gebruik `registerEmbeddingProvider(...)` / contract `embeddingProviders` voor
herbruikbare providers van vectorembeddings. Dit contract is bewust breder
dan geheugen: tools, zoekfuncties, retrieval, importers of toekomstige functieplugins
kunnen embeddings gebruiken zonder afhankelijk te zijn van de geheugenengine. Zoeken in het geheugen
gebruikt ook de generieke `embeddingProviders`.

De oudere geheugenspecifieke registratie-API en het contract `memoryEmbeddingProviders`
zijn verouderd. Gebruik `registerEmbeddingProvider` en
`embeddingProviders` voor alle nieuwe embeddingproviders.

## Reviewchecklist

Controleer het volgende voordat je een nieuwe capability uitbrengt:

- Geen enkel kanaal of tool importeert rechtstreeks leverancierscode.
- De runtimehelper is het gedeelde pad.
- Ten minste één contracttest controleert gebundeld eigenaarschap.
- De configuratiedocumentatie vermeldt de nieuwe model-/configuratiesleutel.
- De plugindocumentatie legt de eigenaarschapsgrens uit.

Als een PR de capability-laag overslaat en leveranciersgedrag hardcodeert in een kanaal of tool, stuur deze dan terug en definieer eerst het contract.

## Gerelateerd

- [Interne werking van plugins](/nl/plugins/architecture) — capability-model, eigenaarschap, laadpijplijn, runtimehelpers.
- [Plugins bouwen](/nl/plugins/building-plugins) — tutorial voor de eerste plugin.
- [SDK-overzicht](/nl/plugins/sdk-overview) — referentie voor de importstructuur en registratie-API.
- [Skills maken](/nl/tools/creating-skills) — aanvullend oppervlak voor bijdragers.
