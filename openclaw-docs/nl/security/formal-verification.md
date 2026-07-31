---
permalink: /security/formal-verification/
read_when:
    - Formele garanties of beperkingen van het beveiligingsmodel beoordelen
    - TLA+/TLC-beveiligingsmodelcontroles reproduceren of bijwerken
summary: Machinaal gecontroleerde beveiligingsmodellen voor de paden met het hoogste risico van OpenClaw.
title: Formele verificatie (beveiligingsmodellen)
x-i18n:
    generated_at: "2026-07-27T05:22:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 185ee5c1cff7325f10827330c0c7e55ddc3ca40caf6088d4c930ae5e090d6b27
    source_path: security/formal-verification.md
    workflow: 16
---

De formele beveiligingsmodellen van OpenClaw (momenteel TLA+/TLC) leveren een machinaal gecontroleerd argument dat specifieke paden met het hoogste risico — autorisatie, sessie-isolatie, toolafscherming en veiligheid bij onjuiste configuratie — het beoogde beleid afdwingen, onder expliciet vermelde aannames.

> Opmerking: sommige oudere links verwijzen mogelijk naar de vorige projectnaam.

## Wat dit is

Een uitvoerbare, door aanvallers aangestuurde regressietestsuite voor beveiliging:

- Elke claim heeft een uitvoerbare modelcontrole over een eindige toestandsruimte.
- Veel claims hebben een gekoppeld negatief model dat een tegenvoorbeeldtrace produceert voor een realistische foutcategorie.

Dit is **geen** bewijs dat OpenClaw in alle opzichten veilig is en het verifieert niet de volledige TypeScript-implementatie.

## Waar de modellen staan

