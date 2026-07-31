---
read_when:
    - Sie debuggen Ablehnungen von Provider-Anfragen, die mit der Struktur des Transkripts zusammenhängen.
    - Sie ändern die Logik zur Bereinigung von Transkripten oder zur Reparatur von Tool-Aufrufen
    - Sie untersuchen Abweichungen bei Tool-Call-IDs zwischen Providern
summary: 'Referenz: providerspezifische Regeln zur Bereinigung und Reparatur von Transkripten'
title: Transkripthygiene
x-i18n:
    generated_at: "2026-07-26T18:04:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 33d978772062cb2a81eb358bb5c62bd1261b433ffdc8acdbaa6679b121fbbf62
    source_path: reference/transcript-hygiene.md
    workflow: 16
---

OpenClaw wendet vor einem Lauf **Provider-spezifische Korrekturen** auf Transkripte an
(beim Erstellen des Modellkontexts). Die meisten davon sind Anpassungen **im Arbeitsspeicher**, die dazu dienen,
strenge Provider-Anforderungen zu erfüllen. Ein separater Reparaturdurchlauf für Sitzungsdateien kann
außerdem gespeicherte JSONL-Daten umschreiben, bevor die Sitzung geladen wird, jedoch nur bei
fehlerhaften Zeilen oder persistierten Gesprächsbeiträgen, die keine gültigen dauerhaften Datensätze sind.
Ausgelieferte Assistentenantworten bleiben auf dem Datenträger erhalten; das Provider-spezifische
Entfernen von Assistenten-Prefills erfolgt nur beim Erstellen ausgehender
Payloads.

Wenn eine Reparatur erfolgt, wird die ursprüngliche Datei vor dem atomaren Ersetzen in eine temporäre
`*.bak-<pid>-<ts>`-Geschwisterdatei geschrieben und nach erfolgreichem
Ersetzen entfernt. Die Sicherung bleibt nur erhalten, wenn die Bereinigung selbst fehlschlägt;
in diesem Fall wird der Pfad zurückgemeldet.

Der Umfang umfasst:

- Nur zur Laufzeit verwendeter Prompt-Kontext bleibt außerhalb benutzersichtbarer Transkriptbeiträge
- Bereinigung von Tool-Aufruf-IDs
- Validierung der Eingaben von Tool-Aufrufen
- Reparatur der Zuordnung von Tool-Ergebnissen
- Validierung/Reihenfolge von Gesprächsbeiträgen
- Bereinigung von Gedankensignaturen
- Bereinigung von Denkprozesssignaturen
- Bereinigung von Bild-Payloads
- Bereinigung leerer Textblöcke vor der Provider-Wiedergabe
- Bereinigung unvollständiger, ausschließlich aus Schlussfolgerungen bestehender Längenlimit-Beiträge vor der Provider-Wiedergabe
- Kennzeichnung der Herkunft von Benutzereingaben (für zwischen Sitzungen weitergeleitete Prompts)
- Reparatur leerer Assistenten-Fehlerbeiträge für die Bedrock-Converse-Wiedergabe

Einzelheiten zur Transkriptspeicherung finden Sie unter
[Ausführliche Erläuterung zur Sitzungsverwaltung](/de/reference/session-management-compaction).

---

## Globale Regel: Laufzeitkontext ist kein Benutzertranskript

Laufzeit-/Systemkontext kann für einen Gesprächsbeitrag zum Modell-Prompt hinzugefügt werden, ist jedoch
kein von Endbenutzern verfasster Inhalt. OpenClaw führt einen separaten, für das Transkript bestimmten
Prompt-Textkörper für Gateway-Antworten, Folgeanfragen in der Warteschlange, ACP, CLI und eingebettete
OpenClaw-Läufe. Gespeicherte sichtbare Benutzerbeiträge verwenden diesen Transkript-Textkörper anstelle
des um Laufzeitinformationen erweiterten Prompts.

Bei älteren Sitzungen, in denen Laufzeit-Wrapper bereits persistiert wurden, wenden Gateway-Verlaufsoberflächen
eine Anzeigeprojektion an, bevor sie Nachrichten an WebChat-,
TUI-, REST- oder SSE-Clients zurückgeben.

