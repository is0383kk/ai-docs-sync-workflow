---
read_when:
    - Heartbeat-Frequenz oder Nachrichtenübermittlung anpassen
    - Entscheidung zwischen Heartbeat und Cron für geplante Aufgaben
sidebarTitle: Heartbeat
summary: Heartbeat-Abrufnachrichten und Benachrichtigungsregeln
title: Heartbeat
x-i18n:
    generated_at: "2026-07-26T18:27:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 44c78e797987d8dccab910cd82fc1f482df86afce40677846d8f26522d33f6fa
    source_path: gateway/heartbeat.md
    workflow: 16
---

<Note>
**Heartbeat oder cron?** Unter [Automatisierung](/de/automation) finden Sie Hinweise dazu, wann welche Option verwendet werden sollte.
</Note>

Heartbeat führt in der Hauptsitzung **regelmäßige Agent-Durchläufe** aus, damit das Modell auf alles aufmerksam machen kann, was beachtet werden muss, ohne Sie mit Nachrichten zu überhäufen.

Heartbeat ist ein geplanter Durchlauf der Hauptsitzung – dabei werden **keine** Datensätze für [Hintergrundaufgaben](/de/automation/tasks) erstellt. Aufgabendatensätze sind für entkoppelte Arbeiten vorgesehen (ACP-Ausführungen, Subagenten, isolierte cron-Aufträge).

Intern wird der Heartbeat-Takt vom cron-Scheduler verwaltet: Das Gateway verwaltet für jeden Agenten mit aktiviertem Heartbeat einen systemeigenen cron-Auftrag (in `openclaw cron list --all` als `Heartbeat (agent-id)` sichtbar). Die Heartbeat-Konfiguration bleibt die Eingabe für den gewünschten Zustand, während der persistierte Überwachungszeitplan den tatsächlichen Takt und die anschließende Abkühlphase des Runners bestimmt. Das Gateway übernimmt Konfigurationsänderungen beim Start und beim erneuten Laden der Konfiguration; `openclaw doctor --fix` kann fehlende oder veraltete Überwachungszeilen vor dem nächsten Gateway-Start anlegen. Bearbeiten Sie `agents.*.heartbeat`, nicht den cron-Auftrag.

Geplante Heartbeats erfordern cron. Wenn `cron.enabled` den Wert `false` oder `OPENCLAW_SKIP_CRON=1` hat, protokolliert das Gateway beim Start eine Warnung und führt keine geplanten Heartbeats aus; manuelle und ereignisgesteuerte Heartbeat-Aktivierungen bleiben verfügbar. Es gibt keinen separaten Heartbeat-Ersatz-Timer.

