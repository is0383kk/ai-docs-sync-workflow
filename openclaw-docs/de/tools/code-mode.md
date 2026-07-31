---
read_when:
    - Sie möchten den OpenClaw-Codemodus für einen Agentenlauf aktivieren
    - Sie müssen erklären, warum sich Code Mode vom Codex Code Mode unterscheidet
    - Sie überprüfen den kompakten Tool-Vertrag, die QuickJS-WASI-Sandbox, die TypeScript-Transformation oder die verborgene Tool-Katalog-Bridge
    - Sie fügen eine interne Namespace-Registry-Integration für den Code-Modus hinzu oder überprüfen sie.
sidebarTitle: Code Mode
summary: Verwenden Sie den OpenClaw-Code-Modus, um umfangreiche Tool-Kataloge zu durchsuchen, aufzurufen und in kompakten JavaScript- oder TypeScript-Workflows zu kombinieren
title: Code-Modus
x-i18n:
    generated_at: "2026-07-26T18:06:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a21df3bcfb11668da6dde1f7c69adcc284a28dc491c95f95097ce7f41e5c45bf
    source_path: tools/code-mode.md
    workflow: 16
---

Der Code-Modus ist eine experimentelle, optionale Funktion der OpenClaw-Agent-Laufzeit. Wenn
er aktiviert ist, sieht das Modell nicht mehr jedes aktivierte Tool-Schema, sondern
`exec`, `wait` sowie jedes ausschließlich direkte Tool, dessen strukturiertes Ergebnis nicht über
die reine JSON-Gast-Bridge übertragen werden kann. Das Modell schreibt ein kleines JavaScript- oder TypeScript-
Programm, das den verborgenen Tool-Katalog durchsucht, beschreibt und aufruft.

Diese Seite dokumentiert den OpenClaw-Code-Modus, nicht den Codex Code Mode. Die beiden Funktionen
haben denselben Namen und dieselben Namen für Steuerungs-Tools (`exec`, `wait`), sind jedoch
separate Implementierungen:

- Codex Code Mode wird innerhalb der Codex-Coding-Umgebung ausgeführt. Sein Tool `exec` ist ein
  Tool mit Freiformgrammatik: Das Modell schreibt unverarbeiteten JavaScript-Quellcode (optional
  mit einer vorangestellten `// @exec: {...}`-Pragma-Zeile für Ausführungsoptionen), der
  in der prozessinternen V8-Code-Mode-Laufzeit von Codex ausgeführt wird.
- Der OpenClaw-Code-Modus wird in der generischen OpenClaw-Agent-Laufzeit ausgeführt und ist
  deaktiviert, sofern `tools.codeMode.enabled: true` nicht konfiguriert ist. Sein Tool `exec`
  akzeptiert eine JSON-Nutzlast `{ code, language }`, die in einem QuickJS-WASI-
  Worker ausgeführt wird.

Beide sind JavaScript-Ausführungsoberflächen, keine Oberflächen für Shell-Befehle. Behandeln Sie sie
als unabhängige, unterschiedlich implementierte Funktionen, die zufällig
gleichnamige Tools `exec`/`wait` bereitstellen.

## Funktionsweise

- Die für das Modell sichtbare Tool-Liste wird zu `exec`, `wait` sowie jedem ausschließlich direkten Tool
  wie `computer` oder dem nativen Bildverarbeitungs-Loader `image`, dessen Bildergebnis
  die Gast-Bridge nicht passieren kann.
- `exec` wertet vom Modell generiertes JavaScript oder TypeScript in einem isolierten
  QuickJS-WASI-Worker-Thread aus.
- Jedes aktivierte, für den Katalog geeignete Tool (OpenClaw-Kern, Plugin, MCP, Client) wird als
  eigenständiges Modell-Tool verborgen und innerhalb des Gastprogramms über `ALL_TOOLS`
  und `tools` bereitgestellt.
- Die Beschreibung von `exec` enthält einen begrenzten Schnellindex exakter OpenClaw-/Plugin-
  Katalog-IDs, kompakte Eingabehinweise und kompakte deklarierte Ausgabehinweise, wenn ein
  vertrauenswürdiges Tool ein Ausgabeschema bereitstellt. Beschreibungen, vollständige Schemas,
  MCP-Einträge und überzählige Einträge werden ausgelassen; die Katalogsuche auf Gastseite bleibt die Ausweichlösung.
- Gastcode durchsucht den verborgenen Katalog, beschreibt das Schema eines Tools und ruft
  ein Tool über denselben Ausführungspfad auf, den normale Agent-Durchläufe verwenden (Richtlinien,
  Genehmigungen, Hooks und Telemetrie gelten weiterhin).
- MCP-Tools werden unter dem Namensraum `MCP` gruppiert; im Code-Modus ist dies die
  einzige unterstützte Möglichkeit, sie aufzurufen.
- `wait` setzt einen angehaltenen Code-Modus-Durchlauf fort, wenn verschachtelte Tool-Aufrufe noch
  ausstehen.

Der Code-Modus ändert nur die modellseitige Orchestrierungsoberfläche. Er ersetzt weder
Tools, Plugin-Tools, MCP-Tools, Authentifizierung, Genehmigungsrichtlinien und Kanal-
verhalten noch die Modellauswahl.

## Gründe für die Verwendung

- Kleinere Prompt-Oberfläche: Provider erhalten zwei Steuerungs-Tools, einen begrenzten Index nativer Tools
  und nur die wenigen erforderlichen direkten Tools anstelle von Dutzenden oder Hunderten
  vollständiger Tool-Schemas.
- Bessere Orchestrierung: Das Modell kann Schleifen, Verknüpfungen, kleine Transformationen,
  bedingte Logik und parallele verschachtelte Tool-Aufrufe innerhalb einer Codezelle verwenden.
- Weniger Modell-Roundtrips: Ein deklarierter Ausgabevertrag ermöglicht dem Modell, ein Tool-Ergebnis in einem
  einzigen `exec` aufzurufen und zu transformieren; unbekannte Ausgaben bleiben zunächst unverarbeitet.
- Provider-neutral: Funktioniert für OpenClaw-, Plugin-, MCP- und Client-Tools, ohne
  von der nativen Codeausführung eines Providers abhängig zu sein.
- Sicheres Fehlschlagen: Wenn der Code-Modus aktiviert, die QuickJS-WASI-Laufzeit jedoch
  nicht verfügbar ist, schlägt der Durchlauf fehl, statt stillschweigend auf die breite direkte
  Bereitstellung von Tools zurückzufallen.

Am nützlichsten ist dies für Agenten mit einem großen aktivierten Tool-Katalog oder für Arbeitsabläufe, bei denen
das Modell mehrere Tools suchen, kombinieren und aufrufen muss, bevor es antwortet.

Behalten Sie die direkte Tool-Bereitstellung bei einem kleinen Katalog oder einem Modell bei, das nicht zuverlässig
kurze Programme schreibt. Verwenden Sie [Tool-Suche](/de/tools/tool-search), wenn Sie einen
kompakten Katalog wünschen, jedoch strukturierte Steuerungen zum Suchen, Beschreiben und Aufrufen gegenüber
dem QuickJS-WASI-Gast bevorzugen.

## Schnellstart

### Code-Modus aktivieren

```json5
{
  tools: {
    codeMode: {
      enabled: true,
    },
  },
}
```

Kurzform:

```json5
{
  tools: {
    codeMode: true,
  },
}
```

Der Code-Modus bleibt deaktiviert, wenn `tools.codeMode` ausgelassen wird, `false` ist oder ein Objekt
ohne `enabled: true` vorliegt.

