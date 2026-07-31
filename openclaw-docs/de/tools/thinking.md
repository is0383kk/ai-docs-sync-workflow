---
read_when:
    - Anpassen der Analyse-, Schnellmodus- oder Ausführlichkeitsdirektiven sowie ihrer Verarbeitung oder Standardwerte
summary: Direktivsyntax für /think, /fast, /verbose, /trace und die Sichtbarkeit von Schlussfolgerungen
title: Denkstufen
x-i18n:
    generated_at: "2026-07-26T18:11:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 80968ce58f642090ba0f807874e43eea1206cd31d919414c690b7537dc523658
    source_path: tools/thinking.md
    workflow: 16
---

## Funktionsweise

- Inline-Direktive in einem beliebigen eingehenden Nachrichtentext: `/t <level>`, `/think:<level>` oder `/thinking <level>`.
- Stufen (Aliasse): `off | minimal | low | medium | high | xhigh | adaptive | max | ultra`, ungefähr entsprechend der klassischen Anthropic-Zauberwort-Abstufung „think“ < „think hard“ < „think harder“ < „ultrathink“:
  - minimal ~ „think“
  - low ~ „think hard“
  - medium ~ „think harder“
  - high ~ „ultrathink“ (maximales Budget)
  - xhigh ~ „ultrathink+“ (GPT-5.2+- und Codex-Modelle sowie Anthropic Claude Opus 4.7+ Effort)
  - adaptive → vom Provider verwaltetes adaptives Denken (unterstützt für Claude 4.6 auf Anthropic/Bedrock, Anthropic Claude Opus 4.7+ und dynamisches Denken von Google Gemini)
  - max → maximales Reasoning des Providers (Anthropic Claude Opus 4.7+; Ollama ordnet dies seinem höchsten nativen `think`-Effort zu)
  - ultra → maximales Reasoning des Providers plus proaktive Sub-Agent-Orchestrierung, sofern das ausgewählte Modell bzw. die Runtime dies unterstützt
  - `x-high`, `x_high`, `extra-high`, `extra high` und `extra_high` werden `xhigh` zugeordnet.
  - `highest` wird `high` zugeordnet.
