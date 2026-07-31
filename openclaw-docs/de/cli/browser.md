---
read_when:
    - Sie verwenden `openclaw browser` und möchten Beispiele für häufige Aufgaben
    - Sie möchten einen Browser steuern, der auf einem anderen Rechner über einen Node-Host ausgeführt wird.
    - Sie möchten über Chrome MCP eine Verbindung mit Ihrem lokalen angemeldeten Chrome-Browser herstellen.
summary: CLI-Referenz für `openclaw browser` (Lebenszyklus, Profile, Tabs, Aktionen, Status und Debugging)
title: Browser
x-i18n:
    generated_at: "2026-07-26T18:21:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 62eb41248cda87cef96be7b0dfe3e0d36a9d3e1ee55c165bd8e3efd68d1e9a5e
    source_path: cli/browser.md
    workflow: 16
---

# `openclaw browser`

Verwalten Sie die Browser-Steuerungsoberfläche von OpenClaw und führen Sie Browseraktionen aus: Lebenszyklus, Profile, Tabs, Snapshots, Screenshots, Navigation, Eingabe, Zustandsemulation und Debugging.

Verwandte Informationen: [Browser-Tool](/de/tools/browser)

## Allgemeine Flags

- `--url <gatewayWsUrl>`: Gateway-WebSocket-URL (standardmäßig aus der Konfiguration).
- `--token <token>`: Gateway-Token (falls erforderlich).
- `--timeout <ms>`: Anfrage-Timeout in ms (Standard: `30000`).
- `--expect-final`: Auf eine endgültige Gateway-Antwort warten.
- `--browser-profile <name>`: Ein Browserprofil auswählen (Standard: `openclaw` oder `browser.defaultProfile`).
- `--json`: Maschinenlesbare Ausgabe (sofern unterstützt). Dies ist eine Option auf Browserebene; platzieren Sie sie daher für eine eindeutige Form vor dem Unterbefehl, beispielsweise
  `openclaw browser --json status`. Eine nachgestellte Platzierung wie
  `openclaw browser status --json` funktioniert ebenfalls, wenn der ausgewählte untergeordnete Befehl keine eigene Option
  `--json` definiert.

## Schnellstart (lokal)

```bash
openclaw browser profiles
openclaw browser --browser-profile openclaw start
openclaw browser --browser-profile openclaw open https://example.com
openclaw browser --browser-profile openclaw snapshot
```

Agenten können dieselbe Bereitschaftsprüfung mit `browser({ action: "doctor" })` ausführen.

## Schnelle Fehlerbehebung

Wenn `start` mit `not reachable after start` fehlschlägt, beheben Sie zuerst Probleme mit der CDP-Bereitschaft. Wenn `start` und `tabs` erfolgreich sind, aber `open` oder `navigate` fehlschlägt, ist die Browser-Steuerungsebene funktionsfähig und der Fehler wird normalerweise durch eine SSRF-Richtlinienblockierung der Navigation verursacht.

Minimale Abfolge:

```bash
openclaw browser --browser-profile openclaw doctor
openclaw browser --browser-profile openclaw start
openclaw browser --browser-profile openclaw tabs
openclaw browser --browser-profile openclaw open https://example.com
```