De modellen worden onderhouden in een afzonderlijke repository: [vignesh07/openclaw-formal-models](https://github.com/vignesh07/openclaw-formal-models).

<Note>
Die repository is momenteel onbereikbaar (GitHub retourneert op het moment van schrijven "Repository not found"). Als deze voor jou nog steeds niet werkt, vraag dan in de OpenClaw-kanalen voor maintainers naar de huidige locatie voordat je aanneemt dat de modellen zijn verwijderd.
</Note>

## Kanttekeningen

- Dit zijn modellen, niet de volledige TypeScript-implementatie — afwijkingen tussen model en code zijn mogelijk.
- De resultaten worden begrensd door de toestandsruimte die TLC verkent. Groen impliceert geen beveiliging buiten de gemodelleerde aannames en grenzen.
- Sommige claims berusten op expliciete aannames over de omgeving (bijvoorbeeld een correcte implementatie en correcte configuratie-invoer).

## Resultaten reproduceren

Kloon de modellenrepository en voer TLC uit:

```bash
git clone https://github.com/vignesh07/openclaw-formal-models
cd openclaw-formal-models

# Java 11+ vereist (TLC wordt uitgevoerd op de JVM).
# De repository bevat een vastgezette tla2tools.jar en biedt bin/tlc plus Make-doelen.

make <target>
```

Er is nog geen CI-integratie terug naar deze repository; een toekomstige iteratie zou door CI uitgevoerde modellen met openbare artefacten (tegenvoorbeeldtraces, uitvoerlogboeken) of een gehoste workflow voor "dit model uitvoeren" kunnen toevoegen voor kleine begrensde controles.

## Claims en doelen

### Gateway-blootstelling en onjuiste configuratie van een open Gateway

**Claim:** binding buiten loopback zonder authenticatie kan compromittering op afstand mogelijk maken en vergroot de blootstelling; volgens de aannames van het model houdt een token/wachtwoord niet-geverifieerde aanvallers tegen.

| Resultaat      | Doelen                                                           |
| -------------- | ---------------------------------------------------------------- |
| Groen          | `make gateway-exposure-v2`, `make gateway-exposure-v2-protected` |
| Rood (verwacht) | `make gateway-exposure-v2-negative`                              |

Zie ook `docs/gateway-exposure-matrix.md` in de modellenrepository.

### Uitvoerpijplijn van Node (mogelijkheid met het hoogste risico)

**Claim:** `exec host=node` vereist (a) een toelatingslijst voor Node-opdrachten plus gedeclareerde opdrachten en (b) directe goedkeuring indien geconfigureerd; in het model worden goedkeuringen van tokens voorzien om hergebruik te voorkomen.

| Resultaat      | Doelen                                                          |
| -------------- | --------------------------------------------------------------- |
| Groen          | `make nodes-pipeline`, `make approvals-token`                   |
| Rood (verwacht) | `make nodes-pipeline-negative`, `make approvals-token-negative` |

### Koppelingsopslag (DM-afscherming)

**Claim:** koppelingsverzoeken respecteren de TTL en limieten voor openstaande verzoeken.

| Resultaat      | Doelen                                               |
| -------------- | ---------------------------------------------------- |
| Groen          | `make pairing`, `make pairing-cap`                   |
| Rood (verwacht) | `make pairing-negative`, `make pairing-cap-negative` |

### Afscherming van inkomend verkeer (vermeldingen en omzeiling via besturingsopdrachten)

**Claim:** in groepscontexten waarin een vermelding vereist is, kan een niet-geautoriseerde besturingsopdracht de afscherming via vermeldingen niet omzeilen.

| Resultaat      | Doelen                         |
| -------------- | ------------------------------ |
| Groen          | `make ingress-gating`          |
| Rood (verwacht) | `make ingress-gating-negative` |

### Routering en isolatie van sessiesleutels

**Claim:** DM's van verschillende gesprekspartners worden niet in dezelfde sessie samengevoegd, tenzij ze expliciet zijn gekoppeld of zo zijn geconfigureerd.

| Resultaat      | Doelen                            |
| -------------- | --------------------------------- |
| Groen          | `make routing-isolation`          |
| Rood (verwacht) | `make routing-isolation-negative` |

## v1++-modellen: gelijktijdigheid, nieuwe pogingen en correctheid van traces

Vervolgmodellen die de getrouwheid rond storingsmodi uit de praktijk aanscherpen: niet-atomaire updates, nieuwe pogingen en berichtfan-out.

### Gelijktijdigheid en idempotentie van de koppelingsopslag

**Claim:** de koppelingsopslag dwingt `MaxPending` en idempotentie af, zelfs bij vervlechtingen — controleren en vervolgens schrijven moet atomair/vergrendeld zijn en vernieuwen mag geen duplicaten maken. Concreet: gelijktijdige verzoeken kunnen `MaxPending` voor een kanaal niet overschrijden en herhaalde verzoeken/vernieuwingen voor dezelfde `(channel, sender)` maken geen dubbele actieve openstaande rijen.

| Resultaat      | Doelen                                                                                                                                                                      |
| -------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Groen          | `make pairing-race` (atomaire/vergrendelde limietcontrole), `make pairing-idempotency`, `make pairing-refresh`, `make pairing-refresh-race`                                              |
| Rood (verwacht) | `make pairing-race-negative` (niet-atomaire limietrace bij starten/vastleggen), `make pairing-idempotency-negative`, `make pairing-refresh-negative`, `make pairing-refresh-race-negative` |

### Tracecorrelatie en idempotentie van inkomend verkeer

**Claim:** opname behoudt tracecorrelatie bij fan-out en is idempotent bij nieuwe pogingen van providers. Wanneer één externe gebeurtenis meerdere interne berichten wordt, behoudt elk onderdeel dezelfde trace-/gebeurtenisidentiteit; nieuwe pogingen worden niet dubbel verwerkt; als gebeurtenis-ID's van de provider ontbreken, valt deduplicatie terug op een veilige sleutel (bijvoorbeeld een trace-ID) om te voorkomen dat afzonderlijke gebeurtenissen worden verwijderd.

| Resultaat      | Doelen                                                                                                                                      |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| Groen          | `make ingress-trace`, `make ingress-trace2`, `make ingress-idempotency`, `make ingress-dedupe-fallback`                                     |
| Rood (verwacht) | `make ingress-trace-negative`, `make ingress-trace2-negative`, `make ingress-idempotency-negative`, `make ingress-dedupe-fallback-negative` |

### Routering: voorrang van dmScope en identityLinks

**Claim:** de voorrang van `dmScope` en identiteitskoppelingen gedragen zich deterministisch: het standaardbereik `main` deelt één doorlopende sessie over de DM's van één eigenaar (de standaard voor persoonlijke agents), terwijl elk geconfigureerd isolerend bereik (`per-peer`, `per-channel-peer`, `per-account-channel-peer`) DM-sessies strikt gescheiden houdt. Kanaalspecifieke `dmScope` hebben voorrang op globale standaardwaarden; `identityLinks` voegen sessies alleen samen binnen expliciet gekoppelde groepen, niet tussen niet-gerelateerde gesprekspartners. Voor inboxen met meerdere gebruikers wordt verwacht dat een isolerend bereik wordt ingeschakeld (de beveiligingsaudit van de runtime beveelt dit aan wanneer deze DM-verkeer van meerdere gebruikers detecteert).

| Resultaat      | Doelen                                                                    |
| -------------- | ------------------------------------------------------------------------- |
| Groen          | `make routing-precedence`, `make routing-identitylinks`                   |
| Rood (verwacht) | `make routing-precedence-negative`, `make routing-identitylinks-negative` |

## Gerelateerd

- [Dreigingsmodel](/nl/security/THREAT-MODEL-ATLAS)
- [Bijdragen aan het dreigingsmodel](/nl/security/CONTRIBUTING-THREAT-MODEL)
- [Incidentrespons](/nl/security/incident-response)