Fehlerbehebung: [Geplante Aufgaben](/de/automation/cron-jobs#troubleshooting)

## Schnellstart (Einsteiger)

<Steps>
  <Step title="Takt auswählen">
    Lassen Sie Heartbeats aktiviert (Standard ist `30m` oder `1h`, wenn die Anthropic-Authentifizierung per OAuth/Token konfiguriert ist, einschließlich der Wiederverwendung der Claude CLI), oder legen Sie einen eigenen Takt fest.
  </Step>
  <Step title="Überwachungsnotizen hinzufügen (optional)">
    Speichern Sie mit `openclaw cron scratch <jobId> --set "..."` eine kurze Checkliste in den Notizen der Heartbeat-Überwachung.
  </Step>
  <Step title="Ziel für Heartbeat-Nachrichten festlegen">
    `target: "none"` ist der Standard; legen Sie `target: "last"` fest, um Nachrichten an den letzten Kontakt weiterzuleiten.
  </Step>
  <Step title="Optionale Feinabstimmung">
    - Verwenden Sie einen schlanken Bootstrap-Kontext, wenn Heartbeat-Ausführungen nur die Überwachungsnotizen benötigen.
    - Aktivieren Sie isolierte Sitzungen, damit nicht bei jedem Heartbeat der vollständige Gesprächsverlauf gesendet wird.
    - Beschränken Sie Heartbeats auf aktive Zeiten (Ortszeit).

  </Step>
</Steps>

Beispielkonfiguration:

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // explizite Zustellung an den letzten Kontakt (Standard ist "none")
        directPolicy: "allow", // Standard: direkte/DM-Ziele zulassen; auf "block" setzen, um sie zu unterdrücken
        lightContext: true, // optional: Workspace-Bootstrap-Dateien für Heartbeat-Ausführungen überspringen
        isolatedSession: true, // optional: neue Sitzung bei jeder Ausführung (kein Gesprächsverlauf)
        // activeHours: { start: "08:00", end: "24:00" },
      },
    },
  },
}
```

## Standardwerte

- Intervall: `30m`. Durch Anwenden der Anthropic-Provider-Standardwerte wird dies auf `1h` erhöht, wenn der ermittelte Authentifizierungsmodus OAuth/Token ist (einschließlich der Wiederverwendung der Claude CLI), jedoch nur, solange `heartbeat.every` nicht festgelegt ist. Legen Sie `agents.defaults.heartbeat.every` oder agentenspezifisch `agents.entries.*.heartbeat.every` fest; verwenden Sie zum Deaktivieren `0m`.
- Prompt-Inhalt (über `agents.defaults.heartbeat.prompt` konfigurierbar): `Follow the heartbeat monitor scratch context when provided. Recurring tasks are cron jobs; create or change their schedules with cron tools or the openclaw cron CLI, not heartbeat scratch. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.`
- Zeitüberschreitung: Heartbeat-Durchläufe ohne festgelegten Wert verwenden `agents.defaults.timeoutSeconds`, sofern dieser Wert gesetzt ist. Andernfalls verwenden sie den Heartbeat-Takt, begrenzt auf 600 Sekunden. Legen Sie für längere Heartbeat-Arbeiten `agents.defaults.heartbeat.timeoutSeconds` oder agentenspezifisch `agents.entries.*.heartbeat.timeoutSeconds` fest.
- Der Heartbeat-Prompt wird **unverändert** als Benutzernachricht gesendet. Der System-Prompt enthält einen Abschnitt „Heartbeats“, wenn Heartbeats für den Standardagenten aktiviert sind, und die Ausführung wird intern entsprechend gekennzeichnet.
- Wenn Heartbeats mit `0m` deaktiviert werden, bleibt der cron-Überwachungsauftrag bestehen, wird jedoch deaktiviert. Seine Notizen bleiben erhalten, bis Sie den Takt wieder aktivieren.
- Wenn cron selbst deaktiviert ist, werden geplante Heartbeats nicht ausgeführt, auch wenn der Heartbeat-Takt weiterhin aktiviert ist.
- Aktive Zeiten (`heartbeat.activeHours`) werden in der konfigurierten Zeitzone geprüft. Außerhalb des Zeitfensters werden Heartbeats bis zum nächsten Takt innerhalb des Fensters übersprungen.
- Heartbeats werden automatisch zurückgestellt, solange cron-Arbeiten aktiv sind oder sich in der Warteschlange befinden oder solange die sitzungsschlüsselgebundenen Subagenten- oder verschachtelten Befehls-Lanes dieses Agenten ausgelastet sind. Gleichgeordnete Agenten pausieren einander nicht.

## Zweck des Heartbeat-Prompts

Der Standard-Prompt ist bewusst allgemein gehalten:

- **Hintergrundaufgaben**: „Ausstehende Aufgaben berücksichtigen“ fordert den Agenten auf, Folgemaßnahmen zu prüfen (Posteingang, Kalender, Erinnerungen, Arbeiten in der Warteschlange) und auf dringende Punkte hinzuweisen.
- **Nachfrage beim Menschen**: „Gelegentlich tagsüber nach dem Menschen sehen“ regt zu einer gelegentlichen kurzen Nachricht wie „Benötigen Sie etwas?“ an, vermeidet jedoch durch Verwendung Ihrer konfigurierten lokalen Zeitzone nächtliche Nachrichtenfluten (siehe [Zeitzone](/de/concepts/timezone)).

Heartbeat kann auf abgeschlossene [Hintergrundaufgaben](/de/automation/tasks) reagieren, aber eine Heartbeat-Ausführung selbst erstellt keinen Aufgabendatensatz.

Wenn ein Heartbeat etwas ganz Bestimmtes tun soll (z. B. „Gmail-PubSub-Statistiken prüfen“ oder „Gateway-Zustand überprüfen“), legen Sie `agents.defaults.heartbeat.prompt` (oder `agents.entries.*.heartbeat.prompt`) auf einen benutzerdefinierten Inhalt fest (wird unverändert gesendet).

## Antwortvertrag

- Wenn nichts beachtet werden muss, antworten Sie mit **`HEARTBEAT_OK`**.
- Heartbeat-Ausführungen können stattdessen `heartbeat_respond` mit `notify: false` aufrufen, wenn keine sichtbare Aktualisierung erfolgen soll, oder `notify: true` zusammen mit `notificationText` für eine Warnung. Falls vorhanden, hat die strukturierte Tool-Antwort Vorrang vor dem textbasierten Rückfall.
- Ein aussagekräftiges `heartbeat_respond`-Ergebnis mit `notify: false` bleibt unsichtbar, wird jedoch als begrenzter interner Kontext für den nächsten Benutzerdurchlauf in dieser Sitzung gespeichert. `no_change`-Bestätigungen und sichtbare Benachrichtigungen werden nicht auf diese Weise gespeichert.
- Während Heartbeat-Ausführungen behandelt OpenClaw `HEARTBEAT_OK` als Bestätigung, wenn es am **Anfang oder Ende** der Antwort erscheint. Das Token wird entfernt und die Antwort verworfen, wenn der verbleibende Inhalt höchstens 300 Zeichen umfasst.
- Wenn `HEARTBEAT_OK` in der **Mitte** einer Antwort erscheint, wird es nicht besonders behandelt.
- Fügen Sie bei Warnungen `HEARTBEAT_OK` **nicht** ein; geben Sie ausschließlich den Warntext zurück.

Außerhalb von Heartbeats wird ein unbeabsichtigtes `HEARTBEAT_OK` am Anfang oder Ende einer Nachricht entfernt und protokolliert; eine Nachricht, die nur aus `HEARTBEAT_OK` besteht, wird verworfen.

## Konfiguration

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m", // Standard: 30m (0m deaktiviert)
        model: "anthropic/claude-opus-4-6",
        lightContext: false, // Standard: false; true überspringt Workspace-Bootstrap-Dateien für Heartbeat-Ausführungen
        isolatedSession: false, // Standard: false; true führt jeden Heartbeat in einer neuen Sitzung aus (kein Gesprächsverlauf)
        target: "last", // Standard: none | Optionen: last | none | <channel id> (Kern oder Plugin, z. B. "imessage")
        to: "+15551234567", // optionale kanalspezifische Überschreibung
        accountId: "ops-bot", // optionale Kanal-ID bei mehreren Konten
        prompt: "Befolgen Sie den Kontext der Heartbeat-Überwachungsnotizen, sofern er bereitgestellt wird. Wiederkehrende Aufgaben sind cron-Aufträge; erstellen oder ändern Sie deren Zeitpläne mit cron-Tools oder der openclaw cron CLI, nicht mit Heartbeat-Notizen. Leiten Sie keine alten Aufgaben aus früheren Chats ab und wiederholen Sie diese nicht. Wenn nichts beachtet werden muss, antworten Sie mit HEARTBEAT_OK.",
      },
    },
  },
}
```

