---
read_when:
    - DM-toegangsbeheer instellen
    - Een nieuwe iOS-/Android-Node koppelen
    - De beveiligingsstatus van OpenClaw beoordelen
summary: 'Overzicht van koppelen: keur goed wie je een privébericht kan sturen en welke nodes kunnen deelnemen'
title: Koppelen
x-i18n:
    generated_at: "2026-07-27T05:02:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: dc874d660509f59bc26795c8b3ce13f5d238cd61154c717637f5d545b995fb08
    source_path: channels/pairing.md
    workflow: 16
---

"Koppelen" is de expliciete stap voor toegangstoestemming van OpenClaw.
Deze wordt op twee plaatsen gebruikt:

1. **DM-koppeling** (wie met de bot mag communiceren)
2. **Node-koppeling** (welke apparaten/nodes zich bij het Gateway-netwerk mogen aansluiten)

Beveiligingscontext: [Beveiliging](/nl/gateway/security)

## 1) DM-koppeling (toegang tot inkomende chats)

Wanneer een kanaal is geconfigureerd met DM-beleid `pairing`, ontvangen onbekende afzenders een korte code en wordt hun bericht **niet verwerkt** totdat je toestemming geeft.

Het standaard-DM-beleid is hier gedocumenteerd: [Beveiliging](/nl/gateway/security)

`dmPolicy: "open"` is alleen openbaar wanneer de effectieve DM-toestaanlijst `"*"` bevat.
Voor installatie en validatie is dat jokerteken vereist voor openbaar toegankelijke configuraties. Als de bestaande
status `open` met concrete `allowFrom`-vermeldingen bevat, laat de runtime nog steeds
alleen die afzenders toe en verruimen goedkeuringen in het koppelingsarchief de toegang tot `open` niet.

Koppelingscodes:

- 8 tekens, hoofdletters, geen dubbelzinnige tekens (`0O1I`).
- **Verlopen na 1 uur**. De bot stuurt het koppelingsbericht alleen wanneer een nieuw verzoek wordt aangemaakt (ongeveer eenmaal per uur per afzender).
- Openstaande DM-koppelingsverzoeken zijn beperkt tot **3 per kanaalaccount**; aanvullende verzoeken worden genegeerd totdat er één verloopt of wordt goedgekeurd.

### Goedkeuren via de Control UI

Open **Settings → Channels → DM access requests**. De wachtrij combineert openstaande
verzoeken van elk geconfigureerd kanaalaccount waarvan het DM-beleid `pairing` is.
Filter op kanaal of account, controleer de afzender-ID en metagegevens en kies vervolgens
**Approve**.

Goedkeuring verleent alleen toegang tot directe berichten. Deze verleent geen toegang tot groepen. Het
goedkeuringsvenster biedt, indien ondersteund, ook deze expliciete opties:

- **De aanvrager na goedkeuring informeren**
- **Deze afzender ook de eerste opdrachteigenaar maken**, alleen weergegeven wanneer er geen
  opdrachteigenaar bestaat en de Control UI-sessie `operator.admin` heeft

Kies **Dismiss** om een openstaand verzoek te verwijderen zonder het goed te keuren. Afwijzing is
geen permanente blokkering; de afzender kan later opnieuw toegang aanvragen.

### Goedkeuren via de CLI

```bash
openclaw pairing list telegram
openclaw pairing approve telegram <CODE>
```

Voeg `--notify` toe om de aanvrager via hetzelfde kanaal te informeren. Kanalen met meerdere accounts
accepteren `--account <id>`.

In tegenstelling tot het expliciete selectievakje van de Control UI initialiseert de CLI automatisch
`commands.ownerAllowFrom` wanneer er geen opdrachteigenaar is geconfigureerd, met een vermelding
zoals `telegram:123456789`. Dit geeft nieuwe installaties een expliciete eigenaar voor
bevoorrechte opdrachten en verzoeken om uitvoering goed te keuren. Nadat er een eigenaar bestaat, verlenen latere
koppelingsgoedkeuringen alleen DM-toegang; ze voegen geen extra eigenaren toe.

<Note>
De aanmeldings-QR-code van WhatsApp koppelt een WhatsApp-account aan OpenClaw. DM-toegangsverzoeken
keuren personen goed die berichten naar dat account sturen. Dit zijn afzonderlijke processen.
</Note>

