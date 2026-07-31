---
read_when:
    - Je wilt dat de agent vanuit de chat een skill maakt of bijwerkt
    - Je moet een gegenereerd skillconcept beoordelen, toepassen, afwijzen of in quarantaine plaatsen
    - Je configureert goedkeuring, autonomie, opslag of limieten voor Skill Workshop
    - Je wilt begrijpen waar voorstellen voor zelflerend vermogen worden beoordeeld
sidebarTitle: Skill Workshop
summary: Werkruimtevaardigheden maken en bijwerken via beoordeling in Skill Workshop
title: Skills-workshop
x-i18n:
    generated_at: "2026-07-27T05:20:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2c2590f2a1bcad3b22ef8504eac7b3a44611c3fedc0df3832660f8926ce04252
    source_path: tools/skill-workshop.md
    workflow: 16
---

Skill Workshop is het beheerde traject van OpenClaw voor het maken en bijwerken van
Skills in de werkruimte. Agents en operators schrijven via dit traject nooit rechtstreeks
`SKILL.md` — ze maken een **voorstel** (een concept in afwachting met inhoud,
doelkoppeling, scannerstatus, hashes en terugdraaimetadata) dat pas een actieve
Skill wordt wanneer het wordt toegepast.

Skill Workshop schrijft uitsluitend Skills in de werkruimte. Het raakt nooit meegeleverde,
Plugin-, ClawHub-, extra-root-, beheerde, persoonlijke-agent- of systeem-Skills aan.

## Hoe het werkt

- **Eerst een voorstel:** gegenereerde inhoud wordt opgeslagen als `PROPOSAL.md`, niet
  als `SKILL.md`.
- **Toepassen is de enige actieve schrijfbewerking:** maken, bijwerken en herzien wijzigen
  actieve Skills nooit.
- **Beperkt tot de werkruimte:** nieuwe Skills richten zich op de `skills/`-root van de werkruimte; updates
  zijn alleen toegestaan voor beschrijfbare Skills in de werkruimte.
- **Niet overschrijven:** maken mislukt als de doel-Skill al bestaat.
- **Aan hash gebonden:** updatevoorstellen worden aan de huidige doelhash gebonden en worden
  `stale` als de actieve Skill vóór het toepassen verandert.
- **Door scanner bewaakt:** vóór het schrijven voert toepassen de beveiligingsscanner opnieuw uit.
- **Herstelbaar:** toepassen schrijft terugdraaimetadata voordat actieve bestanden worden gewijzigd.
- **Consistente interfaces:** chat, CLI en Gateway roepen allemaal dezelfde service aan.

## Levenscyclus

```text
maken/bijwerken -> in afwachting
herzien         -> in afwachting
toepassen       -> toegepast
afwijzen        -> afgewezen
in quarantaine  -> in quarantaine
doelwijziging   -> verouderd
```

Alleen een `pending` voorstel kan worden herzien, toegepast, afgewezen of in quarantaine geplaatst.

## Beheer van de levenscyclus

De Gateway houdt het totale gebruik van Skills bij in de gedeelde statusdatabase. Eenmaal per
dag beoordeelt deze Skills die door Skill Workshop zijn gemaakt en toegepast. Skills die langer dan
30 dagen niet zijn gebruikt, worden `stale`; na 90 dagen worden ze `archived` en
niet opgenomen in nieuwe Skill-snapshots van agents. Gearchiveerde Skill-bestanden blijven ongewijzigd op
schijf staan. Handmatig geschreven Skills worden nooit beheerd; alleen Skills die door voorstellen van Skill
Workshop zijn gemaakt, worden opgenomen in het levenscyclusbeheer.

Vastgezette Skills slaan levenscyclusovergangen over. Een verouderde Skill keert terug naar `active`
nadat deze is gebruikt en de volgende controle is uitgevoerd. Gearchiveerde Skills keren alleen terug via
expliciet herstel:

Levenscyclusovergangen en herstelbewerkingen gelden voor nieuwe sessies; actieve sessies behouden
hun huidige Skill-snapshot.

