---
read_when:
    - Implementierung von Provider-Runtime-Hooks, Kanal-Lebenszyklus oder Paket-Bundles
    - Ladereihenfolge von Plugins oder Registry-Status debuggen
    - Hinzufügen einer neuen Plugin-Funktion oder eines Kontext-Engine-Plugins
summary: 'Interna der Plugin-Architektur: Ladepipeline, Registry, Runtime-Hooks, HTTP-Routen und Referenztabellen'
title: Interna der Plugin-Architektur
x-i18n:
    generated_at: "2026-07-26T19:05:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 278ac23a9454ab69407c59fa197e75756fa0dc5880fcae6c3eecc15bd4733a09
    source_path: plugins/architecture-internals.md
    workflow: 16
---

Für das öffentliche Funktionsmodell, die Plugin-Strukturen und die Verträge
zu Zuständigkeit und Ausführung siehe [Plugin-Architektur](/de/plugins/architecture). Diese Seite behandelt
die internen Mechanismen: Lade-Pipeline, Registry, Runtime-Hooks, Gateway-HTTP-
Routen, Importpfade und Schematabellen.

## Lade-Pipeline

Beim Start führt OpenClaw ungefähr Folgendes aus:

1. potenzielle Plugin-Wurzelverzeichnisse ermitteln
2. native oder kompatible Bundle-Manifeste und Paketmetadaten lesen
3. unsichere Kandidaten ablehnen
4. Plugin-Konfiguration normalisieren (`plugins.enabled`, `allow`, `deny`, `entries`,
   `slots`, `load.paths`)
5. Aktivierung für jeden Kandidaten festlegen
6. aktivierte native Module laden: Erstellte gebündelte Module verwenden einen nativen Loader;
   lokale TypeScript-Quellen von Drittanbietern verwenden als Notlösung den Jiti-Fallback
7. native `register(api)`-Hooks aufrufen und Registrierungen in der Plugin-Registry erfassen
8. die Registry für Befehle und Runtime-Oberflächen bereitstellen

Sicherheitsprüfungen werden **vor** der Runtime-Ausführung durchgeführt. Die Ermittlung blockiert einen Kandidaten,
wenn:

- sein aufgelöster Einstiegspunkt außerhalb des Plugin-Wurzelverzeichnisses liegt
- sein Pfad (oder sein Wurzelverzeichnis) für alle Benutzer beschreibbar ist
- bei nicht gebündelten Plugins der Pfadeigentümer nicht mit der aktuellen uid (oder root) übereinstimmt

Bei für alle Benutzer beschreibbaren gebündelten Verzeichnissen wird zunächst direkt vor Ort eine
`chmod`-Reparatur versucht (npm-/globale Installationen können Paketverzeichnisse mit
`0777` ausliefern), bevor die Prüfung erneut erfolgt; bei gebündeltem Ursprung
werden Eigentümerprüfungen vollständig übersprungen.

Blockierte Kandidaten enthalten in der ausgegebenen Diagnose weiterhin ihre Plugin-ID, sofern
diese bekannt ist (einschließlich IDs, die aus einem Manifest innerhalb eines
ansonsten abgelehnten Verzeichnisses aufgelöst wurden). Dadurch wird eine Konfiguration, die auf diese ID verweist,
einem blockierten Plugin mit einer Warnung zur Pfadsicherheit zugeordnet, statt einen nicht damit zusammenhängenden
Fehler „Unbekanntes Plugin“ zu erhalten.

### Manifest-zuerst-Verhalten

Das Manifest ist die maßgebliche Quelle der Steuerungsebene. OpenClaw verwendet es, um:

- das Plugin zu identifizieren
- deklarierte Kanäle/Skills/Konfigurationsschemata oder Bundle-Funktionen zu ermitteln
- `plugins.entries.<id>.config` zu validieren
- Beschriftungen/Platzhalter der Control UI zu ergänzen
- Installations-/Katalogmetadaten anzuzeigen
- leichtgewichtige Aktivierungs- und Einrichtungsdeskriptoren zu erhalten, ohne die Plugin-Runtime zu laden

Bei nativen Plugins bildet das Runtime-Modul den Teil der Datenebene. Es registriert
das tatsächliche Verhalten, beispielsweise Hooks, Werkzeuge, Befehle oder Provider-Abläufe.

Optionale Manifestblöcke `activation` und `setup` verbleiben auf der Steuerungsebene.
Sie sind reine Metadatendeskriptoren für die Aktivierungsplanung und Einrichtungsermittlung;
sie ersetzen weder die Runtime-Registrierung noch `register(...)` oder `setupEntry`.
Aktive Aktivierungs-Consumer verwenden Hinweise zu Befehlen, Kanälen und Providern aus dem Manifest, um
das Laden von Plugins vor einer umfassenderen Materialisierung der Registry einzugrenzen:

- Beim Laden der CLI wird auf Plugins eingegrenzt, denen der angeforderte primäre Befehl gehört
- Bei der Kanaleinrichtung/Plugin-Auflösung wird auf Plugins eingegrenzt, denen die angeforderte
  Kanal-ID gehört
- Bei der expliziten Provider-Einrichtung/Runtime-Auflösung wird auf Plugins eingegrenzt, denen die
  angeforderte Provider-ID gehört
- Die Startplanung des Gateways verwendet `activation.onStartup` für explizite Start-
  Importe; Plugins ohne Startmetadaten werden nur durch engere
  Aktivierungsauslöser geladen

Der Aktivierungsplaner stellt sowohl eine reine ID-API für bestehende Aufrufer als auch eine
Plan-API für Diagnosen bereit. Planeinträge geben an, warum ein Plugin ausgewählt wurde,
und unterscheiden dabei explizite `activation.*`-Hinweise vom Fallback über die Manifest-Zuständigkeit:

| Grund (aus `activation.*`-Hinweisen)   | Grund (aus der Manifest-Zuständigkeit)                                                             |
| ------------------------------------ | -------------------------------------------------------------------------------------------- |
| `activation-agent-harness-hint`      | —                                                                                            |
| `activation-capability-hint`         | —                                                                                            |
| `activation-channel-hint`            | `manifest-channel-owner` (`channels`)                                                        |
| `activation-command-hint`            | `manifest-command-alias` (`commandAliases`)                                                  |
| `activation-provider-hint`           | `manifest-provider-owner` (`providers`), `manifest-setup-provider-owner` (`setup.providers`) |
| `activation-route-hint`              | —                                                                                            |
| — (der Hook-Auslöser besitzt keine Hinweisvariante) | `manifest-hook-owner` (`hooks`), `manifest-tool-contract` (`contracts.tools`)                |

Diese Trennung der Gründe bildet die Kompatibilitätsgrenze: Bestehende Plugin-Metadaten
funktionieren weiterhin, während neuer Code umfassende Hinweise oder Fallback-Verhalten erkennen kann,
ohne die Semantik des Runtime-Ladens zu ändern.

Runtime-Vorabladevorgänge zur Anfragezeit, die den umfassenden Geltungsbereich `all` anfordern, leiten weiterhin
eine explizite effektive Menge von Plugin-IDs aus der Konfiguration, der Startplanung, den konfigurierten
Kanälen, Slots und Regeln zur automatischen Aktivierung ab
(`resolveEffectivePluginIds` in `src/plugins/effective-plugin-ids.ts`). Wenn diese
abgeleitete Menge leer ist, lässt OpenClaw den Geltungsbereich leer, statt ihn auf
jedes ermittelbare Plugin auszuweiten.

Die Einrichtungsermittlung bevorzugt Deskriptor-eigene IDs wie `setup.providers` und
`setup.cliBackends`, um potenzielle Plugins einzugrenzen, bevor auf
`setup-api` für Plugins zurückgegriffen wird, die weiterhin Runtime-Hooks zur Einrichtungszeit benötigen. Listen zur
Provider-Einrichtung verwenden Manifest-`providerAuthChoices`, aus Deskriptoren abgeleitete Einrichtungs-
optionen und Installationskatalog-Metadaten, ohne die Provider-Runtime zu laden. Ein explizites
`setup.requiresRuntime: false` stellt eine reine Deskriptor-Abschaltung dar; ein ausgelassenes
`requiresRuntime` behält aus Kompatibilitätsgründen den Legacy-Fallback über die Einrichtungs-API bei. Wenn
mehr als ein ermitteltes Plugin denselben normalisierten Einrichtungs-Provider oder dieselbe
CLI-Backend-ID beansprucht, lehnt die Einrichtungssuche den mehrdeutigen Zuständigen ab, statt sich auf die
Ermittlungsreihenfolge zu verlassen. Wenn die Einrichtungs-Runtime ausgeführt wird, melden Registry-Diagnosen
Abweichungen zwischen `setup.providers` / `setup.cliBackends` und den Providern oder CLI-
Backends, die tatsächlich durch die Einrichtungs-API registriert wurden, ohne Legacy-Plugins zu blockieren.

### Plugin-Cache-Grenze

OpenClaw speichert weder Ergebnisse der Plugin-Ermittlung noch direkte Daten der Manifest-Registry
hinter zeitbasierten Gültigkeitsfenstern zwischen. Installationen, Manifeständerungen und Änderungen
an Ladepfaden müssen beim nächsten expliziten Lesen der Metadaten oder beim nächsten Neuaufbau des Snapshots
sichtbar werden. Der Manifestdatei-Parser verwendet einen begrenzten Dateisignatur-Cache, dessen Schlüssel aus dem
Pfad des geöffneten Manifests sowie Gerät/Inode, Größe und mtime/ctime besteht; dieser Cache
verhindert lediglich das erneute Parsen unveränderter Bytes und darf keine Antworten zu Ermittlung, Registry,
Zuständigkeit oder Richtlinien zwischenspeichern.

Der sichere schnelle Metadatenpfad basiert auf explizitem Objektbesitz und nicht auf einem verborgenen Cache.
Leistungskritische Pfade beim Gateway-Start sollten den aktuellen `PluginMetadataSnapshot`, den
abgeleiteten `PluginLookUpTable` oder eine explizite Manifest-Registry durch die Aufrufkette
reichen. Konfigurationsvalidierung, automatische Aktivierung beim Start, Plugin-Bootstrap und Provider-
Auswahl können diese Objekte wiederverwenden, solange sie die aktuelle Konfiguration und
den aktuellen Plugin-Bestand darstellen. Die Einrichtungssuche rekonstruiert Manifest-Metadaten weiterhin bei Bedarf,
sofern der jeweilige Einrichtungspfad keine explizite Manifest-Registry erhält; dies sollte
ein Fallback für selten ausgeführte Pfade bleiben, statt verborgene Such-Caches hinzuzufügen. Wenn sich die
Eingabe ändert, erstellen und ersetzen Sie den Snapshot neu, statt ihn zu verändern oder
historische Kopien aufzubewahren. Ansichten der aktiven Plugin-Registry und Bootstrap-
Hilfsfunktionen für gebündelte Kanäle sollten aus der aktuellen Registry/dem aktuellen Wurzelverzeichnis
neu berechnet werden. Kurzlebige Maps innerhalb eines einzelnen Aufrufs sind zulässig, um Arbeit zu deduplizieren oder
Wiedereintritte zu verhindern; sie dürfen nicht zu Prozessmetadaten-Caches werden.

