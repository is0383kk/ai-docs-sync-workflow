---
read_when:
    - Begrijpen wat er gebeurt wanneer de agent voor het eerst wordt uitgevoerd
    - Uitleg over waar bootstrapbestanden zich bevinden
    - Foutopsporing bij het instellen van de identiteit tijdens de onboarding
sidebarTitle: Bootstrapping
summary: Bootstrapritueel voor de agent dat de werkruimte- en identiteitsbestanden initialiseert
title: Agentinitialisatie
x-i18n:
    generated_at: "2026-07-27T06:13:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: efb47e1a6a86d68aef1aa1662fe9c5def9a4e5b45649b84aeb9060bfcba21a5d
    source_path: start/bootstrapping.md
    workflow: 16
---

Bootstrapping is het ritual voor de eerste uitvoering dat een nieuwe agentwerkruimte initialiseert en
de agent begeleidt bij het kiezen van een identiteit. Het wordt één keer uitgevoerd, direct na
de onboarding, tijdens de eerste echte beurt van de agent.

## Wat er gebeurt

Bij de eerste uitvoering met een volledig nieuwe werkruimte (standaard `~/.openclaw/workspace`),
voert OpenClaw het volgende uit:

- Initialiseert `AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`, `HEARTBEAT.md` en `BOOTSTRAP.md`.
- Laat de agent een begrensde geboortesequentie van drie stappen volgen: de agent vraagt hoe je
  hem wilt noemen, deelt één korte regel over zijn ziel/sfeer en vraagt of je de
  minimaal aanbevolen set plugins of maximaal gemak wilt.
- Slaat de afgesproken identiteit tweemaal permanent op: in `IDENTITY.md` en `SOUL.md` (wat de
  agent over zichzelf leest) en via `openclaw agents set-identity` (wat kanalen
  en de UI weergeven).
- Leest app-aanbevelingen die tijdens de onboarding al zijn opgeslagen, zonder opnieuw te scannen.
  Officiële plugins gebruiken `openclaw plugins install <id>`; Skills van derden uit ClawHub
  blijven expliciete opt-ins. Nadat de keuze is verwerkt, bevestigt de agent
  het opgeslagen aanbod, zodat hij er nooit meer naar vraagt.
- Verwijdert `BOOTSTRAP.md` zodra de werkruimte geconfigureerd lijkt, zodat het ritual slechts één keer wordt uitgevoerd.

Een werkruimte geldt als geconfigureerd zodra `SOUL.md`, `IDENTITY.md` of `USER.md`
afwijkt van de beginsjabloon, of als er een map `memory/` bestaat.

<Note>
`BOOTSTRAP.md` omvat het volledige identiteitsgesprek. Bekijk de inhoud in
de [BOOTSTRAP.md-sjabloon](/nl/reference/templates/BOOTSTRAP).
</Note>

## Uitvoeringen met ingesloten en lokale modellen

Voor uitvoeringen met ingesloten of lokale modellen houdt OpenClaw `BOOTSTRAP.md` buiten de
bevoorrechte systeemcontext. Bij de primaire interactieve eerste uitvoering
wordt de bestandsinhoud nog steeds via de gebruikersprompt doorgegeven, zodat modellen die niet
betrouwbaar de tool `read` aanroepen het ritual toch kunnen voltooien. Als de huidige
uitvoering geen veilige toegang tot de werkruimte heeft, krijgt de agent een korte, beperkte bootstrap-
melding in plaats van een algemene begroeting.

## Bootstrapping overslaan

Voer het volgende uit om dit in een vooraf geïnitialiseerde werkruimte over te slaan:

```bash
openclaw onboard --skip-bootstrap
```

## Waar het wordt uitgevoerd

Bootstrapping wordt altijd op de Gateway-host uitgevoerd. Als de macOS-app verbinding maakt met een
externe Gateway, bevinden de werkruimte en de bijbehorende bootstrapbestanden zich op die externe
machine, niet op de Mac.

<Note>
Als de Gateway op een andere machine draait, bewerk je de werkruimtebestanden op de Gateway-
host (bijvoorbeeld `user@gateway-host:~/.openclaw/workspace`).
</Note>

## Gerelateerde documentatie

- Onboarding van de macOS-app: [Onboarding](/nl/start/onboarding)
- Indeling van de werkruimte: [Agentwerkruimte](/nl/concepts/agent-workspace)
- Inhoud van de sjabloon: [BOOTSTRAP.md-sjabloon](/nl/reference/templates/BOOTSTRAP)
