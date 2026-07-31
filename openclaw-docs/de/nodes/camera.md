---
read_when:
    - Hinzufügen oder Ändern der Kameraaufnahme auf Node-Plattformen
    - Erweiterung der für Agenten zugänglichen MEDIA-Workflows für temporäre Dateien
summary: Kameraaufnahme auf iOS-, Android-, macOS- und Linux-Nodes für Fotos und kurze Videoclips
title: Kameraaufnahme
x-i18n:
    generated_at: "2026-07-26T17:52:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b819f7ff3fc9b51757ae998d27f540975bf6c1194ed32fd36b1fbe909e79400c
    source_path: nodes/camera.md
    workflow: 16
---

OpenClaw unterstützt die Kameraaufnahme für Agent-Workflows auf gekoppelten **iOS**-, **Android**-, **macOS**- und **Linux**-Nodes: Nehmen Sie über den Gateway `node.invoke` ein Foto (`jpg`) oder einen kurzen Videoclip (`mp4`, optional mit Audio) auf.

Der gesamte Kamerazugriff wird auf jeder Plattform durch eine benutzergesteuerte Einstellung geregelt.

## iOS-Node

### iOS-Benutzereinstellung

- iOS Settings tab → **Camera** → **Allow Camera** (`camera.enabled`).
  - Standard: **ein** (ein fehlender Schlüssel wird als aktiviert behandelt).
  - Wenn deaktiviert: `camera.*`-Befehle geben `CAMERA_DISABLED` zurück.

### iOS-Befehle (über Gateway `node.invoke`)

- `camera.list`
  - Antwortnutzlast: `devices` – Array von `{ id, name, position, deviceType }`.

- `camera.snap`
  - Parameter:
    - `facing`: `front|back` (Standard: `front`)
    - `maxWidth`: Zahl (optional; Standard: `1600`)
    - `quality`: `0..1` (optional; Standard: `0.9`, begrenzt auf `[0.05, 1.0]`)
    - `format`: derzeit `jpg`
    - `delayMs`: Zahl (optional; Standard: `0`, intern auf `10000` begrenzt)
    - `deviceId`: Zeichenfolge (optional; aus `camera.list`)
  - Antwortnutzlast: `format: "jpg"`, `base64`, `width`, `height`.
  - Nutzlastbegrenzung: Fotos werden erneut komprimiert, damit die Base64-codierte Nutzlast unter 5MB bleibt.

- `camera.clip`
  - Parameter:
    - `facing`: `front|back` (Standard: `front`)
    - `durationMs`: Zahl (Standard: `3000`, begrenzt auf `[250, 60000]`)
    - `includeAudio`: boolescher Wert (Standard: `true`)
    - `format`: derzeit `mp4`
    - `deviceId`: Zeichenfolge (optional; aus `camera.list`)
  - Antwortnutzlast: `format: "mp4"`, `base64`, `durationMs`, `hasAudio`.

### iOS-Vordergrundanforderung

Wie `canvas.*` erlaubt der iOS-Node `camera.*`-Befehle nur im **Vordergrund**. Aufrufe im Hintergrund geben `NODE_BACKGROUND_UNAVAILABLE` zurück.

### CLI-Hilfsfunktion

Am einfachsten erhalten Sie Mediendateien über die CLI-Hilfsfunktion. Sie schreibt die decodierten Medien in eine temporäre Datei und gibt den Speicherpfad aus.

```bash
openclaw nodes camera snap --node <id>                 # Standard: Vorder- und Rückkamera (2 MEDIA-Zeilen)
openclaw nodes camera snap --node <id> --facing front
openclaw nodes camera clip --node <id> --duration 3000
openclaw nodes camera clip --node <id> --no-audio
```

`nodes camera snap` verwendet standardmäßig `--facing both` und erfasst sowohl die Vorder- als auch die Rückansicht, damit dem Agent beide Ansichten zur Verfügung stehen. Übergeben Sie `--device-id` mit genau einer expliziten Ausrichtung (`both` wird abgelehnt, wenn `--device-id` festgelegt ist). Ausgabedateien sind temporär (im temporären Verzeichnis des Betriebssystems), sofern Sie keinen eigenen Wrapper erstellen.

