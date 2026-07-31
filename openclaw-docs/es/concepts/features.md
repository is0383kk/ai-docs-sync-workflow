---
read_when:
    - Se desea una lista completa de lo que admite OpenClaw
summary: Capacidades de OpenClaw en canales, enrutamiento, contenido multimedia y experiencia de usuario.
title: Características
x-i18n:
    generated_at: "2026-07-26T04:35:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5bc3ebdd87a0f6ea0f3d75d029bf7cae469ecd9db84a165bd47c4896936fe303
    source_path: concepts/features.md
    workflow: 16
---

## Aspectos destacados

<Columns>
  <Card title="Canales" icon="message-square" href="/es/channels">
    Discord, iMessage, Signal, Slack, Telegram, WhatsApp, WebChat y más con un único Gateway.
  </Card>
  <Card title="Plugins" icon="plug" href="/es/tools/plugin">
    Los plugins oficiales añaden Matrix, Nextcloud Talk, Nostr, Twitch, Zalo y decenas más con un solo comando de instalación.
  </Card>
  <Card title="Enrutamiento" icon="route" href="/es/concepts/multi-agent">
    Enrutamiento multiagente con sesiones aisladas.
  </Card>
  <Card title="Contenido multimedia" icon="image" href="/es/nodes/images">
    Imágenes, audio, vídeo, documentos y generación de imágenes y vídeos.
  </Card>
  <Card title="Aplicaciones e interfaz de usuario" icon="monitor" href="/es/platforms">
    Windows Hub, la interfaz de control en el navegador, la aplicación para la barra de menús de macOS y nodos móviles.
  </Card>
  <Card title="Nodos móviles" icon="smartphone" href="/es/nodes">
    Nodos de iOS y Android con vinculación, voz/chat y comandos avanzados para el dispositivo.
  </Card>
</Columns>

## Lista completa

**Canales:**

- iMessage, Telegram y WebChat se incluyen con la instalación principal; todos los demás canales son
  plugins oficiales instalados con `openclaw plugins install @openclaw/<id>` (o bajo demanda
  durante `openclaw onboard` / `openclaw channels add`)
- Canales de plugins oficiales: Discord, Feishu, Google Chat, IRC, LINE, Matrix, Mattermost,
  Microsoft Teams, Nextcloud Talk, Nostr, QQ Bot, Raft, Signal, Slack, SMS, Synology Chat,
  Tlon, Twitch, Voice Call, WhatsApp, Zalo y Zalo Personal
- Canales de plugins externos mantenidos fuera del repositorio de OpenClaw: WeChat, Yuanbao y Zalo ClawBot
- Compatibilidad con chats grupales mediante activación basada en menciones
- Seguridad de los mensajes directos mediante listas de permitidos y vinculación

**Agente:**

- Entorno de ejecución de agentes integrado con transmisión de herramientas
- Enrutamiento multiagente con sesiones aisladas por espacio de trabajo o remitente
- Sesiones: los chats directos se agrupan en `main`; los grupos permanecen aislados
- Transmisión y división en fragmentos para respuestas largas

**Autenticación y proveedores:**

- Más de 35 proveedores de modelos (Anthropic, OpenAI, Google y más)
- Autenticación de suscripción mediante OAuth (por ejemplo, OpenAI Codex)
- Compatibilidad con proveedores personalizados y autoalojados (vLLM, SGLang, Ollama, llama.cpp, LM Studio y
  cualquier endpoint compatible con OpenAI o Anthropic)

**Contenido multimedia:**

- Entrada y salida de imágenes, audio, vídeo y documentos
- Superficies de capacidades compartidas para generar imágenes y vídeos
- Transcripción de notas de voz
- Conversión de texto a voz con varios proveedores

**Aplicaciones e interfaces:**

- WebChat e interfaz de control en el navegador
- Aplicación complementaria para la barra de menús de macOS
- Nodo de iOS con vinculación, Canvas, cámara, grabación de pantalla, ubicación y voz
- Nodo de Android con vinculación, chat, voz, Canvas, cámara y comandos del dispositivo

**Herramientas y automatización:**

- Automatización del navegador, ejecución y aislamiento
- Búsqueda web (Brave, DuckDuckGo, Exa, Firecrawl, Gemini, Grok, Kimi, MiniMax Search, Ollama Web Search, Perplexity, SearXNG, Tavily)
- Tareas de Cron y programación de Heartbeat
- Skills, plugins y pipelines de flujos de trabajo (Lobster)

## Contenido relacionado

<CardGroup cols={2}>
  <Card title="Funciones experimentales" href="/es/concepts/experimental-features" icon="flask">
    Funciones opcionales que aún no se han incorporado a la superficie predeterminada.
  </Card>
  <Card title="Entorno de ejecución del agente" href="/es/concepts/agent" icon="robot">
    Modelo del entorno de ejecución del agente y cómo se despachan las ejecuciones.
  </Card>
  <Card title="Canales" href="/es/channels" icon="message-square">
    Conecte Telegram, WhatsApp, Discord, Slack y más desde un único Gateway.
  </Card>
  <Card title="Plugins" href="/es/tools/plugin" icon="plug">
    Plugins oficiales y externos que amplían OpenClaw.
  </Card>
</CardGroup>
