---
read_when:
    - Twitch-chatintegratie instellen voor OpenClaw
sidebarTitle: Twitch
summary: 'Twitch-chatbot: installatie, referenties, toegangsbeheer, tokenvernieuwing'
title: Twitch
x-i18n:
    generated_at: "2026-07-27T04:48:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d827c742ded5fd0b071443dead27b975e2414419b0facb486d7f9c0c9800b060
    source_path: channels/twitch.md
    workflow: 16
---

Twitch-chatondersteuning via de chatinterface (IRC) van Twitch met de Twurple-client. OpenClaw meldt zich aan met een Twitch-botaccount, neemt per geconfigureerd account deel aan één kanaal en antwoordt in dat kanaal.

## Installeren

Twitch wordt geleverd als officiële plugin; het maakt geen deel uit van de kerninstallatie.

<Tabs>
  <Tab title="npm-register">
    ```bash
    openclaw plugins install @openclaw/twitch
    ```
  </Tab>
  <Tab title="Lokale checkout">
    ```bash
    openclaw plugins install ./path/to/local/twitch-plugin
    ```
  </Tab>
</Tabs>

`plugins install` registreert en activeert de plugin. Als je Twitch kiest tijdens `openclaw onboard` of `openclaw channels add`, wordt deze op aanvraag geïnstalleerd. Gebruik alleen de pakketnaam om de huidige release te volgen; zet alleen voor reproduceerbare installaties een exacte versie vast. Vereist OpenClaw 2026.4.10 of nieuwer.

Details: [Plugins](/nl/tools/plugin)

## Snelle configuratie

