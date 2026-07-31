---
read_when:
    - Sie möchten das offizielle Codex-App-Server-Harness verwenden
    - Sie benötigen Konfigurationsbeispiele für die Codex-Harness.
    - Sie möchten, dass reine Codex-Bereitstellungen fehlschlagen, statt auf OpenClaw zurückzufallen
summary: Führen Sie eingebettete OpenClaw-Agenten-Turns über das offizielle Codex-App-Server-Harness aus
title: Codex-Harness
x-i18n:
    generated_at: "2026-07-26T17:54:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e016a1689af65c5520d529ce22a87bd25ee29369f7aedca77b27f943a7f21b0f
    source_path: plugins/codex-harness.md
    workflow: 16
---

Das offizielle Plugin `codex` führt eingebettete OpenAI-Agent-Durchläufe über den Codex
app-server statt über das integrierte OpenClaw-Harness aus. Codex verwaltet die
untergeordnete Agent-Sitzung: natives Fortsetzen von Threads, native Fortsetzung von Tools,
native Compaction und app-server-Ausführung. OpenClaw verwaltet weiterhin Chat-
Kanäle, Sitzungsdateien, Modellauswahl, dynamische OpenClaw-Tools, Genehmigungen,
Medienübermittlung und die sichtbare Transkriptspiegelung.

Verwenden Sie kanonische OpenAI-Modellreferenzen wie `openai/gpt-5.6-sol`. Konfigurieren Sie keine
veralteten Codex-GPT-Referenzen; legen Sie die Authentifizierungsreihenfolge für OpenAI-Agenten unter `auth.order.openai` ab.
Veraltete Codex-Authentifizierungsprofil-IDs und Einträge der veralteten Codex-Authentifizierungsreihenfolge werden
durch `openclaw doctor --fix` repariert.

