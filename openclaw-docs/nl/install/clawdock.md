---
read_when:
    - Je gebruikt OpenClaw vaak met Docker en wilt kortere commando's voor dagelijks gebruik
    - Je wilt een hulplaag voor het dashboard, logboeken, tokenconfiguratie en koppelingsflows
summary: ClawDock-shellhulpprogramma's voor Docker-gebaseerde OpenClaw-installaties
title: ClawDock
x-i18n:
    generated_at: "2026-07-27T05:36:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5bb829a3301178503f910931e86a39f7befeaf186044f4088a25dc80ea99130d
    source_path: install/clawdock.md
    workflow: 16
---

ClawDock is een kleine laag met shellhulpfuncties voor Docker-gebaseerde OpenClaw-installaties.

Hiermee krijg je korte opdrachten zoals `clawdock-start`, `clawdock-dashboard` en `clawdock-fix-token` in plaats van langere `docker compose ...`-aanroepen.

Als je Docker nog niet hebt ingesteld, begin dan met [Docker](/nl/install/docker).

## Installeren

```bash
mkdir -p ~/.clawdock && curl -sL https://raw.githubusercontent.com/openclaw/openclaw/main/scripts/clawdock/clawdock-helpers.sh -o ~/.clawdock/clawdock-helpers.sh
echo 'source ~/.clawdock/clawdock-helpers.sh' >> ~/.zshrc && source ~/.zshrc
```

Als je ClawDock eerder vanuit `scripts/shell-helpers/clawdock-helpers.sh` hebt geïnstalleerd, installeer het dan opnieuw vanuit het huidige `scripts/clawdock/clawdock-helpers.sh`-pad; het oude onbewerkte GitHub-pad is verwijderd.

De hulpfuncties detecteren bij het eerste gebruik automatisch je OpenClaw-check-out (door veelgebruikte paden zoals `~/openclaw` en `~/projects/openclaw` te controleren) en slaan het resultaat op in de cache in `~/.clawdock/config`. Stel `CLAWDOCK_DIR` zelf in als je check-out zich ergens anders bevindt.

## Wat je krijgt

### Basisbewerkingen

| Opdracht            | Beschrijving            |
| ------------------ | ---------------------- |
| `clawdock-start`   | De Gateway starten      |
| `clawdock-stop`    | De Gateway stoppen       |
| `clawdock-restart` | De Gateway opnieuw starten    |
| `clawdock-status`  | Containerstatus controleren |
| `clawdock-logs`    | Gateway-logboeken volgen    |

### Toegang tot de container

| Opdracht                   | Beschrijving                                   |
| ------------------------- | --------------------------------------------- |
| `clawdock-shell`          | Een shell in de Gateway-container openen     |
| `clawdock-cli <command>`  | OpenClaw CLI-opdrachten in Docker uitvoeren           |
| `clawdock-exec <command>` | Een willekeurige opdracht in de container uitvoeren |

### Webinterface en koppeling

| Opdracht                 | Beschrijving                  |
| ----------------------- | ---------------------------- |
| `clawdock-dashboard`    | De URL van de bedieningsinterface openen      |
| `clawdock-devices`      | Openstaande apparaatkoppelingen weergeven |
| `clawdock-approve <id>` | Een koppelingsverzoek goedkeuren    |

### Configuratie en onderhoud

| Opdracht              | Beschrijving                                       |
| -------------------- | ------------------------------------------------- |
| `clawdock-fix-token` | Het Gateway-token naar de containerconfiguratie schrijven |
| `clawdock-update`    | Ophalen, opnieuw bouwen en opnieuw starten                        |
| `clawdock-rebuild`   | Alleen de Docker-installatiekopie opnieuw bouwen                     |
| `clawdock-clean`     | Containers en volumes verwijderen                     |

### Hulpprogramma's

| Opdracht                | Beschrijving                             |
| ---------------------- | --------------------------------------- |
| `clawdock-health`      | Een statuscontrole van de Gateway uitvoeren              |
| `clawdock-token`       | Het Gateway-token weergeven                 |
| `clawdock-cd`          | Naar de OpenClaw-projectmap gaan  |
| `clawdock-config`      | `~/.openclaw` openen                      |
| `clawdock-show-config` | Configuratiebestanden met geredigeerde waarden weergeven |
| `clawdock-workspace`   | De werkruimtemap openen            |
| `clawdock-help`        | Alle ClawDock-opdrachten weergeven              |

## Stappen voor het eerste gebruik

```bash
clawdock-start
clawdock-fix-token
clawdock-dashboard
```

Als de browser meldt dat koppeling vereist is:

```bash
clawdock-devices
clawdock-approve <request-id>
```

## Configuratie en geheimen

ClawDock leest twee afzonderlijke `.env`-bestanden, overeenkomstig de in [Docker](/nl/install/docker) beschreven splitsing:

- Het projectbestand `.env` naast `docker-compose.yml`: Docker-specifieke waarden zoals de naam van de installatiekopie, poorten en `OPENCLAW_GATEWAY_TOKEN`. `clawdock-token` leest het token hieruit.
- `~/.openclaw/.env` (gekoppeld aan de container): door omgevingsvariabelen ondersteunde geheimen die OpenClaw zelf beheert, naast `openclaw.json` en `agents/<agentId>/agent/auth-profiles.json`.

`clawdock-fix-token` kopieert het token vanuit het `.env`-projectbestand naar de configuratiewaarden `gateway.remote.token` en `gateway.auth.token` van de container en start de Gateway opnieuw.

Gebruik `clawdock-show-config` om snel `openclaw.json` en beide `.env`-bestanden te inspecteren; in de weergegeven uitvoer worden `.env`-waarden geredigeerd.

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Docker" href="/nl/install/docker" icon="docker">
    Canonieke Docker-installatie voor OpenClaw.
  </Card>
  <Card title="Docker-VM-runtime" href="/nl/install/docker-vm-runtime" icon="cube">
    Door Docker beheerde VM-runtime voor versterkte isolatie.
  </Card>
  <Card title="Bijwerken" href="/nl/install/updating" icon="arrow-up-right-from-square">
    Het OpenClaw-pakket en beheerde services bijwerken.
  </Card>
</CardGroup>
