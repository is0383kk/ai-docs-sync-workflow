---
read_when:
    - Hinzufügen agentengesteuerter Browserautomatisierung
    - 'Fehlerbehebung: Warum OpenClaw Ihren eigenen Chrome-Browser beeinträchtigt'
    - Implementierung der Browser-Einstellungen und des Lebenszyklus in der macOS-App
summary: Integrierter Dienst zur Browsersteuerung + Aktionsbefehle
title: Browser (von OpenClaw verwaltet)
x-i18n:
    generated_at: "2026-07-26T18:12:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3afa2dda17520ae6c53fe3f1a7a12e7ca8a1414b2c12b79cf4a09ac8906bb3ca
    source_path: tools/browser.md
    workflow: 16
---

OpenClaw kann ein **dediziertes Chrome-/Brave-/Edge-/Chromium-Profil** ausführen, das vom Agenten gesteuert wird. Es wird über einen kleinen lokalen Steuerungsdienst innerhalb des Gateway ausgeführt (nur Loopback) und ist von Ihrem persönlichen Browser isoliert.

- Betrachten Sie es als einen **separaten Browser nur für Agenten**. Das `openclaw`-Profil greift niemals auf Ihr persönliches Browserprofil zu.
- Der Agent öffnet Tabs, liest Seiten, klickt und gibt Text in dieser isolierten Umgebung ein.
- Das integrierte `user`-Profil stellt stattdessen über Chrome DevTools MCP eine Verbindung zu Ihrer tatsächlich angemeldeten Chrome-Sitzung her.

## Funktionsumfang

- Ein separates Browserprofil namens **openclaw** (standardmäßig mit orangefarbener Akzentfarbe).
- Deterministische Tab-Steuerung (auflisten/öffnen/fokussieren/schließen).
- Agentenaktionen (klicken/eingeben/ziehen/auswählen), Snapshots, Screenshots und PDFs.
- Playwright-gestützte Profile speichern Navigationen direkt zu Anhängen im verwalteten Downloadverzeichnis und geben nach der Richtlinienprüfung der endgültigen URL `{ url, suggestedFilename, path }`-Metadaten zurück.
- Playwright-gestützte Agentenaktionen geben ein `downloads`-Array mit denselben verwalteten Metadaten zurück, wenn die Aktion unmittelbar einen oder mehrere Downloads startet.
- Ein gebündeltes `browser-automation`-Skill, das Agenten bei aktiviertem Browser-Plugin den Wiederherstellungsablauf für Snapshots,
  stabile Tabs, veraltete Referenzen und manuelle Blocker vermittelt.
- Optionale Unterstützung mehrerer Profile (`openclaw`, `work`, `remote`, ...).

Dieser Browser ist **nicht** für Ihre tägliche Nutzung vorgesehen. Er bietet eine sichere, isolierte Oberfläche für
Agentenautomatisierung und Verifizierung.

