---
read_when:
    - Erläuterung von Token-Nutzung, Kosten oder Kontextfenstern
    - Kontextwachstum oder Compaction-Verhalten debuggen
summary: Wie OpenClaw den Prompt-Kontext erstellt und Token-Nutzung sowie Kosten meldet
title: Token-Nutzung und Kosten
x-i18n:
    generated_at: "2026-07-26T18:06:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6624bceb0bcbca769c9d569389b73b82f1ea73133e09f0ae9859833196d85911
    source_path: reference/token-use.md
    workflow: 16
---

OpenClaw erfasst **Tokens**, nicht Zeichen. Tokens sind modellspezifisch, aber die meisten
OpenAI-ähnlichen Modelle kommen bei englischem Text auf durchschnittlich ~4 Zeichen pro Token.

## So wird der System-Prompt erstellt

OpenClaw stellt bei jedem Lauf einen eigenen System-Prompt zusammen. Er enthält:

- Werkzeugliste + Kurzbeschreibungen
- Skills-Liste (nur Metadaten; Anweisungen werden bei Bedarf mit `read` geladen). Native
  Codex-Turns erhalten den kompakten Skills-Block als turnbezogene Entwickleranweisungen
  für die Zusammenarbeit; andere Harnesses erhalten ihn auf der normalen Prompt-Oberfläche.
  Begrenzt durch `skills.limits.maxSkillsPromptChars`, mit optionaler Überschreibung pro Agent
  unter `agents.entries.*.skillsLimits.maxSkillsPromptChars`.
- Anweisungen zur Selbstaktualisierung
- Workspace- + Bootstrap-Dateien (`AGENTS.md`, `SOUL.md`, `TOOLS.md`,
  `IDENTITY.md`, `USER.md`, `HEARTBEAT.md`, `BOOTSTRAP.md` bei neuen Workspaces sowie
  `MEMORY.md`, falls vorhanden). Große injizierte Dateien werden durch
  `agents.defaults.bootstrapMaxChars` gekürzt (Standard: `20000`); die gesamte Bootstrap-
  Injektion wird durch `agents.defaults.bootstrapTotalMaxChars` begrenzt (Standard:
  `60000`).
  - Native Codex-Turns fügen kein unformatiertes `MEMORY.md` ein, wenn für
    diesen Workspace Speicherwerkzeuge verfügbar sind; stattdessen erhalten sie einen kleinen
    Speicherverweis in turnbezogenen Entwickleranweisungen für die Zusammenarbeit und verwenden
    Speicherwerkzeuge bei Bedarf. Wenn Werkzeuge deaktiviert sind, die Speichersuche nicht verfügbar ist
    oder sich der aktive Workspace vom Speicher-Workspace des Agenten unterscheidet, greift `MEMORY.md`
    auf den normalen begrenzten Turn-Kontextpfad zurück.
  - Das kleingeschriebene Stammverzeichnis `memory.md` wird niemals injiziert. Es dient als Legacy-Reparatureingabe
    für `openclaw doctor --fix`, das es nach `MEMORY.md` migriert.
  - Tägliche `memory/*.md`-Dateien sind nicht Teil des normalen Bootstrap-Prompts;
    sie bleiben bei gewöhnlichen Turns über Speicherwerkzeuge bei Bedarf verfügbar. Modellläufe
    beim Zurücksetzen oder Starten können für diesen ersten Turn einmalig einen Startkontextblock
    mit aktuellen täglichen Speicherinhalten voranstellen, gesteuert durch
    `agents.defaults.startupContext`. Reine Chat-Befehle `/new` und `/reset` werden
    bestätigt, ohne das Modell aufzurufen.
  - Auszüge aus `AGENTS.md` nach der Compaction erfordern eine ausdrückliche
    Aktivierung über `agents.defaults.compaction.postCompactionSections`; Plugins können über
    `before_prompt_build` weiteren Kontext hinzufügen.
