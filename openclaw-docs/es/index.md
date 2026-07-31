---
read_when:
    - Presentación de OpenClaw a nuevos usuarios
summary: OpenClaw es un gateway multicanal para agentes de IA que se ejecuta en cualquier sistema operativo.
title: OpenClaw
x-i18n:
    generated_at: "2026-07-26T05:43:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0ce948d12d4b4fcbde2597f9b33f50b99c4f677b69e0f5d72677b2f6683291f3
    source_path: index.md
    workflow: 16
---

# OpenClaw 🦞

<p align="center">
    <img
        src="/assets/openclaw-hero-light.png"
        alt="OpenClaw"
        width="500"
        class="dark:hidden"
    />
    <img
        src="/assets/openclaw-hero-dark.png"
        alt="OpenClaw"
        width="500"
        class="hidden dark:block"
    />
</p>

> _«¡EXFOLIA! ¡EXFOLIA!»_ — Probablemente, una langosta espacial

<p align="center">
  <strong>Gateway para cualquier sistema operativo que conecta agentes de IA mediante Discord, Google Chat, iMessage, Matrix, Microsoft Teams, Signal, Slack, Telegram, WhatsApp, Zalo y más.</strong><br />
  Envía un mensaje y recibe en tu bolsillo la respuesta de un agente. Ejecuta un único Gateway para plugins de canales, WebChat y nodos móviles.<br />
  Desarrollado de forma abierta por la <a href="https://openclaw.org">Fundación OpenClaw</a>, una organización sin fines de lucro.
</p>

<Columns>
  <Card title="Primeros pasos" href="/es/start/getting-started" icon="rocket">
    Instala OpenClaw e inicia el Gateway en cuestión de minutos.
  </Card>
  <Card title="Ejecutar la incorporación" href="/es/start/wizard" icon="list-checks">
    Configuración guiada con `openclaw onboard` y flujos de emparejamiento.
  </Card>
  <Card title="Conectar un canal" href="/es/channels" icon="message-circle">
    Vincula Discord, Signal, Telegram, WhatsApp y más para chatear desde cualquier lugar.
  </Card>
  <Card title="Abrir la interfaz de control" href="/es/web/control-ui" icon="layout-dashboard">
    Abre el panel de control del navegador para el chat, la configuración y las sesiones.
  </Card>
</Columns>

## Explorar la documentación

Es posible que los navegadores móviles muestren el menú de secciones sin la barra de pestañas completa de la versión de escritorio. Utiliza
estos enlaces a secciones centrales para acceder desde el cuerpo de la página a las mismas áreas principales de la documentación.

<Columns>
  <Card title="Primeros pasos" href="/es" icon="rocket">
    Descripción general, presentación, primeros pasos y guías de configuración.
  </Card>
  <Card title="Instalación" href="/es/install" icon="download">
    Métodos de instalación, actualizaciones, contenedores, alojamiento y configuración avanzada.
  </Card>
  <Card title="Canales" href="/es/channels" icon="messages-square">
    Canales de mensajería, emparejamiento, enrutamiento, grupos de acceso y control de calidad de canales.
  </Card>
  <Card title="Agentes" href="/es/concepts/architecture" icon="bot">
    Arquitectura, sesiones, contexto, memoria y enrutamiento multiagente.
  </Card>
  <Card title="Capacidades" href="/es/tools" icon="wand-sparkles">
    Herramientas, habilidades, cron, webhooks y capacidades de automatización.
  </Card>
  <Card title="ClawHub" href="/es/clawhub" icon="store">
    Mercado de plugins, publicación, selección y orientación sobre confianza.
  </Card>
  <Card title="Modelos" href="/es/providers" icon="brain">
    Proveedores, configuración de modelos, conmutación por error y servicios de modelos locales.
  </Card>
  <Card title="Plataformas" href="/es/platforms" icon="monitor-smartphone">
    macOS, Windows, iOS, Android, nodos e interfaces web.
  </Card>
  <Card title="Gateway y operaciones" href="/es/gateway" icon="server">
    Configuración, seguridad, diagnóstico y operaciones del Gateway.
  </Card>
  <Card title="Referencia" href="/es/cli" icon="terminal">
    Referencia de la CLI, esquemas, RPC, notas de la versión y plantillas.
  </Card>
  <Card title="Ayuda" href="/es/help" icon="life-buoy">
    Solución de problemas, preguntas frecuentes, pruebas, diagnóstico y comprobaciones del entorno.
  </Card>
</Columns>

## ¿Qué es OpenClaw?

OpenClaw es un **Gateway autoalojado** que conecta tus aplicaciones de chat favoritas —Discord, Google Chat, iMessage, Matrix, Microsoft Teams, Signal, Slack, Telegram, WhatsApp, Zalo y más mediante plugins de canales— con agentes de IA para programación. Ejecutas un único proceso de Gateway en tu propio equipo (o en un servidor), que se convierte en el puente entre tus aplicaciones de mensajería y un asistente de IA siempre disponible.

**¿A quién va dirigido?** A desarrolladores y usuarios avanzados que desean un asistente de IA personal al que puedan enviar mensajes desde cualquier lugar, sin renunciar al control de sus datos ni depender de un servicio alojado.

**¿Qué lo diferencia?**

- **Autoalojado**: se ejecuta en tu hardware y conforme a tus reglas
- **Multicanal**: un solo Gateway atiende simultáneamente a todos los plugins de canales configurados
- **Nativo para agentes**: diseñado para agentes de programación con uso de herramientas, sesiones, memoria y enrutamiento multiagente
- **Código abierto**: con licencia MIT e impulsado por la comunidad

