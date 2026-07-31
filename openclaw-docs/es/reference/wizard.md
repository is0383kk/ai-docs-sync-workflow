---
read_when:
    - Buscar un paso o indicador específico de la incorporación
    - Automatización de la incorporación con el modo no interactivo
    - Depuración del comportamiento de incorporación
sidebarTitle: Onboarding Reference
summary: 'Referencia completa para la incorporación mediante la CLI: cada paso, indicador y campo de configuración'
title: Referencia de incorporación
x-i18n:
    generated_at: "2026-07-26T04:59:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e5e7e42fa3fc1a6d85ad422d0d28dfeda233c89a4d7e97eee4fb974831816372
    source_path: reference/wizard.md
    workflow: 16
---

Esta es la referencia completa de `openclaw onboard`.
Para obtener una descripción general, consulte [Incorporación (CLI)](/es/start/wizard). Para conocer el comportamiento y los resultados
paso a paso, consulte la [Referencia de configuración de la CLI](/es/start/wizard-cli-reference).

## Detalles del flujo (modo local)

<Steps>
  <Step title="Restablecimiento (opcional)">
    - `--reset` restablece el estado antes de ejecutar la configuración; sin esta opción, al volver a ejecutar la incorporación
      se conserva la configuración existente y se reutiliza como valores predeterminados.
    - `--reset-scope` controla lo que elimina `--reset`: `config` (solo el archivo de
      configuración), `config+creds+sessions` (valor predeterminado) o `full` (también elimina el
      espacio de trabajo).
    - Si el archivo de configuración no es válido, la incorporación se detiene e indica que primero se debe ejecutar
      `openclaw doctor` y, después, volver a ejecutar la configuración.
    - El restablecimiento mueve el estado a la papelera (nunca lo elimina directamente).

  </Step>
  <Step title="Aceptación del riesgo">
    - La primera ejecución (o cualquier ejecución anterior a que se establezca `wizard.securityAcknowledgedAt`)
      solicita confirmar que se comprende que los agentes son potentes y que el acceso
      total al sistema implica riesgos.
    - `--non-interactive` requiere `--accept-risk` explícitamente; sin esta opción,
      la incorporación finaliza con un error en lugar de solicitar confirmación.
    - Las ejecuciones interactivas muestran una solicitud de confirmación en lugar de la opción; si se rechaza,
      se cancela la configuración.

  </Step>
  <Step title="Modelo/autenticación">
    - **Clave de API de Anthropic**: utiliza `ANTHROPIC_API_KEY` si está presente o solicita una clave y, después, la guarda para que la use el daemon.
    - **CLI de Anthropic Claude**: ruta local preferida cuando ya existe un inicio de sesión de la CLI de Claude; OpenClaw también admite como alternativa la autenticación mediante token de configuración de Anthropic.
    - **Suscripción a OpenAI Code (Codex) (OAuth)**: flujo del navegador; pegue `code#state`.
      - En una configuración nueva sin modelo principal, establece `agents.defaults.model` en `openai/gpt-5.6-sol` mediante el entorno de ejecución de Codex.
    - **Suscripción a OpenAI Code (Codex) (vinculación de dispositivo)**: flujo de vinculación en el navegador con un código de dispositivo de corta duración.
      - En una configuración nueva sin modelo principal, establece `agents.defaults.model` en `openai/gpt-5.6-sol` mediante el entorno de ejecución de Codex.
    - **Clave de API de OpenAI**: utiliza `OPENAI_API_KEY` si está presente o solicita una clave y, después, la almacena en los perfiles de autenticación.
      - En una configuración nueva sin modelo principal, establece `agents.defaults.model` en `openai/gpt-5.6`; el identificador simple del modelo de API directa se resuelve en el nivel Sol.
    - Al añadir OpenAI o volver a autenticarse, se conserva cualquier modelo principal explícito existente, incluido `openai/gpt-5.5`. Si la cuenta no ofrece GPT-5.6, seleccione `openai/gpt-5.5` explícitamente; OpenClaw no cambia el modelo de forma silenciosa a uno inferior.
    - **OAuth de xAI**: inicio de sesión en el navegador mediante código de dispositivo sin necesidad de devolución de llamada a localhost, por lo que también funciona mediante SSH/Docker/VPS (`--auth-choice xai-oauth`).
    - **Clave de API de xAI**: solicita `XAI_API_KEY` (`--auth-choice xai-api-key`).
    - `--auth-choice xai-device-code` sigue funcionando como alias de compatibilidad exclusivamente manual para el mismo flujo de código de dispositivo OAuth de xAI; utilice `xai-oauth` para scripts nuevos.
    - **OpenCode**: solicita `OPENCODE_API_KEY` (o `OPENCODE_ZEN_API_KEY`; se obtiene en https://opencode.ai/auth) y permite elegir el catálogo Zen o Go.
    - **Ollama**: primero ofrece **Nube + local**, **Solo nube** o **Solo local**. `Cloud only` solicita `OLLAMA_API_KEY` y utiliza `https://ollama.com`; los modos respaldados por un host solicitan la URL base de Ollama (valor predeterminado: `http://127.0.0.1:11434`), detectan los modelos disponibles y descargan automáticamente el modelo local seleccionado cuando es necesario; `Cloud + Local` también comprueba si se ha iniciado sesión en ese host de Ollama para acceder a la nube.
    - Más información: [Ollama](/es/providers/ollama)
    - **Clave de API**: almacena la clave.
    - **Vercel AI Gateway (proxy multimodelo)**: solicita `AI_GATEWAY_API_KEY`.
    - Más información: [Vercel AI Gateway](/es/providers/vercel-ai-gateway)
    - **Cloudflare AI Gateway**: solicita Account ID, Gateway ID y `CLOUDFLARE_AI_GATEWAY_API_KEY`.
    - Más información: [Cloudflare AI Gateway](/es/providers/cloudflare-ai-gateway)
    - **MiniMax**: la configuración se escribe automáticamente; el valor predeterminado alojado es `MiniMax-M3`.
      La configuración mediante clave de API utiliza `minimax/...` y la configuración mediante OAuth utiliza
      `minimax-portal/...`.
    - Más información: [MiniMax](/es/providers/minimax)
    - **StepFun**: la configuración se escribe automáticamente para StepFun estándar o Step Plan en puntos de conexión de China o globales.
    - Actualmente, la versión estándar utiliza `step-3.5-flash` de forma predeterminada; Step Plan también incluye `step-3.5-flash-2603`.
    - Más información: [StepFun](/es/providers/stepfun)
    - **Synthetic (compatible con Anthropic)**: solicita `SYNTHETIC_API_KEY`.
    - Más información: [Synthetic](/es/providers/synthetic)
    - **Moonshot (Kimi K2)**: la configuración se escribe automáticamente.
    - **Kimi Coding**: la configuración se escribe automáticamente.
    - Más información: [Moonshot AI (Kimi + Kimi Coding)](/es/providers/moonshot)
    - **Proveedor personalizado**: funciona con puntos de conexión compatibles con OpenAI, con OpenAI Responses o con Anthropic. Opciones no interactivas: `--auth-choice custom-api-key`, `--custom-base-url`, `--custom-model-id`, `--custom-api-key` (opcional; recurre a `CUSTOM_API_KEY`), `--custom-provider-id` (opcional; se deriva automáticamente de la URL base), `--custom-compatibility openai|openai-responses|anthropic` (valor predeterminado: `openai`), `--custom-image-input` / `--custom-text-input` (anulan la detección inferida del modelo de visión).
    - **Omitir**: todavía no se ha configurado la autenticación.
    - Seleccione un modelo predeterminado entre las opciones detectadas (o introduzca manualmente el proveedor/modelo). Para obtener la mejor calidad y reducir el riesgo de inyección de instrucciones, elija el modelo de última generación más potente disponible en su conjunto de proveedores.
    - La incorporación ejecuta una comprobación del modelo y advierte si el modelo configurado es desconocido o no tiene autenticación.
    - El modo de almacenamiento de claves de API utiliza de forma predeterminada valores de perfiles de autenticación en texto sin formato. Utilice `--secret-input-mode ref` para almacenar referencias respaldadas por variables de entorno (por ejemplo, `keyRef: { source: "env", provider: "default", id: "OPENAI_API_KEY" }`); la variable de entorno a la que se hace referencia debe estar ya establecida o la incorporación fallará de inmediato.
    - Los perfiles de autenticación se encuentran en `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` (claves de API + OAuth). `~/.openclaw/credentials/oauth.json` solo se utiliza para la importación heredada.
    - Más información: [OAuth](/es/concepts/oauth)
    <Note>
    Consejo para servidores/sistemas sin interfaz gráfica: complete OAuth en una máquina con navegador y, después, copie
    el archivo `auth-profiles.json` de ese agente (por ejemplo,
    `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` o la ruta correspondiente de
    `$OPENCLAW_STATE_DIR/...`) al host del Gateway. `credentials/oauth.json`
    es únicamente una fuente de importación heredada.
    </Note>
  </Step>
  <Step title="Espacio de trabajo">
    - Valor predeterminado: `~/.openclaw/workspace` (configurable).
    - Crea los archivos del espacio de trabajo necesarios para el ritual de arranque del agente.
    - Diseño completo del espacio de trabajo y guía de copias de seguridad: [Espacio de trabajo del agente](/es/concepts/agent-workspace)

  </Step>
  <Step title="Gateway">
    - Puerto (valor predeterminado: **18789**), vinculación, modo de autenticación y exposición mediante Tailscale.
    - Recomendación de autenticación: mantenga **Token** incluso para la interfaz de bucle invertido, de modo que los clientes WS locales deban autenticarse.
    - En el modo de token, la configuración interactiva ofrece:
      - **Generar/almacenar token en texto sin formato** (valor predeterminado)
      - **Usar SecretRef** (opcional)
      - El inicio rápido reutiliza las SecretRefs existentes de `gateway.auth.token` entre los proveedores `env`, `file` y `exec` para la comprobación de incorporación y el arranque del panel.
      - Si esa SecretRef está configurada pero no se puede resolver, la incorporación falla de forma anticipada con un mensaje claro para corregir el problema, en lugar de degradar silenciosamente la autenticación del entorno de ejecución.
    - En el modo de contraseña, la configuración interactiva también permite almacenarla en texto sin formato o como SecretRef.
    - Ruta de SecretRef del token no interactivo: `--gateway-token-ref-env <ENV_VAR>`.
      - Requiere una variable de entorno no vacía en el entorno del proceso de incorporación.
      - No se puede combinar con `--gateway-token`.
    - Desactive la autenticación solo si confía plenamente en todos los procesos locales.
    - Las vinculaciones que no sean de bucle invertido siguen requiriendo autenticación.

  </Step>
  <Step title="Canales">
    - [WhatsApp](/es/channels/whatsapp): inicio de sesión opcional mediante QR.
    - [Telegram](/es/channels/telegram): token de bot.
    - [Discord](/es/channels/discord): token de bot.
    - [Google Chat](/es/channels/googlechat): JSON de la cuenta de servicio + audiencia del Webhook.
    - [Mattermost](/es/channels/mattermost) (plugin): token de bot + URL base.
    - [Signal](/es/channels/signal) (plugin): instalación opcional de `signal-cli` + configuración de la cuenta.
    - [iMessage](/es/channels/imessage): ruta de la CLI `imsg` + acceso a la base de datos de Messages; utilice un contenedor SSH cuando el Gateway se ejecute fuera de un Mac.
    - Discord, Feishu, Microsoft Teams, QQ Bot, Slack y otros canales se distribuyen como
      plugins que la incorporación puede instalar. Catálogo completo: [Canales](/es/channels).
    - Seguridad de mensajes directos: el valor predeterminado es la vinculación. El primer mensaje directo envía un código; apruébelo mediante `openclaw pairing approve <channel> <code>` o utilice listas de permitidos.

  </Step>
  <Step title="Búsqueda web">
    - Seleccione un proveedor compatible, como Brave, Codex (Hosted Search), DuckDuckGo, Exa, Firecrawl, Gemini, Grok, Kimi, MiniMax Search, Ollama Web Search, Parallel, Perplexity, SearXNG o Tavily (o bien omita este paso).
    - Los proveedores respaldados por API pueden utilizar variables de entorno o la configuración existente para agilizar la configuración; los proveedores sin clave utilizan en su lugar sus requisitos previos específicos.
    - Omita este paso con `--skip-search`.
    - Configúrelo más adelante: `openclaw configure --section web`.

  </Step>
  <Step title="Instalación del daemon">
    - macOS: LaunchAgent
      - Requiere una sesión de usuario iniciada; para sistemas sin interfaz gráfica, utilice un LaunchDaemon personalizado (no se incluye).
    - Linux (y Windows mediante WSL2): unidad de usuario de systemd
      - La incorporación intenta habilitar la persistencia mediante `loginctl enable-linger <user>` para que el Gateway siga ejecutándose después de cerrar sesión.
      - Puede solicitar sudo (escribe `/var/lib/systemd/linger`); primero lo intenta sin sudo.
    - Windows nativo: primero se usa una tarea programada; si se deniega su creación, OpenClaw recurre a un elemento de inicio de sesión por usuario en la carpeta Inicio e inicia el Gateway inmediatamente.
    - **Selección del entorno de ejecución:** Node es obligatorio porque el almacén canónico de estado del entorno de ejecución utiliza `node:sqlite`. Los servicios heredados de Bun se migran a Node durante la reparación.
    - Si la autenticación mediante token requiere uno y `gateway.auth.token` está gestionado mediante SecretRef, la instalación del daemon lo valida, pero no conserva los valores resueltos del token en texto sin formato en los metadatos del entorno de servicio del supervisor.
    - Si la autenticación mediante token requiere uno y la SecretRef configurada para el token no se puede resolver, se bloquea la instalación del daemon con instrucciones prácticas.
    - Si tanto `gateway.auth.token` como `gateway.auth.password` están configurados y `gateway.auth.mode` no está establecido, se bloquea la instalación del daemon hasta que el modo se configure explícitamente.

  </Step>
  <Step title="Comprobación de estado">
    - Inicia el Gateway (si es necesario) y ejecuta `openclaw health`.
    - Consejo: `openclaw status --deep` añade la comprobación de estado en vivo del Gateway a la salida de estado, incluidas las comprobaciones de canales cuando se admitan (requiere un Gateway accesible).

  </Step>
  <Step title="Skills (recomendadas)">
    - Lee las Skills disponibles y comprueba los requisitos.
    - Permite elegir un gestor de Node: **npm / pnpm / bun**.
    - Instala automáticamente las dependencias opcionales de las Skills incluidas de confianza (algunas utilizan Homebrew en macOS).
    - Omite las Skills cuyo requisito previo de instalación mediante Homebrew, uv o Go no esté disponible, las agrupa con instrucciones de configuración manual e indica `openclaw doctor` una vez instalado el requisito previo.

  </Step>
  <Step title="Finalización">
    - Resumen + pasos siguientes, incluida la pregunta **¿Cómo desea iniciar su agente?** para Terminal, Navegador o más adelante.

  </Step>
</Steps>

<Note>
Si no se detecta ninguna interfaz gráfica, la incorporación muestra instrucciones de reenvío de puertos SSH para la interfaz de control en lugar de abrir un navegador.
Si faltan los recursos de la interfaz de control, la incorporación intenta compilarlos; la alternativa es `pnpm ui:build` (instala automáticamente las dependencias de la interfaz).
</Note>

## Modo no interactivo

Use `--non-interactive --accept-risk` para automatizar o programar la incorporación (la
marca es la confirmación de riesgo obligatoria; la incorporación termina con un error
si no se incluye):

```bash
openclaw onboard --non-interactive --accept-risk \
  --mode local \
  --auth-choice apiKey \
  --anthropic-api-key "$ANTHROPIC_API_KEY" \
  --gateway-port 18789 \
  --gateway-bind loopback \
  --install-daemon \
  --daemon-runtime node \
  --skip-skills
```

Añada `--json` para obtener un resumen legible por máquina.

SecretRef del token del Gateway en modo no interactivo:

```bash
export OPENCLAW_GATEWAY_TOKEN="your-token"
openclaw onboard --non-interactive --accept-risk \
  --mode local \
  --auth-choice skip \
  --gateway-auth token \
  --gateway-token-ref-env OPENCLAW_GATEWAY_TOKEN
```

`--gateway-token` y `--gateway-token-ref-env` son mutuamente excluyentes.

<Note>
`--json` **no** implica el modo no interactivo. Use `--non-interactive --accept-risk` (y `--workspace`) para scripts.
</Note>

Los ejemplos de comandos específicos de cada proveedor se encuentran en [Automatización de la CLI](/es/start/wizard-cli-automation#provider-specific-examples).
Use esta página de referencia para consultar la semántica de las marcas y el orden de los pasos.

### Añadir agente (modo no interactivo)

```bash
openclaw agents add work \
  --workspace ~/.openclaw/workspace-work \
  --model openai/gpt-5.6-sol \
  --bind whatsapp:biz \
  --non-interactive \
  --json
```

`main` es un id. de agente reservado y no puede utilizarse para `openclaw agents add`.

## RPC del asistente del Gateway

El Gateway expone el flujo de incorporación mediante RPC (`wizard.start`, `wizard.next`, `wizard.cancel`, `wizard.status`).
Los clientes (aplicación para macOS, interfaz de control) pueden representar los pasos sin volver a implementar la lógica de incorporación.

## Configuración de Signal (signal-cli)

La incorporación detecta si `signal-cli` está en `PATH` y, si falta, ofrece instalarlo:

- Linux x86-64: descarga la compilación nativa oficial de GraalVM desde las versiones de GitHub de `signal-cli` y la almacena en `~/.openclaw/tools/signal-cli/<version>/`.
- macOS y otras arquitecturas: realiza la instalación mediante Homebrew.
- Windows nativo: aún no es compatible; ejecute la incorporación dentro de WSL2 para usar la ruta de instalación de Linux.
- En cualquier caso, escribe `channels.signal.transport.cliPath` con `kind: "managed-native"`.

## Qué escribe el asistente

Campos habituales en `~/.openclaw/openclaw.json`:

- `agents.defaults.workspace`
- `agents.defaults.skipBootstrap` cuando se proporciona `--skip-bootstrap`
- `agents.defaults.model` / `models.providers` (si se elige Minimax)
- `tools.profile` (la incorporación local utiliza de forma predeterminada `"coding"` cuando no está definido; se conservan los valores explícitos existentes)
- `gateway.*` (modo, enlace, autenticación, Tailscale)
- `session.dmScope` (la incorporación conserva los valores explícitos y, de lo contrario, lo deja sin definir, por lo que el valor predeterminado `"main"` mantiene todos los mensajes directos de todos los canales en la sesión principal continua del agente, que es el valor predeterminado para agentes personales. Para bandejas de entrada compartidas o multiusuario, use `"per-channel-peer"`; `openclaw security audit` recomienda el aislamiento cuando detecta tráfico de mensajes directos de varios usuarios. Detalles: [Referencia de configuración de la CLI](/es/start/wizard-cli-reference#outputs-and-internals))
- `channels.telegram.botToken`, `channels.discord.token`, `channels.matrix.*`, `channels.signal.*`, `channels.imessage.*`
- Listas de remitentes permitidos para mensajes directos de canales cuando se habilitan durante las solicitudes de configuración de canales. Discord, Matrix, Microsoft Teams y Slack convierten los nombres en identificadores cuando es posible; los demás canales reciben los identificadores directamente (por ejemplo, identificadores numéricos de remitentes de Telegram o números de teléfono de WhatsApp).
- `skills.install.nodeManager`
  - `setup --node-manager` acepta `npm`, `pnpm` o `bun`.
  - La configuración manual todavía puede usar `yarn` estableciendo `skills.install.nodeManager` directamente.
- `wizard.lastRunAt`
- `wizard.lastRunVersion`
- `wizard.lastRunCommit`
- `wizard.lastRunCommand`
- `wizard.lastRunMode`
- `wizard.securityAcknowledgedAt`

`openclaw agents add` escribe `agents.entries.*` y el valor opcional `bindings`.

Las credenciales de WhatsApp se guardan en `~/.openclaw/credentials/whatsapp/<accountId>/`.
Las sesiones activas y las transcripciones se almacenan en
`~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`. El directorio
`~/.openclaw/agents/<agentId>/sessions/` se utiliza para las entradas de migración heredadas
y los artefactos de archivado y soporte.

Algunos canales se distribuyen como plugins. Cuando se selecciona uno durante la configuración, la incorporación
solicita instalarlo (desde npm o una ruta local) antes de poder configurarlo.

## Documentación relacionada

- Descripción general de la incorporación: [Incorporación (CLI)](/es/start/wizard)
- Referencia de configuración de la CLI: [Referencia de configuración de la CLI](/es/start/wizard-cli-reference)
- Incorporación en la aplicación para macOS: [Incorporación](/es/start/onboarding)
- Referencia de configuración: [Configuración del Gateway](/es/gateway/configuration)
- Proveedores: [WhatsApp](/es/channels/whatsapp), [Telegram](/es/channels/telegram), [Discord](/es/channels/discord), [Google Chat](/es/channels/googlechat), [Signal](/es/channels/signal), [iMessage](/es/channels/imessage)
- Skills: [Skills](/es/tools/skills), [Configuración de Skills](/es/tools/skills-config)
