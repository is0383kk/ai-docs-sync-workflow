---
read_when:
    - Sie möchten, dass ein OpenClaw-Agent an einem Google-Meet-Anruf teilnimmt
    - Sie möchten, dass ein OpenClaw-Agent einen neuen Google-Meet-Anruf erstellt
    - Sie konfigurieren Chrome, den Chrome-Node oder Twilio als Google-Meet-Transport.
summary: 'Google-Meet-Plugin: Über Chrome oder Twilio expliziten Meet-URLs beitreten, mit Standardeinstellungen für Agent-Antworten'
title: Google-Meet-Plugin
x-i18n:
    generated_at: "2026-07-26T17:54:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8a611e283fe900984a29b563969936a641c7af430b05933eb03b98dc93b5d0c8
    source_path: plugins/google-meet.md
    workflow: 16
---

Das Plugin `google-meet` tritt im Namen eines OpenClaw-Agenten expliziten Meet-URLs bei. Sein Funktionsumfang ist bewusst eng gefasst:

- Es tritt ausschließlich `https://meet.google.com/...`-URLs bei; es wählt sich niemals über eine selbst gefundene Telefonnummer in eine Besprechung ein.
- `googlemeet create` kann über die Google Meet API (oder einen Browser-Fallback) eine neue Meet-URL erstellen und ihr standardmäßig beitreten.
- Die Teilnahme über Chrome verwendet ein angemeldetes Chrome-Profil, optional auf einem gekoppelten Node. Bei der Teilnahme über Twilio wird über das [Plugin für Sprachanrufe](/de/plugins/voice-call) eine Telefonnummer samt PIN/DTMF gewählt; eine Meet-URL kann damit nicht direkt angewählt werden.
- `mode: "agent"` (Standard) transkribiert die Sprache der Teilnehmenden mit einem Echtzeit-Provider, leitet sie an den konfigurierten OpenClaw-Agenten weiter und gibt die Antwort mit der regulären OpenClaw-Sprachausgabe wieder. Mit `mode: "bidi"` antwortet ein Echtzeit-Sprachmodell direkt. `mode: "transcribe"` tritt nur zur Beobachtung ohne Sprachantwort bei.
- Beim Beitritt des Plugins zu einem Anruf erfolgt keine automatische Einwilligungsansage.
- Der CLI-Befehl lautet `googlemeet`; `meet` ist umfassenderen Workflows für Agententelefonkonferenzen vorbehalten.

## Schnellstart

Installieren Sie das Plugin und die lokalen Audioabhängigkeiten und legen Sie anschließend einen Schlüssel für einen Echtzeit-Provider fest. OpenAI ist der standardmäßige Transkriptions-Provider für den Modus `agent`; Google Gemini Live ist als Sprach-Provider für den Modus `bidi` verfügbar:

```bash
openclaw plugins install npm:@openclaw/google-meet
brew install blackhole-2ch sox
export OPENAI_API_KEY=sk-...
# nur erforderlich, wenn realtime.voiceProvider für den bidi-Modus auf "google" gesetzt ist
export GEMINI_API_KEY=...
```

`blackhole-2ch` installiert das virtuelle Audiogerät `BlackHole 2ch`, über das Chrome das Audio leitet. Das Homebrew-Installationsprogramm erfordert einen Neustart, bevor macOS das Gerät bereitstellt:

```bash
sudo reboot
```

Überprüfen Sie nach dem Neustart beide Komponenten:

```bash
system_profiler SPAudioDataType | grep -i BlackHole
command -v sox
```

Das Plugin ist nach der Installation standardmäßig aktiviert. Fügen Sie nur dann einen Eintrag hinzu, wenn Sie es anpassen möchten:

```json5
{
  plugins: {
    entries: {
      "google-meet": {
        config: {},
      },
    },
  },
}
```

Führen Sie `openclaw plugins disable google-meet` aus, wenn das Plugin nicht aktiv sein soll.

Überprüfen Sie die Einrichtung und treten Sie anschließend bei:

```bash
openclaw googlemeet setup
openclaw googlemeet join https://meet.google.com/abc-defg-hij
```

Die Ausgabe von `setup` ist für Agenten lesbar und berücksichtigt Modus und Transport: Sie meldet das Chrome-Profil, die Bindung an einen Node und bei Echtzeitbeitritten über Chrome die BlackHole-/SoX-Audiobrücke sowie die Prüfung der verzögerten Einführung. Bei Beitritten nur zur Beobachtung werden die Echtzeitvoraussetzungen übersprungen:

```bash
openclaw googlemeet setup --transport chrome-node --mode transcribe
```

Wenn die Twilio-Delegierung konfiguriert ist, meldet `setup` außerdem, ob `voice-call`, die Twilio-Anmeldedaten und die öffentliche Webhook-Erreichbarkeit bereit sind. Behandeln Sie jede Prüfung mit dem Ergebnis `ok: false` für den jeweiligen Transport und Modus als Blocker, bevor ein Agent beitritt. Verwenden Sie `--json` für eine maschinenlesbare Ausgabe und `--transport chrome|chrome-node|twilio`, um einen bestimmten Transport vorab zu prüfen:

```bash
openclaw googlemeet setup --transport twilio
```

Alternativ kann ein Agent über das Tool `google_meet` beitreten:

```json
{
  "action": "join",
  "url": "https://meet.google.com/abc-defg-hij",
  "transport": "chrome-node",
  "mode": "agent"
}
```

Auf Gateway-Hosts ohne macOS bleibt `google_meet` für Artefakt-, Kalender-, Einrichtungs-, Transkriptions-, Twilio- und `chrome-node`-Aktionen sichtbar. Die lokale Sprachantwort über Chrome (`transport: "chrome"` mit `mode: "agent"` oder `"bidi"`) wird jedoch blockiert, bevor sie die Audiobrücke erreicht, da dieser Pfad derzeit von `BlackHole 2ch` unter macOS abhängt. Verwenden Sie stattdessen `mode: "transcribe"`, die Twilio-Einwahl oder einen macOS-Host `chrome-node`.

### Eine Besprechung erstellen

```bash
openclaw googlemeet create --transport chrome-node --mode agent
openclaw googlemeet create --no-join
```

`create` verfügt über zwei Pfade, die im Feld `source` des Ergebnisses angegeben werden:

- **`api`**: Wird verwendet, wenn OAuth-Anmeldedaten für Google Meet konfiguriert sind. Deterministisch; unabhängig vom Zustand der Browseroberfläche.
- **`browser`**: Wird ohne OAuth-Anmeldedaten verwendet. OpenClaw öffnet `https://meet.google.com/new` auf dem festgelegten Chrome-Node und wartet darauf, dass Google zu einer echten URL mit Besprechungscode weiterleitet; das OpenClaw-Chrome-Profil auf diesem Node muss bereits bei Google angemeldet sein. Sowohl beim Beitreten als auch beim Erstellen wird zunächst ein vorhandener Meet-Tab (oder ein laufender `.../new`- bzw. Google-Konto-Aufforderungs-Tab) wiederverwendet, bevor ein neuer geöffnet wird. Beim Tab-Abgleich werden unproblematische Abfragezeichenfolgen wie `authuser` ignoriert.

`create` tritt standardmäßig bei und gibt `joined: true` sowie die Beitrittssitzung zurück. Übergeben Sie `--no-join` (CLI) oder `"join": false` (Tool), um nur die URL zu erstellen.

Legen Sie für über die API erstellte Räume eine explizite Zugriffsrichtlinie fest, anstatt den Standardwert des Google-Kontos zu übernehmen:

```bash
openclaw googlemeet create --access-type OPEN --transport chrome-node --mode agent
```

| `--access-type` | Wer ohne Anklopfen beitreten kann                                   |
| --------------- | ------------------------------------------------------------------- |
| `OPEN`          | Alle Personen mit der Meet-URL                                      |
| `TRUSTED`       | Vertrauenswürdige Nutzer der Hostorganisation, eingeladene externe Nutzer und Einwahlnutzer |
| `RESTRICTED`    | Nur eingeladene Personen                                            |

Dies gilt nur für über die API erstellte Räume, daher muss OAuth konfiguriert sein. Wenn Sie sich authentifiziert haben, bevor diese Option verfügbar war, führen Sie `openclaw googlemeet auth login --json` erneut aus, nachdem Sie den Scope `meetings.space.settings` zu Ihrem OAuth-Zustimmungsbildschirm hinzugefügt haben.

Wenn der Browser-Fallback durch eine Google-Anmeldung oder eine Meet-Berechtigungsanforderung blockiert wird, gibt das Tool `manualActionRequired: true` mit `manualActionReason`, `manualActionMessage` und `browser.nodeId`/`browser.targetId`/`browserUrl` zurück. Melden Sie diese Nachricht und öffnen Sie keine weiteren Meet-Tabs, bis die bedienende Person den Schritt im Browser abgeschlossen hat.

### Beitritt nur zur Beobachtung

Setzen Sie `"mode": "transcribe"`, um die bidirektionale Echtzeitbrücke zu überspringen (keine BlackHole-/SoX-Anforderung, keine Sprachantwort). Chrome-Beitritte im Transkriptionsmodus überspringen außerdem die Erteilung der OpenClaw-Mikrofon-/Kameraberechtigung sowie den Meet-Pfad **Use microphone**. Wenn Meet den Zwischendialog zur Audioauswahl anzeigt, versucht die Automatisierung zuerst **Continue without microphone**. Verwaltete Chrome-Transporte installieren in jedem Modus nach Möglichkeit einen Meet-Untertitelbeobachter, damit dauerhafte Notizen verfügbar sind, ohne den Live-Konsultationspfad des Agenten zu ändern. `googlemeet status --json` und `googlemeet doctor` melden `captioning`, `captionsEnabledAttempted`, `transcriptLines`, `lastCaptionAt`, `lastCaptionSpeaker`, `lastCaptionText` sowie einen `recentTranscript`-Nachlauf.

Lesen Sie für das begrenzte Sitzungstranskript den exakt verfolgten Meet-Tab aus:

```bash
openclaw googlemeet transcript <session-id>
openclaw googlemeet transcript <session-id> --since <next-index> --json
```

Der Beobachter bewahrt höchstens 2.000 abgeschlossene Untertitelzeilen auf der Meet-Seite auf. Sichtbarer, fortschreitender Text verbleibt im Status-Nachlauf zur Systemintegrität, bis die Untertitelzeile abgeschlossen ist. Daher kann das Speichern von `nextIndex` eine spätere Texterweiterung nicht überspringen; beim Verlassen werden sichtbare Zeilen vor dem Snapshot abgeschlossen. `droppedLines` meldet die am Anfang verlorenen Zeilen, wenn die Obergrenze überschritten wird. Der begrenzte `googlemeet transcript`-Nachlauf bewahrt weiterhin nur die vier zuletzt beendeten Sitzungen auf und wird mit dem Gateway zurückgesetzt. Unabhängig davon hängt OpenClaw während der gesamten Besprechung abgeschlossene Untertitelzeilen an die gemeinsame Zustandsdatenbank an und schreibt beim Verlassen eine abgeleitete Zusammenfassung. Verwenden Sie [`openclaw transcripts`](/de/cli/transcripts), um diese dauerhaften Notizen anzuzeigen oder zu exportieren.

