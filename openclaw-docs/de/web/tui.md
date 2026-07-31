---
read_when:
    - Sie wünschen eine anfängerfreundliche Einführung in die TUI
    - Sie benötigen die vollständige Liste der TUI-Funktionen, -Befehle und -Tastenkürzel
summary: 'Terminal-Benutzeroberfläche (TUI): Verbindung mit dem Gateway herstellen oder lokal im eingebetteten Modus ausführen'
title: TUI
x-i18n:
    generated_at: "2026-07-26T18:43:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: dc4dc5e2a408b5097b3615283b5a4590e8b55bccb15c26d8e38ab2c84b902f4a
    source_path: web/tui.md
    workflow: 16
---

## Schnellstart

### Gateway-Modus

1. Starten Sie den Gateway.

```bash
openclaw gateway
```

2. Öffnen Sie die TUI.

```bash
openclaw tui
```

3. Geben Sie eine Nachricht ein und drücken Sie die Eingabetaste.

Entfernter Gateway:

```bash
openclaw tui --url ws://<host>:<port> --token <gateway-token>
```

Verwenden Sie `--password`, wenn Ihr Gateway die Passwortauthentifizierung verwendet.

### Lokaler Modus

Führen Sie die TUI ohne Gateway aus:

```bash
openclaw chat
# oder
openclaw tui --local
```

- `openclaw chat` und `openclaw terminal` sind Aliasse für `openclaw tui --local`.
- `--local` kann nicht mit `--url`, `--token` oder `--password` kombiniert werden.
- Der lokale Modus verwendet direkt die eingebettete Agent-Laufzeit. Die meisten lokalen Tools funktionieren, aber Funktionen, die ausschließlich im Gateway verfügbar sind, stehen nicht zur Verfügung.
- Der alleinstehende Aufruf `openclaw` (ohne Unterbefehl) wählt automatisch ein Ziel aus: Bei einer nicht konfigurierten Installation wird das Inferenz-Onboarding ausgeführt; bei einer ungültigen Konfiguration werden die klassischen Doctor-Hinweise geöffnet; bei einem erreichbaren konfigurierten Gateway wird diese TUI-Shell im Gateway-Modus geöffnet; andernfalls wird sie mit einem konfigurierten lokalen Modell im lokalen Modus geöffnet.

## Anzeige

- Kopfzeile: Verbindungs-URL, aktueller Agent, aktuelle Sitzung.
- Chatprotokoll: Benutzernachrichten, Antworten des Assistenten, Systemhinweise, Tool-Karten.
- Statuszeile: Verbindungs-/Ausführungsstatus (Verbindung wird hergestellt, wird ausgeführt, Streaming, inaktiv, Fehler).
- Fußzeile: Agent + Sitzung + Modell + Zielstatus + Denken/schnell/ausführlich/Trace/Reasoning + Token-Anzahlen + Zustellung.
- Eingabe: Texteditor mit automatischer Vervollständigung.

## Grundkonzept: Agenten + Sitzungen

- Agenten besitzen eindeutige Slugs (z. B. `main`, `research`). Der Gateway stellt die Liste bereit.
- Sitzungen gehören zum aktuellen Agenten.
- Sitzungsschlüssel werden als `agent:<agentId>:<sessionKey>` gespeichert.
  - Wenn Sie `/session main` eingeben, erweitert die TUI dies zu `agent:<currentAgent>:main`.
  - Wenn Sie `/session agent:other:main` eingeben, wechseln Sie ausdrücklich zu dieser Agentensitzung.
- Sitzungsumfang:
  - `per-sender` (Standard): Jeder Agent verfügt über mehrere Sitzungen.
  - `global`: Die TUI verwendet immer die Sitzung `global` (die Auswahlliste kann leer sein).
- Der aktuelle Agent und die aktuelle Sitzung sind immer in der Fußzeile sichtbar.
- Wenn die Sitzung ein [Ziel](/de/tools/goal) besitzt, zeigt die Fußzeile dessen kompakten Status an:
  `Pursuing goal`, `Goal paused (/goal resume)`, `Goal blocked (/goal resume)` oder `Goal achieved`.
- Wird die TUI im Gateway-Modus ohne `--session` gestartet, setzt sie die zuletzt ausgewählte Sitzung für denselben Gateway, Agenten und Sitzungsumfang fort, sofern diese Sitzung noch vorhanden ist. Die Übergabe von `--session`, `/session`, `/new` oder `/reset` bleibt ausdrücklich.

## Senden + Zustellung

