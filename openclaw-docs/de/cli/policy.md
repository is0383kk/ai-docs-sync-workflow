---
read_when:
    - Sie möchten die OpenClaw-Einstellungen anhand einer erstellten `policy.jsonc` überprüfen
    - Sie möchten Richtlinienbefunde in der Doctor-Prüfung.
    - Sie benötigen einen Hash für die Richtlinienbestätigung als Auditnachweis.
summary: CLI-Referenz für `openclaw policy`-Konformitätsprüfungen
title: Richtlinie
x-i18n:
    generated_at: "2026-07-26T17:46:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 63e4faeab8dd6535e3d517439d3f58cdc167b6b7fade808a6482742ec9b5acf1
    source_path: cli/policy.md
    workflow: 16
---

# `openclaw policy`

`openclaw policy` wird vom gebündelten Policy-Plugin bereitgestellt. Es handelt sich um eine unternehmensweite
Konformitätsschicht über den bestehenden OpenClaw-Einstellungen, nicht um ein zweites Konfigurationssystem.
Sie definieren Anforderungen in `policy.jsonc`; OpenClaw erfasst den aktiven
Workspace als Nachweis; Policy meldet Abweichungen über `doctor --lint`. Policy
erzwingt keine Tool-Aufrufe und schreibt das Laufzeitverhalten nicht zum Anfragezeitpunkt um. Außerdem
attestiert es keine agentenspezifischen Anmeldedatenspeicher wie `auth-profiles.json`.

Policy prüft konfigurierte Kanäle, MCP-Server, Modell-Provider, die Netzwerk-SSRF-
Sicherheitslage, den Ingress-/Kanalzugriff, die Gateway-Exposition und die Befehlslage von Nodes,
definierte Nachrichten-Routing-Sonden,
den Zugriff auf Agenten-Workspaces, die Sandbox-Sicherheitslage, die Datenverarbeitungslage, die Sicherheitslage von
Secret-Providern/Authentifizierungsprofilen sowie kontrollierte Tool-Metadaten (`TOOLS.md`). Verwenden Sie es,
wenn ein Workspace eine dauerhafte, überprüfbare Aussage benötigt, etwa „Telegram darf
nicht aktiviert sein“ oder „kontrollierte Tools müssen Risiko- und Eigentümermetadaten deklarieren“. Wenn
Sie lediglich lokales Verhalten ohne Attestierung oder Abweichungserkennung benötigen, reicht die normale
Konfiguration aus.

## Schnellstart

```bash
openclaw plugins enable policy
```

Das Plugin bleibt auch dann aktiviert, wenn `policy.jsonc` fehlt, sodass Doctor
das fehlende Artefakt melden kann, anstatt Prüfungen stillschweigend zu überspringen.

Erstellen Sie `policy.jsonc` manuell; es wird nicht aus den aktuellen Einstellungen generiert. Jeder
Abschnitt der obersten Ebene ist ein Regel-Namespace: Eine Prüfung wird nur ausgeführt, wenn darin eine konkrete Regel
vorhanden ist (nicht unterstützte Abschnitte oder Schlüssel führen zu
`policy/policy-jsonc-invalid`, anstatt stillschweigend ignoriert zu werden). Minimales
Beispiel, das jeden unterstützten Abschnitt abdeckt:

```jsonc
{
  "channels": {
    "denyRules": [
      {
        "id": "no-telegram",
        "when": { "provider": "telegram" },
        "reason": "Telegram ist für diesen Workspace nicht genehmigt.",
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

Abschnittsübergreifende Hinweise, die aus den nachstehenden Regeltabellen nicht unmittelbar hervorgehen:

- Wenn `gateway.bind` bei gleichzeitigem Verbot von Bindungen außerhalb der Loopback-Schnittstelle weggelassen wird, akzeptieren Sie
  den Laufzeitstandard; legen Sie `gateway.bind: "loopback"` für strikte Konformität fest.
- Legen Sie für einen schreibgeschützten Agenten den Sandbox-Wert `mode` in den
  entsprechenden Standardwerten bzw. für den entsprechenden Agenten auf `all` oder `non-main` und `workspaceAccess` auf `none` oder `ro` fest. Ein fehlender oder
  auf `off` gesetzter Sandbox-Modus erfüllt keine Schreibschutzrichtlinie.
- `agents.workspace.denyTools` akzeptiert `exec`, `process`, `write`, `edit`,
  `apply_patch`. Die Tool-Verweigerungsgruppen der Konfiguration `group:fs` (Dateiänderungen) und
  `group:runtime` (Shell/Prozess) erfüllen die entsprechende Sicherheitslage.
- Prüfungen der Ausführungsgenehmigungen lesen das aktive Artefakt `exec-approvals.json` nur,
  wenn eine Regel `execApprovals` vorhanden ist; ein fehlendes oder ungültiges Artefakt ist
  nicht beobachtbarer Nachweis und kein künstlich erzeugter Erfolg.
- Nachweise zu Secrets und Authentifizierungsprofilen erfassen nur die Provider-/Quellenlage und
  SecretRef-Metadaten, niemals Rohwerte. Policy liest oder attestiert
  keine agentenspezifischen Anmeldedatenspeicher wie `auth-profiles.json`.
- Nachweise zur Datenverarbeitung stellen lediglich die Sicherheitslage auf Konfigurationsebene dar (Schwärzungsmodus,
  Umschalter für Telemetrieerfassung, Sitzungswartungsmodus, Einstellung zur Transkriptindizierung).
  Sie untersuchen keine Protokolle, Telemetrieexporte, Transkripte oder
  Speicherdateien, und ein einwandfreies Ergebnis beweist nicht, dass darin keine personenbezogenen Daten oder
  Secrets vorhanden sind.
- Routing-Sonden verwenden den Laufzeit-Bindungsresolver von OpenClaw erneut. Routing-Nachweise
  erfassen nur die Sonden-ID, den aufgelösten Agenten, die Übereinstimmungsart und geschwärzte Bindungsmetadaten.
  Sie erfassen niemals Kennungen von Peers, Konten, Guilds, Teams oder Rollen.
  Das Hinzufügen eines Routing-Abschnitts ändert bewusst die Policy- und Attestierungs-
  Hashes; Policies ohne Routing behalten ihre bestehende Nachweisstruktur bei.

### Referenz der Policy-Regeln

Jede nachstehende Regel ist optional; eine Prüfung wird nur ausgeführt, wenn die Regel vorhanden ist. Der
beobachtete Zustand entspricht der bestehenden OpenClaw-Konfiguration oder den Workspace-Metadaten.

#### Bereichsbezogene Overlays

Verwenden Sie `scopes.<scopeName>`, wenn bestimmte Agenten oder Kanäle eine strengere Policy
als die Baseline der obersten Ebene benötigen. Der Bereichsname ist lediglich eine Bezeichnung; die Übereinstimmung verwendet den
Selektor innerhalb des Bereichs. Overlays sind additiv: Die globale Regel wird weiterhin ausgeführt,
und die bereichsbezogene Regel kann für denselben Nachweis einen eigenen Befund hinzufügen.

| Selektor     | Unterstützte Abschnitte                                                             | Verwendungszweck                                          |
| ------------ | ------------------------------------------------------------------------------ | ------------------------------------------------- |
| `agentIds`   | `tools`, `agents.workspace`, `sandbox`, `dataHandling.memory`, `execApprovals` | Ein oder mehrere Laufzeitagenten benötigen strengere Regeln.   |
| `channelIds` | `ingress.channels`                                                             | Ein oder mehrere Kanäle benötigen strengere Ingress-Regeln. |

Wenn ein Eintrag `agentIds` nicht in `agents.entries.*` vorhanden ist, wertet OpenClaw
die bereichsbezogene Regel anhand der geerbten globalen/standardmäßigen Sicherheitslage für die ID dieses Laufzeitagenten
aus, anstatt sie zu überspringen.

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

Derselbe Agent kann in mehreren Bereichen vorkommen, wenn jeder Bereich ein anderes
Feld kontrolliert, wie oben gezeigt. Ein wiederholtes bereichsbezogenes Feld für denselben Agenten muss gleich oder
stärker einschränkend sein; eine schwächere doppelte Vorgabe wird abgelehnt (Positivlisten sind
Teilmengen, Sperrlisten sind Obermengen, erforderliche boolesche Werte sind festgelegt).

Regeln zur Container-Sicherheitslage (`sandbox.containers.*`) werden nur anhand von
Nachweisen geprüft, die das Sandbox-Backend des übereinstimmenden Agenten bereitstellen kann. Wenn ein Backend
eine dafür aktivierte Regel nicht beobachten kann, meldet Policy
`policy/sandbox-container-posture-unobservable`, anstatt die Prüfung als bestanden zu werten; beschränken Sie
Container-Regeln auf die Agentengruppen, die ein Backend verwenden, das sie bereitstellen kann.

`ingress.session.requireDmScope` auf oberster Ebene bleibt global; `session.dmScope` ist
kein einem Kanal zuordenbarer Nachweis und kann daher nicht über `channelIds` eingeschränkt werden.

Jeder in `policy.jsonc` vorhandene Bereich muss gültig und durchsetzbar sein.

#### Kanäle

| Policy-Feld                         | Beobachteter Zustand                          | Verwendungszweck                                                     |
| ------------------------------------ | --------------------------------------- | ------------------------------------------------------------ |
| `channels.denyRules[].when.provider` | Provider und Aktivierungsstatus von `channels.*` | Konfigurierte Kanäle eines Providers wie `telegram` verweigern. |
| `channels.denyRules[].reason`        | Kontext der Befundmeldung und des Reparaturhinweises | Erläutern, warum der Provider verweigert wird.                          |

#### MCP-Server

| Policy-Feld        | Beobachteter Zustand      | Verwendungszweck                                                   |
| ------------------- | ------------------- | ---------------------------------------------------------- |
| `mcp.servers.allow` | IDs von `mcp.servers.*` | Fordern, dass jeder konfigurierte MCP-Server in einer Positivliste enthalten ist. |
| `mcp.servers.deny`  | IDs von `mcp.servers.*` | Bestimmte konfigurierte MCP-Server-IDs verweigern.                   |

#### Modell-Provider

| Policy-Feld             | Beobachteter Zustand                                   | Verwendungszweck                                                                        |
| ------------------------ | ------------------------------------------------ | ------------------------------------------------------------------------------- |
| `models.providers.allow` | IDs von `models.providers.*` und ausgewählte Modellreferenzen | Fordern, dass konfigurierte Provider und ausgewählte Modellreferenzen genehmigte Provider verwenden. |
| `models.providers.deny`  | IDs von `models.providers.*` und ausgewählte Modellreferenzen | Konfigurierte Provider und ausgewählte Modellreferenzen anhand der Provider-ID verweigern.               |

#### Netzwerk

| Richtlinienfeld                   | Beobachteter Zustand                      | Verwenden, wenn                                                           |
| ------------------------------ | ----------------------------------- | ------------------------------------------------------------------ |
| `network.privateNetwork.allow` | SSRF-Ausnahmen für private Netzwerke | Auf `false` setzen, damit der Zugriff auf private Netzwerke deaktiviert bleiben muss. |

#### Nachrichtenrouting

| Richtlinienfeld                        | Beobachteter Zustand                                      | Verwenden, wenn                                                               |
| ----------------------------------- | --------------------------------------------------- | ---------------------------------------------------------------------- |
| `routing.requireBindings`           | Kanalroutenbindungen, ausgenommen ACP-Bindungen      | Mindestens eine Nachrichtenrouting-Bindung verlangen.                          |
| `routing.requireConfiguredChannels` | Kanal-IDs der Bindungen und konfigurierte `channels.*`-IDs | Veraltete oder falsch geschriebene Kanal-IDs der Bindungen erkennen.                        |
| `routing.probes[].route`            | Der öffentliche OpenClaw-Routenauflöser                  | Eine repräsentative eingehende Route beschreiben, ohne eine Nachricht zu senden.     |
| `routing.probes[].expect.agentId`   | Aufgelöste Agenten-ID                                   | Verlangen, dass die Route den überprüften Agenten erreicht.                         |
| `routing.probes[].expect.matchedBy` | Übereinstimmungsart des Auflösers                                 | Die überprüfte Bindungsspezifität für Peer, Konto, Kanal oder eine andere Art verlangen. |

Prüf-IDs müssen eindeutig sein. Eine Route unterstützt `channel`, optional `accountId`,
`peer`, `parentPeer`, `guildId`, `teamId` und `memberRoleIds`. Peer-Arten sind
`direct`, `group` und `channel`. `matchedBy` kann eine oder mehrere Laufzeit-
Übereinstimmungsarten enthalten, darunter `binding.peer`, `binding.account`, `binding.channel`
oder `default`.

Routingprüfungen sind ausschließlich Konformitätsprüfungen. Sie ändern weder den Startvorgang
noch die Nachrichtenzustellung, die Bindungspriorität oder das Fallback-Verhalten. Befunde erfordern
eine Überprüfung durch den Betreiber, da die automatische Änderung einer Bindung
private Nachrichten umleiten könnte.

#### Eingangs- und Kanalzugriff

| Richtlinienfeld                              | Beobachteter Zustand                                                 | Verwenden, wenn                                                           |
| ----------------------------------------- | -------------------------------------------------------------- | ------------------------------------------------------------------ |
| `ingress.session.requireDmScope`          | `session.dmScope`                                              | Einen überprüften Isolationsbereich für Direktnachrichten verlangen.                 |
| `ingress.channels.allowDmPolicies`        | `channels.*.dmPolicy` und veraltete Kanalrichtlinienfelder für Direktnachrichten      | Nur überprüfte Kanalrichtlinien für Direktnachrichten zulassen.               |
| `ingress.channels.denyOpenGroups`         | Eingangsrichtlinie für Kanal, Konto und Gruppe                     | Offenen Gruppeneingang für konfigurierte Kanäle und Konten verweigern.      |
| `ingress.channels.requireMentionInGroups` | Konfiguration der Erwähnungsbeschränkung für Kanal, Konto, Gruppe, Guild und verschachtelte Ebenen | Erwähnungsbeschränkungen verlangen, wenn der Gruppeneingang offen oder durch Erwähnungen beschränkt ist. |

#### Gateway

| Richtlinienfeld                            | Beobachteter Zustand                                 | Verwenden, wenn                                                                             |
| --------------------------------------- | ---------------------------------------------- | ------------------------------------------------------------------------------------ |
| `gateway.exposure.allowNonLoopbackBind` | `gateway.bind`                                 | Auf `false` setzen, um eine Gateway-Bindung an die Loopback-Schnittstelle zu verlangen.                                  |
| `gateway.exposure.allowTailscaleFunnel` | Gateway-Sicherheitslage für Tailscale Serve/Funnel         | Auf `false` setzen, um eine Offenlegung über Tailscale Funnel zu verweigern.                                    |
| `gateway.auth.requireAuth`              | `gateway.auth.mode`                            | Auf `true` setzen, um deaktivierte Gateway-Authentifizierung abzulehnen.                                       |
| `gateway.auth.requireExplicitRateLimit` | `gateway.auth.rateLimit`                       | Auf `true` setzen, um eine explizite Konfiguration der Authentifizierungs-Ratenbegrenzung zu verlangen.                            |
| `gateway.controlUi.allowInsecure`       | Unsichere Authentifizierungs-, Geräte- oder Herkunftsumschalter der Control UI | Auf `false` setzen, um Umschalter für eine unsichere Offenlegung der Control UI zu verweigern.                         |
| `gateway.remote.allow`                  | Remote-Gateway-Modus/-Konfiguration                     | Auf `false` setzen, um den Remote-Gateway-Modus zu verweigern.                                          |
| `gateway.http.denyEndpoints`            | Gateway-HTTP-API-Endpunkte                     | Endpunkt-IDs wie `chatCompletions` oder `responses` verweigern.                          |
| `gateway.http.requireUrlAllowlists`     | URL-Abrufeingaben der Gateway-HTTP-API                  | Auf `true` setzen, um URL-Zulassungslisten für URL-Abrufeingaben zu verlangen.                         |
| `gateway.nodes.denyCommands`            | `gateway.nodes.commands.deny`                  | Verlangen, dass exakte Node-Befehls-IDs wie `system.run` in der OpenClaw-Konfiguration verweigert werden. |

`gateway.nodes.denyCommands` ist eine exakte, groß-/kleinschreibungssensitive Superset-Regel für Richtlinienverweigerungen.
Verwenden Sie sie, wenn die Richtlinie nachweisen muss, dass privilegierte Node-Befehle durch die
OpenClaw-Konfiguration ausdrücklich verweigert werden. Bei einer Bereitstellung, die einen privilegierten
Node-Befehl absichtlich zulässt, sollte `policy.jsonc` nach der Überprüfung aktualisiert werden, statt sich
allein auf `gateway.nodes.commands.allow` zu verlassen.

#### Agenten-Arbeitsbereich

| Richtlinienfeld                     | Beobachteter Zustand                                                                           | Verwenden, wenn                                                                                 |
| -------------------------------- | ---------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| `agents.workspace.allowedAccess` | `agents.defaults.sandbox.workspaceAccess` und `agents.entries.*.sandbox.workspaceAccess` | Nur Werte für den Sandbox-Arbeitsbereichszugriff wie `none` oder `ro` zulassen.                       |
| `agents.workspace.denyTools`     | Globale und agentenspezifische Konfiguration für Tool-Verweigerungen                                                    | Verlangen, dass Mutationstools (`exec`, `process`, `write`, `edit`, `apply_patch`) verweigert werden. |

#### Sandbox-Sicherheitslage

| Richtlinienfeld                                          | Beobachteter Zustand                                          | Verwenden, wenn                                                       |
| ----------------------------------------------------- | ------------------------------------------------------- | -------------------------------------------------------------- |
| `sandbox.requireMode`                                 | `agents.defaults.sandbox.mode` und agentenspezifischer Modus       | Nur überprüfte Sandbox-Modi wie `all` oder `non-main` zulassen. |
| `sandbox.allowBackends`                               | `agents.defaults.sandbox.backend` und agentenspezifisches Backend | Nur überprüfte Sandbox-Backends wie `docker` zulassen.         |
| `sandbox.containers.denyHostNetwork`                  | Netzwerkmodus der containergestützten Sandbox/des Browsers           | Host-Netzwerkmodus verweigern.                                        |
| `sandbox.containers.denyContainerNamespaceJoin`       | Netzwerkmodus der containergestützten Sandbox/des Browsers           | Den Beitritt zum Netzwerk-Namespace eines anderen Containers verweigern.              |
| `sandbox.containers.requireReadOnlyMounts`            | Einbindungsmodus der containergestützten Sandbox/des Browsers             | Schreibgeschützte Einbindungen verlangen.                                |
| `sandbox.containers.denyContainerRuntimeSocketMounts` | Einbindungsziele der containergestützten Sandbox/des Browsers          | Einbindungen von Container-Laufzeit-Sockets verweigern.                          |
| `sandbox.containers.denyUnconfinedProfiles`           | Sicherheitslage des Container-Sicherheitsprofils                      | Uneingeschränkte Container-Sicherheitsprofile verweigern.                   |
| `sandbox.browser.requireCdpSourceRange`               | CDP-Quellbereich des Sandbox-Browsers                        | Verlangen, dass die Browser-CDP-Offenlegung einen Quellbereich deklariert.        |

Die Richtlinie behandelt ein fehlendes `sandbox.mode` als dessen impliziten Standardwert `off`, sodass
`sandbox.requireMode` eine neue oder nicht konfigurierte Sandbox als außerhalb einer
Zulassungsliste wie `["all"]` meldet.

#### Datenverarbeitung

| Richtlinienfeld                                        | Beobachteter Zustand                                                                                     | Verwenden, wenn                                                               |
| --------------------------------------------------- | -------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| `dataHandling.sensitiveLogging.requireRedaction`    | `logging.redactSensitive`                                                                          | Auf `true` setzen, um `logging.redactSensitive: "off"` abzulehnen.              |
| `dataHandling.telemetry.denyContentCapture`         | `diagnostics.otel.captureContent`                                                                  | Auf `true` setzen, um die Erfassung von Telemetrieinhalten abzulehnen.                     |
| `dataHandling.retention.requireSessionMaintenance`  | `session.maintenance.mode`                                                                         | Auf `true` setzen, um den effektiven Sitzungswartungsmodus `enforce` zu verlangen. |
| `dataHandling.memory.denySessionTranscriptIndexing` | `memory.qmd.sessions.enabled`, `memory.search.experimental.sessionMemory` und agentenspezifische Überschreibungen | Auf `true` setzen, um die Indizierung von Sitzungstranskripten im Speicher abzulehnen.       |

#### Geheimnisse

| Richtlinienfeld                      | Beobachteter Zustand                                           | Verwenden, wenn                                                                |
| --------------------------------- | -------------------------------------------------------- | ----------------------------------------------------------------------- |
| `secrets.requireManagedProviders` | SecretRefs der Konfiguration und `secrets.providers.*`-Deklarationen | Auf `true` setzen, damit SecretRefs auf deklarierte Provider verweisen müssen.     |
| `secrets.denySources`             | Quellen von Geheimnis-Providern und SecretRef-Quellen            | Quellen wie `exec`, `file` oder einen anderen konfigurierten Quellnamen verweigern. |
| `secrets.allowInsecureProviders`  | Flags für die unsichere Sicherheitslage von Geheimnis-Providern                   | Auf `false` setzen, um Provider abzulehnen, die eine unsichere Sicherheitslage aktivieren.      |

#### Ausführungsgenehmigungen

Prüfungen der Ausführungsgenehmigungen lesen das Laufzeitartefakt `exec-approvals.json`:
standardmäßig `~/.openclaw/exec-approvals.json` oder
`$OPENCLAW_STATE_DIR/exec-approvals.json`, wenn `OPENCLAW_STATE_DIR` gesetzt ist.
Sicherheitslageregeln unter `execApprovals.defaults.*` oder `execApprovals.agents.*`
verlangen lesbare Artefaktnachweise; ein fehlendes oder ungültiges Artefakt wird
als nicht beobachtbarer Nachweis gemeldet und nicht nach bestem Bemühen als bestanden gewertet. Sobald das Artefakt lesbar ist,
übernehmen ausgelassene Felder die Laufzeitstandardwerte: Ein fehlendes `defaults.security` entspricht `full`, und
eine fehlende Agentensicherheit übernimmt diesen Standardwert. Der Nachweis umfasst `defaults`,
`agents.*`, `agents.*.allowlist[].pattern`, optional `argPattern`, die effektive
`autoAllowSkills`-Sicherheitslage und die Eintragsquelle – niemals Socket-Pfad/-Token,
`commandText`, `lastUsedCommand`, aufgelöste Pfade oder Zeitstempel.

| Richtlinienfeld                                | Beobachteter Zustand                                                                         | Verwenden, wenn                                                                                |
| ------------------------------------------- | -------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| `execApprovals.requireFile`                 | Aktiver Laufzeitpfad `exec-approvals.json`                                              | Auf `true` setzen, um zu verlangen, dass das Genehmigungsartefakt vorhanden ist und geparst werden kann.                     |
| `execApprovals.defaults.allowSecurity`      | `defaults.security`, standardmäßig `full`                                              | Nur genehmigte standardmäßige Sicherheitsmodi für Genehmigungen zulassen.                                    |
| `execApprovals.agents.allowSecurity`        | `agents.*.security`, wobei Standardwerte übernommen werden                                               | Nur genehmigte effektive Sicherheitsmodi für Genehmigungen pro Agent zulassen.                        |
| `execApprovals.agents.allowAutoAllowSkills` | `defaults.autoAllowSkills` und `agents.*.autoAllowSkills`, wobei Laufzeitstandardwerte übernommen werden | Auf `false` setzen, um strikte manuelle Zulassungslisten ohne implizite Genehmigung der Skill-CLI zu verlangen. |
| `execApprovals.agents.allowlist.expected`   | Aggregiertes `agents.*.allowlist[]`-Muster und optionale argPattern-Einträge               | Verlangen, dass die Genehmigungs-Zulassungsliste mit dem geprüften Mustersatz übereinstimmt.                      |

Beispiel: Das Genehmigungsartefakt verlangen, permissive Standardwerte ablehnen und
nur die geprüfte Ausführungs-Genehmigungshaltung für ausgewählte Agenten zulassen.

```jsonc
{
  "execApprovals": {
    "requireFile": true,
    "defaults": {
      // Sicherheitsmodi: "deny", "allowlist" oder "full".
      // Dieser Standardwert erlaubt nur die strikt eingeschränkte Ablehnungshaltung.
      "allowSecurity": ["deny"],
    },
  },
  "scopes": {
    "restricted-shell": {
      "agentIds": ["family-agent", "groups-agent"],
      "execApprovals": {
        "agents": {
          // Ausgewählte Agenten dürfen die geprüfte Zulassungslistenhaltung verwenden, aber nicht "full".
          "allowSecurity": ["allowlist"],
          // false bedeutet, dass Skill-CLIs in der geprüften Zulassungsliste aufgeführt sein müssen, statt
          // durch autoAllowSkills implizit genehmigt zu werden.
          "allowAutoAllowSkills": false,
          "allowlist": {
            "expected": [
              // Einfacher Eintrag: exakt geprüftes Muster für ausführbare Dateien ohne argPattern.
              "travel-hub",
              // Eingeschränkter Eintrag: Muster plus geprüfter regulärer Ausdruck für Argumente.
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

#### Authentifizierungsprofile

| Richtlinienfeld                    | Beobachteter Zustand                               | Verwenden, wenn                                                                                   |
| ------------------------------- | -------------------------------------------- | ------------------------------------------------------------------------------------------ |
| `auth.profiles.requireMetadata` | Provider- und Modusmetadaten von `auth.profiles.*` | Metadatenschlüssel wie `provider` und `mode` für Authentifizierungsprofile in der Konfiguration verlangen.               |
| `auth.profiles.allowModes`      | `auth.profiles.*.mode`                       | Nur unterstützte Modi für Authentifizierungsprofile wie `api_key`, `aws-sdk`, `oauth` oder `token` zulassen. |

#### Tool-Metadaten

| Richtlinienfeld            | Beobachteter Zustand                   | Verwenden, wenn                                                                                   |
| ----------------------- | -------------------------------- | ------------------------------------------------------------------------------------------ |
| `tools.requireMetadata` | Verwaltete `TOOLS.md`-Deklarationen | Verlangen, dass verwaltete Tools Metadatenschlüssel wie `risk`, `sensitivity` oder `owner` deklarieren. |

#### Tool-Haltung

| Richtlinienfeld                    | Beobachteter Zustand                                              | Verwenden, wenn                                                                                                 |
| ------------------------------- | ----------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `tools.profiles.allow`          | `tools.profile` und `agents.entries.*.tools.profile`        | Nur Tool-Profil-IDs wie `minimal`, `messaging` oder `coding` zulassen.                                 |
| `tools.fs.requireWorkspaceOnly` | `tools.fs.workspaceOnly` und agentenspezifische `tools.fs`-Überschreibungen | Auf `true` setzen, um eine auf den Arbeitsbereich beschränkte Haltung der Dateisystem-Tools zu verlangen.                                         |
| `tools.exec.allowSecurity`      | `tools.exec.security` und agentenspezifische Ausführungssicherheit           | Nur Ausführungssicherheitsmodi wie `deny` oder `allowlist` zulassen.                                            |
| `tools.exec.requireAsk`         | `tools.exec.ask` und agentenspezifischer Fragemodus für Ausführungen                | Eine Genehmigungshaltung wie `always` verlangen.                                                               |
| `tools.exec.allowHosts`         | `tools.exec.host` und agentenspezifisches Host-Routing für Ausführungen           | Nur Host-Routing-Modi für Ausführungen wie `sandbox` zulassen.                                                    |
| `tools.elevated.allow`          | `tools.elevated.enabled` und agentenspezifische privilegierte Haltung     | Auf `false` setzen, um zu verlangen, dass der privilegierte Tool-Modus deaktiviert bleibt.                                           |
| `tools.alsoAllow.expected`      | `tools.alsoAllow` und agentenspezifisches `tools.alsoAllow`           | Exakte `alsoAllow`-Einträge verlangen und fehlende oder unerwartete zusätzliche Tool-Berechtigungen melden.                 |
| `tools.denyTools`               | `tools.deny` und `agents.entries.*.tools.deny`              | Verlangen, dass konfigurierte Tool-Sperrlisten Tool-IDs oder Gruppen wie `group:runtime` und `group:fs` enthalten. |

## Prüfungen ausführen

Während der Erstellung ausschließlich Richtlinienprüfungen ausführen:

```bash
openclaw policy check
openclaw policy check --json
openclaw policy check --severity-min error
```

`policy check` führt nur den Satz von Richtlinienprüfungen aus und gibt Nachweise, Befunde
und Attestierungshashes aus. Dieselben Befunde erscheinen auch in
`openclaw doctor --lint`, wenn das Richtlinien-Plugin aktiviert ist.

Eine Betreiber-Richtliniendatei mit einer erstellten Baseline vergleichen:

```bash
openclaw policy compare --baseline official.policy.jsonc
openclaw policy compare --baseline official.policy.jsonc --policy policy.jsonc --json
```

`policy compare` prüft die Syntax der Richtliniendatei gegen die Syntax der Richtliniendatei; dabei
werden weder Laufzeitzustand noch Nachweise, Anmeldedaten oder Geheimnisse untersucht. Es werden dieselben
Regelmetadaten verwendet, die bereichsspezifische Überlagerungen steuern: Zulassungslisten müssen gleich bleiben oder
enger werden, Sperrlisten müssen gleich bleiben oder breiter werden, erforderliche boolesche Werte müssen
ihren Wert beibehalten, geordnete Zeichenfolgen dürfen sich nur zum strengeren Ende der
konfigurierten Reihenfolge bewegen und exakte Listen müssen übereinstimmen. Die Baseline kann eine
von der Organisation erstellte Richtlinie sein; die geprüfte Richtlinie darf strengere Werte oder
zusätzliche Regeln hinzufügen. Eine geprüfte Regel auf oberster Ebene kann eine bereichsspezifische Baseline-Regel erfüllen, wenn
sie gleich oder stärker einschränkend ist. Bereichsnamen müssen zwischen den
Dateien nicht übereinstimmen; der Vergleich erfolgt anhand von Selektor (`agentIds`/`channelIds`) und Feld.
Bei Routing-Prüfungen muss jede Baseline-Prüfungs-ID mit derselben Route
und demselben erwarteten Agenten bestehen bleiben. Eine geprüfte Richtlinie darf Prüfungen hinzufügen oder `matchedBy` einschränken, aber
das Entfernen einer Prüfung, das Ändern ihrer Route oder ihres Agenten oder das Erweitern ihrer akzeptierten Übereinstimmungsarten
ist schwächer.

Erfolgreicher Vergleich (`--json`):

```json
{
  "ok": true,
  "baselinePath": "official.policy.jsonc",
  "policyPath": "policy.jsonc",
  "rulesChecked": 3,
  "findings": []
}
```

Eine erfolgreiche `policy check --json`-Ausgabe enthält stabile Hashes, die ein Betreiber oder
eine Aufsichtsperson aufzeichnen kann:

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

## Richtlinie konfigurieren

Die Richtlinienkonfiguration befindet sich unter `plugins.entries.policy.config`.

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

| Einstellung                   | Zweck                                                         |
| ------------------------- | --------------------------------------------------------------- |
| `enabled`                 | Richtlinienprüfungen aktivieren, noch bevor `policy.jsonc` vorhanden ist.         |
| `workspaceRepairs`        | `doctor --fix` erlauben, richtlinienverwaltete Arbeitsbereichseinstellungen zu bearbeiten. |
| `expectedHash`            | Optionale Hash-Sperre für das genehmigte Richtlinienartefakt.            |
| `expectedAttestationHash` | Optionale Hash-Sperre für die letzte akzeptierte erfolgreiche Richtlinienprüfung.    |
| `path`                    | Arbeitsbereichsrelativer Speicherort des Richtlinienartefakts.             |

`plugins.entries.policy.config.enabled` auf `false` setzen, um Richtlinienprüfungen
für einen Arbeitsbereich zu deaktivieren, während das Plugin installiert bleibt.

## Richtlinienzustand akzeptieren

Beispielhafte JSON-Ausgabe:

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

`attestation.policy.hash` identifiziert das erstellte Regelartefakt. `evidence`
zeichnet den beobachteten OpenClaw-Zustand auf, den die Prüfungen verwenden, und
`workspace.hash` identifiziert diese Evidenznutzlast. `findingsHash` identifiziert
den exakten Befundsatz. `checkedAt` zeichnet auf, wann die Prüfung ausgeführt wurde.
`attestationHash` identifiziert die stabile Aussage (Richtlinien-Hash, Evidenz-Hash,
Befund-Hash und sauberer/geänderter Zustand) und schließt `checkedAt` bewusst aus,
sodass derselbe Richtlinienzustand stets denselben Attestierungs-Hash erzeugt. Zusammen
bilden diese vier Werte das Audit-Tupel für eine Richtlinienprüfung.

Wenn ein Gateway oder Supervisor Richtlinien verwendet, um eine
Laufzeitaktion zu blockieren, zu genehmigen oder mit Anmerkungen zu versehen,
sollte er den Attestierungs-Hash der letzten sauberen Prüfung aufzeichnen.
`checkedAt` verbleibt für Audit-Protokolle in der JSON-Ausgabe, ist jedoch kein
Bestandteil des stabilen Hashs.

Lebenszyklus zum Akzeptieren des Richtlinienzustands:

1. Erstellen oder prüfen Sie `policy.jsonc`.
2. Führen Sie `openclaw policy check --json` aus.
3. Wenn die Prüfung sauber ist, zeichnen Sie `attestation.policy.hash` als `expectedHash` auf.
4. Zeichnen Sie `attestation.attestationHash` als `expectedAttestationHash` auf.
5. Führen Sie `openclaw doctor --lint` in der CI oder in Release-Gates erneut aus.

Wenn Richtlinienregeln absichtlich geändert werden, aktualisieren Sie beide
akzeptierten Hashs anhand einer sauberen Prüfung. Wenn sich nur die
Workspace-Einstellungen ändern (die Richtlinie bleibt unverändert), ändert
sich normalerweise nur `expectedAttestationHash`.

Das Aktivieren oder Aktualisieren von `agents.workspace`-Regeln fügt
`agentWorkspace`-Evidenz zum Workspace-Hash und zum Attestierungs-Hash hinzu;
prüfen Sie die neue Evidenz und aktualisieren Sie nach der Aktivierung die
akzeptierten Attestierungs-Hashes. Das Aktivieren oder Aktualisieren von Regeln
für die Tool-Sicherheitskonfiguration fügt auf dieselbe Weise
`toolPosture`-Evidenz hinzu.

`openclaw policy watch` führt die Prüfung erneut aus und meldet, wenn die aktuelle
Evidenz nicht mehr mit `expectedAttestationHash` übereinstimmt:

```bash
openclaw policy watch --json
```

Verwenden Sie `--once` in der CI oder in Skripten, die eine einmalige
Driftbewertung benötigen. Ohne `--once` wird standardmäßig alle zwei
Sekunden abgefragt; verwenden Sie `--interval-ms`, um das Intervall zu ändern.

## Befunde

| Prüf-ID                                                 | Befund                                                                           |
| -------------------------------------------------------- | --------------------------------------------------------------------------------- |
| `policy/policy-jsonc-missing`                            | Die Richtlinie ist aktiviert, aber `policy.jsonc` fehlt.                                  |
| `policy/policy-jsonc-invalid`                            | Die Richtlinie kann nicht geparst werden oder enthält fehlerhafte Regeleinträge.                       |
| `policy/policy-hash-mismatch`                            | Die Richtlinie stimmt nicht mit dem konfigurierten `expectedHash` überein.                                  |
| `policy/attestation-hash-mismatch`                       | Die aktuellen Richtliniennachweise stimmen nicht mehr mit der akzeptierten Attestierung überein.               |
| `policy/policy-conformance-invalid`                      | Eine Baseline- oder geprüfte Richtliniendatei weist eine ungültige Vergleichssyntax auf.                  |
| `policy/policy-conformance-missing`                      | In einer geprüften Richtliniendatei fehlt eine von der Baseline-Richtliniendatei vorgeschriebene Regel.     |
| `policy/policy-conformance-weaker`                       | Eine geprüfte Richtliniendatei enthält einen schwächeren Wert als die Baseline-Richtliniendatei.           |
| `policy/channels-denied-provider`                        | Ein aktivierter Kanal entspricht einer Kanal-Ablehnungsregel.                                   |
| `policy/mcp-denied-server`                               | Ein konfigurierter MCP-Server wird von der Richtlinie abgelehnt.                                      |
| `policy/mcp-unapproved-server`                           | Ein konfigurierter MCP-Server befindet sich außerhalb der Zulassungsliste.                                 |
| `policy/models-denied-provider`                          | Ein konfigurierter Modell-Provider oder eine Modellreferenz verwendet einen abgelehnten Provider.                  |
| `policy/models-unapproved-provider`                      | Ein konfigurierter Modell-Provider oder eine Modellreferenz befindet sich außerhalb der Zulassungsliste.                |
| `policy/network-private-access-enabled`                  | Eine SSRF-Ausnahmeregelung für private Netzwerke ist aktiviert, obwohl die Richtlinie sie untersagt.             |
| `policy/routing-bindings-required`                       | Die Richtlinie verlangt eine Kanalroutenbindung, es ist jedoch keine konfiguriert.                  |
| `policy/routing-binding-channel-unconfigured`            | Eine Routenbindung nennt einen Kanal, der in `channels.*` nicht vorhanden ist.                         |
| `policy/routing-agent-mismatch`                          | Eine definierte Route wird einem anderen Agenten zugeordnet.                                  |
| `policy/routing-match-kind-mismatch`                     | Eine definierte Route stimmt mit einer unerwarteten Bindungsspezifität überein.                   |
| `policy/ingress-dm-policy-unapproved`                    | Eine Kanal-DM-Richtlinie befindet sich außerhalb der Richtlinien-Zulassungsliste.                              |
| `policy/ingress-dm-scope-unapproved`                     | `session.dmScope` stimmt nicht mit dem von der Richtlinie vorgeschriebenen DM-Isolationsbereich überein.          |
| `policy/ingress-open-groups-denied`                      | Eine Kanalgruppenrichtlinie ist auf `open` gesetzt, obwohl die Richtlinie offenen Gruppeneingang untersagt.          |
| `policy/ingress-group-mention-required`                  | Ein Kanal- oder Gruppeneintrag deaktiviert Erwähnungsschranken, obwohl die Richtlinie sie vorschreibt.       |
| `policy/gateway-non-loopback-bind`                       | Die Gateway-Bindungskonfiguration erlaubt eine Offenlegung außerhalb von Loopback, obwohl die Richtlinie sie untersagt.         |
| `policy/gateway-auth-disabled`                           | Die Gateway-Authentifizierung ist deaktiviert, obwohl die Richtlinie eine Authentifizierung vorschreibt.                     |
| `policy/gateway-rate-limit-missing`                      | Die Konfiguration der Gateway-Authentifizierungs-Ratenbegrenzung ist nicht explizit, obwohl die Richtlinie dies vorschreibt.          |
| `policy/gateway-control-ui-insecure`                     | Schalter für eine unsichere Offenlegung der Gateway Control UI sind aktiviert.                         |
| `policy/gateway-tailscale-funnel`                        | Die Offenlegung des Gateway über Tailscale Funnel ist aktiviert, obwohl die Richtlinie sie untersagt.               |
| `policy/gateway-remote-enabled`                          | Der Gateway-Remote-Modus ist aktiv, obwohl die Richtlinie ihn untersagt.                              |
| `policy/gateway-http-endpoint-enabled`                   | Ein Gateway-HTTP-API-Endpunkt ist aktiviert, obwohl er von der Richtlinie untersagt wird.                    |
| `policy/gateway-http-url-fetch-unrestricted`             | Für die Eingabe zum Abrufen von URLs über Gateway HTTP fehlt eine erforderliche URL-Zulassungsliste.                      |
| `policy/gateway-node-command-denied`                     | Ein von der Richtlinie abgelehnter Node-Befehl wird von der OpenClaw-Konfiguration nicht abgelehnt.                 |
| `policy/agents-workspace-access-denied`                  | Der Agent-Sandbox-Modus oder der Arbeitsbereichszugriff befindet sich außerhalb der Richtlinien-Zulassungsliste.           |
| `policy/agents-tool-not-denied`                          | Eine Agenten- oder Standardkonfiguration lehnt ein von der Richtlinie vorgeschriebenes Tool nicht ab.               |
| `policy/tools-profile-unapproved`                        | Ein konfiguriertes globales oder agentenspezifisches Tool-Profil befindet sich außerhalb der Zulassungsliste.           |
| `policy/tools-fs-workspace-only-required`                | Dateisystem-Tools sind nicht für eine ausschließlich auf den Arbeitsbereich beschränkte Pfadkonfiguration eingerichtet.             |
| `policy/tools-exec-security-unapproved`                  | Der Exec-Sicherheitsmodus befindet sich außerhalb der Richtlinien-Zulassungsliste.                               |
| `policy/tools-exec-ask-unapproved`                       | Der Exec-Abfragemodus befindet sich außerhalb der Richtlinien-Zulassungsliste.                                    |
| `policy/tools-exec-host-unapproved`                      | Das Exec-Host-Routing befindet sich außerhalb der Richtlinien-Zulassungsliste.                                |
| `policy/tools-elevated-enabled`                          | Der Modus für privilegierte Tools ist aktiviert, obwohl die Richtlinie ihn untersagt.                              |
| `policy/tools-also-allow-missing`                        | In einer konfigurierten `alsoAllow`-Liste fehlt ein von der Richtlinie vorgeschriebener Eintrag.             |
| `policy/tools-also-allow-unexpected`                     | Eine konfigurierte `alsoAllow`-Liste enthält einen von der Richtlinie nicht erwarteten Eintrag.           |
| `policy/tools-required-deny-missing`                     | Eine globale oder agentenspezifische Tool-Ablehnungsliste enthält ein vorgeschriebenes abgelehntes Tool nicht.     |
| `policy/sandbox-mode-unapproved`                         | Der Sandbox-Modus befindet sich außerhalb der Richtlinien-Zulassungsliste.                                     |
| `policy/sandbox-backend-unapproved`                      | Das Sandbox-Backend befindet sich außerhalb der Richtlinien-Zulassungsliste.                                  |
| `policy/sandbox-container-posture-unobservable`          | Eine Container-Konfigurationsregel ist für ein Backend aktiviert, das sie nicht beobachten kann.         |
| `policy/sandbox-container-host-network-denied`           | Eine containerbasierte Sandbox oder ein containerbasierter Browser verwendet den Host-Netzwerkmodus.                     |
| `policy/sandbox-container-namespace-join-denied`         | Eine containerbasierte Sandbox oder ein containerbasierter Browser tritt dem Namespace eines anderen Containers bei.          |
| `policy/sandbox-container-mount-mode-required`           | Ein Mount einer containerbasierten Sandbox oder eines containerbasierten Browsers ist nicht schreibgeschützt.                     |
| `policy/sandbox-container-runtime-socket-mount`          | Ein Mount einer containerbasierten Sandbox oder eines containerbasierten Browsers legt den Socket der Container-Runtime offen. |
| `policy/sandbox-container-unconfined-profile`            | Das Container-Sandbox-Profil ist uneingeschränkt, obwohl die Richtlinie dies untersagt.                    |
| `policy/sandbox-browser-cdp-source-range-missing`        | Der CDP-Quellbereich des Sandbox-Browsers fehlt, obwohl die Richtlinie einen solchen vorschreibt.             |
| `policy/data-handling-redaction-disabled`                | Die Schwärzung sensibler Protokolldaten ist deaktiviert, obwohl die Richtlinie sie vorschreibt.                  |
| `policy/data-handling-telemetry-content-capture`         | Die Erfassung von Telemetrieinhalten ist aktiviert, obwohl die Richtlinie sie untersagt.                       |
| `policy/data-handling-session-retention-not-enforced`    | Die Wartung der Sitzungsaufbewahrung wird nicht erzwungen, obwohl die Richtlinie sie vorschreibt.            |
| `policy/data-handling-session-transcript-memory-enabled` | Die Speicherindizierung von Sitzungstranskripten ist aktiviert, obwohl die Richtlinie sie untersagt.              |
| `policy/secrets-unmanaged-provider`                      | Eine SecretRef in der Konfiguration verweist auf einen Provider, der nicht unter `secrets.providers` deklariert ist.  |
| `policy/secrets-denied-provider-source`                  | Ein Secret-Provider oder eine SecretRef in der Konfiguration verwendet eine von der Richtlinie abgelehnte Quelle.             |
| `policy/secrets-insecure-provider`                       | Ein Secret-Provider aktiviert eine unsichere Konfiguration, obwohl die Richtlinie sie untersagt.               |
| `policy/auth-profile-invalid-metadata`                   | In einem Authentifizierungsprofil der Konfiguration fehlen gültige Provider- oder Modusmetadaten.                 |
| `policy/auth-profile-unapproved-mode`                    | Der Modus eines Authentifizierungsprofils der Konfiguration befindet sich außerhalb der Richtlinien-Zulassungsliste.                       |
| `policy/exec-approvals-missing`                          | Die Richtlinie verlangt `exec-approvals.json`, aber das Artefakt fehlt.               |
| `policy/exec-approvals-invalid`                          | Das konfigurierte Artefakt für Exec-Genehmigungen kann nicht geparst werden.                          |
| `policy/exec-approvals-default-security-unapproved`      | Die Standardwerte für Exec-Genehmigungen verwenden einen Sicherheitsmodus außerhalb der Richtlinien-Zulassungsliste.          |
| `policy/exec-approvals-agent-security-unapproved`        | Ein agentenspezifischer effektiver Sicherheitsmodus für Exec-Genehmigungen befindet sich außerhalb der Zulassungsliste.       |
| `policy/exec-approvals-auto-allow-skills-enabled`        | Ein Agent für Exec-Genehmigungen lässt Skills-CLIs implizit automatisch zu, obwohl die Richtlinie dies untersagt.   |
| `policy/exec-approvals-allowlist-missing`                | In der Genehmigungs-Zulassungsliste fehlt ein von der Richtlinie vorgeschriebenes Muster.                  |
| `policy/exec-approvals-allowlist-unexpected`             | Die Genehmigungs-Zulassungsliste enthält ein von der Richtlinie nicht erwartetes Muster.                |
| `policy/tools-missing-risk-level`                        | In einer verwalteten Tool-Deklaration fehlen Risikometadaten.                             |
| `policy/tools-unknown-risk-level`                        | Eine verwaltete Tool-Deklaration verwendet einen unbekannten Risikowert.                           |
| `policy/tools-missing-sensitivity-token`                 | In einer verwalteten Tool-Deklaration fehlen Vertraulichkeitsmetadaten.                      |
| `policy/tools-missing-owner`                             | In einer verwalteten Tool-Deklaration fehlen Eigentümermetadaten.                            |
| `policy/tools-unknown-sensitivity-token`                 | Eine verwaltete Tool-Deklaration verwendet einen unbekannten Vertraulichkeitswert.                    |

Ein Befund kann sowohl `target` (das beobachtete Element im Arbeitsbereich, das
nicht konform ist) als auch `requirement` (die definierte Regel, durch die es zu einem Befund wurde) enthalten.
Beide sind derzeit `oc://`-Adresszeichenfolgen, die Feldnamen beschreiben jedoch die Rolle in der Richtlinie
und nicht das Adressformat.

Beispielbefunde:

```json
{
  "checkId": "policy/channels-denied-provider",
  "severity": "error",
  "message": "Kanal 'telegram' verwendet den abgelehnten Provider 'telegram'.",
  "source": "policy",
  "path": "openclaw config",
  "ocPath": "oc://openclaw.config/channels/telegram",
  "target": "oc://openclaw.config/channels/telegram",
  "requirement": "oc://policy.jsonc/channels/denyRules/#0",
  "fixHint": "Telegram ist für diesen Arbeitsbereich nicht genehmigt."
}
```

```json
{
  "checkId": "policy/tools-missing-risk-level",
  "severity": "error",
  "message": "Das Tool 'deploy' in TOOLS.md hat keine explizite Risikoklassifizierung.",
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
  "message": "Der MCP-Server 'remote' befindet sich nicht in der Richtlinien-Zulassungsliste.",
  "source": "policy",
  "path": "openclaw config",
  "ocPath": "oc://openclaw.config/mcp/servers/remote",
  "target": "oc://openclaw.config/mcp/servers/remote",
  "requirement": "oc://policy.jsonc/mcp/servers/allow"
}
```

```json
{
  "checkId": "policy/models-unapproved-provider",
  "severity": "error",
  "message": "Die Modellreferenz 'anthropic/claude-sonnet-4.7' verwendet den nicht genehmigten Provider 'anthropic'.",
  "source": "policy",
  "path": "openclaw config",
  "ocPath": "oc://openclaw.config/agents/defaults/model/fallbacks/#0",
  "target": "oc://openclaw.config/agents/defaults/model/fallbacks/#0",
  "requirement": "oc://policy.jsonc/models/providers/allow"
}
```

```json
{
  "checkId": "policy/network-private-access-enabled",
  "severity": "error",
  "message": "Die Netzwerkeinstellung 'browser-private-network' erlaubt den Zugriff auf private Netzwerke.",
  "source": "policy",
  "path": "openclaw config",
  "ocPath": "oc://openclaw.config/browser/ssrfPolicy/dangerouslyAllowPrivateNetwork",
  "target": "oc://openclaw.config/browser/ssrfPolicy/dangerouslyAllowPrivateNetwork",
  "requirement": "oc://policy.jsonc/network/privateNetwork/allow"
}
```

```json
{
  "checkId": "policy/gateway-non-loopback-bind",
  "severity": "error",
  "message": "Die Gateway-Bindungseinstellung 'gateway-bind' ermöglicht eine Exposition außerhalb der Loopback-Schnittstelle.",
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
  "message": "Der Gateway-Node-Befehl 'system.run' wird durch die Richtlinie verweigert, aber nicht durch die OpenClaw-Konfiguration.",
  "source": "policy",
  "path": "openclaw config",
  "ocPath": "oc://openclaw.config/gateway/nodes/commands/deny",
  "target": "oc://openclaw.config/gateway/nodes/commands/deny",
  "requirement": "oc://policy.jsonc/gateway/nodes/denyCommands",
  "fixHint": "Fügen Sie 'system.run' zu gateway.nodes.commands.deny hinzu oder aktualisieren Sie die Richtlinie nach der Überprüfung."
}
```

```json
{
  "checkId": "policy/agents-workspace-access-denied",
  "severity": "error",
  "message": "Der Sandbox-Arbeitsbereichszugriff 'rw' unter agents.defaults ist laut Richtlinie nicht zulässig.",
  "source": "policy",
  "path": "openclaw config",
  "ocPath": "oc://openclaw.config/agents/defaults/sandbox/workspaceAccess",
  "target": "oc://openclaw.config/agents/defaults/sandbox/workspaceAccess",
  "requirement": "oc://policy.jsonc/agents/workspace/allowedAccess"
}
```

## Reparatur

`doctor --lint` und `policy check` sind schreibgeschützt.

`doctor --fix` bearbeitet richtlinienverwaltete Arbeitsbereichseinstellungen nur, wenn
`workspaceRepairs` ausdrücklich aktiviert ist; andernfalls melden die Prüfungen, was sie
reparieren würden, und lassen die Einstellungen unverändert.

In dieser Version kann die Reparatur durch `channels.denyRules` verweigerte Kanäle deaktivieren und
die unten aufgeführten automatischen Einschränkungsreparaturen anwenden. Aktivieren Sie `workspaceRepairs`
erst, nachdem die Richtliniendatei überprüft wurde, da eine gültige Regel die
Arbeitsbereichskonfiguration ändern kann:

- `tools.elevated.enabled=false` festlegen, wenn eine globale Richtlinie erweiterte Tools verbietet
- fehlende Tool-IDs für obligatorische Verweigerungen zu `tools.deny` oder
  `agents.entries.*.tools.deny` hinzufügen, wenn die Richtlinie die Verweigerung dieser Tools verlangt
- unsichere `gateway.controlUi.*`-Umschalter auf `false` setzen
- `gateway.mode=local` festlegen, wenn die Richtlinie den Remote-Gateway-Modus verweigert
- gemeldete `gateway.http.endpoints.*.enabled`-Pfade auf `false` setzen, wenn die Richtlinie
  Gateway-HTTP-API-Endpunkte verweigert
- gemeldete `groupPolicy`-Pfade für eingehenden Kanalverkehr auf `allowlist` setzen, wenn die Richtlinie
  offenen Gruppenzugriff verweigert
- gemeldete `requireMention`-Pfade für eingehenden Kanalverkehr auf `true` setzen, wenn die Richtlinie
  Gruppenerwähnungen verlangt
- `logging.redactSensitive=tools` festlegen, wenn die Richtlinie die Schwärzung
  sensibler Protokolldaten verlangt
- `diagnostics.otel.captureContent=false` oder
  `diagnostics.otel.captureContent.enabled=false` für objektbasierte Einstellungen zur
  Telemetrieerfassung festlegen, wenn die Richtlinie die Erfassung von Telemetrieinhalten verweigert

Bereichsbezogene Reparaturen für erweiterte Tools dienen nur der Erkennung. Bereichsbezogene Reparaturen zur Datenverarbeitung werden
ebenfalls übersprungen, wenn der Befund eine gemeinsam verwendete Protokollierungs- oder Telemetriekonfiguration meldet,
da eine Änderung der gemeinsam verwendeten Einstellung mehr als das bereichsbezogene Richtlinienziel
betreffen würde.

Bereichsbezogene Reparaturen obligatorischer Verweigerungen werden übersprungen, wenn der Befund geerbtes
Root-`tools.deny` meldet, da das Hinzufügen des erforderlichen Tools zur Root-Konfiguration
mehr als das bereichsbezogene Richtlinienziel betreffen würde. Agent-lokale Reparaturen obligatorischer Verweigerungen können
den gemeldeten `agents.entries.*.tools.deny`-Pfad aktualisieren.

Bereichsbezogene Reparaturen des eingehenden Kanalverkehrs werden übersprungen, wenn der Befund geerbtes
`channels.defaults.*` meldet, da eine Änderung des gemeinsam verwendeten Kanalstandards
mehr als das bereichsbezogene Richtlinienziel betreffen würde. Befunde zur Positivliste für den HTTP-URL-Abruf des Gateways
bleiben manuell zu bearbeiten, da die automatische Reparatur nicht die korrekten URL-Werte
der Endpunkt-Positivliste auswählen kann.

Befunde zur Gateway-Bindung und zu Node-Befehlen bleiben überprüfungspflichtig. Wenn
`policy/gateway-non-loopback-bind` oder `policy/gateway-node-command-denied`
einem Konfigurationspfad zugeordnet werden können, meldet `doctor --fix` die vorgeschlagene
Änderung von `gateway.bind` oder `gateway.nodes.commands.deny` als übersprungene Vorschauhinweise.
Die Änderung wird nicht angewendet, und der Befund gilt erst dann als
repariert, wenn ein Operator die Konfiguration oder Richtlinie überprüft und aktualisiert.

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

## Exitcodes

| Befehl          | `0`                                                    | `1`                                                                 | `2`                          |
| ---------------- | ------------------------------------------------------ | ------------------------------------------------------------------- | ---------------------------- |
| `policy check`   | Keine Befunde am Schwellenwert.                          | Mindestens ein Befund erreichte den Schwellenwert.                             | Argument- oder Laufzeitfehler. |
| `policy compare` | Die Richtliniendatei ist mindestens so streng wie die Baseline. | Die Richtliniendatei ist ungültig, fehlt oder ist schwächer als die Baseline-Regeln. | Argument- oder Laufzeitfehler. |
| `policy watch`   | Keine Befunde, und der akzeptierte Hash ist aktuell.              | Es liegen Befunde vor oder die akzeptierte Attestierung ist veraltet.                    | Argument- oder Laufzeitfehler. |

## Verwandte Themen

- [Doctor-Lint-Modus](/de/cli/doctor#lint-mode)
- [Pfad-CLI](/de/cli/path)
