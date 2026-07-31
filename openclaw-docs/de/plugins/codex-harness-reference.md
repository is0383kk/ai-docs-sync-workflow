---
read_when:
    - Sie benötigen jedes Konfigurationsfeld des Codex-Harnesses
    - Sie ändern das Transport-, Authentifizierungs-, Erkennungs- oder Zeitüberschreitungsverhalten des App-Servers
    - Sie debuggen den Start des Codex-Harness, die Modellerkennung oder die Umgebungsisolierung
summary: Referenz zu Konfiguration, Authentifizierung, Erkennung und App-Server für das Codex-Harness
title: Codex-Harness-Referenz
x-i18n:
    generated_at: "2026-07-26T19:05:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 149f065f5bef18d0f491c97facc4b5991afc3f7e1077abdc7a4b49f506eac3e0
    source_path: plugins/codex-harness-reference.md
    workflow: 16
---

Diese Referenz behandelt die detaillierte Konfiguration für das offizielle `codex`-Plugin.
Beginnen Sie bei Entscheidungen zu Einrichtung und Routing mit dem
[Codex-Harness](/de/plugins/codex-harness).

## Plugin-Konfigurationsoberfläche

Alle Codex-Harness-Einstellungen befinden sich unter `plugins.entries.codex.config`.

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          discovery: {
            enabled: true,
            timeoutMs: 2500,
          },
          appServer: {
            mode: "guardian",
          },
        },
      },
    },
  },
}
```

Felder der obersten Ebene:

| Feld                       | Standard                 | Bedeutung                                                                                                                                                  |
| -------------------------- | ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `discovery`                | aktiviert                | Einstellungen zur Modellerkennung für Codex-App-Server `model/list`.                                                                                         |
| `appServer`                | verwalteter stdio-App-Server | Einstellungen für Transport, Befehl, Authentifizierung, Genehmigung, Sandbox und Zeitüberschreitung. Der gewöhnliche Harness verwendet standardmäßig agentenspezifischen Zustand. |
| `codexDynamicToolsLoading` | `"searchable"`           | Verwenden Sie `"direct"`, um dynamische OpenClaw-Tools direkt in den anfänglichen Codex-Toolkontext aufzunehmen.                                           |
| `codexDynamicToolsExclude` | `[]`                     | Zusätzliche Namen dynamischer OpenClaw-Tools, die bei Codex-App-Server-Durchläufen ausgelassen werden sollen.                                                |
| `codexPlugins`             | deaktiviert              | Native Codex-Plugin-/App-Unterstützung einschließlich optionalem Zugriff auf Apps verbundener Konten. Siehe [Native Codex-Plugins](/de/plugins/codex-native-plugins). |
| `computerUse`              | deaktiviert              | Einrichtung von Codex Computer Use. Siehe [Codex Computer Use](/de/plugins/codex-computer-use).                                                               |
| `sessionCatalog`           | aktiviert                | Native Codex-Sitzungserkennung für die Seitenleiste. Setzen Sie `enabled: false`, um die Erkennung zu deaktivieren, ohne den Provider oder Harness zu deaktivieren. |
| `supervision`              | deaktiviert              | Agentenseitige Richtlinie für Transkripte und Schreibsteuerung nativer Sitzungen. Siehe [Codex-Überwachung](/de/plugins/codex-supervision).                    |

## Überwachung

Die native Sitzungserkennung listet standardmäßig nicht archivierte Codex-Sitzungen
vom Gateway-Computer und von aktivierten gekoppelten Nodes auf. Deaktivieren Sie
nur diesen Katalog mit:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          sessionCatalog: {
            enabled: false,
          },
        },
      },
    },
  },
}
```

`supervision` steuert agentenseitige Tools separat:

| Feld                  | Standard                | Bedeutung                                                                                                                                                                                                                                  |
| --------------------- | ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `enabled`             | `false`                 | Aktiviert agentenseitige Codex-Überwachungstools. Dies steuert nicht den Katalog authentifizierter Operatorsitzungen.                                                                                                                       |
| `endpoints`           | integrierter lokaler Endpunkt | Kompatibilitäts- und erweiterte Endpunktziele für den beibehaltenen Codex-Überwachungsagenten und eigenständige MCP-Tools. Der Benutzerkatalog und der Branch-Ablauf ignorieren diese Ziele und verwenden den über `appServer` aufgelösten Überwachungs-App-Server. |
| `allowRawTranscripts` | `false`                 | Erlaubt bei aktivierter Überwachung autonome Transkriptlesevorgänge durch Agenten oder eigenständige MCP sowie aus Transkripten abgeleitete Listenfelder. Reine Metadaten-Lesevorgänge über `codex_threads` bleiben verfügbar. Steuert nicht die authentifizierte Fortsetzung in der Control UI. |
| `allowWriteControls`  | `false`                 | Erlaubt bei aktivierter Überwachung autonome `codex_threads`-Mutationen zum Forken, Umbenennen, Archivieren und Wiederherstellen sowie eigenständige MCP-Operationen zum Senden, Steuern und Unterbrechen. Umgeht keine anderen Bindungs-, Host-, Status- oder Bestätigungsprüfungen. |

Endpunkteinträge akzeptieren diese Felder:

| Feld           | Gilt für      | Bedeutung                                                               |
| -------------- | ------------- | ----------------------------------------------------------------------- |
| `id`           | alle          | Stabile Endpunkt-ID.                                                     |
| `label`        | alle          | Optionale Anzeigebeschriftung.                                          |
| `transport`    | alle          | `"stdio-proxy"` oder `"websocket"`.                                     |
| `command`      | `stdio-proxy` | Optionaler App-Server-Befehl.                                            |
| `args`         | `stdio-proxy` | Optionale Befehlsargumente.                                              |
| `cwd`          | `stdio-proxy` | Optionales Arbeitsverzeichnis des untergeordneten Prozesses.             |
| `url`          | `websocket`   | Erforderliche WebSocket- oder unterstützte lokale Socket-URL.            |
| `authTokenEnv` | `websocket`   | Optionale Umgebungsvariable, deren Wert den Endpunkt authentifiziert.     |

