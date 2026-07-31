---
read_when:
    - Gedrag of standaardinstellingen van de type-indicator wijzigen
summary: Wanneer OpenClaw typindicatoren toont en hoe je deze afstemt
title: Typindicatoren
x-i18n:
    generated_at: "2026-07-27T05:45:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b3c66d61ea7e3e809b8e88ae2eabb9794f0886b629094753716ed02912843ffc
    source_path: concepts/typing-indicators.md
    workflow: 16
---

Typindicatoren worden naar het chatkanaal verzonden zolang een uitvoering actief is. Gebruik `agents.defaults.typingMode` om te bepalen **wanneer** het typen begint en `typingIntervalSeconds` om te bepalen **hoe vaak** de indicator wordt vernieuwd (keepalive-interval, standaard 6 seconden).

## Standaardinstellingen

Wanneer `agents.defaults.typingMode` **niet is ingesteld**:

- **Directe chats**: de typindicator start onmiddellijk zodra de modellus begint.
- **Groepschats met een vermelding**: de typindicator start onmiddellijk.
- **Groepschats zonder vermelding**: de typindicator start wanneer de toegelaten uitvoering gebruikerszichtbare activiteit vertoont, zoals uitvoeringsactiviteit van het harnas of berichttekst.
- **Heartbeat-uitvoeringen**: de typindicator start wanneer de Heartbeat-uitvoering begint, als het bepaalde Heartbeat-doel een chat is die typindicatoren ondersteunt en typen niet is uitgeschakeld.

## Modi

Stel `agents.defaults.typingMode` in op een van de volgende waarden:

- `never` - nooit een typindicator.
- `instant` - begin **zodra de modellus begint** met typen, zelfs als de uitvoering later alleen het token voor een stil antwoord retourneert.
- `thinking` - begin met typen bij de **eerste redeneerstap**, of bij actieve uitvoering van het harnas nadat de beurt is geaccepteerd.
- `message` - begin met typen bij de **eerste gebruikerszichtbare antwoordactiviteit**, zoals actieve uitvoering van het harnas of een niet-stille tekststap. Tokens voor stille antwoorden, zoals `NO_REPLY`, tellen niet als tekstactiviteit.

Volgorde van „hoe vroeg de indicator wordt geactiveerd”: `never` -> `message`/`thinking` -> `instant`.

## Configuratie

Stel de standaardwaarde op agentniveau in:

```json5
{
  agents: {
    defaults: {
      typingMode: "thinking",
      typingIntervalSeconds: 6,
    },
  },
}
```

Overschrijf het beleid voor één agent:

```json5
{
  agents: {
    entries: {
      support: {
        typingMode: "message",
      },
    },
  },
}
```

## Opmerkingen

- De modus `message` start niet bij tokens voor stille antwoorden, maar actieve uitvoering kan de typindicator al tonen voordat er tekst van de assistent beschikbaar is.
- `thinking` reageert nog steeds op gestreamde redenering (`reasoningLevel: "stream"`) en kan ook starten bij actieve uitvoering voordat redeneerstappen binnenkomen.
- De Heartbeat-typindicator is een beschikbaarheidssignaal voor het bepaalde afleveringsdoel. Deze start bij aanvang van de Heartbeat-uitvoering in plaats van de streamtiming van `message` of `thinking` te volgen. Stel `typingMode: "never"` in om dit uit te schakelen.
- Heartbeats tonen geen typindicator wanneer het Heartbeat-doel `"none"` is, wanneer het doel niet kan worden bepaald, wanneer chataflevering voor de Heartbeat is uitgeschakeld of wanneer het kanaal geen typindicatoren ondersteunt.
- `agents.defaults.typingIntervalSeconds` bepaalt voor elke agent het **vernieuwingsinterval**, niet de starttijd. Standaard: 6 seconden.

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Aanwezigheid" href="/nl/concepts/presence" icon="signal">
    Hoe de Gateway verbonden clients bijhoudt voor de pagina Apparaten van de Control UI en het tabblad Instanties van macOS.
  </Card>
  <Card title="Streamen en opdelen" href="/nl/concepts/streaming" icon="bars-staggered">
    Gedrag van uitgaande streams, segmentgrenzen en kanaalspecifieke aflevering.
  </Card>
</CardGroup>