---

## Ausführungsort

Die gesamte Transkripthygiene ist im eingebetteten Runner zentralisiert:

- Richtlinienauswahl: `src/agents/transcript-policy.ts`
  (`resolveTranscriptPolicy`, anhand von `provider`, `modelApi` und `modelId`)
- Anwendung der Bereinigung/Reparatur: `sanitizeSessionHistory` in
  `src/agents/embedded-agent-runner/replay-history.ts`

Unabhängig von der Transkripthygiene werden Sitzungsdateien bei Bedarf
vor dem Laden repariert:

- `repairSessionFileIfNeeded` in `src/agents/session-file-repair.ts`
- Aufgerufen von `src/agents/embedded-agent-runner/run/attempt.ts` und
  `src/agents/embedded-agent-runner/compact.ts`

---

## Globale Regel: Bildbereinigung

Bild-Payloads werden immer bereinigt, um eine Provider-seitige Ablehnung aufgrund von
Größenbeschränkungen zu verhindern (Verkleinerung/Neukomprimierung übergroßer Base64-Bilder). Dies hilft außerdem,
den durch Bilder verursachten Token-Druck bei bildverarbeitungsfähigen Modellen zu kontrollieren: niedrigere maximale
Abmessungen reduzieren den Token-Verbrauch, höhere Abmessungen bewahren Details.

Implementierung:

- `sanitizeSessionMessagesImages` in
  `src/agents/embedded-agent-helpers/images.ts`
- `sanitizeContentBlocksImages` in `src/agents/tool-images.ts`
- Die maximale Bildseitenlänge ist über `agents.defaults.imageMaxDimensionPx` konfigurierbar
  (Standard: `1200`)
- Leere Textblöcke werden entfernt, während dieser Durchlauf den Wiedergabeinhalt verarbeitet.
  Assistentenbeiträge, die dadurch leer werden, werden aus der Wiedergabekopie entfernt; Benutzer-
  und Tool-Ergebnis-Beiträge, die dadurch leer werden, erhalten einen nicht leeren
  Platzhalter für ausgelassenen Inhalt.

---

## Globale Regel: fehlerhafte Tool-Aufrufe

Assistenten-Tool-Aufrufblöcke, denen sowohl `input` als auch `arguments` fehlen, werden entfernt,
bevor der Modellkontext erstellt wird. Dies verhindert Provider-Ablehnungen aufgrund
teilweise persistierter Tool-Aufrufe (beispielsweise nach einem Fehler wegen einer Ratenbegrenzung).

Implementierung:

- `sanitizeToolCallInputs` in `src/agents/session-transcript-repair.ts`
- Angewendet in `sanitizeSessionHistory`
  (`src/agents/embedded-agent-runner/replay-history.ts`)

---

## Globale Regel: Zuordnung von Tool-Ergebnissen

Tool-Ergebnisse werden innerhalb jedes Assistentenbeitrags den jeweiligen Vorkommen von Tool-Aufrufen zugeordnet, bevor
Provider-spezifische Aufruf-IDs umgeschrieben werden. Vom Provider erzeugte IDs können sich in späteren
Gesprächsbeiträgen wiederholen, sodass ein Ergebnis neben einem wiederholten Aufruf diesem Vorkommen zugeordnet bleibt. Ein verschobenes
Ergebnis wird nur dann verlegt, wenn genau ein noch nicht zugeordnetes Vorkommen dafür infrage kommt; mehrdeutige
zusätzliche Ergebnisse werden entfernt und fehlende Vorkommen erhalten synthetische Fehlerergebnisse.

Implementierung: `sanitizeToolUseResultPairing` in
`src/agents/session-transcript-repair.ts`

---

## Globale Regel: unvollständige oder stille Beiträge, die ausschließlich aus Schlussfolgerungen bestehen

Assistentenbeiträge werden aus der Wiedergabekopie im Arbeitsspeicher ausgelassen, wenn sie
nach einem der folgenden Ereignisse nur Denkprozess- oder redigierte Denkprozessinhalte enthalten:

- Das Provider-Ausgabelimit beendet den Gesprächsbeitrag mit einem unvollständigen Schlussfolgerungsstatus.
- Die Bereinigung stiller Antworten entfernt den einzigen sichtbaren `NO_REPLY`-Text des Gesprächsbeitrags.

Die Bereinigung stiller Antworten verhindert, dass verborgene Schlussfolgerungen mit einem späteren
Assistentenbeitrag zur Tool-Nutzung zusammengeführt werden, wenn strenge Provider die Unterhaltung neu aufbauen.

Leere Längenlimit-Beiträge bleiben ebenso unverändert wie Längenlimit-Beiträge mit sichtbarem Text,
Tool-Aufrufen oder unbekannten Inhaltsblöcken. Beiträge stiller Antworten mit Tool-Aufrufen oder
unbekannten Inhaltsblöcken bleiben ebenfalls unverändert. Gespeicherte Transkripte werden nicht
umgeschrieben.

Implementierung: `normalizeAssistantReplayContent` in
`src/agents/embedded-agent-runner/replay-history.ts`

---

## Globale Regel: Herkunft sitzungsübergreifender Eingaben

Wenn ein Agent über `sessions_send` einen Prompt an eine andere Sitzung sendet
(einschließlich Antwort-/Ankündigungsschritten zwischen Agenten), persistiert OpenClaw den
erstellten Benutzerbeitrag mit `message.provenance.kind = "inter_session"`.

OpenClaw stellt dem weitergeleiteten Prompt-Text außerdem im selben Gesprächsbeitrag eine
`[Inter-session message] ... isUser=false`-Markierung voran, damit der aktive Modellaufruf
Ausgaben fremder Sitzungen von externen Endbenutzeranweisungen unterscheiden kann. Diese
Markierung enthält, sofern verfügbar, die Quellsitzung, den Kanal und das Tool. Das
Transkript verwendet aus Gründen der Provider-Kompatibilität weiterhin `role: "user"`, aber sowohl der
sichtbare Text als auch die Herkunftsmetadaten kennzeichnen den Gesprächsbeitrag als sitzungsübergreifende
Daten.

Beim Neuaufbau des Kontexts wendet OpenClaw dieselbe Markierung auf ältere persistierte
sitzungsübergreifende Benutzerbeiträge an, die nur Herkunftsmetadaten enthalten.

---

## Provider-Matrix (aktuelles Verhalten)

**OpenAI / OpenAI Codex**

- Nur Bildbereinigung.
- Verwaiste Schlussfolgerungssignaturen (eigenständige Schlussfolgerungselemente ohne
  nachfolgenden Inhaltsblock) werden bei OpenAI-Responses-/Codex-Transkripten entfernt; außerdem werden
  wiedergabefähige OpenAI-Schlussfolgerungen nach einem Wechsel der Modellroute entfernt.
- Die Payloads wiedergabefähiger OpenAI-Responses-Schlussfolgerungselemente bleiben erhalten, einschließlich
  verschlüsselter Elemente mit leerer Zusammenfassung, damit die manuelle/WebSocket-Wiedergabe den erforderlichen
  `rs_*`-Status den Assistentenausgabeelementen zugeordnet hält.
- Native ChatGPT Codex Responses stellt die Codex-Protokollparität her, indem
  vorherige Responses-Schlussfolgerungs-/Nachrichten-/Funktions-Payloads ohne vorherige Element-
  IDs wiedergegeben werden, während das Sitzungs-`prompt_cache_key` erhalten bleibt.
- Die Wiedergabe der OpenAI-Responses-Familie bewahrt kanonische `call_*|fc_*`-
  Schlussfolgerungspaare desselben Modells, normalisiert jedoch fehlerhafte oder
  überlange `call_id`-/Funktionsaufruf-Element-IDs deterministisch vor der Umwandlung in pi-ai-Payloads.
- Die Reparatur der Zuordnung von Tool-Ergebnissen kann tatsächlich übereinstimmende Ausgaben verschieben und
  Codex-artige `aborted`-Ausgaben für fehlende Tool-Aufrufe synthetisieren.