Ondersteunde kanalen (elke geïnstalleerde kanaalplugin die koppeling declareert; externe plugins zoals `openclaw-weixin` kunnen er meer toevoegen): `discord`, `feishu`, `googlechat`, `imessage`, `irc`, `line`, `matrix`, `mattermost`, `msteams`, `nextcloud-talk`, `nostr`, `signal`, `slack`, `sms`, `synology-chat`, `telegram`, `twitch`, `whatsapp`, `zalo`, `zalouser`.

### Herbruikbare afzendergroepen

Gebruik `accessGroups` op het hoogste niveau wanneer dezelfde set vertrouwde afzenders van toepassing moet zijn op
meerdere berichtkanalen of op zowel DM- als groepstoestaanlijsten.

Statische groepen gebruiken `type: "message.senders"` en er wordt vanuit kanaaltoestaanlijsten naar verwezen met
`accessGroup:<name>`:

```json5
{
  accessGroups: {
    operators: {
      type: "message.senders",
      members: {
        discord: ["discord:123456789012345678"],
        telegram: ["987654321"],
        whatsapp: ["+15551234567"],
      },
    },
  },
  channels: {
    telegram: { dmPolicy: "allowlist", allowFrom: ["accessGroup:operators"] },
    whatsapp: { groupPolicy: "allowlist", groupAllowFrom: ["accessGroup:operators"] },
  },
}
```

Toegangsgroepen worden hier uitgebreid gedocumenteerd: [Toegangsgroepen](/nl/channels/access-groups)

### Waar de status wordt opgeslagen

Opgeslagen in de gedeelde SQLite-statusdatabase op
`~/.openclaw/state/openclaw.sqlite`:

- openstaande verzoeken in `channel_pairing_requests`
- goedgekeurde afzenders in `channel_pairing_allow_entries`

Gedrag voor accountafbakening:

- elk verzoek en elke goedgekeurde afzender wordt geïdentificeerd door kanaal en account
- de runtime leest alleen de canonieke SQLite-rijen; deze voegt geen verouderde bestanden samen

Oudere gateways schreven `<channel>-pairing.json` en
`<channel>-<accountId>-allowFrom.json` onder `~/.openclaw/credentials/`.
De opstartmigratie en `openclaw doctor --fix` importeren deze bestanden in SQLite en
verwijderen elke bron na een geslaagde import. Behandel de SQLite-database als
gevoelig, omdat deze rijen de toegang tot je assistent beheren.

<Note>
Het archief met de koppelings-toestaanlijst is bedoeld voor DM-toegang. Groepsautorisatie is afzonderlijk.
Het goedkeuren van een DM-koppelingscode staat die afzender niet automatisch toe om groepsopdrachten
uit te voeren of de bot in groepen te besturen. De initialisatie van de eerste eigenaar is een afzonderlijke configuratiestatus
in `commands.ownerAllowFrom`, en de bezorging van groepschats blijft de
groepstoestaanlijsten van het kanaal volgen (bijvoorbeeld `groupAllowFrom`, `groups` of overschrijvingen per groep
of per onderwerp, afhankelijk van het kanaal).
</Note>

## 2) Node-apparaten koppelen (iOS/Android/macOS/headless nodes)

Nodes maken als **apparaten** met `role: node` verbinding met de Gateway. De Gateway
maakt een apparaatkoppelingsverzoek aan dat moet worden goedgekeurd.

### Koppelen via de Control UI (aanbevolen)

Gebruik een reeds verbonden Control UI-sessie met `operator.admin`-toegang:

1. Open de Control UI en ga naar **Settings → Devices**.
2. Klik op de pagina **Devices** op **Pair mobile device**.
3. Behoud **Full access (recommended)** of selecteer **Limited access** om
   administratieve Gateway-bedieningselementen weg te laten.
4. Klik op **Create setup code**.
5. Open op je telefoon de OpenClaw-app → **Settings** → **Gateway**.
6. Scan de QR-code of plak de installatiecode en maak vervolgens verbinding.

