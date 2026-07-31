---
read_when:
    - Skripte aus dem Repository ausführen
    - Skripte unter ./scripts hinzufügen oder ändern
summary: 'Repository-Skripte: Zweck, Umfang und Sicherheitshinweise'
title: Skripte
x-i18n:
    generated_at: "2026-07-26T17:50:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 323069190ea6647101ee7120e06f6b2a018833d0904a11787fa1b610f5b3d9e1
    source_path: help/scripts.md
    workflow: 16
---

`scripts/` enthält Hilfsskripte für lokale Workflows und Betriebsaufgaben. Verwenden Sie diese, wenn eine Aufgabe eindeutig mit einem Skript verknüpft ist; bevorzugen Sie andernfalls die CLI.

## Konventionen

- Skripte sind **optional**, sofern nicht in der Dokumentation oder in Release-Checklisten darauf verwiesen wird.
- Bevorzugen Sie vorhandene CLI-Schnittstellen (Beispiel: `openclaw models status --check`).
- Gehen Sie davon aus, dass Skripte hostspezifisch sind; lesen Sie sie, bevor Sie sie auf einem neuen Rechner ausführen.

## Skripte zur Authentifizierungsüberwachung

Die allgemeine Modellauthentifizierung wird unter [Authentifizierung](/de/gateway/authentication) behandelt. Die folgenden Skripte bilden ein separates, optionales System zur Überwachung eines **Claude Code CLI-Abonnementtokens** auf einem entfernten/headless Host und zur erneuten Authentifizierung per Smartphone:

- `scripts/setup-auth-system.sh` – einmalige Einrichtung: prüft die aktuelle Authentifizierung, unterstützt beim Erzeugen eines langlebigen `claude setup-token` und gibt Installationsschritte für systemd/Termux aus.
- `scripts/claude-auth-status.sh [full|json|simple]` – prüft den Authentifizierungsstatus von Claude Code und OpenClaw.
- `scripts/auth-monitor.sh` – fragt den Status regelmäßig ab und sendet eine Benachrichtigung (über OpenClaw send und/oder ntfy.sh), wenn das Token kurz vor dem Ablauf steht. Umgebungsvariablen: `WARN_HOURS` (Standardwert `2`), `NOTIFY_PHONE`, `NOTIFY_NTFY`. Führen Sie das Skript planmäßig über den mitgelieferten `scripts/systemd/openclaw-auth-monitor.{service,timer}` aus (alle 30 Minuten).
- `scripts/mobile-reauth.sh` – führt `claude setup-token` erneut aus und gibt URLs aus, die auf einem Smartphone geöffnet werden können; zur Verwendung über SSH aus Termux.
- `scripts/termux-quick-auth.sh`, `scripts/termux-auth-widget.sh`, `scripts/termux-sync-widget.sh` – Termux:Widget-Skripte, die per SSH eine Verbindung zum Host herstellen, eine Statusmeldung einblenden und bei abgelaufener Authentifizierung die Konsole/Anweisungen zur erneuten Authentifizierung öffnen.

## GitHub-Lesehilfsprogramm

Verwenden Sie `scripts/gh-read`, wenn `gh` für Repository-bezogene Leseaufrufe das Installationstoken einer GitHub App verwenden soll, während das normale `gh` für Schreibaktionen weiterhin mit Ihrer persönlichen Anmeldung ausgeführt wird.

Erforderliche Umgebungsvariablen:

- `OPENCLAW_GH_READ_APP_ID`
- `OPENCLAW_GH_READ_PRIVATE_KEY_FILE`

Optionale Umgebungsvariablen:

- `OPENCLAW_GH_READ_INSTALLATION_ID`, wenn Sie die Repository-basierte Suche nach der Installation überspringen möchten
- `OPENCLAW_GH_READ_PERMISSIONS` als kommagetrennte Überschreibung für die anzufordernde Teilmenge der Leseberechtigungen

Reihenfolge der Repository-Auflösung:

- `gh ... -R owner/repo`
- `GH_REPO`
- `git remote origin`

Beispiele:

- `scripts/gh-read pr view 123`
- `scripts/gh-read run list -R openclaw/openclaw`
- `scripts/gh-read api repos/openclaw/openclaw/pulls/123`

## Beim Hinzufügen von Skripten

- Halten Sie Skripte fokussiert und dokumentiert.
- Fügen Sie einen kurzen Eintrag in der relevanten Dokumentation hinzu (oder erstellen Sie diese, falls sie fehlt).

## Verwandte Themen

- [Tests](/de/help/testing)
- [Live-Tests](/de/help/testing-live)