### Geltungsbereich und Rangfolge

- `agents.defaults.heartbeat` legt das globale Heartbeat-Verhalten fest.
- `agents.entries.*.heartbeat` wird darübergelegt; wenn ein Agent einen `heartbeat`-Block besitzt, führen **nur diese Agenten** Heartbeats aus.
- `channels.defaults.heartbeatVisibility` legt die Sichtbarkeitsstandards für alle Kanäle fest.
- `channels.<channel>.heartbeatVisibility` überschreibt die Kanalstandards.
- `channels.<channel>.accounts.<id>.heartbeatVisibility` (Kanäle mit mehreren Konten) überschreibt die kanalspezifischen Einstellungen.

### Agentenspezifische Heartbeats

Wenn ein `agents.entries.*`-Eintrag einen `heartbeat`-Block enthält, führen **nur diese Agenten** Heartbeats aus. Der agentenspezifische Block wird über `agents.defaults.heartbeat` gelegt (Sie können daher gemeinsame Standardwerte einmalig festlegen und sie für einzelne Agenten überschreiben).

Beispiel: zwei Agenten, wobei nur der zweite Agent Heartbeats ausführt.

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // explizite Zustellung an den letzten Kontakt (Standard ist "none")
      },
    },
    list: [
      { id: "main", default: true },
      {
        id: "ops",
        heartbeat: {
          every: "1h",
          target: "whatsapp",
          to: "+15551234567",
          timeoutSeconds: 45,
          prompt: "Befolgen Sie den Kontext der Heartbeat-Überwachungsnotizen, sofern er bereitgestellt wird. Wiederkehrende Aufgaben sind cron-Aufträge; erstellen oder ändern Sie deren Zeitpläne mit cron-Tools oder der openclaw cron CLI, nicht mit Heartbeat-Notizen. Leiten Sie keine alten Aufgaben aus früheren Chats ab und wiederholen Sie diese nicht. Wenn nichts beachtet werden muss, antworten Sie mit HEARTBEAT_OK.",
        },
      },
    ],
  },
}
```

### Beispiel für aktive Zeiten

Beschränken Sie Heartbeats auf Geschäftszeiten in einer bestimmten Zeitzone:

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // explizite Zustellung an den letzten Kontakt (Standard ist "none")
        activeHours: {
          start: "09:00",
          end: "22:00",
          timezone: "America/New_York", // optional; verwendet userTimezone, falls festgelegt, andernfalls die Zeitzone des Hosts
        },
      },
    },
  },
}
```

