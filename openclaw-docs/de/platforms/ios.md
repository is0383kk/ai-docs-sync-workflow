---
read_when:
    - Koppeln oder erneutes Verbinden des iOS-Node
    - Direkten Apple-Watch-Node aktivieren oder Fehler beheben
    - Ausführen der iOS-App aus dem Quellcode
    - Fehlerbehebung bei der Gateway-Erkennung oder bei Canvas-Befehlen
summary: 'iOS-Node-App: Verbindung mit dem Gateway, Kopplung, Canvas und Fehlerbehebung'
title: iOS-App
x-i18n:
    generated_at: "2026-07-26T17:53:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2b01a63fa1e2c445f7fb35843536f7f5918e94bfe885dac19c852d7d52d86342
    source_path: platforms/ios.md
    workflow: 16
---

Verfügbarkeit: iPhone-App-Builds werden über Apple-Kanäle verteilt, wenn dies für ein Release aktiviert ist. Lokale Entwicklungs-Builds können auch aus dem Quellcode ausgeführt werden.

## Funktionsumfang

- Stellt über WebSocket (LAN oder Tailnet) eine Verbindung zu einem Gateway her.
- Stellt Node-Funktionen bereit: Canvas, Bildschirmaufnahme, Kameraaufnahme, Standort, Sprechmodus, Sprachaktivierung und optional aktivierte Gesundheitszusammenfassungen.
- Empfängt `node.invoke`-Befehle und meldet Node-Statusereignisse.
- Ermöglicht über die Agents-Oberfläche (Dateien) den schreibgeschützten Zugriff auf den Arbeitsbereich des ausgewählten Agenten: Navigation durch Verzeichnisse, Textvorschauen mit Syntaxhervorhebung, Bildvorschauen und Export über das Teilen-Menü. Keine Schreibvorgänge; die Größe der Vorschauen wird durch das Gateway begrenzt.
- Hält pro gekoppeltem Gateway einen kleinen schreibgeschützten Offline-Cache der letzten Chatsitzungen und Transkripte vor: Bei einem Kaltstart wird sofort das zuletzt bekannte Transkript angezeigt und aktualisiert, sobald das Gateway antwortet; kürzlich verwendete Chats bleiben auch ohne Verbindung durchsuchbar; Zurücksetzen/Entfernen löscht den geschützten lokalen Cache.
- Stellt Textnachrichten, die ohne Verbindung gesendet werden, in eine dauerhafte, Gateway-spezifische Ausgangswarteschlange (bis zu 50): Nachrichten in der Warteschlange werden im Transkript angezeigt, bei erneuter Verbindung der Reihe nach mit idempotenten Wiederholungsversuchen gesendet, bleiben dauerhaft gespeichert, bis der kanonische Verlauf den Versand bestätigt, werden mit zunehmender Verzögerung erneut versucht, bevor eine Aktion zum Wiederholen/Löschen angezeigt wird, und verfallen, statt nach 48 Stunden ohne Verbindung gesendet zu werden; Zurücksetzen/Entfernen löscht die Warteschlange zusammen mit dem Cache.
- Chat ist die zentrale Oberfläche für Text und Sprache. Über Chat-Aktionen kann der vollständige Sitzungsbildschirm geöffnet werden, ohne Chat zu verlassen, und die Schlussfolgerungen des Assistenten sowie Werkzeugaktivitäten können ein- oder ausgeblendet werden. Tippen Sie zur Diktierung eines Entwurfs auf das Mikrofon, öffnen Sie dessen Menü, um eine Sprachnachricht aufzunehmen, oder verwenden Sie das eingebettete Sprechsteuerelement für Sprache in Echtzeit; während des Zuhörens oder Sprechens wird das Sprechsteuerelement anhand des Live-Mikrofon- oder Wiedergabepegels animiert.
- **Einstellungen -> OpenClaw** öffnet einen speziellen Assistenten für Gateway-Einstellungen, wenn die Operatorverbindung über `operator.admin` verfügt und das Gateway `openclaw.chat` unterstützt. Die Einrichtungskonversation bleibt vom normalen Chat getrennt, schwärzt geheime Antworten lokal und wechselt erst zu Chat, nachdem Sie auf **Chat öffnen** getippt haben.
- Gibt Assistentennachrichten auf Anforderung gesprochen wieder: Drücken Sie im Chat lange auf eine Nachricht und wählen Sie **Anhören**. Die App spielt unterstützte `tts.speak`-Clips des Gateways mit dem konfigurierten TTS-Provider ab und verwendet ersatzweise die Sprachausgabe auf dem Gerät, wenn Gateway-Audio nicht verfügbar oder nicht abspielbar ist. Die Wiedergabe wird beim Wechsel der Sitzung oder beim Verschieben in den Hintergrund beendet.