**¿Qué necesitas?** Node 24.15+ (recomendado), Node 22 LTS (`22.22.3+`) para garantizar la compatibilidad o Node 25.9+, una clave de API del proveedor que elijas y 5 minutos. Para obtener la máxima calidad y seguridad, utiliza el modelo de última generación más potente disponible.

## Cómo funciona

```mermaid
flowchart LR
  A["Aplicaciones de chat + plugins"] --> B["Gateway"]
  B --> C["Agente de OpenClaw"]
  B --> D["CLI"]
  B --> E["Interfaz de control web"]
  B --> F["Aplicación para macOS"]
  B --> G["Nodos iOS y Android"]
```

El Gateway es la única fuente de información fiable para las sesiones, el enrutamiento y las conexiones de canales.

## Capacidades principales

<Columns>
  <Card title="Gateway multicanal" icon="network" href="/es/channels">
    Discord, iMessage, Signal, Slack, Telegram, WhatsApp, WebChat y más con un único proceso de Gateway.
  </Card>
  <Card title="Canales mediante plugins" icon="plug" href="/es/tools/plugin">
    Los plugins de canales incorporan Matrix, Nostr, Twitch, Zalo y más; los plugins oficiales se instalan bajo demanda.
  </Card>
  <Card title="Enrutamiento multiagente" icon="route" href="/es/concepts/multi-agent">
    Sesiones aisladas por agente, espacio de trabajo o remitente.
  </Card>
  <Card title="Compatibilidad multimedia" icon="image" href="/es/nodes/images">
    Envía y recibe imágenes, audio y documentos.
  </Card>
  <Card title="Interfaz de control web" icon="monitor" href="/es/web/control-ui">
    Panel de control del navegador para el chat, la configuración, las sesiones y los nodos.
  </Card>
  <Card title="Nodos móviles" icon="smartphone" href="/es/nodes">
    Empareja nodos iOS y Android para utilizar Canvas, la cámara y flujos de trabajo habilitados para voz.
  </Card>
</Columns>

## Inicio rápido

<Steps>
  <Step title="Instalar OpenClaw">
    ```bash
    npm install -g openclaw@latest
    ```
  </Step>
  <Step title="Completar la incorporación e instalar el servicio">
    ```bash
    openclaw onboard --install-daemon
    ```
  </Step>
  <Step title="Chatear">
    Abre la interfaz de control en el navegador y envía un mensaje:

    ```bash
    openclaw dashboard
    ```

    También puedes conectar un canal ([Telegram](/es/channels/telegram) es el más rápido) y chatear desde el teléfono.

  </Step>
</Steps>

¿Necesitas la configuración completa de instalación y desarrollo? Consulta [Primeros pasos](/es/start/getting-started).

## Panel de control

Abre la interfaz de control en el navegador después de que se inicie el Gateway.

- Valor local predeterminado: [http://127.0.0.1:18789/](http://127.0.0.1:18789/)
- Acceso remoto: [Interfaces web](/es/web) y [Tailscale](/es/gateway/tailscale)

<p align="center">
  <img src="/whatsapp-openclaw.jpg" alt="OpenClaw" width="420" />
</p>

## Configuración (opcional)

La configuración se encuentra en `~/.openclaw/openclaw.json`.

- Si **no haces nada**, OpenClaw utiliza el entorno de ejecución de agente OpenClaw incluido; los mensajes directos comparten la sesión principal del agente y cada chat grupal tiene su propia sesión.
- Si deseas restringir el acceso, comienza con `channels.whatsapp.allowFrom` y, para los grupos, con las reglas de menciones.

Ejemplo:

```json5
{
  channels: {
    whatsapp: {
      allowFrom: ["+15555550123"],
      groups: { "*": { requireMention: true } },
    },
  },
  messages: { groupChat: { mentionPatterns: ["@openclaw"] } },
}
```

## Empieza aquí

<Columns>
  <Card title="Secciones de documentación" href="/es/start/hubs" icon="book-open">
    Toda la documentación y las guías, organizadas por caso de uso.
  </Card>
  <Card title="Configuración" href="/es/gateway/configuration" icon="settings">
    Ajustes principales del Gateway, tokens y configuración del proveedor.
  </Card>
  <Card title="Acceso remoto" href="/es/gateway/remote" icon="globe">
    Patrones de acceso mediante SSH y tailnet.
  </Card>
  <Card title="Canales" href="/es/channels/telegram" icon="message-square">
    Configuración específica de canales para Discord, Feishu, Microsoft Teams, Telegram, WhatsApp y más.
  </Card>
  <Card title="Nodos" href="/es/nodes" icon="smartphone">
    Nodos iOS y Android con emparejamiento, Canvas, cámara y acciones del dispositivo.
  </Card>
  <Card title="Ayuda" href="/es/help" icon="life-buoy">
    Soluciones habituales y punto de partida para resolver problemas.
  </Card>
</Columns>

## Más información

<Columns>
  <Card title="Lista completa de funciones" href="/es/concepts/features" icon="list">
    Capacidades completas de canales, enrutamiento y contenido multimedia.
  </Card>
  <Card title="Enrutamiento multiagente" href="/es/concepts/multi-agent" icon="route">
    Aislamiento de espacios de trabajo y sesiones por agente.
  </Card>
  <Card title="Seguridad" href="/es/gateway/security" icon="shield">
    Tokens, listas de permitidos y controles de seguridad.
  </Card>
  <Card title="Solución de problemas" href="/es/gateway/troubleshooting" icon="wrench">
    Diagnóstico del Gateway y errores habituales.
  </Card>
  <Card title="Información y créditos" href="/es/reference/credits" icon="info">
    Origen del proyecto, colaboradores y licencia.
  </Card>
</Columns>