- Hinweise zu Providern:
  - Menüs und Auswahlfelder für das Denken werden durch das Provider-Profil bestimmt. Provider-Plugins deklarieren den genauen Stufensatz für das ausgewählte Modell, einschließlich Bezeichnungen wie dem binären `on`.
  - `adaptive`, `xhigh`, `max` und `ultra` werden nur für Provider-/Modell-/Runtime-Profile angeboten, die sie unterstützen. Typisierte Direktiven für nicht unterstützte Stufen werden unter Angabe der gültigen Optionen dieses Modells abgelehnt.
  - Vorhandene gespeicherte, nicht unterstützte Stufen werden anhand des Rangs im Provider-Profil neu zugeordnet. `adaptive` fällt bei nicht adaptiven Modellen auf `medium` zurück, während `xhigh` und `max` auf die höchste unterstützte Stufe außer „aus“ für das ausgewählte Modell zurückfallen.
  - Anthropic-Claude-4.6-Modelle verwenden standardmäßig `adaptive`, wenn keine explizite Denkstufe festgelegt ist.
  - Bei Anthropic Claude Opus 4.8 und Opus 4.7 bleibt das Denken deaktiviert, sofern Sie nicht ausdrücklich eine Denkstufe festlegen. Nachdem adaptives Denken aktiviert wurde, lautet der vom Provider vorgegebene Effort-Standardwert von Opus 4.8 `high`.
  - Anthropic Claude Opus 4.7+ ordnet `/think xhigh` adaptivem Denken plus `output_config.effort: "xhigh"` zu, da `/think` eine Denk-Direktive und `xhigh` die Effort-Einstellung von Opus ist.
  - Anthropic Claude Opus 4.7+ stellt außerdem `/think max` bereit; dies wird demselben vom Provider verwalteten Pfad für maximalen Effort zugeordnet.
  - Direkte DeepSeek-V4-Modelle stellen `/think xhigh|max` bereit; beide werden DeepSeek `reasoning_effort: "max"` zugeordnet, während niedrigere Stufen außer „aus“ `high` zugeordnet werden.
  - Über OpenRouter geroutete DeepSeek-V4-Modelle stellen `/think xhigh` bereit und senden von OpenRouter unterstützte `reasoning.effort`-Werte anstelle des nativen DeepSeek-Top-Level-Felds `reasoning_effort`. Niedrigere Stufen außer „aus“ werden `high` zugeordnet, und gespeicherte `max`-Überschreibungen fallen auf `xhigh` zurück.
  - Denkfähige Ollama-Modelle stellen `/think low|medium|high|max` bereit; `max` wird dem nativen `think: "high"` zugeordnet, da die native Ollama-API die Effort-Zeichenfolgen `low`, `medium` und `high` akzeptiert.
  - OpenAI-GPT-Modelle ordnen `/think` über die modellspezifische Effort-Unterstützung der Responses API zu. `/think off` sendet `reasoning.effort: "none"` nur, wenn das Zielmodell dies unterstützt; andernfalls lässt OpenClaw die deaktivierte Reasoning-Nutzlast weg, statt einen nicht unterstützten Wert zu senden.
  - GPT-5.6 Sol und Terra stellen natives `/think ultra` über die Codex-Runtime bereit. GPT-5.6 Luna stellt Stufen bis `max` bereit, da sein Codex-Katalog Ultra nicht ausweist.
  - Die eingebettete OpenClaw-Runtime stellt für GPT-5.6 Sol, Terra und Luna das logische `/think ultra` bereit. Sie sendet maximalen Provider-Effort und fügt laufbezogene Anweisungen zur proaktiven Sub-Agent-Orchestrierung hinzu.
  - Benutzerdefinierte OpenAI-kompatible Katalogeinträge können sich für `/think xhigh` entscheiden, indem `models.providers.<provider>.models[].compat.supportedReasoningEfforts` so festgelegt wird, dass es `"xhigh"` enthält. Dabei werden dieselben Kompatibilitätsmetadaten verwendet, die ausgehende OpenAI-Reasoning-Effort-Nutzlasten zuordnen, sodass Menüs, Sitzungsvalidierung, Agent-CLI und `llm-task` mit dem Transportverhalten übereinstimmen.
  - Veraltete konfigurierte OpenRouter-Hunter-Alpha-Referenzen überspringen die Proxy-Reasoning-Injektion, da diese eingestellte Route endgültigen Antworttext über Reasoning-Felder zurückgeben konnte.
  - Google Gemini ordnet `/think adaptive` dem vom Provider verwalteten dynamischen Denken von Gemini zu. Bei Gemini-3-Anfragen wird ein fester `thinkingLevel` weggelassen, während Gemini-2.5-Anfragen `thinkingBudget: -1` senden; feste Stufen werden weiterhin dem nächstliegenden Gemini-`thinkingLevel` oder Budget für diese Modellfamilie zugeordnet.
  - MiniMax M2.x (`minimax/MiniMax-M2*`) verwendet auf dem Anthropic-kompatiblen Streaming-Pfad standardmäßig `thinking: { type: "disabled" }`, sofern Sie das Denken nicht ausdrücklich in den Modell- oder Anfrageparametern festlegen. Dadurch werden durchgesickerte `reasoning_content`-Deltas aus dem nicht nativen Anthropic-Streamformat von M2.x vermieden. MiniMax-M3 (und M3.x) ist ausgenommen: M3 gibt korrekte Anthropic-Denkblöcke aus und liefert leeren Inhalt zurück, wenn das Denken deaktiviert ist. Daher belässt OpenClaw M3 auf dem vom Provider vorgesehenen Pfad für weggelassenes/adaptives Denken.
  - Z.AI (`zai/*`) ist für die meisten GLM-Modelle binär (`on`/`off`). GLM-5.2 ist die Ausnahme: Es stellt `/think off|low|high|max` bereit, ordnet `low` und `high` dem Z.AI-Wert `reasoning_effort: "high"` zu und ordnet `max` dem Wert `reasoning_effort: "max"` zu.
  - Moonshot API Kimi K3 (`moonshot/kimi-k3`) denkt immer mit `max`, sendet `reasoning_effort: "max"`, lässt das K2-Feld `thinking` sowie feste Sampling-Überschreibungen weg und behält die von K3 unterstützten Werkzeugauswahlen bei. Kimi Code K3 (`kimi/k3` und `kimi/k3[1m]`) stellt `/think off|max` bereit: „aus“ sendet `thinking.type: "disabled"`, während „max“ adaptives Denken mit maximalem Effort sendet. Aktuelle Kimi-Code-Referenzen enthalten außerdem `kimi/kimi-for-coding` und `kimi/kimi-for-coding-highspeed`. Kimi K2.7 Code (`moonshot/kimi-k2.7-code` und `moonshot/kimi-k2.7-code-highspeed`) denkt immer, stellt nur `on` bereit und lässt sowohl das ausgehende `thinking` als auch `reasoning_effort` weg. Andere `moonshot/*`-Modelle ordnen `/think off` dem Wert `thinking: { type: "disabled" }` und jede Stufe außer `off` dem Wert `thinking: { type: "enabled" }` zu. Wenn K2-Denken aktiviert ist, akzeptiert Moonshot für `tool_choice` nur `auto|none`; OpenClaw normalisiert inkompatible Werte zu `auto`.

