---
read_when:
    - Je wilt de OpenClaw-instellingen controleren aan de hand van een opgestelde policy.jsonc
    - Je wilt beleidsbevindingen in doctor lint
    - Je hebt een hash voor beleidsattestatie nodig als auditbewijs
summary: CLI-referentie voor `openclaw policy`-conformiteitscontroles
title: Beleid
x-i18n:
    generated_at: "2026-07-27T04:55:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 63e4faeab8dd6535e3d517439d3f58cdc167b6b7fade808a6482742ec9b5acf1
    source_path: cli/policy.md
    workflow: 16
---

# `openclaw policy`

`openclaw policy` wordt geleverd door de gebundelde Policy-plugin. Het is een bedrijfsbrede
conformiteitslaag boven op bestaande OpenClaw-instellingen, geen tweede configuratie-
systeem. Je stelt vereisten op in `policy.jsonc`; OpenClaw observeert de actieve
werkruimte als bewijs; Policy rapporteert afwijkingen via `doctor --lint`. Policy
dwingt geen toolaanroepen af en herschrijft het runtimegedrag niet tijdens een aanvraag,
en attesteert geen referentieopslagplaatsen per agent, zoals `auth-profiles.json`.

Policy controleert geconfigureerde kanalen, MCP-servers, modelproviders, de SSRF-
beveiliging van het netwerk, toegang via inkomend verkeer/kanalen, blootstelling van de Gateway en de
houding voor Node-opdrachten, opgestelde routeringsprobes voor berichten,
toegang tot agentwerkruimten, sandboxhouding, gegevensverwerkingshouding, de houding
van geheime providers/authenticatieprofielen en metadata van beheerde tools (`TOOLS.md`). Gebruik het
wanneer een werkruimte een duurzame, controleerbare verklaring nodig heeft, zoals 'Telegram mag
niet zijn ingeschakeld' of 'beheerde tools moeten metadata voor risico en eigenaar declareren'. Als
je alleen lokaal gedrag nodig hebt zonder attestatie of afwijkingsdetectie, volstaat een gewone
configuratie.

## Snel aan de slag

```bash
openclaw plugins enable policy
```

De Plugin blijft ingeschakeld, zelfs wanneer `policy.jsonc` ontbreekt, zodat doctor
het ontbrekende artefact kan rapporteren in plaats van controles stilzwijgend over te slaan.

Stel `policy.jsonc` handmatig op; het wordt niet gegenereerd uit de huidige instellingen. Elke
sectie op het hoogste niveau is een regelnaamruimte: een controle wordt alleen uitgevoerd wanneer er
een concrete regel onder staat (niet-ondersteunde secties of sleutels mislukken als
`policy/policy-jsonc-invalid` in plaats van stilzwijgend te worden genegeerd). Minimaal
voorbeeld dat elke ondersteunde sectie omvat:

```jsonc
{
  "channels": {
    "denyRules": [
      {
        "id": "no-telegram",
        "when": { "provider": "telegram" },
        "reason": "Telegram is niet goedgekeurd voor deze werkruimte.",
      },
    ],
  },
  "mcp": {
    "servers": {
      "allow": ["docs"],
      "deny": ["untrusted"],
    },
  },
  "models": {
    "providers": {
      "allow": ["openai", "anthropic"],
      "deny": ["openrouter"],
    },
  },
  "network": {
    "privateNetwork": {
      "allow": false,
    },
  },
  "routing": {
    "requireBindings": true,
    "requireConfiguredChannels": true,
    "probes": [
      {
        "id": "family-dm",
        "route": {
          "channel": "imessage",
          "peer": { "kind": "direct", "id": "+15555550123" },
        },
        "expect": {
          "agentId": "family",
          "matchedBy": ["binding.peer"],
        },
      },
    ],
  },
  "ingress": {
    "session": {
      "requireDmScope": "per-channel-peer",
    },
    "channels": {
      "allowDmPolicies": ["pairing", "allowlist", "disabled"],
      "denyOpenGroups": true,
      "requireMentionInGroups": true,
    },
  },
  "gateway": {
    "exposure": {
      "allowNonLoopbackBind": false,
      "allowTailscaleFunnel": false,
    },
    "auth": {
      "requireAuth": true,
      "requireExplicitRateLimit": true,
    },
    "controlUi": {
      "allowInsecure": false,
    },
    "remote": {
      "allow": false,
    },
    "http": {
      "denyEndpoints": ["chatCompletions", "responses"],
      "requireUrlAllowlists": true,
    },
    "nodes": {
      "denyCommands": ["system.run"],
    },
  },
  "agents": {
    "workspace": {
      "allowedAccess": ["none", "ro"],
      "denyTools": ["exec", "process", "write", "edit", "apply_patch"],
    },
  },
  "dataHandling": {
    "sensitiveLogging": {
      "requireRedaction": true,
    },
    "telemetry": {
      "denyContentCapture": true,
    },
    "retention": {
      "requireSessionMaintenance": true,
    },
    "memory": {
      "denySessionTranscriptIndexing": true,
    },
  },
  "secrets": {
    "requireManagedProviders": true,
    "denySources": ["exec"],
    "allowInsecureProviders": false,
  },
  "auth": {
    "profiles": {
      "requireMetadata": ["provider", "mode"],
      "allowModes": ["api_key", "token"],
    },
  },
  "execApprovals": {
    "requireFile": true,
    "defaults": { "allowSecurity": ["deny"] },
    "agents": {
      "allowSecurity": ["deny", "allowlist"],
      "allowAutoAllowSkills": false,
      "allowlist": { "expected": ["deploy", "status"] },
    },
  },
  "tools": {
    "requireMetadata": ["risk", "sensitivity", "owner"],
    "profiles": {
      "allow": ["messaging", "minimal"],
    },
    "fs": {
      "requireWorkspaceOnly": true,
    },
    "exec": {
      "allowSecurity": ["deny", "allowlist"],
      "requireAsk": ["always"],
      "allowHosts": ["sandbox"],
    },
    "elevated": {
      "allow": false,
    },
    "denyTools": ["group:runtime", "group:fs"],
  },
}
```

Algemene opmerkingen die niet duidelijk blijken uit de onderstaande regeltabellen:

- Als je `gateway.bind` weglaat terwijl niet-loopbackbindingen worden geweigerd, betekent dit dat je
  de standaardwaarde van de runtime accepteert; stel `gateway.bind: "loopback"` in voor strikte conformiteit.
- Stel voor een alleen-lezenagent sandbox `mode` in op `all` of `non-main` bij de
  toepasselijke standaardwaarden/agent en `workspaceAccess` op `none` of `ro`. Een ontbrekende
  sandboxmodus of sandboxmodus `off` voldoet niet aan een alleen-lezenbeleid.
- `agents.workspace.denyTools` accepteert `exec`, `process`, `write`, `edit`,
  `apply_patch`. De groepen voor het weigeren van configuratietools `group:fs` (bestandswijziging) en
  `group:runtime` (shell/proces) voldoen aan de equivalente houding.
- Controles voor uitvoeringsgoedkeuringen lezen het actieve artefact `exec-approvals.json` alleen wanneer
  een regel `execApprovals` aanwezig is; een ontbrekend of ongeldig artefact is
  niet-observeerbaar bewijs, geen kunstmatig geslaagde controle.
- Bewijs voor geheimen en authenticatieprofielen registreert alleen de houding van providers/bronnen en
  SecretRef-metadata, nooit onbewerkte waarden. Policy leest of attesteert geen
  referentieopslagplaatsen per agent, zoals `auth-profiles.json`.
- Bewijs voor gegevensverwerking betreft alleen de houding op configuratieniveau (redactiemodus,
  schakeloptie voor telemetrieverzameling, modus voor sessieonderhoud, instelling voor
  transcriptindexering). Het inspecteert geen logboeken, telemetrie-exports, transcripten of
  geheugenbestanden, en een schoon resultaat bewijst niet dat deze geen persoonsgegevens of
  geheimen bevatten.
