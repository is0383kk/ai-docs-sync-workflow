---
read_when:
    - Zalo Personal instellen voor OpenClaw
    - Problemen met de login of berichtenstroom van Zalo Personal oplossen
summary: Ondersteuning voor persoonlijke Zalo-accounts via native zca-js (inloggen met QR-code), mogelijkheden en configuratie
title: Zalo persoonlijk
x-i18n:
    generated_at: "2026-07-27T06:06:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 09cecad1a9a5b34b932c5e68e2b3164b360fb6af1dcd2fd5b5979d1b2a1bd62b
    source_path: channels/zalouser.md
    workflow: 16
---

Status: experimenteel. Deze integratie automatiseert een **persoonlijk Zalo-account** via native `zca-js`, in-process, zonder externe CLI-binary.

<Warning>
Dit is een onofficiële integratie en kan leiden tot opschorting of blokkering van het account. Gebruik op eigen risico.
</Warning>

## Installeren

Zalo Personal is een officiële externe Plugin en is niet gebundeld in de kern. Installeer deze vóór gebruik:

```bash
openclaw plugins install @openclaw/zalouser
```

- Een versie vastzetten: `openclaw plugins install @openclaw/zalouser@<version>`
- Vanuit een broncheckout: `openclaw plugins install ./path/to/local/zalouser-plugin`
- Details: [Plugins](/nl/tools/plugin)

## Snel instellen

1. Installeer de Plugin (hierboven).
2. Log in (QR, op de Gateway-machine):
   - `openclaw channels login --channel zalouser`
   - Scan de QR-code met de mobiele Zalo-app.
3. Schakel het kanaal in:

```json5
{
  channels: {
    zalouser: {
      enabled: true,
      dmPolicy: "pairing",
    },
  },
}
```

4. Start de Gateway opnieuw (of voltooi de installatie).
5. DM-toegang gebruikt standaard koppeling; keur de koppelingscode bij het eerste contact goed.

## Wat het is

- Draait volledig in-process via de bibliotheek `zca-js` (geen externe binary `zca`/`openzca`).
- Gebruikt native eventlisteners (`message`, `error`) om inkomende berichten te ontvangen.
- Verstuurt antwoorden rechtstreeks via de JS-API (tekst/media/link).
- Ontworpen voor gebruiksscenario's met een 'persoonlijk account' waarin de Zalo Bot API niet beschikbaar is.

## Naamgeving

De kanaal-id is `zalouser` om duidelijk te maken dat hiermee een **persoonlijk Zalo-gebruikersaccount** wordt geautomatiseerd (onofficieel). `zalo` is gereserveerd voor een mogelijke toekomstige officiële Zalo-API-integratie.

## ID's vinden (directory)

```bash
openclaw directory self --channel zalouser
openclaw directory peers list --channel zalouser --query "name"
openclaw directory groups list --channel zalouser --query "work"
```

## Limieten

- Uitgaande tekst wordt opgesplitst in delen van 2000 tekens (limiet van de Zalo-client).
- Streaming wordt niet ondersteund.
- ID's van voltooide inkomende berichten worden 30 dagen bewaard, met een maximum van de 1000 meest recente vermeldingen per account.

## Duurzaamheid van inkomende berichten

OpenClaw slaat elke onbewerkte `zca-js`-berichtcallback op voordat deze wordt verwerkt. Berichten in behandeling worden na een herstart van de Gateway hervat vanuit de accountwachtrij en de verwerking blijft per rechtstreeks gesprek of groep geserialiseerd.

De socketlistener `zca-js` biedt geen ontvangstbevestiging en speelt oude berichten na opnieuw verbinden niet automatisch opnieuw af. De duurzame wachtrij beschermt daarom tegen het lokale crashvenster nadat een callback OpenClaw heeft bereikt; een bericht dat nooit door de socket is afgeleverd, kan hiermee niet worden hersteld. Tombstones voor opnieuw afspelen dienen vooral als beveiliging tegen een herhaalde callback met dezelfde Zalo-bericht-id.

## Toegangsbeheer (DM's)

