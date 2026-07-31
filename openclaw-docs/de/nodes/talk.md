---
read_when:
    - Implementierung des Sprechmodus unter macOS/iOS/Android
    - Sprach-/TTS-/Unterbrechungsverhalten ändern
summary: 'Sprechmodus: kontinuierliche Sprachunterhaltungen über lokale STT/TTS und Echtzeitsprachübertragung'
title: Sprechmodus
x-i18n:
    generated_at: "2026-07-26T17:52:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b21319eee169ba898331f87279a2b2a5170441131a1e9cdc85c15b268d165e21
    source_path: nodes/talk.md
    workflow: 16
---

Der Talk-Modus umfasst fünf Laufzeitvarianten:

- **Nativer Talk unter macOS/iOS/Android**: native Spracherkennung, Gateway-Chat und `talk.speak`-TTS. Die Apple-Spracherkennung unter macOS/iOS kann Netzwerkdienste verwenden; das Verhalten unter Android hängt vom installierten Sprachdienst ab. Nodes geben die Fähigkeit `talk` bekannt und deklarieren, welche `talk.*`-Befehle sie unterstützen.
- **iOS-Talk (Echtzeit)**: clientseitig verwaltetes WebRTC für OpenAI-Echtzeitkonfigurationen, die den Transport `webrtc` auswählen oder keinen Transport angeben. Explizite `gateway-relay`-, `provider-websocket`- und Nicht-OpenAI-Echtzeitkonfigurationen verbleiben beim vom Gateway verwalteten Relay; Nicht-Echtzeitkonfigurationen verwenden die native Sprachschleife.
- **Browser-Talk**: `talk.client.create` für clientseitig verwaltete `webrtc`- beziehungsweise `provider-websocket`-Sitzungen oder `talk.session.create` für vom Gateway verwaltete `gateway-relay`-Sitzungen. `managed-room` ist für die Übergabe an das Gateway und für Walkie-Talkie-Räume reserviert.
- **Android-Talk (Echtzeit)**: Aktivierung mit `talk.realtime.mode: "realtime"` und `talk.realtime.transport: "gateway-relay"`. Andernfalls verwendet Android weiterhin native Spracherkennung, Gateway-Chat und `talk.speak`.
- **Clients nur für Transkription**: `talk.session.create({ mode: "transcription", transport: "gateway-relay", brain: "none" })`, dann `talk.session.appendAudio`, `talk.session.cancelTurn` und `talk.session.close` für Untertitel/Diktate ohne Sprachantwort eines Assistenten. Einmalig hochgeladene Sprachnachrichten verwenden weiterhin den Audiopfad der [Medienanalyse](/de/nodes/media-understanding).

Nativer Talk ist eine fortlaufende Schleife: auf Sprache warten, das Transkript über die aktive Sitzung an das Modell senden, auf die Antwort warten und sie anschließend über den konfigurierten Talk-Provider (`talk.speak`) ausgeben.

Clientseitig verwalteter Echtzeit-Talk leitet Tool-Aufrufe des Providers über `talk.client.toolCall` weiter, anstatt `chat.send` direkt aufzurufen. Während eine Echtzeitkonsultation aktiv ist, können Clients `talk.client.steer` oder `talk.session.steer` aufrufen, um Spracheingaben als `status`, `steer`, `cancel` oder `followup` zu klassifizieren. Akzeptierte Steuerungsanweisungen werden in die aktive eingebettete Ausführung eingereiht; bei abgelehnten Steuerungsanweisungen wird ein Grund wie `no_active_run`, `not_streaming` oder `compacting` zurückgegeben.

Abgeschlossene Echtzeitäußerungen von Benutzern und Assistenten werden stets unmittelbar an die aktive Agentensitzung angehängt, sodass spätere Chat- und Sprachinteraktionen einen gemeinsamen Verlauf verwenden. Clientseitig verwaltete Transporte melden ihre abgeschlossenen Transkripte mit stabilen Eintrags-IDs; Gateway-Relay-Sitzungen hängen dieselben Ereignisse serverseitig an. Providersitzungen erhalten außerdem den begrenzten Echtzeitprofilkontext, den Discord Voice verwendet.

