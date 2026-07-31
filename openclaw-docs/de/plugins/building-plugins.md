---
doc-schema-version: 1
read_when:
    - Sie möchten ein neues OpenClaw-Plugin erstellen
    - Sie benötigen einen Schnellstart für die Plugin-Entwicklung
    - Sie wählen zwischen Dokumentationen zu Kanal, Provider, CLI-Backend, Tool oder Hook.
sidebarTitle: Getting Started
summary: Erstellen Sie Ihr erstes OpenClaw-Plugin in wenigen Minuten
title: Plugins erstellen
x-i18n:
    generated_at: "2026-07-26T17:58:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9d156ea305e46d3ca311a0b2cfc42e2c4522f6f10eb70cdd5526d9e9fcd7d4ef
    source_path: plugins/building-plugins.md
    workflow: 16
---

Plugins erweitern OpenClaw, ohne den Kern zu verändern. Ein Plugin kann einen
Messaging-Kanal, Modell-Provider, ein lokales CLI-Backend, Agenten-Tool, einen Hook, Medien-Provider
oder eine andere Plugin-eigene Funktion hinzufügen.

Sie müssen dem OpenClaw-Repository kein externes Plugin hinzufügen. Veröffentlichen Sie
das Paket auf [ClawHub](/de/clawhub); Benutzer installieren es mit:

```bash
openclaw plugins install clawhub:<package-name>
```

Reine Paketspezifikationen werden während der Einführungsumstellung weiterhin von npm installiert. Verwenden Sie das
Präfix `clawhub:`, wenn Sie die Auflösung über ClawHub wünschen.

## Anforderungen

- Node 22.22.3+, Node 24.15+ oder Node 25.9+ sowie `npm` oder `pnpm`.
- TypeScript-ESM-Module.
- Klonen Sie für die Arbeit an gebündelten Plugins im Repository das Repository und führen Sie `pnpm install` aus.
  Die Plugin-Entwicklung im Quellcode-Checkout erfolgt ausschließlich mit pnpm, da OpenClaw
  gebündelte Plugins aus `extensions/*`-Workspace-Paketen erkennt.

## Plugin-Struktur auswählen

<CardGroup cols={2}>
  <Card title="Kanal-Plugin" icon="messages-square" href="/de/plugins/sdk-channel-plugins">
    Verbinden Sie OpenClaw mit einer Messaging-Plattform.
  </Card>
  <Card title="Provider-Plugin" icon="cpu" href="/de/plugins/sdk-provider-plugins">
    Fügen Sie einen Modell-, Medien-, Such-, Abruf-, Sprach- oder Echtzeit-Provider hinzu.
  </Card>
  <Card title="CLI-Backend-Plugin" icon="terminal" href="/de/plugins/cli-backend-plugins">
    Führen Sie eine lokale KI-CLI über den Modell-Fallback von OpenClaw aus.
  </Card>
  <Card title="Tool-Plugin" icon="wrench" href="/de/plugins/tool-plugins">
    Registrieren Sie Agenten-Tools.
  </Card>
</CardGroup>

## Schnellstart

Erstellen Sie ein minimales Tool-Plugin, indem Sie ein erforderliches Agenten-Tool registrieren. Dies ist die
kürzeste nützliche Plugin-Struktur und deckt Paket, Manifest, Einstiegspunkt und
lokalen Nachweis ab.

<Steps>
  <Step title="Paketmetadaten erstellen">
    <CodeGroup>

```json package.json
{
  "name": "@myorg/openclaw-my-plugin",
  "version": "1.0.0",
  "type": "module",
  "dependencies": {
    "typebox": "1.1.39"
  },
  "peerDependencies": {
    "openclaw": ">=2026.3.24-beta.2"
  },
  "openclaw": {
    "extensions": ["./index.ts"],
    "compat": {
      "pluginApi": ">=2026.3.24-beta.2",
      "minGatewayVersion": "2026.3.24-beta.2"
    },
    "build": {
      "openclawVersion": "2026.3.24-beta.2",
      "pluginSdkVersion": "2026.3.24-beta.2"
    }
  }
}
```

