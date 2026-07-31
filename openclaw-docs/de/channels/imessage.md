---
read_when:
    - iMessage-Unterstützung einrichten
    - Fehlerbehebung beim Senden/Empfangen mit iMessage
summary: Native iMessage-Unterstützung über imsg (JSON-RPC über stdio) mit privaten API-Aktionen für Antworten, Tapbacks, Effekte, Umfragen, Anhänge und Gruppenverwaltung. Bevorzugt für neue OpenClaw-iMessage-Einrichtungen, wenn die Hostanforderungen erfüllt sind.
title: iMessage
x-i18n:
    generated_at: "2026-07-26T18:14:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f3e8b1a65c76b25d03615c06a976f86a8af555cd96d5bfdb10cef9c955893ddc
    source_path: channels/imessage.md
    workflow: 16
---

<Note>
Für die übliche OpenClaw-iMessage-Bereitstellung führen Sie den Gateway und `imsg` auf demselben bei macOS Messages angemeldeten Host aus. Wenn Ihr Gateway an einem anderen Ort ausgeführt wird, verweisen Sie `channels.imessage.cliPath` auf einen transparenten SSH-Wrapper, der `imsg` auf dem Mac ausführt.

**Die Wiederherstellung eingehender Nachrichten erfolgt automatisch.** Nach einem Neustart der Bridge oder des Gateways gibt iMessage die während des Ausfalls verpassten Nachrichten erneut wieder und unterdrückt den veralteten „Backlog-Bomb“-Schwall, den Apple nach einer Push-Wiederherstellung ausgeben kann. Dabei verhindert die Deduplizierung, dass etwas zweimal weitergeleitet wird. Es muss keine Konfiguration aktiviert werden — siehe [Wiederherstellung eingehender Nachrichten nach einem Neustart der Bridge oder des Gateways](#inbound-recovery-after-a-bridge-or-gateway-restart).
</Note>

<Warning>
Die Unterstützung für BlueBubbles wurde entfernt. Migrieren Sie `channels.bluebubbles`-Konfigurationen zu `channels.imessage`; OpenClaw unterstützt iMessage ausschließlich über `imsg`. Lesen Sie zunächst [Entfernung von BlueBubbles und der imsg-iMessage-Pfad](/de/announcements/bluebubbles-imessage) für die kurze Ankündigung oder [Umstieg von BlueBubbles](/de/channels/imessage-from-bluebubbles) für die vollständige Migrationstabelle.
</Warning>

Status: native externe CLI-Integration. Der Gateway startet `imsg rpc` und kommuniziert über stdio mittels JSON-RPC — ohne separaten Daemon oder Port. Der Modus der privaten API wird für einen vollständigen iMessage-Kanal dringend empfohlen; Antworten, Tapbacks, Effekte, Umfragen, Antworten auf Anhänge und Gruppenaktionen erfordern `imsg launch` sowie eine erfolgreiche Prüfung der privaten API.

Für die gängige lokale Einrichtung kann die OpenClaw-Einrichtung eine vom Benutzer bestätigte Homebrew-Installation oder -Aktualisierung von `imsg` auf dem bei Messages angemeldeten Mac anbieten. Die manuelle Einrichtung und Topologien mit SSH-Wrapper werden weiterhin vom Betreiber verwaltet: Installieren oder aktualisieren Sie `imsg` in demselben Benutzerkontext, in dem der Gateway oder Wrapper ausgeführt wird.

<CardGroup cols={3}>
  <Card title="Aktionen der privaten API" icon="wand-sparkles" href="#private-api-actions">
    Antworten, Tapbacks, Effekte, Umfragen, Anhänge und Gruppenverwaltung.
  </Card>
  <Card title="Kopplung" icon="link" href="/de/channels/pairing">
    Für iMessage-Direktnachrichten wird standardmäßig der Kopplungsmodus verwendet.
  </Card>
  <Card title="Entfernter Mac" icon="terminal" href="#remote-mac-over-ssh">
    Verwenden Sie einen SSH-Wrapper, wenn der Gateway nicht auf dem Messages-Mac ausgeführt wird.
  </Card>
  <Card title="Konfigurationsreferenz" icon="settings" href="/de/gateway/config-channels#imessage">
    Vollständige Referenz der iMessage-Felder.
  </Card>
</CardGroup>

## Schnelleinrichtung

<Tabs>
  <Tab title="Lokaler Mac (schneller Weg)">
    <Steps>
      <Step title="imsg installieren und überprüfen">

```bash
brew install steipete/tap/imsg
brew update && brew upgrade imsg
imsg rpc --help
imsg launch
openclaw channels status --probe
```

        Wenn der lokale Einrichtungsassistent einen fehlenden standardmäßigen `imsg`-Befehl erkennt, kann er zur Installation von `steipete/tap/imsg` über Homebrew auffordern. Wenn er ein von Homebrew verwaltetes `imsg` erkennt, kann er zur Neuinstallation oder Aktualisierung auffordern. Benutzerdefinierte `cliPath`-Wrapper werden nicht geändert.

      </Step>

      <Step title="OpenClaw konfigurieren">

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "/usr/local/bin/imsg",
      dbPath: "/Users/user/Library/Messages/chat.db",
    },
  },
}
```

      </Step>

      <Step title="Gateway starten">

```bash
openclaw gateway
```

      </Step>

      <Step title="Erste Kopplung einer Direktnachricht genehmigen (standardmäßige dmPolicy)">

```bash
openclaw pairing list imessage
openclaw pairing approve imessage <CODE>
```

        Kopplungsanfragen laufen nach 1 Stunde ab.
      </Step>
    </Steps>

  </Tab>

  <Tab title="Entfernter Mac über SSH">
    Die meisten Einrichtungen benötigen kein SSH. Verwenden Sie diese Topologie nur, wenn der Gateway nicht auf dem bei Messages angemeldeten Mac ausgeführt werden kann. OpenClaw erfordert lediglich ein stdio-kompatibles `cliPath`, sodass Sie `cliPath` auf ein Wrapper-Skript verweisen können, das per SSH eine Verbindung zu einem entfernten Mac herstellt und `imsg` ausführt.
    Installieren und aktualisieren Sie `imsg` auf diesem entfernten Mac, nicht auf dem Gateway-Host:

```bash
ssh messages-mac 'brew install steipete/tap/imsg && brew update && brew upgrade imsg'
```

```bash
#!/usr/bin/env bash
exec ssh -T messages-mac imsg "$@"
```

    Empfohlene Konfiguration bei aktivierten Anhängen:

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "~/.openclaw/scripts/imsg-ssh",
      remoteHost: "user@gateway-host", // used for SCP attachment fetches
      includeAttachments: true,
      // Optional: extra allowed attachment roots (merged with the default
      // /Users/*/Library/Messages/Attachments).
      attachmentRoots: ["/Users/*/Library/Messages/Attachments"],
      remoteAttachmentRoots: ["/Users/*/Library/Messages/Attachments"],
    },
  },
}
```

    Wenn `remoteHost` nicht festgelegt ist, versucht OpenClaw, es durch Analyse des SSH-Wrapper-Skripts automatisch zu erkennen.
    `remoteHost` muss `host` oder `user@host` sein (keine Leerzeichen oder SSH-Optionen); unsichere Werte werden ignoriert.
    OpenClaw verwendet für SCP eine strikte Hostschlüsselprüfung, daher muss der Hostschlüssel des Relay-Hosts bereits in `~/.ssh/known_hosts` vorhanden sein.
    Anhangspfade werden anhand der zulässigen Stammverzeichnisse (`attachmentRoots` / `remoteAttachmentRoots`) validiert.

<Warning>
Jeder `cliPath`-Wrapper oder SSH-Proxy, den Sie `imsg` vorschalten, MUSS sich für langlebiges JSON-RPC wie eine transparente stdio-Pipe verhalten. OpenClaw tauscht über stdin/stdout des Wrappers während der gesamten Lebensdauer des Kanals kleine, durch Zeilenumbrüche begrenzte JSON-RPC-Nachrichten aus:

- Leiten Sie jeden stdin-Block beziehungsweise jede stdin-Zeile **sofort weiter, sobald Bytes verfügbar sind** — warten Sie nicht auf EOF.
- Leiten Sie jeden stdout-Block beziehungsweise jede stdout-Zeile unverzüglich in umgekehrter Richtung weiter.
- Behalten Sie Zeilenumbrüche bei.
- Vermeiden Sie blockierende Lesevorgänge mit fester Größe (`read(4096)`, `cat | buffer`, standardmäßiges Shell-`read`), die kleine Frames aushungern können.
- Halten Sie stderr vom JSON-RPC-stdout-Datenstrom getrennt.

Ein Wrapper, der stdin puffert, bis ein großer Block gefüllt ist, verursacht Symptome, die wie ein iMessage-Ausfall wirken — `imsg rpc timeout (chats.list)` oder wiederholte Neustarts des Kanals — obwohl `imsg rpc` selbst fehlerfrei funktioniert. `ssh -T host imsg "$@"` (oben) ist sicher, da es die `cliPath`-Argumente von OpenClaw wie `rpc` und `--db` weiterleitet. Pipelines wie `ssh host imsg | grep -v '^DEBUG'` sind NICHT sicher — selbst zeilengepufferte Tools können Frames zurückhalten; verwenden Sie `stdbuf -oL -eL` in jeder Stufe, wenn eine Filterung unvermeidbar ist.
</Warning>

  </Tab>
</Tabs>

## Anforderungen und Berechtigungen (macOS)

