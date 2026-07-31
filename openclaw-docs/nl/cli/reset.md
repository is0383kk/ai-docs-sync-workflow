---
read_when:
    - Je wilt de lokale status wissen, maar de CLI geïnstalleerd houden
    - Je wilt een proefuitvoering van wat er zou worden verwijderd
summary: CLI-referentie voor `openclaw reset` (lokale status/configuratie resetten)
title: Opnieuw instellen
x-i18n:
    generated_at: "2026-07-27T05:41:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 54f1d320ee368dae4a4bfb32dea73d19eb35f9f30edd12d9c2580ab7e6a26fa6
    source_path: cli/reset.md
    workflow: 16
---

# `openclaw reset`

Lokale configuratie/status opnieuw instellen (de CLI blijft geïnstalleerd).

```bash
openclaw reset
openclaw reset --dry-run
openclaw reset --scope config --yes --non-interactive
openclaw reset --scope config+creds+sessions --yes --non-interactive
openclaw reset --scope full --yes --non-interactive
```

## Opties

- `--scope <scope>`: `config`, `config+creds+sessions` of `full`
- `--yes`: bevestigingsvragen overslaan
- `--non-interactive`: vragen uitschakelen; vereist `--scope` en `--yes`
- `--dry-run`: acties weergeven zonder bestanden te verwijderen

## Bereiken

| Bereik                  | Verwijdert                                                                  | Stopt eerst de Gateway |
| ----------------------- | --------------------------------------------------------------------------- | ---------------------- |
| `config`                | alleen het configuratiebestand                                              | nee                    |
| `config+creds+sessions` | configuratiebestand, map met OAuth-/aanmeldgegevens en sessiemappen per agent | ja                     |
| `full`                  | statusmap (inclusief de gedeelde SQLite-database) plus werkruimtemappen      | ja                     |

`config+creds+sessions` en `full` stoppen een actieve beheerde Gateway-service voordat de status wordt verwijderd.

## Opmerkingen

- Voer eerst `openclaw backup create` uit voor een herstelbare momentopname voordat je de lokale status verwijdert.
- De status en attestaties van de werkruimte-instellingen zijn rijen in de gedeelde SQLite-database. Daarom verwijdert `full` deze samen met de statusmap; er zijn momenteel geen afzonderlijke attestatiebestanden die apart moeten worden verwijderd.
- Zonder `--scope` vraagt `openclaw reset` interactief welk bereik moet worden verwijderd.
- `--non-interactive` is alleen geldig wanneer zowel `--scope` als `--yes` zijn ingesteld.
- `config+creds+sessions` en `full` geven na voltooiing `Next: openclaw onboard --install-daemon` weer.

## Gerelateerd

- [CLI-referentie](/nl/cli)
