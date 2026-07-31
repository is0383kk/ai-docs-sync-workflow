---
read_when:
    - Implementierung oder Überprüfung der Sitzungs-Dashboard-Funktion (Boards)
    - Widget-Hosting, Widget-Bridge oder Board-Speicher ändern
summary: 'Sitzungs-Dashboards: Architektur- und Implementierungsplan (technisches Design, vor GA)'
title: Dashboard-Architektur
x-i18n:
    generated_at: "2026-07-26T18:51:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a7c5da94ec19add55c6b7b530f0c17509a027e97fb301469ce48f520b325c169
    source_path: web/dashboard-architecture.md
    workflow: 16
---

<Note>
Technisches Designdokument für die Sitzungs-Dashboard-Funktion, verfasst vor und
während der Implementierung. Es ist die maßgebliche Referenz für den Ausbau. Wenn die
Funktion ausgeliefert wird, wird `/web/dashboard` zur benutzerorientierten Seite, während diese Seite
als Architekturreferenz bestehen bleibt.
</Note>

## Vision

Die Arbeit mit einem Agenten ist heute ein Textstrom. Das Dashboard macht daraus eine
Werkbank: Der Agent rendert interaktive Live-Widgets; Benutzer heften sie an
eine persistente Oberfläche; der Chat wird seitlich angedockt (oder ausgeblendet), und der Hauptinhalt ist
das Board. So wird aus dem „Gespräch mit dem Agenten“ die „Bedienung eines Kontrollpanels, das
der Agent für Sie erstellt hat“, ohne dass Sie die Sitzung jemals verlassen.

Grundsätze:

- **Ein Board ist eine Ansicht einer Sitzung, kein neues Objekt.** Jede Sitzung (Thread)
  hat zwei Ansichten: das Transkript und das Board. Eine Sitzung ohne angeheftete Widgets
  ist ein einfacher Chat. Sobald ein Widget angeheftet wird, existiert das Board. Boards übernehmen
  Identität, Agentenzuordnung, Benennung, Anheftung und Lebenszyklus der
  Sitzung. Es gibt kein `dashboard_create`, keine Board-Registry und kein separates ACL-Modell.
- **Gleichwertigkeit des Agenten.** Alles, was Benutzer auf einem Board tun können, kann auch der Agent
  mit Tools tun: Widgets hinzufügen/aktualisieren/entfernen, sie anordnen, Tabs verwalten, den
  sichtbaren Tab wechseln und den Chat andocken oder ausblenden.
- **Nativ, nicht eingebettet.** Das Board besteht aus Lit-Komponenten in der Control-UI-Shell
  (demselben Designsystem wie der Rest der App). Nur der _Inhalt_ eines Widgets wird
  in iframes sandboxiert. Keine URL-Leiste, keine Browser-Bedienelemente.
- **Kleine Agentenoberfläche.** Widgets werden über stabile Namen adressiert und direkt
  aktualisiert. Das Layout ist ein flexibles, automatisch komprimierendes Raster; der Agent gibt Größen und
  Anker an, niemals Pixel oder Koordinaten.
- **Berechtigungen statt Vertrauen.** Widget-Code ist beliebiges, vom Agenten verfasstes HTML/JS
  in einer streng abgeschotteten Sandbox. Zugriff (Gateway-Daten, Aktionen, Netzwerk) besteht nur über
  ein deklariertes, vom Betreiber gewährtes Berechtigungsmanifest.

## Konzepte

