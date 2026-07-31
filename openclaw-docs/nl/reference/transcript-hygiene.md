---
read_when:
    - Je debugt afwijzingen van providerverzoeken die verband houden met de transcriptstructuur
    - Je wijzigt de opschoning van transcripties of de herstellogica voor toolaanroepen
    - Je onderzoekt verschillen in tool-call-id's tussen providers
summary: 'Referentie: providerspecifieke regels voor transcriptopschoning en -herstel'
title: Transcripthygiëne
x-i18n:
    generated_at: "2026-07-27T05:22:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 33d978772062cb2a81eb358bb5c62bd1261b433ffdc8acdbaa6679b121fbbf62
    source_path: reference/transcript-hygiene.md
    workflow: 16
---

OpenClaw past **providerspecifieke correcties** toe op transcripties vóór een uitvoering
(bij het opbouwen van de modelcontext). De meeste hiervan zijn aanpassingen **in het geheugen** die worden gebruikt om
aan strikte providervereisten te voldoen. Een afzonderlijke herstelstap voor sessiebestanden kan
ook opgeslagen JSONL herschrijven voordat de sessie wordt geladen, maar alleen voor
ongeldige regels of opgeslagen beurten die geen geldige duurzame records zijn.
Afgeleverde assistentantwoorden blijven op schijf behouden; het verwijderen van
providerspecifieke assistent-prefill gebeurt alleen tijdens het samenstellen van uitgaande
payloads.

Wanneer een herstel plaatsvindt, wordt het oorspronkelijke bestand naar een tijdelijk
`*.bak-<pid>-<ts>`-zusterbestand geschreven vóór de atomische vervanging en vervolgens verwijderd zodra de
vervanging slaagt. De back-up blijft alleen behouden als het opschonen zelf mislukt; in
dat geval wordt het pad teruggemeld.

Het bereik omvat:

- Promptcontext die alleen tijdens runtime bestaat buiten voor gebruikers zichtbare transcriptiebeurten houden
- Opschoning van toolaanroep-id's
- Validatie van toolaanroepinvoer
- Herstel van koppeling van toolresultaten
- Validatie / volgorde van beurten
- Opschoning van denksignaturen
- Opschoning van redeneersignaturen
- Opschoning van afbeeldingspayloads
- Opschoning van lege tekstblokken vóór herhaling door de provider
- Opschoning van onvolledige beurten met alleen redenering en een lengtelimiet vóór herhaling door de provider
- Herkomstlabels voor gebruikersinvoer (voor tussen sessies gerouteerde prompts)
- Herstel van lege assistentfoutbeurten voor herhaling met Bedrock Converse

Zie voor details over transcriptieopslag
[Diepgaande uitleg over sessiebeheer](/nl/reference/session-management-compaction).

---

## Algemene regel: runtimecontext is geen gebruikerstranscriptie

Runtime-/systeemcontext kan voor een beurt aan de modelprompt worden toegevoegd, maar is
geen door de eindgebruiker geschreven inhoud. OpenClaw houdt een afzonderlijke, op transcripties gerichte
prompttekst bij voor Gateway-antwoorden, vervolgacties in de wachtrij, ACP, CLI en ingebedde
OpenClaw-uitvoeringen. Opgeslagen zichtbare gebruikersbeurten gebruiken die transcriptietekst in plaats van
de met runtimecontext verrijkte prompt.

Voor verouderde sessies waarin runtimewrappers al zijn opgeslagen, passen
Gateway-geschiedenisoppervlakken een weergaveprojectie toe voordat berichten aan WebChat-,
TUI-, REST- of SSE-clients worden geretourneerd.

---

## Waar dit wordt uitgevoerd

Alle transcriptiehygiëne is gecentraliseerd in de ingebedde runner:

- Beleidsselectie: `src/agents/transcript-policy.ts`
  (`resolveTranscriptPolicy`, gebaseerd op `provider`, `modelApi` en `modelId`)
- Toepassing van opschoning/herstel: `sanitizeSessionHistory` in
  `src/agents/embedded-agent-runner/replay-history.ts`

Los van transcriptiehygiëne worden sessiebestanden vóór het laden hersteld
(indien nodig):

- `repairSessionFileIfNeeded` in `src/agents/session-file-repair.ts`
- Aangeroepen vanuit `src/agents/embedded-agent-runner/run/attempt.ts` en
  `src/agents/embedded-agent-runner/compact.ts`