```bash
openclaw skills curator status
openclaw skills curator pin <skill>
openclaw skills curator unpin <skill>
openclaw skills curator restore <skill>
```

Alle curatoropdrachten accepteren `--json`. De status rapporteert ook deterministische kandidaten
met overlap, maar uitsluitend als suggesties; Skills worden nooit samengevoegd en er wordt nooit een model aangeroepen.

## Chat

Vraag de agent om de gewenste Skill; deze roept `skill_workshop` aan en retourneert een
voorstel-id.

### Leren van recent werk

Gebruik `/learn` om het huidige gesprek of benoemde bronnen om te zetten in één
door standaarden gestuurd Skill-voorstel:

```text
/learn
/learn docs/runbook.md en https://example.com/guide; richt je op herstel
```

Zonder verzoek vraagt `/learn` de agent om de herbruikbare workflow uit
het huidige gesprek te destilleren. Met een verzoek behandelt de agent paden, URL's, geplakte
notities en verwijzingen naar gesprekken als bronnen, met inachtneming van vereisten voor focus, bereik en
naamgeving. De bronnen worden verzameld met de bestaande tools, waarna
`skill_workshop` wordt aangeroepen met `action: "create"`.

Het resulterende voorstel blijft `pending`; `/learn` past het nooit toe. Beoordeel en
pas het toe via de normale goedkeuringsflow of met `openclaw skills workshop`.

Maken:

```text
Maak een Skill met de naam morning-catchup die mijn inboxroutine op maandag uitvoert.
```

Een bestaande Skill in de werkruimte bijwerken:

```text
Werk trip-planning bij zodat ook stoelplattegronden worden gecontroleerd voordat er wordt geboekt.
```

Een voorstel in afwachting iteratief verbeteren:

```text
Toon me het voorstel morning-catchup.
Herzie het zodat ook alles wat als urgent is gemarkeerd, wordt gesignaleerd.
Pas het voorstel morning-catchup toe.
```

Door agents geïnitieerde `apply`, `reject` en `quarantine` worden standaard zonder een extra
goedkeuringsprompt uitgevoerd. Stel `skills.workshop.approvalPolicy` in op `"pending"`
om goedkeuring door een operator te vereisen voordat deze acties worden uitgevoerd.

Wanneer goedkeuring vereist is, vermeldt de prompt het voorstel-id en de doel-Skill,
en toont deze de beschrijving van het voorstel, het aantal ondersteuningsbestanden en de grootte van de hoofdtekst.
Goedkeuringsverzoeken zijn begrensd zodat ze worden afgerond voordat de bewaking van de agenttool ingrijpt. Als vóór
het verlopen van de prompt geen beslissing is ontvangen, wordt de levenscyclusactie niet uitgevoerd:
het voorstel blijft ongewijzigd in afwachting. Beslis later in de gebruikersinterface van Skill Workshop of voer
`openclaw skills workshop apply|reject|quarantine <proposal-id>` uit. Agents mogen
een verlopen levenscyclusactie niet herhaaldelijk opnieuw proberen.

## CLI

```bash
# Maken
openclaw skills workshop propose-create \
  --name morning-catchup \
  --description "Dagelijkse inhaalslag voor de inbox: triëren, archiveren, uitlichten, concepten opstellen, plannen" \
  --proposal ./PROPOSAL.md

# Een bestaande Skill in de werkruimte bijwerken
openclaw skills workshop propose-update trip-planning --proposal ./PROPOSAL.md

# Weergeven en inspecteren
openclaw skills workshop list
openclaw skills workshop inspect <proposal-id>

# Herzien vóór goedkeuring
openclaw skills workshop revise <proposal-id> --proposal ./PROPOSAL.md

# Afronden
openclaw skills workshop apply <proposal-id>
openclaw skills workshop reject <proposal-id> --reason "Duplicaat"
openclaw skills workshop quarantine <proposal-id> --reason "Beveiligingsbeoordeling vereist"
```

Elke subopdracht accepteert `--agent <id>` (doelwerkruimte; standaard afgeleid van
de huidige werkmap en daarna de standaardagent) en `--json` (gestructureerde uitvoer).
`propose-create`, `propose-update` en `revise` accepteren ook `--goal <text>` en
`--evidence <text>` om de context van het voorstel naast `--proposal` vast te leggen.

