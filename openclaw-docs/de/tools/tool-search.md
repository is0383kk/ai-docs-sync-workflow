---
read_when:
    - Sie möchten, dass OpenClaw-Agenten einen großen Tool-Katalog verwenden, ohne jedes Tool-Schema zum Prompt hinzuzufügen
    - Sie möchten OpenClaw-Tools, MCP-Tools und Client-Tools über eine einzige kompakte Laufzeitschnittstelle bereitstellen.
    - Sie implementieren oder debuggen die Tool-Erkennung für OpenClaw-Ausführungen
summary: 'Tool-Suche: Große OpenClaw-Toolkataloge kompakt über Suche, Beschreibung und Aufruf zugänglich machen'
title: Werkzeugsuche
x-i18n:
    generated_at: "2026-07-26T18:41:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d31322d5ef108c52fd14d48771cc3c6c43fcfbc4bfb95652bc29a55fd706c903
    source_path: tools/tool-search.md
    workflow: 16
---

Tool Search ist eine experimentelle Funktion der OpenClaw-Agentenlaufzeit. Sie bietet Agenten eine
kompakte Möglichkeit, große Tool-Kataloge zu durchsuchen und daraus Tools aufzurufen. Dies ist nützlich, wenn für die Ausführung
viele Tools verfügbar sind, das Modell aber voraussichtlich nur wenige davon benötigt.

Diese Seite dokumentiert OpenClaw Tool Search. Sie behandelt nicht die Codex-native
Tool-Suche oder die Oberfläche für dynamische Tools. Der Codex-native Code-Modus, die Tool-Suche, zurückgestellte
dynamische Tools und verschachtelte Tool-Aufrufe sind stabile Oberflächen des Codex-Harness und
hängen nicht von `tools.toolSearch` ab.

Informationen zur generischen OpenClaw-Laufzeit, die statt Tool-Search-Steuerelementen eine QuickJS-WASI-Oberfläche für `exec`/`wait`
bereitstellt, finden Sie unter [Code-Modus](/de/tools/code-mode).

Wenn die Funktion für OpenClaw-Ausführungen aktiviert ist, erhält das Modell standardmäßig ein Tool vom Typ `tool_search_code`
sowie alle ausschließlich direkt verfügbaren Tools, deren strukturierte Ergebnisse nicht über
die kompakte Brücke übertragen werden können. Das Code-Tool führt einen kurzen JavaScript-Block in einem isolierten
Node-Unterprozess mit einer `openclaw.tools`-Brücke aus:

```js
const hits = await openclaw.tools.search("GitHub-Issue erstellen");
const tool = await openclaw.tools.describe(hits[0].id);
return await openclaw.tools.call(tool.id, {
  title: "Absturz beim Start",
  body: "Schritte zum Reproduzieren...",
});
```

Der Katalog kann katalogfähige OpenClaw-Tools, Plugin-Tools, MCP-
Tools und vom Client bereitgestellte Tools enthalten. Das Modell sieht nicht jedes katalogisierte Schema
im Voraus. Stattdessen durchsucht es kompakte Deskriptoren, ruft bei Bedarf das exakte Schema
eines ausgewählten Tools ab und ruft dieses Tool über OpenClaw auf.
Ausschließlich direkt verfügbare Tools bleiben für das Modell sichtbar und werden dem Katalog nicht hinzugefügt.

Ausführungen im Codex-Harness erhalten diese experimentellen Steuerelemente von OpenClaw Tool Search
nicht. OpenClaw übergibt Produktfunktionen als dynamische Tools an Codex, und
Codex verwaltet den stabilen nativen Code-Modus, die native Tool-Suche, zurückgestellte dynamische
Tools und verschachtelte Tool-Aufrufe.

## Ablauf eines Durchgangs

Während der Planung erstellt der eingebettete OpenClaw-Runner den effektiven Katalog für die
Ausführung:

1. Aktive Tool-Richtlinie für Agent, Profil, Sandbox und Sitzung auflösen.
2. Geeignete OpenClaw- und Plugin-Tools auflisten.
3. Geeignete MCP-Tools über die MCP-Laufzeit der Sitzung auflisten.
4. Geeignete, für die aktuelle Ausführung bereitgestellte Client-Tools hinzufügen.
5. Ausschließlich direkt verfügbare Tools für das Modell sichtbar halten und kompakte Deskriptoren für die
   übrigen katalogfähigen Tools indizieren.
6. Die OpenClaw-Code-Brücke, die strukturierten Fallback-Tools oder die
   kompakte Verzeichnisoberfläche neben diesen ausschließlich direkt verfügbaren Tools bereitstellen.