- Routeringsprobes gebruiken opnieuw de runtimebindingsresolver van OpenClaw. Routeringsbewijs
  registreert alleen de probe-id, opgeloste agent, het overeenkomsttype en geredigeerde bindings-
  metadata. Het registreert nooit identificatoren van peers, accounts, guilds, teams of rollen.
  Door een routeringssectie toe te voegen, veranderen de beleids- en attestatiehashes bewust;
  beleidsregels zonder routering behouden hun bestaande bewijsvorm.

### Naslaginformatie voor beleidsregels

Elke onderstaande regel is optioneel; een controle wordt alleen uitgevoerd wanneer de regel aanwezig is. De
waargenomen status bestaat uit bestaande OpenClaw-configuratie of werkruimtemetadata.

#### Overlays met bereik

Gebruik `scopes.<scopeName>` wanneer specifieke agents of kanalen een strenger beleid nodig hebben
dan de basislijn op het hoogste niveau. De bereiknaam is slechts een label; voor overeenkomsten wordt de
selector binnen het bereik gebruikt. Overlays zijn additief: de globale regel blijft actief
en de regel met bereik kan een eigen bevinding aan hetzelfde bewijs toevoegen.

| Selector     | Ondersteunde secties                                                             | Gebruiken wanneer                                          |
| ------------ | ------------------------------------------------------------------------------ | ------------------------------------------------- |
| `agentIds`   | `tools`, `agents.workspace`, `sandbox`, `dataHandling.memory`, `execApprovals` | Een of meer runtimeagents strengere regels nodig hebben.   |
| `channelIds` | `ingress.channels`                                                             | Een of meer kanalen strengere regels voor inkomend verkeer nodig hebben. |

Als een vermelding `agentIds` niet aanwezig is in `agents.entries.*`, evalueert OpenClaw
de regel met bereik aan de hand van de overgenomen globale/standaardhouding voor die runtime-
agent-id in plaats van deze over te slaan.

```jsonc
{
  "tools": {
    "exec": {
      "allowHosts": ["sandbox", "node"],
    },
  },
  "sandbox": {
    "requireMode": ["all", "non-main"],
  },
  "scopes": {
    "release-workspace": {
      "agentIds": ["release-agent", "review-agent"],
      "agents": {
        "workspace": {
          "allowedAccess": ["none", "ro"],
        },
      },
    },
    "release-lockdown": {
      "agentIds": ["release-agent"],
      "tools": {
        "exec": {
          "allowHosts": ["sandbox"],
          "allowSecurity": ["deny", "allowlist"],
          "requireAsk": ["always"],
        },
        "denyTools": ["exec", "process", "write", "edit", "apply_patch"],
      },
      "sandbox": {
        "requireMode": ["all"],
        "allowBackends": ["docker"],
      },
      "dataHandling": {
        "memory": {
          "denySessionTranscriptIndexing": true,
        },
      },
    },
    "shell-sandbox": {
      "agentIds": ["shell-agent"],
      "sandbox": {
        "allowBackends": ["openshell"],
        "containers": {
          "requireReadOnlyMounts": false,
        },
      },
    },
    "telegram-ingress": {
      "channelIds": ["telegram"],
      "ingress": {
        "channels": {
          "allowDmPolicies": ["pairing"],
          "denyOpenGroups": true,
          "requireMentionInGroups": true,
        },
      },
    },
  },
}
```

Dezelfde agent kan in meerdere bereiken voorkomen als elk bereik een ander
veld beheert, zoals hierboven. Een herhaald veld met bereik voor dezelfde agent moet even
streng of strenger zijn; een zwakkere dubbele claim wordt geweigerd (toestaanlijsten zijn
deelverzamelingen, weigerlijsten zijn bovenverzamelingen, vereiste booleaanse waarden liggen vast).

Regels voor containerhouding (`sandbox.containers.*`) worden alleen gecontroleerd aan de hand van
bewijs dat de sandboxbackend van de overeenkomende agent kan blootleggen. Als een backend
een regel die je ervoor hebt ingeschakeld niet kan observeren, rapporteert Policy
`policy/sandbox-container-posture-unobservable` in plaats van een geslaagde controle; beperk
containerregels tot de agentgroepen die een backend gebruiken die ze kan blootleggen.

`ingress.session.requireDmScope` op het hoogste niveau blijft globaal; `session.dmScope` is
geen bewijs dat aan een kanaal kan worden toegeschreven en kan daarom niet worden beperkt via `channelIds`.

Elk bereik dat aanwezig is in `policy.jsonc` moet geldig en afdwingbaar zijn.

#### Kanalen

| Beleidsveld                         | Waargenomen status                          | Gebruiken wanneer                                                     |
| ------------------------------------ | --------------------------------------- | ------------------------------------------------------------ |
| `channels.denyRules[].when.provider` | Provider en ingeschakelde status van `channels.*` | Geconfigureerde kanalen van een provider zoals `telegram` weigeren. |
| `channels.denyRules[].reason`        | Context van bevindingsbericht en hersteladvies | Uitleggen waarom de provider wordt geweigerd.                          |

#### MCP-servers

| Beleidsveld        | Waargenomen status      | Gebruiken wanneer                                                   |
| ------------------- | ------------------- | ---------------------------------------------------------- |
| `mcp.servers.allow` | Id's van `mcp.servers.*` | Vereisen dat elke geconfigureerde MCP-server in een toestaanlijst staat. |
| `mcp.servers.deny`  | Id's van `mcp.servers.*` | Specifieke geconfigureerde MCP-server-id's weigeren.                   |

#### Modelproviders

| Beleidsveld             | Waargenomen status                                   | Gebruiken wanneer                                                                        |
| ------------------------ | ------------------------------------------------ | ------------------------------------------------------------------------------- |
| `models.providers.allow` | Id's en geselecteerde modelreferenties van `models.providers.*` | Vereisen dat geconfigureerde providers en geselecteerde modelreferenties goedgekeurde providers gebruiken. |
| `models.providers.deny`  | Id's en geselecteerde modelreferenties van `models.providers.*` | Geconfigureerde providers en geselecteerde modelreferenties weigeren op provider-id.               |

#### Netwerk

| Beleidsveld                   | Waargenomen status                      | Gebruiken wanneer                                                           |
| ------------------------------ | ----------------------------------- | ------------------------------------------------------------------ |
| `network.privateNetwork.allow` | Uitwijkmogelijkheden voor SSRF naar privénetwerken | Stel in op `false` om te vereisen dat toegang tot privénetwerken uitgeschakeld blijft. |

#### Berichtroutering

| Beleidsveld                        | Waargenomen status                                      | Gebruiken wanneer                                                               |
| ----------------------------------- | --------------------------------------------------- | ---------------------------------------------------------------------- |
| `routing.requireBindings`           | Kanaalroutekoppelingen, exclusief ACP-koppelingen      | Vereis ten minste één koppeling voor berichtroutering.                          |
| `routing.requireConfiguredChannels` | Kanaal-id's van koppelingen en geconfigureerde `channels.*`-id's | Detecteer verouderde of verkeerd gespelde kanaal-id's van koppelingen.                        |
| `routing.probes[].route`            | De openbare OpenClaw-routeresolver                  | Beschrijf een representatieve inkomende route zonder een bericht te verzenden.     |
| `routing.probes[].expect.agentId`   | Opgelost agent-id                                   | Vereis dat de route de beoordeelde agent bereikt.                         |
| `routing.probes[].expect.matchedBy` | Overeenkomsttype van de resolver                                 | Vereis de beoordeelde koppelingsspecificiteit voor peer, account, kanaal of een ander type. |