Wenn die Provider-/Modell-Laufzeitrichtlinie nicht festgelegt oder auf `auto` gesetzt ist, wählt das Präfix `openai/*` allein
dieses Harness niemals aus. OpenAI darf Codex nur dann implizit auswählen, wenn eine
exakte offizielle HTTPS-Route für Platform Responses oder ChatGPT Responses ohne
benutzerdefinierte Anfrageüberschreibung verwendet wird. Siehe
[Implizite Agent-Laufzeit von OpenAI](/de/providers/openai#implicit-agent-runtime).
Wenn Codex die Authentifizierung verwaltet, bevor das Routing zwischen Platform und ChatGPT bekannt ist, verlangt OpenClaw
weiterhin, dass jede mögliche Route Codex-Kompatibilität deklariert. Allein die native
Authentifizierungsverwaltung umgeht diese Routenprüfung niemals.

Wenn keine OpenClaw-Sandbox aktiv ist, startet OpenClaw Codex-app-server-Threads
mit aktiviertem nativen Codex-Codemodus (code-mode-only bleibt standardmäßig deaktiviert), sodass
native Arbeitsbereichs-/Codefunktionen neben dynamischen OpenClaw-Tools verfügbar bleiben,
die über die app-server-Brücke `item/tool/call` geleitet werden. Eine
aktive OpenClaw-Sandbox oder eingeschränkte Tool-Richtlinie deaktiviert den nativen Codemodus
vollständig, sofern Sie nicht ausdrücklich den experimentellen Sandbox-exec-server-Pfad aktivieren.

Mit der Standardeinstellung `tools.exec.host: "auto"` und ohne aktive OpenClaw-Sandbox
erhält Codex außerdem die Tools `node_exec` und `node_process` für Befehle auf gekoppelten
Nodes. Die native Shell verbleibt auf Host und Arbeitsbereich des Codex-app-server
(bei der standardmäßigen stdio-Bereitstellung lokal zum Gateway); `node_exec` wählt eine Node anhand
ihres Namens oder ihrer ID aus und setzt weiterhin die Node-Genehmigungsrichtlinie von OpenClaw durch. Wenn eine endliche
Laufzeit-Zulassungsliste den nativen Codemodus deaktiviert und der Durchlauf dadurch keine
Ausführungsumgebung mehr hat, hält OpenClaw stattdessen seine richtliniengefilterten Tools `exec` und `process`
für die direkte Ausführung ohne Sandbox verfügbar.

Diese native Codex-Funktion unterscheidet sich vom
[OpenClaw-Codemodus](/de/tools/code-mode), einer optionalen QuickJS-WASI-Laufzeit
für allgemeine OpenClaw-Durchläufe mit einer anderen Eingabeform `exec`. Einen Überblick über die
umfassendere Aufteilung zwischen Modell, Provider und Laufzeit finden Sie unter
[Agent-Laufzeiten](/de/concepts/agent-runtimes): `openai/gpt-5.6-sol` ist die Modell-
referenz, `codex` ist die Laufzeit und Telegram, Discord, Slack oder ein anderer
Kanal ist die Kommunikationsoberfläche.

## Anforderungen

- Das offizielle Plugin `@openclaw/codex` muss installiert sein. Nehmen Sie `codex` in
  `plugins.allow` auf, wenn Ihre Konfiguration eine Zulassungsliste verwendet.
- Ein stabiler Codex-app-server von `0.143.0` bis `0.145.0`. Das Plugin verwaltet standardmäßig eine kompatible
  Binärdatei, sodass sich ein Befehl `codex` unter `PATH` nicht auf den normalen
  Start auswirkt.
- Codex-Authentifizierung über `openclaw models auth login --provider openai`, ein
  bereits im Codex-Home des Agenten vorhandenes app-server-Konto oder ein
  explizites Codex-API-Schlüssel-Authentifizierungsprofil.

Informationen zu Authentifizierungspriorität, Umgebungsisolierung, benutzerdefinierten app-server-Befehlen,
Modellerkennung und der vollständigen Liste der Konfigurationsfelder finden Sie in der
[Codex-Harness-Referenz](/de/plugins/codex-harness-reference).

## Schnellstart

Installieren Sie das offizielle Plugin und melden Sie sich anschließend mit Codex OAuth an:

```bash
openclaw plugins install @openclaw/codex
openclaw models auth login --provider openai
```

Aktivieren Sie das Plugin `codex` und wählen Sie ein OpenAI-Agent-Modell aus:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.6-sol",
    },
  },
}
```

Wenn Ihre Konfiguration `plugins.allow` verwendet, fügen Sie dort auch `codex` hinzu:

```json5
{
  plugins: {
    allow: ["codex"],
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
}
```

Starten Sie das Gateway nach einer Änderung der Plugin-Konfiguration neu. Wenn ein Chat bereits eine
Sitzung hat, führen Sie zuerst `/new` oder `/reset` aus, damit der nächste Durchlauf das Harness
anhand der aktuellen Konfiguration bestimmt.

## Threads mit Codex Desktop und der CLI teilen

Die Standardeinstellung `appServer.homeScope: "agent"` isoliert jeden OpenClaw-Agenten vom
nativen Codex-Zustand des Betreibers. Damit ein Eigentümer dieselben nativen Threads untersuchen und verwalten kann,
die in Codex Desktop und der Codex CLI angezeigt werden, aktivieren Sie das
Codex-Home des Benutzers:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            homeScope: "user",
          },
        },
      },
    },
  },
}
```

Der Benutzer-Home-Modus unterstützt einen lokal verwalteten stdio-Prozess oder den gemeinsam genutzten Unix-Socket-
Transport. Er verwendet `$CODEX_HOME`, wenn dies festgelegt ist, andernfalls `~/.codex`, einschließlich
der nativen Codex-Authentifizierung, Konfiguration, Plugins und des Thread-Speichers dieses Homes. OpenClaw
injiziert kein OpenClaw-Authentifizierungsprofil in diesen app-server.

Durchläufe des Eigentümers erhalten das Tool `codex_threads`: native Threads auflisten, durchsuchen, lesen, forken, umbenennen,
archivieren und wiederherstellen. Forken Sie einen Thread, um ihn in
OpenClaw fortzusetzen; der Fork wird an die aktuelle OpenClaw-Sitzung angehängt und bleibt
für andere native Codex-Clients sichtbar. Für die Archivierung ist eine ausdrückliche
Bestätigung erforderlich, dass der Thread an anderer Stelle geschlossen ist. Wenn außerdem die Überwachung
aktiviert ist, benötigen Transkriptfelder und Änderungen die entsprechende Aktivierung von
`supervision.allowRawTranscripts` oder `supervision.allowWriteControls`.

Setzen Sie denselben Thread nicht gleichzeitig über unabhängig verwaltete
stdio-App-Server fort und schreiben Sie nicht gleichzeitig darauf. Codex koordiniert aktive Schreibzugriffe innerhalb eines App Servers, nicht
über separate Prozesse hinweg. Das Forken ist der sichere Koexistenzpfad für gewöhnliche
stdio-Sitzungen im Benutzer-Home.

`appServer.homeScope: "user"` allein steuert nicht den Flottenkatalog. Die native
Sitzungserkennung ist aktiviert, solange das Plugin aktiv ist; setzen Sie
`sessionCatalog.enabled: false`, um sie aus der OpenClaw-Seitenleiste zu entfernen, ohne
Codex zu deaktivieren. Der Katalog verwendet eine separate Überwachungsverbindung; ohne
explizite Verbindungseinstellungen für `appServer` verwendet diese Verbindung standardmäßig verwaltetes
stdio im Benutzer-Home, während das gewöhnliche Harness agentenspezifisch bleibt. Explizite
Einstellungen für `appServer` werden von beiden Pfaden berücksichtigt. Setzen Sie `homeScope: "user"`
wie oben ausdrücklich, wenn auch das gewöhnliche Harness den nativen Zustand gemeinsam nutzen soll.

## Codex-Sitzungen überwachen

Dasselbe Plugin `codex` kann nicht archivierte Codex-Sitzungen auf dem Gateway-
Computer und auf ausdrücklich aktivierten gekoppelten Nodes auflisten. Eine gespeicherte oder inaktive Gateway-lokale Sitzung kann
einen modellgebundenen Chat erstellen, der ihren begrenzten, persistenten Verlauf von Benutzer und Assistent
spiegelt. Seine private Bindung verwendet die Überwachungsverbindung für den nativen
Snapshot, den kanonischen Branch und spätere Durchläufe, während gewöhnliche Codex-Sitzungen
agentenspezifisch bleiben. Beim ersten kanonischen Start werden exakt das Modell und der Provider verwendet, die
Codex für den Snapshot-Fork zurückgibt. Bei späteren Fortsetzungen bleibt die Auswahl der
nativen Codex-Konfiguration überlassen; das äußere OpenClaw-Modell und die Fallback-Kette ersetzen
sie niemals. Gespeicherte und inaktive Zeilen können nach ausdrücklicher Bestätigung,
dass kein anderer Runner vorhanden ist, archiviert werden. Aktive Quellen können weder einen Branch erstellen noch archiviert werden; ein bestehender
überwachter Chat kann weiterhin geöffnet werden. Sitzungen auf gekoppelten Nodes bleiben auf Metadaten beschränkt.

Einrichtung, Branching-Regeln, Beschränkungen gekoppelter Nodes, Metadatenoffenlegung und Fehlerbehebung finden Sie unter [Codex-Sitzungen überwachen](/de/plugins/codex-supervision).

## Konfiguration

| Bedarf                                              | Festlegen                                                                                        | Ort                                |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------ | ---------------------------------- |
| Harness aktivieren                                  | `plugins.entries.codex.enabled: true`                                                            | OpenClaw-Konfiguration             |
| Native Codex-Sitzungserkennung ausblenden           | `plugins.entries.codex.config.sessionCatalog.enabled: false`                                     | Codex-Plugin-Konfiguration         |
| Eine per Zulassungsliste zugelassene Plugin-Installation beibehalten | `codex` in `plugins.allow` aufnehmen                                               | OpenClaw-Konfiguration             |
| Geeigneten OpenAI-Durchläufen die implizite Verwendung von Codex erlauben | Exakte offizielle HTTPS-Responses-/ChatGPT-Route, keine benutzerdefinierte Anfrageüberschreibung, Laufzeit nicht festgelegt/`auto` | OpenAI-Provider-/Modellkonfiguration |
| Mit ChatGPT/Codex OAuth anmelden                    | `openclaw models auth login --provider openai`                                                   | CLI-Authentifizierungsprofil       |
| API-Schlüssel-Backup für Codex-Durchläufe hinzufügen | `openai:*`-API-Schlüsselprofil, das in `auth.order.openai` nach der Abonnementauthentifizierung aufgeführt ist | CLI-Authentifizierungsprofil + OpenClaw-Konfiguration |
| Geschlossen fehlschlagen, wenn Codex nicht verfügbar ist | Provider oder Modell `agentRuntime.id: "codex"`                                               | OpenClaw-Modell-/Provider-Konfiguration |
| Direkten OpenAI-API-Datenverkehr verwenden          | Provider oder Modell `agentRuntime.id: "openclaw"` mit normaler OpenAI-Authentifizierung                    | OpenClaw-Modell-/Provider-Konfiguration |
| app-server-Verhalten abstimmen                      | `plugins.entries.codex.config.appServer.*`                                                       | Codex-Plugin-Konfiguration         |
| Native Codex-Plugin-Apps aktivieren                 | `plugins.entries.codex.config.codexPlugins.*`                                                    | Codex-Plugin-Konfiguration         |
| Codex Computer Use aktivieren                       | `plugins.entries.codex.config.computerUse.*`                                                     | Codex-Plugin-Konfiguration         |

Bevorzugen Sie `auth.order.openai` für eine Reihenfolge mit Abonnement zuerst und API-Schlüssel als Backup.
Vorhandene veraltete Codex-Authentifizierungsprofil-IDs und die veraltete Codex-Authentifizierungsreihenfolge sind
veralteter Zustand ausschließlich für doctor; schreiben Sie keine neuen veralteten Codex-GPT-Referenzen.

```json5
{
  auth: {
    order: {
      openai: ["openai:user@example.com", "openai:api-key-backup"],
    },
  },
}
```

Für eine Codex-kompatible effektive Route bleiben beide oben genannten Profile Kandidaten
für denselben Codex-Durchlauf. Die Profilreihenfolge wählt Anmeldedaten aus, nicht die Laufzeit.
Eine Änderung der Authentifizierungsreihenfolge macht eine benutzerdefinierte, Completions-, HTTP- oder
anfrageüberschriebene Route nicht Codex-kompatibel.

### Compaction

Setzen Sie `compaction.model` oder `compaction.provider` nicht für Codex-gestützte
Agenten. Codex führt Compaction über seinen nativen app-server-Thread-Zustand durch, daher
ignoriert OpenClaw diese lokalen Summarizer-Überschreibungen zur Laufzeit, und
`openclaw doctor --fix` entfernt sie, wenn der Agent Codex verwendet.

Lossless wird weiterhin als Kontext-Engine für Zusammenstellung, Aufnahme und
Wartung rund um Codex-Durchläufe unterstützt und über
`plugins.slots.contextEngine: "lossless-claw"` und
`plugins.entries.lossless-claw.config.summaryModel` konfiguriert, nicht über
`agents.defaults.compaction.provider`. `openclaw doctor --fix` migriert die
alte Form `compaction.provider: "lossless-claw"` in den Lossless-
Kontext-Engine-Slot, wenn Codex die aktive Laufzeit ist, doch die native Codex-Laufzeit verwaltet weiterhin
die Compaction. Das native app-server-Harness unterstützt Kontext-Engines,
die eine Zusammenstellung vor dem Prompt benötigen; generische CLI-Backends, einschließlich `codex-cli`,
stellen diese Host-Funktion nicht bereit.

Bei Codex-gestützten Agenten startet `/compact` die native Codex-app-server-
Compaction auf dem gebundenen Thread und wartet auf ihr endgültiges Ergebnis. Das gemeinsame
Budget `agents.defaults.compaction.timeoutSeconds` gilt; bei einer Zeitüberschreitung
fordert OpenClaw Codex auf, den nativen Durchlauf zu unterbrechen, und behält die threadbezogene Sperre bei,
bis die Beendigung bestätigt ist. Es greift niemals ersatzweise auf eine Kontext-Engine oder
den öffentlichen OpenAI-Summarizer zurück. Wenn die native Codex-Thread-Bindung fehlt oder
veraltet ist, schlägt der Befehl geschlossen fehl, statt unbemerkt zu einem anderen Compaction-
Backend zu wechseln.

### Direkte API mit langem Kontext

Codex-Abonnement und direkter OpenAI-API-Datenverkehr sind separate Verträge. Der
Live-Katalog von ChatGPT/Codex weist üblicherweise ein Modell-Kontextfenster von `272000` Token auf,
während OpenAI für GPT-5.5 und GPT-5.6 ein Platform-API-Fenster von `1050000` Token und
eine maximale Ausgabe von `128000` dokumentiert. Wird die vollständige Ausgabekapazität
reserviert, ergibt sich daraus ein Eingabebudget von `922000` Token. Für Anfragen mit mehr als
`272000` Eingabe-Token gilt die höhere OpenAI-Preisgestaltung für lange Kontexte.

Beginnen Sie mit einem vollständigen Codex-Modellkatalog, der mit der installierten
Codex-Version kompatibel ist. Behalten Sie für jeden direkten GPT-5.5- oder GPT-5.6-Eintrag,
der einen langen Kontext verwenden soll, den restlichen Deskriptor bei und legen Sie Folgendes fest:

```json
{
  "context_window": 922000,
  "max_context_window": 922000,
  "auto_compact_token_limit": 700000
}
```

Codex wendet seine normale Reserve von 95 % des effektiven Fensters auf den Katalogwert
`922000` an und meldet daher ungefähr `875900` nutzbare Token. Eine Compaction bei
`700000` lässt `175900` Token bis zu dieser effektiven Schutzgrenze und
`222000` bis zur Provider-sicheren Eingabekapazität frei. Dieser größere Puffer ist beabsichtigt:
Codex prüft den bereits aufgezeichneten Kontext, bevor die nächste Benutzernachricht und
Kontextaktualisierungen hinzugefügt werden. Der Schwellenwert muss daher sowohl einen großen
eingehenden Turn als auch Tools, Anweisungen, Serialisierung und den Compaction-Turn selbst abdecken.

Für die eigenständige Verwendung der Codex CLI oder Desktop-Anwendung kann ein benutzerdefinierter
Provider mit Befehlsauthentifizierung den API-Schlüssel aus einem System-Schlüsselbund oder
Secret-Manager lesen, während die normale ChatGPT-Anmeldung für Konnektoren verfügbar bleibt:

```toml
model = "gpt-5.6-terra"
model_provider = "openai_api_direct"
model_context_window = 922000
model_auto_compact_token_limit = 700000
model_auto_compact_token_limit_scope = "total"
model_catalog_json = "/absolute/path/to/models-api-1m.json"

[model_providers.openai_api_direct]
name = "OpenAI API direct"
base_url = "https://api.openai.com/v1"
wire_api = "responses"
requires_openai_auth = false

[model_providers.openai_api_direct.auth]
command = "/absolute/path/to/read-openai-inference-key"
timeout_ms = 5000
refresh_interval_ms = 300000
```

Die Authentifizierungshilfe darf ausschließlich den Schlüssel auf stdout ausgeben. Tragen Sie ihn nicht in TOML ein.

Behalten Sie für das Codex-App-Server-Harness von OpenClaw das standardmäßige agentenspezifische
Codex-Home-Verzeichnis bei und lassen Sie OpenClaw ein API-Schlüsselprofil vom Typ
`openai` einschleusen. Übergeben Sie den Katalog und die Kontextgrenzen als native
Argumente des Codex-App-Servers:

```json5
{
  auth: {
    order: {
      openai: ["openai:api-key"],
    },
  },
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            args: [
              "app-server",
              "--listen",
              "stdio://",
              "-c",
              'model_catalog_json="/absolute/path/to/models-api-1m.json"',
              "-c",
              "model_context_window=922000",
              "-c",
              "model_auto_compact_token_limit=700000",
              "-c",
              "model_auto_compact_token_limit_scope=total",
            ],
          },
        },
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.6-terra",
      models: {
        "openai/gpt-5.6-terra": { agentRuntime: { id: "codex" } },
      },
    },
  },
}
```

Ersetzen Sie `openai:api-key` bei Bedarf durch die tatsächliche ID des API-Schlüsselprofils. Der
agentenspezifische App-Server erhält ausschließlich diesen vorbereiteten Schlüssel; die native
ChatGPT-Anmeldung unter `~/.codex`, die Plugins, Konnektoren und der Thread-Speicher des
Betreibers bleiben unverändert. Codex App-Server `0.144.6` fügt bei App-Server-Turns
nicht den Bearer-Token eines benutzerdefinierten Providers mit Befehlsauthentifizierung hinzu.
Verwenden Sie für diese Route daher den oben beschriebenen eingeschleusten API-Schlüsselpfad
anstelle von `homeScope: "user"`.

Starten Sie nach Änderungen am Katalog oder an den App-Server-Argumenten den Gateway neu und
beginnen Sie einen neuen Chat. Bestehende native Threads behalten ihre aufgezeichneten Provider-
und Modelleinstellungen bei. Überprüfen Sie die Laufzeit mit `/status` und
`/codex status` und senden Sie anschließend einen harmlosen direkten API-Turn, bevor Sie eine
lange Sitzung beginnen.

<Warning>
Ein langer Kontext muss bewusst aktiviert werden. Sobald die Eingabe `272000` Token
überschreitet, berechnet OpenAI die gesamte Anfrage mit dem 2-Fachen Eingabe- und dem 1,5-Fachen
Ausgabetarif. Maßgeblich für Zugriff, tatsächliche Grenzwerte und Abrechnung bleibt die API. Siehe
[OpenAI-Modellgrenzen](https://developers.openai.com/api/docs/models/compare) und
[API-Preisgestaltung](https://developers.openai.com/api/docs/pricing).
</Warning>

Der Rest dieser Seite behandelt die Bereitstellungsstruktur, Fail-Closed-Routing,
Guardian-Genehmigungsrichtlinien, native Codex-Plugins und Computer Use. Vollständige Listen
der Optionen, Standardwerte, Enumerationen sowie Informationen zu Erkennung,
Umgebungsisolierung, Zeitüberschreitungen und App-Server-Transportfeldern finden Sie in der
[Referenz zum Codex-Harness](/de/plugins/codex-harness-reference).

## Codex-Laufzeit überprüfen

Verwenden Sie `/status` in dem Chat, in dem Sie Codex erwarten. Ein von Codex gestützter
OpenAI-Agent-Turn zeigt Folgendes an:

```text
Laufzeit: OpenAI Codex
```

Prüfen Sie anschließend den Zustand des Codex-App-Servers:

```text
/codex status
/codex models
/codex binding
```

`/codex binding` meldet den verknüpften nativen Thread und die aktuellen Modelleinstellungen.
`/codex status` meldet die App-Server-Verbindung, das Konto, die Ratenbegrenzungen, MCP-Server
und Skills. `/codex models` führt den Live-Katalog des Codex-App-Servers für das Harness und
das Konto auf. Falls `/status` unerwartet ist, lesen Sie
[Fehlerbehebung](#troubleshooting).

## Routing und Modellauswahl

Halten Sie Provider-Referenzen und Laufzeitrichtlinien getrennt:

- Verwenden Sie `openai/gpt-*` für die kanonische OpenAI-Modellauswahl. Das Präfix allein
  wählt niemals Codex aus.
- Wenn die Laufzeit nicht festgelegt oder auf `auto` gesetzt ist, kann ausschließlich
  eine exakte offizielle HTTPS-Route für Platform Responses oder ChatGPT Responses ohne selbst
  definierte Anfrageüberschreibung Codex implizit auswählen.
- Verwenden Sie keine veralteten Codex-GPT-Referenzen in der Konfiguration. Führen Sie
  `openclaw doctor --fix` aus, um veraltete Referenzen und überholte Sitzungs-Routenbindungen zu reparieren.
- `agentRuntime.id: "codex"` macht Codex für eine kompatible Route zu einer Fail-Closed-Anforderung.
  Eine inkompatible effektive Route wird dadurch nicht kompatibel.
- `agentRuntime.id: "openclaw"` aktiviert für einen Provider oder ein Modell die eingebettete
  OpenClaw-Laufzeit, wenn dies beabsichtigt ist.
- `/codex ...` steuert native Unterhaltungen des Codex-App-Servers aus dem Chat.
- ACP/acpx ist ein separater externer Harness-Pfad. Verwenden Sie ihn nur, wenn der Benutzer
  ACP/acpx oder einen externen Harness-Adapter anfordert.

| Benutzerabsicht                                            | Verwendung                                                                                            |
| ---------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| Aktuellen Chat verknüpfen                                  | `/codex bind [thread-id] [--cwd <path>] [--model <model>] [--provider <provider>]`                    |
| Einen bestehenden Codex-Thread fortsetzen                  | `/codex resume <thread-id>`                                                                           |
| Codex-Threads auflisten oder filtern                        | `/codex threads [filter]`                                                                             |
| Das native Ziel des gebundenen Threads lesen oder ändern   | `/codex goal [status\|set <objective>\|pause\|resume\|block\|complete\|clear]`                        |
| Native Codex-Plugins auflisten                             | `/codex plugins list`                                                                                 |
| Ein konfiguriertes natives Codex-Plugin aktivieren/deaktivieren | `/codex plugins enable <name>`, `/codex plugins disable <name>`                                       |
| Eine gespeicherte Codex-CLI-Sitzung als Paired-Node-Turn fortsetzen | `/codex sessions --host <node> [filter]`, dann `/codex resume <session-id> --host <node> --bind here` |
| Nicht archivierte Codex-Sitzungen computerübergreifend anzeigen | Codex-Überwachung aktivieren und **Codex Sessions** öffnen                                      |
| Modell, Schnellmodus oder Berechtigungen des gebundenen Threads ändern | `/codex model <model>`, `/codex fast [on\|off\|status]`, `/codex permissions [default\|yolo\|status]` |
| Aktiven Turn stoppen oder steuern                           | `/codex stop`, `/codex steer <text>`                                                                  |
| Aktuelle Bindung trennen                                    | `/codex detach` (Alias `/codex unbind`)                                                               |
| Ausschließlich Codex-Feedback senden                        | `/codex diagnostics [note]`                                                                           |
| Eine ACP/acpx-Aufgabe starten                               | ACP/acpx-Sitzungsbefehle, nicht `/codex`                                                               |

| Anwendungsfall                                  | Konfiguration                                                                                               | Überprüfung                             | Hinweise                                    |
| ----------------------------------------------- | ----------------------------------------------------------------------------------------------------------- | --------------------------------------- | ------------------------------------------- |
| Geeignete OpenAI-Route mit nativer Codex-Laufzeit | Exakte offizielle HTTPS-Responses-/ChatGPT-Route ohne selbst definierte Anfrageüberschreibung sowie aktiviertes Plugin `codex` | `/status` zeigt `Runtime: OpenAI Codex` | Impliziter Pfad, wenn die Laufzeit nicht festgelegt/`auto` ist |
| Fail-Closed, wenn Codex nicht verfügbar ist     | Provider oder Modell `agentRuntime.id: "codex"`                                                                | Turn schlägt statt eines eingebetteten Fallbacks fehl | Für reine Codex-Bereitstellungen verwenden |
| Direkter OpenAI-API-Schlüssel-Datenverkehr über OpenClaw | Provider oder Modell `agentRuntime.id: "openclaw"` und normale OpenAI-Authentifizierung                                      | `/status` zeigt die OpenClaw-Laufzeit        | Nur verwenden, wenn OpenClaw beabsichtigt ist |
| Veraltete Konfiguration                         | veraltete Codex-GPT-Referenzen                                                                               | `openclaw doctor --fix` schreibt sie um     | Neue Konfiguration nicht auf diese Weise erstellen |
| ACP/acpx-Codex-Adapter                          | ACP `sessions_spawn({ runtime: "acp" })`                                                                    | ACP-Aufgaben-/Sitzungsstatus            | Vom nativen Codex-Harness getrennt          |

`agents.defaults.imageModel` folgt derselben Präfixaufteilung. Verwenden Sie `openai/gpt-*`
für die normale OpenAI-Route und `codex/gpt-*` nur dann, wenn das Bildverständnis
über einen begrenzten Codex-App-Server-Turn ausgeführt werden soll. Doctor schreibt veraltete
Codex-GPT-Referenzen in `openai/gpt-*` um.

## Bereitstellungsmuster

### Einfache Codex-Bereitstellung

Verwenden Sie die Schnellstartkonfiguration für ein OpenAI-Modell, dessen effektive offizielle
HTTPS-Route geeignet ist, Codex implizit auszuwählen:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.6-sol",
    },
  },
}
```

### Bereitstellung mit gemischten Providern

Behalten Sie Claude als Standardagenten bei und fügen Sie einen benannten Codex-Agenten hinzu:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
  agents: {
    defaults: {
      model: "anthropic/claude-opus-4-6",
    },
    list: [
      {
        id: "main",
        default: true,
        model: "anthropic/claude-opus-4-6",
      },
      {
        id: "codex",
        name: "Codex",
        model: "openai/gpt-5.6-sol",
      },
    ],
  },
}
```

Der Agent `main` verwendet seinen normalen Provider-Pfad. Der Agent
`codex` verwendet den Codex-App-Server, solange seine effektive OpenAI-Route kompatibel
bleibt. Fügen Sie explizit modellbezogenes `agentRuntime.id: "codex"` hinzu, wenn dies eine
Fail-Closed-Anforderung sein soll.

### Fail-Closed-Codex-Bereitstellung

Eine geeignete, exakte offizielle HTTPS-OpenAI-Route kann zu Codex aufgelöst werden, wenn das
gebündelte Plugin verfügbar ist. Fügen Sie für eine ausdrücklich festgelegte
Fail-Closed-Regel eine explizite Laufzeitrichtlinie hinzu:

```json5
{
  models: {
    providers: {
      openai: {
        agentRuntime: {
          id: "codex",
        },
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.6-sol",
    },
  },
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
}
```

Wenn Codex erzwungen wird, bricht OpenClaw frühzeitig ab, falls die effektive Route nicht als
Codex-kompatibel deklariert ist, das Plugin deaktiviert ist, der App-Server zu alt ist oder der
App-Server nicht gestartet werden kann.

## App-Server-Richtlinie

Standardmäßig startet das Plugin die von OpenClaw verwaltete Codex-Binärdatei lokal mit
stdio-Transport. Setzen Sie `appServer.command` nur, wenn absichtlich eine
andere ausführbare Datei verwendet werden soll. Codex stuft den WebSocket-Transport als experimentell
und nicht unterstützt ein; verwenden Sie ihn nur für Tests außerhalb der Produktion mit einem App-Server,
der bereits an anderer Stelle ausgeführt wird:

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
          },
        },
      },
    },
  },
}
```

Lokale stdio-App-Server-Sitzungen verwenden standardmäßig die Vertrauensstellung für
lokale Betreiber: `approvalPolicy: "never"`, `approvalsReviewer: "user"` und
`sandbox: "danger-full-access"`. Wenn lokale Codex-Anforderungen diese
implizite YOLO-Vertrauensstellung nicht zulassen, wählt OpenClaw stattdessen
zulässige Guardian-Berechtigungen aus. Wenn für die Sitzung eine OpenClaw-Sandbox
aktiv ist, deaktiviert OpenClaw für diesen Turn den nativen Code Mode von Codex,
benutzerdefinierte MCP-Server und die Ausführung App-gestützter Plugins, anstatt sich
auf das hostseitige Sandboxing von Codex zu verlassen. Der Shell-Zugriff erfolgt
stattdessen über dynamische, von der OpenClaw-Sandbox unterstützte Tools wie
`sandbox_exec` und `sandbox_process`, wenn die normalen Exec-/Prozesstools
verfügbar sind.

Verwenden Sie den normalisierten OpenClaw-Exec-Modus für die native automatische Überprüfung
von Codex, bevor Sandbox-Ausbrüche oder zusätzliche Berechtigungen eingesetzt werden:

```json5
{
  tools: {
    exec: {
      mode: "auto",
    },
  },
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
}
```

Bei Codex-App-Server-Sitzungen wird `tools.exec.mode: "auto"` auf von Codex
Guardian geprüfte Genehmigungen abgebildet: üblicherweise `approvalPolicy: "on-request"`,
`approvalsReviewer: "auto_review"` und `sandbox: "workspace-write"`, wenn
die lokalen Anforderungen diese Werte zulassen. In `tools.exec.mode: "auto"`
behält OpenClaw veraltete unsichere Codex-Überschreibungen für `approvalPolicy: "never"`
oder `sandbox: "danger-full-access"` nicht bei; verwenden Sie `tools.exec.mode: "full"`
für eine absichtlich genehmigungsfreie Codex-Vertrauensstellung. Die veraltete
Voreinstellung `plugins.entries.codex.config.appServer.mode: "guardian"` funktioniert weiterhin,
aber `tools.exec.mode: "auto"` ist die normalisierte OpenClaw-Oberfläche.

Den Vergleich auf Modusebene mit Host-Exec-Genehmigungen und ACPX-Berechtigungen
finden Sie unter [Berechtigungsmodi](/de/tools/permission-modes). Einzelheiten zu allen
App-Server-Feldern, zur Authentifizierungsreihenfolge, zur Umgebungsisolation und zum Timeout-Verhalten
finden Sie in der [Codex-Harness-Referenz](/de/plugins/codex-harness-reference).

## Befehle und Diagnose

Das Plugin `codex` registriert `/codex` als Slash-Befehl in jedem Kanal, der
OpenClaw-Textbefehle unterstützt.

Native Ausführung und Steuerung erfordern einen Eigentümer oder einen `operator.admin`-
Gateway-Client: Threads binden oder fortsetzen, Turns senden oder stoppen,
Modell, Schnellmodus oder Berechtigungsstatus ändern, Compaction oder Überprüfungen
durchführen und eine Bindung lösen. Andere autorisierte Absender behalten
schreibgeschützte Befehle zur Status-, Hilfe-, Konto-, Modell-, Thread-, nativen Ziel-,
MCP-Server-, Skill- und Bindungsprüfung.

Gängige Formen:

- `/codex status` prüft App-Server-Konnektivität, Modelle, Konto,
  Nutzungslimits, MCP-Server und Skills.
- `/codex models` listet aktive Codex-App-Server-Modelle auf.
- `/codex threads [filter]` listet kürzlich verwendete Codex-App-Server-Threads auf.
- `/codex goal` liest oder aktualisiert das native Codex-Ziel des angehängten Threads. Die automatische Zielfortsetzung von Codex bleibt deaktiviert; OpenClaw verwaltet noch keine autonomen nachfolgenden Turns.
- `/codex resume <thread-id>` hängt die aktuelle OpenClaw-Sitzung an einen
  vorhandenen Codex-Thread an.
- `/codex bind [thread-id] [--cwd <path>] [--model <model>] [--provider <provider>]`
  hängt den aktuellen Chat an.
- `/codex detach` (oder `/codex unbind`) löst die aktuelle Bindung.
- `/codex binding` beschreibt die aktuelle Bindung.
- `/codex stop` stoppt den aktiven Turn; `/codex steer <text>` steuert ihn.
- `/codex model <model>`, `/codex fast [on|off|status]` und
  `/codex permissions [default|yolo|status]` ändern den konversationsbezogenen Status.
- `/codex compact` fordert den Codex-App-Server auf, für den angehängten Thread eine Compaction durchzuführen.
- `/codex review` startet die native Codex-Überprüfung für den angehängten Thread.
- `/codex diagnostics [note]` fragt vor dem Senden von Codex-Feedback für den
  angehängten Thread nach.
- `/codex account` zeigt den Konto- und Nutzungslimitstatus an.
- `/codex mcp` listet den Status der MCP-Server des Codex-App-Servers auf.
- `/codex skills` listet die Skills des Codex-App-Servers auf.
- `/codex plugins list`, `/codex plugins enable <name>` und
  `/codex plugins disable <name>` verwalten konfigurierte native Codex-Plugins.
- `/codex computer-use [status|install]` verwaltet Codex Computer Use.
- `/codex help` listet den vollständigen Befehlsbaum auf.

Beginnen Sie bei den meisten Supportmeldungen mit `/diagnostics [note]` in der
Konversation, in der der Fehler aufgetreten ist. Dadurch wird ein Gateway-Diagnosebericht
erstellt und bei Codex-Harness-Sitzungen um Genehmigung zum Senden des
relevanten Codex-Feedbackpakets gebeten. Informationen zum Datenschutzmodell und zum Verhalten
in Gruppenchats finden Sie unter
[Diagnoseexport](/de/gateway/diagnostics). Verwenden Sie `/codex diagnostics [note]` nur, wenn Sie ausdrücklich
das Codex-Feedback für den derzeit angehängten Thread hochladen möchten, ohne
das vollständige Gateway-Diagnosepaket einzuschließen.

### Codex-Threads lokal prüfen

Die schnellste Möglichkeit, einen fehlerhaften Codex-Lauf zu untersuchen, besteht häufig darin, den nativen
Codex-Thread direkt zu öffnen:

```bash
codex resume <thread-id>
```

Die Thread-ID finden Sie in der abgeschlossenen Antwort von `/diagnostics`, in `/codex binding`
oder in `/codex threads [filter]`.

Informationen zum Upload-Mechanismus und zu den Diagnosegrenzen auf Runtime-Ebene finden Sie unter
[Codex-Harness-Runtime](/de/plugins/codex-harness-runtime#codex-feedback-upload).

### Authentifizierungsreihenfolge

Im standardmäßigen agentenbezogenen Home-Verzeichnis wird die Authentifizierung in dieser Reihenfolge ausgewählt:

1. Geordnete OpenAI-Authentifizierungsprofile für den Agenten, vorzugsweise unter
   `auth.order.openai`. Führen Sie `openclaw doctor --fix` aus, um ältere veraltete
   Codex-Authentifizierungsprofil-IDs und die veraltete Codex-Authentifizierungsreihenfolge zu migrieren.
2. Das vorhandene Konto des App-Servers im Codex-Home-Verzeichnis dieses Agenten.
3. Nur bei lokalen stdio-App-Server-Starts: `CODEX_API_KEY`, danach
   `OPENAI_API_KEY`, wenn kein App-Server-Konto vorhanden ist und weiterhin eine OpenAI-Authentifizierung
   erforderlich ist.

Wenn OpenClaw ein Codex-Authentifizierungsprofil im Stil eines ChatGPT-Abonnements erkennt,
entfernt es `CODEX_API_KEY` und `OPENAI_API_KEY` aus dem gestarteten
Codex-Kindprozess. Dadurch bleiben API-Schlüssel auf Gateway-Ebene für Einbettungen oder
direkte OpenAI-Modelle verfügbar, ohne dass native Turns des Codex-App-Servers
versehentlich über die API abgerechnet werden. Explizite Codex-API-Schlüsselprofile und der lokale
stdio-Fallback auf Umgebungsschlüssel verwenden die App-Server-Anmeldung anstelle der geerbten
Umgebung des Kindprozesses. WebSocket-App-Server-Verbindungen erhalten keinen Fallback auf
Gateway-Umgebungs-API-Schlüssel; verwenden Sie ein explizites Authentifizierungsprofil oder das eigene
Konto des entfernten App-Servers.

Wenn ein Abonnementprofil ein Codex-Nutzungslimit erreicht, zeichnet OpenClaw die
Rücksetzzeit auf, sofern Codex eine meldet, und versucht für denselben Codex-Lauf
das nächste geordnete Authentifizierungsprofil. Nach Ablauf der Rücksetzzeit kann das
Abonnementprofil wieder verwendet werden, ohne dass das ausgewählte `openai/gpt-*`-
Modell oder die Codex-Runtime geändert wird.

Wenn native Codex-Plugins konfiguriert sind, installiert oder aktualisiert OpenClaw
diese Plugins über den verbundenen App-Server, bevor dem Codex-Thread
Plugin-eigene Apps bereitgestellt werden. `app/list` bleibt die maßgebliche Quelle für App-
IDs, Zugänglichkeit und Metadaten, aber OpenClaw verwaltet die threadbezogene
Aktivierungsentscheidung: Wenn die Richtlinie eine aufgeführte zugängliche App zulässt, sendet OpenClaw
`thread/start.config.apps[appId].enabled = true`, selbst wenn `app/list`
diese App derzeit als deaktiviert meldet. Dieser Pfad erfindet keine App-
Installation für unbekannte IDs; OpenClaw aktiviert nur Marketplace-Plugins
mit `plugin/install` und aktualisiert anschließend den Bestand.

### Umgebungsisolation

Bei lokalen stdio-App-Server-Starts setzt OpenClaw `CODEX_HOME` auf ein
agentenbezogenes Verzeichnis, sodass Codex-Konfiguration, Authentifizierungs-/Kontodateien, Plugin-Cache/-Daten
und der native Thread-Status standardmäßig nicht aus dem persönlichen
`~/.codex` des Betreibers gelesen oder dorthin geschrieben werden. OpenClaw behält das normale
prozessbezogene `HOME` bei; von Codex ausgeführte Unterprozesse können weiterhin
Konfigurationen und Token im Benutzer-Home-Verzeichnis finden, und Codex kann gemeinsame Einträge unter
`$HOME/.agents/skills` und `$HOME/.agents/plugins/marketplace.json` erkennen. Mit
`appServer.homeScope: "user"` verwendet OpenClaw stattdessen das native Codex-
Home-Verzeichnis des Benutzers und dessen vorhandenes Konto, ohne ein OpenClaw-Authentifizierungsprofil einzufügen.

Wenn eine Bereitstellung zusätzliche Umgebungsisolation erfordert, fügen Sie diese
Variablen zu `appServer.clearEnv` hinzu:

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

`appServer.clearEnv` wirkt sich nur auf den gestarteten Kindprozess des Codex-App-Servers
aus. OpenClaw entfernt `CODEX_HOME` und `HOME` während
der lokalen Startnormalisierung aus dieser Liste: `CODEX_HOME` verweist weiterhin auf den ausgewählten
Agenten- oder Benutzerbereich, und `HOME` wird weiterhin geerbt, damit Unterprozesse den
normalen Status des Benutzer-Home-Verzeichnisses verwenden können.

### Dynamische Tools und Websuche

Dynamische Codex-Tools verwenden standardmäßig das Laden über `searchable`. OpenClaw stellt normalerweise
keine dynamischen Tools bereit, die native Codex-Arbeitsbereichsoperationen duplizieren:
`read`, `write`, `edit`, `apply_patch`, `exec`, `process`, `update_plan`,
`get_goal`, `create_goal`, `update_goal`, `tool_call`, `tool_describe`,
`tool_search` und `tool_search_code`. Zieloperationen bleiben nativ in Codex,
daher projiziert OpenClaw keinen zweiten Zielspeicher in Codex-Turns. Die meisten
übrigen OpenClaw-Integrationstools, etwa für Messaging, Medien, Cron,
Browser, Nodes, Gateway und `heartbeat_respond`, sind über die
Codex-Tool-Suche im Namespace `openclaw` verfügbar, wodurch der anfängliche Modellkontext
kleiner bleibt. Der Shell-Fallback für eingeschränkte Turns bildet die Ausnahme für
`exec` und `process`, wenn eine endliche Positivliste den nativen Code Mode deaktiviert;
Runtime-Positivlisten und `codexDynamicToolsExclude` gelten weiterhin.

Mit `catalogMode: "direct-only"` gekennzeichnete Tools, einschließlich des OpenClaw-Tools `computer`,
verwenden stattdessen den Namespace `openclaw_direct`. Codex behandelt diesen Namespace
als `DirectModelOnly`, sodass diese Tools in normalen und ausschließlich für den
Code Mode vorgesehenen Threads direkt für das Modell sichtbar bleiben, anstatt verschachtelte
Code-Mode-Aufrufe von `tools.*` zu durchlaufen.

Die Websuche verwendet standardmäßig das gehostete Codex-Tool `web_search`, wenn die Suche
aktiviert und kein verwalteter Provider ausgewählt ist. Die native gehostete Suche und
das verwaltete dynamische OpenClaw-Tool `web_search` schließen sich gegenseitig aus, damit
die verwaltete Suche native Domänenbeschränkungen nicht umgehen kann. OpenClaw verwendet das
verwaltete Tool, wenn die gehostete Suche nicht verfügbar oder ausdrücklich deaktiviert ist oder
durch einen ausgewählten verwalteten Provider ersetzt wurde. OpenClaw lässt die eigenständige
Codex-Erweiterung `web.run` deaktiviert, da produktiver App-Server-Datenverkehr
deren benutzerdefinierten Namespace `web` ablehnt. `tools.web.search.enabled: false`
deaktiviert beide Pfade, ebenso wie reine LLM-Läufe mit deaktivierten Tools. Codex behandelt
`"cached"` als Präferenz und löst sie für uneingeschränkte App-Server-Turns
in einen aktiven externen Zugriff auf. Der automatische verwaltete Fallback schlägt geschlossen fehl, wenn
native `allowedDomains` festgelegt sind, sodass die Positivliste nicht umgangen werden kann.
Dauerhafte Änderungen der effektiven Suchrichtlinie rotieren den gebundenen Codex-Thread
vor dem nächsten Turn; vorübergehende Einschränkungen pro Turn verwenden einen temporären
eingeschränkten Thread und behalten die bestehende Bindung für eine spätere Fortsetzung bei.

`sessions_yield`, `sessions_spawn` und ausschließlich vom Nachrichten-Tool stammende Antworten bleiben
direkt, da sie Verträge zur Ablaufsteuerung oder Delegation darstellen. Die Anleitung
bevorzugt weiterhin das native `spawn_agent` von Codex als primäre Oberfläche für Codex-Unteragenten,
während eine explizite OpenClaw- oder ACP-Delegation weiterhin direkt über
`sessions_spawn` aufgerufen werden kann. Im Codex Code Mode sind generische Ergebnisse dynamischer OpenClaw-
Tools JSON-Text und keine JavaScript-Objekte. Analysieren Sie daher
wie JSON aussehende Ergebnisse, bevor Sie Felder auslesen. Codex serialisiert außerdem verschachtelte
dynamische Aufrufe. Übermitteln Sie mehrere `sessions_spawn`-Aufrufe in einer begrenzten Schleife,
statt zu erwarten, dass `Promise.all` sie gleichzeitig startet. Bereits akzeptierte
untergeordnete Prozesse können sich weiterhin zeitlich überschneiden, während spätere Aufrufe übermittelt werden. Unter
[Schwarm](/de/tools/swarm#use-swarm-from-other-harnesses) finden Sie ein vollständiges Muster.
Anweisungen zur Heartbeat-Zusammenarbeit
weisen Codex an, vor dem Beenden eines Heartbeat-Durchlaufs nach `heartbeat_respond` zu suchen,
wenn das Tool noch nicht geladen ist.

Legen Sie `codexDynamicToolsLoading: "direct"` nur fest, wenn Sie eine Verbindung zu einem benutzerdefinierten
Codex-App-Server herstellen, der nicht nach zurückgestellten dynamischen Tools suchen kann, oder wenn Sie
die vollständige Tool-Nutzlast debuggen.

### Konfigurationsfelder

Unterstützte Codex-Plugin-Felder der obersten Ebene:

| Feld                       | Standardwert   | Bedeutung                                                                                |
| -------------------------- | -------------- | ---------------------------------------------------------------------------------------- |
| `codexDynamicToolsLoading` | `"searchable"` | Verwenden Sie `"direct"`, um dynamische OpenClaw-Tools direkt in den anfänglichen Codex-Toolkontext aufzunehmen. |
| `codexDynamicToolsExclude` | `[]`           | Zusätzliche Namen dynamischer OpenClaw-Tools, die bei Codex-App-Server-Durchläufen ausgelassen werden sollen. |
| `codexPlugins`             | deaktiviert    | Native Codex-Plugin-/App-Unterstützung für migrierte, aus dem Quellcode installierte kuratierte Plugins. |
| `sessionCatalog`           | aktiviert      | Seitenleisten-Erkennung für native Codex-Sitzungen auf diesem Gateway und auf geeigneten gekoppelten Nodes. |
| `supervision`              | deaktiviert    | Agentenseitiges Transkript nativer Sitzungen und Richtlinie zur Schreibsteuerung.        |

Unterstützte `appServer`-Felder:

| Feld                                          | Standardwert                                           | Bedeutung                                                                                                                                                                                                                                                                                                                                                                                       |
| --------------------------------------------- | ------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `transport`                            | `"stdio"`                                     | `"stdio"` startet Codex; das explizite `"unix"` stellt eine Verbindung zum lokalen Steuerungs-Socket her; `"websocket"` stellt eine Verbindung zu `url` her.                                                                                                                                                                                                 |
| `homeScope`                            | `"agent"`                                     | `"agent"` isoliert den gewöhnlichen Harness-Zustand pro OpenClaw-Agent. `"user"` ist eine explizite Opt-in-Option, die das native `$CODEX_HOME` oder `~/.codex` gemeinsam nutzt, native Authentifizierung verwendet und die Thread-Verwaltung ausschließlich für den Eigentümer aktiviert. Der Benutzerbereich unterstützt lokales stdio oder Unix-Transport. Für die separate Überwachungsverbindung wird ein nicht gesetzter Wert bei stdio oder Unix zu `"user"` und bei WebSocket zu `"agent"` aufgelöst. |
| `command`                            | verwaltete Codex-Binärdatei                            | Ausführbare Datei für den stdio-Transport. Lassen Sie den Wert nicht gesetzt, um die verwaltete Binärdatei zu verwenden; legen Sie ihn nur für eine explizite Überschreibung fest.                                                                                                                                                                                                                |
| `args`                            | `["app-server", "--listen", "stdio://"]`                                     | Argumente für den stdio-Transport.                                                                                                                                                                                                                                                                                                                                                              |
| `url`                            | nicht gesetzt                                          | WebSocket-App-Server-URL oder `unix://`-URL. Ein explizit leerer Unix-Pfad wählt den kanonischen Steuerungs-Socket im Benutzerverzeichnis aus.                                                                                                                                                                                                                                          |
| `authToken`                            | nicht gesetzt                                          | Bearer-Token für den WebSocket-Transport. Akzeptiert eine literale Zeichenfolge oder SecretInput wie `${CODEX_APP_SERVER_TOKEN}`.                                                                                                                                                                                                                                                                        |
| `headers`                            | `{}`                                     | Zusätzliche WebSocket-Header. Header-Werte akzeptieren literale Zeichenfolgen oder SecretInput-Werte, beispielsweise `x-codex-client-session-token: "${CODEX_CLIENT_SESSION_TOKEN}"`.                                                                                                                                                                                                                                                        |
| `clearEnv`                            | `[]`                                     | Namen zusätzlicher Umgebungsvariablen, die aus dem gestarteten stdio-App-Server-Prozess entfernt werden, nachdem OpenClaw dessen geerbte Umgebung erstellt hat. OpenClaw behält das ausgewählte `CODEX_HOME` und das geerbte `HOME` für lokale Starts bei.                                                                                                                        |
| `codeModeOnly`                            | `false`                                     | Aktiviert die ausschließlich für den Codemodus vorgesehene Tool-Oberfläche von Codex. Gewöhnliche dynamische OpenClaw-Tools bleiben über verschachtelte `tools.*`-Aufrufe verfügbar; `openclaw_direct`-Tools bleiben für das Modell direkt sichtbar.                                                                                                                                       |
| `remoteWorkspaceRoot`                            | nicht gesetzt                                          | Workspace-Stammverzeichnis des entfernten Codex-App-Servers. Wenn es festgelegt ist, leitet OpenClaw das lokale Workspace-Stammverzeichnis aus dem aufgelösten OpenClaw-Workspace ab, behält das aktuelle cwd-Suffix unter diesem entfernten Stammverzeichnis bei und sendet nur das endgültige App-Server-cwd an Codex. Wenn sich das cwd außerhalb des aufgelösten OpenClaw-Workspace-Stammverzeichnisses befindet, bricht OpenClaw sicher ab, anstatt einen Gateway-lokalen Pfad an den entfernten App-Server zu senden. |
| `requestTimeoutMs`                            | `60000`                                     | Zeitüberschreitung für Aufrufe der App-Server-Steuerungsebene.                                                                                                                                                                                                                                                                                                                                 |
| `turnCompletionIdleTimeoutMs`                            | `60000`                                     | Ruhefenster, nachdem Codex einen Turn akzeptiert hat oder nachdem eine auf den Turn beschränkte App-Server-Anfrage erfolgt ist, während OpenClaw auf `turn/completed` wartet.                                                                                                                                                                                                                   |
| `turnAssistantCompletionIdleTimeoutMs`                            | `10000`                                     | Ruhefenster, nachdem ein endgültiges bzw. nicht kommentierendes Assistentenelement oder der Abschluss einer rohen Assistentenausgabe vor einem Tool die Freigabe der Assistentenausgabe aktiviert, während OpenClaw weiterhin auf `turn/completed` wartet. Durch eine Erhöhung erhält Codex mehr Zeit, `turn/completed` auszugeben, bevor OpenClaw unterbricht und die Session-Lane freigibt. |
| `postToolRawAssistantCompletionIdleTimeoutMs`                            | `300000`                                     | Abschlussleerlauf- und Fortschrittswächter, der nach einer Tool-Übergabe, dem Abschluss eines nativen Tools, dem Fortschritt einer rohen Assistentenausgabe nach einem Tool, dem Abschluss einer rohen Reasoning-Ausgabe oder einem Reasoning-Fortschritt verwendet wird, während OpenClaw auf `turn/completed` wartet. Verwenden Sie dies für vertrauenswürdige oder rechenintensive Workloads, bei denen die Synthese nach einem Tool berechtigterweise länger still bleiben kann als das Freigabebudget für die endgültige Assistentenausgabe. |
| `mode`                            | `"yolo"`, sofern lokale Codex-Anforderungen YOLO nicht ausschließen | Voreinstellung für YOLO oder eine durch Guardian geprüfte Ausführung. Lokale stdio-Anforderungen, die `danger-full-access`, die `never`-Genehmigung oder den `user`-Prüfer auslassen, machen Guardian zum impliziten Standard.                                                                                                                                                    |
| `approvalPolicy`                            | `"never"` oder eine zulässige Guardian-Genehmigungsrichtlinie | Native Codex-Genehmigungsrichtlinie, die beim Starten/Fortsetzen eines Threads bzw. bei einem Turn gesendet wird. Guardian-Standardwerte bevorzugen `"on-request"`, sofern zulässig.                                                                                                                                                                                                          |
| `sandbox`                            | `"danger-full-access"` oder eine zulässige Guardian-Sandbox | Nativer Codex-Sandbox-Modus, der beim Starten/Fortsetzen eines Threads gesendet wird. Guardian-Standardwerte bevorzugen `"workspace-write"`, sofern zulässig, andernfalls `"read-only"`. Wenn eine OpenClaw-Sandbox aktiv ist, verwenden `danger-full-access`-Turns Codex `workspace-write` mit Netzwerkzugriff, der aus der Egress-Einstellung der OpenClaw-Sandbox abgeleitet wird. |
| `approvalsReviewer`                            | `"user"` oder ein zulässiger Guardian-Prüfer | Verwenden Sie `"auto_review"`, damit Codex native Genehmigungsaufforderungen prüft, sofern zulässig, andernfalls `guardian_subagent` oder `user`. `guardian_subagent` bleibt ein Legacy-Alias.                                                                                                                                                                                         |
| `serviceTier`                            | nicht gesetzt                                          | Optionale Servicestufe des Codex-App-Servers. `"priority"` aktiviert das Fast-Mode-Routing, `"flex"` fordert die Flex-Verarbeitung an, `null` entfernt die Überschreibung und das veraltete `"fast"` wird als `"priority"` akzeptiert.                                                                                                                  |
| `networkProxy`                            | deaktiviert                                            | Aktiviert das Netzwerkprofil für Codex-Berechtigungen bei App-Server-Befehlen. OpenClaw definiert die ausgewählte `permissions.<profile>.network`-Konfiguration und wählt sie mit `default_permissions` aus, anstatt `sandbox` zu senden.                                                                                                                                                                 |
| `experimental.sandboxExecServer`                            | `false`                                     | Vorschau-Opt-in, das beim unterstützten Codex-App-Server eine durch die OpenClaw-Sandbox gestützte Codex-Umgebung registriert, sodass die native Codex-Ausführung innerhalb der aktiven OpenClaw-Sandbox erfolgen kann.                                                                                                                                                                            |