## Auflösungsreihenfolge

1. Inline-Direktive in der Nachricht (gilt nur für diese Nachricht).
2. Sitzungsüberschreibung (durch Senden einer Nachricht festgelegt, die nur eine Direktive enthält).
3. Agent-spezifischer Standardwert (`agents.entries.*.thinkingDefault` in der Konfiguration).
4. Globaler Standardwert (`agents.defaults.thinkingDefault` in der Konfiguration).
5. Fallback: vom Provider deklarierter Standardwert, sofern verfügbar; andernfalls werden Reasoning-fähige Modelle auf `medium` oder die nächstliegende unterstützte Stufe ungleich `off` für dieses Modell aufgelöst, und Modelle ohne Reasoning-Fähigkeit bleiben auf `off`.

## Sitzungsstandardwert festlegen

- Senden Sie eine Nachricht, die **nur** die Direktive enthält (Leerzeichen sind zulässig), beispielsweise `/think:medium` oder `/t high`.
- Dies bleibt für die aktuelle Sitzung bestehen (standardmäßig pro Absender). Verwenden Sie `/think default`, um die Sitzungsüberschreibung zu löschen und den konfigurierten bzw. vom Provider vorgegebenen Standardwert zu übernehmen; zu den Aliasen gehören `inherit`, `clear`, `reset` und `unpin`.
- `/think off` speichert eine explizite „Aus“-Überschreibung. Dadurch wird das Denken deaktiviert, bis Sie die Sitzungsüberschreibung ändern oder löschen.
- Eine Bestätigungsantwort wird gesendet (`Thinking level set to high.` / `Thinking disabled.`). Ist die Stufe ungültig (z. B. `/thinking big`), wird der Befehl mit einem Hinweis abgelehnt und der Sitzungsstatus bleibt unverändert.
- Senden Sie `/think` (oder `/think:`) ohne Argument, um die aktuelle Denkstufe anzuzeigen.

## Anwendung durch den Agenten

- **Eingebettetes OpenClaw**: Die aufgelöste Stufe wird an die prozessinterne OpenClaw-Agent-Runtime übergeben.
- **Claude-CLI-Backend**: Konkrete Stufen außer „aus“ werden bei Verwendung von `claude-cli` als `--effort` an Claude Code übergeben; `adaptive` entfernt konfigurierte Effort-Flags und überlässt den effektiven Effort der Umgebung, den Einstellungen und den Modellstandardwerten von Claude Code. Siehe [CLI-Backends](/de/gateway/cli-backends).

## Schnellmodus (/fast)

- Stufen: `auto|on|off|default`.
- Eine Nachricht, die nur eine Direktive enthält, schaltet eine sitzungsbezogene Schnellmodus-Überschreibung um und antwortet mit `Fast mode set to auto.`, `Fast mode enabled.` oder `Fast mode disabled.`. Verwenden Sie `/fast default`, um die Sitzungsüberschreibung zu löschen und den konfigurierten Standardwert zu übernehmen; zu den Aliasen gehören `inherit`, `clear`, `reset` und `unpin`.
- Senden Sie `/fast` (oder `/fast status`) ohne Modus, um den aktuellen effektiven Schnellmodusstatus anzuzeigen.
- OpenClaw löst den Schnellmodus in dieser Reihenfolge auf:
  1. Inline- bzw. Nur-Direktive-Überschreibung `/fast auto|on|off` (`/fast default` löscht diese Ebene)
  2. Sitzungsüberschreibung
  3. Agent-spezifischer Standardwert (`agents.entries.*.fastModeDefault`)
  4. Modellspezifische Konfiguration: `agents.defaults.models["<provider>/<model>"].params.fastMode`
  5. Fallback: `off`