## Android-Node

### Android-Benutzereinstellung

- Android Settings sheet → **Camera** → **Allow Camera** (`camera.enabled`).
  - **Bei Neuinstallationen ist die Einstellung standardmäßig deaktiviert.** Bestehende Installationen, die älter als diese Einstellung sind, werden auf **ein** migriert, damit bei Upgrades ein zuvor funktionierender Kamerazugriff nicht unbemerkt verloren geht.
  - Wenn deaktiviert: `camera.*`-Befehle geben `CAMERA_DISABLED: enable Camera in Settings` zurück.

### Berechtigungen

- `CAMERA` ist sowohl für `camera.snap` als auch für `camera.clip` erforderlich; eine fehlende oder verweigerte Berechtigung gibt `CAMERA_PERMISSION_REQUIRED` zurück.
- `RECORD_AUDIO` ist für `camera.clip` erforderlich, wenn `includeAudio` den Wert `true` hat; eine fehlende oder verweigerte Berechtigung gibt `MIC_PERMISSION_REQUIRED` zurück.

Die App fordert nach Möglichkeit Laufzeitberechtigungen an.

### Android-Vordergrundanforderung

Wie `canvas.*` erlaubt der Android-Node `camera.*`-Befehle nur im **Vordergrund**. Aufrufe im Hintergrund geben `NODE_BACKGROUND_UNAVAILABLE: command requires foreground` zurück.

### Android-Befehle (über Gateway `node.invoke`)

- `camera.list`
  - Antwortnutzlast: `devices` – Array von `{ id, name, position, deviceType }`.

- `camera.snap`
  - Parameter: `facing` (`front|back`, Standard: `front`), `quality` (Standard: `0.95`, begrenzt auf `[0.1, 1.0]`), `maxWidth` (Standard: `1600`), `deviceId` (optional; eine unbekannte ID schlägt mit `INVALID_REQUEST` fehl).
  - Antwortnutzlast: `format: "jpg"`, `base64`, `width`, `height`.
  - Nutzlastbegrenzung: erneute Komprimierung, damit Base64 unter 5MB bleibt (dasselbe Budget wie bei iOS).

- `camera.clip`
  - Parameter: `facing` (Standard: `front`), `durationMs` (Standard: `3000`, begrenzt auf `[200, 60000]`), `includeAudio` (Standard: `true`), `deviceId` (optional).
  - Antwortnutzlast: `format: "mp4"`, `base64`, `durationMs`, `hasAudio`.
  - Nutzlastbegrenzung: Die rohe MP4-Datei ist vor der Base64-Codierung auf 18MB begrenzt; zu große Clips schlagen mit `PAYLOAD_TOO_LARGE` fehl (reduzieren Sie `durationMs` und versuchen Sie es erneut).

## macOS-App

### macOS-Benutzereinstellung

Die macOS-Begleit-App stellt ein Kontrollkästchen bereit:

- **Settings → General → Allow Camera** (`openclaw.cameraEnabled`).
  - Standard: **aus**.
  - Wenn deaktiviert: Kameraanfragen geben `CAMERA_DISABLED: enable Camera in Settings` zurück.

### CLI-Hilfsfunktion (Node-Aufruf)

Verwenden Sie die zentrale `openclaw`-CLI, um Kamerabefehle auf dem macOS-Node aufzurufen.

```bash
openclaw nodes camera list --node <id>                     # Kamera-IDs auflisten
openclaw nodes camera snap --node <id>                     # gibt den Speicherpfad aus
openclaw nodes camera snap --node <id> --max-width 1280
openclaw nodes camera snap --node <id> --delay-ms 2000
openclaw nodes camera snap --node <id> --device-id <id>
openclaw nodes camera clip --node <id> --duration 10s       # gibt den Speicherpfad aus
openclaw nodes camera clip --node <id> --duration-ms 3000   # gibt den Speicherpfad aus (veraltetes Flag)
openclaw nodes camera clip --node <id> --device-id <id>
openclaw nodes camera clip --node <id> --no-audio
```

