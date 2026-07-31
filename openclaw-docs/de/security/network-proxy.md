---
read_when:
    - Sie möchten mehrschichtigen Schutz vor SSRF- und DNS-Rebinding-Angriffen.
    - Konfigurieren eines externen Forward-Proxys für den OpenClaw-Laufzeitdatenverkehr
summary: So leiten Sie den HTTP- und WebSocket-Datenverkehr der OpenClaw-Runtime über einen vom Betreiber verwalteten Filter-Proxy weiter
title: Netzwerk-Proxy
x-i18n:
    generated_at: "2026-07-26T18:38:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e948189d691e2cfe32e911e24071fd77157397b510d606423ef738c2565071b5
    source_path: security/network-proxy.md
    workflow: 16
---

OpenClaw kann HTTP- und WebSocket-Datenverkehr zur Laufzeit über einen vom Betreiber verwalteten Forward-Proxy leiten. Dies ist eine optionale mehrschichtige Schutzmaßnahme: zentrale Kontrolle des ausgehenden Datenverkehrs, stärkerer SSRF-Schutz und Überprüfbarkeit der Ziele an der Netzwerkgrenze. Da der Proxy das Ziel beim Verbindungsaufbau auswertet, also nach der DNS-Auflösung und unmittelbar bevor er die Upstream-Verbindung öffnet, verkleinert er außerdem das Zeitfenster, das ein DNS-Rebinding-Angriff zwischen einer früheren DNS-Prüfung auf Anwendungsebene und der tatsächlichen ausgehenden Verbindung ausnutzt. Eine einheitliche Proxy-Richtlinie bietet Betreibern zudem eine zentrale Stelle, um Zielregeln, Netzwerksegmentierung, Ratenbegrenzungen oder Positivlisten für ausgehende Verbindungen durchzusetzen, ohne OpenClaw neu erstellen zu müssen.

OpenClaw liefert keinen Proxy mit, lädt keinen herunter, startet und konfiguriert keinen und zertifiziert keinen Proxy. Sie betreiben die für Ihre Umgebung geeignete Proxy-Technologie; OpenClaw leitet seine eigenen HTTP- und WebSocket-Clients darüber.

## Konfiguration

```yaml
proxy:
  proxyUrl: http://127.0.0.1:3128
```

Sie können die URL auch über die Umgebung festlegen:

```bash
OPENCLAW_PROXY_URL=http://127.0.0.1:3128 openclaw gateway run
```

`proxy.proxyUrl` hat Vorrang vor `OPENCLAW_PROXY_URL`. Eine konfigurierte URL aktiviert die verwaltete Proxy-Weiterleitung; wenn beide URLs entfernt werden, wird sie deaktiviert.

| Schlüssel              | Typ                                  | Standardwert   | Hinweise                                                                                                                              |
| ---------------------- | ------------------------------------ | -------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `proxy.proxyUrl`     | Zeichenfolge                         | nicht gesetzt  | Forward-Proxy-URL für `http://` oder `https://`. In der URL eingebettete Anmeldedaten werden als vertraulich behandelt und in Snapshots/Protokollen unkenntlich gemacht. |
| `proxy.tls.caFile`     | Zeichenfolge                         | nicht gesetzt  | CA-Bundle zur Überprüfung eines mit einer privaten CA signierten `https://`-Proxy-Endpunkts.                                  |
| `proxy.loopbackMode`     | `gateway-only` \| `proxy` \| `block` | `gateway-only` | Steuert das Verhalten bei der Umgehung von Loopback-Verbindungen; siehe unten.                                                       |

Speichern Sie bei verwalteten Gateway-Diensten die URL in der Konfiguration, damit sie eine Neuinstallation überdauert, anstatt sich auf die Umgebung eines Vordergrundprozesses zu verlassen:

```bash
openclaw config set proxy.proxyUrl http://127.0.0.1:3128
openclaw gateway install --force
openclaw gateway start
```

Der Umgebungs-Fallback `OPENCLAW_PROXY_URL` eignet sich am besten für Vordergrundausführungen. Um ihn mit einem installierten Dienst zu verwenden, tragen Sie ihn in die persistente Umgebung des Dienstes ein (`$OPENCLAW_STATE_DIR/.env`, standardmäßig `~/.openclaw/.env`) und installieren Sie den Dienst anschließend neu, damit launchd/systemd/Geplante Aufgaben ihn übernimmt.