- Zeit (UTC + Zeitzone des Benutzers)
- Antwort-Tags + Heartbeat-Verhalten
- Laufzeitmetadaten (Host/Betriebssystem/Modell/Denken)

Die vollständige Aufschlüsselung finden Sie unter [System-Prompt](/de/concepts/system-prompt).

Verwenden Sie beim Dokumentieren von Anmeldedaten oder Authentifizierungsausschnitten die
[Konventionen für Geheimnisplatzhalter](/de/reference/secret-placeholder-conventions), um
Fehlalarme von Geheimnisscannern bei reinen Dokumentationsänderungen zu vermeiden.

## Was zum Kontextfenster zählt

Alles, was das Modell empfängt, wird auf das Kontextlimit angerechnet:

- System-Prompt (alle obigen Abschnitte)
- Konversationsverlauf (Nachrichten von Benutzer + Assistent)
- Werkzeugaufrufe und Werkzeugergebnisse
- Anhänge/Transkripte (Bilder, Audio, Dateien)
- Compaction-Zusammenfassungen und Bereinigungsartefakte
- Provider-Wrapper oder Sicherheits-Header (nicht sichtbar, werden aber dennoch angerechnet)

Laufzeitintensive Oberflächen haben eigene explizite Begrenzungen unter
`agents.defaults.contextLimits` (Überschreibungen pro Agent unter
`agents.entries.*.contextLimits`):

| Schlüssel                 | Zweck                                                                    |
| ------------------------- | ------------------------------------------------------------------------ |
| `memoryGetMaxChars`      | Maximale Zeichenanzahl, die `memory_get` vor der Kürzung zurückgibt.    |
| `postCompactionMaxChars` | Maximale Zeichenanzahl, die während der Aktualisierung nach der Compaction aus `AGENTS.md` beibehalten wird. |

Dies sind begrenzte Laufzeitauszüge und injizierte, laufzeiteigene Blöcke,
getrennt von Bootstrap-Limits, Startkontextlimits und Limits für Skills-Prompts.

OpenClaw leitet das aktuelle Limit für Werkzeugergebnisse aus dem effektiven
Modellkontextfenster ab: `16000` Zeichen bei weniger als
100K Tokens, `32000` Zeichen ab 100K Tokens und `64000` Zeichen ab 200K Tokens.
Die Laufzeitschutzvorrichtung für den Kontextanteil begrenzt außerdem ein einzelnes Werkzeugergebnis auf 30 % des
Kontextfensters.