| Konzept             | Definition                                                                                                                                                        |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Sitzung (Thread)    | Bestehende Gateway-Sitzung, identifiziert durch den stabilen `sessionKey`. Gehört einem Agenten.                                                                                        |
| Board               | Die Widget-Ansicht einer Sitzung. Existiert genau dann, wenn die Sitzung Widgets/Tabs hat. Überdauert `/new`/`/reset` (an `sessionKey` gebunden, nicht an das Transkript).                 |
| Tab                 | Eine Darstellungsseite eines Boards: welche Widgets, ihre Anordnung und der Zustand des Chat-Docks (`left`/`right`/`bottom`/`hidden`). Boards beginnen mit einem impliziten Tab. |
| Widget              | Benanntes, sandboxiertes HTML/JS-Programm, das der Sitzung gehört. Adressiert als `sessionKey` + `name`. Wird anhand des Namens direkt aktualisiert.                                              |
| Berechtigungsmanifest | Widget-spezifische Deklaration des Zugriffs: `data` (Lesebindungen), `actions` (Verben auf der Positivliste), `prompt` (an Sitzung senden), `net` (zulässige Ursprünge).                      |
| Anheften (Widget)        | Verschieben eines Transkript-Widgets auf das Board der Sitzung (Benutzerfunktion oder Argument eines Agenten-Tools). Durch Lösen wird es vom Board entfernt.                                         |
| Anheften (Sitzung)       | Bestehendes Anheften von Sitzungen in der Seitenleiste. Eine angeheftete Sitzung mit einem Board wird in ihrer Board-Ansicht geöffnet.                                                                      |

## UX-Abläufe

- **Übernahme:** Der Agent ruft `show_widget` in einem beliebigen Chat auf → das Widget wird wie bisher inline
  im Transkript gerendert → beim Darüberfahren erscheint **An Dashboard anheften** → das Widget
  erscheint auf dem Board der Sitzung. Der Agent kann `pin: true` übergeben, um dasselbe zu bewirken.
- **Board-Ansicht:** Eine Sitzung mit einem Board erhält einen Ansichtsumschalter (Chat / Dashboard).
  Board-Ansicht = Tableiste (nur bei >1 Tab) + flexibles Raster + angedockter Chatbereich.
  Das Chat-Dock ist in der Größe veränderbar, verschiebbar (links/rechts/unten) und genau wie
  die Seitenleiste einklappbar. Der Dock-Zustand wird pro Tab gespeichert.
- **Ziehen:** Benutzer ziehen Widgets; das Raster wird automatisch komprimiert (Widgets rücken nach oben,
  benachbarte Elemente ordnen sich neu an). Die Größenänderung über einen Griff rastet in Größenstufen ein. Keine pixelgenaue Platzierung –
  für niemanden.
- **Warnung beim Zurücksetzen:** `/new` / `/reset` fordert bei einer Sitzung mit Board
  in der Web-UI eine Bestätigung an („Der Kontext wird zurückgesetzt, das Dashboard bleibt erhalten“) und behält
  das Board bei.
- **Seitenleiste:** Angeheftete Sitzungen zeigen ihre Board-Ansicht, sofern eine vorhanden ist.
  Das Board der Home-Sitzung ist das standardmäßige „Agenten-Dashboard“.
- **Interaktionen** (drei Stufen, siehe unten): stille Zustandsereignisse, sichtbare
  Prompt-Sendungen und Automatisierungsauslöser.

## Interaktionsstufen

1. **Zustandsereignisse (Standard).** Interaktionen mit der Widget-Benutzeroberfläche, über die das Modell informiert sein sollte,
   auf die es jedoch nicht reagieren soll. `bridge.emitState({...})` fügt einen strukturierten
   Sitzungshinweis hinzu (derselbe Mechanismus wie bei Gruppenaktivitätshinweisen). Es wird kein Agentendurchlauf
   gestartet; das Modell sieht die gesammelten Hinweise bei seinem nächsten Durchlauf.
2. **Prompts (explizite Kommunikation).** `bridge.sendPrompt(text)` – erfordert eine
   Benutzeraktivierung; sendet eine sichtbare Benutzernachricht an die Sitzung (der angedockte Chat
   zeigt sie an). Ratenbegrenzt; jede Sendung muss vom Benutzer bestätigt werden, sofern das Widget nicht über
   die gewährte Berechtigung `prompt` verfügt.
