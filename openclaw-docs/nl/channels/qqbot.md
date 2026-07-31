---
read_when:
    - Je wilt OpenClaw met QQ verbinden
    - Je moet de inloggegevens voor QQ Bot instellen
    - Je wilt ondersteuning voor groeps- of privéchats met QQ Bot
summary: QQ Bot instellen, configureren en gebruiken
title: QQ-bot
x-i18n:
    generated_at: "2026-07-27T05:34:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b185a2b1182471bbec3688b40fb72b671bdf3a2e8351aa6e2f7918f4f5936825
    source_path: channels/qqbot.md
    workflow: 16
---

QQ Bot maakt verbinding met OpenClaw via de officiële QQ Bot-API (WebSocket-gateway).
C2C-privéchats en `@`-vermeldingen in groepen zijn de primaire chattypen, met rich
media (afbeeldingen, spraak, video, bestanden). Berichten in guild-kanalen worden alleen ondersteund voor
tekst en afbeeldingen via externe URL's; spraak, video, bestandsuploads en lokale/Base64-
afbeeldingen zijn niet beschikbaar in guild-kanalen. Reacties en threads worden
nergens ondersteund.

Status: officiële downloadbare plugin.

## Installeren

```bash
openclaw plugins install @openclaw/qqbot
```

## Instellen

