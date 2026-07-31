---
read_when:
    - Sie haben `clawhub package validate` ausgeführt und müssen Plugin-Befunde beheben
    - ClawHub hat die Veröffentlichung eines Plugin-Pakets abgelehnt oder eine Warnung ausgegeben
    - Sie aktualisieren vor der Veröffentlichung die Metadaten des Plugin-Pakets
summary: Beheben Sie vor der Veröffentlichung die bei der Validierung von ClawHub-Plugin-Paketen festgestellten Probleme
title: Fehlerbehebungen bei der Plugin-Validierung
x-i18n:
    generated_at: "2026-07-26T18:20:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2cb869e41c9a9f1c0725f514f6b48095eb3838bf61aaf06c2474a18192f0e819
    source_path: clawhub/plugin-validation-fixes.md
    workflow: 16
---

# Fehlerbehebung bei der Plugin-Validierung

ClawHub validiert Plugin-Pakete vor der Veröffentlichung und kann auch Befunde aus
automatisierten Paket-Scans anzeigen. Diese Seite behandelt Befunde für Autoren, also
Befunde, die Plugin-Autoren in ihren Paketmetadaten, im Manifest, in SDK-
Importen oder im veröffentlichten Artefakt beheben können.

Interne Befunde zur Abdeckung durch den Plugin Inspector werden nicht behandelt. Wenn ein vollständiger Bericht
Wartungscodes des Scanners ohne Hinweise zur Behebung durch Autoren enthält, sind diese
für OpenClaw-Maintainer und nicht für Plugin-Autoren bestimmt.

Führen Sie nach jeder Korrektur erneut Folgendes aus:

```bash
clawhub package validate <path-to-plugin>
```

## Befunde für Autoren