Zur Ausführungszeit kehrt jeder tatsächliche Tool-Aufruf zu OpenClaw zurück. Die isolierte Node-
Laufzeit enthält weder Plugin-Implementierungen noch MCP-Clientobjekte oder Secrets.
`openclaw.tools.call(...)` wird über die Brücke zurück an das Gateway übertragen, wo weiterhin
die reguläre Richtlinien-, Genehmigungs-, Hook-, Protokollierungs- und Ergebnisverarbeitung gilt.

## Modi

`tools.toolSearch` verfügt über drei für das Modell sichtbare Modi:

- `code`: stellt `tool_search_code`, die standardmäßige kompakte JavaScript-Brücke,
  zusammen mit ausschließlich direkt verfügbaren Tools bereit.
- `tools`: stellt `tool_search`, `tool_describe` und `tool_call` als einfache
  strukturierte Tools für Provider bereit, die keinen Code erhalten sollen, zusammen mit
  ausschließlich direkt verfügbaren Tools.
- `directory`: stellt `tool_search`, `tool_describe` und `tool_call` sowie ein
  begrenztes Prompt-Verzeichnis verfügbarer Tool-Namen und -Beschreibungen für
  Provider bereit, die Tool-Namen ohne jedes vollständige Schema sehen sollen. OpenClaw kann
  außerdem einen kleinen begrenzten Satz wahrscheinlicher oder erforderlicher Tool-Schemas direkt
  für den aktuellen Durchgang bereitstellen. Ausschließlich direkt verfügbare Tools bleiben auch in diesem Modus sichtbar.

Alle Modi verwenden denselben richtliniengefilterten Katalog und den regulären OpenClaw-Ausführungspfad.
Mit `catalogMode: "direct-only"` gekennzeichnete Tools bleiben außerhalb dieses Katalogs und
für das Modell sichtbar. Wenn die aktuelle Laufzeit den isolierten untergeordneten Node-Prozess für den Code-Modus
nicht starten kann, weicht der standardmäßige Modus `code` vor der Katalogkomprimierung auf `tools` aus.
Im Modus `directory` bleiben vom Client bereitgestellte Tools für die aktuelle Ausführung
direkt sichtbar, während OpenClaw-Tools, Plugin-Tools und MCP-Tools hinter dem
Verzeichniskatalog komprimiert werden können. Ein direkter Aufruf eines exakten verborgenen
Verzeichnisnamens wird vor der Ausführung aus demselben autorisierten Katalog geladen.

Alle Modi sind experimentell. Bevorzugen Sie die direkte Bereitstellung von Tools für kleine OpenClaw-Tool-
Kataloge und für Ausführungen im Codex-Harness die stabilen Codex-nativen Oberflächen.

Es gibt keine separate Konfiguration zur Quellenauswahl. Wenn Tool Search aktiviert ist, enthält der
Katalog nach der regulären Richtlinienfilterung katalogfähige OpenClaw-, MCP- und Client-Tools;
ausschließlich direkt verfügbare Tools werden separat beibehalten.

## Zweck

Große Kataloge sind nützlich, aber aufwendig. Wenn jedes Tool-Schema an das Modell gesendet wird,
wird die Anfrage größer, die Planung langsamer und die Wahrscheinlichkeit einer versehentlichen Tool-
Auswahl höher.

Tool Search ändert die Struktur:

- Direkte Tools: Das Modell sieht jedes ausgewählte Schema vor dem ersten Token.
- Code-Modus von Tool Search: Das Modell sieht ein kompaktes Code-Tool, einen kurzen API-
  Vertrag und alle ausschließlich direkt verfügbaren Tools.
- Tool-Modus von Tool Search: Das Modell sieht drei kompakte strukturierte Fallback-
  Tools sowie alle ausschließlich direkt verfügbaren Tools.
- Verzeichnismodus von Tool Search: Das Modell sieht ein begrenztes Verzeichnis sowie
  Steuerelemente zum Suchen, Beschreiben und Aufrufen, einen kleinen begrenzten Satz wahrscheinlicher oder erforderlicher
  Schemas und alle ausschließlich direkt verfügbaren Tools.
- Während des Durchgangs: Das Modell kann die übrigen Schemas nach Bedarf laden.

Die direkte Bereitstellung von Tools ist weiterhin die richtige Standardeinstellung für kleine Kataloge. Tool Search
eignet sich am besten, wenn eine Ausführung auf viele Tools zugreifen kann, insbesondere von MCP-Servern oder
vom Client bereitgestellten App-Tools.

## API

`openclaw.tools.search(query, options?)`

