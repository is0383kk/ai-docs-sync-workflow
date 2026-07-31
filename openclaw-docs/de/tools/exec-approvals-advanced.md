---
read_when:
    - Sichere Bins oder benutzerdefinierte Safe-Bin-Profile konfigurieren
    - Weiterleiten von Genehmigungen an Slack/Discord/Telegram oder andere Chatkanäle
    - Implementieren eines nativen Genehmigungsclients für einen Kanal
summary: 'Erweiterte Ausführungsgenehmigungen: sichere Binärdateien, Interpreter-Bindung, Weiterleitung von Genehmigungen, native Zustellung'
title: Ausführungsgenehmigungen — erweitert
x-i18n:
    generated_at: "2026-07-26T18:08:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ac90d41f867a8ae4f14b6c9c13f3732d102a65707f456623932b858145a9bf46
    source_path: tools/exec-approvals-advanced.md
    workflow: 16
---

Fortgeschrittene Themen zu Ausführungsgenehmigungen: der `safeBins`-Schnellpfad, die Bindung von Interpreter/Laufzeit
und die Weiterleitung von Genehmigungen an Chat-Kanäle (einschließlich nativer Zustellung).
Die grundlegende Richtlinie und den Genehmigungsablauf finden Sie unter [Ausführungsgenehmigungen](/de/tools/exec-approvals).

## Sichere Binaries (nur stdin)

`tools.exec.safeBins` bezeichnet Binaries, die **ausschließlich stdin** verwenden (zum Beispiel `cut`) und
im Zulassungslistenmodus **ohne** explizite Zulassungslisteneinträge ausgeführt werden. Sichere Binaries lehnen
positionsbezogene Dateiargumente und pfadähnliche Token ab, sodass sie nur den
eingehenden Datenstrom verarbeiten können. Betrachten Sie dies als eng begrenzten Schnellpfad für Datenstromfilter, nicht als
allgemeine Vertrauensliste.

<Warning>
Fügen Sie **keine** Interpreter- oder Laufzeit-Binaries (zum Beispiel `python3`, `node`,
`ruby`, `bash`, `sh`, `zsh`) zu `safeBins` hinzu. Wenn ein Befehl konstruktionsbedingt Code auswerten,
Unterbefehle ausführen oder Dateien lesen kann, verwenden Sie vorzugsweise explizite Zulassungslisteneinträge
und lassen Sie Genehmigungsaufforderungen aktiviert. Benutzerdefinierte sichere Binaries müssen ein explizites
Profil in `tools.exec.safeBinProfiles.<bin>` definieren.
</Warning>

Standardmäßige sichere Binaries:

[//]: # "SAFE_BIN_DEFAULTS:START"

`cut`, `uniq`, `head`, `tail`, `tr`, `wc`

[//]: # "SAFE_BIN_DEFAULTS:END"

`grep` und `sort` sind nicht in der Standardliste enthalten. Wenn Sie sie aktivieren, behalten Sie explizite
Zulassungslisteneinträge für ihre Arbeitsabläufe ohne stdin bei. Geben Sie für `grep` im Modus für sichere Binaries
das Muster mit `-e`/`--regexp` an; die positionsbezogene Musterform wird abgelehnt,
damit Dateioperanden nicht als mehrdeutige Positionsargumente eingeschleust werden können.

### Argv-Validierung und abgelehnte Flags

Die Validierung erfolgt deterministisch allein anhand der argv-Struktur (ohne Prüfungen,
ob Dateien im Host-Dateisystem vorhanden sind). Dadurch wird verhindert, dass Unterschiede zwischen Zulassen und Ablehnen
als Orakel für die Existenz von Dateien dienen. Dateibezogene Optionen werden für standardmäßige sichere Binaries abgelehnt; lange
Optionen werden nach dem Fail-Closed-Prinzip validiert (unbekannte Flags und mehrdeutige Abkürzungen werden
abgelehnt). Erkannte schreibgeschützte boolesche Flags der standardmäßigen Binaries (zum Beispiel
`wc -l`, `tr -d`, `uniq -c`) werden akzeptiert, während nicht erkannte kurze Flags
weiterhin nach dem Fail-Closed-Prinzip behandelt werden und eine manuelle Genehmigung erfordern.

Nach Profil des sicheren Binary abgelehnte Flags:

[//]: # "SAFE_BIN_DENIED_FLAGS:START"

- `grep`: `--dereference-recursive`, `--directories`, `--exclude-from`, `--file`, `--recursive`, `-R`, `-d`, `-f`, `-r`
- `jq`: `--argfile`, `--from-file`, `--library-path`, `--rawfile`, `--slurpfile`, `-L`, `-f`
- `sort`: `--compress-program`, `--files0-from`, `--output`, `--random-source`, `--temporary-directory`, `-T`, `-o`
- `tail`: `--follow`, `--retry`, `-F`, `-f`
- `wc`: `--files0-from`

[//]: # "SAFE_BIN_DENIED_FLAGS:END"

Sichere Binaries erzwingen außerdem, dass argv-Token bei der Ausführung für Segmente,
die ausschließlich stdin verwenden, als **wörtlicher Text** behandelt werden (kein Globbing und keine `$VARS`-Expansion), sodass
Muster wie `*` oder `$HOME/...` nicht zum Einschleusen von Dateilesevorgängen verwendet werden können. `awk`,
`sed` und `jq` werden als sichere Binaries immer abgelehnt, da ihre Semantik nicht
als ausschließlich stdin verwendend validiert werden kann: `jq` kann Umgebungsdaten lesen und jq-Code aus
Modulen oder Startdateien laden. Verwenden Sie für diese Werkzeuge anstelle von `safeBins` einen expliziten
Zulassungslisteneintrag oder eine Genehmigungsaufforderung.

### Vertrauenswürdige Binary-Verzeichnisse

Sichere Binaries müssen aus vertrauenswürdigen Binary-Verzeichnissen aufgelöst werden (Systemstandards sowie
optional `tools.exec.safeBinTrustedDirs`). Einträge in `PATH` gelten niemals automatisch als vertrauenswürdig.
Die standardmäßig vertrauenswürdigen Verzeichnisse sind absichtlich auf `/bin` und `/usr/bin` beschränkt. Wenn
sich die ausführbare Datei Ihres sicheren Binary in Paketmanager- oder Benutzerpfaden befindet (zum Beispiel
`/opt/homebrew/bin`, `/usr/local/bin`, `/opt/local/bin`, `/snap/bin`), fügen Sie diese
explizit zu `tools.exec.safeBinTrustedDirs` hinzu.

### Shell-Verkettung, Wrapper und Multiplexer

Shell-Verkettung (`&&`, `||`, `;`) ist zulässig, wenn jedes Segment der obersten Ebene
die Zulassungsliste erfüllt (einschließlich sicherer Binaries oder automatischer Skill-Zulassung). Umleitungen
werden im Zulassungslistenmodus weiterhin nicht unterstützt. Befehlssubstitution (`$()` / Backticks) wird
bei der Analyse der Zulassungsliste abgelehnt, auch innerhalb doppelter Anführungszeichen; verwenden Sie einfache
Anführungszeichen, wenn Sie `$()` als wörtlichen Text benötigen.

Bei Genehmigungen über die macOS-Begleit-App wird roher Shell-Text mit Shell-Steuerungs- oder
Expansionssyntax (`&&`, `||`, `;`, `|`, `` ` ``, `$`, `<`, `>`, `(`, `)`) als
nicht in der Zulassungsliste enthalten behandelt, sofern das Shell-Binary selbst nicht auf der Zulassungsliste steht.

Bei Shell-Wrappern (`bash|sh|zsh ... -c/-lc`) werden anforderungsspezifische Umgebungsüberschreibungen
auf eine kleine explizite Zulassungsliste beschränkt (`TERM`, `LANG`, `LC_*`, `COLORTERM`,
`NO_COLOR`, `FORCE_COLOR`).

Bei `allow-always`-Entscheidungen im Zulassungslistenmodus speichern transparente Dispatch-Wrapper
(zum Beispiel `env`, `flock`, `nice`, `nohup`, `stdbuf`, `timeout`) den Pfad
der inneren ausführbaren Datei anstelle des Wrapper-Pfads. Shell-Multiplexer
(`busybox`, `toybox`) werden für Shell-Applets (`sh`, `ash` usw.) auf
dieselbe Weise entpackt. Wenn ein Wrapper oder Multiplexer nicht sicher entpackt werden kann, wird
kein Zulassungslisteneintrag automatisch gespeichert.

Wenn Sie Interpreter wie `python3` oder `node` in die Zulassungsliste aufnehmen, verwenden Sie vorzugsweise
`tools.exec.strictInlineEval=true`, damit die Inline-Auswertung weiterhin eine explizite
Genehmigung erfordert. Im strikten Modus kann `allow-always` weiterhin unbedenkliche
Interpreter-/Skriptaufrufe speichern, Träger von Inline-Auswertungen werden jedoch nicht
automatisch gespeichert.

### Sichere Binaries im Vergleich zur Zulassungsliste

| Thema            | `tools.exec.safeBins`                                  | Zulassungsliste (`exec-approvals.json`)                                                  |
| ---------------- | ------------------------------------------------------ | ---------------------------------------------------------------------------------- |
| Ziel             | Eng begrenzte stdin-Filter automatisch zulassen        | Bestimmten ausführbaren Dateien explizit vertrauen                                    |
| Abgleichstyp     | Name der ausführbaren Datei + argv-Richtlinie des sicheren Binary | Glob für den aufgelösten Pfad der ausführbaren Datei oder Glob für den reinen Befehlsnamen bei über PATH aufgerufenen Befehlen |
| Argumentumfang   | Durch das Profil des sicheren Binary und Regeln für wörtliche Token eingeschränkt | Standardmäßig Pfadabgleich; optional kann `argPattern` das analysierte argv einschränken              |
| Typische Beispiele | `head`, `tail`, `tr`, `wc`                             | `jq`, `python3`, `node`, `ffmpeg`, benutzerdefinierte CLIs                                     |
| Beste Verwendung | Texttransformationen mit geringem Risiko in Pipelines | Jedes Werkzeug mit breiterem Verhalten oder Nebenwirkungen                                     |

Konfigurationsort:

- `safeBins` stammt aus der Konfiguration (`tools.exec.safeBins` oder agentspezifisch `agents.entries.*.tools.exec.safeBins`).
- `safeBinTrustedDirs` stammt aus der Konfiguration (`tools.exec.safeBinTrustedDirs` oder agentspezifisch `agents.entries.*.tools.exec.safeBinTrustedDirs`).
- `safeBinProfiles` stammt aus der Konfiguration (`tools.exec.safeBinProfiles` oder agentspezifisch `agents.entries.*.tools.exec.safeBinProfiles`). Agentspezifische Profilschlüssel überschreiben globale Schlüssel.
- Zulassungslisteneinträge befinden sich in der hostlokalen Genehmigungsdatei unter `agents.<id>.allowlist` (oder über die Control UI / `openclaw approvals allowlist ...`).
- `openclaw security audit` warnt mit `tools.exec.safe_bins_interpreter_unprofiled`, wenn Interpreter-/Laufzeit-Binaries ohne explizite Profile in `safeBins` vorkommen.
- `openclaw doctor --fix` kann fehlende benutzerdefinierte `safeBinProfiles.<bin>`-Einträge als `{}` anlegen (anschließend prüfen und einschränken). Interpreter-/Laufzeit-Binaries werden nicht automatisch angelegt.

Beispiel für ein benutzerdefiniertes Profil:

```json5
{
  tools: {
    exec: {
      safeBins: ["myfilter"],
      safeBinProfiles: {
        myfilter: {
          minPositional: 0,
          maxPositional: 0,
          allowedValueFlags: ["-n", "--limit"],
          deniedFlags: ["-f", "--file", "-c", "--command"],
        },
      },
    },
  },
}
```

## Interpreter-/Laufzeitbefehle

Genehmigungspflichtige Interpreter-/Laufzeitausführungen sind bewusst restriktiv:

- Der exakte argv-/cwd-/env-Kontext wird immer gebunden.
- Direkte Shell-Skript- und direkte Laufzeitdateiformen werden nach bestem Bemühen an einen konkreten lokalen
  Dateisnapshot gebunden.
- Gängige Paketmanager-Wrapper-Formen, die weiterhin zu genau einer direkten lokalen Datei aufgelöst werden (zum Beispiel
  `pnpm exec`, `pnpm node`, `npm exec`, `npx`), werden vor der Bindung entpackt.
- Wenn OpenClaw für einen Interpreter-/Laufzeitbefehl nicht genau eine konkrete lokale Datei identifizieren kann
  (zum Beispiel Paketskripte, Auswertungsformen, laufzeitspezifische Loader-Ketten oder mehrdeutige Mehrdateiformen),
  wird die genehmigungspflichtige Ausführung abgelehnt, anstatt eine semantische Abdeckung zu behaupten, die
  nicht gegeben ist.
- Verwenden Sie für solche Arbeitsabläufe vorzugsweise Sandboxing, eine separate Host-Grenze oder einen explizit vertrauenswürdigen
  Zulassungslisten-/vollständigen Arbeitsablauf, bei dem der Betreiber die umfassendere Laufzeitsemantik akzeptiert.

Wenn Genehmigungen erforderlich sind, gibt das Ausführungswerkzeug sofort eine Genehmigungs-ID zurück. Verwenden Sie diese ID,
um spätere Systemereignisse der genehmigten Ausführung zuzuordnen (`Exec finished` und, sofern konfiguriert, `Exec running`).
Wenn vor Ablauf des Zeitlimits keine Entscheidung eintrifft, wird die Anfrage als Zeitüberschreitung der Genehmigung behandelt und
als endgültige Ablehnung des Host-Befehls ausgegeben. Bei asynchronen Genehmigungen des Haupt-Agenten mit einer ursprünglichen
Sitzung setzt OpenClaw diese Sitzung außerdem mit einer internen Folgenachricht fort, damit der Agent erkennt, dass
der Befehl nicht ausgeführt wurde, anstatt später ein fehlendes Ergebnis zu reparieren. Ausstehende Ausführungsgenehmigungen verfallen
standardmäßig nach 30 Minuten.

### Verhalten bei der Zustellung von Folgenachrichten

Nachdem eine genehmigte asynchrone Ausführung abgeschlossen ist, sendet OpenClaw einen nachfolgenden `agent`-Turn an dieselbe Sitzung.
Abgelehnte asynchrone Genehmigungen verwenden für den Ablehnungsstatus denselben Folgenachrichtenpfad der Hauptsitzung, registrieren jedoch
keine erhöht privilegierten Laufzeitübergaben und führen den Befehl nicht aus. Ablehnungen ohne fortsetzbare
Hauptsitzung werden entweder unterdrückt oder über einen sicheren direkten Pfad gemeldet, sofern ein solcher vorhanden ist.

- Wenn ein gültiges externes Zustellungsziel vorhanden ist (zustellbarer Kanal plus Ziel `to`), verwendet die Zustellung der Folgenachricht diesen Kanal.
- Bei reinen Webchat- oder internen Sitzungsabläufen ohne externes Ziel bleibt die Zustellung der Folgenachricht auf die Sitzung beschränkt (`deliver: false`).
- Wenn ein Aufrufer ausdrücklich eine strikte externe Zustellung anfordert, ohne dass ein externer Kanal aufgelöst werden kann, schlägt die Anfrage mit `INVALID_REQUEST` fehl.
- Wenn `bestEffortDeliver` aktiviert ist und kein externer Kanal aufgelöst werden kann, wird die Zustellung auf eine rein sitzungsbezogene Zustellung herabgestuft, anstatt fehlzuschlagen.

## Minimale Berechtigungsumfänge für Drittanbieter-Clients

Die Auflösung von Gateway-Genehmigungen wird durch den dedizierten Berechtigungsumfang `operator.approvals` geschützt. Dies gilt sowohl für die eigentümerspezifische Methode `exec.approval.resolve` als auch für die artunabhängige Methode `approval.resolve`; `operator.write` schließt diesen Umfang nicht ein. Dashboards und Integrationen sollten nur die Berechtigungsumfänge anfordern, die für die von ihnen verwendeten Methoden erforderlich sind. Behandeln Sie den Zugriff auf die Genehmigungsauflösung als eine der Remote-Ausführung entsprechende Berechtigung und gewähren Sie `operator.approvals` bewusst, selbst wenn der Client nur eine kleine Genehmigungsoberfläche darstellt.

## Weiterleitung von Genehmigungen an Chat-Kanäle

Sie können Ausführungsfreigabeaufforderungen an jeden Chatkanal (einschließlich Plugin-Kanälen) weiterleiten und
sie mit `/approve` genehmigen. Hierfür wird die normale Pipeline für ausgehende Zustellungen verwendet.

Konfiguration:

```json5
{
  approvals: {
    exec: {
      enabled: true,
      mode: "session", // "session" | "targets" | "both"
      agentFilter: ["main"],
      sessionFilter: ["discord"], // substring or regex
      targets: [
        { channel: "slack", to: "U12345678" },
        { channel: "telegram", to: "123456789" },
      ],
    },
  },
}
```

Im Chat antworten:

```
/approve <id> allow-once
/approve <id> allow-always
/approve <id> deny
```

Der Befehl `/approve` verarbeitet sowohl Ausführungsfreigaben als auch Plugin-Freigaben. Wenn die ID keiner ausstehenden Ausführungsfreigabe entspricht, werden stattdessen automatisch die Plugin-Freigaben geprüft. Dieser Fallback ist auf Fehler vom Typ „Freigabe nicht gefunden“ beschränkt; bei einer tatsächlichen Ablehnung oder einem Fehler der Ausführungsfreigabe erfolgt nicht unbemerkt ein erneuter Versuch als Plugin-Freigabe.

### Weiterleitung von Plugin-Freigaben

Die Weiterleitung von Plugin-Freigaben verwendet dieselbe Zustellungspipeline wie Ausführungsfreigaben, besitzt jedoch eine eigene,
unabhängige Konfiguration unter `approvals.plugin`. Das Aktivieren oder Deaktivieren der einen hat keine Auswirkungen auf die andere.
Informationen zum Verhalten bei der Plugin-Entwicklung, zu Anfragefeldern und zur Entscheidungssemantik finden Sie unter
[Plugin-Berechtigungsanfragen](/plugins/plugin-permission-requests).

```json5
{
  approvals: {
    plugin: {
      enabled: true,
      mode: "targets",
      agentFilter: ["main"],
      targets: [
        { channel: "slack", to: "U12345678" },
        { channel: "telegram", to: "123456789" },
      ],
    },
  },
}
```

Die Konfigurationsstruktur ist identisch mit `approvals.exec`: `enabled`, `mode`, `agentFilter`,
`sessionFilter` und `targets` funktionieren auf dieselbe Weise.

Kanäle, die gemeinsame interaktive Antworten unterstützen, zeigen dieselben Freigabeschaltflächen für Ausführungs- und
Plugin-Freigaben an. Kanäle ohne gemeinsame interaktive Benutzeroberfläche greifen auf Klartext mit Anweisungen zu `/approve`
zurück. Plugin-Freigabeanfragen können die verfügbaren Entscheidungen einschränken: Freigabeoberflächen verwenden
die in der Anfrage deklarierte Entscheidungsmenge, und der Gateway weist Versuche zurück, eine nicht
angebotene Entscheidung zu übermitteln.

### Freigaben im selben Chat auf jedem Kanal

Wenn eine Ausführungs- oder Plugin-Freigabeanfrage von einer zustellfähigen Chatoberfläche stammt, kann dieser Chat
sie standardmäßig mit `/approve` genehmigen. Dies gilt für Slack, Matrix, Microsoft Teams und
ähnliche zustellfähige Chats zusätzlich zu den bestehenden Abläufen in der Web- und Terminal-Benutzeroberfläche, wobei das
normale Kanalauthentifizierungsmodell für diese Konversation verwendet wird. Wenn der ursprüngliche Chat bereits Befehle senden
und Antworten empfangen kann, benötigen Freigabeanfragen nicht länger einen separaten nativen Zustelladapter, nur um
ausstehend zu bleiben.

Discord, Telegram und QQ bot unterstützen ebenfalls `/approve` im selben Chat, diese Kanäle verwenden jedoch weiterhin ihre
ermittelte Genehmigerliste zur Autorisierung, selbst wenn die native Freigabezustellung deaktiviert ist.

### Native Freigabezustellung

Einige Kanäle können auch als native Freigabeclients fungieren: Discord, Slack, Telegram, Matrix und QQ bot.
Native Clients ergänzen den gemeinsamen Ablauf für `/approve` im selben Chat um Direktnachrichten an Genehmiger,
die Verteilung im ursprünglichen Chat und eine kanalspezifische interaktive Freigabeoberfläche.

Wenn native Freigabekarten oder -schaltflächen verfügbar sind, ist diese native Benutzeroberfläche der primäre Pfad für den Agenten.
Der Agent sollte nicht zusätzlich einen doppelten Klartextbefehl `/approve` im Chat ausgeben, es sei denn, das Werkzeugergebnis besagt,
dass Chatfreigaben nicht verfügbar sind oder eine manuelle Freigabe der einzige verbleibende Pfad ist.

Wenn ein nativer Freigabeclient konfiguriert ist, aber für den ursprünglichen
Kanal keine native Laufzeit aktiv ist, hält OpenClaw die lokale deterministische Aufforderung `/approve` sichtbar. Wenn die native Laufzeit
aktiv ist und eine Zustellung versucht, aber kein Ziel die Karte erhält, sendet OpenClaw im selben Chat einen Fallback-
Hinweis mit dem exakten Befehl `/approve <id> <decision>`, damit die Anfrage weiterhin bearbeitet werden kann.

Allgemeines Modell:

- Die Ausführungsrichtlinie des Hosts entscheidet weiterhin, ob eine Ausführungsfreigabe erforderlich ist
- `approvals.exec` steuert die Weiterleitung von Freigabeaufforderungen an andere Chatziele
- `channels.<channel>.execApprovals` steuert, ob Discord, Slack, Telegram, QQ bot und ähnliche
  kanalspezifische native Clients aktiviert sind
- Slack-Plugin-Freigaben können den nativen Freigabeclient von Slack verwenden, wenn die Anfrage von Slack stammt
  und Slack-Plugin-Genehmiger ermittelt werden; `approvals.plugin` kann Plugin-Freigaben auch an Slack-
  Sitzungen oder -Ziele weiterleiten, selbst wenn Slack-Ausführungsfreigaben deaktiviert sind
- Native Google-Chat-Freigabekarten verarbeiten Ausführungs- und Plugin-Freigaben, die aus Google-
  Chat-Bereichen oder -Threads stammen, wenn stabile Genehmiger für `users/<id>` aus `dm.allowFrom` oder
  `defaultTo` ermittelt werden; für Entscheidungen verwenden sie keine Reaktionsereignisse
- Die Zustellung von Reaktionsfreigaben für WhatsApp und Signal wird durch `approvals.exec` und
  `approvals.plugin` gesteuert; sie besitzen keine `channels.<channel>.execApprovals`-Blöcke

Native Freigabeclients aktivieren automatisch die Zustellung vorrangig per Direktnachricht, wenn alle folgenden Bedingungen erfüllt sind:

- Der Kanal unterstützt die native Freigabezustellung
- Genehmiger können aus expliziten `execApprovals.approvers` oder einer Eigentümer-
  identität wie `commands.ownerAllowFrom` ermittelt werden
- `channels.<channel>.execApprovals.enabled` ist nicht gesetzt oder `"auto"`

Setzen Sie `enabled: false`, um einen nativen Freigabeclient ausdrücklich zu deaktivieren. Setzen Sie `enabled: true`, um
ihn bei ermittelten Genehmigern zu aktivieren. Die öffentliche Zustellung im ursprünglichen Chat bleibt über
`channels.<channel>.execApprovals.target` explizit. Wenn natives `target` die Zustellung im ursprünglichen Chat aktiviert,
enthalten Freigabeaufforderungen den Befehlstext.

FAQ: [Warum gibt es zwei Konfigurationen für Ausführungsfreigaben bei Chatfreigaben?](/help/faq-first-run)

- Discord: `channels.discord.execApprovals.*`
- Slack: `channels.slack.execApprovals.*`
- Telegram: `channels.telegram.execApprovals.*`
- QQ bot: `channels.qqbot.execApprovals.*`
- Google Chat: Konfigurieren Sie stabile Genehmiger mit `channels.googlechat.dm.allowFrom` oder
  `channels.googlechat.defaultTo`; kein `execApprovals`-Block ist erforderlich
- WhatsApp: Verwenden Sie `approvals.exec` und `approvals.plugin`, um Freigabeaufforderungen an WhatsApp weiterzuleiten
- Signal: Verwenden Sie `approvals.exec` und `approvals.plugin`, um Freigabeaufforderungen an Signal weiterzuleiten

Spezifisches Routing nativer Clients:

- Telegram verwendet standardmäßig Direktnachrichten an Genehmiger (`target: "dm"`). Wechseln Sie zu `channel` oder `both`, um
  Freigabeaufforderungen zusätzlich im ursprünglichen Telegram-Chat beziehungsweise -Thema anzuzeigen. Bei Telegram-Forenthemen behält OpenClaw
  das Thema für die Freigabeaufforderung und die anschließende Rückmeldung nach der Freigabe bei.
- Genehmiger für Discord und Telegram können explizit angegeben (`execApprovals.approvers`) oder aus
  `commands.ownerAllowFrom` abgeleitet werden; nur ermittelte Genehmiger können genehmigen oder ablehnen.
- Slack-Genehmiger können explizit angegeben (`execApprovals.approvers`) oder aus
  `commands.ownerAllowFrom` abgeleitet werden. Direktnachrichten für Slack-Plugin-Freigaben verwenden Slack-Plugin-Genehmiger aus `allowFrom`
  und das standardmäßige Konto-Routing, nicht die Genehmiger für Slack-Ausführungsfreigaben. Native Slack-Schaltflächen bewahren die Art der Freigabe-ID,
  sodass `plugin:`-IDs Plugin-Freigaben ohne eine zweite lokale Slack-Fallback-Ebene auflösen können.
- Native Google-Chat-Karten bewahren den manuellen Fallback `/approve` im Nachrichtentext, aber Rückrufe von Kartenschaltflächen
  übertragen nur undurchsichtige Aktionstoken; Freigabe-ID und Entscheidung werden aus dem
  serverseitigen ausstehenden Status wiederhergestellt.
- WhatsApp-Emoji-Freigaben verarbeiten sowohl Ausführungs- als auch Plugin-Aufforderungen, wenn die entsprechende übergeordnete
  Weiterleitungsfamilie an WhatsApp weiterleitet. Aufforderungen nativen Ursprungs werden direkt gebunden; die gemeinsame Zustellung im Zielmodus
  bindet dieselben typisierten Freigabemetadaten an die akzeptierte WhatsApp-Nachrichtenquittung.
- Signal-Reaktionsfreigaben verarbeiten Ausführungs- und Plugin-Aufforderungen nur, wenn die entsprechende übergeordnete
  Weiterleitungsfamilie aktiviert ist und an Signal weiterleitet. Direkte Signal-Ausführungsfreigaben im selben Chat können
  den lokalen Fallback `/approve` ohne explizite Genehmiger unterdrücken; die Auflösung von Signal-Reaktionen
  erfordert weiterhin explizite Signal-Genehmiger aus `channels.signal.allowFrom` oder `defaultTo`.
- Das native Routing von Matrix per Direktnachricht oder Kanal sowie Reaktionsabkürzungen verarbeiten sowohl Ausführungs- als auch Plugin-Freigaben;
  die Plugin-Autorisierung stammt weiterhin aus `channels.matrix.dm.allowFrom`. Native Matrix-Aufforderungen
  enthalten beim ersten Aufforderungsereignis benutzerdefinierte Ereignisinhalte vom Typ `com.openclaw.approval`, sodass OpenClaw-kompatible
  Matrix-Clients den strukturierten Freigabestatus lesen können, während Standardclients den Klartext-
  Fallback `/approve` beibehalten.
- Native Freigabeschaltflächen von Discord und Telegram enthalten in transportprivaten Rückrufdaten eine explizite Eigentümerart für Ausführung oder Plugin
  und lösen nur diesen Eigentümer auf. Ältere `/approve`-Steuerelemente ohne
  Art bleiben ein begrenzter Kompatibilitätspfad: Sie versuchen nur Eigentümerarten, die der Akteur genehmigen darf,
  fahren nur nach dem Ergebnis „Freigabe nicht gefunden“ fort und leiten die Eigentümerschaft niemals aus der Freigabe-ID ab.
- Die anfragende Person muss kein Genehmiger sein.
- Wenn keine Bedieneroberfläche und kein konfigurierter Freigabeclient die Anfrage annehmen kann, greift die Aufforderung auf
  `askFallback` zurück.

Sensible, nur Eigentümern vorbehaltene Gruppenbefehle wie `/diagnostics` und `/export-trajectory` verwenden privates
Eigentümer-Routing für Freigabeaufforderungen und Endergebnisse. OpenClaw versucht zunächst eine private Route auf derselben
Oberfläche, auf der der Eigentümer den Befehl ausgeführt hat. Wenn diese Oberfläche keine private Eigentümerroute besitzt, wird
auf die erste verfügbare Eigentümerroute aus `commands.ownerAllowFrom` zurückgegriffen, sodass ein Discord-Gruppenbefehl
die Freigabe und das Ergebnis weiterhin an die Telegram-Direktnachrichten des Eigentümers senden kann, wenn Telegram als
primäre private Oberfläche konfiguriert ist. Der Gruppenchat erhält lediglich eine kurze Bestätigung.

Siehe:

- [Discord](/channels/discord)
- [Telegram](/channels/telegram)
- [QQ bot](/channels/qqbot)

### Offizielle mobile Bediener-Apps

Die offiziellen iOS- und Android-Apps können auch ausstehende, dem Gateway zugeordnete
Ausführungsfreigaben prüfen, wenn eine `operator.admin`-Verbindung verwendet wird oder wenn ihr gekoppeltes
`operator.approvals`-Gerät von der Anfrage ausdrücklich als Ziel angegeben wurde. Sie lesen
denselben bereinigten dauerhaften Datensatz, den die
Control UI verwendet, übermitteln eine artbezogene Entscheidung und zeigen das kanonische
Erstantwortergebnis des Gateways an. Die Apple Watch spiegelt diese Freigabeaufforderungen über
das gekoppelte iPhone mit Aktionen zum einmaligen Zulassen und Ablehnen. Im direkten Watch-Gateway-Modus
werden keine Freigaben geprüft.

Eine verlorene Auflösungsbestätigung macht die übermittelte Auswahl nicht maßgeblich:
Die App deaktiviert die Steuerelemente und liest den Datensatz erneut. Wenn eine andere Oberfläche
zuvorgekommen ist, zeigt die App diese aufgezeichnete Entscheidung an. Ausstehende Aufforderungen bleiben an den
Gateway gebunden, der sie ausgegeben hat, sodass ein Wechsel des aktiven Gateways eine
alte Freigabe-ID nicht umleiten kann.

### macOS-IPC-Ablauf

```
Gateway -> Node-Dienst (WS)
                 |  IPC (UDS + Token + HMAC + TTL)
                 v
             Mac-App (Benutzeroberfläche + Freigaben + system.run)
```

Sicherheitshinweise:

- Unix-Socket-Modus `0600`, Token gespeichert in `exec-approvals.json`.
- Peer-Prüfung auf dieselbe UID.
- Challenge-Response-Verfahren (Nonce + HMAC-Token + Anfragehash) + kurze TTL.

## FAQ

### Wann würden `accountId` und `threadId` für ein Freigabeziel verwendet?

Verwenden Sie `accountId`, wenn für den Kanal mehrere Identitäten konfiguriert sind und die Freigabeaufforderung
über ein bestimmtes Konto gesendet werden muss. Verwenden Sie `threadId`, wenn das Ziel Themen oder
Threads unterstützt und die Aufforderung innerhalb dieses Threads statt im übergeordneten Chat verbleiben soll.

Ein konkretes Telegram-Beispiel ist eine Betriebs-Supergruppe mit Forenthemen und zwei Telegram-Bot-
Konten. Der Wert `to` bezeichnet die Supergruppe, `accountId` wählt das Bot-Konto aus und `threadId`
wählt das Forenthema aus:

```json5
{
  approvals: {
    exec: {
      enabled: true,
      mode: "targets",
      targets: [
        {
          channel: "telegram",
          to: "-1001234567890",
          accountId: "ops-bot",
          threadId: "77",
        },
      ],
    },
  },
  channels: {
    telegram: {
      accounts: {
        default: {
          name: "Primary bot",
          botToken: "env:TELEGRAM_PRIMARY_BOT_TOKEN",
        },
        "ops-bot": {
          name: "Operations bot",
          botToken: "env:TELEGRAM_OPS_BOT_TOKEN",
        },
      },
    },
  },
}
```

Mit dieser Einrichtung werden weitergeleitete Ausführungsgenehmigungen vom Telegram-Konto `ops-bot` im Thema
`77` des Chats `-1001234567890` veröffentlicht. Ein Ziel ohne `accountId` verwendet das Standardkonto des Kanals, und
ein Ziel ohne `threadId` veröffentlicht im übergeordneten Ziel.

### Wenn Genehmigungen an eine Sitzung gesendet werden, kann dann jeder in dieser Sitzung sie erteilen?

Nein. Die Zustellung an eine Sitzung steuert nur, wo die Aufforderung angezeigt wird. Sie berechtigt nicht automatisch alle
Teilnehmenden dieses Chats, die Genehmigung zu erteilen.

Für generische `/approve` im selben Chat muss der Absender bereits für Befehle in dieser
Kanalsitzung autorisiert sein. Wenn der Kanal explizite Genehmigungsberechtigte bereitstellt, können diese Berechtigten
die Aktion `/approve` autorisieren, selbst wenn sie ansonsten in dieser Sitzung nicht zur Ausführung von Befehlen berechtigt sind.

Einige Kanäle sind strenger. Native Genehmigungs-Direktnachrichten von Discord, Telegram, Matrix und Slack sowie ähnliche
native Genehmigungsclients verwenden ihre ermittelten Listen der Genehmigungsberechtigten für die Genehmigungsautorisierung. Beispielsweise
kann eine Genehmigungsaufforderung in einem Telegram-Forumsthema für alle Personen im Thema sichtbar sein, aber nur numerische
Telegram-Benutzer-IDs, die aus `channels.telegram.execApprovals.approvers` oder
`commands.ownerAllowFrom` ermittelt wurden, können sie genehmigen oder ablehnen.

## Verwandte Themen

- [Ausführungsgenehmigungen](/de/tools/exec-approvals) — zentrale Richtlinie und Genehmigungsablauf
- [Ausführungswerkzeug](/de/tools/exec)
- [Erweiterter Modus](/de/tools/elevated)
- [Skills](/de/tools/skills) — Skill-gestütztes Verhalten für automatische Zulassung
