---
read_when:
    - Je beheert gekoppelde nodes (camera's, scherm, canvas)
    - Je moet verzoeken goedkeuren of Node-opdrachten aanroepen
summary: CLI-referentie voor `openclaw nodes` (status, koppelen, aanroepen, camera/canvas/scherm/locatie/melding)
title: Nodes
x-i18n:
    generated_at: "2026-07-27T05:00:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 53003bcd3d30b0e754aa0717452700595c0cf69d9ecd6301b8a1bf320ea1838a
    source_path: cli/nodes.md
    workflow: 16
---

# `openclaw nodes`

Beheer gekoppelde Nodes (apparaten) en roep Node-mogelijkheden aan.

Gerelateerd: [Overzicht van Nodes](/nl/nodes) - [Actieve computeraanwezigheid](/nl/nodes/presence) - [Camera-Nodes](/nl/nodes/camera) - [Afbeeldings-Nodes](/nl/nodes/images)

Algemene opties voor elke subopdracht: `--url <url>`, `--token <token>`, `--timeout <ms>` (standaard `10000`), `--json`.

## Status

```bash
openclaw nodes status
openclaw nodes status --connected
openclaw nodes status --last-connected 24h
openclaw nodes list
openclaw nodes describe --node <idOrNameOrIp>
```

`status` en `list` accepteren beide `--connected` (alleen verbonden Nodes) en `--last-connected <duration>` (bijv. `24h`, `7d`; alleen Nodes die binnen de duur verbinding hebben gemaakt). `list` toont wachtende en gekoppelde Nodes in afzonderlijke tabellen, waarbij gekoppelde rijen de tijd sinds de recentste verbinding (Last Connect) bevatten; `status` toont één samengevoegde tabel met per Node details over mogelijkheden, versie en laatste invoer. Een verbonden macOS-Node rapporteert de laatste invoer pas nadat de gebruiker **Actieve computerdetectie** heeft ingeschakeld en Toegankelijkheid heeft verleend; de recentste rij wordt gemarkeerd met `active`. Zie [Actieve computeraanwezigheid](/nl/nodes/presence). `describe` geeft de mogelijkheden, machtigingen, activiteit en effectieve/wachtende aanroepopdrachten van één Node weer.

## Koppelen

```bash
openclaw nodes pending
openclaw nodes approve <requestId>
openclaw nodes reject <requestId>
openclaw nodes remove --node <id|name|ip>
openclaw nodes rename --node <id|name|ip> --name <displayName>
```

Deze opdrachten beheren de door de Gateway beheerde `node.pair.*`-opslag, los van de apparaatkoppeling (`openclaw devices approve`) die de WS-`connect`-handshake van de Node afschermt. Zie [Nodes](/nl/nodes) voor hoe beide zich tot elkaar verhouden.

- `remove` trekt de gekoppelde rolvermelding van de Node in. Voor een door een apparaat ondersteunde Node trekt dit de rol `node` in de opslag voor apparaatkoppelingen in en verbreekt het de Node-rolsessies: een apparaat met gemengde rollen behoudt zijn rij en verliest alleen de rol `node`; een rij voor een apparaat met alleen een Node wordt verwijderd. Ook wordt elke overeenkomende verouderde, door de Gateway beheerde Node-koppelingsrecord gewist.
- `pending` heeft alleen het bereik `operator.pairing` nodig.
- `gateway.nodes.pairing.autoApproveCidrs` kan de wachtstap overslaan voor expliciet vertrouwde, eerste apparaatkoppelingen van `role: node`. Standaard uitgeschakeld; keurt rolupgrades niet goed.
- `gateway.nodes.pairing.sshVerify` (standaard ingeschakeld) keurt een eerste apparaatkoppeling van `role: node` automatisch goed wanneer de Gateway de apparaatsleutel via SSH naar de Node-host kan verifiëren; het eerste mogelijkhedenoppervlak wordt in dezelfde stap goedgekeurd. Zie [Node-koppeling](/nl/gateway/pairing#ssh-verified-device-auto-approval-default).
- De bereikvereisten voor `approve` volgen de gedeclareerde opdrachten van het wachtende verzoek:
  - verzoek zonder opdracht: `operator.pairing`
  - gewone Node-opdrachten: `operator.pairing` + `operator.write`
  - beheerdergevoelige opdrachten (`system.run`, `system.run.prepare`, `system.which`, `browser.proxy`, `fs.listDir` en `system.execApprovals.get/set`): `operator.pairing` + `operator.admin`
- Bereik van `remove`: `operator.pairing` kan Node-rijen zonder operator verwijderen; een aanroeper met een apparaattoken die zijn eigen Node-rol op een apparaat met gemengde rollen intrekt, heeft daarnaast `operator.admin` nodig.

## Aanroepen

```bash
openclaw nodes invoke --node <id> --command system.which --params '{"bins":["uname"]}'
```

Vlaggen:

- `--command <command>` (vereist): bijv. `canvas.eval`.
- `--params <json>`: tekenreeks met JSON-object (standaard `{}`).
- `--invoke-timeout <ms>`: time-out voor het aanroepen van de Node (standaard `15000`).
- `--idempotency-key <key>`: optionele idempotentiesleutel.

`system.run` en `system.run.prepare` worden hier geblokkeerd; gebruik in plaats daarvan de tool `exec` met `host=node` voor shelluitvoering. `system.which` is toegestaan via `invoke`.

## Meldingen, pushberichten, locatie en scherm

```bash
openclaw nodes notify --node <id> --title "Build" --body "Done" --priority timeSensitive
openclaw nodes push --node <id> --title "OpenClaw" --environment sandbox
openclaw nodes location get --node <id> --accuracy precise
openclaw nodes screen record --node <id> --duration 10s --fps 10 --out ./clip.mp4
```

- `notify` stuurt een lokale melding naar een Node die `system.notify` declareert, waaronder macOS-, iOS-, Android- en directe watchOS-Nodes. Voor directe bezorging via watchOS moet OpenClaw actief zijn. Vereist `--title` of `--body`. Opties: `--sound <name>`, `--priority <passive|active|timeSensitive>`, `--delivery <system|overlay|auto>` (standaard `system`), `--invoke-timeout <ms>` (standaard `15000`).
- `push` stuurt een APNs-testpushbericht naar een iOS-Node. Opties: `--title <text>` (standaard `OpenClaw`), `--body <text>`, `--environment <sandbox|production>` om de gedetecteerde APNs-omgeving te overschrijven.
- `location get` haalt de huidige locatie van de Node op. Opties: `--max-age <ms>` (een gecachte positiebepaling hergebruiken), `--accuracy <coarse|balanced|precise>`, `--location-timeout <ms>` (standaard `10000`), `--invoke-timeout <ms>` (standaard `20000`).
- `screen record` legt een korte clip vast en geeft het opgeslagen pad weer (of schrijft JSON met `--json`). Opties: `--screen <index>` (standaard `0`), `--duration <ms|10s>` (standaard `10000`), `--fps <fps>` (standaard `10`), `--no-audio`, `--out <path>`, `--invoke-timeout <ms>` (standaard `120000`).

Opdrachten voor Camera en Canvas hebben eigen documentatie: [Camera-Nodes](/nl/nodes/camera), [Canvas](/nl/platforms/mac/canvas). Canvas wordt geïmplementeerd door de meegeleverde experimentele Canvas-Plugin; de kern behoudt `openclaw nodes canvas` als compatibiliteitskoppelpunt.

## Gerelateerd

- [CLI-referentie](/nl/cli)
- [Nodes](/nl/nodes)
