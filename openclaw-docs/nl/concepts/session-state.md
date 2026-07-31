---
read_when:
    - Je wilt dat agents merken wanneer mensen of andere agents buiten hun medeweten een sessie wijzigen
    - Je debugt meldingen over statuswijzigingen, watch-cursors of session_status changesSince
    - Je wilt begrijpen hoe bovenliggende agents gesynchroniseerd blijven met onderliggende sessies
sidebarTitle: Session state awareness
summary: 'Logboek met signalen voor duurzame sessiestatus: statusversies, watchers, meldingen over verouderde status en afstemming'
title: Bewustzijn van sessiestatus
x-i18n:
    generated_at: "2026-07-27T05:32:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: bb4126a0802e1ca4418f225c792490493a78886089b81c3b4567f72090ce34f4
    source_path: concepts/session-state.md
    workflow: 16
---

Wanneer meerdere sessies aan hetzelfde probleem werken — een manager die taken delegeert aan onderliggende sessies, een mens die rechtstreeks in een werksessie springt, twee agents die coördineren via [`sessions_send`](/nl/concepts/session-tool) — bouwt elke sessie aannames op over de andere sessies. Die aannames raken achterhaald zodra een andere actor ingrijpt. Bewustzijn van de sessiestatus is het mechanisme dat de ingreep detecteert, de betreffende sessie één keer informeert en deze een snelle manier biedt om bij te werken voordat ze handelt.

Drie onderdelen werken samen:

1. Een **duurzaam signaallogboek** registreert geselecteerde statuswijzigingen per sessie.
2. **Watchers** bewaren cursors per doel en ontvangen één samengevoegde melding over een achterhaalde status.
3. **Afstemming** haalt de exacte delta op via `session_status` met `changesSince`.

## Het signaallogboek

OpenClaw voegt een getypeerde gebeurtenis toe aan de gedeelde statusdatabase (`session_state_events`) wanneer een bewaakte sessie wezenlijk verandert. Gebeurtenissen bevatten metadata en een samenvatting van één regel — nooit berichtinhoud.

| Soort                  | Geregistreerd wanneer                                    | Informeert watchers |
| ---------------------- | -------------------------------------------------------- | ------------------- |
| `human_direct_message` | Een mens rechtstreeks een beurt naar een bewaakte sessie stuurt | Ja                  |
| `upstream_missing`     | De upstreambron van een geadopteerde sessie verdwijnt    | Ja                  |
| `goal_changed`         | De doelstatus van de sessie wordt aangemaakt, bijgewerkt of gewist | Ja                  |
| `child_spawned`        | Een sub-agent- of onderliggende ACP-sessie wordt aangemaakt | Nee (initialiseert cursor) |
| `run_completed`        | Een onderliggende uitvoering met succes eindigt          | Nee (alleen logboek) |
| `run_failed`           | Een onderliggende uitvoering mislukt, een time-out bereikt of wordt geannuleerd | Nee (alleen logboek) |
| `compacted`            | De geschiedenis van de sessie wordt gecompacteerd        | Nee (alleen logboek) |
| `adopted`              | Een catalogussessie in OpenClaw wordt geadopteerd         | Nee (alleen logboek) |

Elke gebeurtenis vermeldt de actor (`human`, `agent` of `system`). Geannuleerde onderliggende uitvoeringen en uitvoeringen met een time-out worden als mislukkingen geregistreerd, waarbij de precieze uitkomst (`cancelled`, `timeout` of `error`) in de gebeurtenispayload behouden blijft.

De **statusversie** van een sessie is eenvoudigweg het hoogste volgnummer in het logboek, bijgehouden in een duurzame kop per sessie die opschoning overleeft. `sessions_list`-rijen bevatten `stateVersion` wanneer een sessie wijzigingen heeft geregistreerd; `session_status` rapporteert dit altijd.

Soorten die alleen worden gelogd, bestaan voor de afstemmingsgeschiedenis en niet voor meldingen: de normale levering van voltooiingsmeldingen voor onderliggende uitvoeringen blijft onder beheer van [sub-agentmeldingen](/nl/tools/subagents), en het signaallogboek dupliceert deze nooit.

## Watchers

