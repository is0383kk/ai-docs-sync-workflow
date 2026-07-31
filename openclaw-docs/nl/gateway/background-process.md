---
read_when:
    - Achtergrondgedrag voor uitvoering toevoegen of wijzigen
    - Langlopende exec-taken debuggen
summary: Uitvoering op de achtergrond en procesbeheer
title: Uitvoering op de achtergrond en procestool
x-i18n:
    generated_at: "2026-07-27T06:13:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 37cb65ddf67227e32be972e77d16b9835d592120ecd12e041d05c48536fd2204
    source_path: gateway/background-process.md
    workflow: 16
---

OpenClaw voert shellopdrachten uit via de tool `exec` en houdt langlopende taken in het geheugen. De tool `process` beheert die achtergrondsessies.

## Tool exec

Parameters:

| Parameter    | Beschrijving                                                                                                                                                |
| ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `command`    | Vereist. Uit te voeren shellopdracht.                                                                                                                       |
| `workdir`    | Werkmap; laat weg om de standaard-cwd te gebruiken.                                                                                                        |
| `env`        | Extra omgevingsvariabelen voor de opdracht.                                                                                                                |
| `yieldMs`    | Aantal milliseconden dat wordt gewacht voordat de opdracht naar de achtergrond gaat (standaard 10000).                                                     |
| `background` | Voer onmiddellijk op de achtergrond uit.                                                                                                                   |
| `timeout`    | Time-out in seconden (standaard `tools.exec.timeoutSeconds`); beëindigt het proces wanneer deze verstrijkt. Stel `timeout: 0` in om de time-out van het exec-proces voor die aanroep uit te schakelen. |
| `pty`        | Voer indien beschikbaar uit in een pseudoterminal (CLI's die een TTY vereisen, codeeragents).                                                              |
| `elevated`   | Voer buiten de sandbox uit als de verhoogde modus is ingeschakeld/toegestaan (standaard `gateway`, of `node` wanneer het exec-doel `node` is).                              |
| `host`       | Exec-doel: `auto`, `sandbox`, `gateway` of `node`.                                                                                                      |
| `node`       | Node-id/-naam, gebruikt met `host: "node"`.                                                                                                             |

Gedrag:

- Voorgronduitvoeringen retourneren uitvoer rechtstreeks.
- Wanneer een uitvoering naar de achtergrond gaat (expliciet of via de time-out `yieldMs`), retourneert de tool `status: "running"` + `sessionId` en een kort laatste deel van de uitvoer.
- Uitvoeringen op de achtergrond en `yieldMs` nemen `tools.exec.timeoutSeconds` over, tenzij de aanroep expliciet een `timeout` doorgeeft.
- Uitvoer blijft in het geheugen totdat de sessie wordt opgevraagd of gewist.
- Als de tool `process` niet is toegestaan, worden `exec`-uitvoeringen synchroon uitgevoerd en worden `yieldMs`/`background` genegeerd.
- Gestarte exec-opdrachten ontvangen `OPENCLAW_SHELL=exec` voor contextbewuste shell-/profielregels.
- Voor langlopend werk dat nu begint: start het eenmaal en vertrouw op automatisch ontwaken bij voltooiing (indien ingeschakeld) zodra de opdracht uitvoer produceert of mislukt.
- Als automatisch ontwaken bij voltooiing niet beschikbaar is, of als je een stille succesvolle uitvoering wilt bevestigen voor een opdracht die zonder uitvoer correct eindigt, vraag je de status op met `process`.
- Boots herinneringen of uitgestelde vervolgacties niet na met `sleep`-lussen of herhaald opvragen — gebruik Cron voor toekomstig werk.

### Omgevingsoverschrijvingen

| Variabele                                | Effect                                                                                                           |
| ---------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| `OPENCLAW_BASH_YIELD_MS`                 | Standaardwachttijd voordat de opdracht naar de achtergrond gaat (ms). Standaard 10000, begrensd op 10-120000.    |
| `OPENCLAW_BASH_MAX_OUTPUT_CHARS`         | Limiet voor uitvoer in het geheugen (tekens).                                                                    |
| `OPENCLAW_BASH_PENDING_MAX_OUTPUT_CHARS` | Limiet voor wachtende stdout/stderr per stream (tekens).                                                         |
| `OPENCLAW_BASH_JOB_TTL_MS`               | TTL voor voltooide sessies (ms), begrensd op 1m-3h.                                                              |
| `OPENCLAW_PROCESS_INPUT_WAIT_IDLE_MS`    | Drempel voor inactieve uitvoer waarna beschrijfbare achtergrondsessies worden gemarkeerd als waarschijnlijk wachtend op invoer. Standaard 15000. |

### Configuratie (te verkiezen boven omgevingsoverschrijvingen)

| Sleutel                               | Standaard | Effect                                                                         |
| ------------------------------------- | --------- | ------------------------------------------------------------------------------ |
| `tools.exec.backgroundMs`             | 10000     | Hetzelfde als `OPENCLAW_BASH_YIELD_MS`.                                              |
| `tools.exec.timeoutSeconds`           | 1800      | Standaardtime-out per aanroep.                                                 |
| `tools.exec.cleanupMs`                | 1800000   | Hetzelfde als `OPENCLAW_BASH_JOB_TTL_MS`.                                              |
| `tools.exec.notifyOnExit`             | true      | Plaats een systeemgebeurtenis in de wachtrij en vraag een Heartbeat aan wanneer een exec op de achtergrond eindigt. |
| `tools.exec.notifyOnExitEmptySuccess` | false     | Plaats ook voltooiingsgebeurtenissen in de wachtrij voor succesvolle achtergronduitvoeringen zonder uitvoer. |

## Overbrugging van onderliggende processen

Wanneer je langlopende onderliggende processen buiten de exec-/proces-tools start (CLI-herstarts, Gateway-helpers), koppel je de helper voor de onderliggende-procesbrug zodat beëindigingssignalen worden doorgestuurd en listeners bij afsluiten/fouten worden losgekoppeld. Dit voorkomt verweesde processen onder systemd en houdt het afsluiten consistent tussen platforms.

## Tool process

Acties:

| Actie       | Effect                                                                        |
| ----------- | ----------------------------------------------------------------------------- |
| `list`      | Actieve + voltooide sessies.                                                  |
| `poll`      | Lees nieuwe uitvoer van een sessie uit (rapporteert ook de afsluitstatus).     |
| `log`       | Lees samengevoegde uitvoer en aanwijzingen voor invoerherstel. Ondersteunt `offset` + `limit`. |
| `write`     | Stuur stdin (`data`, optioneel `eof`).                                        |
| `send-keys` | Stuur expliciete toetstokens of bytes naar een sessie met PTY.                 |
| `submit`    | Stuur Enter/regeleinde naar een sessie met PTY.                                |
| `paste`     | Stuur letterlijke tekst, eventueel verpakt in de modus voor bracketed paste.  |
| `kill`      | Beëindig een achtergrondsessie.                                                |
| `clear`     | Verwijder een voltooide sessie uit het geheugen.                               |
| `remove`    | Beëindig indien actief, wis anders indien voltooid.                            |

Opmerkingen:

- Alleen sessies op de achtergrond worden weergegeven/bewaard — uitsluitend in het geheugen, niet op schijf. Sessies gaan verloren wanneer het proces opnieuw wordt gestart.
- Een actieve achtergrondsessie blokkeert coöperatieve opschorting van de host en een veilige herstart van de Gateway totdat de proceseigenaar bevestigt dat het proces daadwerkelijk is afgesloten.
- `process remove` kan een actieve sessie direct verbergen nadat beëindiging is aangevraagd; opschorting en herstart blijven geblokkeerd totdat afsluiting is bevestigd.
- Sessielogboeken worden alleen in de chatgeschiedenis opgeslagen als je `process poll`/`log` uitvoert en het toolresultaat wordt vastgelegd.
- `process` is per agent afgebakend; het ziet alleen sessies die door die agent zijn gestart.
- Gebruik `poll`/`log` voor status, logboeken of bevestiging van voltooiing wanneer automatisch ontwaken bij voltooiing niet beschikbaar is.
- Gebruik `log` voordat je een interactieve CLI herstelt, zodat het huidige transcript, de stdin-status en de aanwijzing over wachten op invoer samen zichtbaar zijn.
- Gebruik `write`/`send-keys`/`submit`/`paste`/`kill` wanneer invoer of ingrijpen nodig is.
- `process list` bevat een afgeleide `name` (opdrachtwerkwoord + doel) om snel te kunnen scannen.
- `process list`, `poll` en `log` rapporteren `waitingForInput` alleen wanneer de sessie nog beschrijfbare stdin heeft en langer inactief is dan de drempel voor wachten op invoer (standaard 15000 ms, `OPENCLAW_PROCESS_INPUT_WAIT_IDLE_MS`).
- `process log` gebruikt regelgebaseerde `offset`/`limit`. Wanneer beide zijn weggelaten, worden de laatste 200 regels met een pagineringsaanwijzing geretourneerd. Wanneer `offset` is ingesteld en `limit` niet, wordt alles vanaf `offset` tot het einde geretourneerd (niet beperkt tot 200).
- `poll`'s `timeout` wacht maximaal het opgegeven aantal milliseconden voordat het terugkeert; waarden boven 30000 worden begrensd op 30000.
- Opvragen is bedoeld voor status op aanvraag, niet voor planning met wachtlussen. Als het werk later moet plaatsvinden, gebruik je Cron.

## Voorbeelden

Voer een lange taak uit en vraag de status later op:

```json
{ "tool": "exec", "command": "sleep 5 && echo done", "yieldMs": 1000 }
```

```json
{ "tool": "process", "action": "poll", "sessionId": "<id>" }
```

Inspecteer een interactieve sessie voordat je invoer verzendt:

```json
{ "tool": "process", "action": "log", "sessionId": "<id>" }
```

Start onmiddellijk op de achtergrond:

```json
{ "tool": "exec", "command": "npm run build", "background": true }
```

Stuur stdin:

```json
{ "tool": "process", "action": "write", "sessionId": "<id>", "data": "y\n" }
```

Stuur PTY-toetsen:

```json
{ "tool": "process", "action": "send-keys", "sessionId": "<id>", "keys": ["C-c"] }
```

Verzend de huidige regel:

```json
{ "tool": "process", "action": "submit", "sessionId": "<id>" }
```

Plak letterlijke tekst:

```json
{ "tool": "process", "action": "paste", "sessionId": "<id>", "text": "line1\nline2\n" }
```

## Gerelateerd

- [Tool exec](/nl/tools/exec)
- [Exec-goedkeuringen](/nl/tools/exec-approvals)
