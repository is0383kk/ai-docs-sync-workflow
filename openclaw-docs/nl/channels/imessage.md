---
read_when:
    - iMessage-ondersteuning instellen
    - Problemen met het verzenden/ontvangen van iMessage oplossen
summary: Native iMessage-ondersteuning via imsg (JSON-RPC via stdio), met private API-acties voor antwoorden, tapbacks, effecten, peilingen, bijlagen en groepsbeheer. Aanbevolen voor nieuwe OpenClaw iMessage-configuraties wanneer aan de hostvereisten wordt voldaan.
title: iMessage
x-i18n:
    generated_at: "2026-07-27T05:25:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f3e8b1a65c76b25d03615c06a976f86a8af555cd96d5bfdb10cef9c955893ddc
    source_path: channels/imessage.md
    workflow: 16
---

<Note>
Voor de gebruikelijke OpenClaw iMessage-implementatie voer je de Gateway en `imsg` uit op dezelfde macOS-host waarop je bij Berichten bent ingelogd. Als je Gateway elders wordt uitgevoerd, laat je `channels.imessage.cliPath` verwijzen naar een transparante SSH-wrapper die `imsg` op de Mac uitvoert.

**Inkomend herstel verloopt automatisch.** Na een herstart van een bridge of gateway speelt iMessage de berichten opnieuw af die tijdens de uitval zijn gemist en onderdrukt het de verouderde ‘backlogbom’ die Apple na een Push-herstel kan wegschrijven, waarbij deduplicatie voorkomt dat iets tweemaal wordt verzonden. Er is geen configuratie nodig om dit in te schakelen — zie [Inkomend herstel na een herstart van een bridge of gateway](#inbound-recovery-after-a-bridge-or-gateway-restart).
</Note>

<Warning>
Ondersteuning voor BlueBubbles is verwijderd. Migreer `channels.bluebubbles`-configuraties naar `channels.imessage`; OpenClaw ondersteunt iMessage uitsluitend via `imsg`. Begin met [Verwijdering van BlueBubbles en het imsg-pad voor iMessage](/nl/announcements/bluebubbles-imessage) voor de korte aankondiging, of [Overstappen vanuit BlueBubbles](/nl/channels/imessage-from-bluebubbles) voor de volledige migratietabel.
</Warning>

Status: systeemeigen externe CLI-integratie. De Gateway start `imsg rpc` en communiceert via JSON-RPC over stdio — zonder afzonderlijke daemon of poort. De Private API-modus wordt sterk aanbevolen voor een volledig iMessage-kanaal; antwoorden, tapbacks, effecten, peilingen, antwoorden op bijlagen en groepsacties vereisen `imsg launch` en een geslaagde Private API-controle.

Voor de gebruikelijke lokale configuratie kan de OpenClaw-installatie, na bevestiging door de gebruiker, aanbieden om `imsg` via Homebrew te installeren of bij te werken op de Mac waarop je bij Berichten bent ingelogd. Handmatige configuraties en topologieën met SSH-wrappers blijven onder beheer van de operator: installeer of werk `imsg` bij binnen dezelfde gebruikerscontext waarin de Gateway of wrapper wordt uitgevoerd.

<CardGroup cols={3}>
  <Card title="Private API-acties" icon="wand-sparkles" href="#private-api-actions">
    Antwoorden, tapbacks, effecten, peilingen, bijlagen en groepsbeheer.
  </Card>
  <Card title="Koppelen" icon="link" href="/nl/channels/pairing">
    Privéberichten via iMessage gebruiken standaard de koppelingsmodus.
  </Card>
  <Card title="Externe Mac" icon="terminal" href="#remote-mac-over-ssh">
    Gebruik een SSH-wrapper wanneer de Gateway niet op de Mac met Berichten wordt uitgevoerd.
  </Card>
  <Card title="Configuratiereferentie" icon="settings" href="/nl/gateway/config-channels#imessage">
    Volledige referentie voor iMessage-velden.
  </Card>
</CardGroup>

## Snelle configuratie

<Tabs>
  <Tab title="Lokale Mac (snelste methode)">
    <Steps>
      <Step title="imsg installeren en verifiëren">

```bash
brew install steipete/tap/imsg
brew update && brew upgrade imsg
imsg rpc --help
imsg launch
openclaw channels status --probe
```

        Wanneer de lokale configuratiewizard een ontbrekende standaardopdracht voor `imsg` detecteert, kan deze vragen om `steipete/tap/imsg` via Homebrew te installeren. Als een door Homebrew beheerde `imsg` wordt gedetecteerd, kan de wizard vragen om deze opnieuw te installeren of bij te werken. Aangepaste wrappers voor `cliPath` worden niet gewijzigd.

      </Step>

      <Step title="OpenClaw configureren">

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "/usr/local/bin/imsg",
      dbPath: "/Users/user/Library/Messages/chat.db",
    },
  },
}
```

      </Step>

      <Step title="Gateway starten">

```bash
openclaw gateway
```

      </Step>

      <Step title="Eerste koppeling voor privéberichten goedkeuren (standaard dmPolicy)">

```bash
openclaw pairing list imessage
openclaw pairing approve imessage <CODE>
```

        Koppelingsverzoeken verlopen na 1 uur.
      </Step>
    </Steps>

  </Tab>

  <Tab title="Externe Mac via SSH">
    Voor de meeste configuraties is SSH niet nodig. Gebruik deze topologie alleen wanneer de Gateway niet kan worden uitgevoerd op de Mac waarop je bij Berichten bent ingelogd. OpenClaw vereist alleen een stdio-compatibele `cliPath`, zodat je `cliPath` kunt laten verwijzen naar een wrapperscript dat via SSH verbinding maakt met een externe Mac en daar `imsg` uitvoert.
    Installeer en werk `imsg` bij op die externe Mac, niet op de Gateway-host:

```bash
ssh messages-mac 'brew install steipete/tap/imsg && brew update && brew upgrade imsg'
```

```bash
#!/usr/bin/env bash
exec ssh -T messages-mac imsg "$@"
```

    Aanbevolen configuratie wanneer bijlagen zijn ingeschakeld:

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "~/.openclaw/scripts/imsg-ssh",
      remoteHost: "user@gateway-host", // gebruikt om bijlagen via SCP op te halen
      includeAttachments: true,
      // Optioneel: extra toegestane hoofdlocaties voor bijlagen (samengevoegd met de standaardlocatie
      // /Users/*/Library/Messages/Attachments).
      attachmentRoots: ["/Users/*/Library/Messages/Attachments"],
      remoteAttachmentRoots: ["/Users/*/Library/Messages/Attachments"],
    },
  },
}
```

    Als `remoteHost` niet is ingesteld, probeert OpenClaw deze automatisch te detecteren door het SSH-wrapperscript te parseren.
    `remoteHost` moet `host` of `user@host` zijn (geen spaties of SSH-opties); onveilige waarden worden genegeerd.
    OpenClaw gebruikt strikte controle van hostsleutels voor SCP, dus de hostsleutel van de relayhost moet al in `~/.ssh/known_hosts` staan.
    Paden naar bijlagen worden gevalideerd aan de hand van toegestane hoofdlocaties (`attachmentRoots` / `remoteAttachmentRoots`).

