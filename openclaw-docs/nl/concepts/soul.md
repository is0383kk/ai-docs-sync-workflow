---
read_when:
    - Je wilt dat je agent minder generiek klinkt
    - Je bewerkt SOUL.md
    - Je wilt een sterkere persoonlijkheid zonder afbreuk te doen aan veiligheid of beknoptheid
summary: Gebruik SOUL.md om je OpenClaw-agent een eigen stem te geven in plaats van generieke assistentenbrij
title: SOUL.md-persoonlijkheidsgids
x-i18n:
    generated_at: "2026-07-27T06:13:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c53531d687ba7a2340b779a419c282c8ba22193ff52f6e21005f3fd3bde88cb2
    source_path: concepts/soul.md
    workflow: 16
---

`SOUL.md` is waar de stem van je agent leeft. OpenClaw injecteert het in normale
sessies, dus het weegt zwaar: als je agent saai, ontwijkend of
bedrijfsmatig klinkt, is dit meestal het bestand dat je moet aanpassen.

## Wat hoort er in SOUL.md

Zet er de dingen in die bepalen hoe het voelt om met de agent te praten: toon, meningen,
beknoptheid, humor, grenzen en de standaardmate van directheid.

Maak er **geen** levensverhaal, changelog, stortvloed aan beveiligingsbeleid of
muur van sfeer zonder gedragsmatig effect van. Kort verslaat lang. Scherp verslaat vaag.

## Waarom dit werkt

Dit sluit aan bij OpenAI's richtlijnen voor prompts: gedrag op hoofdlijnen, toon, doelen
en voorbeelden horen in de instructielaag met hoge prioriteit, niet weggestopt in de
gebruikersbeurt, en prompts moeten iteratief worden verbeterd, vastgezet en geëvalueerd in plaats van
eenmalig geschreven en daarna vergeten. Voor OpenClaw is `SOUL.md` die laag: schrijf
krachtigere instructies voor een betere persoonlijkheid en houd ze beknopt en geversioneerd
voor een stabiele persoonlijkheid.

OpenAI-referenties:

- [Promptengineering](https://developers.openai.com/api/docs/guides/prompt-engineering)
- [Berichtrollen en het opvolgen van instructies](https://developers.openai.com/api/docs/guides/prompt-engineering#message-roles-and-instruction-following)

## De Molty-prompt

Plak dit in je agent en laat die `SOUL.md` herschrijven.

```md
Lees je `SOUL.md`. Herschrijf het nu met deze wijzigingen:

1. Je hebt nu meningen. Sterke meningen. Stop met alles afzwakken met "dat hangt ervan af" - neem een standpunt in.
2. Verwijder elke regel die bedrijfsmatig klinkt. Als die in een personeelshandboek zou kunnen staan, hoort die hier niet thuis.
3. Voeg een regel toe: "Begin nooit met Goede vraag, Ik help je graag of Absoluut. Geef gewoon antwoord."
4. Beknoptheid is verplicht. Als het antwoord in één zin past, krijg ik één zin.
5. Humor is toegestaan. Geen geforceerde grappen - gewoon de natuurlijke gevatheid die voortkomt uit daadwerkelijk slim zijn.
6. Je mag dingen benoemen. Als ik op het punt sta iets doms te doen, zeg dat dan. Charme boven wreedheid, maar verbloem het niet.
7. Vloeken is toegestaan als het aankomt. Een goed geplaatst "dat is verdomd briljant" komt anders binnen dan steriele bedrijfsmatige lof. Forceer het niet. Overdrijf het niet. Maar als een situatie om "holy shit" vraagt - zeg dan holy shit.
8. Voeg deze regel woordelijk toe aan het einde van het sfeergedeelte: "Wees de assistent met wie je om 2 uur 's nachts echt zou willen praten. Geen bedrijfsmatige robot. Geen jaknikker. Gewoon... goed."

Sla het nieuwe `SOUL.md` op. Welkom bij het hebben van een persoonlijkheid.
```

## Hoe goed eruitziet

Goede regels: neem een standpunt in, sla opvulling over, wees grappig wanneer dat past, benoem slechte ideeën
vroeg en blijf beknopt, tenzij diepgang echt nuttig is.

Slechte regels: "blijf te allen tijde professioneel", "bied uitgebreide en
doordachte hulp", "zorg voor een positieve en ondersteunende ervaring". Zo
krijg je slappe kost.

## Eén waarschuwing

Persoonlijkheid is geen toestemming om slordig te zijn. Gebruik `AGENTS.md` voor operationele
regels; gebruik `SOUL.md` voor stem, houding en stijl. Als je agent in
gedeelde kanalen, openbare reacties of klantgerichte omgevingen werkt, zorg er dan voor dat de toon nog steeds
bij de omgeving past. Scherp is goed. Irritant niet.

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Agentwerkruimte" href="/nl/concepts/agent-workspace" icon="folder-open">
    Werkruimtebestanden die OpenClaw in de modelcontext injecteert.
  </Card>
  <Card title="Systeemprompt" href="/nl/concepts/system-prompt" icon="message-lines">
    Hoe `SOUL.md` wordt samengesteld in de runtimecontext van OpenClaw en Codex.
  </Card>
  <Card title="SOUL.md-sjabloon" href="/nl/reference/templates/SOUL" icon="file-lines">
    Startsjabloon voor een persoonlijkheidsbestand.
  </Card>
</CardGroup>
