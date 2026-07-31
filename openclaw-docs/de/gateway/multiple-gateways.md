---
read_when:
    - Mehr als ein Gateway auf demselben Rechner ausführen
    - Sie benötigen separate Konfigurationen, Zustände und Ports für jedes Gateway
summary: Mehrere OpenClaw-Gateways auf einem Host ausführen (Isolierung, Ports und Profile)
title: Mehrere Gateways
x-i18n:
    generated_at: "2026-07-26T18:23:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 655fa865a98064d7c017a7c2eb08ea9a9683002d96a3dbe45a8c16cbd3c86ba1
    source_path: gateway/multiple-gateways.md
    workflow: 16
---

Die meisten Setups benötigen ein Gateway – ein einzelnes Gateway verarbeitet mehrere Messaging-Verbindungen und Agenten. Führen Sie separate Gateways mit isolierten Profilen/Ports nur aus, wenn Sie eine stärkere Isolation oder Redundanz benötigen (z. B. einen Rettungs-Bot).

## Schnellstart für den Rettungs-Bot

Das einfachste Setup für einen Rettungs-Bot:

- Belassen Sie den Haupt-Bot im Standardprofil.
- Führen Sie den Rettungs-Bot unter `--profile rescue` mit einem eigenen Telegram-Bot-Token aus.
- Legen Sie für den Rettungs-Bot einen anderen Basisport fest, z. B. `19789`.

So kann der Rettungs-Bot Fehler diagnostizieren oder Konfigurationsänderungen anwenden, wenn der primäre Bot ausgefallen ist. Lassen Sie zwischen den Basisports mindestens 20 Ports frei, damit abgeleitete Browser-/CDP-Ports niemals kollidieren.

```bash
# Rettungs-Bot (separater Telegram-Bot, separates Profil, Port 19789)
openclaw --profile rescue onboard
openclaw --profile rescue gateway install --port 19789
```

Wenn Ihr Haupt-Bot bereits ausgeführt wird, ist dies normalerweise alles, was Sie benötigen. Falls das Onboarding den Rettungsdienst bereits installiert hat, überspringen Sie den abschließenden Befehl `gateway install`.

Während `openclaw --profile rescue onboard`:

- Verwenden Sie ein separates Telegram-Bot-Token, das ausschließlich für das Rettungskonto vorgesehen ist (so lässt es sich leicht auf Operatoren beschränken, bleibt von der Kanal-/App-Installation des Haupt-Bots unabhängig und bietet einen einfachen DM-basierten Wiederherstellungsweg).
- Behalten Sie den Profilnamen `rescue` bei.
- Verwenden Sie einen Basisport, der mindestens 20 höher als der des Haupt-Bots ist.
- Übernehmen Sie den standardmäßigen Rettungs-Workspace, sofern Sie nicht bereits selbst einen verwalten.

### Was `--profile rescue onboard` ändert

`--profile rescue onboard` führt den normalen Onboarding-Ablauf aus, schreibt jedoch alles in ein separates Profil, sodass der Rettungs-Bot Folgendes separat erhält:

- Profil-/Konfigurationsdatei
- Statusverzeichnis
- Workspace (Standard: `~/.openclaw/workspace-rescue`)
- Name des verwalteten Dienstes
- Basisport (zuzüglich abgeleiteter Ports)
- Telegram-Bot-Token

Die Eingabeaufforderungen sind ansonsten mit denen des normalen Onboardings identisch.

## Allgemeines Multi-Gateway-Setup

Dasselbe Isolationsmuster funktioniert für jedes Paar oder jede Gruppe von Gateways auf einem Host – weisen Sie jedem zusätzlichen Gateway ein eigenes benanntes Profil und einen eigenen Basisport zu:

```bash
# Haupt-Gateway (Standardprofil)
openclaw setup
openclaw gateway --port 18789

# zusätzliches Gateway
openclaw --profile ops setup
openclaw --profile ops gateway --port 19789
```

Benannte Profile auf beiden Seiten funktionieren ebenfalls:

```bash
openclaw --profile main setup
openclaw --profile main gateway --port 18789

openclaw --profile ops setup
openclaw --profile ops gateway --port 19789
```

Dienste folgen demselben Muster:

```bash
openclaw gateway install
openclaw --profile ops gateway install --port 19789
```

Verwenden Sie den Schnellstart für den Rettungs-Bot als Ausweichkanal für Operatoren. Verwenden Sie das allgemeine Profilmuster für mehrere langlebige Gateways über verschiedene Kanäle, Mandanten, Workspaces oder Betriebsrollen hinweg.

## Isolations-Checkliste

Halten Sie diese Einstellungen für jede Gateway-Instanz eindeutig:

| Einstellung                      | Zweck                                     |
| -------------------------------- | ----------------------------------------- |
| `OPENCLAW_CONFIG_PATH`       | Konfigurationsdatei pro Instanz            |
| `OPENCLAW_STATE_DIR`         | Sitzungen, Anmeldedaten und Caches pro Instanz |
| `agents.defaults.workspace`  | Workspace-Stammverzeichnis pro Instanz     |
| `gateway.port` (oder `--port`) | Eindeutig pro Instanz                      |
| Abgeleitete Browser-/CDP-Ports   | Siehe unten                               |

Wenn Sie eine dieser Ressourcen gemeinsam verwenden, führt dies zu Konflikten bei Konfiguration, Status oder Ports. Der Gateway-Start
erzwingt eine eindeutige Eigentümerschaft des Statusverzeichnisses, selbst wenn
`OPENCLAW_ALLOW_MULTI_GATEWAY=1` die Singleton-Prüfung pro Konfiguration überspringt.

## Portzuordnung (abgeleitet)

Basisport = `gateway.port` (oder `OPENCLAW_GATEWAY_PORT` / `--port`).

- Port des Browser-Steuerungsdienstes = Basisport + 2 (nur Loopback).
- Der Canvas-Host wird auf dem Gateway-HTTP-Server selbst bereitgestellt (derselbe Port wie `gateway.port`).
- Die CDP-Ports der Browserprofile werden automatisch im Bereich von `browser control port + 9` bis `+ 108` zugewiesen.

Wenn Sie einen dieser Werte in der Konfiguration oder über Umgebungsvariablen überschreiben, müssen Sie ihn für jede Instanz eindeutig halten.

## Hinweise zu Browser/CDP (häufige Fehlerquelle)

- Legen Sie `browser.cdpUrl` auf mehreren Instanzen **nicht** auf denselben Wert fest.
- Jede Instanz benötigt einen eigenen Browser-Steuerungsport und CDP-Bereich (abgeleitet von ihrem Gateway-Port).
- Legen Sie für explizite CDP-Ports `browser.profiles.<name>.cdpPort` pro Instanz fest.
- Verwenden Sie für Remote-Chrome `browser.profiles.<name>.cdpUrl` (pro Profil und Instanz).

## Manuelles Beispiel mit Umgebungsvariablen

```bash
OPENCLAW_CONFIG_PATH=~/.openclaw/main.json \
OPENCLAW_STATE_DIR=~/.openclaw \
openclaw gateway --port 18789

OPENCLAW_CONFIG_PATH=~/.openclaw/rescue.json \
OPENCLAW_STATE_DIR=~/.openclaw-rescue \
openclaw gateway --port 19789
```

## Schnellprüfungen

```bash
openclaw gateway status --deep
openclaw --profile rescue gateway status --deep
openclaw --profile rescue gateway probe
openclaw status
openclaw --profile rescue status
openclaw --profile rescue browser status
```

- `gateway status --deep` erkennt veraltete launchd-/systemd-/schtasks-Dienste aus älteren Installationen.
- Warntexte von `gateway probe` wie `multiple reachable gateway identities detected` sind nur zu erwarten, wenn Sie absichtlich mehr als ein isoliertes Gateway ausführen oder wenn OpenClaw nicht nachweisen kann, dass erreichbare Prüfziele dasselbe Gateway sind. Ein SSH-Tunnel, eine Proxy-URL oder eine konfigurierte Remote-URL zum selben Gateway stellt ein Gateway mit mehreren Transportwegen dar, selbst wenn sich deren Ports unterscheiden.

## Verwandte Themen

- [Gateway-Betriebshandbuch](/de/gateway)
- [Gateway-Sperre](/de/gateway/gateway-lock)
- [Konfiguration](/de/gateway/configuration)
