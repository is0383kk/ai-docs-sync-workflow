---
read_when:
    - Je wilt gehoste Ollama-modellen gebruiken zonder een lokale Ollama-server
    - Je hebt de provider-id, sleutel of het eindpunt van ollama-cloud nodig
summary: Gebruik Ollama Cloud rechtstreeks met OpenClaw
title: Ollama Cloud
x-i18n:
    generated_at: "2026-07-27T05:31:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 966e5237e37134cef109979079db390e9844714001e921e7976dc8ca7f58bcc4
    source_path: providers/ollama-cloud.md
    workflow: 16
---

Ollama Cloud is de gehoste model-API van Ollama. De provider `ollama-cloud` roept deze
rechtstreeks aan via `https://ollama.com` met de native `/api/chat`-API van Ollama, zonder
lokale Ollama-server en zonder lokale Ollama-app die bij de cloudmodus is aangemeld. Gebruik model-
referenties zoals `ollama-cloud/kimi-k2.6`.

OpenClaw registreert `ollama-cloud` als een eigen provider-id, zodat cloudspecifieke
inloggegevens, live catalogusdetectie en modelselectie niet worden vermengd met
een lokale `ollama`-host. Zie [Ollama](/nl/providers/ollama) voor lokale Ollama, hybride routering
via cloud en lokaal, embeddings en details over aangepaste hosts.

## Configuratie

Maak een Ollama Cloud-API-sleutel aan op [ollama.com/settings/keys](https://ollama.com/settings/keys) en voer daarna het volgende uit:

```bash
openclaw onboard --auth-choice ollama-cloud
```

Of stel het volgende in:

```bash
export OLLAMA_API_KEY="<your-ollama-cloud-api-key>" # pragma: allowlist secret
```

Niet-interactieve onboarding accepteert de sleutel rechtstreeks:

```bash
openclaw onboard --auth-choice ollama-cloud --ollama-cloud-api-key "<key>"
```

Onboarding stelt het standaardmodel in op `ollama-cloud/kimi-k2.5:cloud`.

## Standaardwaarden

- Provider: `ollama-cloud`
- Basis-URL: `https://ollama.com`
- Omgevingsvariabele: `OLLAMA_API_KEY`
- API-stijl: native Ollama-`/api/chat`
- Standaardmodel voor onboarding: `ollama-cloud/kimi-k2.5:cloud`

## Wanneer kies je voor Ollama Cloud?

- Je wilt gehoste Ollama-modellen zonder `ollama serve` lokaal uit te voeren.
- Je wilt dezelfde native vorm van de Ollama-chat-API die OpenClaw voor lokale
  Ollama gebruikt, maar dan gericht op `https://ollama.com`.
- Je wilt een eenvoudig cloudpad voor modellen die al in de gehoste
  catalogus van Ollama staan.
- Je hebt geen lokale modeldownloads, lokale GPU-besturing of inferentie uitsluitend via het LAN nodig.

Gebruik in plaats daarvan [Ollama](/nl/providers/ollama) wanneer je uitsluitend lokale routering of
routering via cloud en lokaal wilt gebruiken via een aangemelde Ollama-host. Gebruik een
OpenAI-compatibele provider wanneer je `/v1/chat/completions`-
semantiek of providerspecifieke functies in OpenAI-stijl nodig hebt.

## Modellen

De provider vereist een API-sleutel; zonder sleutel blijft deze inactief. Met een sleutel
detecteert OpenClaw Ollama Cloud-modellen live vanuit de gehoste catalogus:

```bash
openclaw models list --provider ollama-cloud
openclaw models set ollama-cloud/kimi-k2.6
```

Gehoste id's in de live catalogus zijn onder andere `deepseek-v4-flash`, `glm-5`,
`gpt-oss:20b`, `kimi-k2.6` en `minimax-m2.7`. Wanneer live detectie
niets oplevert, valt OpenClaw terug op de meegeleverde vermeldingen `kimi-k2.5:cloud`,
`minimax-m2.7:cloud`, `glm-5.1:cloud` en `glm-5.2:cloud`.

Model-id's zijn id's uit de cloudcatalogus, geen namen voor lokale downloads. Als een modelnaam werkt op
een lokale Ollama-host maar ontbreekt in de gehoste catalogus, gebruik dan in plaats daarvan de provider `ollama`
met die lokale host.

## Livetest

Richt voor rooktests met een Ollama Cloud-API-sleutel de Ollama-livetest op het gehoste
eindpunt en kies een model uit je huidige catalogus:

```bash
export OLLAMA_API_KEY="<your-ollama-cloud-api-key>" # pragma: allowlist secret

OPENCLAW_LIVE_TEST=1 \
OPENCLAW_LIVE_OLLAMA=1 \
OPENCLAW_LIVE_OLLAMA_BASE_URL=https://ollama.com \
OPENCLAW_LIVE_OLLAMA_MODEL=kimi-k2.6 \
pnpm test:live -- extensions/ollama/ollama.live.test.ts
```

De cloud-rooktest voert tekst, native streaming en zoeken op het web uit; stel
`OPENCLAW_LIVE_OLLAMA_WEB_SEARCH=0` in om zoeken op het web over te slaan. Embeddings worden
standaard overgeslagen voor `https://ollama.com`, omdat Ollama Cloud-API-sleutels
mogelijk geen autorisatie geven voor `/api/embed`; forceer ze met `OPENCLAW_LIVE_OLLAMA_EMBEDDINGS=1`.

## Probleemoplossing

- `Ollama Cloud requires an API key`- / `Set OLLAMA_API_KEY`-fouten: geef een
  echte cloud-API-sleutel op. De lokale markering `ollama-local` is alleen bedoeld voor lokale of
  privé-Ollama-hosts.
- Fouten over onbekende modellen: voer `openclaw models list --provider ollama-cloud` uit en
  kopieer de id van het gehoste model exact.
- Problemen met toolaanroepen of onbewerkte JSON op aangepaste Ollama-hosts: controleer of je
  per ongeluk een OpenAI-compatibele `/v1`-URL gebruikt. Ollama-routes moeten
  de native basis-URL zonder het achtervoegsel `/v1` gebruiken.

## Gerelateerd

- [Ollama](/nl/providers/ollama)
- [Modelproviders](/nl/concepts/model-providers)
- [Alle providers](/nl/providers/index)
