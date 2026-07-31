---
read_when:
    - Je wilt OpenClaw verbinden met IRC-kanalen of privéberichten
    - Je configureert IRC-toelatingslijsten, groepsbeleid of vermeldingsbeperking
summary: Installatie, toegangsbeheer en probleemoplossing voor de IRC-plugin
title: IRC
x-i18n:
    generated_at: "2026-07-27T05:42:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 85c3da80b45d6611872ddbd10b3be4a5742b46e355e8bb554353a478f2a1702f
    source_path: channels/irc.md
    workflow: 16
---

Gebruik IRC wanneer je OpenClaw wilt inzetten in klassieke kanalen (`#room`) en privéberichten.
Installeer de officiële IRC-plugin en configureer deze vervolgens onder `channels.irc`.

## Snel aan de slag

1. Installeer de plugin:

```bash
openclaw plugins install @openclaw/irc
```

2. Stel in `~/.openclaw/openclaw.json` ten minste de host, nick en kanalen in waaraan moet worden deelgenomen:

```json5
{
  channels: {
    irc: {
      enabled: true,
      host: "irc.example.com",
      port: 6697,
      tls: true,
      nick: "openclaw-bot",
      channels: ["#openclaw"],
    },
  },
}
```

3. Start of herstart de Gateway:

```bash
openclaw gateway run
```

Geef voor botcoördinatie de voorkeur aan een privé-IRC-server. Als je bewust een openbaar IRC-netwerk gebruikt, zijn Libera.Chat, OFTC en Snoonet gangbare keuzes. Vermijd voorspelbare openbare kanalen voor backchannelverkeer van bots of zwermen.

## Duurzaamheid van inkomende berichten

OpenClaw schrijft elk geaccepteerd IRC-`PRIVMSG` naar zijn duurzame wachtrij voor inkomende berichten voordat de normale beleidscontroles en agentdispatch plaatsvinden. Openstaande of opnieuw te proberen berichten blijven behouden na een herstart van de Gateway en blijven per kanaal of gesprekspartner voor privéberichten geserialiseerd.

IRC biedt geen herbruikbare leverings-ID en verzendt geen berichten opnieuw die een niet-verbonden client heeft gemist. Daarom wijst OpenClaw een lokale ID toe die alleen binnen de huidige TCP-verbinding stabiel is. De wachtrij beschermt het lokale venster tussen acceptatie en dispatch; een bericht dat OpenClaw nooit heeft bereikt, kan hiermee niet worden hersteld, en een opnieuw verzonden serverbericht kan niet over verbindingen heen worden gededupliceerd.

## Verbindingsinstellingen

| Sleutel                       | Standaard                     | Opmerkingen                                                 |
| ----------------------------- | ----------------------------- | ----------------------------------------------------------- |
| `host`                        | geen (verplicht)              | Hostnaam van de IRC-server                                  |
| `port`                        | `6697` met TLS, `6667` onbeveiligd | 1-65535                                                     |
| `tls`                         | `true`                        | Stel `false` alleen in voor bewust gebruik van platte tekst |
| `nick`                        | geen (verplicht)              | Nick van de bot                                             |
| `username`                    | nick, anders `openclaw`       | IRC-gebruikersnaam                                          |
| `realname`                    | `OpenClaw`                    | Realname-/GECOS-veld                                        |
| `password` / `passwordFile`   | geen                          | Serverwachtwoord; het bestand moet een regulier bestand zijn |
| `channels`                    | geen                          | Kanalen om aan deel te nemen (`["#openclaw"]`)             |
| `accounts` / `defaultAccount` | geen                          | Configuratie met meerdere accounts; omgevingsvariabelen vullen alleen het standaardaccount in |

## Standaardbeveiliging

- IRC gebruikt onbewerkte TCP-/TLS-sockets buiten de door de OpenClaw-operator beheerde routering via de forward proxy. Stel bij implementaties waarin al het uitgaande verkeer via die forward proxy moet lopen `channels.irc.enabled=false` in, tenzij direct uitgaand IRC-verkeer uitdrukkelijk is goedgekeurd.
- `channels.irc.dmPolicy` is standaard `"pairing"`: onbekende afzenders van privéberichten krijgen een koppelcode die je goedkeurt met `openclaw pairing approve irc <code>`.
- `channels.irc.groupPolicy` is standaard `"allowlist"`.
- Stel bij `groupPolicy="allowlist"` `channels.irc.groups` in om toegestane kanalen te definiëren.
- Gebruik TLS (`channels.irc.tls=true`), tenzij je bewust transport in platte tekst accepteert.

