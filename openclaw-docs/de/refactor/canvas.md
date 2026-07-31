---
read_when:
    - Verlagerung der Zuständigkeit für Canvas-Host, Tools, Befehle, Dokumentation oder Protokoll
    - Prüfen, ob Canvas weiterhin dem Core zugeordnet ist
    - Vorbereiten oder Überprüfen des PRs für das experimentelle Canvas-Plugin
summary: Planungs- und Audit-Checkliste für die Verlagerung von Canvas aus dem Kern in ein gebündeltes experimentelles Plugin.
title: Canvas-Plugin-Refaktorierung
x-i18n:
    generated_at: "2026-07-26T18:04:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ead3f865ea80acb1e47f45a5ab07acf19a6470035c00c81006b2b1230bedd71e
    source_path: refactor/canvas.md
    workflow: 16
---

# Refaktorierung des Canvas-Plugins

Canvas wird wenig genutzt und ist experimentell. Behandeln Sie es als gebündeltes Plugin, nicht als Kernfunktion. Der Kern kann generische Infrastruktur für Gateway, Node, HTTP, Authentifizierung, Konfiguration und native Clients beibehalten, Canvas-spezifisches Verhalten sollte jedoch unter `extensions/canvas` angesiedelt sein.

## Ziel

Die Zuständigkeit für Canvas nach `extensions/canvas` verschieben und dabei das aktuelle Verhalten gekoppelter Nodes beibehalten:

- Das agentenseitige `canvas`-Tool wird vom Canvas-Plugin registriert
- Canvas-Node-Befehle sind nur zulässig, wenn das Canvas-Plugin sie registriert
- A2UI-Host-/Quelldateien befinden sich im Canvas-Plugin
- Die Materialisierung von Canvas-Dokumenten befindet sich im Canvas-Plugin
- Die Implementierung des CLI-Befehls befindet sich im Canvas-Plugin oder delegiert über ein Plugin-eigenes Runtime-Barrel
- Dokumentation und Plugin-Inventar beschreiben Canvas als experimentell und Plugin-gestützt

## Nichtziele

- Die Canvas-Benutzeroberfläche der nativen App darf bei dieser Refaktorierung nicht neu gestaltet werden.
- Die Canvas-Protokoll-/Client-Unterstützung darf nicht aus iOS, Android oder macOS entfernt werden, sofern nicht eine separate Produktentscheidung die Löschung von Canvas vorsieht.
- Es darf kein umfassendes Plugin-Service-Framework nur für Canvas entwickelt werden, sofern nicht mindestens ein weiteres gebündeltes Plugin dieselbe Schnittstelle benötigt.

## Aktueller Branch-Stand

Erledigt:

- Gebündeltes Plugin-Paket in `extensions/canvas` hinzugefügt.
- `extensions/canvas/openclaw.plugin.json` hinzugefügt.
- Das agentenseitige `canvas`-Tool von `src/agents/tools/canvas-tool.ts` nach `extensions/canvas/src/tool.ts` verschoben.
- Die Kernregistrierung von `createCanvasTool` aus `src/agents/openclaw-tools.ts` entfernt.
- Die Canvas-Host-Implementierung von `src/canvas-host` nach `extensions/canvas/src/host` verschoben.
- `extensions/canvas/runtime-api.ts` als Plugin-eigenes Kompatibilitäts-Barrel für Tests, Paketierung und externe öffentliche Canvas-Hilfsfunktionen beibehalten.
- Die Materialisierung von Canvas-Dokumenten von `src/gateway/canvas-documents.ts` nach `extensions/canvas/src/documents.ts` verschoben.
- Die Canvas-CLI-Implementierung und A2UI-JSONL-Hilfsfunktionen nach `extensions/canvas/src/cli.ts` verschoben.
- Die Canvas-Host-URL und Hilfsfunktionen für bereichsgebundene Fähigkeiten nach `extensions/canvas/src` verschoben.
- Die Standardwerte für Canvas-Node-Befehle aus hartcodierten Kernlisten in `nodeInvokePolicies` des Plugins verschoben.
- Plugin-eigene Canvas-Host-Konfiguration unter `plugins.entries.canvas.config.host` hinzugefügt.
- Die HTTP-Bereitstellung von Canvas und A2UI hinter die Registrierung der HTTP-Routen des Canvas-Plugins verschoben.
- Generische Plugin-WebSocket-Upgrade-Weiterleitung für Plugin-eigene HTTP-Routen hinzugefügt.
- Canvas-spezifische Gateway-Host-URL und Authentifizierung von Node-Fähigkeiten durch generische Hilfsfunktionen für gehostete Plugin-Oberflächen und Node-Fähigkeiten ersetzt.
- Plugin-eigene Resolver für gehostete Medien hinzugefügt, sodass URLs von Canvas-Dokumenten über das Canvas-Plugin aufgelöst werden, anstatt dass der Kern interne Canvas-Dokumentfunktionen importiert.
- `api.registerNodeCliFeature(...)` hinzugefügt, sodass Canvas `openclaw nodes canvas` als Plugin-eigene Node-Funktion deklarieren kann, ohne den übergeordneten Befehlspfad manuell auszuschreiben.
- Produktionsimporte von `extensions/canvas/runtime-api.js` aus `src/**` entfernt.
- Die Quelle des A2UI-Bundles von `apps/shared/OpenClawKit/Tools/CanvasA2UI` nach `extensions/canvas/src/host/a2ui-app` verschoben.
- Die Implementierung zum Erstellen/Kopieren von A2UI unter `extensions/canvas/scripts` verschoben und die Build-Verkabelung auf Root-Ebene durch generische Asset-Hooks für gebündelte Plugins ersetzt.
- Den veralteten Top-Level-Konfigurationsalias `canvasHost` aus der Runtime entfernt.
- Die Canvas-Doctor-Migration beibehalten, sodass `openclaw doctor --fix` alte `canvasHost`-Konfigurationen in `plugins.entries.canvas.config.host` umschreibt.
- Die Kompatibilität mit dem Canvas-Protokoll alter Agenten wurde hinter Gateway-Protokoll v4 entfernt. Native Clients und Gateways verwenden jetzt ausschließlich `pluginSurfaceUrls.canvas` zusammen mit `node.pluginSurface.refresh`; der veraltete Pfad über `canvasHostUrl`, `canvasCapability` und `node.canvas.capability.refresh` wird bei dieser experimentellen Refaktorierung absichtlich nicht unterstützt.
- Das generierte Plugin-Inventar wurde um Canvas ergänzt.
- Plugin-Referenzdokumentation unter `docs/plugins/reference/canvas.md` hinzugefügt.

Bekannte verbleibende Canvas-Oberflächen im Kern:

- Canvas-Handler nativer Apps unter `apps/` verwenden weiterhin absichtlich die Oberfläche des Canvas-Plugins
- Canvas-Protokoll-/Client-Handler nativer Apps unter `apps/`
- Die Ausgabe veröffentlichter Artefakte verwendet für die abwärtskompatible Runtime-Suche weiterhin `dist/canvas-host/a2ui`, der Kopierschritt ist jedoch jetzt Plugin-eigen

## Zielstruktur

`extensions/canvas` sollte für Folgendes zuständig sein:

- Plugin-Manifest und Paketmetadaten
- Registrierung des Agenten-Tools
- Richtlinie für Node-Aufrufbefehle
- Canvas-Host und A2UI-Runtime
- Quelle des Canvas-A2UI-Bundles und Skripte zum Erstellen/Kopieren von Assets
- Erstellung von Canvas-Dokumenten und Auflösung von Assets
- Canvas-CLI-Implementierung
- Canvas-Dokumentationsseite und Eintrag im Plugin-Inventar

Der Kern sollte nur für generische Schnittstellen zuständig sein:

- Erkennung und Registrierung von Plugins
- Generische Registrierung von Agenten-Tools
- Generische Richtlinienregistrierung für Node-Aufrufe
- Generische Gateway-HTTP-/Authentifizierungs- und WebSocket-Upgrade-Weiterleitung
- Generische URL-Auflösung für gehostete Plugin-Oberflächen
- Generische Registrierung von Resolvern für gehostete Medien
- Generischer Transport von Node-Fähigkeiten
- Generische Konfigurationsinfrastruktur
- Generische Erkennung von Asset-Hooks gebündelter Plugins

Native Apps können Canvas-Befehlshandler als Clients des Protokolls beibehalten. Sie sind nicht für die Plugin-Runtime zuständig.

## Migrationsschritte

1. `plugins.entries.canvas.config.host` als Plugin-eigene Konfigurationsoberfläche behandeln.
2. Die Dokumentation aktualisieren, sodass Canvas als experimentelles gebündeltes Plugin beschrieben wird.
3. Gezielte Canvas-Tests, Prüfungen des Plugin-Inventars, Prüfungen der Plugin-SDK-API sowie die durch Runtime-Grenzen betroffenen Build-/Typprüfungen ausführen.

## Audit-Checkliste

Bevor die Refaktorierung als abgeschlossen gilt:

- `rg "src/canvas-host|../canvas-host"` gibt keine aktiven Quellimporte zurück.
- `rg "canvas-tool|createCanvasTool" src` findet keine Canvas-Tool-Implementierung im Kern.
- `rg "canvas.present|canvas.snapshot|canvas.a2ui" src/gateway` findet außerhalb generischer Tests für Plugin-Richtlinien keine hartcodierten Standard-Positivlisten.
- `rg "extensions/canvas/runtime-api" src --glob '!**/*.test.ts'` ist leer.
- `rg "canvas-documents" src` ist leer.
- `rg "registerNodesCanvasCommands|nodes-canvas" src` ist leer; das Canvas-Plugin registriert `openclaw nodes canvas` über verschachtelte Plugin-CLI-Metadaten.
- `rg "createCanvasHostHandler|handleA2uiHttpRequest" src/gateway` gibt keine Zuständigkeit für die Gateway-Runtime zurück.
- `rg "apps/shared/OpenClawKit/Tools/CanvasA2UI|canvas-a2ui-copy|extensions/canvas/src/host/a2ui" scripts .github package.json` findet nur Kompatibilitäts-Wrapper oder Plugin-eigene Pfade.
- `pnpm plugins:inventory:check` ist erfolgreich.
- `pnpm plugin-sdk:api:check` ist erfolgreich, oder generierte API-Vertragsdatensätze werden absichtlich aktualisiert und geprüft.
- Gezielte Canvas-Tests sind erfolgreich.
- Tests der geänderten Lanes für Canvas-Host-/A2UI-Pfade sind erfolgreich.
- Der PR-Text besagt ausdrücklich, dass Canvas experimentell und Plugin-gestützt ist.

## Verifizierungsbefehle

Verwenden Sie während der Iteration gezielte lokale Prüfungen:

```sh
pnpm test extensions/canvas/src/host/server.test.ts extensions/canvas/src/host/server.state-dir.test.ts extensions/canvas/src/host/file-resolver.test.ts
pnpm test src/gateway/server.plugin-node-capability-auth.test.ts src/gateway/server-import-boundary.test.ts
pnpm test extensions/canvas/src/config-migration.test.ts src/commands/doctor-legacy-config.migrations.test.ts
pnpm test test/scripts/changed-lanes.test.ts test/scripts/build-all.test.ts extensions/canvas/scripts/bundle-a2ui.test.ts test/scripts/bundled-plugin-assets.test.ts extensions/canvas/scripts/copy-a2ui.test.ts src/infra/run-node.test.ts
pnpm tsgo:extensions
pnpm plugins:inventory:check
pnpm plugin-sdk:api:check
```

Führen Sie vor dem Push `pnpm build` aus, wenn sich das Runtime-Barrel, Lazy Imports, die Paketierung oder veröffentlichte Plugin-Oberflächen ändern.
