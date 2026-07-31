---
read_when:
    - Je wilt een gratis promotioneel modelaanbod van ClawHub proberen
    - Je configureert een provider via een promotie in plaats van via onboarding
summary: CLI-referentie voor `openclaw promos` (promotionele modelaanbiedingen weergeven en claimen)
title: Promoties
x-i18n:
    generated_at: "2026-07-27T05:01:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 779eab2e9500b7376fabf9accb333e83ff5f84b085d51b7d551b5507b1e73adb
    source_path: cli/promos.md
    workflow: 16
---

# `openclaw promos`

Ontdek en claim promotionele modelaanbiedingen die op ClawHub zijn gepubliceerd. Het claimen van een
promotie configureert de provider (authenticatie en Plugin, indien nodig) en registreert
de modellen van de promotie — zonder de onboarding opnieuw uit te voeren en zonder
je standaardmodel te wijzigen, tenzij je daarvoor kiest.

Gerelateerd:

- Standaardmodel en fallbacks: [Modellen](/nl/cli/models)
- Authenticatie-instelling voor providers: [Aan de slag](/nl/start/getting-started)

## Opdrachten

```bash
openclaw promos list
openclaw promos claim <slug>
openclaw promos claim <slug> --api-key <key> --set-default
```

## `openclaw promos list`

Toont promoties die momenteel actief zijn, met hun modellen, de voorgestelde
standaard, de resterende tijd en de exacte claimopdracht. `--json` toont de onbewerkte
payload.

## `openclaw promos claim <slug>`

Claimt een actieve promotie:

1. Haalt de promotie op bij ClawHub en verifieert dat deze binnen de geldigheidsperiode valt.
2. Valideert de provider, de authenticatiekeuze en de opgegeven Plugin-pakketten van de promotie
   aan de hand van je geïnstalleerde OpenClaw-versie. Onbekende id's of niet-overeenkomende pakketten worden
   geweigerd — een promotie kan de CLI nooit iets laten uitvoeren wat deze niet al
   weet uit te voeren.
3. Hergebruikt je bestaande providerreferenties wanneer je die hebt. Anders wordt
   de normale authenticatieflow van de provider doorlopen (waarbij eerst de aanmeldings-URL van de promotie
   voor een gratis sleutel wordt weergegeven). `--api-key <key>` voltooit API-sleutelauthenticatie zonder
   prompts, overeenkomstig de niet-interactieve vlaggen van `openclaw onboard`; exporteer in plaats daarvan
   de omgevingsvariabele van de provider om de sleutel van de opdrachtregel te houden
   (bijvoorbeeld `OPENROUTER_API_KEY`) — bestaande referenties in omgevingsvariabelen worden
   automatisch gedetecteerd en er is geen vlag nodig.
4. Registreert de modellen van de promotie met hun aliassen. Bestaande aliassen worden
   nooit overschreven.
5. Biedt aan om het voorgestelde model van de promotie als je standaard in te stellen —
   `--set-default` slaat de vraag over; anders verandert er niets aan je
   standaardinstellingen.

Wanneer de geldigheidsperiode van de promotie eindigt, stopt de provider met het aanbieden van de gratis modellen;
je configuratie en referenties blijven ongewijzigd. Schakel op elk moment terug met
`openclaw models set <model>`.

## Passieve ontdekking in `models list`

`openclaw models list` toont ook promoties zonder dat je ClawHub
rechtstreeks raadpleegt:

- Actieve aanbiedingen waarvan je de modellen niet hebt geconfigureerd, verschijnen in een
  groep 'Beschikbaar via promotie' onder de tabel, elk met de bijbehorende
  claimopdracht.
- Modellen die je via `promos claim` hebt geregistreerd, hebben een tag `promo`, die
  verandert in `promo ended` zodra de geldigheidsperiode van de aanbieding is verstreken.
- De eerste keer dat een nieuwe aanbieding wordt gezien, verwijst een eenmalige melding naar
  `openclaw promos list`. Aanbiedingen die je al hebt weergegeven of geclaimd, worden nooit
  opnieuw aangekondigd.

Hierbij wordt een lokaal gecachte kopie van de gehoste promotiefeed van ClawHub gelezen
(normaal eenmaal per dag vernieuwd met een voorwaardelijk verzoek, of eerder wanneer de
gecachete momentopname verloopt; mislukte vernieuwingen worden stilzwijgend overgeslagen). Een verouderde
vernieuwing wacht maximaal 2.5 seconden en onderbreekt de lijstweergave nooit. De uitvoer van `--json` en
`--plain` blijft geschikt voor machines: geen promotiesecties of meldingen.
Bij het claimen wordt altijd opnieuw gevalideerd via de live ClawHub-API, zodat een voortijdig ingetrokken
aanbieding wordt geweigerd, zelfs wanneer een gecachte kopie deze nog steeds toont.
