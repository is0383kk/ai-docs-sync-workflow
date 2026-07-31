---
read_when:
    - Je wilt dat OpenClaw herbruikbare procedures leert uit voltooide gesprekken
    - Je beslist of je autonome voorstellen voor Skills wilt inschakelen
    - Je moet inzicht krijgen in de veiligheid, kosten, geschiktheid of probleemoplossing van zelflerende systemen
sidebarTitle: Self-learning
summary: Laat OpenClaw herbruikbare Skills voorstellen op basis van correcties en substantieel voltooid werk
title: Zelflerend
x-i18n:
    generated_at: "2026-07-27T06:16:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b10618c1a64441bdf0ba58f03e02972bdf2b1d59643a78358910594f8139ccb8
    source_path: tools/self-learning.md
    workflow: 16
---

Zelflerend vermogen stelt OpenClaw in staat om nuttig bewijs uit gesprekken om te zetten in openstaande
voorstellen voor [Skill Workshop](/nl/tools/skill-workshop). Het traint geen modelgewichten,
bewerkt geen actieve skills en wijzigt het gedrag van agents niet stilzwijgend. Elke geleerde
procedure blijft openstaan totdat een operator deze beoordeelt en toepast.

Zelflerend vermogen is **standaard uitgeschakeld**. Schakel het alleen in wanneer een extra
modeluitvoering op de achtergrond en beoordeling van transcripties geschikt zijn voor jouw werkruimte.

## Zelflerend vermogen inschakelen

Open in de Control UI **Plugins → Workshop** en schakel **Zelflerend vermogen** in. De
wijziging wordt onmiddellijk van kracht. Wanneer een andere configuratieschrijver het
bestand heeft bijgewerkt, vernieuwt de Control UI de configuratiesnapshot en probeert deze de schakelaar opnieuw
zonder de pagina of Gateway opnieuw te laden.

Gebruik de CLI:

```bash
openclaw config set skills.workshop.autonomous.enabled true --strict-json
```

Of bewerk `~/.openclaw/openclaw.json`:

```json5
{
  skills: {
    workshop: {
      autonomous: {
        enabled: true,
      },
    },
  },
}
```

Schakel het weer uit met:

```bash
openclaw config set skills.workshop.autonomous.enabled false --strict-json
```

Door de gebruiker aangevraagde creatie van skills, `/learn` en handmatige bewerkingen in Skill Workshop
blijven werken wanneer zelflerend vermogen is uitgeschakeld.

## Eerdere sessies handmatig beoordelen

Handmatige beoordeling van de geschiedenis is het behoudende alternatief voor autonome vastlegging.
Open **Plugins → Workshop** in de Control UI en selecteer **Skill-ideeën zoeken**.
Dit wijzigt `skills.workshop.autonomous.enabled` niet.

Elke scan:

- begint met de nieuwste nog niet beoordeelde sessies en werkt terug;
- beoordeelt maximaal 20 substantiële sessies met minstens zes modelbeurten;
- slaat Cron-, Heartbeat-, hook-, subagent-, ACP-, Plugin-eigen en interne beoordelingssessies
  over;
- redigeert herkende geheimen en begrenst de transcriptiebundel voordat deze
  naar het geconfigureerde model van de geselecteerde agent wordt verzonden;
- hanteert dezelfde hoge drempel als autonome ervaringsbeoordeling; en
- kan maximaal drie openstaande voorstellen maken of herzien, maar nooit actieve skills.

De Workshop rapporteert het cumulatieve aantal sessies, het datumbereik en de gevonden ideeën.
Selecteer **Eerder werk scannen** voor het volgende oudere venster. Wanneer de cursor
het begin van de in aanmerking komende geschiedenis bereikt, verandert de actie in **Nieuw werk scannen**.
OpenClaw bewaart alleen cursor- en dekkingsmetadata in de gedeelde statusdatabase;
het maakt geen tweede transcriptiearchief.

