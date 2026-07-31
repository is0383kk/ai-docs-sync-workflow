---
read_when:
    - OpenClaw.app verpakken
    - Foutopsporing voor de macOS Gateway-launchd-service
    - De Gateway-CLI voor macOS installeren
summary: Gateway-runtime op macOS (externe launchd-service)
title: Gateway op macOS
x-i18n:
    generated_at: "2026-07-27T05:38:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 30c1ae14d8f8eaab73d0e2b725292d7411c2c8b5e0e0c32ad13989c01340d054
    source_path: platforms/mac/bundled-gateway.md
    workflow: 16
---

OpenClaw.app bevat geen ingebouwde Node- of Gateway-runtime. De macOS-app
verwacht een **externe** `openclaw` CLI-installatie, start de Gateway niet als
een onderliggend proces en beheert een launchd-service per gebruiker om de Gateway
actief te houden (of maakt verbinding met een al actieve lokale Gateway).

## Automatische configuratie

Kies tijdens de onboarding op een nieuwe Mac **Deze Mac**. De app voert vóór de
Gateway-wizard het ondertekende, meegeleverde installatiescript uit: het installeert een
Node-runtime in de gebruikersomgeving en de bijbehorende `openclaw` CLI onder `~/.openclaw`,
en installeert en start vervolgens de launchd-service per gebruiker. Voor dit traject zijn geen
Terminal, Homebrew of beheerderstoegang nodig.

De app bevat alleen het installatiescript, niet de Node- of Gateway-payload;
voor de configuratie is een internetverbinding nodig om de runtime en het bijbehorende
OpenClaw-pakket te downloaden.

## Handmatig herstel

Node 24.15+ wordt aanbevolen voor een handmatige installatie; Node 22.22.3+ werkt ook. Installeer
`openclaw` globaal:

```bash
npm install -g openclaw@<version>
```

Gebruik **Configuratie opnieuw proberen** nadat de automatische configuratie is mislukt. Als dat nog steeds mislukt,
installeer je de CLI handmatig met de bovenstaande opdracht en kies je vervolgens **Opnieuw controleren**
tijdens de onboarding.

## Launchd (Gateway als LaunchAgent)

Label: `ai.openclaw.gateway` (standaardprofiel), of `ai.openclaw.<profile>`
voor een benoemd profiel.

Plist-locatie (per gebruiker): `~/Library/LaunchAgents/ai.openclaw.gateway.plist`
(of `ai.openclaw.<profile>.plist`).

De macOS-app beheert de installatie en updates van de LaunchAgent voor het standaardprofiel in
de lokale modus. De CLI kan deze ook rechtstreeks installeren: `openclaw gateway install`
(benoemde profielen worden geselecteerd via de omgevingsvariabele `OPENCLAW_PROFILE`).

Gedrag:

- "OpenClaw actief" schakelt de LaunchAgent in of uit.
- Het afsluiten van de app stopt de Gateway **niet** (launchd houdt deze actief).
- Als er al een Gateway actief is op de geconfigureerde poort, maakt de app daar
  verbinding mee in plaats van een nieuwe te starten.

Logboekregistratie:

- stdout van launchd: `~/Library/Logs/openclaw/gateway.log` (profielen gebruiken
  `gateway-<profile>.log`)
- stderr van launchd: onderdrukt
- Als de host in een lus terechtkomt met herhaalde `EADDRINUSE` of snelle herstarts, controleer dan op
  dubbele `ai.openclaw.gateway`- / `ai.openclaw.node`-LaunchAgents en de
  tijdelijke oplossing met de launchd-markering in
  [Problemen met de Gateway oplossen](/nl/gateway/troubleshooting#macos-launchd-supervisor-loop-with-duplicate-gatewaynode-launchagents).

## Versiecompatibiliteit

De macOS-app controleert de Gateway-versie aan de hand van de eigen versie. Tijdens de onboarding
wordt automatisch de beheerde configuratie uitgevoerd wanneer een bestaande CLI ontbreekt of
incompatibel is. Gebruik **Configuratie opnieuw proberen** om de installatie te herhalen, of **Opnieuw controleren**
nadat je een externe CLI hebt hersteld.

## Statusmap op macOS

Bewaar de OpenClaw-status op een lokale, niet-gesynchroniseerde schijf. Vermijd iCloud Drive en andere
met de cloud gesynchroniseerde mappen; synchronisatievertragingen en bestandsvergrendelingen kunnen gevolgen hebben voor sessies,
inloggegevens en de Gateway-status.

Stel `OPENCLAW_STATE_DIR` alleen in op een lokaal pad wanneer je een aangepaste waarde nodig hebt.
`openclaw doctor` waarschuwt voor veelvoorkomende met de cloud gesynchroniseerde statuspaden en raadt aan
terug te gaan naar lokale opslag. Zie
[omgevingsvariabelen](/nl/help/environment#path-related-env-vars) en
[Doctor](/nl/gateway/doctor).

## Problemen met app-connectiviteit opsporen

Gebruik de macOS-foutopsporings-CLI vanuit een broncode-checkout om dezelfde Gateway-
WebSocket-handshake en detectielogica uit te voeren die de app gebruikt:

```bash
cd apps/macos
swift run openclaw-mac connect --json
swift run openclaw-mac discover --timeout 3000 --json
```

`connect` accepteert `--url`, `--token`, `--timeout`, `--probe` en `--json`
(plus overschrijvingen voor de clientidentiteit; voer uit met `--help` voor de volledige lijst).
`discover` accepteert `--timeout`, `--json` en `--include-local`. Vergelijk
de detectie-uitvoer met `openclaw gateway discover --json` wanneer je
CLI-detectie wilt onderscheiden van verbindingsproblemen aan de appzijde.

## Snelle controle

```bash
openclaw --version

OPENCLAW_SKIP_CHANNELS=1 \
OPENCLAW_SKIP_CANVAS_HOST=1 \
openclaw gateway --port 18999 --bind loopback
```

Vervolgens:

```bash
openclaw gateway call health --url ws://127.0.0.1:18999 --timeout 3000
```

## Gerelateerd

- [macOS-app](/nl/platforms/macos)
- [Gateway-draaiboek](/nl/gateway)
