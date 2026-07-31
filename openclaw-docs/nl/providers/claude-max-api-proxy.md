---
read_when:
    - Je wilt een Claude Max-abonnement gebruiken met OpenAI-compatibele tools
    - Je wilt een lokale API-server die de Claude Code CLI omhult
    - Je wilt Anthropic-toegang op basis van een abonnement vergelijken met toegang op basis van een API-sleutel
summary: Communityproxy om Claude-abonnementsgegevens beschikbaar te stellen als een OpenAI-compatibel eindpunt
title: Claude Max API-proxy
x-i18n:
    generated_at: "2026-07-27T06:05:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5d0d9a70e14d7d444e57e9bcf169816fec4013a2680dfc9b1761e6ab32109e9f
    source_path: providers/claude-max-api-proxy.md
    workflow: 16
---

**claude-max-api-proxy** is een community-npm-pakket (geen OpenClaw-plugin) dat
een Claude Max/Pro-abonnement beschikbaar stelt als een OpenAI-compatibel API-eindpunt, zodat
je elk OpenAI-compatibel hulpprogramma naar je abonnement kunt laten verwijzen in plaats van naar een
Anthropic-API-sleutel.

<Warning>
Alleen technisch compatibel, geen officieel goedgekeurde methode. Anthropic heeft
in het verleden bepaald abonnementsgebruik buiten Claude Code geblokkeerd; controleer
de huidige factureringsregels van Anthropic voordat je hierop vertrouwt.