- Keine Validierung oder Neuordnung von Gesprächsbeiträgen; kein Entfernen von Gedankensignaturen.

**OpenAI-kompatible Chat Completions**

- Historische Assistenten-Denkprozess-/Schlussfolgerungsblöcke werden vor der Wiedergabe entfernt,
  damit lokale und Proxy-artige OpenAI-kompatible Server keine
  Schlussfolgerungsfelder früherer Gesprächsbeiträge wie `reasoning` oder `reasoning_content` erhalten.
- Aktuelle Fortsetzungen von Tool-Aufrufen im selben Gesprächsbeitrag behalten den Assistenten-Schlussfolgerungsblock
  am Tool-Aufruf, bis das Tool-Ergebnis wiedergegeben wurde.
- Benutzerdefinierte/selbst gehostete Modelleinträge mit `reasoning: true` bewahren wiedergegebene
  Schlussfolgerungsmetadaten.
- Provider-eigene Ausnahmen können dies deaktivieren, wenn ihr Protokoll
  wiedergegebene Schlussfolgerungsmetadaten erfordert.

**Google (Generative AI / Gemini CLI / Antigravity)**

- Bereinigung von Tool-Aufruf-IDs: strikt alphanumerisch.
- Reparatur der Zuordnung von Tool-Ergebnissen und synthetische Tool-Ergebnisse.
- Validierung von Gesprächsbeiträgen (Gemini-artiger Wechsel der Gesprächsbeiträge).
- Korrektur der Google-Reihenfolge von Gesprächsbeiträgen (ein kurzer Benutzer-Startbeitrag wird vorangestellt, wenn der Verlauf
  mit dem Assistenten beginnt).
- Antigravity Claude: Denkprozesssignaturen werden normalisiert; unsignierte Denkprozessblöcke
  werden entfernt.

**Anthropic / Minimax (Anthropic-kompatibel)**

- Reparatur der Zuordnung von Tool-Ergebnissen und synthetische Tool-Ergebnisse.
- Validierung von Gesprächsbeiträgen (aufeinanderfolgende Benutzerbeiträge werden zusammengeführt, um einen strikten
  Wechsel einzuhalten).
- Nachgestellte Assistenten-Prefill-Beiträge werden aus ausgehenden Anthropic-
  Messages-Payloads entfernt, wenn der Denkprozess aktiviert ist, einschließlich Routen über das Cloudflare AI
  Gateway.
- Assistenten-Denkprozesssignaturen vor der Compaction werden vor der Provider-
  Wiedergabe entfernt, wenn eine Sitzung komprimiert wurde. Denkprozesssignaturen sind
  zum Erzeugungszeitpunkt kryptografisch an das Präfix der Unterhaltung gebunden;
  nach der Compaction ändert sich das Präfix (zusammengefasster Inhalt ersetzt den
  ursprünglichen), sodass die Wiedergabe der ursprünglichen Signaturen dazu führt, dass Anthropic
  die Anfrage mit "Invalid signature in thinking block" ablehnt. Der
  Denkprozesstext bleibt als unsignierter Block erhalten und wird anschließend von der
  nachstehenden Regel verarbeitet.
- Denkprozessblöcke mit fehlenden, leeren oder ausschließlich aus Leerzeichen bestehenden Wiedergabesignaturen werden
  vor der Provider-Umwandlung entfernt. Wenn dadurch ein Assistentenbeitrag leer wird,
  bewahrt OpenClaw die Form des Gesprächsbeitrags mit einem nicht leeren Text für ausgelassene Schlussfolgerungen.
- Ältere Assistentenbeiträge, die ausschließlich aus Denkprozessen bestehen und entfernt werden müssen, werden
  durch einen nicht leeren Text für ausgelassene Schlussfolgerungen ersetzt, damit Provider-Adapter
  den Wiedergabebeitrag nicht verwerfen.

**Amazon Bedrock (Converse API)**