`appServer.networkProxy` ist explizit, da es den Codex-Sandbox-Vertrag
ändert. Wenn diese Option aktiviert ist, legt OpenClaw außerdem `features.network_proxy.enabled`
und `default_permissions` in der Codex-Thread-Konfiguration fest, damit das generierte
Berechtigungsprofil das von Codex verwaltete Networking starten kann. Standardmäßig
generiert OpenClaw aus dem Profilinhalt einen kollisionsresistenten
`openclaw-network-<fingerprint>`-Profilnamen; verwenden Sie `profileName` nur, wenn ein
stabiler lokaler Name erforderlich ist.

```json5
{
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
              unixSockets: {
                "/tmp/proxy.sock": "allow",
                "/tmp/blocked.sock": "none",
              },
              allowUpstreamProxy: true,
              proxyUrl: "http://127.0.0.1:3128",
            },
          },
        },
      },
    },
  },
}
```

Wenn die normale App-Server-Laufzeit `danger-full-access` wäre, verwendet die
Aktivierung von `networkProxy` für das generierte Berechtigungsprofil einen
Dateisystemzugriff im Workspace-Stil: Die von Codex verwaltete Netzwerkdurchsetzung
ist ein Sandbox-Networking, sodass ein Profil mit vollständigem Zugriff ausgehenden
Datenverkehr nicht schützen würde. Domaineinträge verwenden `allow` oder
`deny`; Unix-Socket-Einträge verwenden die Codex-Werte
`allow` oder `none`.