Probe-id's moeten uniek zijn. Een route ondersteunt `channel`, optioneel `accountId`,
`peer`, `parentPeer`, `guildId`, `teamId` en `memberRoleIds`. Peertypen zijn
`direct`, `group` en `channel`. `matchedBy` kan een of meer runtime-
overeenkomsttypen bevatten, waaronder `binding.peer`, `binding.account`, `binding.channel`
of `default`.

Routeringscontroles zijn uitsluitend conformiteitscontroles. Ze wijzigen het opstarten,
de berichtbezorging, de prioriteit van koppelingen of het terugvalgedrag niet. Bevindingen vereisen
beoordeling door de beheerder, omdat het automatisch wijzigen van een koppeling
privéberichten kan omleiden.

#### Inkomend verkeer en kanaaltoegang

| Beleidsveld                              | Waargenomen status                                                 | Gebruiken wanneer                                                           |
| ----------------------------------------- | -------------------------------------------------------------- | ------------------------------------------------------------------ |
| `ingress.session.requireDmScope`          | `session.dmScope`                                              | Vereis een beoordeeld isolatiebereik voor directe berichten.                 |
| `ingress.channels.allowDmPolicies`        | `channels.*.dmPolicy` en verouderde kanaalbeleidsvelden voor directe berichten      | Sta alleen beoordeeld kanaalbeleid voor directe berichten toe.               |
| `ingress.channels.denyOpenGroups`         | Beleid voor inkomend verkeer van kanalen, accounts en groepen                     | Weiger open inkomend groepsverkeer voor geconfigureerde kanalen en accounts.      |
| `ingress.channels.requireMentionInGroups` | Configuratie van vermeldingcontroles voor kanalen, accounts, groepen, guilds en geneste vermeldingen | Vereis vermeldingcontroles wanneer inkomend groepsverkeer open is of door vermeldingen wordt beperkt. |

#### Gateway

| Beleidsveld                            | Waargenomen status                                 | Gebruiken wanneer                                                                             |
| --------------------------------------- | ---------------------------------------------- | ------------------------------------------------------------------------------------ |
| `gateway.exposure.allowNonLoopbackBind` | `gateway.bind`                                 | Stel in op `false` om een Gateway-koppeling aan loopback te vereisen.                                  |
| `gateway.exposure.allowTailscaleFunnel` | Gateway-configuratie voor Tailscale Serve/Funnel         | Stel in op `false` om blootstelling via Tailscale Funnel te weigeren.                                    |
| `gateway.auth.requireAuth`              | `gateway.auth.mode`                            | Stel in op `true` om uitgeschakelde Gateway-authenticatie te weigeren.                                       |
| `gateway.auth.requireExplicitRateLimit` | `gateway.auth.rateLimit`                       | Stel in op `true` om expliciete configuratie voor snelheidsbeperking van authenticatie te vereisen.                            |
| `gateway.controlUi.allowInsecure`       | Onveilige schakelaars voor authenticatie, apparaten en oorsprongen in de Control UI | Stel in op `false` om schakelaars voor onveilige blootstelling van de Control UI te weigeren.                         |
| `gateway.remote.allow`                  | Externe Gateway-modus/configuratie                     | Stel in op `false` om de externe Gateway-modus te weigeren.                                          |
| `gateway.http.denyEndpoints`            | HTTP-API-eindpunten van de Gateway                     | Weiger eindpunt-id's zoals `chatCompletions` of `responses`.                          |
| `gateway.http.requireUrlAllowlists`     | URL-ophaalinvoer van de HTTP-API van de Gateway                  | Stel in op `true` om URL-toegestane lijsten voor URL-ophaalinvoer te vereisen.                         |
| `gateway.nodes.denyCommands`            | `gateway.nodes.commands.deny`                  | Vereis dat exacte Node-opdracht-id's zoals `system.run` in de OpenClaw-configuratie worden geweigerd. |

`gateway.nodes.denyCommands` is een exacte, hoofdlettergevoelige beleidsregel voor een weigeringssuperset.
Gebruik deze wanneer het beleid moet aantonen dat bevoorrechte Node-opdrachten expliciet
door de OpenClaw-configuratie worden geweigerd. Een implementatie die bewust een bevoorrechte
Node-opdracht toestaat, moet na beoordeling `policy.jsonc` bijwerken in plaats van alleen op
`gateway.nodes.commands.allow` te vertrouwen.

#### Agentwerkruimte

| Beleidsveld                     | Waargenomen status                                                                           | Gebruiken wanneer                                                                                 |
| -------------------------------- | ---------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| `agents.workspace.allowedAccess` | `agents.defaults.sandbox.workspaceAccess` en `agents.entries.*.sandbox.workspaceAccess` | Sta alleen waarden voor toegang tot de sandboxwerkruimte toe, zoals `none` of `ro`.                       |
| `agents.workspace.denyTools`     | Algemene en agentspecifieke configuratie voor het weigeren van tools                                                    | Vereis dat mutatietools (`exec`, `process`, `write`, `edit`, `apply_patch`) worden geweigerd. |

#### Sandboxconfiguratie

| Beleidsveld                                          | Waargenomen status                                          | Gebruiken wanneer                                                       |
| ----------------------------------------------------- | ------------------------------------------------------- | -------------------------------------------------------------- |
| `sandbox.requireMode`                                 | `agents.defaults.sandbox.mode` en agentspecifieke modus       | Sta alleen beoordeelde sandboxmodi toe, zoals `all` of `non-main`. |
| `sandbox.allowBackends`                               | `agents.defaults.sandbox.backend` en agentspecifieke backend | Sta alleen beoordeelde sandboxbackends toe, zoals `docker`.         |
| `sandbox.containers.denyHostNetwork`                  | Netwerkmodus van de containergebaseerde sandbox/browser           | Weiger de hostnetwerkmodus.                                        |
| `sandbox.containers.denyContainerNamespaceJoin`       | Netwerkmodus van de containergebaseerde sandbox/browser           | Weiger deelname aan de netwerknaamruimte van een andere container.              |
| `sandbox.containers.requireReadOnlyMounts`            | Koppelmodus van de containergebaseerde sandbox/browser             | Vereis dat koppelingen alleen-lezen zijn.                                |
| `sandbox.containers.denyContainerRuntimeSocketMounts` | Koppeldoelen van de containergebaseerde sandbox/browser          | Weiger koppelingen van sockets van de containerruntime.                          |
| `sandbox.containers.denyUnconfinedProfiles`           | Configuratie van containerbeveiligingsprofielen                      | Weiger onbeperkte containerbeveiligingsprofielen.                   |
| `sandbox.browser.requireCdpSourceRange`               | CDP-bronbereik van de sandboxbrowser                        | Vereis dat blootstelling van browser-CDP een bronbereik opgeeft.        |

Het beleid behandelt een ontbrekende `sandbox.mode` als de impliciete standaardwaarde `off`, zodat
`sandbox.requireMode` een nieuwe of niet-geconfigureerde sandbox rapporteert als buiten een
toegestane lijst zoals `["all"]`.

#### Gegevensverwerking

| Beleidsveld                                        | Waargenomen status                                                                                     | Gebruiken wanneer                                                               |
| --------------------------------------------------- | -------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| `dataHandling.sensitiveLogging.requireRedaction`    | `logging.redactSensitive`                                                                          | Stel in op `true` om `logging.redactSensitive: "off"` te weigeren.              |
| `dataHandling.telemetry.denyContentCapture`         | `diagnostics.otel.captureContent`                                                                  | Stel in op `true` om het vastleggen van telemetrie-inhoud te weigeren.                     |
| `dataHandling.retention.requireSessionMaintenance`  | `session.maintenance.mode`                                                                         | Stel in op `true` om de effectieve sessieonderhoudsmodus `enforce` te vereisen. |
| `dataHandling.memory.denySessionTranscriptIndexing` | `memory.qmd.sessions.enabled`, `memory.search.experimental.sessionMemory` en agentspecifieke overschrijvingen | Stel in op `true` om het indexeren van sessietranscripten in het geheugen te weigeren.       |

