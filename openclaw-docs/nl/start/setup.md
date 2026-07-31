---
read_when:
    - Een nieuwe machine instellen
    - Je wilt het nieuwste van het nieuwste zonder je persoonlijke configuratie te verstoren
summary: Geavanceerde installatie- en ontwikkelworkflows voor OpenClaw
title: Instellen
x-i18n:
    generated_at: "2026-07-27T05:23:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c40d6d2bf2814465f3cc49c65d4c1498671420af728ce8012d13af3fba67025a
    source_path: start/setup.md
    workflow: 16
---

<Note>
Als je dit voor het eerst instelt, begin dan met [Aan de slag](/nl/start/getting-started).
Zie [Onboarding (CLI)](/nl/start/wizard) voor meer informatie over de onboarding.
</Note>

## Kort samengevat

Kies een installatieworkflow op basis van hoe vaak je updates wilt en of je de Gateway zelf wilt uitvoeren:

- **Aanpassingen blijven buiten de repository:** bewaar je configuratie en werkruimte in `~/.openclaw/openclaw.json` en `~/.openclaw/workspace/`, zodat repository-updates deze niet wijzigen.
- **Stabiele workflow (aanbevolen voor de meesten):** installeer de macOS-app en laat deze de meegeleverde Gateway uitvoeren.
- **Workflow met de allernieuwste ontwikkelversie (dev):** voer de Gateway zelf uit via `pnpm gateway:watch` en laat de macOS-app vervolgens verbinding maken in de modus Lokaal.

## Vereisten (vanuit de broncode)

- Node 24.15+ aanbevolen (Node 22 LTS, momenteel `22.22.3+`, wordt nog ondersteund)
- `pnpm` is vereist voor broncodecheck-outs. OpenClaw laadt meegeleverde plugins vanuit de
  `extensions/*` pnpm-werkruimtepakketten in de ontwikkelmodus, waardoor `npm install` in de hoofdmap
  niet de volledige broncodestructuur voorbereidt.
- Docker (optioneel; alleen voor installatie in containers/e2e — zie [Docker](/nl/install/docker))

## Aanpassingsstrategie (zodat updates geen problemen veroorzaken)

Als je een installatie wilt die "100% op mij is afgestemd" _en_ eenvoudig kan worden bijgewerkt, bewaar je aanpassingen dan in:

- **Configuratie:** `~/.openclaw/openclaw.json` (JSON/ongeveer JSON5)
- **Werkruimte:** `~/.openclaw/workspace` (skills, prompts, geheugens; maak hiervan een privérepository in git)

Initialiseer de configuratie- en werkruimtemappen eenmalig zonder de volledige onboardingwizard uit te voeren:

```bash
openclaw setup --baseline
```

Nog geen globale installatie? Voer de opdracht dan vanuit deze repository uit:

```bash
pnpm openclaw setup --baseline
```

(Alleen `openclaw setup`, zonder `--baseline`, is een alias voor `openclaw onboard` en voert de volledige interactieve wizard uit.)

## De Gateway vanuit deze repository uitvoeren

Na `pnpm build` kun je de verpakte CLI rechtstreeks uitvoeren:

```bash
node openclaw.mjs gateway --port 18789 --verbose
```

## Stabiele workflow (eerst de macOS-app)

1. Installeer en start **OpenClaw.app** (menubalk).
2. Voltooi de controlelijst voor onboarding en machtigingen (TCC-prompts).
3. Zorg dat de Gateway op **Lokaal** staat en actief is (de app beheert deze).
4. Koppel kanalen (bijvoorbeeld WhatsApp):

```bash
openclaw channels login
```

5. Snelle controle:

```bash
openclaw health
```

Als onboarding niet beschikbaar is in jouw build:

- Voer `openclaw setup` uit, daarna `openclaw channels login`, en start vervolgens de Gateway handmatig (`openclaw gateway`).

## Workflow met de allernieuwste ontwikkelversie (Gateway in een terminal)

Doel: aan de TypeScript-Gateway werken, automatisch herladen gebruiken en de gebruikersinterface van de macOS-app verbonden houden.

### 0) (Optioneel) Voer ook de macOS-app vanuit de broncode uit

Als je ook voor de macOS-app de allernieuwste ontwikkelversie wilt gebruiken:

```bash
./scripts/restart-mac.sh
```

### 1) Start de ontwikkel-Gateway

```bash
pnpm install
# Alleen bij de eerste uitvoering (of na het opnieuw instellen van de lokale OpenClaw-configuratie/-werkruimte)
pnpm openclaw setup
pnpm gateway:watch
```

