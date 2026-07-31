---
read_when:
    - Sie möchten, dass Agenten Code- oder Markdown-Änderungen als Diffs anzeigen.
    - Sie benötigen eine Canvas-fähige Viewer-URL oder eine gerenderte Diff-Datei
    - Sie benötigen kontrollierte, temporäre Diff-Artefakte mit sicheren Standardeinstellungen
sidebarTitle: Diffs
summary: Schreibgeschützter Diff-Viewer und Datei-Renderer für Agenten (optionales Plugin-Tool)
title: Differenzen
x-i18n:
    generated_at: "2026-07-26T18:40:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: baeb5dd1277120e57178f092e3ae1616edd3389a54721c929d8711301535d302
    source_path: tools/diffs.md
    workflow: 16
---

`diffs` ist ein optionales gebündeltes Plugin-Tool, das Vorher-/Nachher-Text oder einen vereinheitlichten Patch in ein schreibgeschütztes Diff-Artefakt umwandelt. Es stellt dem System-Prompt außerdem kurze Agent-Anweisungen voran und enthält ein begleitendes Skill mit ausführlicheren Anweisungen.

Eingabe: `before`- und `after`-Text oder ein vereinheitlichter `patch` (schließen sich gegenseitig aus).

Ausgabe: eine Gateway-Viewer-URL für die Canvas-Darstellung, ein gerenderter PNG-/PDF-Dateipfad für die Nachrichtenzustellung oder beides.

## Schnellstart

<Steps>
  <Step title="Plugin installieren">
    ```bash
    openclaw plugins install diffs
    ```
  </Step>
  <Step title="Plugin aktivieren">
    ```json5
    {
      plugins: {
        entries: {
          diffs: {
            enabled: true,
          },
        },
      },
    }
    ```
  </Step>
  <Step title="Modus auswählen">
    <Tabs>
      <Tab title="view">
        Canvas-orientierte Abläufe: Agenten rufen `diffs` mit `mode: "view"` auf und öffnen `details.viewerUrl` mit `canvas present`.
      </Tab>
      <Tab title="file">
        Zustellung von Chat-Dateien: Agenten rufen `diffs` mit `mode: "file"` auf und senden `details.filePath` mit `message` unter Verwendung von `path` oder `filePath`.
      </Tab>
      <Tab title="both">
        Kombiniert (Standard): Agenten rufen `diffs` mit `mode: "both"` auf, um beide Artefakte mit einem einzigen Aufruf abzurufen.
      </Tab>
    </Tabs>
  </Step>
</Steps>

## Integrierte Systemanweisungen deaktivieren

Um das Tool beizubehalten, aber die vorangestellten System-Prompt-Anweisungen zu entfernen, setzen Sie `plugins.entries.diffs.hooks.allowPromptInjection` auf `false`:

```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
        hooks: {
          allowPromptInjection: false,
        },
      },
    },
  },
}
```

Dadurch wird der `before_prompt_build`-Hook des Plugins blockiert, während das Tool und das Skill verfügbar bleiben. Um sowohl die Anweisungen als auch das Tool zu deaktivieren, deaktivieren Sie stattdessen das Plugin.

## Referenz der Tool-Eingaben

Alle Felder sind optional, sofern nicht anders angegeben.

<ParamField path="before" type="string">
  Ursprünglicher Text. Zusammen mit `after` erforderlich, wenn `patch` ausgelassen wird.
</ParamField>
<ParamField path="after" type="string">
  Aktualisierter Text. Zusammen mit `before` erforderlich, wenn `patch` ausgelassen wird.
</ParamField>
<ParamField path="patch" type="string">
  Vereinheitlichter Diff-Text. Schließt sich mit `before` und `after` gegenseitig aus.
</ParamField>
<ParamField path="path" type="string">
  Angezeigter Dateiname für den Vorher-/Nachher-Modus.
</ParamField>
<ParamField path="lang" type="string">
  Hinweis zum Überschreiben der Sprache für den Vorher-/Nachher-Modus. Unbekannte Werte und Sprachen außerhalb des standardmäßigen Viewer-Satzes greifen auf Klartext zurück, sofern das
  Diff Viewer Language Pack-Plugin nicht installiert ist.
</ParamField>
<ParamField path="title" type="string">
  Überschreibung des Viewer-Titels.
