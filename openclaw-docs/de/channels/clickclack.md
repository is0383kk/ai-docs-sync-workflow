---
read_when:
    - OpenClaw mit einem ClickClack-Workspace verbinden
    - ClickClack-Bot-Identitäten testen
summary: Einrichtung des ClickClack-Kanals mit Bot-Token und Zielsyntax
title: ClickClack
x-i18n:
    generated_at: "2026-07-26T17:38:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 761538cdd7a916415719131b9ff2f40bf3e3e0eab0f7bda450250886acde8a64
    source_path: channels/clickclack.md
    workflow: 16
---

ClickClack verbindet OpenClaw über vollwertige ClickClack-Bot-Tokens mit einem selbst gehosteten ClickClack-Workspace.

Verwenden Sie dies, wenn ein OpenClaw-Agent als ClickClack-Bot-Benutzer auftreten soll. ClickClack unterstützt unabhängige Dienst-Bots und benutzereigene Bots; benutzereigene Bots behalten eine `owner_user_id` und erhalten nur die von Ihnen gewährten Token-Berechtigungsbereiche.

## Schnelleinrichtung

Öffnen Sie in ClickClack **Workspace settings → Integrations → OpenClaw**, erstellen Sie
mit **Setup code (recommended)** einen Bot und kopieren Sie den generierten Befehl:

```bash
openclaw channels add clickclack --code 'https://clickclack.example.com/#XXXX-XXXX-XXXX'
```

Bei getrennten Frontend- und API-Ursprüngen oder einer unter einem Pfad eingebundenen API gibt ClickClack
stattdessen einen exakten Anforderungsendpunkt aus:

```bash
openclaw channels add clickclack --code 'https://api.example.com/services/clickclack/api/bot-setup-codes/claim#XXXX-XXXX-XXXX'
```

Der Einrichtungscode kann nur einmal verwendet werden und läuft nach 10 Minuten ab. OpenClaw löst ihn ein,
empfängt das neu ausgestellte Bot-Token und die Workspace-Einstellungen, speichert das Konto,
überprüft die Verbindung und meldet, ob das laufende Gateway sie übernommen hat.
Bei versionierten exakten Endpunkten validiert und speichert OpenClaw die von ClickClack
zurückgegebene kanonische API-Basis einschließlich eines etwaigen Pfadpräfixes. Der Einrichtungscode selbst wird
nicht in der OpenClaw-Konfiguration gespeichert.

Anforderungen mit Einrichtungscode verwenden für öffentliche Server HTTPS. Unverschlüsseltes HTTP wird auch für
lokale Installationen unter Loopback-Adressen wie `localhost` und `127.0.0.1` unterstützt.

Wenn OpenClaw bereits ausgeführt wird, stellt ClickClack automatisch eine Verbindung her und es ist kein zweiter
Befehl erforderlich. Starten Sie es andernfalls mit:

```bash
openclaw gateway
```

Sie können den Code auch getrennt von der Server-URL übergeben:

```bash
openclaw channels add clickclack --code XXXX-XXXX-XXXX --base-url https://clickclack.example.com
```

Führen Sie für die geführte Einrichtung Folgendes aus:

```bash
openclaw onboard
```

Wählen Sie ClickClack aus und geben Sie bei Aufforderung die Server-URL, das Bot-Token und den Workspace ein.
Die geführte Einrichtung überprüft nach dem Speichern den Server, das Token und den Workspace; eine
fehlgeschlagene Überprüfung verwirft die Konfiguration nicht.

### Alternative: Manuelles Token

Wählen Sie beim Konfigurieren eines Clients, der nicht OpenClaw verwendet, oder wenn Sie das Token
ausdrücklich selbst verwalten müssen, in ClickClack **Manual token** aus:

```bash
openclaw channels add clickclack --base-url https://clickclack.example.com --token ccb_... --workspace default
```

