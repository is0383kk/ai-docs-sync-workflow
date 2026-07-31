---
read_when:
    - Je wilt een terminalinterface voor de Gateway (geschikt voor extern gebruik)
    - Je wilt url/token/session vanuit scripts doorgeven
    - Je wilt de TUI in lokale ingesloten modus uitvoeren zonder een Gateway
    - Je wilt openclaw chat of openclaw tui --local gebruiken
summary: CLI-referentie voor `openclaw tui` (door Gateway ondersteunde of lokaal ingebedde terminalinterface)
title: TUI
x-i18n:
    generated_at: "2026-07-27T05:01:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5406f25bbd22c64867296c15112fafcaf8e1580c759e5fdc81fccfb62ae1e318
    source_path: cli/tui.md
    workflow: 16
---

# `openclaw tui`

Open de terminal-UI die met de Gateway is verbonden, of voer deze uit in de lokale ingebedde
modus.

Gerelateerde handleiding: [TUI](/nl/web/tui)

## Opties

| Vlag                         | Standaardwaarde                           | Beschrijving                                                                      |
| ---------------------------- | ----------------------------------------- | --------------------------------------------------------------------------------- |
| `--local`                    | `false`                                   | Uitvoeren met de lokale ingebedde agentruntime in plaats van een Gateway.         |
| `--url <url>`                | `gateway.remote.url` uit de configuratie | WebSocket-URL van de Gateway.                                                      |
| `--token <token>`            | (geen)                                    | Gateway-token indien vereist.                                                     |
| `--password <pass>`          | (geen)                                    | Gateway-wachtwoord indien vereist.                                                |
| `--tls-fingerprint <sha256>` | `gateway.remote.tlsFingerprint`           | Verwachte TLS-certificaatvingerafdruk voor een vastgezette `wss://`-Gateway.       |
| `--session <key>`            | `main` (of `global` wanneer het bereik globaal is) | Sessiesleutel. Binnen een agentwerkruimte wordt die agent automatisch geselecteerd, tenzij een voorvoegsel is opgegeven. |
| `--deliver`                  | `false`                                   | Antwoorden van de assistent via geconfigureerde kanalen afleveren.                |
| `--thinking <level>`         | (standaardwaarde van model)                | Overschrijving van het denkniveau.                                                |
| `--message <text>`           | (geen)                                    | Na het verbinden een eerste bericht verzenden.                                    |
| `--timeout-ms <ms>`          | `agents.defaults.timeoutSeconds`          | Time-out van de agent. Ongeldige waarden registreren een waarschuwing en worden genegeerd. |
| `--history-limit <n>`        | `200`                                     | Aantal geschiedenisitems dat bij het koppelen wordt geladen.                      |

Aliassen: `openclaw chat` en `openclaw terminal` roepen deze opdracht aan waarbij
`--local` impliciet wordt gebruikt.

## Opmerkingen

- `--local` kan niet worden gecombineerd met `--url`, `--token`, `--password` of `--tls-fingerprint`.
- `tui` herleidt waar mogelijk geconfigureerde SecretRefs voor Gateway-authenticatie voor token-/wachtwoordauthenticatie
  (`env`-, `file`- en `exec`-providers).
- Zonder expliciete URL of poort volgt `tui` de actieve lokale Gateway-poort
  die door de actieve Gateway is vastgelegd. Expliciete `--url`, `OPENCLAW_GATEWAY_URL`,
  `OPENCLAW_GATEWAY_PORT` en de configuratie van een externe Gateway hebben voorrang.
- Wanneer de TUI vanuit de map van een geconfigureerde agentwerkruimte wordt gestart, selecteert deze
  die agent automatisch als standaardwaarde voor de sessiesleutel (tenzij `--session` expliciet
  `agent:<id>:...` is).
- De lokale modus gebruikt de ingebedde agentruntime rechtstreeks. De meeste lokale tools werken,
  maar functies die alleen via de Gateway beschikbaar zijn, zijn niet beschikbaar.
- De lokale modus voegt `/auth [provider]` toe aan de opdrachtmogelijkheden van de TUI.
- Goedkeuringspoorten van Plugins blijven van toepassing in de lokale modus: tools waarvoor goedkeuring is vereist,
  vragen in de terminal om een beslissing; niets wordt stilzwijgend automatisch goedgekeurd.
- Sessie[doelen](/nl/tools/goal) verschijnen in de voettekst en kunnen worden beheerd met
  `/goal`.

## Voorbeelden

```bash
openclaw chat
openclaw tui --local
openclaw tui
openclaw tui --url ws://127.0.0.1:18789 --token <token>
openclaw tui --session main --deliver
openclaw chat --message "Vergelijk mijn configuratie met de documentatie en vertel me wat ik moet repareren"
# wanneer uitgevoerd in een agentwerkruimte, wordt die agent automatisch afgeleid
openclaw tui --session bugfix
```

## Herstelcyclus voor configuratie

Gebruik de lokale modus om de ingebedde agent de huidige configuratie te laten inspecteren, deze
met de documentatie te vergelijken en vanuit dezelfde terminal te helpen herstellen.

Als `openclaw config validate` al mislukt, voer dan eerst `openclaw configure` of
`openclaw doctor --fix` uit; `openclaw chat` omzeilt de
beveiliging tegen ongeldige configuratie niet.

```bash
openclaw chat
```

Vervolgens in de TUI:

```text
!openclaw config file
!openclaw docs gateway auth token secretref
!openclaw config validate
!openclaw doctor
```

Pas gerichte correcties toe met `openclaw config set` of `openclaw configure` en
voer daarna `openclaw config validate` opnieuw uit. Zie [TUI](/nl/web/tui) en
[Configuratie](/nl/cli/config).

## Gerelateerd

- [CLI-referentie](/nl/cli)
- [TUI](/nl/web/tui)
- [Doel](/nl/tools/goal)