Unter macOS können Sie Cookies explizit aus einem Systemprofil der Chrome-Familie in ein separates verwaltetes Profil kopieren. Der verwaltete Browser verwendet weiterhin sein eigenes Benutzerdatenverzeichnis; nur die ausgewählten Cookies werden kopiert, während lokaler Speicher und IndexedDB zurückbleiben. Importbefehle und Einschränkungen finden Sie unter [Profile](#profiles-multi-browser) oder in der [`openclaw browser`-CLI-Referenz](/de/cli/browser).

## Schnellstart

```bash
openclaw browser --browser-profile openclaw doctor
openclaw browser --browser-profile openclaw doctor --deep
openclaw browser --browser-profile openclaw status
openclaw browser --browser-profile openclaw start
openclaw browser --browser-profile openclaw open https://example.com
openclaw browser --browser-profile openclaw snapshot
```

„Browser deaktiviert“ bedeutet, dass das Plugin oder `browser.enabled` deaktiviert ist; siehe
[Konfiguration](#configuration) und [Plugin-Steuerung](#plugin-control).

Wenn `openclaw browser` vollständig fehlt oder der Agent meldet, dass das Browser-Tool
nicht verfügbar ist, wechseln Sie zu [Fehlender Browserbefehl oder fehlendes Browser-Tool](#missing-browser-command-or-tool).

## Plugin-Steuerung

Das standardmäßige `browser`-Tool ist ein gebündeltes Plugin. Deaktivieren Sie es, um es durch ein anderes Plugin zu ersetzen, das denselben `browser`-Toolnamen registriert:

```json5
{
  plugins: {
    entries: {
      browser: {
        enabled: false,
      },
    },
  },
}
```

Die Standardeinstellungen erfordern sowohl `plugins.entries.browser.enabled` **als auch** `browser.enabled=true`. Wenn nur das Plugin deaktiviert wird, werden die `openclaw browser`-CLI, die `browser.request`-Gateway-Methode, das Agenten-Tool und der Steuerungsdienst als Einheit entfernt; Ihre `browser.*`-Konfiguration bleibt für einen Ersatz erhalten.

Änderungen an der Browserkonfiguration erfordern einen Neustart des Gateway, damit das Plugin seinen Dienst erneut registrieren kann.

## Hinweise für Agenten

Hinweis zum Tool-Profil: `tools.profile: "coding"` umfasst `web_search` und
`web_fetch`, jedoch nicht das vollständige `browser`-Tool. Damit der Agent oder ein
erzeugter Unteragent die Browserautomatisierung verwenden kann, fügen Sie „browser“ in der Profilphase
hinzu:

```json5
{
  tools: {
    profile: "coding",
    alsoAllow: ["browser"],
  },
}
```

Verwenden Sie für einen einzelnen Agenten `agents.entries.*.tools.alsoAllow: ["browser"]`.
`tools.subagents.tools.allow: ["browser"]` allein reicht nicht aus, da die Richtlinie für Unteragenten
nach der Profilfilterung angewendet wird.

Das Browser-Plugin stellt Hinweise für Agenten auf zwei Ebenen bereit:

- Die Beschreibung des `browser`-Tools enthält den kompakten, stets aktiven Vertrag: das
  richtige Profil auswählen, Referenzen im selben Tab beibehalten, `tabId`/Bezeichnungen zur Tab-
  Auswahl verwenden und für mehrstufige Aufgaben das Browser-Skill laden.
- Das gebündelte `browser-automation`-Skill enthält den ausführlicheren Betriebsablauf:
  zuerst Status und Tabs prüfen, Aufgaben-Tabs bezeichnen, vor Aktionen einen Snapshot erstellen, nach
  UI-Änderungen erneut einen Snapshot erstellen, veraltete Referenzen einmal wiederherstellen und
  Blocker wie Anmeldung/2FA/CAPTCHA oder Kamera/Mikrofon als erforderliche manuelle Aktion melden, statt zu raten.

Vom Plugin gebündelte Skills werden in den verfügbaren Skills des Agenten aufgeführt, wenn das
Plugin aktiviert ist. Die vollständigen Skill-Anweisungen werden bei Bedarf geladen, sodass bei
Routinevorgängen nicht die gesamten Token-Kosten anfallen.

## Fehlender Browserbefehl oder fehlendes Browser-Tool

Wenn `openclaw browser` nach einem Upgrade unbekannt ist, `browser.request` fehlt oder der Agent meldet, dass das Browser-Tool nicht verfügbar ist, liegt die Ursache üblicherweise in einer `plugins.allow`-Liste, in der `browser` fehlt, während kein `browser`-Konfigurationsblock auf Stammebene vorhanden ist. Fügen Sie ihn hinzu:

```json5
{
  plugins: {
    allow: ["telegram", "browser"],
  },
}
```

Ein expliziter `browser`-Block auf Stammebene (ein beliebiger Schlüssel unter `browser`, beispielsweise
`browser.enabled=true` oder `browser.profiles.<name>`) aktiviert das gebündelte
Browser-Plugin auch bei einer restriktiven `plugins.allow` und entspricht damit dem Verhalten der gebündelten
Kanalkonfiguration. `plugins.entries.browser.enabled=true` und
`tools.alsoAllow: ["browser"]` ersetzen für sich allein nicht die Mitgliedschaft in der Zulassungsliste.
Wenn `plugins.allow` vollständig entfernt wird, wird ebenfalls die Standardeinstellung wiederhergestellt.

## Profile: `openclaw`, `user`, `chrome`

- `openclaw`: verwalteter, isolierter Browser (keine Erweiterung erforderlich).
- `user`: integriertes Chrome-DevTools-MCP-Verbindungsprofil für Ihre **tatsächlich
  angemeldete Chrome-Sitzung**. Beim ersten Verbindungsaufbau durch OpenClaw zeigt Chrome die blockierende
  Aufforderung „Allow remote debugging?“ an, daher muss sich jemand am Computer befinden.
- `chrome`: integriertes [Chrome-Erweiterungsprofil](/de/tools/chrome-extension) für
  Ihre **tatsächlich angemeldete Chrome-Sitzung**. Funktioniert von einem Mobiltelefon aus, ohne dass jemand am
  Schreibtisch sein muss, da Tabs über die OpenClaw-Browsererweiterung statt über
  den Remote-Debugging-Port gesteuert werden; daher erscheint keine Aufforderung „Allow remote debugging?“.

Für Aufrufe des Browser-Tools durch Agenten gilt:

- Standard: den isolierten `openclaw`-Browser verwenden.
- `profile="chrome"` (Erweiterung) bevorzugen, wenn bestehende angemeldete Sitzungen benötigt werden
  und der Benutzer **nicht am Computer** ist (Telegram, WhatsApp usw.).
- `profile="user"` (Chrome MCP) bevorzugen, wenn bestehende angemeldete Sitzungen benötigt werden
  und der Benutzer **am Computer** ist, um die Verbindungsaufforderung zu bestätigen.
- `profile` dient als explizite Überschreibung, wenn Sie einen bestimmten Browsermodus verwenden möchten.

Legen Sie `browser.defaultProfile: "openclaw"` fest, wenn Sie standardmäßig den verwalteten Modus verwenden möchten.

## Konfiguration

Die Browsereinstellungen befinden sich in `~/.openclaw/openclaw.json`.

```json5
{
  browser: {
    enabled: true, // Standard: true
    evaluateEnabled: true, // Standard: true; false deaktiviert act:evaluate (beliebiges JS)
    ssrfPolicy: {
      // dangerouslyAllowPrivateNetwork: true, // nur für vertrauenswürdigen Zugriff auf private Netzwerke aktivieren
      // hostnameAllowlist: ["*.example.com", "example.com"],
      // allowedHostnames: ["localhost"],
    },
    // cdpUrl: "http://127.0.0.1:18792", // veraltete Überschreibung für ein einzelnes Profil
    tabCleanup: {
      enabled: true, // Standard: true
    },
    // snapshotDefaults: { mode: "efficient" }, // standardmäßiger Snapshot-Modus, wenn der Aufrufer keinen angibt
    defaultProfile: "openclaw",
    color: "#FF4500",
    headless: false,
    noSandbox: false,
    attachOnly: false,
    executablePath: "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser",
    profiles: {
      openclaw: { cdpPort: 18800, color: "#FF4500" },
      work: {
        cdpPort: 18801,
        color: "#0066CC",
        headless: true,
        executablePath: "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome",
      },
      user: {
        driver: "existing-session",
        attachOnly: true,
        color: "#00AA00",
      },
      brave: {
        driver: "existing-session",
        attachOnly: true,
        userDataDir: "~/Library/Application Support/BraveSoftware/Brave-Browser",
        color: "#FB542B",
      },
      remote: { cdpUrl: "http://10.0.0.42:9222", color: "#00AA00" },
    },
  },
}
```

`browser.snapshotDefaults.mode: "efficient"` ändert den standardmäßigen `snapshot`-
Extraktionsmodus, wenn ein Aufrufer weder `snapshotFormat` noch
`mode` explizit übergibt; Snapshot-Optionen für einzelne Aufrufe finden Sie unter [API zur Browsersteuerung](/de/tools/browser-control).

### Zuständigkeit für die Tab-Bereinigung

Die Bereinigung von Sitzungstabs gilt nur für Tabs, die vom OpenClaw-Browser-Tool
mit `action: "open"` erstellt wurden. OpenClaw übernimmt keine Tabs, die bereits geöffnet waren,
vom Benutzer geöffnet wurden oder deren Eigentümerschaft anderweitig unbekannt ist. Der
`browser.tabCleanup`-Block steuert regelmäßige Bereinigungen nach Inaktivität und bei Überschreitung der Obergrenze für primäre
Sitzungen; seine Deaktivierung deaktiviert nicht die explizite Bereinigung im Rahmen des Sitzungslebenszyklus.

Bei hostlokalen Öffnungsvorgängen wird die Eigentümerschaft mit einem stabilen nativen CDP-Ziel und einer Browser-
identität im gemeinsamen SQLite-Zustand gespeichert. Diese Datensätze bleiben nach einem Neustart des Gateway
erhalten und weiterhin für `/new` sowie andere Bereinigungen im Rahmen des Sitzungslebenszyklus relevant;
die Bereinigung im Rahmen des Sitzungslebenszyklus umfasst das Ende von Unteragenten-, Cron- und ACP-Sitzungen.
Datensätze, deren für das Tool sichtbares Ziel dem nativen CDP-Ziel entspricht, bleiben nach einem Neustart ebenfalls
für Bereinigungen nach Inaktivität und bei Überschreitung der sitzungsbezogenen Obergrenze relevant. Chrome-MCP-Zielhandles sind
prozesslokal, daher warten kalte Datensätze bestehender Sitzungen auf die Bereinigung im Rahmen des Sitzungslebenszyklus,
statt eine Inaktivitätsbereinigung für Aktivitäten zu riskieren, die nach einem Neustart nicht zuverlässig zugeordnet
werden können. Dieser dauerhafte Pfad kann von OpenClaw verwaltete Profile,
reguläre Remote-CDP-Profile und Profile bestehender Sitzungen mit einem expliziten
`cdpUrl` abdecken, sofern OpenClaw sowohl das native Ziel als auch eine stabile
Browseridentität auflösen kann. Vor dem Schließen eines dauerhaften Datensatzes prüft OpenClaw, ob das
konfigurierte Profil und die Browserinstanz weiterhin übereinstimmen.

Chrome-MCP-`--autoConnect`, CDP-Endpunkte, deren `/json/version`-Antwort keine
stabile Browseridentität enthält, sowie Öffnungsvorgänge, deren natives Ziel nicht aufgelöst werden kann,
verbleiben in einer prozesslokalen Best-Effort-Verfolgung. Sie können bereinigt werden, solange dieser
Gateway-Prozess ausgeführt wird, werden jedoch nach einem Neustart des
Gateway nicht automatisch geschlossen. Tabs, die vor Verfügbarkeit der dauerhaften Verfolgung geöffnet blieben, werden nicht
rückwirkend übernommen; schließen Sie diese Tabs manuell.

Die Bereinigung erfolgt nach bestem Bemühen; es besteht keine Garantie, dass jeder geeignete Tab
sofort geschlossen wird. Bei einem vorübergehenden Fehler der Eigentümerschaftsprüfung oder beim Schließen bleibt die dauerhafte
Bereinigung für einen späteren erneuten Versuch ausstehend. Die Wiederholungsversuche sind nicht unbegrenzt: Wenn der Browser
weiterhin nicht erreichbar ist und der Tab seit über einem Tag nicht verwendet wurde, wird der Verfolgungsdatensatz
entfernt, damit sich der dauerhafte Speicher nicht mit Tabs füllt, die nie wieder
überprüft werden können.

### Screenshot-Bilderkennung (Unterstützung für reine Textmodelle)

Wenn das Hauptmodell ausschließlich Text unterstützt (keine Bild- oder multimodale Unterstützung), geben Browser-
Screenshots Bildblöcke zurück, die das Modell nicht lesen kann. Browser-Screenshots
verwenden die bestehende Konfiguration zur Bildauswertung erneut, sodass ein für die Medienauswertung
konfiguriertes Bildmodell Screenshots ohne browserspezifische
Modelleinstellungen als Text beschreiben kann.

```json5
{
  tools: {
    media: {
      image: {
        models: [
          { provider: "bytedance", model: "doubao-seed-2.0-pro" },
          // Fallback-Kandidaten hinzufügen; der erste Erfolg gewinnt
          { provider: "openai", model: "gpt-4o" },
        ],
      },
      // Gemeinsam genutzte Medienmodelle funktionieren ebenfalls, wenn sie für Bildunterstützung gekennzeichnet sind.
      // models: [{ provider: "openai", model: "gpt-4o", capabilities: ["image"] }],
    },
  },
  agents: {
    defaults: {
      // Bestehende Standardeinstellungen für Bildmodelle werden ebenfalls berücksichtigt.
      // imageModel: { primary: "openai/gpt-4o" },
    },
  },
}
```

**Funktionsweise:**

1. Der Agent ruft `browser screenshot` auf, und wie üblich wird ein Bild auf dem Datenträger gespeichert.
2. Das Browser-Tool fragt die vorhandene Laufzeit für Bildverständnis, ob sie
   den Screenshot mithilfe konfigurierter Medienbildmodelle, gemeinsam genutzter
   Medienmodelle, Bildmodell-Standardeinstellungen oder eines authentifizierungsgestützten Bild-Providers beschreiben kann.
3. Das Vision-Modell gibt eine Textbeschreibung zurück, die mit
   `wrapExternalContent` (Schutz vor Prompt-Injection) umschlossen und an den Agenten
   als Textblock statt als Bildblock zurückgegeben wird.
4. Wenn das Bildverständnis nicht verfügbar ist, übersprungen wird oder fehlschlägt, gibt der Browser
   ersatzweise den ursprünglichen Bildblock zurück.

Screenshot-Bildblöcke sind private Tool-Ergebnisse: Der Agent kann sie untersuchen,
aber OpenClaw hängt sie nicht automatisch an Kanalantworten an. Um einen
Screenshot zu teilen, weisen Sie den Agenten an, ihn ausdrücklich mit dem Nachrichten-Tool zu senden.

Verwenden Sie die vorhandenen Felder `tools.media.image` / `tools.media.models` für Modell-
Fallbacks, Zeitüberschreitungen, Bytegrenzen, Profile und Provider-Anfrageeinstellungen.

Wenn das aktive Hauptmodell bereits Vision unterstützt und kein ausdrückliches Modell
für Bildverständnis konfiguriert ist, behält OpenClaw das normale Bildergebnis bei, damit das
Hauptmodell den Screenshot direkt lesen kann.

<AccordionGroup>

<Accordion title="Ports und Erreichbarkeit">

- Der Steuerungsdienst bindet sich an die Loopback-Schnittstelle auf einem von `gateway.port` abgeleiteten Port (Standardwert `18791` = Gateway + 2). `OPENCLAW_GATEWAY_PORT` hat Vorrang vor `gateway.port`; beide verschieben die abgeleiteten Ports derselben Familie entsprechend.
- Lokale `openclaw`-Profile weisen `cdpPort`/`cdpUrl` automatisch aus einem Bereich zu, der 9 Ports über dem Steuerungsport beginnt (standardmäßig `18800`-`18899`); legen Sie diese nur für
  Remote-CDP-Profile oder zum Anhängen an den Endpunkt einer vorhandenen Sitzung fest. `cdpUrl` verwendet standardmäßig
  den verwalteten lokalen CDP-Port, wenn kein Wert festgelegt ist.
- Die Erreichbarkeit von Remote- und `attachOnly`-CDP, WebSocket-Handshakes und der lokale
  Start des verwalteten Chrome verwenden integrierte Fristen.
- Wiederholte Fehler beim Starten oder bei der Bereitschaft des verwalteten Chrome werden pro
  Profil durch einen Circuit Breaker begrenzt. Nach mehreren aufeinanderfolgenden Fehlern pausiert OpenClaw neue
  Startversuche kurzzeitig, statt bei jedem Aufruf des Browser-Tools Chromium zu starten. Beheben Sie
  das Startproblem, deaktivieren Sie den Browser, wenn er nicht benötigt wird, oder starten Sie nach der
  Reparatur das Gateway neu.

</Accordion>

<Accordion title="SSRF-Richtlinie">

- Browsernavigation und Anfragen zum Öffnen von Tabs werden vorab geprüft. Während der Aktion und einer begrenzten Karenzzeit danach fangen geschützte Playwright-Interaktionen (Klick, Koordinatenklick, Zeigen, Ziehen, Scrollen, Auswählen, Tastendruck, Eingabe, Ausfüllen von Formularen und Auswerten) durch die Richtlinie abgelehnte Dokumentladevorgänge auf oberster Ebene und in Unterframes ab, bevor HTTP-Anfragebytes gesendet werden, und prüfen anschließend nach bestem Bemühen erneut die endgültige `http(s)`-URL.
- Vor jedem neuen von OpenClaw verwalteten Chrome-Start deaktiviert OpenClaw nach bestem Bemühen die Netzwerkvorhersage und unterdrückt damit die beobachteten spekulativen Vorverbindungen von Chromium für diese abgelehnten Ladevorgänge. Dies ist mehrschichtige Absicherung und keine Richtliniengrenze: Ein Browser, der über einen Neustart des Steuerungsdienstes hinweg wiederverwendet wird, sowie andere Browser-Backends verfügen möglicherweise nicht über dieselbe Härtung. Playwright-Routing ist weiterhin keine Netzwerk-Firewall und fängt weder Weiterleitungsstationen noch die erste Anfrage eines Pop-ups, Service-Worker-Datenverkehr, nach Ablauf des begrenzten Schutzfensters ausgeführten Seitencode oder jeden Hintergrund-/Unterressourcenpfad ab. Eine vollständige Egress-Isolation erfordert eine Isolation auf Eigentümerseite oder einen richtliniendurchsetzenden Proxy.
- Im strikten SSRF-Modus werden auch die Ermittlung von Remote-CDP-Endpunkten und `/json/version`-Prüfungen (`cdpUrl`) überprüft.
- Die Gateway-/Provider-Umgebungsvariablen `HTTP_PROXY`, `HTTPS_PROXY`, `ALL_PROXY` und `NO_PROXY` leiten den von OpenClaw verwalteten Browser nicht automatisch über einen Proxy. Verwaltetes Chrome startet standardmäßig direkt, damit Provider-Proxy-Einstellungen die SSRF-Prüfungen des Browsers nicht schwächen.
- Von OpenClaw verwaltete lokale CDP-Bereitschaftsprüfungen und DevTools-WebSocket-Verbindungen umgehen den verwalteten Netzwerk-Proxy für den exakt gestarteten Loopback-Endpunkt, sodass `openclaw browser start` weiterhin funktioniert, wenn ein Betreiber-Proxy Loopback-Egress blockiert.
- Um den verwalteten Browser selbst über einen Proxy zu leiten, übergeben Sie ausdrückliche Chrome-Proxy-Flags über `browser.extraArgs`, beispielsweise `--proxy-server=...` oder `--proxy-pac-url=...`. Der strikte SSRF-Modus blockiert ausdrückliches Browser-Proxy-Routing, sofern der Browserzugriff auf private Netzwerke nicht absichtlich aktiviert ist.
- `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork` ist standardmäßig deaktiviert; aktivieren Sie es nur, wenn dem Browserzugriff auf private Netzwerke ausdrücklich vertraut wird.
- `browser.ssrfPolicy.allowPrivateNetwork` wird weiterhin als veralteter Alias unterstützt.

</Accordion>

<Accordion title="Profilverhalten">

- `attachOnly: true` bedeutet, niemals einen lokalen Browser zu starten; es wird nur eine Verbindung hergestellt, wenn bereits einer läuft.
- `headless` kann global oder für jedes lokal verwaltete Profil festgelegt werden. Profilspezifische Werte überschreiben `browser.headless`, sodass ein lokal gestartetes Profil headless bleiben kann, während ein anderes sichtbar bleibt.
- `POST /start?headless=true` und `openclaw browser start --headless` fordern einen
  einmaligen Headless-Start für lokal verwaltete Profile an, ohne
  `browser.headless` oder die Profilkonfiguration neu zu schreiben. Profile für vorhandene Sitzungen, reine Anhängeprofile und
  Remote-CDP-Profile lehnen die Überschreibung ab, da OpenClaw diese
  Browserprozesse nicht startet.
- Auf Linux-Hosts ohne `DISPLAY` oder `WAYLAND_DISPLAY` verwenden lokal verwaltete Profile
  automatisch standardmäßig den Headless-Modus, wenn weder die Umgebung noch die Profil-/globale
  Konfiguration ausdrücklich den Modus mit Benutzeroberfläche auswählt. Verwenden Sie die eindeutige Form auf Browserebene
  `openclaw browser --json status`; ein nachgestelltes `openclaw browser status --json`
  funktioniert ebenfalls, da `status` kein eigenes `--json` definiert. Der Befehl meldet
  `headlessSource` als `env`, `profile`, `config`,
  `request`, `linux-display-fallback` oder `default`.
- `OPENCLAW_BROWSER_HEADLESS=1` erzwingt für den aktuellen Prozess Headless-Starts lokal
  verwalteter Browser. `OPENCLAW_BROWSER_HEADLESS=0` erzwingt für gewöhnliche
  Starts den Modus mit Benutzeroberfläche und gibt auf Linux-Hosts ohne Anzeigeserver einen Fehler mit konkreter Handlungsanweisung zurück;
  eine ausdrückliche `start --headless`-Anforderung hat für diesen einzelnen Start weiterhin Vorrang.
- Die Browser-Steuerungsroute und der programmatische Client behalten die menschenlesbare
  `error` des Fehlers wegen fehlender Anzeige bei und stellen den stabilen Grund
  `no_display_for_headed_profile` bereit. Seine `details` enthalten nur `profile`,
  `requestedHeadless`, `headlessSource` und `displayPresent`, sodass API-Clients
  die richtige Abhilfemaßnahme auswählen können, ohne den Nachrichtentext abzugleichen.
- Für ein laufendes lokal verwaltetes Profil fragen Status und Doctor den
  CDP-Endpunkt von Chrome auf Browserebene nach Renderer, Backend, Gerät/Treiber, Funktionsstatus,
  Treiber-Workarounds und Fähigkeiten zur beschleunigten Videowiedergabe ab. Das Ergebnis wird
  für diesen Browserprozess zwischengespeichert und vollständig über
  `openclaw browser --json status` bereitgestellt. Ein passiver Statusaufruf startet Chrome nicht.
  Browser für vorhandene Sitzungen, Erweiterungen, Remote-CDP und Sandboxes bleiben getrennt
  und werden nicht über diesen Pfad des verwalteten Hosts untersucht.
- Headless verwaltetes Chrome verwendet weiterhin den konservativen Standardwert `--disable-gpu`.
  Die Diagnose aktiviert keine Beschleunigung, fügt keine globale Beschleunigungseinstellung hinzu
  und gewährt Sandbox-Browsern keinen Gerätezugriff.
- `executablePath` kann global oder für jedes lokal verwaltete Profil festgelegt werden. Profilspezifische Werte überschreiben `browser.executablePath`, sodass verschiedene verwaltete Profile unterschiedliche Chromium-basierte Browser starten können. Beide Formen akzeptieren `~` für das Home-Verzeichnis Ihres Betriebssystems.
- `color` (auf oberster Ebene und pro Profil) färbt die Browseroberfläche ein, damit Sie erkennen können, welches Profil aktiv ist.
- Das Standardprofil ist `openclaw` (eigenständig verwaltet). Verwenden Sie `defaultProfile: "user"`, um den angemeldeten Benutzerbrowser ausdrücklich zu aktivieren.
- Reihenfolge der automatischen Erkennung: der Systemstandardbrowser, sofern er Chromium-basiert ist; andernfalls Chrome, Brave, Edge, Chromium, Chrome Canary.
- `driver: "existing-session"` verwendet Chrome DevTools MCP anstelle von unverarbeitetem CDP. Die Verbindung kann über die automatische Verbindungsherstellung von Chrome MCP oder über `cdpUrl` hergestellt werden, wenn Sie bereits über einen DevTools-Endpunkt für den laufenden Browser verfügen.
- `driver: "extension"` steuert Ihr angemeldetes Chrome über die [OpenClaw-Chrome-Erweiterung](/de/tools/chrome-extension). Das Relay verwaltet seinen Loopback-Endpunkt, daher akzeptieren diese Profile `cdpUrl` nicht. Dies ist der einzige Modus für einen angemeldeten Browser, der funktioniert, wenn niemand am Computer sitzt.
- Legen Sie `browser.profiles.<name>.userDataDir` fest, wenn ein Profil für eine vorhandene Sitzung an ein nicht standardmäßiges Chromium-Benutzerprofil (Brave, Edge usw.) angehängt werden soll. Dieser Pfad akzeptiert außerdem `~` für das Home-Verzeichnis Ihres Betriebssystems.

</Accordion>

</AccordionGroup>

## Brave oder einen anderen Chromium-basierten Browser verwenden

Wenn Ihr **Systemstandardbrowser** Chromium-basiert ist (Chrome/Brave/Edge usw.),
verwendet OpenClaw ihn automatisch. Legen Sie `browser.executablePath` fest, um die
automatische Erkennung zu überschreiben. Werte von `executablePath` auf oberster Ebene und pro Profil akzeptieren `~`
für das Home-Verzeichnis Ihres Betriebssystems:

```bash
openclaw config set browser.executablePath "/usr/bin/google-chrome"
openclaw config set browser.profiles.work.executablePath "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
```

Oder legen Sie ihn je Plattform in der Konfiguration fest:

<Tabs>
  <Tab title="macOS">
```json5
{
  browser: {
    executablePath: "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser",
  },
}
```
  </Tab>
  <Tab title="Windows">
```json5
{
  browser: {
    executablePath: "C:\\Program Files\\BraveSoftware\\Brave-Browser\\Application\\brave.exe",
  },
}
```
  </Tab>
  <Tab title="Linux">
```json5
{
  browser: {
    executablePath: "/usr/bin/brave-browser",
  },
}
```
  </Tab>
</Tabs>

Der profilspezifische Wert `executablePath` wirkt sich nur auf lokal verwaltete Profile aus, die OpenClaw
startet. `existing-session`-Profile stellen stattdessen eine Verbindung zu einem bereits laufenden Browser her,
und Remote-CDP-Profile verwenden den Browser hinter `cdpUrl`.

## Lokale und Remote-Steuerung

- **Lokale Steuerung (Standard):** Das Gateway startet den Loopback-Steuerungsdienst und kann einen lokalen Browser starten.
- **Remote-Steuerung (Node-Host):** Führen Sie einen Node-Host auf dem Computer aus, auf dem sich der Browser befindet; das Gateway leitet Browseraktionen an ihn weiter.
- **Remote-CDP:** Legen Sie `browser.profiles.<name>.cdpUrl` (oder `browser.cdpUrl`) fest, um
  eine Verbindung zu einem entfernten Chromium-basierten Browser herzustellen. In diesem Fall startet OpenClaw keinen lokalen Browser.
- Legen Sie für extern verwaltete CDP-Dienste auf der Loopback-Schnittstelle (beispielsweise Browserless in
  Docker, veröffentlicht unter `127.0.0.1`) zusätzlich `attachOnly: true` fest. Loopback-CDP
  ohne `attachOnly` wird als lokales, von OpenClaw verwaltetes Browserprofil behandelt.
- `headless` wirkt sich nur auf lokal verwaltete Profile aus, die OpenClaw startet. Vorhandene Sitzungs- oder Remote-CDP-Browser werden dadurch weder neu gestartet noch geändert.
- `executablePath` folgt derselben Regel für lokal verwaltete Profile. Wird der Wert bei einem
  laufenden lokal verwalteten Profil geändert, wird dieses Profil für einen Neustart/Abgleich vorgemerkt, sodass
  beim nächsten Start die neue Binärdatei verwendet wird.

Das Verhalten beim Beenden unterscheidet sich je nach Profilmodus:

- lokal verwaltete Profile: `openclaw browser stop` beendet den Browserprozess, den
  OpenClaw gestartet hat
- reine Anhängeprofile und Remote-CDP-Profile: `openclaw browser stop` schließt die aktive
  Steuerungssitzung und hebt Playwright-/CDP-Emulationsüberschreibungen auf (Viewport,
  Farbschema, Gebietsschema, Zeitzone, Offline-Modus und ähnliche Zustände), obwohl
  OpenClaw keinen Browserprozess gestartet hat

Remote-CDP-URLs können Authentifizierungsdaten enthalten:

- Abfragetoken (z. B. `https://provider.example?token=<token>`)
- HTTP-Basisauthentifizierung (z. B. `https://user:pass@provider.example`)

OpenClaw behält die Authentifizierung beim Aufrufen von `/json/*`-Endpunkten und beim Herstellen einer Verbindung
zum CDP-WebSocket bei. Verwenden Sie für Tokens vorzugsweise Umgebungsvariablen oder Secret-Manager,
statt sie in Konfigurationsdateien zu committen.

## Node-Browser-Proxy (konfigurationsfreier Standard)

Wenn Sie einen **Node-Host** auf dem Computer ausführen, auf dem sich Ihr Browser befindet, kann OpenClaw
Aufrufe von Browser-Tools ohne zusätzliche Browserkonfiguration automatisch an diesen Node weiterleiten.
Dies ist der Standardpfad für entfernte Gateways.

Hinweise:

- Der Node-Host stellt seinen lokalen Browser-Steuerungsserver über einen **Proxy-Befehl** bereit.
- Profile stammen aus der eigenen `browser.profiles`-Konfiguration des Nodes (wie bei lokaler Ausführung).
- Der Proxy-Befehl erlaubt unabhängig von `allowProfiles` niemals dauerhafte Profiländerungen (`create-profile`, `delete-profile`, `reset-profile`); nehmen Sie diese Änderungen direkt auf dem Node vor.
- `nodeHost.browserProxy.allowProfiles` ist optional. Lassen Sie es für das bisherige/standardmäßige Verhalten leer: Alle konfigurierten Profile bleiben über den Proxy erreichbar.
- Wenn Sie `nodeHost.browserProxy.allowProfiles` festlegen, behandelt OpenClaw dies als Least-Privilege-Grenze, die einschränkt, auf welche Profilnamen der Proxy zugreifen kann.
- Deaktivieren Sie dies, wenn Sie es nicht verwenden möchten:
  - Auf dem Node: `nodeHost.browserProxy.enabled=false`
  - Auf dem Gateway: `gateway.nodes.browser.mode="off"` (akzeptiert auch `"auto"`, um einen einzelnen verbundenen Browser-Node auszuwählen, oder `"manual"`, um einen expliziten Node-Parameter zu verlangen)

## Browserless (gehostetes entferntes CDP)

[Browserless](https://browserless.io) ist ein gehosteter Chromium-Dienst, der
CDP-Verbindungs-URLs über HTTPS und WebSocket bereitstellt. OpenClaw kann beide Formen verwenden, aber
für ein entferntes Browserprofil ist die direkte WebSocket-URL
aus der Verbindungsdokumentation von Browserless die einfachste Option.

Beispiel:

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "browserless",
    profiles: {
      browserless: {
        cdpUrl: "wss://production-sfo.browserless.io?token=<BROWSERLESS_API_KEY>",
        color: "#00AA00",
      },
    },
  },
}
```

Hinweise:

- Ersetzen Sie `<BROWSERLESS_API_KEY>` durch Ihr tatsächliches Browserless-Token.
- Wählen Sie den regionalen Endpunkt, der Ihrem Browserless-Konto entspricht (siehe deren Dokumentation).
- Wenn Browserless Ihnen eine HTTPS-Basis-URL bereitstellt, können Sie diese entweder für eine direkte CDP-Verbindung in
  `wss://` umwandeln oder die HTTPS-URL beibehalten und OpenClaw
  `/json/version` ermitteln lassen.

### Browserless Docker auf demselben Host

Wenn Browserless selbst in Docker gehostet wird und OpenClaw auf dem Host ausgeführt wird, behandeln Sie
Browserless als extern verwalteten CDP-Dienst:

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "browserless",
    profiles: {
      browserless: {
        cdpUrl: "ws://127.0.0.1:3000",
        attachOnly: true,
        color: "#00AA00",
      },
    },
  },
}
```

Die Adresse in `browser.profiles.browserless.cdpUrl` muss für den
OpenClaw-Prozess erreichbar sein. Browserless muss außerdem einen passenden erreichbaren Endpunkt bekannt geben;
setzen Sie Browserless `EXTERNAL` auf dieselbe von außen für OpenClaw erreichbare WebSocket-Basis,
beispielsweise `ws://127.0.0.1:3000`, `ws://browserless:3000` oder eine stabile private
Docker-Netzwerkadresse. Wenn `/json/version` den Wert `webSocketDebuggerUrl` zurückgibt, der auf
eine für OpenClaw nicht erreichbare Adresse verweist, kann CDP-HTTP funktionsfähig erscheinen, während das
Anhängen über WebSocket dennoch fehlschlägt.

