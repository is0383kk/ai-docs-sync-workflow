---
read_when:
    - Je deelt één OpenClaw-agent met andere operators
    - Je moet de indicatoren voor sessie-eigenaar en aanwezigheid begrijpen
    - Je beslist of één gedeelde agent voldoende isolatie biedt
summary: Hoe sessie-eigenaarschap en aanwezigheid werken wanneer meerdere personen één agent bedienen
title: Modus voor meerdere gebruikers
x-i18n:
    generated_at: "2026-07-27T05:49:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c6a5a0e37b8dbeb2ebb7f32c3518acc6f3995dbfc09102f4d58c85e9cd62dfc2
    source_path: concepts/multi-user.md
    workflow: 16
---

In de modus voor meerdere gebruikers kunnen meerdere vertrouwde personen dezelfde OpenClaw-agent bedienen. Deze modus voegt sessie-eigenaarschap, live aanwezigheid en filtering op maker toe, zodat een team kan zien wie het werk is gestart en wie het momenteel bekijkt.

## Vertrouwensgrens

Iedereen die een agent kan bedienen, kan die alles laten doen waartoe de agent in staat is. Sessie-eigenaarschap, zichtbaarheid in de zijbalk en aanwezigheidsindicatoren zijn gebruiksfuncties, geen beveiligingsgrenzen.

Als personen geen toegang mogen hebben tot elkaars sessies, tools, aanmeldgegevens of bestanden, geef ze dan afzonderlijke agents of afzonderlijke vertrouwensgrenzen voor de Gateway/host. Vertrouw voor isolatie niet op eigenaarsavatars of filters.

## Eigenaarschap en aanwezigheid

Nieuwe sessies registreren een eenmalig instelbare `createdActor` wanneer het aanmaakpad kan bewijzen wie de sessie heeft veroorzaakt. Geauthenticeerde personen gebruiken hun permanente Gateway-profiel-id; aanvragende agents en systeempaden gebruiken hetzelfde actorveld. Sessies die zonder een bewezen actor zijn aangemaakt, blijven zonder toeschrijving.

Weergavenamen van personen worden opgehaald uit het huidige Gateway-profiel wanneer sessierijen worden geretourneerd. OpenClaw slaat geen labels op in sessie-items, dus als een profielnaam wordt gewijzigd, wordt de gebruikersinterface voor eigenaarschap bijgewerkt zonder de sessiegeschiedenis te herschrijven.

De webapp houdt eigenaarschap en aanwezigheid visueel gescheiden:

- Een effen eigenaarsavatar blijft gedurende de volledige levensduur van die sessie bestaan.
- Aanwezigheidsavatars met een rand of een doorschijnende weergave tonen personen die momenteel verbonden zijn of meekijken.
- Het personenfilter in de zijbalk toont sessies die door één identiteit zijn aangemaakt, met behoud van de bestaande aangepaste groepen.

Wanneer in de geladen sessielijst minder dan twee verschillende makers voorkomen, verbergt OpenClaw alle elementen voor eigenaarschap en personenfilters. Een Gateway met één gebruiker ziet er daarom ongewijzigd uit.

## Concepten

Start een sessie als concept om werk in uitvoering uit de zijbalken van teamgenoten te houden totdat je het publiceert. Concepten zijn nooit verborgen voor beheerders, die concepten van anderen zien met een vervaagd spookpictogram. Dit is een coördinatiefunctie, geen beveiligingsgrens.

## Toeschrijving van beurten

De toeschrijving van de afzender van een beurt gebeurt naar beste vermogen. Bijsturing kan invoer samenvoegen met een actieve beurt, waardoor het transcript niet altijd de bijdrage van elke persoon als een afzonderlijke beurt kan weergeven.

## Gerelateerd

- [De hoofdsessie](/concepts/main-session)
- [Sessiebeheer](/nl/concepts/session)
- [Aanwezigheid](/nl/concepts/presence)
- [Gateway-beveiliging](/nl/gateway/security)