Außerhalb dieses Zeitfensters (vor 9 Uhr oder nach 22 Uhr Eastern Time) werden Heartbeats übersprungen. Der nächste geplante Takt innerhalb des Fensters wird normal ausgeführt.

### Einrichtung von 24/7

Wenn Heartbeats ganztägig ausgeführt werden sollen, verwenden Sie eines dieser Muster:

- Lassen Sie `activeHours` vollständig weg (keine Zeitfensterbeschränkung; dies ist das Standardverhalten).
- Legen Sie ein ganztägiges Zeitfenster fest: `activeHours: { start: "00:00", end: "24:00" }`.

<Warning>
Legen Sie für `start` und `end` nicht dieselbe Uhrzeit fest (beispielsweise `08:00` bis `08:00`). Dies wird als Zeitfenster mit einer Breite von null behandelt, sodass Heartbeats immer übersprungen werden.
</Warning>

### Beispiel mit mehreren Konten

Verwenden Sie `accountId`, um auf Kanälen mit mehreren Konten wie Telegram ein bestimmtes Konto auszuwählen:

```json5
{
  agents: {
    list: [
      {
        id: "ops",
        heartbeat: {
          every: "1h",
          target: "telegram",
          to: "12345678:topic:42", // optional: an ein bestimmtes Thema/einen bestimmten Thread weiterleiten
          accountId: "ops-bot",
        },
      },
    ],
  },
  channels: {
    telegram: {
      accounts: {
        "ops-bot": { botToken: "YOUR_TELEGRAM_BOT_TOKEN" },
      },
    },
  },
}
```

### Feldhinweise

<ParamField path="every" type="string">
  Heartbeat-Intervall (Zeitdauerzeichenfolge; Standardeinheit = Minuten).
</ParamField>
<ParamField path="model" type="string">
  Optionale Modellüberschreibung für Heartbeat-Ausführungen (`provider/model`).
</ParamField>
<ParamField path="lightContext" type="boolean" default="false">
  Bei true verwenden Heartbeat-Ausführungen einen schlanken Bootstrap-Kontext und überspringen Workspace-Bootstrap-Dateien. Die Überwachungsnotizen werden in jedem Fall vom Heartbeat-Runner eingefügt.
</ParamField>
<ParamField path="isolatedSession" type="boolean" default="false">
  Bei true wird jeder Heartbeat in einer neuen Sitzung ohne vorherigen Gesprächsverlauf ausgeführt. Verwendet dasselbe Isolationsmuster wie cron `sessionTarget: "isolated"`. Reduziert die Token-Kosten pro Heartbeat erheblich. Kombinieren Sie dies mit `lightContext: true`, um maximale Einsparungen zu erzielen. Die Zustellungsweiterleitung verwendet weiterhin den Kontext der Hauptsitzung.
</ParamField>
<ParamField path="session" type="string">
  Optionaler Sitzungsschlüssel für Heartbeat-Ausführungen.

- `main` (Standard): Hauptsitzung des Agenten.
- Expliziter Sitzungsschlüssel (aus `openclaw sessions --json` oder der [Sitzungs-CLI](/de/cli/sessions) kopieren).
- Formate für Sitzungsschlüssel: siehe [Sitzungen](/de/concepts/session) und [Gruppen](/de/channels/groups).

</ParamField>
<ParamField path="target" type="string">
- `last`: an den zuletzt verwendeten externen Kanal zustellen.
- Expliziter Kanal: eine beliebige konfigurierte Kanal- oder Plugin-ID, zum Beispiel `discord`, `matrix`, `telegram` oder `whatsapp`.
- `none` (Standard): den Heartbeat ausführen, aber **nicht extern zustellen**.

</ParamField>
<ParamField path="directPolicy" type='"allow" | "block"' default="allow">
  Steuert das Zustellungsverhalten für direkte Nachrichten/DMs. `allow`: Heartbeat-Zustellung für direkte Nachrichten/DMs zulassen. `block`: Zustellung für direkte Nachrichten/DMs unterdrücken (`reason=dm-blocked`).