</ParamField>
<ParamField path="mode" type='"view" | "file" | "both"'>
  Ausgabemodus. Standardmäßig wird der Plugin-Standardwert `defaults.mode` (`both`) verwendet. Veralteter Alias: `"image"` verhält sich identisch zu `"file"`.
</ParamField>
<ParamField path="theme" type='"light" | "dark"'>
  Viewer-Theme. Standardmäßig wird der Plugin-Standardwert `defaults.theme` verwendet.
</ParamField>
<ParamField path="layout" type='"unified" | "split"'>
  Diff-Layout. Standardmäßig wird der Plugin-Standardwert `defaults.layout` verwendet.
</ParamField>
<ParamField path="expandUnchanged" type="boolean">
  Unveränderte Abschnitte erweitern, wenn der vollständige Kontext verfügbar ist. Nur Option pro Aufruf (kein Plugin-Standardschlüssel).
</ParamField>
<ParamField path="fileFormat" type='"png" | "pdf"'>
  Gerendertes Dateiformat. Standardmäßig wird der Plugin-Standardwert `defaults.fileFormat` verwendet.
</ParamField>
<ParamField path="fileQuality" type='"standard" | "hq" | "print"'>
  Qualitätsvoreinstellung für das PNG-/PDF-Rendering.
</ParamField>
<ParamField path="fileScale" type="number">
  Überschreibung der Geräteskalierung (`1`-`4`).
</ParamField>
<ParamField path="fileMaxWidth" type="number">
  Maximale Rendering-Breite in CSS-Pixeln (`640`-`2400`).
</ParamField>
<ParamField path="ttlSeconds" type="number" default="1800">
  Artefakt-TTL in Sekunden für Viewer- und eigenständige Dateiausgaben. Maximal `21600`.
</ParamField>
<ParamField path="baseUrl" type="string">
  Überschreibung des Ursprungs der Viewer-URL. Überschreibt Plugin-`viewerBaseUrl`. Muss `http` oder `https` sein, ohne Abfrage/Hash.
</ParamField>

<AccordionGroup>
  <Accordion title="Validierung und Grenzwerte">
    - `before`/`after`: jeweils maximal 512 KiB.
    - `patch`: maximal 2 MiB.
    - `path`: maximal 2048 Byte.
    - `lang`: maximal 128 Byte.
    - `title`: maximal 1024 Byte.
    - Komplexitätsgrenze für Patches: maximal 128 Dateien und insgesamt 120000 Zeilen.
    - `patch` zusammen mit `before`/`after` wird abgelehnt.
    - Sicherheitsgrenzen für gerenderte Dateien (PNG und PDF):
      - `fileQuality: "standard"`: maximal 8 MP (8,000,000 gerenderte Pixel).
      - `fileQuality: "hq"`: maximal 14 MP.
      - `fileQuality: "print"`: maximal 24 MP.
      - PDF ist außerdem auf 50 Seiten begrenzt.

  </Accordion>
</AccordionGroup>

## Syntaxhervorhebung

Integrierte Sprachen:

`javascript`, `typescript`, `tsx`, `jsx`, `json`, `markdown`, `yaml`, `css`, `html`, `sh`, `python`, `go`, `rust`, `java`, `c`, `cpp`, `csharp`, `php`, `sql`, `docker`, `ruby`, `swift`, `kotlin`, `r`, `dart`, `lua`, `powershell`, `xml` und `toml`.

Gängige Aliasse (`js`, `ts`, `bash`, `md`, `yml`, `c++`, `dockerfile`, `rb`, `kt`, `ps1` usw.) werden auf diese Sprachen normalisiert.

Installieren Sie das Diff Viewer Language Pack-Plugin für weitere Sprachen (Astro, Vue, Svelte, MDX, GraphQL, Terraform/HCL, Nix, Clojure, Elixir, Haskell, OCaml, Scala, Zig, Solidity, Verilog/VHDL, Fortran, MATLAB, LaTeX, Mermaid, Sass/Less/SCSS, Nginx, Apache, CSV, dotenv, INI, diff und weitere):

```bash
openclaw plugins install clawhub:@openclaw/diffs-language-pack
```

