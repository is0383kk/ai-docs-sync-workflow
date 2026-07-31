---
read_when:
    - Je bepaalt of een plugin in het npm-kernpakket wordt meegeleverd of afzonderlijk wordt geïnstalleerd
    - Je werkt de pakketmetadata of releaseautomatisering van gebundelde plugins bij
    - Je hebt de canonieke lijst met interne versus externe plugins nodig
summary: Gegenereerde inventaris van OpenClaw-plugins die in de kern worden meegeleverd, extern worden gepubliceerd of alleen als broncode worden behouden
title: Plugininventaris
x-i18n:
    generated_at: "2026-07-27T05:08:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2d835087afbe9d75f883c3db9739f914bedab5ac87a9c20b69c248304b61c594
    source_path: plugins/plugin-inventory.md
    workflow: 16
---

# Plugininventaris

Deze pagina wordt gegenereerd op basis van `extensions/*/package.json`, `openclaw.plugin.json`
en de uitsluitingen van het hoofd-npm-pakket `files`. Genereer de pagina opnieuw met:

```bash
pnpm plugins:inventory:gen
```

## Definities

- **Kern-npm-pakket:** ingebouwd in het npm-pakket `openclaw` en beschikbaar zonder afzonderlijke installatie van een plugin.
- **Officieel extern pakket:** door OpenClaw onderhouden plugin die niet in het kern-npm-pakket is opgenomen, in deze officiële inventaris wordt bijgehouden en op aanvraag via ClawHub en/of npm wordt geïnstalleerd.
- **Alleen broncheckout:** lokale plugin uit de repository die niet in gepubliceerde npm-artefacten is opgenomen en niet als installeerbaar pakket wordt aangeboden.

Broncheckouts verschillen van npm-installaties: na `pnpm install` worden gebundelde
plugins geladen vanuit `extensions/<id>`, zodat lokale wijzigingen en pakketlokale
workspace-afhankelijkheden beschikbaar zijn.

## Een plugin installeren

Gebruik de installatieroute in elke vermelding om te bepalen of installatie nodig is. Plugins
waarbij `included in OpenClaw` staat, zijn al aanwezig in het kernpakket.
Officiële externe pakketten vereisen één installatie en daarna een herstart van de Gateway.

Discord is bijvoorbeeld een officieel extern pakket:

```bash
openclaw plugins install @openclaw/discord
openclaw gateway restart
openclaw plugins inspect discord --runtime --json
```

Tijdens de overgang bij de lancering worden gewone kale pakketspecificaties nog steeds vanuit npm geïnstalleerd.
Gebruik `clawhub:@openclaw/discord` of `npm:@openclaw/discord` wanneer je een
expliciete bron nodig hebt. Volg na de installatie de configuratiedocumentatie van de plugin, zoals
[Discord](/nl/channels/discord), om referenties en kanaalconfiguratie toe te voegen. Zie
[Plugins beheren](/nl/plugins/manage-plugins) voor opdrachten voor bijwerken, verwijderen en publiceren.

Elke vermelding bevat het pakket, de distributieroute en een beschrijving.

## Kern-npm-pakket

70 plugins

- **[admin-http-rpc](/nl/plugins/reference/admin-http-rpc)** (`@openclaw/admin-http-rpc`) - opgenomen in OpenClaw. HTTP RPC-eindpunt voor OpenClaw-beheer.

- **[alibaba](/nl/plugins/reference/alibaba)** (`@openclaw/alibaba-provider`) - opgenomen in OpenClaw. Voegt ondersteuning voor een aanbieder van videogeneratie toe.

- **[anthropic](/nl/plugins/reference/anthropic)** (`@openclaw/anthropic-provider`) - opgenomen in OpenClaw. Anthropic-modellen, Claude CLI en een systeemeigen catalogus met Claude-sessies.

- **[azure-speech](/nl/plugins/reference/azure-speech)** (`@openclaw/azure-speech`) - opgenomen in OpenClaw. Tekst-naar-spraak met Azure AI Speech (MP3, systeemeigen Ogg/Opus-spraakberichten, PCM-telefonie).

- **[bonjour](/nl/plugins/reference/bonjour)** (`@openclaw/bonjour`) - opgenomen in OpenClaw. Kondigt de lokale OpenClaw-Gateway aan via Bonjour/mDNS.

- **[browser](/nl/plugins/reference/browser)** (`@openclaw/browser-plugin`) - opgenomen in OpenClaw. Voegt door agents aanroepbare hulpmiddelen toe.

- **[byteplus](/nl/plugins/reference/byteplus)** (`@openclaw/byteplus-provider`) - opgenomen in OpenClaw. Voegt ondersteuning voor de modelaanbieders BytePlus en BytePlus Plan toe aan OpenClaw.

