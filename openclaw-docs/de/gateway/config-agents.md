---
read_when:
    - Agent-Standardeinstellungen anpassen (Modelle, Denkprozess, Arbeitsbereich, Heartbeat, Medien, Skills)
    - Konfigurieren von Multi-Agent-Routing und Bindungen
    - Anpassen des Verhaltens von Sitzungen, Nachrichtenzustellung und Sprechmodus
summary: Agent-Standardeinstellungen, Multi-Agent-Routing sowie Konfiguration von Sitzungen, Nachrichten und Sprachausgabe
title: Konfiguration — Agenten
x-i18n:
    generated_at: "2026-07-26T18:26:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7a161d65b02e3333c15a2d998421419ee37d36be4d02ebb3a86e66282df06adb
    source_path: gateway/config-agents.md
    workflow: 16
---

Agentenbezogene Konfigurationsschlüssel unter `agents.*`, `multiAgent.*`, `session.*`,
`messages.*` und `talk.*`. Informationen zu Kanälen, Tools, der Gateway-Laufzeit und anderen
Schlüsseln auf oberster Ebene finden Sie in der [Konfigurationsreferenz](/de/gateway/configuration-reference).

## Agentenstandards

### `agents.defaults.workspace`

Standard: `OPENCLAW_WORKSPACE_DIR`, wenn festgelegt, andernfalls `~/.openclaw/workspace` (oder `~/.openclaw/workspace-<profile>`, wenn `OPENCLAW_PROFILE` auf ein vom Standard abweichendes Profil festgelegt ist).

```json5
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
}
```

Ein expliziter Wert für `agents.defaults.workspace` hat Vorrang vor
`OPENCLAW_WORKSPACE_DIR`. Verwenden Sie die Umgebungsvariable, um Standardagenten
auf einen eingebundenen Arbeitsbereich zu verweisen, wenn Sie diesen Pfad nicht in die Konfiguration schreiben möchten.

### `agents.defaults.repoRoot`

Optionaler Repository-Stamm, der in der Runtime-Zeile des System-Prompts angezeigt wird. Wenn nicht festgelegt, erkennt OpenClaw ihn automatisch, indem es vom Arbeitsbereich aus nach oben navigiert.

```json5
{
  agents: { defaults: { repoRoot: "~/Projects/openclaw" } },
}
```

### `agents.defaults.skills`

Optionale standardmäßige Skill-Zulassungsliste für Agenten, die
`agents.entries.*.skills` nicht festlegen.

```json5
{
  agents: {
    defaults: { skills: ["github", "weather"] },
    list: [
      { id: "writer" }, // übernimmt github, weather
      { id: "docs", skills: ["docs-search"] }, // ersetzt die Standardwerte
      { id: "locked-down", skills: [] }, // keine Skills
    ],
  },
}
```

- Lassen Sie `agents.defaults.skills` weg, um Skills standardmäßig nicht einzuschränken.
- Lassen Sie `agents.entries.*.skills` weg, um die Standardwerte zu übernehmen.
- Legen Sie `agents.entries.*.skills: []` fest, um keine Skills zuzulassen.
- Eine nicht leere Liste `agents.entries.*.skills` ist die endgültige Auswahl für diesen Agenten; sie
  wird nicht mit den Standardwerten zusammengeführt.

### `agents.defaults.skipBootstrap`

Deaktiviert die automatische Erstellung von Bootstrap-Dateien im Arbeitsbereich (`AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`, `BOOTSTRAP.md`).

```json5
{
  agents: { defaults: { skipBootstrap: true } },
}
```

### `agents.defaults.skipOptionalBootstrapFiles`

Überspringt die Erstellung ausgewählter optionaler Arbeitsbereichsdateien, während erforderliche Bootstrap-Dateien (`AGENTS.md`, `TOOLS.md`, `BOOTSTRAP.md`) weiterhin geschrieben werden. Gültige Werte: `SOUL.md`, `USER.md` und `IDENTITY.md` (`HEARTBEAT.md` wird akzeptiert, hat jedoch keine Wirkung, da der Heartbeat-Kontext in den temporären Bereich des Cron-Monitors verschoben wurde).

```json5
{
  agents: {
    defaults: {
      skipOptionalBootstrapFiles: ["SOUL.md", "USER.md"],
    },
  },
}
```

### `agents.defaults.contextInjection`

Steuert, wann Bootstrap-Dateien des Arbeitsbereichs in den System-Prompt eingefügt werden. Standard: `"always"`.

- `"continuation-skip"`: Bei sicheren Fortsetzungsdurchläufen (nach einer abgeschlossenen Assistentenantwort) wird das erneute Einfügen des Arbeitsbereich-Bootstraps übersprungen, wodurch sich die Prompt-Größe verringert. Heartbeat-Ausführungen und Wiederholungsversuche nach der Compaction bauen den Kontext weiterhin neu auf.
- `"never"`: Deaktiviert bei jedem Durchlauf das Einfügen des Arbeitsbereich-Bootstraps und der Kontextdateien. Verwenden Sie dies nur für Agenten, die ihren Prompt-Lebenszyklus vollständig selbst verwalten (benutzerdefinierte Kontext-Engines, native Laufzeiten, die ihren eigenen Kontext erstellen, oder spezialisierte Arbeitsabläufe ohne Bootstrap). Auch bei Heartbeat- und Compaction-Wiederherstellungsdurchläufen wird das Einfügen übersprungen.

```json5
{
  agents: { defaults: { contextInjection: "continuation-skip" } },
}
```

Agentenspezifische Überschreibung: `agents.entries.*.contextInjection`. Nicht angegebene Werte übernehmen
`agents.defaults.contextInjection`.

### `agents.defaults.bootstrapMaxChars`

Maximale Zeichenanzahl pro Bootstrap-Datei des Arbeitsbereichs vor der Kürzung. Standard: `20000`.

```json5
{
  agents: { defaults: { bootstrapMaxChars: 20000 } },
}
```

Agentenspezifische Überschreibung: `agents.entries.*.bootstrapMaxChars`. Nicht angegebene Werte übernehmen
`agents.defaults.bootstrapMaxChars`.

### `agents.defaults.bootstrapTotalMaxChars`

Maximale Gesamtzahl der Zeichen, die aus allen Bootstrap-Dateien des Arbeitsbereichs eingefügt werden. Standard: `60000`.

```json5
{
  agents: { defaults: { bootstrapTotalMaxChars: 60000 } },
}
```

Agentenspezifische Überschreibung: `agents.entries.*.bootstrapTotalMaxChars`. Nicht angegebene Werte
übernehmen `agents.defaults.bootstrapTotalMaxChars`.

### Agentenspezifische Überschreibungen des Bootstrap-Profils

Verwenden Sie agentenspezifische Überschreibungen des Bootstrap-Profils, wenn ein Agent ein anderes Verhalten für die Prompt-Einfügung
als die gemeinsamen Standardwerte benötigt. Nicht angegebene Felder übernehmen die Werte aus
`agents.defaults`.

```json5
{
  agents: {
    defaults: {
      contextInjection: "continuation-skip",
      bootstrapMaxChars: 20000,
      bootstrapTotalMaxChars: 60000,
    },
    list: [
      {
        id: "strict-worker",
        contextInjection: "always",
        bootstrapMaxChars: 50000,
        bootstrapTotalMaxChars: 300000,
      },
    ],
  },
}
```

### `agents.defaults.bootstrapPromptTruncationWarning`

Steuert den für Agenten sichtbaren Hinweis im System-Prompt, wenn der Bootstrap-Kontext gekürzt wird.
Standard: `"always"`.

- `"off"`: Fügt niemals einen Hinweistext zur Kürzung in den System-Prompt ein.
- `"once"`: Fügt für jede eindeutige Kürzungssignatur einmal einen knappen Hinweis ein.
- `"always"`: Fügt bei jeder Ausführung einen knappen Hinweis ein, wenn eine Kürzung vorliegt (empfohlen).

Detaillierte Roh-/Einfügungszählwerte und Felder zur Konfigurationsoptimierung verbleiben in Diagnosen wie
Kontext-/Statusberichten und Protokollen; der routinemäßige WebChat-Benutzer-/Laufzeitkontext erhält nur
den knappen Wiederherstellungshinweis.

```json5
{
  agents: { defaults: { bootstrapPromptTruncationWarning: "always" } }, // off | once | always
}
```

### Zuordnung der Kontextbudget-Zuständigkeiten

OpenClaw verfügt über mehrere umfangreiche Prompt-/Kontextbudgets, die
bewusst nach Subsystem aufgeteilt sind, anstatt alle über einen einzigen generischen
Regler zu steuern.

| Budget                                                         | Umfasst                                                                                                                                                          |
| -------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `agents.defaults.bootstrapMaxChars` / `bootstrapTotalMaxChars` | Normale Einfügung des Arbeitsbereich-Bootstraps                                                                                                                            |
| `agents.defaults.startupContext.*`                             | Einmalige Präambel für Modellläufe beim Zurücksetzen/Starten, einschließlich aktueller täglicher `memory/*.md`-Dateien. Reine Chatbefehle `/new` und `/reset` werden bestätigt, ohne das Modell aufzurufen |
| `skills.limits.*`                                              | Die kompakte Skills-Liste, die in den System-Prompt eingefügt wird                                                                                                         |
| `agents.defaults.contextLimits.*`                              | Begrenzte Laufzeitauszüge und eingefügte, von der Laufzeit verwaltete Blöcke                                                                                                      |
| `memory.qmd.limits.*`                                          | Größe des indizierten Speichersuchauszugs und der Einfügung                                                                                                              |

Entsprechende agentenspezifische Überschreibungen:

- `agents.entries.*.skillsLimits.maxSkillsPromptChars`
- `agents.entries.*.contextInjection`
- `agents.entries.*.bootstrapMaxChars`
- `agents.entries.*.bootstrapTotalMaxChars`
- `agents.entries.*.contextLimits.*`

#### `agents.defaults.startupContext`

Steuert die beim ersten Durchlauf eingefügte Startpräambel für Modellläufe beim Zurücksetzen/Starten.
Reine Chatbefehle `/new` und `/reset` bestätigen das Zurücksetzen, ohne
das Modell aufzurufen, und laden diese Präambel daher nicht.

```json5
{
  agents: {
    defaults: {
      startupContext: {
        enabled: true,
        applyOn: ["new", "reset"],
        dailyMemoryDays: 2,
        maxFileBytes: 16384,
        maxFileChars: 1200,
        maxTotalChars: 2800,
      },
    },
  },
}
```

#### `agents.defaults.contextLimits`

Gemeinsame Standardwerte für begrenzte Laufzeitkontext-Oberflächen.

```json5
{
  agents: {
    defaults: {
      contextLimits: {
        memoryGetMaxChars: 12000,
        postCompactionMaxChars: 1800,
      },
    },
  },
}
```

- `memoryGetMaxChars`: Standardmäßige Begrenzung für `memory_get`-Auszüge, bevor Kürzungsmetadaten
  und ein Fortsetzungshinweis hinzugefügt werden.
- Wenn `memory_get` den Wert `lines` nicht angibt, verwendet OpenClaw ein integriertes Fenster von 120 Zeilen und
  wendet anschließend `memoryGetMaxChars` an.
- Live-Tool-Ergebnisse verwenden eine automatische Modellkontextbegrenzung: `16000` Zeichen unter 100K
  Token, `32000` Zeichen ab 100K Token und `64000` Zeichen ab 200K Token.
- `postCompactionMaxChars`: Begrenzung für AGENTS.md-Auszüge, die bei der Aktualisierungseinfügung
  nach der Compaction verwendet wird.

#### `agents.entries.*.contextLimits`

Agentenspezifische Überschreibung für die gemeinsamen `contextLimits`-Regler. Nicht angegebene Felder übernehmen
die Werte aus `agents.defaults.contextLimits`.

```json5
{
  agents: {
    defaults: {
      contextLimits: { memoryGetMaxChars: 12000 },
    },
    list: [
      {
        id: "tiny-local",
        contextLimits: {
          memoryGetMaxChars: 6000,
        },
      },
    ],
  },
}
```

#### `skills.limits.maxSkillsPromptChars`

Globale Begrenzung für die kompakte Skills-Liste, die in den System-Prompt eingefügt wird. Dies
wirkt sich nicht auf das bedarfsgesteuerte Lesen von `SKILL.md`-Dateien aus.

```json5
{
  skills: { limits: { maxSkillsPromptChars: 18000 } },
}
```

#### `agents.entries.*.skillsLimits.maxSkillsPromptChars`

Agentenspezifische Überschreibung für das Skills-Prompt-Budget.

```json5
{
  agents: {
    list: [{ id: "tiny-local", skillsLimits: { maxSkillsPromptChars: 6000 } }],
  },
}
```

### `agents.defaults.imageMaxDimensionPx`

Maximale Pixelgröße für die längste Bildseite in Transkript-/Tool-Bildblöcken vor Provider-Aufrufen.
Standard: `1200`.

Niedrigere Werte reduzieren bei screenshotintensiven Ausführungen normalerweise die Nutzung von Vision-Token und die Größe der Anfragenutzlast.
Höhere Werte bewahren mehr visuelle Details.

```json5
{
  agents: { defaults: { imageMaxDimensionPx: 1200 } },
}
```

### `agents.defaults.imageQuality`

Komprimierungs-/Detailpräferenz des Bild-Tools für Bilder, die aus Dateipfaden, URLs und Medienreferenzen geladen werden.
Standard: `auto`.

