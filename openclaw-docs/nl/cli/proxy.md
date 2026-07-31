---
read_when:
    - Je moet door de operator beheerde proxyrouting valideren vóór de implementatie
    - Je moet OpenClaw-transportverkeer lokaal vastleggen voor foutopsporing
    - Je wilt debugproxysessies, blobs of ingebouwde queryvoorinstellingen inspecteren
summary: CLI-referentie voor `openclaw proxy`, inclusief door de operator beheerde proxyvalidatie en de lokale inspecteur voor het vastleggen van debugproxyverkeer
title: Proxyserver
x-i18n:
    generated_at: "2026-07-27T05:05:39Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 91583f785032bfffe455a1963804108550f6fbb735ac4de1dd91d0ca5ae0df35
    source_path: cli/proxy.md
    workflow: 16
---

# `openclaw proxy`

Valideer door de operator beheerde proxyrouting, of voer de lokale expliciete debugproxy uit en inspecteer het vastgelegde verkeer.

```bash
openclaw proxy validate [--json] [--proxy-url <url>] [--proxy-ca-file <path>] [--allowed-url <url>] [--denied-url <url>] [--apns-reachable] [--apns-authority <url>] [--timeout-ms <ms>]
openclaw proxy start [--host <host>] [--port <port>]
openclaw proxy run [--host <host>] [--port <port>] -- <cmd...>
openclaw proxy coverage
openclaw proxy sessions [--limit <count>]
openclaw proxy query --preset <name> [--session <id>]
openclaw proxy blob --id <blobId>
openclaw proxy purge
```

`validate` voert voorafgaande controles uit op een door de operator beheerde forward proxy. De overige opdrachten zijn debughulpmiddelen voor onderzoek op transportniveau: start een lokale proxy die verkeer vastlegt, voer er een onderliggende opdracht door uit, vermeld vastlegsessies, bevraag verkeerspatronen, lees vastgelegde blobs en wis lokale vastleggegevens.

## Valideren

Controleert de effectieve URL van de door de operator beheerde proxy uit `--proxy-url`, de configuratie (`proxy.proxyUrl`) of `OPENCLAW_PROXY_URL`, in die prioriteitsvolgorde. Meldt een configuratieprobleem als geen proxy is ingeschakeld en geconfigureerd; geef `--proxy-url` door voor een eenmalige voorafgaande controle zonder de configuratie te wijzigen.

Beheerde proxy-URL's gebruiken `http://` voor een gewone forward-proxylistener, of `https://` wanneer OpenClaw zelf een TLS-verbinding met het proxyeindpunt moet openen voordat proxyverzoeken worden verzonden. Gebruik `--proxy-ca-file` om een privé-CA voor die TLS-verbinding te vertrouwen.

Standaard wordt het volgende uitgevoerd:

- één **toegestane** controle op `https://example.com/` (overschrijf/voeg toe met `--allowed-url`, herhaalbaar)
- één **geweigerde** controle op een tijdelijke loopback-kanarie (overschrijf met `--denied-url`, herhaalbaar)

Aangepaste `--denied-url`-doelen werken fail-closed: zowel HTTP-antwoorden als ambigue transportfouten gelden als fouten, tenzij je onafhankelijk een implementatiespecifiek weigeringssignaal kunt verifiëren. De ingebouwde loopback-kanarie is het enige doel waarbij een transportfout geldt als bewijs van blokkering.

Voeg `--apns-reachable` toe om ook een APNs HTTP/2 CONNECT-tunnel via de proxy te openen en te bevestigen dat de APNs-sandbox antwoordt. De test verzendt opzettelijk een ongeldig providertoken, zodat een APNs-antwoord met `403 InvalidProviderToken` geldt als een succesvol bereikbaarheidsignaal (niet als een fout).

### Opties

| Vlag                     | Effect                                                                                                             |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------ |
| `--json`                 | druk machineleesbare JSON af                                                                                        |
| `--proxy-url <url>`      | valideer deze `http://`-/`https://`-proxy-URL in plaats van de configuratie of omgeving                                              |
| `--proxy-ca-file <path>` | vertrouw dit PEM-CA-bestand voor TLS-verificatie van een HTTPS-proxyeindpunt                                             |
| `--allowed-url <url>`    | bestemming die naar verwachting via de proxy slaagt (herhaalbaar)                                                     |
| `--denied-url <url>`     | bestemming die naar verwachting door de proxy wordt geblokkeerd (herhaalbaar)                                                       |
| `--apns-reachable`       | verifieer ook dat APNs HTTP/2 voor de sandbox via de proxy bereikbaar is                                                     |
| `--apns-authority <url>` | te testen APNs-authority (standaard `https://api.sandbox.push.apple.com`; productie is `https://api.push.apple.com`) |
| `--timeout-ms <ms>`      | time-out per verzoek                                                                                                |

Wordt afgesloten met code 1 wanneer de proxyconfiguratie of bestemmingscontroles mislukken.

Zie [Netwerkproxy](/nl/security/network-proxy) voor implementatierichtlijnen en de semantiek van weigeringen.

## Debugproxy

`start` start een lokale proxy die verkeer vastlegt en drukt de URL, het pad naar het CA-certificaat en het pad naar de vastlegdatabase af; stop met Ctrl+C. Standaard wordt aan `127.0.0.1` gebonden, tenzij `--host` is ingesteld.

`run` start een lokale debugproxy en voert vervolgens `<cmd...>` (na `--`) uit met de proxyomgeving toegepast, binnen een eigen vastlegsessie.

De directe upstreamdoorsturing van de debugproxy opent upstreamsockets voor diagnostiek. Wanneer de beheerde proxymodus van OpenClaw actief is, is directe doorsturing voor proxyverzoeken en CONNECT-tunnels standaard uitgeschakeld; stel `OPENCLAW_DEBUG_PROXY_ALLOW_DIRECT_CONNECT_WITH_MANAGED_PROXY=1` alleen in voor goedgekeurde lokale diagnostiek.

`coverage` drukt een JSON-rapport af (`summary` + `entries` per transport) waarin staat welke transporten worden vastgelegd, alleen via de proxy lopen of niet worden gedekt.

`sessions` vermeldt recente vastlegsessies (`--limit`, standaard 20).

`query --preset <name>` voert een ingebouwde query uit op vastgelegd verkeer, optioneel beperkt tot `--session <id>`. Voorinstellingen:

- `double-sends`
- `retry-storms`
- `cache-busting`
- `ws-duplicate-frames`
- `missing-ack`
- `error-bursts`

`blob --id <blobId>` drukt de onbewerkte inhoud van een vastgelegde payloadblob af.

`purge` verwijdert alle metadata en blobs van vastgelegd verkeer. Vastleggingen zijn lokale debuggegevens; wis ze wanneer je klaar bent.

## Gerelateerd

- [CLI-referentie](/nl/cli)
- [Netwerkproxy](/nl/security/network-proxy)
- [Verificatie via vertrouwde proxy](/nl/gateway/trusted-proxy-auth)
