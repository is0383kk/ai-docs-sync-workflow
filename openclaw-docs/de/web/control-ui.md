---
read_when:
    - Sie möchten das Gateway über einen Browser bedienen
    - Sie möchten Tailnet-Zugriff ohne SSH-Tunnel.
sidebarTitle: Control UI
summary: Browserbasierte Steuerungsoberfläche für den Gateway (Chat, Aktivität, Nodes, Konfiguration)
title: Steuerungsoberfläche
x-i18n:
    generated_at: "2026-07-26T18:53:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 069bad7f3c8fce46759893e16d2dac86047c0929d6d866d25ce3b080204c1180
    source_path: web/control-ui.md
    workflow: 16
---

Die Control UI ist eine kleine, vom Gateway bereitgestellte **Vite + Lit**-Single-Page-App:

- Standard: `http://<host>:18789/`
- optionales Präfix: `gateway.controlUi.basePath` festlegen (z. B. `/openclaw`)

Sie kommuniziert **direkt mit dem Gateway-WebSocket** am selben Port.

Während Sie eine laufende Sitzung beobachten, kann das Gateway das Utility-Modell dieses Agenten verwenden, um eine kompakte Statuszusammenfassung zu erstellen. Im Chat wird sie als einzeilige Statusanzeige dargestellt, die sich zu einer Karte mit der Einschätzung, dem Planfortschritt, den Pull Requests und der verstrichenen Zeit erweitern lässt. Die Karte kann einmal erweitert werden, wenn eine Ausführung hängen bleibt oder eine Eingabe benötigt; der Seitenchat `/btw` hat Vorrang vor der erweiterten Karte.

Die erweiterte Karte nimmt außerdem kurze Fragen zur Ausführung entgegen. Die Antworten verwenden ausschließlich die aktuelle Zusammenfassung des Beobachters und bereinigte, begrenzte Notizen, verbleiben für diese Sitzung im Browser und gelangen niemals in die Hauptausführung des Agenten oder unterbrechen sie. Wenn die Beobachtungen keine Antwort enthalten, gibt der Beobachter an, dass er sie nicht kennen kann.

Nach Eingang der ersten Zusammenfassung bestimmt sie anstelle heuristischer Live-Aktivitäten den Untertitel dieser Ausführung in der Seitenleiste. Eine abschließende Zusammenfassung über den erfolgreichen oder fehlgeschlagenen Abschluss bleibt sichtbar, solange die Sitzung ungelesen ist; anschließend zeigt die Zeile wieder ihren normalen Arbeitsuntertitel an.

Die Sitzungsbeobachtung ist standardmäßig aktiviert. Unter **Settings > Appearance > Sidebar** können Sie sie Gateway-weit deaktivieren, das ermittelte kleine Modell und dessen Herkunft prüfen, automatisches Routing auswählen, Utility-Aufgaben deaktivieren oder ein bestimmtes `agents.defaults.utilityModel` auswählen. Die entsprechenden Konfigurationsoptionen sind `gateway.controlUi.sessionObserver: false` und `agents.defaults.utilityModel: ""`.

## Schnell öffnen (lokal)

