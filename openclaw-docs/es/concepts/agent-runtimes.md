---
read_when:
    - Está eligiendo entre OpenClaw, Codex, ACP u otro entorno de ejecución de agentes nativo
    - Le confunden las etiquetas de proveedor/modelo/runtime en el estado o la configuración
    - Se está documentando la paridad de compatibilidad de un arnés nativo
summary: Cómo separa OpenClaw los proveedores de modelos, los modelos, los canales y los entornos de ejecución de agentes
title: Entornos de ejecución de agentes
x-i18n:
    generated_at: "2026-07-26T05:07:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 980d112946535df1566f2df4e3e71abacc2b073b51717c1e85fbb678691d39cb
    source_path: concepts/agent-runtimes.md
    workflow: 16
---

Un **entorno de ejecución de agente** posee un bucle de modelo preparado: recibe el prompt,
controla la salida del modelo, gestiona las llamadas a herramientas nativas y devuelve el turno finalizado
a OpenClaw.

Es fácil confundir los entornos de ejecución con los proveedores porque ambos aparecen cerca de la
configuración del modelo. Son capas diferentes:

| Capa                       | Ejemplos                                     | Significado                                                                                  |
| -------------------------- | -------------------------------------------- | -------------------------------------------------------------------------------------------- |
| Proveedor                  | `anthropic`, `github-copilot`, `openai`      | Cómo autentica OpenClaw, descubre modelos y asigna nombres a las referencias de modelos.     |
| Modelo                     | `claude-opus-4-6`, `gpt-5.6-sol`             | El modelo seleccionado para el turno del agente.                                             |
| Entorno de ejecución de agente | `claude-cli`, `codex`, `copilot`, `openclaw` | El bucle de bajo nivel o backend que ejecuta el turno preparado.                              |
| Canal                      | Discord, Slack, Telegram, WhatsApp           | Por dónde entran y salen los mensajes de OpenClaw.                                           |

Un **harness** es la implementación que proporciona un entorno de ejecución de agente (término de
código). Por ejemplo, el harness Codex incluido implementa el entorno de ejecución `codex`.
La configuración pública usa `agentRuntime.id` en las entradas de proveedor o modelo; las claves de
entorno de ejecución para todo el agente son heredadas y se ignoran. `openclaw doctor --fix` elimina las
asignaciones antiguas de entorno de ejecución para todo el agente y reescribe las referencias heredadas
de modelos de entorno de ejecución como referencias canónicas de proveedor/modelo, además de añadir
la política de entorno de ejecución con ámbito de modelo cuando sea necesario.

Dos familias de entornos de ejecución:

- Los **harnesses integrados** se ejecutan dentro del bucle de agente preparado de OpenClaw: el
  entorno de ejecución `openclaw` incorporado, además de harnesses de plugins registrados como
  `codex` y `copilot`.
- Los **backends de CLI** ejecutan un proceso de CLI local mientras mantienen canónica la referencia del modelo.
  Por ejemplo, `anthropic/claude-opus-5` con un
  `agentRuntime.id: "claude-cli"` con ámbito de modelo significa «seleccionar el modelo de Anthropic y ejecutarlo
  mediante Claude CLI». `claude-cli` no es un identificador de harness integrado y no debe
  pasarse a la selección de AgentHarness.

El harness `copilot` es un harness de plugin externo independiente y opcional para la
CLI de GitHub Copilot; consulte [Entorno de ejecución de agente de GitHub Copilot](/es/plugins/copilot) para
conocer la decisión de cara al usuario entre PI, Codex y el entorno de ejecución de agente de GitHub Copilot.

## Superficies de Codex

Varias superficies comparten el nombre Codex:

| Superficie                                       | Nombre/configuración de OpenClaw       | Función                                                                                                                      |
| ------------------------------------------------ | -------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| Entorno de ejecución nativo del servidor de aplicaciones de Codex | Referencias de modelo `openai/*`      | Ejecuta turnos de agente integrados de OpenAI mediante el servidor de aplicaciones de Codex. Esta es la configuración habitual de suscripción a ChatGPT/Codex. |
| Perfiles de autenticación OAuth de Codex         | Perfiles OAuth `openai`              | Almacena la autenticación de suscripción a ChatGPT/Codex que consume el harness del servidor de aplicaciones de Codex.       |
| Adaptador ACP de Codex                           | `runtime: "acp"`, `agentId: "codex"` | Ejecuta Codex mediante el plano de control externo ACP/acpx. Úselo solo cuando se solicite explícitamente ACP/acpx.           |
| Conjunto nativo de comandos de control de chat de Codex | `/codex ...`                         | Vincula, reanuda, dirige, detiene e inspecciona hilos del servidor de aplicaciones de Codex desde el chat.                   |
| Ruta de la API de la plataforma OpenAI para superficies que no son de agente | `openai/*` más autenticación mediante clave de API | API directas de OpenAI, como imágenes, embeddings, voz y tiempo real.                                                        |

