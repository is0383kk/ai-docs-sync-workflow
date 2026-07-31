---
read_when:
    - Je wilt dat agents zorgvuldig geselecteerde 1Password-geheimen opvragen
    - Je hebt een goedkeuringsbeleid en auditgeschiedenis per geheim nodig
    - Je configureert een 1Password-serviceaccount voor OpenClaw
summary: Gebruik de optionele 1Password-plugin als gecontroleerde geheimenbroker voor agents
title: 1Password-geheimenbroker
x-i18n:
    generated_at: "2026-07-27T06:26:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 255ab4fd2c63754fef29d3ea87dcedc9ca2bd2f34bec1f81139e2ce5b6acdba2
    source_path: plugins/onepassword.md
    workflow: 16
---

# 1Password-geheimenbroker

De meegeleverde `onepassword`-plugin biedt agents één beleidsmatig beheerde tool voor
het lezen van een samengestelde set 1Password-velden. Deze is standaard uitgeschakeld en doet
niets totdat `plugins.entries.onepassword.config` aanwezig is.

Dit is een agenttool, geen SecretRef-provider. Deze injecteert geen omgevingsvariabelen
en lost geen geheimen in de OpenClaw-configuratie op.

## Beveiligingsmodel

- Alleen authenticatie met een serviceaccount. Het token blijft in een lokaal bestand met aanmeldgegevens
  en wordt nooit geaccepteerd in `openclaw.json`.
- Alleen een samengestelde registry. Agents kunnen geconfigureerde slugs weergeven, maar de plugin
  inventariseert nooit een 1Password-kluis.
- Beleid per slug: `auto`, `approve` of `deny`.
- Goedkeuringstoekenningen verlopen. Een gecachte waarde omzeilt nooit het huidige beleid.
- Elke toegangspoging wordt vastgelegd in de gedeelde SQLite-status van OpenClaw. Audit-
  rijen bevatten de opgegeven reden; zorg dat redenen geen gevoelige informatie bevatten. De broker
  kopieert een opgehaalde waarde of het servicetoken nooit naar een auditrij.
- Na de huidige tooluitvoering vervangt de door OpenClaw beheerde transcriptopslag
  een geslaagde `get`-waarde door geredigeerde metadata.
- De waarde is tijdens die uitvoering zichtbaar voor het model. Als het model deze naar een
  latere toolaanroep of een later antwoord kopieert, valt die afzonderlijke registratie buiten de
  opslaghook van deze plugin. Houd het beleid beperkt en vraag het model niet om een
  waarde te herhalen.
- De plugin roept `op` eenmaal aan per cachemisser. Limietoverschrijdingen of
  andere fouten worden niet opnieuw geprobeerd.
- Elke `op`-aanroep wordt uitgevoerd met een minimale omgeving die de integratie met de
  1Password-desktopapp uitschakelt (`OP_LOAD_DESKTOP_APP_SETTINGS=false`,
  `OP_BIOMETRIC_UNLOCK_ENABLED=false`), zodat een 1Password-app die op de
  Gateway-host is geïnstalleerd nooit biometrische of macOS-machtigingsvensters activeert.

Geef het serviceaccount alleen leestoegang tot de kluizen en items die in
de pluginconfiguratie zijn geregistreerd.

## Voordat je begint

Je hebt het volgende nodig:

- de 1Password-CLI (`op`) geïnstalleerd op de Gateway-host
- een 1Password-serviceaccount met toegang tot de geselecteerde items
- een speciaal tokenbestand voor het serviceaccount

Schakel de meegeleverde plugin in:

```bash
openclaw plugins enable onepassword
```

Maak de tokenmap en het tokenbestand aan onder de statusmap van OpenClaw:

```bash
mkdir -p ~/.openclaw/credentials/onepassword
chmod 700 ~/.openclaw/credentials/onepassword
printf '%s' "$OP_SERVICE_ACCOUNT_TOKEN" > \
  ~/.openclaw/credentials/onepassword/service-account-token
chmod 600 ~/.openclaw/credentials/onepassword/service-account-token
unset OP_SERVICE_ACCOUNT_TOKEN
```

