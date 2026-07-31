---
read_when:
    - Geheime verwijzingen tijdens runtime opnieuw omzetten
    - Platte-tekstresten en onopgeloste verwijzingen controleren
    - SecretRefs configureren en eenrichtingsopschoningswijzigingen toepassen
summary: CLI-referentie voor `openclaw secrets` (opnieuw laden, controleren, configureren, toepassen)
title: Geheimen
x-i18n:
    generated_at: "2026-07-27T06:09:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 61f6f81e358ca2e6a97ac9498186b32f7a74d16052d226c398dad0030d47211e
    source_path: cli/secrets.md
    workflow: 16
---

# `openclaw secrets`

Beheer SecretRefs en houd de actieve runtimesnapshot gezond.

| Opdracht    | Rol                                                                                                                                                                                                                      |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `reload`    | Gateway-RPC (`secrets.reload`): lost refs opnieuw op en publiceert atomair de eigenaarsbewuste runtimesnapshot (zonder configuratie te schrijven); in aanmerking komende fouten van eigenaren kunnen als koude of verouderde waarschuwingen worden gepubliceerd |
| `audit`     | Alleen-lezen-scan van configuratie-/auth-/gegenereerde-modelarchieven en verouderde restanten op platte tekst, niet-opgeloste refs en prioriteitsafwijkingen (exec-refs worden overgeslagen tenzij `--allow-exec`)                      |
| `configure` | Interactieve planner voor providerconfiguratie, doeltoewijzing en preflight (vereist een TTY)                                                                                                       |
| `apply`     | Voert een opgeslagen plan uit (`--dry-run` valideert alleen en slaat exec-controles standaard over; schrijfmodus weigert plannen met exec tenzij `--allow-exec`) en verwijdert vervolgens gerichte restanten in platte tekst |

Aanbevolen beheerderscyclus:

```bash
openclaw secrets audit --check
openclaw secrets configure
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json
openclaw secrets audit --check
openclaw secrets reload
```

Als je plan `exec` SecretRefs/providers bevat, geef je `--allow-exec` door bij zowel de dry-run- als de schrijfopdrachten voor `apply`.

Afsluitcodes voor CI/controles:

- `audit --check` retourneert `1` bij bevindingen.
- Niet-opgeloste refs retourneren `2` (ongeacht `--check`).

Gerelateerd: [Geheimenbeheer](/nl/gateway/secrets) · [Referentieoppervlak voor SecretRef-aanmeldgegevens](/nl/reference/secretref-credential-surface) · [Beveiliging](/nl/gateway/security)

## Runtimesnapshot opnieuw laden

```bash
openclaw secrets reload
openclaw secrets reload --json
openclaw secrets reload --url ws://127.0.0.1:18789 --token <token>
```

Gebruikt de Gateway-RPC-methode `secrets.reload`. Gezonde eigenaren worden onafhankelijk vernieuwd. In aanmerking komende mislukte eigenaren worden alleen verouderd wanneer hun ref-identiteiten, providerdefinities en volledige niet-geheime eigenaarscontract ongewijzigd zijn; nieuwe of gewijzigde fouten worden koud. Deze gedegradeerde activering slaagt en rapporteert `warningCount`. Strikte of niet-toegewezen fouten retourneren een fout en behouden de eerder actieve snapshot.

Opties: `--url <url>`, `--token <token>`, `--timeout <ms>`, `--json`.

## Audit

Scant de OpenClaw-status op:

- opslag van geheimen in platte tekst
- niet-opgeloste refs
- prioriteitsafwijkingen (`auth-profiles.json`-aanmeldgegevens die `openclaw.json`-refs overschaduwen)
- gegenereerde `agents/*/agent/models.json`-restanten (providerwaarden voor `apiKey` en gevoelige providerheaders)
- verouderde restanten (verouderde vermeldingen in het auth-archief, OAuth-herinneringen)

De `.env`-scan omvat de effectieve statusmap en de map met de actieve configuratie. Wanneer beide paden hetzelfde bestand aanduiden, wordt het eenmaal gescand.

Detectie van gevoelige providerheaders is gebaseerd op heuristiek van de naam: headers worden gemarkeerd wanneer hun naam overeenkomt met veelvoorkomende auth-/aanmeldgegevensfragmenten (`authorization`, `x-api-key`, `token`, `secret`, `password`, `credential`).

```bash
openclaw secrets audit
openclaw secrets audit --check
openclaw secrets audit --json
openclaw secrets audit --allow-exec
```

Rapportstructuur:

- `status`: `clean | findings | unresolved`
- `resolution`: `refsChecked`, `skippedExecRefs`, `resolvabilityComplete`
- `summary`: `plaintextCount`, `unresolvedRefCount`, `shadowedRefCount`, `legacyResidueCount`
- bevindingcodes: `PLAINTEXT_FOUND`, `REF_UNRESOLVED`, `REF_SHADOWED`, `LEGACY_RESIDUE`

## Configureren (interactieve hulp)

Stel provider- en SecretRef-wijzigingen interactief samen, voer een preflight uit en pas ze eventueel toe:

```bash
openclaw secrets configure
openclaw secrets configure --plan-out /tmp/openclaw-secrets-plan.json
openclaw secrets configure --apply --yes
openclaw secrets configure --providers-only
openclaw secrets configure --skip-provider-setup
openclaw secrets configure --agent ops
openclaw secrets configure --json
```

Proces: eerst providerconfiguratie (`secrets.providers`-aliassen toevoegen/bewerken/verwijderen), vervolgens toewijzing van aanmeldgegevens (velden selecteren en `{source, provider, id}`-refs toewijzen), daarna preflight en optioneel toepassen.

Vlaggen:

- `--providers-only`: configureer alleen `secrets.providers`, sla de toewijzing van aanmeldgegevens over
- `--skip-provider-setup`: sla providerconfiguratie over, wijs aanmeldgegevens toe aan bestaande providers
- `--agent <id>`: beperk het ontdekken en schrijven van `auth-profiles.json`-doelen tot één agentarchief
- `--allow-exec`: sta controles van exec-SecretRefs toe tijdens preflight/toepassen (kan provideropdrachten uitvoeren)

`--providers-only` en `--skip-provider-setup` kunnen niet worden gecombineerd.

Opmerkingen:

- Vereist een interactieve TTY.
- Richt zich op velden met geheimen in `openclaw.json` plus `auth-profiles.json` voor het geselecteerde agentbereik; canoniek ondersteund oppervlak: [Referentieoppervlak voor SecretRef-aanmeldgegevens](/nl/reference/secretref-credential-surface).
- Ondersteunt het rechtstreeks maken van nieuwe `auth-profiles.json`-toewijzingen in het selectieproces.
- Voert vóór het toepassen preflight-resolutie uit.
- Bij gegenereerde plannen zijn verwijderingsopties standaard ingeschakeld (`scrubEnv`, `scrubAuthProfilesForProviderTargets`, `scrubLegacyAuthJson`). Toepassen is eenrichtingsverkeer voor verwijderde waarden in platte tekst.
- `--plan-out` weigert een plan te maken waarvan de geserialiseerde UTF-8-vorm groter is dan 16 MiB (16,777,216 bytes), overeenkomstig de invoerlimiet van `apply --from`.
- Zonder `--apply` toont de CLI na de preflight nog steeds de prompt `Apply this plan now?`.
- Met `--apply` (en zonder `--yes`) toont de CLI een extra bevestiging voor de onomkeerbare migratie.
- `--json` drukt het plan en het preflightrapport af, maar vereist nog steeds een interactieve TTY.

### Veiligheid van exec-providers

Homebrew-installaties stellen vaak via symbolische koppelingen beschikbare binaire bestanden onder `/opt/homebrew/bin/*` bloot. Stel `allowSymlinkCommand: true` alleen in wanneer dat nodig is voor vertrouwde paden van pakketbeheerders, in combinatie met `trustedDirs` (bijvoorbeeld `["/opt/homebrew"]`). Als op Windows ACL-verificatie niet beschikbaar is voor een providerpad, sluit OpenClaw dit uit veiligheid af; stel alleen voor vertrouwde paden `allowInsecurePath: true` in voor die provider om de padbeveiligingscontrole te omzeilen.

## Een opgeslagen plan toepassen

```bash
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --allow-exec
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run --allow-exec
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --json
```

`--dry-run` valideert de preflight zonder bestanden te schrijven; controles van exec-SecretRefs worden standaard overgeslagen tijdens een dry-run. De schrijfmodus weigert plannen met exec-SecretRefs/providers tenzij `--allow-exec`. Gebruik `--allow-exec` om in een van beide modi expliciet toestemming te geven voor controles/uitvoering van exec-providers.

`--from` moet verwijzen naar een normaal bestand van maximaal 16 MiB (16,777,216 bytes). De bytelimiet geldt voor het volledige geserialiseerde bestand, inclusief witruimte.

Wat `apply` kan bijwerken:

- `openclaw.json` (SecretRef-doelen + providers bijwerken/toevoegen/verwijderen)
- `auth-profiles.json` (opschonen van providerdoelen)
- verouderde `auth.json`-restanten
- `.env`-bestanden in de effectieve status- en actieve-configuratiemappen, voor bekende geheime sleutels waarvan de waarden zijn gemigreerd

Details van het plancontract (toegestane doelpaden, validatieregels, foutsemantiek): [Contract voor het toepassen van geheimenplannen](/nl/gateway/secrets-plan-contract).

### Waarom er geen rollbackback-ups zijn

`secrets apply` schrijft opzettelijk geen rollbackback-ups met oude waarden in platte tekst. De veiligheid komt voort uit een strikte preflight plus vrijwel atomair toepassen, met bij een fout een herstelpoging in het geheugen.

## Voorbeeld

```bash
openclaw secrets audit --check
openclaw secrets configure
openclaw secrets audit --check
```

Als `audit --check` nog steeds bevindingen in platte tekst rapporteert, werk je de resterende gerapporteerde doelpaden bij en voer je de audit opnieuw uit.

## Gerelateerd

- [CLI-referentie](/nl/cli)
- [Geheimenbeheer](/nl/gateway/secrets)
- [Vault SecretRefs](/nl/plugins/vault)
