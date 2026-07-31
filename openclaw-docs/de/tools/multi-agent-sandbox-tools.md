---
read_when: You want per-agent sandboxing or per-agent tool allow/deny policies in a multi-agent gateway.
sidebarTitle: Multi-agent sandbox and tools
status: active
summary: Sandbox pro Agent + Tool-Einschränkungen, Priorität und Beispiele
title: Multi-Agenten-Sandbox und Tools
x-i18n:
    generated_at: "2026-07-26T18:09:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0e07d07c30b844be1e1d93db62fcdaab72c47a5248367559642a959bf09ad193
    source_path: tools/multi-agent-sandbox-tools.md
    workflow: 16
---

Jeder Agent in einer Multi-Agent-Konfiguration kann die globale Sandbox- und Tool-Richtlinie überschreiben. Diese Seite behandelt die agentenspezifische Konfiguration, Vorrangregeln und Beispiele.

<CardGroup cols={3}>
  <Card title="Sandboxing" href="/de/gateway/sandboxing">
    Backends und Modi – vollständige Sandbox-Referenz.
  </Card>
  <Card title="Sandbox vs. Tool-Richtlinie vs. Elevated" href="/de/gateway/sandbox-vs-tool-policy-vs-elevated">
    Fehlerdiagnose für „Warum wird dies blockiert?“
  </Card>
  <Card title="Elevated-Modus" href="/de/tools/elevated">
    Elevated-Ausführung für vertrauenswürdige Absender.
  </Card>
</CardGroup>

<Warning>
Die Authentifizierung ist auf den jeweiligen Agenten beschränkt: Jeder Agent verfügt über einen eigenen `agentDir`-Authentifizierungsspeicher in `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`. Verwenden Sie `agentDir` niemals agentenübergreifend. Agenten können auf die Authentifizierungsprofile des standardmäßigen/Hauptagenten zurückgreifen, wenn sie kein lokales Profil besitzen; OAuth-Aktualisierungstoken werden jedoch nicht in die Speicher sekundärer Agenten kopiert. Wenn Sie Anmeldedaten manuell kopieren, kopieren Sie nur übertragbare statische `api_key`- oder `token`-Profile.
</Warning>

---

## Konfigurationsbeispiele

<AccordionGroup>
  <Accordion title="Beispiel 1: Persönlicher Agent und eingeschränkter Familienagent">
    ```json
    {
      "agents": {
        "list": [
          {
            "id": "main",
            "default": true,
            "name": "Personal Assistant",
            "workspace": "~/.openclaw/workspace",
            "sandbox": { "mode": "off" }
          },
          {
            "id": "family",
            "name": "Family Bot",
            "workspace": "~/.openclaw/workspace-family",
            "sandbox": {
              "mode": "all",
              "scope": "agent"
            },
            "tools": {
              "allow": ["read", "message"],
              "deny": ["exec", "write", "edit", "apply_patch", "process", "browser"],
              "message": {
                "crossContext": {
                  "allowWithinProvider": false,
                  "allowAcrossProviders": false
                }
              }
            }
          }
        ]
      },
      "bindings": [
        {
          "agentId": "family",
          "match": {
            "provider": "whatsapp",
            "accountId": "*",
            "peer": {
              "kind": "group",
              "id": "120363424282127706@g.us"
            }
          }
        }
      ]
    }
    ```

    **Ergebnis:**

    - `main`-Agent: wird auf dem Host ausgeführt und hat vollständigen Tool-Zugriff.
    - `family`-Agent: wird in Docker ausgeführt (ein Container pro Agent) und kann nur `read` sowie Nachrichten in der aktuellen Unterhaltung senden.

  </Accordion>
  <Accordion title="Beispiel 2: Arbeitsagent mit gemeinsam genutzter Sandbox">
    ```json
    {
      "agents": {
        "list": [
          {
            "id": "personal",
            "workspace": "~/.openclaw/workspace-personal",
            "sandbox": { "mode": "off" }
          },
          {
            "id": "work",
            "workspace": "~/.openclaw/workspace-work",
            "sandbox": {
              "mode": "all",
              "scope": "shared",
              "workspaceRoot": "/tmp/work-sandboxes"
            },
            "tools": {
              "allow": ["read", "write", "apply_patch", "exec"],
              "deny": ["browser", "gateway", "discord"]
            }
          }
        ]
      }
    }
    ```
  </Accordion>
  <Accordion title="Beispiel 2b: Globales Coding-Profil und Agent ausschließlich für Nachrichten">
    ```json
    {
      "tools": { "profile": "coding" },
      "agents": {
        "list": [
          {
            "id": "support",
            "tools": { "profile": "messaging", "allow": ["slack"] }
          }
        ]
      }
    }
    ```

    **Ergebnis:**

    - Standardagenten erhalten Coding-Tools.
    - Der `support`-Agent kann ausschließlich Nachrichten verarbeiten (+ Slack-Tool).

  </Accordion>
  <Accordion title="Beispiel 3: Unterschiedliche Sandbox-Modi pro Agent">
    ```json
    {
      "agents": {
        "defaults": {
          "sandbox": {
            "mode": "non-main",
            "scope": "session"
          }
        },
        "list": [
          {
            "id": "main",
            "workspace": "~/.openclaw/workspace",
            "sandbox": {
              "mode": "off"
            }
          },
          {
            "id": "public",
            "workspace": "~/.openclaw/workspace-public",
            "sandbox": {
              "mode": "all",
              "scope": "agent"
            },
            "tools": {
              "allow": ["read"],
              "deny": ["exec", "write", "edit", "apply_patch"]
            }
          }
        ]
      }
    }
    ```
  </Accordion>