### HTTPS-Proxy-Endpunkt mit privater CA

```yaml
proxy:
  proxyUrl: https://proxy.corp.example:8443
  tls:
    caFile: /etc/openclaw/proxy-ca.pem
```

`proxy.tls.caFile` überprüft das eigene TLS-Zertifikat des Proxy-Endpunkts. Dies ist weder eine Vertrauenseinstellung für einen MITM am Ziel noch ein Clientzertifikat oder ein Ersatz für die Zielrichtlinie des Proxys. Verwenden Sie stattdessen `NODE_EXTRA_CA_CERTS` nur, wenn der gesamte Node-Prozess bereits beim Start einer zusätzlichen CA vertrauen muss (zum Beispiel bei einem unternehmensweiten TLS-Inspektionssystem, das jedes HTTPS-Zielzertifikat neu signiert) — diese Variable gilt prozessweit und muss vor dem Start von Node gesetzt werden. Daher kann OpenClaw sie nicht während der Ausführung anwenden, wie dies bei `proxy.tls.caFile` möglich ist. Bevorzugen Sie `proxy.tls.caFile` für das Vertrauen in HTTPS-Proxy-Endpunkte: Es ist auf die verwaltete Proxy-Weiterleitung beschränkt, statt für den gesamten Prozess zu gelten.

```bash
openclaw config set proxy.proxyUrl https://proxy.corp.example:8443
openclaw config set proxy.tls.caFile /etc/openclaw/proxy-ca.pem
openclaw gateway run
```

## Funktionsweise der Weiterleitung

Mit einer gültigen Proxy-URL leiten geschützte Laufzeitprozesse (`openclaw gateway run`, `openclaw node run`, `openclaw agent --local`) normalen ausgehenden HTTP- und WebSocket-Datenverkehr über den Proxy:

```text
OpenClaw-Prozess
  fetch, node:http, node:https, WebSocket-Clients  -> Betreiber-Proxy -> Ziel
```