Officiële OpenClaw-apps voor iOS en Android worden automatisch goedgekeurd wanneer hun
metagegevens voor de installatiecode overeenkomen. Als **Pending approval** een verzoek toont (bijvoorbeeld
voor een niet-officiële client of niet-overeenkomende metagegevens), controleer dan de rol en
bereiken voordat je het goedkeurt.

De knop is uitgeschakeld wanneer de huidige Control UI-sessie geen
beheerderstoegang heeft. Gebruik in dat geval de onderstaande CLI-goedkeuringsprocedure vanaf de Gateway-host.

### Koppelen via Telegram

Als je de `device-pair`-plugin gebruikt, kun je de eerste apparaatkoppeling volledig vanuit Telegram uitvoeren:

1. Stuur in Telegram een bericht naar je bot: `/pair`
2. De bot antwoordt met twee berichten: een instructiebericht en een afzonderlijk **installatiecode**-bericht (eenvoudig te kopiëren en plakken in Telegram).
3. Open op je telefoon de OpenClaw-app voor iOS → Settings → Gateway.
4. Scan de QR-code (`/pair qr`) of plak de installatiecode en maak verbinding.
5. De officiële mobiele app maakt automatisch verbinding. Als `/pair pending` een
   verzoek toont, controleer dan de rol en bereiken voordat je het goedkeurt.

De installatiecode is een met base64 gecodeerde JSON-payload die het volgende bevat:

- `url`: de Gateway-WebSocket-URL (`ws://...` of `wss://...`)
- `urls`: indien beschikbaar, de geordende LAN-/Tailnet-routes die de mobiele app kan proberen
- `bootstrapToken`: een eenmalig bootstrap-token voor de eerste koppelingshandshake; de Gateway laat dit na 10 minuten verlopen

Voer `/pair cleanup` uit om ongebruikte installatiecodes ongeldig te maken nadat de koppeling is voltooid.

Dat bootstrap-token bevat het ingebouwde bootstrap-profiel voor koppeling:

- een veilige `wss://`-installatie (of loopback op dezelfde host) gebruikt standaard `node` plus volledige
  systeemeigen mobiele `operator`-toegang
- het overgedragen `node`-token blijft `scopes: []`
- het standaard overgedragen `operator`-token bevat `operator.admin`,
  `operator.approvals`, `operator.read`, `operator.talk.secrets` en
  `operator.write`
- Control UI **Limited access** en `openclaw qr --limited` laten
  `operator.admin` weg, terwijl de andere operatorbereiken behouden blijven
- een installatie via LAN met tekst zonder opmaak en `ws://` gebruikt automatisch hetzelfde beperkte profiel;
  configureer `wss://` of Tailscale Serve en genereer een nieuwe code voor volledige toegang
- latere tokenrotatie/-intrekking blijft begrensd door zowel het goedgekeurde
  rolcontract van het apparaat als de operatorbereiken van de aanroepende sessie

Behandel de installatiecode als een wachtwoord zolang deze geldig is.

De iOS- en Android-pagina's **Settings → Gateway** tonen **Full**- of **Limited**-
toegang. Om een beperkt toegankelijke telefoon te upgraden, configureer je eerst een veilige `wss://`- of
Tailscale Serve-route. Genereer vervolgens een nieuwe installatiecode voor volledige toegang, scan of plak
deze op die instellingenpagina en maak opnieuw verbinding.

Gebruik voor koppeling via Tailscale, het openbare internet of andere externe mobiele verbindingen Tailscale Serve/Funnel
of een andere `wss://`-Gateway-URL. Installatiecodes met `ws://` en tekst zonder opmaak worden alleen geaccepteerd
voor loopback, privé-LAN-adressen, `.local`-Bonjour-hosts en de Android-
emulatorhost. Niet-loopbackroutes met tekst zonder opmaak krijgen beperkte toegang. Tailnet-
CGNAT-adressen, `.ts.net`-namen en openbare hosts blijven gesloten voordat
QR-/installatiecodes worden uitgegeven.

Voor `gateway.bind=lan`-installatie-URL's detecteert OpenClaw permanente Tailscale Serve-
HTTPS-hoofdroutes die de loopbackpoort van de actieve Gateway als proxy doorgeven en adverteert deze
naast de LAN-route. De installatieopdracht voegt deze terugvaloptie alleen toe
voor `lan`; `custom` en `tailnet` behouden hun expliciet geadverteerde routes. De
iOS-app test de geadverteerde routes op volgorde en slaat het eerste bereikbare
eindpunt op.