Die Seite **Codex-Sitzungen** verwendet den Überwachungs-App-Server des Plugins
und zeigt nur nicht archivierte Sitzungen an. Ohne explizite
`appServer`-Verbindungseinstellungen wird diese Verbindung als
verwaltetes Benutzer-Home-stdio betrieben. Gespeicherte oder inaktive lokale
Zeilen können einen modellgebundenen Chat mit begrenztem Benutzer- und
Assistentenverlauf bis zum letzten dauerhaft gespeicherten abschließenden
Quelldurchlauf erstellen. Seine private Bindung hält den Snapshot-Fork, den
kanonischen Branch der `appServer`-Quelle, die Verlaufsinjektion und
spätere Durchläufe auf dieser Verbindung. Beim ersten kanonischen Start wird
das vom Fork zurückgegebene Paar verwendet. Bei späteren Fortsetzungen werden
OpenClaw-Modell- und Provider-Überschreibungen ausgelassen, sodass Codex das
dauerhaft gespeicherte Paar des kanonischen Threads wiederherstellt. Eine
separate native Änderung kann dieses Paar aktualisieren, aber das äußere Modell
und die Fallback-Kette ersetzen es niemals. Gespeicherte und inaktive Zeilen
können nach der Bestätigung, dass kein anderer Runner vorhanden ist, archiviert
werden, sofern nicht eine andere aktive OpenClaw-Bindung das exakte Ziel oder
einen seiner nicht archivierten erzeugten Nachfolger besitzt. OpenClaw folgt
der Nachfolger-Paginierung von Codex und bricht bei Aufzählungsfehlern, Zyklen
oder dem Ausschöpfen der Sicherheitsgrenze sicher ab. Die Bestätigung deckt
weiterhin unbekannte native Clients und das Zeitfenster zwischen Statusprüfung
und Archivierung ab. Ein überwachter modellgebundener Chat kann nicht gelöscht
werden, solange er die native Bindung schützt. Aktive Quellen können weder
einen Branch erstellen noch archiviert werden, ein bestehender überwachter
Chat kann jedoch weiterhin geöffnet werden. Jede Zeile eines gekoppelten Nodes
bleibt schreibgeschützt; der Node-Transport stellt den für den Harness
erforderlichen Streaming-Lebenszyklus noch nicht bereit.

`appServer.homeScope: "user"` allein ändert, welches Codex-Home ein verwalteter
Harness-Prozess verwendet; die Fleet-Katalogisierung wird dadurch nicht
veröffentlicht. Das Aktivieren der Überwachung ändert den Standardwert des
Harness nicht. Stattdessen verwendet die separate Überwachungsverbindung
standardmäßig verwaltetes Benutzer-Home-stdio, wenn keine expliziten
`appServer`-Verbindungseinstellungen vorhanden sind. Explizite
Einstellungen werden für diese Verbindung berücksichtigt. Ausstehende und
festgeschriebene überwachte Bindungen behalten diese Verbindung für jeden
Durchlauf bei; eine deaktivierte Überwachung oder eine Abweichung bei Verbindung
oder Lebenszyklus führt zu einem sicheren Abbruch, statt auf den Agent-Home-
Harness zurückzufallen. Die Standardverbindung teilt gespeicherte Sitzungen mit
nativen Codex-Clients, nicht deren prozesslokalen Aktivitätszustand.

Veraltete `plugins.entries.codex-supervisor`-Einstellungen werden nicht mehr unterstützt. Führen
Sie `openclaw doctor --fix` aus, um den alten Eintrag, die Endpunktdefinitionen,
Richtlinien-Flags und Plugin-Zulassungs-/Ablehnungsreferenzen in diesen Block zu
migrieren. Explizite kanonische `codex.config.supervision`-Werte haben bei Konflikten
Vorrang.

## App-Server-Transport

Für gewöhnliche Harness-Durchläufe startet OpenClaw die verwaltete Codex-Binärdatei,
die mit dem offiziellen Plugin ausgeliefert wird (derzeit `@openai/codex`
`0.145.0`):

```bash
codex app-server --listen stdio://
```

Dadurch bleibt die App-Server-Version an das offizielle
`codex`-Plugin gebunden und nicht an eine beliebige separat lokal
installierte Codex CLI. Setzen Sie `appServer.command` nur, wenn Sie bewusst eine
andere ausführbare Datei verwenden möchten. Gewöhnliche verwaltete Durchläufe
mit dem standardmäßigen isolierten Agent-Home bevorzugen dieses fixierte Paket
auch dann, wenn ein macOS-Desktop-Bundle installiert ist. Wenn
[Computer Use](/de/plugins/codex-computer-use) aktiviert ist oder wenn
`homeScope` den Wert `"user"` hat und nativen
Computer-Use-Zustand laden kann, bevorzugt der verwaltete Start stattdessen die
Binärdatei der Desktop-App, die über die erforderlichen macOS-Berechtigungen
verfügt. Dieselbe Desktop-zuerst-Regel gilt, wenn die wirksame Codex-Konfiguration
eines isolierten Agent-Home natives Computer Use aktiviert. Wenn kein
Desktop-App-Bundle installiert ist, greift OpenClaw auf die Binärdatei des
fixierten Pakets zurück.

Die Übergabe ausführbarer Dateien und die Abschirmung nativer Konfiguration
koordinieren Clients innerhalb eines laufenden Gateway-Prozesses. Starten Sie
den Gateway neu, nachdem ein anderer Prozess die native Codex-Plugin-Konfiguration
geändert hat.

Die Überwachung löst eine separate Verbindung auf. Ohne explizite
`appServer`-Verbindungseinstellungen verwendet sie verwaltetes stdio mit
`homeScope: "user"`; der gewöhnliche Harness bleibt bei verwaltetem stdio mit
`homeScope: "agent"`. Explizite Verbindungseinstellungen werden von beiden Pfaden
berücksichtigt. Setzen Sie `homeScope: "user"` explizit, wenn der gewöhnliche
Harness `$CODEX_HOME` (oder `~/.codex`) mit nativen Clients teilen
soll. Eine private überwachte Bindung verwendet unabhängig vom Standardwert des
gewöhnlichen Harness die Überwachungsverbindung. Unabhängige App-Server-Prozesse
behalten getrennte Live-Status- und Genehmigungszustände bei.

