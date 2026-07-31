---
read_when:
    - Quieres migrar desde Hermes u otro sistema de agentes a OpenClaw
    - Está añadiendo un proveedor de migración propiedad del plugin
summary: Referencia de la CLI para `openclaw migrate` (importar el estado desde otro sistema de agentes)
title: Migrar
x-i18n:
    generated_at: "2026-07-26T05:08:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f492535019f8a69706ff918462ba74cf5d26e733d2e4e9493b3c76bd77f2584d
    source_path: cli/migrate.md
    workflow: 16
---

# `openclaw migrate`

Importa el estado desde otro sistema de agentes mediante un proveedor de migración propiedad de un plugin. Los proveedores incluidos abarcan Claude, Codex CLI y [Hermes](/es/install/migrating-hermes); los plugins pueden registrar proveedores adicionales.

<Tip>
Para consultar guías orientadas al usuario, véanse [Migración desde Claude](/es/install/migrating-claude) y [Migración desde Hermes](/es/install/migrating-hermes). El [centro de migración](/es/install/migrating) enumera todas las rutas.
</Tip>

## Comandos

```bash
openclaw migrate list
openclaw migrate claude --dry-run
openclaw migrate codex --dry-run
openclaw migrate codex --skill gog-vault77-google-workspace
openclaw migrate codex --plugin google-calendar --dry-run
openclaw migrate codex --plugin google-calendar --verify-plugin-apps --dry-run
openclaw migrate hermes --dry-run
openclaw migrate hermes
openclaw migrate apply codex --yes --skill gog-vault77-google-workspace
openclaw migrate apply codex --yes --plugin google-calendar
openclaw migrate apply codex --yes
openclaw migrate apply claude --yes
openclaw migrate apply hermes --yes
openclaw migrate apply hermes --include-secrets --yes
openclaw onboard --flow import
openclaw onboard --import-from claude --import-source ~/.claude
openclaw onboard --import-from hermes --import-source ~/.hermes
```

Ejecutar `openclaw migrate <provider>` sin otras marcas planifica, muestra una vista previa y, en una TTY, solicita confirmación antes de aplicar. `openclaw migrate plan <provider>` y `openclaw migrate apply <provider>` separan la vista previa y la aplicación en subcomandos distintos con las mismas marcas.

<ParamField path="<provider>" type="string">
  Nombre de un proveedor de migración registrado, por ejemplo, `hermes`. Ejecute `openclaw migrate list` para ver los proveedores instalados.
</ParamField>
<ParamField path="--dry-run" type="boolean">
  Genera el plan y sale sin cambiar el estado.
</ParamField>
<ParamField path="--from <path>" type="string">
  Anula el directorio de estado de origen. Hermes sigue `$HERMES_HOME` y el perfil activo, y después utiliza el valor predeterminado de la plataforma (`~/.hermes` o `%LOCALAPPDATA%\hermes`). El valor predeterminado de Codex es `~/.codex` (o `$CODEX_HOME`) y el de Claude es `~/.claude`.
</ParamField>
<ParamField path="--include-secrets" type="boolean">
  Importa las credenciales compatibles sin solicitar confirmación. La aplicación interactiva pregunta antes de importar las credenciales de autenticación detectadas, con «sí» seleccionado de forma predeterminada; el uso no interactivo de `--yes` requiere `--include-secrets` para importarlas.
</ParamField>
<ParamField path="--no-auth-credentials" type="boolean">
  Omite la importación de credenciales de autenticación, incluida la solicitud interactiva.
</ParamField>
<ParamField path="--overwrite" type="boolean">
  Permite que la aplicación sustituya destinos existentes cuando el plan informa de conflictos.
</ParamField>
<ParamField path="--yes" type="boolean">
  Omite la solicitud de confirmación. Es obligatorio en el modo no interactivo.
</ParamField>
<ParamField path="--skill <name>" type="string">
  Selecciona un elemento de copia de skill por el nombre de la skill o el identificador del elemento. Repita la marca para migrar varias skills. Cuando se omite, las migraciones interactivas de Codex muestran un selector de casillas y las migraciones no interactivas conservan todas las skills planificadas.
