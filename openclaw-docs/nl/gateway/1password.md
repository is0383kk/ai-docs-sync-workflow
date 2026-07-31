---
read_when:
    - Je wilt API-sleutels uit openclaw.json halen en in 1Password opslaan
    - Je voert de Gateway headless uit en hebt serviceaccountverificatie nodig voor op
    - Je wilt dat agents geheimen lezen of invoegen met de op CLI
summary: Los Gateway-geheimen op met de 1Password CLI en laat agents de meegeleverde 1password-skill gebruiken
title: 1Password
x-i18n:
    generated_at: "2026-07-27T05:33:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5bb14944f0b3ce1ee3f90bf666a53e8673e7a9861e3e138a5fabe9c8e070cbd7
    source_path: gateway/1password.md
    workflow: 16
---

OpenClaw werkt op drie onafhankelijke manieren samen met **1Password**:

- **Configuratiegeheimen:** elk [SecretRef](/nl/gateway/secrets)-veld in `openclaw.json` kan tijdens runtime worden opgehaald via de `op`-CLI, zodat API-sleutels nooit in het configuratiebestand staan.
- **Agentworkflows:** de meegeleverde `1password`-skill leert agents om in te loggen en met `op` geheimen te lezen of te injecteren voor hun eigen taken.
- **Inloggen via de browser:** de `claude-cli`-backend kan Claude Codes Chrome-integratie gebruiken met [1Password voor Claude](https://support.1password.com/1password-claude/), zodat de agent op websites kan inloggen zonder dat het wachtwoord ooit bij het model of OpenClaw terechtkomt.

## Vereisten

- De [1Password-CLI](https://developer.1password.com/docs/cli/get-started/) (`op`) moet op de Gateway-host zijn geïnstalleerd (`brew install 1password-cli` op macOS).
- Een verificatiemethode voor `op`:
  - **Serviceaccount** (aanbevolen voor headless Gateways): exporteer `OP_SERVICE_ACCOUNT_TOKEN` in de omgeving van de Gateway-service. Geen desktop-app en geen interactieve aanmelding.
  - **Integratie met de desktop-app**: de 1Password-app draait op dezelfde machine en de CLI-integratie is ingeschakeld. De eerste aanroepen kunnen Touch ID of systeemverificatie activeren.
  - **Zelfstandig inloggen**: `op signin` vraagt dit per sessie. Dit is via de skill bruikbaar voor agents, maar niet geschikt om configuratiegeheimen op een headless Gateway op te halen.

## Configuratiegeheimen ophalen met op

Declareer een exec-provider voor geheimen die `op read` uitvoert met een `op://vault/item/field`-verwijzing en laat vervolgens elk veld dat SecretRef ondersteunt hiernaar verwijzen:

```json5
{
  secrets: {
    providers: {
      onepassword_openai: {
        source: "exec",
        command: "/opt/homebrew/bin/op",
        allowSymlinkCommand: true, // vereist voor via symbolische koppelingen geïnstalleerde Homebrew-binaire bestanden
        trustedDirs: ["/opt/homebrew"],
        args: ["read", "op://Personal/OpenClaw QA API Key/password"],
        passEnv: ["HOME"],
        jsonOnly: false,
      },
    },
  },
  models: {
    providers: {
      openai: {
        baseUrl: "https://api.openai.com/v1",
        models: [{ id: "gpt-5", name: "gpt-5" }],
        apiKey: { source: "exec", provider: "onepassword_openai", id: "value" },
      },
    },
  },
}
```

Zo passen de onderdelen bij elkaar:

- `command` moet een absoluut pad zijn; `trustedDirs` markeert de bijbehorende map als vertrouwd en `allowSymlinkCommand` is nodig omdat Homebrew `op` als symbolische koppeling installeert.
- `args` geeft de `op://vault/item/field`-verwijzing ongewijzigd door. OpenClaw verwerkt het `op://`-schema niet zelf; het binaire bestand `op` haalt de waarde op.
- `passEnv` geeft de vermelde variabelen uit de Gateway-omgeving door. Voor integratie met de desktop-app is `HOME` nodig; voor serviceaccounts moet `OP_SERVICE_ACCOUNT_TOKEN` ook aanwezig zijn in de omgeving van de Gateway-service (voeg deze toe aan `passEnv`, of stel deze alleen via `env` in als je accepteert dat het token in het configuratiebestand kan worden gelezen).
- Behoud `id: "value"` voor uitvoer met één waarde. Gebruik bij `jsonOnly: true` en een JSON-payload in plaats daarvan een JSON-pointer-id om velden te adresseren.
- Met één providervermelding per geheim blijven verwijzingen controleerbaar; vernoem providers naar hun gebruiker (`onepassword_openai`, `onepassword_telegram`).

Zie [Gateway-geheimen](/nl/gateway/secrets) voor de ophaalvolgorde, caching en het gedrag bij fouten, en [SecretRef-referentieoppervlak](/nl/reference/secretref-credential-surface) voor elk veld dat SecretRefs accepteert.

## Een serviceaccount instellen voor headless Gateways

1. Maak een serviceaccount aan in je 1Password-account en geef dit alleen leestoegang tot de kluisitems die de Gateway nodig heeft.
2. Geef `OP_SERVICE_ACCOUNT_TOKEN` door aan de Gateway-service (launchd-plist, systemd-unit of containeromgeving).
3. Voeg `"OP_SERVICE_ACCOUNT_TOKEN"` toe aan de `passEnv`-lijst van de provider.
4. Controleer dit vanuit de omgeving van de Gateway-host: `op whoami` moet het serviceaccount zonder prompt weergeven.

Voor leesbewerkingen van serviceaccounts moet de kluis expliciet in de `op://`-verwijzing worden genoemd. Beperk de accountrechten zorgvuldig; het is een bearer-referentie.

## De 1password-skill voor agents

OpenClaw bevat een `1password`-skill die agents verandert in bekwame `op`-operators: deze detecteert de beschikbare verificatiemethode (serviceaccount, integratie met de desktop-app of zelfstandig inloggen), verifieert met `op whoami` de toegang voordat iets wordt gelezen en geeft de voorkeur aan `op run` / `op inject` boven het naar schijf schrijven van geheime waarden. Voor de skill is het binaire bestand `op` vereist; als dit ontbreekt, wordt installatie via Homebrew aangeboden.

Agents gebruiken deze skill voor hun eigen workflows, bijvoorbeeld om halverwege een taak een implementatietoken te lezen of om omgevingsvariabelen in een opdracht te injecteren. Dit staat los van het ophalen van configuratiegeheimen; de Gateway haalt SecretRefs op zonder dat daarbij een skill betrokken is.

## Inloggen via de browser met 1Password voor Claude

Met [1Password voor Claude](https://support.1password.com/1password-claude/) kan Claude om aanmeldgegevens vragen, waarna de 1Password-browserextensie de referentie via een versleuteld kanaal rechtstreeks op de pagina invult. Het geheim komt nooit in de modelcontext, het transcript of OpenClaw terecht. Wanneer OpenClaw de [`claude-cli`-backend](/nl/gateway/cli-backends#claude-cli-specifics) uitvoert terwijl Claude Codes Chrome-integratie is ingeschakeld, kunnen agenttaken deze flow gebruiken voor websites waarvoor een echte aangemelde sessie nodig is.

Naast de backend zelf is hiervoor het volgende vereist:

- Een macOS-Gateway-host met Chrome, de verbonden [Claude in Chrome-extensie](https://code.claude.com/docs/en/chrome), de 1Password-desktop-app en de 1Password-browserextensie (beide versie 8.12.28 of nieuwer).
- Claude Code moet zijn aangemeld bij een rechtstreeks Anthropic-abonnement (Pro, Max, Team of Enterprise). Chrome-integratie is niet beschikbaar via Amazon Bedrock, Google Cloud of andere externe providers.
- De eenmalige 1Password-verbinding aan de kant van Anthropic: 1Password voor Claude wordt ingesteld via de Claude-desktop-app of de extensieflow die wordt beschreven in [de handleiding van 1Password](https://support.1password.com/1password-claude/), en is momenteel een macOS-bèta. Bij 1Password Business moet een beheerder eerst "Allow AI agents to autofill for users" inschakelen onder Policies; bij Anthropic Team/Enterprise-abonnementen is de integratie eveneens standaard uitgeschakeld totdat een Owner deze inschakelt.
- Een [CLI-backendplugin](/nl/plugins/cli-backend-plugins) die `--chrome` toevoegt aan de startargumenten van Claude; de meegeleverde backend schakelt Chrome niet in.
- Een persoon bij de Gateway-host: bij elk gebruik van een referentie verschijnt daar een 1Password-prompt die moet worden bevestigd (bijvoorbeeld met Touch ID). Bij een restrictief uitvoeringsbeleid worden de browsertoolaanroepen zelf ook eerst als OpenClaw-goedkeuringen naar je kanaal doorgestuurd.

Voordat je dit met OpenClaw verbindt, controleer je de onderdelen in een interactieve sessie op de Gateway-host: voer `claude --chrome` uit, bevestig dat de extensie verbinding maakt en controleer of de `claude-in-chrome`-tools de referentietools bevatten. Als deze daar niet verschijnen, verschijnen ze ook niet via OpenClaw.

Eenmalige toegangscodes worden door 1Password op dezelfde pagina ingevuld; stuur verificatiecodes of wachtwoorden nooit via chat door. Headless Gateways en externe Gateways kunnen deze flow momenteel niet gebruiken, omdat zowel de goedkeuring als de browser zich op de Gateway-host bevinden.

## Beveiligingsopmerkingen

- Geheime waarden die via exec-providers worden opgehaald, blijven in het geheugen van de Gateway; in configuratiemomentopnamen en `config.get`-antwoorden worden SecretRef-velden geredigeerd.
- Plaats geheime waarden nooit in `openclaw.json`, logboeken of chatberichten. Bewaar itemnamen in de configuratie en waarden in 1Password.
- Het auditlogboek van 1Password toont elke leesbewerking door een serviceaccount, waardoor sleutelrotatie en incidentonderzoek praktisch uitvoerbaar zijn.

## Problemen oplossen

- `command not found`- of spawnfouten: gebruik het absolute pad naar `op` en neem de bijbehorende map op in `trustedDirs`.
- `op` wordt gevonden, maar leesbewerkingen mislukken met fouten over symbolische koppelingen: stel `allowSymlinkCommand: true` in voor Homebrew-installaties.
- `account is not signed in`: controleer voor serviceaccounts of `OP_SERVICE_ACCOUNT_TOKEN` de Gateway-service bereikt en in `passEnv` wordt vermeld; controleer voor desktopintegratie of de app actief en ontgrendeld is.
- Trage eerste leesbewerkingen: verhoog `timeoutMs` voor de provider; koude starts van `op` kunnen op drukke hosts strikte time-outs overschrijden.
