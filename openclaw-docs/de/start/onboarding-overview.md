---
read_when:
    - Auswahl eines Onboarding-Pfads
    - Einrichten einer neuen Umgebung
sidebarTitle: Onboarding Overview
summary: Überblick über die Onboarding-Optionen und -Abläufe von OpenClaw
title: Onboarding-Übersicht
x-i18n:
    generated_at: "2026-07-26T18:38:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4bcda1dcfb91f388ca6bef59f9bdf5177571d93c0d89c45025ef837628fa7ba0
    source_path: start/onboarding-overview.md
    workflow: 16
---

OpenClaw bietet ein Onboarding im Terminal und in der macOS-App. Beide richten zuerst die Inferenz ein:
Sie erkennen vorhandenen KI-Zugriff, erfordern eine erfolgreiche Live-Vervollständigung und starten erst dann
OpenClaw, um die verbleibende Einrichtung zu konfigurieren. Bei einem erreichbaren, konfigurierten Gateway,
dessen Standard-Agent bereits über ein konfiguriertes Modell verfügt, wird das Onboarding übersprungen und
die normale Agenten-Benutzeroberfläche geöffnet. Der Terminal-Ablauf bietet außerdem den vollständigen klassischen Assistenten für
eine detaillierte Einrichtung.

## Welchen Weg sollte ich verwenden?

|                  | CLI-Onboarding                          | Onboarding in der macOS-App      |
| ---------------- | --------------------------------------- | -------------------------------- |
| **Plattformen**  | macOS, Linux, Windows (nativ oder WSL2) | Nur macOS                        |
| **Oberfläche**   | Inferenz-Einrichtung, dann OpenClaw     | Inferenz-Einrichtung, dann OpenClaw |
| **Geeignet für** | Server, Headless-Betrieb, volle Kontrolle | Desktop-Mac, visuelle Einrichtung |
| **Automatisierung** | `--non-interactive` für Skripte       | Nur manuell                      |
| **Befehl**       | `openclaw onboard`                      | App starten                      |

Die meisten Benutzer sollten mit dem **CLI-Onboarding** beginnen — es funktioniert überall und bietet
Ihnen die größte Kontrolle.

## Was das Onboarding konfiguriert

Die geführte Inferenzphase richtet nur Folgendes ein:

1. **Modell-Provider und Authentifizierung** — erkannter Zugriff oder eine verifizierte Provider-Anmeldung,
   ein API-Schlüssel oder Token
2. **Verifizierte Inferenz** — eine tatsächliche Vervollständigung mit dem effektiven
   Modell des Standard-Agenten

Nachdem diese Vervollständigung erfolgreich war, kann OpenClaw den Arbeitsbereich, das Gateway,
den Gateway-Dienst, Kanäle, Agenten, Plugins und weitere optionale Funktionen konfigurieren.

Der klassische CLI-Assistent kann zusätzlich Folgendes konfigurieren:

1. **Kanäle** (optional) — integrierte und mitgelieferte Chat-Kanäle wie
   Discord, Feishu, Google Chat, iMessage, Mattermost, Microsoft Teams,
   Telegram, WhatsApp und weitere
2. **Erweiterte Gateway-Steuerung** — Remote-Modus, Netzwerkeinstellungen und Daemon-Optionen

## CLI-Onboarding

In einem beliebigen Terminal ausführen:

```bash
openclaw onboard
```

Der geführte Ablauf erkennt vorhandenen KI-Zugriff, testet Kandidaten der Reihe nach live
und fährt bei einem Fehler mit dem nächsten fort. Wenn die Erkennung ausgeschöpft ist, werden zuerst OpenAI,
Anthropic, xAI (Grok), Google und OpenRouter angezeigt. **Mehr…** enthält die
verbleibenden Provider in Provider-Gruppen sowie Regionen, Tarife und unterstützte
Browser-, Geräte-, API-Schlüssel- oder Token-Methoden in einem zweiten Menü. Modell
und Zugangsdaten werden erst nach einer erfolgreichen Vervollständigung gespeichert. Anschließend wird OpenClaw gestartet, um
den Arbeitsbereich, das Gateway, Kanäle, Agenten, Plugins und weitere optionale
Funktionen zu konfigurieren. **Vorerst überspringen** beendet den Ablauf, ohne OpenClaw zu starten. Innerhalb
des Ablaufs erfolgt keine Übergabe an den klassischen Assistenten. Beenden Sie den Ablauf und führen Sie `openclaw onboard --classic` aus, wenn Sie stattdessen
den klassischen Assistenten verwenden möchten.

