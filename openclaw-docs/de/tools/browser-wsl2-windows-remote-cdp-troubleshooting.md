---
read_when:
    - OpenClaw Gateway in WSL2 ausführen, während Chrome unter Windows läuft
    - Sich überschneidende Browser-/Control-UI-Fehler unter WSL2 und Windows feststellen
    - Entscheidung zwischen hostlokalem Chrome MCP und unverarbeitetem Remote-CDP in Setups mit getrennten Hosts
summary: WSL2-Gateway und Windows-Chrome-Remote-CDP schichtweise beheben
title: Fehlerbehebung für WSL2 + Windows + Remote-Chrome-CDP
x-i18n:
    generated_at: "2026-07-26T18:06:23Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 66ec4ed5bfccc66b594a43d56296c69242e8b9cf50b36c6cb3990b1d6ea58faa
    source_path: tools/browser-wsl2-windows-remote-cdp-troubleshooting.md
    workflow: 16
---

Bei der üblichen Konfiguration mit getrennten Hosts läuft das OpenClaw Gateway innerhalb von WSL2, Chrome läuft
unter Windows, und die Browsersteuerung muss die WSL2/Windows-Grenze überqueren. Mehrere
unabhängige Probleme können gleichzeitig auftreten (siehe
[Issue #39369](https://github.com/openclaw/openclaw/issues/39369)): CDP-
Transport, Ursprungssicherheit der Control UI sowie Token/Kopplung können jeweils
unabhängig fehlschlagen und dabei ähnlich aussehende Fehler erzeugen. Arbeiten Sie die
folgenden Ebenen der Reihe nach ab, anstatt zu raten, welche davon defekt ist.

## Wählen Sie zuerst den richtigen Browsermodus

### Option 1: direktes Remote-CDP von WSL2 zu Windows

Verwenden Sie ein Remote-Browserprofil, das von WSL2 auf einen Chrome-CDP-
Endpunkt unter Windows verweist. Wählen Sie dies, wenn das Gateway innerhalb von WSL2 bleibt, Chrome unter
Windows läuft und die Browsersteuerung die WSL2/Windows-Grenze überqueren muss.

### Option 2: hostlokales Chrome MCP

Verwenden Sie den `existing-session`-Treiber (Profil `user`) nur, wenn das Gateway
auf demselben Host wie Chrome läuft, Sie den lokalen angemeldeten Browserstatus verwenden möchten,
keinen hostübergreifenden Browsertransport benötigen und weder `responsebody`,
PDF-Export, Download-Abfangung noch Batch-Aktionen benötigen (Chrome-MCP-Profile
unterstützen diese nicht).

Verwenden Sie für WSL2 Gateway + Windows Chrome direktes Remote-CDP. Chrome MCP ist
hostlokal und keine Brücke zwischen WSL2 und Windows.

## Funktionsfähige Architektur

- WSL2 führt das Gateway auf `127.0.0.1:18789` aus
- Windows öffnet die Control UI in einem normalen Browser unter `http://127.0.0.1:18789/`
- Chrome unter Windows stellt einen CDP-Endpunkt auf Port `9222` bereit
- WSL2 kann diesen Windows-CDP-Endpunkt erreichen
- OpenClaw richtet ein Browserprofil auf die von WSL2 erreichbare Adresse

## Kritische Regel für die Control UI

Wenn die UI unter Windows geöffnet wird, verwenden Sie Windows-Localhost, sofern Sie nicht
bewusst HTTPS eingerichtet haben:

```text
http://127.0.0.1:18789/
```

Verwenden Sie nicht standardmäßig eine LAN-IP. Einfaches HTTP über eine LAN- oder Tailnet-Adresse kann
vom CDP selbst unabhängiges Verhalten bezüglich unsicheren Ursprungs bzw. Geräteauthentifizierung
auslösen. Siehe
[Control UI](/de/web/control-ui).

## Validierung nach Ebenen

Arbeiten Sie von oben nach unten; überspringen Sie keine Schritte. Nach der Behebung einer Ebene kann
weiterhin ein anderer Fehler aus einer tieferen Ebene sichtbar sein.

### Ebene 1: Prüfen, ob Chrome unter Windows CDP bereitstellt

```powershell
chrome.exe --remote-debugging-port=9222 --user-data-dir="$env:LOCALAPPDATA\OpenClaw\ChromeCDP"
```

Chrome 136 und höher ignorieren Befehlszeilenoptionen für Remote-Debugging beim
standardmäßigen Chrome-Datenverzeichnis. Verwenden Sie wie oben gezeigt ein separates,
nicht standardmäßiges Datenverzeichnis. Siehe Chromes
[Sicherheitsänderung für Remote-Debugging](https://developer.chrome.com/blog/remote-debugging-port).
Dadurch wird das normale angemeldete Chrome-Profil nicht remote steuerbar.

Prüfen Sie unter Windows zunächst Chrome selbst:

```powershell
curl.exe http://127.0.0.1:9222/json/version
curl.exe http://127.0.0.1:9222/json/list
```

Falls dies fehlschlägt, diagnostizieren Sie die Windows-Listener wie unten beschrieben. OpenClaw ist
zu diesem Zeitpunkt noch nicht das Problem.

#### IPv4 und IPv6 diagnostizieren, bevor portproxy geändert wird

Chromium versucht zuerst, Remote-Debugging an `127.0.0.1` zu binden, und weicht nur dann auf
`[::1]` aus, wenn die IPv4-Bindung fehlschlägt. Eine dauerhafte `v4tov4`-Regel, die auf
`127.0.0.1:9222` lauscht, kann diesen Endpunkt belegen, bevor Chrome startet. Chrome
weicht dann auf `[::1]:9222` aus, während die alte Regel IPv4-Datenverkehr an
ihren eigenen Listener zurückleitet und eine leere Antwort zurückgibt.

Prüfen Sie die tatsächlichen Listener und Proxyregeln unter Windows, anstatt sie
aus der Chrome-Version abzuleiten:

```powershell
netstat -ano | findstr :9222
netsh interface portproxy show all
curl.exe http://127.0.0.1:9222/json/version
curl.exe http://[::1]:9222/json/version
```

Verwenden Sie `tasklist /fi "PID eq <PID>"` für jede PID aus `netstat`.

- Wenn `chrome.exe` unter `127.0.0.1` antwortet, entfernen Sie jede portproxy-Regel, die ebenfalls
  auf `127.0.0.1:9222` lauscht. Leiten Sie nur die von WSL2 erreichbare Windows-Adapteradresse
  an `127.0.0.1` weiter.
- Wenn `chrome.exe` nur unter `[::1]` antwortet, richten Sie den von WSL2 erreichbaren Listener mit
  `v4tov6` auf `::1`, anstatt an eine ungenutzte IPv4-Adresse weiterzuleiten:

  ```powershell
  netsh interface portproxy add v4tov6 listenaddress=WINDOWS_HOST_OR_IP listenport=9222 connectaddress=::1 connectport=9222
  ```

Binden Sie den Listener an die Adapteradresse, die WSL2 benötigt. Stellen Sie den CDP-
Port nicht auf `0.0.0.0`, einer LAN-Adresse oder einer Tailnet-Adresse bereit: CDP gewährt die Kontrolle über
die Browsersitzung.

### Ebene 2: Prüfen, ob WSL2 diesen Windows-Endpunkt erreichen kann

Testen Sie in WSL2 genau die Adresse, die Sie in `cdpUrl` verwenden möchten:

```bash
curl http://WINDOWS_HOST_OR_IP:9222/json/version
curl http://WINDOWS_HOST_OR_IP:9222/json/list
```

Gutes Ergebnis:

- `/json/version` gibt JSON mit Browser-/Protocol-Version-Metadaten zurück
- `/json/list` gibt JSON zurück (ein leeres Array ist in Ordnung, wenn keine Seiten geöffnet sind)

Falls dies fehlschlägt, stellt Windows den Port noch nicht für WSL2 bereit, die Adresse ist
für die WSL2-Seite falsch oder Firewall, Portweiterleitung bzw. Proxying fehlen. Beheben Sie
dies, bevor Sie die OpenClaw-Konfiguration ändern.

### Ebene 3: Das richtige Browserprofil konfigurieren

Richten Sie OpenClaw auf die von WSL2 erreichbare Adresse:

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

Hinweise:

- Verwenden Sie die von WSL2 erreichbare Adresse, nicht eine, die nur unter Windows funktioniert
- Behalten Sie `attachOnly: true` für extern verwaltete Browser bei
- `cdpUrl` kann `http://`, `https://`, `ws://` oder `wss://` sein
- Verwenden Sie HTTP(S), wenn OpenClaw `/json/version` ermitteln soll
- Verwenden Sie WS(S) nur, wenn der Browser-Provider Ihnen eine direkte DevTools-
  Socket-URL bereitstellt
- Testen Sie dieselbe URL mit `curl`, bevor Sie erwarten, dass OpenClaw erfolgreich arbeitet

### Ebene 4: Die Control-UI-Ebene separat prüfen

Öffnen Sie `http://127.0.0.1:18789/` unter Windows und prüfen Sie anschließend:

- Der Ursprung der Seite entspricht den Erwartungen von `gateway.controlUi.allowedOrigins`
- Token-Authentifizierung oder Kopplung ist korrekt konfiguriert
- Sie untersuchen kein Authentifizierungsproblem der Control UI fälschlicherweise als Browser-
  Problem

Hilfreiche Seite: [Control UI](/de/web/control-ui).

### Ebene 5: Durchgängige Browsersteuerung prüfen

Unter WSL2:

```bash
openclaw browser --browser-profile remote open https://example.com
openclaw browser --browser-profile remote tabs
```

Gutes Ergebnis:

- Der Tab wird in Chrome unter Windows geöffnet
- `browser tabs` gibt das Ziel zurück
- Spätere Aktionen (`snapshot`, `screenshot`, `navigate`) funktionieren mit demselben
  Profil

## Häufige irreführende Fehler

| Meldung                                                                                 | Bedeutung                                                                                                                                                                           |
| --------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `control-ui-insecure-auth`                                                              | Problem mit UI-Ursprung/sicherem Kontext, kein Problem mit dem CDP-Transport                                                                                                                     |
| `token_missing`                                                                         | Problem mit der Authentifizierungskonfiguration                                                                                                                                                        |
| `pairing required`                                                                      | Problem mit der Gerätegenehmigung                                                                                                                                                           |
| `Remote CDP for profile "remote" is not reachable`                                      | WSL2 kann die konfigurierte `cdpUrl` nicht erreichen                                                                                                                                         |
| leere CDP-Antwort / `other side closed` über einen portproxy                               | Nicht übereinstimmende Windows-Listener oder eine Selbstschleife; prüfen Sie beide Loopback-Adressfamilien und `netsh interface portproxy show all`                                                                 |
| `Browser attachOnly is enabled and CDP websocket for profile "remote" is not reachable` | Der HTTP-Endpunkt hat geantwortet, aber der DevTools-WebSocket konnte nicht geöffnet werden                                                                                                        |
| veraltete Viewport-, Dark-Mode-, Gebietsschema- oder Offline-Überschreibungen nach einer Remote-Sitzung          | Führen Sie `openclaw browser --browser-profile remote stop` aus, um die Sitzung zu schließen und die zwischengespeicherte Playwright/CDP-Verbindung freizugeben, ohne das Gateway oder den externen Browser neu zu starten |
| Zeitüberschreitung beim Prüfen der CDP-Erreichbarkeit                                                         | In der Regel weiterhin ein Problem mit der CDP-Erreichbarkeit oder ein langsamer/nicht erreichbarer Remote-Endpunkt                                                                                                             |
| `Playwright page enumeration timed out after 3000ms`                                    | Die Remote-CDP-Verbindung wurde hergestellt, aber das dauerhafte Lesen des Tabs ist ins Stocken geraten                                                                                                                     |
| `No Chrome tabs found for profile="user"`                                               | Lokales Chrome-MCP-Profil wurde ausgewählt, obwohl keine hostlokalen Tabs verfügbar sind                                                                                                          |

## Checkliste für eine schnelle Fehlerdiagnose

1. Windows: Welche der Adressen `127.0.0.1` oder `[::1]` antwortet auf `/json/version`, und
   gehört dieser Listener zu `chrome.exe`?
2. WSL2: Funktioniert `curl http://WINDOWS_HOST_OR_IP:9222/json/version`?
3. OpenClaw-Konfiguration: Verwendet `browser.profiles.<name>.cdpUrl` genau diese
   von WSL2 erreichbare Adresse?
4. Control UI: Öffnen Sie `http://127.0.0.1:18789/` statt einer LAN-IP?
5. Versuchen Sie, `existing-session` über WSL2 und Windows hinweg zu verwenden,
   anstatt direktes Remote-CDP einzusetzen?

Prüfen Sie zuerst den Windows-Chrome-Endpunkt lokal, prüfen Sie anschließend denselben Endpunkt
von WSL2 aus und untersuchen Sie erst dann die OpenClaw-Konfiguration oder die Authentifizierung der Control UI.

## Verwandte Themen

- [Browser](/de/tools/browser)
- [Browser-Anmeldung](/de/tools/browser-login)
- [Fehlerbehebung für Browser unter Linux](/de/tools/browser-linux-troubleshooting)
