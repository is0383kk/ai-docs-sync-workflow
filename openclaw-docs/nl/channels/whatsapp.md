---
read_when:
    - Werken aan het gedrag van WhatsApp-/webkanalen of routering van het Postvak IN
summary: Ondersteuning voor het WhatsApp-kanaal, toegangscontroles, afleveringsgedrag en beheer
title: WhatsApp
x-i18n:
    generated_at: "2026-07-27T05:02:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7489b37f91775868d0694daea8a0958ee000d1411674d1800bb1e77df5961e68
    source_path: channels/whatsapp.md
    workflow: 16
---

Status: productierijp via WhatsApp Web (Baileys). De gateway beheert de gekoppelde sessie(s); er is geen afzonderlijk Twilio WhatsApp-kanaal.

## Installeren

`openclaw onboard` en `openclaw channels add --channel whatsapp` vragen om de plugin te installeren wanneer je deze voor het eerst selecteert; `openclaw channels login --channel whatsapp` biedt dezelfde installatiestroom als de plugin ontbreekt. Ontwikkelcheck-outs gebruiken het lokale pluginpad; stabiele/bèta-installaties installeren eerst `@openclaw/whatsapp` vanuit ClawHub en vallen terug op npm. De WhatsApp-runtime wordt buiten het kernpakket van OpenClaw op npm geleverd, zodat de runtime-afhankelijkheden bij de externe plugin blijven. Handmatige installatie:

```bash
openclaw plugins install clawhub:@openclaw/whatsapp
```

Gebruik het kale npm-pakket (`@openclaw/whatsapp`) alleen voor de registerfallback; zet een exacte versie alleen vast voor een reproduceerbare installatie.

<CardGroup cols={3}>
  <Card title="Koppelen" icon="link" href="/nl/channels/pairing">
    Het standaard DM-beleid voor onbekende afzenders is koppelen.
  </Card>
  <Card title="Probleemoplossing voor kanalen" icon="wrench" href="/nl/channels/troubleshooting">
    Kanaaloverschrijdende diagnostiek en herstelprocedures.
  </Card>
  <Card title="Gateway-configuratie" icon="settings" href="/nl/gateway/configuration">
    Volledige configuratiepatronen en voorbeelden voor kanalen.
  </Card>
</CardGroup>

## Snelle configuratie

<Steps>
  <Step title="Toegangsbeleid configureren">

```json5
{
  channels: {
    whatsapp: {
      dmPolicy: "pairing",
      allowFrom: ["+15551234567"],
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15551234567"],
    },
  },
}
```

  </Step>

  <Step title="WhatsApp koppelen (QR)">

```bash
openclaw channels login --channel whatsapp
```

    Aanmelden werkt uitsluitend via QR. Zorg op externe hosts of hosts zonder grafische interface vóór het aanmelden voor een betrouwbare manier om de actuele QR-code aan de telefoon te leveren; in de terminal weergegeven QR-codes, schermafbeeldingen of chatbijlagen kunnen onderweg verlopen.

    Voor een specifiek account:

```bash
openclaw channels login --channel whatsapp --account work
```

    Om vóór het aanmelden een bestaande/aangepaste authenticatiemap te koppelen:

```bash
openclaw channels add --channel whatsapp --account work --auth-dir /path/to/wa-auth
openclaw channels login --channel whatsapp --account work
```

  </Step>

  <Step title="De gateway starten">

```bash
openclaw gateway
```

  </Step>

  <Step title="Het eerste DM-toegangsverzoek goedkeuren (koppelingsmodus)">

    Open **Settings → Channels → DM access requests**, zoek het WhatsApp-account
    en keur de afzender goed. Als je liever de CLI gebruikt:

```bash
openclaw pairing list whatsapp
openclaw pairing approve whatsapp <CODE>
```

    DM-toegangsverzoeken verlopen na 1 uur; er kunnen maximaal 3 verzoeken per
    account in behandeling zijn. Deze goedkeuring staat los van de WhatsApp-aanmeldings-QR
    waarmee het account zelf wordt gekoppeld.

  </Step>
</Steps>

<Note>
Een afzonderlijk WhatsApp-nummer wordt aanbevolen (de configuratie en metagegevens zijn hiervoor geoptimaliseerd), maar configuraties met een persoonlijk nummer/zelfchat worden volledig ondersteund.
</Note>

## Implementatiepatronen