3. **Automatisierung.** `bridge.runAction(name, args)` – löst eine im Manifest deklarierte
   Aktion aus. Anfänglicher Satz von Verben: `cron.trigger` (einen vorhandenen Cron-Job jetzt ausführen) und
   `binding.refresh`. Cron-Jobs werden bereits in sichtbaren, isolierten Ausführungssitzungen ausgeführt
   und können ein günstigeres Modell verwenden: Das ist der Weg „ein kleines Modell treibt das Widget an“.
   Es gibt nirgends versteckte Sitzungen.

## Widget-Modell und Hosting

Widget-HTML/JS wird vom Agenten verfasst (typischerweise über `show_widget`), in
die Standard-Dokument-Shell eingebettet (CSP-Meta, Größenmelder, Bridge-Bootstrap) und
in `<iframe sandbox="allow-scripts">` gerendert (niemals `allow-same-origin`).

- **Inline-Widgets (Transkript)** behalten die aktuelle Canvas-Dokument-Pipeline bei:
  im Zustandsverzeichnis gespeichert, vom Gateway bereitgestellt, nach Geltungsbereich bereinigt, keine
  Genehmigung (sie haben konstruktionsbedingt keine Berechtigungen – Prompt-Sendungen werden vom Benutzer bestätigt).
- **Board-Widgets** sind Sitzungszustand: Die Bytes liegen in der SQLite-Datenbank
  des zuständigen Agenten (`board_widgets`) und werden über eine zentrale Gateway-Route
  (`/__openclaw__/board/<agentId>/<sessionKey>/<name>/`) bereitgestellt, die die Datenbank liest.
  Beim Anheften eines Transkript-Widgets werden die Bytes kopiert. Begrenzungen: 256 KB pro Widget,
  48 Widgets pro Board.
- **Direkte Aktualisierung:** Das erneute Ausgeben eines Widgets mit demselben `name` ersetzt die
  Bytes, erhöht `revision`, sendet `board.changed` und veranlasst Live-Ansichten, nur
  dieses iframe neu zu laden.
- **Byte-Bindung:** Gewährte Berechtigungen werden an den sha256-Hash der Widget-
  Bytes gebunden. Beim Ändern der Bytes bleiben Gewährungen für `data`/`net`/`actions` nur erhalten, wenn die neue
  Revision eine Teilmenge des gewährten Manifests deklariert; ein erweitertes Manifest
  fordert den Betreiber erneut zur Bestätigung auf.

### Widgets hosten Inhalte; MCP-Apps sind eine Inhaltsart

Das **Widget ist das OpenClaw-Grundelement**: die benannte, angeheftete, dimensionierte,
sitzungseigene Board-Zelle mit einem Gewährungsdatensatz. Was darin gerendert wird, ist eine
Inhaltsart:

- `html` – vom Agenten über `show_widget` verfasst, Bytes im Board-Speicher.
- `mcp-app` – eine MCP-App-Ansicht eines Drittanbieters (`ui://`-Ressource eines konfigurierten
  Servers), die innerhalb der Widget-Zelle gehostet wird.

MCP-Apps definieren das Widget-Modell nicht; Widgets haben die Fähigkeit erhalten, sie zu
hosten. Identität, Platzierung, Anheftung, Gewährungen und die API für Autoren bleiben
OpenClaw-eigen – dadurch bleibt `show_widget`-Code so kurz wie heute und muss niemals
wissen, dass die MCP-Apps-Spezifikation existiert.

Gemeinsame zugrunde liegende Infrastruktur (hier findet die Vereinfachung statt):

- **Ein Sandbox-Host.** `html`-Widgets werden über dieselbe gehärtete
  Pipeline gerendert, mit der MCP-Apps ausgeliefert wurden (doppeltes iframe auf dem dedizierten Sandbox-
  Ursprung, pro Widget deklarierte CSP, die ausfallsicher dekodiert wird), statt über einen zweiten
  maßgeschneiderten iframe-Host. Der Proxy empfängt HTML als Wert, daher sind lokale Inhalte
  der natürliche Anwendungsfall.