Durch Sprache ausgelöste Konsultationsausführungen erfordern vor Aktionen mit erheblichen Auswirkungen eine neue, ausdrückliche mündliche Bestätigung. Dazu zählen das Senden von Nachrichten, die Steuerung von Nodes, Browser-/Computeraktionen, Dienständerungen, destruktive Shell-Befehle oder Veröffentlichungen. Die Bestätigung gilt nur für die exakten Argumente des blockierten Tools und wird einmalig verbraucht; unabhängige gleichzeitige Ausführungen bleiben unbeeinflusst. Wenn ein Anruf endet, kann OpenClaw eine kompakte Zusammenfassung der **Änderungen durch den Sprachanruf** für verändernde Tools an das letzte Nicht-WebChat-Zustellungsziel der Sitzung senden.

Talk nur für Transkription gibt denselben Talk-Ereignisumschlag wie Echtzeit- und STT/TTS-Sitzungen aus, verwendet jedoch `mode: "transcription"` und `brain: "none"`. Alle Talk-Sitzungen übertragen Ereignisse über den Kanal `talk.event`; Clients abonnieren ihn für Aktualisierungen vorläufiger/endgültiger Transkripte (`transcript.delta`/`transcript.done`) und andere Sitzungstelemetrie.

Browser-Video-Talk ist für OpenAI Realtime WebRTC und Google-Live-
Provider-WebSocket-Sitzungen verfügbar. OpenAI erhält ein einzelnes begrenztes JPEG, wenn
`describe_view` visuellen Kontext anfordert; es erhält keinen kontinuierlichen
Kamerastream. Google Live empfängt begrenzte JPEG-Frames direkt vom
Browser mit bis zu einem Frame pro Sekunde, während `describe_view` den
Status des Kamerastreams meldet. In beiden Fällen umgehen die Kameraframes das Gateway, und
beim Beenden von Talk werden die Kamera- und Mikrofonspuren freigegeben.

## Verhalten (macOS)

- Ständig sichtbares Overlay, solange der Talk-Modus aktiviert ist.
- Phasenübergänge **Zuhören &rarr; Nachdenken &rarr; Sprechen**.
- Bei einer kurzen Pause (Stillefenster) wird das aktuelle Transkript gesendet.
- Antworten werden in WebChat geschrieben (wie bei einer Texteingabe).
- **Unterbrechung bei Spracheingabe** (standardmäßig aktiviert): Wenn der Benutzer spricht, während der Assistent eine Antwort ausgibt, wird die Wiedergabe beendet und der Zeitpunkt der Unterbrechung für den nächsten Prompt vermerkt.

## Sprachanweisungen in Antworten

Der Assistent kann einer Antwort eine einzelne JSON-Zeile voranstellen, um die Stimme zu steuern:

```json
{ "voice": "<voice-id>", "once": true }
```

Regeln:

- Nur die erste nicht leere Zeile; die JSON-Zeile wird vor der TTS-Wiedergabe entfernt.
- Unbekannte Schlüssel werden ignoriert.
- `once: true` gilt nur für die aktuelle Antwort; ohne diesen Schlüssel wird die Stimme zum neuen Standard des Talk-Modus.

Unterstützte Schlüssel: `voice` / `voice_id` / `voiceId`, `model` / `model_id` / `modelId`, `speed`, `rate` (WPM), `stability`, `similarity`, `style`, `speakerBoost`, `seed`, `normalize`, `lang`, `output_format`, `latency_tier`, `once`.

## Konfiguration (`~/.openclaw/openclaw.json`)

```json5
{
  talk: {
    provider: "elevenlabs",
    providers: {
      elevenlabs: {
        voiceId: "elevenlabs_voice_id",
        modelId: "eleven_v3",
        outputFormat: "mp3_44100_128",
        apiKey: "elevenlabs_api_key",
      },
      mlx: {
        modelId: "mlx-community/Soprano-80M-bf16",
      },
      system: {},
    },
    speechLocale: "ru-RU",
    silenceTimeoutMs: 1500,
    interruptOnSpeech: true,
    realtime: {
      provider: "openai",
      providers: {
        openai: {
          apiKey: "openai_api_key",
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
        },
      },
      instructions: "Sprechen Sie freundlich und halten Sie die Antworten kurz.",
      mode: "realtime",
      transport: "webrtc",
      brain: "agent-consult",
    },
  },
}
```