Wanneer `OPENCLAW_STATE_DIR` is ingesteld, vervang je `~/.openclaw` door die map.
De plugin waarschuwt eenmaal wanneer het tokenbestand leesbaar of beschrijfbaar is voor de groep of
andere gebruikers.

## Geregistreerde geheimen configureren

Voeg pluginconfiguratie toe aan `openclaw.json`:

```jsonc
{
  "plugins": {
    "entries": {
      "onepassword": {
        "enabled": true,
        "config": {
          "vault": "Automation",
          "defaultPolicy": "approve",
          "cacheTtlSeconds": 300,
          "grantTtlHours": 720,
          "opTimeoutMs": 15000,
          "items": {
            "repository-token": {
              "item": "Repository automation token",
              "field": "credential",
              "policy": "approve",
              "description": "Token for repository automation",
            },
            "model-key": {
              "item": "Model provider key",
              "vault": "Agent credentials",
              "policy": "auto",
            },
          },
        },
      },
    },
  },
}
```

Slugs gebruiken kleine letters, cijfers en koppeltekens, beginnen met een letter of
cijfer en bevatten maximaal 64 tekens. Een registry kan maximaal 32
slugs bevatten; beschrijvingen kunnen maximaal 200 tekens bevatten. `field` accepteert één veldlabel
of ID, mag geen komma bevatten en is standaard `credential`.
Een `vault` op itemniveau overschrijft de standaardkluis. `opBin` kan een absoluut
pad naar het uitvoerbare bestand `op` instellen; anders zoekt de plugin `op` op via `PATH`.
Itemtitels mogen niet met een koppelteken beginnen.

## De agenttool gebruiken

De toolnaam is `onepassword`.

Geef geregistreerde slugs weer:

```json
{ "action": "list" }
```

Het resultaat bevat alleen de slug, beschrijving, het beleid en of er een permanente
toekenning actief is. Het bevat nooit een geheime waarde en raadpleegt 1Password niet.

Vraag één geheim op:

```json
{
  "action": "get",
  "slug": "repository-token",
  "reason": "Authenticate the requested repository operation"
}
```

`reason` is verplicht, mag niet leeg zijn en is beperkt tot 300 tekens. Een
geslaagde `get` retourneert de waarde plus de geconfigureerde slug, itemtitel en
het veldlabel.

Het toolschema declareert ook een interne parameter `authorizationNonce`. De
beleidslaag injecteert deze na beoordeling van het verzoek om de autorisatie
aan de uitvoerende toolaanroep door te geven. Stel deze nooit handmatig in: de beleidshook overschrijft
elke opgegeven waarde en een onbekende waarde laat het verzoek mislukken.

## Beleidsniveaus en goedkeuringen

- `auto`: onmiddellijk ophalen en het verzoek auditen.
- `deny`: het verzoek blokkeren en auditen.
- `approve`: een niet-verlopen permanente toekenning gebruiken of een persoon vragen om eenmalig
  of altijd toestemming te geven, of te weigeren.

Eenmalig toestaan autoriseert alleen de huidige toolaanroep. Altijd toestaan schrijft een permanente
toekenning voor die agent en slug naar SQLite; andere agents moeten hun eigen
goedkeuring ontvangen. OpenClaw biedt altijd toestaan alleen aan wanneer de aanroeper een concrete agentidentiteit
heeft. De toekenning verloopt na `grantTtlHours`, standaard 720 uur.
Een niet-beantwoorde of verlopen goedkeuring weigert het verzoek; de maximale wachttijd voor
goedkeuring is 600 seconden. De plugin bewaart maximaal 1.024 permanente toekenningen; bij die
grens wordt de oudste toekenning verwijderd en moet de betreffende agent de volgende toegang goedkeuren.

Elke beoordeelde autorisatie is eenmalig bruikbaar en wordt via de gedeelde SQLite-status
doorgegeven aan de uitvoerende toolaanroep, zodat de overdracht ook werkt wanneer meer dan één
plugininstantie actief is in het Gateway-proces. Ongebruikte autorisaties verlopen
na het goedkeuringsvenster van 600 seconden.