`workspace` akzeptiert eine Workspace-ID (`wsp_...`), einen Slug oder einen Anzeigenamen.
`--code` kann nicht mit `--token`, `--token-file` oder `--use-env` kombiniert werden.

### Alternative: Umgebungsbasiertes Token

Das Standardkonto kann `CLICKCLACK_BOT_TOKEN` lesen, statt ein Token
in der Konfiguration zu speichern:

```bash
export CLICKCLACK_BOT_TOKEN="ccb_..."
openclaw channels add clickclack --base-url https://clickclack.example.com --workspace default --use-env
openclaw gateway
```

Benannte Konten müssen ein konfiguriertes Token oder eine Token-Datei verwenden; die gemeinsam genutzte
Umgebungsvariable ist absichtlich auf das Standardkonto beschränkt.

### JSON5-Referenz

Die entsprechende Konfigurationsstruktur lautet:

```json5
{
  channels: {
    clickclack: {
      enabled: true,
      baseUrl: "https://clickclack.example.com",
      token: { source: "env", provider: "default", id: "CLICKCLACK_BOT_TOKEN" },
      workspace: "default",
      defaultTo: "channel:general",
    },
  },
}
```

Ein Konto gilt nur dann als konfiguriert, wenn `baseUrl`, eine Token-Quelle und
`workspace` festgelegt sind. Eine Token-Quelle kann für das Standardkonto `token`, `tokenFile` oder
`CLICKCLACK_BOT_TOKEN` sein. `workspace` akzeptiert eine Workspace-
ID (`wsp_...`), einen Slug oder einen Namen; das Gateway löst diesen beim Start in die ID auf.

### Konfigurationsschlüssel für Konten

