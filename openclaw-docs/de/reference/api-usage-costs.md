---
read_when:
    - Sie möchten verstehen, welche Funktionen möglicherweise kostenpflichtige APIs aufrufen.
    - Sie müssen Schlüssel, Kosten und die Sichtbarkeit der Nutzung überprüfen.
    - Sie erklären die Kostenangaben von `/status` oder `/usage`
summary: Prüfen Sie, wodurch Kosten entstehen können, welche Schlüssel verwendet werden und wie Sie die Nutzung anzeigen können
title: API-Nutzung und Kosten
x-i18n:
    generated_at: "2026-07-26T18:45:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 22caad8b8fa168739563223b3663a04adceeef7e83576a53dc9cdf885a35750d
    source_path: reference/api-usage-costs.md
    workflow: 16
---

Übersicht der OpenClaw-Funktionen, die kostenpflichtige Provider-APIs aufrufen können, wo sie jeweils ihre Anmeldedaten lesen und wo die daraus entstehenden Kosten angezeigt werden.

## Wo Kosten angezeigt werden

**`/status`** (Snapshot pro Sitzung)

- Zeigt das aktuelle Sitzungsmodell, die Kontextnutzung und die Token der letzten Antwort an.
- Fügt für die letzte Antwort **geschätzte Kosten** hinzu, wenn OpenClaw über Nutzungsmetadaten und lokale Preisinformationen für das aktive Modell verfügt. Dies schließt ausdrücklich bepreiste Provider ohne API-Schlüssel ein, etwa Bedrock-`aws-sdk`-Modelle.
- Wenn der Live-Sitzungs-Snapshot nur wenige Daten enthält, stellt `/status` Token-/Cache-Zähler und die Bezeichnung des aktiven Modells aus dem neuesten Nutzungseintrag des Transkripts wieder her. Vorhandene Live-Werte ungleich null haben Vorrang vor Transkriptdaten; eine der Prompt-Größe entsprechende Transkript-Gesamtsumme kann dennoch Vorrang haben, wenn die gespeicherte Gesamtsumme fehlt oder kleiner ist.

**`/usage`** (Fußzeile pro Nachricht)

- `/usage full` hängt an jede Antwort eine Nutzungsfußzeile an, einschließlich **geschätzter Kosten**, wenn lokale Preisinformationen konfiguriert und Nutzungsmetadaten verfügbar sind.
- `/usage tokens` zeigt nur Token an. Abonnementbasierte OAuth-/Token- und CLI-Runtimes zeigen nur Token an, sofern sie nicht kompatible Nutzungsmetadaten sowie einen expliziten lokalen Preis bereitstellen.
- `/usage cost` gibt eine lokale Kostenübersicht aus; `/usage off` deaktiviert die Fußzeile.
- Hinweis zur Gemini CLI: Sowohl die Ausgabe von `stream-json` als auch die ältere Ausgabe von `json` enthält Nutzungsdaten unter `stats`. OpenClaw normalisiert `stats.cached` in `cacheRead` und leitet bei Bedarf die Eingabe-Token aus `stats.input_tokens - stats.cached` ab.

**Control UI → Nutzung** (sitzungsübergreifende Analyse)

- Zeigt aus Transkripten abgeleitete Token-Gesamtsummen und geschätzte Gesamtkosten für den ausgewählten Datumsbereich an, aufgeschlüsselt nach Provider, Modell, Agent, Kanal und Token-Typ.
- Vergleicht kürzere Kalenderzeiträume, die am Enddatum des ausgewählten Bereichs enden. Fehlende Datumsangaben zählen als Kalendertage ohne Nutzung; sie werden nicht übersprungen, um einen dichteren Zeitraum zu erzeugen.
- Beschriftet die Skalierung des Tagesdiagramms direkt. Ein `√`-Badge bedeutet, dass eine Quadratwurzelkompression Tage mit geringer Nutzung sichtbar hält.
- Diese Gesamtsummen beschreiben den verfügbaren lokalen Sitzungsverlauf, nicht eine Provider-Rechnung oder ein Abrechnungsprotokoll über die gesamte Nutzungsdauer. Die Benutzeroberfläche warnt, wenn für einige Einträge Preisinformationen fehlen.

**CLI-Nutzungszeiträume** (Provider-Kontingente, keine Kosten pro Nachricht)