Automatische Notizen sind standardmäßig aktiviert. Setzen Sie `transcripts.enabled: false`, um
dauerhafte Notizen global zu deaktivieren; der explizite Modus `transcribe` stellt weiterhin nur
seinen begrenzten Live-Nachlauf bereit. Twilio-Beitritte verfügen nicht über den Untertitelstream des Browsers und
werden über diesen Pfad nicht erfasst.

Für eine Ja/Nein-Abhörprüfung:

```bash
openclaw googlemeet test-listen <meet-url> --transport chrome-node
```

Der Befehl tritt im Transkriptionsmodus bei, wartet auf neue Bewegungen bei Untertiteln oder Transkripten und gibt `listenVerified`, `listenTimedOut`, Felder für manuelle Aktionen sowie den aktuellen Untertitelzustand zurück.

### Zustand der Echtzeitsitzung

Während Sitzungen mit Sprachantwort meldet der Status `google_meet` den Zustand von Chrome und der Audiobrücke: `inCall`, `manualActionRequired`, `providerConnected`, `realtimeReady`, `audioInputActive`, `audioOutputActive`, Zeitstempel der letzten Ein- und Ausgabe, Bytezähler sowie den Zustand der geschlossenen Brücke. Verwaltete Chrome-Sitzungen geben die Einführungs-/Testphrase erst aus, nachdem der Zustand `inCall: true` gemeldet wurde; andernfalls wird `speechReady: false` ausgegeben und der Sprechversuch blockiert, statt ohne Hinweis wirkungslos zu bleiben.

Lokale Chrome-Beitritte erfolgen über das angemeldete OpenClaw-Browserprofil und benötigen `BlackHole 2ch` für den Mikrofon-/Lautsprecherpfad. Ein einzelnes BlackHole-Gerät genügt für einen ersten Funktionstest, kann jedoch ein Echo erzeugen. Verwenden Sie separate virtuelle Geräte oder eine Loopback-ähnliche Verschaltung für sauberes bidirektionales Audio.

## Lokales Gateway und Chrome unter Parallels

Innerhalb einer macOS-VM sind weder ein vollständiges Gateway noch ein Modell-API-Schlüssel erforderlich, wenn sie lediglich Chrome bereitstellen soll. Führen Sie das Gateway und den Agenten lokal und einen Node-Host in der VM aus.

| Ausführungsort        | Komponenten                                                                                     |
| -------------------- | ----------------------------------------------------------------------------------------------- |
| Gateway-Host         | OpenClaw Gateway, Agenten-Workspace, Modell-/API-Schlüssel, Echtzeit-Provider, Konfiguration des Google-Meet-Plugins |
| Parallels-macOS-VM   | OpenClaw CLI/Node-Host, Chrome, SoX, BlackHole 2ch, bei Google angemeldetes Chrome-Profil       |
| In der VM nicht erforderlich | Gateway-Dienst, Agentenkonfiguration, Einrichtung des Modell-Providers                    |

Installieren Sie die VM-Abhängigkeiten, starten Sie neu und überprüfen Sie die Installation:

```bash
brew install blackhole-2ch sox
sudo reboot
system_profiler SPAudioDataType | grep -i BlackHole
command -v sox
```

Installieren Sie das Plugin in der VM, wo es standardmäßig aktiviert ist, und starten Sie den Node-Host:

```bash
openclaw plugins install npm:@openclaw/google-meet
openclaw node run --host <gateway-host> --port 18789 --display-name parallels-macos
```

Wenn `<gateway-host>` eine LAN-IP-Adresse ohne TLS ist, lassen Sie dies für das vertrauenswürdige private Netzwerk ausdrücklich zu:

```bash
OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1 \
  openclaw node run --host <gateway-lan-ip> --port 18789 --display-name parallels-macos
```

Verwenden Sie beim Installieren als LaunchAgent dasselbe Flag. Es handelt sich um eine Prozessumgebungsvariable, die in der LaunchAgent-Umgebung gespeichert wird, wenn sie beim Installationsbefehl vorhanden ist, und nicht um eine `openclaw.json`-Einstellung:

```bash
OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1 \
  openclaw node install --host <gateway-lan-ip> --port 18789 --display-name parallels-macos --force
openclaw node restart
```

Genehmigen Sie den Node auf dem Gateway-Host und bestätigen Sie anschließend, dass er sowohl `googlemeet.chrome` als auch die Browserfunktion/`browser.proxy` ankündigt:

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw nodes status
```

Leiten Sie Meet über diesen Node:

```json5
{
  gateway: {
    nodes: {
      commands: { allow: ["googlemeet.chrome", "browser.proxy"] },
    },
  },
  plugins: {
    entries: {
      "google-meet": {
        enabled: true,
        config: {
          defaultTransport: "chrome-node",
          chrome: {
            guestName: "OpenClaw Agent",
            autoJoin: true,
            reuseExistingTab: true,
          },
          chromeNode: {
            node: "parallels-macos",
          },
        },
      },
    },
  },
}
```

Treten Sie nun wie gewohnt vom Gateway-Host aus bei:

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij
```

Für einen Funktionstest mit einem einzigen Befehl, der eine Sitzung erstellt oder wiederverwendet, eine bekannte Phrase ausgibt und den Sitzungszustand anzeigt:

```bash
openclaw googlemeet test-speech https://meet.google.com/abc-defg-hij
```

Während des Echtzeit-Beitritts trägt die Browserautomatisierung den Gastnamen ein, klickt auf Join/Ask to join und bestätigt die beim ersten Start angezeigte Meet-Aufforderung „Use microphone“ (oder „Continue without microphone“ beim Beitritt im reinen Beobachtungsmodus und bei der rein browserbasierten Besprechungserstellung). Wenn das Profil abgemeldet ist, Meet auf die Zulassung durch den Host wartet, Chrome eine Mikrofon-/Kameraberechtigung benötigt oder Meet bei einer nicht beantworteten Aufforderung hängen bleibt, meldet das Ergebnis `manualActionRequired: true` mit `manualActionReason` und `manualActionMessage`. Beenden Sie die Wiederholungsversuche, melden Sie diese Nachricht zusammen mit `browserUrl`/`browserTitle` und versuchen Sie es erst erneut, nachdem die manuelle Aktion abgeschlossen wurde.

Wenn `chromeNode.node` ausgelassen wird, trifft OpenClaw nur dann automatisch eine Auswahl, wenn genau ein verbundener Node sowohl `googlemeet.chrome` als auch Browsersteuerung anbietet. Legen Sie `chromeNode.node` (Node-ID, Anzeigename oder Remote-IP) fest, wenn mehrere geeignete Nodes verbunden sind.

### Häufige Fehlerprüfungen

| Symptom                                                  | Behebung                                                                                                                                                                                                                                                                              |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Configured Google Meet node ... is not usable: offline` | Der festgelegte Node ist bekannt, aber nicht verfügbar. Melden Sie den Einrichtungsblocker; weichen Sie nicht stillschweigend auf einen anderen Transport aus, sofern dies nicht angefordert wurde.                                                                                     |
| `No connected Google Meet-capable node`                  | Installieren Sie `npm:@openclaw/google-meet` in der VM, führen Sie `openclaw plugins enable browser` aus, starten Sie `openclaw node run` und genehmigen Sie die Kopplung. Wenn Google Meet ausdrücklich deaktiviert wurde, aktivieren Sie es ebenfalls. Vergewissern Sie sich, dass `gateway.nodes.commands.allow` `googlemeet.chrome` und `browser.proxy` enthält. |
| `BlackHole 2ch audio device not found`                   | Installieren Sie `blackhole-2ch` auf dem zu prüfenden Host und starten Sie ihn neu.                                                                                                                                                                                                  |
| `BlackHole 2ch audio device not found on the node`       | Installieren Sie `blackhole-2ch` in der VM und starten Sie die VM neu.                                                                                                                                                                                                               |
| Chrome wird geöffnet, kann aber nicht beitreten          | Melden Sie sich im Browserprofil der VM an oder lassen Sie `chrome.guestName` gesetzt. Der automatische Beitritt als Gast verwendet die OpenClaw-Browserautomatisierung über den Node-Browserproxy. Richten Sie `browser.defaultProfile` des Nodes (oder ein benanntes Profil einer bestehenden Sitzung) auf das gewünschte Profil aus. |
| Doppelte Meet-Tabs                                       | Lassen Sie `chrome.reuseExistingTab: true`. OpenClaw aktiviert einen vorhandenen Tab für dieselbe URL, und die Erstellung verwendet einen laufenden `.../new`- oder Google-Konto-Aufforderungstab erneut, bevor ein weiterer geöffnet wird.                                                   |
| Kein Ton                                                 | Leiten Sie Meet-Mikrofon und -Lautsprecher über den von OpenClaw verwendeten virtuellen Audiopfad. Verwenden Sie für sauberes Duplex-Audio getrennte virtuelle Geräte oder ein Routing nach Art von Loopback.                                                                           |

## Installationshinweise

Die standardmäßige Chrome-Antwortfunktion verwendet zwei externe Werkzeuge, die OpenClaw weder bündelt noch weiterverteilt. Installieren Sie sie über Homebrew als Host-Abhängigkeiten:

- `sox`: Befehlszeilen-Audiowerkzeug. Das Plugin führt für die standardmäßige 24-kHz-PCM16-Audiobrücke explizite CoreAudio-Gerätebefehle aus.
- `blackhole-2ch`: Virtueller macOS-Audiotreiber, der das Gerät `BlackHole 2ch` bereitstellt, über das Chrome/Meet geleitet wird.

SoX ist unter `LGPL-2.0-only AND GPL-2.0-only` lizenziert; BlackHole unter GPL-3.0. Wenn Sie ein Installationsprogramm oder eine Appliance erstellen, das bzw. die BlackHole mit OpenClaw bündelt, prüfen Sie die Lizenzierung des BlackHole-Upstream-Projekts oder erwerben Sie eine separate Lizenz von Existential Audio.

## Transporte

| Transport     | Verwenden, wenn                                                                             |
| ------------- | -------------------------------------------------------------------------------------------- |
| `chrome`      | Chrome/Audio auf dem Gateway-Host ausgeführt werden                                          |
| `chrome-node` | Chrome/Audio auf einem gekoppelten Node ausgeführt werden (beispielsweise einer Parallels-macOS-VM) |
| `twilio`      | Telefonische Einwahl als Ausweichlösung über das Voice-Call-Plugin, wenn eine Teilnahme über Chrome nicht verfügbar ist |

### Chrome

Öffnet die Meet-URL über die OpenClaw-Browsersteuerung und tritt mit dem angemeldeten OpenClaw-Browserprofil bei. Unter macOS prüft das Plugin vor dem Start auf `BlackHole 2ch` und führt, sofern konfiguriert, einen Befehl zur Zustandsprüfung/zum Start der Audiobrücke aus, bevor Chrome geöffnet wird. Wählen Sie für lokales Chrome das Profil mit `browser.defaultProfile`; `chrome.browserProfile` wird stattdessen an `chrome-node`-Hosts übergeben.

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij --transport chrome
openclaw googlemeet join https://meet.google.com/abc-defg-hij --transport chrome-node
```