## Toegangsbeheer

Er zijn twee afzonderlijke 'poorten' voor IRC-kanalen:

1. **Kanaaltoegang** (`groupPolicy` + `groups`): of de bot überhaupt berichten uit een kanaal accepteert.
2. **Afzendertoegang** (`groupAllowFrom` / `groups["#channel"].allowFrom` per kanaal): wie de bot binnen dat kanaal mag activeren.

Configuratiesleutels:

- Toestaanlijst voor privéberichten (toegang voor afzenders van privéberichten): `channels.irc.allowFrom`
- Toestaanlijst voor groepsafzenders (toegang voor kanaalafzenders): `channels.irc.groupAllowFrom`
- Besturing per kanaal (regels voor kanaal, afzender en vermeldingen): `channels.irc.groups["#channel"]` met `requireMention`, `allowFrom`, `enabled`, `tools`, `toolsBySender`, `skills` en `systemPrompt`
- `channels.irc.groupPolicy="open"` staat niet-geconfigureerde kanalen toe (**standaard geldt nog steeds een vermeldingsvereiste**)

Vermeldingen in de toestaanlijst moeten stabiele afzenderidentiteiten gebruiken (`nick!user@host`).
Overeenkomst op basis van alleen de nick is veranderlijk en wordt alleen ingeschakeld wanneer `channels.irc.dangerouslyAllowNameMatching: true`.

### Veelvoorkomende valkuil: `allowFrom` is voor privéberichten, niet voor kanalen

Als je logboekregels ziet zoals:

- `irc: drop group sender alice!ident@host (policy=allowlist)`

...betekent dit dat de afzender niet was toegestaan voor **groeps-/kanaalberichten**. Los dit op door:

- `channels.irc.groupAllowFrom` in te stellen (globaal voor alle kanalen), of
- toestaanlijsten voor afzenders per kanaal in te stellen: `channels.irc.groups["#channel"].allowFrom`

Voorbeeld (iedereen in `#openclaw` toestaan met de bot te praten):

```json5
{
  channels: {
    irc: {
      groupPolicy: "allowlist",
      groups: {
        "#openclaw": { allowFrom: ["*"] },
      },
    },
  },
}
```

## Antwoorden activeren (vermeldingen)

Zelfs als een kanaal is toegestaan (via `groupPolicy` + `groups`) en de afzender is toegestaan, past OpenClaw in groepscontexten standaard een **vermeldingsvereiste** toe. De bot geldt als vermeld wanneer het bericht de verbonden botnick bevat of overeenkomt met je geconfigureerde vermeldingspatronen.

Dit betekent dat je logboekregels zoals `drop channel … (missing-mention)` kunt zien, tenzij het bericht een vermeldingspatroon bevat dat met de bot overeenkomt.

Schakel de vermeldingsvereiste voor dat kanaal uit om de bot **zonder vereiste vermelding** in een IRC-kanaal te laten antwoorden:

```json5
{
  channels: {
    irc: {
      groupPolicy: "allowlist",
      groups: {
        "#openclaw": {
          requireMention: false,
          allowFrom: ["*"],
        },
      },
    },
  },
}
```

Of om **alle** IRC-kanalen toe te staan (zonder toestaanlijst per kanaal) en toch zonder vermeldingen te antwoorden:

```json5
{
  channels: {
    irc: {
      groupPolicy: "open",
      groups: {
        "*": { requireMention: false, allowFrom: ["*"] },
      },
    },
  },
}
```

## Beveiligingsopmerking (aanbevolen voor openbare kanalen)

Als je `allowFrom: ["*"]` in een openbaar kanaal toestaat, kan iedereen de bot een prompt geven.
Beperk de tools voor dat kanaal om het risico te verkleinen.

### Dezelfde tools voor iedereen in het kanaal

