---
read_when:
    - Werken aan functies voor het Tlon/Urbit-kanaal
summary: Ondersteuningsstatus, mogelijkheden en configuratie van Tlon/Urbit
title: Tlon
x-i18n:
    generated_at: "2026-07-27T05:44:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d742628d6cf9aaf82d79a8d96b1685229905e9452c9fc4d3a494d2dee8d69943
    source_path: channels/tlon.md
    workflow: 16
---

Tlon is een gedecentraliseerde messenger die op Urbit is gebouwd. OpenClaw maakt verbinding met je Urbit-ship en
reageert op DM's en groepschatberichten. Voor groepsreacties is standaard een @-vermelding vereist, met
autorisatieregels en daarbovenop een goedkeuringsflow voor de eigenaar.

Status: gebundelde plugin. DM's, groepsvermeldingen, threads, opgemaakte tekst, uploaden/downloaden van afbeeldingen en een
goedkeuringssysteem voor de eigenaar worden ondersteund. Reacties en peilingen niet.

## Gebundelde plugin

Tlon wordt gebundeld meegeleverd in huidige OpenClaw-releases; voor verpakte builds is geen afzonderlijke installatie nodig.

Installeer vanuit npm bij een oudere build of aangepaste installatie waarin de plugin niet is opgenomen:

```bash
openclaw plugins install @openclaw/tlon
```

Gebruik alleen de pakketnaam om de huidige releasetag te volgen. Zet een versie vast (`@openclaw/tlon@x.y.z`)
alleen voor reproduceerbare installaties.

Vanuit een lokale checkout:

```bash
openclaw plugins install ./path/to/local/tlon-plugin
```

Details: [Plugins](/nl/tools/plugin)

## Installatie

```bash
openclaw channels add --channel tlon --ship ~sampel-palnet --url https://your-ship-host --code lidlut-tabwed-pillex-ridrup
```

Of bewerk de configuratie rechtstreeks:

```json5
{
  channels: {
    tlon: {
      enabled: true,
      ship: "~sampel-palnet",
      url: "https://your-ship-host",
      code: "lidlut-tabwed-pillex-ridrup",
      ownerShip: "~your-main-ship", // aanbevolen: je ship, altijd geautoriseerd
    },
  },
}
```

Herstart de Gateway nadat je de configuratie rechtstreeks hebt bewerkt. Stuur de bot vervolgens een DM of vermeld deze met @ in een
groepskanaal.

## Duurzaamheid van inkomende berichten

OpenClaw slaat geaccepteerde Tlon-DM- en groepschatgebeurtenissen permanent op voordat ze naar de agent worden gestuurd. Openstaande of opnieuw uitvoerbare beurten overleven een herstart van de Gateway en het werk blijft per groepskanaal of directe gesprekspartner geserialiseerd. Stabiele Urbit-bericht-ID's onderdrukken ook een opnieuw aangeleverde gebeurtenis zolang de bijbehorende wachtrijrecord of bewaarde voltooiingsrecord bestaat.

Levering over de grens tussen wachtrij en agent vindt ten minste eenmaal plaats: een crash tijdens de overdracht kan een beurt opnieuw afspelen. Agentacties die externe neveneffecten veroorzaken, moeten daarom waar mogelijk idempotent blijven.

## Privé-/LAN-ships

OpenClaw blokkeert standaard privé-/interne hostnamen en IP-bereiken ter bescherming tegen SSRF. Als je
ship op een privénetwerk draait (localhost, LAN-IP, interne hostnaam), moet je dit expliciet toestaan:

```json5
{
  channels: {
    tlon: {
      url: "http://localhost:8080",
      network: {
        dangerouslyAllowPrivateNetwork: true,
      },
    },
  },
}
```

Dit geldt voor doelen zoals `http://localhost:8080`, `http://192.168.x.x:8080` en
`http://my-ship.local:8080`. Schakel dit alleen in voor een ship-URL die je vertrouwt; hiermee wordt de SSRF-
bescherming voor de HTTP-verzoeken van dat account uitgeschakeld.