<Steps>
  <Step title="Installeer de plugin">
    Zie [Installeren](#install) hierboven.
  </Step>
  <Step title="Maak een Twitch-botaccount">
    Maak een afzonderlijk Twitch-account voor de bot (of gebruik een bestaand account).
  </Step>
  <Step title="Genereer aanmeldgegevens">
    Gebruik [Twitch Token Generator](https://twitchtokengenerator.com/):

    - Selecteer **Bot Token**
    - Controleer of de scopes `chat:read` en `chat:write` zijn geselecteerd
    - Kopieer de **Client ID** en **Access Token**

  </Step>
  <Step title="Zoek je Twitch-gebruikers-ID">
    Gebruik [https://www.streamweasels.com/tools/convert-twitch-username-to-user-id/](https://www.streamweasels.com/tools/convert-twitch-username-to-user-id/) om een gebruikersnaam naar een Twitch-gebruikers-ID om te zetten.
  </Step>
  <Step title="Configureer het token">
    - Omgevingsvariabele: `OPENCLAW_TWITCH_ACCESS_TOKEN=...` (alleen standaardaccount)
    - Of configuratie: `channels.twitch.accessToken`

    Als beide zijn ingesteld, heeft de configuratie voorrang (de omgevingsvariabele dient alleen als terugvaloptie voor het standaardaccount).

  </Step>
  <Step title="Start de Gateway">
    ```bash
    openclaw gateway run
    ```
  </Step>
</Steps>

<Warning>
Voeg toegangsbeheer toe (`allowFrom` of `allowedRoles`) om te voorkomen dat onbevoegde gebruikers de bot activeren. `requireMention` is standaard `true`.
</Warning>

Minimale configuratie:

```json5
{
  channels: {
    twitch: {
      enabled: true,
      username: "openclaw", // Twitch-account van de bot (voor authenticatie)
      accessToken: "oauth:abc123...", // OAuth-toegangstoken (of gebruik de omgevingsvariabele OPENCLAW_TWITCH_ACCESS_TOKEN)
      clientId: "xyz789...", // Client-ID uit Token Generator
      channel: "yourchannel", // Aan welke Twitch-kanaalchat moet worden deelgenomen (verplicht)
      allowFrom: ["123456789"], // (aanbevolen) Alleen jouw Twitch-gebruikers-ID
    },
  },
}
```

## Wat het is

- Een Twitch-kanaal dat eigendom is van de Gateway.
- Deterministische routering: antwoorden gaan altijd terug naar het Twitch-kanaal waaruit het bericht afkomstig is.
- Elk kanaal waaraan wordt deelgenomen, wordt toegewezen aan een geïsoleerde groepssessiesleutel `agent:<agentId>:twitch:group:<channel>`.
- `username` is het account van de bot (dat zich authenticeert), `channel` is de chatruimte waaraan wordt deelgenomen. Elke accountvermelding neemt aan precies één kanaal deel.
- Tokens werken met of zonder het voorvoegsel `oauth:`; OpenClaw normaliseert beide vormen (de configuratiewizard verwacht de vorm `oauth:`).

## Duurzaamheid van inkomende berichten

OpenClaw plaatst elk geaccepteerd Twitch-chatbericht duurzaam in de wachtrij vóór de normale verwerking. Berichten die in behandeling zijn of opnieuw kunnen worden geprobeerd, blijven behouden na een herstart van de Gateway, worden voor het geconfigureerde kanaal sequentieel verwerkt en gebruiken de bericht-ID van Twitch om dubbele wachtrijvermeldingen te onderdrukken zolang de actieve of bewaarde voltooiingsregistratie bestaat.

Twitch-chat speelt een `PRIVMSG` niet opnieuw af nadat de client het heeft geaccepteerd. Dit beschermt het lokale crashvenster tussen acceptatie en verwerking, maar kan geen berichten herstellen die vóór duurzame opname zijn gemist. Als het toevoegen aan de wachtrij zelf mislukt, registreert OpenClaw de fout; opnieuw verbinding maken vraagt Twitch niet om dat bericht opnieuw te verzenden.

## Tokenvernieuwing (optioneel)

Tokens van [Twitch Token Generator](https://twitchtokengenerator.com/) kunnen niet door OpenClaw worden vernieuwd. Genereer ze opnieuw wanneer ze zijn verlopen (ze blijven enkele uren geldig; er is geen appregistratie nodig).

Maak voor automatische vernieuwing je eigen app in de [Twitch Developer Console](https://dev.twitch.tv/console) en voeg het volgende toe:

```json5
{
  channels: {
    twitch: {
      clientSecret: "your_client_secret",
      refreshToken: "your_refresh_token",
    },
  },
}
```

Als beide zijn ingesteld, gebruikt de plugin een vernieuwende authenticatieprovider die tokens vóór het verlopen vernieuwt en elke vernieuwing registreert. Zonder `refreshToken` registreert deze `token refresh disabled (no refresh token)`; zonder `clientSecret` valt deze terug op een statisch (niet-vernieuwend) token.

## Ondersteuning voor meerdere accounts

Gebruik `channels.twitch.accounts` met aanmeldgegevens per account. Zie [Configuratie](/nl/gateway/configuration) voor het gedeelde patroon.

Voorbeeld (één botaccount in twee kanalen):

```json5
{
  channels: {
    twitch: {
      accounts: {
        channel1: {
          username: "openclaw",
          accessToken: "oauth:abc123...",
          clientId: "xyz789...",
          channel: "yourchannel",
        },
        channel2: {
          username: "openclaw",
          accessToken: "oauth:def456...",
          clientId: "uvw012...",
          channel: "secondchannel",
        },
      },
    },
  },
}
```

<Note>
Elke accountvermelding heeft een eigen `accessToken` nodig (de omgevingsvariabele geldt alleen voor het standaardaccount). Een account neemt aan precies één kanaal deel, dus voor deelname aan twee kanalen zijn twee accounts nodig. `channels.twitch.defaultAccount` bepaalt welk account het standaardaccount is.
</Note>

## Toegangsbeheer

`allowFrom` is een strikte toelatingslijst van Twitch-gebruikers-ID's. Wanneer deze is ingesteld, wordt `allowedRoles` genegeerd; laat `allowFrom` oningesteld om in plaats daarvan rolgebaseerde toegang te gebruiken.

**Beschikbare rollen:** `"moderator"`, `"owner"`, `"vip"`, `"subscriber"`, `"all"`.

<Tabs>
  <Tab title="Toelatingslijst met gebruikers-ID's (veiligst)">
    ```json5
    {
      channels: {
        twitch: {
          accounts: {
            default: {
              allowFrom: ["123456789", "987654321"],
            },
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="Rolgebaseerd">
    ```json5
    {
      channels: {
        twitch: {
          accounts: {
            default: {
              allowedRoles: ["moderator", "vip"],
            },
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="Vereiste voor @vermelding uitschakelen">
    Standaard is `requireMention` `true`. Om op alle toegestane berichten te reageren:

    ```json5
    {
      channels: {
        twitch: {
          accounts: {
            default: {
              requireMention: false,
            },
          },
        },
      },
    }
    ```

  </Tab>
</Tabs>

<Note>
**Waarom gebruikers-ID's?** Gebruikersnamen kunnen veranderen, waardoor imitatie mogelijk is. Gebruikers-ID's zijn permanent.

Zoek die van jou met de [omzetter van gebruikersnaam naar ID](https://www.streamweasels.com/tools/convert-twitch-username-to-user-id/).
</Note>

## Problemen oplossen

Voer eerst diagnostische opdrachten uit:

```bash
openclaw doctor
openclaw channels status --probe
```

<AccordionGroup>
  <Accordion title="Bot reageert niet op berichten">
    - **Controleer het toegangsbeheer:** Zorg dat je gebruikers-ID in `allowFrom` staat, of verwijder tijdelijk `allowFrom` en stel `allowedRoles: ["all"]` in om te testen.
    - **Controleer de vermeldingspoort:** Met `requireMention: true` (standaard) moeten berichten de gebruikersnaam van de bot met een @vermelden.
    - **Controleer of de bot in het kanaal aanwezig is:** De bot neemt alleen deel aan het kanaal dat in `channel` is opgegeven.

  </Accordion>
  <Accordion title="Tokenproblemen">
    Fouten zoals "Kan geen verbinding maken" of authenticatiefouten:

    - Controleer of `accessToken` de waarde van het OAuth-toegangstoken is (het voorvoegsel `oauth:` is optioneel)
    - Controleer of het token de scopes `chat:read` en `chat:write` heeft
    - Controleer bij gebruik van tokenvernieuwing of `clientSecret` en `refreshToken` zijn ingesteld

  </Accordion>
  <Accordion title="Tokenvernieuwing werkt niet">
    Controleer de logboeken op vernieuwingsgebeurtenissen:

    ```text
    Omgevingsvariabel tokenbron wordt gebruikt voor mybot
    Toegangstoken vernieuwd voor gebruiker 123456 (verloopt over 14400s)
    ```

    Als je `token refresh disabled (no refresh token)` ziet:

    - Zorg dat `clientSecret` is opgegeven
    - Zorg dat `refreshToken` is opgegeven

  </Accordion>
</AccordionGroup>

## Configuratie

### Accountconfiguratie

<ParamField path="username" type="string" required>
  Gebruikersnaam van de bot (het account dat zich authenticeert).
</ParamField>
<ParamField path="accessToken" type="string" required>
  OAuth-toegangstoken met `chat:read` en `chat:write` (configuratie of omgevingsvariabele voor het standaardaccount).
</ParamField>
<ParamField path="clientId" type="string" required>
  Twitch-client-ID (uit Token Generator of je app). Optioneel in het schema, maar vereist om verbinding te maken.
</ParamField>
<ParamField path="channel" type="string" required>
  Kanaal waaraan moet worden deelgenomen.
</ParamField>
<ParamField path="enabled" type="boolean" default="true">
  Activeer dit account.
</ParamField>
<ParamField path="clientSecret" type="string">
  Optioneel: voor automatische tokenvernieuwing.
</ParamField>
<ParamField path="refreshToken" type="string">
  Optioneel: voor automatische tokenvernieuwing.
</ParamField>
<ParamField path="expiresIn" type="number">
  Vervaltijd van het token in seconden (bijhouden van vernieuwing).
</ParamField>
<ParamField path="obtainmentTimestamp" type="number">
  Tijdstempel waarop het token is verkregen (bijhouden van vernieuwing).
</ParamField>
<ParamField path="allowFrom" type="string[]">
  Toelatingslijst met gebruikers-ID's. Wanneer deze is ingesteld, worden rollen genegeerd.
</ParamField>
<ParamField path="allowedRoles" type='Array<"moderator" | "owner" | "vip" | "subscriber" | "all">'>
  Rolgebaseerd toegangsbeheer.
</ParamField>
<ParamField path="requireMention" type="boolean" default="true">
  Vereis een @vermelding om de bot te activeren.
</ParamField>
<ParamField path="responsePrefix" type="string">
  Overschrijving van het voorvoegsel voor uitgaande antwoorden voor dit account.
</ParamField>

### Provideropties

- `channels.twitch.enabled` - Opstarten van het kanaal in-/uitschakelen
- `channels.twitch.username` / `accessToken` / `clientId` / `channel` - Vereenvoudigde configuratie voor één account (impliciet account `default`; heeft voorrang op `accounts.default`)
- `channels.twitch.accounts.<accountName>` - Configuratie voor meerdere accounts (alle bovenstaande accountvelden)
- `channels.twitch.defaultAccount` - Welke accountnaam de standaard is
- `channels.twitch.markdown.tables` - Weergavemodus voor Markdown-tabellen (`off` | `bullets` | `code` | `block`)

Volledig voorbeeld:

```json5
{
  channels: {
    twitch: {
      enabled: true,
      username: "openclaw",
      accessToken: "oauth:abc123...",
      clientId: "xyz789...",
      channel: "yourchannel",
      clientSecret: "secret123...",
      refreshToken: "refresh456...",
      allowFrom: ["123456789"],
      accounts: {
        second: {
          username: "mybot",
          accessToken: "oauth:def456...",
          clientId: "uvw012...",
          channel: "your_channel",
          enabled: true,
          expiresIn: 14400,
          obtainmentTimestamp: 1706092800000,
          allowedRoles: ["moderator"],
        },
      },
    },
  },
}
```

## Toolacties

De agent kan Twitch-berichten verzenden via de actie `send` van de berichtentool:

```json5
{
  channel: "twitch",
  action: "send",
  to: "#mychannel",
  message: "Hallo Twitch!",
}
```

`to` is optioneel en gebruikt standaard de geconfigureerde `channel` van het account.

## Veiligheid en beheer

- **Behandel tokens als wachtwoorden** - commit tokens nooit naar git.
- **Gebruik automatische tokenvernieuwing** voor bots die langdurig actief zijn.
- **Gebruik toelatingslijsten met gebruikers-ID's** in plaats van gebruikersnamen voor toegangsbeheer.
- **Controleer logboeken** op gebeurtenissen rond tokenvernieuwing en de verbindingsstatus.
- **Beperk het bereik van tokens tot het minimum** - vraag alleen `chat:read` en `chat:write` aan.
- **Als je vastloopt**: start de Gateway opnieuw nadat je hebt gecontroleerd dat geen ander proces eigenaar is van de sessie.

## Limieten

- **500 tekens** per bericht; langere antwoorden worden bij woordgrenzen opgesplitst.
- Markdown wordt vóór verzending verwijderd (Twitch-chat bestaat uit platte tekst; nieuwe regels worden spaties).
- OpenClaw voegt zelf geen snelheidsbeperking toe; de Twurple-chatclient handelt de snelheidslimieten van Twitch af.

## Gerelateerd

- [Kanaalroutering](/nl/channels/channel-routing) — sessieroutering voor berichten
- [Overzicht van kanalen](/nl/channels) — alle ondersteunde kanalen
- [Groepen](/nl/channels/groups) — gedrag van groepschats en filtering op vermeldingen
- [Koppelen](/nl/channels/pairing) — DM-authenticatie en koppelingsproces
- [Beveiliging](/nl/gateway/security) — toegangsmodel en versterking