### Dynamische Zeitlimits für Tool-Aufrufe

OpenClaw-eigene dynamische Tool-Aufrufe werden unabhängig von
`appServer.requestTimeoutMs` begrenzt: Codex-Anfragen vom Typ `item/tool/call`
verwenden standardmäßig einen OpenClaw-Watchdog von 90 Sekunden. Ein positives
aufrufspezifisches Argument `timeoutMs` verlängert oder verkürzt das Budget
dieses konkreten Tools, begrenzt auf 600000 ms. Das Tool `image_generate`
verwendet `agents.defaults.mediaModels.image.timeoutMs`, wenn der Tool-Aufruf kein eigenes Zeitlimit angibt,
andernfalls gilt für die Bilderzeugung ein Standardwert von 120 Sekunden. Das
Medienanalyse-Tool `image` verwendet `timeoutSeconds` des ausgewählten
bildfähigen Eintrags `tools.media.models[]` oder seinen Medienstandardwert von
60 Sekunden; bei der Bildanalyse gilt dieses Zeitlimit für die Anfrage selbst und
wird nicht durch vorherige Vorbereitungsarbeiten verkürzt. Bei einer
Zeitüberschreitung bricht OpenClaw das Tool-Signal ab, sofern dies unterstützt wird,
und gibt eine fehlgeschlagene Antwort des dynamischen Tools an Codex zurück, damit
der Turn fortgesetzt werden kann, anstatt die Sitzung in `processing` zu
belassen. Dieser Watchdog bildet das äußere dynamische Budget
`item/tool/call`; providerspezifische Anfragezeitlimits laufen innerhalb dieses
Aufrufs und behalten ihre eigene Zeitlimitsemantik bei.