Für Tests außerhalb der Produktion mit einem bereits laufenden App-Server ist
der WebSocket-Transport verfügbar:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            transport: "websocket",
            url: "ws://gateway-host:39175",
            authToken: "${CODEX_APP_SERVER_TOKEN}",
            requestTimeoutMs: 60000,
          },
        },
      },
    },
  },
}
```

Codex stuft den WebSocket-Transport als experimentell und nicht unterstützt ein.
Bevorzugen Sie für Produktionsworkloads verwaltetes stdio oder den lokalen
Unix-Steuerungssocket.

Felder von `appServer`:

| Feld                                          | Standard                                               | Bedeutung                                                                                                                                                                                                                                                                                                                                                                                       |
| --------------------------------------------- | ------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `transport`                            | `"stdio"`                                     | `"stdio"` startet Codex; ein explizites `"unix"` stellt eine Verbindung zum lokalen Steuerungssocket her; `"websocket"` stellt eine Verbindung zu `url` her.                                                                                                                                                                                                 |
| `homeScope`                            | `"agent"`                                     | `"agent"` isoliert den regulären Harness-Status für jeden OpenClaw-Agenten. `"user"` ist eine explizite Opt-in-Option, die das native `$CODEX_HOME` oder `~/.codex` gemeinsam nutzt, native Authentifizierung verwendet und die Thread-Verwaltung ausschließlich für den Eigentümer aktiviert. Der Benutzerbereich unterstützt lokales stdio oder Unix-Transport. Für die separate Überwachungsverbindung wird ein nicht gesetzter Wert bei stdio oder Unix in `"user"` und bei WebSocket in `"agent"` aufgelöst. |
| `command`                            | verwaltete Codex-Binärdatei                            | Ausführbare Datei für den stdio-Transport. Lassen Sie die Einstellung leer, um die verwaltete Binärdatei zu verwenden.                                                                                                                                                                                                                                                                           |
| `args`                            | `["app-server", "--listen", "stdio://"]`                                     | Argumente für den stdio-Transport.                                                                                                                                                                                                                                                                                                                                                              |
| `url`                            | nicht gesetzt                                          | WebSocket-App-Server-URL oder `unix://`-URL. Ein explizit leerer Unix-Pfad wählt das kanonische Steuerungssocket im Benutzerverzeichnis aus.                                                                                                                                                                                                                                             |
| `authToken`                            | nicht gesetzt                                          | Bearer-Token für den WebSocket-Transport. Akzeptiert eine Literalzeichenfolge oder SecretInput wie `${CODEX_APP_SERVER_TOKEN}`.                                                                                                                                                                                                                                                                           |
| `headers`                            | `{}`                                     | Zusätzliche WebSocket-Header. Headerwerte akzeptieren Literalzeichenfolgen oder SecretInput-Werte, beispielsweise `x-codex-client-session-token: "${CODEX_CLIENT_SESSION_TOKEN}"`.                                                                                                                                                                                                                                                            |
| `clearEnv`                            | `[]`                                     | Namen zusätzlicher Umgebungsvariablen, die aus dem gestarteten stdio-App-Server-Prozess entfernt werden, nachdem OpenClaw dessen geerbte Umgebung erstellt hat.                                                                                                                                                                                                                                  |
| `remoteWorkspaceRoot`                            | nicht gesetzt                                          | Workspace-Stammverzeichnis des entfernten Codex-App-Servers. Wenn dieser Wert gesetzt ist, leitet OpenClaw das lokale Workspace-Stammverzeichnis aus dem aufgelösten OpenClaw-Workspace ab, behält das aktuelle cwd-Suffix unter diesem entfernten Stammverzeichnis bei und sendet nur das endgültige App-Server-cwd an Codex. Befindet sich das cwd außerhalb des aufgelösten OpenClaw-Workspace-Stammverzeichnisses, schlägt OpenClaw sicher fehl, statt einen Gateway-lokalen Pfad an den entfernten App-Server zu senden. |
| `loopDetectionPreToolUseRelay`                            | `true`                                     | Installiert den Codex-Unterprozess `PreToolUse`, der ausschließlich für die OpenClaw-Schleifenerkennung und deren explizite Kennzeichnung „keine Richtlinie“ verwendet wird. Setzen Sie `false`, um die Prozessauffächerung pro Tool zu reduzieren. Plugin-Hooks vor der Tool-Ausführung und die Richtlinie für vertrauenswürdige Tools installieren weiterhin ihr erforderliches Relay. |
| `requestTimeoutMs`                            | `60000`                                     | Zeitüberschreitung für Steuerungsebenenaufrufe des App-Servers.                                                                                                                                                                                                                                                                                                                                 |
| `turnCompletionIdleTimeoutMs`                            | `60000`                                     | Ruhefenster, nachdem Codex einen Turn angenommen hat oder nachdem eine Turn-bezogene App-Server-Anfrage erfolgt ist, während OpenClaw auf `turn/completed` wartet.                                                                                                                                                                                                                              |
| `turnAssistantCompletionIdleTimeoutMs`                            | `10000`                                     | Ruhefenster, nachdem ein abschließendes bzw. nicht kommentierendes Assistentenelement oder der rohe Assistentenabschluss vor einem Tool die Freigabe der Assistentenausgabe aktiviert hat, während OpenClaw weiterhin auf `turn/completed` wartet. Eine Erhöhung gibt Codex mehr Zeit, `turn/completed` auszugeben, bevor OpenClaw unterbricht und die Sitzungsspur freigibt. |
| `postToolRawAssistantCompletionIdleTimeoutMs`                            | `300000`                                     | Abschlussleerlauf- und Fortschrittswächter, der nach einer Tool-Übergabe, dem Abschluss eines nativen Tools, rohem Assistentenfortschritt nach einem Tool, dem Abschluss roher Schlussfolgerungen oder Schlussfolgerungsfortschritt verwendet wird, während OpenClaw auf `turn/completed` wartet. Verwenden Sie dies für vertrauenswürdige oder rechenintensive Workloads, bei denen die Synthese nach einem Tool berechtigterweise länger still bleiben kann als das Freigabebudget des abschließenden Assistenten. |
| `mode`                            | `"yolo"`, sofern lokale Codex-Anforderungen YOLO nicht ausschließen | Voreinstellung für YOLO oder durch einen Guardian überprüfte Ausführung.                                                                                                                                                                                                                                                                                                                        |
| `approvalPolicy`                            | `"never"` oder eine zulässige Guardian-Genehmigungsrichtlinie | Native Codex-Genehmigungsrichtlinie, die beim Start und Fortsetzen eines Threads sowie beim Turn gesendet wird.                                                                                                                                                                                                                                                                                  |
| `sandbox`                            | `"danger-full-access"` oder eine zulässige Guardian-Sandbox | Nativer Codex-Sandbox-Modus, der beim Start und Fortsetzen eines Threads gesendet wird. Aktive OpenClaw-Sandboxes beschränken `danger-full-access`-Turns auf Codex `workspace-write`; das Netzwerk-Flag des Turns folgt dem ausgehenden Datenverkehr der OpenClaw-Sandbox.                                                                                                                               |
| `approvalsReviewer`                            | `"user"` oder ein zulässiger Guardian-Prüfer | Verwenden Sie `"auto_review"`, damit Codex native Genehmigungsaufforderungen überprüfen kann, sofern dies zulässig ist.                                                                                                                                                                                                                                                                       |
| `defaultWorkspaceDir`                            | aktuelles Prozessverzeichnis                           | Workspace, der von `/codex bind` verwendet wird, wenn `--cwd` weggelassen wird.                                                                                                                                                                                                                                                                                                 |
| `serviceTier`                            | nicht gesetzt                                          | Optionale Dienstklasse des Codex-App-Servers. `"priority"` aktiviert das Routing im Schnellmodus, `"flex"` fordert flexible Verarbeitung an und `null` entfernt die Überschreibung. Das veraltete `"fast"` wird als `"priority"` akzeptiert.                                                                                                                                                           |
| `networkProxy`                            | deaktiviert                                            | Aktiviert optional das Netzwerkprofil für Codex-Berechtigungen bei App-Server-Befehlen. OpenClaw definiert die ausgewählte `permissions.<profile>.network`-Konfiguration und wählt sie mit `default_permissions` aus, statt `sandbox` zu senden.                                                                                                                                                            |
| `experimental.sandboxExecServer`              | `false`                                                | Optionale Vorschauaktivierung, die eine durch die OpenClaw-Sandbox gestützte Codex-Umgebung beim unterstützten Codex-App-Server registriert, sodass die native Codex-Ausführung innerhalb der aktiven OpenClaw-Sandbox ausgeführt werden kann.                                                                                                                                                                                                            |