</ParamField>
<ParamField path="to" type="string">
  Optionale Überschreibung des Empfängers (kanalspezifische ID, z. B. E.164 für WhatsApp oder eine Telegram-Chat-ID). Verwenden Sie für Telegram-Themen/Threads `<chatId>:topic:<messageThreadId>`.

</ParamField>
<ParamField path="accountId" type="string">
  Optionale Konto-ID für Kanäle mit mehreren Konten. Bei `target: "last"` gilt die Konto-ID für den ermittelten letzten Kanal, sofern dieser Konten unterstützt; andernfalls wird sie ignoriert. Wenn die Konto-ID keinem konfigurierten Konto des ermittelten Kanals entspricht, wird die Zustellung übersprungen.

</ParamField>
<ParamField path="prompt" type="string">
  Überschreibt den standardmäßigen Prompt-Inhalt (wird nicht zusammengeführt).

</ParamField>
<ParamField path="timeoutSeconds" type="number" default="global timeout or min(every, 600)">
  Maximale Anzahl von Sekunden, die ein Heartbeat-Agentendurchlauf dauern darf, bevor er abgebrochen wird. Lassen Sie dies ungesetzt, um `agents.defaults.timeoutSeconds` zu verwenden, sofern festgelegt; andernfalls wird das Heartbeat-Intervall mit einer Obergrenze von 600 Sekunden verwendet.

</ParamField>
<ParamField path="activeHours" type="object">
  Beschränkt Heartbeat-Ausführungen auf ein Zeitfenster. Objekt mit `start` (HH:MM, einschließlich; verwenden Sie `00:00` für den Tagesbeginn), `end` (HH:MM, ausschließlich; `24:00` für das Tagesende zulässig) und optional `timezone`.

- Nicht angegeben oder `"user"`: verwendet Ihre Einstellung `agents.defaults.userTimezone`, sofern festgelegt; andernfalls wird auf die Zeitzone des Hostsystems zurückgegriffen.
- `"local"`: verwendet immer die Zeitzone des Hostsystems.
- Beliebige IANA-Kennung (z. B. `America/New_York`): wird direkt verwendet; ist sie ungültig, wird auf das oben beschriebene Verhalten von `"user"` zurückgegriffen.
- `start` und `end` dürfen für ein aktives Zeitfenster nicht gleich sein; gleiche Werte werden als Fenster mit einer Breite von null behandelt (immer außerhalb des Fensters).
- Außerhalb des aktiven Zeitfensters werden Heartbeats bis zum nächsten Tick innerhalb des Fensters übersprungen.

</ParamField>

## Zustellungsverhalten

<AccordionGroup>
  <Accordion title="Sitzungs- und Zielrouting">
    - Heartbeats werden standardmäßig in der Hauptsitzung des Agenten ausgeführt (`agent:<id>:<mainKey>`) oder in `global`, wenn `session.scope = "global"`. Legen Sie `session` fest, um dies mit einer bestimmten Kanalsitzung (Discord/WhatsApp/usw.) zu überschreiben.
    - `session` wirkt sich nur auf den Ausführungskontext aus; die Zustellung wird durch `target` und `to` gesteuert.
    - Um an einen bestimmten Kanal/Empfänger zuzustellen, legen Sie `target` + `to` fest. Mit `target: "last"` verwendet die Zustellung den letzten externen Kanal dieser Sitzung.
    - Heartbeat-Zustellungen lassen standardmäßig direkte Ziele/DM-Ziele zu. Legen Sie `directPolicy: "block"` fest, um Sendungen an direkte Ziele zu unterdrücken, während der Heartbeat-Durchlauf weiterhin ausgeführt wird.
    - Wenn die Hauptwarteschlange, die Ziel-Sitzungsspur, die Cron-Spur oder ein aktiver Cron-Job ausgelastet ist, wird der Heartbeat übersprungen und später erneut versucht.
    - Wenn `target` kein externes Ziel ergibt, wird der Durchlauf dennoch ausgeführt, aber keine ausgehende Nachricht gesendet.

  </Accordion>
  <Accordion title="Sichtbarkeits- und Überspringverhalten">
    - Wenn `showOk`, `showAlerts` und `useIndicator` alle deaktiviert sind, wird der Durchlauf vorab als `reason=alerts-disabled` übersprungen.
    - Wenn nur die Alarmzustellung deaktiviert ist, kann OpenClaw den Heartbeat dennoch ausführen, die Zeitstempel fälliger Aufgaben aktualisieren, den Leerlauf-Zeitstempel der Sitzung wiederherstellen und die nach außen gerichtete Alarmnutzlast unterdrücken.
    - Wenn das ermittelte Heartbeat-Ziel Tippanzeigen unterstützt, zeigt OpenClaw während des aktiven Heartbeat-Durchlaufs eine Tippanzeige an. Dabei wird dasselbe Ziel verwendet, an das der Heartbeat Chat-Ausgaben senden würde; durch `typingMode: "never"` wird dies deaktiviert.

  </Accordion>
  <Accordion title="Sitzungslebenszyklus und Audit">
    - Reine Heartbeat-Antworten halten die Sitzung **nicht** aktiv. Heartbeat-Metadaten können die Sitzungszeile aktualisieren, für den Ablauf wegen Inaktivität wird jedoch `lastInteractionAt` aus der letzten echten Benutzer-/Kanalnachricht verwendet und für den täglichen Ablauf `sessionStartedAt`.
    - Der Verlauf in Control UI und WebChat blendet Heartbeat-Prompts und reine OK-Bestätigungen aus. Das zugrunde liegende Sitzungsprotokoll kann diese Durchläufe für Audit/Wiedergabe weiterhin enthalten.
    - Abgekoppelte [Hintergrundaufgaben](/de/automation/tasks) können ein Systemereignis in die Warteschlange stellen und den Heartbeat aktivieren, wenn die Hauptsitzung schnell auf etwas aufmerksam werden soll. Diese Aktivierung macht den Heartbeat-Durchlauf nicht zu einer Hintergrundaufgabe.

  </Accordion>