- Leere Assistentenbeiträge mit Stream-Fehlern werden vor der Wiedergabe mit einem nicht leeren Ersatz-
  Textblock repariert. Bedrock Converse lehnt Assistentennachrichten
  mit `content: []` ab, daher werden persistierte Assistentenbeiträge mit `stopReason:
"error"` und leerem Inhalt vor dem Laden auch auf dem Datenträger repariert.
- Assistentenbeiträge mit Stream-Fehlern, die nur leere Textblöcke enthalten, werden aus
  der Wiedergabekopie im Arbeitsspeicher entfernt, statt einen ungültigen leeren Block wiederzugeben.
- Assistenten-Denkprozesssignaturen vor der Compaction werden vor der Converse-
  Wiedergabe entfernt, wenn eine Sitzung komprimiert wurde, und zwar aus demselben Grund wie
  bei Anthropic weiter oben.
- Claude-Denkprozessblöcke mit fehlenden, leeren oder ausschließlich aus Leerzeichen bestehenden Wiedergabesignaturen
  werden vor der Converse-Wiedergabe entfernt. Wenn dadurch ein Assistentenbeitrag leer wird,
  bewahrt OpenClaw die Form des Gesprächsbeitrags mit einem nicht leeren Text für ausgelassene Schlussfolgerungen.
- Ältere Assistentenbeiträge, die ausschließlich aus Denkprozessen bestehen und entfernt werden müssen, werden
  durch einen nicht leeren Text für ausgelassene Schlussfolgerungen ersetzt, damit die Converse-Wiedergabe
  die strikte Form der Gesprächsbeiträge beibehält.
- Die Wiedergabe filtert OpenClaw-Beiträge des Assistenten, die durch Auslieferungsspiegelung oder das Gateway
  eingefügt wurden.
- Die Bildbereinigung erfolgt gemäß der globalen Regel.

**Mistral (einschließlich Erkennung anhand der Modell-ID)**

- Bereinigung von Tool-Aufruf-IDs: strict9 (alphanumerisch, Länge 9).

**OpenRouter Gemini**

- Bereinigung von Gedankensignaturen: Nicht-Base64-Werte von `thought_signature` entfernen
  (Base64 beibehalten).

**OpenRouter Anthropic**

- Nachgestellte Assistenten-Prefill-Beiträge werden aus verifizierten OpenRouter-
  Payloads OpenAI-kompatibler Anthropic-Modelle entfernt, wenn Schlussfolgerungen aktiviert sind;
  dies entspricht dem Wiedergabeverhalten von direktem Anthropic und Cloudflare Anthropic.

**Alles andere**

- Nur Bildbereinigung.

---

## Historisches Verhalten (vor 2026.1.22)

Vor der Version 2026.1.22 wendete OpenClaw mehrere Ebenen der Transkript-
hygiene an:

- Eine **transcript-sanitize-Erweiterung** wurde bei jedem Kontextaufbau ausgeführt und konnte:
  - Die Zuordnung von Tool-Aufrufen und -Ergebnissen reparieren.
  - Tool-Aufruf-IDs bereinigen (einschließlich eines nicht strikten Modus, der
    `_`/`-` beibehielt).
- Der Runner führte außerdem eine providerspezifische Bereinigung durch, wodurch
  Arbeit doppelt ausgeführt wurde.
- Weitere Änderungen erfolgten außerhalb der Provider-Richtlinie, darunter
  das Entfernen von `<final>`-Tags aus Assistententext vor der Persistierung, das Verwerfen
  leerer fehlerhafter Assistentenbeiträge und das Kürzen von Assistenteninhalten nach Tool-
  Aufrufen.

Diese Komplexität verursachte providerübergreifende Regressionen (insbesondere bei der
Zuordnung von `openai-responses` und `call_id|fc_id`). Mit der Bereinigung in Version 2026.1.22 wurde
die Erweiterung entfernt, die Logik im Runner zentralisiert und OpenAI blieb über die Bildbereinigung hinaus
**unverändert**.

## Verwandte Themen

- [Sitzungsverwaltung](/de/concepts/session)
- [Sitzungsbereinigung](/de/concepts/session-pruning)