- **[canvas](/nl/plugins/reference/canvas)** (`@openclaw/canvas-plugin`) - opgenomen in OpenClaw. Experimentele Canvas-besturing en A2UI-renderingoppervlakken voor gekoppelde nodes.

- **[clawrouter](/nl/plugins/reference/clawrouter)** (`@openclaw/clawrouter`) - opgenomen in OpenClaw. Voegt ondersteuning voor de modelaanbieder ClawRouter toe aan OpenClaw.

- **[cohere](/nl/plugins/reference/cohere)** (`@openclaw/cohere-provider`) - opgenomen in OpenClaw; npm; ClawHub: `clawhub:@openclaw/cohere-provider`. OpenClaw-plugin voor de Cohere-aanbieder.

- **[comfy](/nl/plugins/reference/comfy)** (`@openclaw/comfy-provider`) - opgenomen in OpenClaw. Voegt ondersteuning voor de modelaanbieder ComfyUI toe aan OpenClaw.

- **[copilot-proxy](/nl/plugins/reference/copilot-proxy)** (`@openclaw/copilot-proxy`) - opgenomen in OpenClaw. Voegt ondersteuning voor de modelaanbieder Copilot Proxy toe aan OpenClaw.

- **[crabbox](/nl/plugins/reference/crabbox)** (`@openclaw/crabbox-provider`) - opgenomen in OpenClaw. Aanbieder van cloudworkers, ondersteund door de Crabbox CLI.

- **[cua-computer](/nl/plugins/reference/cua-computer)** (`@openclaw/cua-computer`) - opgenomen in OpenClaw. Experimentele computerbesturing via cua-driver voor Windows- en Linux-nodehosts.

- **[deepgram](/nl/plugins/reference/deepgram)** (`@openclaw/deepgram-provider`) - opgenomen in OpenClaw. Voegt ondersteuning voor een aanbieder van mediabegrip toe. Voegt ondersteuning voor een aanbieder van realtime transcriptie toe.

- **[document-extract](/nl/plugins/reference/document-extract)** (`@openclaw/document-extract-plugin`) - opgenomen in OpenClaw. Extraheert tekst en reservepagina-afbeeldingen uit lokale documentbijlagen.

- **[duckduckgo](/nl/plugins/reference/duckduckgo)** (`@openclaw/duckduckgo-plugin`) - opgenomen in OpenClaw. Voegt ondersteuning voor een webzoekaanbieder toe.

- **[elevenlabs](/nl/plugins/reference/elevenlabs)** (`@openclaw/elevenlabs-speech`) - opgenomen in OpenClaw. Voegt ondersteuning voor een aanbieder van mediabegrip toe. Voegt ondersteuning voor een aanbieder van realtime transcriptie toe. Voegt ondersteuning voor een tekst-naar-spraakaanbieder toe.

- **[fal](/nl/plugins/reference/fal)** (`@openclaw/fal-provider`) - opgenomen in OpenClaw. Voegt ondersteuning voor de modelaanbieder fal toe aan OpenClaw.

- **[file-transfer](/nl/plugins/reference/file-transfer)** (`@openclaw/file-transfer`) - opgenomen in OpenClaw. Haalt bestanden op, geeft ze weer en schrijft ze op gekoppelde nodes via speciale node-opdrachten. Omzeilt afkapping van bash-stdout door base64 via node.invoke te gebruiken voor binaire bestanden tot 16 MB.

- **[github-copilot](/nl/plugins/reference/github-copilot)** (`@openclaw/github-copilot-provider`) - opgenomen in OpenClaw. Voegt ondersteuning voor de modelaanbieder GitHub Copilot toe aan OpenClaw.

- **[google](/nl/plugins/reference/google)** (`@openclaw/google-plugin`) - opgenomen in OpenClaw. Voegt ondersteuning voor de modelaanbieders Google, Google Gemini CLI en Google Vertex toe aan OpenClaw.

- **[huggingface](/nl/plugins/reference/huggingface)** (`@openclaw/huggingface-provider`) - opgenomen in OpenClaw. Voegt ondersteuning voor de modelaanbieder Hugging Face toe aan OpenClaw.

- **[imessage](/nl/plugins/reference/imessage)** (`@openclaw/imessage`) - opgenomen in OpenClaw. Voegt het iMessage-kanaaloppervlak toe voor het verzenden en ontvangen van OpenClaw-berichten.

- **[linux-canvas](/nl/plugins/reference/linux-canvas)** (`@openclaw/linux-canvas`) - opgenomen in OpenClaw. Canvas-renderingbrug voor de OpenClaw-desktopapp voor Linux.

- **[linux-node](/nl/plugins/reference/linux-node)** (`@openclaw/linux-node`) - opgenomen in OpenClaw. Bureaubladmeldingen, camera-opname en locatie voor Linux-nodehosts.

- **[litellm](/nl/plugins/reference/litellm)** (`@openclaw/litellm-provider`) - opgenomen in OpenClaw. Voegt ondersteuning voor de modelaanbieder LiteLLM toe aan OpenClaw.

