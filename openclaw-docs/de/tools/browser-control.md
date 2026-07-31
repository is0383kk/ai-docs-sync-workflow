---
read_when:
    - Skripterstellung oder Debugging des Agent-Browsers über die lokale Steuerungs-API
    - Suche nach der `openclaw browser`-CLI-Referenz
    - Benutzerdefinierte Browserautomatisierung mit Snapshots und Referenzen hinzufügen
summary: OpenClaw-Browsersteuerungs-API, CLI-Referenz und Skriptaktionen
title: Browsersteuerungs-API
x-i18n:
    generated_at: "2026-07-26T18:47:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 812358a5ad366e419413b78507d3620ea9f3981224bc8cc62fb512b87eaadd9b
    source_path: tools/browser-control.md
    workflow: 16
---

Informationen zu Einrichtung, Konfiguration und Fehlerbehebung finden Sie unter [Browser](/de/tools/browser).
Diese Seite dient als Referenz für die lokale HTTP-Steuerungs-API, die `openclaw browser`
CLI und Skripting-Muster (Snapshots, Refs, Wartevorgänge, Debug-Abläufe).

## Steuerungs-API (optional)

Nur für lokale Integrationen stellt das Gateway eine kleine Loopback-HTTP-API bereit.
Dieser eigenständige Server ist optional — setzen Sie die Umgebungsvariable
`OPENCLAW_EAGER_BROWSER_CONTROL_SERVER=1` in der Umgebung des Gateway-Dienstes
und starten Sie das Gateway neu, bevor die HTTP-Endpunkte verfügbar werden. Ohne
diese Variable funktioniert die Browser-Steuerungslaufzeit weiterhin über die CLI und
Agent-Tools, aber am Loopback-Steuerungsport lauscht kein Dienst.

- Status/Start/Stopp: `GET /`, `GET /doctor`, `POST /start`, `POST /stop`, `POST /reset-profile`
- Profile: `GET /profiles`, `POST /profiles/create`, `DELETE /profiles/:name`
- Tabs: `GET /tabs`, `POST /tabs/open`, `POST /tabs/focus`, `DELETE /tabs/:targetId`, `POST /tabs/action`
- Snapshot/Screenshot: `GET /snapshot`, `POST /screenshot`
- Aktionen: `POST /navigate`, `POST /act`
- Hooks: `POST /hooks/file-chooser`, `POST /hooks/dialog`
- Downloads: `POST /download`, `POST /wait/download`
- Berechtigungen: `POST /permissions/grant`
- Debugging: `GET /console`, `POST /pdf`
- Debugging: `GET /errors`, `GET /requests`, `GET /dialogs`, `POST /trace/start`, `POST /trace/stop`, `POST /highlight`
- Netzwerk: `POST /response/body`
- Zustand: `GET /cookies`, `POST /cookies/set`, `POST /cookies/clear`
- Zustand: `GET /storage/:kind`, `POST /storage/:kind/set`, `POST /storage/:kind/clear`
- Einstellungen: `POST /set/offline`, `POST /set/headers`, `POST /set/credentials`, `POST /set/geolocation`, `POST /set/media`, `POST /set/timezone`, `POST /set/locale`, `POST /set/device`

`POST /tabs/action` ist die gebündelte Form, welche die CLI intern für
`browser tab`-Unterbefehle verwendet (`{"action":"new"|"label"|"select"|"close"|"list", ...}`);
bevorzugen Sie beim direkten Skripting die oben aufgeführten zweckgebundenen Tab-Routen.

Alle Endpunkte akzeptieren `?profile=<name>`. `POST /start?headless=true` fordert einen
einmaligen Headless-Start für lokale verwaltete Profile an, ohne die persistierte
Browserkonfiguration zu ändern; reine Anbindungs-, Remote-CDP- und bestehende Sitzungsprofile lehnen
diese Überschreibung ab, da OpenClaw diese Browserprozesse nicht startet.

