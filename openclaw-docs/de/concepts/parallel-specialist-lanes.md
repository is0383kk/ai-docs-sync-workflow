---
read_when:
    - Sie leiten Gruppenchats an dedizierte Agenten weiter
    - Sie möchten paralleles Arbeiten, ohne dass eine lange Aufgabe jeden Chat blockiert
    - Sie entwerfen eine Multi-Agenten-Betriebsumgebung
sidebarTitle: Specialist lanes
status: active
summary: Führen Sie spezialisierte Agents parallel aus, ohne gemeinsam genutzte Modell- und Tool-Kapazitäten zu überlasten
title: Parallele Spezialistenbereiche
x-i18n:
    generated_at: "2026-07-26T17:48:26Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 09852b6cf5a790e98fb5e0805b0df57b2f3719b1387ecfacfb4973bb6841abb4
    source_path: concepts/parallel-specialist-lanes.md
    workflow: 16
---

Parallele Spezialistenbereiche ermöglichen es einem Gateway, verschiedene Chats oder Räume an
unterschiedliche Agenten weiterzuleiten und dabei eine schnelle Nutzungserfahrung zu gewährleisten. Betrachten Sie Parallelität als
Gestaltungsproblem für knappe Ressourcen, nicht nur als „mehr Agenten“.

## Grundprinzipien

Ein Spezialistenbereich verbessert den Durchsatz nur, wenn er die Konkurrenz um die
tatsächlichen Engpässe verringert:

- **Sitzungssperren**: Es sollte jeweils nur ein Lauf eine bestimmte Sitzung verändern.
- **Globale Modellkapazität**: Alle sichtbaren Chat-Läufe unterliegen weiterhin gemeinsam den Provider-Limits.
- **Tool-Kapazität**: Arbeiten mit Shell, Browser, Netzwerk und Repository können langsamer sein
  als der Modellschritt selbst.
- **Kontextbudget**: Lange Transkripte machen jeden zukünftigen Schritt langsamer und weniger
  fokussiert.
- **Unklare Zuständigkeit**: Doppelte Agenten, die dieselbe Aufgabe erledigen, verschwenden Kapazität.

OpenClaw serialisiert Läufe bereits pro Sitzung und begrenzt die globale Parallelität
über die [Befehlswarteschlange](/de/concepts/queue). Spezialistenbereiche ergänzen darauf aufbauend Richtlinien:
Welcher Agent ist für welche Arbeit zuständig, was verbleibt im Chat und was wird
zur Hintergrundarbeit.

## Empfohlene Einführung

### Phase 1: Bereichsverträge und aufwendige Hintergrundarbeit

Geben Sie jedem Bereich einen schriftlichen Vertrag in seinem Workspace und System-Prompt:

- **Zweck**: die Arbeit, für die dieser Bereich zuständig ist.
- **Nicht-Ziele**: Arbeit, die er weitergeben sollte, statt sie selbst zu versuchen.
- **Chatbudget**: Kurze Antworten verbleiben im Chat; lange Aufgaben werden kurz bestätigt
  und anschließend in einem Hintergrund-Sub-Agenten oder einer Hintergrundaufgabe ausgeführt.
- **Übergaberegel**: Wenn ein anderer Bereich für die Arbeit zuständig ist, geben Sie an, wohin sie gehört,
  und stellen Sie eine kompakte Übergabezusammenfassung bereit.
- **Tool-Risikoregel**: Bevorzugen Sie die kleinste Tool-Oberfläche, mit der sich die Aufgabe erledigen lässt.

Dies ist die kostengünstigste Phase und beseitigt die meisten Blockaden: Ein einzelner Programmierauftrag
macht den Recherchebereich nicht mehr quälend langsam, und jeder Chat hält seinen eigenen Kontext
übersichtlich.

### Phase 2: Prioritäts- und Nebenläufigkeitssteuerung

Stimmen Sie Warteschlangen- und Modellkapazität auf den geschäftlichen Wert jedes Bereichs ab:

```json5
{
  agents: {
    defaults: {
      maxConcurrent: 4,
      subagents: { maxConcurrent: 8, delegationMode: "prefer" },
    },
  },
  messages: {
    queue: {
      mode: "collect",
      debounceMs: 1000,
      cap: 20,
      drop: "summarize",
    },
  },
}
```

Verwenden Sie Direkt-/persönliche Chats und Agenten für den Produktionsbetrieb für Aufgaben mit hoher Priorität. Lassen Sie
Recherche, Entwurfserstellung und gebündelte Programmieraufgaben in den Hintergrund wechseln, wenn das System
ausgelastet ist.

### Phase 3: Koordinator/Verkehrssteuerung

Fügen Sie ein einfaches Koordinatormuster hinzu, sobald mehrere Bereiche aktiv sind:

- Verfolgen Sie aktive Bereichsaufgaben und Zuständigkeiten.
- Erkennen Sie doppelte Anfragen über Gruppen hinweg.
- Leiten Sie Übergabezusammenfassungen zwischen Bereichen weiter.
- Zeigen Sie nur Blockaden, abgeschlossene Ergebnisse und Entscheidungen an, die ein Mensch treffen muss.

Beginnen Sie nicht hier. Ein Koordinator ohne Bereichsverträge koordiniert lediglich Chaos.

## Minimale Vorlage für einen Bereichsvertrag

```md
# Bereichsvertrag

## Zuständig für

- <job this lane is responsible for>

## Nicht zuständig für

- <work to hand off>

## Chatbudget

- Beantworten Sie kurze Fragen direkt.
- Bei mehrstufigen, langsamen oder Tool-intensiven Aufgaben: Bestätigen Sie kurz, starten Sie die Arbeit
  als Sub-Agent oder im Hintergrund und geben Sie das Ergebnis nach Abschluss zurück.

## Übergabe

Wenn ein anderer Bereich für die Anfrage zuständig ist, antworten Sie mit:

- Zielbereich
- Ziel
- relevantem Kontext
- genauer nächster Aktion

## Tool-Strategie

Verwenden Sie die kleinste Tool-Oberfläche, mit der sich die Aufgabe abschließen lässt. Vermeiden Sie umfangreiche Shell- oder
Netzwerkarbeiten, sofern dieser Bereich nicht ausdrücklich dafür zuständig ist.
```

## Verwandte Themen

- [Multi-Agent-Routing](/de/concepts/multi-agent)
- [Befehlswarteschlange](/de/concepts/queue)
- [Sub-Agenten](/de/tools/subagents)