`gateway:watch` start of herstart het controleproces van de Gateway in een benoemde tmux-
sessie (`openclaw-gateway-watch-main`) en maakt vanuit interactieve
terminals automatisch verbinding. Niet-interactieve shells blijven losgekoppeld en tonen
`tmux attach -t openclaw-gateway-watch-main`; gebruik
`OPENCLAW_GATEWAY_WATCH_ATTACH=0 pnpm gateway:watch` om een interactieve uitvoering
losgekoppeld te houden, of `pnpm gateway:watch:raw` voor de controlemodus op de voorgrond. Het controleproces
stopt de geïnstalleerde Gateway-service van het actieve profiel voordat het de
geconfigureerde/standaardpoort overneemt, zodat het servicebeheer het
broncodeproces niet vervangt. De service blijft geïnstalleerd; voer `pnpm openclaw gateway start` uit
wanneer je klaar bent met controleren. Het tmux-deelvenster blijft na een opstartfout beschikbaar,
zodat een andere terminal of agent verbinding kan maken of de logboeken kan vastleggen. Het controleproces
herlaadt bij relevante wijzigingen in de broncode, configuratie en metadata van meegeleverde plugins. Als de
gecontroleerde Gateway tijdens het opstarten wordt afgesloten, voert `gateway:watch`
eenmaal `openclaw doctor --fix --non-interactive` uit en probeert het opnieuw; stel
`OPENCLAW_GATEWAY_WATCH_AUTO_DOCTOR=0` in om die uitsluitend voor ontwikkeling bedoelde herstelpoging uit te schakelen.
`pnpm gateway:watch` bouwt `dist/control-ui` niet opnieuw, dus voer `pnpm ui:build` opnieuw uit na wijzigingen aan `ui/` of gebruik `pnpm ui:dev` tijdens het ontwikkelen van de Control UI.

### 2) Laat de macOS-app verbinding maken met je actieve Gateway

In **OpenClaw.app**:

- Verbindingsmodus: **Lokaal**
  De app maakt verbinding met de actieve Gateway op de geconfigureerde poort.

### 3) Verifiëren

- De Gateway-status in de app moet **"Bestaande Gateway wordt gebruikt …"** aangeven
- Of via de CLI:

```bash
openclaw health
```

### Veelvoorkomende valkuilen

- **Verkeerde poort:** Gateway WS gebruikt standaard `ws://127.0.0.1:18789`; gebruik voor de app en CLI dezelfde poort.
- **Waar de status wordt opgeslagen:**
  - Kanaal-/providerstatus: `~/.openclaw/credentials/`
  - Modelauthenticatieprofielen: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
  - Sessies en transcripties: `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`
  - Verouderde/gearchiveerde sessieartefacten: `~/.openclaw/agents/<agentId>/sessions/`
  - Logboeken: `/tmp/openclaw/`

## Overzicht van de opslag van aanmeldgegevens

Gebruik dit bij het oplossen van authenticatieproblemen of om te bepalen waarvan je een back-up moet maken:

- **WhatsApp**: `~/.openclaw/credentials/whatsapp/<accountId>/creds.json`
- **Telegram-bottoken**: configuratie/omgevingsvariabele of `channels.telegram.tokenFile` (alleen een normaal bestand; symbolische koppelingen worden geweigerd)
- **Discord-bottoken**: configuratie/omgevingsvariabele of SecretRef (providers voor omgevingsvariabele/bestand/uitvoering)
- **Slack-tokens**: configuratie/omgevingsvariabele (`channels.slack.*`)
- **Toelatingslijsten voor koppeling**:
  - `~/.openclaw/credentials/<channel>-allowFrom.json` (standaardaccount)
  - `~/.openclaw/credentials/<channel>-<accountId>-allowFrom.json` (niet-standaardaccounts)
- **Modelauthenticatieprofielen**: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
- **Bestandsgebaseerde geheime payload (optioneel)**: `~/.openclaw/secrets.json`
- **Verouderde OAuth-import**: `~/.openclaw/credentials/oauth.json`
  Meer informatie: [Beveiliging](/nl/gateway/security#credential-storage-map).

## Bijwerken (zonder je installatie te beschadigen)

- Beschouw `~/.openclaw/workspace` en `~/.openclaw/` als "jouw eigen bestanden"; plaats geen persoonlijke prompts/configuratie in de repository `openclaw`.
- De broncode bijwerken: `git pull` + `pnpm install` + blijf `pnpm gateway:watch` gebruiken.

## Linux (systemd-gebruikersservice)

Linux-installaties gebruiken een systemd-**gebruikersservice**. Standaard stopt systemd gebruikersservices
bij afmelden/inactiviteit, waardoor de Gateway wordt beëindigd. Onboarding probeert
lingering voor je in te schakelen (mogelijk wordt om sudo gevraagd). Als het nog steeds uitgeschakeld is, voer je dit uit:

```bash
sudo loginctl enable-linger $USER
```

Overweeg voor altijd actieve servers of servers met meerdere gebruikers een **systeemservice** in plaats van een
gebruikersservice (lingering is dan niet nodig). Zie het [Gateway-draaiboek](/nl/gateway) voor de systemd-opmerkingen.

## Gerelateerde documentatie

- [Gateway-draaiboek](/nl/gateway) (vlaggen, beheer, poorten)
- [Gateway-configuratie](/nl/gateway/configuration) (configuratieschema + voorbeelden)
- [Discord](/nl/channels/discord) en [Telegram](/nl/channels/telegram) (antwoordtags + replyToMode-instellingen)
- [OpenClaw-assistent instellen](/nl/start/openclaw)
- [macOS-app](/nl/platforms/macos) (levenscyclus van de Gateway)
