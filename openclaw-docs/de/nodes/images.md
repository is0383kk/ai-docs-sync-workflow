---
read_when:
    - Medien-Pipeline oder Anhänge ändern
summary: Regeln für die Bild- und Medienverarbeitung beim Senden sowie in Gateway- und Agent-Antworten
title: Unterstützung für Bilder und Medien
x-i18n:
    generated_at: "2026-07-26T18:26:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 71f5591f4268593c142056370802b702899787a79f9ca1fbde6ea8e422f34023
    source_path: nodes/images.md
    workflow: 16
---

Der WhatsApp-Kanal läuft auf Baileys Web. Diese Seite behandelt die Regeln für die Medienverarbeitung beim Senden, im Gateway und bei Agentenantworten.

## Ziele

- Medien mit einer optionalen Bildunterschrift über `openclaw message send --media` senden.
- Automatische Antworten aus dem Web-Posteingang dürfen Medien zusammen mit Text enthalten.
- Sinnvolle und vorhersehbare Grenzwerte je Medientyp beibehalten.

## CLI-Oberfläche

`openclaw message send --target <dest> --media <path-or-url> [--message <caption>]`

- `--media <path-or-url>` — Medien anhängen (Bild/Audio/Video/Dokument); akzeptiert lokale Pfade oder URLs. Optional; bei Sendungen, die nur Medien enthalten, kann die Bildunterschrift leer sein.
- `--gif-playback` — Videomedien als GIF-Wiedergabe behandeln (nur WhatsApp).
- `--force-document` — Medien als Dokument senden, um die Komprimierung durch den Kanal zu vermeiden (Telegram, WhatsApp); gilt für Bilder, GIFs und Videos.
- `--reply-to <id>`, `--thread-id <id>`, `--pin`, `--silent` — Zustellungs-/Threading-Optionen, die auch für reine Textsendungen gelten.
- `--dry-run` — die aufgelöste Nutzlast ausgeben und das Senden überspringen.
- `--json` — das Ergebnis als JSON ausgeben: `{ action, channel, dryRun, handledBy, messageId?, payload }` (`payload` enthält das kanalspezifische Sendeergebnis einschließlich etwaiger Medienreferenzen).

## Verhalten des WhatsApp-Web-Kanals

- Eingabe: lokaler Dateipfad **oder** HTTP(S)-URL.
- Ablauf: in einen Puffer laden, den Medientyp erkennen und anschließend die ausgehende Nutzlast entsprechend dem Typ erstellen:
  - **Bilder:** werden so optimiert, dass sie unter `channels.whatsapp.mediaMaxMb` passen (standardmäßig 50MB). Undurchsichtige Bilder werden erneut als JPEG komprimiert (die Standardstaffel für Seitenlängen beginnt bei 2048px und wird bei wiederholter Überschreitung der Größe schrittweise reduziert); Bilder mit Transparenz bleiben PNG. Wenn die Quelle bereits ein zulässiges JPEG/PNG/WebP innerhalb des Größen- und Seitenlängenlimits ist, werden die ursprünglichen Bytes unverändert beibehalten und nicht erneut komprimiert. Animierte GIFs werden niemals neu codiert, sondern nur auf ihre Größe geprüft.
  - **Audio/Sprache:** Sofern es sich nicht bereits um natives Sprachaudio handelt (`.ogg`/`.opus` oder `audio/ogg`/`audio/opus`), wird ausgehendes Audio vor dem Senden als Sprachnachricht (`ptt: true`) über `ffmpeg` in Opus/OGG transcodiert (48kHz Mono, 64kbps, auf 20 Minuten begrenzt).
  - **Video:** unveränderte Weiterleitung bis zu 16MB.
  - **Dokumente:** alles andere bis zu 100MB; der Dateiname bleibt erhalten, sofern verfügbar.
- GIF-artige Wiedergabe in WhatsApp: Senden Sie eine MP4-Datei mit `gifPlayback: true` (CLI: `--gif-playback`), damit mobile Clients sie inline in einer Schleife wiedergeben.
- Bei der MIME-Erkennung haben anhand magischer Bytes erkannte Typen Vorrang, gefolgt von der Dateierweiterung und anschließend den Antwort-Headern; ein generisch erkannter Container (`application/octet-stream`, `zip`) überschreibt niemals eine spezifischere Zuordnung anhand der Erweiterung (beispielsweise XLSX gegenüber ZIP).
- Die Bildunterschrift stammt aus `--message` oder `reply.text`; eine leere Bildunterschrift ist zulässig.
- Protokollierung: Ohne ausführliche Ausgabe werden `↩️`/`✅` angezeigt; die ausführliche Ausgabe enthält Größe und Quellpfad/URL.

<Note>
Die oben genannten Werte von 16MB für Audio/Video und 100MB für Dokumente sind die gemeinsamen Standardwerte je Medientyp, wenn kein explizites Byte-Limit übergeben wird. WhatsApp-Sendungen legen anhand von `channels.whatsapp.mediaMaxMb` ein explizites Limit fest (standardmäßig 50MB), das für dieses Konto einheitlich für alle Medientypen gilt.
</Note>