Sessies worden alleen gescand wanneer OpenClaw het eigenaarschap ervan kan aantonen en
inhoud van externe hooks kan uitsluiten. Na een upgrade kan de huidige transcriptie van vóór de upgrade
lokaal worden geclassificeerd, maar geroteerde transcripties van vóór de upgrade zonder herkomst
per uitvoering worden overgeslagen. Nieuwe transcripties behouden deze herkomst tijdens rotatie.

Handmatige scans brengen nog steeds kosten van de modelprovider met zich mee en sturen in aanmerking komende gespreksinhoud
naar de geconfigureerde provider. Gebruik ze alleen wanneer die beoordeling voldoet aan de
privacy- en gegevensverwerkingsvereisten van de werkruimte.

## Wat OpenClaw kan leren

Zelflerend vermogen heeft twee behoudende paden:

1. **Directe instructies en correcties.** OpenClaw detecteert duurzame formuleringen
   zoals ‘vanaf nu’, ‘de volgende keer’ en correcties op een mislukte aanpak.
   Wanneer zelflerend vermogen is ingeschakeld, kan het die signalen omzetten in openstaande voorstellen
   zonder op een nieuwe prompt te wachten. Dit deterministische pad kan gerelateerde
   instructies groeperen in maximaal drie voorstellen, een beschrijfbare werkruimteskill als doel kiezen
   of een eigen gerelateerd openstaand voorstel herzien. Het wordt ook uitgevoerd na mislukte beurten,
   omdat het de instructies van de gebruiker vastlegt in plaats van de voltooiing te beoordelen.
2. **Ervaringsbeoordeling.** Na een geslaagde, substantiële voorgrondbeurt
   kan OpenClaw het voltooide werk beoordelen op een herbruikbare hersteltechniek of
   een stabiele procedure die minstens twee toekomstige model- of toolrondreizen
   zou voorkomen.

Goede kandidaten zijn onder meer:

- een betrouwbaar herstel na herhaalde tool- of modelfouten;
- een niet voor de hand liggende volgordebeperking die een terugkerende fout voorkwam;
- een stabiele workflow met meerdere stappen waarvoor herhaald onderzoek nodig was; of
- een herbruikbare voorafgaande controle die meerdere toekomstige aanroepen zou voorkomen.

De beoordelaar moet zich onthouden bij routinematig geslaagd werk, eenmalige verzoeken,
persoonlijke feiten, eenvoudige voorkeuren, tijdelijke omgevingsfouten, algemeen
advies, niet-onderbouwde negatieve beweringen en geheimen.

## Wanneer ervaringsbeoordeling wordt uitgevoerd

Ervaringsbeoordeling wordt bewust vertraagd en begrensd:

- De voorgrondbeurt moet succesvol worden voltooid.
- De huidige beurt moet minstens tien modeliteraties bevatten.
- Cron-, Heartbeat-, geheugen-, overflow-, hook-, subagent- en beoordelingssessies zijn
  uitgesloten.
- De voorgronduitvoering moet een provider en model hebben bepaald en daadwerkelijk
  toegang hebben gehad tot `skill_workshop`.
- OpenClaw wacht 30 seconden na voltooiing. Een latere voltooiing op de voorgrond in
  dezelfde sessie start die rustperiode opnieuw.
- Als een agent- of antwoorduitvoering nog actief is, wacht de beoordeling nog eens 30 seconden.
- Er wordt slechts één ervaringsbeoordeling tegelijk uitgevoerd.
- Uitgestelde beoordeling is proceslokaal Gateway-werk. De Gateway moet actief blijven
  tijdens het inactiviteitsvenster; eenmalige lokale en door de CLI ondersteunde runtimes behouden
  onvoldoende traject- en toolbeschikbaarheidscontext om deze in te plannen.