Für Tab-Endpunkte ist `targetId` der Kompatibilitätsfeldname. Übergeben Sie vorzugsweise
`suggestedTargetId` aus `GET /tabs` oder `POST /tabs/open`; Bezeichnungen und `tabId`-
Handles wie `t1` werden ebenfalls akzeptiert. Unverarbeitete CDP-Ziel-IDs und eindeutige Präfixe
unverarbeiteter Ziel-IDs funktionieren weiterhin, sind jedoch flüchtige Diagnose-Handles.

Wenn die Gateway-Authentifizierung mit einem gemeinsamen Geheimnis konfiguriert ist, erfordern auch Browser-HTTP-Routen eine Authentifizierung:

- `Authorization: Bearer <gateway token>`
- `x-openclaw-password: <gateway password>` oder HTTP-Basic-Authentifizierung mit diesem Passwort

Hinweise:

- Diese eigenständige Loopback-Browser-API verwendet **keine** Identitäts-Header von vertrauenswürdigen Proxys oder
  Tailscale Serve.
- Wenn `gateway.auth.mode` auf `none` oder `trusted-proxy` gesetzt ist, übernehmen diese Loopback-Browser-
  Routen diese identitätstragenden Modi nicht; beschränken Sie sie auf Loopback.

### `/act`-Fehlervertrag

`POST /act` verwendet für Validierungs- und
Richtlinienfehler auf Routenebene eine strukturierte Fehlerantwort:

```json
{ "error": "<message>", "code": "ACT_*" }
```

Aktuelle `code`-Werte:

- `ACT_KIND_REQUIRED` (HTTP 400): `kind` fehlt oder wird nicht erkannt.
- `ACT_INVALID_REQUEST` (HTTP 400): Die Aktionsnutzlast konnte nicht normalisiert oder validiert werden.
- `ACT_SELECTOR_UNSUPPORTED` (HTTP 400): `selector` wurde mit einem nicht unterstützten Aktionstyp verwendet.
- `ACT_EVALUATE_DISABLED` (HTTP 403): `evaluate` (oder `wait --fn`) ist durch die Konfiguration deaktiviert.
- `ACT_TARGET_ID_MISMATCH` (HTTP 403): `targetId` auf oberster Ebene oder in einer gebündelten Anfrage steht im Konflikt mit dem Anfrageziel.
- `ACT_EXISTING_SESSION_UNSUPPORTED` (HTTP 501): Die Aktion wird für bestehende Sitzungsprofile nicht unterstützt.

Andere Laufzeitfehler können weiterhin `{ "error": "<message>" }` ohne ein
`code`-Feld zurückgeben.

### Playwright-Anforderung

Einige Funktionen (Navigation/Aktion/KI-Snapshot/Rollen-Snapshot, Element-Screenshots,
PDF) erfordern Playwright. Wenn Playwright nicht installiert ist, geben diese Endpunkte
einen eindeutigen 501-Fehler zurück.

Was weiterhin ohne Playwright funktioniert:

- ARIA-Snapshots
- Barrierefreiheits-Snapshots im Rollenstil (`--interactive`, `--compact`,
  `--depth`, `--efficient`), wenn ein tabbezogener CDP-WebSocket verfügbar ist. Dies ist
  eine Ausweichlösung für die Inspektion und Ref-Ermittlung; Playwright bleibt die primäre
  Aktions-Engine.
- Seiten-Screenshots für den verwalteten `openclaw`-Browser, wenn ein tabbezogener CDP-
  WebSocket verfügbar ist
- Seiten-Screenshots für `existing-session`-/Chrome-MCP-Profile
- `existing-session`-Ref-basierte Screenshots (`--ref`) aus der Snapshot-Ausgabe

Was weiterhin Playwright erfordert:

- `navigate`
- `act`
- KI-Snapshots, die vom nativen KI-Snapshot-Format von Playwright abhängen
- Element-Screenshots mit CSS-Selektor (`--element`)
- vollständiger Browser-PDF-Export

Element-Screenshots lehnen außerdem `--full-page` ab; die Route gibt `fullPage is
not supported for element screenshots` zurück.