## Pipeline für automatische Antworten

- `getReplyFromConfig` gibt eine Antwortnutzlast (oder ein Array von Nutzlasten) zurück, die neben anderen Feldern `text?`, `mediaUrl?` und `mediaUrls?` enthält.
- Wenn Medien vorhanden sind, löst der Web-Sender lokale Pfade oder URLs mithilfe derselben Pipeline wie `openclaw message send` auf.
- Mehrere Medieneinträge werden, sofern vorhanden, nacheinander gesendet.

## Eingehende Medien für Befehle

- Wenn eingehende Web-Nachrichten Medien enthalten, lädt OpenClaw diese in eine temporäre Datei herunter und stellt Vorlagenvariablen bereit:
  - `{{AttachmentUrl}}` — ursprüngliche URL oder Provider-Referenz für den aktuellen Anhang.
  - `{{AttachmentPath}}` — lokaler temporärer Pfad, der vor der Ausführung des Befehls geschrieben wird.
  - `{{AttachmentContentType}}` — MIME-Inhaltstyp.
  - `{{AttachmentDir}}` — Verzeichnis, das den lokalen Pfad enthält.
  - `{{AttachmentIndex}}` — nullbasierter Index des Quellenfakts.
- Wenn eine sitzungsspezifische Docker-Sandbox aktiviert ist, werden eingehende Medien in den Sandbox-Arbeitsbereich kopiert und der Pfad bzw. die Referenz des Anhangs wird in einen Sandbox-relativen Pfad wie `media/inbound/<filename>` umgeschrieben.
- `{{MediaPath}}`, `{{MediaUrl}}`, `{{MediaType}}` und `{{MediaDir}}` bleiben während des Migrationszeitraums des Plugin-SDK veraltete Kompatibilitätsaliase.
- Die Medienanalyse (konfiguriert über `tools.media.*` oder das gemeinsam verwendete `tools.media.models`) wird vor der Vorlagenerstellung ausgeführt und kann die Blöcke `[Image]`, `[Audio]` und `[Video]` in `Body` einfügen.
  - Audio setzt `{{Transcript}}` und verwendet das Transkript für die Befehlsanalyse, damit Slash-Befehle weiterhin funktionieren.
  - Video- und Bildbeschreibungen behalten etwaigen Bildunterschriftstext für die Befehlsanalyse bei.
  - Wenn das aktive primäre Modell bereits nativ Bildverarbeitung unterstützt, überspringt OpenClaw den Zusammenfassungsblock `[Image]` und übergibt stattdessen das Originalbild an das Modell.
- Standardmäßig wird nur der erste passende Bild-, Audio- oder Videoanhang verarbeitet; verwenden Sie `tools.media.<capability>.attachments`, um mehrere Anhänge auszuwählen.

## Grenzwerte und Fehler

**Grenzwerte für ausgehende Sendungen (WhatsApp-Web-Versand)**

- Bilder: nach der Optimierung bis zu `channels.whatsapp.mediaMaxMb` (standardmäßig 50MB).
- Audio/Video: Grenzwert von 16MB (gemeinsamer Standardwert; beim Senden über WhatsApp durch `mediaMaxMb` überschrieben).
- Dokumente: Grenzwert von 100MB (gemeinsamer Standardwert; beim Senden über WhatsApp durch `mediaMaxMb` überschrieben).
- Zu große oder nicht lesbare Medien erzeugen einen eindeutigen Fehler in den Protokollen, und die Antwort wird übersprungen.

**Grenzwerte für die Medienanalyse (Transkription/Beschreibung)**

- Standardwert für Bilder: 10MB (überschreibbar mit `tools.media.image.maxBytes` oder pro
  `tools.media.models[]`-Eintrag mit `maxBytes`).
- Standardwert für Audio: 20MB (überschreibbar mit `tools.media.audio.maxBytes` oder pro Eintrag).
- Standardwert für Video: 50MB (überschreibbar mit `tools.media.video.maxBytes` oder pro Eintrag).
- Bei zu großen Medien wird die Analyse übersprungen, die Antwort wird jedoch weiterhin mit dem ursprünglichen Text verarbeitet.

## Hinweise für Tests

- Sende- und Antwortabläufe für Bild-, Audio- und Dokumentfälle abdecken.
- Größenbeschränkungen nach der Bildoptimierung sowie das Sprachnachrichten-Flag für Audio validieren.
- Sicherstellen, dass Antworten mit mehreren Medien in aufeinanderfolgende Sendungen aufgefächert werden.

## Verwandte Themen

- [Kameraaufnahme](/de/nodes/camera)
- [Medienanalyse](/de/nodes/media-understanding)
- [Audio und Sprachnachrichten](/de/nodes/audio)
