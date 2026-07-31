---
read_when:
    - Een ClawHub-beveiligingsprobleem melden
    - Inzicht in de openbaarmaking van kwetsbaarheden in ClawHub
    - ClawHub-platformproblemen onderscheiden van problemen met Skills of Plugins van derden
sidebarTitle: Security
summary: Hoe je beveiligingsproblemen met ClawHub meldt en wanneer kwetsbaarheden openbaar worden gemaakt.
title: Beveiliging
x-i18n:
    generated_at: "2026-07-27T04:58:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 64dcc75ad4210082c79326e617bac38908e927fd8d260fd029aa87e0a11cd324
    source_path: clawhub/security.md
    workflow: 16
---

# Beveiliging

Beveiligingsproblemen met ClawHub kunnen via GitHub Security Advisories worden gemeld voor
`openclaw/clawhub`.

Gebruik GitHub Security Advisories voor kwetsbaarheden in ClawHub zelf. Goede
adviesmeldingen voor ClawHub omvatten fouten in:

- de ClawHub-website, API of CLI
- publicatie in het register, downloads, installaties of artefactintegriteit
- authenticatie, autorisatie of API-tokens
- scannen, moderatie of afhandeling van meldingen

Gebruik ClawHub-adviezen niet voor kwetsbaarheden in de eigen broncode van een Skill of
Plugin van derden. Meld deze rechtstreeks aan de uitgever of de broncode-
repository waarnaar vanuit de ClawHub-vermelding wordt verwezen.

## Openbaarmaking van kwetsbaarheden

Omdat ClawHub een gehoste cloudapplicatie is, worden kwetsbaarheden in de
ClawHub-service standaard niet openbaar gemaakt. Ze worden openbaar gemaakt
wanneer er bewijs is van daadwerkelijke gevolgen voor gebruikers of wanneer
gebruikers actie moeten ondernemen.

Voorbeelden van daadwerkelijke gevolgen voor gebruikers zijn bevestigde uitbuiting,
blootstelling van gebruikersgegevens of geheimen, schadelijke inhoud die gebruikers
bereikt door een platformfout, of elk probleem waardoor gebruikers inloggegevens
moeten vervangen, lokale software moeten bijwerken of andere beschermende
maatregelen moeten nemen.

Kwetsbaarheden in door gebruikers geïnstalleerde software worden openbaar gemaakt,
zoals ClawHub CLI-pakketten, binaire bestanden, bibliotheken of andere
releaseartefacten die gebruikers lokaal moeten bijwerken.

## Gerelateerde pagina's

Zie [Beveiligingsaudits](/clawhub/security-audits) voor auditlabels tijdens de
installatie, risiconiveaus, bevindingen en interpretatie.

Zie [Moderatie en accountveiligheid](/clawhub/moderation) voor marktplaatsmeldingen,
moderatieblokkeringen, verborgen vermeldingen, verbanningen en accountstatus.