Große Provider-Fenster werden nicht automatisch aktiviert, wenn sie Kosten
oder Latenz wesentlich verändern. Beispielsweise veröffentlichen direkte OpenAI-Modelle GPT-5.5 und GPT-5.6
ein Gesamtfenster von `1050000` Tokens, aber OpenClaw setzt ihr aktives
Laufzeitbudget standardmäßig auf `272000` Tokens. Das optional aktivierbare Eingabebudget `922000` reserviert das
vollständige Ausgabelimit von `128000`, und OpenAI wendet die höheren Preise für lange Kontexte
auf die gesamte Anfrage an, sobald die Eingabe `272000` Tokens überschreitet. Siehe
[Standardwerte für das OpenAI-Kontextfenster](/de/providers/openai#context-window-defaults-and-long-context-opt-in).

Für Bilder verkleinert OpenClaw Bildnutzlasten aus Transkripten/Werkzeugen vor
Provider-Aufrufen. Passen Sie dies mit `agents.defaults.imageMaxDimensionPx` an (Standard:
`1200`):

- Niedrigere Werte reduzieren die Nutzung von Vision-Tokens und die Nutzlastgröße.
- Höhere Werte bewahren mehr visuelle Details für OCR/UI-intensive Screenshots.

Für eine praktische Aufschlüsselung (nach injizierter Datei, Werkzeugen, Skills und Größe des
System-Prompts) verwenden Sie `/context list` oder `/context detail`. Siehe
[Kontext](/de/concepts/context).

## So zeigen Sie die aktuelle Token-Nutzung an

Im Chat:

- `/status` -> Emoji-reiche Statuskarte mit dem Sitzungsmodell, der Kontextnutzung,
  den Eingabe-/Ausgabe-Tokens der letzten Antwort und den geschätzten Kosten, wenn lokale Preise
  für das aktive Modell konfiguriert sind.
- `/usage off|tokens|full` -> fügt jeder Antwort eine Nutzungsfußzeile pro Antwort
  hinzu. Bleibt pro Sitzung erhalten (gespeichert als `responseUsage`).
  - `/usage reset` (Aliasse: `inherit`, `clear`, `default`) löscht die
    Sitzungsüberschreibung, sodass wieder der konfigurierte Standard übernommen wird.
  - `/usage tokens` zeigt Token-/Cache-Details des Turns.
  - `/usage full` zeigt kompakte Modell-/Kontext-/Kostendetails; geschätzte Kosten
    erscheinen nur, wenn OpenClaw über Nutzungsmetadaten und lokale Preise für das
    aktive Modell verfügt. Benutzerdefinierte `messages.usageTemplate`-Layouts können
    Token-/Cache-Felder enthalten.
- `/usage cost` -> lokale Kostenzusammenfassung aus OpenClaw-Sitzungsprotokollen.

Andere Oberflächen:

- **TUI/Web-TUI:** `/status` und `/usage` werden unterstützt.
- **CLI:** `openclaw status --usage` und `openclaw channels list` zeigen
  normalisierte Provider-Kontingentfenster (`X% left`, keine Kosten pro Antwort).
  Aktuelle Provider für Nutzungsfenster: Claude (Anthropic), ClawRouter, Copilot
  (GitHub), DeepSeek, Gemini (Google Gemini CLI), MiniMax, OpenAI, Xiaomi,
  Xiaomi Token Plan und z.ai.

Nutzungsoberflächen normalisieren vor der Anzeige gängige Provider-native Feldaliase.
Für Responses-Datenverkehr der OpenAI-Familie umfasst dies sowohl
`input_tokens`/`output_tokens` als auch `prompt_tokens`/`completion_tokens`, sodass
transportspezifische Feldnamen `/status`, `/usage` oder Sitzungszusammenfassungen
nicht verändern. Die Nutzung der Gemini CLI wird ebenfalls normalisiert: Der standardmäßige `stream-json`-
Parser liest Assistentenereignisse vom Typ `message`, und `stats.cached` wird
auf `cacheRead` abgebildet, wobei `stats.input_tokens - stats.cached` verwendet wird, wenn die CLI
kein explizites Feld `stats.input` bereitstellt. Legacy-JSON-Überschreibungen lesen den Antworttext weiterhin
aus `response`.

Bei nativem Responses-Datenverkehr der OpenAI-Familie werden WebSocket-/SSE-Nutzungsaliase
auf dieselbe Weise normalisiert, und Gesamtwerte greifen auf normalisierte Eingabe + Ausgabe
zurück, wenn `total_tokens` fehlt oder `0` ist.

Wenn der aktuelle Sitzungssnapshot unvollständig ist, können `/status` und `session_status`
Token-/Cache-Zähler und die Bezeichnung des aktiven Laufzeitmodells aus dem
neuesten Transkript-Nutzungsprotokoll wiederherstellen. Vorhandene Live-Werte ungleich null haben weiterhin
Vorrang vor Transkript-Rückfallwerten, und größere promptorientierte
Transkript-Gesamtwerte können Vorrang erhalten, wenn gespeicherte Gesamtwerte fehlen oder kleiner sind.

Die Nutzungsauthentifizierung für Provider-Kontingentfenster stammt zuerst aus providerspezifischen Hooks;
wenn ein Provider keinen Hook besitzt (oder der Hook kein Token auflöst),
greift OpenClaw auf passende OAuth-/API-Schlüssel-Anmeldedaten aus Authentifizierungsprofilen,
Umgebungsvariablen oder der Konfiguration zurück.

Assistenten-Transkripteinträge speichern dieselbe normalisierte Nutzungsstruktur,
einschließlich `usage.cost`, wenn für das aktive Modell Preise konfiguriert sind und der
Provider Nutzungsmetadaten zurückgibt. Dadurch erhalten `/usage cost` und
der transkriptgestützte Sitzungsstatus eine stabile Quelle, selbst nachdem der aktive
Laufzeitstatus nicht mehr vorhanden ist.

OpenClaw hält die Provider-Nutzungsabrechnung vom aktuellen Kontextsnapshot
getrennt. Provider-`usage.total` kann zwischengespeicherte Eingaben, Ausgaben und
mehrere Modellaufrufe in Werkzeugschleifen enthalten. Daher ist es für Kosten und Telemetrie nützlich, kann
das aktuelle Kontextfenster jedoch überbewerten. Kontextanzeigen und Diagnosen verwenden
den neuesten Prompt-Snapshot (`promptTokens` oder den letzten Modellaufruf, wenn kein
Prompt-Snapshot verfügbar ist) für `context.used`.

## Kostenschätzung (wenn angezeigt)

Kosten werden anhand Ihrer Modellpreiskonfiguration geschätzt:

```text
models.providers.<provider>.models[].cost
```

Dies sind **USD pro 1M Tokens** für `input`, `output`, `cacheRead` und
`cacheWrite`. Wenn Preisangaben fehlen, lässt `/usage full` die Kosten weg; verwenden Sie
`/usage tokens` oder ein benutzerdefiniertes `messages.usageTemplate`, wenn Sie
Token-/Cache-Details in jeder Antwort benötigen. Die Kostenanzeige ist nicht auf die Authentifizierung mit API-Schlüssel
beschränkt: Provider ohne API-Schlüssel wie `aws-sdk` können geschätzte Kosten anzeigen, wenn
ihr konfigurierter Modelleintrag lokale Preise enthält und der Provider
Nutzungsmetadaten zurückgibt.

Nachdem Sidecars und Kanäle den Bereitschaftspfad des Gateways erreicht haben, startet OpenClaw einen
optionalen Hintergrund-Bootstrap für Preise von konfigurierten Modellreferenzen, die noch
keine lokalen Preise besitzen. Dieser Bootstrap ruft entfernte Preiskataloge von OpenRouter und
LiteLLM ab. Setzen Sie `models.pricing.enabled: false`, um diese
Katalogabrufe in Offline- oder eingeschränkten Netzwerken zu überspringen; explizite
`models.providers.*.models[].cost`-Einträge bestimmen weiterhin lokale Kostenschätzungen.

## Auswirkungen von Cache-TTL und Bereinigung

Das Prompt-Caching des Providers gilt nur innerhalb des Cache-TTL-Fensters. OpenClaw
kann optional eine **Cache-TTL-Bereinigung** durchführen: Die Sitzung wird bereinigt, sobald die
Cache-TTL abgelaufen ist, anschließend wird das Cache-Fenster zurückgesetzt, sodass nachfolgende Anfragen
den neu zwischengespeicherten Kontext wiederverwenden, anstatt den vollständigen Verlauf erneut zwischenzuspeichern.
Dadurch bleiben die Kosten für Cache-Schreibvorgänge niedriger, wenn eine Sitzung länger als die TTL inaktiv bleibt.

Konfigurieren Sie dies in der [Gateway-Konfiguration](/de/gateway/configuration) und lesen Sie die
Verhaltensdetails unter [Sitzungsbereinigung](/de/concepts/session-pruning).

Heartbeat kann den Cache über Inaktivitätsphasen hinweg **warm** halten. Wenn die Cache-
TTL Ihres Modells `1h` beträgt, kann ein Heartbeat-Intervall knapp darunter (z. B. `55m`)
verhindern, dass der vollständige Prompt erneut zwischengespeichert wird, und so die Kosten für Cache-Schreibvorgänge senken.

In Multi-Agent-Konfigurationen können Sie eine gemeinsame Modellkonfiguration beibehalten und das Cache-
Verhalten pro Agent mit `agents.entries.*.params.cacheRetention` anpassen.

Eine vollständige Anleitung zu jeder einzelnen Einstellung finden Sie unter [Prompt-Caching](/de/reference/prompt-caching).

Bei der API-Preisgestaltung von Anthropic sind Cache-Lesevorgänge deutlich günstiger als Eingabe-
Tokens, während Cache-Schreibvorgänge mit einem höheren Multiplikator abgerechnet werden. Die neuesten Preise und TTL-Multiplikatoren
für Prompt-Caching finden Sie bei Anthropic:
[https://docs.anthropic.com/docs/build-with-claude/prompt-caching](https://docs.anthropic.com/docs/build-with-claude/prompt-caching)

### Beispiel: 1h-Cache mit Heartbeat warm halten

```yaml
agents:
  defaults:
    model:
      primary: "anthropic/claude-opus-4-6"
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "long"
    heartbeat:
      every: "55m"
```

### Beispiel: Gemischter Datenverkehr mit Cache-Strategie pro Agent

```yaml
agents:
  defaults:
    model:
      primary: "anthropic/claude-opus-4-6"
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "long" # Standardbasiswert für die meisten Agenten
  list:
    - id: "research"
      default: true
      heartbeat:
        every: "55m" # langen Cache für umfangreiche Sitzungen aktiv halten
    - id: "alerts"
      params:
        cacheRetention: "none" # Cache-Schreibvorgänge bei stoßweisen Benachrichtigungen vermeiden
```

`agents.entries.*.params` wird mit den `params` des ausgewählten Modells zusammengeführt, sodass Sie
nur `cacheRetention` überschreiben und andere Modellstandardwerte
unverändert übernehmen können.

### Anthropic-Kontext mit 1M

OpenClaw dimensioniert allgemein verfügbare Claude-4.x-Modelle wie Opus 4.8, Opus 4.7, Opus
4.6 und Sonnet 4.6 mit dem 1M-Kontextfenster von Anthropic. Für diese Modelle benötigen Sie
`params.context1m: true` nicht.

```yaml
agents:
  defaults:
    models:
      "anthropic/claude-opus-4-6":
        alias: opus
```

Ältere Konfigurationen können `context1m: true` beibehalten, aber OpenClaw sendet
für diese Einstellung nicht mehr den eingestellten Anthropic-Beta-Header `context-1m-2025-08-07` und
erweitert nicht unterstützte ältere Claude-Modelle nicht auf 1M.

Voraussetzung: Die Anmeldedaten müssen für die Nutzung langer Kontexte berechtigt sein. Andernfalls
antwortet Anthropic für diese Anfrage mit einem Provider-seitigen Ratenbegrenzungsfehler.

Wenn Sie sich bei Anthropic mit OAuth-/Abonnement-Tokens
(`sk-ant-oat-*`) authentifizieren, behält OpenClaw die für OAuth erforderlichen Anthropic-Beta-
Header bei und entfernt zugleich den eingestellten Beta-Wert `context-1m-*`, falls er noch in
einer älteren Konfiguration vorhanden ist.

## Tipps zur Verringerung des Token-Drucks

- Verwenden Sie `/compact`, um lange Sitzungen zusammenzufassen.
- Kürzen Sie große Tool-Ausgaben in Ihren Workflows.
- Verringern Sie `agents.defaults.imageMaxDimensionPx` bei Sitzungen mit vielen Screenshots.
- Halten Sie Skill-Beschreibungen kurz (die Skill-Liste wird in den Prompt eingefügt).
- Bevorzugen Sie kleinere Modelle für ausführliche, explorative Arbeiten.

Die genaue Formel für den zusätzlichen Umfang der Skill-Liste finden Sie unter [Skills](/de/tools/skills).

## Verwandte Themen

- [API-Nutzung und Kosten](/de/reference/api-usage-costs)
- [Prompt-Caching](/de/reference/prompt-caching)
- [Nutzungsverfolgung](/de/concepts/usage-tracking)
