---
read_when:
    - Genereren of beoordelen van `openclaw secrets apply`-plannen
    - Fouten met `Invalid plan target path` opsporen
    - Inzicht in het gedrag van doeltype- en padvalidatie
summary: 'Contract voor `secrets apply`-plannen: doelvalidatie, padvergelijking en doelbereik van `auth-profiles.json`'
title: Contract voor het toepassen van het geheimenplan
x-i18n:
    generated_at: "2026-07-27T05:48:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 71ee8afd958646930af4db3bbad08e033ff79da48890a989d72b361abcbda3bb
    source_path: gateway/secrets-plan-contract.md
    workflow: 16
---

Deze pagina definieert het strikte contract dat door `openclaw secrets apply` wordt afgedwongen. Als een doel niet aan deze regels voldoet, mislukt het toepassen voordat een bestand wordt gewijzigd.

## Vereisten voor het planbestand

`openclaw secrets apply --from <plan.json>` accepteert reguliere bestanden tot 16 MiB (16,777,216 bytes). De limiet geldt voor het volledige geserialiseerde bestand, inclusief witruimte. Mappen, FIFO's, apparaatbestanden en bestanden die groter zijn dan de limiet worden geweigerd voordat JSON-parsing of doelvalidatie plaatsvindt.

`openclaw secrets configure --plan-out <plan.json>` dwingt dezelfde limiet af voor de als UTF-8 geserialiseerde uitvoer voordat het bestand wordt aangemaakt. Handmatig geschreven plannen en externe plangeneratoren moeten het geserialiseerde bestand eveneens binnen deze grens houden.

## Structuur van het planbestand

`openclaw secrets apply --from <plan.json>` verwacht een `targets`-array met plandoelen:

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

`openclaw secrets configure` genereert plannen met deze structuur. Je kunt er ook zelf een schrijven of bewerken.

## Providers invoegen, bijwerken en verwijderen

Plannen kunnen ook twee optionele velden op het hoogste niveau bevatten die naast de schrijfbewerkingen per doel de `secrets.providers`-toewijzing wijzigen:

- `providerUpserts` -- een object met provideraliassen als sleutels. Elke waarde is een providerdefinitie (dezelfde structuur die onder `secrets.providers.<alias>` in `openclaw.json` wordt geaccepteerd, bijvoorbeeld een `exec`- of `file`-provider).
- `providerDeletes` -- een array met te verwijderen provideraliassen.

`providerUpserts` wordt vóór `targets` uitgevoerd, zodat een `target.ref.provider` kan verwijzen naar een provideralias die hetzelfde plan in `providerUpserts` introduceert. Zonder deze volgorde mislukken plannen die verwijzen naar een alias die nog niet in `openclaw.json` is geconfigureerd met `provider "<alias>" is not configured`.

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

