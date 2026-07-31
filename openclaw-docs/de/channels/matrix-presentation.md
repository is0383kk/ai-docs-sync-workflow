---
read_when:
    - Matrix-Clients erstellen, die Rich-Responses von OpenClaw darstellen
    - Debuggen von Ereignisinhalten für com.openclaw.presentation
summary: Matrix-MessagePresentation-Metadaten für OpenClaw-kompatible Clients
title: Matrix-Präsentationsmetadaten
x-i18n:
    generated_at: "2026-07-26T17:39:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c0de4d13c6cefc6f91dcc7a4b0edeea6bf001f3bd71f52c9f0498ad422783d8a
    source_path: channels/matrix-presentation.md
    workflow: 16
---

OpenClaw fügt ausgehenden Matrix-`m.room.message`-Ereignissen unter dem Inhaltsschlüssel `com.openclaw.presentation` normalisierte `MessagePresentation`-Metadaten hinzu.

Standardmäßige Matrix-Clients stellen weiterhin den Klartext `body` dar. OpenClaw-kompatible Clients können die strukturierten Metadaten lesen und native Benutzeroberflächenelemente wie Schaltflächen, Auswahllisten, Kontextzeilen und Trennlinien darstellen.

## Ereignisinhalt

```json
{
  "msgtype": "m.text",
  "body": "Modell auswählen\n\nModell auswählen:\n- DeepSeek",
  "com.openclaw.presentation": {
    "version": 1,
    "type": "message.presentation",
    "title": "Modell auswählen",
    "tone": "info",
    "blocks": [
      {
        "type": "select",
        "placeholder": "Modell auswählen",
        "options": [
          {
            "label": "DeepSeek",
            "value": "/model deepseek/deepseek-chat"
          }
        ]
      }
    ]
  }
}
```

- `version` ist die Version des Metadatenschemas; die aktuelle Version ist `1`. `type` ist ein stabiler Diskriminator und lautet immer `"message.presentation"`. Der Matrix-Adapter gibt nur Payloads mit exakt dieser Version und diesem Typ aus; Clients sollten ebenso unbekannte Versionen, die sie nicht sicher interpretieren können, unbekannte `type`-Werte und unbekannte Blocktypen ignorieren.
- `title` und `tone` (`info`, `success`, `warning`, `danger`, `neutral`) sind optionale Hinweise.
- Schaltflächen und Auswahloptionen können neben der veralteten Zeichenfolge `value` eine typisierte `action` (`{ "type": "command", "command": "/..." }` oder `{ "type": "callback", "value": "..." }`) enthalten. Wenn beide vorhanden sind, ist `action` zu bevorzugen.

## Fallback-Verhalten

OpenClaw stellt in `body` immer einen lesbaren Klartext-Fallback bereit. Die strukturierten Metadaten sind additiv und dürfen für die grundlegende Matrix-Interoperabilität nicht erforderlich sein.

Regeln für die Fallback-Darstellung:

- Inhalte vom Typ `title`, `text` und `context` werden als Klartextzeilen dargestellt.
- Schaltflächen mit einer `command`-Aktion werden als ``label: `/command` `` dargestellt, damit der Befehl kopierbar bleibt. Schaltflächen mit einer `callback`-Aktion oder nur einem veralteten `value` werden ausschließlich mit ihrer Beschriftung dargestellt, damit nicht transparente Callback-Werte vertraulich bleiben; deaktivierte Schaltflächen werden immer ausschließlich mit ihrer Beschriftung dargestellt. URL- und Web-App-Schaltflächen werden als `label: URL` dargestellt.
- Auswahlblöcke stellen den Platzhalter (oder `Options:`) als Überschrift sowie Optionszeilen dar, die ausschließlich die jeweilige Beschriftung enthalten.
- Wenn nichts dargestellt wird, beispielsweise bei einer Präsentation, die nur aus einer Trennlinie besteht, fällt der Nachrichtentext auf `---` zurück.

Nicht unterstützte Clients zeigen weiterhin den Fallback-Text an. OpenClaw-kompatible Clients können für die Anzeige die strukturierten Metadaten bevorzugen und gleichzeitig den Fallback für Kopieren, Suche, Benachrichtigungen und Barrierefreiheit beibehalten.

## Unterstützte Blöcke

Der ausgehende Matrix-Adapter weist native Unterstützung für Folgendes aus:

- `buttons`
- `select`
- `context`
- `divider`

`text`-Blöcke werden über den Fallback-Nachrichtentext immer unterstützt. Behandeln Sie alle Blöcke als nach bestem Bemühen zu berücksichtigende Darstellungshinweise; ignorieren Sie unbekannte Felder und Blocktypen, anstatt die gesamte Nachricht abzulehnen.

## Interaktionen

Diese Metadaten fügen keine Matrix-Callback-Semantik hinzu. Schaltflächen- und Auswahlwerte sind Fallback-Interaktions-Payloads, üblicherweise Slash-Befehle oder Textbefehle. Ein Matrix-Client, der Interaktionen unterstützen möchte, ermittelt den Steuerelementwert (`action.command`, dann `action.value`, dann `value`) und sendet ihn als normale Nachricht an den Raum zurück.

Eine Schaltfläche mit dem Wert `/model deepseek/deepseek-chat` kann beispielsweise verarbeitet werden, indem dieser Wert im selben Raum als verschlüsselte Matrix-Textnachricht gesendet wird.

## Beziehung zu Genehmigungsmetadaten

`com.openclaw.presentation` dient der allgemeinen Darstellung formatierter Nachrichten.

Genehmigungsaufforderungen verwenden die dedizierten `com.openclaw.approval`-Metadaten, da Genehmigungen sicherheitsrelevante Zustände, Entscheidungen sowie Ausführungs- und Plugin-Details enthalten. Wenn beide Metadatenschlüssel im selben Ereignis vorhanden sind, sollten Clients den dedizierten Genehmigungs-Renderer bevorzugen.

## Mediennachrichten

Wenn eine Antwort mehrere Medien-URLs enthält, sendet OpenClaw pro Medien-URL ein Matrix-Ereignis. Beschriftungstext und Präsentationsmetadaten werden nur dem ersten Ereignis hinzugefügt, sodass Clients einen einzigen stabilen strukturierten Payload ohne doppelte Renderer erhalten. Dieselbe Regel gilt, wenn langer Text auf mehrere Ereignisse aufgeteilt wird: Die Metadaten werden nur im ersten Ereignis übermittelt.

Halten Sie Präsentationsmetadaten kompakt. Umfangreicher für Benutzer sichtbarer Text sollte in `body` verbleiben und den normalen Pfad zur Aufteilung von Matrix-Text verwenden.
