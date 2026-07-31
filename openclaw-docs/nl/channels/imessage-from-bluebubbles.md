---
read_when:
    - Een overstap van BlueBubbles naar de meegeleverde iMessage-plugin plannen
    - BlueBubbles-configuratiesleutels vertalen naar iMessage-equivalenten
    - imsg verifiëren voordat je de iMessage-plugin inschakelt
summary: 'Migreer oude BlueBubbles-configuraties naar de gebundelde iMessage-plugin: sleuteltoewijzing, groepsallowlist-controles en verificatie van de omschakeling.'
title: Afkomstig van BlueBubbles
x-i18n:
    generated_at: "2026-07-27T05:01:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5984ad1319b4bb3060496666bea6de663eba0105a89f82d13030c015c5df159d
    source_path: channels/imessage-from-bluebubbles.md
    workflow: 16
---

BlueBubbles-ondersteuning is verwijderd. OpenClaw ondersteunt iMessage alleen via de meegeleverde `imessage`-Plugin, die [`steipete/imsg`](https://github.com/steipete/imsg) via JSON-RPC aanstuurt en toegang biedt tot hetzelfde private API-oppervlak als BlueBubbles (`react`, `edit`, `unsend`, `reply`, `sendWithEffect`, native peilingen, groepsbeheer, bijlagen). Eén CLI-binair bestand vervangt de BlueBubbles-server + client-app + webhook-infrastructuur: geen REST-eindpunt, geen webhookauthenticatie.

Deze handleiding migreert oude `channels.bluebubbles`-configuraties naar `channels.imessage`. Er is geen ander ondersteund migratiepad. In de huidige OpenClaw is een achtergebleven `channels.bluebubbles`-blok inert — geen enkele runtime leest het.

<Note>
Zie [Verwijdering van BlueBubbles en het imsg-pad voor iMessage](/nl/announcements/bluebubbles-imessage) voor de korte aankondiging en samenvatting voor beheerders.
</Note>

## Migratiechecklist

Dit is het kortste veilige pad als je jouw oude BlueBubbles-configuratie al kent:

1. Verifieer `imsg` rechtstreeks op de Mac waarop Messages.app draait (`imsg chats`, `imsg history`, `imsg send`, `imsg rpc --help`).
2. Kopieer gedragssleutels van `channels.bluebubbles` naar `channels.imessage`: `dmPolicy`, `allowFrom`, `groupPolicy`, `groupAllowFrom`, `groups`, `includeAttachments`, `attachmentRoots`, `mediaMaxMb`, `textChunkLimit` en `actions`.
3. Verwijder transportsleutels die niet meer bestaan: `serverUrl`, `password`, webhook-URL's en de BlueBubbles-serverconfiguratie.
4. Als de Gateway niet op de Messages-Mac draait, stel je `channels.imessage.cliPath` in op een SSH-wrapper en stel je `remoteHost` in voor het op afstand ophalen van bijlagen.
5. Schakel `channels.imessage` in, start de Gateway opnieuw en voer vervolgens `openclaw channels status --probe --channel imessage` uit.
6. Test één privébericht, één toegestane groep, bijlagen als die zijn ingeschakeld en elke private API-actie die de agent naar verwachting zal gebruiken.
7. Verwijder de BlueBubbles-server en de oude `channels.bluebubbles`-configuratie nadat het iMessage-pad is geverifieerd.

## Wat imsg doet

`imsg` is een lokale macOS-CLI voor Messages. OpenClaw start `imsg rpc` als onderliggend proces en communiceert via JSON-RPC over stdin/stdout. Er is geen HTTP-server, webhook-URL, achtergronddemon, launch agent of poort die moet worden opengesteld.

- Leesbewerkingen komen uit `~/Library/Messages/chat.db` via een alleen-lezen SQLite-handle.
- Live binnenkomende berichten komen uit `imsg watch` / `watch.subscribe`, dat `chat.db`-bestandssysteemgebeurtenissen volgt met polling als terugvaloptie.
- Voor normale tekst- en bestandsverzending wordt automatisering van Messages.app gebruikt.
- Geavanceerde acties gebruiken `imsg launch` om de `imsg`-helper in Messages.app te injecteren. Dit maakt leesbevestigingen, typindicatoren, uitgebreide verzendingen, bewerken, verzenden ongedaan maken, antwoorden in threads, tapbacks, peilingen en groepsbeheer mogelijk.
- Linux-builds kunnen een gekopieerde `chat.db` inspecteren, maar kunnen niet verzenden, de live Mac-database bewaken of Messages.app aansturen. Voor OpenClaw iMessage voer je `imsg` uit op de aangemelde Mac of via een SSH-wrapper naar die Mac.

## Voordat je begint

1. Installeer `imsg` op de Mac waarop Messages.app draait:

   ```bash
   brew install steipete/tap/imsg
   brew update && brew upgrade imsg
   imsg --version
   imsg chats --limit 3
   ```

   Voor de gebruikelijke lokale configuratie kan de OpenClaw-configuratie na bevestiging door de gebruiker aanbieden om `imsg` via Homebrew te installeren of bij te werken op de aangemelde Messages-Mac. Handmatige configuraties en topologieën met een SSH-wrapper blijven onder beheer van de operator: herhaal de Homebrew-update in dezelfde lokale of externe gebruikerscontext waarin `imsg` wordt uitgevoerd. Als `imsg chats` mislukt met `unable to open database file`, lege uitvoer of `authorization denied`, verleen je volledige schijftoegang aan de terminal, editor, het Node-proces, de Gateway-service of het bovenliggende SSH-proces dat `imsg` start. Open dat bovenliggende proces daarna opnieuw.

2. Verifieer de oppervlakken voor lezen, bewaken, verzenden en RPC voordat je de OpenClaw-configuratie wijzigt:

   ```bash
   imsg chats --limit 10 --json | jq -s
   imsg history --chat-id 42 --limit 10 --attachments --json | jq -s
   imsg watch --chat-id 42 --reactions --json
   imsg send --chat-id 42 --text "OpenClaw imsg test"
   imsg rpc --help
   ```

   Vervang `42` door een echte chat-id uit `imsg chats`. Voor verzenden is automatiseringsmachtiging voor Messages.app vereist. Als OpenClaw via SSH wordt uitgevoerd, voer je deze opdrachten uit via dezelfde SSH-wrapper of gebruikerscontext die OpenClaw zal gebruiken. Als lezen werkt maar verzenden mislukt met AppleEvents `-1743`, controleer je of de automatiseringsmachtiging aan `/usr/libexec/sshd-keygen-wrapper` is verleend; zie [Verzenden via een SSH-wrapper mislukt met AppleEvents -1743](/nl/channels/imessage#requirements-and-permissions-macos).

3. Schakel de private API-bridge in. Dit wordt sterk aanbevolen voor OpenClaw iMessage, omdat antwoorden, tapbacks, effecten, peilingen, antwoorden op bijlagen en groepsacties ervan afhankelijk zijn:

   ```bash
   imsg launch
   imsg status --json
   ```

   Voor `imsg launch` moet SIP zijn uitgeschakeld (en op moderne macOS-versies moet bibliotheekvalidatie zijn versoepeld — zie [De private API van imsg inschakelen](/nl/channels/imessage#enabling-the-imsg-private-api)). Basisfuncties voor verzenden, geschiedenis en bewaken werken zonder `imsg launch`; het volledige iMessage-actieoppervlak van OpenClaw niet.

4. Nadat je `channels.imessage` hebt ingeschakeld en de Gateway hebt gestart, verifieer je de bridge via OpenClaw:

   ```bash
   openclaw channels status --probe
   ```

   Het iMessage-account moet `works` rapporteren; met `--json` bevat de testpayload `privateApi.available: true`. Als het `false` rapporteert, los je dat eerst op — zie [Detectie van mogelijkheden](/nl/channels/imessage#private-api-actions). Voor testen is een bereikbare Gateway nodig (anders valt de CLI terug op uitvoer die alleen op de configuratie is gebaseerd) en alleen geconfigureerde, ingeschakelde accounts worden getest.

5. Maak een momentopname van je configuratie:

   ```bash
   cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak
   ```

## Configuratievertaling

iMessage en BlueBubbles delen de meeste gedragssleutels op kanaalniveau. Wat verandert, is het transport (REST-server tegenover lokale CLI) en de sleutelindeling van het groepsregister.

| BlueBubbles                                                | gebundelde iMessage                       | Opmerkingen                                                                                                                                                                                                                                                                       |
| ---------------------------------------------------------- | ----------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `channels.bluebubbles.enabled`                             | `channels.imessage.enabled`               | Dezelfde semantiek (standaard `true` zodra het blok bestaat).                                                                                                                                                                                                                     |
| `channels.bluebubbles.serverUrl`                           | _(verwijderd)_                            | Geen REST-server — de plugin start `imsg rpc` via stdio.                                                                                                                                                                                                                              |
| `channels.bluebubbles.password`                            | _(verwijderd)_                            | Geen Webhook-authenticatie nodig.                                                                                                                                                                                                                                                   |
| _(impliciet)_                                              | `channels.imessage.cliPath`               | Pad naar `imsg` (standaard `imsg`); gebruik een wrapperscript voor SSH.                                                                                                                                                                                                     |
| _(impliciet)_                                              | `channels.imessage.dbPath`                | Optionele overschrijving van `chat.db` voor Messages.app; automatisch gedetecteerd wanneer weggelaten.                                                                                                                                                                           |
| _(impliciet)_                                              | `channels.imessage.remoteHost`            | `host` of `user@host` — alleen nodig wanneer `cliPath` een SSH-wrapper is en je bijlagen via SCP wilt ophalen.                                                                                                                                                         |
| `channels.bluebubbles.dmPolicy`                            | `channels.imessage.dmPolicy`              | Dezelfde waarden (`pairing` / `allowlist` / `open` / `disabled`); standaard `pairing`.                                                                                                                                                                     |
| `channels.bluebubbles.allowFrom`                           | `channels.imessage.allowFrom`             | Dezelfde indelingen voor handles (`+15555550123`, `user@example.com`). Goedkeuringen uit de koppelingsopslag worden niet overgedragen — zie hieronder.                                                                                                                               |
| `channels.bluebubbles.groupPolicy`                         | `channels.imessage.groupPolicy`           | Dezelfde waarden (`allowlist` / `open` / `disabled`); standaard `allowlist`.                                                                                                                                                                               |
| `channels.bluebubbles.groupAllowFrom`                      | `channels.imessage.groupAllowFrom`        | Hetzelfde. Wanneer niet ingesteld, valt iMessage terug op `allowFrom`; een expliciet lege `groupAllowFrom: []` blokkeert alle groepen onder `groupPolicy: "allowlist"`.                                                                                                                   |
| `channels.bluebubbles.groups`                              | `channels.imessage.groups`                | Kopieer de jokertekenvermelding `"*"` letterlijk; geef vermeldingen per groep nieuwe sleutels op basis van de numerieke iMessage-`chat_id` — zie 'Valkuil bij het groepsregister'. `requireMention`, `tools`, `toolsBySender`, `systemPrompt` worden overgenomen. |
| `channels.bluebubbles.sendReadReceipts`                    | `channels.imessage.sendReadReceipts`      | Standaard `true`. Met de gebundelde plugin wordt dit alleen geactiveerd wanneer de controle van de privé-API actief is.                                                                                                                                                            |
| `channels.bluebubbles.includeAttachments`                  | `channels.imessage.includeAttachments`    | Dezelfde vorm, eveneens standaard uitgeschakeld. Als bijlagen via BlueBubbles binnenkwamen, stel dit dan expliciet in — inkomende foto's/media worden stilzwijgend verwijderd (geen `Inbound message`-logregel) totdat je dat doet.                                                      |
| `channels.bluebubbles.attachmentRoots`                     | `channels.imessage.attachmentRoots`       | Lokale hoofdmappen; dezelfde jokertekenregels.                                                                                                                                                                                                                                      |
| _(n.v.t.)_                                                  | `channels.imessage.remoteAttachmentRoots` | Alleen gebruikt wanneer `remoteHost` is ingesteld voor ophalen via SCP.                                                                                                                                                                                                            |
| `channels.bluebubbles.mediaMaxMb`                          | `channels.imessage.mediaMaxMb`            | Standaard 16 MB op iMessage (de standaard van BlueBubbles was 8 MB). Stel dit expliciet in om de lagere limiet te behouden.                                                                                                                                                              |
| `channels.bluebubbles.textChunkLimit`                      | `channels.imessage.textChunkLimit`        | Standaard 4000 op beide.                                                                                                                                                                                                                                                          |
| `channels.bluebubbles.coalesceSameSenderDms`               | _(verwijderd)_                            | Migreer deze sleutel niet. `imsg` 0.13.1 en nieuwer voegt gesplitste verzendingen van Apple-URL-voorbeelden samen voordat OpenClaw ze ontvangt; `openclaw doctor --fix` verwijdert een verouderde iMessage-sleutel.                                                               |
| `channels.bluebubbles.enrichGroupParticipantsFromContacts` | _(n.v.t.)_                                | `imsg` toont al de weergavenamen van afzenders uit `chat.db`.                                                                                                                                                                                                            |
| `channels.bluebubbles.actions.*`                           | `channels.imessage.actions.*`             | Dezelfde schakelaars per actie (`reactions`, `edit`, `unsend`, `reply`, `sendWithEffect`, `renameGroup`, `setGroupIcon`, `addParticipant`, `removeParticipant`, `leaveGroup`, `sendAttachment`) plus de nieuwe `polls`. Ze zijn allemaal standaard ingeschakeld; acties via de privé-API vereisen nog steeds de bridge. |

Configuraties met meerdere accounts (`channels.bluebubbles.accounts.*`) worden één-op-één omgezet naar `channels.imessage.accounts.*`.

## Valkuil bij het groepsregister

De gebundelde iMessage-plugin voert twee groepscontroles direct na elkaar uit. Een groepsbericht moet beide doorstaan om de agent te bereiken:

1. **Toegestane afzenders/chatdoelen** (`channels.imessage.groupAllowFrom`) — komt overeen met de afzenderhandle of het chatdoel (vermeldingen `chat_id:`, `chat_guid:`, `chat_identifier:`). Wanneer `groupAllowFrom` niet is ingesteld, valt deze controle terug op `allowFrom`; een expliciete `groupAllowFrom: []` schakelt die terugval uit en verwijdert elk groepsbericht onder `groupPolicy: "allowlist"`.
2. **Groepsregister** (`channels.imessage.groups`) — gebruikt de numerieke iMessage-`chat_id` als sleutel:
   - Geen `groups`-blok (of een leeg blok): groepen doorstaan deze controle zolang controle 1 een effectieve, niet-lege lijst met toegestane afzenders heeft; afzenderfiltering bepaalt de toegang en er verschijnt bij het opstarten geen waarschuwing dat alles wordt verwijderd.
   - `groups` met vermeldingen maar zonder `"*"`: alleen de vermelde `chat_id`-sleutels worden doorgelaten. Zodra een groep wordt vermeld, verandert het register in een lijst met toegestane groepen, zelfs onder `groupPolicy: "open"`.
   - `groups: { "*": { ... } }`: elke groep doorstaat deze controle.

De migratievalkuil: BlueBubbles gebruikte de chat-GUID/chat-ID als sleutel voor `groups`-vermeldingen, terwijl het iMessage-register de numerieke `chat_id` als sleutel gebruikt. Letterlijk gekopieerde vermeldingen per groep maken een niet-leeg register waarvan de sleutels nooit overeenkomen, waardoor elk groepsbericht bij controle 2 wordt verwijderd. Kopieer het jokerteken `"*"` letterlijk; geef specifieke groepsvermeldingen nieuwe sleutels met `chat_id`-waarden uit `imsg chats`.

Beide verwijderingspaden zijn op het standaardlogniveau zichtbaar via `warn`-regels:

- Eenmaal per account bij het opstarten, wanneer `groupPolicy: "allowlist"` is ingesteld en de effectieve lijst met toegestane groepsafzenders leeg is: `imessage: groupPolicy="allowlist" for account "<id>" but no group sender allowlist is configured ...`. Stel `groupAllowFrom` (of `allowFrom`) in om afzenders toe te laten; alleen `groups` toevoegen voldoet niet aan de afzendercontrole.
- Eenmaal per `chat_id` tijdens runtime, wanneer het register een groep verwijdert: `imessage: dropping group message from chat_id=<id> ... not in channels.imessage.groups allowlist`, met de exacte sleutel die moet worden toegevoegd.

Privéberichten blijven in beide gevallen werken — ze volgen een ander codepad, dus succesvolle privéberichten bewijzen niet dat groepsroutering werkt.

De minimale afzendergebonden configuratie met `groupPolicy: "allowlist"`:

```json5
{
  channels: {
    imessage: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15555550123", "chat_guid:any;-;..."],
    },
  },
}
```

Hiermee worden de geconfigureerde afzenders in elke groep toegelaten. Voeg `groups`-vermeldingen toe om toegestane chats te beperken of opties per chat in te stellen, zoals `requireMention`; kopieer de BlueBubbles-vermelding `"*"` letterlijk, maar geef specifieke vermeldingen nieuwe sleutels met numerieke iMessage-`chat_id`-waarden.

## Stap voor stap

1. Zet de configuratie om. Houd het nieuwe blok uitgeschakeld tijdens het bewerken; het oude `channels.bluebubbles`-blok wordt door de huidige OpenClaw genegeerd en kan ernaast blijven staan als referentie:

   ```json5
   {
     channels: {
       imessage: {
         enabled: false, // zet op true wanneer je klaar bent om over te schakelen
         cliPath: "/opt/homebrew/bin/imsg",
         dmPolicy: "pairing",
         allowFrom: ["+15555550123"], // kopieer uit bluebubbles.allowFrom
         groupPolicy: "allowlist",
         groupAllowFrom: [], // kopieer uit bluebubbles.groupAllowFrom
         groups: { "*": { requireMention: true } }, // jokerteken wordt letterlijk gekopieerd; geef vermeldingen per chat nieuwe sleutels op basis van chat_id
         // acties zijn standaard ingeschakeld; stel afzonderlijke schakelaars in op false om ze uit te schakelen
       },
     },
   }
   ```

2. **Schakel over en voer een controle uit.** Stel `channels.imessage.enabled: true` in, start de Gateway opnieuw en controleer of het kanaal een gezonde status meldt:

   ```bash
   openclaw gateway restart
   openclaw channels status --probe --channel imessage   # verwacht "works"; --json toont privateApi.available: true
   ```

   De probe vereist een bereikbare Gateway en controleert alleen geconfigureerde, ingeschakelde accounts. Gebruik de directe `imsg`-opdrachten in [Voordat je begint](#before-you-start) om de Mac zelf te valideren.

3. **Controleer DM's.** Stuur de agent een direct bericht en bevestig dat het antwoord aankomt.

4. **Controleer groepen afzonderlijk.** DM's en groepen gebruiken verschillende codepaden — een geslaagde DM bewijst niet dat groepen correct worden gerouteerd. Stuur een bericht in een toegestane groepschat en bevestig dat het antwoord aankomt. Als de groep stil blijft (geen antwoord van de agent, geen fout), controleer dan het Gateway-logboek op de twee `warn`-regels uit 'Group registry footgun' hierboven. De opstartwaarschuwing betekent dat de effectieve afzenderstoestaanlijst leeg is; een waarschuwing per `chat_id` betekent dat een gevulde `groups`-registratie die chat niet bevat.

5. **Controleer de beschikbare acties.** Vraag de agent vanuit een gekoppelde DM om een reactie toe te voegen, een bericht te bewerken, het verzenden ongedaan te maken, te antwoorden, een foto te sturen en (in een groep) de groep te hernoemen of een deelnemer toe te voegen of te verwijderen. Elke actie moet als systeemeigen actie in Messages.app aankomen. Als een actie `iMessage <action> requires the imsg private API bridge` oplevert, voer je `imsg launch` opnieuw uit en vernieuw je met `openclaw channels status --probe`.

6. **Verwijder de BlueBubbles-server en het `channels.bluebubbles`-blok** zodra iMessage-DM's, groepen en acties zijn gecontroleerd. OpenClaw leest `channels.bluebubbles` niet.

## Actiepariteit in één oogopslag

| Actie                                               | verouderde BlueBubbles | meegeleverde iMessage                                                         |
| --------------------------------------------------- | ---------------------- | ----------------------------------------------------------------------------- |
| Tekst verzenden / terugvallen op sms                | ✅                     | ✅                                                                            |
| Media verzenden (foto, video, bestand, spraak)      | ✅                     | ✅                                                                            |
| Antwoord in thread (`reply_to_guid`)             | ✅                     | ✅ (sluit [#51892](https://github.com/openclaw/openclaw/issues/51892))        |
| Tapback (`react`)                        | ✅                     | ✅                                                                            |
| Bewerken / verzenden ongedaan maken (ontvangers met macOS 13+) | ✅          | ✅                                                                            |
| Verzenden met schermeffect                          | ✅                     | ✅ (sluit een deel van [#9394](https://github.com/openclaw/openclaw/issues/9394)) |
| Tekst vet / cursief / onderstreept / doorgestreept opmaken | ✅              | ✅ (opmaak met getypeerde runs via attributedBody)                            |
| Systeemeigen peilingen in Messages (maken en stemmen) | ❌                   | ✅ (`actions.polls`; ontvangers hebben iOS/macOS 26+ nodig voor systeemeigen weergave) |
| Groep hernoemen / groepspictogram instellen         | ✅                     | ✅                                                                            |
| Deelnemer toevoegen / verwijderen, groep verlaten   | ✅                     | ✅                                                                            |
| Leesbevestigingen en typindicator                   | ✅                     | ✅ (afhankelijk van de probe van de private API)                              |
| Samenvoeging van opgesplitste Apple URL-voorvertoningen | ✅                 | ✅ (upstream afgehandeld door `imsg` 0.13.1 en nieuwer; geen OpenClaw-instelling) |
| Herstel van inkomende berichten na een herstart     | ✅                     | ✅ (automatisch: `since_rowid` opnieuw afspelen + deduplicatie op GUID; ruimer venster bij lokale installatie) |

iMessage herstelt berichten die zijn gemist terwijl de Gateway niet actief was: bij het opstarten worden ze vanaf de laatst verzonden rowid opnieuw afgespeeld via `imsg watch.subscribe` `since_rowid`, op GUID gededupliceerd en onderdrukt een leeftijdsgrens voor verouderde achterstand de 'backlog bomb' van de Push-flush. Dit verloopt via de `imsg`-RPC-verbinding, waardoor het ook werkt voor externe SSH-configuraties met `cliPath`; lokale configuraties krijgen een ruimer herstelvenster omdat ze `chat.db` kunnen lezen. Zie [Herstel van inkomende berichten na een herstart van een bridge of Gateway](/nl/channels/imessage#inbound-recovery-after-a-bridge-or-gateway-restart).

## Koppeling, sessies en ACP-bindingen

- **Toestaanlijsten worden per handle overgenomen.** `channels.imessage.allowFrom` herkent dezelfde `+15555550123`- / `user@example.com`-tekenreeksen die BlueBubbles gebruikte — kopieer ze letterlijk.
- **Goedkeuringen uit de koppelingsopslag worden niet overgedragen.** De koppelingsopslag is per kanaal en niets migreert de oude BlueBubbles-opslag. Afzenders die uitsluitend via koppeling waren goedgekeurd, moeten nogmaals onder iMessage koppelen, of je voegt hun handles toe aan `allowFrom`.
- **Sessies** blijven beperkt tot één agent en chat. DM's worden met de standaardwaarde `session.dmScope=main` samengevoegd in de hoofdsessie van de agent; groepssessies blijven afzonderlijk per `chat_id` (`agent:<agentId>:imessage:group:<chat_id>`). Oude gespreksgeschiedenis onder BlueBubbles-sessiesleutels wordt niet overgenomen in iMessage-sessies.
- **ACP-bindingen** die naar `match.channel: "bluebubbles"` verwijzen, moeten worden gewijzigd in `"imessage"`. De `match.peer.id`-vormen (`chat_id:`, `chat_guid:`, `chat_identifier:`, alleen de handle) zijn identiek.

## Geen kanaal om terug te draaien

Er is geen ondersteunde BlueBubbles-runtime om naar terug te schakelen. Als de iMessage-controle mislukt, stel je `channels.imessage.enabled: false` in, herstart je de Gateway, verhelp je het `imsg`-blokkerende probleem en probeer je de migratie opnieuw.

De antwoordcache bevindt zich in de SQLite-status van de Plugin. `openclaw doctor --fix` importeert en archiveert het oude `imessage/reply-cache.jsonl`-zijbestand wanneer dit aanwezig is.

## Gerelateerd

- [Verwijdering van BlueBubbles en het imsg-pad voor iMessage](/nl/announcements/bluebubbles-imessage) — korte aankondiging en samenvatting voor beheerders.
- [iMessage](/nl/channels/imessage) — volledige naslag voor het iMessage-kanaal, inclusief configuratie van `imsg launch` en detectie van mogelijkheden.
- `/channels/bluebubbles` — verouderde URL die doorverwijst naar deze migratiehandleiding.
- [Koppeling](/nl/channels/pairing) — DM-authenticatie en koppelingsproces.
- [Kanaalroutering](/nl/channels/channel-routing) — hoe de Gateway een kanaal kiest voor uitgaande antwoorden.