`channels.zalouser.dmPolicy`: `pairing | allowlist | open | disabled` (standaard: `pairing`).

`channels.zalouser.allowFrom` moet stabiele Zalo-gebruikers-ID's gebruiken. Er kan ook worden verwezen naar statische toegangsgroepen voor afzenders (`accessGroup:<name>`). Tijdens de interactieve configuratie kunnen ingevoerde namen via de in-process contactzoekfunctie van de Plugin naar ID's worden omgezet.

Als een onbewerkte naam in de configuratie blijft staan, wordt deze bij het opstarten alleen omgezet wanneer `channels.zalouser.dangerouslyAllowNameMatching: true` is ingeschakeld. Zonder die expliciete inschakeling controleren runtimecontroles voor afzenders uitsluitend ID's en worden onbewerkte namen voor autorisatie genegeerd.

Goedkeuren via:

- `openclaw pairing list zalouser`
- `openclaw pairing approve zalouser <code>`

## Groepstoegang (optioneel)

- Standaard: `channels.zalouser.groupPolicy = "allowlist"` (groepen vereisen een expliciete vermelding in de toelatingslijst).
- Alle groepen openen: `channels.zalouser.groupPolicy = "open"`.
- Alle groepen blokkeren: `channels.zalouser.groupPolicy = "disabled"`.
- Met `groupPolicy = "allowlist"`:
  - Sleutels voor `channels.zalouser.groups` moeten stabiele groeps-ID's zijn; namen worden bij het opstarten alleen naar ID's omgezet wanneer `channels.zalouser.dangerouslyAllowNameMatching: true` is ingeschakeld.
  - `channels.zalouser.groupAllowFrom` bepaalt welke afzenders in toegestane groepen de bot kunnen activeren; met `accessGroup:<name>` kan naar statische toegangsgroepen voor afzenders worden verwezen.
- De configuratiewizard kan om groepstoelatingslijsten vragen.
- Overeenkomsten met de groepstoelatingslijst worden standaard uitsluitend op basis van ID bepaald. Niet-omgezette namen worden voor autorisatie genegeerd, tenzij `channels.zalouser.dangerouslyAllowNameMatching: true` is ingeschakeld.
- `channels.zalouser.dangerouslyAllowNameMatching: true` is een noodcompatibiliteitsmodus die veranderlijke naamomzetting bij het opstarten en runtimevergelijking van groepsnamen opnieuw inschakelt.
- `groupAllowFrom` valt voor normale groepsberichten **niet** terug op `allowFrom`: als deze waarde voor een groep op de toelatingslijst leeg blijft, wordt die groep voor elke afzender geopend. Geautoriseerde beheeropdrachten (bijvoorbeeld `/new`) vormen de uitzondering; controles van de afzender van opdrachten vallen terug op `allowFrom` wanneer `groupAllowFrom` leeg is.

Voorbeeld:

```json5
{
  channels: {
    zalouser: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["1471383327500481391"],
      groups: {
        "123456789": { enabled: true },
        "Work Chat": { enabled: true },
      },
    },
  },
}
```

<Note>
`channels.zalouser.groups.<id>.allow` is een verouderde veldnaam; de huidige configuratie gebruikt `enabled`. `openclaw doctor --fix` migreert `allow` automatisch naar `enabled`.
</Note>

### Vermeldingsvereiste voor groepen

- `channels.zalouser.groups.<group>.requireMention` bepaalt of voor groepsantwoorden een vermelding vereist is.
- Volgorde van omzetting: groeps-id -> alias `group:<id>` -> groepsnaam/slug (kandidaten op basis van namen zijn alleen van toepassing wanneer `dangerouslyAllowNameMatching: true`) -> `*` -> standaard (`true`).
- Geldt zowel voor groepen op de toelatingslijst als voor de open groepsmodus.
- Het citeren van een botbericht geldt als een impliciete vermelding voor groepsactivering.
- Geautoriseerde beheeropdrachten (bijvoorbeeld `/new`) kunnen de vermeldingsvereiste omzeilen.
- Wanneer een groepsbericht wordt overgeslagen omdat een vermelding vereist is, slaat OpenClaw het op als groepsgeschiedenis in behandeling en neemt het op in het volgende verwerkte groepsbericht.
- Limiet voor groepsgeschiedenis: `channels.zalouser.historyLimit`, vervolgens `messages.groupChat.historyLimit`, en daarna een terugvalwaarde van `50`.

