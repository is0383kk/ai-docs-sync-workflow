---
read_when:
    - Je gebruikt nog steeds `openclaw daemon ...` in scripts
    - Je hebt opdrachten voor de servicelevenscyclus nodig (installeren/starten/stoppen/herstarten/status)
summary: CLI-referentie voor `openclaw daemon` (verouderde alias voor beheer van de Gateway-service)
title: Daemon
x-i18n:
    generated_at: "2026-07-27T04:59:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 629852ebf3efe86dedc4c84f6ddc9349b25ddde832df5d78521641fe4b137658
    source_path: cli/daemon.md
    workflow: 16
---

# `openclaw daemon`

Verouderde alias voor het beheer van de Gateway-service. `openclaw daemon ...` verwijst naar dezelfde opdrachten voor servicebeheer als `openclaw gateway ...`. Gebruik bij voorkeur [`openclaw gateway`](/nl/cli/gateway) voor actuele documentatie en voorbeelden.

## Gebruik

```bash
openclaw daemon status
openclaw daemon install
openclaw daemon start
openclaw daemon stop
openclaw daemon restart
openclaw daemon uninstall
```

## Subopdrachten en opties

| Subopdracht  | Opties                                                                                          |
| ----------- | ------------------------------------------------------------------------------------------------ |
| `status`    | `--url`, `--token`, `--password`, `--timeout`, `--no-probe`, `--require-rpc`, `--deep`, `--json` |
| `install`   | `--port`, `--runtime <node>`, `--token`, `--wrapper <path>`, `--force`, `--json`                 |
| `uninstall` | `--json`                                                                                         |
| `start`     | `--json`                                                                                         |
| `stop`      | `--json`, `--disable` (alleen launchd: onderdruk KeepAlive/RunAtLoad blijvend tot de volgende start) |
| `restart`   | `--force`, `--safe`, `--skip-deferral`, `--wait <duration>`, `--json`                            |

- `status`: toont de installatiestatus van de service (launchd/systemd/schtasks) en controleert de status van de Gateway.
- `install`: installeert de service; `--force` installeert een bestaande installatie opnieuw of overschrijft deze.
- `restart --safe`: vraagt de actieve Gateway om lopende werkzaamheden vooraf te controleren en één samengevoegde herstart in te plannen nadat het werk is afgerond, met een limiet van 5 minuten. Wanneer die tijdslimiet verstrijkt, wordt de herstart alsnog geforceerd. Een gewone `restart` gebruikt de servicebeheerder rechtstreeks; `--force` is de onmiddellijke overschrijving.
- `restart --safe --skip-deferral`: omzeilt de uitstelblokkering voor actieve werkzaamheden, zodat de Gateway onmiddellijk opnieuw wordt gestart, zelfs wanneer blokkeringen worden gemeld. Vereist `--safe`.

## Opmerkingen

- `status` herleidt waar mogelijk geconfigureerde SecretRefs voor authenticatie van de controle. Als een vereiste SecretRef niet kan worden herleid, meldt `status --json` `rpc.authWarning`; geef `--token`/`--password` expliciet door of los eerst de geheime bron op. Waarschuwingen over onopgeloste authenticatie worden onderdrukt zodra de controle verder slaagt.
- `status --deep` voegt een systeemscan op basis van beste inspanning toe voor andere Gateway-achtige services (toont opruimtips; één Gateway per machine blijft de aanbeveling) en voert configuratievalidatie uit in een Plugin-bewuste modus, waarbij waarschuwingen uit Plugin-manifesten worden weergegeven die door het snelle standaardpad worden overgeslagen.
- Bij Linux-installaties met systemd inspecteren controles op tokenafwijkingen zowel `Environment=` als `EnvironmentFile=` als bronnen voor units.
- Controles op tokenafwijkingen herleiden `gateway.auth.token`-SecretRefs met behulp van de samengevoegde runtimeomgeving (eerst de opdrachtomgeving van de service, daarna de procesomgeving). Als tokenauthenticatie niet daadwerkelijk actief is (`gateway.auth.mode` van `password`/`none`/`trusted-proxy`, of niet ingesteld terwijl het wachtwoord voorrang kan krijgen), wordt het herleiden van het configuratietoken overgeslagen.
- `install` controleert of een door een SecretRef beheerde `gateway.auth.token` kan worden herleid, maar slaat de herleide waarde nooit op in de omgevingsmetadata van de service; als herleiden niet lukt, wordt de installatie uit veiligheidsoverwegingen afgebroken.
- Als zowel `gateway.auth.token` als `gateway.auth.password` zijn geconfigureerd en `gateway.auth.mode` niet is ingesteld, blokkeert `install` totdat je de modus expliciet instelt.
- Op macOS houdt `install` de LaunchAgent-plists en het gegenereerde omgevingsbestand/wrapperscript uitsluitend toegankelijk voor de eigenaar (modus `0600`/`0700`), in plaats van geheimen in `EnvironmentVariables` in te sluiten.
- Meerdere Gateways op één host uitvoeren: isoleer poorten, configuratie/status en werkruimten. Zie [Meerdere Gateways](/nl/gateway#multiple-gateways-same-host).

## Gerelateerd

- [CLI-referentie](/nl/cli)
- [Gateway-draaiboek](/nl/gateway)
