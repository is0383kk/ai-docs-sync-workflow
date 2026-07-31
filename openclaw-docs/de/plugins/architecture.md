---
read_when:
    - Native OpenClaw-Plugins erstellen oder debuggen
    - Das Plugin-Fähigkeitsmodell oder die Zuständigkeitsgrenzen verstehen
    - Arbeit an der Plugin-Ladepipeline oder Registry
    - Implementierung von Provider-Runtime-Hooks oder Kanal-Plugins
sidebarTitle: Internals
summary: 'Plugin-Interna: Fähigkeitsmodell, Zuständigkeiten, Verträge, Lade-Pipeline und Laufzeithilfen'
title: Plugin-Interna
x-i18n:
    generated_at: "2026-07-26T17:56:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d47551b1bc2f71ce2ade3dfdd14bff8ee187616c3807f8101c1a3236e1443cc1
    source_path: plugins/architecture.md
    workflow: 16
---

Dies ist die **ausführliche Architekturreferenz** für das Plugin-System von OpenClaw. Praktische Anleitungen finden Sie auf den folgenden themenspezifischen Seiten.

<CardGroup cols={2}>
  <Card title="Plugins installieren und verwenden" icon="plug" href="/de/tools/plugin">
    Anleitung für Endbenutzer zum Hinzufügen, Aktivieren und Beheben von Problemen mit Plugins.
  </Card>
  <Card title="Plugins entwickeln" icon="rocket" href="/de/plugins/building-plugins">
    Tutorial für das erste Plugin mit dem kleinstmöglichen funktionsfähigen Manifest.
  </Card>
  <Card title="Kanal-Plugins" icon="comments" href="/de/plugins/sdk-channel-plugins">
    Entwickeln Sie ein Plugin für einen Messaging-Kanal.
  </Card>
  <Card title="Provider-Plugins" icon="microchip" href="/de/plugins/sdk-provider-plugins">
    Entwickeln Sie ein Plugin für einen Modell-Provider.
  </Card>
  <Card title="SDK-Übersicht" icon="book" href="/de/plugins/sdk-overview">
    Referenz zur Importzuordnung und Registrierungs-API.
  </Card>
</CardGroup>

## Öffentliches Fähigkeitsmodell

Fähigkeiten bilden das öffentliche Modell für **native Plugins** innerhalb von OpenClaw. Jedes native OpenClaw-Plugin registriert sich für einen oder mehrere Fähigkeitstypen:

| Fähigkeit                  | Registrierungsmethode                           | Beispiel-Plugins                                            |
| -------------------------- | ------------------------------------------------ | ----------------------------------------------------------- |
| Textinferenz               | `api.registerProvider(...)`                      | `anthropic`, `openai`                                       |
| CLI-Inferenz-Backend       | `api.registerCliBackend(...)`                    | `anthropic`, `openai`                                       |
| Embeddings                 | `api.registerEmbeddingProvider(...)`             | Provider-eigene Vektor-Plugins                              |
| Sprache                    | `api.registerSpeechProvider(...)`                | `elevenlabs`, `microsoft`                                   |
| Echtzeittranskription      | `api.registerRealtimeTranscriptionProvider(...)` | `openai`                                                    |
| Echtzeitsprache            | `api.registerRealtimeVoiceProvider(...)`         | `google`, `openai`                                          |
| Medienverständnis          | `api.registerMediaUnderstandingProvider(...)`    | `google`, `openai`                                          |
| Transkriptquelle           | `api.registerTranscriptSourceProvider(...)`      | `discord`, `google-meet`, `teams-meetings`, `zoom-meetings` |
| Bilderzeugung              | `api.registerImageGenerationProvider(...)`       | `fal`, `google`, `openai`                                   |
| Musikerzeugung             | `api.registerMusicGenerationProvider(...)`       | `fal`, `google`, `minimax`                                  |
| Videoerzeugung             | `api.registerVideoGenerationProvider(...)`       | `fal`, `google`, `qwen`                                     |
| Webabruf                   | `api.registerWebFetchProvider(...)`              | `firecrawl`                                                 |
| Websuche                   | `api.registerWebSearchProvider(...)`             | `brave`, `firecrawl`, `google`                              |
| Kanal/Messaging            | `api.registerChannel(...)`                       | `matrix`, `msteams`                                         |
| Gateway-Erkennung          | `api.registerGatewayDiscoveryService(...)`       | `bonjour`                                                   |

<Note>
Ein Plugin, das keine Fähigkeiten registriert, aber Hooks, Tools, Erkennungsdienste oder Hintergrunddienste bereitstellt, ist ein **veraltetes reines Hook-Plugin**. Dieses Muster wird weiterhin vollständig unterstützt.
</Note>

### Haltung zur externen Kompatibilität

Das Fähigkeitsmodell ist im Kern implementiert und wird derzeit von gebündelten und nativen Plugins verwendet. Für die Kompatibilität externer Plugins muss jedoch ein strengerer Maßstab gelten als „es wird exportiert und ist daher unveränderlich“.

