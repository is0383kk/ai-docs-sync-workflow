---
read_when:
    - Werken aan de resolutie van authenticatieprofielen of de routering van aanmeldgegevens
    - Fouten met modelauthenticatie of de profielvolgorde opsporen
summary: Canonieke semantiek voor de geschiktheid en resolutie van inloggegevens voor authenticatieprofielen
title: Semantiek van authenticatiegegevens
x-i18n:
    generated_at: "2026-07-27T05:24:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6b0516b1bb23f400d5ac5fd39a628736034440216ac22823eef061b38564dff0
    source_path: auth-credential-semantics.md
    workflow: 16
---

Deze semantiek houdt het authenticatiegedrag tijdens selectie en tijdens runtime op elkaar afgestemd. Ze wordt gedeeld door:

- `resolveAuthProfileOrder` (profielvolgorde)
- `resolveApiKeyForProfile` (oplossen van runtime-aanmeldgegevens)
- `openclaw models status --probe`
- `openclaw doctor`-authenticatiecontroles (`doctor-auth`)

## Stabiele redencodes voor probes

Proberesultaten bevatten een `status`-categorie (`ok`, `auth`, `rate_limit`, `billing`, `timeout`, `format`, `unknown`, `no_model`) plus een stabiele `reasonCode` wanneer de probe nooit een modelaanroep heeft bereikt:

| `reasonCode`             | Betekenis                                                                    |
| ------------------------ | ---------------------------------------------------------------------------- |
| `excluded_by_auth_order` | Profiel weggelaten uit de expliciete authenticatievolgorde voor de provider. |
| `missing_credential`     | Er is geen inline-aanmeldgegeven of SecretRef geconfigureerd.                 |
| `expired`                | Token `expires` ligt in het verleden.                                |
| `invalid_expires`        | `expires` is geen geldige positieve Unix-tijdstempel in ms.          |
| `unresolved_ref`         | De geconfigureerde SecretRef kon niet worden opgelost.                        |
| `ineligible_profile`     | Profiel is niet compatibel met de providerconfiguratie (inclusief onjuist gevormde sleutelinvoer). |
| `no_model`               | Er bestaan aanmeldgegevens, maar er is geen kandidaat voor een testbaar model gevonden. |

Geschiktheidscontroles rapporteren `ok` als de redencode voor bruikbare aanmeldgegevens.

## Tokenaanmeldgegevens

Tokenaanmeldgegevens (`type: "token"`) ondersteunen inline `token` en/of `tokenRef`.

### Geschiktheidsregels

1. Een tokenprofiel is ongeschikt wanneer zowel `token` als `tokenRef` ontbreken (`missing_credential`).
2. `expires` is optioneel. Indien aanwezig, moet het een eindig aantal milliseconden sinds de Unix-epoch zijn dat groter is dan `0` en niet groter dan de maximale JavaScript-`Date`-tijdstempel (8640000000000000).
3. Als `expires` ongeldig is (verkeerd type, `NaN`, `0`, negatief, niet-eindig of groter dan dat maximum), is het profiel ongeschikt met `invalid_expires`.
4. Als `expires` in het verleden ligt, is het profiel ongeschikt met `expired`.
5. `tokenRef` omzeilt de validatie van `expires` niet.

### Oplosregels

1. De semantiek van de resolver komt voor `expires` overeen met de geschiktheidssemantiek.
2. Voor geschikte profielen kan tokenmateriaal worden opgelost vanuit de inlinewaarde of `tokenRef`.
3. Niet-oplosbare verwijzingen leveren `unresolved_ref` op in de uitvoer van `models status --probe`.

## Overdraagbaarheid van agentkopieën

Overerving van agentauthenticatie werkt via doorlezing. Wanneer een agent geen lokaal profiel heeft, worden profielen tijdens runtime vanuit de standaard-/hoofdagentopslag opgelost zonder geheim materiaal naar de eigen opslag voor aanmeldgegevens te kopiëren (`agents/<agentId>/agent/openclaw-agent.sqlite`).

Expliciete kopieerstromen, zoals `openclaw agents add`, gebruiken dit overdraagbaarheidsbeleid:

- `api_key`- en `token`-profielen zijn overdraagbaar, tenzij `copyToAgents: false`.
- `oauth`-profielen zijn standaard niet overdraagbaar, omdat vernieuwingstokens eenmalig bruikbaar of gevoelig voor rotatie kunnen zijn.
- OAuth-stromen die eigendom zijn van de provider mogen zich alleen aanmelden met `copyToAgents: true` wanneer bekend is dat het veilig is om vernieuwingsmateriaal tussen agents te kopiëren; deze aanmelding geldt alleen wanneer het profiel inline toegangs-/vernieuwingsmateriaal bevat.

