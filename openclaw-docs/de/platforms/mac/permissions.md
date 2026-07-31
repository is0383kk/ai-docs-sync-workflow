---
read_when:
    - Fehlerbehebung bei fehlenden oder hängen gebliebenen macOS-Berechtigungsabfragen
    - Entscheiden, ob Node oder einer CLI-Laufzeitumgebung Bedienungshilfen gewährt werden sollen
    - Paketieren oder Signieren der macOS-App
    - Bundle-IDs oder App-Installationspfade ändern
summary: Persistenz von macOS-Berechtigungen (TCC) und Signierungsanforderungen
title: macOS-Berechtigungen
x-i18n:
    generated_at: "2026-07-26T18:34:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e561aa641e44fc1e1b95a3db244f31124e4e51d13ae709bee188d86054301e34
    source_path: platforms/mac/permissions.md
    workflow: 16
---

macOS-Berechtigungsfreigaben sind fragil. TCC verknüpft eine Berechtigungsfreigabe mit der Codesignatur, der Bundle-ID und dem Speicherpfad der App. Wenn sich eine dieser Angaben ändert, behandelt macOS die App als neu und verwirft oder verbirgt möglicherweise Aufforderungen.

## Anforderungen für stabile Berechtigungen

- Gleicher Pfad: Führen Sie die App von einem festen Speicherort aus (für OpenClaw: `dist/OpenClaw.app`).
- Gleiche Bundle-ID: Die Bundle-ID von OpenClaw lautet `ai.openclaw.mac`; eine Änderung erzeugt eine neue Berechtigungsidentität.
- Signierte App: Bei unsignierten oder ad hoc signierten Builds bleiben Berechtigungen nicht erhalten.
- Konsistente Signatur: Verwenden Sie ein echtes Apple-Development- oder Developer-ID-Zertifikat, damit die Signatur über erneute Builds hinweg stabil bleibt.

Ad-hoc-Signaturen erzeugen bei jedem Build eine neue Identität. macOS vergisst vorherige Freigaben, und Aufforderungen können vollständig verschwinden, bis die veralteten Einträge gelöscht werden.

## Bedienungshilfen-Freigaben für Node- und CLI-Laufzeitumgebungen

Gewähren Sie Bedienungshilfen vorzugsweise OpenClaw.app, Peekaboo.app oder einem anderen signierten Hilfsprogramm mit eigener Bundle-ID statt einer generischen `node`-Binärdatei.

macOS TCC gewährt Bedienungshilfen für die Codeidentität des erkannten Prozesses. Wenn ein Homebrew-, nvm-, pnpm- oder npm-Workflow dazu führt, dass eine gemeinsam genutzte ausführbare `node`-Datei Bedienungshilfen erhält, kann jedes über dieselbe ausführbare Datei gestartete JavaScript-Paket Berechtigungen zur GUI-Automatisierung erben.

Behandeln Sie einen `node`-Eintrag in den Systemeinstellungen als weitreichende Berechtigung für diese Node-Laufzeitumgebung, nicht als Berechtigung für ein einzelnes npm-Paket. Gewähren Sie `node` keine Bedienungshilfen, sofern Sie nicht jedem Skript und Paket vertrauen, das über genau diese Node-Installation gestartet wird.

Die Genehmigung für Bedienungshilfen aktiviert nicht die Freigabe von Aktivitätsdaten. **Settings -> Permissions -> Active computer detection** ist eine separate, standardmäßig deaktivierte Einstellung zur Weitergabe einer begrenzten Leerlaufdauer an Ihren Gateway. Durch das Deaktivieren werden gespeicherte Aktivitätsdaten gelöscht, ohne die Bedienungshilfen zu widerrufen oder die Verbindung zum Node zu trennen.

Wenn Sie `node` versehentlich Bedienungshilfen gewährt haben, entfernen Sie diesen Eintrag unter System Settings -> Privacy & Security -> Accessibility. Gewähren Sie die Berechtigung anschließend der signierten App oder dem Hilfsprogramm, die bzw. das die UI-Automatisierung übernehmen soll.

## Checkliste zur Wiederherstellung, wenn Aufforderungen verschwinden

1. Beenden Sie die App.
2. Entfernen Sie den App-Eintrag unter System Settings -> Privacy & Security.
3. Starten Sie die App erneut vom selben Pfad und gewähren Sie die Berechtigungen erneut.
4. Wenn die Aufforderung weiterhin nicht angezeigt wird, setzen Sie die TCC-Einträge mit `tccutil` zurück und versuchen Sie es erneut.
5. Einige Berechtigungen werden erst nach einem vollständigen Neustart von macOS wieder angezeigt.

Beispiele für das Zurücksetzen (mit der Bundle-ID von OpenClaw, `ai.openclaw.mac`):

```bash
sudo tccutil reset Accessibility ai.openclaw.mac
sudo tccutil reset ScreenCapture ai.openclaw.mac
sudo tccutil reset AppleEvents
```

## Berechtigungen für Dateien und Ordner (Schreibtisch/Dokumente/Downloads)

macOS kann den Zugriff auf Schreibtisch, Dokumente und Downloads auch für Terminal- und Hintergrundprozesse beschränken. Wenn das Lesen von Dateien oder das Auflisten von Verzeichnissen nicht abgeschlossen wird, gewähren Sie dem Prozesskontext Zugriff, der die Dateioperationen ausführt (beispielsweise Terminal/iTerm, eine über LaunchAgent gestartete App oder ein SSH-Prozess).

Problemumgehung: Verschieben Sie die Dateien in den OpenClaw-Arbeitsbereich (`~/.openclaw/workspace`), wenn Sie einzelne Ordnerfreigaben vermeiden möchten.

Wenn Sie Berechtigungen testen, signieren Sie immer mit einem echten Zertifikat. Ad-hoc-Builds sind nur für kurze lokale Ausführungen akzeptabel, bei denen Berechtigungen keine Rolle spielen.

## Verwandte Themen

- [macOS-App](/de/platforms/macos)
- [macOS-Signierung](/de/platforms/mac/signing)