<Note>
`channels.tlon.allowPrivateNetwork` (platte sleutel) is buiten gebruik gesteld. `openclaw doctor --fix` verplaatst deze automatisch naar
`channels.tlon.network.dangerouslyAllowPrivateNetwork`.
</Note>

## Groepskanalen

Zet kanalen handmatig vast of schakel automatische detectie in:

```json5
{
  channels: {
    tlon: {
      groupChannels: ["chat/~host-ship/general", "chat/~host-ship/support"],
      autoDiscoverChannels: true,
    },
  },
}
```

`autoDiscoverChannels` is standaard `false` wanneer deze niet in de configuratie is ingesteld; de installatiewizard stelt de
prompt standaard in op ja en schrijft `true` expliciet. Wanneer dit is ingeschakeld, vraagt OpenClaw bij het opstarten aangesloten groepen op,
bewaakt het nieuwe kanalen wanneer groepsuitnodigingen worden geaccepteerd en controleert het elke 2 minuten opnieuw.

## Toegangsbeheer

Toestaanlijst voor DM's (leeg = geen DM's toegestaan, tenzij de afzender `ownerShip` is):

```json5
{
  channels: {
    tlon: {
      dmAllowlist: ["~zod", "~nec"],
    },
  },
}
```

Groepsautorisatie is standaard `restricted` per kanaal. Stel `defaultAuthorizedShips` in als
basis en overschrijf dit per kanaalnest:

```json5
{
  channels: {
    tlon: {
      defaultAuthorizedShips: ["~zod"],
      authorization: {
        channelRules: {
          "chat/~host-ship/general": {
            mode: "restricted",
            allowedShips: ["~zod", "~nec"],
          },
          "chat/~host-ship/announcements": {
            mode: "open",
          },
        },
      },
    },
  },
}
```

Zodra de bot binnen een thread heeft geantwoord, blijft deze op latere berichten in die thread reageren
zonder nog een vermelding te vereisen.

Stel `channels.tlon.implicitMentions.threadParticipation: false` in om voor die vervolgberichten een nieuwe expliciete vermelding
te vereisen. Overschrijvingen voor accounts gebruiken `channels.tlon.accounts.<id>.implicitMentions`. Tlon
produceert momenteel geen `replyToBot`- of `quotedBot`-feiten, dus die vlaggen hebben hier geen effect.

## Eigenaar en goedkeuringssysteem

```json5
{
  channels: {
    tlon: {
      ownerShip: "~your-main-ship",
    },
  },
}
```

De ship van de eigenaar is overal geautoriseerd: DM-uitnodigingen worden altijd automatisch geaccepteerd, groepsuitnodigingen worden
altijd automatisch geaccepteerd en kanaalberichten slagen altijd voor de autorisatie. De eigenaar hoeft niet
in `dmAllowlist`, `defaultAuthorizedShips` of `groupInviteAllowlist` te staan.

Wanneer `ownerShip` is ingesteld, worden ongeautoriseerde verzoeken niet alleen verwijderd — ze worden als openstaande
goedkeuring in de wachtrij geplaatst en de eigenaar ontvangt een DM:

- DM-verzoeken van ships die niet op `dmAllowlist` staan
- Vermeldingen in kanalen waarin de afzender niet door de autorisatie komt
- Groepsuitnodigingen van ships die niet op `groupInviteAllowlist` staan (wanneer automatisch accepteren is uitgeschakeld, of ingeschakeld maar de
  uitnodiger niet op de toestaanlijst staat)

De eigenaar antwoordt per DM om op een verzoek te reageren:

| Antwoord van eigenaar         | Effect                                               |
| ---------------------------- | ---------------------------------------------------- |
| `approve` / `deny` / `block` | Handelt de meest recente openstaande goedkeuring af |
| `approve <id>` / `deny <id>` | Handelt een specifieke goedkeuring af op basis van ID |
| `block`                      | Blokkeert de ship ook systeem-eigen, zodat deze niet opnieuw verbinding kan maken |
| `unblock ~ship`              | Maakt een systeem-eigen blokkering ongedaan          |
| `blocked`                    | Toont de momenteel geblokkeerde ships                |
| `pending`                    | Toont openstaande goedkeuringsverzoeken              |

