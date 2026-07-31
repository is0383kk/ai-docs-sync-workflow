---
read_when:
    - Je wilt met OpenClaw chatten voor configuratie of herstel
    - Je voert de eerste configuratie uit met de onboardingwizard
    - Je wilt het standaardpad voor de werkruimte instellen
    - Je hebt de configuratievlag voor alleen de basislijn nodig voor scripts
summary: CLI-referentie voor `openclaw setup` (chat met systeemagent met onboarding als terugvaloptie)
title: Installatie
x-i18n:
    generated_at: "2026-07-27T05:06:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b3b4f70f2631683fcb03007a80fe43a06387be3d7e4d533381e5e536333af051
    source_path: cli/setup.md
    workflow: 16
---

# `openclaw setup`

`openclaw setup` is het toegangspunt voor de systeemagent. Op een geconfigureerd systeem opent alleen
`openclaw setup` een interactieve OpenClaw-chat. Op een nieuw systeem
wordt in plaats daarvan de begeleide onboarding gestart. Gebruik `-m`/`--message` voor één verzoek of
`--baseline` om configuratie-/werkruimtemappen zonder de wizard te initialiseren.

Routeringsvolgorde:

1. Elke onboardingoptie (`--wizard`, `--baseline`, werkruimte, reset,
   niet-interactief, flow, modus, Gateway, daemon, overslaan, importeren, extern of authenticatieopties)
   voert de onboarding precies zo uit als `openclaw onboard`.
2. `-m`/`--message` of `--yes` voert de systeemagent uit.
3. Zonder routeringsoptie opent een geconfigureerd interactief systeem OpenClaw. Op een
   nieuw systeem wordt de onboarding uitgevoerd. Op een geconfigureerd systeem drukt `--json` het
   systeemoverzicht af, zelfs zonder TTY; bij een onboardingoptie blijft de
   JSON-samenvatting van de onboarding behouden.

In de begeleide modus is `--workspace <dir>` de werkruimte die aan OpenClaw wordt voorgesteld;
deze wordt pas opgeslagen nadat je het voorstel hebt goedgekeurd. Bij een nieuwe installatie
slaan de basis-, klassieke en niet-interactieve installatie de opgegeven werkruimte via hun normale flow op.
Wanneer een bestaande lijst met agents opnieuw zou worden toegewezen,
vereist de klassieke wizard expliciete bevestiging; bij een niet-interactieve installatie blijft de
huidige werkruimte van de fleet behouden en wordt een waarschuwing afgedrukt.

Begeleide inferentiedetectie wordt uitgevoerd op de Gateway-host met macOS of Linux. De CLI
en de macOS-app roepen dezelfde detector van de Gateway aan, die geconfigureerde
modellen, ondersteunde CLI-aanmeldingen, omgevingsvariabelen voor API-sleutels en reeds
geïnstalleerde modellen van Ollama of LM Studio controleert. Lokale modellen worden tijdens deze
automatische controle nooit gedownload. Gedetecteerde lokale runtimes worden na CLI- en API-sleutelkandidaten
automatisch getest; wanneer meerdere lokale modellen beschikbaar zijn, geeft OpenClaw de voorkeur aan
de krachtigste instructfamilie met ondersteuning voor toolaanroepen. De geselecteerde kandidaat moet een
echte voltooiing beantwoorden voordat de provider- en modelconfiguratie wordt opgeslagen.
Geïnstalleerde CLI's van Gemini, Antigravity, Pi en OpenCode worden ook gemeld wanneer
ze niet als herbruikbare inferentieroute voor de begeleide installatie kunnen dienen.

`setup` accepteert dezelfde onboardingvlaggen als `openclaw onboard`, waaronder
authenticatie (`--auth-choice`, `--token`, vlaggen voor providersleutels), Gateway
(`--gateway-port`, `--gateway-bind`, `--gateway-auth`, `--install-daemon`),
Tailscale (`--tailscale`), reset (`--reset`, `--reset-scope`), flow
(`--flow quickstart|advanced|manual|import`) en overslavlaggen
(`--skip-channels`, `--skip-skills`, `--skip-bootstrap`, `--skip-search`,
`--skip-health`, `--skip-ui`, `--skip-hooks`). Geef `--tui` door om dezelfde
terminaluitweg te gebruiken als `openclaw onboard --tui`. Zie [Onboarding](/nl/cli/onboard) en
[CLI-automatisering](/nl/start/wizard-cli-automation) voor het volledige vlaggenoverzicht en
niet-interactieve voorbeelden. `openclaw onboard --modern` blijft een compatibiliteitstoegangspunt
voor dezelfde door inferentie afgeschermde OpenClaw-assistent.