| Plugin-Situation                                      | Empfehlung                                                                                                             |
| ----------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| Bestehende externe Plugins                            | Hook-basierte Integrationen müssen weiterhin funktionieren; dies ist die Kompatibilitätsbasis.                         |
| Neue gebündelte/native Plugins                        | Bevorzugen Sie eine explizite Fähigkeitsregistrierung gegenüber anbieterspezifischen Zugriffen oder neuen reinen Hook-Konzepten. |
| Externe Plugins mit Fähigkeitsregistrierung           | Zulässig, aber behandeln Sie fähigkeitsspezifische Hilfsschnittstellen als in Entwicklung, sofern die Dokumentation sie nicht als stabil kennzeichnet. |

Die Fähigkeitsregistrierung ist die vorgesehene Entwicklungsrichtung. Veraltete Hooks bleiben während des Übergangs für externe Plugins der sicherste Weg ohne Kompatibilitätsbrüche. Exportierte Hilfsunterpfade sind nicht alle gleichwertig — bevorzugen Sie eng gefasste dokumentierte Verträge gegenüber beiläufigen Hilfsexporten.

### Plugin-Formen

OpenClaw ordnet jedes geladene Plugin anhand seines tatsächlichen Registrierungsverhaltens einer Form zu (nicht nur anhand statischer Metadaten):

<AccordionGroup>
  <Accordion title="plain-capability">
    Registriert genau einen Fähigkeitstyp (beispielsweise ein reines Provider-Plugin wie `arcee` oder `chutes`).
  </Accordion>
  <Accordion title="hybrid-capability">
    Registriert mehrere Fähigkeitstypen (beispielsweise ist `openai` für Textinferenz, Sprache, Medienverständnis und Bilderzeugung zuständig).
  </Accordion>
  <Accordion title="hook-only">
    Registriert ausschließlich Hooks (typisiert oder benutzerdefiniert), jedoch keine Fähigkeiten, Tools, Befehle oder Dienste.
  </Accordion>
  <Accordion title="non-capability">
    Registriert Tools, Befehle, Dienste oder Routen, jedoch keine Fähigkeiten.
  </Accordion>
</AccordionGroup>