Het voorgrondantwoord wordt nooit vertraagd om te leren. Een mislukte of niet in aanmerking komende
beurt start geen ervaringsbeoordeling, hoewel directe correcties van de gebruiker
nog steeds als suggestie kunnen worden aangeboden wanneer autonomie is uitgeschakeld.

## Wat de beoordelaar ontvangt

De beoordelaar op de achtergrond ontvangt alleen de huidige beurt, vanaf het meest
recente bericht van de gebruiker. Het weergegeven traject is beperkt tot 60.000 tekens;
waar nodig behoudt OpenClaw het eerste bericht en het nieuwste bewijs en
markeert het weggelaten middenstuk.

De beoordelaar hergebruikt de bepaalde provider en het bepaalde model. Deze hergebruikt het
authenticatieprofiel van de voorgrond wanneer die identiteit beschikbaar is en schakelt modelfallbacks uit. De
beoordeling start daarom een extra modeluitvoering bij de geconfigureerde provider.
Die uitvoering kan meer dan één providerverzoek doen wanneer een voorstel wordt geïnspecteerd of opgesteld.
De prijs- en gegevensverwerkingsvoorwaarden van de provider gelden op dezelfde manier als voor de
voorgrondbeurt.

Voordat OpenClaw begint, laadt het de huidige runtimeconfiguratie opnieuw en controleert het opnieuw het
effectieve sandbox- en toolbeleid voor het oorspronkelijke gesprek. Als de uitvoering zich
in een sandbox bevindt, het beleid `skill_workshop` niet langer toestaat of vereiste runtimefeiten
ontbreken, wordt de beoordeling gesloten bij fouten en wordt er niets gemaakt.

<Warning>
  Het inschakelen van zelflerend vermogen staat toe dat in aanmerking komende gespreksinhoud, waaronder toolinvoer
  en resultaten van de huidige beurt, voor één extra beoordeling naar de geselecteerde
  modelprovider wordt verzonden. Schakel dit niet in een werkruimte in waar
  die beoordeling de vereisten voor gegevensverwerking zou schenden.
</Warning>

## Veiligheid van voorstellen

De beoordelaar wordt uitgevoerd in een geïsoleerde sessie met een bewust beperkt
tooloppervlak:

- Deze kan alleen Workshop-voorstellen weergeven of inspecteren en één
  openstaand voorstel maken of herzien.
- Deze kan geen actieve skill bijwerken, een voorstel toepassen, een voorstel afwijzen, een voorstel
  in quarantaine plaatsen, een bericht verzenden of algemene agenttools gebruiken.
- Eén mutatiebudget wordt gedeeld over modelherhalingen, zodat een beoordeling maximaal
  één voorstel kan maken of herzien.
- Het beoordeelde traject wordt behandeld als niet-vertrouwd bewijs, niet als instructies
  voor de agent op de achtergrond.
- Skill Workshop scant de inhoud van voorstellen en weigert herkende letterlijke
  referenties voordat de voorstelstatus wordt opgeslagen.

De normale Workshop-limieten blijven van toepassing, waaronder `maxPending`, `maxSkillBytes`,
beperkingen voor ondersteuningsbestanden, scannercontroles en schrijfbewerkingen die beperkt zijn tot de werkruimte. De
instelling `approvalPolicy: "auto"` geeft de beoordelaar op de achtergrond geen toegang
tot levenscyclusacties.

## Geleerde voorstellen beoordelen

Zelflerend vermogen produceert dezelfde openstaande voorstellen als handmatig gebruik van Workshop.
Inspecteer ze voordat je ze toepast:

```bash
openclaw skills workshop list
openclaw skills workshop inspect <proposal-id>
openclaw skills workshop apply <proposal-id>
```

Herzie, weiger of plaats voorstellen in quarantaine die nuttig maar nog niet gereed zijn:

```bash
openclaw skills workshop revise <proposal-id> --proposal ./PROPOSAL.md
openclaw skills workshop reject <proposal-id> --reason "Te specifiek"
openclaw skills workshop quarantine <proposal-id> --reason "Beveiligingsbeoordeling vereist"
```