Nachdem Codex einen Turn akzeptiert hat und nachdem OpenClaw auf eine
turnbezogene App-Server-Anfrage geantwortet hat, erwartet das Harness, dass Codex
im aktuellen Turn Fortschritte erzielt und den nativen Turn schließlich mit
`turn/completed` abschließt. Wenn der App-Server für `appServer.turnCompletionIdleTimeoutMs`
inaktiv bleibt, unterbricht OpenClaw nach bestem Bemühen den Codex-Turn, zeichnet
eine diagnostische Zeitüberschreitung auf und gibt die OpenClaw-Sitzungsspur frei,
damit nachfolgende Chatnachrichten nicht hinter einem veralteten nativen Turn
eingereiht werden. Die meisten nicht terminalen Benachrichtigungen desselben Turns
deaktivieren diesen kurzen Watchdog, da Codex damit nachgewiesen hat, dass der Turn
noch aktiv ist.

Tool-Übergaben verwenden ein längeres Leerlaufbudget nach dem Tool: nachdem
OpenClaw eine Antwort vom Typ `item/tool/call` zurückgegeben hat, nachdem native
Tool-Elemente wie `commandExecution` abgeschlossen wurden, nach unverarbeiteten
`custom_tool_call_output`-Abschlüssen sowie nach unverarbeitetem Assistentenfortschritt,
Reasoning-Abschlüssen oder Reasoning-Fortschritt nach dem Tool. Die Schutzlogik
verwendet `appServer.postToolRawAssistantCompletionIdleTimeoutMs`, wenn dies konfiguriert ist, und andernfalls
standardmäßig fünf Minuten; dasselbe Budget verlängert außerdem den
Fortschritts-Watchdog für das stille Synthesefenster, bevor Codex das nächste
Ereignis des aktuellen Turns ausgibt. Globale App-Server-Benachrichtigungen wie
Aktualisierungen von Ratenbegrenzungen setzen den Leerlauffortschritt des Turns
nicht zurück. Auf Reasoning-Abschlüsse, Kommentarabschlüsse vom Typ
`agentMessage` sowie unverarbeiteten Reasoning- oder Assistentenfortschritt vor
einem Tool kann eine automatische endgültige Antwort folgen. Daher verwenden sie
die Antwortschutzlogik nach Fortschritt, anstatt die Sitzungsspur sofort
freizugeben.

