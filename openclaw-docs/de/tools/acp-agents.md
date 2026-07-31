---
read_when:
    - Coding-Harnesses über ACP ausführen
    - Einrichten konversationsgebundener ACP-Sitzungen in Messaging-Kanälen
    - Eine Unterhaltung in einem Nachrichtenkanal an eine persistente ACP-Sitzung binden
    - Fehlerbehebung für ACP-Backend, Plugin-Anbindung oder Übermittlung von Abschlüssen
    - /acp-Befehle über den Chat ausführen
sidebarTitle: ACP agents
summary: Externe Coding-Harnesses (Claude Code, Cursor, Gemini CLI, explizites Codex ACP, OpenClaw ACP, OpenCode) über das ACP-Backend ausführen
title: ACP-Agenten
x-i18n:
    generated_at: "2026-07-26T18:07:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fc7f32ff927c7e949be1595f6aa00ed034a51185c6a6b1e0df01a242954667d1
    source_path: tools/acp-agents.md
    workflow: 16
---

[Agent-Client-Protokoll (ACP)](https://agentclientprotocol.com/)-Sitzungen ermöglichen es
OpenClaw, externe Coding-Harnesses (Claude Code, Cursor, Copilot, Droid,
OpenClaw ACP, OpenCode, Gemini CLI und andere unterstützte ACPX-Harnesses)
über ein ACP-Backend-Plugin auszuführen. Jeder gestartete Prozess wird als
[Hintergrundaufgabe](/de/automation/tasks) verfolgt.

<Note>
**ACP ist der Pfad für externe Harnesses, nicht der standardmäßige Codex-Pfad.** Das native
Codex-App-Server-Plugin verwaltet `/codex ...`-Steuerelemente und die standardmäßige
eingebettete `openai/gpt-*`-Runtime für Agentenrunden; ACP verwaltet `/acp ...`-Steuerelemente
und `sessions_spawn({ runtime: "acp" })`-Sitzungen.

Damit Codex oder Claude Code als externer MCP-Client eine direkte Verbindung zu
bestehenden OpenClaw-Kanalunterhaltungen herstellen kann, verwenden Sie
[`openclaw mcp serve`](/de/cli/mcp) anstelle von ACP.
</Note>

## Welche Seite benötige ich?

| Sie möchten ...                                                                                  | Verwenden Sie                         | Hinweise                                                                                                                                                                       |
| ----------------------------------------------------------------------------------------------- | ------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Codex in der aktuellen Unterhaltung binden oder steuern                                         | `/codex bind`, `/codex threads`       | Nativer Codex-App-Server-Pfad, wenn das `codex`-Plugin aktiviert ist: gebundene Chatantworten, Bildweiterleitung, Modell/Schnellmodus/Berechtigungen, Stoppen und Steuern. ACP ist ein expliziter Fallback |
| Claude Code, Gemini CLI, explizites Codex ACP oder ein anderes externes Harness _über_ OpenClaw ausführen | Diese Seite                     | An Chats gebundene Sitzungen, `/acp spawn`, `sessions_spawn({ runtime: "acp" })`, Hintergrundaufgaben, Runtime-Steuerelemente |
| Eine OpenClaw-Gateway-Sitzung _als_ ACP-Server für einen Editor oder Client bereitstellen        | [`openclaw acp`](/de/cli/acp)            | Bridge-Modus: Eine IDE/ein Client kommuniziert über stdio/WebSocket per ACP mit OpenClaw                                                                                                      |
| Eine lokale KI-CLI als reines Text-Fallbackmodell wiederverwenden                               | [CLI-Backends](/de/gateway/cli-backends) | Kein ACP: keine OpenClaw-Tools, keine ACP-Steuerelemente, keine Harness-Runtime                                                                                                             |

## Funktioniert dies sofort?

Ja, nach der Installation des offiziellen ACP-Runtime-Plugins:

```bash
openclaw plugins install @openclaw/acpx
openclaw config set plugins.entries.acpx.enabled true
```

Quellcode-Checkouts können nach
`pnpm install` das lokale `extensions/acpx`-Workspace-Plugin verwenden. Führen Sie `/acp doctor` für eine Bereitschaftsprüfung aus.

OpenClaw informiert Agenten nur dann über das Starten per ACP, wenn ACP **tatsächlich verwendbar** ist:
ACP muss aktiviert sein, der Dispatch darf nicht deaktiviert sein, die aktuelle Sitzung darf
nicht durch die Sandbox blockiert sein und ein Runtime-Backend muss geladen und funktionsfähig sein. Wenn
eine dieser Bedingungen nicht erfüllt ist, bleiben ACP-Skills und die `sessions_spawn`-ACP-Anleitung ausgeblendet,
damit der Agent kein nicht verfügbares Backend vorschlägt.

<AccordionGroup>
  <Accordion title="Fallstricke beim ersten Start">
    - Wenn `plugins.allow` festgelegt ist, handelt es sich um ein restriktives Plugin-Inventar, das `acpx` **enthalten muss**. Andernfalls wird das installierte ACP-Backend absichtlich blockiert (`/acp doctor` meldet den fehlenden Eintrag in der Zulassungsliste).
    - Der Codex-ACP-Adapter wird mit dem `acpx`-Plugin ausgeliefert und nach Möglichkeit lokal gestartet.
    - Codex ACP wird mit einem isolierten `CODEX_HOME` ausgeführt. OpenClaw kopiert vertrauenswürdige Projekt-Vertrauenseinträge sowie sichere Modell-/Provider-Routing-Konfigurationen (`model`, `model_provider`, `model_reasoning_effort`, `sandbox_mode` und sichere `model_providers.<name>`-Felder) aus der Codex-Konfiguration des Hosts; Authentifizierung, Benachrichtigungen und Hooks verbleiben ausschließlich in der Hostkonfiguration.
    - Andere Ziel-Harness-Adapter können bei der ersten Verwendung bei Bedarf mit `npx` abgerufen werden.
    - Die Authentifizierung beim Hersteller muss für dieses Harness bereits auf dem Host vorhanden sein.
    - Wenn der Host weder über npm noch über Netzwerkzugriff verfügt, schlagen Adapterabrufe beim ersten Start fehl, bis die Caches vorab gefüllt wurden oder der Adapter auf andere Weise installiert wurde.

  </Accordion>
  <Accordion title="Runtime-Voraussetzungen">
    ACP startet einen echten externen Harness-Prozess. OpenClaw verwaltet Routing,
    den Zustand von Hintergrundaufgaben, Zustellung, Bindungen und Richtlinien; das Harness verwaltet
    seine Provider-Anmeldung, seinen Modellkatalog, sein Dateisystemverhalten und seine nativen Tools.

    Bevor Sie OpenClaw als Ursache ansehen, überprüfen Sie Folgendes:

    - `/acp doctor` meldet ein aktiviertes, funktionsfähiges Backend.
    - Die Ziel-ID ist durch `acp.allowedAgents` zugelassen, wenn diese Zulassungsliste festgelegt ist.
    - Der Harness-Befehl kann auf dem Gateway-Host gestartet werden.
    - Für dieses Harness ist eine Provider-Authentifizierung vorhanden (`claude`, `codex`, `gemini`, `opencode`, `droid` usw.).
    - Das ausgewählte Modell ist für dieses Harness verfügbar – Modell-IDs sind nicht zwischen Harnesses übertragbar.
    - Das angeforderte `cwd` ist vorhanden und zugänglich; lassen Sie andernfalls `cwd` weg, damit das Backend seinen Standardwert verwendet.
    - Der Berechtigungsmodus passt zur Aufgabe. Nicht interaktive Sitzungen können nicht auf native Berechtigungsaufforderungen klicken. Daher benötigen Coding-Ausführungen mit vielen Schreib- oder Ausführungsvorgängen normalerweise ein ACPX-Berechtigungsprofil, das ohne Benutzerinteraktion fortfahren kann.

  </Accordion>
</AccordionGroup>

OpenClaw-Plugin-Tools und integrierte OpenClaw-Tools werden ACP-Harnesses standardmäßig
**nicht** bereitgestellt. Aktivieren Sie die expliziten MCP-Bridges unter
[ACP-Agenten – Einrichtung](/de/tools/acp-agents-setup) nur, wenn das Harness
diese Tools direkt aufrufen soll.

## Unterstützte Harness-Ziele

Verwenden Sie mit dem `acpx`-Backend diese IDs als `/acp spawn <id>`- oder
`sessions_spawn({ runtime: "acp", agentId: "<id>" })`-Ziele:

| Harness-ID   | Typisches Backend                              | Hinweise                                                                               |
| ------------ | ---------------------------------------------- | ----------------------------------------------------------------------------------- |
| `claude`     | Claude-Code-ACP-Adapter                        | Erfordert Claude-Code-Authentifizierung auf dem Host.                                              |
| `codex`      | Codex-ACP-Adapter                              | Expliziter ACP-Fallback nur, wenn natives `/codex` nicht verfügbar ist oder ACP angefordert wird. |
| `copilot`    | GitHub-Copilot-ACP-Adapter                     | Erfordert Copilot-CLI-/Runtime-Authentifizierung.                                                  |
| `cursor`     | Cursor-CLI-ACP (`cursor-agent acp`)            | Überschreiben Sie den acpx-Befehl, wenn eine lokale Installation einen anderen ACP-Einstiegspunkt bereitstellt.    |
| `droid`      | Factory-Droid-CLI                              | Erfordert Factory-/Droid-Authentifizierung oder `FACTORY_API_KEY` in der Harness-Umgebung.        |
| `fast-agent` | fast-agent-mcp-ACP-Adapter                     | Wird bei Bedarf mit `uvx` abgerufen.                                                       |
| `gemini`     | Gemini-CLI-ACP-Adapter                         | Erfordert Gemini-CLI-Authentifizierung oder die Einrichtung eines API-Schlüssels.                                          |
| `iflow`      | iFlow CLI                                      | Verfügbarkeit des Adapters und Modellsteuerung hängen von der installierten CLI ab.                 |
| `kilocode`   | Kilo Code CLI                                  | Verfügbarkeit des Adapters und Modellsteuerung hängen von der installierten CLI ab.                 |
| `kimi`       | Kimi/Moonshot CLI                              | Erfordert Kimi-/Moonshot-Authentifizierung auf dem Host.                                            |
| `kiro`       | Kiro CLI                                       | Verfügbarkeit des Adapters und Modellsteuerung hängen von der installierten CLI ab.                 |
| `mux`        | Mux-CLI-ACP-Adapter                            | Wird bei Bedarf mit `npx` abgerufen.                                                       |
| `opencode`   | OpenCode-ACP-Adapter                           | Erfordert OpenCode-CLI-/Provider-Authentifizierung.                                                |
| `openclaw`   | OpenClaw-Gateway-Bridge über `openclaw acp` | Ermöglicht einem ACP-fähigen Harness die Rückkommunikation mit einer OpenClaw-Gateway-Sitzung.                 |
| `qoder`      | Qoder CLI                                      | Verfügbarkeit des Adapters und Modellsteuerung hängen von der installierten CLI ab.                 |
| `qwen`       | Qwen Code / Qwen CLI                           | Erfordert Qwen-kompatible Authentifizierung auf dem Host.                                          |
| `trae`       | Trae-CLI-ACP-Adapter                           | Verfügbarkeit des Adapters und Modellsteuerung hängen von der installierten CLI ab.                 |

`pi` (pi-acp) ist ebenfalls im acpx-Backend registriert, jedoch kein Coding-
Harness im gleichen Sinne wie die oben aufgeführten.

Benutzerdefinierte acpx-Agenten-Aliasse können in acpx selbst konfiguriert werden, die OpenClaw-
Richtlinie prüft jedoch vor dem Dispatch weiterhin `acp.allowedAgents` und jede
`agents.entries.*.runtime.acp.agent`-Zuordnung.

## Betriebshandbuch

Schneller `/acp`-Ablauf aus dem Chat:

<Steps>
  <Step title="Starten">
    `/acp spawn claude --bind here`,
    `/acp spawn gemini --mode persistent --thread auto` oder explizit
    `/acp spawn codex --bind here`.
  </Step>
  <Step title="Arbeiten">
    Fahren Sie in der gebundenen Unterhaltung oder im gebundenen Thread fort (oder geben Sie den Sitzungsschlüssel
    explizit als Ziel an).
  </Step>
  <Step title="Status prüfen">
    `/acp status`
  </Step>
  <Step title="Anpassen">
    `/acp model <provider/model>`, `/acp permissions <profile>`,
    `/acp timeout <seconds>`.
  </Step>
  <Step title="Steuern">
    Ohne den Kontext zu ersetzen: `/acp steer tighten logging and continue`.
  </Step>
  <Step title="Stoppen">
    `/acp cancel` (aktuelle Runde) oder `/acp close` (Sitzung und Bindungen).
  </Step>
</Steps>

<AccordionGroup>
  <Accordion title="Details zum Lebenszyklus">
    - Beim Starten wird eine ACP-Laufzeitsitzung erstellt oder fortgesetzt, ACP-Metadaten werden im OpenClaw-Sitzungsspeicher erfasst und es kann eine Hintergrundaufgabe erstellt werden, wenn der Lauf einer übergeordneten Aufgabe gehört.
    - ACP-Sitzungen, die einer übergeordneten Aufgabe gehören, werden auch dann als Hintergrundarbeit behandelt, wenn die Laufzeitsitzung persistent ist; Abschluss und oberflächenübergreifende Zustellung erfolgen über die Benachrichtigungsfunktion der übergeordneten Aufgabe, statt sich wie eine normale benutzerseitige Chatsitzung zu verhalten.
    - Die Aufgabenverwaltung schließt beendete oder verwaiste, einer übergeordneten Aufgabe gehörende einmalige ACP-Sitzungen. Persistente ACP-Sitzungen bleiben erhalten, solange eine aktive Konversationsbindung besteht; veraltete persistente Sitzungen ohne aktive Bindung werden geschlossen, damit sie nicht unbemerkt fortgesetzt werden können, nachdem die zugehörige Aufgabe abgeschlossen wurde oder ihr Aufgabeneintrag nicht mehr vorhanden ist.
    - Gebundene Folgenachrichten werden direkt an die ACP-Sitzung gesendet, bis die Bindung geschlossen, der Fokus aufgehoben, sie zurückgesetzt oder abgelaufen ist.
    - Gateway-Befehle bleiben lokal. `/acp ...`, `/status` und `/unfocus` werden niemals als normaler Prompt-Text an ein gebundenes ACP-Harness gesendet.
    - `cancel` bricht den aktiven Durchlauf ab, wenn das Backend den Abbruch unterstützt; die Bindung oder die Sitzungsmetadaten werden dadurch nicht gelöscht.
    - `close` beendet die ACP-Sitzung aus Sicht von OpenClaw und entfernt die Bindung. Ein Harness kann seinen eigenen vorgelagerten Verlauf weiterhin beibehalten, wenn es die Fortsetzung unterstützt.
    - Das acpx-Plugin bereinigt nach `close` die OpenClaw-eigenen Wrapper- und Adapter-Prozessbäume und beendet beim Start des Gateways veraltete verwaiste OpenClaw-eigene ACPX-Prozesse.
    - Inaktive Laufzeit-Worker können nach dem integrierten Inaktivitätszeitraum bereinigt werden; gespeicherte Sitzungsmetadaten bleiben für `/acp sessions` verfügbar.

  </Accordion>
  <Accordion title="Routingregeln für natives Codex">
    Auslöser in natürlicher Sprache, die an das **native Codex-Plugin** weitergeleitet
    werden sollten, wenn es aktiviert ist:

    - „Binden Sie diesen Discord-Kanal an Codex.“
    - „Verknüpfen Sie diesen Chat mit dem Codex-Thread `<id>`.“
    - „Zeigen Sie Codex-Threads an und binden Sie dann diesen.“

    Die native Codex-Konversationsbindung ist der standardmäßige Pfad zur Chat-Steuerung.
    Dynamische OpenClaw-Tools werden weiterhin über OpenClaw ausgeführt, während Codex-native
    Tools wie shell/apply-patch innerhalb von Codex ausgeführt werden. Für Codex-native
    Tool-Ereignisse fügt OpenClaw pro Durchlauf ein natives Hook-Relay ein, damit Plugin-Hooks
    `before_tool_call` blockieren, `after_tool_call` beobachten und Codex-
    `PermissionRequest`-Ereignisse über OpenClaw-Genehmigungen weiterleiten können. Codex-
    `Stop`-Hooks werden an OpenClaw `before_agent_finalize` weitergeleitet, wo Plugins
    einen weiteren Modelldurchlauf anfordern können, bevor Codex seine Antwort abschließt.
    Das Relay bleibt bewusst konservativ: Es verändert weder Argumente Codex-nativer Tools
    noch schreibt es Codex-Thread-Datensätze um. Verwenden Sie explizites ACP nur, wenn Sie
    das ACP-Laufzeit-/Sitzungsmodell verwenden möchten. Die Grenze der eingebetteten
    Codex-Unterstützung ist im
    [Supportvertrag für Codex Harness v1](/de/plugins/codex-harness-runtime#v1-support-contract)
    dokumentiert.

  </Accordion>
  <Accordion title="Kurzübersicht zur Modell-, Provider- und Laufzeitauswahl">
    - Veraltete Codex-Modellreferenzen – veraltete Codex-OAuth-/Abonnement-Modellroute, die durch doctor repariert wird.
    - `openai/*` – eingebettete native Codex-App-Server-Laufzeit für OpenAI-Agentendurchläufe.
    - `/codex ...` – native Codex-Konversationssteuerung.
    - `/acp ...` oder `runtime: "acp"` – explizite ACP-/acpx-Steuerung.

  </Accordion>
  <Accordion title="Natürlichsprachliche Auslöser für ACP-Routing">
    Auslöser, die an die ACP-Laufzeit weitergeleitet werden sollten:

    - „Führen Sie dies als einmalige Claude-Code-ACP-Sitzung aus und fassen Sie das Ergebnis zusammen.“
    - „Verwenden Sie Gemini CLI für diese Aufgabe in einem Thread und führen Sie Folgenachrichten anschließend im selben Thread fort.“
    - „Führen Sie Codex über ACP in einem Hintergrund-Thread aus.“

    OpenClaw wählt `runtime: "acp"`, löst das Harness `agentId` auf, bindet es,
    sofern unterstützt, an die aktuelle Konversation oder den aktuellen Thread und leitet
    Folgenachrichten bis zum Schließen oder Ablaufen an diese Sitzung weiter. Codex folgt
    diesem Pfad nur, wenn ACP/acpx explizit angegeben wurde oder das native Codex-Plugin
    für den angeforderten Vorgang nicht verfügbar ist.

    Für `sessions_spawn` wird `runtime: "acp"` nur angeboten, wenn ACP
    aktiviert ist, die anfragende Instanz nicht in einer Sandbox ausgeführt wird und
    ein ACP-Laufzeit-Backend geladen ist. `acp.dispatch.enabled=false` pausiert die automatische
    ACP-Thread-Weiterleitung, blendet explizite `sessions_spawn({ runtime: "acp" })`-Aufrufe jedoch weder
    aus noch blockiert es sie. Das Ziel sind ACP-Harness-IDs wie `codex`,
    `claude`, `droid`, `gemini` oder `opencode`.
    Übergeben Sie keine normale OpenClaw-Konfigurations-Agenten-ID aus `agents_list`,
    sofern dieser Eintrag nicht ausdrücklich mit `agents.entries.*.runtime.type="acp"` konfiguriert ist;
    verwenden Sie andernfalls die standardmäßige Sub-Agent-Laufzeit. Wenn ein
    OpenClaw-Agent mit `runtime.type="acp"` konfiguriert ist, verwendet OpenClaw
    `runtime.acp.agent` als zugrunde liegende Harness-ID.

  </Accordion>
</AccordionGroup>

## ACP im Vergleich zu Sub-Agents

Verwenden Sie ACP, wenn Sie eine externe Harness-Laufzeit benötigen. Verwenden Sie den
**nativen Codex-App-Server** für die Bindung und Steuerung von Codex-Konversationen, wenn
das Plugin `codex` aktiviert ist. Verwenden Sie **Sub-Agents**, wenn Sie
OpenClaw-native delegierte Läufe benötigen.

| Bereich        | ACP-Sitzung                           | Sub-Agent-Lauf                     |
| -------------- | ------------------------------------- | ---------------------------------- |
| Laufzeit       | ACP-Backend-Plugin (zum Beispiel acpx) | OpenClaw-native Sub-Agent-Laufzeit |
| Sitzungsschlüssel | `agent:<agentId>:acp:<uuid>`          | `agent:<agentId>:subagent:<uuid>`  |
| Hauptbefehle   | `/acp ...`                            | `/subagents ...`                   |
| Start-Tool     | `sessions_spawn` mit `runtime:"acp"` | `sessions_spawn` (Standardlaufzeit) |

Siehe auch [Sub-Agents](/de/tools/subagents).

## So führt ACP Claude Code aus

Für Claude Code über ACP besteht der Stack aus:

1. OpenClaw-Steuerungsebene für ACP-Sitzungen.
2. Offizielles Laufzeit-Plugin `@openclaw/acpx`.
3. Claude-ACP-Adapter.
4. Claude-seitige Laufzeit-/Sitzungsmechanik.

ACP Claude ist eine **Harness-Sitzung** mit ACP-Steuerungen, Sitzungsfortsetzung,
Hintergrundaufgabenverfolgung und optionaler Konversations-/Thread-Bindung.

CLI-Backends sind separate, ausschließlich textbasierte lokale Fallback-Laufzeiten – siehe
[CLI-Backends](/de/gateway/cli-backends).

Für Betreiber gilt in der Praxis:

- **Benötigen Sie `/acp spawn`, bindbare Sitzungen, Laufzeitsteuerungen oder persistente Harness-Arbeit?** Verwenden Sie ACP.
- **Benötigen Sie einen einfachen lokalen Text-Fallback über die unverarbeitete CLI?** Verwenden Sie CLI-Backends.

## Gebundene Sitzungen

### Mentales Modell

- **Chat-Oberfläche** – der Ort, an dem Personen weiter kommunizieren (Discord-Kanal, Telegram-Thema, iMessage-Chat).
- **ACP-Sitzung** – der dauerhafte Codex-/Claude-/Gemini-Laufzeitzustand, an den OpenClaw weiterleitet.
- **Untergeordneter Thread/untergeordnetes Thema** – eine optionale zusätzliche Nachrichtenoberfläche, die nur von `--thread ...` erstellt wird.
- **Laufzeit-Arbeitsbereich** – der Dateisystemspeicherort (`cwd`, Repository-Checkout, Backend-Arbeitsbereich), an dem das Harness ausgeführt wird. Unabhängig von der Chat-Oberfläche.

### Bindungen an die aktuelle Konversation

`/acp spawn <harness> --bind here` bindet die aktuelle Konversation an die
gestartete ACP-Sitzung – kein untergeordneter Thread, dieselbe Chat-Oberfläche. OpenClaw
behält die Kontrolle über Transport, Authentifizierung, Sicherheit und Zustellung.
Folgenachrichten in dieser Konversation werden an dieselbe Sitzung weitergeleitet;
`/new` und `/reset` setzen die Sitzung direkt zurück;
`/acp close` entfernt die Bindung.

Beispiele:

```text
/codex bind                                              # native Codex-Bindung; künftige Nachrichten hierher weiterleiten
/codex model gpt-5.4                                     # den gebundenen nativen Codex-Thread anpassen
/codex stop                                              # den aktiven nativen Codex-Durchlauf steuern
/acp spawn codex --bind here                             # expliziter ACP-Fallback für Codex
/acp spawn codex --thread auto                           # kann einen untergeordneten Thread/ein untergeordnetes Thema erstellen und dort binden
/acp spawn codex --bind here --cwd /workspace/repo       # dieselbe Chat-Bindung; Codex wird in /workspace/repo ausgeführt
```

<AccordionGroup>
  <Accordion title="Bindungsregeln und Exklusivität">
    - `--bind here` und `--thread ...` schließen sich gegenseitig aus.
    - `--bind here` funktioniert nur auf Kanälen, die eine Bindung an die aktuelle Konversation anbieten; andernfalls gibt OpenClaw eine eindeutige Meldung aus, dass dies nicht unterstützt wird. Bindungen bleiben über Gateway-Neustarts hinweg bestehen.
    - Bei Discord steuert `spawnSessions` die Erstellung untergeordneter Threads für `--thread auto|here` – nicht für `--bind here`.
    - Wenn Sie ohne `--cwd` einen anderen ACP-Agenten starten, übernimmt OpenClaw standardmäßig den Arbeitsbereich des **Ziel-Agenten**. Fehlende übernommene Pfade (`ENOENT`/`ENOTDIR`) greifen auf den Backend-Standard zurück; andere Zugriffsfehler (z. B. `EACCES`) werden als Startfehler ausgegeben.
    - Gateway-Verwaltungsbefehle bleiben in gebundenen Konversationen lokal – `/acp ...`-Befehle werden von OpenClaw verarbeitet, auch wenn normaler Folgenachrichtentext an die gebundene ACP-Sitzung weitergeleitet wird; `/status` und `/unfocus` bleiben ebenfalls lokal, sofern die Befehlsverarbeitung für diese Oberfläche aktiviert ist.

  </Accordion>
  <Accordion title="An Threads gebundene Sitzungen">
    Wenn Thread-Bindungen für einen Kanaladapter aktiviert sind:

    - OpenClaw bindet einen Thread an eine Ziel-ACP-Sitzung.
    - Folgenachrichten in diesem Thread werden an die gebundene ACP-Sitzung weitergeleitet.
    - ACP-Ausgaben werden an denselben Thread zurückgesendet.
    - Aufheben des Fokus, Schließen, Archivieren, eine Inaktivitätsüberschreitung oder das Ablaufen des Höchstalters entfernt die Bindung.
    - `/acp close`, `/acp cancel`, `/acp status`, `/status` und `/unfocus` sind Gateway-Befehle und keine Prompts für das ACP-Harness.

    Erforderliche Funktionsschalter für threadgebundenes ACP:

    - `acp.enabled=true`
    - `acp.dispatch.enabled` ist standardmäßig aktiviert (setzen Sie `false`, um die automatische ACP-Thread-Weiterleitung zu pausieren; explizite `sessions_spawn({ runtime: "acp" })`-Aufrufe funktionieren weiterhin).
    - Das Starten von Thread-Sitzungen durch Kanaladapter ist aktiviert (Standard: `true`):
      - Discord/Telegram: `session.threadBindings.spawnSessions=true`

    Die Unterstützung für Thread-Bindungen ist adapterspezifisch. Wenn der aktive
    Kanaladapter keine Thread-Bindungen unterstützt, gibt OpenClaw eine eindeutige
    Meldung aus, dass diese nicht unterstützt oder nicht verfügbar sind.

  </Accordion>
  <Accordion title="Kanäle mit Thread-Unterstützung">
    - Jeder Kanaladapter, der Funktionen zur Sitzungs-/Thread-Bindung bereitstellt.
    - Aktuelle integrierte Unterstützung: **Discord**-Threads/-Kanäle, **Telegram**-Themen (Forenthemen in Gruppen/Supergruppen und DM-Themen).
    - Plugin-Kanäle können Unterstützung über dieselbe Bindungsschnittstelle hinzufügen.

  </Accordion>
</AccordionGroup>

## Persistente Kanalbindungen

Konfigurieren Sie für nicht kurzlebige Workflows persistente ACP-Bindungen in
`bindings[]`-Einträgen der obersten Ebene.

### Bindungsmodell

<ParamField path="bindings[].type" type='"acp"'>
  Kennzeichnet eine persistente ACP-Konversationsbindung.
</ParamField>
<ParamField path="bindings[].match" type="object">
  Identifiziert die Zielkonversation. Kanalspezifische Strukturen:

- **Discord-Kanal/-Thread:** `match.channel="discord"` + `match.peer.id="<channelOrThreadId>"`
- **Slack-Kanal/DM:** `match.channel="slack"` + `match.peer.id="<channelId|channel:<channelId>|#<channelId>|userId|user:<userId>|slack:<userId>|<@userId>>"`. Bevorzugen Sie stabile Slack-IDs; Kanalbindungen erfassen auch Antworten innerhalb der Threads dieses Kanals.
- **Telegram-Forumsthema:** `match.channel="telegram"` + `match.peer.id="<chatId>:topic:<topicId>"`
- **WhatsApp-DM/-Gruppe:** `match.channel="whatsapp"` + `match.peer.id="<E.164|group JID>"`. Verwenden Sie für direkte Chats E.164-Nummern wie `+15555550123` und für Gruppen WhatsApp-Gruppen-JIDs wie `120363424282127706@g.us`.
- **iMessage-DM/-Gruppe:** `match.channel="imessage"` + `match.peer.id="<handle|chat_id:*|chat_guid:*|chat_identifier:*>"`. Bevorzugen Sie `chat_id:*` für stabile Gruppenbindungen.

</ParamField>
<ParamField path="bindings[].agentId" type="string">
  Die ID des zuständigen OpenClaw-Agenten.
</ParamField>
<ParamField path="bindings[].acp.mode" type='"persistent" | "oneshot"'>
  Optionale ACP-Überschreibung.
</ParamField>
<ParamField path="bindings[].acp.label" type="string">
  Optionale, für Bediener sichtbare Bezeichnung.
</ParamField>
<ParamField path="bindings[].acp.cwd" type="string">
  Optionales Laufzeit-Arbeitsverzeichnis.
</ParamField>
<ParamField path="bindings[].acp.backend" type="string">
  Optionale Backend-Überschreibung.
</ParamField>

### Laufzeitstandardwerte pro Agent

Verwenden Sie `agents.entries.*.runtime`, um ACP-Standardwerte einmal pro Agent zu definieren:

- `agents.entries.*.runtime.type="acp"`
- `agents.entries.*.runtime.acp.agent` (Harness-ID, z. B. `codex` oder `claude`)
- `agents.entries.*.runtime.acp.backend`
- `agents.entries.*.runtime.acp.mode`
- `agents.entries.*.runtime.acp.cwd`

**Überschreibungsrangfolge für ACP-gebundene Sitzungen:**

1. `bindings[].acp.*`
2. `agents.entries.*.runtime.acp.*`
3. Globale ACP-Standardwerte (z. B. `acp.backend`)

### Beispiel

```json5
{
  agents: {
    list: [
      {
        id: "codex",
        runtime: {
          type: "acp",
          acp: {
            agent: "codex",
            backend: "acpx",
            mode: "persistent",
            cwd: "/workspace/openclaw",
          },
        },
      },
      {
        id: "claude",
        runtime: {
          type: "acp",
          acp: { agent: "claude", backend: "acpx", mode: "persistent" },
        },
      },
    ],
  },
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "discord",
        accountId: "default",
        peer: { kind: "channel", id: "222222222222222222" },
      },
      acp: { label: "codex-main" },
    },
    {
      type: "acp",
      agentId: "claude",
      match: {
        channel: "telegram",
        accountId: "default",
        peer: { kind: "group", id: "-1001234567890:topic:42" },
      },
      acp: { cwd: "/workspace/repo-b" },
    },
    {
      type: "route",
      agentId: "main",
      match: { channel: "discord", accountId: "default" },
    },
    {
      type: "route",
      agentId: "main",
      match: { channel: "telegram", accountId: "default" },
    },
  ],
  channels: {
    discord: {
      guilds: {
        "111111111111111111": {
          channels: {
            "222222222222222222": { requireMention: false },
          },
        },
      },
    },
    telegram: {
      groups: {
        "-1001234567890": {
          topics: { "42": { requireMention: false } },
        },
      },
    },
  },
}
```

### Verhalten

- OpenClaw stellt nach der kanalspezifischen Zulassung und vor der Verwendung sicher, dass die konfigurierte ACP-Sitzung vorhanden ist.
- Nachrichten in diesem Kanal, Thema oder Chat werden an die konfigurierte ACP-Sitzung weitergeleitet.
- Konfigurierte ACP-Bindungen sind für ihre Sitzungsroute zuständig. Die Broadcast-Auffächerung des Kanals ersetzt bei einer übereinstimmenden Bindung nicht die konfigurierte ACP-Sitzung.
- In gebundenen Unterhaltungen setzen `/new` und `/reset` denselben ACP-Sitzungsschlüssel direkt zurück.
- Temporäre Laufzeitbindungen (beispielsweise durch Thread-Fokus-Abläufe erstellte) gelten weiterhin, sofern vorhanden.
- Bei agentenübergreifenden ACP-Starts ohne explizites `cwd` übernimmt OpenClaw den Arbeitsbereich des Zielagenten aus der Agentenkonfiguration.
- Fehlende übernommene Arbeitsbereichspfade greifen auf das standardmäßige Backend-cwd zurück; Zugriffsfehler bei vorhandenen Pfaden werden als Startfehler ausgegeben.

## ACP-Sitzungen starten

Es gibt zwei Möglichkeiten, eine ACP-Sitzung zu starten:

<Tabs>
  <Tab title="Über sessions_spawn">
    Verwenden Sie `runtime: "acp"`, um eine ACP-Sitzung aus einem Agentendurchlauf oder
    Tool-Aufruf zu starten.

    ```json
    {
      "task": "Öffnen Sie das Repository und fassen Sie die fehlgeschlagenen Tests zusammen",
      "runtime": "acp",
      "agentId": "codex",
      "thread": true,
      "mode": "session"
    }
    ```

    <Note>
    `runtime` verwendet standardmäßig `subagent`; setzen Sie daher `runtime: "acp"` für
    ACP-Sitzungen explizit. Wenn `agentId` weggelassen wird, verwendet OpenClaw `acp.defaultAgent`,
    sofern konfiguriert. `mode: "session"` erfordert `thread: true`, um eine
    dauerhaft gebundene Unterhaltung beizubehalten.
    </Note>

  </Tab>
  <Tab title="Über den Befehl /acp">
    Verwenden Sie `/acp spawn` für die explizite Bedienersteuerung über den Chat.

    ```text
    /acp spawn codex --mode persistent --thread auto
    /acp spawn codex --mode oneshot --thread off
    /acp spawn codex --bind here
    /acp spawn codex --thread here
    ```

    Wichtige Flags:

    - `--mode persistent|oneshot`
    - `--bind here|off`
    - `--thread auto|here|off`
    - `--cwd <absolute-path>`
    - `--label <name>`

    Siehe [Slash-Befehle](/de/tools/slash-commands).

  </Tab>
</Tabs>

### Parameter von `sessions_spawn`

<ParamField path="task" type="string" required>
  An die ACP-Sitzung gesendete initiale Anweisung.
</ParamField>
<ParamField path="runtime" type='"acp"' required>
  Muss für ACP-Sitzungen `"acp"` sein.
</ParamField>
<ParamField path="agentId" type="string">
  ID des ACP-Ziel-Harnesses. Greift auf `acp.defaultAgent` zurück, sofern festgelegt.
</ParamField>
<ParamField path="thread" type="boolean" default="false">
  Fordert den Thread-Bindungsablauf an, sofern unterstützt.
</ParamField>
<ParamField path="mode" type='"run" | "session"' default="run">
  `"run"` ist einmalig; `"session"` ist dauerhaft. Wenn `thread: true` und
  `mode` weggelassen werden, kann OpenClaw abhängig vom
  Laufzeitpfad standardmäßig dauerhaftes Verhalten verwenden. `mode: "session"` erfordert `thread: true`.
</ParamField>
<ParamField path="cwd" type="string">
  Angefordertes Laufzeit-Arbeitsverzeichnis (durch die Backend-/Laufzeitrichtlinie validiert).
  Wenn es weggelassen wird, übernimmt der ACP-Start den Arbeitsbereich des Zielagenten, sofern konfiguriert;
  fehlende übernommene Pfade greifen auf die Backend-Standardwerte zurück, während tatsächliche
  Zugriffsfehler zurückgegeben werden.
</ParamField>
<ParamField path="label" type="string">
  Für Bediener sichtbare Bezeichnung, die im Sitzungs-/Bannertext verwendet wird.
</ParamField>
<ParamField path="resumeSessionId" type="string">
  Setzt eine vorhandene ACP-Sitzung fort, anstatt eine neue zu erstellen. Der Agent
  spielt den Unterhaltungsverlauf über `session/load` erneut ab. Erfordert
  `runtime: "acp"`.
</ParamField>
<ParamField path="streamTo" type='"parent"'>
  `"parent"` überträgt Zusammenfassungen des Fortschritts des initialen ACP-Durchlaufs als Systemereignisse
  an die anfragende Sitzung zurück. OpenClaw zeichnet den vollständigen Weiterleitungsverlauf im
  SQLite-Zustand des untergeordneten Agenten auf und entfernt ihn zusammen mit der untergeordneten Sitzung. Übergeordnete
  Fortschrittsstreams zeigen standardmäßig Assistentenkommentare und ACP-Statusfortschritte an, sofern nicht
  `streaming.progress.commentary=false`. Discord verwendet für übergeordnete
  Vorschauen ebenfalls standardmäßig den Fortschrittsmodus, wenn kein Streammodus konfiguriert ist. Der
  Statusfortschritt berücksichtigt weiterhin `acp.stream.tagVisibility`, sodass Tags wie `plan`
  verborgen bleiben, sofern sie nicht ausdrücklich aktiviert werden.
</ParamField>

ACP-`sessions_spawn`-Durchläufe verwenden `agents.defaults.subagents.runTimeoutSeconds`
als standardmäßiges Limit für untergeordnete Durchläufe. Das Tool akzeptiert keine
Zeitüberschreibungen pro Aufruf (`runTimeoutSeconds`/`timeoutSeconds` werden mit einem
Fehler zurückgewiesen, der zum Konfigurieren des Standardwerts auffordert).

<ParamField path="model" type="string">
  Explizite Modellüberschreibung für die untergeordnete ACP-Sitzung. Codex-ACP-Starts
  normalisieren OpenAI-Referenzen wie `openai/gpt-5.4` vor `session/new` in die
  Codex-ACP-Startkonfiguration; Slash-Formen wie `openai/gpt-5.4/high` legen außerdem
  den Codex-ACP-Reasoning-Aufwand fest. Wenn der Wert weggelassen wird, verwendet `sessions_spawn({ runtime: "acp" })`
  vorhandene Standardmodelle für Subagenten (`agents.defaults.subagents.model` oder
  `agents.entries.*.subagents.model`), sofern konfiguriert; andernfalls verwendet das ACP-
  Harness sein eigenes Standardmodell. Andere Harnesses müssen ACP-
  `models` bekannt geben und `session/set_model` unterstützen; andernfalls schlägt OpenClaw/acpx
  eindeutig fehl, anstatt stillschweigend auf den Standardwert des Zielagenten zurückzugreifen.
</ParamField>
<ParamField path="thinking" type="string">
  Expliziter Denk-/Reasoning-Aufwand. Für Codex ACP wird `minimal` einem niedrigen
  Aufwand zugeordnet, `low`/`medium`/`high`/`xhigh` werden direkt zugeordnet und bei `off` wird die
  Startüberschreibung für den Reasoning-Aufwand weggelassen. Wenn der Wert weggelassen wird, verwenden ACP-Starts vorhandene
  Standardwerte für das Denken von Subagenten sowie das modellspezifische
  `agents.defaults.models["provider/model"].params.thinking` für das ausgewählte
  Modell.
</ParamField>

## Bindungs- und Thread-Modi beim Start

<Tabs>
  <Tab title="--bind here|off">
    | Modus   | Verhalten                                                               |
    | ------ | ----------------------------------------------------------------------- |
    | `here` | Bindet die aktuell aktive Unterhaltung direkt; schlägt fehl, wenn keine aktiv ist. |
    | `off`  | Erstellt keine Bindung für die aktuelle Unterhaltung.                          |

    Hinweise:

    - `--bind here` ist der einfachste Bedienerpfad, um „diesen Kanal oder Chat mit Codex zu betreiben“.
    - `--bind here` erstellt keinen untergeordneten Thread.
    - `--bind here` ist nur auf Kanälen verfügbar, die Bindungen für aktuelle Unterhaltungen unterstützen.
    - `--bind` und `--thread` können nicht im selben `/acp spawn`-Aufruf kombiniert werden.

  </Tab>
  <Tab title="--thread auto|here|off">
    | Modus   | Verhalten                                                                                            |
    | ------ | ------------------------------------------------------------------------------------------------- |
    | `auto` | In einem aktiven Thread: bindet diesen Thread. Außerhalb eines Threads: erstellt/bindet einen untergeordneten Thread, sofern unterstützt. |
    | `here` | Erfordert einen aktuell aktiven Thread; schlägt fehl, wenn keiner aktiv ist.                                                  |
    | `off`  | Keine Bindung. Die Sitzung startet ungebunden.                                                                 |

    Hinweise:

    - Auf Bindungsoberflächen ohne Threads entspricht das Standardverhalten effektiv `off`.
    - Ein threadgebundener Start erfordert Unterstützung durch die Kanalrichtlinie:
      - Discord/Telegram: `session.threadBindings.spawnSessions=true`
    - Verwenden Sie `--bind here`, wenn Sie die aktuelle Unterhaltung anheften möchten, ohne einen untergeordneten Thread zu erstellen.

  </Tab>
</Tabs>

## Zustellungsmodell

ACP-Sitzungen können entweder interaktive Arbeitsbereiche oder vom übergeordneten Prozess verwaltete
Hintergrundarbeit sein. Der Zustellungspfad hängt von dieser Ausprägung ab.

<AccordionGroup>
  <Accordion title="Interaktive ACP-Sitzungen">
    Interaktive Sitzungen sind dafür vorgesehen, die Unterhaltung auf einer sichtbaren Chatoberfläche fortzusetzen:

    - `/acp spawn ... --bind here` bindet die aktuelle Unterhaltung an die ACP-Sitzung.
    - `/acp spawn ... --thread ...` bindet einen Kanal-Thread/ein Kanalthema an die ACP-Sitzung.
    - Dauerhaft konfigurierte `bindings[].type="acp"` leiten übereinstimmende Unterhaltungen an dieselbe ACP-Sitzung weiter.

    Folgenachrichten in der gebundenen Unterhaltung werden direkt an die ACP-
    Sitzung weitergeleitet, und ACP-Ausgaben werden an denselben
    Kanal/Thread/dasselbe Thema zurückgesendet.

    Was OpenClaw an das Harness sendet:

    - Normale gebundene Folgeanfragen werden als Prompt-Text gesendet, mit Anhängen nur dann, wenn die Harness-/Backend-Unterstützung dafür vorhanden ist.
    - `/acp`-Verwaltungsbefehle und lokale Gateway-Befehle werden vor der ACP-Weiterleitung abgefangen.
    - Zur Laufzeit erzeugte Abschlussereignisse werden für jedes Ziel materialisiert. OpenClaw-Agenten erhalten den internen Laufzeitkontext-Umschlag von OpenClaw; externe ACP-Harnesses erhalten einen einfachen Prompt mit dem Ergebnis des untergeordneten Prozesses und einer Anweisung. Der unverarbeitete `<<<BEGIN_OPENCLAW_INTERNAL_CONTEXT>>>`-Umschlag darf niemals an externe Harnesses gesendet oder als Text eines ACP-Benutzertranskripts gespeichert werden.
    - ACP-Transkripteinträge verwenden den für Benutzer sichtbaren Auslösetext oder den einfachen Abschlussprompt. Interne Ereignismetadaten bleiben in OpenClaw nach Möglichkeit strukturiert und werden nicht als vom Benutzer verfasster Chat-Inhalt behandelt.

  </Accordion>
  <Accordion title="Übergeordnete einmalige ACP-Sitzungen">
    Einmalige ACP-Sitzungen, die von einem anderen Agentenlauf erzeugt werden, sind
    untergeordnete Hintergrundprozesse, ähnlich wie Unteragenten:

    - Der übergeordnete Prozess fordert mit `sessions_spawn({ runtime: "acp", mode: "run" })` Arbeit an.
    - Der untergeordnete Prozess wird in seiner eigenen ACP-Harness-Sitzung ausgeführt.
    - Untergeordnete Durchläufe werden auf derselben Hintergrundspur wie native Unteragentenstarts ausgeführt, sodass ein langsames ACP-Harness nicht die Arbeit anderer Hauptsitzungen blockiert.
    - Der Abschluss wird über den Ankündigungspfad für Aufgabenabschlüsse zurückgemeldet. OpenClaw wandelt interne Abschlussmetadaten in einen einfachen ACP-Prompt um, bevor dieser an ein externes Harness gesendet wird, sodass Harnesses keine OpenClaw-spezifischen Laufzeitkontextmarkierungen sehen.
    - Der übergeordnete Prozess formuliert das Ergebnis des untergeordneten Prozesses in normaler Assistentensprache neu, wenn eine für Benutzer sichtbare Antwort sinnvoll ist.

    Behandeln Sie diesen Pfad **nicht** als Peer-to-Peer-Chat zwischen dem übergeordneten und
    dem untergeordneten Prozess. Der untergeordnete Prozess verfügt bereits über einen Abschlusskanal zurück zum übergeordneten Prozess.

  </Accordion>
  <Accordion title="sessions_send und A2A-Zustellung">
    `sessions_send` kann nach dem Start auf eine andere Sitzung zielen. Für normale Peer-
    Sitzungen verwendet OpenClaw nach dem Einspeisen der Nachricht einen
    Agent-zu-Agent-Folgepfad (A2A):

    - Auf die Antwort der Zielsitzung warten.
    - Optional eine begrenzte Anzahl von Folgedurchläufen zwischen anfragender und Zielinstanz zulassen.
    - Das Ziel auffordern, eine Ankündigungsnachricht zu erstellen.
    - Diese Ankündigung an den sichtbaren Kanal oder Thread zustellen.

    Dieser A2A-Pfad dient als Rückfalloption für Peer-Sendungen, bei denen der Absender eine
    sichtbare Folgeantwort benötigt. Er bleibt aktiviert, wenn eine nicht zugehörige Sitzung ein
    ACP-Ziel sehen und ihm Nachrichten senden kann, beispielsweise bei umfassenden
    `tools.sessions.visibility`-Einstellungen.

    OpenClaw überspringt die A2A-Folgeaktion nur, wenn die anfragende Instanz der übergeordnete Prozess
    ihres eigenen, übergeordneten einmaligen ACP-Kinds ist. In diesem Fall kann die Ausführung von A2A zusätzlich
    zum Aufgabenabschluss den übergeordneten Prozess mit dem Ergebnis des untergeordneten Prozesses aktivieren, die
    Antwort des übergeordneten Prozesses zurück an den untergeordneten Prozess weiterleiten und eine
    Echo-Schleife zwischen übergeordnetem und untergeordnetem Prozess erzeugen. Das Ergebnis von `sessions_send` meldet
    für diesen Fall eines eigenen untergeordneten Prozesses `delivery.status="skipped"`, da der Abschlusspfad bereits
    für das Ergebnis zuständig ist.

  </Accordion>
  <Accordion title="Vorhandene Sitzung fortsetzen">
    Verwenden Sie `resumeSessionId`, um eine frühere ACP-Sitzung fortzusetzen, anstatt
    neu zu beginnen. Der Agent spielt seinen Gesprächsverlauf über
    `session/load` erneut ab und setzt somit mit dem vollständigen bisherigen Kontext fort.

    ```json
    {
      "task": "Fahren Sie dort fort, wo wir aufgehört haben – beheben Sie die verbleibenden Testfehler",
      "runtime": "acp",
      "agentId": "codex",
      "resumeSessionId": "<previous-session-id>"
    }
    ```

    Häufige Anwendungsfälle:

    - Eine Codex-Sitzung vom Laptop auf das Smartphone übergeben – weisen Sie Ihren Agenten an, dort fortzufahren, wo Sie aufgehört haben.
    - Eine Programmiersitzung fortsetzen, die Sie interaktiv in der CLI begonnen haben, nun ohne Benutzeroberfläche über Ihren Agenten.
    - Arbeit wiederaufnehmen, die durch einen Neustart des Gateway oder ein Inaktivitätszeitlimit unterbrochen wurde.

    Hinweise:

    - `resumeSessionId` gilt nur, wenn `runtime: "acp"`; die standardmäßige Unteragenten-Laufzeit ignoriert dieses ausschließlich für ACP bestimmte Feld.
    - `streamTo` gilt nur, wenn `runtime: "acp"`; die standardmäßige Unteragenten-Laufzeit ignoriert dieses ausschließlich für ACP bestimmte Feld.
    - `resumeSessionId` ist eine hostlokale ACP-/Harness-Fortsetzungs-ID und kein OpenClaw-Kanalsitzungsschlüssel; OpenClaw prüft vor der Weiterleitung weiterhin die ACP-Startrichtlinie und die Richtlinie des Zielagenten, während das ACP-Backend oder Harness für die Autorisierung zum Laden dieser vorgelagerten ID zuständig ist.
    - `resumeSessionId` stellt den vorgelagerten ACP-Gesprächsverlauf wieder her; `thread` und `mode` gelten weiterhin wie gewohnt für die neue OpenClaw-Sitzung, die Sie erstellen, daher erfordert `mode: "session"` weiterhin `thread: true`.
    - Der Zielagent muss `session/load` unterstützen (Codex und Claude Code tun dies).
    - Wenn die Sitzungs-ID nicht gefunden wird, schlägt der Start mit einer eindeutigen Fehlermeldung fehl – es erfolgt kein stiller Rückfall auf eine neue Sitzung.

  </Accordion>
  <Accordion title="Smoke-Test nach der Bereitstellung">
    Führen Sie nach einer Gateway-Bereitstellung eine aktive End-to-End-Prüfung durch, anstatt sich auf
    Unit-Tests zu verlassen:

    1. Die bereitgestellte Gateway-Version und den Commit auf dem Zielhost überprüfen.
    2. Eine temporäre ACPX-Bridge-Sitzung zu einem aktiven Agenten öffnen.
    3. Diesen Agenten auffordern, `sessions_spawn` mit `runtime: "acp"`, `agentId: "codex"`, `mode: "run"` und der Aufgabe `Reply with exactly LIVE-ACP-SPAWN-OK` aufzurufen.
    4. `accepted=yes`, einen echten `childSessionKey` und das Ausbleiben eines Validierungsfehlers überprüfen.
    5. Die temporäre Bridge-Sitzung bereinigen.

    Behalten Sie das Gate für `mode: "run"` bei und überspringen Sie `streamTo: "parent"` –
    Thread-gebundene `mode: "session"`- und Stream-Relay-Pfade sind separate, umfangreichere
    Integrationsdurchläufe.

  </Accordion>
</AccordionGroup>

## Sandbox-Kompatibilität

ACP-Sitzungen werden derzeit in der Host-Laufzeit ausgeführt, **nicht** innerhalb der OpenClaw-
Sandbox.

<Warning>
**Sicherheitsgrenze:**

- Das externe Harness kann entsprechend seinen eigenen CLI-Berechtigungen und dem ausgewählten `cwd` lesen und schreiben.
- Die Sandbox-Richtlinie von OpenClaw umschließt die Ausführung des ACP-Harnesses **nicht**.
- OpenClaw erzwingt weiterhin ACP-Funktions-Gates, zulässige Agenten, Sitzungseigentum, Kanalbindungen und die Gateway-Zustellungsrichtlinie.
- Verwenden Sie `runtime: "subagent"` für OpenClaw-native Arbeit mit erzwungener Sandbox.

</Warning>

Aktuelle Einschränkungen:

- Wenn die anfragende Sitzung in einer Sandbox ausgeführt wird, werden ACP-Starts sowohl für `sessions_spawn({ runtime: "acp" })` als auch für `/acp spawn` blockiert.
- `sessions_spawn` mit `runtime: "acp"` unterstützt `sandbox: "require"` nicht.

## Auflösung des Sitzungsziels

Die meisten `/acp`-Aktionen akzeptieren ein optionales Sitzungsziel (`session-key`,
`session-id` oder `session-label`).

**Auflösungsreihenfolge:**

1. Explizites Zielargument (oder `--session` für `/acp steer`)
   - versucht zuerst den Schlüssel
   - dann eine UUID-förmige Sitzungs-ID
   - dann die Bezeichnung
2. Aktuelle Thread-Bindung (wenn diese Unterhaltung/dieser Thread an eine ACP-Sitzung gebunden ist).
3. Rückfall auf die aktuelle anfragende Sitzung.

Sowohl Bindungen der aktuellen Unterhaltung als auch Thread-Bindungen sind an Schritt 2 beteiligt.

Wenn kein Ziel aufgelöst werden kann, gibt OpenClaw einen eindeutigen Fehler zurück
(`Unable to resolve session target: ...`).

## ACP-Steuerung

| Befehl              | Funktion                                              | Beispiel                                                       |
| -------------------- | --------------------------------------------------------- | ------------------------------------------------------------- |
| `/acp spawn`         | ACP-Sitzung erstellen; optionale aktuelle Bindung oder Thread-Bindung. | `/acp spawn codex --bind here --cwd /repo`                    |
| `/acp cancel`        | Laufenden Durchlauf für die Zielsitzung abbrechen.                 | `/acp cancel agent:codex:acp:<uuid>`                          |
| `/acp steer`         | Steueranweisung an die laufende Sitzung senden.                | `/acp steer --session support inbox prioritize failing tests` |
| `/acp close`         | Sitzung schließen und Thread-Ziele lösen.                  | `/acp close`                                                  |
| `/acp status`        | Backend, Modus, Status, Laufzeitoptionen und Fähigkeiten anzeigen. | `/acp status`                                                 |
| `/acp set-mode`      | Laufzeitmodus für die Zielsitzung festlegen.                      | `/acp set-mode plan`                                          |
| `/acp set`           | Generischen Wert einer Laufzeitkonfigurationsoption schreiben.                      | `/acp set model openai/gpt-5.4`                               |
| `/acp cwd`           | Überschreibung des Laufzeitarbeitsverzeichnisses festlegen.                   | `/acp cwd /Users/user/Projects/repo`                          |
| `/acp permissions`   | Profil der Genehmigungsrichtlinie festlegen.                              | `/acp permissions strict`                                     |
| `/acp timeout`       | Laufzeit-Zeitlimit (Sekunden) festlegen.                            | `/acp timeout 120`                                            |
| `/acp model`         | Überschreibung des Laufzeitmodells festlegen.                               | `/acp model anthropic/claude-opus-4-6`                        |
| `/acp reset-options` | Überschreibungen der Sitzungslaufzeitoptionen entfernen.                  | `/acp reset-options`                                          |
| `/acp sessions`      | Kürzlich verwendete ACP-Sitzungen aus dem Speicher auflisten.                      | `/acp sessions`                                               |
| `/acp doctor`        | Backend-Zustand, Fähigkeiten und umsetzbare Korrekturen.           | `/acp doctor`                                                 |
| `/acp install`       | Deterministische Installations- und Aktivierungsschritte ausgeben.             | `/acp install`                                                |

Laufzeitsteuerungen (`spawn`, `cancel`, `steer`, `close`, `status`, `set-mode`,
`set`, `cwd`, `permissions`, `timeout`, `model` und `reset-options`) erfordern
bei externen Kanälen die Eigentümeridentität und bei internen
Gateway-Clients `operator.admin`. Autorisierte Absender ohne Eigentümerstatus können weiterhin `sessions`,
`doctor`, `install` und `help` verwenden. Für Absender ohne Eigentümerstatus listet `/acp sessions`
nur die aktuell gebundene oder anfragende Sitzung auf; Eigentümeridentitäten und
`operator.admin`-Clients sehen alle kürzlich verwendeten Sitzungen.

`/acp status` zeigt die effektiven Laufzeitoptionen sowie Sitzungskennungen auf Laufzeit-
und Backend-Ebene. Fehler bei nicht unterstützten Steuerungen werden
eindeutig angezeigt, wenn einem Backend eine Fähigkeit fehlt. Befehle, die Zieltokens akzeptieren
(`session-key`, `session-id` oder `session-label`), lösen diese über die Gateway-
Sitzungserkennung auf, einschließlich benutzerdefinierter agentenspezifischer `session.store`-Stammverzeichnisse. `/acp sessions`
akzeptiert kein Zieltoken.

### Zuordnung der Laufzeitoptionen

`/acp` verfügt über Komfortbefehle und einen generischen Setter. Gleichwertige Vorgänge:

| Befehl                      | Entspricht                              | Hinweise                                                                                                                                                                                                      |
| ---------------------------- | ------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `/acp model <id>`            | Laufzeit-Konfigurationsschlüssel `model`           | Für Codex ACP normalisiert OpenClaw `openai/<model>` zur Adapter-Modell-ID und ordnet Reasoning-Suffixe mit Schrägstrich wie `openai/gpt-5.4/high` `reasoning_effort` zu.                                         |
| `/acp set thinking <level>`  | kanonische Option `thinking`          | OpenClaw sendet die vom Backend angegebene Entsprechung, sofern vorhanden, wobei `thinking`, danach `effort`, `reasoning_effort` oder `thought_level` bevorzugt wird. Für Codex ACP ordnet der Adapter die Werte `reasoning_effort` zu. |
| `/acp permissions <profile>` | kanonische Option `permissionProfile` | OpenClaw sendet die vom Backend angegebene Entsprechung, sofern vorhanden, beispielsweise `approval_policy`, `permission_profile`, `permissions` oder `permission_mode`.                                                       |
| `/acp timeout <seconds>`     | kanonische Option `timeoutSeconds`    | OpenClaw sendet die vom Backend angegebene Entsprechung, sofern vorhanden, beispielsweise `timeout` oder `timeout_seconds`.                                                                                                     |
| `/acp cwd <path>`            | Überschreibung des Laufzeit-Arbeitsverzeichnisses                 | Direkte Aktualisierung.                                                                                                                                                                                             |
| `/acp set <key> <value>`     | generisch                              | `key=cwd` verwendet den überschriebenen Pfad des Arbeitsverzeichnisses.                                                                                                                                                                      |
| `/acp reset-options`         | löscht alle Laufzeitüberschreibungen         | -                                                                                                                                                                                                          |

## acpx-Harness, Plugin-Einrichtung und Berechtigungen

Informationen zur Konfiguration des acpx-Harness (Aliasse für Claude Code / Codex / Gemini CLI),
zu den MCP-Bridges für Plugin-Tools und OpenClaw-Tools sowie zu den ACP-Berechtigungsmodi
finden Sie unter [ACP-Agenten – Einrichtung](/de/tools/acp-agents-setup).

## Fehlerbehebung

| Symptom                                                                                   | Wahrscheinliche Ursache                                                                                                           | Lösung                                                                                                                                                                      |
| ----------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `ACP runtime backend is not configured`                                                   | Backend-Plugin fehlt, ist deaktiviert oder wird durch `plugins.allow` blockiert.                                                       | Installieren und aktivieren Sie das Backend-Plugin, nehmen Sie `acpx` in `plugins.allow` auf, wenn diese Positivliste festgelegt ist, und führen Sie anschließend `/acp doctor` aus.                                                 |
| `ACP is disabled by policy (acp.enabled=false)`                                           | ACP ist global deaktiviert.                                                                                                 | Legen Sie `acp.enabled=true` fest.                                                                                                                                                  |
| `ACP dispatch is disabled by policy (acp.dispatch.enabled=false)`                         | Die automatische Weiterleitung normaler Thread-Nachrichten ist deaktiviert.                                                               | Legen Sie `acp.dispatch.enabled=true` fest, um die automatische Thread-Weiterleitung wieder zu aktivieren; explizite Aufrufe von `sessions_spawn({ runtime: "acp" })` funktionieren weiterhin.                                      |
| `ACP agent "<id>" is not allowed by policy`                                               | Der Agent befindet sich nicht in der Positivliste.                                                                                                | Verwenden Sie eine zulässige `agentId` oder aktualisieren Sie `acp.allowedAgents`.                                                                                                                     |
| `/acp doctor` meldet direkt nach dem Start, dass das Backend nicht bereit ist                               | Das Backend-Plugin fehlt, ist deaktiviert, wird durch eine Zulassungs-/Ablehnungsrichtlinie blockiert oder die konfigurierte ausführbare Datei ist nicht verfügbar.        | Installieren/aktivieren Sie das Backend-Plugin, führen Sie `/acp doctor` erneut aus und untersuchen Sie den Installations- oder Richtlinienfehler des Backends, falls es weiterhin fehlerhaft bleibt.                                           |
| Harness-Befehl nicht gefunden                                                                 | Die Adapter-CLI ist nicht installiert, das externe Plugin fehlt oder der erstmalige Abruf über `npx` ist bei einem Nicht-Codex-Adapter fehlgeschlagen. | Führen Sie `/acp doctor` aus, installieren Sie den Adapter auf dem Gateway-Host bzw. wärmen Sie ihn dort vor oder konfigurieren Sie den acpx-Agentenbefehl explizit.                                                      |
| „Modell nicht gefunden“ vom Harness                                                          | Die Modell-ID ist für einen anderen Provider/ein anderes Harness gültig, jedoch nicht für dieses ACP-Ziel.                                                | Verwenden Sie ein von diesem Harness aufgeführtes Modell, konfigurieren Sie das Modell im Harness oder lassen Sie die Überschreibung weg.                                                                            |
| Authentifizierungsfehler des Anbieters vom Harness                                                        | OpenClaw ist funktionsfähig, aber die Ziel-CLI bzw. der Ziel-Provider ist nicht angemeldet.                                                     | Melden Sie sich an oder stellen Sie den erforderlichen Provider-Schlüssel in der Umgebung des Gateway-Hosts bereit.                                                                                             |
| `Unable to resolve session target: ...`                                                   | Ungültiges Schlüssel-/ID-/Bezeichnungs-Token.                                                                                                | Führen Sie `/acp sessions` aus, kopieren Sie den exakten Schlüssel bzw. die exakte Bezeichnung und versuchen Sie es erneut.                                                                                                                        |
| `--bind here requires running /acp spawn inside an active ... conversation`               | `--bind here` wurde ohne aktive bindungsfähige Unterhaltung verwendet.                                                            | Wechseln Sie zum Zielchat/-kanal und versuchen Sie es erneut oder verwenden Sie einen ungebundenen Start.                                                                                                         |
| `Conversation bindings are unavailable for <channel>.`                                    | Dem Adapter fehlt die ACP-Bindungsfunktion für die aktuelle Unterhaltung.                                                             | Verwenden Sie `/acp spawn ... --thread ...`, sofern unterstützt, konfigurieren Sie `bindings[]` auf oberster Ebene oder wechseln Sie zu einem unterstützten Kanal.                                                     |
| `--thread here requires running /acp spawn inside an active ... thread`                   | `--thread here` wurde außerhalb eines Thread-Kontexts verwendet.                                                                         | Wechseln Sie zum Ziel-Thread oder verwenden Sie `--thread auto`/`off`.                                                                                                                      |
| `Only <user-id> can rebind this channel/conversation/thread.`                             | Ein anderer Benutzer besitzt das aktive Bindungsziel.                                                                           | Binden Sie es als Eigentümer erneut oder verwenden Sie eine andere Unterhaltung bzw. einen anderen Thread.                                                                                                               |
| `Thread bindings are unavailable for <channel>.`                                          | Dem Adapter fehlt die Thread-Bindungsfunktion.                                                                               | Verwenden Sie `--thread off` oder wechseln Sie zu einem unterstützten Adapter/Kanal.                                                                                                                 |
| `Sandboxed sessions cannot spawn ACP sessions ...`                                        | Die ACP-Laufzeit befindet sich auf dem Host; die Sitzung des Anfordernden befindet sich in einer Sandbox.                                                              | Verwenden Sie `runtime="subagent"` aus Sandbox-Sitzungen oder starten Sie ACP aus einer Sitzung ohne Sandbox.                                                                         |
| `sessions_spawn sandbox="require" is unsupported for runtime="acp" ...`                   | `sandbox="require"` wurde für die ACP-Laufzeit angefordert.                                                                         | Verwenden Sie `runtime="subagent"`, wenn Sandboxing erforderlich ist, oder verwenden Sie ACP mit `sandbox="inherit"` aus einer Sitzung ohne Sandbox.                                                      |
| `Cannot apply --model ... did not advertise model support`                                | Das Ziel-Harness stellt keine generische ACP-Modellumschaltung bereit.                                                        | Verwenden Sie ein Harness, das ACP-`models`/`session/set_model` angibt, verwenden Sie Codex-ACP-Modellreferenzen oder konfigurieren Sie das Modell direkt im Harness, falls es über ein eigenes Start-Flag verfügt. |
| Fehlende ACP-Metadaten für gebundene Sitzung                                                    | Veraltete/gelöschte ACP-Sitzungsmetadaten.                                                                                    | Erstellen Sie sie mit `/acp spawn` neu und binden bzw. fokussieren Sie anschließend den Thread erneut.                                                                                                                    |
| `PermissionPromptUnavailableError: Permission prompt unavailable in non-interactive mode` | `permissionMode` blockiert Schreibvorgänge/Ausführungen in einer nicht interaktiven ACP-Sitzung.                                                    | Setzen Sie `plugins.entries.acpx.config.permissionMode` auf `approve-all` und starten Sie das Gateway neu. Siehe [Berechtigungskonfiguration](/de/tools/acp-agents-setup#permission-configuration). |
| ACP-Sitzung schlägt frühzeitig mit wenig Ausgabe fehl                                                | Berechtigungsabfragen werden durch `permissionMode`/`nonInteractivePermissions` blockiert.                                        | Prüfen Sie die Gateway-Protokolle auf `AcpRuntimeError`. Legen Sie für vollständige Berechtigungen `permissionMode=approve-all` fest; für eine kontrollierte Einschränkung legen Sie `nonInteractivePermissions=deny` fest.        |
| ACP-Sitzung bleibt nach Abschluss der Arbeit unbegrenzt hängen                                     | Der Harness-Prozess ist beendet, aber die ACP-Sitzung hat den Abschluss nicht gemeldet.                                                    | Aktualisieren Sie OpenClaw; die aktuelle acpx-Bereinigung beendet beim Schließen und beim Start des Gateway veraltete Wrapper- und Adapterprozesse, die OpenClaw gehören.                                             |
| Harness sieht `<<<BEGIN_OPENCLAW_INTERNAL_CONTEXT>>>`                                      | Eine interne Ereignishülle ist über die ACP-Grenze gelangt.                                                                | Aktualisieren Sie OpenClaw und führen Sie den Abschlussablauf erneut aus; externe Harnesses sollten nur reine Abschlussaufforderungen erhalten.                                                          |

<Note>
`Command blocked by PreToolUse hook: Native hook relay unavailable` gehört zum
nativen Codex-Hook-Relay, nicht zu ACP/acpx. Starten Sie in einem gebundenen Codex-Chat eine
neue Sitzung mit `/new` oder `/reset`; wenn es einmal funktioniert und dann beim
nächsten nativen Tool-Aufruf erneut auftritt, starten Sie den Codex-App-Server oder das OpenClaw Gateway neu,
anstatt `/new` zu wiederholen. Siehe
[Fehlerbehebung für das Codex-Harness](/de/plugins/codex-harness#troubleshooting).
</Note>

## Verwandte Themen

- [ACP-Agenten – Einrichtung](/de/tools/acp-agents-setup)
- [Agent senden](/de/tools/agent-send)
- [CLI-Backends](/de/gateway/cli-backends)
- [Codex-Harness](/de/plugins/codex-harness)
- [Codex-Harness-Laufzeit](/de/plugins/codex-harness-runtime)
- [Multi-Agent-Sandbox-Tools](/de/tools/multi-agent-sandbox-tools)
- [`openclaw acp` (Bridge-Modus)](/de/cli/acp)
- [Unteragenten](/de/tools/subagents)