</ParamField>
<ParamField path="--plugin <name>" type="string">
  Selecciona un elemento de instalación de plugin de Codex por el nombre del plugin o el identificador del elemento. Repita la marca para migrar varios plugins de Codex. Cuando se omite, las migraciones interactivas de Codex muestran un selector de casillas de plugins nativos de Codex y las migraciones no interactivas conservan todos los plugins planificados. Solo se aplica a los plugins de Codex `openai-curated` instalados desde el código fuente que detecta el inventario del servidor de aplicaciones de Codex.
</ParamField>
<ParamField path="--verify-plugin-apps" type="boolean">
  Solo para Codex. Fuerza un recorrido `app/list` nuevo del servidor de aplicaciones de Codex de origen antes de planificar la activación de plugins nativos. Está desactivado de forma predeterminada para que la planificación de la migración sea rápida.
</ParamField>
<ParamField path="--backup-output <path>" type="string">
  Ruta o directorio del archivo de copia de seguridad anterior a la migración. Se transfiere a `openclaw backup create`.
</ParamField>
<ParamField path="--no-backup" type="boolean">
  Omite la copia de seguridad previa a la aplicación. Requiere `--force` cuando existe estado local de OpenClaw.
</ParamField>
<ParamField path="--force" type="boolean">
  Es obligatorio junto con `--no-backup` cuando, de lo contrario, la aplicación se negaría a omitir la copia de seguridad.
</ParamField>
<ParamField path="--json" type="boolean">
  Imprime el plan o el resultado de la aplicación como JSON. Con `--json` y sin `--yes`, la aplicación imprime el plan y no modifica el estado.
</ParamField>

## Modelo de seguridad

`openclaw migrate` prioriza la vista previa.

<AccordionGroup>
  <Accordion title="Vista previa antes de aplicar">
    El proveedor devuelve un plan detallado por elementos antes de que cambie nada, incluidos los conflictos y los elementos omitidos y confidenciales. Los planes JSON, la salida de la aplicación y los informes de migración censuran las claves anidadas que parecen contener secretos, como claves de API, tokens, encabezados de autorización, cookies y contraseñas.

    `openclaw migrate apply <provider>` muestra una vista previa del plan y solicita confirmación antes de cambiar el estado, salvo que se establezca `--yes`. En el modo no interactivo, la aplicación requiere `--yes`.

  </Accordion>
  <Accordion title="Copias de seguridad">
    La aplicación crea y verifica una copia de seguridad de OpenClaw antes de aplicar la migración. Si todavía no existe estado local de OpenClaw, se omite el paso de copia de seguridad y la migración continúa. Para omitir una copia de seguridad cuando existe estado, indique tanto `--no-backup` como `--force`.
  </Accordion>
  <Accordion title="Conflictos">
    La aplicación se niega a continuar cuando el plan tiene conflictos. Revise el plan y vuelva a ejecutar el comando con `--overwrite` si la sustitución de los destinos existentes es intencionada. Los proveedores pueden seguir creando copias de seguridad por elemento de los archivos sobrescritos en el directorio de informes de migración.
  </Accordion>
  <Accordion title="Secretos">
    La aplicación interactiva pregunta si se deben importar las credenciales de autenticación detectadas, con «sí» seleccionado de forma predeterminada. Utilice `--no-auth-credentials` para omitirlas o `--include-secrets` para importar credenciales sin supervisión con `--yes`.
  </Accordion>
</AccordionGroup>

## Proveedor de Claude

El proveedor de Claude incluido detecta de forma predeterminada el estado de Claude Code en `~/.claude`. Utilice `--from <path>` para importar un directorio principal o una raíz de proyecto específicos de Claude Code.

<Tip>
Para consultar una guía orientada al usuario, véase [Migración desde Claude](/es/install/migrating-claude).
</Tip>

### Qué importa Claude

- Markdown de memoria automática de Claude Code de `~/.claude/projects/*/memory` y un
  `autoMemoryDirectory` configurado por el usuario, copiado en
  `memory/imports/claude-code/` para su recuperación indexada.
