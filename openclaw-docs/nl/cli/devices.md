---
read_when:
    - Je keurt koppelingsverzoeken van apparaten goed
    - Je moet apparaattokens roteren of intrekken
summary: CLI-referentie voor `openclaw devices` (apparaatkoppeling + tokenrotatie/-intrekking)
title: Apparaten
x-i18n:
    generated_at: "2026-07-27T05:27:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 83fb10f7a484fec06bfa5e53ae50181b12a9724746176bbace330ec468235494
    source_path: cli/devices.md
    workflow: 16
---

# `openclaw devices`

Beheer aanvragen voor apparaatkoppeling en apparaatspecifieke tokens.

## Algemene opties

- `--url <url>`: WebSocket-URL van de Gateway (standaard `gateway.remote.url` indien geconfigureerd)
- `--token <token>`: Gateway-token (indien vereist)
- `--password <password>`: Gateway-wachtwoord (wachtwoordauthenticatie)
- `--timeout <ms>`: RPC-time-out
- `--json`: JSON-uitvoer (aanbevolen voor scripts)

<Warning>
Wanneer je `--url` instelt, valt de CLI niet terug op referenties uit de configuratie of omgeving. Geef `--token` of `--password` expliciet door, anders mislukt de opdracht.
</Warning>

## Opdrachten

### `openclaw devices list`

Geef openstaande koppelingsaanvragen en gekoppelde apparaten weer.

```bash
openclaw devices list
openclaw devices list --json
```

Bij een openstaande aanvraag voor een al gekoppeld apparaat toont de uitvoer de aangevraagde toegang naast de momenteel goedgekeurde toegang van het apparaat, zodat upgrades van bereik of rol zichtbaar zijn en niet op een verloren koppeling lijken.

Weergavenamen van gekoppelde apparaten gebruiken deze prioriteitsvolgorde: operatorlabel (`operatorLabel` uit `devices rename`), vervolgens `displayName` van de client, daarna `clientId` en ten slotte `deviceId`.

### `openclaw devices approve [requestId] [--latest]`

Keur een openstaande koppelingsaanvraag goed aan de hand van de exacte `requestId`. Als je `requestId` weglaat of `--latest` doorgeeft, wordt alleen een voorbeeld van de nieuwste openstaande aanvraag weergegeven en wordt de opdracht afgesloten (code 1); voer de opdracht opnieuw uit met de exacte aanvraag-ID om deze goed te keuren.

```bash
openclaw devices approve
openclaw devices approve <requestId>
openclaw devices approve --latest
```

<Note>
Als een apparaat opnieuw probeert te koppelen met gewijzigde authenticatiegegevens (rol, bereiken of openbare sleutel), vervangt OpenClaw de vorige openstaande vermelding door een nieuwe `requestId`. Voer vlak vóór de goedkeuring `openclaw devices list` uit om de huidige ID op te halen.
</Note>

Gedrag bij goedkeuring:

- Als het apparaat al is gekoppeld en bredere bereiken of een andere rol aanvraagt, behoudt OpenClaw de bestaande goedkeuring en maakt het een nieuwe openstaande upgradeaanvraag. Vergelijk `Requested` met `Approved` in `openclaw devices list`, of bekijk een voorbeeld met `--latest`, voordat je de aanvraag goedkeurt.
- Voor het goedkeuren van een rol `node` of een andere niet-operatorrol is `operator.admin` vereist. `operator.pairing` volstaat voor goedkeuringen van operatorapparaten, maar alleen wanneer de aangevraagde operatorbereiken binnen de eigen bereiken van de aanroeper blijven. Zie [Operatorbereiken](/nl/gateway/operator-scopes).
- Als `gateway.nodes.pairing.autoApproveCidrs` is geconfigureerd, kunnen eerste `role: node`-aanvragen van overeenkomende client-IP-adressen automatisch worden goedgekeurd voordat ze in deze lijst verschijnen. Dit is standaard uitgeschakeld en geldt nooit voor operator-/browserclients of upgradeaanvragen.
- `gateway.nodes.pairing.sshVerify` (standaard ingeschakeld) keurt eerste `role: node`-aanvragen automatisch goed wanneer de Gateway de apparaatsleutel via SSH bij de Node-host verifieert. Aanvragen kunnen daarom kort nadat ze verschijnen als goedgekeurd worden afgehandeld. Stel `sshVerify: false` in om SSH-verificatie uit te schakelen; dit staat los van `autoApproveCidrs`, dus schakel die optie ook uit als je alleen handmatig wilt koppelen.