- **Ein Autorisierungsmodell.** Der Zugriff eines Widgets ist eine gewährte Positivliste,
  unabhängig von seiner Art: für `html`-Widgets Host-Tools; für `mcp-app`-Widgets
  die für die App sichtbaren Tools des Servers (über den bestehenden `allowedAppToolNames`-
  Mechanismus, dauerhaft pro Widget statt pro Erzeugungsdurchlauf).
- **Host-Tools für `html`-Widgets** (über die Widget-Bridge verfügbar gemacht und
  anhand der Gewährung geprüft):
  - `openclaw.prompt.send` – Stufe 2; über den sichtbaren Composer geleitet,
    vom Benutzer bestätigt, sofern nicht gewährt
  - `openclaw.state.emit` – Sitzungshinweise der Stufe 1 (zusammengeführt, größenbegrenzt)
  - `openclaw.data.read` – parametrisierte schreibgeschützte Bindungen (bestehender
    Satz von Lese-RPCs auf der Positivliste), Gateway-seitig aufgelöst
  - `openclaw.cron.trigger` – Automatisierung der Stufe 3
- **`net` = CSP.** Der Netzwerkzugriff verwendet die bereits ausgelieferte, Widget-spezifische CSP-
  Deklaration (`connect-src`-Ursprünge) – das selbstaktualisierende Wetter-Widget
  ruft seine API direkt aus der Sandbox ab, ohne Beteiligung des Gateways.