| Code                                    | Einstieg                                                                                                                    |
| --------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| `package-json-missing`                  | [Paketmetadaten hinzufügen](/de/clawhub/plugin-validation-fixes#package-json-missing)                                              |
| `package-openclaw-metadata-missing`     | [Den OpenClaw-Block des Pakets hinzufügen](/de/clawhub/plugin-validation-fixes#package-openclaw-metadata-missing)                   |
| `package-openclaw-entry-missing`        | [OpenClaw-Paketeinstiegspunkte deklarieren](/de/clawhub/plugin-validation-fixes#package-openclaw-entry-missing)                     |
| `package-entrypoint-missing`            | [Den deklarierten Einstiegspunkt veröffentlichen](/de/clawhub/plugin-validation-fixes#package-entrypoint-missing)                   |
| `package-install-metadata-incomplete`   | [Installationsmetadaten vervollständigen](/de/clawhub/plugin-validation-fixes#package-install-metadata-incomplete)                  |
| `package-plugin-api-compat-missing`     | [Kompatibilität der Plugin-API deklarieren](/de/clawhub/plugin-validation-fixes#package-plugin-api-compat-missing)                  |
| `package-min-host-version-drift`        | [Minimale Hostversion angleichen](/de/clawhub/plugin-validation-fixes#package-min-host-version-drift)                               |
| `package-manifest-version-drift`        | [Paket- und Manifestversionen angleichen](/de/clawhub/plugin-validation-fixes#package-manifest-version-drift)                       |
| `package-openclaw-unsupported-metadata` | [Nicht unterstützte OpenClaw-Paketmetadaten entfernen](/de/clawhub/plugin-validation-fixes#package-openclaw-unsupported-metadata)   |
| `package-npm-pack-unavailable`          | [Das npm-Artefakt packbar machen](/de/clawhub/plugin-validation-fixes#package-npm-pack-unavailable)                                 |
| `package-npm-pack-entrypoint-missing`   | [Einstiegspunkte in die npm-Packausgabe aufnehmen](/de/clawhub/plugin-validation-fixes#package-npm-pack-entrypoint-missing)          |
| `package-npm-pack-metadata-missing`     | [Metadaten in die npm-Packausgabe aufnehmen](/de/clawhub/plugin-validation-fixes#package-npm-pack-metadata-missing)                  |
| `manifest-name-missing`                 | [Einen Anzeigenamen zum Manifest hinzufügen](/de/clawhub/plugin-validation-fixes#manifest-name-missing)                            |
| `manifest-unknown-fields`               | [Nicht unterstützte Manifestfelder entfernen](/de/clawhub/plugin-validation-fixes#manifest-unknown-fields)                          |
| `manifest-unknown-contracts`            | [Nicht unterstützte Vertragsschlüssel entfernen](/de/clawhub/plugin-validation-fixes#manifest-unknown-contracts)                    |
| `legacy-root-sdk-import`                | [SDK-Importe aus dem Stamm ersetzen](/de/clawhub/plugin-validation-fixes#legacy-root-sdk-import)                                    |
| `reserved-sdk-import`                   | [Reservierte SDK-Importe entfernen](/de/clawhub/plugin-validation-fixes#reserved-sdk-import)                                       |
| `sdk-load-session-store`                | [Zugriff auf den gesamten Sitzungsspeicher ersetzen](/de/clawhub/plugin-validation-fixes#sdk-load-session-store)                    |
| `sdk-session-store-write`               | [Schreibvorgänge in den gesamten Sitzungsspeicher ersetzen](/de/clawhub/plugin-validation-fixes#sdk-session-store-write)            |
| `sdk-session-file-helper`               | [Hilfsfunktionen für Sitzungspfade ersetzen](/de/clawhub/plugin-validation-fixes#sdk-session-file-helper)                           |
| `sdk-session-transcript-file-target`    | [Veraltete Ziele für Transkriptdateien ersetzen](/de/clawhub/plugin-validation-fixes#sdk-session-transcript-file-target)            |
| `sdk-session-transcript-low-level`      | [Low-Level-Hilfsfunktionen für Transkripte ersetzen](/de/clawhub/plugin-validation-fixes#sdk-session-transcript-low-level)           |
| `legacy-before-agent-start`             | [before_agent_start ersetzen](/de/clawhub/plugin-validation-fixes#legacy-before-agent-start)                                       |
| `provider-auth-env-vars`                | [Provider-Umgebungsvariablen in Setup-Metadaten verschieben](/de/clawhub/plugin-validation-fixes#provider-auth-env-vars)            |
| `channel-env-vars`                      | [Kanal-Umgebungsvariablen in aktuellen Metadaten spiegeln](/de/clawhub/plugin-validation-fixes#channel-env-vars)              |
| `security-manifest-schema-unavailable`  | [Nicht verfügbare Schema-Referenzen für Sicherheitsmanifeste entfernen](/de/clawhub/plugin-validation-fixes#security-manifest-schema-unavailable) |
| `unrecognized-security-manifest`        | [Nicht unterstützte Sicherheitsmanifestdateien entfernen](/de/clawhub/plugin-validation-fixes#unrecognized-security-manifest)      |

## Paketmetadaten

### package-json-missing

Das Stammverzeichnis des Pakets enthält kein `package.json`, sodass ClawHub das
npm-Paket, seine Version, seine Einstiegspunkte oder seine OpenClaw-Metadaten nicht identifizieren kann.

- Fügen Sie `package.json` mit `name`, `version` und `type` hinzu.
- Fügen Sie einen `openclaw`-Block hinzu, wenn das Paket ein OpenClaw-Plugin ausliefert.
- Unter [Plugins erstellen](/de/plugins/building-plugins) finden Sie ein minimales Paketbeispiel
  und unter [Plugin-Manifest](/de/plugins/manifest#manifest-versus-packagejson)
  die Aufteilung zwischen Paket und Manifest.
- Führen Sie `clawhub package validate <path-to-plugin>` erneut aus.

### package-openclaw-metadata-missing

Das Paket enthält `package.json`, deklariert aber keine OpenClaw-
Paketmetadaten.

- Fügen Sie `package.json#openclaw` hinzu.
- Nehmen Sie Einstiegspunktmetadaten wie `openclaw.extensions` oder
  `openclaw.runtimeExtensions` auf.
- Fügen Sie Kompatibilitäts- und Installationsmetadaten hinzu, wenn das Paket über ClawHub veröffentlicht oder
  installiert werden soll.
- Siehe [package.json-Felder, die die Erkennung beeinflussen](/de/plugins/manifest#packagejson-fields-that-affect-discovery).
- Führen Sie `clawhub package validate <path-to-plugin>` erneut aus.

### package-openclaw-entry-missing

Die Paketmetadaten sind vorhanden, deklarieren aber keinen OpenClaw-Laufzeit-
Einstiegspunkt.

- Fügen Sie `openclaw.extensions` für native Plugin-Einstiegspunkte hinzu.
- Fügen Sie `openclaw.runtimeExtensions` hinzu, wenn das veröffentlichte Paket kompiliertes
  JavaScript laden soll.
- Belassen Sie alle Einstiegspunktpfade innerhalb des Paketverzeichnisses.
- Siehe [Plugin-Einstiegspunkte](/de/plugins/sdk-entrypoints) und
  [package.json-Felder, die die Erkennung beeinflussen](/de/plugins/manifest#packagejson-fields-that-affect-discovery).
- Führen Sie `clawhub package validate <path-to-plugin>` erneut aus.

### package-entrypoint-missing

Das Paket deklariert einen OpenClaw-Einstiegspunkt, aber die referenzierte Datei fehlt
im validierten Paket.

- Prüfen Sie jeden Pfad in `openclaw.extensions`, `openclaw.runtimeExtensions`,
  `openclaw.setupEntry` und `openclaw.runtimeSetupEntry`.
- Erstellen Sie das Paket, wenn der Einstiegspunkt in `dist` generiert wird.
- Aktualisieren Sie die Metadaten, wenn der Einstiegspunkt verschoben wurde.
- Siehe [Plugin-Einstiegspunkte](/de/plugins/sdk-entrypoints).
- Führen Sie `clawhub package validate <path-to-plugin>` erneut aus.

### package-install-metadata-incomplete

ClawHub kann nicht feststellen, wie das Paket installiert oder aktualisiert werden soll.

- Füllen Sie `openclaw.install` mit der unterstützten Installationsquelle aus, beispielsweise
  `clawhubSpec`, `npmSpec` oder `localPath`.
- Legen Sie `openclaw.install.defaultChoice` fest, wenn mehr als eine Installationsquelle
  verfügbar ist.
- Verwenden Sie `openclaw.install.minHostVersion` für die minimale OpenClaw-Hostversion.
- Siehe [package.json-Felder, die die Erkennung beeinflussen](/de/plugins/manifest#packagejson-fields-that-affect-discovery).
- Führen Sie `clawhub package validate <path-to-plugin>` erneut aus.

### package-plugin-api-compat-missing

Das Paket deklariert nicht den unterstützten Versionsbereich der OpenClaw-Plugin-API.

- Fügen Sie `openclaw.compat.pluginApi` zu `package.json` hinzu.
- Verwenden Sie die Version der OpenClaw-Plugin-API oder die SemVer-Untergrenze, mit der Sie das Paket erstellt und
  getestet haben.
- Halten Sie dies von der Paketversion getrennt. Die Paketversion beschreibt das
  Plugin-Release; `openclaw.compat.pluginApi` beschreibt den Host-API-Vertrag.
- Siehe [package.json-Felder, die die Erkennung beeinflussen](/de/plugins/manifest#packagejson-fields-that-affect-discovery).
- Führen Sie `clawhub package validate <path-to-plugin>` erneut aus.

### package-min-host-version-drift

Die minimale Hostversion des Pakets stimmt nicht mit den OpenClaw-Versionsmetadaten überein,
mit denen das Paket erstellt wurde.

- Prüfen Sie `openclaw.install.minHostVersion`.
- Prüfen Sie alle OpenClaw-Build-Metadaten im Paket, beispielsweise die während des Releases
  verwendete OpenClaw-Version.
- Gleichen Sie die minimale Hostversion an den Hostversionsbereich an, den das Paket
  tatsächlich unterstützt.
- Siehe [package.json-Felder, die die Erkennung beeinflussen](/de/plugins/manifest#packagejson-fields-that-affect-discovery).
- Führen Sie `clawhub package validate <path-to-plugin>` erneut aus.

### package-manifest-version-drift

Die Paketversion und die Version des Plugin-Manifests stimmen nicht überein.

- Verwenden Sie vorzugsweise `package.json#version` als Release-Version des Pakets.
- Wenn `openclaw.plugin.json` ebenfalls `version` enthält, aktualisieren Sie es entsprechend oder entfernen Sie
  veraltete Versionsmetadaten aus dem Manifest, wenn die Paketmetadaten maßgeblich sind.
- Veröffentlichen Sie nach einer Änderung veröffentlichter Metadaten eine neue Paketversion.
- Siehe [Plugin-Manifest](/de/plugins/manifest).
- Führen Sie `clawhub package validate <path-to-plugin>` erneut aus.

### package-openclaw-unsupported-metadata

Der `package.json#openclaw`-Block enthält Felder, die nicht als
OpenClaw-Paketmetadaten unterstützt werden.

- Entfernen Sie nicht unterstützte Felder wie `openclaw.bundle`.
- Belassen Sie native Plugin-Metadaten in `openclaw.plugin.json`.
- Belassen Sie Paketeinstiegspunkte sowie Kompatibilitäts-, Installations-, Setup- und Katalogmetadaten
  in unterstützten `package.json#openclaw`-Feldern.
- Siehe [package.json-Felder, die die Erkennung beeinflussen](/de/plugins/manifest#packagejson-fields-that-affect-discovery).
- Führen Sie `clawhub package validate <path-to-plugin>` erneut aus.

## Veröffentlichtes Artefakt

### package-npm-pack-unavailable

Das Paket kann nicht in das Artefakt gepackt werden, das ClawHub prüfen oder
veröffentlichen würde.

- Führen Sie `npm pack --dry-run` im Stammverzeichnis des Pakets aus.
- Beheben Sie ungültige Paketmetadaten, fehlerhafte Lebenszyklusskripte oder Dateieinträge, die
  das Packen verhindern.
- Entfernen Sie `private: true`, wenn dieses Paket für eine öffentliche Veröffentlichung vorgesehen ist.
- Führen Sie `clawhub package validate <path-to-plugin>` erneut aus.

### package-npm-pack-entrypoint-missing

Das Paket kann gepackt werden, aber das gepackte Artefakt enthält nicht die
in `package.json#openclaw` deklarierten Einstiegspunktdateien.

- Führen Sie `npm pack --dry-run` aus und prüfen Sie die Dateien, die aufgenommen würden.
- Erstellen Sie generierte Einstiegspunkte vor dem Packen.
- Aktualisieren Sie `files`, `.npmignore` oder die Build-Ausgabe, damit deklarierte Einstiegspunkte
  aufgenommen werden.
- Siehe [Plugin-Einstiegspunkte](/de/plugins/sdk-entrypoints).
- Führen Sie `clawhub package validate <path-to-plugin>` erneut aus.

### package-npm-pack-metadata-missing

Im gepackten Artefakt fehlen OpenClaw-Metadaten, die in Ihrem Quellpaket
vorhanden sind.

- Führen Sie `npm pack --dry-run` aus und prüfen Sie die enthaltenen Metadatendateien.
- Stellen Sie sicher, dass `package.json` den Block `openclaw` im gepackten Artefakt enthält.
- Stellen Sie sicher, dass `openclaw.plugin.json` enthalten ist, wenn das Paket ein natives
  OpenClaw-Plugin ist.
- Aktualisieren Sie `files` oder `.npmignore`, damit Paketmetadaten nicht ausgeschlossen werden.
- Siehe [Plugins erstellen](/de/plugins/building-plugins).
- Führen Sie `clawhub package validate <path-to-plugin>` erneut aus.

## Manifestmetadaten

### manifest-name-missing

Das native Plugin-Manifest enthält keinen Anzeigenamen.

- Fügen Sie `openclaw.plugin.json` ein nicht leeres Feld `name` hinzu.
- Halten Sie `name` für Menschen lesbar und verwenden Sie `id` weiterhin als stabile Maschinen-ID.
- Siehe [Plugin-Manifest](/de/plugins/manifest).
- Führen Sie `clawhub package validate <path-to-plugin>` erneut aus.

### manifest-unknown-fields

Das Plugin-Manifest enthält Felder auf oberster Ebene, die OpenClaw nicht unterstützt.

- Vergleichen Sie jedes Feld auf oberster Ebene mit der
  [Referenz der Manifestfelder](/de/plugins/manifest#top-level-field-reference).
- Entfernen Sie benutzerdefinierte Felder aus `openclaw.plugin.json`.
- Verschieben Sie Paket- oder Installationsmetadaten in unterstützte Felder von `package.json#openclaw`
  statt in das Manifest.
- Führen Sie `clawhub package validate <path-to-plugin>` erneut aus.

### manifest-unknown-contracts

Das Manifest deklariert nicht unterstützte Schlüssel innerhalb von `contracts`.

- Vergleichen Sie jeden Schlüssel unter `contracts` mit der
  [Referenz der Verträge](/de/plugins/manifest#contracts-reference).
- Entfernen Sie nicht unterstützte Vertragsschlüssel.
- Verschieben Sie das Laufzeitverhalten in den Plugin-Registrierungscode und beschränken Sie `contracts`
  auf statische Metadaten zur Zuständigkeit für Funktionen.
- Führen Sie `clawhub package validate <path-to-plugin>` erneut aus.

## SDK- und Kompatibilitätsmigration

### legacy-root-sdk-import

Das Plugin importiert aus dem veralteten SDK-Root-Barrel:
`openclaw/plugin-sdk`.

- Ersetzen Sie Importe aus dem Root-Barrel durch gezielte Importe aus öffentlichen Unterpfaden.
- Verwenden Sie `openclaw/plugin-sdk/plugin-entry` für `definePluginEntry`.
- Verwenden Sie `openclaw/plugin-sdk/channel-core` für Hilfsfunktionen von Kanaleinstiegspunkten.
- Verwenden Sie [Importkonventionen](/de/plugins/building-plugins#import-conventions) und
  [Unterpfade des Plugin-SDKs](/de/plugins/sdk-subpaths), um den spezifischen Import zu finden.
- Führen Sie `clawhub package validate <path-to-plugin>` erneut aus.

### reserved-sdk-import

Das Plugin importiert einen SDK-Pfad, der gebündelten Plugins oder der internen
Kompatibilität vorbehalten ist.

- Ersetzen Sie reservierte interne OpenClaw-SDK-Importe durch dokumentierte öffentliche
  Unterpfade von `openclaw/plugin-sdk/*`.
- Wenn für das Verhalten kein öffentliches SDK verfügbar ist, behalten Sie die Hilfsfunktion in Ihrem Paket oder
  fordern Sie eine öffentliche OpenClaw-API an.
- Verwenden Sie [Unterpfade des Plugin-SDKs](/de/plugins/sdk-subpaths) und
  [SDK-Migration](/de/plugins/sdk-migration), um einen unterstützten Import auszuwählen.
- Führen Sie `clawhub package validate <path-to-plugin>` erneut aus.

### sdk-load-session-store

Das Plugin verwendet weiterhin die veraltete Hilfsfunktion für den gesamten Sitzungsspeicher
`loadSessionStore`.

- Verwenden Sie `getSessionEntry(...)` oder `listSessionEntries(...)`, wenn Sie den Sitzungszustand
  lesen.
- Verwenden Sie `patchSessionEntry(...)` oder `upsertSessionEntry(...)`, wenn Sie den Sitzungszustand
  schreiben.
- Vermeiden Sie es, das gesamte Sitzungsspeicherobjekt zu laden, zu verändern und zu speichern.
- Behalten Sie `loadSessionStore(...)` nur bei, solange Ihr deklarierter Kompatibilitätsbereich
  noch ältere OpenClaw-Versionen unterstützt, die diese Hilfsfunktion benötigen.
- Siehe [Laufzeit-API](/de/plugins/sdk-runtime#agent-session-state) und
  [Unterpfade des Plugin-SDKs](/de/plugins/sdk-subpaths).
- Führen Sie `clawhub package validate <path-to-plugin>` erneut aus.

### sdk-session-store-write

Das Plugin verwendet weiterhin eine veraltete Schreibhilfsfunktion für den gesamten Sitzungsspeicher, beispielsweise
`saveSessionStore` oder `updateSessionStore`.

- Verwenden Sie `patchSessionEntry(...)`, wenn Sie Felder eines vorhandenen Sitzungseintrags
  aktualisieren.
- Verwenden Sie `upsertSessionEntry(...)`, wenn Sie einen Sitzungseintrag ersetzen oder erstellen.
- Vermeiden Sie es, das gesamte Sitzungsspeicherobjekt zu laden, zu verändern und zu speichern.
- Behalten Sie Schreibhilfsfunktionen für den gesamten Speicher nur bei, solange Ihr deklarierter Kompatibilitätsbereich
  noch ältere OpenClaw-Versionen unterstützt, die sie benötigen.
- Siehe [Laufzeit-API](/de/plugins/sdk-runtime#agent-session-state) und
  [Unterpfade des Plugin-SDKs](/de/plugins/sdk-subpaths).
- Führen Sie `clawhub package validate <path-to-plugin>` erneut aus.

### sdk-session-file-helper

Das Plugin verwendet weiterhin veraltete Hilfsfunktionen für Sitzungspfade, beispielsweise
`resolveSessionFilePath` oder `resolveAndPersistSessionFile`.

- Verwenden Sie `getSessionEntry(...)`, um Sitzungsmetadaten anhand der Agenten- und Sitzungsidentität
  zu lesen.
- Verwenden Sie `patchSessionEntry(...)` oder `upsertSessionEntry(...)`, um Sitzungsmetadaten
  dauerhaft zu speichern.
- Verwenden Sie Hilfsfunktionen für die Transkriptidentität oder das Transkriptziel, wenn der Code einen
  Transkriptvorgang vorbereitet.
- Speichern Sie keine veralteten Transkriptdateipfade dauerhaft und machen Sie sich nicht von ihnen abhängig.
- Siehe [Laufzeit-API](/de/plugins/sdk-runtime#agent-session-state) und
  [Unterpfade des Plugin-SDKs](/de/plugins/sdk-subpaths).
- Führen Sie `clawhub package validate <path-to-plugin>` erneut aus.

### sdk-session-transcript-file-target

Das Plugin verwendet weiterhin die veraltete Hilfsfunktion für Transkriptdateiziele
`resolveSessionTranscriptLegacyFileTarget`.

- Verwenden Sie `resolveSessionTranscriptIdentity(...)`, wenn der Code nur die öffentliche
  Sitzungsidentität benötigt.
- Verwenden Sie `resolveSessionTranscriptTarget(...)`, wenn der Code ein strukturiertes Ziel
  für einen Transkriptvorgang benötigt.
- Vermeiden Sie es, veraltete Transkriptdateiziele direkt zu lesen oder zu erstellen.
- Behalten Sie die veraltete Hilfsfunktion nur bei, solange Ihr deklarierter Kompatibilitätsbereich noch
  ältere OpenClaw-Versionen unterstützt, die sie benötigen.
- Siehe [Laufzeit-API](/de/plugins/sdk-runtime#agent-session-state) und
  [Unterpfade des Plugin-SDKs](/de/plugins/sdk-subpaths).
- Führen Sie `clawhub package validate <path-to-plugin>` erneut aus.

### sdk-session-transcript-low-level

Das Plugin verwendet weiterhin veraltete systemnahe Transkripthilfsfunktionen, beispielsweise
`appendSessionTranscriptMessage` oder `emitSessionTranscriptUpdate`.

- Verwenden Sie `appendSessionTranscriptMessageByIdentity(...)`, um Transkripte zu ergänzen.
- Verwenden Sie `publishSessionTranscriptUpdateByIdentity(...)` für Benachrichtigungen über
  Transkriptaktualisierungen.
- Bevorzugen Sie die strukturierte Transkript-Laufzeitschnittstelle, damit OpenClaw die
  korrekten Transaktionsgrenzen und die Identitätsverarbeitung anwenden kann.
- Behalten Sie systemnahe Transkripthilfsfunktionen nur bei, solange Ihr deklarierter Kompatibilitätsbereich
  noch ältere OpenClaw-Versionen unterstützt, die sie benötigen.
- Siehe [Laufzeit-API](/de/plugins/sdk-runtime#agent-session-state) und
  [Unterpfade des Plugin-SDKs](/de/plugins/sdk-subpaths).
- Führen Sie `clawhub package validate <path-to-plugin>` erneut aus.

### legacy-before-agent-start

Das Plugin verwendet weiterhin den veralteten Hook `before_agent_start`.

- Verschieben Sie Überschreibungen des Modells oder Providers nach `before_model_resolve`.
- Verschieben Sie Änderungen an Prompt oder Kontext nach `before_prompt_build`.
- Behalten Sie `before_agent_start` nur bei, solange Ihr deklarierter Kompatibilitätsbereich noch
  ältere OpenClaw-Versionen unterstützt, die diesen Hook benötigen.
- Siehe [Hooks](/de/plugins/hooks) und
  [Plugin-Kompatibilität](/de/plugins/compatibility).
- Führen Sie `clawhub package validate <path-to-plugin>` erneut aus.

### provider-auth-env-vars

Das Manifest verwendet weiterhin veraltete Metadaten für die Provider-Authentifizierung in `providerAuthEnvVars`.

- Spiegeln Sie die Metadaten der Provider-Umgebungsvariablen nach `setup.providers[].envVars`.
- Behalten Sie `providerAuthEnvVars` nur als Kompatibilitätsmetadaten bei, solange Ihr unterstützter
  OpenClaw-Versionsbereich sie noch benötigt.
- Siehe [Setup-Referenz](/de/plugins/manifest#setup-reference) und
  [SDK-Migration](/de/plugins/sdk-migration).
- Führen Sie `clawhub package validate <path-to-plugin>` erneut aus.

### channel-env-vars

Das Manifest verwendet veraltete oder ältere Metadaten für Kanalumgebungsvariablen ohne die aktuellen
Setup- oder Konfigurationsmetadaten, die ClawHub erwartet.

- Halten Sie Metadaten für Kanalumgebungsvariablen deklarativ, damit OpenClaw den Setup-Status prüfen kann,
  ohne die Kanallaufzeit zu laden.
- Spiegeln Sie die umgebungsvariablengesteuerte Kanaleinrichtung in die aktuellen Setup-, Kanalkonfigurations- oder
  Paketkanalmetadaten, die von Ihrer Plugin-Struktur verwendet werden.
- Behalten Sie `channelEnvVars` nur als Kompatibilitätsmetadaten bei, solange ältere unterstützte
  OpenClaw-Versionen sie noch benötigen.
- Siehe [Plugin-Manifest](/de/plugins/manifest) und
  [Kanal-Plugins](/plugins/sdk-channel-plugins).
- Führen Sie `clawhub package validate <path-to-plugin>` erneut aus.

## Sicherheitsmanifest

### security-manifest-schema-unavailable

Das Paket liefert `openclaw.security.json` mit einem Schemaverweis aus, den ClawHub
nicht als verfügbar erkennt.

- Entfernen Sie die Schema-URL, wenn sie nur informativen Charakter hat.
- Verwenden Sie ein dokumentiertes versioniertes Schema erst, nachdem OpenClaw eines veröffentlicht hat.
- Führen Sie `clawhub package validate <path-to-plugin>` erneut aus.

### unrecognized-security-manifest

Das Paket liefert eine nicht unterstützte Sicherheitsmanifestdatei aus.

- Entfernen Sie `openclaw.security.json`, bis OpenClaw ein versioniertes Schema für Sicherheitsmanifeste und das
  Verhalten von ClawHub dokumentiert.
- Dokumentieren Sie sicherheitsrelevantes Verhalten weiterhin in Ihrer öffentlichen Paketdokumentation oder
  README, bis der Manifestvertrag verfügbar ist.
- Führen Sie `clawhub package validate <path-to-plugin>` erneut aus.

## Verwandte Themen

- [ClawHub-CLI](/de/clawhub/cli)
- [Veröffentlichen auf ClawHub](/de/clawhub/publishing)
- [Plugins erstellen](/de/plugins/building-plugins)
- [Plugin-Manifest](/de/plugins/manifest)
- [Plugin-Einstiegspunkte](/de/plugins/sdk-entrypoints)
- [Plugin-Kompatibilität](/de/plugins/compatibility)
