---
read_when:
    - Een fout met een nieuwere databaseschema-versie diagnosticeren
    - Databasecompatibiliteit controleren vóór een update of downgrade
    - Een database herstellen voor een oudere OpenClaw-release
summary: Locaties van OpenClaw SQLite-databases, schemaversies, integriteitscontroles en herstel na een downgrade
title: Databaseschema's
x-i18n:
    generated_at: "2026-07-27T05:15:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 73993e2c593ba460784108aedef70bbfb499e525c709d6d6bdd956ccf93e0ddc
    source_path: reference/database-schemas.md
    workflow: 16
---

OpenClaw slaat control-plane-status op in een globale SQLite-database en agentgegevens in één SQLite-database per agent. Schemamigraties worden voorwaarts uitgevoerd wanneer een database wordt geopend. Oudere OpenClaw-builds weigeren databases die door een nieuwer schema zijn geschreven.

## Database-indeling

| Bereik               | Standaardpad                                               | Inhoud                                                                                                      |
| -------------------- | ---------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| Globaal control plane | `~/.openclaw/state/openclaw.sqlite`                        | Gedeelde configuratiestatus, registers, goedkeuringen, pluginstatus en gedeelde runtimestatus                |
| Data plane per agent | `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite` | Sessies, transcripties, geheugenindexen, authenticatiestatus, gespreksstatus en agentgebonden runtimestatus |

Enkele functies met een hoog volume of een specifieke levenscyclus gebruiken afzonderlijke SQLite-opslag, waaronder het taakregister en trajectgegevens.

## Versiecontract

Elke database legt het schema op twee plaatsen vast:

- `PRAGMA user_version` is de SQLite-schemaversie.
- De primaire rij `schema_meta` legt `role`, `agent_id`, `schema_version` en `app_version` vast. `app_version` is de OpenClaw-build die de schemametadata het laatst heeft geschreven.

OpenClaw past uitsluitend voorwaartse migraties toe wanneer het een oudere ondersteunde database opent. Het weigert een database waarvan `user_version` nieuwer is dan de actieve build en meldt een fout `newer schema version`. De Gateway controleert vóór het opstarten alle geregistreerde databases. `openclaw update` weigert ook een pakket- of brondoel waarvan de opgegeven schemaondersteuning ouder is dan een database op schijf. Doelpakketten die zijn gepubliceerd voordat schemametadata werden toegevoegd, kunnen niet vooraf worden gecontroleerd.

Als je OpenClaw handmatig via npm installeert, omzeil je de beveiliging van de updater. Controles bij het openen van de database weigeren nog steeds een incompatibele build.

## Geschiedenis van het agentschema