- `CLAUDE.md` y `.claude/CLAUDE.md` del proyecto en el espacio de trabajo del agente de OpenClaw (`AGENTS.md`).
- `~/.claude/CLAUDE.md` del usuario anexado a `USER.md` del espacio de trabajo.
- Definiciones de servidores MCP de `.mcp.json` del proyecto, `~/.claude.json` de Claude Code (incluidas sus entradas por proyecto) y `claude_desktop_config.json` de Claude Desktop.
- Directorios de skills de Claude que incluyen `SKILL.md` (`~/.claude/skills` del usuario y `.claude/skills` del proyecto).
- Archivos Markdown de comandos de Claude (`~/.claude/commands` del usuario y `.claude/commands` del proyecto) convertidos en skills de OpenClaw únicamente con invocación manual.

### Estado archivado y de revisión manual

Los hooks, permisos, valores predeterminados del entorno, `CLAUDE.local.md` y `.claude/rules` del proyecto, los directorios `agents/` del usuario y del proyecto, y el historial del proyecto (`projects`, `cache` y `plans` en `~/.claude`) de Claude se conservan en el informe de migración o se notifican como elementos de revisión manual. OpenClaw no ejecuta hooks, copia listas amplias de elementos permitidos ni importa automáticamente el estado de credenciales de OAuth/Desktop.

## Proveedor de Codex

El proveedor de Codex incluido detecta de forma predeterminada el estado de Codex CLI en `~/.codex`, o en `CODEX_HOME` cuando se establece esa variable de entorno. Utilice `--from <path>` para inventariar un directorio principal específico de Codex.

Utilice este proveedor al migrar al entorno de ejecución de Codex de OpenClaw si se desea trasladar deliberadamente recursos personales útiles de Codex CLI. Los inicios locales del servidor de aplicaciones de Codex utilizan un `CODEX_HOME` por agente, por lo que no leen de forma predeterminada el `~/.codex` personal. El proceso normal `HOME` se sigue heredando, por lo que Codex puede ver las skills y las entradas del marketplace de plugins compartidas de `$HOME/.agents/*`, y los subprocesos pueden encontrar la configuración y los tokens del directorio principal del usuario.

Ejecutar `openclaw migrate codex` en un terminal interactivo muestra una vista previa del plan completo y después abre selectores de casillas antes de la confirmación final de la aplicación. Primero se solicitan los elementos de copia de skills. Utilice `Toggle all on` o `Toggle all off` para realizar una selección masiva. Pulse Space para alternar filas o Enter para activar la fila resaltada y continuar. Las skills planificadas comienzan marcadas, las skills con conflictos comienzan desmarcadas y `Skip for now` omite las copias de skills en esta ejecución, pero continúa con la selección de plugins. Cuando se pueden migrar plugins seleccionados de Codex instalados desde el código fuente y no se ha indicado `--plugin`, la migración solicita después la activación de plugins nativos de Codex por nombre de plugin. Los elementos de plugins comienzan marcados, salvo que la configuración de plugins de Codex de OpenClaw de destino ya contenga ese plugin. Los plugins existentes en el destino comienzan desmarcados y muestran una indicación de conflicto como `conflict: plugin exists`; seleccione `Toggle all off` para no migrar ningún plugin nativo de Codex en esa ejecución o `Skip for now` para detener el proceso antes de aplicar.

Para ejecuciones mediante scripts o exactas, seleccione explícitamente una o varias skills o plugins:

```bash
openclaw migrate codex --dry-run --skill gog-vault77-google-workspace
openclaw migrate apply codex --yes --skill gog-vault77-google-workspace
openclaw migrate codex --dry-run --plugin google-calendar
openclaw migrate apply codex --yes --plugin google-calendar
```

### Qué importa Codex

- `MEMORY.md` y `memory_summary.md` consolidados de Codex procedentes de
  `$CODEX_HOME/memories`, copiados en `memory/imports/codex/` para su recuperación
  indexada. No se importa la memoria sin procesar de las ejecuciones.
