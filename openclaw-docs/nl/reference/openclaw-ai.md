---
read_when:
    - Je wilt de modeltransports van OpenClaw opnieuw gebruiken in een andere toepassing
    - Je wijzigt packages/ai of de hostpoorten voor AI-transport
    - Je controleert wat de OpenClaw-release naast het hoofdpakket naar npm publiceert
summary: 'Het npm-pakket @openclaw/ai: herbruikbare modeltransports, geïsoleerde runtimes en poorten voor hostbeleid'
title: '@openclaw/ai-pakket'
x-i18n:
    generated_at: "2026-07-27T06:08:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 610057caae0a9bbf9f74074cda75fc40c0b9aa9d3441f8263151f08f1a3f35a8
    source_path: reference/openclaw-ai.md
    workflow: 16
---

`@openclaw/ai` is de publiceerbare bibliotheekvorm van OpenClaws modeluitvoeringslaag: providerneutrale contracten voor berichten/tools/streams, validatie, diagnostiek, gebeurtenisstromen, een geïsoleerd runtimeregister en lazy adapters voor de acht ingebouwde API-families (Anthropic Messages, OpenAI Completions, OpenAI Responses, Azure OpenAI Responses, ChatGPT/Codex Responses, Google Generative AI, Google Vertex, Mistral Conversations).

Deze wordt bij elke release samen met het hoofdpackage `openclaw` gepubliceerd, vastgezet op dezelfde versie, met een eigen `npm-shrinkwrap.json` zodat de transitieve afhankelijkheidsboom tijdens de installatie wordt vergrendeld. Bij installatie van `openclaw` wordt automatisch de overeenkomende `@openclaw/ai` geïnstalleerd; bibliotheekgebruikers kunnen er rechtstreeks van afhankelijk zijn zonder enige OpenClaw-applicatiecode.

## Snel aan de slag

```js
import { createLlmRuntime } from "@openclaw/ai";
import { registerBuiltInApiProviders } from "@openclaw/ai/providers";

const runtime = createLlmRuntime();
registerBuiltInApiProviders(runtime.registry);

const stream = runtime.streamSimple(model, { messages }, { apiKey });
for await (const event of stream) {
  if (event.type === "text_delta") process.stdout.write(event.delta);
}
const result = await stream.result();
```

Een uitvoerbare versie staat in de repository op `examples/ai-chat`.

## Ontwerpcontract

- **Standaard beperkt tot de instantie.** Bij het importeren van het package wordt niets globaal geregistreerd. `createApiRegistry()` / `createLlmRuntime()` retourneren geïsoleerde instanties; `registerBuiltInApiProviders(registry)` schakelt voor één register de ingebouwde transports in. SDK-modules van providers worden bij het eerste gebruik lazy geladen.
- **Hostbeleid wordt geïnjecteerd, niet meegeleverd.** Bewaking van fetch voor aanvragen (bijvoorbeeld SSRF-beleid), het redigeren van geheimen uit tekst bij het opnieuw afspelen van toolresultaten, standaardinstellingen voor strikte OpenAI-tools en diagnostische logboekregistratie zijn `AiTransportHost`-poorten die met `configureAiTransportHost` worden geconfigureerd. De standaardinstellingen van de bibliotheek zijn inactief; OpenClaw installeert de werkelijke implementaties in zijn streamfacade.
- **Eén identiteit voor gebeurtenisstromen.** `@openclaw/ai/event-stream` is de canonieke `EventStream`-constructor die wordt gedeeld door OpenClaw-core, agent-core en externe gebruikers.
- **`internal/*`-subpaden zijn geen API.** Ze bestaan voor de OpenClaw-applicatie zelf en bieden geen semver-garantie.
- Provider-ID's, aanmeldgegevens, modelcatalogi, nieuwe pogingen en failover blijven verantwoordelijkheden van de applicatie. OpenClaw voegt die lagen rond dit package toe; een bibliotheekgebruiker levert rechtstreeks een `Model`-object en opties.

## Subpad-exports

| Subpad          | Inhoud                                                                         |
| ---------------- | ------------------------------------------------------------------------------ |
| `.`              | Contracten, `createApiRegistry`, `createLlmRuntime`, `configureAiTransportHost` |
| `./providers`    | `registerBuiltInApiProviders`, `resetApiProviders`                             |
| `./types`        | Typen voor modellen/berichten/tools/streams                                    |
| `./validation`   | Validatie van toolargumenten                                                   |
| `./diagnostics`  | Diagnostiekcontracten                                                          |
| `./event-stream` | Gedeelde `EventStream`-implementatie                                           |
| `./internal/*`   | Intern voor OpenClaw, geen semver-garantie                                     |