Beim Laden von Plugins ist das Runtime-Laden die persistente Cache-Schicht. Sie kann
Loader-Zustände wiederverwenden, wenn Code oder installierte Artefakte tatsächlich geladen werden, beispielsweise:

- `PluginLoaderCacheState` und kompatible aktive Runtime-Registries
- Jiti-/Modul-Caches und Loader-Caches für öffentliche Oberflächen, die verhindern, dass
  dieselbe Runtime-Oberfläche wiederholt importiert wird
- Dateisystem-Caches für installierte Plugin-Artefakte
- kurzlebige Maps pro Aufruf für die Pfadnormalisierung oder Auflösung von Duplikaten

Diese Caches sind Implementierungsdetails der Datenebene. Sie dürfen keine
Fragen der Steuerungsebene beantworten, etwa „Welchem Plugin gehört dieser Provider?“, sofern der
Aufrufer nicht ausdrücklich das Laden der Runtime angefordert hat.

Fügen Sie keine persistenten oder zeitbasierten Caches hinzu für:

- Ermittlungsergebnisse
- direkte Manifest-Registries
- aus dem Index installierter Plugins rekonstruierte Manifest-Registries
- Suche nach Provider-Zuständigen, Modellunterdrückung, Provider-Richtlinien oder Metadaten
  öffentlicher Artefakte
- andere aus Manifesten abgeleitete Antworten, bei denen ein geändertes Manifest, ein geänderter installierter Index
  oder ein geänderter Ladepfad beim nächsten Lesen der Metadaten sichtbar sein sollte

Aufrufer, die Manifest-Metadaten aus dem persistenten Index installierter Plugins
neu aufbauen, rekonstruieren diese Registry bei Bedarf. Der installierte Index ist ein dauerhafter
Zustand der Quellebene; er ist kein verborgener prozessinterner Metadaten-Cache.

## Registry-Modell

Geladene Plugins verändern nicht direkt beliebige globale Variablen des Kerns. Sie registrieren sich in einer
zentralen Plugin-Registry (`PluginRegistry` in `src/plugins/registry-types.ts`),
die Plugin-Datensätze (Identität, Quelle, Ursprung, Status, Diagnosen)
sowie Arrays für jede Funktion verwaltet: Werkzeuge, Legacy-Hooks und typisierte Hooks,
Kanäle, Provider, Gateway-RPC-Handler, HTTP-Routen, CLI-Registrierungsfunktionen,
Hintergrunddienste, Plugin-eigene Befehle und Dutzende weitere typisierte Provider-
Familien (Sprachausgabe, Einbettungen, Bild-/Video-/Musikgenerierung, Web-
Abruf/-Suche, Agent-Harnesses, Sitzungsaktionen und so weiter).

Kernfunktionen lesen anschließend aus dieser Registry, statt direkt mit Plugin-
Modulen zu kommunizieren. Dadurch bleibt der Ladevorgang unidirektional:

- Plugin-Modul -> Registry-Registrierung
- Kern-Runtime -> Registry-Nutzung

Diese Trennung ist für die Wartbarkeit wichtig. Dadurch benötigen die meisten Kernoberflächen nur
einen Integrationspunkt: „Registry lesen“, nicht „jedes
Plugin-Modul als Sonderfall behandeln“.

## Callbacks für Konversationsbindungen

Plugins, die eine Konversation binden, können reagieren, wenn eine Genehmigung abgeschlossen wurde.

Verwenden Sie `api.onConversationBindingResolved(...)`, um einen Callback zu erhalten, nachdem eine Bindungs-
anfrage genehmigt oder abgelehnt wurde:

```ts
export default {
  id: "my-plugin",
  register(api) {
    api.onConversationBindingResolved(async (event) => {
      if (event.status === "approved") {
        // Für dieses Plugin und diese Konversation besteht jetzt eine Bindung.
        console.log(event.binding?.conversationId);
        return;
      }

      // Die Anfrage wurde abgelehnt; lokalen ausstehenden Zustand löschen.
      console.log(event.request.conversation.conversationId);
    });
  },
};
```

Felder der Callback-Nutzlast:

- `status`: `"approved"` oder `"denied"`
- `decision`: `"allow-once"`, `"allow-always"` oder `"deny"`
- `binding`: die aufgelöste Bindung für genehmigte Anfragen
- `request`: die Zusammenfassung der ursprünglichen Anfrage, der Hinweis zum Trennen, die Absender-ID und
  die Konversationsmetadaten

Dieser Callback dient ausschließlich zur Benachrichtigung. Er ändert nicht, wer eine
Konversation binden darf, und wird ausgeführt, nachdem die Verarbeitung der Genehmigung im Kern abgeschlossen ist.

## Provider-Runtime-Hooks

Provider-Plugins verfügen über drei Ebenen:

- **Manifest-Metadaten** für eine leichtgewichtige Suche vor der Runtime:
  `setup.providers[].envVars`, `providerAuthAliases`, `providerAuthChoices`
  und `channelConfigs`.