## Inhoud van voorstellen

Zolang het voorstel in afwachting is, wordt het opgeslagen als `PROPOSAL.md` met frontmatter
die alleen voor voorstellen geldt:

```markdown
---
name: "morning-catchup"
description: "Dagelijkse inhaalslag voor de inbox: triëren, archiveren, uitlichten, concepten opstellen, plannen"
status: proposal
version: "v1"
date: "2026-05-30T00:00:00.000Z"
---
```

Bij het toepassen schrijft Skill Workshop de actieve `SKILL.md` en verwijdert het
de velden die alleen voor voorstellen gelden: `status`, voorstel-`version` en voorstel-`date`.

## Ondersteuningsbestanden

Gebruik `--proposal-dir` wanneer de voorgestelde Skill bestanden naast
`PROPOSAL.md` nodig heeft:

```bash
openclaw skills workshop propose-create \
  --name weekly-update \
  --description "Afronding op vrijdag: statistieken, hoogtepunten, de drie belangrijkste punten voor volgende week" \
  --proposal-dir ./weekly-update-proposal
```

De map moet `PROPOSAL.md` bevatten. Ondersteuningsbestanden moeten zich bevinden onder
`assets/`, `examples/`, `references/`, `scripts/` of `templates/`. Skill
Workshop scant en hasht ze, en slaat ze bij het voorstel op. Pas bij het toepassen worden ze
naast de actieve `SKILL.md` geschreven.

Niet-toegestane paden voor ondersteuningsbestanden: absolute paden, verborgen padsegmenten,
padtraversal, overlappende paden, uitvoerbare bestanden, tekst die niet UTF-8 is, null-bytes
en paden buiten de standaardmappen voor ondersteuning.

## Agenttool

Het model gebruikt `skill_workshop` met één vereiste `action`:
`create | update | revise | list | inspect | apply | reject | quarantine`.
Andere parameters zijn afhankelijk van de actie:

| Parameter                  | Gebruikt door                                         | Opmerkingen                                                          |
| -------------------------- | ----------------------------------------------------- | -------------------------------------------------------------------- |
| `name`                     | `create`, `inspect`, `revise`                        | Vereist voor `create`; zoekt anders een voorstel in afwachting op naam op |
| `description`              | `create`, `update`, `revise`                         | Maximaal 160 bytes                                                    |
| `skill_name`               | `update`                                             | Naam of sleutel van bestaande Skill                                  |
| `proposal_content`         | `create`, `update`, `revise`                         | Opgeslagen als `PROPOSAL.md`; begrensd door `skills.workshop.maxSkillBytes` |
| `support_files`            | `create`, `update`, `revise`                         | Array van `{ path, content }`                                        |
| `goal`, `evidence`         | `create`, `update`, `revise`                         | Vrije tekst als context                                               |
| `proposal_id`              | `inspect`, `revise`, `apply`, `reject`, `quarantine` | Doelvoorstel                                                         |
| `reason`                   | `apply`, `reject`, `quarantine`                      | Optioneel                                                            |
| `query`, `status`, `limit` | `list`                                               | Filteren/pagineren; `limit` maximaal 50, standaard 20     |

Agents moeten `skill_workshop` gebruiken voor gegenereerd Skill-werk. Ze mogen
voorstelbestanden niet maken of wijzigen via `write`, `edit`, `exec`, shellopdrachten
of rechtstreekse bestandssysteembewerkingen.

<Note>
`skill_workshop` is een ingebouwde agenttool en is opgenomen in
`tools.profile: "coding"`. Als deze door strenger beleid wordt verborgen, voeg dan
`skill_workshop` toe aan de actieve lijst `tools.allow`, of gebruik
`tools.alsoAllow: ["skill_workshop"]` wanneer het bereik een profiel zonder een
expliciete `tools.allow` gebruikt. In runs met een sandbox wordt de hosttool
Skill Workshop niet samengesteld. Voer acties voor het beoordelen van voorstellen daarom uit vanuit een normale
agentsessie aan de hostzijde of via de CLI.
</Note>

