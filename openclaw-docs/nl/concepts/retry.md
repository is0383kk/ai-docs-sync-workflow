---
read_when:
    - Retrygedrag of standaardwaarden van providers bijwerken
    - Fouten bij verzending via providers of snelheidslimieten opsporen
summary: Beleid voor nieuwe pogingen bij uitgaande provider-aanroepen
title: Beleid voor nieuwe pogingen
x-i18n:
    generated_at: "2026-07-27T04:58:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9be2bcb5af829b90042bfcbc5c0e5f5cc5a3cb03dd5472737c80fa0f15803361
    source_path: concepts/retry.md
    workflow: 16
---

## Doelen

- Probeer opnieuw per HTTP-verzoek, niet per meerstapsflow.
- Behoud de volgorde door alleen de huidige stap opnieuw te proberen.
- Voorkom duplicatie van niet-idempotente bewerkingen.

## Standaardwaarden

| Instelling                | Standaard  |
| ------------------------- | ---------- |
| Pogingen                  | 3          |
| Maximale vertragingslimiet | 30000 ms  |
| Jitter                    | 0.1 (10%)  |
| Minimale Telegram-vertraging | 400 ms |
| Minimale Discord-vertraging | 500 ms  |

## Gedrag

### Modelproviders

- OpenClaw laat provider-SDK's normale korte nieuwe pogingen afhandelen.
- Voor op Stainless gebaseerde SDK's, zoals Anthropic en OpenAI, kunnen antwoorden waarvoor een nieuwe poging mogelijk is (`408`, `409`, `429` en `5xx`) `retry-after-ms` of `retry-after` bevatten. Wanneer die wachttijd langer is dan 60 seconden, injecteert OpenClaw `x-should-retry: false`, zodat de SDK de fout onmiddellijk doorgeeft en model-failover kan overschakelen naar een ander authenticatieprofiel of fallbackmodel.
- Overschrijf de limiet met `OPENCLAW_SDK_RETRY_MAX_WAIT_SECONDS=<seconds>`. Stel deze in op `0`, `false`, `off`, `none` of `disabled`, zodat SDK's lange `Retry-After`-wachttijden intern kunnen respecteren.

### Discord

- Probeert opnieuw bij fouten door snelheidslimieten (HTTP 429), time-outs van verzoeken, HTTP 5xx-antwoorden en tijdelijke transportfouten, zoals mislukte DNS-zoekopdrachten, opnieuw ingestelde verbindingen, gesloten sockets en mislukte fetch-aanroepen.
- Gebruikt Discord `retry_after` wanneer beschikbaar, anders exponentiële back-off.

### Telegram

- Probeert opnieuw bij tijdelijke fouten (429, time-out, verbinding mislukt/opnieuw ingesteld/gesloten, tijdelijk niet beschikbaar).
- Gebruikt `retry_after` wanneer beschikbaar, anders exponentiële back-off.
- Parseerfouten in HTML/Markdown worden niet opnieuw geprobeerd; bij de eerste poging wordt teruggevallen op platte tekst.

## Configuratie

Stel het beleid voor nieuwe pogingen per provider in `~/.openclaw/openclaw.json` in:

```json5
{
  channels: {
    telegram: {
      retry: {
        attempts: 3,
        minDelayMs: 400,
        maxDelayMs: 30000,
        jitter: 0.1,
      },
    },
    discord: {
      retry: {
        attempts: 3,
        minDelayMs: 500,
        maxDelayMs: 30000,
        jitter: 0.1,
      },
    },
  },
}
```

## Opmerkingen

- Nieuwe pogingen gelden per verzoek (bericht verzenden, media uploaden, reactie, peiling, sticker).
- Samengestelde flows proberen voltooide stappen niet opnieuw.

## Gerelateerd

- [Model-failover](/nl/concepts/model-failover)
- [Opdrachtwachtrij](/nl/concepts/queue)
