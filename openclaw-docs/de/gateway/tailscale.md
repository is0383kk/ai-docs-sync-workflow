---
read_when:
    - Bereitstellen der Gateway Control UI außerhalb von localhost
    - Tailnet- oder öffentlichen Dashboard-Zugriff automatisieren
summary: Integriertes Tailscale Serve/Funnel für das Gateway-Dashboard
title: Tailscale
x-i18n:
    generated_at: "2026-07-26T18:30:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e201a64ac427994401fae1b934d94e0c5afe976b4acd34d45b059978f5f1807e
    source_path: gateway/tailscale.md
    workflow: 16
---

OpenClaw kann Tailscale **Serve** (Tailnet) oder **Funnel** (öffentlich) für das Gateway-Dashboard und den WebSocket-Port automatisch konfigurieren. Dadurch bleibt das Gateway an die Loopback-Schnittstelle gebunden, während Tailscale HTTPS, Routing und (bei Serve) Identitäts-Header bereitstellt.

## Modi

`gateway.tailscale.mode`:

| Modus            | Verhalten                                                                    |
| --------------- | --------------------------------------------------------------------------- |
| `serve`         | Serve nur im Tailnet über `tailscale serve`. Das Gateway bleibt auf `127.0.0.1`. |
| `funnel`        | Öffentliches HTTPS über `tailscale funnel`. Erfordert ein gemeinsames Passwort.            |
| `off` (Standard) | Keine Tailscale-Automatisierung.                                                    |

Status- und Audit-Ausgaben verwenden **Tailscale-Exposition** für diesen Serve-/Funnel-Modus von OpenClaw. `off` bedeutet, dass OpenClaw Serve oder Funnel nicht verwaltet; es bedeutet nicht, dass der lokale Tailscale-Daemon angehalten oder abgemeldet ist.

## Konfigurationsbeispiele

### Nur Tailnet (Serve)

```json5
{
  gateway: {
    bind: "loopback",
    tailscale: { mode: "serve" },
  },
}
```

Öffnen: `https://<magicdns>/` (oder Ihr konfiguriertes `gateway.controlUi.basePath`)

Um die Control UI über einen benannten Tailscale-Service statt über den Geräte-Hostnamen bereitzustellen, setzen Sie `gateway.tailscale.serviceName` auf den Servicenamen:

```json5
{
  gateway: {
    bind: "loopback",
    tailscale: { mode: "serve", serviceName: "svc:openclaw" },
  },
}
```

Beim Start wird dann die Service-URL als `https://openclaw.<tailnet-name>.ts.net/` statt des Geräte-Hostnamens ausgegeben. Tailscale Services setzen voraus, dass der Host ein genehmigter, mit einem Tag versehener Node in Ihrem Tailnet ist. Konfigurieren Sie das Tag und genehmigen Sie den Service in Tailscale, bevor Sie diese Funktion aktivieren; andernfalls schlägt `tailscale serve --service=...` beim Start des Gateways fehl.

### Nur Tailnet (Bindung an Tailnet-IP)

Verwenden Sie dies, damit das Gateway ohne Serve/Funnel direkt auf der Tailnet-IP lauscht:

```json5
{
  gateway: {
    bind: "tailnet",
    auth: { mode: "token", token: "your-token" },
  },
}
```

Von einem anderen Tailnet-Gerät verbinden:

- Control UI: `http://<tailscale-ip>:18789/`
- WebSocket: `ws://<tailscale-ip>:18789`

<Note>
Wenn eine bindungsfähige Tailnet-IPv4-Adresse vorhanden ist, benötigt das Gateway für authentifizierte Clients auf demselben Host zusätzlich `http://127.0.0.1:18789`. Wenn beim Start keine Tailnet-Adresse verfügbar ist, wird ausschließlich auf Loopback zurückgegriffen. Starten Sie das Gateway neu, nachdem Tailscale verfügbar geworden ist, um direkten Tailnet-Zugriff hinzuzufügen. Keiner der beiden Pfade ermöglicht einen Zugriff über das LAN oder das öffentliche Internet.
</Note>

