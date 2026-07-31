---
read_when:
    - Implementieren oder Aktualisieren von Gateway-WS-Clients
    - Debugging von Protokollabweichungen oder Verbindungsfehlern
    - Protokollschema/-modelle neu generieren
summary: 'Gateway-WebSocket-Protokoll: Handshake, Frames, Versionierung'
title: Gateway-Protokoll
x-i18n:
    generated_at: "2026-07-26T18:28:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 89d637a9070bc6512a182fea0fd890b56287e0080515ba4fba9b2591c6247e0d
    source_path: gateway/protocol.md
    workflow: 16
---

Das Gateway-WS-Protokoll ist die zentrale Steuerungsebene und der Node-Transport für
OpenClaw. Operator- und Node-Clients (CLI, Web-UI, macOS-App, iOS-/Android-Nodes,
headless Nodes) stellen eine Verbindung über WebSocket her und deklarieren beim
Handshake eine **Rolle** und einen **Scope**.

## npm-Pakete

Diese Pakete werden mit den OpenClaw-Release-Zyklen ausgeliefert. Während der anfänglichen Einführung
kann npm `E404` zurückgeben, bis das erste Release mit diesen Paketen veröffentlicht wurde.

- [`@openclaw/gateway-protocol`](https://www.npmjs.com/package/@openclaw/gateway-protocol)
  veröffentlicht die Schemas, Validatoren, TypeScript-Typen, schlanken Frame- und Fehler-
  Hilfsfunktionen sowie Versionskonstanten. Das Tarball enthält den generierten
  [`protocol.schema.json`](https://unpkg.com/@openclaw/gateway-protocol/protocol.schema.json)
  maschinenlesbaren Vertrag.
- [`@openclaw/gateway-client`](https://www.npmjs.com/package/@openclaw/gateway-client)
  veröffentlicht den Referenz-Node-Client und einen browsersicheren Einstiegspunkt unter
  `@openclaw/gateway-client/browser`.

Hinweise zum Anwendungslebenszyklus finden Sie unter
[Gateway-Client erstellen](https://docs.openclaw.ai/gateway/clients). Informationen zu Apps,
die das Gateway als untergeordneten Prozess überwachen, finden Sie unter
[OpenClaw einbetten](https://docs.openclaw.ai/gateway/embedding).

## Transport und Framing

- WebSocket, Text-Frames, JSON-Nutzdaten.
- Der erste Frame **muss** eine `connect`-Anfrage sein.
- Frames vor dem Verbindungsaufbau sind auf 64 KiB (`MAX_PREAUTH_PAYLOAD_BYTES`) begrenzt. Nach
  dem Handshake gelten `hello-ok.policy.maxPayload` und
  `hello-ok.policy.maxBufferedBytes`. Bei aktivierter Diagnose lösen übergroße
  eingehende Frames und langsame ausgehende Puffer `payload.large`-Ereignisse aus, bevor
  das Gateway die Verbindung schließt oder den Frame verwirft. Diese Ereignisse enthalten `surface`, Byte-
  Größen, Grenzwerte und einen sicheren Ursachencode, jedoch niemals Nachrichteninhalte, Inhalte von
  Anhängen, rohe Frame-Bytes, Tokens, Cookies oder Geheimnisse.

Frame-Formen:

- Anfrage: `{type:"req", id, method, params}`
- Antwort: `{type:"res", id, ok, payload|error}`
- Ereignis: `{type:"event", event, payload, seq?, stateVersion?}`

Antwortfehler verwenden `{ code, message, details?, retryable?, retryAfterMs? }`.
Clients sollten anhand von `code` und `details.code` verzweigen; `message` bleibt menschenlesbar
und kann sich ändern, sofern ein Kompatibilitätshinweis nichts anderes angibt. Autorisierungsfehler
auf Methodenebene verwenden das `code: "FORBIDDEN"`-Feld auf oberster Ebene mit strukturierten
Details zu fehlenden Scopes:

- Fehlender Scope: `{ code: "MISSING_SCOPE", missingScope, requiredScopes }`.
  `requiredScopes` ist die vollständige Menge bekannter Scopes für den angeforderten Vorgang.
  Die alte `missing scope: <scope>`-Meldung bleibt für ältere Clients erhalten.

Clients sollten zuerst `details` lesen und die alte Meldung nur als Kompatibilitäts-
Fallback verwenden. `readMissingScopeError` und `readMissingScopeErrorDetails` werden aus
`@openclaw/gateway-protocol/gateway-error-details` exportiert; der browsersichere Gateway-Client
exportiert sie erneut aus `@openclaw/gateway-client/browser`.

Die Schemas werden als `GatewayErrorDetailsSchema`,
`MissingScopeErrorDetailsSchema` aus `@openclaw/gateway-protocol/schema` exportiert.
HTTP-Scope-Fehler spiegeln das `MISSING_SCOPE`-Objekt unter `error.details` wider und
verwenden den HTTP-Status `403`.

Methoden mit Nebenwirkungen erfordern Idempotenzschlüssel (siehe Schema).

## Handshake

Das Gateway sendet vor dem Verbindungsaufbau eine Challenge:

```json
{
  "type": "event",
  "event": "connect.challenge",
  "payload": { "nonce": "…", "ts": 1737264000000 }
}
```

Der Client antwortet mit `connect`:

```json
{
  "type": "req",
  "id": "…",
  "method": "connect",
  "params": {
    "minProtocol": 4,
    "maxProtocol": 4,
    "client": {
      "id": "cli",
      "version": "1.2.3",
      "platform": "macos",
      "mode": "operator"
    },
    "role": "operator",
    "scopes": ["operator.read", "operator.write"],
    "caps": [],
    "commands": [],
    "permissions": {},
    "auth": { "token": "…" },
    "locale": "en-US",
    "userAgent": "openclaw-cli/1.2.3",
    "device": {
      "id": "device_fingerprint",
      "publicKey": "…",
      "signature": "…",
      "signedAt": 1737264000000,
      "nonce": "…"
    }
  }
}
```

Das Gateway antwortet mit `hello-ok`:

```json
{
  "type": "res",
  "id": "…",
  "ok": true,
  "payload": {
    "type": "hello-ok",
    "protocol": 4,
    "server": { "version": "…", "connId": "…" },
    "features": { "methods": ["…"], "events": ["…"] },
    "snapshot": { "…": "…" },
    "auth": {
      "role": "operator",
      "scopes": ["operator.read", "operator.write"]
    },
    "policy": {
      "maxPayload": 26214400,
      "maxBufferedBytes": 52428800,
      "tickIntervalMs": 15000
    }
  }
}
```

`server`, `features`, `snapshot`, `policy` und `auth` sind gemäß
`HelloOkSchema` (`packages/gateway-protocol/src/schema/frames.ts`) alle erforderlich. `auth`
meldet die ausgehandelte Rolle und die ausgehandelten Scopes, selbst wenn kein Geräte-Token ausgegeben wird (Form
siehe oben). `pluginSurfaceUrls` ist optional und ordnet Plugin-Oberflächennamen (z. B.
`canvas`) URLs mit Scope für gehostete Inhalte zu; der Eintrag kann ablaufen, daher rufen Nodes
`node.pluginSurface.refresh` mit `{ "surface": "canvas" }` auf, um einen neuen Eintrag zu erhalten.
Der veraltete Pfad `canvasHostUrl` / `canvasCapability` / `node.canvas.capability.refresh`
wird nicht unterstützt; verwenden Sie Plugin-Oberflächen.
Das optionale `appliedConfigHash` des Snapshots ist die aufgelöste Revision der Quellkonfiguration,
die von der aktiven Gateway-Laufzeit akzeptiert wurde. Clients können sie mit
`config.get.configRevisionHash` vergleichen, um festzustellen, ob eine neuere gespeicherte Konfiguration weiterhin
einen Neustart erfordert. `config.get.hash` bleibt die rohe Revision der Stammdatei, die von
Konfliktsicherungen beim Schreiben der Konfiguration verwendet wird.

Während das Gateway den Start seiner Sidecars noch abschließt, kann `connect` einen
wiederholbaren `UNAVAILABLE`-Fehler mit `details.reason: "startup-sidecars"` und
`retryAfterMs` zurückgeben. Wiederholen Sie den Vorgang innerhalb Ihres Verbindungsbudgets, statt dies als
endgültigen Handshake-Fehler zu behandeln.

Wenn ein Geräte-Token ausgegeben wird, fügt `hello-ok.auth` es hinzu:

```json
{
  "auth": {
    "deviceToken": "…",
    "role": "operator",
    "scopes": ["operator.read", "operator.write"]
  }
}
```

Der integrierte Bootstrap über QR-/Einrichtungscode ist ein Übergabepfad für Mobilgeräte. Eine erfolgreiche
Basisverbindung per Einrichtungscode gibt ein primäres Node-Token sowie ein eingeschränktes
Operator-Token zurück:

```json
{
  "auth": {
    "deviceToken": "…",
    "role": "node",
    "scopes": [],
    "deviceTokens": [
      {
        "deviceToken": "…",
        "role": "operator",
        "scopes": ["operator.approvals", "operator.read", "operator.talk.secrets", "operator.write"]
      }
    ]
  }
}
```

Diese Operator-Übergabe ist absichtlich eingeschränkt: Sie reicht aus, um die mobile
Operator-Schleife und die native Einrichtung zu starten, einschließlich `operator.talk.secrets` für Lesevorgänge
der Talk-Konfiguration, enthält jedoch keine Scopes für Pairing-Änderungen und kein `operator.admin`. Umfassenderer
Pairing-/Administratorzugriff erfordert einen separaten genehmigten Pairing- oder Token-Ablauf. Speichern Sie
`hello-ok.auth.deviceTokens` nur dauerhaft, wenn die Bootstrap-Authentifizierung über einen vertrauenswürdigen
Transport erfolgte (`wss://` oder Loopback/lokales Pairing).

Vertrauenswürdige Backend-Clients im selben Prozess (`client.id: "gateway-client"`,
`client.mode: "backend"`) dürfen `device` bei direkten Loopback-Verbindungen weglassen, wenn sie
sich mit dem gemeinsam genutzten Gateway-Token/-Passwort authentifizieren. Dieser Pfad ist
internen RPCs der Steuerungsebene vorbehalten (z. B. Sitzungsaktualisierungen von Subagenten) und verhindert,
dass veraltete CLI-/Geräte-Pairing-Baselines lokale Backend-Arbeit blockieren. Entfernte,
aus Browsern stammende, Node- sowie explizite Geräte-Token-/Geräteidentitäts-Clients durchlaufen weiterhin
die normalen Pairing- und Scope-Upgrade-Prüfungen.

### Worker-Rolle und geschlossenes Protokoll

Cloud-Worker verwenden einen dedizierten Loopback-Eingang durch den Gateway-eigenen,
per Hostschlüssel fixierten SSH-Tunnel. Er akzeptiert ausschließlich Worker-Identitäten und leitet niemals
allgemeine Authentifizierung, Node-Ereignisse, Operator-RPCs oder Plugin-Methoden weiter. Ein striktes `connect`
verifiziert einen im Ruhezustand gehashten, kurzlebigen Berechtigungsnachweis, der an die Umgebung, den Bundle-
Hash, die Eigentümerepoche, die RPC-Satzversion, das Ablaufdatum und eine nullable Sitzung gebunden ist; zusätzlich
werden die aktuelle Version und der Funktionsumfang separat geprüft. Bei Erfolg wird ein minimales
`worker-hello-ok` zurückgegeben; die Funktionsaushandlung ist von der allgemeinen Protokollversion
unabhängig. Frames bleiben unter 64 KiB, mit Ausnahme eines ausgehandelten `worker.inference.start`-
Frames, der bis zu 25 MiB groß sein darf. Die geschlossene Positivliste enthält `worker.heartbeat`,
`worker.transcript.commit`, `worker.live-event`, `worker.inference.start` und
`worker.inference.cancel`.

Transkript-Commits verwenden Fencing anhand der Eigentümerepoche, eine Gateway-eigene Sitzungsbindung,
Compare-and-Swap des Basisblatts und dauerhafte Sequenzwiedergabe; das Gateway erzeugt
Transkripteintrags- und übergeordnete IDs über den normalen Sitzungsschreiber. Eigentümerschaft und
Ablauf werden bei jedem RPC erneut geprüft.

### Client-Fähigkeiten

Operator-Clients können in `connect.params.caps` optionale Fähigkeiten ankündigen:

- `tool-events`: akzeptiert strukturierte Ereignisse zum Lebenszyklus von Tools.
- `inline-widgets`: kann Ergebnisse gehosteter Inline-Widget-Tools darstellen.

Client-Fähigkeiten beschreiben den verbundenen Client, nicht die Autorisierung. Agent-Tools können erforderliche Fähigkeiten deklarieren; das Gateway lässt diese Tools aus, sofern nicht jede Anforderung in `caps` des ursprünglichen Clients enthalten ist. Über einen Kanal gestartete Ausführungen verfügen über keine Gateway-Client-Fähigkeiten, daher sind fähigkeitsgebundene Tools auch dann nicht verfügbar, wenn die Tool-Richtlinie sie ausdrücklich zulässt.

### Beispiel für eine Node-Verbindung

```json
{
  "type": "req",
  "id": "…",
  "method": "connect",
  "params": {
    "minProtocol": 4,
    "maxProtocol": 4,
    "client": {
      "id": "ios-node",
      "version": "1.2.3",
      "platform": "ios",
      "mode": "node"
    },
    "role": "node",
    "scopes": [],
    "caps": ["camera", "canvas", "screen", "location", "voice"],
    "commands": ["camera.snap", "canvas.navigate", "screen.record", "location.get"],
    "permissions": { "camera.capture": true, "screen.record": false },
    "auth": { "token": "…" },
    "locale": "en-US",
    "userAgent": "openclaw-ios/1.2.3",
    "device": {
      "id": "device_fingerprint",
      "publicKey": "…",
      "signature": "…",
      "signedAt": 1737264000000,
      "nonce": "…"
    }
  }
}
```

Nodes deklarieren beim Verbindungsaufbau Fähigkeitsansprüche:

- `caps`: übergeordnete Kategorien wie `camera`, `canvas`, `screen`,
  `location`, `voice`, `talk`.
- `commands`: Positivliste der Befehle für Aufrufe.
- `permissions`: granulare Umschalter (z. B. `screen.record`, `camera.capture`).

Das Gateway behandelt diese Angaben als Ansprüche und erzwingt serverseitige Positivlisten.

## Rollen und Scopes

Das vollständige Operator-Scope-Modell, Prüfungen zum Genehmigungszeitpunkt und die
Semantik gemeinsam genutzter Geheimnisse finden Sie unter [Operator-Scopes](/de/gateway/operator-scopes).

Rollen:

- `operator`: Client der Steuerungsebene (CLI/UI/Automatisierung).
- `node`: Fähigkeitshost (Kamera/Bildschirm/Canvas/system.run).
- `worker`: Cloud-Ausführungshost im dedizierten, geschlossenen Worker-Protokoll.

Operator-Scopes (`src/gateway/operator-scopes.ts`), die vollständige geschlossene Menge:

- `operator.read`
- `operator.write`
- `operator.admin`
- `operator.approvals`
- `operator.pairing`
- `operator.talk.secrets`

`talk.config` mit `includeSecrets: true` erfordert `operator.talk.secrets` (oder
`operator.admin`). Wenn Geheimnisse enthalten sind, lesen Sie den Berechtigungsnachweis des aktiven Talk-Providers
aus `talk.resolved.config.apiKey`; `talk.providers.<id>.apiKey`
behält die Form der Quelle bei und kann ein SecretRef-Objekt oder eine redigierte Zeichenfolge sein.

Vom Plugin registrierte Gateway-RPC-Methoden können einen eigenen Operator-Scope anfordern,
diese reservierten Core-Präfixe werden jedoch immer zu `operator.admin`
(`src/shared/gateway-method-policy.ts`) aufgelöst: `config.*`, `exec.approvals.*`,
`wizard.*`, `update.*`.

Der Methoden-Scope ist nur die erste Zugriffsschranke. Einige über
`chat.send` erreichte Slash-Befehle wenden strengere Prüfungen auf Befehlsebene an: Dauerhafte Schreibvorgänge für `/config set` und
`/config unset` erfordern `operator.admin`, selbst bei Gateway-Clients, die
bereits über einen niedrigeren Operator-Scope verfügen.

`node.pair.approve` verfügt zusätzlich zum grundlegenden
Methoden-Scope (`operator.pairing`) über eine weitere Scope-Prüfung zum Genehmigungszeitpunkt,
die auf dem deklarierten `commands` (`src/infra/node-pairing-authz.ts`) der ausstehenden Anfrage basiert:

| Deklarierte Befehle                                                                                                           | Erforderliche Berechtigungsbereiche    |
| ----------------------------------------------------------------------------------------------------------------------------- | ------------------------------------- |
| keine                                                                                                                         | `operator.pairing`                    |
| gewöhnliche Befehle                                                                                                           | `operator.pairing` + `operator.write` |
| enthält `system.run`, `system.run.prepare`, `system.which`, `browser.proxy`, `fs.listDir` oder `system.execApprovals.get/set` | `operator.pairing` + `operator.admin` |

### Fähigkeiten/Befehle/Berechtigungen (Node)

Nodes deklarieren beim Verbindungsaufbau ihre beanspruchten Fähigkeiten:

- `caps`: übergeordnete Fähigkeitskategorien wie `camera`, `canvas`, `screen`,
  `location`, `voice` und `talk`.
- `commands`: Befehls-Zulassungsliste für Aufrufe.
- `permissions`: granulare Umschalter (z. B. `screen.record`, `camera.capture`).

Der Gateway behandelt diese als **Angaben** und erzwingt serverseitige Zulassungslisten.
Verbundene Nodes können nach einem erfolgreichen Verbindungsaufbau oder
erneuten Verbindungsaufbau mit `node.pluginTools.update` optionale, für Agenten sichtbare Plugin- oder MCP-Tool-
Deskriptoren veröffentlichen. Headless-Node-Hosts werden neu gestartet, um Änderungen
am deklarativen MCP-Inventar anzuwenden. Diese Aktualisierungsmethode ist der einzige Veröffentlichungspfad;
Plugin-Tool-Deskriptoren werden in den Parametern von `connect` nicht akzeptiert. Jeder Deskriptor muss
einen Provider-sicheren Tool-`name` verwenden und einen `command` aus der aktuellen
Befehls-Zulassungsliste des Nodes benennen. Der Gateway vertraut den Deskriptor-Metadaten des gekoppelten
Nodes, filtert Deskriptoren außerhalb der genehmigten Befehlsoberfläche, entfernt sie beim Trennen
des Nodes und weist Versuche von Operatoren zurück, den Katalog eines anderen Nodes zu ändern. Setzen Sie
`gateway.nodes.pluginTools.enabled: false`, um von Nodes veröffentlichte Deskriptoren zu ignorieren.

Verbundene Node-Hosts veröffentlichen ihren vollständigen Skill-Ersetzungskatalog mit
`node.skills.update`. Diese Methode der Node-Rolle ist der einzige Veröffentlichungspfad für Node-Skills;
Skills werden in den Parametern von `connect` nicht akzeptiert. Jeder Deskriptor enthält einen
sicheren Namen, eine Beschreibung und begrenzten `SKILL.md`-Inhalt. Der Gateway verarbeitet diesen
Inhalt mit dem normalen Skills-Loader, nimmt ihn in die Skill-Snapshots der Agenten auf,
solange der Node verbunden ist, und entfernt ihn beim Trennen der Verbindung. Setzen Sie
`gateway.nodes.allowSkills: false`, um von Nodes veröffentlichte Skills zu ignorieren.

## Präsenz

- `system-presence` gibt nach Geräteidentität indizierte Einträge zurück, einschließlich
  `deviceId`, `roles` und `scopes`, sodass Benutzeroberflächen auch dann eine Zeile pro Gerät anzeigen können,
  wenn es sowohl als Operator als auch als Node verbunden ist.
- `node.list` enthält optional `lastSeenAtMs` und `lastSeenReason`. Verbundene
  Nodes melden die aktuelle Verbindungszeit mit dem Grund `connect`; gekoppelte Nodes können
  außerdem über ein vertrauenswürdiges Node-Ereignis eine dauerhafte Hintergrundpräsenz melden.

Native macOS-Nodes können außerdem authentifizierte `node.presence.activity`-Ereignisse
mit begrenzter Leerlaufzeit der Eingabe senden. Der Gateway leitet Aktivitätszeitstempel anhand seiner
eigenen Uhr ab, stellt den zuletzt aktiven verbundenen Mac über `node.list` und
`node.describe` bereit und überträgt `node.presence`-Aktualisierungen an Clients mit Leseberechtigung.
Die App sendet `{ "action": "clear" }`, wenn der Benutzer die Aktivitätsfreigabe deaktiviert;
der Gateway löscht Zeitstempel nur für genau diese authentifizierte Node-Verbindung.
Gateways, die älter als diese bestätigte Aktion sind, geben sie als unbehandelt zurück, sodass der Mac-
Node die Verbindung einmal neu herstellt und die Bereinigung beim Trennen den alten Verbindungsstatus entfernt.
Unter [Präsenz des aktiven Computers](/de/nodes/presence) finden Sie Informationen zu Auswahl, Datenschutz, Modell-
kontext und dem Verhalten beim Weiterleiten von Benachrichtigungen.

### Hintergrund-Aktivereignis des Nodes

Nodes rufen `node.event` mit `event: "node.presence.alive"` auf, um zu erfassen, dass ein
gekoppelter Node während einer Hintergrundaktivierung aktiv war, ohne ihn als verbunden zu markieren:

```json
{
  "event": "node.presence.alive",
  "payloadJSON": "{\"trigger\":\"silent_push\",\"sentAtMs\":1737264000000,\"displayName\":\"Peters iPhone\",\"version\":\"2026.4.28\",\"platform\":\"iOS 18.4.0\",\"deviceFamily\":\"iPhone\",\"modelIdentifier\":\"iPhone17,1\",\"pushTransport\":\"relay\"}"
}
```

`trigger` ist eine geschlossene Aufzählung: `background`, `silent_push`, `bg_app_refresh`,
`significant_location`, `manual`, `connect`. Unbekannte Werte werden zu
`background` (`src/shared/node-presence.ts`) normalisiert. Das Ereignis wird nur für
authentifizierte Node-Gerätesitzungen dauerhaft gespeichert; Sitzungen ohne Gerät oder ohne Kopplung geben
`handled: false` zurück.

Erfolgreiche Gateways geben ein strukturiertes Ergebnis zurück:

```json
{
  "ok": true,
  "event": "node.presence.alive",
  "handled": true,
  "reason": "persisted"
}
```

Ältere Gateways geben für `node.event` möglicherweise nur `{ "ok": true }` zurück; behandeln Sie dies
als bestätigten RPC-Aufruf und nicht als dauerhafte Speicherung der Präsenz.

## Geltungsbereich von Broadcast-Ereignissen

Vom Server übertragene Broadcast-Ereignisse sind durch Berechtigungsbereiche beschränkt, damit Sitzungen,
die auf die Kopplung oder ausschließlich auf Nodes beschränkt sind, nicht passiv Sitzungsinhalte empfangen
(`src/gateway/server-broadcast.ts`):

- Chat-, Agenten- und Tool-Ergebnis-Frames (gestreamte `agent`-Ereignisse, Tool-Ergebnis-
  Ereignisse) erfordern mindestens `operator.read`. Sitzungen ohne diese Berechtigung überspringen diese
  Frames vollständig.
- Von Plugins definierte `plugin.*`-Broadcasts sind standardmäßig auf `operator.write` oder
  `operator.admin` beschränkt; explizite Einträge wie
  `plugin.approval.requested` / `plugin.approval.resolved` verwenden
  stattdessen `operator.approvals`.
- Status-/Transportereignisse (`heartbeat`, `presence`, `tick`, Lebenszyklus
  von Verbindungsaufbau/-trennung) bleiben unbeschränkt, sodass der Transportzustand für jede
  authentifizierte Sitzung sichtbar ist.
- Unbekannte Familien von Broadcast-Ereignissen werden standardmäßig durch Berechtigungsbereiche beschränkt
  (Fail-Closed), sofern ein registrierter Handler diese Beschränkung nicht ausdrücklich lockert.

Jede Clientverbindung verwaltet ihre eigene clientbezogene Sequenznummer, sodass Broadcasts
auf diesem Socket monoton geordnet bleiben, selbst wenn verschiedene Clients
unterschiedliche, nach Berechtigungsbereichen gefilterte Teilmengen des Ereignisstroms sehen.

## RPC-Methodenfamilien

`hello-ok.features.methods` ist eine konservative Ermittlungsliste, die aus
`src/gateway/server-methods-list.ts` sowie den exportierten Methoden geladener Plugins/Kanäle
erstellt wird – sie ist kein generierter Auszug aller Methoden, und einige Methoden (zum
Beispiel `push.test`, `web.login.start`, `web.login.wait`, `sessions.usage`)
sind absichtlich von der Ermittlung ausgeschlossen, obwohl es sich um echte, aufrufbare
Methoden handelt. Betrachten Sie dies als Funktionsermittlung und nicht als vollständige Aufzählung von
`src/gateway/server-methods/*.ts`.

<AccordionGroup>
  <Accordion title="System und Identität">
    - `health` gibt den zwischengespeicherten oder neu abgefragten Zustands-Snapshot des Gateways zurück.
    - `diagnostics.stability` gibt die aktuelle begrenzte Diagnoseaufzeichnung der Stabilität zurück: Ereignisnamen, Anzahlen, Bytegrößen, Speicherwerte, Warteschlangen-/Sitzungsstatus, Kanal-/Plugin-Namen, Sitzungs-IDs. Keine Chattexte, Webhook-Inhalte, Tool-Ausgaben, unverarbeiteten Anfrage-/Antwortinhalte, Token, Cookies oder Geheimnisse. Erfordert `operator.read`.
    - `status` gibt die Gateway-Zusammenfassung im Stil von `/status` zurück; vertrauliche Felder nur für Operator-Clients mit Administratorberechtigung.
    - `gateway.identity.get` gibt die Gateway-Geräteidentität zurück, die von Relay- und Kopplungsabläufen verwendet wird.
    - `system-presence` gibt den aktuellen Präsenz-Snapshot für verbundene Operator-/Node-Geräte zurück.
    - `system-event` fügt ein Systemereignis an und kann den Präsenzkontext aktualisieren/übertragen.
    - `last-heartbeat` gibt das zuletzt dauerhaft gespeicherte Heartbeat-Ereignis zurück.
    - `set-heartbeats` schaltet die Heartbeat-Verarbeitung auf dem Gateway ein oder aus.
    - `gateway.suspend.prepare` erstellt nur dann eine kurze Lease zur kooperativen Unterbrechung, wenn die nachverfolgte Gateway-Arbeit inaktiv ist. `gateway.suspend.status` prüft diese Lease, und `gateway.suspend.resume` gibt sie nach dem Reaktivieren oder einem abgebrochenen Hostvorgang frei.

  </Accordion>

  <Accordion title="Modelle und Nutzung">
    - `models.list` gibt den zur Laufzeit zulässigen Modellkatalog zurück. Siehe „`models.list`-Ansichten“ unten.
    - `usage.status` gibt Nutzungsfenster/Zusammenfassungen des verbleibenden Kontingents des Providers zurück.
    - `usage.cost` gibt aggregierte Kostennutzungszusammenfassungen für einen Datumsbereich zurück. Übergeben Sie `agentId` für einen Agenten oder `agentScope: "all"`, um konfigurierte Agenten zu aggregieren.
    - `doctor.memory.status` gibt die Bereitschaft des Vektorspeichers/zwischengespeicherter Embeddings für den aktiven Standard-Agenten-Workspace zurück. Übergeben Sie `{ "probe": true }` oder `{ "deep": true }` nur für einen expliziten Live-Ping des Embedding-Providers. Übergeben Sie `{ "agentId": "agent-id" }`, um die Statistiken des Dreaming-Speichers auf einen Agenten-Workspace zu beschränken; ohne diesen Parameter werden konfigurierte Dreaming-Workspaces aggregiert.
    - `doctor.memory.dreamDiary`, `doctor.memory.backfillDreamDiary`, `doctor.memory.resetDreamDiary`, `doctor.memory.resetGroundedShortTerm`, `doctor.memory.repairDreamingArtifacts` und `doctor.memory.dedupeDreamDiary` akzeptieren optional `{ "agentId": "agent-id" }`; ohne diesen Parameter verwenden sie den konfigurierten Standard-Agenten-Workspace.
    - `doctor.memory.remHarness` gibt für Remote-Clients der Steuerungsebene eine begrenzte, schreibgeschützte Vorschau des REM-Harness zurück, einschließlich Workspace-Pfaden, Speicherausschnitten, gerendertem fundiertem Markdown und Kandidaten für die tiefgreifende Hochstufung. Erfordert `operator.read`.
    - `sessions.usage` gibt Nutzungszusammenfassungen pro Sitzung zurück. Übergeben Sie `agentId` für einen Agenten oder `agentScope: "all"`, um konfigurierte Agenten gemeinsam aufzulisten.
      Beide Nutzungsmethoden akzeptieren `mode: "specific"` mit einer IANA-`timeZone` für DST-konforme Grenzen und Buckets von Kalendertagen. `utcOffset` wird weiterhin für ältere Clients und als Fallback unterstützt, wenn die Gateway-Laufzeit die angeforderte Zone nicht erkennt.
    - `sessions.usage.timeseries` gibt Zeitreihennutzungsdaten für eine Sitzung zurück.
    - `sessions.usage.logs` gibt Nutzungsprotokolleinträge für eine Sitzung zurück.

  </Accordion>

  <Accordion title="Kanäle und Anmeldehilfen">
    - `channels.status` gibt Statuszusammenfassungen integrierter und gebündelter Kanäle/Plugins zurück.
    - `channels.logout` meldet einen bestimmten Kanal/ein bestimmtes Konto ab, sofern der Kanal dies unterstützt.
    - `web.login.start` startet einen QR-/Web-Anmeldeablauf für den aktuellen Webkanal-Provider mit QR-Unterstützung.
    - `web.login.wait` wartet auf den Abschluss dieses Ablaufs und startet bei Erfolg den Kanal.
    - `push.test` sendet eine APNs-Test-Push-Nachricht an einen registrierten iOS-Node.
    - `voicewake.get` gibt die gespeicherten Aktivierungswort-Trigger zurück.
    - `voicewake.set` aktualisiert Aktivierungswort-Trigger und überträgt die Änderung.

  </Accordion>

  <Accordion title="Plugin-Verwaltung">
    - `plugins.list` (`operator.read`) gibt das Inventar der installierten Plugins sowie lokal kuratierte offizielle Empfehlungen, Diagnosedaten und die Information zurück, ob der aktuelle Installationsmodus Änderungen zulässt.
    - `plugins.search` (`operator.read`) sucht nach installierbaren ClawHub-Code-Plugin- und Bundle-Plugin-Familien. Übergeben Sie einen nicht leeren Wert für `query` und optional einen Wert für `limit` von 1 bis 100.
    - `plugins.install` (`operator.admin`) installiert entweder einen offiziellen Katalogeintrag mit `{ source: "official", pluginId }` oder ein ClawHub-Paket mit `{ source: "clawhub", packageName, version?, acknowledgeClawHubRisk? }`. ClawHub-Installationen behalten die Prüfungen des Gateways für Vertrauenswürdigkeit, Integrität und Installationsrichtlinien bei. Erfolgreiche Installationen erfordern einen Neustart des Gateways.
    - `plugins.setEnabled` (`operator.admin`) ändert mit `{ pluginId, enabled }` die Aktivierungsrichtlinie eines installierten Plugins. Die Antwort enthält den aktualisierten Katalogeintrag, Neustartmetadaten und etwaige Warnungen zur Slot-Auswahl.
    - `plugins.uninstall` (`operator.admin`) entfernt mit `{ pluginId }` ein extern installiertes Plugin: Konfigurationsverweise, den Installationsdatensatz und verwaltete Dateien. Mitgelieferte Plugins können nicht deinstalliert, sondern nur deaktiviert werden. Die Antwort listet die Entfernungsvorgänge auf und erfordert immer einen Neustart des Gateways.

  </Accordion>

  <Accordion title="Nachrichten und Protokolle">
    - `send` ist der direkte RPC für ausgehende Zustellungen an bestimmte Kanäle, Konten und Threads außerhalb des Chat-Runners.
    - `logs.tail` gibt das konfigurierte Ende des Gateway-Dateiprotokolls mit Steuerelementen für Cursor, Limit und maximale Byteanzahl zurück.

  </Accordion>

  <Accordion title="Operator-Terminal">
    - `terminal.open` startet ein Host-PTY für einen expliziten Wert von `agentId` oder den Standard-Agenten und gibt den aufgelösten Agenten, das Arbeitsverzeichnis, die Shell und den Einschränkungsstatus zurück.
    - `terminal.input`, `terminal.resize` und `terminal.close` arbeiten ausschließlich mit Sitzungen, deren Eigentümer die aufrufende Verbindung ist.
    - `terminal.upload` akzeptiert eine Base64-Datei mit bis zu 16 MiB, stellt sie in einem privaten temporären Verzeichnis mit einer Lebensdauer von 24 Stunden auf dem Gateway der Sitzung oder dem Host des gekoppelten Nodes bereit und gibt den absoluten Pfad zurück. Der Aufrufer muss diesen Pfad weiterhin einfügen oder anderweitig verwenden; der RPC schreibt niemals Terminaleingaben und führt keinen Befehl aus.
    - Ereignisse vom Typ `terminal.data` und `terminal.exit` werden ausschließlich an die Verbindung gestreamt, der die Sitzung gehört.
    - Sitzungen, deren Verbindung abbricht, werden getrennt, nicht beendet: Sie können für `gateway.terminal.detachedSessionTimeoutSeconds` (Standardwert 300; `0` stellt das Beenden bei Verbindungstrennung wieder her) erneut verbunden werden, während sich die jüngsten Ausgaben in einem begrenzten serverseitigen Puffer ansammeln.
    - `terminal.list` gibt verbindbare Sitzungen zurück; `terminal.attach` bindet eine aktive oder getrennte Sitzung erneut an die aufrufende Verbindung und gibt den Wiedergabepuffer zurück (Übernahme im tmux-Stil – ein vorheriger aktiver Eigentümer erhält `terminal.exit` mit dem Grund `detached`); `terminal.text` liest den Puffer als Klartext, ohne eine Verbindung herzustellen.
    - Jede Terminalmethode erfordert `operator.admin`; `gateway.terminal.enabled` muss explizit auf „true“ gesetzt sein. Vollständig sandboxgeschützte Agenten werden abgelehnt, und eine Änderung der Agentenrichtlinie schließt bestehende und gerade gestartete PTYs einschließlich der getrennten PTYs.

  </Accordion>

  <Accordion title="Talk und TTS">
    - `talk.catalog` gibt den schreibgeschützten Talk-Provider-Katalog für Sprache, Streaming-Transkription und Echtzeit-Sprachkommunikation zurück: kanonische Provider-IDs, Registry-Aliasse, Bezeichnungen, Konfigurationsstatus, ein optionales Ergebnis für `ready` auf Gruppenebene, verfügbare Modell-/Sprach-IDs, kanonische Modi, Transporte, Brain-Strategien sowie Echtzeit-Audio-/Funktionsflags, ohne Provider-Geheimnisse zurückzugeben oder die globale Konfiguration zu ändern. Aktuelle Gateways setzen `ready`, nachdem die Provider-Auswahl zur Laufzeit angewendet wurde; behandeln Sie das Fehlen dieses Werts bei älteren Gateways als nicht verifiziert.
    - `talk.config` gibt die effektive Talk-Konfigurationsnutzlast zurück; `includeSecrets` erfordert `operator.talk.secrets` (oder `operator.admin`).
    - `talk.session.create` erstellt eine Gateway-eigene Talk-Sitzung für `realtime/gateway-relay`, `transcription/gateway-relay` oder `stt-tts/managed-room`. Bei `stt-tts/managed-room` müssen Aufrufer von `operator.write`, die `sessionKey` übergeben, zusätzlich `spawnedBy` übergeben, damit Sitzungsschlüssel innerhalb des Gültigkeitsbereichs sichtbar sind; die Erstellung eines nicht bereichsgebundenen `sessionKey` und `brain: "direct-tools"` erfordern `operator.admin`.
    - `talk.session.join` validiert ein Sitzungstoken für einen verwalteten Raum, gibt bei Bedarf `session.ready` oder `session.replaced` aus und gibt Raum-/Sitzungsmetadaten sowie die jüngsten Talk-Ereignisse zurück, jedoch niemals das Klartexttoken oder dessen Hash.
    - `talk.session.appendAudio` fügt Gateway-eigenen Echtzeit-Relay- und Transkriptionssitzungen Base64-codierte PCM-Eingangsaudiodaten hinzu.
    - `talk.session.startTurn`, `talk.session.endTurn` und `talk.session.cancelTurn` steuern den Lebenszyklus von Gesprächsbeiträgen in verwalteten Räumen und lehnen veraltete Gesprächsbeiträge ab, bevor der Status gelöscht wird.
    - `talk.session.cancelOutput` stoppt die Audioausgabe des Assistenten, hauptsächlich für durch VAD gesteuerte Unterbrechungen in Gateway-Relay-Sitzungen.
    - `talk.session.submitToolResult` schließt einen Provider-Tool-Aufruf ab, der von einer Gateway-eigenen Echtzeit-Relay-Sitzung ausgegeben wurde. Die Anfrage wartet auf alle asynchronen Abschlusssignale, die von der Provider-Bridge bereitgestellt werden; fehlgeschlagene Übermittlungen lassen den verknüpften Lauf aktiv und geben kein Ereignis für ein erfolgreiches Tool-Ergebnis aus. Übergeben Sie `options: { willContinue: true }` für vorläufige Tool-Ausgaben oder `options: { suppressResponse: true }`, wenn die Provider-Bridge Unterstützung für die Unterdrückung signalisiert und das Ergebnis keine weitere Antwort starten soll.
    - `talk.session.steer` sendet die Sprachsteuerung für einen aktiven Lauf an eine Gateway-eigene, agentengestützte Talk-Sitzung: `{ sessionId, text, mode? }`, wobei `mode` entweder `status`, `steer`, `cancel` oder `followup` ist; ein ausgelassener Modus wird anhand des gesprochenen Textes klassifiziert.
    - `talk.session.close` schließt eine Gateway-eigene Relay-, Transkriptions- oder verwaltete Raumsitzung und gibt abschließende Talk-Ereignisse aus.
    - `talk.mode` setzt den aktuellen Talk-Modusstatus für WebChat-/Control-UI-Clients und überträgt ihn.
    - `talk.client.create` erstellt eine clientseitig verwaltete Echtzeit-Provider-Sitzung mithilfe von `webrtc` oder `provider-websocket` oder setzt sie fort, während das Gateway Anmeldedaten, Anweisungen, Tool-Richtlinien und den zurückgegebenen Wert `voiceSessionId` verwaltet. Clients übergeben `sessionKey` und verwenden `voiceSessionId` erneut, wenn der Provider-Transport während eines Anrufs ersetzt wird.
    - `talk.client.transcript` fügt der normalen Agentensitzung ein abgeschlossenes Element vom Typ `{ role, text }` hinzu. Der erforderliche Wert `entryId` ist innerhalb von `voiceSessionId` idempotent; Wiederholungsversuche duplizieren keine Transkriptnachrichten.
    - `talk.client.close` schließt die logische Sprachsitzung nach ausstehenden Transkriptschreibvorgängen. Das Schließen ist idempotent und kann eine ausschließlich Änderungen enthaltende Anrufzusammenfassung an den letzten Kanal der Sitzung senden, der nicht WebChat ist.
    - `talk.client.toolCall` ermöglicht clientseitig verwalteten Echtzeittransporten, Provider-Tool-Aufrufe an die Gateway-Richtlinie weiterzuleiten. Das erste unterstützte Tool ist `openclaw_agent_consult`; Clients erhalten eine Lauf-ID und warten auf normale Chat-Lebenszyklusereignisse, bevor sie das providerspezifische Tool-Ergebnis übermitteln. Sprachgebundene Aktionen mit hoher Auswirkung geben `VOICE_CONFIRMATION_REQUIRED:<id>` zurück, bis eine spätere abgeschlossene Benutzeräußerung genau diese Aktion ausdrücklich bestätigt und die nächste Abfrage `confirmationId` bereitstellt.
    - `talk.client.steer` sendet die Sprachsteuerung für einen aktiven Lauf an clientseitig verwaltete Echtzeittransporte. Das Gateway löst den aktiven eingebetteten Lauf aus `sessionKey` auf und gibt ein strukturiertes Ergebnis mit Annahme oder Ablehnung zurück, anstatt die Steuerung stillschweigend zu verwerfen.
    - `talk.event` ist der zentrale Talk-Ereigniskanal für Echtzeit-, Transkriptions-, STT-/TTS-, verwaltete Raum-, Telefonie- und Meeting-Adapter.
    - `talk.speak` synthetisiert Sprache über den aktiven Talk-Sprach-Provider.
    - `tts.status` gibt den TTS-Aktivierungsstatus, den aktiven Provider, Fallback-Provider und den Provider-Konfigurationsstatus zurück.
    - `tts.providers` gibt das sichtbare TTS-Provider-Inventar zurück.
    - `tts.enable` und `tts.disable` schalten den Status der TTS-Einstellungen um.
    - `tts.setProvider` aktualisiert den bevorzugten TTS-Provider.
    - `tts.convert` führt eine einmalige Text-zu-Sprache-Konvertierung durch.
    - `tts.speak` (`operator.write`) rendert einen nicht leeren Wert von `text` mit der konfigurierten allgemeinen TTS-Provider-Kette und gibt einen vollständigen Clip inline als `audioBase64` sowie `provider` und optionale Metadaten für `outputFormat`, `mimeType` und `fileExtension` zurück. Im Gegensatz zu `tts.convert` wird kein Gateway-lokaler Pfad zurückgegeben; im Gegensatz zu `talk.speak` ist kein Talk-Provider erforderlich. Text oberhalb von `tts.maxTextLength` gibt `INVALID_REQUEST` zurück; Synthesefehler geben `UNAVAILABLE` zurück.

  </Accordion>

  <Accordion title="Secrets, Konfiguration, Aktualisierung und Assistent">
    - `secrets.reload` löst aktive SecretRefs erneut auf und veröffentlicht atomar einen eigentümerbezogenen Laufzeitstatus. Fehler berechtigter Eigentümer können mit `warningCount` als kalte oder veraltete Beeinträchtigung veröffentlicht werden; strikte oder nicht zugeordnete Fehler lehnen das erneute Laden ab und bewahren den aktiven Snapshot.
    - `secrets.resolve` löst Zuweisungen von Befehlsziel-Secrets für eine bestimmte Gruppe von Befehlen und Zielen auf.
    - `config.get` gibt den aktuellen Konfigurations-Snapshot auf dem Datenträger, die unverarbeitete Stammdatei `hash`, die aufgelöste `configRevisionHash` und optional `appliedConfigHash` für die aufgelöste Revision zurück, die von der aktiven Gateway-Laufzeit akzeptiert wurde.
    - `config.set` schreibt eine validierte Konfigurationsnutzlast.
    - `config.patch` führt eine partielle Konfigurationsaktualisierung zusammen. Eine destruktive Array-Ersetzung erfordert den betroffenen Pfad in `replacePaths`; verschachtelte Arrays unter Array-Einträgen verwenden `[]`-Pfade wie `agents.entries.*.skills`.
    - `config.apply` validiert und ersetzt die vollständige Konfigurationsnutzlast.
    - `config.schema` gibt die Nutzlast des aktiven Konfigurationsschemas zurück, die von Control UI und CLI-Werkzeugen verwendet wird: Schema, `uiHints`, Version, Generierungsmetadaten sowie Metadaten zu Plugin- und Kanalschemas, sofern sie geladen werden können. Sie enthält `title`- / `description`-Metadaten aus denselben Beschriftungen und Hilfetexten wie die UI, einschließlich verschachtelter Objekt-, Platzhalter- und Array-Element-Zweige sowie `anyOf`- / `oneOf`- / `allOf`-Kompositionszweige, wenn eine passende Felddokumentation vorhanden ist.
    - `config.schema.lookup` gibt für einen Konfigurationspfad eine pfadbezogene Suchnutzlast zurück: normalisierter Pfad, ein flacher Schemaknoten, übereinstimmender Hinweis und `hintPath`, optional `reloadKind` sowie Zusammenfassungen der unmittelbar untergeordneten Elemente für die Detailnavigation in UI und CLI. `reloadKind` ist entweder `restart`, `hot` oder `none` (`src/config/schema.ts`) und entspricht für den angeforderten Pfad dem Planer zum erneuten Laden der Gateway-Konfiguration. Schemaknoten der Suche behalten die benutzerorientierte Dokumentation und gängige Validierungsfelder bei (`title`, `description`, `type`, `enum`, `const`, `format`, `pattern`, numerische Grenzen sowie Grenzen für Zeichenfolgen, Arrays und Objekte, `additionalProperties`, `deprecated`, `readOnly`, `writeOnly`). Zusammenfassungen untergeordneter Elemente stellen `key`, normalisierte `path`, `type`, `required`, `hasChildren`, optional `reloadKind` sowie die übereinstimmenden `hint` / `hintPath` bereit.
    - `update.run` führt den Gateway-Aktualisierungsablauf aus und plant einen Neustart nur, wenn die Aktualisierung erfolgreich war; Aufrufer mit einer Sitzung können `continuationMessage` einbeziehen, sodass beim Start über die Fortsetzungswarteschlange für Neustarts eine weitere Agent-Ausführung fortgesetzt wird. Paketmanager-Aktualisierungen und überwachte Aktualisierungen von Git-Checkouts aus der Steuerungsebene verwenden eine getrennte Übergabe an einen verwalteten Dienst, anstatt den Paketbaum zu ersetzen oder Checkout-/Build-Ausgaben innerhalb des aktiven Gateways zu verändern. Eine gestartete Übergabe gibt `ok: true` mit `result.reason: "managed-service-handoff-started"` und `handoff.status: "started"` zurück. Eine zweite gleichzeitige `update.run`, die vom selben Gateway-Prozess verarbeitet wird, gibt `ok: false` mit `result.reason: "managed-service-handoff-already-running"` und `handoff.status: "already-running"` zurück; ihre Fortsetzung wird nicht akzeptiert, sodass der Aufrufer es erneut versuchen kann, nachdem die aktive Aktualisierung abgeschlossen ist. Eigenständige CLI-Aktualisierungsprogramme und ersetzende Gateway-Prozesse unterliegen nicht dieser prozesslokalen Schutzvorrichtung. Nicht verfügbare oder fehlgeschlagene Übergaben geben `ok: false` mit `managed-service-handoff-unavailable` oder `managed-service-handoff-failed` sowie `handoff.command` zurück, wenn eine manuelle Shell-Aktualisierung erforderlich ist. „Nicht verfügbar“ bedeutet, dass OpenClaw keine sichere Supervisor-Grenze oder dauerhafte Dienstidentität besitzt, beispielsweise `OPENCLAW_SYSTEMD_UNIT` für systemd. Während einer gestarteten Übergabe kann der Neustart-Sentinel kurzzeitig `stats.reason: "restart-health-pending"` melden; die Fortsetzung wird verzögert, bis die CLI das neu gestartete Gateway überprüft und den endgültigen `ok`-Sentinel schreibt.
    - `update.status` aktualisiert den neuesten Neustart-Sentinel der Aktualisierung und gibt ihn zurück, einschließlich der nach dem Neustart ausgeführten Version, sofern verfügbar.
    - `wizard.start`, `wizard.next`, `wizard.status` und `wizard.cancel` stellen den Onboarding-Assistenten über WS-RPC bereit.

  </Accordion>

  <Accordion title="Hilfsfunktionen für Agent und Arbeitsbereich">
    - `agents.list` gibt für das Gateway sichtbare Agent-Einträge zurück, einschließlich effektiver Modell-/Laufzeitmetadaten und optionaler semantischer `kind` (`agent` oder `system`). Clients geben beim Handshake die Fähigkeit `agent-kind` an, um die vollständige typisierte Liste zu erhalten; Clients ohne diese Fähigkeit behalten die veraltete, auswahlsichere Liste ohne Systemzeilen bei. Typbewusste Clients schließen `system`-Zeilen aus gewöhnlichen Auswahlfeldern aus, behalten sie jedoch in Diagnoseansichten bei. Ältere v4-Gateways können Zeilen ohne `kind` zurückgeben.
    - `agents.create`, `agents.update` und `agents.delete` verwalten Agent-Datensätze und die Verknüpfung von Arbeitsbereichen.
    - `agents.files.list`, `agents.files.get` und `agents.files.set` verwalten die Bootstrap-Arbeitsbereichsdateien, die für einen Agent bereitgestellt werden.
    - `audit.activity.list` gibt das versionierte, ausschließlich Metadaten enthaltende Aktivitätsjournal zurück; `audit.list` bleibt der kompatibilitätssichere RPC für Ausführungen und Werkzeuge.
    - `agents.workspace.list` und `agents.workspace.get` (`operator.read`) ermöglichen Clients in der vertrauenswürdigen Operatordomäne, die unter [Operatorbereiche](/de/gateway/operator-scopes) beschrieben ist, das schreibgeschützte, paginierte Durchsuchen des Arbeitsbereichsverzeichnisses eines Agents. Anfragen akzeptieren ausschließlich arbeitsbereichsrelative Pfade; Lesezugriffe bleiben auf das per realpath aufgelöste Stammverzeichnis des Arbeitsbereichs beschränkt (Ausbrüche über symbolische Links und Hardlinks werden abgelehnt), sind größenbeschränkt und auf UTF-8-Text sowie gängige Bildtypen (Base64) begrenzt. Antworten legen den Pfad des Arbeitsbereichs auf dem Host nicht offen. Dieser Namensraum enthält keine Schreiboperationen.
    - `tasks.list`, `tasks.get` und `tasks.cancel` stellen das Gateway-Aufgabenjournal für SDK- und Operator-Clients bereit. Siehe unten [RPCs des Aufgabenjournals](#task-ledger-rpcs).
    - `artifacts.list`, `artifacts.get` und `artifacts.download` stellen aus Transkripten abgeleitete Artefaktzusammenfassungen und Downloads für einen expliziten `sessionKey`-, `runId`- oder `taskId`-Bereich bereit. Ausführungs- und Aufgabenabfragen ermitteln serverseitig die zugehörige Sitzung und geben nur Transkriptmedien mit übereinstimmender Herkunft zurück; unsichere oder lokale URL-Quellen führen zu nicht unterstützten Downloads, anstatt serverseitig abgerufen zu werden.
    - `environments.list` und `environments.status` behalten die Gateway-lokale und Node-Umgebungserkennung bei. Konfigurierte Cloud-Worker und dauerhafte Datensätze, die von früheren Profilen hinterlassen wurden, fügen `worker`-Metadaten mit `providerId`, optional `leaseId`, `state`, `ageMs`, optional `idleMs` und `attachedSessionIds` hinzu. Die Lebenszykluszustände von Workern sind `requested`, `provisioning`, `bootstrapping`, `ready`, `attached`, `idle`, `draining`, `destroying`, `destroyed`, `failed` und `orphaned`.
    - `environments.create` (`{ profileId, idempotencyKey }`) stellt einen Worker aus einem konfigurierten Provider-Profil eines Plugins bereit; Wiederholungsversuche mit demselben Schlüssel verwenden den dauerhaften Vorgang erneut. `environments.destroy` (`{ environmentId }`) fordert den idempotenten Abbau einer dauerhaften Worker-Umgebung an. Beide erfordern `operator.admin`, sind Schreibvorgänge der Steuerungsebene und geben dieselbe Form der Umgebungszusammenfassung zurück, die von Statusantworten verwendet wird.
    - `agent.identity.get` gibt die effektive Assistentenidentität für einen Agent oder eine Sitzung zurück.
    - `agent.wait` wartet auf den Abschluss einer Ausführung und gibt den abschließenden Snapshot zurück, sofern verfügbar.

  </Accordion>

  <Accordion title="Sitzungssteuerung">
    - `sessions.list` gibt den aktuellen Sitzungsindex zurück, einschließlich zeilenbezogener `agentRuntime`-Metadaten, wenn ein Agent-Runtime-Backend konfiguriert ist. Wenn die Platzierung auf Cloud-Workern aktiviert ist oder ein dauerhafter Wiederherstellungszustand vorliegt, enthalten Sitzungszeilen außerdem einen abgeschlossenen `placement`-Zustand (`local`, `requested`, `provisioning`, `syncing`, `starting`, `active`, `draining`, `reconciling`, `reclaimed` oder `failed`) sowie zustandsspezifische Umgebungs-, Owner-Epoch-, Workspace-, Bundle-, ACK-Cursor- oder Wiederherstellungsfelder.
    - `sessions.subscribe` und `sessions.unsubscribe` schalten Abonnements für Sitzungsänderungsereignisse für den aktuellen WS-Client ein oder aus.
    - `sessions.messages.subscribe` und `sessions.messages.unsubscribe` schalten Abonnements für Transkript-/Nachrichtenereignisse einer Sitzung ein oder aus. Übergeben Sie `includeApprovals: true`, um zusätzlich bereinigte `session.approval`-Lebenszyklusereignisse für Genehmigungen zu empfangen, deren persistierte Zielgruppe genau diese Sitzung umfasst und deren Prüferbindung den abonnierenden Client autorisiert. Die Abonnementantwort enthält dann eine begrenzte ausstehende `approvalReplay`; sie ist maßgeblich, wenn `truncated` falsch ist. Die Zustimmung gilt pro Abonnementaufruf und ist nicht dauerhaft: Wenn dieselbe Sitzung ohne `includeApprovals: true` erneut abonniert wird, wird ein vorhandenes Genehmigungsabonnement entfernt. Zusätzlich zur normalen Leseberechtigung für Sitzungen erfordert diese Zustimmung `operator.admin` oder `operator.approvals` auf einem gekoppelten Gerät.
    - `sessions.preview` gibt begrenzte Transkriptvorschauen für bestimmte Sitzungsschlüssel zurück.
    - `sessions.describe` gibt eine Gateway-Sitzungszeile für einen exakten Sitzungsschlüssel zurück.
    - `sessions.resolve` löst ein Sitzungsziel auf oder kanonisiert es.
    - `sessions.create` erstellt einen neuen Sitzungseintrag. Optionale Werte für `model` und `thinkingLevel` persistieren das anfängliche Modell und die Reasoning-Überschreibungen atomar. `worktree: true` stellt einen verwalteten Worktree bereit; die optionalen Werte `worktreeBaseRef`/`worktreeName` wählen die Basisreferenz und den Branch-Namen aus, und `execNode` (`operator.admin`) bindet die Sitzungsausführung an einen Node-Host. Der erstellte Worktree wird im Ergebnis zurückgegeben und in der Sitzungszeile (`worktree: { id, branch, repoRoot }`) persistiert. Wenn der Eintrag erstellt wird, sein verschachtelter anfänglicher `chat.send` jedoch abgelehnt wird, enthält das erfolgreiche Ergebnis `runStarted: false` und `runError`; Clients können den Prompt beibehalten und den Versuch mit dem zurückgegebenen Sitzungsschlüssel wiederholen. Ein Aufrufer, der `parentSessionKey` mit `emitCommandHooks: true` übergibt, sollte außerdem die Lebenszyklusdisposition eines separaten untergeordneten Elements deklarieren: `succeedsParent: true` beendet das übergeordnete Element mit `session_end`, während `false` das übergeordnete Element aktiv hält und nur `session_start` des untergeordneten Elements ausgibt. Wird `succeedsParent` weggelassen, bleibt für bestehende Clients das bisherige Rollover-Verhalten des übergeordneten Elements erhalten. Die Disposition erfordert sowohl eine Verknüpfung mit dem übergeordneten Element als auch Command-Hooks; ein Fork kann sein übergeordnetes Element nicht erfolgreich abschließen. Das Reset-in-Place-Verhalten der Hauptsitzung bleibt unverändert, da kein separates untergeordnetes Element erstellt wird. Neue Zeilen werden über die vertrauenswürdige Erstellungsschnittstelle mit einmalig schreibbarer Erstellungsprovenienz (`createdVia`, `createdActor`, `createdAt`) versehen; bei der Übernahme eines vorhandenen Schlüssels wird sie nie neu gesetzt. Für menschliche Profilakteure wird `createdActor.label` bei der Projektion der Zeile aus dem aktuellen Benutzerprofil aufgelöst und nie im Sitzungseintrag gespeichert, sodass Profilumbenennungen keine Abweichungen verursachen. Sitzungszeilen enthalten außerdem `parentSessionKey` (Navigations-Elternelement, persistiert), `controlOwnerSessionKey` (Runtime-Controller, wenn aktiv), `forkSource` (exakter Quellschlüssel + Transkriptgeneration für Forks) und `previousSessionId` (vorherige Transkriptgeneration unter demselben Schlüssel).
    - `sessions.dispatch` (`operator.admin`) verschiebt eine vorhandene lokale OpenClaw-Sitzung mit einem sitzungseigenen verwalteten Worktree in ein konfiguriertes Cloud-Worker-Profil. Übergeben Sie `{ key, profileId, agentId? }`. Die Methode ist nicht vorhanden, wenn kein Worker-Profil konfiguriert ist, schließt die lokale Zulassung von Turns, bevor aktive Arbeit abgearbeitet wird, und kehrt erst zurück, nachdem die Platzierung die Worker-Eigentümerschaft `active` erreicht hat. Der Versand erfolgt nur in eine Richtung; das Zurückholen vom Worker auf das lokale System ist nicht Teil dieses RPC.
    - `sessions.groups.list`, `sessions.groups.put`, `sessions.groups.rename` und `sessions.groups.delete` verwalten den Gateway-eigenen Katalog benutzerdefinierter Sitzungsgruppen (Namen + Anzeigereihenfolge). Die Mitgliedschaft verbleibt im Feld `category` jeder Sitzung; beim Umbenennen und Löschen werden die Mitgliedssitzungen serverseitig aktualisiert.
    - `sessions.send` sendet eine Nachricht an eine vorhandene Sitzung.
    - `sessions.steer` ist die Variante zum Unterbrechen und Steuern einer aktiven Sitzung.
    - `sessions.abort` bricht aktive Arbeit für eine Sitzung ab. Übergeben Sie `key` sowie optional `runId` oder nur `runId` für aktive Ausführungen, die das Gateway einer Sitzung zuordnen kann. Durch Angabe von `runId` bleibt der Abbruch auf diese Ausführung beschränkt. Setzen Sie `clearQueued: true` bei einer nicht globalen Anfrage, die nur einen Schlüssel enthält, um außerdem die dieser Sitzung zugeordneten Folge- und Lane-Warteschlangen zu verwerfen. Bei vorhandenen Aufrufern, die `clearQueued` weglassen, bleiben diese Warteschlangen erhalten. Der literale Schlüssel `global` behält die bestehenden agentenqualifizierten Eigentumsregeln für `chat.abort` bei und führt keine nicht globale Bereinigung von Folge- oder Lane-Warteschlangen durch.
    - `sessions.patch` aktualisiert Sitzungsmetadaten/-überschreibungen und meldet das aufgelöste kanonische Modell sowie den effektiven Wert `agentRuntime`. Die Spawn-Abstammung (`spawnedBy`, `spawnedWorkspaceDir`, `spawnedCwd`, `spawnDepth`, `subagentRole`, `subagentControlScope`) kann nicht mehr öffentlich gepatcht werden; diese Fakten werden von vertrauenswürdigen Erstellungspfaden einmalig geschrieben, und Anfragen, die sie weiterhin senden, werden abgelehnt.
    - `sessions.reset`, `sessions.delete` und `sessions.compact` führen Sitzungswartungsaufgaben aus.
    - `sessions.get` gibt die vollständig gespeicherte Sitzungszeile zurück.
    - Die Chat-Ausführung verwendet weiterhin `chat.history`, `chat.send`, `chat.abort` und `chat.inject`. `chat.history` wird für UI-Clients an die Anzeige angepasst: Inline-Direktiven-Tags werden aus sichtbarem Text entfernt, Nur-Text-XML-Nutzlasten von Tool-Aufrufen (`<tool_call>...</tool_call>`, `<function_call>...</function_call>`, `<tool_calls>...</tool_calls>`, `<function_calls>...</function_calls>` und abgeschnittene Tool-Aufrufblöcke) sowie durchgesickerte Modellsteuerungstoken in ASCII oder voller Breite werden entfernt, reine Assistant-Zeilen mit Silent-Token (exakt `NO_REPLY` / `no_reply`) werden ausgelassen, und übergroße Zeilen können durch Platzhalter ersetzt werden.
    - `chat.message.get` ist der additive, begrenzte Leser vollständiger Nachrichten für einen einzelnen sichtbaren Transkripteintrag. Übergeben Sie `sessionKey`, optional `agentId`, wenn die Sitzungsauswahl agentenbezogen ist, und einen Transkriptwert `messageId`, der zuvor über `chat.history` bereitgestellt wurde; das Gateway gibt dieselbe an die Anzeige angepasste Projektion ohne die Begrenzung zur Kürzung des leichtgewichtigen Verlaufs zurück, sofern der gespeicherte Eintrag noch verfügbar und nicht übergroß ist.
    - `chat.toolTitles` gibt kurze Zweckbezeichnungen für Tool-Aufrufe zurück, die in der Control UI dargestellt werden (gebündelt, maximal 24 Elemente mit begrenzten Eingaben). Die Funktion wird über `gateway.controlUi.toolTitles` aktiviert (standardmäßig deaktiviert); deaktivierte Gateways beantworten `{ titles: {}, disabled: true }` ohne Modellaufruf, damit Clients keine weiteren Anfragen stellen. Wenn die Funktion aktiviert ist, verwenden die Bezeichnungen das standardmäßige Routing für Utility-Modelle: entweder ein explizit konfiguriertes `utilityModel` (eine Betreiberentscheidung, die wie alle Utility-Aufgaben begrenzte Aufgabeninhalte an den ausgewählten Provider senden kann) oder andernfalls den deklarierten Standard des Sitzungs-Providers für kleine Modelle, sodass nicht implizit ein neues Egress-Ziel entsteht; ein leerer Wert `utilityModel` deaktiviert sie vollständig. Die Bezeichnungen greifen nie auf das primäre Modell zurück. Ergebnisse werden in der agentenbezogenen Zustandsdatenbank anhand von Tool-Name + Eingabe zwischengespeichert, sodass wiederholte Ansichten dieselben Aufrufe nie erneut abrechnen.
    - `chat.send` akzeptiert den für einen Turn geltenden Wert `fastMode: "auto"`, um den Schnellmodus für Modellaufrufe zu verwenden, die vor dem automatischen Grenzwert gestartet wurden, und spätere Wiederholungs-, Fallback-, Tool-Ergebnis- oder Fortsetzungsaufrufe anschließend ohne Schnellmodus zu starten. Der Grenzwert beträgt standardmäßig 60 Sekunden (`DEFAULT_FAST_MODE_AUTO_ON_SECONDS`) und kann mit `agents.defaults.models["<provider>/<model>"].params.fastAutoOnSeconds` pro Modell konfiguriert werden. Ein Aufrufer von `chat.send` kann den für einen Turn geltenden Wert `fastAutoOnSeconds` übergeben, um den Grenzwert für diese Anfrage zu überschreiben. Übergeben Sie `queueMode` (`steer`, `followup`, `collect` oder `interrupt`), um den gespeicherten Warteschlangenmodus nur für diese Anfrage zu überschreiben; explizite Steuerungsaktionen der Control UI verwenden `queueMode: "steer"`. Interaktive Clients können `expectedLeafEntryId` mit dem aktiven Blatt des angezeigten Transkript-Branches oder `null` für ein maßgebliches leeres Transkript übergeben; das Gateway lehnt das Senden mit `details.reason: "active-leaf-changed"` ab, wenn zuvor ein anderer Client den Branch gewechselt hat.

  </Accordion>

  <Accordion title="Gerätekopplung und Gerätetoken">
    - `device.pair.list` gibt ausstehende und genehmigte gekoppelte Geräte zurück.
    - `device.pair.setupCode` erstellt einen mobilen Einrichtungscode und standardmäßig eine PNG-QR-Daten-URL. Dies erfordert `operator.admin` und wird bewusst nicht in der angekündigten Discovery aufgeführt. Das Ergebnis enthält `setupCode`, optional `qrDataUrl`, `gatewayUrl`, die nicht geheime Bezeichnung `auth` und `urlSource`.
    - `device.pair.approve`, `device.pair.reject` und `device.pair.remove` verwalten Datensätze zur Gerätekopplung.
    - `device.pair.rename` weist eine Betreiberbezeichnung (`{ deviceId, label }`) zu, die gegenüber dem vom Client gemeldeten Anzeigenamen bevorzugt wird und eine Gerätereparatur oder erneute Genehmigung überdauert.
    - `device.token.rotate` rotiert ein Token eines gekoppelten Geräts innerhalb der Grenzen seiner genehmigten Rolle und des Aufruferbereichs.
    - `device.token.revoke` widerruft ein Token eines gekoppelten Geräts innerhalb der Grenzen seiner genehmigten Rolle und des Aufruferbereichs.

    Der Einrichtungscode enthält kurzzeitig gültige Bootstrap-Zugangsdaten. Clients dürfen
    diese nicht protokollieren oder über den Kopplungsablauf hinaus persistieren.

  </Accordion>

  <Accordion title="Node-Kopplung, Aufruf und ausstehende Arbeit">
    - `node.pair.list`, `node.pair.approve`, `node.pair.reject` und `node.pair.remove` decken Genehmigungen für Node-Fähigkeiten ab. `node.pair.request` und `node.pair.verify` wurden 2026.7 zusammen mit dem eigenständigen Speicher für Node-Kopplungen entfernt; ausstehende Anfragen werden vom Gateway während der Verbindung von Nodes erstellt.
    - `node.list` und `node.describe` geben den Zustand bekannter/verbundener Nodes zurück.
    - `node.rename` aktualisiert die Bezeichnung eines gekoppelten Nodes.
    - `node.invoke` leitet einen Befehl an einen verbundenen Node weiter.
    - `node.invoke.result` gibt das Ergebnis einer Aufrufanfrage zurück.
    - `mcp.tools.call.v1` ist der Headless-Node-Host-Befehl zum Aufrufen eines konfigurierten Node-lokalen MCP-Tools. Er wird über `node.invoke` übertragen, setzt voraus, dass der Node den Befehl deklariert, und unterliegt weiterhin der Kopplungsgenehmigung sowie `gateway.nodes.commands.deny`.
    - `node.event` überträgt von Nodes stammende Ereignisse zurück an das Gateway.
    - `node.pluginTools.update` ist der einzige Veröffentlichungspfad zum Ersetzen der für den Agent sichtbaren Plugin-/MCP-Tool-Deskriptoren des verbundenen Nodes; die Parameter von `connect` übertragen diese nicht.
    - `node.pending.pull` und `node.pending.ack` sind die Warteschlangen-APIs für verbundene Nodes.
    - `node.pending.enqueue` und `node.pending.drain` verwalten dauerhafte ausstehende Arbeit für offline befindliche/getrennte Nodes.

  </Accordion>

  <Accordion title="Genehmigungsfamilien">
    - `approval.history` gibt die neuesten zuerst aufgeführten, 30 Tage lang aufbewahrten abschließenden Genehmigungen für Exec-, Plugin- und System-Agent-Anfragen zurück (Geltungsbereich `operator.approvals`). Es unterstützt Cursor-Paginierung sowie einen optionalen Artfilter; ausstehende Genehmigungen sind keine Verlaufseinträge.
    - `approval.get` und `approval.resolve` sind die artunabhängigen, dauerhaft gespeicherten Genehmigungsmethoden (Geltungsbereich `operator.approvals`). `approval.get` gibt eine bereinigte Projektion einer ausstehenden oder aufbewahrten abschließenden Genehmigung mit einer stabilen `urlPath` zurück; `approval.resolve` akzeptiert die kanonische Genehmigungs-ID, eine explizite `kind` und eine Entscheidung, wendet die Auflösung nach dem Prinzip „erste Antwort gewinnt“ an und gibt stets das aufgezeichnete kanonische Ergebnis zurück.
    - `exec.approval.request`, `exec.approval.get`, `exec.approval.list` und `exec.approval.resolve` decken einmalige Exec-Genehmigungsanfragen sowie das Nachschlagen und erneute Abspielen ausstehender Genehmigungen ab. Sie sind Adapter an der Protokollgrenze über derselben dauerhaft gespeicherten Genehmigungsregistrierung.
    - `exec.approval.waitDecision` wartet auf eine ausstehende Exec-Genehmigung und gibt die endgültige Entscheidung zurück (oder `null` bei Zeitüberschreitung).
    - `exec.approvals.get` und `exec.approvals.set` verwalten Snapshots der Gateway-Richtlinie für Exec-Genehmigungen.
    - `exec.approvals.node.get` und `exec.approvals.node.set` verwalten die Node-lokale Richtlinie für Exec-Genehmigungen über Node-Relay-Befehle.
    - `plugin.approval.request`, `plugin.approval.list`, `plugin.approval.waitDecision` und `plugin.approval.resolve` decken Plugin-definierte Genehmigungsabläufe ab.

  </Accordion>

  <Accordion title="Control-UI-Befehle">
    - `ui.command` ermöglicht es einem `operator.write`-Aufrufer, typisierte Layout- und Navigationsbefehle an verbundene Control-UI-Clients zu senden, welche die Fähigkeit `ui-commands` bekannt geben.
    - Die Befehle decken das Teilen, Schließen und Fokussieren von Bereichen, die Sichtbarkeit der Seitenleiste, die Sichtbarkeit und Andockposition des Terminal-/Browser-Bereichs sowie die Sitzungsnavigation ab.
    - Protokoll v1 verteilt die Befehle absichtlich an jede verbundene, dazu fähige Control UI. Ist keine verbunden, schlägt die Anfrage mit `UNAVAILABLE` fehl, statt vorzugeben, das Layout habe sich geändert.

  </Accordion>

  <Accordion title="Automatisierung, Skills und Werkzeuge">
    - Automatisierung: `wake` plant die sofortige oder beim nächsten Heartbeat erfolgende Einspeisung eines Aktivierungstexts; `cron.get`, `cron.list`, `cron.status`, `cron.add`, `cron.update`, `cron.remove`, `cron.run`, `cron.runs` verwalten geplante Aufgaben.
    - `cron.run` bleibt ein RPC im Einreihungsstil für manuelle Ausführungen. Clients, die Abschlusssemantik benötigen, sollten die zurückgegebene `runId` lesen und `cron.runs` abfragen.
    - `cron.runs` akzeptiert einen optionalen, nicht leeren `runId`-Filter, damit Clients eine einzelne eingereihte manuelle Ausführung verfolgen können, ohne mit anderen Verlaufseinträgen für dieselbe Aufgabe in Konflikt zu geraten.
    - Skills und Werkzeuge: `commands.list`, `skills.*`, `tools.catalog`, `tools.effective`, `tools.invoke`. Siehe unten [Hilfsmethoden für Operatoren](#operator-helper-methods).

  </Accordion>
</AccordionGroup>

### Allgemeine Ereignisfamilien

- `chat`: UI-Chat-Aktualisierungen wie `chat.inject` und andere ausschließlich das Transkript betreffende Chat-
  Ereignisse. In Protokoll v4 enthalten Delta-Nutzlasten `deltaText`; `message` bleibt
  der kumulative Assistenten-Snapshot. Ersetzungen, die keine Präfixe sind, setzen
  `replace=true` und verwenden `deltaText` als Ersetzungstext.
- `session.message`, `session.operation`, `session.tool`: Aktualisierungen des Transkripts, laufender
  Sitzungsvorgänge und des Ereignisstroms für eine abonnierte Sitzung.
- `session.approval`: bereinigter Wahrheitsstand zu ausstehenden und abschließenden Genehmigungen für einen
  ausdrücklich angemeldeten Abonnenten einer exakten Sitzung. Untergeordnete Genehmigungen verwenden die
  dauerhaft gespeicherte Zielgruppe des Vorfahren; Ereignisse verändern niemals Transkripte und aktivieren keine Agenten.
- `sessions.changed`: Sitzungsindex oder Metadaten wurden geändert.
- `presence`: Aktualisierungen des Systempräsenz-Snapshots.
- `tick`: periodisches Keepalive-/Erreichbarkeitsereignis.
- `health`: Aktualisierung des Gateway-Zustands-Snapshots.
- `heartbeat`: Aktualisierung des Heartbeat-Ereignisstroms.
- `cron`: Ereignis einer Änderung an einer Cron-Ausführung oder -Aufgabe.
- `shutdown`: Benachrichtigung über das Herunterfahren des Gateways.
- `node.pair.requested` / `node.pair.resolved`: Lebenszyklus der Node-Kopplung.
- `node.invoke.request`: Übertragung einer Node-Aufrufanfrage.
- `device.pair.requested` / `device.pair.resolved`: Lebenszyklus gekoppelter Geräte.
- `voicewake.changed`: Konfiguration des Aktivierungswort-Auslösers wurde geändert.
- `config.changed`: eine Konfigurationsänderung wurde dauerhaft gespeichert (die Nutzlast enthält den Konfigurationspfad,
  den neuen Snapshot-Hash und einen Zeitstempel – niemals Konfigurationsinhalte). Auf
  Operator-Lesezugriff beschränkt; Clients aktualisieren über `config.get`.
- `exec.approval.requested` / `exec.approval.resolved`: Lebenszyklus der Exec-
  Genehmigung.
- `plugin.approval.requested` / `plugin.approval.resolved`: Lebenszyklus der Plugin-
  Genehmigung.

### Node-Hilfsmethoden

Nodes können `skills.bins` aufrufen, um die aktuelle Liste ausführbarer Skill-Dateien
für Prüfungen der automatischen Zulassung abzurufen.

## RPC für das Audit-Ledger

`audit.activity.list` bietet Operator-Clients eine stabile, nach neuesten Einträgen zuerst sortierte Ansicht der Metadaten zum Lebenszyklus von Agenten-
ausführungen, Werkzeugaktionen und optional erfassten Nachrichten. Es erfordert
`operator.read`. Abfragen schließen Datensätze aus, die älter als 30 Tage sind, und das gemeinsam genutzte
SQLite-Ledger ist auf 100,000 Datensätze begrenzt. Abgelaufene Zeilen werden beim
Start des Gateways, bei der stündlichen Wartung und bei späteren Schreibvorgängen gelöscht. Siehe
[Audit-Verlauf](/de/gateway/audit) für das Datenmodell und die Datenschutzsemantik.

- Parameter: optionale exakte `agentId`, `sessionKey` oder `runId`; optionale `kind`
  (`"agent_run"`, `"tool_action"` oder `"message"`); optionale `status`
  (`"started"`, `"succeeded"`, `"failed"`, `"cancelled"`, `"timed_out"`,
  `"blocked"` oder `"unknown"`); optionale Nachrichten-`direction` (`"inbound"` oder
  `"outbound"`) und exakte `channel`; optionale inklusive Unix-Millisekunden-Grenzen `after` / `before`;
  optionale `limit` von `1` bis `500`; und optionale
  Zeichenfolge `cursor` von der vorherigen Seite.
- Ergebnis: `{ "events": AuditActivityEventV1[], "nextCursor"?: string }`.

Die benannte V1-Ergebnis-Union besitzt separate Schemas für Agentenausführungen, Werkzeugaktionen, eingehende Nachrichten
und ausgehende Nachrichten. Der Diskriminator `eventType` lautet jeweils
`agent_run`, `tool_action`, `inbound_message` oder `outbound_message`; `kind` und die
Nachrichten-`direction` bleiben für Filterung und Anzeige verfügbar. Jedes Ereignis besitzt eine
ganzzahlige `schemaVersion: 1`. Referenzen auf Nachrichtenidentitäten verwenden das exakte
Format `hmac-sha256:v1:<32 hex key id>:<64 hex digest>`; die Akteur-ID eines Kanalabsenders
verwendet dasselbe Format.

Alle Varianten erfordern `eventType`, `schemaVersion`, `eventId`, `sequence`,
`sourceSequence`, `occurredAt`, `kind`, `action`, `status`, `actor` und
`redaction`. Die Variantenfelder sind:

| `eventType`        | Erforderliche Felder                                               | Optionale Felder                                                                                                                 |
| ------------------ | ----------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `agent_run`        | `agentId`, `runId`; `kind: "agent_run"`                           | `sessionKey`, `sessionId`, `errorCode`                                                                                          |
| `tool_action`      | `agentId`, `runId`; `kind: "tool_action"`                         | `sessionKey`, `sessionId`, `toolCallId`, `toolName`, `errorCode`                                                                |
| `inbound_message`  | `direction: "inbound"`, `channel`, `conversationKind`, `outcome`  | `agentId`, `runId`, `durationMs`, `resultCount`, Identitätsreferenzen, `reasonCode`, `errorCode`                                 |
| `outbound_message` | `direction: "outbound"`, `channel`, `conversationKind`, `outcome` | `agentId`, `runId`, `durationMs`, `resultCount`, Identitätsreferenzen, `reasonCode`, `deliveryKind`, `failureStage`, `errorCode` |

Die geschlossenen Nachrichten-Enums sind:

- `conversationKind`: `direct`, `group`, `channel` oder `unknown`.
- Eingehende `outcome`: `completed`, `skipped` oder `failed`; optionale
  `reasonCode`: `duplicate`, `reply_operation_active`,
  `reply_operation_aborted`, `fast_abort`, `plugin_bound_handled`,
  `plugin_bound_unavailable`, `plugin_bound_declined`, `plugin_bound_error`,
  `before_dispatch_handled`, `acp_dispatch_completed`, `acp_dispatch_failed`,
  `acp_dispatch_empty` oder `acp_dispatch_aborted`.
- Ausgehende `outcome`: `sent`, `suppressed`, `failed` oder `unknown`; optionale
  `reasonCode`: `cancelled_by_message_sending_hook`,
  `cancelled_by_reply_payload_sending_hook`,
  `empty_after_message_sending_hook`, `empty_after_reply_payload_sending_hook`
  oder `no_visible_payload`. Ein Adapter, der keine Plattformidentität zurückgibt, ist
  `unknown`, da die externe Nebenwirkung nicht widerlegt werden kann.
- `deliveryKind`: `text`, `media` oder `other`; `failureStage`:
  `platform_send`, `queue` oder `unknown`.

Abschlussfelder sind miteinander korreliert und nicht unabhängig optional:

| Variante          | Abschlusszuordnung                                                                                                                                                   |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Agentenausführung        | `started` besitzt keine `errorCode`; jeder abgeschlossene Status ohne Erfolg erfordert seinen entsprechenden `run_*`-Code.                                                                 |
| Werkzeugaktion      | `started` und „erfolgreich“ besitzen keine `errorCode`; jeder andere abgeschlossene Status erfordert seinen entsprechenden `tool_*`-Code.                                                       |
| Eingehende Nachricht  | erfolgreich = `completed`; blockiert = `skipped`; fehlgeschlagen = `failed` plus `message_processing_failed`. `reasonCode` muss, sofern vorhanden, zu dieser Abschlussfamilie gehören. |
| Ausgehende Nachricht | erfolgreich = `sent`; blockiert = `suppressed` plus `reasonCode`; fehlgeschlagen = `failed` plus `errorCode` und `failureStage`; unbekannt = `unknown` plus `failureStage`.      |

Jedes Aktivitätsereignis enthält eine stabile Ereignis-ID, eine monotone Ledger-Sequenz,
eine Quellereignissequenz, einen Zeitstempel, einen Akteur, eine Aktion, einen Status, die ganzzahlige
`schemaVersion: 1` und `redaction: "metadata_only"`. Ausführungs- und Werkzeugdatensätze
erfordern die Herkunft des Agenten und der Ausführung und können die Herkunft der Sitzung enthalten. Nachrichten-
datensätze können Agenten- und Ausführungs-IDs enthalten, schließen jedoch absichtlich stets
`sessionKey` und `sessionId` aus; der Abfragefilter `sessionKey` gilt daher
nur für Ausführungs- und Werkzeugzeilen. Werkzeugereignisse können die Werkzeugaufruf-ID und den Werkzeugnamen enthalten.

Nachrichtendatensätze verwenden `message.inbound.processed` oder
`message.outbound.finished` und ergänzen Richtung, Kanal, Konversationsart,
normalisiertes Ergebnis sowie optional Zustellungsart, Fehlerphase, Dauer,
Ergebnisanzahl, Ursachencode und installationslokale, schlüsselbasierte
Konto-/Konversations-/Nachrichten-/Zielpseudonyme. Diese Pseudonyme erleichtern
die Korrelation, stellen jedoch keine Anonymisierung dar: Die Zustandsdatenbank enthält ihren Schlüssel,
RPC- und CLI-Exporte hingegen nicht. Das Ledger speichert keine Prompts, Nachrichteninhalte,
Toolargumente, Toolergebnisse, Befehlsausgaben oder unformatierten Fehlertexte.
Run-/Tool-`sessionKey`-Werte bleiben unverarbeitete Korrelationsmetadaten und können
Plattformkonto- oder Peer-IDs enthalten; Nachrichtendatensätze lassen Sitzungsschlüssel aus.

Bei eingehenden Zeilen misst `durationMs` den Core-Dispatch bis zu seinem Abschluss, und
`resultCount` zählt finalisierte, in die Warteschlange eingereihte Tool-, Block- und Antwort-Payloads. Bei
ausgehenden Zeilen umfasst `durationMs` die Zustellungsverantwortung bis zur Bestätigung,
Dead-Letter-Behandlung oder Abstimmung (einschließlich Wartezeit in der Warteschlange), und `resultCount`
zählt identifizierte physische Sendungen an die Plattform. `deliveryKind` beschreibt, sofern vorhanden,
den effektiven Payload nach Hooks und Rendering; unterdrückte oder
bei Abstürzen mehrdeutige Zeilen lassen diesen Wert aus.

Die aktuelle Nachrichtenabdeckung umfasst akzeptierte eingehende Nachrichten, die den Core-
Dispatch erreichen, einschließlich Core-Ergebnissen für Duplikate und Abschlüsse. Für ausgehende Nachrichten wird
eine Abschlusszeile pro ursprünglichem logischen Antwort-Payload geschrieben, der die gemeinsame dauerhafte
Zustellung erreicht; Chunking und Adapter-Fan-out werden in `resultCount` zusammengefasst. In die Warteschlange eingereihte,
wiederholbare oder mehrdeutige Sendungen werden erst nach Bestätigung, Dead-Letter-Behandlung
oder Abstimmung aufgezeichnet. Plugin-lokale und direkte Sendepfade, die diese
gemeinsamen Grenzen umgehen, sind noch nicht abgedeckt. Die begrenzte Worker-Warteschlange arbeitet nach bestem Bemühen
und kann bei Fehlern oder Überlastung Datensätze verwerfen; daher ist diese Oberfläche kein
verlustfreies Compliance-Archiv.

Die Aufzeichnung ist standardmäßig aktiviert und wird über
[`audit.enabled`](/de/gateway/configuration-reference#audit) gesteuert. Die Nachrichtenaufzeichnung wird
separat durch `audit.messages` gesteuert und verwendet standardmäßig `"off"`. Wenn
die Aufzeichnung deaktiviert ist, stellt `audit.activity.list` zuvor geschriebene Datensätze
weiterhin bereit, bis sie ablaufen.

Die ausgelieferten Schemas für `audit.list`-Anfragen, -Ergebnisse und `AuditEvent` bleiben
unverändert und geben ausschließlich Agentenlauf- und Toolaktionsdatensätze zurück. Neue Operator-
Clients sollten `audit.activity.list` aufrufen, wenn der Gateway dies ankündigt. Ältere
Gateways melden bei einer auf Lesezugriff beschränkten Anfrage möglicherweise entweder `unknown method: audit.activity.list` oder, da
die Autorisierung in ausgelieferten Versionen vor der Methodensuche erfolgte, `missing scope:
operator.admin`. Behandeln Sie Letzteres
nur dann als fehlende Methode, wenn die Methode nicht angekündigt wurde. Ein Client darf anschließend `audit.list`
nur dann erneut versuchen, wenn seine Filter keine Unterstützung für Nachrichtenart, Richtung oder Kanal
erfordern.

Verwenden Sie [`openclaw audit`](/de/cli/audit) für Textabfragen und begrenzte JSON-Exporte.

## Task-Ledger-RPCs

Operator-Clients prüfen und stornieren Datensätze von Gateway-Hintergrundaufgaben über
die Task-Ledger-RPCs (`packages/gateway-protocol/src/schema/tasks.ts`). Diese
geben bereinigte Aufgabenzusammenfassungen zurück, keinen unverarbeiteten Laufzeitstatus.

- `tasks.list` erfordert `operator.read`.
  - Parameter: optional `status` (`"queued"`, `"running"`, `"completed"`,
    `"failed"`, `"cancelled"` oder `"timed_out"`) oder ein Array dieser Statuswerte,
    optional `agentId`, optional `sessionKey`, optional `limit` von `1` bis
    `500` und optional die Zeichenfolge `cursor`.
  - Ergebnis: `{ "tasks": TaskSummary[], "nextCursor"?: string }`.
- `tasks.get` erfordert `operator.read`.
  - Parameter: `{ "taskId": string }`.
  - Ergebnis: `{ "task": TaskSummary }`.
  - Fehlende Aufgaben-IDs geben die Gateway-Fehlerstruktur für „nicht gefunden“ zurück.
- `tasks.cancel` erfordert `operator.write`.
  - Parameter: `{ "taskId": string, "reason"?: string }`.
  - Ergebnis: `{ "found": boolean, "cancelled": boolean, "reason"?: string, "task"?: TaskSummary }`.
  - `found` gibt an, ob das Ledger eine passende Aufgabe enthielt. `cancelled`
    gibt an, ob die Laufzeit die Stornierung akzeptiert oder aufgezeichnet hat.

`TaskSummary` enthält `id`, `status` und optionale Metadaten: `kind`,
`runtime`, `title`, `agentId`, `sessionKey`, `childSessionKey`, `ownerKey`,
`runId`, `taskId`, `flowId`, `parentTaskId`, `sourceId`, Zeitstempel, Fortschritt,
Abschlusszusammenfassung und bereinigten Fehlertext. `agentId` identifiziert den Agenten,
der die Aufgabe ausführt; `sessionKey` und `ownerKey` bewahren den Kontext des Anfragenden und der Steuerung.

## Hilfsmethoden für Operatoren

- `commands.list` (`operator.read`) ruft das Laufzeit-Befehlsinventar für
  einen Agenten ab.
  - `agentId` ist optional; lassen Sie es aus, um den Standardarbeitsbereich des Agenten zu lesen.
  - `scope` steuert, auf welche Oberfläche das primäre `name` abzielt: `text` gibt
    das primäre Textbefehlstoken ohne das vorangestellte `/` zurück; `native` und der
    standardmäßige `both`-Pfad geben, sofern verfügbar, Provider-spezifische native Namen zurück.
  - `textAliases` enthält exakte Slash-Aliase wie `/model` und `/m`.
  - `nativeName` enthält den Provider-spezifischen nativen Befehlsnamen, sofern
    einer vorhanden ist.
  - `provider` ist optional und wirkt sich nur auf die native Benennung sowie die Verfügbarkeit nativer Plugin-
    Befehle aus.
  - `includeArgs=false` lässt serialisierte Argumentmetadaten in der Antwort aus.
- `tools.catalog` (`operator.read`) ruft den Laufzeit-Toolkatalog für einen
  Agenten ab. Die Antwort enthält gruppierte Tools und Herkunftsmetadaten:
  - `source`: `core` oder `plugin`
  - `pluginId`: Plugin-Eigentümer, wenn `source="plugin"`
  - `optional`: ob ein Plugin-Tool optional ist
- `tools.effective` (`operator.read`) ruft das laufzeiteffektive Tool-
  Inventar für eine Sitzung ab.
  - `sessionKey` ist erforderlich.
  - Der Gateway leitet den vertrauenswürdigen Laufzeitkontext serverseitig aus der Sitzung ab,
    statt vom Aufrufer bereitgestellten Authentifizierungs- oder Zustellungskontext zu akzeptieren.
  - Die Antwort ist eine sitzungsbezogene, serverseitig abgeleitete Projektion des aktiven
    Inventars, einschließlich Core-, Plugin-, Kanal- und bereits erkannter MCP-
    Server-Tools.
  - `tools.effective` ist für MCP schreibgeschützt: Es kann einen aufgewärmten MCP-Katalog der Sitzung
    durch die endgültige Toolrichtlinie projizieren, erstellt jedoch keine MCP-Laufzeiten,
    verbindet keine Transporte und gibt kein `tools/list` aus. Wenn kein passender aufgewärmter Katalog
    vorhanden ist, kann die Antwort einen Hinweis wie `mcp-not-yet-connected`,
    `mcp-not-yet-listed` oder `mcp-stale-catalog` enthalten.
  - Effektive Tooleinträge verwenden `source="core"`, `source="plugin"`,
    `source="channel"` oder `source="mcp"`.
- `tools.invoke` (`operator.write`) ruft ein verfügbares Tool über denselben
  Gateway-Richtlinienpfad wie `/tools/invoke` auf.
  - `name` ist erforderlich. `args`, `sessionKey`, `agentId`, `confirm` und
    `idempotencyKey` sind optional.
  - Wenn sowohl `sessionKey` als auch `agentId` vorhanden sind, muss der aufgelöste Sitzungsagent
    mit `agentId` übereinstimmen.
  - Nur für Eigentümer bestimmte Core-Wrapper wie `cron`, `gateway` und `nodes` erfordern
    eine Eigentümer-/Administratoridentität (`operator.admin`), obwohl `tools.invoke` selbst
    `operator.write` ist.
  - Die Antwort ist ein SDK-seitiger Umschlag mit `ok`, `toolName`, optional
    `output` und typisierten `error`-Feldern. Ablehnungen aufgrund von Genehmigungen oder Richtlinien geben
    `ok:false` im Payload zurück, anstatt die Gateway-Toolrichtlinien-
    Pipeline zu umgehen.
- `skills.status` (`operator.read`) ruft das sichtbare Skills-Inventar für einen
  Agenten ab.
  - `agentId` ist optional; lassen Sie es aus, um den Standardarbeitsbereich des Agenten zu lesen.
  - Die Antwort enthält Eignung, fehlende Anforderungen, Konfigurationsprüfungen
    und bereinigte Installationsoptionen, ohne unverarbeitete Geheimniswerte offenzulegen.
- `skills.search` und `skills.detail` (`operator.read`) geben ClawHub-
  Erkennungsmetadaten zurück.
- `skills.upload.begin`, `skills.upload.chunk` und `skills.upload.commit`
  (`operator.admin`) stellen ein privates Skills-Archiv bereit, bevor es installiert wird. Dies
  ist ein separater Administrator-Uploadpfad für vertrauenswürdige Clients, nicht der normale ClawHub-
  Installationsablauf für Skills, und ist standardmäßig deaktiviert, sofern
  `skills.install.allowUploadedArchives` nicht aktiviert ist.
  - `skills.upload.begin({ kind: "skill-archive", slug, sizeBytes, sha256?, force?, idempotencyKey? })`
    erstellt einen Upload, der an diesen Slug und Force-Wert gebunden ist.
  - `skills.upload.chunk({ uploadId, offset, dataBase64 })` hängt Bytes am
    exakten dekodierten Offset an.
  - `skills.upload.commit({ uploadId, sha256? })` überprüft die endgültige Größe und
    SHA-256. Der Commit schließt nur den Upload ab; er installiert den Skill nicht.
  - Hochgeladene Skills-Archive sind ZIP-Archive, die ein `SKILL.md`-Stammverzeichnis enthalten. Der
    interne Verzeichnisname des Archivs bestimmt niemals das Installationsziel.
- `skills.install` (`operator.admin`) hat drei Modi:
  - ClawHub-Modus: `{ source: "clawhub", slug, version?, force? }` installiert einen
    Skills-Ordner in das `skills/`-Verzeichnis des Standardarbeitsbereichs des Agenten.
  - Upload-Modus: `{ source: "upload", uploadId, slug, force?, sha256?, timeoutMs? }`
    installiert einen abgeschlossenen Upload in das `skills/<slug>`-Verzeichnis des
    Standardarbeitsbereichs des Agenten. Der Slug und der Force-Wert müssen mit der
    ursprünglichen `skills.upload.begin`-Anfrage übereinstimmen. Wird abgelehnt, sofern
    `skills.install.allowUploadedArchives` nicht aktiviert ist; die Einstellung wirkt sich nicht
    auf ClawHub-Installationen aus.
  - Gateway-Installationsmodus: `{ name, installId, timeoutMs? }` führt eine deklarierte
    `metadata.openclaw.install`-Aktion auf dem Gateway-Host aus. Ältere Clients können
    weiterhin `dangerouslyForceUnsafeInstall` senden; dieses Feld ist veraltet,
    wird nur aus Gründen der Protokollkompatibilität akzeptiert und ignoriert. Verwenden Sie
    `security.installPolicy` für vom Operator verantwortete Installationsentscheidungen.
- `skills.update` (`operator.admin`) hat zwei Modi:
  - Der ClawHub-Modus aktualisiert einen nachverfolgten Slug oder alle nachverfolgten ClawHub-Installationen im
    Standardarbeitsbereich des Agenten.
  - Der Konfigurationsmodus patcht `skills.entries.<skillKey>`-Werte wie `enabled`,
    `apiKey` und `env`.

### `models.list`-Ansichten

`models.list` akzeptiert einen optionalen `view`-Parameter
(`src/agents/model-catalog-visibility.ts`):

- Ausgelassen oder `"default"`: Wenn `agents.defaults.modelPolicy.allow` konfiguriert ist, besteht die
  Antwort aus dem zulässigen Katalog, einschließlich dynamisch erkannter Modelle
  für `provider/*`-Einträge. Andernfalls besteht die Antwort aus dem vollständigen Gateway-
  Katalog.
- `"configured"`: Verhalten mit für eine Auswahl geeigneter Größe. Wenn `agents.defaults.modelPolicy.allow`
  konfiguriert ist, hat es weiterhin Vorrang, einschließlich Provider-bezogener Erkennung für
  `provider/*`-Einträge. Ohne Zulassungsliste verwendet die Antwort explizite
  `models.providers.<provider>.models`-Einträge und greift nur dann auf den vollständigen
  Katalog zurück, wenn keine konfigurierten Modellzeilen vorhanden sind.
- `"provider-config"`: vom Quellautor erstelltes `models.providers.*.models`-Inventar,
  unabhängig von Auswahl-Zulassungslisten. Zeilen enthalten öffentliche Modellfähigkeiten und
  routenbezogene Verfügbarkeit, lassen jedoch Provider-Endpunkte, Authentifizierungsmaterial und
  Laufzeit-Anfragekonfiguration aus.
- `"all"`: vollständiger Gateway-Katalog unter Umgehung von `agents.defaults.modelPolicy.allow`. Für
  Diagnose-/Erkennungsoberflächen verwenden, nicht für normale Modellauswahlen.

## Ausführungsgenehmigungen

- Wenn eine exec-Anfrage eine Genehmigung benötigt, sendet das Gateway
  `exec.approval.requested`.
- Operator-Clients lösen sie durch Aufrufen von `exec.approval.resolve` auf (erfordert
  `operator.approvals`).
- Für `host=node` muss `exec.approval.request` `systemRunPlan` enthalten
  (kanonische `argv`/`cwd`/`rawCommand`/Sitzungsmetadaten). Anfragen ohne
  `systemRunPlan` werden abgelehnt.
- Nach der Genehmigung verwenden weitergeleitete `node.invoke system.run`-Aufrufe diesen
  kanonischen `systemRunPlan` als maßgeblichen Befehls-/cwd-/Sitzungskontext.
- Wenn ein Aufrufer `command`, `rawCommand`, `cwd`, `agentId` oder
  `sessionKey` zwischen der Vorbereitung und der endgültigen genehmigten `system.run`-Weiterleitung verändert,
  lehnt das Gateway die Ausführung ab, anstatt der veränderten Nutzlast zu vertrauen.

## Fallback bei der Agent-Zustellung

- `agent`-Anfragen können `deliver=true` enthalten, um eine ausgehende Zustellung anzufordern.
- `bestEffortDeliver=false` (der Standardwert) behält das strikte Verhalten bei: Nicht auflösbare oder
  ausschließlich interne Zustellungsziele geben `INVALID_REQUEST` zurück.
- `bestEffortDeliver=true` ermöglicht einen Fallback auf eine Ausführung ausschließlich in der Sitzung, wenn keine
  extern zustellbare Route aufgelöst werden kann (beispielsweise bei internen/Webchat-
  Sitzungen oder mehrdeutigen Mehrkanalkonfigurationen).
- Endgültige `agent`-Ergebnisse können `result.deliveryStatus` enthalten, wenn eine Zustellung
  angefordert wurde, wobei dieselben Status `sent`, `suppressed`, `partial_failed` und
  `failed` verwendet werden, die für
  [`openclaw agent --json --deliver`](/de/cli/agent#json-delivery-status) dokumentiert sind.

## Versionierung

- `PROTOCOL_VERSION`, `MIN_CLIENT_PROTOCOL_VERSION`,
  `MIN_NODE_PROTOCOL_VERSION` und `MIN_PROBE_PROTOCOL_VERSION` befinden sich in
  `packages/gateway-protocol/src/version.ts`.
- Clients senden `minProtocol` + `maxProtocol`. Operator- und UI-Clients müssen
  das aktuelle Protokoll in diesem Bereich enthalten; aktuelle Clients und Server verwenden
  Protokoll v4.
- Authentifizierte Clients mit sowohl `role: "node"` als auch `client.mode: "node"`
  können das N-1-Node-Protokoll verwenden (derzeit v3). Leichtgewichtige Neustartprüfungen verwenden
  dasselbe N-1-Fenster. Geräteauthentifizierung, Kopplung, Geltungsbereiche, Befehlsrichtlinien und exec-
  Genehmigungen bleiben von diesem Kompatibilitätsfenster unverändert. Plugin-eigene Node-
  Fähigkeiten und Befehle werden zurückgehalten, bis die Node auf das aktuelle
  Protokoll aktualisiert wurde, da ihre gehosteten Oberflächen nicht Teil des N-1-Vertrags sind.
- Schemas und Modelle werden aus TypeBox-Definitionen generiert:
  - `pnpm protocol:gen`
  - `pnpm protocol:gen:swift`
  - `pnpm protocol:check`

### Client-Konstanten

Die Referenzimplementierung des Clients befindet sich in `packages/gateway-client/src/`
(OpenClaw bindet sie über die schlanke `src/gateway/client.ts`-Fassade ein). Diese
Standardwerte sind über Protokoll v4 hinweg stabil und bilden die erwartete Ausgangsbasis für
Clients von Drittanbietern.

| Konstante                                 | Standardwert                                          | Quelle                                                                                                                    |
| ----------------------------------------- | ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `PROTOCOL_VERSION`                        | `4`                                                   | `packages/gateway-protocol/src/version.ts`                                                                                |
| `MIN_CLIENT_PROTOCOL_VERSION`             | `4`                                                   | `packages/gateway-protocol/src/version.ts`                                                                                |
| `MIN_NODE_PROTOCOL_VERSION`               | `3`                                                   | `packages/gateway-protocol/src/version.ts`                                                                                |
| `MIN_PROBE_PROTOCOL_VERSION`              | `3`                                                   | `packages/gateway-protocol/src/version.ts`                                                                                |
| Anfrage-Timeout (pro RPC)                 | `30_000` ms                                           | `packages/gateway-client/src/client.ts` (`requestTimeoutMs`)                                                              |
| Timeout für Preauth/Verbindungs-Challenge | `15_000` ms                                           | `packages/gateway-client/src/timeouts.ts` (die Umgebungsvariable `OPENCLAW_HANDSHAKE_TIMEOUT_MS` kann das gekoppelte Server-/Client-Budget erhöhen) |
| Anfänglicher Backoff für erneute Verbindung | `1_000` ms                                            | `packages/gateway-client/src/client.ts` (`GATEWAY_RECONNECT_POLICY`)                                                      |
| Maximaler Backoff für erneute Verbindung  | `30_000` ms                                           | `packages/gateway-client/src/client.ts` (`GATEWAY_RECONNECT_POLICY`)                                                      |
| Begrenzung schneller Wiederholungsversuche nach dem Schließen wegen eines Geräte-Tokens | `250` ms                                              | `packages/gateway-client/src/client.ts`                                                                                   |
| Karenzzeit für erzwungenes Beenden vor `terminate()` | `250` ms                                              | `FORCE_STOP_TERMINATE_GRACE_MS`                                                                                           |
| Standard-Timeout für `stopAndWait()`   | `1_000` ms                                            | `STOP_AND_WAIT_TIMEOUT_MS`                                                                                                |
| Standardmäßiges Tick-Intervall (vor `hello-ok`) | `30_000` ms                                           | `packages/gateway-client/src/client.ts`                                                                                   |
| Schließen bei Tick-Timeout                | Code `4000`, wenn die Stille `tickIntervalMs * 2` überschreitet | `packages/gateway-client/src/client.ts`                                                                                   |
| `MAX_PAYLOAD_BYTES`                       | `25 * 1024 * 1024` (25 MB)                            | `src/gateway/server-constants.ts`                                                                                         |

Der Server gibt die effektiven Werte für `policy.tickIntervalMs`,
`policy.maxPayload` und `policy.maxBufferedBytes` in `hello-ok` bekannt; Clients
sollten diese Werte anstelle der Standardwerte vor dem Handshake berücksichtigen.

Der Referenz-Client überlässt Anfragen mit endlicher Laufzeit ihre konfigurierte Frist, wenn
jede ausstehende Anfrage eine solche besitzt. Eine `expectFinal`-Anfrage ohne endlichen
`timeoutMs`, eine beliebige Anfrage mit `timeoutMs: null` oder eine Mischung aus zeitlich begrenzten und
unbegrenzten Anfragen lässt den Tick-Watchdog aktiv. Wenn eingehende Ereignisse und
Antworten über den Tick-Timeout-Schwellenwert hinaus ausbleiben, schließt der Client den
Socket mit Code `4000`, lehnt alle ausstehenden Anfragen ab und stellt die Verbindung erneut her. Er
führt abgelehnte Anfragen nach der erneuten Verbindung nicht erneut aus.

## Authentifizierung

- Die Gateway-Authentifizierung mit gemeinsamem Geheimnis verwendet je nach konfiguriertem
  `gateway.auth.mode` (`"none" | "token" | "password" | "trusted-proxy"`) entweder `connect.params.auth.token` oder
  `connect.params.auth.password`.
- Identitätstragende Modi wie Tailscale Serve (`gateway.auth.allowTailscale: true`)
  oder ein nicht an Loopback gebundenes `gateway.auth.mode: "trusted-proxy"` erfüllen die
  Authentifizierungsprüfung beim Verbindungsaufbau anhand der Anfrage-Header statt anhand von `connect.params.auth.*`.
- Bei privatem Ingress überspringt `gateway.auth.mode: "none"` die Authentifizierung beim Verbindungsaufbau
  mit gemeinsamem Geheimnis vollständig; stellen Sie diesen Modus nicht über öffentlichen/nicht vertrauenswürdigen Ingress bereit.
- Nach dem Pairing stellt das Gateway ein auf Verbindungsrolle und Geltungsbereiche
  beschränktes Geräte-Token aus, das in `hello-ok.auth.deviceToken` zurückgegeben wird. Clients sollten
  es nach jedem erfolgreichen Verbindungsaufbau dauerhaft speichern.
- Beim erneuten Verbindungsaufbau mit diesem gespeicherten Geräte-Token sollte auch der gespeicherte
  genehmigte Satz von Geltungsbereichen für dieses Token wiederverwendet werden. Dadurch bleiben bereits
  gewährte Lese-, Prüf- und Statuszugriffe erhalten, und erneute Verbindungen werden nicht unbemerkt auf einen engeren,
  impliziten, ausschließlich für Administratoren vorgesehenen Geltungsbereich reduziert.
- Clientseitige Zusammenstellung der Authentifizierung für den Verbindungsaufbau (`selectConnectAuth` in
  `packages/gateway-client/src/client.ts`):
  - `auth.password` ist unabhängig und wird immer weitergeleitet, wenn es gesetzt ist.
  - `auth.token` wird in folgender Prioritätsreihenfolge befüllt: zuerst ein explizites gemeinsames Token,
    dann ein explizites `deviceToken`, anschließend ein gespeichertes gerätespezifisches Token (indiziert nach
    `deviceId` + `role`).
  - `auth.bootstrapToken` wird nur gesendet, wenn keine der vorstehenden Optionen
    `auth.token` aufgelöst hat. Ein gemeinsames Token oder ein beliebiges aufgelöstes Geräte-Token unterdrückt es.
  - Die automatische Heraufstufung eines gespeicherten Geräte-Tokens beim einmaligen
    Wiederholungsversuch mit `AUTH_TOKEN_MISMATCH` ist ausschließlich für vertrauenswürdige Endpunkte zulässig: Loopback
    oder `wss://` mit einem angehefteten `tlsFingerprint`. Öffentliches `wss://` ohne Anheftung
    erfüllt die Voraussetzungen nicht.
- Der integrierte Bootstrap per Einrichtungscode gibt den primären Node
  `hello-ok.auth.deviceToken` sowie ein begrenztes Operator-Token in
  `hello-ok.auth.deviceTokens` für die vertrauenswürdige Übergabe an Mobilgeräte zurück. Das Operator-Token
  enthält `operator.talk.secrets` für native Lesezugriffe auf die Talk-Konfiguration, schließt jedoch
  Geltungsbereiche für Pairing-Änderungen und `operator.admin` aus.
- Während ein Einrichtungscode-Bootstrap außerhalb der Basiskonfiguration auf die Genehmigung wartet,
  enthalten die Details von `PAIRING_REQUIRED` die Felder `recommendedNextStep: "wait_then_retry"`,
  `retryable: true` und `pauseReconnect: false`. Stellen Sie die Verbindung mit demselben
  Bootstrap-Token wiederholt her, bis die Anfrage genehmigt wurde oder das Token
  ungültig wird.
- Speichern Sie `hello-ok.auth.deviceTokens` nur dauerhaft, wenn beim Verbindungsaufbau die Bootstrap-
  Authentifizierung über einen vertrauenswürdigen Transport wie `wss://` oder über lokales/Loopback-Pairing verwendet wurde.
- Wenn ein Client ein explizites `deviceToken` oder explizites `scopes` bereitstellt, bleibt dieser
  vom Aufrufer angeforderte Satz von Geltungsbereichen maßgeblich; zwischengespeicherte Geltungsbereiche werden nur
  wiederverwendet, wenn der Client das gespeicherte gerätespezifische Token erneut verwendet.
- Geräte-Token können über `device.token.rotate` und
  `device.token.revoke` rotiert/widerrufen werden (erfordert `operator.pairing`). Das Rotieren oder Widerrufen eines
  Node oder einer anderen Nicht-Operator-Rolle erfordert außerdem `operator.admin`.
- `device.token.rotate` gibt Rotationsmetadaten zurück. Das Ersatz-
  Bearer-Token wird nur bei Aufrufen desselben Geräts zurückgegeben, die bereits mit diesem
  Geräte-Token authentifiziert wurden, damit Clients, die ausschließlich Token verwenden, ihren Ersatz vor dem
  erneuten Verbindungsaufbau dauerhaft speichern können. Bei Rotationen über gemeinsame Token oder Administratorzugriff wird das Bearer-Token nicht zurückgegeben.
- Ausstellung, Rotation und Widerruf von Token bleiben auf den genehmigten Rollensatz
  beschränkt, der im Pairing-Eintrag dieses Geräts verzeichnet ist; Token-Änderungen können keine
  Geräterolle erweitern oder adressieren, die bei der Pairing-Genehmigung nie gewährt wurde.
- Bei Token-Sitzungen gekoppelter Geräte ist die Geräteverwaltung auf das eigene Gerät beschränkt, sofern
  der Aufrufer nicht zusätzlich über `operator.admin` verfügt: Aufrufer ohne Administratorrechte können nur das
  Operator-Token ihres eigenen Geräteeintrags verwalten. Die Verwaltung von Node- und anderen Nicht-Operator-Token
  ist ausschließlich Administratoren vorbehalten, selbst beim eigenen Gerät des Aufrufers.
- `device.token.rotate` und `device.token.revoke` prüfen außerdem den Satz von Geltungsbereichen des
  adressierten Operator-Tokens gegen die aktuellen Sitzungsgeltungsbereiche des Aufrufers.
  Aufrufer ohne Administratorrechte können kein Operator-Token rotieren oder widerrufen, dessen Geltungsbereich weiter gefasst ist als ihr
  eigener.
- Authentifizierungsfehler enthalten `error.details.code` sowie Hinweise zur Wiederherstellung:
  - `error.details.canRetryWithDeviceToken` (boolescher Wert)
  - `error.details.recommendedNextStep`: einer der Werte `retry_with_device_token`,
    `update_auth_configuration`, `update_auth_credentials`,
    `wait_then_retry`, `review_auth_configuration`
    (`packages/gateway-protocol/src/connect-error-details.ts`).
- Clientverhalten für `AUTH_TOKEN_MISMATCH`:
  - Vertrauenswürdige Clients dürfen einen einzigen begrenzten Wiederholungsversuch mit einem zwischengespeicherten gerätespezifischen
    Token unternehmen.
  - Wenn dieser Wiederholungsversuch fehlschlägt, beenden Sie automatische Schleifen zum erneuten Verbindungsaufbau und zeigen Sie
    Hinweise zu den erforderlichen Maßnahmen des Operators an.
- `AUTH_SCOPE_MISMATCH` bedeutet, dass das Geräte-Token erkannt wurde, aber die
  angeforderte Rolle bzw. die angeforderten Geltungsbereiche nicht abdeckt. Stellen Sie dies nicht als ungültiges Token dar; fordern Sie
  den Operator auf, das Gerät erneut zu koppeln oder den engeren/weiteren Geltungsbereichsvertrag zu genehmigen.

## Geräteidentität und Pairing

- Nodes sollten eine stabile Geräteidentität (`device.id`) enthalten, die aus dem
  Fingerabdruck eines Schlüsselpaars abgeleitet wird.
- Gateways stellen Token pro Gerät und Rolle aus.
- Pairing-Genehmigungen sind für neue Geräte-IDs erforderlich, sofern die lokale
  automatische Genehmigung nicht aktiviert ist.
- Die automatische Pairing-Genehmigung ist auf direkte lokale Loopback-Verbindungen ausgerichtet.
- OpenClaw verfügt außerdem über einen eng begrenzten lokalen Selbstverbindungspfad für Backend/Container bei
  vertrauenswürdigen Hilfsabläufen mit gemeinsamem Geheimnis.
- Tailnet- oder LAN-Verbindungen auf demselben Host werden beim Pairing weiterhin als remote
  behandelt und müssen genehmigt werden.
- WS-Clients geben normalerweise während `connect` eine `device`-Identität an (Operator +
  Node). Die einzigen Ausnahmen für Operatoren ohne Gerät sind explizite Vertrauenspfade:
  - erfolgreiche Operatorauthentifizierung der Control UI über `gateway.auth.mode: "trusted-proxy"`.
  - Direkte Loopback-Backend-RPCs über `gateway-client` auf dem reservierten internen
    Hilfspfad.
- Das Weglassen der Geräteidentität hat Auswirkungen auf die Geltungsbereiche. Wenn eine gerätelose
  Operatorverbindung über einen expliziten Vertrauenspfad zugelassen wird, setzt OpenClaw
  selbst deklarierte Geltungsbereiche dennoch auf eine leere Menge zurück, sofern dieser Pfad keine
  benannte Ausnahme zur Beibehaltung von Geltungsbereichen hat. Methoden mit Geltungsbereichsprüfung schlagen dann mit
  `missing scope` fehl.
- Der reservierte direkte Loopback-Backend-Hilfspfad `gateway-client` behält
  Geltungsbereiche nur für interne lokale RPCs der Steuerungsebene bei; benutzerdefinierte Backend-IDs
  erhalten diese Ausnahme nicht.
- Alle Verbindungen müssen die vom Server bereitgestellte Nonce `connect.challenge` signieren.

### Migrationsdiagnose für die Geräteauthentifizierung

Für ältere Clients, die weiterhin das Signaturverhalten vor Einführung der Challenge verwenden, gibt `connect`
unter `error.details.code` die Detailcodes `DEVICE_AUTH_*` mit einem stabilen
`error.details.reason` zurück.

Häufige Migrationsfehler:

| Meldung                     | details.code                     | details.reason           | Bedeutung                                            |
| --------------------------- | -------------------------------- | ------------------------ | -------------------------------------------------- |
| `device nonce required`     | `DEVICE_AUTH_NONCE_REQUIRED`     | `device-nonce-missing`   | Der Client hat `device.nonce` ausgelassen (oder leer gesendet).     |
| `device nonce mismatch`     | `DEVICE_AUTH_NONCE_MISMATCH`     | `device-nonce-mismatch`  | Der Client hat mit einer veralteten/falschen Nonce signiert.            |
| `device signature invalid`  | `DEVICE_AUTH_SIGNATURE_INVALID`  | `device-signature`       | Die Signaturnutzlast entspricht nicht der v2-Nutzlast.       |
| `device signature expired`  | `DEVICE_AUTH_SIGNATURE_EXPIRED`  | `device-signature-stale` | Der signierte Zeitstempel liegt außerhalb der zulässigen Abweichung.          |
| `device identity mismatch`  | `DEVICE_AUTH_DEVICE_ID_MISMATCH` | `device-id-mismatch`     | `device.id` entspricht nicht dem Fingerabdruck des öffentlichen Schlüssels. |
| `device public key invalid` | `DEVICE_AUTH_PUBLIC_KEY_INVALID` | `device-public-key`      | Format/Kanonisierung des öffentlichen Schlüssels ist fehlgeschlagen.         |

Migrationsziel:

- Warten Sie immer auf `connect.challenge`.
- Signieren Sie die v2-Nutzlast, die die Server-Nonce enthält.
- Senden Sie dieselbe Nonce in `connect.params.device.nonce`.
- Die bevorzugte Signaturnutzlast ist `v3`
  (`buildDeviceAuthPayloadV3` in `packages/gateway-client/src/device-auth.ts`),
  die zusätzlich zu Geräte-/Client-/Rollen-/Geltungsbereichs-/Token-/Nonce-Feldern auch
  `platform` und `deviceFamily` bindet.
- Ältere `v2`-Signaturen werden aus Kompatibilitätsgründen weiterhin akzeptiert, aber die Fixierung
  der Metadaten gekoppelter Geräte steuert beim erneuten Verbindungsaufbau weiterhin die Befehlsrichtlinie.

## TLS und Anheftung

- TLS wird für WS-Verbindungen unterstützt (`gateway.tls`-Konfiguration).
- Clients können den Fingerabdruck des Gateway-Zertifikats optional über
  `gateway.remote.tlsFingerprint` oder die CLI-Option `--tls-fingerprint` anheften.

## Umfang

Dieses Protokoll stellt die vollständige Gateway-API bereit: Status, Kanäle, Modelle, Chat,
Agent, Sitzungen, Nodes, Genehmigungen und mehr. Die genaue Oberfläche wird durch
die aus `packages/gateway-protocol/src/schema.ts` erneut exportierten TypeBox-Schemata definiert.

## Verwandte Themen

- [Einen Gateway-Client erstellen](https://docs.openclaw.ai/gateway/clients)
- [OpenClaw einbetten](https://docs.openclaw.ai/gateway/embedding)
- [Bridge-Protokoll](/de/gateway/bridge-protocol)
- [Gateway-Betriebshandbuch](/de/gateway)