```json5
{
  channels: {
    irc: {
      groups: {
        "#openclaw": {
          allowFrom: ["*"],
          tools: {
            deny: ["group:runtime", "group:fs", "gateway", "nodes", "cron", "browser"],
          },
        },
      },
    },
  },
}
```

### Verschillende tools per afzender (de eigenaar krijgt meer bevoegdheden)

Gebruik `toolsBySender` om een strenger beleid toe te passen op `"*"` en een minder streng beleid op je nick:

```json5
{
  channels: {
    irc: {
      groups: {
        "#openclaw": {
          allowFrom: ["*"],
          toolsBySender: {
            "*": {
              deny: ["group:runtime", "group:fs", "gateway", "nodes", "cron", "browser"],
            },
            "id:alice": {
              deny: ["gateway", "nodes", "cron"],
            },
          },
        },
      },
    },
  },
}
```

Opmerkingen:

- Sleutels voor `toolsBySender` moeten expliciete voorvoegsels gebruiken (`channel:`, `id:`, `e164:`, `username:`, `name:`). Gebruik voor IRC `id:` met de identiteitswaarde van de afzender: `id:alice` of `id:alice!~alice@203.0.113.7` voor een sterkere overeenkomst.
- Verouderde sleutels zonder voorvoegsel worden nog steeds geaccepteerd, uitsluitend als `id:` gematcht en geven een afschrijvingswaarschuwing.
- Het eerste overeenkomende afzenderbeleid heeft voorrang; `"*"` is de jokertekenfallback.

Zie voor meer informatie over groepstoegang versus vermeldingsvereisten (en hun onderlinge wisselwerking): [/channels/groups](/nl/channels/groups).

## NickServ

Identificeren bij NickServ na het verbinden:

```json5
{
  channels: {
    irc: {
      nickserv: {
        enabled: true,
        service: "NickServ",
        password: "your-nickserv-password",
      },
    },
  },
}
```

Identificatie bij NickServ wordt standaard uitgevoerd wanneer een wachtwoord is ingesteld (`enabled` hoeft alleen `false` te zijn om dit uit te schakelen). `service` is standaard `NickServ`; `passwordFile` is een alternatief voor inline `password`.

Optionele eenmalige registratie bij het verbinden (`register: true` vereist `registerEmail`):

```json5
{
  channels: {
    irc: {
      nickserv: {
        register: true,
        registerEmail: "bot@example.com",
      },
    },
  },
}
```

Schakel `register` uit nadat de nick is geregistreerd om herhaalde REGISTER-pogingen te voorkomen.

## Omgevingsvariabelen

Het standaardaccount ondersteunt:

- `IRC_HOST`
- `IRC_PORT`
- `IRC_TLS`
- `IRC_NICK`
- `IRC_USERNAME`
- `IRC_REALNAME`
- `IRC_PASSWORD`
- `IRC_CHANNELS` (door komma's gescheiden)
- `IRC_NICKSERV_PASSWORD`
- `IRC_NICKSERV_REGISTER_EMAIL`

`IRC_HOST` kan niet worden ingesteld vanuit een `.env` van een werkruimte; zie [`.env`-bestanden van werkruimten](/nl/gateway/security).

## Problemen oplossen

- Als de bot verbinding maakt maar nooit in kanalen antwoordt, controleer dan `channels.irc.groups` **en** of berichten door de vermeldingsvereiste worden geweigerd (`missing-mention`). Stel `requireMention:false` voor het kanaal in als je wilt dat de bot zonder pings antwoordt.
- Als aanmelden mislukt, controleer dan de beschikbaarheid van de nick en het serverwachtwoord.
- Als TLS op een aangepast netwerk mislukt, controleer dan de host, poort en certificaatconfiguratie.

## Gerelateerd

- [Overzicht van kanalen](/nl/channels) — alle ondersteunde kanalen
- [Koppelen](/nl/channels/pairing) — authenticatie van privéberichten en koppelingsflow
- [Groepen](/nl/channels/groups) — gedrag van groepschats en vermeldingsvereisten
- [Kanaalroutering](/nl/channels/channel-routing) — sessieroutering voor berichten
- [Beveiliging](/nl/gateway/security) — toegangsmodel en versterking
