---
read_when:
    - Je wilt een korte tussenvraag stellen over de huidige sessie
    - Je implementeert of debugt BTW-gedrag in verschillende clients
summary: Tijdelijke tussenvragen met /btw
title: Trouwens, tussendoorvragen
x-i18n:
    generated_at: "2026-07-27T05:52:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 338a54d0e15ec90aebaeeaee551559a26f1437f7b6dcdde4a4b1e63347ad0759
    source_path: tools/btw.md
    workflow: 16
---

`/btw` (alias `/side`) stelt een snelle tussenvraag over de **huidige
sessie** zonder deze aan de gespreksgeschiedenis toe te voegen. Het is gemodelleerd naar
`/btw` van Claude Code en aangepast aan de Gateway- en multichannelarchitectuur
van OpenClaw.

```text
/btw wat is er veranderd?
/side wat betekent deze fout?
```

## Wat het doet

1. Maakt een momentopname van de huidige sessie als achtergrondcontext (inclusief een
   eventuele prompt van de actieve hoofdtaak).
2. Voert een afzonderlijke, eenmalige tussenvraag uit en instrueert het model om alleen de
   tussenvraag te beantwoorden en de hoofdtaak niet te hervatten of bij te sturen.
3. Levert het antwoord als een live zijresultaat, niet als een normaal assistentbericht.
4. Schrijft de vraag of het antwoord nooit naar de sessiegeschiedenis of `chat.history`.

De hoofdtaak blijft, als die actief is, onaangeroerd.

Voor Codex-harness-sessies splitst BTW de actieve Codex-app-serverthread af naar
een tijdelijke onderliggende thread, in plaats van een afzonderlijke provideroproep uit te voeren. Hierdoor
blijven Codex OAuth en het native gedrag van tools en threads intact, en behoudt de afgesplitste
thread het huidige goedkeuringsbeleid, de sandbox en het native
tooloppervlak van de bovenliggende thread. De afgesplitste thread krijgt een grensprompt die het model vertelt dat
alles ervoor overgenomen referentiecontext is en geen actieve instructies zijn,
en dat alleen berichten na de grens actief zijn. `/btw` vereist een
bestaande Codex-thread; stuur eerst een normaal bericht.

Voor CLI-runtimealiassen roept BTW de verantwoordelijke CLI-backend aan in eenmalige
tussenvraagmodus: het voegt opgeschoonde gesprekscontext toe aan een nieuwe CLI-
aanroep, waarbij toolbundeling en herbruikbare sessiestatus zijn uitgeschakeld, en voegt
alle door de backend ondersteunde vlaggen voor niet-hervatten en geen-tools toe. Directe runtimes (niet-CLI)
gebruiken in plaats daarvan een directe, eenmalige provideroproep.

## Wat het niet doet

`/btw` maakt geen duurzame sessie, zet de onvoltooide hoofdtaak niet voort,
slaat vraag- en antwoordgegevens niet op in de transcriptgeschiedenis en blijft niet behouden na opnieuw laden.

## Leveringsmodel

Normale assistentchat gebruikt de Gateway-gebeurtenis `chat`. BTW gebruikt een afzonderlijke
gebeurtenis `chat.side_result`, zodat clients deze niet kunnen verwarren met de gewone
gespreksgeschiedenis. Omdat deze niet opnieuw wordt afgespeeld vanuit `chat.history`,
verdwijnt deze na opnieuw laden.

## Gedrag per oppervlak

| Oppervlak         | Gedrag                                                                                                                                                                                                                                                                              |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| TUI               | Wordt inline weergegeven in het chatlogboek, zichtbaar anders dan een normaal antwoord, en kan worden gesloten met `Enter` of `Esc`.                                                                                                                                       |
| Externe kanalen   | Wordt geleverd als een duidelijk gelabeld eenmalig antwoord (Telegram, WhatsApp en Discord hebben geen lokale tijdelijke overlay).                                                                                                                                                   |
| Control UI / web  | Wordt weergegeven als een zwevend paneel 'Zijchat' dat aan de thread is vastgemaakt. Antwoorden worden als beurten verzameld en via een invoerveld 'Vervolgvraag' stel je de volgende tussenvraag. Sluiten (`Esc` of de X) bewaart het gesprek en opent het opnieuw bij het volgende antwoord; de prullenbakknop verwijdert het en stopt een lopende uitvoering. |

## Selectiepop-up (Control UI)

Wanneer je tekst in een chatbericht in de Control UI markeert, wordt een kleine
selectiepop-up met twee acties geopend:

- **Meer details** verzendt onmiddellijk een impliciete `/btw`-vraag waarin het
  model wordt gevraagd de gemarkeerde tekst uit te leggen binnen de context van de huidige
  sessie. Het antwoord verschijnt in het zwevende zijchatpaneel.
- **Vraag in zijchat** vult het invoerveld vooraf met een `/btw`-concept dat de
  gemarkeerde tekst citeert, zodat je er je eigen vraag over kunt typen.

Beide acties volgen de normale semantiek van `/btw`: de vraag en het antwoord blijven buiten
de sessiegeschiedenis en de hoofdtaak blijft onaangeroerd.

## Wanneer je het gebruikt

Gebruik `/btw` voor een snelle verduidelijking, een feitelijk antwoord op een tussenvraag terwijl een lange taak
nog bezig is, of een tijdelijk antwoord dat niet in toekomstige
sessiecontext moet worden opgenomen.

```text
/btw welk bestand bewerken we?
/btw vat de huidige taak in één zin samen
/btw hoeveel is 17 * 19?
```

Voor alles wat je deel wilt laten uitmaken van de toekomstige werkcontext
van de sessie, stel je de vraag in plaats daarvan op de normale manier in de hoofdsessie.

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Slash-opdrachten" href="/nl/tools/slash-commands" icon="terminal">
    Catalogus met native opdrachten en chatinstructies.
  </Card>
  <Card title="Denkniveaus" href="/nl/tools/thinking" icon="brain">
    Niveaus voor redeneerinspanning bij de modeloproep voor de tussenvraag.
  </Card>
  <Card title="Sessie" href="/nl/concepts/session" icon="comments">
    Sessiesleutels, geschiedenis en persistentiesemantiek.
  </Card>
  <Card title="Bijstuur-opdracht" href="/nl/tools/steer" icon="arrow-right">
    Voeg een bijsturingsbericht toe aan de actieve taak zonder deze te beëindigen.
  </Card>
</CardGroup>
