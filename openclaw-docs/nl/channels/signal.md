---
read_when:
    - Signal-ondersteuning instellen
    - Problemen met het verzenden/ontvangen via Signal oplossen
summary: Signal-ondersteuning via signal-cli (native daemon of bbernhard-container), configuratiepaden en nummermodel
title: Signal
x-i18n:
    generated_at: "2026-07-27T04:48:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 744f817e425d378e9f3e160df534019a6fc865227eb3fc68959a12ad46c0b714
    source_path: channels/signal.md
    workflow: 16
---

Signal is een downloadbare kanaalplugin (`@openclaw/signal`). De Gateway communiceert via HTTP met `signal-cli`: ofwel de systeemeigen daemon (JSON-RPC + SSE), ofwel de [bbernhard/signal-cli-rest-api](https://github.com/bbernhard/signal-cli-rest-api)-container (REST + WebSocket). OpenClaw bevat libsignal niet.

## Het nummermodel (lees dit eerst)

- De Gateway maakt verbinding met een **Signal-apparaat**: het `signal-cli`-account.
- Als je de bot op **je persoonlijke Signal-account** uitvoert, negeert deze je eigen berichten (lusbeveiliging).
- Gebruik voor "ik stuur de bot een bericht en hij antwoordt" een **afzonderlijk botnummer**.

## Installeren

```bash
openclaw plugins install @openclaw/signal
```

Bij kale pluginspecificaties wordt eerst ClawHub geprobeerd, met npm als terugvaloptie. Dwing een bron af met `openclaw plugins install clawhub:@openclaw/signal` of `npm:@openclaw/signal`. `plugins install` registreert en activeert de plugin; er is geen afzonderlijke `enable`-stap nodig. Zie [Plugins](/nl/tools/plugin) voor algemene installatieregels.

## Snelle configuratie

<Steps>
  <Step title="Kies een nummer">
    Gebruik een **afzonderlijk Signal-nummer** voor de bot (aanbevolen).
  </Step>
  <Step title="Installeer de plugin">
    ```bash
    openclaw plugins install @openclaw/signal
    ```
  </Step>
  <Step title="Voer de begeleide configuratie uit">
    ```bash
    openclaw channels add
    ```
    De wizard detecteert of `signal-cli` zich op `PATH` bevindt en biedt, als het ontbreekt, aan om het te installeren: de officiële systeemeigen GraalVM-build wordt gedownload op Linux x86-64, of het wordt via Homebrew geïnstalleerd op macOS en andere architecturen. Vervolgens wordt gevraagd naar het botnummer en het pad van `signal-cli`.

    Voor niet-interactieve configuratie accepteert `openclaw channels add --channel signal` ook `--signal-number <e164>` voor het telefoonnummer van de bot, plus `--http-host <host>` en `--http-port <port>` voor het eindpunt van de Signal-daemon (standaard `127.0.0.1:8080`).

  </Step>
  <Step title="Koppel of registreer het account">
    - **Koppelen via QR (snelst):** `signal-cli link -n "OpenClaw"`, scan vervolgens met Signal. Zie [Pad A](#setup-path-a-link-existing-signal-account-qr).
    - **Registratie via sms:** afzonderlijk nummer met captcha en sms-verificatie. Zie [Pad B](#setup-path-b-register-dedicated-bot-number-sms-linux).

  </Step>
  <Step title="Verifieer en koppel">
    ```bash
    openclaw gateway call channels.status --params '{"probe":true}'
    ```
    Stuur een eerste privébericht en keur de koppeling goed: `openclaw pairing approve signal <CODE>`.
  </Step>
</Steps>

Minimale configuratie:

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15551234567",
      transport: {
        kind: "managed-native",
        cliPath: "signal-cli",
      },
      dmPolicy: "pairing",
      allowFrom: ["+15557654321"],
    },
  },
}
```

| Veld       | Beschrijving                                       |
| ----------- | ------------------------------------------------- |
| `account`   | Telefoonnummer van de bot in E.164-indeling (`+15551234567`) |
| `transport` | Signal-verbinding en procesmodus die bij het account horen  |
| `dmPolicy`  | Toegangsbeleid voor privéberichten (`pairing` aanbevolen)          |
| `allowFrom` | Telefoonnummers of `uuid:<id>`-waarden die privéberichten mogen sturen |

Ondersteuning voor meerdere accounts: gebruik `channels.signal.accounts` met een configuratie per account en optioneel `name`. Elk benoemd account beheert zijn eigen `transport`; het neemt het transport op het hoogste niveau niet over. Het transport op het hoogste niveau behoort alleen tot het impliciete `default`-account. Zie [Kanalen met meerdere accounts](/nl/gateway/config-channels#multi-account-all-channels) voor het gedeelde patroon.

## Wat het is

- Deterministische routering: antwoorden gaan altijd terug naar Signal.
- Privéberichten delen de hoofdsessie van de agent; groepen zijn geïsoleerd (`agent:<agentId>:signal:group:<groupId>`).
- Standaard kan Signal configuratie-updates schrijven die door `/config set|unset` worden geactiveerd (vereist `commands.config: true`). Schakel dit uit met `channels.signal.configWrites: false`.

## Configuratiepad A: bestaand Signal-account koppelen (QR)

1. Installeer `signal-cli` (JVM of systeemeigen build), of laat `openclaw channels add` het voor je installeren.
2. Koppel een botaccount: `signal-cli link -n "OpenClaw"`, scan vervolgens de QR-code in Signal.
3. Configureer Signal en start de Gateway.

## Configuratiepad B: afzonderlijk botnummer registreren (sms, Linux)

Gebruik dit voor een afzonderlijk botnummer in plaats van een bestaand Signal-appaccount te koppelen. De onderstaande procedure is getest op Ubuntu 24.

1. Neem een nummer dat sms kan ontvangen (of spraakverificatie voor vaste lijnen). Een afzonderlijk botnummer voorkomt conflicten tussen accounts en sessies.
2. Installeer `signal-cli` op de Gateway-host:

```bash
VERSION=$(curl -Ls -o /dev/null -w %{url_effective} https://github.com/AsamK/signal-cli/releases/latest | sed -e 's/^.*\/v//')
curl -L -O "https://github.com/AsamK/signal-cli/releases/download/v${VERSION}/signal-cli-${VERSION}-Linux-native.tar.gz"
sudo tar xf "signal-cli-${VERSION}-Linux-native.tar.gz" -C /opt
sudo ln -sf /opt/signal-cli /usr/local/bin/
signal-cli --version
```

Als je de JVM-build (`signal-cli-${VERSION}.tar.gz`) gebruikt, installeer dan eerst een JRE. Houd `signal-cli` bijgewerkt; upstream wordt vermeld dat oude releases niet meer kunnen werken wanneer de Signal-server-API's veranderen.

3. Registreer en verifieer het nummer:

```bash
signal-cli -a +<BOT_PHONE_NUMBER> register
```

Als een captcha vereist is (browsertoegang is nodig om deze stap te voltooien):

1. Open `https://signalcaptchas.org/registration/generate.html`.
2. Voltooi de captcha en kopieer het doel van de `signalcaptcha://...`-link uit "Open Signal".
3. Voer de opdracht indien mogelijk uit vanaf hetzelfde externe IP-adres als de browsersessie (captcha-tokens verlopen snel).
4. Registreer en verifieer onmiddellijk:

```bash
signal-cli -a +<BOT_PHONE_NUMBER> register --captcha '<SIGNALCAPTCHA_URL>'
signal-cli -a +<BOT_PHONE_NUMBER> verify <VERIFICATION_CODE>
```

4. Configureer OpenClaw, start de Gateway opnieuw en verifieer het kanaal:

```bash
# Als je de Gateway uitvoert als een systemd-gebruikersservice:
systemctl --user restart openclaw-gateway.service

# Verifieer vervolgens:
openclaw doctor
openclaw channels status --probe
```

5. Koppel de afzender van je privéberichten:
   - Stuur een willekeurig bericht naar het botnummer.
   - Keur het op de server goed: `openclaw pairing approve signal <PAIRING_CODE>`.
   - Sla het botnummer als contactpersoon op je telefoon op om "Unknown contact" te voorkomen.

<Warning>
Als je een telefoonnummeraccount registreert met `signal-cli`, kan de hoofdsessie van de Signal-app voor dat nummer worden afgemeld. Gebruik bij voorkeur een afzonderlijk botnummer of gebruik de QR-koppelmodus om de bestaande configuratie van je telefoonapp te behouden.
</Warning>

Upstream-referenties:

- `signal-cli` README: `https://github.com/AsamK/signal-cli`
- Captchaprocedure: `https://github.com/AsamK/signal-cli/wiki/Registration-with-captcha`
- Koppelprocedure: `https://github.com/AsamK/signal-cli/wiki/Linking-other-devices-(Provisioning)`

## Modus met externe systeemeigen daemon

Als je `signal-cli` zelf wilt beheren (trage koude starts van de JVM, initialisatie van containers, gedeelde CPU's), voer je de daemon afzonderlijk uit en laat je OpenClaw ernaar verwijzen:

Selecteer voor niet-interactieve configuratie indien nodig expliciet het type eindpunt:

```bash
openclaw channels add --channel signal --signal-number +15551234567 \
  --http-url http://127.0.0.1:8080 --signal-transport external-native
```

```json5
{
  channels: {
    signal: {
      transport: {
        kind: "external-native",
        url: "http://127.0.0.1:8080",
      },
    },
  },
}
```

Hiermee worden automatisch starten en de opstartwachttijd van OpenClaw overgeslagen. Stel `channels.signal.transport.startupTimeoutMs` in voor een beheerde daemon die langzaam opstart.

## Containermodus (bbernhard/signal-cli-rest-api)

In plaats van `signal-cli` systeemeigen uit te voeren, kun je de Docker-container [bbernhard/signal-cli-rest-api](https://github.com/bbernhard/signal-cli-rest-api) gebruiken, die `signal-cli` achter een REST- en WebSocket-interface plaatst.

```bash
openclaw channels add --channel signal --signal-number +15551234567 \
  --http-url http://signal-cli:8080 --signal-transport container
```

Vereisten:

- De container **moet** met `MODE=json-rpc` worden uitgevoerd om berichten in realtime te ontvangen.
- Registreer of koppel je Signal-account in de container voordat je OpenClaw verbindt.

Voorbeeldservice voor `docker-compose.yml`:

```yaml
signal-cli:
  image: bbernhard/signal-cli-rest-api:latest
  environment:
    MODE: json-rpc
  ports:
    - "8080:8080"
  volumes:
    - signal-cli-data:/home/.local/share/signal-cli
```

OpenClaw-configuratie:

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15551234567",
      transport: {
        kind: "container",
        url: "http://signal-cli:8080",
      },
    },
  },
}
```

`transport.kind` bepaalt welk protocol en welke proceslevenscyclus OpenClaw gebruikt:

| Waarde               | Gedrag                                                                                                                                                     |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `"managed-native"`  | Start de systeemeigen signal-cli en gebruik JSON-RPC op `/api/v1/rpc` plus SSE op `/api/v1/events`; `url` kan een verbindingseindpunt selecteren dat verschilt van de binding van de daemon |
| `"external-native"` | Maak verbinding met een al actieve systeemeigen signal-cli-daemon                                                                                                       |
| `"container"`       | Maak verbinding met bbernhard REST op `/v2/send` en WebSocket op `/v1/receive/{account}`                                                                             |

De configuratie en `openclaw doctor --fix` kunnen een bestaand eindpunt eenmaal onderzoeken om het concrete type ervan te bepalen. Tijdens runtime worden protocollen niet automatisch gedetecteerd of gewisseld.

De containermodus ondersteunt dezelfde Signal-bewerkingen als de systeemeigen modus wanneer de container overeenkomstige API's beschikbaar stelt: verzenden, ontvangen, bijlagen, typindicatoren, lees- en bekekenbevestigingen, reacties, groepen en opgemaakte tekst. OpenClaw vertaalt systeemeigen Signal-RPC-aanroepen naar de REST-payloads van de container, waaronder `group.{base64(internal_id)}`-groeps-ID's en `text_mode: "styled"` voor opgemaakte tekst.

Operationele opmerkingen:

- Gebruik `MODE=json-rpc` voor ontvangst. Door `MODE=normal` kan `/v1/about` gezond lijken, maar `/v1/receive/{account}` voert geen WebSocket-upgrade uit, waardoor de controle van de ontvangststream van de container mislukt.
- Stel `kind: "container"` in voor de bbernhard REST-API en `kind: "external-native"` voor systeemeigen `signal-cli` JSON-RPC/SSE.
- Voor het downloaden van containerbijlagen gelden dezelfde limieten voor mediabytes als in de systeemeigen modus. Te grote antwoorden worden geweigerd voordat ze volledig worden gebufferd wanneer de server `Content-Length` verzendt, en anders tijdens het streamen.

## Toegangsbeheer (privéberichten + groepen)

Privéberichten:

- Standaard: `channels.signal.dmPolicy = "pairing"`.
- Onbekende afzenders krijgen een koppelcode; berichten worden genegeerd totdat ze zijn goedgekeurd (codes verlopen na 1 uur).
- Keur goed via `openclaw pairing list signal` en `openclaw pairing approve signal <CODE>`.
- Koppeling is de standaard tokenuitwisseling voor Signal-privéberichten. Details: [Koppeling](/nl/channels/pairing)
- Afzenders met alleen een UUID (van `sourceUuid`) worden als `uuid:<id>` opgeslagen in `channels.signal.allowFrom`.

Groepen:

- `channels.signal.groupPolicy = open | allowlist | disabled`.
- `channels.signal.groupAllowFrom` bepaalt welke groepen of afzenders groepsantwoorden kunnen activeren wanneer `allowlist` is ingesteld; vermeldingen kunnen Signal-groeps-ID's (onbewerkt, `group:<id>` of `signal:group:<id>`), telefoonnummers van afzenders, `uuid:<id>`-waarden of `*` zijn.
- `channels.signal.groups["<group-id>" | "*"]` kan groepsgedrag overschrijven met `requireMention`, `tools` en `toolsBySender`.
- Gebruik `channels.signal.accounts.<id>.groups` voor overschrijvingen per account in configuraties met meerdere accounts.
- Het toevoegen van een Signal-groep aan de toelatingslijst via `groupAllowFrom` schakelt de vermeldingsbeperking niet automatisch uit. Een specifiek geconfigureerde `channels.signal.groups["<group-id>"]`-vermelding verwerkt elk groepsbericht, tenzij `requireMention=true` is ingesteld.
- Met `requireMention=true` worden systeemeigen @vermeldingen van Signal aan de hand van gestructureerde vermeldingsmetadata vergeleken met het telefoonnummer of de `accountUuid` van het botaccount. Geconfigureerde `mentionPatterns` blijven beschikbaar als terugvaloptie voor platte tekst.
- Runtime-opmerking: als `channels.signal` volledig ontbreekt, valt de runtime voor groepscontroles terug op `groupPolicy="allowlist"` (zelfs als `channels.defaults.groupPolicy` is ingesteld).

Groep met vermeldingsbeperking en begrensde context:

```json5
{
  channels: {
    signal: {
      account: "+15551234567",
      accountUuid: "bot-signal-uuid",
      groupPolicy: "allowlist",
      groupAllowFrom: ["group:<signal-group-id>"],
      historyLimit: 8,
      groups: {
        "<signal-group-id>": { requireMention: true },
      },
    },
  },
  messages: {
    groupChat: {
      mentionPatterns: ["\\bopenclaw\\b"],
    },
  },
}
```

Toegestane groepsberichten waarin de bot niet wordt vermeld, blijven onbeantwoord en worden alleen bewaard in het begrensde venster met wachtende geschiedenis. Wanneer een latere systeemeigen @vermelding of tekstuele terugvalvermelding de bot activeert, neemt OpenClaw die recente context op en antwoordt het in dezelfde groep. De inhoud van overgeslagen bijlagen wordt niet gedownload; deze kan alleen als compacte mediaplaatsaanduiding in de wachtende context verschijnen.

## Hoe het werkt (gedrag)

- Systeemeigen modus: `signal-cli` wordt als daemon uitgevoerd; de Gateway leest gebeurtenissen via SSE.
- Containermodus: de Gateway verzendt via de REST-API en ontvangt via WebSocket.
- Inkomende berichten worden genormaliseerd naar de gedeelde kanaalenvelop.
- Antwoorden worden altijd teruggestuurd naar hetzelfde nummer of dezelfde groep.
- Antwoorden op inkomende berichten bevatten systeemeigen citaatmetadata van Signal wanneer de backend de tijdstempel en auteur van het inkomende bericht accepteert; als citaatmetadata ontbreekt of wordt geweigerd, verzendt OpenClaw het antwoord als een normaal bericht.
- Configureer het gebruik van systeemeigen citaten met `channels.signal.replyToMode = off | first | all | batched`, of `channels.signal.replyToModeByChatType.direct/group` voor overschrijvingen per chattype. Waarden op accountniveau onder `channels.signal.accounts.<id>` hebben voorrang.

## Media en limieten

- Uitgaande tekst wordt opgedeeld volgens `channels.signal.textChunkLimit` (standaard 4000).
- Optionele opdeling op nieuwe regels: stel `channels.signal.streaming.chunkMode="newline"` in om eerst op lege regels (alineagrenzen) te splitsen en daarna op lengte.
- Bijlagen worden ondersteund (base64 opgehaald uit `signal-cli`).
- Spraaknotitiebijlagen gebruiken de bestandsnaam `signal-cli` als MIME-terugval wanneer `contentType` ontbreekt, zodat audiotranscriptie AAC-spraakmemo's nog steeds kan classificeren.
- Standaard medialimiet: `channels.signal.mediaMaxMb` (standaard 8).
- Gebruik `channels.signal.ignoreAttachments` om het downloaden van media voor elk transport over te slaan.
- De context van de groepsgeschiedenis gebruikt `channels.signal.historyLimit` (of `channels.signal.accounts.*.historyLimit`) en valt terug op `messages.groupChat.historyLimit`. Stel `0` in om dit uit te schakelen (standaard 50).

## Typindicatoren en leesbevestigingen

- **Typindicatoren**: OpenClaw verzendt typsignalen via `signal-cli sendTyping` en vernieuwt ze zolang een antwoord wordt uitgevoerd.
- **Leesbevestigingen**: wanneer `channels.signal.sendReadReceipts` waar is, stuurt OpenClaw leesbevestigingen door voor toegestane privéberichten.
- `signal-cli` stelt geen leesbevestigingen voor groepen beschikbaar.

## Statusreacties tijdens de levenscyclus

Stel `messages.statusReactions.enabled: true` in om Signal de gedeelde levenscyclusreacties voor in wachtrij/denken/tool/Compaction/gereed/fout bij inkomende beurten te laten weergeven. Signal gebruikt de tijdstempel van het inkomende bericht als reactiedoel; groepsreacties worden verzonden met de Signal-groeps-ID en de oorspronkelijke afzender als doelauteur.

Statusreacties vereisen ook een bevestigingsreactie en een overeenkomende `messages.ackReactionScope` (`direct`, `group-all`, `group-mentions` of `all`). Stel `channels.signal.reactionLevel: "off"` in om Signal-statusreacties uit te schakelen.

Signal herstelt de oorspronkelijke bevestigingsreactie na de uiteindelijke status gereed/fout.

## Reacties (berichtentool)

Gebruik `message action=react` met `channel=signal`.

- Doelen: E.164-nummer of UUID van de afzender (gebruik `uuid:<id>` uit de koppelingsuitvoer; een losse UUID werkt ook).
- `messageId` is de Signal-tijdstempel van het bericht waarop je reageert.
- Groepsreacties vereisen `targetAuthor` of `targetAuthorUuid`.

```text
message action=react channel=signal target=uuid:123e4567-e89b-12d3-a456-426614174000 messageId=1737630212345 emoji=🔥
message action=react channel=signal target=+15551234567 messageId=1737630212345 emoji=🔥 remove=true
message action=react channel=signal target=signal:group:<groupId> targetAuthor=uuid:<sender-uuid> messageId=1737630212345 emoji=✅
```

Configuratie:

- `channels.signal.actions.reactions`: reactieacties in-/uitschakelen (standaard waar).
- `channels.signal.reactionLevel`: `off | ack | minimal | extensive` (standaard `minimal`).
  - `off`/`ack` schakelt agentreacties uit (berichtentool `react` geeft fouten).
  - `minimal`/`extensive` schakelt agentreacties in en stelt het begeleidingsniveau in.
- Overschrijvingen per account: `channels.signal.accounts.<id>.actions.reactions`, `channels.signal.accounts.<id>.reactionLevel`.

## Goedkeuringsreacties

Signal-prompts voor uitvoerings- en Plugin-goedkeuring gebruiken de routeringsblokken `approvals.exec` en `approvals.plugin` op het hoogste niveau. Signal heeft geen `channels.signal.execApprovals`-blok.

- `👍` keurt eenmaal goed.
- `👎` weigert.
- Gebruik `/approve <id> allow-always` wanneer een verzoek permanente goedkeuring aanbiedt.

Voor het afhandelen van goedkeuringsreacties zijn expliciete Signal-goedkeurders vereist uit `channels.signal.allowFrom`, `channels.signal.defaultTo` of de overeenkomende velden op accountniveau. Rechtstreekse uitvoeringsgoedkeuringsprompts binnen dezelfde chat kunnen de dubbele lokale terugval `/approve` nog steeds onderdrukken zonder expliciete goedkeurders; bij groepsgoedkeuringen zonder goedkeurders blijft de lokale terugval zichtbaar.

## Vraagreacties

Voor een `ask_user`-prompt met één niet-geheime enkelkeuzevraag en één tot vier opties toont Signal `1️⃣` tot en met `4️⃣` naast de optielabels. Reageer op de afgeleverde prompt met het overeenkomende nummer om de vraag te beantwoorden. OpenClaw controleert of de reactie is gericht op het door de bot geschreven bericht en koppelt het nummer vervolgens via de Gateway aan de canonieke optie. Verouderde of dubbele tikken worden genegeerd. Prompts met meerdere vragen, meervoudige selectie en vrije tekst kunnen alleen via een tekstantwoord worden beantwoord; de normale toelatingsregels voor privéberichten en groepen van Signal autoriseren de afzender.

## Bezorgingsdoelen (CLI/Cron)

- Privéberichten: `signal:+15551234567` (of gewoon E.164).
- UUID-privéberichten: `uuid:<id>` (of losse UUID).
- Groepen: `signal:group:<groupId>`.
- Gebruikersnamen: `username:<name>` (indien ondersteund door je Signal-account).

## Aliassen

Configureer aliassen als stabiele namen voor terugkerende Signal-doelen. Aliassen bestaan alleen in de OpenClaw-configuratie; ze maken of bewerken geen Signal-contacten.

```json5
{
  channels: {
    signal: {
      aliases: {
        me: "+15557654321",
        jane: "uuid:123e4567-e89b-12d3-a456-426614174000",
        ops: "group:<groupId>",
      },
      defaultTo: "signal:me",
    },
  },
}
```

Gebruik aliassen overal waar Signal-bezorgingsdoelen worden geaccepteerd:

```bash
openclaw message send --channel signal --target signal:ops --message "Implementatie is voltooid"
```

Aliassen per account nemen de aliassen op het hoogste niveau over en kunnen namen toevoegen of overschrijven:

```json5
{
  channels: {
    signal: {
      aliases: {
        me: "+15557654321",
      },
      accounts: {
        work: {
          aliases: {
            ops: "group:<workGroupId>",
          },
        },
      },
    },
  },
}
```

`openclaw directory peers list --channel signal` en `openclaw directory groups list --channel signal` geven de geconfigureerde aliassen weer. De Signal-directory wordt door configuratie ondersteund; deze vraagt Signal-contacten niet live op en wijzigt het Signal-account niet.

## Problemen oplossen

Voer eerst deze reeks uit:

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

Controleer daarna indien nodig de koppelingsstatus van privéberichten:

```bash
openclaw pairing list signal
```

Veelvoorkomende fouten:

- Daemon bereikbaar maar geen antwoorden: controleer `account`, `transport.kind`, de transport-URL en de ontvangstmodus.
- Privéberichten genegeerd: de afzender wacht op koppelingsgoedkeuring.
- Groepsberichten genegeerd: beperkingen voor groepsafzenders/vermeldingen blokkeren de bezorging.
- Configuratievalidatiefouten na bewerkingen: voer `openclaw doctor --fix` uit.
- Signal ontbreekt in diagnostische gegevens: controleer `channels.signal.enabled: true`.

Extra controles:

```bash
openclaw pairing list signal
pgrep -af signal-cli
openclaw logs --plain --limit 500 | grep -i "signal" | tail -20
```

Voor de triageflow: [Problemen met kanalen oplossen](/nl/channels/troubleshooting).

## Beveiligingsopmerkingen

- `signal-cli` slaat accountsleutels lokaal op (doorgaans `~/.local/share/signal-cli/data/`).
- Maak een back-up van de Signal-accountstatus vóór een servermigratie of herbouw.
- Behoud `channels.signal.dmPolicy: "pairing"`, tenzij je expliciet bredere toegang tot privéberichten wilt.
- Sms-verificatie is alleen nodig voor registratie- of herstelprocessen, maar als je de controle over het nummer/account verliest, kan herregistratie ingewikkelder worden.

## Configuratiereferentie (Signal)

Volledige configuratie: [Configuratie](/nl/gateway/configuration)

Provideropties:

- `channels.signal.enabled`: opstarten van het kanaal in-/uitschakelen.
- `channels.signal.account`: E.164 voor het botaccount.
- `channels.signal.accountUuid`: optionele UUID van het botaccount voor systeemeigen detectie van @vermeldingen en lusbeveiliging.
- `channels.signal.transport`: transport in eigendom van het account. Laat dit weg voor beheerde systeemeigen standaardwaarden.
- `channels.signal.transport.kind`: `managed-native | external-native | container`.
- `channels.signal.transport.url`: vereist voor `external-native` en `container`; optioneel voor `managed-native` wanneer het verbindingseindpunt afwijkt van de daemonbinding.
- `channels.signal.transport.cliPath`: beheerd systeemeigen pad naar `signal-cli`.
- `channels.signal.transport.configPath`: optionele beheerde systeemeigen map voor `signal-cli --config`.
- `channels.signal.transport.httpHost`, `channels.signal.transport.httpPort`: beheerde systeemeigen daemonbinding (standaard `127.0.0.1:8080`).
- `channels.signal.transport.startupTimeoutMs`: beheerde systeemeigen wachttijd bij het opstarten in ms (min. 1000, max. 120000; standaard 30000).
- `channels.signal.transport.receiveMode`: beheerde systeemeigen `on-start | manual`.
- `channels.signal.ignoreAttachments`: downloads van inkomende bijlagen voor dit account overslaan.
- `channels.signal.transport.ignoreStories`: beheerde systeemeigen schakeloptie voor verhalen.
- `channels.signal.sendReadReceipts`: leesbevestigingen doorsturen.
- `channels.signal.dmPolicy`: `pairing | allowlist | open | disabled` (standaard: koppelen).
- `channels.signal.allowFrom`: toelatingslijst voor privéberichten (E.164 of `uuid:<id>`). `open` vereist `"*"`. Signal heeft geen gebruikersnamen; gebruik telefoon-/UUID-ID's.
- `channels.signal.aliases`: OpenClaw-aliassen voor afleveringsdoelen van privéberichten of groepen.
- `channels.signal.groupPolicy`: `open | allowlist | disabled` (standaard: toelatingslijst).
- `channels.signal.groupAllowFrom`: toelatingslijst voor groepen; accepteert Signal-groeps-ID's (onbewerkt, `group:<id>` of `signal:group:<id>`), E.164-nummers van afzenders of `uuid:<id>`-waarden.
- `channels.signal.groups`: overschrijvingen per groep, geïndexeerd op Signal-groeps-ID (of `"*"`). Ondersteunde velden: `requireMention`, `tools`, `toolsBySender`.
- `channels.signal.accounts.<id>.groups`: versie per account van `channels.signal.groups` voor configuraties met meerdere accounts.
- `channels.signal.accounts.<id>.aliases`: aliassen per account, samengevoegd met aliassen op het hoogste niveau.
- `channels.signal.replyToMode`: systeemeigen modus voor antwoordcitaten, `off | first | all | batched` (standaard: `all`).
- `channels.signal.replyToModeByChatType.direct`, `channels.signal.replyToModeByChatType.group`: systeemeigen overschrijvingen voor antwoordcitaten per chattype.
- `channels.signal.accounts.<id>.replyToMode`, `channels.signal.accounts.<id>.replyToModeByChatType.direct`, `channels.signal.accounts.<id>.replyToModeByChatType.group`: overschrijvingen voor antwoordcitaten per account.
- `channels.signal.historyLimit`: maximaal aantal groepsberichten dat als context wordt opgenomen (0 schakelt dit uit).
- `channels.signal.dmHistoryLimit`: geschieden limiet voor privéberichten in gebruikersbeurten. Overschrijvingen per gebruiker: `channels.signal.dms["<phone_or_uuid>"].historyLimit`.
- `channels.signal.textChunkLimit`: grootte van uitgaande delen in tekens (standaard 4000).
- `channels.signal.streaming.chunkMode`: `length` (standaard) of `newline` om eerst op lege regels (alineagrenzen) te splitsen en daarna op lengte.
- `channels.signal.mediaMaxMb`: limiet voor inkomende/uitgaande media in MB (standaard 8).
- `channels.signal.reactionLevel`: `off | ack | minimal | extensive` (standaard `minimal`). Zie [Reacties](#reactions-message-tool).
- `channels.signal.reactionNotifications`: `off | own | all | allowlist` (standaard `own`) - wanneer de agent op de hoogte wordt gesteld van inkomende reacties van anderen.
- `channels.signal.reactionAllowlist`: afzenders van wie reacties de agent op de hoogte stellen wanneer `reactionNotifications: "allowlist"`.
- `channels.signal.streaming.block.enabled`, `channels.signal.streaming.block.coalesce`: besturingselementen voor streaming in blokmodus die door kanalen worden gedeeld. Zie [Streaming](/nl/concepts/streaming).

Gerelateerde globale opties:

- `agents.entries.*.groupChat.mentionPatterns` (terugvaloptie voor platte tekst; systeemeigen @vermeldingen van Signal worden gedetecteerd via gestructureerde metagegevens wanneer de identiteit van het botaccount is geconfigureerd).
- `messages.groupChat.mentionPatterns` (globale terugvaloptie).
- `channels.signal.responsePrefix` of een `responsePrefix` op accountniveau.

## Gerelateerd

- [Overzicht van kanalen](/nl/channels) - alle ondersteunde kanalen
- [Koppelen](/nl/channels/pairing) - authenticatie van privéberichten en koppelingsflow
- [Groepen](/nl/channels/groups) - gedrag van groepschats en toegangscontrole via vermeldingen
- [Kanaalroutering](/nl/channels/channel-routing) - sessieroutering voor berichten
- [Beveiliging](/nl/gateway/security) - toegangsmodel en beveiligingsversterking