</AccordionGroup>

---

## Konfigurationsvorrang

Wenn sowohl globale (`agents.defaults.*`) als auch agentenspezifische (`agents.entries.*.*`) Konfigurationen vorhanden sind:

### Sandbox-Konfiguration

Agentenspezifische Einstellungen überschreiben globale Einstellungen:

```text
agents.entries.*.sandbox.mode > agents.defaults.sandbox.mode
agents.entries.*.sandbox.scope > agents.defaults.sandbox.scope
agents.entries.*.sandbox.workspaceRoot > agents.defaults.sandbox.workspaceRoot
agents.entries.*.sandbox.workspaceAccess > agents.defaults.sandbox.workspaceAccess
agents.entries.*.sandbox.docker.* > agents.defaults.sandbox.docker.*
agents.entries.*.sandbox.browser.* > agents.defaults.sandbox.browser.*
agents.entries.*.sandbox.prune.* > agents.defaults.sandbox.prune.*
```

<Note>
`agents.entries.*.sandbox.{docker,browser,prune}.*` überschreibt `agents.defaults.sandbox.{docker,browser,prune}.*` für diesen Agenten (wird ignoriert, wenn der Sandbox-Geltungsbereich zu `"shared"` aufgelöst wird).
</Note>

### Tool-Einschränkungen

Die Filterreihenfolge lautet:

<Steps>
  <Step title="Tool-Profil">
    `tools.profile` oder `agents.entries.*.tools.profile`.
  </Step>
  <Step title="Provider-Tool-Profil">
    `tools.byProvider[provider].profile` oder `agents.entries.*.tools.byProvider[provider].profile`.
  </Step>
  <Step title="Globale Tool-Richtlinie">
    `tools.allow` / `tools.deny`.
  </Step>
  <Step title="Provider-Tool-Richtlinie">
    `tools.byProvider[provider].allow/deny`.
  </Step>
  <Step title="Agentenspezifische Tool-Richtlinie">
    `agents.entries.*.tools.allow/deny`.
  </Step>
  <Step title="Agenten-Provider-Richtlinie">
    `agents.entries.*.tools.byProvider[provider].allow/deny`.
  </Step>
  <Step title="Sandbox-Tool-Richtlinie">
    `tools.sandbox.tools` oder `agents.entries.*.tools.sandbox.tools`.
  </Step>
  <Step title="Subagenten-Tool-Richtlinie">
    `tools.subagents.tools`, falls zutreffend.
  </Step>
</Steps>

