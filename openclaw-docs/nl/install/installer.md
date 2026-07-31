---
read_when:
    - Je wilt `openclaw.ai/install.sh` begrijpen
    - Je wilt installaties automatiseren (CI / headless)
    - Je wilt installeren vanuit een GitHub-checkout
summary: Hoe de installatiescripts werken (install.sh, install-cli.sh, install.ps1), vlaggen en automatisering
title: Interne werking van het installatieprogramma
x-i18n:
    generated_at: "2026-07-27T05:02:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7878f10903893b4e1902bbc79991f43edaa436bd802d5fecde41421e3e05bc2b
    source_path: install/installer.md
    workflow: 16
---

OpenClaw wordt geleverd met drie installatiescripts, aangeboden via `openclaw.ai`.

| Script                             | Platform             | Wat het doet                                                                                   |
| ---------------------------------- | -------------------- | ---------------------------------------------------------------------------------------------- |
| [`install.sh`](#installsh)         | macOS / Linux / WSL  | Installeert indien nodig Node, installeert OpenClaw via npm (standaard) of git en kan de onboarding uitvoeren.       |
| [`install-cli.sh`](#install-clish) | macOS / Linux / WSL  | Installeert Node + OpenClaw via npm of git in een lokaal voorvoegsel (`~/.openclaw`). Geen root vereist. |
| [`install.ps1`](#installps1)       | Windows (PowerShell) | Installeert indien nodig Node, installeert OpenClaw via npm (standaard) of git en kan de onboarding uitvoeren.       |

Alle drie ondersteunen Node **22.22.3+, 24.15+ of 25.9+**; Node 24 is het standaarddoel voor nieuwe installaties.

## Snelle commando's

<Tabs>
  <Tab title="install.sh">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash
    ```

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --help
    ```

  </Tab>
  <Tab title="install-cli.sh">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install-cli.sh | bash
    ```

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install-cli.sh | bash -s -- --help
    ```

  </Tab>
  <Tab title="install.ps1">
    ```powershell
    iwr -useb https://openclaw.ai/install.ps1 | iex
    ```

    ```powershell
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -Tag beta -NoOnboard -DryRun
    ```

  </Tab>
</Tabs>

<Note>
Als de installatie slaagt maar `openclaw` niet wordt gevonden in een nieuwe terminal, raadpleeg dan [Problemen met Node.js oplossen](/nl/install/node#troubleshooting).
</Note>

---

<a id="installsh"></a>

## install.sh

<Tip>
Aanbevolen voor de meeste interactieve installaties op macOS/Linux/WSL.
</Tip>

### Verloop (install.sh)

<Steps>
  <Step title="Besturingssysteem detecteren">
    Ondersteunt macOS en Linux (inclusief WSL).
  </Step>
  <Step title="Standaard Node.js 24 beschikbaar maken">
    Controleert de Node-versie en installeert indien nodig Node 24 (Homebrew op macOS, NodeSource-installatiescripts op Linux met apt/dnf/yum). Op macOS wordt Homebrew alleen geïnstalleerd wanneer het installatieprogramma dit nodig heeft voor Node of Git. Node 22.22.3+, Node 24.15+ en Node 25.9+ worden ondersteund; Node 23 wordt niet ondersteund.
    Op Alpine/musl Linux gebruikt het installatieprogramma apk-pakketten in plaats van NodeSource en verifieert het de daadwerkelijk gekoppelde SQLite-versie. De huidige stabiele pakketstromen van Alpine kunnen een voldoende nieuwe Node leveren met een kwetsbare systeemversie van SQLite; gebruik in dat geval een officiële `node:24-alpine`-container of een op glibc gebaseerde host.
  </Step>
  <Step title="Git beschikbaar maken">
    Installeert Git als dit ontbreekt met de gedetecteerde pakketbeheerder, waaronder Homebrew op macOS en apk op Alpine.
  </Step>
  <Step title="OpenClaw installeren">
    - `npm`-methode (standaard): globale npm-installatie
    - `git`-methode: repository klonen/bijwerken, afhankelijkheden installeren met pnpm, bouwen en vervolgens de wrapper installeren op `~/.local/bin/openclaw`

  </Step>
  <Step title="Taken na de installatie">
    - Zoekt het zojuist geïnstalleerde binaire bestand `openclaw` op voor vervolgcommando's
    - Start voor een niet-geconfigureerde installatie de onboarding vóór doctor- of Gateway-controles. Met `--no-onboard` of zonder TTY wordt het commando weergegeven waarmee de configuratie later kan worden voltooid.
    - Vernieuwt en herstart voor een geconfigureerde installatie zo goed mogelijk een geladen Gateway-service en voert doctor uit. Bij upgrades worden waar mogelijk plugins bijgewerkt, of wordt het handmatige commando weergegeven bij een headless uitvoering waarbij prompts zijn ingeschakeld.
    - Wanneer `--verify` wordt uitgevoerd, controleert dit de geïnstalleerde versie en alleen de status van de Gateway nadat er een configuratie bestaat.

  </Step>
</Steps>

### Detectie van broncode-checkout

Bij uitvoering in een OpenClaw-checkout (`package.json` + `pnpm-workspace.yaml`) biedt het script de volgende opties:

- checkout gebruiken (`git`), of
- globale installatie gebruiken (`npm`)

Als er geen TTY beschikbaar is en geen installatiemethode is ingesteld, wordt standaard `npm` gebruikt en verschijnt er een waarschuwing.

Het script wordt afgesloten met code `2` bij een ongeldige methodeselectie of ongeldige waarden voor `--install-method`.

### Voorbeelden (install.sh)

<Tabs>
  <Tab title="Standaard">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash
    ```
  </Tab>
  <Tab title="Onboarding overslaan">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --no-onboard
    ```
  </Tab>
  <Tab title="Git-installatie">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```
  </Tab>
  <Tab title="Checkout van GitHub main">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git --version main
    ```
  </Tab>
  <Tab title="Proefuitvoering">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --dry-run
    ```
  </Tab>
  <Tab title="Verifiëren na installatie">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --no-onboard --verify
    ```
  </Tab>
</Tabs>

<AccordionGroup>
  <Accordion title="Overzicht van vlaggen">

| Vlag                                    | Beschrijving                                                             |
| --------------------------------------- | ----------------------------------------------------------------------- |
| `--install-method \| --method npm\|git` | Installatiemethode kiezen (standaard: `npm`)                                  |
| `--npm`                                 | Snelkoppeling voor de npm-methode                                                 |
| `--git \| --github`                     | Snelkoppeling voor de git-methode                                                 |
| `--version <version\|dist-tag\|spec>`   | npm-versie, dist-tag of pakketspecificatie (standaard: `latest`)              |
| `--beta`                                | Indien beschikbaar de bèta-dist-tag gebruiken, anders terugvallen op `latest`              |
| `--git-dir \| --dir <path>`             | Checkoutmap (standaard: `~/openclaw`)                              |
| `--no-git-update`                       | `git pull` overslaan voor bestaande checkout                                   |
| `--no-prompt`                           | Prompts uitschakelen                                                         |
| `--no-onboard`                          | Onboarding overslaan                                                         |
| `--onboard`                             | Onboarding inschakelen                                                       |
| `--verify`                              | Een snelle verificatie na de installatie uitvoeren (`--version`, Gateway-status indien geladen) |
| `--dry-run`                             | Acties weergeven zonder wijzigingen toe te passen                                  |
| `--verbose`                             | Debuguitvoer inschakelen (`set -x`, npm-logboeken op meldingsniveau)                   |
| `--help \| -h`                          | Gebruiksinformatie weergeven                                                              |

  </Accordion>

  <Accordion title="Overzicht van omgevingsvariabelen">

| Variabele                                          | Beschrijving                                                        |
| ------------------------------------------------- | ------------------------------------------------------------------ |
| `OPENCLAW_INSTALL_METHOD=git\|npm`                | Installatiemethode                                                     |
| `OPENCLAW_VERSION=latest\|next\|<semver>\|<spec>` | npm-versie, dist-tag of pakketspecificatie                             |
| `OPENCLAW_BETA=0\|1`                              | Bèta gebruiken indien beschikbaar                                              |
| `OPENCLAW_HOME=<path>`                            | Basismap voor OpenClaw-status en standaardpaden voor git/onboarding |
| `OPENCLAW_GIT_DIR=<path>`                         | Checkoutmap                                                 |
| `OPENCLAW_GIT_UPDATE=0\|1`                        | Git-updates in- of uitschakelen                                                 |
| `OPENCLAW_NO_PROMPT=1`                            | Prompts uitschakelen                                                    |
| `OPENCLAW_VERIFY_INSTALL=1`                       | De snelle verificatie na installatie uitvoeren                                  |
| `OPENCLAW_NO_ONBOARD=1`                           | Onboarding overslaan                                                    |
| `OPENCLAW_DRY_RUN=1`                              | Proefuitvoeringsmodus                                                       |
| `OPENCLAW_VERBOSE=1`                              | Debugmodus                                                         |
| `OPENCLAW_NPM_LOGLEVEL=error\|warn\|notice`       | npm-logniveau (standaard: `error`, verbergt npm-meldingen over verouderde functies)      |

  </Accordion>
</AccordionGroup>

---

<a id="install-clish"></a>

## install-cli.sh

<Info>
Ontworpen voor omgevingen waarin alles onder een lokaal voorvoegsel
(standaard `~/.openclaw`) moet staan, zonder afhankelijkheid van een systeeminstallatie van Node. Ondersteunt standaard
npm-installaties en daarnaast git-checkoutinstallaties binnen hetzelfde voorvoegselverloop.
</Info>

### Verloop (install-cli.sh)

<Steps>
  <Step title="Lokale Node-runtime installeren">
    Downloadt een vastgezette, ondersteunde Node LTS-tarball (de versie is in het script opgenomen en wordt onafhankelijk bijgewerkt, standaard `24.15.0`) naar `<prefix>/tools/node-v<version>` en verifieert SHA-256.
    Linux ARMv7 gebruikt Node `22.22.3`, omdat officiële binaire bestanden van Node 24+ voor ARMv7 niet beschikbaar zijn.
    Op Alpine/musl Linux, waarvoor Node geen compatibele tarballs voor de vastgezette runtime publiceert, worden `nodejs` en `npm` geïnstalleerd met `apk`, waarna zowel Node als de daadwerkelijk gekoppelde SQLite-bibliotheek worden geverifieerd. De huidige stabiele pakketstromen van Alpine kunnen zelfs met een voldoende nieuwe Node nog steeds een kwetsbare SQLite-versie koppelen; gebruik een officiële `node:24-alpine`-container of een op glibc gebaseerde host wanneer het pakket door de veiligheidscontrole wordt afgewezen.
  </Step>
  <Step title="Git beschikbaar maken">
    Als Git ontbreekt, wordt geprobeerd dit te installeren via apt/dnf/yum/apk op Linux of Homebrew op macOS.
  </Step>
  <Step title="OpenClaw onder het voorvoegsel installeren">
    - `npm`-methode (standaard): installeert met npm onder het voorvoegsel en schrijft vervolgens de wrapper naar `<prefix>/bin/openclaw`
    - `git`-methode: kloont/werkt een checkout bij (standaard `~/openclaw`) en schrijft de wrapper nog steeds naar `<prefix>/bin/openclaw`

  </Step>
  <Step title="Geladen Gateway-service vernieuwen">
    Als er al een Gateway-service vanuit hetzelfde voorvoegsel is geladen, voert het script
    `openclaw gateway install --force` uit, waarmee de vervangende service wordt geactiveerd,
    en controleert het vervolgens zo goed mogelijk de status van de Gateway.
  </Step>
</Steps>

### Voorbeelden (install-cli.sh)

<Tabs>
  <Tab title="Standaard">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install-cli.sh | bash
    ```
  </Tab>
  <Tab title="Aangepast voorvoegsel + versie">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install-cli.sh | bash -s -- --prefix /opt/openclaw --version latest
    ```
  </Tab>
  <Tab title="Git-installatie">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install-cli.sh | bash -s -- --install-method git --git-dir ~/openclaw
    ```
  </Tab>
  <Tab title="JSON-uitvoer voor automatisering">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install-cli.sh | bash -s -- --json --prefix /opt/openclaw
    ```
  </Tab>
  <Tab title="Onboarding uitvoeren">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install-cli.sh | bash -s -- --onboard
    ```
  </Tab>
</Tabs>

<AccordionGroup>
  <Accordion title="Overzicht van vlaggen">

| Vlag                                    | Beschrijving                                                                     |
| --------------------------------------- | ------------------------------------------------------------------------------- |
| `--prefix <path>`                       | Installatievoorvoegsel (standaard: `~/.openclaw`)                                         |
| `--install-method \| --method npm\|git` | Installatiemethode kiezen (standaard: `npm`)                                          |
| `--npm`                                 | Snelkoppeling voor de npm-methode                                                         |
| `--git \| --github`                     | Snelkoppeling voor de git-methode                                                         |
| `--git-dir \| --dir <path>`             | Git-checkoutmap (standaard: `~/openclaw`)                                  |
| `--version <ver>`                       | OpenClaw-versie of dist-tag (standaard: `latest`)                                |
| `--node-version <ver>`                  | Node-versie (standaard: `24.15.0`; `22.22.3` op Linux ARMv7)                     |
| `--json`                                | NDJSON-gebeurtenissen uitvoeren                                                              |
| `--onboard`                             | `openclaw onboard` uitvoeren na installatie                                            |
| `--no-onboard`                          | Onboarding overslaan (standaard)                                                       |
| `--set-npm-prefix`                      | Op Linux het npm-voorvoegsel forceren naar `~/.npm-global` als het huidige voorvoegsel niet beschrijfbaar is |
| `--help \| -h`                          | Gebruik tonen                                                                      |

  </Accordion>

  <Accordion title="Naslag voor omgevingsvariabelen">

| Variabele                                    | Beschrijving                                                        |
| ------------------------------------------- | ------------------------------------------------------------------ |
| `OPENCLAW_PREFIX=<path>`                    | Installatievoorvoegsel                                                     |
| `OPENCLAW_INSTALL_METHOD=git\|npm`          | Installatiemethode                                                     |
| `OPENCLAW_VERSION=<ver>`                    | OpenClaw-versie of dist-tag                                       |
| `OPENCLAW_NODE_VERSION=<ver>`               | Node-versie                                                       |
| `OPENCLAW_HOME=<path>`                      | Basismap voor OpenClaw-status en standaardpaden voor git/onboarding |
| `OPENCLAW_GIT_DIR=<path>`                   | Git-checkoutmap voor git-installaties                            |
| `OPENCLAW_GIT_UPDATE=0\|1`                  | Git-updates in- of uitschakelen voor bestaande checkouts                          |
| `OPENCLAW_NO_ONBOARD=1`                     | Onboarding overslaan                                                    |
| `OPENCLAW_NPM_LOGLEVEL=error\|warn\|notice` | npm-logniveau (standaard: `error`)                                   |

  </Accordion>
</AccordionGroup>

<Note>
`openclaw@main` en andere GitHub-bronspecificaties zijn geen geldige `--version`-doelen voor npm-installaties. Gebruik in plaats daarvan `--install-method git --version main`.
</Note>

---

<a id="installps1"></a>

## install.ps1

### Verloop (install.ps1)

<Steps>
  <Step title="PowerShell- en Windows-omgeving garanderen">
    Vereist PowerShell 5+.
  </Step>
  <Step title="Standaard Node.js 24 garanderen">
    Als deze ontbreekt, wordt eerst installatie via winget geprobeerd, daarna Chocolatey en vervolgens Scoop. Als er geen pakketbeheerder beschikbaar is, downloadt het script het officiële Windows-zipbestand van Node.js 24 naar `%LOCALAPPDATA%\OpenClaw\deps\portable-node` en voegt het dit toe aan het PATH van het huidige proces en de gebruiker. Node 22.22.3+, Node 24.15+ en Node 25.9+ worden ondersteund; Node 23 wordt niet ondersteund.
  </Step>
  <Step title="OpenClaw installeren">
    - `npm`-methode (standaard): globale npm-installatie met de geselecteerde `-Tag`, gestart vanuit een beschrijfbare tijdelijke installatiemap zodat shells die in beveiligde mappen zoals `C:\` zijn geopend, blijven werken
    - `git`-methode: repository klonen/bijwerken, installeren/bouwen met pnpm en wrapper installeren in `%USERPROFILE%\.local\bin\openclaw.cmd`. Als Git ontbreekt, initialiseert het script gebruikerslokale MinGit onder `%LOCALAPPDATA%\OpenClaw\deps\portable-git` en voegt het dit toe aan het PATH van het huidige proces en de gebruiker.

  </Step>
  <Step title="Taken na installatie">
    - Voegt waar mogelijk de benodigde bin-map toe aan het gebruikers-PATH
    - Vernieuwt zo goed mogelijk een geladen Gateway-service (`openclaw gateway install --force`, daarna opnieuw starten)
    - Voert `openclaw doctor --non-interactive` uit bij upgrades en git-installaties (zo goed mogelijk)

  </Step>
  <Step title="Fouten afhandelen">
    `iwr ... | iex`- en scriptblock-installaties melden een beëindigende fout zonder de huidige PowerShell-sessie te sluiten. Directe installaties via `powershell -File` / `pwsh -File` worden voor automatisering nog steeds afgesloten met een niet-nulstatus.
  </Step>
</Steps>

### Voorbeelden (install.ps1)

<Tabs>
  <Tab title="Standaard">
    ```powershell
    iwr -useb https://openclaw.ai/install.ps1 | iex
    ```
  </Tab>
  <Tab title="Git-installatie">
    ```powershell
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -InstallMethod git
    ```
  </Tab>
  <Tab title="Checkout van GitHub main">
    ```powershell
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -InstallMethod git -Tag main
    ```
  </Tab>
  <Tab title="Aangepaste git-map">
    ```powershell
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -InstallMethod git -GitDir "C:\openclaw"
    ```
  </Tab>
  <Tab title="Proefuitvoering">
    ```powershell
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -DryRun
    ```
  </Tab>
</Tabs>

<AccordionGroup>
  <Accordion title="Naslag voor vlaggen">

| Vlag                        | Beschrijving                                                |
| --------------------------- | ---------------------------------------------------------- |
| `-InstallMethod npm\|git`   | Installatiemethode (standaard: `npm`)                            |
| `-Tag <tag\|version\|spec>` | npm-dist-tag, versie of pakketspecificatie (standaard: `latest`) |
| `-GitDir <path>`            | Checkoutmap (standaard: `%USERPROFILE%\openclaw`)     |
| `-NoOnboard`                | Onboarding overslaan                                            |
| `-NoGitUpdate`              | `git pull` overslaan                                            |
| `-DryRun`                   | Alleen acties afdrukken                                         |

  </Accordion>

  <Accordion title="Naslag voor omgevingsvariabelen">

| Variabele                           | Beschrijving        |
| ---------------------------------- | ------------------ |
| `OPENCLAW_INSTALL_METHOD=git\|npm` | Installatiemethode     |
| `OPENCLAW_GIT_DIR=<path>`          | Checkoutmap |
| `OPENCLAW_NO_ONBOARD=1`            | Onboarding overslaan    |
| `OPENCLAW_GIT_UPDATE=0`            | Git pull uitschakelen   |
| `OPENCLAW_DRY_RUN=1`               | Proefuitvoeringsmodus       |

  </Accordion>
</AccordionGroup>

<Note>
Als `-InstallMethod git` wordt gebruikt en Git ontbreekt, probeert het script gebruikerslokale MinGit te initialiseren voordat de koppeling naar Git for Windows wordt weergegeven.
</Note>

---

## CI en automatisering

Gebruik niet-interactieve vlaggen/omgevingsvariabelen voor voorspelbare uitvoeringen.

<Tabs>
  <Tab title="install.sh (niet-interactieve npm)">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --no-prompt --no-onboard
    ```
  </Tab>
  <Tab title="install.sh (niet-interactieve git)">
    ```bash
    OPENCLAW_INSTALL_METHOD=git OPENCLAW_NO_PROMPT=1 \
      curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash
    ```
  </Tab>
  <Tab title="install-cli.sh (JSON)">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install-cli.sh | bash -s -- --json --prefix /opt/openclaw
    ```
  </Tab>
  <Tab title="install.ps1 (onboarding overslaan)">
    ```powershell
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -NoOnboard
    ```
  </Tab>
</Tabs>

---

## Probleemoplossing

<AccordionGroup>
  <Accordion title="Waarom is Git vereist?">
    Git is vereist voor de installatiemethode `git`. Voor `npm`-installaties wordt Git nog steeds gecontroleerd/geïnstalleerd om `spawn git ENOENT`-fouten te voorkomen wanneer afhankelijkheden git-URL's gebruiken.
  </Accordion>

  <Accordion title="Waarom treedt bij npm EACCES op Linux op?">
    Sommige Linux-configuraties verwijzen met het globale npm-voorvoegsel naar paden die eigendom zijn van root. `install.sh` kan het voorvoegsel wijzigen in `~/.npm-global` en PATH-exports toevoegen aan shell-rc-bestanden (wanneer die bestanden bestaan).
  </Accordion>

  <Accordion title='Windows: "npm error spawn git / ENOENT"'>
    Voer het installatieprogramma opnieuw uit zodat het gebruikerslokale MinGit kan initialiseren, of installeer Git for Windows en open PowerShell opnieuw.
  </Accordion>

  <Accordion title='Windows: "openclaw is not recognized"'>
    Voer `npm config get prefix` uit en voeg die map toe aan je gebruikers-PATH (op Windows is geen achtervoegsel `\bin` nodig). Open daarna PowerShell opnieuw.
  </Accordion>

  <Accordion title="Windows: uitgebreide uitvoer van het installatieprogramma verkrijgen">
    `install.ps1` biedt geen schakeloptie `-Verbose`.
    Gebruik PowerShell-tracering voor diagnostiek op scriptniveau:

    ```powershell
    Set-PSDebug -Trace 1
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -NoOnboard
    Set-PSDebug -Trace 0
    ```

  </Accordion>

  <Accordion title="openclaw niet gevonden na installatie">
    Meestal is dit een PATH-probleem. Zie [Probleemoplossing voor Node.js](/nl/install/node#troubleshooting).
  </Accordion>
</AccordionGroup>

## Gerelateerd

- [Installatieoverzicht](/nl/install)
- [Bijwerken](/nl/install/updating)
- [Verwijderen](/nl/install/uninstall)
