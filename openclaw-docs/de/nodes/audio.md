---
read_when:
    - Ändern der Audiotranskription oder Medienverarbeitung
summary: Wie eingehende Audio-/Sprachnachrichten heruntergeladen, transkribiert und in Antworten eingefügt werden
title: Audio- und Sprachnachrichten
x-i18n:
    generated_at: "2026-07-26T17:54:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4076e3e55eb5c6dcc94cfdd842619697c8d756b924956d7b266d18446b4dd9be
    source_path: nodes/audio.md
    workflow: 16
---

## Funktionsweise

Wenn die Audioerkennung aktiviert (oder automatisch erkannt) ist, führt OpenClaw Folgendes aus:

1. Sucht den ersten Audioanhang (lokaler Pfad oder URL) und lädt ihn bei Bedarf herunter.
2. Erzwingt `maxBytes`, bevor der Anhang an jeden Modelleintrag gesendet wird.
3. Führt den ersten geeigneten Modelleintrag der Reihe nach aus (Provider oder CLI); wenn ein Eintrag fehlschlägt oder übersprungen wird (Größe/Zeitüberschreitung), wird der nächste Eintrag versucht.
4. Ersetzt bei Erfolg `Body` durch einen `[Audio]`-Block und setzt `{{Transcript}}`.

Wenn die Transkription erfolgreich ist, werden `CommandBody`/`RawBody` ebenfalls auf das Transkript gesetzt, damit Slash-Befehle weiterhin funktionieren. Mit `--verbose` zeigen die Protokolle, wann die Transkription ausgeführt wird und wann sie den Nachrichtentext ersetzt.

## Automatische Erkennung (Standard)

Wenn Sie keine Modelle konfiguriert haben und `tools.media.audio.enabled` nicht `false` ist, führt OpenClaw die automatische Erkennung in der folgenden Reihenfolge durch und stoppt bei der ersten funktionierenden Option:

1. **Aktives Antwortmodell**, wenn dessen Provider Audioerkennung unterstützt.
2. **Konfigurierte Provider-Authentifizierung** — jeder `models.providers.*`-Eintrag mit verfügbarer Authentifizierung für einen Provider, der Audiotranskription unterstützt. Dies wird vor lokalen CLIs geprüft, sodass ein konfigurierter API-Schlüssel stets Vorrang vor einem lokalen Programm in `PATH` hat.
   Provider-Priorität, wenn mehrere konfiguriert sind: Groq, OpenAI, xAI, Deepgram, Google, SenseAudio, ElevenLabs, Mistral.
3. **Lokale CLIs** (nur wenn keine Provider-Authentifizierung ermittelt wurde). OpenClaw erstellt eine geordnete Fallback-Liste:
   - `whisper-cli`, vor CPU-Standardeinstellungen nur dann, wenn bei einem früheren Modellaufruf im aktuellen Prozess Metal oder CUDA festgestellt wurde
   - `sherpa-onnx-offline` mit seinem standardmäßigen CPU-Provider (erfordert `SHERPA_ONNX_MODEL_DIR` mit `tokens.txt`, `encoder.onnx`, `decoder.onnx` und `joiner.onnx`)
   - `whisper-cli`, wenn Metal/CUDA lediglich beim Build unterstützt wird oder das ausgewählte Backend anderweitig noch nicht beobachtet wurde
   - `parakeet-mlx` auf Apple Silicon (MLX-fähig; die Gerätenutzung bleibt unbeobachtet)
   - `whisper` (Python-CLI; lädt Modelle automatisch herunter)

Die Herkunft der Installation bzw. Verknüpfung ist ein Nachweis der Fähigkeit, nicht der Ausführung. Sie verschafft einem Kandidaten für sich allein niemals Vorrang vor CPU-Sherpa. OpenClaw lädt während der Einrichtung oder bei Statusprüfungen kein Modell, nur um ein Backend zu testen.
Das automatisch erkannte whisper.cpp behält seine normalen Modelllauf-Protokolle bei, damit OpenClaw die vorgelagerte Zeile `using … backend` erfassen kann. Explizite CLI-Einträge behalten ihre konfigurierten Ausgabe-Flags bei.