Nur abgeschlossene endgültige bzw. nicht kommentierende Elemente vom Typ
`agentMessage` und unverarbeitete Assistentenabschlüsse vor einem Tool
aktivieren die Freigabe nach Assistentenausgabe: Wenn Codex anschließend ohne
`turn/completed` inaktiv bleibt, unterbricht OpenClaw nach bestem Bemühen den
nativen Turn und gibt die Sitzungsspur frei. Wenn eine andere Turn-Überwachung
dieses Freigaberennen gewinnt, akzeptiert OpenClaw das abgeschlossene endgültige
Assistentenelement dennoch, sobald keine native Anfrage, kein Element und kein
Abschluss eines dynamischen Tools mehr aktiv ist, die Freigabe nach
Assistentenausgabe weiterhin zum zuletzt abgeschlossenen Element gehört und kein
späterer Elementabschluss vorliegt. Dadurch kann die endgültige Antwort nach
abgeschlossener Tool-Arbeit erhalten bleiben, ohne den Turn erneut auszuführen.
Partielle Assistenten-Deltas, veraltete frühere Antworten und leere spätere
Abschlüsse erfüllen die Voraussetzungen nicht.

Wiederholungssichere Fehler des stdio-App-Servers, einschließlich
Zeitüberschreitungen im Leerlauf beim Turn-Abschluss ohne Hinweise auf Assistent,
Tool, aktives Element oder Nebeneffekte, werden einmal mit einem neuen
App-Server-Versuch wiederholt. Bei unsicheren Zeitüberschreitungen wird der
hängende App-Server-Client dennoch außer Betrieb genommen und die
OpenClaw-Sitzungsspur freigegeben; außerdem wird die veraltete Bindung des nativen
Threads entfernt, anstatt sie automatisch erneut auszuführen. Zeitüberschreitungen
der Abschlussüberwachung zeigen Codex-spezifischen Zeitlimittext an: In
wiederholungssicheren Fällen wird darauf hingewiesen, dass die Antwort möglicherweise
unvollständig ist, während unsichere Fälle den Benutzer auffordern, vor einem
erneuten Versuch den aktuellen Zustand zu überprüfen. Öffentliche
Zeitüberschreitungsdiagnosen enthalten strukturelle Felder wie die Methode der
letzten App-Server-Benachrichtigung, ID/Typ/Rolle des unverarbeiteten
Assistentenantwortelements, die Anzahl aktiver Anfragen und Elemente sowie den
aktivierten Überwachungszustand. Wenn die letzte Benachrichtigung ein
unverarbeitetes Assistentenantwortelement ist, enthalten sie außerdem eine
begrenzte Vorschau des Assistententextes. Sie enthalten weder den unverarbeiteten
Prompt noch Tool-Inhalte.