Voorbeeld:

```json5
{
  channels: {
    zalouser: {
      groupPolicy: "allowlist",
      groups: {
        "*": { enabled: true, requireMention: true },
        "Work Chat": { enabled: true, requireMention: false },
      },
    },
  },
}
```

## Meerdere accounts

Accounts worden toegewezen aan `zalouser`-profielen in de OpenClaw-status. Voorbeeld:

```json5
{
  channels: {
    zalouser: {
      enabled: true,
      defaultAccount: "default",
      accounts: {
        work: { enabled: true, profile: "work" },
      },
    },
  },
}
```

## Omgevingsvariabelen

Profielselectie kan ook uit omgevingsvariabelen komen:

| Variabele                | Doel                                                                    |
| ------------------ | -------------------------------------------------------------------------- |
| `ZALOUSER_PROFILE` | Profielnaam die wordt gebruikt wanneer geen `profile` is ingesteld in de kanaal- of accountconfiguratie. |
| `ZCA_PROFILE`      | Verouderde terugvalwaarde, alleen gebruikt wanneer `ZALOUSER_PROFILE` niet is ingesteld.             |

Profielnamen selecteren de opgeslagen Zalo-inloggegevens in de OpenClaw-status. Volgorde van omzetting:

1. Expliciete `profile` in de configuratie.
2. `ZALOUSER_PROFILE`.
3. `ZCA_PROFILE`.
4. De account-id voor niet-standaardaccounts, of `default` voor het standaardaccount.

Voor configuraties met meerdere accounts verdient het de voorkeur om `profile` voor elk account in de configuratie in te stellen, zodat één omgevingsvariabele er niet toe leidt dat meerdere accounts dezelfde inlogsessie delen.

## Typen, reacties en ontvangstbevestigingen

- OpenClaw verzendt een typegebeurtenis voordat een antwoord wordt verstuurd (naar beste vermogen).
- De berichtreactieactie `react` wordt ondersteund voor `zalouser` in kanaalacties.
  - Gebruik `remove: true` om een specifieke reactie-emoji van een bericht te verwijderen.
  - Semantiek van reacties: [Reacties](/nl/tools/reactions)
- Voor inkomende berichten die gebeurtenismetadata bevatten, verzendt OpenClaw bevestigingen voor afgeleverd + gezien (naar beste vermogen).

## Probleemoplossing

**Inloggen blijft niet behouden:**

- `openclaw channels status --probe`
- Opnieuw inloggen: `openclaw channels logout --channel zalouser && openclaw channels login --channel zalouser`

**Naam in toelatingslijst/groepsnaam is niet omgezet:**

- Gebruik numerieke ID's in `allowFrom`/`groupAllowFrom` en stabiele groeps-ID's in `groups`. Als je bewust exacte namen van vrienden/groepen nodig hebt, schakel je `channels.zalouser.dangerouslyAllowNameMatching: true` in.

**Geüpgraded vanaf een oude externe `zca`-installatie/installatie op basis van de CLI:**

- Verwijder alle aannames over een extern `zca`-proces; het kanaal draait nu volledig in-process via `zca-js`, zonder externe CLI-binary.

## Gerelateerd

- [Overzicht van kanalen](/nl/channels) - alle ondersteunde kanalen
- [Koppeling](/nl/channels/pairing) - DM-authenticatie en koppelingsflow
- [Groepen](/nl/channels/groups) - gedrag van groepschats en vermeldingsvereiste
- [Kanaalroutering](/nl/channels/channel-routing) - sessieroutering voor berichten
- [Beveiliging](/nl/gateway/security) - toegangsmodel en beveiliging aanscherpen