Niet-overdraagbare profielen blijven beschikbaar via overerving door doorlezing, tenzij de doelagent zich afzonderlijk aanmeldt en een eigen lokaal profiel maakt.

## Authenticatieroutes uitsluitend via configuratie

`auth.profiles`-vermeldingen met `mode: "aws-sdk"` zijn routeringsmetadata, geen opgeslagen aanmeldgegevens. Ze zijn geldig wanneer de doelprovider `models.providers.<id>.auth: "aws-sdk"` gebruikt, de route die door de Plugin beheerde Amazon Bedrock-installatie schrijft. Deze profiel-id's kunnen voorkomen in `auth.order` en sessieoverschrijvingen, zelfs wanneer er geen overeenkomende vermelding in de opslag voor aanmeldgegevens bestaat.

Schrijf `type: "aws-sdk"` niet naar de opslag voor aanmeldgegevens; opgeslagen aanmeldgegevens zijn uitsluitend `api_key`, `token` of `oauth`. Als een verouderde `auth-profiles.json` zo'n markering bevat, verplaatst `openclaw doctor --fix` deze naar `auth.profiles` en verwijdert het de markering uit de opslag.

## Filteren op expliciete authenticatievolgorde

- Wanneer `auth.order.<provider>` of de volgordeoverschrijving van de authenticatieopslag voor een provider is ingesteld, test `models status --probe` alleen profiel-id's die in de opgeloste authenticatievolgorde voor die provider blijven staan. De opgeslagen overschrijving heeft voorrang op de `auth.order`-configuratie.
- Een opgeslagen profiel voor die provider dat uit de expliciete volgorde is weggelaten, wordt later niet stilzwijgend alsnog geprobeerd. De probe-uitvoer rapporteert het met `reasonCode: excluded_by_auth_order` en het detail `Excluded by auth.order for this provider.`

## Oplossen van probedoelen

- Probedoelen kunnen afkomstig zijn van authenticatieprofielen, omgevingsaanmeldgegevens of `models.json` (resultaat `source`: `profile`, `env`, `models.json`).
- Als een provider aanmeldgegevens heeft, maar OpenClaw er geen kandidaat voor een testbaar model voor kan oplossen, rapporteert `models status --probe` `status: no_model` met `reasonCode: no_model`.

## Externe CLI-aanmeldgegevens detecteren

- Aanmeldgegevens die uitsluitend tijdens runtime beschikbaar zijn en door externe CLI's worden beheerd (Claude CLI voor `claude-cli`, Codex CLI voor `openai`, MiniMax CLI voor `minimax-portal`), worden alleen gedetecteerd wanneer de provider, runtime of het authenticatieprofiel binnen het bereik van de huidige bewerking valt, of wanneer er al een opgeslagen lokaal profiel voor die externe bron bestaat.
- Aanroepers van de authenticatieopslag kiezen een expliciete detectiemodus voor externe CLI's: `none` uitsluitend voor opgeslagen/Plugin-authenticatie, `existing` voor het vernieuwen van reeds opgeslagen externe CLI-profielen, of `scoped` voor een concrete set providers/profielen.
- Alleen-lezen-/statuspaden geven `allowKeychainPrompt: false` door; ze gebruiken uitsluitend bestandsgebaseerde externe CLI-aanmeldgegevens en lezen of hergebruiken geen resultaten uit de macOS-sleutelhanger.

## OAuth SecretRef-beleidscontrole

SecretRef-invoer is uitsluitend bedoeld voor statische aanmeldgegevens. OAuth-aanmeldgegevens kunnen tijdens runtime worden gewijzigd (vernieuwingsstromen slaan geroteerde tokens op), waardoor OAuth-materiaal met SecretRef-ondersteuning de wijzigbare toestand over meerdere opslaglocaties zou verdelen.

- Als een profielaanmeldgegeven `type: "oauth"` is, worden SecretRef-objecten geweigerd voor elk veld met aanmeldgegevensmateriaal in dat profiel.
- Als `auth.profiles.<id>.mode` `"oauth"` is, wordt SecretRef-gebaseerde `keyRef`-/`tokenRef`-invoer voor dat profiel geweigerd.
- Overtredingen zijn harde fouten (geworpen fouten) in de paden voor de voorbereiding van geheimen bij opstarten/herladen en voor het oplossen van profielen.

## Berichten die compatibel zijn met oudere versies

Voor scriptcompatibiliteit behouden probefouten deze eerste regel ongewijzigd:

`Auth profile credentials are missing or expired.`

Gebruiksvriendelijke details en de stabiele redencode volgen op volgende regels in de vorm `↳ Auth reason [code]: ...`.

## Gerelateerd

- [Beheer van geheimen](/nl/gateway/secrets)
- [Authenticatieopslag](/nl/concepts/oauth)
