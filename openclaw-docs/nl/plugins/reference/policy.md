---
read_when:
    - Je installeert, configureert of controleert de beleidsplugin
summary: Voegt door beleid ondersteunde doctor-controles toe voor werkruimteconformiteit.
title: Beleidsplugin
x-i18n:
    generated_at: "2026-07-27T05:27:09Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 440f2f46e4149fdd5e65bf0140d4981c6d840e8e8c8a85d05eeb23a0839a61ac
    source_path: plugins/reference/policy.md
    workflow: 16
---

# Beleidsplugin

Voegt door beleid ondersteunde doctor-controles toe voor conformiteit van de workspace.

## Distributie

- Pakket: `@openclaw/policy`
- Installatieroute: opgenomen in OpenClaw

## Oppervlak

plugin

<!-- openclaw-plugin-reference:manual-start -->

## Gedrag

De beleidsplugin voegt doctor-statuscontroles toe voor door beleid beheerde OpenClaw-instellingen en gereguleerde workspace-declaraties. Het beleid omvat momenteel kanaalconformiteit, gereguleerde toolmetadata, de beveiligingsstatus van MCP-servers, de status van modelproviders, de status van toegang tot privénetwerken, de blootstellingsstatus van de Gateway, de status van de workspace en tools van agents, de geconfigureerde globale en agentspecifieke toolstatus, de status van de geconfigureerde sandboxruntime, de status van ingress- en kanaaltoegang, de gegevensverwerkingsstatus en de status van providers van configuratiegeheimen en authenticatieprofielen van OpenClaw.

Het beleid slaat opgestelde vereisten op in `policy.jsonc`, gebruikt bestaande OpenClaw-instellingen en workspace-declaraties als bewijsmateriaal en rapporteert afwijkingen via `openclaw policy check` en `openclaw doctor --lint`. Een geslaagde beleidscontrole levert hashes van het beleid, het bewijsmateriaal, de bevindingen en de attestatie op die beheerders voor audits kunnen vastleggen.

`openclaw policy compare --baseline <file>` vergelijkt het ene beleidsbestand met het andere beleidsbestand. Dit betreft uitsluitend conformiteit op configuratieniveau: de metadata van beleidsregels wordt gebruikt om te verifiëren dat het gecontroleerde beleid niets mist en niet minder streng is dan de opgestelde basislijn. De runtimestatus, aanmeldgegevens en waarden van geheimen worden niet geïnspecteerd.

Regels voor de toolstatus kunnen goedgekeurde profielen, uitsluitend op de workspace gerichte bestandssysteemtools, begrensde instellingen voor uitvoeringsbeveiliging, vragen en hosts, een uitgeschakelde modus met verhoogde bevoegdheden, exacte `alsoAllow`-vermeldingen en verplichte vermeldingen voor geweigerde tools vereisen. Het bewijsmateriaal registreert aanvullende `alsoAllow`-vermeldingen, omdat deze de effectieve toolstatus kunnen verruimen. Deze controles observeren uitsluitend configuratieconformiteit; ze lezen de goedkeuringsstatus van de runtime niet en voegen geen runtimehandhaving toe.

Regels voor de sandboxstatus kunnen goedgekeurde sandboxmodi en -backends vereisen, containernetwerken van de host en koppelingen met containernamespaces verbieden, alleen-lezen-containermounts vereisen, mounts van containerruntimesockets en onbeperkte containerprofielen verbieden en bronbereiken voor sandboxbrowser-CDP vereisen.
Deze controles observeren uitsluitend configuratieconformiteit; ze lezen de goedkeuringsstatus van de runtime niet, inspecteren geen actieve containers en voegen geen runtimehandhaving toe.

Regels voor gegevensverwerking kunnen redactie van gevoelige loggegevens vereisen, het vastleggen van telemetrie-inhoud verbieden, onderhoud van sessiebewaring vereisen en geheugenindexering van sessietranscripten verbieden. Deze controles observeren uitsluitend configuratieconformiteit; ze inspecteren geen onbewerkte logboeken, telemetrie-exports, transcripten, geheugenbestanden, geheimen of persoonsgegevens.

Benoemde beleidsbereiken onder `scopes.<scopeName>` kunnen strengere reguliere beleidssecties toevoegen voor de selector die ze vermelden. `agentIds` ondersteunt `tools`, `agents.workspace`, `sandbox` en `dataHandling.memory`; `channelIds` ondersteunt `ingress.channels`.
Runtime-agent-id's die niet expliciet in `agents.entries.*` staan, worden gecontroleerd aan de hand van de overgenomen globale/standaardstatus in plaats van stilzwijgend zonder bewijsmateriaal te slagen. Elk bereik in `policy.jsonc` moet geldig en afdwingbaar zijn voor de bijbehorende selector. Overlayregels zijn aanvullende claims; ze verzwakken daarom het beleid op het hoogste niveau niet en kunnen hun eigen bevindingen opleveren wanneer dezelfde geobserveerde configuratie beide bereiken schendt.

<!-- openclaw-plugin-reference:manual-end -->

## Gerelateerde documentatie

- [beleid](/nl/cli/policy)