Toepassen is de enige bewerking die een actieve `SKILL.md` schrijft. Zie
[Skill Workshop](/nl/tools/skill-workshop) voor het volledige levenscyclus- en opslagmodel.

## Configuratie

| Instelling                                  | Standaard | Effect van zelflerend vermogen                                                                                                    |
| ------------------------------------------ | -------- | --------------------------------------------------------------------------------------------------------------------------------- |
| `skills.workshop.autonomous.enabled`       | `false`  | Schakelt het vastleggen van directe correcties en uitgestelde ervaringsbeoordeling in.                                              |
| `skills.workshop.approvalPolicy`           | `"auto"` | Bepaalt goedkeuringsprompts voor normale, door agents geïnitieerde levenscyclusacties; dit breidt de machtigingen van de beoordelaar op de achtergrond niet uit. |
| `skills.workshop.maxPending`               | `50`     | Beperkt het aantal openstaande en in quarantaine geplaatste voorstellen per werkruimte.                                           |
| `skills.workshop.maxSkillBytes`            | `40000`  | Beperkt de grootte van de voorsteltekst in bytes.                                                                                 |
| `skills.workshop.allowSymlinkTargetWrites` | `false`  | Heeft alleen invloed op het toepassen; zelflerend vermogen schrijft zelf voorstelstatus, geen actieve skilldoelen.                 |

Zie voor het volledige schema, bereiken en gerelateerde skillinstellingen
[Skills-configuratie](/nl/tools/skills-config#workshop-skills-workshop).

## Problemen oplossen

### Er verschijnt geen voorstel na een lange beurt

Controleer al het volgende:

1. `skills.workshop.autonomous.enabled` is `true` in de actieve Gateway-configuratie.
2. De beurt is geslaagd en bevatte minstens tien modeliteraties na het meest
   recente bericht van de gebruiker.
3. Het gesprek was een normale voorgronduitvoering, geen geplande, geheugen-,
   hook- of subagentuitvoering.
4. De oorspronkelijke uitvoering had toegang tot `skill_workshop` en bevond zich niet in een sandbox.
5. Het systeem bleef lang genoeg inactief voor de uitgestelde beoordeling.
6. Het langlopende Gateway-proces bleef actief tijdens het inactiviteitsvenster; een
   eenmalige lokale opdracht wacht niet op uitgestelde beoordeling.

Een in aanmerking komende beoordeling levert mogelijk nog steeds geen voorstel op. Onthouding is het verwachte
resultaat wanneer het bewijs niet voldoet aan de drempel voor een herbruikbare procedure.

### Doctor meldt dat de Workshop-tool verborgen is

Wanneer zelflerend vermogen is ingeschakeld, controleert `openclaw doctor` of het effectieve
toolbeleid van de standaardagent `skill_workshop` toestaat. Volg de gemelde wijziging van
`tools.allow` of `tools.alsoAllow`, of schakel zelflerend vermogen uit.

### Er verschijnen te veel voorstellen met weinig waarde

Schakel zelflerend vermogen uit en blijf `/learn` of expliciete Workshop-verzoeken gebruiken:

```bash
openclaw config set skills.workshop.autonomous.enabled false --strict-json
```

Openstaande voorstellen blijven beoordeelbaar nadat de functie is uitgeschakeld. Het uitschakelen
van zelflerend vermogen past ze niet toe, wijst ze niet af en verwijdert ze niet.

## Gerelateerd

- [Skillworkshop](/nl/tools/skill-workshop) voor beoordeling, goedkeuring en
  opslag van voorstellen
- [Skills maken](/nl/tools/creating-skills) voor handmatig gemaakte skills en
  de structuur van `SKILL.md`
- [Skills-configuratie](/nl/tools/skills-config) voor alle instellingen van `skills.*`
- [Skills-CLI](/nl/cli/skills) voor Workshop- en curatoropdrachten