</AccordionGroup>

## Sichtbarkeitssteuerung

Standardmäßig werden `HEARTBEAT_OK`-Bestätigungen unterdrückt, während Alarminhalte zugestellt werden. Sie können dies pro Kanal oder pro Konto anpassen:

```yaml
channels:
  defaults:
    heartbeat:
      showOk: false # HEARTBEAT_OK ausblenden (Standard)
      showAlerts: true # Alarmmeldungen anzeigen (Standard)
      useIndicator: true # Indikatorereignisse ausgeben (Standard)
  telegram:
    heartbeat:
      showOk: true # OK-Bestätigungen auf Telegram anzeigen
  whatsapp:
    accounts:
      work:
        heartbeat:
          showAlerts: false # Alarmzustellung für dieses Konto unterdrücken
```

Priorität: pro Konto → pro Kanal → Kanalstandards → integrierte Standards.

### Funktion der einzelnen Flags

- `showOk`: sendet eine `HEARTBEAT_OK`-Bestätigung, wenn das Modell eine reine OK-Antwort zurückgibt.
- `showAlerts`: sendet den Alarminhalt, wenn das Modell eine andere als eine OK-Antwort zurückgibt.
- `useIndicator`: gibt Indikatorereignisse für UI-Statusoberflächen aus.

Wenn **alle drei** auf false gesetzt sind, überspringt OpenClaw den Heartbeat-Durchlauf vollständig (kein Modellaufruf).

### Beispiele für Einstellungen pro Kanal und pro Konto

```yaml
channels:
  defaults:
    heartbeat:
      showOk: false
      showAlerts: true
      useIndicator: true
  slack:
    heartbeat:
      showOk: true # alle Slack-Konten
    accounts:
      ops:
        heartbeat:
          showAlerts: false # Alarme nur für das ops-Konto unterdrücken
  telegram:
    heartbeat:
      showOk: true
```

### Häufige Muster

| Ziel                                             | Konfiguration                                                                            |
| ------------------------------------------------ | ---------------------------------------------------------------------------------------- |
| Standardverhalten (stille OKs, Alarme aktiviert) | _(keine Konfiguration erforderlich)_                                                     |
| Vollständig still (keine Nachrichten, kein Indikator) | `channels.defaults.heartbeat: { showOk: false, showAlerts: false, useIndicator: false }` |
| Nur Indikator (keine Nachrichten)                 | `channels.defaults.heartbeat: { showOk: false, showAlerts: false, useIndicator: true }`  |
| OKs nur in einem Kanal                            | `channels.telegram.heartbeat: { showOk: true }`                                          |

## Monitor-Notizen (optional)

Jeder Cron-Job des Heartbeat-Monitors besitzt ein privates Notizdokument, das in der gemeinsamen Zustandsdatenbank gespeichert ist. Betrachten Sie es als Ihre „Heartbeat-Checkliste“: klein, stabil und sicher alle 30 Minuten zu berücksichtigen. Wenn Notizen vorhanden sind, wird ihr Inhalt an den Heartbeat-Prompt angehängt.

Verwalten Sie sie mit der Cron-CLI (die Job-ID stammt aus `openclaw cron list --all`):

