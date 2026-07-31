---
read_when:
    - Añadir o modificar migraciones de doctor
    - Introducción de cambios incompatibles en la configuración
sidebarTitle: Doctor
summary: 'Comando doctor: comprobaciones de estado, migraciones de configuración y pasos de reparación'
title: Diagnóstico
x-i18n:
    generated_at: "2026-07-26T05:39:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3f599553a2455759cd0fe56bafbc16948f7ab4d381d344b08a496bf19c9dc636
    source_path: gateway/doctor.md
    workflow: 16
---

`openclaw doctor` es la herramienta de reparación y migración de OpenClaw. Corrige configuraciones y estados obsoletos, comprueba el estado del sistema y proporciona pasos de reparación prácticos.

## Inicio rápido

```bash
openclaw doctor
```

### Modos sin interfaz y de automatización

<Tabs>
  <Tab title="--yes">
    ```bash
    openclaw doctor --yes
    ```

    Acepta los valores predeterminados sin solicitar confirmación (incluidos los pasos de reparación de reinicio, servicio y entorno aislado cuando corresponda).

  </Tab>
  <Tab title="--fix">
    ```bash
    openclaw doctor --fix
    ```

    Aplica las reparaciones recomendadas sin solicitar confirmación (`--repair` es un alias).

  </Tab>
  <Tab title="--lint">
    ```bash
    openclaw doctor --lint
    openclaw doctor --lint --json
    ```

    Ejecuta comprobaciones de estado estructuradas para la Pipeline de CI o la automatización previa. Es de solo lectura: no
    solicita confirmación ni realiza reparaciones, migraciones, reinicios o escrituras de estado.

  </Tab>
  <Tab title="--fix --force">
    ```bash
    openclaw doctor --fix --force
    ```

    También aplica reparaciones agresivas (sobrescribe las configuraciones personalizadas del supervisor).

  </Tab>
  <Tab title="--non-interactive">
    ```bash
    openclaw doctor --non-interactive
    ```

    Se ejecuta sin solicitar confirmación y aplica únicamente migraciones seguras (normalización de la configuración +
    traslados del estado en disco). Omite las acciones de reinicio, servicio y entorno aislado que requieren
    confirmación humana. Las migraciones de estado heredado se siguen ejecutando automáticamente cuando se detectan.

  </Tab>
  <Tab title="--deep">
    ```bash
    openclaw doctor --deep
    ```

    Examina los servicios del sistema en busca de instalaciones adicionales del Gateway (launchd/systemd/schtasks).

  </Tab>
</Tabs>

Para revisar los cambios antes de escribirlos, abre primero el archivo de configuración:

```bash
cat ~/.openclaw/openclaw.json
```

## Modo de análisis de solo lectura

`openclaw doctor --lint` es la variante orientada a la automatización de
`openclaw doctor --fix`. Comparten el mismo registro de reglas de Doctor, pero no
seleccionan ni aplican las reglas del mismo modo:

| Modo                     | Solicita confirmación | Escribe configuración/estado | Salida                    | Uso previsto                          |
| ------------------------ | ---------------------- | ---------------------------- | ------------------------- | ------------------------------------- |
| `openclaw doctor`        | sí                     | no                           | informe de estado legible | una persona que comprueba el estado   |
| `openclaw doctor --fix`  | a veces                | sí, con política de reparación | registro de reparación legible | aplicar reparaciones aprobadas    |
| `openclaw doctor --lint` | no                     | no                           | hallazgos estructurados   | Pipeline de CI, comprobaciones previas y controles de revisión |

De forma predeterminada, `doctor --lint` ejecuta el perfil amplio y seguro de automatización: comprobaciones
estáticas, locales y útiles para la salida de la Pipeline de CI o de las comprobaciones previas. Omite las comprobaciones opcionales
que son informativas, dependen del entorno o de servicios activos, inventarían cuentas o espacios de trabajo,
o realizan limpieza histórica. Usa `doctor --lint --all` para ejecutar la
auditoría completa de análisis registrada, incluidas esas comprobaciones opcionales, o `--only <id>` para
una comprobación específica.

`doctor --fix` no utiliza el perfil de análisis predeterminado y no acepta
`--all`. Ejecuta la ruta de reparación ordenada de Doctor: las comprobaciones de estado modernas pueden proporcionar
una implementación opcional de `repair()`, mientras que las áreas más antiguas siguen utilizando su flujo de reparación
heredado de Doctor. Algunos hallazgos del análisis son deliberadamente solo diagnósticos, por lo que
la aparición de una comprobación en `--lint --all` no implica que `--fix` vaya a modificar esa área.
El contrato separa `detect()` (informa de hallazgos) de `repair()` (informa de
cambios, diferencias y efectos secundarios), lo que deja abierta la posibilidad de un futuro
`doctor --fix --dry-run` sin convertir las comprobaciones de análisis en planificadores de modificaciones.

Algunas comprobaciones integradas están desactivadas internamente de forma predeterminada para que sigan disponibles para
`--all`, `--only` y los flujos de reparación de Doctor sin formar parte del perfil de automatización
predeterminado de `doctor --lint`. La gravedad de cada hallazgo se sigue emitiendo
individualmente (`info`, `warning` o `error`); la selección predeterminada no es un nivel de
gravedad.

```bash
openclaw doctor --lint
openclaw doctor --lint --severity-min warning
openclaw doctor --lint --json
openclaw doctor --lint --all
openclaw doctor --lint --only core/doctor/gateway-config --json
```

Campos de la salida JSON:

- `ok`: indica si algún hallazgo alcanzó el umbral de gravedad seleccionado
- `checksRun` / `checksSkipped`: recuentos (omitidos por el perfil, `--only` o `--skip`)
- `findings`: diagnósticos estructurados con `checkId`, `severity`, `message` y, opcionalmente, `path`, `line`, `column`, `ocPath`, `source`, `target`, `requirement`, `fixHint`

Códigos de salida:

| Código | Significado                                              |
| ------ | -------------------------------------------------------- |
| `0`  | no hay hallazgos que alcancen o superen el umbral seleccionado |
| `1`  | uno o más hallazgos alcanzaron el umbral seleccionado    |
| `2`  | fallo del comando o del entorno de ejecución antes de poder emitir los hallazgos |

Opciones:

- `--severity-min info|warning|error` (valor predeterminado: `warning`): controla tanto lo que se muestra como lo que produce una salida distinta de cero.
- `--all`: ejecuta todas las comprobaciones de análisis registradas, incluidas las opcionales excluidas del conjunto de automatización predeterminado.
- `--only <id>` (repetible): ejecuta únicamente los identificadores de comprobación indicados; un identificador desconocido se notifica como un hallazgo de error.
- `--skip <id>` (repetible): excluye una comprobación y mantiene activo el resto de la ejecución.
- `--json`, `--severity-min`, `--all`, `--only` y `--skip` requieren `--lint`; las ejecuciones simples de `openclaw doctor` y `--fix` las rechazan.

## Qué hace (resumen)