#### Geheimen

| Beleidsveld                      | Waargenomen status                                           | Gebruiken wanneer                                                                |
| --------------------------------- | -------------------------------------------------------- | ----------------------------------------------------------------------- |
| `secrets.requireManagedProviders` | SecretRefs in de configuratie en `secrets.providers.*`-declaraties | Stel in op `true` om te vereisen dat SecretRefs naar gedeclareerde providers verwijzen.     |
| `secrets.denySources`             | Bronnen van geheimproviders en SecretRef-bronnen            | Weiger bronnen zoals `exec`, `file` of een andere geconfigureerde bronnaam. |
| `secrets.allowInsecureProviders`  | Onveilige configuratievlaggen van geheimproviders                   | Stel in op `false` om providers te weigeren die voor een onveilige configuratie kiezen.      |

#### Exec-goedkeuringen

Controles van Exec-goedkeuringen lezen het runtime-artefact `exec-approvals.json`:
standaard `~/.openclaw/exec-approvals.json`, of
`$OPENCLAW_STATE_DIR/exec-approvals.json` wanneer `OPENCLAW_STATE_DIR` is ingesteld.
Configuratieregels onder `execApprovals.defaults.*` of `execApprovals.agents.*`
vereisen leesbaar bewijs uit het artefact; een ontbrekend of ongeldig artefact wordt gerapporteerd als
niet-waarneembaar bewijs in plaats van een goedkeuring op basis van een best-effortpoging. Zodra het leesbaar is, nemen weggelaten
velden de runtime-standaardwaarden over: een ontbrekende `defaults.security` is `full`, en
ontbrekende agentbeveiliging neemt die standaardwaarde over. Bewijs omvat `defaults`,
`agents.*`, `agents.*.allowlist[].pattern`, optioneel `argPattern`, de effectieve
`autoAllowSkills`-configuratie en de bron van de vermelding — nooit het socketpad/token,
`commandText`, `lastUsedCommand`, opgeloste paden of tijdstempels.

| Beleidsveld                                | Waargenomen status                                                                         | Gebruiken wanneer                                                                                |
| ------------------------------------------- | -------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| `execApprovals.requireFile`                 | Pad van actieve runtime-`exec-approvals.json`                                              | Stel in op `true` om te vereisen dat het goedkeuringsartefact bestaat en kan worden geparseerd.                     |
| `execApprovals.defaults.allowSecurity`      | `defaults.security`, met standaardwaarde `full`                                              | Sta alleen goedgekeurde standaardbeveiligingsmodi voor goedkeuring toe.                                    |
| `execApprovals.agents.allowSecurity`        | `agents.*.security`, waarbij standaardwaarden worden overgenomen                                               | Sta alleen goedgekeurde effectieve beveiligingsmodi voor goedkeuring per agent toe.                        |
| `execApprovals.agents.allowAutoAllowSkills` | `defaults.autoAllowSkills` en `agents.*.autoAllowSkills`, waarbij standaardwaarden van de runtime worden overgenomen | Stel in op `false` om strikte handmatige toestemmingslijsten zonder impliciete goedkeuring van Skills-CLI's te vereisen. |
| `execApprovals.agents.allowlist.expected`   | Geaggregeerd `agents.*.allowlist[]`-patroon en optionele argPattern-vermeldingen               | Vereis dat de goedkeuringstoestemmingslijst overeenkomt met de beoordeelde patroonset.                      |

Voorbeeld: vereis het goedkeuringsartefact, weiger ruime standaardwaarden en sta
alleen een beoordeelde houding voor uitvoeringsgoedkeuring toe voor geselecteerde agents.

```jsonc
{
  "execApprovals": {
    "requireFile": true,
    "defaults": {
      // Beveiligingsmodi: "deny", "allowlist" of "full".
      // Deze standaardwaarde staat alleen de vergrendelde weigeringshouding toe.
      "allowSecurity": ["deny"],
    },
  },
  "scopes": {
    "restricted-shell": {
      "agentIds": ["family-agent", "groups-agent"],
      "execApprovals": {
        "agents": {
          // Geselecteerde agents mogen de beoordeelde toestemmingslijsthouding gebruiken, maar niet "full".
          "allowSecurity": ["allowlist"],
          // false betekent dat Skills-CLI's in de beoordeelde toestemmingslijst moeten voorkomen in plaats van
          // impliciet te worden goedgekeurd door autoAllowSkills.
          "allowAutoAllowSkills": false,
          "allowlist": {
            "expected": [
              // Eenvoudige vermelding: exact beoordeeld uitvoerbaar patroon zonder argPattern.
              "travel-hub",
              // Beperkte vermelding: patroon plus beoordeelde reguliere expressie voor argumenten.
              { "pattern": "calendar-cli", "argPattern": "^sync\\b" },
              "/bin/date",
            ],
          },
        },
      },
    },
  },
}
```

#### Authenticatieprofielen

| Beleidsveld                    | Waargenomen status                               | Gebruiken wanneer                                                                                   |
| ------------------------------- | -------------------------------------------- | ------------------------------------------------------------------------------------------ |
| `auth.profiles.requireMetadata` | Metadata over de `auth.profiles.*`-provider en -modus | Vereis metadatasleutels zoals `provider` en `mode` voor authenticatieprofielen in de configuratie.               |
| `auth.profiles.allowModes`      | `auth.profiles.*.mode`                       | Sta alleen ondersteunde authenticatieprofielmodi toe, zoals `api_key`, `aws-sdk`, `oauth` of `token`. |

#### Toolmetadata

| Beleidsveld            | Waargenomen status                   | Gebruiken wanneer                                                                                   |
| ----------------------- | -------------------------------- | ------------------------------------------------------------------------------------------ |
| `tools.requireMetadata` | Beheerde `TOOLS.md`-declaraties | Vereis dat beheerde tools metadatasleutels declareren, zoals `risk`, `sensitivity` of `owner`. |

#### Toolhouding

| Beleidsveld                    | Waargenomen status                                              | Gebruiken wanneer                                                                                                 |
| ------------------------------- | ----------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `tools.profiles.allow`          | `tools.profile` en `agents.entries.*.tools.profile`        | Sta alleen toolprofiel-id's toe, zoals `minimal`, `messaging` of `coding`.                                 |
| `tools.fs.requireWorkspaceOnly` | `tools.fs.workspaceOnly` en `tools.fs`-overschrijvingen per agent | Stel in op `true` om een toolhouding voor alleen het werkgebiedbestandssysteem te vereisen.                                         |
| `tools.exec.allowSecurity`      | `tools.exec.security` en uitvoeringsbeveiliging per agent           | Sta alleen uitvoeringsbeveiligingsmodi toe, zoals `deny` of `allowlist`.                                            |
| `tools.exec.requireAsk`         | `tools.exec.ask` en vraagmodus voor uitvoering per agent                | Vereis een goedkeuringshouding zoals `always`.                                                               |
| `tools.exec.allowHosts`         | `tools.exec.host` en hostroutering voor uitvoering per agent           | Sta alleen hostrouteringsmodi voor uitvoering toe, zoals `sandbox`.                                                    |
| `tools.elevated.allow`          | `tools.elevated.enabled` en verhoogde houding per agent     | Stel in op `false` om te vereisen dat de verhoogde toolmodus uitgeschakeld blijft.                                           |
| `tools.alsoAllow.expected`      | `tools.alsoAllow` en `tools.alsoAllow` per agent           | Vereis exacte `alsoAllow`-vermeldingen en rapporteer ontbrekende of onverwachte aanvullende tooltoekenningen.                 |
| `tools.denyTools`               | `tools.deny` en `agents.entries.*.tools.deny`              | Vereis dat geconfigureerde toolweigeringslijsten tool-id's of groepen bevatten, zoals `group:runtime` en `group:fs`. |