Mikrofon-/Lautsprecheraudio von Chrome wird über die lokale OpenClaw-Audiobrücke geleitet. Wenn `BlackHole 2ch` nicht installiert ist, schlägt der Beitritt mit einem Einrichtungsfehler fehl, statt ohne Audiopfad beizutreten.

### Twilio

Ein strikter, an das [Voice-Call-Plugin](/de/plugins/voice-call) delegierter Wählplan. Meet-Seiten werden nicht nach Telefonnummern durchsucht; Google Meet muss für die Besprechung eine telefonische Einwahlnummer und PIN bereitstellen.

Aktivieren Sie Voice Call auf dem Gateway-Host, nicht auf dem Chrome-Node:

```json5
{
  plugins: {
    allow: ["google-meet", "voice-call", "google"],
    entries: {
      "google-meet": {
        enabled: true,
        config: {
          defaultTransport: "chrome-node",
          // oder auf "twilio" setzen, wenn Twilio der Standard sein soll
        },
      },
      "voice-call": {
        enabled: true,
        config: {
          provider: "twilio",
          inboundPolicy: "allowlist",
          realtime: {
            enabled: true,
            provider: "google",
            instructions: "Treten Sie diesem Google Meet als OpenClaw-Agent bei. Fassen Sie sich kurz.",
            toolPolicy: "safe-read-only",
            providers: {
              google: {
                silenceDurationMs: 500,
                startSensitivity: "high",
              },
            },
          },
        },
      },
      google: {
        enabled: true,
      },
    },
  },
}
```

Stellen Sie die Twilio-Anmeldedaten über die Umgebung bereit, damit Geheimnisse nicht in `openclaw.json` gespeichert werden:

```bash
export TWILIO_ACCOUNT_SID=AC...
export TWILIO_AUTH_TOKEN=...
export TWILIO_FROM_NUMBER=+15550001234
export GEMINI_API_KEY=...
```

Verwenden Sie stattdessen `realtime.provider: "openai"` mit `OPENAI_API_KEY`, wenn OpenAI der Echtzeit-Sprach-Provider ist.

Starten oder laden Sie das Gateway nach der Aktivierung von `voice-call` neu; Änderungen an der Plugin-Konfiguration werden erst nach dem Neuladen wirksam. Überprüfung:

```bash
openclaw config validate
openclaw plugins list | grep -E 'google-meet|voice-call'
openclaw googlemeet setup
```

Wenn die Twilio-Delegierung eingerichtet ist, enthält `googlemeet setup` Prüfungen für `twilio-voice-call-plugin`, `twilio-voice-call-credentials` und `twilio-voice-call-webhook`.

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij \
  --transport twilio \
  --dial-in-number +15551234567 \
  --pin 123456
```

Verwenden Sie `--dtmf-sequence` für eine benutzerdefinierte Sequenz, mit vorangestelltem `w` oder Kommas für eine Pause vor der PIN:

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij \
  --transport twilio \
  --dial-in-number +15551234567 \
  --dtmf-sequence ww123456#
```

## OAuth und Vorabprüfung

OAuth ist für die Erstellung eines Meet-Links optional, da `googlemeet create` auf Browserautomatisierung ausweichen kann. Konfigurieren Sie OAuth für die Erstellung über die offizielle API, die Space-Auflösung oder die Vorabprüfung der Meet Media API. Beitritte über Chrome/Chrome-Node hängen nie von OAuth ab; sie verwenden in jedem Fall ein angemeldetes Chrome-Profil, BlackHole/SoX und (für `chrome-node`) einen verbundenen Node.

### Google-Anmeldedaten erstellen

In der Google Cloud Console:

<Steps>
<Step title="Projekt erstellen oder auswählen">
</Step>
<Step title="Google Meet REST API aktivieren">
</Step>
<Step title="OAuth-Zustimmungsbildschirm konfigurieren">
Internal ist für eine Google-Workspace-Organisation am einfachsten. External eignet sich für persönliche/Testeinrichtungen. Solange sich die App in Testing befindet, fügen Sie jedes Google-Konto, das sie autorisieren soll, als Testnutzer hinzu.
</Step>
<Step title="Angeforderte Bereiche hinzufügen">
- `https://www.googleapis.com/auth/meetings.space.created`
- `https://www.googleapis.com/auth/meetings.space.readonly`
- `https://www.googleapis.com/auth/meetings.space.settings`
- `https://www.googleapis.com/auth/meetings.conference.media.readonly`
- `https://www.googleapis.com/auth/calendar.events.readonly` (Kalendersuche)
- `https://www.googleapis.com/auth/drive.meet.readonly` (Export des Dokumentinhalts von Transkripten/intelligenten Notizen)

</Step>
<Step title="OAuth-Client-ID erstellen">
Anwendungstyp **Web application**. Autorisierte Weiterleitungs-URI:

```text
http://localhost:8085/oauth2callback
```

</Step>
<Step title="Client-ID und Client-Secret kopieren">
</Step>
</Steps>

`meetings.space.created` wird von `spaces.create` benötigt. `meetings.space.readonly` löst Meet-URLs/-Codes in Spaces auf. Mit `meetings.space.settings` kann OpenClaw bei der API-Raumerstellung `SpaceConfig`-Einstellungen wie `accessType` übergeben. `meetings.conference.media.readonly` ist für die Vorabprüfung und Medienfunktionen der Meet Media API vorgesehen; Google kann für die tatsächliche Nutzung der Media API eine Anmeldung zum Developer Preview verlangen. `calendar.events.readonly` wird nur für die Kalendersuche mit `--today`/`--event` benötigt. `drive.meet.readonly` wird nur für den Export mit `--include-doc-bodies` benötigt. Wenn Sie ausschließlich browserbasierte Chrome-Beitritte benötigen, überspringen Sie OAuth vollständig.

### Aktualisierungstoken erzeugen

Konfigurieren Sie `oauth.clientId` und optional `oauth.clientSecret` (oder übergeben Sie sie als Umgebungsvariablen) und führen Sie anschließend Folgendes aus:

```bash
openclaw googlemeet auth login --json
```

Dadurch wird ein PKCE-Ablauf mit einem Localhost-Callback auf `http://localhost:8085/oauth2callback` ausgeführt und ein `oauth`-Konfigurationsblock mit einem Aktualisierungstoken ausgegeben. Fügen Sie `--manual` für einen Kopieren-und-Einfügen-Ablauf hinzu, wenn der Browser den lokalen Callback nicht erreichen kann:

```bash
OPENCLAW_GOOGLE_MEET_CLIENT_ID="your-client-id" \
OPENCLAW_GOOGLE_MEET_CLIENT_SECRET="your-client-secret" \
openclaw googlemeet auth login --json --manual
```

JSON-Ausgabe:

```json
{
  "oauth": {
    "clientId": "your-client-id",
    "clientSecret": "your-client-secret",
    "refreshToken": "refresh-token",
    "accessToken": "access-token",
    "expiresAt": 1770000000000
  },
  "scope": "..."
}
```

Speichern Sie das `oauth`-Objekt unter der Plugin-Konfiguration:

```json5
{
  plugins: {
    entries: {
      "google-meet": {
        enabled: true,
        config: {
          oauth: {
            clientId: "your-client-id",
            clientSecret: "your-client-secret",
            refreshToken: "refresh-token",
          },
        },
      },
    },
  },
}
```

Bevorzugen Sie Umgebungsvariablen, wenn das Aktualisierungstoken nicht in der Konfiguration gespeichert werden soll; zuerst wird die Konfiguration aufgelöst, danach dient die Umgebung als Ausweichlösung. Wenn Sie sich authentifiziert haben, bevor Unterstützung für Besprechungserstellung, Kalendersuche oder den Export von Dokumentinhalten verfügbar war, führen Sie `openclaw googlemeet auth login --json` erneut aus, damit das Aktualisierungstoken den aktuellen Bereichssatz abdeckt.

### OAuth mit doctor überprüfen

```bash
openclaw googlemeet doctor --oauth --json
```

Dies prüft, ob die OAuth-Konfiguration vorhanden ist und das Aktualisierungstoken ein Zugriffstoken ausstellen kann, ohne die Chrome-Laufzeit zu laden oder eine verbundene Node zu benötigen. Der Bericht enthält ausschließlich Statusfelder (`ok`, `configured`, `tokenSource`, `expiresAt`, Prüfmeldungen) und gibt niemals das Zugriffstoken, das Aktualisierungstoken oder das Client-Geheimnis aus.

| Prüfung              | Bedeutung                                                                        |
| -------------------- | -------------------------------------------------------------------------------- |
| `oauth-config`       | `oauth.clientId` plus `oauth.refreshToken` oder ein zwischengespeichertes Zugriffstoken ist vorhanden |
| `oauth-token`        | Das zwischengespeicherte Zugriffstoken ist noch gültig oder das Aktualisierungstoken hat ein neues ausgestellt |
| `meet-spaces-get`    | Die optionale Prüfung `--meeting` hat einen vorhandenen Meet-Raum aufgelöst |
| `meet-spaces-create` | Die optionale Prüfung `--create-space` hat einen neuen Meet-Raum erstellt       |

Weisen Sie die Aktivierung der Meet API und den Geltungsbereich `spaces.create` mit der zustandsverändernden Erstellungsprüfung nach:

```bash
openclaw googlemeet doctor --oauth --create-space --json
```

Weisen Sie den Lesezugriff auf einen vorhandenen Raum nach:

```bash
openclaw googlemeet doctor --oauth --meeting https://meet.google.com/abc-defg-hij --json
openclaw googlemeet resolve-space --meeting https://meet.google.com/abc-defg-hij
```

Ein `403` bei diesen Prüfungen bedeutet normalerweise, dass die Meet REST API deaktiviert ist, dem Aktualisierungstoken der erforderliche Geltungsbereich fehlt oder das Google-Konto nicht auf diesen Raum zugreifen kann. Ein Aktualisierungstokenfehler bedeutet, dass `openclaw googlemeet auth login --json` erneut ausgeführt und der neue Block `oauth` gespeichert werden muss.

Für den Browser-Fallback ist kein OAuth erforderlich; die Google-Authentifizierung stammt dort aus dem angemeldeten Chrome-Profil auf der ausgewählten Node und nicht aus der OpenClaw-Konfiguration.

Diese Umgebungsvariablen werden als Fallbacks akzeptiert:

- `OPENCLAW_GOOGLE_MEET_CLIENT_ID` oder `GOOGLE_MEET_CLIENT_ID`
- `OPENCLAW_GOOGLE_MEET_CLIENT_SECRET` oder `GOOGLE_MEET_CLIENT_SECRET`
- `OPENCLAW_GOOGLE_MEET_REFRESH_TOKEN` oder `GOOGLE_MEET_REFRESH_TOKEN`
- `OPENCLAW_GOOGLE_MEET_ACCESS_TOKEN` oder `GOOGLE_MEET_ACCESS_TOKEN`
- `OPENCLAW_GOOGLE_MEET_ACCESS_TOKEN_EXPIRES_AT` oder `GOOGLE_MEET_ACCESS_TOKEN_EXPIRES_AT`
- `OPENCLAW_GOOGLE_MEET_DEFAULT_MEETING` oder `GOOGLE_MEET_DEFAULT_MEETING`
- `OPENCLAW_GOOGLE_MEET_PREVIEW_ACK` oder `GOOGLE_MEET_PREVIEW_ACK`

### Artefakte auflösen, vorab prüfen und lesen

```bash
openclaw googlemeet resolve-space --meeting https://meet.google.com/abc-defg-hij
openclaw googlemeet preflight --meeting https://meet.google.com/abc-defg-hij
```

Nachdem Meet Konferenzdatensätze erstellt hat:

```bash
openclaw googlemeet artifacts --meeting https://meet.google.com/abc-defg-hij
openclaw googlemeet attendance --meeting https://meet.google.com/abc-defg-hij
openclaw googlemeet export --meeting https://meet.google.com/abc-defg-hij --output ./meet-export
```

Mit `--meeting` verwenden `artifacts` und `attendance` standardmäßig den neuesten Konferenzdatensatz; übergeben Sie `--all-conference-records` für jeden aufbewahrten Datensatz.

Die Kalendersuche löst die Besprechungs-URL aus Google Calendar auf, bevor Artefakte gelesen werden (erfordert ein Aktualisierungstoken, das den schreibgeschützten Geltungsbereich für Kalenderereignisse enthält):

```bash
openclaw googlemeet latest --today
openclaw googlemeet calendar-events --today --json
openclaw googlemeet artifacts --event "Weekly sync"
openclaw googlemeet attendance --today --format csv --output attendance.csv
```

`--today` durchsucht den heutigen Kalender `primary` nach einem Ereignis mit einem Meet-Link; `--event <query>` durchsucht übereinstimmenden Ereignistext; `--calendar <id>` richtet sich an einen nicht primären Kalender. `calendar-events` zeigt übereinstimmende Ereignisse in der Vorschau an und kennzeichnet, welches davon `latest`/`artifacts`/`attendance`/`export` auswählt.

Wenn Sie die ID des Konferenzdatensatzes bereits kennen, sprechen Sie ihn direkt an:

```bash
openclaw googlemeet latest --meeting https://meet.google.com/abc-defg-hij
openclaw googlemeet artifacts --conference-record conferenceRecords/abc123 --json
openclaw googlemeet attendance --conference-record conferenceRecords/abc123 --json
```

Schließen Sie den Raum für einen über die API erstellten Raum:

```bash
openclaw googlemeet end-active-conference https://meet.google.com/abc-defg-hij
```

Ruft `spaces.endActiveConference` auf und erfordert OAuth mit dem Geltungsbereich `meetings.space.created` für einen Raum, den das autorisierte Konto verwalten kann. Akzeptiert eine Meet-URL, einen Besprechungscode oder `spaces/{id}` und löst die Angabe zunächst in die API-Raumressource auf. Dies ist von `googlemeet leave` getrennt: `leave` beendet die lokale/Sitzungsteilnahme von OpenClaw; `end-active-conference` fordert Google Meet auf, die aktive Konferenz für den Raum zu beenden.

Schreiben Sie einen lesbaren Bericht:

```bash
openclaw googlemeet artifacts --conference-record conferenceRecords/abc123 \
  --format markdown --output meet-artifacts.md
openclaw googlemeet attendance --conference-record conferenceRecords/abc123 \
  --format csv --output meet-attendance.csv
openclaw googlemeet export --conference-record conferenceRecords/abc123 \
  --include-doc-bodies --zip --output meet-export
openclaw googlemeet export --conference-record conferenceRecords/abc123 \
  --include-doc-bodies --dry-run
```

`artifacts` gibt Metadaten des Konferenzdatensatzes sowie Ressourcenmetadaten zu Teilnehmern, Aufzeichnungen, Transkripten, strukturierten Transkripteinträgen und intelligenten Notizen zurück, wenn Google diese bereitstellt. `--no-transcript-entries` überspringt die Eintragssuche bei großen Besprechungen. `attendance` erweitert Teilnehmer zu Teilnehmer-Sitzungszeilen mit Zeitpunkten der ersten/letzten Anwesenheit, gesamter Sitzungsdauer, Kennzeichnungen für Verspätung/vorzeitiges Verlassen sowie doppelten Teilnehmerressourcen, die anhand des angemeldeten Benutzers oder Anzeigenamens zusammengeführt werden; `--no-merge-duplicates` hält Rohressourcen getrennt, `--late-after-minutes`/`--early-before-minutes` passen die Schwellenwerte an.

`export` schreibt einen Ordner mit `summary.md`, `attendance.csv`, `transcript.md`, `artifacts.json`, `attendance.json` und `manifest.json`. `manifest.json` zeichnet die ausgewählte Eingabe, Exportoptionen, Konferenzdatensätze, Ausgabedateien, Anzahlen, Tokenquelle, jedes verwendete Kalenderereignis und Warnungen zu unvollständigen Abrufen auf. `--zip` schreibt außerdem ein portables Archiv neben den Ordner. `--include-doc-bodies` exportiert den Text verknüpfter Google Docs für Transkripte/intelligente Notizen über Drive `files.export` (erfordert den schreibgeschützten Drive-Meet-Geltungsbereich); ohne diese Option enthalten Exporte nur Meet-Metadaten und strukturierte Transkripteinträge. Bei einem teilweisen Artefaktfehler (Fehler beim Auflisten intelligenter Notizen, bei Transkripteinträgen oder Dokumentinhalten) bleibt die Warnung in der Zusammenfassung/im Manifest erhalten, anstatt den gesamten Export fehlschlagen zu lassen. `--dry-run` ruft dieselben Daten ab und gibt das Manifest-JSON aus, ohne den Ordner oder die ZIP-Datei zu erstellen.