Exec-providers die via `providerUpserts` worden geïntroduceerd, vallen nog steeds onder de regels voor exec-toestemming in [Toestemmingsgedrag voor exec-providers](#exec-provider-consent-behavior): plannen met exec-providers vereisen `--allow-exec` in schrijfmodus.

## Ondersteund doelbereik

Plandoelen worden geaccepteerd voor ondersteunde referentiepaden in [SecretRef-referentieoppervlak](/nl/reference/secretref-credential-surface).

## Gedrag van doeltypen

`target.type` moet een herkend doeltype zijn en het genormaliseerde `target.path` moet overeenkomen met de geregistreerde padstructuur van dat type.

Sommige doeltypen accepteren naast hun canonieke typenaam ook een compatibiliteitsalias als `target.type` voor bestaande plannen:

| Canoniek type                        | Geaccepteerde alias                            |
| ------------------------------------ | ----------------------------------------------- |
| `models.providers.apiKey`            | `models.providers.*.apiKey`                     |
| `skills.entries.apiKey`              | `skills.entries.*.apiKey`                       |
| `channels.googlechat.serviceAccount` | `channels.googlechat.accounts.*.serviceAccount` |

## Regels voor padvalidatie

Elk doel wordt aan de hand van al het volgende gevalideerd:

- `type` moet een herkend doeltype zijn.
- `path` moet een niet-leeg, door punten gescheiden pad zijn.
- `pathSegments` mag worden weggelaten. Als het is opgegeven, moet het exact naar hetzelfde pad als `path` worden genormaliseerd.
- Verboden segmenten worden geweigerd: `__proto__`, `prototype`, `constructor`.
- Het genormaliseerde pad moet overeenkomen met de geregistreerde padstructuur voor het doeltype.
- Als `providerId` of `accountId` is ingesteld, moet dit overeenkomen met de in het pad gecodeerde id.
- `auth-profiles.json`-doelen vereisen `agentId`.
- Neem `authProfileProvider` op wanneer je een nieuwe `auth-profiles.json`-toewijzing maakt.

## Gedrag bij fouten

Als de validatie van een doel mislukt, wordt het toepassen beëindigd met een fout zoals:

```text
Ongeldig plandoelpad voor models.providers.apiKey: models.providers.openai.baseUrl
```

Voor een ongeldig plan worden geen schrijfbewerkingen vastgelegd: doelresolutie en padvalidatie worden uitgevoerd voordat een bestand wordt aangeraakt. Zodra een geldig plan begint te schrijven, maakt het toepassen bovendien eerst momentopnamen van elk aangeraakt bestand en herstelt het deze als een latere schrijfbewerking tijdens dezelfde uitvoering mislukt, zodat een gedeeltelijke schrijfbewerking de configuratie-, authenticatieprofiel- of omgevingsstatus nooit uit synchronisatie brengt.

## Toestemmingsgedrag voor exec-providers

- `--dry-run` slaat controles van exec-SecretRefs standaard over.
- Plannen met exec-SecretRefs/providers worden in schrijfmodus geweigerd, tenzij `--allow-exec` is ingesteld.
- Geef bij het valideren/toepassen van plannen met exec `--allow-exec` door in zowel de droogloop- als de schrijfopdracht.

## Opmerkingen over runtime- en auditbereik

- Alleen-uit-referenties-bestaande `auth-profiles.json`-vermeldingen (`keyRef`/`tokenRef`) worden meegenomen in de runtime-resolutie van referenties en de auditdekking.
- `secrets apply` schrijft ondersteunde `openclaw.json`-doelen, ondersteunde `auth-profiles.json`-doelen en drie optionele opschoningsrondes, die elk standaard zijn ingeschakeld: `scrubEnv` (verwijdert gemigreerde waarden in platte tekst uit `.env`-bestanden in de effectieve status- en actieve-configuratiemappen), `scrubAuthProfilesForProviderTargets` (wist restanten van platte tekst/ongebruikte referenties in `auth-profiles.json` voor providers die zojuist door een plan zijn gemigreerd) en `scrubLegacyAuthJson` (verwijdert gemigreerde `api_key`-vermeldingen uit verouderde `auth.json`-opslagplaatsen). Stel een van `options.scrubEnv`, `options.scrubAuthProfilesForProviderTargets`, `options.scrubLegacyAuthJson` in het plan in op `false` om die ronde over te slaan.

## Controles voor beheerders

```bash
# Plan valideren zonder schrijfbewerkingen
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run

# Daarna daadwerkelijk toepassen
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json

# Voor plannen met exec: expliciet aanmelden in beide modi
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run --allow-exec
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --allow-exec
```

Als het toepassen mislukt met een melding over een ongeldig doelpad, genereer het plan dan opnieuw met `openclaw secrets configure` of corrigeer het doelpad naar een hierboven ondersteunde structuur.

## Gerelateerde documentatie

- [Geheimenbeheer](/nl/gateway/secrets)
- [CLI `secrets`](/nl/cli/secrets)
- [SecretRef-referentieoppervlak](/nl/reference/secretref-credential-surface)
- [Configuratiereferentie](/nl/gateway/configuration-reference)