---

## Algemene regel: opschoning van afbeeldingen

Afbeeldingspayloads worden altijd opgeschoond om afwijzing door de provider vanwege
groottelimieten te voorkomen (te grote base64-afbeeldingen verkleinen/opnieuw comprimeren). Dit helpt ook
de door afbeeldingen veroorzaakte tokendruk voor modellen met visiemogelijkheden te
beheersen: lagere maximale afmetingen verminderen het tokengebruik, hogere afmetingen behouden details.

Implementatie:

- `sanitizeSessionMessagesImages` in
  `src/agents/embedded-agent-helpers/images.ts`
- `sanitizeContentBlocksImages` in `src/agents/tool-images.ts`
- De maximale afbeeldingszijde is configureerbaar via `agents.defaults.imageMaxDimensionPx`
  (standaard: `1200`)
- Lege tekstblokken worden verwijderd terwijl deze stap de herhalingsinhoud doorloopt.
  Assistentbeurten die daardoor leeg worden, worden uit de herhalingskopie verwijderd; gebruikers-
  en toolresultaatbeurten die leeg worden, krijgen een niet-lege
  tijdelijke aanduiding voor weggelaten inhoud.

---

## Algemene regel: ongeldige toolaanroepen

Assistentblokken voor toolaanroepen waarin zowel `input` als `arguments` ontbreken, worden verwijderd
voordat de modelcontext wordt opgebouwd. Dit voorkomt afwijzingen door providers vanwege
gedeeltelijk opgeslagen toolaanroepen (bijvoorbeeld na een fout door een snelheidslimiet).

Implementatie:

- `sanitizeToolCallInputs` in `src/agents/session-transcript-repair.ts`
- Toegepast in `sanitizeSessionHistory`
  (`src/agents/embedded-agent-runner/replay-history.ts`)

---

## Algemene regel: koppeling van toolresultaten

Toolresultaten worden binnen elke assistentbeurt gekoppeld aan voorkomens van toolaanroepen voordat
providerspecifieke aanroep-id's worden herschreven. Door providers gegenereerde id's kunnen in latere
beurten worden herhaald, zodat een resultaat naast een herhaalde aanroep aan dat voorkomen gekoppeld blijft. Een verplaatst
resultaat wordt alleen verplaatst wanneer precies één onopgelost voorkomen er eigenaar van kan zijn; dubbelzinnige
extra resultaten worden verwijderd en ontbrekende voorkomens krijgen synthetische foutresultaten.

Implementatie: `sanitizeToolUseResultPairing` in
`src/agents/session-transcript-repair.ts`

---

## Algemene regel: onvolledige of stille beurten met alleen redenering

Assistentbeurten worden uit de herhalingskopie in het geheugen weggelaten wanneer ze
na een van deze gebeurtenissen alleen denk- of geredigeerde denkinhoud bevatten:

- De uitvoerlimiet van de provider beëindigt de beurt met een onvolledige redeneerstatus.
- De opschoning van stille antwoorden verwijdert de enige zichtbare `NO_REPLY`-tekst van de beurt.

De opschoning van stille antwoorden voorkomt dat verborgen redenering wordt samengevoegd met een latere
assistentbeurt met toolgebruik wanneer strikte providers het gesprek opnieuw opbouwen.

Lege beurten met een lengtelimiet blijven ongewijzigd, net als beurten met een lengtelimiet en zichtbare tekst,
toolaanroepen of onbekende inhoudsblokken. Beurten met stille antwoorden en toolaanroepen of
onbekende inhoudsblokken blijven eveneens ongewijzigd. Opgeslagen transcripties worden niet
herschreven.

Implementatie: `normalizeAssistantReplayContent` in
`src/agents/embedded-agent-runner/replay-history.ts`

---

## Algemene regel: herkomst van invoer tussen sessies

Wanneer een agent via `sessions_send` een prompt naar een andere sessie stuurt
(inclusief antwoord-/aankondigingsstappen tussen agents), slaat OpenClaw de
aangemaakte gebruikersbeurt op met `message.provenance.kind = "inter_session"`.