### Öffentliches Internet (Funnel + gemeinsames Passwort)

```json5
{
  gateway: {
    bind: "loopback",
    tailscale: { mode: "funnel" },
    auth: { mode: "password", password: "replace-me" },
  },
}
```

Verwenden Sie vorzugsweise `OPENCLAW_GATEWAY_PASSWORD`, statt ein Passwort auf dem Datenträger zu speichern.

## CLI-Beispiele

```bash
openclaw gateway --tailscale serve
openclaw gateway --tailscale funnel --auth password
```

## Authentifizierung

`gateway.auth.mode` steuert den Handshake:

| Modus                                                   | Anwendungsfall                                                                            |
| ------------------------------------------------------ | ----------------------------------------------------------------------------------- |
| `none`                                                 | Nur privater Ingress                                                                |
| `token` (Standard, wenn `OPENCLAW_GATEWAY_TOKEN` gesetzt ist) | Gemeinsames Token                                                                        |
| `password`                                             | Gemeinsames Geheimnis über `OPENCLAW_GATEWAY_PASSWORD` oder die Konfiguration                             |
| `trusted-proxy`                                        | Identitätsbewusster Reverse-Proxy; siehe [Authentifizierung über vertrauenswürdigen Proxy](/de/gateway/trusted-proxy-auth) |

### Tailscale-Identitäts-Header (nur Serve)

Wenn `tailscale.mode: "serve"` und `gateway.auth.allowTailscale` auf `true` gesetzt sind, kann die Authentifizierung der Control UI bzw. von WebSocket Tailscale-Identitäts-Header (`tailscale-user-login`) anstelle eines Tokens/Passworts verwenden. OpenClaw überprüft den Header, indem es die `x-forwarded-for`-Adresse der Anfrage über den lokalen Tailscale-Daemon (`tailscale whois`) auflöst und sie mit der Anmeldung im Header abgleicht, bevor die Anfrage akzeptiert wird. Eine Anfrage erfüllt die Voraussetzungen für diesen Pfad nur, wenn sie von der Loopback-Schnittstelle eingeht und die Tailscale-Header `x-forwarded-for`, `x-forwarded-proto` und `x-forwarded-host` enthält.

Dieser tokenlose Ablauf setzt voraus, dass der Gateway-Host vertrauenswürdig ist. Wenn nicht vertrauenswürdiger lokaler Code auf demselben Host ausgeführt werden kann, setzen Sie `gateway.auth.allowTailscale: false` und verlangen Sie stattdessen eine Token-/Passwortauthentifizierung.

Umfang der Umgehung:

- Gilt nur für die WebSocket-Authentifizierungsoberfläche der Control UI. HTTP-API-Endpunkte (`/v1/*`, `/tools/invoke`, `/api/channels/*` usw.) verwenden niemals eine Authentifizierung über Tailscale-Identitäts-Header; sie folgen immer dem normalen HTTP-Authentifizierungsmodus des Gateways.
- Bei Bedienersitzungen der Control UI, die bereits eine Browser-Geräteidentität enthalten, überspringt eine verifizierte Tailscale-Identität den Bootstrap-Token-/QR-Kopplungsdurchlauf.
- Die Geräteidentität selbst wird dadurch nicht umgangen: Clients ohne Gerät werden weiterhin abgelehnt, und Verbindungen mit Node-Rolle durchlaufen weiterhin die normalen Kopplungs- und Authentifizierungsprüfungen.

## Hinweise