```json openclaw.plugin.json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "description": "Adds a custom tool to OpenClaw",
  "contracts": {
    "tools": ["my_tool"]
  },
  "activation": {
    "onStartup": true
  },
  "configSchema": {
    "type": "object",
    "additionalProperties": false
  }
}
```

    </CodeGroup>

    Veröffentlichte externe Plugins sollten Laufzeiteinträge auf erstellte JavaScript-
    Dateien verweisen lassen. Den vollständigen Einstiegspunktvertrag finden Sie unter [SDK-Einstiegspunkte](/de/plugins/sdk-entrypoints).

    Jedes Plugin benötigt ein Manifest, auch ohne Konfiguration. Laufzeit-Tools müssen
    in `contracts.tools` aufgeführt sein, damit OpenClaw die Zuständigkeit erkennen kann, ohne
    jede Plugin-Laufzeit vorzeitig zu laden. Legen Sie `activation.onStartup`
    bewusst fest; dieses Beispiel wird beim Start des Gateway geladen.

    Auch vom Host als vertrauenswürdig eingestufte Plugin-Oberflächen sind durch das Manifest beschränkt und erfordern für
    installierte Plugins eine ausdrückliche Deklaration: Für `api.registerAgentToolResultMiddleware(...)`
    muss jede Ziellaufzeit in `contracts.agentToolResultMiddleware` aufgeführt sein,
    und für `api.registerTrustedToolPolicy(...)` muss jede Richtlinien-ID in
    `contracts.trustedToolPolicies` aufgeführt sein. Diese Deklarationen stimmen die Prüfung bei der Installation
    mit der Laufzeitregistrierung ab.

    Informationen zu allen Manifestfeldern finden Sie unter [Plugin-Manifest](/de/plugins/manifest).

  </Step>

  <Step title="Tool registrieren">
    ```typescript index.ts
    import { Type } from "typebox";
    import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

    export default definePluginEntry({
      id: "my-plugin",
      name: "My Plugin",
      description: "Adds a custom tool to OpenClaw",
      register(api) {
        api.registerTool({
          name: "my_tool",
          description: "Echo one input value",
          parameters: Type.Object({ input: Type.String() }),
          outputSchema: Type.Object(
            { input: Type.String() },
            { additionalProperties: false },
          ),
          async execute(_id, params) {
            const details = { input: params.input };
            return {
              content: [{ type: "text", text: `Got: ${params.input}` }],
              details,
            };
          },
        });
      },
    });
    ```

    Verwenden Sie `definePluginEntry` für Plugins, die keine Kanal-Plugins sind. Kanal-Plugins verwenden
    stattdessen `defineChannelPluginEntry` aus `openclaw/plugin-sdk/core`.

  </Step>

  <Step title="Laufzeit testen">
    Prüfen Sie bei einem installierten oder externen Plugin die geladene Laufzeit:

    ```bash
    openclaw plugins inspect my-plugin --runtime --json
    ```

    Wenn das Plugin einen CLI-Befehl registriert, führen Sie auch diesen Befehl aus und überprüfen Sie
    die Ausgabe, beispielsweise `openclaw demo-plugin ping`.

    Bei einem gebündelten Plugin in diesem Repository erkennt OpenClaw Plugin-Pakete
    aus dem Quellcode-Checkout im Workspace `extensions/*`. Führen Sie den am besten passenden gezielten
    Test aus:

    ```bash
    pnpm test extensions/my-plugin/
    pnpm check
    ```

  </Step>

  <Step title="Paketinstallation testen">
    Testen Sie vor der Veröffentlichung eines paketfertigen Plugins dieselbe Installationsform, die Benutzer
    erhalten werden. Fügen Sie zunächst einen Build-Schritt hinzu, lassen Sie Laufzeiteinträge wie
    `openclaw.extensions` auf erstelltes JavaScript wie `./dist/index.js` verweisen und stellen Sie
    sicher, dass `npm pack` diese `dist/`-Ausgabe enthält. TypeScript-Quellcodeeinträge sind
    ausschließlich für Quellcode-Checkouts und lokale Entwicklungspfade vorgesehen.

    Packen Sie dann das Plugin und installieren Sie das Tarball mit `npm-pack:`:

    ```bash
    npm pack --pack-destination /tmp
    openclaw plugins install npm-pack:/tmp/<plugin-package>.tgz --force
    openclaw plugins inspect my-plugin --runtime --json
    ```

    `npm-pack:` verwendet das von OpenClaw verwaltete npm-Projekt pro Plugin und erkennt daher
    Fehler bei Laufzeitabhängigkeiten, die Tests im Quellcode-Checkout verbergen können. Dies weist
    die Paket- und Abhängigkeitsstruktur nach, nicht jedoch den mit einem Katalog verknüpften offiziellen Vertrauensstatus.
    Laufzeitimporte müssen in `dependencies` oder `optionalDependencies` enthalten sein;
    Abhängigkeiten, die nur in `devDependencies` verbleiben, werden für das
    verwaltete Laufzeitprojekt nicht installiert.

    Verwenden Sie eine direkte Archiv-/Pfadinstallation nicht als abschließenden Nachweis für offizielles oder
    privilegiertes Plugin-Verhalten. Direkte Quellen sind für die lokale Fehlersuche nützlich,
    weisen jedoch nicht denselben Abhängigkeitspfad wie Installationen über npm oder ClawHub nach. Wenn
    Ihr Plugin auf dem Vertrauensstatus eines offiziellen Plugins beruht, ergänzen Sie einen zweiten Nachweis
    über eine kataloggestützte offizielle Installation oder einen veröffentlichten Paketpfad, der
    den offiziellen Vertrauensstatus aufzeichnet. Einzelheiten zu Installationsstamm und
    Zuständigkeit für Abhängigkeiten finden Sie unter
    [Auflösung von Plugin-Abhängigkeiten](/de/plugins/dependency-resolution).

  </Step>

  <Step title="Veröffentlichen">
    Validieren Sie das Paket vor der Veröffentlichung:

    ```bash
    clawhub package publish your-org/your-plugin --dry-run
    clawhub package publish your-org/your-plugin
    ```

    Kanonische ClawHub-Paketausschnitte befinden sich in `docs/snippets/plugin-publish/`.

  </Step>

  <Step title="Installieren">
    Installieren Sie das veröffentlichte Paket über ClawHub:

    ```bash
    openclaw plugins install clawhub:your-org/your-plugin
    ```

  </Step>