<AccordionGroup>
  <Accordion title="Estado, interfaz y actualizaciones">
    - Actualización previa opcional para instalaciones mediante git (solo en modo interactivo).
    - Comprobación de vigencia del protocolo de la interfaz (recompila la interfaz de control cuando el esquema del protocolo es más reciente).
    - Comprobación de estado y solicitud de reinicio.
    - Notas sobre Skills y plugins solo cuando hay problemas; el inventario sin problemas permanece en `openclaw skills check` y `openclaw plugins list`.

  </Accordion>
  <Accordion title="Configuración y migraciones">
    - Normalización de la configuración para formatos de valores heredados.
    - Migración de la configuración de conversación desde los campos planos heredados de `talk.*` a `talk.provider` + `talk.providers.<provider>`.
    - Comprobaciones de migración del navegador para configuraciones heredadas de la extensión de Chrome y la preparación de Chrome MCP.
    - Advertencias de sustitución del proveedor OpenCode (`models.providers.opencode` / `opencode-zen` / `opencode-go`).
    - Migración del proveedor y perfil heredados de OpenAI Codex (`openai-codex` → `openai`) y advertencias de ocultación por `models.providers.openai-codex` obsoletos.
    - Comprobación de los requisitos previos de TLS para perfiles OAuth de OpenAI Codex.
    - Advertencias sobre listas de permitidos de plugins y herramientas cuando `plugins.allow` es restrictivo, pero la política de herramientas sigue solicitando comodines o herramientas pertenecientes a plugins.
    - Migración del estado heredado en disco (sesiones, directorio del agente y autenticación de WhatsApp).
    - Migración de claves heredadas del contrato del manifiesto del plugin (`speechProviders`, `realtimeTranscriptionProviders`, `realtimeVoiceProviders`, `mediaUnderstandingProviders`, `imageGenerationProviders`, `videoGenerationProviders`, `webFetchProviders`, `webSearchProviders` → `contracts`).
    - Migración del almacén de Cron heredado (`jobId`, `schedule.cron`, campos de entrega y carga útil de nivel superior, `provider` de la carga útil y tareas de Webhook de reserva de `notify: true`).
    - Reparación de la fijación del entorno de ejecución de la CLI de Codex (`agentRuntime.id: "codex-cli"` → `"codex"`) en `agents.defaults`, `agents.entries.*` y `models.providers.*` (incluidas las entradas por modelo).
    - Limpieza de la configuración obsoleta de plugins cuando estos están habilitados; con `plugins.enabled=false`, las referencias obsoletas a plugins se conservan como configuración de contención inerte.

  </Accordion>
  <Accordion title="Estado e integridad">
    - Inspección de archivos de bloqueo de sesiones y limpieza de bloqueos obsoletos.
    - Reparación de transcripciones de sesiones con ramas duplicadas de reescritura de instrucciones creadas por las compilaciones afectadas de 2026.4.24.
    - Detección de marcadores de recuperación tras reinicio para sesiones principales y subagentes bloqueados. Doctor informa de las sesiones bloqueadas y solo repara las marcas de interrupción obsoletas que entran en conflicto con un marcador existente; no vuelve a habilitar la recuperación automática.
    - Comprobaciones de integridad del estado y permisos (sesiones, transcripciones y directorio de estado).
    - Comprobaciones de permisos del archivo de configuración (chmod 600) cuando se ejecuta localmente.
    - Estado de la autenticación de modelos: comprueba la caducidad de OAuth, puede renovar los tokens próximos a caducar e informa de los estados de espera o desactivación de los perfiles de autenticación.

  </Accordion>
  <Accordion title="Gateway, servicios y supervisores">
    - Reparación de la imagen del entorno aislado cuando este está habilitado.
    - Migración de servicios heredados y detección de Gateways adicionales.
    - Migración del estado heredado del canal Matrix (en modo `--fix` / `--repair`).
    - Comprobaciones del entorno de ejecución del Gateway (servicio instalado pero no iniciado; etiqueta de launchd almacenada en caché).
    - Advertencias sobre el estado de los canales (consultado desde el Gateway en ejecución).
    - Las comprobaciones de permisos específicas de cada canal se encuentran en `openclaw channels capabilities`; por ejemplo, los permisos de los canales de voz de Discord se auditan con `openclaw channels capabilities --channel discord --target channel:<channel-id>`.
    - Comprobaciones de capacidad de respuesta de WhatsApp ante un estado degradado del bucle de eventos del Gateway mientras siguen ejecutándose clientes TUI locales; `--fix` detiene únicamente los clientes TUI locales verificados.
    - Reparación de rutas de Codex para referencias heredadas de modelos `openai-codex/*` en modelos principales, alternativas, modelos de generación de imágenes y vídeos, sustituciones de Heartbeat, subagente y Compaction, hooks, sustituciones de modelos de canales y fijaciones de rutas de sesión; `--fix` las reescribe como `openai/*`, migra los perfiles y el orden de autenticación de `openai-codex:*` a `openai:*`, elimina las fijaciones obsoletas del entorno de ejecución de sesiones y agentes completos, y permite que la ruta efectiva reparada determine si Codex es compatible.
    - Auditoría de la configuración del supervisor (launchd/systemd/schtasks) con reparación opcional.
    - Limpieza del entorno de proxy incorporado para servicios del Gateway que capturaron valores de shell `HTTP_PROXY` / `HTTPS_PROXY` / `NO_PROXY` durante la instalación o actualización.
    - Comprobaciones del entorno de ejecución del Gateway (servicios heredados de Bun no compatibles y rutas de gestores de versiones).
    - Diagnósticos de colisión de puertos del Gateway (valor predeterminado: `18789`).

  </Accordion>
  <Accordion title="Autenticación, seguridad y emparejamiento">
    - Advertencias de seguridad para políticas de mensajes directos abiertas.
    - Comprobaciones de autenticación del Gateway para el modo de token local (ofrece generar un token cuando no existe ninguna fuente de tokens; no sobrescribe configuraciones SecretRef de tokens).
    - Detección de problemas de emparejamiento de dispositivos (solicitudes pendientes de primer emparejamiento, mejoras pendientes de rol o ámbito, divergencias obsoletas en la caché local de tokens de dispositivo y divergencias de autenticación en registros emparejados).

  </Accordion>
  <Accordion title="Espacio de trabajo y shell">
    - Comprobación de persistencia de systemd en Linux.
    - Comprobación del tamaño de los archivos de inicialización del espacio de trabajo (advertencias de truncamiento o proximidad al límite para archivos de contexto).
    - Comprobación de preparación de Skills para el agente predeterminado; informa de las Skills permitidas a las que les faltan binarios, variables de entorno, configuración o requisitos del sistema operativo, y `--fix` puede deshabilitar las Skills no disponibles en `skills.entries`.
    - Comprobación del estado del completado de shell e instalación o actualización automática.
    - Comprobación de preparación del proveedor de incrustaciones para la búsqueda en memoria (modelo local, clave de API remota o binario QMD).
    - Comprobaciones de instalaciones desde el código fuente (discrepancia del espacio de trabajo de pnpm, recursos de interfaz ausentes y binario tsx ausente).
    - Escribe la configuración actualizada y los metadatos del asistente.

  </Accordion>
</AccordionGroup>

