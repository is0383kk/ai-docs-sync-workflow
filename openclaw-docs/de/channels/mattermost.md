---
read_when:
    - Mattermost einrichten
    - Mattermost-Routing debuggen
sidebarTitle: Mattermost
summary: Mattermost-Bot-Einrichtung und OpenClaw-Konfiguration
title: Mattermost
x-i18n:
    generated_at: "2026-07-26T18:19:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ea41fb9a7e4e9ea6bd8d04a4f2c6d2d7f2e43cf71830e445f1e28e2e8737f3cb
    source_path: channels/mattermost.md
    workflow: 16
---

Status: herunterladbares Plugin (Bot-Token + WebSocket-Ereignisse). Kanäle, private Kanäle, Gruppen-DMs und DMs werden unterstützt. Mattermost ist eine selbst hostbare Team-Messaging-Plattform ([mattermost.com](https://mattermost.com)).

## Installation

<Tabs>
  <Tab title="npm-Registry">
    ```bash
    openclaw plugins install @openclaw/mattermost
    ```
  </Tab>
  <Tab title="Lokaler Checkout">
    ```bash
    openclaw plugins install ./path/to/local/mattermost-plugin
    ```
  </Tab>
</Tabs>

Details: [Plugins](/de/tools/plugin)

## Schnelleinrichtung

<Steps>
  <Step title="Verfügbarkeit des Plugins sicherstellen">
    Installieren Sie `@openclaw/mattermost` mit dem obigen Befehl und starten Sie anschließend den Gateway neu, falls er bereits ausgeführt wird.
  </Step>
  <Step title="Mattermost-Bot erstellen">
    Erstellen Sie ein Mattermost-Bot-Konto, kopieren Sie das **Bot-Token** und fügen Sie den Bot den Teams und Kanälen hinzu, die er lesen soll.
  </Step>
  <Step title="Basis-URL kopieren">
    Kopieren Sie die Mattermost-**Basis-URL** (z. B. `https://chat.example.com`). Ein nachgestelltes `/api/v4` wird automatisch entfernt.
  </Step>
  <Step title="OpenClaw konfigurieren und Gateway starten">
    Minimalkonfiguration:

    ```json5
    {
      channels: {
        mattermost: {
          enabled: true,
          botToken: "mm-token",
          baseUrl: "https://chat.example.com",
          dmPolicy: "pairing",
        },
      },
    }
    ```

    Nicht interaktive Alternative:

    ```bash
    openclaw channels add --channel mattermost --bot-token <token> --http-url https://chat.example.com
    ```

  </Step>
</Steps>

<Note>
Selbst gehostetes Mattermost unter einer privaten/LAN-/Tailnet-Adresse: Ausgehende Mattermost-API-Anfragen durchlaufen einen SSRF-Schutz, der private und interne IP-Adressen standardmäßig blockiert. Aktivieren Sie dies mit `channels.mattermost.network.dangerouslyAllowPrivateNetwork: true` (pro Konto: `channels.mattermost.accounts.<id>.network.dangerouslyAllowPrivateNetwork`).
</Note>

## Native Slash-Befehle

Native Slash-Befehle müssen explizit aktiviert werden. Wenn sie aktiviert sind, registriert OpenClaw `oc_*` Slash-Befehle in jedem Team, dem der Bot angehört, und empfängt Callback-POST-Anfragen auf dem HTTP-Server des Gateways.

```json5
{
  channels: {
    mattermost: {
      commands: {
        native: true,
        nativeSkills: true,
        callbackPath: "/api/channels/mattermost/command",
        // Verwenden, wenn Mattermost den Gateway nicht direkt erreichen kann (Reverse-Proxy/öffentliche URL).
        callbackUrl: "https://gateway.example.com/api/channels/mattermost/command",
      },
    },
  },
}
```

Registrierte Befehle: `/oc_status`, `/oc_model`, `/oc_models`, `/oc_new`, `/oc_help`, `/oc_think`, `/oc_reasoning`, `/oc_verbose`, `/oc_queue`. Mit `nativeSkills: true` werden Skill-Befehle ebenfalls als `/oc_<skill>` registriert.

<AccordionGroup>
  <Accordion title="Hinweise zum Verhalten">
    - `native` und `nativeSkills` verwenden standardmäßig `"auto"`, was für Mattermost als deaktiviert aufgelöst wird. Setzen Sie sie ausdrücklich auf `true`.
    - `callbackPath` verwendet standardmäßig `/api/channels/mattermost/command`.
    - Wenn `callbackUrl` weggelassen wird, leitet OpenClaw `http://<gateway.customBindHost or localhost>:<gateway.port, default 18789><callbackPath>` ab. Bei Platzhalter-Bind-Hosts (`0.0.0.0`, `::`) wird auf `localhost` zurückgegriffen.
    - Bei Konfigurationen mit mehreren Konten kann `commands` auf der obersten Ebene oder unter `channels.mattermost.accounts.<id>.commands` festgelegt werden (Kontowerte überschreiben Felder der obersten Ebene).
    - Vorhandene Slash-Befehle mit demselben Auslöser, die von anderen Integrationen erstellt wurden, bleiben unverändert (sie werden bei der Registrierung übersprungen); vom Bot erstellte Befehle werden aktualisiert oder neu erstellt, wenn sich die Callback-URL geändert hat.
    - Befehls-Callbacks werden anhand der befehlsspezifischen Token validiert, die Mattermost zurückgibt, wenn OpenClaw `oc_*` Befehle registriert.
    - OpenClaw aktualisiert die aktuelle Mattermost-Befehlsregistrierung, bevor jeder Callback akzeptiert wird. Dadurch werden veraltete Token gelöschter oder neu generierter Slash-Befehle ohne Neustart des Gateways nicht mehr akzeptiert.
    - Die Callback-Validierung schlägt sicher fehl, wenn die Mattermost-API nicht bestätigen kann, dass der Befehl noch aktuell ist; fehlgeschlagene Validierungen werden kurzzeitig zwischengespeichert, gleichzeitige Abfragen werden zusammengeführt und der Start neuer Abfragen wird pro Befehl ratenbegrenzt, um die Belastung durch Wiederholungsangriffe zu begrenzen.
    - Slash-Callbacks schlagen sicher fehl, wenn die Registrierung fehlgeschlagen ist, der Start nur teilweise abgeschlossen wurde oder das Callback-Token nicht mit dem registrierten Token des aufgelösten Befehls übereinstimmt (ein für einen Befehl gültiges Token kann die vorgelagerte Validierung für einen anderen Befehl nicht erreichen).
    - Akzeptierte Callbacks werden mit einer flüchtigen Antwort „Verarbeitung läuft …“ bestätigt; die eigentliche Antwort trifft als normale Nachricht ein.

  </Accordion>
  <Accordion title="Erreichbarkeitsanforderung">
    Der Callback-Endpunkt muss vom Mattermost-Server aus erreichbar sein.

    - Setzen Sie `callbackUrl` nicht auf `localhost`, es sei denn, Mattermost wird auf demselben Host bzw. im selben Netzwerk-Namespace wie OpenClaw ausgeführt.
    - Setzen Sie `callbackUrl` nicht auf Ihre Mattermost-Basis-URL, es sei denn, diese URL leitet `/api/channels/mattermost/command` per Reverse-Proxy an OpenClaw weiter.
    - Eine schnelle Prüfung ist `curl https://<gateway-host>/api/channels/mattermost/command`; eine GET-Anfrage sollte `405 Method Not Allowed` von OpenClaw zurückgeben, nicht `404`.

  </Accordion>
  <Accordion title="Mattermost-Ausgangs-Allowlist">
    Wenn Ihre Callback-Ziele private, Tailnet- oder interne Adressen sind, setzen Sie Mattermost `ServiceSettings.AllowedUntrustedInternalConnections` so, dass der Callback-Host bzw. die Callback-Domain enthalten ist.

    Verwenden Sie Host-/Domain-Einträge, keine vollständigen URLs.

    - Richtig: `gateway.tailnet-name.ts.net`
    - Falsch: `https://gateway.tailnet-name.ts.net`

  </Accordion>
</AccordionGroup>

## Umgebungsvariablen (Standardkonto)

Legen Sie diese auf dem Gateway-Host fest, wenn Sie Umgebungsvariablen bevorzugen:

- `MATTERMOST_BOT_TOKEN=...`
- `MATTERMOST_URL=https://chat.example.com`

<Note>
Umgebungsvariablen gelten nur für das **Standardkonto** (`default`). Andere Konten müssen Konfigurationswerte verwenden.

`MATTERMOST_URL` kann nicht über eine Workspace-`.env` festgelegt werden; siehe [Workspace-.env-Dateien](/de/gateway/security).
</Note>

## Chatmodi

Mattermost antwortet automatisch auf Direktnachrichten. Das Kanalverhalten wird durch `chatmode` gesteuert:

<Tabs>
  <Tab title="oncall (default)">
    Nur antworten, wenn in Kanälen eine @Erwähnung erfolgt.
  </Tab>
  <Tab title="onmessage">
    Auf jede Kanalnachricht antworten.
  </Tab>
  <Tab title="onchar">
    Antworten, wenn eine Nachricht mit einem Auslösepräfix beginnt.
  </Tab>
</Tabs>

Konfigurationsbeispiel:

```json5
{
  channels: {
    mattermost: {
      chatmode: "onchar",
      oncharPrefixes: [">", "!"], // Standard
    },
  },
}
```

Hinweise:

- `onchar` reagiert weiterhin auf ausdrückliche @Erwähnungen.
- `channels.mattermost.requireMention` wird weiterhin berücksichtigt, `chatmode` wird jedoch bevorzugt. Kanalspezifische `groups.<channelId>.requireMention`-Einstellungen haben Vorrang vor beiden.
- Nachdem der Bot in einem Kanal-Thread eine sichtbare Antwort gesendet hat, werden spätere Nachrichten im selben Thread ohne erneute @Erwähnung oder ein `onchar`-Präfix beantwortet, sodass mehrstufige Thread-Unterhaltungen fortgesetzt werden. Die Teilnahme wird nach der letzten Antwort des Bots in diesem Thread 7 Tage lang gespeichert und bleibt über Gateway-Neustarts hinweg bestehen. Threads, die der Bot lediglich beobachtet hat, sind davon nicht betroffen; beginnen Sie eine neue Nachricht auf oberster Ebene, damit wieder eine ausdrückliche Erwähnung erforderlich ist.
- Setzen Sie `channels.mattermost.implicitMentions.threadParticipation: false`, damit Folgenachrichten in Threads mit Bot-Beteiligung die Erwähnungsprüfung nicht umgehen. Kontospezifische Überschreibungen verwenden `channels.mattermost.accounts.<id>.implicitMentions`. Mattermost erzeugt derzeit keine `replyToBot`- oder `quotedBot`-Fakten, daher haben diese Flags hier keine Wirkung.

## Threads und Sitzungen

Verwenden Sie `channels.mattermost.replyToMode`, um festzulegen, ob Kanal- und Gruppenantworten im Hauptkanal verbleiben oder einen Thread unter dem auslösenden Beitrag beginnen.

- `off` (Standard): Nur in einem Thread antworten, wenn sich der eingehende Beitrag bereits in einem Thread befindet.
- `first`: Für Kanal- oder Gruppenbeiträge auf oberster Ebene einen Thread unter diesem Beitrag beginnen und die Unterhaltung an eine Thread-spezifische Sitzung weiterleiten.
- `all` und `batched`: Derzeit dasselbe Verhalten wie `first` für Mattermost, da nach dem Vorhandensein eines Thread-Stammbeitrags in Mattermost nachfolgende Abschnitte und Medien im selben Thread verbleiben.
- Direktnachrichten verwenden standardmäßig `off`, selbst wenn `replyToMode` festgelegt ist.

Verwenden Sie `channels.mattermost.replyToModeByChatType`, um den Modus für Chats des Typs `direct`, `group` oder `channel` zu überschreiben. Setzen Sie `direct`, um Threads für Direktnachrichten zu aktivieren:

- `off` (Standard): Direktnachrichten bleiben ohne Threads in einer fortlaufenden Sitzung.
- `first`, `all` oder `batched`: Jede Direktnachricht auf oberster Ebene beginnt einen Mattermost-Thread, dem eine neue, unabhängige Sitzung zugrunde liegt.

```json5
{
  channels: {
    mattermost: {
      replyToMode: "all",
      replyToModeByChatType: {
        direct: "first",
      },
    },
  },
}
```

Hinweise:

- Thread-spezifische Sitzungen verwenden die ID des auslösenden Beitrags als Thread-Stamm.
- `first` und `all` sind derzeit gleichwertig, da nach dem Vorhandensein eines Thread-Stamms in Mattermost nachfolgende Abschnitte und Medien im selben Thread verbleiben.
- Überschreibungen pro Chattyp haben Vorrang vor `replyToMode`. Ohne eine `direct`-Überschreibung behalten bestehende Bereitstellungen flache Direktnachrichten ohne Threads bei.

## Zugriffskontrolle (Direktnachrichten)

- Standard: `channels.mattermost.dmPolicy = "pairing"` (unbekannte Absender erhalten einen Kopplungscode). Weitere Werte: `allowlist`, `open`, `disabled`.
- Genehmigung über:
  - `openclaw pairing list mattermost`
  - `openclaw pairing approve mattermost <CODE>`
- Öffentliche Direktnachrichten: `channels.mattermost.dmPolicy="open"` plus `channels.mattermost.allowFrom=["*"]` (das Konfigurationsschema erzwingt den Platzhalter).
- `channels.mattermost.allowFrom` akzeptiert Benutzer-IDs (empfohlen) und `accessGroup:<name>`-Einträge. Siehe [Zugriffsgruppen](/de/channels/access-groups).

## Kanäle (Gruppen)

- Standard: `channels.mattermost.groupPolicy = "allowlist"` (Erwähnung erforderlich).
- Lassen Sie Absender mit `channels.mattermost.groupAllowFrom` zu (Benutzer-IDs empfohlen).
- `channels.mattermost.groupAllowFrom` akzeptiert `accessGroup:<name>`-Einträge. Siehe [Zugriffsgruppen](/de/channels/access-groups).
- Kanalspezifische Überschreibungen der Erwähnungseinstellung befinden sich unter `channels.mattermost.groups.<channelId>.requireMention` oder standardmäßig unter `channels.mattermost.groups["*"].requireMention`.
- Der Abgleich von `@username` ist veränderlich und nur aktiviert, wenn `channels.mattermost.dangerouslyAllowNameMatching: true`.
- Offene Kanäle: `channels.mattermost.groupPolicy="open"` (Erwähnung erforderlich).
- Auflösungsreihenfolge: `channels.mattermost.groupPolicy`, dann `channels.defaults.groupPolicy`, dann `"allowlist"`.
- Laufzeithinweis: Wenn der Abschnitt `channels.mattermost` vollständig fehlt, verwendet die Laufzeit für Gruppenprüfungen standardmäßig restriktiv `groupPolicy="allowlist"` (selbst wenn `channels.defaults.groupPolicy` festgelegt ist) und protokolliert einmalig eine Warnung.

Beispiel:

```json5
{
  channels: {
    mattermost: {
      groupPolicy: "open",
      groups: {
        "*": { requireMention: true },
        "team-channel-id": { requireMention: false },
      },
    },
  },
}
```

## Ziele für ausgehende Zustellung

Verwenden Sie diese Zielformate mit `openclaw message send` oder Cron/Webhooks:

| Ziel                                | Zustellung an                                                  |
| ----------------------------------- | -------------------------------------------------------------- |
| `channel:<id>`                      | Kanal anhand der ID                                            |
| `channel:<name>` oder `#channel-name` | Kanal anhand des Namens; Suche in allen Teams des Bots         |
| `user:<id>` oder `mattermost:<id>`    | Direktnachricht an diesen Benutzer                             |
| `@username`                         | Direktnachricht (Benutzername wird über die Mattermost-API aufgelöst) |

Ausgehende Nachrichten unterstützen höchstens einen Anhang pro Nachricht; teilen Sie mehrere Dateien auf separate Sendevorgänge auf.

<Warning>
Nicht näher spezifizierte undurchsichtige IDs (wie `64ifufp...`) sind in Mattermost **mehrdeutig** (Benutzer-ID oder Kanal-ID).

OpenClaw löst sie **zuerst als Benutzer** auf:

- Wenn die ID als Benutzer vorhanden ist (`GET /api/v4/users/<id>` erfolgreich ist), sendet OpenClaw eine **Direktnachricht**, indem der direkte Kanal über `/api/v4/channels/direct` aufgelöst wird.
- Andernfalls wird die ID als **Kanal-ID** behandelt.

Wenn Sie deterministisches Verhalten benötigen, verwenden Sie immer die expliziten Präfixe (`user:<id>` / `channel:<id>`).
</Warning>

## Wiederholungsversuche für Direktnachrichtenkanäle

Wenn OpenClaw an ein Mattermost-Direktnachrichtenziel sendet und zuerst den direkten Kanal auflösen muss, wiederholt es standardmäßig vorübergehend fehlgeschlagene Erstellungsversuche für direkte Kanäle.

Verwenden Sie `channels.mattermost.dmChannelRetry`, um dieses Verhalten global für das Mattermost-Plugin anzupassen, oder `channels.mattermost.accounts.<id>.dmChannelRetry` für ein einzelnes Konto. Standardwerte:

```json5
{
  channels: {
    mattermost: {
      dmChannelRetry: {
        maxRetries: 3,
        initialDelayMs: 1000,
        maxDelayMs: 10000,
        timeoutMs: 30000,
      },
    },
  },
}
```

Hinweise:

- Dies gilt nur für die Erstellung von Direktnachrichtenkanälen (`/api/v4/channels/direct`), nicht für jeden Mattermost-API-Aufruf.
- Wiederholungsversuche verwenden exponentielles Backoff mit Jitter und gelten für vorübergehende Fehler wie Ratenbegrenzungen, 5xx-Antworten sowie Netzwerk- oder Zeitüberschreitungsfehler.
- Andere 4xx-Clientfehler als `429` werden als dauerhaft behandelt und nicht erneut versucht.

## Vorschau-Streaming

Mattermost streamt Gedankengänge, Tool-Aktivitäten und Teile des Antworttexts in einen **Vorschauentwurf**, der an Ort und Stelle finalisiert wird, sobald die endgültige Antwort sicher gesendet werden kann. Im Modus `partial` wird die Vorschau unter derselben Beitrags-ID aktualisiert, statt den Kanal mit Nachrichten für jedes Fragment zu überfluten. Im Modus `block` wechselt die Vorschau zwischen abgeschlossenen Text- und Tool-Aktivitätsblöcken, sodass frühere Blöcke als eigene Beiträge sichtbar bleiben, statt vom nächsten überschrieben zu werden. Endgültige Medien-/Fehlerantworten brechen ausstehende Vorschauänderungen ab und verwenden die normale Zustellung, statt einen überflüssigen Vorschauentwurf zu veröffentlichen.

Vorschau-Streaming ist im Modus `partial` **standardmäßig aktiviert**. Konfigurieren Sie es über `channels.mattermost.streaming.mode` (veraltete skalare/boolesche `streaming`-Werte werden durch `openclaw doctor --fix` migriert):

```json5
{
  channels: {
    mattermost: {
      streaming: { mode: "partial" }, // off | partial | block | progress
    },
  },
}
```

<AccordionGroup>
  <Accordion title="Streaming-Modi">
    - `partial` (Standard): ein Vorschauentwurf, der mit zunehmendem Antwortumfang bearbeitet und anschließend mit der vollständigen Antwort finalisiert wird.
    - `block` wechselt die Vorschau zwischen abgeschlossenen Text- und Tool-Aktivitätsblöcken, sodass jeder Block als eigener Beitrag sichtbar bleibt, statt an Ort und Stelle überschrieben zu werden. Parallele und aufeinanderfolgende Tool-Aktualisierungen verwenden gemeinsam den aktuellen Tool-Aktivitätsbeitrag.
    - `progress` zeigt während der Generierung eine Statusvorschau und veröffentlicht die endgültige Antwort erst nach Abschluss.
    - `off` deaktiviert das Vorschau-Streaming. Mit `streaming.block.enabled: true` werden abgeschlossene Assistentenblöcke weiterhin als normale Blockantworten (separate Beiträge) statt als einzelner zusammengeführter endgültiger Beitrag zugestellt.

  </Accordion>
  <Accordion title="Hinweise zum Streaming-Verhalten">
    - Wenn der Stream nicht an Ort und Stelle finalisiert werden kann (beispielsweise weil der Beitrag während des Streamings gelöscht wurde), sendet OpenClaw ersatzweise einen neuen endgültigen Beitrag, damit die Antwort niemals verloren geht.
    - Nutzdaten, die ausschließlich Gedankengänge enthalten, werden in Kanalbeiträgen unterdrückt, einschließlich Text, der als `> Thinking`-Blockzitat eingeht. Setzen Sie `/reasoning on`, um Gedankengänge auf anderen Oberflächen anzuzeigen; der endgültige Mattermost-Beitrag enthält nur die Antwort.
    - Die Kanalzuordnungsmatrix finden Sie unter [Streaming](/de/concepts/streaming#preview-streaming-modes).

  </Accordion>
</AccordionGroup>

## Reaktionen (Nachrichten-Tool)

- Verwenden Sie `message action=react` mit `channel=mattermost`.
- `messageId` ist die Mattermost-Beitrags-ID.
- `emoji` akzeptiert Namen wie `thumbsup` oder `:+1:` (Doppelpunkte sind optional).
- Setzen Sie `remove=true` (boolesch), um eine Reaktion zu entfernen.
- Ereignisse zum Hinzufügen/Entfernen von Reaktionen werden als Systemereignisse an die zugeordnete Agentensitzung weitergeleitet und unterliegen denselben Richtlinienprüfungen für Direktnachrichten/Gruppen wie Nachrichten.

Beispiele:

```text
message action=react channel=mattermost target=channel:<channelId> messageId=<postId> emoji=thumbsup
message action=react channel=mattermost target=channel:<channelId> messageId=<postId> emoji=thumbsup remove=true
```

Konfiguration:

- `channels.mattermost.actions.reactions`: Reaktionsaktionen aktivieren/deaktivieren (standardmäßig true).
- Kontospezifische Überschreibung: `channels.mattermost.accounts.<id>.actions.reactions`.

## Interaktive Schaltflächen (Nachrichten-Tool)

Senden Sie Nachrichten mit anklickbaren Schaltflächen. Wenn ein Benutzer auf eine Schaltfläche klickt, erhält der Agent die Auswahl und kann antworten.

Schaltflächen stammen aus der semantischen `presentation`-Nutzlast (in normalen Agentenantworten und in `message action=send`). OpenClaw stellt Werteschaltflächen als interaktive Mattermost-Schaltflächen dar, lässt URL-Schaltflächen im Nachrichtentext sichtbar und stuft Auswahlmenüs zu lesbarem Text herab.

```text
message action=send channel=mattermost target=channel:<channelId> presentation={"blocks":[{"type":"buttons","buttons":[{"label":"Yes","value":"yes"},{"label":"No","value":"no"}]}]}
```

Felder für Darstellungsschaltflächen:

<ParamField path="label" type="string" required>
  Anzeigebeschriftung (Alias: `text`).
</ParamField>
<ParamField path="value" type="string">
  Beim Klicken zurückgesendeter Wert, der als Aktions-ID verwendet wird (Aliasse: `callback_data`, `callbackData`). Für eine anklickbare Schaltfläche erforderlich, sofern `url` nicht gesetzt ist.
</ParamField>
<ParamField path="url" type="string">
  Link-Schaltfläche; wird als `label: url`-Text im Nachrichtentext statt als interaktive Schaltfläche dargestellt.
</ParamField>
<ParamField path="style" type='"primary" | "secondary" | "success" | "danger"'>
  Schaltflächenstil. Mattermost verwendet für nicht unterstützte Werte den Standardstil.
</ParamField>

Um die Unterstützung für Schaltflächen im Agenten-Systemprompt anzugeben, fügen Sie `inlineButtons` zu den Kanalfunktionen hinzu:

```json5
{
  channels: {
    mattermost: {
      capabilities: ["inlineButtons"],
    },
  },
}
```

Wenn ein Benutzer auf eine Schaltfläche klickt:

<Steps>
  <Step title="Zugriffsprüfung">
    Die klickende Person muss dieselben Richtlinienprüfungen für Direktnachrichten/Gruppen wie ein Nachrichtenabsender bestehen; bei nicht autorisierten Klicks wird ein flüchtiger Hinweis angezeigt und der Klick ignoriert.
  </Step>
  <Step title="Schaltflächen durch Bestätigung ersetzt">
    Alle Schaltflächen werden durch eine Bestätigungszeile ersetzt (z. B. „✓ **Yes** ausgewählt von @user“).
  </Step>
  <Step title="Agent erhält die Auswahl">
    Der Agent erhält die Auswahl als eingehende Nachricht (zusätzlich zu einem Systemereignis) und antwortet.
  </Step>
</Steps>

<AccordionGroup>
  <Accordion title="Implementierungshinweise">
    - Schaltflächen-Callbacks verwenden eine HMAC-SHA256-Verifizierung (automatisch, keine Konfiguration erforderlich).
    - Beim Klicken wird der gesamte Anhangsblock ersetzt, sodass alle Schaltflächen gemeinsam entfernt werden – eine teilweise Entfernung ist nicht möglich.
    - Aktions-IDs mit Bindestrichen oder Unterstrichen werden automatisch bereinigt (Mattermost-Routing-Beschränkung).
    - Klicks, deren `action_id` mit keiner Aktion des ursprünglichen Beitrags übereinstimmt, werden mit `403` („Unbekannte Aktion“) abgelehnt.

  </Accordion>
  <Accordion title="Konfiguration und Erreichbarkeit">
    - `channels.mattermost.capabilities`: Array von Funktionszeichenfolgen. Fügen Sie `"inlineButtons"` hinzu, um die Beschreibung des Schaltflächen-Tools im Agenten-Systemprompt zu aktivieren.
    - `channels.mattermost.interactions.callbackBaseUrl`: optionale externe Basis-URL für Schaltflächen-Callbacks (beispielsweise `https://gateway.example.com`). Verwenden Sie diese, wenn Mattermost den Gateway unter dessen Bind-Host nicht direkt erreichen kann.
    - Bei Konfigurationen mit mehreren Konten können Sie dasselbe Feld auch unter `channels.mattermost.accounts.<id>.interactions.callbackBaseUrl` festlegen.
    - Wenn `interactions.callbackBaseUrl` weggelassen wird, leitet OpenClaw die Callback-URL aus `gateway.customBindHost` + `gateway.port` (Standard 18789) ab und greift anschließend auf `http://localhost:<port>` zurück. Der Callback-Pfad lautet `/mattermost/interactions/<accountId>`.
    - Erreichbarkeitsregel: Die Schaltflächen-Callback-URL muss vom Mattermost-Server aus erreichbar sein. `localhost` funktioniert nur, wenn Mattermost und OpenClaw auf demselben Host/im selben Netzwerk-Namespace ausgeführt werden.
    - `channels.mattermost.interactions.allowedSourceIps`: Quell-IP-Zulassungsliste für Schaltflächen-Callbacks. Ohne diese werden nur Loopback-Quellen (`127.0.0.1`, `::1`) akzeptiert. Daher muss ein entfernter Mattermost-Server hier in die Zulassungsliste aufgenommen werden, andernfalls werden seine Klicks mit `403` abgelehnt. Legen Sie hinter einem Reverse-Proxy außerdem `gateway.trustedProxies` fest, damit die tatsächliche Client-IP aus weitergeleiteten Headern ermittelt wird.
    - Wenn Ihr Callback-Ziel privat/im Tailnet/intern ist, fügen Sie dessen Host/Domain zu Mattermost `ServiceSettings.AllowedUntrustedInternalConnections` hinzu.

  </Accordion>
</AccordionGroup>

### Direkte API-Integration (externe Skripte)

Externe Skripte und Webhooks können Schaltflächen direkt über die Mattermost-REST-API veröffentlichen, statt das `message`-Tool des Agenten zu verwenden. Bevorzugen Sie das `message`-Tool von OpenClaw. Importieren Sie für direkte Integrationen `buildButtonAttachments` aus `@openclaw/mattermost/api.js`; wenn Sie unverarbeitetes JSON veröffentlichen, befolgen Sie diese Regeln:

**Nutzlaststruktur:**

```json5
{
  channel_id: "<channelId>",
  message: "Choose an option:",
  props: {
    attachments: [
      {
        actions: [
          {
            id: "mybutton01", // alphanumeric only - see below
            type: "button", // required, or clicks are silently ignored
            name: "Approve", // display label
            style: "primary", // optional: "default", "primary", "danger"
            integration: {
              url: "https://gateway.example.com/mattermost/interactions/default",
              context: {
                action_id: "mybutton01", // must match button id
                action: "approve",
                // ... any custom fields ...
                _token: "<hmac>", // see HMAC section below
              },
            },
          },
        ],
      },
    ],
  },
}
```

<Warning>
**Kritische Regeln**

1. Anhänge gehören in `props.attachments`, nicht in `attachments` auf oberster Ebene (wird stillschweigend ignoriert).
2. Jede Aktion benötigt `type: "button"` – ohne dieses Feld werden Klicks stillschweigend verworfen.
3. Jede Aktion benötigt ein `id`-Feld – Mattermost ignoriert Aktionen ohne IDs.
4. Die Aktions-`id` darf **nur alphanumerische Zeichen** enthalten (`[a-zA-Z0-9]`). Bindestriche und Unterstriche beeinträchtigen das serverseitige Aktions-Routing von Mattermost (Antwort 404). Entfernen Sie sie vor der Verwendung.
5. `context.action_id` muss mit der `id` der Schaltfläche übereinstimmen; der Gateway lehnt Klicks ab, deren `action_id` im Beitrag nicht vorhanden ist.
6. `context.action_id` ist erforderlich – der Interaktionshandler gibt ohne dieses Feld 400 zurück.
7. Die Callback-Quell-IP muss zugelassen sein (siehe `interactions.allowedSourceIps` oben).

</Warning>

**HMAC-Token-Generierung**

Der Gateway verifiziert Schaltflächenklicks mit HMAC-SHA256. Externe Skripte müssen Token generieren, die der Verifizierungslogik des Gateways entsprechen:

<Steps>
  <Step title="Geheimnis aus dem Bot-Token ableiten">
    `HMAC-SHA256(key="openclaw-mattermost-interactions", data=botToken)`, hexadezimal codiert.
  </Step>
  <Step title="Kontextobjekt erstellen">
    Erstellen Sie das Kontextobjekt mit allen Feldern **außer** `_token`.
  </Step>
  <Step title="Mit sortierten Schlüsseln serialisieren">
    Serialisieren Sie mit **rekursiv sortierten Schlüsseln** und **ohne Leerzeichen** (der Gateway kanonisiert auch verschachtelte Objekte und erzeugt kompaktes JSON).
  </Step>
  <Step title="Nutzlast signieren">
    `HMAC-SHA256(key=secret, data=serializedContext)`
  </Step>
  <Step title="Token hinzufügen">
    Fügen Sie den resultierenden hexadezimalen Digest als `_token` zum Kontext hinzu.
  </Step>
</Steps>

Python-Beispiel:

```python
import hmac, hashlib, json

secret = hmac.new(
    b"openclaw-mattermost-interactions",
    bot_token.encode(), hashlib.sha256
).hexdigest()

ctx = {"action_id": "mybutton01", "action": "approve"}
payload = json.dumps(ctx, sort_keys=True, separators=(",", ":"))
token = hmac.new(secret.encode(), payload.encode(), hashlib.sha256).hexdigest()

context = {**ctx, "_token": token}
```

<AccordionGroup>
  <Accordion title="Häufige HMAC-Fallstricke">
    - Pythons `json.dumps` fügt standardmäßig Leerzeichen hinzu (`{"key": "val"}`). Verwenden Sie `separators=(",", ":")`, damit die Ausgabe dem kompakten Format von JavaScript entspricht (`{"key":"val"}`).
    - Signieren Sie stets **alle** Kontextfelder (mit Ausnahme von `_token`). Der Gateway entfernt `_token` und signiert anschließend alle verbleibenden Felder. Das Signieren einer Teilmenge führt zu einem stillen Verifizierungsfehler.
    - Verwenden Sie `sort_keys=True` – der Gateway sortiert die Schlüssel vor dem Signieren, und Mattermost kann die Kontextfelder beim Speichern der Nutzlast neu anordnen.
    - Leiten Sie das Geheimnis aus dem Bot-Token ab (deterministisch), nicht aus zufälligen Bytes. Das Geheimnis muss in dem Prozess, der die Schaltflächen erstellt, und im Gateway, der sie verifiziert, identisch sein.

  </Accordion>
</AccordionGroup>

## Verzeichnisadapter

Das Mattermost-Plugin enthält einen Verzeichnisadapter, der Kanal- und Benutzernamen über die Mattermost-API auflöst. Dies ermöglicht `#channel-name`- und `@username`-Ziele in `openclaw message send` sowie bei Cron-/Webhook-Zustellungen.

Es ist keine Konfiguration erforderlich – der Adapter verwendet das Bot-Token aus der Kontokonfiguration.

## Mehrere Konten

Mattermost unterstützt mehrere Konten unter `channels.mattermost.accounts`:

```json5
{
  channels: {
    mattermost: {
      accounts: {
        default: { name: "Primary", botToken: "mm-token", baseUrl: "https://chat.example.com" },
        alerts: { name: "Alerts", botToken: "mm-token-2", baseUrl: "https://alerts.example.com" },
      },
    },
  },
}
```

Kontowerte überschreiben Felder der obersten Ebene; `channels.mattermost.defaultAccount` bestimmt, welches Konto verwendet wird, wenn keines angegeben ist.

## Fehlerbehebung

<AccordionGroup>
  <Accordion title="Keine Antworten in Kanälen">
    Stellen Sie sicher, dass sich der Bot im Kanal befindet, und erwähnen Sie ihn (oncall), verwenden Sie ein Auslösepräfix (onchar) oder setzen Sie `chatmode: "onmessage"`.
  </Accordion>
  <Accordion title="Authentifizierungs- oder Mehrkontenfehler">
    - Überprüfen Sie das Bot-Token, die Basis-URL und ob das Konto aktiviert ist.
    - Probleme mit mehreren Konten: Umgebungsvariablen gelten nur für das Konto `default`.
    - Private/LAN-Mattermost-Hosts benötigen `network.dangerouslyAllowPrivateNetwork: true` (der SSRF-Schutz blockiert standardmäßig private IP-Adressen).

  </Accordion>
  <Accordion title="Native Slash-Befehle schlagen fehl">
    - `Unauthorized: invalid command token.`: OpenClaw hat das Callback-Token nicht akzeptiert. Typische Ursachen:
      - Die Registrierung des Slash-Befehls ist fehlgeschlagen oder wurde beim Start nur teilweise abgeschlossen.
      - Der Callback erreicht den falschen Gateway bzw. das falsche Konto.
      - Mattermost verfügt noch über alte Befehle, die auf ein vorheriges Callback-Ziel verweisen.
      - Der Gateway wurde neu gestartet, ohne die Slash-Befehle erneut zu aktivieren.
    - Wenn native Slash-Befehle nicht mehr funktionieren, prüfen Sie die Protokolle auf `mattermost: failed to register slash commands` oder `mattermost: native slash commands enabled but no commands could be registered`.
    - Wenn `callbackUrl` weggelassen wird und die Protokolle davor warnen, dass der Callback zu einer Loopback-URL wie `http://localhost:18789/...` aufgelöst wurde, ist diese URL wahrscheinlich nur erreichbar, wenn Mattermost im selben Host-/Netzwerk-Namespace wie OpenClaw ausgeführt wird. Legen Sie stattdessen explizit eine extern erreichbare `commands.callbackUrl` fest.

  </Accordion>
  <Accordion title="Probleme mit Schaltflächen">
    - Schaltflächen erscheinen als weiße Kästchen oder überhaupt nicht: Die Schaltflächendaten sind fehlerhaft. Jede Präsentationsschaltfläche benötigt `label` und `value` (Schaltflächen, bei denen eines davon fehlt, werden verworfen).
    - Schaltflächen werden dargestellt, aber Klicks haben keine Wirkung: Stellen Sie sicher, dass der Gateway vom Mattermost-Server aus erreichbar ist, die IP-Adresse des Mattermost-Servers in `channels.mattermost.interactions.allowedSourceIps` enthalten ist (ohne diese Angabe wird nur Loopback akzeptiert) und `ServiceSettings.AllowedUntrustedInternalConnections` bei privaten Zielen den Callback-Host enthält.
    - Schaltflächen geben beim Klicken 404 zurück: Die `id` der Schaltfläche enthält wahrscheinlich Bindestriche oder Unterstriche. Der Aktionsrouter von Mattermost funktioniert nicht mit nicht alphanumerischen IDs. Verwenden Sie ausschließlich `[a-zA-Z0-9]`.
    - Der Gateway protokolliert `rejected callback source`: Der Klick kam von einer IP-Adresse außerhalb von `interactions.allowedSourceIps`. Nehmen Sie den Mattermost-Server oder Ihren Ingress in die Positivliste auf und setzen Sie `gateway.trustedProxies`, wenn Sie einen Reverse-Proxy verwenden.
    - Der Gateway protokolliert `invalid _token`: HMAC stimmt nicht überein. Prüfen Sie, ob Sie alle Kontextfelder (nicht nur eine Teilmenge) signieren, sortierte Schlüssel und kompaktes JSON (ohne Leerzeichen) verwenden. Weitere Informationen finden Sie im HMAC-Abschnitt oben.
    - Der Gateway protokolliert `missing _token in context`: Das Feld `_token` ist nicht im Kontext der Schaltfläche enthalten. Stellen Sie sicher, dass es beim Erstellen der Integrationsnutzlast einbezogen wird.
    - Der Gateway lehnt den Klick mit `Unknown action` ab: `context.action_id` stimmt mit keiner `id`-Aktion des Beitrags überein. Setzen Sie beide auf denselben bereinigten Wert.
    - Der Agent bietet keine Schaltflächen an: Fügen Sie `capabilities: ["inlineButtons"]` zur Mattermost-Kanalkonfiguration hinzu.

  </Accordion>
</AccordionGroup>

## Verwandte Themen

- [Kanalrouting](/de/channels/channel-routing) – Sitzungsrouting für Nachrichten
- [Kanalübersicht](/de/channels) – alle unterstützten Kanäle
- [Gruppen](/de/channels/groups) – Verhalten von Gruppenchats und Steuerung durch Erwähnungen
- [Kopplung](/de/channels/pairing) – DM-Authentifizierung und Kopplungsablauf
- [Sicherheit](/de/gateway/security) – Zugriffsmodell und Absicherung
