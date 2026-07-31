---
read_when:
    - Sie haben die Inferenz eingerichtet und möchten, dass OpenClaw den Rest konfiguriert
    - Sie müssen OpenClaw mit dem lokalen Einrichtungsagenten überprüfen oder reparieren.
    - Sie konzipieren oder aktivieren den Rettungsmodus für Nachrichtenkanäle
summary: CLI-Referenz und Sicherheitsmodell für den inferenzgestützten OpenClaw-Einrichtungs- und Reparaturassistenten
title: OpenClaw-Einrichtungsagent
x-i18n:
    generated_at: "2026-07-26T18:17:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9578d1493ff514ea6dd07dae995bf83443e9e17f2c2134bc801faa45254615bf
    source_path: cli/openclaw.md
    workflow: 16
---

# `openclaw setup`

OpenClaw wird mit einem integrierten Systemagenten ausgeliefert – er spricht als „OpenClaw“ – für
lokale Einrichtung, Reparatur und Konfiguration (früher Crestodian genannt). Er startet erst, nachdem das tatsächlich verwendete Standardmodell einen echten Turn abgeschlossen hat.
Bei Neuinstallationen wird zuerst die Inferenz eingerichtet; eine fehlerhafte Konfiguration verbleibt im
klassischen Doctor-Ablauf.

## Wann er startet

Die Ausführung von `openclaw` ohne Unterbefehl wird abhängig vom Konfigurationszustand weitergeleitet:

- Konfiguration fehlt oder ist vorhanden, enthält aber keine benutzerdefinierten Einstellungen (leer oder nur Schlüssel vom Typ `$schema`/`meta`): startet das geführte Onboarding mit Live-KI-Verifizierung.
- Konfiguration ist vorhanden, besteht aber die Validierung nicht: startet das klassische Onboarding, das die Probleme meldet und Sie an `openclaw doctor` verweist.
- Konfiguration ist vorhanden und gültig: öffnet die normale Agenten-TUI. Ein erreichbarer
  konfigurierter Gateway, dessen Standardagent ein Modell besitzt, wechselt direkt zu dieser Oberfläche,
  ohne Onboarding oder OpenClaw. Verwenden Sie später `/openclaw` innerhalb der TUI oder führen Sie
  `openclaw setup` direkt aus, um OpenClaw aufzurufen.

Beim Ausführen von `openclaw setup` wird zuerst das konfigurierte Standardmodell live getestet. Ein erfolgreicher Turn startet OpenClaw. Bei einem interaktiven Fehler wird die geführte Inferenzeinrichtung geöffnet und nach erfolgreicher Prüfung eines Kandidaten an OpenClaw übergeben. Einmalige, JSON- und andere nicht interaktive Anfragen schlagen mit der Anweisung fehl, `openclaw onboard` auszuführen, wenn keine Inferenz verfügbar ist. `openclaw --help` und `openclaw --version` behalten ihre normalen schnellen Pfade bei.

Das nicht interaktive alleinige Ausführen von `openclaw` (ohne TTY) wird mit einer kurzen Meldung beendet, statt die Hilfe des Stammbefehls auszugeben: Bei einer neuen oder ungültigen Installation verweist sie auf das nicht interaktive Onboarding, bei gültiger Konfiguration auf `openclaw agent --local ...`.

`openclaw onboard --modern` bleibt ein Kompatibilitätsalias für OpenClaw, verwendet jedoch dieselbe Inferenzprüfung: Eine funktionierende Inferenz öffnet den Chat, interaktive Fehler starten die geführte Inferenzeinrichtung und nicht interaktive Fehler werden mit Onboarding-Hinweisen beendet. `openclaw onboard --classic` öffnet den vollständigen Schritt-für-Schritt-Assistenten.

## Was OpenClaw anzeigt

Das interaktive OpenClaw öffnet dieselbe TUI-Shell wie `openclaw tui`, jedoch mit einem OpenClaw-Chat-Backend. Die Begrüßung beim Start umfasst:

- Gültigkeit der Konfiguration und den Standardagenten
- das verifizierte Modell, das OpenClaw verwendet
- die Erreichbarkeit des Gateways gemäß der ersten Startprüfung
- die nächste empfohlene Debugging-Aktion