Een watcher is een sessie die een cursor (`session_watch_cursors`) op een doel bijhoudt. Cursors zijn afkomstig uit twee bronnen:

- **Impliciet (spawn-relaties).** Wanneer een sessie een sub-agent of onderliggende ACP-sessie spawnt, wordt de cursor van de bovenliggende sessie automatisch geïnitialiseerd op de spawnversie van de onderliggende sessie. Bovenliggende sessies abonneren zich nooit handmatig.
- **Expliciet (`sessions_send watch: true`).** Elke coördinator kan een doel bewaken dat niet door spawning is ontstaan: geef `watch: true` door aan `sessions_send`, waarna de afzender na een geslaagde verzending wordt geregistreerd als watcher van de sessie die het bericht daadwerkelijk heeft ontvangen. De registratie begint bij de huidige statusversie van het doel — eerdere geschiedenis levert nooit meldingen op. Het toolresultaat rapporteert `watched: true|false` wanneer de parameter was ingesteld.

De identiteit van een watcher moet een sessiesleutel zijn die met een agent is gekwalificeerd. Onder `session.scope="global"` is de gedeelde sleutel `global` ambigu tussen agents, waardoor zulke sessies wel het duurzame logboek en `changesSince` krijgen, maar geen proactieve meldingen.

Bewakingen ruimen zichzelf op: cursorrijen verlopen volgens de bewaartermijn van het signaallogboek, worden verwijderd wanneer de watchersessie wordt gereset en worden verwijderd wanneer een van beide sessies wordt verwijderd. In v1 bestaat geen opdracht om de bewaking te stoppen.

Bewaakte sessies die uit een sessiecatalogus zijn geadopteerd, worden met een vast interval gecontroleerd op rechtstreekse menselijke activiteit in de upstreambron. Gedetecteerde activiteit komt terecht in hetzelfde signaallogboek en dezelfde watcherflow als andere rechtstreekse menselijke beurten.

Als de upstreambron van een geadopteerde sessie extern wordt verwijderd, leveren drie opeenvolgende ontbrekende controles (ongeveer drie monitorintervallen) één `upstream_missing`-signaal op voor de watchers en wordt de upstreamkoppeling verwijderd. Als de catalogussessie opnieuw wordt voortgezet, wordt een nieuwe koppeling gemaakt.

## Meldingen: één, niet veel

Wanneer een gebeurtenis die voor melding in aanmerking komt binnenkomt en de cursor van een watcher achterloopt, ontvangt de watcher tijdens de volgende beurt één systeemmelding:

```
Sessie "agent:main:subagent:child" is gewijzigd (andere actor). Stem af voordat je handelt: session_status sessionKey "agent:main:subagent:child" changesSince 12.
```

Watchers van hoofdsessies worden ook onmiddellijk gewekt via een Heartbeat-wake; geneste sub-agentwatchers krijgen de melding tijdens hun volgende beurt.

Het protocol voorkomt bewust spam:

- **Eén openstaande melding per watcher-doelpaar.** De meldingstekst blijft bytegewijs gelijk zolang deze openstaat en de wachtrij voor systeemgebeurtenissen dedupliceert op basis daarvan, zodat twintig snelle wijzigingen aan hetzelfde doel nog steeds slechts één regel in de prompt van de watcher opleveren.
- **Bevroren watermerk.** De cursor bevriest zijn gemelde positie wanneer een melding in de wachtrij wordt geplaatst. Verdere wezenlijke gebeurtenissen verhogen alleen het wezenlijke watermerk; ze leiden niet tot een nieuwe melding.
- **Bevestigen bij uitlezen, alleen opnieuw openen bij tussentijdse activiteit.** Wanneer de beurt van de watcher de melding verwerkt, wordt de cursor vooruitgezet. Als er tussen het in de wachtrij plaatsen en het uitlezen meer wezenlijke gebeurtenissen zijn binnengekomen, wordt precies één nieuwe melding geopend voor het resterende deel.
- **Zelfonderdrukking.** Een watcher krijgt nooit meldingen over gebeurtenissen die deze zelf heeft veroorzaakt.
- **Herstel na herstart.** Openstaande meldingen bevinden zich in een wachtrij in het geheugen; na een herstart van de Gateway maakt een opstartscan ze opnieuw aan vanuit duurzame cursors.