Wenn `Playwright is not available in this gateway build` angezeigt wird, fehlt dem paketierten
Gateway die zentrale Browser-Laufzeitabhängigkeit. Installieren oder aktualisieren Sie
OpenClaw und starten Sie anschließend das Gateway neu. Installieren Sie für Docker außerdem die Chromium-
Browser-Binärdateien wie unten dargestellt.

#### Docker-Playwright-Installation

Wenn Ihr Gateway in Docker ausgeführt wird, vermeiden Sie `npx playwright` (Konflikte mit npm-Überschreibungen).
Integrieren Sie bei benutzerdefinierten Images Chromium in das Image:

```bash
OPENCLAW_INSTALL_BROWSER=1 ./scripts/docker/setup.sh
```

Installieren Sie bei einem vorhandenen Image stattdessen über die mitgelieferte CLI:

```bash
docker compose run --rm openclaw-cli \
  node /app/node_modules/playwright-core/cli.js install chromium
```

Um Browser-Downloads persistent zu speichern, setzen Sie `PLAYWRIGHT_BROWSERS_PATH` (zum Beispiel
`/home/node/.cache/ms-playwright`) und stellen Sie sicher, dass `/home/node` über
`OPENCLAW_HOME_VOLUME` oder einen Bind-Mount persistent gespeichert wird. OpenClaw erkennt das persistierte
Chromium unter Linux automatisch. Siehe [Docker](/de/install/docker).

## Funktionsweise (intern)

Ein kleiner Loopback-Steuerungsserver nimmt HTTP-Anfragen entgegen und stellt über CDP eine Verbindung zu Chromium-basierten Browsern her. Erweiterte Aktionen (Klicken/Eingeben/Snapshot/PDF) werden über Playwright auf CDP ausgeführt; wenn Playwright fehlt, sind nur Vorgänge ohne Playwright verfügbar. Der Agent verwendet eine einheitliche stabile Schnittstelle, während lokale und entfernte Browser sowie Profile darunter beliebig ausgetauscht werden können.

## CLI-Kurzreferenz

Alle Befehle akzeptieren `--browser-profile <name>`, um ein bestimmtes Profil auszuwählen, und `--json` für maschinenlesbare Ausgaben.

<AccordionGroup>

<Accordion title="Grundlagen: Status, Tabs, Öffnen/Fokussieren/Schließen">

```bash
openclaw browser status
openclaw browser doctor
openclaw browser doctor --deep    # Live-Snapshot-Prüfung hinzufügen
openclaw browser start
openclaw browser start --headless # einmaliger lokaler verwalteter Headless-Start
openclaw browser stop            # löscht auch die Emulation bei reiner Anbindung/Remote-CDP
openclaw browser reset-profile   # verschiebt die Browserdaten des Profils in den Papierkorb
openclaw browser tabs
openclaw browser tab             # Kurzform für den aktuellen Tab
openclaw browser tab new
openclaw browser tab new --label research
openclaw browser tab label abcd1234 research
openclaw browser tab select 2
openclaw browser tab close 2
openclaw browser open https://example.com
openclaw browser focus abcd1234
openclaw browser close abcd1234
```

</Accordion>

<Accordion title="Profile: Auflisten, Erstellen, Löschen">

```bash
openclaw browser profiles
openclaw browser create-profile --name research --color "#0066CC"
openclaw browser create-profile --name attach --driver existing-session --cdp-url http://127.0.0.1:9222
openclaw browser delete-profile --name research
```

</Accordion>

<Accordion title="Inspektion: Screenshot, Snapshot, Konsole, Fehler, Anfragen">

```bash
openclaw browser screenshot
openclaw browser screenshot --full-page
openclaw browser screenshot --ref 12        # oder --ref e12
openclaw browser screenshot --labels
openclaw browser snapshot
openclaw browser snapshot --format aria --limit 200
openclaw browser snapshot --interactive --compact --depth 6
openclaw browser snapshot --efficient
openclaw browser snapshot --labels
openclaw browser snapshot --urls
openclaw browser snapshot --selector "#main" --interactive
openclaw browser snapshot --frame "iframe#main" --interactive
openclaw browser snapshot --out snapshot.txt
openclaw browser console --level error
openclaw browser errors --clear
openclaw browser requests --filter api --clear
openclaw browser pdf
openclaw browser responsebody "**/api" --max-chars 5000
```