- Directorios de skills de Codex CLI en `$CODEX_HOME/skills`, sin incluir la caché `.system` de Codex.
- AgentSkills personales en `$HOME/.agents/skills`, copiadas en el espacio de trabajo del agente actual de OpenClaw para que sean propiedad de cada agente.
- Plugins de Codex `openai-curated` instalados desde el código fuente y detectados mediante `plugin/list` del servidor de aplicaciones de Codex. La planificación lee `plugin/read` para cada plugin instalado y habilitado.

La migración de plugins respaldados por aplicaciones tiene comprobaciones adicionales:

- Los plugins respaldados por aplicaciones requieren que la cuenta del servidor de aplicaciones de Codex de origen sea una cuenta con suscripción a ChatGPT. Las respuestas correspondientes a cuentas que no sean de ChatGPT o a cuentas ausentes se omiten con `codex_subscription_required`.
- De forma predeterminada, la migración no llama a `app/list` del origen, por lo que los plugins respaldados por aplicaciones que superan la comprobación de la cuenta se planifican sin verificar la accesibilidad de las aplicaciones de origen, y los fallos de transporte al consultar la cuenta se omiten con `codex_account_unavailable`.
- Indique `--verify-plugin-apps` para forzar una instantánea nueva de `app/list` del origen y exigir que todas las aplicaciones propias estén presentes, habilitadas y accesibles antes de planificar la activación nativa. En ese modo, los fallos de transporte al consultar la cuenta pasan a la verificación del inventario de aplicaciones de origen. La instantánea se conserva en memoria únicamente durante el proceso actual; nunca se escribe en la salida de la migración ni en la configuración de destino.

Los plugins deshabilitados, los detalles de plugins ilegibles, las cuentas de origen restringidas por suscripción y, cuando se establece `--verify-plugin-apps`, las aplicaciones ausentes, deshabilitadas o inaccesibles se convierten en elementos manuales omitidos con motivos tipificados, en lugar de entradas de configuración de destino. La aplicación llama a `plugin/install` del servidor de aplicaciones para cada plugin apto seleccionado, incluso si el servidor de aplicaciones de destino ya informa de que ese plugin está instalado y habilitado. Los plugins de Codex migrados solo pueden utilizarse en sesiones que seleccionen el entorno de ejecución nativo de Codex; no están disponibles para las ejecuciones de proveedores de OpenClaw, las vinculaciones de conversaciones ACP ni otros entornos de ejecución.

### Estado de Codex para revisión manual

Codex `config.toml`, `hooks/hooks.json` nativos, marketplaces no seleccionados, paquetes de plugins almacenados en caché que no son plugins seleccionados instalados desde el código fuente y plugins instalados desde el código fuente que no superan la comprobación de suscripción de origen no se activan automáticamente. Cuando se establece `--verify-plugin-apps`, también se omiten los plugins que no superan la comprobación del inventario de aplicaciones de origen. Todos ellos se copian o se incluyen en el informe de migración para su revisión manual.

Para los plugins seleccionados migrados e instalados desde el código fuente, se aplican las siguientes escrituras:

- `plugins.entries.codex.enabled: true`
- `plugins.entries.codex.config.codexPlugins.enabled: true`
- `plugins.entries.codex.config.codexPlugins.allow_destructive_actions: true`
- una entrada de plugin explícita con `marketplaceName: "openai-curated"` y `pluginName` para cada plugin seleccionado

La migración nunca escribe `plugins["*"]` ni almacena rutas de caché de marketplaces locales.

Los plugins omitidos no se escriben en la configuración de destino. Los fallos de suscripción del lado del origen se incluyen en los elementos manuales con motivos tipificados: `codex_subscription_required`, `codex_account_unavailable`, `plugin_disabled` o `plugin_read_unavailable`. Con `--verify-plugin-apps`, los fallos del inventario de aplicaciones de origen también pueden aparecer como `app_inaccessible`, `app_disabled`, `app_missing` o `app_inventory_unavailable`. Las instalaciones del lado del destino que requieren autenticación se incluyen en el elemento del plugin afectado con `status: "skipped"`, `reason: "auth_required"` e identificadores de aplicación saneados; sus entradas de configuración explícitas se escriben deshabilitadas hasta que se vuelvan a autorizar y habilitar. Los demás fallos de instalación generan resultados `error` asociados al elemento.

