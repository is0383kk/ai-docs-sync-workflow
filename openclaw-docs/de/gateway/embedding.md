---
read_when:
    - OpenClaw in eine Desktop- oder Serveranwendung einbetten
    - Überwachung des Gateway als untergeordneter Prozess
    - Gateway-Bereitschaft, Neustart, Herunterfahren oder ungültige Konfiguration ohne Auswertung von Protokollen handhaben
summary: Den OpenClaw Gateway als untergeordneten Prozess über Electron oder eine andere Host-App überwachen
title: OpenClaw einbetten
x-i18n:
    generated_at: "2026-07-26T17:50:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ca67e03994f21446bfeca58c95c2cb624dde767b9983a89982627145f80dfb90
    source_path: gateway/embedding.md
    workflow: 16
---

Ein einbettender Host sollte die installierte ausführbare Datei `openclaw` überwachen, das
Gateway-WebSocket-Protokoll als Steuerungsebene verwenden und den Kindprozess als
austauschbare Laufzeit behandeln. Dadurch bleiben Prozessverantwortung, Bereitschaft, Wiederherstellung nach Fehlern
und Upgrades explizit, ohne von der privaten Zustandsstruktur von OpenClaw abhängig zu sein.

Informationen zur Client-Authentifizierung und zum Wiederverbindungsstatus finden Sie unter
[Erstellen eines Gateway-Clients](https://docs.openclaw.ai/gateway/clients).

## Kindprozess mit einer Einbettungsvoreinstellung starten

Verwenden Sie eine echte `node_modules`-Installation und starten Sie die ausführbare Datei des Pakets. Eine sinnvolle
Ausgangsbasis für einen Host, der Erkennung, Neustart und den Lebenszyklus der Kanäle verwaltet, ist:

```ts
import { spawn } from "node:child_process";
import { dirname, resolve } from "node:path";
import { fileURLToPath } from "node:url";

// Geben Sie einen absoluten Pfad zu einer echten Node-Laufzeit an, die von der Hostanwendung verwaltet wird.
declare const hostNodeExecutable: string;

const packageEntry = fileURLToPath(import.meta.resolve("openclaw"));
const openclawEntry = resolve(dirname(packageEntry), "..", "openclaw.mjs");
const gateway = spawn(hostNodeExecutable, [openclawEntry, "gateway", "--allow-unconfigured"], {
  env: {
    ...process.env,
    OPENCLAW_DISABLE_BONJOUR: "1",
    OPENCLAW_EXEC_SHELL_SNAPSHOT: "0",
    OPENCLAW_NO_RESPAWN: "1",
    OPENCLAW_SKIP_CHANNELS: "1",
  },
  stdio: ["ignore", "inherit", "inherit"],
});
```

Lösen Sie OpenClaw wie gezeigt über das installierte Paket auf; gehen Sie nicht davon aus, dass eine
projektlokale ausführbare Datei `openclaw` im `PATH` des Hostprozesses enthalten ist. Das Beispiel
erbt die Ausgabe, damit der Kindprozess nicht durch volle stdout- oder stderr-Pipes blockiert werden kann. Wenn der
Host diese Streams stattdessen erfasst, binden Sie unmittelbar nach dem Start Verbraucher an.

| Einstellung                      | Auswirkung auf die Einbettung                                                                                                                                                             |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `OPENCLAW_DISABLE_BONJOUR=1`     | Deaktiviert die vom Gateway verwaltete LAN-Multicast-Ankündigung, wenn der Host die Erkennung verwaltet.                                                                                   |
| `OPENCLAW_NO_RESPAWN=1`          | Verhindert in einem nicht verwalteten eingebetteten Kindprozess, dass OpenClaw einen Update-Neustart an einen abgetrennten Kindprozess übergibt. Reguläre Neustarts verbleiben im Prozess, sodass der Host die Verantwortung für die verfolgte PID behält. |
| `OPENCLAW_EXEC_SHELL_SNAPSHOT=0` | Deaktiviert die Erfassung eines Login-Shell-Snapshots für Ausführungsbefehle des Hosts.                                                                                                    |
| `OPENCLAW_SKIP_CHANNELS=1`       | Überspringt den Start und das Neuladen von Kanälen. Legen Sie dies nur fest, wenn die einbettende App ein Gateway ausschließlich für die Steuerungsebene oder WebChat verwenden soll.      |

`--allow-unconfigured` umgeht nur die Startprüfung `gateway.mode=local`. Die Option
schreibt keine Konfiguration und repariert keine ungültige Datei. Lassen Sie sie weg, wenn die einbettende
App über das Onboarding, die Konfigurations-CLI oder Gateway-RPC eine normale lokale Konfiguration bereitstellt.

### Warnung zum Electron-Shell-Snapshot

Bei der Erfassung des Shell-Snapshots wird `process.execPath -e <script>` aus einer Login-Shell ausgeführt. In
einem normalen Node-Prozess ist `process.execPath` die ausführbare Node-Datei. Unter Electron
ist es die Electron-Binärdatei, die den Aufruf als Anwendungsstart interpretieren
und ein Popup mit der Meldung „Unable to find Electron app“ anzeigen kann. Legen Sie
`OPENCLAW_EXEC_SHELL_SNAPSHOT=0` in der Umgebung des Gateway-Kindprozesses fest, nicht nur im
Renderer-Prozess. Aus demselben Grund muss `hostNodeExecutable` auf eine
echte Node-Laufzeit und nicht auf Electrons `process.execPath` verweisen.

## Ungültige Konfiguration anhand des Exit-Codes behandeln

Beim Start verwendet das Gateway den Exit-Code `78` (`EX_CONFIG`) für konfigurationsbedingte
Startfehler, einschließlich einer ungültigen Konfiguration. Verzweigen Sie anhand des Exit-Codes, anstatt
menschenlesbare stderr-Ausgaben auszuwerten:

1. Führen Sie `openclaw doctor --fix --yes --non-interactive` mit derselben Konfigurations- und
   Zustandsumgebung wie der Gateway-Kindprozess aus.
2. Versuchen Sie den Gateway-Start einmal erneut, nachdem Doctor erfolgreich beendet wurde.
3. Wenn der Kindprozess erneut mit `78` beendet wird, beenden Sie die Reparaturschleife und zeigen Sie dem Benutzer
   den Konfigurationsfehler an.

Bewahren Sie stderr für Diagnosezwecke auf, treffen Sie jedoch keine Entscheidungen zum Lebenszyklus anhand des Wortlauts.

Nach einem erfolgreichen Start hat eine ungültige Änderung der aktiven Konfiguration weniger schwerwiegende Folgen. Der
Konfigurations-Watcher protokolliert, dass das Neuladen übersprungen wurde, und verwendet weiterhin die zuletzt akzeptierte
In-Memory-Konfiguration. Reparieren Sie die Datei und lassen Sie den Watcher anschließend den nächsten gültigen
Snapshot übernehmen.

## Auf Protokollbereitschaft warten

Verwenden Sie WebSocket-Signale statt einer Teilzeichenfolge im Protokoll:

1. Öffnen Sie den Gateway-WebSocket.
2. Warten Sie auf das Ereignis `connect.challenge`. Es belegt, dass der Listener den
   WebSocket akzeptiert hat und der Challenge-Handshake beginnen kann.
3. Senden Sie `connect` mit der an die Challenge gebundenen Gerätesignatur.
4. Behandeln Sie `hello-ok` als Anwendungsbereitschaft für authentifizierte RPC-Aufrufe.

Die Challenge erfolgt absichtlich vor der vollständigen Initialisierung. Wenn beim Start
Nebenprozesse noch ausstehen, gibt `connect` einen wiederholbaren Fehler `UNAVAILABLE` mit
`details.reason: "startup-sidecars"` und einem begrenzten `retryAfterMs` zurück und schließt anschließend
mit dem Code `1013` und dem Grund `gateway starting`. Verwenden Sie
`resolveGatewayStartupRetryAfterMs` aus
`@openclaw/gateway-protocol/startup-unavailable` oder die integrierte
Richtlinie des Referenzclients und stellen Sie anschließend die Verbindung erneut her.

## Neustart und Herunterfahren interpretieren

Vor einem geordneten Schließen sendet das Gateway ein Ereignis `shutdown` mit `reason`
und `restartExpectedMs`. Ein nicht nullwertiges `restartExpectedMs` bedeutet, dass ein prozessinterner oder
überwachter Neustart erwartet wird; `null` bedeutet ein endgültiges Herunterfahren.

Der anschließende WebSocket-Schließcode lautet in beiden Fällen `1012`. Der gewöhnliche
Schließgrund des Clients lautet ebenfalls in beiden Fällen `service restart`, sodass weder der Schließcode noch
der Grund zwischen Neustart und Herunterfahren unterscheiden. Bewahren Sie die vorherige `shutdown`-Nutzlast
auf, wenn sie eintrifft, und kombinieren Sie sie mit der eigenen Beendigungsabsicht des Hosts und dem
Exit-Status des Kindprozesses. Wenn die Verbindung ohne dieses Ereignis abbricht, verwenden Sie die normale
begrenzte Richtlinie für Wiederverbindungen und die Überwachung des Kindprozesses.

## RPC statt Zustandsdateien verwenden

Behalten Sie das Gateway als alleinigen Eigentümer des OpenClaw-Zustands bei. Für gängige Einbettungsoperationen
stehen bereits RPC-Methoden zur Verfügung:

| Aufgabe                       | RPC-Methoden                                          |
| ----------------------------- | ---------------------------------------------------- |
| Sitzungskatalog und -lebenszyklus | `sessions.list`, `sessions.patch`, `sessions.delete` |
| Transkriptanzeige             | `chat.history`                                       |
| Kosten- und Nutzungsberichte  | `usage.cost`, `sessions.usage`                       |
| Status der Modell-Anmeldedaten | `models.authStatus`                                  |
| Konfiguration                 | `config.get`, `config.patch`                         |

`config.get` schwärzt sensible Werte und SecretRef-Bezeichner, bevor der
Snapshot zurückgegeben wird. Schreibmethoden geben ebenfalls eine geschwärzte Konfiguration zurück. Ein Client muss den
Schwärzungsplatzhalter als undurchsichtig behandeln und den dokumentierten Vertrag zum Schreiben der Konfiguration verwenden; er
darf niemals erwarten, dass das Gateway Geheimnisse im Klartext zurückgibt.

Lesen oder verändern Sie keine Dateien, SQLite-Tabellen, Transkriptdateien oder Cache-Verzeichnisse
unter `~/.openclaw`, um App-Funktionen zu implementieren. Diese Strukturen sind private
Implementierungsdetails der Laufzeit und können ohne Protokollkompatibilität verschoben oder geändert werden.

## Installieren, nicht vereinfachen

Das Stammpaket `openclaw` ist kein Ziel für das Vendoring als einzelne Datei. Gebündelte Laufzeitdateien
unter `dist/extensions` behalten unqualifizierte Selbstimporte wie
`openclaw/plugin-sdk/*` bei, während das npm-Paket absichtlich
erweiterungsspezifische `node_modules`-Verzeichnisbäume ausschließt.

Installieren Sie OpenClaw über npm, pnpm oder eine andere normale Node-Paketinstallation, damit
Node die Paketexporte und den Abhängigkeitsbaum des Stammpakets auflösen kann. Starten Sie die installierte
ausführbare Datei `openclaw`. Kopieren Sie nicht nur `dist`, vereinfachen Sie das Paket nicht zu einem App-
Bundle und übernehmen Sie keine ausgewählten Erweiterungsdateien in das Projekt.

## Verwandte Themen

- [Erstellen eines Gateway-Clients](https://docs.openclaw.ai/gateway/clients)
- [Gateway-Protokoll](https://docs.openclaw.ai/gateway/protocol)
- [Gateway-CLI](https://docs.openclaw.ai/cli/gateway)
- [Gateway-Integrationen für externe Apps](https://docs.openclaw.ai/gateway/external-apps)