- Messages muss auf dem Mac angemeldet sein, auf dem `imsg` ausgeführt wird.
- Vollständiger Festplattenzugriff ist für den Prozesskontext erforderlich, in dem OpenClaw/`imsg` ausgeführt wird (Zugriff auf die Messages-Datenbank).
- Die Automatisierungsberechtigung ist erforderlich, um Nachrichten über Messages.app zu senden.
- Für erweiterte Aktionen (Reaktion / Bearbeiten / Senden rückgängig machen / Antwort im Thread / Effekte / Umfragen / Gruppenoperationen) muss der Systemintegritätsschutz deaktiviert sein — siehe [Private API von imsg aktivieren](#enabling-the-imsg-private-api). Das grundlegende Senden und Empfangen von Text und Medien funktioniert ohne diese Deaktivierung.

<Tip>
Berechtigungen werden pro Prozesskontext erteilt. Wenn der Gateway ohne Benutzeroberfläche ausgeführt wird (LaunchAgent/SSH), führen Sie einmalig einen interaktiven Befehl in demselben Kontext aus, um die Eingabeaufforderungen auszulösen:

```bash
imsg chats --limit 1
# or
imsg send <handle> "test"
```

</Tip>

<Accordion title="Senden über SSH-Wrapper schlägt mit AppleEvents -1743 fehl">
  Eine Einrichtung mit entferntem Zugriff über SSH kann Chats lesen, `channels status --probe` bestehen und eingehende Nachrichten verarbeiten, während das Senden ausgehender Nachrichten weiterhin mit einem AppleEvents-Autorisierungsfehler fehlschlägt:

```text
Not authorized to send Apple events to Messages. (-1743)
```

Prüfen Sie die TCC-Datenbank des auf dem Mac angemeldeten Benutzers oder System Settings > Privacy & Security > Automation. Wenn der Automation-Eintrag für `/usr/libexec/sshd-keygen-wrapper` statt für den `imsg`- oder lokalen Shell-Prozess erfasst wurde, stellt macOS möglicherweise keinen verwendbaren Messages-Schalter für diesen serverseitigen SSH-Client bereit:

```text
kTCCServiceAppleEvents | /usr/libexec/sshd-keygen-wrapper | auth_value=0 | com.apple.MobileSMS
```

In diesem Zustand können wiederholtes `tccutil reset AppleEvents` oder die erneute Ausführung von `imsg send` über denselben SSH-Wrapper weiterhin fehlschlagen, weil der Prozesskontext, der die Messages-Automatisierung benötigt, der SSH-Wrapper ist und nicht eine Anwendung, der die Benutzeroberfläche die Berechtigung erteilen kann.

Verwenden Sie stattdessen einen der unterstützten `imsg`-Prozesskontexte:

- Führen Sie den Gateway oder zumindest die `imsg`-Bridge in der lokalen Sitzung des bei Messages angemeldeten Benutzers aus.
- Starten Sie den Gateway mit einem LaunchAgent für diesen Benutzer, nachdem Sie in derselben Sitzung vollständigen Festplattenzugriff und die Automatisierungsberechtigung erteilt haben.
- Wenn Sie die Zwei-Benutzer-SSH-Topologie beibehalten, überprüfen Sie vor dem Aktivieren des Kanals, ob ein tatsächlicher ausgehender `imsg send` über genau diesen Wrapper erfolgreich ist. Wenn ihm keine Automatisierungsberechtigung erteilt werden kann, konfigurieren Sie stattdessen eine `imsg`-Einrichtung mit nur einem Benutzer, anstatt sich beim Senden auf den SSH-Wrapper zu verlassen.

</Accordion>

## Private API von imsg aktivieren

`imsg` wird mit zwei Betriebsmodi ausgeliefert. Für OpenClaw ist der Modus der privaten API die empfohlene Einrichtung, da er dem Kanal die nativen iMessage-Aktionen bereitstellt, die Benutzer erwarten. Der Basismodus bleibt für Installationen mit geringem Risiko, die erste Überprüfung oder Hosts nützlich, auf denen SIP nicht deaktiviert werden kann.

- **Basismodus** (Standard, keine SIP-Änderungen erforderlich): ausgehender Text und ausgehende Medien über `send`, Überwachung/Verlauf eingehender Nachrichten, Chatliste. Dies ist der sofort verfügbare Funktionsumfang einer neuen `brew install steipete/tap/imsg`-Installation mit den oben genannten standardmäßigen macOS-Berechtigungen.
- **Modus der privaten API**: `imsg` injiziert eine Hilfsbibliothek vom Typ dylib in `Messages.app`, um interne `IMCore`-Funktionen aufzurufen. Dadurch werden `react`, `edit`, `unsend`, `reply` (in Threads), `sendWithEffect`, `poll` und `poll-vote` (native Messages-Umfragen), `renameGroup`, `setGroupIcon`, `addParticipant`, `removeParticipant`, `leaveGroup` sowie Tippindikatoren und Lesebestätigungen ermöglicht.

Die auf dieser Seite empfohlene Aktionsoberfläche erfordert den Modus der privaten API. Die README-Datei von `imsg` beschreibt die Anforderung ausdrücklich:

> Erweiterte Funktionen wie `read`, `typing`, `launch`, Bridge-gestütztes Senden umfangreicher Inhalte, Nachrichtenänderungen und Chatverwaltung sind optional. Sie erfordern, dass SIP deaktiviert und eine Hilfsbibliothek vom Typ dylib in `Messages.app` injiziert wird. `imsg launch` verweigert die Injektion, wenn SIP aktiviert ist.

Die Technik zur Injektion der Hilfsbibliothek verwendet die eigene dylib von `imsg`, um auf die privaten APIs von Messages zuzugreifen. Im OpenClaw-iMessage-Pfad gibt es weder einen Drittanbieterserver noch eine BlueBubbles-Laufzeitumgebung.

<Warning>
**Das Deaktivieren von SIP ist ein tatsächlicher Sicherheitskompromiss.** SIP gehört zu den zentralen Schutzmechanismen von macOS gegen die Ausführung veränderten Systemcodes; die systemweite Deaktivierung eröffnet zusätzliche Angriffsflächen und hat Nebenwirkungen. Insbesondere wird durch **das Deaktivieren von SIP auf Macs mit Apple Silicon auch die Möglichkeit deaktiviert, iOS-Apps auf Ihrem Mac zu installieren und auszuführen**.

Behandeln Sie dies als bewusste betriebliche Entscheidung, insbesondere auf einem primären persönlichen Mac. Für eine produktionsreife OpenClaw-iMessage-Umgebung empfiehlt sich ein dedizierter Mac oder ein macOS-Bot-Benutzer, bei dem Sie die Bridge bedenkenlos aktivieren können. Wenn Ihr Bedrohungsmodell eine Deaktivierung von SIP nirgends zulässt, ist das gebündelte iMessage auf den Basismodus beschränkt — nur Senden und Empfangen von Text und Medien, keine Reaktionen / Bearbeitung / Rückgängigmachen des Sendens / Effekte / Gruppenoperationen.
</Warning>

### Einrichtung

1. **Installieren (oder aktualisieren) Sie `imsg`** auf dem Mac, auf dem Messages.app ausgeführt wird:

   ```bash
   brew install steipete/tap/imsg
   brew update && brew upgrade imsg
   imsg --version
   imsg status --json
   ```

   Die Ausgabe von `imsg status --json` meldet `bridge_version`, `rpc_methods` und methodenspezifische `selectors`, sodass Sie vor dem Start erkennen können, welche Funktionen der aktuelle Build unterstützt.

2. **Deaktivieren Sie den Systemintegritätsschutz und (auf modernen macOS-Versionen) die Bibliotheksvalidierung.** Das Einschleusen einer nicht von Apple stammenden Hilfs-dylib in die von Apple signierte `Messages.app` erfordert, dass SIP deaktiviert **und** die Bibliotheksvalidierung gelockert ist. Der SIP-Schritt im Wiederherstellungsmodus hängt von der macOS-Version ab:
   - **macOS 10.13-10.15 (Sierra-Catalina):** Deaktivieren Sie die Bibliotheksvalidierung über das Terminal, starten Sie im Wiederherstellungsmodus neu, führen Sie `csrutil disable` aus und starten Sie erneut.
   - **macOS 11+ (Big Sur und neuer), Intel:** Starten Sie im Wiederherstellungsmodus (oder über die Internetwiederherstellung), führen Sie `csrutil disable` aus und starten Sie neu.
   - **macOS 11+, Apple Silicon:** Verwenden Sie die Startsequenz mit dem Ein-/Ausschalter, um die Wiederherstellung aufzurufen; halten Sie bei aktuellen macOS-Versionen die Taste **Left Shift** gedrückt, wenn Sie auf Continue klicken, und führen Sie anschließend `csrutil disable` aus. Für virtuelle Maschinen gilt ein separater Ablauf; erstellen Sie daher zuerst einen VM-Snapshot.

   **Unter macOS 11 und neuer reicht `csrutil disable` allein normalerweise nicht aus.** Apple erzwingt weiterhin die Bibliotheksvalidierung für `Messages.app` als Plattformbinärdatei, sodass eine ad hoc signierte Hilfskomponente selbst bei deaktiviertem SIP abgewiesen wird (`Library Validation failed: ... platform binary, but mapped file is not`). Deaktivieren Sie nach SIP auch die Bibliotheksvalidierung und starten Sie neu:

   ```bash
   sudo defaults write /Library/Preferences/com.apple.security.libraryvalidation.plist DisableLibraryValidation -bool true
   ```

   **macOS 26 (Tahoe), verifiziert unter 26.5.1:** Deaktiviertes SIP **zusammen mit** dem obigen Befehl `DisableLibraryValidation` genügt, um die Hilfskomponente unter den Versionen 26.0 bis 26.5.x einzuschleusen. **Es sind keine Boot-Argumente erforderlich.** Die plist ist der entscheidende Faktor und der am häufigsten fehlende Schritt, wenn das Einschleusen unter Tahoe fehlschlägt:
   - **Mit der plist:** `imsg launch` schleust die Hilfskomponente ein und `imsg status` meldet `advanced_features: true`.
   - **Ohne die plist (selbst bei deaktiviertem SIP):** `imsg launch` schlägt mit `Failed to launch: Timeout waiting for Messages.app to initialize` fehl. AMFI weist die ad hoc signierte Hilfskomponente beim Laden ab, sodass die Bridge nie bereit wird und der Start wegen einer Zeitüberschreitung abbricht. Diese Zeitüberschreitung ist das Symptom, auf das die meisten Personen unter Tahoe stoßen; die Lösung ist die obige plist und keine drastischere Maßnahme.

   Falls das Einschleusen über `imsg launch` oder bestimmte `selectors` nach einem macOS-Upgrade beginnen, false zurückzugeben, ist diese Sperre üblicherweise die Ursache. Prüfen Sie den Status von SIP und der Bibliotheksvalidierung, bevor Sie davon ausgehen, dass der SIP-Schritt selbst fehlgeschlagen ist. Wenn diese Einstellungen korrekt sind und die Bridge weiterhin nichts einschleusen kann, erfassen Sie `imsg status --json` sowie die Ausgabe von `imsg launch` und melden Sie dies dem Projekt `imsg`, statt weitere systemweite Sicherheitskontrollen zu schwächen.

3. **Schleusen Sie die Hilfskomponente ein.** Bei deaktiviertem SIP und angemeldeter Messages.app:

   ```bash
   imsg launch
   ```

   `imsg launch` verweigert das Einschleusen, solange SIP aktiviert ist. Dies dient daher zugleich als Bestätigung, dass Schritt 2 erfolgreich war.

4. **Überprüfen Sie die Bridge über OpenClaw:**

   ```bash
   openclaw channels status --probe
   ```

   Der iMessage-Eintrag sollte `works` melden, und `imsg status --json | jq '{rpc_methods, selectors}'` sollte die von Ihrem macOS-Build bereitgestellten Fähigkeiten anzeigen. Das Erstellen von Umfragen erfordert `selectors.pollPayloadMessage`; Abstimmungen erfordern sowohl `selectors.pollVoteMessage` als auch die RPC-Methode `poll.vote`. Das OpenClaw-Plugin bietet nur Aktionen an, die von der zwischengespeicherten Prüfung unterstützt werden, während ein leerer Cache optimistisch bleibt und beim ersten Versand eine Prüfung durchführt.

Wenn `openclaw channels status --probe` den Kanal als `works` meldet, bestimmte Aktionen aber beim Versand den Fehler „iMessage `<action>` requires the imsg private API bridge“ auslösen, führen Sie `imsg launch` erneut aus. Die Hilfskomponente kann ausfallen (Neustart von Messages.app, Betriebssystemupdate usw.), und der zwischengespeicherte Status `available: true` bietet weiterhin Aktionen an, bis die nächste Prüfung ihn aktualisiert.

### Wenn SIP aktiviert bleibt

Falls das Deaktivieren von SIP für Ihr Bedrohungsmodell nicht akzeptabel ist:

- `imsg` wechselt in den Basismodus zurück — nur Text, Medien und Empfang.
- Das OpenClaw-Plugin bietet weiterhin das Senden von Text und Medien sowie die Überwachung eingehender Nachrichten an; `react`, `edit`, `unsend`, `reply`, `sendWithEffect` und Gruppenoperationen werden aus der Aktionsoberfläche ausgeblendet (entsprechend der Fähigkeitssperre pro Methode).
- Sie können für die iMessage-Arbeitslast einen separaten Mac ohne Apple Silicon (oder einen dedizierten Bot-Mac) mit deaktiviertem SIP verwenden, während SIP auf Ihren primären Geräten aktiviert bleibt. Siehe unten [Dedizierter macOS-Bot-Benutzer (separate iMessage-Identität)](#deployment-patterns).

## Zugriffskontrolle und Routing

<Tabs>
  <Tab title="Richtlinie für Direktnachrichten">
    `channels.imessage.dmPolicy` steuert Direktnachrichten:

    - `pairing` (Standard)
    - `allowlist` (erfordert mindestens einen Eintrag in `allowFrom`)
    - `open` (erfordert, dass `allowFrom` den Wert `"*"` enthält)
    - `disabled`

    Feld für die Positivliste: `channels.imessage.allowFrom`.

    Einträge der Positivliste müssen Absender identifizieren: Handles oder statische Absenderzugriffsgruppen (`accessGroup:<name>`). Verwenden Sie `channels.imessage.groupAllowFrom` für Chat-Ziele wie `chat_id:*`, `chat_guid:*` oder `chat_identifier:*`; verwenden Sie `channels.imessage.groups` für numerische Registrierungsschlüssel vom Typ `chat_id`.

  </Tab>

  <Tab title="Gruppenrichtlinie und Erwähnungen">
    `channels.imessage.groupPolicy` steuert die Gruppenverarbeitung:

    - `allowlist` (Standard)
    - `open`
    - `disabled`

    Positivliste für Gruppenabsender: `channels.imessage.groupAllowFrom`.

    Einträge in `groupAllowFrom` können auch auf statische Absenderzugriffsgruppen (`accessGroup:<name>`) verweisen.

    Laufzeit-Fallback: Wenn `groupAllowFrom` nicht gesetzt ist, verwenden Prüfungen von iMessage-Gruppenabsendern `allowFrom`; setzen Sie `groupAllowFrom`, wenn die Zulassung für Direktnachrichten und Gruppen unterschiedlich sein soll. Ein ausdrücklich leeres `groupAllowFrom: []` greift nicht auf einen Fallback zurück — es blockiert unter `allowlist` alle Gruppenabsender.
    Laufzeithinweis: Wenn `channels.imessage` vollständig fehlt, greift die Laufzeit auf `groupPolicy="allowlist"` zurück und protokolliert eine Warnung (selbst wenn `channels.defaults.groupPolicy` gesetzt ist).

    <Warning>
    Das Gruppen-Routing unter `groupPolicy: "allowlist"` durchläuft **zwei** direkt aufeinanderfolgende Sperren:

    1. **Absender-Positivliste** (`channels.imessage.groupAllowFrom`) — Handle, `accessGroup:<name>`, `chat_guid`, `chat_identifier` oder `chat_id`. Eine leere effektive Liste (kein `groupAllowFrom` und kein Fallback über `allowFrom`) blockiert jeden Gruppenabsender.
    2. **Gruppenregistrierung** (`channels.imessage.groups`) — wird erzwungen, sobald die Zuordnung Einträge enthält: Der Chat muss einem expliziten Eintrag pro `chat_id` oder dem Platzhalter `groups: { "*": { ... } }` entsprechen. Wenn `groups` leer ist oder fehlt, entscheidet allein die Absender-Positivliste über die Zulassung.

    Wenn keine effektive Positivliste für Gruppenabsender konfiguriert ist, wird jede Gruppennachricht vor der Registrierungssperre verworfen. Jede Sperre hat auf der standardmäßigen Protokollierungsstufe ein eigenes Signal der Stufe `warn`, und jedes nennt eine andere Lösung:

    - einmalig pro Konto beim Start, wenn die effektive Positivliste für Gruppenabsender leer ist: `imessage: groupPolicy="allowlist" for account "<id>" but no group sender allowlist is configured ...` — beheben Sie dies, indem Sie `channels.imessage.groupAllowFrom` (oder `allowFrom`) festlegen; das alleinige Hinzufügen von Einträgen in `groups` führt weiterhin dazu, dass Sperre 1 jeden Absender blockiert.
    - einmalig pro `chat_id` zur Laufzeit, wenn ein Absender Sperre 1 passiert hat, der Chat aber in einer befüllten `groups`-Registrierung fehlt: `imessage: dropping group message from chat_id=<id> ...` — beheben Sie dies, indem Sie diesen `chat_id` (oder `"*"`) unter `channels.imessage.groups` hinzufügen.

    Direktnachrichten sind davon nicht betroffen — sie verwenden einen anderen Codepfad.

    Empfohlene Konfiguration für den Gruppenfluss unter `groupPolicy: "allowlist"`:

    ```json5
    {
      channels: {
        imessage: {
          groupPolicy: "allowlist",
          groupAllowFrom: ["+15555550123"],
          groups: { "*": { "requireMention": true } },
        },
      },
    }
    ```

    `groupAllowFrom` allein lässt diese Absender in jeder Gruppe zu; fügen Sie den Block `groups` hinzu, um festzulegen, welche Chats zulässig sind (und um chatbezogene Optionen wie `requireMention` zu setzen).
    </Warning>

    Erwähnungssperre für Gruppen:

    - iMessage besitzt keine nativen Metadaten für Erwähnungen
    - die Erkennung von Erwähnungen verwendet reguläre Ausdrucksmuster (`agents.entries.*.groupChat.mentionPatterns`, Fallback `messages.groupChat.mentionPatterns`)
    - ohne konfigurierte Muster kann die Erwähnungssperre nicht durchgesetzt werden
    - Steuerbefehle autorisierter Absender umgehen die Erwähnungssperre

    Gruppenbezogenes `systemPrompt`:

    Jeder Eintrag unter `channels.imessage.groups.*` akzeptiert eine optionale Zeichenfolge `systemPrompt`, die bei jedem Turn, der eine Nachricht in dieser Gruppe verarbeitet, in den System-Prompt des Agenten eingefügt wird. Die Auflösung entspricht `channels.whatsapp.groups`:

    1. **Gruppenspezifischer System-Prompt** (`groups["<chat_id>"].systemPrompt`): wird verwendet, wenn der spezifische Gruppeneintrag in der Zuordnung vorhanden **und** sein Schlüssel `systemPrompt` definiert ist. Wenn `systemPrompt` eine leere Zeichenfolge (`""`) ist, wird der Platzhalter unterdrückt und auf diese Gruppe kein System-Prompt angewendet.
    2. **System-Prompt für den Gruppenplatzhalter** (`groups["*"].systemPrompt`): wird verwendet, wenn der spezifische Gruppeneintrag in der Zuordnung vollständig fehlt oder wenn er vorhanden ist, aber keinen Schlüssel `systemPrompt` definiert.

    ```json5
    {
      channels: {
        imessage: {
          groupPolicy: "allowlist",
          groupAllowFrom: ["+15555550123"],
          groups: {
            "*": { systemPrompt: "Verwenden Sie britische Rechtschreibung." },
            "8421": {
              requireMention: true,
              systemPrompt: "Dies ist der Chat für die Rufbereitschaft. Begrenzen Sie Antworten auf höchstens 3 Sätze.",
            },
            "9907": {
              // explizite Unterdrückung: Der Platzhalter "Verwenden Sie britische Rechtschreibung." gilt hier nicht
              systemPrompt: "",
            },
          },
        },
      },
    }
    ```

    Gruppenbezogene Prompts gelten nur für Gruppennachrichten — Direktnachrichten sind davon nicht betroffen.

  </Tab>

  <Tab title="Sitzungen und deterministische Antworten">
    - Direktnachrichten verwenden direktes Routing; Gruppen verwenden Gruppen-Routing.
    - Mit dem standardmäßigen `session.dmScope=main` werden iMessage-Direktnachrichten in der Hauptsitzung des Agenten zusammengeführt.
    - Gruppensitzungen sind isoliert (`agent:<agentId>:imessage:group:<chat_id>`).
    - Antworten werden anhand der Metadaten des ursprünglichen Kanals und Ziels an iMessage zurückgeleitet.

    Verhalten gruppenähnlicher Threads:

    Einige iMessage-Threads mit mehreren Teilnehmenden können mit `is_group=false` eintreffen.
    Wenn dieser `chat_id` ausdrücklich unter `channels.imessage.groups` konfiguriert ist, behandelt OpenClaw ihn als Gruppenverkehr (Gruppensperren und Isolierung der Gruppensitzung).

  </Tab>
</Tabs>

## ACP-Konversationsbindungen

iMessage-Chats können an ACP-Sitzungen gebunden werden.

Schneller Ablauf für Betriebspersonal:

- Führen Sie `/acp spawn codex --bind here` innerhalb der Direktnachricht oder des zulässigen Gruppenchats aus.
- Künftige Nachrichten in derselben iMessage-Konversation werden an die erzeugte ACP-Sitzung weitergeleitet.
- `/new` und `/reset` setzen dieselbe gebundene ACP-Sitzung direkt zurück.
- `/acp close` schließt die ACP-Sitzung und entfernt die Bindung.

Konfigurierte persistente Bindungen verwenden Einträge der obersten Ebene `bindings[]` mit `type: "acp"` und `match.channel: "imessage"`.

`match.peer.id` kann Folgendes verwenden:

- normalisiertes Handle für Direktnachrichten wie `+15555550123` oder `user@example.com`
- `chat_id:<id>` (für stabile Gruppenbindungen empfohlen)
- `chat_guid:<guid>`
- `chat_identifier:<identifier>`

Beispiel:

```json5
{
  agents: {
    list: [
      {
        id: "codex",
        runtime: {
          type: "acp",
          acp: { agent: "codex", backend: "acpx", mode: "persistent" },
        },
      },
    ],
  },
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "imessage",
        accountId: "default",
        peer: { kind: "group", id: "chat_id:123" },
      },
      acp: { label: "codex-group" },
    },
  ],
}
```

Informationen zum gemeinsamen Verhalten von ACP-Bindungen finden Sie unter [ACP-Agenten](/de/tools/acp-agents).

## Bereitstellungsmuster

<AccordionGroup>
  <Accordion title="Dedizierter macOS-Bot-Benutzer (separate iMessage-Identität)">
    Verwenden Sie eine dedizierte Apple-ID und einen dedizierten macOS-Benutzer, damit der Bot-Datenverkehr von Ihrem persönlichen Messages-Profil getrennt ist.

    Typischer Ablauf:

    1. Erstellen Sie einen dedizierten macOS-Benutzer bzw. melden Sie sich bei diesem an.
    2. Melden Sie sich in diesem Benutzerkonto mit der Apple-ID des Bots bei Messages an.
    3. Installieren Sie `imsg` in diesem Benutzerkonto.
    4. Erstellen Sie einen SSH-Wrapper, damit OpenClaw `imsg` im Kontext dieses Benutzers ausführen kann.
    5. Verweisen Sie `channels.imessage.accounts.<id>.cliPath` und `.dbPath` auf dieses Benutzerprofil.

    Beim ersten Start sind möglicherweise GUI-Genehmigungen (Automation + Full Disk Access) in der Sitzung dieses Bot-Benutzers erforderlich.

  </Accordion>

  <Accordion title="Entfernter Mac über Tailscale (Beispiel)">
    Übliche Topologie:

    - Gateway wird unter Linux/in einer VM ausgeführt
    - iMessage + `imsg` werden auf einem Mac in Ihrem Tailnet ausgeführt
    - Der `cliPath`-Wrapper verwendet SSH, um `imsg` auszuführen
    - `remoteHost` aktiviert das Abrufen von Anhängen per SCP

    Beispiel:

    ```json5
    {
      channels: {
        imessage: {
          enabled: true,
          cliPath: "~/.openclaw/scripts/imsg-ssh",
          remoteHost: "bot@mac-mini.tailnet-1234.ts.net",
          includeAttachments: true,
          dbPath: "/Users/bot/Library/Messages/chat.db",
        },
      },
    }
    ```

    ```bash
    #!/usr/bin/env bash
    exec ssh -T bot@mac-mini.tailnet-1234.ts.net imsg "$@"
    ```

    Verwenden Sie SSH-Schlüssel, damit sowohl SSH als auch SCP nicht interaktiv ausgeführt werden.
    Stellen Sie zuerst sicher, dass dem Hostschlüssel vertraut wird (zum Beispiel `ssh bot@mac-mini.tailnet-1234.ts.net`), damit `known_hosts` befüllt ist.

  </Accordion>

  <Accordion title="Muster für mehrere Konten">
    iMessage unterstützt eine kontospezifische Konfiguration unter `channels.imessage.accounts`.

    Jedes Konto kann Felder wie `cliPath`, `dbPath`, `allowFrom`, `groupPolicy`, `mediaMaxMb`, Verlaufseinstellungen und Positivlisten für Anhangsstammverzeichnisse überschreiben.

  </Accordion>

  <Accordion title="Direktnachrichtenverlauf">
    Legen Sie `channels.imessage.dmHistoryLimit` fest, um neue Direktnachrichtensitzungen mit dem kürzlich decodierten `imsg`-Verlauf dieser Unterhaltung zu initialisieren. Verwenden Sie `channels.imessage.dms["<sender>"].historyLimit` für absenderspezifische Überschreibungen, einschließlich `0`, um den Verlauf für einen Absender zu deaktivieren.

    Der iMessage-Direktnachrichtenverlauf wird bei Bedarf aus `imsg` abgerufen. Wenn `dmHistoryLimit` nicht festgelegt ist, ist die globale Initialisierung des Direktnachrichtenverlaufs deaktiviert; ein positiver absenderspezifischer Wert für `channels.imessage.dms["<sender>"].historyLimit` aktiviert die Initialisierung für diesen Absender jedoch weiterhin.

  </Accordion>
</AccordionGroup>

## Medien, Aufteilung und Zustellungsziele

<AccordionGroup>
  <Accordion title="Anhänge und Medien">
    - Die Verarbeitung eingehender Anhänge ist **standardmäßig deaktiviert** — legen Sie `channels.imessage.includeAttachments: true` fest, um Fotos, Sprachnotizen, Videos und andere Anhänge an den Agenten weiterzuleiten. Ist sie deaktiviert, werden iMessages, die nur Anhänge enthalten, verworfen, bevor sie den Agenten erreichen, und erzeugen möglicherweise überhaupt keine `Inbound message`-Protokollzeile.
    - Entfernte Anhangspfade können per SCP abgerufen werden, wenn `remoteHost` festgelegt ist
    - Anhangspfade müssen zulässigen Stammverzeichnissen entsprechen:
      - `channels.imessage.attachmentRoots` (lokal)
      - `channels.imessage.remoteAttachmentRoots` (entfernter SCP-Modus)
      - Konfigurierte Stammverzeichnisse erweitern das standardmäßige Stammverzeichnismuster `/Users/*/Library/Messages/Attachments` (zusammengeführt, nicht ersetzt)
    - SCP verwendet eine strikte Hostschlüsselprüfung (`StrictHostKeyChecking=yes`)
    - Die Größe ausgehender Medien verwendet `channels.imessage.mediaMaxMb` (Standard: 16 MB)

  </Accordion>

  <Accordion title="Ausgehender Text und Aufteilung">
    - Textabschnittslimit: `channels.imessage.textChunkLimit` (Standard: 4000)
    - Aufteilungsmodus: `channels.imessage.streaming.chunkMode`
      - `length` (Standard)
      - `newline` (bevorzugte Aufteilung nach Absätzen)
    - Ausgehende Markdown-Formatierung für Fett, Kursiv, Unterstrichen und Durchgestrichen wird in nativ formatierten Text umgewandelt (Empfänger mit macOS 15+ stellen die Formatierung dar; ältere Empfänger sehen einfachen Text ohne die Markierungen); Markdown-Tabellen werden gemäß dem Markdown-Tabellenmodus des Kanals umgewandelt
    - `channels.imessage.sendTransport` (Standard: `auto`, `bridge`, `applescript`) bestimmt, wie `imsg` Nachrichten zustellt

  </Accordion>

  <Accordion title="Adressierungsformate">
    Bevorzugte explizite Ziele:

    - `chat_id:123` (für stabiles Routing empfohlen)
    - `chat_guid:...`
    - `chat_identifier:...`

    Handle-Ziele werden ebenfalls unterstützt:

    - `imessage:+1555...`
    - `sms:+1555...`
    - `user@example.com`

    ```bash
    imsg chats --limit 20
    ```

  </Accordion>
</AccordionGroup>

## Aktionen der privaten API

Wenn `imsg launch` ausgeführt wird und `openclaw channels status --probe` den Wert `privateApi.available: true` meldet, kann das Nachrichten-Tool zusätzlich zum normalen Textversand iMessage-native Aktionen verwenden.

Alle Aktionen sind standardmäßig aktiviert; verwenden Sie `channels.imessage.actions`, um einzelne Aktionen zu deaktivieren:

```json5
{
  channels: {
    imessage: {
      actions: {
        reactions: true,
        edit: true,
        unsend: true,
        reply: true,
        sendWithEffect: true,
        sendAttachment: true,
        renameGroup: true,
        setGroupIcon: true,
        addParticipant: true,
        removeParticipant: true,
        leaveGroup: true,
        polls: true,
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="Verfügbare Aktionen">
    - **react**: iMessage-Tapbacks hinzufügen/entfernen (`messageId`, `emoji`, `remove`). Unterstützte Tapbacks werden „Liebe“, „Gefällt mir“, „Gefällt mir nicht“, „Lachen“, „Hervorheben“ und „Frage“ zugeordnet. Beim Entfernen ohne Emoji wird das jeweils gesetzte Tapback gelöscht.
    - **reply**: Eine Antwort in einem Thread auf eine vorhandene Nachricht senden (`messageId`, `text` oder `message` sowie `chatGuid`, `chatId`, `chatIdentifier` oder `to`). Für eine Antwort mit Anhang ist zusätzlich ein `imsg`-Build erforderlich, dessen `send-rich` `--file` unterstützt.
    - **sendWithEffect**: Text mit einem iMessage-Effekt senden (`text` oder `message`, `effect` oder `effectId`). Kurznamen: slam, loud, gentle, invisibleink, confetti, lasers, fireworks, balloon, heart, echo, happybirthday, shootingstar, sparkles, spotlight.
    - **edit**: Eine gesendete Nachricht unter unterstützten macOS-/privaten API-Versionen bearbeiten (`messageId`, `text` oder `newText`). Nur Nachrichten, die das Gateway selbst gesendet hat, können bearbeitet werden.
    - **unsend**: Eine gesendete Nachricht unter unterstützten macOS-/privaten API-Versionen zurückziehen (`messageId`). Nur Nachrichten, die das Gateway selbst gesendet hat, können zurückgezogen werden.
    - **upload-file**: Medien/Dateien senden (`buffer` als Base64 oder ein hydratisiertes `media`/`path`/`filePath`, `filename`, optional `asVoice`). Legacy-Alias: `sendAttachment`.
    - **renameGroup**, **setGroupIcon**, **addParticipant**, **removeParticipant**, **leaveGroup**: Gruppenchats verwalten, wenn das aktuelle Ziel eine Gruppenunterhaltung ist. Diese Aktionen verändern die Messages-Identität des Hosts und erfordern daher einen Eigentümer als Absender oder einen `operator.admin`-Gateway-Client.
    - **poll**: Eine native Umfrage in Apple Messages erstellen (`pollQuestion`, `pollOption` 2- bis 12-mal wiederholt sowie `chatGuid`, `chatId`, `chatIdentifier` oder `to`). Empfänger mit iOS/iPadOS/macOS 26+ sehen die Umfrage nativ und können nativ abstimmen; ältere Betriebssystemversionen erhalten ersatzweise den Text „Sent a poll“. Erfordert `selectors.pollPayloadMessage`.
    - **poll-vote**: Über eine vorhandene Umfrage abstimmen (`pollId` oder `messageId` sowie genau einen der Werte `pollOptionIndex`, `pollOptionId` oder `pollOptionText`). Erfordert `selectors.pollVoteMessage` und die RPC-Methode `poll.vote`.

    Akzeptierte eingehende Umfragen werden für den Agenten mit der Frage, nummerierten Optionsbezeichnungen, Stimmenzahlen und der für `poll-vote` benötigten Umfragenachrichten-ID dargestellt.

  </Accordion>

  <Accordion title="Nachrichten-IDs">
    Der Kontext eingehender iMessages enthält sowohl kurze `MessageSid`-Werte als auch vollständige Nachrichten-GUIDs (`MessageSidFull`), sofern verfügbar. Kurze IDs gelten nur im aktuellen SQLite-basierten Antwortcache und werden vor der Verwendung mit dem aktuellen Chat abgeglichen. Wenn eine kurze ID abläuft, versuchen Sie es erneut mit ihrem `MessageSidFull` und geben Sie dabei die Unterhaltung als Ziel an, aus der die ID stammt. Vollständige IDs umgehen die Bindung an Unterhaltung oder Konto nicht. Ersetzen Sie daher eine ID aus einem anderen Chat durch eine ID des aktuellen Ziels. Entfernt delegierte Aufrufe können veraltete vollständige IDs ablehnen, wenn keine Nachweise für die aktuelle Unterhaltung verfügbar sind.

  </Accordion>

  <Accordion title="Funktionserkennung">
    OpenClaw blendet Aktionen der privaten API nur aus, wenn der zwischengespeicherte Prüfstatus angibt, dass die Bridge nicht verfügbar ist. Ist der Status unbekannt, bleiben die Aktionen sichtbar und führen Prüfungen beim Dispatch verzögert durch, sodass die erste Aktion nach `imsg launch` ohne separate manuelle Statusaktualisierung erfolgreich sein kann.

  </Accordion>

  <Accordion title="Lesebestätigungen und Tippanzeige">
    Wenn die Bridge der privaten API aktiv ist, werden akzeptierte eingehende Chats als gelesen markiert, und Direktchats zeigen eine Tippblase an, sobald der Turn akzeptiert wurde, während der Agent den Kontext vorbereitet und Inhalte generiert. Deaktivieren Sie die Lesemarkierung mit:

    ```json5
    {
      channels: {
        imessage: {
          sendReadReceipts: false,
        },
      },
    }
    ```

    Ältere `imsg`-Builds, die vor der Liste der methodenspezifischen Fähigkeiten entstanden sind, deaktivieren Tippanzeige und Lesebestätigungen stillschweigend; OpenClaw protokolliert einmal pro Neustart eine Warnung, damit die fehlende Bestätigung zugeordnet werden kann.

  </Accordion>

  <Accordion title="Eingehende Tapbacks">
    OpenClaw abonniert iMessage-Tapbacks und leitet akzeptierte Reaktionen als Systemereignisse statt als normalen Nachrichtentext weiter, sodass ein Benutzer-Tapback keine gewöhnliche Antwortschleife auslöst.

    Der Benachrichtigungsmodus wird durch `channels.imessage.reactionNotifications` gesteuert:

    - `"own"` (Standard): Nur benachrichtigen, wenn Benutzer auf vom Bot verfasste Nachrichten reagieren.
    - `"all"`: Bei allen eingehenden Tapbacks von autorisierten Absendern benachrichtigen.
    - `"off"`: Eingehende Tapbacks ignorieren.

    Kontospezifische Überschreibungen verwenden `channels.imessage.accounts.<id>.reactionNotifications`.

  </Accordion>

  <Accordion title="Genehmigungsreaktionen (👍 / 👎)">
    Wenn `approvals.exec.enabled` oder `approvals.plugin.enabled` auf „true“ gesetzt ist und die Anfrage an iMessage weitergeleitet wird, stellt das Gateway eine Genehmigungsaufforderung nativ zu und akzeptiert ein Tapback, um sie zu entscheiden:

    - `👍` („Gefällt mir“-Tapback) → `allow-once`
    - `👎` („Gefällt mir nicht“-Tapback) → `deny`
    - `allow-always` bleibt als manuelle Ausweichlösung verfügbar: Senden Sie `/approve <id> allow-always` als normale Antwort.

    Für die Verarbeitung von Reaktionen muss das Handle des reagierenden Benutzers explizit als Genehmiger eingetragen sein. Die Genehmigerliste wird aus `channels.imessage.allowFrom` (oder `channels.imessage.accounts.<id>.allowFrom`) gelesen. Fügen Sie die Telefonnummer des Benutzers im E.164-Format oder seine Apple-ID-E-Mail-Adresse hinzu (Chatziele wie `chat_id:*` sind keine gültigen Genehmigereinträge). Der Platzhaltereintrag `"*"` wird berücksichtigt, ermöglicht jedoch jedem Absender die Genehmigung; eine leere Genehmigerliste deaktiviert die Reaktionsverknüpfung vollständig. Die Reaktionsverknüpfung umgeht absichtlich `reactionNotifications`, `dmPolicy` und `groupAllowFrom`, da die explizite Genehmiger-Positivliste die einzige relevante Zugriffskontrolle für Genehmigungsentscheidungen ist.

    Die Autorisierung des Textbefehls `/approve` folgt derselben Liste: Wenn `channels.imessage.allowFrom` nicht leer ist, wird `/approve <id> <decision>` anhand dieser Genehmigerliste autorisiert (nicht anhand der umfassenderen Direktnachrichten-Positivliste), und Absender, die in der Direktnachrichten-Positivliste zugelassen sind, aber nicht in `allowFrom` stehen, erhalten eine ausdrückliche Ablehnung. Wenn `allowFrom` leer ist, bleibt die Ausweichregel für denselben Chat aktiv und `/approve` autorisiert alle Personen, die gemäß der Direktnachrichten-Positivliste zugelassen sind. Fügen Sie jeden Bediener, der Genehmigungen erteilen soll — über `/approve` oder über Reaktionen — zu `allowFrom` hinzu.

    Hinweise für Betreiber:
    - Die Reaktionszuordnung wird sowohl im Arbeitsspeicher als auch im persistenten schlüsselbasierten Speicher des Gateways gespeichert (die TTL entspricht dem Ablaufzeitpunkt der Genehmigung). Außerdem fragt das Gateway ausstehende Aufforderungen nach Tapbacks ab, sodass ein Tapback, das kurz nach einem Neustart des Gateways eintrifft, die Genehmigung weiterhin auflöst.
    - Das eigene `is_from_me=true`-Tapback des Betreibers (beispielsweise von einem gekoppelten Apple-Gerät) löst die Genehmigung auf, wenn dieser Handle ausdrücklich als Genehmiger festgelegt ist.
    - Genehmigungsaufforderungen werden nur dann an eine Gruppenunterhaltung weitergeleitet, wenn ausdrückliche Genehmiger konfiguriert sind; andernfalls könnte jedes Gruppenmitglied die Genehmigung erteilen.
    - Ältere Tapbacks im Textformat (`Liked "…"`-Klartext von sehr alten Apple-Clients) können Genehmigungen nicht auflösen, da sie keine Nachrichten-GUID enthalten; die Reaktionsauflösung erfordert die strukturierten Tapback-Metadaten, die aktuelle macOS-/iOS-Clients ausgeben.

  </Accordion>

  <Accordion title="Reaktionen auf Fragen (1️⃣ / 2️⃣ / 3️⃣ / 4️⃣)">
    Bei einer `ask_user`-Aufforderung mit einer einzelnen, nicht geheimen Einfachauswahlfrage und ein bis vier Optionen fügt OpenClaw nummerierte Emoji-Auswahlmöglichkeiten hinzu. Reagieren Sie auf die zugestellte Aufforderung mit der entsprechenden Zahl, um sie zu beantworten. Die Reaktion muss die stabile GUID der vom Bot verfassten Nachricht enthalten; OpenClaw ordnet die Zahl dann über das Gateway der kanonischen Option zu. Veraltete oder doppelte Eingaben werden ignoriert.

    Aufforderungen mit mehreren Fragen, Mehrfachauswahl oder Freitext können weiterhin nur per Textantwort beantwortet werden. Reaktionen auf Fragen unterliegen den normalen Zulassungsregeln für iMessage-Direktnachrichten und -Gruppen. Sie werden auch erkannt, wenn das allgemeine `reactionNotifications` auf `"off"` gesetzt ist, ohne dass dadurch nicht zugehörige Reaktionen zu Agentenereignissen werden.

  </Accordion>
</AccordionGroup>

## Konfigurationsänderungen

iMessage erlaubt standardmäßig vom Kanal initiierte Konfigurationsänderungen (für `/config set|unset`, wenn `commands.config: true`).

Deaktivieren:

```json5
{
  channels: {
    imessage: {
      configWrites: false,
    },
  },
}
```

<a id="coalescing-split-send-dms-command--url-in-one-composition"></a>

## Zusammenführen aufgeteilt gesendeter Direktnachrichten (Befehl + URL in einer Eingabe)

Apple kann einen Befehl und dessen URL-Vorschau als separate physische `chat.db`-Zeilen speichern. `imsg` ab Version 0.13.1 führt diese Zeilen zusammen, bevor Überwachung, Verlauf oder Suche die Nachricht zurückgeben. Dadurch empfängt OpenClaw eine einzige logische eingehende Nachricht, ohne eine kanalspezifische Latenz für Direktnachrichten hinzuzufügen.

Für iMessage ist keine Einstellung zum Zusammenführen erforderlich. Der außer Betrieb genommene Schlüssel `channels.imessage.coalesceSameSenderDms` wird von `openclaw doctor --fix` entfernt. Die allgemeine `messages.inbound`-Entprellung bleibt verfügbar, wenn Sie absichtlich schnell aufeinanderfolgende Textnachrichten eines Kanals bündeln möchten.

Wenn Sendungen aus Befehl und URL als separate Agentendurchläufe eintreffen, aktualisieren Sie `imsg` auf dem Messages-Mac:

```bash
brew update && brew upgrade imsg
```

## Wiederherstellung eingehender Nachrichten nach einem Neustart der Bridge oder des Gateways

iMessage stellt Nachrichten wieder her, die während des Ausfalls des Gateways verpasst wurden, und unterdrückt gleichzeitig die veraltete „Backlog-Bombe“, die Apple nach einer Push-Wiederherstellung ausgeben kann. Das auf dauerhaftem Eingang und einer Altersgrenze basierende Standardverhalten ist immer aktiviert.

- **Dauerhafter Schutz vor Wiederholungen.** Bevor OpenClaw den Wiederherstellungscursor fortschreibt, protokolliert es jede Rohzeile in der gemeinsamen SQLite-Eingangswarteschlange und verwendet deren Apple-GUID als Ereignis-ID. Eine abgeschlossene Zeile hinterlässt für etwa 4 Stunden einen Tombstone, wobei die Anzahl auf 10.000 Einträge begrenzt ist. Dadurch wird eine Wiederholung mit derselben GUID auch nach einem Neustart verworfen. Eine ausstehende Zeile bleibt wiederherstellbar, bis sie vom Versand übernommen wird.
- **Wiederherstellung nach Ausfallzeiten.** Beim Start merkt sich der Monitor die Zeilen-ID der letzten dauerhaft angenommenen `chat.db`-Zeile (einen persistenten Cursor pro Konto) und übergibt sie als `since_rowid` an `imsg watch.subscribe`. Dadurch gibt imsg Zeilen erneut wieder, die noch nicht protokolliert wurden, und folgt anschließend den Live-Ereignissen. Vor einem Absturz protokollierte Zeilen werden aus SQLite fortgesetzt. Die Wiederholung ist auf die neuesten 500 Zeilen und auf Nachrichten mit einem Alter von bis zu etwa 2 Stunden begrenzt; GUID-Tombstones verwerfen alle bereits verarbeiteten Einträge.
- **Altersgrenze für veraltete Rückstände.** Zeilen oberhalb der Startgrenze sind tatsächlich live; liegt das Sendedatum einer solchen Zeile mehr als etwa 15 Minuten vor ihrem Eingang, handelt es sich um den durch Push ausgegebenen Rückstand, der unterdrückt wird. Wiederholte Zeilen (an oder unterhalb der Grenze) verwenden stattdessen das größere Wiederherstellungsfenster. Dadurch wird eine kürzlich verpasste Nachricht zugestellt, während alte Verlaufsdaten nicht zugestellt werden.

Die Wiederherstellung funktioniert sowohl bei lokalen als auch bei entfernten `cliPath`-Konfigurationen, da die `since_rowid`-Wiederholung über dieselbe `imsg`-RPC-Verbindung ausgeführt wird. Der Unterschied liegt im Zeitfenster: Wenn das Gateway `chat.db` lesen kann (lokal), verankert es die Startgrenze der Zeilen-ID, begrenzt den Wiederholungsumfang und stellt verpasste Nachrichten mit einem Alter von bis zu einigen Stunden zu. Über eine entfernte SSH-`cliPath` kann es die Datenbank nicht lesen. Daher ist die Wiederholung unbegrenzt und jede Zeile verwendet die Live-Altersgrenze. Kürzlich verpasste Nachrichten werden weiterhin wiederhergestellt und alte Rückstände weiterhin unterdrückt, jedoch mit dem kleineren Live-Zeitfenster. Führen Sie das Gateway auf dem Messages-Mac aus, um das größere Wiederherstellungsfenster zu verwenden.

### Für Betreiber sichtbares Signal

Unterdrückte Rückstände werden auf der Standardprotokollierungsstufe protokolliert und niemals stillschweigend verworfen (das Flag `recovery` zeigt, welches Zeitfenster angewendet wurde):

```text
imessage: veralteter eingehender Rückstand unterdrückt account=<id> sent=<iso> recovery=<bool> (<N> seit dem Start unterdrückt)
```

### Migration

`channels.imessage.catchup.*` ist veraltet – die Wiederherstellung nach Ausfallzeiten erfolgt automatisch und benötigt bei neuen Konfigurationen keine Konfiguration. Vorhandene Konfigurationen mit `catchup.enabled: true` werden weiterhin als Kompatibilitätsprofil für das Wiederholungsfenster der Wiederherstellung berücksichtigt. Deaktivierte Catchup-Blöcke (`enabled: false` oder ohne `enabled: true`) wurden außer Betrieb genommen; `openclaw doctor --fix` entfernt sie.

## Fehlerbehebung

<AccordionGroup>
  <Accordion title="imsg nicht gefunden oder RPC nicht unterstützt">
    Prüfen Sie das Binärprogramm und die RPC-Unterstützung:

    ```bash
    imsg rpc --help
    imsg status --json
    openclaw channels status --probe
    ```

    Wenn die Prüfung meldet, dass RPC nicht unterstützt wird, aktualisieren Sie `imsg`. Wenn Aktionen der privaten API nicht verfügbar sind, führen Sie `imsg launch` in der Sitzung des angemeldeten macOS-Benutzers aus und prüfen Sie erneut. Wenn das Gateway nicht unter macOS ausgeführt wird, verwenden Sie statt des lokalen Standardpfads `imsg` die oben beschriebene Konfiguration eines entfernten Macs über SSH.

  </Accordion>

  <Accordion title="Nachrichten werden gesendet, aber eingehende iMessages kommen nicht an">
    Prüfen Sie zunächst, ob die Nachricht den lokalen Mac erreicht hat. Wenn sich `chat.db` nicht ändert, kann OpenClaw die Nachricht nicht empfangen, selbst wenn `imsg status --json` eine funktionsfähige Bridge meldet.

```bash
imsg chats --limit 10 --json
imsg watch --chat-id <chat-id> --json
sqlite3 ~/Library/Messages/chat.db \
  "select datetime(max(date)/1000000000 + 978307200, 'unixepoch', 'localtime'), max(ROWID) from message;"
```

    Wenn vom Telefon gesendete Nachrichten keine neuen Zeilen erzeugen, reparieren Sie die macOS-Ebenen Messages und Apple Push, bevor Sie die OpenClaw-Konfiguration ändern. Eine einmalige Aktualisierung der Dienste reicht häufig aus:

```bash
launchctl kickstart -k system/com.apple.apsd
launchctl kickstart -k gui/$(id -u)/com.apple.CommCenter
launchctl kickstart -k gui/$(id -u)/com.apple.identityservicesd
launchctl kickstart -k gui/$(id -u)/com.apple.imagent
imsg launch
openclaw gateway restart
```

    Senden Sie eine neue iMessage vom Telefon und bestätigen Sie eine neue `chat.db`-Zeile oder ein neues `imsg watch`-Ereignis, bevor Sie OpenClaw-Sitzungen untersuchen. Führen Sie dies nicht als regelmäßige Schleife zum Neustarten der Bridge aus; wiederholte `imsg launch`-Vorgänge zusammen mit Gateway-Neustarts während aktiver Arbeit können Zustellungen unterbrechen und laufende Kanaldurchläufe blockieren.

  </Accordion>

  <Accordion title="Das Gateway wird nicht unter macOS ausgeführt">
    Der standardmäßige `cliPath: "imsg"` muss auf dem Mac ausgeführt werden, der bei Messages angemeldet ist. Setzen Sie unter Linux oder Windows `channels.imessage.cliPath` auf ein Wrapper-Skript, das per SSH eine Verbindung zu diesem Mac herstellt und `imsg "$@"` ausführt.

```bash
#!/usr/bin/env bash
exec ssh -T messages-mac imsg "$@"
```

    Führen Sie anschließend Folgendes aus:

```bash
openclaw channels status --probe --channel imessage
```

  </Accordion>

  <Accordion title="Direktnachrichten werden ignoriert">
    Prüfen Sie:

    - `channels.imessage.dmPolicy`
    - `channels.imessage.allowFrom`
    - Kopplungsgenehmigungen (`openclaw pairing list imessage`)

  </Accordion>

  <Accordion title="Gruppennachrichten werden ignoriert">
    Prüfen Sie:

    - `channels.imessage.groupPolicy`
    - `channels.imessage.groupAllowFrom`
    - Verhalten der Zulassungsliste `channels.imessage.groups`
    - Konfiguration des Erwähnungsmusters (`agents.entries.*.groupChat.mentionPatterns`)

  </Accordion>

  <Accordion title="Entfernte Anhänge schlagen fehl">
    Prüfen Sie:

    - `channels.imessage.remoteHost`
    - `channels.imessage.remoteAttachmentRoots`
    - SSH-/SCP-Schlüsselauthentifizierung vom Gateway-Host
    - Der Hostschlüssel ist in `~/.ssh/known_hosts` auf dem Gateway-Host vorhanden
    - Lesbarkeit des entfernten Pfads auf dem Mac, auf dem Messages ausgeführt wird

  </Accordion>

  <Accordion title="macOS-Berechtigungsaufforderungen wurden übersehen">
    Führen Sie die Befehle erneut in einem interaktiven GUI-Terminal im selben Benutzer- und Sitzungskontext aus und genehmigen Sie die Aufforderungen:

    ```bash
    imsg chats --limit 1
    imsg send <handle> "test"
    ```

    Vergewissern Sie sich, dass für den Prozesskontext, in dem OpenClaw/`imsg` ausgeführt wird, Full Disk Access und Automation gewährt wurden.

  </Accordion>
</AccordionGroup>

## Verweise zur Konfigurationsreferenz

- [Konfigurationsreferenz – iMessage](/de/gateway/config-channels#imessage)
- [Gateway-Konfiguration](/de/gateway/configuration)
- [Kopplung](/de/channels/pairing)

## Verwandte Themen

- [Kanalübersicht](/de/channels) — alle unterstützten Kanäle
- [Entfernung von BlueBubbles und der imsg-iMessage-Pfad](/de/announcements/bluebubbles-imessage) — Ankündigung und Zusammenfassung der Migration
- [Umstieg von BlueBubbles](/de/channels/imessage-from-bluebubbles) — Tabelle zur Übertragung der Konfiguration und schrittweise Umstellung
- [Kopplung](/de/channels/pairing) — Authentifizierung von Direktnachrichten und Kopplungsablauf
- [Gruppen](/de/channels/groups) — Verhalten von Gruppenchats und Erwähnungsbeschränkung
- [Kanal-Routing](/de/channels/channel-routing) — Sitzungs-Routing für Nachrichten
- [Sicherheit](/de/gateway/security) — Zugriffsmodell und Härtung