Durchsucht den effektiven Katalog der aktuellen Ausführung. Die Ergebnisse sind kompakt und können sicher
wieder in den Prompt-Kontext eingefügt werden. Jeder Treffer enthält eine begrenzte TypeScript-artige
`input`-Signatur, beispielsweise `{ id: string; mode?: "drip" | "flood" }`, sodass das
Modell `describe` überspringen kann, wenn diese Signatur ausreicht. Ein vertrauenswürdiges
OpenClaw-Core- oder Plugin-Tool kann außerdem einen kompakten `output`-Hinweis enthalten, beispielsweise
`Array<{ id: string; paid: boolean }>`. Angaben zu Ausgabeschemas von MCP und Clients
werden nicht zu diesem vertrauenswürdigen Hinweis hochgestuft. Auch deren nicht vertrauenswürdige Eingabeschemas werden
als `input: "unknown"` zurückgestellt; verwenden Sie vor dem Aufruf `describe`. Bei offenen,
übergroßen oder anderweitig unvollständigen Ausgabeschemas wird der Hinweis ausgelassen; sie bleiben stattdessen
über `describe` verfügbar.

```js
const hits = await openclaw.tools.search("Kalenderereignis", { limit: 5 });
```

`openclaw.tools.describe(id)`

Lädt die vollständigen Metadaten für ein Suchergebnis, einschließlich des exakten Eingabeschemas und
des vollständigen vertrauenswürdigen `outputSchema`, sofern das Tool eines deklariert.

```js
const calendarCreate = await openclaw.tools.describe("mcp:calendar:create_event");
```

`openclaw.tools.call(id, args)`

Ruft ein ausgewähltes Tool über OpenClaw auf und gibt den unverarbeiteten `{ tool, result }`-
Umschlag zurück. Tools mit JSON-Rückgabe legen ihren Wert normalerweise in
`result.details` ab. Wenn ein vertrauenswürdiges Tool `outputSchema` deklariert, kompiliert OpenClaw
das Schema vor der Ausführung und validiert nach den regulären Tool-
Hooks das endgültige `details`, bevor der Katalogaufruf zurückgegeben wird.

```js
await openclaw.tools.call(calendarCreate.id, {
  summary: "Planung",
  start: "2026-05-09T14:00:00Z",
});
```