OpenClaw voegt ook een `[Inter-session message] ... isUser=false`-markering voor dezelfde beurt
toe vóór de gerouteerde prompttekst, zodat de actieve modelaanroep
uitvoer uit een andere sessie kan onderscheiden van externe instructies van eindgebruikers. Deze
markering bevat waar beschikbaar de bronsessie, het kanaal en de tool. De
transcriptie gebruikt voor providercompatibiliteit nog steeds `role: "user"`, maar zowel de
zichtbare tekst als de herkomstmetagegevens markeren de beurt als gegevens uit een andere sessie.

Tijdens het opnieuw opbouwen van de context past OpenClaw dezelfde markering toe op oudere opgeslagen
gebruikersbeurten uit andere sessies die alleen herkomstmetagegevens hebben.

---

## Providermatrix (huidig gedrag)

**OpenAI / OpenAI Codex**

- Alleen opschoning van afbeeldingen.
- Verweesde redeneersignaturen verwijderen (zelfstandige redeneeritems zonder een
  volgend inhoudsblok) voor OpenAI Responses-/Codex-transcripties en
  herhaalbare OpenAI-redenering verwijderen na een wisseling van modelroute.
- Payloads van herhaalbare OpenAI Responses-redeneeritems behouden, inclusief
  versleutelde items met een lege samenvatting, zodat handmatige/WebSocket-herhaling de vereiste
  `rs_*`-status gekoppeld houdt aan assistentuitvoeritems.
- Native ChatGPT Codex Responses volgt Codex-protocolpariteit door
  eerdere Responses-payloads voor redenering/berichten/functies te herhalen zonder eerdere item-
  id's, terwijl de `prompt_cache_key` van de sessie behouden blijft.
- Herhaling binnen de OpenAI Responses-familie behoudt canonieke `call_*|fc_*`-
  redeneerparen voor hetzelfde model, maar normaliseert ongeldige of
  te lange id's van `call_id`/functieaanroepitems deterministisch vóór pi-ai-payloadconversie.
- Herstel van koppeling van toolresultaten kan echte overeenkomende uitvoer verplaatsen en
  Codex-achtige `aborted`-uitvoer synthetiseren voor ontbrekende toolaanroepen.
- Geen validatie of herschikking van beurten; geen verwijdering van denksignaturen.

**OpenAI-compatibele Chat Completions**

- Historische denk-/redeneerblokken van assistenten worden vóór herhaling verwijderd,
  zodat lokale en proxy-achtige OpenAI-compatibele servers geen
  redeneervelden uit eerdere beurten ontvangen, zoals `reasoning` of `reasoning_content`.
- Huidige voortzettingen van toolaanroepen binnen dezelfde beurt houden het redeneerblok van de assistent
  aan de toolaanroep gekoppeld totdat het toolresultaat is herhaald.
- Aangepaste/zelfgehoste modelvermeldingen met `reasoning: true` behouden herhaalde
  redeneermetagegevens.
- Uitzonderingen die eigendom zijn van providers kunnen zich afmelden wanneer hun protocol
  herhaalde redeneermetagegevens vereist.

**Google (Generative AI / Gemini CLI / Antigravity)**

- Opschoning van toolaanroep-id's: strikt alfanumeriek.
- Herstel van koppeling van toolresultaten en synthetische toolresultaten.
- Validatie van beurten (Gemini-achtige beurtwisseling).
- Correctie van de Google-beurtvolgorde (voeg een kleine gebruikersbootstrap vooraan toe als de geschiedenis
  met de assistent begint).
- Antigravity Claude: redeneersignaturen normaliseren; niet-ondertekende redeneerblokken
  verwijderen.

**Anthropic / Minimax (Anthropic-compatibel)**

- Herstel van koppeling van toolresultaten en synthetische toolresultaten.
- Validatie van beurten (opeenvolgende gebruikersbeurten samenvoegen om aan strikte
  afwisseling te voldoen).
- Afsluitende assistent-prefillbeurten worden uit uitgaande Anthropic
  Messages-payloads verwijderd wanneer redeneren is ingeschakeld, inclusief Cloudflare AI
  Gateway-routes.
- Redeneersignaturen van de assistent van vóór Compaction worden vóór herhaling door de provider
  verwijderd wanneer een sessie is gecompacteerd. Redeneersignaturen zijn
  op het moment van generatie cryptografisch aan het gespreksvoorvoegsel gebonden;
  na Compaction verandert het voorvoegsel (samengevatte inhoud vervangt het
  origineel), waardoor Anthropic het verzoek bij herhaling van de oorspronkelijke signaturen
  afwijst met "Ongeldige signatuur in redeneerblok". De
  redeneertekst blijft behouden als een niet-ondertekend blok en wordt vervolgens volgens de
  onderstaande regel verwerkt.
