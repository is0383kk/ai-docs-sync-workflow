---
read_when:
    - Je wilt QMD instellen als je geheugenbackend
    - Je wilt geavanceerde geheugenfuncties, zoals herrangschikking of extra geïndexeerde paden
summary: Local-first zoeksidecar met BM25, vectoren, herrangschikking en query-uitbreiding
title: QMD-geheugenengine
x-i18n:
    generated_at: "2026-07-27T05:31:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c0e54dc9a18d834036e4c79d6b7bdecb268a29976d9f30ea6e82a56ca5d71fda
    source_path: concepts/memory-qmd.md
    workflow: 16
---

[QMD](https://github.com/tobi/qmd) is een local-first zoeksidecar die naast
OpenClaw draait. Deze combineert BM25, vectorzoekopdrachten en herrangschikking in één
binair bestand en kan inhoud buiten de geheugenbestanden van je werkruimte indexeren.

## Wat het toevoegt ten opzichte van de ingebouwde engine

- **Herrangschikking en query-uitbreiding** voor een betere recall.
- **Extra mappen indexeren** - projectdocumentatie, teamnotities, alles op schijf.
- **Sessietranscripten indexeren** - eerdere gesprekken terugvinden.
- **Volledig lokaal** - werkt met de officiële llama.cpp-providerplugin en
  downloadt automatisch GGUF-modellen.
- **Automatische terugval** - als QMD niet beschikbaar is, valt OpenClaw naadloos
  terug op de ingebouwde engine.

## Aan de slag

### Vereisten

- Installeer QMD: `npm install -g @tobilu/qmd` of `bun install -g @tobilu/qmd`
- Een SQLite-build die extensies toestaat (`brew install sqlite` op macOS).
- QMD moet in de `PATH` van de Gateway staan.
- macOS en Linux werken direct. Windows wordt het best ondersteund via WSL2.

### Inschakelen

```json5
{
  memory: {
    backend: "qmd",
  },
}
```

OpenClaw maakt een zelfstandige QMD-home aan onder
`~/.openclaw/agents/<agentId>/qmd/` en beheert de levenscyclus van de sidecar
automatisch - verzamelingen, updates en embedding-runs worden voor je afgehandeld.
Het geeft de voorkeur aan de huidige QMD-vormen voor verzamelingen en MCP-query's, maar valt indien nodig terug op
alternatieve flags voor verzamelingspatronen en oudere namen van MCP-tools.
De reconciliatie bij het opstarten maakt verouderde beheerde verzamelingen ook opnieuw aan met hun
canonieke patronen wanneer er nog een oudere QMD-verzameling met dezelfde naam
aanwezig is.

## Hoe de sidecar werkt

- OpenClaw maakt verzamelingen op basis van geheugenbestanden in de werkruimte en geconfigureerde
  `memory.qmd.paths`. De QMD-adapter beheert heuristieken voor updates, embeddings, debounce en
  time-outs; deze zijn niet door de gebruiker configureerbaar.
- QMD blijft eigenaar van zijn `index.sqlite`, YAML-configuratie voor verzamelingen en modeldownloads
  onder de QMD-home per agent; dit zijn artefacten van een externe tool,
  geen OpenClaw-statustabellen. Coördinatie die eigendom is van OpenClaw bevindt zich uitsluitend in SQLite:
  één gedeelde lease beperkt embeddingwerk voor alle agents, terwijl één lease in elke
  agentdatabase de schrijfoperaties voor verzamelingen, updates en embeddings van die agent serialiseert.
  De runtime maakt geen sidecars voor QMD-bestandsvergrendeling meer aan. `openclaw doctor --fix`
  verwijdert buiten gebruik gestelde sidecars pas nadat is vastgesteld dat hun oude proceseigenaar niet meer actief is.
  Upgrades zijn een volledige omschakeling: stop en herstart elk OpenClaw-proces dat
  de statusmap deelt voordat je de nieuwe versie gebruikt. Gemengde oude/nieuwe QMD-
  writers worden niet ondersteund; de runtime past bewust geen dubbele vergrendeling toe op de buiten gebruik gestelde
  sidecars.
- De standaardverzameling voor de werkruimte volgt `MEMORY.md` plus de `memory/`-
  boom. `memory.md` in kleine letters wordt niet geïndexeerd als hoofdgeheugenbestand.
- De eigen scanner van QMD negeert verborgen paden en gangbare mappen voor afhankelijkheden/builds,
  zoals `.git`, `.cache`, `node_modules`, `vendor`, `dist` en
  `build`. Bij het opstarten van de Gateway blijft QMD lui; de manager wordt geïnitialiseerd wanneer het geheugen
  voor het eerst wordt gebruikt.
- Zoekopdrachten gebruiken de geconfigureerde `searchMode` (standaard: `search`; ondersteunt ook
  `vsearch` en `query`). `search` gebruikt alleen BM25, dus OpenClaw slaat in die modus
  gereedheidscontroles voor semantische vectoren en onderhoud van embeddings over. Als een modus
  mislukt, probeert OpenClaw het opnieuw met `qmd query`.
- Wanneer `searchMode` is ingesteld op `query`, stel je `memory.qmd.rerank` in op `false` om
  het hybride querypad van QMD zonder de reranker te gebruiken (vereist QMD 2.1 of nieuwer).
  OpenClaw geeft `--no-rerank` door aan het directe QMD-CLI-pad en
  `rerank: false` aan de MCP-querytool van QMD.
- Met QMD-releases die filters voor meerdere verzamelingen aanbieden, groepeert OpenClaw
  verzamelingen met dezelfde bron in één QMD-zoekaanroep. Oudere QMD-releases
  behouden de compatibele terugval per verzameling.
- Als QMD volledig uitvalt, valt OpenClaw terug op de ingebouwde SQLite-engine.
  Herhaalde pogingen tijdens chatbeurten nemen na een fout bij het openen kort gas terug, zodat een
  ontbrekend binair bestand of defecte sidecarafhankelijkheid geen storm van nieuwe pogingen veroorzaakt;
  `openclaw memory status` en eenmalige CLI-controles controleren QMD nog steeds
  rechtstreeks opnieuw.

<Info>
De eerste zoekopdracht kan traag zijn - QMD downloadt bij de eerste
`qmd query`-run automatisch GGUF-modellen (~2 GB) voor herrangschikking en query-uitbreiding.
</Info>

## Zoekprestaties en compatibiliteit

OpenClaw houdt het QMD-zoekpad compatibel met zowel huidige als oudere QMD-
installaties.

Bij het opstarten controleert OpenClaw eenmaal per manager de helptekst van de geïnstalleerde QMD. Als
het binaire bestand ondersteuning voor meerdere verzamelingsfilters aangeeft, doorzoekt OpenClaw
alle verzamelingen met dezelfde bron met één opdracht:

```bash
qmd search "router notes" --json -n 10 -c memory-root-main -c memory-dir-main
```

Zo hoeft er niet voor elke duurzame geheugenverzameling een afzonderlijk QMD-subproces te worden gestart.
Verzamelingen met sessietranscripten blijven in hun eigen brongroep, zodat gemengde
zoekopdrachten voor `memory` + `sessions` de resultaatdiversificator nog steeds invoer uit
beide bronnen geven.

Oudere QMD-builds accepteren slechts één verzamelingsfilter. Wanneer OpenClaw een
van deze builds detecteert, behoudt het het compatibiliteitspad en doorzoekt het elke verzameling
afzonderlijk voordat de resultaten worden samengevoegd en gededupliceerd.

Voer het volgende uit om het geïnstalleerde contract handmatig te inspecteren:

```bash
qmd --help | grep -i collection
```

De huidige QMD-help vermeldt dat één of meer verzamelingen kunnen worden geselecteerd. Oudere help
beschrijft meestal één verzameling.

## Modeloverschrijvingen

Omgevingsvariabelen voor QMD-modellen worden ongewijzigd doorgegeven vanuit het Gateway-
proces, zodat je QMD globaal kunt afstemmen zonder nieuwe OpenClaw-configuratie toe te voegen:

```bash
export QMD_EMBED_MODEL="hf:Qwen/Qwen3-Embedding-0.6B-GGUF/Qwen3-Embedding-0.6B-Q8_0.gguf"
export QMD_RERANK_MODEL="/absolute/path/to/reranker.gguf"
export QMD_GENERATE_MODEL="/absolute/path/to/generator.gguf"
```

Voer de embeddings na het wijzigen van het embeddingmodel opnieuw uit, zodat de index overeenkomt met de
nieuwe vectorruimte.

## Extra paden indexeren

Laat QMD naar aanvullende mappen verwijzen om ze doorzoekbaar te maken:

```json5
{
  memory: {
    backend: "qmd",
    qmd: {
      paths: [{ name: "docs", path: "~/notes", pattern: "**/*.md" }],
    },
  },
}
```

Fragmenten uit extra paden verschijnen als `qmd/<collection>/<relative-path>` in
zoekresultaten. `memory_get` begrijpt dit voorvoegsel en leest uit de
juiste hoofdmap van de verzameling.

## Sessietranscripten indexeren

Schakel sessie-indexering in om eerdere gesprekken terug te vinden. QMD heeft zowel de
algemene sessiebron `memory.search` als de QMD-transcriptexporteur nodig:

```json5
{
  memory: {
    backend: "qmd",
    search: {
      experimental: { sessionMemory: true },
      sources: ["memory", "sessions"],
    },
    qmd: {
      sessions: { enabled: true },
    },
  },
}
```

Transcripten worden als opgeschoonde beurten van Gebruiker/Assistent geëxporteerd naar een speciale QMD-
verzameling onder `~/.openclaw/agents/<id>/qmd/sessions/`. Alleen
`sources: ["sessions"]` instellen exporteert geen transcripten naar QMD; schakel ook
`rememberAcrossConversations` of expliciete QMD-sessie-export in.

Sessieresultaten worden nog steeds gefilterd op
[`tools.sessions.visibility`](/nl/gateway/config-tools#toolssessions). De
standaardzichtbaarheid `tree` omvat de huidige sessie, de daaruit gestarte sessies
en groepssessies van dezelfde agent die via omgevingsbewustzijn van groepen worden gevolgd. Met
`session.dmScope: "main"` delen gebruikers in een DM-configuratie voor meerdere gebruikers de hoofdsessie
en kunnen ze inhoud uit de gevolgde groepen terugvinden. Gebruik een `dmScope` per peer
voor DM-isolatie of stel de zichtbaarheid in op `"self"` om omgevingsgebonden
leesacties uit gevolgde sessies uit te schakelen. Andere niet-gerelateerde sessies van dezelfde agent vereisen nog steeds
zichtbaarheid `"agent"`.

## Zoekbereik

Standaard worden QMD-zoekresultaten alleen weergegeven in directe sessies (niet
in groeps- of kanaalchats). Configureer `memory.qmd.scope` om dit te wijzigen:

```json5
{
  memory: {
    qmd: {
      scope: {
        default: "deny",
        rules: [{ action: "allow", match: { chatType: "direct" } }],
      },
    },
  },
}
```

Het bovenstaande fragment is de daadwerkelijke standaardregel. Wanneer het bereik een zoekopdracht weigert,
registreert OpenClaw een waarschuwing met het afgeleide kanaal en chattype, zodat lege
resultaten eenvoudiger te debuggen zijn.

## Bronvermeldingen

Wanneer `memory.citations` is ingesteld op `auto` of `on`, krijgen zoekfragmenten een
voettekst `Source: <path>#L<line>` (of `#L<start>-L<end>`) toegevoegd. In de modus `auto`
wordt de voettekst alleen toegevoegd voor directe chatsessies. Stel
`memory.citations = "off"` in om de voettekst weg te laten terwijl het pad intern nog steeds aan
de agent wordt doorgegeven.

## Wanneer te gebruiken

Kies QMD wanneer je het volgende nodig hebt:

- Herrangschikking voor resultaten van hogere kwaliteit.
- Projectdocumentatie of notities buiten de werkruimte doorzoeken.
- Eerdere sessiegesprekken terugvinden.
- Volledig lokaal zoeken zonder API-sleutels.

Voor eenvoudigere configuraties werkt de [ingebouwde engine](/nl/concepts/memory-builtin) goed
zonder extra afhankelijkheden.

## Problemen oplossen

**QMD niet gevonden?** Zorg ervoor dat het binaire bestand in de `PATH` van de Gateway staat. Als OpenClaw
als service draait, maak je een symbolische koppeling:
`sudo ln -s ~/.bun/bin/qmd /usr/local/bin/qmd`.

Als `qmd --version` in je shell werkt, maar OpenClaw nog steeds
`spawn qmd ENOENT` meldt, heeft het Gateway-proces waarschijnlijk een andere `PATH` dan
je interactieve shell. Leg het binaire bestand expliciet vast:

```json5
{
  memory: {
    backend: "qmd",
    qmd: {
      command: "/absolute/path/to/qmd",
    },
  },
}
```

Gebruik `command -v qmd` in de omgeving waarin QMD is geïnstalleerd en controleer het daarna opnieuw
met `openclaw memory status --deep`.

**Eerste zoekopdracht erg traag?** QMD downloadt GGUF-modellen bij het eerste gebruik. Warm vooraf op
met `qmd query "test"` en gebruik daarbij dezelfde XDG-mappen als OpenClaw.

**Veel QMD-subprocessen tijdens het zoeken?** Werk QMD indien mogelijk bij. OpenClaw
gebruikt één proces voor zoekopdrachten in meerdere verzamelingen met dezelfde bron, maar alleen wanneer de
geïnstalleerde QMD ondersteuning voor meerdere `-c`-filters aangeeft; anders
behoudt het voor de juistheid de oudere terugval per verzameling.

**Probeert QMD met alleen BM25 nog steeds llama.cpp te bouwen?** Stel
`memory.qmd.searchMode = "search"` in. OpenClaw behandelt die modus als
uitsluitend lexicaal, slaat QMD-statuscontroles voor vectoren en onderhoud van embeddings over en
laat semantische gereedheidscontroles over aan configuraties met `vsearch` of `query`.

**Time-out bij zoeken?** Verhoog `memory.qmd.limits.timeoutMs` (standaard: 4000ms).
Stel deze voor tragere hardware hoger in, bijvoorbeeld op `120000`. Deze limiet geldt voor
de eigen zoekopdrachten van QMD tijdens `memory_search`-aanroepen van de agent; installatie, synchronisatie,
ingebouwde terugval en aanvullend corpuswerk behouden hun eigen kortere deadlines.

**Lege resultaten in groeps- of kanaalchats?** Dit is te verwachten met de
standaard-`memory.qmd.scope`, die alleen directe sessies toestaat. Voeg een
`allow`-regel toe voor chattypen `group` of `channel` als je daar QMD-resultaten
wilt.

**Is het zoeken in het hoofdgeheugen plotseling te breed geworden?** Herstart de Gateway of wacht
op de volgende reconciliatie bij het opstarten. OpenClaw maakt verouderde beheerde
verzamelingen opnieuw aan met canonieke patronen `MEMORY.md` en `memory/` wanneer het
een conflict met dezelfde naam detecteert.

**Veroorzaken tijdelijke repo's die zichtbaar zijn in de werkruimte `ENAMETOOLONG` of defecte indexering?**
QMD-doorloop volgt de onderliggende QMD-scanner in plaats van de
ingebouwde regels voor symbolische koppelingen van OpenClaw. Bewaar tijdelijke monorepo-checkouts onder verborgen
mappen zoals `.tmp/` of buiten geïndexeerde QMD-hoofdmappen totdat QMD
cyclusveilige doorloop of expliciete uitsluitingsopties aanbiedt.

## Configuratie

Zie voor het volledige configuratieoppervlak (`memory.qmd.*`), zoekmodi, update-intervallen,
bereikregels en alle andere instellingen de
[referentie voor geheugenconfiguratie](/nl/reference/memory-config).

## Gerelateerd

- [Geheugenoverzicht](/nl/concepts/memory)
- [Ingebouwde geheugenengine](/nl/concepts/memory-builtin)
- [Honcho-geheugen](/nl/concepts/memory-honcho)
