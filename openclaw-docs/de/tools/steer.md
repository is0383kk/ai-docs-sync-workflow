---
read_when:
    - /steer oder /tell verwenden, während bereits ein Agent ausgeführt wird
    - Vergleich von /steer mit den /queue-Modi
    - Entscheiden, ob der aktuelle Lauf oder eine ACP-Sitzung gesteuert werden soll
sidebarTitle: Steer
summary: Einen aktiven Lauf steuern, ohne den Warteschlangenmodus zu ändern
title: Steuern
x-i18n:
    generated_at: "2026-07-26T18:52:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d420e14982d52520e415103ffa6d86923fad6f13c43ff7741ebbd8dde0d0073f
    source_path: tools/steer.md
    workflow: 16
---

`/steer` versucht zuerst, Anweisungen an einen bereits aktiven Lauf zu senden. Dies ist für Situationen vorgesehen, in denen Sie
„diesen Lauf anpassen möchten, während er noch ausgeführt wird“. Wenn die aktuelle Laufzeitumgebung
keine Steuerung annehmen kann, sendet OpenClaw die Nachricht stattdessen als normalen Prompt,
anstatt sie zu verwerfen.

## Aktuelle Sitzung

Verwenden Sie `/steer` auf oberster Ebene, um den aktiven Lauf der aktuellen Sitzung anzusprechen:

```text
/steer den kleineren Patch bevorzugen und die Tests fokussiert halten
/tell vor dem nächsten Tool-Aufruf zusammenfassen
```

Verhalten:

- Zielt nur auf den aktiven Lauf der aktuellen Sitzung ab.
- Funktioniert unabhängig vom `/queue`-Modus der Sitzung.
- Startet eine normale Interaktion mit derselben Nachricht, wenn die Sitzung inaktiv ist oder der
  aktive Lauf keine Steuerung annehmen kann.
- Verwendet den Steuerungspfad der aktiven Laufzeitumgebung, sodass das Modell die Anweisung an
  der nächsten unterstützten Laufzeitgrenze erhält.

## Steuern oder einreihen

`/queue steer` bewirkt, dass normale eingehende Nachrichten versuchen, den aktiven Lauf zu steuern, wenn
sie während eines aktiven Laufs eintreffen. `/steer <message>` ist ein expliziter Befehl,
der versucht, die Nachricht dieses Befehls an der nächsten unterstützten Laufzeitgrenze in den aktiven Lauf
einzufügen, unabhängig von der gespeicherten Einstellung `/queue`. Wenn
dieses Einfügen nicht verfügbar ist, wird das Befehlspräfix entfernt und `<message>`
als normaler Prompt fortgesetzt.

Der explizite Befehl `/steer` (und `/tell`) wird durch den Gateway unterstützt. Wählen Sie in
`openclaw chat` oder `openclaw tui --local` `/queue steer` aus und senden Sie die
Anweisung als normale Nachricht; die eingebettete Laufzeitumgebung wendet dieselbe Steuerungsrichtlinie an,
ohne einen Gateway-Befehl weiterzuleiten.

Verwenden Sie:

- `/steer <message>`, wenn Sie den aktiven Lauf sofort steuern möchten.
- `/queue steer`, wenn zukünftige normale Nachrichten aktive Läufe standardmäßig
  steuern sollen.
- `/queue collect` oder `/queue followup`, wenn zukünftige normale Nachrichten auf eine
  spätere Interaktion warten sollen, anstatt den aktiven Lauf zu steuern.
- `/queue interrupt`, wenn die neueste Nachricht den aktiven Lauf ersetzen soll,
  anstatt ihn zu steuern.

Informationen zu Warteschlangenmodi und Steuerungsgrenzen finden Sie unter [Befehlswarteschlange](/de/concepts/queue) und
[Steuerungswarteschlange](/de/concepts/queue-steering).

## Sub-Agenten

`/steer` auf oberster Ebene zielt auf den aktiven Lauf der aktuellen Sitzung ab. Sub-Agenten melden
an ihre übergeordnete/anfordernde Sitzung zurück; `/subagents` dient nur der Sichtbarkeit.

## ACP-Sitzungen

Verwenden Sie `/acp steer`, wenn das Ziel eine ACP-Harness-Sitzung ist:

```text
/acp steer --session agent:main:acp:codex die Reproduktion präzisieren
```

Weitere Informationen zur Auswahl von ACP-Sitzungen und zum Laufzeitverhalten finden Sie unter [ACP-Agenten](/de/tools/acp-agents).

## Verwandte Themen

- [Slash-Befehle](/de/tools/slash-commands)
- [Befehlswarteschlange](/de/concepts/queue)
- [Steuerungswarteschlange](/de/concepts/queue-steering)
- [Sub-Agenten](/de/tools/subagents)