## Voorgestelde Skills

OpenClaw detecteert duurzame instructies zoals ‘de volgende keer’, ‘onthoud om’ en reactieve correcties
wanneer een interactieve beurt eindigt, ook na mislukte beurten. Tijdens de volgende beurt biedt de agent aan
om de meest recent gedetecteerde workflow op te slaan via `skill_workshop`; de gebruiker beslist of er een
voorstel wordt gemaakt. Deze ingebouwde suggestie maakt of wijzigt op zichzelf geen Skill. Schakel
`skills.workshop.autonomous.enabled` in om in plaats daarvan rechtstreeks voorstellen in afwachting te maken. Op het tabblad Workshop
in de Control UI is dezelfde instelling beschikbaar als de schakelaar **Zelflerend** in de paginakop en
als een inschakelknop op het lege voorstelbord.

### Eerdere sessies scannen

De Control UI kan ouder werk beoordelen zonder autonoom zelfleren in te schakelen.
Open **Plugins → Workshop** en selecteer **Skill-ideeën zoeken**. De scan begint bij
de nieuwste geschikte sessies en beoordeelt een begrensd venster met substantieel werk.
Cron-, Heartbeat-, hook-, subagent-, ACP-, Plugin-eigen en interne beoordelingssessies
worden overgeslagen, evenals gesprekken met minder dan zes modelbeurten.

De beoordelaar gebruikt het geconfigureerde model van de geselecteerde agent en ontvangt een
op grootte begrensde transcriptbundel waarin geheimen zijn geredigeerd. Dezelfde conservatieve
drempel als bij ervaringsbeoordeling wordt toegepast: een concreet herstelpatroon of een stabiele procedure die
ten minste twee toekomstige model- of toolaanroepen zou voorkomen. Routinematig werk en eenmalige
feiten horen geen voorstel op te leveren.

Eén scan kan maximaal drie voorstellen in afwachting maken of herzien. De scan kan geen actieve Skill
toepassen, afwijzen, in quarantaine plaatsen of bewerken. Workshop toont de cumulatieve dekking,
bijvoorbeeld **20 sessies beoordeeld · 18 jun.–vandaag · 2 ideeën gevonden**. Selecteer
**Eerder werk scannen** om door te gaan vanaf de opgeslagen cursor van de oudste sessie. Nadat
de beschikbare geschiedenis volledig is verwerkt, verandert de actie in **Nieuw werk scannen**.

Historische beoordeling gebeurt handmatig, zelfs wanneer
`skills.workshop.autonomous.enabled` `false` is. Elke klik start een modeluitvoering,
dus de prijzen en voorwaarden voor gegevensverwerking van de provider zijn van toepassing. De cursor en dekkingsaantallen
worden opgeslagen in de gedeelde OpenClaw-statusdatabase; transcriptinhoud wordt niet gekopieerd
naar de scanstatus.

Als autonome vastlegging is ingeschakeld, kan OpenClaw ook een conservatieve beoordeling uitvoeren na succesvol,
substantieel werk en nadat het volledige agentsysteem inactief is geworden. Die geïsoleerde beoordeling kan maximaal
één voorstel in behandeling maken of herzien. Deze kan geen actieve skill bijwerken en een
voorstel niet toepassen, afwijzen of in quarantaine plaatsen, zelfs niet wanneer `approvalPolicy` `"auto"` is.

Zie [Zelflerend vermogen](/nl/tools/self-learning) voor details over inschakeling, geschiktheid, privacy en kosten,
de voorsteldrempel en probleemoplossing.

## Goedkeuring en autonomie

```json5
{
  skills: {
    workshop: {
      autonomous: {
        enabled: false,
      },
      allowSymlinkTargetWrites: false,
      approvalPolicy: "auto",
      maxPending: 50,
      maxSkillBytes: 40000,
    },
  },
}
```

