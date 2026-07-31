---
read_when:
    - Onboarding- of configuratiestromen testen met een lokaal verpakte Plugin
    - Een pluginpakket verifiëren voordat je het publiceert
    - Een automatische Plugin-installatie vervangen door een testartefact
sidebarTitle: Install overrides
summary: Test overschrijvingen van verpakte plugins met installatiestromen tijdens de configuratie
title: Overschrijvingen voor Plugin-installatie
x-i18n:
    generated_at: "2026-07-27T05:40:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: adc823f49ea9f8fa86e6a89933e43fdc309d808ac24397770495dbe81cb4b0d7
    source_path: plugins/install-overrides.md
    workflow: 16
---

Overrides voor Plugin-installatie stellen beheerders in staat om installaties van plugins tijdens de configuratie naar een
specifiek npm-pakket of lokaal npm-pack-tarball te verwijzen in plaats van naar de catalogus,
gebundelde of standaard npm-bron. Ze bestaan uitsluitend voor E2E- en pakketvalidatie;
normale gebruikers installeren plugins met
[`openclaw plugins install`](/nl/cli/plugins).

<Warning>
Overrides voeren Plugin-code uit vanuit de bron die je opgeeft. Gebruik ze alleen in een
geïsoleerde statusmap of op een tijdelijke testmachine.
</Warning>

## Omgeving

Overrides zijn uitgeschakeld tenzij beide variabelen zijn ingesteld:

```bash
export OPENCLAW_ALLOW_PLUGIN_INSTALL_OVERRIDES=1
export OPENCLAW_PLUGIN_INSTALL_OVERRIDES='{
  "codex": "npm-pack:/tmp/openclaw-codex-2026.5.8.tgz",
  "openclaw-web-search": "npm:@openclaw/web-search@2026.5.8"
}'
```

De overridemap is JSON met de Plugin-id als sleutel. Waarden ondersteunen:

| Voorvoegsel                | Bron                                                                                           |
| --------------------- | ------------------------------------------------------------------------------------------------ |
| `npm:<registry-spec>` | Registerpakketten, exacte versies of tags                                                       |
| `npm-pack:<path.tgz>` | Lokale tarballs geproduceerd door `npm pack`; relatieve paden worden opgelost vanuit de huidige werkmap |

## Gedrag

Wanneer een configuratieproces een Plugin installeert waarvan de id in de map voorkomt, gebruikt OpenClaw
de overridebron in plaats van de catalogus, gebundelde of standaard
npm-bron. Dit geldt voor onboarding en elk ander proces dat het gedeelde
installatieprogramma voor plugins tijdens de configuratie gebruikt.

- Overrides blijven de verwachte Plugin-id afdwingen: een tarball die aan `codex`
  is gekoppeld, moet een Plugin installeren waarvan de manifest-id `codex` is.
- Overrides nemen de officiële status van vertrouwde bron niet over. Zelfs wanneer het
  catalogusitem normaal gesproken een pakket van OpenClaw vertegenwoordigt, wordt een override
  behandeld als door de beheerder verstrekte testinvoer.
- Werkruimtebestanden `.env` kunnen installatie-overrides niet inschakelen; beide omgevingsvariabelen staan op
  de lijst met geblokkeerde dotenv-variabelen voor de werkruimte. Stel ze in via de vertrouwde shell, CI-taak of
  externe testopdracht waarmee OpenClaw wordt gestart.

## Pakket-E2E

Gebruik een geïsoleerde statusmap, zodat pakketinstallaties en installatierecords
je normale OpenClaw-status niet beïnvloeden:

```bash
npm pack extensions/codex --pack-destination /tmp

OPENCLAW_STATE_DIR="$(mktemp -d)" \
OPENCLAW_ALLOW_PLUGIN_INSTALL_OVERRIDES=1 \
OPENCLAW_PLUGIN_INSTALL_OVERRIDES='{"codex":"npm-pack:/tmp/openclaw-codex-2026.5.8.tgz"}' \
pnpm openclaw onboard --mode local
```

Controleer het geïnstalleerde pakket onder de statusmap:

```bash
find "$OPENCLAW_STATE_DIR/npm/projects" -path '*/node_modules/@openclaw/codex/package.json' -print
grep -R '"@openclaw/codex"' "$OPENCLAW_STATE_DIR/npm/projects"/*/package-lock.json
```

Haal voor live provider-E2E de echte API-sleutel uit een vertrouwde shell of een
CI-secret voordat je de testopdracht start. Druk sleutels niet af; rapporteer alleen de
bron en of de sleutel aanwezig was.
