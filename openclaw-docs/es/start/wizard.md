---
read_when:
    - Ejecutar o configurar la incorporación mediante la CLI
    - Configuración de una máquina nueva
sidebarTitle: 'Onboarding: CLI'
summary: 'Incorporación mediante la CLI: verificar la inferencia y, a continuación, dejar la configuración restante en manos de OpenClaw'
title: Incorporación (CLI)
x-i18n:
    generated_at: "2026-07-26T05:30:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 150adfac1424b42d66fa3035339082574cc631ce0dc3db09ad32376ef139bf1c
    source_path: start/wizard.md
    workflow: 16
---

```bash
openclaw onboard
```

La incorporación mediante la CLI es la ruta de configuración recomendada desde la terminal en macOS, Linux y
Windows (nativo o WSL2). De forma predeterminada, detecta el acceso a la IA ya disponible en
el equipo, lo verifica con una finalización real e inicia OpenClaw para
configurar el espacio de trabajo, el Gateway y las funciones opcionales. `openclaw setup` ejecuta el mismo flujo ([Configuración](/es/cli/setup) describe
la variante `--baseline` que solo configura). Los usuarios de escritorio de Windows también pueden empezar
desde [Windows Hub](/es/platforms/windows).

La incorporación guiada establece primero la inferencia. Detecta el acceso disponible a la IA,
requiere una finalización real y solo entonces inicia [OpenClaw](/es/cli/openclaw)
para configurar el resto de OpenClaw. Elegir **Omitir por ahora** cierra la incorporación
sin iniciar OpenClaw.

El asistente clásico sigue disponible para proveedores personalizados, la configuración de un Gateway
remoto, la vinculación de canales, los controles del daemon, las habilidades y las importaciones. Ejecútelo explícitamente
con `openclaw onboard --classic`; el selector de inferencia guiada no delega
en él. Una vez superada la inferencia, OpenClaw puede usar `open channel wizard for
<channel>` para derivar la configuración de canales que requiere secretos a un asistente de terminal con datos ocultos.
Para cambiar el proveedor del modelo o su autenticación, salga de OpenClaw y ejecute
`openclaw onboard`; OpenClaw no abre los flujos guiados ni clásicos de proveedores.

<Info>
La forma más rápida de iniciar el primer chat: complete la configuración guiada, ejecute `openclaw dashboard` y converse en
el navegador mediante la interfaz de control. Documentación: [Panel de control](/es/web/dashboard).
</Info>

## Configuración regional

El asistente localiza el texto fijo de incorporación. Usa el primer valor no vacío de
`OPENCLAW_LOCALE`, `LC_ALL`, `LC_MESSAGES` y `LANG`, en ese orden, y después
recurre al inglés. Configuraciones regionales compatibles: `en`, `zh-CN`, `zh-TW`.

```bash
OPENCLAW_LOCALE=zh-CN openclaw onboard
OPENCLAW_LOCALE=en openclaw onboard # Sustitución explícita por inglés
```

Los nombres de productos, comandos, claves de configuración, URL, identificadores de proveedores, identificadores de modelos y
etiquetas de plugins o canales permanecen en inglés independientemente de la configuración regional.

Para volver a configurar posteriormente los ajustes no relacionados con la inferencia:

```bash
openclaw configure
openclaw agents add <name>
```

<Note>
`--json` no implica el modo no interactivo. Para scripts, use `--non-interactive` (consulte [Automatización de la CLI](/es/start/wizard-cli-automation)).
</Note>

<Tip>
El asistente clásico incluye un paso de búsqueda web donde se puede elegir un proveedor: Brave,
DuckDuckGo, Exa, Firecrawl, Gemini, Grok, Kimi, MiniMax Search, Ollama Web
Search, Perplexity, SearXNG o Tavily. Algunos necesitan una clave de API; otros
no requieren claves. Configure esto más adelante con `openclaw configure --section web`. Documentación:
[Herramientas web](/es/tools/web).
</Tip>

## Flujo guiado predeterminado

El comando `openclaw onboard` sin opciones sigue esta ruta:

1. Acepte el aviso de seguridad.
2. Detecte los modelos configurados, las variables de entorno con claves de API, las CLI
   locales de IA compatibles y los modelos con capacidad para usar herramientas ya instalados en servidores Ollama o LM
   Studio accesibles desde el host del Gateway. Esta revisión de solo lectura nunca descarga un
   modelo. También se informa de las instalaciones de Gemini CLI, Antigravity, Pi y OpenCode
   cuando no pueden servir como ruta de inferencia reutilizable para la configuración guiada.
   Gemini y Antigravity no pueden aplicar la prueba sin herramientas; Pi y OpenCode
   son entornos completos de agentes, no rutas de inferencia para la configuración.
3. Pruebe el primer candidato detectado con una finalización real. Si falla, muestre el
   motivo y continúe con el siguiente candidato utilizable.
