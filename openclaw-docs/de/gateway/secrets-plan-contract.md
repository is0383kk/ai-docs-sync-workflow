---
read_when:
    - Generieren oder Überprüfen von `openclaw secrets apply`-Plänen
    - Debugging von `Invalid plan target path`-Fehlern
    - Verhalten bei der Validierung von Zieltyp und Pfad verstehen
summary: 'Vertrag für `secrets apply`-Pläne: Zielvalidierung, Pfadabgleich und Zielumfang von `auth-profiles.json`'
title: Vertrag für den Plan zur Anwendung von Secrets
x-i18n:
    generated_at: "2026-07-26T18:28:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 71ee8afd958646930af4db3bbad08e033ff79da48890a989d72b361abcbda3bb
    source_path: gateway/secrets-plan-contract.md
    workflow: 16
---

Diese Seite definiert den strikten Vertrag, den `openclaw secrets apply` durchsetzt. Wenn ein Ziel diesen Regeln nicht entspricht, schlägt die Anwendung fehl, bevor eine Datei verändert wird.

## Anforderungen an die Plandatei

`openclaw secrets apply --from <plan.json>` akzeptiert reguläre Dateien bis zu 16 MiB (16.777.216 Byte). Der Grenzwert gilt für die vollständige serialisierte Datei einschließlich Leerzeichen. Verzeichnisse, FIFOs, Gerätedateien und Dateien, die den Grenzwert überschreiten, werden vor dem JSON-Parsing oder der Zielvalidierung abgelehnt.

`openclaw secrets configure --plan-out <plan.json>` erzwingt denselben Grenzwert für die UTF-8-serialisierte Ausgabe, bevor die Datei erstellt wird. Manuell erstellte Pläne und externe Plangeneratoren müssen die serialisierte Datei ebenfalls innerhalb dieses Grenzwerts halten.

## Struktur der Plandatei

`openclaw secrets apply --from <plan.json>` erwartet ein `targets`-Array von Planzielen:

```json5
{
  version: 1,
  protocolVersion: 1,
  targets: [
    {
      type: "models.providers.apiKey",
      path: "models.providers.openai.apiKey",
      pathSegments: ["models", "providers", "openai", "apiKey"],
      providerId: "openai",
      ref: { source: "env", provider: "default", id: "OPENAI_API_KEY" },
    },
    {
      type: "auth-profiles.api_key.key",
      path: "profiles.openai:default.key",
      pathSegments: ["profiles", "openai:default", "key"],
      agentId: "main",
      ref: { source: "env", provider: "default", id: "OPENAI_API_KEY" },
    },
  ],
}
```

`openclaw secrets configure` generiert Pläne in dieser Struktur. Sie können einen solchen Plan auch manuell erstellen oder bearbeiten.

## Provider-Aktualisierungen und -Löschungen

Pläne können außerdem zwei optionale Felder auf oberster Ebene enthalten, die neben den zielbezogenen Schreibvorgängen die `secrets.providers`-Zuordnung verändern:

- `providerUpserts` -- ein Objekt, dessen Schlüssel Provider-Aliasse sind. Jeder Wert ist eine Provider-Definition (dieselbe Struktur, die unter `secrets.providers.<alias>` in `openclaw.json` akzeptiert wird, z. B. ein `exec`- oder `file`-Provider).
- `providerDeletes` -- ein Array der zu entfernenden Provider-Aliasse.

`providerUpserts` wird vor `targets` ausgeführt, sodass ein `target.ref.provider` auf einen Provider-Alias verweisen kann, den derselbe Plan in `providerUpserts` einführt. Ohne diese Reihenfolge schlagen Pläne, die auf einen noch nicht in `openclaw.json` konfigurierten Alias verweisen, mit `provider "<alias>" is not configured` fehl.

```json5
{
  version: 1,
  protocolVersion: 1,
  providerUpserts: {
    onepassword_anthropic: {
      source: "exec",
      command: "/usr/bin/op",
      args: ["read", "op://Vault/Anthropic/credential"],
    },
  },
  providerDeletes: ["legacy_unused_alias"],
  targets: [
    {
      type: "models.providers.apiKey",
      path: "models.providers.anthropic.apiKey",
      pathSegments: ["models", "providers", "anthropic", "apiKey"],
      providerId: "anthropic",
      ref: { source: "exec", provider: "onepassword_anthropic", id: "credential" },
    },
  ],
}
```