`appServer.networkProxy` ist explizit, da es den Sandbox-Vertrag von Codex
ändert. Wenn diese Option aktiviert ist, setzt OpenClaw außerdem `features.network_proxy.enabled` und
`default_permissions` in der Codex-Thread-Konfiguration, sodass das generierte Berechtigungsprofil
die von Codex verwaltete Netzwerkfunktion starten kann. OpenClaw generiert standardmäßig
einen kollisionsresistenten Profilnamen `openclaw-network-<fingerprint>` aus dem
Profilinhalt; verwenden Sie `profileName` nur, wenn ein stabiler lokaler Name
erforderlich ist.

```js
export default {
  plugins: {
    entries: {
      codex: {
        config: {
          appServer: {
            sandbox: "workspace-write",
            networkProxy: {
              enabled: true,
              domains: {
                "api.openai.com": "allow",
                "blocked.example.com": "deny",
              },
              allowUpstreamProxy: true,
              proxyUrl: "http://127.0.0.1:3128",
            },
          },
        },
      },
    },
  },
};
```

Wenn die normale App-Server-Laufzeit `danger-full-access` wäre, verwendet die Aktivierung von
`networkProxy` stattdessen einen Arbeitsbereich-ähnlichen Dateisystemzugriff für das
generierte Berechtigungsprofil. Die von Codex verwaltete Netzwerkdurchsetzung ist eine
Sandbox-Netzwerkfunktion, daher würde ein Profil mit Vollzugriff den ausgehenden Datenverkehr nicht schützen.

Das Plugin blockiert ältere, neuere nicht validierte, Vorab-, mit Build-Suffix versehene oder
unversionierte App-Server-Handshakes. Der Codex-App-Server muss eine stabile Version
von `0.143.0` bis einschließlich des gebündelten `0.145.0` melden.

OpenClaw behandelt WebSocket-App-Server-URLs, die nicht auf die Loopback-Schnittstelle verweisen, als remote und verlangt
identitätstragende WebSocket-Authentifizierung über `appServer.authToken` oder einen
`Authorization`-Header. `appServer.authToken` und jeder `appServer.headers.*`-Wert
können ein SecretInput sein; die Secrets-Laufzeit löst SecretRefs und Env-Kurzformen auf,
bevor OpenClaw die App-Server-Startoptionen erstellt, und nicht aufgelöste
strukturierte SecretRefs führen zu einem Fehler, bevor Token oder Header gesendet werden. Wenn native
Codex-Plugins konfiguriert sind, verwendet OpenClaw die Plugin-Steuerungsebene des verbundenen App-Servers,
um diese Plugins zu installieren oder zu aktualisieren, und aktualisiert anschließend das
App-Inventar, damit Plugin-eigene Apps für den Codex-Thread sichtbar sind. `app/list` ist
weiterhin die maßgebliche Inventar- und Metadatenquelle, aber die OpenClaw-Richtlinie
entscheidet, ob `thread/start` für eine aufgelistete, zugängliche App
`config.apps[appId].enabled = true` sendet, selbst wenn Codex sie derzeit als deaktiviert kennzeichnet. Unbekannte oder
fehlende App-IDs bleiben standardmäßig gesperrt; dieser Pfad aktiviert lediglich Marketplace-Plugins
über `plugin/install` und aktualisiert das Inventar. Verbinden Sie OpenClaw nur mit
Remote-App-Servern, denen Sie zutrauen, von OpenClaw verwaltete Plugin-Installationen
und Aktualisierungen des App-Inventars zu akzeptieren.

## Genehmigungs- und Sandbox-Modi

Lokale stdio-App-Server-Sitzungen verwenden standardmäßig den YOLO-Modus:
`approvalPolicy: "never"`, `approvalsReviewer: "user"` und
`sandbox: "danger-full-access"`. Diese vertrauenswürdige Haltung für lokale Operatoren ermöglicht
unbeaufsichtigten OpenClaw-Durchläufen und Heartbeats Fortschritte ohne native
Genehmigungsaufforderungen, die niemand beantworten kann.

Wenn die lokale Systemanforderungsdatei von Codex implizite YOLO-Genehmigungs-,
Prüfer- oder Sandbox-Werte nicht zulässt, behandelt OpenClaw den impliziten Standard
stattdessen als Guardian und wählt zulässige Guardian-Berechtigungen aus. `tools.exec.mode: "auto"`
erzwingt ebenfalls von Guardian geprüfte Codex-Genehmigungen und behält unsichere
veraltete Überschreibungen durch `approvalPolicy: "never"` oder `sandbox: "danger-full-access"` nicht bei;
setzen Sie `tools.exec.mode: "full"` für eine bewusst gewählte Haltung ohne Genehmigungen.
Mit dem Hostnamen übereinstimmende `[[remote_sandbox_config]]`-Einträge in derselben
Anforderungsdatei werden bei der Entscheidung über den Sandbox-Standard berücksichtigt.