Lassen Sie `attachOnly` für ein Browserless-Profil mit Loopback-Adresse nicht ungesetzt. Ohne
`attachOnly` behandelt OpenClaw den Loopback-Port als lokal verwaltetes Browserprofil
und meldet möglicherweise, dass der Port verwendet wird, aber nicht OpenClaw gehört.

## Direkte WebSocket-CDP-Provider

Einige gehostete Browserdienste stellen statt der standardmäßigen HTTP-basierten
CDP-Ermittlung (`/json/version`) einen **direkten WebSocket**-Endpunkt bereit. OpenClaw akzeptiert drei
Formen von CDP-URLs und wählt automatisch die passende Verbindungsstrategie:

- **HTTP(S)-Ermittlung** – `http://host[:port]` oder `https://host[:port]`.
  OpenClaw ruft `/json/version` auf, um die WebSocket-Debugger-URL zu ermitteln, und stellt anschließend
  die Verbindung her. Kein WebSocket-Fallback.
- **Direkte WebSocket-Endpunkte** – `ws://host[:port]/devtools/<kind>/<id>` oder
  `wss://...` mit einem `/devtools/browser|page|worker|shared_worker|service_worker/<id>`-Pfad.
  OpenClaw stellt die Verbindung direkt über einen WebSocket-Handshake her und überspringt
  `/json/version` vollständig.