```bash
openclaw cron scratch <jobId>                 # aktuelle Notizen ausgeben
openclaw cron scratch <jobId> --set "..."     # durch den exakten Text ersetzen
openclaw cron scratch <jobId> --file notes.md # durch den Inhalt einer Datei ersetzen (- für stdin)
openclaw cron scratch <jobId> --unset         # entfernen
```

Schreibvorgänge sind durch Compare-and-Swap geschützt: Übergeben Sie `--expected-revision <n>`, damit der Vorgang fehlschlägt, statt eine gleichzeitige Bearbeitung zu überschreiben. Die Notizen sind auf 256 KiB begrenzt und erscheinen niemals in der Ausgabe von `cron list`/`cron runs`.

Der Agent kann auch seine eigenen Notizen aktualisieren: Während eines Heartbeat-Durchlaufs akzeptiert `heartbeat_respond` eine optionale Zeichenfolge `scratch`, die die Notizen des Monitors für zukünftige Heartbeats vollständig ersetzt.

<Note>
**Migration von HEARTBEAT.md oder einem ausschließlich über die Konfiguration festgelegten Intervall?** Führen Sie `openclaw doctor --fix` aus. Doctor erstellt oder aktualisiert zunächst die systemeigenen Monitorzeilen anhand von `agents.*.heartbeat`, importiert dann die `HEARTBEAT.md` aus dem Arbeitsbereich jedes Agenten in die Notizen des Monitors, wandelt gültige ältere `tasks:`-Einträge in Cron-Jobs um, archiviert das Original im Zustandsverzeichnis (`backups/heartbeat-migration/`) und entfernt die Datei. Heartbeat-Anweisungen zur Laufzeit stammen ausschließlich aus den Datenbanknotizen; die Laufzeit liest `HEARTBEAT.md` niemals.
</Note>

Wenn Notizen vorhanden, aber praktisch leer sind (nur Leerzeilen, Markdown-/HTML-Kommentare, Markdown-Überschriften wie `# Heading`, Fence-Markierungen oder leere Checklisten-Platzhalter), überspringt OpenClaw den Heartbeat-Durchlauf, um API-Aufrufe zu sparen. Dieses Überspringen wird als `reason=empty-heartbeat-file` gemeldet. Sind keine Notizen vorhanden, wird der Heartbeat dennoch ausgeführt und das Modell entscheidet, was zu tun ist.

Halten Sie sie knapp (kurze Checkliste oder Erinnerungen), um ein unnötiges Anwachsen des Prompts zu vermeiden.

Beispielnotizen:

```md
# Heartbeat-Checkliste

- Kurz prüfen: Gibt es etwas Dringendes in den Posteingängen?
- Wenn es Tag ist und sonst nichts ansteht, kurz und unaufwendig nachfragen.
- Wenn eine Aufgabe blockiert ist, notieren, _was fehlt_, und Peter beim nächsten Mal fragen.
```

### Wiederkehrende Prüfungen mit Cron planen

Heartbeat-Notizen sind Prompt-Kontext und kein Zeitplaner. Erstellen Sie jede wiederkehrende Prüfung als [Cron-Job](/de/automation/cron-jobs), damit sie über ein eigenes Intervall, einen eigenen Aktivierungsstatus und einen eigenen Ausführungsverlauf verfügt. Cron-Jobs können weiterhin auf die Hauptsitzung abzielen, wenn für die Prüfung der normale Gesprächskontext verwendet werden soll.

Ältere Notizen können einen strukturierten `tasks:`-Block enthalten. Führen Sie nach dem Upgrade einmal `openclaw doctor --fix` aus: Doctor wandelt jeden gültigen Eintrag in einen unabhängig geplanten Cron-Job um, behält sein Intervall und den vorherigen Zeitpunkt der letzten Ausführung bei und entfernt den eingestellten Block, während der umgebende Notiztext erhalten bleibt. Heartbeat-Durchläufe zur Laufzeit interpretieren `tasks:`-Text nicht als Zeitpläne.

Von Doctor erstellte Heartbeat-Aufgabenjobs behalten die aktiven Heartbeat-Zeiten sowie die Schutzmechanismen für Abkühlzeit, Überlastung und Auslastung bei. Gleichzeitig fällige Jobs können zu einem einzigen Heartbeat-Durchlauf zusammengefasst werden. Ein Vorkommen außerhalb der aktiven Zeiten wird übersprungen und beim nächsten Cron-Vorkommen erneut versucht.

### Kann der Agent seine Notizen aktualisieren?