### `openclaw devices reject <requestId>`

Wijs een openstaande aanvraag voor apparaatkoppeling af.

```bash
openclaw devices reject <requestId>
```

### `openclaw devices remove <deviceId>`

Verwijder één vermelding van een gekoppeld apparaat.

```bash
openclaw devices remove <deviceId>
openclaw devices remove <deviceId> --json
```

Een aanroeper die is geauthenticeerd met een token van een gekoppeld apparaat, kan alleen de vermelding van het **eigen** apparaat verwijderen. Voor het verwijderen van een ander apparaat is `operator.admin` vereist.

### `openclaw devices rename --device <id> --name <label>`

Wijs een operatorlabel toe aan een gekoppeld apparaat. Labels zijn status aan de eigenaarszijde: ze blijven behouden na reparaties van koppelingen en nieuwe rolgoedkeuringen, en wijzigen de stabiele `deviceId` niet.

```bash
openclaw devices rename --device <deviceId> --name "Kitchen Mac"
openclaw devices rename --device <deviceId> --name "Kitchen Mac" --json
```

- `--name` is vereist, wordt ontdaan van omringende witruimte, mag niet leeg zijn en is beperkt tot 64 tekens.
- Weergaveoppervlakken (CLI-lijst, inventaris van de Control UI) geven de voorkeur aan het operatorlabel boven de door de client gerapporteerde weergavenaam.
- Een aanroeper van een gekoppeld apparaat zonder beheerdersrechten kan alleen het **eigen** apparaat hernoemen. Voor het hernoemen van een ander apparaat is `operator.admin` vereist.

### `openclaw devices clear --yes [--pending]`

Wis gekoppelde apparaten in bulk. Afgeschermd door `--yes`.

```bash
openclaw devices clear --yes
openclaw devices clear --yes --pending
openclaw devices clear --yes --pending --json
```

`--pending` wijst ook alle openstaande koppelingsaanvragen af.

### `openclaw devices rotate --device <id> --role <role> [--scope <scope...>]`

Roteer een apparaattoken voor een rol en werk desgewenst de bereiken ervan bij.

```bash
openclaw devices rotate --device <deviceId> --role operator --scope operator.read --scope operator.write
```

- De doelrol moet al bestaan in het goedgekeurde koppelingscontract van dat apparaat; rotatie kan geen nieuwe, niet-goedgekeurde rol uitgeven.
- Als je `--scope` weglaat, worden bij latere herverbindingen de in de cache opgeslagen goedgekeurde bereiken van het opgeslagen token opnieuw gebruikt. Als je expliciete `--scope`-waarden doorgeeft, wordt de opgeslagen verzameling bereiken voor toekomstige herverbindingen met een gecachet token vervangen.
- Een aanroeper van een gekoppeld apparaat zonder beheerdersrechten kan alleen het token van het **eigen** apparaat roteren en de verzameling doelbereiken moet binnen de eigen operatorbereiken van de aanroeper blijven; rotatie kan geen token uitgeven of behouden dat bredere rechten heeft dan de aanroeper al bezit.

Retourneert rotatiemetadata als JSON. Als de aanroeper het eigen token roteert terwijl deze met dat apparaattoken is geauthenticeerd, bevat het antwoord het vervangende token zodat de client het vóór de herverbinding kan opslaan. Bij gedeelde rotaties of rotaties door een beheerder wordt het bearer-token nooit teruggestuurd.

### `openclaw devices revoke --device <id> --role <role>`

Trek een apparaattoken voor een rol in.

```bash
openclaw devices revoke --device <deviceId> --role node
```

Een aanroeper van een gekoppeld apparaat zonder beheerdersrechten kan alleen het token van het **eigen** apparaat intrekken. Voor het intrekken van het token van een ander apparaat is `operator.admin` vereist. De verzameling doelbereiken moet ook binnen de eigen operatorbereiken van de aanroeper vallen; aanroepers die alleen koppelingsrechten hebben, kunnen geen beheer-/schrijftokens van operators intrekken.

## Opmerkingen