- `auto` behält für die Sitzung bzw. Konfiguration den automatischen Modus bei, löst jedoch jeden neuen Modellaufruf unabhängig auf. Bei Aufrufen, die vor dem automatischen Grenzwert beginnen, ist der Schnellmodus aktiviert; spätere Wiederholungs-, Fallback-, Werkzeugergebnis- oder Fortsetzungsaufrufe beginnen mit deaktiviertem Schnellmodus. Der Grenzwert beträgt standardmäßig 60 Sekunden; legen Sie `agents.defaults.models["<provider>/<model>"].params.fastAutoOnSeconds` für das aktive Modell fest, um ihn zu ändern.
- Für `openai/*` wird der Schnellmodus der priorisierten Verarbeitung von OpenAI zugeordnet, indem bei unterstützten Responses-Anfragen `service_tier=priority` gesendet wird.
- Für Codex-gestützte `openai/*`- bzw. `openai-codex/*`-Modelle sendet der Schnellmodus dasselbe `service_tier=priority`-Flag bei Codex-Responses. Native Codex-App-Server-Turns erhalten die Stufe nur bei `turn/start` oder beim Starten/Fortsetzen eines Threads. Daher kann `auto` einen bereits laufenden App-Server-Turn nicht neu einstufen; die Einstellung gilt für den nächsten von OpenClaw gestarteten Modell-Turn.
- Bei direkten öffentlichen `anthropic/*`-Anfragen, einschließlich OAuth-authentifiziertem Datenverkehr an `api.anthropic.com`, wird der Schnellmodus Anthropic-Servicestufen zugeordnet: `/fast on` setzt `service_tier=auto`, `/fast off` setzt `service_tier=standard_only`.
- Für `minimax/*` auf dem Anthropic-kompatiblen Pfad schreibt `/fast on` (oder `params.fastMode: true`) `MiniMax-M2.7` in `MiniMax-M2.7-highspeed` um.
- Explizite Anthropic-Modellparameter `serviceTier` / `service_tier` überschreiben den Schnellmodus-Standardwert, wenn beide gesetzt sind. OpenClaw überspringt die Injektion der Anthropic-Servicestufe weiterhin bei nicht zu Anthropic gehörenden Proxy-Basis-URLs.
- `/status` zeigt `Fast` an, wenn der Schnellmodus aktiviert ist, und `Fast:auto`, wenn als Modus „automatisch“ konfiguriert ist.

## Ausführliche Direktiven (/verbose oder /v)

- Stufen: `on` (minimal) | `full` | `off` (Standard).
- Eine Nachricht, die nur eine Direktive enthält, schaltet die ausführliche Ausgabe der Sitzung um und antwortet mit `Verbose logging enabled.` / `Verbose logging disabled.`; ungültige Stufen geben einen Hinweis zurück, ohne den Zustand zu ändern.
- `/verbose off` speichert eine explizite Sitzungsüberschreibung; löschen Sie sie über die Sitzungs-UI, indem Sie `inherit` auswählen.
- Autorisierte Absender externer Kanäle können die Überschreibung für die ausführliche Sitzungsausgabe dauerhaft speichern. Interne Gateway-/Webchat-Clients benötigen `operator.admin`, um sie dauerhaft zu speichern.
- Eine Inline-Direktive wirkt sich nur auf diese Nachricht aus; andernfalls gelten die Sitzungs-/globalen Standardwerte.
- Senden Sie `/verbose` (oder `/verbose:`) ohne Argument, um die aktuelle Ausführlichkeitsstufe anzuzeigen.
- Wenn die ausführliche Ausgabe aktiviert ist, senden Agents, die strukturierte Werkzeugergebnisse ausgeben, jeden Werkzeugaufruf als eigene Nachricht zurück, die nur Metadaten enthält und, sofern verfügbar, mit `<emoji> <tool-name>: <arg>` beginnt. Diese Werkzeugzusammenfassungen werden gesendet, sobald das jeweilige Werkzeug startet (in separaten Sprechblasen), nicht als Streaming-Deltas.
- Zusammenfassungen von Werkzeugfehlern bleiben im normalen Modus sichtbar, aber Suffixe mit rohen Fehlerdetails werden ausgeblendet, sofern die Ausführlichkeitsstufe nicht `full` ist.
- Wenn die Ausführlichkeitsstufe `full` ist, werden Werkzeugausgaben nach Abschluss ebenfalls weitergeleitet (in einer separaten Sprechblase und auf eine sichere Länge gekürzt). Wenn Sie `/verbose on|full|off` während eines laufenden Durchlaufs umschalten, berücksichtigen nachfolgende Werkzeug-Sprechblasen die neue Einstellung.
- `agents.defaults.toolProgressDetail` steuert die Form der `/verbose`-Werkzeugzusammenfassungen und der Werkzeugzeilen in Fortschrittsentwürfen. Verwenden Sie `"explain"` (Standard) für kompakte, menschenlesbare Bezeichnungen wie `🛠️ Exec: checking JS syntax`; verwenden Sie `"raw"`, wenn zu Debugging-Zwecken auch der rohe Befehl bzw. die Details angehängt werden sollen. Die Agent-spezifische Einstellung `agents.entries.*.toolProgressDetail` überschreibt den Standardwert.
  - `explain`: `🛠️ Exec: check JS syntax for /tmp/app.js`
  - `raw`: `🛠️ Exec: check JS syntax for /tmp/app.js, node --check /tmp/app.js`