Intern installiert OpenClaw [Proxyline](https://github.com/openclaw/proxyline) als prozessweite Laufzeit für die Weiterleitung. Sie deckt `fetch`, auf undici basierende Clients, `node:http`/`node:https`, gängige WebSocket-Clients und durch Hilfsfunktionen erstellte `CONNECT`-Tunnel ab und ersetzt vom Aufrufer bereitgestellte Node-HTTP-Agents, sodass explizite Agents (einschließlich `axios`, `got`, `node-fetch` und ähnlicher auf Node-Agents basierender Clients) den Proxy nicht unbemerkt umgehen können.

Das Schema der Proxy-URL beschreibt die Verbindung von OpenClaw zum Proxy, nicht zum endgültigen Ziel:

- `http://proxy.example:3128` — unverschlüsselte TCP-Verbindung zum Proxy; OpenClaw sendet HTTP-Proxy-Anfragen, einschließlich `CONNECT` für HTTPS-Ziele.
- `https://proxy.example:8443` — OpenClaw baut eine TLS-Verbindung zum Proxy selbst auf (einschließlich Überprüfung des Proxy-Zertifikats) und sendet anschließend innerhalb dieser Sitzung HTTP-Proxy-Anfragen.

Ziel-TLS ist unabhängig von TLS am Proxy-Endpunkt: Bei einem HTTPS-Ziel fordert OpenClaw den Proxy immer zu einem `CONNECT`-Tunnel auf und startet Ziel-TLS durch diesen Tunnel.

Während der Proxy aktiv ist, löscht OpenClaw `no_proxy`/`NO_PROXY`. Diese Umgehungslisten sind zielbasiert; würden `localhost` oder `127.0.0.1` darin verbleiben, könnten SSRF-Ziele den Proxy vollständig umgehen. Beim Herunterfahren stellt OpenClaw die vorherige Proxy-Umgebung wieder her und setzt den zwischengespeicherten Weiterleitungsstatus zurück.

Einige Plugins besitzen einen benutzerdefinierten Transport, der auch bei aktiver prozessweiter Weiterleitung eine eigene Proxy-Anbindung benötigt. Der Bot-API-Client von Telegram verwendet einen eigenen HTTP/1-undici-Dispatcher und berücksichtigt separat die Proxy-Umgebung des Prozesses sowie den Fallback `OPENCLAW_PROXY_URL`.

### Gateway-Loopback-Modus

Lokale Clients der Gateway-Steuerungsebene verbinden sich normalerweise mit einem Loopback-WebSocket wie `ws://127.0.0.1:18789`. `proxy.loopbackMode` steuert, ob dieser Datenverkehr den verwalteten Proxy umgeht:

```yaml
proxy:
  proxyUrl: http://127.0.0.1:3128
  loopbackMode: gateway-only # gateway-only, proxy, or block
```

Ein konfiguriertes `proxyUrl` oder `OPENCLAW_PROXY_URL` aktiviert die verwaltete Weiterleitung. Legen Sie
`proxy.enabled: false` nur als erweiterte Abwahloption fest, bei der die URL gespeichert bleibt,
ohne sie zu aktivieren.

| Modus                         | Verhalten                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `gateway-only` (Standard) | OpenClaw registriert die aktive Loopback-Autorität des Gateways als Ausnahme für direkte Verbindungen, sodass sich lokaler Gateway-WebSocket-Datenverkehr ohne Proxy verbindet. Benutzerdefinierte Loopback-Ports funktionieren, da die Ausnahme genau auf den konfigurierten Host/Port ausgerichtet ist. Das mitgelieferte Browser-Plugin registriert dieselbe Art von Ausnahme für die exakten lokalen CDP-Bereitschafts- und DevTools-WebSocket-URLs verwalteter, von OpenClaw gestarteter Browser; der mitgelieferte Provider für Ollama-Speichereinbettungen verfügt über einen enger begrenzten, abgesicherten direkten Pfad für seinen exakt konfigurierten lokalen Loopback-Ursprung der Einbettungen. |
| `proxy`            | Es werden keine Loopback-Ausnahmen registriert; der Loopback-Datenverkehr von Gateway und Ollama wird über den Proxy geleitet. Ein entfernter Proxy muss zurück zum Loopback-Dienst des OpenClaw-Hosts routen können (zum Beispiel über einen erreichbaren Hostnamen, eine IP-Adresse oder einen Tunnel) — ein gewöhnlicher entfernter Proxy löst `127.0.0.1`/`localhost` relativ zu sich selbst auf, nicht relativ zum OpenClaw-Host.                                                                                                                                                           |
| `block`            | OpenClaw verweigert Loopback-Verbindungen zur Gateway-Steuerungsebene sowie abgesicherte Loopback-Verbindungen für Ollama-Einbettungen, bevor ein Socket geöffnet wird.                                                                                                                                                                                                                                                                                                                                                                                               |

Die Umgehung für die Gateway-Steuerungsebene ist auf `localhost` und URLs mit wörtlichen Loopback-IP-Adressen beschränkt — verwenden Sie `ws://127.0.0.1:18789`, `ws://[::1]:18789` oder `ws://localhost:18789`. Andere Hostnamen werden wie gewöhnlicher Datenverkehr weitergeleitet.

### Container

Für `openclaw --container ...`-Befehle leitet OpenClaw `OPENCLAW_PROXY_URL` an die auf den Container ausgerichtete untergeordnete CLI weiter, wenn die Variable gesetzt ist. Die URL muss aus dem Container heraus erreichbar sein — `127.0.0.1` bezeichnet dort den Container selbst, nicht den Host. OpenClaw lehnt Loopback-Proxy-URLs für auf Container ausgerichtete Befehle ab, sofern Sie nicht `OPENCLAW_CONTAINER_ALLOW_LOOPBACK_PROXY_URL=1` festlegen, um diese Prüfung ausdrücklich zu überschreiben.

## Verwandte Proxy-Begriffe

- `proxy.enabled` / `proxy.proxyUrl` — ausgehende Forward-Proxy-Weiterleitung für Laufzeitdatenverkehr. Diese Seite.
- `gateway.auth.mode: "trusted-proxy"` — eingehende identitätsbezogene Reverse-Proxy-Authentifizierung für den Gateway-Zugriff. Siehe [Authentifizierung über vertrauenswürdige Proxys](/de/gateway/trusted-proxy-auth).
- `openclaw proxy` — lokaler Debug-Proxy und Erfassungsinspektor für Entwicklung und Support. Siehe [openclaw proxy](/de/cli/proxy).
- `tools.web.fetch.useTrustedEnvProxy` — Opt-in für `web_fetch`, damit ein vom Betreiber kontrollierter HTTP(S)-Umgebungs-Proxy DNS auflösen kann, während standardmäßig eine strikte DNS-Bindung und Hostnamenrichtlinie beibehalten werden. Siehe [Webabruf](/de/tools/web-fetch#trusted-env-proxy).
- Kanal- oder Provider-spezifische Proxy-Einstellungen — besitzerspezifische Überschreibungen für einen einzelnen Transport. Bevorzugen Sie den verwalteten Netzwerk-Proxy für eine zentrale Kontrolle des ausgehenden Datenverkehrs über die gesamte Laufzeit hinweg.

## Proxy validieren

Die Zielrichtlinie des Proxys bildet die eigentliche Sicherheitsgrenze; OpenClaw kann nicht überprüfen, ob Ihr Proxy die richtigen Ziele blockiert. Konfigurieren Sie ihn so, dass er:

- nur an Loopback oder eine private vertrauenswürdige Schnittstelle bindet, die ausschließlich für den OpenClaw-Prozess/-Host/-Container oder das Dienstkonto erreichbar ist.
- Ziele selbst auflöst und nach der DNS-Auflösung beim Verbindungsaufbau anhand der IP blockiert, sowohl für unverschlüsseltes HTTP als auch für HTTPS-`CONNECT`-Tunnel.
- zielbasierte Umgehungen für Loopback-, private, Link-Local-, Metadaten-, Multicast-, reservierte und Dokumentationsadressbereiche ablehnt.
- Positivlisten für Hostnamen vermeidet, sofern Sie dem DNS-Auflösungspfad nicht vollständig vertrauen.
- Ziel, Entscheidung, Status und Begründung protokolliert — niemals Anfragetexte, Autorisierungsheader, Cookies oder andere Geheimnisse.
- die Richtlinie unter Versionskontrolle hält und Änderungen als sicherheitskritisch überprüft.

Validieren Sie von demselben Host, Container oder Dienstkonto aus, unter dem OpenClaw ausgeführt wird:

```bash
openclaw proxy validate --proxy-url http://127.0.0.1:3128
```

Mit einem HTTPS-Proxy-Endpunkt mit privater CA:

```bash
openclaw proxy validate --proxy-url https://proxy.corp.example:8443 --proxy-ca-file /etc/openclaw/proxy-ca.pem
```

| Flag                     | Zweck                                                                |
| ------------------------ | -------------------------------------------------------------------- |
| `--proxy-url <url>`      | Diese URL validieren, statt Konfiguration/Umgebung aufzulösen.       |
| `--proxy-ca-file <path>` | CA-Bundle für einen HTTPS-Proxy-Endpunkt.                             |
| `--allowed-url <url>`    | Ziel, das erwartungsgemäß erreichbar ist (wiederholbar).             |
| `--denied-url <url>`     | Ziel, das erwartungsgemäß blockiert wird (wiederholbar).             |
| `--apns-reachable`       | Zusätzlich prüfen, ob der Proxy einen direkten APNs-HTTP/2-Test der Sandbox tunneln kann. |
| `--apns-authority <url>` | Die mit `--apns-reachable` geprüfte APNs-Autorität überschreiben.    |
| `--timeout-ms <ms>`      | Zeitüberschreitung pro Anfrage.                                      |
| `--json`                 | Maschinenlesbare Ausgabe.                                            |

Wenn weder eine Konfiguration noch eine Umgebungsvariable oder ein Wert für `--proxy-url` verfügbar ist, meldet der Befehl ein Konfigurationsproblem; übergeben Sie `--proxy-url` für eine einmalige Vorabprüfung, bevor Sie die Konfiguration ändern.

Ohne `--allowed-url`/`--denied-url` gelten folgende Standardprüfungen: `https://example.com/` muss erfolgreich sein, und ein temporärer Loopback-Canary-Server, den der Proxy nicht erreichen darf, muss blockiert werden. Die Loopback-Prüfung ist bei einem Transportfehler oder bei einer Nicht-2xx-Antwort ohne das ausführungsspezifische Token des Canary erfolgreich. Sie schlägt bei einer 2xx-Antwort ohne das Token fehl (ein unerwarteter Erfolg von etwas anderem als dem Canary) und insbesondere bei jeder Antwort mit dem passenden Token, da dies beweist, dass der Proxy tatsächlich ein Loopback-Ziel weitergeleitet hat, das er hätte ablehnen müssen. Benutzerdefinierte `--denied-url`-Ziele verfügen über kein solches Canary-Token und verwenden daher ein Fail-Closed-Verhalten: Jede HTTP-Antwort gilt als erreichbar (Fehler), und ein Transportfehler wird als nicht eindeutig statt als nachweislich blockiert gemeldet, da OpenClaw nicht bestätigen kann, ob Ihr Proxy einen erreichbaren Ursprung abgelehnt hat oder ob ein anderer Fehler aufgetreten ist. `--apns-reachable` sendet absichtlich ein ungültiges Provider-Token, sodass eine `403 InvalidProviderToken`-Antwort als Nachweis gilt, dass der Tunnel Apple erreicht hat. Der Befehl wird bei jedem Validierungsfehler mit `1` beendet; Anmeldedaten in der Proxy-URL werden sowohl in der Text- als auch in der JSON-Ausgabe unkenntlich gemacht.

```json
{
  "ok": true,
  "config": {
    "enabled": true,
    "proxyUrl": "http://127.0.0.1:3128/",
    "source": "override",
    "errors": []
  },
  "checks": [
    { "kind": "allowed", "url": "https://example.com/", "ok": true, "status": 200 },
    { "kind": "apns", "url": "https://api.sandbox.push.apple.com", "ok": true, "status": 403 }
  ]
}
```

Manuelle `curl`-Prüfung (die öffentliche Anfrage sollte erfolgreich sein; die Loopback- und Metadatenanfragen sollten vom Proxy selbst blockiert werden — `curl` allein kann eine Ablehnung durch den Proxy nicht von einem nicht erreichbaren Ursprung unterscheiden, wie dies der integrierte Canary von `openclaw proxy validate` kann):

```bash
curl -x http://127.0.0.1:3128 https://example.com/
curl -x http://127.0.0.1:3128 http://127.0.0.1/
curl -x http://127.0.0.1:3128 http://169.254.169.254/
```

## Empfohlene blockierte Ziele

Ausgangssperrliste für jeden Forward-Proxy, jede Firewall oder jede Egress-Richtlinie. Der OpenClaw-eigene SSRF-Klassifizierer befindet sich in `src/infra/net/ssrf.ts` und `packages/net-policy/src/ip.ts` (`BLOCKED_HOSTNAMES`, `BLOCKED_IPV4_SPECIAL_USE_RANGES`, `BLOCKED_IPV6_SPECIAL_USE_RANGES`, das RFC-2544-Benchmark-Präfix und die Behandlung eingebetteter IPv4-Adressen für NAT64-/6to4-/Teredo-/ISATAP-/IPv4-gemappte Formen) — nützliche Referenzen, OpenClaw exportiert oder erzwingt diese Regeln jedoch nicht in Ihrem externen Proxy.

| Bereich oder Host                                                                      | Grund für die Blockierung                         |
| -------------------------------------------------------------------------------------- | ------------------------------------------------- |
| `127.0.0.0/8`, `localhost`, `localhost.localdomain`                                  | IPv4-Loopback                                     |
| `::1/128`                                                                            | IPv6-Loopback                                     |
| `0.0.0.0/8`, `::/128`                                                                | Nicht spezifizierte Adressen/Adressen dieses Netzwerks |
| `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`                                      | Private RFC-1918-Netzwerke                        |
| `169.254.0.0/16`, `fe80::/10`                                                        | Link-lokal, einschließlich gängiger Cloud-Metadatenpfade |
| `169.254.169.254`, `metadata.google.internal`                                        | Cloud-Metadatendienste                            |
| `100.64.0.0/10`                                                                      | Gemeinsam genutzter Adressraum für Carrier-Grade-NAT |
| `198.18.0.0/15`, `2001:2::/48`                                                       | Benchmark-Bereiche                                |
| `192.0.0.0/24`, `192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24`, `2001:db8::/32` | Bereiche für besondere Verwendung und Dokumentation |
| `224.0.0.0/4`, `ff00::/8`                                                            | Multicast                                         |
| `240.0.0.0/4`                                                                        | Reserviertes IPv4                                 |
| `fc00::/7`, `fec0::/10`                                                              | Lokale/private IPv6-Bereiche                      |
| `100::/64`, `2001:20::/28`                                                           | IPv6-Verwerfungs- und ORCHIDv2-Bereiche           |
| `64:ff9b::/96`, `64:ff9b:1::/48`                                                     | NAT64-Präfixe mit eingebettetem IPv4              |
| `2002::/16`, `2001::/32`                                                             | 6to4 und Teredo mit eingebettetem IPv4            |
| `::/96`, `::ffff:0:0/96`                                                             | IPv4-kompatibles und IPv4-gemapptes IPv6          |

Fügen Sie alle zusätzlichen Metadatenhosts oder reservierten Bereiche hinzu, die Ihr Cloud-Provider oder Ihre Netzwerkplattform dokumentiert.

## Einschränkungen

| Oberfläche                                                   | Status des verwalteten Proxys                                                                                                                            |
| ------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `fetch`, `node:http`, `node:https`, gängige WebSocket-Clients | Werden bei entsprechender Konfiguration über verwaltete Proxy-Hooks geleitet.                                                                            |
| Direktes APNs-HTTP/2                                         | Wird über den verwalteten APNs-Helper `CONNECT` geleitet.                                                                                       |
| Loopback der Gateway-Steuerungsebene                         | Nur für die exakt konfigurierte lokale Loopback-Gateway-URL direkt.                                                                                      |
| Upstream-Weiterleitung des Debug-Proxys                      | Ist im verwalteten Proxy-Modus deaktiviert, sofern sie nicht ausdrücklich für lokale Diagnosen aktiviert wurde.                                          |
| IRC                                                          | Rohes TCP/TLS; wird nicht über den verwalteten HTTP-Proxy-Modus weitergeleitet. Legen Sie `channels.irc.enabled: false` fest, wenn Ihre Bereitstellung sämtlichen Egress-Datenverkehr über den Forward-Proxy leiten muss. |
| Andere rohe Client-Aufrufe von `net`, `tls` oder `http2` | Müssen vor der Übernahme durch die Schutzfunktion für rohe Sockets klassifiziert werden.                                                                 |

- Dies bietet Abdeckung auf Prozessebene für JavaScript-HTTP-/WebSocket-Clients, keine Netzwerk-Sandbox auf Betriebssystemebene.
- Rohe `net`-, `tls`- und `http2`-Sockets, native Add-ons sowie untergeordnete Prozesse außerhalb von OpenClaw können das Routing auf Node-Ebene umgehen, sofern sie Proxy-Umgebungsvariablen nicht erben und berücksichtigen. Abgespaltene untergeordnete OpenClaw-CLIs erben die verwaltete Proxy-URL und den Zustand von `proxy.loopbackMode`.
- Lokale WebUIs der Benutzer und lokale Modellserver werden nicht durch eine allgemeine Umgehung für lokale Netzwerke abgedeckt — nehmen Sie sie bei Bedarf in die Zulassungsliste der Proxy-Richtlinie des Betreibers auf. Eine Ausnahme bildet der geschützte direkte Pfad des gebündelten Ollama-Providers für Memory-Embeddings, der auf den exakten hostlokalen Loopback-Ursprung aus dessen konfiguriertem `baseUrl` beschränkt ist; Ollama-Hosts im LAN, Tailnet, privaten Netzwerk und öffentlichen Netzwerk verwenden weiterhin den verwalteten Proxy.
- Die direkte Upstream-Weiterleitung des lokalen Debug-Proxys (für Proxy-Anfragen und `CONNECT`-Tunnel) ist standardmäßig deaktiviert, solange der verwaltete Proxy-Modus aktiv ist; aktivieren Sie sie nur für genehmigte lokale Diagnosen.
- OpenClaw überprüft, testet oder zertifiziert Ihre Proxy-Richtlinie nicht. Behandeln Sie Änderungen an der Proxy-Richtlinie als sicherheitskritische betriebliche Änderungen.