Nach erfolgreicher Inferenz kann OpenClaw die Kanaleinrichtung an einen Terminal-Assistenten
mit maskierter Eingabe übergeben. Dieser öffnet weder die geführte noch die klassische Provider-Einrichtung. Beenden Sie OpenClaw und
führen Sie `openclaw onboard` aus, um den Modell-Provider oder dessen Authentifizierung zu ändern.

Verwenden Sie `openclaw onboard --classic` für die detaillierte Einrichtung von Modell/Authentifizierung, Kanälen, Skills,
Remote-Gateway oder Importen. Durch zusätzliches Angeben von `--install-daemon` wird außerdem der
klassische Ablauf ausgewählt und der Hintergrunddienst in einem Schritt installiert. Verwenden Sie `openclaw
openclaw` für die dialogorientierte Einrichtung und Reparatur ohne Inferenz. `openclaw
onboard --modern` ist ein Kompatibilitätsalias, der dieselbe Live-Inferenzprüfung
verwendet.

Vollständige Referenz: [Onboarding (CLI)](/de/start/wizard)
Dokumentation zum CLI-Befehl: [`openclaw onboard`](/de/cli/onboard)

## Onboarding in der macOS-App

Öffnen Sie die OpenClaw-App. Wenn das konfigurierte lokale oder Remote-Gateway erreichbar ist
und der Standard-Agent bereits über ein konfiguriertes Modell verfügt, überspringt die App das Onboarding
und OpenClaw und öffnet sofort die normale Agenten-Benutzeroberfläche.

Bei einem neuen oder unvollständig eingerichteten Gateway erkennt der Ablauf beim ersten Start vorhandenen KI-
Zugriff (Claude Code, Codex oder API-Schlüssel), testet die beste
Option live und speichert sie erst nach einer tatsächlichen Antwort. Dabei greift er automatisch auf Alternativen zurück und
bietet einen verifizierten manuellen API-Schlüssel-Schritt an, wenn nichts gefunden wird. Vertrauliche
Zugangsdaten werden maskiert eingegeben. Sobald die Inferenz erfolgreich ist, startet OpenClaw und
hilft bei der Konfiguration der übrigen Komponenten.

Gemini CLI bleibt nach der Einrichtung für normale Agenten verfügbar, wird für diese
Inferenzprüfung jedoch nicht angeboten, da damit die Ausführung der Prüfung ohne Tools nicht erzwungen werden kann.

Vollständige Referenz: [Onboarding (macOS-App)](/de/start/onboarding)

## Benutzerdefinierte oder nicht aufgeführte Provider

Wenn Ihr Provider nicht aufgeführt ist, führen Sie `openclaw onboard --classic` aus, wählen Sie
**Benutzerdefinierter Provider** und geben Sie Folgendes ein:

- Endpunkt-Kompatibilität: OpenAI-kompatibel (`/chat/completions`), mit OpenAI Responses kompatibel (`/responses`), Anthropic-kompatibel (`/messages`) oder unbekannt (prüft alle drei und erkennt sie automatisch)
- Basis-URL und API-Schlüssel (der API-Schlüssel ist optional, wenn der Endpunkt keinen erfordert)
- Modell-ID und optionaler Modellalias

Mehrere benutzerdefinierte Endpunkte können gleichzeitig vorhanden sein — jeder erhält eine eigene Endpunkt-ID.

## Verwandte Themen

- [Erste Schritte](/de/start/getting-started)
- [Referenz zur CLI-Einrichtung](/de/start/wizard-cli-reference)