## Plugin-Trace-Direktiven (/trace)

- Stufen: `on` | `off` (Standard).
- Eine Nachricht, die nur eine Direktive enthält, schaltet die Plugin-Trace-Ausgabe der Sitzung um und antwortet mit `Plugin trace enabled.` / `Plugin trace disabled.`.
- Eine Inline-Direktive wirkt sich nur auf diese Nachricht aus; andernfalls gelten die Sitzungs-/globalen Standardwerte.
- Senden Sie `/trace` (oder `/trace:`) ohne Argument, um die aktuelle Trace-Stufe anzuzeigen.
- `/trace` ist enger gefasst als `/verbose`: Es legt nur Plugin-eigene Trace-/Debug-Zeilen offen, beispielsweise Debug-Zusammenfassungen von Active Memory.
- Trace-Zeilen können in `/status` und als nachfolgende Diagnosemeldung nach der normalen Assistant-Antwort erscheinen.

## Sichtbarkeit der Schlussfolgerungen (/reasoning)

- Stufen: `on|off|stream`.
- Eine Nachricht, die nur eine Direktive enthält, schaltet um, ob Denkblöcke in Antworten angezeigt werden.
- Wenn diese Option aktiviert ist, werden Schlussfolgerungen als **separate Nachricht** gesendet, der `Thinking` vorangestellt ist.
- `stream`: Streamt Schlussfolgerungen während der Generierung der Antwort, wenn der aktive Kanal Vorschauen von Schlussfolgerungen unterstützt, und sendet anschließend die endgültige Antwort ohne Schlussfolgerungen.
- Alias: `/reason`.
- Senden Sie `/reasoning` (oder `/reasoning:`) ohne Argument, um die aktuelle Schlussfolgerungsstufe anzuzeigen.
- Auflösungsreihenfolge: Inline-Direktive, dann Sitzungsüberschreibung, dann Agent-spezifischer Standardwert (`agents.entries.*.reasoningDefault`), dann globaler Standardwert (`agents.defaults.reasoningDefault`), dann Rückfallwert (`off`).

Fehlerhafte Schlussfolgerungs-Tags lokaler Modelle werden konservativ behandelt. Geschlossene `<think>...</think>`-Blöcke bleiben in normalen Antworten ausgeblendet, und nicht geschlossene Schlussfolgerungen nach bereits sichtbarem Text werden ebenfalls ausgeblendet. Wenn eine Antwort vollständig von einem einzelnen nicht geschlossenen öffnenden Tag umschlossen ist und andernfalls als leerer Text zugestellt würde, entfernt OpenClaw das fehlerhafte öffnende Tag und stellt den verbleibenden Text zu.

## Verwandte Themen

- Die Dokumentation zum erweiterten Modus finden Sie unter [Erweiterter Modus](/de/tools/elevated).

## Heartbeats

- Der Text der Heartbeat-Prüfung ist der konfigurierte Heartbeat-Prompt (Standard: `Follow the heartbeat monitor scratch context when provided. Recurring tasks are cron jobs; create or change their schedules with cron tools or the openclaw cron CLI, not heartbeat scratch. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.`). Inline-Direktiven in einer Heartbeat-Nachricht werden wie üblich angewendet (vermeiden Sie jedoch, Sitzungsstandardwerte durch Heartbeats zu ändern).
- Bei der Heartbeat-Zustellung wird standardmäßig nur die endgültige Nutzlast gesendet. Um zusätzlich die separate Nachricht `Thinking` zu senden (sofern verfügbar), legen Sie `agents.defaults.heartbeat.includeReasoning: true` oder Agent-spezifisch `agents.entries.*.heartbeat.includeReasoning: true` fest.