## Voraussetzungen

- Ein Gateway, das auf einem anderen Gerät ausgeführt wird (macOS, Linux oder Windows über WSL2).
- Netzwerkpfad:
  - Dasselbe LAN über Bonjour, **oder**
  - Tailnet über Unicast-DNS-SD (Beispieldomain: `openclaw.internal.`), **oder**
  - Manuell angegebener Host/Port (Ausweichlösung).

## Schnellstart (koppeln und verbinden)

Beim ersten Start führt die App durch eine kurze Erklärung zur Kopplung und eine
Berechtigungsseite (Mitteilungen, Kamera, Mikrofon, Fotos, Kontakte,
Kalender, Erinnerungen, Standort). Jede Berechtigung ist optional und kann
später unter **Einstellungen** -> **Berechtigungen** oder in der iOS-App „Einstellungen“
geändert werden.

1. Starten Sie ein authentifiziertes Gateway mit einer Route, die Ihr Telefon erreichen kann. Tailscale
   Serve ist der empfohlene Remote-Pfad:

```bash
openclaw gateway --port 18789 --tailscale serve
```

Verwenden Sie für eine vertrauenswürdige Einrichtung im selben LAN stattdessen ein authentifiziertes `gateway.bind: "lan"`.
Die standardmäßige Bindung an die Loopback-Schnittstelle ist von einem Telefon aus nicht erreichbar. Wenn das
Gateway noch nicht konfiguriert wurde, führen Sie zuerst `openclaw onboard` aus, damit für die Erstellung
des Einrichtungscodes ein Authentifizierungspfad mit Token oder Passwort vorhanden ist.

2. Öffnen Sie die [Control UI](/de/web/control-ui), wählen Sie **Nodes** aus und klicken Sie
   auf der Seite **Devices** auf **Pair mobile device**. Vollzugriff wird empfohlen
   und ist standardmäßig ausgewählt; wählen Sie Limited access nur aus, wenn Sie
   administrative Gateway-Steuerelemente ausschließen möchten, und klicken Sie dann auf **Create setup code**.

3. Öffnen Sie in der iOS-App **Einstellungen** -> **Gateway**, scannen Sie den QR-Code (oder fügen Sie
   den Einrichtungscode ein) und stellen Sie die Verbindung her.

   Wenn der Einrichtungscode sowohl LAN- als auch Tailscale-Serve-Routen enthält, prüft die App
   diese der Reihe nach und speichert den ersten erreichbaren Endpunkt.

   Gekoppelte Gateways bleiben in der Liste **Gateways**. Das Häkchen kennzeichnet
   das fokussierte Gateway; verwenden Sie das Blitz-Steuerelement in einer anderen Zeile, um dessen
   Operatorsitzung gleichzeitig verbunden zu halten. Beim Wechseln des Fokus werden
   andere aktivierte Gateways nicht getrennt. Nur das fokussierte Gateway erhält die
   funktionsführende Node-Sitzung des iPhones, sodass Kamera-, Bildschirm-, Standort- und
   andere Gerätebefehle stets genau einen eindeutigen Besitzer haben. iOS kann
   diese Vordergrundverbindungen aussetzen, nachdem die App in den Hintergrund gewechselt ist.

4. Die offizielle App stellt die Verbindung automatisch her. Wenn **Pending approval** eine
   Anfrage anzeigt, prüfen Sie deren Rolle und Geltungsbereiche, bevor Sie sie genehmigen.

   **Einstellungen → Gateway** zeigt an, ob die gespeicherte Operatorverbindung über
   **Vollzugriff** oder **Eingeschränkten Zugriff** verfügt. Eine Klartext-LAN-Einrichtung mit `ws://` wird zur
   Sicherheit des Bearer-Tokens automatisch eingeschränkt. Wenn sie eingeschränkt ist, konfigurieren Sie `wss://` oder
   Tailscale Serve, scannen Sie einen neuen Code für Vollzugriff aus der Control UI oder `openclaw qr`
   und stellen Sie anschließend erneut eine Verbindung her, um Einstellungen und Upgrades zu aktivieren.

