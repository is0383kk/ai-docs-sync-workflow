---
read_when:
    - Sie möchten Kanalkonten hinzufügen oder entfernen (Discord, Google Chat, iMessage, Matrix, Signal, Slack, Telegram, WhatsApp und weitere)
    - Sie möchten den Kanalstatus prüfen oder Kanalprotokolle fortlaufend anzeigen.
    - Sie müssen ein fehlgeschlagenes eingehendes Kanalereignis prüfen oder erneut übermitteln
summary: CLI-Referenz für `openclaw channels` (Konten, Status, unzustellbare Nachrichten, Funktionen, Auflösung, Protokolle, Anmeldung/Abmeldung)
title: Kanäle
x-i18n:
    generated_at: "2026-07-26T18:51:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8e5b7d674264af51d6fec34c8c95256129d66918b7c4515ac0f2c2bd311f2c3b
    source_path: cli/channels.md
    workflow: 16
---

# `openclaw channels`

Verwalten Sie Chatkanalkonten und deren Laufzeitstatus auf dem Gateway.

Zugehörige Dokumentation:

- Kanalleitfäden: [Kanäle](/de/channels)
- Gateway-Konfiguration: [Konfiguration](/de/gateway/configuration)

## Häufig verwendete Befehle

```bash
openclaw channels list
openclaw channels list --all
openclaw channels status
openclaw channels capabilities
openclaw channels capabilities --channel discord --target channel:123
openclaw channels resolve --channel slack "#general" "@jane"
openclaw channels logs --channel all
openclaw channels dead-letters list --channel telegram --account default
```

`channels list` zeigt nur Chatkanäle an: standardmäßig konfigurierte Konten mit den Status-Tags `installed`, `configured` und `enabled` pro Konto (`--json` für maschinenlesbare Ausgabe). Übergeben Sie `--all`, um auch gebündelte Kanäle anzuzeigen, für die noch kein Konto konfiguriert ist, sowie installierbare Katalogkanäle, die sich noch nicht auf dem Datenträger befinden. Provider-Authentifizierung und Modellnutzung werden an anderer Stelle verwaltet: `openclaw models auth list` für Provider-Authentifizierungsprofile, `openclaw status` oder `openclaw models list` für Nutzung/Kontingent.

## Status / Fähigkeiten / Auflösung / Protokolle

- `channels status`: `--channel <name>`, `--probe`, `--timeout <ms>` (Standard: `10000`), `--json`
- `channels capabilities`: `--channel <name>`, `--account <id>` (erfordert `--channel`), `--target <dest>` (erfordert `--channel`), `--timeout <ms>` (Standard: `10000`, begrenzt auf `30000`), `--json`
- `channels resolve <entries...>`: `--channel <name>`, `--account <id>`, `--kind <auto|user|group>` (Standard: `auto`), `--json`
- `channels logs`: `--channel <name|all>` (Standard: `all`), `--lines <n>` (Standard: `200`), `--json`

`channels status --probe` ist der Live-Pfad: Auf einem erreichbaren Gateway führt er pro Konto
`probeAccount` und optionale `auditAccount`-Prüfungen aus, sodass die Ausgabe den Transportstatus
sowie Prüfergebnisse wie `works`, `probe failed`, `audit ok` oder `audit failed` enthalten kann.
Wenn das Gateway nicht erreichbar ist, greift `channels status` statt einer Ausgabe von Live-Prüfungen
auf reine Konfigurationszusammenfassungen zurück.

## Eingehende Dead Letters

Eingehende Ereignisse, die ihre Wiederholungsrichtlinie ausschöpfen, verbleiben für den bestehenden Aufbewahrungszeitraum fehlgeschlagener Warteschlangeneinträge in der gemeinsamen Zustandsdatenbank. Prüfen Sie ein Kanalkonto mit:

```bash
openclaw channels dead-letters list --channel telegram --account default
openclaw channels dead-letters list --channel telegram --account default --json
```

Die Textansicht zeigt Ereignis-IDs, Fehlerursachen, die Anzahl der Versuche und das Alter der Fehler. Die JSON-Ausgabe enthält zur Diagnose außerdem die aufbewahrte Nutzlast, Metadaten, Lane und Zeitstempel der Versuche.

Nachdem Sie das zugrunde liegende Problem behoben haben, reihen Sie ein Ereignis mit seiner ursprünglichen Ereignis-ID erneut ein:

```bash
openclaw channels dead-letters resubmit <event-id> --channel telegram --account default
```

Führen Sie diese Befehle auf dem Gateway-Host aus, damit sie auf dieselbe gemeinsame Zustandsdatenbank wie die Kanallaufzeit zugreifen. Die erneute Einreihung erhält Nutzlast, Metadaten und Lane, setzt jedoch den Versuchszähler und das Warteschlangenalter zurück. Sie ersetzt die Fehlermarkierung dieses Ereignisses atomar. Daher wird eine Wiederholung des Befehls abgelehnt, solange das Ereignis aussteht oder beansprucht ist, anstatt eine zweite Zustellung zu erstellen. Der laufende Kanal übernimmt es bei der nächsten Verarbeitung eingehender Ereignisse. Abgeschlossene Ereignisse bleiben endgültig und können nicht erneut eingereiht werden. Fehlgeschlagene Zeilen, die vor Einführung der Nutzlastaufbewahrung erstellt wurden, können weiterhin in der Liste erscheinen. Ihre erneute Einreihung wird jedoch abgelehnt, da ihre Nutzlast nicht verfügbar ist.

`openclaw health` meldet pro Kanalkonto die Anzahl der Dead Letters und das Alter des ältesten Fehlers. `openclaw doctor` nennt betroffene Konten und verweist auf den Prüfungsbefehl.

Verwenden Sie `openclaw sessions`, Gateway-`sessions.list` oder das Agent-Tool
`sessions_list` nicht als Signal für den Zustand des Kanalsockets. Diese Oberflächen melden
gespeicherte Konversationszeilen und nicht den Laufzeitstatus des Providers. Nach einem Neustart des Discord-Providers
kann ein verbundenes, aber inaktives Konto fehlerfrei sein, obwohl keine Discord-Sitzungszeile
erscheint, bis das nächste ein- oder ausgehende Konversationsereignis eintritt.

## Konten hinzufügen/entfernen

```bash
openclaw channels add --channel telegram --token <bot-token>
openclaw channels add --channel nostr --private-key "$NOSTR_PRIVATE_KEY"
openclaw channels remove --channel telegram --delete
```

<Tip>
`openclaw channels add telegram --help` oder `openclaw channels add --channel telegram --help` zeigt nur die Einrichtungs-Flags von Telegram an. `openclaw channels add --help` zeigt nur die gemeinsame Befehlshülle an.
</Tip>

`channels remove` arbeitet nur mit installierten/konfigurierten Kanal-Plugins. Verwenden Sie zuerst `channels add` für installierbare Katalogkanäle. Ohne `--delete` werden Sie gefragt, ob das Konto deaktiviert werden soll, und seine Konfiguration bleibt erhalten; `--delete` entfernt die Konfigurationseinträge ohne Rückfrage.
Bei laufzeitgestützten Kanal-Plugins fordert `channels remove` außerdem das laufende Gateway auf, das ausgewählte Konto zu stoppen, bevor die Konfiguration aktualisiert wird. Dadurch bleibt der alte Listener nach dem Deaktivieren oder Löschen eines Kontos nicht bis zum Neustart aktiv.

Die gemeinsame Steuerungshülle enthält nur `--channel`, `--account` und die optionale Kontoanzeige `--name`. Jedes moderne Kanal-Plugin verwaltet seine eigenen Anmeldedaten sowie seine transport- und providerspezifische Semantik. Sobald ein Kanal anhand einer Positions-ID oder mit `--channel <id>` ausgewählt wurde, erstellt die CLI ausschließlich die Optionen dieses Kanals aus den Paketmetadaten des gebündelten oder installierten Plugins, ohne Kanallaufzeitcode zu laden.

Allgemein wirkende Flags wie `--token`, `--url` oder `--use-env` gehören weiterhin dem Kanal, wenn sie von einem modernen Vertrag verarbeitet werden. Wenn ein ausgewähltes Drittanbieter-Plugin noch den älteren gemeinsamen Einrichtungsadapter verwendet, registriert der Kern ausschließlich für diesen Kanal den veröffentlichten Satz an Kompatibilitäts-Flags zusammen mit dessen älterem `cliAddOptions`. Nicht zugehörige ältere Felder gelangen nicht in andere Kanäle, und ein ausgewählter moderner Kanal lehnt Kompatibilitäts-Flags ab, die er nicht deklariert hat.