<Warning>
Elke wrapper voor `cliPath` of SSH-proxy die je vóór `imsg` plaatst, MOET zich voor langdurige JSON-RPC gedragen als een transparante stdio-pijp. OpenClaw wisselt gedurende de levensduur van het kanaal kleine, door nieuwe regels begrensde JSON-RPC-berichten uit via stdin/stdout van de wrapper:

- Stuur elk stdin-fragment/elke stdin-regel door **zodra bytes beschikbaar zijn** — wacht niet op EOF.
- Stuur elk stdout-fragment/elke stdout-regel onmiddellijk in omgekeerde richting door.
- Behoud nieuwe regels.
- Vermijd blokkerende leesbewerkingen met een vaste grootte (`read(4096)`, `cat | buffer`, standaard-`read` van de shell) die kleine frames kunnen uithongeren.
- Houd stderr gescheiden van de JSON-RPC-stream op stdout.

Een wrapper die stdin buffert totdat een groot blok vol is, veroorzaakt symptomen die op een iMessage-storing lijken — `imsg rpc timeout (chats.list)` of herhaalde herstarts van het kanaal — hoewel `imsg rpc` zelf correct werkt. `ssh -T host imsg "$@"` (hierboven) is veilig omdat deze de `cliPath`-argumenten van OpenClaw doorstuurt, zoals `rpc` en `--db`. Pijplijnen zoals `ssh host imsg | grep -v '^DEBUG'` zijn dat NIET — zelfs regelgebufferde hulpmiddelen kunnen frames vasthouden; gebruik `stdbuf -oL -eL` voor elke fase als je toch moet filteren.
</Warning>

  </Tab>
</Tabs>

## Vereisten en machtigingen (macOS)