OpenClaw passt die Größenänderungsabstufung an das ausgewählte Bildmodell an. Beispielsweise können Claude Opus 4.8, OpenAI GPT-5.6 Sol, Qwen VL und gehostete Llama-4-Vision-Modelle größere Bilder als ältere bzw. standardmäßige hochdetaillierte Vision-Pfade verwenden, während Durchläufe mit mehreren Bildern im Modus `auto` aggressiver komprimiert werden, um Token- und Latenzkosten zu begrenzen.

Werte:

- `auto`: Anpassung an Modellgrenzen und Bildanzahl.
- `efficient`: Bevorzugt kleinere Bilder für eine geringere Token- und Byte-Nutzung.
- `balanced`: Verwendet die standardmäßige ausgewogene Abstufung.
- `high`: Bewahrt mehr Details bei Screenshots, Diagrammen und Dokumentbildern.

```json5
{
  agents: { defaults: { imageQuality: "auto" } },
}
```

### `agents.defaults.userTimezone`

Zeitzone für den Kontext des System-Prompts (nicht für Nachrichtenzeitstempel). Fällt auf die Zeitzone des Hosts zurück.

```json5
{
  agents: { defaults: { userTimezone: "America/Chicago" } },
}
```

### `agents.defaults.timeFormat`

Zeitformat im System-Prompt. Standard: `auto` (Betriebssystemeinstellung).

```json5
{
  agents: { defaults: { timeFormat: "auto" } }, // auto | 12 | 24
}
```

