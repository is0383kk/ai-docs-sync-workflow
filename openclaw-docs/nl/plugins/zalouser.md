---
read_when:
    - Je wilt ondersteuning voor Zalo Personal (onofficieel) in OpenClaw
    - Je configureert of ontwikkelt de zalouser-plugin
summary: 'Zalo Personal-plugin: inloggen via QR-code + berichten via native zca-js (Plugin-installatie + kanaalconfiguratie + tool)'
title: Zalo-persoonlijke plugin
x-i18n:
    generated_at: "2026-07-27T06:30:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cb0bdaa10340b5d78dc32abf6b0520fda6cf5f65e2e17b551b4e9bd72acfbbf2
    source_path: plugins/zalouser.md
    workflow: 16
---

Zalo Personal-ondersteuning voor OpenClaw via een Plugin die de ingebouwde `zca-js` gebruikt om
een normaal Zalo-gebruikersaccount te automatiseren. Er is geen extern `zca`/`openzca`-CLI-programma
vereist.

<Warning>
Niet-officiële automatisering kan leiden tot opschorting of blokkering van het account. Gebruik op eigen risico.
</Warning>

## Naamgeving

De kanaal-id is `zalouser` om duidelijk te maken dat hiermee een **persoonlijk Zalo-
gebruikersaccount** wordt geautomatiseerd (niet-officieel). De afzonderlijke kanaal-id `zalo` is de officiële,
meegeleverde Zalo Bot/Webhook-integratie — zie [Zalo](/nl/channels/zalo).

## Waar het wordt uitgevoerd

Deze Plugin wordt **binnen het Gateway-proces** uitgevoerd. Installeer/configureer de Plugin bij een externe Gateway
op die host en start vervolgens de Gateway opnieuw.

## Installeren

### Vanuit npm

```bash
openclaw plugins install @openclaw/zalouser
```

Gebruik alleen het pakket om de huidige officiële releasetag te volgen; leg alleen een exacte
versie vast wanneer je een reproduceerbare installatie nodig hebt. Start de Gateway
daarna opnieuw.

### Vanuit een lokale map (ontwikkeling)

```bash
PLUGIN_SRC=./path/to/local/zalouser-plugin
openclaw plugins install "$PLUGIN_SRC"
cd "$PLUGIN_SRC" && pnpm install
```

Start de Gateway daarna opnieuw.

## Configuratie

De kanaalconfiguratie staat onder `channels.zalouser` (niet `plugins.entries.*`):

```json5
{
  channels: {
    zalouser: {
      enabled: true,
      dmPolicy: "pairing",
    },
  },
}
```

Zie [Configuratie van het persoonlijke Zalo-kanaal](/nl/channels/zalouser) voor toegangsbeheer van privéberichten/groepen,
configuratie met meerdere accounts, omgevingsvariabelen en probleemoplossing.

## CLI

```bash
openclaw channels login --channel zalouser
openclaw channels login --channel zalouser --account <name>
openclaw channels logout --channel zalouser
openclaw channels status --probe
openclaw message send --channel zalouser --target <threadId> --message "Hallo van OpenClaw"
openclaw directory self --channel zalouser
openclaw directory peers list --channel zalouser --query "naam"
openclaw directory groups list --channel zalouser --query "naam"
openclaw directory groups members --channel zalouser --group-id <id>
```

## Agenttool

Toolnaam: `zalouser`

Acties: `send`, `image`, `link`, `friends`, `groups`, `me`, `status`

Kanaalberichtacties (niet de agenttool) ondersteunen ook `react` voor
reacties op berichten.

## Gerelateerd

- [Configuratie van het persoonlijke Zalo-kanaal](/nl/channels/zalouser)
- [Zalo (officieel Bot-/Webhook-kanaal)](/nl/channels/zalo)
- [Plugins bouwen](/nl/plugins/building-plugins)
- [ClawHub](/clawhub)