- **[llm-task](/nl/plugins/reference/llm-task)** (`@openclaw/llm-task`) - opgenomen in OpenClaw. Algemene LLM-tool die uitsluitend JSON gebruikt voor gestructureerde taken die vanuit workflows kunnen worden aangeroepen.

- **[lmstudio](/nl/plugins/reference/lmstudio)** (`@openclaw/lmstudio-provider`) - opgenomen in OpenClaw. Voegt ondersteuning voor de modelaanbieder LM Studio toe aan OpenClaw.

- **[logbook](/nl/plugins/reference/logbook)** (`@openclaw/logbook`) - opgenomen in OpenClaw. Automatisch werkdagboek: maakt periodieke schermafbeeldingen van een gekoppelde node en zet die om in een controleerbare tijdlijn van je dag.

- **[memory-core](/nl/plugins/reference/memory-core)** (`@openclaw/memory-core`) - opgenomen in OpenClaw. Voegt door agents aanroepbare hulpmiddelen toe.

- **[memory-wiki](/nl/plugins/reference/memory-wiki)** (`@openclaw/memory-wiki`) - opgenomen in OpenClaw. Permanente wikicompiler en Obsidian-vriendelijke kennisopslag voor OpenClaw.

- **[meta](/nl/plugins/reference/meta)** (`@openclaw/meta-provider`) - opgenomen in OpenClaw; npm; ClawHub: `clawhub:@openclaw/meta-provider`. Voegt ondersteuning voor de modelaanbieder Meta toe aan OpenClaw.

- **[microsoft](/nl/plugins/reference/microsoft)** (`@openclaw/microsoft-speech`) - opgenomen in OpenClaw. Voegt ondersteuning voor een tekst-naar-spraakaanbieder toe.

- **[microsoft-foundry](/nl/plugins/reference/microsoft-foundry)** (`@openclaw/microsoft-foundry`) - opgenomen in OpenClaw. Voegt ondersteuning voor de modelaanbieder Microsoft Foundry toe aan OpenClaw.

- **[migrate-claude](/nl/plugins/reference/migrate-claude)** (`@openclaw/migrate-claude`) - opgenomen in OpenClaw. Importeert instructies, MCP-servers, skills en veilige configuratie van Claude Code en Claude Desktop in OpenClaw.

- **[migrate-hermes](/nl/plugins/reference/migrate-hermes)** (`@openclaw/migrate-hermes`) - opgenomen in OpenClaw. Importeert configuratie, herinneringen, skills en ondersteunde referenties van Hermes in OpenClaw.

- **[minimax](/nl/plugins/reference/minimax)** (`@openclaw/minimax-provider`) - opgenomen in OpenClaw. Voegt ondersteuning voor de modelaanbieders MiniMax en MiniMax Portal toe aan OpenClaw.

- **[mistral](/nl/plugins/reference/mistral)** (`@openclaw/mistral-provider`) - opgenomen in OpenClaw. Voegt ondersteuning voor de modelaanbieder Mistral toe aan OpenClaw.

- **[novita](/nl/plugins/reference/novita)** (`@openclaw/novita-provider`) - opgenomen in OpenClaw. Voegt ondersteuning voor de modelaanbieders Novita, Novita AI en Novitaai toe aan OpenClaw.

- **[nvidia](/nl/plugins/reference/nvidia)** (`@openclaw/nvidia-provider`) - opgenomen in OpenClaw. Voegt ondersteuning voor de modelaanbieder NVIDIA toe aan OpenClaw.

- **[oc-path](/nl/plugins/reference/oc-path)** (`@openclaw/oc-path`) - opgenomen in OpenClaw. Voegt de openclaw path CLI toe voor het adresseren van workspace-bestanden via oc://.

- **[ollama](/nl/plugins/reference/ollama)** (`@openclaw/ollama-provider`) - opgenomen in OpenClaw. Voegt ondersteuning voor de modelaanbieders Ollama en Ollama Cloud toe aan OpenClaw.

- **[onepassword](/nl/plugins/reference/onepassword)** (`@openclaw/onepassword`) - opgenomen in OpenClaw. Samengestelde broker voor 1Password-geheimen met goedkeuringsbeleid en SQLite-auditgeschiedenis.

- **[open-prose](/nl/plugins/reference/open-prose)** (`@openclaw/open-prose`) - opgenomen in OpenClaw. OpenProse VM-skillpakket met een /prose-slashopdracht.

- **[openai](/nl/plugins/reference/openai)** (`@openclaw/openai-provider`) - opgenomen in OpenClaw. Voegt ondersteuning voor de modelaanbieder OpenAI toe aan OpenClaw.

- **[opencode](/nl/plugins/reference/opencode)** (`@openclaw/opencode-provider`) - opgenomen in OpenClaw. Voegt ondersteuning voor de modelaanbieder OpenCode toe aan OpenClaw.