- Voor deze opdrachten is het bereik `operator.pairing` (of `operator.admin`) vereist. Voor apparaatrollen die geen operator zijn, is altijd `operator.admin` vereist; zie [Operatorbereiken](/nl/gateway/operator-scopes).
- Tokenrotatie en -intrekking blijven binnen de goedgekeurde verzameling koppelingsrollen en de basislijn van bereiken van het apparaat. Een losstaande gecachete tokenvermelding verleent geen doel voor tokenbeheer.
- Voor tokensessies van gekoppelde apparaten is apparaatoverschrijdend beheer (`remove`, `rename`, `rotate`, `revoke`) beperkt tot het eigen apparaat, tenzij de aanroeper `operator.admin` heeft.
- Tokenrotatie retourneert een nieuw token (gevoelig) — behandel dit als een geheim.
- Als het koppelingsbereik niet beschikbaar is via lokale loopback en geen expliciete `--url` wordt doorgegeven, kunnen `list`/`approve` terugvallen op de lokale koppelingsstatus.

## Controlelijst voor herstel van tokenafwijkingen

Gebruik dit wanneer de Control UI of andere clients blijven mislukken met `AUTH_TOKEN_MISMATCH`, `AUTH_DEVICE_TOKEN_MISMATCH` of `AUTH_SCOPE_MISMATCH`.

1. Controleer de huidige bron van het Gateway-token:

   ```bash
   openclaw config get gateway.auth.token
   ```

2. Geef de gekoppelde apparaten weer en identificeer de ID van het getroffen apparaat:

   ```bash
   openclaw devices list
   ```

3. Roteer het operatortoken voor het getroffen apparaat:

   ```bash
   openclaw devices rotate --device <deviceId> --role operator
   ```

4. Als rotatie niet voldoende is, verwijder je de verouderde koppeling en keur je deze opnieuw goed:

   ```bash
   openclaw devices remove <deviceId>
   openclaw devices list
   openclaw devices approve <requestId>
   ```

5. Probeer de clientverbinding opnieuw met het huidige gedeelde token/wachtwoord.

Opmerkingen:

- Normale authenticatievolgorde bij herverbinding: eerst een expliciet gedeeld token/wachtwoord, vervolgens expliciete `deviceToken`, daarna het opgeslagen apparaattoken en ten slotte het bootstraptoken.
- Vertrouwd herstel met `AUTH_TOKEN_MISMATCH` kan tijdelijk zowel het gedeelde token als het opgeslagen apparaattoken samen verzenden voor één begrensde nieuwe poging.
- `AUTH_SCOPE_MISMATCH` betekent dat het apparaattoken is herkend, maar niet de aangevraagde verzameling bereiken bevat; herstel het goedkeuringscontract voor koppeling/bereiken voordat je de gedeelde Gateway-authenticatie wijzigt.

Gerelateerd:

- [Problemen met dashboardauthenticatie oplossen](/nl/web/dashboard#if-you-see-unauthorized-1008)
- [Problemen met de Gateway oplossen](/nl/gateway/troubleshooting#dashboard-control-ui-connectivity)

## Goedkeuring bij de eerste uitvoering van Paperclip / `openclaw_gateway`

Paperclip-agents die via de `openclaw_gateway`-adapter verbinding maken, doorlopen dezelfde goedkeuring voor apparaatkoppeling bij de eerste uitvoering als elke andere nieuwe client. Als Paperclip `openclaw_gateway_pairing_required` meldt, keur je het openstaande apparaat goed en probeer je het opnieuw.

```bash
openclaw devices approve --latest
```

Het voorbeeld toont de exacte `openclaw devices approve <requestId>`-opdracht; controleer de details en voer die opdracht vervolgens opnieuw uit met de aanvraag-ID om deze goed te keuren. Geef voor een externe Gateway of expliciete referenties dezelfde opties door bij zowel de voorbeeldweergave als de goedkeuring:

```bash
openclaw devices approve --latest --url <gateway-ws-url> --token <gateway-token>
```

Om te voorkomen dat je na elke herstart opnieuw goedkeuring moet geven, configureer je een permanente `adapterConfig.devicePrivateKeyPem` in Paperclip in plaats van bij elke uitvoering een nieuwe tijdelijke apparaatidentiteit te laten genereren:

```json
{
  "adapterConfig": {
    "devicePrivateKeyPem": "<ed25519-private-key-pkcs8-pem>"
  }
}
```

Als de goedkeuring blijft mislukken, voer je eerst `openclaw devices list` uit om te controleren of er een openstaande aanvraag bestaat.

## Gerelateerd

- [CLI-referentie](/nl/cli)
- [Nodes](/nl/nodes)