Die Schaltfläche der Control UI erfordert eine bereits gekoppelte Sitzung mit `operator.admin`.
Wählen Sie als Terminal-Ausweichlösung in der iOS-App ein gefundenes Gateway aus (oder aktivieren Sie
Manual Host und geben Sie Host/Port ein) und genehmigen Sie anschließend die Anfrage auf dem Gateway-Host:

```bash
openclaw devices list
openclaw devices approve <requestId>
```

Wenn die App die Kopplung mit geänderten Authentifizierungsdetails (Rolle/Geltungsbereiche/öffentlicher Schlüssel) erneut versucht, wird die vorherige ausstehende Anfrage ersetzt und ein neuer `requestId` erstellt. Führen Sie vor der Genehmigung erneut `openclaw devices list` aus.

Optional: Wenn der iOS-Node immer aus einem streng kontrollierten Subnetz eine Verbindung herstellt, können Sie die automatische erstmalige Node-Genehmigung mit expliziten CIDRs oder exakten IP-Adressen aktivieren:

```json5
{
  gateway: {
    nodes: {
      pairing: {
        autoApproveCidrs: ["192.168.1.0/24"],
      },
    },
  },
}
```

Dies ist standardmäßig deaktiviert. Es gilt nur für eine neue `role: node`-Kopplung ohne angeforderte Geltungsbereiche. Die Kopplung von Operatoren/Browsern sowie jede Änderung an Rolle, Geltungsbereich, Metadaten oder öffentlichem Schlüssel erfordert weiterhin eine manuelle Genehmigung.

5. Überprüfen Sie die Verbindung:

```bash
openclaw nodes status
openclaw gateway call node.list --params "{}"
```

## Gesundheitszusammenfassungen

Der iOS-Node kann ein optional aktiviertes, schreibgeschütztes HealthKit-Aggregat für den aktuellen
Kalendertag zurückgeben. Die Zustimmung auf dem iOS-Gerät und die ausdrückliche Autorisierung des Gateway-Befehls sind
unabhängige Voraussetzungen. Informationen zu Einrichtung, Aufruf, Nutzlastfeldern, Datenschutzverhalten
und Fehlerbehebung finden Sie unter [HealthKit-Zusammenfassungen](/de/platforms/ios-healthkit).

Standardmäßig verwendet die Apple-Watch-Begleit-App weiterhin das vorhandene iPhone-Relay und
benötigt keine separate Gateway-Kopplung. Koppeln Sie die Watch in
Apples Watch-App mit dem iPhone, installieren Sie OpenClaw über **Watch app -> My Watch -> Available
Apps** und öffnen Sie OpenClaw anschließend einmal auf beiden Geräten.

## Genehmigungen für Befehle prüfen

Eine Operatorverbindung mit `operator.admin` oder eine gekoppelte,
vom Gateway ausdrücklich als Ziel festgelegte `operator.approvals`-Verbindung kann ausstehende
Ausführungsanfragen auf dem iPhone prüfen. Die Genehmigungskarte zeigt die
bereinigte Befehlsvorschau, Warnung, den Host-Kontext und die Ablaufzeit des Gateways sowie ausschließlich die
von der Anfrage angebotenen Entscheidungen. Die gekoppelte Apple Watch erhält über das vorhandene
iPhone-Relay dieselbe für Prüfer geeignete Aufforderung und bietet die kompakte
Entscheidungsauswahl „einmal zulassen/ablehnen“. Im direkten Watch-Gateway-Modus werden keine
Genehmigungsaufforderungen übertragen.

Der Genehmigungsstatus wird mit der Control UI und unterstützten Chat-Oberflächen geteilt. Die
erste verbindlich übermittelte Antwort gilt. iPhone und Watch rufen den kanonischen
Terminaldatensatz des Gateways ab, nachdem eine andere Oberfläche die Anfrage bearbeitet hat, nach einer externen
Benachrichtigung über die Bearbeitung und immer dann, wenn eine Bestätigung der Bearbeitung möglicherweise
verloren gegangen ist. Aktionen bleiben nicht verfügbar, bis dieser erneute Abruf bestätigt, ob die
Anfrage weiterhin aussteht.

Die Zuständigkeit für Genehmigungen ist an das ausgewählte Gateway gebunden. Durch einen Gateway-Wechsel kann
eine alte Aufforderung nicht auf die neue Verbindung angewendet werden. Gateways, die älter als die
vereinheitlichten Genehmigungsmethoden sind, greifen auf die ausgelieferten ausführungsspezifischen Methoden zurück;
für beibehaltenen Terminalstatus und umfangreichere oberflächenübergreifende Ergebnisse ist ein aktualisiertes
Gateway erforderlich.