</Accordion>

<Accordion title="Aktionen: Navigieren, Klicken, Eingeben, Ziehen, Warten, Auswerten">

```bash
openclaw browser navigate https://example.com
openclaw browser resize 1280 720
openclaw browser click 12 --double           # oder e12 für Rollen-Refs
openclaw browser click-coords 120 340        # Viewport-Koordinaten
openclaw browser type 23 "hello" --submit
openclaw browser press Enter
openclaw browser hover 44
openclaw browser scrollintoview e12
openclaw browser drag 10 11
openclaw browser select 9 OptionA OptionB
openclaw browser download e12 report.pdf
openclaw browser waitfordownload report.pdf
openclaw browser upload /tmp/openclaw/uploads/file.pdf
openclaw browser upload /tmp/openclaw/uploads/file.pdf --ref e12
openclaw browser upload media://inbound/file.pdf
openclaw browser fill --fields '[{"ref":"1","type":"text","value":"Ada"}]'
openclaw browser dialog --accept
openclaw browser dialog --dismiss --dialog-id d1
openclaw browser wait --text "Done"
openclaw browser wait "#main" --url "**/dash" --load networkidle --fn "window.ready===true"
openclaw browser evaluate --fn '(el) => el.textContent' --ref 7
openclaw browser evaluate --fn 'const title = document.title; return title;'
openclaw browser evaluate --timeout-ms 30000 --fn 'async () => { await window.ready; return true; }'
openclaw browser highlight e12
openclaw browser trace start
openclaw browser trace stop
```

</Accordion>

<Accordion title="Zustand: Cookies, Speicher, Offline-Modus, Header, Standort, Gerät">

```bash
openclaw browser cookies
openclaw browser cookies set session abc123 --url "https://example.com"
openclaw browser cookies clear
openclaw browser storage local get
openclaw browser storage local set theme dark
openclaw browser storage session clear
openclaw browser set offline on
openclaw browser set headers --headers-json '{"X-Debug":"1"}'
openclaw browser set credentials user pass            # mit --clear entfernen
openclaw browser set geo 37.7749 -122.4194 --origin "https://example.com"
openclaw browser set media dark
openclaw browser set timezone America/New_York
openclaw browser set locale en-US
openclaw browser set device "iPhone 14"
```

</Accordion>

</AccordionGroup>

Hinweise:

- Das agentenseitige Tool `browser` stellt `action=download` (erforderlich: `ref` und
  `path`) sowie `action=waitfordownload` (optional: `path`) bereit. Beide geben die gespeicherte
  Download-URL, den vorgeschlagenen Dateinamen und den abgesicherten lokalen Pfad zurück. Das explizite Abfangen von
  Downloads ist für verwaltete Playwright-Profile verfügbar; Profile mit vorhandener Sitzung
  geben einen Fehler wegen eines nicht unterstützten Vorgangs zurück.
- Bevorzugen Sie atomare Uploads über die Dateiauswahl: Übergeben Sie den Auslöser `--ref` zusammen mit dem Upload, damit OpenClaw die Dateiauswahl in einer Anfrage vorbereitet und anklickt. `upload` nur mit Pfaden wird weiterhin unterstützt, wenn ein späterer Auslöser beabsichtigt ist. Verwenden Sie `--input-ref` oder `--element`, um eine Dateieingabe direkt festzulegen. `dialog` ist ein Vorbereitungsaufruf; führen Sie ihn vor dem Klick oder Tastendruck aus, der den Dialog auslöst. Wenn eine Aktion ein modales Fenster öffnet, enthält die Aktionsantwort `blockedByDialog` und `browserState.dialogs.pending`; übergeben Sie dieses `dialogId`, um direkt zu antworten. Außerhalb von OpenClaw behandelte Dialoge werden unter `browserState.dialogs.recent` angezeigt.
- `click`/`type`/usw. erfordern ein `ref` aus `snapshot` (numerisches `12`, Rollen-Ref `e12` oder ausführbares ARIA-Ref `ax12`). CSS-Selektoren werden für Aktionen bewusst nicht unterstützt. Verwenden Sie `click-coords`, wenn die sichtbare Position im Viewport das einzig zuverlässige Ziel ist.
- Download- und Trace-Pfade sind auf temporäre OpenClaw-Stammverzeichnisse beschränkt: `/tmp/openclaw{,/downloads}` (Fallback: `${os.tmpdir()}/openclaw/...`).
- `upload` akzeptiert Dateien aus dem temporären Upload-Stammverzeichnis von OpenClaw und
  von OpenClaw verwaltete eingehende Medien. Verwaltete eingehende Medien können als
  `media://inbound/<id>`, sandboxrelatives `media/inbound/<id>` oder als aufgelöster
  Pfad innerhalb des Verzeichnisses für verwaltete eingehende Medien referenziert werden. Verschachtelte Medien-Refs,
  Verzeichnisdurchquerung, symbolische Links, harte Links und beliebige lokale Pfade werden weiterhin abgelehnt.