</Steps>

<a id="registering-agent-tools"></a>

## Tools registrieren

Tools können erforderlich oder optional sein. Erforderliche Tools sind immer verfügbar, wenn das
Plugin aktiviert ist. Optionale Tools erfordern die ausdrückliche Zustimmung des Benutzers, bevor OpenClaw
die zugehörige Plugin-Laufzeit lädt.

Tool-Factories erhalten einen vertrauenswürdigen Laufzeitkontext, einschließlich `deliveryContext`,
`nativeChannelId` für die aktive Plattformkonversation, sofern verfügbar, und
`requesterSenderId`.

```typescript
register(api) {
  api.registerTool(
    {
      name: "workflow_tool",
      description: "Run a workflow",
      parameters: Type.Object({ pipeline: Type.String() }),
      outputSchema: Type.Object(
        { pipeline: Type.String() },
        { additionalProperties: false },
      ),
      async execute(_id, params) {
        return {
          content: [{ type: "text", text: params.pipeline }],
          details: { pipeline: params.pipeline },
        };
      },
    },
    { optional: true },
  );
}
```

`outputSchema` ist optional. Es beschreibt den strukturierten `details`-Wert, der von
[Code Mode](/de/tools/code-mode) und [Tool-Suche](/de/tools/tool-search) verwendet wird. Katalog-
Aufrufe lehnen ungültige Schemas vor der Ausführung ab und validieren den endgültigen Wert nach
Tool-Hooks. Lassen Sie es bei Tools ohne stabiles JSON-Ergebnis weg. Den vollständigen Vertrag finden Sie unter
[Tool-Plugins](/de/plugins/tool-plugins#output-contracts).

Jedes mit `api.registerTool(...)` registrierte Tool muss außerdem im
Plugin-Manifest deklariert werden:

```json
{
  "contracts": {
    "tools": ["workflow_tool"]
  },
  "toolMetadata": {
    "workflow_tool": {
      "optional": true
    }
  }
}
```

Benutzer stimmen mit `tools.allow` zu:

```json5
{
  tools: { allow: ["workflow_tool"] }, // or ["my-plugin"] for every tool from one plugin
}
```

Optionale Tools steuern, ob ein Tool dem Modell bereitgestellt wird. Verwenden Sie
[Plugin-Berechtigungsanfragen](/de/plugins/plugin-permission-requests), wenn ein Tool
oder Hook nach der Auswahl durch das Modell und vor der
Ausführung der Aktion eine Genehmigung anfordern soll.

Verwenden Sie optionale Tools für Nebeneffekte, ungewöhnliche Binärdateien oder Funktionen, die
standardmäßig nicht bereitgestellt werden sollten. Tool-Namen dürfen nicht mit Namen von Kern-Tools
kollidieren; Konflikte werden übersprungen und in der Plugin-Diagnose gemeldet. Fehlerhafte
Registrierungen werden übersprungen und auf dieselbe Weise gemeldet: ein fehlendes, nicht leeres
`name`, ein `execute`, das keine Funktion ist, oder ein Tool-Deskriptor ohne ein `parameters`-
Objekt.

Tool-Factories erhalten ein von der Laufzeit bereitgestelltes Kontextobjekt. Verwenden Sie `ctx.activeModel`,
wenn ein Tool das aktive Modell für den aktuellen
Durchlauf protokollieren, anzeigen oder sich daran anpassen muss; es kann `provider`, `modelId` und `modelRef` enthalten. Behandeln Sie es als
informative Laufzeitmetadaten und nicht als Sicherheitsgrenze gegenüber dem lokalen
Betreiber, installiertem Plugin-Code oder einer modifizierten OpenClaw-Laufzeit. Sensible
lokale Tools sollten weiterhin eine ausdrückliche Zustimmung für das Plugin oder durch den Betreiber erfordern und
geschlossen fehlschlagen, wenn Metadaten zum aktiven Modell fehlen oder ungeeignet sind.

Das Manifest deklariert Zuständigkeit und Erkennung; die Ausführung ruft weiterhin die aktive
registrierte Tool-Implementierung auf. Halten Sie `toolMetadata.<tool>.optional: true`
mit `api.registerTool(..., { optional: true })` konsistent, damit OpenClaw das Laden
dieser Plugin-Laufzeit vermeiden kann, bis das Tool ausdrücklich in die Zulassungsliste aufgenommen wird.

## Importkonventionen

Importieren Sie aus spezifischen SDK-Unterpfaden:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
```

Verwenden Sie innerhalb Ihres Plugin-Pakets lokale Barrel-Dateien wie `api.ts` und
`runtime-api.ts` für interne Importe. Importieren Sie Ihr eigenes Plugin nicht über einen
SDK-Pfad. Provider-spezifische Hilfsfunktionen sollten im Provider-Paket verbleiben, sofern
die Schnittstelle nicht wirklich generisch ist.

Benutzerdefinierte Gateway-RPC-Methoden sind ein erweiterter Einstiegspunkt. Verwenden Sie dafür ein
Plugin-spezifisches Präfix; zentrale Admin-Namespaces wie `config.*`,
`exec.approvals.*`, `operator.admin.*`, `wizard.*` und `update.*` bleiben reserviert
und werden zu `operator.admin` aufgelöst. Die
`openclaw/plugin-sdk/gateway-method-runtime`-Bridge ist für Plugin-HTTP-
Routen reserviert, die `contracts.gatewayMethodDispatch: ["authenticated-request"]` deklarieren.

Die vollständige Importübersicht finden Sie unter [Plugin-SDK-Übersicht](/de/plugins/sdk-overview).

Die OpenClaw-SDK-Kompatibilitätsfelder enthalten TypeScript-`@deprecated`-Annotationen,
die Editoren als Migrationswarnungen anzeigen. Um sie während des Builds durchzusetzen,
aktivieren Sie eine typbewusste Regel wie
[`@typescript-eslint/no-deprecated`](https://typescript-eslint.io/rules/no-deprecated/).
Oxlint ist nicht typbewusst und kann diese Annotationen daher nicht durchsetzen.

## Checkliste vor der Einreichung

<Check>**package.json** enthält korrekte `openclaw`-Metadaten</Check>
<Check>Das **openclaw.plugin.json**-Manifest ist vorhanden und gültig</Check>
<Check>Der Einstiegspunkt verwendet `defineChannelPluginEntry` oder `definePluginEntry`</Check>
<Check>Alle Importe verwenden spezifische `plugin-sdk/<subpath>`-Pfade</Check>
<Check>Interne Importe verwenden lokale Module, keine SDK-Selbstimporte</Check>
<Check>Tests sind erfolgreich (`pnpm test <bundled-plugin-root>/my-plugin/`)</Check>
<Check>`pnpm check` ist erfolgreich (repo-interne Plugins)</Check>

## Gegen Beta-Releases testen

1. Beobachten Sie die Releases von [openclaw/openclaw](https://github.com/openclaw/openclaw/releases) (`Watch` > `Releases`). Beta-Tags sehen wie `v2026.3.N-beta.1` aus. Sie können außerdem [@openclaw](https://x.com/openclaw) auf X folgen, um Release-Ankündigungen zu erhalten.
2. Testen Sie Ihr Plugin gegen den Beta-Tag, sobald er erscheint. Das Zeitfenster vor dem stabilen Release beträgt in der Regel nur wenige Stunden.
3. Veröffentlichen Sie nach dem Testen im Thread Ihres Plugins im Discord-Kanal `plugin-forum` ([discord.gg/clawd](https://discord.gg/clawd)) entweder `all good` oder eine Beschreibung dessen, was nicht funktioniert hat. Erstellen Sie einen Thread, falls Sie noch keinen haben.
4. Wenn etwas nicht funktioniert, öffnen oder aktualisieren Sie ein Issue mit dem Titel `Beta blocker: <plugin-name> - <summary>` und weisen Sie ihm das Label `beta-blocker` zu. Verlinken Sie das Issue in Ihrem Thread.
5. Öffnen Sie einen PR für `main` mit dem Titel `fix(<plugin-id>): beta blocker - <summary>` und verlinken Sie das Issue sowohl im PR als auch in Ihrem Discord-Thread. Mitwirkende können PRs keine Labels zuweisen, daher dient der Titel als PR-seitiges Signal für Maintainer und Automatisierung. Blocker mit einem PR werden zusammengeführt; Blocker ohne PR werden möglicherweise trotzdem ausgeliefert.
6. Keine Rückmeldung bedeutet grünes Licht. Wenn Sie das Zeitfenster verpassen, wird Ihre Fehlerbehebung normalerweise erst im nächsten Zyklus aufgenommen.

## Nächste Schritte

<CardGroup cols={2}>
  <Card title="Kanal-Plugins" icon="messages-square" href="/de/plugins/sdk-channel-plugins">
    Ein Plugin für einen Nachrichtenkanal entwickeln
  </Card>
  <Card title="Provider-Plugins" icon="cpu" href="/de/plugins/sdk-provider-plugins">
    Ein Plugin für einen Modell-Provider entwickeln
  </Card>
  <Card title="CLI-Backend-Plugins" icon="terminal" href="/de/plugins/cli-backend-plugins">
    Ein lokales KI-CLI-Backend registrieren
  </Card>
  <Card title="SDK-Übersicht" icon="book-open" href="/de/plugins/sdk-overview">
    Referenz für Importzuordnung und Registrierungs-API
  </Card>
  <Card title="Runtime-Hilfsfunktionen" icon="settings" href="/de/plugins/sdk-runtime">
    TTS, Suche und Subagent über api.runtime
  </Card>
  <Card title="Tests" icon="test-tubes" href="/de/plugins/sdk-testing">
    Testhilfsprogramme und -muster
  </Card>
  <Card title="Plugin-Manifest" icon="file-json" href="/de/plugins/manifest">
    Vollständige Referenz zum Manifestschema
  </Card>
</CardGroup>

## Verwandte Themen

- [Plugin-Hooks](/de/plugins/hooks)
- [Plugin-Architektur](/de/plugins/architecture)
