---
read_when:
    - WhatsApp-groepen specifiek configureren
    - WhatsApp-activeringsmodi wijzigen (`mention` versus `always`)
    - WhatsApp-groepssessiesleutels of context voor wachtende berichten afstemmen
sidebarTitle: WhatsApp groups
summary: Afhandeling van WhatsApp-groepsberichten — activering, toelatingslijsten, sessies en contextinjectie
title: WhatsApp-groepsberichten
x-i18n:
    generated_at: "2026-07-27T04:47:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7325dd3ae64d7abca8c1de0504f294ae280394fa5dd336d2532c5eaefcb03828
    source_path: channels/group-messages.md
    workflow: 16
---

Voor het model voor groepen over meerdere kanalen (Discord, iMessage, Matrix, Microsoft Teams, QQBot, Signal, Slack, Telegram, WhatsApp, Zalo), zie [Groepen](/nl/channels/groups). Deze pagina behandelt het WhatsApp-specifieke gedrag boven op dat model: activering, toelatingslijsten voor groepen, sessiesleutels per groep en contextinjectie van wachtende berichten.

Doel: OpenClaw laten deelnemen aan WhatsApp-groepen, alleen activeren wanneer het wordt gepingd en die thread gescheiden houden van de persoonlijke DM-sessie.

<Note>
`agents.entries.*.groupChat.mentionPatterns` wordt gedeeld met de vermeldingsfilter van de andere kanalen. Stel dit voor configuraties met meerdere agents per agent in, of gebruik `messages.groupChat.mentionPatterns` als globale terugvaloptie. Als geen van beide is ingesteld, worden patronen afgeleid van de identiteitsnaam/-emoji van de agent.
</Note>

## Gedrag

- Activeringsmodi: `mention` (standaard) of `always`. `mention` vereist een ping: een echte WhatsApp-@vermelding (`mentionedJids`), een geconfigureerd regex-patroon, de E.164-cijfers van de bot ergens in de tekst of een geciteerd antwoord op een van de berichten van de bot (behalve bij zelfchatconfiguraties met een gedeeld nummer). `always` activeert de agent bij elk bericht, maar de geïnjecteerde groepsprompt draagt de agent op alleen te antwoorden wanneer dat waarde toevoegt en anders exact het stille token `NO_REPLY` (hoofdletterongevoelig) te retourneren. Standaardwaarden komen uit de configuratie (`channels.whatsapp.groups` `requireMention`) en kunnen per groep worden overschreven via `/activation`.
- Toelatingslijst voor groepen: wanneer `channels.whatsapp.groups` is ingesteld, worden alleen vermelde groeps-JID's toegelaten (neem `"*"` op om alle groepen toe te staan); berichten uit niet-vermelde groepen worden verwijderd met een aanwijzing in het logboek.
- Groepsbeleid: `channels.whatsapp.groupPolicy` bepaalt of groepsberichten worden geaccepteerd (`open|disabled|allowlist`). `allowlist` gebruikt `channels.whatsapp.groupAllowFrom` (terugvaloptie: expliciete `channels.whatsapp.allowFrom`). De standaardwaarde is `allowlist` (geblokkeerd totdat je afzenders toevoegt).
- Sessies per groep: sessiesleutels zien eruit als `agent:<agentId>:whatsapp:group:<jid>` (bij niet-standaardaccounts wordt `:thread:whatsapp-account-<accountId>` toegevoegd), zodat instructies zoals `/verbose on`, `/trace on` of `/think high` (verzonden als zelfstandige berichten) tot die groep beperkt blijven; de status van persoonlijke DM's blijft onaangetast.
- Contextinjectie: **alleen wachtende** groepsberichten (standaard 50) die _geen_ uitvoering activeerden, krijgen het voorvoegsel `[Chat messages since your last reply - for context]`, met de activerende regel onder `[Current message - respond to this]`. Het venster met wachtende berichten wordt na de uitvoering gewist; berichten die al in de sessie staan, worden niet opnieuw geïnjecteerd.
- Toeschrijving aan afzender: elke groepsregel bevat het afzenderlabel in de berichtenvelop, bijvoorbeeld `[WhatsApp <groupJid> <timestamp>] Alice (+447700900123): text`, en de identiteit van de afzender plus het groepsonderwerp en de groepsleden worden opgenomen in het blok met niet-vertrouwde gespreksmetadata.
- Tijdelijk/eenmalig bekijken: wrappers worden uitgepakt voordat tekst en vermeldingen worden geëxtraheerd, zodat pings erin nog steeds activeren.
- Systeemprompt voor groepen: de eerste beurt van een groepssessie (en elke beurt nadat `/activation` de modus wijzigt) injecteert activeringsinstructies in de systeemprompt (`Activation: trigger-only ...` of `Activation: always-on ...`, plus "richt je tot de specifieke afzender"). Permanente bezorgingsinstructies voor groepschats ("Je bevindt je in een WhatsApp-groepschat...") worden altijd opgenomen.

