---
read_when:
    - Sie möchten Auto-Compaction und `/compact` verstehen
    - Sie debuggen lange Sitzungen, die an Kontextgrenzen stoßen
summary: Wie OpenClaw lange Unterhaltungen zusammenfasst, um die Modelllimits einzuhalten
title: Compaction
x-i18n:
    generated_at: "2026-07-26T18:19:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: eb1f794fa60affd602378bcff8b07786bfeca55ab3fa09d5fa7214a05fa48806
    source_path: concepts/compaction.md
    workflow: 16
---

Jedes Modell verfügt über ein Kontextfenster: die maximale Anzahl von Tokens, die es verarbeiten kann. Wenn sich eine Unterhaltung diesem Limit nähert, **komprimiert** OpenClaw ältere Nachrichten zu einer Zusammenfassung, damit der Chat fortgesetzt werden kann.

## Funktionsweise

1. Ältere Gesprächsrunden werden zu einem kompakten Eintrag zusammengefasst.
2. Die Zusammenfassung wird im Sitzungstranskript gespeichert.
3. Neuere Nachrichten bleiben unverändert erhalten.

OpenClaw hält Tool-Aufrufe des Assistenten mit den zugehörigen `toolResult`-Einträgen zusammen, wenn es einen Trennpunkt für die Compaction auswählt. Liegt der Punkt innerhalb eines Tool-Blocks, verschiebt OpenClaw die Grenze, sodass das Paar zusammenbleibt und der aktuelle, nicht zusammengefasste Rest erhalten bleibt.

Der vollständige Gesprächsverlauf verbleibt auf dem Datenträger. Die Compaction ändert nur, was das Modell in der nächsten Gesprächsrunde sieht.

<Note>
Bei neuen Konfigurationen ist `agents.defaults.compaction.mode` standardmäßig auf `"safeguard"` gesetzt (strengere Schutzmechanismen, Qualitätsprüfungen für Zusammenfassungen). Legen Sie `mode: "default"` ausdrücklich fest, um dies zu deaktivieren.
</Note>

## Automatische Compaction

Die automatische Compaction ist standardmäßig aktiviert. Sie wird ausgeführt, wenn sich die Sitzung dem Kontextlimit nähert oder das Modell einen Kontextüberlauffehler zurückgibt (in diesem Fall führt OpenClaw die Compaction durch und versucht es erneut).

Folgendes wird angezeigt:

- `embedded run auto-compaction start` / `complete` in normalen Gateway-Protokollen.
- `🧹 Auto-compaction complete` im ausführlichen Modus.
- `/status` mit `🧹 Compactions: <count>`.

<Info>
Vor der Compaction erinnert OpenClaw den Agenten automatisch daran, wichtige Notizen in [Memory](/de/concepts/memory)-Dateien zu speichern. Dadurch wird ein Kontextverlust verhindert.
</Info>

<AccordionGroup>
  <Accordion title="Von OpenClaw erkannte Muster für Überlauffehler">
    OpenClaw erkennt Dutzende providerspezifischer Fehlermeldungen für Überläufe (Anthropic, OpenAI, Bedrock, Gemini, Ollama, OpenRouter und weitere). Häufige Beispiele:

    - `request_too_large`
    - `context length exceeded`
    - `input exceeds the maximum number of tokens`
    - `input token count exceeds the maximum number of input tokens` (Bedrock)
    - `input is too long for the model`
    - `ollama error: context length exceeded`

  </Accordion>
</AccordionGroup>

## Manuelle Compaction

Geben Sie in einem beliebigen Chat `/compact` ein, um eine Compaction zu erzwingen. Fügen Sie Anweisungen hinzu, um die Zusammenfassung zu steuern:

```text
/compact Konzentriere dich auf die Entscheidungen zum API-Design
```

Wenn `agents.defaults.compaction.keepRecentTokens` festgelegt ist (Standardwert: 20,000), berücksichtigt die manuelle Compaction diesen Trennpunkt und behält den neueren Rest im neu aufgebauten Kontext. Ohne ein ausdrücklich festgelegtes Beibehaltungsbudget verhält sich die manuelle Compaction wie ein fester Prüfpunkt und setzt den Vorgang ausschließlich mit der neuen Zusammenfassung fort.

## Konfiguration

Konfigurieren Sie die Compaction unter `agents.defaults.compaction` in Ihrer `openclaw.json`. Die gebräuchlichsten Optionen sind nachstehend aufgeführt. Die vollständige Referenz finden Sie unter [Ausführliche Informationen zur Sitzungsverwaltung](/de/reference/session-management-compaction).

