---
read_when:
    - Planung eines Wechsels von BlueBubbles zum gebündelten iMessage-Plugin
    - BlueBubbles-Konfigurationsschlüssel in iMessage-Entsprechungen übersetzen
    - imsg vor dem Aktivieren des iMessage-Plugins überprüfen
summary: 'Migrieren Sie alte BlueBubbles-Konfigurationen zum gebündelten iMessage-Plugin: Schlüsselzuordnung, Prüfungen der Gruppen-Zulassungsliste und Verifizierung der Umstellung.'
title: Von BlueBubbles kommend
x-i18n:
    generated_at: "2026-07-26T17:38:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5984ad1319b4bb3060496666bea6de663eba0105a89f82d13030c015c5df159d
    source_path: channels/imessage-from-bluebubbles.md
    workflow: 16
---

Die Unterstützung für BlueBubbles wurde entfernt. OpenClaw unterstützt iMessage nur über das gebündelte `imessage`-Plugin, das [`steipete/imsg`](https://github.com/steipete/imsg) über JSON-RPC steuert und auf dieselbe private API-Oberfläche zugreift wie zuvor BlueBubbles (`react`, `edit`, `unsend`, `reply`, `sendWithEffect`, native Umfragen, Gruppenverwaltung, Anhänge). Ein einziges CLI-Binärprogramm ersetzt den BlueBubbles-Server, die Client-App und die Webhook-Infrastruktur: kein REST-Endpunkt, keine Webhook-Authentifizierung.

Diese Anleitung migriert alte `channels.bluebubbles`-Konfigurationen zu `channels.imessage`. Es gibt keinen anderen unterstützten Migrationspfad. In der aktuellen OpenClaw-Version ist ein verbliebener `channels.bluebubbles`-Block wirkungslos – keine Laufzeitkomponente liest ihn.

<Note>
Eine kurze Ankündigung und eine Zusammenfassung für Betreiber finden Sie unter [Entfernung von BlueBubbles und der imsg-Pfad für iMessage](/de/announcements/bluebubbles-imessage).
</Note>

## Checkliste für die Migration

Der kürzeste sichere Weg, wenn Ihre alte BlueBubbles-Konfiguration bereits bekannt ist:

1. Überprüfen Sie `imsg` direkt auf dem Mac, auf dem Messages.app ausgeführt wird (`imsg chats`, `imsg history`, `imsg send`, `imsg rpc --help`).
2. Kopieren Sie die Verhaltensschlüssel von `channels.bluebubbles` nach `channels.imessage`: `dmPolicy`, `allowFrom`, `groupPolicy`, `groupAllowFrom`, `groups`, `includeAttachments`, `attachmentRoots`, `mediaMaxMb`, `textChunkLimit` und `actions`.
3. Entfernen Sie nicht mehr vorhandene Transportschlüssel: `serverUrl`, `password`, Webhook-URLs und die Einrichtung des BlueBubbles-Servers.
4. Wenn der Gateway nicht auf dem Messages-Mac ausgeführt wird, legen Sie `channels.imessage.cliPath` auf einen SSH-Wrapper fest und konfigurieren Sie `remoteHost` für den Remote-Abruf von Anhängen.
5. Aktivieren Sie `channels.imessage`, starten Sie den Gateway neu und führen Sie anschließend `openclaw channels status --probe --channel imessage` aus.
6. Testen Sie eine Direktnachricht, eine zulässige Gruppe, Anhänge, sofern aktiviert, und jede private API-Aktion, die der Agent voraussichtlich verwenden wird.
7. Löschen Sie den BlueBubbles-Server und die alte `channels.bluebubbles`-Konfiguration, nachdem der iMessage-Pfad überprüft wurde.

## Funktionsweise von imsg

`imsg` ist eine lokale macOS-CLI für Messages. OpenClaw startet `imsg rpc` als untergeordneten Prozess und kommuniziert über stdin/stdout mittels JSON-RPC. Es gibt keinen HTTP-Server, keine Webhook-URL, keinen Hintergrund-Daemon, keinen Launch Agent und keinen offenzulegenden Port.

- Lesezugriffe erfolgen aus `~/Library/Messages/chat.db` über einen schreibgeschützten SQLite-Handle.
- Eingehende Live-Nachrichten stammen aus `imsg watch` / `watch.subscribe`, das `chat.db`-Dateisystemereignissen folgt und ersatzweise Polling verwendet.
- Zum normalen Senden von Texten und Dateien wird die Automatisierung von Messages.app verwendet.
- Erweiterte Aktionen verwenden `imsg launch`, um den `imsg`-Helper in Messages.app einzuschleusen. Dadurch werden Lesebestätigungen, Tippindikatoren, Rich-Media-Sendungen, Bearbeiten, Zurückziehen, Antworten in Threads, Tapbacks, Umfragen und Gruppenverwaltung ermöglicht.
- Linux-Builds können eine kopierte `chat.db` untersuchen, aber weder Nachrichten senden noch die Live-Datenbank des Macs überwachen oder Messages.app steuern. Führen Sie für OpenClaw iMessage `imsg` auf dem angemeldeten Mac oder über einen SSH-Wrapper zu diesem Mac aus.

## Vorbereitungen

1. Installieren Sie `imsg` auf dem Mac, auf dem Messages.app ausgeführt wird:

   ```bash
   brew install steipete/tap/imsg
   brew update && brew upgrade imsg
   imsg --version
   imsg chats --limit 3
   ```

   Bei der üblichen lokalen Einrichtung kann die OpenClaw-Einrichtung eine vom Benutzer bestätigte Homebrew-Installation oder -Aktualisierung von `imsg` auf dem bei Messages angemeldeten Mac anbieten. Manuelle Einrichtungen und Topologien mit SSH-Wrapper bleiben in der Verantwortung des Betreibers: Wiederholen Sie die Homebrew-Aktualisierung in demselben lokalen oder entfernten Benutzerkontext, in dem `imsg` ausgeführt wird. Wenn `imsg chats` mit `unable to open database file`, leerer Ausgabe oder `authorization denied` fehlschlägt, gewähren Sie dem Terminal, Editor, Node-Prozess, Gateway-Dienst oder übergeordneten SSH-Prozess, der `imsg` startet, vollständigen Festplattenzugriff und öffnen Sie diesen übergeordneten Prozess anschließend erneut.

2. Überprüfen Sie vor einer Änderung der OpenClaw-Konfiguration die Oberflächen für Lesen, Überwachen, Senden und RPC:

   ```bash
   imsg chats --limit 10 --json | jq -s
   imsg history --chat-id 42 --limit 10 --attachments --json | jq -s
   imsg watch --chat-id 42 --reactions --json
   imsg send --chat-id 42 --text "OpenClaw imsg test"
   imsg rpc --help
   ```

   Ersetzen Sie `42` durch eine echte Chat-ID aus `imsg chats`. Das Senden erfordert eine Automatisierungsberechtigung für Messages.app. Wenn OpenClaw über SSH ausgeführt wird, führen Sie diese Befehle über denselben SSH-Wrapper oder Benutzerkontext aus, den OpenClaw verwenden wird. Wenn Lesezugriffe funktionieren, das Senden jedoch mit AppleEvents `-1743` fehlschlägt, prüfen Sie, ob die Automatisierungsberechtigung für `/usr/libexec/sshd-keygen-wrapper` erteilt wurde; siehe [Senden über den SSH-Wrapper schlägt mit AppleEvents -1743 fehl](/de/channels/imessage#requirements-and-permissions-macos).

3. Aktivieren Sie die private API-Bridge. Dies wird für OpenClaw iMessage dringend empfohlen, da Antworten, Tapbacks, Effekte, Umfragen, Antworten auf Anhänge und Gruppenaktionen davon abhängen:

   ```bash
   imsg launch
   imsg status --json
   ```

   `imsg launch` erfordert die Deaktivierung von SIP (und unter modernen macOS-Versionen eine gelockerte Bibliotheksvalidierung – siehe [Aktivieren der privaten imsg-API](/de/channels/imessage#enabling-the-imsg-private-api)). Grundlegendes Senden, der Verlauf und die Überwachung funktionieren ohne `imsg launch`; der vollständige iMessage-Aktionsumfang von OpenClaw jedoch nicht.

4. Nachdem Sie `channels.imessage` aktiviert und den Gateway gestartet haben, überprüfen Sie die Bridge über OpenClaw:

   ```bash
   openclaw channels status --probe
   ```

   Das iMessage-Konto sollte `works` melden; bei `--json` enthält die Prüfantwort `privateApi.available: true`. Wenn `false` gemeldet wird, beheben Sie zunächst dieses Problem – siehe [Funktionserkennung](/de/channels/imessage#private-api-actions). Für die Prüfung muss ein erreichbarer Gateway vorhanden sein (andernfalls gibt die CLI ersatzweise nur auf der Konfiguration basierende Ausgaben zurück), und es werden ausschließlich konfigurierte, aktivierte Konten geprüft.

5. Erstellen Sie eine Sicherungskopie Ihrer Konfiguration:

   ```bash
   cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak
   ```

## Übertragung der Konfiguration

iMessage und BlueBubbles verwenden die meisten Verhaltensschlüssel auf Kanalebene gemeinsam. Geändert werden der Transport (REST-Server gegenüber lokaler CLI) und das Schlüsselformat des Gruppenregisters.

| BlueBubbles                                                | gebündeltes iMessage                      | Hinweise                                                                                                                                                                                                                                                                            |
| ---------------------------------------------------------- | ----------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `channels.bluebubbles.enabled`                             | `channels.imessage.enabled`               | Gleiche Semantik (standardmäßig `true`, sobald der Block vorhanden ist).                                                                                                                                                                                                                           |
| `channels.bluebubbles.serverUrl`                           | _(entfernt)_                               | Kein REST-Server — das Plugin startet `imsg rpc` über stdio.                                                                                                                                                                                                                        |
| `channels.bluebubbles.password`                            | _(entfernt)_                               | Keine Webhook-Authentifizierung erforderlich.                                                                                                                                                                                                                                                |
| _(implizit)_                                               | `channels.imessage.cliPath`               | Pfad zu `imsg` (Standard: `imsg`); verwenden Sie für SSH ein Wrapper-Skript.                                                                                                                                                                                                                   |
| _(implizit)_                                               | `channels.imessage.dbPath`                | Optionale Überschreibung von Messages.app mit `chat.db`; wird automatisch erkannt, wenn nicht angegeben.                                                                                                                                                                                                            |
| _(implizit)_                                               | `channels.imessage.remoteHost`            | `host` oder `user@host` — nur erforderlich, wenn `cliPath` ein SSH-Wrapper ist und Sie Anhänge per SCP abrufen möchten.                                                                                                                                                                        |
| `channels.bluebubbles.dmPolicy`                            | `channels.imessage.dmPolicy`              | Gleiche Werte (`pairing` / `allowlist` / `open` / `disabled`); Standard: `pairing`.                                                                                                                                                                                                  |
| `channels.bluebubbles.allowFrom`                           | `channels.imessage.allowFrom`             | Gleiche Handle-Formate (`+15555550123`, `user@example.com`). Genehmigungen aus dem Kopplungsspeicher werden nicht übertragen — siehe unten.                                                                                                                                                                   |
| `channels.bluebubbles.groupPolicy`                         | `channels.imessage.groupPolicy`           | Gleiche Werte (`allowlist` / `open` / `disabled`); Standard: `allowlist`.                                                                                                                                                                                                            |
| `channels.bluebubbles.groupAllowFrom`                      | `channels.imessage.groupAllowFrom`        | Identisch. Wenn nicht gesetzt, greift iMessage auf `allowFrom` zurück; ein explizit leeres `groupAllowFrom: []` blockiert unter `groupPolicy: "allowlist"` alle Gruppen.                                                                                                                               |
| `channels.bluebubbles.groups`                              | `channels.imessage.groups`                | Kopieren Sie den Platzhaltereintrag `"*"` unverändert; vergeben Sie für gruppenspezifische Einträge anhand der numerischen iMessage-`chat_id` neue Schlüssel — siehe „Stolperfalle der Gruppenregistrierung“. `requireMention`, `tools`, `toolsBySender`, `systemPrompt` werden übernommen.                                                                            |
| `channels.bluebubbles.sendReadReceipts`                    | `channels.imessage.sendReadReceipts`      | Standard: `true`. Beim gebündelten Plugin wird dies nur ausgelöst, wenn die Prüfung der privaten API verfügbar ist.                                                                                                                                                                                        |
| `channels.bluebubbles.includeAttachments`                  | `channels.imessage.includeAttachments`    | Gleiche Struktur, ebenfalls standardmäßig deaktiviert. Wenn Anhänge über BlueBubbles übertragen wurden, legen Sie dies explizit fest — eingehende Fotos/Medien werden andernfalls ohne Hinweis verworfen (keine `Inbound message`-Protokollzeile).                                                                                             |
| `channels.bluebubbles.attachmentRoots`                     | `channels.imessage.attachmentRoots`       | Lokale Stammverzeichnisse; gleiche Platzhalterregeln.                                                                                                                                                                                                                                                |
| _(k. A.)_                                                    | `channels.imessage.remoteAttachmentRoots` | Wird nur verwendet, wenn `remoteHost` für SCP-Abrufe festgelegt ist.                                                                                                                                                                                                                              |
| `channels.bluebubbles.mediaMaxMb`                          | `channels.imessage.mediaMaxMb`            | Standardmäßig 16 MB bei iMessage (BlueBubbles verwendete standardmäßig 8 MB). Legen Sie den Wert explizit fest, um die niedrigere Obergrenze beizubehalten.                                                                                                                                                                                  |
| `channels.bluebubbles.textChunkLimit`                      | `channels.imessage.textChunkLimit`        | Bei beiden standardmäßig 4000.                                                                                                                                                                                                                                                            |
| `channels.bluebubbles.coalesceSameSenderDms`               | _(entfernt)_                               | Migrieren Sie diesen Schlüssel nicht. `imsg` 0.13.1 und neuer führt von Apple-URL-Vorschauen aufgeteilte Sendungen zusammen, bevor OpenClaw sie empfängt; `openclaw doctor --fix` entfernt einen veralteten iMessage-Schlüssel.                                                                                                    |
| `channels.bluebubbles.enrichGroupParticipantsFromContacts` | _(k. A.)_                                   | `imsg` stellt bereits die Anzeigenamen der Absender aus `chat.db` bereit.                                                                                                                                                                                                                     |
| `channels.bluebubbles.actions.*`                           | `channels.imessage.actions.*`             | Gleiche aktionsspezifische Umschalter (`reactions`, `edit`, `unsend`, `reply`, `sendWithEffect`, `renameGroup`, `setGroupIcon`, `addParticipant`, `removeParticipant`, `leaveGroup`, `sendAttachment`) sowie das neue `polls`. Alle sind standardmäßig aktiviert; Aktionen der privaten API erfordern weiterhin die Bridge. |

Konfigurationen mit mehreren Konten (`channels.bluebubbles.accounts.*`) werden eins zu eins in `channels.imessage.accounts.*` übertragen.

## Stolperfalle der Gruppenregistrierung

Das gebündelte iMessage-Plugin führt zwei Gruppenprüfungen direkt nacheinander aus. Eine Gruppennachricht muss beide bestehen, um den Agenten zu erreichen:

1. **Zulassungsliste für Absender/Chat-Ziel** (`channels.imessage.groupAllowFrom`) — gleicht das Absender-Handle oder das Chat-Ziel ab (Einträge `chat_id:`, `chat_guid:`, `chat_identifier:`). Wenn `groupAllowFrom` nicht gesetzt ist, greift diese Prüfung auf `allowFrom` zurück; ein explizites `groupAllowFrom: []` deaktiviert diesen Rückgriff und verwirft unter `groupPolicy: "allowlist"` jede Gruppennachricht.
2. **Gruppenregistrierung** (`channels.imessage.groups`) — nach numerischer iMessage-`chat_id` verschlüsselt:
   - Kein `groups`-Block (oder ein leerer Block): Gruppen bestehen diese Prüfung, solange Prüfung 1 über eine nicht leere effektive Absender-Zulassungsliste verfügt; die Absenderfilterung steuert den Zugriff und beim Start wird keine Warnung zum Verwerfen aller Nachrichten ausgegeben.
   - `groups` mit Einträgen, aber ohne `"*"`: Nur die aufgeführten `chat_id`-Schlüssel werden zugelassen. Sobald eine Gruppe aufgeführt wird, fungiert die Registrierung selbst unter `groupPolicy: "open"` als Zulassungsliste.
   - `groups: { "*": { ... } }`: Jede Gruppe besteht diese Prüfung.

Die Migrationsfalle: BlueBubbles hat `groups`-Einträge anhand der Chat-GUID/Chat-ID verschlüsselt, während die iMessage-Registrierung numerische `chat_id` als Schlüssel verwendet. Werden gruppenspezifische Einträge unverändert kopiert, entsteht eine nicht leere Registrierung, deren Schlüssel niemals übereinstimmen, sodass jede Gruppennachricht bei Prüfung 2 verworfen wird. Kopieren Sie den Platzhalter `"*"` unverändert; vergeben Sie für bestimmte Gruppeneinträge mit den `chat_id`-Werten aus `imsg chats` neue Schlüssel.

Beide Verwerfungspfade sind auf der standardmäßigen Protokollierungsstufe anhand von `warn`-Zeilen sichtbar:

- Einmal pro Konto beim Start, wenn `groupPolicy: "allowlist"` festgelegt und die effektive Absender-Zulassungsliste für Gruppen leer ist: `imessage: groupPolicy="allowlist" for account "<id>" but no group sender allowlist is configured ...`. Legen Sie `groupAllowFrom` (oder `allowFrom`) fest, um Absender zuzulassen; nur `groups` hinzuzufügen, erfüllt die Absenderprüfung nicht.
- Einmal pro `chat_id` zur Laufzeit, wenn die Registrierung eine Gruppe verwirft: `imessage: dropping group message from chat_id=<id> ... not in channels.imessage.groups allowlist`, wobei der genaue hinzuzufügende Schlüssel genannt wird.

Direktnachrichten funktionieren in beiden Fällen weiterhin — sie verwenden einen anderen Codepfad, daher beweist der Erfolg von Direktnachrichten nicht, dass das Gruppen-Routing funktioniert.

Die minimale absenderbezogene Konfiguration mit `groupPolicy: "allowlist"`:

```json5
{
  channels: {
    imessage: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15555550123", "chat_guid:any;-;..."],
    },
  },
}
```

Damit werden die konfigurierten Absender in jeder Gruppe zugelassen. Fügen Sie `groups`-Einträge hinzu, um die zulässigen Chats einzuschränken oder chatbezogene Optionen wie `requireMention` festzulegen; kopieren Sie den BlueBubbles-Eintrag `"*"` unverändert, vergeben Sie aber für bestimmte Einträge mit den numerischen iMessage-`chat_id`-Werten neue Schlüssel.

## Schritt für Schritt

1. Übertragen Sie die Konfiguration. Lassen Sie den neuen Block während der Bearbeitung deaktiviert; der alte `channels.bluebubbles`-Block wird von der aktuellen OpenClaw-Version ignoriert und kann als Referenz parallel bestehen bleiben:

   ```json5
   {
     channels: {
       imessage: {
         enabled: false, // auf true setzen, sobald die Umstellung erfolgen kann
         cliPath: "/opt/homebrew/bin/imsg",
         dmPolicy: "pairing",
         allowFrom: ["+15555550123"], // aus bluebubbles.allowFrom kopieren
         groupPolicy: "allowlist",
         groupAllowFrom: [], // aus bluebubbles.groupAllowFrom kopieren
         groups: { "*": { requireMention: true } }, // Platzhalter unverändert kopieren; chatbezogene Einträge anhand von chat_id neu verschlüsseln
         // Aktionen sind standardmäßig aktiviert; einzelne Umschalter zum Deaktivieren auf false setzen
       },
     },
   }
   ```

2. **Umstellen und prüfen.** Legen Sie `channels.imessage.enabled: true` fest, starten Sie den Gateway neu und vergewissern Sie sich, dass der Kanal einen fehlerfreien Zustand meldet:

   ```bash
   openclaw gateway restart
   openclaw channels status --probe --channel imessage   # „works“ erwarten; --json zeigt privateApi.available: true
   ```

   Die Prüfung erfordert einen erreichbaren Gateway und prüft nur konfigurierte, aktivierte Konten. Verwenden Sie die direkten `imsg`-Befehle unter [Bevor Sie beginnen](#before-you-start), um den Mac selbst zu überprüfen.

3. **Direktnachrichten überprüfen.** Senden Sie dem Agenten eine Direktnachricht und bestätigen Sie, dass die Antwort ankommt.

4. **Gruppen separat überprüfen.** Direktnachrichten und Gruppen verwenden unterschiedliche Codepfade – eine erfolgreiche Direktnachricht beweist nicht, dass das Routing für Gruppen funktioniert. Senden Sie eine Nachricht in einem zulässigen Gruppenchat und bestätigen Sie, dass die Antwort ankommt. Falls die Gruppe stumm bleibt (keine Antwort des Agenten, kein Fehler), prüfen Sie das Gateway-Protokoll auf die beiden `warn`-Zeilen aus „Group registry footgun“ weiter oben. Die Startwarnung bedeutet, dass die effektive Absender-Zulassungsliste leer ist; eine Warnung pro `chat_id` bedeutet, dass eine befüllte `groups`-Registrierung diesen Chat nicht enthält.

5. **Aktionsumfang überprüfen.** Bitten Sie den Agenten aus einer gekoppelten Direktnachricht heraus, eine Reaktion hinzuzufügen, eine Nachricht zu bearbeiten, zurückzuziehen und zu beantworten, ein Foto zu senden sowie (in einer Gruppe) die Gruppe umzubenennen oder einen Teilnehmer hinzuzufügen beziehungsweise zu entfernen. Jede Aktion sollte nativ in Messages.app ausgeführt werden. Falls eine Aktion `iMessage <action> requires the imsg private API bridge` auslöst, führen Sie `imsg launch` erneut aus und aktualisieren Sie mit `openclaw channels status --probe`.

6. **Entfernen Sie den BlueBubbles-Server und den `channels.bluebubbles`-Block**, sobald iMessage-Direktnachrichten, -Gruppen und -Aktionen überprüft wurden. OpenClaw liest `channels.bluebubbles` nicht.

## Aktionsparität im Überblick

| Aktion                                              | bisheriges BlueBubbles | gebündeltes iMessage                                                              |
| --------------------------------------------------- | ---------------------- | --------------------------------------------------------------------------------- |
| Text senden / SMS-Ausweichlösung                    | ✅                     | ✅                                                                                |
| Medien senden (Foto, Video, Datei, Sprache)         | ✅                     | ✅                                                                                |
| Antwort im Thread (`reply_to_guid`)              | ✅                     | ✅ (schließt [#51892](https://github.com/openclaw/openclaw/issues/51892))          |
| Tapback (`react`)                        | ✅                     | ✅                                                                                |
| Bearbeiten / zurückziehen (Empfänger mit macOS 13+) | ✅                     | ✅                                                                                |
| Mit Bildschirmeffekt senden                         | ✅                     | ✅ (schließt einen Teil von [#9394](https://github.com/openclaw/openclaw/issues/9394)) |
| Rich-Text: fett / kursiv / unterstrichen / durchgestrichen | ✅              | ✅ (Formatierung typisierter Bereiche über attributedBody)                        |
| Native Messages-Umfragen (erstellen und abstimmen)  | ❌                     | ✅ (`actions.polls`; Empfänger benötigen iOS/macOS 26+ für die native Darstellung) |
| Gruppe umbenennen / Gruppensymbol festlegen         | ✅                     | ✅                                                                                |
| Teilnehmer hinzufügen / entfernen, Gruppe verlassen | ✅                    | ✅                                                                                |
| Lesebestätigungen und Tippanzeige                   | ✅                     | ✅ (abhängig von erfolgreicher Prüfung der privaten API)                          |
| Zusammenführung aufgeteilter Apple-URL-Vorschauen   | ✅                     | ✅ (wird upstream von `imsg` 0.13.1 und neuer verarbeitet; keine OpenClaw-Einstellung) |
| Wiederherstellung eingehender Nachrichten nach einem Neustart | ✅            | ✅ (automatisch: `since_rowid`-Wiedergabe + GUID-Deduplizierung; größeres Zeitfenster bei lokaler Ausführung) |

iMessage stellt Nachrichten wieder her, die während eines Ausfalls des Gateways verpasst wurden: Beim Start erfolgt über `imsg watch.subscribe` `since_rowid` eine Wiedergabe ab der zuletzt weitergeleiteten rowid, eine Deduplizierung anhand der GUID und eine Altersgrenze für veraltete Rückstände unterdrückt die „backlog bomb“ beim Push-Leeren. Dies läuft über die `imsg`-RPC-Verbindung und funktioniert daher auch bei entfernten SSH-`cliPath`-Einrichtungen; lokale Einrichtungen erhalten ein größeres Wiederherstellungszeitfenster, da sie `chat.db` lesen können. Siehe [Wiederherstellung eingehender Nachrichten nach dem Neustart einer Bridge oder des Gateways](/de/channels/imessage#inbound-recovery-after-a-bridge-or-gateway-restart).

## Kopplung, Sitzungen und ACP-Bindungen

- **Zulassungslisten werden anhand des Handles übernommen.** `channels.imessage.allowFrom` erkennt dieselben `+15555550123`- / `user@example.com`-Zeichenfolgen, die BlueBubbles verwendet hat – kopieren Sie sie unverändert.
- **Genehmigungen aus dem Kopplungsspeicher werden nicht übertragen.** Der Kopplungsspeicher ist kanalspezifisch, und der alte BlueBubbles-Speicher wird nicht migriert. Absender, die ausschließlich über die Kopplung genehmigt wurden, müssen sich unter iMessage erneut koppeln; alternativ fügen Sie ihre Handles zu `allowFrom` hinzu.
- **Sitzungen** bleiben auf Agent und Chat beschränkt. Direktnachrichten werden unter dem standardmäßigen `session.dmScope=main` in der Hauptsitzung des Agenten zusammengeführt; Gruppensitzungen bleiben pro `chat_id` (`agent:<agentId>:imessage:group:<chat_id>`) isoliert. Der alte Gesprächsverlauf unter BlueBubbles-Sitzungsschlüsseln wird nicht in iMessage-Sitzungen übernommen.
- **ACP-Bindungen**, die auf `match.channel: "bluebubbles"` verweisen, müssen in `"imessage"` geändert werden. Die `match.peer.id`-Formen (`chat_id:`, `chat_guid:`, `chat_identifier:`, reines Handle) sind identisch.

## Kein Kanal für ein Rollback

Es gibt keine unterstützte BlueBubbles-Laufzeit, zu der zurückgewechselt werden kann. Falls die iMessage-Überprüfung fehlschlägt, setzen Sie `channels.imessage.enabled: false`, starten Sie den Gateway neu, beheben Sie den `imsg`-Blocker und versuchen Sie die Umstellung erneut.

Der Antwort-Cache befindet sich im SQLite-Plugin-Status. `openclaw doctor --fix` importiert und archiviert die alte `imessage/reply-cache.jsonl`-Sidecar-Datei, sofern sie vorhanden ist.

## Verwandte Themen

- [Entfernung von BlueBubbles und der imsg-iMessage-Pfad](/de/announcements/bluebubbles-imessage) – kurze Ankündigung und Zusammenfassung für Betreiber.
- [iMessage](/de/channels/imessage) – vollständige Referenz zum iMessage-Kanal, einschließlich Einrichtung von `imsg launch` und Funktionserkennung.
- `/channels/bluebubbles` – bisherige URL, die zu diesem Migrationsleitfaden weiterleitet.
- [Kopplung](/de/channels/pairing) – Authentifizierung von Direktnachrichten und Kopplungsablauf.
- [Kanal-Routing](/de/channels/channel-routing) – wie der Gateway einen Kanal für ausgehende Antworten auswählt.