- **[opencode-go](/nl/plugins/reference/opencode-go)** (`@openclaw/opencode-go-provider`) - opgenomen in OpenClaw. Voegt ondersteuning voor de modelaanbieder OpenCode Go toe aan OpenClaw.

- **[openrouter](/nl/plugins/reference/openrouter)** (`@openclaw/openrouter-provider`) - opgenomen in OpenClaw. Voegt ondersteuning voor de modelaanbieder OpenRouter toe aan OpenClaw.

- **[policy](/nl/plugins/reference/policy)** (`@openclaw/policy`) - opgenomen in OpenClaw. Voegt door beleid ondersteunde doctor-controles toe voor workspace-conformiteit.

- **[reef](/nl/plugins/reference/reef)** (`@openclaw/reef`) - opgenomen in OpenClaw. Beveiligd end-to-end versleuteld claw-kanaal.

- **[runway](/nl/plugins/reference/runway)** (`@openclaw/runway-provider`) - opgenomen in OpenClaw. Voegt ondersteuning voor een aanbieder van videogeneratie toe.

- **[senseaudio](/nl/plugins/reference/senseaudio)** (`@openclaw/senseaudio-provider`) - opgenomen in OpenClaw. Voegt ondersteuning voor een aanbieder van mediabegrip toe.

- **[sglang](/nl/plugins/reference/sglang)** (`@openclaw/sglang-provider`) - opgenomen in OpenClaw. Voegt ondersteuning voor de modelaanbieder SGLang toe aan OpenClaw.

- **[synthetic](/nl/plugins/reference/synthetic)** (`@openclaw/synthetic-provider`) - opgenomen in OpenClaw. Voegt ondersteuning voor de modelaanbieder Synthetic toe aan OpenClaw.

- **[teams-meetings](/nl/plugins/reference/teams-meetings)** (`@openclaw/teams-meetings`) - opgenomen in OpenClaw. Neem als gast via de Chrome-browser deel aan Microsoft Teams-vergaderingen.

- **[telegram](/nl/plugins/reference/telegram)** (`@openclaw/telegram`) - opgenomen in OpenClaw. Voegt het Telegram-kanaaloppervlak toe voor het verzenden en ontvangen van OpenClaw-berichten.

- **[together](/nl/plugins/reference/together)** (`@openclaw/together-provider`) - opgenomen in OpenClaw. Voegt ondersteuning voor de modelaanbieder Together toe aan OpenClaw.

- **[tts-local-cli](/nl/plugins/reference/tts-local-cli)** (`@openclaw/tts-local-cli`) - opgenomen in OpenClaw. Voegt ondersteuning voor een tekst-naar-spraakaanbieder toe.

- **[vault](/nl/plugins/reference/vault)** (`@openclaw/vault`) - opgenomen in OpenClaw. Integratie met de HashiCorp Vault SecretRef-provider.

- **[vllm](/nl/plugins/reference/vllm)** (`@openclaw/vllm-provider`) - opgenomen in OpenClaw. Voegt ondersteuning voor de vLLM-modelprovider toe aan OpenClaw.

- **[volcengine](/nl/plugins/reference/volcengine)** (`@openclaw/volcengine-provider`) - opgenomen in OpenClaw. Voegt ondersteuning voor de modelproviders Volcengine en Volcengine Plan toe aan OpenClaw.

- **[voyage](/nl/plugins/reference/voyage)** (`@openclaw/voyage-provider`) - opgenomen in OpenClaw. Voegt ondersteuning voor een provider van geheugen-embeddings toe.

- **[vydra](/nl/plugins/reference/vydra)** (`@openclaw/vydra-provider`) - opgenomen in OpenClaw. Voegt ondersteuning voor de Vydra-modelprovider toe aan OpenClaw.

- **[web-readability](/nl/plugins/reference/web-readability)** (`@openclaw/web-readability-plugin`) - opgenomen in OpenClaw. Extraheert leesbare artikelinhoud uit lokale HTML-responsen van webophaalacties.

- **[webhooks](/nl/plugins/reference/webhooks)** (`@openclaw/webhooks`) - opgenomen in OpenClaw. Geverifieerde inkomende webhooks die externe automatisering aan OpenClaw TaskFlows koppelen.

- **[workboard](/nl/plugins/reference/workboard)** (`@openclaw/workboard`) - opgenomen in OpenClaw. Dashboardwerkbord voor issues en sessies die eigendom zijn van agents.

- **[xai](/nl/plugins/reference/xai)** (`@openclaw/xai-plugin`) - opgenomen in OpenClaw. Voegt ondersteuning voor de xAI-modelprovider toe aan OpenClaw.