Verwenden Sie `openclaw plugins inspect <id>`, um die Form und die Aufschlüsselung der Fähigkeiten eines Plugins anzuzeigen. Weitere Informationen finden Sie in der [CLI-Referenz](/de/cli/plugins#inspect).

### Kompatibilitätssignale

`openclaw doctor`, `openclaw plugins inspect <id>`, `openclaw status --all` und `openclaw plugins doctor` zeigen die folgenden Kompatibilitätshinweise an:

| Signal                                     | Bedeutung                                                                                                                    |
| ------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------- |
| **Konfiguration gültig**                   | Die Konfiguration wird fehlerfrei geparst und die Plugins werden aufgelöst                                                   |
| **nur Hooks** (Info)                       | Das Plugin registriert ausschließlich Hooks; dies ist ein unterstützter Weg, wurde jedoch noch nicht auf die Fähigkeitsregistrierung migriert |
| **veraltete Memory-Embedding-API** (Warnung) | Ein nicht gebündeltes Plugin verwendet anstelle von `registerEmbeddingProvider` die alte Memory-spezifische Embedding-Provider-API |
| **schwerwiegender Fehler**                 | Die Konfiguration ist ungültig oder das Plugin konnte nicht geladen werden                                                   |

Keines der Hinweis- oder Warnsignale beeinträchtigt Ihr Plugin derzeit. Diese Signale werden auch in `openclaw status --all` und `openclaw plugins doctor` angezeigt.

## Architekturübersicht

Das Plugin-System von OpenClaw besteht aus vier Ebenen:

<Steps>
  <Step title="Manifest und Erkennung">
    OpenClaw findet potenzielle Plugins in konfigurierten Pfaden, Workspace-Stammverzeichnissen, globalen Plugin-Stammverzeichnissen und unter den gebündelten Plugins. Bei der Erkennung werden zuerst native `openclaw.plugin.json`-Manifeste sowie unterstützte Bundle-Manifeste gelesen.
  </Step>
  <Step title="Aktivierung und Validierung">
    Der Kern entscheidet, ob ein erkanntes Plugin aktiviert, deaktiviert, blockiert oder für einen exklusiven Platz wie Memory ausgewählt wird.
  </Step>
  <Step title="Laden zur Laufzeit">
    Native OpenClaw-Plugins werden prozessintern geladen und registrieren Fähigkeiten in einer zentralen Registry. Paketiertes JavaScript wird über natives `require` geladen; lokaler TypeScript-Quellcode von Drittanbietern verwendet als Notfalllösung Jiti. Kompatible Bundles werden in Registry-Einträge normalisiert, ohne Laufzeitcode zu importieren.
  </Step>
  <Step title="Nutzung der Oberflächen">
    Der übrige Teil von OpenClaw liest die Registry, um Tools, Kanäle, Provider-Einrichtung, Hooks, HTTP-Routen, CLI-Befehle und Dienste bereitzustellen.
  </Step>
</Steps>

Speziell für die Plugin-CLI ist die Erkennung von Stammbefehlen in zwei Phasen aufgeteilt:

- Metadaten zur Parse-Zeit stammen aus `registerCli(..., { descriptors: [...] })`
- das eigentliche CLI-Modul des Plugins kann verzögert bleiben und sich beim ersten Aufruf registrieren

Dadurch verbleibt der CLI-Code des Plugins im Plugin, während OpenClaw dennoch vor dem Parsen Namen für Stammbefehle reservieren kann.

Die wichtige Entwurfsgrenze:

- Die Manifest- und Konfigurationsvalidierung sollte anhand von **Manifest-/Schema-Metadaten** funktionieren, ohne Plugin-Code auszuführen
- Die native Fähigkeitserkennung darf vertrauenswürdigen Plugin-Einstiegscode laden, um einen nicht aktivierenden Registry-Snapshot zu erstellen
- Das native Laufzeitverhalten stammt aus dem `register(api)`-Pfad des Plugin-Moduls mit `api.registrationMode === "full"`

Durch diese Trennung kann OpenClaw die Konfiguration validieren, fehlende oder deaktivierte Plugins erläutern und Hinweise für Benutzeroberfläche und Schema erstellen, bevor die vollständige Laufzeit aktiv ist.

### Snapshot der Plugin-Metadaten und Nachschlagetabelle

Beim Start des Gateways wird ein `PluginMetadataSnapshot` für den aktuellen Konfigurations-Snapshot erstellt. Der Snapshot enthält ausschließlich Metadaten: Er speichert den Index der installierten Plugins, die Manifest-Registry, Manifestdiagnosen, Zuordnungen der Zuständigkeiten, einen Normalisierer für Plugin-IDs und Manifesteinträge. Er enthält keine geladenen Plugin-Module, Provider-SDKs, Paketinhalte oder Laufzeitexporte.

Die Plugin-bezogene Konfigurationsvalidierung, die automatische Aktivierung beim Start und der Plugin-Bootstrap des Gateways verwenden diesen Snapshot, anstatt Manifest- und Indexmetadaten unabhängig voneinander neu zu erstellen. `PluginLookUpTable` wird aus demselben Snapshot abgeleitet und ergänzt den Plugin-Startplan für die aktuelle Laufzeitkonfiguration.

Nach dem Start behält das Gateway den aktuellen Metadaten-Snapshot als austauschbares Laufzeitprodukt bei. Wiederholte Provider-Erkennungen zur Laufzeit können diesen Snapshot verwenden, anstatt den Index der installierten Plugins und die Manifest-Registry bei jedem Durchlauf des Provider-Katalogs neu zu erstellen. Beim Herunterfahren des Gateways, bei Änderungen an der Konfiguration oder am Plugin-Bestand sowie beim Schreiben des installierten Index wird der Snapshot gelöscht oder ersetzt; Aufrufer greifen auf den kalten Manifest-/Indexpfad zurück, wenn kein kompatibler aktueller Snapshot vorhanden ist. Kompatibilitätsprüfungen müssen Plugin-Erkennungsstammverzeichnisse wie `plugins.load.paths` und den standardmäßigen Agenten-Workspace einbeziehen, da Workspace-Plugins zum Metadatenumfang gehören.

Der Snapshot und die Nachschlagetabelle halten wiederholte Startentscheidungen auf dem schnellen Pfad:

- Kanalzuständigkeit
- verzögerter Kanalstart
- Plugin-IDs beim Start
- Zuständigkeit für Provider und CLI-Backends
- Zuständigkeit für Einrichtungs-Provider, Befehlsalias, Modellkatalog-Provider und Manifestvertrag
- Validierung des Plugin-Konfigurationsschemas und des Kanalkonfigurationsschemas
- Entscheidungen zur automatischen Aktivierung beim Start

Die Sicherheitsgrenze besteht im Ersetzen des Snapshots, nicht in seiner Mutation. Erstellen Sie den Snapshot neu, wenn sich die Konfiguration, der Plugin-Bestand, Installationsdatensätze oder persistierte Indexrichtlinien ändern. Behandeln Sie ihn nicht als umfassende veränderliche globale Registry und bewahren Sie keine unbegrenzte Anzahl historischer Snapshots auf. Das Laden von Plugins zur Laufzeit bleibt von Metadaten-Snapshots getrennt, damit veralteter Laufzeitstatus nicht hinter einem Metadaten-Cache verborgen werden kann.

Die Cache-Regel ist unter [Interne Plugin-Architektur](/de/plugins/architecture-internals#plugin-cache-boundary) dokumentiert: Manifest- und Erkennungsmetadaten sind aktuell, sofern ein Aufrufer nicht einen expliziten Snapshot, eine Nachschlagetabelle oder eine Manifest-Registry für den aktuellen Ablauf vorhält. Verborgene Metadaten-Caches und zeitbasierte TTLs sind nicht Bestandteil des Ladens von Plugins. Nur Caches für Laufzeitlader, Module und Abhängigkeitsartefakte dürfen fortbestehen, nachdem Code oder installierte Artefakte tatsächlich geladen wurden.

Einige Cold-Path-Aufrufer rekonstruieren Manifest-Registrys weiterhin direkt aus dem persistent gespeicherten Index installierter Plugins, anstatt eine Gateway-`PluginLookUpTable` zu erhalten. Dieser Pfad rekonstruiert die Registry nun bei Bedarf; übergeben Sie vorzugsweise die aktuelle Lookup-Tabelle oder eine explizite Manifest-Registry durch die Runtime-Abläufe, wenn einem Aufrufer bereits eine vorliegt.

### Aktivierungsplanung

Die Aktivierungsplanung ist Teil der Steuerungsebene. Aufrufer können vor dem Laden umfassenderer Runtime-Registrys ermitteln, welche Plugins für einen konkreten Befehl, Provider, Kanal, eine Route, ein Agent-Harness oder eine Fähigkeit relevant sind.

Der Planer wahrt die Kompatibilität mit dem aktuellen Manifestverhalten:

- `activation.*`-Felder sind explizite Hinweise für den Planer
- `providers`, `channels`, `commandAliases`, `setup.providers`, `contracts.tools` und Hooks bleiben der Fallback für die Manifestzuständigkeit
- die Planer-API, die nur IDs zurückgibt, bleibt für bestehende Aufrufer verfügbar
- die Plan-API meldet Begründungsbezeichnungen, damit die Diagnose zwischen expliziten Hinweisen und dem Zuständigkeits-Fallback unterscheiden kann

<Warning>
Behandeln Sie `activation` nicht als Lifecycle-Hook oder Ersatz für `register(...)`. Es handelt sich um Metadaten zur Eingrenzung des Ladevorgangs. Bevorzugen Sie Zuständigkeitsfelder, wenn sie die Beziehung bereits beschreiben; verwenden Sie `activation` nur für zusätzliche Hinweise an den Planer.
</Warning>

### Kanal-Plugins und das gemeinsame Nachrichtenwerkzeug

Kanal-Plugins müssen für normale Chataktionen kein separates Werkzeug zum Senden, Bearbeiten oder Reagieren registrieren. OpenClaw hält ein gemeinsames `message`-Werkzeug im Core vor, während Kanal-Plugins die dahinterliegende kanalspezifische Ermittlung und Ausführung übernehmen.

Die aktuelle Abgrenzung lautet:

- der Core übernimmt den Host des gemeinsamen `message`-Werkzeugs, die Prompt-Verkabelung, die Sitzungs-/Thread-Verwaltung und die Ausführungsverteilung
- Kanal-Plugins übernehmen die bereichsbezogene Aktionsermittlung, Fähigkeitsermittlung und alle kanalspezifischen Schemafragmente
- Kanal-Plugins übernehmen die providerspezifische Grammatik für Sitzungskonversationen, etwa wie Konversations-IDs Thread-IDs kodieren oder von übergeordneten Konversationen erben
- Kanal-Plugins führen die abschließende Aktion über ihren Aktionsadapter aus

Für Kanal-Plugins ist die SDK-Oberfläche `ChannelMessageActionAdapter.describeMessageTool(...)`. Mit diesem einheitlichen Ermittlungsaufruf kann ein Plugin seine sichtbaren Aktionen, Fähigkeiten und Schemabeiträge gemeinsam zurückgeben, sodass diese Bestandteile nicht auseinanderlaufen.

Namen von Nachrichtenaktionen verwenden bewusst ein geschlossenes, vom Core verwaltetes Vokabular, damit jeder Transport jede Aktion darstellen kann. Plugins fügen Aktionsnamen über einen Core-PR hinzu; eine Registrierung zur Laufzeit wird absichtlich nicht unterstützt.

Wenn ein kanalspezifischer Parameter des Nachrichtenwerkzeugs eine Medienquelle wie einen lokalen Pfad oder eine Remote-Medien-URL enthält, sollte das Plugin außerdem `mediaSourceParams` aus `describeMessageTool(...)` zurückgeben. Der Core verwendet diese explizite Liste, um die Normalisierung von Sandbox-Pfaden und Hinweise für den ausgehenden Medienzugriff anzuwenden, ohne Parameternamen fest zu kodieren, die dem Plugin gehören. Bevorzugen Sie dort aktionsbezogene Zuordnungen statt einer einzigen kanalweiten flachen Liste, damit ein ausschließlich für Profile bestimmter Medienparameter nicht bei unabhängigen Aktionen wie `send` normalisiert wird.

Der Core übergibt den Runtime-Kontext an diesen Ermittlungsschritt. Zu den wichtigen Feldern gehören:

- `accountId`
- `currentChannelId`
- `currentThreadTs`
- `currentMessageId`
- `sessionKey`
- `sessionId`
- `agentId`
- vertrauenswürdige eingehende `requesterSenderId`

Dies ist für kontextsensitive Plugins wichtig. Ein Kanal kann Nachrichtenaktionen abhängig vom aktiven Konto, aktuellen Raum/Thread/der aktuellen Nachricht oder der Identität des vertrauenswürdigen Anfragenden ausblenden oder bereitstellen, ohne kanalspezifische Verzweigungen im zentralen `message`-Werkzeug fest zu kodieren.

Deshalb bleiben Routingänderungen für eingebettete Runner Plugin-Arbeit: Der Runner ist dafür verantwortlich, die aktuelle Chat-/Sitzungsidentität an die Ermittlungsgrenze des Plugins weiterzuleiten, damit das gemeinsame `message`-Werkzeug für den aktuellen Durchlauf die richtige, vom Kanal verwaltete Oberfläche bereitstellt.

Für kanaleigene Ausführungshilfen sollten Kanal-Plugins die Ausführungs-Runtime innerhalb ihrer eigenen Plugin-Module halten. Der Core verwaltet die Runtimes für Nachrichtenaktionen von Discord, Slack, Telegram oder WhatsApp unter `src/agents/tools` nicht mehr. Wir veröffentlichen keine separaten `plugin-sdk/*-action-runtime`-Unterpfade, und diese Plugins sollten ihren eigenen lokalen Runtime-Code direkt aus ihren Plugin-eigenen Modulen importieren.

Dieselbe Abgrenzung gilt allgemein für nach Providern benannte SDK-Übergänge: Der Core sollte keine kanalspezifischen Convenience-Barrels für Discord, Signal, Slack, WhatsApp oder ähnliche Plugins importieren. Wenn der Core ein Verhalten benötigt, sollte er entweder das eigene `api.ts`- / `runtime-api.ts`-Barrel des gebündelten Plugins verwenden oder den Bedarf in eine eng gefasste generische Fähigkeit im gemeinsamen SDK überführen.

Für gebündelte Plugins gilt dieselbe Regel. Das `runtime-api.ts` eines gebündelten Plugins sollte nicht seine eigene markenspezifische `openclaw/plugin-sdk/<plugin-id>`-Fassade erneut exportieren. Diese markenspezifischen Fassaden bleiben Kompatibilitäts-Shims für externe Plugins und ältere Nutzer, gebündelte Plugins sollten jedoch lokale Exporte sowie eng gefasste generische SDK-Unterpfade wie `openclaw/plugin-sdk/channel-policy`, `openclaw/plugin-sdk/runtime-store` oder `openclaw/plugin-sdk/webhook-ingress` verwenden. Neuer Code sollte keine Plugin-ID-spezifischen SDK-Fassaden hinzufügen, sofern die Kompatibilitätsgrenze eines bestehenden externen Ökosystems dies nicht erfordert.

Speziell für Umfragen gibt es zwei Ausführungspfade:

- `outbound.sendPoll` ist die gemeinsame Basis für Kanäle, die dem allgemeinen Umfragemodell entsprechen
- `actions.handleAction("poll")` ist der bevorzugte Pfad für kanalspezifische Umfragesemantik oder zusätzliche Umfrageparameter

Der Core verschiebt die gemeinsame Umfrageanalyse nun, bis die Plugin-Umfrageverteilung die Aktion ablehnt. Dadurch können Plugin-eigene Umfragehandler kanalspezifische Umfragefelder akzeptieren, ohne zuvor vom generischen Umfrageparser blockiert zu werden.

Die vollständige Startsequenz finden Sie unter [Interna der Plugin-Architektur](/de/plugins/architecture-internals).

## Zuständigkeitsmodell für Fähigkeiten

OpenClaw behandelt ein natives Plugin als Zuständigkeitsgrenze für ein **Unternehmen** oder eine **Funktion**, nicht als Sammelsurium voneinander unabhängiger Integrationen.

Das bedeutet:

- ein Unternehmens-Plugin sollte in der Regel alle OpenClaw-seitigen Oberflächen dieses Unternehmens verwalten
- ein Funktions-Plugin sollte in der Regel die gesamte von ihm eingeführte Funktionsoberfläche verwalten
- Kanäle sollten gemeinsame Core-Fähigkeiten verwenden, statt Providerverhalten ad hoc neu zu implementieren

<AccordionGroup>
  <Accordion title="Provider mit mehreren Fähigkeiten">
    `google` verwaltet Textinferenz, CLI-Backend, Embeddings, Sprache, Echtzeitsprachkommunikation, Medienverständnis, Bild-/Musik-/Videogenerierung und Websuche. `openai` verwaltet Textinferenz, Embeddings, Sprache, Echtzeittranskription, Echtzeitsprachkommunikation, Medienverständnis sowie Bild-/Videogenerierung. `minimax` verwaltet Textinferenz sowie Medienverständnis, Sprache, Bild-/Musik-/Videogenerierung und Websuche.
  </Accordion>
  <Accordion title="Provider mit einer einzelnen Fähigkeit">
    `arcee` und `chutes` verwalten ausschließlich Textinferenz; `microsoft` verwaltet ausschließlich Sprache. Ein Provider-Plugin kann so eng gefasst bleiben, bis es weitere Bereiche dieses Providers abdecken muss.
  </Accordion>
  <Accordion title="Funktions-Plugin">
    `voice-call` verwaltet Anruftransport, Werkzeuge, CLI, Routen und die Überbrückung von Twilio-Medienstreams, verwendet jedoch gemeinsame Fähigkeiten für Sprache, Echtzeittranskription und Echtzeitsprachkommunikation, statt Provider-Plugins direkt zu importieren.
  </Accordion>
</AccordionGroup>

Der angestrebte Endzustand ist:

- die OpenClaw-seitige Oberfläche eines Providers befindet sich in einem Plugin, selbst wenn sie Textmodelle, Sprache, Bilder und Video umfasst
- andere Provider können dasselbe für ihren jeweiligen Oberflächenbereich tun
- Kanäle müssen nicht wissen, welches Provider-Plugin den Provider verwaltet; sie verwenden den vom Core bereitgestellten gemeinsamen Fähigkeitsvertrag

Dies ist die entscheidende Unterscheidung:

- **Plugin** = Zuständigkeitsgrenze
- **Fähigkeit** = Core-Vertrag, den mehrere Plugins implementieren oder verwenden können

Wenn OpenClaw also eine neue Domäne wie Video hinzufügt, lautet die erste Frage nicht: „Welcher Provider sollte die Videoverarbeitung fest kodieren?“ Die erste Frage lautet: „Wie sieht der Core-Vertrag für die Videofähigkeit aus?“ Sobald dieser Vertrag vorhanden ist, können Provider-Plugins Implementierungen dafür registrieren und Kanal-/Funktions-Plugins ihn verwenden.

Wenn die Fähigkeit noch nicht vorhanden ist, ist das richtige Vorgehen normalerweise:

<Steps>
  <Step title="Fähigkeit definieren">
    Definieren Sie die fehlende Fähigkeit im Core.
  </Step>
  <Step title="Über das SDK bereitstellen">
    Stellen Sie sie typisiert über die Plugin-API/-Runtime bereit.
  </Step>
  <Step title="Nutzer anbinden">
    Binden Sie Kanäle/Funktionen an diese Fähigkeit an.
  </Step>
  <Step title="Providerimplementierungen">
    Lassen Sie Provider-Plugins Implementierungen registrieren.
  </Step>
</Steps>

Dadurch bleibt die Zuständigkeit explizit, während Core-Verhalten vermieden wird, das von einem einzelnen Provider oder einem einmaligen Plugin-spezifischen Codepfad abhängt.

### Schichtung der Fähigkeiten

Verwenden Sie dieses Denkmodell, wenn Sie entscheiden, wohin Code gehört:

<Tabs>
  <Tab title="Core-Fähigkeitsschicht">
    Gemeinsame Orchestrierung, Richtlinien, Fallback, Regeln für das Zusammenführen von Konfigurationen, Zustellungssemantik und typisierte Verträge.
  </Tab>
  <Tab title="Provider-Plugin-Schicht">
    Providerspezifische APIs, Authentifizierung, Modellkataloge, Sprachsynthese, Bildgenerierung, Video-Backends und Nutzungsendpunkte.
  </Tab>
  <Tab title="Kanal-/Funktions-Plugin-Schicht">
    Integration von Discord/Slack/Sprachanrufen usw., die Core-Fähigkeiten verwendet und sie auf einer Oberfläche bereitstellt.
  </Tab>
</Tabs>

TTS folgt beispielsweise diesem Muster:

- der Core verwaltet die TTS-Richtlinie zum Antwortzeitpunkt, die Fallback-Reihenfolge, Einstellungen und Kanalzustellung
- `elevenlabs`, `google`, `microsoft` und `openai` verwalten die Syntheseimplementierungen
- `voice-call` verwendet die Runtime-Hilfe für Telefonie-TTS

Dasselbe Muster sollte für zukünftige Fähigkeiten bevorzugt werden.

### Beispiel für ein Unternehmens-Plugin mit mehreren Fähigkeiten

Ein Unternehmens-Plugin sollte nach außen geschlossen wirken. Wenn OpenClaw gemeinsame Verträge für Modelle, Sprache, Echtzeittranskription, Echtzeitsprachkommunikation, Medienverständnis, Bildgenerierung, Videogenerierung, Webabruf und Websuche besitzt, kann ein Provider alle seine Oberflächen an einem Ort verwalten:

```ts
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { exampleAiMedia } from "./exampleai-media.js";

export default definePluginEntry({
  id: "exampleai",
  name: "ExampleAI",
  description: "ExampleAI-Modelle und Medienfähigkeiten.",
  register(api) {
    api.registerProvider({
      id: "exampleai",
      // Hooks für Authentifizierung, Modellkatalog und Runtime
    });

    api.registerSpeechProvider({
      id: "exampleai",
      // Sprachkonfiguration des Providers — die SpeechProviderPlugin-Schnittstelle direkt implementieren
    });

    api.registerMediaUnderstandingProvider({
      id: "exampleai",
      capabilities: ["image", "audio", "video"],
      describeImage: (req) => exampleAiMedia.describeImage(req),
      transcribeAudio: (req) => exampleAiMedia.transcribeAudio(req),
      describeVideo: (req) => exampleAiMedia.describeVideo(req),
    });

    api.registerWebSearchProvider({
      id: "exampleai-search",
      createTool() {
        // Das vom Provider verwaltete Websuchwerkzeug zurückgeben.
      },
    });
  },
});
```

Entscheidend sind nicht die genauen Namen der Hilfen, sondern die Struktur:

- ein Plugin verwaltet die Provideroberfläche
- der Core verwaltet weiterhin die Fähigkeitsverträge
- die Übersetzung von Provideranfragen und HTTP-Hilfen verbleiben im Provider-Plugin
- Kanäle und Funktions-Plugins verwenden `api.runtime.*`-Hilfen, nicht Providercode
- Vertragstests können sicherstellen, dass das Plugin die Fähigkeiten registriert hat, für die es die Zuständigkeit beansprucht

### Fähigkeitsbeispiel: Videoverständnis

OpenClaw behandelt das Verständnis von Bildern, Audio und Video bereits als eine gemeinsame Fähigkeit. Dort gilt dasselbe Zuständigkeitsmodell:

<Steps>
  <Step title="Core definiert den Vertrag">
    Core definiert den Vertrag für das Medienverständnis.
  </Step>
  <Step title="Provider-Plugins registrieren sich">
    Provider-Plugins registrieren je nach Bedarf `describeImage`, `transcribeAudio` und `describeVideo`.
  </Step>
  <Step title="Nutzer verwenden das gemeinsame Verhalten">
    Kanäle und Funktions-Plugins nutzen das gemeinsame Core-Verhalten, anstatt direkt an Provider-Code angebunden zu werden.
  </Step>
</Steps>

Dadurch werden die Annahmen eines einzelnen Providers zur Videoverarbeitung nicht fest in Core integriert. Das Plugin besitzt die Provider-Oberfläche; Core besitzt den Fähigkeitsvertrag und das Fallback-Verhalten.

Die Videogenerierung verwendet bereits dieselbe Abfolge: Core besitzt den typisierten Fähigkeitsvertrag und die Laufzeithilfe, und Provider-Plugins registrieren dafür `api.registerVideoGenerationProvider(...)`-Implementierungen.

Benötigen Sie eine konkrete Checkliste für die Einführung? Siehe [Fähigkeitskochbuch](/de/plugins/adding-capabilities).

## Verträge und Durchsetzung

Die Plugin-API-Oberfläche ist absichtlich typisiert und in `OpenClawPluginApi` zentralisiert. Dieser Vertrag definiert die unterstützten Registrierungspunkte und die Laufzeithilfen, auf die sich ein Plugin verlassen darf.

Warum das wichtig ist:

- Plugin-Autoren erhalten einen einheitlichen stabilen internen Standard
- Core kann doppelte Zuständigkeiten ablehnen, beispielsweise wenn zwei Plugins dieselbe Provider-ID registrieren
- Beim Start können umsetzbare Diagnosen für fehlerhafte Registrierungen angezeigt werden
- Vertragstests können die Zuständigkeit gebündelter Plugins durchsetzen und unbemerkte Abweichungen verhindern

Die Durchsetzung erfolgt auf zwei Ebenen:

<AccordionGroup>
  <Accordion title="Durchsetzung bei der Laufzeitregistrierung">
    Die Plugin-Registry validiert Registrierungen beim Laden der Plugins. Beispiele: Doppelte Provider-IDs, doppelte Sprach-Provider-IDs und fehlerhafte Registrierungen erzeugen Plugin-Diagnosen anstelle von undefiniertem Verhalten.
  </Accordion>
  <Accordion title="Vertragstests">
    Gebündelte Plugins werden während Testläufen in Vertrags-Registries erfasst, damit OpenClaw die Zuständigkeit ausdrücklich überprüfen kann. Derzeit wird dies für Modell-Provider, Sprach-Provider, Websuch-Provider und die Zuständigkeit für gebündelte Registrierungen verwendet.
  </Accordion>
</AccordionGroup>

In der Praxis bedeutet dies, dass OpenClaw von Anfang an weiß, welches Plugin für welche Oberfläche zuständig ist. Dadurch können Core und Kanäle nahtlos zusammenarbeiten, weil die Zuständigkeit deklariert, typisiert und testbar statt implizit ist.

### Was in einen Vertrag gehört

<Tabs>
  <Tab title="Gute Verträge">
    - typisiert
    - klein
    - fähigkeitsspezifisch
    - im Besitz von Core
    - durch mehrere Plugins wiederverwendbar
    - durch Kanäle/Funktionen ohne Kenntnis des Providers nutzbar

  </Tab>
  <Tab title="Schlechte Verträge">
    - Provider-spezifische Richtlinien, die in Core verborgen sind
    - einmalige Plugin-Auswege, die die Registry umgehen
    - Kanalcode, der direkt auf eine Provider-Implementierung zugreift
    - Ad-hoc-Laufzeitobjekte, die nicht Teil von `OpenClawPluginApi` oder `api.runtime` sind

  </Tab>
</Tabs>

Im Zweifel sollte die Abstraktionsebene erhöht werden: Definieren Sie zuerst die Fähigkeit und lassen Sie anschließend Plugins daran anknüpfen.

## Ausführungsmodell

Native OpenClaw-Plugins werden **innerhalb des Prozesses** zusammen mit dem Gateway ausgeführt. Sie befinden sich nicht in einer Sandbox. Ein geladenes natives Plugin hat dieselbe Vertrauensgrenze auf Prozessebene wie Core-Code.

<Warning>
Folgen nativer Plugins: Ein Plugin kann Tools, Netzwerk-Handler, Hooks und Dienste registrieren; ein Plugin-Fehler kann das Gateway zum Absturz bringen oder destabilisieren; und ein bösartiges natives Plugin entspricht der Ausführung beliebigen Codes innerhalb des OpenClaw-Prozesses.
</Warning>

Kompatible Bundles sind standardmäßig sicherer, da OpenClaw sie derzeit als Metadaten-/Inhaltspakete behandelt. In aktuellen Versionen bedeutet dies hauptsächlich gebündelte Skills.

Verwenden Sie Positivlisten und explizite Installations-/Ladepfade für nicht gebündelte Plugins. Behandeln Sie Workspace-Plugins als Code für die Entwicklungsphase und nicht als Produktionsstandard.

Bei gebündelten Workspace-Paketnamen muss die Plugin-ID standardmäßig im npm-Namen verankert bleiben: `@openclaw/<id>`; alternativ kann ein genehmigtes typisiertes Suffix wie `-provider`, `-plugin`, `-speech`, `-sandbox` oder `-media-understanding` verwendet werden, wenn das Paket absichtlich eine enger gefasste Plugin-Rolle bereitstellt.

<Note>
**Vertrauenshinweis:** `plugins.allow` vertraut **Plugin-IDs**, nicht der Herkunft des Quellcodes. Ein Workspace-Plugin mit derselben ID wie ein gebündeltes Plugin überschattet absichtlich die gebündelte Kopie, wenn dieses Workspace-Plugin aktiviert oder in die Positivliste aufgenommen wurde. Dies ist normal und nützlich für die lokale Entwicklung, Patch-Tests und Hotfixes. Das Vertrauen in gebündelte Plugins wird anhand des Quellcode-Snapshots bestimmt – des Manifests und des Codes, die zum Ladezeitpunkt auf dem Datenträger vorhanden sind – und nicht anhand von Installationsmetadaten. Ein beschädigter oder ausgetauschter Installationsdatensatz kann die Vertrauensoberfläche eines gebündelten Plugins nicht unbemerkt über die Angaben der tatsächlichen Quelle hinaus erweitern.
</Note>

## Exportgrenze

OpenClaw exportiert Fähigkeiten, keine Implementierungsbequemlichkeiten.

Halten Sie die Fähigkeitsregistrierung öffentlich. Reduzieren Sie Exporte von Hilfen, die nicht zum Vertrag gehören:

- Hilfsunterpfade für bestimmte gebündelte Plugins
- Unterpfade der Laufzeit-Infrastruktur, die nicht als öffentliche API vorgesehen sind
- Provider-spezifische Komfortfunktionen
- Einrichtungs-/Onboarding-Hilfen, bei denen es sich um Implementierungsdetails handelt

Reservierte Hilfsunterpfade für gebündelte Plugins wurden aus der generierten SDK-Exportzuordnung entfernt. Belassen Sie zuständigkeitsspezifische Hilfen im jeweils zuständigen Plugin-Paket; stufen Sie nur wiederverwendbares Host-Verhalten zu generischen SDK-Verträgen wie `plugin-sdk/gateway-runtime`, `plugin-sdk/security-runtime` und eingespeisten Plugin-API-Fähigkeiten hoch.

## Interna und Referenz

Informationen zur Ladepipeline, zum Registry-Modell, zu Provider-Laufzeit-Hooks, Gateway-HTTP-Routen, Schemas für Nachrichtentools, zur Auflösung von Kanalzielen, zu Provider-Katalogen, Kontext-Engine-Plugins und zur Anleitung zum Hinzufügen einer neuen Fähigkeit finden Sie unter [Interna der Plugin-Architektur](/de/plugins/architecture-internals).

## Verwandte Themen

- [Plugins erstellen](/de/plugins/building-plugins)
- [Plugin-Manifest](/de/plugins/manifest)
- [Plugin-SDK einrichten](/de/plugins/sdk-setup)