Ausführliche Anleitung: [Fehlerbehebung für den Browser](/de/tools/browser#cdp-startup-failure-vs-navigation-ssrf-block)

## Lebenszyklus

```bash
openclaw browser status
openclaw browser doctor
openclaw browser doctor --deep
openclaw browser start
openclaw browser start --headless
openclaw browser stop
openclaw browser --browser-profile openclaw reset-profile
```

- `doctor --deep` fügt eine Live-Snapshot-Prüfung hinzu: nützlich, wenn die grundlegende CDP-Bereitschaft gegeben ist, Sie aber einen Nachweis benötigen, dass der aktuelle Tab untersucht werden kann.
- Für ein laufendes lokal verwaltetes Profil melden `status` und `doctor` zwischengespeicherte
  Grafikdiagnosen aus Chrome: Hardware-/Softwareklassifizierung, Renderer,
  Backend, Gerät/Treiber, Funktions- und Deaktivierungsstatusdetails sowie beschleunigte
  Videofunktionen. `openclaw browser --json status` gibt die vollständigen strukturierten Nutzdaten zurück.
  Der passive Status startet Chrome niemals nur zur Erfassung dieser Fakten.
- `stop` schließt die aktive Steuerungssitzung und entfernt temporäre Emulationsüberschreibungen auch für `attachOnly` und Remote-CDP-Profile, bei denen OpenClaw den Browserprozess nicht selbst gestartet hat. Bei lokal verwalteten Profilen beendet `stop` außerdem den gestarteten Browserprozess.
- `start --headless` gilt nur für diese Startanfrage und nur, wenn OpenClaw einen lokal verwalteten Browser startet. Es schreibt weder `browser.headless` noch die Profilkonfiguration um und hat bei einem bereits laufenden Browser keine Wirkung.
- Auf Linux-Hosts ohne `DISPLAY` oder `WAYLAND_DISPLAY` werden lokal verwaltete Profile automatisch im Headless-Modus ausgeführt, sofern `OPENCLAW_BROWSER_HEADLESS=0`, `browser.headless=false` oder `browser.profiles.<name>.headless=false` nicht ausdrücklich einen sichtbaren Browser anfordert.

## Wenn der Befehl fehlt

Wenn `openclaw browser` ein unbekannter Befehl ist, überprüfen Sie `plugins.allow` in `~/.openclaw/openclaw.json`. Wenn `plugins.allow` vorhanden ist, führen Sie das gebündelte Browser-Plugin ausdrücklich auf, sofern die Konfiguration nicht bereits einen `browser`-Block auf der obersten Ebene enthält:

```json5
{
  plugins: {
    allow: ["telegram", "browser"],
  },
}
```

Ein ausdrücklicher `browser`-Block auf der obersten Ebene (beispielsweise `browser.enabled=true` oder `browser.profiles.<name>`) aktiviert das gebündelte Browser-Plugin ebenfalls unter einer restriktiven Plugin-Zulassungsliste.

Verwandte Informationen: [Browser-Tool](/de/tools/browser#missing-browser-command-or-tool)

## Profile

Profile sind benannte Browser-Routing-Konfigurationen:

- `openclaw` (Standard): Startet eine dedizierte, von OpenClaw verwaltete Chrome-Instanz oder stellt eine Verbindung zu ihr her (isoliertes Benutzerdatenverzeichnis).
- `user`: Steuert Ihre vorhandene angemeldete Chrome-Sitzung über Chrome DevTools MCP.
- Benutzerdefinierte CDP-Profile: Verweisen auf einen lokalen oder entfernten CDP-Endpunkt.

```bash
openclaw browser profiles
openclaw browser system-profiles
openclaw browser system-profiles --browser brave
openclaw browser import-profile --browser chrome --system Default --into imported
openclaw browser import-profile --system "Profile 1" --into work --domains google.com,youtube.com
openclaw browser create-profile --name work --color "#FF5A36"
openclaw browser create-profile --name chrome-live --driver existing-session
openclaw browser create-profile --name remote --cdp-url https://browser-host.example.com
openclaw browser delete-profile --name work
```

Verwenden Sie bei einem beliebigen Unterbefehl mit `--browser-profile <name>` ein bestimmtes Profil, beispielsweise `openclaw browser --browser-profile work tabs`.

Unter macOS listet `system-profiles` die tatsächlich auf dem Host verfügbaren Chrome-, Brave-, Edge- oder Chromium-Profile auf. `import-profile` entschlüsselt deren Cookies nach einer einmaligen Zustimmung über den macOS-Schlüsselbund/Touch ID und fügt sie in ein neues, von OpenClaw verwaltetes Profil ein. Dabei werden nur Cookies importiert; lokaler Speicher und IndexedDB bleiben unverändert. Einige Google-Sitzungen verwenden gerätegebundene Sitzungsanmeldedaten (DBSC) und können nach dem Import weiterhin eine erneute Authentifizierung erfordern.

Wenn die macOS-App ein lokales Gateway verwendet, kann sie diesen Import einmalig anbieten und das isolierte importierte Profil als Standard für das Browsen durch Agenten festlegen. Der Import erfordert immer einen ausdrücklichen Klick; ein erfolgreicher Import oder das Schließen der Aufforderung unterdrückt spätere automatische Aufforderungen, und **Settings → General → Browser login** bleibt für einen erneuten Import verfügbar.

Der Import von Systemprofilen ist standardmäßig aktiviert. Setzen Sie `browser.allowSystemProfileImport=false`, um sowohl CLI- als auch durch Agenten ausgelöste Importe zu deaktivieren. Der Import erfolgt lokal auf dem Host und kann nicht über den Browser-Node-Proxy ausgeführt werden.

## Tabs

```bash
openclaw browser tabs
openclaw browser tab new --label docs
openclaw browser tab label t1 docs
openclaw browser tab select 2
openclaw browser tab close 2
openclaw browser open https://docs.openclaw.ai --label docs
openclaw browser focus docs
openclaw browser close t1
```

`tabs` gibt zuerst `suggestedTargetId`, dann die stabile `tabId` (beispielsweise `t1`), die optionale Bezeichnung und die rohe `targetId` zurück. Übergeben Sie `suggestedTargetId` erneut an `focus`, `close`, Snapshots und Aktionen. Weisen Sie mit `open --label`, `tab new --label` oder `tab label` eine Bezeichnung zu; Bezeichnungen, Tab-IDs, rohe Ziel-IDs und eindeutige Ziel-ID-Präfixe werden sämtlich akzeptiert. Das Anfragefeld heißt aus Kompatibilitätsgründen weiterhin `targetId`, akzeptiert jedoch alle diese Tab-Referenzen.

Rohe Ziel-IDs sind flüchtige Diagnosekennungen und kein dauerhafter Agentenspeicher: Wenn Chromium das zugrunde liegende rohe Ziel während einer Navigation oder Formularübermittlung ersetzt, behält OpenClaw die stabile `tabId`/Bezeichnung am Ersatz-Tab bei, sofern die Übereinstimmung nachgewiesen werden kann. Bevorzugen Sie `suggestedTargetId`.

## Snapshot / Screenshot / Aktionen

Snapshot:

```bash
openclaw browser snapshot
openclaw browser snapshot --urls
```

Screenshot:

```bash
openclaw browser screenshot
openclaw browser screenshot --full-page
openclaw browser screenshot --ref e12
openclaw browser screenshot --labels
```

- `--full-page` ist ausschließlich für Seitenaufnahmen vorgesehen und kann nicht mit `--ref` oder `--element` kombiniert werden.
- `existing-session`- / `user`-Profile unterstützen Seiten-Screenshots und `--ref`-Screenshots aus der Snapshot-Ausgabe, jedoch keine CSS-`--element`-Screenshots.
- `--labels` überlagert den Screenshot mit den aktuellen Snapshot-Referenzen. Bei Playwright-basierten Profilen funktioniert dies mit `--full-page` (ganzseitige Überlagerung), `--ref` (Elementausschnitt-Überlagerung anhand einer ARIA-Referenz) und `--element` (Elementausschnitt-Überlagerung anhand eines CSS-Selektors); in den Elementausschnittmodi werden Bezeichnungen relativ zum Element projiziert. Die Antwort enthält außerdem ein `annotations`-Array (wird bei Leerstand ausgelassen) mit dem Begrenzungsrahmen jeder Referenz: `ref`, `number`, `role`, optional `name` und `box: {x, y, width, height}` im Koordinatenraum des aufgenommenen Bildes (Viewport / ganze Seite / elementrelativ).
  `existing-session`-Profile rendern bei Seiten-Screenshots eine chrome-mcp-Überlagerung, verwenden jedoch nicht die Playwright-Projektionshilfe und enthalten nicht `annotations`; CSS-`--element`-Screenshots werden dort nicht unterstützt. Ohne Playwright oder chrome-mcp sind beschriftete Screenshots nicht verfügbar.
- `snapshot --urls` hängt erkannte Linkziele an KI-Snapshots an, damit Agenten direkte Navigationsziele auswählen können, anstatt ausschließlich anhand des Linktexts zu raten.

Navigieren/Klicken/Eingeben (referenzbasierte UI-Automatisierung):

```bash
openclaw browser navigate https://example.com
openclaw browser click <ref>
openclaw browser click-coords 120 340
openclaw browser type <ref> "hello"
openclaw browser press Enter
openclaw browser hover <ref>
openclaw browser scrollintoview <ref>
openclaw browser drag <startRef> <endRef>
openclaw browser select <ref> OptionA OptionB
openclaw browser fill --fields '[{"ref":"1","value":"Ada"}]'
openclaw browser wait --text "Done"
openclaw browser evaluate --fn '(el) => el.textContent' --ref <ref>
openclaw browser evaluate --fn 'const title = document.title; return title;'
openclaw browser evaluate --timeout-ms 30000 --fn 'async () => { await window.ready; return true; }'
```

`evaluate --fn` akzeptiert eine Funktionsquelle, einen Ausdruck oder einen Anweisungsrumpf. Anweisungsrümpfe werden als asynchrone Funktionen gekapselt; verwenden Sie daher `return` für den gewünschten Rückgabewert. Verwenden Sie `--timeout-ms`, wenn die seitenseitige Funktion möglicherweise länger als der standardmäßige Auswertungs-Timeout benötigt. `browser.evaluateEnabled=false` (Standard: `true`) deaktiviert sowohl `evaluate` als auch `wait --fn`.

Aktionsantworten geben die aktuelle rohe `targetId` nach einem durch eine Aktion ausgelösten Seitenaustausch zurück, sofern OpenClaw den Ersatz-Tab nachweisen kann. Skripte sollten für langlebige Workflows weiterhin `suggestedTargetId`/Bezeichnungen speichern und übergeben.

Hilfsfunktionen für Dateien und Dialogfelder:

```bash
openclaw browser upload /tmp/openclaw/uploads/file.pdf --ref <ref>
openclaw browser upload media://inbound/file.pdf --ref <ref>
openclaw browser waitfordownload
openclaw browser download <ref> report.pdf
openclaw browser dialog --accept
openclaw browser dialog --dismiss --dialog-id d1
```

Verwaltete Chrome-Profile speichern gewöhnliche, durch Klicks ausgelöste Downloads im OpenClaw-Downloadverzeichnis (standardmäßig `/tmp/openclaw/downloads` oder im konfigurierten temporären Stammverzeichnis). Verwenden Sie `waitfordownload` oder `download`, wenn der Agent auf eine bestimmte Datei warten und deren Pfad zurückgeben muss; diese ausdrücklichen Warteoperationen übernehmen den nächsten Download. Für Uploads werden Dateien aus dem temporären Upload-Stammverzeichnis von OpenClaw und von OpenClaw verwaltete eingehende Medien akzeptiert, einschließlich `media://inbound/<id>`- und Sandbox-relativer `media/inbound/<id>`-Referenzen. Verschachtelte Medienreferenzen, Verzeichnisdurchquerung und beliebige lokale Pfade werden abgelehnt.

Wenn eine Aktion ein modales Dialogfeld öffnet, gibt die Aktionsantwort `blockedByDialog` mit `browserState.dialogs.pending` zurück; übergeben Sie `--dialog-id`, um es direkt zu beantworten. Außerhalb von OpenClaw verarbeitete Dialogfelder erscheinen unter `browserState.dialogs.recent`.

Stapelaktionen:

```bash
openclaw browser batch --actions '[{"kind":"wait","timeMs":500},{"kind":"click","ref":"12"},{"kind":"type","ref":"23","text":"hello"}]'
openclaw browser batch --actions-file plan.json
openclaw browser batch --actions-file - --continue
```

`openclaw browser batch` sendet eine `kind="batch"`-`/act`-Anfrage mit verschachtelten `BrowserActRequest`-Aktionen (`wait`, `click`, `type`, `evaluate`, ...) – nicht `open`/`navigate`/`snapshot`/`screenshot`, bei denen es sich um CLI-Unterbefehle und nicht um `/act`-Arten handelt. `--continue` legt `stopOnError=false` fest (standardmäßig wird beim ersten Fehler abgebrochen); `--target-id` beschränkt den gesamten Batch auf einen Tab. Eine fehlgeschlagene verschachtelte Aktion führt dazu, dass der Befehl mit einem von null verschiedenen Statuscode beendet wird; verwenden Sie `--json`, um die geordnete `results`-Antwort beizubehalten. Den vollständigen Vertrag (Lebenszyklus von Referenzen, Konflikte bei Ziel-IDs, Fehlerzusammenfassung) finden Sie unter [Browser-Batch-CLI](/de/tools/browser-control#browser-batch-cli). `batch` wird bei `profile="user"`- / bestehenden Sitzungsprofilen nicht unterstützt.

## Status und Speicher

Viewport und Emulation:

```bash
openclaw browser resize 1280 720
openclaw browser set viewport 1280 720
openclaw browser set offline on
openclaw browser set media dark
openclaw browser set timezone Europe/London
openclaw browser set locale en-GB
openclaw browser set geo 51.5074 -0.1278 --accuracy 25
openclaw browser set device "iPhone 14"
openclaw browser set headers '{"x-test":"1"}'
openclaw browser set credentials myuser mypass
```

Cookies und Speicher:

```bash
openclaw browser cookies
openclaw browser cookies set session abc123 --url https://example.com
openclaw browser cookies clear
openclaw browser storage local get
openclaw browser storage local set token abc123
openclaw browser storage session clear
```

## Fehlerbehebung

```bash
openclaw browser console --level error
openclaw browser pdf
openclaw browser responsebody "**/api"
openclaw browser highlight <ref>
openclaw browser errors --clear
openclaw browser requests --filter api
openclaw browser trace start
openclaw browser trace stop --out trace.zip
```

## Vorhandenes Chrome über MCP

Verwenden Sie das integrierte Profil `user` oder erstellen Sie Ihr eigenes Profil `existing-session`:

```bash
openclaw browser --browser-profile user tabs
openclaw browser create-profile --name chrome-live --driver existing-session
openclaw browser create-profile --name brave-live --driver existing-session --user-data-dir "~/Library/Application Support/BraveSoftware/Brave-Browser"
openclaw browser create-profile --name chrome-port --driver existing-session --cdp-url http://127.0.0.1:9222
openclaw browser --browser-profile chrome-live tabs
```

Der Standardpfad für bestehende Sitzungen ist die automatische Verbindung von Chrome MCP ausschließlich auf dem Host. Wenn der Browser bereits mit einem DevTools-Endpunkt ausgeführt wird, übergeben Sie `--cdp-url`, damit Chrome MCP stattdessen eine Verbindung zu diesem Endpunkt herstellt. Verwenden Sie für Docker, Browserless oder andere Remote-Konfigurationen, bei denen die Semantik von Chrome MCP nicht benötigt wird, stattdessen ein CDP-Profil.

Aktuelle Einschränkungen für bestehende Sitzungen:

- Snapshot-gesteuerte Aktionen verwenden Referenzen, keine CSS-Selektoren.
- Unterstützte `act`-Anfragen verwenden einen integrierten Standardwert von 60000 ms, wenn Aufrufer `timeoutMs` auslassen; der aufrufbezogene Wert `timeoutMs` hat weiterhin Vorrang.
- `click` unterstützt nur Linksklicks.
- `type` unterstützt `slowly=true` nicht.
- `press` unterstützt `delayMs` nicht.
- `hover`, `scrollintoview`, `drag`, `select` und `fill` lehnen aufrufbezogene Zeitüberschreibungen ab; `evaluate` akzeptiert `--timeout-ms`.
- `select` unterstützt nur einen Wert.
- `wait --load networkidle` wird nicht unterstützt (funktioniert mit verwalteten und unformatierten/Remote-CDP-Profilen).
- Datei-Uploads erfordern `--ref` / `--input-ref`, unterstützen keine CSS-`--element` und jeweils nur eine Datei.
- Dialog-Hooks unterstützen `--timeout` nicht.
- Screenshots unterstützen Seitenaufnahmen und `--ref`, jedoch keine CSS-`--element`.
- `responsebody`, das Abfangen von Downloads, der PDF-Export und Batch-Aktionen erfordern weiterhin einen verwalteten Browser oder ein unformatiertes CDP-Profil.

## Remote-Browsersteuerung (Node-Host-Proxy)

Wenn das Gateway auf einem anderen Rechner als der Browser ausgeführt wird, führen Sie auf dem Rechner mit Chrome/Brave/Edge/Chromium einen **Node-Host** aus. Das Gateway leitet Browseraktionen an diesen Node weiter; ein separater Browsersteuerungsserver ist nicht erforderlich.

Verwenden Sie `gateway.nodes.browser.mode`, um das automatische Routing zu steuern, und `gateway.nodes.browser.node`, um einen bestimmten Node festzulegen, wenn mehrere verbunden sind.

Sicherheit und Remote-Einrichtung: [Browser-Tool](/de/tools/browser), [Remote-Zugriff](/de/gateway/remote), [Tailscale](/de/gateway/tailscale), [Sicherheit](/de/gateway/security)

## Verwandte Themen

- [CLI-Referenz](/de/cli)
- [Browser](/de/tools/browser)