<AccordionGroup>
  <Accordion title="Afzonderlijk nummer (aanbevolen)">
    - afzonderlijke WhatsApp-identiteit voor OpenClaw
    - duidelijkere DM-toelatingslijsten en routeringsgrenzen
    - kleinere kans op verwarring met zelfchat

    ```json5
    {
      channels: {
        whatsapp: {
          dmPolicy: "allowlist",
          allowFrom: ["+15551234567"],
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Fallback voor persoonlijk nummer">
    Onboarding ondersteunt de modus voor persoonlijke nummers en schrijft een basisconfiguratie die geschikt is voor zelfchat: `dmPolicy: "allowlist"`, `allowFrom` inclusief je eigen nummer, `selfChatMode: true`. Runtime-beveiligingen voor zelfchat gebruiken het gekoppelde eigen nummer plus `allowFrom` als sleutel.
  </Accordion>
</AccordionGroup>

## Runtimemodel

- De gateway beheert de WhatsApp-socket en de herverbindingslus.
- Een watchdog volgt twee signalen onafhankelijk: ruwe WhatsApp Web-transportactiviteit en activiteit van applicatieberichten. Een stille maar verbonden sessie wordt niet opnieuw gestart alleen omdat er onlangs geen bericht is binnengekomen; een herverbinding wordt alleen afgedwongen wanneer er gedurende een vast intern venster (niet door de gebruiker configureerbaar) geen transportframes meer binnenkomen of applicatieberichten langer dan 4x de normale berichttime-out uitblijven. Direct na een herverbinding voor een onlangs actieve sessie gebruikt dat eerste venster de kortere normale berichttime-out in plaats van het 4x-venster. OpenClaw kan automatisch antwoorden op offlineberichten die Baileys vroeg tijdens die herverbinding aflevert, begrensd door de levensduur van de deduplicatie van inkomende bericht-ID's; bij de eerste opstart blijft de korte beveiliging tegen verouderde geschiedenis actief.
- Voor uitgaande verzendingen is een actieve WhatsApp-listener voor het doelaccount vereist; anders mislukken verzendingen direct.
- Groepsverzendingen voegen systeemeigen vermeldingsmetagegevens toe voor `@+<digits>`- en `@<digits>`-tokens (in tekst en mediabijschriften) wanneer het token overeenkomt met de huidige metagegevens van deelnemers, ook voor groepen op basis van LID.
- Status- en broadcastchats (`@status`, `@broadcast`) worden genegeerd.
- Directe chats gebruiken DM-sessieregels (`session.dmScope`; de standaardwaarde `main` voegt DM's samen in de hoofdsessie van de agent). Groepssessies worden per JID geïsoleerd (`agent:<agentId>:whatsapp:group:<jid>`).
- WhatsApp Channels/Newsletters kunnen expliciete uitgaande doelen zijn via hun systeemeigen `@newsletter`-JID, waarbij kanaalsessiemetagegevens (`agent:<agentId>:whatsapp:channel:<jid>`) worden gebruikt in plaats van DM-semantiek.
- Het WhatsApp Web-transport respecteert standaard proxy-omgevingsvariabelen op de gatewayhost (`HTTPS_PROXY`, `HTTP_PROXY`, `NO_PROXY`, varianten in kleine letters). Geef de voorkeur aan proxyconfiguratie op hostniveau boven instellingen per kanaal.

## De huidige aanvrager bellen met MeowCaller (experimenteel)

De plugin kan `whatsapp_call` beschikbaar stellen in agentbeurten die vanuit WhatsApp zijn gestart. De plugin gebruikt [MeowCaller](https://github.com/purpshell/meowcaller) om een WhatsApp-spraakoproep naar de huidige geautoriseerde aanvrager te plaatsen en na beantwoording een OpenClaw-TTS-bericht af te spelen. De tool heeft geen parameter voor het bestemmingsnummer, zodat een prompt de oproep niet kan omleiden. Standaard uitgeschakeld.

<Warning>
MeowCaller is experimenteel, heeft geen getagde release en gebruikt een afzonderlijk gekoppelde whatsmeow-sessie voor gekoppelde apparaten — deze kan de Baileys-referenties van de plugin niet hergebruiken. Door koppeling wordt nog een gekoppeld apparaat aan hetzelfde WhatsApp-account toegevoegd; scan met de identiteit die OpenClaw gebruikt. De modus voor een persoonlijk nummer/zelfchat kan zichzelf niet bellen; gebruik een afzonderlijk OpenClaw-nummer om je persoonlijke nummer te bellen.
</Warning>

<Steps>
  <Step title="Experimentele oproepen inschakelen">

    Voeg `actions.calls: true` toe aan de configuratie van het WhatsApp-kanaal en start de gateway opnieuw:

```json
{
  "channels": {
    "whatsapp": {
      "actions": {
        "calls": true
      }
    }
  }
}
```

    Als deze ontbreekt of `false` is, stelt OpenClaw de tool `whatsapp_call` niet beschikbaar.

  </Step>

  <Step title="De beoordeelde MeowCaller-CLI installeren">

    De adapter verwacht een uitvoerbaar bestand `meowcaller` in `PATH` van de gatewayhost. Bouw de beoordeelde branch totdat [MeowCaller PR #7](https://github.com/purpshell/meowcaller/pull/7) is samengevoegd:

```bash
git clone --branch feat/send-only-notify https://github.com/steipete/meowcaller.git
cd meowcaller
git checkout 752050471fc2bf7a8cdfbf7dbd3cd4e865d85d3f
mkdir -p "$HOME/.local/bin"
go build -o "$HOME/.local/bin/meowcaller" ./cmd/meowcaller
```

    Zorg dat `$HOME/.local/bin` in `PATH` van de gatewayservice staat. Deze revisie heeft expliciete opdrachten `pair` en alleen-verzenden `notify`; `notify` opent geen microfoon, luidspreker, videoapparaat of diagnostische opname. Vervang dit niet door de opdracht `play` van de upstream-voorbeeld-CLI.

  </Step>

  <Step title="Het gekoppelde MeowCaller-apparaat koppelen">

    Vraag de WhatsApp-agent om de oproepconfiguratie te controleren (de statusactie `whatsapp_call` meldt de accountspecifieke statusmap en koppelingsopdracht). Voor het standaardaccount:

```bash
state_dir="$HOME/.openclaw/credentials/whatsapp-calls/default"
mkdir -p "$state_dir"
chmod 700 "$state_dir"
meowcaller pair --store "$state_dir/wa-voip.db"
```

    Voer dit interactief uit, scan de QR-code via **WhatsApp > Linked devices** en wacht op `MeowCaller linked device ready`. Houd `wa-voip.db` privé — dit is de MeowCaller-sessie. Niet-standaardaccounts krijgen via de statusactie hun eigen opslagpad; voer op Windows de bijbehorende PowerShell-opdracht uit.

  </Step>

  <Step title="TTS configureren en bellen vanuit WhatsApp">

    Configureer een voor telefonie geschikte [TTS-provider](/nl/tools/tts), start de gateway opnieuw en verzend vervolgens een verzoek zoals `Call me and say the build finished.` De tool bepaalt de afzender vanuit vertrouwde inkomende context, synthetiseert een tijdelijk privé-WAV-bestand, voert MeowCaller gedurende een begrensd oproepvenster uit en verwijdert het audiobestand daarna. OpenClaw geeft de opslag van het account expliciet door, wacht na beantwoorden/afspelen/ophangen op een afsluitstatus van nul en behandelt een time-out of een afsluitstatus anders dan nul als een mislukte toolaanroep.

  </Step>
</Steps>

Beperkingen: uitsluitend één-op-één uitgaande audio-oproepen, geen willekeurige bestemmingsnummers, geen gedeelde authenticatie met de chatverbinding, geen oproepen naar zichzelf vanuit de modus voor een persoonlijk nummer/zelfchat, gesynthetiseerde audio beperkt tot 60 seconden, geen ontvangstbevestiging voor hoorbaarheid aan de telefoonkant behalve de voltooiing van beantwoorden/afspelen/ophangen door MeowCaller, en OpenClaw stopt het begeleidende proces na een begrensd venster van 115-175 seconden (voor de verbindings-, beantwoordings-, afspeel- en afsluitfasen van MeowCaller).

## Goedkeuringsprompts

WhatsApp kan goedkeuringsprompts voor uitvoering en plugins weergeven als `👍`-/`👎`-reacties, geregeld door de configuratie voor het doorsturen van goedkeuringen op het hoogste niveau:

```json5
{
  approvals: {
    exec: {
      enabled: true,
      mode: "session",
    },
    plugin: {
      enabled: true,
      mode: "targets",
      targets: [{ channel: "whatsapp", to: "+15551234567" }],
    },
  },
}
```

`approvals.exec` en `approvals.plugin` zijn onafhankelijk; als WhatsApp als kanaal wordt ingeschakeld, wordt alleen het transport gekoppeld en wordt niets verzonden tenzij de overeenkomende goedkeuringsfamilie is ingeschakeld en ernaartoe wordt gerouteerd. De sessiemodus levert systeemeigen emoji-goedkeuringen alleen voor goedkeuringen die vanuit WhatsApp afkomstig zijn. De doelmodus gebruikt de gedeelde doorstuurpijplijn voor expliciete doelen en maakt geen afzonderlijke fan-out naar DM's van goedkeurders.

Voor WhatsApp-goedkeuringsreacties zijn expliciete goedkeurders vereist in `allowFrom` (of `"*"`). `defaultTo` stelt gewone standaardberichtdoelen in, geen lijst met goedkeurders. Handmatige `/approve`-opdrachten doorlopen vóór de afhandeling van de goedkeuring nog steeds het normale autorisatiepad voor WhatsApp-afzenders.

## Reacties op vragen

Voor een `ask_user`-prompt met één niet-geheime vraag met enkele keuze en één tot vier opties toont WhatsApp `1️⃣` tot en met `4️⃣` naast de optielabels. Reageer op de afgeleverde prompt met het overeenkomende nummer om de vraag te beantwoorden. OpenClaw koppelt het nummer via de Gateway aan de canonieke optie; verouderde of dubbele tikken worden genegeerd. Prompts met meerdere vragen, meerdere selecties of vrije tekst blijven uitsluitend via een tekstantwoord te beantwoorden. De normale toelatingsregels voor WhatsApp-DM's/-groepen autoriseren de reagerende afzender.

## Plugin-hooks en privacy

Inkomende WhatsApp-berichten kunnen persoonlijke inhoud, telefoonnummers, groeps-ID's, namen van afzenders en velden voor sessiecorrelatie bevatten. WhatsApp zendt inkomende `message_received`-hookpayloads niet uit naar plugins, tenzij je je hiervoor aanmeldt:

```json5
{
  channels: {
    whatsapp: {
      pluginHooks: {
        messageReceived: true,
      },
    },
  },
}
```

Beperk de aanmelding tot één account onder `channels.whatsapp.accounts.<id>.pluginHooks.messageReceived`. Schakel dit alleen in voor plugins die je vertrouwt met inkomende WhatsApp-inhoud en -ID's.

## Toegangsbeheer en activering

<Tabs>
  <Tab title="DM-beleid">
    `channels.whatsapp.dmPolicy`:

    | Waarde | Gedrag |
    | --- | --- |
    | `pairing` (standaard) | Onbekende afzenders vragen om koppeling; de eigenaar keurt dit goed |
    | `allowlist` | Alleen afzenders uit `allowFrom` worden toegelaten |
    | `open` | Vereist dat `allowFrom` `"*"` bevat |
    | `disabled` | Alle DM's blokkeren |

    `allowFrom` accepteert nummers in E.164-stijl (intern genormaliseerd). Het is alleen een toegangsbeheerlijst voor afzenders van privéberichten — deze beperkt geen expliciete uitgaande verzendingen naar groeps-JID's of kanaal-JID's van `@newsletter`.

    Overschrijving voor meerdere accounts: `channels.whatsapp.accounts.<id>.dmPolicy` (en `.allowFrom`) hebben voor dat account voorrang op standaardwaarden op kanaalniveau.

    Runtime-opmerkingen:

    - koppelingen blijven behouden in de toelatingsopslag van het kanaal en worden samengevoegd met de geconfigureerde `allowFrom`
    - geplande automatisering en de terugvalontvanger voor Heartbeat gebruiken expliciete bezorgingsdoelen of de geconfigureerde `allowFrom`; goedkeuringen van koppelingen voor privéberichten zijn niet impliciet ontvangers voor Cron/Heartbeat
    - als er geen toelatingslijst is geconfigureerd, is het gekoppelde eigen nummer standaard toegestaan
    - OpenClaw koppelt uitgaande privéberichten van `fromMe` nooit automatisch (berichten die je vanaf het gekoppelde apparaat naar jezelf stuurt)

  </Tab>

  <Tab title="Groepsbeleid en toelatingslijsten">
    Groepstoegang heeft twee lagen:

    1. **Toelatingslijst voor groepslidmaatschap** (`channels.whatsapp.groups`): als `groups` wordt weggelaten, komen alle groepen in aanmerking; indien aanwezig, fungeert deze als een toelatingslijst voor groepen (`"*"` laat alle groepen toe).
    2. **Beleid voor groepsafzenders** (`channels.whatsapp.groupPolicy` + `groupAllowFrom`): `open` omzeilt de toelatingslijst voor afzenders, `allowlist` vereist een overeenkomst met `groupAllowFrom` (of `*`), `disabled` blokkeert alle inkomende groepsberichten.

    Als `groupAllowFrom` niet is ingesteld, vallen afzendercontroles terug op `allowFrom` wanneer deze vermeldingen bevat. Toelatingslijsten voor afzenders worden vóór activering via vermelding/antwoord geëvalueerd.

    Als er helemaal geen `channels.whatsapp`-blok bestaat, valt de runtime terug op `groupPolicy: "allowlist"` (met een waarschuwing in het logboek), zelfs als `channels.defaults.groupPolicy` op iets anders is ingesteld.

    <Note>
    Het bepalen van groepslidmaatschap heeft een vangnet voor één account: als slechts één WhatsApp-account is geconfigureerd en de `accounts.<id>.groups` daarvan een expliciet leeg object (`{}`) is, wordt dit behandeld als ‘niet ingesteld’ en wordt teruggevallen op de hoofdmap `channels.whatsapp.groups`, in plaats van stilzwijgend elke groep te blokkeren. Als 2+ accounts zijn geconfigureerd, blijft een expliciet lege accountmap leeg en wordt er niet teruggevallen — zo kan één account doelbewust alle groepen uitschakelen zonder andere accounts te beïnvloeden.
    </Note>

  </Tab>

  <Tab title="Vermeldingen en /activering">
    Groepsantwoorden vereisen standaard een vermelding. Detectie van vermeldingen omvat:

    - expliciete WhatsApp-vermeldingen van de botidentiteit
    - geconfigureerde regex-patronen voor vermeldingen (`agents.entries.*.groupChat.mentionPatterns`, terugval `messages.groupChat.mentionPatterns`)
    - transcripties van inkomende spraakberichten voor geautoriseerde groepsberichten
    - impliciete detectie van antwoorden aan de bot (de afzender van het antwoord komt overeen met de botidentiteit)

    Beveiliging: citeren/antwoorden voldoet alleen aan de vermeldingscontrole — het verleent **geen** afzenderautorisatie. Met `groupPolicy: "allowlist"` blijven afzenders die niet op de toelatingslijst staan geblokkeerd, zelfs wanneer ze antwoorden op een bericht van een gebruiker die wel op de toelatingslijst staat.

    Activeringsopdracht op sessieniveau: `/activation mention` of `/activation always`. Hiermee wordt de sessiestatus bijgewerkt (niet de globale configuratie) en dit is beperkt tot de eigenaar.

  </Tab>
</Tabs>

## Geconfigureerde ACP-bindingen

WhatsApp ondersteunt permanente ACP-bindingen via `bindings[]` op het hoogste niveau:

```json5
{
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "whatsapp",
        accountId: "work",
        peer: { kind: "direct", id: "+15555550123" },
      },
    },
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "whatsapp",
        accountId: "work",
        peer: { kind: "group", id: "120363424282127706@g.us" },
      },
    },
  ],
}
```

Directe chats komen overeen met E.164-nummers; groepen komen overeen met WhatsApp-groeps-JID's. Groepstoelatingslijsten, afzenderbeleid en vermeldings-/activeringscontroles worden uitgevoerd voordat OpenClaw ervoor zorgt dat de gebonden ACP-sessie bestaat. Een overeenkomende binding beheert de route — broadcastgroepen waaieren die beurt niet uit naar gewone WhatsApp-sessies.

## Gedrag voor persoonlijke nummers en chats met jezelf

Wanneer het gekoppelde eigen nummer ook aanwezig is in `allowFrom`, worden beveiligingen voor chats met jezelf geactiveerd: leesbevestigingen voor beurten in chats met jezelf worden overgeslagen, automatisch activeringsgedrag op basis van vermeldings-JID's dat jezelf zou pingen wordt genegeerd en antwoorden vallen standaard terug op `[{identity.name}]` (of `[openclaw]`) wanneer `responsePrefix` voor het kanaal/account niet is ingesteld.

## Berichtnormalisatie en context

<AccordionGroup>
  <Accordion title="Inkomende envelop en antwoordcontext">
    Inkomende berichten worden verpakt in de gedeelde inkomende envelop. Een geciteerd antwoord voegt context in deze vorm toe:

    ```text
    [Replying to <sender> id:<stanzaId>]
    <quoted body or media placeholder>
    [/Replying]
    ```

    Antwoordmetagegevens (`ReplyToId`, `ReplyToBody`, `ReplyToSender`, JID/E.164 van afzender) worden ingevuld wanneer ze beschikbaar zijn. Als het geciteerde doel downloadbare media bevat, slaat OpenClaw deze op via de normale opslag voor inkomende media en stelt het `MediaPath`/`MediaType` beschikbaar, zodat de agent deze rechtstreeks kan inspecteren in plaats van alleen `<media:image>` te zien.

  </Accordion>

  <Accordion title="Media-aanduidingen en extractie van locaties/contacten">
    Berichten met alleen media worden genormaliseerd tot aanduidingen: `<media:image>`, `<media:video>`, `<media:audio>`, `<media:document>`, `<media:sticker>`.

    Geautoriseerde groepsspraakberichten worden vóór de vermeldingscontrole getranscribeerd wanneer de inhoud alleen `<media:audio>` is, zodat het uitspreken van de botvermelding in het spraakbericht het antwoord kan activeren. Als de transcriptie de bot nog steeds niet vermeldt, blijft deze in de geschiedenis van wachtende groepsberichten staan in plaats van de onbewerkte aanduiding.

    Locatie-inhoud wordt weergegeven als beknopte coördinatentekst. Locatielabels/-opmerkingen en contact-/vCard-gegevens worden weergegeven als omheinde niet-vertrouwde metagegevens, niet als inline prompttekst.

  </Accordion>

  <Accordion title="Injectie van wachtende groepsgeschiedenis">
    Niet-verwerkte groepsberichten worden gebufferd en als context geïnjecteerd wanneer de bot uiteindelijk wordt geactiveerd.

    - standaardlimiet: `50`
    - configuratie: `channels.whatsapp.historyLimit`, terugval `messages.groupChat.historyLimit`
    - `0` schakelt dit uit

    Injectiemarkeringen: `[Chat messages since your last reply - for context]` en `[Current message - respond to this]`.

  </Accordion>

  <Accordion title="Leesbevestigingen">
    Standaard ingeschakeld voor geaccepteerde inkomende berichten. Globaal uitschakelen:

    ```json5
    { channels: { whatsapp: { sendReadReceipts: false } } }
    ```

    Overschrijving per account: `channels.whatsapp.accounts.<id>.sendReadReceipts`. Voor beurten in chats met jezelf worden leesbevestigingen overgeslagen, zelfs wanneer ze globaal zijn ingeschakeld.

  </Accordion>
</AccordionGroup>

## Bezorging, opdelen en media

<AccordionGroup>
  <Accordion title="Tekst opdelen">
    - standaardlimiet per deel: `channels.whatsapp.textChunkLimit = 4000`
    - `channels.whatsapp.streaming.chunkMode = "length" | "newline"`; `newline` geeft de voorkeur aan alineagrenzen (lege regels) en valt vervolgens terug op opdelen met een veilige lengte

  </Accordion>

  <Accordion title="Gedrag voor uitgaande media">
    - ondersteunt payloads voor afbeeldingen, video, audio (PTT-spraakbericht) en documenten
    - audio wordt verzonden als de Baileys-payload `audio` met `ptt: true`, waardoor deze als een push-to-talk-spraakbericht wordt weergegeven; `audioAsVoice` blijft behouden in antwoordpayloads, zodat TTS-uitvoer als spraakbericht dit pad blijft gebruiken, ongeacht de bronindeling van de provider
    - native Ogg/Opus-audio wordt verzonden als `audio/ogg; codecs=opus`; al het andere (inclusief MP3-/WebM-uitvoer van Microsoft Edge TTS) wordt met `ffmpeg` getranscodeerd naar 48 kHz mono Ogg/Opus vóór PTT-bezorging
    - `/tts latest` verzendt het laatste antwoord van de assistent als één spraakbericht en onderdrukt herhaalde verzendingen van hetzelfde antwoord; `/tts chat on|off|default` regelt automatische TTS voor de huidige chat
    - `gifPlayback: true` bij videoverzending schakelt het afspelen als geanimeerde GIF in
    - `forceDocument`/`asDocument` leidt uitgaande afbeeldingen, GIF's en video's via de documentpayload van Baileys om mediacompressie van WhatsApp te voorkomen, met behoud van de bepaalde bestandsnaam en het MIME-type
    - bijschriften worden toegepast op het eerste media-item in een antwoord met meerdere media, behalve bij PTT-spraakberichten: de audio wordt eerst zonder bijschrift verzonden, waarna het bijschrift als een afzonderlijk tekstbericht wordt verzonden (WhatsApp-clients geven bijschriften bij spraakberichten niet consistent weer)
    - de mediabron kan HTTP(S), `file://` of een lokaal pad zijn

  </Accordion>

  <Accordion title="Limieten voor mediagrootte en terugvalgedrag">
    - opslaglimiet voor inkomende media en verzendlimiet voor uitgaande media: `channels.whatsapp.mediaMaxMb` (standaard `50`)
    - overschrijving per account: `channels.whatsapp.accounts.<id>.mediaMaxMb`
    - afbeeldingen worden automatisch geoptimaliseerd (formaat-/kwaliteitsaanpassing) om binnen de limieten te passen, tenzij `forceDocument`/`asDocument` om bezorging als document vraagt
    - als het verzenden van media mislukt, verzendt de terugval voor het eerste item een tekstwaarschuwing in plaats van het antwoord stilzwijgend te laten vervallen

  </Accordion>
