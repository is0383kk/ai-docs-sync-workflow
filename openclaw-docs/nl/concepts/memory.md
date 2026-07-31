---
read_when:
    - Je wilt begrijpen hoe geheugen werkt
    - Je wilt weten welke geheugenbestanden je moet schrijven
summary: Hoe OpenClaw dingen tussen sessies onthoudt
title: Geheugenoverzicht
x-i18n:
    generated_at: "2026-07-27T05:48:39Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cdfd5276d6289a4ee38b5203eb5443312c4b040d4ea67abe4a9c579703136339
    source_path: concepts/memory.md
    workflow: 16
---

OpenClaw onthoudt dingen door gewone Markdown-bestanden naar de werkruimte van je agent te schrijven (standaard `~/.openclaw/workspace`). Het model onthoudt alleen wat op schijf wordt opgeslagen; er is geen verborgen status.

## Hoe het werkt

Je agent heeft drie geheugengerelateerde bestanden:

- **`MEMORY.md`** — langetermijngeheugen. Duurzame feiten, voorkeuren en beslissingen. Wordt aan het begin van een sessie geladen.
- **`memory/YYYY-MM-DD.md`** (of `memory/YYYY-MM-DD-<slug>.md`) — dagelijkse notities. Doorlopende context en observaties. De gedateerde notities van vandaag en gisteren worden automatisch geladen bij een kale `/new` of `/reset`; varianten met een slug, zoals die welke door de gebundelde session-memory-hook worden geschreven, worden naast het bestand met alleen de datum opgepikt.
- **`DREAMS.md`** (optioneel) — Droomdagboek en samenvattingen van dreaming-rondes voor menselijke beoordeling, inclusief onderbouwde historische backfill-vermeldingen.

<Tip>
Als je wilt dat je agent iets onthoudt, vraag je dat gewoon: "Onthoud dat ik de voorkeur geef aan TypeScript." De agent schrijft de notitie naar het juiste bestand.
</Tip>

## Wat waar hoort

`MEMORY.md` is de compacte, gecureerde laag: duurzame feiten, voorkeuren, permanente beslissingen en korte samenvattingen die aan het begin van een sessie beschikbaar moeten zijn. Het is geen onbewerkt transcript, dagelijks logboek of volledig archief.

`memory/YYYY-MM-DD.md`-bestanden vormen de werklaag: gedetailleerde dagelijkse notities, observaties, sessiesamenvattingen en onbewerkte context die later nog nuttig kunnen zijn. Deze worden geïndexeerd voor `memory_search` en `memory_get`, maar niet bij elke beurt in de bootstrap-prompt geïnjecteerd.

Na verloop van tijd destilleert de agent nuttig materiaal uit dagelijkse notities naar `MEMORY.md` en verwijdert die verouderde langetermijnvermeldingen. Gegenereerde werkruimte-instructies en de Heartbeat-flow doen dit periodiek; je hoeft `MEMORY.md` niet voor elk detail handmatig te bewerken.

Als `MEMORY.md` het budget voor bootstrap-bestanden overschrijdt, houdt OpenClaw het bestand op schijf intact, maar kort het de kopie in die in de context wordt geïnjecteerd. Beschouw dit als een signaal om gedetailleerd materiaal naar `memory/*.md` te verplaatsen, alleen een duurzame samenvatting in `MEMORY.md` te bewaren of de bootstrap-limieten te verhogen als je meer promptbudget wilt besteden. Gebruik `/context list`, `/context detail` of `openclaw doctor` om de onbewerkte en geïnjecteerde grootten en de afkappingsstatus te bekijken.

## Importeren uit codeerassistenten

De Control UI kan bestaand lokaal geheugen uit Codex en Claude Code importeren. Open **Settings** → **Import Memory**, kies de doelagent, controleer de gedetecteerde bestanden en bevestig de import. OpenClaw kopieert alleen Markdown-geheugen:

- Codex: de samengevoegde `MEMORY.md`- en `memory_summary.md`-bestanden onder `~/.codex/memories` (of `CODEX_HOME/memories`). Onbewerkte rollout- en transcriptbestanden worden niet geïmporteerd.
- Claude Code: Markdown-bestanden uit de automatische geheugenmap van elk project onder `~/.claude/projects/*/memory`, plus een door de gebruiker geconfigureerde `autoMemoryDirectory` indien aanwezig. Projectinstructies, sessies, instellingen en inloggegevens maken geen deel uit van deze actie die uitsluitend het geheugen betreft.

