---
read_when:
    - Implementierung oder Änderung der Bonjour-Erkennung/-Ankündigung
    - Remote-Verbindungsmodi anpassen (direkt oder per SSH)
    - Konzeption der Node-Erkennung und -Kopplung für Remote-Nodes
summary: Node-Erkennung und Übertragungswege (Bonjour, Tailscale, SSH) zum Auffinden des Gateways
title: Erkennung und Transporte
x-i18n:
    generated_at: "2026-07-26T18:26:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3a3f1a6a1212ab0bc7021e77c88de059edcb8e09eff90d3e1e59451b9b20876b
    source_path: gateway/discovery.md
    workflow: 16
---

OpenClaw hat zwei zusammenhängende, aber unterschiedliche Discovery-Anforderungen:

1. **Fernsteuerung durch Bediener**: Die macOS-Menüleisten-App steuert einen Gateway, der an einem anderen Ort ausgeführt wird.
2. **Node-Kopplung**: iOS/Android (und zukünftige Nodes) finden einen Gateway und koppeln sich sicher mit ihm.

Die gesamte Netzwerk-Discovery und -Ankündigung erfolgt im **Node Gateway**
(`openclaw gateway`); Clients (Mac-App, iOS) sind lediglich Nutzer dieser Informationen.

## Begriffe

- **Gateway**: ein einzelner, dauerhaft ausgeführter Prozess, der den Zustand verwaltet (Sitzungen,
  Kopplung, Node-Registrierung) und Kanäle ausführt. Die meisten Konfigurationen verwenden einen pro Host;
  isolierte Konfigurationen mit mehreren Gateways sind möglich.
- **Gateway-WS (Steuerungsebene)**: der WebSocket-Endpunkt, standardmäßig unter `127.0.0.1:18789`;
  binden Sie ihn über `gateway.bind` an das LAN/Tailnet.
- **Direkter WS-Transport**: ein für das LAN/Tailnet erreichbarer Gateway-WS-Endpunkt (ohne SSH).
- **SSH-Transport (Fallback)**: Fernsteuerung durch Weiterleitung von
  `127.0.0.1:18789` über SSH.
- **Veraltete TCP-Bridge (entfernt)**: älterer Node-Transport (siehe
  [Bridge-Protokoll](/de/gateway/bridge-protocol)); wird nicht mehr für die
  Discovery angekündigt und ist nicht mehr Bestandteil aktueller Builds.

Protokolldetails: [Gateway-Protokoll](/de/gateway/protocol),
[Bridge-Protokoll (veraltet)](/de/gateway/bridge-protocol).

## Warum sowohl Direktverbindungen als auch SSH vorhanden sind

- **Direktes WS** bietet im selben Netzwerk und innerhalb eines Tailnets die beste Benutzererfahrung: automatische
  LAN-Discovery über Bonjour, vom Gateway verwaltete Kopplungstoken und ACLs
  sowie kein erforderlicher Shell-Zugriff.
- **SSH** ist der universelle Fallback: Es funktioniert überall, wo SSH-Zugriff besteht, selbst
  über voneinander unabhängige Netzwerke hinweg, ist unempfindlich gegenüber Multicast-/mDNS-Problemen und benötigt außer SSH
  keinen neuen eingehenden Port.

## Discovery-Quellen

### 1) Bonjour / DNS-SD

Multicast-Bonjour arbeitet nach dem Best-Effort-Prinzip und funktioniert nicht netzwerkübergreifend. OpenClaw unterstützt außerdem
das Durchsuchen desselben Gateway-Beacons über eine konfigurierte Wide-Area-DNS-SD-
Domain, sodass die Discovery sowohl `local.` im selben LAN als auch eine konfigurierte
Unicast-DNS-SD-Domain für die netzwerkübergreifende Discovery abdecken kann.

Der **Gateway** kündigt seinen WS-Endpunkt über Bonjour an, wenn das gebündelte
Plugin `bonjour` aktiviert ist; Clients durchsuchen die Einträge und zeigen eine Liste zur Auswahl eines Gateways an,
anschließend speichern sie den ausgewählten Endpunkt.

Fehlerbehebung und Beacon-Details: [Bonjour](/de/gateway/bonjour).

#### Details zum Service-Beacon