Wenn Sie Sandbox-Agenten mit konfigurierten MCP-Servern verwenden, müssen Sie außerdem das
mitgelieferte MCP-Plugin in der Sandbox-Tool-Richtlinie zulassen, beispielsweise
`tools.sandbox.tools.alsoAllow: ["bundle-mcp"]`. Siehe
[Konfiguration – Tools und benutzerdefinierte Provider](/de/gateway/config-tools#mcp-and-plugin-tools-inside-sandbox-tool-policy).

Legen Sie für engere Grenzen explizite Limits fest:

```json5
{
  tools: {
    codeMode: {
      enabled: true,
      timeoutMs: 10000,
      memoryLimitBytes: 67108864,
      maxOutputBytes: 65536,
      maxSnapshotBytes: 10485760,
      maxPendingToolCalls: 16,
      snapshotTtlSeconds: 900,
      searchDefaultLimit: 8,
      maxSearchLimit: 50,
    },
  },
}
```

### Vorgehen des Modells

Für ein Tool mit einer deklarierten Ausgabe wie
`Array<{ id: string; paid: boolean; tons: number }>` kann ein Gastprogramm
es auswählen, aufrufen und transformieren:

```javascript
const [shipmentTool] = await tools.search("list shipments");
const shipments = await tools.callValue(shipmentTool.id, {});
return shipments.filter((shipment) => !shipment.paid && shipment.tons > 10);
```

Wenn eine Schnellindexzeile mit `-> ?` endet, ist die Ausgabeform unbekannt. Das erste
`exec` muss `await tools.callValue(...)` unverändert zurückgeben. Ein späteres `exec` kann
den beobachteten Wert transformieren. Dies kostet einen zusätzlichen Modelldurchlauf, verhindert jedoch, dass das
Modell Feldnamen errät.

### Aktive Oberfläche überprüfen

Um die Form der Modellnutzlast bei der Fehlerdiagnose zu bestätigen, führen Sie das Gateway mit
gezielter Protokollierung aus:

```bash
OPENCLAW_DEBUG_CODE_MODE=1 \
OPENCLAW_DEBUG_MODEL_TRANSPORT=1 \
OPENCLAW_DEBUG_MODEL_PAYLOAD=tools \
openclaw gateway
```

Bei aktivem Code-Modus sollten die protokollierten modellseitigen Tool-Namen `exec` und
`wait` lauten. Für die vollständige geschwärzte Provider-Nutzlast fügen Sie für eine
kurze Debugging-Sitzung `OPENCLAW_DEBUG_MODEL_PAYLOAD=full-redacted` hinzu.

## Swarm zur Agenten-Auffächerung verwenden

[Swarm](/de/tools/swarm) fügt die Gast-Globals `agents.run()`, `phase()` und `log()`
zur Orchestrierung gleichzeitiger Unteragenten aus Code-Modus-Skripten hinzu. Aktivieren Sie sowohl
`tools.codeMode` als auch `tools.swarm` und verwenden Sie anschließend den normalen JavaScript-Kontrollfluss für
Auffächerung, Entscheidungsschranken und strukturierte Erfassung. Swarm ist eine separate optionale
Schranke; die alleinige Aktivierung des Code-Modus stellt die API `agents.*` nicht bereit.

## Technischer Überblick

Der restliche Teil dieser Seite behandelt den Laufzeitvertrag und Implementierungsdetails
für Maintainer, Plugin-Autoren bei der Fehlerdiagnose der Tool-Bereitstellung und Betreiber,
die risikoreiche Bereitstellungen validieren.

## Laufzeitstatus

|                     |                                                                                             |
| ------------------- | ------------------------------------------------------------------------------------------- |
| Laufzeit            | [`quickjs-wasi`](https://github.com/vercel-labs/quickjs-wasi)                               |
| Standardzustand     | deaktiviert                                                                                    |
| Stabilität          | experimentelle OpenClaw-Oberfläche (Codex Code Mode ist eine separate, stabile Codex-Umgebungsoberfläche) |
| Zieloberfläche      | generische OpenClaw-Agent-Durchläufe                                                                 |
| Sicherheitsannahme  | Modellcode ist feindlich                                                                       |
| Zusage an Benutzer  | die Aktivierung des Code-Modus fällt niemals stillschweigend auf die breite direkte Tool-Bereitstellung zurück                  |

## Umfang

Der Code-Modus ist für die modellseitige Orchestrierungsform eines vorbereiteten Durchlaufs zuständig. Er
ist nicht für Modellauswahl, Kanalverhalten, Authentifizierung, Tool-Richtlinien oder Tool-
Implementierungen zuständig.

Im Umfang enthalten: für das Modell sichtbare Definitionen von Steuerungs-/direkten Tools, Aufbau des verborgenen Tool-Katalogs,
Ausführung von JavaScript-/TypeScript-Gastcode, der QuickJS-WASI-Worker-
Laufzeit, Host-Callbacks zum Suchen/Beschreiben/Aufrufen, fortsetzbarer Zustand für
angehaltene Gastprogramme, Grenzen für Ausgabe/Timeout/Arbeitsspeicher/ausstehende Aufrufe/Snapshots
sowie Telemetrie-/Verlaufsprojektion für verschachtelte Tool-Aufrufe.

Nicht im Umfang enthalten: Provider-native entfernte Codeausführung, Semantik der Shell-Ausführung,
Änderungen an bestehender Tool-Autorisierung, persistente benutzererstellte
Skripte, Zugriff auf Paketmanager/Dateien/Netzwerk/Module im Gastcode und die direkte
Wiederverwendung interner Komponenten von Codex Code Mode.

Provider-eigene Tools wie entfernte Python-Sandboxes sind separate Tools. Siehe
[Codeausführung](/de/tools/code-execution).

## Begriffe

- **Code-Modus**: der OpenClaw-Laufzeitmodus, der katalogkompatible Modell-
  Tools verbirgt und `exec`, `wait` sowie erforderliche ausschließlich direkte Tools bereitstellt.
- **Gastlaufzeit**: die QuickJS-WASI-JavaScript-VM, die Modellcode auswertet.
- **Host-Bridge**: die schmale JSON-kompatible Callback-Oberfläche vom Gastcode
  zurück zu OpenClaw.
- **Katalog**: die auf den Durchlauf beschränkte Liste effektiver Tools nach der normalen Auflösung von Tool-
  Richtlinien, Plugins, MCP und Client-Tools.
- **Verschachtelter Tool-Aufruf**: ein Tool-Aufruf, der aus Gastcode über die Host-
  Bridge erfolgt.
- **Snapshot**: serialisierter Zustand der QuickJS-WASI-VM, der gespeichert wird, damit `wait`
  einen angehaltenen Code-Modus-Durchlauf fortsetzen kann.

## Konfiguration

`tools.codeMode.enabled` ist die Aktivierungsschranke; das Festlegen anderer Felder
aktiviert die Funktion nicht selbstständig.

| Feld                  | Standard                       | Begrenzung                                      |
| --------------------- | ------------------------------ | ----------------------------------------------- |
| `enabled`             | `false`                        | boolesch; nur `true` aktiviert den Code-Modus          |
| `runtime`             | `"quickjs-wasi"`               | einziger unterstützter Wert                            |
| `mode`                | `"only"`                       | stellt Steuerungs-/direkte Tools bereit und katalogisiert den Rest |
| `languages`           | `["javascript", "typescript"]` | beliebige Teilmenge der beiden                           |
| `timeoutMs`           | `10000`                        | `100`-`60000`                                   |
| `memoryLimitBytes`    | `67108864`                     | `1048576`-`1073741824`                          |
| `maxOutputBytes`      | `65536`                        | `1024`-`10485760`                               |
| `maxSnapshotBytes`    | `10485760`                     | `1024`-`268435456`                              |
| `maxPendingToolCalls` | `16`                           | `1`-`128`                                       |
| `snapshotTtlSeconds`  | `900`                          | `1`-`86400`                                     |
| `searchDefaultLimit`  | `8`                            | auf `maxSearchLimit` begrenzt                     |
| `maxSearchLimit`      | `50`                           | `1`-`50`                                        |

Wenn der Code-Modus aktiviert ist, QuickJS-WASI jedoch nicht geladen werden kann, schlägt OpenClaw
für diesen Durchlauf sicher fehl; normale Tools werden nicht stillschweigend als Ausweichlösung bereitgestellt.

## Aktivierung

Der Code-Modus wird ausgewertet, nachdem die effektive Tool-Richtlinie bekannt ist und bevor die
endgültige Modellanforderung zusammengestellt wird:

1. Lösen Sie die Richtlinie für Agent, Modell, Provider, Sandbox, Kanal, Absender und Ausführung
   auf.
2. Erstellen Sie die effektive OpenClaw-Werkzeugliste und fügen Sie zulässige Plugin-, MCP- und
   Client-Werkzeuge hinzu.
3. Wenden Sie die Zulassungs-/Ablehnungsrichtlinie an.
4. Wenn `tools.codeMode.enabled` false ist, fahren Sie mit der normalen Werkzeugbereitstellung fort.
5. Wenn die Funktion aktiviert ist und Werkzeuge für die Ausführung aktiv sind, behalten Sie erforderliche, ausschließlich direkte
   Werkzeuge bei und registrieren Sie jedes katalogfähige effektive Werkzeug im Code-Mode-
   Katalog.
6. Entfernen Sie die katalogisierten Werkzeuge aus der für das Modell sichtbaren Liste; fügen Sie `exec` und
   `wait` neben den beibehaltenen, ausschließlich direkten Werkzeugen hinzu.

Ausführungen, die absichtlich keine Werkzeuge haben (direkte Modellaufrufe, `disableTools: true`
oder eine leere `tools.allow`-Liste), aktivieren die Code-Mode-Oberfläche nicht, selbst
wenn `tools.codeMode.enabled: true` konfiguriert ist. Code Mode und OpenClaw Tool
Search schließen sich für eine Ausführung gegenseitig aus; wenn Code Mode aktiviert wird, findet die
Compaction von Tool Search nicht statt.

Der Code-Mode-Katalog gilt nur für die jeweilige Ausführung und darf keine Werkzeuge eines anderen
Agenten, einer anderen Sitzung, eines anderen Absenders oder einer anderen Ausführung offenlegen.

## Für das Modell sichtbare Werkzeuge

Wenn Code Mode aktiv ist, sieht das Modell `exec`, `wait` und alle erforderlichen,
ausschließlich direkten Werkzeuge. Jedes andere aktivierte Werkzeug wird aus der für das Modell bestimmten
Werkzeugliste ausgeblendet und im Code-Mode-Katalog registriert.

Verwenden Sie `exec` für die Werkzeugorchestrierung, das Zusammenführen von Daten, Schleifen, parallele verschachtelte Aufrufe
und strukturierte Transformationen. Verwenden Sie `wait` nur, wenn `exec` ein fortsetzbares
`waiting`-Ergebnis zurückgibt.

## `exec`

`exec` startet eine Code-Mode-Zelle und gibt ein Ergebnis zurück. Der Eingabecode wird vom Modell
generiert und muss als feindselig behandelt werden.

Eingabe:

```typescript
type CodeModeExecInput = {
  code?: string;
  command?: string;
  language?: "javascript" | "typescript";
};
```

Regeln:

- Eines von `code` oder `command` darf nicht leer sein.
- `code` ist das dokumentierte, für das Modell sichtbare Feld.
- `command` wird als exec-kompatibler Alias für Hook-Richtlinien und
  vertrauenswürdige Umschreibungen akzeptiert (das normale OpenClaw-Shell-exec-Werkzeug verwendet ebenfalls ein `command`-
  Feld); wenn beide vorhanden sind, müssen die Werte übereinstimmen.
- `language` verwendet standardmäßig `"javascript"`; das Schema stellt es als flache
  String-Aufzählung (`"javascript" | "typescript"`) bereit, nicht als `oneOf`- bzw. `anyOf`-Union,
  da einige Provider solche Strukturen ablehnen.
- Wenn `language` den Wert `"typescript"` hat, transpiliert OpenClaw vor der Auswertung.
- `exec` lehnt `import`, `require`, dynamische Importe und Module-Loader-
  Muster ab.
- `exec` stellt die normale Shell-Implementierung von `exec` niemals rekursiv bereit.
- Äußere Code-Mode-Hook-Ereignisse für `exec` enthalten `toolKind: "code_mode_exec"` und
  `toolInputKind: "javascript" | "typescript"` (sofern bekannt), damit Richtlinien
  Code-Mode-Zellen von Shell-artigen `exec`-Aufrufen unterscheiden können, die denselben
  Werkzeugnamen verwenden.

Ergebnis:

```typescript
type CodeModeResult = CodeModeCompletedResult | CodeModeWaitingResult | CodeModeFailedResult;

type CodeModeCompletedResult = {
  status: "completed";
  value: unknown;
  output?: CodeModeOutput[];
  telemetry: CodeModeTelemetry;
};

type CodeModeWaitingResult = {
  status: "waiting";
  runId: string;
  reason: "pending_tools" | "yield";
  pendingToolCalls?: CodeModePendingToolCall[];
  output?: CodeModeOutput[];
  telemetry: CodeModeTelemetry;
};

type CodeModeFailedResult = {
  status: "failed";
  error: string;
  code?: CodeModeErrorCode;
  output?: CodeModeOutput[];
  telemetry: CodeModeTelemetry;
};
```

`exec` gibt `waiting` zurück, wenn der Gast mit fortsetzbarem Zustand angehalten wird, der noch
eine für das Modell sichtbare Fortsetzung benötigt – ein explizites `yield_control(...)` oder einen
Bridge-Werkzeugaufruf, der nicht innerhalb der exec-Frist aufgelöst wurde. Das Ergebnis
enthält eine `runId` für `wait`. Bridge-Werkzeugaufrufe – `tools.search`/`describe`/
`call` sowie Namespace-Aufrufe einschließlich MCP-Namespace-Aufrufen – werden automatisch
innerhalb desselben `exec`-/`wait`-Aufrufs abgearbeitet, solange sie innerhalb der Frist aufgelöst werden. Dadurch wird ein
kompakter Codeblock, der mehrere Werkzeuge erwartet, in einem Modell-
Durchlauf abgeschlossen, statt einen Modell-Werkzeugaufruf pro await zu erzwingen. Neustartsichere Ausführungen werden niemals
automatisch abgearbeitet; ihre ausstehenden Aufgaben durchlaufen weiterhin die wiederholungssicheren Prüfungen.

`exec` gibt `completed` nur zurück, wenn die Gast-VM keine ausstehenden Aufgaben hat und der
endgültige Wert nach Ausführung des OpenClaw-Ausgabeadapters JSON-kompatibel ist.

## `wait`

`wait` setzt eine angehaltene Code-Mode-VM fort.

Eingabe:

```typescript
type CodeModeWaitInput = {
  runId: string;
};
```

Die Ausgabe ist dieselbe `CodeModeResult`-Union, die von `exec` zurückgegeben wird.

`wait` existiert, weil verschachtelte OpenClaw-Werkzeuge langsam, interaktiv oder genehmigungspflichtig sein
oder Teilaktualisierungen streamen können; das Modell sollte keinen langen
`exec`-Aufruf offen halten müssen, während der Host auf externe Aufgaben wartet.

QuickJS-WASI-Snapshot/-Wiederherstellung ist der Fortsetzungsmechanismus:

1. `exec` wertet Code bis zum Abschluss, Fehlschlag oder Anhalten aus.
2. Beim Anhalten erstellt OpenClaw einen Snapshot der QuickJS-VM und zeichnet ausstehende Host-
   Aufgaben auf.
3. Wenn ausstehende Aufgaben abgeschlossen sind, stellt `wait` den VM-Snapshot wieder her und
   registriert Host-Callbacks anhand stabiler Namen erneut.
4. OpenClaw übermittelt Ergebnisse verschachtelter Werkzeuge an die wiederhergestellte VM und arbeitet
   ausstehende QuickJS-Aufgaben ab.
5. `wait` gibt `completed`, `failed` oder ein weiteres `waiting`-Ergebnis zurück.

Snapshots sind Laufzeitzustand und keine Benutzerartefakte: Sie befinden sich ausschließlich in einer
prozessinternen Map (kein Schreiben in Datenbank oder auf Datenträger), sind größenbegrenzt, laufen ab und sind
auf die Ausführung und Sitzung beschränkt, in denen sie erstellt wurden.

`wait` schlägt (als `failed`-Ergebnis) fehl, wenn:

- `runId` unbekannt oder sein Snapshot bereits abgelaufen ist.
- sich der Aufrufer nicht im selben Ausführungs-/Sitzungsbereich wie die angehaltene Ausführung befindet.
- für diese `runId` bereits eine `wait` ausgeführt wird.
- die QuickJS-WASI-Wiederherstellung fehlschlägt.
- die Fortsetzung `maxOutputBytes` oder `maxSnapshotBytes` überschreiten würde.

## Gast-Laufzeit-API

```typescript
declare const ALL_TOOLS: ToolCatalogEntry[];
declare const tools: ToolCatalog;
declare const MCP: Record<string, unknown>;
declare const namespaces: Record<string, unknown>;

declare function text(value: unknown): void;
declare function json(value: unknown): void;
declare function yield_control(reason?: string): Promise<void>;
```

`ALL_TOOLS` enthält kompakte Metadaten für den ausführungsbezogenen Katalog; vollständige
Schemata sind standardmäßig nicht enthalten. Die für das Modell sichtbare Beschreibung von `exec` enthält außerdem eine
begrenzte, deterministische Teilmenge exakter OpenClaw-/Plugin-IDs, kompakter Eingabe-
Hinweise und vertrauenswürdiger deklarierter Ausgabehinweise. Beschreibungen bleiben zurückgestellt, damit
feindselige Katalogtexte das Modell nicht beeinflussen können. Wenn dieses Verzeichnis ein Werkzeug auslässt,
lesen Sie `ALL_TOOLS` oder rufen Sie `tools.search(...)` innerhalb des Gastprogramms auf.

Der Pfeil in jeder Zeile des Schnellverzeichnisses beschreibt den Wert `tools.callValue(...)`.
`-> Array<{ id: string }>` ist ein deklarierter Ausgabehinweis; `-> ?` bedeutet, dass die Ausgabe unbekannt ist.
Bei unbekannten Ausgaben hat der Rohwert Vorrang: Geben Sie den Wert unverändert zurück, prüfen Sie ihn und
filtern oder transformieren Sie ihn anschließend in einem späteren `exec`, anstatt Feldnamen zu erraten. Dies
gilt auch, wenn ein Lesevorgang mit deklarierter Ausgabe einen abschließenden `-> ?`-Aufruf speist: Geben Sie den
Rohwert dieses Aufrufs zurück, ohne ihn in die gewünschte Antwortstruktur einzubetten.

```typescript
type ToolCatalogEntry = {
  id: string;
  name: string;
  label?: string;
  description: string;
  source: "openclaw" | "mcp" | "client";
  sourceName?: string;
  input: string;
  output?: string;
};
```

`input` ist eine begrenzte TypeScript-artige Signatur für den häufigsten Fall. Verwenden Sie
`tools.describe(...)`, wenn weiterhin das exakte vollständige Schema benötigt wird. Entfernte MCP-
und Client-Einträge verwenden `input: "unknown"`, damit ihre nicht vertrauenswürdigen Schemata bis
`describe` zurückgestellt bleiben. `output` ist
nur für einen vollständigen kompakten Hinweis vorhanden, der aus einem vertrauenswürdigen OpenClaw-Core-
oder Plugin-`outputSchema` abgeleitet wurde. Behauptungen von MCP und Clients zu Ausgabeschemata werden nicht
in diesen vertrauenswürdigen Kataloghinweis übernommen.

Plugin-Werkzeuge verwenden `source: "openclaw"`, wobei `sourceName` auf die ID des besitzenden
Plugins gesetzt ist; es gibt keinen separaten `"plugin"`-Quellwert. `source: "mcp"` wird
nur für MCP-Einträge in den Metadaten `sourceName`/`mcp` verwendet (und wird aus
`ALL_TOOLS`/`tools.*` herausgefiltert, siehe unten).

Das vollständige Schema wird nur bei Bedarf geladen:

```typescript
type ToolCatalogEntryWithSchema = ToolCatalogEntry & {
  parameters: unknown;
  outputSchema?: unknown;
};
```

Kataloghilfsfunktionen:

```typescript
type ToolCatalog = {
  search(query: string, options?: { limit?: number }): Promise<ToolCatalogEntry[]>;
  describe(id: string): Promise<ToolCatalogEntryWithSchema>;
  callValue(id: string, input?: unknown): Promise<unknown>;
  call(id: string, input?: unknown): Promise<unknown>;
  [safeToolName: string]: unknown;
};
```

Komfort-Werkzeugfunktionen werden nur für eindeutige, sichere Namen installiert:

```typescript
const files = await tools.search("read local file");
const fileRead = await tools.describe(files[0].id);
const content = await tools.callValue(fileRead.id, { path: "README.md" });

// Wenn der ausgeblendete Katalog einen eindeutigen `web_search`-Eintrag enthält:
const hits = await tools.web_search({ query: "OpenClaw code mode" });
```

`tools.callValue(...)` gibt den JSON-Wert `details` eines normalen Werkzeugs direkt zurück.
`tools.call(...)` bewahrt den unverarbeiteten `{ tool, result }`-Umschlag für Aufrufer auf,
die Inhaltsblöcke oder andere Ergebnismetadaten benötigen.

## Deklarierte Ausgabeverträge

OpenClaw-Werkzeuge können `outputSchema` für den strukturierten Wert deklarieren, der in
`AgentToolResult.details` abgelegt wird. Dies ist für Code Mode und Tool Search nützlich; es ist
kein Provider-natives Werkzeugantwortschema und ändert die direkte
Werkzeugbereitstellung nicht.

Deklarieren Sie für ein mit `defineToolPlugin` erstelltes Werkzeug das Schema neben
`parameters`:

```typescript
import { Type } from "typebox";
import { defineToolPlugin } from "openclaw/plugin-sdk/tool-plugin";

const Shipment = Type.Object(
  {
    id: Type.String(),
    paid: Type.Boolean(),
    tons: Type.Number(),
  },
  { additionalProperties: false },
);

export default defineToolPlugin({
  id: "shipping",
  name: "Shipping",
  description: "Shipment tools.",
  tools: (tool) => [
    tool({
      name: "shipping_list",
      description: "List shipments.",
      parameters: Type.Object({}),
      outputSchema: Type.Array(Shipment),
      execute: async () => loadShipments(),
    }),
  ],
});
```

Für `api.registerTool(...)` oder ein Factory-Werkzeug setzen Sie dieselbe `outputSchema`-
Eigenschaft auf das zurückgegebene `AnyAgentTool`-Objekt.

Zu den aktuellen integrierten Verträgen gehören `agents_list`, `apply_patch`,
`conversations_list`, `conversations_send`, `conversations_turn`, `edit`,
`openclaw`, `read`, `screen`,
`sessions_history`, `sessions_list`, `sessions_search`, `sessions_send`,
`session_status`, `spawn_task`, `terminal`, `web_fetch` und `web_search`.
Exakte Durchleitungen können das Protokollschema ihres Eigentümers
wiederverwenden, statt einen ausschließlich für das Modell bestimmten Vertrag zu
duplizieren. Beispielsweise stellen die Konversationswerkzeuge dieselben
Gateway-Ergebnisschemata bereit, die von `conversations.list`,
`conversations.send` und `conversations.turn` verwendet werden; `web_fetch` besitzt ein
werkzeuglokales Schema, dessen Hinweis stabile Metadaten, Text, Cache-Status und
verschachtelte Auslagerungsmetadaten bereitstellt; `web_search` deklariert seine exakte
normalisierte Vereinigung aus Ergebnissen/Antworten/Fehlern/Rohdaten als vollständigen
Schnellindex-Hinweis. Dateisystemverträge geben strukturierte Ergebnisse für
gelesenen Text, Bilder, Kürzungen und optional nicht gefundene Elemente zurück;
außerdem einen expliziten Bearbeitungsänderungsstatus samt Diff-/Patch-Daten sowie
Pfadzusammenfassungen für angewendete Patches. Wenn der Schnellindex die Felder
deklariert, kann eine Zelle Ermittlung und Zustellung ohne einen separaten
Inspektionsdurchlauf kombinieren:

```javascript
const listed = await tools.conversations_list({ query: "build bot" });
const target = listed.conversations.find((item) => item.label === "Build bot");
if (!target) throw new Error("Konversation nicht gefunden");
return await tools.conversations_send({
  conversationRef: target.conversationRef,
  message: "Build abgeschlossen.",
});
```

Für die verschachtelten Aufrufe gelten weiterhin die normalen
Werkzeugrichtlinien, Hooks und Genehmigungen. Wenn ein vollständiger Vertrag exakt,
aber für den begrenzten Schnellindex zu groß ist, bleibt er über
`tools.describe(...)` verfügbar und der Pfeil bleibt `-> ?`.

Die Vertragsregeln sind strikt:

- Beschreiben Sie den exakten JSON-kompatiblen Wert `details`, nicht gerenderte
`content`-Blöcke oder eine Provider-Hülle.
- Nehmen Sie jede nicht auslösende Erfolgs- oder Fehlervariante auf. Lassen Sie
`outputSchema` weg, wenn das Werkzeug kein stabiles strukturiertes Ergebnis besitzt.
- Schließen Sie Objektebenen mit `{ additionalProperties: false }`, um einen vollständigen
  Schnellindex-Hinweis zu erhalten. Offene, übergroße oder anderweitig partielle
  Schemata bleiben über `tools.describe(...)` verfügbar, ermöglichen jedoch keine
  Feldverwendung in einem Durchlauf.
- OpenClaw kompiliert das Schema vor der Ausführung des Werkzeugs und validiert
  anschließend das endgültige `details` nach den normalen Werkzeug-Hooks und bevor
  ein Katalogaufruf zurückkehrt. Bei einem ungültigen Schema kann das Werkzeug
  nicht ausgeführt werden; eine Abweichung schlägt fehl, ohne den Wert auszugeben.
- Kompakte Hinweise sind deterministisch und begrenzt. `tools.describe(...)` stellt
  das vollständige vertrauenswürdige Schema bereit, wenn der kompakte Hinweis nicht ausreicht.
- Installierter Plugin-Code ist bereits vertrauenswürdiger lokaler Code. Metadaten
  von entfernten MCPs und Clients bleiben nicht vertrauenswürdig und können diese
  Schnellindex-Hinweise nicht aktivieren.

Details zur Plugin-Entwicklung finden Sie unter [Werkzeug-Plugins](/de/plugins/tool-plugins#output-contracts).

MCP-Katalogeinträge können im Codemodus nicht über `tools.callValue(...)`,
`tools.call(...)` oder Komfortfunktionen aufgerufen werden; sie werden
ausschließlich über den generierten Namespace `MCP` bereitgestellt.
Deklarationsdateien im TypeScript-Stil sind über die schreibgeschützte virtuelle
Dateioberfläche `API` verfügbar, sodass Agenten MCP-Signaturen prüfen
können, ohne MCP-Schemata zum Prompt hinzuzufügen:

```typescript
const files = await API.list("mcp");
const githubApi = await API.read("mcp/github.d.ts");

const issue = await MCP.github.createIssue({
  owner: "openclaw",
  repo: "openclaw",
  title: "Gateway-Protokolle untersuchen",
});

const snapshot = await MCP.chromeDevtools.takeSnapshot({ output: "markdown" });
const resource = await MCP.docs.resources.read({ uri: "memo://one" });
const prompt = await MCP.docs.prompts.get({
  name: "brief",
  arguments: { topic: "release" },
});
```

`API.read("mcp/<server>.d.ts")` gibt kompakte, aus den MCP-Werkzeugmetadaten
abgeleitete Deklarationen zurück:

```typescript
type McpToolResult = {
  content?: unknown[];
  structuredContent?: unknown;
  isError?: boolean;
  [key: string]: unknown;
};

declare namespace MCP.github {
  /** Diesen API-Header im TypeScript-Stil zurückgeben. */
  function $api(toolName?: string, options?: { schema?: boolean }): Promise<McpApiHeader>;

  /**
   * Ein GitHub-Issue erstellen.
   * @param owner Repository-Eigentümer
   * @param repo Repository-Name
   * @param title Issue-Titel
   */
  function createIssue(input: {
    owner: string;
    repo: string;
    title: string;
    body?: string;
  }): Promise<McpToolResult>;
}
```

Deklarationsdateien sind virtuell und werden nicht im Workspace- oder
Statusverzeichnis gespeichert. Für jeden Codemodus-Aufruf von
`exec` erstellt OpenClaw den laufbezogenen Werkzeugkatalog,
behält die sichtbaren MCP-Einträge bei, rendert `mcp/index.d.ts` sowie je
ein `mcp/<server>.d.ts` pro sichtbarem Server und injiziert diese kleine
schreibgeschützte Tabelle in den QuickJS-Worker. Gastcode sieht nur das Objekt
`API`: `API.list(prefix?)` gibt Dateimetadaten zurück und
`API.read(path)` den Inhalt der ausgewählten Deklaration. Unbekannte Pfade
und Segmente vom Typ `.`/`..` werden abgelehnt.

Dadurch bleiben große MCP-Schemata aus dem Modell-Prompt heraus: Der Agent erfährt
aus der Werkzeugbeschreibung `exec`, dass die virtuelle API existiert,
liest nur die benötigte Deklarationsdatei und ruft anschließend
`MCP.<server>.<tool>()` mit einem Objektargument auf.
`MCP.<server>.$api()` bleibt als Inline-Ausweichlösung für eine
Einzelwerkzeug-Schemaantwort innerhalb des Programms verfügbar.

Die Gastlaufzeit sieht Hostobjekte niemals direkt. Eingaben und Ausgaben
überqueren die Brücke als JSON-kompatible Werte mit expliziten Größenbegrenzungen.

## Interne Namespaces

Interne Namespaces stellen dem Codemodus eine kompakte Domänen-API bereit, ohne
weitere für das Modell sichtbare Werkzeuge hinzuzufügen. Eine dem Loader
gehörende Integration registriert einen Namespace wie `Issues` oder
`Calendar`; der Gastcode ruft diesen Namespace anschließend innerhalb des
QuickJS-Programms auf, während das Modell weiterhin nur die kompakte
Steuerungs-/Direktoberfläche sieht.

Namespaces sind derzeit intern. Es gibt keine öffentliche Namespace-API im
Plugin-SDK: Externe Plugin-Namespaces benötigen einen dem Loader gehörenden
Vertrag, damit Plugin-Identität, installierte Manifeste, Authentifizierungsstatus
und zwischengespeicherte Katalogdeskriptoren nicht von den Plugin-Werkzeugen
abweichen können, die dem Namespace zugrunde liegen. Der zentrale Codemodus ist
nur für Sandbox, Serialisierung, Katalogzugriffskontrolle und
Brückenweiterleitung zuständig.

Gastcode kann entweder die direkte globale Variable oder die Zuordnung
`namespaces` verwenden:

```javascript
const open = await Issues.list({ state: "open" });
const alsoOpen = await namespaces.Issues.list({ state: "open" });
return { count: open.length, alsoCount: alsoOpen.length };
```

### Lebenszyklus der Registry

Die Namespace-Registry ist prozesslokal und nach Namespace-ID indiziert:

1. Ein vertrauenswürdiger Loader ruft `registerCodeModeNamespaceForPlugin(pluginId, registration)` auf.
2. Der Codemodus erstellt das verborgene `ToolSearchRuntime` für den Lauf und liest
   dessen laufbezogenen Katalog.
3. `createCodeModeNamespaceRuntime(ctx, catalog)` behält nur Registrierungen bei,
   deren `requiredToolNames` sämtlich sichtbar sind und demselben `pluginId` gehören.
4. Jeder sichtbare Namespace ruft `createScope(ctx)` für den aktuellen Lauf auf
   und erhält dabei Laufkontext wie `agentId`, `sessionKey`,
   `sessionId`, `runId`, Konfiguration und Abbruchstatus.
5. Bereichsdaten werden in einen einfachen Deskriptor serialisiert und als direkte
   globale Variablen sowie als `namespaces.<globalName>` in QuickJS injiziert.
6. Gastaufrufe werden über die Worker-Brücke angehalten, lösen den Namespace-Pfad
   auf dem Host auf, ordnen den Aufruf einem deklarierten, dem Plugin gehörenden
   Katalogwerkzeug zu und führen dieses Werkzeug über `ToolSearchRuntime.callExactId` aus.
7. Bereite Namespace-Brückenaufrufe werden innerhalb des aktiven
   `exec`-/`wait`-Aufrufs automatisch abgearbeitet; wenn beim Timeout
   noch Namespace-Arbeit aussteht oder der Gast explizit abgibt, setzt
   `wait` dieselbe Namespace-Laufzeit später fort.
8. Beim Zurücksetzen oder Deinstallieren eines Plugins wird
   `clearCodeModeNamespacesForPlugin(pluginId)` aufgerufen, damit veraltete globale Variablen einen
   fehlgeschlagenen Plugin-Ladevorgang nicht überdauern.

Namespace-Aufrufe sind Katalogwerkzeugaufrufe: Für sie gelten dieselben
Richtlinien-Hooks, Genehmigungen, Abbruchbehandlungen, Telemetrie,
Transkriptprojektionen und Anhalte-/Fortsetzungsverhalten wie für
`tools.call(...)`.

### Registrierungsstruktur

Registrieren Sie Namespaces aus der Integration heraus, der die zugrunde
liegenden Werkzeuge gehören. Halten Sie den Bereich klein und stellen Sie nur
Domänenoperationen bereit, die deklarierten Katalogwerkzeugen zugeordnet sind.

```typescript
import {
  createCodeModeNamespaceTool,
  registerCodeModeNamespaceForPlugin,
} from "../agents/code-mode-namespaces.js";

const pluginId = "github";

registerCodeModeNamespaceForPlugin(pluginId, {
  id: "github-issues",
  globalName: "Issues",
  description: "GitHub-Issue-Hilfsfunktionen für das aktuelle Repository.",
  requiredToolNames: ["github_list_issues", "github_update_issue"],
  prompt: "Verwenden Sie Issues.list(params) und Issues.update(number, patch).",
  createScope: (ctx) => ({
    repository: ctx.config,
    list: createCodeModeNamespaceTool("github_list_issues", ([params]) => params ?? {}),
    update: createCodeModeNamespaceTool("github_update_issue", ([number, patch]) => ({
      number,
      patch,
    })),
  }),
});
```

`createCodeModeNamespaceTool(toolName, inputMapper)` kennzeichnet ein Bereichselement als
aufrufbare Namespace-Funktion. Das optionale `inputMapper` empfängt die
Gastargumente und gibt das Eingabeobjekt für das zugrunde liegende
Katalogwerkzeug zurück; ohne diese Funktion wird das erste Gastargument oder,
falls es fehlt, `{}` verwendet.

Unverarbeitete Hostfunktionen werden abgelehnt, bevor der Gastcode ausgeführt wird:

```typescript
createScope: () => ({
  // Falsch: Dies umgeht den Lebenszyklus des Katalogwerkzeugs und wird abgelehnt.
  list: async () => githubClient.listIssues(),
});
```

### Eigentümerschaft und Sichtbarkeit

Die Namespace-Eigentümerschaft ist an den `pluginId` des
Registrierungsaufrufers gebunden. `requiredToolNames` dient sowohl als
Sichtbarkeitsprüfung als auch als Prüfung der Eigentümerschaft:

- Jedes erforderliche Werkzeug muss im Laufkatalog vorhanden sein.
- Jedes erforderliche Werkzeug muss über `sourceName === pluginId` verfügen.
- Der Namespace wird ausgeblendet, wenn ein erforderliches Werkzeug fehlt oder
  einem anderen Plugin gehört.
- Jeder aufrufbare Pfad darf nur auf ein Werkzeug verweisen, das in
  `requiredToolNames` genannt ist.

Dies verhindert, dass ein anderes Plugin durch die Registrierung eines
gleichnamigen Werkzeugs einen Namespace bereitstellt, und hält Namespaces mit
den gewöhnlichen Agentenrichtlinien im Einklang: Wenn der Lauf die zugrunde
liegenden Werkzeuge nicht sehen kann, kann er auch den Namespace nicht sehen.

Beispielsweise sollte ein GitHub-Namespace hinter einem GitHub-eigenen Plugin
liegen, das für GitHub-Authentifizierung, REST-/GraphQL-Clients,
Ratenbegrenzungen, Schreibgenehmigungen und Tests zuständig ist. Der zentrale
Codemodus sollte keine GitHub-spezifischen APIs, Token-Verarbeitung oder
Provider-Richtlinien einbetten.

### Regeln für die Bereichsserialisierung

`createScope(ctx)` kann ein einfaches Objekt zurückgeben, das
JSON-kompatible Werte, Arrays, verschachtelte Objekte und
`createCodeModeNamespaceTool(...)`-Aufrufmarkierungen enthält. Hostobjekte gelangen niemals
direkt in QuickJS.

Der Serialisierer lehnt Folgendes ab:

- unverarbeitete Funktionen
- zyklische Objektgraphen
- unsichere Pfadsegmente: `__proto__`, `constructor`,
  `prototype`, leere Schlüssel oder Schlüssel, die das interne Pfadtrennzeichen enthalten
- `globalName`-Werte, die keine JavaScript-Bezeichner sind
- `globalName`-Kollisionen mit integrierten globalen Variablen des
  Codemodus wie `tools`, `namespaces`, `text`,
  `json`, `yield_control`, `MCP`, `API`,
  `ALL_TOOLS` oder `__openclaw*`

Werte, die sich nicht als JSON serialisieren lassen, werden vor dem Überqueren
der Brücke in JSON-sichere Ersatzwerte umgewandelt. Binärdaten, Handles,
Sockets, Clients und Klasseninstanzen sollten hinter gewöhnlichen
Katalogwerkzeugen verbleiben.

### Prompts

Der Namespace `description` und das optionale `prompt` werden nur
dann an das für das Modell sichtbare Schema `exec` angehängt, wenn
der Namespace für diesen Lauf sichtbar ist. Verwenden Sie sie, um die kleinste
nützliche Oberfläche zu vermitteln:

```typescript
{
  description: "Hilfsfunktionen für den Dienst zur Produktion fiktionaler Inhalte.",
  prompt:
    "Verwenden Sie Fictions.riskAudit(), Fictions.promoteIfReady(id, status) und Fictions.unpaidOver(amount).",
}
```

Beschränken Sie Prompts auf den Namespace-Vertrag, nicht auf die Authentifizierungseinrichtung, die Implementierungshistorie oder unabhängiges Plugin-Verhalten.

### Bereinigung

Namespaces sind prozesslokale Registrierungen. Entfernen Sie sie, wenn das besitzende Plugin deaktiviert, deinstalliert oder zurückgesetzt wird:

```typescript
clearCodeModeNamespacesForPlugin(pluginId);
```

Die Bereinigung im Codemodus liegt in der Verantwortung des Plugins; löschen Sie die Namespace-Registrierungen des Plugins, wenn dessen Lebenszyklus endet, anstatt für jeden Namespace eigene Handles zum Aufheben zu behalten. Tests können `clearCodeModeNamespacesForTest()` aufrufen, damit keine Registrierungen zwischen Testfällen bestehen bleiben.

### Testprüfliste

Namespace-Änderungen sollten die Sicherheitsgrenze und das Gastverhalten abdecken:

- Namespace-Prompttext wird nur angezeigt, wenn die zugrunde liegenden Tools sichtbar sind
- gleichnamige Tools aus einem anderen `sourceName` legen den Namespace nicht offen
- unverarbeitete Bereichsfunktionen werden abgelehnt
- gefälschte Namespace-IDs und gefälschte Pfade werden abgelehnt
- aufrufbare Pfade können nicht auf nicht deklarierte Tools verweisen
- verschachtelte Objekte und gemeinsam verwendete Referenzen werden korrekt serialisiert
- Namespace-Aufrufe werden über Katalog-Tools ausgeführt und geben JSON-sichere Details zurück
- Fehler können durch Gastcode abgefangen werden
- ausgesetzte Namespace-Aufrufe werden über `wait` fortgesetzt
- ein Plugin-Rollback löscht die zugehörigen Namespace-Registrierungen

Namespaces ergänzen den generischen `tools.search`-/`tools.call`-Katalog: Verwenden Sie den Katalog für beliebige aktivierte OpenClaw-, Plugin- und Client-Tools; verwenden Sie `MCP` für MCP-Tools; verwenden Sie andere Namespaces für Plugin-eigene, dokumentierte Domänen-APIs, bei denen prägnanter Code zuverlässiger als wiederholte Schemaabfragen ist.

## Ausgabe-API

- `text(value)` fügt dem Array `output` eine menschenlesbare Ausgabe hinzu.
- `json(value)` fügt nach einer JSON-kompatiblen Serialisierung ein strukturiertes Ausgabeelement hinzu.
- Der endgültige Rückgabewert des Gastcodes wird in einem `completed`-Ergebnis zu `value`.

```typescript
type CodeModeOutput = { type: "text"; text: string } | { type: "json"; value: unknown };
```

Regeln: Die Ausgabereihenfolge entspricht den Gastaufrufen; die Ausgabe wird durch `maxOutputBytes` begrenzt; nicht serialisierbare Werte werden in einfache Zeichenfolgen oder Fehler umgewandelt; Binärwerte werden nicht unterstützt. Bilder und Dateien werden über gewöhnliche OpenClaw-Tools übertragen, nicht über die Codemodus-Bridge.

## Tool-Katalog

Der verborgene Katalog enthält Tools nach der effektiven Richtlinienfilterung in dieser Reihenfolge: OpenClaw-Kern-Tools, gebündelte Plugin-Tools, externe Plugin-Tools, MCP-Tools und anschließend vom Client bereitgestellte Tools für den aktuellen Lauf.

Katalog-IDs sind innerhalb eines Laufs stabil und nach Möglichkeit über gleichwertige Tool-Sätze hinweg deterministisch. Tatsächliche Form:

```text
<source>:<owner>:<tool-name>
```

Dabei ist `<source>` entweder `openclaw`, `mcp` oder `client` (Plugin-Tools verwenden `openclaw` mit der Plugin-ID als `<owner>`; Kern-Tools verwenden `openclaw:core:*`). Beispiele:

```text
openclaw:core:message
openclaw:browser:browser_request
mcp:github:create_issue
client:app:select_file
```

Der Katalog lässt Codemodus-Steuerungs-Tools (`exec`, `wait`, `tool_search_code`, `tool_search`, `tool_describe`, `tool_call`) und ausschließlich direkt verfügbare Tools aus. Steuerungen dürfen nicht rekursiv über den Katalog aufgerufen werden; ausschließlich direkt verfügbare Tools bleiben für das Modell sichtbar, da ihre strukturierten Ergebnisse die QuickJS-Bridge nicht passieren können.

MCP-Einträge verbleiben im laufbezogenen Katalog, damit Richtlinien, Genehmigungen, Hooks, Telemetrie, Transkriptprojektion und exakte Tool-IDs gemeinsam mit der normalen Tool-Ausführung verwendet werden. Die für Gäste sichtbaren Ansichten `ALL_TOOLS`, `tools.search(...)`, `tools.describe(...)`, `tools.callValue(...)` und `tools.call(...)` lassen MCP-Einträge aus. Der generierte Namespace `MCP.<server>.<tool>({ ...input })` wird wieder zur exakten Katalog-ID aufgelöst und über denselben Ausführungspfad weitergeleitet.

## Interaktion mit der Tool-Suche

Der Codemodus ersetzt die OpenClaw-Modelloberfläche der Tool-Suche für Läufe, in denen er aktiv ist.

Wenn `tools.codeMode.enabled` wahr ist und der Codemodus aktiviert wird:

- OpenClaw stellt `tool_search_code`, `tool_search`, `tool_describe` oder `tool_call` nicht als für das Modell sichtbare Tools bereit.
- Dasselbe Katalogisierungskonzept wird in die Gastlaufzeit verlagert.
- Die Gastlaufzeit erhält kompakte `ALL_TOOLS`-Metadaten sowie Hilfsfunktionen zum Suchen, Beschreiben und Aufrufen von Nicht-MCP-Tools.
- MCP-Aufrufe verwenden anstelle von `tools.call(...)` den generierten Namespace `MCP` und dessen `$api()`-Header.
- Verschachtelte Aufrufe werden über denselben OpenClaw-Ausführungspfad weitergeleitet, den die Tool-Suche verwendet.

Weitere Informationen zur kompakten OpenClaw-Katalog-Bridge, die der Codemodus bei aktiven Läufen ersetzt, finden Sie unter [Tool-Suche](/de/tools/tool-search).

## Tool-Namen und Kollisionen

Das für das Modell sichtbare Tool `exec` ist das Codemodus-Tool. Wenn das normale OpenClaw-Shell-Tool `exec` aktiviert ist, wird es vor dem Modell verborgen und wie jedes andere Tool katalogisiert.

Innerhalb der Gastlaufzeit:

- `tools.call("openclaw:core:exec", input)` kann das Shell-Ausführungs-Tool aufrufen, wenn die Richtlinie dies zulässt.
- `tools.exec(...)` wird nur installiert, wenn der Shell-Ausführungs-Katalogeintrag einen eindeutigen sicheren Namen hat.
- das Codemodus-Tool `exec` ist niemals rekursiv über `tools` verfügbar.

Wenn zwei Tools zum selben sicheren Kurznamen normalisiert werden, lässt OpenClaw die Kurzfunktion aus und verlangt `tools.call(id, input)`.

## Verschachtelte Tool-Ausführung

Jeder verschachtelte Tool-Aufruf passiert die Host-Bridge und tritt erneut in OpenClaw ein. Dabei bleiben folgende Elemente erhalten: aktive Agenten-ID, Sitzungs-ID und -Schlüssel, Absender- und Kanalkontext, Sandbox-Richtlinie, Genehmigungsrichtlinie, Plugin-`before_tool_call`-Hooks, Abbruchsignal, Streaming-Aktualisierungen, sofern verfügbar, sowie Verlaufs-/Audit-Ereignisse.

Verschachtelte Aufrufe werden im Transkript als echte Tool-Aufrufe dargestellt, damit Support-Pakete zeigen, was geschehen ist. Die Darstellung identifiziert dabei den übergeordneten Codemodus-Tool-Aufruf und die verschachtelte Tool-ID.

Bis zu `maxPendingToolCalls` parallele verschachtelte Aufrufe sind zulässig.

## Lebenszyklus von Lauf und Snapshot

Jeder Codemodus-Lauf wird in einer prozessinternen Map erfasst, deren Schlüssel `runId` ist (keine Persistierung auf Festplatte oder in einer Datenbank). `exec`/`wait` geben einen von drei Ergebnisstatus zurück: `completed`, `waiting` oder `failed`.

- Ein `waiting`-Ergebnis speichert den QuickJS-Snapshot, ausstehende Bridge-Anfragen und Bereichsmetadaten (Agentenlauf-ID, Sitzungs-ID/-Schlüssel), bis `wait` den Lauf fortsetzt oder er abläuft.
- Ablauf, eine falsche Sitzung, ein falscher Lauf und unbekannte/bereits in Fortsetzung befindliche `runId`-Werte erzeugen keinen eigenen terminalen Status; sie werden als `failed`-Ergebnis (`code: "invalid_input"`) mit einer Meldung wie `code mode
run is unavailable or expired.` oder `code mode run belongs to a different
session.` ausgegeben.
- Der Snapshot eines Laufs wird aus der Map entfernt, sobald er den Status `completed` oder `failed` erreicht, oder beim Herunterfahren des Gateways verworfen (einen Neustart überlebt nichts: Dies ist flüchtiger Laufzeitzustand).
- Für schreibgeschützte Arbeit kann `exec` den Wert `restartSafe: true` festlegen. OpenClaw lehnt dann Katalogaufrufe mit Nebenwirkungen und Plugin-Namespaces vor der Ausführung ab und kennzeichnet ausgesetzte Ergebnisse als sicher wiederholbar. Wenn ein Neustart `wait` unterbricht, rekonstruiert die [Neustartwiederherstellung](/de/gateway/restart-recovery) den Turn aus dem Transkript, anstatt den prozesslokalen Snapshot wiederherzustellen. Der Wiederherstellungs-Turn selbst bleibt auf auditierte schreibgeschützte Kern-Tools und ausdrücklich als sicher wiederholbar gekennzeichnete Plugin-Tools beschränkt.
- OpenClaw begrenzt die Anzahl gleichzeitig ausgesetzter Läufe pro Prozess auf 64 und lehnt neue Aussetzungen über dieser Grenze mit `too many suspended code mode
runs.` ab.

Der Snapshot-Speicher wird durch `maxSnapshotBytes` pro Lauf, die oben genannte prozessbezogene Höchstzahl ausgesetzter Läufe und `snapshotTtlSeconds` begrenzt.

## QuickJS-WASI-Laufzeit

OpenClaw lädt `quickjs-wasi` als direkte Abhängigkeit im zuständigen Paket; es verwendet keine transitive Kopie, die für eine unabhängige Abhängigkeit installiert wurde.

Aufgaben der Laufzeit: das QuickJS-WASI-WebAssembly-Modul kompilieren/laden; für jeden Codemodus-Lauf oder jede Fortsetzung eine isolierte VM erstellen; Host-Callbacks unter stabilen Namen registrieren; Speicher- und Unterbrechungsgrenzen festlegen; JavaScript auswerten; ausstehende Aufträge abarbeiten; den Zustand ausgesetzter VMs als Snapshot speichern; Snapshots für `wait` wiederherstellen; VM-Handles und Snapshots nach terminalen Zuständen freigeben.

Die Laufzeit wird in einem Node.js-Worker-Thread außerhalb der Haupt-Ereignisschleife von OpenClaw ausgeführt. Eine Endlosschleife im Gast darf den Gateway-Prozess nicht unbegrenzt blockieren; der Unterbrechungshandler des Workers erzwingt das Zeitlimit unabhängig von der Kooperation des Gastcodes.

## TypeScript

Die TypeScript-Unterstützung ist lediglich eine Quelltransformation: Als Eingabe wird eine einzelne TypeScript-Codezeichenfolge akzeptiert; die Ausgabe ist eine JavaScript-Zeichenfolge, die von QuickJS-WASI ausgewertet wird. Es gibt keine Typprüfung, keine Modulauflösung und keine `import`/`require`. Diagnosen werden als `failed`-Ergebnisse zurückgegeben.

Der TypeScript-Compiler wird nur für TypeScript-Zellen verzögert geladen; reine JavaScript-Zellen und ein deaktivierter Codemodus laden ihn nie.

## Sicherheitsgrenze

Modellcode ist feindselig. Die Laufzeit setzt auf mehrschichtige Abwehr:

- führt QuickJS-WASI außerhalb der Haupt-Ereignisschleife in einem Worker-Thread aus
- lädt `quickjs-wasi` als direkte Abhängigkeit, nicht über Codex oder ein transitives Paket
- kein Dateisystem, Netzwerk, Unterprozess, Modulimport, keine Umgebungsvariablen und keine globalen Hostobjekte im Gast
- verwendet QuickJS-Speicher- und Unterbrechungsgrenzen sowie ein von einem übergeordneten Prozess durchgesetztes Echtzeitlimit
- erzwingt Grenzwerte für Ausgabe, Snapshots, Protokolle und ausstehende Aufrufe
- serialisiert Host-Bridge-Werte über einen eng begrenzten JSON-Adapter
- wandelt Hostfehler in einfache Gastfehler um, niemals in Objekte aus dem Host-Realm
- verwirft Snapshots bei Zeitüberschreitung, Abbruch, Sitzungsende oder Ablauf
- lehnt rekursiven Zugriff auf `exec`, `wait` und Steuerungs-Tools der Tool-Suche ab
- verhindert, dass Kollisionen von Kurznamen Katalog-Hilfsfunktionen verdecken

Die Sandbox ist eine Sicherheitsebene; Betreiber benötigen für risikoreiche Bereitstellungen möglicherweise dennoch eine Härtung auf Betriebssystemebene.

## Fehlercodes

```typescript
type CodeModeErrorCode =
  | "invalid_input"
  | "runtime_unavailable"
  | "timeout"
  | "output_limit_exceeded"
  | "snapshot_limit_exceeded"
  | "internal_error";
```

`invalid_input` umfasst ungültige `exec`-/`wait`-Argumente, deaktivierte Sprachen, abgelehnten Modulzugriff, Fehler bei der TypeScript-Transformation, unbekannte/abgelaufene/bereichsfremde `runId`-Werte und zu viele ausgesetzte Läufe. `runtime_unavailable` umfasst einen QuickJS-Worker, der nicht gestartet werden kann oder mit einem von null verschiedenen Status beendet wird.

An den Gast zurückgegebene Fehler sind einfache Daten; Host-`Error`-Instanzen, Stack-Objekte, Prototypen und Hostfunktionen werden nicht an QuickJS übergeben.

## Telemetrie

Das Feld `telemetry` jedes Ergebnisses meldet: die Größe des verborgenen Katalogs und eine Aufschlüsselung nach Quelle (`openclaw`-/`mcp`-/`client`-Anzahlen), kumulierte Anzahlen der Such-, Beschreibungs- und Aufrufvorgänge für den Katalog des Laufs sowie die für das Modell sichtbaren Tool-Namen (`exec`, `wait` und beibehaltene ausschließlich direkt verfügbare Tools).

Die Telemetrie darf keine Secrets, unverarbeiteten Umgebungswerte oder ungeschwärzten Tool-Eingaben enthalten, die über die bestehende OpenClaw-Verlaufsrichtlinie hinausgehen.

## Fehlerbehebung

Verwenden Sie gezielte Protokollierung des Modelltransports, wenn sich der Codemodus anders als ein normaler Tool-Lauf verhält:

```bash
OPENCLAW_DEBUG_CODE_MODE=1 \
OPENCLAW_DEBUG_MODEL_TRANSPORT=1 \
OPENCLAW_DEBUG_MODEL_PAYLOAD=tools \
OPENCLAW_DEBUG_SSE=events \
openclaw gateway
```

Für das Debugging der Payload-Struktur verwenden Sie `OPENCLAW_DEBUG_MODEL_PAYLOAD=full-redacted`.
Damit wird ein begrenzter, redigierter JSON-Snapshot der Modellanfrage protokolliert; verwenden Sie dies nur
während des Debuggings, da Prompts und Nachrichtentext weiterhin enthalten sein können.

Für das Stream-Debugging verwenden Sie `OPENCLAW_DEBUG_SSE=peek`, um die ersten fünf
redigierten SSE-Ereignisse zu protokollieren. Der Code-Modus schlägt außerdem sicher fehl, wenn die endgültige Provider-
Payload nicht genau ein `exec`, ein `wait` und ausschließlich genehmigte
Nur-Direkt-Tools enthält, nachdem die Code-Modus-Oberfläche aktiviert wurde.

## Implementierungsstruktur

- Konfigurationsvertrag: `tools.codeMode`
- Katalog-Builder: effektive Tools in kompakte Einträge und ID-Zuordnung überführen
- Modelloberflächen-Adapter: sichtbare Tools durch Steuerungs-/Direkt-Tools ersetzen
- QuickJS-WASI-Laufzeitadapter: laden, auswerten, Snapshot erstellen, wiederherstellen, freigeben
- Worker-Supervisor: Zeitüberschreitung, Abbruch, Absturzisolierung
- Bridge-Adapter: JSON-sichere Host-Callbacks und Ergebnisübermittlung
- TypeScript-Transformationsadapter
- Snapshot-Speicher: TTL, Größenbegrenzungen, Lauf-/Sitzungsbereich
- Trajektionsprojektion für verschachtelte Tool-Aufrufe
- Telemetriezähler und Diagnosefunktionen

Die Implementierung verwendet Katalog- und Executor-Konzepte aus Tool Search wieder,
verwendet jedoch kein untergeordnetes `node:vm` als Sandbox.

## Validierungscheckliste

Die Abdeckung des Code-Modus sollte Folgendes nachweisen:

- Eine deaktivierte Konfiguration lässt die bestehende Tool-Bereitstellung unverändert
- Eine Objektkonfiguration ohne `enabled: true` lässt den Code-Modus deaktiviert
- Eine aktivierte Konfiguration stellt dem Modell `exec`, `wait` und ausschließlich erforderliche Nur-Direkt-Tools bereit,
  wenn Tools für den Lauf aktiv sind
- Unverarbeitete Läufe ohne Tools, `disableTools` und leere Zulassungslisten lösen keine
  Erzwingung der Code-Modus-Payload aus
- Alle für den Katalog geeigneten effektiven Nicht-MCP-Tools erscheinen in `ALL_TOOLS`
- Nur-Direkt-Tools bleiben für das Modell sichtbar und erscheinen nicht in `ALL_TOOLS`
- Abgelehnte Tools erscheinen nicht in `ALL_TOOLS`
- `tools.search`, `tools.describe`, `tools.callValue` und `tools.call` funktionieren für OpenClaw-Tools
- `API.list("mcp")` und `API.read("mcp/<server>.d.ts")` stellen MCP-Deklarationen im TypeScript-Stil
  ohne Bridge-/Tool-Aufruf bereit
- Der MCP-Namespace `$api()` bleibt als Inline-Fallback für Schemas verfügbar
- MCP-Namespace-Aufrufe funktionieren für sichtbare MCP-Tools mit einer Objekteingabe, während
  direkte MCP-Katalogeinträge in `tools.*` fehlen
- Steuerungs-Tools von Tool Search sind sowohl auf der Modelloberfläche als auch im
  verborgenen Katalog ausgeblendet
- Verschachtelte Aufrufe bewahren das Genehmigungs- und Hook-Verhalten
- Shell-`exec` ist für das Modell ausgeblendet, kann jedoch bei entsprechender
  Zulassung über die Katalog-ID aufgerufen werden
- Rekursive Code-Modus-`exec` und `wait` können nicht aus Gastcode aufgerufen werden
- TypeScript-Eingaben werden transformiert und ausgewertet, ohne TypeScript in
  deaktivierten oder reinen JavaScript-Pfaden zu laden
- Zugriffe auf `import`, `require`, Dateisystem, Netzwerk und Umgebung schlagen fehl
- Endlosschleifen führen zu einer Zeitüberschreitung und können den Gateway nicht blockieren
- Fehler aufgrund der Speicherbegrenzung beenden die Gast-VM
- Ausgabe- und Snapshot-Begrenzungen werden für abgeschlossene und ausgesetzte Aufrufe durchgesetzt
- `wait` setzt einen ausgesetzten Snapshot fort und gibt den endgültigen Wert zurück
- Abgelaufene, abgebrochene, einer falschen Sitzung zugeordnete und unbekannte `runId`-Werte schlagen fehl
- Transkriptwiedergabe und -persistenz bewahren Steuerungsaufrufe des Code-Modus
- Transkript und Telemetrie stellen verschachtelte Tool-Aufrufe eindeutig dar

## E2E-Testplan

Führen Sie diese beim Ändern der Laufzeit als Integrations- oder End-to-End-Tests aus:

1. Starten Sie einen Gateway mit `tools.codeMode.enabled: false`.
2. Senden Sie einen Agentendurchlauf mit einer kleinen Menge direkter Tools.
3. Stellen Sie sicher, dass die für das Modell sichtbaren Tools unverändert sind.
4. Starten Sie mit `tools.codeMode.enabled: true` neu.
5. Senden Sie einen Agentendurchlauf mit OpenClaw-, Plugin-, MCP- und Client-Test-Tools.
6. Stellen Sie sicher, dass die für das Modell sichtbare Tool-Liste aus `exec`, `wait` sowie ausschließlich den konfigurierten
   Nur-Direkt-Tools besteht.
7. Lesen Sie in `exec` den Wert `ALL_TOOLS` und stellen Sie sicher, dass die für den Katalog geeigneten effektiven Test-
   Tools vorhanden sind, während Nur-Direkt-Tools fehlen.
8. Rufen Sie in `exec` OpenClaw-/Plugin-/Client-Tools über `tools.search`,
   `tools.describe` und `tools.callValue` (oder unverarbeitetes `tools.call`) auf.
9. Rufen Sie in `exec` die Werte `API.list("mcp")` und `API.read("mcp/<server>.d.ts")` auf und
   stellen Sie sicher, dass die Deklarationsdateien sichtbare MCP-Tools beschreiben.
10. Rufen Sie in `exec` MCP-Tools über `MCP.<server>.<tool>({ ...input })` auf und
    stellen Sie sicher, dass direkte MCP-Katalogeinträge in `ALL_TOOLS` und
    `tools.*` fehlen.
11. Stellen Sie sicher, dass abgelehnte Tools fehlen und nicht über eine erratene ID aufgerufen werden können.
12. Starten Sie einen verschachtelten Tool-Aufruf, der aufgelöst wird, nachdem `exec` den Wert `waiting` zurückgibt.
13. Rufen Sie `wait` auf und stellen Sie sicher, dass die wiederhergestellte VM das Tool-Ergebnis empfängt.
14. Stellen Sie sicher, dass die endgültige Antwort eine nach der Wiederherstellung erzeugte Ausgabe enthält.
15. Stellen Sie sicher, dass Zeitüberschreitung, Abbruch und Snapshot-Ablauf den Laufzeitstatus bereinigen.
16. Exportieren Sie die Trajektorie und stellen Sie sicher, dass verschachtelte Aufrufe unter dem übergeordneten
    Code-Modus-Aufruf sichtbar sind.

Bei reinen Dokumentationsänderungen an dieser Seite sollte weiterhin `pnpm check:docs` ausgeführt werden.

## Verwandte Themen

- [Swarm](/de/tools/swarm) für die Agentenorchestrierung mit Auffächerung aus Code-Modus-Skripten
- [Tool Search](/de/tools/tool-search)
- [Agentenlaufzeiten](/de/concepts/agent-runtimes)
- [Exec-Tool](/de/tools/exec)
- [Codeausführung](/de/tools/code-execution)