## Controles uitvoeren

Voer tijdens het opstellen alleen beleidscontroles uit:

```bash
openclaw policy check
openclaw policy check --json
openclaw policy check --severity-min error
```

`policy check` voert alleen de beleidscontroleset uit en produceert bewijsmateriaal, bevindingen
en attestatiehashes. Dezelfde bevindingen verschijnen ook in
`openclaw doctor --lint` wanneer de Policy-plugin is ingeschakeld.

Vergelijk een beleidsbestand van een beheerder met een opgestelde basislijn:

```bash
openclaw policy compare --baseline official.policy.jsonc
openclaw policy compare --baseline official.policy.jsonc --policy policy.jsonc --json
```

`policy compare` controleert de syntaxis van een beleidsbestand tegen de syntaxis van een beleidsbestand; het
inspecteert geen runtimestatus, bewijsmateriaal, aanmeldgegevens of geheimen. Het gebruikt dezelfde
regelmetadata die overlays met een bereik beheert: toestemmingslijsten moeten gelijk of
beperkter blijven, weigeringslijsten moeten gelijk of ruimer blijven, vereiste booleaanse waarden moeten
hun waarde behouden, geordende tekenreeksen mogen alleen naar het strengere uiteinde van de
geconfigureerde volgorde bewegen en exacte lijsten moeten overeenkomen. De basislijn kan een
door een organisatie opgesteld beleid zijn; het gecontroleerde beleid mag strengere waarden of
extra regels toevoegen. Een gecontroleerde regel op het hoogste niveau kan voldoen aan een basislijnregel met een bereik wanneer
deze even beperkend of beperkter is. Bereiknamen hoeven tussen
bestanden niet overeen te komen; de vergelijking wordt bepaald door selector (`agentIds`/`channelIds`) en veld.
Voor routeringsprobes moet elke probe-id in de basislijn behouden blijven met dezelfde route
en verwachte agent. Een gecontroleerd beleid mag probes toevoegen of `matchedBy` beperken, maar
het verwijderen van een probe, het wijzigen van de route of agent, of het verruimen van de geaccepteerde overeenkomsttypen
is minder streng.

Geslaagde vergelijking (`--json`):

```json
{
  "ok": true,
  "baselinePath": "official.policy.jsonc",
  "policyPath": "policy.jsonc",
  "rulesChecked": 3,
  "findings": []
}
```

Geslaagde `policy check --json`-uitvoer bevat stabiele hashes die een beheerder of
toezichthouder kan vastleggen:

```json
{
  "ok": true,
  "attestation": {
    "policy": {
      "path": "policy.jsonc",
      "hash": "sha256:..."
    },
    "workspace": {
      "scope": "policy",
      "hash": "sha256:..."
    },
    "findingsHash": "sha256:...",
    "attestationHash": "sha256:..."
  },
  "checksRun": 5,
  "checksSkipped": 0,
  "findings": []
}
```

## Beleid configureren

De beleidsconfiguratie bevindt zich onder `plugins.entries.policy.config`.

```jsonc
{
  "plugins": {
    "entries": {
      "policy": {
        "enabled": true,
        "config": {
          "enabled": true,
          "path": "policy.jsonc",
          "workspaceRepairs": false,
          "expectedHash": "sha256:...",
          "expectedAttestationHash": "sha256:...",
        },
      },
    },
  },
}
```

| Instelling                   | Doel                                                         |
| ------------------------- | --------------------------------------------------------------- |
| `enabled`                 | Schakel beleidscontroles in, zelfs voordat `policy.jsonc` bestaat.         |
| `workspaceRepairs`        | Sta toe dat `doctor --fix` door beleid beheerde werkgebiedinstellingen bewerkt. |
| `expectedHash`            | Optionele hashvergrendeling voor het goedgekeurde beleidsartefact.            |
| `expectedAttestationHash` | Optionele hashvergrendeling voor de laatst geaccepteerde geslaagde beleidscontrole.    |
| `path`                    | Werkgebiedrelatieve locatie van het beleidsartefact.             |

Stel `plugins.entries.policy.config.enabled` in op `false` om beleidscontroles
voor een werkgebied uit te schakelen terwijl de plugin geïnstalleerd blijft.

## Beleidsstatus accepteren

Voorbeeld van JSON-uitvoer:

```json
{
  "ok": true,
  "attestation": {
    "checkedAt": "2026-05-10T20:00:00.000Z",
    "policy": {
      "path": "policy.jsonc",
      "hash": "sha256:..."
    },
    "workspace": {
      "scope": "policy",
      "hash": "sha256:..."
    },
    "findingsHash": "sha256:...",
    "attestationHash": "sha256:..."
  },
  "evidence": {
    "channels": [
      {
        "id": "telegram",
        "provider": "telegram",
        "source": "oc://openclaw.config/channels/telegram",
        "enabled": false
      }
    ],
    "mcpServers": [
      {
        "id": "docs",
        "transport": "stdio",
        "source": "oc://openclaw.config/mcp/servers/docs",
        "command": "npx"
      }
    ],
    "modelProviders": [
      {
        "id": "openai",
        "source": "oc://openclaw.config/models/providers/openai"
      }
    ],
    "modelRefs": [
      {
        "ref": "openai/gpt-5.6-sol",
        "provider": "openai",
        "model": "gpt-5.6-sol",
        "source": "oc://openclaw.config/agents/defaults/model"
      }
    ],
    "network": [
      {
        "id": "browser-private-network",
        "source": "oc://openclaw.config/browser/ssrfPolicy/dangerouslyAllowPrivateNetwork",
        "value": false
      }
    ],
    "gatewayExposure": [
      {
        "id": "gateway-bind",
        "kind": "bind",
        "source": "oc://openclaw.config/gateway/bind",
        "value": "loopback",
        "nonLoopback": false,
        "explicit": true
      }
    ],
    "agentWorkspace": [
      {
        "id": "agents-defaults-workspace-access",
        "kind": "workspaceAccess",
        "source": "oc://openclaw.config/agents/defaults/sandbox/workspaceAccess",
        "scope": "defaults",
        "value": "ro",
        "sandboxMode": "all",
        "sandboxModeSource": "oc://openclaw.config/agents/defaults/sandbox/mode",
        "sandboxEnabled": true,
        "explicit": true
      },
      {
        "id": "agents-defaults-tool-exec",
        "kind": "toolDeny",
        "source": "oc://openclaw.config/tools/deny",
        "scope": "defaults",
        "tool": "exec",
        "denied": true,
        "explicit": true
      }
    ],
    "secrets": [
      {
        "id": "vault",
        "kind": "provider",
        "source": "oc://openclaw.config/secrets/providers/vault",
        "providerSource": "env"
      },
      {
        "id": "oc://openclaw.config/models/providers/openai/apiKey",
        "kind": "input",
        "source": "oc://openclaw.config/models/providers/openai/apiKey",
        "provenance": "secretRef",
        "refSource": "env",
        "refProvider": "vault"
      }
    ],
    "authProfiles": [
      {
        "id": "github",
        "source": "oc://openclaw.config/auth/profiles/github",
        "validMetadata": true,
        "provider": "github",
        "mode": "token"
      }
    ],
    "tools": [
      {
        "id": "deploy",
        "source": "oc://TOOLS.md/tools/deploy",
        "line": 12,
        "risk": "critical",
        "sensitivity": "restricted",
        "capabilities": ["IRREVERSIBLE_EXTERNAL"]
      }
    ]
  },
  "checksRun": 30,
  "checksSkipped": 0,
  "findings": []
}
```