### Een Node-apparaat goedkeuren

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw devices reject <requestId>
```

Wanneer een expliciete goedkeuring wordt geweigerd omdat de goedkeurende sessie van het gekoppelde apparaat
is geopend met alleen een koppelingsbereik, probeert de CLI hetzelfde verzoek opnieuw met
`operator.admin`. Hierdoor kan een bestaand gekoppeld apparaat met beheerdersmogelijkheden een nieuwe
Control UI-/browserkoppeling herstellen zonder het koppelingsarchief handmatig te bewerken. De
Gateway valideert de opnieuw geprobeerde verbinding nog steeds; tokens die niet kunnen verifiëren
met `operator.admin` blijven geblokkeerd.

Als hetzelfde apparaat het opnieuw probeert met andere verificatiegegevens (bijvoorbeeld een andere
rol, andere bereiken of een andere openbare sleutel), wordt het vorige openstaande verzoek vervangen en wordt er een nieuwe
`requestId` aangemaakt.

<Note>
Een reeds gekoppeld apparaat krijgt niet stilzwijgend ruimere toegang. Als het opnieuw verbinding maakt en om meer bereiken of een ruimere rol vraagt, behoudt OpenClaw de bestaande goedkeuring ongewijzigd en maakt het een nieuw openstaand upgradeverzoek aan. Gebruik `openclaw devices list` om de momenteel goedgekeurde toegang te vergelijken met de nieuw aangevraagde toegang voordat je deze goedkeurt.
</Note>

### Optionele automatische goedkeuring van nodes uit vertrouwde CIDR's

Apparaatkoppeling blijft standaard handmatig. Voor streng beheerde Node-netwerken
kun je automatische goedkeuring van een eerste Node-koppeling inschakelen met expliciete CIDR's of exacte IP-adressen:

```json5
{
  gateway: {
    nodes: {
      pairing: {
        autoApproveCidrs: ["192.168.1.0/24"],
      },
    },
  },
}
```

Dit is alleen van toepassing op nieuwe `role: node`-koppelingsverzoeken zonder aangevraagde
bereiken. Clients voor operators, browsers, de Control UI en WebChat vereisen nog steeds handmatige
goedkeuring. Wijzigingen in rol, bereik, metagegevens en openbare sleutel vereisen nog steeds handmatige
goedkeuring.

### Opslag van Node-koppelingsstatus

Opgeslagen in de gedeelde SQLite-statusdatabase op `~/.openclaw/state/openclaw.sqlite`:

- openstaande apparaatkoppelingsverzoeken (kortstondig; ze verlopen na 5 minuten)
- gekoppelde apparaten en tokens

Oudere gateways bewaarden deze status in `~/.openclaw/devices/*.json`; die bestanden worden
bij het opstarten van de Gateway in SQLite geïmporteerd en met het achtervoegsel `.migrated` gearchiveerd.

### Opmerkingen

- De API `node.pair.*` (CLI: `openclaw nodes pending|approve|reject|remove|rename`) beheert
  goedkeuringen voor Node-mogelijkheden die in dezelfde gekoppelde apparaatrecords zijn opgeslagen. WS-nodes
  vereisen nog steeds apparaatkoppeling; zie [Node-koppeling](/nl/gateway/pairing).
- De koppelingsrecord is de duurzame bron van waarheid voor goedgekeurde rollen. Actieve
  apparaattokens blijven beperkt tot die set goedgekeurde rollen; een losse tokenvermelding
  buiten de goedgekeurde rollen verleent geen nieuwe toegang.

## Gerelateerde documentatie

- Beveiligingsmodel + promptinjectie: [Beveiliging](/nl/gateway/security)
- Veilig bijwerken (voer doctor uit): [Bijwerken](/nl/install/updating)
- Kanaalconfiguraties:
  - Telegram: [Telegram](/nl/channels/telegram)
  - WhatsApp: [WhatsApp](/nl/channels/whatsapp)
  - Signal: [Signal](/nl/channels/signal)
  - iMessage: [iMessage](/nl/channels/imessage)
  - Discord: [Discord](/nl/channels/discord)
  - Slack: [Slack](/nl/channels/slack)