Estas superficies son independientes de forma intencionada. Activar el plugin `codex`
hace que estén disponibles las funciones nativas del servidor de aplicaciones; `openclaw doctor --fix` se encarga
de reparar rutas heredadas de Codex y limpiar asignaciones de sesión obsoletas. Seleccionar `openai/*`
para un modelo de agente ahora significa «ejecutar esto mediante Codex», salvo que se esté usando una superficie
de API de OpenAI que no sea de agente.

La configuración habitual de suscripción a ChatGPT/Codex usa OAuth de Codex para la autenticación, pero
mantiene la referencia del modelo como `openai/*` y selecciona el entorno de ejecución `codex`:

```json5
{
  agents: {
    defaults: {
      model: "openai/gpt-5.6-sol",
    },
  },
}
```

Esto significa que OpenClaw selecciona una referencia de modelo de OpenAI y después solicita al entorno de ejecución
del servidor de aplicaciones de Codex que ejecute el turno de agente integrado. No significa «usar facturación de
API», ni que el canal, el catálogo de proveedores de modelos o el almacén de sesiones de OpenClaw pasen a ser Codex.

Cuando el plugin `codex` incluido esté activado, use la superficie de comandos nativa `/codex`
(`/codex bind`, `/codex threads`, `/codex resume`, `/codex steer`,
`/codex stop`) para controlar Codex mediante lenguaje natural en lugar de ACP. Use ACP para
Codex solo cuando se solicite explícitamente ACP/acpx o se esté probando la ruta del adaptador ACP.
Claude Code, Gemini CLI, OpenCode, Cursor y otros harnesses externos similares siguen usando ACP.

Árbol de decisión:

1. **Vincular/controlar/hilo/reanudar/dirigir/detener Codex** -> superficie de comandos nativa `/codex` cuando el plugin `codex` incluido esté activado.
2. **Codex como entorno de ejecución integrado** o la experiencia normal de agente Codex respaldada por una suscripción -> `openai/<model>`.
3. **OpenClaw elegido explícitamente para un modelo de OpenAI** -> mantenga la referencia del modelo como `openai/<model>` y establezca la política de entorno de ejecución del proveedor/modelo en `agentRuntime.id: "openclaw"`. Un perfil OAuth `openai` seleccionado se enruta internamente mediante el transporte de autenticación de Codex de OpenClaw.
4. **Referencias heredadas de modelos Codex en la configuración** -> repárelas con `openclaw doctor --fix` para convertirlas en `openai/<model>`; doctor conserva la ruta de autenticación de Codex añadiendo `agentRuntime.id: "codex"` con ámbito de proveedor/modelo cuando la referencia antigua del modelo lo implicaba. Las referencias heredadas de modelos **`codex-cli/*`** se reparan para usar la misma ruta `openai/<model>` del servidor de aplicaciones de Codex; OpenClaw ya no mantiene un backend de CLI de Codex incluido.
5. **ACP, acpx o adaptador ACP de Codex solicitado explícitamente** -> `runtime: "acp"` y `agentId: "codex"`.
6. **Claude Code, Gemini CLI, OpenCode, Cursor, Droid u otro harness externo** -> ACP/acpx, no el entorno de ejecución nativo de subagentes.

| Si se refiere a...                             | Use...                                       |
| ---------------------------------------------- | -------------------------------------------- |
| Control de chat/hilos del servidor de aplicaciones de Codex | `/codex ...` del plugin `codex` incluido |
| Entorno de ejecución de agente integrado del servidor de aplicaciones de Codex | Referencias de modelos de agente `openai/*` |
| OAuth de OpenAI Codex                          | Perfiles OAuth `openai`                      |
| Claude Code u otro harness externo             | ACP/acpx                                     |

