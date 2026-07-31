---
read_when:
    - OpenClaw Gateway uitvoeren in WSL2 terwijl Chrome op Windows draait
    - Overlappende browser-/control-ui-fouten in WSL2 en Windows zien
    - Kiezen tussen hostlokale Chrome MCP en onbewerkte externe CDP in configuraties met gescheiden hosts
summary: Problemen met WSL2 Gateway + externe CDP van Windows Chrome stapsgewijs oplossen
title: Probleemoplossing voor WSL2 + Windows + externe Chrome CDP
x-i18n:
    generated_at: "2026-07-27T05:24:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 66ec4ed5bfccc66b594a43d56296c69242e8b9cf50b36c6cb3990b1d6ea58faa
    source_path: tools/browser-wsl2-windows-remote-cdp-troubleshooting.md
    workflow: 16
---

In de gebruikelijke configuratie met gescheiden hosts draait OpenClaw Gateway binnen WSL2, draait Chrome
op Windows en moet browserbesturing de WSL2/Windows-grens overschrijden. Meerdere
onafhankelijke problemen kunnen tegelijk optreden (zie
[issue #39369](https://github.com/openclaw/openclaw/issues/39369)): CDP-
transport, oorsprongsbeveiliging van de Control UI en token/koppeling kunnen elk
afzonderlijk mislukken en daarbij vergelijkbare fouten produceren. Doorloop de onderstaande
lagen op volgorde in plaats van te raden welke defect is.

## Kies eerst de juiste browsermodus

### Optie 1: rechtstreekse externe CDP van WSL2 naar Windows

Gebruik een extern browserprofiel dat vanuit WSL2 naar een CDP-
eindpunt van Windows Chrome verwijst. Kies dit wanneer de Gateway binnen WSL2 blijft, Chrome op
Windows draait en browserbesturing de WSL2/Windows-grens moet overschrijden.

### Optie 2: hostlokale Chrome MCP

Gebruik het `existing-session`-stuurprogramma (`user`-profiel) alleen wanneer de Gateway
op dezelfde host als Chrome draait, je de lokale aangemelde browserstatus wilt gebruiken, je
geen browsertransport tussen hosts nodig hebt en je geen `responsebody`,
PDF-export, onderschepping van downloads of batchacties nodig hebt (Chrome MCP-profielen
ondersteunen deze niet).

Gebruik voor WSL2 Gateway + Windows Chrome rechtstreekse externe CDP. Chrome MCP is
hostlokaal, geen brug van WSL2 naar Windows.

## Werkende architectuur

- WSL2 voert de Gateway uit op `127.0.0.1:18789`
- Windows opent de Control UI in een normale browser op `http://127.0.0.1:18789/`
- Windows Chrome stelt een CDP-eindpunt beschikbaar op poort `9222`
- WSL2 kan dat Windows-CDP-eindpunt bereiken
- OpenClaw laat een browserprofiel verwijzen naar het vanuit WSL2 bereikbare adres

## Cruciale regel voor de Control UI

Wanneer de UI vanuit Windows wordt geopend, gebruik je Windows-localhost, tenzij je bewust
HTTPS hebt ingesteld:

```text
http://127.0.0.1:18789/
```

Gebruik niet standaard een LAN-IP. Gewone HTTP op een LAN- of tailnet-adres kan
gedrag rond een onveilige oorsprong/apparaatauthenticatie veroorzaken dat losstaat van CDP zelf. Zie
[Control UI](/nl/web/control-ui).

## Valideer in lagen

Werk van boven naar beneden; sla niets over. Nadat één laag is hersteld, kan nog steeds een
andere fout uit een lagere laag zichtbaar zijn.

### Laag 1: controleer of Chrome CDP aanbiedt op Windows

```powershell
chrome.exe --remote-debugging-port=9222 --user-data-dir="$env:LOCALAPPDATA\OpenClaw\ChromeCDP"
```

Chrome 136 en hoger negeren opdrachtregelopties voor externe foutopsporing voor de
standaardgegevensmap van Chrome. Gebruik een afzonderlijke, niet-standaardgegevensmap, zoals
hierboven weergegeven. Zie de
[beveiligingswijziging voor externe foutopsporing](https://developer.chrome.com/blog/remote-debugging-port)
van Chrome.
Hierdoor wordt het normale aangemelde Chrome-profiel niet extern bestuurbaar.

Controleer vanuit Windows eerst Chrome zelf:

```powershell
curl.exe http://127.0.0.1:9222/json/version
curl.exe http://127.0.0.1:9222/json/list
```

Als dit mislukt, onderzoek je de onderstaande Windows-listeners. OpenClaw is nog niet het
probleem.

#### Onderzoek IPv4 en IPv6 voordat je portproxy wijzigt

Chromium probeert externe foutopsporing eerst aan `127.0.0.1` te binden en valt alleen terug op
`[::1]` als de IPv4-binding mislukt. Een permanente `v4tov4`-regel die luistert op
`127.0.0.1:9222` kan dat eindpunt bezetten voordat Chrome start. Chrome valt vervolgens
terug op `[::1]:9222`, terwijl de oude regel IPv4-verkeer terugstuurt naar
zijn eigen listener en een leeg antwoord retourneert.

Controleer vanuit Windows de werkelijke listeners en proxyregels in plaats van ze
uit de Chrome-versie af te leiden:

```powershell
netstat -ano | findstr :9222
netsh interface portproxy show all
curl.exe http://127.0.0.1:9222/json/version
curl.exe http://[::1]:9222/json/version
```

Gebruik `tasklist /fi "PID eq <PID>"` voor elke PID uit `netstat`.

- Als `chrome.exe` antwoordt op `127.0.0.1`, verwijder je elke portproxy-regel die ook
  luistert op `127.0.0.1:9222`. Stuur alleen het vanuit WSL2 bereikbare Windows-adapteradres
  door naar `127.0.0.1`.
- Als `chrome.exe` alleen antwoordt op `[::1]`, laat je de vanuit WSL2 bereikbare listener
  met `v4tov6` naar `::1` verwijzen in plaats van door te sturen naar een ongebruikt IPv4-adres:

  ```powershell
  netsh interface portproxy add v4tov6 listenaddress=WINDOWS_HOST_OR_IP listenport=9222 connectaddress=::1 connectport=9222
  ```

Bind de listener aan het adapteradres dat WSL2 nodig heeft. Stel de CDP-
poort niet beschikbaar op `0.0.0.0`, een LAN-adres of een tailnet-adres: CDP verleent controle over
de browsersessie.

### Laag 2: controleer of WSL2 dat Windows-eindpunt kan bereiken

Test vanuit WSL2 het exacte adres dat je in `cdpUrl` wilt gebruiken:

```bash
curl http://WINDOWS_HOST_OR_IP:9222/json/version
curl http://WINDOWS_HOST_OR_IP:9222/json/list
```

Goed resultaat:

- `/json/version` retourneert JSON met Browser-/Protocol-Version-metagegevens
- `/json/list` retourneert JSON (een lege array is prima als er geen pagina's geopend zijn)

Als dit mislukt, stelt Windows de poort nog niet beschikbaar aan WSL2, is het adres
onjuist voor de WSL2-zijde of ontbreekt een firewallregel, poortdoorsturing of proxy. Herstel
dat voordat je de OpenClaw-configuratie aanraakt.

### Laag 3: configureer het juiste browserprofiel

Laat OpenClaw verwijzen naar het vanuit WSL2 bereikbare adres:

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "remote",
    profiles: {
      remote: {
        cdpUrl: "http://WINDOWS_HOST_OR_IP:9222",
        attachOnly: true,
        color: "#00AA00",
      },
    },
  },
}
```

Opmerkingen:

- gebruik het vanuit WSL2 bereikbare adres, niet een adres dat alleen op Windows werkt
- behoud `attachOnly: true` voor extern beheerde browsers
- `cdpUrl` kan `http://`, `https://`, `ws://` of `wss://` zijn
- gebruik HTTP(S) wanneer je wilt dat OpenClaw `/json/version` detecteert
- gebruik WS(S) alleen wanneer de browserprovider je een rechtstreekse DevTools-
  socket-URL geeft
- test dezelfde URL met `curl` voordat je verwacht dat OpenClaw slaagt

### Laag 4: controleer de Control UI-laag afzonderlijk

Open `http://127.0.0.1:18789/` vanuit Windows en controleer vervolgens:

- de oorsprong van de pagina komt overeen met wat `gateway.controlUi.allowedOrigins` verwacht
- tokenauthenticatie of koppeling is correct geconfigureerd
- je onderzoekt geen authenticatieprobleem van de Control UI alsof het een browserprobleem
  is

Nuttige pagina: [Control UI](/nl/web/control-ui).

### Laag 5: controleer de volledige browserbesturing

Vanuit WSL2:

```bash
openclaw browser --browser-profile remote open https://example.com
openclaw browser --browser-profile remote tabs
```

Goed resultaat:

- het tabblad wordt geopend in Windows Chrome
- `browser tabs` retourneert het doel
- latere acties (`snapshot`, `screenshot`, `navigate`) werken vanuit hetzelfde
  profiel

## Veelvoorkomende misleidende fouten

| Bericht                                                                                 | Betekenis                                                                                                                                                                           |
| --------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `control-ui-insecure-auth`                                                              | probleem met UI-oorsprong/beveiligde context, geen probleem met CDP-transport                                                                                                                     |
| `token_missing`                                                                         | probleem met authenticatieconfiguratie                                                                                                                                                        |
| `pairing required`                                                                      | probleem met apparaatgoedkeuring                                                                                                                                                           |
| `Remote CDP for profile "remote" is not reachable`                                      | WSL2 kan de geconfigureerde `cdpUrl` niet bereiken                                                                                                                                         |
| leeg CDP-antwoord / `other side closed` via een portproxy                               | niet-overeenkomende Windows-listener of een zelflus; controleer beide loopbackfamilies en `netsh interface portproxy show all`                                                                 |
| `Browser attachOnly is enabled and CDP websocket for profile "remote" is not reachable` | het HTTP-eindpunt antwoordde, maar de DevTools-WebSocket kon niet worden geopend                                                                                                        |
| verouderde viewport-/donkere-modus-/locale-/offline-overschrijvingen na een externe sessie          | voer `openclaw browser --browser-profile remote stop` uit om de sessie te sluiten en de gecachete Playwright-/CDP-verbinding vrij te geven zonder de Gateway of de externe browser opnieuw te starten |
| time-out tijdens controle van CDP-bereikbaarheid                                                         | doorgaans nog steeds CDP-bereikbaarheid of een traag/onbereikbaar extern eindpunt                                                                                                             |
| `Playwright page enumeration timed out after 3000ms`                                    | er is verbinding gemaakt met de externe CDP, maar het langdurig uitlezen van tabbladen liep vast                                                                                                                     |
| `No Chrome tabs found for profile="user"`                                               | lokaal Chrome MCP-profiel geselecteerd terwijl er geen hostlokale tabbladen beschikbaar zijn                                                                                                          |

## Checklist voor snelle triage

1. Windows: welke van `127.0.0.1` of `[::1]` antwoordt op `/json/version`, en
   behoort die listener tot `chrome.exe`?
2. WSL2: werkt `curl http://WINDOWS_HOST_OR_IP:9222/json/version`?
3. OpenClaw-configuratie: gebruikt `browser.profiles.<name>.cdpUrl` exact dat
   vanuit WSL2 bereikbare adres?
4. Control UI: open je `http://127.0.0.1:18789/` in plaats van een LAN-IP?
5. Probeer je `existing-session` tussen WSL2 en Windows te gebruiken in plaats
   van rechtstreekse externe CDP?

Controleer eerst lokaal het Windows Chrome-eindpunt, controleer daarna hetzelfde eindpunt
vanuit WSL2 en onderzoek pas vervolgens de OpenClaw-configuratie of authenticatie van de Control UI.

## Gerelateerd

- [Browser](/nl/tools/browser)
- [Browseraanmelding](/nl/tools/browser-login)
- [Probleemoplossing voor Browser op Linux](/nl/tools/browser-linux-troubleshooting)