Beispiele für kanaleigene Flags:

| Kanal       | Flags                                                                                                |
| ----------- | ---------------------------------------------------------------------------------------------------- |
| Google Chat | `--webhook-path`, `--webhook-url`, `--audience-type`, `--audience`                                   |
| iMessage    | `--cli-path`, `--db-path`, `--service`, `--region`                                                   |
| Matrix      | `--homeserver`, `--user-id`, `--access-token`, `--password`, `--device-name`, `--initial-sync-limit` |
| Nostr       | `--private-key`, `--relay-urls`                                                                      |
| Signal      | `--signal-number`, `--signal-transport`, `--cli-path`, `--http-url`, `--http-host`, `--http-port`    |
| Tlon        | `--ship`, `--url`, `--code`, `--group-channels`, `--dm-allowlist`, `--auto-discover-channels`        |
| WhatsApp    | `--auth-dir`                                                                                         |

Wenn ein Kanal-Plugin während eines Flag-gesteuerten Hinzufügebefehls installiert werden muss, verwendet OpenClaw die Standardinstallationsquelle des Kanals, ohne die interaktive Aufforderung zur Plugin-Installation zu öffnen.

Sowohl die geführte als auch die Flag-gesteuerte Einrichtung durchlaufen den Parser, die Validierung, die Kontoauflösung, den Konfigurationsschreiber und die Hooks nach dem Schreiben des ausgewählten Kanals. Nicht unterstützte Flags schlagen mit dem Einrichtungsfehler des zuständigen Kanals fehl, statt über eine globale Eingabesammlung akzeptiert zu werden.

Wenn Sie `openclaw channels add` ohne direkte Konto-, Anmeldedaten- oder Kanalkonfigurations-Flags ausführen, kann der interaktive Assistent Eingaben abfragen. Sowohl eine Kanal-ID als Positionsargument als auch `--channel <id>` wählen diesen Kanal vorab aus, ohne die Anleitung zu umgehen:

```bash
openclaw channels add telegram
openclaw channels add --channel telegram
```

Der Assistent kann Folgendes abfragen:

- Konto-IDs pro ausgewähltem Kanal
- optionale Anzeigenamen für diese Konten
- `Route these channel accounts to agents now?`

Wenn Sie die sofortige Bindung bestätigen, fragt der Assistent, welcher Agent jedes konfigurierte Kanalkonto verwalten soll, und schreibt kontobezogene Routing-Bindungen.

Dieselben Routing-Regeln können Sie später auch mit `openclaw agents bindings`, `openclaw agents bind` und `openclaw agents unbind` verwalten (siehe [Agenten](/de/cli/agents)).

Wenn Sie einem Kanal, der noch kontenübergreifende Einstellungen der obersten Ebene für ein einzelnes Konto verwendet, ein Konto hinzufügen, das nicht das Standardkonto ist, überführt OpenClaw diese Werte der obersten Ebene in die Kontenzuordnung des Kanals, bevor es das neue Konto schreibt. Bei der Überführung wird ein vorhandenes benanntes Konto wiederverwendet, wenn der Kanal genau eines besitzt oder wenn `defaultAccount` auf eines verweist. Andernfalls werden die Werte in `channels.<channel>.accounts.default` abgelegt.

Das Routing-Verhalten bleibt konsistent:

- Bestehende reine Kanalbindungen (ohne `accountId`) stimmen weiterhin mit dem Standardkonto überein.
- `channels add` erstellt oder überschreibt Bindungen im nicht interaktiven Modus nicht automatisch.
- Die interaktive Einrichtung kann optional kontobezogene Bindungen hinzufügen.

Wenn Ihre Konfiguration bereits einen gemischten Zustand aufwies (benannte Konten waren vorhanden und Werte der obersten Ebene für ein einzelnes Konto weiterhin gesetzt), führen Sie `openclaw doctor --fix` aus, um kontobezogene Werte in das für diesen Kanal ausgewählte überführte Konto zu verschieben.

## An- und Abmeldung (interaktiv)

```bash
openclaw channels login --channel whatsapp
openclaw channels logout --channel whatsapp
```

