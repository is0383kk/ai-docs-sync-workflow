---
read_when:
    - Je moet iets vinden dat in een eerdere sessie is besproken
    - Je wilt inzicht in de privacy of indexering van het zoeken in sessies
summary: Doorzoek eerdere sessietranscripten en open de overeenkomende context opnieuw
title: Sessies zoeken
x-i18n:
    generated_at: "2026-07-27T05:09:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3e9cda6b656b689eef0636592914f4890a64dca5e955aa03908377903aaa29c9
    source_path: concepts/session-search.md
    workflow: 16
---

# Sessies doorzoeken

`sessions_search` doorzoekt de tekst van de gebruiker en de assistent in je eigen eerdere sessies. Elk resultaat
bevat een `sessionKey`, tijdstempel, rol en een kort overeenkomend fragment. Geef de geretourneerde
`sessionKey` door aan `sessions_history` wanneer je de omliggende conversatie nodig hebt.

## Zichtbaarheid en uitvoer

De zoekfunctie gebruikt dezelfde regels voor sessiezichtbaarheid als `sessions_history`. Resultaten buiten de
zichtbare sessieboom van de aanroeper worden verwijderd voordat resultaatlimieten worden toegepast. Agents in een sandbox blijven
beperkt tot sessies die ze zelf hebben gestart wanneer zichtbaarheid van gestarte sessies is ingeschakeld.

Fragmenten worden geredigeerd voordat ze naar het model worden teruggestuurd. Resultaten worden ook begrensd op aantal, fragmentlengte
en totale responsgrootte.

## Levenscyclus van de index

OpenClaw slaat een volledige-tekstindex op naast de transcriptierijen in de SQLite-database van elke agent.
Nieuwe berichten van gebruikers en assistenten worden geïndexeerd in dezelfde transactie waarin ze worden opgeslagen, zodat de
index nooit achterloopt op actieve conversaties; toolresultaten, redeneringsblokken en afbeeldingen worden uitgesloten.
Alleen de actieve tak van het transcript kan worden doorzocht.

Transcripties van vóór de index (bijvoorbeeld sessies die zijn geïmporteerd door `openclaw doctor`) en
sessies waarvan de actieve tak is teruggedraaid, worden opnieuw geïndexeerd door een afstemming op de achtergrond die
bij de volgende zoekopdracht start. Een respons met `indexing: true` kan daarom onvolledig zijn; probeer het opnieuw nadat
het indexeren is voltooid. Als een sessie wordt verwijderd, worden de indexvermeldingen ervan in dezelfde transactie verwijderd.

De zoekfunctie gebruikt momenteel de Unicode-woordtokenizer van SQLite waarbij diakritische tekens worden verwijderd. Trigramtokenisatie
voor het zoeken naar CJK-subtekenreeksen is een toekomstige verbetering.

## Sessies doorzoeken versus geheugen doorzoeken

Gebruik `sessions_search` voor exacte woorden of woordgroepen uit onbewerkte sessietranscripties. Gebruik
[`memory_search`](/nl/concepts/memory-search) voor duurzame geheugenbestanden en semantisch terugzoeken. Het
experimentele sessiegeheugencorpus vormt de semantische aanvulling op deze exacte zoekfunctie voor transcripties.
