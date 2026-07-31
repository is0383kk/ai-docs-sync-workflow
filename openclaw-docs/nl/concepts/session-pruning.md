---
read_when:
    - Je wilt de contextgroei door tooluitvoer beperken
    - Je wilt de optimalisatie van de Anthropic-promptcache begrijpen
summary: Oude toolresultaten inkorten om de context compact en caching efficiënt te houden
title: Sessies opschonen
x-i18n:
    generated_at: "2026-07-27T05:03:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: dd5cb4582cb8d9d7265213abe1f5b5893634882b9f8b3ce1deef746293dd07db
    source_path: concepts/session-pruning.md
    workflow: 16
---

Sessiesnoeiing verwijdert **oude toolresultaten** uit de context vóór elke LLM-aanroep. Dit beperkt contextvervuiling door verzamelde tooluitvoer (uitvoeringsresultaten, gelezen bestanden, zoekresultaten) zonder normale conversatietekst te herschrijven.

<Info>
Snoeiing vindt alleen in het geheugen plaats -- de op schijf opgeslagen sessietranscriptie wordt niet gewijzigd. Je volledige geschiedenis blijft altijd behouden.
</Info>

## Waarom dit belangrijk is

Tijdens lange sessies stapelt tooluitvoer zich op, waardoor het contextvenster uitdijt. Dit verhoogt de kosten en kan [Compaction](/nl/concepts/compaction) eerder dan nodig afdwingen.

Snoeiing is vooral waardevol voor **Anthropic-promptcaching**. Nadat de TTL van de cache verloopt, wordt bij het volgende verzoek de volledige prompt opnieuw gecachet. Snoeiing verkleint de te schrijven cacheomvang en verlaagt zo rechtstreeks de kosten.

## Hoe het werkt

Snoeiing wordt uitgevoerd in de modus `cache-ttl`, waarbij zowel een tijdscontrole als een controle van de contextomvang geldt:

1. Wacht tot de TTL van de cache verloopt (standaard 5 minuten wanneer deze handmatig is ingesteld; zie [Slimme standaardwaarden](#smart-defaults) voor de automatische standaardwaarde van Anthropic). Voordat de TTL is verstreken, wordt snoeiing volledig overgeslagen om hergebruik van de promptcache voor kort opeenvolgende beurten te behouden.
2. Schat zodra de TTL is verstreken de totale contextomvang ten opzichte van het contextvenster van het model. Als de verhouding lager is dan `softTrimRatio` (standaard 0.3), sla je het snoeien over en laat je de TTL-klok doorlopen.
3. Pas een **lichte inkorting** toe op te grote toolresultaten boven de verhouding: behoud het begin en einde (standaard elk 1500 tekens, begrensd op in totaal 4000 tekens) en voeg daartussen `...` in.
4. Als de verhouding nog steeds gelijk is aan of hoger is dan `hardClearRatio` (standaard 0.5) en er ten minste `minPrunableToolChars` (standaard 50,000) aan snoeibare toolinhoud overblijft, **wis** je die resultaten **volledig**: vervang de inhoud door een tijdelijke aanduiding (standaard `[Old tool result content cleared]`).
5. Stel de TTL-klok alleen opnieuw in wanneer het snoeien de context daadwerkelijk heeft gewijzigd, zodat vervolgverzoeken de nieuwe cache hergebruiken.

Ongeacht de drempelwaarden gelden twee veiligheidsregels: de recentste `keepLastAssistants` assistentbeurten (standaard 3) worden nooit gesnoeid en niets vóór het eerste gebruikersbericht van de sessie wordt ooit gesnoeid (dit beschermt opstartlezingen zoals `SOUL.md`/`USER.md`).

Alleen `toolResult`-berichten komen in aanmerking; normale conversatietekst blijft ongewijzigd. Gebruik `agents.defaults.contextPruning.tools.{allow,deny}` om te bepalen welke toolnamen mogen worden gesnoeid.

## Opschoning van verouderde afbeeldingen

OpenClaw bouwt ook een afzonderlijke idempotente herhalingsweergave voor sessies die onbewerkte afbeeldingsblokken of mediamarkeringen voor promptvoorbereiding in de geschiedenis bewaren.

- Hierbij blijven de **3 recentste voltooide beurten** byte voor byte behouden, zodat promptcachevoorvoegsels stabiel blijven voor recente vervolgvragen. Deze telling omvat alle voltooide beurten, niet alleen beurten met afbeeldingen, waardoor beurten met alleen tekst ook meetellen voor het venster.
- In de herhalingsweergave worden oudere, al verwerkte afbeeldingsblokken uit de geschiedenis van `user` of `toolResult` vervangen door `[image data removed - already processed by model]`.
- Oudere tekstuele mediaverwijzingen zoals `[media attached: ...]`, `[Image: source: ...]` en `media://inbound/...` worden vervangen door `[media reference removed - already processed by model]`. Bijlagemarkeringen van de huidige beurt blijven intact, zodat visuele modellen nieuwe afbeeldingen nog steeds kunnen voorbereiden.
- De onbewerkte sessietranscriptie wordt niet herschreven, zodat geschiedenisweergaven de oorspronkelijke berichtvermeldingen en bijbehorende afbeeldingen nog steeds kunnen weergeven.
- Dit staat los van de normale snoeiing op basis van de cache-TTL hierboven. Het voorkomt dat herhaalde afbeeldingspayloads of verouderde mediaverwijzingen promptcaches tijdens latere beurten ongeldig maken.

## Slimme standaardwaarden

De meegeleverde Anthropic-plugin configureert automatisch de snoeiing en Heartbeat-frequentie wanneer voor het eerst een Anthropic-authenticatieprofiel (of Claude CLI-authenticatieprofiel) wordt gevonden, maar alleen voor velden die je nog niet expliciet hebt ingesteld:

| Authenticatiemodus                       | `contextPruning.mode` | `contextPruning.ttl` | `heartbeat.every` |
| ---------------------------------------- | --------------------- | -------------------- | ----------------- |
| OAuth/token (inclusief hergebruik van Claude CLI) | `cache-ttl`           | `1h`                 | `1h`              |
| API-sleutel                              | `cache-ttl`           | `1h`                 | `30m`             |

Als je `agents.defaults.contextPruning.mode` of `agents.defaults.heartbeat.every` zelf instelt, overschrijft OpenClaw deze niet. Deze automatische standaardwaarde wordt alleen toegepast op authenticatie uit de Anthropic-familie; andere providers krijgen snoeiing `off`, tenzij je deze configureert.

## In- of uitschakelen

Snoeiing is standaard uitgeschakeld voor providers die niet van Anthropic zijn. Inschakelen:

```json5
{
  agents: {
    defaults: {
      contextPruning: { mode: "cache-ttl", ttl: "5m" },
    },
  },
}
```

Uitschakelen: stel `mode: "off"` in.

## Snoeiing versus Compaction

|             | Snoeiing               | Compaction                |
| ----------- | ---------------------- | ------------------------- |
| **Wat**     | Kort toolresultaten in | Vat de conversatie samen  |
| **Opgeslagen?** | Nee (per verzoek) | Ja (in de transcriptie)   |
| **Bereik**  | Alleen toolresultaten  | Volledige conversatie     |

Ze vullen elkaar aan -- snoeiing houdt tooluitvoer compact tussen Compaction-cycli.

## Verder lezen

- [Compaction](/nl/concepts/compaction): contextvermindering op basis van samenvattingen
- [Gateway-configuratie](/nl/gateway/configuration): alle configuratieopties voor snoeiing (`contextPruning.*`)

## Gerelateerd

- [Sessiebeheer](/nl/concepts/session)
- [Sessietools](/nl/concepts/session-tool)
- [Contextengine](/nl/concepts/context-engine)