`attestation.policy.hash` identificeert het opgestelde regelartefact. `evidence`
registreert de waargenomen OpenClaw-status die door de controles is gebruikt, en
`workspace.hash` identificeert die bewijspayload. `findingsHash` identificeert
de exacte reeks bevindingen. `checkedAt` registreert wanneer de controle is uitgevoerd.
`attestationHash` identificeert de stabiele claim (beleidshash, bewijshash,
bevindingenhash en schone/vuile status) en sluit bewust `checkedAt` uit,
zodat dezelfde beleidsstatus altijd dezelfde attestatiehash oplevert. Samen
vormen deze vier waarden het audittupel voor één beleidscontrole.

Als een Gateway of supervisor beleid gebruikt om een runtimeactie te blokkeren,
goed te keuren of van een annotatie te voorzien, moet deze de attestatiehash van
de laatste schone controle registreren. `checkedAt` blijft in de JSON-uitvoer
voor auditlogboeken, maar maakt geen deel uit van de stabiele hash.

Levenscyclus voor het accepteren van de beleidsstatus:

1. Stel `policy.jsonc` op of beoordeel dit.
2. Voer `openclaw policy check --json` uit.
3. Registreer bij een schone status `attestation.policy.hash` als `expectedHash`.
4. Registreer `attestation.attestationHash` als `expectedAttestationHash`.
5. Voer `openclaw doctor --lint` opnieuw uit in CI- of releasepoorten.

Als beleidsregels opzettelijk veranderen, werk je beide geaccepteerde hashes bij
op basis van een schone controle. Als alleen werkruimte-instellingen veranderen
(het beleid blijft hetzelfde), verandert doorgaans alleen `expectedAttestationHash`.

Het inschakelen of upgraden van `agents.workspace`-regels voegt
`agentWorkspace`-bewijs toe aan de werkruimtehash en attestatiehash; beoordeel
het nieuwe bewijs en vernieuw de geaccepteerde attestatiehashes na het
inschakelen. Het inschakelen of upgraden van regels voor de toolhouding voegt
op dezelfde manier `toolPosture`-bewijs toe.

`openclaw policy watch` voert de controle opnieuw uit en meldt wanneer het huidige
bewijs niet meer overeenkomt met `expectedAttestationHash`:

```bash
openclaw policy watch --json
```

Gebruik `--once` in CI of scripts die één driftevaluatie nodig hebben.
Zonder `--once` wordt standaard elke twee seconden gepolld; gebruik
`--interval-ms` om het interval te wijzigen.

## Bevindingen

