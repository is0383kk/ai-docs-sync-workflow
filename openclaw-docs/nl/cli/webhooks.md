---
read_when:
    - Je wilt Gmail Pub/Sub-gebeurtenissen aan OpenClaw koppelen
    - Je hebt de volledige lijst met vlaggen en standaardwaarden nodig
summary: CLI-referentie voor `openclaw webhooks` (Gmail Pub/Sub-configuratie en runner)
title: Webhooks
x-i18n:
    generated_at: "2026-07-27T05:47:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 83fff0ac2ce247402f45523eda0b5cdd551bd65212636118698e45cb8740236c
    source_path: cli/webhooks.md
    workflow: 16
---

# `openclaw webhooks`

Webhook-helpers en -integraties. Momenteel is dit oppervlak beperkt tot Gmail Pub/Sub-flows die zijn gebouwd op de meegeleverde `gog`-watcher.

## Subopdrachten

```bash
openclaw webhooks gmail setup --account <email> [...]
openclaw webhooks gmail run   [--account <email>] [...]
```

| Subopdracht    | Beschrijving                                                                           |
| ------------- | ------------------------------------------------------------------------------------- |
| `gmail setup` | Eenmalige wizard: Gmail-watch, Pub/Sub-onderwerp/-abonnement en levering aan de OpenClaw-hook. |
| `gmail run`   | Voer `gog watch serve` plus de lus voor automatische verlenging van de watch op de voorgrond uit.               |

