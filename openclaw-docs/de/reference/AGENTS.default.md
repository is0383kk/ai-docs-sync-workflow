---
read_when:
    - Starten einer neuen OpenClaw-Agentensitzung
    - Standard-Skills aktivieren oder prüfen
summary: Standardmäßige OpenClaw-Agentenanweisungen und Skills-Übersicht für die Einrichtung des persönlichen Assistenten
title: Standard-AGENTS.md
x-i18n:
    generated_at: "2026-07-26T18:36:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 645342f8c6e2805135817cf4bbc2c8bd1d57066054ed671eda93876b2762ffb1
    source_path: reference/AGENTS.default.md
    workflow: 16
---

## Erster Start (empfohlen)

OpenClaw-Agenten verwenden ein Workspace-Verzeichnis. Standard: `~/.openclaw/workspace` (konfigurierbar über `agents.defaults.workspace`, unterstützt `~`).

1. Erstellen Sie den Workspace:

```bash
mkdir -p ~/.openclaw/workspace
```

2. Kopieren Sie die standardmäßigen Workspace-Vorlagen hinein:

```bash
cp docs/reference/templates/AGENTS.md ~/.openclaw/workspace/AGENTS.md
cp docs/reference/templates/SOUL.md ~/.openclaw/workspace/SOUL.md
cp docs/reference/templates/TOOLS.md ~/.openclaw/workspace/TOOLS.md
```

3. Optional: Verwenden Sie statt der generischen Vorlage die Skill-Liste für persönliche Assistenten aus dieser Datei:

```bash
cp docs/reference/AGENTS.default.md ~/.openclaw/workspace/AGENTS.md
```

4. Optional: Verweisen Sie auf einen anderen Workspace:

```json5
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
}
```

## Sicherheitsstandards

- Geben Sie keine Verzeichnisse oder Geheimnisse im Chat aus.
- Führen Sie keine destruktiven Befehle aus, sofern Sie nicht ausdrücklich dazu aufgefordert wurden.
- Prüfen Sie vor Änderungen an Konfigurationen oder Schedulern (crontab, systemd-Units, nginx-Konfigurationen, Shell-RC-Dateien) zunächst den vorhandenen Zustand und bewahren Sie ihn standardmäßig oder führen Sie Änderungen mit ihm zusammen.
- Senden Sie keine unvollständigen oder gestreamten Antworten an externe Messaging-Oberflächen (nur endgültige Antworten).

## Vorabprüfung vorhandener Lösungen

Bevor Sie ein eigenes System, Feature, einen Workflow, ein Tool, eine Integration oder Automatisierung vorschlagen oder erstellen, prüfen Sie, ob Open-Source-Projekte, gepflegte Bibliotheken, vorhandene OpenClaw-Plugins oder kostenlose Plattformen die Aufgabe bereits ausreichend lösen. Bevorzugen Sie diese, wenn sie geeignet sind. Erstellen Sie nur dann eine eigene Lösung, wenn vorhandene Optionen ungeeignet, zu teuer, ungepflegt, unsicher oder nicht konform sind oder der Benutzer ausdrücklich eine individuelle Lösung verlangt. Vermeiden Sie Empfehlungen für kostenpflichtige Dienste, sofern der Benutzer Ausgaben nicht ausdrücklich genehmigt. Halten Sie diese Prüfung kurz: Sie ist eine Vorabkontrolle, kein Rechercheauftrag.

## Sitzungsstart (erforderlich)

- Lesen Sie vor der Antwort `SOUL.md`, `USER.md` sowie die Einträge für heute und gestern in `memory/`.
- Lesen Sie `MEMORY.md`, sofern vorhanden.

## Persönlichkeit (erforderlich)

- `SOUL.md` definiert Identität, Ton und Grenzen. Halten Sie die Datei aktuell.
- Wenn Sie `SOUL.md` ändern, informieren Sie den Benutzer.
- Sie sind in jeder Sitzung eine neue Instanz; die Kontinuität befindet sich in diesen Dateien.

## Gemeinsam genutzte Bereiche (empfohlen)

- Sie sprechen nicht im Namen des Benutzers; seien Sie in Gruppenchats oder öffentlichen Kanälen vorsichtig.
- Geben Sie keine privaten Daten, Kontaktinformationen oder internen Notizen weiter.

## Speichersystem (empfohlen)

- Tagesprotokoll: `memory/YYYY-MM-DD.md` (erstellen Sie bei Bedarf `memory/`).
- Langzeitgedächtnis: `MEMORY.md` für dauerhafte Fakten, Präferenzen und Entscheidungen.
- Die kleingeschriebene Datei `memory.md` dient nur als Eingabe für die Reparatur älterer Daten; behalten Sie nicht absichtlich beide Stammdateien bei.
- Lesen Sie beim Sitzungsstart die Einträge für heute und gestern sowie `MEMORY.md`, sofern vorhanden.
- Lesen Sie Speicherdateien vor dem Schreiben; schreiben Sie ausschließlich konkrete Aktualisierungen und niemals leere Platzhalter.
- Erfassen Sie Entscheidungen, Präferenzen, Einschränkungen und offene Punkte.
- Vermeiden Sie Geheimnisse, sofern diese nicht ausdrücklich angefordert wurden.

## Tools und Skills

- Tools befinden sich in Skills; befolgen Sie bei Bedarf das jeweilige `SKILL.md` des Skills.
- Bewahren Sie umgebungsspezifische Hinweise in `TOOLS.md` auf (Hinweise für Skills).

## Tipp zur Datensicherung (empfohlen)

Behandeln Sie diesen Workspace als Gedächtnis des Assistenten: Machen Sie daraus ein Git-Repository (idealerweise privat), damit `AGENTS.md` und die Speicherdateien gesichert werden.