</AccordionGroup>

## Antwoorden citeren

`channels.whatsapp.replyToMode` regelt het native citeren van antwoorden (uitgaande antwoorden citeren zichtbaar het inkomende bericht):

| Waarde             | Gedrag                                                       |
| ----------------- | -------------------------------------------------------------- |
| `"off"` (standaard) | Nooit citeren; als gewoon bericht verzenden                           |
| `"first"`         | Alleen het eerste deel van het uitgaande antwoord citeren                      |
| `"all"`           | Elk deel van het uitgaande antwoord citeren                               |
| `"batched"`       | Gebundelde antwoorden in de wachtrij citeren; directe antwoorden ongeciteerd laten |

Overschrijving per account: `channels.whatsapp.accounts.<id>.replyToMode`.

```json5
{ channels: { whatsapp: { replyToMode: "first" } } }
```

## Reactieniveau

`channels.whatsapp.reactionLevel` regelt hoe ruim de agent emoji-reacties gebruikt:

| Niveau                 | Bevestigingsreacties | Door agent geïnitieerde reacties  |
| --------------------- | ------------- | -------------------------- |
| `"off"`               | Nee            | Nee                         |
| `"ack"`               | Ja           | Nee                         |
| `"minimal"` (standaard) | Ja           | Ja, terughoudende richtlijn |
| `"extensive"`         | Ja           | Ja, aangemoedigde richtlijn   |

