---
read_when:
    - Je gebruikt de spraakoproepplugin en wilt elk CLI-ingangspunt
    - Je hebt tabellen met vlaggen en standaardwaarden nodig voor setup, smoke, call, continue, speak, dtmf, end, status, tail, latency, expose en start
summary: CLI-referentie voor `openclaw voicecall` (opdrachtinterface van de voice-call-plugin)
title: Spraakoproep
x-i18n:
    generated_at: "2026-07-27T05:42:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: aec445886cccb79c9212dd9f1f448ff9634274deb380632be786478c9bb29670
    source_path: cli/voicecall.md
    workflow: 16
---

# `openclaw voicecall`

`voicecall` is een door een plugin geleverde opdracht. Deze verschijnt alleen wanneer de voice-call-
plugin is geïnstalleerd en ingeschakeld.

Wanneer de Gateway actief is, worden operationele opdrachten (`call`, `start`,
`continue`, `speak`, `dtmf`, `end`, `status`) doorgestuurd naar de voice-call-runtime
van die Gateway. Als geen Gateway bereikbaar is, vallen ze terug op een zelfstandige
CLI-runtime.

## Subopdrachten

```bash
openclaw voicecall setup    [--json]
openclaw voicecall smoke    [-t <phone>] [--message <text>] [--mode <m>] [--yes] [--json]
openclaw voicecall call     -m <text> [-t <phone>] [--mode <m>]
openclaw voicecall start    --to <phone> [--message <text>] [--mode <m>]
openclaw voicecall continue --call-id <id> --message <text>
openclaw voicecall speak    --call-id <id> --message <text>
openclaw voicecall dtmf     --call-id <id> --digits <digits>
openclaw voicecall end      --call-id <id>
openclaw voicecall status   [--call-id <id>] [--json]
openclaw voicecall tail     [--file <path>] [--since <n>] [--poll <ms>]
openclaw voicecall latency  [--file <path>] [--last <n>]
openclaw voicecall expose   [--mode <m>] [--path <p>] [--port <port>] [--serve-path <p>]
```

| Subopdracht | Beschrijving                                                     |
| ---------- | --------------------------------------------------------------- |
| `setup`    | Toon controles voor de gereedheid van de provider en Webhook.                     |
| `smoke`    | Voer gereedheidscontroles uit; plaats alleen met `--yes` een live testoproep. |
| `call`     | Start een uitgaande spraakoproep.                                |
| `start`    | Alias voor `call`, waarbij `--to` verplicht en `--message` optioneel is. |
| `continue` | Spreek een bericht uit en wacht op het volgende antwoord.                 |
| `speak`    | Spreek een bericht uit zonder op een antwoord te wachten.                 |
| `dtmf`     | Stuur DTMF-cijfers naar een actieve oproep.                             |
| `end`      | Beëindig een actieve oproep.                                         |
| `status`   | Inspecteer actieve oproepen (of één via `--call-id`).                   |
| `tail`     | Volg `calls.jsonl` (nuttig tijdens providertests).              |
| `latency`  | Vat statistieken over beurtlatentie uit `calls.jsonl` samen.              |
| `expose`   | Schakel Tailscale Serve/Funnel voor het Webhook-eindpunt in of uit.         |

## Installatie en rooktest

### `setup`

Drukt standaard voor mensen leesbare gereedheidscontroles af. Geef `--json` door voor scripts.

```bash
openclaw voicecall setup
openclaw voicecall setup --json
```

### `smoke`

Voert dezelfde gereedheidscontroles uit. Plaatst alleen een echte telefoonoproep wanneer zowel
`--to` als `--yes` aanwezig zijn.

| Vlag               | Standaard                           | Beschrijving                             |
| ------------------ | --------------------------------- | --------------------------------------- |
| `-t, --to <phone>` | (geen)                            | Telefoonnummer dat voor een live rooktest moet worden gebeld.  |
| `--message <text>` | `OpenClaw voice call smoke test.` | Bericht dat tijdens de rooktestoproep wordt uitgesproken. |
| `--mode <mode>`    | `notify`                          | Oproepmodus: `notify` of `conversation`.  |
| `--yes`            | `false`                           | Plaats de live uitgaande oproep daadwerkelijk.  |
| `--json`           | `false`                           | Druk machineleesbare JSON af.            |

```bash
openclaw voicecall smoke
openclaw voicecall smoke --to "+15555550123"        # droge uitvoering
openclaw voicecall smoke --to "+15555550123" --yes  # live meldingsoproep
```

<Note>
Voor externe providers (`plivo`, `telnyx`, `twilio`) vereisen `setup` en `smoke` een openbare Webhook-URL van `publicUrl`, een tunnel of beschikbaarstelling via Tailscale. Een terugvaloptie via loopback of een privéserver wordt geweigerd omdat telecomaanbieders deze niet kunnen bereiken.
</Note>

## Levenscyclus van oproepen

### `call`

Start een uitgaande spraakoproep.

