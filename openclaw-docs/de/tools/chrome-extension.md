---
read_when:
    - Sie möchten, dass ein Agent Ihr tatsächlich angemeldetes Chrome von Ihrem Smartphone aus steuert
    - Sie stoßen immer wieder auf die Chrome-Abfrage „Allow remote debugging?“, während niemand am Schreibtisch sitzt
    - Sie möchten das Sicherheitsmodell der Browserübernahme über die Erweiterung verstehen
summary: 'Chrome-Erweiterung: Lassen Sie OpenClaw Ihr angemeldetes Chrome ohne Aufforderung zum Remote-Debugging steuern'
title: Chrome-Erweiterung
x-i18n:
    generated_at: "2026-07-26T18:48:23Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3d974f62bb5697a23dd6a6852137ce6af5a8a4a2a8ff738eec0098f259e8faa0
    source_path: tools/chrome-extension.md
    workflow: 16
---

# Chrome-Erweiterung

Mit der OpenClaw-Chrome-Erweiterung kann ein Agent Ihre **angemeldeten Chrome-
Tabs** steuern, ohne einen separaten verwalteten Browser zu starten und **ohne**
Chromes blockierende Abfrage „Allow remote debugging?“.

Dies ist wichtig, wenn Sie OpenClaw von einem Mobiltelefon aus steuern (Telegram, WhatsApp usw.):
Das [Profil `user`](/de/tools/browser#profiles-openclaw-user-chrome) stellt über
Chromes Port für Remote-Debugging eine Verbindung her, wodurch ein Zustimmungsdialog auf dem Desktop
angezeigt wird, den niemand anklicken kann, wenn Sie nicht vor Ort sind. Die Erweiterung verwendet stattdessen die
API `chrome.debugger`, sodass der einzige Hinweis auf der Seite Chromes ausblendbares Banner
„OpenClaw started debugging this browser“ ist.

Dies entspricht dem Ansatz, den Anthropic für Claude in Chrome und OpenAI für die Codex-
Chrome-Erweiterungen verwenden.

## Funktionsweise

Drei Komponenten:

- **Browsersteuerungsdienst** (Gateway- oder Node-Host): die API, die das Werkzeug `browser`
  aufruft.
- **Erweiterungs-Relay** (Loopback-WebSocket): ein kleiner Server, den der Steuerungsdienst
  auf `127.0.0.1` startet. Er stellt OpenClaw einen Endpunkt des Chrome DevTools Protocol bereit
  und kommuniziert mit der Erweiterung. Beide Seiten authentifizieren sich mit einem
  hostlokalen Token (siehe unten).
- **OpenClaw-Chrome-Erweiterung** (MV3): verbindet sich über `chrome.debugger` mit Tabs,
  leitet CDP-Datenverkehr weiter und verwaltet die **OpenClaw-Tabgruppe**.

OpenClaw sieht und steuert nur Tabs, die sich in der **OpenClaw-Tabgruppe** befinden. Die
Gruppe bildet die Zustimmungsgrenze: Ziehen Sie einen Tab hinein, um ihn freizugeben, und ziehen Sie ihn heraus
(oder klicken Sie auf die Symbolleistenschaltfläche), um den Zugriff sofort zu widerrufen.

## Installieren und koppeln

1. Pfad der entpackten Erweiterung ausgeben:

   ```bash
   openclaw browser extension path
   ```

2. Öffnen Sie `chrome://extensions`, aktivieren Sie **Developer mode**, klicken Sie auf **Load
   unpacked** und wählen Sie das ausgegebene Verzeichnis aus.

3. Kopplungszeichenfolge ausgeben:

   ```bash
   openclaw browser extension pair
   ```

4. Klicken Sie auf das OpenClaw-Symbol in der Symbolleiste und fügen Sie die Kopplungszeichenfolge in das Pop-up ein.
   Das Badge wechselt zu **ON**, wenn sich die Erweiterung mit dem Relay verbindet.

Das Kopplungstoken ist ein **hostlokales Geheimnis**, das bei der ersten Verwendung erstellt und
unter `credentials/` im Zustandsverzeichnis gespeichert wird (Modus `0600`). Jeder Rechner, auf dem
ein Browser ausgeführt wird – der Gateway-Host und jeder Browser-Node-Host – besitzt sein eigenes
Token, sodass keine Anmeldedaten zwischen Rechnern übertragen werden müssen. Um es zu rotieren, löschen Sie die
Datei `browser-extension-relay.secret` und führen Sie die Kopplung erneut durch.

## Verwendung

Wählen Sie in einem Aufruf des Werkzeugs `browser` das integrierte Profil `chrome` aus oder legen Sie es als
Standard fest:

```bash
openclaw config set browser.defaultProfile chrome
```

```json5
{
  browser: {
    profiles: {
      chrome: { driver: "extension", color: "#FF4500" },
    },
  },
}
```

- Tab freigeben: Klicken Sie in diesem Tab auf die OpenClaw-Symbolleistenschaltfläche (dadurch tritt er der
  OpenClaw-Tabgruppe bei) oder ziehen Sie einen beliebigen Tab in die Gruppe.
- Der Agent kann auch neue Tabs öffnen; diese werden automatisch der Gruppe hinzugefügt.
- Widerrufen: Klicken Sie erneut auf die Schaltfläche, ziehen Sie den Tab aus der Gruppe oder schließen Sie
  Chromes Debugging-Banner. Der Agent verliert sofort den Zugriff auf diesen Tab.

### Tab-Copilot-Seitenleiste

Klicken Sie nach dem Koppeln der Erweiterung im Symbolleisten-Pop-up auf **Open tab copilot**.
OpenClaw konfiguriert `sidepanel.html` für genau diesen Chrome-Tab; das Manifest enthält
keinen globalen Seitenleistenpfad. Jeder Tab erhält daher ein separates Seitenleistendokument,
eine separate Gateway-Sitzung, ein separates Nachrichtenabonnement und eine typisierte Browserwerkzeug-Bindung.

Die Seitenleiste fügt Ihrer Nachricht weder die Seiten-URL noch den Titel, das DOM oder sichtbaren Text
hinzu. Sie sendet nur den von Ihnen eingegebenen Text. Browseraktionen übertragen eine separate,
vom Gateway authentifizierte Bindung, die den Chrome-Tab und das CDP-Ziel enthält, und das
Browserwerkzeug weist Versuche zurück, dieses Ziel zu ersetzen oder browserweite
Aktionen zu verwenden. Antworten verbleiben in der Seitenleiste (`deliver: false`); sie übernehmen keine
Telegram-, Discord- oder andere Kanalroute.

Der Copilot ist ein dediziertes gekoppeltes Gateway-Gerät mit den Geltungsbereichen `operator.read` und
`operator.write`. Prüfen und genehmigen Sie bei der ersten Verwendung seine Anfrage:

```bash
openclaw devices list
openclaw devices approve <requestId>
```

Die Erweiterung behält diese Geräteidentität und das vom Gateway ausgestellte Gerätetoken bei,
beschränkt auf den kanonischen Gateway-Endpunkt, der sie ausgestellt hat. Durch das Koppeln mit einem anderen
Gateway entstehen eine separate Identität sowie eine separate Token- und Sitzungsverwahrung; Anmeldedaten und
Sitzungen werden niemals endpunktübergreifend wiederverwendet. Die Erweiterung speichert das
gemeinsame Gateway-Geheimnis nicht dauerhaft. Eine Seitenleiste kann nur ihre eigenen Tab-Sitzungen abonnieren, und
das Gateway filtert diese Ereignisse vor der Zustellung.

Wenn die Gateway-Verbindung während eines Laufs abbricht, verwahrt die Erweiterung dauerhaft
die ID dieses Laufs. Nach dem erneuten Verbinden bricht sie den nicht abgeschlossenen Lauf ab, bevor
eine Seitenleiste wieder aktiviert wird, und lädt anschließend den Transkriptverlauf neu. Dieser Fail-Closed-Schritt
verhindert, dass Browseraktionen während einer Zustellungslücke unbemerkt fortgesetzt werden.

Beim Schließen eines Tabs werden dessen Live-Abonnement sofort entfernt, jeder sichtbare
Lauf abgebrochen und die Sitzung dieses Tabs als archiviert markiert. Wenn das Gateway vorübergehend
offline ist, speichert die Erweiterung die ausstehende Archivierung und versucht sie erst erneut, wenn sich
derselbe Gateway-Endpunkt wieder verbindet; sie sendet niemals eine Archivierungsanfrage an ein
anderes Gateway. Nach einem Browserabsturz archiviert der nächste Start Sitzungen,
die von der vorherigen Browserinstanz zurückgelassen wurden. Archivierte Sitzungen weisen neue Aufgaben zurück, während
ihre Transkripte im Sitzungsverlauf verfügbar bleiben. Browser-Copilot-Schlüssel sind
Thread-Sitzungen, sodass sie durch die normale Wartung nach Alter und Eintragsanzahl erhalten bleiben. Das
Datenträgerbudget pro Agentensitzung gilt weiterhin (Standard: `2gb`) und kann unter hoher Auslastung die
ältesten Sitzungen entfernen; siehe [Sitzungswartung](/de/reference/session-management-compaction#store-maintenance-and-disk-controls).

Die Seitenleiste erfordert derzeit entweder ein vom Gateway gehostetes Erweiterungs-Relay oder ein
direktes Remote-Gateway-Relay. Ein Loopback-Relay auf einem Browser-Node kann die für
die typisierte Tab-Bindung erforderliche Node-Route noch nicht bereitstellen, daher lehnt die Seitenleiste
diese Topologie ab, anstatt auf browserweites Routing zurückzufallen.

## Eine Seite an OpenClaw senden

Verwenden Sie **Send page to OpenClaw** im Symbolleisten-Pop-up, um lesbaren Seitentext
mit Ihrer OpenClaw-Hauptsitzung zu teilen. Sie können eine optionale Notiz hinzufügen, das Kontextmenü
der Seite oder Auswahl verwenden oder `Alt+Shift+S` drücken. OpenClaw bevorzugt Ihre aktuelle
Auswahl, sofern eine vorhanden ist, reiht die Freigabe als Systemereignis ein und aktiviert die
Hauptsitzung sofort.

Der Tab muss sich nicht in der OpenClaw-Tabgruppe befinden. Dies ist eine einmalige,
explizite Freigabe: Nichts anderes auf der Seite wird offengelegt, und sie gewährt keinen fortlaufenden
Zugriff. Google Docs werden mit Ihrer angemeldeten Browser-
sitzung als Klartext exportiert, ohne dass eine Einrichtung der Google API erforderlich ist. X- und Twitter-Threads werden ohne
die umgebende Benutzeroberfläche extrahiert.

Seitentext wird in OpenClaws Sicherheitsgrenze für externe Inhalte eingeschlossen. Ihre
optionale Notiz bleibt als Ihre eigene Anweisung außerhalb dieser Grenze. Seitentext
und Auswahlen sind auf etwa 120.000 Zeichen begrenzt und enthalten bei Kürzung eine
Markierung.

Die Seitenfreigabe funktioniert, wenn das Erweiterungs-Relay vom Gateway gehostet wird und
eine Kopplung auf demselben Host oder eine direkte Gateway-Kopplung über `wss://` verwendet wird. Auf Nodes gehostete Relays geben
derzeit einen eindeutigen Fehler zurück. Um das Tastenkürzel neu zuzuweisen, öffnen Sie
`chrome://extensions/shortcuts`.

## Remote / rechnerübergreifend

Chrome muss nicht auf dem Gateway-Host ausgeführt werden. Drei Topologien funktionieren:

- **Derselbe Host** (Gateway + Chrome auf einem Rechner): Koppeln Sie auf diesem Rechner mit
  `openclaw browser extension pair`. Das Relay ist ausschließlich über Loopback erreichbar.
  Wenn das lokale Gateway TLS verwendet, geben Sie den Hostnamen seines Zertifikats explizit mit
  `--gateway-url wss://gateway-host.example` an; bei der Kopplung wird niemals eine Loopback-IP eingesetzt.
- **Direkt mit einem Remote-Gateway** (Chrome auf Ihrem Laptop, Gateway auf einem VPS und
  **nichts anderes auf dem Laptop**): Führen Sie auf dem Gateway
  `openclaw browser extension pair --gateway-url wss://your-gateway.example.com` aus.
  Dadurch wird eine Zeichenfolge `wss://…/browser/extension#<secret>` ausgegeben; laden und koppeln Sie die
  Erweiterung auf dem Laptop. Die Erweiterung verbindet sich über `wss://` **direkt mit dem Gateway**
  – auf dem Laptop sind weder eine OpenClaw-Installation, ein Node, eine CLI noch ein offener eingehender Port
  erforderlich. Dies ist der Pfad für verwaltetes Hosting.
- **Über einen Browser-Node-Host** (Chrome auf einem Rechner, auf dem bereits ein OpenClaw-
  Node ausgeführt wird): Führen Sie `pair` auf dem Node aus und koppeln Sie lokal; das Gateway leitet Browser-
  aktionen über die bestehende authentifizierte Node-Verbindung an den Node weiter.

Das Kopplungsgeheimnis gilt pro Host (im direkten Fall das des Gateways) und wird von
der Route `/browser/extension` des Gateways validiert. Stellen Sie das Gateway für den direkten Pfad
über TLS (`wss://`) bereit, damit das Kopplungsgeheimnis und der CDP-Datenverkehr verschlüsselt werden.
Das Geheimnis verbleibt im URL-Fragment der Kopplungszeichenfolge und wird während
des WebSocket-Handshakes als Subprotokoll-Anmeldedatum übermittelt, sodass normale Proxy-Zugriffs-
protokolle es nicht in der Anfrage-URL erhalten. Stellen Sie sicher, dass jeder Reverse-Proxy
den Standardheader `Sec-WebSocket-Protocol` beibehält.

## Diagnose

```bash
openclaw browser status --browser-profile chrome
openclaw browser doctor --browser-profile chrome
```

`doctor` meldet die Prüfung des **Chrome-Erweiterungs-Relays** als fehlgeschlagen, bis das
Erweiterungs-Pop-up **Connected** anzeigt.

## Sicherheitsmodell

- Das Relay bindet ausschließlich an Loopback; beide WebSocket-Seiten werden mit dem
  abgeleiteten Token authentifiziert, und die Herkunft der Erweiterungsseite wird gegen `chrome-extension://` geprüft.
- Bei direkter Gateway-Kopplung wird das Relay-Token nicht in der Anfrage-URL akzeptiert;
  die mitgelieferte Erweiterung überträgt es stattdessen in der WebSocket-Subprotokollliste.
- Der Agent kann nur Tabs in der **OpenClaw-Tabgruppe** sehen und steuern. Ihre
  anderen Tabs bleiben privat.
- Läufe in der Seitenleiste sind doppelt beschränkt: Die Gateway-Zustellung verwendet eine sitzungsspezifische
  Zulassungsliste, und Browserwerkzeuge erzwingen die außerhalb des Prompts übertragene Bindung
  an den Chrome-Tab und das Ziel.
- Im Vergleich zum Profil `user` (Chrome MCP), das nach Genehmigung der Remote-Debugging-Abfrage Ihren gesamten
  angemeldeten Browser offenlegt, beschränkt die Erweiterung
  die freigegebene Oberfläche auf eine Tabgruppe, die Sie auf einen Blick kontrollieren können.

Siehe auch: [Browser](/de/tools/browser) für das vollständige Profilmodell und die
verwalteten Profile `openclaw` und Chrome MCP `user`.