- Nachrichten werden immer an den Gateway gesendet (oder im lokalen Modus an die eingebettete Laufzeit); die Antwort des Assistenten anschließend an einen Chat-Provider zuzustellen, ist ein separater Schritt, der standardmäßig deaktiviert ist.
- Die TUI ist wie WebChat eine interne Quelloberfläche und kein generischer ausgehender Kanal. Harnesses, die `tools.message` für sichtbare Antworten benötigen, können den aktiven TUI-Zug mit einem ziellosen `message.send` erfüllen; die explizite Zustellung über einen Provider verwendet weiterhin die normal konfigurierten Kanäle und greift niemals auf `lastChannel` zurück.
- Die Zustellung wird beim Start für die gesamte TUI-Sitzung festgelegt: Starten Sie sie mit `openclaw tui --deliver`, um die Zustellung zu aktivieren. Es gibt keinen Slash-Befehl `/deliver` und keinen Schalter in den Einstellungen, um sie während der Sitzung umzuschalten; starten Sie die TUI neu, um die Einstellung zu ändern.

## Auswahllisten + Overlays

- Modellauswahl: Listet verfügbare Modelle auf und legt die Sitzungsüberschreibung fest.
- Agentenauswahl: Wählen Sie einen anderen Agenten aus.
- Sitzungsauswahl: Zeigt bis zu 50 Sitzungen des aktuellen Agenten an, die innerhalb der letzten 7 Tage aktualisiert wurden. Verwenden Sie `/session <key>`, um zu einer älteren bekannten Sitzung zu wechseln.
- Einstellungen (`/settings`): Schalten Sie die Erweiterung der Tool-Ausgabe und die Sichtbarkeit des Denkprozesses um. Dieses Panel steuert nicht die Zustellung.

## Tastenkombinationen

- Eingabetaste: Nachricht senden
- Esc: Aktive Ausführung abbrechen
- Strg+C: Eingabe löschen (zum Beenden zweimal drücken)
- Strg+D: Beenden
- Strg+L: Modellauswahl
- Strg+G: Agentenauswahl
- Strg+P: Sitzungsauswahl
- Strg+O: Erweiterung der Tool-Ausgabe umschalten
- Strg+T: Sichtbarkeit des Denkprozesses umschalten (lädt den Verlauf neu)

## Slash-Befehle

Kernfunktionen:

- `/help`
- `/status` (an den Gateway weitergeleitet; zeigt eine Sitzungs-/Modellzusammenfassung)
- `/gateway-status` (Alias `/gwstatus`; zeigt direkt den Status der Gateway-Verbindung)
- `/agent <id>` (oder `/agents`)
- `/session <key>` (oder `/sessions`)
- `/model <provider/model>` (oder `/models`)

Sitzungssteuerung:

- `/think <off|minimal|low|medium|high>` (höhere Stufen können je nach Modell Ebenen wie `xhigh`/`max` hinzufügen)
- `/fast <status|auto|on|off>`
- `/verbose <on|full|off>`
- `/trace <on|off>`
- `/reasoning <on|off|stream>`
- `/usage <off|tokens|full|reset>` (`reset`/`inherit`/`clear`/`default` löscht die Sitzungsüberschreibung)
- `/goal [status] | /goal start <objective> | /goal edit <objective> | /goal pause|resume|complete|block|clear`
- `/elevated <on|off|ask|full>` (Alias: `/elev`)
- `/activation <mention|always>`
- `/queue <steer|followup|collect|interrupt> [debounce:<duration>] [cap:<n>] [drop:<summarize|old|new>]`
- `/queue default` (oder `/queue reset`) löscht die Sitzungsüberschreibung

Sitzungslebenszyklus:

- `/new` (erstellt eine neue, isolierte Sitzung unter einem neuen Schlüssel; andere TUI-Clients in der alten Sitzung bleiben unberührt)
- `/reset` (setzt den aktuellen Sitzungsschlüssel direkt zurück)
- `/abort` (bricht die aktive Ausführung ab)
- `/settings`
- `/exit` (oder `/quit`)

Nur im lokalen Modus:

- `/auth [provider]` öffnet den Authentifizierungs-/Anmeldeablauf des Providers innerhalb der TUI.

Der lokale Modus implementiert dieselben Warteschlangenmodi innerhalb der eingebetteten Laufzeit. Eine
während der Ausführung eingegebene Anfrage folgt der Richtlinie `/queue` der Sitzung: `steer` fügt sie ein, wenn die
Laufzeit sie annehmen kann, `followup` wartet auf einen separaten Zug, `collect` fasst
ausstehende Anfragen zusammen und `interrupt` beendet die aktuelle Ausführung, bevor die neue
gestartet wird. Das explizite `/steer <message>` ist ausschließlich im Gateway verfügbar; verwenden Sie im lokalen Modus `/queue steer` zusammen mit einer
normalen Nachricht.

OpenClaw:

- `/openclaw [request]` führt von der normalen Agenten-TUI zurück zum Einrichtungs-/Reparaturchat [OpenClaw](#openclaw-setup-and-repair-helper) und leitet optional eine Anfrage weiter.

Andere Gateway-Slash-Befehle (beispielsweise `/context`) werden an den Gateway weitergeleitet und als Systemausgabe angezeigt. Siehe [Slash-Befehle](/de/tools/slash-commands).

## Lokale Shell-Befehle

- Stellen Sie einer Zeile `!` voran, um einen lokalen Shell-Befehl auf dem TUI-Host auszuführen.
- Die TUI fragt einmal pro Sitzung nach der Erlaubnis zur lokalen Ausführung; bei Ablehnung bleibt `!` für die Sitzung deaktiviert.
- Befehle werden in einer neuen, nicht interaktiven Shell im Arbeitsverzeichnis der TUI ausgeführt (kein persistentes `cd`/keine persistente Umgebung).
- Lokale Shell-Befehle erhalten `OPENCLAW_SHELL=tui-local` in ihrer Umgebung.
- Ein alleinstehendes `!` wird als normale Nachricht gesendet; führende Leerzeichen lösen keine lokale Ausführung aus.

## OpenClaw-Helfer für Einrichtung und Reparatur

OpenClaw ist der Ring-0-Assistent für Einrichtung und Reparatur. Er wird als `openclaw setup` bereitgestellt, nachdem das konfigurierte Standardmodell eine Live-Inferenzprüfung bestanden hat. Wenn keine Inferenz verfügbar ist, kehrt ein interaktiver Aufruf zum Inferenz-Onboarding zurück und die Automatisierung schlägt mit Reparaturhinweisen fehl. Er wird innerhalb derselben lokalen TUI-Shell wie `openclaw tui --local` ausgeführt und verwendet einen KI-Agenten, der auf die typisierten und genehmigungspflichtigen OpenClaw-Operationen beschränkt ist:

```bash
openclaw setup                       # interaktiv starten
openclaw setup -m "status"           # eine Anfrage ausführen und beenden
openclaw setup -m "set default model openai/gpt-5.2" --yes   # eine Konfigurationsänderung anwenden
```

- Persistente Konfigurationsänderungen benötigen eine Genehmigung: Bestätigen Sie entweder interaktiv oder übergeben Sie `--yes`.
- `--json` gibt die Startübersicht als JSON aus, anstatt den Chat zu starten.
- Innerhalb von OpenClaw beendet eine `open-tui`-Anfrage (beispielsweise die Bitte, mit einem normalen Agenten zu sprechen) OpenClaw und öffnet die reguläre Agenten-TUI; verwenden Sie dort `/openclaw`, um zurückzukehren.

Verwenden Sie den lokalen Modus, wenn die aktuelle Konfiguration bereits gültig ist und der eingebettete Agent sie auf demselben Rechner untersuchen, mit der Dokumentation vergleichen und bei der Behebung von Abweichungen helfen soll, ohne von einem laufenden Gateway abhängig zu sein.

Wenn `openclaw config validate` bereits fehlschlägt, beginnen Sie zunächst mit `openclaw configure` oder `openclaw doctor --fix`; `openclaw chat` benötigt zum Starten weiterhin eine ladbare Konfiguration.

Typischer Ablauf:

1. Starten Sie den lokalen Modus:

```bash
openclaw chat
```

2. Teilen Sie dem Agenten mit, was geprüft werden soll, beispielsweise:

```text
Vergleiche meine Gateway-Authentifizierungskonfiguration mit der Dokumentation und schlage die kleinste Korrektur vor.
```

3. Verwenden Sie lokale Shell-Befehle für genaue Nachweise und die Validierung:

```text
!openclaw config file
!openclaw docs gateway auth token secretref
!openclaw config validate
!openclaw doctor
```

4. Wenden Sie gezielte Änderungen mit `openclaw config set` oder `openclaw configure` an und führen Sie anschließend `!openclaw config validate` erneut aus.
5. Wenn Doctor eine automatische Migration oder Reparatur empfiehlt, prüfen Sie diese und führen Sie `!openclaw doctor --fix` aus.

Tipps:

- Verwenden Sie vorzugsweise `openclaw config set` oder `openclaw configure`, statt `openclaw.json` manuell zu bearbeiten.
- `openclaw docs "<query>"` durchsucht den aktuellen Dokumentationsindex auf demselben Rechner.
- `openclaw config validate --json` ist nützlich, wenn Sie strukturierte Schema- sowie SecretRef-/Auflösbarkeitsfehler benötigen.

## Tool-Ausgabe

- Tool-Aufrufe werden als Karten mit Argumenten und Ergebnissen angezeigt.
- Strg+O schaltet zwischen der eingeklappten und der erweiterten Ansicht um.
- Während Tools ausgeführt werden, erscheinen Teilaktualisierungen fortlaufend in derselben Karte.

## Terminalfarben

- Die TUI zeigt den Fließtext des Assistenten in der Standardvordergrundfarbe Ihres Terminals an, damit er sowohl in dunklen als auch in hellen Terminals lesbar bleibt.
- Wenn Ihr Terminal einen hellen Hintergrund verwendet und die automatische Erkennung fehlerhaft ist, setzen Sie vor dem Start von `openclaw tui` die Variable `OPENCLAW_THEME=light`.
- Um stattdessen die ursprüngliche dunkle Farbpalette zu erzwingen, setzen Sie `OPENCLAW_THEME=dark`.

## Verlauf + Streaming

- Beim Verbindungsaufbau lädt die TUI den neuesten Verlauf (standardmäßig 200 Nachrichten).
- Streaming-Antworten werden bis zu ihrem Abschluss direkt aktualisiert.
- Die TUI überwacht außerdem Tool-Ereignisse des Agenten, um umfangreichere Tool-Karten anzuzeigen.

## Verbindungsdetails

- Die TUI stellt die Verbindung mit der Client-ID `openclaw-tui` im allgemeinen Client-Modus `ui` her (demselben Modus, den Control UI und WebChat für Gateway-Richtlinien verwenden).
- Bei erneuten Verbindungen wird eine Systemmeldung angezeigt; Ereignislücken werden im Protokoll ausgewiesen.

## Optionen

- `--local`: Mit der lokalen eingebetteten Agent-Laufzeit ausführen
- `--url <url>`: Gateway-WebSocket-URL (standardmäßig `gateway.remote.url` aus der Konfiguration oder `ws://127.0.0.1:<port>` auf der Loopback-Schnittstelle)
- `--token <token>`: Gateway-Token (falls erforderlich)
- `--password <password>`: Gateway-Passwort (falls erforderlich)
- `--tls-fingerprint <sha256>`: Erwarteter TLS-Zertifikatfingerabdruck für ein angeheftetes `wss://`-Gateway
- `--session <key>`: Sitzungsschlüssel (Standard: `main` oder `global`, wenn der Geltungsbereich global ist)
- `--deliver`: Antworten des Assistenten an den Provider übermitteln (standardmäßig deaktiviert)
- `--thinking <level>`: Denkstufe für Sendevorgänge überschreiben
- `--message <text>`: Nach dem Verbindungsaufbau eine erste Nachricht senden
- `--timeout-ms <ms>`: Agent-Zeitüberschreitung in ms (standardmäßig `agents.defaults.timeoutSeconds`)
- `--history-limit <n>`: Anzahl zu ladender Verlaufseinträge (Standard: `200`)

<Warning>
Wenn Sie `--url` festlegen, greift die TUI nicht ersatzweise auf Anmeldedaten aus der Konfiguration oder der Umgebung zurück. Übergeben Sie `--token` oder `--password` explizit sowie `--tls-fingerprint`, wenn das Ziel ein angeheftetes Zertifikat verwendet. Fehlende explizite Anmeldedaten führen zu einem Fehler. Übergeben Sie im lokalen Modus nicht `--url`, `--token`, `--password` oder `--tls-fingerprint`.
</Warning>

## Fehlerbehebung

Keine Ausgabe nach dem Senden einer Nachricht:

- Führen Sie `/status` in der TUI aus, um zu bestätigen, dass das Gateway verbunden und im Leerlauf/beschäftigt ist.
- Prüfen Sie die Gateway-Protokolle: `openclaw logs --follow`.
- Bestätigen Sie, dass der Agent ausgeführt werden kann: `openclaw status` und `openclaw models status`.
- Wenn Sie Nachrichten in einem Chatkanal erwarten, bestätigen Sie, dass die TUI mit `--deliver` gestartet wurde (dies kann später nicht ohne Neustart aktiviert werden).

## Fehlerbehebung bei Verbindungen

- `disconnected`: Stellen Sie sicher, dass das Gateway ausgeführt wird und Ihre `--url/--token/--password` korrekt sind.
- Keine Agents in der Auswahl: Prüfen Sie `openclaw agents list` und Ihre Routing-Konfiguration.
- Leere Sitzungsauswahl: Möglicherweise befinden Sie sich im globalen Geltungsbereich oder es sind noch keine Sitzungen vorhanden.

## Verwandte Themen

- [Steuerungsoberfläche](/de/web/control-ui) — webbasierte Steuerungsoberfläche
- [Konfiguration](/de/cli/config) — `openclaw.json` prüfen, validieren und bearbeiten
- [Doctor](/de/cli/doctor) — geführte Reparatur- und Migrationsprüfungen
- [CLI-Referenz](/de/cli) — vollständige Referenz der CLI-Befehle