### Anderes Modell verwenden

Standardmäßig verwendet die Compaction das primäre Modell des Agenten. Legen Sie `agents.defaults.compaction.model` fest, um die Zusammenfassung an ein leistungsfähigeres oder spezialisiertes Modell zu delegieren. Die Überschreibung akzeptiert eine `provider/model-id`-Zeichenfolge oder einen einfachen Alias, der unter `agents.defaults.models` konfiguriert ist:

```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "model": "openrouter/anthropic/claude-sonnet-4-6"
      }
    }
  }
}
```

Einfache konfigurierte Aliasse werden vor Beginn der Compaction in ihren kanonischen Provider und ihr kanonisches Modell aufgelöst. Wenn ein einfacher Wert sowohl einem Alias als auch einer konfigurierten wörtlichen Modell-ID entspricht, hat die wörtliche Modell-ID Vorrang. Ein nicht übereinstimmender einfacher Wert bleibt eine Modell-ID beim aktiven Provider.

Dies funktioniert auch mit lokalen Modellen, beispielsweise mit einem zweiten Ollama-Modell, das ausschließlich für Zusammenfassungen vorgesehen ist:

```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "model": "ollama/llama3.1:8b"
      }
    }
  }
}
```

Wenn keine Einstellung vorgenommen wurde, beginnt die Compaction mit dem aktiven Sitzungsmodell. Schlägt die Zusammenfassung mit einem Providerfehler fehl, der einen Modell-Fallback zulässt, wiederholt OpenClaw diesen Compaction-Versuch über die vorhandene Modell-Fallback-Kette der Sitzung. Die Fallback-Auswahl ist vorübergehend und wird nicht in den Sitzungsstatus zurückgeschrieben. Eine ausdrückliche Überschreibung durch `agents.defaults.compaction.model` bleibt exakt und übernimmt die Fallback-Kette der Sitzung nicht.

### Beibehaltung von Bezeichnern

Bei der Compaction-Zusammenfassung bleiben undurchsichtige Bezeichner standardmäßig erhalten (`identifierPolicy: "strict"`). Mit `identifierPolicy: "off"` können Sie dies deaktivieren. Benutzerdefinierte Vorgaben gehören in die `summarize()`-Implementierung eines Compaction-Providers.

### Byte-Schutz für aktive Transkripte

Wenn `agents.defaults.compaction.maxActiveTranscriptBytes` festgelegt ist, löst OpenClaw
vor einem Lauf eine normale lokale Compaction aus, sobald der Transkriptverlauf
diese Größe erreicht. Dies ist bei lang laufenden Sitzungen hilfreich, bei denen
die providerseitige Kontextverwaltung den Modellkontext intakt halten kann,
während der persistierte Transkriptverlauf weiter wächst. Dabei werden keine
Rohbytes aufgeteilt; stattdessen wird die normale Compaction-Pipeline
aufgefordert, eine semantische Zusammenfassung zu erstellen.

<Warning>
Der Byte-Schutz gilt für den aktiven SQLite-Transkriptverlauf. Ältere
JSONL-Prüfpunktartefakte sind nicht das aktive Ziel der Compaction.
</Warning>

### Nachfolgetranskripte

Wenn `agents.defaults.compaction.truncateAfterCompaction` aktiviert ist, schreibt OpenClaw das vorhandene Transkript nicht direkt um. Es erstellt aus der Compaction-Zusammenfassung, dem beibehaltenen Status und dem nicht zusammengefassten Rest ein neues aktives Nachfolgetranskript und zeichnet anschließend Prüfpunktmetadaten auf, die Verzweigungs- und Wiederherstellungsabläufe auf diesen komprimierten Nachfolger verweisen.
Nachfolgetranskripte verwerfen außerdem exakt duplizierte lange Benutzereingaben,
die innerhalb eines kurzen Wiederholungszeitfensters eintreffen, sodass durch
Kanalwiederholungen verursachte Anfragestürme nach der Compaction nicht in das
nächste aktive Transkript übernommen werden.

OpenClaw schreibt für neue Compactions keine separaten Kopien unter
`.checkpoint.*.jsonl` mehr. Vorhandene ältere Prüfpunktdateien können weiterhin
verwendet werden, solange auf sie verwiesen wird, und werden bei der normalen
Sitzungsbereinigung entfernt.

### Compaction-Benachrichtigungen

