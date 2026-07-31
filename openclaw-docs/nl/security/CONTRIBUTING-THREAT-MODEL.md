---
read_when:
    - Je wilt beveiligingsbevindingen of dreigingsscenario's bijdragen
    - Het dreigingsmodel beoordelen of bijwerken
summary: Bijdragen aan het dreigingsmodel van OpenClaw
title: Bijdragen aan het dreigingsmodel
x-i18n:
    generated_at: "2026-07-27T06:34:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4e2e5cd95e8a2bf5ee4bd167afedfadf9aa876e4260e2d0bfb5f414cd4255410
    source_path: security/CONTRIBUTING-THREAT-MODEL.md
    workflow: 16
---

Het [dreigingsmodel](/nl/security/THREAT-MODEL-ATLAS) is een levend document. Bijdragen van iedereen zijn welkom; je hebt geen achtergrond in beveiliging of MITRE ATLAS nodig.

<Note>
Dit is bedoeld voor aanvullingen op het dreigingsmodel, niet voor het melden van actuele kwetsbaarheden. Als je een misbruikbare kwetsbaarheid hebt gevonden, volg dan in plaats daarvan de instructies voor verantwoorde openbaarmaking op de [Trust-pagina](https://trust.openclaw.ai).
</Note>

## Manieren om bij te dragen

**Voeg een dreiging toe.** Open een issue op [openclaw/trust](https://github.com/openclaw/trust/issues) en beschrijf het aanvalsscenario in je eigen woorden. Nuttig, maar niet verplicht:

- Het aanvalsscenario en hoe dit kan worden misbruikt.
- Welke componenten worden getroffen (CLI, Gateway, kanalen, ClawHub, MCP-servers enzovoort).
- Jouw inschatting van de ernst (laag / gemiddeld / hoog / kritiek).
- Links naar gerelateerd onderzoek, CVE's of praktijkvoorbeelden.

Beheerders wijzen tijdens de beoordeling de ATLAS-toewijzing, dreigings-ID en het risiconiveau toe.

**Stel een risicobeperkende maatregel voor.** Open een issue of PR die naar de dreiging verwijst. Wees specifiek en praktisch: "snelheidsbeperking per afzender van 10 berichten/minuut bij de Gateway" is nuttiger dan "implementeer snelheidsbeperking".

**Stel een aanvalsketen voor.** Aanvalsketens laten zien hoe meerdere dreigingen samen een realistisch scenario vormen. Beschrijf de stappen en hoe een aanvaller ze zou combineren; een kort verhaal werkt beter dan een formele sjabloon.

**Verbeter bestaande inhoud of los problemen op.** Typefouten, verduidelijkingen, verouderde informatie en betere voorbeelden: PR's zijn welkom, een issue is niet nodig.

## Frameworkreferentie

Dreigingen worden gekoppeld aan [MITRE ATLAS](https://atlas.mitre.org/) (Adversarial Threat Landscape for AI Systems), een framework voor AI/ML-specifieke dreigingen, zoals promptinjectie, misbruik van tools en uitbuiting van agents. Je hoeft ATLAS niet te kennen om bij te dragen; beheerders koppelen inzendingen tijdens de beoordeling.

**Dreigings-ID's.** Elke dreiging krijgt een ID zoals `T-EXEC-003`, dat tijdens de beoordeling door beheerders wordt toegewezen.

| Code    | Categorie                                           |
| ------- | --------------------------------------------------- |
| RECON   | Verkenning - informatie verzamelen                  |
| ACCESS  | Initiële toegang - toegang verkrijgen               |
| EXEC    | Uitvoering - schadelijke acties uitvoeren           |
| PERSIST | Persistentie - toegang behouden                     |
| EVADE   | Omzeiling van verdediging - detectie vermijden      |
| DISC    | Ontdekking - meer te weten komen over de omgeving   |
| EXFIL   | Exfiltratie - gegevens stelen                       |
| IMPACT  | Impact - schade of verstoring                       |

**Risiconiveaus.** Als je niet zeker bent van het niveau, beschrijf dan alleen de impact; beheerders beoordelen het niveau.

| Niveau       | Betekenis                                                         |
| ------------ | ----------------------------------------------------------------- |
| **Kritiek**  | Volledige systeemcompromittering, of hoge kans + kritieke impact  |
| **Hoog**     | Aanzienlijke schade waarschijnlijk, of gemiddelde kans + kritieke impact |
| **Gemiddeld** | Gemiddeld risico, of lage kans + hoge impact                     |
| **Laag**     | Onwaarschijnlijk en beperkte impact                               |

## Beoordelingsproces

1. **Triage** - nieuwe inzendingen worden binnen 48 uur beoordeeld.
2. **Beoordeling** - beheerders verifiëren de haalbaarheid, wijzen een ATLAS-koppeling en dreigings-ID toe en valideren het risiconiveau.
3. **Documentatie** - controle van opmaak en volledigheid.
4. **Samenvoeging** - toegevoegd aan het dreigingsmodel en de visualisatie.

## Bronnen

- [ATLAS-website](https://atlas.mitre.org/)
- [ATLAS-technieken](https://atlas.mitre.org/techniques/)
- [ATLAS-casestudy's](https://atlas.mitre.org/studies/)

## Contact

- **Beveiligingskwetsbaarheden:** raadpleeg de [Trust-pagina](https://trust.openclaw.ai) voor meldingsinstructies, of `security@openclaw.ai`.
- **Vragen over het dreigingsmodel:** open een issue op [openclaw/trust](https://github.com/openclaw/trust/issues).
- **Algemene chat:** Discord-kanaal `#security`.

## Erkenning

Bijdragers aan het dreigingsmodel worden voor belangrijke bijdragen vermeld in de dankbetuigingen van het dreigingsmodel, de releaseopmerkingen en de OpenClaw-eregalerij voor beveiliging.

## Gerelateerd

- [Dreigingsmodel](/nl/security/THREAT-MODEL-ATLAS)
- [Incidentrespons](/nl/security/incident-response)
- [Formele verificatie](/nl/security/formal-verification)