| Schlüssel                | Standardwert        | Hinweise                                                                                |
| ----------------------- | ------------------- | --------------------------------------------------------------------------------------- |
| `baseUrl`               | keiner (erforderlich) | Öffentliche ClickClack-URL für Links, die im Browser geöffnet werden.                    |
| `apiBaseUrl`            | `baseUrl`           | Optionaler Server-zu-Server-Endpunkt für REST- und Echtzeit-WebSocket-Datenverkehr.      |
| `token`                 | keiner              | Bot-Token als Klartextzeichenfolge oder Geheimnisreferenz (`source: "env" \| "file" \| "exec"`).          |
| `tokenFile`             | keiner              | Pfad zu einer Bot-Token-Datei; hat Vorrang vor `token`.                       |
| `workspace`             | keiner (erforderlich) | Workspace-ID, Slug oder Name.                                                            |
| `replyMode`             | `"agent"`           | `"agent"` führt die vollständige Agent-Pipeline aus; `"model"` sendet kurze direkte Modellvervollständigungen. |
| `defaultTo`             | `"channel:general"` | Ziel, das verwendet wird, wenn ein ausgehender Pfad kein Ziel angibt.                    |
| `allowFrom`             | `["*"]`             | Zulassungsliste mit Benutzer-IDs für eingehende Direktnachrichten und Kanalnachrichten.  |
| `botUserId`             | automatisch erkannt | Wird beim Start anhand der Bot-Token-Identität aufgelöst.                                |
| `agentId`               | Routing-Standardwert | Ordnet die eingehenden Nachrichten dieses Kontos fest einem Agenten zu.                  |
| `toolsAllow`            | keiner              | Werkzeug-Zulassungsliste für Agentenantworten dieses Kontos.                             |
| `model`, `systemPrompt` | keiner              | Wird von `replyMode: "model"`-Vervollständigungen verwendet.                               |
| `commandMenu`           | `true`              | Veröffentlicht native Befehle für die automatische Vervollständigung im ClickClack-Editor. |
| `reconnectMs`           | `1500`              | Verzögerung für die erneute Echtzeitverbindung (100 bis 60000).                          |
| `discussions`           | deaktiviert         | Verwaltete Kanaleinstellungen pro Sitzung; siehe [Sitzungsdiskussionen](#session-discussions). |

### Einen öffentlich authentifizierungsgeschützten Hostnamen beibehalten

Verwenden Sie `apiBaseUrl`, wenn ClickClack und das OpenClaw-Gateway auf demselben Host ausgeführt werden,
der öffentliche ClickClack-Hostname jedoch durch ein Authentifizierungs-Gateway
wie Cloudflare Access geschützt ist:

```json5
{
  channels: {
    clickclack: {
      baseUrl: "https://clack.openclaw.ai",
      apiBaseUrl: "http://127.0.0.1:8484",
      token: { source: "env", provider: "default", id: "CLICKCLACK_BOT_TOKEN" },
      workspace: "default",
    },
  },
}
```

Der öffentliche Hostname kann für Browserbenutzer vollständig authentifizierungsgeschützt bleiben. OpenClaw
verwendet den Loopback-Endpunkt für REST-Anfragen, die Einrichtungsüberprüfung und den
Echtzeit-WebSocket, während die Diskussionslinks `embedUrl` und `openUrl` weiterhin die
öffentliche `baseUrl` verwenden. Wenn `apiBaseUrl` weggelassen wird, verwendet der gesamte Datenverkehr
`baseUrl`, wodurch das bestehende Verhalten beibehalten wird.

Wenn `plugins.allow` eine nicht leere restriktive Liste ist, wird durch die explizite Auswahl von
ClickClack bei der Kanaleinrichtung oder die Ausführung von `openclaw plugins enable clickclack`
`clickclack` an diese Liste angehängt. Die Installation während des Onboardings verwendet dasselbe
Verhalten bei expliziter Auswahl. Diese Pfade überschreiben weder `plugins.deny` noch eine
globale `plugins.enabled: false`-Einstellung. Die direkte Ausführung von
`openclaw plugins install @openclaw/clickclack` folgt der üblichen
Plugin-Installationsrichtlinie und trägt ClickClack ebenfalls in eine vorhandene Zulassungsliste ein.

## Mehrere Bots

Jedes Konto öffnet eine eigene ClickClack-Echtzeitverbindung und verwendet sein eigenes Bot-Token.

```json5
{
  channels: {
    clickclack: {
      enabled: true,
      baseUrl: "https://clickclack.example.com",
      defaultAccount: "service",
      accounts: {
        service: {
          token: { source: "env", provider: "default", id: "CLICKCLACK_SERVICE_BOT_TOKEN" },
          workspace: "default",
          defaultTo: "channel:general",
          agentId: "service-bot",
        },
        support: {
          token: { source: "env", provider: "default", id: "CLICKCLACK_SUPPORT_BOT_TOKEN" },
          workspace: "default",
          defaultTo: "dm:usr_...",
          agentId: "support-bot",
        },
      },
    },
  },
}
```

## Sitzungsdiskussionen

Aktivieren Sie Diskussionen für ein ClickClack-Konto, damit jede OpenClaw-Sitzung einen
eigenen ClickClack-Kanal erhält. Das Konto-Token muss
`channels:write` enthalten (das `bot:admin`-Paket enthält es); das normale `bot:write`-
Einrichtungstoken kann keine Kanäle erstellen oder synchronisieren.

```json5
{
  channels: {
    clickclack: {
      enabled: true,
      baseUrl: "https://clickclack.example.com",
      token: { source: "env", provider: "default", id: "CLICKCLACK_BOT_TOKEN" },
      workspace: "default",
      discussions: {
        enabled: true,
        workspace: "default",
        controlUrlBase: "https://team.openclaw.ai",
        section: "Sessions",
      },
    },
  },
}
```

`discussions.workspace` akzeptiert dieselbe Workspace-ID, denselben Slug oder Anzeigenamen
wie `workspace` auf Kontoebene und verwendet standardmäßig diesen Wert. `section` steuert
den Abschnitt in der ClickClack-Seitenleiste und verwendet standardmäßig `Sessions`. Wenn
`controlUrlBase` festgelegt ist, verweist der verwaltete Kanal zurück auf die tatsächliche Sitzungsroute
der Control UI, `/chat?session=<encoded-session-key>`.

Aktivieren Sie Diskussionen für genau ein ClickClack-Konto. Der Gateway-Provider besitzt
keine Kontoauswahl, daher werden mehrere Konten mit aktivierten Diskussionen abgelehnt,
statt eines anhand der Konfigurationsreihenfolge auszuwählen.

Beim Öffnen einer Diskussion wird ein öffentlicher ClickClack-Kanal erstellt, der als extern
verwaltet gekennzeichnet ist. Das Plugin hält Sitzungsbezeichnung, Kategorie und Archivierungsstatus
synchron. Beim Wiederherstellen einer Sitzung wird ihr Kanal wiederhergestellt; durch das Leeren der Sitzungskategorie
wird der Kanal wieder in den konfigurierten Standardabschnitt verschoben. Beim Löschen einer
OpenClaw-Sitzung wird der ClickClack-Kanal archiviert statt gelöscht, sodass sein
Verlauf verfügbar bleibt. Das Plugin gleicht Bindungen ab, wenn Diskussions-RPCs
verwendet werden, und ungefähr einmal pro Minute, solange Bindungen vorhanden sind.

Eingehende Nachrichten in einem verwalteten Kanal verwenden eine deterministische Nebensitzung unter
derselben Agenten-ID wie die verknüpfte Hauptsitzung. Dem Nebenagenten wird mitgeteilt, welche
Hauptsitzung er beobachten soll, und er kann `sessions_history` und `session_status` verwenden
(`changesSince` ist für inkrementelle Überprüfungen nützlich). Er verwendet `sessions_send` nur,
wenn Personen in der Diskussion ihn bitten, Informationen an die Hauptsitzung weiterzuleiten oder sie zu steuern.
Die Bindung, die Referenz auf den verwalteten Eigentümer und die Peer-Identität der Nebensitzung enthalten
die konkrete OpenClaw-Sitzungs-ID sowie den fest zugeordneten ClickClack-Server und
-Kanal. Durch das Zurücksetzen eines wiederverwendbaren Sitzungsschlüssels oder das Neuausrichten eines Kontos wird der
alte Kanal lokal widerrufen, archiviert, wenn die alten Anmeldedaten weiterhin verwendbar sind, und
sein Nebenprotokoll kann nicht wiederverwendet werden. Nachrichten, die über eine
archivierte, zurückgesetzte, deaktivierte oder neu ausgerichtete Bindung eingehen, werden verworfen, statt
auf das normale Kanal-Routing des Kontos zurückzufallen. Freigegebene Bindungen hinterlassen eine dauerhafte
Markierung für widerrufene Kanäle, sodass verzögerte Echtzeitereignisse weiterhin nach dem Fail-Closed-Prinzip behandelt werden. Die entfernte
Eigentümerschaft wird anhand des ClickClack-Servers und der Kanal-ID bestimmt, sodass das Umbenennen des lokalen
Kontos einen verwalteten Kanal nicht in einen gewöhnlichen Kanal umwandeln kann.

Belassen Sie `tools.sessions.visibility` beim sichereren Standardwert `tree`. Das Plugin
installiert eine hostbezogene Berechtigung ausschließlich zwischen jeder Nebensitzung und ihrer verknüpften
Hauptsitzung sowie einen Werkzeugrichtlinien-Hook, der Sitzungserkennung und
sitzungsübergreifende Ziele blockiert. Es erlaubt `sessions_history`, `session_status` und
`sessions_send` nur für die verknüpfte Hauptsitzung und verhindert, dass der Statusaufruf
das Modell dieser Sitzung ändert. Diese Werkzeuge müssen weiterhin in der
effektiven Werkzeug-Zulassungsliste des Agenten vorhanden sein. Der System-Prompt dient als Anleitung; die hostbezogene Berechtigung
und der Hook bilden die Autorisierungsgrenze.

Der ClickClack-Server muss bei der Erstellung und
Aktualisierung von Kanälen die Felder für verwaltete Kanäle (`external_managed`,
`external_ref`, `external_url` und `sidebar_section`) unterstützen und
sie in Kanalantworten zurückgeben. OpenClaw überprüft diesen Vertrag, bevor eine
Bindung dauerhaft gespeichert wird. Geht eine Erstellungsantwort verloren,
übernimmt der nächste Öffnungsvorgang den Kanal anhand seines serverseitig
erzwungenen `external_ref`, anstatt einen weiteren zu erstellen. Bis dieses
Ergebnis abgeglichen ist, stellt die ausstehende Reservierung ansonsten
ungebundene Ereignisse im Ziel-Workspace unter Quarantäne. Der grobe
Abgleich übernimmt den Kanal, wenn dieselbe Sitzung noch aktiv ist, oder
archiviert ihn nach einem Zurücksetzen; er löscht die Reservierung, wenn kein
Remote-Kanal erstellt wurde. Diese Referenz enthält einen dauerhaften Namespace
pro OpenClaw-Installation sowie einen Hash des Sitzungsschlüssels, die konkrete
Sitzungs-ID, das ClickClack-Ziel und die dauerhafte Bindungsgeneration. Separate
Gateways können die Kanäle der jeweils anderen nicht übernehmen,
zurückgesetzte Sitzungen können keinen alten Kanalverlauf erben und ein
Roundtrip über ein Konto oder einen Workspace kann einen vorherigen Kanal nicht
erneut übernehmen. Bindungen sind außerdem an die konfigurierte
ClickClack-Server-URL gebunden und werden ungültig, wenn das Konto auf ein
anderes Ziel umgestellt wird. Das Ändern oder Entfernen von
`controlUrlBase` aktualisiert oder löscht die Verknüpfung des verwalteten
Kanals beim nächsten Abgleichdurchlauf. Beim Ändern von
`discussions.workspace` wird die alte Bindung archiviert und freigegeben, bevor ein
Kanal im neuen Workspace geöffnet werden kann, sofern die Anmeldedaten des
alten Workspace weiterhin konfiguriert sind. Wurde das Token durch auf einen
Workspace beschränkte Anmeldedaten ersetzt, die nicht auf den alten Workspace
zugreifen können, vermerkt OpenClaw den alten Kanal als widerrufen und gibt die
Bindung frei, ohne das Ersatz-Token zu verwenden; archivieren Sie diesen
verbliebenen Kanal in ClickClack.

Die angehängte Hauptsitzung erhält außerdem ein ausschließlich lesendes
`discussion`-Tool. Es liest die neuesten Nachrichten und kürzlich
eingegangenen Thread-Antworten als jeweils einen maskierten Datensatz mit
Urheberangabe pro Nachricht und hat keine Nebenwirkungen auf Schreibvorgänge
oder den Lebenszyklus. Abfragen von Kanalwurzeln und Threads haben feste
Anfragebudgets; das Ergebnis warnt ausdrücklich, wenn durch diese
Sicherheitsgrenze ein älterer aktiver Thread ausgelassen werden kann.

## Antwortmodi

- `replyMode: "agent"` (Standard) leitet eingehende Nachrichten durch die normale Agent-Pipeline, einschließlich Sitzungsaufzeichnung und Tool-Richtlinie.
- `replyMode: "model"` überspringt die Agent-Pipeline und verwendet `llm.complete` der Plugin-Laufzeit für direkte Bot-Antworten, die optional durch `model` und `systemPrompt` gestaltet werden. Der ausgewählte Provider und das Modell bestimmen das Completion-Budget.

Im Modellmodus werden Completions für die aufgelöste Bot-Agent-ID ausgeführt.
Dafür ist das ausdrückliche `plugins.entries.clickclack.llm.allowAgentIdOverride: true`-Vertrauensbit
erforderlich:

```json5
{
  plugins: {
    entries: {
      clickclack: {
        llm: {
          allowAgentIdOverride: true,
        },
      },
    },
  },
}
```

Lassen Sie das Vertrauensbit deaktiviert, wenn Sie nur den standardmäßigen
Antwortmodus `agent` verwenden; dort wird es nicht benötigt.

## Befehlsmenü

Beim Start des Gateway veröffentlicht jedes konfigurierte Konto die nativen
Befehle von OpenClaw in ClickClack. Sie erscheinen in der automatischen
Vervollständigung des Eingabefelds und sind mit dem Handle des Bots
gekennzeichnet. Die veröffentlichte Menge wird bei jedem Start vollständig
ersetzt; dies schließt das Löschen eines veralteten Menüs ein, wenn der Katalog
nativer Befehle leer ist.

Die Synchronisierung des Befehlsmenüs ist standardmäßig aktiviert. Legen Sie
für ein Konto `commandMenu: false` fest, um sie zu deaktivieren:

```json5
{
  channels: {
    clickclack: {
      enabled: true,
      token: { source: "env", provider: "default", id: "CLICKCLACK_BOT_TOKEN" },
      workspace: "default",
      commandMenu: false,
    },
  },
}
```

Das Token benötigt `commands:write`. Die aktuellen ClickClack-Pakete
`bot:write` und `bot:admin` enthalten diesen
Berechtigungsumfang; er kann auch einzeln gewährt werden. Bei Tokens, die vor
der Einführung von Befehlsmenüs erstellt wurden, muss der Berechtigungsumfang
möglicherweise hinzugefügt oder das Token ersetzt werden.

Die Synchronisierung erfolgt nach bestem Bemühen einmal pro Gateway-Start. Ein
fehlender Berechtigungsumfang oder ein Netzwerkfehler wird als Warnung
protokolliert; bei einem älteren ClickClack-Server ohne den Endpunkt erfolgt die
Protokollierung auf Debug-Ebene. Keiner dieser Fehler blockiert den
Echtzeitstart. Menüs bleiben verfügbar, während der Agent offline ist, und
werden entfernt, wenn der Bot den Workspace verlässt.

Diese Version veröffentlicht ausschließlich Spezifikationen nativer Befehle.
Aliase sowie Kataloge für Skills, Plugins oder benutzerdefinierte Befehle werden
dem Menü nicht hinzugefügt. Wenn ein Name außerdem als HTTP-Slash-Befehl
registriert ist, verarbeitet ClickClack zuerst diese Registrierung; andere
Menübefehle werden weiterhin über die normale Nachrichtenzustellung
übermittelt.

Verwenden Sie den Modus `agent` als Nachweis für eine
dienstübergreifende Korrelation. Für eine maßgebliche ClickClack-Nachrichten-ID
in ihrer kanonischen Form `msg_<ulid>` leitet der Kanal die
deterministische OpenClaw-Ausführungs-ID `clickclack:<message-id>` ab. Jeder
Modellaufruf ist dann in der Diagnose als `clickclack:<message-id>:model:<n>` sichtbar; wenn
dieser Turn ClawRouter verwendet, wird dieselbe Modellaufruf-ID als
`X-Request-ID` gesendet. Der Modus `model` umgeht die normale
Diagnose der Agent-Ausführung und Sitzung und eignet sich daher nicht für
diesen Nachweispfad.

Wenn ein Echtzeitereignis einen validierten `payload.correlation_id` enthält,
überträgt der Kanal ihn als `X-Correlation-ID` bei der maßgeblichen
Nachrichtenabfrage und den daraus resultierenden ClickClack-Antwortanfragen.
Die Werte verwenden ClickClacks sicheren Zeichensatz mit 128 Zeichen
(`A-Z`, `a-z`, `0-9`,
`.`, `_`, `:` und
`-`); ungültige Werte werden ausgelassen. Diese Verknüpfungen
enthalten ausschließlich Bezeichner, niemals Nachrichtentexte, Prompts,
Completions, Anmeldedaten oder Tool-Ausgaben.

## Dauerhafte Medienzustellung

Agent-Antworten mit Medien verwenden die erforderliche dauerhafte Zustellung.
OpenClaw weist vor dem ersten ClickClack-Schreibvorgang stabile Nachrichten-
und Upload-Nonces pro Teil zu, sodass ein erneuter Versuch denselben Upload und
dieselbe Nachricht verwendet, anstatt Speicherkontingent zu verbrauchen oder
Duplikate zu veröffentlichen. Wenn nach einem Neustart bereits ein Upload
vorhanden ist, liest OpenClaw den ursprünglichen lokalen Pfad oder die
Remote-Medien-URL nicht erneut.

Dieser Wiederherstellungsvertrag erfordert einen ClickClack-Server, der
Folgendes unterstützt:

- `GET /api/uploads/by-nonce` mit
  `X-ClickClack-Upload-Nonce: supported` bei gefundenen und fehlenden Ergebnissen.
- `GET /api/messages/by-nonce` mit
  `X-ClickClack-Message-Nonce: supported` bei gefundenen und fehlenden Ergebnissen.
- Idempotente Nachrichtenerstellung und Zuordnung von Anhängen für dieselbe
  eigentümerbezogene Nonce und denselben Upload.

Ein generischer 404-Fehler eines älteren Servers gilt nicht als Nachweis dafür,
dass ein Sendevorgang nicht vorhanden ist. OpenClaw lässt die Zustellung
ungeklärt, anstatt ein Duplikat zu riskieren; aktualisieren Sie ClickClack,
bevor Sie Agent-Antworten aktivieren, die Medien erzeugen.

## Agent-Aktivitätszeilen

Standardmäßig zeigt ein ClickClack-Kanal während eines laufenden Agent-Turns nichts an; nur die endgültige Antwort wird eingestellt. Legen Sie für ein Konto `agentActivity: true` fest, um während des laufenden Turns dauerhafte Nachrichtenzeilen vom Typ `agent_commentary` und `agent_tool` zu veröffentlichen:

```json5
{
  channels: {
    clickclack: {
      enabled: true,
      token: { source: "env", provider: "default", id: "CLICKCLACK_BOT_TOKEN" },
      workspace: "default",
      agentActivity: true,
    },
  },
}
```

Anforderungen und Verhalten:

- **Standardmäßig deaktiviert.** Standardkonfigurationen und ältere ClickClack-Server bleiben unverändert.
- **Erfordert den Token-Berechtigungsumfang `agent_activity:write`.** Dieser Berechtigungsumfang ist von `bot:write` getrennt und wird nicht davon übernommen; erstellen Sie das Bot-Token mit `--scopes bot:write,agent_activity:write` (oder gewähren Sie einem vorhandenen Token den Berechtigungsumfang), bevor Sie die Option aktivieren.
- **Beeinträchtigung nach bestem Bemühen.** Wenn dem Token `agent_activity:write` fehlt oder der Server Aktivitätsschreibvorgänge ablehnt, werden die Fehler protokolliert und die endgültige Antwort dennoch normal zugestellt; es erscheinen keine Aktivitätszeilen.
- Zeilen werden pro Turn gruppiert (`turn_id`) und so zusammengeführt, dass ein logischer Schritt einer Zeile entspricht. Tool-Zeilen verwenden dieselbe Fortschrittsformatierung wie Discord/Slack/Telegram (Tool-Name plus Befehlsdetails).
- **Attributionsmetadaten.** Vom Agent verfasste Beiträge (Aktivitätszeilen und die endgültige Antwort) enthalten die Felder `author_model` und `author_thinking`, die anhand des tatsächlich für den Turn verwendeten Modells aufgelöst werden (auch nach einem Fallback). Server, die diese Spalten nicht definieren, ignorieren die unbekannten JSON-Felder; Server, die sie dauerhaft speichern, können pro Nachricht beantworten, „welches Modell diese Zeile auf welcher Denkstufe ausgegeben hat“.

## Ziele

- `channel:<name-or-id>` sendet an einen Workspace-Kanal. Ziele ohne Präfix verwenden standardmäßig `channel:`.
- `dm:<user_id>` erstellt eine direkte Unterhaltung mit diesem Benutzer oder verwendet eine vorhandene.
- `thread:<message_id>` antwortet in dem Thread, dessen Wurzel diese Nachricht ist.

Explizite ausgehende Ziele können außerdem das Provider-Präfix
`clickclack:` oder `cc:` enthalten.

Ausgehende Medien verwenden die Upload-API von ClickClack und hängen den
dauerhaften Upload anschließend an die erstellte Kanalnachricht, Thread-Antwort
oder Direktnachricht an. Lokale Dateien und unterstützte Remote-Medien-URLs
unterliegen der normalen Medienzugriffsrichtlinie von OpenClaw mit einem Limit
von 64 MiB pro Datei. Dauerhafte Sendevorgänge in der Warteschlange verwenden
separate eigentümerbezogene Nonces für jeden Upload und Nachrichtenteil und
wiederholen anschließend die Zuordnung von Anhängen mit denselben Objekten.
Informationen zum Serververtrag und Wiederherstellungsverhalten finden Sie
unter [Dauerhafte Medienzustellung](#durable-media-delivery).

Beispiele:

```bash
openclaw message send --channel clickclack --target channel:general --message "hello"
openclaw message send --channel clickclack --target dm:usr_123 --message "hello"
openclaw message send --channel clickclack --target thread:msg_123 --message "following up"
```

## Berechtigungen

Die Berechtigungsumfänge von ClickClack-Tokens werden von der ClickClack-API
durchgesetzt.

- `bot:read`: Workspace-, Kanal-, Nachrichten-, Thread-, Direktnachrichten-, Echtzeit- und Profildaten lesen.
- `bot:write`: `bot:read` sowie Kanalnachrichten, Thread-Antworten, Direktnachrichten, Uploads und die Veröffentlichung des Befehlsmenüs.
- `bot:admin`: `bot:write` sowie die Kanalerstellung.
- `commands:write`: das Befehlsmenü des Bots veröffentlichen. In den aktuellen Paketen `bot:write` und `bot:admin` enthalten und einzeln gewährbar.
- `agent_activity:write`: dauerhafte Agent-Aktivitätszeilen (`agent_commentary` / `agent_tool`). Wird nicht von `bot:write` oder `bot:admin` übernommen; nur erforderlich, wenn `agentActivity: true` festgelegt ist.

OpenClaw benötigt für den normalen Agent-Chat und die Synchronisierung des
Befehlsmenüs nur das aktuelle `bot:write`. Fügen Sie
`agent_activity:write` hinzu, wenn Sie
[Agent-Aktivitätszeilen](#agent-activity-rows) aktivieren.

## Fehlerbehebung

- `ClickClack is not configured for account "<id>"`: Legen Sie für dieses Konto `baseUrl`, `token` (beispielsweise über `CLICKCLACK_BOT_TOKEN`) und `workspace` fest.
- `ClickClack workspace not found: <value>`: Legen Sie `workspace` auf die von ClickClack zurückgegebene Workspace-ID, den Slug oder den Namen fest.
- Keine eingehenden Antworten: Vergewissern Sie sich, dass das Token über Echtzeit-Lesezugriff verfügt, und beachten Sie, dass der Bot seine eigenen Nachrichten und Nachrichten anderer Bots ignoriert.
- Kanal-Sendevorgänge schlagen fehl: Stellen Sie sicher, dass der Bot Mitglied des Workspace ist und über `bot:write` verfügt.
- Kein Befehlsmenü: Vergewissern Sie sich, dass `commandMenu` nicht `false` ist, der ClickClack-Server `PUT /api/bots/self/commands` unterstützt und das Token über `commands:write` verfügt.