- **Reine WebSocket-Stammadressen** – `ws://host[:port]` oder `wss://host[:port]` ohne
  `/devtools/...`-Pfad (z. B. [Browserless](https://browserless.io),
  [Browserbase](https://www.browserbase.com)). OpenClaw versucht zunächst die HTTP-
  `/json/version`-Ermittlung (wobei das Schema zu `http`/`https` normalisiert wird);
  wenn die Ermittlung einen `webSocketDebuggerUrl` zurückgibt, wird dieser verwendet, andernfalls
  greift OpenClaw auf einen direkten WebSocket-Handshake an der reinen Stammadresse zurück. Wenn der bekannt gegebene
  WebSocket-Endpunkt den CDP-Handshake ablehnt, die konfigurierte reine Stammadresse
  ihn jedoch akzeptiert, greift OpenClaw ebenfalls auf diese Stammadresse zurück. Dadurch kann eine reine `ws://`,
  die auf eine lokale Chrome-Instanz verweist, weiterhin eine Verbindung herstellen, da Chrome WebSocket-
  Upgrades nur für den spezifischen zielbezogenen Pfad aus `/json/version` akzeptiert, während gehostete
  Provider weiterhin ihren WebSocket-Stammendpunkt verwenden können, wenn ihr Ermittlungsendpunkt
  eine kurzlebige URL bekannt gibt, die für Playwright CDP ungeeignet ist.

`openclaw browser doctor` verwendet dieselbe Logik mit vorrangiger Ermittlung und WebSocket-Fallback
wie das Anhängen zur Laufzeit, sodass eine reine Stamm-URL, die erfolgreich eine Verbindung herstellt, von der
Diagnose nicht als unerreichbar gemeldet wird.

### Browserbase

[Browserbase](https://www.browserbase.com) ist eine Cloud-Plattform zum Ausführen
headless betriebener Browser mit integrierter CAPTCHA-Lösung, Stealth-Modus und
Residential-Proxys.

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "browserbase",
    profiles: {
      browserbase: {
        cdpUrl: "wss://connect.browserbase.com?apiKey=<BROWSERBASE_API_KEY>",
        color: "#F97316",
      },
    },
  },
}
```

Hinweise:

- [Registrieren Sie sich](https://www.browserbase.com/sign-up) und kopieren Sie Ihren **API Key**
  aus dem [Overview dashboard](https://www.browserbase.com/overview).
- Ersetzen Sie `<BROWSERBASE_API_KEY>` durch Ihren tatsächlichen Browserbase-API-Schlüssel.
- Browserbase erstellt beim Herstellen der WebSocket-Verbindung automatisch eine Browsersitzung, sodass kein
  manueller Schritt zur Sitzungserstellung erforderlich ist.
- Aktuelle Limits des kostenlosen Tarifs und kostenpflichtige Tarife finden Sie unter [Preise](https://www.browserbase.com/pricing).
- Die vollständige API-Referenz, SDK-Anleitungen und Integrationsbeispiele finden Sie in der [Browserbase-Dokumentation](https://docs.browserbase.com).

### Notte

[Notte](https://www.notte.cc) ist eine Cloud-Plattform zum Ausführen headless
betriebener Browser mit integriertem Stealth-Modus, Residential-Proxys und einem CDP-nativen
WebSocket-Gateway.

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "notte",
    profiles: {
      notte: {
        cdpUrl: "wss://us-prod.notte.cc/sessions/connect?token=<NOTTE_API_KEY>",
        color: "#7C3AED",
      },
    },
  },
}
```

Hinweise:

- [Registrieren Sie sich](https://console.notte.cc) und kopieren Sie Ihren **API Key** von der
  Konsoleneinstellungsseite.
- Ersetzen Sie `<NOTTE_API_KEY>` durch Ihren tatsächlichen Notte-API-Schlüssel.
- Notte erstellt beim Herstellen der WebSocket-Verbindung automatisch eine Browsersitzung, sodass kein manueller
  Schritt zur Sitzungserstellung erforderlich ist. Die Sitzung wird beendet, wenn die
  WebSocket-Verbindung getrennt wird.
- Aktuelle Limits des kostenlosen Tarifs und kostenpflichtige Tarife finden Sie unter [Preise](https://www.notte.cc/#pricing).
- Die vollständige API-Referenz, SDK-Anleitungen und Integrationsbeispiele finden Sie in der [Notte-Dokumentation](https://docs.notte.cc).

## Sicherheit

Grundgedanken:

- Die Browsersteuerung ist ausschließlich über Loopback erreichbar; der Zugriff erfolgt über die Authentifizierung des Gateways oder die Node-Kopplung.
- Die eigenständige Loopback-Browser-HTTP-API verwendet **ausschließlich Shared-Secret-Authentifizierung**:
  Gateway-Token-Bearer-Authentifizierung, `x-openclaw-password` oder HTTP-Basic-Authentifizierung mit dem
  konfigurierten Gateway-Passwort.
- Tailscale-Serve-Identitätsheader und `gateway.auth.mode: "trusted-proxy"`
  authentifizieren diese eigenständige Loopback-Browser-API **nicht**.
- Wenn die Browsersteuerung aktiviert und keine Shared-Secret-Authentifizierung konfiguriert ist, generiert und speichert OpenClaw
  beim Start automatisch einen Berechtigungsnachweis für die Browsersteuerung:
  ein Token, wenn `gateway.auth.mode` den Wert `none` hat, oder ein Passwort, wenn der Wert
  `trusted-proxy` lautet (über `gateway.auth.password` gespeichert, damit prozessexterne
  Loopback-Clients ihn auflösen können). Die automatische Generierung wird übersprungen, wenn für diesen Modus bereits
  ein expliziter Berechtigungsnachweis als Zeichenfolge konfiguriert ist oder wenn
  `gateway.auth.mode` den Wert `password` hat.
- Konfigurieren Sie `gateway.auth.token`, `gateway.auth.password`, `OPENCLAW_GATEWAY_TOKEN` oder
  `OPENCLAW_GATEWAY_PASSWORD` explizit, wenn Sie anstelle des generierten Secrets
  ein stabiles, von Ihnen kontrolliertes Secret verwenden möchten.

Tipps für entferntes CDP:

- Bevorzugen Sie nach Möglichkeit verschlüsselte Endpunkte (HTTPS oder WSS) und kurzlebige Tokens.
- Vermeiden Sie es, langlebige Tokens direkt in Konfigurationsdateien einzubetten.
- Betreiben Sie das Gateway und alle Node-Hosts in einem privaten Netzwerk (Tailscale); vermeiden Sie eine öffentliche Erreichbarkeit.
- Behandeln Sie entfernte CDP-URLs und -Tokens als Secrets; verwenden Sie vorzugsweise Umgebungsvariablen oder einen Secret-Manager.

## Profile (mehrere Browser)

OpenClaw unterstützt mehrere benannte Profile (Routing-Konfigurationen). Profile können folgende Typen haben:

- **von OpenClaw verwaltet**: eine dedizierte Chromium-basierte Browserinstanz mit eigenem Benutzerdatenverzeichnis und CDP-Port
- **entfernt**: eine explizite CDP-URL (ein andernorts ausgeführter Chromium-basierter Browser)
- **bestehende Sitzung**: Ihr bestehendes Chrome-Profil über die automatische Verbindung von Chrome DevTools MCP

Standardeinstellungen:

- Das Profil `openclaw` wird automatisch erstellt, wenn es fehlt.
- Das Profil `user` ist für das Anhängen an eine bestehende Chrome-MCP-Sitzung integriert.
- Profile für bestehende Sitzungen müssen mit Ausnahme von `user` explizit aktiviert werden; erstellen Sie sie mit `--driver existing-session`.
- Lokale CDP-Ports werden standardmäßig aus dem Bereich **18800-18899** zugewiesen.
- Beim Löschen eines Profils wird dessen lokales Datenverzeichnis in den Papierkorb verschoben.

Alle Steuerungsendpunkte akzeptieren `?profile=<name>`; die CLI verwendet `--browser-profile`.

## Bestehende Sitzung über Chrome DevTools MCP

OpenClaw kann über den offiziellen Chrome-DevTools-MCP-Server auch eine Verbindung zu einem laufenden
Chromium-basierten Browserprofil herstellen. Dabei werden die bereits in diesem Browserprofil geöffneten
Tabs und der Anmeldestatus wiederverwendet.

Offizielle Hintergrund- und Einrichtungsreferenzen:

- [Chrome für Entwickler: Chrome DevTools MCP mit Ihrer Browsersitzung verwenden](https://developer.chrome.com/blog/chrome-devtools-mcp-debug-your-browser-session)
- [README von Chrome DevTools MCP](https://github.com/ChromeDevTools/chrome-devtools-mcp)

Integriertes Profil: `user`. Erstellen Sie ein eigenes benutzerdefiniertes Profil für bestehende Sitzungen, wenn
Sie einen anderen Namen, eine andere Farbe oder ein anderes Browserdatenverzeichnis verwenden möchten.

Standardmäßig verwendet das integrierte Profil `user` die automatische Verbindung von Chrome MCP,
die auf das lokale Standardprofil von Google Chrome abzielt. Verwenden Sie `userDataDir` für Brave,
Edge, Chromium oder ein vom Standard abweichendes Chrome-Profil. `~` wird zum Home-Verzeichnis Ihres
Betriebssystems erweitert:

```json5
{
  browser: {
    profiles: {
      brave: {
        driver: "existing-session",
        attachOnly: true,
        userDataDir: "~/Library/Application Support/BraveSoftware/Brave-Browser",
        color: "#FB542B",
      },
    },
  },
}
```

Anschließend im entsprechenden Browser:

1. Öffnen Sie die Inspektionsseite dieses Browsers für das Remote-Debugging.
2. Aktivieren Sie das Remote-Debugging.
3. Lassen Sie den Browser geöffnet und bestätigen Sie die Verbindungsaufforderung, wenn OpenClaw die Verbindung herstellt.

Gängige Inspektionsseiten:

- Chrome: `chrome://inspect/#remote-debugging`
- Brave: `brave://inspect/#remote-debugging`
- Edge: `edge://inspect/#remote-debugging`

Live-Smoke-Test für das Anhängen:

```bash
openclaw browser --browser-profile user start
openclaw browser --browser-profile user status
openclaw browser --browser-profile user tabs
openclaw browser --browser-profile user snapshot --format ai
```

So sieht eine erfolgreiche Ausführung aus:

- `status` zeigt `driver: existing-session`
- `status` zeigt `transport: chrome-mcp`
- `status` zeigt `running: true`
- `tabs` listet Ihre bereits geöffneten Browser-Tabs auf
- `snapshot` gibt Referenzen aus dem ausgewählten aktiven Tab zurück

Was zu prüfen ist, wenn das Anhängen nicht funktioniert:

- Der Chromium-basierte Zielbrowser hat die Version `144+`
- Remote-Debugging ist auf der Inspektionsseite dieses Browsers aktiviert
- Der Browser hat die Zustimmungsaufforderung zum Anhängen angezeigt und Sie haben sie akzeptiert
- Wenn Chrome mit einem expliziten `--remote-debugging-port` gestartet wurde, setzen Sie
  `browser.profiles.<name>.cdpUrl` auf diesen DevTools-Endpunkt, statt sich
  auf die automatische Verbindung von Chrome MCP zu verlassen
- `openclaw doctor` migriert alte erweiterungsbasierte Browserkonfigurationen und prüft, ob
  Chrome für standardmäßige Profile mit automatischer Verbindung lokal installiert ist, kann jedoch
  das browserseitige Remote-Debugging nicht für Sie aktivieren

Verwendung durch den Agenten:

- Verwenden Sie `profile="user"`, wenn Sie den angemeldeten Browserzustand des Benutzers benötigen.
- Wenn Sie ein benutzerdefiniertes Profil für eine bestehende Sitzung verwenden, übergeben Sie dessen expliziten Profilnamen.
- Wählen Sie diesen Modus nur, wenn der Benutzer am Computer sitzt, um die Aufforderung
  zum Anhängen zu genehmigen.
- Der Gateway- oder Node-Host kann `npx chrome-devtools-mcp@latest --autoConnect` starten.

Hinweise:

- Dieser Pfad birgt ein höheres Risiko als das isolierte Profil `openclaw`, da er
  innerhalb Ihrer angemeldeten Browsersitzung agieren kann.
- OpenClaw startet den Browser für diesen Treiber nicht, sondern hängt sich nur an.
- OpenClaw verwendet hier den offiziellen `--autoConnect`-Ablauf von Chrome DevTools MCP. Wenn
  `userDataDir` gesetzt ist, wird es weitergereicht, um dieses Benutzerdatenverzeichnis als Ziel zu verwenden.
- Eine bestehende Sitzung kann auf dem ausgewählten Host oder über eine verbundene
  Browser-Node angehängt werden. Wenn Chrome an einem anderen Ort ausgeführt wird und keine Browser-Node verbunden ist, verwenden Sie
  stattdessen Remote-CDP oder einen Node-Host.
- Chrome-MCP-Ziele und Snapshot-Referenzen sind auf einen MCP-Unterprozess beschränkt. Nachdem
  dieser Prozess neu gestartet wurde, führen Sie `browser tabs` erneut aus, wählen Sie vor zielspezifischen
  Arbeiten ausdrücklich ein neues Ziel aus und erstellen Sie einen neuen Snapshot, bevor Sie Referenzen verwenden.
  Jede Referenz ist nur für ihr Ziel und dessen neuesten Snapshot gültig. Alte Aliasse werden
  nicht auf einen Ersatztab übertragen, selbst wenn dessen URL übereinstimmt.
- Chrome DevTools MCP leitet Seitenwerkzeuge derzeit anhand einer prozesslokalen numerischen Seiten-ID
  weiter. Prozessgebundene Handles verhindern die Wiederverwendung nach dem Ersetzen eines Unterprozesses, doch ein
  Ersetzen des Browserkontexts innerhalb des Prozesses zwischen aufeinanderfolgenden Werkzeugaufrufen kann eine Aktion weiterhin
  auf ein anderes Ziel umleiten. Eine vollständig atomare Weiterleitung erfordert Upstream-Unterstützung der Seitenwerkzeuge
  für stabile Ziel-IDs.

### Benutzerdefinierter Start von Chrome MCP

Überschreiben Sie den gestarteten Chrome-DevTools-MCP-Server pro Profil, wenn der standardmäßige
`npx chrome-devtools-mcp@latest`-Ablauf nicht Ihren Anforderungen entspricht (Offline-Hosts,
festgelegte Versionen, mitgelieferte Binärdateien):

| Feld        | Funktion                                                                                                               |
| ------------ | -------------------------------------------------------------------------------------------------------------------------- |
| `mcpCommand` | Ausführbare Datei, die anstelle von `npx` gestartet wird. Wird unverändert aufgelöst; absolute Pfade werden berücksichtigt.                                          |
| `mcpArgs`    | Argument-Array, das unverändert an `mcpCommand` übergeben wird. Ersetzt die standardmäßigen `chrome-devtools-mcp@latest --autoConnect`-Argumente. |

Wenn `cdpUrl` für ein Profil einer bestehenden Sitzung gesetzt ist, überspringt OpenClaw
`--autoConnect` und leitet den Endpunkt automatisch an Chrome MCP weiter:

- `http(s)://...` → `--browserUrl <url>` (HTTP-Discovery-Endpunkt von DevTools).
- `ws(s)://...` → `--wsEndpoint <url>` (direkter CDP-WebSocket).

Endpunkt-Flags und `userDataDir` können nicht kombiniert werden: Wenn `cdpUrl` gesetzt ist,
wird `userDataDir` beim Start von Chrome MCP ignoriert, da Chrome MCP sich an den
hinter dem Endpunkt laufenden Browser anhängt, statt ein Profilverzeichnis
zu öffnen.

<Accordion title="Funktionseinschränkungen bestehender Sitzungen">

Im Vergleich zum verwalteten Profil `openclaw` unterliegen Treiber für bestehende Sitzungen stärkeren Einschränkungen:

- **Screenshots** – Seitenerfassungen und `--ref`-Elementerfassungen funktionieren; CSS-Selektoren vom Typ `--element` nicht. Playwright ist für Seiten- oder referenzbasierte Element-Screenshots nicht erforderlich. (`--full-page` kann in keinem Profil mit `--ref` oder `--element` kombiniert werden, nicht nur bei bestehenden Sitzungen.)
- **Aktionen** – `click`, `type`, `hover`, `scrollIntoView`, `drag` und `select` erfordern Snapshot-Referenzen (keine CSS-Selektoren). `click-coords` klickt auf sichtbare Viewport-Koordinaten und erfordert keine Snapshot-Referenz. `click` unterstützt nur die linke Maustaste (keine Überschreibungen der Taste oder Modifikatortasten). `type` unterstützt `slowly=true` nicht; verwenden Sie `fill` oder `press`. `press` unterstützt `delayMs` nicht. `type`, `hover`, `scrollIntoView`, `drag`, `select` und `fill` unterstützen keine aufrufspezifischen `timeoutMs`-Überschreibungen; `evaluate` unterstützt sie. `select` akzeptiert einen einzelnen Wert. `batch` wird nicht unterstützt; senden Sie Aktionen einzeln.
- **Warten / Hochladen / Dialog** – `wait --url` unterstützt exakte Muster, Teilzeichenfolgen und Glob-Muster (wie bei verwalteten Profilen); `wait --load networkidle` wird bei Profilen bestehender Sitzungen nicht unterstützt (es funktioniert bei verwalteten und unformatierten/Remote-CDP-Profilen). Upload-Hooks erfordern `ref` oder `inputRef`, jeweils eine Datei, ohne CSS-`element`. Dialog-Hooks unterstützen weder Timeout-Überschreibungen noch `dialogId`.
- **Dialogsichtbarkeit** – Antworten verwalteter Browseraktionen enthalten `blockedByDialog` und `browserState.dialogs.pending`, wenn eine Aktion einen modalen Dialog öffnet; Snapshots enthalten ebenfalls den Status ausstehender Dialoge. Antworten Sie mit `browser dialog --accept/--dismiss --dialog-id <id>`, solange ein Dialog aussteht. Außerhalb von OpenClaw behandelte Dialoge erscheinen unter `browserState.dialogs.recent`.
- **Nur bei verwalteten Profilen verfügbare Funktionen** – PDF-Export, Download-Abfangen und `responsebody` erfordern weiterhin den verwalteten Browserpfad.

</Accordion>

## Isolationsgarantien

- **Dediziertes Benutzerdatenverzeichnis**: Berührt niemals Ihr persönliches Browserprofil.
- **Dedizierte Ports**: Vermeidet `9222`, um Kollisionen mit Entwicklungsabläufen zu verhindern.
- **Deterministische Tab-Steuerung**: `tabs` gibt zuerst `suggestedTargetId` zurück, danach
  stabile `tabId`-Handles wie `t1`, optionale Bezeichnungen und die unverarbeitete `targetId`.
  Agenten sollten `suggestedTargetId` wiederverwenden; unverarbeitete IDs bleiben für
  Debugging und Kompatibilität verfügbar.

## Browserauswahl

Beim lokalen Start wählt OpenClaw den ersten verfügbaren Browser:

1. Chrome
2. Brave
3. Edge
4. Chromium
5. Chrome Canary

Sie können dies mit `browser.executablePath` überschreiben.

Plattformen:

- macOS: Prüft `/Applications` und `~/Applications`.
- Linux: Prüft gängige Chrome-/Brave-/Edge-/Chromium-Speicherorte unter `/usr/bin`,
  `/snap/bin`, `/opt/google`, `/opt/brave.com`, `/usr/lib/chromium` und
  `/usr/lib/chromium-browser` sowie von Playwright verwaltetes Chromium unter
  `PLAYWRIGHT_BROWSERS_PATH` oder `~/.cache/ms-playwright`.
- Windows: Prüft gängige Installationsorte.

## Steuerungs-API (optional)

Für Skripterstellung und Debugging stellt das Gateway eine kleine, **nur über Loopback erreichbare HTTP-
Steuerungs-API** sowie eine entsprechende `openclaw browser`-CLI bereit (Snapshots, Referenzen, erweiterte
Wartefunktionen, JSON-Ausgabe, Debugging-Abläufe). Die vollständige Referenz finden Sie unter
[Browser-Steuerungs-API](/de/tools/browser-control).

## Fehlerbehebung

Informationen zu Linux-spezifischen Problemen (insbesondere mit Snap Chromium) finden Sie unter
[Browser-Fehlerbehebung](/de/tools/browser-linux-troubleshooting).

Informationen zu Split-Host-Konfigurationen mit WSL2-Gateway und Windows Chrome finden Sie unter
[Fehlerbehebung für WSL2 + Windows + Remote-Chrome-CDP](/de/tools/browser-wsl2-windows-remote-cdp-troubleshooting).

### CDP-Startfehler im Vergleich zur SSRF-Blockierung der Navigation

Dies sind unterschiedliche Fehlerklassen, die auf unterschiedliche Codepfade verweisen.

- **CDP-Start- oder Bereitschaftsfehler** bedeutet, dass OpenClaw nicht bestätigen kann, dass die Browser-Steuerungsebene funktionsfähig ist.
- **SSRF-Blockierung der Navigation** bedeutet, dass die Browser-Steuerungsebene funktionsfähig ist, ein Seiten-Navigationsziel jedoch durch eine Richtlinie abgelehnt wird.

Typische Beispiele:

- CDP-Start- oder Bereitschaftsfehler:
  - `Chrome CDP websocket for profile "openclaw" is not reachable after start`
  - `Remote CDP for profile "<name>" is not reachable at <cdpUrl>`
  - `Port <port> is in use for profile "<name>" but not by openclaw`, wenn ein
    externer Loopback-CDP-Dienst ohne `attachOnly: true` konfiguriert ist
- SSRF-Blockierung der Navigation:
  - `open`, `navigate`, Snapshot- oder Tab-Öffnungsabläufe schlagen mit einem Browser-/Netzwerkrichtlinienfehler fehl, während `start` und `tabs` weiterhin funktionieren

Verwenden Sie diese minimale Abfolge, um die beiden Fälle zu unterscheiden:

```bash
openclaw browser --browser-profile openclaw start
openclaw browser --browser-profile openclaw tabs
openclaw browser --browser-profile openclaw open https://example.com
```

So interpretieren Sie die Ergebnisse:

- Wenn `start` mit `not reachable after start` fehlschlägt, beheben Sie zuerst die Probleme mit der CDP-Bereitschaft.
- Wenn `start` erfolgreich ist, aber `tabs` fehlschlägt, ist die Steuerungsebene weiterhin nicht funktionsfähig. Behandeln Sie dies als CDP-Erreichbarkeitsproblem, nicht als Problem der Seitennavigation.
- Wenn `start` und `tabs` erfolgreich sind, aber `open` oder `navigate` fehlschlägt, ist die Browser-Steuerungsebene aktiv und der Fehler liegt in der Navigationsrichtlinie oder auf der Zielseite.
- Wenn `start`, `tabs` und `open` alle erfolgreich sind, ist der grundlegende verwaltete Browser-Steuerungspfad funktionsfähig.

Wichtige Verhaltensdetails:

- Die Browserkonfiguration verwendet standardmäßig ein bei Fehlern geschlossenes SSRF-Richtlinienobjekt, selbst wenn Sie `browser.ssrfPolicy` nicht konfigurieren.
- Beim lokalen verwalteten Loopback-Profil `openclaw` überspringen CDP-Funktionsprüfungen absichtlich die Durchsetzung der Browser-SSRF-Erreichbarkeit für OpenClaws eigene lokale Steuerungsebene.
- Der Navigationsschutz ist davon getrennt. Ein erfolgreiches Ergebnis von `start` oder `tabs` bedeutet nicht, dass ein späteres Ziel von `open` oder `navigate` zulässig ist.

Sicherheitshinweise:

- Lockern Sie die Browser-SSRF-Richtlinie standardmäßig **nicht**.
- Bevorzugen Sie eng gefasste Hostausnahmen wie `hostnameAllowlist` oder `allowedHostnames` gegenüber einem umfassenden Zugriff auf private Netzwerke.
- Verwenden Sie `dangerouslyAllowPrivateNetwork: true` nur in bewusst vertrauenswürdigen Umgebungen, in denen der Browserzugriff auf private Netzwerke erforderlich und geprüft ist.

## Agentenwerkzeuge und Funktionsweise der Steuerung

Der Agent erhält **ein Werkzeug** für die Browserautomatisierung:

- `browser` – Diagnose/Status/Start/Stopp/Tabs/Öffnen/Fokussieren/Schließen/Snapshot/Screenshot/Navigieren/Ausführen

Zuordnung:

- `browser snapshot` gibt einen stabilen UI-Baum (AI oder ARIA) zurück.
- `browser act` verwendet die `ref`-IDs des Snapshots zum Klicken, Eingeben, Ziehen und Auswählen.
- `browser screenshot` erfasst Pixel (gesamte Seite, Element oder beschriftete Referenzen).
- `browser doctor` prüft die Bereitschaft von Gateway, Plugin, Profil, Browser und Tab.
- `browser` akzeptiert:
  - `profile`, um ein benanntes Browserprofil (openclaw, chrome oder Remote-CDP) auszuwählen.
  - `target` (`sandbox` | `host` | `node`), um auszuwählen, wo der Browser ausgeführt wird.
  - In Sandbox-Sitzungen erfordert `target: "host"` die Option `agents.defaults.sandbox.browser.allowHostControl=true`.
  - Wenn `target` weggelassen wird: Sandbox-Sitzungen verwenden standardmäßig `sandbox`, Sitzungen ohne Sandbox standardmäßig `host`.
  - Wenn eine browserfähige Node verbunden ist, kann das Tool Anfragen automatisch dorthin weiterleiten, sofern Sie nicht `target="host"` oder `target="node"` festlegen.

Dadurch bleibt der Agent deterministisch und instabile Selektoren werden vermieden.

## Verwandte Themen

- [Tool-Übersicht](/de/tools) – alle verfügbaren Agent-Tools
- [Sandboxing](/de/gateway/sandboxing) – Browsersteuerung in Sandbox-Umgebungen
- [Sicherheit](/de/gateway/security) – Risiken und Absicherung der Browsersteuerung