Si el inventario de plugins del servidor de aplicaciones de Codex no está disponible durante la planificación, la migración recurre a elementos informativos de paquetes almacenados en caché en lugar de hacer que falle toda la migración.

## Proveedor Hermes

El proveedor Hermes incluido sigue `$HERMES_HOME` y el perfil activo, y después utiliza el valor predeterminado de la plataforma (`~/.hermes` o `%LOCALAPPDATA%\hermes`). Se utiliza `--from <path>` para anular la detección.

### Qué importa Hermes

- Configuración predeterminada del modelo de `config.yaml`.
- Proveedores de modelos configurados y endpoints personalizados compatibles con OpenAI de `model`, `providers` y `custom_providers`.
- Definiciones de servidores MCP de `mcp_servers` o `mcp.servers`. Las correspondencias exactas de OpenClaw abarcan el enrutamiento HTTP transmisible predeterminado, el ámbito de OAuth, la verificación TLS booleana, rutas separadas para el certificado y la clave del cliente, y la política de herramientas nativas, de recursos y de prompts de Hermes. Los campos de tiempo de ejecución o credenciales exclusivos de Hermes que no sean compatibles se incluyen para su revisión manual.
- `SOUL.md` y `AGENTS.md` en el espacio de trabajo del agente de OpenClaw.
- `memories/MEMORY.md` y `memories/USER.md` anexados a los archivos de memoria del espacio de trabajo.
  En cambio, las superficies exclusivas de memoria (la página de memoria de incorporación y la página de
  importación de memoria de la interfaz de control) copian estos archivos en `memory/imports/hermes/` para
  permitir su recuperación indexada sin modificar la memoria existente del espacio de trabajo.
- Valores predeterminados de configuración de memoria para la memoria de archivos de OpenClaw, además de elementos de archivado o revisión manual para proveedores de memoria externos como Honcho.
- Skills que incluyan un archivo `SKILL.md` en cualquier ubicación bajo `skills/`; las Skills anidadas se aplanan en el directorio de Skills del espacio de trabajo.
- Valores de configuración por Skill de `skills.config`.
- Credenciales OAuth actuales de OpenAI Codex de Hermes y credenciales OAuth de OpenAI de OpenCode cuando se acepta la migración interactiva de credenciales o cuando se establece `--include-secrets`. No se debe mantener a Hermes y OpenClaw utilizando la misma concesión de actualización importada.
- Claves de API y tokens compatibles de `.env` de Hermes y `auth.json` de OpenCode cuando se acepta la migración interactiva de credenciales o cuando se establece `--include-secrets`.

### Claves `.env` compatibles

`AI_GATEWAY_API_KEY`, `ALIBABA_API_KEY`, `ANTHROPIC_API_KEY`, `ARCEEAI_API_KEY`, `CEREBRAS_API_KEY`, `CHUTES_API_KEY`, `CLOUDFLARE_AI_GATEWAY_API_KEY`, `COPILOT_GITHUB_TOKEN`, `DASHSCOPE_API_KEY`, `DEEPINFRA_API_KEY`, `DEEPSEEK_API_KEY`, `FIREWORKS_API_KEY`, `GEMINI_API_KEY`, `GH_TOKEN`, `GITHUB_TOKEN`, `GLM_API_KEY`, `GOOGLE_API_KEY`, `GROQ_API_KEY`, `HF_TOKEN`, `HUGGINGFACE_HUB_TOKEN`, `KILOCODE_API_KEY`, `KIMICODE_API_KEY`, `KIMI_API_KEY`, `KIMI_CODING_API_KEY`, `MINIMAX_API_KEY`, `MINIMAX_CODING_API_KEY`, `MISTRAL_API_KEY`, `MODELSTUDIO_API_KEY`, `MOONSHOT_API_KEY`, `NVIDIA_API_KEY`, `OPENAI_API_KEY`, `OPENCODE_API_KEY`, `OPENCODE_GO_API_KEY`, `OPENCODE_ZEN_API_KEY`, `OPENROUTER_API_KEY`, `QIANFAN_API_KEY`, `QWEN_API_KEY`, `TOGETHER_API_KEY`, `VENICE_API_KEY`, `XAI_API_KEY`, `XIAOMI_API_KEY`, `ZAI_API_KEY`, `Z_AI_API_KEY`.

