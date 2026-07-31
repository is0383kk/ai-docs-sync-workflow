---
doc-schema-version: 1
read_when:
    - Sie möchten OpenClaw-Plugins von Drittanbietern finden
    - Sie möchten Ihr eigenes Plugin auf ClawHub veröffentlichen oder auflisten.
summary: Von der Community gepflegte OpenClaw-Plugins finden und veröffentlichen
title: Community-Plugins
x-i18n:
    generated_at: "2026-07-26T18:36:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6a9eb477f20da8171a35c22ea6b112d77ff4afe0878f60314c052746aef4e0ac
    source_path: plugins/community.md
    workflow: 16
---

Community-Plugins sind Drittanbieterpakete, die OpenClaw um Kanäle, Tools, Provider, Hooks oder andere Funktionen erweitern. Verwenden Sie [ClawHub](/de/clawhub) als primäre Plattform zur Entdeckung öffentlicher Community-Plugins.

## Plugins finden

Durchsuchen Sie ClawHub über die CLI:

```bash
openclaw plugins search "calendar"
```

Installieren Sie ein ClawHub-Plugin mit einem expliziten Quellpräfix:

```bash
openclaw plugins install clawhub:<package-name>
```

npm bleibt während der Umstellung zur Einführung ein unterstützter direkter Installationsweg:

```bash
openclaw plugins install npm:<package-name>
```

Unter [Plugins verwalten](/de/plugins/manage-plugins) finden Sie gängige Beispiele zum Installieren, Aktualisieren, Prüfen und Deinstallieren. Die vollständige Befehlsreferenz und die Regeln zur Quellauswahl finden Sie unter [`openclaw plugins`](/de/cli/plugins).

## Plugins veröffentlichen

Veröffentlichen Sie öffentliche Community-Plugins auf ClawHub, damit OpenClaw-Benutzer sie entdecken und installieren können. ClawHub verwaltet die aktuelle Paketliste, den Veröffentlichungsverlauf, den Scanstatus und die Installationshinweise; die Dokumentation führt keinen statischen Katalog von Drittanbieter-Plugins.

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

Stellen Sie vor der Veröffentlichung sicher, dass das Plugin über Paketmetadaten, ein Plugin-Manifest, eine Einrichtungsdokumentation und einen eindeutig benannten Wartungsverantwortlichen verfügt. ClawHub validiert den Eigentümerbereich, den Paketnamen, die Version, die Dateilimits und die Quellmetadaten, bevor eine Veröffentlichung erstellt wird. Anschließend bleiben neue Veröffentlichungen auf den regulären Installations- und Downloadoberflächen verborgen, bis die Prüfung und Verifizierung abgeschlossen sind.

Checkliste vor der Veröffentlichung:

| Anforderung                 | Grund                                                   |
| --------------------------- | ------------------------------------------------------- |
| Auf ClawHub veröffentlicht  | Benutzer benötigen funktionierende `openclaw plugins install`-Hinweise |
| Öffentliches GitHub-Repository | Quellcodeprüfung, Problemverfolgung, Transparenz      |
| Einrichtungs- und Nutzungsdokumentation | Benutzer müssen wissen, wie das Plugin konfiguriert wird |
| Aktive Wartung              | Kürzliche Aktualisierungen oder schnelle Bearbeitung von Problemen |

Vollständiger Veröffentlichungsvertrag:

- [Veröffentlichen auf ClawHub](/de/clawhub/publishing) - Eigentümer, Bereiche, Veröffentlichungen,
  Prüfung, Paketvalidierung und Paketübertragung
- [Plugins erstellen](/de/plugins/building-plugins) - die Struktur des Plugin-Pakets
  und der Ablauf der ersten Veröffentlichung
- [Plugin-Manifest](/de/plugins/manifest) - native Felder des Plugin-Manifests

## Verwandte Themen

- [Plugins](/de/tools/plugin) - installieren, konfigurieren, neu starten und Fehler beheben
- [Plugins verwalten](/de/plugins/manage-plugins) - Befehlsbeispiele
- [Veröffentlichen auf ClawHub](/de/clawhub/publishing) - Veröffentlichungs- und Release-Regeln