Standardmäßig wird die Compaction ohne Benachrichtigung ausgeführt. Legen Sie `notifyUser` fest, um kurze Statusmeldungen beim Start und Abschluss der Compaction anzuzeigen und einen Hinweis auf eine eingeschränkte Funktion einzublenden, wenn ein Memory-Flush vor der Compaction ausgeschöpft ist, die Antwort aber dennoch fortgesetzt wird:

```json5
{
  agents: {
    defaults: {
      compaction: {
        notifyUser: true,
      },
    },
  },
}
```

### Memory-Flush

Vor der Compaction kann OpenClaw eine **stille Memory-Flush**-Runde ausführen, um dauerhafte Notizen auf dem Datenträger zu speichern. Legen Sie `agents.defaults.compaction.memoryFlush.model` fest, wenn diese Wartungsrunde statt des aktiven Gesprächsmodells ein lokales Modell verwenden soll:

```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "memoryFlush": {
          "model": "ollama/qwen3:8b"
        }
      }
    }
  }
}
```

Die Überschreibung des Memory-Flush-Modells ist exakt und übernimmt die Fallback-Kette der aktiven Sitzung nicht. Einzelheiten und Konfigurationsinformationen finden Sie unter [Memory](/de/concepts/memory).

## Austauschbare Compaction-Provider

Plugins können über `registerCompactionProvider()` in der Plugin-API einen benutzerdefinierten Compaction-Provider registrieren. Wenn ein Provider registriert und konfiguriert ist, delegiert OpenClaw die Zusammenfassung an ihn und verwendet nicht die integrierte LLM-Pipeline.

Um einen registrierten Provider zu verwenden, legen Sie dessen ID in Ihrer Konfiguration fest:

```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "provider": "my-provider"
      }
    }
  }
}
```

Das Festlegen eines `provider` erzwingt automatisch `mode: "safeguard"`. Provider erhalten dieselben Compaction-Anweisungen und dieselbe Richtlinie zur Beibehaltung von Bezeichnern wie der integrierte Pfad. OpenClaw bewahrt zudem nach der Provider-Ausgabe den Suffixkontext der neueren Gesprächsrunden und der geteilten Gesprächsrunde.

<Note>
Wenn der Provider fehlschlägt oder ein leeres Ergebnis zurückgibt, greift OpenClaw auf die integrierte LLM-Zusammenfassung zurück.
</Note>

## Compaction im Vergleich zur Bereinigung

|                  | Compaction                              | Bereinigung                               |
| ---------------- | --------------------------------------- | ----------------------------------------- |
| **Funktion**     | Fasst ältere Unterhaltungen zusammen    | Kürzt alte Tool-Ergebnisse                 |
| **Gespeichert?** | Ja (im Sitzungstranskript)              | Nein (nur im Arbeitsspeicher, pro Anfrage) |
| **Umfang**       | Gesamte Unterhaltung                    | Nur Tool-Ergebnisse                        |

Die [Sitzungsbereinigung](/de/concepts/session-pruning) ist eine schlankere Ergänzung, die Tool-Ausgaben kürzt, ohne sie zusammenzufassen.

## Fehlerbehebung

**Compaction wird zu häufig ausgeführt?** Das Kontextfenster des Modells ist möglicherweise klein oder die Tool-Ausgaben sind möglicherweise umfangreich. Versuchen Sie, die [Sitzungsbereinigung](/de/concepts/session-pruning) zu aktivieren.

**Der Kontext wirkt nach der Compaction veraltet?** Verwenden Sie `/compact Focus on <topic>`, um die Zusammenfassung zu steuern, oder aktivieren Sie den [Memory-Flush](/de/concepts/memory), damit Notizen erhalten bleiben.

**Benötigen Sie einen Neuanfang?** `/new` startet eine neue Sitzung ohne Compaction.

Informationen zur erweiterten Konfiguration (reservierte Tokens, Beibehaltung von Bezeichnern, benutzerdefinierte Kontext-Engines, serverseitige Compaction von OpenAI) finden Sie unter [Ausführliche Informationen zur Sitzungsverwaltung](/de/reference/session-management-compaction).

## Verwandte Themen

- [Sitzung](/de/concepts/session): Sitzungsverwaltung und Lebenszyklus.
- [Sitzungsbereinigung](/de/concepts/session-pruning): Kürzen von Tool-Ergebnissen.
- [Kontext](/de/concepts/context): Aufbau des Kontexts für Agentenrunden.
- [Hooks](/de/automation/hooks): Hooks für den Compaction-Lebenszyklus (`before_compaction`, `after_compaction`).
