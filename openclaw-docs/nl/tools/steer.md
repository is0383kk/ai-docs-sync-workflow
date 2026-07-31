---
read_when:
    - /steer of /tell gebruiken terwijl er al een agent actief is
    - /steer vergelijken met /queue-modi
    - Beslissen of je de huidige run of een ACP-sessie wilt bijsturen
sidebarTitle: Steer
summary: Stuur een actieve run bij zonder de wachtrijmodus te wijzigen
title: Sturen
x-i18n:
    generated_at: "2026-07-27T06:17:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d420e14982d52520e415103ffa6d86923fad6f13c43ff7741ebbd8dde0d0073f
    source_path: tools/steer.md
    workflow: 16
---

`/steer` probeert eerst instructies naar een reeds actieve run te sturen. Dit is bedoeld voor
momenten waarop je „deze run wilt aanpassen terwijl die nog bezig is”. Als de huidige runtime
geen bijsturing kan accepteren, stuurt OpenClaw het bericht in plaats daarvan als een normale prompt,
zodat het niet verloren gaat.

## Huidige sessie

Gebruik `/steer` op het hoogste niveau om de actieve run voor de huidige sessie aan te sturen:

```text
/steer geef de voorkeur aan de kleinere patch en houd de tests gericht
/tell vat samen voordat je de volgende toolaanroep doet
```

Gedrag:

- Richt zich alleen op de actieve run van de huidige sessie.
- Werkt onafhankelijk van de `/queue`-modus van de sessie.
- Start een normale beurt met hetzelfde bericht wanneer de sessie inactief is of de
  actieve run geen bijsturing kan accepteren.
- Gebruikt het bijsturingspad van de actieve runtime, zodat het model de instructies bij
  de volgende ondersteunde runtimegrens ziet.

## Bijsturen versus in de wachtrij plaatsen

`/queue steer` zorgt ervoor dat normale inkomende berichten de actieve run proberen bij te sturen wanneer
ze aankomen terwijl een run actief is. `/steer <message>` is een expliciete opdracht
die probeert het bericht van die opdracht bij de volgende ondersteunde
runtimegrens in de actieve run in te voegen, ongeacht de opgeslagen instelling `/queue`. Wanneer
die invoeging niet beschikbaar is, wordt het opdrachtvoorvoegsel verwijderd en gaat `<message>`
verder als een normale prompt.

De expliciete opdracht `/steer` (en `/tell`) wordt door de Gateway ondersteund. Selecteer in
`openclaw chat` of `openclaw tui --local` `/queue steer` en verstuur de
instructies als een normaal bericht; de ingebedde runtime past hetzelfde bijsturingsbeleid toe
zonder een Gateway-opdracht door te sturen.

Gebruik:

- `/steer <message>` wanneer je de actieve run direct wilt begeleiden.
- `/queue steer` wanneer je wilt dat toekomstige normale berichten standaard actieve runs
  bijsturen.
- `/queue collect` of `/queue followup` wanneer toekomstige normale berichten op een
  latere beurt moeten wachten in plaats van de actieve run bij te sturen.
- `/queue interrupt` wanneer het nieuwste bericht de actieve run moet vervangen
  in plaats van deze bij te sturen.

Zie [Opdrachtwachtrij](/nl/concepts/queue) en
[Bijsturingswachtrij](/nl/concepts/queue-steering) voor wachtrijmodi en bijsturingsgrenzen.

## Sub-agents

`/steer` op het hoogste niveau richt zich op de actieve run van de huidige sessie. Sub-agents rapporteren
terug aan hun bovenliggende/aanvragende sessie; `/subagents` is alleen bedoeld voor zichtbaarheid.

## ACP-sessies

Gebruik `/acp steer` wanneer het doel een ACP-harnesssessie is:

```text
/acp steer --session agent:main:acp:codex maak de reproductie strakker
```

Zie [ACP-agents](/nl/tools/acp-agents) voor de selectie van ACP-sessies en het
runtimegedrag.

## Gerelateerd

- [Slash-opdrachten](/nl/tools/slash-commands)
- [Opdrachtwachtrij](/nl/concepts/queue)
- [Bijsturingswachtrij](/nl/concepts/queue-steering)
- [Sub-agents](/nl/tools/subagents)