Overschrijving per account: `channels.whatsapp.accounts.<id>.reactionLevel`.

```json5
{ channels: { whatsapp: { reactionLevel: "ack" } } }
```

## Bevestigingsreacties

`channels.whatsapp.ackReaction` verzendt onmiddellijk een reactie bij ontvangst van een inkomend bericht, beperkt door `reactionLevel` (onderdrukt wanneer `"off"`):

```json5
{
  channels: {
    whatsapp: {
      ackReaction: {
        emoji: "👀",
        direct: true,
        group: "mentions", // altijd | vermeldingen | nooit
      },
    },
  },
}
```

Opmerkingen: wordt onmiddellijk verzonden nadat het inkomende bericht is geaccepteerd (vóór het antwoord); als `ackReaction` aanwezig is zonder `emoji`, gebruikt WhatsApp de identiteitsemoji van de gerouteerde agent, met "👀" als terugval (laat `ackReaction` weg of stel `emoji: ""` in voor geen bevestiging); fouten worden geregistreerd maar blokkeren de bezorging van het antwoord niet; groepsmodus `mentions` reageert alleen op beurten die door een vermelding zijn geactiveerd, terwijl groepsactivering `always` die controle omzeilt; WhatsApp gebruikt alleen `channels.whatsapp.ackReaction` (verouderde `messages.ackReaction` is hier niet van toepassing).

