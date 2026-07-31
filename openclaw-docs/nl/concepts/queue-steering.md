---
read_when:
    - Uitleg over hoe steer zich gedraagt terwijl een agent tools gebruikt
    - Gedrag van de wachtrij voor actieve uitvoeringen of integratie van runtime-aansturing wijzigen
    - Vergelijking van sturing met de wachtrijmodi followup, collect en interrupt
summary: Hoe sturing tijdens actieve runs berichten in wachtrijen plaatst bij runtimegrenzen
title: Aansturingswachtrij
x-i18n:
    generated_at: "2026-07-27T05:49:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 131f04f19934b9b1f6dd8ffb2cf2428950c319483abdc2ccdecec741809cda2a
    source_path: concepts/queue-steering.md
    workflow: 16
---

Wanneer een normale prompt binnenkomt terwijl een sessierun al streamt en de wachtrijmodus `steer` is (de standaardinstelling, geen configuratie nodig), probeert OpenClaw die prompt naar de actieve runtime te sturen. OpenClaw en de systeemeigen Codex-app-serverharnas implementeren de bezorgingsdetails op verschillende manieren.

Deze pagina behandelt sturing via de wachtrijmodus voor normale inkomende berichten in de modus `steer`. In de modus `followup` of `collect` slaan normale berichten dit pad over en wachten ze totdat de actieve run is voltooid. Zie [Sturen](/nl/tools/steer) voor de expliciete opdracht `/steer <message>`.

## Runtimegrens

Sturing onderbreekt geen toolaanroep die al wordt uitgevoerd. OpenClaw controleert op sturingsberichten in de wachtrij bij modelgrenzen:

1. De assistent vraagt om toolaanroepen.
2. OpenClaw voert de batch toolaanroepen van het huidige assistentbericht uit.
3. OpenClaw genereert de gebeurtenis voor het einde van de beurt.
4. OpenClaw verwerkt de sturingsberichten in de wachtrij.
5. OpenClaw voegt die berichten als gebruikersberichten toe vóór de volgende LLM-aanroep.

Hierdoor blijven toolresultaten gekoppeld aan het assistentbericht waarin erom werd gevraagd, waarna de volgende modelaanroep de nieuwste gebruikersinvoer kan zien.

Het systeemeigen Codex-app-serverharnas stelt `turn/steer` beschikbaar in plaats van de interne sturingswachtrij van de OpenClaw-runtime. OpenClaw bundelt prompts in de wachtrij gedurende het geconfigureerde stiltevenster en verzendt vervolgens één `turn/steer`-verzoek met alle verzamelde gebruikersinvoer in volgorde van binnenkomst.

Codex-review- en handmatige Compaction-beurten wijzen sturing binnen dezelfde beurt af. Wanneer een runtime in de modus `steer` geen sturing kan accepteren, wacht OpenClaw totdat de actieve run is voltooid voordat de prompt wordt gestart.

## Modi

| Modus        | Gedrag tijdens actieve run                                    | Later gedrag                                                                      |
| ----------- | ------------------------------------------------------ | ----------------------------------------------------------------------------------- |
| `steer`     | Stuurt de prompt waar mogelijk naar de actieve runtime. | Wacht totdat de actieve run is voltooid als sturing niet beschikbaar is.                      |
| `followup`  | Stuurt niet.                                        | Voert berichten in de wachtrij later uit nadat de actieve run is beëindigd.                               |
| `collect`   | Stuurt niet.                                        | Voegt compatibele berichten in de wachtrij na het debouncevenster samen tot één latere beurt. |
| `interrupt` | Breekt de actieve run af in plaats van deze te sturen.          | Start het nieuwste bericht na het afbreken.                                           |

## Voorbeeld van een berichtenpiek

Als vier gebruikers berichten verzenden terwijl de agent een toolaanroep uitvoert:

- Bij het standaardgedrag ontvangt de actieve runtime alle vier de berichten in volgorde van binnenkomst vóór de volgende modelbeslissing. OpenClaw verwerkt ze bij de volgende modelgrens; Codex ontvangt ze als één gebundelde `turn/steer`.
- Met `/queue collect` stuurt OpenClaw niet. Het wacht totdat de actieve run is beëindigd en maakt vervolgens na het debouncevenster een vervolgbeurt met compatibele berichten in de wachtrij.
- Met `/queue interrupt` breekt OpenClaw de actieve run af en start het nieuwste bericht in plaats van te sturen.

## Bereik

Sturing is altijd gericht op de huidige actieve sessierun. Er wordt geen nieuwe sessie gemaakt, het toolbeleid van de actieve run wordt niet gewijzigd en berichten worden niet per afzender gesplitst. In kanalen met meerdere gebruikers bevatten inkomende prompts al context over de afzender en route, zodat de volgende modelaanroep kan zien wie elk bericht heeft verzonden.

Gebruik `followup` of `collect` als je wilt dat berichten standaard in de wachtrij worden geplaatst in plaats van de actieve run te sturen. Gebruik `interrupt` wanneer de nieuwste prompt de actieve run moet vervangen.

## Debounce

De ingebouwde debounce van de wachtrij is van toepassing op de bezorging van `followup` en `collect` in de wachtrij. In de modus `steer` met het systeemeigen Codex-harnas stelt deze ook het stiltevenster in voordat gebundelde `turn/steer` worden verzonden. Voor OpenClaw gebruikt actieve sturing zelf de debouncetimer niet, omdat OpenClaw berichten van nature bundelt tot de volgende modelgrens.

## Gerelateerd

- [Opdrachtwachtrij](/nl/concepts/queue)
- [Sturen](/nl/tools/steer)
- [Berichten](/nl/concepts/messages)
- [Agentlus](/nl/concepts/agent-loop)
