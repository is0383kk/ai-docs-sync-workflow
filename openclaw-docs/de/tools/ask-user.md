---
read_when:
    - Sie möchten, dass ein Agent dem Benutzer eine strukturierte Frage stellt
    - Sie beantworten oder debuggen eine ask_user-Eingabeaufforderung
    - Sie benötigen das `ask_user`-Schema, das Zeitlimit oder das Kanalverhalten
summary: Wie ask_user einen Agenten-Durchlauf für eine strukturierte menschliche Entscheidung pausiert
title: Benutzer fragen
x-i18n:
    generated_at: "2026-07-26T18:39:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 32556314a34c26054c3aabfdd8ecc474cf85196e5cc71adb833face596edbd24
    source_path: tools/ask-user.md
    workflow: 16
---

`ask_user` ermöglicht es dem Agenten, dem Menschen eine bis drei strukturierte Fragen zu stellen und
auf die Antworten zu warten. Es ist für Entscheidungen vorgesehen, die tatsächlich der Benutzer treffen muss,
nicht für routinemäßige Bestätigungen oder Informationen, die der Agent aus der Anfrage,
dem Code oder einer sinnvollen Standardeinstellung ableiten kann.

Das Tool ist nur in der Hauptsitzung verfügbar. Subagenten und andere nicht primäre
Ausführungen erhalten es nicht.

## Eine Frage beantworten

Sie können über jede unterstützte Konversationsoberfläche antworten:

- Die webbasierte Control UI verankert ein Fragen-Panel direkt über dem Eingabefeld. Bei
  Aufforderungen mit mehreren Fragen zeigt das Panel jeweils eine Frage an und führt
  über einen kurzen Stepper durch die Fragen. Nach der Beantwortung wird das Panel geschlossen und im Chat
  bleibt nur eine kompakte Zusammenfassung der Antworten erhalten.
- Telegram, Discord und Slack zeigen native Schaltflächen für eine Frage
  mit Einfachauswahl an.
- Eine Klartextantwort funktioniert auf jedem Kanal. Antworten Sie mit einer Zahl, einer Optionsbezeichnung
  oder Ihrer eigenen Antwort.

OpenClaw aktiviert stets die Freitextantwort **Sonstiges**. Der Agent darf der erstellten Optionsliste keine
Option `Other` hinzufügen.

## Plattformverhalten

Antworten funktionieren auf jeder unterstützten Konversationsoberfläche. Die webbasierte Control UI verwendet einen
verankerten Stepper, der im ausgeklappten Zustand das Eingabefeld ersetzt; beim Einklappen wird
das vollständige Eingabefeld unter einer schmalen Fragenleiste wiederhergestellt. iOS, macOS und Android zeigen
Inline-Karten an; mehrere Fragen bleiben als bewusst berührungsfreundliches
Bedienmuster gestapelt. Jede Plattform behält die Zusammenfassung von Fragen und Antworten ohne zeitgesteuertes Entfernen
in der aktiven Chat-Chronik bei, und **Überspringen** ist überall verfügbar.

Aufforderungen, die keine nativen Schaltflächen verwenden können, darunter Aufforderungen mit mehreren Fragen und
Mehrfachauswahl, werden auf Kanälen als lesbarer Text dargestellt. Die Control UI
behält den vollständigen strukturierten Stepper bei.

## Zeitüberschreitung und keine Antwort

Die standardmäßige Zeitüberschreitung beträgt 900 Sekunden. `timeoutSeconds` wird auf den Bereich
von 30 bis 3600 Sekunden begrenzt.

Wenn die Frage abläuft oder abgebrochen wird, bevor eine Antwort eingeht, gibt das Tool
`status: "no_answer"` zurück. Der Agent fährt dann nach bestem Ermessen fort.
Eine abgebrochene Agentenausführung bricht die ausstehende Gateway-Frage ab.

## Tool-Schema

```ts
{
  questions: Array<{
    id: string; // eindeutiger Antwortschlüssel in snake_case
    header: string; // kurze Bezeichnung; auf 12 Zeichen gekürzt
    question: string; // ein Satz
    options: Array<{
      label: string;
      description?: string;
    }>; // 2-4 Optionen
    multiSelect?: boolean;
  }>; // 1-3 Fragen
  timeoutSeconds?: number; // Ganzzahl; Standardwert 900, begrenzt auf 30-3600
}
```

Mit `multiSelect: true` kann der Benutzer mehr als eine Option auswählen. Die Antwortwerte
werden für jede Frage als Array zurückgegeben.

Beispiel für ein beantwortetes Ergebnis:

```json
{
  "status": "answered",
  "answers": {
    "answers": {
      "deploy_target": ["Staging (Recommended)"]
    }
  }
}
```

## Modellrichtlinien

Der für das Modell bestimmte Vertrag weist den Agenten an:

- nur zu fragen, wenn er aufgrund einer Entscheidung blockiert ist, die tatsächlich der Benutzer treffen muss;
- vorzugsweise eine Frage und höchstens drei zu verwenden;
- die empfohlene Option an die erste Stelle zu setzen und ihre Bezeichnung mit `(Recommended)` zu ergänzen;
- eine erstellte Option `Other` wegzulassen, da Freitext automatisch hinzugefügt wird;
- nach `no_answer` nach bestem Ermessen fortzufahren.

Der Agent sollte `ask_user` nicht verwenden, um zu fragen, ob er fortfahren darf, oder um
seinen eigenen Plan bestätigen zu lassen.