<AccordionGroup>
  <Accordion title="Vorrangregeln">
    - Jede Ebene kann Tools weiter einschränken, aber keine auf früheren Ebenen verweigerten Tools erneut gewähren.
    - Wenn `agents.entries.*.tools.sandbox.tools` festgelegt ist, ersetzt es `tools.sandbox.tools` für diesen Agenten.
    - Wenn `agents.entries.*.tools.profile` festgelegt ist, überschreibt es `tools.profile` für diesen Agenten.
    - Provider-Tool-Schlüssel akzeptieren entweder `provider` (z. B. `google-antigravity`) oder `provider/model` (z. B. `openai/gpt-5.4`).

  </Accordion>
  <Accordion title="Verhalten bei leerer Zulassungsliste">
    Wenn eine explizite Zulassungsliste in dieser Kette dazu führt, dass für die Ausführung keine aufrufbaren Tools verbleiben, hält OpenClaw an, bevor der Prompt an das Modell übermittelt wird. Dies ist beabsichtigt: Ein Agent, der mit einem fehlenden Tool wie `agents.entries.*.tools.allow: ["query_db"]` konfiguriert ist, soll mit einem deutlichen Fehler abbrechen, bis das Plugin aktiviert wird, das `query_db` registriert, statt als reiner Textagent fortzufahren.
  </Accordion>
</AccordionGroup>