Setzen Sie `appServer.mode: "guardian"` für von Codex Guardian geprüfte Genehmigungen:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            mode: "guardian",
            serviceTier: "priority",
          },
        },
      },
    },
  },
}
```

Die Voreinstellung `guardian` wird zu `approvalPolicy: "on-request"`,
`approvalsReviewer: "auto_review"` und `sandbox: "workspace-write"` erweitert, wenn diese
Werte zulässig sind. Einzelne Richtlinienfelder überschreiben `mode`. Der ältere
Prüferwert `guardian_subagent` wird weiterhin als Kompatibilitätsalias akzeptiert,
neue Konfigurationen sollten jedoch `auto_review` verwenden.

Wenn eine OpenClaw-Sandbox aktiv ist, wird der lokale Codex-App-Server-Prozess weiterhin
auf dem Gateway-Host ausgeführt. OpenClaw deaktiviert daher für diesen Durchlauf den nativen Code Mode von Codex,
MCP-Server des Benutzers und die App-gestützte Plugin-Ausführung, anstatt
die Host-seitige Sandbox von Codex als gleichwertig mit dem OpenClaw-Sandbox-Backend
zu behandeln. Shell-Zugriff wird über von der OpenClaw-Sandbox gestützte dynamische Tools
wie `sandbox_exec` und `sandbox_process` bereitgestellt, wenn die normalen Exec-/Prozess-Tools
verfügbar sind.

<Note>
Auf Docker-gestützten OpenClaw-Sandbox-Hosts (`agents.defaults.sandbox.mode` ist auf
ein Docker-Backend gesetzt) prüft `openclaw doctor`, ob der Host die
unprivilegierten Benutzer-Namespaces (und, wenn der Netzwerk-Egress der Docker-Sandbox deaktiviert ist,
Netzwerk-Namespaces) zulässt, die das verschachtelte Codex-`bwrap` für die
`workspace-write`-Shell-Ausführung innerhalb des Sandbox-Containers benötigt. Eine fehlgeschlagene Prüfung zeigt sich
auf Ubuntu-/AppArmor-Hosts üblicherweise als `bwrap: setting up uid map: Permission denied` oder
`bwrap: loopback: Failed RTM_NEWADDR: Operation not permitted`.
Korrigieren Sie die gemeldete Host-Namespace-Richtlinie für den OpenClaw-
Dienstbenutzer und starten Sie das Gateway neu; bevorzugen Sie ein auf den
Dienstprozess beschränktes AppArmor-Profil gegenüber dem hostweiten
`kernel.apparmor_restrict_unprivileged_userns=0`-Fallback und gewähren Sie keine
weiterreichenden Docker-Container-Berechtigungen, nur um die Anforderungen des verschachtelten `bwrap` zu erfüllen.
</Note>

## Native Ausführung in der Sandbox

Der stabile Standard ist „Fail-Closed“: Eine aktive OpenClaw-Sandbox deaktiviert native
Codex-Ausführungsoberflächen, die andernfalls auf dem Host des Codex-App-Servers
ausgeführt würden. Verwenden Sie `appServer.experimental.sandboxExecServer: true` nur, wenn Sie
die Remote-Umgebungsunterstützung von Codex mit dem Sandbox-Backend von OpenClaw ausprobieren möchten.
Dieser Vorschaupfad funktioniert mit jeder unterstützten Codex-App-Server-Version.

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            experimental: {
              sandboxExecServer: true,
            },
          },
        },
      },
    },
  },
}
```

Wenn das Flag aktiviert und die aktuelle OpenClaw-Sitzung in einer Sandbox ausgeführt wird, startet OpenClaw
einen lokalen Loopback-Exec-Server, der von der aktiven Sandbox gestützt wird, registriert ihn
beim Codex-App-Server und startet den Codex-Thread und -Durchlauf mit dieser
OpenClaw-eigenen Umgebung. Wenn der App-Server die Umgebung nicht registrieren kann,
schlägt der Durchlauf standardmäßig fehl, anstatt stillschweigend auf die Host-Ausführung zurückzufallen.

Dieser Vorschaupfad ist nur lokal verfügbar. Ein Remote-WebSocket-App-Server kann den
Loopback-Exec-Server nur erreichen, wenn er auf demselben Host ausgeführt wird, daher lehnt OpenClaw
diese Kombination ab.

## Authentifizierungs- und Umgebungsisolation

Im standardmäßigen agentenspezifischen Home-Verzeichnis wird die Authentifizierung in dieser Reihenfolge ausgewählt:

1. Ein explizites OpenClaw-Codex-Authentifizierungsprofil für den Agenten.
2. Das bestehende Konto des App-Servers im Codex-Home-Verzeichnis dieses Agenten.
3. Nur für lokale stdio-App-Server-Starts: `CODEX_API_KEY`, anschließend
   `OPENAI_API_KEY`, wenn kein App-Server-Konto vorhanden ist und weiterhin eine
   OpenAI-Authentifizierung erforderlich ist.

Wenn OpenClaw ein Codex-Authentifizierungsprofil im Stil eines ChatGPT-Abonnements erkennt (OAuth oder
Anmeldedatentyp „Token“), entfernt es `CODEX_API_KEY` und `OPENAI_API_KEY` aus
dem erzeugten Codex-Kindprozess. Dadurch bleiben API-Schlüssel auf Gateway-Ebene
für Einbettungen oder direkte OpenAI-Modelle verfügbar, ohne dass native Durchläufe des
Codex-App-Servers versehentlich über die API abgerechnet werden.

Explizite Codex-API-Schlüsselprofile und der lokale stdio-Fallback auf Env-Schlüssel verwenden
die App-Server-Anmeldung statt einer geerbten Kindprozessumgebung. WebSocket-App-Server-
Verbindungen erhalten keinen Fallback auf API-Schlüssel aus der Gateway-Umgebung; verwenden Sie ein explizites
Authentifizierungsprofil oder das eigene Konto des Remote-App-Servers.

Stdio-App-Server-Starts erben standardmäßig die Prozessumgebung von OpenClaw.
OpenClaw verwaltet die Kontobrücke des Codex-App-Servers und setzt `CODEX_HOME` auf ein
agentenspezifisches Verzeichnis unter dem OpenClaw-Status dieses Agenten. Dadurch bleiben Codex-
Konfiguration, Konten, Plugin-Cache/-Daten und Thread-Status auf den OpenClaw-
Agenten beschränkt, statt aus dem persönlichen `~/.codex`-Home-Verzeichnis des Operators einzusickern.

Setzen Sie `appServer.homeScope: "user"`, um den nativen Codex-Status mit Codex
Desktop und der CLI zu teilen. Dieser lokale Benutzer-Home-Modus unterstützt verwaltetes stdio und
expliziten Unix-Transport. Er verwendet `$CODEX_HOME`, wenn gesetzt, andernfalls `~/.codex`,
einschließlich nativer Authentifizierung, Konfiguration, Plugins und Threads.
OpenClaw überspringt seine Authentifizierungsprofilbrücke für den App-Server. Verifizierte
Eigentümerdurchläufe können `codex_threads` verwenden, um diese Threads aufzulisten (mit einem optionalen
`search`-Filter), zu lesen, zu forken, umzubenennen, zu archivieren und die Archivierung aufzuheben.
Forken Sie einen Thread, bevor Sie ihn in OpenClaw fortsetzen; unabhängige Codex-Prozesse
koordinieren keine gleichzeitigen Schreibzugriffe auf denselben Thread.

Diese Zustimmung zu `homeScope` gilt für gewöhnliche Harness-Sitzungen. Ein über
Codex Sessions erstellter Chat verwendet stattdessen seine private Überwachungsverbindung, wodurch
die Authentifizierungs- und Provider-Konfiguration der nativen Verbindung für den
kanonischen Branch und zukünftige Fortsetzungen erhalten bleibt.

