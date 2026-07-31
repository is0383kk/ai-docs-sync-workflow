---
read_when:
    - Sie möchten, dass ein OpenClaw-Agent an einer Microsoft Teams-Besprechung teilnimmt
    - Sie konfigurieren Chrome, BlackHole oder SoX für die Sprachwiedergabe in Teams-Besprechungen
summary: 'Microsoft-Teams-Besprechungs-Plugin: Als Gast im Chrome-Browser an geschäftlichen oder privaten Besprechungen teilnehmen'
title: Microsoft Teams-Besprechungs-Plugin
x-i18n:
    generated_at: "2026-07-26T18:01:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6f84e58d478185d026dd79a02a8500af48f51689ef6865d56badb0e27c6d2814
    source_path: plugins/teams-meetings.md
    workflow: 16
---

Das `teams-meetings`-Plugin tritt Microsoft Teams-Links als Gast im OpenClaw-Chrome-Profil bei. Es akzeptiert geschäftliche Links unter `teams.microsoft.com/l/meetup-join/...` und Links für Privatkunden unter `teams.live.com/meet/...`. Es erstellt keine Besprechungen, wählt sich nicht ein, ruft Microsoft Graph nicht auf und zeichnet weder Audio noch Video auf.

## Einrichtung

Die Sprachausgabe verwendet dieselben lokalen Audiovoraussetzungen wie das [Google Meet-Plugin](/de/plugins/google-meet): macOS, das virtuelle Audiogerät `BlackHole 2ch` und SoX.

```bash
brew install blackhole-2ch sox
sudo reboot
system_profiler SPAudioDataType | grep -i BlackHole
command -v sox
```

Das Plugin ist enthalten und standardmäßig aktiviert. Fügen Sie nur dann einen Eintrag hinzu, wenn Sie es anpassen möchten, und prüfen Sie anschließend die Einrichtung:

```json5
{
  plugins: {
    entries: {
      "teams-meetings": {
        config: {
          defaultMode: "agent",
          chrome: { guestName: "OpenClaw Agent" },
        },
      },
    },
  },
}
```

Führen Sie `openclaw plugins disable teams-meetings` aus, wenn das Plugin nicht aktiv sein soll.

```bash
openclaw teamsmeetings setup
openclaw teamsmeetings join 'https://teams.microsoft.com/l/meetup-join/...'
```

Verwenden Sie `chromeNode.node`, um Chrome, BlackHole und SoX auf einer gekoppelten macOS-Node auszuführen. Die Node muss `teamsmeetings.chrome` und `browser.proxy` zulassen.

## Modi

| Modus         | Verhalten                                                                    |
| ------------ | --------------------------------------------------------------------------- |
| `agent`      | Die Echtzeittranskription konsultiert den konfigurierten OpenClaw-Agenten; TTS antwortet. |
| `bidi`       | Ein Echtzeit-Sprachmodell hört zu und antwortet direkt.                        |
| `transcribe` | Beitritt nur zur Beobachtung mit Momentaufnahmen des Live-Untertiteltranskripts.                   |

Teams-Liveuntertitel werden nach der Zulassung in jedem Modus aktiviert, damit OpenClaw
Notizen mit Sprecherzuordnung dauerhaft speichern kann. Die Aktion `transcript` gibt weiterhin nur für
`transcribe`-Sitzungen den begrenzten Live-Puffer zurück. Beim Verlassen speichert OpenClaw
das dauerhafte Transkript und die daraus abgeleitete Zusammenfassung in der gemeinsamen Zustandsdatenbank; listen
oder exportieren Sie diese mit [`openclaw transcripts`](/de/cli/transcripts).

Automatische Notizen sind standardmäßig aktiviert. Setzen Sie `transcripts.enabled: false`, um
dauerhafte Notizen global zu deaktivieren; der explizite Modus `transcribe` stellt weiterhin nur
seinen begrenzten Live-Ausschnitt bereit.

## Einschränkungen beim Gastbeitritt

Der Browseradapter schließt den App-Zwischenbildschirm, trägt den Gastnamen ein, schaltet die Kamera aus, konfiguriert das Mikrofon für den ausgewählten Modus und klickt auf die Schaltfläche zum Beitritt. Im Anrufstatus wird das Steuerelement zum Auflegen verwendet; bei Lobby-, Mandantenanmeldungs- und Geräteberechtigungsstatus werden eindeutige Gründe für erforderliche manuelle Aktionen zurückgegeben. Weiterleitungen des Startprogramms für Besprechungen für Privatkunden und die von Chrome angezeigten `BlackHole 2ch (Virtual)`-Beschriftungen werden unterstützt.

Die Teams-Mandantenrichtlinie kann eine Anmeldung, E-Mail-Verifizierung oder Zulassung durch den Organisator erfordern. Führen Sie diesen Schritt im OpenClaw-Chrome-Profil aus und prüfen Sie anschließend erneut den Status oder die Sprachausgabe. Das Plugin umgeht die Mandantenrichtlinie nicht.

Der Teams-Webclient für Privatkunden wurde für den App-Zwischenbildschirm, die Eingabe des Gastnamens, Mikrofon-/Kamera-Umschalter vor dem Beitritt, den Beitritt, die Zulassung aus der Lobby, Medienberechtigungen, die Erkennung laufender Anrufe, Liveuntertitel, die Ein-/Ausgabeweiterleitung über BlackHole, das Verlassen und die Erkennung nach dem Anruf live validiert. Geschäftliche Mandanten können abweichende Richtlinien für Anmeldung, E-Mail-Verifizierung, Zulassung und Bestätigung beim Verlassen durchsetzen; führen Sie alle gemeldeten manuellen Aktionen im OpenClaw-Chrome-Profil aus.

## Tool- und Gateway-Oberfläche

Das Agenten-Tool `teams_meetings` unterstützt `join`, `leave`, `status`, `transcript` und `speak`. Gateway-Methoden verwenden das Präfix `teamsmeetings.*`. Der Node-Befehl lautet `teamsmeetings.chrome`.

## Verwandte Themen

- [Übersicht über Besprechungs-Plugins](/de/plugins/meeting-plugins)
- [Microsoft Teams-Kanal](/de/channels/msteams)