Tool-Richtlinien unterstützen `group:*`-Kurzformen, die zu mehreren Tools erweitert werden. Die vollständige Liste finden Sie unter [Tool-Gruppen](/de/gateway/sandbox-vs-tool-policy-vs-elevated#tool-groups-shorthands).

Agentenspezifische Elevated-Überschreibungen (`agents.entries.*.tools.elevated`) können die Elevated-Ausführung für bestimmte Agenten weiter einschränken. Weitere Informationen finden Sie unter [Elevated-Modus](/de/tools/elevated).

---

## Migration von einem einzelnen Agenten

<Tabs>
  <Tab title="Vorher (einzelner Agent)">
    ```json
    {
      "agents": {
        "defaults": {
          "workspace": "~/.openclaw/workspace",
          "sandbox": {
            "mode": "non-main"
          }
        }
      },
      "tools": {
        "sandbox": {
          "tools": {
            "allow": ["read", "write", "apply_patch", "exec"],
            "deny": []
          }
        }
      }
    }
    ```
  </Tab>
  <Tab title="Nachher (mehrere Agenten)">
    ```json
    {
      "agents": {
        "list": [
          {
            "id": "main",
            "default": true,
            "workspace": "~/.openclaw/workspace",
            "sandbox": { "mode": "off" }
          }
        ]
      }
    }
    ```
  </Tab>
</Tabs>

<Note>
Veraltete `agents.defaults.*`/`agents.entries.*.*`-Konfigurationsschlüssel (wie `sandbox.perSession`, `agentRuntime`, `embeddedPi`) werden durch `openclaw doctor` migriert; verwenden Sie künftig vorzugsweise `agents.defaults` + `agents.entries`.
</Note>

---

## Beispiele für Tool-Einschränkungen

<Tabs>
  <Tab title="Schreibgeschützter Agent">
    ```json
    {
      "tools": {
        "allow": ["read"],
        "deny": ["exec", "write", "edit", "apply_patch", "process"]
      }
    }
    ```
  </Tab>
  <Tab title="Shell-Ausführung mit deaktivierten Dateisystem-Tools">
    ```json
    {
      "tools": {
        "allow": ["read", "exec", "process"],
        "deny": ["write", "edit", "apply_patch", "browser", "gateway"]
      }
    }
    ```

    <Warning>
    Diese Richtlinie deaktiviert die Dateisystem-Tools von OpenClaw, aber `exec` ist weiterhin eine Shell und kann überall dort Dateien schreiben, wo das Dateisystem des ausgewählten Hosts oder der Sandbox dies zulässt. Verweigern Sie für einen schreibgeschützten Agenten `exec` und `process`, oder kombinieren Sie den Shell-Zugriff mit Sandbox-Dateisystemkontrollen wie `agents.defaults.sandbox.workspaceAccess: "ro"` oder `"none"`.
    </Warning>

  </Tab>
  <Tab title="Nur Kommunikation">
    ```json
    {
      "tools": {
        "sessions": { "visibility": "tree" },
        "allow": ["sessions_list", "sessions_send", "sessions_history", "session_status"],
        "deny": ["exec", "write", "edit", "apply_patch", "read", "browser"]
      }
    }
    ```

    `sessions_history` gibt in diesem Profil weiterhin eine begrenzte, bereinigte Erinnerungsansicht statt eines Rohabzugs des Transkripts zurück. Die Assistentenerinnerung entfernt Denk-Tags, `<relevant-memories>`-Gerüstcode, XML-Nutzdaten von Tool-Aufrufen im Klartext (einschließlich `<tool_call>...</tool_call>`, `<function_call>...</function_call>`, `<tool_calls>...</tool_calls>`, `<function_calls>...</function_calls>` und abgeschnittener Tool-Aufrufblöcke), herabgestuften Tool-Aufruf-Gerüstcode, offengelegte ASCII-/vollbreite Modellsteuerungstoken und fehlerhaftes MiniMax-Tool-Aufruf-XML vor der Schwärzung/Kürzung.

  </Tab>
</Tabs>

---

## Häufiger Fallstrick: „non-main“

<Warning>
`agents.defaults.sandbox.mode: "non-main"` vergleicht den Sitzungsschlüssel mit dem Hauptsitzungsschlüssel (immer `"main"`; `session.mainKey` kann nicht benutzerseitig konfiguriert werden, und OpenClaw warnt bei jedem anderen Wert und ignoriert ihn), nicht mit der Agenten-ID. Gruppen-/Kanalsitzungen erhalten stets eigene Schlüssel, werden daher als Nicht-Hauptsitzungen behandelt und in einer Sandbox ausgeführt. Wenn ein Agent niemals in einer Sandbox ausgeführt werden soll, legen Sie `agents.entries.*.sandbox.mode: "off"` fest.
</Warning>

---

## Testen

Nach der Konfiguration der Multi-Agent-Sandbox und der Tools:

<Steps>
  <Step title="Agentenauflösung prüfen">
    ```bash
    openclaw agents list --bindings
    ```
  </Step>
  <Step title="Sandbox-Container überprüfen">
    ```bash
    docker ps --filter "name=openclaw-sbx-"
    ```
  </Step>
  <Step title="Tool-Einschränkungen testen">
    - Senden Sie eine Nachricht, die eingeschränkte Tools erfordert.
    - Überprüfen Sie, dass der Agent verweigerte Tools nicht verwenden kann.

  </Step>
  <Step title="Protokolle überwachen">
    ```bash
    openclaw logs --follow | grep -E "routing|sandbox|tools"
    ```
  </Step>
</Steps>

---

## Fehlerbehebung

<AccordionGroup>
  <Accordion title="Agent wird trotz `mode: 'all'` nicht in einer Sandbox ausgeführt">
    - Prüfen Sie, ob ein globales `agents.defaults.sandbox.mode` vorhanden ist, das diese Einstellung überschreibt.
    - Die agentenspezifische Konfiguration hat Vorrang; legen Sie daher `agents.entries.*.sandbox.mode: "all"` fest.

  </Accordion>
  <Accordion title="Trotz Sperrliste weiterhin verfügbare Tools">
    - Prüfen Sie die [vollständige Filterreihenfolge](#tool-restrictions): Profil → Provider-Profil → globale Richtlinie → Provider-Richtlinie → Agentenrichtlinie → Agenten-Provider-Richtlinie → Sandbox → Subagent.
    - Jede Ebene kann nur weitere Einschränkungen vornehmen, nicht jedoch Berechtigungen wieder gewähren.
    - Eine schrittweise Anleitung zur Fehlerbehebung finden Sie unter [Sandbox vs. Tool-Richtlinie vs. erhöhter Modus](/de/gateway/sandbox-vs-tool-policy-vs-elevated).

  </Accordion>
  <Accordion title="Container nicht pro Agent isoliert">
    - Der Standardwert für `scope` ist `"agent"` (ein Container pro Agenten-ID).
    - Legen Sie `scope: "session"` für einen Container pro Sitzung oder `scope: "shared"` fest, um einen Container agentenübergreifend wiederzuverwenden.

  </Accordion>
</AccordionGroup>

---

## Verwandte Themen

- [Erhöhter Modus](/de/tools/elevated)
- [Multi-Agenten-Routing](/de/concepts/multi-agent)
- [Sandbox-Konfiguration](/de/gateway/config-agents#agentsdefaultssandbox)
- [Sandbox vs. Tool-Richtlinie vs. erhöhter Modus](/de/gateway/sandbox-vs-tool-policy-vs-elevated) — Fehlerbehebung: „Warum wird dies blockiert?“
- [Sandboxing](/de/gateway/sandboxing) — vollständige Sandbox-Referenz (Modi, Geltungsbereiche, Backends, Images)
- [Sitzungsverwaltung](/de/concepts/session)