| Controle-id                                                 | Bevinding                                                                           |
| -------------------------------------------------------- | --------------------------------------------------------------------------------- |
| `policy/policy-jsonc-missing`                            | Beleid is ingeschakeld, maar `policy.jsonc` ontbreekt.                                  |
| `policy/policy-jsonc-invalid`                            | Beleid kan niet worden geparseerd of bevat onjuist gevormde regelvermeldingen.                       |
| `policy/policy-hash-mismatch`                            | Beleid komt niet overeen met de geconfigureerde `expectedHash`.                                  |
| `policy/attestation-hash-mismatch`                       | Het huidige beleidsbewijs komt niet meer overeen met de geaccepteerde attestatie.               |
| `policy/policy-conformance-invalid`                      | Een basislijn- of gecontroleerd beleidsbestand bevat ongeldige vergelijkingssyntaxis.                  |
| `policy/policy-conformance-missing`                      | In een gecontroleerd beleidsbestand ontbreekt een regel die door het basislijnbeleidsbestand wordt vereist.     |
| `policy/policy-conformance-weaker`                       | Een gecontroleerd beleidsbestand heeft een zwakkere waarde dan het basislijnbeleidsbestand.           |
| `policy/channels-denied-provider`                        | Een ingeschakeld kanaal komt overeen met een weigeringsregel voor kanalen.                                   |
| `policy/mcp-denied-server`                               | Een geconfigureerde MCP-server wordt door het beleid geweigerd.                                      |
| `policy/mcp-unapproved-server`                           | Een geconfigureerde MCP-server staat niet op de toelatingslijst.                                 |
| `policy/models-denied-provider`                          | Een geconfigureerde modelprovider of modelreferentie gebruikt een geweigerde provider.                  |
| `policy/models-unapproved-provider`                      | Een geconfigureerde modelprovider of modelreferentie staat niet op de toelatingslijst.                |
| `policy/network-private-access-enabled`                  | Een uitwijkmogelijkheid voor SSRF via het privénetwerk is ingeschakeld terwijl het beleid dit weigert.             |
| `policy/routing-bindings-required`                       | Het beleid vereist een routeringskoppeling voor een kanaal, maar er is geen geconfigureerd.                  |
| `policy/routing-binding-channel-unconfigured`            | Een routeringskoppeling noemt een kanaal dat ontbreekt in `channels.*`.                         |
| `policy/routing-agent-mismatch`                          | Een vastgelegde route wordt naar een andere agent omgezet.                                  |
| `policy/routing-match-kind-mismatch`                     | Een vastgelegde route komt overeen met een onverwachte koppelingsspecificiteit.                   |
| `policy/ingress-dm-policy-unapproved`                    | Een DM-beleid voor een kanaal staat niet op de toelatingslijst van het beleid.                              |
| `policy/ingress-dm-scope-unapproved`                     | `session.dmScope` komt niet overeen met het door het beleid vereiste isolatiebereik voor DM's.          |
| `policy/ingress-open-groups-denied`                      | Een groepsbeleid voor een kanaal is `open` terwijl het beleid open groepsingang weigert.          |
| `policy/ingress-group-mention-required`                  | Een kanaal- of groepsvermelding schakelt vermeldingspoorten uit terwijl het beleid deze vereist.       |
| `policy/gateway-non-loopback-bind`                       | De bindingsconfiguratie van de Gateway staat blootstelling buiten de loopback-interface toe terwijl het beleid dit weigert.         |
| `policy/gateway-auth-disabled`                           | Gateway-authenticatie is uitgeschakeld terwijl het beleid authenticatie vereist.                     |
| `policy/gateway-rate-limit-missing`                      | De configuratie voor snelheidsbegrenzing van Gateway-authenticatie is niet expliciet terwijl het beleid dit vereist.          |
| `policy/gateway-control-ui-insecure`                     | Schakelaars voor onveilige blootstelling van de Gateway Control UI zijn ingeschakeld.                         |
| `policy/gateway-tailscale-funnel`                        | Blootstelling via Gateway Tailscale Funnel is ingeschakeld terwijl het beleid dit weigert.               |
| `policy/gateway-remote-enabled`                          | De externe modus van de Gateway is actief terwijl het beleid dit weigert.                              |
| `policy/gateway-http-endpoint-enabled`                   | Een HTTP-API-eindpunt van de Gateway is ingeschakeld terwijl het door het beleid wordt geweigerd.                    |
| `policy/gateway-http-url-fetch-unrestricted`             | Invoer voor het ophalen van URL's via Gateway HTTP mist een vereiste URL-toelatingslijst.                      |
| `policy/gateway-node-command-denied`                     | Een Node-opdracht die door het beleid wordt geweigerd, wordt niet door de OpenClaw-configuratie geweigerd.                 |
| `policy/agents-workspace-access-denied`                  | De sandboxmodus of werkruimtetoegang van de agent staat niet op de toelatingslijst van het beleid.           |
| `policy/agents-tool-not-denied`                          | Een agent- of standaardconfiguratie weigert een door het beleid vereist hulpmiddel niet.               |
| `policy/tools-profile-unapproved`                        | Een geconfigureerd globaal of agentspecifiek hulpmiddelenprofiel staat niet op de toelatingslijst.           |
| `policy/tools-fs-workspace-only-required`                | Bestandssysteemhulpmiddelen zijn niet geconfigureerd met een padconfiguratie die uitsluitend de werkruimte toestaat.             |
| `policy/tools-exec-security-unapproved`                  | De beveiligingsmodus voor uitvoering staat niet op de toelatingslijst van het beleid.                               |
| `policy/tools-exec-ask-unapproved`                       | De vraagmodus voor uitvoering staat niet op de toelatingslijst van het beleid.                                    |
| `policy/tools-exec-host-unapproved`                      | De hostroutering voor uitvoering staat niet op de toelatingslijst van het beleid.                                |
| `policy/tools-elevated-enabled`                          | De verhoogde hulpmiddelenmodus is ingeschakeld terwijl het beleid dit weigert.                              |
| `policy/tools-also-allow-missing`                        | In een geconfigureerde `alsoAllow`-lijst ontbreekt een door het beleid vereiste vermelding.             |
| `policy/tools-also-allow-unexpected`                     | Een geconfigureerde `alsoAllow`-lijst bevat een vermelding die niet door het beleid wordt verwacht.           |
| `policy/tools-required-deny-missing`                     | Een globale of agentspecifieke weigeringslijst voor hulpmiddelen bevat een vereist geweigerd hulpmiddel niet.     |
| `policy/sandbox-mode-unapproved`                         | De sandboxmodus staat niet op de toelatingslijst van het beleid.                                     |
| `policy/sandbox-backend-unapproved`                      | De sandboxbackend staat niet op de toelatingslijst van het beleid.                                  |
| `policy/sandbox-container-posture-unobservable`          | Een regel voor containerconfiguratie is ingeschakeld voor een backend die deze niet kan waarnemen.         |
| `policy/sandbox-container-host-network-denied`           | Een containergebaseerde sandbox of browser gebruikt de netwerkmodus van de host.                     |
| `policy/sandbox-container-namespace-join-denied`         | Een containergebaseerde sandbox of browser neemt deel aan de naamruimte van een andere container.          |
| `policy/sandbox-container-mount-mode-required`           | Een koppelpunt van een containergebaseerde sandbox of browser is niet alleen-lezen.                     |
| `policy/sandbox-container-runtime-socket-mount`          | Een koppelpunt van een containergebaseerde sandbox of browser stelt de socket van de containerruntime bloot. |
| `policy/sandbox-container-unconfined-profile`            | Het containersandboxprofiel is onbeperkt terwijl het beleid dit weigert.                    |
| `policy/sandbox-browser-cdp-source-range-missing`        | Het CDP-bronbereik van de sandboxbrowser ontbreekt terwijl het beleid dit vereist.             |
| `policy/data-handling-redaction-disabled`                | Redactie van gevoelige logboekgegevens is uitgeschakeld terwijl het beleid dit vereist.                  |
| `policy/data-handling-telemetry-content-capture`         | Het vastleggen van telemetrie-inhoud is ingeschakeld terwijl het beleid dit weigert.                       |
| `policy/data-handling-session-retention-not-enforced`    | Onderhoud voor sessiebewaring wordt niet afgedwongen terwijl het beleid dit vereist.            |
| `policy/data-handling-session-transcript-memory-enabled` | Geheugenindexering van sessietranscripten is ingeschakeld terwijl het beleid dit weigert.              |
| `policy/secrets-unmanaged-provider`                      | Een SecretRef in de configuratie verwijst naar een provider die niet is gedeclareerd onder `secrets.providers`.  |
| `policy/secrets-denied-provider-source`                  | Een provider van configuratiegeheimen of SecretRef gebruikt een bron die door het beleid wordt geweigerd.             |
| `policy/secrets-insecure-provider`                       | Een geheimprovider kiest voor een onveilige configuratie terwijl het beleid dit weigert.               |
| `policy/auth-profile-invalid-metadata`                   | In een authenticatieprofiel in de configuratie ontbreken geldige metadata voor de provider of modus.                 |
| `policy/auth-profile-unapproved-mode`                    | De modus van een authenticatieprofiel in de configuratie staat niet op de toelatingslijst van het beleid.                       |
| `policy/exec-approvals-missing`                          | Het beleid vereist `exec-approvals.json`, maar het artefact ontbreekt.               |
| `policy/exec-approvals-invalid`                          | Het geconfigureerde artefact voor uitvoeringsgoedkeuringen kan niet worden geparseerd.                          |
| `policy/exec-approvals-default-security-unapproved`      | De standaardwaarden voor uitvoeringsgoedkeuringen gebruiken een beveiligingsmodus die niet op de toelatingslijst van het beleid staat.          |
| `policy/exec-approvals-agent-security-unapproved`        | Een effectieve agentspecifieke beveiligingsmodus voor uitvoeringsgoedkeuringen staat niet op de toelatingslijst.       |
| `policy/exec-approvals-auto-allow-skills-enabled`        | Een agent voor uitvoeringsgoedkeuringen staat CLI's van Skills impliciet automatisch toe terwijl het beleid dit weigert.   |
| `policy/exec-approvals-allowlist-missing`                | In de toelatingslijst voor goedkeuringen ontbreekt een door het beleid vereist patroon.                  |
| `policy/exec-approvals-allowlist-unexpected`             | De toelatingslijst voor goedkeuringen bevat een patroon dat niet door het beleid wordt verwacht.                |
| `policy/tools-missing-risk-level`                        | In een declaratie van een beheerd hulpmiddel ontbreken risicomMetadata.                             |
| `policy/tools-unknown-risk-level`                        | Een declaratie van een beheerd hulpmiddel gebruikt een onbekende risicowaarde.                           |
| `policy/tools-missing-sensitivity-token`                 | In een declaratie van een beheerd hulpmiddel ontbreken gevoeligheidsmetadata.                      |
| `policy/tools-missing-owner`                             | In een declaratie van een beheerd hulpmiddel ontbreken eigenaarsmetadata.                            |
| `policy/tools-unknown-sensitivity-token`                 | Een declaratie van een beheerd hulpmiddel gebruikt een onbekende gevoeligheidswaarde.                    |

Een bevinding kan zowel `target` bevatten (het waargenomen element in de werkruimte dat
niet voldoet) als `requirement` (de vastgelegde regel waardoor het een bevinding werd).
Beide zijn momenteel `oc://`-adresreeksen, maar de veldnamen beschrijven de beleidsrol
in plaats van de adresindeling.

Voorbeeldbevindingen:

```json
{
  "checkId": "policy/channels-denied-provider",
  "severity": "error",
  "message": "Kanaal 'telegram' gebruikt de geweigerde provider 'telegram'.",
  "source": "policy",
  "path": "openclaw-configuratie",
  "ocPath": "oc://openclaw.config/channels/telegram",
  "target": "oc://openclaw.config/channels/telegram",
  "requirement": "oc://policy.jsonc/channels/denyRules/#0",
  "fixHint": "Telegram is niet goedgekeurd voor deze werkruimte."
}
```

```json
{
  "checkId": "policy/tools-missing-risk-level",
  "severity": "error",
  "message": "Het hulpmiddel 'deploy' in TOOLS.md heeft geen expliciete risicoclassificatie.",
  "source": "policy",
  "path": "TOOLS.md",
  "line": 12,
  "ocPath": "oc://TOOLS.md/tools/deploy",
  "target": "oc://TOOLS.md/tools/deploy",
  "requirement": "oc://policy.jsonc/tools/requireMetadata"
}
```

```json
{
  "checkId": "policy/mcp-unapproved-server",
  "severity": "error",
  "message": "MCP-server 'remote' staat niet op de toelatingslijst van het beleid.",
  "source": "policy",
  "path": "openclaw-configuratie",
  "ocPath": "oc://openclaw.config/mcp/servers/remote",
  "target": "oc://openclaw.config/mcp/servers/remote",
  "requirement": "oc://policy.jsonc/mcp/servers/allow"
}
```