De cache in het geheugen staat standaard op 300 seconden en is begrensd door de geconfigureerde
slugregistry. Stel `cacheTtlSeconds` in op `0` om deze uit te schakelen. Het beleid wordt vóór
elke cachezoekactie beoordeeld en cachetreffers worden geaudit. Herladen van de runtimeconfiguratie
wordt van kracht bij elke beleids- en uitvoeringsgrens; als de plugin wordt uitgeschakeld of
een slug wordt verwijderd, geweigerd of naar een ander doel wordt verwezen, worden openstaande autorisaties en
gecachete waarden ongeldig.

## Status en auditgeschiedenis bekijken

Toon de gereedheid en registry-aantallen:

```bash
openclaw onepassword status
```

Dit rapporteert of het tokenbestand bestaat, of `op` is gevonden en via welk pad,
het aantal geregistreerde items en de aantallen per beleid. Het leest of toont nooit het
token of geheime waarden.

Toon de 50 meest recente auditrijen:

```bash
openclaw onepassword audit
openclaw onepassword audit --limit 100
```

Rijen worden met de nieuwste eerst weergegeven en tonen tijdstempel, agent, slug, resultaat, een `errorCode`
wanneer de poging is mislukt, en een afgekorte reden. De reden wordt opgeslagen zoals
opgegeven; de broker voegt de opgehaalde waarde nooit toe aan het auditlogboek.

## Gedrag van de 1Password-CLI

Bij elke cachemisser wordt `op item get` uitgevoerd met het geconfigureerde item, de kluis en de exacte
veldselector, JSON-uitvoer, een begrensde time-out en `--cache=false`. Het childproces
ontvangt alleen dat veld in plaats van het volledige item. Alleen
`OP_SERVICE_ACCOUNT_TOKEN` en `HOME` zijn aanwezig in de omgeving van het childproces.

De plugin doet één poging. Bij `RATE_LIMITED`-fouten moet worden gewacht
voordat een agent later opnieuw een verzoek doet; de plugin maakt geen automatische lus voor nieuwe pogingen.

## Foutcodes

Mislukte pogingen bevatten één afgebakende foutcode in het toolresultaat en de auditrij.

1Password-toegangsfouten:

| Code              | Betekenis                                                        |
| ----------------- | ---------------------------------------------------------------- |
| `TOKEN_MISSING`   | Tokenbestand ontbreekt of is leeg                                |
| `OP_NOT_FOUND`    | Binair bestand `op` kon niet worden gevonden                    |
| `ITEM_NOT_FOUND`  | Geconfigureerd item bevindt zich niet in de kluis                |
| `FIELD_NOT_FOUND` | Geconfigureerd veld bevindt zich niet op het item; beschikbare labels worden vermeld |
| `RATE_LIMITED`    | Limiet voor het 1Password-serviceaccount bereikt                 |
| `AUTH_FAILED`     | Authenticatie van het serviceaccount mislukt                     |
| `TIMEOUT`         | `op` overschreed `opTimeoutMs`                               |
| `OP_ERROR`        | Elke andere `op`-fout of ongeldige uitvoer                     |

Beleids- en validatiefouten:

| Code                                               | Betekenis                                                                    |
| -------------------------------------------------- | ---------------------------------------------------------------------------- |
| `INVALID_ACTION`, `INVALID_REASON`, `INVALID_SLUG` | Invoervalidatie van verzoek mislukt                                          |
| `UNKNOWN_SLUG`                                     | Slug staat niet in de geconfigureerde registry                               |
| `TOOL_CALL_ID_MISSING`                             | Aanroep is zonder toolaanroep-ID binnengekomen                               |
| `POLICY_NOT_EVALUATED`                             | Geen overeenkomende autorisatie voor deze aanroep; het verzoek is niet beleidsmatig goedgekeurd |
| `POLICY_CHANGED`                                   | Configuratie is gewijzigd tussen goedkeuring en uitvoering                   |
| `GRANT_EXPIRED`                                    | Permanente toekenning is vóór de uitvoering verlopen                        |
| `APPROVAL_CANCELLED`                               | De uitvoering is afgebroken terwijl de goedkeuring in behandeling was       |