Ja. Während eines Heartbeat-Durchlaufs kann der Agent einen `scratch`-Wert an `heartbeat_respond` übergeben, um den Monitor-Text für zukünftige Heartbeats vollständig zu ersetzen. Sie können ihn auch in einem normalen Chat auffordern, `openclaw cron scratch <jobId> --set ...` auszuführen, oder die Notizen selbst mit demselben Befehl bearbeiten. Verwalten Sie wiederkehrende Zeitpläne mit Cron, statt Zeitplanersyntax in die Notizen zu schreiben.

<Warning>
Speichern Sie keine Geheimnisse (API-Schlüssel, Telefonnummern, private Tokens) in den Monitor-Notizen – sie werden Teil des Prompt-Kontexts.
</Warning>

## Manuelle Aktivierung (bei Bedarf)

Verwenden Sie `openclaw system event`, um ein Systemereignis in die Warteschlange zu stellen und optional sofort einen Heartbeat auszulösen:

```bash
openclaw system event --text "Auf dringende Nachfassaktionen prüfen" --mode now
```

| Flag                         | Beschreibung                                                                                      |
| ---------------------------- | ------------------------------------------------------------------------------------------------ |
| `--text <text>`              | Systemereignistext (erforderlich).                                                                    |
| `--mode <mode>`              | `now` führt sofort einen Heartbeat aus; `next-heartbeat` (Standard) wartet auf den nächsten geplanten Tick. |
| `--session-key <sessionKey>` | Richtet das Ereignis an eine bestimmte Sitzung; standardmäßig wird die Hauptsitzung des Agenten verwendet.                   |
| `--json`                     | Gibt JSON aus.                                                                                     |

Wenn kein `--session-key` angegeben ist und für mehrere Agenten `heartbeat` konfiguriert ist, führt `--mode now` die Heartbeats all dieser Agenten sofort aus.

Zugehörige Heartbeat-Steuerelemente in derselben CLI-Gruppe:

```bash
openclaw system heartbeat last     # letztes Heartbeat-Ereignis anzeigen
openclaw system heartbeat enable   # Heartbeats aktivieren
openclaw system heartbeat disable  # Heartbeats deaktivieren
```

## Kostenbewusstsein

Heartbeats führen vollständige Agentendurchläufe aus. Kürzere Intervalle verbrauchen mehr Token. So lassen sich die Kosten reduzieren:

- Verwenden Sie `isolatedSession: true`, damit nicht der vollständige Konversationsverlauf gesendet wird (~100K Token werden auf ~2-5K pro Ausführung reduziert).
- Verwenden Sie `lightContext: true`, um Workspace-Bootstrap-Dateien bei Heartbeat-Ausführungen zu überspringen.
- Legen Sie ein kostengünstigeres `model` fest (z. B. `ollama/llama3.2:1b`).
- Halten Sie den temporären Monitorbereich klein.
- Verwenden Sie `target: "none"`, wenn Sie nur interne Statusaktualisierungen wünschen.

## Kontextüberlauf nach einem Heartbeat

Heartbeats behalten nach Abschluss der Ausführung das vorhandene Laufzeitmodell der gemeinsam genutzten Sitzung bei. Daher kann ein Heartbeat, der eine Sitzung auf ein kleineres lokales Modell umgestellt hat (beispielsweise ein Ollama-Modell mit einem 32k-Kontextfenster), dieses Modell für den nächsten Durchlauf der Hauptsitzung aktiv lassen. Wenn dieser nächste Durchlauf dann einen Kontextüberlauf meldet und das zuletzt verwendete Laufzeitmodell der Sitzung mit dem konfigurierten `heartbeat.model` übereinstimmt, nennt die Wiederherstellungsmeldung von OpenClaw die Übernahme des Heartbeat-Modells als wahrscheinliche Ursache und schlägt eine Korrektur vor.

So vermeiden Sie dies: Verwenden Sie `isolatedSession: true`, um Heartbeats in einer neuen Sitzung auszuführen (optional in Kombination mit `lightContext: true` für den kleinstmöglichen Prompt), oder wählen Sie ein Heartbeat-Modell mit einem Kontextfenster, das groß genug für die gemeinsam genutzte Sitzung ist.

## Verwandte Themen

- [Automatisierung](/de/automation) – alle Automatisierungsmechanismen auf einen Blick
- [Hintergrundaufgaben](/de/automation/tasks) – wie abgekoppelte Arbeiten nachverfolgt werden
- [Zeitzone](/de/concepts/timezone) – wie sich die Zeitzone auf die Heartbeat-Planung auswirkt
- [Fehlerbehebung](/de/automation/cron-jobs#troubleshooting) – Fehler bei der Automatisierung diagnostizieren
