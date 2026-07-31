---
read_when:
    - Sie entwickeln ein OpenClaw-Plugin
    - Sie müssen ein Plugin-Konfigurationsschema bereitstellen oder Validierungsfehler des Plugins beheben.
summary: Plugin-Manifest + Anforderungen an das JSON-Schema (strikte Konfigurationsvalidierung)
title: Plugin-Manifest
x-i18n:
    generated_at: "2026-07-26T17:56:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 244e5c8265ff79b0ff6e8f4b60c9635cccc3ba66093cecab458676beb9578264
    source_path: plugins/manifest.md
    workflow: 16
---

Diese Seite behandelt das **native OpenClaw-Plugin-Manifest**, `openclaw.plugin.json`. Informationen zu kompatiblen Bundle-Layouts (Codex, Claude, Cursor) finden Sie unter [Plugin-Bundles](/de/plugins/bundles).

Kompatible Bundle-Formate verwenden stattdessen eigene Manifestdateien:

- Codex-Bundle: `.codex-plugin/plugin.json`
- Claude-Bundle: `.claude-plugin/plugin.json` oder das standardmäßige Claude-Komponentenlayout ohne Manifest
- Cursor-Bundle: `.cursor-plugin/plugin.json`

OpenClaw erkennt diese Layouts automatisch, validiert sie jedoch nicht anhand des unten aufgeführten `openclaw.plugin.json`-Schemas. Bei einem kompatiblen Bundle liest OpenClaw Bundle-Metadaten, deklarierte Skill-Stammverzeichnisse, Claude-Befehlsstammverzeichnisse, Claude-`settings.json`-Standardwerte, Claude-LSP-Standardwerte und unterstützte Hook-Pakete, sofern das Layout den Laufzeiterwartungen von OpenClaw entspricht.

Jedes native OpenClaw-Plugin **muss** `openclaw.plugin.json` im **Plugin-Stammverzeichnis** enthalten. OpenClaw liest diese Datei, um die Konfiguration **ohne Ausführung des Plugin-Codes** zu validieren. Ein fehlendes oder ungültiges Manifest verhindert die Konfigurationsvalidierung und wird als Plugin-Fehler behandelt.