### Estado destinado únicamente al archivado

El estado de Hermes que OpenClaw no puede interpretar de forma segura se copia en el informe de migración para su revisión manual, pero no se carga en la configuración ni en las credenciales activas de OpenClaw. Esto incluye `plugins/`, `sessions/`, `logs/`, `cron/`, `mcp-tokens/`, `plans/`, `workspace/`, `skins/`, `kanban/`, el estado de emparejamiento/plataforma, el estado de enrutamiento/proceso del Gateway y las bases de datos SQLite de Hermes detectadas.

### Después de aplicar

```bash
openclaw doctor
```

## Contrato de plugins

Las fuentes de migración son plugins. Un plugin declara los identificadores de sus proveedores en `openclaw.plugin.json`:

```json
{
  "contracts": {
    "migrationProviders": ["hermes"]
  }
}
```

En tiempo de ejecución, el plugin llama a `api.registerMigrationProvider(...)`. El proveedor implementa `detect`, `plan` y `apply`. El núcleo gestiona la coordinación de la CLI, la política de copias de seguridad, los prompts, la salida JSON y la comprobación previa de conflictos. El núcleo pasa el plan revisado a `apply(ctx, plan)`, y los proveedores solo pueden reconstruir el plan cuando ese argumento no esté presente por motivos de compatibilidad. Los elementos de migración pueden establecer `applyPhase: "after-promotion"` para los efectos de activación externa que la incorporación debe aplazar hasta que los datos locales preparados se hayan publicado de forma duradera. Esos proveedores deben declarar `deferredApply: { retrySafe: true }` y hacer que cada efecto aplazado se pueda repetir de forma segura después de una interrupción del proceso; la incorporación rechaza los efectos aplazados no declarados. Una operación idempotente sin efecto debe devolver un elemento que no realice modificaciones con `deferredCompletion: true` para que la recuperación pueda registrarlo como completado. La operación independiente `openclaw migrate` sigue aplicando el plan completo mediante su flujo habitual respaldado por copias de seguridad.

Los plugins de proveedores pueden utilizar `openclaw/plugin-sdk/migration` para crear elementos y obtener recuentos de resumen, además de `openclaw/plugin-sdk/migration-runtime` para realizar copias de archivos que tengan en cuenta los conflictos, copias de informes destinadas únicamente al archivado, contenedores del entorno de ejecución de configuración almacenados en caché e informes de migración.

## Integración con la incorporación

La incorporación puede ofrecer la migración cuando un proveedor detecta un origen conocido. Tanto `openclaw onboard --flow import` como `openclaw setup --wizard --import-from hermes` utilizan el mismo proveedor de migración del plugin y siguen mostrando una vista previa antes de aplicar los cambios. A diferencia de la migración independiente, la ruta de incorporación para un destino nuevo prepara los artefactos locales y las credenciales importadas, verifica o repara la inferencia importada dentro del entorno de preparación y, a continuación, promociona el espacio de trabajo y el estado del agente antes de confirmar la configuración. Un diario de promoción en modo `0600` permite que la siguiente ejecución finalice o revierta una publicación interrumpida, incluida cualquier activación externa aplazada, sin volver a procesar los datos locales importados.

<Note>
Las importaciones durante la incorporación requieren una instalación nueva de OpenClaw. Si ya existe un estado local, primero se deben restablecer la configuración, las credenciales, las sesiones y el espacio de trabajo. Las importaciones mediante copia de seguridad y sobrescritura o mediante combinación están sujetas a una función controlada para las instalaciones existentes.
</Note>

## Contenido relacionado

- [Migración desde Hermes](/es/install/migrating-hermes): guía paso a paso para el usuario.
- [Migración desde Claude](/es/install/migrating-claude): guía paso a paso para el usuario.
- [Migración](/es/install/migrating): trasladar OpenClaw a una máquina nueva.
- [Doctor](/es/gateway/doctor): comprobación de estado después de aplicar una migración.
- [Plugins](/es/tools/plugin): instalación y registro de plugins.