- Servicetyp: `_openclaw-gw._tcp` (Beacon für den Gateway-Transport).
- TXT-Schlüssel (nicht geheim):

  | Schlüssel                   | Hinweise                                                                                                                                                         |
  | --------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
  | `role=gateway`              | Immer vorhanden.                                                                                                                                                |
  | `transport=gateway`         | Immer vorhanden.                                                                                                                                                |
  | `displayName=<name>`        | Vom Bediener konfigurierter Anzeigename.                                                                                                                        |
  | `lanHost=<hostname>.local`  | Nur beim LAN-mDNS-Advertiser; wird nicht durch Wide-Area-DNS-SD geschrieben.                                                                                     |
  | `gatewayPort=18789`         | Gateway-WS- und HTTP-Port.                                                                                                                                      |
  | `gatewayTls=1`              | Nur bei aktiviertem TLS.                                                                                                                                        |
  | `gatewayTlsSha256=<sha256>` | Nur bei aktiviertem TLS und verfügbarem Fingerabdruck.                                                                                                           |
  | `tailnetDns=<magicdns>`     | Optionaler Hinweis; wird automatisch erkannt, wenn Tailscale verfügbar ist.                                                                                      |
  | `sshPort=<port>`            | Nur vorhanden, wenn `discovery.mdns.mode="full"`; im standardmäßigen Modus `"minimal"` weggelassen (SSH verwendet standardmäßig `22`), sowohl beim LAN-Advertiser als auch bei Wide-Area-DNS-SD. |
  | `cliPath=<path>`            | Dieselbe `discovery.mdns.mode="full"`-Bedingung wie bei `sshPort`; ein Hinweis für Remote-Installationen zum CLI-Pfad.                                           |

  Ein TXT-Schlüssel `canvasPort` ist im Plugin-Discovery-Vertrag für einen
  zukünftigen Canvas-Host-Port definiert, aber kein aktueller Codepfad legt einen Wert fest, sodass er
  derzeit nie ausgegeben wird.

Sicherheitshinweise:

- Bonjour-/mDNS-TXT-Einträge sind **nicht authentifiziert**. Clients dürfen TXT-
  Werte nur als Hinweise für die Benutzeroberfläche behandeln.
- Beim Routing (Host/Port) sollte der **aufgelöste Service-Endpunkt**
  (SRV + A/AAAA) gegenüber den per TXT bereitgestellten Werten `lanHost`, `tailnetDns` oder `gatewayPort` bevorzugt werden.
- Beim TLS-Pinning darf ein angekündigter Wert `gatewayTlsSha256` niemals einen
  zuvor gespeicherten Pin überschreiben.
- iOS-/Android-Nodes sollten eine ausdrückliche Bestätigung zum Vertrauen dieses Fingerabdrucks
  verlangen, bevor ein erstmaliger Pin gespeichert wird (Verifizierung über einen separaten Kanal),
  wenn die ausgewählte Route sicher bzw. TLS-basiert ist.

Aktivieren, deaktivieren und überschreiben:

- `openclaw plugins enable bonjour` aktiviert die LAN-Multicast-Ankündigung.
- `discovery.mdns.mode` in `openclaw.json` steuert die mDNS-Übertragung:
  `"minimal"` (Standard), `"full"` (fügt `cliPath`/`sshPort` sowohl dem LAN-
  Beacon als auch allen Wide-Area-DNS-SD-Zonen hinzu) oder `"off"` (deaktiviert mDNS).
- `OPENCLAW_DISABLE_BONJOUR=1` erzwingt die Deaktivierung der Ankündigung; `discovery.mdns.mode="off"`
  deaktiviert sie unabhängig davon. `OPENCLAW_DISABLE_BONJOUR=0` ist eine ausdrückliche
  Zustimmung, welche die automatische Deaktivierung des Plugins innerhalb eines erkannten Containers
  (Docker, containerd, Kubernetes, LXC) außer Kraft setzt; sie überschreibt nicht
  `discovery.mdns.mode="off"`. Das gebündelte Plugin `bonjour` startet automatisch auf
  macOS-Hosts (`enabledByDefaultOnPlatforms: ["darwin"]`) und deaktiviert sich automatisch
  innerhalb erkannter Container; Linux, Windows und andere containerisierte
  Bereitstellungen müssen `plugins enable bonjour` ausdrücklich festlegen.
- `gateway.bind` in `~/.openclaw/openclaw.json` steuert den Bindungsmodus des Gateways.
- `OPENCLAW_SSH_PORT` überschreibt den angekündigten SSH-Port (wird nur wirksam,
  wenn `discovery.mdns.mode="full"`).
