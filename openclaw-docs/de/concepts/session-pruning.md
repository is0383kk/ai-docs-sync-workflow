---
read_when:
    - Sie möchten das Anwachsen des Kontexts durch Tool-Ausgaben reduzieren
    - Sie möchten die Optimierung des Anthropic-Prompt-Caches verstehen
summary: Alte Tool-Ergebnisse kürzen, um den Kontext schlank und das Caching effizient zu halten
title: Sitzungsbereinigung
x-i18n:
    generated_at: "2026-07-26T17:45:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: dd5cb4582cb8d9d7265213abe1f5b5893634882b9f8b3ce1deef746293dd07db
    source_path: concepts/session-pruning.md
    workflow: 16
---

Session-Pruning kürzt **alte Tool-Ergebnisse** vor jedem LLM-Aufruf aus dem Kontext. Dadurch wird eine Aufblähung des Kontexts durch angesammelte Tool-Ausgaben (Ausführungsergebnisse, gelesene Dateien, Suchergebnisse) reduziert, ohne normalen Konversationstext umzuschreiben.

<Info>
Das Pruning erfolgt nur im Arbeitsspeicher – es verändert das auf dem Datenträger gespeicherte Sitzungsprotokoll nicht. Ihr vollständiger Verlauf bleibt stets erhalten.
</Info>

## Warum es wichtig ist

In langen Sitzungen sammeln sich Tool-Ausgaben an, die das Kontextfenster vergrößern. Dies erhöht die Kosten und kann eine [Compaction](/de/concepts/compaction) früher als nötig erzwingen.

Pruning ist besonders wertvoll für das **Prompt-Caching von Anthropic**. Nachdem die Cache-TTL abgelaufen ist, speichert die nächste Anfrage den vollständigen Prompt erneut im Cache. Pruning reduziert die Größe des Cache-Schreibvorgangs und senkt dadurch direkt die Kosten.

## Funktionsweise

Pruning wird im Modus `cache-ttl` ausgeführt und hängt sowohl von einer Zeitprüfung als auch von einer Prüfung der Kontextgröße ab:

1. Warten Sie, bis die Cache-TTL abgelaufen ist (standardmäßig 5 Minuten bei manueller Festlegung; den automatischen Anthropic-Standardwert finden Sie unter [Intelligente Standardwerte](#smart-defaults)). Vor Ablauf der TTL wird das Pruning vollständig übersprungen, damit der Prompt-Cache für zeitlich nahe aufeinanderfolgende Interaktionen wiederverwendet werden kann.
2. Nach Ablauf der TTL wird die Gesamtkontextgröße im Verhältnis zum Kontextfenster des Modells geschätzt. Liegt das Verhältnis unter `softTrimRatio` (standardmäßig 0.3), wird das Pruning übersprungen und die TTL-Uhr läuft weiter.
3. **Soft-Trim** für übergroße Tool-Ergebnisse oberhalb des Verhältnisses: Anfang und Ende werden beibehalten (standardmäßig jeweils 1500 Zeichen, zusammen auf 4000 Zeichen begrenzt) und dazwischen wird `...` eingefügt.
4. Wenn das Verhältnis weiterhin mindestens `hardClearRatio` (standardmäßig 0.5) beträgt und mindestens `minPrunableToolChars` (standardmäßig 50,000) an kürzbaren Tool-Inhalten verbleiben, werden diese Ergebnisse **vollständig gelöscht**: Ihr Inhalt wird durch einen Platzhalter ersetzt (standardmäßig `[Old tool result content cleared]`).
5. Die TTL-Uhr wird nur zurückgesetzt, wenn das Pruning den Kontext tatsächlich verändert hat, sodass Folgeanfragen den neuen Cache wiederverwenden.

Unabhängig von den Schwellenwerten gelten zwei Sicherheitsregeln: Die neuesten `keepLastAssistants` Assistant-Interaktionen (standardmäßig 3) werden niemals gekürzt, und Inhalte vor der ersten Benutzernachricht der Sitzung werden ebenfalls niemals gekürzt (dies schützt initiale Lesevorgänge wie `SOUL.md`/`USER.md`).

Nur Nachrichten vom Typ `toolResult` kommen infrage; normaler Konversationstext bleibt unverändert. Mit `agents.defaults.contextPruning.tools.{allow,deny}` legen Sie fest, welche Tool-Namen gekürzt werden dürfen.

## Bereinigung älterer Bilder

OpenClaw erstellt außerdem eine separate idempotente Wiedergabeansicht für Sitzungen, in deren Verlauf rohe Bildblöcke oder Medienmarker zur Prompt-Hydration gespeichert sind.

- Sie behält die **3 neuesten abgeschlossenen Interaktionen** Byte für Byte bei, damit die Präfixe des Prompt-Caches für kürzlich erfolgte Folgeanfragen stabil bleiben. Diese Anzahl umfasst alle abgeschlossenen Interaktionen, nicht nur solche mit Bildern; daher belegen auch reine Textinteraktionen das Fenster.
- In der Wiedergabeansicht werden ältere, bereits verarbeitete Bildblöcke aus dem Verlauf von `user` oder `toolResult` durch `[image data removed - already processed by model]` ersetzt.
- Ältere textuelle Medienverweise wie `[media attached: ...]`, `[Image: source: ...]` und `media://inbound/...` werden durch `[media reference removed - already processed by model]` ersetzt. Anhangsmarker der aktuellen Interaktion bleiben unverändert, damit Vision-Modelle weiterhin neue Bilder laden können.
- Das rohe Sitzungsprotokoll wird nicht umgeschrieben, sodass Verlaufsansichten weiterhin die ursprünglichen Nachrichteneinträge und deren Bilder darstellen können.
- Dies erfolgt unabhängig vom oben beschriebenen normalen Cache-TTL-Pruning. Es verhindert, dass wiederholte Bildnutzlasten oder veraltete Medienverweise bei späteren Interaktionen den Prompt-Cache ungültig machen.

## Intelligente Standardwerte

Das mitgelieferte Anthropic-Plugin konfiguriert beim erstmaligen Auflösen eines Authentifizierungsprofils für Anthropic (oder die Claude CLI) automatisch das Pruning und den Heartbeat-Takt, jedoch nur für Felder, die Sie nicht bereits explizit festgelegt haben:

| Authentifizierungsmodus                  | `contextPruning.mode` | `contextPruning.ttl` | `heartbeat.every` |
| ---------------------------------------- | --------------------- | -------------------- | ----------------- |
| OAuth/Token (einschließlich Wiederverwendung der Claude CLI) | `cache-ttl`           | `1h`                 | `1h`              |
| API-Schlüssel                            | `cache-ttl`           | `1h`                 | `30m`             |

Wenn Sie `agents.defaults.contextPruning.mode` oder `agents.defaults.heartbeat.every` selbst festlegen, überschreibt OpenClaw diese Werte nicht. Dieser automatische Standardwert wird nur bei Authentifizierung für die Anthropic-Familie angewendet; bei anderen Providern ist das Pruning `off`, sofern Sie es nicht konfigurieren.

## Aktivieren oder deaktivieren

Für Provider außerhalb der Anthropic-Familie ist das Pruning standardmäßig deaktiviert. So aktivieren Sie es:

```json5
{
  agents: {
    defaults: {
      contextPruning: { mode: "cache-ttl", ttl: "5m" },
    },
  },
}
```

Zum Deaktivieren: Setzen Sie `mode: "off"`.

## Pruning im Vergleich zur Compaction

|                 | Pruning                    | Compaction                    |
| --------------- | -------------------------- | ----------------------------- |
| **Was**         | Kürzt Tool-Ergebnisse      | Fasst die Konversation zusammen |
| **Gespeichert?** | Nein (pro Anfrage)        | Ja (im Protokoll)             |
| **Umfang**      | Nur Tool-Ergebnisse        | Gesamte Konversation          |

Beide ergänzen einander – Pruning hält Tool-Ausgaben zwischen Compaction-Zyklen kompakt.

## Weiterführende Informationen

- [Compaction](/de/concepts/compaction): kontextreduzierende Zusammenfassung
- [Gateway-Konfiguration](/de/gateway/configuration): alle Konfigurationsoptionen für das Pruning (`contextPruning.*`)

## Verwandte Themen

- [Sitzungsverwaltung](/de/concepts/session)
- [Sitzungs-Tools](/de/concepts/session-tool)
- [Kontext-Engine](/de/concepts/context-engine)