- `upload` kann Dateieingaben auch direkt über `--input-ref` oder `--element` festlegen.

Stabile Tab-IDs und -Bezeichnungen bleiben beim Ersetzen von Chromium-Raw-Targets erhalten, wenn OpenClaw
den Ersatz-Tab eindeutig bestimmen kann, etwa anhand eines eindeutigen alten/neuen Paars für dieselbe URL oder
wenn nach dem Absenden eines Formulars aus einem einzelnen alten Tab ein einzelner neuer Tab wird. Mehrdeutige
Ersetzungen mit identischer URL erhalten neue Handles. Raw-Target-IDs bleiben
flüchtig; bevorzugen Sie in Skripten `suggestedTargetId` aus `tabs`.

Snapshot-Flags im Überblick:

- `--format ai` (Standard mit Playwright): KI-Snapshot mit numerischen Refs (`aria-ref="<n>"`).
- `--format aria`: Barrierefreiheitsbaum mit `axN`-Refs. Wenn Playwright verfügbar ist, bindet OpenClaw Refs mithilfe von Backend-DOM-IDs an die aktive Seite, sodass nachfolgende Aktionen sie verwenden können; andernfalls ist die Ausgabe ausschließlich zur Überprüfung bestimmt.
- `--efficient` (oder `--mode efficient`): kompakte Voreinstellung für Rollen-Snapshots. Legen Sie `browser.snapshotDefaults.mode: "efficient"` fest, um dies zum Standard zu machen (siehe [Gateway-Konfiguration](/de/gateway/configuration-reference#browser)).
- `--interactive`, `--compact`, `--depth`, `--selector` erzwingen einen Rollen-Snapshot mit `ref=e12`-Refs. `--frame "<iframe>"` beschränkt Rollen-Snapshots auf einen iframe.
- Mit Playwright fügt `--labels` einen Screenshot mit überlagerten Ref-Bezeichnungen hinzu
  (gibt `MEDIA:<path>` aus), ergänzt um ein `annotations`-Array mit dem Begrenzungsrahmen
  jedes Refs. Bei `screenshot` funktionieren Playwright-gestützte Bezeichnungen mit `--full-page`,
  `--ref` und `--element`; bei `snapshot` bleibt der zugehörige Screenshot
  auf den Viewport beschränkt. Profile mit vorhandener Sitzung bzw. chrome-mcp-Profile rendern überlagerte Bezeichnungen auf
  Seiten-Screenshots, geben jedoch kein `annotations` zurück und verwenden nicht den Playwright-
  Projektionshelfer für vollständige Seiten, Refs oder Elemente. Ohne Playwright oder chrome-mcp
  sind beschriftete Screenshots nicht verfügbar.
- `--urls` hängt erkannte Linkziele an KI-Snapshots an.

## Snapshots und Refs

OpenClaw unterstützt zwei „Snapshot“-Stile:

- **KI-Snapshot (numerische Refs)**: `openclaw browser snapshot` (Standard; `--format ai`)
  - Ausgabe: ein Text-Snapshot mit numerischen Refs.
  - Aktionen: `openclaw browser click 12`, `openclaw browser type 23 "hello"`.
  - Intern wird das Ref über `aria-ref` von Playwright aufgelöst.

- **Rollen-Snapshot (Rollen-Refs wie `e12`)**: `openclaw browser snapshot --interactive` (oder `--compact`, `--depth`, `--selector`, `--frame`)
  - Ausgabe: eine rollenbasierte Liste bzw. Baumstruktur mit `[ref=e12]` (und optional `[nth=1]`).
  - Aktionen: `openclaw browser click e12`, `openclaw browser highlight e12`.
  - Intern wird das Ref über `getByRole(...)` aufgelöst (zuzüglich `nth()` bei Duplikaten).
  - Fügen Sie `--labels` hinzu, um einen Screenshot mit überlagerten `e12`-Bezeichnungen einzuschließen. Bei
    Playwright-gestützten Profilen werden dadurch außerdem Metadaten zum Begrenzungsrahmen jedes Refs zurückgegeben
    (`annotations[]`).
  - Fügen Sie `--urls` hinzu, wenn der Linktext mehrdeutig ist und der Agent konkrete
    Navigationsziele benötigt.

- **ARIA-Snapshot (ARIA-Refs wie `ax12`)**: `openclaw browser snapshot --format aria`
  - Ausgabe: der Barrierefreiheitsbaum als strukturierte Nodes.
  - Aktionen: `openclaw browser click ax12` funktioniert, wenn der Snapshot-Pfad
    das Ref über Playwright und Chrome-Backend-DOM-IDs binden kann.
- Wenn Playwright nicht verfügbar ist, können ARIA-Snapshots weiterhin
  zur Überprüfung nützlich sein, die Refs sind jedoch möglicherweise nicht ausführbar. Erstellen Sie mit `--format ai`
  oder `--interactive` einen neuen Snapshot, wenn Sie Aktions-Refs benötigen.
- Docker-Nachweis für den Raw-CDP-Fallback-Pfad: `pnpm test:docker:browser-cdp-snapshot`
  startet Chromium mit CDP, führt `browser doctor --deep` aus und verifiziert, dass Rollen-
  Snapshots Link-URLs, durch den Cursor als anklickbar erkannte Elemente und iframe-Metadaten enthalten.

Verhalten von Refs:

- Refs sind **über Navigationen hinweg nicht stabil**; wenn etwas fehlschlägt, führen Sie `snapshot` erneut aus und verwenden Sie ein neues Ref.
- `/act` gibt nach einem durch eine Aktion ausgelösten Austausch das aktuelle Raw-`targetId` zurück,
  wenn der Ersatz-Tab eindeutig bestimmt werden kann. Verwenden Sie für
  nachfolgende Befehle weiterhin stabile Tab-IDs und -Bezeichnungen.
- Wenn der Rollen-Snapshot mit `--frame` erstellt wurde, sind Rollen-Refs bis zum nächsten Rollen-Snapshot auf diesen iframe beschränkt.
- Unbekannte oder veraltete `axN`-Refs schlagen sofort fehl, statt auf
  den `aria-ref`-Selektor von Playwright zurückzufallen. Erstellen Sie in diesem Fall
  einen neuen Snapshot desselben Tabs.

## Browser-Batch-CLI

`openclaw browser batch` führt ein Array verschachtelter `/act`-Aktionen in einem einzigen `/act`-
Aufruf aus (dieselbe über das Agenten-Tool erreichbare `kind="batch"`-Laufzeit), sodass CLI-
Benutzer und Skripte Aktionen wie `wait`, `click`, `type` und
`evaluate` ohne Roundtrips für einzelne Aktionen zu einem einzigen wiederholbaren Plan kombinieren können. Jeder
Eintrag in `actions[]` ist ein `BrowserActRequest` – die abgeschlossene Union, die die `/act`-
Route akzeptiert (`click`, `clickCoords`, `type`, `press`, `hover`,
`scrollIntoView`, `drag`, `select`, `fill`, `resize`, `wait`, `evaluate`,
`close`, `batch`) – und keine beliebigen `openclaw browser`-Unterbefehle. `batch` wird
bei `profile="user"` und anderen Profilen mit vorhandener Sitzung (chrome-mcp)
nicht unterstützt; senden Sie die Aktionen dort einzeln.

- CLI: `openclaw browser batch --actions '<json>'`, `openclaw browser batch
--actions-file plan.json` oder `openclaw browser batch --actions-file -`, um
  das JSON-Array aus stdin zu lesen. `--continue` legt `stopOnError=false` fest;
  standardmäßig wird beim ersten Fehler abgebrochen. `--target-id` beschränkt den gesamten Batch auf
  einen Tab.
- Ref-Lebenszyklus: Refs stammen aus einer vor dem Batch ausgeführten `snapshot`-Ausführung (Snapshot ist
  keine verschachtelte Aktion). Eine verschachtelte Aktion, die den Seitenzustand ändert – etwa ein
  `click`, das eine Navigation auslöst, oder ein `evaluate`, das das DOM verändert – kann
  frühere Refs für den Rest des Batches ungültig machen. Platzieren Sie zustandsändernde Aktionen
  zuerst oder teilen Sie sie nach der erneuten Snapshot-Erstellung in einen nachfolgenden Batch auf. Navigation und
  erneute Snapshot-Erstellung finden außerhalb des Batches statt (`openclaw browser navigate` /
  `snapshot`), da `open`, `navigate` und `snapshot` keine `/act`-Arten sind.
- Konflikte bei Ziel-IDs: Eine verschachtelte Aktion kann `targetId` auslassen oder das
  `targetId` auf Anfrageebene wiederholen; ein explizites verschachteltes `targetId`, das zu einem
  anderen Tab aufgelöst wird, wird mit `ACT_TARGET_ID_MISMATCH` abgelehnt, bevor eine Aktion
  ausgeführt wird. Batch-Aktionen verwenden konstruktionsbedingt gemeinsam den Tab der Anfrage.
- Fehlerzusammenfassung: Die Antwort ist `{ "results": [{ "ok": true }, { "ok": false,
"error": "<message>" }, ...] }`, mit einem Eintrag pro Aktion in der ursprünglichen Reihenfolge. Wenn
  `stopOnError` der Standardwert ist, endet das Array beim ersten Fehler; mit
  `--continue` umfasst es jede Aktion. Jeder fehlgeschlagene Eintrag bewirkt, dass die CLI
  mit einem von null verschiedenen Status beendet wird; übergeben Sie `--json`, um die vollständige geordnete Antwort für Skripte beizubehalten.

## Erweiterte Wartefunktionen

Sie können nicht nur auf Zeit oder Text warten:

- Auf URL warten (Globs werden von Playwright unterstützt):
  - `openclaw browser wait --url "**/dash"`
- Auf Ladestatus warten:
  - `openclaw browser wait --load networkidle`
  - Unterstützt bei verwalteten `openclaw`- und Raw-/Remote-CDP-Profilen. Profile, die den `existing-session`-Treiber verwenden (einschließlich des standardmäßigen `user`-Profils), lehnen `networkidle` ab; verwenden Sie dort `--url`, `--text`, einen Selektor oder `--fn`-Wartevorgänge.
- Auf ein JS-Prädikat warten:
  - `openclaw browser wait --fn "window.ready===true"`
- Warten, bis ein Selektor sichtbar wird:
  - `openclaw browser wait "#main"`

Diese Optionen können kombiniert werden:

```bash
openclaw browser wait "#main" \
  --url "**/dash" \
  --load networkidle \
  --fn "window.ready===true" \
  --timeout-ms 15000
```

## Debugging-Abläufe

Wenn eine Aktion fehlschlägt (z. B. „nicht sichtbar“, „Verletzung des strikten Modus“, „verdeckt“):

1. `openclaw browser snapshot --interactive`
2. Verwenden Sie `click <ref>` / `type <ref>` (bevorzugen Sie im interaktiven Modus Rollen-Refs)
3. Wenn sie weiterhin fehlschlägt: `openclaw browser highlight <ref>`, um zu sehen, worauf Playwright zielt
4. Wenn sich die Seite ungewöhnlich verhält:
   - `openclaw browser errors --clear`
   - `openclaw browser requests --filter api --clear`
5. Für eine tiefgehende Fehlersuche: Zeichnen Sie einen Trace auf:
   - `openclaw browser trace start`
   - Reproduzieren Sie das Problem
   - `openclaw browser trace stop` (gibt `TRACE:<path>` aus)

## JSON-Ausgabe

`--json` ist für Skripterstellung und strukturierte Werkzeuge vorgesehen.

Beispiele:

```bash
openclaw browser --json status
openclaw browser --json snapshot --interactive
openclaw browser --json requests --filter api
openclaw browser --json cookies
```

Rollen-Snapshots im JSON enthalten `refs` sowie einen kleinen `stats`-Block (Zeilen/Zeichen/Refs/interaktiv), damit Werkzeuge Größe und Dichte der Nutzdaten beurteilen können.

## Zustands- und Umgebungsoptionen

Diese sind für Abläufe nach dem Muster „Die Website soll sich wie X verhalten“ nützlich:

- Cookies: `cookies`, `cookies set`, `cookies clear`
- Speicher: `storage local|session get|set|clear`
- Offline: `set offline on|off`
- Header: `set headers --headers-json '{"X-Debug":"1"}'` (oder die positionale Form `set headers '{"X-Debug":"1"}'`)
- HTTP-Basisauthentifizierung: `set credentials user pass` (oder `--clear`)
- Geolokalisierung: `set geo <lat> <lon> --origin "https://example.com"` (oder `--clear`)
- Medien: `set media dark|light|no-preference|none`
- Zeitzone / Gebietsschema: `set timezone ...`, `set locale ...`
- Gerät / Viewport:
  - `set device "iPhone 14"` (Playwright-Gerätevoreinstellungen)
  - `set viewport 1280 720`

## Sicherheit und Datenschutz

- Das openclaw-Browserprofil kann angemeldete Sitzungen enthalten; behandeln Sie es als vertraulich.
- `browser act kind=evaluate` / `openclaw browser evaluate` und `wait --fn`
  führen beliebiges JavaScript im Seitenkontext aus. Prompt-Injection kann dies
  steuern. Deaktivieren Sie es mit `browser.evaluateEnabled=false`, wenn Sie es nicht benötigen.
- `openclaw browser evaluate --fn` akzeptiert den Quelltext einer Funktion, einen Ausdruck oder
  einen Anweisungsblock. Anweisungsblöcke werden als asynchrone Funktionen umschlossen; verwenden Sie daher
  `return` für den Wert, den Sie zurückerhalten möchten. Verwenden Sie `--timeout-ms <ms>`, wenn die
  Funktion auf der Seite möglicherweise länger als das standardmäßige Zeitlimit für die Auswertung benötigt.
- Hinweise zu Anmeldungen und Anti-Bot-Maßnahmen (X/Twitter usw.) finden Sie unter [Browser-Anmeldung und Beiträge auf X/Twitter](/de/tools/browser-login).
- Halten Sie den Gateway-/Node-Host privat (nur Loopback oder Tailnet).
- Remote-CDP-Endpunkte sind leistungsfähig; tunneln und schützen Sie sie.

Beispiel für den strikten Modus (private/interne Ziele standardmäßig blockieren):

```json5
{
  browser: {
    ssrfPolicy: {
      dangerouslyAllowPrivateNetwork: false,
      hostnameAllowlist: ["*.example.com", "example.com"],
      allowedHostnames: ["localhost"], // optional exact allow
    },
  },
}
```

## Verwandte Themen

- [Browser](/de/tools/browser) – Übersicht, Konfiguration, Profile, Sicherheit
- [Browser-Anmeldung](/de/tools/browser-login) – Anmeldung bei Websites
- [Fehlerbehebung für Browser unter Linux](/de/tools/browser-linux-troubleshooting)
- [Fehlerbehebung für Browser unter WSL2](/de/tools/browser-wsl2-windows-remote-cdp-troubleshooting)
