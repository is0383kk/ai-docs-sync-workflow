---
read_when:
    - Een gebruiker meldt dat agents vastlopen en toolaanroepen blijven herhalen
    - Je moet de beveiliging tegen herhaalde aanroepen beheren
    - Je bewerkt het beleid voor agenttools en de runtime
    - Je krijgt te maken met `compaction_loop_persisted` afbrekingen na een nieuwe poging wegens contextoverschrijding
summary: Guardrails inschakelen die herhalende lussen van toolaanroepen detecteren
title: Detectie van tool-lussen
x-i18n:
    generated_at: "2026-07-27T06:16:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 79b5aa1d85e02b8cf46a95b3bcebb255178b91456517cab804cce77b8f3b818e
    source_path: tools/loop-detection.md
    workflow: 16
---

OpenClaw heeft twee samenwerkende beveiligingsmechanismen tegen repetitieve patronen van toolaanroepen,
beide geconfigureerd onder `tools.loopDetection`:

1. **Lusdetectie** (`enabled`) - standaard uitgeschakeld. Bewaakt de voortschrijdende
   geschiedenis van toolaanroepen op herhaalde patronen en nieuwe pogingen met onbekende tools.
2. **Beveiliging na Compaction** - ingeschakeld wanneer
   `enabled` niet expliciet `false` is. Wordt na elke nieuwe poging na Compaction geactiveerd en
   breekt de uitvoering af als de agent dezelfde `(tool, args, result)`-combinatie van drie elementen
   binnen het venster herhaalt.

Stel `tools.loopDetection.enabled: false` in om beide beveiligingsmechanismen uit te schakelen.

## Waarom dit bestaat

- Detecteer repetitieve reeksen die geen voortgang opleveren.
- Detecteer hoogfrequente lussen zonder resultaat (dezelfde tool, dezelfde invoer, herhaalde
  fouten).
- Detecteer specifieke patronen van herhaalde aanroepen voor bekende pollingtools.
- Doorbreek cycli van contextoverloop -> Compaction -> dezelfde lus in plaats van ze
  onbeperkt te laten doorgaan.

## Configuratieblok

Globale instelling:

```json5
{
  tools: {
    loopDetection: {
      enabled: false, // hoofdschakelaar voor de detectoren met voortschrijdende geschiedenis
    },
  },
}
```

Overschrijving per agent (optioneel, bij `agents.entries.*.tools.loopDetection`):

```json5
{
  agents: {
    list: [
      {
        id: "safe-runner",
        tools: {
          loopDetection: {
            enabled: true,
          },
        },
      },
    ],
  },
}
```

De instelling per agent overschrijft de globale instelling.

### Gedrag van velden

| Veld      | Standaard | Effect                                                                                            |
| --------- | --------- | ------------------------------------------------------------------------------------------------- |
| `enabled` | `false` | Hoofdschakelaar voor de detectoren met voortschrijdende geschiedenis. `false` schakelt ook de beveiliging na Compaction uit. |

Voor `exec` vergelijkt hashing voor ontbrekende voortgang stabiele opdrachtresultaten (status,
afsluitcode, time-outvlag, uitvoer) en negeert het vluchtige runtime-metagegevens zoals
duur, PID, sessie-ID en werkmap. Resultaten van uitgaande berichtverzendingen
worden gehasht nadat vluchtige ID's per aanroep (bericht-ID, bestands-ID, tijdstempel)
zijn verwijderd, zodat een resultaat met de status "verzonden" niet identiek lijkt aan een ander resultaat met de status "verzonden".
Wanneer een uitvoerings-ID beschikbaar is, wordt de geschiedenis alleen binnen die uitvoering geëvalueerd,
zodat geplande Heartbeat-cycli en nieuwe uitvoeringen geen verouderde aantallen lussen
van eerdere uitvoeringen overnemen.

## Aanbevolen configuratie

- Stel voor kleinere modellen `enabled: true` in. Topmodellen hebben zelden detectie met voortschrijdende geschiedenis nodig en kunnen
  de hoofdschakelaar op `false` laten staan, terwijl ze nog steeds profiteren van de
  beveiliging na Compaction.
- Stel expliciet
  `tools.loopDetection.enabled: false` in om alles uit te schakelen, inclusief de beveiliging na Compaction.

## Beveiliging na Compaction

Na een nieuwe poging met Compaction als gevolg van een contextoverloop activeert de uitvoerder
voor de volgende paar toolaanroepen een beveiliging met een kort venster. Als de agent dezelfde
`(toolName, argsHash, resultHash)`-combinatie van drie elementen binnen dat venster vaak genoeg produceert, concludeert de beveiliging dat Compaction de
lus niet heeft doorbroken en breekt deze de uitvoering af met een `compaction_loop_persisted`-fout.

De beveiliging wordt aangestuurd door de algemene vlag `tools.loopDetection.enabled`, met één
bijzonderheid: ze blijft **ingeschakeld wanneer de vlag niet is ingesteld of `true` is**, en wordt alleen
uitgeschakeld wanneer de vlag expliciet `false` is. Dit is opzettelijk: de beveiliging
dient om te ontsnappen aan Compaction-lussen die anders onbeperkt tokens zouden verbruiken,
zodat een gebruiker zonder configuratie nog steeds beschermd is.

```json5
{
  tools: {
    loopDetection: {
      // hoofdschakelaar; stel deze in op false om de beveiliging samen met de voortschrijdende detectoren uit te schakelen
      enabled: true,
    },
  },
}
```

- De beveiliging breekt nooit af zolang de resultaten veranderen; alleen byte-identieke
  resultaten binnen het venster activeren haar.
- Ze wordt alleen direct na een nieuwe poging met Compaction geactiveerd, niet op andere
  momenten tijdens een uitvoering.

<Note>
  De beveiliging na Compaction wordt uitgevoerd wanneer de algemene vlag niet expliciet `false` is, zelfs als je nooit een `tools.loopDetection`-blok hebt geschreven. Zoek ter controle direct na een Compaction-gebeurtenis naar `post-compaction guard armed for N attempts` in het Gateway-logboek.
</Note>

## Logboeken en verwacht gedrag

Wanneer een lus wordt gedetecteerd, registreert OpenClaw een lusgebeurtenis en waarschuwt of blokkeert het
de volgende toolcyclus, afhankelijk van de ernst. Zo beschermt het tegen onbeheerst tokenverbruik
en vastlopers, terwijl normale toegang tot tools behouden blijft.

- Waarschuwingen komen eerst.
- Blokkering volgt zodra een patroon langer aanhoudt dan de waarschuwingsdrempel.
- Kritieke drempels blokkeren de volgende toolcyclus en tonen een duidelijke
  reden voor lusdetectie in de uitvoeringsregistratie.
- De beveiliging na Compaction produceert `compaction_loop_persisted`-fouten die
  de betreffende tool en het aantal identieke aanroepen vermelden.

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Uitvoeringsgoedkeuringen" href="/nl/tools/exec-approvals" icon="shield">
    Beleid voor toestaan/weigeren van shelluitvoering.
  </Card>
  <Card title="Denkniveaus" href="/nl/tools/thinking" icon="brain">
    Niveaus van redeneerinspanning en interactie met providerbeleid.
  </Card>
  <Card title="Subagenten" href="/nl/tools/subagents" icon="users">
    Geïsoleerde agenten starten om onbeheerst gedrag te begrenzen.
  </Card>
  <Card title="Configuratiereferentie" href="/nl/gateway/config-tools#toolsloopdetection" icon="gear">
    Volledig `tools.loopDetection`-schema en samenvoegingssemantiek.
  </Card>
</CardGroup>