In einem modellgebundenen überwachten Chat kann `codex_threads` weder einen anderen
Fork anhängen noch den an den Chat gebundenen nativen Thread archivieren. Auflisten und
reines Lesen von Metadaten bleiben verfügbar. Das Lesen von Rohtranskripten erfordert `allowRawTranscripts`;
wenn diese Option deaktiviert ist, wird auch die Listensuche abgelehnt, da die native Suche
Übereinstimmungen in Transkriptvorschauen finden kann. Umbenennen, Aufheben der Archivierung, abgetrenntes Forken und
Archivieren eines nicht zugehörigen Threads, der keinem anderen OpenClaw-Chat gehört, erfordern
`allowWriteControls`. Keine der Optionen umgeht eine gesperrte Bindung.

OpenClaw schreibt `HOME` bei normalen lokalen App-Server-Starts nicht um.
Von Codex ausgeführte Unterprozesse wie `openclaw`, `gh`, `git`, Cloud-CLIs und Shell-
Befehle sehen das normale Prozess-Home-Verzeichnis und können Benutzer-Home-Konfigurationen und
Token finden. Codex kann außerdem `$HOME/.agents/skills` und
`$HOME/.agents/plugins/marketplace.json` erkennen; diese `.agents`-Erkennung wird
bewusst mit dem Operator-Home-Verzeichnis geteilt und ist vom isolierten
`~/.codex`-Status getrennt.

Im standardmäßigen Agentenbereich werden OpenClaw-Plugins und Momentaufnahmen von OpenClaw-Skills
weiterhin über die eigene Plugin-Registry und den Skills-Loader von OpenClaw bereitgestellt; persönliche
Codex-`~/.codex`-Assets dagegen nicht. Wenn Sie nützliche Codex-CLI-Skills oder
Plugins aus einem Codex-Home-Verzeichnis haben, die Teil eines isolierten OpenClaw-
Agenten werden sollen, erfassen Sie diese explizit:

```bash
openclaw migrate codex --dry-run
openclaw migrate apply codex --yes
```