## Levenscyclusstatusreacties

Stel `messages.statusReactions.enabled: true` in om WhatsApp tijdens een beurt de bevestigingsreactie te laten vervangen in plaats van een statische ontvangstemoji te laten staan, waarbij statussen zoals in de wachtrij, nadenken, toolactiviteit, Compaction, voltooid en fout worden doorlopen:

```json5
{
  messages: {
    statusReactions: {
      enabled: true,
    },
  },
}
```

Opmerkingen: `channels.whatsapp.ackReaction` bepaalt nog steeds of privéberichten en groepen in aanmerking komen; de status in de wachtrij gebruikt dezelfde effectieve emoji als gewone bevestigingsreacties; WhatsApp heeft één botreactiepositie per bericht, dus levenscyclusupdates vervangen de huidige reactie ter plaatse en herstellen de bevestiging na de definitieve status voltooid/fout.

## Meerdere accounts en aanmeldgegevens

<AccordionGroup>
  <Accordion title="Accountselectie en standaardwaarden">
    Account-id's zijn afkomstig uit `channels.whatsapp.accounts`. De standaardaccountselectie is `default` als die aanwezig is; anders wordt het eerste geconfigureerde account-id gebruikt (alfabetisch gesorteerd). Account-id's worden intern genormaliseerd voor opzoekbewerkingen.
  </Accordion>

  <Accordion title="Paden naar aanmeldgegevens en compatibiliteit met oudere versies">
    - huidig authenticatiepad: `~/.openclaw/credentials/whatsapp/<accountId>/creds.json` (back-up: `creds.json.bak`)
    - de verouderde standaardaunthenticatie in `~/.openclaw/credentials/` wordt nog steeds herkend/gemigreerd voor standaardaccountflows

  </Accordion>

  <Accordion title="Afmeldgedrag">
    `openclaw channels logout --channel whatsapp [--account <id>]` wist de WhatsApp-authenticatiestatus voor dat account. Wanneer een Gateway bereikbaar is, stopt afmelden eerst de actieve listener voor dat account, zodat de gekoppelde sessie vóór de volgende herstart geen berichten meer ontvangt. `openclaw channels remove --channel whatsapp` stopt ook de actieve listener voordat de accountconfiguratie wordt uitgeschakeld of verwijderd.

    In verouderde authenticatiemappen blijft `oauth.json` behouden terwijl de Baileys-authenticatiebestanden worden verwijderd.

  </Accordion>
