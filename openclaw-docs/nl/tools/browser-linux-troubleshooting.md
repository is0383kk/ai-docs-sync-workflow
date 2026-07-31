---
read_when: Browser control fails on Linux, especially with snap Chromium
summary: Problemen met het starten van CDP in Chrome/Brave/Edge/Chromium voor OpenClaw-browserbesturing op Linux oplossen
title: Problemen met de browser oplossen
x-i18n:
    generated_at: "2026-07-27T06:35:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8e5db2da2d43129862f0c005213df828f6eae81f5561e57d41795ea90787822a
    source_path: tools/browser-linux-troubleshooting.md
    workflow: 16
---

## Probleem: Chrome CDP kon niet worden gestart op poort 18800

```json
{ "error": "Fout: Chrome CDP kon niet worden gestart op poort 18800 voor profiel \"openclaw\"." }
```

### Hoofdoorzaak

Op Ubuntu en de meeste Linux-distributies installeert `apt install chromium` een snap-wrapper,
geen echte browser:

```text
Let op: 'chromium-browser' wordt geselecteerd in plaats van 'chromium'
chromium-browser is al de nieuwste versie (2:1snap1-0ubuntu2).
```

De AppArmor-isolatie van Snap verstoort de manier waarop OpenClaw het
browserproces start en bewaakt.

Andere veelvoorkomende startfouten op Linux:

- `The profile appears to be in use by another Chromium process`: verouderde
  `Singleton*`-vergrendelingsbestanden in de beheerde profielmap. OpenClaw verwijdert
  deze vergrendelingen en probeert het eenmaal opnieuw wanneer de vergrendeling verwijst naar een beëindigd proces of
  een proces op een andere host.
- `Missing X server or $DISPLAY`: er is expliciet om een zichtbare browser gevraagd
  op een host zonder desktopsessie. Lokale beheerde profielen schakelen op Linux terug naar
  headless-modus wanneer zowel `DISPLAY` als `WAYLAND_DISPLAY` niet zijn ingesteld.
  Als je `OPENCLAW_BROWSER_HEADLESS=0`, `browser.headless: false` of
  `browser.profiles.<name>.headless: false` instelt, verwijder dan die override voor de zichtbare modus, stel
  `OPENCLAW_BROWSER_HEADLESS=1` in, start `Xvfb`, voer
  `openclaw browser start --headless` uit voor een eenmalige beheerde start, of voer
  OpenClaw uit in een echte desktopsessie.

### Oplossing 1: Google Chrome installeren (aanbevolen)

```bash
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
sudo dpkg -i google-chrome-stable_current_amd64.deb
sudo apt --fix-broken install -y  # als er afhankelijkheidsfouten zijn
```

Werk `~/.openclaw/openclaw.json` bij:

```json
{
  "browser": {
    "enabled": true,
    "executablePath": "/usr/bin/google-chrome-stable",
    "headless": true,
    "noSandbox": true
  }
}
```

### Oplossing 2: snap Chromium in de modus voor alleen koppelen gebruiken

Als je snap Chromium moet behouden, configureer OpenClaw dan om verbinding te maken met een
handmatig gestarte browser in plaats van deze zelf te starten:

```json
{
  "browser": {
    "enabled": true,
    "attachOnly": true,
    "headless": true,
    "noSandbox": true
  }
}
```

Start Chromium handmatig:

```bash
chromium-browser --headless --no-sandbox --disable-gpu \
  --remote-debugging-port=18800 \
  --user-data-dir=$HOME/.openclaw/browser/openclaw/user-data \
  about:blank &
```

Je kunt deze desgewenst automatisch laten starten met een systemd-gebruikersservice:

```ini
# ~/.config/systemd/user/openclaw-browser.service
[Unit]
Description=OpenClaw-browser (Chrome CDP)
After=network.target

[Service]
ExecStart=/snap/bin/chromium --headless --no-sandbox --disable-gpu --remote-debugging-port=18800 --user-data-dir=%h/.openclaw/browser/openclaw/user-data about:blank
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

```bash
systemctl --user enable --now openclaw-browser.service
```

### Controleren of de browser werkt

```bash
curl -s http://127.0.0.1:18791/ | jq '{running, pid, chosenBrowser}'
curl -s -X POST http://127.0.0.1:18791/start
curl -s http://127.0.0.1:18791/tabs
```

### Configuratiereferentie

| Optie                       | Beschrijving                                                          | Standaardwaarde                                                            |
| --------------------------- | -------------------------------------------------------------------- | ------------------------------------------------------------------ |
| `browser.enabled`           | Browserbesturing inschakelen                                               | `true`                                                             |
| `browser.executablePath`    | Pad naar een browserbinair bestand op basis van Chromium (Chrome/Brave/Edge/Chromium) | automatisch gedetecteerd (geeft de voorkeur aan de standaardbrowser van het besturingssysteem als die op Chromium is gebaseerd) |
| `browser.headless`          | Zonder GUI uitvoeren                                                      | `false`                                                            |
| `OPENCLAW_BROWSER_HEADLESS` | Override per proces voor de headless-modus van de lokale beheerde browser         | niet ingesteld                                                              |
| `browser.noSandbox`         | De vlag `--no-sandbox` toevoegen (vereist voor sommige Linux-configuraties)               | `false`                                                            |
| `browser.attachOnly`        | Geen browser starten; alleen verbinding maken met een bestaande browser              | `false`                                                            |

Gebruik op Raspberry Pi, oudere VPS-hosts of trage opslag een handmatig gestarte
browser met `attachOnly` wanneer Chrome meer tijd nodig heeft om het CDP HTTP-
eindpunt beschikbaar te maken of gereed te worden dan de deadline van de beheerde browser toestaat.

### Probleem: geen Chrome-tabbladen gevonden voor profile="user"

Je gebruikt het profiel `user` (`existing-session` / Chrome MCP) en er zijn geen
tabbladen geopend om verbinding mee te maken.

Mogelijke oplossingen:

1. Gebruik in plaats daarvan de beheerde browser:
   `openclaw browser --browser-profile openclaw start` (of stel
   `browser.defaultProfile: "openclaw"` in).
2. Houd lokale Chrome actief met ten minste één geopend tabblad en probeer het vervolgens opnieuw met
   `--browser-profile user`.

Opmerkingen:

- `user` werkt alleen op de host. Geef op Linux-servers, in containers of op externe hosts de voorkeur aan
  CDP-profielen.
- `user` en andere `existing-session`-profielen hebben dezelfde huidige beperkingen van Chrome MCP:
  alleen acties op basis van verwijzingen, één bestand per upload, geen overrides voor dialoogvensters via `timeoutMs`,
  geen `wait --load networkidle` en geen `responsebody`, PDF-export,
  onderschepping van downloads of batchacties.
- Lokale profielen met het `openclaw`-stuurprogramma wijzen `cdpPort`/`cdpUrl` automatisch toe; stel
  deze alleen handmatig in voor externe CDP.
- Externe CDP-profielen accepteren `http://`, `https://`, `ws://` en `wss://`.
  Gebruik HTTP(S) voor `/json/version`-detectie, of WS(S) wanneer je browserservice
  je een directe URL voor een DevTools-socket geeft.

## Gerelateerd

- [Browser](/nl/tools/browser)
- [Browseraanmelding](/nl/tools/browser-login)
- [Probleemoplossing voor Browser met externe CDP via WSL2](/nl/tools/browser-wsl2-windows-remote-cdp-troubleshooting)