4. Si se agotan los resultados de la detección, elija OpenAI, Anthropic, xAI (Grok), Google u
   OpenRouter, o elija **Más…** para consultar los proveedores restantes. Las
   regiones, los planes y los métodos compatibles mediante navegador, dispositivo, clave de API o token de cada proveedor
   aparecen en un segundo menú y se prueban con la misma finalización real.
   Elija **Omitir por ahora** para salir sin iniciar OpenClaw.
5. Guarde únicamente la ruta del modelo verificada y cualquier estado de credenciales o plugins que
   requiera. Los ajustes del espacio de trabajo y del Gateway permanecen intactos.
6. Inicie OpenClaw con el modelo verificado para que pueda configurar el espacio de trabajo,
   el Gateway, los canales, los agentes, los plugins y el resto de la configuración opcional.

Al volver a ejecutar el comando en una instalación configurada, primero se prueba el modelo
predeterminado actual, por lo que el flujo guiado sirve como proceso de verificación y reparación. Una comprobación
fallida nunca sustituye automáticamente el modelo configurado; la incorporación se detiene y
pregunta cómo continuar. Ejecute `openclaw channels add` o `openclaw configure` para
realizar posteriormente adiciones no relacionadas con la inferencia; use `openclaw onboard` para cambiar
el proveedor o la ruta de autenticación.

## Asistente clásico: inicio rápido frente a avanzado

Ejecute `openclaw onboard --classic` para abrir el asistente completo. Comienza con una
elección entre **Inicio rápido** (valores predeterminados) y **Avanzado** (control total). Pase
`--flow quickstart` o `--flow advanced` (alias `manual`) para seleccionar el flujo
clásico y omitir esa pregunta.