</AccordionGroup>

## Tools, acties en configuratieschrijfbewerkingen

- Ondersteuning voor agenttools omvat de WhatsApp-reactieactie (`react`).
- Actiepoorten: `channels.whatsapp.actions.reactions`, `channels.whatsapp.actions.polls` (bestaande acties hebben standaard `true`), `channels.whatsapp.actions.calls` (standaard `false`, zie MeowCaller hierboven).
- Door het kanaal geïnitieerde configuratieschrijfbewerkingen zijn standaard ingeschakeld; schakel ze uit via `channels.whatsapp.configWrites: false`.

## Probleemoplossing

<AccordionGroup>
  <Accordion title="Niet gekoppeld (QR vereist)">
    Symptoom: de kanaalstatus meldt dat het kanaal niet is gekoppeld.

```bash
openclaw channels login --channel whatsapp
openclaw channels status
```

  </Accordion>

  <Accordion title="Gekoppeld maar niet verbonden / herverbindingslus">
    Symptoom: een gekoppeld account met herhaalde verbrekingen of herverbindingspogingen.

    Stille accounts kunnen langer verbonden blijven dan de normale berichttime-out; de watchdog start alleen opnieuw wanneer de transportactiviteit van WhatsApp Web stopt, de socket wordt gesloten of activiteit op applicatieniveau langer uitblijft dan het langere veiligheidsvenster (zie Runtimemodel hierboven).

    Oplossing:

    ```bash
    openclaw channels status --probe
    openclaw doctor
    openclaw logs --follow
    openclaw gateway status
    ```

    Als de lus blijft bestaan nadat de hostconnectiviteit en timing zijn hersteld, maak je een back-up van de authenticatiemap van het account en koppel je het opnieuw:

    ```bash
    cp -a ~/.openclaw/credentials/whatsapp/<accountId> \
      ~/.openclaw/credentials/whatsapp/<accountId>.bak
    openclaw channels logout --channel whatsapp --account <accountId>
    openclaw channels login --channel whatsapp --account <accountId>
    ```

    Als `~/.openclaw/logs/whatsapp-health.log` `Gateway inactive` meldt, maar `openclaw gateway status` en `openclaw channels status --probe` beide een gezonde status tonen, voer je `openclaw doctor` uit. Op Linux waarschuwt doctor voor verouderde crontab-vermeldingen die het buiten gebruik gestelde script `~/.openclaw/bin/ensure-whatsapp.sh` aanroepen; verwijder die vermeldingen met `crontab -e` — Cron beschikt mogelijk niet over de systemd-gebruikersbusomgeving, waardoor dat oude script de status van de Gateway onjuist kan rapporteren.

  </Accordion>

  <Accordion title="QR-aanmelding verloopt achter een proxy">
    Symptoom: `openclaw channels login --channel whatsapp` mislukt voordat een bruikbare QR wordt weergegeven, met `status=408 Request Time-out` of een verbroken TLS-socketverbinding.

    Aanmelding bij WhatsApp Web gebruikt de standaardproxyomgeving van de Gateway-host (`HTTPS_PROXY`, `HTTP_PROXY`, varianten met kleine letters, `NO_PROXY`). Controleer of het Gateway-proces de proxyomgeving overneemt en of `NO_PROXY` niet overeenkomt met `mmg.whatsapp.net`.

  </Accordion>

  <Accordion title="Geen actieve listener bij verzenden">
    Uitgaande verzendingen mislukken onmiddellijk wanneer er geen actieve Gateway-listener voor het doelaccount bestaat. Controleer of de Gateway actief is en het account is gekoppeld.
  </Accordion>

  <Accordion title="Antwoord verschijnt in het transcript maar niet in WhatsApp">
    Transcriptregels registreren wat de agent heeft gegenereerd; bezorging via WhatsApp wordt afzonderlijk gecontroleerd. OpenClaw beschouwt een automatisch antwoord pas als verzonden nadat Baileys voor ten minste één zichtbare tekst- of mediaverzending een id voor een uitgaand bericht retourneert.

    Bevestigingsreacties zijn onafhankelijke ontvangstbevestigingen vóór het antwoord — een geslaagde reactie bewijst niet dat het latere tekst-/media-antwoord is geaccepteerd. Controleer de Gateway-logboeken op `auto-reply delivery failed` of `auto-reply was not accepted by WhatsApp provider`.

  </Accordion>

  <Accordion title="Groepsberichten worden onverwacht genegeerd">
    Controleer in deze volgorde: `groupPolicy`, `groupAllowFrom`/`allowFrom`, vermeldingen in de toelatingslijst van `groups`, vermeldingscontrole (`requireMention` + vermeldingspatronen) en dubbele sleutels in `openclaw.json` (latere JSON5-vermeldingen overschrijven eerdere — behoud één `groupPolicy` per bereik).

    Als `channels.whatsapp.groups` aanwezig is, kan WhatsApp nog steeds berichten uit andere groepen waarnemen, maar OpenClaw verwijdert ze vóór de sessieroutering. Voeg de groeps-JID toe aan `channels.whatsapp.groups`, of voeg `groups["*"]` toe om alle groepen toe te laten terwijl de autorisatie van afzenders onder `groupPolicy`/`groupAllowFrom` blijft vallen.

  </Accordion>

  <Accordion title="Bun-runtimewaarschuwing">
    OpenClaw-Gateways vereisen Node. Bun biedt niet de `node:sqlite`-API die door de canonieke statusopslag wordt gebruikt, en doctor migreert verouderde Bun-services naar Node.
  </Accordion>
