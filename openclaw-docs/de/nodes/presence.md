---
read_when:
    - Sie möchten, dass OpenClaw den aktiven Mac identifiziert
    - Sie debuggen die Aktivität der letzten Eingabe oder die Auswahl des aktiven Node
    - Sie möchten das Routing von Node-Verbindungsbenachrichtigungen verstehen
summary: Den zuletzt verwendeten Mac erkennen und Node-Warnungen dorthin weiterleiten
title: Aktive Computerpräsenz
x-i18n:
    generated_at: "2026-07-26T17:54:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c3f1d1d0e98b1f3b7478cf80696dc693677b57897b07260cce30938e9187c314
    source_path: nodes/presence.md
    workflow: 16
---

Die aktive Computerpräsenz teilt dem Gateway mit, welcher verbundene macOS-Node
die letzte physische Maus- oder Tastatureingabe empfangen hat. OpenClaw verwendet dieses Signal, um
einen Mac als `active` zu markieren, dem Agenten einen stabilen Hinweis auf den aktiven Node zu geben und
Verbindungswarnungen für Nodes an den Computer weiterzuleiten, an dem Sie sich höchstwahrscheinlich befinden.

Dies unterscheidet sich von der [Systempräsenz](/de/concepts/presence), also der aktuellen
Liste der Gateway-Clients, sowie von dauerhaften `node.presence.alive`-Signalen, die
aufzeichnen, wann ein mobiler Node zuletzt aktiviert wurde, ohne ihn als verbunden zu behandeln.

## Anforderungen

- Die OpenClaw-macOS-App ist gekoppelt und im Node-Modus verbunden.
- **Settings -> Permissions -> Active computer detection** ist aktiviert. Standardmäßig ist diese Option deaktiviert.
- Der signierten OpenClaw-App wurde die Berechtigung **Accessibility** erteilt.
- Für Verbindungswarnungen wurde außerdem die Berechtigung **Notifications** erteilt und der
  Mac-Node stellt `system.notify` bereit.

Die Aktivitätsmeldung ist derzeit im nativen macOS-Node implementiert. iOS-,
Android-, watchOS- und monitorlose Node-Hosts können den Verbindungsstatus oder
den letzten Hintergrundkontakt melden, konkurrieren jedoch nicht um die Kennzeichnung als aktiver Computer.

## Aktiven Computer prüfen

1. Öffnen Sie in der macOS-App **Settings -> Permissions**, aktivieren Sie
   **Active computer detection** und erteilen Sie unter den macOS-Systemeinstellungen die Berechtigung **Accessibility**.
2. Vergewissern Sie sich, dass der Mac-Node verbunden ist:

   ```bash
   openclaw nodes status --connected
   ```

3. Bewegen Sie auf diesem Mac die Maus oder drücken Sie eine Taste und führen Sie anschließend Folgendes aus:

   ```bash
   openclaw nodes status
   openclaw nodes describe --node <node-id-or-name>
   ```

Der aktuellste geeignete Mac wird mit `active` markiert. Die Statusausgabe zeigt das Alter seiner letzten Eingabe;
`describe` stellt `active`, `lastActiveAtMs` und `presenceUpdatedAtMs` bereit.
Aktivitätsmeldungen werden bewusst zusammengefasst. Daher kann es bis zu etwa 15
Sekunden dauern, bis nach einer kürzlich erfolgten Meldung eine weitere Eingabe angezeigt wird.

## Wie Aktivität zur Präsenz wird

Der macOS-Reporter liest alle zwei Sekunden die Systemleerlaufzeit des HID aus. Er
meldet einmal, wenn eine Node-Verbindung bereit ist, und danach neuere physische
Aktivität höchstens einmal alle 15 Sekunden. Im Leerlauf sendet er alle drei
Minuten ein Keepalive. Die Leerlaufdauer ist auf 30 Tage begrenzt, damit ein sehr alter Messwert
nicht nach vorn wandern und fälschlicherweise zum neuesten Computer werden kann.

Durch das Deaktivieren von **Active computer detection** wird die Abfrage beendet und über die aktuelle
Node-Verbindung ein authentifiziertes Löschereignis gesendet. Das Gateway entfernt sofort
die gespeicherten Aktivitätszeitstempel dieses Macs und ermittelt den aktiven Computer neu;
andere Node-Funktionen und laufende Aufgaben bleiben verbunden. Wenn das verbundene
Gateway älter als diese Löschaktion ist, verbindet sich der Mac-Node einmal neu, damit
die Bereinigung beim Trennen der Verbindung stattdessen die gespeicherte Aktivität entfernen kann.

Das Gateway akzeptiert Aktivität nur, wenn alle folgenden Bedingungen erfüllt sind:

- Das Ereignis gehört zur aktuellen authentifizierten Verbindung dieser Node-ID;
- der Node verfügt über die wirksame Berechtigung `accessibility: true`;
- die Nutzlast enthält einen begrenzten ganzzahligen Wert für `idleSeconds`.

Das Gateway zieht `idleSeconds` von seiner eigenen Beobachtungszeit ab, um
`lastActiveAtMs` abzuleiten. Es vertraut niemals einem vom Node bereitgestellten Zeitstempel der Systemuhr. Unter
den verbundenen geeigneten Macs gewinnt der neueste Wert für `lastActiveAtMs`; bei Gleichstand wird die
aktuellste Präsenzaktualisierung verwendet.