<Note>
`openclaw setup` is bedoeld voor installaties met een wijzigbare configuratie. In de Nix-modus (`OPENCLAW_NIX_MODE=1`) weigert OpenClaw installatiewijzigingen te schrijven, omdat het configuratiebestand door Nix wordt beheerd. Gebruik de officiële [nix-openclaw-snelstart](https://github.com/openclaw/nix-openclaw#quick-start) of de equivalente bronconfiguratie voor een ander Nix-pakket.
</Note>

## Opties

| Vlag                       | Beschrijving                                                                                          |
| -------------------------- | ---------------------------------------------------------------------------------------------------- |
| `-m, --message <text>`     | Voer één OpenClaw-verzoek uit.                                                                            |
| `--yes`                    | Keur permanente configuratiewijzigingen voor één `--message`-verzoek goed.                                        |
| `--workspace <dir>`        | Werkruimtevoorstel; bestaande fleets vereisen klassieke bevestiging en blijven niet-interactief behouden. |
| `--baseline`               | Maak basisconfiguratie-, werkruimte- en sessiemappen zonder onboarding.                                 |
| `--wizard`                 | Dwing interactieve onboarding af.                                                                        |
| `--tui`                    | Gebruik de terminaluitweg in plaats van de overdracht naar de browser.                                               |
| `--non-interactive`        | Voer de onboarding zonder prompts uit.                                                                      |
| `--accept-risk`            | Erken het risico van systeemagenttoegang tot het volledige systeem; vereist met `--non-interactive`.                        |
| `--mode <mode>`            | Onboardingmodus: `local` of `remote`.                                                                |
| `--flow <flow>`            | Onboardingflow: `quickstart`, `advanced`, `manual` of `import`.                                       |
| `--reset`                  | Reset configuratie + referenties + sessies vóór de onboarding (werkruimte alleen met `--reset-scope full`).  |
| `--reset-scope <scope>`    | Resetbereik: `config`, `config+creds+sessions` of `full`.                                           |
| `--import-from <provider>` | Migratieprovider die tijdens de onboarding moet worden uitgevoerd.                                                         |
| `--import-source <path>`   | Bronmap van de agent voor `--import-from`.                                                               |
| `--import-secrets`         | Importeer ondersteunde geheimen tijdens de onboardingmigratie.                                                |
| `--remote-url <url>`       | WebSocket-URL van de externe Gateway.                                                                        |
| `--remote-token <token>`   | Token van de externe Gateway (optioneel).                                                                     |
| `--json`                   | Geconfigureerd systeem: OpenClaw-overzicht. Onboardingroute: onboardingsamenvatting.                          |

`--classic` en `--non-interactive` sluiten elkaar uit: klassiek opent de
wizard met prompts, terwijl de niet-interactieve installatie het automatiseringspad gebruikt.
Bij interactieve onboarding vullen `--remote-url` en `--remote-token` de
stap voor de externe Gateway vooraf in en hebben ze voor die uitvoering voorrang op opgeslagen externe waarden.
Wanneer je de URL wijzigt, worden opgeslagen referenties niet hergebruikt, tenzij je ook een token doorgeeft.
Het token blijft gemaskeerd en gebruikt de in de wizard geselecteerde opslagmodus voor platte tekst of SecretRef.

### Basismodus

`openclaw setup --baseline` behoudt het oudere gedrag met alleen de basisconfiguratie: hiermee worden
de configuratie-, werkruimte- en sessiemappen gemaakt, waarna het programma afsluit zonder
de onboarding uit te voeren. Deze modus accepteert `--workspace` en onschadelijke uitvoeropties, maar
weigert expliciete onboarding-, Gateway-, authenticatie-, reset- of daemonopties in plaats van
ze stilzwijgend te negeren. Als een bestaande configuratie ongeldig is, behoudt de basisinstallatie
deze en wordt gevraagd `openclaw doctor` uit te voeren voordat je het opnieuw probeert.

## Voorbeelden

```bash
openclaw setup
openclaw setup -m "status"
openclaw setup -m "restart gateway" --yes
openclaw setup --json
openclaw setup --wizard
openclaw setup --baseline
openclaw setup --workspace ~/.openclaw/workspace
openclaw setup --import-from hermes --import-source ~/.hermes
openclaw setup --non-interactive --accept-risk --mode remote --remote-url wss://gateway-host:18789 --remote-token <token>
```

## Opmerkingen

- Voer na de basisinstallatie `openclaw onboard` uit voor het volledige begeleide traject, `openclaw configure` voor gerichte wijzigingen of `openclaw channels add` om kanaalaccounts toe te voegen.
- Als een Hermes-status wordt gedetecteerd, kan de interactieve onboarding automatisch migratie aanbieden. Importonboarding vereist een nieuwe installatie; gebruik [Migreren](/nl/cli/migrate) voor proefplannen, back-ups en de overschrijfmodus buiten de onboarding.

## Gerelateerd

- [CLI-naslaginformatie](/nl/cli)
- [Onboarding](/nl/cli/onboard)
- [Onboarding (CLI)](/nl/start/wizard)
- [Aan de slag](/nl/start/getting-started)
- [Installatieoverzicht](/nl/install)