- **Hooks zur Konfigurationszeit**: `catalog` plus `applyConfigDefaults`.
- **Runtime-Hooks**: mehr als 40 optionale Hooks für Authentifizierung, Modellauflösung,
  Stream-Wrapper, Denkstufen, Wiederholungsrichtlinien und Nutzungsendpunkte. Siehe
  [Hook-Reihenfolge und Nutzung](#hook-order-and-usage).

OpenClaw ist weiterhin für die generische Agentenschleife, das Failover, die Transkriptverarbeitung und
die Tool-Richtlinie zuständig. Diese Hooks bilden die Erweiterungsschnittstelle für providerspezifisches
Verhalten, ohne dass ein vollständig benutzerdefinierter Inferenztransport erforderlich ist.

Verwenden Sie Manifest `setup.providers[].envVars`, wenn der Provider umgebungsvariablenbasierte
Anmeldedaten besitzt, die generische Authentifizierungs-, Status- und Modellauswahlpfade erkennen sollen, ohne
die Plugin-Laufzeit zu laden. Verwenden Sie Manifest `providerAuthAliases`,
wenn eine Provider-ID die Umgebungsvariablen, Authentifizierungsprofile,
konfigurationsgestützte Authentifizierung und Auswahl für das API-Schlüssel-Onboarding einer anderen Provider-ID wiederverwenden soll. Verwenden Sie Manifest
`providerAuthChoices`, wenn CLI-Oberflächen für Onboarding und Authentifizierungsauswahl die
Auswahl-ID des Providers, Gruppenbeschriftungen und eine einfache Authentifizierungsanbindung über ein einzelnes Flag kennen sollen, ohne
die Provider-Laufzeit zu laden. Behalten Sie Provider-Laufzeit-
`envVars` für Hinweise für Betreiber bei, etwa Onboarding-Beschriftungen oder Variablen
zur Einrichtung von OAuth-Client-ID und -Client-Secret.

Beschreiben Sie die umgebungsvariablengesteuerte Kanaleinrichtung und Authentifizierung über die zugehörigen
`channelConfigs.<id>.schema`- und Einrichtungsdeskriptoren.

### Reihenfolge und Verwendung der Hooks

Bei Modell-/Provider-Plugins ruft OpenClaw die Hooks ungefähr in dieser Reihenfolge auf.
Die Spalte „Wann verwenden“ dient als schnelle Entscheidungshilfe.
Ausschließlich der Kompatibilität dienende Provider-Felder, die OpenClaw nicht mehr aufruft, wie
`ProviderPlugin.capabilities` und `suppressBuiltInModel`, sind hier bewusst nicht
aufgeführt.

| Hook                              | Funktion                                                                                                       | Verwendungszweck                                                                                                                              |
| --------------------------------- | -------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `catalog`                         | Provider-Konfiguration während der Generierung von `models.json` in `models.providers` veröffentlichen        | Der Provider verwaltet einen Katalog oder Standardwerte für die Basis-URL                                                                     |
| `applyConfigDefaults`             | Provider-eigene globale Konfigurationsstandardwerte bei der Konfigurationsmaterialisierung anwenden             | Standardwerte hängen vom Authentifizierungsmodus, von Umgebungsvariablen oder von der Semantik der Modellfamilie des Providers ab              |
| _(integrierte Modellsuche)_       | OpenClaw versucht zuerst den normalen Registry-/Katalogpfad                                                    | _(kein Plugin-Hook)_                                                                                                                          |
| `normalizeModelId`                | Aliasse für veraltete oder Vorschau-Modell-IDs vor der Suche normalisieren                                     | Der Provider verwaltet die Aliasbereinigung vor der kanonischen Modellauflösung                                                               |
| `normalizeTransport`              | Providerfamilien-spezifische `api` / `baseUrl` vor der generischen Modellzusammenstellung normalisieren | Der Provider verwaltet die Transportbereinigung für benutzerdefinierte Provider-IDs derselben Transportfamilie                                |
| `normalizeConfig`                 | `models.providers.<id>` vor der Laufzeit-/Providerauflösung normalisieren                                      | Der Provider benötigt eine Konfigurationsbereinigung, die beim Plugin angesiedelt sein sollte; gebündelte Hilfsfunktionen der Google-Familie sichern zudem unterstützte Google-Konfigurationseinträge ab |
| `applyNativeStreamingUsageCompat` | Native Kompatibilitätsanpassungen für die Streaming-Nutzung auf konfigurierte Provider anwenden                 | Der Provider benötigt endpunktgesteuerte Korrekturen an nativen Metadaten zur Streaming-Nutzung                                               |
| `resolveConfigApiKey`             | Authentifizierung über Umgebungsmarker für konfigurierte Provider vor dem Laden der Laufzeitauthentifizierung auflösen | Provider stellen eigene Hooks zur Auflösung von API-Schlüsseln über Umgebungsmarker bereit                                                     |
| `resolveSyntheticAuth`            | Lokale, selbst gehostete oder konfigurationsgestützte Authentifizierung ohne Speicherung im Klartext bereitstellen | Der Provider kann mit einem synthetischen/lokalen Anmeldedatenmarker betrieben werden                                                         |
| `resolveExternalAuthProfiles`     | Provider-eigene externe Authentifizierungsprofile überlagern; der Standardwert für `persistence` ist `runtime-only` für CLI-/App-eigene Anmeldedaten | Der Provider verwendet externe Authentifizierungsdaten erneut, ohne kopierte Aktualisierungstoken zu speichern; deklarieren Sie `contracts.externalAuthProviders` im Manifest |
| `shouldDeferSyntheticProfileAuth` | Gespeicherte synthetische Profilplatzhalter hinter umgebungs-/konfigurationsgestützter Authentifizierung nachrangig behandeln | Der Provider speichert synthetische Platzhalterprofile, die bei der Priorisierung nicht gewinnen sollten                                      |
| `resolveDynamicModel`             | Synchroner Fallback für Provider-eigene Modell-IDs, die noch nicht in der lokalen Registry enthalten sind       | Der Provider akzeptiert beliebige Modell-IDs des Upstream-Systems                                                                              |
| `prepareDynamicModel`             | Asynchrone Aufwärmphase, anschließend wird `resolveDynamicModel` erneut ausgeführt                            | Der Provider benötigt Netzwerkmetadaten, bevor unbekannte IDs aufgelöst werden können                                                         |
| `normalizeResolvedModel`          | Abschließende Anpassung, bevor der eingebettete Runner das aufgelöste Modell verwendet                          | Der Provider benötigt Transportanpassungen, verwendet jedoch weiterhin einen Kerntransport                                                    |
| `normalizeToolSchemas`            | Tool-Schemas normalisieren, bevor der eingebettete Runner sie verarbeitet                                      | Der Provider benötigt eine Schemasäuberung für die Transportfamilie                                                                           |
| `inspectToolSchemas`              | Provider-eigene Schemadiagnosen nach der Normalisierung bereitstellen                                          | Der Provider möchte Warnungen zu Schlüsselwörtern ausgeben, ohne den Kern um Provider-spezifische Regeln zu erweitern                          |
| `resolveReasoningOutputMode`      | Vertrag für native oder markierte Reasoning-Ausgaben auswählen                                                 | Der Provider benötigt markierte Reasoning-/Endausgaben anstelle nativer Felder                                                                |
| `prepareExtraParams`              | Anfrageparameter vor generischen Wrappern für Stream-Optionen normalisieren                                    | Der Provider benötigt Standard-Anfrageparameter oder eine parameterspezifische Bereinigung pro Provider                                       |
| `createStreamFn`                  | Den normalen Stream-Pfad vollständig durch einen benutzerdefinierten Transport ersetzen                        | Der Provider benötigt ein benutzerdefiniertes Übertragungsprotokoll, nicht nur einen Wrapper                                                   |
| `wrapStreamFn`                    | Stream-Wrapper, nachdem generische Wrapper angewendet wurden                                                   | Der Provider benötigt Kompatibilitäts-Wrapper für Anfrageheader, -text oder Modell, jedoch keinen benutzerdefinierten Transport                 |
| `resolveTransportTurnState`       | Native Transportheader oder Metadaten pro Durchlauf anhängen                                                   | Der Provider möchte, dass generische Transporte eine Provider-native Durchlaufidentität senden                                                 |
| `resolveWebSocketSessionPolicy`   | Native WebSocket-Header oder Richtlinie zur Abkühlzeit von Sitzungen anhängen                                   | Der Provider möchte bei generischen WS-Transporten Sitzungsheader oder die Fallback-Richtlinie anpassen                                        |
| `formatApiKey`                    | Formatierer für Authentifizierungsprofile: Das gespeicherte Profil wird zur Laufzeitzeichenfolge `apiKey` | Der Provider speichert zusätzliche Authentifizierungsmetadaten und benötigt ein benutzerdefiniertes Format des Laufzeittokens                  |
| `refreshOAuth`                    | OAuth-Aktualisierung für benutzerdefinierte Aktualisierungsendpunkte oder Richtlinien bei Aktualisierungsfehlern überschreiben | Der Provider ist nicht mit den gemeinsamen Aktualisierungsmechanismen von OpenClaw kompatibel                                                  |
| `buildAuthDoctorHint`             | Reparaturhinweis anhängen, wenn die OAuth-Aktualisierung fehlschlägt                                           | Der Provider benötigt nach einem Aktualisierungsfehler Provider-eigene Hinweise zur Reparatur der Authentifizierung                            |
| `matchesContextOverflowError`     | Provider-eigene Erkennung einer Überschreitung des Kontextfensters                                             | Der Provider liefert rohe Überlauffehler, die generische Heuristiken nicht erkennen würden                                                     |
| `classifyFailoverReason`          | Provider-eigene Klassifizierung von Failover-Gründen                                                           | Der Provider kann rohe API-/Transportfehler auf Ratenbegrenzung, Überlastung usw. abbilden                                                     |
| `isCacheTtlEligible`              | Prompt-Cache-Richtlinie für Proxy-/Backhaul-Provider                                                           | Der Provider benötigt Proxy-spezifische Einschränkungen für die Cache-TTL                                                                     |
| `buildMissingAuthMessage`         | Ersatz für die generische Wiederherstellungsmeldung bei fehlender Authentifizierung                            | Der Provider benötigt einen Provider-spezifischen Wiederherstellungshinweis bei fehlender Authentifizierung                                   |
| `augmentModelCatalog`             | Nach der Erkennung angehängte synthetische/abschließende Katalogzeilen (veraltet, siehe unten)                  | Der Provider benötigt synthetische Zeilen für Vorwärtskompatibilität in `models list` und Auswahlfeldern                                 |
| `resolveThinkingProfile`          | Modellspezifische `/think`-Stufen, Anzeigebezeichnungen und Standardwert                               | Der Provider stellt für ausgewählte Modelle eine benutzerdefinierte Denkstufenfolge oder binäre Bezeichnung bereit                             |
| `isBinaryThinking`                | Kompatibilitäts-Hook zum Ein-/Ausschalten von Reasoning                                                        | Der Provider stellt nur binäres Ein-/Ausschalten des Denkens bereit                                                                           |
| `supportsXHighThinking`           | Kompatibilitäts-Hook für `xhigh`-Reasoning-Unterstützung                                                       | Der Provider möchte `xhigh` nur für eine Teilmenge der Modelle aktivieren                                                                     |
| `resolveDefaultThinkingLevel`     | Kompatibilitäts-Hook für die Standardstufe `/think`                                                            | Der Provider verwaltet die Standardrichtlinie für `/think` einer Modellfamilie                                                                |
| `isModernModelRef`                | Matcher für moderne Modelle für Live-Profilfilter und Smoke-Auswahl                                            | Der Provider verwaltet den Abgleich bevorzugter Modelle für Live-/Smoke-Tests                                                                 |
| `prepareRuntimeAuth`              | Konfigurierte Anmeldedaten unmittelbar vor der Inferenz gegen das tatsächliche Laufzeittoken bzw. den tatsächlichen Laufzeitschlüssel austauschen | Der Provider benötigt einen Tokenaustausch oder kurzlebige Anmeldedaten für Anfragen                                                           |
| `resolveUsageAuth`                | Anmeldedaten für Nutzung/Abrechnung für `/usage` und verwandte Statusoberflächen auflösen                   | Der Provider benötigt eine benutzerdefinierte Analyse von Nutzungs-/Kontingenttokens oder andere Anmeldedaten für die Nutzung                  |
| `fetchUsageSnapshot`              | Provider-spezifische Nutzungs-/Kontingentmomentaufnahmen nach Auflösung der Authentifizierung abrufen und normalisieren | Der Provider benötigt einen Provider-spezifischen Nutzungsendpunkt oder Parser für Nutzlasten                                                  |
| `createEmbeddingProvider`         | Einen Provider-eigenen Embedding-Adapter für Speicher/Suche erstellen                                                     | Das Verhalten von Speicher-Embeddings gehört in das Provider-Plugin                                                                                    |
| `buildReplayPolicy`               | Eine Replay-Richtlinie zurückgeben, die die Transkriptverarbeitung für den Provider steuert                                        | Der Provider benötigt eine benutzerdefinierte Transkriptrichtlinie (zum Beispiel das Entfernen von Denkblöcken)                                                               |
| `sanitizeReplayHistory`           | Den Replay-Verlauf nach der generischen Transkriptbereinigung umschreiben                                                        | Der Provider benötigt Provider-spezifische Replay-Umschreibungen über die gemeinsam genutzten Compaction-Hilfsfunktionen hinaus                                                             |
| `validateReplayTurns`             | Abschließende Validierung oder Umformung des Replay-Turns vor dem eingebetteten Runner                                           | Der Provider-Transport benötigt nach der generischen Bereinigung eine strengere Turn-Validierung                                                                    |
| `onModelSelected`                 | Provider-eigene Nebeneffekte nach der Auswahl ausführen                                                                 | Der Provider benötigt Telemetrie oder Provider-eigenen Zustand, wenn ein Modell aktiv wird                                                                  |

`normalizeModelId`, `normalizeTransport` und `normalizeConfig` prüfen zuerst das
übereinstimmende Provider-Plugin und durchlaufen dann weitere Hook-fähige Provider-Plugins,
bis eines tatsächlich die Modell-ID oder den Transport/die Konfiguration ändert. Dadurch funktionieren
Alias-/Kompatibilitäts-Provider-Shims weiterhin, ohne dass der Aufrufer wissen muss, welches
gebündelte Plugin für die Umschreibung zuständig ist. Wenn kein Provider-Hook einen unterstützten
Konfigurationseintrag der Google-Familie umschreibt, führt der gebündelte Google-Konfigurationsnormalisierer
weiterhin diese Kompatibilitätsbereinigung durch.

Wenn der Provider ein vollständig benutzerdefiniertes Wire-Protokoll oder einen benutzerdefinierten Request-Executor benötigt,
handelt es sich um eine andere Erweiterungsklasse. Diese Hooks sind für Provider-Verhalten vorgesehen,
das weiterhin in der normalen Inferenzschleife von OpenClaw ausgeführt wird.

`resolveUsageAuth` entscheidet, ob OpenClaw `fetchUsageSnapshot` aufrufen oder
für Nutzungs-/Statusoberflächen auf die generische Auflösung von Zugangsdaten
zurückgreifen soll. Geben Sie `{ token, accountId?, subscriptionType?, rateLimitTier? }` zurück, wenn der Provider
über Nutzungszugangsdaten verfügt (die optionalen Tarifmetadaten fließen in
`fetchUsageSnapshot` ein), geben Sie
`{ handled: true }` zurück, wenn die Provider-eigene Nutzungsauthentifizierung die Anfrage verarbeitet hat und
den generischen API-Schlüssel-/OAuth-Fallback unterdrücken muss, und geben Sie `null` oder `undefined`
zurück, wenn der Provider die Nutzungsauthentifizierung nicht verarbeitet hat.

Deklarieren Sie Organisations- oder Abrechnungszugangsdaten im Manifest
`providerUsageAuthEnvVars`. Dadurch können generische Erkennungs- und Secret-Bereinigungsoberflächen
sie erkennen, ohne sie zu Kandidaten für die Inferenzauthentifizierung zu machen.

### Provider-Beispiel

```ts
api.registerProvider({
  id: "example-proxy",
  label: "Beispiel-Proxy",
  auth: [],
  catalog: {
    order: "simple",
    run: async (ctx) => {
      const apiKey = ctx.resolveProviderApiKey("example-proxy").apiKey;
      if (!apiKey) {
        return null;
      }
      return {
        provider: {
          baseUrl: "https://proxy.example.com/v1",
          apiKey,
          api: "openai-completions",
          models: [{ id: "auto", name: "Automatisch" }],
        },
      };
    },
  },
  resolveDynamicModel: (ctx) => ({
    id: ctx.modelId,
    name: ctx.modelId,
    provider: "example-proxy",
    api: "openai-completions",
    baseUrl: "https://proxy.example.com/v1",
    reasoning: false,
    input: ["text"],
    cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
    contextWindow: 128000,
    maxTokens: 8192,
  }),
  prepareRuntimeAuth: async (ctx) => {
    const exchanged = await exchangeToken(ctx.apiKey);
    return {
      apiKey: exchanged.token,
      baseUrl: exchanged.baseUrl,
      expiresAt: exchanged.expiresAt,
    };
  },
  resolveUsageAuth: async (ctx) => {
    const auth = await ctx.resolveOAuthToken();
    return auth ? { token: auth.token } : null;
  },
  fetchUsageSnapshot: async (ctx) => {
    return await fetchExampleProxyUsage(ctx.token, ctx.timeoutMs, ctx.fetchFn);
  },
});
```

### Integrierte Beispiele

Gebündelte Provider-Plugins kombinieren die oben genannten Hooks entsprechend den Katalog-,
Authentifizierungs-, Denk-, Wiederholungs- und Nutzungsanforderungen der jeweiligen Anbieter. Der maßgebliche Hook-Satz befindet sich
bei jedem Plugin unter `extensions/`; diese Seite veranschaulicht die Strukturen, statt
die Liste zu spiegeln.

<AccordionGroup>
  <Accordion title="Provider mit durchgereichtem Katalog">
    OpenRouter, Kilocode, Z.AI und xAI registrieren `catalog` sowie
    `resolveDynamicModel` / `prepareDynamicModel`, damit sie vorgelagerte
    Modell-IDs vor dem statischen Katalog von OpenClaw bereitstellen können.
  </Accordion>
  <Accordion title="Provider mit OAuth- und Nutzungsendpunkten">
    GitHub Copilot, Gemini CLI, ChatGPT Codex, MiniMax, Xiaomi und z.ai kombinieren
    `prepareRuntimeAuth` oder `formatApiKey` mit `resolveUsageAuth` +
    `fetchUsageSnapshot`, um Token-Austausch und die Integration von `/usage`
    zu übernehmen.
  </Accordion>
  <Accordion title="Familien für Wiederholungs- und Transkriptbereinigung">
    Gemeinsame benannte Familien (`google-gemini`, `passthrough-gemini`,
    `anthropic-by-model`, `hybrid-anthropic-openai`) ermöglichen Providern, sich über
    `buildReplayPolicy` für Transkriptrichtlinien zu entscheiden, statt die Bereinigung
    in jedem Plugin erneut zu implementieren.
  </Accordion>
  <Accordion title="Provider ausschließlich mit Katalog">
    `byteplus`, `cloudflare-ai-gateway`, `huggingface`, `kimi-coding`, `nvidia`,
    `qianfan`, `synthetic`, `together`, `venice`, `vercel-ai-gateway` und
    `volcengine` registrieren nur `catalog` und verwenden die gemeinsame Inferenzschleife.
  </Accordion>
  <Accordion title="Anthropic-spezifische Stream-Hilfsfunktionen">
    Beta-Header, `/fast` / `serviceTier` und `context1m` befinden sich innerhalb der
    öffentlichen `api.ts`- / `contract-api.ts`-Schnittstelle des Anthropic-Plugins
    (`wrapAnthropicProviderStream`, `resolveAnthropicBetas`,
    `resolveAnthropicFastMode`, `resolveAnthropicServiceTier`) und nicht im
    generischen SDK.
  </Accordion>
</AccordionGroup>

## Runtime-Hilfsfunktionen

Plugins können über `api.runtime` auf ausgewählte Kern-Hilfsfunktionen zugreifen. Für TTS:

```ts
const clip = await api.runtime.tts.textToSpeech({
  text: "Hallo von OpenClaw",
  cfg: api.config,
});

const result = await api.runtime.tts.textToSpeechTelephony({
  text: "Hallo von OpenClaw",
  cfg: api.config,
});

const voices = await api.runtime.tts.listVoices({
  provider: "elevenlabs",
  cfg: api.config,
});
```

Hinweise:

- `textToSpeech` gibt die normale TTS-Ausgabe-Payload des Kerns für Datei-/Sprachnachrichtenoberflächen zurück.
- Verwendet die Kernkonfiguration `tts` und die Provider-Auswahl.
- Gibt einen PCM-Audiopuffer und die Abtastrate zurück. Plugins müssen für Provider neu abtasten/kodieren.
- `listVoices` ist pro Provider optional. Verwenden Sie es für anbietereigene Sprachauswahl- oder Einrichtungsabläufe.
- Der Kern übergibt eine aufgelöste Anfragefrist an Provider-Hooks vom Typ `listVoices`; Provider-spezifische Zeitüberschreitungseinstellungen können sie überschreiben.
- Sprachlisten können umfangreichere Metadaten wie Gebietsschema, Geschlecht und Persönlichkeits-Tags für Provider-bewusste Auswahlfelder enthalten.
- OpenAI und ElevenLabs unterstützen derzeit Telefonie. Microsoft nicht.

Plugins können über `api.registerSpeechProvider(...)` auch Sprachanbieter registrieren.

```ts
api.registerSpeechProvider({
  id: "acme-speech",
  label: "Acme-Sprache",
  isConfigured: ({ config }) => Boolean(config.messages?.tts),
  synthesize: async (req) => {
    return {
      audioBuffer: Buffer.from([]),
      outputFormat: "mp3",
      fileExtension: ".mp3",
      voiceCompatible: false,
    };
  },
});
```

Hinweise:

- Belassen Sie TTS-Richtlinien, Fallback und Antwortzustellung im Kern.
- Verwenden Sie Sprachanbieter für anbietereigenes Syntheseverhalten.
- Die ältere Microsoft-Eingabe `edge` wird auf die Provider-ID `microsoft` normalisiert.
- Das bevorzugte Zuständigkeitsmodell ist unternehmensorientiert: Ein Anbieter-Plugin kann
  Text-, Sprach-, Bild- und zukünftige Medien-Provider verwalten, wenn OpenClaw diese
  Fähigkeitsverträge hinzufügt.

Für das Verstehen von Bildern, Audio und Videos registrieren Plugins einen typisierten
Provider für Medienverständnis statt einer generischen Schlüssel/Wert-Sammlung:

```ts
api.registerMediaUnderstandingProvider({
  id: "google",
  capabilities: ["image", "audio", "video"],
  describeImage: async (req) => ({ text: "..." }),
  transcribeAudio: async (req) => ({ text: "..." }),
  describeVideo: async (req) => ({ text: "..." }),
});
```

Hinweise:

- Belassen Sie Orchestrierung, Fallback, Konfiguration und Kanalverdrahtung im Kern.
- Belassen Sie anbieterspezifisches Verhalten im Provider-Plugin.
- Additive Erweiterungen sollten typisiert bleiben: neue optionale Methoden, neue optionale
  Ergebnisfelder, neue optionale Fähigkeiten.
- Die Videoerzeugung folgt bereits demselben Muster:
  - Der Kern verwaltet den Fähigkeitsvertrag und die Runtime-Hilfsfunktion.
  - Anbieter-Plugins registrieren `api.registerVideoGenerationProvider(...)`.
  - Funktions-/Kanal-Plugins verwenden `api.runtime.videoGeneration.*`.

Für Runtime-Hilfsfunktionen zum Medienverständnis können Plugins Folgendes aufrufen:

```ts
const image = await api.runtime.mediaUnderstanding.describeImageFile({
  filePath: "/tmp/inbound-photo.jpg",
  cfg: api.config,
  agentDir: "/tmp/agent",
});

const video = await api.runtime.mediaUnderstanding.describeVideoFile({
  filePath: "/tmp/inbound-video.mp4",
  cfg: api.config,
});

const extraction = await api.runtime.mediaUnderstanding.extractStructuredWithModel({
  provider: "codex",
  model: "gpt-5.6-sol",
  input: [
    {
      type: "image",
      buffer: receiptImageBuffer,
      fileName: "receipt.png",
      mime: "image/png",
    },
    { type: "text", text: "Verwenden Sie die gedruckten Felder als maßgebliche Quelle." },
  ],
  instructions: "Geben Sie Entitäten und durchsuchbare Tags zurück.",
  schemaName: "example.evidence",
  jsonSchema: {
    type: "object",
    properties: {
      entities: { type: "array", items: { type: "string" } },
      tags: { type: "array", items: { type: "string" } },
    },
  },
  cfg: api.config,
});
```

Für die Audiotranskription können Plugins entweder die Runtime für Medienverständnis
oder den älteren STT-Alias verwenden:

```ts
const { text } = await api.runtime.mediaUnderstanding.transcribeAudioFile({
  filePath: "/tmp/inbound-audio.ogg",
  cfg: api.config,
  // Optional, wenn der MIME-Typ nicht zuverlässig abgeleitet werden kann:
  mime: "audio/ogg",
});
```

Hinweise:

- `api.runtime.mediaUnderstanding.*` ist die bevorzugte gemeinsame Oberfläche für
  das Verstehen von Bildern, Audio und Videos.
- `extractStructuredWithModel(...)` ist die Plugin-seitige Schnittstelle für begrenzte,
  Provider-eigene, bildorientierte Extraktion. Fügen Sie mindestens eine Bildeingabe ein;
  Texteingaben sind ergänzender Kontext. Produkt-Plugins verwalten ihre Routen und
  Schemas, während OpenClaw die Provider-/Runtime-Grenze verwaltet.
- Verwendet die Audio-Kernkonfiguration für Medienverständnis (`tools.media.audio`) und die Provider-Fallback-Reihenfolge.
- Gibt `{ text: undefined }` zurück, wenn keine Transkriptionsausgabe erzeugt wird (beispielsweise bei übersprungener/nicht unterstützter Eingabe).

Plugins können über `api.runtime.subagent` auch Subagent-Hintergrundläufe starten:

```ts
const result = await api.runtime.subagent.run({
  sessionKey: "agent:main:subagent:search-helper",
  message: "Erweitern Sie diese Abfrage zu gezielten Folgesuchen.",
  toolsAlsoAllow: ["my_plugin_progress"],
  provider: "openai",
  model: "gpt-4.1-mini",
  deliver: false,
});
```

Hinweise:

- `provider` und `model` sind optionale Überschreibungen pro Lauf und keine dauerhaften Sitzungsänderungen.
- `toolsAlsoAllow` akzeptiert exakte, eindeutig zugeordnete Werkzeugnamen, die vom aufrufenden Plugin registriert wurden. Kernnamen und mehrdeutige Namen werden abgelehnt. Es ergänzt das normale Profil, doch Betreiber-Zulassungs- und Sperrlisten bleiben maßgeblich.
- OpenClaw berücksichtigt diese Überschreibungsfelder nur für vertrauenswürdige Aufrufer.
- Für Plugin-eigene Fallback-Läufe müssen Betreiber sich mit `plugins.entries.<id>.subagent.allowModelOverride: true` ausdrücklich dafür entscheiden.
- Verwenden Sie `plugins.entries.<id>.subagent.allowedModels`, um vertrauenswürdige Plugins auf bestimmte kanonische `provider/model`-Ziele zu beschränken, oder `"*"`, um ausdrücklich jedes Ziel zuzulassen.
- Subagent-Läufe nicht vertrauenswürdiger Plugins funktionieren weiterhin, Überschreibungsanforderungen werden jedoch abgelehnt, statt stillschweigend auf einen Fallback zurückzugreifen.
- Von Plugins erstellte Subagent-Sitzungen werden mit der ID des erstellenden Plugins gekennzeichnet. Fallback `api.runtime.subagent.deleteSession(...)` darf nur diese zugehörigen Sitzungen löschen; das Löschen beliebiger Sitzungen erfordert weiterhin eine Gateway-Anfrage mit Administratorbereich.

Für die Websuche können Plugins die gemeinsame Runtime-Hilfsfunktion verwenden, statt
auf die Verdrahtung der Agentenwerkzeuge zuzugreifen:

```ts
const providers = api.runtime.webSearch.listProviders({
  config: api.config,
});

const result = await api.runtime.webSearch.search({
  config: api.config,
  args: {
    query: "Runtime-Hilfsfunktionen für OpenClaw-Plugins",
    count: 5,
  },
});
```

Plugins können über
`api.registerWebSearchProvider(...)` auch Websuch-Provider registrieren.

Hinweise:

- Belassen Sie Provider-Auswahl, Auflösung von Zugangsdaten und gemeinsame Anfragesemantik im Kern.
- Verwenden Sie Websuch-Provider für anbieterspezifische Suchtransporte.
- `api.runtime.webSearch.*` ist die bevorzugte gemeinsame Oberfläche für Funktions-/Kanal-Plugins, die Suchverhalten benötigen, ohne vom Wrapper des Agentenwerkzeugs abhängig zu sein.

### `api.runtime.imageGeneration`

```ts
const result = await api.runtime.imageGeneration.generate({
  config: api.config,
  args: { prompt: "Ein freundliches Hummer-Maskottchen", size: "1024x1024" },
});

const providers = api.runtime.imageGeneration.listProviders({
  config: api.config,
});
```

- `generate(...)`: Generiert ein Bild mithilfe der konfigurierten Provider-Kette für die Bilderzeugung.
- `listProviders(...)`: Listet verfügbare Provider für die Bilderzeugung und deren Fähigkeiten auf.

## Gateway-HTTP-Routen

Plugins können mit `api.registerHttpRoute(...)` HTTP-Endpunkte bereitstellen.

```ts
api.registerHttpRoute({
  path: "/acme/webhook",
  auth: "plugin",
  match: "exact",
  handler: async (_req, res) => {
    res.statusCode = 200;
    res.end("ok");
    return true;
  },
});
```

Routenfelder:

- `path`: Routenpfad unter dem Gateway-HTTP-Server.
- `auth`: Erforderlich, `"gateway"` oder `"plugin"`. Verwenden Sie `"gateway"`, um die normale Gateway-Authentifizierung zu verlangen, oder `"plugin"` für eine vom Plugin verwaltete Authentifizierung/Webhook-Verifizierung.
- `match`: Optional. `"exact"` (Standard) oder `"prefix"`.
- `handleUpgrade`: Optionaler Handler für WebSocket-Upgrade-Anfragen auf derselben Route.
- `replaceExisting`: Optional. Ermöglicht demselben Plugin, seine eigene vorhandene Routenregistrierung zu ersetzen.
- `handler`: Gibt `true` zurück, wenn die Route die Anfrage verarbeitet hat.

Hinweise:

- `api.registerHttpHandler(...)` wurde entfernt und verursacht einen Fehler beim Laden des Plugins. Verwenden Sie stattdessen `api.registerHttpRoute(...)`.
- Plugin-Routen müssen `auth` ausdrücklich deklarieren.
- Exakte Konflikte bei `path + match` werden abgelehnt, außer bei `replaceExisting: true`; außerdem kann ein Plugin die Route eines anderen Plugins nicht ersetzen.
- Überlappende Routen mit unterschiedlichen `auth`-Stufen werden abgelehnt. Behalten Sie `exact`-/`prefix`-Durchreichungsketten ausschließlich auf derselben Authentifizierungsstufe.
- `auth: "plugin"`-Routen erhalten **nicht** automatisch Laufzeitbereiche für Operatoren. Sie sind für vom Plugin verwaltete Webhooks bzw. Signaturverifizierung vorgesehen, nicht für privilegierte Aufrufe von Gateway-Hilfsfunktionen.
- `auth: "gateway"`-Routen werden innerhalb eines Gateway-Anfragelaufzeitbereichs ausgeführt. Die Standardoberfläche (`gatewayRuntimeScopeSurface: "write-default"`) ist absichtlich restriktiv:
  - Die Bearer-Authentifizierung mit gemeinsamem Geheimnis (`gateway.auth.mode = "token"` / `"password"`) sowie jede Authentifizierungsmethode ohne vertrauenswürdigen Proxy erhalten einen einzelnen `operator.write`-Bereich, selbst wenn der Aufrufer `x-openclaw-scopes` sendet.
  - `trusted-proxy`-Aufrufer ohne expliziten `x-openclaw-scopes`-Header behalten ebenfalls die bisherige, ausschließlich auf `operator.write` beschränkte Oberfläche.
  - `trusted-proxy`-Aufrufer, die `x-openclaw-scopes` senden, erhalten stattdessen die deklarierten Bereiche.
  - Eine Route kann `gatewayRuntimeScopeSurface: "trusted-operator"` aktivieren, um `x-openclaw-scopes` bei identitätstragenden Authentifizierungsmodi stets zu berücksichtigen (fehlt der Header, wird auf den vollständigen Standardsatz der CLI-Bereiche zurückgegriffen).
- Sandbox-isolierte externe Control-UI-Registerkarten, die auf `auth: "gateway"`-Routen basieren, verwenden eine kurzlebige, signierte Cookie-Berechtigung, die ausschließlich durch einen authentifizierten Bootstrap ausgestellt wird; Registerkarten mit Plugin-Authentifizierung behalten ihren direkten iframe-Pfad. Vor dem Einbinden führt das übergeordnete Element innerhalb derselben opaken Sandbox eine routeneigene Prüfung aus und verweigert den Zugriff, wenn die Datenschutzrichtlinie des Browsers das Cookie blockiert. Die Berechtigung ist an das zuständige Plugin, den Stamm der übereinstimmenden Route und die aktuelle Authentifizierungsgeneration gebunden. Ihr prozesszufälliger Cookie-Name verhindert, dass vertrauenswürdige Gateways auf demselben Host einander überschreiben; Cookies isolieren jedoch niemals TCP-Ports. Der Gateway-Hostname bildet daher eine einzelne Grenze für Anmeldedaten: Stellen Sie auf diesem Hostnamen keine gegenseitig nicht vertrauenswürdigen Dienste bereit, auch nicht auf anderen Ports. Die Routenzustellung lehnt eine Wiederverwendung für eine verschachtelte Route ab, die einem anderen Plugin gehört. Da Sandbox-Nachfahren für Cookie-Zwecke websiteübergreifend sind, akzeptiert die Berechtigung ausschließlich `GET` und `HEAD` mit `operator.read`; Änderungen und WebSocket-Upgrades verbleiben auf explizit Gateway-authentifizierten Oberflächen. Das Cookie kann absichtlich kein CHIPS verwenden: Aktuelle Browser beziehen ein Bit für websiteübergreifende Vorfahren in den Partitionierungsschlüssel ein, sodass verschachtelte opake Sandbox-Frames den Zugriff auf Ressourcen derselben Route verlieren würden. Das Cookie erfordert einen sicheren Kontext und die Browserberechtigung für websiteübergreifende Cookies. Daher sind Gateway-authentifizierte externe Registerkarten auf reinen HTTP-LAN-Ursprüngen oder bei vollständiger Blockierung von Drittanbieter-Cookies nicht verfügbar; verwenden Sie HTTPS/Tailscale Serve oder einen vom Browser als vertrauenswürdig eingestuften Loopback mit einer kompatiblen Cookie-Richtlinie.
- Die Berechtigung verhindert die Offenlegung des Gateway-Bearer-Tokens und eine versehentliche Wiederverwendung von Routen oder Bereichen; sie schafft keine Sicherheitsgrenze zwischen nativen Plugins. Nativer Plugin-Code und die von ihm bereitgestellten UI-Inhalte bleiben Teil derselben vertrauenswürdigen, prozessinternen Plugin-Grenze.
- Praktische Regel: Gehen Sie nicht davon aus, dass eine Gateway-authentifizierte Plugin-Route implizit eine Administratoroberfläche ist. Wenn Ihre Route ausschließlich Administratoren vorbehaltenes Verhalten benötigt, aktivieren Sie die `trusted-operator`-Bereichsoberfläche, verlangen Sie einen identitätstragenden Authentifizierungsmodus und dokumentieren Sie den expliziten Vertrag für den `x-openclaw-scopes`-Header.
- Nach Routenabgleich und Authentifizierung nehmen gewöhnliche Handler an der Zulassung von Gateway-Stammaufgaben teil. Ein vorbereitetes oder neu startendes Gateway gibt `503` zurück, bevor es den Handler aufruft. Die enge Ausnahme bildet eine durch das Manifest berechtigte `auth: "gateway"`-Route, die zusätzlich die routenspezifische `trusted-operator`-Oberfläche aktiviert. Sie bleibt erreichbar, damit die Zustellung der Aussetzungssteuerung nicht blockiert wird, während gewöhnliche gleichgeordnete Routen desselben Plugins hinter der Zulassungsgrenze verbleiben. Der Besitz eines WebSocket-`handleUpgrade` verwendet dieselbe atomare Zulassungsgrenze. Sobald der Handler einen Socket akzeptiert, liegt dessen weitere Lebensdauer in der Verantwortung des Plugins und wird von dieser Grenze nicht nachverfolgt.

## Importpfade des Plugin-SDK

Verwenden Sie beim Erstellen neuer Plugins schmale SDK-Unterpfade statt des monolithischen
`openclaw/plugin-sdk`-Stamm-Barrels. Kernunterpfade:

| Unterpfad                          | Zweck                                        |
| ---------------------------------- | -------------------------------------------- |
| `openclaw/plugin-sdk/plugin-entry` | Grundelemente für die Plugin-Registrierung   |
| `openclaw/plugin-sdk/channel-core` | Hilfsfunktionen für Kanaleinstieg und -Build |
| `openclaw/plugin-sdk/core`         | Generische gemeinsame Hilfsfunktionen und übergreifender Vertrag |

Kanal-Plugins wählen aus einer Familie schmaler Schnittstellen — `channel-setup`,
`setup-runtime`, `setup-tools`, `channel-pairing`,
`channel-contract`, `channel-feedback`, `channel-inbound`, `channel-outbound`,
`command-auth`, `secret-input`, `webhook-ingress`,
`channel-targets` und `channel-actions`. Das Genehmigungsverhalten sollte in
einem einzigen `approvalCapability`-Vertrag zusammengeführt werden, statt es über
nicht zusammenhängende Plugin-Felder zu verteilen. Siehe [Kanal-Plugins](/de/plugins/sdk-channel-plugins).

Laufzeit- und Konfigurationshilfen befinden sich unter entsprechenden fokussierten `*-runtime`-Unterpfaden
(`approval-runtime`, `agent-runtime`, `lazy-runtime`, `directory-runtime`,
`text-runtime`, `runtime-store`, `system-event-runtime`, `heartbeat-runtime`,
`channel-activity-runtime` usw.). Bevorzugen Sie `config-contracts`,
`plugin-config-runtime`, `runtime-config-snapshot` und `config-mutation`
gegenüber dem breiten Kompatibilitäts-Barrel `config-runtime`.

<Info>
`openclaw/plugin-sdk/channel-lifecycle`, kleine Fassaden für Kanalhilfen,
`openclaw/plugin-sdk/config-runtime` und `openclaw/plugin-sdk/infra-runtime`
sind veraltete Kompatibilitäts-Shims für ältere Plugins. Neuer Code sollte stattdessen
schmalere generische Grundelemente importieren.
</Info>

Repo-interne Einstiegspunkte (je Stamm des gebündelten Plugin-Pakets):

- `index.js` — Einstiegspunkt des gebündelten Plugins
- `api.js` — Barrel für Hilfsfunktionen und Typen
- `runtime-api.js` — ausschließlich für die Laufzeit vorgesehenes Barrel
- `setup-entry.js` — Einstiegspunkt des Einrichtungs-Plugins

Externe Plugins sollten ausschließlich `openclaw/plugin-sdk/*`-Unterpfade importieren. Importieren Sie niemals
`src/*` eines anderen Plugin-Pakets aus dem Kern oder einem anderen Plugin.
Über Fassaden geladene Einstiegspunkte bevorzugen den aktiven Schnappschuss der Laufzeitkonfiguration,
sofern vorhanden, und greifen andernfalls auf die aufgelöste Konfigurationsdatei auf dem Datenträger zurück.

Fähigkeitsspezifische Unterpfade wie `image-generation`, `media-understanding`
und `speech` existieren, weil gebündelte Plugins sie derzeit verwenden. Sie sind nicht
automatisch langfristig unveränderliche externe Verträge — prüfen Sie die entsprechende
SDK-Referenzseite, wenn Sie sich auf sie verlassen.

## Schemas für Nachrichtenwerkzeuge

Plugins sollten kanalspezifische Beiträge zum `describeMessageTool(...)`-Schema
für Grundelemente außerhalb von Nachrichten wie Reaktionen, Lesebestätigungen und Umfragen besitzen.
Die gemeinsame Sendedarstellung sollte den generischen `MessagePresentation`-Vertrag
anstelle Provider-nativer Felder für Schaltflächen, Komponenten, Blöcke oder Karten verwenden.
Informationen zum Vertrag, zu Rückfallregeln, zur Provider-Zuordnung und zur Checkliste für Plugin-Autoren
finden Sie unter [Nachrichtendarstellung](/de/plugins/message-presentation).

Sendefähige Plugins deklarieren über Nachrichtenfähigkeiten, was sie darstellen können:

- `presentation` für semantische Darstellungsblöcke (`text`, `context`,
  `divider`, `chart`, `table`, `buttons`, `select`)
- `delivery-pin` für angeheftete Zustellungsanfragen

Der Kern entscheidet, ob die Darstellung nativ gerendert oder auf Text reduziert wird.
Stellen Sie über das generische Nachrichtenwerkzeug keine Provider-nativen UI-Ausweichmöglichkeiten bereit.
Veraltete SDK-Hilfsfunktionen für ältere native Schemas bleiben für bestehende
Drittanbieter-Plugins exportiert, neue Plugins sollten sie jedoch nicht verwenden.

## Auflösung von Kanalzielen

Kanal-Plugins sollten die kanalspezifische Zielsemantik besitzen. Halten Sie den gemeinsamen
ausgehenden Host generisch und verwenden Sie die Messaging-Adapter-Oberfläche für Provider-Regeln:

- `messaging.inferTargetChatType({ to })` entscheidet vor der Verzeichnissuche, ob ein normalisiertes Ziel
  als `direct`, `group` oder `channel` behandelt werden soll.
- `messaging.targetResolver.looksLikeId(raw, normalized)` teilt dem Kern mit, ob eine
  Eingabe direkt zur ID-ähnlichen Auflösung wechseln und die Verzeichnissuche überspringen soll.
- `messaging.targetResolver.reservedLiterals` listet einzelne Wörter auf, die
  Kanal-/Sitzungsreferenzen für diesen Provider sind. Bei der Auflösung bleiben konfigurierte
  Verzeichniseinträge erhalten, bevor reservierte Literale abgelehnt werden; anschließend wird bei einem
  Fehlschlag im Verzeichnis der Zugriff verweigert.
- `messaging.targetResolver.resolveTarget(...)` ist der Plugin-Rückfall, wenn
  der Kern nach der Normalisierung oder einem Fehlschlag im Verzeichnis eine abschließende, dem Provider
  zugeordnete Auflösung benötigt.
- `messaging.resolveOutboundSessionRoute(...)` besitzt die Provider-spezifische Konstruktion der
  Sitzungsroute, sobald ein Ziel aufgelöst wurde.

Empfohlene Aufteilung:

- Verwenden Sie `inferTargetChatType` für Kategorieentscheidungen, die vor
  der Suche nach Peers/Gruppen erfolgen sollen.
- Verwenden Sie `looksLikeId` für Prüfungen nach dem Muster „Dies als explizite/native Ziel-ID behandeln“.
- Verwenden Sie `resolveTarget` als Provider-spezifischen Rückfall für die Normalisierung, nicht für
  eine umfassende Verzeichnissuche.
- Bewahren Sie Provider-native IDs wie Chat-IDs, Thread-IDs, JIDs, Handles und Raum-IDs
  in `target`-Werten oder Provider-spezifischen Parametern auf, nicht in generischen SDK-Feldern.

## Konfigurationsgestützte Verzeichnisse

Plugins, die Verzeichniseinträge aus der Konfiguration ableiten, sollten diese Logik im
Plugin belassen und die gemeinsamen Hilfsfunktionen aus
`openclaw/plugin-sdk/directory-runtime` wiederverwenden.

Verwenden Sie dies, wenn ein Kanal konfigurationsgestützte Peers/Gruppen benötigt, beispielsweise:

- durch eine Positivliste gesteuerte DM-Peers
- konfigurierte Kanal-/Gruppenzuordnungen
- kontobezogene statische Verzeichnisrückfälle

Die gemeinsamen Hilfsfunktionen in `directory-runtime` verarbeiten ausschließlich generische Operationen:

- Abfragefilterung
- Anwendung von Begrenzungen
- Hilfsfunktionen für Deduplizierung/Normalisierung
- Erstellung von `ChannelDirectoryEntry[]`

Kanalspezifische Kontoprüfung und ID-Normalisierung sollten in der
Plugin-Implementierung verbleiben.

## Provider-Kataloge

Provider-Plugins können mit `registerProvider({ catalog: { run(...) { ... } } })`
Modellkataloge für Inferenz definieren.

`catalog.run(...)` gibt dieselbe Struktur zurück, die OpenClaw in
`models.providers` schreibt:

- `{ provider }` für einen Provider-Eintrag
- `{ providers }` für mehrere Provider-Einträge

Verwenden Sie `catalog`, wenn das Plugin Provider-spezifische Modell-IDs, Standardwerte für die Basis-URL
oder authentifizierungsabhängige Modellmetadaten verwaltet.

`catalog.order` steuert, wann der Katalog eines Plugins relativ zu den integrierten
impliziten Providern von OpenClaw zusammengeführt wird:

- `simple`: einfache API-Schlüssel- oder umgebungsgesteuerte Provider
- `profile`: Provider, die angezeigt werden, wenn Authentifizierungsprofile vorhanden sind
- `paired`: Provider, die mehrere zusammengehörige Provider-Einträge erzeugen
- `late`: letzter Durchlauf nach anderen impliziten Providern

Bei Schlüsselkollisionen haben später geladene Provider Vorrang, sodass Plugins einen
integrierten Provider-Eintrag mit derselben Provider-ID absichtlich überschreiben können.

Plugins können außerdem schreibgeschützte Modellzeilen über
`api.registerModelCatalogProvider({ provider, kinds, staticCatalog, liveCatalog
})` veröffentlichen. Dies ist der vorgesehene Pfad für Listen-, Hilfe- und Auswahloberflächen und unterstützt
Zeilen vom Typ `text`, `voice`, `image_generation`, `video_generation` und `music_generation`.
Provider-Plugins bleiben für Live-Endpunktaufrufe, den Token-Austausch und
die Zuordnung von Anbieterantworten zuständig; der Core verwaltet die gemeinsame Zeilenstruktur, Quellenbezeichnungen und
die Formatierung der Hilfe für Medienwerkzeuge. Registrierungen von Providern zur Mediengenerierung erzeugen
automatisch statische Katalogzeilen aus `defaultModel`, `models` und
`capabilities`.

Kompatibilität:

- `discovery` funktioniert weiterhin als Legacy-Alias, gibt jedoch eine Veraltungswarnung aus
- wenn sowohl `catalog` als auch `discovery` registriert sind, verwendet OpenClaw `catalog`
  und gibt eine Warnung aus
- `augmentModelCatalog` ist veraltet; gebündelte Provider sollten
  ergänzende Zeilen über `registerModelCatalogProvider` veröffentlichen

## Schreibgeschützte Kanalprüfung

Wenn Ihr Plugin einen Kanal registriert, sollten Sie
`plugin.config.inspectAccount(cfg, accountId)` zusammen mit `resolveAccount(...)` implementieren.

Gründe:

- `resolveAccount(...)` ist der Laufzeitpfad. Er darf davon ausgehen, dass Anmeldedaten
  vollständig materialisiert sind, und kann schnell fehlschlagen, wenn erforderliche Geheimnisse fehlen.
- Schreibgeschützte Befehlspfade wie `openclaw status`, `openclaw status --all`,
  `openclaw channels status`, `openclaw channels resolve` sowie Reparaturabläufe für
  Doctor und Konfiguration sollten Laufzeitanmeldedaten nicht materialisieren müssen, nur um
  die Konfiguration zu beschreiben.

Empfohlenes Verhalten für `inspectAccount(...)`:

- Geben Sie nur einen beschreibenden Kontostatus zurück.
- Behalten Sie `enabled` und `configured` bei.
- Fügen Sie gegebenenfalls Felder für Quelle und Status der Anmeldedaten hinzu, beispielsweise:
  - `tokenSource`, `tokenStatus`
  - `botTokenSource`, `botTokenStatus`
  - `appTokenSource`, `appTokenStatus`
  - `signingSecretSource`, `signingSecretStatus`
- Sie müssen keine Rohwerte von Tokens zurückgeben, nur um die schreibgeschützte
  Verfügbarkeit zu melden. Die Rückgabe von `tokenStatus: "available"` (und des zugehörigen
  Quellenfelds) reicht für statusorientierte Befehle aus.
- Verwenden Sie `configured_unavailable`, wenn Anmeldedaten über SecretRef konfiguriert,
  im aktuellen Befehlspfad jedoch nicht verfügbar sind.

Dadurch können schreibgeschützte Befehle „konfiguriert, aber in diesem Befehlspfad
nicht verfügbar“ melden, anstatt abzustürzen oder das Konto fälschlicherweise als nicht konfiguriert zu melden.

## Paket-Packs

Ein Plugin-Verzeichnis kann eine `package.json` mit `openclaw.extensions` enthalten:

```json
{
  "name": "my-pack",
  "openclaw": {
    "extensions": ["./src/safety.ts", "./src/tools.ts"],
    "setupEntry": "./src/setup-entry.ts"
  }
}
```

Jeder Eintrag wird zu einem Plugin. Wenn das Pack mehrere Erweiterungen aufführt, wird die Plugin-ID
zu `<manifestOrPackageName>/<fileBase>` (die Manifest-ID hat Vorrang, wenn
sie vorhanden ist; andernfalls wird der nicht bereichsgebundene Name `package.json` verwendet).

Wenn Ihr Plugin npm-Abhängigkeiten importiert, installieren Sie diese in diesem Verzeichnis, damit
`node_modules` verfügbar ist (`npm install` / `pnpm install`).

Sicherheitsvorgabe: Jeder `openclaw.extensions`-Eintrag muss nach der Auflösung symbolischer Links innerhalb des Plugin-
Verzeichnisses bleiben. Einträge, die das Paketverzeichnis verlassen, werden
abgelehnt.

Sicherheitshinweis: `openclaw plugins install` installiert Plugin-Abhängigkeiten mit einer
projektlokalen `npm install --omit=dev --ignore-scripts` (keine Lebenszyklusskripte,
keine Entwicklungsabhängigkeiten zur Laufzeit) und ignoriert dabei geerbte globale npm-Installationseinstellungen.
Halten Sie die Abhängigkeitsbäume von Plugins „reines JS/TS“ und vermeiden Sie Pakete, die
`postinstall`-Builds erfordern.

Optional: `openclaw.setupEntry` kann auf ein schlankes, ausschließlich für die Einrichtung vorgesehenes Modul verweisen.
Wenn OpenClaw Einrichtungsoberflächen für ein deaktiviertes Kanal-Plugin benötigt oder
wenn ein Kanal-Plugin aktiviert, aber noch nicht konfiguriert ist, lädt es `setupEntry`
anstelle des vollständigen Plugin-Eintrags. Dadurch bleiben Start und Einrichtung schlanker,
wenn Ihr Haupt-Plugin-Eintrag außerdem Werkzeuge, Hooks oder anderen ausschließlich zur Laufzeit benötigten
Code einbindet.

Optional: `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen`
kann ein Kanal-Plugin während der Startphase vor dem Lauschen des Gateways für denselben
`setupEntry`-Pfad aktivieren, selbst wenn der Kanal bereits konfiguriert ist.

Verwenden Sie dies nur, wenn `setupEntry` die Startoberfläche vollständig abdeckt, die vorhanden sein muss,
bevor das Gateway zu lauschen beginnt. In der Praxis bedeutet dies, dass der Einrichtungseintrag
jede kanaleigene Fähigkeit registrieren muss, von der der Start abhängt, beispielsweise:

- die Kanalregistrierung selbst
- alle HTTP-Routen, die verfügbar sein müssen, bevor das Gateway zu lauschen beginnt
- alle Gateway-Methoden, Werkzeuge oder Dienste, die während desselben Zeitfensters vorhanden sein müssen

Wenn Ihr vollständiger Eintrag weiterhin eine erforderliche Startfähigkeit verwaltet, aktivieren Sie
dieses Flag nicht. Behalten Sie das Standardverhalten des Plugins bei und lassen Sie OpenClaw während des
Starts den vollständigen Eintrag laden.

Gebündelte Kanäle können außerdem ausschließlich für die Einrichtung vorgesehene Hilfsfunktionen für Vertragsoberflächen veröffentlichen, die der Core
abfragen kann, bevor die vollständige Kanallaufzeit geladen wird. Die aktuelle
Einrichtungsoberfläche für die Heraufstufung ist:

- `singleAccountKeysToMove`
- `namedAccountPromotionKeys`
- `resolveSingleAccountPromotionTarget(...)`

Der Core verwendet diese Oberfläche, wenn er eine Legacy-Kanalkonfiguration für ein einzelnes Konto in
`channels.<id>.accounts.*` überführen muss, ohne den vollständigen Plugin-Eintrag zu laden.
Matrix ist das aktuelle gebündelte Beispiel: Wenn benannte Konten bereits vorhanden sind, verschiebt es nur
Authentifizierungs-/Bootstrap-Schlüssel in ein benanntes heraufgestuftes Konto und kann einen
konfigurierten, nicht kanonischen Schlüssel für das Standardkonto beibehalten, statt immer
`accounts.default` zu erstellen.

Diese Einrichtungs-Patch-Adapter sorgen dafür, dass die Erkennung gebündelter Vertragsoberflächen verzögert erfolgt. Die Importzeit
bleibt kurz; die Heraufstufungsoberfläche wird erst bei der ersten Verwendung geladen, statt
beim Modulimport den Start gebündelter Kanäle erneut auszuführen.

Wenn diese Startoberflächen Gateway-RPC-Methoden enthalten, verwenden Sie dafür ein
Plugin-spezifisches Präfix. Die Core-Administrationsnamensräume (`config.*`,
`exec.approvals.*`, `wizard.*`, `update.*`) bleiben reserviert und werden immer zu
`operator.admin` aufgelöst, selbst wenn ein Plugin einen engeren Geltungsbereich anfordert.

Beispiel:

```json
{
  "name": "@scope/my-channel",
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "startup": {
      "deferConfiguredChannelFullLoadUntilAfterListen": true
    }
  }
}
```

### Metadaten des Kanalkatalogs

Kanal-Plugins können Einrichtungs-/Erkennungsmetadaten über `openclaw.channel` und
Installationshinweise über `openclaw.install` bereitstellen. Dadurch enthält der Core-Katalog keine Daten.

Beispiel:

```json
{
  "name": "@openclaw/nextcloud-talk",
  "openclaw": {
    "extensions": ["./index.ts"],
    "channel": {
      "id": "nextcloud-talk",
      "label": "Nextcloud Talk",
      "selectionLabel": "Nextcloud Talk (selbst gehostet)",
      "docsPath": "/channels/nextcloud-talk",
      "docsLabel": "nextcloud-talk",
      "blurb": "Selbst gehosteter Chat über Nextcloud-Talk-Webhook-Bots.",
      "order": 65,
      "aliases": ["nc-talk", "nc"]
    },
    "install": {
      "npmSpec": "@openclaw/nextcloud-talk",
      "localPath": "<bundled-plugin-local-path>",
      "defaultChoice": "npm"
    }
  }
}
```

Nützliche `openclaw.channel`-Felder über das Minimalbeispiel hinaus:

- `detailLabel`: sekundäre Bezeichnung für umfangreichere Katalog-/Statusoberflächen
- `docsLabel`: Linktext für den Dokumentationslink überschreiben
- `preferOver`: Plugin-/Kanal-IDs mit niedrigerer Priorität, die dieser Katalogeintrag übertreffen soll
- `selectionDocsPrefix`, `selectionDocsOmitLabel`, `selectionExtras`: Steuerung der Texte auf Auswahloberflächen
- `markdownCapable`: kennzeichnet den Kanal für Entscheidungen zur ausgehenden Formatierung als Markdown-fähig
- `exposure.configured`: blendet den Kanal auf Listenoberflächen für konfigurierte Kanäle aus, wenn auf `false` gesetzt
- `exposure.setup`: blendet den Kanal in interaktiven Auswahlfeldern für Einrichtung/Konfiguration aus, wenn auf `false` gesetzt
- `exposure.docs`: kennzeichnet den Kanal für Dokumentationsnavigationsoberflächen als intern/privat
- `quickstartAllowFrom`: nimmt den Kanal in den standardmäßigen Schnellstartablauf `allowFrom` auf
- `forceAccountBinding`: erfordert eine explizite Kontobindung, selbst wenn nur ein Konto vorhanden ist
- `preferSessionLookupForAnnounceTarget`: bevorzugt bei der Auflösung von Ankündigungszielen die Sitzungssuche

OpenClaw kann außerdem **externe Kanalkataloge** zusammenführen (beispielsweise einen Export aus einer MPM-
Registry). Legen Sie eine JSON-Datei an einem der folgenden Orte ab:

- `~/.openclaw/mpm/plugins.json`
- `~/.openclaw/mpm/catalog.json`
- `~/.openclaw/plugins/catalog.json`

Alternativ können Sie `OPENCLAW_PLUGIN_CATALOG_PATHS` (oder `OPENCLAW_MPM_CATALOG_PATHS`) auf
eine oder mehrere JSON-Dateien verweisen lassen (durch Kommas, Semikolons oder `PATH` getrennt). Jede Datei sollte
`{ "entries": [ { "name": "@scope/pkg", "openclaw": { "channel": {...}, "install": {...} } } ] }` enthalten. Der Parser akzeptiert außerdem `"packages"` oder `"plugins"` als Legacy-Aliasse für den Schlüssel `"entries"`.

Generierte Kanalkatalogeinträge und Katalogeinträge für Provider-Installationen stellen
normalisierte Fakten zur Installationsquelle neben dem unverarbeiteten `openclaw.install`-Block bereit. Die
normalisierten Fakten geben an, ob die npm-Spezifikation eine exakte Version oder ein variabler
Selektor ist, ob die erwarteten Integritätsmetadaten vorhanden sind und ob außerdem ein lokaler
Quellpfad verfügbar ist. Wenn die Katalog-/Paketidentität bekannt ist, warnen die
normalisierten Fakten, falls der geparste npm-Paketname von dieser Identität abweicht.
Sie warnen außerdem, wenn `defaultChoice` ungültig ist oder auf eine nicht verfügbare
Quelle verweist sowie wenn npm-Integritätsmetadaten ohne eine gültige npm-
Quelle vorhanden sind. Verbraucher sollten `installSource` als additives optionales Feld behandeln, damit
manuell erstellte Einträge und Katalog-Shims es nicht erzeugen müssen.
Dadurch können Onboarding und Diagnosen den Zustand der Quellenebene erläutern, ohne
die Plugin-Laufzeit zu importieren.

Offizielle externe npm-Einträge sollten eine exakte `npmSpec` zusammen mit
`expectedIntegrity` bevorzugen. Reine Paketnamen und Dist-Tags funktionieren aus
Kompatibilitätsgründen weiterhin, zeigen jedoch Warnungen auf Quellenebene an, sodass sich der Katalog zu
fixierten, integritätsgeprüften Installationen weiterentwickeln kann, ohne vorhandene Plugins zu beeinträchtigen.
Wenn das Onboarding aus einem lokalen Katalogpfad installiert, zeichnet es einen verwalteten Eintrag im Plugin-
Index mit `source: "path"` und nach Möglichkeit einem arbeitsbereichsrelativen
`sourcePath` auf. Der absolute operative Ladepfad verbleibt in
`plugins.load.paths`; der Installationseintrag vermeidet es, lokale Pfade der Arbeitsstation
in die langfristige Konfiguration zu duplizieren. Dadurch bleiben lokale Entwicklungsinstallationen für
Diagnosen auf Quellenebene sichtbar, ohne eine zweite Oberfläche zur Offenlegung unverarbeiteter Dateisystempfade
hinzuzufügen. Die persistierte SQLite-Tabelle `installed_plugin_index` ist die maßgebliche Quelle für
Installationen und kann aktualisiert werden, ohne Plugin-Laufzeitmodule zu laden.
Ihre `installRecords`-Zuordnung bleibt auch dann dauerhaft erhalten, wenn ein Plugin-Manifest fehlt oder
ungültig ist; ihre `plugins`-Nutzlast ist eine wiederherstellbare Manifestansicht.

## Plugins für die Kontext-Engine

Plugins für die Kontext-Engine verwalten die Orchestrierung des Sitzungskontexts für Aufnahme, Zusammenstellung
und Compaction. Registrieren Sie sie aus Ihrem Plugin mit
`api.registerContextEngine(id, factory)` und wählen Sie anschließend die aktive Engine mit
`plugins.slots.contextEngine` aus.

Verwenden Sie dies, wenn Ihr Plugin die standardmäßige Kontext-
Pipeline ersetzen oder erweitern muss, statt lediglich eine Speichersuche oder Hooks hinzuzufügen.

```ts
import { buildMemorySystemPromptAddition } from "openclaw/plugin-sdk/core";

export default function (api) {
  api.registerContextEngine("lossless-claw", (ctx) => ({
    info: { id: "lossless-claw", name: "Lossless Claw", ownsCompaction: true },
    async ingest() {
      return { ingested: true };
    },
    async assemble({ messages, sessionKey, availableTools, citationsMode }) {
      return {
        messages,
        estimatedTokens: 0,
        systemPromptAddition: buildMemorySystemPromptAddition({
          availableTools: availableTools ?? new Set(),
          citationsMode,
          agentSessionKey: sessionKey,
        }),
      };
    },
    async compact() {
      return { ok: true, compacted: false };
    },
  }));
}
```

Die Factory `ctx` stellt optionale Werte für `config`, `agentDir` und `workspaceDir`
zur Initialisierung bei der Konstruktion bereit.

Der Host schließt die registrierte asynchrone Vorbereitung des Memory-Prompts ab, bevor er
`assemble()` einer nicht veralteten Engine aufruft. `buildMemorySystemPromptAddition(...)` bleibt
synchron und liest diesen unveränderlichen Lauf-Snapshot, während `assemble()` aktiv ist.
Reichen Sie den bereitgestellten Werkzeug- und Zitationskontext unverändert weiter, damit der Snapshot
keine Laufgrenzen überschreiten kann.

`assemble()` kann `contextProjection` zurückgeben, wenn das aktive Harness über einen
persistenten Backend-Thread verfügt. Lassen Sie es bei der veralteten Projektion pro Durchlauf weg. Geben Sie
`{ mode: "thread_bootstrap", epoch }` zurück, wenn der zusammengesetzte Kontext einmalig in einen
Backend-Thread eingefügt und wiederverwendet werden soll, bis sich die Epoche ändert. Ändern Sie
die Epoche, nachdem sich der semantische Kontext der Engine geändert hat, beispielsweise nach einem
von der Engine verwalteten Compaction-Durchlauf. Hosts können Metadaten von Werkzeugaufrufen, die Eingabeform
und redigierte Werkzeugergebnisse in einer Thread-Bootstrap-Projektion beibehalten, damit neue
Backend-Threads die Werkzeugkontinuität bewahren, ohne unverarbeitete, geheimnistragende
Payloads zu kopieren.

Wenn Ihre Engine den Compaction-Algorithmus **nicht** verwaltet, behalten Sie die Implementierung von `compact()`
bei und delegieren Sie ihn ausdrücklich:

```ts
import {
  buildMemorySystemPromptAddition,
  delegateCompactionToRuntime,
} from "openclaw/plugin-sdk/core";

export default function (api) {
  api.registerContextEngine("my-memory-engine", (ctx) => ({
    info: {
      id: "my-memory-engine",
      name: "My Memory Engine",
      ownsCompaction: false,
    },
    async ingest() {
      return { ingested: true };
    },
    async assemble({ messages, sessionKey, availableTools, citationsMode }) {
      return {
        messages,
        estimatedTokens: 0,
        systemPromptAddition: buildMemorySystemPromptAddition({
          availableTools: availableTools ?? new Set(),
          citationsMode,
          agentSessionKey: sessionKey,
        }),
      };
    },
    async compact(params) {
      return await delegateCompactionToRuntime(params);
    },
  }));
}
```

## Neue Capability hinzufügen

Wenn ein Plugin ein Verhalten benötigt, das nicht zur aktuellen API passt, umgehen Sie
das Plugin-System nicht durch einen privaten Direktzugriff. Fügen Sie die fehlende Capability hinzu.

Empfohlene Reihenfolge:

1. **Definieren Sie den Core-Vertrag.** Legen Sie fest, welches gemeinsame Verhalten der Core verwalten soll:
   Richtlinie, Fallback, Konfigurationszusammenführung, Lebenszyklus, kanalbezogene Semantik und
   Form der Runtime-Hilfsfunktion.
2. **Fügen Sie typisierte Oberflächen für Plugin-Registrierung und Runtime hinzu.** Erweitern Sie
   `OpenClawPluginApi` und/oder `api.runtime` um die kleinste zweckmäßige typisierte
   Capability-Oberfläche.
3. **Binden Sie Core- sowie Kanal-/Feature-Consumer an.** Kanäle und Feature-Plugins
   sollten die neue Capability über den Core nutzen, statt die Implementierung eines Providers
   direkt zu importieren.
4. **Registrieren Sie Provider-Implementierungen.** Provider-Plugins registrieren anschließend ihre
   Backends für die Capability.
5. **Fügen Sie Vertragsabdeckung hinzu.** Fügen Sie Tests hinzu, damit Eigentümerschaft und Registrierungsform
   dauerhaft explizit bleiben.

So bleibt OpenClaw meinungsstark, ohne fest auf die Sichtweise eines einzelnen
Providers zugeschnitten zu werden. Eine konkrete Datei-Checkliste und ein ausgearbeitetes Beispiel finden Sie im [Capability-Kochbuch](/de/plugins/adding-capabilities).

### Capability-Checkliste

Wenn Sie eine neue Capability hinzufügen, sollte die Implementierung üblicherweise diese
Oberflächen gemeinsam berühren:

- Core-Vertragstypen in `src/<capability>/types.ts`
- Core-Runner/-Runtime-Hilfsfunktion in `src/<capability>/runtime.ts`
- Registrierungsoberfläche der Plugin-API in `src/plugins/types.ts`
- Verdrahtung der Plugin-Registry in `src/plugins/registry.ts`
- Runtime-Bereitstellung des Plugins in `src/plugins/runtime/*`, wenn Feature-/Kanal-
  Plugins sie nutzen müssen
- Erfassungs-/Testhilfen in `src/test-utils/plugin-registration.ts`
- Eigentümerschafts-/Vertragszusicherungen in `src/plugins/contracts/registry.ts`
- Dokumentation für Betreiber/Plugins in `docs/`

Wenn eine dieser Oberflächen fehlt, ist dies üblicherweise ein Zeichen dafür, dass die Capability
noch nicht vollständig integriert ist.

### Capability-Vorlage

Minimales Muster:

```ts
// core contract
export type VideoGenerationProviderPlugin = {
  id: string;
  label: string;
  generateVideo: (req: VideoGenerationRequest) => Promise<VideoGenerationResult>;
};

// plugin API
api.registerVideoGenerationProvider({
  id: "openai",
  label: "OpenAI",
  async generateVideo(req) {
    return await generateOpenAiVideo(req);
  },
});

// shared runtime helper for feature/channel plugins
const clip = await api.runtime.videoGeneration.generate({
  prompt: "Show the robot walking through the lab.",
  cfg,
});
```

Vertragstestmuster (`src/plugins/contracts/registry.ts` stellt Eigentümerschaftsabfragen
wie `providerContractPluginIds` bereit; Tests bestätigen, dass die
`contracts.videoGenerationProviders`-Liste eines Plugins mit seinen tatsächlichen Registrierungen übereinstimmt):

```ts
expect(pluginManifest.contracts?.videoGenerationProviders).toEqual(["openai"]);
```

Dadurch bleibt die Regel einfach:

- Der Core verwaltet den Capability-Vertrag und die Orchestrierung
- Provider-Plugins verwalten die Provider-Implementierungen
- Feature-/Kanal-Plugins nutzen Runtime-Hilfsfunktionen
- Vertragstests halten die Eigentümerschaft explizit fest

## Verwandte Themen

- [Plugin-Architektur](/de/plugins/architecture) — öffentliches Capability-Modell und Formen
- [Unterpfade des Plugin SDK](/de/plugins/sdk-subpaths)
- [Einrichtung des Plugin SDK](/de/plugins/sdk-setup)
- [Plugins erstellen](/de/plugins/building-plugins)