- `openclaw nodes camera snap` verwendet standardmäßig `maxWidth=1600`, sofern dies nicht überschrieben wird.
- `camera.snap` wartet nach dem Aufwärmen und Einregeln der Belichtung `delayMs` (Standard: 2000ms, begrenzt auf `[0, 10000]`), bevor die Aufnahme erfolgt.
- Fotonutzlasten werden erneut komprimiert, damit Base64 unter 5MB bleibt.

## Linux-Node-Host

Das mitgelieferte Linux-Node-Plugin fügt dem CLI-Dienst `openclaw node` die Kameraaufnahme hinzu. Es funktioniert auf einem Headless-Host und erfordert nicht die Linux-Desktop-App.

Der Kamerazugriff ist standardmäßig deaktiviert. Aktivieren Sie ihn unter dem Plugin-Eintrag und starten Sie anschließend den Node-Dienst neu, damit seine Gateway-Ankündigung neu erstellt wird:

```json5
{
  plugins: {
    entries: {
      "linux-node": {
        config: {
          camera: { enabled: true },
        },
      },
    },
  },
}
```

Anforderungen:

- FFmpeg mit V4L2-Eingabe, `libx264` und AAC-Unterstützung
- ein `/dev/video*`-Gerät, das für den Benutzer des Node-Dienstes lesbar ist; fügen Sie diesen Benutzer bei gängigen Distributionen der Gruppe `video` hinzu
- für Clips mit dem standardmäßigen `includeAudio: true` einen funktionierenden PulseAudio-Server oder eine PipeWire-PulseAudio-Kompatibilitätsschicht mit einer Standardquelle

Linux gibt über `camera.list` aufnahmefähige, lesbare V4L2-Gerätepfade zurück; FFmpeg prüft jeden `/dev/video*`-Kandidaten und lässt Metadaten- oder reine Ausgabegeräte aus. Das Geräteattribut `position` lautet `unknown`, sodass Ausrichtungsanfragen ohne `deviceId` ein Foto oder einen Clip mit der Position `unknown` erzeugen, anstatt eine Vorder- oder Rückkamera auszuweisen. Verwenden Sie `deviceId`, wenn ein Host über mehrere Kameras verfügt. `camera.snap` verwendet für `delayMs` die FFmpeg-Eingabeaufwärmung und behält das Seitenverhältnis bei, während die Breite begrenzt wird. `camera.clip` zeichnet Mikrofonaudio als MP4-Audiospur auf; OpenClaw stellt bewusst keinen eigenständigen Mikrofonbefehl bereit.

Das Plugin verwendet `libx264` für MP4-Video und wechselt nicht stillschweigend den Codec. Ein FFmpeg-Build ohne die erforderliche Eingabe oder die erforderlichen Encoder gibt `CAMERA_UNAVAILABLE` zurück. Fotos und Clips, die das Base64-Nutzlastbudget von 25MB überschreiten würden, schlagen mit `PAYLOAD_TOO_LARGE` fehl.

`camera.snap` und `camera.clip` bleiben gefährliche Befehle. Fügen Sie sie nur dann zu `gateway.nodes.commands.allow` hinzu, wenn Sie die Aufnahme bewusst aktivieren möchten; allein durch das Aktivieren des Plugins wird die Gateway-Richtlinie nicht umgangen.

## Sicherheit und praktische Grenzen

- Der Kamera- und Mikrofonzugriff löst die üblichen Berechtigungsabfragen des Betriebssystems aus (und erfordert Verwendungsbeschreibungen in `Info.plist`).
- Videoclips sind auf 60s begrenzt, um übergroße Node-Nutzlasten zu vermeiden (Base64-Overhead plus Nachrichtengrenzen).

## macOS-Bildschirmvideo (auf Betriebssystemebene)

Verwenden Sie für _Bildschirmvideos_ (nicht für Kameraaufnahmen) die macOS-Begleit-App:

```bash
openclaw nodes screen record --node <id> --duration 10s --fps 15   # gibt den Speicherpfad aus
```

Erfordert die macOS-Berechtigung **Screen Recording** (TCC).

## Verwandte Themen

- [Bild- und Medienunterstützung](/de/nodes/images)
- [Medienverständnis](/de/nodes/media-understanding)
- [Standortbefehl](/de/nodes/location-command)