Die automatische Erkennung der Gemini CLI zur Medienerkennung wurde für Bilder und Videos durch einen Sandbox-basierten Fallback auf die Antigravity CLI (`agy`) ersetzt; Audio verwendet außer den oben aufgeführten lokalen Programmen keinen CLI-Fallback.

Um die automatische Erkennung zu deaktivieren, setzen Sie `tools.media.audio.enabled: false`. Zur Anpassung fügen Sie `tools.media.models` Einträge mit Fähigkeits-Tags hinzu.

<Note>
Die Erkennung von Programmen erfolgt unter macOS/Linux/Windows nach bestem Bemühen. Stellen Sie sicher, dass sich die CLI in `PATH` befindet (`~` wird aufgelöst), oder legen Sie ein explizites CLI-Modell mit dem vollständigen Befehlspfad fest.
</Note>

Prüfen Sie die lokale Auswahl, ohne Audio zu transkribieren:

```bash
openclaw capability audio providers
openclaw doctor --lint --only core/doctor/local-audio-acceleration --severity-min info
```

Das Provider-Inventar meldet den Gewinner des lokalen Fallbacks getrennt von der globalen Provider-Auswahl sowie Felder für fähige, angeforderte und beobachtete Backends. Nach einer Transkription meldet `/status` in der Medienzeile das angeforderte oder beobachtete Backend. Explizite audiofähige `tools.media.models`-CLI-Einträge umgehen weiterhin die automatische Auswahl; verwenden Sie deren Backend-spezifische Flags wie Sherpa `--provider=cuda` oder whisper.cpp `--no-gpu`/`--device`.

## Konfigurationsbeispiele

### Provider- und CLI-Fallback (OpenAI und Whisper CLI)

```json5
{
  tools: {
    media: {
      models: [
        { provider: "openai", model: "gpt-4o-transcribe", capabilities: ["audio"] },
        {
          type: "cli",
          command: "whisper",
          args: ["--model", "base", "{{AttachmentPath}}"],
          timeoutSeconds: 45,
          capabilities: ["audio"],
        },
      ],
      audio: { enabled: true, preferredModel: "openai/gpt-4o-transcribe" },
    },
  },
}
```

### Nur Provider (Deepgram)

```json5
{
  tools: {
    media: {
      models: [{ provider: "deepgram", model: "nova-3", capabilities: ["audio"] }],
      audio: { enabled: true },
    },
  },
}
```

### Nur Provider (Mistral Voxtral)

```json5
{
  tools: {
    media: {
      models: [{ provider: "mistral", model: "voxtral-mini-latest", capabilities: ["audio"] }],
      audio: { enabled: true },
    },
  },
}
```

### Nur Provider (SenseAudio)

```json5
{
  tools: {
    media: {
      models: [
        {
          provider: "senseaudio",
          model: "senseaudio-asr-pro-1.5-260319",
          capabilities: ["audio"],
        },
      ],
      audio: { enabled: true },
    },
  },
}
```