## Webchat-UI

- Beim Laden der Seite spiegelt die Denkauswahl des Webchats die gespeicherte Stufe der Sitzung aus dem eingehenden Sitzungsspeicher bzw. der Konfiguration wider.
- Die Auswahl einer anderen Stufe schreibt die Sitzungsüberschreibung sofort über `sessions.patch`; sie wartet nicht auf das nächste Senden und ist keine einmalige `thinkingOnce`-Überschreibung.
- Wenn beim Senden Änderungen an der Modell-, Schlussfolgerungs- oder Geschwindigkeitsauswahl noch angewendet werden, wird auf alle ausstehenden Auswahl-Patches gewartet; schlägt eine Änderung fehl, bleibt die Nachricht zur Überprüfung ungesendet.
- Die erste Option ist immer die Auswahl zum Löschen der Überschreibung. Sie zeigt `Inherited: <resolved level>` an, einschließlich `Inherited: Off`, wenn das übernommene Denken deaktiviert ist.
- Explizite Auswahlen verwenden ihre direkten Stufenbezeichnungen und behalten vorhandene Provider-Bezeichnungen bei (beispielsweise `Maximum` für eine mit einem Provider bezeichnete Option `max`).
- Die Auswahl verwendet `thinkingLevels`, das von der Gateway-Sitzungszeile bzw. den Standardwerten zurückgegeben wird, wobei `thinkingOptions` als Liste veralteter Bezeichnungen beibehalten wird. Die Browser-UI führt keine eigene Liste regulärer Ausdrücke für Provider; Plugins besitzen die modellspezifischen Stufensätze.
- `/think:<level>` funktioniert weiterhin und aktualisiert dieselbe gespeicherte Sitzungsstufe, sodass Chat-Direktiven und die Auswahl synchron bleiben.

## Provider-Profile

- Provider-Plugins können `resolveThinkingProfile(ctx)` bereitstellen, um die unterstützten Stufen und den Standardwert des Modells zu definieren.
- Provider-Plugins, die Claude-Modelle weiterleiten, sollten `resolveClaudeThinkingProfile(modelId)` aus `openclaw/plugin-sdk/provider-model-shared` wiederverwenden, damit direkte Anthropic-Kataloge und Proxy-Kataloge übereinstimmen.
- Jede Profilstufe verfügt über einen gespeicherten kanonischen `id` (`off`, `minimal`, `low`, `medium`, `high`, `xhigh`, `adaptive`, `max` oder `ultra`) und kann einen Anzeige-`label` enthalten. Binäre Provider verwenden `{ id: "low", label: "on" }`.
- Profil-Hooks erhalten, sofern verfügbar, zusammengeführte Kataloginformationen, darunter `reasoning`, `compat.thinkingFormat` und `compat.supportedReasoningEfforts`. Verwenden Sie diese Informationen, um binäre oder benutzerdefinierte Profile nur dann bereitzustellen, wenn der konfigurierte Anfragevertrag die entsprechende Nutzlast unterstützt.
- Werkzeug-Plugins, die eine explizite Denküberschreibung validieren müssen, sollten `api.runtime.agent.resolveThinkingPolicy({ provider, model, agentRuntime })` zusammen mit `api.runtime.agent.normalizeThinkingLevel(...)` verwenden; sie sollten keine eigenen Provider-/Modell-Stufenlisten führen. Übergeben Sie `agentRuntime`, wenn das Werkzeug den Ausführungspfad besitzt, beispielsweise bei einem stets eingebetteten Durchlauf.
- Werkzeug-Plugins mit Zugriff auf konfigurierte benutzerdefinierte Modellmetadaten können `catalog` an `resolveThinkingPolicy` übergeben, damit `compat.supportedReasoningEfforts`-Aktivierungen bei der Plugin-seitigen Validierung berücksichtigt werden.
- Veröffentlichte veraltete Hooks (`supportsXHighThinking`, `isBinaryThinking` und `resolveDefaultThinkingLevel`) bleiben als Kompatibilitätsadapter erhalten, neue benutzerdefinierte Stufensätze sollten jedoch `resolveThinkingProfile` verwenden.
- Gateway-Zeilen bzw. -Standardwerte stellen `thinkingLevels`, `thinkingOptions` und `thinkingDefault` bereit, damit ACP-/Chat-Clients dieselben Profil-IDs und -Bezeichnungen darstellen, die von der Laufzeitvalidierung verwendet werden.