<Note>
De Gateway start `gog gmail watch serve` ook automatisch bij het opstarten zodra `hooks.enabled=true` en `hooks.gmail.account` zijn ingesteld (ingesteld door `gmail setup`). `gmail run` voert dezelfde logica op de voorgrond uit, wat nuttig is voor foutopsporing of wanneer de Gateway-watcher is uitgeschakeld. Zie [Gmail Pub/Sub-integratie](/nl/automation/cron-jobs#gmail-pubsub-integration) voor details over automatisch starten en de `OPENCLAW_SKIP_GMAIL_WATCHER`-opt-out.
</Note>

## `webhooks gmail setup`

```bash
openclaw webhooks gmail setup --account you@example.com
openclaw webhooks gmail setup --account you@example.com --project my-gcp-project --json
openclaw webhooks gmail setup --account you@example.com --hook-url https://gateway.example.com/hooks/gmail
```

Installeert `gcloud` en `gog` als deze ontbreken, verifieert `gcloud`, maakt het Pub/Sub-onderwerp en -abonnement, start de Gmail-watch en schrijft de `hooks.gmail`-configuratie met `hooks.enabled=true`. Geeft `Next: openclaw webhooks gmail run` weer.

### Vereist

| Vlag                | Beschrijving             |
| ------------------- | ----------------------- |
| `--account <email>` | Gmail-account dat moet worden bewaakt. |

### Pub/Sub-opties

| Vlag                    | Standaard                | Beschrijving                                                                                                                             |
| ----------------------- | ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `--project <id>`        | (geen)                 | GCP-project-id (de eigenaar van de OAuth-client). Valt terug op de eigen project-id van het onderwerp en daarna op het project dat is bepaald aan de hand van de `gog`-referenties. |
| `--topic <name>`        | `gog-gmail-watch`      | Naam van het Pub/Sub-onderwerp.                                                                                                                     |
| `--subscription <name>` | `gog-gmail-watch-push` | Naam van het Pub/Sub-abonnement.                                                                                                              |
| `--label <label>`       | `INBOX`                | Gmail-label dat moet worden bewaakt.                                                                                                                   |
| `--push-endpoint <url>` | (geen)                 | Expliciet Pub/Sub-push-eindpunt. Overschrijft Tailscale.                                                                                    |

### OpenClaw-leveringsopties

| Vlag                   | Standaard                                      | Beschrijving                                |
| ---------------------- | -------------------------------------------- | ------------------------------------------ |
| `--hook-url <url>`     | Samengesteld uit `hooks.path` en de Gateway-poort | OpenClaw-webhook-URL.                      |
| `--hook-token <token>` | `hooks.token`, of een gegenereerd token          | OpenClaw-webhooktoken.                    |
| `--push-token <token>` | Gegenereerd token                              | Pushtoken dat wordt doorgestuurd naar `gog watch serve`. |

### `gog watch serve`-opties

| Vlag                  | Standaard         | Beschrijving                                                                                                                                  |
| --------------------- | --------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `--bind <host>`       | `127.0.0.1`     | Bindhost van `gog watch serve`.                                                                                                                 |
| `--port <port>`       | `8788`          | Poort van `gog watch serve`.                                                                                                                      |
| `--path <path>`       | `/gmail-pubsub` | Pad van `gog watch serve`. Wordt gedwongen ingesteld op `/` wanneer Tailscale is ingeschakeld zonder een expliciet doel, omdat Tailscale het pad verwijdert voordat het proxyverkeer doorstuurt. |
| `--include-body`      | `true`          | Neem fragmenten van de e-mailtekst op. Er is geen CLI-vlag om dit uit te schakelen; stel in plaats daarvan `hooks.gmail.includeBody: false` in de configuratie in.                  |
| `--max-bytes <n>`     | `20000`         | Maximaal aantal bytes per tekstfragment.                                                                                                                  |
| `--renew-minutes <n>` | `720` (12h)     | Verleng de Gmail-watch elke N minuten.                                                                                                           |

### Beschikbaar stellen via Tailscale

| Vlag                      | Standaard  | Beschrijving                                                      |
| ------------------------- | -------- | ---------------------------------------------------------------- |
| `--tailscale <mode>`      | `funnel` | Stel het push-eindpunt beschikbaar via Tailscale: `funnel`, `serve` of `off`. |
| `--tailscale-path <path>` | (geen)   | Pad voor Tailscale serve/funnel.                                 |
| `--tailscale-target <t>`  | (geen)   | Doel voor Tailscale serve/funnel (poort, `host:port` of URL).       |

### Uitvoer

| Vlag     | Beschrijving                                       |
| -------- | ------------------------------------------------- |
| `--json` | Geef een machineleesbare samenvatting weer in plaats van tekst. |

## `webhooks gmail run`

```bash
openclaw webhooks gmail run --account you@example.com
```

Voert `gog watch serve` plus de lus voor automatische verlenging van de watch op de voorgrond uit en start `gog watch serve` na een vertraging van 2s opnieuw als deze onverwacht wordt afgesloten.

`run` accepteert dezelfde Pub/Sub-, OpenClaw-leverings-, `gog watch serve`- en Tailscale-vlaggen als `setup`, behalve:

- `--account` is **optioneel** voor `run`; deze valt terug op `hooks.gmail.account`.
- `run` accepteert `--project`, `--push-endpoint` of `--json` **niet**.
- Elke vlag valt terug op de overeenkomende `hooks.gmail.*`-configuratiewaarde (geschreven door `setup`) en daarna op dezelfde ingebouwde standaardwaarde die `setup` gebruikt, met één uitzondering: `--tailscale` is standaard `off` voor `run` (niet `funnel`) wanneer noch de vlag, noch `hooks.gmail.tailscale.mode` is ingesteld.

| Categorie          | Vlaggen                                                                            |
| ----------------- | -------------------------------------------------------------------------------- |
| Pub/Sub           | `--account`, `--topic`, `--subscription`, `--label`                              |
| OpenClaw-levering | `--hook-url`, `--hook-token`, `--push-token`                                     |
| `gog watch serve` | `--bind`, `--port`, `--path`, `--include-body`, `--max-bytes`, `--renew-minutes` |
| Tailscale         | `--tailscale`, `--tailscale-path`, `--tailscale-target`                          |

<Note>
Voor `run` is de waarde van `--topic` het volledige Pub/Sub-onderwerppad (`projects/.../topics/...`), niet alleen de korte onderwerpnaam.
</Note>

## Gerelateerd

- [CLI-referentie](/nl/cli)
- [Webhook-automatisering](/nl/automation/cron-jobs)
- [Gmail Pub/Sub-integratie](/nl/automation/cron-jobs#gmail-pubsub-integration)
