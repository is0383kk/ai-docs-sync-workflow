---
read_when:
    - Authenticatie van modellen of verlopen OAuth debuggen
    - Authenticatie of opslag van referenties documenteren
summary: 'Modelauthenticatie: OAuth, API-sleutels, hergebruik van de Claude CLI en Anthropic-installatietoken'
title: Authenticatie
x-i18n:
    generated_at: "2026-07-27T05:49:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1fd4bf1c73f41d297638811f568c1b11e920eba3bd1527206cbb760df51531f2
    source_path: gateway/authentication.md
    workflow: 16
---

<Note>
Deze pagina behandelt authenticatie bij **modelproviders** (API-sleutels, OAuth, hergebruik van de Claude CLI, Anthropic-installatietoken). Zie [Configuratie](/nl/gateway/configuration) en [Authenticatie via vertrouwde proxy](/nl/gateway/trusted-proxy-auth) voor authenticatie van de **Gateway-verbinding** (token, wachtwoord, vertrouwde proxy).
</Note>

OpenClaw ondersteunt OAuth en API-sleutels voor modelproviders. Voor een Gateway-host die altijd actief is, is een API-sleutel de meest voorspelbare optie; abonnements-/OAuth-stromen werken ook wanneer ze aansluiten bij het accountmodel van je provider.

- Volledige OAuth-stroom en opslagindeling: [/concepts/oauth](/nl/concepts/oauth)
- Authenticatie op basis van SecretRef (`env`/`file`/`exec`-providers): [Geheimenbeheer](/nl/gateway/secrets)
- Geschiktheids-/redencodes voor aanmeldgegevens die door `models status --probe` worden gebruikt: [Semantiek van aanmeldgegevens voor authenticatie](/nl/auth-credential-semantics)

## Aanbevolen configuratie: API-sleutel (elke provider)

1. Maak een API-sleutel aan in de beheerconsole van je provider.
2. Plaats deze op de **Gateway-host** (de machine waarop `openclaw gateway` wordt uitgevoerd):

```bash
export <PROVIDER>_API_KEY="..."
openclaw models status
```

3. Als de Gateway onder systemd/launchd draait, plaats je de sleutel in `~/.openclaw/.env`, zodat de daemon deze kan lezen:

```bash
cat >> ~/.openclaw/.env <<'EOF'
<PROVIDER>_API_KEY=...
EOF
```

4. Start het Gateway-proces (of de daemon) opnieuw en controleer het daarna nogmaals:

```bash
openclaw models status
openclaw doctor
```

`openclaw onboard` kan ook API-sleutels opslaan voor gebruik door de daemon als je omgevingsvariabelen niet zelf wilt beheren. Zie [Omgevingsvariabelen](/nl/help/environment) voor de volledige voorrangsvolgorde bij het laden van omgevingsvariabelen (`env.shellEnv`, `~/.openclaw/.env`, systemd/launchd).

## Anthropic: hergebruik van de Claude CLI

Authenticatie met een Anthropic-installatietoken blijft een ondersteunde methode. Hergebruik van de Claude CLI (gebruik in de stijl van `claude -p`) is ook toegestaan voor deze integratie; wanneer een Claude CLI-aanmelding beschikbaar is op de host, heeft die methode de voorkeur voor lokaal/desktopgebruik. Voor langlevende Gateway-hosts blijft een Anthropic-API-sleutel de meest voorspelbare keuze, met expliciete controle over facturering aan de serverzijde.

Hostconfiguratie voor hergebruik van de Claude CLI:

```bash
# Uitvoeren op de Gateway-host
claude auth login
claude auth status --text
openclaw models auth login --provider anthropic --method cli --set-default
```

Dit bestaat uit twee stappen: meld Claude Code op de host aan bij Anthropic en geef vervolgens OpenClaw opdracht om de selectie van Anthropic-modellen via de lokale `claude-cli`-backend te routeren en het bijbehorende OpenClaw-authenticatieprofiel op te slaan.

De Gateway-service moet `claude` kunnen vinden via `PATH`. Als een implementatie een
niet-standaardpad naar het uitvoerbare bestand vereist, registreer je een wrapper via een
[CLI-backendplugin](/nl/plugins/cli-backend-plugins).

## Handmatige tokeninvoer

Werkt voor elke provider; schrijft naar de SQLite-authenticatieopslag per agent en werkt de configuratie bij:

```bash
openclaw models auth paste-token --provider openrouter
```