1. Ga naar het [QQ Open Platform](https://q.qq.com/) en scan de QR-code met QQ op je
   telefoon om je te registreren / aan te melden.
2. Klik op **Create Bot** om een nieuwe QQ-bot te maken.
3. Zoek **AppID** en **AppSecret** op de instellingenpagina van de bot en kopieer ze.

<Note>
AppSecret wordt niet als platte tekst opgeslagen. Als je de pagina verlaat zonder het op te slaan, moet je een nieuw geheim genereren.
</Note>

4. Voeg het kanaal toe:

```bash
openclaw channels add --channel qqbot --token "AppID:AppSecret"
```

5. Start de Gateway opnieuw.

## Duurzaamheid van inkomende berichten

Voor beurtgebeurtenissen van de QQ-gateway slaat OpenClaw de onbewerkte gebeurtenis permanent op voordat de opgeslagen hervattingsvolgorde van de gateway wordt bijgewerkt. Wachtende of opnieuw uitvoerbare beurten blijven behouden na een herstart van de Gateway, blijven per gesprek geserialiseerd en gebruiken de gebeurtenis-ID van de provider om dubbele wachtrij-items te onderdrukken zolang de actieve of bewaarde voltooiingsrecord bestaat.

Als duurzame toelating mislukt, beëindigt OpenClaw de huidige gateway-socket zonder de volgorde bij te werken. Het pad voor opnieuw verbinden/hervatten kan de niet-vastgelegde gebeurtenis vervolgens opnieuw opvragen. De aflevering blijft minstens één keer plaatsvinden over de grens tussen wachtrij en agent, zodat een crash tijdens de overdracht een beurt opnieuw kan afspelen.

Interactieve instelling:

```bash
openclaw channels add
```

De wizard biedt ook koppeling via een QR-code als alternatief voor het handmatig
invoeren van AppID/AppSecret: scan de code met de telefoonapp die aan de doel-QQ Bot
is gekoppeld om de koppeling te voltooien. OpenClaw slaat de geretourneerde aanmeldgegevens
permanent op binnen het configuratiebereik van het account.

## Configureren

Minimale configuratie:

```json5
{
  channels: {
    qqbot: {
      enabled: true,
      appId: "YOUR_APP_ID",
      clientSecret: "YOUR_APP_SECRET",
    },
  },
}
```

Omgevingsvariabelen voor het standaardaccount (alleen het account op het hoogste niveau):

- `QQBOT_APP_ID`
- `QQBOT_CLIENT_SECRET`

AppSecret uit een bestand:

```json5
{
  channels: {
    qqbot: {
      enabled: true,
      appId: "YOUR_APP_ID",
      clientSecretFile: "/path/to/qqbot-secret.txt",
    },
  },
}
```

AppSecret via een SecretRef-omgevingsvariabele:

```json5
{
  channels: {
    qqbot: {
      enabled: true,
      appId: "YOUR_APP_ID",
      clientSecret: { source: "env", provider: "default", id: "QQBOT_CLIENT_SECRET" },
    },
  },
}
```

Opmerkingen:

- `openclaw channels add --channel qqbot --token-file ...` stelt alleen het AppSecret
  in; `appId` moet al zijn ingesteld in de configuratie of `QQBOT_APP_ID`.
- `clientSecret` accepteert een tekenreeks met platte tekst, een bestandspad (`clientSecretFile`),
  of een gestructureerd SecretRef-object.
- Verouderde markerreeksen `secretref:...` / `secretref-env:...` worden geweigerd voor
  `clientSecret`; gebruik in plaats daarvan een gestructureerd SecretRef-object.

### Streaming

```json5
{
  channels: {
    qqbot: {
      streaming: {
        mode: "partial", // blokstreaming: "partial" (standaard) of "off"
        nativeTransport: true, // gebruik QQ's officiële C2C stream_messages-API voor privéberichten
      },
    },
  },
}
```

- `streaming.mode: "off"` schakelt blokstreaming voor het account uit.
- `streaming.nativeTransport: true` streamt C2C-antwoorden (privéberichten) via QQ's
  officiële `stream_messages`-API; doelen in groepen/kanalen worden niet beïnvloed.
- Verouderde scalaire waarden van `streaming: true|false` en de sleutel `streaming.c2cStreamApi`
  worden via `openclaw doctor --fix` naar deze structuur gemigreerd.
- `/bot-streaming on|off` schakelt dezelfde configuratie vanuit een privébericht om.

### Toegangsbeleid

- `allowFrom` / `groupAllowFrom` bepalen wie in een C2C- /
  groepscontext met de bot kan chatten. `dmPolicy` / `groupPolicy` (`open` | `allowlist` | `disabled`)
  bepalen de handhavingsmodus. `dmPolicy` wordt standaard `allowlist` zodra
  `allowFrom` een concrete vermelding (zonder jokerteken) bevat, anders `open`.
  `groupPolicy` wordt standaard `allowlist` zodra `groupAllowFrom` of
  `allowFrom` een concrete vermelding bevat, anders `open`.
- Slash-opdrachten met "Auth: allowlist" vereisen een expliciete vermelding zonder jokerteken in
  `allowFrom` (of `groupAllowFrom` voor aanroepen vanuit een groep), ongeacht
  `dmPolicy` / `groupPolicy` — zie [Slash-opdrachten](#slash-commands).

### Instelling voor meerdere accounts

Voer meerdere QQ-bots uit binnen één OpenClaw-instantie:

```json5
{
  channels: {
    qqbot: {
      enabled: true,
      appId: "111111111",
      clientSecret: "secret-of-bot-1",
      accounts: {
        bot2: {
          enabled: true,
          appId: "222222222",
          clientSecret: "secret-of-bot-2",
        },
      },
    },
  },
}
```

Elk account beschikt over een geïsoleerde WebSocket-verbinding, API-client en token-
cache, geïndexeerd op `appId`. Logregels worden gemarkeerd met de ID van het bijbehorende account, zodat
diagnostische gegevens gescheiden blijven wanneer je meerdere bots onder één Gateway uitvoert.

Voeg via de CLI een tweede bot toe:

```bash
openclaw channels add --channel qqbot --account bot2 --token "222222222:secret-of-bot-2"
```

### Groepschats

Groepsondersteuning gebruikt QQ-groeps-OpenID's, geen weergavenamen. Voeg de bot toe aan een
groep en vermeld deze vervolgens, of configureer de groep om zonder vermelding te werken.

```json5
{
  channels: {
    qqbot: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["member_openid"],
      groups: {
        "*": {
          requireMention: true,
          commandLevel: "all",
          historyLimit: 50,
          tools: { deny: ["exec", "read", "write"] },
        },
        GROUP_OPENID: {
          name: "Release room",
          requireMention: false,
          ignoreOtherMentions: true,
          commandLevel: "safety",
          historyLimit: 20,
          prompt: "Keep replies short and operational.",
        },
      },
    },
  },
}
```

`groups["*"]` stelt de standaardwaarden voor elke groep in; een concrete vermelding voor `groups.GROUP_OPENID`
overschrijft die standaardwaarden voor één groep. Groepsinstellingen:

| Veld                  | Standaard        | Beschrijving                                                                                       |
| --------------------- | ---------------- | -------------------------------------------------------------------------------------------------- |
| `requireMention`      | `true`           | Vereis een `@`-vermelding voordat de bot antwoordt.                                                |
| `commandLevel`        | `all`            | Welke ingebouwde slash-opdrachten in de groep kunnen worden uitgevoerd (zie hieronder).            |
| `ignoreOtherMentions` | `false`          | Negeer berichten waarin iemand anders wordt vermeld, maar de bot niet.                             |
| `historyLimit`        | `50`             | Recente berichten zonder vermelding die als context voor de volgende vermelde beurt worden bewaard. `0` schakelt de geschiedenis uit. |
| `tools`               | —                | Sta tools toe of weiger ze voor de hele groep.                                                     |
| `toolsBySender`       | —                | Tooloverschrijvingen per afzender; zie [Groepen](/nl/channels/groups#groupchannel-tool-restrictions-optional). |
| `name`                | OpenID-voorvoegsel | Gebruiksvriendelijk label dat in logs en groepscontext wordt gebruikt.                             |
| `prompt`              | ingebouwde standaard | Gedragsprompt per groep die aan de agentcontext wordt toegevoegd.                                  |

`commandLevel` accepteert:

| Niveau  | Gedrag                                                                                                                                        |
| -------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `all`    | Bestaande ingebouwde opdrachten blijven beschikbaar. Sommige blijven verborgen in menu's, maar geautoriseerde gebruikers kunnen ze nog steeds in de groep uitvoeren. |
| `safety` | `/help`, `/btw`, `/stop` blijven zichtbaar in de groep; gevoelige opdrachten (`/config`, `/tools`, `/bash`, enz.) moeten in een privéchat worden uitgevoerd. |
| `strict` | Alleen besturingselementen voor groepssessies die voor strikt gebruik nodig zijn, zijn toegestaan. `/stop` blijft werken, zodat een geautoriseerde afzender een actieve uitvoering kan onderbreken. |

Oude QQBot-vermeldingen voor `toolPolicy` zijn buiten gebruik gesteld. Voer `openclaw doctor --fix` uit om ze naar `tools` te migreren.

De activeringsmodi zijn `mention` en `always`. `requireMention: true` wordt toegewezen aan
`mention`; `requireMention: false` wordt toegewezen aan `always`. Een activeringsoverride
op sessieniveau heeft, indien aanwezig, voorrang op de configuratie.

De wachtrij voor inkomende berichten is per peer. Groepspeers krijgen een hogere wachtrijlimiet (50 tegenover 20
voor directe peers), verwijderen bij een volle wachtrij door de bot geschreven berichten vóór menselijke berichten
en voegen reeksen normale groepsberichten samen tot één beurt met bronvermelding. Slash-
opdrachten worden één voor één uitgevoerd, onafhankelijk van een eventuele samenvoegbatch.

### Spraak (STT / TTS)

STT en TTS ondersteunen configuratie op twee niveaus met terugval op basis van prioriteit:

| Instelling | Plugin-specifiek                                         | Terugval van het framework                        |
| ---------- | -------------------------------------------------------- | ------------------------------------------------- |
| STT        | `channels.qqbot.stt`                                     | eerste audiocompatibele vermelding in `tools.media.models[]` |
| TTS        | `channels.qqbot.tts`, `channels.qqbot.accounts.<id>.tts` | `tts`                                            |

```json5
{
  channels: {
    qqbot: {
      stt: {
        provider: "your-provider",
        model: "your-stt-model",
      },
      tts: {
        provider: "your-provider",
        model: "your-tts-model",
        voice: "your-voice",
      },
      accounts: {
        "qq-main": {
          tts: {
            providers: {
              openai: { voice: "shimmer" },
            },
          },
        },
      },
    },
  },
}
```

Stel `enabled: false` op een van beide in om deze uit te schakelen. TTS-overschrijvingen op accountniveau gebruiken
dezelfde structuur als `tts` en worden recursief samengevoegd met de TTS-configuratie op kanaal-/globaal niveau.

STT-aanvragen verlopen standaard na 60 seconden. Plugin-specifieke STT gebruikt de
geselecteerde overschrijving voor `models.providers.<id>.timeoutSeconds`. Audio-STT van het framework
gebruikt eerst `timeoutSeconds` van de geselecteerde audiocompatibele vermelding in `tools.media.models[]` en daarna de geselecteerde provideroverride.

Inkomende QQ-spraakbijlagen worden aan agents aangeboden als metadata voor audiomedia,
terwijl onbewerkte spraakbestanden buiten de algemene `MediaPaths` blijven. `[[audio_as_voice]]`
in een antwoord met platte tekst synthetiseert TTS en verzendt een native QQ-spraakbericht wanneer
TTS is geconfigureerd.

Het upload-/transcodeergedrag voor uitgaande audio kan ook worden aangepast met
`channels.qqbot.audioFormatPolicy`:

- `sttDirectFormats`
- `uploadDirectFormats`
- `transcodeEnabled`

## Doelindelingen

| Indeling                   | Beschrijving       |
| -------------------------- | ------------------ |
| `qqbot:c2c:OPENID`         | Privéchat (C2C)    |
| `qqbot:group:GROUP_OPENID` | Groepschat         |
| `qqbot:channel:CHANNEL_ID` | Guild-kanaal       |

<Note>
Elke bot heeft een eigen set OpenID's van gebruikers. Een OpenID dat door Bot A is ontvangen, kan **niet** worden gebruikt om berichten via Bot B te verzenden.
</Note>

## Slash-opdrachten

Ingebouwde opdrachten die vóór de AI-wachtrij worden onderschept:

| Opdracht              | Auth      | Bereik        | Beschrijving                                                                    |
| -------------------- | --------- | ------------ | ------------------------------------------------------------------------------ |
| `/bot-ping`          | —         | elk          | Latentietest                                                                   |
| `/bot-help`          | —         | elk          | Alle opdrachten weergeven                                                              |
| `/bot-me`            | —         | alleen privé | De QQ-gebruikers-ID (openid) van de afzender weergeven voor het instellen van `allowFrom` / `groupAllowFrom` |
| `/bot-version`       | —         | alleen privé | De versie van het OpenClaw-framework en de pluginversie weergeven                         |
| `/bot-upgrade`       | —         | alleen privé | De link naar de QQBot-upgradehandleiding weergeven                                              |
| `/bot-approve`       | toelatingslijst | alleen privé | De configuratie voor goedkeuring van opdrachtuitvoering beheren (aan / uit / altijd / resetten / status)  |
| `/bot-logs`          | toelatingslijst | alleen privé | Recente Gateway-logboeken als bestand exporteren                                           |
| `/bot-clear-storage` | toelatingslijst | alleen privé | Gecachte downloads in de QQBot-mediamap verwijderen                        |
| `/bot-streaming`     | toelatingslijst | alleen privé | Streamingantwoorden voor C2C in- of uitschakelen                                                   |
| `/bot-group-allways` | toelatingslijst | alleen privé | De standaardactiveringsmodus voor groepen omschakelen (vermelding vereist versus altijd actief)      |

Voeg `?` toe aan een opdracht voor gebruikshulp (bijvoorbeeld `/bot-upgrade ?`).

Opdrachten met "Auth: toelatingslijst" vereisen bovendien dat de openid van de afzender in een
expliciete `allowFrom`-lijst zonder jokerteken staat (`groupAllowFrom` heeft voorrang voor
opdrachten die vanuit groepen worden gegeven, met terugval op `allowFrom`). Een jokerteken
`allowFrom: ["*"]` staat chatten toe, maar niet deze opdrachten. Wanneer een van deze opdrachten
buiten een privéchat of zonder autorisatie wordt uitgevoerd, wordt een aanwijzing teruggestuurd in plaats van
het bericht stilzwijgend te negeren.

`/bot-me`, `/bot-version` en `/bot-upgrade` zijn alleen beschikbaar in privéchats, maar
vereisen de toelatingslijst niet — elke C2C-afzender kan ze uitvoeren.

Wanneer uitvoeringsgoedkeuringen voor QQ Bot de standaardterugval naar dezelfde chat gebruiken, volgen klikken op systeemeigen
goedkeuringsknoppen dezelfde expliciete opdrachtenlijst zonder jokerteken. Configureer
`channels.qqbot.execApprovals.approvers` om alleen toegang tot goedkeuringen toe te staan zonder bredere toegang tot opdrachten.
Systeemeigen uitvoeringsgoedkeuringen zijn standaard
ingeschakeld.

## Media en opslag

- Inkomende, uitgaande en via de Gateway-brug verzonden media delen één hoofdmap voor payloads onder
  `~/.openclaw/media/qqbot` (waarbij `OPENCLAW_HOME` wordt gerespecteerd indien ingesteld), zodat uploads,
  downloads en transcoderingscaches binnen één beveiligde map blijven.
- De levering van rijke media aan C2C- en groepsdoelen verloopt via één `sendMedia`-pad.
  Lokale bestanden en buffers in het geheugen van 5&nbsp;MiB of meer gebruiken de
  endpoints voor uploads in delen van QQ; kleinere payloads en bronnen via externe URL's/Base64 gebruiken
  de API voor eenmalige uploads.
- Als een hot-upgrade de Gateway onderbreekt voordat deze klaar is met het schrijven van
  `openclaw.json`, herstelt de plugin bij de volgende start de laatst bekende `appId` / `clientSecret`
  voor dat account vanuit een interne momentopname (waarbij een opzettelijke configuratiewijziging nooit
  wordt overschreven), zodat het opnieuw scannen van de QR-code niet
  nodig is.

## Problemen oplossen

- **Gateway start niet / geen inkomende berichten:** controleer of `appId` en
  `clientSecret` correct zijn en of de bot is ingeschakeld op het QQ Open Platform.
  Een ontbrekend toegangsmiddel wordt weergegeven als "QQBot niet geconfigureerd (appId of
  clientSecret ontbreekt)".
- **Instellen met `--token-file` wordt nog steeds als niet geconfigureerd weergegeven:** `--token-file`
  stelt alleen het AppSecret in. `appId` moet nog steeds worden ingesteld in de configuratie of `QQBOT_APP_ID`.
- **Piekgewijze groepsantwoorden botsen:** wanneer de wachtrij van een peer volloopt, verwijdert de inkomende wachtrij
  berichten die door bots zijn opgesteld vóór menselijke berichten, en voegt deze
  pieken van normale groepsberichten (geen opdrachten) samen tot één beurt met bronvermelding, zodat
  een stortvloed aan botberichten menselijke berichten niet zou moeten verdringen.
- **Proactieve berichten komen niet aan:** QQ kan door de bot geïnitieerde berichten blokkeren als
  de gebruiker recent geen interactie heeft gehad.
- **Spraak wordt niet getranscribeerd:** zorg dat STT is geconfigureerd en dat de provider
  bereikbaar is.

## Gerelateerd

- [Koppelen](/nl/channels/pairing)
- [Groepen](/nl/channels/groups)
- [Problemen met kanalen oplossen](/nl/channels/troubleshooting)