- `openclaw status --usage` und `openclaw channels list` zeigen **Nutzungszeiträume** des Providers als `X% left` an.
- Aktuelle Provider für Nutzungszeiträume: Anthropic, ClawRouter, DeepSeek, GitHub Copilot, Gemini CLI, MiniMax, OpenAI (umfasst ChatGPT/Codex-OAuth-/Token-Authentifizierung), Xiaomi und z.ai. Die vollständige Liste der Provider und Flags finden Sie unter [Modell-CLI](/de/cli/models) und [Kanal-CLI](/de/cli/channels).
- Die Rohfelder `usage_percent` / `usagePercent` von MiniMax geben das verbleibende Kontingent an, daher kehrt OpenClaw sie um; zählerbasierte Felder haben Vorrang, sofern vorhanden. Wenn die Antwort ein `model_remains`-Array enthält, wählt OpenClaw den Eintrag des Chatmodells aus, leitet bei Bedarf die Bezeichnung des Zeitraums aus Zeitstempeln ab und nimmt den Modellnamen in die Planbezeichnung auf.
- Die Nutzungsauthentifizierung stammt aus providerspezifischen Hooks, sofern verfügbar; andernfalls greift OpenClaw auf passende OAuth-/API-Schlüssel-Anmeldedaten aus Authentifizierungsprofilen, Umgebungsvariablen oder der Konfiguration zurück.

Ausführliche Beispiele finden Sie unter [Token-Nutzung und Kosten](/de/reference/token-use).

<Note>
Anthropic hat bestätigt, dass die Wiederverwendung der Claude CLI (einschließlich `claude -p`) ein zulässiges Integrationsmuster ist, sofern keine neue Richtlinie veröffentlicht wird. Anthropic stellt keine Kostenschätzung in Dollar pro Nachricht bereit, daher kann `/usage full` keine Kosten für die Nutzung der Claude CLI anzeigen.
</Note>

## So werden Schlüssel gefunden

- **Authentifizierungsprofile**: pro Agent, gespeichert in `auth-profiles.json`.
- **Umgebungsvariablen**: beispielsweise `OPENAI_API_KEY`, `BRAVE_API_KEY`, `FIRECRAWL_API_KEY`.
- **Konfiguration**: `models.providers.*.apiKey`, `plugins.entries.*.config.webSearch.apiKey`, `plugins.entries.firecrawl.config.webFetch.apiKey`, `memory.search.*`, `talk.providers.*.apiKey`.
- **Skills**: `skills.entries.<name>.apiKey`, wodurch der Schlüssel möglicherweise in die Prozessumgebung des Skills exportiert wird.

## Funktionen, die Schlüssel kostenpflichtig nutzen können

### Antworten des Kernmodells (Chat + Tools)

Jede Antwort und jeder Tool-Aufruf wird über den aktuellen Modell-Provider ausgeführt. Dies ist die Hauptquelle für Nutzung und Kosten, einschließlich abonnementbasierter gehosteter Tarife, deren Abrechnung außerhalb der lokalen Benutzeroberfläche von OpenClaw erfolgt: OpenAI Codex, Alibaba Cloud Model Studio Coding Plan, MiniMax Coding Plan, Z.AI/GLM Coding Plan und der Claude-Anmeldepfad von Anthropic mit aktivierter Option Extra Usage.

Informationen zur Preiskonfiguration finden Sie unter [Modelle](/de/providers/models), Informationen zur Anzeige unter [Token-Nutzung und Kosten](/de/reference/token-use).

### Medienanalyse (Audio/Bild/Video)

Eingehende Medien können über eine Provider-API zusammengefasst oder transkribiert werden, bevor die Antwort-Pipeline ausgeführt wird. Die Provider-Unterstützung wird pro Plugin registriert und ändert sich, wenn Plugins hinzugefügt werden; die aktuelle Liste und Konfiguration finden Sie unter [Medienanalyse](/de/nodes/media-understanding).

### Bild- und Videogenerierung

`image_generate` und `video_generate` leiten Anfragen an einen verfügbaren authentifizierten Provider weiter. Beide können einen durch Authentifizierung gestützten Standard-Provider ableiten, wenn ihr Eintrag `agents.defaults.mediaModels` nicht festgelegt ist.

Die aktuelle Provider-Liste finden Sie unter [Bildgenerierung](/de/tools/image-generation) und [Videogenerierung](/de/tools/video-generation).

### Speicher-Embeddings und semantische Suche

Die semantische Speichersuche verwendet Embedding-APIs, wenn `memory.search.provider` einen Remote-Adapter bezeichnet (zum Beispiel `openai`, `gemini`, `voyage`, `mistral`, `deepinfra`, `github-copilot`, `amazon-bedrock`). `memory.search.provider = "lmstudio"` oder `"ollama"` wird auf einem lokalen bzw. selbst gehosteten Server ausgeführt und verursacht üblicherweise keine Kosten für gehostete Dienste. `memory.search.provider = "local"` belässt alles auf dem Gerät, ohne API-Nutzung. Ein optionaler `memory.search.fallback`-Provider kann Ausfälle lokaler Embeddings abdecken.

Siehe [Speicher](/de/concepts/memory).

### Websuch-Tool

`web_search` kann abhängig vom ausgewählten Provider Nutzungskosten verursachen. Jeder Provider liest seinen Schlüssel zunächst aus einer Umgebungsvariablen und anschließend aus `plugins.entries.<id>.config.webSearch.apiKey`:

| Provider               | Umgebungsvariable(n)                                                                                                                                                  |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Brave Search           | `BRAVE_API_KEY`                                                                                                                                                    |
| DuckDuckGo             | ohne Schlüssel; inoffiziell, HTML-basiert, keine Abrechnung                                                                                                          |
| Exa                    | `EXA_API_KEY`                                                                                                                                                    |
| Firecrawl              | `FIRECRAWL_API_KEY`                                                                                                                                                    |
| Gemini (Google Search) | `GEMINI_API_KEY`                                                                                                                                                    |
| Grok (xAI)             | xAI-OAuth-Profil oder `XAI_API_KEY`                                                                                                                             |
| Kimi (Moonshot)        | `KIMI_API_KEY` oder `MOONSHOT_API_KEY`                                                                                                                           |
| MiniMax Search         | `MINIMAX_CODE_PLAN_KEY`, `MINIMAX_CODING_API_KEY`, `MINIMAX_OAUTH_TOKEN` oder `MINIMAX_API_KEY`                                                                                   |
| Ollama Web Search      | ohne Schlüssel für einen erreichbaren, lokal angemeldeten Host; die direkte Suche über `https://ollama.com` verwendet `OLLAMA_API_KEY`; authentifizierungsgeschützte Hosts verwenden die normale Bearer-Authentifizierung des Ollama-Providers |
| Parallel               | `PARALLEL_API_KEY`                                                                                                                                                    |
| Perplexity Search API  | `PERPLEXITY_API_KEY` oder `OPENROUTER_API_KEY`                                                                                                                           |
| SearXNG                | `SEARXNG_BASE_URL`; ohne Schlüssel/selbst gehostet, keine Abrechnung für gehostete Dienste                                                                            |
| Tavily                 | `TAVILY_API_KEY`                                                                                                                                                    |

Ältere `tools.web.search.*`-Konfigurationspfade werden weiterhin über einen Kompatibilitäts-Shim geladen, sind jedoch nicht mehr die empfohlene Oberfläche.

**Kostenloses Guthaben von Brave Search**: Jeder Tarif enthält ein monatlich erneuertes kostenloses Guthaben von $5. Der Search-Tarif kostet $5 pro 1,000 Anfragen, sodass das Guthaben 1,000 Anfragen pro Monat kostenlos abdeckt. Legen Sie im Brave-Dashboard ein Nutzungslimit fest, um unerwartete Kosten zu vermeiden.

Siehe [Web-Tools](/de/tools/web).

### Webabruf-Tool (Firecrawl)

`web_fetch` kann Firecrawl mit einem schlüssellosen Starterzugang aufrufen; fügen Sie `FIRECRAWL_API_KEY` (oder `plugins.entries.firecrawl.config.webFetch.apiKey`) hinzu, um höhere Limits zu erhalten. Wenn Firecrawl nicht konfiguriert ist, greift das Tool auf direkten Abruf sowie das mitgelieferte `web-readability`-Plugin zurück (keine kostenpflichtige API). Deaktivieren Sie `plugins.entries.web-readability.enabled`, um die lokale Readability-Extraktion zu überspringen.

Siehe [Web-Tools](/de/tools/web).

### Provider-Nutzungs-Snapshots (Status/Funktionsfähigkeit)

`openclaw status --usage` und `openclaw models status --json` rufen Provider-Nutzungsendpunkte auf, um Kontingentzeiträume oder den Authentifizierungsstatus anzuzeigen. Die Anzahl der Aufrufe ist gering, sie greifen jedoch dennoch auf Provider-APIs zu.

Siehe [Modell-CLI](/de/cli/models).

### Schutzmechanismus zur Zusammenfassung bei der Compaction

Der Compaction-Schutzmechanismus kann den Sitzungsverlauf mithilfe des aktuellen Modells zusammenfassen und ruft bei seiner Ausführung Provider-APIs auf.

Siehe [Sitzungsverwaltung und Compaction](/de/reference/session-management-compaction).

### Modellscan/-prüfung

`openclaw models scan` kann OpenRouter-Modelle prüfen und verwendet `OPENROUTER_API_KEY`, wenn die Prüfung aktiviert ist.

Siehe [Modell-CLI](/de/cli/models).

### Sprechen (Sprachausgabe)

Der Sprechmodus kann ElevenLabs aufrufen, wenn Folgendes konfiguriert ist: `ELEVENLABS_API_KEY` oder `talk.providers.elevenlabs.apiKey`.

Siehe [Sprechmodus](/de/nodes/talk).

### Skills (Drittanbieter-APIs)

Skills können `apiKey` in `skills.entries.<name>.apiKey` speichern. Wenn ein Skill diesen Schlüssel für eine externe API verwendet, richten sich die Kosten nach dem Provider des Skills.

Siehe [Skills](/de/tools/skills).

## Verwandte Themen

- [Token-Nutzung und Kosten](/de/reference/token-use)
- [Prompt-Caching](/de/reference/prompt-caching)
- [Nutzungsverfolgung](/de/concepts/usage-tracking)
