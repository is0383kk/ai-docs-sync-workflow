---
read_when:
    - Je wilt shellaanvullingen voor zsh/bash/fish/PowerShell
    - Je moet scripts voor automatische aanvulling cachen in de OpenClaw-statusopslag
summary: CLI-referentie voor `openclaw completion` (shell-aanvullingsscripts genereren/installeren)
title: Voltooiing
x-i18n:
    generated_at: "2026-07-27T05:27:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 67cb52a47036745150887c752d18e2dfa84fab2722c27c696142d23080bb2efd
    source_path: cli/completion.md
    workflow: 16
---

# `openclaw completion`

Genereer shell-aanvullingsscripts, sla ze op in de cache onder de OpenClaw-status en installeer ze desgewenst in je shellprofiel.

## Gebruik

```bash
openclaw completion                          # zsh-script naar stdout schrijven
openclaw completion --shell fish             # fish-script schrijven
openclaw completion --write-state            # scripts voor alle shells in de cache opslaan
openclaw completion --write-state --install  # in één stap in de cache opslaan en installeren
openclaw completion --shell bash --write-state
```

## Opties

- `-s, --shell <shell>`: doelshell (`zsh`, `bash`, `powershell`, `fish`; standaard: `zsh`)
- `-i, --install`: shell-aanvulling installeren door een source-regel voor het gecachete script aan je shellprofiel toe te voegen
- `--write-state`: aanvullingsscript(s) naar `$OPENCLAW_STATE_DIR/completions` schrijven (standaard `~/.openclaw/completions`) zonder ze naar stdout te schrijven; met `--shell` wordt alleen die shell geschreven, anders alle vier
- `-y, --yes`: bevestigingsvragen voor installatie overslaan (niet-interactief)

## Installatieproces

`--install` laat je profiel naar het gecachete script verwijzen, dus de cache moet eerst bestaan: als deze ontbreekt, mislukt de opdracht en wordt aangegeven dat je `openclaw completion --write-state` moet uitvoeren. Combineer dit met `--write-state --install` om beide in één stap uit te voeren. Zonder `--shell` detecteert `--install` de shell via `$SHELL` (met zsh als terugvaloptie).

De installatie schrijft een klein `# OpenClaw Completion`-blok naar je shellprofiel en vervangt eventuele oudere, trage `source <(openclaw completion ...)`-regels door de gecachete source-regel:

| Shell      | Profiel                                                                                                                                                                                    |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| bash       | `~/.bashrc` (valt terug op `~/.bash_profile` wanneer `~/.bashrc` ontbreekt)                                                                                                                  |
| fish       | `~/.config/fish/config.fish`                                                                                                                                                               |
| powershell | `~/.config/powershell/Microsoft.PowerShell_profile.ps1` (op Windows: `Documents/PowerShell/Microsoft.PowerShell_profile.ps1`, of `Documents/WindowsPowerShell/...` voor Windows PowerShell) |
| zsh        | `~/.zshrc`                                                                                                                                                                                 |

## Opmerkingen

- Zonder `--install` of `--write-state` schrijft de opdracht het script naar stdout.
- Bij het genereren van aanvullingen wordt de volledige opdrachtstructuur direct geladen, inclusief CLI-opdrachten van plugins, zodat geneste subopdrachten worden opgenomen.
- `openclaw update` vernieuwt de aanvullingscache automatisch na een geslaagde update; `openclaw doctor` kan ontbrekende of verouderde aanvullingsconfiguraties herstellen.

## Gerelateerd

- [CLI-referentie](/nl/cli)