Geïmporteerde bestanden blijven afzonderlijk onder `memory/imports/codex/` en `memory/imports/claude-code/` in de geselecteerde agentwerkruimte. Ze worden geïndexeerd voor `memory_search` en zijn beschikbaar via `memory_get`; ze worden niet samengevoegd met de bootstrap-`MEMORY.md` van de agent. De bronbestanden blijven ongewijzigd.

In het voorbeeld worden conflicten op de bestemming gemarkeerd. Schakel **Replace existing imports** in om die bestanden te vervangen; bij toepassen wordt een geverifieerde back-up van vóór de import gemaakt en worden kopieën op itemniveau van overschreven bestanden in het migratierapport bewaard.

## Actiegevoelige herinneringen

De meeste herinneringen zijn gewone Markdown-notities. Sommige beïnvloeden wat de agent later moet doen; leg daarvoor vast wanneer het veilig is om op basis van de notitie te handelen, niet alleen het feit zelf.

Leg die actiegrens vast wanneer een notitie betrekking heeft op:

- vereisten voor goedkeuring of toestemming,
- tijdelijke beperkingen,
- overdrachten naar een andere sessie, thread of persoon,
- vervalvoorwaarden,
- het moment waarop veilig kan worden gehandeld,
- de bevoegdheid van de bron of eigenaar,
- instructies om een verleidelijke handeling te vermijden.

Een nuttige actiegevoelige herinnering maakt duidelijk:

- wat toekomstig gedrag verandert,
- wanneer of onder welke voorwaarde die van toepassing is,
- wanneer die vervalt of wat handelen mogelijk maakt,
- wat de agent niet moet doen,
- wie de bron of eigenaar is, als dit van invloed is op vertrouwen of bevoegdheid.

Het geheugen kan goedkeuringscontext bewaren, maar dwingt geen beleid af. Gebruik de goedkeuringsinstellingen, sandboxing en geplande taken van OpenClaw voor harde operationele controles.

Voorbeeld:

```md
De API-migratie wordt in een andere sessie ontworpen. In toekomstige beurten
mag de API-implementatie niet vanuit deze thread worden bewerkt; gebruik de
bevindingen hier alleen als ontwerpinput totdat het migratieplan is vastgesteld.
```

Nog een voorbeeld:

```md
Een rapport van een niet-vertrouwde bron moet worden beoordeeld voordat het
wordt bevorderd. Behandel het in toekomstige beurten alleen als bewijs; sla
het niet als duurzaam geheugen op totdat een vertrouwde beoordelaar de inhoud
bevestigt.
```

Dit is geen vereist schema voor elke herinnering; eenvoudige feiten kunnen beknopt blijven. Gebruik actiegevoelige grenzen wanneer het verlies van context over timing, bevoegdheid, vervaldatum of veilig handelen ertoe kan leiden dat de agent later het verkeerde doet.

Gebruik [geplande taken](/nl/automation/cron-jobs) voor exacte herinneringen, tijdgebonden controles en terugkerend werk. Het geheugen kan nog steeds de duurzame context rond dat werk samenvatten.

## Beëindigde afgeleide toezeggingen

Sommige toekomstige vervolgacties zijn geen duurzame feiten. Als je een sollicitatiegesprek morgen noemt, kan de nuttige herinnering "vraag na het sollicitatiegesprek hoe het ging" zijn, niet "bewaar dit voor altijd in `MEMORY.md`."

Het experiment met afgeleide toezeggingen is beëindigd. OpenClaw extraheert of levert die vervolgacties niet langer. Gebruik [geplande taken](/nl/automation/cron-jobs) voor toekomstige acties; de verouderde opdracht `openclaw commitments` blijft beschikbaar om bestaande opgeslagen rijen te inspecteren of te verwijderen.

## Geheugenhulpmiddelen

De agent heeft twee hulpmiddelen om met geheugen te werken:

- **`memory_search`** — vindt relevante notities met semantisch zoeken, zelfs wanneer de formulering afwijkt van het origineel.
- **`memory_get`** — leest een specifiek geheugenbestand of regelbereik.

Beide hulpmiddelen worden geleverd door de actieve geheugenplugin (standaard: `memory-core`).

## Zoeken in het geheugen

Wanneer een embeddingprovider is geconfigureerd, gebruikt `memory_search` hybride zoeken: vectorovereenkomst (semantische betekenis) gecombineerd met trefwoordovereenkomst (exacte termen zoals ID's en codesymbolen). Dit werkt direct met een API-sleutel voor elke ondersteunde provider.

<Info>
OpenClaw gebruikt standaard OpenAI-embeddings. Stel `memory.search.provider` expliciet in om Gemini, Voyage, Mistral, Bedrock, DeepInfra, lokale GGUF, Ollama, LM Studio, GitHub Copilot of een generiek OpenAI-compatibel eindpunt te gebruiken.
</Info>

Zie [Zoeken in het geheugen](/nl/concepts/memory-search) voor de werking van zoeken, afstemmingsopties en providerconfiguratie.

## Geheugenbackends

<CardGroup cols={3}>
<Card title="Ingebouwd (standaard)" icon="database" href="/nl/concepts/memory-builtin">
Gebaseerd op SQLite. Werkt direct met zoeken op trefwoorden, vectorovereenkomst en hybride zoeken. Geen extra afhankelijkheden.
</Card>
<Card title="QMD" icon="search" href="/nl/concepts/memory-qmd">
Local-first-sidecar met herrangschikking, query-uitbreiding en de mogelijkheid om mappen buiten de werkruimte te indexeren.
</Card>
<Card title="Honcho" icon="brain" href="/nl/concepts/memory-honcho">
AI-native geheugen voor meerdere sessies met gebruikersmodellering, semantisch zoeken en bewustzijn van meerdere agents. Plugin-installatie.
</Card>
<Card title="LanceDB" icon="layers" href="/nl/plugins/memory-lancedb">
Op LanceDB gebaseerd geheugen met OpenAI-compatibele embeddings, automatisch terughalen, automatisch vastleggen en ondersteuning voor lokale Ollama-embeddings. Plugin-installatie.
</Card>
</CardGroup>

## Kenniswikilaag

Als je wilt dat duurzaam geheugen zich meer als een onderhouden kennisbank gedraagt dan als onbewerkte notities, gebruik je de gebundelde Plugin `memory-wiki`. Deze compileert duurzame kennis naar een wikikluis met een deterministische paginastructuur, gestructureerde beweringen en bewijs, bijhouden van tegenstrijdigheden en actualiteit, gegenereerde dashboards, gecompileerde samenvattingen en wikispecifieke hulpmiddelen (`wiki_status`, `wiki_search`, `wiki_get`, `wiki_apply`, `wiki_lint`).

`memory-wiki` vervangt de actieve geheugenplugin niet; de actieve geheugenplugin blijft verantwoordelijk voor terughalen, bevordering en dreaming. `memory-wiki` voegt er een kennislaag met uitgebreide herkomstinformatie naast toe.

<CardGroup cols={1}>
<Card title="Geheugenwiki" icon="book" href="/nl/plugins/memory-wiki">
Compileert duurzaam geheugen naar een wikikluis met uitgebreide herkomstinformatie, beweringen, dashboards, bridge-modus en Obsidian-vriendelijke workflows.
</Card>
</CardGroup>

## Automatisch geheugen wegschrijven

Voordat [Compaction](/nl/concepts/compaction) je gesprek samenvat, voert OpenClaw een stille beurt uit die de agent eraan herinnert belangrijke context in geheugenbestanden op te slaan. Dit is standaard ingeschakeld; stel `agents.defaults.compaction.memoryFlush.enabled: false` in om het uit te schakelen.

Om die onderhoudsbeurt op een lokaal model uit te voeren, stel je een exacte override in die alleen van toepassing is op de beurt voor het wegschrijven van het geheugen (deze neemt de fallback-keten van het model van de actieve sessie niet over):

```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "memoryFlush": {
          "model": "ollama/qwen3:8b"
        }
      }
    }
  }
}
```

<Tip>
Het wegschrijven van het geheugen voorkomt contextverlies tijdens Compaction. Als je agent belangrijke feiten in het gesprek heeft die nog niet naar een bestand zijn geschreven, worden ze automatisch opgeslagen voordat de samenvatting plaatsvindt.
</Tip>

## Dreaming

Dreaming is een optionele consolidatieronde voor het geheugen die op de achtergrond wordt uitgevoerd. Deze verzamelt signalen voor het terughalen van kortetermijninformatie, kent kandidaten scores toe en bevordert alleen gekwalificeerde items naar het langetermijngeheugen (`MEMORY.md`):

- **Opt-in**: standaard uitgeschakeld.
- **Gepland**: wanneer ingeschakeld, beheert `memory-core` automatisch één terugkerende Cron-taak voor een volledige dreaming-ronde.
- **Met drempelwaarden**: bevorderingen moeten voldoen aan poorten voor score, terughaalfrequentie en querydiversiteit.
- **Beoordeelbaar**: fasesamenvattingen en dagboekvermeldingen worden naar `DREAMS.md` geschreven voor menselijke beoordeling.

Zie [Dreaming](/nl/concepts/dreaming) voor fasegedrag, scoringssignalen en details over het Droomdagboek.

## Onderbouwde backfill en livebevordering

Het dreamingsysteem heeft twee gerelateerde beoordelingstrajecten:

- **Live dreaming** werkt vanuit de kortetermijnopslag voor dreaming onder `memory/.dreams/` en wordt door de normale diepe fase gebruikt om te bepalen wat naar `MEMORY.md` promoveert.
- **Onderbouwde backfill** leest historische `memory/YYYY-MM-DD.md`-notities als afzonderlijke dagbestanden en schrijft gestructureerde beoordelingsuitvoer naar `DREAMS.md`.

Onderbouwde backfill is nuttig om oudere notities opnieuw af te spelen en te inspecteren wat het systeem als duurzaam beschouwt, zonder `MEMORY.md` handmatig te bewerken.

```bash
openclaw memory rem-backfill --path ./memory --stage-short-term
```

De vlag `--stage-short-term` plaatst onderbouwde duurzame kandidaten in dezelfde kortetermijnopslag voor dreaming die de normale diepe fase al gebruikt; de kandidaten worden niet rechtstreeks bevorderd. Dus:

- `DREAMS.md` blijft het oppervlak voor menselijke beoordeling.
- De kortetermijnopslag blijft het machinegerichte oppervlak voor rangschikking.
- `MEMORY.md` wordt nog steeds alleen door diepe bevordering geschreven.

Een herhaling ongedaan maken zonder gewone dagboekvermeldingen of de normale terughaalstatus te wijzigen:

```bash
openclaw memory rem-backfill --rollback
openclaw memory rem-backfill --rollback-short-term
```

## CLI

```bash
openclaw memory status          # Indexstatus en provider controleren
openclaw memory search "query"  # Zoeken vanaf de opdrachtregel
openclaw memory index --force   # De index opnieuw opbouwen
```

## Verder lezen

- [Geheugen doorzoeken](/nl/concepts/memory-search): zoekpijplijn, providers en afstemming.
- [Ingebouwde geheugenengine](/nl/concepts/memory-builtin): standaard SQLite-backend.
- [QMD-geheugenengine](/nl/concepts/memory-qmd): geavanceerde, local-first sidecar.
- [Honcho-geheugen](/nl/concepts/memory-honcho): AI-native geheugen voor meerdere sessies.
- [Memory LanceDB](/nl/plugins/memory-lancedb): door LanceDB ondersteunde Plugin met OpenAI-compatibele embeddings.
- [Memory Wiki](/nl/plugins/memory-wiki): gecompileerde kennisopslag en wiki-native tools.
- [Dreaming](/nl/concepts/dreaming): achtergrondpromotie van kortetermijnherinneringen naar het langetermijngeheugen.
- [Referentie voor geheugenconfiguratie](/nl/reference/memory-config): alle configuratieopties.
- [Compaction](/nl/concepts/compaction): hoe Compaction samenwerkt met het geheugen.
- [Active Memory](/nl/concepts/active-memory): geheugen van subagents voor interactieve chatsessies.