## Fragen des Agenten beantworten

Chat zeigt ausstehende Gateway-Fragen als native Karten für Operatorverbindungen
mit `operator.questions` (oder `operator.admin`) an. Die Karten unterstützen Optionen mit Einfach- und
Mehrfachauswahl, Optionsbeschreibungen, Freitextantworten unter **Sonstiges** und einen
Countdown bis zum Ablauf. Bei einer erneuten Verbindung werden ausstehende Fragen vom Gateway neu geladen. Eine Karte
wird gesperrt, wenn dieses Gerät sie beantwortet, eine andere Oberfläche sie zuerst beantwortet oder die
Frage abläuft oder abgebrochen wird.

## Optionaler direkter Apple-Watch-Node

Im direkten Modus erhält die Watch eine eigene signierte Node-Identität und Gateway-Verbindung.
Unterstützte Node-Befehle funktionieren weiterhin über WLAN oder Mobilfunk der Watch, während
OpenClaw aktiv ist, selbst wenn das gekoppelte iPhone nicht verfügbar ist.

Voraussetzungen:

- Das iPhone ist mit dem Geltungsbereich `operator.admin` mit dem Gateway verbunden.
- Der Einrichtungscode kündigt einen `wss://`-Gateway-Endpunkt mit einem von watchOS als vertrauenswürdig eingestuften Zertifikat an;
  die Watch fragt den zugehörigen `https://`-Ursprung ab. Reines HTTP und
  selbstsigniertes oder ausschließlich auf Fingerabdrücken basierendes Vertrauen werden nicht unterstützt. Informationen zur Endpunktkonfiguration finden Sie unter [Gateway-eigene
  Kopplung](/de/gateway/pairing). Loopback-, ausschließlich über das iPhone erreichbare und ausschließlich über das Tailnet erreichbare
  Routen sind für die Watch nicht unabhängig erreichbar.