### `agents.defaults.model`

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": { alias: "opus" },
        "minimax/MiniMax-M2.7": { alias: "minimax" },
      },
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["minimax/MiniMax-M2.7"],
      },
      utilityModel: "openai/gpt-5.4-mini",
      imageModel: {
        primary: "openrouter/qwen/qwen-2.5-vl-72b-instruct:free",
        fallbacks: ["openrouter/google/gemini-2.0-flash-vision:free"],
      },
      mediaModels: {
        image: {
          primary: "openai/gpt-image-2",
          fallbacks: ["google/gemini-3.1-flash-image"],
        },
        video: {
          primary: "qwen/wan2.6-t2v",
          fallbacks: ["qwen/wan2.6-i2v"],
        },
      },
      pdfModel: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["openai/gpt-5.4-mini"],
      },
      params: { cacheRetention: "long" }, // globale standardmäßige Provider-Parameter
      pdfMaxMb: 10,
      pdfMaxPages: 20,
      thinkingDefault: "low",
      verboseDefault: "off",
      toolProgressDetail: "explain",
      reasoningDefault: "off",
      elevatedDefault: "on",
      timeoutSeconds: 600,
      mediaMaxMb: 5,
      contextTokens: 200000,
      maxConcurrent: 4,
    },
  },
}
```

- `model`: akzeptiert entweder eine Zeichenfolge (`"provider/model"`) oder ein Objekt (`{ primary, fallbacks }`).
  - Die Zeichenfolgenform legt nur das primäre Modell fest.
  - Die Objektform legt das primäre Modell sowie geordnete Failover-Modelle fest.
- `utilityModel`: optionale `provider/model`-Referenz oder Alias für kurze interne Aufgaben. Sie wird derzeit für generierte Sitzungstitel der Control UI, Titel von Telegram-DM-Themen, automatisch erstellte Discord-Thread-Titel und die [Erzählung von Fortschrittsentwürfen](/de/concepts/progress-drafts#narrated-status) verwendet. Wenn sie nicht festgelegt ist, leitet OpenClaw den vom primären Provider deklarierten Standard für kleine Modelle ab, sofern einer vorhanden ist (OpenAI → `gpt-5.6-luna`, Anthropic → `claude-haiku-4-5`); andernfalls verwenden Titelaufgaben das primäre Modell des Agenten, und die Erzählung bleibt deaktiviert. Wenn ein separates Hilfsmodell einen generierten Titel nicht vorbereiten oder fertigstellen kann, versucht OpenClaw diesen Titel einmal erneut mit dem primären Modell. Für Dashboard-Titel verwenden die automatische Ableitung des Hilfsmodells und der reguläre Fallback den effektiven Sitzungs-Provider und das effektive Authentifizierungsprofil; ein explizites Hilfsmodell behält seinen konfigurierten Provider und seine konfigurierte Authentifizierung bei. Legen Sie `utilityModel: ""` fest, um die alternative Hilfsroute zu überspringen; die Generierung von Dashboard-Titeln erfolgt weiterhin direkt über das reguläre Sitzungsmodell. `agents.entries.*.utilityModel` überschreibt den Standard, und eine operationsspezifische Modellüberschreibung hat Vorrang vor beiden. Hilfsaufgaben führen separate Modellaufrufe aus und senden aufgabenspezifische Inhalte an den ausgewählten Modell-Provider. Die Generierung von Dashboard-Titeln sendet höchstens die ersten 1.000 Zeichen der ersten Nachricht, die kein Befehl ist; die Erzählung sendet die eingehende Anfrage sowie kompakte, geschwärzte Werkzeugzusammenfassungen. Wählen Sie einen Provider, der Ihren Anforderungen an Kosten und Datenverarbeitung entspricht.
- `imageModel`: akzeptiert entweder eine Zeichenfolge (`"provider/model"`) oder ein Objekt (`{ primary, fallbacks }`).
  - Wird vom Werkzeugpfad `image` als dessen Konfiguration für das Bildverarbeitungsmodell verwendet, wenn das aktive Modell keine Bilder akzeptieren kann. Modelle mit nativer Bildverarbeitung erhalten stattdessen die geladenen Bildbytes direkt.
  - Wird außerdem als Fallback-Routing verwendet, wenn das ausgewählte bzw. standardmäßige Modell keine Bildeingaben akzeptieren kann.
  - Bevorzugen Sie explizite `provider/model`-Referenzen. Bloße IDs werden aus Kompatibilitätsgründen akzeptiert; wenn eine bloße ID eindeutig einem konfigurierten, bildfähigen Eintrag in `models.providers.*.models` entspricht, ergänzt OpenClaw sie um diesen Provider. Bei mehrdeutigen konfigurierten Übereinstimmungen ist ein explizites Provider-Präfix erforderlich.
- `mediaModels.image`: akzeptiert entweder eine Zeichenfolge (`"provider/model"`) oder ein Objekt (`{ primary, fallbacks }`).
  - Wird von der gemeinsamen Bildgenerierungsfunktion und allen zukünftigen Werkzeug-/Plugin-Oberflächen verwendet, die Bilder generieren.
  - Typische Werte: `google/gemini-3.1-flash-image` für die native Gemini-Bildgenerierung, `fal/fal-ai/flux/dev` für fal, `openai/gpt-image-2` für OpenAI Images oder `openai/gpt-image-1.5` für OpenAI-PNG-/WebP-Ausgaben mit transparentem Hintergrund.
  - Wenn Sie einen Provider bzw. ein Modell direkt auswählen, konfigurieren Sie auch die passende Provider-Authentifizierung (zum Beispiel `GEMINI_API_KEY` oder `GOOGLE_API_KEY` für `google/*`, `OPENAI_API_KEY` oder OpenAI Codex OAuth für `openai/gpt-image-2` / `openai/gpt-image-1.5`, `FAL_KEY` für `fal/*`).
  - Falls nicht angegeben, kann `image_generate` weiterhin einen durch Authentifizierung gestützten Provider-Standard ableiten. Dabei wird zuerst der aktuelle Standard-Provider und anschließend die übrigen registrierten Bildgenerierungs-Provider in der Reihenfolge ihrer Provider-IDs ausprobiert.
- `mediaModels.music`: akzeptiert entweder eine Zeichenfolge (`"provider/model"`) oder ein Objekt (`{ primary, fallbacks }`).
  - Wird von der gemeinsamen Musikgenerierungsfunktion und dem integrierten Werkzeug `music_generate` verwendet.
  - Typische Werte: `google/lyria-3-clip-preview`, `google/lyria-3-pro-preview` oder `minimax/music-2.6`.
  - Falls nicht angegeben, kann `music_generate` weiterhin einen durch Authentifizierung gestützten Provider-Standard ableiten. Dabei wird zuerst der aktuelle Standard-Provider und anschließend die übrigen registrierten Musikgenerierungs-Provider in der Reihenfolge ihrer Provider-IDs ausprobiert.
  - Wenn Sie einen Provider bzw. ein Modell direkt auswählen, konfigurieren Sie auch die passende Provider-Authentifizierung bzw. den passenden API-Schlüssel.
- `mediaModels.video`: akzeptiert entweder eine Zeichenfolge (`"provider/model"`) oder ein Objekt (`{ primary, fallbacks }`).
  - Wird von der gemeinsamen Videogenerierungsfunktion und dem integrierten Werkzeug `video_generate` verwendet.
  - Typische Werte: `qwen/wan2.6-t2v`, `qwen/wan2.6-i2v`, `qwen/wan2.6-r2v`, `qwen/wan2.6-r2v-flash` oder `qwen/wan2.7-r2v`.
  - Falls nicht angegeben, kann `video_generate` weiterhin einen durch Authentifizierung gestützten Provider-Standard ableiten. Dabei wird zuerst der aktuelle Standard-Provider und anschließend die übrigen registrierten Videogenerierungs-Provider in der Reihenfolge ihrer Provider-IDs ausprobiert.
  - Wenn Sie einen Provider bzw. ein Modell direkt auswählen, konfigurieren Sie auch die passende Provider-Authentifizierung bzw. den passenden API-Schlüssel.
  - Das offizielle Qwen-Plugin zur Videogenerierung unterstützt bis zu 1 Ausgabevideo, 1 Eingabebild, 4 Eingabevideos, eine Dauer von 10 Sekunden sowie die Optionen `size`, `aspectRatio`, `resolution`, `audio` und `watermark` auf Provider-Ebene.
- `pdfModel`: akzeptiert entweder eine Zeichenfolge (`"provider/model"`) oder ein Objekt (`{ primary, fallbacks }`).
  - Wird vom Werkzeug `pdf` für das Modell-Routing verwendet.
  - Falls nicht angegeben, greift das PDF-Werkzeug zunächst auf `imageModel` und anschließend auf das aufgelöste Sitzungs-/Standardmodell zurück.
- `pdfMaxMb`: standardmäßige PDF-Größenbeschränkung für das Werkzeug `pdf`, wenn `maxBytesMb` beim Aufruf nicht übergeben wird.
- `pdfMaxPages`: standardmäßige maximale Seitenzahl, die im Extraktions-Fallback-Modus des Werkzeugs `pdf` berücksichtigt wird.
- `verboseDefault`: standardmäßige Ausführlichkeitsstufe für Agenten. Werte: `"off"`, `"on"`, `"full"`. Standard: `"off"`.
- `toolProgressDetail`: Detailmodus für Werkzeugzusammenfassungen von `/verbose` und Werkzeugzeilen in Fortschrittsentwürfen. Werte: `"explain"` (Standard, kompakte menschenlesbare Bezeichnungen) oder `"raw"` (hängt den Rohbefehl bzw. Details an, sofern verfügbar). Das agentenspezifische `agents.entries.*.toolProgressDetail` überschreibt diesen Standard.
- `reasoningDefault`: standardmäßige Sichtbarkeit der Schlussfolgerungen für Agenten. Werte: `"off"`, `"on"`, `"stream"`. Das agentenspezifische `agents.entries.*.reasoningDefault` überschreibt diesen Standard. Konfigurierte Schlussfolgerungsstandards werden nur für Eigentümer, autorisierte Absender oder Gateway-Kontexte mit Operator-Administratorrechten angewendet, wenn keine nachrichten- oder sitzungsspezifische Schlussfolgerungsüberschreibung festgelegt ist.
- `elevatedDefault`: standardmäßige Stufe für Ausgaben mit erhöhten Rechten bei Agenten. Werte: `"off"`, `"on"`, `"ask"`, `"full"`. Standard: `"on"`.
- `model.primary`: Format `provider/model` (z. B. `openai/gpt-5.6-sol` für den Zugriff über Codex OAuth). Wenn Sie den Provider weglassen, versucht OpenClaw zuerst einen Alias, dann eine eindeutige Übereinstimmung eines konfigurierten Providers für genau diese Modell-ID und greift erst danach auf den konfigurierten Standard-Provider zurück (veraltetes Kompatibilitätsverhalten; bevorzugen Sie daher ein explizites `provider/model`). Wenn dieser Provider das konfigurierte Standardmodell nicht mehr bereitstellt, greift OpenClaw auf den ersten konfigurierten Provider bzw. das erste konfigurierte Modell zurück, statt einen veralteten Standard eines entfernten Providers auszugeben.
- `contextTokens`: optionale agentenweite Obergrenze. Sie kann das effektive Budget eines größeren Modells senken, ein Modell jedoch nicht über sein konfiguriertes oder ermitteltes `contextTokens` hinaus anheben. Um für ein direktes OpenAI-Modell dessen größeres natives Kontextfenster zu aktivieren, legen Sie für dieses Modell `models.providers.openai.models[].contextWindow` und `contextTokens` fest; siehe [Standards für OpenAI-Kontextfenster](/de/providers/openai#context-window-defaults-and-long-context-opt-in).
- `models`: konfigurierte Aliase und modellspezifische Einstellungen. Jeder Eintrag kann `alias` (Kurzbezeichnung) und `params` (Provider-spezifisch, zum Beispiel `temperature`, `maxTokens`, `cacheRetention`, `context1m`, `responsesServerCompaction`, `responsesCompactThreshold`, OpenRouter-`provider`-Routing, `chat_template_kwargs`, `extra_body`/`extraBody`) enthalten. Das Hinzufügen von Einträgen schränkt Modellüberschreibungen nicht ein.
  - Verwenden Sie `provider/*`-Einträge wie `"openai/*": {}` oder `"vllm/*": {}`, um alle ermittelten Modelle für ausgewählte Provider anzuzeigen, ohne jede Modell-ID manuell aufzuführen.
  - Fügen Sie einem `provider/*`-Eintrag `agentRuntime` hinzu, wenn jedes dynamisch ermittelte Modell dieses Providers dieselbe Runtime verwenden soll. Die exakte Runtime-Richtlinie `provider/model` hat weiterhin Vorrang vor dem Platzhalter.
  - Sichere Metadatenänderungen: Verwenden Sie `openclaw config set agents.defaults.models '<json>' --strict-json --merge`, um Einträge hinzuzufügen. `config set` verweigert Ersetzungen, durch die vorhandene Einträge entfernt würden, sofern Sie nicht `--replace` übergeben.
- `modelPolicy.allow`: explizite Positivliste für Überschreibungen. Akzeptiert Aliase, exakte `provider/model`-Referenzen und abschließende Präfix-Platzhalter wie `openai/*` oder `clawrouter/anthropic/*`. Lassen Sie die Angabe weg oder verwenden Sie `[]`, um jedes Modell zuzulassen. `agents.entries.*.modelPolicy.allow` ersetzt die Standardrichtlinie für diesen Agenten; eine explizite leere Liste aktiviert für diesen Agenten die Zulassung aller Modelle.
  - Provider-spezifische Konfigurations-/Onboarding-Abläufe führen die ausgewählten Provider-Modelle mit dieser Zuordnung zusammen und behalten bereits konfigurierte, nicht betroffene Provider bei.
  - Für direkte OpenAI-Responses-Modelle ist die serverseitige Compaction automatisch aktiviert. Verwenden Sie `params.responsesServerCompaction: false`, um das Einfügen von `context_management` zu unterbinden, oder `params.responsesCompactThreshold`, um den Schwellenwert zu überschreiben. Siehe [Serverseitige OpenAI-Compaction](/de/providers/openai#advanced-configuration).
- `params`: globale standardmäßige Provider-Parameter, die auf alle Modelle angewendet werden. Festzulegen unter `agents.defaults.params` (z. B. `{ cacheRetention: "long" }`).
- Zusammenführungspriorität von `params` (Konfiguration): `agents.defaults.params` (globale Basis) wird von `agents.defaults.models["provider/model"].params` (modellspezifisch) überschrieben; anschließend überschreibt `agents.entries.*.params` (übereinstimmende Agenten-ID) schlüsselweise. Einzelheiten finden Sie unter [Prompt-Caching](/de/reference/prompt-caching).
- `models.providers.openrouter.params.provider`: OpenRouter-weite Standardrichtlinie für das Provider-Routing. OpenClaw leitet diese an das Anfrageobjekt `provider` von OpenRouter weiter; modellspezifische `agents.defaults.models["openrouter/<model>"].params.provider` und Agentenparameter überschreiben schlüsselweise. Siehe [OpenRouter-Provider-Routing](/de/providers/openrouter#advanced-configuration).
- `params.extra_body`/`params.extraBody`: erweitertes, unverändert weitergereichtes JSON, das für OpenAI-kompatible Proxys mit den `api: "openai-completions"`-Anfragetexten zusammengeführt wird. Wenn es mit generierten Anfrageschlüsseln kollidiert, hat der zusätzliche Anfragetext Vorrang; nicht native Vervollständigungsrouten entfernen anschließend weiterhin das nur für OpenAI bestimmte `store`.
- `params.chat_template_kwargs`: Argumente für Chat-Vorlagen von vLLM/OpenAI-kompatiblen Systemen, die mit den Anfragetexten der obersten Ebene `api: "openai-completions"` zusammengeführt werden. Für `vllm/nemotron-3-*` mit deaktiviertem Denken sendet das gebündelte vLLM-Plugin automatisch `enable_thinking: false` und `force_nonempty_content: true`; explizite `chat_template_kwargs` überschreiben generierte Standardwerte, und `extra_body.chat_template_kwargs` hat weiterhin die endgültige Priorität. Konfigurierte vLLM-Qwen- und Nemotron-Denkmodelle bieten binäre `/think`-Auswahlmöglichkeiten (`off`, `on`) statt der mehrstufigen Aufwandsabstufung.
- `compat.thinkingFormat`: Stil der OpenAI-kompatiblen Denknutzlast. Verwenden Sie `"together"` für Together-artiges `reasoning.enabled`, `"qwen"` für Qwen-artiges `enable_thinking` auf oberster Ebene oder `"qwen-chat-template"` für `chat_template_kwargs.enable_thinking` auf Backends der Qwen-Familie, die Chat-Vorlagen-Schlüsselwortargumente auf Anfrageebene unterstützen, etwa vLLM. OpenClaw ordnet deaktiviertes Denken `false` und aktiviertes Denken `true` zu; konfigurierte vLLM-Qwen-Modelle bieten für diese Formate binäre `/think`-Auswahlmöglichkeiten.
- `compat.supportedReasoningEfforts`: Liste des Reasoning-Aufwands pro Modell für OpenAI-kompatible Schnittstellen. Fügen Sie `"xhigh"` für benutzerdefinierte Endpunkte hinzu, die dies tatsächlich akzeptieren; OpenClaw stellt dann `/think xhigh` in Befehlsmenüs, Gateway-Sitzungszeilen, der Validierung von Sitzungspatches, der Agent-CLI-Validierung und der `llm-task`-Validierung für diesen konfigurierten Provider und dieses Modell bereit. Verwenden Sie `compat.reasoningEffortMap`, wenn das Backend für eine kanonische Stufe einen providerspezifischen Wert erwartet.
- `params.preserveThinking`: Nur für Z.AI verfügbare Opt-in-Option für beibehaltenes Thinking. Wenn sie aktiviert und Thinking eingeschaltet ist, sendet OpenClaw `thinking.clear_thinking: false` und spielt vorherige `reasoning_content` erneut ein; siehe [Z.AI-Thinking und beibehaltenes Thinking](/de/providers/zai#advanced-configuration).
- `localService`: optionaler Prozessmanager auf Providerebene für lokale bzw. selbst gehostete Modellserver. Wenn das ausgewählte Modell zu diesem Provider gehört, prüft OpenClaw `healthUrl` (oder `baseUrl + "/models"`), startet `command` mit `args`, falls der Endpunkt nicht erreichbar ist, wartet bis zu `readyTimeoutMs` und sendet anschließend die Modellanfrage. `command` muss ein absoluter Pfad sein. `idleStopMs: 0` hält den Prozess aktiv, bis OpenClaw beendet wird; ein positiver Wert beendet den von OpenClaw gestarteten Prozess nach der entsprechenden Anzahl inaktiver Millisekunden. Siehe [Lokale Modelldienste](/de/gateway/local-model-services).
- Laufzeitrichtlinien gehören zu Providern oder Modellen, nicht zu `agents.defaults`. Verwenden Sie `models.providers.<provider>.agentRuntime` für providerweite Regeln oder `agents.defaults.models["provider/model"].agentRuntime` / `agents.entries.*.models["provider/model"].agentRuntime` für modellspezifische Regeln. Ein Provider-/Modellpräfix allein wählt niemals ein Harness aus. Wenn die Laufzeit nicht festgelegt oder auf `auto` gesetzt ist, darf OpenAI Codex nur bei einer exakten offiziellen HTTPS-Route für Platform Responses oder ChatGPT Responses ohne explizite Anfrageüberschreibung implizit auswählen. Siehe [Implizite OpenAI-Agent-Laufzeit](/de/providers/openai#implicit-agent-runtime).
- Konfigurationsschreiber, die diese Felder ändern (beispielsweise `/models set`, `/models set-image` sowie Befehle zum Hinzufügen oder Entfernen von Fallbacks), speichern die kanonische Objektform und behalten vorhandene Fallback-Listen nach Möglichkeit bei.
- `maxConcurrent`: maximale Anzahl paralleler Agent-Ausführungen über Sitzungen hinweg (jede Sitzung wird weiterhin serialisiert). Standard: `4`.

### Runtime-Richtlinie

```json5
{
  models: {
    providers: {
      openai: {
        agentRuntime: { id: "codex" },
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.6-sol",
      models: {
        "anthropic/claude-opus-5": {
          agentRuntime: { id: "claude-cli" },
        },
        "vllm/*": {
          agentRuntime: { id: "openclaw" },
        },
      },
    },
  },
}
```

- `id`: `"auto"`, `"openclaw"`, die ID eines registrierten Plugin-Harnesses oder ein unterstützter CLI-Backend-Alias. Das mitgelieferte Codex-Plugin registriert `codex`; das mitgelieferte Anthropic-Plugin stellt das CLI-Backend `claude-cli` bereit.
- `id: "auto"` ermöglicht registrierten Plugin-Harnesses, effektive Routen zu übernehmen, die ihren Supportvertrag deklarieren oder anderweitig erfüllen, und verwendet OpenClaw, wenn kein Harness übereinstimmt. Eine explizite Plugin-Runtime wie `id: "codex"` erfordert dieses Harness und eine kompatible effektive Route; sie schlägt sicher geschlossen fehl, wenn eines davon nicht verfügbar ist oder die Ausführung fehlschlägt.
- `id: "pi"` wird nur als veralteter Alias für `openclaw` akzeptiert, um ausgelieferte Konfigurationen aus v2026.5.22 und früher zu erhalten. Neue Konfigurationen sollten `openclaw` verwenden.
- Bei der Runtime-Priorität gilt zuerst die Richtlinie für das exakte Modell (`agents.entries.*.models["provider/model"]`, `agents.defaults.models["provider/model"]` oder `models.providers.<provider>.models[]`), dann `agents.entries.*` / `agents.defaults.models["provider/*"]` und anschließend die Provider-weite Richtlinie unter `models.providers.<provider>.agentRuntime`.
- Runtime-Schlüssel für den gesamten Agenten sind veraltet. `agents.defaults.agentRuntime`, `agents.entries.*.agentRuntime`, Runtime-Festlegungen für Sitzungen und `OPENCLAW_AGENT_RUNTIME` werden bei der Runtime-Auswahl ignoriert. Führen Sie `openclaw doctor --fix` aus, um veraltete Werte zu entfernen.
- Geeignete exakte offizielle HTTPS-Routen für OpenAI Responses/ChatGPT ohne selbst definierte Anfrageüberschreibung können das Codex-Harness implizit verwenden. Provider/Modell `agentRuntime.id: "codex"` macht Codex zu einer sicher geschlossen fehlschlagenden Anforderung, macht eine inkompatible Route jedoch nicht kompatibel.
- Bevorzugen Sie für Claude-CLI-Bereitstellungen `model: "anthropic/claude-opus-5"` zusammen mit dem modellspezifischen `agentRuntime.id: "claude-cli"`. Veraltete `claude-cli/<model>`-Referenzen funktionieren aus Kompatibilitätsgründen weiterhin, neue Konfigurationen sollten die Provider-/Modellauswahl jedoch kanonisch halten und das Ausführungs-Backend in der Provider-/Modell-Runtime-Richtlinie festlegen.
- Dies steuert nur die Ausführung textbasierter Agenten-Turns. Mediengenerierung, Vision, PDF, Musik, Video und TTS verwenden weiterhin ihre Provider-/Modelleinstellungen.

**Integrierte Alias-Kurzformen** (gelten nur, wenn sich das Modell in `agents.defaults.models` befindet):

| Alias               | Modell                          |
| ------------------- | ------------------------------- |
| `opus`              | `anthropic/claude-opus-5`       |
| `sonnet`            | `anthropic/claude-sonnet-5`     |
| `gpt`               | `openai/gpt-5.4`                |
| `gpt-mini`          | `openai/gpt-5.4-mini`           |
| `gpt-nano`          | `openai/gpt-5.4-nano`           |
| `gemini`            | `google/gemini-3.1-pro-preview` |
| `gemini-flash`      | `google/gemini-3-flash-preview` |
| `gemini-flash-lite` | `google/gemini-3.1-flash-lite`  |

Ihre konfigurierten Aliasse haben immer Vorrang vor den Standardwerten.

Z.AI-GLM-4.x-Modelle aktivieren automatisch den Denkmodus, sofern Sie nicht `--thinking off` festlegen oder `agents.defaults.models["zai/<model>"].params.thinking` selbst definieren.
Z.AI-Modelle aktivieren standardmäßig `tool_stream` für das Streaming von Tool-Aufrufen. Setzen Sie `agents.defaults.models["zai/<model>"].params.tool_stream` auf `false`, um dies zu deaktivieren.
Bei Anthropic Claude Opus 4.8 bleibt das Denken in OpenClaw standardmäßig deaktiviert; wenn adaptives Denken ausdrücklich aktiviert wird, lautet der Provider-eigene Standardwert für den Aufwand von Anthropic `high`. Claude-4.6-Modelle verwenden standardmäßig `adaptive`, wenn keine explizite Denkstufe festgelegt ist.

### Auswahl des CLI-Backends

Die Mechanik der CLI-Adapter wird von Plugins registriert und nicht unter den Agenten-
Standardwerten konfiguriert. Wählen Sie ein registriertes CLI-Backend wie oben gezeigt
mit dem modellspezifischen `agentRuntime.id` aus. Informationen zum Betrieb finden Sie unter
[CLI-Backends](/de/gateway/cli-backends) und Informationen zur Registrierung von Befehlen,
Sitzungen, Bildern und Parsern unter [CLI-Backend-Plugins erstellen](/de/plugins/cli-backend-plugins).

### `agents.defaults.promptOverlays`

Provider-unabhängige Prompt-Overlays, die je Modellfamilie auf von OpenClaw zusammengestellte Prompt-Oberflächen angewendet werden. Modell-IDs der GPT-5-Familie erhalten den gemeinsamen Verhaltensvertrag über OpenClaw-/Provider-Routen hinweg; `personality` steuert nur die Ebene des freundlichen Interaktionsstils. Native Codex-App-Server-Routen behalten die Codex-eigenen Basis-/Modellanweisungen anstelle dieses OpenClaw-GPT-5-Overlays bei, und OpenClaw deaktiviert die integrierte Persönlichkeit von Codex für native Threads.

```json5
{
  agents: {
    defaults: {
      promptOverlays: {
        gpt5: {
          personality: "friendly", // freundlich | ein | aus
        },
      },
    },
  },
}
```

- `"friendly"` (Standard) und `"on"` aktivieren die Ebene des freundlichen Interaktionsstils.
- `"off"` deaktiviert nur die freundliche Ebene; der markierte GPT-5-Verhaltensvertrag bleibt aktiviert.
- Das veraltete `plugins.entries.openai.config.personality` wird weiterhin gelesen, wenn diese gemeinsame Einstellung nicht festgelegt ist.

### `agents.defaults.heartbeat`

Regelmäßige Heartbeat-Läufe.

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m", // 0m deaktiviert
        model: "openai/gpt-5.4-mini",
        includeReasoning: false,
        includeSystemPromptSection: true, // Standard: true; false lässt den Heartbeat-Abschnitt im System-Prompt weg
        lightContext: false, // Standard: false; true überspringt Workspace-Bootstrap-Dateien bei Heartbeat-Läufen
        isolatedSession: false, // Standard: false; true führt jeden Heartbeat in einer neuen Sitzung aus (kein Gesprächsverlauf)
        skipWhenBusy: false, // Standard: false; true wartet auch auf die Subagenten-/verschachtelten Lanes dieses Agenten
        session: "main",
        to: "+15555550123",
        directPolicy: "allow", // allow (Standard) | block
        target: "none", // Standard: none | Optionen: last | whatsapp | telegram | discord | ...
        prompt: "Dem Scratch-Kontext des Heartbeat-Monitors folgen ...",
        ackMaxChars: 300,
        suppressToolErrorWarnings: false,
        timeoutSeconds: 45,
      },
    },
  },
}
```

- `every`: Zeitdauerzeichenfolge (ms/s/m/h). Standard: `30m` (API-Schlüssel-Authentifizierung) oder `1h` (OAuth-Authentifizierung). Zum Deaktivieren auf `0m` setzen.
- Die Taktung wird in eine systemeigene Cron-Monitorzeile geschrieben. Führen Sie `openclaw doctor --fix` aus, um eine fehlende oder veraltete Zeile anzulegen. Wenn Cron deaktiviert ist, werden geplante Heartbeats nicht ausgeführt und das Gateway protokolliert beim Start eine Warnung.
- `includeSystemPromptSection`: Wenn der Wert false ist, wird der Heartbeat-Abschnitt im System-Prompt weggelassen. Standard: `true`.
- `suppressToolErrorWarnings`: Wenn der Wert true ist, werden Nutzdaten mit Tool-Fehlerwarnungen während Heartbeat-Läufen unterdrückt.
- `timeoutSeconds`: Maximal zulässige Dauer eines Heartbeat-Agenten-Turns in Sekunden, bevor er abgebrochen wird. Lassen Sie die Einstellung weg, um `agents.defaults.timeoutSeconds` zu verwenden, sofern festgelegt; andernfalls wird die Heartbeat-Taktung mit einer Obergrenze von 600 Sekunden verwendet.
- `directPolicy`: Richtlinie für direkte/DM-Zustellung. `allow` (Standard) erlaubt die Zustellung an direkte Ziele. `block` unterdrückt die Zustellung an direkte Ziele und gibt `reason=dm-blocked` aus.
- `lightContext`: Wenn der Wert true ist, verwenden Heartbeat-Läufe einen schlanken Bootstrap-Kontext und überspringen Workspace-Bootstrap-Dateien. Der Monitor-Scratch-Kontext wird in beiden Fällen vom Heartbeat-Runner eingefügt.
- `isolatedSession`: Wenn der Wert true ist, wird jeder Heartbeat in einer neuen Sitzung ohne vorherigen Gesprächsverlauf ausgeführt. Dasselbe Isolationsmuster wie bei Cron `sessionTarget: "isolated"`. Reduziert die Token-Kosten je Heartbeat von ~100K auf ~2-5K Token.
- `skipWhenBusy`: Wenn der Wert true ist, werden Heartbeat-Läufe aufgeschoben, solange die zusätzlichen ausgelasteten Lanes dieses Agenten aktiv sind: seine eigene, an Sitzungsschlüssel gebundene Subagenten- oder verschachtelte Befehlsarbeit. Cron-Lanes schieben Heartbeats auch ohne dieses Flag immer auf.
- Pro Agent: Legen Sie `agents.entries.*.heartbeat` fest. Sobald ein Agent `heartbeat` definiert, führen **nur diese Agenten** Heartbeats aus.
- Heartbeats führen vollständige Agenten-Turns aus — kürzere Intervalle verbrauchen mehr Token.

### `agents.defaults.compaction`

```json5
{
  agents: {
    defaults: {
      compaction: {
        mode: "safeguard", // default | safeguard
        provider: "my-provider", // ID eines registrierten Compaction-Provider-Plugins (optional)
        thinkingLevel: "low", // optionale, ausschließlich für Compaction geltende Überschreibung des Denkaufwands
        timeoutSeconds: 180,
        keepRecentTokens: 50000,
        recentTurnsPreserve: 3,
        identifierPolicy: "strict", // strict | off
        qualityGuard: { enabled: true, maxRetries: 1 },
        midTurnPrecheck: { enabled: false }, // optionale Prüfung auf Tool-Schleifendruck während des Turns
        postIndexSync: "async", // off | async | await
        postCompactionSections: ["Session Startup", "Red Lines"],
        model: "openrouter/anthropic/claude-sonnet-4-6", // optionale, ausschließlich für Compaction geltende Modellüberschreibung
        truncateAfterCompaction: true, // nach der Compaction in eine kleinere nachfolgende JSONL-Datei rotieren
        maxActiveTranscriptBytes: "20mb", // optionaler Vorab-Auslöser für lokale Compaction
        notifyUser: true, // Benachrichtigungen beim Start/Abschluss der Compaction und bei Beeinträchtigungen des Speicher-Flushs (Standard: false)
        memoryFlush: {
          enabled: true,
          model: "ollama/qwen3:8b", // optionale, ausschließlich für den Speicher-Flush geltende Modellüberschreibung
          softThresholdTokens: 6000,
          forceFlushTranscriptBytes: "2mb",
        },
      },
    },
  },
}
```

- `mode`: `default` oder `safeguard` (abschnittsweise Zusammenfassung für lange Verläufe). Siehe [Compaction](/de/concepts/compaction).
- `provider`: ID eines registrierten Compaction-Provider-Plugins. Wenn festgelegt, wird `summarize()` des Providers anstelle der integrierten LLM-Zusammenfassung aufgerufen. Bei einem Fehler wird auf die integrierte Funktion zurückgegriffen. Das Festlegen eines Providers erzwingt `mode: "safeguard"`. Siehe [Compaction](/de/concepts/compaction).
- `thinkingLevel`: optionale Denkstufe, die nur für eingebettete OpenClaw-Compaction-Zusammenfassungen verwendet wird (`off`, `minimal`, `low`, `medium`, `high`, `xhigh`, `adaptive`, `max` oder `ultra`). Sie überschreibt die aktuelle Denkstufe der Sitzung und wird auf den Bereich des ausgewählten Compaction-Modells bzw. der ausgewählten Runtime begrenzt. Lassen Sie die Einstellung unausgefüllt, um die Sitzungsstufe zu übernehmen. Die native Compaction des Codex-App-Servers ignoriert diese Einstellung, da die native Compact-Anfrage keine Denkstufenüberschreibung pro Vorgang unterstützt; OpenClaw protokolliert eine Warnung, wenn sie konfiguriert ist.
- `timeoutSeconds`: maximal zulässige Anzahl von Sekunden für einen einzelnen Compaction-Vorgang, bevor OpenClaw ihn abbricht. Standard: `180`.
- `keepRecentTokens`: Budget für den Schnittpunkt des Agenten, um das neueste Transkriptende wortgetreu beizubehalten. Manuelles `/compact` berücksichtigt dies, wenn es ausdrücklich festgelegt ist; andernfalls ist die manuelle Compaction ein fester Prüfpunkt.
- `recentTurnsPreserve`: Anzahl der neuesten Benutzer-/Assistentenwechsel, die außerhalb der Schutzmechanismus-Zusammenfassung wortgetreu beibehalten werden. Standard: `3`.
- `identifierPolicy`: `strict` (Standard) oder `off`. `strict` stellt bei der Compaction-Zusammenfassung integrierte Anweisungen zur Beibehaltung opaker Kennungen voran.
- `qualityGuard`: Prüfungen mit Wiederholungsversuch bei fehlerhaft formatierter Ausgabe für Schutzmechanismus-Zusammenfassungen. Im Schutzmechanismusmodus standardmäßig aktiviert; setzen Sie `enabled: false`, um die Prüfung zu überspringen.
- `midTurnPrecheck`: optionale Prüfung des Tool-Schleifendrucks. Wenn `enabled: true`, prüft OpenClaw den Kontextdruck, nachdem Tool-Ergebnisse angefügt wurden und bevor das nächste Modell aufgerufen wird. Wenn der Kontext nicht mehr passt, wird der aktuelle Versuch vor dem Senden des Prompts abgebrochen und der bestehende Wiederherstellungspfad der Vorabprüfung wiederverwendet, um Tool-Ergebnisse zu kürzen oder eine Compaction mit anschließendem Wiederholungsversuch durchzuführen. Funktioniert mit den Compaction-Modi `default` und `safeguard`. Standard: deaktiviert.
- `postIndexSync`: Modus zur Neuindizierung des Sitzungsspeichers nach der Compaction. Standard: `"async"`. Verwenden Sie `"await"` für größtmögliche Aktualität, `"async"` für eine geringere Compaction-Latenz oder `"off"` nur, wenn die Synchronisierung des Sitzungsspeichers anderweitig erfolgt.
- `postCompactionSections`: optionale Namen von H2-/H3-Abschnitten aus AGENTS.md, die nach der Compaction erneut eingefügt werden. Lassen Sie die Einstellung unausgefüllt oder verwenden Sie `[]`, um dies zu deaktivieren.
- `model`: optionales `provider/model-id` oder einfacher Alias aus `agents.defaults.models`, ausschließlich für die Compaction-Zusammenfassung. Einfache Aliasse werden vor der Weiterleitung aufgelöst; konfigurierte wörtliche Modell-IDs haben bei Kollisionen Vorrang. Verwenden Sie dies, wenn die Hauptsitzung ein Modell beibehalten soll, Compaction-Zusammenfassungen jedoch mit einem anderen ausgeführt werden sollen; wenn nicht festgelegt, verwendet die Compaction das primäre Modell der Sitzung.
- `truncateAfterCompaction`: rotiert das aktive Sitzungstranskript nach der Compaction, sodass zukünftige Wechsel nur die Zusammenfassung und das nicht zusammengefasste Ende laden, während das vorherige vollständige Transkript archiviert bleibt. Verhindert ein unbegrenztes Wachstum des aktiven Transkripts in lang laufenden Sitzungen. Standard: `false`.
- `maxActiveTranscriptBytes`: optionaler Schwellenwert in Byte (`number` oder Zeichenfolgen wie `"20mb"`), der vor einem Lauf die normale lokale Compaction auslöst, wenn der Transkriptverlauf den Schwellenwert überschreitet. Erfordert `truncateAfterCompaction`, damit eine erfolgreiche Compaction zu einem kleineren Nachfolgetranskript rotieren kann. Deaktiviert, wenn nicht festgelegt oder `0`.
- `notifyUser`: sendet bei `true` kurze Hinweise zur Kontextpflege an den Benutzer: wenn die Compaction beginnt und abgeschlossen ist (zum Beispiel „Kontext wird komprimiert ...“ und „Compaction abgeschlossen“) sowie wenn eine Speicherleerung vor der Compaction ausgeschöpft ist, sodass die Antwort in einem eingeschränkten Zustand fortgesetzt wird (zum Beispiel „Die Speicherpflege ist vorübergehend fehlgeschlagen; Ihre Antwort wird fortgesetzt.“). Standardmäßig deaktiviert, damit diese Hinweise nicht angezeigt werden.
- `memoryFlush`: stiller agentischer Wechsel vor der automatischen Compaction, um dauerhafte Erinnerungen zu speichern. Setzen Sie `model` auf einen genauen Provider/ein genaues Modell wie `ollama/qwen3:8b`, wenn dieser Wartungswechsel auf einem lokalen Modell verbleiben soll; die Überschreibung übernimmt nicht die aktive Fallback-Kette der Sitzung. `forceFlushTranscriptBytes` erzwingt die Leerung, wenn die Transkriptgröße den Schwellenwert erreicht, selbst wenn die Token-Zähler veraltet sind. Wird übersprungen, wenn der Workspace schreibgeschützt ist.

Benutzerdefinierte Compaction-Anweisungen gehören zum Code. Implementieren Sie ein Compaction-Provider-
Plugin mit `summarize()` für die benutzerdefinierte Erstellung von Zusammenfassungen und verwenden Sie
`before_prompt_build`, wenn der Kontext nach der Compaction in spätere
Modell-Prompts eingefügt werden muss. Doctor entfernt die eingestellten veralteten Anweisungsfelder und verweist auf diese
Schnittstellen.

### `agents.defaults.contextPruning`

Entfernt **alte Tool-Ergebnisse** aus dem speicherinternen Kontext, bevor dieser an das LLM gesendet wird. Ändert den Sitzungsverlauf auf dem Datenträger **nicht**. Standardmäßig deaktiviert; setzen Sie `mode: "cache-ttl"`, um die Funktion zu aktivieren.

```json5
{
  agents: {
    defaults: {
      contextPruning: {
        mode: "cache-ttl", // aus (Standard) | cache-ttl
      },
    },
  },
}
```

<Accordion title="Verhalten des Modus cache-ttl">

- `mode: "cache-ttl"` aktiviert Bereinigungsdurchläufe.
- Bei der Bereinigung werden übergroße Tool-Ergebnisse zunächst schonend gekürzt und ältere Tool-Ergebnisse anschließend bei Bedarf vollständig entfernt.

**Schonendes Kürzen** behält Anfang und Ende bei und fügt `...` in der Mitte ein.

**Vollständiges Entfernen** ersetzt das gesamte Tool-Ergebnis durch den Platzhalter.

Hinweise:

- Bildblöcke werden niemals gekürzt oder entfernt.
- Verhältnisse basieren auf Zeichen (Näherungswerte), nicht auf exakten Token-Anzahlen.
- Die neuesten Assistentennachrichten bleiben erhalten.

</Accordion>

Verhaltensdetails finden Sie unter [Sitzungsbereinigung](/de/concepts/session-pruning).

### Block-Streaming

```json5
{
  agents: {
    defaults: {
      blockStreamingDefault: "off", // on | off
      blockStreamingBreak: "text_end", // text_end | message_end
      blockStreamingChunk: { minChars: 800, maxChars: 1200, breakPreference: "paragraph" },
      blockStreamingCoalesce: { idleMs: 1000 },
      humanDelay: { mode: "natural" }, // off (Standard) | natural | custom (verwendet minMs/maxMs)
    },
  },
}
```

- Kanäle außer Telegram erfordern ein ausdrückliches `*.streaming.block.enabled: true`, um Blockantworten zu aktivieren. QQ Bot bildet die Ausnahme: Er besitzt keine `streaming.block`-Schlüssel und streamt Blockantworten, sofern `channels.qqbot.streaming.mode` nicht `"off"` ist.
- Kanalspezifische Überschreibungen: `channels.<channel>.streaming.block.coalesce` (sowie Varianten pro Konto). Discord, Google Chat, Mattermost, MS Teams, Signal und Slack verwenden standardmäßig `minChars: 1500` / `idleMs: 1000`.
- `blockStreamingChunk.breakPreference`: bevorzugte Abschnittsgrenze (`"paragraph" | "newline" | "sentence"`).
- `humanDelay`: zufällige Pause zwischen Blockantworten. Standard: `off`. `natural` = 800-2500ms. `custom` verwendet `minMs`/`maxMs` (für jede nicht festgelegte Grenze wird auf den natürlichen Bereich zurückgegriffen). Überschreibung pro Agent: `agents.entries.*.humanDelay`.

Details zu Verhalten und Abschnittsbildung finden Sie unter [Streaming](/de/concepts/streaming).

### Tippindikatoren

```json5
{
  agents: {
    defaults: {
      typingMode: "instant", // never | instant | thinking | message
      typingIntervalSeconds: 6,
    },
  },
}
```

- Standardwerte: `instant` für Direktchats/Erwähnungen, `message` für Gruppenchats ohne Erwähnung.
- `typingIntervalSeconds`-Standardwert: `6`.
- Überschreibung pro Agent: `agents.entries.*.typingMode`.

Siehe [Tippindikatoren](/de/concepts/typing-indicators).

<a id="agentsdefaultssandbox"></a>

### `agents.defaults.sandbox`

Optionales Sandboxing für den eingebetteten Agenten. Die vollständige Anleitung finden Sie unter [Sandboxing](/de/gateway/sandboxing).

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main", // off (Standard) | non-main | all
        backend: "docker", // docker (Standard) | ssh | openshell
        scope: "agent", // session | agent (Standard) | shared
        workspaceAccess: "none", // none (Standard) | ro | rw
        workspaceRoot: "~/.openclaw/sandboxes",
        docker: {
          image: "openclaw-sandbox:bookworm-slim",
          containerPrefix: "openclaw-sbx-",
          workdir: "/workspace",
          readOnlyRoot: true,
          tmpfs: ["/tmp", "/var/tmp", "/run"],
          network: "none",
          user: "1000:1000",
          capDrop: ["ALL"],
          env: { LANG: "C.UTF-8" },
          setupCommand: "apt-get update && apt-get install -y git curl jq",
          pidsLimit: 256,
          memory: "1g",
          memorySwap: "2g",
          cpus: 1,
          gpus: "all",
          ulimits: {
            nofile: { soft: 1024, hard: 2048 },
            nproc: 256,
          },
          seccompProfile: "/path/to/seccomp.json",
          apparmorProfile: "openclaw-sandbox",
          dns: ["1.1.1.1", "8.8.8.8"],
          extraHosts: ["internal.service:10.0.0.5"],
          binds: ["/home/user/source:/source:rw"],
        },
        ssh: {
          target: "user@gateway-host:22",
          command: "ssh",
          workspaceRoot: "/tmp/openclaw-sandboxes",
          strictHostKeyChecking: true,
          updateHostKeys: true,
          identityFile: "~/.ssh/id_ed25519",
          certificateFile: "~/.ssh/id_ed25519-cert.pub",
          knownHostsFile: "~/.ssh/known_hosts",
          // SecretRefs / Inline-Inhalte werden ebenfalls unterstützt:
          // identityData: { source: "env", provider: "default", id: "SSH_IDENTITY" },
          // certificateData: { source: "env", provider: "default", id: "SSH_CERTIFICATE" },
          // knownHostsData: { source: "env", provider: "default", id: "SSH_KNOWN_HOSTS" },
        },
        browser: {
          enabled: false,
          image: "openclaw-sandbox-browser:bookworm-slim",
          network: "openclaw-sandbox-browser",
          cdpPort: 9222,
          cdpSourceRange: "172.21.0.1/32",
          vncPort: 5900,
          noVncPort: 6080,
          headless: false,
          enableNoVnc: true,
          allowHostControl: false,
          autoStart: true,
          autoStartTimeoutMs: 12000,
        },
        prune: {
          idleHours: 24,
          maxAgeDays: 7,
        },
      },
    },
  },
  tools: {
    sandbox: {
      tools: {
        allow: [
          "exec",
          "process",
          "read",
          "write",
          "edit",
          "apply_patch",
          "sessions_list",
          "sessions_history",
          "sessions_send",
          "sessions_spawn",
          "session_status",
        ],
        deny: ["browser", "canvas", "nodes", "cron", "discord", "gateway"],
      },
    },
  },
}
```

Die oben gezeigten Standardwerte (`off`/`docker`/`agent`/`none`/`bookworm-slim`-Image/`none`-Netzwerk usw.) sind die tatsächlichen OpenClaw-Standardwerte und nicht nur Beispielwerte.

<Accordion title="Sandbox-Details">

**Backend:**

- `docker`: lokale Docker-Runtime (Standard)
- `ssh`: generische SSH-gestützte Remote-Runtime
- `openshell`: OpenShell-Runtime

Wenn `backend: "openshell"` ausgewählt ist, werden Runtime-spezifische Einstellungen nach
`plugins.entries.openshell.config` verschoben.

**SSH-Backend-Konfiguration:**

- `target`: SSH-Ziel im Format `user@host[:port]`
- `command`: SSH-Client-Befehl (Standard: `ssh`)
- `workspaceRoot`: absoluter Remote-Stammpfad für Arbeitsbereiche je Geltungsbereich (Standard: `/tmp/openclaw-sandboxes`)
- `identityFile` / `certificateFile` / `knownHostsFile`: vorhandene lokale Dateien, die an OpenSSH übergeben werden
- `identityData` / `certificateData` / `knownHostsData`: Inline-Inhalte oder SecretRefs, die OpenClaw zur Laufzeit in temporären Dateien materialisiert
- `strictHostKeyChecking` / `updateHostKeys`: Optionen für die OpenSSH-Hostschlüsselrichtlinie (beide standardmäßig `true`)

**SSH-Authentifizierungsrangfolge:**

- `identityData` hat Vorrang vor `identityFile`
- `certificateData` hat Vorrang vor `certificateFile`
- `knownHostsData` hat Vorrang vor `knownHostsFile`
- SecretRef-basierte `*Data`-Werte werden aus dem aktiven Laufzeit-Snapshot der Secrets aufgelöst, bevor die Sandbox-Sitzung beginnt

**Verhalten des SSH-Backends:**

- initialisiert den Remote-Arbeitsbereich einmalig nach der Erstellung oder Neuerstellung
- behält anschließend den Remote-SSH-Arbeitsbereich als kanonische Quelle bei
- leitet `exec`, Dateiwerkzeuge und Medienpfade über SSH
- synchronisiert Remote-Änderungen nicht automatisch zurück zum Host
- unterstützt keine Sandbox-Browsercontainer

**Arbeitsbereichszugriff:**

- `none`: Sandbox-Arbeitsbereich je Geltungsbereich unter `~/.openclaw/sandboxes` (Standard)
- `ro`: Sandbox-Arbeitsbereich unter `/workspace`, Agentenarbeitsbereich schreibgeschützt unter `/agent` eingehängt
- `rw`: Agentenarbeitsbereich mit Lese-/Schreibzugriff unter `/workspace` eingehängt

**Geltungsbereich:**

- `session`: Container und Arbeitsbereich je Sitzung
- `agent`: ein Container und Arbeitsbereich je Agent (Standard)
- `shared`: gemeinsam genutzter Container und Arbeitsbereich (keine sitzungsübergreifende Isolation)

**OpenShell-Plugin-Konfiguration:**

```json5
{
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          mode: "mirror", // Spiegelung (Standard) | Remote
          command: "openshell",
          from: "openclaw",
          remoteWorkspaceDir: "/sandbox",
          remoteAgentWorkspaceDir: "/agent",
          gateway: "lab", // optional
          gatewayEndpoint: "https://lab.example", // optional
          policy: "strict", // optionale OpenShell-Richtlinien-ID
          providers: ["openai"], // optional
          autoProviders: true,
          timeoutSeconds: 120,
        },
      },
    },
  },
}
```

**OpenShell-Modus:**

- `mirror`: Remote-Bereich vor der Ausführung aus dem lokalen Bereich initialisieren und nach der Ausführung zurücksynchronisieren; der lokale Arbeitsbereich bleibt kanonisch
- `remote`: Remote-Bereich einmalig bei der Erstellung der Sandbox initialisieren und anschließend den Remote-Arbeitsbereich als kanonische Quelle beibehalten

Im Modus `remote` werden außerhalb von OpenClaw vorgenommene hostlokale Änderungen nach dem Initialisierungsschritt nicht automatisch in die Sandbox synchronisiert.
Der Transport erfolgt per SSH in die OpenShell-Sandbox, aber das Plugin verwaltet den Sandbox-Lebenszyklus und die optionale Spiegelungssynchronisierung.

**`setupCommand`** wird einmalig nach der Containererstellung ausgeführt (über `sh -lc`). Erfordert ausgehenden Netzwerkzugriff, ein beschreibbares Stammverzeichnis und den Root-Benutzer.

**Container verwenden standardmäßig `network: "none"`** — legen Sie `"bridge"` (oder ein benutzerdefiniertes Bridge-Netzwerk) fest, wenn der Agent ausgehenden Zugriff benötigt.
`"host"` ist blockiert. `"container:<id>"` ist standardmäßig blockiert, sofern Sie nicht ausdrücklich
`sandbox.docker.dangerouslyAllowContainerNamespaceJoin: true` festlegen (Notfallzugriff).
Codex-App-Server-Durchläufe in einer aktiven OpenClaw-Sandbox verwenden dieselbe Einstellung für ausgehenden Datenverkehr für ihren nativen Netzwerkzugriff im Codemodus.

**Eingehende Anhänge** werden unter `media/inbound/*` im aktiven Arbeitsbereich bereitgestellt.

**`docker.binds`** hängt zusätzliche Hostverzeichnisse ein; globale und agentenspezifische Bind-Mounts werden zusammengeführt.

**Sandbox-Browser** (`sandbox.browser.enabled`, Standard `false`): Chromium und CDP in einem Container. Die noVNC-URL wird in den System-Prompt eingefügt. Erfordert `browser.enabled` in `openclaw.json` nicht.
Der noVNC-Beobachterzugriff verwendet standardmäßig VNC-Authentifizierung, und OpenClaw gibt eine kurzlebige Token-URL aus, anstatt das Passwort in der gemeinsam genutzten URL offenzulegen.

- `allowHostControl: false` (Standard) verhindert, dass Sandbox-Sitzungen den Hostbrowser ansprechen.
- `network` verwendet standardmäßig `openclaw-sandbox-browser` (dediziertes Bridge-Netzwerk). Legen Sie `bridge` nur fest, wenn Sie ausdrücklich eine globale Bridge-Konnektivität wünschen. `"host"` ist auch hier blockiert.
- `cdpSourceRange` beschränkt optional den eingehenden CDP-Zugriff am Containerrand auf einen CIDR-Bereich (zum Beispiel `172.21.0.1/32`).
- `sandbox.browser.binds` hängt zusätzliche Hostverzeichnisse ausschließlich in den Sandbox-Browsercontainer ein. Wenn dieser Wert festgelegt ist (einschließlich `[]`), ersetzt er `docker.binds` für den Browsercontainer.
- Chromium im Sandbox-Browsercontainer wird immer mit `--no-sandbox --disable-setuid-sandbox` gestartet (Container verfügen nicht über die Kernel-Primitive, die Chromes eigene Sandbox benötigt); hierfür gibt es keine Konfigurationsoption.
- Die Startstandards sind in `scripts/sandbox-browser-entrypoint.sh` definiert und für Containerhosts optimiert:
  - `--remote-debugging-address=127.0.0.1`
  - `--remote-debugging-port=<derived from OPENCLAW_BROWSER_CDP_PORT>`
  - `--user-data-dir=${HOME}/.chrome`
  - `--no-first-run`
  - `--no-default-browser-check`
  - `--disable-dev-shm-usage`
  - `--disable-background-networking`
  - `--disable-breakpad`
  - `--disable-crash-reporter`
  - `--no-zygote`
  - `--metrics-recording-only`
  - `--password-store=basic`
  - `--use-mock-keychain`
  - `--disable-3d-apis`, `--disable-gpu` und `--disable-software-rasterizer` sind
    standardmäßig aktiviert und können mit
    `OPENCLAW_BROWSER_DISABLE_GRAPHICS_FLAGS=0` deaktiviert werden, wenn die WebGL-/3D-Nutzung dies erfordert.
  - `--disable-extensions` (standardmäßig aktiviert); `OPENCLAW_BROWSER_DISABLE_EXTENSIONS=0`
    aktiviert Erweiterungen wieder, wenn Ihr Arbeitsablauf von ihnen abhängt.
  - standardmäßig `--renderer-process-limit=2`; ändern Sie dies mit
    `OPENCLAW_BROWSER_RENDERER_PROCESS_LIMIT=<N>`; legen Sie `0` fest, um Chromiums
    standardmäßige Prozessbegrenzung zu verwenden.
  - `--headless=new` nur, wenn `headless` aktiviert ist.
  - Die Standardwerte entsprechen der Basis des Container-Images; verwenden Sie ein benutzerdefiniertes Browser-Image mit einem benutzerdefinierten
    Einstiegspunkt, um die Containerstandardwerte zu ändern.

</Accordion>

Browser-Sandboxing und `sandbox.docker.binds` sind ausschließlich mit Docker verfügbar.

Images erstellen (aus einem Quellcode-Checkout):

```bash
scripts/sandbox-setup.sh           # Haupt-Sandbox-Image
scripts/sandbox-browser-setup.sh   # optionales Browser-Image
```

Informationen zu npm-Installationen ohne Quellcode-Checkout finden Sie unter [Sandboxing § Images und Einrichtung](/de/gateway/sandboxing#images-and-setup) für Inline-Befehle vom Typ `docker build`.

### `agents.entries` (agentenspezifische Überschreibungen)

Verwenden Sie `agents.entries.*.tts`, um einem Agenten einen eigenen TTS-Provider, eine eigene Stimme, ein eigenes Modell,
einen eigenen Stil oder einen eigenen automatischen TTS-Modus zuzuweisen. Der Agentenblock wird rekursiv über die globale
Konfiguration `tts` gelegt, sodass gemeinsam genutzte Anmeldedaten an einer Stelle verbleiben können, während einzelne
Agenten nur die benötigten Sprach- oder Provider-Felder überschreiben. Die Überschreibung des aktiven Agenten
gilt für automatische gesprochene Antworten, `/tts audio`, `/tts status` und
das Agentenwerkzeug `tts`. Beispiele für Provider und die Rangfolge finden Sie unter [Text-to-Speech](/de/tools/tts#per-agent-voice-overrides).

```json5
{
  agents: {
    list: [
      {
        id: "main",
        default: true,
        name: "Main Agent",
        workspace: "~/.openclaw/workspace",
        agentDir: "~/.openclaw/agents/main/agent",
        model: "anthropic/claude-opus-4-6", // oder { primary, fallbacks }
        utilityModel: "openai/gpt-5.4-mini",
        thinkingDefault: "high", // agentenspezifische Überschreibung der Denkstufe
        reasoningDefault: "on", // agentenspezifische Überschreibung der Sichtbarkeit von Schlussfolgerungen
        fastModeDefault: false, // agentenspezifische Überschreibung des Schnellmodus
        params: { cacheRetention: "none" }, // überschreibt übereinstimmende defaults.models-Parameter nach Schlüssel
        tts: {
          providers: {
            elevenlabs: { speakerVoiceId: "EXAVITQu4vr4xnSDxMaL" },
          },
        },
        skills: ["docs-search"], // ersetzt agents.defaults.skills, wenn festgelegt
        identity: {
          name: "Samantha",
          theme: "helpful sloth",
          emoji: "🦥",
          avatar: "avatars/samantha.png",
        },
        groupChat: { mentionPatterns: ["@openclaw"] },
        sandbox: { mode: "off" },
        runtime: {
          type: "acp",
          acp: {
            agent: "codex",
            backend: "acpx",
            mode: "persistent", // persistent | oneshot
            cwd: "/workspace/openclaw",
          },
        },
        subagents: { allowAgents: ["*"] },
        tools: {
          profile: "coding",
          allow: ["browser"],
          deny: ["canvas"],
          elevated: { enabled: true },
        },
      },
    ],
  },
}
```

- `id`: stabile Agent-ID (erforderlich).
- `default`: Wenn mehrere festgelegt sind, hat der erste Vorrang (Warnung wird protokolliert). Wenn keiner festgelegt ist, ist der erste Listeneintrag der Standardwert.
- `model`: Die Zeichenkettenform legt ein striktes agentenspezifisches Primärmodell ohne Modell-Fallback fest; die Objektform `{ primary }` ist ebenfalls strikt, sofern Sie nicht `fallbacks` hinzufügen. Verwenden Sie `{ primary, fallbacks: [...] }`, um Fallbacks für diesen Agent zu aktivieren, oder `{ primary, fallbacks: [] }`, um das strikte Verhalten ausdrücklich festzulegen. Cron-Aufträge, die nur `primary` überschreiben, übernehmen weiterhin die Standard-Fallbacks, sofern Sie nicht `fallbacks: []` festlegen.
- `utilityModel`: optionale agentenspezifische Überschreibung für kurze interne Aufgaben wie generierte Sitzungs- und Thread-Titel. Fällt auf `agents.defaults.utilityModel` und danach auf den deklarierten Standardwert für kleine Modelle des effektiven Sitzungs-Providers zurück. Dashboard-Titel versuchen es einmal erneut mit dem effektiven regulären Sitzungsmodell. Eine leere Zeichenkette überspringt die alternative Hilfsroute für diesen Agent, ohne die Generierung von Dashboard-Titeln zu deaktivieren.
- `params`: agentenspezifische Stream-Parameter, die über den ausgewählten Modelleintrag in `agents.defaults.models` gelegt werden. Verwenden Sie dies für agentenspezifische Überschreibungen wie `cacheRetention`, `temperature` oder `maxTokens`, ohne den gesamten Modellkatalog zu duplizieren.
- `tts`: optionale agentenspezifische Text-zu-Sprache-Überschreibungen. Der Block wird rekursiv über `tts` zusammengeführt. Belassen Sie daher gemeinsam genutzte Provider-Anmeldedaten und die Fallback-Richtlinie in `tts` und legen Sie hier nur personenspezifische Werte wie Provider, Stimme, Modell, Stil oder Automatikmodus fest.
- `skills`: optionale agentenspezifische Skills-Zulassungsliste. Wenn sie fehlt, übernimmt der Agent `agents.defaults.skills`, sofern festgelegt; eine ausdrückliche Liste ersetzt die Standardwerte, statt sie zusammenzuführen, und `[]` bedeutet keine Skills.
- `thinkingDefault`: optionale agentenspezifische Standard-Denkstufe (`off | minimal | low | medium | high | xhigh | adaptive | max`). Überschreibt `agents.defaults.thinkingDefault` für diesen Agent, wenn keine nachrichten- oder sitzungsspezifische Überschreibung festgelegt ist. Das ausgewählte Provider-/Modellprofil bestimmt, welche Werte gültig sind; bei Google Gemini behält `adaptive` das vom Provider gesteuerte dynamische Denken bei (`thinkingLevel` wird bei Gemini 3/3.1 weggelassen, `thinkingBudget: -1` bei Gemini 2.5).
- `reasoningDefault`: optionale agentenspezifische Standardsichtbarkeit der Schlussfolgerungen (`on | off | stream`). Überschreibt `agents.defaults.reasoningDefault` für diesen Agent, wenn keine nachrichten- oder sitzungsspezifische Überschreibung der Schlussfolgerungen festgelegt ist.
- `fastModeDefault`: optionaler agentenspezifischer Standardwert für den Schnellmodus (`"auto" | true | false`). Gilt, wenn keine nachrichten- oder sitzungsspezifische Überschreibung des Schnellmodus festgelegt ist.
- `models`: optionale agentenspezifische Überschreibungen des Modellkatalogs bzw. der Laufzeit, die nach vollständigen `provider/model`-IDs indiziert sind. Verwenden Sie `models["provider/model"].agentRuntime` für agentenspezifische Laufzeitausnahmen.
- `runtime`: optionale agentenspezifische Laufzeitbeschreibung. Verwenden Sie `type: "acp"` mit den `runtime.acp`-Standardwerten (`agent`, `backend`, `mode`, `cwd`), wenn der Agent standardmäßig ACP-Harness-Sitzungen verwenden soll.
- `identity.avatar`: arbeitsbereichsrelativer Pfad, `http(s)`-URL oder `data:`-URI.
- Lokale arbeitsbereichsrelative `identity.avatar`-Bilddateien sind auf 2 MB begrenzt. `http(s)`-URLs und `data:`-URIs werden nicht anhand der lokalen Dateigrößenbegrenzung geprüft.
- `identity` leitet Standardwerte ab: `ackReaction` aus `emoji`, `mentionPatterns` aus `name`/`emoji`.
- `subagents.allowAgents`: Zulassungsliste konfigurierter Agent-IDs für ausdrückliche `sessions_spawn.agentId`-Ziele (`["*"]` = jedes konfigurierte Ziel; Standard: nur derselbe Agent). Nehmen Sie die ID des Anforderers auf, wenn selbstadressierte `agentId`-Aufrufe zulässig sein sollen. Veraltete Einträge, deren Agent-Konfiguration gelöscht wurde, werden von `sessions_spawn` abgelehnt und aus `agents_list` weggelassen; führen Sie `openclaw doctor --fix` aus, um sie zu bereinigen, oder fügen Sie einen minimalen `agents.entries.*`-Eintrag hinzu, wenn dieses Ziel weiterhin erzeugt werden können und dabei Standardwerte übernehmen soll.
- Schutz für die Sandbox-Vererbung: Wenn die Sitzung des Anforderers in einer Sandbox ausgeführt wird, lehnt `sessions_spawn` Ziele ab, die ohne Sandbox ausgeführt würden.
- `subagents.requireAgentId`: Wenn wahr, werden `sessions_spawn`-Aufrufe blockiert, die `agentId` auslassen (erzwingt die ausdrückliche Profilauswahl; Standard: falsch).
- `subagents.maxConcurrent`: maximale Anzahl gleichzeitig ausgeführter untergeordneter Agents über die Subagent-Ausführung hinweg. Standard: `8`.
- `subagents.maxChildrenPerAgent`: maximale Anzahl aktiver untergeordneter Agents, die eine einzelne Agent-Sitzung erzeugen kann. Standard: `5`.
- `subagents.maxSpawnDepth`: maximale Verschachtelungstiefe beim Erzeugen von Subagents (`1`-`5`). Standard: `1` (keine Verschachtelung).
- `subagents.archiveAfterMinutes`: Zeitspanne, nach der der Zustand abgeschlossener Subagents archiviert wird. Standard: `60`.

---

## Multi-Agent-Routing

Führen Sie mehrere isolierte Agents innerhalb eines Gateways aus. Siehe [Multi-Agent](/de/concepts/multi-agent).

```json5
{
  agents: {
    list: [
      { id: "home", default: true, workspace: "~/.openclaw/workspace-home" },
      { id: "work", workspace: "~/.openclaw/workspace-work" },
    ],
  },
  bindings: [
    { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
    { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } },
  ],
}
```

### Abgleichfelder für Bindungen

- `type` (optional): `route` für normales Routing (bei fehlendem Typ wird standardmäßig die Route verwendet), `acp` für persistente ACP-Konversationsbindungen.
- `match.channel` (erforderlich)
- `match.accountId` (optional; `*` = jedes Konto; weggelassen = Standardkonto)
- `match.peer` (optional; `{ kind: direct|group|channel, id }`)
- `match.guildId` / `match.teamId` (optional; kanalspezifisch)
- `acp` (optional; nur für `type: "acp"`): `{ mode, label, cwd, backend }`

**Deterministische Abgleichreihenfolge:**

1. `match.peer`
2. `match.guildId`
3. `match.teamId`
4. `match.accountId` (exakt, kein Peer/Guild/Team)
5. `match.accountId: "*"` (kanalweit)
6. Standard-Agent

Innerhalb jeder Stufe hat der erste passende `bindings`-Eintrag Vorrang.

Bei `type: "acp"`-Einträgen erfolgt die Auflösung durch OpenClaw anhand der exakten Konversationsidentität (`match.channel` + Konto + `match.peer.id`); die oben aufgeführte Stufenreihenfolge der Routenbindungen wird nicht verwendet.

### Agentenspezifische Zugriffsprofile

<Accordion title="Vollzugriff (keine Sandbox)">

```json5
{
  agents: {
    list: [
      {
        id: "personal",
        workspace: "~/.openclaw/workspace-personal",
        sandbox: { mode: "off" },
      },
    ],
  },
}
```

</Accordion>

<Accordion title="Schreibgeschützte Werkzeuge und schreibgeschützter Arbeitsbereich">

```json5
{
  agents: {
    list: [
      {
        id: "family",
        workspace: "~/.openclaw/workspace-family",
        sandbox: { mode: "all", scope: "agent", workspaceAccess: "ro" },
        tools: {
          allow: [
            "read",
            "sessions_list",
            "sessions_history",
            "sessions_send",
            "sessions_spawn",
            "session_status",
          ],
          deny: ["write", "edit", "apply_patch", "exec", "process", "browser"],
        },
      },
    ],
  },
}
```

</Accordion>

<Accordion title="Kein Dateisystemzugriff (nur Nachrichten)">

```json5
{
  agents: {
    list: [
      {
        id: "public",
        workspace: "~/.openclaw/workspace-public",
        sandbox: { mode: "all", scope: "agent", workspaceAccess: "none" },
        tools: {
          allow: [
            "sessions_list",
            "sessions_history",
            "sessions_send",
            "sessions_spawn",
            "session_status",
            "whatsapp",
            "telegram",
            "slack",
            "discord",
            "gateway",
          ],
          deny: [
            "read",
            "write",
            "edit",
            "apply_patch",
            "exec",
            "process",
            "browser",
            "canvas",
            "nodes",
            "cron",
            "gateway",
            "image",
          ],
        },
      },
    ],
  },
}
```

</Accordion>

Einzelheiten zur Rangfolge finden Sie unter [Multi-Agent-Sandbox und -Werkzeuge](/de/tools/multi-agent-sandbox-tools).

---

## Sitzung

```json5
{
  session: {
    scope: "per-sender",
    dmScope: "main", // main | per-peer | per-channel-peer | per-account-channel-peer
    identityLinks: {
      alice: ["telegram:123456789", "discord:987654321012345678"],
    },
    reset: {
      mode: "daily", // daily | idle
      atHour: 4,
      idleMinutes: 60,
    },
    resetByType: {
      thread: { mode: "daily", atHour: 4 },
      direct: { mode: "idle", idleMinutes: 240 },
      group: { mode: "idle", idleMinutes: 120 },
    },
    resetByChannel: {
      discord: { mode: "idle", idleMinutes: 30 },
    },
    resetTriggers: ["/new", "/reset"],
    store: "~/.openclaw/agents/{agentId}/sessions/sessions.json",
    maintenance: {
      mode: "enforce", // enforce (Standard) | warn
      pruneAfter: "30d",
      maxEntries: 500,
      resetArchiveRetention: "30d", // Dauer oder false
      maxDiskBytes: "500mb", // optionales festes Budget
      highWaterBytes: "400mb", // optionales Bereinigungsziel
    },
    threadBindings: {
      enabled: true,
      idleHours: 24, // standardmäßige automatische Aufhebung des Fokus nach Inaktivität in Stunden (`0` deaktiviert dies)
      maxAgeHours: 0, // standardmäßiges festes Höchstalter in Stunden (`0` deaktiviert dies)
    },
    sharing: {
      readOnly: true,
      suggest: true,
      drafts: true,
    },
    mainKey: "main", // veraltet (die Laufzeit verwendet immer "main")
    sendPolicy: {
      rules: [{ action: "deny", match: { channel: "discord", chatType: "group" } }],
      default: "allow",
    },
  },
}
```

<Accordion title="Details zu Sitzungsfeldern">

- **`scope`**: grundlegende Strategie zur Sitzungsgruppierung für Gruppenchats.
  - `per-sender` (Standard): Jeder Absender erhält innerhalb eines Kanalkontexts eine isolierte Sitzung.
  - `global`: Alle Teilnehmer in einem Kanalkontext teilen sich eine einzige Sitzung (nur verwenden, wenn ein gemeinsamer Kontext beabsichtigt ist).
- **`dmScope`**: Gruppierung von Direktnachrichten.
  - `main`: Alle Direktnachrichten teilen sich die Hauptsitzung.
  - `per-peer`: kanalübergreifend nach Absender-ID isolieren.
  - `per-channel-peer`: nach Kanal und Absender isolieren (für Posteingänge mit mehreren Benutzern empfohlen).
  - `per-account-channel-peer`: nach Konto, Kanal und Absender isolieren (für mehrere Konten empfohlen).
- **`identityLinks`**: kanonische IDs für die kanalübergreifende Sitzungsfreigabe Provider-präfixierten Kommunikationspartnern zuordnen. Andockbefehle wie `/dock_discord` verwenden dieselbe Zuordnung, um die Antwortweiterleitung der aktiven Sitzung auf einen anderen verknüpften Kommunikationspartner eines Kanals umzustellen; siehe [Kanal-Andocken](/de/concepts/channel-docking).
- **`reset`**: primäre Richtlinie zum Zurücksetzen. `none` deaktiviert das automatische Zurücksetzen und ist der Standard; stattdessen begrenzt Compaction den aktiven Kontext. `daily` setzt um `atHour` Ortszeit zurück; `idle` setzt nach `idleMinutes` zurück. Wenn beide konfiguriert sind, gilt der zuerst ablaufende Wert. `/new` und `/reset` bleiben in jedem Modus verfügbar. Für die Aktualität des täglichen Zurücksetzens wird `sessionStartedAt` der Sitzungszeile verwendet; für die Aktualität des Leerlauf-Zurücksetzens wird `lastInteractionAt` verwendet. Schreibvorgänge durch Hintergrund- oder Systemereignisse wie Heartbeat, Cron-Aktivierungen, Ausführungsbenachrichtigungen und Gateway-Verwaltung können `updatedAt` aktualisieren, halten tägliche oder Leerlaufsitzungen jedoch nicht aktuell.
  - **`resetByType`**: typbezogene Überschreibungen (`direct`, `group`, `thread`). Doctor migriert veraltete `dm`-Einträge zu `direct`; das Schema lehnt `dm` ab.
- **`resetByChannel`**: kanalbezogene Überschreibungen des Zurücksetzens, die nach Provider-/Kanal-ID verschlüsselt sind. Wenn der Kanal der Sitzung einen passenden Eintrag besitzt, hat dieser für die betreffende Sitzung uneingeschränkt Vorrang vor `resetByType`/`reset`. Nur verwenden, wenn ein Kanal ein von der typbezogenen Richtlinie abweichendes Rücksetzverhalten benötigt.
- **`mainKey`**: veraltetes Feld. Die Laufzeit verwendet für den Hauptbereich direkter Chats immer `"main"`.
- **`sendPolicy`**: Abgleich anhand von `channel`, `chatType` (`direct|group|channel`, mit dem veralteten Alias `dm`), `keyPrefix` oder `rawKeyPrefix`. Die erste Ablehnung gilt.
- **`maintenance`**: Bereinigung des Sitzungsspeichers und Aufbewahrungssteuerung.
  - `mode`: `enforce` führt die Bereinigung durch und ist der Standard; `warn` gibt nur Warnungen aus.
  - `pruneAfter`: Altersgrenze für veraltete Einträge (Standard: `30d`).
  - `maxEntries`: maximale Anzahl von SQLite-Sitzungseinträgen (Standard: `500`). Laufzeitschreibvorgänge führen die Bereinigung stapelweise mit einem kleinen Hochwasserpuffer für produktionsübliche Obergrenzen aus; `openclaw sessions cleanup --enforce` wendet die Obergrenze sofort an.
  - Kurzlebige Gateway-Prüfsitzungen für Modellläufe verwenden eine feste Aufbewahrungsdauer von `24h`, die Bereinigung erfolgt jedoch nur bei entsprechendem Druck: Veraltete Zeilen strikt definierter Modelllauf-Prüfungen werden nur entfernt, wenn die Wartung der Sitzungseinträge beziehungsweise die Obergrenze dies erfordert. Nur strikt explizite Prüfschlüssel, die `agent:*:explicit:model-run-<uuid>` entsprechen, kommen infrage; normale Direkt-, Gruppen-, Thread-, Cron-, Hook-, Heartbeat-, ACP- und Unteragentensitzungen übernehmen diese 24-stündige Aufbewahrung nicht. Wenn die Modelllaufbereinigung ausgeführt wird, erfolgt sie vor der umfassenderen Bereinigung veralteter Einträge gemäß `pruneAfter` und vor der Obergrenze `maxEntries`.
  - Das veraltete `rotateBytes` wird vom aktuellen Schema abgelehnt; `openclaw doctor --fix` entfernt es aus älteren Konfigurationen.
  - `resetArchiveRetention`: altersbasierte Aufbewahrung zurückgesetzter oder gelöschter Transkriptarchive. Standardmäßig bleiben Archive bis zur Verdrängung aufgrund des Datenträgerbudgets erhalten; legen Sie eine Dauer fest, um die Löschung nach verstrichener Zeit zu aktivieren, oder `false`, um sie ausdrücklich zu deaktivieren.
  - `maxDiskBytes`: optionales Datenträgerbudget für das Sitzungsverzeichnis. Im Modus `warn` werden Warnungen protokolliert; im Modus `enforce` werden die ältesten Artefakte/Sitzungen zuerst entfernt.
  - `highWaterBytes`: optionaler Zielwert nach der budgetbedingten Bereinigung. Standardmäßig `80%` von `maxDiskBytes`.
- **`threadBindings`**: globale Standardwerte für Thread-gebundene Sitzungsfunktionen.
  - `enabled`: Hauptschalter für unterstützte Kanal-Thread-Bindungen
  - `idleHours`: standardmäßige automatische Aufhebung des Fokus nach Inaktivität in Stunden (`0` deaktiviert sie; Provider können den Wert überschreiben)
  - `maxAgeHours`: standardmäßiges maximales Höchstalter in Stunden (`0` deaktiviert es; Provider können den Wert überschreiben)
  - `spawnSessions`: standardmäßige Zugangssteuerung für das Erstellen Thread-gebundener Arbeitssitzungen aus `sessions_spawn` und ACP-Thread-Starts. Standardmäßig `true`, wenn Thread-Bindungen aktiviert sind; Provider/Konten können den Wert überschreiben.
  - `defaultSpawnContext`: standardmäßiger nativer Unteragentenkontext für Thread-gebundene Starts (`"fork"` oder `"isolated"`). Standardmäßig `"fork"`.
- **`sharing`**: steuert, welche sitzungsbezogenen Zusammenarbeitsmodi Eigentümer und `operator.admin`-Verbindungen auswählen dürfen. Jedes Flag hat standardmäßig den Wert `true`; wird eines auf `false` gesetzt, wird diese Auswahl aus der Control UI entfernt und bei der Sichtbarkeit zum Erstellungszeitpunkt oder durch `session.visibility.set` abgelehnt. Neue Sitzungen beginnen als `shared`, sofern sie nicht über die Control UI als Entwurf gestartet werden.
  - `readOnly`: `read-only` zulassen, wobei Nichtmitglieder zusehen, aber keine Nachrichten senden, steuern, abbrechen, genehmigen oder den Sitzungsstatus ändern können.
  - `suggest`: `suggest` zulassen. In dieser Phase wird dasselbe Zugangsverhalten wie bei `read-only` erzwungen; die Vorschlagswarteschlange ist eine spätere Funktion.
  - `drafts`: `draft` zulassen, wodurch die Sitzung in Sitzungslisten und Ereignisübertragungen vor Personen verborgen wird, die weder Administratoren noch Eigentümer sind.

Änderungen an Mitgliedschaft und Sichtbarkeit werden als Systemhinweise in das Sitzungstranskript geschrieben. Diese Steuerungen koordinieren Operatoren, die gemeinsam einen Agenten verwenden; sie stellen keine Sicherheitsgrenze zwischen Mandanten dar. Verwenden Sie separate Gateways oder Agenten, wenn die Arbeit eine Isolierung erfordert.

</Accordion>

---

## Nachrichten

```json5
{
  messages: {
    responsePrefix: "🦞", // oder "auto"
    ackReaction: "👀",
    ackReactionScope: "group-mentions", // group-mentions | group-all | direct | all | off | none
    queue: {
      mode: "steer", // steer (Standard) | followup | collect | interrupt
      debounceMs: 500,
      cap: 20,
      drop: "summarize", // old | new | summarize (Standard)
      byChannel: {
        whatsapp: "followup",
        telegram: "followup",
      },
    },
    inbound: {
      debounceMs: 2000, // 0 deaktiviert
      byChannel: {
        whatsapp: 5000,
        slack: 1500,
      },
    },
  },
}
```

### Antwortpräfix

Kanal-/kontobezogene Überschreibungen: `channels.<channel>.responsePrefix`, `channels.<channel>.accounts.<id>.responsePrefix`.

Auflösung (der spezifischste Wert gilt): Konto → Kanal → global. `""` deaktiviert die Funktion und beendet die Kaskade. `"auto"` leitet `[{identity.name}]` ab.

**Vorlagenvariablen:**

| Variable          | Beschreibung            | Beispiel                     |
| ----------------- | ---------------------- | --------------------------- |
| `{model}`         | Kurzer Modellname       | `claude-opus-4-6`           |
| `{modelFull}`     | Vollständige Modellkennung  | `anthropic/claude-opus-4-6` |
| `{provider}`      | Providername          | `anthropic`                 |
| `{thinkingLevel}` | Aktuelle Denkstufe | `high`, `low`, `off`        |
| `{identity.name}` | Name der Agentenidentität    | (identisch mit `"auto"`)          |

Bei Variablen wird nicht zwischen Groß- und Kleinschreibung unterschieden. `{think}` ist ein Alias für `{thinkingLevel}`.

### Bestätigungsreaktion

- Standardmäßig wird `identity.emoji` des aktiven Agenten verwendet, andernfalls `"👀"`. Zum Deaktivieren `""` festlegen.
- Kanalbezogene Überschreibungen: `channels.<channel>.ackReaction`, `channels.<channel>.accounts.<id>.ackReaction`.
- Auflösungsreihenfolge: Konto → Kanal → `messages.ackReaction` → Rückgriff auf die Identität.
- Geltungsbereich: `group-mentions` (Standard), `group-all`, `direct`, `all` oder `off`/`none` (deaktiviert Bestätigungsreaktionen vollständig).
- `messages.statusReactions.enabled`: aktiviert Reaktionen auf Lebenszyklusstatus bei Slack, Discord, Signal, Telegram und WhatsApp.
  Bei Discord bleiben Statusreaktionen ohne festgelegten Wert aktiviert, wenn Bestätigungsreaktionen aktiv sind.
  Bei Slack, Signal, Telegram und WhatsApp muss der Wert ausdrücklich auf `true` gesetzt werden, um Reaktionen auf Lebenszyklusstatus zu aktivieren.
  Slack verwendet standardmäßig seinen nativen Assistenten-Thread-Status und wechselnde Lademeldungen für den Fortschritt, während die konfigurierte Bestätigungsreaktion unverändert bleibt.

### Warteschlange

- `mode`: Warteschlangenstrategie für eingehende Nachrichten, die eintreffen, während ein Sitzungslauf aktiv ist. Standard: `"steer"`.
  - `steer`: die neue Eingabeaufforderung in den aktiven Lauf einfügen.
  - `followup`: die neue Eingabeaufforderung ausführen, nachdem der aktive Lauf abgeschlossen ist.
  - `collect`: kompatible Nachrichten bündeln und später gemeinsam ausführen.
  - `interrupt`: den aktiven Lauf abbrechen, bevor die neueste Eingabeaufforderung gestartet wird.
- `debounceMs`: Verzögerung vor dem Weiterleiten einer in die Warteschlange gestellten oder steuernden Nachricht. Standard: `500`.
- `cap`: maximale Anzahl von Nachrichten in der Warteschlange, bevor die Verwerfungsrichtlinie angewendet wird. Standard: `20`.
- `drop`: Strategie bei Überschreitung der Obergrenze. `"summarize"` (Standard) verwirft die ältesten Einträge, behält jedoch kompakte Zusammenfassungen bei; `"old"` verwirft die ältesten Einträge ohne Zusammenfassungen; `"new"` lehnt den neuesten Eintrag ab.
- `byChannel`: kanalbezogene `mode`-Überschreibungen, die nach Provider-ID verschlüsselt sind.
- `debounceMsByChannel`: kanalbezogene `debounceMs`-Überschreibungen, die nach Provider-ID verschlüsselt sind.

### Entprellung eingehender Nachrichten

Bündelt schnell aufeinanderfolgende reine Textnachrichten desselben Absenders zu einem einzigen Agentendurchlauf. Medien/Anhänge lösen die Verarbeitung sofort aus. Steuerbefehle umgehen die Entprellung. Standardwert für `debounceMs`: `2000`.

### Weitere Nachrichtenschlüssel

- `channels.whatsapp.responsePrefix`: Präfix für ausgehende WhatsApp-Antworten. Doctor verschiebt den veralteten eingehenden Wert `messagePrefix` nur hierher, wenn dieser kanonische Wert nicht festgelegt ist.
- `messages.visibleReplies`: steuert sichtbare Quellantworten in Direkt-, Gruppen- und Kanalunterhaltungen (`"message_tool"` erfordert `message(action=send)` für eine sichtbare Ausgabe; `"automatic"` veröffentlicht normale Antworten wie zuvor).
- `messages.usageTemplate` / `messages.responseUsage`: benutzerdefinierte `/usage`-Fußzeilenvorlage und standardmäßiger Verwendungsmodus pro Antwort (`off | tokens | full`, zusätzlich der veraltete Alias `on` für `tokens`).
- `messages.groupChat.mentionPatterns` / `historyLimit`: Auslöser für Erwähnungen in Gruppennachrichten und Größe des Verlaufsfensters.
- `messages.suppressToolErrors`: unterdrückt bei `true` die dem Benutzer angezeigten `⚠️`-Werkzeugfehlerwarnungen (der Agent sieht die Fehler weiterhin im Kontext und kann den Vorgang erneut versuchen). Standard: `false`.

### TTS (Text-zu-Sprache)

```json5
{
  tts: {
    auto: "off", // off (default) | always | inbound | tagged
    mode: "final", // final | all
    provider: "elevenlabs",
    summaryModel: "openai/gpt-5.4-mini",
    modelOverrides: { enabled: true },
    maxTextLength: 4000,
    timeoutMs: 30000,
    providers: {
      elevenlabs: {
        apiKey: "example-elevenlabs-api-key",
        baseUrl: "https://api.elevenlabs.io",
        speakerVoiceId: "voice_id",
        modelId: "eleven_multilingual_v2",
        seed: 42,
        applyTextNormalization: "auto",
        languageCode: "en",
        voiceSettings: {
          stability: 0.5,
          similarityBoost: 0.75,
          style: 0.0,
          useSpeakerBoost: true,
          speed: 1.0,
        },
      },
      microsoft: {
        speakerVoice: "en-US-MichelleNeural",
        lang: "en-US",
        outputFormat: "audio-24khz-48kbitrate-mono-mp3",
      },
      openai: {
        apiKey: "example-openai-api-key",
        baseUrl: "https://api.openai.com/v1",
        model: "gpt-4o-mini-tts",
        speakerVoice: "coral",
      },
    },
  },
}
```

Der globale Einstellungspfad ist ein Maschinenzustand (standardmäßig
`~/.openclaw/settings/tts.json`; mit `OPENCLAW_TTS_PREFS` überschreiben). Erweiterte
Multi-Agent-Konfigurationen können `agents.entries.<id>.tts.prefsPath` für separate
agentenspezifische Einstellungsspeicher festlegen.

- `auto` steuert den standardmäßigen automatischen TTS-Modus: `off`, `always`, `inbound` oder `tagged`. `/tts on|off` kann lokale Einstellungen überschreiben, und `/tts status` zeigt den wirksamen Zustand an.
- `summaryModel` überschreibt `agents.defaults.model.primary` für die automatische Zusammenfassung.
- `modelOverrides` ist standardmäßig aktiviert (`enabled !== false`); `modelOverrides.allowProvider` muss explizit aktiviert werden.
- API-Schlüssel greifen ersatzweise auf `ELEVENLABS_API_KEY`/`XI_API_KEY` und `OPENAI_API_KEY` zurück.
- Mitgelieferte Sprachausgabe-Provider gehören den jeweiligen Plugins. Wenn `plugins.allow` festgelegt ist, nehmen Sie jedes gewünschte TTS-Provider-Plugin auf, beispielsweise `microsoft` für Edge TTS. Die alte Provider-ID `edge` wird als Alias für `microsoft` akzeptiert.
- `providers.openai.baseUrl` überschreibt den OpenAI-TTS-Endpunkt. Die Auflösungsreihenfolge lautet: Konfiguration, dann `OPENAI_TTS_BASE_URL`, dann `https://api.openai.com/v1`.
- Wenn `providers.openai.baseUrl` auf einen Nicht-OpenAI-Endpunkt verweist, behandelt OpenClaw ihn als OpenAI-kompatiblen TTS-Server und lockert die Modell- und Stimmenvalidierung.

---

## Talk

Standardwerte für den Talk-Modus (macOS/iOS/Android und die Control UI im Browser).

```json5
{
  talk: {
    provider: "elevenlabs",
    providers: {
      elevenlabs: {
        speakerVoiceId: "elevenlabs_voice_id",
        voiceAliases: {
          Clawd: "EXAVITQu4vr4xnSDxMaL",
          Roger: "CwhRBWXzGAHq8TQ4Fs17",
        },
        modelId: "eleven_multilingual_v2",
        outputFormat: "mp3_44100_128",
        apiKey: "elevenlabs_api_key",
      },
      mlx: {
        modelId: "mlx-community/Soprano-80M-bf16",
      },
      system: {},
    },
    consultThinkingLevel: "low",
    consultFastMode: true,
    speechLocale: "ru-RU",
    silenceTimeoutMs: 1500,
    interruptOnSpeech: true,
    realtime: {
      provider: "openai",
      providers: {
        openai: {
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
        },
      },
      instructions: "Sprechen Sie freundlich und halten Sie die Antworten kurz.",
      mode: "realtime", // realtime | stt-tts | transcription
      transport: "webrtc", // webrtc | provider-websocket | gateway-relay | managed-room
      vadThreshold: 0.5,
      silenceDurationMs: 500,
      prefixPaddingMs: 300,
      reasoningEffort: "medium",
      brain: "agent-consult", // agent-consult | direct-tools | none
    },
  },
}
```

- `talk.provider` muss einem Schlüssel in `talk.providers` entsprechen, wenn mehrere Talk-Provider konfiguriert sind.
- Die alten flachen Talk-Schlüssel (`talk.voiceId`, `talk.voiceAliases`, `talk.modelId`, `talk.outputFormat`, `talk.apiKey`) dienen ausschließlich der Kompatibilität. Führen Sie `openclaw doctor --fix` aus, um die gespeicherte Konfiguration in `talk.providers.<provider>` umzuschreiben.
- Stimmen-IDs greifen ersatzweise auf `ELEVENLABS_VOICE_ID` oder `SAG_VOICE_ID` zurück (Verhalten des macOS-Talk-Clients).
- `providers.*.apiKey` akzeptiert Klartextzeichenfolgen oder SecretRef-Objekte.
- Der Rückgriff auf `ELEVENLABS_API_KEY` gilt nur, wenn kein Talk-API-Schlüssel konfiguriert ist.
- `providers.*.voiceAliases` ermöglicht Talk-Direktiven die Verwendung leicht verständlicher Namen.
- `providers.mlx.modelId` wählt das Hugging-Face-Repository aus, das vom lokalen MLX-Hilfsprogramm unter macOS verwendet wird. Wenn die Angabe fehlt, verwendet macOS `mlx-community/Soprano-80M-bf16`.
- Die MLX-Wiedergabe unter macOS erfolgt über das mitgelieferte Hilfsprogramm `openclaw-mlx-tts`, sofern vorhanden, oder über eine ausführbare Datei in `PATH`; `OPENCLAW_MLX_TTS_BIN` überschreibt den Pfad des Hilfsprogramms für die Entwicklung.
- `consultThinkingLevel` steuert die Denktiefe für den vollständigen OpenClaw-Agentenlauf hinter den Echtzeitaufrufen `openclaw_agent_consult` von Talk in der Control UI. Lassen Sie die Einstellung ungesetzt, um das normale Sitzungs- und Modellverhalten beizubehalten.
- `consultFastMode` legt eine einmalige Überschreibung des Schnellmodus für Echtzeitkonsultationen von Talk in der Control UI fest, ohne die normale Schnellmoduseinstellung der Sitzung zu ändern.
- `speechLocale` legt die BCP-47-Gebietsschema-ID fest, die von der Talk-Spracherkennung unter Android, iOS und macOS verwendet wird. Android verwendet außerdem deren Sprachkomponente, um die Echtzeittranskription der Eingabe zu steuern. Lassen Sie die Einstellung ungesetzt, um den Gerätestandard zu verwenden.
- `silenceTimeoutMs` steuert, wie lange der Talk-Modus nach einer Sprechpause des Benutzers wartet, bevor er das Transkript sendet. Wenn die Einstellung ungesetzt ist, bleibt das standardmäßige Pausenfenster der Plattform erhalten (`700 ms on macOS and Android, 900 ms on iOS`).
- `realtime.instructions` hängt an Provider gerichtete Systemanweisungen an die integrierte Echtzeitvorgabe von OpenClaw an, sodass sich der Sprachstil konfigurieren lässt, ohne die standardmäßigen Vorgaben von `openclaw_agent_consult` zu verlieren.
- `realtime.vadThreshold` legt den Schwellenwert des Providers für Sprachaktivität von `0` (höchste Empfindlichkeit) bis `1` (niedrigste Empfindlichkeit) fest. Wenn die Einstellung ungesetzt ist, bleibt der Standardwert des Providers erhalten.
- `realtime.silenceDurationMs` legt das positive ganzzahlige Stillefenster fest, nach dem der Provider einen Echtzeit-Benutzerbeitrag bestätigt. Wenn die Einstellung ungesetzt ist, bleibt der Standardwert des Providers erhalten.
- `realtime.prefixPaddingMs` legt die nicht negative ganzzahlige Audiomenge fest, die vor dem Beginn der erkannten Sprache beibehalten wird. Wenn die Einstellung ungesetzt ist, bleibt der Standardwert des Providers erhalten.
- `realtime.reasoningEffort` legt die providerspezifische Denktiefe für Echtzeitsitzungen fest. Wenn die Einstellung ungesetzt ist, bleibt der Standardwert des Providers erhalten.
- `realtime.consultRouting`: `"provider-direct"` (Standard) behält direkte Antworten des Providers bei, wenn der Echtzeit-Provider ein endgültiges Benutzertranskript ohne `openclaw_agent_consult` erzeugt. `"force-agent-consult"` leitet die abgeschlossene Anfrage stattdessen über OpenClaw weiter.

---

## Verwandte Themen

- [Konfigurationsreferenz](/de/gateway/configuration-reference) — alle weiteren Konfigurationsschlüssel
- [Konfiguration](/de/gateway/configuration) — häufige Aufgaben und Schnelleinrichtung
- [Konfigurationsbeispiele](/de/gateway/configuration-examples)