</AccordionGroup>

## Systeemprompts

WhatsApp ondersteunt systeemprompts in Telegram-stijl voor groepen en directe chats via de maps `groups` en `direct`.

Resolutie voor groepsberichten: eerst wordt de effectieve map `groups` bepaald — als het account zelf überhaupt een sleutel `groups` definieert, vervangt die de hoofdmap `groups` volledig (geen diepe samenvoeging). Daarna wordt de prompt opgezocht in die ene resulterende map:

1. **Groepsspecifieke prompt** (`groups["<groupId>"].systemPrompt`): wordt gebruikt wanneer de groepsvermelding bestaat **en** de sleutel `systemPrompt` ervan is gedefinieerd. Een lege tekenreeks (`""`) onderdrukt het jokerteken en past geen prompt toe.
2. **Prompt met groepsjokerteken** (`groups["*"].systemPrompt`): wordt gebruikt wanneer de specifieke groepsvermelding ontbreekt of bestaat zonder een sleutel `systemPrompt`.

Resolutie voor directe berichten volgt hetzelfde patroon voor de map `direct` en `direct["*"]`.

<Note>
`dms` blijft de lichtgewicht bucket voor geschiedenisoverschrijvingen per privébericht (`dms.<id>.historyLimit`). Promptoverschrijvingen bevinden zich onder `direct`.
</Note>