- Für die Mobilfunknutzung ist eine mobilfunkfähige Apple Watch mit aktivem Tarif erforderlich.
- OpenClaw ist auf der Watch aktiv. Apple erlaubt gewöhnlichen watchOS-Apps nicht,
  generische WebSocket-/TCP-Verbindungen aufrechtzuerhalten. Daher verwendet der direkte Node kurze HTTPS-
  Abfragen und stellt erneut eine Verbindung her, wenn die App in den Vordergrund zurückkehrt. Weitere Informationen finden Sie in Apples
  [Hinweisen zu Low-Level-Netzwerken unter watchOS](https://developer.apple.com/documentation/technotes/tn3135-low-level-networking-on-watchOS).

Einrichtung:

1. Öffnen Sie auf dem iPhone **Einstellungen -> Apple Watch**.
2. Tippen Sie auf **Direkte Gateway-Verbindung aktivieren**.
3. Öffnen Sie OpenClaw auf der Watch, bevor der kurzlebige Einrichtungscode abläuft.
4. Überprüfen Sie die separate Apple-Watch-Zeile mit `openclaw nodes status`.

Der Einrichtungscode enthält kurzlebige Bootstrap-Anmeldedaten ausschließlich für den Node; behandeln Sie diese
bis zu ihrem Ablauf wie ein Passwort. Er enthält niemals das gespeicherte Gateway-
Passwort oder Token des iPhones. Nach der Kopplung speichert die Watch ihr eigenes Geräte-Token und
löscht die Bootstrap-Anmeldedaten. Der direkte Modus deckt nur die nachstehenden Befehle ab.
Chat, Sprechmodus, Genehmigungen und der vorhandene `watch.*`-Benachrichtigungsablauf bleiben
iPhone-Relay-Funktionen und erfordern weiterhin das gekoppelte iPhone.

Direkte watchOS-Node-Befehle:

| Oberfläche     | Befehle                        | Hinweise                                                    |
| -------------- | ------------------------------ | ----------------------------------------------------------- |
| Gerät          | `device.info`, `device.status` | Watch-Identität, Akku, Temperatur, Speicher und Netzwerk.    |
| Mitteilungen   | `system.notify`                | Während die App aktiv ist; erfordert eine Watch-Berechtigung. |

watchOS stellt Drittanbieter-Apps kein WebKit zur Verfügung, daher
kündigt der direkte Watch-Node keine Canvas-Befehle an.

## Relay-gestützte Push-Benachrichtigungen für offizielle Builds

Offiziell verteilte iOS-Builds verwenden ein externes Push-Relay, anstatt das unverarbeitete APNs-Token für das Gateway zu veröffentlichen. Offizielle App-Store-Builds aus dem öffentlichen Release-Kanal verwenden das gehostete Relay unter `https://ios-push-relay.openclaw.ai`; diese Basis-URL ist für die App-Store-Verteilung fest einprogrammiert und berücksichtigt keine Überschreibung.

Benutzerdefinierte Relay-Bereitstellungen erfordern einen bewusst getrennten iOS-Build-/Bereitstellungspfad, dessen Relay-URL mit der Gateway-Relay-URL übereinstimmt. Der App-Store-Release-Kanal akzeptiert niemals eine benutzerdefinierte Relay-URL. Wenn Sie einen benutzerdefinierten Relay-Build verwenden, legen Sie die übereinstimmende Gateway-Relay-URL fest:

```json5
{
  gateway: {
    push: {
      apns: {
        relay: {
          baseUrl: "https://relay.example.com",
        },
      },
    },
  },
}
```

So funktioniert der Ablauf:

- Die iOS-App registriert sich mithilfe von App Attest und einer StoreKit-App-Transaktions-JWS beim Relay.
- Das Relay gibt ein nicht interpretierbares Relay-Handle sowie eine auf die Registrierung beschränkte Sendeberechtigung zurück.
- Die iOS-App ruft die Identität des gekoppelten Gateways (`gateway.identity.get`) ab und fügt sie der Relay-Registrierung hinzu, sodass die Relay-gestützte Registrierung an dieses spezifische Gateway delegiert wird.
- Die App leitet diese Relay-gestützte Registrierung mit `push.apns.register` an das gekoppelte Gateway weiter.
- Das Gateway verwendet dieses gespeicherte Relay-Handle für `push.test`, Hintergrundaktivierungen und Aktivierungsimpulse.
- Wenn die App später eine Verbindung zu einem anderen Gateway oder zu einem Build mit einer anderen Relay-Basis-URL herstellt, aktualisiert sie die Relay-Registrierung, anstatt die alte Bindung wiederzuverwenden.

Was das Gateway für diesen Pfad **nicht** benötigt: kein bereitstellungsweites Relay-Token und keinen direkten APNs-Schlüssel für Relay-gestützte Sendungen der offiziellen App-Store-Version.

Erwarteter Ablauf für Betreiber:

1. Installieren Sie die offizielle iOS-App.
2. Optional: Legen Sie `gateway.push.apns.relay.baseUrl` auf dem Gateway nur fest, wenn Sie bewusst einen separaten benutzerdefinierten Relay-Build verwenden.
3. Koppeln Sie die App mit dem Gateway und lassen Sie den Verbindungsaufbau abschließen.
4. Die App veröffentlicht `push.apns.register`, sobald sie über ein APNs-Token verfügt, die Betreibersitzung verbunden ist und die Relay-Registrierung erfolgreich war.
5. Danach können `push.test`, Aktivierungen bei erneuter Verbindung und Aktivierungsimpulse die gespeicherte Relay-gestützte Registrierung verwenden.

## Hintergrund-Lebenszeichen

Wenn iOS die App für eine stille Push-Nachricht, eine Hintergrundaktualisierung oder ein Ereignis aufgrund einer signifikanten Standortänderung aktiviert, versucht die App eine kurze erneute Node-Verbindung und ruft anschließend `node.event` mit `event: "node.presence.alive"` auf. Das Gateway zeichnet dies erst dann als `lastSeenAtMs`/`lastSeenReason` in den Metadaten der gekoppelten Node bzw. des gekoppelten Geräts auf, wenn die Identität des authentifizierten Node-Geräts bekannt ist.

Die App betrachtet eine Hintergrundaktivierung nur dann als erfolgreich aufgezeichnet, wenn die Gateway-Antwort `handled: true` enthält. Ältere Gateways bestätigen `node.event` möglicherweise mit `{ "ok": true }`; diese Antwort ist kompatibel, gilt jedoch nicht als dauerhafte Aktualisierung des letzten Aktivitätszeitpunkts.

Kompatibilitätshinweis:

- `OPENCLAW_APNS_RELAY_BASE_URL` funktioniert weiterhin als temporäre Umgebungsüberschreibung für das Gateway (`gateway.push.apns.relay.baseUrl` ist der bevorzugte Konfigurationspfad).
- Der Push-Modus des App-Store-Release-Builds enthält den Host des gehosteten Relays fest im Code und liest niemals eine Überschreibung der Relay-URL – die Build-Zeit-Umgebungsvariable `OPENCLAW_PUSH_RELAY_BASE_URL` wirkt sich nur auf lokale bzw. Sandbox-iOS-Build-Modi aus.

## Authentifizierungs- und Vertrauensablauf

Das Relay dient dazu, zwei Einschränkungen durchzusetzen, die direkte APNs-Nutzung auf dem Gateway für offizielle iOS-Builds nicht gewährleisten kann:

- Nur echte, über Apple vertriebene OpenClaw-iOS-Builds können das gehostete Relay verwenden.
- Ein Gateway kann Relay-gestützte Push-Nachrichten nur an iOS-Geräte senden, die mit genau diesem Gateway gekoppelt wurden.

Schritt für Schritt:

1. `iOS app -> gateway`: Die App wird über den normalen Gateway-Authentifizierungsablauf mit dem Gateway gekoppelt und erhält dadurch eine authentifizierte Node-Sitzung sowie eine authentifizierte Betreibersitzung. Die Betreibersitzung ruft `gateway.identity.get` auf.
2. `iOS app -> relay`: Die App ruft die Relay-Registrierungsendpunkte über HTTPS mit einem App-Attest-Nachweis sowie einer StoreKit-App-Transaktions-JWS auf. Das Relay validiert die Bundle-ID, den App-Attest-Nachweis und den Apple-Verteilungsnachweis und verlangt den offiziellen Produktionsverteilungsweg. Dadurch wird verhindert, dass lokale Xcode-/Entwicklungs-Builds das gehostete Relay verwenden, da ein lokaler Build den offiziellen Apple-Verteilungsnachweis nicht erbringen kann.
3. `gateway identity delegation`: Vor der Relay-Registrierung ruft die App die Identität des gekoppelten Gateways von `gateway.identity.get` ab und fügt sie der Relay-Registrierungsnutzlast hinzu. Das Relay gibt ein Relay-Handle und eine auf die Registrierung beschränkte Sendeberechtigung zurück, die an diese Gateway-Identität delegiert ist.
4. `gateway -> relay`: Das Gateway speichert das Relay-Handle und die Sendeberechtigung aus `push.apns.register`. Bei `push.test`, Aktivierungen bei erneuter Verbindung und Aktivierungsimpulsen signiert das Gateway die Sendeanfrage mit seiner eigenen Geräteidentität. Das Relay prüft sowohl die gespeicherte Sendeberechtigung als auch die Gateway-Signatur anhand der bei der Registrierung angegebenen delegierten Gateway-Identität. Ein anderes Gateway kann diese gespeicherte Registrierung nicht wiederverwenden, selbst wenn es irgendwie in den Besitz des Handles gelangt.
5. `relay -> APNs`: Das Relay verwaltet die produktiven APNs-Anmeldedaten und das unverarbeitete APNs-Token für den offiziellen Build. Bei Relay-gestützten offiziellen Builds speichert das Gateway niemals das unverarbeitete APNs-Token; das Relay sendet die endgültige Push-Nachricht im Namen des gekoppelten Gateways an APNs.

Warum dieses Design entwickelt wurde: um produktive APNs-Anmeldedaten von den Gateways der Benutzer fernzuhalten, die Speicherung unverarbeiteter APNs-Token offizieller Builds auf dem Gateway zu vermeiden, die Nutzung des gehosteten Relays ausschließlich offiziellen OpenClaw-iOS-Builds zu erlauben und zu verhindern, dass ein Gateway Aktivierungs-Push-Nachrichten an iOS-Geräte sendet, die einem anderen Gateway zugeordnet sind.

Lokale bzw. manuelle Builds verwenden weiterhin direkte APNs. Wenn Sie diese Builds ohne Relay testen, benötigt das Gateway weiterhin direkte APNs-Anmeldedaten:

```bash
export OPENCLAW_APNS_TEAM_ID="TEAMID"
export OPENCLAW_APNS_KEY_ID="KEYID"
export OPENCLAW_APNS_PRIVATE_KEY_P8="$(cat /path/to/AuthKey_KEYID.p8)"
```

Dies sind Laufzeit-Umgebungsvariablen des Gateway-Hosts und keine Fastlane-Einstellungen. `apps/ios/fastlane/.env` speichert nur die App-Store-Connect-Authentifizierung wie `APP_STORE_CONNECT_KEY_ID` und `APP_STORE_CONNECT_ISSUER_ID`; es konfiguriert nicht die direkte APNs-Zustellung für lokale iOS-Builds.

Empfohlene Speicherung auf dem Gateway-Host, konsistent mit anderen Provider-Anmeldedaten unter `~/.openclaw/credentials/`:

```bash
mkdir -p ~/.openclaw/credentials/apns
chmod 700 ~/.openclaw/credentials/apns
mv /path/to/AuthKey_KEYID.p8 ~/.openclaw/credentials/apns/AuthKey_KEYID.p8
chmod 600 ~/.openclaw/credentials/apns/AuthKey_KEYID.p8
export OPENCLAW_APNS_PRIVATE_KEY_PATH="$HOME/.openclaw/credentials/apns/AuthKey_KEYID.p8"
```

Committen Sie die Datei `.p8` nicht und legen Sie sie nicht im Repository-Checkout ab.

## Erkennungspfade

### Bonjour (LAN)

Die iOS-App durchsucht `_openclaw-gw._tcp` auf `local.` und, sofern konfiguriert, dieselbe Wide-Area-DNS-SD-Erkennungsdomäne. Gateways im selben LAN werden automatisch über `local.` angezeigt; für die netzwerkübergreifende Erkennung kann die konfigurierte Wide-Area-Domäne verwendet werden, ohne den Beacon-Typ zu ändern.

### Tailnet (netzwerkübergreifend)

Wenn mDNS blockiert ist, verwenden Sie eine Unicast-DNS-SD-Zone (wählen Sie eine Domäne; Beispiel: `openclaw.internal.`) und Split DNS von Tailscale. Ein CoreDNS-Beispiel finden Sie unter [Bonjour](/de/gateway/bonjour).

### Manueller Host/Port

Aktivieren Sie in Settings die Option **Manual Host** und geben Sie den Gateway-Host und -Port ein (Standard: `18789`).

## Mehrere Gateways

Die App führt ein Verzeichnis aller Gateways, mit denen sie gekoppelt wurde, sodass Sie zwischen ihnen wechseln können, ohne sie erneut zu koppeln:

- Unter **Settings -> Gateway** wird eine Liste **Paired Gateways** angezeigt, in der das aktive Gateway markiert ist. Tippen Sie auf einen Eintrag, um zu wechseln. Die App beendet die aktuellen Sitzungen und stellt eine neue Verbindung zum ausgewählten Gateway her. Wenn mehr als ein Gateway gekoppelt ist, wird neben der Verbindungszeile ein Schnellwechselmenü angezeigt.
- Anmeldedaten, TLS-Vertrauensentscheidungen, Gateway-spezifische Einstellungen und der zwischengespeicherte Chatverlauf werden für jedes Gateway separat gespeichert. Beim Wechsel werden Zustände verschiedener Gateways niemals vermischt, und die Push-Registrierung folgt dem aktiven Gateway.
- Wischen Sie über ein gekoppeltes Gateway oder verwenden Sie dessen Kontextmenü, um es mit **Forget** zu entfernen. Dadurch werden seine Anmeldedaten, Gerätetoken, der TLS-Pin und zwischengespeicherte Chats gelöscht.
- Erkannte Gateways müssen im Netzwerk sichtbar sein, damit zu ihnen gewechselt werden kann; manuell konfigurierte Gateways stellen die Verbindung über den gespeicherten Host und Port wieder her.

## Canvas + A2UI

Die iOS-Node rendert eine WKWebView-Canvas. Steuern Sie sie mit `node.invoke`:

```bash
openclaw nodes invoke --node "iOS Node" --command canvas.navigate --params '{"url":"http://<gateway-host>:18789/__openclaw__/canvas/"}'
```

Hinweise:

- Der Canvas-Host des Gateways stellt `/__openclaw__/canvas/` und `/__openclaw__/a2ui/` über den Gateway-HTTP-Server bereit (derselbe Port wie `gateway.port`, Standard: `18789`).
- Die iOS-Node behält das integrierte Grundgerüst als verbundene Standardansicht bei. `canvas.a2ui.push` und `canvas.a2ui.reset` verwenden die gebündelte, App-eigene A2UI-Seite.
- A2UI-Seiten eines entfernten Gateways können unter iOS nur gerendert werden; native A2UI-Schaltflächenaktionen werden ausschließlich von gebündelten, App-eigenen Seiten akzeptiert.
- Kehren Sie mit `canvas.navigate` und `{"url":""}` zum integrierten Grundgerüst zurück.

## Beziehung zu Computer Use

Die iOS-App ist eine mobile Node-Oberfläche und kein Backend für Codex Computer Use. Codex Computer Use und `cua-driver mcp` steuern über MCP-Tools einen lokalen macOS-Desktop; die iOS-App stellt iPhone-Funktionen über OpenClaw-Node-Befehle wie `canvas.*`, `camera.*`, `screen.*`, `location.*` und `talk.*` bereit.

Agenten können die iOS-App weiterhin über OpenClaw bedienen, indem sie Node-Befehle aufrufen. Diese Aufrufe durchlaufen jedoch das Gateway-Node-Protokoll und unterliegen den iOS-Beschränkungen für Vorder- und Hintergrundbetrieb. Verwenden Sie [Codex Computer Use](/de/plugins/codex-computer-use) für die lokale Desktop-Steuerung und diese Seite für die Funktionen der iOS-Node.

### Canvas-Auswertung/Snapshot

```bash
openclaw nodes invoke --node "iOS Node" --command canvas.eval --params '{"javaScript":"(() => { const {ctx} = window.__openclaw; ctx.clearRect(0,0,innerWidth,innerHeight); ctx.lineWidth=6; ctx.strokeStyle=\"#ff2d55\"; ctx.beginPath(); ctx.moveTo(40,40); ctx.lineTo(innerWidth-40, innerHeight-40); ctx.stroke(); return \"ok\"; })()"}'
```

```bash
openclaw nodes invoke --node "iOS Node" --command canvas.snapshot --params '{"maxWidth":900,"format":"jpeg"}'
```

## Sprachaktivierung + Sprechmodus

- Sprachaktivierung und Sprechmodus sind in Settings verfügbar.
- OpenAI-Echtzeit-Talk verwendet vom Client verwaltetes WebRTC, wenn `talk.realtime.transport` den Wert `webrtc` hat; eine explizite `gateway-relay`-Konfiguration bleibt in der Verantwortung des Gateways. Siehe [Sprechmodus](/de/nodes/talk).
- Talk-fähige iOS-Nodes kündigen die Fähigkeit `talk` an und können `talk.ptt.start`, `talk.ptt.stop`, `talk.ptt.cancel` und `talk.ptt.once` deklarieren; das Gateway erlaubt diese Push-to-Talk-Befehle standardmäßig für vertrauenswürdige Talk-fähige Nodes.
- iOS kann die Audiowiedergabe im Hintergrund aussetzen; betrachten Sie Sprachfunktionen als Best-Effort-Funktionen, wenn die App nicht aktiv ist.

## Häufige Fehler

- `NODE_BACKGROUND_UNAVAILABLE`: Bringen Sie die iOS-App in den Vordergrund (Canvas-, Kamera- und Bildschirmbefehle erfordern dies).
- `A2UI_HOST_UNAVAILABLE`: Die gebündelte A2UI-Seite war in der WebView der App nicht erreichbar; lassen Sie die App im Vordergrund auf dem Tab Screen und versuchen Sie es erneut.
- Die Kopplungsaufforderung erscheint nie: Führen Sie `openclaw devices list` aus und genehmigen Sie die Kopplung manuell.
- Die Watch zeigt keinen iPhone-Status an: Vergewissern Sie sich, dass das iPhone `watchPaired: true`
  und `watchAppInstalled: true` in `watch.status` meldet. Wenn die Kopplung falsch ist, koppeln Sie die
  Watch in Apples Watch-App. Wenn die Installation falsch ist, installieren Sie die Begleit-App
  über **My Watch -> Available Apps**. Öffnen Sie OpenClaw nach jeder dieser Änderungen einmal auf der
  Watch. Für die sofortige Erreichbarkeit müssen weiterhin beide Apps ausgeführt werden,
  während Aktualisierungen in der Warteschlange später im Hintergrund eintreffen können.
- Die erneute Verbindung schlägt nach einer Neuinstallation fehl: Das Kopplungstoken im Schlüsselbund wurde gelöscht; koppeln Sie die Node erneut.

## Verwandte Dokumentation

- [Kopplung](/de/channels/pairing)
- [Erkennung](/de/gateway/discovery)
- [Bonjour](/de/gateway/bonjour)