- **Gewährungen.** Ein Widget, das nichts deklariert, wird sofort gerendert (sandboxiert,
  `default-src 'none'`, Prompt-Sendungen werden einzeln bestätigt) – dasselbe Vertrauensniveau wie
  bei heutigen Inline-Chat-Widgets. Deklarierte Tools/Ursprünge versetzen das Widget auf dem Board in
  `pending`: Eine Platzhalterkarte listet sie in verständlicher Form auf, mit
  **Zulassen**/**Ablehnen** per einfachem Tippen. Gewährungen gelten pro Widget-Namen; bei `html`-Widgets
  sind sie an die Bytes gebunden (sha256), und bei geänderten Bytes bleibt die Gewährung nur erhalten, wenn die
  Deklaration verkleinert wurde.
- **Autoren-Shim.** Der Dokument-Wrapper injiziert `window.openclaw.prompt`,
  `window.openclaw.state`, `window.openclaw.data` und `window.openclaw.cron`
  als stabile Autoren-API. Dashboard-Aufrufe teilen sich einen einzigen, an das Ansichtsticket gebundenen
  Anfragekanal; Größenmeldungen und Theme-Tokens bleiben separate Host-
  Benachrichtigungen.

### Plugin-Berechtigungsdeklarationen

Aktivierte Plugins können den Widget-Host über `dashboard.dataBindings`
und `dashboard.actionVerbs` in `openclaw.plugin.json` erweitern. Plugin-lokale IDs werden
zu Gewährungsnamen mit dem Präfix der Plugin-ID, beispielsweise `workboard.cards.list` und
`workboard.dispatch`; `%` und `.` werden im Plugin-ID-Segment maskiert, damit eine
andere Aufteilung von Plugin und lokaler ID nicht dieselbe persistierte Gewährung übernehmen kann. Während
der Plugin-Registrierung überprüft OpenClaw, dass jede Bindung auf einen RPC verweist,
der vom selben Plugin mit `operator.read` registriert wurde, und jede Aktion auf einen,
der mit `operator.write` registriert wurde; ungültige Deklarationen führen zum Fehlschlagen des Plugin-Ladevorgangs. Die validierte
Registry wird nur bei Änderungen am Plugin-Lebenszyklus neu aufgebaut, während Widget-Gewährungen
Widget-spezifisch sowie an Bytes und Revision gebunden bleiben.

### Modelliertes Restrisiko: WebRTC-Datenkanäle

Die Sandbox-CSP gibt die vorgeschlagene `webrtc 'block'`-Direktive aus, aber
[Chromiums aktueller Satz von CSP-Direktiven](https://chromium.googlesource.com/chromium/src/+/main/services/network/public/mojom/content_security_policy.mojom#95)
implementiert sie nicht. Skriptfähige Widgets können daher im aktuellen Chromium WebRTC-Datenkanäle
für ausgehende Verbindungen verwenden. Dasselbe Restrisiko besteht bereits bei
Inline-Chat-Widgets und dem MCP-Apps-Host auf `main`.

**Akzeptierter Kompromiss:** OpenClaw sperrt skriptfähige Widgets nicht aufgrund dieses
Restrisikos. Widget-Inhalte erhalten Zugriff auf sensible OpenClaw-Daten nur über
eine vom Betreiber gewährte, bytegenau fixierte `data:read`-Berechtigung, und die
Permissions Policy der Sandbox blockiert den Zugriff auf Kamera und Mikrofon. Eine
DOM-API-Schutzvorkehrung ist eine Best-Effort-Tiefenverteidigung, keine Sicherheitsgrenze,
und gehört in eine nachgelagerte Härtung.

### Transkriptanzeige: eine Widget-Karte

Die Inline-Anzeige wird auf dem Widget-Primitiv vereinheitlicht. Wenn ein Tool-Ergebnis eine UI enthält —
`show_widget`-Ausgabe oder ein MCP-Tool-Ergebnis mit einer App-Ressource — materialisiert das System
ein **flüchtiges, automatisch benanntes Widget** (sitzungsbezogen, bereinigt), und
das Transkript rendert eine einzelne Widget-Karte, die nach Inhaltsart weiterleitet.
Die automatische MCP-App-Anzeige bleibt exakt so, wie es die Spezifikation erwartet (keine zusätzliche Modellarbeit);
darunter _ist_ sie lediglich ein Widget. Dadurch entfallen die parallelen `mcpApp`-
Sonderfälle beim Chat-Rendering (Oberflächenfreigabe, separate Deduplizierung), jede
Inline-UI erhält dieselbe Anheftoption, und die Widget-Registry wird zum primären
Pfad für das erneute Öffnen (die Rekonstruktion durch Durchsuchen des Transkripts bleibt als Rückfalloption für nie angeheftete
Verläufe bestehen). Der schreibgeschützte, ticketgebundene eigenständige Host überschneidet sich mit Boards als
dauerhafte Oberfläche zum erneuten Öffnen — ein in T6 zu prüfender Konsolidierungskandidat, keine
Annahme.

Komposition: v1 verwendet Raster-Nachbarschaft (Agent-Chrome-Widget neben einem App-Widget auf
einem Tab). v2 ergänzt **hostverwaltete App-Slots** — das HTML des Agent-Widgets deklariert eine
Slot-Region, und der Host setzt die tatsächliche App-Ansicht als gleichrangige Sandbox zusammen.
Die App wird niemals innerhalb des iFrames des Agenten gerendert: Eine Verschachtelung würde die Bridge-
Identität aufbrechen und Overlays beziehungsweise Clickjacking der freigegebenen App-UI ermöglichen, daher ist der Slot ein
Layout-Vertrag und keine Einbettung.

### Serverseitig bereitgestellte Widgets (angeheftete MCP-Apps)

Mit dem vereinheitlichten Host ist das Anheften einer MCP-App eines Drittanbieters lediglich ein Widget, dessen
Inhalt vom Server abgerufen statt gespeichert wird: `board_widgets` enthält den
Deskriptor (`serverName`, `toolName`, `uiResourceUri`, ursprüngliche
`toolCallId` + `sessionKey`) anstelle von HTML-Bytes, und das Board stellt die
Ansichts-Lease über die 10-minütige TTL des Chat-Durchlaufs hinaus neu aus (bei Veraltung wird die `ui://`-Ressource
erneut abgerufen). Inline-MCP-App-Ansichten im Chat erhalten dieselbe **An Dashboard anheften**-
Option wie Agent-Widgets. Erneut geöffnete Ansichten sind derzeit bewusst schreibgeschützt;
angeheftete Apps, die interaktiv bleiben sollen, erhalten eine dauerhafte Freigabe für die
app-sichtbaren Tools des Servers (explizite Positivliste, die dem Betreiber beim Anheften angezeigt wird), entkoppelt
vom ausstellenden Lauf. Nicht freigegebene angeheftete Ansichten bleiben schreibgeschützt — und sind weiterhin für Anzeige-
Dashboards nützlich. v1 heftet an das Board der ursprünglichen Sitzung an; sitzungsübergreifendes Anheften
benötigt einen Lease-Broker und muss warten. Abstimmung mit dem offenen PR #109807 (`ui/message`-
Composer-Routing, Weitergabe von Theme und Größe).

### WorkBoard-Integration

Das WorkBoard-Integrationsprogramm belässt Karten und Boards im Besitz des Plugins und verknüpft versandte Karten über die bestehenden `sessionKey` und `runId` wieder mit ihren Sitzungs-Boards, stellt WorkBoard-Feeds und Versand über vom Plugin deklarierte Bindings und Aktionen bereit und kombiniert diese Ergebnisse mit den bestehenden Widget-Arten `html` und `mcp-app`, statt einen WorkBoard-spezifischen Widget-Typ einzuführen.

## Layout: flexibles Raster

12 Spalten, feste Zeilenhöhe, **automatische Verdichtung** (Schwerkraft nach oben, beim
Ziehen zur Seite schieben — Gridstack-Semantik, nativ implementiert; die Rasterberechnung bleibt rein und
DOM-frei). Widget-Layoutstatus pro Tab: `{ name, w (1-12), h (rows) }` plus
Reihenfolge. Agent-Vokabular:

- `size`: `sm` (3×3) · `md` (6×4) · `lg` (8×6) · `xl` (12×8) · `full`
  (Tab mit einem einzelnen Widget)
- `after: <widgetName>` optionaler Sortieranker; weggelassen = anhängen
- Benutzer können frei ziehen und die Größe ändern; dasselbe Reihenfolge-und-Größe-Modell lässt sich verlustfrei hin- und zurückübertragen.

## Datenmodell (agentenspezifische DB)

Neue Tabellen in `agents/<agentId>/agent/openclaw-agent.sqlite`
(**erfordert eine Erhöhung der Schemaversion der Agent-DB — Zustimmung des Betreibers erforderlich,
bevor dies integriert wird**):

```sql
CREATE TABLE board_tabs (
  session_key TEXT NOT NULL,
  tab_id      TEXT NOT NULL,           -- Slug
  title       TEXT NOT NULL,
  position    INTEGER NOT NULL,
  chat_dock   TEXT NOT NULL DEFAULT 'right',  -- left|right|bottom|hidden
  created_by  TEXT NOT NULL,           -- 'user' | 'agent'
  PRIMARY KEY (session_key, tab_id)
) STRICT;

CREATE TABLE board_widgets (
  session_key  TEXT NOT NULL,
  name         TEXT NOT NULL,          -- stabiler Widget-Name
  tab_id       TEXT NOT NULL,
  title        TEXT,
  html         BLOB NOT NULL,          -- Quelle des umschlossenen Dokuments
  sha256       TEXT NOT NULL,
  revision     INTEGER NOT NULL,
  size_w       INTEGER NOT NULL,
  size_h       INTEGER NOT NULL,
  position     INTEGER NOT NULL,       -- Reihenfolge innerhalb des Tabs (Eingabe für automatische Verdichtung)
  manifest     TEXT NOT NULL DEFAULT '{}',  -- Berechtigungsmanifest als JSON
  grant_state  TEXT NOT NULL DEFAULT 'none', -- none|pending|granted|rejected
  granted_sha  TEXT,                   -- bytegenau fixierte Freigabe
  created_by   TEXT NOT NULL,
  created_at   INTEGER NOT NULL,
  updated_at   INTEGER NOT NULL,
  PRIMARY KEY (session_key, name)
) STRICT;
```

Ein Board existiert, sobald Zeilen für den `sessionKey` vorhanden sind. Beim Löschen einer Sitzung werden ihre
Board-Zeilen gelöscht. `/new`/`/reset` berührt sie nicht.

## Protokolloberfläche

RPCs (zentrale Methodentabelle, TypeBox-Schemas in `gateway-protocol`):

- `board.get { sessionKey }` → Tabs + Widget-Metadaten (keine Bytes) — `operator.read`
- `board.update { sessionKey, ops[] }` — CRUD/Neusortierung von Tabs, Verschieben/Größenänderung/
  Entfernen/Lösen von Widgets, Dock-Status, Tab fokussieren — `operator.write`
- `board.widget.put { sessionKey, name, html, manifest, placement }` —
  `operator.write` (Agent-Tool-Pfad und Anheftpfad)
- `board.widget.grant { sessionKey, name, decision }` — `operator.approvals`
- `board.event { ticket, payload }` — ticketgebundene Aufnahme von Statusereignissen der Stufe 1;
  die bisherige Form `{ sessionKey, widget, payload }` für vertrauenswürdige Hosts bleibt bestehen —
  `operator.write`
- `board.prompt.authorize { ticket }` — gibt zurück, ob das Senden einer sichtbaren Eingabeaufforderung
  weiterhin eine Bestätigung bei jedem Klick benötigt — `operator.read`
- `board.data.read { ticket, bindingId, params? }` — Gateway-seitige Auflösung von
  Core-Lesebindings oder Lesebindings aktiver Plugins anhand einer Positivliste — `operator.read`
- `board.action { ticket, action, ... }` — Automatisierungsversand mit exakter Freigabe
  über den bestehenden Cron-Sofortausführungspfad oder ein validiertes Aktionsverb
  eines aktiven Plugins — `operator.write`

Ereignisse (in `EVENT_SCOPE_GUARDS`, Lesebereich):

- `board.changed { sessionKey, revision, widget? }` — persistierter Status wurde geändert;
  die UI ruft erneut ab (und lädt einen iFrame neu, wenn `widget` vorhanden ist).
- `board.command { sessionKey, command }` — vorübergehende UI-Steuerung (Agent wechselt
  den sichtbaren Tab, schaltet das Chat-Dock um) — das `ui.command`-Muster.

Widget-Bytes werden über die authentifizierte HTTP-Oberfläche bereitgestellt, nicht über den Socket.

## Agent-Tools

Insgesamt drei Tools (Core, immer registriert; Rendering wie bisher durch die
Client-Berechtigung `inline-widgets` beschränkt):

- `show_widget { title, widget_code, name?, pin?, size?, tab?, after?,
capabilities? }` — nach Namen erstellen/aktualisieren; `pin` platziert es auf dem Board.
  Ohne `name`/`pin` verhält es sich exakt wie bisher (inline, flüchtig).
- `dashboard { action, ... }` — Board-Verwaltungsverben: `read`, `tab_create`,
  `tab_update`, `tab_delete`, `tabs_reorder`, `widget_move`, `widget_remove`,
  `unpin`, `focus_tab`, `set_chat_dock`.
- Bestehende `cron`-Tools decken die Automatisierungsstufe ab; kein neues Tool erforderlich.

Tool-Beschreibungen vermitteln das Größen-/Anker-Vokabular und das Stufenmodell. Der
Agent wird über Sitzungsbenachrichtigungen über Benutzerereignisse der Stufe 1 informiert, z. B.
`[dashboard] user clicked "Refresh" on widget weather (tab main)`.

## Was dadurch ersetzt wird

- **`extensions/workspaces` wird gelöscht.** Experimentell, `enabledByDefault:
false`, nie in einer stabilen Version enthalten (erstmals in den Betas von 2026.7.2 erschienen). Keine
  Migration; eine Doctor-Regel entfernt veraltete `<stateDir>/workspaces/`, falls vorhanden.
  Übernommene Ideen: reine Rasterberechnung, Bridge-Sicherheitsmodell (Port-Bootstrap,
  Binding-Beschränkung, Ratenbegrenzungen), bytegenau fixierte Genehmigung.
- **Das Widget-Hosting wird von `extensions/canvas` in den Core verschoben.** Der Canvas-Dokument-
  speicher, der Dokument-Wrapper, die HTTP-Bereitstellung und das Tool `show_widget` werden Teil des Cores
  (`src/canvas/`); das Plugin behält das Node-Canvas-Steuerungstool (`canvas`) und
  A2UI. Die `pluginSurfaceUrls["canvas"]`-Ankündigung und die
  `/__openclaw__/canvas`-Pfade sind ausgelieferte Verträge für native Clients und bleiben
  stabil. Discord-Sitzungen behalten die Discord-eigene Variante `show_widget`.

## Nichtziele (dieses Programm)

- Gemeinsame Nutzung von Boards durch mehrere Benutzer/ACLs (zukünftig; wird über die Sitzungsfreigabe eingeführt).
- Natives Board-Rendering unter macOS/iOS (dort ist es überall verfügbar, wo die
  Control UI eingebettet wird; der Inline-Widget-Pfad bleibt unverändert).
- Integrierte Daten-Widgets (Sitzungs-/Nutzungs-/Cron-Karten) — die Berechtigungs-Bridge und
  von Agenten erstellte Widgets decken v1 ab; eine Registry integrierter Arten kann später hinzukommen.

## Implementierungsplan

Unabhängige Worktrees, mit Codex erstellt, nacheinander prüfen und integrieren. Erst integrieren, dann korrigieren.

| #   | Branch                               | Umfang                                                                                                                                                                              | Abhängig von                       |
| --- | ------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------- |
| T1  | `claude/dashboard-remove-workspaces` | Workspaces-Plugin + UI + Dokumentation + i18n-Schlüssel löschen; Doctor-Bereinigungsregel                                                                                                              | —                                |
| T2  | `claude/dashboard-canvas-core`       | Widget-Hosting + `show_widget` in den Core überführen; Canvas-Plugin behält Node-Tool; keine Verhaltensänderung                                                                                | —                                |
| T3  | `claude/dashboard-domain`            | Agent-DB-Tabellen (Schemaerhöhung), `board.*`-RPCs + Ereignisse, Tool `dashboard`, `show_widget`-Argumente für Anheften/Name/Manifest, Stufe-1-Benachrichtigungen, Zurücksetzen behält Board bei                                  | T2                               |
| T4  | `claude/dashboard-ui`                | Board-Ansicht + Tableiste + flexibles, automatisch verdichtendes Raster + Chat-Dock (links/rechts/unten/ausgeblendet) + Anheftoption im Transkript + Board-Ansicht in der Seitenleiste + Bestätigung beim Zurücksetzen                           | T3 (zuerst Mocks über Entwicklungs-Fixtures) |
| T5  | `claude/dashboard-capabilities`      | Freigabespeicher/UI + bytegenaue Fixierung; `html`-Widgets auf den gemeinsamen Sandbox-Host verschieben; Host-Tools (`openclaw.prompt.send/state.emit/data.read/cron.trigger`); `net`-CSP; Erstellungskompatibilitätsschicht | T3, T4                           |
| T7  | `claude/dashboard-mcp-apps`          | Inhaltsart `mcp-app`: Anheftoption in Inline-App-Ansichten, Deskriptorspeicherung, erneute Lease-Ausstellung/Aktualisierung, dauerhafte Server-Tool-Freigaben (verwendet den ausgelieferten MCP-Apps-Host erneut)                   | T3, T4                           |
| T6  | Feinschliff                               | Live-E2E auf einem Test-Gateway (echte Schlüssel), Screenshots, Korrekturen, benutzerorientierte Überarbeitung von `/web/dashboard`, Prüfung der standardmäßigen Aktivierung                                                     | alle                              |

Validierung gemäß Repository-Regeln: fokussiertes Vitest lokal, vollständige Prüfungen auf
Crabbox/Testbox, `$autoreview` vor jeder Integration, Live-Nachweis für T6.