- Tailscale Serve/Funnel setzt voraus, dass die `tailscale`-CLI installiert und angemeldet ist.
- `tailscale.mode: "funnel"` verweigert den Start, sofern der Authentifizierungsmodus nicht `password` ist, um eine öffentliche Exposition zu vermeiden.
- `gateway.tailscale.serviceName` gilt nur für den Serve-Modus und wird an `tailscale serve --service=<name>` übergeben. Der Wert muss das `svc:<dns-label>`-Format von Tailscale verwenden, beispielsweise `svc:openclaw`. Tailscale setzt voraus, dass Service-Hosts Nodes mit Tags sind, und der Service muss möglicherweise in der Administrationskonsole genehmigt werden, bevor Serve ihn veröffentlichen kann.
- `gateway.tailscale.resetOnExit` macht die Konfiguration von `tailscale serve`/`tailscale funnel` beim Herunterfahren rückgängig.
- `gateway.tailscale.preserveFunnel: true` hält eine extern konfigurierte `tailscale funnel`-Route über Gateway-Neustarts hinweg aktiv. Bei `mode: "serve"` prüft OpenClaw `tailscale funnel status`, bevor Serve erneut angewendet wird, und überspringt dies, wenn eine Funnel-Route den Gateway-Port bereits abdeckt. Die ausschließlich passwortbasierte Richtlinie für von OpenClaw verwaltete Funnel-Verbindungen bleibt unverändert.
- `gateway.bind: "tailnet"` verwendet eine direkte Tailnet-Bindung (kein HTTPS, kein Serve/Funnel) sowie das erforderliche lokale `127.0.0.1`, wenn eine Tailnet-IPv4-Adresse verfügbar ist; andernfalls wird ausschließlich auf Loopback zurückgegriffen.
- `gateway.bind: "auto"` bevorzugt Loopback; verwenden Sie `tailnet`, um die Netzwerkexposition auf das Tailnet zu beschränken und gleichzeitig den Loopback-Zugriff auf demselben Host beizubehalten.
- Serve/Funnel stellen nur die **Gateway-Control-UI und WS** bereit. Nodes verbinden sich über denselben Gateway-WS-Endpunkt, daher funktioniert Serve auch für den Node-Zugriff.

### Voraussetzungen und Einschränkungen von Tailscale

- Serve setzt voraus, dass HTTPS für Ihr Tailnet aktiviert ist; die CLI fordert Sie dazu auf, falls es fehlt.
- Serve fügt Tailscale-Identitäts-Header ein; Funnel nicht.
- Funnel setzt Tailscale v1.38.3+, MagicDNS, aktiviertes HTTPS und ein Funnel-Node-Attribut voraus.
- Funnel unterstützt über TLS nur die Ports `443`, `8443` und `10000`.
- Funnel unter macOS setzt die Open-Source-Variante der Tailscale-App voraus.

## Browsersteuerung (entferntes Gateway + lokaler Browser)

Um das Gateway auf einem Rechner auszuführen, aber einen Browser auf einem anderen zu steuern, führen Sie einen **Node-Host** auf dem Browserrechner aus und belassen Sie beide im selben Tailnet. Das Gateway leitet Browseraktionen an den Node weiter; ein separater Steuerungsserver oder eine Serve-URL ist nicht erforderlich.

Vermeiden Sie Funnel für die Browsersteuerung; behandeln Sie die Node-Kopplung wie Bedienerzugriff.

## Weitere Informationen

- Übersicht über Tailscale Serve: [https://tailscale.com/kb/1312/serve](https://tailscale.com/kb/1312/serve)
- `tailscale serve`-Befehl: [https://tailscale.com/kb/1242/tailscale-serve](https://tailscale.com/kb/1242/tailscale-serve)
- Übersicht über Tailscale Funnel: [https://tailscale.com/kb/1223/tailscale-funnel](https://tailscale.com/kb/1223/tailscale-funnel)
- `tailscale funnel`-Befehl: [https://tailscale.com/kb/1311/tailscale-funnel](https://tailscale.com/kb/1311/tailscale-funnel)

## Verwandte Themen

- [Remotezugriff](/de/gateway/remote)
- [Erkennung](/de/gateway/discovery)
- [Authentifizierung](/de/gateway/authentication)