- **[xiaomi](/nl/plugins/reference/xiaomi)** (`@openclaw/xiaomi-provider`) - opgenomen in OpenClaw. Voegt ondersteuning voor de modelproviders Xiaomi en Xiaomi Token Plan toe aan OpenClaw.

- **[zoom-meetings](/plugins/reference/zoom-meetings)** (`@openclaw/zoom-meetings`) - opgenomen in OpenClaw. Neem als gast via de Chrome-browser deel aan Zoom-vergaderingen.

## Officiële externe pakketten

72 plugins

- **[acpx](/nl/plugins/reference/acpx)** (`@openclaw/acpx`) - npm; ClawHub. OpenClaw ACP-runtimebackend met sessie- en transportbeheer dat eigendom is van de plugin.

- **[amazon-bedrock](/nl/plugins/reference/amazon-bedrock)** (`@openclaw/amazon-bedrock-provider`) - npm; ClawHub. OpenClaw-providerplugin voor Amazon Bedrock met modeldetectie en ondersteuning voor embeddings en beveiligingsregels.

- **[amazon-bedrock-mantle](/nl/plugins/reference/amazon-bedrock-mantle)** (`@openclaw/amazon-bedrock-mantle-provider`) - npm; ClawHub. OpenClaw-providerplugin voor Amazon Bedrock Mantle voor OpenAI-compatibele modelroutering.

- **[anthropic-vertex](/nl/plugins/reference/anthropic-vertex)** (`@openclaw/anthropic-vertex-provider`) - npm; ClawHub. OpenClaw-providerplugin voor Anthropic Vertex voor Claude-modellen op Google Vertex AI.

- **[arcee](/nl/plugins/reference/arcee)** (`@openclaw/arcee-provider`) - npm; ClawHub: `clawhub:@openclaw/arcee-provider`. Voegt ondersteuning voor de Arcee-modelprovider toe aan OpenClaw.

- **[baseten](/plugins/reference/baseten)** (`@openclaw/baseten-provider`) - npm; ClawHub: `clawhub:@openclaw/baseten-provider`. OpenClaw-providerplugin voor Baseten.

- **[brave](/nl/plugins/reference/brave)** (`@openclaw/brave-plugin`) - npm; ClawHub. OpenClaw-providerplugin voor Brave Search voor zoeken op het web.

- **[cerebras](/nl/plugins/reference/cerebras)** (`@openclaw/cerebras-provider`) - npm; ClawHub: `clawhub:@openclaw/cerebras-provider`. Voegt ondersteuning voor de Cerebras-modelprovider toe aan OpenClaw.

- **[chutes](/nl/plugins/reference/chutes)** (`@openclaw/chutes-provider`) - npm; ClawHub: `clawhub:@openclaw/chutes-provider`. Voegt ondersteuning voor de Chutes-modelprovider toe aan OpenClaw.

- **[clickclack](/nl/plugins/reference/clickclack)** (`@openclaw/clickclack`) - npm; ClawHub: `clawhub:@openclaw/clickclack`. Voegt het Clickclack-kanaaloppervlak toe voor het verzenden en ontvangen van OpenClaw-berichten.

- **[cloudflare-ai-gateway](/nl/plugins/reference/cloudflare-ai-gateway)** (`@openclaw/cloudflare-ai-gateway-provider`) - npm; ClawHub: `clawhub:@openclaw/cloudflare-ai-gateway-provider`. Voegt ondersteuning voor de Cloudflare AI Gateway-modelprovider toe aan OpenClaw.

- **[codex](/nl/plugins/reference/codex)** (`@openclaw/codex`) - npm; ClawHub. Harnas voor de Codex-appserver en systeemeigen sessiecatalogus.

- **[copilot](/nl/plugins/reference/copilot)** (`@openclaw/copilot`) - npm; ClawHub: `clawhub:@openclaw/copilot`. Registreert de GitHub Copilot-agentruntime.

- **[deepinfra](/nl/plugins/reference/deepinfra)** (`@openclaw/deepinfra-provider`) - npm; ClawHub: `clawhub:@openclaw/deepinfra-provider`. Voegt ondersteuning voor de DeepInfra-modelprovider toe aan OpenClaw.

- **[deepseek](/nl/plugins/reference/deepseek)** (`@openclaw/deepseek-provider`) - npm; ClawHub: `clawhub:@openclaw/deepseek-provider`. Voegt ondersteuning voor de DeepSeek-modelprovider toe aan OpenClaw.

- **[diagnostics-otel](/nl/plugins/reference/diagnostics-otel)** (`@openclaw/diagnostics-otel`) - npm; ClawHub: `clawhub:@openclaw/diagnostics-otel`. OpenClaw-diagnostiekexporter voor OpenTelemetry voor metrische gegevens, traces en logs.

- **[diagnostics-prometheus](/nl/plugins/reference/diagnostics-prometheus)** (`@openclaw/diagnostics-prometheus`) - npm; ClawHub: `clawhub:@openclaw/diagnostics-prometheus`. OpenClaw-diagnostiekexporter voor Prometheus voor metrische runtimegegevens.