Wenn das Gateway auf demselben Computer ausgeführt wird, öffnen Sie [http://127.0.0.1:18789/](http://127.0.0.1:18789/) (oder [http://localhost:18789/](http://localhost:18789/)).

Wenn die Seite nicht geladen werden kann, starten Sie zuerst das Gateway: `openclaw gateway`.

<Note>
Bei nativen Windows-LAN-Bindungen können die Windows-Firewall oder eine von der Organisation verwaltete Gruppenrichtlinie die angegebene LAN-URL weiterhin blockieren, selbst wenn `127.0.0.1` auf dem Gateway-Host funktioniert. Führen Sie `openclaw gateway status --deep` auf dem Windows-Host aus; der Befehl meldet wahrscheinlich blockierte Ports, nicht übereinstimmende Profile und lokale Firewallregeln, die möglicherweise von Richtlinien ignoriert werden.
</Note>

Die Authentifizierung wird während des WebSocket-Handshakes über Folgendes bereitgestellt:

- `connect.params.auth.token`
- `connect.params.auth.password`
- Tailscale-Serve-Identitätsheader, wenn `gateway.auth.allowTailscale: true`
- Identitätsheader eines vertrauenswürdigen Proxys, wenn `gateway.auth.mode: "trusted-proxy"`

Die Gateway-Authentifizierung erfolgt vor der Gerätekopplung. Eine direkte Loopback-Verbindung umgeht weder die Token- noch die Passwortauthentifizierung. Im Einstellungsbereich des Dashboards wird ein Token für die aktuelle Browser-Tab-Sitzung und die ausgewählte Gateway-URL gespeichert; Passwörter werden nicht dauerhaft gespeichert. Nach der Kopplung kann der Browser bei späteren Verbindungen sein gespeichertes gerätespezifisches Token verwenden.

Beim Onboarding wird normalerweise ein Gateway-Token für die Authentifizierung mit einem gemeinsamen Geheimnis konfiguriert. Wenn das Gateway im Token-Modus ohne konfiguriertes Token startet, erzeugt es stattdessen für diesen Prozess ein flüchtiges Laufzeit-Token. Das Laufzeit-Token wird nicht in die Konfiguration geschrieben, daher kann `openclaw config get gateway.auth.token` es nicht abrufen, und ein Loopback-Browser ohne dieses Token wird abgewiesen. Führen Sie `openclaw doctor --generate-gateway-token` aus, starten Sie das Gateway neu und fügen Sie anschließend das konfigurierte Token in den Einstellungen der Control UI ein. Alternativ funktioniert die Passwortauthentifizierung, wenn `gateway.auth.mode` auf `"password"` gesetzt ist.

## Gerätekopplung (erste Verbindung)

Nachdem die Gateway-Authentifizierung erfolgreich war, erfordert die Verbindung von einem neuen Browser oder Gerät normalerweise eine **einmalige Kopplungsgenehmigung**, die als `disconnected (1008): pairing required` angezeigt wird.

<Warning>
Beim direkten Upgrade von einer Version, die die außer Betrieb genommene
Notfalleinstellung `gateway.controlUi.dangerouslyDisableDeviceAuth=true` verwendete,
hält OpenClaw den per Token/Passwort oder vertrauenswürdigem Proxy authentifizierten Zugriff auf die Control UI
für die reine Kopplungsbehebung verfügbar. Wenn der Browser unverschlüsseltes HTTP verwendet und keine Geräteidentität erstellen kann,
öffnen Sie ihn zunächst erneut über HTTPS oder localhost. Klicken Sie anschließend im
Warnbanner auf **Secure this browser**. Das Gateway kehrt erst zur normalen Durchsetzung der Geräteauthentifizierung zurück,
nachdem ein signierter Browser ausdrücklich gekoppelt wurde; es erstellt oder genehmigt niemals eine
Identität für einen Browser ohne Gerät. Der Übergang ist nicht verfügbar, wenn
bereits ein anderes Betreibergerät gekoppelt ist. Sowohl der Start des Gateways als auch
`openclaw doctor --fix` melden diese Migration ausdrücklich, anstatt
den alten Schlüssel stillschweigend zu verwerfen.
</Warning>

<Steps>
  <Step title="Ausstehende Anfragen auflisten">
    ```bash
    openclaw devices list
    ```
  </Step>
  <Step title="Nach Anfrage-ID genehmigen">
    ```bash
    openclaw devices approve <requestId>
    ```
  </Step>
</Steps>

Wenn der Browser die Kopplung mit geänderten Authentifizierungsdetails (Rolle/Bereiche/öffentlicher Schlüssel) erneut versucht, wird die vorherige ausstehende Anfrage ersetzt und eine neue `requestId` erstellt; führen Sie `openclaw devices list` vor der Genehmigung erneut aus.

Der Wechsel eines bereits gekoppelten Remote-Browsers vom Lesezugriff zum Schreib-/Administratorzugriff wird als Genehmigungsupgrade und nicht als stillschweigende erneute Verbindung behandelt: OpenClaw lässt die alte Genehmigung aktiv, blockiert die erneute Verbindung mit den umfassenderen Rechten und fordert Sie auf, die neuen Berechtigungsbereiche ausdrücklich zu genehmigen. Eine geeignete direkte Loopback-Verbindung der Control UI kann das Upgrade nach erfolgreicher Authentifizierung stillschweigend genehmigen.

Nach der Genehmigung wird das Gerät gespeichert und erfordert keine erneute Genehmigung, sofern Sie diese nicht mit `openclaw devices revoke --device <id> --role <role>` widerrufen. Informationen zur Token-Rotation, zum Widerruf und zum Ablauf der erstmaligen Genehmigung für Paperclip / `openclaw_gateway` finden Sie unter [Geräte-CLI](/de/cli/devices).

<Note>
- Direkte lokale Control-UI-Verbindungen von einem Loopback-TCP-Peer (`127.0.0.1` oder `::1`, normalerweise über `localhost` aufgerufen) ohne Weiterleitungs-/Proxy-Header können die Gerätekopplung erst automatisch genehmigen, nachdem die Gateway-Authentifizierung erfolgreich war und der Browser eine Geräteidentität vorlegt. Im Token-/Passwortmodus benötigt die erste Verbindung weiterhin das konfigurierte gemeinsame Geheimnis; diese automatische Genehmigung umgeht das Token nicht.
- Eine direkte Loopback-Verbindung benötigt nur dann kein gemeinsames Geheimnis, wenn `gateway.auth.mode: "none"` ausdrücklich konfiguriert ist. Dadurch wird die Gateway-Authentifizierung deaktiviert; dies ist nicht die empfohlene Einrichtung der Control UI. Im Tailscale-Serve-Modus und im Modus mit vertrauenswürdigem Proxy muss kein gemeinsames Geheimnis eingefügt werden, wenn die jeweiligen Identitätsprüfungen erfolgreich sind.
- Tailscale Serve kann den Kopplungsdurchlauf für Betreiber-Sitzungen der Control UI überspringen, wenn `gateway.auth.allowTailscale: true`, die Tailscale-Identität verifiziert wird und der Browser seine Geräteidentität vorlegt. Browser ohne Geräteidentität und Verbindungen mit Node-Rolle durchlaufen weiterhin die normalen Geräteprüfungen.
- Direkte Tailnet-Bindungen und Browserverbindungen über das LAN erfordern weiterhin eine ausdrückliche Genehmigung. Browserprofile ohne Geräteidentität können die automatische Loopback-Genehmigung nicht verwenden.
- Jedes Browserprofil erzeugt eine eindeutige Geräte-ID. Daher ist beim Wechsel des Browsers oder beim Löschen der Browserdaten eine erneute Kopplung erforderlich.

</Note>

## Mobilgerät koppeln

Ein bereits gekoppelter Administrator kann den iOS-/Android-Verbindungs-QR-Code erstellen, ohne ein Terminal zu öffnen:

<Steps>
  <Step title="Mobile Kopplung öffnen">
    Wählen Sie **Devices** aus und klicken Sie anschließend auf der Karte **Devices** auf **Pair mobile device**.
  </Step>
  <Step title="Telefon verbinden">
    Öffnen Sie in der mobilen OpenClaw-App **Settings** → **Gateway** und scannen Sie den QR-Code. Alternativ können Sie den Einrichtungscode kopieren und einfügen.
  </Step>
  <Step title="Verbindung bestätigen">
    Die offizielle iOS-/Android-App stellt automatisch eine Verbindung her. Wenn **Pending approval** eine Anfrage anzeigt, prüfen Sie deren Rolle und Berechtigungsbereiche, bevor Sie sie genehmigen.
  </Step>
</Steps>

Zum Erstellen eines Einrichtungscodes ist `operator.admin` erforderlich; für Sitzungen ohne diese Berechtigung ist die Schaltfläche deaktiviert. Ein Einrichtungscode enthält kurzlebige Bootstrap-Anmeldedaten. Behandeln Sie daher den QR-Code und den kopierten Code wie ein Passwort, solange sie gültig sind. Für die Remote-Kopplung muss das Gateway zu `wss://` aufgelöst werden (beispielsweise über Tailscale Serve/Funnel); unverschlüsseltes `ws://` ist auf Loopback- und private LAN-Adressen beschränkt. Die vollständigen Sicherheits- und Fallback-Details finden Sie unter [Kopplung](/de/channels/pairing#pair-from-the-control-ui-recommended).

## Persönliche Identität (browserlokal)

Die Control UI unterstützt eine browserbezogene persönliche Identität (Anzeigename und Avatar), die ausgehenden Nachrichten zugeordnet wird, um sie in gemeinsam genutzten Sitzungen zuordnen zu können. Sie befindet sich im Browserspeicher, ist auf das aktuelle Browserprofil beschränkt und wird weder mit anderen Geräten synchronisiert noch serverseitig über die normalen Metadaten zur Autorenschaft der von Ihnen gesendeten Nachrichten hinaus gespeichert. Durch das Löschen der Websitedaten oder einen Browserwechsel wird sie zurückgesetzt.

Das Überschreiben des Assistentenavatars folgt demselben browserlokalen Muster: Hochgeladene Überschreibungen überlagern die vom Gateway ermittelte Identität lokal und werden niemals über `config.patch` an das Gateway und zurück übertragen. Das gemeinsame Konfigurationsfeld `ui.assistant.avatar` steht weiterhin für Nicht-UI-Clients zur Verfügung, die das Feld direkt schreiben.

## Endpunkt für die Laufzeitkonfiguration

Die Control UI ruft ihre Laufzeiteinstellungen von `/control-ui-config.json` ab, relativ zum Basispfad der Control UI des Gateways aufgelöst (beispielsweise `/__openclaw__/control-ui-config.json` unter dem Basispfad `/__openclaw__/`). Dieser Endpunkt wird durch dieselbe Gateway-Authentifizierung geschützt wie die übrige HTTP-Oberfläche: Nicht authentifizierte Browser können ihn nicht abrufen, und ein erfolgreicher Abruf erfordert ein gültiges Gateway-Token/-Passwort, eine Tailscale-Serve-Identität oder eine Identität eines vertrauenswürdigen Proxys.

## Status des Gateway-Hosts

Öffnen Sie **Settings → General**, um die Karte **Gateway Host** mit dem Gateway-Rechner, der LAN-Adresse, dem Betriebssystem, der Laufzeit, der Betriebszeit, der CPU-Auslastung, dem Arbeitsspeicher und dem Speicherplatz des Status-Volumes anzuzeigen. Solange die Karte sichtbar ist, wird sie alle 10 Sekunden über den Gateway-RPC `system.info` aktualisiert, der den Berechtigungsbereich `operator.read` erfordert. Bei älteren Gateways und Verbindungen ohne diesen Berechtigungsbereich wird die Karte nicht angezeigt.

## Sprachunterstützung

Die Control UI lokalisiert sich beim ersten Laden anhand des Gebietsschemas Ihres Browsers. Um es später zu ändern, öffnen Sie **Settings -> General -> Language** (die Auswahl befindet sich auf der Seite General und nicht unter Appearance).

- Unterstützte Gebietsschemas: `en`, `ar`, `de`, `es`, `fa`, `fr`, `hi`, `id`, `it`, `ja-JP`, `ko`, `nl`, `pl`, `pt-BR`, `ru`, `th`, `tr`, `uk`, `vi`, `zh-CN`, `zh-TW`
- Nicht englische Übersetzungen werden im Browser verzögert geladen.
- Das ausgewählte Gebietsschema wird im Browserspeicher gespeichert und bei späteren Besuchen erneut verwendet.
- Fehlende Übersetzungsschlüssel greifen auf Englisch zurück.

Dokumentationsübersetzungen werden für dieselben nicht englischen Gebietsschemas erstellt, aber die integrierte Mintlify-Sprachauswahl der Dokumentationswebsite listet nur Gebietsschemacodes auf, die Mintlify akzeptiert. Die thailändische (`th`) und persische (`fa`) Dokumentation wird weiterhin im Veröffentlichungs-Repository erstellt; möglicherweise wird sie in dieser Auswahl erst angezeigt, wenn Mintlify diese Codes unterstützt.

## Darstellungsthemen

Der Bereich Appearance enthält die integrierten Designs Claw, Knot und Dash (Claw ist die Standardeinstellung) sowie einen browserlokalen tweakcn-Importplatz. Um ein Design zu importieren, öffnen Sie den [tweakcn-Editor](https://tweakcn.com/editor/theme), wählen oder erstellen Sie ein Design, klicken Sie auf **Share** und fügen Sie den kopierten Link unter Appearance ein. Der Importer akzeptiert außerdem `https://tweakcn.com/r/themes/<id>`-Registry-URLs, Editor-URLs wie `https://tweakcn.com/editor/theme?theme=amethyst-haze`, relative `/themes/<id>`-Pfade, rohe Design-IDs und Namen von Standarddesigns wie `amethyst-haze`.

Importierte Designs werden ausschließlich im aktuellen Browserprofil gespeichert; sie werden nicht in die Gateway-Konfiguration geschrieben und nicht zwischen Geräten synchronisiert. Beim Ersetzen des importierten Designs wird der eine lokale Platz aktualisiert; beim Löschen wird zu Claw zurückgewechselt, falls das importierte Design aktiv war.

Appearance verfügt außerdem über die Einstellung Text size. Sie gilt für Chattext, Editor-Text, Werkzeugkarten und Chat-Seitenleisten und hält Texteingaben auf mindestens 16px, damit Mobile Safari beim Fokussieren nicht automatisch vergrößert.

Design, Designmodus, Textgröße, Sprache und Einstellungen für die Chatdarstellung werden über die Gateway-Konfiguration (`ui.prefs`) synchronisiert. Dadurch stehen sie geräteübergreifend zur Verfügung, und Agenten können sie über die Genehmigungsschranke ändern — verbundene Clients übernehmen Änderungen unmittelbar über die `config.changed`-Benachrichtigung des Gateways. Jeder Browser hält für einen sofortigen Start eine lokale Kopie vor; Clients, die die Konfiguration nicht schreiben können (Betrachterbereich, offline), behalten Änderungen nur auf dem jeweiligen Gerät. Siehe [Konfigurationsreferenz](/de/gateway/configuration-reference#ui).

## OpenClaw-Systempflege

Öffnen Sie **Einstellungen → OpenClaw fragen**, um mit dem Agenten für Systemeinrichtung und -reparatur zu kommunizieren. Außerhalb des Onboardings kann diese Seite pro Besuch höchstens einen ausblendbaren Ereignis-Chip anzeigen. Bei routinemäßigem Gateway-Datenverkehr bleibt sie stumm und reagiert nur auf Zustandsübersichten, die einen deaktivierten Konfigurations-Neulader, eine Trennung oder Beeinträchtigung eines konfigurierten Kanals, eine fehlgeschlagene Kanalprüfung oder nicht verfügbare Kanal-Anmeldedaten melden. Ein neueres Ereignis ersetzt den ausstehenden Chip nur, wenn es schwerwiegender ist; durch Ausblenden oder Verwenden des Chips werden Ereignisaufforderungen für diesen Besuch stummgeschaltet. Durch Klicken auf den Chip wird dessen Diagnosefrage als echte `openclaw.chat`-Nachricht gesendet, sodass das Transkript die Anfrage aufzeichnet und OpenClaw die Diagnose durchführt. Während des Onboardings werden diese Ereignis-Chips nie angezeigt.

## Plugins verwalten

Öffnen Sie **Plugins** in der Seitenleiste oder verwenden Sie `/settings/plugins` relativ zum
konfigurierten Basispfad der Control UI, um Plugins zu durchsuchen und zu verwalten, ohne
die Control UI zu verlassen. Beispielsweise verwendet der Basispfad `/openclaw`
den Pfad `/openclaw/settings/plugins`. Die Seite ist immer verfügbar, selbst wenn alle
optionalen Plugins deaktiviert sind.

Plugins ist ein zentraler Bereich mit vier Registerkarten: **Installiert** und **Entdecken** verwalten Plugin-
Code unter `/settings/plugins`, **Skills** enthält die agentenspezifische Skill-Verwaltung unter
`/skills`, und **Workshop** enthält die Prüfung von Skill-Workshop-Vorschlägen unter
`/skills/workshop`. Jede Registerkarte behält ihre eigene URL, während die Seitenleiste
für alle einen einzigen Eintrag „Plugins“ anzeigt.

Die Registerkarte **Installiert** zeigt den vollständigen lokalen Bestand nach Kategorien gruppiert und mit
Übersichtszahlen. Jede Zeile öffnet eine Detailansicht; über das Überlaufmenü (`…`)
kann das Plugin aktiviert oder deaktiviert werden, und für extern installierte Plugins steht **Entfernen** zur Verfügung.
Außerdem werden konfigurierte [MCP-Server](/de/cli/mcp) aufgeführt, die direkt
hinzugefügt, deaktiviert und entfernt werden können. Dieselben Serversteuerelemente befinden sich unter **Einstellungen → MCP**.
Die Registerkarte **Entdecken** ist der Store: hervorgehobene, in OpenClaw enthaltene Plugins,
offizielle externe Plugins und mit einem Klick installierbare MCP-Konnektoren für verbreitete Dienste.
Eingaben in das Suchfeld durchsuchen
[ClawHub](https://clawhub.ai/plugins) direkt und fügen einen Abschnitt **Von ClawHub**
mit Downloadzahlen und Kennzeichnungen zur Quellenüberprüfung hinzu. Deep Links können
mit `/settings/plugins?tab=discover` direkt auf den Store verweisen.

Die Registerkarte **Skills** enthält den Skill-Statusbericht, Schalter zum Aktivieren und Deaktivieren, die Eingabe
von API-Schlüsseln und die direkte ClawHub-Skill-Suche, jeweils beschränkt auf den ausgewählten Agenten. Die
Registerkarte **Workshop** enthält das Skill-Workshop-Board und den heutigen Prüfablauf für
[Skill-Vorschläge](/de/tools/skill-workshop). **Skill-Ideen finden** prüft ein begrenztes
Fenster umfangreicher Sitzungen von der neuesten bis zur ältesten und hinterlegt alle Ergebnisse als
ausstehende Vorschläge. Das Panel zeigt die kumulative Abdeckung; **Frühere Arbeit durchsuchen**
setzt die Suche ab dem gespeicherten Cursor fort und wird zu **Neue Arbeit durchsuchen**, sobald der ältere
Verlauf vollständig verarbeitet wurde. Die manuelle Verlaufsprüfung funktioniert auch bei deaktiviertem autonomem Selbstlernen
und verwendet das für den ausgewählten Agenten konfigurierte Modell.

Enthaltene Plugins sind bereits auf dem Gateway vorhanden und zeigen **Aktivieren** oder
**Deaktivieren** anstelle von **Installieren**. Workboard ist beispielsweise in
OpenClaw enthalten, aber standardmäßig deaktiviert; seine Aktion lautet daher **Aktivieren**. Gebündelte Plugins
können nicht entfernt, sondern nur deaktiviert werden.

Das Lesen des Katalogs und die Suche in ClawHub erfordern `operator.read`. Das Installieren,
Aktivieren, Deaktivieren oder Entfernen eines Plugins sowie Änderungen an MCP-Servern erfordern
`operator.admin`; für Operatoren mit schreibgeschütztem Zugriff bleiben diese Aktionen deaktiviert.

ClawHub-Installationen werden über das Gateway ausgeführt und unterliegen denselben Prüfungen der Vertrauenswürdigkeit, Integrität
und Plugin-Installationsrichtlinien wie andere über das Gateway vermittelte Installationen. Das Installieren
oder Entfernen von Plugin-Code erfordert einen Neustart des Gateways. Das Aktivieren oder Deaktivieren eines
installierten Plugins kann ohne Neustart wirksam werden, wenn das Plugin und die aktuelle
Gateway-Laufzeit dies unterstützen; andernfalls meldet die Benutzeroberfläche, dass ein Neustart
erforderlich ist. OAuth-gestützte MCP-Konnektoren benötigen nach dem Hinzufügen eine einmalige
Ausführung von `openclaw mcp login <name>` über die CLI.

Die Seite konzentriert sich bewusst auf Bestand, Entdeckung, Installation, Aktivierung
und Entfernung. Verwenden Sie [`openclaw plugins`](/de/cli/plugins) für beliebige npm-, git- oder
lokale Pfadquellen, Aktualisierungen und erweiterte Plugin-Konfigurationen.

## Apps und Erweiterungen

Öffnen Sie **Apps** über das Seitenleistenmenü **Mehr**, die Befehlspalette oder das
Agentenmenü der Seitenleiste (**Apps herunterladen**) oder verwenden Sie `/apps` relativ zum
konfigurierten Basispfad der Control UI. Die Seite enthält Installationslinks für alle
OpenClaw-Begleitoberflächen: die [iOS](/de/platforms/ios)- und
[Android](/de/platforms/android)-Apps, die darin enthaltenen Begleit-Apps für Apple Watch und Wear OS,
die Desktop-Apps für [macOS](/de/platforms/macos), [Windows](/de/platforms/windows)
und [Linux](/de/platforms/linux), die
[Chrome-Erweiterung](/de/tools/chrome-extension), den integrierten Plugins-Bereich mit
[ClawHub](https://clawhub.ai) sowie die Discord-Community und Dokumentation.

## Navigation in der Seitenleiste

Die Seitenleiste ordnet alles rund um den Agenten an. Die Identitätszeile oben zeigt den aktiven Agenten; darunter beginnt der Abschnitt **Seiten** mit **Startseite** — der fortlaufenden Hauptsitzung des Agenten, versehen mit einer Kennzeichnung für ihren ungelesenen oder laufenden Zustand — gefolgt von den angehefteten Zielen (standardmäßig **Automatisierungen** und **Plugins**). Das Anpassungssteuerelement in der Überschrift „Seiten“ öffnet ein Menü mit allen weiteren Zielen, einschließlich **Nutzung** und von Plugins bereitgestellten Registerkarten, sowie **Angeheftete Elemente bearbeiten**; ein Rechtsklick auf den Navigationsbereich öffnet direkt den Editor für angeheftete Elemente. Die darunterliegende Sitzungsliste ist in Bereiche unterteilt: **Threads** für die Chatsitzungen des Agenten (die Hauptsitzung bleibt hinter „Startseite“; von ihr gestartete Sitzungen erscheinen hier als Threads der obersten Ebene, und benannte Threads werden ohne Typpräfix angezeigt), **Gruppen** für Gruppen- und Raumunterhaltungen und **Programmierung** für Sitzungen, die an einen verwalteten Worktree oder Ausführungs-Node gebunden sind (Zeilen zeigen eine `repo ⎇ branch`-Zeile sowie den Node-Host), ACP-gestützte Harness-Sitzungen und die Codex-/Claude-CLI-Kataloge. „Programmierung“ ist bei der ersten Ausführung eingeklappt und merkt sich Ihre Auswahl; die eingeklappte Überschrift behält die tatsächliche Anzahl bei und zeigt eine Aktivitätsanzeige, während enthaltene Sitzungen arbeiten. Benutzerdefinierte Gruppen (die Sitzungs-`category`) und **Angeheftet**-Zeilen stehen oberhalb von „Threads“, und die Zuordnung einer Sitzung zu einer benutzerdefinierten Gruppe hat immer Vorrang vor der automatischen Bereichsklassifizierung. Die Überschrift „Threads“ enthält die Sortiersteuerung (Erstellt oder Zuletzt aktualisiert, Gruppieren nach sowie einen gespeicherten **Status**-Filter für Aktiv, Archiviert oder Alle) und das **+**, das die Seite „Neue Sitzung“ öffnet. Archivierte Zeilen bleiben abgeblendet und mit einem Archivsymbol versehen an ihrer Position; sie tragen nicht zum Ungelesen- oder Aufmerksamkeitsstatus bei und werden bei der Abstammungshochstufung nicht berücksichtigt. Beim Öffnen einer Sitzung wird die Auswahlmarkierung verschoben, ohne die Zeilen neu zu sortieren. Übergeordnete Sitzungen mit kürzlich ausgeführten untergeordneten Sitzungen zeigen ein Offenlegungssymbol und die Anzahl der untergeordneten Sitzungen; durch Aufklappen können verschachtelte untergeordnete Sitzungen, ihr aktiver oder abgeschlossener Status und ihre Laufzeit geprüft werden, ohne die Seitenleiste zu verlassen. Bei Auswahl einer untergeordneten Sitzung wird ihr Chat geöffnet und ihr Pfad zu den übergeordneten Sitzungen automatisch eingeblendet. Untergeordnete Zeilen sind von der Gruppierung auf Stammebene, dem Anheften, Ziehen, der Mehrfachauswahl und der Seitennavigation ausgeschlossen; eingeklappte Bereiche verbrauchen nichts vom sichtbaren Seitenbudget. Sitzungen mit neuer Aktivität seit dem letzten Lesen zeigen einen Punkt für ungelesene Inhalte, und beim Öffnen werden sie als gelesen markiert. Ein Agent kann außerdem eine kurze, ablaufende Statuszeile veröffentlichen und optional mit einem ausgewählten bernsteinfarbenen Symbol um Aufmerksamkeit bitten; diese Angabe wird gelöscht, wenn Sie die Sitzung öffnen, die nächste Nachricht senden, sie ausdrücklich löschen oder ihre TTL abläuft. Lebenszykluszustände von Cloud-Workern verwenden ein Globus-Symbol; lokale und zurückgeholte Sitzungen zeigen kein Platzierungssymbol, da die lokale Ausführung der Standard ist. Jede Stammsitzungszeile besitzt ein Kontextmenü (Dreipunkt-Schaltfläche oder Rechtsklick) mit Anheften/Loslösen, Als ungelesen/gelesen markieren, Umbenennen, Forken, In Gruppe verschieben (einschließlich Neue Gruppe und Aus Gruppe entfernen), Archivieren oder Dearchivieren und Löschen; in Touch-Layouts bleiben die direkten Steuerelemente zum Anheften und für das Menü sichtbar. Mit Cmd-/Strg-Klick werden Stammzeilen einer Mehrfachauswahl hinzugefügt oder daraus entfernt, und mit Umschalt-Klick wird die Auswahl über die sichtbare Reihenfolge erweitert; beim Öffnen des Menüs einer ausgewählten Zeile werden anschließend Stapelaktionen angeboten (N als ungelesen/gelesen markieren, N in Gruppe verschieben, N archivieren, N löschen), die auf jede ausgewählte Sitzung angewendet werden, wobei für das stapelweise Löschen eine einzige Bestätigung genügt. Ziehen Sie eine Stammsitzung auf **Angeheftet**, um sie anzuheften, oder auf eine benutzerdefinierte Gruppe, um sie zu verschieben. Überschriften benutzerdefinierter Gruppen können eingeklappt, aufgeklappt oder zur Änderung ihrer Reihenfolge gezogen werden; Gruppennamen und ihre Reihenfolge werden im Gateway (`sessions.groups.*`) gespeichert und stehen dadurch browserübergreifend zur Verfügung, während der eingeklappte Zustand im Browserprofil verbleibt. Gruppenüberschriften besitzen ebenfalls ein Menü (Dreipunkt-Schaltfläche oder Rechtsklick) mit Gruppe umbenennen, Neue Gruppe und Gruppe löschen; beim Umbenennen oder Löschen einer Gruppe werden alle zugehörigen Sitzungen serverseitig aktualisiert, einschließlich archivierter Sitzungen. Beim Löschen einer Gruppe bleiben ihre Sitzungen erhalten und werden zurück in „Threads“ verschoben.

## Seite „Neue Sitzung“

Das **+** in der Überschrift der Sitzungsliste in der Seitenleiste öffnet unter `/new` einen ganzseitigen Entwurf: Bis zum Senden der ersten Nachricht wird nichts erstellt. Eine einheitliche Auswahl **Ort** legt den Arbeitsordner und für Admin-Operatoren das Ausführungsziel fest: **Gateway · lokal**, einen gekoppelten Node, der `system.run` bereitstellt, oder ein verfügbares Cloud-Profil. Der Ordner ist standardmäßig der Arbeitsbereich des Agenten; ein anderer absoluter Gateway-Pfad erfordert `operator.admin`, kann jedoch direkt ausgeführt werden, ohne ein Git-Checkout zu sein. Wenn der ausgewählte Gateway-Ordner ein Git-Checkout ist, bietet dieselbe Auswahl eine optionale **Worktree**-Isolation mit einer durch `worktrees.branches` gestützten Auswahl des Basis-Branches (ohne Abruf) und einem optionalen Worktree-Namen (der Branch wird zu `openclaw/<name>`). Cloud-Worker benötigen diesen verwalteten Worktree-Pfad; gekoppelte Nodes bieten ihn nie an. In der Fußzeile des Eingabebereichs werden Modell und Reasoning-Stufe der neuen Sitzung ausgewählt. Der Schalter **Inkognito** erstellt einen ausschließlich webbasierten Thread, dessen Sitzungseintrag, Transkript und Compaction-Zustand bis zum Neustart des Gateways im Arbeitsspeicher verbleiben; OpenClaw überspringt außerdem die automatische Speicherübertragung. Der Agent behält seine normalen Werkzeuge, sodass eine ausdrückliche Speicheranforderung oder ein werkzeuggestützter Dateischreibvorgang weiterhin Daten dauerhaft speichern kann. Der Modell-Provider verarbeitet die Nachrichten weiterhin, und inhaltsfreie Audit-Metadaten werden weiterhin aufgezeichnet. Bei Cloud-Starts werden die ausgewählten Modell- und Reasoning-Einstellungen gespeichert, bevor die Sitzung an ihren Worker übergeben wird.

Auf Gateways mit mehreren Benutzern können ausschließlich Verbindungen mit Admin-Berechtigungsbereich Inkognito-Threads erstellen oder anzeigen, und andere Sitzungen können weder über Agenten-Sitzungswerkzeuge noch über die Transkriptsuche auf sie zugreifen. Inkognito schützt vor Speicherung und anderen über das Gateway vermittelten Benutzern, nicht jedoch vor dem Gateway-Eigentümer oder Prozess-Operator, der aktive Sitzungen jederzeit beobachten kann.

**Ordner durchsuchen** öffnet den integrierten Verzeichnisbrowser der Ortsauswahl, der auf der nur für Admins verfügbaren Methode `fs.listDir` basiert und auf das ausgewählte Gateway oder den ausgewählten Node beschränkt ist. Gateway und durchsuchungsfähige Nodes listen ihr Dateisystem auf; ein ausführungsfähiger Node ohne `fs.listDir` akzeptiert weiterhin einen manuell eingegebenen absoluten Pfad. Zuletzt verwendete Orte können einen Ordner zusammen mit seinem zugehörigen Node wiederherstellen, ohne Pfade zwischen Hosts zu übertragen. Beim Absenden wird `sessions.create` mit der ersten Nachricht aufgerufen, sodass die Ausführung im selben Roundtrip beginnt und die Benutzeroberfläche zum Chat der neuen Sitzung wechselt. Wenn das Gateway die Sitzung erstellt, aber den ersten Sendevorgang ablehnt, bleiben die Eingabeaufforderung und der Fehler auch nach dem Neuladen im Chat erhalten; **Erneut versuchen** sendet die Nachricht über die bereits erstellte Sitzung, anstatt eine weitere zu erstellen.

Innerhalb von **Einstellungen** enthält die separate Seitenleiste **OpenClaw fragen** und beginnt mit dem Feld **Einstellungen durchsuchen**, über das Einstellungsabschnitte schnell gefunden werden können.

Auf Desktop-Websites befindet sich oben links im Inhaltsbereich eine feste Steuerungsgruppe — das Web-Pendant zur macOS-Titelleistenleiste — mit dem Schalter zum Einklappen der Seitenleiste (⌘B) und der Suchschaltfläche für die Befehlspalette (⌘K). Durch Klicken auf die Agentenidentitätszeile oben in der Seitenleiste wird das Agentenmenü geöffnet; **Startseite** öffnet die Hauptsitzung. Wenn Handlungsbedarf besteht — fehlgeschlagene oder überfällige Cron-Aufträge, bald ablaufende oder abgelaufene Modellauthentifizierung — werden kompakte Hinweischips über der Fußzeile der Seitenleiste angezeigt, die beim Anklicken zur zuständigen Seite führen. Die Identitätszeile zeigt den Avatar des Agenten (Identitätsbild oder Emoji), seinen Namen, einen Verbindungspunkt und einen live aktualisierten Untertitel. Das agentenspezifische Menü enthält den integrierten Agentenwechsler (bei Konfigurationen mit mehreren Agenten), **Neuer Agent**, „Was kann dieser Agent?“ und **Agenteneinstellungen**. Bei mehr als zehn Agenten wird ein Filterfeld angezeigt und angeheftete Agenten werden zuerst aufgeführt; Agenten können auf der Einstellungsseite „Agenten“ angeheftet oder gelöst werden, wobei die angeheftete Auswahl im Browserprofil gespeichert wird. Durch die Auswahl eines Agenten werden Chat sowie Nutzung, Automatisierungen, Aufgaben, Workboard und Sitzungen auf diesen Agenten beschränkt. Jede entsprechend beschränkte Seite bietet ein Steuerelement **Agent** mit **Alle Agenten** zum Aufheben der Beschränkung; dadurch wird der Umfang der gemeinsam genutzten Seite erweitert, ohne den konkreten Chat-Agenten zu ändern, während direkte Sitzungslinks weiterhin ihr jeweiliges Ziel öffnen. Die Einstellungsseite „Agenten“ behält ihre eigene `?agent=`-Auswahl bei und folgt nicht dem gemeinsamen Seitenumfang. Die Fußzeile besteht aus einer Identitätskarte über die gesamte Breite, die auch offline verfügbar bleibt und unter dem zuletzt bekannten Kontonamen **Verbindung wird wiederhergestellt…** anzeigt. Sie öffnet das App-/Kontomenü, in dem auf die Profilidentitäts-Kopfzeile **Einstellungen**, **Nutzung**, die Kopplung mit Mobilgeräten, **Apps herunterladen**, **Hilfe** (Hilfe, Discord, Dokumentation und Änderungsprotokoll), bei Bedarf eine Aktion für einen erneuten Offline-Versuch, der Versions-/Build-Chip und der Farbschema-Umschalter folgen. Der Build-Chip öffnet die Seite „Über“. Wenn das Gateway aus einem Quellcode-Checkout auf einem anderen Branch als `main` ausgeführt wird, zeigt die Fußzeile zusätzlich den Namen dieses Branches in Rot an, sodass ein Gateway außerhalb einer Release-Version auf einen Blick erkennbar ist (Release-Installationen zeigen ihn niemals an). Umschalt-Befehl-Komma auf Apple-Plattformen beziehungsweise Strg-Umschalt-Komma auf anderen Plattformen öffnet **Einstellungen**, ohne das einfache Browser-Tastenkürzel Befehl-Komma zu überschreiben. Beim Einklappen der Seitenleiste (⌘B oder der Schalter in der Steuerungsgruppe) wird sie vollständig ausgeblendet, sodass ein Arbeitsbereich über die gesamte Breite entsteht; im eingeklappten Zustand behält die Steuerungsgruppe oben links den Schalter zum Ausklappen und die Suche bei und erhält zusätzlich eine Schaltfläche für einen neuen Thread — entsprechend den Elementen, die die macOS-App nativ in ihrer Titelleiste bereitstellt. Die Seitenleiste ist auf dem Desktop das einzige Navigationselement; eine obere Leiste gibt es nicht. Bei schmalen Ansichtsbereichen wird die Seitenleiste durch ein seitlich eingeblendetes Panel hinter einer kompakten Kopfzeile ersetzt, die den Panel-Schalter, die Marke und die Suche der Befehlspalette enthält; auf Smartphones übernimmt der Chat diese Navigationszeile in seine Titelleiste, wobei sich die Menü- und Suchsteuerelemente neben dem Sitzungstitel befinden. In der macOS-App integriert die separate Kopfzeile den Freiraum der Titelleiste in eine einzige kompakte Leiste neben den Fenstersteuerelementen. Die Navigation verwendet den regulären Browserverlauf, sodass sie mit den Zurück-/Vorwärts-Schaltflächen des Browsers durchlaufen werden kann; die macOS-App ergänzt neben den Fenstersteuerelementen einen nativen Schalter für die Seitenleiste sowie Trackpad-Wischgesten. Im ausgeklappten Zustand befinden sich Zurück-/Vorwärts-Schaltflächen am rechten Rand der Seitenleiste, im eingeklappten Zustand native Schaltflächen für die Suche (Befehlspalette) und eine neue Sitzung.

Ausstehende Genehmigungen fügen ebenfalls einen Hinweischip über der Fußzeile der Seitenleiste hinzu;
wählen Sie ihn aus, um die zuständige Genehmigungsseite zu öffnen.

## Funktionsumfang (heute)

<AccordionGroup>
  <Accordion title="Chat und Gespräch">
    - Chatten Sie über Gateway-WS mit dem Modell (`chat.history`, `chat.send`, `chat.abort`, `chat.inject`). Bei archivierten Sitzungen bleibt der Eingabebereich deaktiviert und ein Banner mit der Aktion **Archivierung aufheben** wird angezeigt, bevor die Unterhaltung fortgesetzt werden kann.
    - Bei Aktualisierungen des Chatverlaufs wird ein begrenztes Fenster der letzten Nachrichten mit Textbeschränkungen pro Nachricht angefordert, sodass der Browser bei großen Sitzungen nicht erst die vollständigen Transkriptdaten rendern muss, bevor der Chat verwendet werden kann.
    - Wenn der Mauszeiger über einem öffentlichen GitHub-Issue- oder Pull-Request-Link verweilt oder dieser per Tastatur fokussiert wird, werden Status, Titel, Autor, letzte Aktivität, Kommentare und Änderungsstatistiken angezeigt. Das verbundene Gateway ruft öffentliche Metadaten ab und speichert sie zwischen, ohne das Linkziel zu ändern, auch wenn die Benutzeroberfläche ein entferntes Gateway verwendet. Das Gateway verwendet `GH_TOKEN` oder `GITHUB_TOKEN`, sofern verfügbar, nachdem bestätigt wurde, dass das Repository öffentlich ist; andernfalls verwendet es die anonyme GitHub-API mit einem länger gültigen Cache.
    - Führen Sie Gespräche über Echtzeitsitzungen im Browser. OpenAI verwendet direktes WebRTC, Google Live verwendet über WebSocket ein eingeschränktes, einmalig nutzbares Browser-Token und ausschließlich im Backend ausgeführte Echtzeit-Sprach-Plugins verwenden den Relay-Transport des Gateways. Videofähige Browsersitzungen können in den Einstellungen eine gerätelokale Kamera auswählen oder in der Live-Vorschau zwischen Kameras wechseln; der Browser erfasst JPEG-Frames für den Echtzeit-Provider, ohne Kameravideo über das Gateway zu streamen. Vom Client verwaltete Provider-Sitzungen beginnen mit `talk.client.create`; Gateway-Relay-Sitzungen beginnen mit `talk.session.create`. Das Relay bewahrt die Provider-Anmeldedaten auf dem Gateway auf, während der Browser Mikrofon-PCM über `talk.session.appendAudio` streamt, `openclaw_agent_consult`-Tool-Aufrufe des Providers über `talk.client.toolCall` zur Anwendung der Gateway-Richtlinien und zur Verarbeitung durch das größere konfigurierte OpenClaw-Modell weiterleitet und die Sprachsteuerung aktiver Ausführungen über `talk.client.steer` oder `talk.session.steer` routet.
    - Streamen Sie Tool-Aufrufe und live aktualisierte Tool-Ausgabekarten im Chat (Agentenereignisse). Tool-Aktivitäten werden als nach Typ differenzierte Zeilen dargestellt: Shell-Befehle zeigen den Befehl mit Syntaxhervorhebung und einer Ausgabe im Terminalstil; unterstützte Bearbeitungs- und Schreibaufrufe zeigen begrenzte Inline-Diffs, sofern verfügbar Zeilennummern sowie `+added -removed`-Statistiken; aufeinanderfolgende Aufrufe werden zu einer Zusammenfassung wie „13 Befehle ausgeführt, 6 Dateien gelesen, 9 Dateien bearbeitet“ zusammengefasst. Während eine Ausführung aktiv ist, bestimmt der neueste laufende Aufruf die Überschrift der Gruppe. Klappen Sie eine Zeile aus, um die übrigen Argumente und die Rohausgabe zu prüfen.
    - Optionale KI-generierte Zweckbezeichnungen für komplexe Tool-Aufrufe (lange Shell-Befehle, Plugin-Tools mit vielen Argumenten), aktiviert mit `gateway.controlUi.toolTitles: true` (standardmäßig deaktiviert). Die Bezeichnungen stammen aus der gebündelten `chat.toolTitles`-Methode über das standardmäßige Routing für Hilfsmodelle — entweder ein explizites `utilityModel` (vom Betreiber ausgewählter Provider, wie bei anderen Hilfsaufgaben) oder andernfalls das deklarierte Standard-Kleinmodell des Sitzungs-Providers — und werden Gateway-seitig pro Agent zwischengespeichert. Wenn die optionale Funktion deaktiviert ist oder kein kostengünstiges Modell verwendet werden kann, behalten die Zeilen ihre deterministischen Bezeichnungen und es erfolgt kein Modellaufruf.
    - Starten oder verwerfen Sie kurzlebige, vom Modell vorgeschlagene Folgeaufgaben; angenommene Vorschläge öffnen eine neue Sitzung in einem verwalteten Worktree mit dem vorgeschlagenen Prompt.
    - Aktivitätsregisterkarte mit browserlokalen, vorrangig redigierten Zusammenfassungen der Live-Tool-Aktivitäten aus der bestehenden Bereitstellung von `session.tool`- beziehungsweise Tool-Ereignissen.

  </Accordion>
  <Accordion title="Kanäle, Sitzungen, Speicher">
    - Kanäle: Status integrierter sowie gebündelter/externer Plugin-Kanäle, QR-Anmeldung und kanalspezifische Konfiguration (`channels.status`, `web.login.*`, `config.patch`).
    - Bei Aktualisierungen durch Kanalprüfungen bleibt der vorherige Snapshot sichtbar, während langsame Provider-Prüfungen abgeschlossen werden; unvollständige Snapshots werden gekennzeichnet, wenn eine Prüfung oder ein Audit das Zeitbudget der Benutzeroberfläche überschreitet.
    - Threads (eine Arbeitsbereichsseite unter `/sessions`, mit der danebenliegenden Registerkarte **Worktrees**): Standardmäßig werden die Sitzungen konfigurierter Agenten aufgeführt; häufig verwendete Sitzungen können angeheftet, umbenannt, archiviert oder nach Inaktivität wiederhergestellt werden, veraltete Sitzungsschlüssel nicht mehr konfigurierter Agenten werden abgefangen und sitzungsspezifische Überschreibungen für Modell, Denkmodus, Schnelligkeit, Ausführlichkeit, Ablaufverfolgung und Schlussfolgerung können angewendet werden (`sessions.list`, `sessions.patch`). Ein dreistufiger Filter **Aktiv / Archiviert / Alle** steuert sowohl diese Seite als auch die Seitenleiste; bei „Alle“ werden archivierte Zeilen abgeblendet und ausdrücklich gekennzeichnet. Archivierte Sitzungen behalten ihre Transkripte, werden niemals automatisch bereinigt und bleiben zurückgestellt, bis ihre Archivierung ausdrücklich aufgehoben oder sie gelöscht werden. Zeilen zeigen für aktive Sitzungen mit Aktivitäten seit dem letzten Lesen einen Ungelesen-Punkt sowie Aktionen zum Markieren als ungelesen/gelesen (`sessions.patch { unread }`) und eine Aktion zum Forken, die das Transkript in eine neue Sitzung verzweigt (`sessions.create { parentSessionKey, fork: true }`). Übersichtskacheln über der Tabelle fassen die geladene Liste zusammen (Anzahl der Sitzungen, aktive Ausführungen, ungelesene Sitzungen, Gesamtzahl der Tokens und, sofern verfügbar, Anzahl der archivierten Sitzungen). Jede Zeile enthält ein Symbol für den Typ mit einem Punkt für aktive Ausführungen, der Status wird als einfacher Punkt mit Bezeichnung dargestellt und die Spalte „Tokens“ zeigt eine Auslastungsanzeige für das Kontextfenster, wenn die Sitzung Token- und Kontextgrößen meldet. Aktionen zur Zeilenverwaltung befinden sich in einem zeilenspezifischen Menü (Dreipunkt-Schaltfläche oder Rechtsklick), das dem Sitzungsmenü der Seitenleiste entspricht. Das Detailpanel der Zeile zeigt neben den anderen Sitzungsdetails auch die Agentenlaufzeit und die Ausführungsdauer.
    - Native Claude- und Codex-Kataloge in der Seitenleiste streamen jeweils einen Host und werden anschließend nach Änderungen der Node-Verbindung, beim Fokussieren der Seite und während der Sichtbarkeit höchstens alle 30 Sekunden abgeglichen. Katalogänderungen lösen einen schnelleren Folgedurchlauf aus, sodass in den nativen Tools erstellte Sitzungen ohne Neuladen der Control UI angezeigt werden. Zeilen von Claude Desktop behalten außerdem ihre lokale benutzerdefinierte Gruppenbezeichnung bei, sofern vorhanden; OpenClaw liest diese Zuordnung aus dem lokalen Speicher von Desktop und schreibt niemals darin.
    - Sitzungsgruppierung: Ein Steuerelement „Gruppieren nach“ organisiert die Sitzungstabelle nach benutzerdefinierten Gruppen, Kanal, Typ, Agent oder Datum in Abschnitte. Benutzerdefinierte Gruppen bleiben über `sessions.patch` (`category`) sitzungsspezifisch erhalten, sodass auch Sitzungen kategorisiert werden können, die über Nachrichtenkanäle (Discord, Telegram, WhatsApp, ...) gestartet wurden; Gruppen können durch Ziehen von Zeilen auf einen Abschnitt oder über die zeilenspezifische Gruppenauswahl zugewiesen und mit der Aktion „Neue Gruppe“ erstellt werden.
    - Speicher (eine Registerkarte auf der Seite „Agenten“, die auf den ausgewählten Agenten beschränkt ist): Dreaming-Status, Schalter zum Aktivieren/Deaktivieren und Leser für das Traumtagebuch (`doctor.memory.status`, `doctor.memory.dreamDiary`, `config.patch`).
    - Speicher importieren (`/memory-import`, erreichbar über die Registerkarte „Speicher“ der Seite „Agenten“): Vorschau und Kopieren lokaler Auto-Memory-Daten von Claude Code, konsolidierter Codex-Memory-Daten oder Hermes-Memory-Dateien in den Arbeitsbereich des ausgewählten Agenten (`migrations.memory.plan`, `migrations.memory.apply`).
    - Speicherangebot beim Onboarding: Wenn die Control UI im Onboarding-Modus geöffnet wird (`?onboarding=1`, von der Linux-Begleit-App nach der Erstinstallation verwendet), bietet ein einseitiger Dialog den Import erkannter Memory-Daten mit demselben Plan-/Anwendungsablauf an; beim Überspringen bleibt die Einstellungsseite als späterer Einstiegspunkt verfügbar.

  </Accordion>
  <Accordion title="Cron, Aufgaben, Plugins, Skills, Geräte, Ausführungsgenehmigungen">
    - Automatisierungen (Cron-Jobs): Statistikkarten (Anzahl der Automatisierungen, Anzahl der fehlgeschlagenen Ausführungen, Scheduler-Status, nächste Aktivierung) über einem Tab-Umschalter für Automatisierungen/Ausführungsverlauf; der Tab „Automatisierungen“ listet Jobs in einer filterbaren Tabelle auf (Alle/Aktiv/Pausiert, Suche, Zeitplan- und Filter für die letzte Ausführung, Aktionsmenü pro Zeile) und zeigt darunter Einstiegsvorschläge, während der Tab „Ausführungsverlauf“ die letzten Ausführungen aller Automatisierungen anzeigt (`cron.*`).
    - Aufgaben: fortlaufendes Verzeichnis aktiver und kürzlich ausgeführter Hintergrundaufgaben mit verknüpften Sitzungen und Abbruchmöglichkeit (`tasks.*`). Die Leiste „Hintergrundaufgaben“ im Chat gruppiert laufende und abgeschlossene Arbeiten; wählen Sie eine Zeile aus, um deren begrenzten Prompt sowie die Ausgabe- oder Fehlerzusammenfassung zu prüfen.
    - Plugins: Durchsuchen Sie den installierten Bestand und den kuratierten Store, durchsuchen Sie ClawHub, installieren und entfernen Sie Plugin-Code und aktivieren oder deaktivieren Sie installierte Plugins (`plugins.*`); in MCP-Server-Zeilen wird `mcp.servers` über die Konfigurationsmethoden bearbeitet.
    - Skills: Status, Aktivierung/Deaktivierung, Installation, Aktualisierung von API-Schlüsseln (`skills.*`).
    - Geräte: Ein gemeinsamer Bestand führt Datensätze gekoppelter Geräte, den Node-Katalog und die Live-Präsenz zusammen (`device.pair.list`, `node.list`, `system-presence`). Der Gateway-Host ist an erster Stelle angeheftet; gekoppelte Clients zeigen Verbindungsstatus, Rollen, Tokens, Funktionen und Befehle. Doppelte Kopplungen werden in einer aufklappbaren Gruppe zusammengefasst, und **N veraltete bereinigen** entfernt nach Bestätigung durch einen Administrator alle Offline-Duplikate, die automatisch genehmigt wurden (stilles lokales Verfahren, vertrauenswürdiges CIDR oder SSH-Verifizierung) oder aus der Zeit vor der Erfassung des Genehmigungsnachweises stammen. Einträge können entfernt werden (`node.pair.remove`, `device.pair.remove`), Gerätekopplungen und erneute Node-Genehmigungen werden direkt verarbeitet (`device.pair.*`, `node.pair.approve`/`reject`), und mobile Einrichtungscodes werden über dieselbe Karte erstellt.
    - Ausführungsgenehmigungen: Bearbeiten Sie Gateway- oder Node-Zulassungslisten und die Abfragerichtlinie für `exec host=gateway/node` (`exec.approvals.*`).

  </Accordion>
  <Accordion title="Konfiguration">
    - `~/.openclaw/openclaw.json` anzeigen/bearbeiten (`config.get`, `config.set`).
    - Die Einstellungsnavigation beginnt mit „OpenClaw fragen“ und gruppiert die Seiten anschließend nach Relevanz: Allgemein, Darstellung und Benachrichtigungen oben; Verbindungen (Verbindung, Kanäle, Kommunikation, Geräte); Agenten und Tools (Agenten, KI und Agenten, Modell-Provider, MCP, Automatisierung, Labs); Datenschutz und Sicherheit (Sicherheit, Genehmigungen); sowie System (Infrastruktur, Erweitert, Debugging, Protokolle, Info). „Allgemein“ ist eine kompakte Übersichtsseite mit Modellstandards, Sprache und Statistiken zum Gateway-Host; jede andere Einstellung befindet sich auf genau einer Seite.
    - Datenschutz und Sicherheit: kuratierte Zeilen für Gateway-Authentifizierung, Ausführungsrichtlinie, Browseraktivierung, Tool-Profil, Geräteauthentifizierung und mobile Kopplung über den schemabasierten Abschnitten `security`/`approvals`.
    - „Genehmigungen“ enthält einen nach Aktualität absteigend sortierten 30-Tage-Verlauf abgeschlossener Anfragen für Ausführungen, Plugins und Systemagenten. Filtern Sie nach Art oder blättern Sie durch ältere Zeilen, um die vom Gateway aufgezeichnete Entscheidung, Begründung, Quellsitzung und Zuordnung der entscheidenden Person zu prüfen.
    - „Labs“ stellt ausgelieferte experimentelle Schalter bereit. Code Mode und Swarm sind die aktuellen Einträge und speichern `tools.codeMode.enabled` und `tools.swarm.enabled` sofort; nicht ausgelieferte Experimente werden weder angezeigt noch schreiben sie spekulative Konfigurationsschlüssel.
    - Benachrichtigungen: Web-Push-Status des Browsers, Abonnieren/Abbestellen und Testversand.
    - Erweitert: jeder Konfigurationsabschnitt ohne eigene kuratierte Seite sowie der JSON5-Rohdateneditor (zuvor der erweiterte Modus der Seite „Allgemein“).
    - Die Modelleinrichtung (`/settings/model-setup`) ist eine Unterseite von „Modell-Provider“, die über deren Kopfzeile geöffnet wird.
    - Agenten: eine Einstellungsseite (**Einstellungen → Agenten**, `/settings/agents`) mit Tabs pro Agent (Übersicht, Dateien, Tools, Skills, Kanäle, Automatisierungen, Speicher). Im Tab „Übersicht“ wird die Identität des Agenten bearbeitet – Anzeigename, Emoji und ein Avatarbild, das im Browser herunterskaliert und in seiner Größe begrenzt wird, bevor `agents.update`. Beim Speichern werden die konfigurierten Identitätsfelder gespeichert und in `IDENTITY.md` des Workspace gespiegelt; konfigurierte Werte haben Vorrang vor manuellen Änderungen an denselben Dateifeldern.
    - Profil: eine Einstellungsseite, die die Identität des Standardagenten zusammen mit Nutzungsstatistiken für den gesamten Zeitraum zeigt – Tokens über die gesamte Laufzeit, Spitzentag, längste Sitzung, Aktivitätsserien, eine Token-Heatmap für ein ganzes Jahr, meistgenutzte Tools und Kanalhöhepunkte (`usage.cost`, `sessions.usage`).
    - MCP verfügt über eine eigene Einstellungsseite mit Serverzeilen (Transport, Aktivierungsstatus, Zusammenfassungen zu OAuth/Filtern/Parallelität), direkten Steuerelementen zum Hinzufügen/Aktivieren/Deaktivieren/Entfernen, gängigen Operatorbefehlen und dem bereichsspezifischen Konfigurationseditor für `mcp`. Die Seite „Plugins“ bleibt die zentrale Stelle für Konnektoren mit einem Klick und deren Erkennung.
    - Modell-Provider: eine Einstellungsseite, die jeden konfigurierten Modell-Provider mit seinem Markensymbol, Authentifizierungsstatus (`models.authStatus`), seiner Modellverfügbarkeit (`models.list`), aktuellen Tarif-/Kontingent-/Abrechnungsdaten, sofern der Provider diese meldet (`usage.status`), und den lokalen Sitzungsausgaben der letzten 30 Tage (`sessions.usage`) auflistet. Die Aktion „Aktualisieren“ liest den Anmeldedatenstatus und die Provider-Nutzung erneut ein.
    - Verbindung: eine Einstellungsseite (unter **Verbindungen**), die die Gateway-Verbindung des Dashboards verwaltet – WebSocket-URL, Gateway-Token, Passwort und Standardsitzungsschlüssel – sowie den neuesten Handshake-Schnappschuss (Status, Betriebszeit, Tick-Intervall, letzte Aktualisierung der Kanäle). Die Offline-Anmeldesperre behandelt den Fall einer getrennten Verbindung; auf dieser Seite wird die Verbindung im verbundenen Zustand bearbeitet.
    - Mit Validierung anwenden und neu starten (`config.apply`) und anschließend die zuletzt aktive Sitzung aktivieren.
    - Schreibvorgänge enthalten eine Schutzprüfung des Basis-Hashs, um das Überschreiben gleichzeitiger Änderungen zu verhindern.
    - Schreibvorgänge (`config.set`/`config.apply`/`config.patch`) prüfen vorab die aktive SecretRef-Auflösung für Referenzen in den übermittelten Konfigurationsdaten; nicht aufgelöste aktive Referenzen in der Übermittlung werden vor dem Schreiben abgelehnt.
    - Beim Speichern von Formularen werden veraltete geschwärzte Platzhalter verworfen, die nicht aus der gespeicherten Konfiguration wiederhergestellt werden können, während geschwärzte Werte erhalten bleiben, die weiterhin gespeicherten Geheimnissen zugeordnet sind.
    - Schema und Formulardarstellung stammen aus `config.schema` / `config.schema.lookup`, einschließlich der Felder `title`/`description`, passender UI-Hinweise, direkter Zusammenfassungen untergeordneter Elemente, Dokumentationsmetadaten für verschachtelte Objekt-/Platzhalter-/Array-/Kompositions-Nodes sowie Plugin- und Kanalschemata, sofern verfügbar. Der JSON-Rohdateneditor ist nur verfügbar, wenn der Schnappschuss sicher unverändert im Rohformat verarbeitet werden kann; andernfalls erzwingt die Control UI den Formularmodus.
    - „Auf gespeicherte Version zurücksetzen“ im JSON-Rohdateneditor behält die im Rohformat erstellte Struktur (Formatierung, Kommentare, `$include`-Layout) bei, statt einen abgeflachten Schnappschuss neu darzustellen. Dadurch bleiben externe Änderungen bei einer Zurücksetzung erhalten, sofern der Schnappschuss sicher unverändert verarbeitet werden kann.
    - Strukturierte SecretRef-Objektwerte werden in Formulartexteingaben schreibgeschützt dargestellt, um eine versehentliche Beschädigung durch Umwandlung von Objekten in Zeichenfolgen zu verhindern.

  </Accordion>
  <Accordion title="Nutzung">
    - Die aus Sitzungen abgeleitete Analyse von Tokens und geschätzten Kosten bleibt von der Provider-Abrechnung getrennt.
    - Provider-Karten rufen `usage.status` auf und zeigen aktuelle Tarifnamen, Kontingentzeiträume, Guthaben, Ausgaben und Budgets an, die von konfigurierten Provider-Plugins gemeldet werden.
    - Ein Fehler bei der Provider-Nutzung blockiert das Sitzungs-/Kosten-Dashboard nicht; nicht verfügbare Provider-Karten zeigen einen eigenen Fehlerstatus an.

  </Accordion>
  <Accordion title="Debugging, Protokolle, Aktualisierung">
    - Debugging: Status-/Integritäts-/Modellschnappschüsse, Ereignisprotokoll und manuelle RPC-Aufrufe (`status`, `health`, `models.list`).
    - Das Ereignisprotokoll enthält Aktualisierungs-/RPC-Zeitmessungen der Control UI, Zeitmessungen für langsames Chat-/Konfigurations-Rendering sowie Einträge zur Browserreaktionsfähigkeit bei langen Animationsframes oder lang laufenden Aufgaben, sofern der Browser diese PerformanceObserver-Eintragstypen bereitstellt.
    - Protokolle: Live-Anzeige der Gateway-Dateiprotokolle mit Filter-/Exportfunktion (`logs.tail`).
    - Aktualisierung: Führen Sie eine Paket-/Git-Aktualisierung und einen Neustart (`update.run`) mit einem Neustartbericht aus und fragen Sie anschließend nach der erneuten Verbindung `update.status` ab, um die ausgeführte Gateway-Version zu überprüfen.

  </Accordion>
  <Accordion title="Hinweise zum Automatisierungsbereich">
    - Durch Auswählen einer Zeile wird eine ganzseitige Detailansicht mit einem Schalter „Aktiv/Pausiert“ und „Jetzt ausführen“ in der Kopfzeile geöffnet („Bei Fälligkeit ausführen“, „Klonen“ und „Entfernen“ befinden sich im zugehörigen Menü); im Tab „Einstellungen“ wird die Automatisierung direkt bearbeitet (Prompt, Details, Häufigkeit, erweiterte Überschreibungen), und der Tab „Ausführungsverlauf“ zeigt die Ausführungen dieser Automatisierung an.
    - Einstiegsautomatisierungen unter der Tabelle füllen das Erstellungsformular mit einem bearbeitbaren Prompt und Zeitplan vorab aus.
    - Bei isolierten Aufgaben wird standardmäßig eine Zusammenfassung angekündigt; wechseln Sie für rein interne Ausführungen zu „Keine“.
    - Die Felder für Kanal/Ziel werden angezeigt, wenn „Ankündigen“ ausgewählt ist.
    - Der Webhook-Modus verwendet `delivery.mode = "webhook"`, wobei `delivery.to` auf eine gültige HTTP(S)-Webhook-URL festgelegt ist.
    - Für Aufgaben der Hauptsitzung sind die Übermittlungsmodi „Webhook“ und „Keine“ verfügbar.
    - Zu den erweiterten Bearbeitungssteuerelementen gehören „Nach Ausführung löschen“, „Agentenüberschreibung löschen“, exakte/gestaffelte Cron-Optionen, Überschreibungen für Agentenmodell/Denken und Schalter für eine Best-Effort-Übermittlung.
    - Die Formularvalidierung erfolgt direkt mit Fehlern auf Feldebene; ungültige Werte deaktivieren die Schaltfläche „Speichern“, bis sie korrigiert wurden.
    - Legen Sie `cron.webhookToken` fest, um ein dediziertes Bearer-Token zu senden; wenn es nicht angegeben wird, wird der Webhook ohne Authentifizierungsheader gesendet.
    - `cron.webhook` ist ein eingestellter Legacy-Fallback, der von der aktuellen Konfigurationsvalidierung abgelehnt wird. Führen Sie `openclaw doctor --fix` aus, um gespeicherte Jobs, die weiterhin `notify: true` verwenden, zu einer expliziten Webhook- oder Abschlussübermittlung pro Job zu migrieren und den alten Schlüssel zu entfernen.

  </Accordion>
</AccordionGroup>

## Assistentenspeicher importieren

Öffnen Sie **Einstellungen** → **Speicher importieren**, um lokalen Speicher von Codex oder Claude Code
in einen OpenClaw-Agenten zu übernehmen. Der Gateway erkennt unterstützten lokalen Speicher selbstständig
auf seinem Host. Daher importiert eine entfernte Control UI vom Gateway-Computer und nicht vom
Browser-Computer.

1. Wählen Sie den Zielagenten aus.
2. Prüfen Sie die erkannten Quellsammlungen und Markdown-Dateinamen. Dateiinhalte
   werden weder in der Planantwort gesendet noch auf der Seite angezeigt.
3. Wählen Sie die zu importierenden Sammlungen aus und bestätigen Sie. Beim Anwenden wird der Plan vor dem
   Schreiben neu erstellt, sodass veraltete Auswahlen sicher fehlschlagen.
4. Wenn Dateien bereits vorhanden sind, aktivieren Sie **Vorhandene Importe ersetzen**, aktualisieren Sie die
   Vorschau und bestätigen Sie den Austausch.

Codex importiert nur seine konsolidierten `MEMORY.md` und `memory_summary.md`. Claude
Code importiert Markdown aus den automatischen Speicherverzeichnissen des Projekts und einem konfigurierten
`autoMemoryDirectory`; Sitzungen, Einstellungen, Anweisungen oder
Anmeldedaten werden über diese Seite nicht importiert. Dateien werden unter `memory/imports/` im
ausgewählten Workspace kopiert, wo das aktive Speicher-Plugin sie indizieren kann. Quellen werden
niemals geändert.

Planung und Anwendung erfordern `operator.admin`. Bei jeder Anwendung wird ein verifiziertes
OpenClaw-Backup erstellt, sofern ein Zustand vorhanden ist, ein geschwärzter Migrationsbericht geschrieben und es werden
Sicherungen einzelner Elemente aufbewahrt, bevor vorhandene Zieldateien ersetzt werden. Unter
[Speicherübersicht](/de/concepts/memory#import-from-coding-assistants) finden Sie Informationen zu Pfaden und
Abrufverhalten.

## MCP-Seite

Die dedizierte MCP-Seite ist eine Operatoransicht für von OpenClaw verwaltete MCP-Server unter `mcp.servers`. Sie startet MCP-Transporte nicht selbstständig; verwenden Sie sie, um die gespeicherte Konfiguration zu prüfen und zu bearbeiten, und verwenden Sie anschließend `openclaw mcp doctor --probe`, wenn Sie einen Live-Nachweis des Servers benötigen.

Typischer Arbeitsablauf:

1. Öffnen Sie **MCP** über die Seitenleiste.
2. Prüfen Sie die Übersichtskarten auf die Gesamtzahl sowie die Anzahl aktivierter, OAuth-verwendender und gefilterter Server.
3. Prüfen Sie jede Serverzeile auf Transport, Aktivierungsstatus, Authentifizierung, Filter, Zeitüberschreitungen und Befehlshinweise.
4. Fügen Sie Server direkt auf der MCP-Seite hinzu, aktivieren, deaktivieren oder entfernen Sie sie. Wählen Sie ausdrücklich Streamable HTTP, SSE oder stdio aus; stdio-Befehlszeilen akzeptieren Argumente in Anführungszeichen, beispielsweise Pfade mit Leerzeichen. Verwenden Sie die Seite **Plugins** für Konnektoren mit nur einem Klick und die Erkennung.
5. Bearbeiten Sie den zugehörigen Konfigurationsabschnitt `mcp` für erweiterte Serverfelder wie Umgebungsvariablen, Arbeitsverzeichnisse, Header, TLS-/mTLS-Pfade, OAuth-Metadaten, Toolfilter und Codex-Projektionsmetadaten.
6. Verwenden Sie **Speichern**, um die Konfiguration zu schreiben, oder **Speichern und veröffentlichen**, wenn der laufende Gateway die geänderte Konfiguration anwenden soll.
7. Führen Sie `openclaw mcp status --verbose`, `openclaw mcp doctor --probe` oder `openclaw mcp reload` in einem Terminal aus, um statische Diagnosen, einen Live-Nachweis oder die Bereinigung der zwischengespeicherten Laufzeitumgebung durchzuführen.

Die Seite schwärzt URL-ähnliche Werte mit Anmeldedaten vor der Darstellung und setzt Servernamen in Befehlsausschnitten in Anführungszeichen, damit kopierte Befehle auch bei Leerzeichen oder Shell-Metazeichen funktionieren. Vollständige CLI- und Konfigurationsreferenz: [MCP](/de/cli/mcp).

## Registerkarte „Aktivität“

Die Registerkarte „Aktivität“ befindet sich unter **Einstellungen › System** neben „Protokolle“ und „Debuggen“. Sie ist ein flüchtiger, browserlokaler Beobachter für Live-Toolaktivitäten, der aus demselben Gateway-Ereignisstrom `session.tool` bzw. Tool-Ereignisstrom abgeleitet wird, der die Toolkarten im Chat versorgt. Sie fügt weder eine weitere Gateway-Ereignisfamilie noch einen Endpunkt, einen dauerhaften Aktivitätsspeicher, einen Metrik-Feed oder einen externen Beobachterstrom hinzu.

Aktivitätseinträge enthalten ausschließlich bereinigte Zusammenfassungen und geschwärzte, gekürzte Ausgabevorschauen. Werte von Toolargumenten werden nicht im Aktivitätsstatus gespeichert; die Benutzeroberfläche weist darauf hin, dass Argumente ausgeblendet sind, und erfasst nur die Anzahl der Argumentfelder. Die In-Memory-Liste ist an die aktuelle Browserregisterkarte gebunden, bleibt beim Navigieren innerhalb der Control UI erhalten und wird beim Neuladen der Seite, beim Wechseln der Sitzung oder durch **Löschen** zurückgesetzt.

## Operator-Terminal

Das andockbare Operator-Terminal ist standardmäßig deaktiviert. Um es zu aktivieren, setzen Sie `gateway.terminal.enabled: true` und starten Sie den Gateway neu. Das Terminal erfordert eine `operator.admin`-Verbindung und öffnet ein Host-PTY im Arbeitsbereich des aktiven Agenten. Neue Registerkarten folgen dem aktuell ausgewählten Chat-Agenten.

<Warning>
Das Terminal ist eine uneingeschränkte Host-Shell und übernimmt die Umgebung des Gateway-Prozesses. Aktivieren Sie es nur für vertrauenswürdige Operator-Bereitstellungen. OpenClaw verweigert Terminalsitzungen für Agenten mit `sandbox.mode: "all"`; wird ein aktiver Agent in diesen Modus versetzt, werden seine bestehenden und laufenden Terminalsitzungen geschlossen.
</Warning>

Verwenden Sie **Strg + Backtick**, um das Dock ein- oder auszublenden. Das Layout unterstützt das Andocken unten und rechts, passt seine Größe an den Browser-Viewport an und verwaltet mehrere Shell-Registerkarten. Informationen zu `gateway.terminal.enabled` und zur optionalen Außerkraftsetzung `gateway.terminal.shell` finden Sie unter [Gateway-Konfiguration](/de/gateway/configuration-reference#gateway).

Vom Eigentümer autorisierte Agenten ohne Sandbox können das Tool `terminal` für langwierige oder interaktive Arbeiten verwenden, die der Operator beobachten soll. Jeder Toolaufruf kann die eigenen Gateway-PTYs des Agenten öffnen, lesen, beschreiben, in der Größe ändern, schließen oder auflisten. Neue Sitzungen öffnen standardmäßig eine gemeinsam verbundene Registerkarte der Control UI, sodass Agent und Operator dieselbe Ausgabe sehen und beide Eingaben vornehmen oder die Größe ändern können. Der Agentenzugriff ist exakt auf die Sitzung beschränkt: Ein Agent kann weder vom Operator erstellte Terminals noch von einer anderen Agentensitzung geöffnete Terminals lesen oder steuern.

Ziehen Sie eine oder mehrere Dateien auf das aktive Terminal oder wählen Sie über die Büroklammer-Schaltfläche Dateien aus. OpenClaw stellt jede Datei auf dem Rechner bereit, dem das PTY gehört, und fügt am Cursor Shell-maskierte absolute Pfade ein; es drückt niemals die Eingabetaste und führt die Eingabe nicht aus. Eine kompakte Stapelanzeige zeigt die aktuelle Datei und die Anzahl der abgeschlossenen Übertragungen. Durch Abbrechen wird der verbleibende Stapel gestoppt, ohne Pfade einzufügen; eine fehlgeschlagene Übertragung bleibt sichtbar, sodass Sie ab dieser Datei erneut versuchen können, ohne bereits abgeschlossene Dateien nochmals hochzuladen. Bilder, PDFs, Archive und andere Dateitypen werden bis zu 16 MiB pro Datei akzeptiert. Bereitgestellte Dateien verwenden auf POSIX-Hosts ein privates temporäres Systemverzeichnis (Verzeichnismodus `0700`, Dateimodus `0600`) oder unter Windows ein Verzeichnis innerhalb der ACL-Grenze des Benutzerprofils sowie einen Bereinigungstimer von 24 Stunden. Verschieben oder kopieren Sie daher alle Dateien, die Sie behalten möchten.

Das Einfügen von Pfaden unterstützt PowerShell, `cmd.exe` und erkannte POSIX-Shells (`sh`, Bash, Dash, Ash, Ksh, Zsh und Fish), einschließlich Git Bash unter Windows. Andere Shell-Außerkraftsetzungen werden abgelehnt, weil sich deren Quotierungsregeln nicht sicher ableiten lassen; führen Sie den Gateway innerhalb von WSL aus, um ein natives WSL-Terminal und Linux-Uploadpfade zu erhalten. `cmd.exe`-Pfade, die `%` oder `!` enthalten, werden ebenfalls abgelehnt, da diese Shell die betreffenden Zeichen selbst innerhalb doppelter Anführungszeichen expandiert.

Codex- und Claude-Code-Sitzungen, die in der Sitzungsseitenleiste erkannt werden, können in ihrer nativen CLI innerhalb desselben Terminalbereichs geöffnet werden. Stellen Sie unter **Einstellungen › Chat** die Option **Codex-/Claude-Threads öffnen in** auf **Terminal**, damit ein normaler Klick auf eine Zeile `codex resume` oder `claude --resume` öffnet; standardmäßig wird weiterhin der schreibgeschützte OpenClaw-Viewer verwendet. Das Kontext- oder Drei-Punkte-Menü einer Zeile bietet stets beide Möglichkeiten, und der Viewer-Header enthält **Im Terminal öffnen**, wenn die Sitzung dafür geeignet ist.

Die Eignung wird pro Sitzung und Host bestimmt. Gateway-lokale Sitzungen starten den vom Provider bereitgestellten Fortsetzungsbefehl auf dem Gateway-Host. Sitzungen auf gekoppelten Nodes starten einen auf der Zulassungsliste stehenden Provider-Befehl auf der zuständigen Node und übertragen ausschließlich Ausgabe-, Eingabe- und Größenänderungsereignisse dieses PTYs; dadurch wird weder eine allgemeine Node-Shell offengelegt noch werden vom Browser bereitgestellte Befehle akzeptiert. Datei-Uploads verwenden den separaten, größenbegrenzten Node-Befehl `terminal.upload` und bleiben an die bereits geöffnete Terminalsitzung gebunden. Genehmigen Sie das Upgrade der Node-Kopplung, wenn dieser Befehl erstmals angezeigt wird. Nodes, die den passenden Befehl zum Fortsetzen des Terminals nicht bekannt geben, darunter eingebettete Worker-Bridges ohne Duplex-Streaming, stellen weiterhin den Viewer bereit und zeigen das Öffnen des Terminals als nicht verfügbar an; ältere Nodes können weiterhin ein Terminal ausführen, aber keine hineingezogenen Dateien empfangen.

Verbindungseigene Sitzungen überstehen Verbindungsabbrüche: Beim Neuladen einer Seite, beim Ruhezustand des Laptops oder bei einer kurzen Netzwerkunterbrechung wird die Sitzung am Gateway getrennt, anstatt beendet zu werden. Dieselbe Browserregisterkarte stellt die Verbindung nach dem Wiederverbinden erneut her und gibt die kürzlich erfolgte Ausgabe wieder. Getrennte verbindungseigene Sitzungen werden nach `gateway.terminal.detachedSessionTimeoutSeconds` beendet (standardmäßig 300 Sekunden; `0` stellt das Beenden bei Verbindungsabbruch wieder her). Das Verbinden mit einer dieser Sitzungen bleibt eine Übernahme nach Art von tmux.

Agenteneigene Sitzungen sind nicht an eine Browserverbindung gebunden. `terminal.attach` fügt jeden Browser als Betrachter hinzu, ohne die Eigentümerschaft zu übernehmen, und beim Schließen einer Viewer-Registerkarte wird nur dieser Browser getrennt. Das PTY bleibt bestehen, bis der zuständige Agent es schließt, sein Prozess beendet wird, eine Richtlinie es deaktiviert oder der Gateway heruntergefahren wird. `terminal.list` kennzeichnet jeden Eintrag als verbindungs- oder agenteneigen, und `terminal.text` ermöglicht einer Admin-Verbindung, die aktuelle Klartextausgabe zu lesen, ohne sich zu verbinden.

Das Terminal ist außerdem als bildschirmfüllendes, ausschließlich für das Terminal vorgesehenes Dokument unter `/?view=terminal` verfügbar. Die iOS- und Android-Apps betten diese Seite in ihre Terminalansichten ein und verwenden dabei die gespeicherten Gateway-Anmeldedaten erneut; die Verfügbarkeit richtet sich nach denselben Bedingungen `gateway.terminal.enabled` und `operator.admin`, und die Seite zeigt einen Hinweis an, wenn der verbundene Gateway das Terminal nicht bereitstellt.

## Browserbereich

Die Control UI enthält einen andockbaren Browserbereich, der den vom Gateway gesteuerten Browser – denselben, den Agenten über das [Browser-Tool](/de/tools/browser-control) steuern – in jedem gewöhnlichen Webbrowser darstellt; eine native Webview ist nicht erforderlich. Er wird angezeigt, wenn der verbundene Gateway einer `operator.admin`-Verbindung `browser.request` bekannt gibt; die Globus-Schaltfläche in der Arbeitsbereichsleiste des Threads blendet ihn ein oder aus. Der Bereich zeigt einen Live-Snapshot der Seite mit Registerkarten, einer bearbeitbaren URL-Leiste, Zurück-, Vorwärts- und Neuladefunktionen sowie einer Option zum Öffnen im eigenen Browser, lässt sich rechts oder unten andocken und leitet Klicks, Mausrad-Scrollen und einfache Tastatureingaben an die Remote-Seite weiter.

Zwei Erfassungsmodi stellen dem Agenten Seitenkontext bereit:

- **Kommentieren (Stift)**: Zeichnen Sie Freihandmarkierungen über die Seite. **An Chat senden** fügt die Striche in den Screenshot ein, hängt das Bild an den aktiven Chat-Editor an und füllt eine Eingabeaufforderung vor, die die Seiten-URL, den Titel und jeden markierten Bereich beschreibt, damit der Agent genau weiß, was Sie eingekreist haben.
- **Untersuchen (Zeiger)**: Bewegen Sie den Mauszeiger über die Seite, um das Element unter dem Cursor anzuzeigen (Selektor, zugänglicher Name, Rolle, Größe); klicken Sie, um die Details dieses Elements zusammen mit einem hervorgehobenen Screenshot über denselben Editor-Ablauf zu senden. Untersuchen, Mausrad-Scrollen und Zurück/Vorwärts erfordern `browser.evaluateEnabled` (standardmäßig aktiviert).

Die macOS-App behält ihre native Link-Browser-Seitenleiste für im Dashboard angeklickte Links bei; der Browserbereich funktioniert auch dort und dient auf allen anderen Plattformen zum Kommentieren von Seiten.

## Chatverhalten

<AccordionGroup>
  <Accordion title="Send and history semantics">
    - `chat.send` ist **nicht blockierend**: Es bestätigt sofort mit `{ runId, status: "started" }`, und die Antwort wird über `chat`-Ereignisse gestreamt. Vertrauenswürdige Control-UI-Clients können außerdem optionale ACK-Zeitmetadaten für die lokale Diagnose erhalten.
    - Chat-Uploads akzeptieren Bilder sowie Dateien, die keine Videos sind. Bilder behalten den nativen Bildpfad; andere Dateien werden als verwaltete Medien gespeichert und im Verlauf als Anhangslinks angezeigt.
    - Erneutes Senden mit demselben `idempotencyKey` gibt während der Ausführung `{ status: "in_flight" }` und nach Abschluss `{ status: "ok" }` zurück.
    - `chat.history`-Antworten sind zur Sicherheit der Benutzeroberfläche größenbegrenzt. Wenn Transkripteinträge zu groß sind, kann der Gateway lange Textfelder kürzen, umfangreiche Metadatenblöcke auslassen und übergroße Nachrichten durch einen Platzhalter (`[chat.history omitted: message too large]`) ersetzen.
    - Wenn eine sichtbare Assistentennachricht in `chat.history` gekürzt wurde, kann der Seitenleser den vollständigen, für die Anzeige normalisierten Transkripteintrag bei Bedarf über `chat.message.get` anhand von `sessionKey`, bei Bedarf aktivem `agentId` und Transkript-`messageId` abrufen. Wenn der Gateway weiterhin nicht mehr zurückgeben kann, zeigt der Leser einen ausdrücklichen Status „nicht verfügbar“ an, statt die gekürzte Vorschau stillschweigend zu wiederholen.
    - Vom Assistenten erzeugte Bilder werden als verwaltete Medienreferenzen dauerhaft gespeichert und über authentifizierte Gateway-Medien-URLs wieder bereitgestellt, sodass Neuladevorgänge nicht davon abhängen, dass rohe Base64-Bildnutzdaten in der Antwort des Chatverlaufs verbleiben.
    - Beim Rendern von `chat.history` entfernt die Control UI ausschließlich für die Anzeige bestimmte Inline-Direktiven-Tags aus dem sichtbaren Assistententext (zum Beispiel `[[reply_to_*]]` und `[[audio_as_voice]]`), XML-Nutzdaten von Tool-Aufrufen im Klartext (einschließlich `<tool_call>...</tool_call>`, `<function_call>...</function_call>`, `<tool_calls>...</tool_calls>`, `<function_calls>...</function_calls>` und gekürzter Tool-Aufrufblöcke) sowie durchgesickerte Modell-Steuerungstoken in ASCII- oder Vollbreitenschreibweise. Assistenteneinträge, deren gesamter sichtbarer Text ausschließlich aus dem exakten Stille-Token `NO_REPLY` / `no_reply` oder dem Heartbeat-Bestätigungstoken `HEARTBEAT_OK` besteht, werden ausgelassen.
    - Während eines aktiven Sendevorgangs und der abschließenden Aktualisierung des Verlaufs hält die Chatansicht lokale optimistische Benutzer- und Assistentennachrichten sichtbar, wenn `chat.history` kurzzeitig einen älteren Schnappschuss zurückgibt; das kanonische Transkript ersetzt diese lokalen Nachrichten, sobald der Gateway-Verlauf aufgeholt hat.
    - Live-`chat`-Ereignisse stellen den Zustellstatus dar, während `chat.history` aus dem dauerhaften Sitzungstranskript neu aufgebaut wird. Nach abschließenden Tool-Ereignissen lädt die Control UI den Verlauf neu und führt nur ein kleines optimistisches Ende zusammen; die Transkriptgrenze ist unter [WebChat](/de/web/webchat) dokumentiert.
    - `chat.inject` hängt eine Assistentennotiz an das Sitzungstranskript an und sendet ein `chat`-Ereignis für reine UI-Aktualisierungen (kein Agentenlauf, keine Kanalzustellung).
    - Die Seitenleiste führt jede geladene aktive Sitzung nach Agentenabschnitt und in den Bereichen „Angeheftet“, „Kanal“, „Arbeit“, benutzerdefinierte Gruppen und „Chats“ auf, mit einer einzigen Aktion „Neue Sitzung“, die den Entwurfsdialog öffnet. Beim Öffnen einer sichtbaren Zeile wird nur die Hervorhebung verschoben. Sitzungen können auf „Angeheftet“ gezogen werden, um sie anzuheften, oder auf eine benutzerdefinierte Gruppe beziehungsweise „Chats“, um sie zu verschieben; benutzerdefinierte Gruppen können ein- und ausgeklappt sowie per Drag-and-drop neu angeordnet werden, Gruppennamen und Reihenfolge werden über den Gateway synchronisiert, und der eingeklappte Zustand bleibt im Browser erhalten. Eine neue Dashboard-Sitzung erhält asynchron einen knappen generierten Titel aus ihrer ersten Nachricht, die kein Befehl ist; explizite Namen und die authentifizierte Absenderidentität bleiben getrennt, sodass Kontonamen niemals als generierte Titel verwendet werden. Setzen Sie `agents.defaults.utilityModel` (oder `agents.entries.*.utilityModel`), um diesen separaten Modellaufruf an ein kostengünstigeres Modell weiterzuleiten; wenn dieses separate Modell fehlschlägt, wird die Titelgenerierung einmal mit dem primären Modell wiederholt. Durch Aufklappen eines anderen Agentenabschnitts können die Sitzungen dieses Agenten durchsucht werden, ohne den geöffneten Chat zu verlassen.
    - Die Thread-Suche befindet sich in der Befehlspalette (⌘K oder die Suchschaltfläche in der Steuerungsgruppe oben links): Bei Eingabe einer Suchanfrage wird eine begrenzte Anzahl übereinstimmender Seiten agentenübergreifend durchsucht, interne untergeordnete und Cron-Zeilen werden herausgefiltert und sichtbare Treffer neben Navigationsbefehlen aufgeführt. Die Seite „Threads“ behält die vollständige durchsuchbare Liste mit Filtern bei.
    - Jede Seitenleistenzeile bietet direkten Zugriff auf das Anheften sowie ein vollständiges Kontextmenü für den Ungelesen-Status, Umbenennen, Forken, Gruppieren, Archivieren und Löschen. Für mehrfach ausgewählte Zeilen (Cmd-/Strg-Klick, Umschalt-Klick für Bereiche) steht ein Stapelmenü für den Ungelesen-Status, das Gruppieren, Archivieren und Löschen zur Verfügung; das stapelweise Archivieren oder Löschen bleibt deaktiviert, sofern nicht jede ausgewählte Sitzung archiviert werden kann. Ein aktiver Lauf und die Hauptsitzung eines Agenten können nicht archiviert werden. Beim Archivieren oder Löschen der aktuell ausgewählten Sitzung wechselt der Chat zurück zur Hauptsitzung dieses Agenten.
    - In der macOS-App verwendet das OpenClaw-Zeichen den ansonsten leeren nativen Titelleistenbereich neben den Fenstersteuerelementen, statt eine Zeile der Seitenleiste zu belegen.
    - Bei Desktop-Breiten bleiben die Chat-Steuerelemente in einer kompakten Zeile und werden beim Scrollen nach unten durch das Transkript eingeklappt; beim Scrollen nach oben, bei der Rückkehr zum Anfang oder beim Erreichen des Endes werden die Steuerelemente wiederhergestellt.
    - Der Sitzungskopf zeigt neben dem Arbeitsbereichs-Chip eine kleine Avatargruppe an, wenn andere Personen dieselbe Sitzung ansehen; sie führt bis zu vier Betrachter-Avatare mit einer Anzahl für weitere Betrachter auf und verschwindet, wenn Sie allein sind.
    - Aufeinanderfolgende identische Nachrichten, die nur Text enthalten, werden als eine Sprechblase mit einem Anzahl-Badge dargestellt. Nachrichten mit Bildern, Anhängen, Tool-Ausgaben oder Canvas-Vorschauen bleiben unzusammengefasst.
    - Sprechblasen für Benutzernachrichten enthalten Transkriptaktionen: eine beim Darüberfahren eingeblendete Rückspulschaltfläche (Bestätigungs-Popover mit der Option „Nicht erneut fragen“) sowie per Rechtsklick **Bis hierher zurückspulen** und **Von hier forken**. Beim Zurückspulen wird die Sitzung auf den Zustand unmittelbar vor dieser Nachricht zurückgesetzt und deren Text zum Bearbeiten und erneuten Senden in den Eingabebereich übernommen (`sessions.rewind`, `operator.admin`); beim Forken wird aus dem Präfix des aktiven Pfads vor der Nachricht eine neue Sitzung erstellt, geöffnet und ihr Eingabebereich mit demselben Text vorbelegt (`sessions.fork`, `operator.write`). Beide Aktionen sind während der Arbeit des Agenten deaktiviert und zeigen einen erklärenden Tooltip an, gelten nur für dauerhaft gespeicherte Benutzernachrichten und werden für Sitzungen abgelehnt, deren Unterhaltung einer externen Agenten-Harness gehört. Beim Zurückspulen wird nur der Chatkontext verschoben — Dateien und andere Seiteneffekte von Tools werden nicht rückgängig gemacht — und das Transkript vor dem Zurückspulen bleibt im nur anhängbaren Sitzungsspeicher erhalten. Wenn dieser Speicher mehrere Transkriptzweige enthält, zeigt die Chat-Titelleiste ein Zweigmenü mit der neuesten Nachricht, der Nachrichtenanzahl und der Aktualität jedes Zweigs an; durch Auswahl eines inaktiven Zweigs wird die aktuelle Sitzung auf diesen erhaltenen Pfad zurückgesetzt (`sessions.branches.list`, `operator.read`; `sessions.branches.switch`, `operator.admin`). Auch das Wechseln von Zweigen ist während der Arbeit des Agenten nicht verfügbar, und die Auswahl des bereits aktiven Zweigs führt an der RPC-Grenze zu einem typisierten No-op-Fehler. Die separate Ausblendaktion an Benutzersprechblasen blendet eine Nachricht nur im aktuellen Browser aus; die Nachricht verbleibt im Transkript und ist für den Agenten weiterhin sichtbar.
    - Wenn sich der Checkout einer Sitzung auf einem Nicht-Standard-Branch eines GitHub-Repositorys befindet, heftet die Chatansicht Pull-Request-Chips oberhalb des Eingabebereichs an: PR-Nummer, Repository, Branch, Diff-Anzahlen, eine CI-Kennzeichnung sowie der Status „Entwurf“, „Zusammengeführt“ oder „Geschlossen“, jeweils mit einem Link zum PR. Die Zeile zeigt höchstens zwei Chips an — Live-PRs (offen/Entwurf) zuerst — und eine Schaltfläche „Mehr anzeigen“ blendet den eingeklappten Verlauf zusammengeführter oder geschlossener PRs ein. Die CI-Kennzeichnung öffnet ein kleines Popover zur CI-Überwachung mit der Anzahl bestandener, fehlgeschlagener, laufender und übersprungener Prüfungen sowie einem Link zur Prüfungsseite des PRs. Die Erkennung erfolgt serverseitig über `controlUi.sessionPullRequests`, das die `GH_TOKEN`/`GITHUB_TOKEN` des Gateways wiederverwendet, sofern diese gesetzt sind. Wenn das Ratenlimit der GitHub-API erreicht ist, behalten die Chips den zuletzt bekannten Status bei und zeigen eine Warnung an, dass der Status möglicherweise veraltet ist; durch Verwerfen eines Chips wird er für diese Sitzung im aktuellen Browserprofil ausgeblendet. Bevor ein PR vorhanden ist, zeigt die Zeile den Branch selbst an — Repository, Branch-Name und die +/−-Größe des Diffs gegenüber der Merge-Basis des Standard-Branchs (committete und nicht committete Arbeit). Sobald der gepushte Branch vergleichbare Commits enthält, fügt die Zeile eine Schaltfläche „PR erstellen“ hinzu, die die GitHub-Seite für neue Pull Requests öffnet; davor wird die Zeile für eine Sitzung mit geänderten Dateien (committet, nicht committet oder nicht verfolgt) weiterhin angezeigt, jedoch ohne die Schaltfläche. Die Zeile blendet sich aus, solange ein offener PR oder PR-Entwurf vorhanden ist. Die Branch-Zeile basiert ausschließlich auf lokalem Git und bleibt daher auch bei einer Ratenbegrenzung durch GitHub verfügbar; sie zeigt dieselbe Warnung über einen möglicherweise veralteten Status an, da „kein PR gefunden“ erst nach Zurücksetzen des Limits als verlässlich gelten kann.
    - Das Diff-Panel der Sitzung zeigt, was der Checkout einer Sitzung tatsächlich geändert hat: Die Branch-Schaltfläche in der Arbeitsbereichsleiste oder der Chat-Titelleiste öffnet das Detailpanel mit einem dateiweisen Diff der Branch-, nicht committeten und nicht verfolgten Arbeit gegenüber der Merge-Basis des Standard-Branchs des Checkouts — Statuspunkt, Umbenennungspfeil, dateiweise +/−-Anzahlen, einklappbare Dateien und Markierungen „N unveränderte Zeilen“ zwischen den Hunks. Diffs werden serverseitig über die Gateway-Methode `sessions.diff` berechnet (Gültigkeitsbereich `operator.read`); Binärdateien und übergroße Dateien werden auf Einträge mit reinen Statistiken reduziert, und die Schaltfläche erscheint nur, wenn der verbundene Gateway `sessions.diff` bekannt gibt.
    - Jeder Chat-Bereich verfügt über eine Titelleiste. Klicken Sie auf den Sitzungstitel, um ihn umzubenennen; der Arbeitsbereichs-Chip kopiert den Checkout-Pfad oder Branch und kann lokale Gateway-Arbeitsbereiche im Dateimanager des Hosts anzeigen. Remote- und Exec-Node-Sitzungen behalten die Kopieraktionen bei, blenden die Anzeigeaktion jedoch aus.
    - Die Thread-Arbeitsbereichsleiste in jedem Chat-Bereich führt Thread-Dateien, Projektdateien und Artefakte auf. Standardmäßig ist sie am rechten Rand des Bereichs angedockt; ziehen Sie ihren Kopfbereich (oder verwenden Sie die Andockschaltfläche), um sie nach unten zu verschieben. Die Auswahl wird im aktuellen Browserprofil gespeichert. Eine eingeklappte Leiste belegt überhaupt keinen Platz: Öffnen Sie sie mit ⇧⌘B oder dem Datei-Umschalter in der Titelleiste erneut, der ein Badge mit der Anzahl geänderter Dateien trägt. Das separate Detailpanel für Dateien, Tools und Canvas bleibt davon unberührt.
    - Durch Klicken auf eine Dateireferenz im Chat, einen Dateipfad in einer aufgeklappten Tool-Karte zum Lesen/Bearbeiten/Schreiben oder eine Dateizeile in der Arbeitsbereichsleiste wird das Datei-Detailpanel geöffnet: eine Codeansicht auf CodeMirror-Basis mit Syntaxhervorhebung, Zeilennummern, Sprung zu einer Zeile, dateiinterner Suche, Kopieraktionen und einem Menü zum Öffnen in einem externen Editor. Wenn der Gateway einer `operator.admin`-Verbindung `sessions.files.set` bekannt gibt, fügt das Panel einen Bearbeitungsmodus mit Änderungsverfolgung und Speichern über Cmd/Strg-S hinzu; nicht gespeicherte Entwürfe bleiben beim Navigieren zwischen Dateien, Panels und Sitzungen im aktuellen Browser-Tab erhalten, bis sie ausdrücklich gespeichert oder verworfen werden. Speichervorgänge verwenden Compare-and-Swap anhand eines von `sessions.files.get` zurückgegebenen Inhalts-Hashes: Wenn sich die Datei seit dem Laden auf dem Datenträger geändert hat (beispielsweise weil der Agent weitergearbeitet hat), zeigt das Panel einen Konflikthinweis mit den Aktionen Reload (neuesten Inhalt übernehmen) und Overwrite (lokale Bearbeitung beibehalten) an. Schreibvorgänge durchlaufen dieselben dateisystemsicheren Arbeitsbereichsschutzmechanismen wie Lesevorgänge — Pfadbegrenzung, Ablehnung von symbolischen Links und Hardlinks sowie eine UTF-8-Obergrenze von 256 KB — und überschreiben ausschließlich vorhandene Dateien; der Editor erstellt oder löscht niemals Dateien.
    - Die Leiste für Hintergrundaufgaben in jedem Chat-Bereich führt die Hintergrundaufgaben und Subagenten des aktuellen Agenten auf (`tasks.list`, nach Agent begrenzt und durch `task`-Ereignisse aktuell gehalten): Laufende Arbeit zeigt einen live aktualisierten Timer für die verstrichene Zeit, die Anzahl der Tool-Verwendungen, das derzeit verwendete Tool und ein Steuerelement zum Stoppen an; der einklappbare Abschnitt für abgeschlossene Aufgaben ergänzt die Laufzeiten; und ein Link „Transkript anzeigen“ öffnet die untergeordnete Sitzung der Aufgabe im Bereich. Öffnen Sie die Leiste mit dem Aktivitätsumschalter in der Titelleiste; der Aufgaben-Schnappschuss wird vorab geladen und zeigt daher ein Badge mit der Anzahl laufender Aufgaben an, ohne dass die Leiste zuvor geöffnet werden muss. Die Seite „Aufgaben“ bleibt das vollständige agentenübergreifende Verzeichnis.
    - Die Arbeitsbereichsleiste, die Leiste für Hintergrundaufgaben und das Detailfenster passen sich jeweils an die Breite des eigenen Bereichs statt an die des Fensters an: In einem schmalen Bereich oder kompakten Fenster werden beide Leisten als untere Streifen dargestellt (die Steuerelemente zum seitlichen Andocken bleiben ausgeblendet, bis der Bereich breiter wird; die Arbeitsbereichsleiste hat Vorrang für den seitlichen Platz, wenn nur eine Spalte hineinpasst), und das Detailfenster wird unterhalb des Threads mit einem horizontalen Griff zur Größenänderung angeordnet, statt sich mit ihm eine Zeile zu teilen. Bei Viewports in Telefongröße wird das Detailfenster weiterhin im Vollbildmodus geöffnet.
    - Die Auswahlfelder für Modell und Thinking im Chat-Header aktualisieren die aktive Sitzung sofort über `sessions.patch`; sie sind dauerhafte Sitzungsüberschreibungen und keine nur für einen einzelnen Sendevorgang geltenden Optionen.
    - **Geteilte Ansicht:** Öffnen Sie sie über die Titelleiste des Chats (neben den Umschaltern für Thread-Diff, Hintergrundaufgaben und Thread-Dateien) und teilen Sie anschließend den aktiven Bereich nach rechts oder unten in so viele Bereiche, wie Platz finden. Jeder Bereich verfügt über einen eigenen Thread, ein eigenes Transkript, einen eigenen Composer und einen eigenen Tool-Stream.
    - Agenten mit dem Tool `screen` können dieselben Änderungen an Bereichen, Seitenleiste, Terminal, Browser, Fokus und Navigation anfordern, während eine geeignete Control UI verbunden ist. Protokoll v1 wendet den Befehl auf jede verbundene geeignete Control UI an; siehe [Bildschirm](/de/tools/screen).
    - Ziehen Sie eine Sitzung aus der Seitenleiste in den Chat, um sie in einem Bereich zu öffnen. Eine animierte Ablagevorschau gleitet zwischen den Zonen und kennzeichnet das Ergebnis — „Teilen“ über genau der Hälfte, die ein neuer Bereich einnehmen wird, „Hier öffnen“ über einem vollständigen Bereich — und das Ablegen funktioniert auch im Einzelbereichsmodus.
    - Der aktive geteilte Bereich steuert die Auswahl in der Seitenleiste und die URL. Seine Titelleiste enthält zusätzliche Steuerelemente zum Teilen und Schließen; Trennlinien ermöglichen die Größenänderung von Spalten und gestapelten Bereichen, und der Browser speichert das Layout lokal über Neuladevorgänge hinweg.
    - Auf schmalen Bildschirmen behält die geteilte Ansicht das Layout bei, stellt jedoch nur den aktiven Bereich dar, einschließlich seines Headers mit dem Steuerelement zum Schließen.
    - Wenn Sie eine Nachricht senden, während eine Änderung im Modell-Auswahlfeld für dieselbe Sitzung noch gespeichert wird, wartet der Composer auf diese Sitzungsaktualisierung, bevor `chat.send` aufgerufen wird, damit beim Senden das ausgewählte Modell verwendet wird.
    - Durch Eingabe von `/new` wird dieselbe neue Dashboard-Sitzung wie bei „Neuer Chat“ erstellt und auf sie gewechselt, außer wenn `session.dmScope: "main"` konfiguriert ist und die aktuelle übergeordnete Sitzung die Hauptsitzung des Agenten ist; in diesem Fall wird die Hauptsitzung direkt zurückgesetzt. Durch Eingabe von `/reset` wird weiterhin das explizite direkte Zurücksetzen des Gateways für die aktuelle Sitzung verwendet.
    - Das Modell-Auswahlfeld des Chats fordert die konfigurierte Modellansicht des Gateways an. Wenn `agents.defaults.modelPolicy.allow` nicht leer ist, bestimmt diese Richtlinie das Auswahlfeld, einschließlich der Einträge unter `provider/*`, durch die Provider-spezifische Kataloge dynamisch bleiben. Andernfalls zeigt das Auswahlfeld konfigurierte Einträge sowie Provider mit verwendbarer Authentifizierung an; Aliasse und Einstellungen unter `agents.defaults.models` schränken es nicht ein. Der vollständige Katalog bleibt über den Debug-RPC `models.list` mit `view: "all"` verfügbar.
    - Wenn aktuelle Berichte zur Sitzungsauslastung des Gateways die derzeitigen Kontext-Token enthalten, zeigt die Symbolleiste des Chat-Composers einen kleinen Ring zur Kontextauslastung mit dem verwendeten Prozentsatz an. Öffnen Sie den Ring, um das aktuelle Kontextfenster, die Token-Anzahlen des letzten Durchlaufs und die geschätzten Gesamtkosten, die Provider-/Modellidentität sowie die vom Provider gemeldete Aufschlüsselung der Eingabe-, Ausgabe- und Cache-Kosten der neuesten Antwort anzuzeigen. Bei hoher Kontextauslastung wechselt der Ring zu einem Warnstil und zeigt bei empfohlenen Compaction-Schwellenwerten eine kompakte Schaltfläche an, die den normalen Compaction-Pfad der Sitzung ausführt. Veraltete Token-Momentaufnahmen bleiben ausgeblendet, bis das Gateway erneut aktuelle Auslastungsdaten meldet.

  </Accordion>
  <Accordion title="Sprechmodus (Browser-Echtzeit)">
    Der Sprechmodus verwendet einen registrierten Echtzeit-Sprach-Provider. Konfigurieren Sie OpenAI mit `talk.realtime.provider: "openai"` sowie einem `openai`-API-Schlüsselprofil, `talk.realtime.providers.openai.apiKey` oder `OPENAI_API_KEY`. OpenAI Realtime verwendet die öffentliche Platform API und erfordert einen Platform-API-Schlüssel; eine Codex-OAuth-Anmeldung reicht für diese Schnittstelle nicht aus. Konfigurieren Sie Google mit `talk.realtime.provider: "google"` sowie `talk.realtime.providers.google.apiKey`. Der Browser erhält niemals einen regulären Provider-API-Schlüssel: OpenAI erhält ein kurzlebiges Realtime-Client-Secret für WebRTC, und Google Live erhält ein einmalig verwendbares, eingeschränktes Live-API-Authentifizierungstoken für eine Browser-WebSocket-Sitzung, wobei Anweisungen und Tool-Deklarationen durch das Gateway fest im Token verankert werden. Provider, die nur eine Echtzeit-Backend-Bridge bereitstellen, werden über den Gateway-Relay-Transport ausgeführt, sodass Anmeldedaten und Anbieter-Sockets serverseitig verbleiben, während Browser-Audio über authentifizierte Gateway-RPCs übertragen wird. Der Prompt der Realtime-Sitzung wird vom Gateway zusammengestellt; `talk.client.create` akzeptiert keine vom Aufrufer bereitgestellten Überschreibungen der Anweisungen.

    Dauerhafte Standardwerte für Provider, Modell, Stimme, Transport, Reasoning-Aufwand, exakten VAD-Schwellenwert, Stilledauer und Präfix-Padding befinden sich unter **Settings → Communications → Talk**; für Änderungen ist Zugriff auf `operator.admin` erforderlich. Die Konfiguration des Gateway-Relays erzwingt den Backend-Relay-Pfad; bei der Konfiguration von WebRTC verbleibt die Sitzung im Besitz des Clients und schlägt fehl, statt stillschweigend auf Relay zurückzufallen, wenn der Provider keine Browser-Sitzung erstellen kann.

    Das Sprechmodus-Steuerelement selbst ist die Mikrofonschaltfläche in der Composer-Symbolleiste. Das zugehörige Aufklappmenü listet **System default** und jedes vom Browser bereitgestellte Mikrofon auf, einschließlich USB-, Bluetooth- und virtueller Eingänge. Die ausgewählte Geräte-ID verbleibt lokal im Browser und wird niemals an das Gateway gesendet; wenn genau dieses Gerät nicht mehr verfügbar ist, fordert der Sprechmodus Sie auf, einen anderen Eingang auszuwählen, statt stillschweigend mit einem anderen Mikrofon aufzunehmen. Während der Sprechmodus aktiv ist, wird die Mikrofonschaltfläche zu einer pillenförmigen Anzeige mit der Live-Eingangspegelanzeige; ein Klick darauf beendet die Spracheingabe, und beim Darüberfahren wird das Stoppsymbol eingeblendet. Screenreader geben `Connecting voice input...`, `Listening...` oder `Asking OpenClaw...` aus, während ein Echtzeit-Tool-Aufruf über `talk.client.toolCall` das konfigurierte größere Modell konsultiert. Das Beenden einer laufenden Agentenantwort erfolgt weiterhin über ein separates quadratisches **Stop**-Steuerelement neben der pillenförmigen Anzeige.

    **Video-Sprechmodus** ist für OpenAI-Realtime-WebRTC- und Google-Live-Browsersitzungen verfügbar. Klicken Sie auf die Kameraschaltfläche, erlauben Sie den Kamera- und Mikrofonzugriff und bestätigen Sie die lokale Vorschau. OpenAI sendet einen begrenzten JPEG-Frame über seinen Browser-Datenkanal, wenn `describe_view` visuellen Kontext anfordert. Google Live sendet begrenzte JPEG-Frames direkt vom Browser an den Provider, mit dem unterstützten Höchstwert von einem Frame pro Sekunde, und beantwortet `describe_view`-Funktionsaufrufe mit dem Status des Kamerastreams. Kameraframes werden niemals durch das Gateway geleitet. Beim Beenden des Sprechmodus wird die Vorschau geschlossen, und beide Medientracks werden freigegeben. Informationen zu den Übertragungsverträgen des Providers finden Sie in Googles Dokumentation zu den [Funktionen der Live API](https://ai.google.dev/gemini-api/docs/live-api/capabilities#video) und im [Leitfaden für Funktionsaufrufe](https://ai.google.dev/gemini-api/docs/live-api/tools).

    Live-Smoke-Test für Maintainer: `OPENAI_API_KEY=... GEMINI_API_KEY=... node --import tsx scripts/dev/realtime-talk-live-smoke.ts` überprüft die OpenAI-Backend-WebSocket-Bridge, den OpenAI-Browser-WebRTC-SDP-Austausch, die Einrichtung des eingeschränkten Tokens für Google Live im Browser mit einem JPEG-Frame und einem `describe_view`-Funktions-Roundtrip sowie den Gateway-Relay-Browseradapter mit simulierten Mikrofonmedien. Der Befehl gibt nur den Provider-Status aus und protokolliert keine Secrets.

  </Accordion>
  <Accordion title="Stoppen und abbrechen">
    - Klicken Sie auf **Stop**. Ausführungen mit einer exakten lokalen Ausführungs-ID rufen `chat.abort` auf; wenn der Status der ausgewählten Sitzung aktive Arbeit meldet, die Control UI jedoch keine lokale Ausführungs-ID besitzt, wird stattdessen `sessions.abort` aufgerufen. Bei nicht globalen Sitzungen verwirft dieser Pfad der ausgewählten Sitzung außerdem in der Warteschlange befindliche Folgeaktionen, damit diese nach dem Stoppen keine Arbeit erneut starten können.
    - Während eine Ausführung aktiv ist, verwenden normale Folgeaktionen den effektiven `messages.queue`-Modus des Gateways. `steer` wird in den laufenden Turn eingeschleust; andere Modi behalten die dauerhafte Warteschlangenzustellung des Browsers bei. Wird das Steuern abgelehnt, erfolgt ebenfalls ein Rückfall auf diese Warteschlange. Klicken Sie bei einer Nachricht in der Warteschlange auf **Steer**, um sie manuell einzuschleusen.
    - **Settings → Appearance → Chat → Follow-ups while the agent is working** kann diesen Serverstandard für den aktuellen Browser überschreiben. Die Seite kennzeichnet eine Überschreibung ausdrücklich und bietet **Reset to server default** an. `Steer into the active run` sendet Folgeaktionen sofort, während `Queue until the run ends` sie zurückhält, bis die Ausführung abgeschlossen ist.
    - Geben Sie `/stop` (oder eigenständige Abbruchformulierungen wie `stop`, `stop action`, `stop run`, `stop openclaw`, `please stop`) ein, um außerhalb des regulären Ablaufs abzubrechen.
    - `chat.abort` unterstützt `{ sessionKey }` (ohne `runId`), um alle aktiven Ausführungen dieser Sitzung abzubrechen. Die Control UI verwendet `sessions.abort`, wenn keine lokale Ausführungs-ID vorhanden ist.

  </Accordion>
  <Accordion title="Beibehalten von Teilinhalten bei einem Abbruch">
    - Wenn eine Ausführung abgebrochen wird, kann unvollständiger Assistententext weiterhin in der Benutzeroberfläche angezeigt werden.
    - Das Gateway speichert bei einem Abbruch unvollständigen Assistententext im Transkriptverlauf, wenn gepufferte Ausgaben vorhanden sind.
    - Gespeicherte Einträge enthalten Abbruchmetadaten, damit Transkriptkonsumenten unvollständige Abbruchausgaben von regulär abgeschlossenen Ausgaben unterscheiden können.

  </Accordion>
</AccordionGroup>

## Verbindungsverlust und erneute Verbindung

Sobald eine Sitzung hergestellt ist, werden Sie durch eine unterbrochene Gateway-Verbindung nicht abgemeldet. Das Dashboard
bleibt sichtbar und zeigt unter der oberen Leiste eine schwebende bernsteinfarbene Anzeige „Gateway-Verbindung verloren — Verbindung wird wiederhergestellt…“,
während der Client automatisch mit Backoff erneut versucht, eine Verbindung herzustellen (800 ms bis zu 15 s). Live-Aktualisierungen und
Echtzeit-/Sitzungsaktionen werden pausiert, bis die Verbindung wiederhergestellt ist; **Retry now** in der Anzeige erzwingt einen
sofortigen Versuch. Der Chat bleibt bearbeitbar: Normale Text- und Anhangsendungen werden im
Gateway-/sitzungsbezogenen Browserspeicher des aktuellen Tabs aufbewahrt, als auf die erneute Verbindung wartend angezeigt und
automatisch gesendet, sobald das Gateway wieder verfügbar ist. Live-Steuerelemente und Slash-Befehle bleiben im
Offlinezustand nicht verfügbar; ausgenommen ist **Stop**, das eine exakte lokale Ausführungs-ID zur späteren Wiedergabe in die Warteschlange stellen kann. Ein nur auf die Sitzung bezogener Stopp
wird nicht erneut ausgeführt, da in dieser Sitzung möglicherweise neuere Arbeit beginnt, bevor die Verbindung wiederhergestellt ist.

Wenn dieser Browser bereits über Anmeldedaten verfügt (ein konfiguriertes Token/Passwort oder ein genehmigtes Geräte-
Token), wird beim ersten Öffnen und beim Neuladen eine kleine animierte OpenClaw-Markierung angezeigt, während die Verbindung
hergestellt wird, statt kurzzeitig den Anmeldedialog einzublenden. Der Anmeldedialog erscheint nur, wenn noch keine Anmeldedaten
gespeichert sind oder wenn das Gateway sie aktiv ablehnt (ungültiges Token/Passwort, widerrufene Kopplung) —
also in Zuständen, die Ihre Eingabe erfordern, statt lediglich abzuwarten.

## PWA-Installation und Web Push

Die Control UI enthält ein `manifest.webmanifest` und einen Service Worker, sodass moderne Browser sie als eigenständige PWA installieren können. Mit Web Push kann das Gateway die installierte PWA durch Benachrichtigungen aktivieren, selbst wenn der Tab oder das Browserfenster nicht geöffnet ist.

Innerhalb der macOS-App zeigt die Seite mit den Benachrichtigungseinstellungen die native Benachrichtigungsberechtigung der App statt Browser-Push an, da die App Benachrichtigungen nativ zustellt.

Wenn auf der Seite direkt nach einem OpenClaw-Update **Protocol mismatch** angezeigt wird, öffnen Sie zunächst das Dashboard erneut mit `openclaw dashboard` und führen Sie eine vollständige Aktualisierung durch. Wenn der Fehler weiterhin besteht, löschen Sie die Websitedaten für den Ursprung des Dashboards oder testen Sie in einem privaten Browserfenster; ein alter Tab oder der Service-Worker-Cache des Browsers kann weiterhin ein Control-UI-Bundle von vor dem Update mit dem neueren Gateway ausführen.

| Oberfläche                                         | Funktion                                                                     |
| -------------------------------------------------- | ---------------------------------------------------------------------------- |
| `ui/public/manifest.webmanifest`                   | PWA-Manifest. Browser bieten „Install app“ an, sobald es erreichbar ist.     |
| `ui/public/sw.js`                                  | Service Worker, der `push`-Ereignisse und Klicks auf Benachrichtigungen verarbeitet. |
| `state/openclaw.sqlite` → `web_push_vapid_keys`    | Automatisch generiertes VAPID-Schlüsselpaar zum Signieren von Web-Push-Nutzlasten. |
| `state/openclaw.sqlite` → `web_push_subscriptions` | Dauerhaft gespeicherte Browser-Abonnementendpunkte, Schlüssel und Registrierungszeitstempel. |

Upgrades aus den eingestellten Speichern `push/vapid-keys.json` und `push/web-push-subscriptions.json` werden durch `openclaw doctor --fix` importiert. Stoppen Sie das Gateway, bevor Sie diese Reparatur ausführen, damit ein älterer Prozess während des Imports keinen eingestellten Zustand neu erstellen kann. Führen Sie die Reparatur nach einem Upgrade aus, bevor Sie Web Push verwenden; Registrierung, Zustellung, Löschung und Schlüsselauflösung werden verweigert, solange entweder eine eingestellte Quelle oder ein unterbrochener Doctor-Claim vorhanden ist. Die Gateway-Laufzeit liest und schreibt ausschließlich SQLite.

Überschreiben Sie das VAPID-Schlüsselpaar über Umgebungsvariablen des Gateway-Prozesses, wenn Sie Schlüssel fest vorgeben möchten (Bereitstellungen mit mehreren Hosts, Rotation von Secrets oder Tests):

- `OPENCLAW_VAPID_PUBLIC_KEY`
- `OPENCLAW_VAPID_PRIVATE_KEY`
- `OPENCLAW_VAPID_SUBJECT` (standardmäßig `https://openclaw.ai`)

Die Control UI verwendet diese durch Berechtigungsbereiche eingeschränkten Gateway-Methoden, um Browser-Abonnements zu registrieren und zu testen:

- `push.web.vapidPublicKey` ruft den aktiven öffentlichen VAPID-Schlüssel ab.
- `push.web.subscribe` registriert ein `endpoint` zusammen mit `keys.p256dh`/`keys.auth`.
- `push.web.unsubscribe` entfernt einen registrierten Endpunkt.
- `push.web.test` sendet eine Testbenachrichtigung an das Abonnement des Aufrufers.

<Note>
Web Push ist unabhängig vom iOS-APNS-Relay-Pfad (Informationen zu Relay-gestütztem Push finden Sie unter [Konfiguration](/de/gateway/configuration)) und von der Methode `push.test`, die auf die native Kopplung mobiler Geräte ausgerichtet ist.
</Note>

## Gehostete Einbettungen

Assistentennachrichten können mit dem Shortcode `[embed ...]` gehostete Webinhalte inline darstellen. Die iframe-Sandbox-Richtlinie wird durch `gateway.controlUi.embedSandbox` gesteuert:

Das zentrale Tool [`show_widget`](/de/tools/show-widget) rendert eigenständiges SVG oder HTML direkt aus einem Tool-Aufruf. Der Browser und unterstützte native Chatclients melden die Gateway-Fähigkeit `inline-widgets`, und das resultierende Canvas-Dokument bleibt verfügbar, wenn der Chatverlauf neu geladen wird. Discord Activities stellt auf Discord denselben Tool-Namen bereit; Ausführungen, die aus anderen Kanälen stammen, erhalten ihn nicht.

<Tabs>
  <Tab title="streng">
    Deaktiviert die Skriptausführung innerhalb gehosteter Einbettungen.
  </Tab>
  <Tab title="Skripte (Standard)">
    Ermöglicht interaktive Einbettungen unter Beibehaltung der Ursprungsisolation; dies reicht in der Regel für eigenständige Browsergames/Widgets aus.
  </Tab>
  <Tab title="vertrauenswürdig">
    Ergänzt `allow-scripts` um `allow-same-origin` für Dokumente derselben Website, die bewusst stärkere Berechtigungen benötigen.
  </Tab>
</Tabs>

```json5
{
  gateway: {
    controlUi: {
      embedSandbox: "scripts",
    },
  },
}
```

<Warning>
Verwenden Sie `trusted` nur, wenn das eingebettete Dokument tatsächlich Verhalten desselben Ursprungs benötigt. Für die meisten von Agenten generierten Spiele und interaktiven Canvas-Inhalte ist `scripts` die sicherere Wahl.
</Warning>

Absolute externe `http(s)`-Einbettungs-URLs bleiben standardmäßig blockiert. Damit `[embed url="https://..."]` Seiten von Drittanbietern laden kann, legen Sie `gateway.controlUi.allowExternalEmbedUrls: true` fest.

## Layout des Chattranskripts

Das Chat-Transkript verwendet einen zentrierten, gut lesbaren Rahmen, der am Eingabebereich ausgerichtet ist. Ausgaben des Assistenten und von Tools bleiben linksbündig, während Ihre eigenen Nachrichten innerhalb dieses Rahmens rechtsbündig bleiben. In Sitzungen mit mehreren Benutzern (beispielsweise einem Gruppenchat, der von einem Kanal-Plugin weitergeleitet wird) werden Nachrichten anderer zugeordneter Teilnehmer linksbündig mit Avatar und Namen des Autors sowie einer stabilen Farbe pro Identität dargestellt, sodass nur die Nachrichten des angemeldeten Betrachters als „meine“ erscheinen. Wenn zwei oder mehr zugeordnete Teilnehmer anwesend sind, enthalten Antworten des Assistenten eine kleine Markierung „Antwort an Name“, die den Teilnehmer nennt, dessen Nachricht den Durchlauf ausgelöst hat. Systemeinträge wie die lokale Ausgabe von Slash-Befehlen werden als zentrierte Hinweiszeilen ohne Avatar dargestellt.

## Breite der Chatnachrichten

Benutzer mit breiten Monitoren können die Breite des Transkripts unter **Settings → Chat →
Message width** überschreiben. Die Einstellung bleibt im lokalen Speicher dieses Browsers erhalten. Unterstützte
Formen umfassen einfache Längen und Prozentwerte wie `960px` oder `82%` sowie
begrenzte Breitenausdrücke vom Typ `min(...)`, `max(...)`, `clamp(...)`, `calc(...)` und
`fit-content(...)`.

## Tailnet-Zugriff (empfohlen)

<Tabs>
  <Tab title="Integriertes Tailscale Serve (bevorzugt)">
    Lassen Sie das Gateway an die Loopback-Schnittstelle gebunden und verwenden Sie Tailscale Serve als HTTPS-Proxy:

    ```bash
    openclaw gateway --tailscale serve
    ```

    Öffnen Sie `https://<magicdns>/` (oder Ihre konfigurierte Adresse `gateway.controlUi.basePath`).

    Standardmäßig können sich Control-UI-/WebSocket-Serve-Anfragen über Tailscale-Identitätsheader (`tailscale-user-login`) authentifizieren, wenn `gateway.auth.allowTailscale` auf `true` gesetzt ist. OpenClaw überprüft die Identität, indem es die Adresse `x-forwarded-for` mit `tailscale whois` auflöst und mit dem Header abgleicht. Diese Anfragen werden nur akzeptiert, wenn sie die Loopback-Schnittstelle mit den `x-forwarded-*`-Headern von Tailscale erreichen. Bei Control-UI-Bediensitzungen mit Browsergeräteidentität überspringt dieser verifizierte Serve-Pfad außerdem den Durchlauf zur Gerätekopplung; Browser ohne Geräteidentität und Verbindungen mit Node-Rolle unterliegen weiterhin den normalen Geräteprüfungen. Setzen Sie `gateway.auth.allowTailscale: false`, wenn Sie selbst für Serve-Datenverkehr explizite Anmeldedaten in Form eines gemeinsamen Geheimnisses verlangen möchten, und verwenden Sie dann `gateway.auth.mode: "token"` oder `"password"`.

    Bei diesem asynchronen Serve-Identitätspfad werden fehlgeschlagene Authentifizierungsversuche für dieselbe Client-IP und denselben Authentifizierungsbereich vor Schreibvorgängen für die Ratenbegrenzung serialisiert. Bei gleichzeitig ausgeführten fehlerhaften Wiederholungsversuchen aus demselben Browser kann die zweite Anfrage daher `retry later` anzeigen, anstatt dass zwei einfache Nichtübereinstimmungen parallel miteinander konkurrieren.

    <Warning>
    Die tokenlose Serve-Authentifizierung setzt voraus, dass der Gateway-Host vertrauenswürdig ist. Wenn auf diesem Host nicht vertrauenswürdiger lokaler Code ausgeführt werden könnte, verlangen Sie eine Token-/Passwortauthentifizierung.
    </Warning>

  </Tab>
  <Tab title="An Tailnet binden + Token">
    ```bash
    openclaw gateway --bind tailnet --token "$(openssl rand -hex 32)"
    ```

    Öffnen Sie `http://<tailscale-ip>:18789/` (oder Ihre konfigurierte Adresse `gateway.controlUi.basePath`).

    Fügen Sie das entsprechende gemeinsame Geheimnis in die UI-Einstellungen ein (wird als `connect.params.auth.token` oder `connect.params.auth.password` gesendet).

  </Tab>
</Tabs>

## Unsicheres HTTP

Wenn Sie das Dashboard über einfaches HTTP (`http://<lan-ip>` oder `http://<tailscale-ip>`) öffnen, wird der Browser in einem **unsicheren Kontext** ausgeführt und blockiert WebCrypto. Standardmäßig **blockiert** OpenClaw Control-UI-Verbindungen ohne Geräteidentität.

Die unterstützte Ausnahme ohne Geräteidentität ist eine erfolgreiche Control-UI-Authentifizierung für Bediener
über `gateway.auth.mode: "trusted-proxy"`. Es gibt keinen dauerhaften Konfigurationsschalter,
der die Geräteidentität deaktiviert.

**Empfohlene Lösung:** Verwenden Sie HTTPS (Tailscale Serve) oder öffnen Sie die UI lokal unter `https://<magicdns>/` (Serve) beziehungsweise `http://127.0.0.1:18789/` (auf dem Gateway-Host).

<AccordionGroup>
  <Accordion title="Hinweis zum vertrauenswürdigen Proxy">
    - Eine erfolgreiche Authentifizierung über einen vertrauenswürdigen Proxy kann **Bediener**-Control-UI-Sitzungen ohne Geräteidentität zulassen.
    - Dies gilt **nicht** für Control-UI-Sitzungen mit Node-Rolle.
    - Loopback-Reverse-Proxys auf demselben Host erfüllen die Authentifizierung über einen vertrauenswürdigen Proxy weiterhin nicht; siehe [Authentifizierung über einen vertrauenswürdigen Proxy](/de/gateway/trusted-proxy-auth).

  </Accordion>
</AccordionGroup>

Anleitungen zur HTTPS-Einrichtung finden Sie unter [Tailscale](/de/gateway/tailscale).

## Inhaltssicherheitsrichtlinie

Die Control UI wird mit einer strikten `img-src`-Richtlinie ausgeliefert: Zulässig sind nur Ressourcen desselben Ursprungs, `data:`-URLs und lokal erzeugte `blob:`-URLs. Entfernte `http(s)`- und protokollrelative Bild-URLs werden vom Browser abgelehnt und lösen niemals Netzwerkanfragen aus.

In der Praxis:

- Avatare und Bilder, die unter relativen Pfaden bereitgestellt werden (beispielsweise `/avatars/<id>`), werden weiterhin dargestellt. Dies gilt auch für authentifizierte Avatar-Routen, die von der UI abgerufen und in lokale `blob:`-URLs umgewandelt werden.
- Eingebettete `data:image/...`-URLs werden weiterhin dargestellt.
- Von der Control UI erstellte lokale `blob:`-URLs werden weiterhin dargestellt.
- Avatare für GitHub-Linkvorschauen werden vom Gateway vom festgelegten Avatar-Host von GitHub abgerufen und als begrenzte `data:`-URLs zurückgegeben; der Browser des Bedieners kontaktiert den entfernten Avatar-Host niemals.
- Von Kanalmetadaten ausgegebene entfernte Avatar-URLs werden von den Avatar-Hilfsfunktionen der Control UI entfernt und durch das integrierte Logo/Abzeichen ersetzt. Dadurch kann ein kompromittierter oder bösartiger Kanal keine beliebigen entfernten Bildabrufe aus dem Browser eines Bedieners erzwingen.

Dies ist immer aktiviert und nicht konfigurierbar.

## Authentifizierung der Avatar-Route

Wenn die Gateway-Authentifizierung konfiguriert ist, erfordert der Avatar-Endpunkt der Control UI dasselbe Gateway-Token wie der Rest der API:

- `GET /avatar/<agentId>` gibt das Avatarbild nur an authentifizierte Aufrufer zurück. `GET /avatar/<agentId>?meta=1` gibt die Avatarmetadaten nach derselben Regel zurück.
- Nicht authentifizierte Anfragen an eine der beiden Routen werden abgelehnt (entsprechend der benachbarten Route für Assistentenmedien), sodass die Avatar-Route auf ansonsten geschützten Hosts keine Agentenidentität preisgeben kann.
- Die Control UI leitet beim Abrufen von Avataren das Gateway-Token als Bearer-Header weiter und verwendet authentifizierte Blob-URLs, damit das Bild weiterhin in Dashboards dargestellt wird.

Wenn Sie die Gateway-Authentifizierung deaktivieren (auf gemeinsam genutzten Hosts nicht empfohlen), wird entsprechend dem übrigen Gateway auch die Avatar-Route nicht authentifiziert.

## Authentifizierung der Route für Assistentenmedien

Wenn die Gateway-Authentifizierung konfiguriert ist, verwenden lokale Medienvorschauen des Assistenten eine zweistufige Route:

- `GET /__openclaw__/assistant-media?meta=1&source=<path>` erfordert die normale Control-UI-Bedienerauthentifizierung; der Browser sendet das Gateway-Token als Bearer-Header, wenn er die Verfügbarkeit prüft.
- Erfolgreiche Metadatenantworten enthalten ein kurzlebiges `mediaTicket`, das auf genau diesen Quellpfad beschränkt ist.
- Vom Browser dargestellte URLs für Bilder, Audio, Videos und Dokumente verwenden `mediaTicket=<ticket>` anstelle des aktiven Gateway-Tokens oder Passworts. Das Ticket läuft schnell ab und kann keine andere Quelle autorisieren.

Dadurch bleibt die Mediendarstellung mit browsernativen Medienelementen kompatibel, ohne wiederverwendbare Gateway-Anmeldedaten in sichtbaren Medien-URLs offenzulegen.

## Genehmigungslinks

Genehmigungsbenachrichtigungen für Bediener können direkt auf ein eigenständiges Genehmigungsdokument verlinken, das unter dem reservierten Namespace `${controlUiBasePath}/approve/{approvalId}` bereitgestellt wird (beispielsweise `/approve/<approvalId>` oder `/openclaw/approve/<approvalId>` bei einem konfigurierten Basispfad). Die URL bleibt während der gesamten Gültigkeitsdauer der Genehmigung stabil und kann sicher zwischen Ihren eigenen Geräten weitergeleitet werden: Sie identifiziert die Genehmigung, autorisiert sie jedoch niemals.

- Der aus einem Segment bestehende Namespace `/approve/<approvalId>` wird vom Gateway vor den HTTP-Routen von Plugins für **alle** HTTP-Methoden reserviert, sodass eine Plugin-Route ein Genehmigungsdokument niemals überlagern oder abfangen kann.
- Zum Öffnen eines Genehmigungsdokuments ist dieselbe Gateway-Authentifizierung wie für die übrige Control UI erforderlich (Token/Passwort, Tailscale-Serve-Identität oder Identität eines vertrauenswürdigen Proxys); Anmeldedaten sind niemals Bestandteil der Genehmigungs-URL.
- Wenn die Bereitstellung der Control UI deaktiviert ist, geben Anfragen an den Namespace `404` zurück, anstatt an Plugin-Handler weitergereicht zu werden.
- Die Anmeldung in einem Genehmigungsdokument ist nur für diese Seite temporär: Sie überschreibt nicht die Gateway-Auswahl oder die Einstellungen, die von der vollständigen Control UI im selben Browser gespeichert wurden.

Das Gateway stellt statische Dateien aus `dist/control-ui` bereit:

```bash
pnpm ui:build
```

Optionale absolute Basis (feste Ressourcen-URLs):

```bash
OPENCLAW_CONTROL_UI_BASE_PATH=/openclaw/ pnpm ui:build
```

Lokale Entwicklung (separater Entwicklungsserver):

```bash
pnpm ui:dev
```

Richten Sie die UI anschließend auf die WebSocket-URL Ihres Gateways (z. B. `ws://127.0.0.1:18789`).

## Leere Control-UI-Seite

Wenn der Browser ein leeres Dashboard lädt und die Entwicklertools keinen hilfreichen Fehler anzeigen, hat möglicherweise eine Erweiterung oder ein früh ausgeführtes Inhaltsskript die Auswertung der JavaScript-Modulanwendung verhindert. Die statische Seite enthält ein einfaches HTML-Wiederherstellungsfenster, das angezeigt wird, wenn `<openclaw-app>` nach dem Start nicht registriert ist.

Verwenden Sie nach einer Änderung der Browserumgebung die Aktion **Try again** des Fensters oder laden Sie die Seite nach den folgenden Prüfungen manuell neu:

- Deaktivieren Sie Erweiterungen, die Inhalte in alle Seiten einschleusen, insbesondere Erweiterungen mit `<all_urls>`-Inhaltsskripten.
- Versuchen Sie es mit einem privaten Fenster, einem sauberen Browserprofil oder einem anderen Browser.
- Lassen Sie das Gateway weiterlaufen und überprüfen Sie nach dem Browserwechsel dieselbe Dashboard-URL.

## Debugging/Tests: Entwicklungsserver + entferntes Gateway

Die Control UI besteht aus statischen Dateien; das WebSocket-Ziel ist konfigurierbar und kann sich vom HTTP-Ursprung unterscheiden. Das ist praktisch, wenn Sie den Vite-Entwicklungsserver lokal verwenden möchten, das Gateway jedoch an anderer Stelle ausgeführt wird.

<Steps>
  <Step title="UI-Entwicklungsserver starten">
    ```bash
    pnpm ui:dev
    ```
  </Step>
  <Step title="Mit gatewayUrl öffnen">
    ```text
    http://localhost:5173/?gatewayUrl=ws%3A%2F%2F<gateway-host>%3A18789
    ```

    Optionale einmalige Authentifizierung (falls erforderlich):

    ```text
    http://localhost:5173/?gatewayUrl=wss%3A%2F%2F<gateway-host>%3A18789#token=<gateway-token>
    ```

  </Step>
</Steps>

<AccordionGroup>
  <Accordion title="Hinweise">
    - `gatewayUrl` wird nach dem Laden in localStorage gespeichert und aus der URL entfernt.
    - Wenn Sie über `gatewayUrl` einen vollständigen `ws://`- oder `wss://`-Endpunkt übergeben, URL-kodieren Sie den Wert, damit der Browser die Abfragezeichenfolge korrekt verarbeitet.
    - `token` sollte nach Möglichkeit über das URL-Fragment (`#token=...`) übergeben werden. Fragmente werden nicht an den Server gesendet, wodurch eine Offenlegung über Anfrageprotokolle und den Referer vermieden wird. Veraltete `?token=`-Abfrageparameter werden aus Kompatibilitätsgründen weiterhin einmalig importiert, jedoch nur als Rückfalloption, und unmittelbar nach dem Bootstrap entfernt.
    - `password` wird nur im Arbeitsspeicher aufbewahrt.
    - Wenn `gatewayUrl` gesetzt ist, greift die UI nicht auf Anmeldedaten aus der Konfiguration oder Umgebung zurück. Geben Sie `token` (oder `password`) explizit an; fehlende explizite Anmeldedaten sind ein Fehler.
    - Verwenden Sie `wss://`, wenn sich das Gateway hinter TLS befindet (Tailscale Serve, HTTPS-Proxy usw.).
    - `gatewayUrl` wird nur in einem Fenster der obersten Ebene akzeptiert (nicht eingebettet), um Clickjacking zu verhindern.
    - Öffentliche Control-UI-Bereitstellungen außerhalb der Loopback-Schnittstelle müssen `gateway.controlUi.allowedOrigins` explizit festlegen (vollständige Ursprünge). Private gleichursprüngliche LAN-/Tailnet-Ladevorgänge von Loopback-, RFC1918-/Link-Local-, `.local`-, `.ts.net`- oder Tailscale-CGNAT-Hosts werden akzeptiert, ohne den Rückfall auf den Host-Header zu aktivieren.
    - Beim Start kann das Gateway lokale Ursprünge wie `http://localhost:<port>` und `http://127.0.0.1:<port>` aus der effektiven Laufzeitbindung und dem Port übernehmen; entfernte Browserursprünge benötigen jedoch weiterhin explizite Einträge.
    - Verwenden Sie `gateway.controlUi.allowedOrigins: ["*"]` ausschließlich für streng kontrollierte lokale Tests; dies bedeutet, dass jeder Browserursprung zugelassen wird, nicht „mit dem jeweils verwendeten Host übereinstimmen“.
    - `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback=true` aktiviert den Modus für den Rückfall auf den Ursprung des Host-Headers, stellt jedoch einen gefährlichen Sicherheitsmodus dar.

  </Accordion>
</AccordionGroup>

```json5
{
  gateway: {
    controlUi: {
      allowedOrigins: ["http://localhost:5173"],
    },
  },
}
```

Details zur Einrichtung des Remotezugriffs: [Remotezugriff](/de/gateway/remote).

## Verwandte Themen

- [Dashboard](/de/web/dashboard) — Gateway-Dashboard
- [Integritätsprüfungen](/de/gateway/health) — Überwachung des Gateway-Zustands
- [TUI](/de/web/tui) — Terminal-Benutzeroberfläche
- [WebChat](/de/web/webchat) — browserbasierte Chat-Oberfläche
