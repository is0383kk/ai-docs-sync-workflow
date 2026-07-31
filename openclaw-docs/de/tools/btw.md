---
read_when:
    - Sie möchten eine kurze Nebenfrage zur aktuellen Sitzung stellen
    - Sie implementieren oder debuggen das BTW-Verhalten über mehrere Clients hinweg
summary: Kurzlebige Nebenfragen mit /btw
title: Übrigens, Nebenfragen
x-i18n:
    generated_at: "2026-07-26T18:39:39Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 338a54d0e15ec90aebaeeaee551559a26f1437f7b6dcdde4a4b1e63347ad0759
    source_path: tools/btw.md
    workflow: 16
---

`/btw` (Alias `/side`) stellt eine kurze Nebenfrage zur **aktuellen
Sitzung**, ohne sie zum Gesprächsverlauf hinzuzufügen. Die Funktion ist
`/btw` von Claude Code nachempfunden und an die Gateway- und
Mehrkanalarchitektur von OpenClaw angepasst.

```text
/btw was hat sich geändert?
/side was bedeutet dieser Fehler?
```

## Funktionsweise

1. Erstellt einen Snapshot der aktuellen Sitzung als Hintergrundkontext (einschließlich eines
   etwaigen Prompts des laufenden Hauptdurchlaufs).
2. Führt eine separate, einmalige Nebenabfrage aus, bei der das Modell angewiesen wird, nur die
   Nebenfrage zu beantworten und die Hauptaufgabe weder fortzusetzen noch zu steuern.
3. Stellt die Antwort als direktes Nebenergebnis bereit, nicht als normale Assistentennachricht.
4. Schreibt weder die Frage noch die Antwort in den Sitzungsverlauf oder in `chat.history`.

Der Hauptdurchlauf bleibt, sofern einer aktiv ist, unverändert.

Bei Codex-Harness-Sitzungen verzweigt BTW den aktiven Codex-App-Server-Thread
in einen kurzlebigen untergeordneten Thread, statt einen separaten Provider-Aufruf
auszuführen. Dadurch bleiben Codex OAuth sowie das native Werkzeug- und Thread-Verhalten
erhalten, und der verzweigte Thread behält die aktuelle Genehmigungsrichtlinie, Sandbox und
native Werkzeugoberfläche des übergeordneten Threads bei. Der verzweigte Thread erhält einen
Abgrenzungsprompt, der das Modell darauf hinweist, dass alles davor geerbter Referenzkontext
und keine aktiven Anweisungen ist und dass nur Nachrichten nach der Abgrenzung aktiv sind.
`/btw` erfordert einen vorhandenen Codex-Thread; senden Sie zuerst eine normale Nachricht.

Bei CLI-Laufzeit-Aliasen ruft BTW das zuständige CLI-Backend im Modus für
einmalige Nebenfragen auf: Es übergibt bereinigten Gesprächskontext an einen neuen
CLI-Aufruf, bei dem die Werkzeugbündelung und der wiederverwendbare Sitzungsstatus
deaktiviert sind, und fügt alle vom Backend unterstützten Flags zum Verhindern der
Fortsetzung und der Werkzeugnutzung hinzu. Direkte Laufzeiten (ohne CLI)
verwenden stattdessen einen direkten, einmaligen Provider-Aufruf.

## Was die Funktion nicht tut

`/btw` erstellt keine dauerhafte Sitzung, setzt die unfertige Hauptaufgabe nicht fort,
speichert keine Frage-/Antwortdaten im Transkriptverlauf und bleibt nach einem Neuladen
nicht erhalten.

## Bereitstellungsmodell

Normale Assistentenchats verwenden das Gateway-Ereignis `chat`. BTW verwendet ein
separates Ereignis `chat.side_result`, damit Clients es nicht mit dem regulären
Gesprächsverlauf verwechseln können. Da es nicht aus `chat.history` erneut wiedergegeben wird,
verschwindet es nach dem Neuladen.

## Verhalten der Oberflächen

| Oberfläche        | Verhalten                                                                                                                                                                                                                                                                           |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| TUI               | Wird inline im Chatprotokoll dargestellt, ist visuell klar von einer normalen Antwort unterscheidbar und kann mit `Enter` oder `Esc` geschlossen werden.                                                                                                                                                                           |
| Externe Kanäle    | Wird als klar gekennzeichnete, einmalige Antwort bereitgestellt (Telegram, WhatsApp und Discord verfügen über keine lokale kurzlebige Einblendung).                                                                                                                                  |
| Control UI / Web  | Wird als schwebendes, am Thread angeheftetes Panel „Nebenchat“ dargestellt. Antworten sammeln sich als Gesprächsbeiträge, und über das Eingabefeld „Nachfrage“ kann die nächste Nebenfrage gestellt werden. Durch Schließen (`Esc` oder das X) bleibt das Gespräch erhalten und wird bei der nächsten Antwort erneut geöffnet; die Papierkorb-Schaltfläche verwirft es und beendet einen ausstehenden Durchlauf. |

## Auswahl-Popup (Control UI)

Wenn Text innerhalb einer Chatnachricht in der Control UI markiert wird,
öffnet sich ein kleines Auswahl-Popup mit zwei Aktionen:

- **Weitere Details** sendet sofort eine implizite `/btw`-Frage, mit der
  das Modell aufgefordert wird, den markierten Text im Kontext der aktuellen
  Sitzung zu erläutern. Die Antwort erscheint im schwebenden Nebenchat-Panel.
- **Im Nebenchat fragen** füllt den Eingabebereich mit einem `/btw`-Entwurf vor, der den
  markierten Text zitiert, sodass Sie Ihre eigene Frage dazu eingeben können.

Beide Aktionen folgen der normalen `/btw`-Semantik: Frage und Antwort werden
nicht in den Sitzungsverlauf aufgenommen, und der Hauptdurchlauf bleibt unverändert.

## Verwendung

Verwenden Sie `/btw` für eine kurze Klärung, eine sachliche Nebenantwort, während
ein langer Durchlauf noch läuft, oder eine temporäre Antwort, die nicht in den künftigen
Sitzungskontext aufgenommen werden soll.

```text
/btw welche Datei bearbeiten wir?
/btw fasse die aktuelle Aufgabe in einem Satz zusammen
/btw was ist 17 * 19?
```

Wenn etwas Teil des künftigen Arbeitskontexts der Sitzung werden soll,
fragen Sie stattdessen auf normale Weise in der Hauptsitzung.

## Verwandte Themen

<CardGroup cols={2}>
  <Card title="Slash-Befehle" href="/de/tools/slash-commands" icon="terminal">
    Nativer Befehlskatalog und Chatdirektiven.
  </Card>
  <Card title="Denkstufen" href="/de/tools/thinking" icon="brain">
    Stufen des Denkaufwands für den Modellaufruf der Nebenfrage.
  </Card>
  <Card title="Sitzung" href="/de/concepts/session" icon="comments">
    Sitzungsschlüssel, Verlauf und Persistenzsemantik.
  </Card>
  <Card title="Steuerungsbefehl" href="/de/tools/steer" icon="arrow-right">
    Fügt eine steuernde Nachricht in den aktiven Durchlauf ein, ohne ihn zu beenden.
  </Card>
</CardGroup>