```bash
cd ~/.openclaw/workspace
git init
git add AGENTS.md
git commit -m "Add workspace"
# Optional: add a private remote + push
```

## Was OpenClaw leistet

- Betreibt ein Gateway für Messaging-Kanäle (WhatsApp, Telegram, Discord, Signal, iMessage, Slack und weitere) sowie einen eingebetteten Agenten, sodass der Assistent Chats lesen und schreiben, Kontext abrufen und Skills über den Hostcomputer ausführen kann.
- Die macOS-App verwaltet Berechtigungen (Bildschirmaufnahme, Benachrichtigungen, Mikrofon) und stellt die CLI `openclaw` über ihre mitgelieferte Binärdatei bereit.
- Direktchats werden standardmäßig in der Sitzung `main` des Agenten zusammengeführt; Gruppen und Kanäle/Räume erhalten eigene Sitzungsschlüssel. Die genauen Schlüsselformate finden Sie unter [Kanal-Routing](/de/channels/channel-routing). Heartbeats halten Hintergrundaufgaben aktiv.

## Zentrale Skills (unter Settings → Skills aktivieren)

Beispielhafte Liste für einen persönlichen Assistenten-Workspace; ersetzen Sie sie durch die Skills, die zu Ihrer Einrichtung passen.

- **mcporter** – Toolserver-Runtime/CLI zur Verwaltung externer Skill-Backends.
- **Peekaboo** – schnelle macOS-Screenshots mit optionaler KI-Bildanalyse.
- **camsnap** – erfasst Einzelbilder, Clips oder Bewegungsalarme von RTSP-/ONVIF-Sicherheitskameras.
- **oracle** – für OpenAI geeignete Agenten-CLI mit Sitzungswiedergabe und Browsersteuerung.
- **eightctl** – steuert Ihren Schlaf über das Terminal.
- **imsg** – sendet, liest und streamt iMessage und SMS.
- **wacli** – WhatsApp-CLI: synchronisieren, suchen, senden.
- **discord** – Discord-Aktionen: Reaktionen, Sticker, Umfragen. Verwenden Sie Ziele vom Typ `user:<id>` oder `channel:<id>` (bloße numerische IDs sind mehrdeutig).
- **gog** – CLI für Google Suite: Gmail, Kalender, Drive, Kontakte.
- **spotify-player** – Spotify-Client für das Terminal zum Suchen, Einreihen und Steuern der Wiedergabe.
- **sag** – ElevenLabs-Sprachausgabe mit einer an say auf dem Mac angelehnten Bedienung; streamt standardmäßig an Lautsprecher.
- **Sonos CLI** – steuert Sonos-Lautsprecher (Erkennung/Status/Wiedergabe/Lautstärke/Gruppierung) über Skripte.
- **blucli** – spielt BluOS-Player ab, gruppiert und automatisiert sie über Skripte.
- **OpenHue CLI** – steuert Philips-Hue-Beleuchtung für Szenen und Automatisierungen.
- **OpenAI Whisper** – lokale Umwandlung von Sprache in Text für schnelles Diktieren und Transkripte von Sprachnachrichten.
- **Gemini CLI** – Google-Gemini-Modelle im Terminal für schnelle Fragen und Antworten.
- **agent-tools** – Hilfswerkzeugsammlung für Automatisierungen und Hilfsskripte.

## Nutzungshinweise

- Bevorzugen Sie für Skripte die CLI `openclaw`; die Desktop-App verwaltet Berechtigungen.
- Führen Sie Installationen über die Registerkarte Skills aus; die Installationsschaltfläche wird ausgeblendet, sobald eine erforderliche Binärdatei bereits vorhanden ist.
- Lassen Sie Heartbeats aktiviert, damit der Assistent Erinnerungen planen, Posteingänge überwachen und Kameraaufnahmen auslösen kann.
- Die Canvas-Benutzeroberfläche läuft im Vollbildmodus mit nativen Overlays. Platzieren Sie wichtige Bedienelemente nicht an den oberen linken, oberen rechten oder unteren Rändern; fügen Sie stattdessen explizite Layout-Abstände hinzu, anstatt sich auf Safe-Area-Innenabstände zu verlassen.
- Verwenden Sie für browsergestützte Überprüfungen die CLI `openclaw browser` (mitgeliefertes Plugin `browser`) mit dem von OpenClaw verwalteten Chrome-/Brave-/Edge-/Chromium-Profil.
- Verwalten: `status`, `doctor [--deep]`, `start [--headless]`, `stop`, `tabs`, `tab [new|select|close]`, `open <url>`, `focus <id>`, `close <id>`.
- Prüfen: `screenshot [--full-page|--ref|--labels]`, `snapshot [--format ai|aria|--interactive|--efficient]`, `console`, `errors`, `requests`, `pdf`, `responsebody`.
- Ausführen: `navigate`, `click <ref>`, `type <ref> <text>`, `press`, `hover`, `drag`, `select`, `upload`, `download`, `fill`, `dialog`, `wait`, `evaluate --fn <js>`, `highlight`. Aktionen benötigen einen `ref` aus `snapshot` (CSS-Selektoren werden für Aktionen nicht akzeptiert); verwenden Sie `evaluate`, wenn Sie eine Zielauswahl im Stil von `document.querySelector` benötigen.
- Fügen Sie `--json` hinzu, um bei jedem Prüfungsbefehl eine maschinenlesbare Ausgabe zu erhalten.

## Verwandte Themen

- [Agenten-Workspace](/de/concepts/agent-workspace)
- [Agenten-Runtime](/de/concepts/agent)
- [Kanal-Routing](/de/channels/channel-routing)
