---
read_when:
    - Je wilt de Gateway-service en/of lokale status verwijderen
    - Je wilt eerst een proefrun uitvoeren
summary: CLI-referentie voor `openclaw uninstall` (Gateway-service + lokale gegevens verwijderen)
title: Deïnstalleren
x-i18n:
    generated_at: "2026-07-27T05:06:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1e2e3996cf6d5c0fd11e5054c8fe60f7f8d25047193bb13944ca170bf77b581a
    source_path: cli/uninstall.md
    workflow: 16
---

# `openclaw uninstall`

Verwijder de Gateway-service en/of lokale gegevens. De CLI zelf wordt niet
verwijderd; verwijder die afzonderlijk via npm/pnpm.

## Opties

| Vlag                | Standaard | Beschrijving                                          |
| ------------------- | ------- | ---------------------------------------------------- |
| `--service`         | `false` | Verwijder de Gateway-service.                          |
| `--state`           | `false` | Verwijder status en configuratie.                             |
| `--workspace`       | `false` | Verwijder werkruimtemappen.                        |
| `--app`             | `false` | Verwijder de macOS-app.                                |
| `--all`             | `false` | Afkorting voor `--service --state --workspace --app`. |
| `--yes`             | `false` | Sla bevestigingsvragen over.                           |
| `--non-interactive` | `false` | Schakel vragen uit; vereist `--yes`.                   |
| `--dry-run`         | `false` | Toon geplande acties zonder bestanden te verwijderen.        |

Zonder bereikvlaggen wordt via een interactieve meervoudige selectie gevraagd welke onderdelen
moeten worden verwijderd (standaard zijn service, status en werkruimte vooraf geselecteerd).

## Voorbeelden

```bash
openclaw backup create
openclaw uninstall
openclaw uninstall --service --yes --non-interactive
openclaw uninstall --state --workspace --yes --non-interactive
openclaw uninstall --all --yes
openclaw uninstall --dry-run
```

## Opmerkingen

- Voer eerst `openclaw backup create` uit voor een herstelbare momentopname voordat je
  status of werkruimten verwijdert.
- `--state` behoudt geconfigureerde werkruimtemappen, tenzij `--workspace`
  ook is geselecteerd.

## Gerelateerd

- [CLI-referentie](/nl/cli)
- [Verwijderen](/nl/install/uninstall)