| Instelling                  | Standaard | Effect                                                                                                                                                              |
| -------------------------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `autonomous.enabled`       | `false`  | Maakt voorstellen in behandeling op basis van expliciete correcties en, na een periode van inactiviteit, substantieel voltooid werk met herbruikbaar herstel of betekenisvolle besparingen op retourverkeer.   |
| `allowSymlinkTargetWrites` | `false`  | Maakt het mogelijk om bij toepassing te schrijven via symlinks van workspace-skills waarvan het werkelijke doel wordt vermeld in `skills.load.allowSymlinkTargets`.                                                 |
| `approvalPolicy`           | `"auto"` | `"auto"` slaat een extra vraag over voor door een agent geïnitieerde `apply`, `reject` of `quarantine` (de agent moet de actie nog steeds aanroepen). `"pending"` vereist goedkeuring. |
| `maxPending`               | `50`     | Beperkt het aantal voorstellen in behandeling en in quarantaine per workspace (1-200).                                                                                                       |
| `maxSkillBytes`            | `40000`  | Beperkt de grootte van de voorsteltekst in bytes (1024-200000).                                                                                                                     |

Autonome vastlegging herkent prospectieve regels (bijvoorbeeld „vanaf nu”) en reactieve
correcties (bijvoorbeeld „dat is niet wat ik vroeg”). Nieuwe instructies worden per onderwerp gegroepeerd in maximaal
drie voorstellen per beurt, overeenkomsten in woordgebruik worden doorgestuurd naar bestaande beschrijfbare workspace-skills en
het eigen voorstel in behandeling wordt herzien wanneer een andere correctie op dezelfde skill is gericht.

Voor succesvol substantieel werk zonder expliciete correctie bepaalt een geïsoleerde uitvoering van het geselecteerde
model of het voltooide traject aan de conservatieve voorsteldrempel voldoet. Het
voorgrondmodel wordt vóór het antwoorden niet gevraagd om te leren. De achtergrondbeoordelaar behoudt de
voorgronduitvoering als herkomst van het voorstel, heeft geen toegang tot algemene agenttools en kan geen beslissingen over de levenscyclus
nemen. De beoordeling begint alleen wanneer de voorgrondruntime zowel het exact opgeloste model meldt
als dat `skill_workshop` daadwerkelijk beschikbaar was. Een restrictief of onbekend toolbeleid
wordt daarom gesloten afgehandeld en maakt geen voorstel.

Zie [Zelflerend vermogen](/nl/tools/self-learning) voor het volledige autonome beoordelingsgedrag en
veiligheidsmodel.

Voorstelbeschrijvingen zijn altijd beperkt tot 160 bytes, onafhankelijk van
`maxSkillBytes`.

## Gateway-methoden

| Methode                            | Bereik           |
| ---------------------------------- | ---------------- |
| `skills.proposals.list`            | `operator.read`  |
| `skills.proposals.inspect`         | `operator.read`  |
| `skills.proposals.historyStatus`   | `operator.read`  |
| `skills.proposals.historyScan`     | `operator.admin` |
| `skills.proposals.create`          | `operator.admin` |
| `skills.proposals.update`          | `operator.admin` |
| `skills.proposals.revise`          | `operator.admin` |
| `skills.proposals.requestRevision` | `operator.admin` |
| `skills.proposals.apply`           | `operator.admin` |
| `skills.proposals.reject`          | `operator.admin` |
| `skills.proposals.quarantine`      | `operator.admin` |
| `skills.curator.status`            | `operator.read`  |
| `skills.curator.pin`               | `operator.admin` |
| `skills.curator.unpin`             | `operator.admin` |
| `skills.curator.restore`           | `operator.admin` |

`requestRevision` is alleen beschikbaar voor de Gateway (zonder equivalent in de CLI of agenttools): deze methode
stuurt revisie-instructies in vrije tekst door naar de chatsessie van de verantwoordelijke agent
in plaats van `PROPOSAL.md` rechtstreeks te vervangen, voor gebruikersinterfaces die de agent vragen om
een herziening in plaats van letterlijk nieuwe inhoud in te dienen.

`historyStatus` en `historyScan` zijn ondersteuningsmethoden voor de Control UI. `historyScan`
accepteert `direction: "older" | "newer"`; resultaten blijven altijd als
voorstellen in behandeling staan.

## Opslag

