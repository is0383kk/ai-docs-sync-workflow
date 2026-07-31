---
read_when:
    - Je wilt cloudbeheerde sandboxes in plaats van lokale Docker-containers
    - Je stelt de OpenShell-plugin in
    - Je moet kiezen tussen de modi voor een gespiegelde en een externe werkruimte
summary: Gebruik OpenShell als beheerde sandboxbackend voor OpenClaw-agents
title: OpenShell
x-i18n:
    generated_at: "2026-07-27T05:00:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: bf5c33912bd0db759a01cf58ea26712a8ada68c0804bf16f69f1f7cdd496828c
    source_path: gateway/openshell.md
    workflow: 16
---

OpenShell is een beheerde sandboxbackend: in plaats van Docker-containers
lokaal uit te voeren, delegeert OpenClaw de sandboxlevenscyclus aan de `openshell`-CLI, die
externe omgevingen inricht en opdrachten via SSH uitvoert.

De Plugin hergebruikt hetzelfde SSH-transport en dezelfde externe bestandssysteembridge als de
algemene [SSH-backend](/nl/gateway/sandboxing#ssh-backend), en voegt de OpenShell-
levenscyclus (`sandbox create/get/delete/ssh-config`) plus een optionele `mirror`-
modus voor werkruimtesynchronisatie toe.

## Vereisten

- OpenShell-Plugin geïnstalleerd (`openclaw plugins install @openclaw/openshell-sandbox`)
- `openshell`-CLI op `PATH` (of een aangepast pad via
  `plugins.entries.openshell.config.command`)
- Een OpenShell-account met sandboxtoegang
- OpenClaw Gateway actief op de host

## Snel aan de slag

```bash
openclaw plugins install @openclaw/openshell-sandbox
```

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "openshell",
        scope: "session",
        workspaceAccess: "rw",
      },
    },
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "remote",
        },
      },
    },
  },
}
```

Start de Gateway opnieuw. Bij de volgende agentbeurt maakt OpenClaw een OpenShell-
sandbox en leidt het de uitvoering van tools erdoorheen. Controleer dit met:

```bash
openclaw sandbox list
openclaw sandbox explain
```

## Werkruimtemodi

Dit is de belangrijkste keuze voor OpenShell.

### mirror (standaard)

`plugins.entries.openshell.config.mode: "mirror"` houdt de **lokale werkruimte
canoniek**:

- Vóór `exec` synchroniseert OpenClaw de lokale werkruimte naar de sandbox.
- Na `exec` synchroniseert OpenClaw de externe werkruimte terug naar lokaal.
- Bestandstools lopen via de sandboxbridge, maar lokaal blijft tussen
  beurten de bron van waarheid.

Het meest geschikt voor ontwikkelworkflows: lokale wijzigingen buiten OpenClaw verschijnen bij de
volgende uitvoering en de sandbox gedraagt zich vrijwel hetzelfde als de Docker-backend.

Nadeel: bij elke uitvoeringsbeurt zijn er upload- en downloadkosten.

### remote

`mode: "remote"` maakt de **OpenShell-werkruimte canoniek**:

- Wanneer de sandbox voor het eerst wordt gemaakt, vult OpenClaw de externe werkruimte
  één keer vanuit de lokale werkruimte.
- Daarna werken `exec`, `read`, `write`, `edit` en `apply_patch`
  rechtstreeks op de externe werkruimte. OpenClaw synchroniseert externe wijzigingen
  **niet** terug naar lokaal.
- Het lezen van media tijdens het opstellen van prompts blijft werken (bestands-/mediatools lezen via de
  sandboxbridge).

Het meest geschikt voor langlopende agents en CI: minder overhead per beurt en lokale
wijzigingen op de host kunnen de externe status niet ongemerkt overschrijven.

<Warning>
Bestandswijzigingen op de host buiten OpenClaw zijn na de initiële vulling niet zichtbaar voor de externe sandbox. Voer `openclaw sandbox recreate` uit om deze opnieuw te vullen.
</Warning>

### Een modus kiezen

|                          | `mirror`                   | `remote`                  |
| ------------------------ | -------------------------- | ------------------------- |
| **Canonieke werkruimte** | Lokale host                | Externe OpenShell         |
| **Synchronisatierichting** | Bidirectioneel (elke uitvoering) | Eenmalige vulling    |
| **Overhead per beurt**   | Hoger (upload + download)  | Lager (rechtstreekse externe bewerkingen) |
| **Lokale wijzigingen zichtbaar?** | Ja, bij de volgende uitvoering | Nee, tot opnieuw aanmaken |
| **Het meest geschikt voor** | Ontwikkelworkflows      | Langlopende agents, CI    |

## Configuratiereferentie

Alle OpenShell-configuratie bevindt zich onder `plugins.entries.openshell.config`:

| Sleutel                   | Type                     | Standaard      | Beschrijving                                                                           |
| ------------------------- | ------------------------ | -------------- | -------------------------------------------------------------------------------------- |
| `mode`                    | `"mirror"` of `"remote"` | `"mirror"`    | Modus voor werkruimtesynchronisatie                                                     |
| `command`                 | `string`                 | `"openshell"` | Pad naar of naam van de `openshell`-CLI                                               |
| `from`                    | `string`                 | `"openclaw"`  | Sandboxbron voor de eerste aanmaak                                                      |
| `gateway`                 | `string`                 | niet ingesteld | Naam van de OpenShell-gateway (`--gateway` op het hoogste niveau)                |
| `gatewayEndpoint`         | `string`                 | niet ingesteld | Eindpunt van de OpenShell-gateway (`--gateway-endpoint` op het hoogste niveau)            |
| `policy`                  | `string`                 | niet ingesteld | OpenShell-beleids-ID voor het aanmaken van de sandbox                                   |
| `providers`               | `string[]`               | `[]`          | Providernamen die bij het aanmaken van de sandbox worden gekoppeld (ontdubbeld, één `--provider`-vlag per item) |
| `gpu`                     | `boolean`                | `false`       | GPU-resources aanvragen (`--gpu`)                                           |
| `autoProviders`           | `boolean`                | `true`        | `--auto-providers` doorgeven (of `--no-auto-providers` wanneer onwaar) tijdens het aanmaken |
| `remoteWorkspaceDir`      | `string`                 | `"/sandbox"`  | Primaire beschrijfbare werkruimte in de sandbox                                         |
| `remoteAgentWorkspaceDir` | `string`                 | `"/agent"`    | Koppelpad voor de agentwerkruimte (alleen-lezen wanneer werkruimtetoegang niet `rw` is) |
| `timeoutSeconds`          | `number`                 | `120`         | Time-out voor bewerkingen van de `openshell`-CLI                                |

`remoteWorkspaceDir` en `remoteAgentWorkspaceDir` moeten absolute paden zijn en
binnen de beheerde hoofdpaden `/sandbox` of `/agent` blijven; andere absolute paden worden
geweigerd.

Instellingen op sandboxniveau (`mode`, `scope`, `workspaceAccess`) bevinden zich onder
`agents.defaults.sandbox`, net als bij elke backend. Zie
[Sandboxing](/nl/gateway/sandboxing) voor de volledige matrix.

## Voorbeelden

### Minimale externe configuratie

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "openshell",
      },
    },
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "remote",
        },
      },
    },
  },
}
```