OpenClaw leest authenticatieprofielen uit de `openclaw-agent.sqlite` van elke agent. Endpointdetails (`baseUrl`, `api`, model-id's, headers, time-outs) horen onder `models.providers.<id>` in `openclaw.json` of `models.json`, niet in authenticatieprofielen.

Als een oudere installatie nog `auth-profiles.json`, `auth-state.json` of een platte structuur zoals `{ "openrouter": { "apiKey": "..." } }` bevat, voer je `openclaw doctor --fix` uit om deze in SQLite te importeren; doctor bewaart back-ups met tijdstempels naast de oorspronkelijke JSON-bestanden.

Externe authenticatieroutes, zoals Bedrock `auth: "aws-sdk"`, zijn geen aanmeldgegevens. Stel voor een benoemde Bedrock-route `auth.profiles.<id>.mode: "aws-sdk"` in `openclaw.json` in — schrijf `type: "aws-sdk"` niet naar de opslag voor authenticatieprofielen. `openclaw doctor --fix` migreert verouderde AWS SDK-markeringen van de opslag voor aanmeldgegevens naar configuratiemetadata.

### Aanmeldgegevens op basis van SecretRef

- `api_key`-aanmeldgegevens kunnen `keyRef: { source, provider, id }` gebruiken
- `token`-aanmeldgegevens kunnen `tokenRef: { source, provider, id }` gebruiken
- Profielen in OAuth-modus weigeren SecretRef-aanmeldgegevens: als `auth.profiles.<id>.mode` gelijk is aan `"oauth"`, wordt een door SecretRef ondersteunde `keyRef`/`tokenRef` voor dat profiel geweigerd.

## De authenticatiestatus van modellen controleren

```bash
openclaw models status
openclaw doctor
```

Automatiseringsvriendelijke controle, met afsluitcode `1` bij verlopen/ontbrekende gegevens en `2` bij bijna verlopen gegevens:

```bash
openclaw models status --check
```

Live-authenticatiecontroles (voeg `--probe-provider`, `--probe-profile`, `--probe-timeout`, `--probe-concurrency` of `--probe-max-tokens` toe om het bereik te beperken):

```bash
openclaw models status --probe
```

Opmerkingen:

- Controleregels kunnen afkomstig zijn van authenticatieprofielen, aanmeldgegevens uit omgevingsvariabelen of `models.json`.
- Als `auth.order.<provider>` een opgeslagen profiel weglaat, meldt de controle `excluded_by_auth_order` voor dat profiel in plaats van het te proberen.
- Als authenticatie aanwezig is, maar OpenClaw geen controleerbaar model voor die provider kan vinden, meldt de controle `status: no_model`.
- Afkoelperiodes na snelheidsbeperkingen kunnen modelspecifiek zijn: een profiel dat voor één model in een afkoelperiode zit, kan nog steeds een verwant model van dezelfde provider bedienen.

Optionele beheerscripts (systemd/Termux): [Scripts voor authenticatiebewaking](/nl/help/scripts#auth-monitoring-scripts).

## Rotatie van API-sleutels (Gateway)

Sommige providers proberen een aanvraag opnieuw met een andere geconfigureerde sleutel wanneer een aanroep de snelheidslimiet van de provider bereikt.

Prioriteitsvolgorde van sleutels per provider:

1. `OPENCLAW_LIVE_<PROVIDER>_KEY` (één overschrijving, zet één sleutel vast)
2. `<PROVIDER>_API_KEYS` (lijst gescheiden door komma's/spaties/puntkomma's)
3. `<PROVIDER>_API_KEY`
4. `<PROVIDER>_API_KEY_*` (elke omgevingsvariabele met dit voorvoegsel)

Google-providers (`google`, `google-vertex`) vallen daarnaast terug op `GOOGLE_API_KEY`. De gecombineerde lijst wordt vóór gebruik ontdubbeld.

OpenClaw schakelt alleen over naar de volgende sleutel wanneer de foutmelding overeenkomt met: `rate_limit`, `rate limit`, `429`, `quota exceeded`/`quota_exceeded`, `resource exhausted`/`resource_exhausted` of `too many requests`. Andere fouten worden niet opnieuw geprobeerd met alternatieve sleutels. Als alle sleutels mislukken, wordt de uiteindelijke fout van de laatste poging geretourneerd.

<Note>
Providerspecifieke formuleringen zoals `ThrottlingException`, `concurrency limit reached` of `workers_ai ... quota limit exceeded` bepalen de **classificatie voor failover/opnieuw proberen** (overschakelen tussen modellen of providers na herhaalde fouten), een afzonderlijk mechanisme van de bovenstaande rotatie van API-sleutels.
</Note>

Het verwijderen van opgeslagen authenticatie trekt de sleutel bij de provider niet in — roteer of trek deze in via het dashboard van de provider wanneer je de sleutel aan de providerzijde ongeldig wilt maken.

## Providerauthenticatie verwijderen terwijl de Gateway actief is

Wanneer je providerauthenticatie verwijdert via het besturingsvlak van de Gateway, verwijdert OpenClaw de opgeslagen authenticatieprofielen voor die provider en breekt het actieve chat-/agentruns af waarvan de geselecteerde modelprovider overeenkomt met de verwijderde provider. Afgebroken runs zenden de normale annulerings-/levenscyclusgebeurtenissen uit met `stopReason: "auth-revoked"`, zodat verbonden clients kunnen tonen dat de run is gestopt omdat de aanmeldgegevens zijn verwijderd.

## Bepalen welke aanmeldgegevens worden gebruikt

### OpenAI en verouderde `openai-codex`-id's

OpenAI-profielen met API-sleutels en ChatGPT/Codex OAuth-profielen gebruiken beide de canonieke provider-id `openai`. Gebruik `openai:*`-profiel-id's en `auth.order.openai` voor nieuwe configuratie.

Als je `openai-codex` aantreft in een oudere configuratie, authenticatieprofiel-id's of `auth.order.openai-codex`, behandel je dit als invoer voor een verouderde migratie — maak geen nieuwe `openai-codex`-profielen aan. Voer het volgende uit:

```bash
openclaw doctor --fix
openclaw models auth list --provider openai
```

Doctor herschrijft verouderde `openai-codex:*`-profiel-id's en `auth.order.openai-codex`-vermeldingen naar de canonieke `openai`-route. Zie [OpenAI](/nl/providers/openai) voor OpenAI-specifieke routering van modellen/runtimes.

### Tijdens aanmelding (CLI)

```bash
openclaw models auth login --provider openai --profile-id openai:ritsuko
openclaw models auth login --provider openai --profile-id openai:lain
```

`--profile-id` houdt meerdere OAuth-aanmeldingen voor dezelfde provider binnen één agent gescheiden.

`--force` verwijdert de opgeslagen authenticatieprofielen voor die provider in de geselecteerde agentmap en voert daarna dezelfde authenticatiestroom opnieuw uit. Gebruik dit wanneer een opgeslagen profiel vastzit, verlopen is of aan het verkeerde account is gekoppeld. Hiermee worden de aanmeldgegevens bij de provider niet ingetrokken.

```bash
openclaw models auth login --provider anthropic --force
```

### Per sessie (chatopdracht)

- `/model <alias-or-id>@<profileId>` zet specifieke provideraanmeldgegevens vast voor de huidige sessie (voorbeeldprofiel-id's: `anthropic:default`, `anthropic:work`).
- `/model` (of `/model list`) toont een compacte kiezer; `/model status` toont de volledige weergave (kandidaten + volgend authenticatieprofiel, plus details van het providerendpoint indien geconfigureerd).

Als je de authenticatievolgorde of het vastzetten van profielen wijzigt voor een chat die al actief is, stuur je `/new` of `/reset` om een nieuwe sessie te starten — bestaande sessies behouden hun huidige model-/profielselectie totdat ze opnieuw worden ingesteld.

### Per agent (CLI-overschrijving)

Overschrijvingen van de authenticatievolgorde worden opgeslagen in de SQLite-authenticatiestatus van die agent:

```bash
openclaw models auth order get --provider anthropic
openclaw models auth order set --provider anthropic anthropic:default
openclaw models auth order clear --provider anthropic
```

Gebruik `--agent <id>` om een specifieke agent te selecteren; laat dit weg om de geconfigureerde standaardagent te gebruiken. `openclaw models status --probe` toont weggelaten opgeslagen profielen als `excluded_by_auth_order` in plaats van ze stilzwijgend over te slaan.

## Problemen oplossen

### "Geen aanmeldgegevens gevonden"

Configureer een Anthropic-API-sleutel op de **Gateway-host** of stel het Anthropic-installatietokenpad in en controleer het daarna opnieuw:

```bash
openclaw models status
```

### Token verloopt binnenkort/is verlopen

Voer `openclaw models status` uit om te zien welk profiel binnenkort verloopt. Als een Anthropic-tokenprofiel ontbreekt of verlopen is, vernieuw je het via een installatietoken of migreer je naar een Anthropic-API-sleutel.

## Gerelateerd

- [Geheimenbeheer](/nl/gateway/secrets)
- [Externe toegang](/nl/gateway/remote)
- [Authenticatieopslag](/nl/concepts/oauth)
