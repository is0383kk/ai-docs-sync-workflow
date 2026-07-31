---
read_when:
    - Verwenden oder Ändern des exec-Tools
    - Debugging des stdin- oder TTY-Verhaltens
summary: Verwendung des Exec-Tools, stdin-Modi und TTY-Unterstützung
title: Exec-Tool
x-i18n:
    generated_at: "2026-07-26T18:13:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9c16b5122c527c069a4d1a0c1649726073339e95b9084100c1a0f45ebcae759d
    source_path: tools/exec.md
    workflow: 16
---

Führen Sie Shell-Befehle im Arbeitsbereich aus. `exec` ist eine verändernde Shell-Oberfläche: Befehle können Dateien überall dort erstellen, bearbeiten oder löschen, wo das Dateisystem des ausgewählten Hosts oder der Sandbox dies zulässt. Durch das Deaktivieren von OpenClaw-Dateisystemwerkzeugen wie `write`, `edit` oder `apply_patch` wird `exec` nicht schreibgeschützt.

Unterstützt die Ausführung im Vorder- und Hintergrund über `process`. Wenn `process` nicht zulässig ist, wird `exec` synchron ausgeführt und ignoriert `yieldMs`/`background`. Hintergrundsitzungen sind auf den jeweiligen Agenten beschränkt; `process` sieht nur Sitzungen desselben Agenten.

## Parameter

<ParamField path="command" type="string" required>
Auszuführender Shell-Befehl.
</ParamField>

<ParamField path="workdir" type="string" default="cwd">
Arbeitsverzeichnis für den Befehl.
</ParamField>

<ParamField path="env" type="object">
Schlüssel/Wert-Umgebungsüberschreibungen, die mit Vorrang vor der geerbten Umgebung zusammengeführt werden.
</ParamField>

<ParamField path="yieldMs" type="number" default="10000">
Den Befehl nach dieser Verzögerung (ms) automatisch in den Hintergrund verschieben.
</ParamField>

<ParamField path="background" type="boolean" default="false">
Den Befehl sofort im Hintergrund ausführen, statt auf `yieldMs` zu warten.
</ParamField>

<ParamField path="timeout" type="number" default="tools.exec.timeoutSeconds">
Das konfigurierte Zeitlimit für die Ausführung bei diesem Aufruf in Sekunden überschreiben. Gilt für die Ausführung im Vordergrund, im Hintergrund, über `yieldMs`, Gateway, Sandbox und Node `system.run`. `timeout: 0` deaktiviert das Zeitlimit für den Ausführungsprozess bei diesem Aufruf.
</ParamField>

<ParamField path="pty" type="boolean" default="false">
Wenn verfügbar, in einem Pseudoterminal ausführen. Für reine TTY-CLIs, Coding-Agenten und Terminal-Benutzeroberflächen verwenden.
</ParamField>

<ParamField path="host" type="'auto' | 'sandbox' | 'gateway' | 'node'" default="auto">
Ort der Ausführung. `auto` wird bei aktiver Sandbox-Laufzeitumgebung zu `sandbox` aufgelöst, andernfalls zu `gateway`.
</ParamField>

<ParamField path="security" type="'deny' | 'allowlist' | 'full'">
Wird bei normalen Werkzeugaufrufen ignoriert. Die Sicherheit für `gateway`/`node` wird aus `tools.exec.mode` und der Host-Genehmigungsdatei abgeleitet; der Modus mit erhöhten Rechten kann vollständigen Zugriff nur erzwingen, wenn der Betreiber diesen ausdrücklich gewährt.
</ParamField>

<ParamField path="ask" type="'off' | 'on-miss' | 'always'">
Der grundlegende Abfragemodus wird aus `tools.exec.mode` und den Host-Genehmigungen abgeleitet. Bei Modellaufrufen aus einem Kanal wird `ask` pro Aufruf ignoriert, wenn die effektive Host-Abfrage `off` ist; andernfalls kann dadurch nur ein strengerer Modus festgelegt werden.
</ParamField>

<ParamField path="node" type="string">
Node-ID/-Name bei `host=node`.
</ParamField>

<ParamField path="elevated" type="boolean" default="false">
Modus mit erhöhten Rechten anfordern: die Sandbox verlassen und zum konfigurierten Host-Pfad wechseln. `security=full` wird nur erzwungen, wenn der Modus mit erhöhten Rechten zu `full` aufgelöst wird.
</ParamField>