Agenten verwenden dieselben Aktionen über das Werkzeug `google_meet` (`export`, `create` mit `accessType`, `end_active_conference`, `test_listen`); siehe [Werkzeug](#tool).

### Live-Funktionstest

```bash
OPENCLAW_LIVE_TEST=1 \
OPENCLAW_GOOGLE_MEET_LIVE_MEETING=https://meet.google.com/abc-defg-hij \
pnpm test:live -- extensions/google-meet/google-meet.live.test.ts
```

```bash
openclaw googlemeet setup --transport chrome-node --mode transcribe
openclaw googlemeet test-listen https://meet.google.com/abc-defg-hij --transport chrome-node --timeout-ms 30000
```

| Variable                                                                                                                  | Zweck                                                                  |
| ------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| `OPENCLAW_LIVE_TEST=1`                                                                                                    | Aktiviert abgesicherte Live-Tests                                      |
| `OPENCLAW_GOOGLE_MEET_LIVE_MEETING`                                                                                       | Aufbewahrte Meet-URL, Code oder `spaces/{id}`                     |
| `OPENCLAW_GOOGLE_MEET_CLIENT_ID` / `GOOGLE_MEET_CLIENT_ID`                                                                | OAuth-Client-ID                                                        |
| `OPENCLAW_GOOGLE_MEET_REFRESH_TOKEN` / `GOOGLE_MEET_REFRESH_TOKEN`                                                        | Aktualisierungstoken                                                   |
| `OPENCLAW_GOOGLE_MEET_CLIENT_SECRET`, `OPENCLAW_GOOGLE_MEET_ACCESS_TOKEN`, `OPENCLAW_GOOGLE_MEET_ACCESS_TOKEN_EXPIRES_AT` | Optional; dieselben Fallback-Namen funktionieren auch ohne das Präfix `OPENCLAW_` |

Der grundlegende Funktionstest für Artefakte/Anwesenheit benötigt `meetings.space.readonly` und `meetings.conference.media.readonly`. Die Kalendersuche benötigt `calendar.events.readonly`. Der Export von Drive-Dokumentinhalten benötigt `drive.meet.readonly`.

### Erstellungsbeispiele

```bash
openclaw googlemeet create
```

Gibt den neuen Besprechungs-URI, die Quelle und die Beitrittssitzung aus. Mit OAuth verwendet der Befehl die Meet API; ohne OAuth das angemeldete Profil der angehefteten Chrome-Node. JSON des Browser-Fallbacks:

```json
{
  "source": "browser",
  "meetingUri": "https://meet.google.com/abc-defg-hij",
  "joined": true,
  "browser": {
    "nodeId": "ba0f4e4bc...",
    "targetId": "tab-1"
  },
  "join": {
    "session": {
      "id": "meet_...",
      "url": "https://meet.google.com/abc-defg-hij"
    }
  }
}
```

Wenn der Browser-Fallback zuerst auf die Google-Anmeldung oder eine Meet-Berechtigungssperre stößt, gibt `google_meet` strukturierte Details anstelle einer einfachen Zeichenfolge zurück:

```json
{
  "source": "browser",
  "error": "google-login-required: Melden Sie sich im OpenClaw-Browserprofil bei Google an und versuchen Sie dann erneut, die Besprechung zu erstellen.",
  "manualActionRequired": true,
  "manualActionReason": "google-login-required",
  "manualActionMessage": "Melden Sie sich im OpenClaw-Browserprofil bei Google an und versuchen Sie dann erneut, die Besprechung zu erstellen.",
  "browser": {
    "nodeId": "ba0f4e4bc...",
    "targetId": "tab-1",
    "browserUrl": "https://accounts.google.com/signin",
    "browserTitle": "Sign in - Google Accounts"
  }
}
```

JSON der API-Erstellung:

```json
{
  "source": "api",
  "meetingUri": "https://meet.google.com/abc-defg-hij",
  "joined": true,
  "space": {
    "name": "spaces/abc-defg-hij",
    "meetingCode": "abc-defg-hij",
    "meetingUri": "https://meet.google.com/abc-defg-hij"
  },
  "join": {
    "session": {
      "id": "meet_...",
      "url": "https://meet.google.com/abc-defg-hij"
    }
  }
}
```

Beim Erstellen erfolgt standardmäßig ein Beitritt, aber Chrome/Chrome-Node benötigt weiterhin ein angemeldetes Google-Profil, um über den Browser beizutreten; wenn es abgemeldet ist, meldet OpenClaw `manualActionRequired: true` oder einen Browser-Fallbackfehler und fordert den Betreiber auf, die Google-Anmeldung abzuschließen, bevor er es erneut versucht.

Setzen Sie `preview.enrollmentAcknowledged: true` erst, nachdem bestätigt wurde, dass Ihr Cloud-Projekt, Ihr OAuth-Prinzipal und die Besprechungsteilnehmer am Google Workspace Developer Preview Program für Meet-Medien-APIs teilnehmen.

## Konfiguration

Der übliche Chrome-Agentenpfad benötigt lediglich das aktivierte Plugin, BlackHole, SoX, einen Echtzeit-Provider-Schlüssel und einen konfigurierten OpenClaw-TTS-Provider:

```json5
{
  plugins: {
    entries: {
      "google-meet": {
        enabled: true,
        config: {},
      },
    },
  },
}
```

### Standardwerte

| Schlüssel                           | Standardwert                             | Hinweise                                                                                                                                                                                                          |
| ----------------------------------- | ---------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `defaultTransport`                  | `"chrome"`                       |                                                                                                                                                                                                                   |
| `defaultMode`                  | `"agent"`                       | `"realtime"` wird als veralteter Alias für `"agent"` akzeptiert; neue Aufrufer sollten `"agent"` verwenden                                                                                 |
| `chromeNode.node`                  | nicht gesetzt                            | Node-ID/-Name/-IP für `chrome-node`; erforderlich, wenn mehr als ein geeigneter Node verbunden sein kann                                                                                                     |
| `chrome.launch`                  | `true`                       | Startet Chrome für den Beitritt; legen Sie `false` nur fest, wenn eine bereits geöffnete Sitzung wiederverwendet wird                                                                                   |
| `chrome.audioBackend`                  | `"blackhole-2ch"`                       |                                                                                                                                                                                                                   |
| `chrome.guestName`                  | `"OpenClaw Agent"`                       | Wird auf dem Meet-Gastbildschirm im abgemeldeten Zustand angezeigt                                                                                                                                                |
| `chrome.autoJoin`                  | `true`                       | Versucht nach bestem Ermessen, den Gastnamen einzutragen und auf `chrome-node` zu klicken                                                                                                                     |
| `chrome.reuseExistingTab`                  | `true`                       | Aktiviert einen vorhandenen Meet-Tab, anstatt Duplikate zu öffnen                                                                                                                                                  |
| `chrome.waitForInCallMs`                  | `20000`                       | Wartet, bis der Meet-Tab meldet, dass der Anruf läuft, bevor die Talkback-Begrüßung ausgelöst wird                                                                                                                 |
| `chrome.audioFormat`                  | `"pcm16-24khz"`                       | Audioformat für Befehlspaare; `"g711-ulaw-8khz"` ist nur für veraltete/benutzerdefinierte Befehlspaare vorgesehen, die Telefonie-Audio ausgeben                                                                    |
| `chrome.audioBufferBytes`                  | `4096`                       | SoX-Verarbeitungspuffer für erzeugte Audiobefehle von Befehlspaaren (die Hälfte des SoX-Standardpuffers von 8192 Byte, wodurch die Pipe-Latenz sinkt); Werte werden auf mindestens 17 Byte begrenzt                   |
| `chrome.audioInputCommand`                  | erzeugter SoX-Befehl                     | Liest von CoreAudio `BlackHole 2ch` und schreibt Audio im Format `chrome.audioFormat`                                                                                                                             |
| `chrome.audioOutputCommand`                  | erzeugter SoX-Befehl                     | Liest Audio im Format `chrome.audioFormat` und schreibt nach CoreAudio `BlackHole 2ch`                                                                                                                            |
| `chrome.bargeInInputCommand`                  | nicht gesetzt                            | Optionaler lokaler Mikrofonbefehl, der vorzeichenbehaftetes, monophones 16-Bit-Little-Endian-PCM zur Erkennung menschlichen Dazwischenredens während der Assistentenwiedergabe schreibt; gilt für die vom Gateway gehostete Befehlspaar-Bridge |
| `chrome.bargeInRmsThreshold`                  | `650`                       | RMS-Pegel, der als menschliche Unterbrechung gewertet wird                                                                                                                                                         |
| `chrome.bargeInPeakThreshold`                  | `2500`                       | Spitzenpegel, der als menschliche Unterbrechung gewertet wird                                                                                                                                                      |
| `chrome.bargeInCooldownMs`                  | `900`                       | Mindestverzögerung zwischen wiederholten Löschvorgängen aufgrund von Unterbrechungen                                                                                                                               |
| `mode` (pro Anfrage)    | `"agent"`                       | Talkback-Modus; siehe Tabelle [Agenten- und bidirektionale Modi](#agent-and-bidi-modes)                                                                                                                            |
| `realtime.provider`                  | `"openai"`                       | Kompatibilitäts-Fallback, der verwendet wird, wenn die nachstehenden bereichsspezifischen Felder nicht gesetzt sind                                                                                                |
| `realtime.transcriptionProvider`                  | `"openai"`                       | Provider-ID, die im Modus `agent` für die Echtzeittranskription verwendet wird                                                                                                                          |
| `realtime.voiceProvider`                  | nicht gesetzt                            | Provider-ID, die im Modus `bidi` für direkte Echtzeitsprachübertragung verwendet wird; setzen Sie sie für Gemini Live auf `"google"`, während für die Transkription im Agentenmodus OpenAI verwendet wird. Kombinieren Sie dies mit `realtime.model`, um das konkrete Gemini-Live-Modell auszuwählen. |
| `realtime.toolPolicy`                  | `"safe-read-only"`                       | Siehe [Agenten- und bidirektionale Modi](#agent-and-bidi-modes)                                                                                                                                                    |
| `realtime.instructions`                  | kurze Anweisungen für gesprochene Antworten | Weist das Modell an, sich kurz zu fassen und für ausführlichere Antworten `openclaw_agent_consult` zu verwenden                                                                                                          |
| `realtime.introMessage`                  | `"Say exactly: I'm here and listening."`                       | Wird einmal gesprochen, wenn die Echtzeit-Bridge eine Verbindung herstellt; auf `""` setzen, um ohne Begrüßung beizutreten                                                                           |
| `realtime.agentId`                  | `"main"`                       | OpenClaw-Agenten-ID, die für `openclaw_agent_consult` verwendet wird                                                                                                                                                     |
| `voiceCall.enabled`                  | `true`                       | Delegiert den Twilio-PSTN-Anruf, DTMF und die einleitende Begrüßung an das Voice-Call-Plugin                                                                                                                        |
| `voiceCall.dtmfDelayMs`                  | `12000`                       | Anfängliche Wartezeit vor der Wiedergabe einer aus einer PIN abgeleiteten DTMF-Sequenz über Twilio                                                                                                                 |
| `voiceCall.postDtmfSpeechDelayMs`                  | `5000`                       | Verzögerung, bevor die Echtzeit-Begrüßung angefordert wird, nachdem Voice Call den Twilio-Verbindungsabschnitt gestartet hat                                                                                       |

Mit `chrome.audioBridgeCommand` und `chrome.audioBridgeHealthCommand` kann eine externe Bridge den gesamten lokalen Audiopfad anstelle von `chrome.audioInputCommand`/`chrome.audioOutputCommand` übernehmen; die Einschränkung, welcher Modus sie verwenden kann, finden Sie unter [Hinweise](#notes).

Für die veraltete Struktur `realtime.provider: "google"` ist eine `openclaw doctor --fix`-Migration vorhanden: Sie überträgt diese Absicht auf `realtime.voiceProvider: "google"` und `realtime.transcriptionProvider: "openai"`, sofern diese Felder nicht bereits gesetzt sind.

### Optionale Überschreibungen

```json5
{
  defaults: {
    meeting: "https://meet.google.com/abc-defg-hij",
  },
  browser: {
    defaultProfile: "openclaw",
  },
  chrome: {
    guestName: "OpenClaw Agent",
    waitForInCallMs: 30000,
    bargeInInputCommand: [
      "sox",
      "-q",
      "-t",
      "coreaudio",
      "External Microphone",
      "-r",
      "24000",
      "-c",
      "1",
      "-b",
      "16",
      "-e",
      "signed-integer",
      "-t",
      "raw",
      "-",
    ],
  },
  chromeNode: {
    node: "parallels-macos",
  },
  defaultMode: "agent",
  realtime: {
    provider: "openai",
    transcriptionProvider: "openai",
    voiceProvider: "google",
    model: "gemini-3.1-flash-live-preview",
    agentId: "jay",
    toolPolicy: "owner",
    introMessage: "Sagen Sie genau: Ich bin hier.",
    providers: {
      google: {
        speakerVoice: "Kore",
      },
    },
  },
}
```

ElevenLabs sowohl für das Zuhören als auch für das Sprechen im Agentenmodus:

```json5
{
  tts: {
    provider: "elevenlabs",
    providers: {
      elevenlabs: {
        modelId: "eleven_v3",
        speakerVoiceId: "pMsXgVXv3BLzUgSXRplE",
      },
    },
  },
  plugins: {
    entries: {
      "google-meet": {
        config: {
          realtime: {
            transcriptionProvider: "elevenlabs",
            providers: {
              elevenlabs: {
                modelId: "scribe_v2_realtime",
                audioFormat: "ulaw_8000",
                sampleRate: 8000,
                commitStrategy: "vad",
              },
            },
          },
        },
      },
    },
  },
}
```

Die dauerhafte Meet-Stimme stammt aus `tts.providers.elevenlabs.speakerVoiceId`. Agentenantworten können auch antwortspezifische `[[tts:speakerVoiceId=... model=eleven_v3]]`-Direktiven verwenden, wenn Überschreibungen des TTS-Modells aktiviert sind; für Meetings ist die Konfiguration jedoch der deterministische Standard. Beim Beitritt zeigen die Protokolle `transcriptionProvider=elevenlabs`, und jede gesprochene Antwort protokolliert `provider=elevenlabs model=eleven_v3 speakerVoiceId=<voiceId>`.

Nur-Twilio-Konfiguration:

```json5
{
  defaultTransport: "twilio",
  twilio: {
    defaultDialInNumber: "+15551234567",
    defaultPin: "123456",
  },
  voiceCall: {
    gatewayUrl: "ws://127.0.0.1:18789",
  },
}
```

Mit `voiceCall.enabled: true` (dem Standardwert) und Twilio-Transport sendet Voice Call die DTMF-Sequenz, bevor der Echtzeit-Medienstream geöffnet wird, und verwendet anschließend den gespeicherten Begrüßungstext als anfängliche Echtzeit-Begrüßung. Wenn `voice-call` nicht aktiviert ist, kann Google Meet den Wählplan weiterhin validieren und aufzeichnen, aber den Twilio-Anruf nicht tätigen.

Lassen Sie `voiceCall.gatewayUrl` nicht gesetzt, um die lokale vertrauenswürdige Gateway-Laufzeit zu verwenden, die den
aufrufenden Agenten für den gesamten Aufruf beibehält. Eine konfigurierte Gateway-URL bleibt ein explizites WebSocket-Ziel und
kann die Herkunft des Plugins nicht authentifizieren; Beitritte von nicht standardmäßigen Agenten schlagen geschlossen fehl, statt stillschweigend
einen anderen Agenten zu verwenden. Führen Sie Google Meet und Voice Call im selben Gateway-Prozess aus, wenn ein agentenspezifisches
Routing erforderlich ist.

## Tool

Agenten verwenden das Tool `google_meet`:

```json
{
  "action": "join",
  "url": "https://meet.google.com/abc-defg-hij",
  "transport": "chrome-node",
  "mode": "agent"
}
```

| `action`                | Zweck                                                                                             |
| ----------------------- | ------------------------------------------------------------------------------------------------- |
| `join`                  | Einer expliziten Meet-URL beitreten                                                               |
| `create`                | Einen Raum erstellen (und standardmäßig beitreten); unterstützt `accessType`/`entryPointAccess` |
| `status`                | Aktive Sitzungen auflisten oder eine anhand von `sessionId` untersuchen                    |
| `setup_status`          | Dieselben Prüfungen wie `googlemeet setup` ausführen                                               |
| `resolve_space`         | Eine URL/einen Code/`spaces/{id}` über `spaces.get` auflösen                           |
| `preflight`             | Voraussetzungen für OAuth und die Besprechungsauflösung validieren                                 |
| `latest`                | Den neuesten Konferenzdatensatz für eine Besprechung finden                                        |
| `calendar_events`       | Kalenderereignisse mit Meet-Links in der Vorschau anzeigen                                         |
| `artifacts`             | Konferenzdatensätze und Metadaten zu Teilnehmern/Aufzeichnungen/Transkripten/Smart Notes auflisten |
| `attendance`            | Teilnehmer und Teilnehmersitzungen auflisten                                                       |
| `export`                | Das Paket mit Artefakten/Anwesenheit/Transkript/Manifest schreiben; `"dryRun": true` nur für das Manifest setzen |
| `recover_current_tab`   | Einen vorhandenen Meet-Tab fokussieren/untersuchen, ohne einen neuen zu öffnen                     |
| `transcript`            | Das begrenzte Untertiteltranskript lesen; `sinceIndex` setzt ab dem vorherigen `nextIndex` fort |
| `leave`                 | Eine Sitzung beenden (Chrome klickt auf Leave; schließt nur selbst geöffnete Tabs; Twilio legt auf) |
| `end_active_conference` | Die aktive Google-Meet-Konferenz für einen API-verwalteten Raum beenden                            |
| `speak`                 | Den Echtzeit-Agenten anhand von `sessionId` und `message` sofort sprechen lassen   |
| `test_speech`           | Eine Sitzung erstellen/wiederverwenden, eine bekannte Formulierung auslösen und den Chrome-Zustand zurückgeben |
| `test_listen`           | Eine reine Beobachtungssitzung erstellen/wiederverwenden und auf Bewegung bei Untertiteln/Transkript warten |

`test_speech` erzwingt immer `mode: "agent"` oder `"bidi"` und schlägt fehl, wenn die Ausführung in `mode: "transcribe"` angefordert wird, da reine Beobachtungssitzungen keine Sprache ausgeben können. `speechOutputVerified` erfordert sowohl neue Echtzeit-Ausgabebytes als auch neues nicht stummes Audio, das während dieser Ausgabe über den Mikrofon-Aufnahmepfad der Bridge zurückkehrt. Ältere Ausgaben oder Loopback-Signale einer wiederverwendeten Sitzung zählen nicht, und das alleinige Anwachsen der Sink-Bytes meldet keine verifizierte Sprachausgabe mehr.

Bei Chrome-Transporten lässt `leave` einen wiederverwendeten Tab im Besitz des Benutzers geöffnet, nachdem auf die Meet-Schaltfläche Leave call geklickt wurde. Von OpenClaw geöffnete Tabs werden nach dem Verlassen geschlossen.

Verwenden Sie `transport: "chrome"`, wenn Chrome auf dem Gateway-Host ausgeführt wird, und `transport: "chrome-node"`, wenn es auf einer gekoppelten Node ausgeführt wird. In beiden Fällen werden die Modell-Provider und `openclaw_agent_consult` auf dem Gateway-Host ausgeführt, sodass die Modell-Anmeldedaten dort verbleiben. Protokolle im Agentenmodus enthalten beim Start der Bridge den aufgelösten Transkriptions-Provider und das Modell sowie nach jeder synthetisierten Antwort den TTS-Provider, das Modell, die Stimme, das Ausgabeformat und die Abtastrate. Das unverarbeitete `mode: "realtime"` wird weiterhin als Legacy-Kompatibilitätsalias für `mode: "agent"` akzeptiert, aber nicht mehr in der `mode`-Enumeration des Tools angeboten.

`create` mit einem API-gestützten Raum und einer expliziten Zugriffsrichtlinie:

```json
{
  "action": "create",
  "transport": "chrome-node",
  "mode": "agent",
  "accessType": "OPEN"
}
```

Die aktive Konferenz eines bekannten Raums beenden:

```json
{
  "action": "end_active_conference",
  "meeting": "https://meet.google.com/abc-defg-hij"
}
```

Validierung mit vorrangigem Zuhören, bevor eine Besprechung als nutzbar bezeichnet wird:

```json
{
  "action": "test_listen",
  "url": "https://meet.google.com/abc-defg-hij",
  "transport": "chrome-node",
  "timeoutMs": 30000
}
```

Sprechen auf Anforderung:

```json
{
  "action": "speak",
  "sessionId": "meet_...",
  "message": "Sagen Sie genau: Ich bin hier und höre zu."
}
```

`status` enthält den Chrome-Zustand, sofern verfügbar:

| Feld                                                                  | Bedeutung                                                                                                               |
| --------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| `inCall`                                                              | Chrome scheint sich im Meet-Anruf zu befinden                                                                           |
| `micMuted`                                                            | Nach bestem Bemühen ermittelter Meet-Mikrofonstatus                                                                      |
| `manualActionRequired` / `manualActionReason` / `manualActionMessage` | Das Browserprofil benötigt eine manuelle Anmeldung, die Zulassung durch den Meet-Host, Berechtigungen oder eine Reparatur der Browsersteuerung, bevor die Sprachausgabe funktionieren kann |
| `speechReady` / `speechBlockedReason` / `speechBlockedMessage`        | Ob die verwaltete Chrome-Sprachausgabe derzeit zulässig ist; `speechReady: false` bedeutet, dass OpenClaw die Einleitungs-/Testformulierung nicht gesendet hat |
| `providerConnected` / `realtimeReady`                                 | Zustand der Echtzeit-Sprach-Bridge                                                                                       |
| `lastInputAt` / `lastOutputAt`                                        | Zuletzt von der Bridge empfangenes/an sie gesendetes Audio                                                               |
| `audioOutputRouted` / `audioOutputDeviceLabel`                        | Ob die Medienausgabe des Meet-Tabs aktiv an das BlackHole-Gerät der Bridge weitergeleitet wurde                         |
| `lastOutputLoopbackAt` / `outputLoopbackSignalBytes`                  | Neue Ausgabe, deren Wellenform-Fingerabdruck mit dem BlackHole-Mikrofon-Aufnahmepfad korreliert wurde                    |
| `lastOutputLoopbackCorrelation`                                       | Korrelationswert, der das aufgenommene Signal der aktuellen Generierung der Assistentenausgabe zuordnet                 |
| `outputGeneration` / `verifiedOutputGeneration`                       | Monotone IDs; Gleichheit bedeutet, dass die aktuelle Ausgabe und nicht eine ältere Äußerung den Loopback-Nachweis bestanden hat |
| `lastOutputLoopbackRms` / `lastOutputLoopbackPeak`                    | Audioenergiediagnosen für den zuletzt verifizierten Loopback-Aufnahmeblock                                               |
| `lastSuppressedInputAt` / `suppressedInputBytes`                      | Loopback-Eingabe wird ignoriert, während die Assistentenwiedergabe aktiv ist                                             |

## Agenten- und Bidi-Modi

| Modus   | Wer über die Antwort entscheidet | Pfad der Sprachausgabe                 | Verwenden, wenn                                        |
| ------- | --------------------------------- | -------------------------------------- | ------------------------------------------------------ |
| `agent` | Der konfigurierte OpenClaw-Agent | Normale OpenClaw-TTS-Laufzeit          | Sie das Verhalten „Mein Agent ist in der Besprechung“ wünschen |
| `bidi`  | Das Echtzeit-Sprachmodell         | Audioantwort des Echtzeit-Sprach-Providers | Sie die Konversations-Sprachschleife mit der geringsten Latenz wünschen |

`agent`-Modus: Der Echtzeit-Transkriptions-Provider hört das Besprechungsaudio, endgültige Teilnehmertranskripte werden durch den konfigurierten OpenClaw-Agenten geleitet und die Antwort wird über die reguläre OpenClaw-TTS ausgegeben. Zeitlich nahe endgültige Transkriptfragmente werden vor der Konsultation zusammengeführt, damit eine gesprochene Äußerung nicht mehrere veraltete Teilantworten erzeugt; Echtzeiteingaben werden unterdrückt, solange Audioausgaben des Assistenten in der Warteschlange noch wiedergegeben werden, und kürzlich aufgezeichnete, assistentenähnliche Transkriptechos werden vor der Konsultation ignoriert, damit der BlackHole-Loopback den Agenten nicht auf seine eigene Sprachausgabe antworten lässt.

`bidi`-Modus: Das Echtzeit-Sprachmodell antwortet direkt und kann `openclaw_agent_consult` für tiefere Schlussfolgerungen, aktuelle Informationen oder normale OpenClaw-Tools aufrufen. Das Konsultationstool führt im Hintergrund den regulären OpenClaw-Agenten mit dem Kontext des aktuellen Besprechungstranskripts aus und gibt eine prägnante gesprochene Antwort zurück; im `agent`-Modus sendet OpenClaw diese Antwort direkt an TTS, im `bidi`-Modus kann das Echtzeit-Sprachmodell sie aussprechen. Es verwendet dieselbe gemeinsame Konsultationslogik wie Voice Call.

Standardmäßig werden Konsultationen für den Agenten `main` ausgeführt; setzen Sie `realtime.agentId`, um eine Meet-Spur auf einen dedizierten Agenten-Arbeitsbereich, Modellstandards, eine Tool-Richtlinie, den Speicher und den Sitzungsverlauf auszurichten. Konsultationen im Agentenmodus verwenden einen besprechungsspezifischen `agent:<id>:subagent:google-meet:<session>`-Sitzungsschlüssel, damit Folgefragen den Besprechungskontext beibehalten und zugleich die normale Agentenrichtlinie übernehmen. Wenn ein Agent `google_meet` im Agentenmodus aufruft, verzweigt die Konsultationssitzung vor der Beantwortung der Teilnehmeräußerung vom aktuellen Transkript des Aufrufers; die Meet-Sitzung bleibt getrennt, damit Folgefragen in der Besprechung das Transkript des Aufrufers nicht direkt verändern.

`realtime.toolPolicy` steuert den Konsultationslauf:

| Richtlinie       | Verhalten                                                                                                                        |
| ---------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `safe-read-only` | Das Konsultationstool bereitstellen; den regulären Agenten auf `read`, `web_search`, `web_fetch`, `x_search`, `memory_search`, `memory_get` beschränken |
| `owner`          | Das Konsultationstool bereitstellen; dem regulären Agenten die Verwendung seiner normalen Tool-Richtlinie erlauben               |
| `none`           | Das Konsultationstool dem Echtzeit-Sprachmodell nicht bereitstellen                                                              |

Der Konsultationssitzungsschlüssel ist auf die jeweilige Meet-Sitzung beschränkt, sodass nachfolgende Konsultationsaufrufe während derselben Besprechung den vorherigen Konsultationskontext wiederverwenden.

Eine gesprochene Bereitschaftsprüfung erzwingen, nachdem Chrome vollständig beigetreten ist:

```bash
openclaw googlemeet speak meet_... "Say exactly: I'm here and listening."
```

Vollständiger Beitritts- und Sprachtest:

```bash
openclaw googlemeet test-speech https://meet.google.com/abc-defg-hij \
  --transport chrome-node \
  --message "Say exactly: I'm here and listening."
```

## Checkliste für Live-Tests

Bevor Sie eine Besprechung an einen unbeaufsichtigten Agenten übergeben:

```bash
openclaw googlemeet setup
openclaw nodes status
openclaw googlemeet test-speech https://meet.google.com/abc-defg-hij \
  --transport chrome-node \
  --message "Say exactly: Google Meet speech test complete."
```

Erwarteter Chrome-Node-Zustand:

- `googlemeet setup` ist vollständig grün und umfasst `chrome-node-connected`, wenn Chrome-node der Standardtransport oder ein Node festgelegt ist.
- `nodes status` zeigt den ausgewählten Node als verbunden an, wobei sowohl `googlemeet.chrome` als auch `browser.proxy` angekündigt werden.
- Der Meet-Tab tritt bei, und `test-speech` gibt den Chrome-Zustand mit `inCall: true` zurück.

Für einen entfernten Chrome-Host wie eine Parallels-macOS-VM ist dies die kürzeste sichere Prüfung nach einer Aktualisierung des Gateway oder der VM:

```bash
openclaw googlemeet setup
openclaw nodes status --connected
openclaw nodes invoke \
  --node parallels-macos \
  --command googlemeet.chrome \
  --params '{"action":"setup"}'
```

Damit wird nachgewiesen, dass das Gateway-Plugin geladen ist, der VM-Node mit dem aktuellen Token verbunden ist und die Meet-Audiobrücke verfügbar ist, bevor ein Agent einen echten Besprechungs-Tab öffnet.

Verwenden Sie für einen Twilio-Smoke-Test eine Besprechung, die Telefoneinwahldaten bereitstellt:

```bash
openclaw googlemeet setup
openclaw googlemeet join https://meet.google.com/abc-defg-hij \
  --transport twilio \
  --dial-in-number +15551234567 \
  --pin 123456
```

Erwarteter Twilio-Status:

- `googlemeet setup` umfasst grüne Prüfungen für `twilio-voice-call-plugin`, `twilio-voice-call-credentials` und `twilio-voice-call-webhook`.
- `voicecall` ist nach dem Neuladen des Gateway in der CLI verfügbar.
- Die zurückgegebene Sitzung enthält `transport: "twilio"` und eine `twilio.voiceCallId`.
- `openclaw logs --follow` zeigt, dass DTMF-TwiML vor Echtzeit-TwiML bereitgestellt wurde, gefolgt von einer Echtzeitbrücke mit der eingereihten anfänglichen Begrüßung.
- `googlemeet leave <sessionId>` beendet den delegierten Sprachanruf.

## Fehlerbehebung

### Agent kann das Google-Meet-Tool nicht sehen

Vergewissern Sie sich, dass das Plugin aktiviert ist, und laden Sie das Gateway neu. Der laufende Agent sieht nur Plugin-Tools, die vom aktuellen Gateway-Prozess registriert wurden:

```bash
openclaw plugins list | grep google-meet
openclaw googlemeet setup
```

Auf Gateway-Hosts ohne macOS bleibt `google_meet` sichtbar, lokale Chrome-Talkback-Aktionen werden jedoch blockiert, bevor sie die Audiobrücke erreichen. Verwenden Sie statt des standardmäßigen lokalen Chrome-Agent-Pfads `mode: "transcribe"`, die Twilio-Einwahl oder einen macOS-Host vom Typ `chrome-node`.

### Kein verbundener Google-Meet-fähiger Node

Auf dem Node-Host:

```bash
openclaw plugins install npm:@openclaw/google-meet
openclaw plugins enable browser
OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1 \
  openclaw node run --host <gateway-lan-ip> --port 18789 --display-name parallels-macos
```

Auf dem Gateway-Host:

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw nodes status
```

Der Node muss verbunden sein und `googlemeet.chrome` sowie `browser.proxy` auflisten. Die Gateway-Konfiguration muss beides zulassen:

```json5
{
  gateway: {
    nodes: {
      commands: { allow: ["browser.proxy", "googlemeet.chrome"] },
    },
  },
}
```

Wenn `googlemeet setup` bei `chrome-node-connected` fehlschlägt oder das Gateway-Protokoll `gateway token mismatch` meldet, installieren Sie den Node mit dem aktuellen Gateway-Token neu oder starten Sie ihn damit neu:

```bash
OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1 \
  openclaw node install \
  --host <gateway-lan-ip> \
  --port 18789 \
  --display-name parallels-macos \
  --force
```

Laden Sie anschließend den Node-Dienst neu und führen Sie Folgendes erneut aus:

```bash
openclaw googlemeet setup
openclaw nodes status --connected
```

### Browser wird geöffnet, aber Agent kann nicht beitreten

Führen Sie `googlemeet test-listen` für reine Beobachterbeitritte oder `googlemeet test-speech` für Echtzeitbeitritte aus und prüfen Sie anschließend den zurückgegebenen Chrome-Zustand. Wenn eine der beiden Aktionen `manualActionRequired: true` meldet, zeigen Sie dem Bediener `manualActionMessage` und versuchen Sie es erst erneut, wenn die Browseraktion abgeschlossen ist.

Häufige manuelle Aktionen: beim Chrome-Profil anmelden; den Gast über das Meet-Hostkonto zulassen; Chrome Mikrofon-/Kameraberechtigungen erteilen, wenn die native Eingabeaufforderung erscheint; einen hängen gebliebenen Meet-Berechtigungsdialog schließen oder beheben.

Melden Sie nicht „nicht angemeldet“, nur weil Meet fragt „Do you want people to hear you in the meeting?“; dies ist die Audioauswahl-Zwischenseite von Meet. OpenClaw klickt, sofern verfügbar, per Browserautomatisierung auf **Use microphone** und wartet weiter auf den tatsächlichen Besprechungsstatus. Beim reinen Erstellen über den Browser-Fallback kann OpenClaw stattdessen auf **Continue without microphone** klicken, da zum Erzeugen der URL kein Echtzeit-Audiopfad erforderlich ist.

### Erstellen der Besprechung schlägt fehl

`googlemeet create` verwendet bei konfiguriertem OAuth die Meet-API `spaces.create`, andernfalls den Browser des festgelegten Chrome-Node. Prüfen Sie Folgendes:

- **API-Erstellung**: `oauth.clientId` und `oauth.refreshToken` (oder entsprechende `OPENCLAW_GOOGLE_MEET_*`-Umgebungsvariablen) sind vorhanden, und das Aktualisierungstoken wurde erstellt, nachdem die Erstellungsunterstützung hinzugefügt wurde. Älteren Token fehlt möglicherweise `meetings.space.created`; führen Sie daher `openclaw googlemeet auth login --json` erneut aus.
- **Browser-Fallback**: `defaultTransport: "chrome-node"` und `chromeNode.node` verweisen auf einen verbundenen Node mit `browser.proxy` und `googlemeet.chrome`. Das OpenClaw-Chrome-Profil auf diesem Node ist angemeldet und kann `https://meet.google.com/new` öffnen.
- **Browser-Fallback-Wiederholungen**: Verwenden Sie einen vorhandenen `.../new`- oder Google-Konto-Eingabeaufforderungs-Tab erneut, bevor Sie einen neuen öffnen. Wiederholen Sie den Tool-Aufruf, statt manuell einen weiteren Tab zu öffnen.
- **Manuelle Aktion**: Wenn das Tool `manualActionRequired: true` zurückgibt, verwenden Sie `browser.nodeId`, `browser.targetId`, `browserUrl` und `manualActionMessage`, um den Bediener anzuleiten. Wiederholen Sie den Vorgang nicht in einer Schleife.
- **Audioauswahl-Zwischenseite**: Wenn Meet „Do you want people to hear you in the meeting?“ anzeigt, lassen Sie den Tab geöffnet. OpenClaw sollte auf **Use microphone** oder – nur beim Erstellen – auf **Continue without microphone** klicken und weiter auf die generierte URL warten. Ist dies nicht möglich, sollte der Fehler `meet-audio-choice-required` und nicht `google-login-required` erwähnen.

### Agent tritt bei, spricht aber nicht

```bash
openclaw googlemeet setup
openclaw googlemeet doctor
```

Verwenden Sie `mode: "agent"` für den Pfad STT -> OpenClaw-Agent -> TTS und `mode: "bidi"` für den direkten Echtzeit-Sprachfallback. `mode: "transcribe"` startet absichtlich keine Talkback-Brücke. Führen Sie zur Fehlerbehebung im reinen Beobachtermodus `openclaw googlemeet status --json <session-id>` aus, nachdem Teilnehmer gesprochen haben, und prüfen Sie `captioning`, `transcriptLines` und `lastCaptionText`. Wenn `inCall` wahr ist, `transcriptLines` jedoch `0` bleibt, sind möglicherweise Meet-Untertitel deaktiviert, seit der Installation des Beobachters hat niemand gesprochen, die Meet-Benutzeroberfläche wurde geändert oder Live-Untertitel sind für die Sprache beziehungsweise das Konto der Besprechung nicht verfügbar.

`googlemeet test-speech` prüft immer den Echtzeitpfad und meldet, ob bei diesem Aufruf Ausgabebytes der Brücke beobachtet wurden. Wenn `speechOutputVerified` falsch und `speechOutputTimedOut` wahr ist, hat der Echtzeit-Provider die Äußerung möglicherweise akzeptiert, OpenClaw hat jedoch nicht beobachtet, dass neue Ausgabebytes die Chrome-Audiobrücke erreichten.

Prüfen Sie außerdem Folgendes: Auf dem Gateway-Host ist ein Schlüssel für einen Echtzeit-Provider (`OPENAI_API_KEY` oder `GEMINI_API_KEY`) verfügbar; `BlackHole 2ch` ist auf dem Chrome-Host sichtbar; `sox` ist dort vorhanden; Meet-Mikrofon und -Lautsprecher werden über den virtuellen Audiopfad geleitet (`doctor` sollte bei lokalen Chrome-Echtzeitbeitritten `meet output routed: yes` anzeigen).

`googlemeet doctor [session-id]` gibt Sitzung, Node, Anrufstatus, Grund für die manuelle Aktion, Verbindung zum Echtzeit-Provider, `realtimeReady`, Audioeingabe-/Audioausgabeaktivität, letzte Audiozeitstempel, Bytezähler und Browser-URL aus. Verwenden Sie `googlemeet status [session-id] --json` für unformatiertes JSON und `googlemeet doctor --oauth` (ergänzen Sie `--meeting` oder `--create-space`), um die OAuth-Aktualisierung zu überprüfen, ohne Token offenzulegen.

Wenn bei einem Agent eine Zeitüberschreitung aufgetreten ist und bereits ein Meet-Tab geöffnet ist, prüfen Sie ihn, ohne einen weiteren zu öffnen:

```bash
openclaw googlemeet recover-tab
openclaw googlemeet recover-tab https://meet.google.com/abc-defg-hij
```

Die entsprechende Tool-Aktion ist `recover_current_tab`: Sie fokussiert und prüft einen vorhandenen Meet-Tab für den ausgewählten Transport – lokale Browsersteuerung für `chrome`, der konfigurierte Node für `chrome-node` –, ohne einen neuen Tab oder eine neue Sitzung zu öffnen, und meldet die aktuelle Blockade (Anmeldung, Zulassung, Berechtigungen, Audioauswahlstatus). Der CLI-Befehl kommuniziert mit dem konfigurierten Gateway, das ausgeführt werden muss. `chrome-node` setzt außerdem voraus, dass der Node verbunden ist.

### Twilio-Einrichtungsprüfungen schlagen fehl

`twilio-voice-call-plugin` schlägt fehl, wenn `voice-call` nicht zulässig oder nicht aktiviert ist: Fügen Sie es zu `plugins.allow` hinzu, aktivieren Sie `plugins.entries.voice-call` und laden Sie das Gateway neu.

`twilio-voice-call-credentials` schlägt fehl, wenn im Twilio-Backend die Konto-SID, das Authentifizierungstoken oder die Anrufernummer fehlt:

```bash
export TWILIO_ACCOUNT_SID=AC...
export TWILIO_AUTH_TOKEN=...
export TWILIO_FROM_NUMBER=+15550001234
```

`twilio-voice-call-webhook` schlägt fehl, wenn `voice-call` keine öffentliche Webhook-Bereitstellung besitzt oder `publicUrl` auf den Loopback-/privaten Netzwerkbereich verweist. Verwenden Sie `localhost`, `127.0.0.1`, `0.0.0.0`, `10.x`, `172.16.x`-`172.31.x`, `192.168.x`, `169.254.x`, `fc00::/7` oder `fd00::/8` nicht als `publicUrl`; Netzbetreiber-Callbacks können diese Adressen nicht erreichen. Legen Sie `plugins.entries.voice-call.config.publicUrl` auf eine öffentliche URL fest oder konfigurieren Sie die Bereitstellung über einen Tunnel/Tailscale:

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        enabled: true,
        config: {
          provider: "twilio",
          fromNumber: "+15550001234",
          publicUrl: "https://voice.example.com/voice/webhook",
        },
      },
    },
  },
}
```

Verwenden Sie für die lokale Entwicklung statt einer privaten Host-URL die Bereitstellung über einen Tunnel oder Tailscale:

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          tunnel: { provider: "ngrok" },
          // oder
          tailscale: { mode: "funnel", path: "/voice/webhook" },
        },
      },
    },
  },
}
```