Den vollständigen Leitfaden zum Plugin-System finden Sie unter [Plugins](/de/tools/plugin), Informationen zum nativen Capability-Modell und zur aktuellen Kompatibilität mit externen Formaten unter [Capability-Modell](/de/plugins/architecture#public-capability-model).

## Zweck dieser Datei

`openclaw.plugin.json` enthält Metadaten, die OpenClaw **vor dem Laden Ihres Plugin-Codes** liest. Alle enthaltenen Informationen müssen sich ohne Starten der Plugin-Laufzeitumgebung mit geringem Aufwand prüfen lassen.

**Verwenden Sie die Datei für:**

- Plugin-Identität, Konfigurationsvalidierung und Hinweise für die Konfigurationsoberfläche
- Metadaten für Authentifizierung, Onboarding und Einrichtung (Alias, automatische Aktivierung, Provider-Umgebungsvariablen, Authentifizierungsoptionen)
- Aktivierungshinweise für Control-Plane-Oberflächen
- Zuordnung abgekürzter Modellfamilien
- statische Snapshots der Capability-Zuständigkeit (`contracts`)
- Datenbindungen und Aktionsverben für Dashboard-Widgets
- statische MCP-Server, die verfügbar sein sollen, während das Plugin aktiviert ist
- QA-Runner-Metadaten, die der gemeinsame `openclaw qa`-Host prüfen kann
- kanalspezifische Konfigurationsmetadaten, die in Katalog- und Validierungsoberflächen zusammengeführt werden

**Verwenden Sie die Datei nicht für:** die Registrierung nativer Laufzeit-Hooks, die Deklaration von Einstiegspunkten für Plugin-Code oder npm-Installationsmetadaten. Diese gehören in Ihren Plugin-Code und in `package.json`.

## Minimales Beispiel

```json
{
  "id": "voice-call",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

## Umfangreiches Beispiel

```json
{
  "id": "openrouter",
  "name": "OpenRouter",
  "description": "OpenRouter-Provider-Plugin",
  "version": "1.0.0",
  "providers": ["openrouter"],
  "modelSupport": {
    "modelPrefixes": ["router-"]
  },
  "modelIdNormalization": {
    "providers": {
      "openrouter": {
        "prefixWhenBare": "openrouter"
      }
    }
  },
  "providerEndpoints": [
    {
      "endpointClass": "openrouter",
      "hostSuffixes": ["openrouter.ai"]
    }
  ],
  "providerRequest": {
    "providers": {
      "openrouter": {
        "family": "openrouter"
      }
    }
  },
  "cliBackends": ["openrouter-cli"],
  "syntheticAuthRefs": ["openrouter-cli"],
  "setup": {
    "providers": [
      {
        "id": "openrouter",
        "envVars": ["OPENROUTER_API_KEY"]
      }
    ]
  },
  "providerAuthAliases": {
    "openrouter-coding": "openrouter"
  },
  "providerAuthChoices": [
    {
      "provider": "openrouter",
      "method": "api-key",
      "choiceId": "openrouter-api-key",
      "choiceLabel": "OpenRouter-API-Schlüssel",
      "groupId": "openrouter",
      "groupLabel": "OpenRouter",
      "optionKey": "openrouterApiKey",
      "cliFlag": "--openrouter-api-key",
      "cliOption": "--openrouter-api-key <key>",
      "cliDescription": "OpenRouter-API-Schlüssel",
      "onboardingScopes": ["text-inference"]
    }
  ],
  "uiHints": {
    "apiKey": {
      "label": "API-Schlüssel",
      "placeholder": "sk-or-v1-...",
      "sensitive": true
    }
  },
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "apiKey": {
        "type": "string"
      }
    }
  }
}
```

## Referenz der Felder auf oberster Ebene

| Feld                                 | Erforderlich | Typ                          | Bedeutung                                                                                                                                                                                                                                                                                      |
| ------------------------------------ | ------------ | ---------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`                   | Ja           | `string`           | Kanonische Plugin-ID. Dies ist die in `plugins.entries.<id>` verwendete ID.                                                                                                                                                                                                                         |
| `configSchema`                   | Ja           | `object`           | Inline-JSON-Schema für die Konfiguration dieses Plugins.                                                                                                                                                                                                                                        |
| `requiresPlugins`                   | Nein         | `string[]`           | Plugin-IDs, die ebenfalls installiert sein müssen, damit dieses Plugin wirksam wird. Die Erkennung lässt das Plugin weiterhin laden, warnt jedoch, wenn ein erforderliches Plugin fehlt.                                                                                                         |
| `enabledByDefault`                   | Nein         | `true`           | Kennzeichnet ein gebündeltes Plugin als standardmäßig aktiviert. Lassen Sie den Wert weg oder setzen Sie einen anderen Wert als `true`, damit das Plugin standardmäßig deaktiviert bleibt.                                                                                           |
| `enabledByDefaultOnPlatforms`                   | Nein         | `string[]`           | Kennzeichnet ein gebündeltes Plugin nur auf den aufgeführten Node.js-Plattformen als standardmäßig aktiviert, beispielsweise `["darwin"]`. Eine explizite Konfiguration hat weiterhin Vorrang.                                                                                             |
| `legacyPluginIds`                   | Nein         | `string[]`           | Veraltete IDs, die auf diese kanonische Plugin-ID normalisiert werden.                                                                                                                                                                                                                           |
| `autoEnableWhenConfiguredProviders`                   | Nein         | `string[]`           | Provider-IDs, bei deren Erwähnung in Authentifizierung, Konfiguration oder Modellreferenzen dieses Plugin automatisch aktiviert werden soll.                                                                                                                                                    |
| `kind`                   | Nein         | `PluginKind \| PluginKind[]`           | Deklariert eine oder mehrere exklusive Plugin-Arten (`"memory"`, `"context-engine"`), die von `plugins.slots.*` verwendet werden. Ein Plugin, dem beide Slots gehören, deklariert beide Arten in einem einzigen Array.                                                                     |
| `channels`                   | Nein         | `string[]`           | Kanal-IDs, die diesem Plugin gehören. Werden für die Erkennung und Konfigurationsvalidierung verwendet.                                                                                                                                                                                          |
| `providers`                   | Nein         | `string[]`           | Provider-IDs, die diesem Plugin gehören.                                                                                                                                                                                                                                                        |
| `providerCatalogEntry`                   | Nein         | `string`           | Relativ zum Plugin-Stammverzeichnis angegebener Pfad zu einem schlanken Provider-Katalogmodul für manifestbezogene Provider-Katalogmetadaten, die geladen werden können, ohne die vollständige Plugin-Laufzeit zu aktivieren.                                                                    |
| `modelSupport`                   | Nein         | `object`           | Manifestverwaltete Kurzform-Metadaten zur Modellfamilie, mit denen das Plugin vor der Laufzeit automatisch geladen wird.                                                                                                                                                                        |
| `modelCatalog`                   | Nein         | `object`           | Deklarative Modellkatalogmetadaten für Provider, die diesem Plugin gehören. Dies ist der Control-Plane-Vertrag für zukünftige schreibgeschützte Auflistungen, das Onboarding, Modellauswahlfelder, Aliasse und Unterdrückungen, ohne die Plugin-Laufzeit zu laden.                                  |
| `modelPricing`                   | Nein         | `object`           | Providerverwaltete Richtlinie zur externen Preisabfrage. Verwenden Sie sie, um lokale oder selbst gehostete Provider von entfernten Preiskatalogen auszunehmen oder Providerreferenzen OpenRouter-/LiteLLM-Katalog-IDs zuzuordnen, ohne Provider-IDs im Kern fest zu codieren.                      |
| `modelIdNormalization`                   | Nein         | `object`           | Providerverwaltete Bereinigung von Modell-ID-Aliassen und -Präfixen, die ausgeführt werden muss, bevor die Provider-Laufzeit geladen wird.                                                                                                                                                        |
| `providerEndpoints`                   | Nein         | `object[]`           | Manifestverwaltete Metadaten zu Endpunkt-Hosts und baseUrl für Provider-Routen, die der Kern klassifizieren muss, bevor die Provider-Laufzeit geladen wird.                                                                                                                                      |
| `providerRequest`                   | Nein         | `object`           | Ressourcenschonende Metadaten zur Providerfamilie und Anfragekompatibilität, die von der generischen Anfragerichtlinie verwendet werden, bevor die Provider-Laufzeit geladen wird.                                                                                                               |
| `secretProviderIntegrations`                   | Nein         | `Record<string, object>`           | Deklarative Voreinstellungen für SecretRef-Ausführungs-Provider, die Einrichtungs- oder Installationsoberflächen anbieten können, ohne providerspezifische Integrationen im Kern fest zu codieren.                                                                                               |
| `cliBackends`                   | Nein         | `string[]`           | IDs der CLI-Inferenz-Backends, die diesem Plugin gehören. Werden zur automatischen Aktivierung beim Start anhand expliziter Konfigurationsreferenzen verwendet.                                                                                                                                 |
| `syntheticAuthRefs`                   | Nein         | `string[]`           | Provider- oder CLI-Backendreferenzen, deren pluginverwalteter Hook für synthetische Authentifizierung während der anfänglichen Modellerkennung geprüft werden soll, bevor die Laufzeit geladen wird.                                                                                            |
| `nonSecretAuthMarkers`                   | Nein         | `string[]`           | Platzhalterwerte für API-Schlüssel, die einem gebündelten Plugin gehören und einen nicht geheimen lokalen, OAuth- oder umgebungsbasierten Anmeldedatenstatus darstellen.                                                                                                                        |
| `commandAliases`                   | Nein         | `object[]`           | Befehlsnamen, die diesem Plugin gehören und pluginbezogene Konfigurations- und CLI-Diagnosen erzeugen sollen, bevor die Laufzeit geladen wird.                                                                                                                                                   |
| `providerUsageAuthEnvVars`                   | Nein         | `Record<string, string[]>`           | Provider-Anmeldedaten, die ausschließlich für Nutzung und Abrechnung vorgesehen sind. OpenClaw verwendet diese Namen zur Nutzungserkennung und zum Bereinigen von Geheimnissen, jedoch niemals zur Inferenz-Authentifizierung.                                                                  |
| `providerAuthAliases`                   | Nein         | `Record<string, string>`           | Provider-IDs, die für die Authentifizierungssuche eine andere Provider-ID wiederverwenden sollen, beispielsweise ein Coding-Provider, der den API-Schlüssel und die Authentifizierungsprofile des Basis-Providers gemeinsam nutzt.                                                             |
| `providerAuthChoices`                   | Nein         | `object[]`           | Ressourcenschonende Metadaten zu Authentifizierungsoptionen für Onboarding-Auswahlfelder, die Auflösung bevorzugter Provider und die einfache Anbindung von CLI-Flags.                                                                                                                           |
| `activation`                   | Nein         | `object`           | Ressourcenschonende Metadaten des Aktivierungsplaners für das durch Start, Provider, Befehl, Kanal, Route und Fähigkeit ausgelöste Laden. Nur Metadaten; das tatsächliche Verhalten verbleibt im Besitz der Plugin-Laufzeit.                                                                      |
| `setup`                   | Nein         | `object`           | Ressourcenschonende Einrichtungs- und Onboarding-Deskriptoren, die Erkennungs- und Einrichtungsoberflächen prüfen können, ohne die Plugin-Laufzeit zu laden.                                                                                                                                     |
| `qaRunners`                   | Nein         | `object[]`           | Ressourcenschonende Deskriptoren für QA-Runner, die vom gemeinsamen `openclaw qa`-Host verwendet werden, bevor die Plugin-Laufzeit geladen wird.                                                                                                                                           |
| `dashboard`                   | Nein         | `object`           | Datenbindungen und Aktionsverben für Dashboard-Widgets. Jeder Eintrag wird anhand einer von diesem Plugin registrierten Gateway-Methode mit dem erforderlichen Lese- oder Schreibbereich validiert. Siehe [Dashboard-Referenz](#dashboard-reference).                                             |
| `mcpServers`                         | Nein       | `Record<string, object>`     | Statische MCP-Serverdefinitionen, die bereitgestellt werden, solange dieses Plugin aktiviert ist. Relative Befehlsargumente und Arbeitsverzeichnisse werden vom Plugin-Stammverzeichnis aus aufgelöst. `mcp.servers`-Einträge des Betreibers überschreiben oder deaktivieren gleichnamige Definitionen. Siehe [MCP-Server-Referenz](#mcp-server-reference). |
| `contracts`                          | Nein       | `object`                     | Statische Momentaufnahme der Zuständigkeiten für externe Authentifizierungs-Hooks, Embeddings, Sprache, Echtzeittranskription, Echtzeitsprachübertragung, Medienverständnis, Bild-/Video-/Musikgenerierung, Webabruf, Websuche, Worker-Provider, Dokument-/Webinhalts-Extraktion und Tool-Zuständigkeiten.                     |
| `configContracts`                    | Nein       | `object`                     | Manifestgesteuertes Konfigurationsverhalten, das von generischen Core-Hilfsfunktionen verwendet wird: Erkennung gefährlicher Flags, Migrationsziele für SecretRef und Eingrenzung veralteter Konfigurationspfade. Siehe [configContracts-Referenz](#configcontracts-reference).                                                                         |
| `mediaUnderstandingProviderMetadata` | Nein       | `Record<string, object>`     | Kostengünstige Standardeinstellungen für das Medienverständnis bei Provider-IDs, die in `contracts.mediaUnderstandingProviders` deklariert sind.                                                                                                                                                                                       |
| `imageGenerationProviderMetadata`    | Nein       | `Record<string, object>`     | Kostengünstige Authentifizierungsmetadaten für die Bildgenerierung bei Provider-IDs, die in `contracts.imageGenerationProviders` deklariert sind, einschließlich providereigener Authentifizierungsaliase und Schutzprüfungen für Basis-URLs.                                                                                                                             |
| `videoGenerationProviderMetadata`    | Nein       | `Record<string, object>`     | Kostengünstige Authentifizierungsmetadaten für die Videogenerierung bei Provider-IDs, die in `contracts.videoGenerationProviders` deklariert sind, einschließlich providereigener Authentifizierungsaliase und Schutzprüfungen für Basis-URLs.                                                                                                                             |
| `musicGenerationProviderMetadata`    | Nein       | `Record<string, object>`     | Kostengünstige Authentifizierungsmetadaten für die Musikgenerierung bei Provider-IDs, die in `contracts.musicGenerationProviders` deklariert sind, einschließlich providereigener Authentifizierungsaliase und Schutzprüfungen für Basis-URLs.                                                                                                                             |
| `toolMetadata`                       | Nein       | `Record<string, object>`     | Kostengünstige Verfügbarkeitsmetadaten für plugineigene Tools, die in `contracts.tools` deklariert sind. Verwenden Sie sie, wenn ein Tool die Laufzeit nicht laden soll, sofern keine Konfigurations-, Umgebungs- oder Authentifizierungsnachweise vorhanden sind.                                                                                                                      |
| `channelConfigs`                     | Nein       | `Record<string, object>`     | Manifestgesteuerte Metadaten für die Kanalkonfiguration, die vor dem Laden der Laufzeit in Ermittlungs- und Validierungsoberflächen zusammengeführt werden.                                                                                                                                                                                     |
| `skills`                             | Nein       | `string[]`                   | Zu ladende Skill-Verzeichnisse, relativ zum Plugin-Stammverzeichnis.                                                                                                                                                                                                                                        |
| `name`                               | Nein       | `string`                     | Für Menschen lesbarer Plugin-Name.                                                                                                                                                                                                                                                                    |
| `description`                        | Nein       | `string`                     | Kurze Zusammenfassung, die auf Plugin-Oberflächen angezeigt wird.                                                                                                                                                                                                                                                        |
| `catalog`                            | Nein       | `object`                     | Optionale Darstellungshinweise für Plugin-Katalogoberflächen. Diese Metadaten installieren oder aktivieren kein Plugin und gewähren ihm kein Vertrauen.                                                                                                                                                                   |
| `icon`                               | Nein       | `string`                     | HTTPS-Bild-URL für Marketplace-/Katalogkarten. ClawHub akzeptiert jede gültige `https://`-URL und verwendet das standardmäßige Plugin-Symbol, wenn diese Angabe fehlt oder ungültig ist.                                                                                                                             |
| `version`                            | Nein       | `string`                     | Informative Plugin-Version.                                                                                                                                                                                                                                                                  |
| `uiHints`                            | Nein       | `Record<string, object>`     | UI-Beschriftungen, Platzhalter und Hinweise zur Vertraulichkeit von Konfigurationsfeldern.                                                                                                                                                                                                                              |

## MCP-Server-Referenz

`mcpServers` ermöglicht es einem nativen Plugin, einen MCP-Server einschließlich einer MCP App bereitzustellen, ohne dass Betreiber dessen statische Prozessdefinition in `openclaw.json` duplizieren müssen:

```json
{
  "mcpServers": {
    "example": {
      "transport": "stdio",
      "command": "node",
      "args": ["./mcp-server.js"]
    }
  }
}
```

OpenClaw bindet diese Server nur ein, solange das besitzende Plugin aktiviert ist. Relative Pfade für `command`, `args`, `cwd` und `workingDirectory` werden vom Plugin-Stammverzeichnis aus aufgelöst. Die Benutzerkonfiguration bleibt maßgeblich: `mcp.servers.<name>` kann einen Plugin-Standardwert ersetzen oder `enabled: false` festlegen, um ihn auszulassen. Das Rendern von MCP Apps und Aufrufe von Server-Tools erfordern weiterhin die normale MCP-Apps-Einstellung und die wirksame Tool-Richtlinie; die Deklaration eines Servers umgeht keine dieser beiden Grenzen.

## Dashboard-Referenz

`dashboard` ermöglicht es einem aktivierten Plugin, vorhandene Gateway-RPCs für berechtigte Dashboard-Widgets verfügbar zu machen, ohne Plugin-Richtlinien zum Kern hinzuzufügen. Datenbindungen müssen eine Methode benennen, die dasselbe Plugin mit `operator.read` registriert; Aktionsverben müssen eine Methode benennen, die es mit `operator.write` registriert. Bei einer Abweichung wird das Plugin während der Registrierung abgelehnt.

```json
{
  "dashboard": {
    "dataBindings": [
      {
        "id": "items.list",
        "method": "example.items.list",
        "description": "Beispielelemente auflisten."
      }
    ],
    "actionVerbs": [
      {
        "id": "refresh",
        "method": "example.items.refresh",
        "description": "Beispielelemente aktualisieren.",
        "paramShape": {
          "type": "object",
          "additionalProperties": false,
          "properties": {
            "force": { "type": "boolean" }
          }
        }
      }
    ]
  }
}
```

Die Manifest-IDs sind Plugin-lokal. Widget-Berechtigungen verwenden `<plugin-id>.<id>`, beispielsweise `example.items.list` und `example.refresh`. Damit der persistierte Berechtigungsnamensraum eindeutig bleibt, maskiert OpenClaw `%` und `.` im Plugin-ID-Segment als `%25` und `%2E`; gewöhnliche Plugin-IDs behalten die natürliche Form. `paramShape` ist ein optionales JSON Schema, das auf das Aktionsparameterobjekt angewendet wird, bevor OpenClaw den Plugin-RPC aufruft.

## Katalogreferenz

`catalog` stellt optionale Anzeigehinweise für Plugin-Browser bereit. Hosts können diese Hinweise ignorieren. Sie installieren oder aktivieren das Plugin niemals und ändern weder dessen Laufzeitverhalten noch dessen Vertrauensstufe.

```json
{
  "catalog": {
    "featured": true,
    "order": 10
  }
}
```

| Feld       | Typ       | Bedeutung                                                                  |
| ---------- | --------- | -------------------------------------------------------------------------- |
| `featured` | `boolean` | Ob Katalogoberflächen dieses Plugin hervorheben sollen.                    |
| `order`    | `number`  | Aufsteigender Anzeigehinweis unter kuratierten Plugins; niedrigere Werte erscheinen früher. |

## Referenz für Metadaten von Generierungs-Providern

Die Metadatenfelder für Generierungs-Provider beschreiben statische Authentifizierungssignale für Provider, die in der entsprechenden Liste `contracts.*GenerationProviders` deklariert sind. OpenClaw liest diese Felder, bevor die Provider-Laufzeit geladen wird, sodass Kern-Tools entscheiden können, ob ein Generierungs-Provider verfügbar ist, ohne jedes Provider-Plugin zu importieren.

Verwenden Sie diese Felder nur für kostengünstig ermittelbare, deklarative Fakten. Transport, Anfragetransformationen, Token-Aktualisierung, Anmeldedatenvalidierung und das eigentliche Generierungsverhalten verbleiben in der Plugin-Laufzeit.

```json
{
  "contracts": {
    "imageGenerationProviders": ["example-image"]
  },
  "imageGenerationProviderMetadata": {
    "example-image": {
      "aliases": ["example-image-oauth"],
      "authProviders": ["example-image"],
      "configSignals": [
        {
          "rootPath": "plugins.entries.example-image.config",
          "overlayPath": "image",
          "mode": {
            "path": "mode",
            "default": "local",
            "allowed": ["local"]
          },
          "requiredAny": ["workflow", "workflowPath"],
          "required": ["promptNodeId"]
        }
      ],
      "authSignals": [
        {
          "provider": "example-image"
        },
        {
          "provider": "example-image-oauth",
          "providerBaseUrl": {
            "provider": "example-image",
            "defaultBaseUrl": "https://api.example.com/v1",
            "allowedBaseUrls": ["https://api.example.com/v1"]
          }
        }
      ]
    }
  }
}
```

Jeder Metadateneintrag unterstützt:

| Feld                   | Erforderlich | Typ        | Bedeutung                                                                                                                                           |
| ---------------------- | ------------ | ---------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| `aliases`              | Nein         | `string[]` | Zusätzliche Provider-IDs, die als statische Authentifizierungsaliase für den Generierungs-Provider gelten sollen.                                    |
| `authProviders`        | Nein         | `string[]` | Provider-IDs, deren konfigurierte Authentifizierungsprofile als Authentifizierung für diesen Generierungs-Provider gelten sollen.                    |
| `configSignals`        | Nein         | `object[]` | Kostengünstige, ausschließlich konfigurationsbasierte Verfügbarkeitssignale für lokale oder selbst gehostete Provider, die ohne Authentifizierungsprofile oder Umgebungsvariablen konfiguriert werden können. |
| `authSignals`          | Nein         | `object[]` | Explizite Authentifizierungssignale. Wenn vorhanden, ersetzen diese den Standardsignalsatz aus der Provider-ID, `aliases` und `authProviders`. |
| `referenceAudioInputs` | Nein         | `boolean`  | Nur für Videogenerierung. Auf `true` setzen, wenn der Provider Referenzaudio-Assets akzeptiert; andernfalls blendet `video_generate` Audioreferenzparameter aus. |

Jeder `configSignals`-Eintrag unterstützt:

| Feld             | Erforderlich | Typ        | Bedeutung                                                                                                                                                                                 |
| ---------------- | ------------ | ---------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `rootPath`       | Ja           | `string`   | Punktpfad zum Plugin-eigenen Konfigurationsobjekt, das geprüft werden soll, beispielsweise `plugins.entries.example.config`.                                                                              |
| `overlayPath`    | Nein         | `string`   | Punktpfad innerhalb der Stammkonfiguration, dessen Objekt das Stammobjekt vor der Auswertung des Signals überlagern soll. Verwenden Sie dies für funktionsspezifische Konfigurationen wie `image`, `video` oder `music`. |
| `overlayMapPath` | Nein         | `string`   | Punktpfad innerhalb der Stammkonfiguration, dessen Objektwerte jeweils das Stammobjekt überlagern sollen. Verwenden Sie dies für benannte Kontozuordnungen wie `accounts`, bei denen jedes konfigurierte Konto ausreichen soll. |
| `required`       | Nein         | `string[]` | Punktpfade innerhalb der effektiven Konfiguration, die konfigurierte Werte enthalten müssen. Zeichenfolgen dürfen nicht leer sein; Objekte und Arrays dürfen nicht leer sein.              |
| `requiredAny`    | Nein         | `string[]` | Punktpfade innerhalb der effektiven Konfiguration, von denen mindestens einer einen konfigurierten Wert enthalten muss.                                                                    |
| `mode`           | Nein         | `object`   | Optionaler Zeichenfolgen-Moduswächter innerhalb der effektiven Konfiguration. Verwenden Sie diesen, wenn die rein konfigurationsbasierte Verfügbarkeit nur für einen Modus gilt.           |

Jeder `mode`-Wächter unterstützt:

| Feld         | Erforderlich | Typ        | Bedeutung                                                                          |
| ------------ | ------------ | ---------- | ---------------------------------------------------------------------------------- |
| `path`       | Nein         | `string`   | Punktpfad innerhalb der effektiven Konfiguration. Standardmäßig `mode`. |
| `default`    | Nein         | `string`   | Zu verwendender Moduswert, wenn der Pfad in der Konfiguration fehlt.                |
| `allowed`    | Nein         | `string[]` | Wenn vorhanden, besteht das Signal nur, wenn der effektive Modus einer dieser Werte ist. |
| `disallowed` | Nein         | `string[]` | Wenn vorhanden, schlägt das Signal fehl, wenn der effektive Modus einer dieser Werte ist. |

Jeder `authSignals`-Eintrag unterstützt:

| Feld              | Erforderlich | Typ      | Bedeutung                                                                                                                                                                   |
| ----------------- | ------------ | -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `provider`        | Ja           | `string` | Provider-ID, die in den konfigurierten Authentifizierungsprofilen geprüft werden soll.                                                                                       |
| `providerBaseUrl` | Nein         | `object` | Optionaler Wächter, durch den das Signal nur zählt, wenn der referenzierte konfigurierte Provider eine zulässige Basis-URL verwendet. Verwenden Sie dies, wenn ein Authentifizierungsalias nur für bestimmte APIs gültig ist. |

Jeder `providerBaseUrl`-Wächter unterstützt:

| Feld              | Erforderlich | Typ        | Bedeutung                                                                                                                                             |
| ----------------- | ------------ | ---------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| `provider`        | Ja           | `string`   | Provider-Konfigurations-ID, deren `baseUrl` geprüft werden soll.                                                                              |
| `defaultBaseUrl`  | Nein         | `string`   | Anzunehmende Basis-URL, wenn `baseUrl` in der Provider-Konfiguration fehlt.                                                                  |
| `allowedBaseUrls` | Ja           | `string[]` | Zulässige Basis-URLs für dieses Authentifizierungssignal. Das Signal wird ignoriert, wenn die konfigurierte oder standardmäßige Basis-URL mit keinem dieser normalisierten Werte übereinstimmt. |

## Referenz für Tool-Metadaten

`toolMetadata` verwendet dieselben Strukturen `configSignals` und `authSignals` wie die Metadaten von Generierungs-Providern, jeweils nach Tool-Name indiziert. `contracts.tools` deklariert die Zuständigkeit. `toolMetadata` deklariert kostengünstig ermittelbare Verfügbarkeitsnachweise, sodass OpenClaw vermeiden kann, eine Plugin-Laufzeit nur deshalb zu importieren, damit deren Tool-Factory `null` zurückgibt.

```json
{
  "setup": {
    "providers": [
      {
        "id": "example",
        "envVars": ["EXAMPLE_API_KEY"]
      }
    ]
  },
  "contracts": {
    "tools": ["example_search"]
  },
  "toolMetadata": {
    "example_search": {
      "authSignals": [
        {
          "provider": "example"
        }
      ],
      "configSignals": [
        {
          "rootPath": "plugins.entries.example.config",
          "overlayPath": "search",
          "required": ["apiKey"]
        }
      ]
    }
  }
}
```

`toolMetadata`-Einträge akzeptieren zusätzlich `optional` (kennzeichnet das Tool als nicht erforderlich für die Plugin-Aktivierung) und `replaySafe` (kennzeichnet die Tool-Ausführung als sicher wiederholbar nach einem unvollständigen Modell-Durchlauf), ergänzend zu den oben genannten gemeinsamen Feldern `configSignals`/`authSignals`.

Wenn ein Tool kein `toolMetadata` besitzt, behält OpenClaw das bestehende Verhalten bei und lädt das zugehörige Plugin, wenn der Tool-Vertrag der Richtlinie entspricht. Bei Tools im kritischen Ausführungspfad, deren Factory von Authentifizierung/Konfiguration abhängt, sollten Plugin-Autoren `toolMetadata` deklarieren, statt Core die Runtime importieren zu lassen, um sie abzufragen.

## Referenz zu providerAuthChoices

Jeder `providerAuthChoices`-Eintrag beschreibt eine Onboarding- oder Authentifizierungsoption. OpenClaw liest diesen Eintrag, bevor die Provider-Runtime geladen wird. Listen für die Provider-Einrichtung verwenden diese Manifestoptionen, aus Deskriptoren abgeleitete Einrichtungsoptionen und Metadaten des Installationskatalogs, ohne die Provider-Runtime zu laden.

| Feld                  | Erforderlich | Typ                                                                   | Bedeutung                                                                                                 |
| --------------------- | ------------ | --------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| `provider`            | Ja           | `string`                                                              | Provider-ID, zu der diese Option gehört.                                                                  |
| `method`              | Ja           | `string`                                                              | ID der Authentifizierungsmethode, an die weitergeleitet werden soll.                                      |
| `choiceId`            | Ja           | `string`                                                              | Stabile ID der Authentifizierungsoption, die von Onboarding- und CLI-Abläufen verwendet wird.             |
| `choiceLabel`         | Nein         | `string`                                                              | Für Benutzer sichtbare Bezeichnung. Wenn sie fehlt, greift OpenClaw auf `choiceId` zurück.        |
| `choiceHint`          | Nein         | `string`                                                              | Kurzer Hilfetext für die Auswahl.                                                                         |
| `icon`                | Nein         | HTTPS-URL                                                             | Grafik, die in unterstützten Onboarding-Clients neben dieser Option angezeigt wird.                       |
| `website`             | Nein         | HTTPS-URL                                                             | Produkt-, Anmelde- oder Installationsseite, die von unterstützten Onboarding-Clients angezeigt wird.      |
| `assistantPriority`   | Nein         | `number`                                                              | Niedrigere Werte werden in assistentengesteuerten interaktiven Auswahldialogen weiter vorne einsortiert.  |
| `assistantVisibility` | Nein         | `"visible"` \| `"manual-only"`                                        | Blendet die Option in Assistentenauswahldialogen aus, erlaubt jedoch weiterhin die manuelle CLI-Auswahl.  |
| `deprecatedChoiceIds` | Nein         | `string[]`                                                            | Veraltete Options-IDs, die Benutzer zu dieser Ersatzoption weiterleiten sollen.                            |
| `groupId`             | Nein         | `string`                                                              | Optionale Gruppen-ID zur Gruppierung zusammengehöriger Optionen.                                         |
| `groupLabel`          | Nein         | `string`                                                              | Für Benutzer sichtbare Bezeichnung dieser Gruppe.                                                         |
| `groupHint`           | Nein         | `string`                                                              | Kurzer Hilfetext für die Gruppe.                                                                          |
| `onboardingFeatured`  | Nein         | `boolean`                                                             | Zeigt diese Gruppe in der hervorgehobenen Ebene der interaktiven Onboarding-Auswahl vor dem Eintrag „More...“ an. |
| `optionKey`           | Nein         | `string`                                                              | Interner Optionsschlüssel für einfache Authentifizierungsabläufe mit einem einzelnen Flag.                |
| `cliFlag`             | Nein         | `string`                                                              | Name des CLI-Flags, beispielsweise `--openrouter-api-key`.                                                    |
| `cliOption`           | Nein         | `string`                                                              | Vollständige Form der CLI-Option, beispielsweise `--openrouter-api-key <key>`.                                     |
| `cliDescription`      | Nein         | `string`                                                              | In der CLI-Hilfe verwendete Beschreibung.                                                                 |
| `appGuidedSecret`     | Nein         | `boolean`                                                             | Ein eingefügtes Secret zusammen mit den Provider-Standardeinstellungen genügt für die appgestützte Einrichtung. |
| `appGuidedDiscovery`  | Nein         | `boolean`                                                             | Die entsprechende Runtime-Authentifizierungsmethode ist über `appGuidedSetup` für die schreibgeschützte lokale Erkennung zuständig. |
| `appGuidedAuth`       | Nein         | `"oauth"` \| `"device-code"`                                          | Provider-eigene interaktive Anmeldung, die native Einrichtungsclients generisch darstellen können.       |
| `onboardingScopes`    | Nein         | `Array<"text-inference" \| "image-generation" \| "music-generation">` | In welchen Onboarding-Oberflächen diese Option erscheinen soll. Wenn nicht angegeben, lautet der Standardwert `["text-inference"]`. |

Wenn `appGuidedDiscovery` wahr ist, muss die entsprechende Provider-Authentifizierungsmethode
`appGuidedSetup.detect` und `appGuidedSetup.prepare` bereitstellen. Die Erkennung muss
schreibgeschützt sein: keine Anmeldung, kein Modellabruf, kein Download und kein Schreiben der Konfiguration. Die Vorbereitung prüft
das exakt ausgewählte Modell erneut und gibt einen Konfigurationsvorschlag zurück; OpenClaw testet diesen
Vorschlag isoliert im Live-Betrieb und übernimmt ihn erst nach erfolgreichem Abschluss.

## Referenz zu commandAliases

Verwenden Sie `commandAliases`, wenn ein Plugin einen Runtime-Befehlsnamen besitzt, den Benutzer irrtümlicherweise in `plugins.allow` eintragen oder als Root-CLI-Befehl ausführen könnten. OpenClaw verwendet diese Metadaten für die Diagnose, ohne den Runtime-Code des Plugins zu importieren.

```json
{
  "commandAliases": [
    {
      "name": "dreaming",
      "kind": "runtime-slash",
      "cliCommand": "memory"
    }
  ]
}
```

| Feld         | Erforderlich | Typ               | Bedeutung                                                               |
| ------------ | ------------ | ----------------- | ----------------------------------------------------------------------- |
| `name`       | Ja           | `string`          | Befehlsname, der zu diesem Plugin gehört.                               |
| `kind`       | Nein         | `"runtime-slash"` | Kennzeichnet den Alias als Slash-Befehl im Chat statt als Root-CLI-Befehl. |
| `cliCommand` | Nein         | `string`          | Zugehöriger Root-CLI-Befehl, der für CLI-Vorgänge vorgeschlagen werden soll, sofern vorhanden. |

## Referenz zur Aktivierung

Verwenden Sie `activation`, wenn das Plugin mit geringem Aufwand deklarieren kann, bei welchen Steuerungsebenenereignissen es in einen Aktivierungs-/Ladeplan aufgenommen werden soll.

Dieser Block enthält Planer-Metadaten und ist keine Lebenszyklus-API. Er registriert kein Runtime-Verhalten, ersetzt `register(...)` nicht und garantiert nicht, dass Plugin-Code bereits ausgeführt wurde. Der Aktivierungsplaner verwendet diese Felder, um die infrage kommenden Plugins einzugrenzen, bevor er auf bestehende Manifest-Metadaten zur Zuständigkeit wie `providers`, `channels`, `commandAliases`, `setup.providers`, `contracts.tools` und Hooks zurückgreift.

Bevorzugen Sie die engsten Metadaten, die die Zuständigkeit bereits beschreiben. Verwenden Sie `providers`, `channels`, `commandAliases`, Einrichtungsdeskriptoren oder `contracts`, wenn diese Felder die Beziehung ausdrücken. Verwenden Sie `activation` für zusätzliche Planerhinweise, die nicht durch diese Zuständigkeitsfelder dargestellt werden können. Verwenden Sie `cliBackends` auf oberster Ebene für CLI-Runtime-Aliasse wie `claude-cli`, `my-cli` oder `google-gemini-cli`; `activation.onAgentHarnesses` ist ausschließlich für eingebettete Agent-Harness-IDs vorgesehen, die noch kein Zuständigkeitsfeld besitzen.

Jedes Plugin sollte `activation.onStartup` bewusst festlegen. Setzen Sie den Wert nur dann auf `true`, wenn das Plugin während des Gateway-Starts ausgeführt werden muss. Setzen Sie ihn auf `false`, wenn das Plugin beim Start inaktiv ist und nur durch engere Auslöser geladen werden soll. Wenn `onStartup` fehlt, wird das Plugin nicht mehr implizit beim Start geladen; verwenden Sie explizite Aktivierungsmetadaten für Start-, Kanal-, Konfigurations-, Agent-Harness-, Speicher- oder andere engere Aktivierungsauslöser.

```json
{
  "activation": {
    "onStartup": false,
    "onProviders": ["openai"],
    "onCommands": ["models"],
    "onChannels": ["web"],
    "onRoutes": ["gateway-webhook"],
    "onConfigPaths": ["browser"],
    "onCapabilities": ["provider", "tool"]
  }
}
```

| Feld               | Erforderlich | Typ                                                  | Bedeutung                                                                                                                                                                                   |
| ------------------ | ------------ | ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `onStartup`        | Nein         | `boolean`                                            | Explizite Aktivierung beim Gateway-Start. Jedes Plugin sollte dies festlegen. `true` importiert das Plugin beim Start; `false` hält das Laden beim Start verzögert, sofern kein anderer passender Auslöser das Laden erfordert. |
| `onProviders`      | Nein         | `string[]`                                           | Provider-IDs, die dieses Plugin in Aktivierungs-/Ladepläne aufnehmen sollen.                                                                                                                |
| `onAgentHarnesses` | Nein         | `string[]`                                           | Laufzeit-IDs eingebetteter Agent-Harnesses, die dieses Plugin in Aktivierungs-/Ladepläne aufnehmen sollen. Verwenden Sie `cliBackends` auf oberster Ebene für CLI-Backend-Aliasse.          |
| `onCommands`       | Nein         | `string[]`                                           | Befehls-IDs, die dieses Plugin in Aktivierungs-/Ladepläne aufnehmen sollen.                                                                                                                 |
| `onChannels`       | Nein         | `string[]`                                           | Kanal-IDs, die dieses Plugin in Aktivierungs-/Ladepläne aufnehmen sollen.                                                                                                                   |
| `onRoutes`         | Nein         | `string[]`                                           | Routentypen, die dieses Plugin in Aktivierungs-/Ladepläne aufnehmen sollen.                                                                                                                 |
| `onConfigPaths`    | Nein         | `string[]`                                           | Relativ zum Stammverzeichnis angegebene Konfigurationspfade, die dieses Plugin in Start-/Ladepläne aufnehmen sollen, wenn der Pfad vorhanden und nicht ausdrücklich deaktiviert ist.         |
| `onCapabilities`   | Nein         | `Array<"provider" \| "channel" \| "tool" \| "hook">` | Allgemeine Funktionshinweise für die Aktivierungsplanung der Steuerungsebene. Verwenden Sie nach Möglichkeit spezifischere Felder.                                                         |

Aktuelle aktive Verbraucher:

- Die Gateway-Startplanung verwendet `activation.onStartup` für den expliziten Import beim Start.
- Die befehlsausgelöste CLI-Planung greift auf das veraltete `commandAliases[].cliCommand` oder `commandAliases[].name` zurück.
- Die Startplanung der Agent-Laufzeit verwendet `activation.onAgentHarnesses` für eingebettete Harnesses und `cliBackends[]` auf oberster Ebene für CLI-Laufzeit-Aliasse.
- Die kanalbezogene Einrichtungs-/Kanalplanung greift auf die veraltete Eigentümerschaft gemäß `channels[]` zurück, wenn explizite Metadaten zur Kanalaktivierung fehlen.
- Die Plugin-Planung beim Start verwendet `activation.onConfigPaths` für kanalunabhängige Stammkonfigurationsoberflächen wie den Block `browser` des gebündelten Browser-Plugins.
- Die providerausgelöste Einrichtungs-/Laufzeitplanung greift auf die veraltete Eigentümerschaft gemäß `providers[]` und `cliBackends[]` auf oberster Ebene zurück, wenn explizite Metadaten zur Provider-Aktivierung fehlen.

Planerdiagnosen können explizite Aktivierungshinweise von einem Rückgriff auf die Manifest-Eigentümerschaft unterscheiden. Beispielsweise bedeutet `activation-command-hint`, dass `activation.onCommands` übereinstimmte, während `manifest-command-alias` bedeutet, dass der Planer stattdessen die Eigentümerschaft gemäß `commandAliases` verwendete. Diese Begründungsbezeichnungen dienen Hostdiagnosen und Tests; Plugin-Autoren sollten weiterhin die Metadaten deklarieren, die die Eigentümerschaft am besten beschreiben.

## qaRunners-Referenz

Verwenden Sie `qaRunners`, wenn ein Plugin einen oder mehrere Transport-Runner unterhalb
des gemeinsamen Stamms `openclaw qa` bereitstellt. Halten Sie diese Metadaten schlank und statisch; die Plugin-
Laufzeit ist weiterhin für die eigentliche CLI-Registrierung über eine leichtgewichtige
`runtime-api.ts`-Oberfläche zuständig, die passende `qaRunnerCliRegistrations` exportiert. Ein
optionales `adapterFactory` stellt den Transport gemeinsamen QA-Szenarien bereit, ohne
den Runner des registrierten Befehls zu ändern.

```json
{
  "qaRunners": [
    {
      "commandName": "matrix",
      "description": "Die Docker-gestützte Matrix-Live-QA-Spur gegen einen temporären Homeserver ausführen"
    }
  ]
}
```

| Feld          | Erforderlich | Typ      | Bedeutung                                                          |
| ------------- | ------------ | -------- | ------------------------------------------------------------------ |
| `commandName` | Ja           | `string` | Unterbefehl unterhalb von `openclaw qa`, beispielsweise `matrix`. |
| `description` | Nein         | `string` | Ersatz-Hilfetext, wenn der gemeinsame Host einen Platzhalterbefehl benötigt. |

Die ID `adapterFactory` muss mit `commandName` übereinstimmen. Exportieren Sie keine Registrierungen
für Befehle, die nicht im Manifest enthalten sind.

## setup-Referenz

Verwenden Sie `setup`, wenn Einrichtungs- und Onboarding-Oberflächen schlanke Plugin-eigene Metadaten benötigen, bevor die Laufzeit geladen wird.

```json
{
  "setup": {
    "providers": [
      {
        "id": "openai",
        "authMethods": ["api-key"],
        "envVars": ["OPENAI_API_KEY"],
        "authEvidence": [
          {
            "type": "local-file-with-env",
            "fileEnvVar": "OPENAI_CREDENTIALS_FILE",
            "requiresAllEnv": ["OPENAI_PROJECT"],
            "credentialMarker": "openai-local-credentials",
            "source": "lokale OpenAI-Anmeldedaten"
          }
        ]
      }
    ],
    "cliBackends": ["openai-cli"],
    "configMigrations": ["legacy-openai-auth"],
    "requiresRuntime": false
  }
}
```

`cliBackends` auf oberster Ebene bleibt gültig und beschreibt weiterhin CLI-Inferenz-Backends. `setup.cliBackends` ist die einrichtungsspezifische Deskriptoroberfläche für Steuerungsebenen-/Einrichtungsabläufe, die ausschließlich auf Metadaten basieren sollten.

Sofern vorhanden, sind `setup.providers` und `setup.cliBackends` die bevorzugte Deskriptor-zuerst-Nachschlageoberfläche für die Einrichtungserkennung. Wenn der Deskriptor lediglich das infrage kommende Plugin eingrenzt und die Einrichtung weiterhin umfangreichere Laufzeit-Hooks zur Einrichtungszeit benötigt, legen Sie `requiresRuntime: true` fest und behalten Sie `setup-api` als Ausweich-Ausführungspfad bei.

OpenClaw bezieht `setup.providers[].envVars` in generische Nachschlagevorgänge für Provider-Authentifizierung und Umgebungsvariablen ein. Hinterlegen Sie dort Umgebungsmetadaten für Einrichtung und Status.

Verwenden Sie `providerUsageAuthEnvVars`, wenn Anmeldedaten auf Abrechnungs- oder Organisationsebene `resolveUsageAuth` aktivieren müssen, ohne zu Inferenz-Anmeldedaten zu werden. Diese Namen werden in die Blockierung von Workspace-Dotenv-Werten, die Bereinigung von ACP-Unterprozessen, die Sandbox-Filterung von Geheimnissen und die allgemeine Bereinigung von Geheimnissen aufgenommen. Die Provider-Laufzeit liest und klassifiziert den Wert weiterhin innerhalb von `resolveUsageAuth`.

OpenClaw kann außerdem einfache Einrichtungsoptionen aus `setup.providers[].authMethods` ableiten, wenn kein Einrichtungseintrag verfügbar ist oder wenn `setup.requiresRuntime: false` angibt, dass keine Einrichtungslaufzeit erforderlich ist. Explizite `providerAuthChoices`-Einträge werden weiterhin für benutzerdefinierte Bezeichnungen, CLI-Flags, den Onboarding-Umfang und Assistentenmetadaten bevorzugt.

Legen Sie `requiresRuntime: false` nur fest, wenn diese Deskriptoren für die Einrichtungsoberfläche ausreichen. OpenClaw behandelt ein explizites `false` als ausschließlich deskriptorbasierten Vertrag und führt `setup-api` oder `openclaw.setupEntry` nicht für die Einrichtungssuche aus. Wenn ein ausschließlich deskriptorbasiertes Plugin dennoch einen dieser Einrichtungslaufzeit-Einträge bereitstellt, meldet OpenClaw eine zusätzliche Diagnose und ignoriert ihn weiterhin. Wird `requiresRuntime` weggelassen, bleibt das veraltete Rückgriffverhalten erhalten, damit vorhandene Plugins, die Deskriptoren ohne das Flag hinzugefügt haben, nicht beeinträchtigt werden.

Da die Einrichtungssuche Plugin-eigenen `setup-api`-Code ausführen kann, müssen normalisierte `setup.providers[].id`- und `setup.cliBackends[]`-Werte über alle erkannten Plugins hinweg eindeutig bleiben. Bei mehrdeutiger Eigentümerschaft wird der Vorgang sicher abgebrochen, anstatt anhand der Erkennungsreihenfolge einen Gewinner auszuwählen.

Wenn die Einrichtungslaufzeit ausgeführt wird, melden die Diagnosen der Einrichtungsregistrierung Deskriptorabweichungen, falls `setup-api` einen Provider oder ein CLI-Backend registriert, den beziehungsweise das die Manifest-Deskriptoren nicht deklarieren, oder falls für einen Deskriptor keine passende Laufzeitregistrierung vorhanden ist. Diese Diagnosen sind ergänzend und weisen veraltete Plugins nicht zurück.

### setup.providers-Referenz

| Feld           | Erforderlich | Typ        | Bedeutung                                                                                          |
| -------------- | ------------ | ---------- | -------------------------------------------------------------------------------------------------- |
| `id`           | Ja           | `string`   | Provider-ID, die während der Einrichtung oder des Onboardings bereitgestellt wird. Halten Sie normalisierte IDs global eindeutig. |
| `authMethods`  | Nein         | `string[]` | IDs der Einrichtungs-/Authentifizierungsmethoden, die dieser Provider ohne Laden der vollständigen Laufzeit unterstützt.           |
| `envVars`      | Nein         | `string[]` | Umgebungsvariablen, die generische Einrichtungs-/Statusoberflächen vor dem Laden der Plugin-Laufzeit prüfen können.                 |
| `authEvidence` | Nein         | `object[]` | Schlanke Prüfungen lokaler Authentifizierungsnachweise für Provider, die sich über nicht geheime Marker authentifizieren können.   |

`authEvidence` ist für Provider-eigene Marker lokaler Anmeldedaten vorgesehen, die ohne Laden von Laufzeitcode überprüft werden können. Diese Prüfungen müssen schlank und lokal bleiben: keine Netzwerkaufrufe, keine Zugriffe auf Schlüsselbund oder Geheimnisverwaltung, keine Shell-Befehle und keine Abfragen der Provider-API.

Unterstützte Nachweiseinträge:

| Feld               | Erforderlich | Typ        | Bedeutung                                                                                                      |
| ------------------ | ------------ | ---------- | -------------------------------------------------------------------------------------------------------------- |
| `type`             | Ja           | `string`   | Derzeit `local-file-with-env`.                                                                                   |
| `fileEnvVar`       | Nein         | `string`   | Umgebungsvariable, die einen expliziten Pfad zu einer Anmeldedatendatei enthält.                               |
| `fallbackPaths`    | Nein         | `string[]` | Pfade lokaler Anmeldedatendateien, die geprüft werden, wenn `fileEnvVar` fehlt oder leer ist. Unterstützt `${HOME}` und `${APPDATA}`. |
| `requiresAnyEnv`   | Nein         | `string[]` | Mindestens eine aufgeführte Umgebungsvariable muss nicht leer sein, damit der Nachweis gültig ist.             |
| `requiresAllEnv`   | Nein         | `string[]` | Jede aufgeführte Umgebungsvariable muss nicht leer sein, damit der Nachweis gültig ist.                        |
| `credentialMarker` | Ja           | `string`   | Nicht geheimer Marker, der zurückgegeben wird, wenn der Nachweis vorhanden ist.                               |
| `source`           | Nein         | `string`   | Benutzerseitig sichtbare Quellenbezeichnung für Authentifizierungs-/Statusausgaben.                            |

### setup-Felder

| Feld               | Erforderlich | Typ        | Bedeutung                                                                                           |
| ------------------ | ------------ | ---------- | --------------------------------------------------------------------------------------------------- |
| `providers`        | Nein         | `object[]` | Während der Einrichtung und des Onboardings bereitgestellte Beschreibungen der Provider-Einrichtung. |
| `cliBackends`      | Nein         | `string[]` | Bei der Einrichtung verwendete Backend-IDs für die beschreibungsbasierte Einrichtungssuche. Normalisierte IDs müssen global eindeutig sein. |
| `configMigrations` | Nein         | `string[]` | IDs für Konfigurationsmigrationen, die zur Einrichtungsoberfläche dieses Plugins gehören.           |
| `requiresRuntime`  | Nein         | `boolean`  | Ob die Einrichtung nach der Beschreibungssuche weiterhin die Ausführung von `setup-api` benötigt. |

## Referenz zu uiHints

`uiHints` ist eine Zuordnung von Namen der Konfigurationsfelder zu kleinen Darstellungshinweisen. Schlüssel können Punkte für verschachtelte Konfigurationsfelder verwenden, aber kein Pfadsegment darf `__proto__`, `constructor` oder `prototype` lauten; die Einrichtung weist diese Namen zurück.

```json
{
  "uiHints": {
    "apiKey": {
      "label": "API-Schlüssel",
      "help": "Wird für OpenRouter-Anfragen verwendet",
      "placeholder": "sk-or-v1-...",
      "sensitive": true
    }
  }
}
```

Jeder Feldhinweis kann Folgendes enthalten:

| Feld           | Typ              | Bedeutung                                                                                                         |
| -------------- | ---------------- | ----------------------------------------------------------------------------------------------------------------- |
| `label`        | `string`         | Für Benutzer sichtbare Feldbezeichnung.                                                                           |
| `help`         | `string`         | Kurzer Hilfetext.                                                                                                 |
| `tags`         | `string[]`       | Optionale UI-Tags.                                                                                                |
| `advanced`     | `boolean`        | Kennzeichnet das Feld als erweitert.                                                                              |
| `sensitive`    | `boolean`        | Kennzeichnet das Feld als geheim oder vertraulich.                                                                |
| `placeholder`  | `string`         | Platzhaltertext für Formulareingaben.                                                                             |
| `presentation` | `"phone-number"` | Nur zur Anzeige bestimmte lokalisierte Telefonnummernformatierung für analysierbare internationale Werte (`+...`); Rohwerte bleiben unverändert. |

## Referenz zu contracts

Verwenden Sie `contracts` ausschließlich für statische Metadaten zur Zuständigkeit für Fähigkeiten, die OpenClaw lesen kann, ohne die Plugin-Laufzeit zu importieren.

```json
{
  "contracts": {
    "agentToolResultMiddleware": ["openclaw", "codex"],
    "trustedToolPolicies": ["workflow-budget"],
    "externalAuthProviders": ["acme-ai"],
    "embeddingProviders": ["openai-compatible"],
    "speechProviders": ["openai"],
    "realtimeTranscriptionProviders": ["openai"],
    "realtimeVoiceProviders": ["openai"],
    "memoryEmbeddingProviders": ["local"],
    "mediaUnderstandingProviders": ["openai"],
    "imageGenerationProviders": ["openai"],
    "videoGenerationProviders": ["qwen"],
    "musicGenerationProviders": ["stability-audio"],
    "documentExtractors": ["example-docs"],
    "webContentExtractors": ["firecrawl"],
    "webFetchProviders": ["firecrawl"],
    "webSearchProviders": ["gemini"],
    "workerProviders": ["example-worker"],
    "usageProviders": ["acme-ai"],
    "migrationProviders": ["hermes"],
    "gatewayMethodDispatch": ["authenticated-request"],
    "tools": ["firecrawl_search", "firecrawl_scrape"]
  }
}
```

Jede Liste ist optional:

| Feld                             | Typ        | Bedeutung                                                                                                                            |
| -------------------------------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `embeddedExtensionFactories`     | `string[]` | Factory-IDs für Codex-App-Server-Erweiterungen, derzeit `codex-app-server`.                                                           |
| `agentToolResultMiddleware`      | `string[]` | Laufzeit-IDs, für die dieses Plugin Middleware für Tool-Ergebnisse registrieren darf.                                                |
| `trustedToolPolicies`            | `string[]` | Plugin-lokale IDs für vertrauenswürdige Richtlinien vor der Tool-Ausführung, die ein installiertes Plugin registrieren darf. Mitgelieferte Plugins dürfen Richtlinien ohne dieses Feld registrieren. |
| `externalAuthProviders`          | `string[]` | Provider-IDs, deren Hook für externe Authentifizierungsprofile diesem Plugin gehört.                                                 |
| `embeddingProviders`             | `string[]` | IDs allgemeiner Embedding-Provider, die diesem Plugin für die wiederverwendbare Erzeugung von Vektor-Embeddings einschließlich Memory gehören. |
| `speechProviders`                | `string[]` | IDs von Sprach-Providern, die diesem Plugin gehören.                                                                                 |
| `realtimeTranscriptionProviders` | `string[]` | IDs von Providern für Echtzeittranskription, die diesem Plugin gehören.                                                              |
| `realtimeVoiceProviders`         | `string[]` | IDs von Providern für Echtzeitstimme, die diesem Plugin gehören.                                                                    |
| `memoryEmbeddingProviders`       | `string[]` | Veraltete IDs Memory-spezifischer Embedding-Provider, die diesem Plugin gehören.                                                     |
| `mediaUnderstandingProviders`    | `string[]` | IDs von Providern für Medienverständnis, die diesem Plugin gehören.                                                                 |
| `transcriptSourceProviders`      | `string[]` | IDs von Providern für Transkriptquellen, die diesem Plugin gehören.                                                                 |
| `documentExtractors`             | `string[]` | IDs von Providern für Dokumentextraktoren (beispielsweise PDF), die diesem Plugin gehören.                                          |
| `imageGenerationProviders`       | `string[]` | IDs von Providern für Bilderzeugung, die diesem Plugin gehören.                                                                     |
| `videoGenerationProviders`       | `string[]` | IDs von Providern für Videoerzeugung, die diesem Plugin gehören.                                                                    |
| `musicGenerationProviders`       | `string[]` | IDs von Providern für Musikerzeugung, die diesem Plugin gehören.                                                                    |
| `webContentExtractors`           | `string[]` | IDs von Providern für die Inhaltsextraktion aus Webseiten, die diesem Plugin gehören.                                               |
| `webFetchProviders`              | `string[]` | IDs von Web-Abruf-Providern, die diesem Plugin gehören.                                                                              |
| `webSearchProviders`             | `string[]` | IDs von Websuch-Providern, die diesem Plugin gehören.                                                                                |
| `workerProviders`                | `string[]` | IDs von Cloud-Worker-Providern, die diesem Plugin für die Bereitstellung und den profilgestützten Lease-Lebenszyklus gehören.         |
| `usageProviders`                 | `string[]` | Provider-IDs, deren Hooks für Nutzungsauthentifizierung und Nutzungsmomentaufnahmen diesem Plugin gehören.                           |
| `migrationProviders`             | `string[]` | IDs von Import-Providern, die diesem Plugin für `openclaw migrate` gehören.                                                          |
| `gatewayMethodDispatch`          | `string[]` | Reservierte Berechtigung für authentifizierte Plugin-HTTP-Routen, die Gateway-Methoden prozessintern weiterleiten.                   |
| `tools`                          | `string[]` | Namen von Agent-Tools, die diesem Plugin gehören.                                                                                    |

`contracts.embeddedExtensionFactories` bleibt für mitgelieferte Erweiterungs-Factorys erhalten, die ausschließlich für den Codex-App-Server bestimmt sind. Mitgelieferte Transformationen von Tool-Ergebnissen sollten stattdessen `contracts.agentToolResultMiddleware` deklarieren und sich mit `api.registerAgentToolResultMiddleware(...)` registrieren. Installierte Plugins dürfen dieselbe Middleware-Schnittstelle nur verwenden, wenn sie ausdrücklich aktiviert wurde, und nur für Laufzeiten, die sie in `contracts.agentToolResultMiddleware` deklarieren.

Installierte Plugins, die die vom Host als vertrauenswürdig eingestufte Richtlinienebene vor der Tool-Ausführung benötigen, müssen jede registrierte lokale ID in `contracts.trustedToolPolicies` deklarieren und ausdrücklich aktiviert werden. Mitgelieferte Plugins behalten den bestehenden Pfad für vertrauenswürdige Richtlinien bei, installierte Plugins mit nicht deklarierten Richtlinien-IDs werden jedoch vor der Registrierung zurückgewiesen. Richtlinien-IDs sind auf das registrierende Plugin beschränkt, sodass zwei Plugins jeweils `workflow-budget` deklarieren und registrieren dürfen; ein einzelnes Plugin darf dieselbe lokale ID nicht zweimal registrieren.

Laufzeitregistrierungen von `api.registerTool(...)` müssen mit `contracts.tools` übereinstimmen. Die Tool-Ermittlung verwendet diese Liste, um nur die Plugin-Laufzeiten zu laden, denen die angeforderten Tools gehören können.

Provider-Plugins, die `resolveExternalAuthProfiles` implementieren, sollten `contracts.externalAuthProviders` deklarieren; nicht deklarierte Hooks für externe Authentifizierung werden ignoriert.

Provider-Plugins, die sowohl `resolveUsageAuth` als auch `fetchUsageSnapshot` implementieren, sollten jede automatisch ermittelte Provider-ID in `contracts.usageProviders` deklarieren. Die Nutzungsermittlung liest diesen Vertrag vor dem Laden des Laufzeitcodes und überprüft anschließend beide Hooks, nachdem nur die deklarierten zuständigen Plugins geladen wurden.

Allgemeine Embedding-Provider sollten `contracts.embeddingProviders` für jeden mit `api.registerEmbeddingProvider(...)` registrierten Adapter deklarieren. Verwenden Sie den allgemeinen Vertrag für die wiederverwendbare Vektorerzeugung, einschließlich Providern, die von der Memory-Suche verwendet werden. `contracts.memoryEmbeddingProviders` ist eine veraltete Memory-spezifische Kompatibilität und bleibt nur bestehen, solange vorhandene Provider zur generischen Schnittstelle für Embedding-Provider migrieren.

Worker-Provider müssen jede `api.registerWorkerProvider(...)`-ID in `contracts.workerProviders` deklarieren. Core speichert die dauerhafte Absicht, bevor `provision` aufgerufen wird; Provider validieren ihre Einstellungen vor der externen Zuweisung, und wiederholte Aufrufe mit derselben Vorgangs-ID müssen denselben Lease übernehmen. Core speichert außerdem diese Momentaufnahme der validierten Einstellungen und übergibt sie zusammen mit `leaseId` an `inspect({ leaseId, profile })` und `destroy({ leaseId, profile })`, auch nachdem das benannte Profil geändert oder entfernt wurde. Die Zerstörung ist idempotent, die Inspektion gibt die geschlossene Statusvereinigung aus `active` / `destroyed` / `unknown` zurück, und auf privates SSH-Schlüsselmaterial wird ausschließlich über `SecretRef` verwiesen. Bereitgestellte SSH-Endpunkte müssen außerdem einen öffentlichen `hostKey` aus einer vertrauenswürdigen Bereitstellungsausgabe exakt als `algorithm base64` enthalten, ohne Hostnamen oder Kommentar, damit Core den Host vor dem Verbindungsaufbau anheften kann. Provider, die dynamische Identitätsreferenzen erzeugen, können das maßgebliche `resolveSshIdentity({ leaseId, profile, keyRef })` implementieren; Provider ohne diese Implementierung verwenden den generischen Secret-Resolver von Core. Ein maßgebliches `unknown` verwaist einen aktiven lokalen Datensatz; nach einer gespeicherten Zerstörungsanforderung bestätigt es den Abbau.

`contracts.gatewayMethodDispatch` akzeptiert derzeit `"authenticated-request"`. Es handelt sich um eine API-Hygiene-Schranke für native Plugin-HTTP-Routen, die absichtlich Gateway-Control-Plane-Methoden prozessintern aufrufen, nicht um eine Sandbox gegen bösartige native Plugins. Verwenden Sie sie nur für streng geprüfte gebündelte bzw. Operator-Oberflächen, die bereits eine Gateway-HTTP-Authentifizierung erfordern. Eine berechtigte Route bleibt bei geschlossener Gateway-Zulassung für Root-Arbeit nur erreichbar, wenn sie zusätzlich `auth: "gateway"` und das routenspezifische `gatewayRuntimeScopeSurface: "trusted-operator"` deklariert; gewöhnliche benachbarte Routen desselben Plugins bleiben hinter der Zulassungsgrenze. Dadurch bleiben der Sperrstatus und das Fortsetzen erreichbar, ohne dem gesamten Plugin eine Umgehung der Zulassung zu gewähren. Halten Sie das Parsen und die Antwortaufbereitung außerhalb des Dispatches begrenzt; wesentliche oder verändernde Arbeit muss über den Gateway-Methoden-Dispatch erfolgen, der die Zulassungs- und Bereichsdurchsetzung verantwortet.

## Referenz zu configContracts

Verwenden Sie `configContracts` für manifestgesteuertes Konfigurationsverhalten, das generische Core-Hilfsfunktionen benötigen, ohne die Plugin-Laufzeit zu importieren: Erkennung gefährlicher Flags, SecretRef-Migrationsziele und Eingrenzung veralteter Konfigurationspfade.

```json
{
  "configContracts": {
    "compatibilityMigrationPaths": ["legacyProvider"],
    "compatibilityRuntimePaths": ["legacyProvider.webhook"],
    "dangerousFlags": [
      {
        "path": "accounts.*.allowUnverifiedSenders",
        "equals": true
      }
    ],
    "secretInputs": {
      "bundledDefaultEnabled": false,
      "paths": [
        {
          "path": "routes.*.secret",
          "expected": "string",
          "ownerKind": "route"
        }
      ]
    }
  }
}
```

| Feld                          | Erforderlich | Typ        | Bedeutung                                                                                                                                                                                                                             |
| ----------------------------- | ------------ | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `compatibilityMigrationPaths` | Nein         | `string[]` | Auf den Stamm bezogene Konfigurationspfade, die darauf hinweisen, dass die Kompatibilitätsmigrationen dieses Plugins zur Einrichtungszeit anwendbar sein könnten. Dadurch können generische Laufzeit-Konfigurationslesevorgänge sämtliche Einrichtungsoberflächen des Plugins überspringen, wenn die Konfiguration nie auf das Plugin verweist. |
| `compatibilityRuntimePaths`   | Nein         | `string[]` | Auf den Stamm bezogene Kompatibilitätspfade, die dieses Plugin während der Laufzeit bedienen kann, bevor der Plugin-Code vollständig aktiviert wird. Verwenden Sie dies für veraltete Oberflächen, die Mengen gebündelter Kandidaten eingrenzen sollen, ohne jede kompatible Plugin-Laufzeit zu importieren. |
| `dangerousFlags`              | Nein         | `object[]` | Konfigurationsliterale, die `openclaw doctor` bei Aktivierung als unsicher oder gefährlich kennzeichnen soll. Siehe unten.                                                                                                              |
| `secretInputs`                | Nein         | `object`   | Konfigurationspfade unter `plugins.entries.<id>.config` für SecretRef-Migration, Audit, Materialisierung beim Start und optionale Laufzeitisolierung des Eigentümers. Siehe unten.                                                                  |

Jeder `dangerousFlags`-Eintrag unterstützt:

| Feld     | Erforderlich | Typ                                   | Bedeutung                                                                                                                |
| -------- | ------------ | ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `path`   | Ja           | `string`                              | Durch Punkte getrennter Konfigurationspfad relativ zu `plugins.entries.<id>.config`. Unterstützt `*`-Platzhalter für Karten-/Array-Segmente. |
| `equals` | Ja           | `string \| number \| boolean \| null` | Exaktes Literal, das diesen Konfigurationswert als gefährlich kennzeichnet.                                              |

`secretInputs` unterstützt:

| Feld                    | Erforderlich | Typ        | Bedeutung                                                                                                                                                                                                                                                                                                                                                 |
| ----------------------- | ------------ | ---------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `bundledDefaultEnabled` | Nein         | `boolean`  | Überschreibt die standardmäßige Aktivierung gebündelter Plugins bei der Entscheidung, ob diese SecretRef-Oberfläche aktiv ist. Verwenden Sie dies, wenn das Plugin gebündelt ist, die Oberfläche jedoch inaktiv bleiben soll, bis sie ausdrücklich in der Konfiguration aktiviert wird.                                                                        |
| `paths`                 | Ja           | `object[]` | Geheimnisartige Konfigurationspfade, jeweils mit `path` (durch Punkte getrennt, relativ zu `plugins.entries.<id>.config`, unterstützt `*`-Platzhalter), optionalem `expected` (derzeit nur `"string"`) und optionalem `ownerKind` (derzeit nur `"route"`). Ein deklarierter Eigentümer isoliert bei fehlgeschlagener Auflösung nur den exakt übereinstimmenden Pfad; seine Eigentümer-ID ist der vollständige Konfigurationspfad. |

## Referenz zu mediaUnderstandingProviderMetadata

Verwenden Sie `mediaUnderstandingProviderMetadata`, wenn ein Provider für Medienverständnis Standardmodelle, eine Priorität für den automatischen Authentifizierungs-Fallback oder native Dokumentunterstützung besitzt, die generische Core-Hilfsfunktionen vor dem Laden der Laufzeit benötigen. Schlüssel müssen außerdem in `contracts.mediaUnderstandingProviders` deklariert werden.

```json
{
  "contracts": {
    "mediaUnderstandingProviders": ["example"]
  },
  "mediaUnderstandingProviderMetadata": {
    "example": {
      "capabilities": ["image", "audio"],
      "defaultModels": {
        "image": "example-vision-latest",
        "audio": "example-transcribe-latest"
      },
      "autoPriority": {
        "image": 40
      },
      "nativeDocumentInputs": ["pdf"],
      "documentModels": {
        "pdf": {
          "textExtraction": "example-doc-text-latest",
          "image": "example-doc-vision-latest"
        }
      }
    }
  }
}
```

Jeder Provider-Eintrag kann Folgendes enthalten:

| Feld                   | Typ                                                              | Bedeutung                                                                                                              |
| ---------------------- | ---------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `capabilities`         | `("image" \| "audio" \| "video")[]`                              | Von diesem Provider bereitgestellte Medienfunktionen.                                                                  |
| `defaultModels`        | `Record<string, string>`                                         | Zuordnungen von Funktionen zu Standardmodellen, die verwendet werden, wenn die Konfiguration kein Modell angibt.       |
| `autoPriority`         | `Record<string, number>`                                         | Niedrigere Zahlen werden beim automatischen, zugangsdatenbasierten Provider-Fallback früher einsortiert.                |
| `nativeDocumentInputs` | `"pdf"[]`                                                        | Vom Provider unterstützte native Dokumenteingaben.                                                                     |
| `documentModels`       | `{ pdf?: { textExtraction?: string; image?: string \| false } }` | Modellspezifische Überschreibungen je Dokumenttyp. Setzen Sie `image: false`, um die bildbasierte Extraktion für diesen Dokumenttyp zu deaktivieren. |

## Referenz zu channelConfigs

Verwenden Sie `channelConfigs`, wenn ein Kanal-Plugin kostengünstig verfügbare Konfigurationsmetadaten benötigt, bevor die Laufzeit geladen wird. Die schreibgeschützte Ermittlung von Kanaleinrichtung und -status kann diese Metadaten direkt für konfigurierte externe Kanäle verwenden, wenn kein Einrichtungseintrag verfügbar ist oder wenn `setup.requiresRuntime: false` erklärt, dass keine Einrichtungslaufzeit erforderlich ist.

`channelConfigs` sind Plugin-Manifest-Metadaten und kein neuer Konfigurationsabschnitt auf oberster Ebene für Benutzer. Benutzer konfigurieren Kanalinstanzen weiterhin unter `channels.<channel-id>`. OpenClaw liest die Manifest-Metadaten, um zu bestimmen, welches Plugin den konfigurierten Kanal besitzt, bevor der Plugin-Laufzeitcode ausgeführt wird.

Für ein Kanal-Plugin beschreiben `configSchema` und `channelConfigs` unterschiedliche Pfade:

- `configSchema` validiert `plugins.entries.<plugin-id>.config`
- `channelConfigs.<channel-id>.schema` validiert `channels.<channel-id>`

Nicht gebündelte Plugins, die `channels[]` deklarieren, sollten außerdem passende `channelConfigs`-Einträge deklarieren. Ohne sie kann OpenClaw das Plugin weiterhin laden, aber Konfigurationsschema-, Einrichtungs- und Control-UI-Oberflächen für den Kaltpfad können die Form der kanaleigenen Optionen oder ausschließlich zur Anzeige bestimmten UI-Hinweise erst erkennen, nachdem die Plugin-Laufzeit ausgeführt wurde.

`channelConfigs.<channel-id>.commands.nativeCommandsAutoEnabled` und `nativeSkillsAutoEnabled` können statische `auto`-Standardwerte für Prüfungen der Befehlskonfiguration deklarieren, die vor dem Laden der Kanallaufzeit ausgeführt werden. Gebündelte Kanäle können dieselben Standardwerte außerdem über `package.json#openclaw.channel.commands` zusammen mit ihren übrigen paketeigenen Kanalkatalog-Metadaten veröffentlichen.

```json
{
  "channelConfigs": {
    "matrix": {
      "schema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "homeserverUrl": { "type": "string" }
        }
      },
      "uiHints": {
        "homeserverUrl": {
          "label": "Homeserver-URL",
          "placeholder": "https://matrix.example.com"
        }
      },
      "label": "Matrix",
      "description": "Matrix-Homeserver-Verbindung",
      "commands": {
        "nativeCommandsAutoEnabled": true,
        "nativeSkillsAutoEnabled": true
      },
      "preferOver": ["matrix-legacy"]
    }
  }
}
```

Jeder Kanaleintrag kann Folgendes enthalten:

| Feld          | Typ                      | Bedeutung                                                                                                                |
| ------------- | ------------------------ | ------------------------------------------------------------------------------------------------------------------------ |
| `schema`      | `object`                 | JSON-Schema für `channels.<id>`. Für jeden deklarierten Kanalkonfigurationseintrag erforderlich.                      |
| `uiHints`     | `Record<string, object>` | Optionale Bezeichnungen, Platzhalter, Vertraulichkeitsangaben und ausschließlich zur Anzeige bestimmte Darstellungshinweise für diesen Kanalkonfigurationsabschnitt. |
| `label`       | `string`                 | Kanalbezeichnung, die in Auswahl- und Inspektionsoberflächen zusammengeführt wird, wenn Laufzeitmetadaten noch nicht verfügbar sind. |
| `description` | `string`                 | Kurze Kanalbeschreibung für Inspektions- und Katalogoberflächen.                                                         |
| `commands`    | `object`                 | Statische automatische Standardwerte für native Befehle und native Skills bei Konfigurationsprüfungen vor der Laufzeit. |
| `preferOver`  | `string[]`               | Veraltete oder niedriger priorisierte Plugin-IDs, die dieser Kanal in Auswahloberflächen übertreffen soll.               |

### Ersetzen eines anderen Kanal-Plugins

Verwenden Sie `preferOver`, wenn Ihr Plugin der bevorzugte Eigentümer einer Kanal-ID ist, die auch von einem anderen Plugin bereitgestellt werden kann. Häufige Fälle sind eine umbenannte Plugin-ID, ein eigenständiges Plugin, das ein gebündeltes Plugin ersetzt, oder ein gepflegter Fork, der zur Konfigurationskompatibilität dieselbe Kanal-ID beibehält.

```json
{
  "id": "acme-chat",
  "channels": ["chat"],
  "channelConfigs": {
    "chat": {
      "schema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "webhookUrl": { "type": "string" }
        }
      },
      "preferOver": ["chat"]
    }
  }
}
```

Wenn `channels.chat` konfiguriert ist, berücksichtigt OpenClaw sowohl die Kanal-ID als auch die bevorzugte Plugin-ID. Wenn das Plugin mit niedrigerer Priorität nur ausgewählt wurde, weil es gebündelt oder standardmäßig aktiviert ist, deaktiviert OpenClaw es in der effektiven Laufzeitkonfiguration, sodass ein Plugin für den Kanal und dessen Tools zuständig ist. Eine explizite Benutzerauswahl hat weiterhin Vorrang: Wenn beide Plugins explizit aktiviert werden (über `plugins.allow` oder eine maßgebliche `plugins.entries`-Konfiguration), behält OpenClaw diese Auswahl bei und meldet Diagnosen zu doppelten Kanälen oder Tools, anstatt die angeforderte Plugin-Gruppe stillschweigend zu ändern.

Beschränken Sie `preferOver` auf Plugin-IDs, die tatsächlich denselben Kanal bereitstellen können. Es ist kein allgemeines Prioritätsfeld und benennt keine Benutzerkonfigurationsschlüssel um.

## Referenz zu modelSupport

Verwenden Sie `modelSupport`, wenn OpenClaw Ihr Provider-Plugin anhand verkürzter Modell-IDs wie `gpt-5.6-sol` oder `claude-sonnet-4.6` ableiten soll, bevor die Plugin-Laufzeit geladen wird.

```json
{
  "modelSupport": {
    "modelPrefixes": ["gpt-", "o1", "o3", "o4"],
    "modelPatterns": ["^computer-use-preview"]
  }
}
```

OpenClaw wendet folgende Rangfolge an:

- Explizite `provider/model`-Referenzen verwenden die Manifest-Metadaten der zugehörigen `providers`
- `modelPatterns` haben Vorrang vor `modelPrefixes`
- Wenn sowohl ein nicht gebündeltes als auch ein gebündeltes Plugin übereinstimmen, hat das nicht gebündelte Plugin Vorrang
- Verbleibende Mehrdeutigkeiten werden ignoriert, bis eine Provider-Angabe durch den Benutzer oder die Konfiguration erfolgt

Felder:

| Feld            | Typ        | Bedeutung                                                                       |
| --------------- | ---------- | ------------------------------------------------------------------------------- |
| `modelPrefixes` | `string[]` | Präfixe, die mit `startsWith` gegen verkürzte Modell-IDs abgeglichen werden.     |
| `modelPatterns` | `string[]` | Regex-Quellen, die nach dem Entfernen des Profilsuffixes gegen verkürzte Modell-IDs abgeglichen werden. |

`modelPatterns`-Einträge werden über `compileSafeRegex` kompiliert, wobei Muster mit verschachtelten Wiederholungen (zum Beispiel `(a+)+$`) abgelehnt werden. Muster, welche die Sicherheitsprüfung nicht bestehen, werden ebenso wie syntaktisch ungültige reguläre Ausdrücke stillschweigend übersprungen. Halten Sie Muster einfach und vermeiden Sie verschachtelte Quantifizierer.

## Referenz zu modelCatalog

Verwenden Sie `modelCatalog`, wenn OpenClaw die Modellmetadaten des Providers kennen soll, bevor die Plugin-Laufzeit geladen wird. Dies ist die vom Manifest verwaltete Quelle für feste Katalogzeilen, Provider-Aliasse, Unterdrückungsregeln und den Ermittlungsmodus. Die Aktualisierung zur Laufzeit verbleibt im Laufzeitcode des Providers, das Manifest teilt dem Kern jedoch mit, wann die Laufzeit erforderlich ist.

```json
{
  "providers": ["openai"],
  "modelCatalog": {
    "providers": {
      "openai": {
        "baseUrl": "https://api.openai.com/v1",
        "api": "openai-responses",
        "models": [
          {
            "id": "gpt-5.4",
            "name": "GPT-5.4",
            "input": ["text", "image"],
            "reasoning": true,
            "contextWindow": 256000,
            "maxTokens": 128000,
            "cost": {
              "input": 1.25,
              "output": 10,
              "cacheRead": 0.125
            },
            "status": "available",
            "tags": ["default"]
          }
        ]
      }
    },
    "aliases": {
      "azure-openai-responses": {
        "provider": "openai",
        "api": "azure-openai-responses"
      }
    },
    "suppressions": [
      {
        "provider": "azure-openai-responses",
        "model": "gpt-5.3-codex-spark",
        "reason": "not available on Azure OpenAI Responses"
      }
    ],
    "discovery": {
      "openai": "static"
    }
  }
}
```

Felder der obersten Ebene:

| Feld             | Typ                                                      | Bedeutung                                                                                                   |
| ---------------- | -------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| `providers`      | `Record<string, object>`                                 | Katalogzeilen für Provider-IDs, die diesem Plugin gehören. Die Schlüssel sollten auch im übergeordneten `providers` erscheinen. |
| `aliases`        | `Record<string, object>`                                 | Provider-Aliasse, die für die Katalog- oder Unterdrückungsplanung in einen zugehörigen Provider aufgelöst werden sollen. |
| `suppressions`   | `object[]`                                               | Modellzeilen aus einer anderen Quelle, die dieses Plugin aus einem providerspezifischen Grund unterdrückt.  |
| `discovery`      | `Record<string, "static" \| "refreshable" \| "runtime">` | Gibt an, ob der Provider-Katalog aus Manifest-Metadaten gelesen, im Cache aktualisiert oder nur über die Laufzeit abgerufen werden kann. |
| `runtimeAugment` | `boolean`                                                | Nur auf `true` setzen, wenn die Provider-Laufzeit nach der Manifest-/Konfigurationsplanung Katalogzeilen ergänzen muss. |

`aliases` ist an der Ermittlung der Provider-Zuständigkeit für die Modellkatalogplanung beteiligt. Aliasziele müssen Provider der obersten Ebene sein, die demselben Plugin gehören. Wenn eine nach Provider gefilterte Liste einen Alias verwendet, kann OpenClaw das zugehörige Manifest lesen und die API-/Basis-URL-Überschreibungen des Alias anwenden, ohne die Provider-Laufzeit zu laden. Aliasse erweitern ungefilterte Katalogauflistungen nicht; umfassende Listen geben nur die Zeilen des zugehörigen kanonischen Providers aus.

`suppressions` ersetzt den alten `suppressBuiltInModel`-Hook der Provider-Laufzeit. Unterdrückungseinträge werden nur berücksichtigt, wenn der Provider dem Plugin gehört oder als `modelCatalog.aliases`-Schlüssel deklariert ist, der auf einen zugehörigen Provider verweist. Laufzeit-Hooks zur Unterdrückung werden bei der Modellauflösung nicht mehr aufgerufen.

Provider-Felder:

| Feld                  | Typ                      | Bedeutung                                                                                                                                                                                                        |
| --------------------- | ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `baseUrl`             | `string`                 | Optionale Standard-Basis-URL für Modelle in diesem Provider-Katalog.                                                                                                                                              |
| `api`                 | `ModelApi`               | Optionaler Standard-API-Adapter für Modelle in diesem Provider-Katalog.                                                                                                                                           |
| `headers`             | `Record<string, string>` | Optionale statische Header, die für diesen Provider-Katalog gelten.                                                                                                                                               |
| `defaultUtilityModel` | `string`                 | Optionale, vom Provider empfohlene ID eines kleinen Modells für kurze interne Hilfsaufgaben (Titel, Fortschrittsbeschreibung). Wird verwendet, wenn `agents.defaults.utilityModel` nicht gesetzt ist und dieser Provider das primäre Modell des Agenten bereitstellt. |
| `models`              | `object[]`               | Erforderliche Modellzeilen. Zeilen ohne `id` werden ignoriert.                                                                                                                                      |

Modellfelder:

| Feld               | Typ                                                            | Bedeutung                                                                   |
| ------------------ | -------------------------------------------------------------- | --------------------------------------------------------------------------- |
| `id`               | `string`                                                       | Provider-lokale Modell-ID ohne das Präfix `provider/`.                      |
| `name`             | `string`                                                       | Optionaler Anzeigename.                                                     |
| `api`              | `ModelApi`                                                     | Optionale modellspezifische API-Überschreibung.                             |
| `baseUrl`          | `string`                                                       | Optionale modellspezifische Überschreibung der Basis-URL.                   |
| `headers`          | `Record<string, string>`                                       | Optionale modellspezifische statische Header.                               |
| `input`            | `Array<"text" \| "image" \| "document">`                       | Modalitäten, die das Modell akzeptiert. Andere Werte werden stillschweigend verworfen. |
| `reasoning`        | `boolean`                                                      | Gibt an, ob das Modell Reasoning-Verhalten bereitstellt.                    |
| `contextWindow`    | `number`                                                       | Natives Kontextfenster des Providers.                                       |
| `contextTokens`    | `number`                                                       | Optionale effektive Kontextobergrenze der Laufzeit, falls sie von `contextWindow` abweicht. |
| `maxTokens`        | `number`                                                       | Maximale Anzahl von Ausgabe-Tokens, sofern bekannt.                         |
| `thinkingLevelMap` | `Record<string, string \| null>`                               | Optionale, nach Denkstufe aufgeschlüsselte Überschreibungen der Modell-ID oder Parameter. |
| `cost`             | `object`                                                       | Optionale Preise in USD pro Million Tokens, einschließlich des optionalen `tieredPricing`. |
| `compat`           | `object`                                                       | Optionale Kompatibilitätsflags, die der Kompatibilität der OpenClaw-Modellkonfiguration entsprechen. |
| `mediaInput`       | `object`                                                       | Optionale Eingabekonfiguration pro Modalität, derzeit nur für Bilder.        |
| `status`           | `"available"` \| `"preview"` \| `"deprecated"` \| `"disabled"` | Auflistungsstatus. Nur unterdrücken, wenn die Zeile überhaupt nicht erscheinen darf. |
| `statusReason`     | `string`                                                       | Optionaler Grund, der bei einem Status „nicht verfügbar“ angezeigt wird.    |
| `replaces`         | `string[]`                                                     | Ältere Provider-lokale Modell-IDs, die dieses Modell ersetzt.               |
| `replacedBy`       | `string`                                                       | Provider-lokale ID des Ersatzmodells für veraltete Zeilen.                  |
| `tags`             | `string[]`                                                     | Stabile Tags, die von Auswahlfeldern und Filtern verwendet werden.          |

Unterdrückungsfelder:

| Feld                       | Typ        | Bedeutung                                                                                                 |
| -------------------------- | ---------- | --------------------------------------------------------------------------------------------------------- |
| `provider`                 | `string`   | Provider-ID der zu unterdrückenden Upstream-Zeile. Muss diesem Plugin gehören oder als eigener Alias deklariert sein. |
| `model`                    | `string`   | Provider-lokale Modell-ID, die unterdrückt werden soll.                                                    |
| `reason`                   | `string`   | Optionale Meldung, die angezeigt wird, wenn die unterdrückte Zeile direkt angefordert wird.               |
| `when.baseUrlHosts`        | `string[]` | Optionale Liste der Hosts effektiver Provider-Basis-URLs, die erforderlich sind, bevor die Unterdrückung greift. |
| `when.providerConfigApiIn` | `string[]` | Optionale Liste exakter `api`-Werte der Provider-Konfiguration, die erforderlich sind, bevor die Unterdrückung greift. |

Legen Sie keine reinen Laufzeitdaten in `modelCatalog` ab. Verwenden Sie `static` nur, wenn die Manifestzeilen vollständig genug sind, damit nach Provider gefilterte Listen- und Auswahloberflächen die Registry-/Laufzeitermittlung überspringen können. Verwenden Sie `refreshable`, wenn Manifestzeilen als auflistbare Ausgangsdaten oder Ergänzungen nützlich sind, aber eine Aktualisierung bzw. ein Cache später weitere Zeilen hinzufügen kann; aktualisierbare Zeilen sind für sich genommen nicht maßgeblich. Verwenden Sie `runtime`, wenn OpenClaw die Provider-Laufzeit laden muss, um die Liste zu ermitteln.

## Referenz zu modelIdNormalization

Verwenden Sie `modelIdNormalization` für einfache, dem Provider zugehörige Bereinigungen von Modell-IDs, die erfolgen müssen, bevor die Provider-Laufzeit geladen wird. Dadurch verbleiben Aliasse wie kurze Modellnamen, ältere Provider-lokale IDs und Regeln für Proxy-Präfixe im Manifest des zuständigen Plugins statt in den zentralen Tabellen zur Modellauswahl.

```json
{
  "providers": ["anthropic", "openrouter"],
  "modelIdNormalization": {
    "providers": {
      "anthropic": {
        "aliases": {
          "sonnet-4.6": "claude-sonnet-4-6"
        }
      },
      "openrouter": {
        "prefixWhenBare": "openrouter"
      }
    }
  }
}
```

Provider-Felder:

| Feld                                 | Typ                     | Bedeutung                                                                                 |
| ------------------------------------ | ----------------------- | ----------------------------------------------------------------------------------------- |
| `aliases`                            | `Record<string,string>` | Exakte Modell-ID-Aliasse ohne Berücksichtigung der Groß-/Kleinschreibung. Werte werden wie angegeben zurückgegeben. |
| `stripPrefixes`                      | `string[]`              | Präfixe, die vor der Alias-Suche entfernt werden; nützlich bei älteren Dopplungen von Provider und Modell. |
| `prefixWhenBare`                     | `string`                | Präfix, das hinzugefügt wird, wenn die normalisierte Modell-ID noch kein `/` enthält. |
| `prefixWhenBareAfterAliasStartsWith` | `object[]`              | Bedingte Präfixregeln für IDs ohne Präfix nach der Alias-Suche, nach `modelPrefix` und `prefix` geordnet. |

## Referenz zu providerEndpoints

Verwenden Sie `providerEndpoints` für die Endpunktklassifizierung, die allgemeine Anfragerichtlinien kennen müssen, bevor die Provider-Laufzeit geladen wird. Der Kern bestimmt weiterhin die Bedeutung jeder `endpointClass`; Plugin-Manifeste enthalten die Host- und Basis-URL-Metadaten.

Offiziell externalisierte Provider-Plugins sind von der Kern-Distribution ausgeschlossen, sodass
ihre Manifeste bis zur Installation nicht sichtbar sind. Ihre `providerEndpoints` müssen
auch in `scripts/lib/official-external-provider-catalog.json` gespiegelt werden, damit
die Endpunktklassifizierung ohne das Plugin weiterhin funktioniert; ein Vertragstest
erzwingt diese Spiegelung.

Endpunktfelder:

| Feld                           | Typ        | Bedeutung                                                                                     |
| ------------------------------ | ---------- | ---------------------------------------------------------------------------------------------- |
| `endpointClass`                | `string`   | Bekannte Kern-Endpunktklasse, beispielsweise `openrouter`, `moonshot-native` oder `google-vertex`. |
| `hosts`                        | `string[]` | Exakte Hostnamen, die der Endpunktklasse zugeordnet werden.                                    |
| `hostSuffixes`                 | `string[]` | Host-Suffixe, die der Endpunktklasse zugeordnet werden. Stellen Sie `.` voran, um nur Domain-Suffixe abzugleichen. |
| `baseUrls`                     | `string[]` | Exakte normalisierte HTTP(S)-Basis-URLs, die der Endpunktklasse zugeordnet werden.              |
| `googleVertexRegion`           | `string`   | Statische Google-Vertex-Region für exakte globale Hosts.                                       |
| `googleVertexRegionHostSuffix` | `string`   | Suffix, das von übereinstimmenden Hosts entfernt wird, um das Präfix der Google-Vertex-Region freizulegen. |

## Referenz zu providerRequest

Verwenden Sie `providerRequest` für einfache Metadaten zur Anfragekompatibilität, die allgemeine Anfragerichtlinien benötigen, ohne die Provider-Laufzeit zu laden. Verhalten-spezifische Umschreibungen der Nutzlast gehören in Laufzeit-Hooks des Providers oder gemeinsame Hilfsfunktionen der Provider-Familie.

```json
{
  "providerRequest": {
    "providers": {
      "vllm": {
        "family": "vllm",
        "openAICompletions": {
          "supportsStreamingUsage": true
        }
      }
    }
  }
}
```

Provider-Felder:

| Feld                  | Typ          | Bedeutung                                                                            |
| --------------------- | ------------ | -------------------------------------------------------------------------------------- |
| `family`              | `string`     | Bezeichnung der Provider-Familie, die für allgemeine Entscheidungen zur Anfragekompatibilität und für Diagnosen verwendet wird. |
| `compatibilityFamily` | `"moonshot"` | Optionaler Kompatibilitätsbereich der Provider-Familie für gemeinsame Anfragehilfsfunktionen. |
| `openAICompletions`   | `object`     | Anfrage-Flags für OpenAI-kompatible Vervollständigungen, derzeit `supportsStreamingUsage`. |

## Referenz zu secretProviderIntegrations

Verwenden Sie `secretProviderIntegrations`, wenn ein Plugin eine wiederverwendbare Voreinstellung für einen SecretRef-Exec-Provider veröffentlichen kann. OpenClaw liest diese Metadaten, bevor die Plugin-Laufzeit geladen wird, speichert die Plugin-Zuständigkeit in `secrets.providers.<alias>.pluginIntegration` und überlässt die eigentliche Auflösung von Geheimnissen der SecretRef-Laufzeit. Voreinstellungen werden nur für gebündelte Plugins und installierte Plugins angeboten, die in den verwalteten Plugin-Installationsverzeichnissen gefunden wurden, beispielsweise Installationen über Git und ClawHub.

```json
{
  "secretProviderIntegrations": {
    "secret-store": {
      "providerAlias": "team-secrets",
      "displayName": "Team secrets",
      "source": "exec",
      "command": "${node}",
      "args": ["./bin/resolve-secrets.mjs"]
    }
  }
}
```

Der Map-Schlüssel ist die Integrations-ID. Wenn `providerAlias` weggelassen wird, verwendet OpenClaw die Integrations-ID als SecretRef-Provider-Alias. Provider-Aliasse müssen dem üblichen Muster für SecretRef-Provider-Aliasse entsprechen, beispielsweise `team-secrets` oder `onepassword-work`.

Wenn eine zuständige Person die Voreinstellung auswählt, schreibt OpenClaw eine Provider-Referenz wie diese:

```json
{
  "secrets": {
    "providers": {
      "team-secrets": {
        "source": "exec",
        "pluginIntegration": {
          "pluginId": "acme-secrets",
          "integrationId": "secret-store"
        }
      }
    }
  }
}
```

Beim Start bzw. Neuladen löst OpenClaw diesen Provider auf, indem es die aktuellen Metadaten des Plugin-Manifests lädt, prüft, ob das zuständige Plugin installiert und aktiv ist, und den Exec-Befehl aus dem Manifest erzeugt. Wird das Plugin deaktiviert oder entfernt, wird der Provider für aktive SecretRefs widerrufen. Zuständige Personen, die eine eigenständige Exec-Konfiguration wünschen, können weiterhin manuelle `command`-/`args`-Provider direkt angeben.

Derzeit werden nur `source: "exec"`-Voreinstellungen unterstützt. `command` muss `${node}` sein und `args[0]` muss ein `./`-Auflösungsskript relativ zum Plugin-Stammverzeichnis sein. OpenClaw setzt dies beim Start bzw. Neuladen in die aktuelle ausführbare Node-Datei und den absoluten Skriptpfad innerhalb des Plugins um. Node-Optionen wie `--require`, `--import`, `--loader`, `--env-file`, `--eval` und `--print` sind nicht Teil des Vertrags für Manifest-Voreinstellungen. Zuständige Personen, die Nicht-Node-Befehle benötigen, können eigenständige manuelle Exec-Provider direkt konfigurieren.

OpenClaw leitet `trustedDirs` für Manifest-Voreinstellungen aus dem Plugin-Stammverzeichnis und bei `${node}`-Voreinstellungen aus dem Verzeichnis der aktuellen ausführbaren Node-Datei ab. Im Manifest definierte `trustedDirs` werden ignoriert. Andere Optionen des Exec-Providers wie `timeoutMs`, `noOutputTimeoutMs`, `maxOutputBytes`, `jsonOnly`, `env`, `passEnv` und `allowInsecurePath` werden an die normale Konfiguration des SecretRef-Exec-Providers weitergereicht.

## Referenz zu modelPricing

Verwenden Sie `modelPricing`, wenn ein Provider das Preisverhalten der Steuerungsebene festlegen muss, bevor die Laufzeit geladen wird. Der Preis-Cache des Gateways liest diese Metadaten, ohne den Laufzeitcode des Providers zu importieren.

```json
{
  "providers": ["ollama", "openrouter"],
  "modelPricing": {
    "providers": {
      "ollama": {
        "external": false
      },
      "openrouter": {
        "openRouter": {
          "passthroughProviderModel": true
        },
        "liteLLM": false
      }
    }
  }
}
```

Provider-Felder:

| Feld         | Typ               | Bedeutung                                                                                         |
| ------------ | ----------------- | -------------------------------------------------------------------------------------------------- |
| `external`   | `boolean`         | Setzen Sie `false` für lokale bzw. selbst gehostete Provider, die niemals Preisdaten von OpenRouter oder LiteLLM abrufen sollen. |
| `openRouter` | `false \| object` | Zuordnung für die Preissuche über OpenRouter. `false` deaktiviert die OpenRouter-Suche für diesen Provider. |
| `liteLLM`    | `false \| object` | Zuordnung für die Preissuche über LiteLLM. `false` deaktiviert die LiteLLM-Suche für diesen Provider. |

Quellfelder:

| Feld                       | Typ                | Bedeutung                                                                                                             |
| -------------------------- | ------------------ | -------------------------------------------------------------------------------------------------------------------- |
| `provider`                 | `string`           | Provider-ID im externen Katalog, wenn sie von der OpenClaw-Provider-ID abweicht, beispielsweise `z-ai` für einen `zai`-Provider. |
| `passthroughProviderModel` | `boolean`          | Modell-IDs mit Schrägstrichen als verschachtelte Provider-/Modellreferenzen behandeln; nützlich für Proxy-Provider wie OpenRouter. |
| `modelIdTransforms`        | `"version-dots"[]` | Zusätzliche Modell-ID-Varianten für externe Kataloge. `version-dots` versucht Versions-IDs mit Punkten wie `claude-opus-4.6`. |

### OpenClaw-Provider-Index

Der OpenClaw-Provider-Index besteht aus OpenClaw-eigenen Vorschau-Metadaten für Provider, deren Plugins möglicherweise noch nicht installiert sind. Er ist nicht Teil eines Plugin-Manifests. Plugin-Manifeste bleiben die maßgebliche Quelle für installierte Plugins. Der Provider-Index ist der interne Rückfallvertrag, den künftige Oberflächen für installierbare Provider und die Modellauswahl vor der Installation verwenden, wenn ein Provider-Plugin nicht installiert ist.

Reihenfolge der Katalogautorität:

1. Benutzerkonfiguration.
2. Installiertes Plugin-Manifest `modelCatalog`.
3. Modellkatalog-Cache aus einer expliziten Aktualisierung.
4. Vorschauzeilen des OpenClaw-Provider-Index.

Der Provider-Index darf keine Geheimnisse, Aktivierungszustände, Runtime-Hooks oder Live-Modelldaten enthalten, die für ein bestimmtes Konto gelten. Seine Vorschaukataloge verwenden dieselbe `modelCatalog`-Provider-Zeilenstruktur wie Plugin-Manifeste, sollten jedoch auf stabile Anzeigemetadaten beschränkt bleiben, sofern Runtime-Adapterfelder wie `api`, `baseUrl`, Preise oder Kompatibilitäts-Flags nicht absichtlich mit dem installierten Plugin-Manifest synchron gehalten werden. Provider mit Live-Erkennung über `/models` sollten aktualisierte Zeilen über den expliziten Cache-Pfad des Modellkatalogs schreiben, statt bei der normalen Auflistung oder beim Onboarding Provider-APIs aufzurufen.

Einträge im Provider-Index können außerdem Metadaten für installierbare Plugins enthalten, wenn das Plugin eines Providers aus dem Kern verlagert wurde oder aus anderen Gründen noch nicht installiert ist. Diese Metadaten entsprechen dem Muster des Kanalkatalogs: Paketname, npm-Installationsspezifikation, erwartete Integrität und einfache Bezeichnungen für Authentifizierungsoptionen reichen aus, um eine installierbare Einrichtungsoption anzuzeigen. Sobald das Plugin installiert ist, hat sein Manifest Vorrang, und der Eintrag im Provider-Index wird für diesen Provider ignoriert.

`openclaw doctor --fix` migriert eine kleine, abgeschlossene Menge veralteter Manifest-Fähigkeitsschlüssel der obersten Ebene nach `contracts.*`: `speechProviders`, `mediaUnderstandingProviders`, `imageGenerationProviders` und `tools`. Keiner dieser Schlüssel – und auch keine andere Fähigkeitsliste – wird weiterhin als Manifestfeld der obersten Ebene gelesen; das normale Laden von Manifesten erkennt sie nur unter `contracts`.

## Manifest im Vergleich zu package.json

Die beiden Dateien erfüllen unterschiedliche Aufgaben:

| Datei                   | Verwendung                                                                                                                       |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.plugin.json` | Erkennung, Konfigurationsvalidierung, Metadaten für Authentifizierungsoptionen und UI-Hinweise, die vorhanden sein müssen, bevor Plugin-Code ausgeführt wird                         |
| `package.json`         | npm-Metadaten, Installation von Abhängigkeiten und der `openclaw`-Block für Einstiegspunkte, Installationsbedingungen, Einrichtung oder Katalogmetadaten |

Wenn unklar ist, wohin bestimmte Metadaten gehören, gilt folgende Regel:

- wenn OpenClaw sie vor dem Laden des Plugin-Codes kennen muss, gehören sie in `openclaw.plugin.json`
- wenn sie die Paketierung, Einstiegsdateien oder das npm-Installationsverhalten betreffen, gehören sie in `package.json`

### package.json-Felder, die die Erkennung beeinflussen

Einige Plugin-Metadaten, die vor der Runtime benötigt werden, befinden sich absichtlich in `package.json` unter dem `openclaw`-Block statt in `openclaw.plugin.json`. `openclaw.bundle` und `openclaw.bundle.json` sind keine OpenClaw-Plugin-Verträge; native Plugins müssen `openclaw.plugin.json` zusammen mit den nachfolgend unterstützten `package.json#openclaw`-Feldern verwenden.

Wichtige Beispiele:

| Feld                                                                                      | Bedeutung                                                                                                                                                                             |
| ------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.extensions`                                                                      | Deklariert native Plugin-Einstiegspunkte. Sie müssen innerhalb des Plugin-Paketverzeichnisses bleiben.                                                                                                        |
| `openclaw.runtimeExtensions`                                                               | Deklariert erstellte JavaScript-Runtime-Einstiegspunkte für installierte Pakete. Sie müssen innerhalb des Plugin-Paketverzeichnisses bleiben.                                                                      |
| `openclaw.setupEntry`                                                                      | Einfacher Einstiegspunkt ausschließlich für die Einrichtung, der beim Onboarding, beim verzögerten Kanalstart sowie für schreibgeschützte Kanalstatus- und SecretRef-Erkennung verwendet wird. Er muss innerhalb des Plugin-Paketverzeichnisses bleiben.      |
| `openclaw.runtimeSetupEntry`                                                               | Deklariert den erstellten JavaScript-Einrichtungseinstiegspunkt für installierte Pakete. Er erfordert `setupEntry`, muss vorhanden sein und innerhalb des Plugin-Paketverzeichnisses bleiben.                              |
| `openclaw.channel`                                                                         | Einfache Kanalkatalog-Metadaten wie Bezeichnungen, Dokumentationspfade, Aliasse und Auswahltexte.                                                                                                      |
| `openclaw.channel.approvalFlags`                                                           | Abgeschlossene Flags für das Genehmigungsverhalten, die vor dem Laden der Runtime verfügbar sind. `native` bedeutet, dass der Kanal eine native Genehmigungs-UI und die Auflösung im selben Durchlauf verwaltet.                                                |
| `openclaw.channel.commands`                                                                | Statische Metadaten für automatische Standardwerte nativer Befehle und nativer Skills, die von Konfigurations-, Audit- und Befehlslistenoberflächen verwendet werden, bevor die Kanal-Runtime geladen wird.                                               |
| `openclaw.channel.cliAddOptions`                                                           | Plugin-eigene `openclaw channels add`-Optionen. Jeder Eintrag deklariert `flags`, `description`, optional `defaultValue` und optional `valueType` (`int` oder `list`) für die generische Eingabekonvertierung. |
| `openclaw.channel.configuredState`                                                         | Einfache Metadaten zur Prüfung des Konfigurationszustands, mit denen sich ohne Laden der vollständigen Kanal-Runtime beantworten lässt: „Ist bereits eine ausschließlich umgebungsbasierte Einrichtung vorhanden?“                                              |
| `openclaw.channel.persistedAuthState`                                                      | Einfache Metadaten zur Prüfung persistierter Authentifizierung, mit denen sich ohne Laden der vollständigen Kanal-Runtime beantworten lässt: „Ist bereits irgendwo eine Anmeldung vorhanden?“                                                    |
| `openclaw.install.clawhubSpec` / `openclaw.install.npmSpec` / `openclaw.install.localPath` | Installations- und Aktualisierungshinweise für gebündelte und extern veröffentlichte Plugins.                                                                                                                        |
| `openclaw.install.defaultChoice`                                                           | Bevorzugter Installationspfad, wenn mehrere Installationsquellen verfügbar sind.                                                                                                                       |
| `openclaw.install.minHostVersion`                                                          | Unterstützte Mindestversion des OpenClaw-Hosts unter Verwendung einer semantischen Versionsuntergrenze wie `>=2026.3.22` oder `>=2026.5.1-beta.1`.                                                                                  |
| `openclaw.compat.pluginApi`                                                                | Vom Paket benötigter Mindestversionsbereich der OpenClaw-Plugin-API unter Verwendung einer semantischen Versionsuntergrenze wie `>=2026.5.27`.                                                                                      |
| `openclaw.install.expectedIntegrity`                                                       | Erwartete npm-Dist-Integritätszeichenfolge wie `sha512-...`; Installations- und Aktualisierungsabläufe prüfen das abgerufene Artefakt dagegen.                                                                 |
| `openclaw.install.allowInvalidConfigRecovery`                                              | Ermöglicht einen eng begrenzten Wiederherstellungspfad zur Neuinstallation gebündelter Plugins, wenn die Konfiguration ungültig ist.                                                                                                            |
| `openclaw.install.requiredPlatformPackages`                                                | npm-Paketaliasse, die bereitgestellt werden müssen, wenn ihre Lockfile-Plattformbeschränkungen mit dem aktuellen Host übereinstimmen.                                                                                |
| `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen`                          | Ermöglicht das Laden von Kanaloberflächen der Einrichtungs-Runtime vor dem Lauschen und verzögert anschließend das vollständige konfigurierte Kanal-Plugin bis zur Aktivierung nach Beginn des Lauschens.                                                      |

Manifestmetadaten bestimmen, welche Provider-, Kanal- und Einrichtungsoptionen beim Onboarding angezeigt werden, bevor die Runtime geladen wird. `package.json#openclaw.install` teilt dem Onboarding mit, wie dieses Plugin abgerufen oder aktiviert werden soll, wenn eine dieser Optionen ausgewählt wird. Verschieben Sie Installationshinweise nicht nach `openclaw.plugin.json`.

Verwenden Sie für `openclaw.channel.cliAddOptions` die Langoptionssyntax von Commander, beispielsweise `--initial-sync-limit <n>`. Setzen Sie `valueType: "int"`, um eine nicht negative Ganzzahl zu parsen, oder `valueType: "list"`, um durch Kommas, Semikolons oder Zeilenumbrüche getrennte Eingaben in Zeichenfolgen aufzuteilen, bevor der Plugin-Einrichtungsadapter sie empfängt. Lassen Sie `valueType` weg, um den von Commander geparsten Wert unverändert weiterzugeben.

`openclaw.install.minHostVersion` wird während der Installation und beim Laden der Manifestregistrierung für nicht gebündelte Plugin-Quellen durchgesetzt. Ungültige Werte werden abgelehnt; neuere, aber gültige Werte führen dazu, dass externe Plugins auf älteren Hosts übersprungen werden. Bei gebündelten Quell-Plugins wird davon ausgegangen, dass sie dieselbe Version wie der Host-Checkout aufweisen.

`openclaw.install.requiredPlatformPackages` ist für npm-Pakete vorgesehen, die erforderliche native Binärdateien über optionale, plattformspezifische Aliasse bereitstellen. Geben Sie für jeden unterstützten Plattformalias den reinen npm-Paketnamen an. Während der npm-Installation überprüft OpenClaw nur den deklarierten Alias, dessen Lockfile-Beschränkungen mit dem aktuellen Host übereinstimmen. Wenn npm Erfolg meldet, diesen Alias jedoch auslässt, wiederholt OpenClaw den Vorgang einmal mit einem frischen Cache und setzt die Installation zurück, falls der Alias weiterhin fehlt.

`openclaw.compat.pluginApi` wird während der Paketinstallation für nicht gebündelte Plugin-Quellen durchgesetzt. Verwenden Sie es für die Mindestversion der OpenClaw-Plugin-SDK-/Runtime-API, gegen die das Paket erstellt wurde. Sie kann strenger als `minHostVersion` sein, wenn ein Plugin-Paket eine neuere API benötigt, aber für andere Abläufe weiterhin einen niedrigeren Installationshinweis beibehält. Die offizielle OpenClaw-Release-Synchronisierung hebt vorhandene offizielle Plugin-API-Mindestversionen standardmäßig auf die OpenClaw-Release-Version an. Reine Plugin-Releases können jedoch eine niedrigere Mindestversion beibehalten, wenn das Paket absichtlich ältere Hosts unterstützt. Verwenden Sie nicht allein die Paketversion als Kompatibilitätsvertrag. `peerDependencies.openclaw` bleibt npm-Paketmetadatum; OpenClaw verwendet den `openclaw.compat.pluginApi`-Vertrag für Entscheidungen zur Installationskompatibilität.

Offizielle Metadaten für die Installation bei Bedarf sollten `clawhubSpec` verwenden, wenn das Plugin auf ClawHub veröffentlicht ist; das Onboarding behandelt dies als bevorzugte Remote-Quelle und zeichnet nach der Installation Fakten zum ClawHub-Artefakt auf. `npmSpec` bleibt der Kompatibilitäts-Fallback für Pakete, die noch nicht zu ClawHub verschoben wurden.

Die exakte Fixierung der npm-Version befindet sich bereits in `npmSpec`, beispielsweise `"npmSpec": "@wecom/wecom-openclaw-plugin@1.2.3"`. Offizielle externe Katalogeinträge sollten exakte Spezifikationen mit `expectedIntegrity` kombinieren, damit Aktualisierungsabläufe sicher abbrechen, wenn das abgerufene npm-Artefakt nicht mehr dem fixierten Release entspricht. Das interaktive Onboarding bietet aus Kompatibilitätsgründen weiterhin vertrauenswürdige npm-Spezifikationen aus der Registry an, einschließlich reiner Paketnamen und Dist-Tags. Katalogdiagnosen können zwischen exakten, variablen, integritätsfixierten, ohne Integritätsangabe versehenen, durch abweichende Paketnamen gekennzeichneten und ungültigen Standardauswahlquellen unterscheiden. Sie warnen außerdem, wenn `expectedIntegrity` vorhanden ist, aber keine gültige npm-Quelle existiert, die damit fixiert werden kann. Wenn `expectedIntegrity` vorhanden ist, setzen Installations- und Aktualisierungsabläufe es durch; wenn es weggelassen wird, wird die Registry-Auflösung ohne Integritätsfixierung aufgezeichnet.

Kanal-Plugins sollten `openclaw.setupEntry` bereitstellen, wenn Status-, Kanallisten- oder SecretRef-Prüfungen konfigurierte Konten identifizieren müssen, ohne die vollständige Runtime zu laden. Der Einrichtungseinstiegspunkt sollte Kanalmetadaten sowie einrichtungssichere Adapter für Konfiguration, Status und Geheimnisse bereitstellen; Netzwerkclients, Gateway-Listener und Transport-Runtimes gehören in den Haupteinstiegspunkt der Erweiterung.

Laufzeit-Einstiegspunktfelder setzen die Paketgrenzenprüfungen für Quell-Einstiegspunktfelder nicht außer Kraft. Beispielsweise kann `openclaw.runtimeExtensions` einen ausbrechenden `openclaw.extensions`-Pfad nicht ladbar machen.

`openclaw.install.allowInvalidConfigRecovery` ist absichtlich eng begrenzt. Dadurch werden nicht beliebige fehlerhafte Konfigurationen installierbar. Derzeit können Installationsabläufe damit nur bestimmte veraltete Fehler bei Upgrades gebündelter Plugins beheben, etwa einen fehlenden Pfad eines gebündelten Plugins oder einen veralteten `channels.<id>`-Eintrag für dasselbe gebündelte Plugin. Nicht damit zusammenhängende Konfigurationsfehler blockieren die Installation weiterhin und verweisen Betreiber auf `openclaw doctor --fix`.

`openclaw.channel.persistedAuthState` enthält Paketmetadaten für ein kleines Prüfmodul:

```json
{
  "openclaw": {
    "channel": {
      "id": "whatsapp",
      "persistedAuthState": {
        "specifier": "./auth-presence",
        "exportName": "hasAnyWhatsAppAuth"
      }
    }
  }
}
```

Verwenden Sie dies, wenn Einrichtung, Doctor, Status oder schreibgeschützte Anwesenheitsabläufe eine kostengünstige Ja/Nein-Authentifizierungsprüfung benötigen, bevor das vollständige Kanal-Plugin geladen wird. Persistierter Authentifizierungsstatus ist kein konfigurierter Kanalstatus: Verwenden Sie diese Metadaten nicht, um Plugins automatisch zu aktivieren, Laufzeitabhängigkeiten zu reparieren oder zu entscheiden, ob eine Kanallaufzeit geladen werden soll. Der Zielexport sollte eine kleine Funktion sein, die ausschließlich den persistierten Status liest; leiten Sie ihn nicht durch das vollständige Kanallaufzeit-Barrel.

`openclaw.channel.configuredState` unterstützt kostengünstige Prüfungen des Konfigurationsstatus. Bevorzugen Sie deklarative Umgebungsmetadaten, wenn Umgebungsvariablen ausreichen:

```json
{
  "openclaw": {
    "channel": {
      "id": "telegram",
      "configuredState": {
        "env": {
          "allOf": ["TELEGRAM_BOT_TOKEN"]
        }
      }
    }
  }
}
```

Verwenden Sie `env.allOf`, wenn jede aufgeführte Variable erforderlich ist, und `env.anyOf`, wenn eine beliebige nicht leere Variable ausreicht. Wenn eine kleine, laufzeitunabhängige Prüfung mehr als Umgebungsmetadaten benötigt, verwenden Sie `specifier` zusammen mit `exportName`, wie für `persistedAuthState` gezeigt; wenn `env` vorhanden ist, verwendet OpenClaw dies, ohne das betreffende Modul zu laden. Wenn die Prüfung eine vollständige Konfigurationsauflösung oder die tatsächliche Kanallaufzeit benötigt, belassen Sie diese Logik stattdessen im `config.hasConfiguredState`-Hook des Plugins.

## Ermittlungspriorität (doppelte Plugin-IDs)

OpenClaw ermittelt Plugins aus drei Stammverzeichnissen, die in dieser Reihenfolge geprüft werden: mit OpenClaw ausgelieferte gebündelte Plugins, das globale Installationsstammverzeichnis (`~/.openclaw/extensions`) und das aktuelle Arbeitsbereichsstammverzeichnis (`<workspace>/.openclaw/extensions`) sowie alle expliziten `plugins.load.paths`-Einträge.

Wenn zwei Ermittlungen dieselbe `id` aufweisen, wird nur das Manifest mit der **höchsten Priorität** beibehalten; Duplikate mit niedrigerer Priorität werden verworfen, statt parallel dazu geladen zu werden. Priorität, von der höchsten zur niedrigsten:

1. **Durch Konfiguration ausgewählt** — ein explizit in `plugins.entries.<id>` festgelegter Pfad
2. **Globale Installation mit passendem nachverfolgtem Installationsdatensatz** — ein über `openclaw plugin install`/`openclaw plugin update` installiertes Plugin, das von OpenClaws Installationsverfolgung für dieselbe ID erkannt wird, selbst wenn die ID auch zu einem gebündelten Plugin gehört
3. **Gebündelt** — mit OpenClaw ausgelieferte Plugins
4. **Arbeitsbereich** — relativ zum aktuellen Arbeitsbereich ermittelte Plugins
5. Alle anderen ermittelten Kandidaten

Auswirkungen:

- Eine geforkte oder veraltete Kopie eines gebündelten Plugins, die sich nicht nachverfolgt im Arbeitsbereich oder globalen Stammverzeichnis befindet, überschattet den gebündelten Build nicht.
- Um ein gebündeltes Plugin zu überschreiben, führen Sie entweder `openclaw plugin install` für diese ID aus, sodass die nachverfolgte globale Installation eine höhere Priorität als die gebündelte Kopie erhält, oder legen Sie über `plugins.entries.<id>` einen bestimmten Pfad fest, damit dieser aufgrund der konfigurationsgesteuerten Priorität Vorrang erhält.
- Das Verwerfen von Duplikaten wird protokolliert, damit Doctor und die Startdiagnose auf die verworfene Kopie verweisen können.
- Durch Konfiguration ausgewählte Überschreibungen von Duplikaten werden in der Diagnose als explizite Überschreibungen bezeichnet, lösen aber weiterhin eine Warnung aus, damit veraltete Forks und versehentliche Überschattungen sichtbar bleiben.

## Anforderungen an das JSON-Schema

- **Jedes Plugin muss ein JSON-Schema ausliefern**, auch wenn es keine Konfiguration akzeptiert.
- Ein leeres Schema ist zulässig (beispielsweise `{ "type": "object", "additionalProperties": false }`).
- Schemas werden beim Lesen und Schreiben der Konfiguration validiert, nicht zur Laufzeit.
- Wenn Sie ein gebündeltes Plugin um neue Konfigurationsschlüssel erweitern oder forken, aktualisieren Sie gleichzeitig dessen `openclaw.plugin.json` `configSchema`. Schemas gebündelter Plugins sind strikt. Daher wird das Hinzufügen von `plugins.entries.<id>.config.myNewKey` zur Benutzerkonfiguration ohne gleichzeitiges Hinzufügen von `myNewKey` zu `configSchema.properties` abgelehnt, bevor die Plugin-Laufzeit geladen wird.

Beispiel für eine Schemaerweiterung:

```json
{
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "myNewKey": {
        "type": "string"
      }
    }
  }
}
```

## Validierungsverhalten

- Unbekannte `channels.*`-Schlüssel sind **Fehler**, sofern die Kanal-ID nicht durch ein Plugin-Manifest deklariert wird. Wenn dieselbe ID auch in `plugins.allow`, `plugins.entries` oder `plugins.installs` erscheint (ein referenziertes, aber derzeit nicht ermittelbares Plugin), stuft OpenClaw dies stattdessen zu einer **Warnung** herab.
- `plugins.entries.<id>`, `plugins.allow` und `plugins.deny`, die auf unbekannte Plugin-IDs verweisen, sind **Warnungen** („veralteter Konfigurationseintrag ignoriert“) und keine Fehler, damit Upgrades sowie entfernte oder umbenannte Plugins den Start des Gateways nicht blockieren.
- `plugins.slots.memory`, das auf eine unbekannte Plugin-ID verweist, ist ein **Fehler**. Eine Ausnahme bildet das bekannte offizielle externe Plugin `memory-lancedb`, für das stattdessen eine Warnung ausgegeben wird.
- Wenn ein Plugin installiert ist, aber ein fehlerhaftes oder fehlendes Manifest oder Schema aufweist, schlägt die Validierung fehl und Doctor meldet den Plugin-Fehler.
- Wenn eine Plugin-Konfiguration vorhanden, das Plugin jedoch **deaktiviert** ist, wird die Konfiguration beibehalten und in Doctor und den Protokollen eine **Warnung** angezeigt.

Das vollständige `plugins.*`-Schema finden Sie in der [Konfigurationsreferenz](/de/gateway/configuration).

## Hinweise

- Das Manifest ist **für native OpenClaw-Plugins erforderlich**, einschließlich Ladevorgängen aus dem lokalen Dateisystem. Die Laufzeit lädt das Plugin-Modul weiterhin separat; das Manifest dient ausschließlich der Ermittlung und Validierung.
- Native Manifeste werden mit JSON5 geparst. Daher werden Kommentare, abschließende Kommata und Schlüssel ohne Anführungszeichen akzeptiert, solange der endgültige Wert weiterhin ein Objekt ist.
- Der Manifest-Loader liest ausschließlich dokumentierte Manifestfelder. Vermeiden Sie benutzerdefinierte Schlüssel auf oberster Ebene.
- `channels`, `providers`, `cliBackends` und `skills` können alle weggelassen werden, wenn ein Plugin sie nicht benötigt.
- `providerCatalogEntry` muss leichtgewichtig bleiben und sollte keinen umfangreichen Laufzeitcode importieren; verwenden Sie es für statische Metadaten des Provider-Katalogs oder eng begrenzte Ermittlungsdeskriptoren, nicht für die Ausführung während einer Anfrage.
- Exklusive Plugin-Arten werden über `plugins.slots.*` ausgewählt: `kind: "memory"` über `plugins.slots.memory` (Standardwert `memory-core`), `kind: "context-engine"` über `plugins.slots.contextEngine` (Standardwert `legacy`).
- Deklarieren Sie die exklusive Plugin-Art in diesem Manifest. Der Laufzeiteintrag `OpenClawPluginDefinition.kind` ist veraltet und bleibt nur als Kompatibilitäts-Fallback für ältere Plugins bestehen.
- Metadaten für Umgebungsvariablen in `setup.providers[].envVars` sind rein deklarativ. Status, Audit, Validierung der Cron-Zustellung und andere schreibgeschützte Oberflächen wenden weiterhin die Plugin-Vertrauens- und effektiven Aktivierungsrichtlinien an, bevor sie eine Umgebungsvariable als konfiguriert behandeln.
- Laufzeitmetadaten für Assistenten, die Provider-Code benötigen, werden unter [Provider-Laufzeit-Hooks](/de/plugins/architecture-internals#provider-runtime-hooks) beschrieben.
- Wenn Ihr Plugin von nativen Modulen abhängt, dokumentieren Sie die Build-Schritte und alle Anforderungen an die Zulassungsliste des Paketmanagers (beispielsweise pnpm `allow-build-scripts` + `pnpm rebuild <package>`).

## Verwandte Themen

<CardGroup cols={3}>
  <Card title="Plugins erstellen" href="/de/plugins/building-plugins" icon="rocket">
    Erste Schritte mit Plugins.
  </Card>
  <Card title="Plugin-Architektur" href="/de/plugins/architecture" icon="diagram-project">
    Interne Architektur und Fähigkeitsmodell.
  </Card>
  <Card title="SDK-Übersicht" href="/de/plugins/sdk-overview" icon="book">
    Plugin-SDK-Referenz und Subpfadimporte.
  </Card>
</CardGroup>