Hinweise:

- `host` akzeptiert nur `auto`, `sandbox`, `gateway` oder `node`. Dies ist keine Auswahl für Hostnamen; hostnamenähnliche Werte werden abgelehnt, bevor der Befehl ausgeführt wird.
- `host=node` pro Aufruf ist von `auto` aus zulässig; `host=gateway` pro Aufruf ist nur zulässig, wenn keine Sandbox-Laufzeitumgebung aktiv ist.
- Ohne zusätzliche Konfiguration funktioniert `host=auto` weiterhin unmittelbar: Ohne Sandbox wird es zu `gateway` aufgelöst; mit einer aktiven Sandbox verbleibt es in der Sandbox.
- `elevated` verlässt die Sandbox und wechselt zum konfigurierten Host-Pfad: standardmäßig `gateway` oder `node`, wenn `tools.exec.host=node` (oder der Sitzungsstandard `host=node` ist). Dies ist nur verfügbar, wenn der Zugriff mit erhöhten Rechten für die aktuelle Sitzung/den aktuellen Provider aktiviert ist.
- Genehmigungen für `gateway`/`node` werden durch die Host-Genehmigungsdatei gesteuert.
- `node` erfordert einen gekoppelten Node (Begleit-App oder Headless-Node-Host). Wenn mehrere Nodes verfügbar sind, legen Sie `exec.node` oder `tools.exec.node` fest, um einen auszuwählen.
- `exec host=node` ist der einzige Pfad zur Shell-Ausführung für Nodes; der veraltete Wrapper `nodes.run` wurde entfernt.
- Auf Nicht-Windows-Hosts verwendet exec `SHELL`, wenn es festgelegt ist; wenn `SHELL` den Wert `fish` hat, wird `bash` (oder `sh`) aus `PATH` bevorzugt, um mit fish inkompatible Bash-Konstrukte zu vermeiden. Falls keines davon vorhanden ist, wird auf `SHELL` zurückgegriffen.
- Auf Windows-Hosts bevorzugt exec die Erkennung von PowerShell 7 (`pwsh`) (Program Files, ProgramW6432, dann PATH) und greift anschließend auf Windows PowerShell 5.1 zurück.
- Auf Nicht-Windows-Gateway-Hosts verwenden mit Bash und Zsh ausgeführte exec-Befehle einen Start-Snapshot. OpenClaw erfasst einbindbare Aliase/Funktionen sowie eine kleine, sichere Gruppe von Umgebungswerten aus Shell-Startdateien in `$OPENCLAW_STATE_DIR/cache/shell-snapshots/` und bindet diesen Snapshot vor jedem exec-Befehl ein. Variablen, die wie Geheimnisse aussehen, werden ausgeschlossen; Sandbox- und Node-Ausführungen verwenden diesen Snapshot nicht. Legen Sie `OPENCLAW_EXEC_SHELL_SNAPSHOT=0` in der Prozessumgebung des Gateways fest, um diesen Snapshot-Pfad zu deaktivieren.
- Die Host-Ausführung (`gateway`/`node`) lehnt `env.PATH` und Loader-Überschreibungen (`LD_*`/`DYLD_*`) ab, um das Kapern von Binärdateien oder eingeschleusten Code zu verhindern.
- OpenClaw legt `OPENCLAW_SHELL=exec` in der Umgebung des gestarteten Befehls fest (einschließlich PTY- und Sandbox-Ausführung), damit Shell-/Profilregeln den Kontext des exec-Werkzeugs erkennen können.
- Bei Ausführungen aus einem Kanal stellt OpenClaw außerdem eine eng begrenzte JSON-Nutzlast zur Absender-/Chat-Identität in `OPENCLAW_CHANNEL_CONTEXT` bereit, sofern der Kanal diese IDs geliefert hat.
- `exec` kann die Shell-Befehle `openclaw channels login` oder `/approve` nicht ausführen: `openclaw channels login` ist ein interaktiver Ablauf zur Kanalauthentifizierung, und `/approve` muss über den Genehmigungsbefehlshandler statt über eine Shell erfolgen. Führen Sie die Kanalanmeldung in einem Terminal auf dem Gateway-Host aus oder verwenden Sie ein kanalspezifisches Agentenwerkzeug zur Anmeldung, sofern eines vorhanden ist (beispielsweise `whatsapp_login`).
- Wichtig: Sandboxing ist **standardmäßig deaktiviert**. Wenn Sandboxing deaktiviert ist, wird implizites `host=auto` zu `gateway` aufgelöst. Explizites `host=sandbox` schlägt weiterhin sicher fehl, statt unbemerkt auf dem Gateway-Host ausgeführt zu werden. Aktivieren Sie Sandboxing oder verwenden Sie `host=gateway` mit Genehmigungen.
- Vorabprüfungen für Skripte (auf häufige Fehler in der Python-/Node-Shell-Syntax) untersuchen nur Dateien innerhalb der effektiven `workdir`-Grenze. Wenn ein Skriptpfad außerhalb von `workdir` aufgelöst wird, wird die Vorabprüfung für diese Datei übersprungen. Die Vorabprüfung wird außerdem vollständig übersprungen, wenn `host=gateway` und die effektive Richtlinie `security=full` mit `ask=off` ist.
- Starten Sie lang laufende Arbeiten, die jetzt beginnen, einmal und verlassen Sie sich auf die automatische Aktivierung nach Abschluss, wenn diese aktiviert ist und der Befehl eine Ausgabe erzeugt oder fehlschlägt. Verwenden Sie `process` für Protokolle, Status, Eingaben oder Eingriffe; bilden Sie keine Zeitplanung mit Schlafschleifen, Zeitlimitschleifen oder wiederholten Abfragen nach.
- Von Agenten gestartete Hintergrundbefehle werden bis zu ihrem Abschluss in den Ansichten für Hintergrundaufgaben im Web, unter iOS und unter Android angezeigt. Das Aufgabenregister wird abgeschlossen, bevor der Heartbeat nach Abschluss den Agenten erneut aktiviert.
- Verwenden Sie für Arbeiten, die später oder nach einem Zeitplan erfolgen sollen, Cron statt Schlaf-/Verzögerungsmustern mit `exec`.

