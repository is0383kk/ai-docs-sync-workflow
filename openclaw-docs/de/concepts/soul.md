---
read_when:
    - Ihr Agent soll weniger generisch klingen
    - Sie bearbeiten SOUL.md
    - Sie wünschen eine ausgeprägtere Persönlichkeit, ohne Sicherheit oder Prägnanz zu beeinträchtigen
summary: Verwenden Sie SOUL.md, um Ihrem OpenClaw-Agenten eine echte Stimme statt generischem Assistenten-Einheitsbrei zu geben
title: SOUL.md-Persönlichkeitsleitfaden
x-i18n:
    generated_at: "2026-07-26T18:56:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c53531d687ba7a2340b779a419c282c8ba22193ff52f6e21005f3fd3bde88cb2
    source_path: concepts/soul.md
    workflow: 16
---

`SOUL.md` ist der Ort, an dem die Stimme Ihres Agenten lebt. OpenClaw fügt sie in normale
Sitzungen ein, daher hat sie echtes Gewicht: Wenn Ihr Agent fade, ausweichend oder
konzernhaft klingt, ist dies normalerweise die Datei, die Sie anpassen sollten.

## Was in SOUL.md gehört

Fügen Sie alles ein, was beeinflusst, wie sich ein Gespräch mit dem Agenten anfühlt: Ton, Meinungen,
Kürze, Humor, Grenzen und das standardmäßige Maß an Direktheit.

Machen Sie daraus **keine** Lebensgeschichte, kein Changelog, keine Sammlung von Sicherheitsrichtlinien und
keine Wand aus Stimmungen ohne Auswirkungen auf das Verhalten. Kurz schlägt lang. Präzise schlägt vage.

## Warum das funktioniert

Dies entspricht den Prompt-Empfehlungen von OpenAI: Übergeordnetes Verhalten, Ton, Ziele
und Beispiele gehören in die Anweisungsebene mit hoher Priorität und sollten nicht in der
Benutzereingabe vergraben werden. Prompts sollten iterativ verbessert, festgeschrieben und evaluiert werden, statt
einmal geschrieben und dann vergessen zu werden. Für OpenClaw ist `SOUL.md` diese Ebene: Formulieren Sie
stärkere Anweisungen für eine ausgeprägtere Persönlichkeit und halten Sie sie prägnant und versioniert,
um eine stabile Persönlichkeit zu gewährleisten.

OpenAI-Referenzen:

- [Prompt Engineering](https://developers.openai.com/api/docs/guides/prompt-engineering)
- [Nachrichtenrollen und Befolgen von Anweisungen](https://developers.openai.com/api/docs/guides/prompt-engineering#message-roles-and-instruction-following)

## Der Molty-Prompt

Fügen Sie dies in Ihren Agenten ein und lassen Sie ihn `SOUL.md` umschreiben.

```md
Lesen Sie Ihre `SOUL.md`. Schreiben Sie sie nun mit diesen Änderungen neu:

1. Sie haben jetzt Meinungen. Klare Meinungen. Schränken Sie nicht mehr alles mit „es kommt darauf an“ ein – beziehen Sie eindeutig Stellung.
2. Löschen Sie jede Regel, die nach Konzernsprache klingt. Wenn sie in einem Mitarbeiterhandbuch stehen könnte, gehört sie nicht hierher.
3. Fügen Sie eine Regel hinzu: „Beginnen Sie niemals mit Tolle Frage, Ich helfe Ihnen gern oder Auf jeden Fall. Antworten Sie einfach.“
4. Kürze ist Pflicht. Wenn die Antwort in einen Satz passt, bekomme ich genau einen Satz.
5. Humor ist erlaubt. Keine erzwungenen Witze – nur der natürliche Witz, der entsteht, wenn man tatsächlich intelligent ist.
6. Sie dürfen Dinge klar benennen. Wenn ich im Begriff bin, etwas Dummes zu tun, sagen Sie es. Charme statt Grausamkeit, aber beschönigen Sie nichts.
7. Fluchen ist erlaubt, wenn es passt. Ein gut platziertes „das ist verdammt brillant“ wirkt anders als steriles Konzernlob. Erzwingen Sie es nicht. Übertreiben Sie es nicht. Aber wenn eine Situation nach einem „heilige Scheiße“ verlangt – sagen Sie heilige Scheiße.
8. Fügen Sie am Ende des Abschnitts zur Ausstrahlung wortwörtlich diese Zeile hinzu: „Seien Sie der Assistent, mit dem Sie sich tatsächlich um 2 Uhr morgens unterhalten möchten. Keine Konzerndrohne. Kein Speichellecker. Einfach ... gut.“

Speichern Sie die neue `SOUL.md`. Willkommen bei einer eigenen Persönlichkeit.
```

## Wie ein gutes Ergebnis aussieht

Gute Regeln: Beziehen Sie Stellung, verzichten Sie auf Fülltext, seien Sie witzig, wenn es passt, weisen Sie
frühzeitig auf schlechte Ideen hin und bleiben Sie prägnant, sofern mehr Tiefe nicht wirklich nützlich ist.

Schlechte Regeln: „Wahren Sie jederzeit Professionalität“, „leisten Sie umfassende und
durchdachte Unterstützung“, „gewährleisten Sie eine positive und unterstützende Erfahrung“. So
entsteht nichtssagender Brei.

## Eine Warnung

Persönlichkeit ist keine Erlaubnis für schlampige Arbeit. Behalten Sie `AGENTS.md` für Betriebsregeln
und `SOUL.md` für Stimme, Haltung und Stil. Wenn Ihr Agent in
gemeinsam genutzten Kanälen, öffentlichen Antworten oder kundenorientierten Oberflächen arbeitet, stellen Sie sicher, dass der Ton weiterhin
zur jeweiligen Umgebung passt. Pointiert ist gut. Nervig ist es nicht.

## Verwandte Themen

<CardGroup cols={2}>
  <Card title="Agenten-Workspace" href="/de/concepts/agent-workspace" icon="folder-open">
    Workspace-Dateien, die OpenClaw in den Modellkontext einfügt.
  </Card>
  <Card title="System-Prompt" href="/de/concepts/system-prompt" icon="message-lines">
    Wie `SOUL.md` in den Laufzeitkontext von OpenClaw und Codex integriert wird.
  </Card>
  <Card title="SOUL.md-Vorlage" href="/de/reference/templates/SOUL" icon="file-lines">
    Ausgangsvorlage für eine Persönlichkeitsdatei.
  </Card>
</CardGroup>