<Tabs>
  <Tab title="Inicio rápido (valores predeterminados)">
    - Gateway local, enlace a la interfaz de bucle invertido
    - Espacio de trabajo predeterminado (o espacio de trabajo existente)
    - Puerto del Gateway **18789**
    - Autenticación del Gateway mediante **token** (generado automáticamente, incluso en la interfaz de bucle invertido)
    - Política de herramientas: `tools.profile: "coding"` para configuraciones nuevas (se conserva cualquier perfil explícito existente)
    - Sesiones de mensajes directos: la incorporación conserva cualquier `session.dmScope` explícito y, de lo contrario, lo deja sin definir, por lo que el valor predeterminado `"main"` mantiene todos los mensajes directos de todos los canales en la sesión principal continua del agente, el valor predeterminado para un agente personal. Para bandejas de entrada compartidas o multiusuario, use `"per-channel-peer"`; `openclaw security audit` recomienda el aislamiento cuando detecta tráfico multiusuario de mensajes directos. Detalles: [Referencia de configuración de la CLI](/es/start/wizard-cli-reference#outputs-and-internals)
    - Exposición mediante Tailscale **Desactivada**
    - Los mensajes directos de Telegram y WhatsApp usan de forma predeterminada una **lista de permitidos**: Telegram solicita un identificador numérico de usuario de Telegram y WhatsApp solicita un número de teléfono

  </Tab>
  <Tab title="Avanzado (control total)">
    - Muestra todos los pasos: modo, espacio de trabajo, Gateway, canales, daemon y habilidades

  </Tab>
</Tabs>

El modo remoto (`--mode remote`) siempre usa el flujo avanzado; únicamente
configura este equipo para conectarse a un Gateway ubicado en otro lugar y nunca instala
ni modifica nada en el host remoto.

## Qué configura la incorporación clásica

El modo local (predeterminado) recorre estos pasos:

1. **Modelo/autenticación**: elija un flujo de autenticación del proveedor (clave de API, OAuth o
   autenticación manual específica del proveedor), incluido un proveedor personalizado
   (compatible con OpenAI, compatible con OpenAI Responses, compatible con Anthropic o
   detección automática desconocida). Elija un modelo predeterminado.
   Una configuración nueva con clave de API de OpenAI usa de forma predeterminada `openai/gpt-5.6` (el identificador
   simple de la API directa se resuelve como Sol); una configuración nueva de ChatGPT/Codex usa de forma predeterminada
   `openai/gpt-5.6-sol`. Al volver a ejecutar la configuración, se conserva cualquier modelo explícito existente,
   incluido `openai/gpt-5.5`. Seleccione `openai/gpt-5.5` explícitamente si la
   cuenta no ofrece GPT-5.6.
   Nota de seguridad: si este agente va a ejecutar herramientas o procesar contenido de
   webhooks o hooks, conviene usar el modelo de última generación más potente disponible y mantener
   una política de herramientas estricta; los niveles más débiles o antiguos son más vulnerables a la inyección de instrucciones.
   Para ejecuciones no interactivas, `--secret-input-mode ref` guarda referencias respaldadas por variables de entorno
   en lugar de valores de claves de API en texto sin formato; la variable de entorno referenciada ya debe
   estar definida o la incorporación falla de inmediato. El modo interactivo de referencia de secretos puede
   apuntar a una variable de entorno o a una referencia de proveedor configurada (`file` o
   `exec`), con una comprobación preliminar rápida antes de guardar. Tras configurar el modelo y la autenticación,
   el asistente ofrece una prueba opcional de finalización en directo; si falla, se puede volver una vez a
   la configuración del modelo y la autenticación, o ignorar el error sin bloquear el resto del
   asistente clásico. Ignorarlo no desbloquea OpenClaw; la configuración conversacional
   sigue requiriendo que se supere la comprobación de inferencia.
2. **Espacio de trabajo**: directorio para los archivos del agente (valor predeterminado: `~/.openclaw/workspace`). Crea los archivos iniciales.
3. **Gateway**: puerto, dirección de enlace, modo de autenticación y exposición mediante Tailscale. En
   el modo interactivo con token, elija guardar el token como texto sin formato (valor predeterminado) u opte
   por una SecretRef. Ruta no interactiva con SecretRef: `--gateway-token-ref-env <ENV_VAR>`.
4. **Canales**: canales de chat integrados y de plugins oficiales, incluidos
   Discord, Feishu, Google Chat, iMessage, Mattermost, Microsoft Teams,
   QQ Bot, Signal, Slack, Telegram, WhatsApp y otros.
5. **Daemon**: instala un LaunchAgent (macOS), una unidad de usuario de systemd
   (Linux/WSL2) o una tarea programada nativa de Windows con una alternativa por usuario
   en la carpeta Startup.
   Si se requiere autenticación mediante token y `gateway.auth.token` está administrado mediante SecretRef,
   la instalación del daemon lo valida, pero no guarda un token resuelto en
   los metadatos del entorno de servicio del supervisor; una SecretRef sin resolver bloquea
   la instalación y muestra instrucciones. Si `gateway.auth.token` y
   `gateway.auth.password` están definidos mientras `gateway.auth.mode` no lo está, la instalación
   queda bloqueada hasta que se defina el modo explícitamente.
6. **Comprobación de estado**: inicia el Gateway y verifica que sea accesible.
7. **Habilidades**: instala las habilidades recomendadas y sus dependencias opcionales.

<Note>
Volver a ejecutar la incorporación **no** elimina nada, salvo que se elija explícitamente
**Restablecer** (o se pase `--reset`). La opción de la CLI `--reset` incluye de forma predeterminada la configuración, las credenciales
y las sesiones; use `--reset-scope full` para eliminar también el espacio de trabajo. Si la
configuración no es válida o contiene claves heredadas, la incorporación solicita ejecutar primero
`openclaw doctor`.
</Note>

`--flow import` ejecuta un flujo de migración detectado (por ejemplo, Hermes) en el
asistente clásico en lugar de una configuración nueva; consulte [Migrar](/es/cli/migrate) y las guías de migración en
[Instalación](/es/install/migrating-hermes). `openclaw onboard --modern` es un
alias de compatibilidad de [OpenClaw](/es/cli/openclaw). Usa la misma
barrera de inferencia que `openclaw setup`: la inferencia verificada inicia el
asistente, mientras que un fallo interactivo devuelve a la configuración guiada de inferencia.

## Añadir otro agente

Use `openclaw agents add <name>` para crear un agente independiente con su propio
espacio de trabajo, sesiones y perfiles de autenticación. Ejecutarlo sin `--workspace` inicia
un flujo interactivo para definir el nombre, el espacio de trabajo, la autenticación, los canales y las vinculaciones; no es
el asistente completo `openclaw onboard`.

Qué establece:

- `agents.entries.*.name`
- `agents.entries.*.workspace`
- `agents.entries.*.agentDir`

Notas:

- Espacio de trabajo predeterminado: `~/.openclaw/workspace-<agentId>` (o dentro de
  `agents.defaults.workspace` si está definido).
- Añada `bindings` para dirigir los mensajes entrantes a este agente (la incorporación puede hacerlo automáticamente).
- Opciones no interactivas: `--model`, `--agent-dir`, `--bind`, `--non-interactive`.

## Referencia completa

Para conocer en detalle el comportamiento paso a paso y los resultados de configuración, consulte la
[Referencia de configuración de la CLI](/es/start/wizard-cli-reference).
Para ver ejemplos no interactivos, consulte [Automatización de la CLI](/es/start/wizard-cli-automation).
Para consultar la referencia completa de opciones, consulte [`openclaw onboard`](/es/cli/onboard).

## Documentación relacionada

- Referencia de comandos de la CLI: [`openclaw onboard`](/es/cli/onboard)
- Descripción general de la incorporación: [Descripción general de la incorporación](/es/start/onboarding-overview)
- Incorporación en la aplicación para macOS: [Incorporación](/es/start/onboarding)
- Ritual de primera ejecución del agente: [Inicialización del agente](/es/start/bootstrapping)