- Redeneerblokken met ontbrekende, lege of blanco herhalingssignaturen worden
  vóór providerconversie verwijderd. Als daardoor een assistentbeurt leeg wordt,
  behoudt OpenClaw de beurtstructuur met niet-lege tekst voor weggelaten redenering.
- Oudere assistentbeurten met alleen redenering die moeten worden verwijderd, worden vervangen
  door niet-lege tekst voor weggelaten redenering, zodat provideradapters
  de herhalingsbeurt niet verwijderen.

**Amazon Bedrock (Converse API)**

- Lege assistentbeurten met streamfouten worden vóór herhaling hersteld naar een niet-leeg
  terugvaltekstblok. Bedrock Converse wijst assistentberichten
  met `content: []` af, zodat opgeslagen assistentbeurten met `stopReason:
"error"` en lege inhoud vóór het laden ook op schijf worden hersteld.
- Assistentbeurten met streamfouten die alleen lege tekstblokken bevatten, worden uit
  de herhalingskopie in het geheugen verwijderd in plaats van een ongeldig leeg blok te herhalen.
- Redeneersignaturen van de assistent van vóór Compaction worden vóór Converse-
  herhaling verwijderd wanneer een sessie is gecompacteerd, om dezelfde reden als bij
  Anthropic hierboven.
- Claude-redeneerblokken met ontbrekende, lege of blanco herhalingssignaturen
  worden vóór Converse-herhaling verwijderd. Als daardoor een assistentbeurt leeg wordt,
  behoudt OpenClaw de beurtstructuur met niet-lege tekst voor weggelaten redenering.
- Oudere assistentbeurten met alleen redenering die moeten worden verwijderd, worden vervangen
  door niet-lege tekst voor weggelaten redenering, zodat de Converse-herhaling
  de strikte beurtstructuur behoudt.
- Herhaling filtert OpenClaw-beurten van assistenten die door afleveringsspiegeling en de Gateway
  zijn ingevoegd.
- Opschoning van afbeeldingen wordt via de algemene regel toegepast.

**Mistral (inclusief detectie op basis van model-id)**

- Opschoning van toolaanroep-id's: strict9 (alfanumeriek, lengte 9).

**OpenRouter Gemini**

- Opschoning van denksignaturen: niet-base64-waarden van `thought_signature` verwijderen
  (base64 behouden).

**OpenRouter Anthropic**

- Afsluitende assistent-prefillbeurten worden verwijderd uit geverifieerde OpenRouter-
  payloads van OpenAI-compatibele Anthropic-modellen wanneer redeneren is ingeschakeld,
  overeenkomstig het herhalingsgedrag van directe Anthropic- en Cloudflare Anthropic-routes.

**Al het overige**

- Alleen opschoning van afbeeldingen.

---

## Historisch gedrag (vóór 2026.1.22)

Vóór de release 2026.1.22 paste OpenClaw meerdere lagen
transcriptiehygiëne toe:

- Een **transcript-sanitize-extensie** werd bij elke contextopbouw uitgevoerd en kon:
  - De koppeling tussen toolgebruik en -resultaat herstellen.
  - Toolaanroep-id's opschonen (inclusief een niet-strikte modus die
    `_`/`-` behield).
- De runner voerde ook providerspecifieke opschoning uit, waardoor
  werk werd gedupliceerd.
- Buiten het providerbeleid vonden aanvullende wijzigingen plaats, waaronder
  het verwijderen van `<final>`-tags uit assistenttekst vóór opslag, het weglaten
  van lege assistentbeurten met fouten en het inkorten van assistentinhoud na
  toolaanroepen.

Deze complexiteit veroorzaakte regressies tussen providers (met name bij de
koppeling van `openai-responses` en `call_id|fc_id`). Bij de opschoning van 2026.1.22 werd
de extensie verwijderd, de logica gecentraliseerd in de runner en OpenAI **ongewijzigd**
gelaten, afgezien van het opschonen van afbeeldingen.

## Gerelateerd

- [Sessiebeheer](/nl/concepts/session)
- [Sessieverkleining](/nl/concepts/session-pruning)
