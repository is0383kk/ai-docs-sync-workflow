---
read_when:
    - Ändern des Dateizugriffs, der Archivextraktion, des Workspace-Speichers oder der Plugin-Dateisystemhilfen
summary: Wie OpenClaw den Zugriff auf lokale Dateien sicher handhabt und warum der optionale Python-Helfer `fs-safe` standardmäßig deaktiviert ist
title: Sichere Dateioperationen
x-i18n:
    generated_at: "2026-07-26T18:24:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5c8edf36ddbb8c8bc1edc52ecdf481affe5395d1779c679a40439167dfe70299
    source_path: gateway/security/secure-file-operations.md
    workflow: 16
---

OpenClaw verwendet [`@openclaw/fs-safe`](https://github.com/openclaw/fs-safe) für sicherheitskritische lokale Dateioperationen: auf ein Stammverzeichnis begrenzte Lese-/Schreibvorgänge, atomare Ersetzung, Archivextraktion, temporäre Arbeitsbereiche, JSON-Zustand und die Verarbeitung geheimer Dateien.

Es handelt sich um eine **Schutzvorkehrung auf Bibliotheksebene** für vertrauenswürdigen OpenClaw-Code, der nicht vertrauenswürdige Pfadnamen empfängt, nicht um eine Sandbox. Die Dateisystemberechtigungen des Hosts, Betriebssystembenutzer, Container und die Agenten-/Tool-Richtlinie bestimmen weiterhin den tatsächlichen Schadensradius.

## Standard: kein Python-Hilfsprozess

OpenClaw setzt den POSIX-Python-Hilfsprozess von fs-safe standardmäßig auf **aus**:

- Der Gateway sollte keinen persistenten Python-Sidecar-Prozess starten, sofern ein Betreiber dies nicht ausdrücklich aktiviert;
- die meisten Installationen benötigen die zusätzliche Absicherung gegen Änderungen an übergeordneten Verzeichnissen nicht;
- die Deaktivierung von Python sorgt für vorhersehbares Laufzeitverhalten in Desktop-, Docker-, CI- und gebündelten App-Umgebungen.

OpenClaw ändert nur den _Standardwert_. Eine explizite Einstellung hat immer Vorrang:

```bash
# Standardverhalten von OpenClaw: reine Node-Fallbacks von fs-safe.
OPENCLAW_FS_SAFE_PYTHON_MODE=off

# Hilfsprozess verwenden, wenn verfügbar, andernfalls auf den Fallback zurückgreifen.
OPENCLAW_FS_SAFE_PYTHON_MODE=auto

# Sicher abbrechen, wenn der Hilfsprozess nicht gestartet werden kann.
OPENCLAW_FS_SAFE_PYTHON_MODE=require

# Optionaler expliziter Pfad zum Interpreter.
OPENCLAW_FS_SAFE_PYTHON=/usr/bin/python3
```

Die generischen fs-safe-Umgebungsvariablen funktionieren ebenfalls: `FS_SAFE_PYTHON_MODE` und `FS_SAFE_PYTHON`.

Verwenden Sie `require` (nicht `auto`), wenn der Hilfsprozess Teil Ihres Sicherheitskonzepts ist; `auto` greift unbemerkt auf reines Node-Verhalten zurück, wenn der Hilfsprozess nicht gestartet werden kann.

## Was ohne Python geschützt bleibt

Bei deaktiviertem Hilfsprozess stehen OpenClaw weiterhin die reinen Node-Schutzvorkehrungen von fs-safe zur Verfügung:

- weist Ausbrüche aus relativen Pfaden (`..`), absolute Pfade und Pfadtrennzeichen zurück, wenn nur einfache Namen zulässig sind;
- führt Operationen über ein vertrauenswürdiges Stammverzeichnis-Handle aus statt über ad-hoc-Prüfungen mit `path.resolve(...).startsWith(...)`;
- lehnt bei APIs, die diese Richtlinie erfordern, Muster mit symbolischen und festen Links ab;
- öffnet Dateien mit Identitätsprüfungen, wenn die API Dateiinhalte zurückgibt oder verarbeitet;
- schreibt Zustands-/Konfigurationsdateien über eine temporäre Geschwisterdatei mit anschließender atomarer Umbenennung;
- setzt Bytegrenzen für Lesevorgänge und die Archivextraktion durch;
- wendet private Dateimodi auf geheime Dateien und Zustandsdateien an, wenn die API dies erfordert.

Dies deckt das normale Bedrohungsmodell von OpenClaw ab: vertrauenswürdiger Gateway-Code verarbeitet nicht vertrauenswürdige Pfadeingaben von Modellen, Plugins und Kanälen innerhalb einer einzelnen vertrauenswürdigen Betreibergrenze.

## Was Python ergänzt

Unter POSIX hält der optionale Hilfsprozess einen persistenten Python-Prozess vor und verwendet auf Dateideskriptoren bezogene Dateisystemoperationen für Änderungen an übergeordneten Verzeichnissen: Umbenennen, Entfernen, Erstellen von Verzeichnissen, Abrufen von Status-/Verzeichnislisten und einige Schreibpfade.

Dies verkleinert Race-Condition-Zeitfenster für Prozesse mit derselben UID, in denen ein anderer Prozess zwischen Validierung und Änderung ein übergeordnetes Verzeichnis austauscht – eine zusätzliche Sicherheitsebene auf Hosts, auf denen nicht vertrauenswürdige lokale Prozesse dieselben Verzeichnisse ändern können, in denen OpenClaw arbeitet.

Wenn dieses Risiko in Ihrer Bereitstellung besteht und die Verfügbarkeit von Python garantiert ist, legen Sie Folgendes fest:

```bash
OPENCLAW_FS_SAFE_PYTHON_MODE=require
```

## Hinweise für Plugins und den Kern

- Der Dateizugriff für Plugins sollte über die Hilfsfunktionen von `openclaw/plugin-sdk/*` statt über rohes `fs` erfolgen, wenn ein Pfad aus einer Nachricht, einer Modellausgabe, einer Konfiguration oder einer Plugin-Eingabe stammt.
- Der Kerncode sollte die fs-safe-Wrapper unter `src/infra/*` verwenden, damit die Prozessrichtlinie von OpenClaw einheitlich angewendet wird.
- Die Archivextraktion sollte die fs-safe-Archivhilfsfunktionen mit expliziten Grenzwerten für Größe, Eintragsanzahl, Links und Ziel verwenden.
- Für Geheimnisse sollten die Geheimnishilfsfunktionen von OpenClaw oder die fs-safe-Hilfsfunktionen für Geheimnisse/private Zustände verwendet werden; implementieren Sie keine eigenen Modusprüfungen um `fs.writeFile`.
- Verlassen Sie sich zur Isolation vor feindseligen lokalen Benutzern nicht allein auf fs-safe. Führen Sie separate Gateways unter separaten Betriebssystembenutzern beziehungsweise auf separaten Hosts aus oder verwenden Sie Sandboxing.

Verwandte Themen: [Sicherheit](/de/gateway/security), [Sandboxing](/de/gateway/sandboxing), [Ausführungsgenehmigungen](/de/tools/exec-approvals), [Geheimnisse](/de/gateway/secrets).
