---
read_when:
    - Sie möchten QMD als Ihr Speicher-Backend einrichten
    - Sie möchten erweiterte Speicherfunktionen wie Reranking oder zusätzliche indizierte Pfade.
summary: Local-First-Such-Sidecar mit BM25, Vektoren, Reranking und Anfrageerweiterung
title: QMD-Speicher-Engine
x-i18n:
    generated_at: "2026-07-26T18:20:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c0e54dc9a18d834036e4c79d6b7bdecb268a29976d9f30ea6e82a56ca5d71fda
    source_path: concepts/memory-qmd.md
    workflow: 16
---

[QMD](https://github.com/tobi/qmd) ist ein lokal ausgerichteter Such-Sidecar, der
parallel zu OpenClaw ausgeführt wird. Er kombiniert BM25, Vektorsuche und Reranking in einer einzigen
Binärdatei und kann Inhalte über die Workspace-Speicherdateien hinaus indizieren.

## Vorteile gegenüber der integrierten Lösung

- **Reranking und Abfrageerweiterung** für einen besseren Recall.
- **Zusätzliche Verzeichnisse indizieren** – Projektdokumentation, Teamnotizen und beliebige Inhalte auf dem Datenträger.
- **Sitzungstranskripte indizieren** – frühere Unterhaltungen wieder abrufen.
- **Vollständig lokal** – wird mit dem offiziellen llama.cpp-Provider-Plugin ausgeführt und
  lädt GGUF-Modelle automatisch herunter.
- **Automatischer Fallback** – wenn QMD nicht verfügbar ist, wechselt OpenClaw nahtlos zur
  integrierten Engine.

## Erste Schritte

### Voraussetzungen

- QMD installieren: `npm install -g @tobilu/qmd` oder `bun install -g @tobilu/qmd`
- SQLite-Build, der Erweiterungen zulässt (`brew install sqlite` unter macOS).
- QMD muss sich im `PATH` des Gateways befinden.
- macOS und Linux funktionieren ohne weitere Konfiguration. Windows wird am besten über WSL2 unterstützt.

### Aktivieren

```json5
{
  memory: {
    backend: "qmd",
  },
}
```

OpenClaw erstellt ein eigenständiges QMD-Home-Verzeichnis unter
`~/.openclaw/agents/<agentId>/qmd/` und verwaltet den Lebenszyklus des Sidecars
automatisch – Collections, Aktualisierungen und Embedding-Läufe werden für Sie verwaltet.
Es bevorzugt aktuelle QMD-Collection- und MCP-Abfragestrukturen, greift bei Bedarf jedoch auf
alternative Flags für Collection-Muster und ältere MCP-Werkzeugnamen zurück.
Die Startabstimmung erstellt außerdem veraltete verwaltete Collections mit ihren
kanonischen Mustern neu, wenn noch eine ältere QMD-Collection mit demselben Namen
vorhanden ist.

## Funktionsweise des Sidecars

- OpenClaw erstellt Collections aus Workspace-Speicherdateien und konfigurierten
  `memory.qmd.paths`. Der QMD-Adapter verwaltet Aktualisierungs-, Embedding-, Debounce- und
  Timeout-Heuristiken; diese sind nicht benutzerkonfigurierbar.
- QMD verwaltet weiterhin seine `index.sqlite`, die YAML-Collection-Konfiguration und Modell-
  Downloads im agentenspezifischen QMD-Home-Verzeichnis; dies sind Artefakte des externen Werkzeugs
  und keine OpenClaw-Zustandstabellen. Die OpenClaw-eigene Koordination befindet sich ausschließlich in SQLite:
  Eine gemeinsame Lease begrenzt die Embedding-Arbeit über mehrere Agenten hinweg, während eine Lease in jeder
  Agentendatenbank die Collection-, Aktualisierungs- und Embedding-Schreibvorgänge dieses Agenten serialisiert.
  Die Runtime erstellt keine QMD-Dateisperr-Sidecars mehr. `openclaw doctor --fix`
  entfernt ausgemusterte Sidecars erst, nachdem nachgewiesen wurde, dass ihr alter Prozesseigentümer nicht mehr aktiv ist.
  Upgrades erfolgen als sauberer Umstieg: Beenden und starten Sie alle OpenClaw-Prozesse neu, die
  das Zustandsverzeichnis gemeinsam verwenden, bevor Sie die neue Version einsetzen. Gemischte alte und neue QMD-
  Schreibprozesse werden nicht unterstützt; die Runtime verwendet absichtlich keine doppelte Sperrung mit den ausgemusterten
  Sidecars.
- Die standardmäßige Workspace-Collection verfolgt `MEMORY.md` sowie den `memory/`-
  Baum. `memory.md` in Kleinschreibung wird nicht als Stamm-Speicherdatei indiziert.
- Der QMD-eigene Scanner ignoriert ausgeblendete Pfade und gängige Abhängigkeits-/Build-
  Verzeichnisse wie `.git`, `.cache`, `node_modules`, `vendor`, `dist` und
  `build`. Beim Gateway-Start bleibt QMD verzögert initialisiert; der Manager wird initialisiert, wenn der Speicher
  erstmals verwendet wird.
- Suchvorgänge verwenden den konfigurierten `searchMode` (Standard: `search`; unterstützt außerdem
  `vsearch` und `query`). `search` verwendet ausschließlich BM25, daher überspringt OpenClaw in diesem Modus semantische
  Vektorbereitschaftsprüfungen und die Embedding-Wartung. Wenn ein Modus
  fehlschlägt, versucht OpenClaw es erneut mit `qmd query`.
- Wenn `searchMode` auf `query` gesetzt ist, setzen Sie `memory.qmd.rerank` auf `false`, um
  den hybriden Abfragepfad von QMD ohne den Reranker zu verwenden (erfordert QMD 2.1 oder neuer).
  OpenClaw übergibt `--no-rerank` an den direkten QMD-CLI-Pfad und
  `rerank: false` an das MCP-Abfragewerkzeug von QMD.
- Bei QMD-Versionen, die Filter für mehrere Collections ausweisen, gruppiert OpenClaw
  Collections derselben Quelle in einem einzigen QMD-Suchaufruf. Ältere QMD-Versionen
  behalten den kompatiblen Fallback pro Collection bei.
- Wenn QMD vollständig fehlschlägt, wechselt OpenClaw zur integrierten SQLite-Engine.
  Wiederholte Versuche in Chat-Runden werden nach einem Fehler beim Öffnen kurz verzögert, damit eine
  fehlende Binärdatei oder defekte Sidecar-Abhängigkeit keinen Wiederholungssturm verursacht;
  `openclaw memory status` und einmalige CLI-Prüfungen überprüfen QMD weiterhin
  direkt.

<Info>
Die erste Suche kann langsam sein – QMD lädt beim ersten
`qmd query`-Lauf automatisch GGUF-Modelle (~2 GB) für Reranking und Abfrageerweiterung herunter.
</Info>

## Suchleistung und Kompatibilität

OpenClaw hält den QMD-Suchpfad sowohl mit aktuellen als auch mit älteren QMD-
Installationen kompatibel.

Beim Start prüft OpenClaw einmal pro Manager den Hilfetext des installierten QMD. Wenn
die Binärdatei Unterstützung für mehrere Collection-Filter ausweist, durchsucht OpenClaw
alle Collections derselben Quelle mit einem Befehl:

```bash
qmd search "router notes" --json -n 10 -c memory-root-main -c memory-dir-main
```

Dadurch wird vermieden, für jede Collection des dauerhaften Speichers einen eigenen QMD-Unterprozess zu starten.
Collections mit Sitzungstranskripten verbleiben in ihrer eigenen Quellgruppe, sodass gemischte
Suchen mit `memory` + `sessions` dem Ergebnis-Diversifizierer weiterhin Eingaben aus
beiden Quellen liefern.

Ältere QMD-Builds akzeptieren nur einen Collection-Filter. Wenn OpenClaw einen
solchen Build erkennt, behält es den Kompatibilitätspfad bei und durchsucht jede Collection
separat, bevor die Ergebnisse zusammengeführt und dedupliziert werden.

Führen Sie Folgendes aus, um den installierten Vertrag manuell zu überprüfen:

```bash
qmd --help | grep -i collection
```

Die aktuelle QMD-Hilfe erwähnt die Ausrichtung auf eine oder mehrere Collections. Ältere Hilfetexte
beschreiben üblicherweise eine einzelne Collection.

## Modellüberschreibungen

QMD-Modellumgebungsvariablen werden unverändert vom Gateway-
Prozess übernommen, sodass Sie QMD global abstimmen können, ohne eine neue OpenClaw-Konfiguration hinzuzufügen:

```bash
export QMD_EMBED_MODEL="hf:Qwen/Qwen3-Embedding-0.6B-GGUF/Qwen3-Embedding-0.6B-Q8_0.gguf"
export QMD_RERANK_MODEL="/absolute/path/to/reranker.gguf"
export QMD_GENERATE_MODEL="/absolute/path/to/generator.gguf"
```

Führen Sie die Embeddings nach einer Änderung des Embedding-Modells erneut aus, damit der Index dem
neuen Vektorraum entspricht.

## Zusätzliche Pfade indizieren

Richten Sie QMD auf zusätzliche Verzeichnisse aus, damit diese durchsuchbar werden:

```json5
{
  memory: {
    backend: "qmd",
    qmd: {
      paths: [{ name: "docs", path: "~/notes", pattern: "**/*.md" }],
    },
  },
}
```

Ausschnitte aus zusätzlichen Pfaden erscheinen in den
Suchergebnissen als `qmd/<collection>/<relative-path>`. `memory_get` erkennt dieses Präfix und liest aus dem
richtigen Collection-Stammverzeichnis.

## Sitzungstranskripte indizieren

Aktivieren Sie die Sitzungsindizierung, um frühere Unterhaltungen wieder abzurufen. QMD benötigt sowohl die
allgemeine Sitzungsquelle `memory.search` als auch den QMD-Transkriptexporter:

```json5
{
  memory: {
    backend: "qmd",
    search: {
      experimental: { sessionMemory: true },
      sources: ["memory", "sessions"],
    },
    qmd: {
      sessions: { enabled: true },
    },
  },
}
```

Transkripte werden als bereinigte Benutzer-/Assistenten-Runden in eine dedizierte QMD-
Collection unter `~/.openclaw/agents/<id>/qmd/sessions/` exportiert. Wenn nur
`sources: ["sessions"]` festgelegt wird, werden keine Transkripte nach QMD exportiert; aktivieren Sie zusätzlich
`rememberAcrossConversations` oder den expliziten QMD-Sitzungsexport.

Sitzungstreffer werden weiterhin nach
[`tools.sessions.visibility`](/de/gateway/config-tools#toolssessions) gefiltert. Die
standardmäßige Sichtbarkeit `tree` umfasst die aktuelle Sitzung, ihre erzeugten Sitzungen
und Gruppensitzungen desselben Agenten, die über die umgebungsbezogene Gruppenwahrnehmung beobachtet werden. Mit
`session.dmScope: "main"` teilen Benutzer in einer Mehrbenutzer-DM-Konfiguration die Hauptsitzung
und können Inhalte aus deren beobachteten Gruppen wieder abrufen. Verwenden Sie für die DM-Isolation eine
peerbezogene `dmScope`, oder setzen Sie die Sichtbarkeit auf `"self"`, um umgebungsbezogene
Lesezugriffe auf beobachtete Sitzungen zu deaktivieren. Andere, nicht zusammenhängende Sitzungen desselben Agenten erfordern weiterhin
die Sichtbarkeit `"agent"`.

## Suchbereich

Standardmäßig werden QMD-Suchergebnisse nur in direkten Sitzungen angezeigt (nicht
in Gruppen- oder Kanalchats). Konfigurieren Sie `memory.qmd.scope`, um dies zu ändern:

```json5
{
  memory: {
    qmd: {
      scope: {
        default: "deny",
        rules: [{ action: "allow", match: { chatType: "direct" } }],
      },
    },
  },
}
```

Der obige Ausschnitt entspricht der tatsächlichen Standardregel. Wenn der Bereich eine Suche verweigert,
protokolliert OpenClaw eine Warnung mit dem abgeleiteten Kanal und Chattyp, damit sich leere
Ergebnisse leichter diagnostizieren lassen.

## Quellenangaben

Wenn `memory.citations` auf `auto` oder `on` gesetzt ist, wird an Suchausschnitte eine
`Source: <path>#L<line>`- (oder `#L<start>-L<end>`-) Fußzeile angehängt. Im Modus `auto`
wird die Fußzeile nur für Direktchat-Sitzungen hinzugefügt. Setzen Sie
`memory.citations = "off"`, um die Fußzeile wegzulassen, während der Pfad intern weiterhin an den
Agenten übergeben wird.

## Einsatzbereiche

Wählen Sie QMD, wenn Sie Folgendes benötigen:

- Reranking für hochwertigere Ergebnisse.
- Durchsuchen von Projektdokumentation oder Notizen außerhalb des Workspace.
- Wiederabrufen früherer Sitzungsunterhaltungen.
- Vollständig lokale Suche ohne API-Schlüssel.

Für einfachere Konfigurationen eignet sich die [integrierte Engine](/de/concepts/memory-builtin)
ohne zusätzliche Abhängigkeiten.

## Fehlerbehebung

**QMD nicht gefunden?** Stellen Sie sicher, dass sich die Binärdatei im `PATH` des Gateways befindet. Wenn OpenClaw
als Dienst ausgeführt wird, erstellen Sie einen symbolischen Link:
`sudo ln -s ~/.bun/bin/qmd /usr/local/bin/qmd`.

Wenn `qmd --version` in Ihrer Shell funktioniert, OpenClaw jedoch weiterhin
`spawn qmd ENOENT` meldet, verwendet der Gateway-Prozess wahrscheinlich einen anderen `PATH` als
Ihre interaktive Shell. Geben Sie die Binärdatei explizit an:

```json5
{
  memory: {
    backend: "qmd",
    qmd: {
      command: "/absolute/path/to/qmd",
    },
  },
}
```

Verwenden Sie `command -v qmd` in der Umgebung, in der QMD installiert ist, und überprüfen Sie anschließend erneut
mit `openclaw memory status --deep`.

**Erste Suche sehr langsam?** QMD lädt GGUF-Modelle bei der ersten Verwendung herunter. Wärmen Sie sie
mit `qmd query "test"` unter Verwendung derselben XDG-Verzeichnisse vor, die OpenClaw verwendet.

**Viele QMD-Unterprozesse während der Suche?** Aktualisieren Sie QMD nach Möglichkeit. OpenClaw
verwendet für Suchen in mehreren Collections derselben Quelle nur dann einen Prozess, wenn das
installierte QMD Unterstützung für mehrere `-c`-Filter ausweist; andernfalls
behält es aus Gründen der Korrektheit den älteren Fallback pro Collection bei.

**Versucht QMD trotz reinem BM25-Modus, llama.cpp zu erstellen?** Setzen Sie
`memory.qmd.searchMode = "search"`. OpenClaw behandelt diesen Modus als
rein lexikalisch, überspringt QMD-Vektorstatusprüfungen und die Embedding-Wartung und
überlässt semantische Bereitschaftsprüfungen den Konfigurationen `vsearch` oder `query`.

**Zeitüberschreitung bei der Suche?** Erhöhen Sie `memory.qmd.limits.timeoutMs` (Standard: 4000ms).
Setzen Sie den Wert bei langsamerer Hardware höher, beispielsweise auf `120000`. Dieses Limit gilt für
die QMD-eigenen Suchbefehle während der `memory_search`-Aufrufe des Agenten; Einrichtung, Synchronisierung,
integrierter Fallback und Arbeit am ergänzenden Korpus behalten ihre eigenen kürzeren Fristen bei.

**Leere Ergebnisse in Gruppen- oder Kanalchats?** Dies ist beim
standardmäßigen `memory.qmd.scope` zu erwarten, das nur direkte Sitzungen zulässt. Fügen Sie eine
`allow`-Regel für die Chattypen `group` oder `channel` hinzu, wenn dort QMD-Ergebnisse
angezeigt werden sollen.

**Ist die Suche im Stamm-Speicher plötzlich zu umfassend?** Starten Sie das Gateway neu oder warten Sie
auf die nächste Startabstimmung. OpenClaw erstellt veraltete verwaltete
Collections mit den kanonischen Mustern `MEMORY.md` und `memory/` neu, wenn es
einen Namenskonflikt erkennt.

**Verursachen im Workspace sichtbare temporäre Repositorys `ENAMETOOLONG` oder eine fehlerhafte Indizierung?**
Die QMD-Traversierung folgt dem zugrunde liegenden QMD-Scanner und nicht den
integrierten Symlink-Regeln von OpenClaw. Bewahren Sie temporäre Monorepository-Checkouts in ausgeblendeten
Verzeichnissen wie `.tmp/` oder außerhalb indizierter QMD-Stammverzeichnisse auf, bis QMD eine
zyklussichere Traversierung oder explizite Ausschlussoptionen bereitstellt.

## Konfiguration

Die vollständige Konfigurationsoberfläche (`memory.qmd.*`), Suchmodi, Aktualisierungsintervalle,
Bereichsregeln und alle weiteren Optionen finden Sie in der
[Referenz zur Speicherkonfiguration](/de/reference/memory-config).

## Verwandte Themen

- [Speicherübersicht](/de/concepts/memory)
- [Integrierte Speicher-Engine](/de/concepts/memory-builtin)
- [Honcho-Speicher](/de/concepts/memory-honcho)