## Restauración y restablecimiento de la interfaz de Dreams

  La escena Dreams de la interfaz de control incluye las acciones **Rellenar**, **Restablecer** y **Borrar Grounded** para el flujo de trabajo de Dreaming fundamentado. Estas utilizan métodos RPC del Gateway al estilo de doctor, pero **no** forman parte de la reparación/migración de la CLI `openclaw doctor`.

  | Acción          | Qué hace                                                                                                                                                                                         |
  | --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
  | Rellenar        | Examina los archivos históricos `memory/YYYY-MM-DD.md` del espacio de trabajo activo, ejecuta el procesamiento del diario REM fundamentado y escribe entradas de relleno reversibles en `DREAMS.md`. |
  | Restablecer     | Elimina únicamente las entradas marcadas del diario de relleno de `DREAMS.md`.                                                                                                            |
  | Borrar Grounded | Elimina únicamente las entradas provisionales a corto plazo exclusivas de Grounded procedentes de la reproducción histórica que aún no han acumulado recuperación en vivo ni respaldo diario.     |

  Ninguna de estas acciones edita `MEMORY.md`, ejecuta migraciones completas de doctor ni incorpora por sí sola candidatos fundamentados al almacén activo de promoción a corto plazo. Para introducir la reproducción histórica fundamentada en la vía normal de promoción profunda, utilice en su lugar el flujo de la CLI:

  ```bash
  openclaw memory rem-backfill --path ./memory --stage-short-term
  ```

  Esto incorpora candidatos duraderos fundamentados al almacén de Dreaming a corto plazo, mientras `DREAMS.md` sigue siendo la superficie de revisión.

  ## Comportamiento detallado y justificación

  <AccordionGroup>
  <Accordion title="0. Actualización opcional (instalaciones mediante git)">
    Si se trata de un checkout de git y doctor se ejecuta de forma interactiva, ofrece actualizar (fetch/rebase/build) antes de ejecutar doctor.
  </Accordion>
  <Accordion title="1. Normalización de la configuración">
    Doctor normaliza las estructuras de valores heredadas al esquema actual. La configuración actual de voz de Talk es `talk.provider` + `talk.providers.<provider>`, con la configuración de voz en tiempo real en `talk.realtime.*`. Doctor reescribe las estructuras antiguas `talk.voiceId` / `talk.voiceAliases` / `talk.modelId` / `talk.outputFormat` / `talk.apiKey` en el mapa de proveedores y reescribe los selectores heredados de nivel superior para tiempo real (`talk.mode`, `talk.transport`, `talk.brain`, `talk.model`, `talk.voice`) en `talk.realtime`.

    Doctor también advierte cuando `plugins.allow` no está vacío y la política de herramientas utiliza entradas con comodines o pertenecientes a plugins. `tools.allow: ["*"]` solo coincide con herramientas de plugins que realmente se cargan; no omite la lista de permitidos exclusiva de plugins.

  </Accordion>
  <Accordion title="2. Migraciones de claves de configuración heredadas">
    Cuando la configuración contiene una clave obsoleta con una migración activa, los demás comandos se niegan a ejecutarse y solicitan que se ejecute `openclaw doctor`. Doctor explica qué claves heredadas se encontraron, muestra la migración aplicada y reescribe `~/.openclaw/openclaw.json` con el esquema actualizado. El inicio del Gateway rechaza los formatos de configuración heredados y solicita que se ejecute `openclaw doctor --fix`; no reescribe `openclaw.json` durante el inicio. `openclaw doctor --fix` también gestiona las migraciones del almacén de tareas de Cron.

    <Note>
      Doctor solo conserva migraciones automáticas durante aproximadamente dos
      meses después de retirar una clave. Las claves heredadas más antiguas (por
      ejemplo, las originales `routing.queue`, `routing.bindings`,
      `routing.agents`/`defaultAgentId`, `routing.transcribeAudio`, la
      `agent.*` de nivel superior o la `identity` de nivel
      superior de la estructura de configuración anterior a los múltiples
      agentes) ya no disponen de una ruta de migración; ahora la configuración
      que las utiliza no supera la validación en lugar de reescribirse. Corrija
      esas claves manualmente conforme a la referencia de configuración actual
      antes de que doctor pueda continuar.
    </Note>

    Migraciones activas:

    | Clave heredada                                                                                    | Clave actual                                                                 |
    | ----------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
    | `routing.allowFrom`                                                                              | `channels.whatsapp.allowFrom`                                                |
    | `routing.groupChat.requireMention`                                                               | `channels.whatsapp/telegram/imessage.groups."*".requireMention`             |
    | `routing.groupChat.historyLimit`                                                                 | `messages.groupChat.historyLimit`                                            |
    | `routing.groupChat.mentionPatterns`                                                              | `messages.groupChat.mentionPatterns`                                         |
    | `channels.telegram.requireMention`                                                               | `channels.telegram.groups."*".requireMention`                               |
    | `channels.webchat`, `gateway.webchat`                                                            | eliminadas (WebChat se ha retirado)                                                 |
    | `channels.feishu.accounts.<accountId>.botName`                                                   | `channels.feishu.accounts.<accountId>.name`                                 |
    | `session.threadBindings.ttlHours`, `channels.<id>.threadBindings.ttlHours` (y por cuenta)      | `...threadBindings.idleHours`                                               |
    | `talk.voiceId`/`talk.voiceAliases`/`talk.modelId`/`talk.outputFormat`/`talk.apiKey` heredadas        | `talk.provider` + `talk.providers.<provider>`                               |
    | selectores de Talk en tiempo real de nivel superior heredados (`talk.mode`/`talk.transport`/`talk.brain`/`talk.model`/`talk.voice`) | `talk.realtime`                                                              |
    | `messages.tts`                                                                                  | `tts` de nivel superior                                                              |
    | `messages.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`)                             | `tts.providers.<provider>`                                                   |
    | `messages.tts.provider: "edge"` / `messages.tts.providers.edge`                                  | `tts.provider: "microsoft"` / `tts.providers.microsoft`                    |
    | `tools.exec.security` + `tools.exec.ask`                                                         | `tools.exec.mode`                                                            |
    | `session.idleMinutes`                                                                            | `session.reset.idleMinutes`                                                  |
    | `messages.responsePrefix` con bloques de canal explícitos                                           | copiada a la `responsePrefix` del canal o la cuenta configurados; se conserva la alternativa global para canales implícitos o personalizados |
    | `web.enabled`                                                                                    | `channels.whatsapp.enabled`                                                  |
    | `meta.lastTouchedAt`, instalaciones de hooks, almacén de Cron, detección incluida, ruta global de preferencias de TTS            | estado SQLite compartido                                                       |
    | campos de hablante de TTS `voice`/`voiceName`/`voiceId`                                                 | `speakerVoice`/`speakerVoiceId`                                              |
    | `channels.<id>.tts.<provider>` / `channels.<id>.accounts.<accountId>.tts.<provider>` (todos los canales excepto Discord)                                          | `...tts.providers.<provider>`                                                |
    | `channels.<id>.voice.tts.<provider>` / `channels.<id>.accounts.<accountId>.voice.tts.<provider>` (todos los canales, incluido Discord)                          | `...voice.tts.providers.<provider>`                                          |
    | `plugins.entries.voice-call.config.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`)     | `plugins.entries.voice-call.config.tts.providers.<provider>`                |
    | `plugins.entries.voice-call.config.tts.provider: "edge"` / `...tts.providers.edge`                | `provider: "microsoft"` / `...tts.providers.microsoft`                      |
    | `plugins.entries.voice-call.config.provider: "log"`                                              | `"mock"`                                                                      |
    | `plugins.entries.voice-call.config.twilio.from`                                                  | `plugins.entries.voice-call.config.fromNumber`                              |
    | `plugins.entries.voice-call.config.streaming.sttProvider`                                        | `plugins.entries.voice-call.config.streaming.provider`                      |
    | `plugins.entries.voice-call.config.streaming.openaiApiKey`/`sttModel`/`silenceDurationMs`/`vadThreshold` | `plugins.entries.voice-call.config.streaming.providers.openai.*`             |
    | `models.providers.*.api: "openai"`                                                               | `"openai-completions"` (el inicio del Gateway también omite los proveedores cuyo `api` sea un valor de enumeración futuro o desconocido, en lugar de impedir el inicio) |
    | `browser.ssrfPolicy.allowPrivateNetwork`                                                         | `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork`                          |
    | `browser.profiles.*.driver: "extension"`                                                         | `"existing-session"`                                                          |
    | `browser.relayBindHost`                                                                          | eliminada (configuración heredada del relé de la extensión de Chrome)                             |
    | `mcp.servers.*.type` (alias nativos de la CLI)                                                        | `mcp.servers.*.transport`                                                    |
    | `mcp.servers.*.disabled`                                                                         | `mcp.servers.*.enabled` inversa                                              |
    | alias de tiempo de espera de MCP `connectTimeout`/`connect_timeout`/`timeout`                                 | `connectionTimeoutMs`/`requestTimeoutMs`                                    |
    | campos de servidor MCP en snake_case                                                                     | campos de servidor MCP en camelCase                                                   |
    | `tools.media.image/audio/video.models`                                                           | `tools.media.models` etiquetada por capacidad                                        |
    | `tools.media.asyncCompletion`                                                                    | eliminada                                                                       |
    | `tools.message.allowCrossContextSend`                                                            | `tools.message.crossContext`                                                  |
    | opciones de `deepgram` del modelo multimedia                                                                   | `providerOptions.deepgram`                                                    |
    | `talk.realtime.voice`, `voice` en tiempo real de Discord                                                 | `speakerVoice`                                                                |
    | `agents.defaults.pdfMaxBytesMb`                                                                  | `agents.defaults.pdfMaxMb`                                                    |
    | `tools.exec.timeoutSec`                                                                          | `tools.exec.timeoutSeconds`                                                   |
    | `browser.ssrfPolicy.hostnameAllowlist`                                                           | `browser.ssrfPolicy.allowedHostnames` compatible con comodines                          |
    | `enableNoVnc` del navegador del entorno aislado                                                                    | `noVncEnabled`                                                                |
    | `media` raíz                                                                                     | `attachments`                                                                |
    | bloques de visibilidad `heartbeat` de canal o cuenta                                                   | `heartbeatVisibility`                                                         |
    | `channels.slack.identity`                                                                        | `channels.slack.postAs`                                                       |
    | `audit` raíz                                                                                     | `logging.audit`                                                               |
    | `gateway.nodes.skills.enabled`                                                                   | `gateway.nodes.allowSkills`                                                   |
    | `gateway.nodes.allowCommands`/`denyCommands`                                                    | `gateway.nodes.commands.allow`/`deny`                                         |
    | valores predeterminados del modelo de generación                                                                       | `agents.defaults.mediaModels.{image,video,music}`                              |
    | controles de ajuste retirados del diseño final                                                               | comportamiento predeterminado incorporado                                                     |
    | `channels.whatsapp.messagePrefix` y `messages.messagePrefix` heredada                            | `channels.whatsapp.responsePrefix`                                            |
    | `channels.whatsapp.ackReaction`                                                                  | `messages.ackReaction` global y `ackReactionScope` donde sean traducibles        |
    | `cron.failureDestination`                                                                        | campos de destino en `cron.failureAlert`                                     |
    | `gateway.controlUi.chatMessageMaxWidth`, claves `ui.prefs` solo de presentación                       | eliminadas (la escala del texto, el ancho del chat y la actividad en directo de la barra lateral son locales del navegador) |
    | `agents.list`                                                                                    | `agents.entries` con claves                                                        |
    | `defaultModel` de nivel superior                                                                         | `agents.defaults.model`                                                      |
    | `messages.messagePrefix`                                                                         | `channels.whatsapp.responsePrefix`                                            |
    | `session.maintenance.pruneDays`, `session.resetByType.dm`                                        | `session.maintenance.pruneAfter`, `session.resetByType.direct`               |
    | `tui` de nivel superior                                                                                  | eliminada (el pie de página de la TUI usa el valor predeterminado compacto)                            |
    | `plugins.entries.codex.config.codexDynamicToolsProfile`                                          | eliminada (el servidor de aplicaciones de Codex siempre mantiene como nativas las herramientas de espacio de trabajo nativas de Codex) |
    | `commands.modelsWrite`                                                                           | eliminada (`/models add` está obsoleta)                                       |
    | `agents.defaults/list[].silentReplyRewrite`, `surfaces.*.silentReplyRewrite`                     | eliminadas (el `NO_REPLY` exacto ya no se reescribe como texto alternativo visible)  |
    | `agents.defaults/list[].systemPromptOverride`                                                    | eliminada (OpenClaw controla el prompt del sistema generado)                        |
    | `agents.defaults/list[].embeddedPi`                                                              | `embeddedAgent`                                                              |
    | `agents.defaults/list[].sandbox.perSession`                                                      | `sandbox.scope`                                                              |
    | `agents.defaults.llm`                                                                             | eliminada (se usa `models.providers.<id>.timeoutSeconds` para tiempos de espera de modelos o proveedores lentos, manteniéndolos por debajo del límite máximo de tiempo de espera del agente o la ejecución) |
    | `memorySearch` de nivel superior, `agents.defaults.memorySearch`                                         | `memory.search`                                                             |
    | `agents.entries.*.memorySearch`                                                                     | `agents.entries.*.memory.search`                                               |
    | `memorySearch.provider: "auto"`                                                                  | `"openai"`                                                                    |
    | `memorySearch.store.path` (en cualquier nivel)                                                            | eliminado (los índices de memoria se encuentran en la base de datos de cada agente)                       |
    | `heartbeat` de nivel superior                                                                            | `agents.defaults.heartbeat` / `channels.defaults.heartbeat`                 |
    | identificadores de política de `plugins.openai-codex`                                                                | `plugins.openai`                                                             |
    | `tools.web.x_search.apiKey`                                                                      | `plugins.entries.xai.config.webSearch.apiKey`                               |
    | `session.maintenance.rotateBytes`, `session.parentForkMaxTokens`                                 | eliminado (obsoleto)                                                        |
    | Opciones de ajuste del entorno de ejecución y del canal retiradas en 2026.7                                               | eliminado (se aplican los valores predeterminados de producción integrados)                               |

    <Note>
      Las filas `plugins.entries.voice-call.config.*` anteriores son normalizadas por
      el propio plugin Voice Call en cada carga de configuración, no por `openclaw
      doctor`. El plugin también registra una advertencia de inicio que señala a `openclaw
      doctor --fix`, pero doctor no reescribe actualmente
      `openclaw.json` para estas claves; la propia normalización del plugin es la que
      aplica el cambio en tiempo de ejecución.
    </Note>

    Orientación sobre la cuenta predeterminada para canales con varias cuentas:

    - Si se configuran dos o más entradas `channels.<channel>.accounts` sin `channels.<channel>.defaultAccount` ni `accounts.default`, doctor advierte que el enrutamiento alternativo puede elegir una cuenta inesperada.
    - Si `channels.<channel>.defaultAccount` se establece en un ID de cuenta desconocido, doctor muestra una advertencia y enumera los ID de cuenta configurados.

  </Accordion>
  <Accordion title="2b. Sustituciones del proveedor OpenCode">
    Si se ha añadido manualmente `models.providers.opencode`, `opencode-zen` o `opencode-go`, se sustituye el catálogo integrado de OpenCode de `openclaw/plugin-sdk/llm`. Esto puede forzar a los modelos a usar la API incorrecta o reducir los costes a cero. Doctor muestra una advertencia para que se pueda eliminar la sustitución y restaurar el enrutamiento de API y los costes por modelo.
  </Accordion>
  <Accordion title="2c. Migración del navegador y preparación de Chrome MCP">
    Si la configuración del navegador aún apunta a la ruta eliminada de la extensión de Chrome, doctor la normaliza al modelo actual de conexión de Chrome MCP local al host (`browser.profiles.*.driver: "extension"` → `"existing-session"`; se elimina `browser.relayBindHost`).

    Doctor también audita la ruta de Chrome MCP local al host cuando se usa `defaultProfile: "user"` o un perfil `existing-session` configurado:

    - comprueba si Google Chrome está instalado en el mismo host para los perfiles de conexión automática predeterminados
    - comprueba la versión de Chrome detectada y muestra una advertencia cuando es anterior a Chrome 144
    - recuerda habilitar la depuración remota en la página de inspección del navegador (por ejemplo, `chrome://inspect/#remote-debugging`, `brave://inspect/#remote-debugging` o `edge://inspect/#remote-debugging`)

    Doctor no puede habilitar la opción de Chrome. Chrome MCP local al host sigue requiriendo un navegador basado en Chromium 144+ en el host del Gateway/Node, ejecutándose localmente, con la depuración remota habilitada y la primera solicitud de consentimiento para la conexión aprobada en el navegador.

    La preparación indicada aquí solo cubre los requisitos previos para la conexión local. La sesión existente mantiene los límites actuales de las rutas de Chrome MCP; las rutas avanzadas como `responsebody`, la exportación a PDF, la interceptación de descargas y las acciones por lotes siguen requiriendo un navegador administrado o un perfil CDP sin procesar. Esta comprobación no se aplica a Docker, entornos aislados, navegadores remotos ni otros flujos sin interfaz gráfica, que siguen usando CDP sin procesar.

  </Accordion>
  <Accordion title="2d. Requisitos previos de TLS para OAuth">
    Cuando se configura un perfil OAuth de OpenAI Codex, doctor sondea el punto de conexión de autorización de OpenAI para verificar que la pila TLS local de Node/OpenSSL pueda validar la cadena de certificados. Si el sondeo falla con un error de certificado (por ejemplo, `UNABLE_TO_GET_ISSUER_CERT_LOCALLY`, un certificado caducado o un certificado autofirmado), doctor muestra instrucciones de corrección específicas de la plataforma. En macOS con un Node de Homebrew, la solución suele ser `brew postinstall ca-certificates`. Con `--deep`, el sondeo se ejecuta incluso si el Gateway funciona correctamente.
  </Accordion>
  <Accordion title="2e. Sustituciones del proveedor OAuth de Codex">
    Si anteriormente se añadieron opciones de transporte heredadas de OpenAI en `models.providers.openai-codex`, estas pueden ocultar la ruta integrada del proveedor OAuth de Codex. Doctor muestra una advertencia cuando detecta esas opciones de transporte antiguas junto con OAuth de Codex, para que se pueda eliminar o reescribir la sustitución de transporte obsoleta y restaurar el comportamiento de enrutamiento actual. Los proxies personalizados y las sustituciones únicamente de encabezados siguen siendo compatibles y no activan esta advertencia, pero esas rutas de solicitud definidas por el usuario no son aptas para la selección implícita de Codex.
  </Accordion>
  <Accordion title="2f. Reparación de rutas de Codex">
    Doctor comprueba si hay referencias de modelos `openai-codex/*` heredadas. El enrutamiento del entorno nativo de Codex usa referencias de modelos `openai/*` canónicas, pero el prefijo por sí solo nunca selecciona Codex. Si la política de tiempo de ejecución no está establecida o es `auto`, solo es apta una ruta oficial exacta de HTTPS Platform Responses o ChatGPT Responses sin ninguna sustitución de solicitud definida por el usuario. Consulte [el tiempo de ejecución implícito de agentes de OpenAI](/es/providers/openai#implicit-agent-runtime).

    En el modo `--fix` / `--repair`, doctor reescribe las referencias afectadas del agente predeterminado y de cada agente, incluidos los modelos principales, las alternativas, los modelos de generación de imágenes/vídeos, las sustituciones de heartbeat/subagente/compaction, los hooks, las sustituciones de modelos de canales y el estado obsoleto de rutas de sesión persistido:

    - `openai-codex/gpt-*` se convierte en `openai/gpt-*`.
    - La intención de Codex se traslada a entradas `agentRuntime.id: "codex"` limitadas por proveedor/modelo para las referencias de modelos de agentes reparadas.
    - Se eliminan la configuración obsoleta de tiempo de ejecución del agente completo y las fijaciones persistidas del tiempo de ejecución de la sesión porque la selección del tiempo de ejecución está limitada por proveedor/modelo.
    - La política existente de tiempo de ejecución del proveedor/modelo se conserva, salvo que la referencia del modelo heredado reparada necesite el enrutamiento de Codex para mantener la ruta de autenticación anterior.
    - Las listas existentes de modelos alternativos se conservan con sus entradas heredadas reescritas; las opciones por modelo copiadas se trasladan de la clave heredada a la clave canónica `openai/*`.
    - Los valores persistidos de sesión `modelProvider`/`providerOverride`, `model`/`modelOverride`, los avisos de uso de alternativas y las fijaciones de perfiles de autenticación se reparan en todos los almacenes de sesiones de agentes detectados.
    - Doctor repara por separado las fijaciones obsoletas `agentRuntime.id: "codex-cli"` (un ID de tiempo de ejecución heredado distinto) a `"codex"` en `agents.defaults`, `agents.entries.*` y las entradas de modelos `models.providers.*`.
    - `/codex ...` significa «controlar o vincular una conversación nativa de Codex desde el chat».
    - `/acp ...` o `runtime: "acp"` significa «usar el adaptador externo ACP/acpx».

  </Accordion>
  <Accordion title="2g. Limpieza de rutas de sesión">
    Doctor también examina los almacenes de sesiones de agentes detectados en busca de estados de rutas obsoletos creados automáticamente después de trasladar los modelos configurados o el tiempo de ejecución fuera de una ruta propiedad de un plugin, como Codex.

    `openclaw doctor --fix` puede borrar estados obsoletos creados automáticamente, como fijaciones de modelos `modelOverrideSource: "auto"`, metadatos de modelos de tiempo de ejecución, ID fijados del entorno, vinculaciones de sesiones de CLI y sustituciones automáticas de perfiles de autenticación cuando su ruta propietaria deja de estar configurada. Las elecciones explícitas del usuario o las elecciones heredadas del modelo de sesión se notifican para su revisión manual y no se modifican; cámbielas con `/model ...`, `/new` o restablezca la sesión cuando esa ruta ya no se necesite.

  </Accordion>
  <Accordion title="3. Migraciones de estado heredado (distribución en disco)">
    Doctor puede migrar distribuciones antiguas en disco a la estructura actual:

    - Almacén de sesiones y transcripciones: de `~/.openclaw/sessions/` a `~/.openclaw/agents/<agentId>/sessions/`
    - Directorio del agente: de `~/.openclaw/agent/` a `~/.openclaw/agents/<agentId>/agent/`
    - Estado de autenticación de WhatsApp (Baileys): del `~/.openclaw/credentials/*.json` heredado (excepto `oauth.json`) a `~/.openclaw/credentials/whatsapp/<accountId>/...` (ID de cuenta predeterminado: `default`)
    - Identidad firmada del dispositivo: de `~/.openclaw/identity/device.json` a la fila `device_identities` de `primary` en `state/openclaw.sqlite`; el archivo independiente de autenticación del dispositivo no se modifica

    Estas migraciones se realizan en la medida de lo posible y son idempotentes; doctor emite advertencias cuando deja carpetas heredadas como copias de seguridad. El Gateway/CLI también migra automáticamente al iniciarse las sesiones heredadas y el directorio del agente, para que el historial, la autenticación y los modelos se guarden en la ruta de cada agente sin tener que ejecutar doctor manualmente. La autenticación de WhatsApp se migra intencionadamente solo mediante `openclaw doctor`. La normalización de proveedores/mapas de proveedores de conversación realiza comparaciones mediante igualdad estructural, por lo que las diferencias debidas únicamente al orden de las claves ya no activan cambios `doctor --fix` repetidos sin efecto.

  </Accordion>
  <Accordion title="3a. Migraciones de manifiestos de plugins heredados">
    Doctor examina todos los manifiestos de plugins instalados en busca de claves de capacidades de nivel superior obsoletas (`speechProviders`, `realtimeTranscriptionProviders`, `realtimeVoiceProviders`, `mediaUnderstandingProviders`, `imageGenerationProviders`, `videoGenerationProviders`, `webFetchProviders`, `webSearchProviders`). Cuando las encuentra, ofrece trasladarlas al objeto `contracts` y reescribir el archivo de manifiesto en el mismo lugar. Esta migración es idempotente; si `contracts` ya contiene los mismos valores, la clave heredada se elimina sin duplicar los datos.
  </Accordion>
  <Accordion title="3b. Migraciones del almacén de Cron heredado">
    Doctor también comprueba el almacén de trabajos de Cron heredado (`~/.openclaw/cron/jobs.json`) en busca de formatos de trabajos antiguos antes de importar las filas canónicas a SQLite.

    Las limpiezas actuales de Cron incluyen:

    - `jobId` → `id`
    - `schedule.cron` → `schedule.expr`
    - campos de carga útil de nivel superior (`message`, `model`, `thinking`, ...) → `payload`
    - campos de entrega de nivel superior (`deliver`, `channel`, `to`, `provider`, ...) → `delivery`
    - alias de entrega `provider` de la carga útil → `delivery.channel` explícito
    - trabajos heredados de Webhook alternativo `notify: true` → entrega explícita mediante Webhook a partir del valor sin procesar retirado `cron.webhook` cuando sea válido; los trabajos de anuncio conservan su entrega por chat y reciben `delivery.completionDestination`. Después, doctor elimina la clave de configuración antigua. Sin un Webhook heredado utilizable, se elimina el marcador de nivel superior inerte `notify` para los trabajos sin destino (se conserva la entrega existente, incluidos los anuncios), ya que la entrega en tiempo de ejecución nunca lo lee.

    El Gateway también sanea las filas de Cron malformadas durante la carga para que los trabajos válidos sigan ejecutándose. Las filas malformadas sin procesar se copian a `jobs-quarantine.json`, junto al almacén activo, antes de eliminarlas de `jobs.json`; doctor informa de las filas en cuarentena para que se puedan revisar o reparar manualmente.

    Al iniciarse, el Gateway normaliza la proyección de tiempo de ejecución e ignora el marcador de nivel superior `notify`, pero deja el estado persistido de Cron para que doctor lo repare. Doctor elimina los marcadores inertes de los trabajos sin un destino de migración (`delivery.mode` ninguno/ausente, un destino de Webhook heredado inutilizable o una entrega existente por anuncio/chat), sin modificar la entrega existente, de modo que las ejecuciones repetidas de `doctor --fix` ya no vuelvan a advertir sobre el mismo trabajo.

    En Linux, doctor también muestra una advertencia cuando el crontab del usuario aún invoca el `~/.openclaw/bin/ensure-whatsapp.sh` heredado. Este script local al host no recibe mantenimiento en la versión actual de OpenClaw y puede escribir mensajes `Gateway inactive` falsos en `~/.openclaw/logs/whatsapp-health.log` cuando Cron no puede acceder al bus de usuario de systemd. Elimine la entrada obsoleta del crontab con `crontab -e`; use `openclaw channels status --probe`, `openclaw doctor` y `openclaw gateway status` para las comprobaciones de estado actuales.

  </Accordion>
  <Accordion title="3c. Limpieza de bloqueos de sesión">
    Doctor examina cada directorio de sesiones de agente en busca de archivos de bloqueo de escritura obsoletos que hayan quedado tras el cierre anómalo de una sesión. Por cada archivo de bloqueo encontrado, informa de lo siguiente: la ruta, el PID, si el PID sigue activo, la antigüedad del bloqueo y si se considera obsoleto (PID inactivo, metadatos del propietario con formato incorrecto, antigüedad superior a 30 minutos o un PID activo del que se haya comprobado que pertenece a un proceso ajeno a OpenClaw). En el modo `--fix` / `--repair`, elimina automáticamente los bloqueos cuyos propietarios estén inactivos, huérfanos, reciclados, tengan metadatos incorrectos y antiguos, o no pertenezcan a OpenClaw. Los bloqueos antiguos que sigan perteneciendo a un proceso activo de OpenClaw se notifican, pero se mantienen para que doctor no interrumpa un proceso activo de escritura de transcripciones.
  </Accordion>
  <Accordion title="3d. Reparación de ramas de transcripciones de sesión">
    Doctor examina los archivos JSONL de sesiones de agente en busca de la estructura de rama duplicada creada por el error de reescritura de transcripciones de prompts de 2026.4.24: un turno de usuario abandonado con contexto de ejecución interno de OpenClaw y una rama hermana activa que contiene el mismo prompt visible del usuario. En el modo `--fix` / `--repair`, doctor crea junto al original una copia de seguridad de cada archivo afectado y reescribe la transcripción para que corresponda a la rama activa, de modo que el historial del gateway y los lectores de memoria dejen de ver turnos duplicados.
  </Accordion>
  <Accordion title="4. Comprobaciones de integridad del estado (persistencia de sesiones, enrutamiento y seguridad)">
    El directorio de estado es el centro neurálgico operativo. Si desaparece, se pierden las sesiones, las credenciales, los registros y la configuración, salvo que existan copias de seguridad en otro lugar.

    Doctor comprueba:

    - **Directorio de estado ausente**: advierte de una pérdida catastrófica del estado, solicita volver a crear el directorio y recuerda que no puede recuperar los datos ausentes.
    - **Permisos del directorio de estado**: verifica que se pueda escribir en él; ofrece reparar los permisos (y muestra una sugerencia `chown` cuando detecta que el propietario o el grupo no coinciden).
    - **Directorio de estado sincronizado con la nube en macOS**: advierte cuando el estado se resuelve dentro de iCloud Drive (`~/Library/Mobile Documents/com~apple~CloudDocs/...`) o `~/Library/CloudStorage/...`, porque las rutas respaldadas por sincronización pueden provocar operaciones de E/S más lentas y conflictos entre bloqueos y sincronización.
    - **Directorio de estado en SD o eMMC en Linux**: advierte cuando el estado se resuelve en una fuente de montaje `mmcblk*`, porque las operaciones de E/S aleatorias respaldadas por SD/eMMC pueden ser más lentas y desgastar el soporte con mayor rapidez al escribir sesiones y credenciales.
    - **Directorio de estado volátil en Linux**: advierte cuando el estado se resuelve en `tmpfs` o `ramfs`, porque las sesiones, las credenciales, la configuración y el estado de SQLite (con archivos auxiliares WAL/de diario) desaparecen al reiniciar. Los montajes `overlay` de Docker no se marcan deliberadamente porque sus capas con capacidad de escritura persisten entre reinicios del host mientras se mantenga el contenedor.
    - **Directorios de sesiones ausentes**: `sessions/` y el directorio del almacén de sesiones son necesarios para conservar el historial y evitar fallos de `ENOENT`.
    - **Discordancia de transcripciones**: advierte cuando las entradas de sesiones recientes no tienen sus archivos de transcripción.
    - **Sesión principal «JSONL de 1 línea»**: señala cuando la transcripción principal solo contiene una línea (el historial no se está acumulando).
    - **Varios directorios de estado**: advierte cuando existen varias carpetas `~/.openclaw` en distintos directorios personales o cuando `OPENCLAW_STATE_DIR` apunta a otro lugar (el historial puede dividirse entre instalaciones).
    - **Recordatorio del modo remoto**: si `gateway.mode=remote`, doctor recuerda que debe ejecutarse en el host remoto (el estado reside allí).
    - **Permisos del archivo de configuración**: advierte si `~/.openclaw/openclaw.json` permite la lectura al grupo o a todo el mundo y ofrece restringir los permisos a `600`.

  </Accordion>
  <Accordion title="5. Estado de la autenticación del modelo (caducidad de OAuth)">
    Doctor inspecciona los perfiles de OAuth del almacén de autenticación, advierte cuando los tokens están próximos a caducar o ya han caducado y puede renovarlos cuando es seguro hacerlo. Si el perfil de OAuth/token de Anthropic está obsoleto, sugiere una clave de API de Anthropic o la ruta del token de configuración de Anthropic. Las solicitudes de renovación solo aparecen durante la ejecución interactiva (TTY); `--non-interactive` omite los intentos de renovación.

    Cuando una renovación de OAuth falla de forma permanente (por ejemplo, `refresh_token_reused`, `invalid_grant` o cuando un proveedor indica que es necesario volver a iniciar sesión), doctor informa de que se requiere volver a autenticarse y muestra el comando `openclaw models auth login --provider ...` exacto que debe ejecutarse.

    Doctor también informa de los perfiles de autenticación que no pueden utilizarse temporalmente debido a periodos breves de espera (límites de frecuencia, tiempos de espera o fallos de autenticación) o a desactivaciones más prolongadas (fallos de facturación o crédito).

    Los perfiles antiguos de OAuth de Codex cuyos tokens residen en el llavero de macOS (incorporaciones anteriores al diseño de archivos auxiliares) solo los repara doctor. Ejecute `openclaw doctor --fix` una vez desde un terminal interactivo para migrar directamente los tokens antiguos respaldados por el llavero a `auth-profiles.json`; después, los turnos integrados (Telegram, cron y delegación a subagentes) los resuelven como perfiles canónicos de OAuth de OpenAI.

  </Accordion>
  <Accordion title="6. Validación del modelo de hooks">
    Si se establece `hooks.gmail.model`, doctor valida la referencia del modelo con respecto al catálogo y la lista de permitidos, y advierte cuando no podrá resolverse o no esté permitida.
  </Accordion>
  <Accordion title="7. Reparación de imágenes del sandbox">
    Cuando el sandbox está habilitado, doctor comprueba las imágenes de Docker y ofrece compilar la imagen actual o cambiar a nombres antiguos si esta no está disponible.
  </Accordion>
  <Accordion title="7b. Limpieza de instalaciones de plugins">
    En el modo `openclaw doctor --fix` / `openclaw doctor --repair`, doctor elimina el estado antiguo de preparación de dependencias de plugins generado por OpenClaw: raíces de dependencias generadas obsoletas, directorios antiguos de preparación de instalaciones, residuos locales de paquetes procedentes del código anterior de reparación de dependencias de plugins incluidos y copias npm administradas huérfanas o recuperadas de plugins `@openclaw/*` incluidos que pueden ocultar el manifiesto incluido actual. Doctor también vuelve a enlazar el paquete `openclaw` del host con los plugins npm administrados que declaran `peerDependencies.openclaw`, para que las importaciones locales del paquete en tiempo de ejecución, como `openclaw/plugin-sdk/*`, sigan resolviéndose después de actualizaciones o reparaciones de npm.

    Doctor también puede reinstalar plugins descargables ausentes cuando la configuración hace referencia a ellos, pero el registro local de plugins no puede encontrarlos (`plugins.entries` material, configuración de canales/proveedores/búsquedas y tiempos de ejecución de agentes configurados). Durante las actualizaciones de paquetes, doctor evita reinstalar paquetes de plugins mientras se sustituye el paquete principal; vuelva a ejecutar `openclaw doctor --fix` después de la actualización si un plugin configurado aún necesita recuperarse. Fuera de la excepción de inicio de imágenes de contenedor descrita a continuación, ni el inicio del gateway ni la recarga de la configuración ejecutan reparaciones de paquetes; las instalaciones de plugins siguen siendo operaciones explícitas de doctor, instalación o actualización.

    El inicio del gateway en contenedores tiene una excepción limitada para actualizaciones: cuando `openclaw gateway run` se inicia con una nueva versión de OpenClaw, ejecuta migraciones seguras del estado y la convergencia existente de plugins posterior al núcleo antes de quedar listo, y después registra un punto de control por versión. Este proceso de inicio puede limpiar registros obsoletos de plugins incluidos, reparar enlaces locales de plugins, reinstalar paquetes de plugins configurados cuando la ruta de convergencia lo requiera y comprobar las cargas útiles de los plugins activos. Si el inicio no puede realizar la reparación de forma segura, ejecute una vez la misma imagen con `openclaw doctor --fix` sobre el mismo estado y configuración montados antes de reiniciar el contenedor con normalidad.

  </Accordion>
  <Accordion title="8. Migraciones del servicio Gateway y sugerencias de limpieza">
    Doctor detecta servicios gateway antiguos (launchd/systemd/schtasks) y ofrece eliminarlos e instalar el servicio OpenClaw con el puerto actual del gateway. También puede buscar otros servicios similares a gateway y mostrar sugerencias de limpieza. Los servicios gateway de OpenClaw con nombre de perfil se consideran de primera clase y no se marcan como «adicionales».

    En Linux, si falta el servicio gateway de nivel de usuario, pero existe un servicio gateway de OpenClaw de nivel del sistema, doctor no instala automáticamente un segundo servicio de nivel de usuario. Inspeccione con `openclaw gateway status --deep` o `openclaw doctor --deep` y, después, elimine el duplicado o establezca `OPENCLAW_SERVICE_REPAIR_POLICY=external` cuando un supervisor del sistema gestione el ciclo de vida del gateway.

  </Accordion>
  <Accordion title="8b. Migración de Matrix al iniciar">
    Cuando una cuenta de canal de Matrix tiene pendiente una migración de estado antiguo o existe una migración que puede ejecutarse, doctor (en el modo `--fix` / `--repair`) crea una instantánea previa a la migración y, después, ejecuta los pasos de migración con el máximo esfuerzo posible: la migración del estado antiguo de Matrix y la preparación del estado cifrado antiguo. Ninguno de los pasos es fatal; los errores se registran y el inicio continúa. En el modo de solo lectura (`openclaw doctor` sin `--fix`), esta comprobación se omite por completo.
  </Accordion>
  <Accordion title="8c. Vinculación de dispositivos y divergencias de autenticación">
    Doctor inspecciona el estado de vinculación de dispositivos como parte de la comprobación normal de estado e informa de:

    - solicitudes pendientes de vinculación inicial
    - actualizaciones pendientes de rol o ámbito para dispositivos ya vinculados
    - reparaciones de discordancias de claves públicas en las que el id. del dispositivo sigue coincidiendo, pero la identidad del dispositivo ya no coincide con el registro aprobado
    - registros vinculados sin un token activo para un rol aprobado
    - tokens vinculados cuyos ámbitos se desvían de la referencia de vinculación aprobada
    - entradas locales almacenadas en caché de tokens de dispositivo para el equipo actual anteriores a una rotación del token en el gateway o que contienen metadatos de ámbito obsoletos

    Doctor no aprueba automáticamente las solicitudes de vinculación ni rota automáticamente los tokens de dispositivo. Muestra los pasos siguientes exactos:

    - inspeccionar las solicitudes pendientes con `openclaw devices list`
    - aprobar la solicitud exacta con `openclaw devices approve <requestId>`
    - rotar un token nuevo con `openclaw devices rotate --device <deviceId> --role <role>`
    - eliminar y volver a aprobar un registro obsoleto con `openclaw devices remove <deviceId>`

    Esto distingue la vinculación inicial de las actualizaciones pendientes de roles o ámbitos y de las divergencias causadas por tokens o identidades de dispositivo obsoletos, lo que resuelve el problema habitual de «el dispositivo ya está vinculado, pero se sigue indicando que se requiere vincularlo».

  </Accordion>
  <Accordion title="9. Advertencias de seguridad">
    Doctor solo muestra una nota de seguridad cuando encuentra una advertencia, como un proveedor abierto a mensajes directos sin una lista de permitidos o una política configurada de forma peligrosa. Use `openclaw security audit` para consultar el inventario de seguridad completo.
  </Accordion>
  <Accordion title="10. Permanencia de systemd (Linux)">
    Si se ejecuta como servicio de usuario de systemd, doctor se asegura de que la permanencia esté habilitada para que el gateway siga activo después de cerrar sesión.
  </Accordion>
  <Accordion title="11. Estado del espacio de trabajo (Skills, plugins y TaskFlows)">
    Doctor muestra los problemas y las acciones del agente predeterminado, no el inventario del estado correcto:

    - **Skills**: enumera los nombres de Skills permitidos pero no utilizables; use `openclaw skills check` para consultar los detalles de los requisitos y los recuentos completos.
    - **Plugins**: informa únicamente de los identificadores de plugins con errores; use `openclaw plugins list` para consultar el inventario de plugins cargados, importados, deshabilitados e incluidos.
    - **Advertencias de compatibilidad de plugins**: señala los plugins que tienen problemas de compatibilidad con el tiempo de ejecución actual.
    - **Diagnósticos de plugins**: muestra todas las advertencias o errores emitidos por el registro de plugins durante la carga.
    - **Recuperación de TaskFlow**: muestra los TaskFlows administrados sospechosos que requieren inspección manual o cancelación.
    - **CLI de Claude**: informa únicamente de problemas con el binario, la autenticación, el perfil, el espacio de trabajo o el directorio del proyecto; se omiten los detalles de las comprobaciones correctas.

  </Accordion>
  <Accordion title="11b. Tamaño de los archivos de arranque">
    Doctor comprueba si los archivos de arranque del espacio de trabajo (por ejemplo, `AGENTS.md`, `CLAUDE.md` u otros archivos de contexto inyectados) se acercan al presupuesto de caracteres configurado o lo superan. Informa, por archivo, del número de caracteres sin procesar frente a los inyectados, el porcentaje de truncamiento, la causa del truncamiento (`max/file` o `max/total`) y el total de caracteres inyectados como fracción del presupuesto total. Cuando los archivos están truncados o cerca del límite, doctor muestra sugerencias para ajustar `agents.defaults.bootstrapMaxChars` y `agents.defaults.bootstrapTotalMaxChars`.
  </Accordion>
  <Accordion title="11c. Autocompletado del shell">
    Doctor comprueba si el autocompletado con el tabulador está instalado para el shell actual (zsh, bash, fish o PowerShell):

    - Si el perfil del shell usa un patrón de completado dinámico lento (`source <(openclaw completion ...)`), doctor lo actualiza a la variante más rápida con archivo en caché.
    - Si el completado está configurado en el perfil, pero falta el archivo de caché, doctor regenera la caché automáticamente.
    - Si no hay ningún completado configurado, doctor solicita instalarlo (solo en modo interactivo; se omite con `--non-interactive`).

    Ejecute `openclaw completion --write-state` para regenerar la caché manualmente.

  </Accordion>
  <Accordion title="11d. Limpieza de plugins de canal obsoletos">
    Cuando `openclaw doctor --fix` elimina un plugin de canal ausente, también elimina la configuración huérfana con ámbito de canal que hacía referencia a ese plugin: entradas `channels.<id>`, destinos de Heartbeat que nombraban el canal y anulaciones `agents.*.models["<channel>/*"]`. Esto evita bucles de arranque del Gateway en los que el runtime del canal ya no existe, pero la configuración aún solicita al Gateway que se vincule a él.
  </Accordion>
  <Accordion title="12. Comprobaciones de autenticación del Gateway (token local)">
    Doctor comprueba que la autenticación mediante token del Gateway local esté lista.

    - Si el modo de token necesita un token y no existe ninguna fuente de tokens, doctor ofrece generar uno.
    - Si `gateway.auth.token` está gestionado mediante SecretRef, pero no está disponible, doctor muestra una advertencia y no lo sobrescribe con texto sin formato.
    - `openclaw doctor --generate-gateway-token` fuerza la generación solo cuando no hay ningún SecretRef de token configurado.

  </Accordion>
  <Accordion title="12b. Reparaciones de solo lectura compatibles con SecretRef">
    Algunos flujos de reparación necesitan inspeccionar las credenciales configuradas sin debilitar el comportamiento de fallo rápido del runtime.

    - `openclaw doctor --fix` usa el mismo modelo de resumen de SecretRef de solo lectura que los comandos de la familia de estado para realizar reparaciones de configuración específicas.
    - Ejemplo: la reparación de `allowFrom` / `groupAllowFrom` `@username` de Telegram intenta usar las credenciales configuradas del bot cuando están disponibles.
    - Si el token del bot de Telegram está configurado mediante SecretRef, pero no está disponible en la ruta del comando actual, doctor informa de que la credencial está configurada, pero no disponible, y omite la resolución automática en lugar de fallar o informar erróneamente de que falta el token.

  </Accordion>
  <Accordion title="13. Comprobación de estado y reinicio del Gateway">
    Doctor ejecuta una comprobación de estado y ofrece reiniciar el Gateway cuando parece no estar en buen estado.
  </Accordion>
  <Accordion title="13b. Disponibilidad de la búsqueda en memoria">
    Doctor comprueba si el proveedor de embeddings configurado para la búsqueda en memoria está listo para el agente predeterminado. El comportamiento depende del backend y del proveedor configurados:

    - **Backend QMD**: comprueba si el binario `qmd` está disponible y puede iniciarse. De no ser así, muestra instrucciones para solucionarlo, incluido `npm install -g @tobilu/qmd` (o el equivalente de Bun), y una opción para indicar manualmente la ruta del binario.
    - **Proveedor local explícito**: comprueba si existe un archivo de modelo local o una URL reconocida de un modelo remoto o descargable. Si falta, sugiere cambiar a un proveedor remoto.
    - **Proveedor remoto explícito** (`openai`, `voyage`, etc.): verifica que haya una clave de API en el entorno o en el almacén de autenticación. Si falta, muestra indicaciones prácticas para solucionarlo.
    - **Proveedor automático heredado**: trata `memorySearch.provider: "auto"` como OpenAI, comprueba la disponibilidad de OpenAI y `doctor --fix` lo reescribe como `provider: "openai"`.

    Cuando hay disponible un resultado almacenado en caché del sondeo del Gateway (el Gateway estaba en buen estado en el momento de la comprobación), doctor compara su resultado con la configuración visible desde la CLI y señala cualquier discrepancia. Doctor no inicia un nuevo sondeo de embeddings en la ruta predeterminada; use el comando de estado detallado de la memoria cuando desee una comprobación en vivo del proveedor.

    Use `openclaw memory status --deep` para verificar la disponibilidad de los embeddings durante el runtime.

  </Accordion>
  <Accordion title="14. Advertencias de estado de los canales">
    Si el Gateway está en buen estado, doctor ejecuta un sondeo del estado de los canales e informa de las advertencias con las soluciones sugeridas.
  </Accordion>
  <Accordion title="15. Auditoría y reparación de la configuración del supervisor">
    Doctor comprueba si faltan valores predeterminados o están desactualizados en la configuración del supervisor instalada (launchd/systemd/schtasks), por ejemplo, las dependencias network-online de systemd y el retraso de reinicio. Cuando encuentra una discrepancia, recomienda una actualización y puede reescribir el archivo de servicio o la tarea con los valores predeterminados actuales.

    Notas:

    - `openclaw doctor` solicita confirmación antes de reescribir la configuración del supervisor.
    - `openclaw doctor --yes` acepta las solicitudes de reparación predeterminadas.
    - `openclaw doctor --fix` aplica las correcciones recomendadas sin solicitar confirmación (`--repair` es un alias).
    - `openclaw doctor --fix --force` sobrescribe las configuraciones personalizadas del supervisor.
    - `OPENCLAW_SERVICE_REPAIR_POLICY=external` mantiene doctor en modo de solo lectura para el ciclo de vida del servicio del Gateway. Sigue informando del estado del servicio y ejecutando reparaciones no relacionadas con el servicio, pero omite la instalación, el inicio, el reinicio y el arranque inicial del servicio, las reescrituras de la configuración del supervisor y la limpieza de servicios heredados, porque un supervisor externo controla ese ciclo de vida.
    - En Linux, doctor no reescribe los metadatos del comando o del punto de entrada mientras la unidad systemd correspondiente del Gateway esté activa. También ignora las unidades adicionales inactivas, no heredadas y similares al Gateway durante el análisis de servicios duplicados, para que los archivos de servicios complementarios no generen avisos innecesarios de limpieza.
    - Si la autenticación mediante token requiere un token y `gateway.auth.token` está gestionado mediante SecretRef, la instalación o reparación del servicio por parte de doctor valida el SecretRef, pero no conserva los valores resueltos del token en texto sin formato en los metadatos del entorno del servicio del supervisor.
    - Doctor detecta los valores gestionados de `.env` o respaldados por SecretRef que las instalaciones anteriores de LaunchAgent, systemd o las tareas programadas de Windows incorporaron directamente en el entorno del servicio, y reescribe los metadatos del servicio para que esos valores se carguen desde la fuente del runtime en lugar de la definición del supervisor.
    - Doctor detecta cuando el comando del servicio sigue fijado a un `--port` anterior después de que cambie `gateway.port`, y reescribe los metadatos del servicio con el puerto actual.
    - Si la autenticación mediante token requiere un token y el SecretRef del token configurado no se puede resolver, doctor bloquea la ruta de instalación o reparación y proporciona instrucciones prácticas.
    - Si tanto `gateway.auth.token` como `gateway.auth.password` están configurados y `gateway.auth.mode` no está definido, doctor bloquea la instalación o reparación hasta que se establezca el modo explícitamente.
    - Para las unidades systemd de usuario de Linux, las comprobaciones de discrepancias de tokens de doctor incluyen las fuentes `Environment=` y `EnvironmentFile=` al comparar los metadatos de autenticación del servicio.
    - Las reparaciones de servicios de doctor se niegan a reescribir, detener o reiniciar un servicio del Gateway desde un binario anterior de OpenClaw cuando la configuración fue escrita por última vez por una versión más reciente. Consulte [Solución de problemas del Gateway](/es/gateway/troubleshooting#split-brain-installs-and-newer-config-guard).
    - Siempre se puede forzar una reescritura completa mediante `openclaw gateway install --force`.

  </Accordion>
  <Accordion title="16. Diagnósticos del runtime y del puerto del Gateway">
    Doctor inspecciona el runtime del servicio (PID, estado de la última salida) y advierte cuando el servicio está instalado, pero en realidad no se está ejecutando. También comprueba si hay conflictos en el puerto del Gateway (valor predeterminado: `18789`) e informa de las causas probables (el Gateway ya está en ejecución o existe un túnel SSH).
  </Accordion>
  <Accordion title="17. Prácticas recomendadas para el runtime del Gateway">
    Doctor advierte cuando el servicio del Gateway se ejecuta en Bun o mediante una ruta de Node gestionada por versiones (`nvm`, `fnm`, `volta`, `asdf`, etc.). Bun no puede abrir el almacén de estado `node:sqlite` de OpenClaw, por lo que las reparaciones migran los servicios Bun heredados a Node. Las rutas de los gestores de versiones pueden dejar de funcionar después de las actualizaciones porque el servicio no carga la inicialización del shell. Doctor ofrece migrar a una instalación de Node del sistema cuando está disponible (Homebrew/apt/choco).

    Los LaunchAgents de macOS recién instalados o reparados usan una PATH canónica del sistema (`/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin`) en lugar de copiar la PATH del shell interactivo, de modo que los binarios del sistema gestionados por Homebrew permanezcan disponibles, mientras que Volta, asdf, fnm, pnpm y otros directorios de gestores de versiones no cambien el Node que resuelven los procesos secundarios. Los servicios de Linux siguen conservando raíces de entorno explícitas (`NVM_DIR`, `FNM_DIR`, `VOLTA_HOME`, `ASDF_DATA_DIR`, `BUN_INSTALL`, `PNPM_HOME`) y directorios estables de binarios del usuario, pero los directorios alternativos inferidos de gestores de versiones solo se escriben en la PATH del servicio cuando existen en el disco.

  </Accordion>
  <Accordion title="18. Escritura de la configuración y metadatos del asistente">
    Doctor conserva todos los cambios de configuración y añade metadatos del asistente para registrar la ejecución de doctor.
  </Accordion>
  <Accordion title="19. Consejos para el espacio de trabajo (copia de seguridad y sistema de memoria)">
    Doctor sugiere un sistema de memoria para el espacio de trabajo cuando falta y muestra un consejo de copia de seguridad si el espacio de trabajo aún no está bajo el control de git.

    Consulte [/concepts/agent-workspace](/es/concepts/agent-workspace) para obtener una guía completa sobre la estructura del espacio de trabajo y la copia de seguridad con git (se recomienda un repositorio privado de GitHub o GitLab).

  </Accordion>
</AccordionGroup>

## Temas relacionados

- [Manual de operaciones del Gateway](/es/gateway)
- [Solución de problemas del Gateway](/es/gateway/troubleshooting)