| Versie | Wijziging                                                                                                                                                                                                                                                      | Eerste release                                   |
| ------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------ |
| 1      | Eerste opslag per agent ([#88349](https://github.com/openclaw/openclaw/pull/88349))                                                                                                                                                                            | `v2026.5.30-beta.1`, stabiel tot en met `v2026.7.1` |
| 2      | Identiteit van geheugenindex ([#104449](https://github.com/openclaw/openclaw/pull/104449))                                                                                                                                                                     | `v2026.7.2-beta.1`                               |
| 4      | Sessies en transcripties verplaatst naar SQLite ([#98236](https://github.com/openclaw/openclaw/pull/98236))                                                                                                                                                    | `v2026.7.2-beta.1`                               |
| 5-6    | Actualiteit van terminals en statuslevenscyclus ([#104859](https://github.com/openclaw/openclaw/pull/104859))                                                                                                                                                   | `v2026.7.2-beta.1`                               |
| 7      | Projectie van levenscyclusstatus per item ([#106151](https://github.com/openclaw/openclaw/pull/106151))                                                                                                                                                         | `v2026.7.2-beta.1`                               |
| 8      | Herkomst van sessie per transcriptie ([#106766](https://github.com/openclaw/openclaw/pull/106766))                                                                                                                                                             | `v2026.7.2-beta.2`                               |
| 9      | Tabellen `STRICT` ([#108663](https://github.com/openclaw/openclaw/pull/108663))                                                                                                                                                                      | `v2026.7.2-beta.2`                               |
| 10     | Gematerialiseerde paden van actieve transcripties ([#108851](https://github.com/openclaw/openclaw/pull/108851))                                                                                                                                                | Niet uitgebracht                                 |
| 11     | Leases, duurzame levering, gespreksadressen en Heartbeat-resultaten ([#109636](https://github.com/openclaw/openclaw/pull/109636), [#95838](https://github.com/openclaw/openclaw/pull/95838), [#109999](https://github.com/openclaw/openclaw/pull/109999)) | Niet uitgebracht                                 |

Versie 3 was een niet-uitgebrachte ontwikkelstap die in versie 4 is opgenomen.

## Geschiedenis van het statusschema

| Versie | Wijziging                                                                                                              | Eerste release       |
| ------ | ---------------------------------------------------------------------------------------------------------------------- | -------------------- |
| 1      | Eerste database voor gedeelde status                                                                                   | `v2026.5.30-beta.1`   |
| 2      | Controlegebeurtenissen voor berichten met alleen metadata ([#103903](https://github.com/openclaw/openclaw/pull/103903)) | `v2026.7.2-beta.1`   |
| 3      | Tabellen `STRICT` en versterking tegen schema-afwijkingen ([#108663](https://github.com/openclaw/openclaw/pull/108663)) | `v2026.7.2-beta.2`   |
| 4      | Herkomst van sessiebewaking vervangt gecodeerde sentinelrijen                                                         | Niet uitgebracht     |

## Integriteitscontroles

| Wanneer                                     | Controle                                                                    |
| ------------------------------------------- | --------------------------------------------------------------------------- |
| Bij elke opening                            | Valideer de tabel `schema_meta` en de primaire metadatarij             |
| Vóór een wachtende migratie                 | Voer een volledige integriteits-, refererende-sleutel-, rol-, schema- en indexscan uit |
| Achtergrondverificatie van de Gateway       | Voer de volledige scan ongeveer eenmaal per dag uit en registreer de resultaten |
| Doctor, back-upverificatie en Compaction    | Voer de volledige scan uit voordat de database wordt geaccepteerd of herschreven |

De voorafgaande controle van de Gateway leest alleen schemakoppen. De achtergrondverificatie voert de tragere volledige scan uit voor databases waarvoor geen migratie nodig is.
Quarantainebeslissingen worden uitsluitend opgeslagen in een afzonderlijke opslag `openclaw-quarantine.sqlite`, zodat ze schade aan de databases die in quarantaine worden geplaatst, overleven. Verificatieresultaten worden geregistreerd.

## Probleemoplossing

### Waarom je na de update naar 2026.7.2 niet terug kunt

Elke release tot en met `v2026.7.1` gebruikte agentschema 1 en statusschema 1. De releasereeks 2026.7.2 (vanaf `v2026.7.2-beta.1`) migreert je databases bij de eerste start voorwaarts. Die migratie is eenrichtingsverkeer: de gegevens worden naar het nieuwere schema herschreven en een daaropvolgende installatie van een oudere OpenClaw-versie maakt dit niet ongedaan. De oudere build weigert te starten met een fout `newer schema version` die de build vermeldt waartoe de database behoort.

Het downgraden van het binaire bestand downgradet de gegevens nooit. Als je na de update een release ouder dan 2026.7.2 moet uitvoeren, heb je drie opties:

1. Herstel een back-up die vóór de update is gemaakt. [Maak en verifieer back-ups](/nl/cli/backup) vóór grote updates.
2. Voer de oudere build uit met een afzonderlijke statusmap (`OPENCLAW_STATE_DIR`). Deze begint opnieuw; je gemigreerde gegevens blijven onaangeroerd voor wanneer je terugkeert naar de nieuwere build.
3. Volg de onderstaande procedure voor handmatig downgraden. Deze wordt niet ondersteund en brengt zonder een geverifieerde back-up risico op gegevensverlies met zich mee.

Sinds 2026.7.2 weigert `openclaw update` een release te installeren die je huidige databases niet kan openen, zodat de updater je niet in deze situatie brengt. Als je handmatig via npm een oudere versie installeert, omzeil je deze beveiliging; de databases weigeren het oude binaire bestand nog steeds, maar pas nadat het is geïnstalleerd.

### De Gateway weigert te starten vanwege een fout over een nieuwere schemaversie

Een nieuwere OpenClaw-build heeft je databases geschreven en de actieve build is ouder. De fout en het opstartlogboek van de Gateway vermelden de build waartoe de database behoort (`app_version`). Installeer die versie of een nieuwere versie, of gebruik een van de bovenstaande opties. Bewerk de database niet om de fout te onderdrukken.

### Een database wordt in quarantaine geplaatst nadat de integriteitsverificatie is mislukt

De achtergrondverificatie heeft aangetoond dat het bestand beschadigd is en elke openingspoging mislukt nu onmiddellijk in plaats van opnieuw te scannen. Herstel de database vanuit een back-up of repareer deze en voer vervolgens `openclaw doctor --fix` uit om de quarantaineregistratie te wissen. Doctor meldt een expliciete fout als de quarantaineregistratie zelf niet kan worden gewist; voer de opdracht opnieuw uit totdat deze meldt dat alles in orde is.

## Downgrades worden niet ondersteund

Handmatige schemadowngrades zijn bedoeld voor agents en beheerders die het risico accepteren. [Maak en verifieer een back-up](/nl/cli/backup) voordat je een database bewerkt. Stop de Gateway en elk proces dat de database kan openen.

De algemene procedure is:

1. Lees het schema en de migraties van de doelrelease.
2. Verwijder in één transactie elke tabel, index, trigger en kolom die na de doelversie is geïntroduceerd.
3. Stel `PRAGMA user_version` en `schema_meta.schema_version` in op de doelversie.
4. Voer de volledige databaseverificatie van de doelrelease uit voordat je de Gateway start.

### Voorbeeld: agentschema 11 naar 9

Schema 10 voegde de projectie van actieve transcripties toe. Schema 11 voegde leases, duurzame levering, status van gespreksadressen en Heartbeat-resultaten toe. QMD-coördinatie gebruikt rijen in `state_leases`; er is geen afzonderlijke QMD-tabel die moet worden behouden.

Voer gelijkwaardige SQL uit op elke betrokken database per agent nadat je het exacte schema hebt geïnspecteerd waarmee deze is geschreven:

```sql
BEGIN IMMEDIATE;

DROP TABLE IF EXISTS heartbeat_outcomes;
DROP TABLE IF EXISTS conversation_deliveries;
DROP TABLE IF EXISTS state_leases;
DROP TABLE IF EXISTS session_transcript_active_events;

ALTER TABLE session_transcript_index_state DROP COLUMN active_event_count;
ALTER TABLE session_transcript_index_state DROP COLUMN active_message_count;
ALTER TABLE conversations DROP COLUMN delivery_target;

PRAGMA user_version = 9;
UPDATE schema_meta
SET schema_version = 9,
    updated_at = unixepoch('now') * 1000
WHERE meta_key = 'primary';

COMMIT;
```

Hiermee wordt status uit versies 10-11 verwijderd, waaronder lopende leveringsbewerkingen, leases, Heartbeat-resultaten en de afgeleide projectie van actieve transcripties. Als een downgrade mislukt, moet je herstellen vanuit de geverifieerde back-up.