- **[diffs](/nl/plugins/reference/diffs)** (`@openclaw/diffs`) - npm; ClawHub. Alleen-lezen diffviewerplugin en bestandsrenderer voor OpenClaw-agents.

- **[diffs-language-pack](/nl/plugins/reference/diffs-language-pack)** (`@openclaw/diffs-language-pack`) - npm; ClawHub: `clawhub:@openclaw/diffs-language-pack`. Voegt syntaxisaccentuering toe voor talen buiten de standaardset van de diffviewer.

- **[discord](/nl/plugins/reference/discord)** (`@openclaw/discord`) - npm; ClawHub. OpenClaw-kanaalplugin voor Discord voor kanalen, privéberichten, opdrachten en appgebeurtenissen.

- **[exa](/nl/plugins/reference/exa)** (`@openclaw/exa-plugin`) - npm; ClawHub: `clawhub:@openclaw/exa-plugin`. Voegt ondersteuning voor een provider voor zoeken op het web toe.

- **[featherless](/nl/plugins/reference/featherless)** (`@openclaw/featherless-provider`) - npm; ClawHub: `clawhub:@openclaw/featherless-provider`. OpenClaw-providerplugin voor Featherless AI.

- **[feishu](/nl/plugins/reference/feishu)** (`@openclaw/feishu`) - npm; ClawHub. OpenClaw-kanaalplugin voor Feishu/Lark voor chats en werkplektools (onderhouden door de community onder leiding van @m1heng).

- **[firecrawl](/nl/plugins/reference/firecrawl)** (`@openclaw/firecrawl-plugin`) - npm; ClawHub: `clawhub:@openclaw/firecrawl-plugin`. Voegt tools toe die agents kunnen aanroepen. Voegt ondersteuning voor een webophaalprovider toe. Voegt ondersteuning voor een provider voor zoeken op het web toe.

- **[fireworks](/nl/plugins/reference/fireworks)** (`@openclaw/fireworks-provider`) - npm; ClawHub: `clawhub:@openclaw/fireworks-provider`. Voegt ondersteuning voor de Fireworks-modelprovider toe aan OpenClaw.

- **[gmi](/nl/plugins/reference/gmi)** (`@openclaw/gmi-provider`) - npm; ClawHub: `clawhub:@openclaw/gmi-provider`. OpenClaw-providerplugin voor GMI Cloud.

- **[google-meet](/nl/plugins/reference/google-meet)** (`@openclaw/google-meet`) - npm; ClawHub. OpenClaw-deelnemersplugin voor Google Meet om via Chrome- of Twilio-transporten aan gesprekken deel te nemen.

- **[googlechat](/nl/plugins/reference/googlechat)** (`@openclaw/googlechat`) - npm; ClawHub. OpenClaw-kanaalplugin voor Google Chat voor ruimten en directe berichten.

- **[gradium](/nl/plugins/reference/gradium)** (`@openclaw/gradium-speech`) - npm; ClawHub: `clawhub:@openclaw/gradium-speech`. Voegt ondersteuning voor een tekst-naar-spraakprovider toe.

- **[groq](/nl/plugins/reference/groq)** (`@openclaw/groq-provider`) - npm; ClawHub: `clawhub:@openclaw/groq-provider`. Voegt ondersteuning voor de Groq-modelprovider toe aan OpenClaw.

- **[inworld](/nl/plugins/reference/inworld)** (`@openclaw/inworld-speech`) - npm; ClawHub: `clawhub:@openclaw/inworld-speech`. Inworld-streaming voor tekst-naar-spraak (MP3, OGG_OPUS, PCM-telefonie).

- **[irc](/nl/plugins/reference/irc)** (`@openclaw/irc`) - npm; ClawHub: `clawhub:@openclaw/irc`. Voegt het IRC-kanaaloppervlak toe voor het verzenden en ontvangen van OpenClaw-berichten.

- **[kilocode](/nl/plugins/reference/kilocode)** (`@openclaw/kilocode-provider`) - npm; ClawHub: `clawhub:@openclaw/kilocode-provider`. Voegt ondersteuning voor de Kilocode-modelprovider toe aan OpenClaw.

- **[kimi](/nl/plugins/reference/kimi)** (`@openclaw/kimi-provider`) - npm; ClawHub: `clawhub:@openclaw/kimi-provider`. Voegt ondersteuning voor de modelproviders Kimi en Kimi Coding toe aan OpenClaw.

- **[line](/nl/plugins/reference/line)** (`@openclaw/line`) - npm; ClawHub. OpenClaw-kanaalplugin voor LINE Bot API-chats.

- **[llama-cpp](/nl/plugins/reference/llama-cpp)** (`@openclaw/llama-cpp-provider`) - npm; ClawHub. Lokale GGUF-tekstinferentie en embeddings via node-llama-cpp.