### Transkript im Chat wiedergeben (Opt-in)

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        echoTranscript: true,
        echoFormat: '📝 "{transcript}"',
      },
    },
  },
}
```

## Hinweise und Einschränkungen

- Die Provider-Authentifizierung folgt der standardmäßigen Reihenfolge der Modellauthentifizierung (Authentifizierungsprofile, Umgebungsvariablen, `models.providers.*.apiKey`).
- Details zur Einrichtung von Groq: [Groq](/de/providers/groq).
- Deepgram übernimmt `DEEPGRAM_API_KEY`, wenn `provider: "deepgram"` verwendet wird. Details zur Einrichtung: [Deepgram](/de/providers/deepgram).
- Details zur Einrichtung von Mistral: [Mistral](/de/providers/mistral).
- SenseAudio übernimmt `SENSEAUDIO_API_KEY`, wenn `provider: "senseaudio"` verwendet wird. Details zur Einrichtung: [SenseAudio](/de/providers/senseaudio).
- Audio-Provider können die Standardeinstellungen unter `tools.media.audio` verwenden oder `baseUrl`, `headers`, `providerOptions` sowie Grenzwerte in ihrem `tools.media.models[]`-Eintrag überschreiben.
- Die integrierte Größenbegrenzung für Audio beträgt 20MB. Eine Überschreibung über `maxBytes` auf Eintragsebene kann sie ändern; zu große Audiodaten werden für dieses Modell übersprungen und der nächste Eintrag wird versucht.
- Audiodateien unter 1024 Byte werden vor der Transkription durch den Provider bzw. die CLI übersprungen.
- Der standardmäßige Wert für `maxChars` bei Audio ist **nicht festgelegt** (vollständiges Transkript). Legen Sie `tools.media.audio.maxChars` oder eintragsspezifisch `maxChars` fest, um die Ausgabe zu kürzen.
- Der Standardwert der automatischen OpenAI-Erkennung ist `gpt-4o-transcribe`; legen Sie `model: "gpt-4o-mini-transcribe"` als kostengünstigere/schnellere Option fest.
- Das Transkript steht Vorlagen als `{{Transcript}}` zur Verfügung.
- `tools.media.audio.echoTranscript` ist standardmäßig deaktiviert; `echoFormat` akzeptiert einen `{transcript}`-Platzhalter.
- Die Standardausgabe der CLI ist auf 5MB begrenzt; halten Sie die CLI-Ausgabe knapp.
- CLI-`args` sollte `{{AttachmentPath}}` für den lokalen Pfad der Audiodatei verwenden. Führen Sie `openclaw doctor --fix` aus, um veraltete `{input}`-Platzhalter aus älteren `audio.transcription.command`-Konfigurationen zu migrieren (eingestellter Schlüssel: `audio.transcription`, ersetzt durch `tools.media.models`). `{{MediaPath}}` bleibt als veralteter Kompatibilitätsalias erhalten.
- `tools.media.concurrency` begrenzt Medienaufgaben; es ist kein GPU-Scheduler.

### Dauerhaft laufende lokale STT

Automatisch erkannte lokale STT bleibt ein Prozess pro Anfrage. OpenClaw verwaltet derzeit keinen dauerhaft laufenden whisper.cpp-Server, da das standardmäßige Homebrew-Paket `whisper-cpp` diesen Server deaktiviert und das vorgelagerte Beispiel keine konfigurierte begrenzte Annahmewarteschlange besitzt. Für einen Plugin-eigenen dauerhaften Lebenszyklus ist ein gepflegter, paketierter Worker mit Zustands-/Startprüfung, Modellresidenz, begrenzter Warteschlange, Abbruch/Zeitüberschreitung, ausschließlich lokalem Betrieb ohne Authentifizierung und ohne Cloud-Fallback erforderlich, bevor er sicher aktiviert werden kann.

### Unterstützung für Proxy-Umgebungen

Provider-basierte Audiotranskription berücksichtigt standardmäßige Umgebungsvariablen für ausgehende Proxys entsprechend der Semantik von `EnvHttpProxyAgent` in undici:

- `HTTPS_PROXY` / `https_proxy`
- `HTTP_PROXY` / `http_proxy`
- `ALL_PROXY` / `all_proxy`

Kleingeschriebene Variablen haben Vorrang vor großgeschriebenen; Einträge in `NO_PROXY`/`no_proxy` (Hostnamen, `*.suffix` oder `host:port`) umgehen den Proxy. Wenn keine Proxy-Umgebungsvariablen gesetzt sind, wird eine direkte ausgehende Verbindung verwendet. Wenn die Proxy-Einrichtung fehlschlägt (fehlerhafte URL), protokolliert OpenClaw eine Warnung und greift auf direkten Abruf zurück.

## Erkennung von Erwähnungen in Gruppen

Auf Kanälen, die Audio-Preflight unterstützen, transkribiert OpenClaw Audio **vor** der Prüfung auf Erwähnungen, wenn `requireMention: true` für einen Gruppenchat gesetzt ist. Dadurch kann eine Sprachnachricht ohne Beschriftung die Erwähnungsprüfung passieren, wenn ihr Transkript ein konfiguriertes Erwähnungsmuster enthält. Kanalspezifische Dokumentationen beschreiben Übertragungswege, die stattdessen eine eingegebene Erwähnung erfordern.

**Funktionsweise:**

1. Wenn eine Sprachnachricht keinen Text enthält und die Gruppe Erwähnungen erfordert, führt OpenClaw eine Preflight-Transkription des ersten Audioanhangs durch.
2. Das Transkript wird auf Erwähnungsmuster geprüft (zum Beispiel `@BotName`, Emoji-Auslöser).
3. Wenn eine Erwähnung gefunden wird, durchläuft die Nachricht die vollständige Antwort-Pipeline.

**Fallback-Verhalten:** Wenn die Preflight-Transkription fehlschlägt (Zeitüberschreitung, API-Fehler usw.), greift die Nachricht auf die rein textbasierte Erkennung von Erwähnungen zurück, sodass gemischte Nachrichten (Text + Audio) niemals verworfen werden.

**Opt-out pro Telegram-Gruppe/-Thema:**

- Setzen Sie `channels.telegram.groups.<chatId>.disableAudioPreflight: true`, um Preflight-Prüfungen des Transkripts auf Erwähnungen für diese Gruppe zu überspringen.
- Setzen Sie `channels.telegram.groups.<chatId>.topics.<threadId>.disableAudioPreflight`, um dies themenspezifisch zu überschreiben (`true` zum Überspringen, `false` zum Erzwingen der Aktivierung).
- Der Standardwert ist `false` (Preflight ist aktiviert, wenn die Bedingungen für eine erforderliche Erwähnung erfüllt sind).

**Beispiel:** Ein Benutzer sendet in einer Telegram-Gruppe mit `requireMention: true` eine Sprachnachricht mit dem Inhalt „Hey @Claude, wie ist das Wetter?“. Die Sprachnachricht wird transkribiert, die Erwähnung erkannt und der Agent antwortet.

## Fallstricke

- Für Bereichsregeln gilt der erste Treffer; `chatType` wird zu `direct`, `group` oder `channel` normalisiert.
- Stellen Sie sicher, dass Ihre CLI mit 0 beendet wird und Klartext ausgibt; JSON-Ausgaben müssen über `jq -r .text` aufbereitet werden.
- Bekannte Dateiausgabemodi sind maßgeblich: Eine leere oder fehlende abgeleitete Transkriptdatei führt zu keinem Transkript, statt auf die Fortschrittsausgabe der CLI zurückzugreifen.
- Verwenden Sie für `parakeet-mlx` `--output-format txt` (oder `all`) mit `--output-dir` und der standardmäßigen Ausgabevorlage `{filename}`. Die vorgelagerten Umgebungsvariablen `PARAKEET_OUTPUT_FORMAT` und `PARAKEET_OUTPUT_TEMPLATE` werden ebenfalls berücksichtigt. OpenClaw liest `<output-dir>/<media-basename>.txt`; beim standardmäßigen Format `srt`, bei anderen Formaten und bei benutzerdefinierten Ausgabevorlagen wird weiterhin die Standardausgabe verwendet.
- Verwenden Sie angemessene Zeitüberschreitungen (`timeoutSeconds`, standardmäßig 60s), um ein Blockieren der Antwortwarteschlange zu vermeiden.
- Die Preflight-Transkription verarbeitet zur Erkennung von Erwähnungen nur den **ersten** Audioanhang. Weitere Audioanhänge werden während der Hauptphase der Medienerkennung verarbeitet.

## Verwandte Themen

- [Medienerkennung](/de/nodes/media-understanding)
- [Sprechmodus](/de/nodes/talk)
- [Sprachaktivierung](/de/nodes/voicewake)