Zonder geconfigureerde `ownerShip` worden ongeautoriseerde DM's en kanaalvermeldingen gewoon verwijderd en gelogd;
er verschijnt geen goedkeuringsprompt.

## Instellingen voor automatisch accepteren

Accepteer automatisch DM-uitnodigingen van ships die al op `dmAllowlist` staan (de eigenaar wordt altijd automatisch geaccepteerd,
ongeacht deze vlag):

```json5
{
  channels: {
    tlon: {
      autoAcceptDmInvites: true,
    },
  },
}
```

Accepteer automatisch groepsuitnodigingen van een toestaanlijst (standaard weigeren: met `autoAcceptGroupInvites: true` en
een lege `groupInviteAllowlist` wordt geen enkele uitnodiging van iemand anders dan de eigenaar geaccepteerd):

```json5
{
  channels: {
    tlon: {
      autoAcceptGroupInvites: true,
      groupInviteAllowlist: ["~zod"],
    },
  },
}
```

## Hot-reload via de Urbit-instellingenopslag

De meeste bovenstaande instellingen (`dmAllowlist`, `groupInviteAllowlist`, `groupChannels`,
`defaultAuthorizedShips`, `autoDiscoverChannels`, `autoAcceptDmInvites`,
`autoAcceptGroupInvites`, `ownerShip`, `showModelSignature`) worden bij de eerste uitvoering gespiegeld naar de
`%settings`-agent van de ship (desk `moltbot`, bucket `tlon`) en vervolgens live daaruit gelezen,
zodat wijzigingen die via een Landscape-client of de instellingencommando's van de gebundelde skill worden aangebracht, zonder
herstart van de Gateway worden toegepast. `channelRules` en openstaande goedkeuringen worden daar ook als JSON opgeslagen. De bestandsconfiguratie
blijft de bron van waarheid voor waarden die nooit naar de instellingenopslag zijn geschreven.

## Bezorgingsdoelen (CLI/cron)

Gebruik met `openclaw message send` of cron-bezorging:

- DM: `~sampel-palnet` of `dm/~sampel-palnet`
- Groep: `chat/~host-ship/channel` of `group:~host-ship/channel`

## Gebundelde skill