## Afstemmen

De melding vertelt de watcher precies wat deze moet doen. `session_status` met `changesSince: <version>` retourneert de getypeerde gebeurtenissen na die versie (maximaal 200), zonder cursors vooruit te zetten:

```json
{
  "stateVersion": 19,
  "stateChanges": {
    "events": [
      {
        "sequence": 14,
        "kind": "human_direct_message",
        "actorType": "human",
        "summary": "menselijk bericht via telegram"
      },
      { "sequence": 19, "kind": "goal_changed", "actorType": "human", "summary": "doel bijgewerkt" }
    ],
    "historyGap": false
  }
}
```

`historyGap: true` betekent dat de aangevraagde versie ouder is dan de bewaarde geschiedenis — vernieuw in plaats van het antwoord als een exacte delta te behandelen de volledige sessiestatus (`sessions_history`, `session_status`). Het signaal voor de ontbrekende geschiedenis is exact: het is afkomstig van een opgeschoond watermerk per sessie en wordt niet afgeleid uit berekeningen met volgnummers.

## Opslag en limieten

De geschiedenis wordt opgeslagen in de gedeelde statusdatabase en is begrensd tot 30 dagen en 50.000 rijen; de koppen per sessie blijven na opschoning monotoon oplopen. Registratie gebeurt op basis van beste inspanning — een mislukte toevoeging wordt gelogd en laat de oorspronkelijke beurt nooit mislukken — dus `stateVersion` is een kop van het signaallogboek en geen transactionele versie voor het vastleggen van gegevenswijzigingen.

Huidige limieten:

- De levering van meldingen veronderstelt dat één Gateway-proces eigenaar is van de gedeelde statusdatabase. Meerdere Gateways delen het duurzame logboek en `changesSince`, maar v1 verstuurt geen meldingen tussen processen.
- Compaction-gebeurtenissen omvatten de eigenaren van Compaction in de ingebedde runtime; Compaction die uitsluitend in de native harness plaatsvindt, wordt niet volledig gelogd.
- Payloaddetails voor geannuleerde uitkomsten worden momenteel geproduceerd door onderliggende ACP-uitvoeringen; annuleringen van native sub-agents worden weergegeven als algemene mislukkingen.
- Detectie van upstream-zelfecho vergelijkt genormaliseerde gebruikerstekst. Een externe prompt die overeenkomt met een van de 10 meest recente gebruikersberichten van de OpenClaw-zijde van de sessie, wordt als zelfecho behandeld.
- Eén lokale Claude JSONL-rij die groter is dan de scanlimiet van 1 MiB per interval blokkeert in v1 de cursor van die sessie; niet-geclassificeerde bytes worden nooit overgeslagen.
- Claude-controles op gekoppelde Nodes classificeren per interval de laatste 50 transcriptitems. Grotere pieken kunnen buiten het scanvenster van v1 vallen.
- Het lezen van Claude-geschiedenis op gekoppelde Nodes geeft geen definitief resultaat wanneer een thread niet wordt gevonden, waardoor externe verwijderingen in Claude in v1 niet als `upstream_missing` worden geclassificeerd.
- Catalogussessies die niet zijn geadopteerd, vallen in v1 buiten de bewustzijnslaag.
- Sessies die vóór deze functie zijn geadopteerd, hebben geen upstreamkoppeling; zet ze één keer voort vanuit de catalogus om upstreambewaking te starten.
- Upstreamkoppelingen gaan ervan uit dat elke geadopteerde sessiesleutel aan één beherende agent is gekoppeld (adoptie gebruikt de standaardagent van de opslag). Adoptie van dezelfde externe thread door meerdere agents wordt in v1 niet bewaakt.

## Gerelateerd

- [Sessietools](/nl/concepts/session-tool) — `sessions_send`, `session_status`, `sessions_list`
- [Sub-agents](/nl/tools/subagents) — spawn-relaties en voltooiingsmeldingen
- [Heartbeat](/nl/gateway/heartbeat) — hoe meldingen in de wachtrij hoofdsessies wekken
- [Sessiebeheer](/nl/concepts/session) — sessiesleutels, bereiken, levenscyclus