De Claude Code-documentatie van Anthropic beschrijft `claude -p` als Agent SDK-/programmatisch
gebruik. Sinds de supportupdate van Anthropic van 15 juni 2026 vallen Claude Agent SDK,
`claude -p` en het gebruik van apps van derden onder de gebruikslimieten van het
aangemelde abonnement (het eerder aangekondigde afzonderlijke tegoedplan voor Agent SDK is
gepauzeerd). Zie het [artikel over het Agent SDK-plan van
Anthropic](https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan),
de artikelen over de [Pro/Max](https://support.claude.com/en/articles/11145838-use-claude-code-with-your-pro-or-max-plan)-
en [Team/Enterprise](https://support.claude.com/en/articles/11845131-use-claude-code-with-your-team-or-enterprise-plan)-plannen,
en de [Anthropic-provider](/nl/providers/anthropic) voor OpenClaws
eigen opmerkingen over facturering voor Claude CLI.
</Warning>

## Waarom dit gebruiken

| Aanpak                    | Kostenroute                                      | Meest geschikt voor                                |
| ------------------------- | ----------------------------------------------- | -------------------------------------------------- |
| Anthropic-API-sleutel     | Betalen per token via Claude Console             | Productie-apps, gedeelde automatisering, volume    |
| Claude-abonnementsproxy   | Plan- en tegoedregels van Claude Code / `claude -p` | Persoonlijke experimenten met compatibele hulpmiddelen |

Met deze proxy werkt een Claude Max- of Pro-abonnement met OpenAI-compatibele
hulpmiddelen. Het is geen onbeperkte route met een vast tarief: de gebruikslimieten
van Claude Code zijn van toepassing. API-sleutels blijven de duidelijkere
factureringsroute voor productiegebruik.

## Hoe het werkt

```text
Jouw app -> claude-max-api-proxy -> Claude Code CLI / claude -p -> Anthropic
     (OpenAI-indeling)             (converteert indeling)           (gebruikt je aanmelding)
```

De proxy start voor elke aanvraag de Claude Code CLI als een subproces, zet
chatverzoeken in OpenAI-indeling om in CLI-prompts en streamt de
respons terug (of retourneert deze) in OpenAI-indeling.

## Aan de slag

<Steps>
  <Step title="De proxy installeren">
    Vereist Node.js 20+ en een geauthenticeerde Claude Code CLI.

    ```bash
    npm install -g claude-max-api-proxy

    # Controleren of Claude CLI is geauthenticeerd
    claude --version
    claude auth login   # indien nog niet geauthenticeerd
    ```

  </Step>
  <Step title="De server starten">
    ```bash
    claude-max-api
    # Server draait op http://localhost:3456
    ```
  </Step>
  <Step title="De proxy testen">
    ```bash
    curl http://localhost:3456/health
    curl http://localhost:3456/v1/models

    curl http://localhost:3456/v1/chat/completions \
      -H "Content-Type: application/json" \
      -d '{
        "model": "claude-opus-4",
        "messages": [{"role": "user", "content": "Hallo!"}]
      }'
    ```

  </Step>
  <Step title="OpenClaw configureren">
    Laat OpenClaw naar de proxy verwijzen als een aangepast OpenAI-compatibel eindpunt:

    ```json5
    {
      env: {
        OPENAI_API_KEY: "not-needed",
        OPENAI_BASE_URL: "http://localhost:3456/v1",
      },
      agents: {
        defaults: {
          model: { primary: "openai/claude-opus-4" },
        },
      },
    }
    ```

  </Step>
</Steps>

<Note>
De onderstaande model-id's behoren tot de eigen catalogus van de proxy, niet tot de
Anthropic-modelverwijzingen van OpenClaw. Elke id wordt gekoppeld aan een Claude Code
CLI-modelalias (`opus`, `sonnet`, `haiku`), waardoor het onderliggende model verandert
wanneer Anthropic die alias in de CLI bijwerkt. Controleer het huidige README-bestand
van de proxy voordat je op een specifieke koppeling vertrouwt.
</Note>

| Model-id          | CLI-alias | Huidige koppeling |
| ----------------- | --------- | ----------------- |
| `claude-opus-4`   | `opus`    | Claude Opus 4.5   |
| `claude-sonnet-4` | `sonnet`  | Claude Sonnet 4   |
| `claude-haiku-4`  | `haiku`   | Claude Haiku 4    |

## Geavanceerde configuratie

<AccordionGroup>
  <Accordion title="Opmerkingen over de proxy-achtige OpenAI-compatibele route">
    Dit gebruikt OpenClaws algemene aangepaste `/v1` OpenAI-compatibele route, hetzelfde
    pad als elke andere zelfgehoste OpenAI-compatibele backend:

    - Aanvraagvorming die uitsluitend voor native OpenAI geldt, is niet van toepassing.
    - `/fast` en `service_tier` zijn alleen van toepassing op rechtstreeks `api.anthropic.com`-
      verkeer; proxyroutes laten `service_tier` ongewijzigd (zie
      [snelle modus van de Anthropic-provider](/nl/providers/anthropic#advanced-configuration)).
    - Geen Responses-`store`, aanwijzingen voor promptcaching of
      OpenAI-reasoningcompatibele vormgeving van payloads.
    - De OpenAI/Codex-toeschrijvingsheaders van OpenClaw (`originator`, `version`,
      `User-Agent`) worden alleen verzonden bij native `api.openai.com` OAuth-verkeer, niet
      naar aangepaste `OPENAI_BASE_URL`-doelen zoals deze proxy.

  </Accordion>

  <Accordion title="Automatisch starten op macOS met LaunchAgent">
    ```bash
    cat > ~/Library/LaunchAgents/com.claude-max-api.plist << 'EOF'
    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
    <plist version="1.0">
    <dict>
      <key>Label</key>
      <string>com.claude-max-api</string>
      <key>RunAtLoad</key>
      <true/>
      <key>KeepAlive</key>
      <true/>
      <key>ProgramArguments</key>
      <array>
        <string>/usr/local/bin/node</string>
        <string>/usr/local/lib/node_modules/claude-max-api-proxy/dist/server/standalone.js</string>
      </array>
      <key>EnvironmentVariables</key>
      <dict>
        <key>PATH</key>
        <string>/usr/local/bin:/opt/homebrew/bin:~/.local/bin:/usr/bin:/bin</string>
      </dict>
    </dict>
    </plist>
    EOF

    launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.claude-max-api.plist
    ```

  </Accordion>
</AccordionGroup>

## Opmerkingen

- Neemt het gedrag voor facturering, gebruikstegoed en snelheidslimieten van `claude -p` van Claude Code over.
- Luistert uitsluitend op `127.0.0.1`; verzendt geen gegevens naar servers van derden, afgezien van de eigen aanroep van de CLI naar Anthropic.
- Streamingresponsen worden ondersteund.
- Authenticatiefouten worden bij het opstarten niet gecontroleerd en worden pas zichtbaar wanneer er daadwerkelijk een chatverzoek wordt uitgevoerd; als de CLI niet is geauthenticeerd, mislukt naar verwachting het eerste verzoek in plaats van dat de server weigert te starten.

<Note>
Zie de [Anthropic-provider](/nl/providers/anthropic) voor native Anthropic-integratie met Claude CLI of API-sleutels. Zie de [OpenAI-provider](/nl/providers/openai) voor OpenAI/Codex-abonnementen.
</Note>

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Anthropic-provider" href="/nl/providers/anthropic" icon="bolt">
    Native OpenClaw-integratie met Claude CLI of API-sleutels.
  </Card>
  <Card title="OpenAI-provider" href="/nl/providers/openai" icon="robot">
    Voor OpenAI/Codex-abonnementen.
  </Card>
  <Card title="Modelselectie" href="/nl/concepts/model-providers" icon="layers">
    Overzicht van alle providers, modelverwijzingen en failovergedrag.
  </Card>
  <Card title="Configuratie" href="/nl/gateway/configuration" icon="gear">
    Volledige configuratiereferentie.
  </Card>
</CardGroup>