- **[lobster](/nl/plugins/reference/lobster)** (`@openclaw/lobster`) - npm; ClawHub. Lobster-workflowtoolplugin voor getypeerde pijplijnen en hervatbare goedkeuringen.

- **[longcat](/nl/plugins/reference/longcat)** (`@openclaw/longcat-provider`) - npm; ClawHub: `clawhub:@openclaw/longcat-provider`. OpenClaw-providerplugin voor LongCat.

- **[matrix](/nl/plugins/reference/matrix)** (`@openclaw/matrix`) - ClawHub: `clawhub:@openclaw/matrix`; npm. OpenClaw-kanaalplugin voor Matrix voor ruimten en directe berichten.

- **[mattermost](/nl/plugins/reference/mattermost)** (`@openclaw/mattermost`) - npm; ClawHub: `clawhub:@openclaw/mattermost`. Voegt het Mattermost-kanaaloppervlak toe voor het verzenden en ontvangen van OpenClaw-berichten.

- **[memory-lancedb](/nl/plugins/reference/memory-lancedb)** (`@openclaw/memory-lancedb`) - npm; ClawHub. Door LanceDB ondersteunde OpenClaw-plugin voor langetermijngeheugen met automatisch ophalen, automatisch vastleggen en vectorzoekopdrachten.

- **[moonshot](/nl/plugins/reference/moonshot)** (`@openclaw/moonshot-provider`) - npm; ClawHub: `clawhub:@openclaw/moonshot-provider`. Voegt ondersteuning voor de Moonshot-modelprovider toe aan OpenClaw.

- **[msteams](/nl/plugins/reference/msteams)** (`@openclaw/msteams`) - npm; ClawHub. OpenClaw-kanaalplugin voor Microsoft Teams voor botgesprekken.

- **[mxc](/nl/plugins/reference/mxc)** (`@openclaw/mxc-sandbox`) - npm; ClawHub. Uitvoering van tools in een sandbox op besturingssysteemniveau via MXC: voert opdrachten uit in een Windows ProcessContainer met geconfigureerde MXC-beleidsbestanden.

- **[nextcloud-talk](/nl/plugins/reference/nextcloud-talk)** (`@openclaw/nextcloud-talk`) - npm; ClawHub. OpenClaw-kanaalplugin voor Nextcloud Talk voor gesprekken.

- **[nostr](/nl/plugins/reference/nostr)** (`@openclaw/nostr`) - npm; ClawHub. OpenClaw-kanaalplugin voor Nostr voor met NIP-04 versleutelde directe berichten.

- **[openshell](/nl/plugins/reference/openshell)** (`@openclaw/openshell-sandbox`) - npm; ClawHub. OpenClaw-sandboxbackend voor de NVIDIA OpenShell CLI met gespiegelde lokale werkruimten en uitvoering van SSH-opdrachten.

- **[parallel](/nl/tools/parallel-search)** (`@openclaw/parallel-plugin`) - npm; ClawHub: `clawhub:@openclaw/parallel-plugin`. Voegt ondersteuning voor een provider voor zoeken op het web toe.

- **[perplexity](/nl/plugins/reference/perplexity)** (`@openclaw/perplexity-plugin`) - npm; ClawHub: `clawhub:@openclaw/perplexity-plugin`. Voegt ondersteuning voor een provider voor zoeken op het web toe.

- **[pixverse](/nl/plugins/reference/pixverse)** (`@openclaw/pixverse-provider`) - npm; ClawHub: `clawhub:@openclaw/pixverse-provider`. OpenClaw-providerplugin voor PixVerse-videogeneratie.

- **[qianfan](/nl/plugins/reference/qianfan)** (`@openclaw/qianfan-provider`) - npm; ClawHub: `clawhub:@openclaw/qianfan-provider`. Voegt ondersteuning voor de Qianfan-modelprovider toe aan OpenClaw.

- **[qqbot](/nl/plugins/reference/qqbot)** (`@openclaw/qqbot`) - npm; ClawHub. OpenClaw-kanaalplugin voor QQ Bot voor groeps- en privéberichtworkflows.

- **[qwen](/nl/plugins/reference/qwen)** (`@openclaw/qwen-provider`) - npm; ClawHub: `clawhub:@openclaw/qwen-provider`. Voegt ondersteuning voor de modelproviders Qwen, Qwen Cloud, Model Studio, DashScope, Qwen Token Plan en Bailian Token Plan toe aan OpenClaw.

- **[raft](/nl/plugins/reference/raft)** (`@openclaw/raft`) - npm; ClawHub. OpenClaw-kanaalplugin voor Raft voor beveiligde CLI-wekbruggen.

- **[searxng](/nl/plugins/reference/searxng)** (`@openclaw/searxng-plugin`) - npm; ClawHub: `clawhub:@openclaw/searxng-plugin`. Voegt ondersteuning voor een provider voor zoeken op het web toe.