### Spiegelmodus met GPU

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "openshell",
        scope: "agent",
        workspaceAccess: "rw",
      },
    },
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "mirror",
          gpu: true,
          providers: ["openai"],
          timeoutSeconds: 180,
        },
      },
    },
  },
}
```

### OpenShell per agent met aangepaste gateway

```json5
{
  agents: {
    defaults: {
      sandbox: { mode: "off" },
    },
    list: [
      {
        id: "researcher",
        sandbox: {
          mode: "all",
          backend: "openshell",
          scope: "agent",
          workspaceAccess: "rw",
        },
      },
    ],
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "remote",
          gateway: "lab",
          gatewayEndpoint: "https://lab.example",
          policy: "strict",
        },
      },
    },
  },
}
```

## Levenscyclusbeheer

```bash
# Alle sandboxruntimes weergeven (Docker + OpenShell)
openclaw sandbox list

# Effectief beleid inspecteren
openclaw sandbox explain

# Opnieuw aanmaken (verwijdert externe werkruimte, wordt bij volgend gebruik opnieuw gevuld)
openclaw sandbox recreate --all
```

Voor de modus `remote` is opnieuw aanmaken bijzonder belangrijk: hierdoor wordt de canonieke
externe werkruimte voor dat bereik verwijderd en bij het volgende gebruik wordt een nieuwe werkruimte vanuit
lokaal gevuld. Voor de modus `mirror` stelt opnieuw aanmaken voornamelijk de externe uitvoeringsomgeving
opnieuw in, omdat lokaal canoniek blijft.

Maak opnieuw aan nadat je een van de volgende zaken hebt gewijzigd:

- `agents.defaults.sandbox.backend`
- `plugins.entries.openshell.config.from`
- `plugins.entries.openshell.config.mode`
- `plugins.entries.openshell.config.policy`

## Beveiligingsversterking

De bestandssysteembridge van de spiegelmodus zet de hoofdmap van de lokale werkruimte vast en controleert
canonieke paden (via realpath) opnieuw vóór elke lees-, schrijf-, mkdir-, verwijder- en
hernoembewerking, waarbij symbolische koppelingen midden in het pad worden geweigerd. Een verwisselde symbolische koppeling of opnieuw gekoppelde werkruimte
kan bestandstoegang niet omleiden naar buiten de gespiegelde structuur.

## Huidige beperkingen

- De sandboxbrowser wordt niet ondersteund door de OpenShell-backend.
- `sandbox.docker.binds` is niet van toepassing op OpenShell; het aanmaken van de sandbox mislukt
  als koppelingen zijn geconfigureerd.
- Docker-specifieke runtimeopties onder `sandbox.docker.*` (behalve `env`)
  zijn alleen van toepassing op de Docker-backend.

## Hoe het werkt

1. OpenClaw voert `sandbox get` uit voor de sandboxnaam (met eventueel geconfigureerde
   `--gateway`/`--gateway-endpoint`); als dat mislukt, maakt het er een aan met
   `sandbox create`, waarbij `--name`, `--from`, `--policy` indien ingesteld, `--gpu`
   indien ingeschakeld, `--auto-providers`/`--no-auto-providers` en één
   `--provider`-vlag per geconfigureerde provider worden doorgegeven.
2. OpenClaw voert `sandbox ssh-config` uit voor de sandboxnaam om de SSH-
   verbindingsgegevens op te halen.
3. De kern schrijft de SSH-configuratie naar een tijdelijk bestand en opent een SSH-sessie via
   dezelfde externe bestandssysteembridge als de algemene SSH-backend.
4. In de modus `mirror`: synchroniseer lokaal naar extern vóór uitvoering, voer uit en synchroniseer daarna terug.
5. In de modus `remote`: vul één keer bij het aanmaken en werk daarna rechtstreeks in de externe
   werkruimte.

## Gerelateerd

- [Sandboxing](/nl/gateway/sandboxing) - modi, bereiken en vergelijking van backends
- [Sandbox versus toolbeleid versus verhoogde bevoegdheden](/nl/gateway/sandbox-vs-tool-policy-vs-elevated) - geblokkeerde tools opsporen
- [Sandbox en tools voor meerdere agents](/nl/tools/multi-agent-sandbox-tools) - overschrijvingen per agent
- [Sandbox-CLI](/nl/cli/sandbox) - `openclaw sandbox`-opdrachten