- `channels login` unterstützt `--account <id>` und `--verbose`; `channels logout` unterstützt `--account <id>`.
- `channels login` und `logout` können den Kanal ableiten, wenn nur ein konfigurierter Kanal diese Aktion unterstützt. Bei mehreren Kanälen übergeben Sie `--channel`.
- `channels logout` bevorzugt den Live-Gateway-Pfad, wenn dieser erreichbar ist, sodass bei der Abmeldung alle aktiven Listener gestoppt werden, bevor der Authentifizierungsstatus des Kanals gelöscht wird. Wenn kein lokales Gateway erreichbar ist, wird ersatzweise eine lokale Bereinigung der Authentifizierung durchgeführt; mit `gateway.mode: "remote"` führt der Gateway-Fehler stattdessen zum Fehlschlagen des Befehls.
- Nach einer erfolgreichen Anmeldung fordert die CLI ein erreichbares lokales Gateway auf, das Konto zu starten. Im Remote-Modus speichert sie die Authentifizierung lokal und weist darauf hin, dass die entfernte Laufzeit nicht neu gestartet wurde.
- Führen Sie `channels login` in einem Terminal auf dem Gateway-Host aus. Agent-`exec` blockiert diesen interaktiven Anmeldeablauf. Kanalnative Agent-Anmeldetools wie `whatsapp_login` sollten, sofern verfügbar, aus dem Chat heraus verwendet werden.

## Fehlerbehebung

- Führen Sie `openclaw status --deep` für eine umfassende Prüfung aus.
- Verwenden Sie `openclaw doctor` für geführte Fehlerbehebungen.
- `openclaw channels status` greift auf reine Konfigurationszusammenfassungen zurück, wenn das Gateway nicht erreichbar ist. Wenn die Anmeldedaten eines unterstützten Kanals über SecretRef konfiguriert, im aktuellen Befehlspfad jedoch nicht verfügbar sind, wird das Konto mit Hinweisen auf den eingeschränkten Zustand als konfiguriert gemeldet, anstatt es als nicht konfiguriert anzuzeigen.

## Fähigkeitsprüfung

Rufen Sie Hinweise zu den Fähigkeiten des Providers (Intents/Berechtigungsbereiche, sofern verfügbar) sowie die statische Funktionsunterstützung ab:

```bash
openclaw channels capabilities
openclaw channels capabilities --channel discord --target channel:123
```

Hinweise:

- `--channel` ist optional; lassen Sie es weg, um alle Kanäle aufzulisten (einschließlich der von Plugins bereitgestellten Kanäle).
- `--account` ist nur zusammen mit `--channel` gültig.
- `--target` akzeptiert `channel:<id>` oder eine unverarbeitete numerische Kanal-ID und gilt nur für Discord. Bei Discord-Sprachkanälen kennzeichnet die Berechtigungsprüfung fehlende `ViewChannel`, `Connect`, `Speak`, `SendMessages` und `ReadMessageHistory`.
- Prüfungen sind Provider-spezifisch: Discord-Bot-Identität und -Intents sowie optionale Kanalberechtigungen; Slack-Bot- und Benutzer-Scopes; Telegram-Bot-Flags und Webhook; Signal-Daemon-Version; Microsoft-Teams-App-Token und Graph-Rollen/-Scopes (soweit bekannt, mit Anmerkungen). Kanäle ohne Prüfungen melden `Probe: unavailable`.

## Namen in IDs auflösen

Lösen Sie Kanal- und Benutzernamen mithilfe des Provider-Verzeichnisses in IDs auf:

```bash
openclaw channels resolve --channel slack "#general" "@jane"
openclaw channels resolve --channel discord "My Server/#support" "@someone"
openclaw channels resolve --channel matrix "Project Room"
```

Hinweise:

- Verwenden Sie `--kind user|group|auto`, um den Zieltyp zu erzwingen.
- Wenn mehrere Einträge denselben Namen haben, werden bei der Auflösung aktive Übereinstimmungen bevorzugt.
- `channels resolve` ist schreibgeschützt. Wenn ein ausgewähltes Konto über SecretRef konfiguriert ist, diese Anmeldeinformation im aktuellen Befehlspfad jedoch nicht verfügbar ist, gibt der Befehl eingeschränkte, nicht aufgelöste Ergebnisse mit Hinweisen zurück, statt den gesamten Durchlauf abzubrechen.
- `channels resolve` installiert keine Kanal-Plugins. Verwenden Sie `channels add --channel <name>`, bevor Sie Namen für einen installierbaren Katalogkanal auflösen.

## Verwandte Themen

- [CLI-Referenz](/de/cli)
- [Kanalübersicht](/de/channels)