- `OPENCLAW_TAILNET_DNS` veröffentlicht einen Hinweis `tailnetDns` (MagicDNS).
- `OPENCLAW_CLI_PATH` überschreibt den angekündigten CLI-Pfad.

### 2) Tailnet (netzwerkübergreifend)

Bei Gateways in verschiedenen physischen Netzwerken hilft Bonjour nicht. Das
empfohlene direkte Ziel ist ein Tailscale-MagicDNS-Name (bevorzugt) oder eine
stabile Tailnet-IP.

Wenn der Gateway erkennt, dass er unter Tailscale ausgeführt wird, veröffentlicht er
`tailnetDns` als optionalen Hinweis für Clients (einschließlich Wide-Area-Beacons).
Die macOS-App bevorzugt bei der Gateway-
Discovery MagicDNS-Namen gegenüber unverarbeiteten Tailscale-IPs. Dies bleibt zuverlässig, wenn sich Tailnet-IPs ändern (Node-Neustarts,
erneute CGNAT-Zuweisung), da MagicDNS automatisch zur aktuellen IP auflöst.

Bei der Kopplung mobiler Nodes lockern Discovery-Hinweise niemals die Transportsicherheit auf
Tailnet-/öffentlichen Routen:

- iOS/Android benötigen weiterhin einen sicheren erstmaligen Verbindungsweg über das Tailnet/öffentliche Netzwerk
  (`wss://` oder Tailscale Serve/Funnel).
- Eine erkannte unverarbeitete Tailnet-IP ist ein Routing-Hinweis und keine Berechtigung zur Verwendung
  einer unverschlüsselten Remote-Verbindung über `ws://`.
- Die direkte private LAN-Verbindung über `ws://` wird weiterhin unterstützt.
- Für den einfachsten Tailscale-Verbindungsweg auf mobilen Nodes verwenden Sie Tailscale Serve, damit
  sowohl die Discovery als auch die Einrichtung denselben sicheren MagicDNS-Endpunkt verwenden.

### 3) Manuelles / SSH-Ziel

Wenn keine direkte Route vorhanden ist (oder Direktverbindungen deaktiviert sind), können Clients jederzeit
über SSH eine Verbindung herstellen, indem sie den Loopback-Port des Gateways weiterleiten. Siehe
[Remote-Zugriff](/de/gateway/remote).

## Transportauswahl (Client-Richtlinie)

1. Wenn ein gekoppelter direkter Endpunkt konfiguriert und erreichbar ist, verwenden Sie ihn.
2. Andernfalls, wenn die Discovery einen Gateway unter `local.` oder in der konfigurierten Wide-Area-
   Domain findet, bieten Sie eine Ein-Tipp-Auswahl zur Verwendung dieses Gateways an und speichern Sie ihn als
   direkten Endpunkt.
3. Andernfalls, wenn eine Tailnet-DNS-Adresse/IP konfiguriert ist, versuchen Sie eine direkte Verbindung. Bei mobilen Nodes auf
   Tailnet-/öffentlichen Routen bedeutet „direkt“ einen sicheren Endpunkt und keine unverschlüsselte
   Remote-Verbindung über `ws://`.
4. Andernfalls verwenden Sie SSH als Fallback.

## Kopplung und Authentifizierung (direkter Transport)

Der Gateway ist die maßgebliche Instanz für die Zulassung von Nodes/Clients:

- Kopplungsanfragen werden im Gateway erstellt, genehmigt oder abgelehnt (siehe
  [Gateway-Kopplung](/de/gateway/pairing)).
- Der Gateway erzwingt die Authentifizierung (Token/Schlüsselpaar), Geltungsbereiche/ACLs (er ist kein unverarbeiteter
  Proxy für jede Methode) und Ratenbegrenzungen.

## Zuständigkeiten nach Komponente

- **Gateway**: kündigt Discovery-Beacons an, verwaltet Kopplungsentscheidungen und stellt
  den WS-Endpunkt bereit.
- **macOS-App**: unterstützt Sie bei der Auswahl eines Gateways, zeigt Kopplungsaufforderungen an und verwendet SSH
  nur als Fallback.
- **iOS-/Android-Nodes**: durchsuchen Bonjour zur Vereinfachung und stellen eine Verbindung zum
  gekoppelten Gateway-WS her.

## Verwandte Themen

- [Remote-Zugriff](/de/gateway/remote)
- [Tailscale](/de/gateway/tailscale)
- [Bonjour-Discovery](/de/gateway/bonjour)
