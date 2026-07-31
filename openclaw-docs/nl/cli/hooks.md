---
read_when:
    - Je wilt agenthooks beheren
    - Je wilt de beschikbaarheid van hooks controleren of workspace-hooks inschakelen
summary: CLI-referentie voor `openclaw hooks` (agenthooks)
title: Hooks
x-i18n:
    generated_at: "2026-07-27T05:46:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d4d58ea2270cf5122018f7be2943401229929f48f448b15fdd126d1cc99e1e56
    source_path: cli/hooks.md
    workflow: 16
---

# `openclaw hooks`

Beheer agenthooks (gebeurtenisgestuurde automatiseringen voor opdrachten zoals `/new`, `/reset` en het starten van de Gateway). Alleen `openclaw hooks` is gelijkwaardig aan `openclaw hooks list`.

Gerelateerd: [Hooks](/nl/automation/hooks) - [Pluginhooks](/nl/plugins/hooks)

## Hooks weergeven

```bash
openclaw hooks list [--eligible] [--json] [-v|--verbose]
```

Geeft hooks weer die zijn gevonden in werkruimte-, beheerde, extra en gebundelde mappen.

- `--eligible`: alleen hooks waarvan aan de vereisten is voldaan.
- `--json`: gestructureerde uitvoer.
- `-v, --verbose`: neem een kolom Missing op met vereisten waaraan niet is voldaan.

```
Hooks (4/5 gereed)

Gereed:
  🚀 boot-md ✓ - Voer BOOT.md uit wanneer de Gateway wordt gestart
  📎 bootstrap-extra-files ✓ - Voeg tijdens het initialiseren van de agent aanvullende bootstrapbestanden uit de werkruimte in
  📝 command-logger ✓ - Registreer alle opdrachtgebeurtenissen in een centraal auditbestand
  💾 session-memory ✓ - Sla de sessiecontext op in het geheugen wanneer de opdracht /new of /reset wordt gegeven
```

## Hookinformatie ophalen

```bash
openclaw hooks info <name> [--json]
```

`<name>` is de hooknaam of hooksleutel (bijvoorbeeld `session-memory`). Toont de bron, bestands-/handlerpaden, homepage, gebeurtenissen en de status per vereiste (binaire bestanden, omgeving, configuratie, besturingssysteem).

## Geschiktheid controleren

```bash
openclaw hooks check [--json]
```

Toont een samenvatting met het aantal gereed/niet gereed; als hooks niet gereed zijn, wordt elke hook met de blokkerende reden weergegeven.

## Een hook inschakelen

```bash
openclaw hooks enable <name>
```

Voegt `hooks.internal.entries.<name>.enabled = true` toe aan de configuratie of werkt deze bij en zet ook de hoofdschakelaar `hooks.internal.enabled` aan (de Gateway laadt geen enkele interne hookhandler totdat er ten minste één is geconfigureerd). Mislukt als de hook niet bestaat, door een Plugin wordt beheerd of niet geschikt is (ontbrekende vereisten).

Door Plugins beheerde hooks tonen `plugin:<id>` in `hooks list` en kunnen hier niet worden in- of uitgeschakeld; schakel in plaats daarvan de verantwoordelijke Plugin in of uit.

Start de Gateway opnieuw na het inschakelen (herstart de macOS-menubalkapp of herstart je Gateway-proces tijdens de ontwikkeling), zodat de hooks opnieuw worden geladen.

## Een hook uitschakelen

```bash
openclaw hooks disable <name>
```

Stelt `hooks.internal.entries.<name>.enabled = false` in. Start de Gateway daarna opnieuw.

## Hookpakketten installeren en bijwerken

```bash
openclaw plugins install <package>        # standaard npm
openclaw plugins install npm:<package>    # alleen npm
openclaw plugins install <package> --pin  # opgeloste versie vastzetten
openclaw plugins install <path>           # lokale map of lokaal archief
openclaw plugins install -l <path>        # een lokale map koppelen in plaats van kopiëren

openclaw plugins update <id>
openclaw plugins update --all
openclaw plugins update --dry-run
```

Hookpakketten worden geïnstalleerd en bijgewerkt via het uniforme installatie- en updateprogramma voor Plugins; `openclaw hooks install` / `openclaw hooks update` werken nog steeds als verouderde aliassen die een waarschuwing tonen en doorsturen naar de opdrachten van `plugins`.

- Npm-specificaties zijn uitsluitend voor het register: een pakketnaam plus een optionele exacte versie of dist-tag. Git-/URL-/bestandsspecificaties en semver-bereiken worden geweigerd. Afhankelijkheden worden projectlokaal geïnstalleerd met `--ignore-scripts`.
- Kale specificaties en `@latest` blijven op het stabiele kanaal; als npm een prerelease oplevert, stopt OpenClaw en wordt je gevraagd je expliciet aan te melden (`@beta`, `@rc` of een exacte prereleaseversie).
- Ondersteunde archieven: `.zip`, `.tgz`, `.tar.gz`, `.tar`.
- `-l, --link` koppelt een lokale map in plaats van deze te kopiëren (voegt deze toe aan `hooks.internal.load.extraDirs`); gekoppelde hookpakketten zijn beheerde hooks uit een door de beheerder geconfigureerde map, geen werkruimtehooks.
- `--pin` registreert npm-installaties als een exact opgeloste `name@version` in de gedeelde SQLite-status.
- De installatie kopieert het pakket naar `~/.openclaw/hooks/<id>`, schakelt de hooks ervan in onder `hooks.internal.entries.*` en registreert de herkomst van de installatie in de gedeelde SQLite-status.
- Als een opgeslagen integriteitshash niet meer overeenkomt met het opgehaalde artefact, waarschuwt OpenClaw en vraagt het om bevestiging voordat het doorgaat; geef de globale optie `--yes` door om deze vraag over te slaan (bijvoorbeeld in CI).

## Gebundelde hooks

| Hook                  | Gebeurtenissen                                    | Functie                                                                                             |
| --------------------- | ------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| boot-md               | `gateway:startup`                                 | Voert `BOOT.md` uit wanneer de Gateway wordt gestart voor elk geconfigureerd agentbereik                                  |
| bootstrap-extra-files | `agent:bootstrap`                                 | Voegt tijdens het initialiseren van de agent extra bootstrapbestanden in (bijvoorbeeld `AGENTS.md`/`TOOLS.md` van een monorepo) |
| command-logger        | `command`                                         | Registreert opdrachtgebeurtenissen in `~/.openclaw/logs/commands.log`                                             |
| compaction-notifier   | `session:compact:before`, `session:compact:after` | Stuurt zichtbare chatmeldingen wanneer de Compaction van de sessie begint en eindigt                             |
| session-memory        | `command:new`, `command:reset`                    | Slaat de sessiecontext op in het geheugen bij `/new` of `/reset`                                              |

Schakel een gebundelde hook in met `openclaw hooks enable <hook-name>`. Volledige details, configuratiesleutels en standaardwaarden: [Gebundelde hooks](/nl/automation/hooks#bundled-hooks).

### Logbestand van command-logger

```bash
tail -n 20 ~/.openclaw/logs/commands.log        # recente opdrachten
cat ~/.openclaw/logs/commands.log | jq .          # netjes weergeven
grep '"action":"new"' ~/.openclaw/logs/commands.log | jq .   # filteren op actie
```

## Opmerkingen

- `hooks list --json`, `info --json` en `check --json` schrijven gestructureerde JSON rechtstreeks naar stdout.

## Gerelateerd

- [CLI-referentie](/nl/cli)
- [Automatiseringshooks](/nl/automation/hooks)