```text
<OPENCLAW_STATE_DIR>/skill-workshop/
  proposals.json
  proposals/<proposal-id>/
    proposal.json
    PROPOSAL.md
    rollback.json
    assets/
    examples/
    references/
    scripts/
    templates/
```

Standaardstatusmap: `~/.openclaw`.

- `proposal.json`: canonieke voorstelrecord.
- `proposals.json`: snelle lijstindex, opnieuw op te bouwen vanuit voorstelmappen.
- `PROPOSAL.md`: skillvoorstel in behandeling.
- `rollback.json`: herstelmetadata die wordt geschreven voordat toepassing actieve bestanden wijzigt.

## Limieten

| Limiet                          | Waarde                                                               |
| ------------------------------- | -------------------------------------------------------------------- |
| Beschrijving                    | 160 bytes                                                            |
| Voorsteltekst                   | `skills.workshop.maxSkillBytes` (standaard 40,000; absolute bovengrens 1 MiB) |
| Ondersteunende bestanden        | 64 per voorstel                                                      |
| Grootte ondersteunend bestand   | Elk 256 KiB, in totaal 2 MiB                                         |
| Voorstellen in behandeling + in quarantaine | `skills.workshop.maxPending` per workspace (standaard 50)              |

## Probleemoplossing

| Probleem                                       | Oplossing                                                                                                                                                                                                  |
| ---------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Skill proposal description is too large`      | Kort `description` in tot 160 bytes of minder.                                                                                                                                                                 |
| `Skill proposal content is too large`          | Kort de voorsteltekst in of verhoog `skills.workshop.maxSkillBytes`.                                                                                                                                         |
| `Target skill changed after proposal creation` | Herzie het voorstel ten opzichte van het huidige doel of maak een nieuw voorstel.                                                                                                                                   |
| `Proposal scan failed`                         | Inspecteer de bevindingen van de scanner en herzie het voorstel vervolgens of plaats het in quarantaine.                                                                                                                                           |
| `untrusted symlink target`                     | Configureer `skills.load.allowSymlinkTargets` en schakel `skills.workshop.allowSymlinkTargetWrites` alleen in voor bewust gedeelde hoofdmappen van skills.                                                                  |
| `Support file paths must be under one of...`   | Verplaats ondersteunende bestanden naar `assets/`, `examples/`, `references/`, `scripts/` of `templates/`.                                                                                                                |
| Voorstel verschijnt niet in de lijst           | Controleer de geselecteerde `--agent`-workspace en `OPENCLAW_STATE_DIR`.                                                                                                                                            |
| Agent kan `skill_workshop` niet aanroepen             | Controleer het actieve toolbeleid en de uitvoeringsmodus. `coding` bevat de tool; restrictieve `tools.allow`-beleidsregels moeten deze expliciet vermelden en uitvoeringen in een sandbox moeten een normale agentsessie aan de hostzijde of de CLI gebruiken. |

### Diagnose van toolbeleid

Wanneer autonome vastlegging is ingeschakeld, voert `openclaw doctor` de
`core/doctor/skill-workshop-tool-policy`-controle uit voor de standaardagent. Als het beleid
`skill_workshop` verbergt, vermeldt de waarschuwing de eerste uitsluitende configuratielaag en
de exacte wijziging aan `allow` of `alsoAllow` die nodig is. Oudere draaiboeken gebruiken mogelijk nog
`openclaw plugins inspect skill-workshop`; die opdracht legt nu uit dat Skill
Workshop is ingebouwd en toont indien van toepassing dezelfde beleidshint.

## Gerelateerd

- [Skills](/nl/tools/skills) voor laadvolgorde, prioriteit en zichtbaarheid
- [Zelflerend vermogen](/nl/tools/self-learning) voor conservatieve skillvoorstellen na een uitvoering
- [Skills maken](/nl/tools/creating-skills) voor de basisprincipes van handmatig geschreven `SKILL.md`
- [Skills-configuratie](/nl/tools/skills-config) voor het volledige `skills.workshop`-schema
- [Skills-CLI](/nl/cli/skills) voor `openclaw skills`-opdrachten
