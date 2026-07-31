---
read_when:
    - PeekabooBridge hosten in OpenClaw.app
    - Peekaboo integreren via Swift Package Manager
    - PeekabooBridge-protocol/-paden wijzigen
    - Kiezen tussen PeekabooBridge, Codex Computer Use en cua-driver MCP
summary: PeekabooBridge-integratie voor macOS-UI-automatisering
title: Peekaboo-bridge
x-i18n:
    generated_at: "2026-07-27T05:55:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 24d4187b2f5c5f11f44a24e25b350adaa3b068f24dce640ec695d52eb61f8e9a
    source_path: platforms/mac/peekaboo.md
    workflow: 16
---

OpenClaw kan **PeekabooBridge** hosten als een lokale, machtigingsbewuste broker voor UI-automatisering (`PeekabooBridgeHostCoordinator`, ondersteund door het Swift-pakket `steipete/Peekaboo`). Hierdoor kan de CLI `peekaboo` UI-automatisering aansturen en daarbij de TCC-machtigingen van de macOS-app hergebruiken.

## Wat dit is (en niet is)

- **Host**: OpenClaw.app kan als PeekabooBridge-host fungeren.
- **Client**: de CLI `peekaboo` (er is geen afzonderlijk `openclaw ui ...`-oppervlak).
- **UI**: visuele overlays blijven in Peekaboo.app; OpenClaw is een lichtgewicht brokerhost.

## Relatie tot andere manieren om het bureaublad te bedienen

OpenClaw heeft vier manieren om het bureaublad te bedienen die bewust gescheiden blijven:

- **PeekabooBridge-host**: OpenClaw.app host de lokale PeekabooBridge-socket. De CLI `peekaboo` is de client en gebruikt de macOS-machtigingen van OpenClaw.app voor schermafbeeldingen, klikken, menu's, dialoogvensters, Dock-acties en vensterbeheer.
- **Computeraansturing door een agent (`computer.act`)**: het ingebouwde hulpmiddel `computer` van de Gateway-agent maakt schermafbeeldingen via `screen.snapshot` en bestuurt de aanwijzer en het toetsenbord met de gevaarlijke Node-opdracht `computer.act`. Een macOS-Node voert `computer.act` in hetzelfde proces uit met de ingebedde Peekaboo-automatiseringsservices die deze bridge beschikbaar stelt, plus beperkte CoreGraphics-primitieven, zonder de PeekabooBridge-socket of de CLI `peekaboo` te gebruiken. Zie [Computergebruik](/nl/nodes/computer-use).
- **Codex Computer Use**: de meegeleverde Plugin `codex` controleert de MCP-plugin `computer-use` van Codex (`extensions/codex/src/app-server/computer-use.ts`) en kan deze installeren. Vervolgens kan Codex tijdens beurten in Codex-modus zelf aanroepen van hulpmiddelen voor systeemeigen bureaubladbediening beheren. OpenClaw stuurt deze acties niet via PeekabooBridge door.
- **Rechtstreekse `cua-driver`-MCP**: OpenClaw kan de upstream-server `cua-driver mcp` van TryCua als een normale MCP-server registreren. Hierdoor krijgen agents de eigen schema's en workflow voor pid/venster/elementindex van het CUA-stuurprogramma, zonder routering via de Codex-marktplaats of de PeekabooBridge-socket.

Gebruik Peekaboo voor uitgebreide macOS-automatisering via de machtigingsbewuste bridgehost van OpenClaw.app. Gebruik computeraansturing door een agent wanneer de Gateway-agent het bureaublad moet kunnen zien en bedienen via een uniforme Node-opdracht `computer.act` die elk visionmodel kan aansturen. Gebruik Codex Computer Use wanneer een agent in Codex-modus moet vertrouwen op de systeemeigen Plugin van Codex. Gebruik rechtstreeks `cua-driver mcp` om het CUA-stuurprogramma als een normale MCP-server beschikbaar te stellen aan elke door OpenClaw beheerde runtime.

## De bridge inschakelen

In de macOS-app: **Settings -> Enable Peekaboo Bridge**. Voor deze schakelaar moet **Allow Computer Control** zijn ingeschakeld, omdat beide lokale UI-automatisering toestaan. Als Computer Control is uitgeschakeld, is de schakelaar niet beschikbaar en wordt de host niet uitgevoerd. Als je Peekaboo zonder Computer Control wilt aansturen, voer je in plaats daarvan de eigen Mac-app van Peekaboo uit als host.

Wanneer deze optie is ingeschakeld (en Computer Control aanstaat), start OpenClaw een lokale UNIX-socketserver op `~/Library/Application Support/OpenClaw/<socket-name>`. Als de optie is uitgeschakeld, stopt de host en valt `peekaboo` terug op andere beschikbare hosts. De coördinator onderhoudt ook verouderde socketsymbolische koppelingen (`clawdbot`, `clawdis`, `moltbot` onder Application Support) die naar de huidige socket verwijzen voor oudere installaties van `peekaboo`.

## Detectievolgorde voor clients

Peekaboo-clients proberen hosts doorgaans in deze volgorde:

1. Peekaboo.app (volledige gebruikerservaring)
2. Claude.app (indien geïnstalleerd)
3. OpenClaw.app (lichtgewicht broker)

Gebruik `peekaboo bridge status --verbose` om te zien welke host actief is en welk socketpad wordt gebruikt. Overschrijf dit met:

```bash
export PEEKABOO_BRIDGE_SOCKET=/path/to/bridge.sock
```

## Beveiliging en machtigingen

- De bridge valideert **codehandtekeningen van aanroepers**; er wordt een toelatingslijst met TeamID's afgedwongen (de TeamID van de Peekaboo-host plus de eigen TeamID van de actieve app).
- Geef voor Toegankelijkheid de voorkeur aan de ondertekende identiteit van de bridge/app boven een generieke `node`-runtime. Als je Toegankelijkheid toekent aan `node`, kan elk pakket dat door dat uitvoerbare Node-bestand wordt gestart toegang tot GUI-automatisering overnemen; zie [macOS-machtigingen](/nl/platforms/mac/permissions#accessibility-grants-for-node-and-cli-runtimes).
- Aanvragen verlopen na 10 seconden (`requestTimeoutSec: 10`).
- Als vereiste machtigingen ontbreken, retourneert de bridge een duidelijke foutmelding in plaats van Systeeminstellingen te openen.

## Gedrag van momentopnamen (automatisering)

Momentopnamen worden in het geheugen opgeslagen met een geldigheidsduur van 10 minuten en een maximum van 50 momentopnamen (`InMemorySnapshotManager`); artefacten worden tijdens het opschonen niet verwijderd. Als je ze langer wilt bewaren, maak je vanuit de client opnieuw een opname.

## Problemen oplossen

- Als `peekaboo` meldt dat "de bridgeclient niet is geautoriseerd", controleer je of de client correct is ondertekend of voer je de host uitsluitend in de **debug**-modus uit met `PEEKABOO_ALLOW_UNSIGNED_SOCKET_CLIENTS=1`.
- Als er geen hosts worden gevonden, open je een van de host-apps (Peekaboo.app of OpenClaw.app) en controleer je of de machtigingen zijn verleend.

## Gerelateerd

- [macOS-app](/nl/platforms/macos)
- [macOS-machtigingen](/nl/platforms/mac/permissions)
