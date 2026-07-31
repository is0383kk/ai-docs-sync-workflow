---
read_when:
    - Je wilt een volwaardig back-uparchief voor de lokale OpenClaw-status
    - Je hebt een compacte, geverifieerde momentopname van één OpenClaw SQLite-database nodig
    - Je wilt vooraf bekijken welke paden worden opgenomen voordat je reset of verwijdert.
summary: CLI-referentie voor `openclaw backup` (archieven en SQLite-snapshots)
title: Back-up
x-i18n:
    generated_at: "2026-07-27T05:39:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: dfb5a118545589b181cede26dab72e9d029d98a1cac5cfccedd9d9cf2c56d3b5
    source_path: cli/backup.md
    workflow: 16
---

# `openclaw backup`

Maak een lokaal back-uparchief voor de status, configuratie, authenticatieprofielen, kanaal-/providerreferenties, sessies en optioneel werkruimten van OpenClaw.

```bash
openclaw backup create
openclaw backup create --output ~/Backups
openclaw backup create --dry-run --json
openclaw backup create --verify
openclaw backup create --no-include-workspace
openclaw backup create --only-config
openclaw backup verify ./2026-03-09T08-00-00.000+08-00-openclaw-backup.tar.gz
openclaw backup sqlite create --global --repository ~/Backups/openclaw-sqlite
openclaw backup sqlite create --agent main --repository ~/Backups/openclaw-sqlite
openclaw backup sqlite list --repository ~/Backups/openclaw-sqlite
openclaw backup sqlite verify ~/Backups/openclaw-sqlite/<snapshot-id>
openclaw backup sqlite verify ~/Backups/openclaw-sqlite/<snapshot-id> --scratch ~/Private/openclaw-scratch
openclaw backup sqlite restore ~/Backups/openclaw-sqlite/<snapshot-id> --target ./restored/openclaw.sqlite
```

## Opmerkingen

- Het archief bevat een ingesloten `manifest.json` met de herleide bronpaden en archiefindeling.
- De standaarduitvoer is een van een tijdstempel voorzien `.tar.gz`-archief in de huidige werkmap. Bestandsnamen met tijdstempels gebruiken de lokale tijdzone van je machine en bevatten de UTC-offset. Als de huidige werkmap zich in een bronstructuur bevindt waarvan een back-up wordt gemaakt, gebruikt OpenClaw je thuismap als standaardlocatie voor het archief.
- Bestaande archiefbestanden worden nooit overschreven. Uitvoerpaden binnen de bronstructuren voor status en werkruimten worden geweigerd om te voorkomen dat het archief zichzelf bevat.
- `openclaw backup verify <archive>` controleert of het archief precies één hoofdmanifest bevat, weigert archiefpaden die padtraversal mogelijk maken en SQLite-zijbestanden, bevestigt dat elke in het manifest gedeclareerde payload bestaat, valideert de bestandsstructuur van elke SQLite-snapshot en voert volledige integriteits- en rolcontroles uit op canonieke OpenClaw-databases. Speciale Pluginschema's blijven ondoorzichtig omdat ze mogelijk door de eigenaar gedefinieerde SQLite-mogelijkheden vereisen. `openclaw backup create --verify` voert die validatie direct na het schrijven van het archief uit.
- `openclaw backup create --only-config` maakt alleen een back-up van het actieve JSON-configuratiebestand.

## SQLite-snapshots

Gebruik `openclaw backup sqlite` wanneer je een overdraagbaar artefact voor één door OpenClaw beheerde SQLite-database nodig hebt in plaats van een breed statusarchief.

Voor het maken van een snapshot wordt precies één benoemde bron geaccepteerd:

| Opdracht                                                         | Database                    |
| --------------------------------------------------------------- | --------------------------- |
| `openclaw backup sqlite create --global --repository <dir>`     | Gedeelde OpenClaw-status    |
| `openclaw backup sqlite create --agent <id> --repository <dir>` | Eén database per agent      |

De opslagplaats bevat één map per vastgelegde snapshot. Elke snapshotmap bevat precies:

- `manifest.json`
- `database.sqlite`

Bij het maken van een snapshot wordt de actieve database vóór het lezen geverifieerd, wordt de online back-up-API van SQLite gebruikt om de vastgelegde WAL-status vast te leggen zonder één lange leestransactie open te houden, wordt de actieve database gesloten, wordt de privékopie gecomprimeerd met `VACUUM`, wordt de gegenereerde database opnieuw geverifieerd en wordt de voltooide map gepubliceerd zonder bestaande paden te overschrijven. Bij globale snapshots worden tijdelijke rijen uit de afleveringswachtrij vóór Compaction verwijderd, zodat verwijderde wachtrijpayloads niet in vrije pagina's behouden blijven.