| Vlag                   | Verplicht | Standaard           | Beschrijving                                                                |
| ---------------------- | -------- | ----------------- | -------------------------------------------------------------------------- |
| `-m, --message <text>` | ja      | (geen)            | Bericht dat wordt uitgesproken wanneer de oproep verbinding maakt.                                   |
| `-t, --to <phone>`     | nee       | configuratie `toNumber` | E.164-telefoonnummer dat moet worden gebeld.                                                |
| `--mode <mode>`        | nee       | `conversation`    | Oproepmodus: `notify` (ophangen na het bericht) of `conversation` (openhouden). |

```bash
openclaw voicecall call --to "+15555550123" --message "Hallo"
openclaw voicecall call -m "Let op" --mode notify
```

### `start`

Alias voor `call` met een andere standaardvorm voor vlaggen.

| Vlag               | Verplicht | Standaard        | Beschrijving                              |
| ------------------ | -------- | -------------- | ---------------------------------------- |
| `--to <phone>`     | ja      | (geen)         | Telefoonnummer dat moet worden gebeld.                    |
| `--message <text>` | nee       | (geen)         | Bericht dat wordt uitgesproken wanneer de oproep verbinding maakt. |
| `--mode <mode>`    | nee       | `conversation` | Oproepmodus: `notify` of `conversation`.   |

### `continue`

Spreek een bericht uit en wacht op een antwoord.

| Vlag               | Verplicht | Beschrijving       |
| ------------------ | -------- | ----------------- |
| `--call-id <id>`   | ja      | Oproep-ID.          |
| `--message <text>` | ja      | Bericht om uit te spreken. |

### `speak`

Spreek een bericht uit zonder op een antwoord te wachten.

| Vlag               | Verplicht | Beschrijving       |
| ------------------ | -------- | ----------------- |
| `--call-id <id>`   | ja      | Oproep-ID.          |
| `--message <text>` | ja      | Bericht om uit te spreken. |

### `dtmf`

Stuur DTMF-cijfers naar een actieve oproep.

| Vlag                | Verplicht | Beschrijving                                      |
| ------------------- | -------- | ------------------------------------------------ |
| `--call-id <id>`    | ja      | Oproep-ID.                                         |
| `--digits <digits>` | ja      | DTMF-cijfers (bijvoorbeeld `ww123456#` voor wachttijden). |

### `end`

Beëindig een actieve oproep.

| Vlag             | Verplicht | Beschrijving |
| ---------------- | -------- | ----------- |
| `--call-id <id>` | ja      | Oproep-ID.    |

### `status`

Inspecteer actieve oproepen.

| Vlag             | Standaard | Beschrijving                  |
| ---------------- | ------- | ---------------------------- |
| `--call-id <id>` | (geen)  | Beperk de uitvoer tot één oproep. |
| `--json`         | `false` | Druk machineleesbare JSON af. |

```bash
openclaw voicecall status
openclaw voicecall status --json
openclaw voicecall status --call-id <id>
```

## Logboeken en statistieken

### `tail`

Volg het JSONL-logboek voor spraakoproepen. Drukt bij het starten de laatste `--since` regels af en
streamt vervolgens nieuwe regels zodra ze worden geschreven.

| Vlag            | Standaard                    | Beschrijving                    |
| --------------- | -------------------------- | ------------------------------ |
| `--file <path>` | bepaald vanuit de pluginopslag | Pad naar `calls.jsonl`.         |
| `--since <n>`   | `25`                       | Regels die vóór het volgen moeten worden afgedrukt. |
| `--poll <ms>`   | `250` (minimaal 50)         | Pollinterval in milliseconden. |

### `latency`

Vat statistieken over beurtlatentie en luisterwachttijd uit `calls.jsonl` samen. De uitvoer is
JSON met samenvattingen voor `recordsScanned`, `turnLatency` en `listenWait`.

| Vlag            | Standaard                    | Beschrijving                          |
| --------------- | -------------------------- | ------------------------------------ |
| `--file <path>` | bepaald vanuit de pluginopslag | Pad naar `calls.jsonl`.               |
| `--last <n>`    | `200` (minimaal 1)          | Aantal recente records om te analyseren. |

## Webhooks beschikbaar stellen

### `expose`

Schakel de Tailscale Serve/Funnel-configuratie voor de spraak-Webhook in of uit,
of wijzig deze.

| Vlag                  | Standaard                                   | Beschrijving                                     |
| --------------------- | ----------------------------------------- | ----------------------------------------------- |
| `--mode <mode>`       | `funnel`                                  | `off`, `serve` (tailnet) of `funnel` (openbaar). |
| `--path <path>`       | configuratie `tailscale.path` of `--serve-path` | Beschikbaar te stellen Tailscale-pad.                       |
| `--port <port>`       | configuratie `serve.port` of `3334`             | Lokale Webhook-poort.                             |
| `--serve-path <path>` | configuratie `serve.path` of `/voice/webhook`   | Lokaal Webhook-pad.                             |

```bash
openclaw voicecall expose --mode serve
openclaw voicecall expose --mode funnel
openclaw voicecall expose --mode off
```

<Warning>
Stel het Webhook-eindpunt alleen beschikbaar voor netwerken die je vertrouwt. Geef waar mogelijk de voorkeur aan Tailscale Serve boven Funnel.
</Warning>

## Gerelateerd

- [CLI-referentie](/nl/cli)
- [Plugin voor spraakoproepen](/nl/plugins/voice-call)