```json
{
  "checkId": "policy/models-unapproved-provider",
  "severity": "error",
  "message": "Modelreferentie 'anthropic/claude-sonnet-4.7' gebruikt de niet-goedgekeurde provider 'anthropic'.",
  "source": "policy",
  "path": "openclaw-configuratie",
  "ocPath": "oc://openclaw.config/agents/defaults/model/fallbacks/#0",
  "target": "oc://openclaw.config/agents/defaults/model/fallbacks/#0",
  "requirement": "oc://policy.jsonc/models/providers/allow"
}
```

```json
{
  "checkId": "policy/network-private-access-enabled",
  "severity": "error",
  "message": "Netwerkinstelling 'browser-private-network' staat toegang tot het privénetwerk toe.",
  "source": "policy",
  "path": "openclaw-configuratie",
  "ocPath": "oc://openclaw.config/browser/ssrfPolicy/dangerouslyAllowPrivateNetwork",
  "target": "oc://openclaw.config/browser/ssrfPolicy/dangerouslyAllowPrivateNetwork",
  "requirement": "oc://policy.jsonc/network/privateNetwork/allow"
}
```

```json
{
  "checkId": "policy/gateway-non-loopback-bind",
  "severity": "error",
  "message": "De Gateway-bindingsinstelling 'gateway-bind' staat blootstelling buiten de loopbackinterface toe.",
  "source": "policy",
  "path": "openclaw config",
  "ocPath": "oc://openclaw.config/gateway/bind",
  "target": "oc://openclaw.config/gateway/bind",
  "requirement": "oc://policy.jsonc/gateway/exposure/allowNonLoopbackBind"
}
```

```json
{
  "checkId": "policy/gateway-node-command-denied",
  "severity": "error",
  "message": "Het Gateway-Node-commando 'system.run' wordt door het beleid geweigerd, maar niet door de OpenClaw-configuratie.",
  "source": "policy",
  "path": "openclaw config",
  "ocPath": "oc://openclaw.config/gateway/nodes/commands/deny",
  "target": "oc://openclaw.config/gateway/nodes/commands/deny",
  "requirement": "oc://policy.jsonc/gateway/nodes/denyCommands",
  "fixHint": "Voeg 'system.run' toe aan gateway.nodes.commands.deny of werk het beleid na controle bij."
}
```

```json
{
  "checkId": "policy/agents-workspace-access-denied",
  "severity": "error",
  "message": "agents.defaults sandbox workspaceAccess 'rw' is volgens het beleid niet toegestaan.",
  "source": "policy",
  "path": "openclaw config",
  "ocPath": "oc://openclaw.config/agents/defaults/sandbox/workspaceAccess",
  "target": "oc://openclaw.config/agents/defaults/sandbox/workspaceAccess",
  "requirement": "oc://policy.jsonc/agents/workspace/allowedAccess"
}
```

## Herstel

`doctor --lint` en `policy check` zijn alleen-lezen.

`doctor --fix` bewerkt alleen door beleid beheerde werkruimte-instellingen wanneer
`workspaceRepairs` expliciet is ingeschakeld; anders melden controles wat ze
zouden herstellen en laten ze de instellingen ongewijzigd.

In deze versie kan herstel kanalen uitschakelen die door `channels.denyRules` worden geweigerd en
de hieronder vermelde automatische inperkingsreparaties toepassen. Schakel `workspaceRepairs`
alleen in nadat het beleidsbestand is gecontroleerd, omdat een geldige regel de
werkruimteconfiguratie kan wijzigen:

- stel `tools.elevated.enabled=false` in wanneer globaal beleid verhoogde tools verbiedt
- voeg ontbrekende verplicht te weigeren tool-id's toe aan `tools.deny` of
  `agents.entries.*.tools.deny` wanneer het beleid vereist dat die tools worden geweigerd
- stel onveilige `gateway.controlUi.*`-schakelaars in op `false`
- stel `gateway.mode=local` in wanneer het beleid de externe Gateway-modus weigert
- stel gemelde `gateway.http.endpoints.*.enabled`-paden in op `false` wanneer het beleid
  Gateway HTTP API-eindpunten weigert
- stel gemelde `groupPolicy`-paden voor kanaalingang in op `allowlist` wanneer het beleid
  open groepsingang weigert
- stel gemelde `requireMention`-paden voor kanaalingang in op `true` wanneer het beleid
  groepsvermeldingen vereist
- stel `logging.redactSensitive=tools` in wanneer het beleid redactie van gevoelige
  loggegevens vereist
- stel `diagnostics.otel.captureContent=false` in, of
  `diagnostics.otel.captureContent.enabled=false` voor telemetrie-
  vastleggingsinstellingen in objectvorm, wanneer het beleid vastlegging van telemetrie-inhoud weigert

Herstel van verhoogde tools met een beperkt bereik gebeurt alleen via detectie. Herstel van gegevensverwerking met een beperkt bereik
wordt ook overgeslagen wanneer de bevinding gedeelde configuratie voor logboekregistratie of telemetrie meldt,
omdat wijziging van de gedeelde instelling meer zou beïnvloeden dan het beleidsdoel
met beperkt bereik.

Herstel van verplichte weigeringen met een beperkt bereik wordt overgeslagen wanneer de bevinding overgenomen
hoofd-`tools.deny` meldt, omdat toevoeging van de vereiste tool aan de hoofdconfiguratie
meer zou beïnvloeden dan het beleidsdoel met beperkt bereik. Herstel van verplichte weigeringen op agentniveau kan
het gemelde `agents.entries.*.tools.deny`-pad bijwerken.

Herstel van kanaalingang met een beperkt bereik wordt overgeslagen wanneer de bevinding overgenomen
`channels.defaults.*` meldt, omdat wijziging van de gedeelde kanaalstandaard
meer zou beïnvloeden dan het beleidsdoel met beperkt bereik. Bevindingen voor de URL-ophaallijst van Gateway HTTP
blijven handmatig, omdat automatisch herstel niet de juiste waarden voor de
toegestane lijst met eindpunt-URL's kan kiezen.

Bevindingen voor Gateway-bindingen en Node-commando's blijven beoordeling vereisen. Wanneer
`policy/gateway-non-loopback-bind` of `policy/gateway-node-command-denied`
aan een configuratiepad kan worden gekoppeld, meldt `doctor --fix` de voorgestelde
wijziging van `gateway.bind` of `gateway.nodes.commands.deny` als overgeslagen
voorbeeldadvies. De wijziging wordt niet toegepast en de bevinding telt pas als
hersteld nadat een beheerder de configuratie of het beleid heeft gecontroleerd en bijgewerkt.

```jsonc
{
  "plugins": {
    "entries": {
      "policy": {
        "config": {
          "workspaceRepairs": true,
        },
      },
    },
  },
}
```

## Afsluitcodes

| Commando          | `0`                                                    | `1`                                                                 | `2`                          |
| ---------------- | ------------------------------------------------------ | ------------------------------------------------------------------- | ---------------------------- |
| `policy check`   | Geen bevindingen op de drempelwaarde.                          | Een of meer bevindingen bereikten de drempelwaarde.                             | Fout in argumenten of tijdens uitvoering. |
| `policy compare` | Het beleidsbestand is minstens even streng als de basislijn. | Het beleidsbestand is ongeldig, ontbreekt of is minder streng dan de basisregels. | Fout in argumenten of tijdens uitvoering. |
| `policy watch`   | Geen bevindingen en de geaccepteerde hash is actueel.              | Er bestaan bevindingen of de geaccepteerde attestatie is verouderd.                    | Fout in argumenten of tijdens uitvoering. |

## Gerelateerd

- [Lintmodus van Doctor](/nl/cli/doctor#lint-mode)
- [Pad-CLI](/nl/cli/path)