| Schlüssel                                  | Standardwert                                | Hinweise                                                                                                                                                                                                                                                                   |
| ---------------------------------------- | ------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `provider`                               | -                                          | Aktiver TTS-Provider für Talk. Verwenden Sie `elevenlabs`, `mlx` oder `system` für lokale Wiedergabepfade unter macOS.                                                                                                                                                                             |
| `providers.<id>.voiceId`                 | -                                          | ElevenLabs greift auf `ELEVENLABS_VOICE_ID` / `SAG_VOICE_ID` oder die erste verfügbare Stimme mit einem API-Schlüssel zurück.                                                                                                                                                             |
| `speechLocale`                           | Gerätestandard                             | BCP-47-Gebietsschema für die native Spracherkennung unter Android, iOS und macOS. Apple Speech verwendet möglicherweise Netzwerkdienste; Android leitet außerdem die Sprachkomponente an die Echtzeittranskription der Eingabe weiter.                                                                                  |
| `providers.elevenlabs.modelId`           | `eleven_v3`                                |                                                                                                                                                                                                                                                                            |
| `providers.mlx.modelId`                  | `mlx-community/Soprano-80M-bf16`           |                                                                                                                                                                                                                                                                            |
| `providers.elevenlabs.apiKey`            | -                                          | Greift auf `ELEVENLABS_API_KEY` zurück (oder auf das Shell-Profil des Gateways, sofern verfügbar).                                                                                                                                                                                                |
| `silenceTimeoutMs`                       | `700` ms macOS/Android, `900` ms iOS       | Pausenfenster, bevor Talk das Transkript sendet.                                                                                                                                                                                                                             |
| `interruptOnSpeech`                      | `true`                                     |                                                                                                                                                                                                                                                                            |
| `outputFormat`                           | `pcm_44100` macOS/iOS, `pcm_24000` Android | Setzen Sie `mp3_*`, um MP3-Streaming zu erzwingen.                                                                                                                                                                                                                                        |
| `consultThinkingLevel`                   | nicht festgelegt                          | Überschreibt die Denkstufe für den Agentenlauf hinter Echtzeitaufrufen von `openclaw_agent_consult`.                                                                                                                                                                                  |
| `consultFastMode`                        | nicht festgelegt                          | Überschreibt den Schnellmodus für Echtzeitaufrufe von `openclaw_agent_consult`.                                                                                                                                                                                                            |
| `realtime.provider`                      | -                                          | `openai` für WebRTC, `google` für den Provider-WebSocket oder ein ausschließlich über eine Bridge angebundener Provider über das Gateway-Relay.                                                                                                                                                                     |
| `realtime.providers.<id>`                | -                                          | Provider-eigene Echtzeitkonfiguration. Browser erhalten ausschließlich kurzlebige/eingeschränkte Sitzungsanmeldedaten, niemals einen regulären API-Schlüssel.                                                                                                                                                 |
| `realtime.providers.openai.speakerVoice` | `alloy`                                    | Integrierte OpenAI-Realtime-Stimm-ID (der ältere Schlüssel `voice` funktioniert weiterhin, ist aber veraltet). Aktuelle Stimmen für `gpt-realtime-2.1`: `alloy`, `ash`, `ballad`, `cedar`, `coral`, `echo`, `marin`, `sage`, `shimmer`, `verse`; `marin` und `cedar` werden für die beste Qualität empfohlen. |
| `realtime.transport`                     | -                                          | `webrtc`: clientseitig verwaltetes OpenAI WebRTC unter iOS und im Browser. `provider-websocket`: browserseitig verwaltet; verbleibt unter iOS auf dem Gateway-Relay. `gateway-relay`: belässt das Provider-Audio auf dem Gateway; Android verwendet Echtzeit nur mit diesem Transport.                                  |
| `realtime.brain`                         | -                                          | `agent-consult` leitet Echtzeit-Tool-Aufrufe über die Gateway-Richtlinie weiter; `direct-tools` dient der Abwärtskompatibilität für direkte Tool-Aufrufe; `none` ist für Transkription/externe Orchestrierung vorgesehen.                                                                                                 |
| `realtime.consultRouting`                | -                                          | `provider-direct` behält die direkte Antwort des Providers bei, wenn dieser `openclaw_agent_consult` überspringt; `force-agent-consult` leitet stattdessen abgeschlossene Benutzertranskripte durch OpenClaw.                                                                                          |
| `realtime.instructions`                  | -                                          | Hängt Provider-seitige Systemanweisungen an den integrierten Echtzeit-Prompt von OpenClaw an (Sprachstil/Tonfall); die standardmäßige Anleitung `openclaw_agent_consult` bleibt erhalten.                                                                                                                |