Kopieer actieve `.sqlite`-, `-wal`-, `-shm`- of `-journal`-bestanden niet als overdraagbaar artefact. Kopieer alleen voltooide snapshotmappen.

SQLite-snapshots kunnen authenticatieprofielen, sessiestatus, Pluginstatus en andere gevoelige records bevatten. Bescherm opslagplaatsen met dezelfde machtigingen, versleuteling, bewaarbeleidsregels en bestemmingsbeperkingen als de actieve OpenClaw-statusmap.

### Verifiëren en herstellen

```bash
openclaw backup sqlite verify <snapshot-directory>
openclaw backup sqlite restore <snapshot-directory> --target <new-database-path>
```

Bij verificatie worden de strikte manifeststructuur, artefactgrootte en SHA-256, SQLite-integriteit, externe sleutels, schemaversie, databaserol en -eigenaar, en door OpenClaw beheerde indexdefinities gecontroleerd.

Bij verificatie wordt een privé, aan de inhoud gekoppelde kopie gevalideerd, zodat racecondities rond padnamen niet de bytes kunnen verwisselen die SQLite inspecteert. Standaard wordt die tijdelijke kopie naast de snapshotopslagplaats gemaakt en verwijderd voordat de opdracht terugkeert. De faseringshoofdmap en de bovenliggende mappen ervan moeten voorkomen dat andere gebruikers deze kunnen vervangen. POSIX-hoofdmappen moeten eigendom zijn van de huidige gebruiker en mogen niet schrijfbaar zijn voor de groep of iedereen; sticky bovenliggende mappen zoals `/tmp` worden geaccepteerd voor onderliggende mappen die eigendom zijn van de gebruiker. macOS-ACL-toekenningen die de fasering toegankelijk of vervangbaar maken, worden geweigerd. Windows-hoofdmappen en bovenliggende mappen moeten eigendom zijn van de huidige gebruiker of een vertrouwde OS-principal, met ACL's die niet-vertrouwde toegang tot de fasering weigeren. Geef voor een alleen-lezen koppelpunt of netwerkshare `--scratch <existing-private-directory>` door op opslag met gelijkwaardige versleuteling en bestemmingscontroles.

Bij het maken van snapshots worden dezelfde controles voor eigenaar, ACL, bovenliggende mappen en padidentiteit op de opslagplaats toegepast voordat databasebytes worden klaargezet of gepubliceerd.

Bij herstel wordt de verificatie herhaald en alleen naar een nieuw doel geschreven. Een bestaand doel of bestaande `-wal`-, `-shm`- of `-journal`-zijbestanden worden geweigerd en een actieve OpenClaw-database wordt nooit ter plaatse vervangen. Voor de bovenliggende doelmap gelden dezelfde padbeveiligingsvereisten als voor de tijdelijke verificatieopslag. Het activeren van een herstelde database blijft een expliciete offline stap voor de beheerder.

Snapshotopslagplaatsen zijn lokale mappen. Planning, uploaden, bewaring, incrementele WAL-bundels, failover en herstel bij het opstarten vallen opzettelijk buiten deze opdracht.

## Waarvan een back-up wordt gemaakt

`openclaw backup create` plant bronnen vanuit je lokale OpenClaw-installatie:

- De statusmap (meestal `~/.openclaw`)
- Het pad van het actieve configuratiebestand
- De herleide map `credentials/` wanneer deze buiten de statusmap bestaat
- Werkruimtemappen die uit de huidige configuratie zijn afgeleid, tenzij je `--no-include-workspace` doorgeeft

Authenticatieprofielen en andere runtimestatus per agent bevinden zich in SQLite onder de statusmap (`agents/<agentId>/agent/openclaw-agent.sqlite`) en vallen daardoor automatisch onder de back-upvermelding voor de status.

`--only-config` slaat de status, referentiemap en werkruimtedetectie over en archiveert alleen het pad van het actieve configuratiebestand.

OpenClaw canonicaliseert paden voordat het archief wordt samengesteld: als de configuratie, referentiemap of een werkruimte zich al in de statusmap bevindt, worden deze niet als afzonderlijke back-upbronnen op het hoogste niveau gedupliceerd. Ontbrekende paden worden overgeslagen.

Tijdens het maken van het archief sluit OpenClaw bekende paden die actief worden gewijzigd uit voordat `tar` ze leest. Dit voorkomt racecondities tussen de vastgelegde grootte van een bestand en gelijktijdige schrijfbewerkingen. Het filter past onder elke statusmap waarvan een back-up wordt gemaakt de volgende regels toe, relatief aan die statusmap:

| Bereik relatief aan de statusmap                | Overgeslagen bestandsachtervoegsels |
| ----------------------------------------------- | ----------------------------------- |
| `sessions/**`                                | `.jsonl`, `.log`              |
| `agents/<agentId>/sessions/**`               | `.jsonl`, `.log`              |
| `cron/runs/**`                               | `.jsonl`, `.log`              |
| `logs/**`                                    | `.jsonl`, `.log`              |
| `delivery-queue/**`                          | `.json`, `.delivered`, `.tmp` |
| `session-delivery-queue/**`                  | `.json`, `.delivered`, `.tmp` |
| Elk pad onder de statusmap waarvan een back-up wordt gemaakt | `.sock`, `.pid`, `.tmp`       |

Deze regels filteren geen werkruimtebestanden buiten de statusmap. Ze laten ook voltooide transcript- en logboekbestanden weg die met de tabel overeenkomen; bewaar die records daarom indien nodig afzonderlijk. `skippedVolatileCount` in het JSON-resultaat meldt hoeveel bestanden opzettelijk zijn weggelaten.

SQLite-databases onder de statusmap worden vastgelegd met de online back-up-API van SQLite en offline gecomprimeerd met `VACUUM`, zodat restanten van verwijderde pagina's niet in het archief terechtkomen en actieve WAL-/SHM-bestanden niet worden gekopieerd. Een database die eigendom is van een Plugin en niet-beschikbare, door de eigenaar gedefinieerde SQLite-mogelijkheden vereist, wordt veilig geweigerd in plaats van terug te vallen op een rechtstreekse bestandskopie. SQLite-bestanden die via werkruimteback-ups worden opgenomen, worden als werkruimtebestanden gekopieerd en vallen niet onder de garantie voor Compaction.

Geïnstalleerde Pluginbron- en manifestbestanden onder de `extensions/`-structuur van de statusmap worden opgenomen, maar hun geneste `node_modules/`-afhankelijkheidsstructuren worden overgeslagen als opnieuw op te bouwen installatieartefacten. Gebruik na het herstellen van een archief `openclaw plugins update <id>` of installeer opnieuw met `openclaw plugins install <spec> --force` als een herstelde Plugin ontbrekende afhankelijkheden meldt.

Door het installatieprogramma beheerde en opnieuw op te bouwen runtimehoofdmappen onder de statusmap worden eveneens overgeslagen: `dev/`, `git/`, `npm/`, de verouderde `npm-runtime/` en `tools/`. Deze bevatten beheerde check-outs, pakketstructuren en gedownloade runtimes in plaats van gezaghebbende gebruikersstatus; installeer of werk de bijbehorende runtime of Plugin na herstel opnieuw bij. Een expliciet geconfigureerd configuratiebestand, een referentiemap of een werkruimte binnen een van deze hoofdmappen blijft opgenomen.

## Gedrag bij een ongeldige configuratie

`openclaw backup` omzeilt de normale configuratiecontrole vooraf, zodat deze opdracht ook tijdens herstel kan helpen. Werkruimtedetectie is afhankelijk van een geldige configuratie, dus `openclaw backup create` stopt onmiddellijk wanneer het configuratiebestand bestaat maar ongeldig is en werkruimteback-up nog is ingeschakeld.

Voer voor een gedeeltelijke back-up in die situatie de opdracht opnieuw uit met `--no-include-workspace`: hierdoor blijven de status, configuratie en externe referentiemap binnen het bereik, terwijl werkruimtedetectie volledig wordt overgeslagen.

`--only-config` werkt ook wanneer de configuratie ongeldig is, omdat deze opdracht de configuratie niet parseert voor werkruimtedetectie.

## Grootte en prestaties

OpenClaw hanteert geen ingebouwde maximale back-upgrootte of limiet voor de grootte per bestand. Een archiefschrijfbewerking die vijf minuten lang geen gegevens produceert, mislukt en verwijdert het gedeeltelijke tijdelijke bestand in plaats van onbeperkt te blijven hangen. Praktische limieten worden verder bepaald door:

- Beschikbare ruimte voor het schrijven van het tijdelijke archief plus het definitieve archief
- De tijd om grote werkruimtestructuren te doorlopen en naar een `.tar.gz` te comprimeren
- De tijd om het archief opnieuw te scannen met `--verify` of `openclaw backup verify`
- Het gedrag van het doelbestandssysteem: OpenClaw vereist publicatie via harde koppelingen zonder overschrijven, zodat een definitief archiefpad nooit een nog niet voltooide kopie toont; niet-ondersteunde bestandssystemen mislukken met een bruikbare foutmelding

Als de bevestiging van de duurzaamheid van de definitieve map na publicatie mislukt, meldt de opdracht een fout, maar behoudt deze de volledige definitieve vermelding om niet het risico te lopen een gelijktijdige vervanging te verwijderen.

Grote werkruimten zijn doorgaans de belangrijkste oorzaak van de archiefgrootte. Gebruik `--no-include-workspace` voor een kleinere/snellere back-up of `--only-config` voor het kleinste archief.

## Gerelateerd

- [CLI-referentie](/nl/cli)