Über `providerUpserts` eingeführte Exec-Provider unterliegen weiterhin den Exec-Zustimmungsregeln unter [Zustimmungsverhalten für Exec-Provider](#exec-provider-consent-behavior): Pläne mit Exec-Providern erfordern im Schreibmodus `--allow-exec`.

## Unterstützter Zielbereich

Planziele werden für unterstützte Anmeldedatenpfade in der [SecretRef-Anmeldedatenoberfläche](/de/reference/secretref-credential-surface) akzeptiert.

## Verhalten der Zieltypen

`target.type` muss ein erkannter Zieltyp sein, und der normalisierte `target.path` muss der registrierten Pfadstruktur dieses Typs entsprechen.

Einige Zieltypen akzeptieren zusätzlich zu ihrem kanonischen Typnamen einen Kompatibilitätsalias als `target.type` für bestehende Pläne:

| Kanonischer Typ                       | Akzeptierter Alias                                  |
| ------------------------------------ | ----------------------------------------------- |
| `models.providers.apiKey`            | `models.providers.*.apiKey`                     |
| `skills.entries.apiKey`              | `skills.entries.*.apiKey`                       |
| `channels.googlechat.serviceAccount` | `channels.googlechat.accounts.*.serviceAccount` |

## Regeln zur Pfadvalidierung

Jedes Ziel wird anhand aller folgenden Regeln validiert:

- `type` muss ein erkannter Zieltyp sein.
- `path` muss ein nicht leerer, durch Punkte getrennter Pfad sein.
- `pathSegments` kann weggelassen werden. Falls angegeben, muss es bei der Normalisierung exakt denselben Pfad wie `path` ergeben.
- Unzulässige Segmente werden abgelehnt: `__proto__`, `prototype`, `constructor`.
- Der normalisierte Pfad muss der für den Zieltyp registrierten Pfadstruktur entsprechen.
- Wenn `providerId` oder `accountId` festgelegt ist, muss der Wert mit der im Pfad codierten ID übereinstimmen.
- Ziele des Typs `auth-profiles.json` erfordern `agentId`.
- Beim Erstellen einer neuen `auth-profiles.json`-Zuordnung muss `authProfileProvider` angegeben werden.

## Fehlerverhalten

Wenn die Validierung eines Ziels fehlschlägt, wird die Anwendung mit einer Fehlermeldung wie dieser beendet:

```text
Ungültiger Planzielpfad für models.providers.apiKey: models.providers.openai.baseUrl
```

Bei einem ungültigen Plan werden keine Schreibvorgänge übernommen: Zielauflösung und Pfadvalidierung werden ausgeführt, bevor eine Datei verändert wird. Sobald ein gültiger Plan mit dem Schreiben beginnt, erstellt die Anwendung außerdem zunächst Momentaufnahmen aller betroffenen Dateien und stellt diese wieder her, wenn ein späterer Schreibvorgang im selben Durchlauf fehlschlägt. Dadurch geraten Konfiguration, Authentifizierungsprofile und Umgebungsstatus durch einen unvollständigen Schreibvorgang nie auseinander.

## Zustimmungsverhalten für Exec-Provider

- `--dry-run` überspringt Prüfungen von Exec-SecretRefs standardmäßig.
- Pläne mit Exec-SecretRefs/-Providern werden im Schreibmodus abgelehnt, sofern `--allow-exec` nicht festgelegt ist.
- Übergeben Sie beim Validieren/Anwenden von Plänen mit Exec-Inhalten sowohl für Probelauf- als auch für Schreibbefehle `--allow-exec`.

## Hinweise zum Laufzeit- und Prüfumfang

- Reine Referenzeinträge des Typs `auth-profiles.json` (`keyRef`/`tokenRef`) werden bei der Laufzeitauflösung von Anmeldedaten und im Prüfumfang berücksichtigt.
- `secrets apply` schreibt unterstützte `openclaw.json`-Ziele, unterstützte `auth-profiles.json`-Ziele sowie drei optionale Bereinigungsdurchläufe, die jeweils standardmäßig aktiviert sind: `scrubEnv` (entfernt migrierte Klartextwerte aus `.env`-Dateien in den Verzeichnissen für den effektiven Status und die aktive Konfiguration), `scrubAuthProfilesForProviderTargets` (entfernt Klartext- und nicht verwendete Referenzrückstände in `auth-profiles.json` für Provider, die ein Plan gerade migriert hat) und `scrubLegacyAuthJson` (entfernt migrierte `api_key`-Einträge aus veralteten `auth.json`-Speichern). Setzen Sie im Plan einen der Werte `options.scrubEnv`, `options.scrubAuthProfilesForProviderTargets` oder `options.scrubLegacyAuthJson` auf `false`, um den jeweiligen Durchlauf zu überspringen.

## Prüfungen für Betreiber

```bash
# Plan ohne Schreibvorgänge validieren
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run

# Anschließend tatsächlich anwenden
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json

# Bei Plänen mit Exec-Inhalten in beiden Modi ausdrücklich zustimmen
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run --allow-exec
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --allow-exec
```

Wenn die Anwendung mit einer Meldung über einen ungültigen Zielpfad fehlschlägt, generieren Sie den Plan mit `openclaw secrets configure` neu oder korrigieren Sie den Zielpfad entsprechend einer der oben aufgeführten unterstützten Strukturen.

## Zugehörige Dokumentation

- [Verwaltung von Geheimnissen](/de/gateway/secrets)
- [CLI `secrets`](/de/cli/secrets)
- [SecretRef-Anmeldedatenoberfläche](/de/reference/secretref-credential-surface)
- [Konfigurationsreferenz](/de/gateway/configuration-reference)