Para conocer la división de prefijos de la familia OpenAI, consulte [OpenAI](/es/providers/openai) y
[Proveedores de modelos](/es/concepts/model-providers). Para conocer el contrato de compatibilidad del entorno de ejecución
de Codex, consulte [Entorno de ejecución del harness Codex](/es/plugins/codex-harness-runtime#v1-support-contract).

## Propiedad del entorno de ejecución

Los distintos entornos de ejecución poseen diferentes partes del bucle:

| Superficie                    | Integrado en OpenClaw                            | Servidor de aplicaciones de Codex                                              |
| ----------------------------- | ------------------------------------------------ | ------------------------------------------------------------------------------ |
| Propietario del bucle del modelo | OpenClaw, mediante el ejecutor integrado de OpenClaw | Servidor de aplicaciones de Codex                                           |
| Estado canónico del hilo      | Transcripción de OpenClaw                        | Hilo de Codex, más un reflejo de la transcripción de OpenClaw                  |
| Herramientas dinámicas de OpenClaw | Bucle de herramientas nativo de OpenClaw    | Conectadas mediante el adaptador de Codex                                      |
| Herramientas nativas de shell y archivos | Ruta de OpenClaw                         | Herramientas nativas de Codex, conectadas mediante hooks nativos cuando se admite |
| Motor de contexto             | Ensamblado de contexto nativo de OpenClaw        | OpenClaw proyecta el contexto ensamblado en el turno de Codex                  |
| Compaction                    | OpenClaw o el motor de contexto seleccionado     | Compaction nativa de Codex, con notificaciones de OpenClaw y mantenimiento del reflejo |
| Entrega por canal             | OpenClaw                                         | OpenClaw                                                                       |

Regla de diseño: si OpenClaw posee la superficie, puede proporcionar el comportamiento normal de los hooks de plugins.
Si el entorno de ejecución nativo posee la superficie, OpenClaw necesita eventos del entorno de ejecución o hooks
nativos. Si el entorno de ejecución nativo posee el estado canónico del hilo, OpenClaw refleja y proyecta el contexto
en lugar de reescribir componentes internos no compatibles.

## Selección del entorno de ejecución

OpenClaw resuelve un entorno de ejecución integrado después de resolver el proveedor y el modelo, en
este orden:

1. La **política de entorno de ejecución con ámbito de modelo** tiene prioridad. Se encuentra en una entrada de modelo
   del proveedor configurado, o en `agents.defaults.models["provider/model"].agentRuntime`
   / `agents.entries.*.models["provider/model"].agentRuntime`. Un comodín de proveedor como `agents.defaults.models["vllm/*"].agentRuntime` se aplica
   después de la política exacta del modelo, para que los modelos de proveedor descubiertos dinámicamente puedan
   compartir un entorno de ejecución sin sobrescribir excepciones exactas por modelo.
2. **Política de entorno de ejecución con ámbito de proveedor**: `models.providers.<provider>.agentRuntime`.
3. **Modo `auto`**: los entornos de ejecución de plugins registrados pueden reclamar pares de proveedor/modelo compatibles.
4. Si nada reclama el turno en el modo `auto`, OpenClaw recurre a
   `openclaw` como entorno de ejecución de compatibilidad. Use un identificador de entorno de ejecución explícito cuando
   la ejecución deba ser estricta.

Las asignaciones de entorno de ejecución para toda la sesión y todo el agente se ignoran: `OPENCLAW_AGENT_RUNTIME`,
el estado de sesión `agentHarnessId`/`agentRuntimeOverride`, `agents.defaults.agentRuntime`
y `agents.entries.*.agentRuntime`. Ejecute `openclaw doctor --fix` para eliminar la
configuración obsoleta del entorno de ejecución para todo el agente y convertir las referencias heredadas de modelos
de entorno de ejecución cuando sea posible conservar la intención.

Los entornos de ejecución de plugins explícitos de proveedor/modelo fallan de forma cerrada: `agentRuntime.id: "codex"`
en un proveedor o modelo significa Codex, o un error claro de selección/entorno de ejecución; nunca se
redirige silenciosamente a OpenClaw. Solo `auto` puede dirigir a OpenClaw un turno sin coincidencias.

Los alias de backends de CLI difieren de los identificadores de harnesses integrados. Forma preferida para Claude CLI:

```json5
{
  agents: {
    defaults: {
      model: "anthropic/claude-opus-5",
      models: {
        "anthropic/claude-opus-5": {
          agentRuntime: { id: "claude-cli" },
        },
      },
    },
  },
}
```

Las referencias heredadas como `claude-cli/claude-opus-4-7` siguen siendo compatibles por
compatibilidad, pero la configuración nueva debe mantener canónicos el proveedor/modelo y
situar el backend de ejecución en la política de entorno de ejecución del proveedor/modelo.

Las referencias heredadas `codex-cli/*` son diferentes: doctor las migra a `openai/*` para que
se ejecuten mediante el harness del servidor de aplicaciones de Codex en lugar de conservar un backend de
CLI de Codex.

El modo `auto` es intencionadamente conservador para la mayoría de los proveedores. Los modelos de agente
de OpenAI son la excepción: tanto un entorno de ejecución no establecido como `auto` se resuelven en el harness
Codex. La configuración explícita del entorno de ejecución de OpenClaw sigue siendo una ruta de compatibilidad
opcional para los turnos de agente `openai/*`; cuando se combina con un perfil OAuth `openai`
seleccionado, OpenClaw dirige internamente esa ruta mediante el transporte de autenticación de Codex
mientras mantiene la referencia pública del modelo como `openai/*`. Las asignaciones obsoletas de sesión
del entorno de ejecución de OpenAI se ignoran durante la selección del entorno de ejecución y pueden limpiarse con
`openclaw doctor --fix`.

Si `openclaw doctor` advierte que el plugin `codex` está habilitado mientras aún quedan
referencias de modelos Codex heredadas en la configuración, trátelo como un estado de ruta heredada y ejecute
`openclaw doctor --fix` para reescribirlo como `openai/*` con el entorno de ejecución Codex.

## Entorno de ejecución de agentes de GitHub Copilot

El plugin externo `@openclaw/copilot` registra un entorno de ejecución `copilot` opcional
respaldado por la CLI de GitHub Copilot (`@github/copilot-sdk`). Este reclama el
proveedor de suscripción canónico `github-copilot` y **nunca** lo selecciona
`auto`. Habilítelo por modelo o por proveedor mediante `agentRuntime.id`:

```json5
{
  agents: {
    defaults: {
      model: "github-copilot/gpt-5.5",
      models: {
        "github-copilot/gpt-5.5": {
          agentRuntime: { id: "copilot" },
        },
      },
    },
  },
}
```

El arnés reclama su proveedor, entorno de ejecución, clave de sesión de la CLI y prefijo
del perfil de autenticación en `extensions/copilot/doctor-contract-api.ts`, que `openclaw doctor`
carga automáticamente. Para obtener información sobre la configuración, la autenticación, la replicación de transcripciones, Compaction, el
contrato declarativo de doctor y la decisión más amplia del SDK entre PI, Codex y Copilot,
consulte [Entorno de ejecución de agentes de GitHub Copilot](/es/plugins/copilot).

## Contrato de compatibilidad

Cuando un entorno de ejecución no sea OpenClaw, su documentación debe indicar qué superficies de OpenClaw
admite:

| Pregunta                               | Por qué es importante                                                                                    |
| -------------------------------------- | ------------------------------------------------------------------------------------------------- |
| ¿Quién controla el bucle del modelo?               | Determina dónde se producen los reintentos, la continuación de herramientas y las decisiones sobre la respuesta final.                   |
| ¿Quién controla el historial canónico del hilo?     | Determina si OpenClaw puede editar el historial o solo replicarlo.                                   |
| ¿Funcionan las herramientas dinámicas de OpenClaw?        | La mensajería, las sesiones, Cron y las herramientas controladas por OpenClaw dependen de ello.                                 |
| ¿Funcionan los hooks de herramientas dinámicas?            | Los Plugins esperan `before_tool_call`, `after_tool_call` y middleware en torno a las herramientas controladas por OpenClaw. |
| ¿Funcionan los hooks de herramientas nativas?             | El shell, los parches y las herramientas controladas por el entorno de ejecución necesitan compatibilidad con hooks nativos para aplicar políticas y realizar observaciones.        |
| ¿Se ejecuta el ciclo de vida del motor de contexto? | Los Plugins de memoria y contexto dependen de las fases de ensamblaje, ingesta, procesamiento posterior al turno y Compaction.      |
| ¿Qué datos de Compaction se exponen?       | Algunos Plugins solo necesitan notificaciones; otros necesitan metadatos de elementos conservados y descartados.                          |
| ¿Qué no se admite intencionadamente?     | Los usuarios no deben presuponer equivalencia con OpenClaw cuando el entorno de ejecución nativo controla más estado.            |

El contrato de compatibilidad del entorno de ejecución Codex se documenta en
[Entorno de ejecución del arnés de Codex](/es/plugins/codex-harness-runtime#v1-support-contract).

## Etiquetas de estado

La salida de estado puede mostrar las etiquetas `Execution` y `Runtime`. Interprételas como
diagnósticos, no como nombres de proveedores:

- Una referencia de modelo como `openai/gpt-5.6-sol` es el proveedor/modelo seleccionado.
- Un identificador de entorno de ejecución como `codex` es el bucle que ejecuta el turno.
- Una etiqueta de canal como Telegram o Discord indica dónde se desarrolla la conversación.

Si una ejecución muestra un entorno de ejecución inesperado, inspeccione primero la política del entorno de ejecución
del proveedor/modelo seleccionado. Las fijaciones heredadas del entorno de ejecución de la sesión ya no determinan el enrutamiento.

## Temas relacionados

- [Arnés de Codex](/es/plugins/codex-harness)
- [Entorno de ejecución del arnés de Codex](/es/plugins/codex-harness-runtime)
- [Entorno de ejecución de agentes de GitHub Copilot](/es/plugins/copilot)
- [OpenAI](/es/providers/openai)
- [Plugins de arnés de agentes](/es/plugins/sdk-agent-harness)
- [Bucle del agente](/es/concepts/agent-loop)
- [Modelos](/es/concepts/models)
- [Estado](/es/cli/status)