De plugin bundelt [`@tloncorp/tlon-skill`](https://github.com/tloncorp/tlon-skill), een CLI voor
rechtstreekse Urbit-bewerkingen, die automatisch beschikbaar is zodra de plugin is geïnstalleerd:

- **Activiteit**: vermeldingen, antwoorden, ongelezen berichten
- **Kanalen**: weergeven, maken, hernoemen
- **Contacten**: profielen weergeven/ophalen/bijwerken
- **Groepen**: maken, deelnemen, uitnodigings-/aanvraagflows, rollen
- **Hooks**: kanaalhooks beheren
- **Berichten**: geschiedenis, zoeken
- **DM's**: verzenden, reageren, accepteren/weigeren
- **Berichten**: reageren, verwijderen
- **Notitieboek**: publiceren naar dagboekkanalen
- **Instellingen**: pluginconfiguratie via hot-reload toepassen met de bovenstaande instellingenopslag

## Mogelijkheden

| Functie         | Status                                        |
| --------------- | --------------------------------------------- |
| Directe berichten | Ondersteund                                  |
| Groepen/kanalen | Ondersteund (standaard alleen na vermelding)  |
| Threads         | Ondersteund (blijft reageren nadat de bot is deelgenomen) |
| Opgemaakte tekst | Markdown geconverteerd naar het systeemeigen formaat van Tlon |
| Afbeeldingen    | Inkomend gedownload, uitgaand geüpload        |
| Reacties        | Alleen via de [gebundelde skill](#bundled-skill) |
| Peilingen       | Niet ondersteund                              |
| Systeemeigen commando's | Standaard alleen voor de eigenaar     |

## Problemen oplossen

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
```

Veelvoorkomende fouten:

- **DM's genegeerd**: afzender staat niet in `dmAllowlist` en er is geen `ownerShip` geconfigureerd voor de goedkeuringsflow.
- **Groepsberichten genegeerd**: kanaal is niet gedetecteerd/vastgezet, of de afzender slaagt niet voor de autorisatie en er is geen
  `ownerShip` om een goedkeuring in de wachtrij te plaatsen.
- **Verbindingsfouten**: controleer of de ship-URL bereikbaar is; stel
  `network.dangerouslyAllowPrivateNetwork` in voor lokale ships.
- **Authenticatiefouten**: aanmeldcodes rouleren — kopieer de huidige code van je ship.

## Configuratiereferentie

Volledige configuratie: [Configuratie](/nl/gateway/configuration)

| Sleutel                                                | Betekenis                                                      |
| ------------------------------------------------------ | -------------------------------------------------------------- |
| `channels.tlon.enabled`                                | Opstarten van kanaal in-/uitschakelen.                         |
| `channels.tlon.ship`                                   | Urbit-shipnaam van de bot (bijv. `~sampel-palnet`).            |
| `channels.tlon.url`                                    | Ship-URL (bijv. `https://sampel-palnet.tlon.network`).                            |
| `channels.tlon.code`                                   | Aanmeldcode van de ship.                                       |
| `channels.tlon.network.dangerouslyAllowPrivateNetwork` | Ship-URL's op localhost/LAN toestaan (expliciete SSRF-toestemming). |
| `channels.tlon.ownerShip`                              | Ship van eigenaar: altijd geautoriseerd, ontvangt goedkeuringsverzoeken. |
| `channels.tlon.dmAllowlist`                            | Ships die een DM mogen sturen (leeg = alleen eigenaar).        |
| `channels.tlon.autoAcceptDmInvites`                    | Automatisch DM's accepteren van ships in `dmAllowlist`.       |
| `channels.tlon.autoAcceptGroupInvites`                 | Automatisch groepsuitnodigingen accepteren van `groupInviteAllowlist`. |
| `channels.tlon.groupInviteAllowlist`                   | Ships waarvan groepsuitnodigingen automatisch worden geaccepteerd. |
| `channels.tlon.autoDiscoverChannels`                   | Aangesloten groepskanalen automatisch detecteren (standaard: `false`). |
| `channels.tlon.implicitMentions.threadParticipation`   | Vervolgberichten in threads waaraan is deelgenomen de vermeldingsvereiste laten omzeilen. |
| `channels.tlon.groupChannels`                          | Handmatig vastgezette kanaalnesten.                            |
| `channels.tlon.defaultAuthorizedShips`                 | Ships die voor alle kanalen zijn geautoriseerd (gebruikt wanneer geen regel overeenkomt). |
| `channels.tlon.authorization.channelRules`             | Autorisatiemodus en toestaanlijst per kanaalnest.              |
| `channels.tlon.showModelSignature`                     | `_[Generated by <model>]_` aan antwoorden toevoegen.                   |
| `channels.tlon.responsePrefix`                         | Statisch voorvoegsel dat vóór uitgaande antwoorden wordt geplaatst. |
| `channels.tlon.accounts.<id>`                          | Aanvullende benoemde accounts (configuraties met meerdere ships). |

## Opmerkingen

- Voor antwoorden in groepen is een @-vermelding vereist (bijv. `~your-bot-ship`), tenzij de bot al aan die thread deelneemt.
- Antwoorden op threads worden in de thread geplaatst; de bot krijgt ook de laatste 10 berichten uit de threadcontext voorgevoegd
  voor de agent.
- Tekst met opmaak (vet, cursief, code, koppen, lijsten) wordt omgezet naar de systeemeigen indeling van Tlon.
- Als je een inkomend bericht verstuurt waarin om een samenvatting van een kanaal wordt gevraagd (bijvoorbeeld "vat dit
  kanaal samen"), wordt een ingebouwde samenvatting van de geschiedenis geactiveerd in plaats van de normale antwoordflow.

## Gerelateerd

- [Overzicht van kanalen](/nl/channels) — alle ondersteunde kanalen
- [Koppelen](/nl/channels/pairing) — DM-authenticatie en koppelingsflow
- [Groepen](/nl/channels/groups) — gedrag van groepschats en vereiste vermeldingen
- [Kanaalroutering](/nl/channels/channel-routing) — sessieroutering voor berichten
- [Beveiliging](/nl/gateway/security) — toegangsmodel en beveiliging