<Note>
Dit gedrag waarbij het account de hoofdmap vervangt voor promptresolutie is een gewone oppervlakkige overschrijving: elke accountsleutel `groups`/`direct`, inclusief een expliciet leeg object, vervangt de hoofdmap. Dit verschilt van de hierboven beschreven controle van de toelatingslijst voor groepslidmaatschap, die een vangnet voor één account heeft voor een per ongeluk lege `groups: {}`.
</Note>

**Verschil met Telegram:** Telegram onderdrukt de hoofdwaarde `groups` voor elk account in een configuratie met meerdere accounts (zelfs voor accounts zonder eigen `groups`) om te voorkomen dat een bot groepsberichten ontvangt voor groepen waarvan deze geen lid is. WhatsApp past die beveiliging niet toe — hoofdwaarden `groups`/`direct` worden overgenomen door elk account zonder eigen overschrijving, ongeacht het aantal accounts. Definieer in een WhatsApp-configuratie met meerdere accounts de volledige map expliciet onder elk account als je prompts per account wilt.

Belangrijk gedrag:

- `channels.whatsapp.groups` is zowel een configuratiemap per groep als de toelatingslijst voor groepen op chatniveau. Op hoofd- of accountniveau betekent `groups["*"]` voor dat bereik "alle groepen worden toegelaten".
- Voeg alleen een jokerteken `systemPrompt` toe als je al wilt dat dat bereik alle groepen toelaat. Om alleen een vaste set groeps-id's in aanmerking te laten komen, herhaal je de prompt bij elke expliciet toegelaten vermelding in plaats van `groups["*"]` te gebruiken.
- Groepstoelating en afzenderautorisatie zijn afzonderlijke controles. `groups["*"]` breidt uit welke groepen door de groepsafhandeling worden verwerkt; het autoriseert niet elke afzender in die groepen — dat blijft geregeld door `groupPolicy`/`groupAllowFrom`.
- `channels.whatsapp.direct` heeft geen gelijkwaardig neveneffect voor privéberichten: `direct["*"]` levert alleen een standaardconfiguratie nadat een privébericht al is toegelaten door `dmPolicy` plus `allowFrom` of regels voor de koppelingsopslag.

Voorbeeld:

```json5
{
  channels: {
    whatsapp: {
      groups: {
        // Alleen gebruiken als alle groepen op hoofdniveau moeten worden toegelaten.
        // Geldt voor alle accounts die geen eigen groups-map definiëren.
        "*": { systemPrompt: "Standaardprompt voor alle groepen." },
      },
      direct: {
        // Geldt voor alle accounts die geen eigen direct-map definiëren.
        "*": { systemPrompt: "Standaardprompt voor alle directe chats." },
      },
      accounts: {
        work: {
          groups: {
            // Dit account definieert zijn eigen groups, waardoor groups op hoofdniveau
            // volledig worden vervangen. Definieer "*" hier ook expliciet om een jokerteken te behouden.
            "120363406415684625@g.us": {
              requireMention: false,
              systemPrompt: "Richt je op projectmanagement.",
            },
            // Alleen gebruiken als alle groepen in dit account moeten worden toegelaten.
            "*": { systemPrompt: "Standaardprompt voor werkgroepen." },
          },
          direct: {
            // Dit account definieert zijn eigen direct-map, waardoor direct-vermeldingen op hoofdniveau
            // volledig worden vervangen. Definieer "*" hier ook expliciet om een jokerteken te behouden.
            "+15551234567": { systemPrompt: "Prompt voor een specifieke directe werkchat." },
            "*": { systemPrompt: "Standaardprompt voor directe werkchats." },
          },
        },
      },
    },
  },
}
```

## Verwijzingen naar het configuratieoverzicht

Primaire verwijzing: [Configuratieoverzicht - WhatsApp](/nl/gateway/config-channels#whatsapp)

| Onderdeel        | Velden                                                                                                         |
| ---------------- | -------------------------------------------------------------------------------------------------------------- |
| Toegang          | `dmPolicy`, `allowFrom`, `groupPolicy`, `groupAllowFrom`, `groups`                                             |
| Bezorging        | `textChunkLimit`, `streaming.chunkMode`, `mediaMaxMb`, `sendReadReceipts`, `ackReaction`, `reactionLevel`      |
| Meerdere accounts | `accounts.<id>.enabled`, `accounts.<id>.authDir` en andere overschrijvingen per account                              |
| Bewerkingen      | `configWrites`, `enabled`                                                                                      |
| Bundeling van inkomende berichten | `messages.inbound.debounceMs`, `messages.inbound.byChannel.whatsapp`                                           |
| Sessiegedrag     | `session.dmScope`, `historyLimit`, `dmHistoryLimit`, `dms.<id>.historyLimit`                                   |
| Prompts          | `groups.<id>.systemPrompt`, `groups["*"].systemPrompt`, `direct.<id>.systemPrompt`, `direct["*"].systemPrompt` |

## Gerelateerd

- [Koppelen](/nl/channels/pairing)
- [Groepen](/nl/channels/groups)
- [Beveiliging](/nl/gateway/security)
- [Kanaalroutering](/nl/channels/channel-routing)
- [Routering met meerdere agents](/nl/concepts/multi-agent)
- [Probleemoplossing](/nl/channels/troubleshooting)