## Konfiguration

| Schlüssel                              | Standardwert             | Hinweise                                                                                                                                                |
| -------------------------------------- | ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `tools.exec.timeoutSeconds`          | `1800`                   | Standardmäßiges Zeitlimit pro exec-Befehl in Sekunden. `timeout` pro Aufruf überschreibt es; `timeout: 0` pro Aufruf deaktiviert das Zeitlimit für den exec-Prozess. |
| `tools.exec.host`                    | `auto`                   | Wird bei aktiver Sandbox-Laufzeitumgebung zu `sandbox` aufgelöst, andernfalls zu `gateway`.                                                      |
| `tools.exec.mode`                    | vom Host abgeleitet      | Kanonische Richtlinieneinstellung. Siehe unten [Modi](#modes).                                                                                          |
| `tools.exec.reviewer.model`          | konfiguriertes primäres Agentenmodell | Optionale Provider-/Modellüberschreibung für die Überprüfung durch `mode=auto`.                                                           |
| `tools.exec.reviewer.timeoutMs`      | `30000`                  | Zeitlimit pro Phase für die Vorbereitung und Fertigstellung durch das Prüfermodell, bevor auf einen Menschen zurückgegriffen wird.                     |
| `tools.exec.node`                    | nicht festgelegt         |                                                                                                                                                         |
| `tools.exec.notifyOnExit`            | `true`                   | Wenn wahr, stellen in den Hintergrund verschobene exec-Sitzungen beim Beenden ein Systemereignis in die Warteschlange und fordern einen Heartbeat an. |
| `tools.exec.approvalRunningNoticeMs` | `10000`                  | Gibt einmalig einen Hinweis zur laufenden Ausführung aus, wenn eine genehmigungspflichtige exec-Ausführung länger als dieser Wert dauert (`0` deaktiviert dies). |
| `tools.exec.strictInlineEval`        | `false`                  | Siehe [Inline-Auswertung](#inline-eval-strictinlineeval).                                                                                               |
| `tools.exec.commandHighlighting`     | `false`                  | Wenn wahr, können Genehmigungsabfragen vom Parser abgeleitete Befehlsabschnitte im Befehlstext hervorheben. Global oder pro Agent festlegen; ändert die Genehmigungsrichtlinie nicht. |
| `tools.exec.pathPrepend`             | nicht festgelegt         | Liste der Verzeichnisse, die bei exec-Ausführungen `PATH` vorangestellt werden (nur Gateway und Sandbox).                                  |
| `tools.exec.safeBins`                | nicht festgelegt         | Sichere Binärdateien, die nur Standardeingaben verwenden und ohne ausdrückliche Einträge in der Zulassungsliste ausgeführt werden können. Siehe [Sichere Binärdateien](/de/tools/exec-approvals-advanced#safe-bins-stdin-only). |
| `tools.exec.safeBinTrustedDirs`      | `/bin`, `/usr/bin`       | Zusätzliche explizite Verzeichnisse, denen bei Pfadprüfungen mit `safeBins` vertraut wird. Einträgen in `PATH` wird nie automatisch vertraut. |
| `tools.exec.safeBinProfiles`         | nicht festgelegt         | Optionale benutzerdefinierte argv-Richtlinie pro sicherer Binärdatei (`minPositional`, `maxPositional`, `allowedValueFlags`, `deniedFlags`). |

Die Host-Ausführung ohne Genehmigung ist der Standard für Gateway und Node (`mode=full`) – dies ergibt sich aus den Standardwerten der Host-Richtlinie, nicht aus `host=auto`. Wenn Sie Genehmigungen oder ein Verhalten mit Zulassungsliste wünschen, legen Sie `tools.exec.mode` fest und verschärfen Sie die Host-Genehmigungsdatei; siehe [Exec-Genehmigungen](/de/tools/exec-approvals#yolo-mode-no-approval). Um unabhängig vom Sandbox-Status die Weiterleitung an das Gateway oder den Node zu erzwingen, legen Sie `tools.exec.host` fest oder verwenden Sie `/exec host=...`.

Beispiel:

```json5
{
  tools: {
    exec: {
      pathPrepend: ["~/bin", "/opt/oss/bin"],
    },
  },
}
```

### Modi

`tools.exec.mode` ist die kanonische persistierte Richtlinieneinstellung. Das Sicherheits- und Genehmigungsverhalten der Laufzeit wird daraus abgeleitet.

| Modus       | Sicherheit | Nachfrage | Verhalten                                                                                                                                |
| ----------- | ----------- | --------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `deny`      | `deny`      | `off`     | Exec wird verweigert.                                                                                                                     |
| `allowlist` | `allowlist` | `off`     | Nur Befehle auf der Zulassungsliste bzw. Safe-Bin-Befehle werden ausgeführt; für nichts anderes erfolgt eine Nachfrage.                   |
| `ask`       | `allowlist` | `on-miss` | Treffer auf der Zulassungsliste werden direkt ausgeführt; für alles andere wird ein Mensch gefragt.                                       |
| `auto`      | `allowlist` | `on-miss` | Treffer auf der Zulassungsliste bzw. Safe-Bin-Treffer werden direkt ausgeführt; alles andere wird vor der Nachfrage bei einem Menschen durch den nativen automatischen Reviewer von OpenClaw geleitet. |
| `full`      | `full`      | `off`     | Keine Genehmigungsschranke.                                                                                                               |

Das sitzungsspezifische `/exec ask=always` fragt unabhängig vom gespeicherten Modus weiterhin jedes Mal einen Menschen.

Die Genehmigung durch die automatische Überprüfung gilt nur einmal. Auf dem Gateway stellt OpenClaw dem Reviewer den aufgelösten Pfad der ausführbaren Datei bereit und bindet die Ausführung an denselben Pfad. Befehle, die sich nicht auf einen einzigen durchsetzbaren Ausführungsplan reduzieren lassen – etwa Heredocs, Shell-Erweiterungen oder nicht unterstützte Anführungszeichen in Wrappern – greifen auf eine menschliche Genehmigung zurück, selbst wenn das Modell sie andernfalls zulassen würde.

Befehlsgenehmigungen des Codex-App-Servers, über die nicht bereits eine explizite Laufzeit- oder native Richtlinie entscheidet, verwenden den Weg der menschlichen Genehmigung. OpenClaw führt für diese Anfragen nicht den konfigurierten Exec-Reviewer aus, da Codex keine durchsetzbare aufgelöste ausführbare Datei bereitstellt, mit der sich die Überprüfungsentscheidung an den von Codex ausgeführten Befehl binden ließe.

### Inline-Auswertung (`strictInlineEval`)

Wenn `tools.exec.strictInlineEval` auf `true` gesetzt ist, erfordern Inline-Auswertungsformen von Interpretern eine Prüfung oder explizite Genehmigung: `python -c`, `node -e`, `ruby -e`, `perl -e`, `php -r`, `lua -e`, `osascript -e` und ähnliche Formen in anderen unterstützten Interpretern und Befehlsträgern (`awk`, `find -exec`, `make`, `sed`, `xargs` und weitere). In `mode=auto` kann der normale Exec-Genehmigungsweg dem nativen automatischen Reviewer erlauben, einen eindeutig risikoarmen einmaligen Befehl zuzulassen; direkte `system.run`-Aufrufe auf dem Node-Host erfordern weiterhin eine explizite Genehmigung, da sie den Befehl nicht an einen Weg zur menschlichen Genehmigung übergeben können. Fordert der Reviewer eine Nachfrage an, wird die Anfrage an einen Menschen weitergeleitet. `allow-always` kann weiterhin unbedenkliche Interpreter-/Skriptaufrufe dauerhaft speichern, Inline-Auswertungsformen werden jedoch nicht zu dauerhaften Zulassungsregeln.

### PATH-Verarbeitung

- `host=gateway`: führt den `PATH` Ihrer Login-Shell mit der Exec-Umgebung zusammen. Überschreibungen von `env.PATH` werden bei der Host-Ausführung abgelehnt. Der Daemon selbst wird weiterhin mit einem minimalen `PATH` ausgeführt:
  - macOS: `/opt/homebrew/bin`, `/usr/local/bin`, `/usr/bin`, `/bin`
  - Linux: `/usr/local/bin`, `/usr/bin`, `/bin`
  - Um zu verhindern, dass die Shell-Konfiguration des Benutzers (wie `~/.zshenv` oder `/etc/zshenv`) beim Start priorisierte Pfade überschreibt, werden `tools.exec.pathPrepend`-Einträge direkt vor der Ausführung innerhalb des Shell-Befehls sicher dem endgültigen `PATH` vorangestellt.
- `host=sandbox`: führt `sh -lc` (Login-Shell) innerhalb des Containers aus, sodass `/etc/profile` den Wert von `PATH` zurücksetzen kann. OpenClaw stellt `env.PATH` nach dem Laden des Profils über eine interne Umgebungsvariable voran (ohne Shell-Interpolation); `tools.exec.pathPrepend` gilt auch hier.
- `host=node`: Nur die von Ihnen übergebenen, nicht blockierten Umgebungsüberschreibungen werden an den Node gesendet. Überschreibungen von `env.PATH` werden bei der Host-Ausführung abgelehnt und von Node-Hosts ignoriert. Wenn Sie zusätzliche PATH-Einträge auf einem Node benötigen, konfigurieren Sie die Dienstumgebung des Node-Hosts (systemd/launchd) oder installieren Sie Werkzeuge an Standardspeicherorten.

Sitzungsspezifische Node-Bindung (verwenden Sie die als Schlüssel verwendete Agent-ID in der Konfiguration):

```bash
openclaw config get agents.entries
openclaw config set 'agents.entries.main.tools.exec.node' "node-id-or-name"
```

Control UI: Die Seite **Geräte** enthält für dieselben Einstellungen einen kleinen Bereich „Exec-Node-Bindung“.

## Sitzungsüberschreibungen (`/exec`)

Verwenden Sie `/exec`, um **sitzungsspezifische** Standardwerte für `host`, `security`, `ask` und `node` festzulegen. Senden Sie `/exec` ohne Argumente, um die aktuellen Werte anzuzeigen.

Beispiel:

```text
/exec host=auto security=allowlist ask=on-miss node=mac-1
```

`/exec` wird nur für **autorisierte Absender** über Kanal-Zulassungslisten/Kopplung und Zugriffsgruppen berücksichtigt. Die Durchsetzung von Zugriffsgruppen ist immer aktiviert. Der Befehl aktualisiert **nur den Sitzungsstatus** und schreibt nicht in die Konfiguration. Autorisierte Absender externer Kanäle können diese Sitzungsstandardwerte festlegen. Interne Gateway-/Webchat-Clients benötigen `operator.admin`, um sie dauerhaft zu speichern.

Um Exec vollständig zu deaktivieren, verweigern Sie es über die Werkzeugrichtlinie (`tools.deny: ["exec"]` oder agentenspezifisch). Host-Genehmigungen gelten weiterhin, sofern Sie nicht ausdrücklich `security=full` und `ask=off` festlegen.

## Exec-Genehmigungen (Begleit-App/Node-Host)

Sandbox-Agenten können vor jeder Ausführung von `exec` auf dem Gateway oder Node-Host eine Genehmigung verlangen. Unter [Exec-Genehmigungen](/de/tools/exec-approvals) finden Sie Informationen zur Richtlinie, Zulassungsliste und zum UI-Ablauf.

Wenn eine menschliche Genehmigung erforderlich ist, geben Node-Host- und nicht native Gateway-Abläufe sofort `status: "approval-pending"` und eine Genehmigungs-ID zurück. Native Chat- und Web-UI-Gateway-Abläufe können stattdessen inline warten und nach der Genehmigung das endgültige Befehlsergebnis zurückgeben. Ein `approval-pending`-Ergebnis bedeutet, dass der Befehl noch nicht gestartet wurde. Warnungen zum Vordergrund-Fallback werden daher nur angezeigt, wenn der genehmigte Befehl tatsächlich inline ausgeführt wird. Genehmigte asynchrone Ausführungen senden Systemereignisse zum Befehlsfortschritt und Abschluss (`Exec running` / `Exec finished`); abgelehnte oder zeitlich abgelaufene Genehmigungen sind endgültig und reaktivieren die Agentensitzung nicht mit einem Systemereignis über die Ablehnung.

Auf Kanälen mit nativen Genehmigungskarten/-schaltflächen sollte sich der Agent zuerst auf diese native UI verlassen und einen manuellen `/approve`-Befehl nur dann angeben, wenn das Werkzeugergebnis ausdrücklich besagt, dass Chat-Genehmigungen nicht verfügbar sind oder die manuelle Genehmigung der einzige Weg ist.

## Zulassungsliste und Safe Bins

Die manuelle Durchsetzung der Zulassungsliste gleicht Globs aufgelöster Binärpfade und Globs reiner Befehlsnamen ab. Reine Namen stimmen nur mit über PATH aufgerufenen Befehlen überein, sodass `rg` mit `/opt/homebrew/bin/rg` übereinstimmen kann, wenn der Befehl `rg` lautet, nicht jedoch mit `./rg` oder `/tmp/rg`.

Wenn `security=allowlist`, werden Shell-Befehle nur dann automatisch zugelassen, wenn jedes Pipeline-Segment auf der Zulassungsliste steht oder ein Safe Bin ist. Verkettungen (`;`, `&&`, `||`) und Umleitungen werden im Zulassungslistenmodus abgelehnt, sofern nicht jedes Segment der obersten Ebene die Zulassungsliste erfüllt (einschließlich Safe Bins). Umleitungen werden weiterhin nicht unterstützt. Dauerhaftes `allow-always`-Vertrauen umgeht diese Regel nicht: Bei einem verketteten Befehl muss weiterhin jedes Segment der obersten Ebene übereinstimmen.

`autoAllowSkills` ist ein separater Komfortweg in Exec-Genehmigungen und nicht dasselbe wie manuelle Pfadeinträge der Zulassungsliste. Lassen Sie `autoAllowSkills` für streng explizites Vertrauen deaktiviert.

Verwenden Sie die beiden Steuerelemente für unterschiedliche Aufgaben:

- `tools.exec.safeBins`: kleine, ausschließlich stdin-basierte Streamfilter.
- `tools.exec.safeBinTrustedDirs`: explizite zusätzliche vertrauenswürdige Verzeichnisse für ausführbare Safe-Bin-Pfade.
- `tools.exec.safeBinProfiles`: explizite argv-Richtlinie für benutzerdefinierte Safe Bins.
- Zulassungsliste: explizites Vertrauen für Pfade ausführbarer Dateien.

Behandeln Sie `safeBins` nicht als allgemeine Zulassungsliste und fügen Sie keine Interpreter-/Laufzeitbinärdateien hinzu (zum Beispiel `python3`, `node`, `ruby`, `bash`). Wenn Sie diese benötigen, verwenden Sie explizite Einträge in der Zulassungsliste und lassen Sie Genehmigungsaufforderungen aktiviert.

`openclaw security audit` warnt, wenn für Interpreter-/Laufzeit-`safeBins`-Einträge explizite Profile fehlen, und `openclaw doctor --fix` kann fehlende benutzerdefinierte `safeBinProfiles`-Einträge vorbereiten. `openclaw security audit` und `openclaw doctor` warnen außerdem, wenn Sie Binärdateien mit weitreichendem Verhalten wie `jq` ausdrücklich wieder zu `safeBins` hinzufügen (`jq` kann Umgebungsdaten lesen und jq-Code aus Modulen oder Startdateien laden; bevorzugen Sie daher stattdessen explizite Einträge in der Zulassungsliste oder genehmigungspflichtige Ausführungen). `jq` wird als Safe Bin verweigert, selbst wenn es ausdrücklich aufgeführt ist. Wenn Sie Interpreter ausdrücklich in die Zulassungsliste aufnehmen, aktivieren Sie `tools.exec.strictInlineEval`, damit Formen zur Inline-Codeauswertung weiterhin eine Prüfung oder explizite Genehmigung erfordern.

Vollständige Richtliniendetails und Beispiele finden Sie unter [Exec-Genehmigungen](/de/tools/exec-approvals-advanced#safe-bins-stdin-only) und [Safe Bins im Vergleich zur Zulassungsliste](/de/tools/exec-approvals-advanced#safe-bins-versus-allowlist).

## Beispiele

Vordergrund:

```json
{ "tool": "exec", "command": "ls -la" }
```

Hintergrund und Abfrage:

```json
{"tool":"exec","command":"npm run build","yieldMs":1000}
{"tool":"process","action":"poll","sessionId":"<id>"}
```

Die Abfrage dient dem Statusabruf bei Bedarf, nicht Warteschleifen. Wenn das automatische Aufwecken bei Abschluss aktiviert ist, kann der Befehl die Sitzung aufwecken, sobald er eine Ausgabe erzeugt oder fehlschlägt.

Tasten senden (im tmux-Stil):

```json
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["Enter"]}
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["C-c"]}
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["Up","Up","Enter"]}
```

Absenden (nur CR senden):

```json
{ "tool": "process", "action": "submit", "sessionId": "<id>" }
```

Einfügen (standardmäßig mit Klammerung):

```json
{ "tool": "process", "action": "paste", "sessionId": "<id>", "text": "line1\nline2\n" }
```

## apply_patch

`apply_patch` ist ein Unterwerkzeug von `exec` für strukturierte Änderungen an mehreren Dateien. Es ist standardmäßig aktiviert und für jeden Modell-Provider verfügbar; `allowModels` kann es einschränken. Verwenden Sie die Konfiguration nur, wenn Sie es deaktivieren oder auf bestimmte Modelle beschränken möchten:

```json5
{
  tools: {
    exec: {
      applyPatch: { workspaceOnly: true, allowModels: ["gpt-5.6-sol"] },
    },
  },
}
```

Hinweise:

- Die Werkzeugrichtlinie gilt weiterhin; `allow: ["write"]` lässt `apply_patch` implizit zu.
- `deny: ["write"]` verweigert `apply_patch` nicht; verweigern Sie `apply_patch` ausdrücklich oder verwenden Sie `deny: ["group:fs"]`, wenn auch Patch-Schreibvorgänge blockiert werden sollen.
- Die Konfiguration befindet sich unter `tools.exec.applyPatch`.
- `tools.exec.applyPatch.enabled` verwendet standardmäßig `true`; setzen Sie es auf `false`, um das Werkzeug zu deaktivieren.
- `tools.exec.applyPatch.workspaceOnly` verwendet standardmäßig `true` (auf den Workspace beschränkt). Setzen Sie es nur dann auf `false`, wenn Sie ausdrücklich möchten, dass `apply_patch` außerhalb des Workspace-Verzeichnisses schreibt/löscht.
- `tools.exec.applyPatch.allowModels` ist eine optionale Zulassungsliste von Modell-IDs (unverarbeitet, wie `gpt-5.4`, oder vollständig, wie `openai/gpt-5.4`). Wenn sie festgelegt ist, erhalten nur übereinstimmende Modelle das Werkzeug; wenn sie nicht festgelegt ist, erhalten es alle Modelle.

## Verwandte Themen

- [Exec-Genehmigungen](/de/tools/exec-approvals) — Genehmigungsschranken für Shell-Befehle
- [Sandboxing](/de/gateway/sandboxing) — Ausführen von Befehlen in Sandbox-Umgebungen
- [Hintergrundprozess](/de/gateway/background-process) — Exec- und Prozesswerkzeug für lang laufende Vorgänge
- [Sicherheit](/de/gateway/security) — Werkzeugrichtlinie und erhöhte Zugriffsrechte
