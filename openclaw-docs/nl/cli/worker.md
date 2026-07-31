---
read_when:
    - Cloudworkers beheren of fouten opsporen die door de Gateway zijn gestart
    - Toelating van workers, sessietoewijzing of isolatie van lokale tools verifiëren
summary: Interne operatorreferentie voor de beperkte cloudworkerruntime
title: Werker
x-i18n:
    generated_at: "2026-07-27T05:01:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0c4749e2abaf4fca00d903114b0661454d67207547fe17711dc5315656e0cd14
    source_path: cli/worker.md
    workflow: 16
---

# `openclaw worker`

`openclaw worker` is het beperkte runtime-ingangspunt dat een cloudworker-
orchestrator binnen een voorbereide workeromgeving start. Het is geen
algemene opdracht voor handmatige workerregistratie.

De Gateway installeert de bijpassende OpenClaw-bundel en opent de omgekeerde
SSH-tunnel met vastgezette hostsleutel. De workerlauncher start deze opdracht met
een voorbereide toewijzing. De opdracht maakt verbinding via de door de tunnel
doorgestuurde lokale socket en wordt toegelaten met de specifieke rol
`worker`.

## Startcontract

De opdracht leest precies één begrensde JSON-startenvelop van de standaardinvoer.
De envelop bevat de locatie van de lokale socket, de aangemaakte workerreferentie,
de bundel- en protocolidentiteit, het eigenaartijdperk, de enige toegewezen sessie
en beurt, en de exacte namen van workerlokale tools die voor die beurt zijn
geautoriseerd. De Gateway bepaalt deze definitieve toolset vóór de overdracht op
basis van het huidige beleid; onbewerkte configuratie en de identiteit van de
ingeplande eigenaar komen nooit in de workerenvelop terecht.
De referentie wordt nooit via opdrachtregelargumenten geaccepteerd en deze pagina
bevat met opzet geen voorbeeld van een referentie of handmatig opgestelde envelop.

Toelating wordt standaard geweigerd als de envelop ongeldig is, de referentie
wordt afgewezen, de bundel- of protocolfuncties niet overeenkomen, of de sessie
en het eigenaartijdperk niet meer actueel zijn. Ontbrekende, dubbele of onbekende
toolnamen maken de envelop eveneens ongeldig. Operators moeten workers via de
cloudworker-orchestrator starten in plaats van dit ingangspunt rechtstreeks aan
te roepen.

## Runtimegrens

Het proces voert de normale ingesloten agentlus uit met een beperkte backend:

- De codeertools `read`, `write`, `edit`, `apply_patch`, `exec` en `process`
  worden lokaal in de workerwerkruimte uitgevoerd wanneer ze aanwezig zijn in de
  door de Gateway verstrekte bevoegdheid voor de beurt. Met een lege bevoegdheid
  wordt het model zonder tools uitgevoerd.
- Modelaanroepen gebruiken de inferentieproxy van de Gateway. Er wordt geen
  lokaal modelauthenticatieprofiel geladen.
- Transcripties worden geschreven via de transcript-commit-RPC van de Gateway.
- Streaming- en toollevenscyclusupdates gebruiken de live-event-RPC van de Gateway.
- Alleen de toegewezen sessie en beurt worden geaccepteerd.

De workermodus start geen kanalen, HTTP-oppervlakken van de Gateway of
automatisch startende plugins buiten de toolset van de toegewezen sessie. De
modus gebruikt een tijdelijke statusmap en heeft geen permanente referenties
voor providers of forges.

Het doorsturen van sessies van worker naar worker is in deze modus niet
beschikbaar. Plaatsing en doorsturen blijven eigendom van de Gateway: een
operator kan via de Gateway een bestaande lokale sessie in een beheerde worktree
doorsturen, terwijl een workerproces zichzelf of een andere worker niet kan
doorsturen.

De voorbereide toewijzing bevat de transcriptcontext, het geaccepteerde
basisblad, de commitreeks en de live-eventcursor. Wanneer de tunnel opnieuw
verbinding maakt, wordt het proces opnieuw toegelaten met dezelfde referentie en
hetzelfde eigenaartijdperk, behoudt het de geaccepteerde transcriptbasis, speelt
het de nog niet bevestigde live-eventstaart opnieuw af en wordt een actieve
inferentiebeurt met dezelfde identiteit opnieuw gekoppeld. Het afsluitende
inferentiebericht is gezaghebbend als gestreamde delta's zijn gemist. Een
vervangend eigenaartijdperk sluit het proces af en zorgt voor een nette
beëindiging.

Een transcriptweigering van `stale-base-leaf` stopt de huidige uitvoering
onmiddellijk. De workermodus probeert de afgewezen reeks niet opnieuw uit op een
ander blad, zodat geen dubbele commit wordt geproduceerd; een nog niet
vastgelegde staart in het geheugen van die uitvoering gaat verloren. Opnieuw
starten valt onder de plaatsingseigenaar van mijlpaal 3, die een nieuwe
toewijzing moet maken op basis van het gezaghebbende transcript en
commitregister van de Gateway. Ook beëindigt een herstart van het Gateway-proces
een wachtende inferentiebeurt met een providerfout; alleen een herverbinding van
de tunnel of worker-WebSocket kan opnieuw koppelen met een actieve
inferentiestroom in hetzelfde proces.

Zie [Gateway-protocol](/nl/gateway/protocol#worker-role-and-closed-protocol) voor het
gesloten worker-RPC-oppervlak en [Plan voor cloudworkers](/nl/plan/cloud-workers)
voor de architectuur en het beveiligingsmodel.