Es gibt keine Secrets aus und lädt nicht allein zum Starten die CLI-Befehle von Plugins.

Verwenden Sie `status` für die detaillierte Bestandsübersicht: Konfigurationspfad, Dokumentations-/Quellpfade, lokale CLI-Prüfungen, Vorhandensein von Schlüsseln/Tokens, Agenten, Modell und Gateway-Details.

OpenClaw verwendet dieselbe Referenzermittlung wie reguläre Agenten: In einem Git-Checkout verweist es auf lokale `docs/` und den Quellbaum; in einer npm-Installation verwendet es die gebündelte Dokumentation und verlinkt auf [https://github.com/openclaw/openclaw](https://github.com/openclaw/openclaw), mit dem Hinweis, den Quellcode zu prüfen, wenn die Dokumentation nicht ausreicht.

## Beispiele

```bash
openclaw
openclaw setup
openclaw setup --json
openclaw setup --message "models"
openclaw setup --message "validate config"
openclaw setup --message "setup workspace ~/Projects/work" --yes
openclaw setup --message "set default model openai/gpt-5.6" --yes
openclaw onboard --modern
```

Innerhalb der OpenClaw-TUI:

```text
Status
Systemzustand
Doctor
Konfiguration validieren
Einrichtung
Arbeitsbereich ~/Projects/work einrichten
config set gateway.port 19001
config set-ref gateway.auth.token env OPENCLAW_GATEWAY_TOKEN
Gateway-Status
Gateway neu starten
Agenten
Agent work mit Arbeitsbereich ~/Projects/work erstellen
Modelle
Modell-Provider konfigurieren
Standardmodell auf openai/gpt-5.6 setzen
Kanäle
Kanalinformationen zu Slack
Slack verbinden
Kanalassistent für Slack öffnen
Plugins auflisten
Plugins nach Slack durchsuchen
Plugin clawhub:openclaw-codex-app-server installieren
mit Agent work sprechen
mit Agent für ~/Projects/work sprechen
Audit
Beenden
```

## Vorgänge und Genehmigung

OpenClaw verwendet typisierte Vorgänge, statt die Konfiguration ad hoc zu bearbeiten.

Schreibgeschützte Vorgänge werden sofort ausgeführt: Übersicht anzeigen, Agenten auflisten, installierte Plugins auflisten, ClawHub-Plugins durchsuchen, Modell-/Backend-Status anzeigen, Status-/Systemzustandsprüfungen ausführen, Erreichbarkeit des Gateways prüfen, Doctor ohne interaktive Korrekturen ausführen, Konfiguration validieren, Pfad des Audit-Protokolls anzeigen.

Auch der Start der geführten Kanaleinrichtung (`connect telegram`) wird sofort ausgeführt. Der Assistent erfasst ausdrückliche Antworten und ist für die daraus resultierenden Schreibvorgänge zuständig.

Persistente Vorgänge erfordern eine Genehmigung im Gespräch (oder `--yes` für einen direkten Befehl): Konfiguration schreiben, `config set`, `config set-ref`, Bootstrap für Einrichtung/Onboarding, Standardmodell ändern, Gateway starten/stoppen/neu starten, Agenten erstellen und Plugins installieren.

Doctor-Reparaturen sind innerhalb von OpenClaw nicht verfügbar, da sie den Provider, die Authentifizierung oder die Inferenzroute des Standardagenten umschreiben können, auf der die Sitzung basiert. Beenden Sie OpenClaw und führen Sie `openclaw doctor --fix` in einem Terminal aus. Das schreibgeschützte `doctor` bleibt innerhalb von OpenClaw verfügbar.

Neue Agenten übernehmen die live verifizierte Standardinferenzroute. Die Agenten-IDs `openclaw` und `crestodian` sind für den Systemagenten reserviert und können nicht als normale Agenten erstellt werden. Die außer Betrieb genommene ID bleibt gesperrt, damit eine alte Konfiguration sie nicht beanspruchen kann.

`config set` und `config set-ref` können jede Einstellung ändern, die ein Benutzer ändern kann,
mit einer kurzen ausschließlich für Menschen bestimmten Sperrliste: `$include`, `auth.*`, `env.*`, `models.*`
und `secrets.*` werden weiterhin abgelehnt, da sie Zugangsdaten,
die Einbindung alternativer Konfigurationen oder die Provider-/Katalogdefinitionen enthalten, die für
die Inferenzweiterleitung verwendet werden. Auch die Inferenzweiterleitung selbst ist geschützt: Routen
des Standardmodells (Modell-/Parameter-/Laufzeitfelder von `agents.defaults`) und die Routingfelder
des Agenten, der die aktive Standardroute bereitstellt, werden ebenso abgelehnt wie Felder für
Agentenidentität und -topologie (`id`, `agentDir`, `default`). Routingfelder für
andere Agenten bleiben nach Genehmigung beschreibbar. Gateway- und Kanalauthentifizierung bleiben
normale Konfigurationsoberflächen. Verwenden Sie `set default model <provider/model>` für eine
bereits konfigurierte Route; die Route wird vor dem Speichern live getestet. Um
den Provider-/Authentifizierungszugriff zu konfigurieren oder zu reparieren, beenden Sie OpenClaw und führen Sie
`openclaw onboard` aus.

Schreibvorgänge über `plugins.entries.<id>.*` (Aktivieren/Deaktivieren/Konfigurieren installierter Plugins)
sind zulässig, sofern das betreffende Plugin nicht die aktive Inferenzroute bereitstellt. Quellen für die
Plugin-Installation und die Laderichtlinie behalten ihre Vertrauensgrenze im typisierten
Plugin-Installationsablauf. Die Deinstallation des Plugins, das die Route bereitstellt, wird
aus demselben Grund abgelehnt; beenden Sie OpenClaw und führen Sie
`openclaw plugins uninstall <id>` in einem Terminal aus.

Die Genehmigung erfolgt mit Ihren eigenen Worten: Eindeutige Antworten („ja“, „sicher“, „fortfahren“, „jetzt nicht“) werden anhand einer geschlossenen deterministischen Liste ausgewertet. Wenn die konfigurierte Route einen separaten Completion-Aufruf unterstützt, können andere Antworten ausschließlich anhand Ihrer Nachricht und des ausstehenden Vorschlags klassifiziert werden – niemals durch das Gesprächsmodell selbst, das sich nicht selbst genehmigen kann. Nicht klassifizierte oder mehrdeutige Antworten lassen den Vorschlag ausstehend, und im Gespräch wird erneut nachgefragt.

### Änderungsverlauf

Die Seite „OpenClaw fragen“ kann kürzlich angewendete Vorgänge des Systemagenten, Doctor-
Migrationen, Konfigurationsschreibvorgänge über Einstellungen und CLI sowie manuelle Änderungen an
`openclaw.json` anzeigen. Das Konfigurationsjournal erkennt externe Änderungen, während der Gateway
sie überwacht, während eines OpenClaw-eigenen Schreibvorgangs oder beim nächsten Start nach einer
Offline-Änderung.

Der Verlauf wird in der Tabelle `diagnostic_events` der gemeinsam genutzten
Datenbank `~/.openclaw/state/openclaw.sqlite` unter den Geltungsbereichen `system-agent-audit`
und `config-audit` gespeichert. Jeder Geltungsbereich bewahrt seine neuesten 50,000 Datensätze auf.
Ermittlungs- und schreibgeschützte Vorgänge sind nicht enthalten. Secrets erscheinen niemals im
Änderungsverlauf; Datensätze des Konfigurationsjournals enthalten geänderte Pfade statt Konfigurationswerten,
und der Wertevergleich verwendet geschützte Fingerabdrücke.

Die Kanaleinrichtung kann als gehostete Unterhaltung ausgeführt werden, bis ein Secret benötigt wird. Die
lokale OpenClaw-TUI akzeptiert keine vertraulichen Antworten des Assistenten, da Eingaben im Terminal-
Chat sichtbar sind. Sie bietet sofort `open channel wizard` an, wobei
der ausgewählte Kanal an den maskierten Terminalassistenten übergeben wird; Sie können auch später
`openclaw channels add --channel <channel>` ausführen.

### Wechsel zur maskierten Kanaleinrichtung

Der lokale Chat kann die Steuerung an den maskierten Kanalassistenten übergeben:

```text
Kanalassistent für Slack öffnen
Kanalinformationen zu Slack
```

`open channel wizard for <channel>` öffnet die maskierte Kanaleinrichtung, nachdem die Chat-
TUI geschlossen wurde. Verwenden Sie zuerst `channel info <channel>`, um die Kanalbezeichnung, den Einrichtungsstatus,
eine Zusammenfassung der Voraussetzungen und den Dokumentationslink anzuzeigen.

OpenClaw ändert innerhalb seiner eigenen Sitzung niemals den Provider-/Authentifizierungszugriff: Die
Sitzung hängt bereits von dieser Inferenzroute ab. Für die Einrichtung oder
Reparatur des Modell-Providers gibt `configure model provider` Hinweise zum Beenden/Onboarding zurück, ohne
einen Assistenten zu starten oder die Konfiguration zu schreiben. Beenden Sie OpenClaw und führen Sie `openclaw
onboard` aus; das Onboarding stellt die Zugangsdaten bereit und speichert nur eine Route, die
einen echten Live-Turn abschließt. Starten Sie OpenClaw erneut, nachdem das Onboarding erfolgreich abgeschlossen wurde.

## Einrichtungs-Bootstrap

`setup` konfiguriert den verbleibenden Zustand des Arbeitsbereichs und Gateways, nachdem das geführte Onboarding die Inferenz bereits eingerichtet hat. Es schreibt ausschließlich über typisierte Konfigurationsvorgänge und bittet zuerst um Genehmigung.

```text
Einrichtung
Arbeitsbereich ~/Projects/work einrichten
```

`setup` behält das verifizierte tatsächlich verwendete Modell bei. Es konfiguriert oder
ersetzt die Inferenz nicht.

Wenn die Inferenz fehlt oder ihre Live-Prüfung fehlschlägt, beenden Sie OpenClaw und führen Sie `openclaw onboard` aus. Das geführte Onboarding versucht zuerst das konfigurierte Modell, dann authentifizierte Abonnement-CLIs, API-Schlüssel und die übrigen unterstützten CLIs; es fordert von jedem Kandidaten eine echte Antwort an und speichert nur eine erfolgreiche Route dauerhaft. OpenClaw startet unmittelbar nach dieser Grenze und kann anschließend den Arbeitsbereich, Gateway, Kanäle, Agenten, Plugins und weitere optionale Funktionen konfigurieren.

Die macOS-App überspringt diese Abfolge vollständig, wenn sie einen konfigurierten Gateway
erreicht, dessen Standardagent bereits über ein konfiguriertes Modell verfügt; sie öffnet die normale
Agentenoberfläche.
Bei einem neuen oder unvollständigen Gateway führt die App die Inferenzabfolge über
die Gateway-Methoden `openclaw.setup.detect` und `openclaw.setup.activate` aus:
Die Erkennung listet jedes gefundene Kandidaten-Backend auf, die Aktivierung testet einen
Kandidaten live (eine echte Completion mit „reply with OK“) und speichert erst nach erfolgreicher Prüfung das Modell,
die Zugangsdaten und den für diese Route benötigten Provider-/Laufzeitzustand dauerhaft. Die Standardwerte für Arbeitsbereich und Gateway bleiben OpenClaw vorbehalten. Ein fehlschlagender Kandidat
ändert niemals die Konfiguration; die App arbeitet die Abfolge automatisch ab und
bietet schließlich einen manuellen Schlüssel-/Token-Schritt an, der anhand der aktiven
Textinferenz-Provider-Plugins des Gateways ausgefüllt wird. Der ausgewählte Provider bestimmt sein Einstiegsmodell
und seine Konfiguration, und die Zugangsdaten werden vor dem Speichern auf dieselbe Weise verifiziert.

Die Codex-Überwachung und andere optionale Plugin-Funktionen bleiben außerhalb dieser
Transaktion zur Inferenzaktivierung. Konfigurieren Sie sie erst, nachdem die Inferenz
funktioniert und OpenClaw gestartet wurde; bestehende Plugin-Richtlinien und ausdrückliche
Deaktivierungen der Überwachung bleiben während der Inferenzeinrichtung unverändert.

## KI-Unterhaltung

Die freie Unterhaltung im interaktiven OpenClaw läuft über dieselbe Agentenschleife wie bei regulären OpenClaw-Agenten, beschränkt auf ein einziges OpenClaw-Autoritätstool der Ringstufe null, `openclaw`, das die typisierten Vorgänge umschließt. Leseaktionen werden frei ausgeführt, Mutationen erfordern Ihre Genehmigung im Gespräch für genau diesen Vorgang (siehe Vorgänge und Genehmigung), und jeder angewendete Schreibvorgang wird auditiert und erneut validiert. Die Agentensitzung bleibt bestehen, sodass OpenClaw über echtes Gedächtnis über mehrere Turns verfügt. Wenn die verifizierte Inferenzroute später nicht mehr funktioniert, kehren Sie zu `openclaw onboard` zurück und reparieren Sie sie, bevor Sie fortfahren.

Der Host wandelt natürlichsprachliche Anfragen nicht selbst in Vorgänge um. Freie
Nachrichten – einschließlich befehlsähnlichen Textes und Fragen wie „Warum wurde mein
Gateway gestoppt?“ – werden an die KI gesendet, die die Anfrage über das Tool
`openclaw` einem typisierten Vorgang zuordnen kann.

Wenn eine Mutation aussteht, werden nur eindeutige Genehmigungs- oder Ablehnungsformulierungen aus einer
geschlossenen Liste ohne Schlussfolgerungen aufgelöst. Mehrdeutige Zustimmung wird an einen
separat konfigurierten Completion-Aufruf weitergeleitet; andernfalls wird nach dem Fail-Closed-Prinzip abgebrochen. Strukturierte
Assistentenfelder und die exakte Host-Navigation sind UI-Steuerelemente und keine Verarbeitung natürlichsprachlicher
Operationen. Eine Ausnahme für die Geheimnishygiene ist besonders wichtig: Ein
exaktes `config set` auf einem sensiblen Pfad (Token, Schlüssel, Passwörter) erreicht niemals
ein Modell. Der Host erstellt einen redigierten Vorschlag, und der Wert wird im
für die KI sichtbaren Verlauf maskiert. Verwenden Sie für Geheimnisse vorzugsweise `config set-ref <path> env <ENV_VAR>`.

Der Rettungsmodus für Nachrichtenkanäle verwendet niemals den modellgestützten Planer. Die Remote-Rettung bleibt deterministisch, damit ein defekter oder kompromittierter normaler Agent-Pfad nicht als Konfigurationseditor verwendet werden kann.

### Vertrauensmodell des CLI-Harnesses

Eingebettete Laufzeitumgebungen und das Codex-App-Server-Harness erzwingen die Ring-Zero-
Beschränkung direkt: Der Lauf enthält eine OpenClaw-Tool-Zulassungsliste, die nur
das Tool `openclaw` umfasst. Für Codex deaktiviert OpenClaw außerdem Umgebungen, native
Ausführung, Multi-Agent-, Ziel-, App-/Plugin-, Skill-/MCP-, Websuche- und
`request_user_input`-Oberflächen für diesen Lauf. Codex fügt weiterhin sein inaktives natives Dienstprogramm `update_plan`
ein; es kann die temporäre Checkliste des Modells aktualisieren, aber keine Dateien
oder die OpenClaw-Konfiguration schreiben. CLI-Harnesse verwenden die Zulassungsliste von OpenClaw nicht,
daher lässt OpenClaw nur Backends zu, deren eigener Vertrag zur Tool-Auswahl
dieselbe Beschränkung nachweisen kann:

- Auswählbare Backends, einschließlich Claude Code, werden mit einer leeren Auswahl nativer Tools
  und einem MCP-Tool, `openclaw`, gestartet. Die generierte MCP-Konfiguration von Claude wird
  mit `--strict-mcp-config` angewendet, sodass keine anderen MCP-Server geladen werden.
- Backends, die keine nativen Tools deklarieren, erhalten denselben dedizierten OpenClaw-
  MCP-Server.
- Backends mit stets aktiven oder unbekannten nativen Tools brechen vor der Inferenz nach dem Fail-Closed-Prinzip ab; sie
  können keine OpenClaw-Sitzung hosten.

Nur OpenClaw-Sitzungen erhalten den openclaw-MCP-Server; normale Agent-Läufe
sehen dieses Tool niemals. Auswählbare CLI-Backends ohne native Tools und API-Schlüssel-Modelle
erzwingen daher die buchstäbliche Ein-Tool-Schleife. Codex-App-Server-Modelle erzwingen
ein einzelnes OpenClaw-Autoritäts-Tool plus das inaktive native Planungsdienstprogramm. In allen
drei Fällen bleiben Setup-Schreibvorgänge auf den auditierten Genehmigungsvertrag
von OpenClaw beschränkt.

Gemini CLI bleibt für normale Agents verfügbar, kann jedoch die
für das Inferenz-Gate erforderliche toolfreie Prüfung nicht erzwingen und daher OpenClaw nicht hosten.

## Zu einem Agent wechseln

Verwenden Sie einen natürlichsprachlichen Selektor, um OpenClaw zu verlassen und die normale TUI zu öffnen:

```text
mit Agent sprechen
mit Arbeits-Agent sprechen
zum Haupt-Agent wechseln
```

`openclaw tui`, `openclaw chat` und `openclaw terminal` öffnen die normale Agent-TUI direkt; sie starten OpenClaw nicht. Nach dem Wechsel in die normale TUI kehrt `/openclaw` zu OpenClaw zurück, optional mit einer Folgeanforderung:

```text
/openclaw
/openclaw Gateway neu starten
```

## Nachrichten-Rettungsmodus

Der Nachrichten-Rettungsmodus ist der Nachrichtenkanal-Einstiegspunkt für OpenClaw: Verwenden Sie ihn, wenn Ihr normaler Agent ausgefallen ist, aber ein vertrauenswürdiger Kanal (zum Beispiel WhatsApp) weiterhin Befehle empfängt.

Dies ist ein deterministischer Handler für Notfallbefehle, nicht der dialogorientierte
OpenClaw-Agent. Er initialisiert kein neues Setup und lockert das Inferenz-
Gate für den OpenClaw-Chat nicht.

Unterstützter Befehl: `/openclaw <request>`. Die Rettung akzeptiert ausschließlich die exakte eingegebene Befehlsgrammatik — natürliche Sprache wird mit einem Hinweis abgelehnt, niemals als Operation interpretiert, und es wird niemals ein Modell konsultiert.

```text
Sie, in einer vertrauenswürdigen Eigentümer-DM: /openclaw status
OpenClaw: OpenClaw-Rettungsmodus. Gateway erreichbar: nein. Konfiguration gültig: nein.
Sie: /openclaw restart gateway
OpenClaw: Plan: Gateway neu starten. Antworten Sie mit /openclaw yes, um ihn anzuwenden.
Sie: /openclaw yes
OpenClaw: Angewendet. Audit-Eintrag geschrieben.
```

Die Erstellung eines Agents kann auch lokal oder über die Rettung in die Warteschlange gestellt werden:

```text
Agent work Arbeitsbereich ~/Projects/work Modell openai/gpt-5.6-sol erstellen
/openclaw create agent work workspace ~/Projects/work
```

Bei der Agent-Erstellung darf nur das aktuell live verifizierte Standardmodell angegeben werden. Lassen Sie das
Modell weg, um diese Route zu übernehmen.

Die Remote-Rettung ist eine Admin-Oberfläche und muss wie eine Remote-Konfigurationsreparatur behandelt werden, nicht wie ein normaler Chat.

Sicherheitsvertrag für die Remote-Rettung:

- Deaktiviert, wenn Sandboxing für den Agent/die Sitzung aktiv ist; OpenClaw verweigert die Remote-Rettung und verweist auf die lokale CLI-Reparatur.
- Der standardmäßig wirksame Zustand ist `auto`: Remote-Rettung nur im vertrauenswürdigen YOLO-Betrieb zulassen, in dem die Laufzeit bereits über uneingeschränkte lokale Befugnisse ohne Sandbox verfügt (`tools.exec.security` wird zu `full` aufgelöst und `tools.exec.ask` wird zu `off` aufgelöst, mit dem Sandbox-Modus `off`).
- Erfordert eine explizite Eigentümeridentität; keine Platzhalter-Absenderregeln, offene Gruppenrichtlinie, nicht authentifizierten Webhooks oder anonymen Kanäle.
- Die Rettung ist auf Eigentümer-DMs beschränkt.
- Plugin-Suche und -Auflistung sind schreibgeschützt. Die Plugin-Installation ist immer nur lokal möglich (in der Rettung blockiert, selbst wenn sie andernfalls aktiviert ist), da sie ausführbaren Code herunterlädt. Die Deinstallation von Plugins wird sowohl im lokalen OpenClaw als auch in der Rettung verweigert; führen Sie `openclaw plugins uninstall <id>` in einem Terminal aus.
- Die Remote-Rettung kann die lokale TUI nicht öffnen oder zu einer interaktiven Agent-Sitzung wechseln; verwenden Sie für die Agent-Übergabe das lokale `openclaw`.
- Dauerhafte Schreibvorgänge erfordern auch im Rettungsmodus weiterhin eine Genehmigung.
- Ausstehende Genehmigungen können einmal verwendet werden. Jeder neuere Rettungsbefehl für dasselbe Konto, denselben Kanal und denselben Absender widerruft den älteren Plan; eine fehlgeschlagene Ausführung verbraucht die Genehmigung ebenfalls. Senden Sie den Befehl daher erneut, um es noch einmal zu versuchen.
- Jede angewendete Rettungsoperation wird auditiert. Die Nachrichtenkanal-Rettung zeichnet Kanal-, Konto-, Absender- und Quelladressmetadaten auf; Operationen, die die Konfiguration verändern, zeichnen außerdem die Konfigurations-Hashes davor und danach auf.
- Geheimnisse werden niemals ausgegeben. Die Überprüfung von SecretRef meldet die Verfügbarkeit, nicht die Werte.
- Wenn das Gateway aktiv ist, bevorzugt die Rettung typisierte Gateway-Operationen; wenn es ausgefallen ist, verwendet die Rettung nur die minimale lokale Reparaturoberfläche, die nicht von der normalen Agent-Schleife abhängt.

Die Rettungsrichtlinie ist integriert: Sie ist nur verfügbar, wenn die wirksame Laufzeit
YOLO ist, Sandboxing deaktiviert ist und die Anfrage eine Eigentümer-DM ist. Ausstehende Genehmigungen für Schreibvorgänge
laufen nach 15 Minuten ab. `openclaw doctor --fix` entfernt die außer Betrieb genommenen
Konfigurationsblöcke `systemAgent` und `crestodian`.

Die Remote-Rettung wird von der Docker-Lane abgedeckt:

```bash
pnpm test:docker:system-agent-rescue
```

Ein optionaler Live-Smoke-Test der Kanal-Befehlsoberfläche prüft `/openclaw status` sowie einen dauerhaften Genehmigungsdurchlauf durch den Rettungs-Handler:

```bash
pnpm test:live:system-agent-rescue-channel
```

Das durch ein Inferenz-Gate geschützte paketierte einmalige Setup wird abgedeckt durch:

```bash
pnpm test:docker:system-agent-first-run
```

Diese paketierte CLI-Lane startet mit einem leeren Zustandsverzeichnis und weist nach, dass OpenClaw
ohne Inferenz nach dem Fail-Closed-Prinzip abbricht. Anschließend testet und aktiviert sie ein simuliertes Claude über
das paketierte Aktivierungsmodul. Erst danach erreicht eine unscharfe Anfrage den
Planer und wird in ein typisiertes Setup aufgelöst, gefolgt von einmaligen Befehlen, die einen
zusätzlichen Agent erstellen, Discord über die Aktivierung eines Plugins plus Token-
SecretRef konfigurieren, die Konfiguration validieren und das Audit-Protokoll prüfen. Diese Lane liefert unterstützende
Gate-/Operationsnachweise; sie testet weder das interaktive Onboarding noch die
OpenClaw-Agent-/Tool-/Genehmigungskonversation. Das nachfolgende QA-Lab-Szenario leitet
auf dieselbe Docker-Lane um:

```bash
pnpm openclaw qa suite --scenario system-agent-ring-zero-setup
```

## Verwandte Themen

- [CLI-Referenz](/de/cli)
- [Doctor](/de/cli/doctor)
- [TUI](/de/cli/tui)
- [Sandbox](/de/cli/sandbox)
- [Sicherheit](/de/cli/security)
