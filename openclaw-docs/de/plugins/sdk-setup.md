---
read_when:
    - Sie fügen einem Plugin einen Einrichtungsassistenten hinzu
    - Sie müssen den Unterschied zwischen setup-entry.ts und index.ts verstehen.
    - Sie definieren Plugin-Konfigurationsschemata oder OpenClaw-Metadaten in `package.json`
sidebarTitle: Setup and config
summary: Einrichtungsassistenten, setup-entry.ts, Konfigurationsschemata und package.json-Metadaten
title: Plugin-Einrichtung und -Konfiguration
x-i18n:
    generated_at: "2026-07-26T18:40:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b07e3fa365939fa9c0885b31b7894f5e734313a7deef2297e316956063d97e45
    source_path: plugins/sdk-setup.md
    workflow: 16
---

Referenz für Plugin-Paketierung (`package.json`-Metadaten), Manifeste (`openclaw.plugin.json`), Einrichtungseinträge und Konfigurationsschemas.

<Tip>
**Sie suchen eine Schritt-für-Schritt-Anleitung?** Die Anleitungen behandeln die Paketierung im jeweiligen Kontext: [Kanal-Plugins](/plugins/sdk-channel-plugins#step-1-package-and-manifest) und [Provider-Plugins](/de/plugins/sdk-provider-plugins#step-1-package-and-manifest).
</Tip>

## Paketmetadaten

Ihr `package.json` benötigt ein `openclaw`-Feld, das dem Plugin-System mitteilt, was Ihr Plugin bereitstellt:

<Tabs>
  <Tab title="Kanal-Plugin">
    ```json
    {
      "name": "@myorg/openclaw-my-channel",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "setupEntry": "./setup-entry.ts",
        "channel": {
          "id": "my-channel",
          "label": "Mein Kanal",
          "blurb": "Kurzbeschreibung des Kanals."
        }
      }
    }
    ```
  </Tab>
  <Tab title="Provider-Plugin / ClawHub-Baseline">
    ```json openclaw-clawhub-package.json
    {
      "name": "@myorg/openclaw-my-plugin",
      "version": "1.0.0",
      "type": "module",
      "dependencies": {
        "typebox": "1.1.39"
      },
      "peerDependencies": {
        "openclaw": ">=2026.3.24-beta.2"
      },
      "openclaw": {
        "extensions": ["./index.ts"],
        "compat": {
          "pluginApi": ">=2026.3.24-beta.2",
          "minGatewayVersion": "2026.3.24-beta.2"
        },
        "build": {
          "openclawVersion": "2026.3.24-beta.2",
          "pluginSdkVersion": "2026.3.24-beta.2"
        }
      }
    }
    ```
  </Tab>
</Tabs>

<Note>
Für die externe Veröffentlichung auf ClawHub sind `compat` und `build` erforderlich. Die kanonischen Veröffentlichungsbeispiele befinden sich in `docs/snippets/plugin-publish/`.
</Note>

### `openclaw`-Felder

<ParamField path="extensions" type="string[]">
  Einstiegspunktdateien (relativ zum Paketstammverzeichnis). Gültige Quelleinträge für die Entwicklung in Workspaces und Git-Checkouts.
</ParamField>
<ParamField path="runtimeExtensions" type="string[]">
  Erstellte JavaScript-Gegenstücke für `extensions`, die bevorzugt werden, wenn OpenClaw ein installiertes npm-Paket lädt. Siehe [SDK-Einstiegspunkte](/de/plugins/sdk-entrypoints) zur Auflösungsreihenfolge von Quell- und erstellten Dateien.
</ParamField>
<ParamField path="setupEntry" type="string">
  Leichtgewichtiger Einstieg nur für die Einrichtung (optional).
</ParamField>
<ParamField path="runtimeSetupEntry" type="string">
  Erstelltes JavaScript-Gegenstück für `setupEntry`. Erfordert, dass auch `setupEntry` festgelegt ist.
</ParamField>
<ParamField path="plugin" type="object">
  `{ id, label }`-Fallback-Identität des Plugins, die verwendet wird, wenn ein Plugin keine Kanal-/Provider-Metadaten besitzt, aus denen eine ID oder Bezeichnung abgeleitet werden kann.
</ParamField>
<ParamField path="channel" type="object">
  Metadaten des Kanalkatalogs für Einrichtung, Auswahl, Schnellstart und Statusoberflächen.
</ParamField>
<ParamField path="install" type="object">
  Installationshinweise: `npmSpec`, `localPath`, `defaultChoice`, `minHostVersion`, `expectedIntegrity`, `allowInvalidConfigRecovery`, `requiredPlatformPackages`.
</ParamField>
<ParamField path="startup" type="object">
  Kennzeichen für das Startverhalten.
</ParamField>
<ParamField path="compat" type="object">
  Von diesem Plugin unterstützter `pluginApi`-Versionsbereich. Für externe Veröffentlichungen auf ClawHub erforderlich.
</ParamField>

<Note>
Provider-IDs (`providers: string[]`) sind Manifestmetadaten, keine Paketmetadaten. Deklarieren Sie sie in `openclaw.plugin.json`, nicht hier – siehe [Plugin-Manifest](/de/plugins/manifest).
</Note>

### `openclaw.channel`

`openclaw.channel` sind leichtgewichtige Paketmetadaten für die Kanalerkennung und Einrichtungsoberflächen, bevor die Laufzeit geladen wird.

### Kanaleigene Einrichtungsfelder

Kanal-Plugins sollten Einrichtungsfelder einmalig im Laufzeitcode mit `defineChannelSetupContract(...)` definieren und die entsprechende serialisierbare Projektion unter `openclaw.channel.setup.fields` veröffentlichen. Die Laufzeitdefinition leitet den Plugin-lokalen Eingabetyp ab, analysiert sowohl geführte als auch nicht interaktive Werte und hält kanalspezifische Schlüssel aus den Kerntypen heraus. Mithilfe der Paketmetadaten können `openclaw channels add <channel-id> --help` und `openclaw channels add --channel <channel-id> --help` ausschließlich die Optionen des ausgewählten Kanals ermitteln, ohne das Plugin zu laden.

```ts
import { defineChannelSetupContract } from "openclaw/plugin-sdk/channel-setup";

export const setupContract = defineChannelSetupContract({
  fields: {
    endpoint: {
      kind: "string",
      cli: { flags: "--endpoint <url>", description: "Dienstendpunkt" },
    },
    transport: {
      kind: "choice",
      choices: ["native", "container"],
      cli: { flags: "--transport <kind>", description: "Transportverantwortlicher" },
    },
  },
  adapter: {
    applyAccountConfig: ({ cfg, input }) => ({
      ...cfg,
      channels: { ...cfg.channels, example: input },
    }),
  },
});
```

```json
{
  "openclaw": {
    "channel": {
      "id": "example",
      "setup": {
        "fields": [
          {
            "key": "endpoint",
            "kind": "string",
            "cli": { "flags": "--endpoint <url>", "description": "Dienstendpunkt" }
          },
          {
            "key": "transport",
            "kind": "choice",
            "choices": ["native", "container"],
            "cli": { "flags": "--transport <kind>", "description": "Transportverantwortlicher" }
          }
        ]
      }
    }
  }
}
```

Unterstützte Feldarten sind `string`, `boolean`, `integer`, `string-list` und `choice`. Verwenden Sie `sensitive: true` für Anmeldedaten. Jeder Feldschlüssel muss dem in camelCase geschriebenen Attributnamen seines langen CLI-Flags entsprechen, einschließlich einer etwaigen negierten Form, beispielsweise `apiToken` für `--api-token`. Boolesche Felder können `cli.negatedFlags` hinzufügen, wenn sowohl positive als auch `--no-*`-Formen benötigt werden. `channel`, `account` und die Kontoanzeige `name` bleiben die gemeinsame Steuerungshülle.

Der veröffentlichte `setup`/`ChannelSetupInput`-Adapter bleibt für bestehende externe Plugins verfügbar. Neue Plugins sollten `setupContract` bereitstellen; OpenClaw bevorzugt diesen immer, wenn beide vorhanden sind.

| Feld                                   | Typ        | Bedeutung                                                                     |
| -------------------------------------- | ---------- | ----------------------------------------------------------------------------- |
| `id`                                   | `string`   | Kanonische Kanal-ID.                                                          |
| `label`                                | `string`   | Primäre Kanalbezeichnung.                                                     |
| `selectionLabel`                       | `string`   | Auswahl-/Einrichtungsbezeichnung, wenn sie von `label` abweichen soll.      |
| `detailLabel`                          | `string`   | Sekundäre Detailbezeichnung für umfangreichere Kanalkataloge und Statusoberflächen. |
| `docsPath`                             | `string`   | Dokumentationspfad für Einrichtungs- und Auswahllinks.                        |
| `docsLabel`                            | `string`   | Überschriebene Bezeichnung für Dokumentationslinks, wenn sie von der Kanal-ID abweichen soll. |
| `blurb`                                | `string`   | Kurze Onboarding-/Katalogbeschreibung.                                        |
| `order`                                | `number`   | Sortierreihenfolge in Kanalkatalogen.                                         |
| `aliases`                              | `string[]` | Zusätzliche Suchaliase für die Kanalauswahl.                                  |
| `preferOver`                           | `string[]` | Plugin-/Kanal-IDs mit niedrigerer Priorität, die dieser Kanal übertreffen soll. |
| `systemImage`                          | `string`   | Optionaler Symbol-/Systembildname für Kanalkataloge der Benutzeroberfläche.   |
| `selectionDocsPrefix`                  | `string`   | Präfixtext vor Dokumentationslinks in Auswahloberflächen.                     |
| `selectionDocsOmitLabel`               | `boolean`  | Den Dokumentationspfad direkt anstelle eines beschrifteten Dokumentationslinks im Auswahltext anzeigen. |
| `selectionExtras`                      | `string[]` | Zusätzliche kurze Zeichenfolgen, die an den Auswahltext angehängt werden.     |
| `markdownCapable`                      | `boolean`  | Kennzeichnet den Kanal für Entscheidungen zur ausgehenden Formatierung als Markdown-fähig. |
| `exposure`                             | `object`   | Sichtbarkeitssteuerung des Kanals für Einrichtung, konfigurierte Listen und Dokumentationsoberflächen. |
| `quickstartAllowFrom`                  | `boolean`  | Nimmt diesen Kanal in den standardmäßigen Schnellstart-`allowFrom`-Einrichtungsablauf auf. |
| `forceAccountBinding`                  | `boolean`  | Erfordert eine explizite Kontobindung, selbst wenn nur ein Konto vorhanden ist. |
| `preferSessionLookupForAnnounceTarget` | `boolean`  | Bevorzugt die Sitzungssuche beim Auflösen von Ankündigungszielen für diesen Kanal. |
| `setup`                                | `object`   | Serialisierbare kanaleigene Einrichtungsfelder für die verzögerte Ermittlung von CLI-Optionen. |

Beispiel:

```json
{
  "openclaw": {
    "channel": {
      "id": "my-channel",
      "label": "Mein Kanal",
      "selectionLabel": "Mein Kanal (selbst gehostet)",
      "detailLabel": "Bot für meinen Kanal",
      "docsPath": "/channels/my-channel",
      "docsLabel": "my-channel",
      "blurb": "Webhook-basierte, selbst gehostete Chat-Integration.",
      "order": 80,
      "aliases": ["mc"],
      "preferOver": ["my-channel-legacy"],
      "selectionDocsPrefix": "Anleitung:",
      "selectionExtras": ["Markdown"],
      "markdownCapable": true,
      "exposure": {
        "configured": true,
        "setup": true,
        "docs": true
      },
      "quickstartAllowFrom": true
    }
  }
}
```

`exposure` unterstützt:

- `configured`: den Kanal in konfigurierten/statusähnlichen Auflistungsoberflächen einschließen
- `setup`: den Kanal in interaktiven Einrichtungs-/Konfigurationsauswahlen einschließen
- `docs`: den Kanal in Dokumentations-/Navigationsoberflächen als öffentlich sichtbar kennzeichnen

### `openclaw.install`

`openclaw.install` sind Paketmetadaten, keine Manifestmetadaten.

| Feld                         | Typ                                 | Bedeutung                                                                         |
| ---------------------------- | ----------------------------------- | --------------------------------------------------------------------------------- |
| `clawhubSpec`                | `string`                            | Kanonische ClawHub-Spezifikation für Installations-/Aktualisierungs- und Onboarding-Abläufe mit bedarfsgesteuerter Installation. |
| `npmSpec`                    | `string`                            | Kanonische npm-Spezifikation für Ausweichabläufe bei Installation und Aktualisierung. |
| `localPath`                  | `string`                            | Lokaler Entwicklungspfad oder gebündelter Installationspfad.                      |
| `defaultChoice`              | `"clawhub"` \| `"npm"` \| `"local"` | Bevorzugte Installationsquelle, wenn mehrere Quellen verfügbar sind.              |
| `minHostVersion`             | `string`                            | Niedrigste unterstützte OpenClaw-Version, `>=x.y.z` oder `>=x.y.z-prerelease`. |
| `expectedIntegrity`          | `string`                            | Erwartete npm-Dist-Integritätszeichenfolge, üblicherweise `sha512-...`, für angeheftete Installationen. |
| `allowInvalidConfigRecovery` | `boolean`                           | Ermöglicht Abläufen zur Neuinstallation gebündelter Plugins die Wiederherstellung nach bestimmten Fehlern durch veraltete Konfigurationen. |
| `requiredPlatformPackages`   | `string[]`                          | Erforderliche plattformspezifische npm-Aliasse, die während der npm-Installation überprüft werden. |

<AccordionGroup>
  <Accordion title="Onboarding-Verhalten">
    Das interaktive Onboarding verwendet `openclaw.install` für Oberflächen zur bedarfsgesteuerten Installation: Wenn Ihr Plugin vor dem Laden der Laufzeit Provider-Authentifizierungsoptionen oder Metadaten für Kanaleinrichtung und -katalog bereitstellt, kann das Onboarding zur Installation über ClawHub, npm oder einen lokalen Pfad auffordern, das Plugin installieren oder aktivieren und anschließend den ausgewählten Ablauf fortsetzen. ClawHub-Optionen verwenden `clawhubSpec` und werden bevorzugt, sofern vorhanden; npm-Optionen erfordern vertrauenswürdige Katalogmetadaten mit einer Registry-`npmSpec` (exakte Versionen und `expectedIntegrity` sind optionale Festlegungen, die bei Installation und Aktualisierung durchgesetzt werden, sofern gesetzt). Halten Sie „was angezeigt werden soll“ in `openclaw.plugin.json` und „wie es installiert wird“ in `package.json`.
  </Accordion>
  <Accordion title="Durchsetzung von minHostVersion">
    Wenn `minHostVersion` gesetzt ist, wird die Angabe sowohl bei der Installation als auch beim Laden der Manifest-Registry für nicht gebündelte Plugins durchgesetzt. Ältere Hosts überspringen externe Plugins; ungültige Versionszeichenfolgen werden abgelehnt. Bei gebündelten Quell-Plugins wird davon ausgegangen, dass sie dieselbe Version wie der Host-Checkout haben.
  </Accordion>
  <Accordion title="Angeheftete npm-Installationen">
    Behalten Sie bei angehefteten npm-Installationen die exakte Version in `npmSpec` bei und fügen Sie die erwartete Artefaktintegrität hinzu:

    ```json
    {
      "openclaw": {
        "install": {
          "npmSpec": "@wecom/wecom-openclaw-plugin@1.2.3",
          "expectedIntegrity": "sha512-REPLACE_WITH_NPM_DIST_INTEGRITY",
          "defaultChoice": "npm"
        }
      }
    }
    ```

  </Accordion>
  <Accordion title="Geltungsbereich von allowInvalidConfigRecovery">
    `allowInvalidConfigRecovery` ist keine allgemeine Umgehung für fehlerhafte Konfigurationen. Die Option dient ausschließlich der eng begrenzten Wiederherstellung gebündelter Plugins und ermöglicht es der Neuinstallation oder Einrichtung, bekannte Überbleibsel von Aktualisierungen zu reparieren, etwa einen fehlenden Pfad zu einem gebündelten Plugin oder einen veralteten `channels.<id>`-Eintrag für dasselbe Plugin. Wenn die Konfiguration aus anderen Gründen fehlerhaft ist, schlägt die Installation weiterhin nach dem Fail-Closed-Prinzip fehl und weist den Betreiber an, `openclaw doctor --fix` auszuführen.
  </Accordion>
</AccordionGroup>

### Verzögertes vollständiges Laden

Kanal-Plugins können das verzögerte Laden wie folgt aktivieren:

```json
{
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "startup": {
      "deferConfiguredChannelFullLoadUntilAfterListen": true
    }
  }
}
```

Wenn diese Option aktiviert ist, lädt OpenClaw während der Startphase vor dem Lauschen nur `setupEntry`, auch bei bereits konfigurierten Kanälen. Der vollständige Einstiegspunkt wird geladen, nachdem der Gateway mit dem Lauschen begonnen hat.

<Warning>
Aktivieren Sie das verzögerte Laden nur, wenn Ihr `setupEntry` alles registriert, was der Gateway benötigt, bevor er mit dem Lauschen beginnt (Kanalregistrierung, HTTP-Routen, Gateway-Methoden). Wenn der vollständige Einstiegspunkt erforderliche Startfunktionen bereitstellt, behalten Sie das Standardverhalten bei.
</Warning>

Wenn Ihr Einrichtungs-/vollständiger Einstiegspunkt Gateway-RPC-Methoden registriert, verwenden Sie dafür ein Plugin-spezifisches Präfix. Reservierte zentrale Administrator-Namensräume (`config.*`, `exec.approvals.*`, `wizard.*`, `update.*`) bleiben dem Kern vorbehalten und werden immer zu `operator.admin` normalisiert.

## Plugin-Manifest

Jedes native Plugin muss eine `openclaw.plugin.json` im Paketstammverzeichnis bereitstellen. OpenClaw verwendet sie, um die Konfiguration zu validieren, ohne Plugin-Code auszuführen.

```json
{
  "id": "my-plugin",
  "name": "Mein Plugin",
  "description": "Fügt OpenClaw Funktionen von Mein Plugin hinzu",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "webhookSecret": {
        "type": "string",
        "description": "Geheimnis zur Webhook-Verifizierung"
      }
    }
  }
}
```

Fügen Sie für Kanal-Plugins `channels` hinzu (Provider-Plugins fügen `providers` hinzu):

```json
{
  "id": "my-channel",
  "channels": ["my-channel"],
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

Auch Plugins ohne Konfiguration müssen ein Schema bereitstellen. Ein leeres Schema ist gültig:

```json
{
  "id": "my-plugin",
  "configSchema": {
    "type": "object",
    "additionalProperties": false
  }
}
```

Die vollständige Schemareferenz finden Sie unter [Plugin-Manifest](/de/plugins/manifest).

## Veröffentlichung auf ClawHub

Skills und Plugin-Pakete verwenden separate ClawHub-Veröffentlichungsbefehle. Verwenden Sie für Plugin-Pakete den paketspezifischen Befehl:

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

<Note>
`clawhub skill publish <path>` ist ein anderer Befehl zum Veröffentlichen eines Skills-Ordners und nicht eines Plugin-Pakets. Siehe [Veröffentlichung auf ClawHub](/de/clawhub/publishing).
</Note>

## Einrichtungseinstiegspunkt

`setup-entry.ts` ist eine schlanke Alternative zu `index.ts`, die OpenClaw lädt, wenn nur Einrichtungsoberflächen benötigt werden (Onboarding, Konfigurationsreparatur, Prüfung deaktivierter Kanäle):

```typescript
// setup-entry.ts
import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
import { myChannelPlugin } from "./src/channel.js";

export default defineSetupPluginEntry(myChannelPlugin);
```

Dadurch wird vermieden, dass umfangreicher Laufzeitcode (Kryptografiebibliotheken, CLI-Registrierungen, Hintergrunddienste) während der Einrichtungsabläufe geladen wird.

Gebündelte Workspace-Kanäle, die einrichtungssichere Exporte in Sidecar-Modulen aufbewahren, können `defineBundledChannelSetupEntry(...)` aus `openclaw/plugin-sdk/channel-entry-contract` anstelle von `defineSetupPluginEntry(...)` verwenden. Dieser gebündelte Vertrag unterstützt außerdem einen optionalen `runtime`-Export, sodass die Laufzeitverdrahtung während der Einrichtung schlank und explizit bleiben kann.

<AccordionGroup>
  <Accordion title="Wann OpenClaw setupEntry anstelle des vollständigen Einstiegspunkts verwendet">
    - Der Kanal ist deaktiviert, benötigt jedoch Einrichtungs-/Onboarding-Oberflächen.
    - Der Kanal ist aktiviert, aber nicht konfiguriert.
    - Das verzögerte Laden ist aktiviert (`deferConfiguredChannelFullLoadUntilAfterListen`).

  </Accordion>
  <Accordion title="Was setupEntry registrieren muss">
    - Das Kanal-Plugin-Objekt (über `defineSetupPluginEntry`).
    - Alle vor dem Lauschen des Gateways erforderlichen HTTP-Routen.
    - Alle während des Starts benötigten Gateway-Methoden.

    Diese Gateway-Methoden für den Start sollten weiterhin reservierte zentrale Administrator-Namensräume wie `config.*` oder `update.*` vermeiden.

  </Accordion>
  <Accordion title="Was setupEntry NICHT enthalten sollte">
    - CLI-Registrierungen.
    - Hintergrunddienste.
    - Umfangreiche Laufzeitimporte (Kryptografie, SDKs).
    - Gateway-Methoden, die erst nach dem Start benötigt werden.

  </Accordion>
</AccordionGroup>

### Schmale Importe für Einrichtungshilfen

Bevorzugen Sie für häufig ausgeführte, ausschließlich der Einrichtung dienende Pfade die schmalen Schnittstellen für Einrichtungshilfen gegenüber dem umfassenderen `plugin-sdk/setup`-Dachmodul, wenn Sie nur einen Teil der Einrichtungsoberfläche benötigen:

| Importpfad                 | Verwendungszweck                                                                          | Wichtige Exporte                                                                                                                                                                                                                                                                                                      |
| -------------------------- | ----------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/setup-runtime` | Laufzeithilfen für die Einrichtung, die in `setupEntry` / beim verzögerten Kanalstart verfügbar bleiben | `createSetupTranslator`, `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
| `plugin-sdk/setup-tools`   | Hilfen für Einrichtungs-/Installations-CLI, Archive und Dokumentation                     | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR`                                                                                                                                                                                                         |

Verwenden Sie die umfassendere `plugin-sdk/setup`-Schnittstelle, wenn Sie den vollständigen gemeinsamen Einrichtungswerkzeugkasten benötigen, einschließlich Hilfen für Konfigurations-Patches wie `moveSingleAccountChannelSectionToDefaultAccount(...)`.

Verwenden Sie `createSetupTranslator(...)` für feste Texte des Einrichtungsassistenten. Dabei wird der erste nicht leere Wert aus `OPENCLAW_LOCALE`, `LC_ALL`, `LC_MESSAGES` und `LANG` in dieser Reihenfolge verwendet; anschließend wird auf Englisch zurückgegriffen. Setzen Sie `OPENCLAW_LOCALE=en`, um Englisch ausdrücklich zu erzwingen. Bewahren Sie Plugin-spezifische Einrichtungstexte im Plugin-eigenen Code auf und verwenden Sie gemeinsame Katalogschlüssel nur für allgemeine Einrichtungsbeschriftungen, Statustexte und offizielle Einrichtungstexte gebündelter Plugins.

Die Adapter für Einrichtungs-Patches bleiben beim Import für häufig ausgeführte Pfade geeignet. Die Suche nach der Vertragsoberfläche für die gebündelte Heraufstufung eines Einzelkontos erfolgt verzögert, sodass der Import von `plugin-sdk/setup-runtime` die Erkennung gebündelter Vertragsoberflächen nicht vorzeitig lädt, bevor der Adapter tatsächlich verwendet wird.

### Kanaleigene Eingabefelder für die Einrichtung

`ChannelSetupInput` ist ein generischer Umschlag, den Einrichtungsaufrufer und Kanal-
Plugins gemeinsam verwenden. Seine dauerhaft typisierten Felder sind `name`, `token`, `tokenFile`,
`useEnv`, `allowFrom` und `defaultTo`. Zusätzliche Plugin-eigene Schlüssel können weiterhin
im Laufzeiteingabeobjekt vorhanden sein, der gemeinsame Typ deklariert jedoch keine
Indexsignatur. Jedes Plugin muss seine eigenen Einrichtungsfelder deklarieren und eingrenzen oder
sie an der Adaptergrenze mit einem Plugin-eigenen Schema validieren:

```typescript
import type { ChannelSetupAdapter, ChannelSetupInput } from "openclaw/plugin-sdk/channel-setup";

type AcmeSetupInput = ChannelSetupInput & {
  workspaceId?: string;
  webhookUrl?: string;
};

export const acmeSetupAdapter: ChannelSetupAdapter = {
  applyAccountConfig: ({ cfg, input }) => {
    const setupInput = input as AcmeSetupInput;
    return {
      ...cfg,
      channels: {
        ...cfg.channels,
        acme: {
          token: setupInput.token,
          workspaceId: setupInput.workspaceId,
          webhookUrl: setupInput.webhookUrl,
        },
      },
    };
  },
};
```

Kanalspezifische Felder, die zuvor direkt auf
`ChannelSetupInput` deklariert wurden, bleiben vorübergehend typisiert, um die Kompatibilität mit externem Quellcode zu gewährleisten.
Sie sind veraltet. Bei einer Registry-Überprüfung am 2026-07-22 von 426 veröffentlichten, außerhalb des Repositorys verwalteten
Kanal-Plugins wurden 21 Felder ohne Leser entfernt und 22 mit bekannten
Lesern beibehalten. Jedes beibehaltene Feld wird gelöscht, sobald kein veröffentlichtes Plugin es mehr liest;
eine Versionsgrenze ist nicht erforderlich. Neue und gebündelte Plugins dürfen sich nicht auf diese
Ebene verlassen; deklarieren Sie die Felder, deren Eigentümer sie sind, lokal.

### Kanaleigene Überführung eines Einzelkontos

Wenn ein Kanal von einer Einzelkonto-Konfiguration auf oberster Ebene auf `channels.<id>.accounts.*` umgestellt wird, verschiebt das standardmäßige gemeinsame Verhalten die überführten kontobezogenen Werte nach `accounts.default`.

Jedes Kanal-Plugin kann diese Überführung über seinen Setup-Adapter erweitern oder einschränken:

- `singleAccountKeysToMove`: zusätzliche Schlüssel auf oberster Ebene, die in das überführte Konto verschoben werden sollen
- `namedAccountPromotionKeys`: wenn bereits benannte Konten vorhanden sind, werden nur diese Schlüssel in das überführte Konto verschoben; gemeinsame Richtlinien-/Zustellungsschlüssel verbleiben im Kanalstamm
- `resolveSingleAccountPromotionTarget(...)`: legt fest, welches bestehende Konto die überführten Werte erhält

Das Vorhandensein von `singleAccountKeysToMove` kennzeichnet den Überführungsvertrag als vollständig. Deklarieren Sie das Feld auch dann, wenn es sich um ein leeres Array handelt, um die Überführung veralteter Schlüssel zu deaktivieren. Adapter, die das Feld auslassen, behalten für bereits veröffentlichte Plugins eine lesergestützte Überführungsebene aus der Zeit vor der Deklaration bei. Bei der Registry-Überprüfung am 2026-07-22 wurden 23 Schlüssel ohne veröffentlichte Abhängige entfernt und sechs gängige Schlüssel sowie der ausschließlich für das Setup verwendete Schlüssel `rooms` beibehalten. Jeder beibehaltene Schlüssel wird gelöscht, sobald seine veröffentlichten Leser zu Deklarationen migriert wurden; eine Versionsgrenze ist nicht erforderlich.

Deklarieren Sie `openclaw.setupFeatures.configPromotion: true` im Paketmanifest des Plugins, wenn Doctor diese Deklarationen aus dem schlanken gebündelten Setup-Artefakt laden muss. Die ausschließlich für das Setup vorgesehene Plugin-Oberfläche und das vollständige Kanal-Plugin müssen dieselben Deklarationen bereitstellen.

Wenn Sie `moveSingleAccountChannelSectionToDefaultAccount(...)` mit einem bereits aufgelösten Plugin aufrufen, übergeben Sie dessen Setup-Adapter als `setupSurface`. Vom Aufrufer bereitgestellte Setup-Oberflächen haben Vorrang vor geladenen und gebündelten Suchmechanismen, wodurch bereichsgebundene oder ausschließlich für das Setup vorgesehene Plugins unabhängig von der globalen Registrierung bleiben.

<Note>
Matrix ist das aktuelle gebündelte Beispiel. Wenn genau ein benanntes Matrix-Konto bereits vorhanden ist oder wenn `defaultAccount` auf einen vorhandenen nicht kanonischen Schlüssel wie `Ops` verweist, behält die Überführung dieses Konto bei, anstatt einen neuen Eintrag `accounts.default` zu erstellen.
</Note>

## Konfigurationsschema

Die Plugin-Konfiguration wird anhand des JSON-Schemas in Ihrem Manifest validiert. Benutzer konfigurieren Plugins über:

```json5
{
  plugins: {
    entries: {
      "my-plugin": {
        config: {
          webhookSecret: "abc123",
        },
      },
    },
  },
}
```

Ihr Plugin erhält diese Konfiguration während der Registrierung als `api.pluginConfig`.

Verwenden Sie für kanalspezifische Konfigurationen stattdessen den Abschnitt für die Kanalkonfiguration:

```json5
{
  channels: {
    "my-channel": {
      token: "bot-token",
      allowFrom: ["user1", "user2"],
    },
  },
}
```

### Erstellen von Schemas für Kanalkonfigurationen

Verwenden Sie `buildChannelConfigSchema`, um ein Zod-Schema in den von Plugin-eigenen Konfigurationsartefakten verwendeten `ChannelConfigSchema`-Wrapper umzuwandeln:

```typescript
import { z } from "zod";
import { buildChannelConfigSchema } from "openclaw/plugin-sdk/channel-config-schema";

const accountSchema = z.object({
  token: z.string().optional(),
  allowFrom: z.array(z.string()).optional(),
  accounts: z.object({}).catchall(z.any()).optional(),
  defaultAccount: z.string().optional(),
});

const configSchema = buildChannelConfigSchema(accountSchema);
```

Wenn Sie den Vertrag bereits als JSON-Schema oder TypeBox erstellen, verwenden Sie den direkten Helfer, damit OpenClaw die Konvertierung von Zod in JSON-Schema auf Metadatenpfaden überspringen kann:

```typescript
import { Type } from "typebox";
import { buildJsonChannelConfigSchema } from "openclaw/plugin-sdk/channel-config-schema";

const configSchema = buildJsonChannelConfigSchema(
  Type.Object({
    token: Type.Optional(Type.String()),
    allowFrom: Type.Optional(Type.Array(Type.String())),
  }),
);
```

Für Drittanbieter-Plugins bleibt das Plugin-Manifest der Vertrag für den Kaltpfad: Spiegeln Sie das generierte JSON-Schema in `openclaw.plugin.json#channelConfigs`, damit Konfigurationsschema-, Setup- und UI-Oberflächen `channels.<id>` untersuchen können, ohne Laufzeitcode zu laden.

## Setup-Assistenten

Kanal-Plugins können interaktive Setup-Assistenten für `openclaw onboard` bereitstellen. Der Assistent ist ein `ChannelSetupWizard`-Objekt auf dem `ChannelPlugin`:

```typescript
import type { ChannelSetupWizard } from "openclaw/plugin-sdk/channel-setup";

const setupWizard: ChannelSetupWizard = {
  channel: "my-channel",
  status: {
    configuredLabel: "Connected",
    unconfiguredLabel: "Not configured",
    resolveConfigured: ({ cfg }) => Boolean((cfg.channels as any)?.["my-channel"]?.token),
  },
  credentials: [
    {
      inputKey: "token",
      providerHint: "my-channel",
      credentialLabel: "Bot token",
      preferredEnvVar: "MY_CHANNEL_BOT_TOKEN",
      envPrompt: "Use MY_CHANNEL_BOT_TOKEN from environment?",
      keepPrompt: "Keep current token?",
      inputPrompt: "Enter your bot token:",
      inspect: ({ cfg, accountId }) => {
        const token = (cfg.channels as any)?.["my-channel"]?.token;
        return {
          accountConfigured: Boolean(token),
          hasConfiguredValue: Boolean(token),
        };
      },
    },
  ],
};
```

`ChannelSetupWizard` unterstützt außerdem `textInputs`, `dmPolicy`, `allowFrom`, `groupAccess`, `prepare`, `finalize` und mehr. Ein vollständiges gebündeltes Beispiel finden Sie unter `src/setup-core.ts` des Discord-Plugins.

<AccordionGroup>
  <Accordion title="Gemeinsame allowFrom-Eingabeaufforderungen">
    Verwenden Sie für Eingabeaufforderungen zu DM-Zulassungslisten, die nur den standardmäßigen Ablauf `note -> prompt -> parse -> merge -> patch` benötigen, vorzugsweise die gemeinsamen Setup-Helfer `createPromptParsedAllowFromForAccount(...)` und `createTopLevelChannelParsedAllowFromPrompt(...)` aus `openclaw/plugin-sdk/setup`.
  </Accordion>
  <Accordion title="Standardstatus der Kanaleinrichtung">
    Verwenden Sie für Statusblöcke der Kanaleinrichtung, die sich nur durch Beschriftungen, Bewertungen und optionale zusätzliche Zeilen unterscheiden, vorzugsweise `createStandardChannelSetupStatus(...)` aus `openclaw/plugin-sdk/setup`, anstatt dasselbe `status`-Objekt in jedem Plugin manuell zu erstellen.
  </Accordion>
  <Accordion title="Optionale Oberfläche zur Kanaleinrichtung">
    Verwenden Sie für optionale Setup-Oberflächen, die nur in bestimmten Kontexten angezeigt werden sollen, `createOptionalChannelSetupSurface` aus `openclaw/plugin-sdk/channel-setup`:

    ```typescript
    import { createOptionalChannelSetupSurface } from "openclaw/plugin-sdk/channel-setup";

    const setupSurface = createOptionalChannelSetupSurface({
      channel: "my-channel",
      label: "My Channel",
      npmSpec: "@myorg/openclaw-my-channel",
      docsPath: "/channels/my-channel",
    });
    // Returns { setupAdapter, setupWizard }
    ```

    `plugin-sdk/channel-setup` stellt außerdem die untergeordneten Builder `createOptionalChannelSetupAdapter(...)` und `createOptionalChannelSetupWizard(...)` bereit, wenn Sie nur eine Hälfte dieser optionalen Installationsoberfläche benötigen.

    Der generierte optionale Adapter/Assistent schlägt bei tatsächlichen Schreibvorgängen der Konfiguration sicher geschlossen fehl. Für `validateInput`, `applyAccountConfig` und `finalize` wird dieselbe Meldung über die erforderliche Installation wiederverwendet; außerdem wird ein Dokumentationslink angehängt, wenn `docsPath` gesetzt ist.

  </Accordion>
  <Accordion title="Binärdateigestützte Setup-Helfer">
    Verwenden Sie für binärdateigestützte Setup-UIs vorzugsweise die gemeinsamen delegierten Helfer, anstatt dieselbe Verknüpfungslogik für Binärdatei und Status in jeden Kanal zu kopieren:

    - `createDetectedBinaryStatus(...)` für Statusblöcke, die sich nur durch Beschriftungen, Hinweise, Bewertungen und Binärdateierkennung unterscheiden
    - `createCliPathTextInput(...)` für pfadgestützte Texteingaben
    - `createDelegatedSetupWizardProxy(...)`, wenn `setupEntry` Status-, Vorbereitungs- oder Abschlussverhalten verzögert an einen umfangreicheren vollständigen Assistenten weiterleiten muss
    - `createDelegatedTextInputShouldPrompt(...)`, wenn `setupEntry` lediglich eine `textInputs[*].shouldPrompt`-Entscheidung delegieren muss

  </Accordion>
</AccordionGroup>

## Veröffentlichen und Installieren

**Externe Plugins:** Veröffentlichen Sie sie auf [ClawHub](/de/clawhub) und installieren Sie sie anschließend:

<Tabs>
  <Tab title="npm">
    ```bash
    openclaw plugins install @myorg/openclaw-my-plugin
    ```

    Reine Paketspezifikationen werden während der Umstellung beim Start von npm installiert, es sei denn, der Name entspricht einer gebündelten oder offiziellen Plugin-ID; in diesem Fall verwendet OpenClaw stattdessen die lokale/offizielle Kopie. Verwenden Sie `clawhub:`, `npm:`, `git:` oder `npm-pack:` für eine deterministische Quellenauswahl – siehe [Plugins verwalten](/de/plugins/manage-plugins).

  </Tab>
  <Tab title="Nur ClawHub">
    ```bash
    openclaw plugins install clawhub:@myorg/openclaw-my-plugin
    ```
  </Tab>
  <Tab title="npm-Paketspezifikation">
    Verwenden Sie npm, wenn ein Paket noch nicht zu ClawHub verschoben wurde oder wenn Sie während der Migration einen
    direkten npm-Installationspfad benötigen:

    ```bash
    openclaw plugins install npm:@myorg/openclaw-my-plugin
    ```

  </Tab>
</Tabs>

**Plugins im Repository:** Platzieren Sie sie im gebündelten Plugin-Workspace-Baum; sie werden während des Builds automatisch erkannt.

<Info>
Bei Installationen aus npm installiert `openclaw plugins install` das Paket in einem Plugin-spezifischen Projekt unter `~/.openclaw/npm/projects`, wobei Lebenszyklusskripte deaktiviert sind (`--ignore-scripts`). Halten Sie Plugin-Abhängigkeitsbäume auf reines JS/TS beschränkt und vermeiden Sie Pakete, die `postinstall`-Builds erfordern.
</Info>

<Note>
Beim Start des Gateway werden keine Plugin-Abhängigkeiten installiert. Die Installationsabläufe von npm/git/ClawHub sind für die Konvergenz der Abhängigkeiten verantwortlich; bei lokalen Plugins müssen die Abhängigkeiten bereits installiert sein.
</Note>

Gebündelte Paketmetadaten werden explizit angegeben und beim Start des Gateway nicht aus dem erstellten JavaScript abgeleitet. Laufzeitabhängigkeiten gehören in das Plugin-Paket, dessen Eigentümer sie sind; der Start einer paketierten OpenClaw-Installation repariert oder spiegelt niemals Plugin-Abhängigkeiten.

## Verwandte Themen

- [Plugins erstellen](/de/plugins/building-plugins) — schrittweise Einführung
- [Plugin-Manifest](/de/plugins/manifest) — vollständige Referenz zum Manifestschema
- [SDK-Einstiegspunkte](/de/plugins/sdk-entrypoints) — `definePluginEntry` und `defineChannelPluginEntry`