### Lokale Überschreibungen der Testumgebung

- `OPENCLAW_CODEX_APP_SERVER_BIN` umgeht die verwaltete Binärdatei, wenn
  `appServer.command` nicht festgelegt ist.
- `OPENCLAW_CODEX_APP_SERVER_ARGS`
- `OPENCLAW_CODEX_APP_SERVER_MODE=yolo|guardian`
- `OPENCLAW_CODEX_APP_SERVER_APPROVAL_POLICY`
- `OPENCLAW_CODEX_APP_SERVER_SANDBOX`

`OPENCLAW_CODEX_APP_SERVER_GUARDIAN=1` wurde entfernt. Verwenden Sie stattdessen
`plugins.entries.codex.config.appServer.mode: "guardian"` oder `OPENCLAW_CODEX_APP_SERVER_MODE=guardian` für einmalige lokale Tests.
Für wiederholbare Bereitstellungen wird die Konfiguration bevorzugt, da sie das
Plugin-Verhalten in derselben geprüften Datei wie die übrige Einrichtung des
Codex-Harness hält.

## Native Codex-Plugins

Die native Unterstützung für Codex-Plugins verwendet die eigenen App- und
Plugin-Funktionen des Codex-App-Servers im selben Codex-Thread wie der
OpenClaw-Harness-Turn. OpenClaw übersetzt Codex-Plugins nicht in synthetische
dynamische OpenClaw-Tools vom Typ `codex_plugin_*`.

`codexPlugins` wirkt sich nur auf Sitzungen aus, die das native Codex-Harness
auswählen. Es hat keine Auswirkung auf Ausführungen mit dem integrierten Harness,
normale Ausführungen des OpenAI-Providers, ACP-Konversationsbindungen oder andere
Harnesses.

Minimale migrierte Konfiguration:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          codexPlugins: {
            enabled: true,
            allow_destructive_actions: true,
            plugins: {
              "google-calendar": {
                enabled: true,
                marketplaceName: "openai-curated",
                pluginName: "google-calendar",
              },
            },
          },
        },
      },
    },
  },
}
```

Die Thread-App-Konfiguration wird berechnet, wenn OpenClaw eine
Codex-Harness-Sitzung einrichtet oder eine veraltete Codex-Thread-Bindung ersetzt;
sie wird nicht bei jedem Turn neu berechnet. Verwenden Sie nach einer Änderung von
`codexPlugins` `/new` oder `/reset`, oder starten Sie
das Gateway neu, damit zukünftige Codex-Harness-Sitzungen mit dem aktualisierten
App-Satz beginnen.

Informationen zu Migrationseignung, App-Inventar, Richtlinien für destruktive
Aktionen, Rückfragen und Diagnosen nativer Plugins finden Sie unter
[Native Codex-Plugins](/de/plugins/codex-native-plugins).

Der App- und Plugin-Zugriff auf OpenAI-Seite wird durch das angemeldete
Codex-Konto und bei Business- und Enterprise/Edu-Workspaces durch die
App-Steuerung des Workspace kontrolliert. Eine Übersicht von OpenAI zur Konto- und
Workspace-Steuerung finden Sie unter
[Codex mit Ihrem ChatGPT-Tarif verwenden](https://help.openai.com/en/articles/11369540-using-codex-with-your-chatgpt-plan).

## Computer Use

Für Computer Use gibt es eine eigene Einrichtungsanleitung:
[Codex Computer Use](/de/plugins/codex-computer-use).

Kurzfassung: OpenClaw liefert die Desktop-Steuerungs-App nicht mit und führt
Desktop-Aktionen nicht selbst aus. Es bereitet den Codex-App-Server vor, überprüft,
ob der MCP-Server `computer-use` verfügbar ist, und überlässt Codex anschließend
während Turns im Codex-Modus die nativen MCP-Tool-Aufrufe.

## Laufzeitgrenzen

Das Codex-Harness ändert nur den eingebetteten Agent-Executor auf niedriger Ebene.

- Dynamische OpenClaw-Tools werden unterstützt. Codex fordert OpenClaw zur
  Ausführung dieser Tools auf, sodass OpenClaw Teil des Ausführungspfads bleibt.
- Codex-native Shell-, Patch-, MCP- und native App-Tools gehören Codex.
  OpenClaw kann ausgewählte native Ereignisse über das unterstützte Relay
  beobachten oder blockieren, schreibt die Argumente nativer Tools jedoch nicht um.
- Codex ist für die native Compaction zuständig. OpenClaw hält einen
  Transkriptspiegel für den Kanalverlauf, die Suche, `/new`,
  `/reset` sowie zukünftige Modell- oder Harness-Wechsel vor, ersetzt die
  Codex-Compaction jedoch nicht durch eine OpenClaw- oder
  Kontext-Engine-Zusammenfassung.
- Medienerzeugung, Medienanalyse, TTS, Genehmigungen und Ausgaben von
  Messaging-Tools laufen weiterhin über die entsprechenden
  OpenClaw-Provider-/Modelleinstellungen.
- `tool_result_persist` gilt für OpenClaw-eigene
  Transkript-Tool-Ergebnisse, nicht für Codex-native Tool-Ergebnisdatensätze.

Informationen zu Hook-Ebenen, unterstützten V1-Oberflächen, nativer
Berechtigungsverarbeitung, Warteschlangensteuerung, dem Upload von
Codex-Feedback und Einzelheiten zur Compaction finden Sie unter
[Codex-Harness-Laufzeit](/de/plugins/codex-harness-runtime).

## Fehlerbehebung

**Codex wird nicht als normaler `/model`-Provider angezeigt:** Dies ist
bei neuen Konfigurationen zu erwarten. Wählen Sie ein `openai/gpt-*`-Modell,
aktivieren Sie `plugins.entries.codex.enabled` und prüfen Sie, ob `plugins.allow`
`codex` ausschließt.

**OpenClaw verwendet das integrierte Harness anstelle von Codex:** Vergewissern
Sie sich, dass die effektive Route exakt eine offizielle HTTPS-Route für Platform
Responses oder ChatGPT Responses ist, keine selbst definierte
Anfrageüberschreibung enthält und dass das Codex-Plugin installiert und aktiviert
ist. Das Präfix `openai/gpt-*` allein reicht nicht aus. Legen Sie für einen
strikten Nachweis während des Tests beim Provider oder Modell
`agentRuntime.id: "codex"` fest; erzwungenes Codex schlägt fehl, anstatt auf eine
Ausweichlösung zurückzugreifen, wenn die Route oder das Harness inkompatibel ist.

**Die OpenAI-Codex-Laufzeit weicht auf den API-Schlüsselpfad aus:** Erfassen Sie
einen redigierten Gateway-Auszug, der Modell, Laufzeit, ausgewählten Provider und
Fehler zeigt. Bitten Sie betroffene Mitwirkende, diesen schreibgeschützten Befehl
auf ihrem OpenClaw-Host auszuführen:

```bash
(
  pattern='openai/gpt-5\.[45]|openai[-]codex|agentRuntime(\.id)?|harnessRuntime|Runtime: OpenAI Codex|legacy OpenAI Codex prefix|resolveSelectedOpenAIRuntimeProvider|candidateProvider[": ]+openai|status[": ]+401|Incorrect API key|No API key|api-key path|API-key path|OAuth'

  if ls /tmp/openclaw/openclaw-*.log >/dev/null 2>&1; then
    grep -E -i -n "$pattern" /tmp/openclaw/openclaw-*.log 2>/dev/null || true
  else
    journalctl --user -u openclaw-gateway --since today --no-pager 2>/dev/null \
      | grep -E -i "$pattern" || true
  fi
) | sed -E \
    -e 's/(Authorization: Bearer )[A-Za-z0-9._~+\/-]+/\1[REDACTED]/Ig' \
    -e 's/(Bearer )[A-Za-z0-9._~+\/-]+/\1[REDACTED]/Ig' \
    -e 's/(api[_ -]?key[=: ]+)[^ ,}"]+/\1[REDACTED]/Ig' \
    -e 's/(OPENAI_API_KEY[=: ]+)[^ ,}"]+/\1[REDACTED]/Ig' \
    -e 's/sk-[A-Za-z0-9_-]{12,}/sk-[REDACTED]/g' \
    -e 's/[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}/[EMAIL-REDACTED]/g' \
  | tail -200
