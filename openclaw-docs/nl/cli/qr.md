---
read_when:
    - Je wilt snel een mobiele Node-app aan een Gateway koppelen
    - Je hebt uitvoer van de installatiecode nodig om deze op afstand/handmatig te delen
summary: CLI-referentie voor `openclaw qr` (QR-code voor mobiele koppeling en configuratiecode genereren)
title: QR
x-i18n:
    generated_at: "2026-07-27T05:29:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f9d60a58126eae7eec5979f28bb511a09fa52b68cdd73727fca0b2de74efa84a
    source_path: cli/qr.md
    workflow: 16
---

# `openclaw qr`

Genereer een QR-code voor mobiele koppeling en een installatiecode op basis van je huidige Gateway-configuratie.

```bash
openclaw qr
openclaw qr --setup-code-only
openclaw qr --json
openclaw qr --remote
openclaw qr --limited
openclaw qr --url wss://gateway.example/ws
```

Officiële OpenClaw-apps voor iOS en Android maken automatisch verbinding wanneer de metadata van hun installatiecode overeenkomt. Als een verzoek in behandeling blijft (bijvoorbeeld voor een niet-officiële client of niet-overeenkomende metadata), controleer en keur je het goed:

```bash
openclaw devices list
openclaw devices approve <requestId>
```

## Opties

- `--remote`: geeft de voorkeur aan `gateway.remote.url`; valt terug op `gateway.tailscale.mode=serve|funnel` als die URL niet is ingesteld. Negeert `device-pair`-Plugin `publicUrl`.
- `--url <url>`: overschrijft de Gateway-URL die in de payload wordt gebruikt
- `--public-url <url>`: overschrijft de openbare URL die in de payload wordt gebruikt
- `--token <token>`: overschrijft het Gateway-token waarmee de bootstrapflow zich verifieert
- `--password <password>`: overschrijft het Gateway-wachtwoord waarmee de bootstrapflow zich verifieert
- `--limited`: laat administratieve Gateway-toegang weg uit het overgedragen operatortoken
- `--setup-code-only`: drukt alleen de installatiecode af
- `--no-ascii`: slaat de ASCII-weergave van de QR-code over
- `--json`: voert JSON uit (`setupCode`, `gatewayUrl`, optioneel `gatewayUrls`, `auth`, `access`, optioneel `accessDowngraded`, `urlSource`)

`--token` en `--password` sluiten elkaar wederzijds uit.

## Inhoud van de installatiecode

De installatiecode bevat een ondoorzichtige, kortlevende `bootstrapToken`, niet het gedeelde Gateway-token of -wachtwoord. Voor een `wss://`-eindpunt (of een loopback op dezelfde host) verstrekt de standaardbootstrapflow:

- een primair `node`-token met `scopes: []`
- een volledig native mobiel `operator`-overdrachtstoken met `operator.admin`, `operator.approvals`, `operator.read`, `operator.talk.secrets` en `operator.write`

Gebruik `--limited` om hetzelfde Node-token te behouden en tegelijkertijd `operator.admin` weg te laten uit de operatoroverdracht. Het bereik voor koppelingsmutaties wordt nooit via een installatiecode overgedragen.

Installatie via `ws://` in platte tekst op het LAN blijft beschikbaar, maar OpenClaw gebruikt automatisch het beperkte profiel omdat een netwerkwaarnemer het bearer-bootstrap-token kan onderscheppen en sneller kan gebruiken. Configureer `wss://` of Tailscale Serve en genereer vervolgens een nieuwe code om volledige toegang te krijgen.

## Gateway-URL-resolutie

Mobiele koppeling weigert standaard Tailscale-/openbare `ws://`-Gateway-URL's: gebruik daarvoor Tailscale Serve/Funnel of een `wss://`-Gateway-URL. Privé-LAN-adressen en `.local`-Bonjour-hosts blijven ondersteund via gewone `ws://`, met beperkte operatortoegang zoals hierboven beschreven.

Wanneer de geselecteerde Gateway-URL afkomstig is van `gateway.bind=lan`, controleert OpenClaw ook permanente `tailscale serve status --json`-routes. Elke HTTPS-Serve-root die de loopbackpoort van de actieve Gateway proxyt, wordt als fallback opgenomen. De QR-opdracht voegt deze fallback alleen toe voor `lan`; `custom` en `tailnet` behouden hun expliciet geadverteerde routes. Huidige iOS-clients testen de geadverteerde routes in volgorde en slaan de eerste bereikbare route op; het verouderde veld `url` blijft ongewijzigd voor oudere clients.

Met `--remote` is een van `gateway.remote.url` of `gateway.tailscale.mode=serve|funnel` vereist.

## Authenticatieresolutie (geen `--remote`)

Wanneer geen CLI-overschrijving voor authenticatie wordt doorgegeven, worden lokale SecretRefs voor Gateway-authenticatie als volgt opgelost:

| Voorwaarde                                                                                                                   | Wordt opgelost als                        |
| ---------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------- |
| `gateway.auth.mode="token"`, of afgeleide modus zonder doorslaggevende wachtwoordbron                                                | `gateway.auth.token`                      |
| `gateway.auth.mode="password"`, of afgeleide modus zonder doorslaggevend token uit authenticatie/omgeving                                         | `gateway.auth.password`                   |
| Zowel `gateway.auth.token` als `gateway.auth.password` zijn geconfigureerd (inclusief SecretRefs) en `gateway.auth.mode` is niet ingesteld | mislukt; stel `gateway.auth.mode` expliciet in |

## Authenticatieresolutie (`--remote`)

Als daadwerkelijk actieve externe referenties als SecretRefs zijn geconfigureerd en noch `--token` noch `--password` wordt doorgegeven, haalt de opdracht ze op uit de momentopname van de actieve Gateway. Als de Gateway niet beschikbaar is, mislukt de opdracht onmiddellijk.

<Note>
Dit opdrachtpad vereist een Gateway die de RPC-methode `secrets.resolve` ondersteunt. Oudere Gateways retourneren een fout voor een onbekende methode.
</Note>

## Gerelateerd

- [CLI-referentie](/nl/cli)
- [Apparaten](/nl/cli/devices)
- [Koppelen](/nl/cli/pairing)