- Berichten moet zijn aangemeld op de Mac waarop `imsg` wordt uitgevoerd.
- Volledige schijftoegang is vereist voor de procescontext waarin OpenClaw/`imsg` wordt uitgevoerd (voor toegang tot de Berichten-database).
- Automatiseringsmachtiging is vereist om berichten via Messages.app te verzenden.
- Voor geavanceerde acties (reageren / bewerken / verzending ongedaan maken / antwoord in thread / effecten / peilingen / groepsbewerkingen) moet System Integrity Protection zijn uitgeschakeld — zie [De Private API van imsg inschakelen](#enabling-the-imsg-private-api). Het verzenden en ontvangen van gewone tekst en media werkt zonder deze wijziging.

<Tip>
Machtigingen worden per procescontext verleend. Als de gateway headless wordt uitgevoerd (LaunchAgent/SSH), voer je eenmalig een interactieve opdracht uit binnen diezelfde context om de prompts te activeren:

```bash
imsg chats --limit 1
# of
imsg send <handle> "test"
```

</Tip>

<Accordion title="Verzenden via SSH-wrapper mislukt met AppleEvents -1743">
  Een configuratie via externe SSH kan chats lezen, slagen voor `channels status --probe` en inkomende berichten verwerken, terwijl uitgaande verzendingen nog steeds mislukken met een AppleEvents-autorisatiefout:

```text
Niet gemachtigd om Apple-events naar Berichten te sturen. (-1743)
```

Controleer de TCC-database van de aangemelde Mac-gebruiker of System Settings > Privacy & Security > Automation. Als de automatiseringsvermelding is vastgelegd voor `/usr/libexec/sshd-keygen-wrapper` in plaats van het proces `imsg` of de lokale shell, biedt macOS mogelijk geen bruikbare schakelaar voor Berichten voor die SSH-client aan de serverzijde:

```text
kTCCServiceAppleEvents | /usr/libexec/sshd-keygen-wrapper | auth_value=0 | com.apple.MobileSMS
```

In deze toestand kunnen het herhalen van `tccutil reset AppleEvents` of het opnieuw uitvoeren van `imsg send` via dezelfde SSH-wrapper blijven mislukken, omdat de SSH-wrapper de procescontext is die automatisering van Berichten nodig heeft en niet een app waaraan de gebruikersinterface toestemming kan verlenen.

Gebruik in plaats daarvan een van de ondersteunde procescontexten voor `imsg`:

- Voer de Gateway, of ten minste de bridge voor `imsg`, uit in de lokale sessie van de gebruiker die bij Berichten is aangemeld.
- Start de Gateway met een LaunchAgent voor die gebruiker nadat je vanuit dezelfde sessie Volledige schijftoegang en Automatisering hebt verleend.
- Als je de SSH-topologie met twee gebruikers behoudt, controleer je voordat je het kanaal inschakelt of een echte uitgaande `imsg send` via de exacte wrapper slaagt. Als hiervoor geen automatiseringsmachtiging kan worden verleend, schakel je over op een configuratie met één gebruiker voor `imsg` in plaats van voor verzendingen op de SSH-wrapper te vertrouwen.

</Accordion>

## De Private API van imsg inschakelen

`imsg` wordt geleverd met twee bedrijfsmodi. Voor OpenClaw is de Private API-modus de aanbevolen configuratie, omdat deze het kanaal de systeemeigen iMessage-acties biedt die gebruikers verwachten. De basismodus blijft nuttig voor installaties met een laag risico, initiële verificatie of hosts waarop SIP niet kan worden uitgeschakeld.

- **Basismodus** (standaard, geen SIP-wijzigingen nodig): uitgaande tekst en media via `send`, bewaking/geschiedenis van inkomende berichten en de chatlijst. Dit krijg je direct met een nieuwe `brew install steipete/tap/imsg` en de standaard macOS-machtigingen hierboven.
- **Private API-modus**: `imsg` injecteert een hulp-dylib in `Messages.app` om interne `IMCore`-functies aan te roepen. Hiermee worden `react`, `edit`, `unsend`, `reply` (in threads), `sendWithEffect`, `poll` en `poll-vote` (systeemeigen peilingen van Berichten), `renameGroup`, `setGroupIcon`, `addParticipant`, `removeParticipant`, `leaveGroup`, plus typindicatoren en leesbevestigingen beschikbaar.

Het aanbevolen actieoppervlak op deze pagina vereist de Private API-modus. De README van `imsg` vermeldt deze vereiste expliciet:

> Geavanceerde functies zoals `read`, `typing`, `launch`, uitgebreide verzending via de bridge, berichtmutatie en chatbeheer zijn optioneel. Hiervoor moet SIP zijn uitgeschakeld en moet een hulp-dylib in `Messages.app` worden geïnjecteerd. `imsg launch` weigert te injecteren wanneer SIP is ingeschakeld.

De techniek voor het injecteren van de helper gebruikt de eigen dylib van `imsg` om de Private API's van Berichten te bereiken. Het iMessage-pad van OpenClaw bevat geen server van derden of BlueBubbles-runtime.

<Warning>
**Het uitschakelen van SIP is een reële beveiligingsafweging.** SIP is een van de belangrijkste macOS-beveiligingen tegen de uitvoering van gewijzigde systeemcode; als je deze systeembreed uitschakelt, ontstaan extra aanvalsoppervlakken en neveneffecten. In het bijzonder geldt dat **het uitschakelen van SIP op Macs met Apple Silicon ook de mogelijkheid uitschakelt om iOS-apps op je Mac te installeren en uit te voeren**.

Behandel dit als een bewuste operationele keuze, vooral op een primaire persoonlijke Mac. Gebruik voor OpenClaw iMessage van productiekwaliteit bij voorkeur een speciale Mac of macOS-botgebruiker waarvoor je de bridge met een gerust gevoel kunt inschakelen. Als je dreigingsmodel nergens toestaat dat SIP is uitgeschakeld, is het ingebouwde iMessage beperkt tot de basismodus — alleen tekst en media verzenden/ontvangen, zonder reacties / bewerken / verzending ongedaan maken / effecten / groepsbewerkingen.
</Warning>

### Configuratie

1. **Installeer (of upgrade) `imsg`** op de Mac waarop Messages.app wordt uitgevoerd:

   ```bash
   brew install steipete/tap/imsg
   brew update && brew upgrade imsg
   imsg --version
   imsg status --json
   ```

   De uitvoer van `imsg status --json` vermeldt `bridge_version`, `rpc_methods` en `selectors` per methode, zodat je vóór de start kunt zien wat de huidige build ondersteunt.

2. **Schakel System Integrity Protection en (op moderne macOS-versies) Library Validation uit.** Voor het injecteren van een niet van Apple afkomstige hulp-dylib in de door Apple ondertekende `Messages.app` moet SIP uitgeschakeld zijn **en** Library Validation versoepeld zijn. De SIP-stap in de herstelmodus verschilt per macOS-versie:
   - **macOS 10.13-10.15 (Sierra-Catalina):** schakel Library Validation uit via Terminal, start opnieuw op in de herstelmodus, voer `csrutil disable` uit en start opnieuw op.
   - **macOS 11+ (Big Sur en later), Intel:** herstelmodus (of Internet Recovery), `csrutil disable`, opnieuw opstarten.
   - **macOS 11+, Apple Silicon:** gebruik de opstartprocedure met de aan/uit-knop om de herstelmodus te openen; houd bij recente macOS-versies de toets **Left Shift** ingedrukt wanneer je op Continue klikt en voer vervolgens `csrutil disable` uit. Voor virtuele machines geldt een afzonderlijke procedure, dus maak eerst een VM-momentopname.

   **Op macOS 11 en later is alleen `csrutil disable` meestal niet voldoende.** Apple dwingt Library Validation nog steeds af voor `Messages.app` als platformbinair bestand, waardoor een ad-hoc ondertekende helper wordt geweigerd (`Library Validation failed: ... platform binary, but mapped file is not`), zelfs als SIP uitstaat. Schakel na SIP ook Library Validation uit en start opnieuw op:

   ```bash
   sudo defaults write /Library/Preferences/com.apple.security.libraryvalidation.plist DisableLibraryValidation -bool true
   ```

   **macOS 26 (Tahoe), geverifieerd op 26.5.1:** uitgeschakelde SIP **plus** de bovenstaande opdracht `DisableLibraryValidation` volstaat om de helper te injecteren in versies 26.0 tot en met 26.5.x. **Er zijn geen boot-args vereist.** De plist is de doorslaggevende factor en de meest voorkomende ontbrekende stap wanneer injectie op Tahoe mislukt:
   - **Met de plist:** `imsg launch` injecteert en `imsg status` meldt `advanced_features: true`.
   - **Zonder de plist (zelfs als SIP uitstaat):** `imsg launch` mislukt met `Failed to launch: Timeout waiting for Messages.app to initialize`. AMFI weigert de ad-hoc helper bij het laden, waardoor de bridge nooit gereed wordt en het starten een time-out bereikt. Die time-out is het symptoom dat de meeste mensen op Tahoe tegenkomen; de oplossing is de bovenstaande plist, niet iets ingrijpenders.

   Als de injectie van `imsg launch` of specifieke `selectors` na een macOS-upgrade false beginnen te retourneren, is deze beveiligingspoort doorgaans de oorzaak. Controleer de status van SIP en Library Validation voordat je aanneemt dat de SIP-stap zelf is mislukt. Als die instellingen correct zijn en de bridge nog steeds niet kan injecteren, verzamel dan `imsg status --json` plus de uitvoer van `imsg launch` en meld dit bij het project `imsg`, in plaats van aanvullende systeembrede beveiligingsmaatregelen te verzwakken.

3. **Injecteer de helper.** Met SIP uitgeschakeld en aangemeld bij Messages.app:

   ```bash
   imsg launch
   ```

   `imsg launch` weigert te injecteren wanneer SIP nog is ingeschakeld, dus dit dient ook als bevestiging dat stap 2 is uitgevoerd.

4. **Verifieer de bridge vanuit OpenClaw:**

   ```bash
   openclaw channels status --probe
   ```

   De iMessage-vermelding moet `works` melden en `imsg status --json | jq '{rpc_methods, selectors}'` moet de mogelijkheden tonen die je macOS-build beschikbaar stelt. Voor het maken van peilingen is `selectors.pollPayloadMessage` vereist; voor stemmen zijn zowel `selectors.pollVoteMessage` als de RPC-methode `poll.vote` vereist. De OpenClaw-plugin kondigt alleen acties aan die door de gecachte probe worden ondersteund, terwijl bij een lege cache optimistisch wordt uitgegaan van ondersteuning en bij de eerste verzending een probe wordt uitgevoerd.

Als `openclaw channels status --probe` het kanaal als `works` meldt, maar specifieke acties tijdens de verzending de fout "iMessage `<action>` requires the imsg private API bridge" geven, voer `imsg launch` dan opnieuw uit — de helper kan wegvallen (door het opnieuw starten van Messages.app, een OS-update enzovoort) en de gecachte status `available: true` blijft acties aankondigen totdat de volgende probe deze vernieuwt.

### Wanneer SIP ingeschakeld blijft

Als het uitschakelen van SIP niet aanvaardbaar is voor je dreigingsmodel:

- `imsg` valt terug op de basismodus — alleen tekst + media + ontvangen.
- De OpenClaw-plugin kondigt nog steeds het verzenden van tekst/media en het bewaken van inkomende berichten aan; `react`, `edit`, `unsend`, `reply`, `sendWithEffect` en groepsbewerkingen worden verborgen uit het actieoppervlak (volgens de mogelijkhedenpoort per methode).
- Je kunt een afzonderlijke Mac zonder Apple Silicon (of een speciale bot-Mac) met uitgeschakelde SIP gebruiken voor de iMessage-werklast, terwijl SIP op je primaire apparaten ingeschakeld blijft. Zie hieronder [Speciale macOS-gebruiker voor de bot (afzonderlijke iMessage-identiteit)](#deployment-patterns).

## Toegangsbeheer en routering

<Tabs>
  <Tab title="DM-beleid">
    `channels.imessage.dmPolicy` beheert directe berichten:

    - `pairing` (standaard)
    - `allowlist` (vereist ten minste één vermelding in `allowFrom`)
    - `open` (vereist dat `allowFrom` `"*"` bevat)
    - `disabled`

    Veld voor de toelatingslijst: `channels.imessage.allowFrom`.

    Vermeldingen in de toelatingslijst moeten afzenders identificeren: handles of statische toegangsgroepen voor afzenders (`accessGroup:<name>`). Gebruik `channels.imessage.groupAllowFrom` voor chatdoelen zoals `chat_id:*`, `chat_guid:*` of `chat_identifier:*`; gebruik `channels.imessage.groups` voor numerieke registersleutels van `chat_id`.

  </Tab>

  <Tab title="Groepsbeleid + vermeldingen">
    `channels.imessage.groupPolicy` beheert de verwerking van groepen:

    - `allowlist` (standaard)
    - `open`
    - `disabled`

    Toelatingslijst voor groepsafzenders: `channels.imessage.groupAllowFrom`.

    Vermeldingen in `groupAllowFrom` kunnen ook naar statische toegangsgroepen voor afzenders verwijzen (`accessGroup:<name>`).

    Terugval tijdens runtime: als `groupAllowFrom` niet is ingesteld, gebruiken controles van afzenders in iMessage-groepen `allowFrom`; stel `groupAllowFrom` in wanneer toelating voor DM's en groepen moet verschillen. Een expliciet lege `groupAllowFrom: []` valt niet terug — deze blokkeert alle groepsafzenders onder `allowlist`.
    Runtime-opmerking: als `channels.imessage` volledig ontbreekt, valt de runtime terug op `groupPolicy="allowlist"` en wordt een waarschuwing gelogd (zelfs als `channels.defaults.groupPolicy` is ingesteld).

    <Warning>
    Groepsroutering onder `groupPolicy: "allowlist"` doorloopt **twee** opeenvolgende poorten:

    1. **Toelatingslijst voor afzenders** (`channels.imessage.groupAllowFrom`) — handle, `accessGroup:<name>`, `chat_guid`, `chat_identifier` of `chat_id`. Een lege effectieve lijst (geen `groupAllowFrom` en geen terugval naar `allowFrom`) blokkeert elke groepsafzender.
    2. **Groepsregister** (`channels.imessage.groups`) — wordt afgedwongen zodra de map vermeldingen bevat: de chat moet overeenkomen met een expliciete vermelding per `chat_id` of met het jokerteken `groups: { "*": { ... } }`. Wanneer `groups` leeg is of ontbreekt, bepaalt alleen de toelatingslijst voor afzenders de toelating.

    Als er geen effectieve toelatingslijst voor groepsafzenders is geconfigureerd, wordt elk groepsbericht vóór de registerpoort verwijderd. Elke poort heeft op het standaardlogniveau een eigen signaal op `warn`-niveau en elk signaal noemt een andere oplossing:

    - eenmalig per account bij het opstarten, wanneer de effectieve toelatingslijst voor groepsafzenders leeg is: `imessage: groupPolicy="allowlist" for account "<id>" but no group sender allowlist is configured ...` — los dit op door `channels.imessage.groupAllowFrom` (of `allowFrom`) in te stellen; alleen vermeldingen aan `groups` toevoegen zorgt ervoor dat poort 1 nog steeds elke afzender blokkeert.
    - eenmalig per `chat_id` tijdens runtime, wanneer een afzender poort 1 is gepasseerd maar de chat ontbreekt in een gevuld register `groups`: `imessage: dropping group message from chat_id=<id> ...` — los dit op door die `chat_id` (of `"*"`) onder `channels.imessage.groups` toe te voegen.

    DM's worden niet beïnvloed — deze volgen een ander codepad.

    Aanbevolen configuratie voor groepsverkeer onder `groupPolicy: "allowlist"`:

    ```json5
    {
      channels: {
        imessage: {
          groupPolicy: "allowlist",
          groupAllowFrom: ["+15555550123"],
          groups: { "*": { "requireMention": true } },
        },
      },
    }
    ```

    Alleen `groupAllowFrom` laat deze afzenders in elke groep toe; voeg het blok `groups` toe om te beperken welke chats zijn toegestaan (en om opties per chat in te stellen, zoals `requireMention`).
    </Warning>

    Vermeldingspoort voor groepen:

    - iMessage heeft geen ingebouwde metagegevens voor vermeldingen
    - detectie van vermeldingen gebruikt regex-patronen (`agents.entries.*.groupChat.mentionPatterns`, met terugval naar `messages.groupChat.mentionPatterns`)
    - zonder geconfigureerde patronen kan de vermeldingspoort niet worden afgedwongen
    - besturingsopdrachten van geautoriseerde afzenders omzeilen de vermeldingspoort

    `systemPrompt` per groep:

    Elke vermelding onder `channels.imessage.groups.*` accepteert een optionele tekenreeks `systemPrompt`, die bij elke beurt waarin een bericht in die groep wordt verwerkt in de systeemprompt van de agent wordt geïnjecteerd. De resolutie komt overeen met `channels.whatsapp.groups`:

    1. **Groepsspecifieke systeemprompt** (`groups["<chat_id>"].systemPrompt`): wordt gebruikt wanneer de vermelding voor de specifieke groep in de map bestaat **en** de sleutel `systemPrompt` ervan is gedefinieerd. Als `systemPrompt` een lege tekenreeks is (`""`), wordt het jokerteken onderdrukt en wordt er geen systeemprompt op die groep toegepast.
    2. **Systeemprompt met groepsjokerteken** (`groups["*"].systemPrompt`): wordt gebruikt wanneer de vermelding voor de specifieke groep volledig ontbreekt in de map, of wanneer deze bestaat maar geen sleutel `systemPrompt` definieert.

    ```json5
    {
      channels: {
        imessage: {
          groupPolicy: "allowlist",
          groupAllowFrom: ["+15555550123"],
          groups: {
            "*": { systemPrompt: "Gebruik de Britse spelling." },
            "8421": {
              requireMention: true,
              systemPrompt: "Dit is de chat voor de wachtdienst. Houd antwoorden korter dan 3 zinnen.",
            },
            "9907": {
              // expliciete onderdrukking: het jokerteken "Gebruik de Britse spelling." is hier niet van toepassing
              systemPrompt: "",
            },
          },
        },
      },
    }
    ```

    Prompts per groep zijn alleen van toepassing op groepsberichten — directe berichten worden niet beïnvloed.

  </Tab>

  <Tab title="Sessies en deterministische antwoorden">
    - DM's gebruiken directe routering; groepen gebruiken groepsroutering.
    - Met de standaardwaarde `session.dmScope=main` worden iMessage-DM's samengevoegd in de hoofdsessie van de agent.
    - Groepssessies zijn geïsoleerd (`agent:<agentId>:imessage:group:<chat_id>`).
    - Antwoorden worden met de oorspronkelijke metadata voor kanaal/doel teruggerouteerd naar iMessage.

    Gedrag van groepachtige threads:

    Sommige iMessage-threads met meerdere deelnemers kunnen binnenkomen met `is_group=false`.
    Als die `chat_id` expliciet onder `channels.imessage.groups` is geconfigureerd, behandelt OpenClaw deze als groepsverkeer (groepspoorten + isolatie van groepssessies).

  </Tab>
</Tabs>

## ACP-gesprekskoppelingen

iMessage-chats kunnen aan ACP-sessies worden gekoppeld.

Snelle procedure voor operators:

- Voer `/acp spawn codex --bind here` uit in de DM of toegestane groepschat.
- Toekomstige berichten in datzelfde iMessage-gesprek worden naar de gestarte ACP-sessie gerouteerd.
- `/new` en `/reset` stellen dezelfde gekoppelde ACP-sessie ter plaatse opnieuw in.
- `/acp close` sluit de ACP-sessie en verwijdert de koppeling.

Geconfigureerde permanente koppelingen gebruiken vermeldingen in `bindings[]` op het hoogste niveau, met `type: "acp"` en `match.channel: "imessage"`.

`match.peer.id` kan het volgende gebruiken:

- genormaliseerde DM-handle, zoals `+15555550123` of `user@example.com`
- `chat_id:<id>` (aanbevolen voor stabiele groepskoppelingen)
- `chat_guid:<guid>`
- `chat_identifier:<identifier>`

Voorbeeld:

```json5
{
  agents: {
    list: [
      {
        id: "codex",
        runtime: {
          type: "acp",
          acp: { agent: "codex", backend: "acpx", mode: "persistent" },
        },
      },
    ],
  },
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "imessage",
        accountId: "default",
        peer: { kind: "group", id: "chat_id:123" },
      },
      acp: { label: "codex-group" },
    },
  ],
}
```

Zie [ACP-agents](/nl/tools/acp-agents) voor gedeeld gedrag van ACP-koppelingen.

## Implementatiepatronen

<AccordionGroup>
  <Accordion title="Speciale macOS-gebruiker voor de bot (afzonderlijke iMessage-identiteit)">
    Gebruik een speciale Apple ID en macOS-gebruiker, zodat botverkeer gescheiden blijft van je persoonlijke Messages-profiel.

    Gebruikelijke procedure:

    1. Maak/meld je aan bij een speciale macOS-gebruiker.
    2. Meld je in Berichten aan met de Apple ID van de bot voor die gebruiker.
    3. Installeer `imsg` voor die gebruiker.
    4. Maak een SSH-wrapper zodat OpenClaw `imsg` in de context van die gebruiker kan uitvoeren.
    5. Laat `channels.imessage.accounts.<id>.cliPath` en `.dbPath` naar dat gebruikersprofiel verwijzen.

    Voor de eerste uitvoering zijn mogelijk GUI-goedkeuringen (Automation + Full Disk Access) vereist in de gebruikerssessie van die bot.

  </Accordion>

  <Accordion title="Externe Mac via Tailscale (voorbeeld)">
    Gebruikelijke topologie:

    - Gateway draait op Linux/VM
    - iMessage + `imsg` draait op een Mac in je tailnet
    - `cliPath`-wrapper gebruikt SSH om `imsg` uit te voeren
    - `remoteHost` maakt het ophalen van bijlagen via SCP mogelijk

    Voorbeeld:

    ```json5
    {
      channels: {
        imessage: {
          enabled: true,
          cliPath: "~/.openclaw/scripts/imsg-ssh",
          remoteHost: "bot@mac-mini.tailnet-1234.ts.net",
          includeAttachments: true,
          dbPath: "/Users/bot/Library/Messages/chat.db",
        },
      },
    }
    ```

    ```bash
    #!/usr/bin/env bash
    exec ssh -T bot@mac-mini.tailnet-1234.ts.net imsg "$@"
    ```

    Gebruik SSH-sleutels zodat zowel SSH als SCP niet-interactief zijn.
    Zorg eerst dat de hostsleutel wordt vertrouwd (bijvoorbeeld `ssh bot@mac-mini.tailnet-1234.ts.net`), zodat `known_hosts` wordt gevuld.

  </Accordion>

  <Accordion title="Patroon voor meerdere accounts">
    iMessage ondersteunt configuratie per account onder `channels.imessage.accounts`.

    Elk account kan velden overschrijven zoals `cliPath`, `dbPath`, `allowFrom`, `groupPolicy`, `mediaMaxMb`, geschiedenisinstellingen en allowlists voor hoofdlocaties van bijlagen.

  </Accordion>

  <Accordion title="Geschiedenis van privéberichten">
    Stel `channels.imessage.dmHistoryLimit` in om nieuwe sessies voor privéberichten te voorzien van recente gedecodeerde `imsg`-geschiedenis voor dat gesprek. Gebruik `channels.imessage.dms["<sender>"].historyLimit` voor overschrijvingen per afzender, waaronder `0` om geschiedenis voor een afzender uit te schakelen.

    De geschiedenis van iMessage-privéberichten wordt op aanvraag opgehaald uit `imsg`. Als `dmHistoryLimit` niet is ingesteld, wordt het globaal vooraf vullen van de geschiedenis van privéberichten uitgeschakeld, maar een positieve `channels.imessage.dms["<sender>"].historyLimit` per afzender schakelt het vooraf vullen voor die afzender nog steeds in.

  </Accordion>
</AccordionGroup>

## Media, opsplitsing en afleveringsdoelen

<AccordionGroup>
  <Accordion title="Bijlagen en media">
    - het verwerken van inkomende bijlagen is **standaard uitgeschakeld** — stel `channels.imessage.includeAttachments: true` in om foto's, spraakmemo's, video's en andere bijlagen naar de agent door te sturen. Als dit is uitgeschakeld, worden iMessages die alleen een bijlage bevatten verwijderd voordat ze de agent bereiken en leveren ze mogelijk helemaal geen `Inbound message`-logregel op.
    - externe bijlagepaden kunnen via SCP worden opgehaald wanneer `remoteHost` is ingesteld
    - bijlagepaden moeten overeenkomen met toegestane hoofdlocaties:
      - `channels.imessage.attachmentRoots` (lokaal)
      - `channels.imessage.remoteAttachmentRoots` (externe SCP-modus)
      - geconfigureerde hoofdlocaties breiden het standaardpatroon voor hoofdlocaties `/Users/*/Library/Messages/Attachments` uit (samengevoegd, niet vervangen)
    - SCP gebruikt strikte controle van hostsleutels (`StrictHostKeyChecking=yes`)
    - de grootte van uitgaande media gebruikt `channels.imessage.mediaMaxMb` (standaard 16 MB)

  </Accordion>

  <Accordion title="Uitgaande tekst en opsplitsing">
    - limiet voor tekstblokken: `channels.imessage.textChunkLimit` (standaard 4000)
    - modus voor opsplitsing: `channels.imessage.streaming.chunkMode`
      - `length` (standaard)
      - `newline` (eerst opsplitsen op alinea's)
    - uitgaande Markdown voor vet/cursief/onderstrepen/doorhalen wordt omgezet in tekst met systeemeigen opmaak (ontvangers met macOS 15+ geven de opmaak weer; oudere ontvangers zien platte tekst zonder de markeringen); Markdown-tabellen worden omgezet volgens de Markdown-tabelmodus van het kanaal
    - `channels.imessage.sendTransport` (`auto` standaard, `bridge`, `applescript`) bepaalt hoe `imsg` verzendingen aflevert

  </Accordion>

  <Accordion title="Adresseringsindelingen">
    Expliciete voorkeursdoelen:

    - `chat_id:123` (aanbevolen voor stabiele routering)
    - `chat_guid:...`
    - `chat_identifier:...`

    Handledoelen worden ook ondersteund:

    - `imessage:+1555...`
    - `sms:+1555...`
    - `user@example.com`

    ```bash
    imsg chats --limit 20
    ```

  </Accordion>
</AccordionGroup>

## Acties van de privé-API

Wanneer `imsg launch` actief is en `openclaw channels status --probe` `privateApi.available: true` rapporteert, kan het berichtentool naast normale tekstverzendingen ook systeemeigen iMessage-acties gebruiken.

Alle acties zijn standaard ingeschakeld; gebruik `channels.imessage.actions` om afzonderlijke acties uit te schakelen:

```json5
{
  channels: {
    imessage: {
      actions: {
        reactions: true,
        edit: true,
        unsend: true,
        reply: true,
        sendWithEffect: true,
        sendAttachment: true,
        renameGroup: true,
        setGroupIcon: true,
        addParticipant: true,
        removeParticipant: true,
        leaveGroup: true,
        polls: true,
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="Beschikbare acties">
    - **react**: Voeg iMessage-tapbacks toe of verwijder ze (`messageId`, `emoji`, `remove`). Ondersteunde tapbacks worden gekoppeld aan liefde, leuk, niet leuk, lachen, benadrukken en vraag. Verwijderen zonder emoji wist de ingestelde tapback.
    - **reply**: Stuur een antwoord in een thread naar een bestaand bericht (`messageId`, `text` of `message`, plus `chatGuid`, `chatId`, `chatIdentifier` of `to`). Voor antwoorden met een bijlage is daarnaast een `imsg`-build nodig waarvan `send-rich` `--file` ondersteunt.
    - **sendWithEffect**: Stuur tekst met een iMessage-effect (`text` of `message`, `effect` of `effectId`). Korte namen: slam, loud, gentle, invisibleink, confetti, lasers, fireworks, balloon, heart, echo, happybirthday, shootingstar, sparkles, spotlight.
    - **edit**: Bewerk een verzonden bericht op ondersteunde versies van macOS/de privé-API (`messageId`, `text` of `newText`). Alleen berichten die de Gateway zelf heeft verzonden, kunnen worden bewerkt.
    - **unsend**: Trek een verzonden bericht in op ondersteunde versies van macOS/de privé-API (`messageId`). Alleen berichten die de Gateway zelf heeft verzonden, kunnen worden ingetrokken.
    - **upload-file**: Stuur media/bestanden (`buffer` als base64 of een gehydrateerde `media`/`path`/`filePath`, `filename`, optioneel `asVoice`). Verouderde alias: `sendAttachment`.
    - **renameGroup**, **setGroupIcon**, **addParticipant**, **removeParticipant**, **leaveGroup**: Beheer groepschats wanneer het huidige doel een groepsgesprek is. Deze wijzigen de Berichten-identiteit van de host en vereisen daarom een eigenaar als afzender of een `operator.admin` Gateway-client.
    - **poll**: Maak een systeemeigen peiling in Apple Berichten (`pollQuestion`, `pollOption` 2 tot 12 keer herhaald, plus `chatGuid`, `chatId`, `chatIdentifier` of `to`). Ontvangers met iOS/iPadOS/macOS 26+ zien de peiling en kunnen er systeemeigen op stemmen; oudere OS-versies krijgen als terugval de tekst "Er is een peiling verzonden". Vereist `selectors.pollPayloadMessage`.
    - **poll-vote**: Stem op een bestaande peiling (`pollId` of `messageId`, plus precies één van `pollOptionIndex`, `pollOptionId` of `pollOptionText`). Vereist `selectors.pollVoteMessage` en de RPC-methode `poll.vote`.

    Geaccepteerde inkomende peilingen worden voor de agent weergegeven met de vraag, genummerde optielabels, aantallen stemmen en de bericht-ID van de peiling die `poll-vote` nodig heeft.

  </Accordion>

  <Accordion title="Bericht-ID's">
    Inkomende iMessage-context bevat zowel korte `MessageSid`-waarden als volledige bericht-GUID's (`MessageSidFull`) wanneer die beschikbaar zijn. Korte ID's zijn beperkt tot de recente, door SQLite ondersteunde antwoordcache en worden vóór gebruik gecontroleerd aan de hand van de huidige chat. Als een korte ID verloopt, probeer het opnieuw met de bijbehorende `MessageSidFull` en richt je op het gesprek dat deze heeft geleverd. Volledige ID's omzeilen de binding aan het gesprek of account niet; vervang daarom een ID uit een andere chat door een ID uit het huidige doel. Extern gedelegeerde aanroepen kunnen verouderde volledige ID's weigeren wanneer bewijs voor het huidige gesprek niet beschikbaar is.

  </Accordion>

  <Accordion title="Detectie van mogelijkheden">
    OpenClaw verbergt acties van de privé-API alleen wanneer de gecachete probestatus aangeeft dat de bridge niet beschikbaar is. Als de status onbekend is, blijven acties zichtbaar en worden probes bij verzending uitgesteld uitgevoerd, zodat de eerste actie na `imsg launch` kan slagen zonder afzonderlijke handmatige statusvernieuwing.

  </Accordion>

  <Accordion title="Leesbevestigingen en typen">
    Wanneer de bridge van de privé-API actief is, worden geaccepteerde inkomende chats als gelezen gemarkeerd en tonen privéchats een typindicator zodra de beurt is geaccepteerd, terwijl de agent context voorbereidt en genereert. Schakel markeren als gelezen uit met:

    ```json5
    {
      channels: {
        imessage: {
          sendReadReceipts: false,
        },
      },
    }
    ```

    Oudere `imsg`-builds van vóór de lijst met mogelijkheden per methode schakelen typen/lezen stilzwijgend uit; OpenClaw registreert per herstart een eenmalige waarschuwing, zodat de ontbrekende bevestiging kan worden herleid.

  </Accordion>

  <Accordion title="Inkomende tapbacks">
    OpenClaw abonneert zich op iMessage-tapbacks en routeert geaccepteerde reacties als systeemgebeurtenissen in plaats van normale berichttekst, zodat een tapback van een gebruiker geen gewone antwoordlus activeert.

    De meldingsmodus wordt geregeld door `channels.imessage.reactionNotifications`:

    - `"own"` (standaard): meld alleen wanneer gebruikers reageren op berichten die door de bot zijn geschreven.
    - `"all"`: meld alle inkomende tapbacks van geautoriseerde afzenders.
    - `"off"`: negeer inkomende tapbacks.

    Overschrijvingen per account gebruiken `channels.imessage.accounts.<id>.reactionNotifications`.

  </Accordion>

  <Accordion title="Goedkeuringsreacties (👍 / 👎)">
    Wanneer `approvals.exec.enabled` of `approvals.plugin.enabled` waar is en het verzoek naar iMessage wordt gerouteerd, levert de Gateway systeemeigen een goedkeuringsprompt af en accepteert deze een tapback om het verzoek af te handelen:

    - `👍` (Like-tapback) → `allow-once`
    - `👎` (Dislike-tapback) → `deny`
    - `allow-always` blijft een handmatige terugval: stuur `/approve <id> allow-always` als een normaal antwoord.

    Voor de verwerking van reacties moet de handle van de reagerende gebruiker een expliciete goedkeurder zijn. De lijst met goedkeurders wordt gelezen uit `channels.imessage.allowFrom` (of `channels.imessage.accounts.<id>.allowFrom`); voeg het telefoonnummer van de gebruiker in E.164-indeling of het e-mailadres van diens Apple ID toe (chatdoelen zoals `chat_id:*` zijn geen geldige vermeldingen voor goedkeurders). De jokertekenvermelding `"*"` wordt gehonoreerd, maar staat elke afzender toe goed te keuren; een lege lijst met goedkeurders schakelt de reactiesnelkoppeling volledig uit. De reactiesnelkoppeling omzeilt bewust `reactionNotifications`, `dmPolicy` en `groupAllowFrom`, omdat de expliciete allowlist met goedkeurders de enige controle is die van belang is voor het afhandelen van goedkeuringen.

    Autorisatie voor de tekstuele opdracht `/approve` volgt dezelfde lijst: wanneer `channels.imessage.allowFrom` niet leeg is, wordt `/approve <id> <decision>` geautoriseerd aan de hand van die lijst met goedkeurders (niet de bredere allowlist voor privéberichten), en afzenders die wel op de allowlist voor privéberichten staan maar niet in `allowFrom`, krijgen een expliciete weigering. Wanneer `allowFrom` leeg is, blijft de terugval naar dezelfde chat van kracht en autoriseert `/approve` iedereen die door de allowlist voor privéberichten wordt toegestaan. Voeg elke operator die goedkeuringen moet kunnen geven — via `/approve` of via reacties — toe aan `allowFrom`.

    Operatornotities:
    - De reactiekoppeling wordt zowel in het geheugen als in de permanente opslag met sleutels van de Gateway bewaard (waarbij de TTL overeenkomt met de vervaldatum van de goedkeuring), en de Gateway controleert openstaande prompts ook op tapbacks, zodat een tapback die kort na een herstart van de Gateway binnenkomt de goedkeuring alsnog afhandelt.
    - De `is_from_me=true`-tapback van de operator zelf (bijvoorbeeld vanaf een gekoppeld Apple-apparaat) handelt de goedkeuring af wanneer die handle expliciet als goedkeurder is ingesteld.
    - Goedkeuringsprompts worden alleen naar een groepsgesprek gerouteerd wanneer expliciete goedkeurders zijn geconfigureerd; anders zou elk groepslid kunnen goedkeuren.
    - Oudere tapbacks in tekstvorm (`Liked "…"` platte tekst van zeer oude Apple-clients) kunnen geen goedkeuringen afhandelen omdat ze geen bericht-GUID bevatten; voor reactieafhandeling zijn de gestructureerde tapbackmetagegevens vereist die huidige macOS-/iOS-clients versturen.

  </Accordion>

  <Accordion title="Vraagreacties (1️⃣ / 2️⃣ / 3️⃣ / 4️⃣)">
    Voor een `ask_user`-prompt met één niet-geheime vraag met één selecteerbare optie en één tot vier opties voegt OpenClaw genummerde emojikeuzes toe. Reageer op de afgeleverde prompt met het overeenkomstige nummer om de vraag te beantwoorden. De reactie moet de stabiele GUID van het door de bot opgestelde bericht bevatten; OpenClaw wijst het nummer vervolgens via de Gateway toe aan de canonieke optie. Verouderde of dubbele tikken worden genegeerd.

    Prompts met meerdere vragen, meerdere selecties of vrije tekst kunnen alleen via een tekstantwoord worden beantwoord. Vraagreacties volgen de normale toelatingsregels voor iMessage-DM's en -groepen. Ze worden ook herkend wanneer algemene `reactionNotifications` `"off"` is, zonder niet-gerelateerde reacties om te zetten in agentgebeurtenissen.

  </Accordion>
</AccordionGroup>

## Configuratieschrijfbewerkingen

iMessage staat standaard door het kanaal geïnitieerde configuratieschrijfbewerkingen toe (voor `/config set|unset` wanneer `commands.config: true`).

Uitschakelen:

```json5
{
  channels: {
    imessage: {
      configWrites: false,
    },
  },
}
```

<a id="coalescing-split-send-dms-command--url-in-one-composition"></a>

## Gesplitst verzonden DM's samenvoegen (opdracht + URL in één compositie)

Apple kan een opdracht en het URL-voorbeeld ervan als afzonderlijke fysieke `chat.db`-rijen opslaan. `imsg` 0.13.1 en nieuwer voegt die rijen samen voordat bewaking, geschiedenis of zoeken het bericht retourneert, zodat OpenClaw één logisch inkomend bericht ontvangt zonder kanaalspecifieke DM-latentie toe te voegen.

Er is geen iMessage-instelling voor samenvoeging nodig. De uitgefaseerde sleutel `channels.imessage.coalesceSameSenderDms` wordt verwijderd door `openclaw doctor --fix`. Generieke `messages.inbound`-debounce blijft beschikbaar wanneer je bewust snel opeenvolgende tekstberichten binnen een kanaal wilt bundelen.

Als verzendingen met een opdracht plus URL als afzonderlijke agentbeurten binnenkomen, werk je `imsg` bij op de Mac met Berichten:

```bash
brew update && brew upgrade imsg
```

## Herstel van inkomende berichten na een herstart van de bridge of Gateway

iMessage herstelt berichten die zijn gemist terwijl de Gateway niet actief was en onderdrukt tegelijkertijd de verouderde 'backlogbom' die Apple na Push-herstel kan doorspoelen. Het standaardgedrag is altijd ingeschakeld en is gebaseerd op duurzame invoer en een leeftijdsgrens.

- **Duurzame bescherming tegen herhaling.** Voordat de herstelcursor wordt vooruitgeschoven, registreert OpenClaw elke onbewerkte rij in de gedeelde SQLite-invoerwachtrij, met de Apple-GUID als gebeurtenis-ID. Een voltooide rij laat ongeveer 4 uur lang een tombstone achter, met een maximum van 10.000 vermeldingen, zodat een herhaling met dezelfde GUID zelfs na een herstart wordt verwijderd. Een openstaande rij blijft herstelbaar totdat de verzending deze overneemt.
- **Herstel na uitvaltijd.** Bij het opstarten onthoudt de monitor de rowid van de laatst duurzaam toegelaten `chat.db`-rij (een permanente cursor per account) en geeft deze als `since_rowid` door aan `imsg watch.subscribe`, zodat imsg rijen herhaalt die nog niet waren geregistreerd en daarna live blijft volgen. Rijen die vóór een crash zijn geregistreerd, worden vanuit SQLite hervat. De herhaling is beperkt tot de meest recente 500 rijen en tot berichten van maximaal ~2 uur oud, en GUID-tombstones verwijderen alles wat al is verwerkt.
- **Leeftijdsgrens voor verouderde backlog.** Rijen boven de opstartgrens zijn daadwerkelijk live; een rij waarvan de verzenddatum meer dan ~15 minuten vóór de aankomsttijd ligt, behoort tot de doorgestroomde Push-backlog en wordt onderdrukt. Herhaalde rijen (op of onder de grens) gebruiken in plaats daarvan het ruimere herstelvenster, zodat een recent gemist bericht wordt afgeleverd maar oude geschiedenis niet.

Herstel werkt met zowel lokale als externe `cliPath`-configuraties, omdat de herhaling door `since_rowid` via dezelfde `imsg`-RPC-verbinding verloopt. Het verschil is het venster: wanneer de Gateway `chat.db` kan lezen (lokaal), verankert deze de rowid-opstartgrens, begrenst deze het herhalingsbereik en levert deze gemiste berichten af die maximaal enkele uren oud zijn. Via een externe SSH-`cliPath` kan de database niet worden gelezen, waardoor de herhaling niet wordt begrensd en elke rij de live leeftijdsgrens gebruikt. Recent gemiste berichten worden nog steeds hersteld en oude backlog wordt nog steeds onderdrukt, maar met het smallere live venster. Voer de Gateway uit op de Mac met Berichten voor het ruimere herstelvenster.

### Voor de operator zichtbaar signaal

Onderdrukte backlog wordt op het standaardniveau vastgelegd en nooit stilzwijgend verwijderd (de vlag `recovery` geeft aan welk venster is toegepast):

```text
imessage: verouderde inkomende backlog onderdrukt account=<id> verzonden=<iso> herstel=<bool> (<N> onderdrukt sinds de start)
```

### Migratie

`channels.imessage.catchup.*` is verouderd — herstel na uitvaltijd gebeurt automatisch en vereist voor nieuwe configuraties geen configuratie. Bestaande configuraties met `catchup.enabled: true` blijven als compatibiliteitsprofiel voor het venster voor herhaling bij herstel ondersteund. Uitgeschakelde inhaalblokken (`enabled: false` of geen `enabled: true`) zijn uitgefaseerd; `openclaw doctor --fix` verwijdert deze.

## Problemen oplossen

<AccordionGroup>
  <Accordion title="imsg niet gevonden of RPC niet ondersteund">
    Valideer het binaire bestand en de RPC-ondersteuning:

    ```bash
    imsg rpc --help
    imsg status --json
    openclaw channels status --probe
    ```

    Als de probe meldt dat RPC niet wordt ondersteund, werk je `imsg` bij. Als acties via de privé-API niet beschikbaar zijn, voer je `imsg launch` uit in de aangemelde macOS-gebruikerssessie en voer je de probe opnieuw uit. Als de Gateway niet op macOS wordt uitgevoerd, gebruik je de bovenstaande configuratie voor een externe Mac via SSH in plaats van het standaard lokale `imsg`-pad.

  </Accordion>

  <Accordion title="Berichten worden verzonden, maar inkomende iMessages komen niet aan">
    Stel eerst vast of het bericht de lokale Mac heeft bereikt. Als `chat.db` niet verandert, kan OpenClaw het bericht niet ontvangen, zelfs wanneer `imsg status --json` een gezonde bridge rapporteert.

```bash
imsg chats --limit 10 --json
imsg watch --chat-id <chat-id> --json
sqlite3 ~/Library/Messages/chat.db \
  "select datetime(max(date)/1000000000 + 978307200, 'unixepoch', 'localtime'), max(ROWID) from message;"
```

    Als vanaf de telefoon verzonden berichten geen nieuwe rijen maken, herstel je de macOS-lagen voor Berichten en Apple Push voordat je de OpenClaw-configuratie wijzigt. Een eenmalige vernieuwing van de services is vaak voldoende:

```bash
launchctl kickstart -k system/com.apple.apsd
launchctl kickstart -k gui/$(id -u)/com.apple.CommCenter
launchctl kickstart -k gui/$(id -u)/com.apple.identityservicesd
launchctl kickstart -k gui/$(id -u)/com.apple.imagent
imsg launch
openclaw gateway restart
```

    Stuur een nieuw iMessage vanaf de telefoon en bevestig een nieuwe `chat.db`-rij of `imsg watch`-gebeurtenis voordat je fouten in OpenClaw-sessies opspoort. Voer dit niet uit als periodieke herstartlus voor de bridge; herhaalde `imsg launch` plus herstarts van de Gateway tijdens actief werk kunnen afleveringen onderbreken en actieve kanaaluitvoeringen laten vastlopen.

  </Accordion>

  <Accordion title="Gateway wordt niet uitgevoerd op macOS">
    De standaard `cliPath: "imsg"` moet worden uitgevoerd op de Mac die bij Berichten is aangemeld. Stel op Linux of Windows `channels.imessage.cliPath` in op een wrapperscript dat via SSH verbinding maakt met die Mac en `imsg "$@"` uitvoert.

```bash
#!/usr/bin/env bash
exec ssh -T messages-mac imsg "$@"
```

    Voer vervolgens uit:

```bash
openclaw channels status --probe --channel imessage
```

  </Accordion>

  <Accordion title="DM's worden genegeerd">
    Controleer:

    - `channels.imessage.dmPolicy`
    - `channels.imessage.allowFrom`
    - koppelingsgoedkeuringen (`openclaw pairing list imessage`)

  </Accordion>

  <Accordion title="Groepsberichten worden genegeerd">
    Controleer:

    - `channels.imessage.groupPolicy`
    - `channels.imessage.groupAllowFrom`
    - gedrag van de toelatingslijst voor `channels.imessage.groups`
    - configuratie van vermeldingspatronen (`agents.entries.*.groupChat.mentionPatterns`)

  </Accordion>

  <Accordion title="Externe bijlagen mislukken">
    Controleer:

    - `channels.imessage.remoteHost`
    - `channels.imessage.remoteAttachmentRoots`
    - SSH-/SCP-sleutelauthenticatie vanaf de Gateway-host
    - of de hostsleutel op de Gateway-host aanwezig is in `~/.ssh/known_hosts`
    - of het externe pad leesbaar is op de Mac waarop Berichten wordt uitgevoerd

  </Accordion>

  <Accordion title="macOS-toestemmingsprompts zijn gemist">
    Voer de opdrachten opnieuw uit in een interactieve GUI-terminal binnen dezelfde gebruikers-/sessiecontext en keur de prompts goed:

    ```bash
    imsg chats --limit 1
    imsg send <handle> "test"
    ```

    Bevestig dat Volledige schijftoegang + Automatisering zijn verleend voor de procescontext waarin OpenClaw/`imsg` wordt uitgevoerd.

  </Accordion>
</AccordionGroup>

## Verwijzingen naar de configuratiereferentie

- [Configuratiereferentie - iMessage](/nl/gateway/config-channels#imessage)
- [Gateway-configuratie](/nl/gateway/configuration)
- [Koppelen](/nl/channels/pairing)

## Gerelateerd

- [Overzicht van kanalen](/nl/channels) — alle ondersteunde kanalen
- [Verwijdering van BlueBubbles en het imsg-pad voor iMessage](/nl/announcements/bluebubbles-imessage) — aankondiging en migratiesamenvatting
- [Overstappen vanaf BlueBubbles](/nl/channels/imessage-from-bluebubbles) — tabel voor configuratievertaling en stapsgewijze overstap
- [Koppelen](/nl/channels/pairing) — DM-authenticatie en koppelingsflow
- [Groepen](/nl/channels/groups) — gedrag van groepschats en toelating op basis van vermeldingen
- [Kanaalroutering](/nl/channels/channel-routing) — sessieroutering voor berichten
- [Beveiliging](/nl/gateway/security) — toegangsmodel en beveiliging aanscherpen