`talk.catalog` stellt kanonische Provider-IDs und Registry-Aliasse, die gültigen Modi/Transporte/Brain-Strategien/Echtzeit-Audioformate/Fähigkeitskennzeichen jedes Providers sowie das zur Laufzeit ausgewählte Bereitschaftsergebnis bereit. Erstanbieter-Talk-Clients sollten diesen Katalog lesen, anstatt Provider-Aliasse lokal zu verwalten; behandeln Sie ein älteres Gateway, das die Gruppenbereitschaft auslässt, als ungeprüft und nicht als definitiv unkonfiguriert. Streaming-Transkriptions-Provider werden über `talk.catalog.transcription` erkannt; das aktuelle Gateway-Relay verwendet die Konfiguration des Voice-Call-Streaming-Providers, bis eine dedizierte Konfigurationsoberfläche für die Talk-Transkription verfügbar ist.

## macOS-Benutzeroberfläche

- Umschalter in der Menüleiste: **Talk**
- Konfigurationsregisterkarte: Gruppe **Talk-Modus** (Stimm-ID + Unterbrechungsumschalter)
- Overlay: Die Kugel zeigt die universelle Talk-Wellenform an (gemeinsam mit iOS, watchOS und Android). Beim Zuhören folgt sie dem Live-Mikrofonpegel, beim Sprechen der tatsächlichen Hüllkurve der TTS-Wiedergabe und beim Denken pulsiert sie sanft. Klicken Sie auf die Kugel, um zu pausieren/fortzufahren, doppelklicken Sie, um die Sprachausgabe zu beenden, und klicken Sie auf X, um den Talk-Modus zu verlassen.

## Android-Benutzeroberfläche

- Die Hauptnavigation von Android umfasst **Startseite**, **Chat** und **Einstellungen**. Die Spracheingabe
  befindet sich im Chat-Eingabefeld statt auf einer separaten Sprachregisterkarte.
- Tippen Sie für die geräteinterne Diktierfunktion auf das Mikrofon im Eingabefeld. Halten Sie es gedrückt, um
  eine Sprachnotiz als Anhang aufzunehmen. Starten Sie kontinuierliches Talk über die Talk-Wellenform.
- Diktieren, Sprachnotizaufnahme und Talk schließen sich als Mikrofonpfade
  gegenseitig aus; das Starten eines Pfads beendet oder blockiert die anderen.
- Echtzeit-Talk bevorzugt das Mikrofon eines verbundenen Bluetooth-Classic- oder BLE-Headsets.
  Wird die Verbindung getrennt, fordert die App einen anderen Headset-Eingang an oder
  greift auf das Standardmikrofon zurück und stellt die Standardeinstellung wieder her, sobald
  die Aufnahme endet.
- Diktieren und Sprachnotizaufnahme werden beendet, wenn die App den Vordergrund verlässt oder
  der Benutzer den Chat verlässt.
- Der Talk-Modus läuft weiter, bis er ausgeschaltet oder die Verbindung zum Node getrennt wird, und verwendet währenddessen den Mikrofon-Typ des Android-Vordergrunddienstes.
- Android unterstützt die Ausgabeformate `pcm_16000`, `pcm_22050`, `pcm_24000` und `pcm_44100` für latenzarmes `AudioTrack`-Streaming.

## Hinweise

- Erfordert Berechtigungen für Spracherkennung und Mikrofon.
- Natives Talk verwendet die aktive Gateway-Sitzung und greift nur dann auf die Abfrage des Verlaufs zurück, wenn keine Antwortereignisse verfügbar sind.
- Das Gateway löst die Talk-Wiedergabe über `talk.speak` mithilfe des aktiven Talk-Providers auf. Android greift nur dann auf die lokale System-TTS zurück, wenn dieser RPC nicht verfügbar ist.
- Die lokale MLX-Wiedergabe unter macOS verwendet den gebündelten Helfer `openclaw-mlx-tts`, sofern vorhanden, oder eine ausführbare Datei unter `PATH`. Setzen Sie `OPENCLAW_MLX_TTS_BIN`, um während der Entwicklung auf eine benutzerdefinierte ausführbare Helferdatei zu verweisen.
- Wertebereiche für Sprachanweisungen (ElevenLabs): `stability`, `similarity` und `style` akzeptieren `0..1`; `speed` akzeptiert `0.5..2`; `latency_tier` akzeptiert `0..4`.

## Verwandte Themen

- [Sprachaktivierung](/de/nodes/voicewake)
- [Audio und Sprachnotizen](/de/nodes/audio)
- [Medienverständnis](/de/nodes/media-understanding)