Ohne das Pack werden nicht unterstützte Sprachen weiterhin als lesbarer Klartext gerendert. Den vorgelagerten Katalog finden Sie unter [Diffs Language Pack-Plugin](/de/plugins/reference/diffs-language-pack) und [Shiki-Sprachen](https://shiki.style/languages).

## Vertrag für Ausgabedetails

Alle erfolgreichen Ergebnisse enthalten `changed`: Bei identischen Vorher-/Nachher-Eingaben wird `false` zurückgegeben, ohne ein Artefakt zu erstellen; gerenderte Ergebnisse geben `true` zurück.

<AccordionGroup>
  <Accordion title="Viewer-Felder (Modi view und both)">
    - `changed`
    - `artifactId`
    - `viewerUrl`
    - `viewerPath`
    - `title`
    - `expiresAt`
    - `inputKind`
    - `fileCount`
    - `mode`
    - `context` (`agentId`, `sessionId`, `messageChannel`, `agentAccountId`, sofern verfügbar)

  </Accordion>
  <Accordion title="Dateifelder (Modi file und both)">
    - `changed`
    - `artifactId`
    - `expiresAt`
    - `filePath`
    - `path` (derselbe Wert wie `filePath`, für die Kompatibilität mit dem Nachrichten-Tool)
    - `fileBytes`
    - `fileFormat`
    - `fileQuality`
    - `fileScale`
    - `fileMaxWidth`

  </Accordion>
</AccordionGroup>

| Modus     | Rückgabe                                                                                         |
| -------- | ----------------------------------------------------------------------------------------------- |
| `"view"` | Nur Viewer-Felder.                                                                             |
| `"file"` | Nur Dateifelder, kein Viewer-Artefakt.                                                           |
| `"both"` | Viewer-Felder plus Dateifelder. Wenn das Datei-Rendering fehlschlägt, wird der Viewer dennoch mit `fileError` zurückgegeben. |

### Reduzierte unveränderte Abschnitte

Der Viewer zeigt Zeilen wie `N unmodified lines`. Steuerelemente zum Erweitern werden nur angezeigt, wenn das gerenderte Diff erweiterbare Kontextdaten enthält (typischerweise bei Vorher-/Nachher-Eingaben). Bei vielen vereinheitlichten Patches fehlen die Kontextinhalte in den Hunk-Abschnitten, sodass die Zeile ohne Steuerelement zum Erweitern erscheinen kann – dies ist erwartetes Verhalten und kein Fehler. `expandUnchanged` gilt nur, wenn erweiterbarer Kontext vorhanden ist.

### Navigation zwischen mehreren Dateien

Patches, die mehr als eine Datei betreffen, beginnen mit einer Übersichtskarte der geänderten Dateien: Gesamtzahlen für `+N` / `-N`, Zahlen pro Datei, Kennzeichnungen für hinzugefügt/gelöscht/umbenannt und Ankerlinks, die zu jeder Datei springen. Gerenderte PNG-/PDF-Dateien behalten die Zahlen in den Dateikopfzeilen bei, lassen jedoch die interaktiven Ansichtsumschalter weg, da diese in einer statischen Datei funktionslose Steuerelemente wären.

## Plugin-Standardwerte

Legen Sie Plugin-weite Standardwerte in `~/.openclaw/openclaw.json` fest:

```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
        config: {
          defaults: {
            fontFamily: "Fira Code",
            fontSize: 15,
            lineSpacing: 1.6,
            layout: "unified",
            showLineNumbers: true,
            diffIndicators: "bars",
            wordWrap: true,
            background: true,
            theme: "dark",
            fileFormat: "png",
            fileQuality: "standard",
            fileScale: 2,
            fileMaxWidth: 960,
            mode: "both",
            ttlSeconds: 21600,
          },
        },
      },
    },
  },
}
```

Unterstützte `defaults`-Schlüssel: `fontFamily`, `fontSize`, `lineSpacing`, `layout`, `showLineNumbers`, `diffIndicators`, `wordWrap`, `background`, `theme`, `fileFormat`, `fileQuality`, `fileScale`, `fileMaxWidth`, `mode`, `ttlSeconds`. Explizite Parameter des Tool-Aufrufs überschreiben diese Werte.

### Konfiguration einer dauerhaften Viewer-URL

<ParamField path="viewerBaseUrl" type="string">
  Plugin-eigener Rückfallwert für zurückgegebene Viewer-Links, wenn ein Tool-Aufruf `baseUrl` nicht übergibt. Muss `http` oder `https` sein, ohne Abfrage/Hash.
</ParamField>

```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
        config: {
          viewerBaseUrl: "https://gateway.example.com/openclaw",
        },
      },
    },
  },
}
```

## Sicherheitskonfiguration

<ParamField path="security.allowRemoteViewer" type="boolean" default="false">
  `false`: Anfragen von Adressen außerhalb der Loopback-Schnittstelle an Viewer-Routen werden abgelehnt. `true`: Remote-Viewer sind zulässig, wenn der tokenisierte Pfad gültig ist.
</ParamField>

```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
        config: {
          security: {
            allowRemoteViewer: false,
          },
        },
      },
    },
  },
}
```

## Lebenszyklus und Speicherung von Artefakten

- Viewer-HTML und Metadaten befinden sich in der gemeinsam genutzten `state/openclaw.sqlite`-Datenbank im Blob-Namespace des Diffs-Plugins. HTML wird mit gzip komprimiert; SQLite speichert nur einen SHA-256-Hash des zufälligen URL-Tokens, nicht das Token selbst.
- Gerenderte PNG-/PDF-Dateien bleiben temporäre Materialisierungen unter `$TMPDIR/openclaw-diffs`, da die Zustellung über Kanäle einen Dateipfad erfordert. SQLite verwaltet ihre Ablaufmetadaten; es werden keine JSON-Sidecar-Dateien geschrieben.
- Standardmäßige TTL für Artefakte: 30 Minuten. Maximal akzeptierte TTL: 6 Stunden.
- Die Bereinigung wird opportunistisch nach jedem Aufruf zur Artefakterstellung ausgeführt. Abgelaufene SQLite-Zeilen werden zuerst gelöscht, gefolgt von den entsprechenden PNG-/PDF-Verzeichnissen.
- Ein zusätzlicher Bereinigungsdurchlauf entfernt zeilenlose temporäre Ordner, die älter als 24 Stunden sind. Veraltete Caches unter `meta.json`, `file-meta.json` und `viewer.html` werden weder importiert noch gelesen.

## Viewer-URL und Netzwerkverhalten

Viewer-Route: `/plugins/diffs/view/{artifactId}/{token}`

Viewer-Assets:

- `/plugins/diffs/assets/viewer.js`
- `/plugins/diffs/assets/viewer-runtime.js`
- `/plugins/diffs-language-pack/assets/viewer.js` (nur wenn der Diff eine Sprache aus einem Sprachpaket verwendet)

Das Viewer-Dokument löst diese Assets relativ zur Viewer-URL auf, sodass ein optionales Pfadpräfix `baseUrl` auch für Asset-Anfragen übernommen wird.

Reihenfolge der URL-Auflösung: `baseUrl` des Tool-Aufrufs (nach strenger Validierung) -> `viewerBaseUrl` des Plugins -> standardmäßig Loopback `127.0.0.1`. Wenn der Gateway-Bindungsmodus `custom` ist und `gateway.customBindHost` festgelegt wurde, wird dieser Host anstelle von Loopback verwendet.

Regeln für `baseUrl`: muss `http://` oder `https://` sein; Abfrageparameter und Hash werden abgelehnt; der Ursprung mit optionalem Basispfad ist zulässig.

## Sicherheitsmodell

<AccordionGroup>
  <Accordion title="Absicherung des Viewers">
    - Standardmäßig nur über Loopback zugänglich.
    - Tokenisierte Viewer-Pfade mit strenger Validierung der ID- und Token-Muster.
    - CSP der Viewer-Antwort: `default-src 'none'`; Skripte/Assets nur aus derselben Quelle; keine ausgehenden `connect-src`.
    - Drosselung fehlgeschlagener Remote-Zugriffe, wenn der Remote-Zugriff aktiviert ist: 40 Fehlschläge innerhalb von 60 Sekunden lösen eine 60-sekündige Sperre aus (`429 Too Many Requests`).

  </Accordion>
  <Accordion title="Absicherung des Datei-Renderings">
    - Das Routing von Browseranfragen für Screenshots verweigert standardmäßig alle Anfragen.
    - Nur lokale Viewer-Assets aus `http://127.0.0.1/plugins/diffs/assets/*` sind zulässig.
    - Externe Netzwerkanfragen werden blockiert.

  </Accordion>
</AccordionGroup>

## Browseranforderungen für den Dateimodus

`mode: "file"` und `mode: "both"` benötigen einen Chromium-kompatiblen Browser.

Auflösungsreihenfolge:

<Steps>
  <Step title="Konfiguration">
    `browser.executablePath` in der OpenClaw-Konfiguration.
  </Step>
  <Step title="Umgebungsvariablen">
    - `OPENCLAW_BROWSER_EXECUTABLE_PATH`
    - `BROWSER_EXECUTABLE_PATH`
    - `PLAYWRIGHT_CHROMIUM_EXECUTABLE_PATH`

  </Step>
  <Step title="Plattform-Fallback">
    Übliche Installationspfade und `PATH`-Suchvorgänge für Chrome, Chromium, Edge und Brave.
  </Step>
</Steps>

Häufige Fehlermeldung: `Diff PNG/PDF rendering requires a Chromium-compatible browser...`. Installieren Sie zur Behebung Chrome, Chromium, Edge oder Brave, oder legen Sie eine der oben genannten Optionen für den Pfad zur ausführbaren Datei fest.

## Fehlerbehebung

<AccordionGroup>
  <Accordion title="Fehler bei der Eingabevalidierung">
    - `Provide patch or both before and after text.` -- geben Sie sowohl `before` als auch `after` an oder stellen Sie `patch` bereit.
    - `Provide either patch or before/after input, not both.` -- kombinieren Sie keine Eingabemodi.
    - `Invalid baseUrl: ...` -- verwenden Sie einen `http(s)`-Ursprung mit optionalem Pfad, ohne Abfrageparameter/Hash.
    - `{field} exceeds maximum size (...)` -- reduzieren Sie die Nutzlastgröße.
    - Ablehnung eines großen Patches -- reduzieren Sie die Anzahl der Patch-Dateien oder die Gesamtzahl der Zeilen.

  </Accordion>
  <Accordion title="Erreichbarkeit des Viewers">
    - Die Viewer-URL wird standardmäßig zu `127.0.0.1` aufgelöst.
    - Legen Sie für den Remote-Zugriff entweder `viewerBaseUrl` des Plugins fest, übergeben Sie bei jedem Aufruf `baseUrl`, oder verwenden Sie `gateway.bind=custom` mit `gateway.customBindHost`.
    - Wenn `gateway.trustedProxies` Loopback für einen Proxy auf demselben Host umfasst (beispielsweise Tailscale Serve), werden direkte Loopback-Anfragen an den Viewer ohne weitergeleitete Header mit der Client-IP absichtlich nach dem Fail-Closed-Prinzip abgelehnt.
    - Bevorzugen Sie für diese Proxy-Topologie `mode: "file"`/`"both"` für einen Anhang, oder aktivieren Sie für einen teilbaren Viewer-Link gezielt `security.allowRemoteViewer` zusammen mit `viewerBaseUrl` des Plugins/einem Proxy-`baseUrl`.
    - Aktivieren Sie `security.allowRemoteViewer` nur, wenn ein externer Viewer-Zugriff vorgesehen ist.

  </Accordion>
  <Accordion title="Die Zeile für unveränderte Zeilen hat keine Schaltfläche zum Erweitern">
    Dies ist bei Patch-Eingaben ohne erweiterbaren Kontext zu erwarten und kein Viewer-Fehler.
  </Accordion>
  <Accordion title="Artefakt nicht gefunden">
    - Das Artefakt ist aufgrund der TTL abgelaufen.
    - Token oder Pfad wurde geändert.
    - Die Bereinigung hat veraltete Daten entfernt.

  </Accordion>
</AccordionGroup>

## Betriebshinweise

- Bevorzugen Sie `mode: "view"` für lokale interaktive Reviews in Canvas.
- Bevorzugen Sie `mode: "file"` für ausgehende Chatkanäle, die einen Anhang benötigen.
- Lassen Sie `allowRemoteViewer` deaktiviert, sofern Ihre Bereitstellung keine Remote-Viewer-URLs erfordert.
- Legen Sie für vertrauliche Diffs ausdrücklich eine kurze `ttlSeconds` fest.
- Vermeiden Sie es, Geheimnisse in der Diff-Eingabe zu senden, wenn dies nicht erforderlich ist.
- Wenn Ihr Kanal Bilder stark komprimiert (beispielsweise Telegram oder WhatsApp), bevorzugen Sie die PDF-Ausgabe (`fileFormat: "pdf"`).

<Note>
Diff-Rendering-Engine unterstützt von [Diffs](https://diffs.com).
</Note>

## Verwandte Themen

- [Browser](/de/tools/browser)
- [Plugins](/de/tools/plugin)
- [Tools im Überblick](/de/tools)
