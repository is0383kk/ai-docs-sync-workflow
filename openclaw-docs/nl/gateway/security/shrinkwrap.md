---
read_when:
    - Je wilt weten wat npm shrinkwrap betekent in een OpenClaw-release
    - Je beoordeelt lockfiles van pakketten, wijzigingen in afhankelijkheden of risico's voor de toeleveringsketen
    - Je valideert root- of plugin-npm-pakketten vóór publicatie
summary: Eenvoudige en technische uitleg van npm shrinkwrap in OpenClaw-releases
title: npm shrinkwrap
x-i18n:
    generated_at: "2026-07-27T05:48:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d1e6c0d4541da9220d50cde0b9db064e5a91b81d6562cb16ac697de7d4017098
    source_path: gateway/security/shrinkwrap.md
    workflow: 16
---

OpenClaw-broncodecheck-outs gebruiken `pnpm-lock.yaml`. Gepubliceerde OpenClaw-npm-pakketten gebruiken `npm-shrinkwrap.json`, npm's publiceerbare vergrendelingsbestand voor afhankelijkheden, zodat pakketinstallaties de tijdens de release beoordeelde afhankelijkheidsgraaf gebruiken.

## Waarom dit belangrijk is

Shrinkwrap is een ontvangstbewijs voor de afhankelijkheidsboom die met een npm-pakket wordt geleverd: het vertelt npm welke exacte transitieve versies moeten worden geïnstalleerd.

| Bestand               | Waar het van belang is        | Wat het betekent                            |
| --------------------- | ----------------------------- | ------------------------------------------- |
| `pnpm-lock.yaml`    | OpenClaw-broncodecheck-out    | Afhankelijkheidsgraaf voor maintainers      |
| `npm-shrinkwrap.json`    | Gepubliceerd npm-pakket       | npm-installatiegraaf voor gebruikers        |
| `package-lock.json`    | Lokale npm-apps               | Niet het publicatiecontract van OpenClaw    |

Voor OpenClaw-releases betekent dit:

- het gepubliceerde pakket vraagt npm niet om tijdens de installatie een nieuwe afhankelijkheidsgraaf te bedenken;
- wijzigingen in afhankelijkheden zijn controleerbaar omdat ze in een diff van het vergrendelingsbestand terechtkomen;
- releasevalidatie test dezelfde graaf die gebruikers zullen installeren;
- verrassingen met pakketgrootte of native afhankelijkheden komen vóór publicatie aan het licht.

Shrinkwrap is geen sandbox. Het maakt een afhankelijkheid op zichzelf niet veilig en vervangt geen hostisolatie, `openclaw security audit`, pakketprovenance of installatierooktests.

OpenClaw is een Gateway, Plugin-host, modelrouter en agentruntime, dus een standaardinstallatie beïnvloedt de opstarttijd, het schijfgebruik, downloads van native pakketten en de blootstelling aan risico's in de toeleveringsketen. Shrinkwrap geeft releasebeoordeling een stabiele grens: reviewers zien bewegingen in transitieve afhankelijkheden, validators wijzen onverwachte afwijkingen in het vergrendelingsbestand af en Plugin-pakketten bevatten hun eigen vergrendelde afhankelijkheidsgraaf in plaats van op het hoofdpakket te vertrouwen.

## Genereren en controleren

Het npm-hoofdpakket `openclaw`, npm-Plugin-pakketten van OpenClaw (bijvoorbeeld `@openclaw/discord`) en publiceerbare workspace-pakketten zoals [`@openclaw/ai`](/nl/reference/openclaw-ai) bevatten bij publicatie `npm-shrinkwrap.json`. Workspace-afhankelijkheden worden uit de hoofd-shrinkwrap weggelaten omdat ze naast het hoofdpakket worden gepubliceerd; elk publiceerbaar workspace-pakket legt in plaats daarvan zijn eigen transitieve boom vast. Geschikte Plugin-pakketten kunnen ook worden gepubliceerd met expliciete `bundledDependencies`, waarbij hun runtime-afhankelijkheidsbestanden in de Plugin-tarball worden opgenomen in plaats van uitsluitend op resolutie tijdens de installatie te vertrouwen.

```bash
# Alle door shrinkwrap beheerde pakketten (hoofd + publiceerbare Plugins)
pnpm deps:shrinkwrap:generate
pnpm deps:shrinkwrap:check

# Alleen het hoofdpakket
pnpm deps:shrinkwrap:root:generate
pnpm deps:shrinkwrap:root:check

# Alleen pakketten waarop de huidige wijzigingenset van invloed is
pnpm deps:shrinkwrap:changed:generate
pnpm deps:shrinkwrap:changed:check
```

De generator zet npm's publiceerbare vergrendelingsindeling om, maar wijst gegenereerde pakketversies af die nog niet in `pnpm-lock.yaml` aanwezig zijn. Zo blijft de beoordelingsgrens voor de ouderdom van pnpm-afhankelijkheden, overrides en patches intact.

Behandel het volgende als beveiligingsgevoelig:

- `pnpm-lock.yaml`
- `npm-shrinkwrap.json`
- afhankelijkheidspayloads van gebundelde Plugins
- elke diff van `package-lock.json`

OpenClaw-pakketvalidators vereisen shrinkwrap in nieuwe tarballs van het hoofdpakket en wijzen `package-lock.json` af voor gepubliceerde pakketten. Het npm-publicatiepad voor Plugins controleert de Plugin-lokale shrinkwrap, installeert pakketlokale gebundelde afhankelijkheden en maakt of publiceert vervolgens het pakket.

## Een gepubliceerd pakket inspecteren

Hoofdpakket:

```bash
npm pack openclaw@<version> --json --pack-destination /tmp/openclaw-pack
tar -tf /tmp/openclaw-pack/openclaw-<version>.tgz | grep '^package/npm-shrinkwrap.json$'
```

Plugin-pakket:

```bash
npm pack @openclaw/discord@<version> --json --pack-destination /tmp/openclaw-plugin-pack
tar -tf /tmp/openclaw-plugin-pack/openclaw-discord-<version>.tgz | grep '^package/npm-shrinkwrap.json$'
tar -tf /tmp/openclaw-plugin-pack/openclaw-discord-<version>.tgz | grep '^package/node_modules/'
```

Achtergrond: [npm-shrinkwrap.json](https://docs.npmjs.com/cli/v11/configuring-npm/npm-shrinkwrap-json).