Wenn eine Bereitstellung zusätzliche Umgebungsisolation benötigt, fügen Sie diese Variablen
zu `appServer.clearEnv` hinzu:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            clearEnv: ["CODEX_API_KEY", "OPENAI_API_KEY"],
          },
        },
      },
    },
  },
}
```

`appServer.clearEnv` wirkt sich nur auf den erzeugten Kindprozess des Codex-App-Servers aus.
OpenClaw entfernt `CODEX_HOME` und `HOME` während der Normalisierung des lokalen Starts
aus dieser Liste: `CODEX_HOME` verweist weiterhin auf den ausgewählten Agenten- oder Benutzerbereich,
und `HOME` wird weiterhin geerbt, sodass Unterprozesse den normalen Status des Benutzer-Home-Verzeichnisses verwenden können.

## Dynamische Tools

Dynamische Codex-Tools verwenden standardmäßig das Laden gemäß `searchable` und werden unter dem
Namespace `openclaw` mit `deferLoading: true` bereitgestellt. OpenClaw stellt normalerweise keine
dynamischen Tools bereit, die native Arbeitsbereichsoperationen von Codex oder
die eigene Toolsuche von Codex duplizieren:

- `read`
- `write`
- `edit`
- `apply_patch`
- `exec`
- `process`
- `update_plan`
- `tool_call`
- `tool_describe`
- `tool_search`
- `tool_search_code`

Wenn eine endliche Laufzeit-Zulassungsliste den nativen Code Mode deaktiviert, sendet OpenClaw eine
leere Auswahl der Ausführungsumgebung. In diesem direkten Fall ohne Sandbox
behält OpenClaw seine richtliniengefilterten Tools `exec` und `process` als Shell-
Fallback bei. Laufzeit-Zulassungslisten und `codexDynamicToolsExclude` gelten weiterhin.

Die meisten verbleibenden OpenClaw-Integrationstools, etwa für Messaging, Medien, Cron,
Browser, Nodes, Gateway, `heartbeat_respond` und `web_search`, sind
über die Codex-Tool-Suche unter diesem Namespace verfügbar. Dadurch bleibt der anfängliche
Modellkontext kleiner. Eine kleine Gruppe von Tools bleibt unabhängig von
`codexDynamicToolsLoading` direkt aufrufbar, da die Codex-Tool-Suche nicht verfügbar sein oder
nur ein Connector-Universum auflösen kann: `agents_list`, `sessions_spawn` und
`sessions_yield`. Entwickleranweisungen lenken normale Codex-Subagenten
für Codex-native Subagentenarbeit weiterhin zu nativem `spawn_agent`, während
`sessions_spawn` für eine explizite Delegation an OpenClaw oder ACP verfügbar bleibt.
Quellantworten, die ausschließlich das Nachrichtentool verwenden, bleiben ebenfalls direkt, da dies ein
Vertrag zur Ablaufsteuerung eines Turns ist.

Der Codex-Code-Modus stellt Ergebnisse generischer dynamischer OpenClaw-Tools als Text dar. Parsen Sie ein
JSON-Ergebnis, bevor Sie Felder auslesen. Verschachtelte dynamische Aufrufe werden von der
Codex-Laufzeit serialisiert, daher übermittelt `Promise.all` sie nicht gleichzeitig; verwenden Sie beim
Starten untergeordneter Collector-Prozesse eine begrenzte sequenzielle Startschleife.

Mit `catalogMode: "direct-only"` gekennzeichnete Tools, einschließlich des OpenClaw-Tools
`computer`, werden unter `openclaw_direct` gruppiert. OpenClaw fügt diesen Namespace
der Liste `code_mode.direct_only_tool_namespaces` von Codex hinzu, ohne
vom Operator bereitgestellte Einträge zu ersetzen. Codex stellt diese Tools daher in normalen und ausschließlich
im Code-Modus ausgeführten Threads als `DirectModelOnly` bereit, statt sie
durch verschachtelte `tools.*`-Aufrufe des Code-Modus zu leiten. Diese Grenze ist für
Ergebnisse mit Bildern erforderlich: Die verschachtelte Serialisierung im Code-Modus reduziert die Bildausgabe auf
Text, wodurch der für die nächste Computeraktion benötigte Screenshot verloren ginge.

Setzen Sie `codexDynamicToolsLoading: "direct"` nur, wenn Sie eine Verbindung zu einem benutzerdefinierten
Codex-App-Server herstellen, der zurückgestellte dynamische Tools nicht durchsuchen kann, oder wenn Sie
die vollständige Tool-Nutzlast debuggen.

## Zeitüberschreitungen

OpenClaw-eigene dynamische Tool-Aufrufe werden unabhängig von
`appServer.requestTimeoutMs` begrenzt. Jede Codex-Anfrage `item/tool/call` verwendet die
erste verfügbare Zeitüberschreitung in dieser Reihenfolge:

- Ein positives aufrufbezogenes Argument `timeoutMs`.
- Für `image_generate`: `agents.defaults.mediaModels.image.timeoutMs`.
- Für `image_generate` ohne konfigurierte Zeitüberschreitung gilt der Standardwert von 120 Sekunden
  für die Bilderzeugung.
- Für das Medienanalyse-Tool `image`: der Wert `timeoutSeconds`
  des ausgewählten bildfähigen Eintrags `tools.media.models[]`, umgerechnet in Millisekunden,
  oder der Medienstandardwert von 60 Sekunden. Bei der Bildanalyse gilt dies für die
  Anfrage selbst und wird nicht durch vorherige Vorbereitungsarbeiten reduziert.
- Für das Tool `message`: ein festes äußeres Zeitbudget von 600 Sekunden, das die Gateway-Zustellung und den begrenzten Abgleich für denselben Schlüssel abdeckt.
- Der Standardwert von 90 Sekunden für dynamische Tools.

Dieser Watchdog ist das äußere Budget für dynamisches `item/tool/call`. Provider-spezifische
Anfragezeitüberschreitungen laufen innerhalb dieses Aufrufs und behalten ihre eigene Zeitüberschreitungssemantik.
Budgets für dynamische Tools sind auf 600000 ms begrenzt. `agents_wait` fügt 30000 ms
äußere Abschlusskulanz hinzu, und der App-Server-Client lässt 660000 ms zu, damit
das strukturierte Warteergebnis Codex erreichen kann. Bei einer Zeitüberschreitung bricht OpenClaw das Tool-Signal
ab, sofern dies unterstützt wird, und gibt eine fehlgeschlagene Antwort des dynamischen Tools an Codex zurück, sodass
der Turn fortgesetzt werden kann, statt die Sitzung in `processing` zu belassen.

Nachdem Codex einen Turn angenommen und OpenClaw auf eine turnbezogene
App-Server-Anfrage geantwortet hat, erwartet das Harness, dass Codex im aktuellen Turn Fortschritte macht
und den nativen Turn schließlich mit `turn/completed` beendet. Wenn der
App-Server für `appServer.turnCompletionIdleTimeoutMs` inaktiv bleibt, versucht OpenClaw
nach bestem Bemühen, den Codex-Turn zu unterbrechen, zeichnet eine diagnostische Zeitüberschreitung auf und
gibt die OpenClaw-Sitzungsspur frei, damit nachfolgende Chatnachrichten nicht
hinter einem veralteten nativen Turn eingereiht werden.

Die meisten nicht abschließenden Benachrichtigungen für denselben Turn deaktivieren diesen kurzen Watchdog,
weil Codex nachgewiesen hat, dass der Turn noch aktiv ist. Tool-Übergaben verwenden ein längeres
Inaktivitätsbudget nach dem Tool: nachdem OpenClaw eine Antwort `item/tool/call` zurückgibt,
nachdem native Tool-Elemente wie `commandExecution` abgeschlossen sind, nach unverarbeiteten
`custom_tool_call_output`-Abschlüssen sowie nach unverarbeitetem Assistentenfortschritt,
Argumentationsabschlüssen oder Argumentationsfortschritt nach einem Tool. Die Schutzfunktion verwendet
`appServer.postToolRawAssistantCompletionIdleTimeoutMs`, wenn dies konfiguriert ist, und
standardmäßig andernfalls fünf Minuten. Dasselbe Budget nach einem Tool verlängert auch
den Fortschritts-Watchdog für das stille Synthesefenster, bevor Codex das
nächste Ereignis des aktuellen Turns ausgibt. Auf Argumentationsabschlüsse, Abschlüsse von
Kommentar-`agentMessage` und unverarbeiteten Argumentations- oder Assistentenfortschritt vor einem Tool
kann eine automatische abschließende Antwort folgen; deshalb verwenden sie die Antwortschutzfunktion
nach Fortschritt, statt die Sitzungsspur sofort freizugeben. Nur abgeschlossene abschließende bzw.
nicht als Kommentar eingestufte `agentMessage`-Elemente und unverarbeitete Assistentenabschlüsse vor einem Tool
aktivieren die Freigabe nach Assistentenausgabe: Wenn Codex anschließend ohne `turn/completed`
inaktiv bleibt, versucht OpenClaw nach bestem Bemühen, den nativen Turn zu unterbrechen, und gibt die
Sitzungsspur frei. Wiederholungssichere Fehler des stdio-App-Servers, einschließlich Zeitüberschreitungen
bei Inaktivität bis zum Turn-Abschluss ohne Hinweise auf Assistenten-, Tool-, aktive Element- oder
Nebeneffektaktivität, werden einmal mit einem neuen App-Server-Versuch wiederholt. Unsichere
Zeitüberschreitungen setzen den blockierten App-Server-Client dennoch außer Betrieb und geben die
OpenClaw-Sitzungsspur frei. Sie löschen außerdem die veraltete Bindung des nativen Threads,
statt automatisch wiederholt zu werden. Zeitüberschreitungen der Abschlussüberwachung zeigen
Codex-spezifischen Zeitüberschreitungstext an: Wiederholungssichere Fälle weisen darauf hin, dass die Antwort
möglicherweise unvollständig ist, während unsichere Fälle den Benutzer anweisen, vor einem erneuten Versuch
den aktuellen Zustand zu prüfen. Öffentliche Zeitüberschreitungsdiagnosen enthalten strukturelle Felder
wie die Methode der letzten App-Server-Benachrichtigung, ID/Typ/Rolle des unverarbeiteten
Assistentenantwortelements, Anzahl aktiver Anfragen und Elemente sowie den Zustand der
aktivierten Überwachung. Wenn die letzte Benachrichtigung ein unverarbeitetes Assistentenantwortelement ist,
enthalten sie außerdem eine begrenzte Vorschau des Assistententexts. Sie enthalten keine
unverarbeiteten Prompt- oder Tool-Inhalte.

## Modellerkennung

Standardmäßig fragt das Codex-Plugin den App-Server nach verfügbaren Modellen. Die
Modellverfügbarkeit liegt in der Verantwortung des Codex-App-Servers, sodass sich die Liste ändern kann, wenn
OpenClaw die gebündelte Version `@openai/codex` aktualisiert oder wenn eine Bereitstellung
`appServer.command` auf eine andere Codex-Binärdatei verweist. Die Verfügbarkeit kann außerdem
kontobezogen sein. Verwenden Sie `/codex models` auf einem laufenden Gateway, um den aktuellen
Katalog für dieses Harness und Konto anzuzeigen.

Wenn die Erkennung fehlschlägt oder eine Zeitüberschreitung auftritt, verwendet OpenClaw einen gebündelten Ausweichkatalog:

| Modell-ID      | Anzeigename   | Argumentationsstufen      |
| -------------- | ------------- | ------------------------- |
| `gpt-5.5`      | gpt-5.5       | low, medium, high, xhigh |
| `gpt-5.4-mini` | GPT-5.4-Mini | low, medium, high, xhigh |

<Note>
Das aktuell gebündelte Harness ist `@openai/codex` `0.145.0`. Eine Abfrage mit `model/list`
an diesen gebündelten App-Server gab folgende öffentliche Auswahlzeilen zurück:

| Modell-ID       | Eingabemodalitäten | Argumentationsstufen                  |
| --------------- | ------------------- | ------------------------------------- |
| `gpt-5.6-sol`   | Text, Bild          | low, medium, high, xhigh, max, ultra |
| `gpt-5.6-terra` | Text, Bild          | low, medium, high, xhigh, max, ultra |
| `gpt-5.6-luna`  | Text, Bild          | low, medium, high, xhigh, max        |
| `gpt-5.5`       | Text, Bild          | low, medium, high, xhigh             |
| `gpt-5.2`       | Text, Bild          | low, medium, high, xhigh             |

Der App-Server-Katalog kann `ultra` melden; die Argumentationssteuerung von OpenClaw
stellt derzeit Stufen bis `max` bereit.

Aktuelle Auswahlzeilen sind kontobezogen und können sich mit dem Konto, dem Codex-Katalog
oder der gebündelten Version ändern; führen Sie `/codex models` aus, um die aktuelle Liste zu erhalten,
statt sich auf eine Momentaufnahme in einer Tabelle zu verlassen. Ausgeblendete Modelle können ebenfalls im
App-Server-Katalog für interne oder spezialisierte Abläufe erscheinen, ohne normale
Auswahlmöglichkeiten im Modellwähler zu sein.
</Note>

Passen Sie die Erkennung unter `plugins.entries.codex.config.discovery` an:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          discovery: {
            enabled: true,
            timeoutMs: 2500,
          },
        },
      },
    },
  },
}
```