Tool-Autoren deklarieren Ausgabeverträge über die Eigenschaft `outputSchema` des Tools.
Sie beschreibt `AgentToolResult.details`, nicht gerenderte Inhaltsblöcke. Schließen Sie
alle Varianten ein, die keine Ausnahme auslösen, oder lassen Sie die Eigenschaft bei instabilen Ergebnissen weg. Weitere Informationen finden Sie unter
[Ausgabeverträge des Code-Modus](/de/tools/code-mode#declared-output-contracts) und
[Tool-Plugins](/de/plugins/tool-plugins#output-contracts).

Der strukturierte Fallback-Modus stellt dieselben Operationen als Tools bereit:

- `tool_search`
- `tool_describe`
- `tool_call`

Der Verzeichnismodus stellt Folgendes bereit:

- `tool_search`
- `tool_describe`
- `tool_call`

Er hält außerdem vom Client bereitgestellte Tools und alle ausschließlich direkt verfügbaren Tools direkt sichtbar
und kann einen kleinen begrenzten Satz wahrscheinlicher oder erforderlicher Schemas von Katalog-Tools
direkt für den aktuellen Durchgang bereitstellen. Wenn das begrenzte Verzeichnis Einträge auslässt, verwenden Sie
`tool_search`, um sie zu finden. Wenn das Modell direkt den exakten Namen eines verborgenen Verzeichnis-
Tools anfordert, lädt OpenClaw es vor der regulären Ausführung aus dem autorisierten Katalog.
Die Namen von Client-Tools im Verzeichnismodus dürfen nicht mit Namen von OpenClaw-, Plugin- oder MCP-
Tools kollidieren, da der exakte zurückgestellte Versand diese Namen verwendet.

## Laufzeitgrenze

Die Code-Brücke wird in einem kurzlebigen Node-Unterprozess ausgeführt. Der Unterprozess startet
mit aktiviertem Node-Berechtigungsmodus, einer leeren Umgebung, ohne Dateisystem- oder
Netzwerkberechtigungen und ohne Berechtigungen für untergeordnete Prozesse oder Worker. OpenClaw erzwingt ein
Zeitlimit für die verstrichene Zeit im übergeordneten Prozess und beendet den Unterprozess bei Überschreitung, auch
nach asynchronen Fortsetzungen.

Die Laufzeit stellt nur Folgendes bereit:

- `console.log`, `console.warn` und `console.error`
- `openclaw.tools.search`
- `openclaw.tools.describe`
- `openclaw.tools.call`

Für endgültige Aufrufe gilt weiterhin das reguläre OpenClaw-Verhalten:

- Richtlinien zum Zulassen und Verweigern von Tools
- Tool-Einschränkungen pro Agent und pro Sandbox
- Tool-Richtlinie für Kanal und Laufzeit
- Genehmigungs-Hooks
- Plugin-Hooks vom Typ `before_tool_call`
- Sitzungsidentität, Protokolle und Telemetrie

## Konfiguration

Aktivieren Sie Tool Search für OpenClaw-Ausführungen mit der standardmäßigen Code-Brücke:

```bash
openclaw config set tools.toolSearch true
```

Entsprechendes JSON:

```json5
{
  tools: {
    toolSearch: true,
  },
}
```

Verwenden Sie stattdessen die strukturierten Fallback-Tools für OpenClaw-Ausführungen:

```json5
{
  tools: {
    toolSearch: {
      mode: "tools",
    },
  },
}
```

Verwenden Sie stattdessen die kompakte Verzeichnisoberfläche für OpenClaw-Ausführungen:

```json5
{
  tools: {
    toolSearch: {
      mode: "directory",
    },
  },
}
```

Passen Sie das Zeitlimit des Code-Modus und die Grenzwerte für Suchergebnisse an (die gezeigten Werte sind die Standardwerte):

```json5
{
  tools: {
    toolSearch: {
      mode: "code",
      codeTimeoutMs: 10000,
      searchDefaultLimit: 8,
      maxSearchLimit: 20,
    },
  },
}
```

Die Laufzeit begrenzt `codeTimeoutMs` auf 1000-60000, `maxSearchLimit` auf 1-50 und
`searchDefaultLimit` auf 1..`maxSearchLimit`.

Deaktivierung:

```json5
{
  tools: {
    toolSearch: false,
  },
}
```

## Prompt und Telemetrie

Tool Search zeichnet ausreichend Telemetriedaten auf, um einen Vergleich mit der direkten Tool-Bereitstellung zu ermöglichen:

- Gesamtzahl der serialisierten Tool- und Prompt-Bytes, die an das Harness gesendet wurden
- Kataloggröße und Aufschlüsselung nach Quellen
- Anzahl der Such-, Beschreibungs- und Aufrufvorgänge
- Über OpenClaw ausgeführte endgültige Tool-Aufrufe
- Ausgewählte Tool-IDs und Quellen

Die Sitzungsprotokolle sollten die Beantwortung folgender Fragen ermöglichen:

- Wie viele Tool-Schemas das Modell im Voraus gesehen hat
- Wie viele Such- und Beschreibungsvorgänge es ausgeführt hat
- Welches endgültige Tool aufgerufen wurde
- Ob das Ergebnis von OpenClaw, MCP oder einem Client-Tool stammte

## E2E-Validierung

Das Gateway-Szenario von QA Lab weist beide Pfade mit der OpenClaw-Laufzeit nach:

```bash
pnpm openclaw qa suite --provider-mode mock-openai --scenario tool-search-gateway-e2e
```

Es erstellt ein temporäres simuliertes Plugin mit einem großen Tool-Katalog, startet den simulierten
OpenAI-Provider, startet ein Gateway einmal im direkten Modus und einmal mit aktivierter Tool Search
und vergleicht anschließend die Anfrage-Payloads des Providers und die Sitzungsprotokolle.

Die Regression weist Folgendes nach:

1. Der Direktmodus kann das simulierte Plugin-Tool aufrufen.
2. Tool Search kann dasselbe simulierte Plugin-Tool aufrufen.
3. Der Direktmodus stellt dem Provider die Schemas des simulierten Plugin-Tools direkt bereit.
4. Tool Search stellt nur die kompakte Bridge sowie alle ausschließlich direkt verfügbaren Tools bereit.
5. Die Tool-Search-Anfragenutzlast ist für den großen simulierten Katalog kleiner.
6. Die Sitzungsprotokolle zeigen die erwartete Anzahl von Tool-Aufrufen und die Telemetrie der über die Bridge vermittelten Aufrufe.

## Fehlerverhalten

Tool Search sollte nach dem Fail-Closed-Prinzip vorgehen:

- Wenn ein Tool nicht von der wirksamen Richtlinie erfasst ist, sollte die Suche es nicht zurückgeben.
- Wenn ein ausgewähltes Tool nicht mehr verfügbar ist, sollte `tool_call` fehlschlagen.
- Wenn die Richtlinie oder Genehmigung die Ausführung blockiert, sollte das Aufrufergebnis diese
  Blockierung melden, statt sie zu umgehen.
- Wenn die Code-Bridge keine isolierte Laufzeit erstellen kann, verwenden Sie `mode: "tools"` oder
  deaktivieren Sie Tool Search für diese Bereitstellung.

## Verwandte Themen

- [Tools und Plugins](/de/tools)
- [Multi-Agent-Sandbox und Tools](/de/tools/multi-agent-sandbox-tools)
- [Exec-Tool](/de/tools/exec)
- [Einrichtung von ACP-Agenten](/de/tools/acp-agents-setup)
- [Plugins entwickeln](/de/plugins/building-plugins)
