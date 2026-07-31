---
read_when:
    - IPC-Verträge oder die IPC der Menüleisten-App bearbeiten
summary: macOS-IPC-Architektur für die OpenClaw-App, den Gateway-Node-Transport und PeekabooBridge
title: macOS-IPC
x-i18n:
    generated_at: "2026-07-26T18:34:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 39e11af2bb9348d1c1f6e4fe6be95e825d23d5c1aa66e32dae713a89afb12b4f
    source_path: platforms/mac/xpc.md
    workflow: 16
---

# OpenClaw-macOS-IPC-Architektur

Ein lokaler Unix-Socket verbindet den Node-Hostdienst mit der macOS-App für Ausführungsgenehmigungen und `system.run`. Eine `openclaw-mac`-Debug-CLI (`apps/macos/Sources/OpenClawMacCLI`) ist für Erkennungs- und Verbindungsprüfungen vorhanden; Agentenaktionen laufen weiterhin über den Gateway-WebSocket und `node.invoke`. Der Node-gestützte `computer.act`-Pfad führt eingebettete Peekaboo-Automatisierung prozessintern aus; eigenständige Peekaboo-Clients verwenden PeekabooBridge.

## Ziele

- Eine einzelne GUI-App-Instanz, die sämtliche TCC-bezogenen Aufgaben übernimmt (Benachrichtigungen, Bildschirmaufzeichnung, Mikrofon, Spracherkennung, AppleScript).
- Eine kleine Automatisierungsoberfläche: Gateway + Node-Befehle, prozessinternes `computer.act` sowie PeekabooBridge für eigenständige Clients zur UI-Automatisierung.
- Vorhersehbare Berechtigungen: stets dieselbe signierte Bundle-ID, von launchd gestartet, damit TCC-Zugriffsrechte erhalten bleiben.

## Funktionsweise

### Gateway- und Node-Transport

- Die App führt das Gateway (im lokalen Modus) aus und verbindet sich damit als Node.
- Agentenaktionen werden über `node.invoke` ausgeführt (z. B. `system.run`, `system.notify`, `canvas.*`).
- Zu den Node-Befehlen gehören `canvas.*`, `camera.snap`, `camera.clip`, `screen.snapshot`, `screen.record`, `computer.act`, `system.run` und `system.notify`.
- Der Node meldet eine `permissions`-Zuordnung, damit Agenten erkennen können, ob Zugriff auf Bildschirm, Kamera, Mikrofon, Spracherkennung, Automatisierung oder Bedienungshilfen verfügbar ist.

### Node-Dienst und App-IPC

- Ein monitorloser Node-Hostdienst stellt eine Verbindung zum Gateway-WebSocket her.
- `system.run`-Anfragen werden über einen lokalen Unix-Socket (`ExecApprovalsSocket.swift`) an die macOS-App weitergeleitet.
- Die App führt die Ausführung im UI-Kontext durch, fordert bei Bedarf eine Bestätigung an und gibt die Ausgabe zurück.

Diagramm (SCI):

```text
Agent -> Gateway -> Node-Dienst (WS)
                      |  IPC (UDS + Token + HMAC + TTL)
                      v
                  Mac-App (UI + TCC + system.run)
```

### PeekabooBridge (UI-Automatisierung)

- Das integrierte Agentenwerkzeug `computer` verwendet diesen Socket **nicht**. Ein gekoppelter macOS-Node führt `computer.act` im App-Prozess mit eingebetteten Peekaboo-Diensten aus.
- Die UI-Automatisierung verwendet einen separaten UNIX-Socket (`~/Library/Application Support/OpenClaw/<socket>`) und das PeekabooBridge-JSON-Protokoll.
- Host-Prioritätsreihenfolge (clientseitig): Peekaboo.app -> Claude.app -> OpenClaw.app -> lokale Ausführung.
- Sicherheit: Bridge-Hosts erfordern eine TeamID aus der Zulassungsliste (das gebündelte `PeekabooBridgeHostCoordinator` lässt ein festgelegtes Team sowie das eigene Signierungsteam der App zu); ein ausschließlich für DEBUG vorgesehener Ausweg für dieselbe UID wird durch `PEEKABOO_ALLOW_UNSIGNED_SOCKET_CLIENTS=1` geschützt (Peekaboo-Konvention).
- Weitere Informationen finden Sie unter [PeekabooBridge-Verwendung](/de/platforms/mac/peekaboo).

## Betriebsabläufe

- Neustart/Neuaufbau: `scripts/restart-mac.sh` beendet vorhandene Instanzen, erstellt die App über Swift neu, paketiert sie erneut und startet sie wieder. Dabei wird automatisch eine verfügbare Signierungsidentität erkannt; wenn keine gefunden wird, erfolgt ein Rückgriff auf `--no-sign`. Übergeben Sie `--sign`, um eine Signierung zu verlangen (schlägt fehl, wenn kein Schlüssel verfügbar ist), oder `--no-sign`, um den unsignierten Pfad zu erzwingen. Das in der Umgebung gesetzte `SIGN_IDENTITY` wird im signierten Pfad entfernt, sodass die Identitäts-Autoerkennung von `scripts/codesign-mac-app.sh` das Zertifikat auswählt.
- Einzelinstanz: Die App prüft `NSWorkspace.runningApplications` auf eine doppelte Bundle-ID und wird beendet, wenn mehr als eine Instanz gefunden wird (`isDuplicateInstance()` in `MenuBar.swift`).

## Hinweise zur Absicherung

- Fordern Sie für alle privilegierten Oberflächen vorzugsweise eine übereinstimmende TeamID.
- PeekabooBridge: `PEEKABOO_ALLOW_UNSIGNED_SOCKET_CLIENTS=1` (nur DEBUG) kann für die lokale Entwicklung Aufrufer mit derselben UID zulassen.
- Die gesamte Kommunikation bleibt ausschließlich lokal; es werden keine Netzwerk-Sockets bereitgestellt.
- TCC-Abfragen stammen ausschließlich aus dem GUI-App-Bundle; halten Sie die signierte Bundle-ID über Neuaufbauten hinweg stabil.
- Absicherung des Sockets für Ausführungsgenehmigungen: Dateimodus `0600`, gemeinsames Token, Prüfung der Peer-UID (`getpeereid`), HMAC-SHA256-Challenge-Response-Verfahren und eine kurze TTL für Anfragen.

## Verwandte Themen

- [macOS-App](/de/platforms/macos)
- [macOS-IPC-Ablauf (Ausführungsgenehmigungen)](/de/tools/exec-approvals-advanced#macos-ipc-flow)