Starten oder laden Sie das Gateway neu und führen Sie anschließend Folgendes aus:

```bash
openclaw googlemeet setup --transport twilio
openclaw voicecall setup
openclaw voicecall smoke
```

`voicecall smoke` prüft standardmäßig nur die Bereitschaft. Führen Sie einen Probelauf für eine bestimmte Nummer durch:

```bash
openclaw voicecall smoke --to "+15555550123"
```

Fügen Sie `--yes` nur hinzu, wenn absichtlich ein echter ausgehender Anruf getätigt werden soll:

```bash
openclaw voicecall smoke --to "+15555550123" --yes
```

### Twilio-Anruf beginnt, tritt der Besprechung jedoch nie bei

Vergewissern Sie sich, dass das Meet-Ereignis Telefoneinwahldaten bereitstellt, und übergeben Sie die genaue Einwahlnummer sowie die PIN oder eine benutzerdefinierte DTMF-Sequenz:

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij \
  --transport twilio \
  --dial-in-number +15551234567 \
  --dtmf-sequence ww123456#
```

Verwenden Sie führende `w` oder Kommas in `--dtmf-sequence` für eine Pause vor der PIN.

Wenn der Anruf erstellt wurde, der Einwahlteilnehmer jedoch nie in der Meet-Teilnehmerliste erscheint:

- `openclaw googlemeet doctor <session-id>`: Prüfen Sie die delegierte Twilio-Anruf-ID, ob DTMF eingereiht wurde und ob die Einführungsbegrüßung angefordert wurde.
- `openclaw voicecall status --call-id <id>`: Prüfen Sie, ob der Anruf noch aktiv ist.
- `openclaw voicecall tail`: Prüfen Sie, ob Twilio-Webhooks beim Gateway eingehen.
- `openclaw logs --follow`: Suchen Sie nach der Twilio-Meet-Sequenz: Google Meet delegiert den Beitritt, Voice Call speichert Pre-Connect-DTMF-TwiML und stellt es bereit, Voice Call stellt Echtzeit-TwiML für den Twilio-Anruf bereit, anschließend fordert Google Meet mit `voicecall.speak` die Einführungsansage an.
- Führen Sie `openclaw googlemeet setup --transport twilio` erneut aus. Eine grüne Einrichtungsprüfung ist erforderlich, weist jedoch nicht nach, dass die Besprechungs-PIN-Sequenz korrekt ist.
- Vergewissern Sie sich, dass Einwahlnummer und PIN zu derselben Meet-Einladung und Region gehören.
- Erhöhen Sie `voiceCall.dtmfDelayMs` gegenüber dem Standardwert von 12 Sekunden, wenn Meet langsam antwortet oder das Anruftranskript nach dem Senden des Pre-Connect-DTMF weiterhin die PIN-Eingabeaufforderung anzeigt.
- Wenn der Teilnehmer beitritt, Sie die Begrüßung jedoch nicht hören, prüfen Sie `openclaw logs --follow` auf die nach dem DTMF gesendete `voicecall.speak`-Anforderung und entweder die TTS-Wiedergabe des Medienstreams oder den Twilio-Fallback `<Say>`. Wenn das Transkript weiterhin „enter the meeting PIN“ anzeigt, ist der Telefonabschnitt dem Meet-Raum noch nicht beigetreten, sodass die Teilnehmer keine Sprache hören.

Wenn Webhooks nicht eintreffen, debuggen Sie zuerst das Voice-Call-Plugin: Der Provider muss `plugins.entries.voice-call.config.publicUrl` oder den konfigurierten Tunnel erreichen können. Siehe [Fehlerbehebung bei Sprachanrufen](/de/plugins/voice-call#troubleshooting).

## Hinweise

Die offizielle Medien-API von Google Meet ist auf den Empfang ausgerichtet, daher ist zum Sprechen in einem Anruf weiterhin ein Teilnehmerpfad erforderlich. Dieses Plugin macht diese Abgrenzung sichtbar: Chrome übernimmt die Browserteilnahme und das lokale Audio-Routing; Twilio übernimmt die Teilnahme per Telefoneinwahl.

Die Talkback-Modi von Chrome benötigen `BlackHole 2ch` sowie eine der folgenden Optionen:

- `chrome.audioInputCommand` plus `chrome.audioOutputCommand`: OpenClaw verwaltet die Bridge und leitet Audio in `chrome.audioFormat` zwischen diesen Befehlen und dem ausgewählten Provider weiter. Der Modus `agent` verwendet Echtzeittranskription plus reguläres TTS; der Modus `bidi` verwendet den Echtzeit-Sprach-Provider. Der Standardpfad ist 24 kHz PCM16 mit `chrome.audioBufferBytes: 4096`; 8 kHz G.711 mu-law bleibt für ältere Befehlspaare verfügbar.
- `chrome.audioBridgeCommand`: Ein externer Bridge-Befehl verwaltet den gesamten lokalen Audiopfad und muss nach dem Starten oder Validieren seines Daemons beendet werden. Nur für `bidi` gültig, da der Modus `agent` für TTS direkten Zugriff auf das Befehlspaar benötigt.

Bei der Chrome-Bridge mit Befehlspaar kann `chrome.bargeInInputCommand` ein separates lokales Mikrofon abhören und die Wiedergabe des Assistenten löschen, sobald ein Mensch zu sprechen beginnt. Dadurch hat menschliche Sprache Vorrang vor der Ausgabe des Assistenten, selbst wenn der gemeinsam genutzte BlackHole-Loopback-Eingang während der Wiedergabe des Assistenten vorübergehend unterdrückt wird. Wie `chrome.audioInputCommand`/`chrome.audioOutputCommand` handelt es sich um einen vom Operator konfigurierten lokalen Befehl: Verwenden Sie einen expliziten vertrauenswürdigen Befehlspfad oder eine Argumentliste, niemals ein Skript aus einem nicht vertrauenswürdigen Speicherort.

Für sauberes Duplex-Audio leiten Sie die Meet-Ausgabe und das Meet-Mikrofon über separate virtuelle Geräte oder einen virtuellen Gerätegraphen im Stil von Loopback; ein einzelnes gemeinsam genutztes BlackHole-Gerät kann Audio anderer Teilnehmer zurück in den Anruf spiegeln.

`googlemeet speak` löst die aktive Talkback-Audio-Bridge für eine Chrome-Sitzung aus; `googlemeet leave` stoppt sie (und legt bei Twilio-Sitzungen, die über Voice Call delegiert wurden, den zugrunde liegenden Anruf auf). Verwenden Sie `googlemeet end-active-conference`, um außerdem die aktive Google-Meet-Konferenz für einen API-verwalteten Bereich zu schließen.

## Verwandte Themen

- [Übersicht über Meeting-Plugins](/de/plugins/meeting-plugins)
- [Voice-Call-Plugin](/de/plugins/voice-call)
- [Sprechmodus](/de/nodes/talk)
- [Plugins erstellen](/de/plugins/building-plugins)