## Configuratievoorbeeld (WhatsApp)

Zorg dat pings met een weergavenaam werken, zelfs wanneer WhatsApp de zichtbare `@` uit de berichttekst verwijdert:

```json5
{
  channels: {
    whatsapp: {
      groups: {
        "*": { requireMention: true },
      },
      historyLimit: 50, // venster voor wachtende groepscontext (standaard 50)
    },
  },
  agents: {
    entries: {
      main: {
        groupChat: {
          mentionPatterns: ["@?openclaw", "\\+?15555550123"],
        },
      },
    },
  },
}
```

Opmerkingen:

- De reguliere expressies zijn hoofdletterongevoelig en gebruiken dezelfde beveiligingen voor veilige regex als andere configuratieoppervlakken voor regex; ongeldige patronen en onveilige geneste herhalingen worden genegeerd.
- WhatsApp verzendt nog steeds canonieke vermeldingen via `mentionedJids` wanneer iemand op het contact tikt, dus de terugvaloptie met het nummer is zelden nodig, maar vormt een nuttig vangnet.
- Het venster voor wachtende context wordt bepaald als `channels.whatsapp.accounts.<id>.historyLimit` → `channels.whatsapp.historyLimit` → `messages.groupChat.historyLimit` → 50.

### Activeringsopdracht (alleen eigenaar)

Gebruik de opdracht voor groepschats:

- `/activation mention`
- `/activation always`

Alleen nummers van eigenaren (uit `channels.whatsapp.allowFrom`, of het eigen E.164-nummer van de bot wanneer dit niet is ingesteld) kunnen dit wijzigen; `/activation` van iemand anders wordt genegeerd en alleen als context opgeslagen. Verzend `/status` als zelfstandig bericht in de groep om de huidige activeringsmodus te bekijken.

## Gebruik

1. Voeg je WhatsApp-account (het account waarop OpenClaw draait) toe aan de groep.
2. Zeg `@openclaw ...` (of neem het nummer op). Alleen afzenders op de toelatingslijst kunnen de agent activeren, tenzij je `groupPolicy: "open"` instelt.
3. De agentprompt bevat de wachtende groepscontext plus regels met afzenderlabels, zodat de agent zich tot de juiste persoon kan richten.
4. Sessie-instructies (`/verbose on`, `/trace on`, `/think high`, `/new` of `/reset`, `/compact`) gelden alleen voor de sessie van die groep; verzend ze als zelfstandige berichten zodat ze worden geregistreerd. Je persoonlijke DM-sessie blijft onafhankelijk.

## Testen/verificatie

- Handmatige snelle test:
  - Verzend een `@openclaw`-ping in de groep en controleer of het antwoord naar de naam van de afzender verwijst.
  - Verzend een tweede ping en controleer of het geschiedenisblok is opgenomen en vervolgens vóór de volgende beurt wordt gewist.
- Controleer de Gateway-logboeken (uitvoeren met `--verbose`) op `inbound web message`-vermeldingen die `from: <groupJid>` en de berichttekst met afzenderlabel tonen.

## Bekende aandachtspunten

- Heartbeats worden uitgevoerd in de hoofdsessie van de agent; groepssessies krijgen nooit Heartbeat-uitvoeringen.
- Echo-onderdrukking onthoudt de gecombineerde prompt (geschiedenis + huidig bericht) per sessie, zodat de eigen bezorgde berichten van de bot deze niet opnieuw activeren; een identieke herhaalde batch kan als echo worden overgeslagen.
- Vermeldingen in het sessiearchief verschijnen als `agent:<agentId>:whatsapp:group:<jid>` in het SQLite-sessiearchief per agent; een ontbrekende vermelding betekent alleen dat de groep nog geen uitvoering heeft geactiveerd.
- Typindicatoren volgen `agents.entries.*.typingMode` / `agents.defaults.typingMode`. Wanneer zichtbare antwoorden zijn ingesteld op een modus die uitsluitend het berichtenhulpmiddel gebruikt, begint het typen standaard onmiddellijk, zodat groepsleden kunnen zien dat de agent bezig is, zelfs als er geen automatisch definitief antwoord wordt geplaatst. Een expliciete configuratie van de typmodus heeft nog steeds voorrang.

## Gerelateerd

- [Groepen](/nl/channels/groups)
- [Kanaalroutering](/nl/channels/channel-routing)
- [Uitzendgroepen](/nl/channels/broadcast-groups)