- **[signal](/nl/plugins/reference/signal)** (`@openclaw/signal`) - npm; ClawHub: `clawhub:@openclaw/signal`. Voegt het Signal-kanaaloppervlak toe voor het verzenden en ontvangen van OpenClaw-berichten.

- **[slack](/nl/plugins/reference/slack)** (`@openclaw/slack`) - npm; ClawHub. OpenClaw-kanaalplugin voor Slack voor kanalen, privéberichten, opdrachten en appgebeurtenissen.

- **[sms](/nl/plugins/reference/sms)** (`@openclaw/sms`) - npm; ClawHub: `clawhub:@openclaw/sms`. Twilio-sms-kanaalplugin voor OpenClaw-tekstberichten.

- **[stepfun](/nl/plugins/reference/stepfun)** (`@openclaw/stepfun-provider`) - npm; ClawHub: `clawhub:@openclaw/stepfun-provider`. Voegt ondersteuning voor de modelproviders StepFun en StepFun Plan toe aan OpenClaw.

- **[synology-chat](/nl/plugins/reference/synology-chat)** (`@openclaw/synology-chat`) - npm; ClawHub. Synology Chat-kanaalplugin voor OpenClaw-kanalen en directe berichten.

- **[tavily](/nl/plugins/reference/tavily)** (`@openclaw/tavily-plugin`) - npm; ClawHub: `clawhub:@openclaw/tavily-plugin`. Voegt tools toe die door agents kunnen worden aangeroepen. Voegt ondersteuning voor een webzoekprovider toe.

- **[tencent](/nl/plugins/reference/tencent)** (`@openclaw/tencent-provider`) - npm; ClawHub: `clawhub:@openclaw/tencent-provider`. Voegt ondersteuning voor de modelproviders Tencent TokenHub en Tencent Tokenplan toe aan OpenClaw.

- **[tlon](/nl/plugins/reference/tlon)** (`@openclaw/tlon`) - npm; ClawHub. OpenClaw-kanaalplugin voor Tlon/Urbit voor chatworkflows.

- **[tokenjuice](/nl/plugins/reference/tokenjuice)** (`@openclaw/tokenjuice`) - npm; ClawHub: `clawhub:@openclaw/tokenjuice`. Comprimeert resultaten van de exec- en bash-tools met Tokenjuice-reducers.

- **[twitch](/nl/plugins/reference/twitch)** (`@openclaw/twitch`) - npm; ClawHub. OpenClaw-kanaalplugin voor Twitch voor chat- en moderatieworkflows.

- **[venice](/nl/plugins/reference/venice)** (`@openclaw/venice-provider`) - npm; ClawHub: `clawhub:@openclaw/venice-provider`. Voegt ondersteuning voor de modelprovider Venice toe aan OpenClaw.

- **[vercel-ai-gateway](/nl/plugins/reference/vercel-ai-gateway)** (`@openclaw/vercel-ai-gateway-provider`) - npm; ClawHub: `clawhub:@openclaw/vercel-ai-gateway-provider`. Voegt ondersteuning voor de modelprovider Vercel AI Gateway toe aan OpenClaw.

- **[voice-call](/nl/plugins/reference/voice-call)** (`@openclaw/voice-call`) - npm; ClawHub. OpenClaw-plugin voor telefoongesprekken via Twilio, Telnyx en Plivo.

- **[whatsapp](/nl/plugins/reference/whatsapp)** (`@openclaw/whatsapp`) - ClawHub: `clawhub:@openclaw/whatsapp`; npm. OpenClaw-kanaalplugin voor WhatsApp Web-chats.

- **[zai](/nl/plugins/reference/zai)** (`@openclaw/zai-provider`) - npm; ClawHub: `clawhub:@openclaw/zai-provider`. Voegt ondersteuning voor de modelprovider Z.AI toe aan OpenClaw.

- **[zalo](/nl/plugins/reference/zalo)** (`@openclaw/zalo`) - npm; ClawHub. OpenClaw-kanaalplugin voor Zalo voor bot- en Webhook-chats.

- **[zalouser](/nl/plugins/reference/zalouser)** (`@openclaw/zalouser`) - npm; ClawHub. OpenClaw-plugin voor persoonlijke Zalo-accounts via native zca-js-integratie.

## Alleen broncode-checkout

2 plugins

- **[qa-channel](/nl/plugins/reference/qa-channel)** (`@openclaw/qa-channel`) - alleen broncode-checkout. Voegt de QA Channel-interface toe voor het verzenden en ontvangen van OpenClaw-berichten.

- **[qa-lab](/nl/plugins/reference/qa-lab)** (`@openclaw/qa-lab`) - alleen broncode-checkout. OpenClaw QA-labplugin met een privé-debuggerinterface en scenariorunner.