Deaktivieren Sie die Erkennung, wenn beim Start keine Abfrage von Codex erfolgen und ausschließlich
der Ausweichkatalog verwendet werden soll:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          discovery: {
            enabled: false,
          },
        },
      },
    },
  },
}
```

## Bootstrap-Dateien des Arbeitsbereichs

Codex verarbeitet `AGENTS.md` selbst über die native Erkennung der Projektdokumentation.
OpenClaw schreibt keine synthetischen Codex-Projektdokumentationsdateien und ist für Persona-Dateien
nicht von Codex-Ausweichdateinamen abhängig, da Codex-Ausweichwerte nur gelten, wenn
`AGENTS.md` fehlt.

Für die Gleichwertigkeit des OpenClaw-Arbeitsbereichs leitet das Codex-Harness die anderen
Bootstrap-Dateien als Entwickleranweisungen weiter, jedoch nicht auf identische Weise:

- `TOOLS.md` wird als **geerbte** Codex-Entwickleranweisung weitergeleitet, sodass
  native Codex-Subagenten, die während des Turns gestartet werden, sie ebenfalls sehen.
- `SOUL.md`, `IDENTITY.md` und `USER.md` werden als **turnbezogene**
  Zusammenarbeitsanweisungen weitergeleitet. Native Codex-Subagenten erben sie nicht,
  wodurch verhindert wird, dass Subagenten-Turns die Persona und das
  Benutzerprofil des übergeordneten Agenten übernehmen.
- Die kompakte Liste der geladenen OpenClaw-Skills wird ebenfalls als turnbezogene
  Entwickleranweisung zur Zusammenarbeit weitergeleitet, sodass native Codex-Subagenten
  auch sie nicht erben.
- Der Inhalt von `HEARTBEAT.md` wird nicht eingefügt; Heartbeat-Turns erhalten einen
  Hinweis im Zusammenarbeitsmodus, die Datei zu lesen, wenn sie vorhanden und
  nicht leer ist.
- Der Inhalt von `MEMORY.md` aus dem konfigurierten Agentenarbeitsbereich wird nicht in
  die Eingabe nativer Codex-Turns eingefügt, wenn für diesen
  Arbeitsbereich Speicher-Tools verfügbar sind; wenn er vorhanden ist, fügt das Harness den turnbezogenen
  Entwickleranweisungen zur Zusammenarbeit einen kurzen Hinweis auf den Arbeitsbereichsspeicher hinzu, und Codex
  sollte `memory_search` oder `memory_get` verwenden, wenn dauerhafter Speicher relevant ist.
  Wenn Tools deaktiviert sind, die Speichersuche nicht verfügbar ist oder der aktive
  Arbeitsbereich vom Agentenspeicher-Arbeitsbereich abweicht, verwendet `MEMORY.md` stattdessen den
  normalen begrenzten Turn-Kontextpfad.
- `BOOTSTRAP.md` wird, sofern vorhanden, als Referenzkontext für OpenClaw-Turn-Eingaben
  weitergeleitet.

## Umgebungsüberschreibungen

Umgebungsüberschreibungen bleiben für lokale Tests verfügbar:

- `OPENCLAW_CODEX_APP_SERVER_BIN`
- `OPENCLAW_CODEX_APP_SERVER_ARGS`
- `OPENCLAW_CODEX_APP_SERVER_MODE=yolo|guardian`
- `OPENCLAW_CODEX_APP_SERVER_APPROVAL_POLICY`
- `OPENCLAW_CODEX_APP_SERVER_SANDBOX`

`OPENCLAW_CODEX_APP_SERVER_BIN` umgeht die verwaltete Binärdatei, wenn
`appServer.command` nicht gesetzt ist.

`OPENCLAW_CODEX_APP_SERVER_GUARDIAN=1` wurde entfernt. Verwenden Sie stattdessen
`plugins.entries.codex.config.appServer.mode: "guardian"` oder
`OPENCLAW_CODEX_APP_SERVER_MODE=guardian` für einmalige lokale Tests. Für
reproduzierbare Bereitstellungen wird die Konfiguration bevorzugt, da dadurch das Plugin-Verhalten in
derselben geprüften Datei wie die übrige Einrichtung des Codex-Harness verbleibt.

## Verwandte Themen

- [Codex-Harness](/de/plugins/codex-harness)
- [Codex-Harness-Laufzeit](/de/plugins/codex-harness-runtime)
- [Codex-Überwachung](/de/plugins/codex-supervision)
- [Native Codex-Plugins](/de/plugins/codex-native-plugins)
- [Codex-Computersteuerung](/de/plugins/codex-computer-use)
- [OpenAI-Provider](/de/providers/openai)
- [Konfigurationsreferenz](/de/gateway/configuration-reference)
