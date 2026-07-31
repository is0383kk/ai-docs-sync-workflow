---
read_when:
    - Sie installieren, konfigurieren oder prüfen das Richtlinien-Plugin
summary: Fügt richtliniengestützte Doctor-Prüfungen für die Workspace-Konformität hinzu.
title: Richtlinien-Plugin
x-i18n:
    generated_at: "2026-07-26T18:03:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 440f2f46e4149fdd5e65bf0140d4981c6d840e8e8c8a85d05eeb23a0839a61ac
    source_path: plugins/reference/policy.md
    workflow: 16
---

# Policy-Plugin

Fügt richtliniengestützte Doctor-Prüfungen für die Workspace-Konformität hinzu.

## Distribution

- Paket: `@openclaw/policy`
- Installationsweg: in OpenClaw enthalten

## Oberfläche

Plugin

<!-- openclaw-plugin-reference:manual-start -->

## Verhalten

Das Policy-Plugin stellt Doctor-Integritätsprüfungen für richtlinienverwaltete OpenClaw-
Einstellungen und geregelte Workspace-Deklarationen bereit. Policy deckt derzeit die
Konformität von Kanälen, geregelte Tool-Metadaten, die Sicherheitskonfiguration von MCP-Servern,
die Sicherheitskonfiguration von Modell-Providern, die Sicherheitskonfiguration des Zugriffs auf
private Netzwerke, die Sicherheitskonfiguration der Gateway-Exposition, die Sicherheitskonfiguration
von Agent-Workspaces und -Tools, die konfigurierte globale und agentenspezifische
Tool-Sicherheitskonfiguration, die konfigurierte Sandbox-Laufzeit-Sicherheitskonfiguration,
die Sicherheitskonfiguration des Ingress- und Kanalzugriffs, die Sicherheitskonfiguration
der Datenverarbeitung sowie die Sicherheitskonfiguration von OpenClaw-Konfigurations-Secret-
Providern und Authentifizierungsprofilen ab.

Policy speichert erstellte Anforderungen in `policy.jsonc`, betrachtet vorhandene
OpenClaw-Einstellungen und Workspace-Deklarationen als Nachweise und meldet Abweichungen
über `openclaw policy check` und `openclaw doctor --lint`. Eine erfolgreiche Policy-
Prüfung gibt Hashes für Policy, Nachweise, Befunde und Attestierung aus, die Betreiber
für Audits aufzeichnen können.

`openclaw policy compare --baseline <file>` vergleicht eine Policy-Datei mit einer anderen
Policy-Datei. Dabei wird ausschließlich die Konformität auf Konfigurationsebene geprüft:
Anhand der Metadaten der Policy-Regeln wird verifiziert, dass die geprüfte Policy gegenüber
der erstellten Baseline weder Anforderungen vermissen lässt noch schwächere Anforderungen
enthält; Laufzeitstatus, Anmeldedaten oder Secret-Werte werden nicht geprüft.

Regeln zur Tool-Sicherheitskonfiguration können genehmigte Profile, auf den Workspace
beschränkte Dateisystem-Tools, begrenzte Einstellungen für Ausführungssicherheit,
Rückfragen und Hosts, einen deaktivierten Modus mit erhöhten Rechten, exakte
`alsoAllow`-Einträge sowie erforderliche Tool-Sperreinträge verlangen. Die Nachweise
erfassen zusätzliche `alsoAllow`-Einträge, da diese die effektive Tool-Sicherheitskonfiguration
erweitern können. Diese Prüfungen beobachten ausschließlich die Konformität der Konfiguration;
sie lesen weder den Laufzeitstatus von Genehmigungen noch fügen sie eine Durchsetzung zur Laufzeit hinzu.

Regeln zur Sandbox-Sicherheitskonfiguration können genehmigte Sandbox-Modi und -Backends
verlangen, Container-Netzwerkzugriff auf den Host und Beitritte zu Container-Namespaces
untersagen, schreibgeschützte Container-Mounts vorschreiben, Mounts von Laufzeitsockets
und nicht eingeschränkte Container-Profile untersagen sowie CDP-Quellbereiche für
Sandbox-Browser verlangen.
Diese Prüfungen beobachten ausschließlich die Konformität der Konfiguration; sie lesen
weder den Laufzeitstatus von Genehmigungen, prüfen aktive Container noch fügen sie eine
Durchsetzung zur Laufzeit hinzu.

Regeln zur Datenverarbeitung können die Schwärzung sensibler Protokolldaten verlangen,
die Erfassung von Inhalten durch Telemetrie untersagen, die Pflege der Sitzungsaufbewahrung
vorschreiben und die Speicherindizierung von Sitzungstranskripten untersagen. Diese Prüfungen
beobachten ausschließlich die Konformität der Konfiguration; sie prüfen weder Rohprotokolle,
Telemetrieexporte, Transkripte, Speicherdateien, Secrets noch personenbezogene Daten.

Benannte Policy-Geltungsbereiche unter `scopes.<scopeName>` können für den jeweils
aufgeführten Selektor strengere reguläre Policy-Abschnitte hinzufügen. `agentIds`
unterstützt `tools`, `agents.workspace`, `sandbox` und
`dataHandling.memory`; `channelIds` unterstützt `ingress.channels`.
Laufzeit-Agent-IDs, die nicht ausdrücklich in `agents.entries.*` aufgeführt sind, werden
anhand der geerbten globalen bzw. standardmäßigen Sicherheitskonfiguration geprüft, statt
ohne Nachweise stillschweigend als konform zu gelten. Jeder in `policy.jsonc` vorhandene
Geltungsbereich muss für seinen Selektor gültig und durchsetzbar sein. Overlay-Regeln stellen
zusätzliche Anforderungen dar; daher schwächen sie die übergeordnete Policy nicht und können
eigene Befunde erzeugen, wenn dieselbe beobachtete Konfiguration gegen beide Geltungsbereiche
verstößt.

<!-- openclaw-plugin-reference:manual-end -->

## Verwandte Dokumentation

- [Policy](/de/cli/policy)