Die Präsenz ist prozesslokal und an die Verbindung gebunden. Wenn die aktuelle
Sitzung getrennt, durch eine andere Sitzung mit derselben Node-ID ersetzt oder
die Berechtigung „Accessibility“ widerrufen wird, wird der Aktivitätsstatus dieses Nodes gelöscht und der aktive Mac neu ermittelt.

## Datenschutz und Modellkontext

Die Aktivitätsfreigabe ist standardmäßig deaktiviert und von der für die
UI-Automatisierung verwendeten Berechtigung „Accessibility“ unabhängig. OpenClaw sendet die Leerlaufdauer, nicht den Inhalt von Eingaben. Es sendet keine Tastenwerte,
Mauskoordinaten, Anwendungsnamen, Fenstertitel oder Rohdaten von Eingabeereignissen. Der
macOS-Reporter liest den Hardware-HID-Status. Daher führen synthetische
Computersteuerungsereignisse nicht dazu, dass ein automatisierter Mac als der von Ihnen physisch
verwendete Computer erscheint.

Kontinuierliche Aktivität erzeugt keine Systemereignisse für das Modell. Die dynamische
Laufzeitzeile enthält nur die authentifizierte Node-ID:

```text
active_node=<node-id>
```

Genaue Zeitstempel und vom Node gesteuerte Anzeigenamen werden aus dem Prompt
herausgehalten, um Prompt-Injection und Cache-Änderungen zu vermeiden. Wenn der Agent aktuelle Details benötigt,
kann das Tool `nodes` stattdessen `node.list` oder `node.describe` lesen.

## Weiterleitung von Verbindungswarnungen

Nachdem ein Node nach seiner Genehmigung den ersten erfolgreichen Gateway-Handshake abgeschlossen hat,
wartet OpenClaw 750 Millisekunden, damit der verbindende Mac seine erste
Aktivitätsmeldung übermitteln kann. Anschließend versucht OpenClaw, die Warnung an den verbundenen, benachrichtigungsfähigen Mac mit der
aktuellsten Aktivität zu senden.

- Wenn die primäre Zustellung erfolgreich ist, erhält kein anderer Mac die Warnung.
- Wenn kein aktiver Mac verfügbar ist oder die primäre Zustellung fehlschlägt, wartet OpenClaw fünf
  Sekunden und versucht es bei allen übrigen verbundenen Macs, die `system.notify` bereitstellen.
- Spätere Neuverbindungen erfolgen ohne Warnung. Das Gateway zeichnet die erfolgreiche Verbindung
  in den Kopplungsmetadaten auf, sodass ein Gateway-Neustart nicht erneut Warnungen für alle
  zuvor verbundenen Nodes auslöst.

Warnungen sind an die authentifizierte Node-Identität gebunden. Eine Ersatzsitzung für
denselben Node übernimmt dessen ausstehende Warnung zur ersten Verbindung. Wenn dieser Node zum
Zeitpunkt der Zustellung nicht mehr verbunden ist, wird die Warnung verworfen.

## Fehlerbehebung

| Symptom                                   | Prüfung                                                                                                                                                                |
| ----------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Keine Zeile ist mit `active` markiert                 | Vergewissern Sie sich, dass die Erkennung des aktiven Computers aktiviert und ein nativer macOS-Node verbunden ist sowie `openclaw nodes describe --node <id>` den Wert `permissions.accessibility: true` anzeigt.   |
| Der falsche Mac bleibt aktiv              | Verwenden Sie diesen Mac physisch, warten Sie das Zusammenfassungsintervall ab und führen Sie `openclaw nodes status` erneut aus. Synthetische Computersteuerungsaktionen zählen nicht.                        |
| Daten zur letzten Eingabe verschwinden                | Prüfen Sie, ob die Verbindung zum Mac getrennt, seine Node-Sitzung ersetzt oder die Berechtigung „Accessibility“ widerrufen wurde. Jede dieser Bedingungen löscht die Aktivität absichtlich.                       |
| Die Warnung wird auf mehreren Macs angezeigt         | Die primäre Zustellung war nicht verfügbar oder ist fehlgeschlagen, weshalb die verzögerte Ausweichzustellung ausgeführt wurde. Vergewissern Sie sich, dass der aktive Mac verbunden ist, Benachrichtigungen zulässt und `system.notify` bereitstellt. |
| Der Agent erwähnt den aktiven Mac nicht | Beginnen Sie nach Aktivitätsänderungen eine neue Interaktion. Der Laufzeithinweis ist stabil und kompakt; verwenden Sie das Tool `nodes` für genaue aktuelle Metadaten.                                    |

Informationen zur TCC-Wiederherstellung finden Sie unter [macOS-Berechtigungen](/de/platforms/mac/permissions). Informationen zu
Node-Verbindungs- und Befehlsfehlern finden Sie unter [Node-Fehlerbehebung](/de/nodes/troubleshooting).

## Verwandte Themen

- [Nodes](/de/nodes)
- [Nodes-CLI](/de/cli/nodes)
- [Systempräsenz](/de/concepts/presence)
- [Gateway-Protokoll](/de/gateway/protocol#presence)
- [macOS-App](/de/platforms/macos)
