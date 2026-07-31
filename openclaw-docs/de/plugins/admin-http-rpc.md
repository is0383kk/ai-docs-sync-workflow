---
read_when:
    - Erstellen von Host-Tools, die den Gateway-WebSocket-RPC-Client nicht verwenden können
    - Gateway-Admin-Automatisierung hinter einem privaten vertrauenswürdigen Ingress bereitstellen
    - Prüfung des Sicherheitsmodells für den HTTP-Zugriff auf Gateway-Methoden
summary: Ausgewählte Methoden der Gateway-Steuerungsebene über das gebündelte, optional aktivierbare Plugin admin-http-rpc bereitstellen
title: Admin-HTTP-RPC-Plugin
x-i18n:
    generated_at: "2026-07-26T18:35:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0709081efd0ce65cef7edac54df9a71978cbad17e2b25df83ac9075de938376c
    source_path: plugins/admin-http-rpc.md
    workflow: 16
---

Das gebündelte Plugin `admin-http-rpc` stellt eine Positivliste von Gateway-Control-Plane-Methoden über HTTP bereit. Es ist für vertrauenswürdige Host-Automatisierung vorgesehen, die keine Gateway-WebSocket-Verbindung offen halten kann.

Es wird mit OpenClaw ausgeliefert, ist jedoch standardmäßig deaktiviert. Im deaktivierten Zustand wird die Route nicht registriert. Wenn es aktiviert ist, fügt es `POST /api/v1/admin/rpc` auf demselben Listener wie das Gateway (`http://<gateway-host>:<port>/api/v1/admin/rpc`) hinzu.

Aktivieren Sie es nur für private Host-Werkzeuge, Tailnet-Automatisierung oder einen vertrauenswürdigen internen Ingress. Stellen Sie diese Route niemals direkt im öffentlichen Internet bereit.

## Vor der Aktivierung

Admin-HTTP-RPC ist eine vollständige Control-Plane-Oberfläche für Operatoren: Jeder Aufrufer, der die Gateway-HTTP-Authentifizierung besteht, kann die unten aufgeführten Methoden aus der Positivliste aufrufen. Aktivieren Sie es nur, wenn alle folgenden Bedingungen erfüllt sind:

- Der Aufrufer ist berechtigt, das Gateway zu betreiben.
- Der Aufrufer kann den WebSocket-RPC-Client nicht verwenden.
- Die Route ist nur über Loopback, ein Tailnet oder einen privaten authentifizierten Ingress erreichbar.
- Sie haben die zulässigen Methoden geprüft und sie entsprechen der geplanten Automatisierung.

Verwenden Sie für OpenClaw-Clients und interaktive Werkzeuge, die eine Gateway-WebSocket-Verbindung offen halten können, stattdessen WebSocket-RPC.

## Aktivieren

Aktivieren Sie das gebündelte Plugin:

<Tabs>
  <Tab title="CLI">
    ```bash
    openclaw plugins enable admin-http-rpc
    openclaw gateway restart
    ```
  </Tab>
  <Tab title="Konfiguration">
    ```json5
    {
      plugins: {
        entries: {
          "admin-http-rpc": { enabled: true },
        },
      },
    }
    ```
  </Tab>
</Tabs>

Die Route wird beim Start des Plugins registriert. Starten Sie daher das Gateway nach einer Änderung der Plugin-Konfiguration neu.

Deaktivieren Sie es, wenn Sie die HTTP-Oberfläche nicht mehr benötigen:

```bash
openclaw plugins disable admin-http-rpc
openclaw gateway restart
```

## Route überprüfen

Verwenden Sie `health` als kleinste sichere Anfrage:

```bash
curl -sS http://<gateway-host>:<port>/api/v1/admin/rpc \
  -H 'Authorization: Bearer <gateway-token>' \
  -H 'Content-Type: application/json' \
  -d '{"method":"health","params":{}}'
```

Eine erfolgreiche Antwort enthält `ok: true`:

```json
{
  "id": "generated-request-id",
  "ok": true,
  "payload": {
    "status": "ok"
  }
}
```

Wenn das Plugin deaktiviert ist, gibt die Route `404` zurück, da sie nicht registriert ist.

## Authentifizierung

Die Plugin-Route verwendet die Gateway-HTTP-Authentifizierung.

Übliche Authentifizierungswege:

- Authentifizierung mit gemeinsamem Geheimnis (`gateway.auth.mode="token"` oder `"password"`): `Authorization: Bearer <token-or-password>`
- vertrauenswürdige identitätstragende HTTP-Authentifizierung (`gateway.auth.mode="trusted-proxy"`): Leiten Sie die Anfrage über den konfigurierten identitätsbewussten Proxy und lassen Sie ihn die erforderlichen Identitäts-Header einfügen
- offene Authentifizierung über privaten Ingress (`gateway.auth.mode="none"`): kein Authentifizierungs-Header erforderlich

## Sicherheitsmodell

Behandeln Sie dieses Plugin als vollständige Operatoroberfläche des Gateways.

- Durch die Aktivierung des Plugins wird absichtlich unter `/api/v1/admin/rpc` Zugriff auf die Admin-RPC-Methoden der Positivliste gewährt.
- Das Plugin deklariert den reservierten Manifest-Vertrag `contracts.gatewayMethodDispatch: ["authenticated-request"]`. Dadurch kann seine Gateway-authentifizierte HTTP-Route Control-Plane-Methoden innerhalb des Prozesses weiterleiten. Dies ist keine Sandbox: Der Vertrag verhindert die versehentliche Verwendung reservierter SDK-Hilfsfunktionen, vertrauenswürdige Plugins werden jedoch weiterhin im Gateway-Prozess ausgeführt.
- Die Bearer-Authentifizierung mit gemeinsamem Geheimnis (Modi `token`/`password`) weist den Besitz des Gateway-Operatorgeheimnisses nach. Enger gefasste `x-openclaw-scopes`-Header werden auf diesem Pfad ignoriert und die normalen vollständigen Operatorstandardwerte werden wiederhergestellt.
- Die vertrauenswürdige identitätstragende HTTP-Authentifizierung (Modus `trusted-proxy`) berücksichtigt `x-openclaw-scopes`, sofern vorhanden.
- `gateway.auth.mode="none"` bedeutet, dass diese Route bei aktiviertem Plugin nicht authentifiziert ist. Verwenden Sie dies nur hinter einem privaten Ingress, dem Sie vollständig vertrauen.
- Nachdem die Authentifizierung der Plugin-Route erfolgreich war, werden Anfragen über dieselben Gateway-Methodenhandler und Bereichsprüfungen wie WebSocket-RPC weitergeleitet.
- Die Route bleibt während einer vorbereiteten Suspendierungslease erreichbar. Begrenzte Anfragevalidierung und die lokale Discovery-Antwort `commands.list` bleiben verfügbar. Von den an das Gateway weitergeleiteten Methoden dürfen bei geschlossener Zulassung nur `gateway.suspend.prepare`, `gateway.suspend.status` und `gateway.suspend.resume` ausgeführt werden; andere Methoden der Positivliste geben die normale wiederholbare Gateway-Antwort `UNAVAILABLE` zurück.
- Beschränken Sie diese Route auf Loopback, ein Tailnet oder einen privaten vertrauenswürdigen Ingress. Stellen Sie sie nicht direkt im öffentlichen Internet bereit. Verwenden Sie separate Gateways, wenn Aufrufer Vertrauensgrenzen überschreiten.

## Anfrage

```http
POST /api/v1/admin/rpc
Authorization: Bearer <gateway-token>
Content-Type: application/json
```

```json
{
  "id": "optional-request-id",
  "method": "health",
  "params": {}
}
```

Felder:

- `id` (Zeichenfolge, optional): wird in die Antwort übernommen. Wenn das Feld ausgelassen wird, wird eine UUID generiert.
- `method` (Zeichenfolge, erforderlich): Name einer zulässigen Gateway-Methode.
- `params` (beliebiger Typ, optional): methodenspezifische Parameter.

Die standardmäßige maximale Größe des Anfragekörpers beträgt 1 MB.

## Antwort

Erfolgreiche Antworten verwenden das Gateway-RPC-Format:

```json
{
  "id": "optional-request-id",
  "ok": true,
  "payload": {}
}
```

Gateway-Methodenfehler verwenden:

```json
{
  "id": "optional-request-id",
  "ok": false,
  "error": {
    "code": "INVALID_REQUEST",
    "message": "bad params"
  }
}
```

Der HTTP-Status richtet sich nach dem Fehlercode:

| Fehlercode                 | HTTP-Status |
| -------------------------- | ----------- |
| `INVALID_REQUEST`          | 400         |
| `APPROVAL_NOT_FOUND`       | 404         |
| `NOT_LINKED`, `NOT_PAIRED` | 409         |
| `UNAVAILABLE`              | 503         |
| `AGENT_TIMEOUT`            | 504         |
| jeder andere Code          | 500         |

## Zulässige Methoden

- Discovery: `commands.list`
  Gibt die Namen der von diesem Plugin zugelassenen HTTP-RPC-Methoden zurück.
- Gateway: `health`, `status`, `logs.tail`, `usage.status`, `usage.cost`, `gateway.restart.request`, `gateway.suspend.prepare`, `gateway.suspend.status`, `gateway.suspend.resume`
- Konfiguration: `config.get`, `config.schema`, `config.schema.lookup`, `config.set`, `config.patch`, `config.apply`
- Kanäle: `channels.status`, `channels.start`, `channels.stop`, `channels.logout`
- Web: `web.login.start`, `web.login.wait`
- Modelle: `models.list`, `models.authStatus`
- Agenten: `agents.list`, `agents.create`, `agents.update`, `agents.delete`
- Genehmigungen: `exec.approvals.get`, `exec.approvals.set`, `exec.approvals.node.get`, `exec.approvals.node.set`
- Cron: `cron.status`, `cron.list`, `cron.get`, `cron.runs`, `cron.add`, `cron.update`, `cron.remove`, `cron.run`
- Geräte: `device.pair.list`, `device.pair.approve`, `device.pair.reject`, `device.pair.remove`
- Nodes: `node.list`, `node.describe`, `node.pair.list`, `node.pair.approve`, `node.pair.reject`, `node.pair.remove`, `node.rename`
- Aufgaben: `tasks.list`, `tasks.get`, `tasks.cancel`
- Diagnose: `doctor.memory.status`, `update.status`

Andere Gateway-Methoden bleiben blockiert, bis sie absichtlich hinzugefügt werden.

## WebSocket-Vergleich

Der normale Gateway-WebSocket-RPC-Pfad bleibt die bevorzugte Control-Plane-API für OpenClaw-Clients. Verwenden Sie Admin-HTTP-RPC nur für Host-Werkzeuge, die eine HTTP-Anfrage-Antwort-Oberfläche benötigen.

WebSocket-Clients mit gemeinsamem Token, die keine vertrauenswürdige Geräteidentität besitzen, können beim Verbindungsaufbau nicht selbst Admin-Bereiche deklarieren. Admin-HTTP-RPC folgt bewusst dem bestehenden Modell für vertrauenswürdige HTTP-Operatoren: Wenn das Plugin aktiviert ist, wird die Bearer-Authentifizierung mit gemeinsamem Geheimnis für diese Admin-Oberfläche als vollständiger Operatorzugriff behandelt.

## Fehlerbehebung

`404 Not Found`

: Das Plugin ist deaktiviert, das Gateway wurde seit der Aktivierung nicht neu gestartet oder die Anfrage wird an einen anderen Gateway-Prozess gesendet.

`401 Unauthorized`

: Die Anfrage hat die Gateway-HTTP-Authentifizierung nicht erfüllt. Prüfen Sie das Bearer-Token oder die Identitäts-Header des vertrauenswürdigen Proxys.

`405 Method Not Allowed`

: Die Anfrage verwendete etwas anderes als `POST`.

`413 Payload Too Large`

: Der Anfragekörper hat das Limit von 1 MB überschritten.

`400 INVALID_REQUEST`

: Der Anfragekörper ist kein gültiges JSON, das Feld `method` fehlt, die Methode befindet sich nicht in der Positivliste des Plugins oder eine Wiederaufnahme-ID der Suspendierung stimmt nicht mit der aktiven Lease überein.

`503 UNAVAILABLE`

: Die Gateway-Methode wird gestartet, ist ratenbegrenzt, suspendiert oder wartet auf einen konkurrierenden Suspendierungs- bzw. Wiederaufnahmevorgang. Prüfen Sie `error.details`, sofern vorhanden, und beachten Sie `error.retryAfterMs`, bevor Sie es erneut versuchen.

## Verwandte Themen

- [Operatorbereiche](/de/gateway/operator-scopes)
- [Gateway-Sicherheit](/de/gateway/security)
- [Remotezugriff](/de/gateway/remote)
- [Plugin-Manifest](/de/plugins/manifest#contracts-reference)
- [SDK-Unterpfade](/de/plugins/sdk-subpaths)