```

Nützliche Auszüge enthalten normalerweise `openai/gpt-5.6-sol` oder
`openai/gpt-5.6-luna`, `Runtime: OpenAI Codex`, `agentRuntime.id` oder
`harnessRuntime`, `candidateProvider: "openai"` sowie ein Ergebnis vom Typ
`401`, `Incorrect API key` oder `No API key`. Eine korrigierte
Ausführung sollte anstelle eines einfachen Fehlers des OpenAI-API-Schlüssels den
OpenAI-OAuth-Pfad anzeigen.

**Die Konfiguration enthält weiterhin veraltete Codex-Modellreferenzen:** Führen Sie `openclaw doctor --fix` aus.
Doctor schreibt veraltete Modellreferenzen in `openai/*` um, entfernt überholte Laufzeit-Pins für Sitzungen und
den gesamten Agenten und behält bestehende Überschreibungen der Authentifizierungsprofile bei.

**Der App-Server wird abgelehnt:** Verwenden Sie über das gebündelte `0.145.0` einen stabilen Codex-App-Server aus `0.143.0`.
Vorabversionen, Versionen mit Build-Suffix und neuere
nicht validierte Releases werden abgelehnt, da OpenClaw generierte Schemas
gegen die gebündelte App-Server-Version validiert.

**`/codex status` kann keine Verbindung herstellen:** Prüfen Sie, ob das Plugin `codex`
aktiviert ist, ob `plugins.allow` es enthält, wenn eine Positivliste
konfiguriert ist, und ob benutzerdefinierte `appServer.command`, `url`, `authToken` oder
Header gültig sind.

**Der Codex-App-Server verwendet zu viel Arbeitsspeicher:** Unterscheiden Sie zunächst die beiden Prozesse.
OpenClaw führt den lokalen Codex-App-Server als separaten untergeordneten Rust-Prozess aus.
`NODE_OPTIONS=--max-old-space-size=...` ändert nur den Node.js-V8-Heap
des Gateways; Codex wird dadurch weder begrenzt noch vergrößert. Verwaltete Gateway-Installationen wählen bereits
einen adaptiven V8-Heap, und seine Vergrößerung kann weniger Host-Arbeitsspeicher für Codex übrig lassen. Verwenden Sie
[Fehlerbehebung beim Gateway-Arbeitsspeicher](/de/gateway/troubleshooting#gateway-exits-during-high-memory-use)
bei Speicherdruck auf das Gateway und prüfen Sie den Host- oder Container-Arbeitsspeicher für den untergeordneten Codex-Prozess.

Das gebündelte Codex hat weder ein Heap- noch ein RSS-Limit und keine konfigurierbare
Verzögerung zum Entladen bei Inaktivität. Nachdem sich der letzte Client abgemeldet hat, kann ein inaktiver Thread
bis zu 30 Minuten geladen bleiben. Reduzieren Sie auf Hosts mit begrenzten Ressourcen die Auffächerung nativer Codex-Subagenten,
bevor Sie den Gateway-Heap vergrößern:

```json5
{
  plugins: {
    entries: {
      codex: {
        config: {
          appServer: {
            args: ["-c", "agents.max_threads=3", "app-server", "--listen", "stdio://"],
          },
        },
      },
    },
  },
}
```

Diese Einstellung begrenzt native untergeordnete Threads für das gebündelte standardmäßige
Multi-Agent-Backend von Codex. Wenn Sie Codex Multi-Agent v2 ausdrücklich aktivieren, verwenden Sie stattdessen
`features.multi_agent_v2.max_concurrent_threads_per_session=3`; das v2-
Limit schließt den Root-Thread ein und kann nicht mit `agents.max_threads` kombiniert werden.
Um Codex mehr Spielraum zu geben, erhöhen Sie die Arbeitsspeicherzuweisung
des Hosts, Containers oder der cgroup. Ein hartes Betriebssystemlimit kann Codex beenden, statt Gegendruck auszuüben.

**Die Modellerkennung ist langsam:** Verringern Sie
`plugins.entries.codex.config.discovery.timeoutMs` oder deaktivieren Sie die Erkennung.
Siehe [Referenz zum Codex-Harness](/de/plugins/codex-harness-reference#model-discovery).

**Der WebSocket-Transport schlägt sofort fehl:** Prüfen Sie `appServer.url`,
`authToken`, die Header und ob der entfernte App-Server dieselbe Codex-
App-Server-Protokollversion verwendet. Der Codex-WebSocket-Transport bleibt experimentell
und wird nicht unterstützt; bevorzugen Sie verwaltetes stdio oder den lokalen Unix-Steuerungssocket.

**Native Shell- oder Patch-Werkzeuge werden mit `Native hook relay
unavailable` blockiert:** Der Codex-Thread versucht weiterhin, eine native Hook-Relay-
ID zu verwenden, die nicht mehr bei OpenClaw registriert ist. Dies ist ein Transportproblem des nativen Codex-Hooks
und kein Fehler des ACP-Backends, Providers, von GitHub oder eines Shell-Befehls.
Starten Sie mit `/new` oder `/reset` eine neue Sitzung im betroffenen Chat
und versuchen Sie anschließend erneut einen harmlosen Befehl. Wenn dies einmal funktioniert, aber der nächste native Werkzeugaufruf
erneut fehlschlägt, behandeln Sie `/new` nur als vorübergehende Problemumgehung: Kopieren Sie den
Prompt nach einem Neustart des Codex-App-Servers oder
OpenClaw Gateways in eine neue Sitzung, damit alte Threads verworfen und native Hook-Registrierungen
neu erstellt werden.

**Codex-Werkzeugaufrufe erzeugen zu viele kurzlebige Hook-Prozesse:** Legen Sie
`plugins.entries.codex.config.appServer.loopDetectionPreToolUseRelay: false` fest
und starten Sie das Gateway neu. Dadurch wird nur der Codex-Unterprozess `PreToolUse`
deaktiviert, der für die OpenClaw-Schleifenerkennung und dessen Marker für fehlende Richtlinien verwendet wird. Erforderliche
`before_tool_call` und Richtlinien-Relays für vertrauenswürdige Werkzeuge bleiben aktiviert.

**Ein Nicht-Codex-Modell verwendet das integrierte Harness:** Dies ist zu erwarten, sofern die Laufzeitrichtlinie des Providers
oder Modells es nicht an ein anderes Harness weiterleitet. Einfache Referenzen von Nicht-OpenAI-
Providern verbleiben im Modus `auto` auf ihrem normalen Provider-Pfad.

**Computer Use ist installiert, aber die Werkzeuge werden nicht ausgeführt:** Prüfen Sie
`/codex computer-use status` in einer neuen Sitzung. Wenn ein Werkzeug
`Native hook relay unavailable` meldet, verwenden Sie die oben beschriebene Wiederherstellung des nativen Hook-Relays.
Siehe [Codex Computer Use](/de/plugins/codex-computer-use#troubleshooting).

## Verwandte Themen

- [Referenz zum Codex-Harness](/de/plugins/codex-harness-reference)
- [Laufzeit des Codex-Harness](/de/plugins/codex-harness-runtime)
- [Codex-Überwachung](/de/plugins/codex-supervision)
- [Native Codex-Plugins](/de/plugins/codex-native-plugins)
- [Codex Computer Use](/de/plugins/codex-computer-use)
- [Agentenlaufzeiten](/de/concepts/agent-runtimes)
- [Modell-Provider](/de/concepts/model-providers)
- [OpenAI-Provider](/de/providers/openai)
- [Hilfe zu OpenAI Codex](https://help.openai.com/en/collections/14937394-codex)
- [Agent-Harness-Plugins](/de/plugins/sdk-agent-harness)
- [Plugin-Hooks](/de/plugins/hooks)
- [Diagnoseexport](/de/gateway/diagnostics)
- [Status](/de/cli/status)
- [Tests](/de/help/testing-live#live-codex-app-server-harness-smoke)
