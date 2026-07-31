---
read_when:
    - Quiere usar el entorno oficial de app-server de Codex
    - Necesita ejemplos de configuración del entorno de Codex
    - Se desea que las implementaciones exclusivas de Codex fallen en lugar de recurrir a OpenClaw como alternativa.
summary: Ejecuta los turnos del agente integrado de OpenClaw mediante el arnés oficial del servidor de aplicaciones de Codex
title: Entorno de Codex
x-i18n:
    generated_at: "2026-07-26T04:43:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e016a1689af65c5520d529ce22a87bd25ee29369f7aedca77b27f943a7f21b0f
    source_path: plugins/codex-harness.md
    workflow: 16
---

El plugin oficial `codex` ejecuta turnos de agente de OpenAI integrados mediante el app-server de Codex
en lugar del entorno integrado de OpenClaw. Codex controla la
sesión de agente de bajo nivel: reanudación nativa de hilos, continuación nativa de herramientas,
Compaction nativa y ejecución del app-server. OpenClaw sigue controlando los canales
de chat, los archivos de sesión, la selección de modelos, las herramientas dinámicas de OpenClaw, las aprobaciones,
la entrega de contenido multimedia y la réplica visible de la transcripción.

Use referencias canónicas de modelos de OpenAI, como `openai/gpt-5.6-sol`. No configure
referencias heredadas de GPT de Codex; defina el orden de autenticación del agente de OpenAI en `auth.order.openai`.
Los identificadores heredados de perfiles de autenticación de Codex y las entradas heredadas del orden de autenticación de Codex se
reparan mediante `openclaw doctor --fix`.

Cuando la política de ejecución del proveedor/modelo no está definida o es `auto`, el prefijo `openai/*` por sí solo
nunca selecciona este entorno. OpenAI puede seleccionar Codex implícitamente solo para una
ruta oficial HTTPS exacta de Responses de la plataforma o de Responses de ChatGPT sin
ninguna sustitución de solicitud definida. Consulte
[Entorno de agente implícito de OpenAI](/es/providers/openai#implicit-agent-runtime).
Si Codex controla la autenticación antes de que se conozca el enrutamiento por la plataforma o ChatGPT, OpenClaw
sigue exigiendo que cada ruta candidata declare compatibilidad con Codex. El control nativo
de la autenticación por sí solo nunca omite esa comprobación de ruta.

Cuando no hay ningún entorno aislado de OpenClaw activo, OpenClaw inicia los hilos del app-server de Codex
con el modo de código nativo de Codex habilitado (el modo de solo código permanece desactivado de forma predeterminada), por lo que
las capacidades nativas de espacio de trabajo y código siguen disponibles junto con las herramientas
dinámicas de OpenClaw enrutadas mediante el puente `item/tool/call` del app-server. Un
entorno aislado de OpenClaw activo o una política de herramientas restringida deshabilita por completo el modo de código nativo,
salvo que se habilite la ruta experimental del servidor de ejecución del entorno aislado.

Con el valor predeterminado `tools.exec.host: "auto"` y sin ningún entorno aislado de OpenClaw activo,
Codex también recibe las herramientas `node_exec` y `node_process` para ejecutar comandos en nodos
emparejados. El shell nativo permanece en el host y el espacio de trabajo del app-server de Codex
(locales al Gateway en la implementación stdio predeterminada); `node_exec` selecciona un nodo por
nombre o identificador y mantiene vigente la política de aprobación de nodos de OpenClaw. Si una lista de permitidos
finita del entorno deshabilita el modo de código nativo y deja el turno sin un
entorno de ejecución, OpenClaw mantiene disponibles en su lugar sus herramientas `exec` y `process`
filtradas por la política para la ejecución directa y sin aislamiento.

Esta función nativa de Codex es independiente del
[modo de código de OpenClaw](/es/tools/code-mode), un entorno de ejecución QuickJS-WASI opcional
para ejecuciones genéricas de OpenClaw con una forma de entrada `exec` diferente. Para conocer
la separación más amplia entre modelo, proveedor y entorno, comience por
[Entornos de agente](/es/concepts/agent-runtimes): `openai/gpt-5.6-sol` es la referencia del
modelo, `codex` es el entorno, y Telegram, Discord, Slack u otro
canal es la superficie de comunicación.

## Requisitos

- El plugin oficial `@openclaw/codex` instalado. Incluya `codex` en
  `plugins.allow` si la configuración usa una lista de permitidos.
- Un app-server estable de Codex desde `0.143.0` hasta `0.145.0`. El plugin administra de forma predeterminada un
  binario compatible, por lo que un comando `codex` en `PATH` no afecta al inicio
  normal.
- Autenticación de Codex mediante `openclaw models auth login --provider openai`, una
  cuenta del app-server ya presente en el directorio principal de Codex del agente o un
  perfil explícito de autenticación con clave de API de Codex.

Para conocer la precedencia de autenticación, el aislamiento del entorno, los comandos personalizados del app-server,
la detección de modelos y la lista completa de campos de configuración, consulte la
[Referencia del entorno de Codex](/es/plugins/codex-harness-reference).

## Inicio rápido

Instale el plugin oficial y, a continuación, inicie sesión con OAuth de Codex:

```bash
openclaw plugins install @openclaw/codex
openclaw models auth login --provider openai
```

Habilite el plugin `codex` y seleccione un modelo de agente de OpenAI:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.6-sol",
    },
  },
}
```

Si la configuración usa `plugins.allow`, añada también `codex` allí:

```json5
{
  plugins: {
    allow: ["codex"],
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
}
```

Reinicie el Gateway después de cambiar la configuración del plugin. Si un chat ya tiene una
sesión, ejecute primero `/new` o `/reset` para que el siguiente turno determine el entorno
a partir de la configuración actual.

## Compartir hilos con Codex Desktop y la CLI

El valor predeterminado `appServer.homeScope: "agent"` aísla cada agente de OpenClaw del
estado nativo de Codex del operador. Para permitir que un propietario inspeccione y administre los
mismos hilos nativos que muestran Codex Desktop y la CLI de Codex, habilite el
directorio principal de usuario de Codex:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            homeScope: "user",
          },
        },
      },
    },
  },
}
```

El modo de directorio principal del usuario admite un proceso stdio administrado localmente o el transporte
compartido mediante socket Unix. Usa `$CODEX_HOME` cuando está definido y `~/.codex` en caso contrario, incluida
la autenticación nativa de Codex, la configuración, los plugins y el almacén de hilos de ese directorio principal. OpenClaw no
inyecta un perfil de autenticación de OpenClaw en este app-server.

Los turnos del propietario obtienen la herramienta `codex_threads`: enumerar, buscar, leer, bifurcar, renombrar,
archivar y restaurar hilos nativos. Bifurque un hilo para continuarlo en
OpenClaw; la bifurcación se vincula a la sesión actual de OpenClaw y permanece
visible para otros clientes nativos de Codex. Para archivarlo se requiere una
confirmación explícita de que el hilo está cerrado en otro lugar. Cuando la supervisión también está
habilitada, los campos y las modificaciones de la transcripción requieren la habilitación
`supervision.allowRawTranscripts` o `supervision.allowWriteControls` correspondiente.

No reanude ni escriba simultáneamente en el mismo hilo mediante app-servers stdio
administrados independientes. Codex coordina los escritores activos dentro de un app-server, no
entre procesos separados. La bifurcación es la ruta de coexistencia segura para las sesiones stdio
normales del directorio principal del usuario.

`appServer.homeScope: "user"` por sí solo no controla el catálogo de la flota. La
detección de sesiones nativas está habilitada mientras el plugin está activo; defina
`sessionCatalog.enabled: false` para eliminarla de la barra lateral de OpenClaw sin
deshabilitar Codex. El catálogo usa una conexión de supervisión independiente; sin
una configuración de conexión `appServer` explícita, esa conexión usa de forma predeterminada stdio
administrado del directorio principal del usuario, mientras que el entorno normal permanece limitado al agente. Ambas
rutas respetan la configuración `appServer` explícita. Defina `homeScope: "user"`
explícitamente, como se muestra arriba, cuando el entorno normal también deba compartir el estado nativo.

## Supervisar sesiones de Codex

El mismo plugin `codex` puede enumerar sesiones de Codex no archivadas del equipo del Gateway
y de los nodos emparejados habilitados. Una sesión almacenada o inactiva local al Gateway puede
crear un chat vinculado al modelo que replica su historial persistente acotado de mensajes del usuario y el asistente.
Su vinculación privada usa la conexión de supervisión para la instantánea nativa,
la rama canónica y los turnos posteriores, mientras que las sesiones normales de Codex permanecen
limitadas al agente. El primer inicio canónico usa exactamente el modelo y el proveedor que
Codex devuelve para la bifurcación de la instantánea. Las reanudaciones posteriores dejan la selección a la
configuración nativa de Codex; el modelo externo de OpenClaw y la cadena de reserva nunca
la sustituyen. Las filas almacenadas e inactivas pueden archivarse después de confirmar explícitamente
que no hay otro ejecutor. Las fuentes activas no pueden crear una rama ni archivarse; aun así, se puede abrir un
chat supervisado existente. Las sesiones de nodos emparejados siguen conteniendo únicamente metadatos.

Consulte [Supervisar sesiones de Codex](/es/plugins/codex-supervision) para obtener información sobre la configuración, las reglas de
bifurcación, los límites de los nodos emparejados, la exposición de metadatos y la resolución de problemas.

## Configuración

| Necesidad                                           | Establecer                                                                                       | Dónde                              |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------ | ---------------------------------- |
| Habilitar el entorno                                | `plugins.entries.codex.enabled: true`                                                            | Configuración de OpenClaw          |
| Ocultar la detección de sesiones nativas de Codex   | `plugins.entries.codex.config.sessionCatalog.enabled: false`                                     | Configuración del plugin de Codex  |
| Conservar la instalación de un plugin incluido en la lista de permitidos | Incluir `codex` en `plugins.allow`                                      | Configuración de OpenClaw          |
| Permitir que los turnos de OpenAI aptos usen Codex implícitamente | Ruta oficial HTTPS exacta de Responses/ChatGPT, sin sustitución de solicitud definida, entorno sin definir/`auto` | Configuración del proveedor/modelo de OpenAI |
| Iniciar sesión con OAuth de ChatGPT/Codex            | `openclaw models auth login --provider openai`                                                   | Perfil de autenticación de la CLI  |
| Añadir una copia de seguridad con clave de API para las ejecuciones de Codex | Perfil de clave de API `openai:*` incluido después de la autenticación de suscripción en `auth.order.openai` | Perfil de autenticación de la CLI + configuración de OpenClaw |
| Impedir la continuación cuando Codex no esté disponible | `agentRuntime.id: "codex"` del proveedor o modelo                                      | Configuración del proveedor/modelo de OpenClaw |
| Usar tráfico directo de la API de OpenAI             | `agentRuntime.id: "openclaw"` del proveedor o modelo con autenticación normal de OpenAI   | Configuración del proveedor/modelo de OpenClaw |
| Ajustar el comportamiento del app-server             | `plugins.entries.codex.config.appServer.*`                                                       | Configuración del plugin de Codex  |
| Habilitar aplicaciones de plugins nativos de Codex   | `plugins.entries.codex.config.codexPlugins.*`                                                    | Configuración del plugin de Codex  |
| Habilitar el uso del ordenador de Codex              | `plugins.entries.codex.config.computerUse.*`                                                     | Configuración del plugin de Codex  |

Prefiera `auth.order.openai` para el orden de prioridad de suscripción con copia de seguridad mediante clave de API.
Los identificadores de perfiles de autenticación heredados de Codex y el orden de autenticación heredado de Codex son
un estado heredado exclusivo de doctor; no escriba nuevas referencias heredadas de GPT de Codex.

```json5
{
  auth: {
    order: {
      openai: ["openai:user@example.com", "openai:api-key-backup"],
    },
  },
}
```

Para una ruta efectiva compatible con Codex, los dos perfiles anteriores siguen siendo candidatos
para la misma ejecución de Codex. El orden de los perfiles elige las credenciales, no el entorno.
Cambiar el orden de autenticación no hace que una ruta personalizada, de Completions, HTTP o
con sustitución de solicitud sea compatible con Codex.

### Compaction

No establezca `compaction.model` ni `compaction.provider` en agentes respaldados
por Codex. Codex realiza la compactación mediante el estado nativo de sus hilos del app-server, por lo que
OpenClaw ignora esas sustituciones locales del resumidor durante la ejecución, y
`openclaw doctor --fix` las elimina cuando el agente usa Codex.

Lossless sigue siendo compatible como motor de contexto para el ensamblaje, la ingesta y el
mantenimiento en torno a los turnos de Codex, configurado mediante
`plugins.slots.contextEngine: "lossless-claw"` y
`plugins.entries.lossless-claw.config.summaryModel`, no mediante
`agents.defaults.compaction.provider`. `openclaw doctor --fix` migra la
forma antigua `compaction.provider: "lossless-claw"` a la ranura del motor
de contexto Lossless cuando Codex es el entorno activo, pero Codex nativo sigue
controlando la compactación. El entorno nativo del app-server admite motores de contexto
que necesitan ensamblaje previo al prompt; los backends genéricos de la CLI, incluido `codex-cli`,
no proporcionan esa capacidad del host.

Para los agentes respaldados por Codex, `/compact` inicia la compactación nativa
del app-server de Codex en el hilo vinculado y espera su resultado terminal. Se aplica el
presupuesto compartido `agents.defaults.compaction.timeoutSeconds`; al agotarse el tiempo de espera,
OpenClaw solicita a Codex que interrumpa el turno nativo y mantiene el bloqueo por hilo
hasta que se confirme la finalización. Nunca recurre a un motor de contexto ni a un
resumidor público de OpenAI. Si falta la vinculación del hilo nativo de Codex o está
obsoleta, el comando impide la continuación en lugar de cambiar silenciosamente el backend
de compactación.

### Contexto largo de la API directa

La suscripción a Codex y el tráfico directo de la API de OpenAI son contratos independientes. El
catálogo activo de ChatGPT/Codex suele ofrecer una ventana de modelo de `272000` tokens,
mientras que OpenAI documenta una ventana de `1050000` tokens para la API de la plataforma y una
salida máxima de `128000` para GPT-5.5 y GPT-5.6. Reservar toda la capacidad de salida
deja un presupuesto de entrada derivado de `922000` tokens. Las solicitudes que superan los
`272000` tokens de entrada usan las tarifas superiores de OpenAI para contextos largos.

Parta de un catálogo de modelos de Codex completo y compatible con la versión
de Codex instalada. Para cada entrada directa de GPT-5.5 o GPT-5.6 que deba usar un contexto largo,
conserve el resto del descriptor y establezca:

```json
{
  "context_window": 922000,
  "max_context_window": 922000,
  "auto_compact_token_limit": 700000
}
```

Codex aplica su reserva normal del 95 % de la ventana efectiva al valor de catálogo
`922000`, por lo que indica aproximadamente `875900` tokens utilizables. Realizar la compactación en
`700000` deja `175900` tokens antes de ese límite efectivo y `222000` antes de la
capacidad de entrada segura del proveedor. Este margen mayor es deliberado: Codex comprueba
el contexto ya registrado antes de añadir el siguiente mensaje del usuario y las
actualizaciones de contexto, por lo que el umbral debe cubrir un turno entrante grande, además de las herramientas,
las instrucciones, la serialización y el propio turno de compactación.

Para usar Codex CLI o Desktop de forma independiente, un proveedor personalizado con autenticación mediante
comando puede leer la clave de API desde un llavero del sistema o un gestor de secretos, mientras que el inicio de sesión
normal de ChatGPT sigue disponible para los conectores:

```toml
model = "gpt-5.6-terra"
model_provider = "openai_api_direct"
model_context_window = 922000
model_auto_compact_token_limit = 700000
model_auto_compact_token_limit_scope = "total"
model_catalog_json = "/absolute/path/to/models-api-1m.json"

[model_providers.openai_api_direct]
name = "OpenAI API direct"
base_url = "https://api.openai.com/v1"
wire_api = "responses"
requires_openai_auth = false

[model_providers.openai_api_direct.auth]
command = "/absolute/path/to/read-openai-inference-key"
timeout_ms = 5000
refresh_interval_ms = 300000
```

El asistente de autenticación debe imprimir únicamente la clave en stdout. No la incluya en TOML.

Para el arnés del servidor de aplicaciones de Codex de OpenClaw, mantenga el directorio principal predeterminado de Codex
con ámbito de agente y permita que OpenClaw inyecte un perfil de clave de API `openai`. Pase el catálogo y
los límites de contexto como argumentos nativos del servidor de aplicaciones de Codex:

```json5
{
  auth: {
    order: {
      openai: ["openai:api-key"],
    },
  },
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            args: [
              "app-server",
              "--listen",
              "stdio://",
              "-c",
              'model_catalog_json="/absolute/path/to/models-api-1m.json"',
              "-c",
              "model_context_window=922000",
              "-c",
              "model_auto_compact_token_limit=700000",
              "-c",
              "model_auto_compact_token_limit_scope=total",
            ],
          },
        },
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.6-terra",
      models: {
        "openai/gpt-5.6-terra": { agentRuntime: { id: "codex" } },
      },
    },
  },
}
```

Sustituya `openai:api-key` por el identificador real del perfil de clave de API si es necesario. El
servidor de aplicaciones con ámbito de agente recibe únicamente esa clave preparada; el inicio de sesión nativo
de ChatGPT `~/.codex`, los plugins, los conectores y el almacén de hilos del operador permanecen
intactos. El servidor de aplicaciones de Codex `0.144.6` no adjunta el portador de un proveedor personalizado
con autenticación mediante comando en los turnos del servidor de aplicaciones, por lo que debe usar la ruta de clave de API inyectada anterior
en lugar de `homeScope: "user"` para esta ruta.

Después de cambiar el catálogo o los argumentos del servidor de aplicaciones, reinicie el Gateway e
inicie un chat nuevo. Los hilos nativos existentes conservan la configuración registrada del proveedor
y del modelo. Verifique el entorno de ejecución con `/status` y `/codex status`, y después
envíe un turno directo e inocuo a la API antes de iniciar una sesión larga.

<Warning>
El contexto largo es deliberadamente opcional. OpenAI factura la solicitud completa con tarifas de
entrada de 2× y de salida de 1,5× cuando la entrada supera los `272000` tokens. La API sigue siendo
la fuente autoritativa respecto al acceso, los límites reales y la facturación. Consulte
[límites de los modelos de OpenAI](https://developers.openai.com/api/docs/models/compare) y
[precios de la API](https://developers.openai.com/api/docs/pricing).
</Warning>

El resto de esta página trata la arquitectura de despliegue, el enrutamiento con cierre ante fallos, la política de
aprobación del guardián, los plugins nativos de Codex y Computer Use. Para consultar listas completas
de opciones, valores predeterminados, enumeraciones, detección, aislamiento del entorno, tiempos de espera y
campos de transporte del servidor de aplicaciones, consulte la
[referencia del arnés de Codex](/es/plugins/codex-harness-reference).

## Verificar el entorno de ejecución de Codex

Use `/status` en el chat en el que espera usar Codex. Un turno de agente de OpenAI
respaldado por Codex muestra:

```text
Entorno de ejecución: OpenAI Codex
```

Después, compruebe el estado del servidor de aplicaciones de Codex:

```text
/codex status
/codex models
/codex binding
```

`/codex binding` informa sobre el hilo nativo adjunto y la configuración actual del modelo.
`/codex status` informa sobre la conectividad del servidor de aplicaciones, la cuenta, los límites de uso, los servidores
MCP y las habilidades. `/codex models` enumera el catálogo activo del servidor de aplicaciones de Codex
para el arnés y la cuenta. Si `/status` resulta inesperado, consulte
[Solución de problemas](#troubleshooting).

## Enrutamiento y selección de modelos

Mantenga separadas las referencias de proveedores y la política del entorno de ejecución:

- Use `openai/gpt-*` para la selección canónica de modelos de OpenAI. El prefijo por sí solo
  nunca selecciona Codex.
- Cuando el entorno de ejecución no está establecido o es `auto`, solo una ruta oficial HTTPS exacta de Platform Responses
  o ChatGPT Responses sin ninguna sustitución de solicitud definida puede seleccionar Codex
  implícitamente.
- No use referencias heredadas de GPT de Codex en la configuración; ejecute `openclaw doctor --fix` para
  reparar las referencias heredadas y las fijaciones obsoletas de rutas de sesión.
- `agentRuntime.id: "codex"` convierte Codex en un requisito con cierre ante fallos para una
  ruta compatible. No convierte en compatible una ruta efectiva incompatible.
- `agentRuntime.id: "openclaw"` habilita para un proveedor o modelo el entorno de ejecución integrado
  de OpenClaw cuando esto es intencionado.
- `/codex ...` controla las conversaciones nativas del servidor de aplicaciones de Codex desde el chat.
- ACP/acpx es una ruta de arnés externo independiente. Úsela solo cuando el usuario
  solicite ACP/acpx o un adaptador de arnés externo.

| Intención del usuario                                      | Uso                                                                                                   |
| ---------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| Adjuntar el chat actual                                    | `/codex bind [thread-id] [--cwd <path>] [--model <model>] [--provider <provider>]`                    |
| Reanudar un hilo de Codex existente                        | `/codex resume <thread-id>`                                                                           |
| Enumerar o filtrar hilos de Codex                          | `/codex threads [filter]`                                                                             |
| Leer o actualizar el objetivo nativo del hilo vinculado    | `/codex goal [status\|set <objective>\|pause\|resume\|block\|complete\|clear]`                        |
| Enumerar plugins nativos de Codex                          | `/codex plugins list`                                                                                 |
| Activar o desactivar un plugin nativo de Codex configurado | `/codex plugins enable <name>`, `/codex plugins disable <name>`                                       |
| Reanudar una sesión almacenada de Codex CLI como turno de nodo emparejado | `/codex sessions --host <node> [filter]`, después `/codex resume <session-id> --host <node> --bind here` |
| Ver sesiones de Codex no archivadas en varios equipos      | Active la supervisión de Codex y abra **Sesiones de Codex**                                           |
| Cambiar el modelo, el modo rápido o los permisos del hilo vinculado | `/codex model <model>`, `/codex fast [on\|off\|status]`, `/codex permissions [default\|yolo\|status]` |
| Detener o dirigir el turno activo                          | `/codex stop`, `/codex steer <text>`                                                                  |
| Desvincular la asociación actual                           | `/codex detach` (alias `/codex unbind`)                                                               |
| Enviar únicamente comentarios sobre Codex                  | `/codex diagnostics [note]`                                                                           |
| Iniciar una tarea de ACP/acpx                              | Comandos de sesión de ACP/acpx, no `/codex`                                                               |

| Caso de uso                                      | Configuración                                                                                                 | Verificación                            | Notas                                      |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------- | --------------------------------------- | ------------------------------------------ |
| Ruta de OpenAI apta con entorno de ejecución nativo de Codex | Ruta oficial HTTPS exacta de Responses/ChatGPT sin ninguna sustitución de solicitud definida, más el plugin `codex` activado | `/status` muestra `Runtime: OpenAI Codex` | Ruta implícita cuando el entorno de ejecución no está establecido/es `auto` |
| Cierre ante fallos si Codex no está disponible   | `agentRuntime.id: "codex"` del proveedor o modelo                                                                     | El turno falla en lugar de recurrir al entorno integrado | Úselo para despliegues exclusivos de Codex |
| Tráfico directo con clave de API de OpenAI a través de OpenClaw | `agentRuntime.id: "openclaw"` del proveedor o modelo y autenticación normal de OpenAI                                    | `/status` muestra el entorno de ejecución de OpenClaw | Úselo solo cuando OpenClaw sea intencional  |
| Configuración heredada                           | referencias heredadas de GPT de Codex                                                                          | `openclaw doctor --fix` las reescribe        | No escriba configuraciones nuevas de este modo |
| Adaptador de Codex para ACP/acpx                 | `sessions_spawn({ runtime: "acp" })` de ACP                                                                                      | Estado de tarea/sesión de ACP           | Independiente del arnés nativo de Codex    |

`agents.defaults.imageModel` sigue la misma separación de prefijos. Use `openai/gpt-*`
para la ruta normal de OpenAI y `codex/gpt-*` solo cuando la comprensión de imágenes
deba ejecutarse mediante un turno acotado del servidor de aplicaciones de Codex. Doctor reescribe las referencias heredadas
de GPT de Codex como `openai/gpt-*`.

## Patrones de despliegue

### Despliegue básico de Codex

Use la configuración de inicio rápido para un modelo de OpenAI cuya ruta HTTPS oficial
efectiva sea apta para seleccionar Codex implícitamente:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.6-sol",
    },
  },
}
```

### Despliegue con varios proveedores

Mantenga Claude como agente predeterminado y añada un agente de Codex con nombre:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
  agents: {
    defaults: {
      model: "anthropic/claude-opus-4-6",
    },
    list: [
      {
        id: "main",
        default: true,
        model: "anthropic/claude-opus-4-6",
      },
      {
        id: "codex",
        name: "Codex",
        model: "openai/gpt-5.6-sol",
      },
    ],
  },
}
```

El agente `main` usa su ruta normal de proveedor. El agente `codex` usa el servidor de aplicaciones
de Codex cuando su ruta efectiva de OpenAI sigue siendo compatible; añada
`agentRuntime.id: "codex"` explícito con ámbito de modelo cuando deba ser un requisito
con cierre ante fallos.

### Despliegue de Codex con cierre ante fallos

Una ruta oficial HTTPS exacta y apta de OpenAI puede resolverse mediante Codex cuando el
plugin incluido está disponible. Añada una política explícita del entorno de ejecución para definir una regla
de cierre ante fallos:

```json5
{
  models: {
    providers: {
      openai: {
        agentRuntime: {
          id: "codex",
        },
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.6-sol",
    },
  },
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
}
```

Con Codex forzado, OpenClaw falla de forma anticipada si la ruta efectiva no está declarada
como compatible con Codex, el plugin está deshabilitado, el app-server es demasiado antiguo o el
app-server no puede iniciarse.

## Política del app-server

De forma predeterminada, el plugin inicia localmente el binario de Codex gestionado por OpenClaw con
transporte stdio. Establezca `appServer.command` solo para ejecutar intencionadamente un
ejecutable diferente. Codex clasifica el transporte WebSocket como experimental
y no compatible; úselo únicamente para pruebas que no sean de producción con un app-server
que ya se esté ejecutando en otro lugar:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            transport: "websocket",
            url: "ws://gateway-host:39175",
            authToken: "${CODEX_APP_SERVER_TOKEN}",
          },
        },
      },
    },
  },
}
```

Las sesiones locales del app-server mediante stdio adoptan de forma predeterminada la
postura del operador local de confianza: `approvalPolicy: "never"`, `approvalsReviewer: "user"` y
`sandbox: "danger-full-access"`. Si los requisitos locales de Codex no permiten esa
postura YOLO implícita, OpenClaw selecciona en su lugar permisos de Guardian
permitidos. Cuando un entorno aislado de OpenClaw está activo para la sesión, OpenClaw
deshabilita el Code Mode nativo de Codex, los servidores MCP del usuario y la ejecución de
plugins respaldados por aplicaciones durante ese turno, en lugar de depender del aislamiento
de Codex en el host. El acceso al shell pasa en su lugar por herramientas dinámicas respaldadas
por el entorno aislado de OpenClaw, como `sandbox_exec` y `sandbox_process`, cuando
están disponibles las herramientas normales de ejecución y procesos.

Use el modo de ejecución normalizado de OpenClaw para la revisión automática nativa de Codex antes de
escapar del entorno aislado o conceder permisos adicionales:

```json5
{
  tools: {
    exec: {
      mode: "auto",
    },
  },
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
}
```

Para las sesiones del app-server de Codex, `tools.exec.mode: "auto"` se asigna a aprobaciones
revisadas por Guardian de Codex: normalmente `approvalPolicy: "on-request"`,
`approvalsReviewer: "auto_review"` y `sandbox: "workspace-write"` cuando
los requisitos locales permiten esos valores. En `tools.exec.mode: "auto"`,
OpenClaw no conserva las anulaciones heredadas no seguras de Codex `approvalPolicy: "never"` ni
`sandbox: "danger-full-access"`; use `tools.exec.mode: "full"` para
adoptar intencionadamente una postura de Codex sin aprobaciones. El preajuste heredado
`plugins.entries.codex.config.appServer.mode: "guardian"` sigue
funcionando, pero `tools.exec.mode: "auto"` es la superficie normalizada de OpenClaw.

Para consultar la comparación entre modos con las aprobaciones de ejecución del host y los
permisos ACPX, consulte [Modos de permisos](/es/tools/permission-modes). Para conocer todos los
campos del app-server, el orden de autenticación, el aislamiento del entorno y el comportamiento
de los tiempos de espera, consulte la [Referencia del arnés de Codex](/es/plugins/codex-harness-reference).

## Comandos y diagnósticos

El plugin `codex` registra `/codex` como comando de barra diagonal en cualquier canal que
admita comandos de texto de OpenClaw.

La ejecución y el control nativos requieren un propietario o un cliente de Gateway
`operator.admin`: vincular o reanudar hilos, enviar o detener turnos,
cambiar el modelo, el modo rápido o el estado de permisos, compactar o revisar y
desvincular una asociación. Los demás remitentes autorizados conservan comandos de solo lectura
para inspeccionar el estado, la ayuda, la cuenta, el modelo, el hilo, el objetivo nativo, el
servidor MCP, la skill y la vinculación.

Formas habituales:

- `/codex status` comprueba la conectividad del app-server, los modelos, la cuenta, los límites de
  uso, los servidores MCP y las skills.
- `/codex models` enumera los modelos activos del app-server de Codex.
- `/codex threads [filter]` enumera los hilos recientes del app-server de Codex.
- `/codex goal` lee o actualiza el objetivo nativo de Codex del hilo adjunto. La continuación automática de objetivos de Codex permanece deshabilitada; OpenClaw todavía no controla turnos de seguimiento autónomos.
- `/codex resume <thread-id>` asocia la sesión actual de OpenClaw con un
  hilo de Codex existente.
- `/codex bind [thread-id] [--cwd <path>] [--model <model>] [--provider <provider>]`
  asocia el chat actual.
- `/codex detach` (o `/codex unbind`) desvincula la asociación actual.
- `/codex binding` describe la asociación actual.
- `/codex stop` detiene el turno activo; `/codex steer <text>` lo dirige.
- `/codex model <model>`, `/codex fast [on|off|status]` y
  `/codex permissions [default|yolo|status]` cambian el estado de cada conversación.
- `/codex compact` solicita al app-server de Codex que compacte el hilo adjunto.
- `/codex review` inicia la revisión nativa de Codex para el hilo adjunto.
- `/codex diagnostics [note]` solicita confirmación antes de enviar comentarios de Codex sobre el
  hilo adjunto.
- `/codex account` muestra el estado de la cuenta y de los límites de uso.
- `/codex mcp` enumera el estado de los servidores MCP del app-server de Codex.
- `/codex skills` enumera las skills del app-server de Codex.
- `/codex plugins list`, `/codex plugins enable <name>` y
  `/codex plugins disable <name>` gestionan los plugins nativos de Codex configurados.
- `/codex computer-use [status|install]` gestiona Computer Use de Codex.
- `/codex help` enumera el árbol de comandos completo.

Para la mayoría de los informes de soporte, comience con `/diagnostics [note]` en la
conversación donde se produjo el error. Esto crea un informe de diagnóstico de Gateway
y, en las sesiones del arnés de Codex, solicita aprobación para enviar el
paquete correspondiente de comentarios de Codex. Consulte
[Exportación de diagnósticos](/es/gateway/diagnostics) para conocer el modelo de privacidad y el
comportamiento de los chats grupales. Use `/codex diagnostics [note]` únicamente cuando desee
específicamente cargar los comentarios de Codex correspondientes al hilo adjunto en ese momento
sin el paquete completo de diagnósticos de Gateway.

### Inspeccionar localmente los hilos de Codex

A menudo, la forma más rápida de inspeccionar una ejecución defectuosa de Codex es abrir directamente el
hilo nativo de Codex:

```bash
codex resume <thread-id>
```

Obtenga el identificador del hilo de la respuesta completada de `/diagnostics`, `/codex binding`
o `/codex threads [filter]`.

Para conocer el mecanismo de carga y los límites de diagnóstico en el nivel del entorno de ejecución, consulte
[Entorno de ejecución del arnés de Codex](/es/plugins/codex-harness-runtime#codex-feedback-upload).

### Orden de autenticación

En el directorio de inicio predeterminado de cada agente, la autenticación se selecciona en este orden:

1. Perfiles de autenticación de OpenAI ordenados para el agente, preferiblemente en
   `auth.order.openai`. Ejecute `openclaw doctor --fix` para migrar los identificadores de
   perfiles de autenticación heredados de Codex y el orden de autenticación heredado de Codex.
2. La cuenta existente del app-server en el directorio de inicio de Codex de ese agente.
3. Solo para inicios locales del app-server mediante stdio, `CODEX_API_KEY` y después
   `OPENAI_API_KEY`, cuando no hay ninguna cuenta del app-server y aún se requiere
   autenticación de OpenAI.

Cuando OpenClaw detecta un perfil de autenticación de Codex basado en una suscripción de ChatGPT,
elimina `CODEX_API_KEY` y `OPENAI_API_KEY` del proceso secundario de Codex
iniciado. Esto mantiene disponibles las claves de API del nivel de Gateway para embeddings o
modelos directos de OpenAI, sin hacer que los turnos nativos del app-server de Codex se
facturen accidentalmente mediante la API. Los perfiles explícitos de clave de API de Codex y la
alternativa local de clave de entorno mediante stdio usan el inicio de sesión del app-server en lugar de
heredar el entorno del proceso secundario. Las conexiones WebSocket al app-server no reciben la
alternativa de clave de API del entorno de Gateway; use un perfil de autenticación explícito o la
cuenta propia del app-server remoto.

Si un perfil de suscripción alcanza un límite de uso de Codex, OpenClaw registra la
hora de restablecimiento cuando Codex la proporciona e intenta usar el siguiente perfil de autenticación
ordenado para la misma ejecución de Codex. Una vez transcurrida la hora de restablecimiento, el perfil de
suscripción vuelve a ser apto sin cambiar el modelo `openai/gpt-*`
seleccionado ni el entorno de ejecución de Codex.

Cuando hay plugins nativos de Codex configurados, OpenClaw instala o actualiza
esos plugins mediante el app-server conectado antes de exponer al hilo de Codex las
aplicaciones propiedad de los plugins. `app/list` sigue siendo la fuente de verdad para los identificadores
de aplicaciones, la accesibilidad y los metadatos, pero OpenClaw controla la decisión de
habilitación por hilo: si la política permite una aplicación accesible de la lista, OpenClaw
envía `thread/start.config.apps[appId].enabled = true` aunque `app/list`
indique en ese momento que la aplicación está deshabilitada. Esta ruta no inventa instalaciones de
aplicaciones para identificadores desconocidos; OpenClaw solo activa plugins del marketplace
con `plugin/install` y después actualiza el inventario.

### Aislamiento del entorno

Para los inicios locales del app-server mediante stdio, OpenClaw establece `CODEX_HOME` en un
directorio por agente para que la configuración, los archivos de autenticación y cuenta, la caché y los datos
de plugins, así como el estado nativo de los hilos de Codex, no lean ni escriban de forma predeterminada en el
`~/.codex` personal del operador. OpenClaw conserva el `HOME` normal del proceso;
los subprocesos ejecutados por Codex pueden seguir encontrando la configuración y los tokens del directorio de inicio
del usuario, y Codex puede descubrir entradas compartidas de `$HOME/.agents/skills` y
`$HOME/.agents/plugins/marketplace.json`. Con
`appServer.homeScope: "user"`, OpenClaw usa en su lugar el directorio de inicio nativo de Codex del usuario
y su cuenta existente sin inyectar un perfil de autenticación de OpenClaw.

Si un despliegue necesita aislamiento adicional del entorno, añada esas
variables a `appServer.clearEnv`:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            clearEnv: ["CODEX_API_KEY", "OPENAI_API_KEY"],
          },
        },
      },
    },
  },
}
```

`appServer.clearEnv` solo afecta al proceso secundario del app-server de Codex
iniciado. OpenClaw elimina `CODEX_HOME` y `HOME` de esta lista durante
la normalización del inicio local: `CODEX_HOME` sigue apuntando al ámbito seleccionado
del agente o del usuario, y `HOME` sigue heredándose para que los subprocesos puedan usar
el estado normal del directorio de inicio del usuario.

### Herramientas dinámicas y búsqueda web

De forma predeterminada, las herramientas dinámicas de Codex usan la carga `searchable`. Normalmente, OpenClaw
no expone herramientas dinámicas que dupliquen las operaciones nativas de Codex en el espacio de trabajo:
`read`, `write`, `edit`, `apply_patch`, `exec`, `process`, `update_plan`,
`get_goal`, `create_goal`, `update_goal`, `tool_call`, `tool_describe`,
`tool_search` y `tool_search_code`. Las operaciones de objetivos permanecen en Codex,
por lo que OpenClaw no proyecta un segundo almacén de objetivos en los turnos de Codex. La mayoría de
las demás herramientas de integración de OpenClaw, como mensajería, contenido multimedia, cron,
navegador, nodos, Gateway y `heartbeat_respond`, están disponibles mediante
la búsqueda de herramientas de Codex en el espacio de nombres `openclaw`, lo que mantiene más reducido el
contexto inicial del modelo. La alternativa de shell para turnos restringidos es la excepción para
`exec` y `process` cuando una lista de permitidos finita deshabilita el Code Mode nativo;
las listas de permitidos del entorno de ejecución y `codexDynamicToolsExclude` siguen aplicándose.

Las herramientas marcadas como `catalogMode: "direct-only"`, incluida la herramienta `computer`
de OpenClaw, usan en su lugar el espacio de nombres `openclaw_direct`. Codex trata ese espacio de nombres
como `DirectModelOnly`, por lo que esas herramientas permanecen visibles directamente para el modelo en los hilos
normales y exclusivos de Code Mode, en lugar de atravesar llamadas anidadas `tools.*` de Code Mode.

La búsqueda web usa de forma predeterminada la herramienta alojada `web_search` de Codex cuando la búsqueda está
habilitada y no se ha seleccionado ningún proveedor gestionado. La búsqueda alojada nativa y
la herramienta dinámica gestionada `web_search` de OpenClaw son mutuamente excluyentes para que
la búsqueda gestionada no pueda eludir las restricciones nativas de dominio. OpenClaw usa la
herramienta gestionada cuando la búsqueda alojada no está disponible, se ha deshabilitado explícitamente o
se ha sustituido por un proveedor gestionado seleccionado. OpenClaw mantiene deshabilitada la
extensión independiente `web.run` de Codex porque el tráfico de producción del app-server rechaza
su espacio de nombres `web` definido por el usuario. `tools.web.search.enabled: false`
deshabilita ambas rutas, al igual que las ejecuciones solo de LLM con las herramientas deshabilitadas. Codex trata
`"cached"` como una preferencia y la resuelve como acceso externo activo para
los turnos sin restricciones del app-server. La alternativa gestionada automática falla de forma segura cuando
se establecen `allowedDomains` nativos, de modo que no se pueda eludir la lista de permitidos.
Los cambios persistentes de la política de búsqueda efectiva rotan el hilo de Codex vinculado
antes del siguiente turno; las restricciones transitorias de cada turno usan un hilo temporal
restringido y conservan la vinculación existente para reanudarla más adelante.

`sessions_yield`, `sessions_spawn` y las respuestas de origen exclusivas de la herramienta de mensajes permanecen
directas porque son contratos de control de turnos o delegación. Las directrices siguen
prefiriendo el `spawn_agent` nativo de Codex como la superficie principal de subagentes de Codex,
mientras que la delegación explícita de OpenClaw o ACP sigue siendo invocable directamente mediante
`sessions_spawn`. En el modo de código de Codex, los resultados genéricos de herramientas
dinámicas de OpenClaw son texto JSON en lugar de objetos JavaScript, por lo que se deben analizar
los resultados que parezcan JSON antes de leer los campos. Codex también serializa las llamadas
dinámicas anidadas; envíe varias llamadas `sessions_spawn` en un bucle acotado en lugar
de esperar que `Promise.all` las inicie simultáneamente. Los procesos secundarios ya aceptados
aún pueden solaparse mientras se envían llamadas posteriores. Consulte
[Swarm](/es/tools/swarm#use-swarm-from-other-harnesses) para ver un patrón completo.
Las instrucciones de colaboración de Heartbeat
indican a Codex que busque `heartbeat_respond` antes de finalizar un turno de Heartbeat
cuando la herramienta aún no esté cargada.

Establezca `codexDynamicToolsLoading: "direct"` únicamente al conectarse a un servidor de aplicaciones
Codex personalizado que no pueda buscar herramientas dinámicas diferidas o al
depurar la carga útil completa de herramientas.

### Campos de configuración

Campos de nivel superior compatibles con el Plugin de Codex:

| Campo                      | Valor predeterminado        | Significado                                                                                  |
| -------------------------- | -------------- | ---------------------------------------------------------------------------------------- |
| `codexDynamicToolsLoading` | `"searchable"` | Use `"direct"` para colocar las herramientas dinámicas de OpenClaw directamente en el contexto inicial de herramientas de Codex. |
| `codexDynamicToolsExclude` | `[]`           | Nombres adicionales de herramientas dinámicas de OpenClaw que se omitirán en los turnos del servidor de aplicaciones Codex.              |
| `codexPlugins`             | deshabilitado       | Compatibilidad nativa de plugins/aplicaciones de Codex con plugins seleccionados migrados e instalados desde el código fuente.           |
| `sessionCatalog`           | habilitado        | Detección en la barra lateral de sesiones nativas de Codex en este Gateway y en los nodos emparejados aptos.   |
| `supervision`              | deshabilitado       | Política de transcripción y control de escritura de sesiones nativas orientada al agente.                         |

Campos `appServer` compatibles:

| Campo                                         | Valor predeterminado                                                | Significado                                                                                                                                                                                                                                                                                                                                                                                         |
| --------------------------------------------- | ------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `transport`                                   | `"stdio"`                                              | `"stdio"` inicia Codex; `"unix"` explícito se conecta al socket de control local; `"websocket"` se conecta a `url`.                                                                                                                                                                                                                                                                                |
| `homeScope`                                   | `"agent"`                                              | `"agent"` aísla el estado ordinario del arnés para cada agente de OpenClaw. `"user"` es una adhesión explícita que comparte el `$CODEX_HOME` o `~/.codex` nativo, utiliza la autenticación nativa y habilita la gestión de hilos solo para el propietario. El ámbito de usuario admite stdio local o transporte Unix. Para la conexión de supervisión independiente, un valor sin establecer se resuelve como `"user"` para stdio o Unix y como `"agent"` para WebSocket.     |
| `command`                                     | binario de Codex administrado                                   | Ejecutable para el transporte stdio. Déjelo sin establecer para utilizar el binario administrado; establézcalo únicamente para una sustitución explícita.                                                                                                                                                                                                                                                                                    |
| `args`                                        | `["app-server", "--listen", "stdio://"]`               | Argumentos para el transporte stdio.                                                                                                                                                                                                                                                                                                                                                                  |
| `url`                                         | sin establecer                                                  | URL del App Server de WebSocket o URL `unix://`. Una ruta Unix explícita vacía selecciona el socket de control canónico del directorio personal del usuario.                                                                                                                                                                                                                                                                          |
| `authToken`                                   | sin establecer                                                  | Token de portador para el transporte WebSocket. Acepta una cadena literal o SecretInput, como `${CODEX_APP_SERVER_TOKEN}`.                                                                                                                                                                                                                                                                              |
| `headers`                                     | `{}`                                                   | Encabezados WebSocket adicionales. Los valores de los encabezados aceptan cadenas literales o valores SecretInput, por ejemplo, `x-codex-client-session-token: "${CODEX_CLIENT_SESSION_TOKEN}"`.                                                                                                                                                                                                                               |
| `clearEnv`                                    | `[]`                                                   | Nombres de variables de entorno adicionales que se eliminan del proceso app-server stdio iniciado después de que OpenClaw construye su entorno heredado. OpenClaw conserva el `CODEX_HOME` seleccionado y el `HOME` heredado para las ejecuciones locales.                                                                                                                                                                           |
| `codeModeOnly`                                | `false`                                                | Habilita la superficie de herramientas exclusiva del modo de código de Codex. Las herramientas dinámicas ordinarias de OpenClaw siguen disponibles mediante llamadas `tools.*` anidadas; las herramientas `openclaw_direct` permanecen visibles directamente para el modelo.                                                                                                                                                                                                             |
| `remoteWorkspaceRoot`                         | sin establecer                                                  | Raíz remota del espacio de trabajo del app-server de Codex. Cuando se establece, OpenClaw infiere la raíz del espacio de trabajo local a partir del espacio de trabajo resuelto de OpenClaw, conserva el sufijo del cwd actual bajo esta raíz remota y envía a Codex únicamente el cwd final del app-server. Si el cwd está fuera de la raíz resuelta del espacio de trabajo de OpenClaw, OpenClaw aplica un cierre seguro en lugar de enviar una ruta local del Gateway al app-server remoto. |
| `requestTimeoutMs`                            | `60000`                                                | Tiempo de espera para las llamadas del plano de control del app-server.                                                                                                                                                                                                                                                                                                                                                     |
| `turnCompletionIdleTimeoutMs`                 | `60000`                                                | Intervalo de inactividad después de que Codex acepta un turno o después de una solicitud del app-server limitada al turno mientras OpenClaw espera `turn/completed`.                                                                                                                                                                                                                                                                    |
| `turnAssistantCompletionIdleTimeoutMs`        | `10000`                                                | Intervalo de inactividad después de que un elemento final o no perteneciente a comentarios del asistente, o una finalización sin procesar del asistente previa a una herramienta, prepara la liberación de la salida del asistente mientras OpenClaw aún espera `turn/completed`. Aumentarlo concede a Codex más tiempo para emitir `turn/completed` antes de que OpenClaw interrumpa y libere el canal de la sesión.                                                                                            |
| `postToolRawAssistantCompletionIdleTimeoutMs` | `300000`                                               | Protección de inactividad de finalización y progreso utilizada después de una transferencia a una herramienta, la finalización de una herramienta nativa, el progreso sin procesar del asistente posterior a una herramienta, la finalización del razonamiento sin procesar o el progreso del razonamiento mientras OpenClaw espera `turn/completed`. Utilícela para cargas de trabajo de confianza o pesadas en las que la síntesis posterior a la herramienta pueda permanecer legítimamente inactiva durante más tiempo que el límite de liberación final del asistente.                                |
| `mode`                                        | `"yolo"` salvo que los requisitos locales de Codex no permitan YOLO | Configuración predefinida para la ejecución YOLO o revisada por un guardián. Los requisitos de stdio local que omiten `danger-full-access`, la aprobación `never` o el revisor `user` hacen que el valor predeterminado implícito sea guardián.                                                                                                                                                                                                           |
| `approvalPolicy`                              | `"never"` o una política de aprobación del guardián permitida       | Política de aprobación nativa de Codex enviada al iniciar, reanudar o realizar un turno del hilo. Los valores predeterminados del guardián prefieren `"on-request"` cuando está permitido.                                                                                                                                                                                                                                                                            |
| `sandbox`                                     | `"danger-full-access"` o un entorno aislado del guardián permitido  | Modo de entorno aislado nativo de Codex enviado al iniciar o reanudar el hilo. Los valores predeterminados del guardián prefieren `"workspace-write"` cuando está permitido; de lo contrario, `"read-only"`. Cuando hay un entorno aislado de OpenClaw activo, los turnos `danger-full-access` utilizan `workspace-write` de Codex con el acceso a la red derivado de la configuración de salida del entorno aislado de OpenClaw.                                                                                     |
| `approvalsReviewer`                           | `"user"` o un revisor del guardián permitido               | Utilice `"auto_review"` para permitir que Codex revise las solicitudes de aprobación nativas cuando esté permitido; de lo contrario, `guardian_subagent` o `user`. `guardian_subagent` sigue siendo un alias heredado.                                                                                                                                                                                                                              |
| `serviceTier`                                 | sin establecer                                                  | Nivel de servicio opcional del app-server de Codex. `"priority"` habilita el enrutamiento en modo rápido, `"flex"` solicita procesamiento flexible, `null` elimina la sustitución y el valor heredado `"fast"` se acepta como `"priority"`.                                                                                                                                                                                                 |
| `networkProxy`                                | deshabilitado                                               | Habilita la red del perfil de permisos de Codex para los comandos del app-server. OpenClaw define la configuración `permissions.<profile>.network` seleccionada y la selecciona mediante `default_permissions` en lugar de enviar `sandbox`.                                                                                                                                                                             |
| `experimental.sandboxExecServer`              | `false`                                                | Adhesión a la versión preliminar que registra un entorno de Codex respaldado por el entorno aislado de OpenClaw con el app-server de Codex compatible, de modo que la ejecución nativa de Codex pueda realizarse dentro del entorno aislado activo de OpenClaw.                                                                                                                                                                                                            |

`appServer.networkProxy` es explícito porque cambia el contrato del entorno aislado
de Codex. Cuando se habilita, OpenClaw también establece `features.network_proxy.enabled`
y `default_permissions` en la configuración del hilo de Codex para que el perfil
de permisos generado pueda iniciar la red administrada por Codex. De forma predeterminada, OpenClaw
genera un nombre de perfil `openclaw-network-<fingerprint>` resistente a colisiones
a partir del cuerpo del perfil; use `profileName` solo cuando se requiera un nombre local
estable.

```json5
{
  plugins: {
    entries: {
      codex: {
        config: {
          appServer: {
            sandbox: "workspace-write",
            networkProxy: {
              enabled: true,
              domains: {
                "api.openai.com": "allow",
                "blocked.example.com": "deny",
              },
              unixSockets: {
                "/tmp/proxy.sock": "allow",
                "/tmp/blocked.sock": "none",
              },
              allowUpstreamProxy: true,
              proxyUrl: "http://127.0.0.1:3128",
            },
          },
        },
      },
    },
  },
}
```

Si el entorno de ejecución normal del servidor de aplicaciones fuera `danger-full-access`, al habilitar
`networkProxy` se utiliza acceso al sistema de archivos con el estilo del espacio de trabajo para el perfil
de permisos generado: la aplicación de red administrada por Codex es una red
aislada, por lo que un perfil de acceso completo no protegería el tráfico saliente.
Las entradas de dominio usan `allow` o `deny`; las entradas de sockets Unix usan los valores
`allow` o `none` de Codex.

### Tiempos de espera de llamadas dinámicas a herramientas

Las llamadas dinámicas a herramientas propiedad de OpenClaw tienen límites independientes de
`appServer.requestTimeoutMs`: las solicitudes `item/tool/call` de Codex usan de forma predeterminada un
supervisor de OpenClaw de 90 segundos. Un argumento `timeoutMs` positivo por llamada
amplía o reduce el presupuesto de esa herramienta específica, con un límite de 600000 ms.
La herramienta `image_generate` usa `agents.defaults.mediaModels.image.timeoutMs`
cuando la llamada a la herramienta no proporciona su propio tiempo de espera, o un valor predeterminado
de generación de imágenes de 120 segundos en caso contrario. La herramienta de comprensión multimedia `image`
usa el valor `timeoutSeconds` de la entrada `tools.media.models[]` seleccionada compatible con imágenes o su valor multimedia predeterminado de 60 segundos; para
la comprensión de imágenes, ese tiempo de espera se aplica a la propia solicitud y no se
reduce por el trabajo de preparación previo. Al agotarse el tiempo, OpenClaw cancela la señal
de la herramienta cuando es compatible y devuelve a Codex una respuesta de herramienta dinámica fallida
para que el turno pueda continuar en lugar de dejar la sesión en `processing`.
Este supervisor es el presupuesto dinámico externo de `item/tool/call`; los tiempos de espera de solicitud
específicos del proveedor se ejecutan dentro de esa llamada y conservan su propia semántica de tiempo de espera.

Después de que Codex acepta un turno y de que OpenClaw responde a una solicitud
del servidor de aplicaciones limitada al turno, el entorno espera que Codex progrese en el turno actual
y finalmente termine el turno nativo con `turn/completed`. Si el
servidor de aplicaciones queda inactivo durante `appServer.turnCompletionIdleTimeoutMs`, OpenClaw
intenta interrumpir el turno de Codex, registra un diagnóstico de tiempo de espera y
libera el canal de sesión de OpenClaw para que los mensajes de chat posteriores no queden
en cola detrás de un turno nativo obsoleto. La mayoría de las notificaciones no terminales del
mismo turno desactivan ese breve supervisor porque Codex ha demostrado que el turno
sigue activo.

Las transferencias de herramientas usan un presupuesto de inactividad posterior a la herramienta más largo: después de que OpenClaw devuelve una
respuesta `item/tool/call`, después de que finalizan elementos de herramientas nativas como
`commandExecution`, después de finalizaciones sin procesar de `custom_tool_call_output`
y después del progreso sin procesar del asistente posterior a la herramienta, las finalizaciones de razonamiento sin procesar
o el progreso del razonamiento. La protección usa
`appServer.postToolRawAssistantCompletionIdleTimeoutMs` cuando está configurado y
usa cinco minutos de forma predeterminada en caso contrario; ese mismo presupuesto también amplía el
supervisor de progreso durante el intervalo silencioso de síntesis antes de que Codex emita el
siguiente evento del turno actual. Las notificaciones globales del servidor de aplicaciones, como
las actualizaciones de límites de frecuencia, no reinician el progreso de inactividad del turno. Las finalizaciones de razonamiento,
las finalizaciones de `agentMessage` de comentarios y el razonamiento sin procesar previo a la herramienta o
el progreso del asistente pueden ir seguidos de una respuesta final automática, por lo que usan
la protección de respuesta posterior al progreso en lugar de liberar el canal de sesión
inmediatamente.

Solo los elementos `agentMessage` finalizados finales o que no sean comentarios y las finalizaciones sin procesar
del asistente previas a la herramienta activan la liberación de salida del asistente: si Codex queda
inactivo sin `turn/completed`, OpenClaw intenta interrumpir el turno nativo
y libera el canal de sesión. Si otra supervisión del turno gana esa carrera de liberación,
OpenClaw sigue aceptando el elemento finalizado del asistente una vez que no
queda activa ninguna solicitud nativa, elemento o finalización de herramienta dinámica y la
liberación de salida del asistente sigue perteneciendo al último elemento finalizado, sin
ninguna finalización de elemento posterior. Esto puede conservar la respuesta final después de
completar el trabajo de herramientas sin repetir el turno. Los deltas parciales del asistente,
las respuestas anteriores obsoletas y las finalizaciones posteriores vacías no cumplen los requisitos.

Los fallos del servidor de aplicaciones stdio que pueden repetirse de forma segura, incluidos los tiempos de espera
de inactividad al completar el turno sin pruebas del asistente, herramientas, elementos activos
o efectos secundarios, se reintentan una vez en un nuevo intento del servidor de aplicaciones. Los tiempos de espera
no seguros retiran de todos modos el cliente del servidor de aplicaciones bloqueado y liberan el
canal de sesión de OpenClaw; también eliminan la asociación obsoleta del hilo nativo en lugar de
repetirse automáticamente. Los tiempos de espera de supervisión de finalización muestran texto
específico de Codex: los casos que pueden repetirse de forma segura indican que la respuesta puede estar incompleta,
mientras que los casos no seguros indican al usuario que verifique el estado actual antes de volver a intentarlo.
Los diagnósticos públicos de tiempo de espera incluyen campos estructurales como el último método
de notificación del servidor de aplicaciones, el identificador, tipo y rol del elemento de respuesta sin procesar del asistente,
los recuentos de solicitudes y elementos activos y el estado de supervisión activado; cuando la última notificación es un
elemento de respuesta sin procesar del asistente, también incluyen una vista previa limitada del texto
del asistente. No incluyen el contenido sin procesar de las instrucciones ni de las herramientas.

### Sustituciones de entorno para pruebas locales

- `OPENCLAW_CODEX_APP_SERVER_BIN` omite el binario administrado cuando
  `appServer.command` no está establecido.
- `OPENCLAW_CODEX_APP_SERVER_ARGS`
- `OPENCLAW_CODEX_APP_SERVER_MODE=yolo|guardian`
- `OPENCLAW_CODEX_APP_SERVER_APPROVAL_POLICY`
- `OPENCLAW_CODEX_APP_SERVER_SANDBOX`

Se eliminó `OPENCLAW_CODEX_APP_SERVER_GUARDIAN=1`. Use
`plugins.entries.codex.config.appServer.mode: "guardian"` en su lugar, o
`OPENCLAW_CODEX_APP_SERVER_MODE=guardian` para pruebas locales puntuales. Se prefiere la configuración
para despliegues reproducibles porque mantiene el comportamiento del plugin
en el mismo archivo revisado que el resto de la configuración del entorno de Codex.

## Plugins nativos de Codex

La compatibilidad con plugins nativos de Codex usa las capacidades propias de aplicaciones y plugins
del servidor de aplicaciones de Codex en el mismo hilo de Codex que el turno del entorno de OpenClaw. OpenClaw
no convierte los plugins de Codex en herramientas dinámicas sintéticas `codex_plugin_*` de OpenClaw.

`codexPlugins` afecta solo a las sesiones que seleccionan el entorno nativo de Codex.
No tiene efecto en las ejecuciones del entorno integrado, las ejecuciones normales del proveedor OpenAI, las asociaciones
de conversaciones ACP ni otros entornos.

Configuración mínima migrada:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          codexPlugins: {
            enabled: true,
            allow_destructive_actions: true,
            plugins: {
              "google-calendar": {
                enabled: true,
                marketplaceName: "openai-curated",
                pluginName: "google-calendar",
              },
            },
          },
        },
      },
    },
  },
}
```

La configuración de aplicaciones del hilo se calcula cuando OpenClaw establece una sesión
del entorno de Codex o sustituye una asociación obsoleta del hilo de Codex; no se vuelve a calcular en
cada turno. Después de cambiar `codexPlugins`, use `/new`, `/reset` o reinicie
el Gateway para que las futuras sesiones del entorno de Codex comiencen con el conjunto de aplicaciones
actualizado.

Para conocer los requisitos de migración, el inventario de aplicaciones, la política de acciones destructivas,
las solicitudes de información y los diagnósticos de plugins nativos, consulte
[Plugins nativos de Codex](/es/plugins/codex-native-plugins).

El acceso a aplicaciones y plugins del lado de OpenAI está controlado por la cuenta de Codex
con sesión iniciada y, en los espacios de trabajo Business y Enterprise/Edu, por los controles de aplicaciones
del espacio de trabajo. Consulte
[Uso de Codex con su plan de ChatGPT](https://help.openai.com/en/articles/11369540-using-codex-with-your-chatgpt-plan)
para obtener una descripción general de OpenAI sobre los controles de cuentas y espacios de trabajo.

## Uso del equipo

El uso del equipo tiene su propia guía de configuración:
[Uso del equipo con Codex](/es/plugins/codex-computer-use).

Versión breve: OpenClaw no incorpora la aplicación de control del escritorio ni ejecuta
por sí mismo acciones en el escritorio. Prepara el servidor de aplicaciones de Codex, verifica que el
servidor MCP `computer-use` esté disponible y, a continuación, permite que Codex gestione las llamadas nativas
a herramientas MCP durante los turnos en modo Codex.

## Límites del entorno de ejecución

El entorno de Codex solo cambia el ejecutor del agente integrado de bajo nivel.

- Se admiten las herramientas dinámicas de OpenClaw. Codex solicita a OpenClaw que ejecute
  esas herramientas, por lo que OpenClaw permanece en la ruta de ejecución.
- El shell, los parches, MCP y las herramientas de aplicaciones nativas de Codex son gestionados por Codex.
  OpenClaw puede observar o bloquear determinados eventos nativos mediante la
  retransmisión compatible, pero no reescribe los argumentos de las herramientas nativas.
- Codex gestiona la Compaction nativa. OpenClaw mantiene un duplicado de la transcripción para
  el historial del canal, la búsqueda, `/new`, `/reset` y futuros cambios de modelo o entorno,
  pero no sustituye la Compaction de Codex por un resumidor de OpenClaw o
  del motor de contexto.
- La generación multimedia, la comprensión multimedia, TTS, las aprobaciones y la salida de las herramientas
  de mensajería siguen pasando por la configuración correspondiente del proveedor o modelo de OpenClaw.
- `tool_result_persist` se aplica a los resultados de herramientas de transcripción propiedad de OpenClaw,
  no a los registros de resultados de herramientas nativas de Codex.

Para obtener información sobre las capas de enlaces, las superficies V1 compatibles, la gestión de permisos nativos, la dirección
de colas, los mecanismos de carga de comentarios de Codex y los detalles de Compaction, consulte
[Entorno de ejecución de Codex](/es/plugins/codex-harness-runtime).

## Solución de problemas

**Codex no aparece como un proveedor `/model` normal:** es lo esperado para las configuraciones
nuevas. Seleccione un modelo `openai/gpt-*`, habilite
`plugins.entries.codex.enabled` y compruebe si `plugins.allow` excluye
`codex`.

**OpenClaw usa el entorno integrado en lugar de Codex:** confirme que la ruta efectiva
sea una ruta oficial HTTPS exacta de Platform Responses o ChatGPT Responses,
que no tenga una sustitución de solicitud definida y que el plugin de Codex esté instalado y
habilitado. El prefijo `openai/gpt-*` por sí solo no basta. Para obtener una prueba estricta durante
las pruebas, establezca `agentRuntime.id: "codex"` en el proveedor o modelo; Codex forzado falla
en lugar de recurrir a una alternativa cuando la ruta o el entorno son incompatibles.

**El entorno OpenAI Codex recurre a la ruta de clave de API:** recopile un extracto
censurado del Gateway que muestre el modelo, el entorno de ejecución, el proveedor seleccionado y el
fallo. Solicite a los colaboradores afectados que ejecuten este comando de solo lectura en su
host de OpenClaw:

```bash
(
  pattern='openai/gpt-5\.[45]|openai[-]codex|agentRuntime(\.id)?|harnessRuntime|Runtime: OpenAI Codex|legacy OpenAI Codex prefix|resolveSelectedOpenAIRuntimeProvider|candidateProvider[": ]+openai|status[": ]+401|Incorrect API key|No API key|api-key path|API-key path|OAuth'

  if ls /tmp/openclaw/openclaw-*.log >/dev/null 2>&1; then
    grep -E -i -n "$pattern" /tmp/openclaw/openclaw-*.log 2>/dev/null || true
  else
    journalctl --user -u openclaw-gateway --since today --no-pager 2>/dev/null \
      | grep -E -i "$pattern" || true
  fi
) | sed -E \
    -e 's/(Authorization: Bearer )[A-Za-z0-9._~+\/-]+/\1[REDACTED]/Ig' \
    -e 's/(Bearer )[A-Za-z0-9._~+\/-]+/\1[REDACTED]/Ig' \
    -e 's/(api[_ -]?key[=: ]+)[^ ,}"]+/\1[REDACTED]/Ig' \
    -e 's/(OPENAI_API_KEY[=: ]+)[^ ,}"]+/\1[REDACTED]/Ig' \
    -e 's/sk-[A-Za-z0-9_-]{12,}/sk-[REDACTED]/g' \
    -e 's/[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}/[EMAIL-REDACTED]/g' \
  | tail -200
```

Los extractos útiles suelen incluir `openai/gpt-5.6-sol` o `openai/gpt-5.6-luna`,
`Runtime: OpenAI Codex`, `agentRuntime.id` o `harnessRuntime`,
`candidateProvider: "openai"` y un resultado `401`, `Incorrect API key` o
`No API key`. Una ejecución corregida debería mostrar la ruta OAuth de OpenAI
en lugar de un fallo simple de la clave de API de OpenAI.

**La configuración de referencias de modelos heredados de Codex permanece:** ejecute `openclaw doctor --fix`.
Doctor reescribe las referencias de modelos heredados como `openai/*`, elimina las asignaciones obsoletas de sesión y
del entorno de ejecución de todo el agente, y conserva las anulaciones existentes de perfiles de autenticación.

**El app-server se rechaza:** use un app-server estable de Codex de `0.143.0`
mediante el `0.145.0` incluido. Las versiones preliminares, las versiones con sufijos de compilación y las versiones más recientes
sin validar se rechazan porque OpenClaw valida los esquemas generados
con respecto a la versión del app-server incluida.

**`/codex status` no puede conectarse:** compruebe que el Plugin `codex`
esté habilitado, que `plugins.allow` lo incluya cuando se configure una lista de permitidos
y que cualquier `appServer.command`, `url`, `authToken` o
encabezado personalizado sea válido.

**El app-server de Codex utiliza demasiada memoria:** distinga primero los dos procesos.
OpenClaw ejecuta el app-server local de Codex como un proceso secundario independiente de Rust.
`NODE_OPTIONS=--max-old-space-size=...` solo cambia el montón de V8 de Node.js
del Gateway; no limita ni amplía Codex. Las instalaciones administradas del Gateway ya eligen
un montón de V8 adaptativo, y aumentarlo puede dejar menos memoria del host para Codex. Consulte
[Solución de problemas de memoria del Gateway](/es/gateway/troubleshooting#gateway-exits-during-high-memory-use)
para la presión sobre el Gateway e inspeccione la memoria del host o del contenedor para el proceso secundario de Codex.

El Codex incluido no tiene límite de montón ni de RSS, ni un retraso configurable
para descargarlo por inactividad. Después de que el último cliente cancele la suscripción, un hilo inactivo puede permanecer cargado
hasta 30 minutos. En hosts con recursos limitados, reduzca la concurrencia de subagentes nativos de Codex
antes de aumentar el montón del Gateway:

```json5
{
  plugins: {
    entries: {
      codex: {
        config: {
          appServer: {
            args: ["-c", "agents.max_threads=3", "app-server", "--listen", "stdio://"],
          },
        },
      },
    },
  },
}
```

Esta configuración limita los hilos secundarios nativos del backend
multiagente predeterminado del Codex incluido. Si habilita explícitamente la versión 2 del sistema multiagente de Codex, use
`features.multi_agent_v2.max_concurrent_threads_per_session=3`; el límite de la versión 2
incluye el hilo raíz y no puede combinarse con `agents.max_threads`.
Para proporcionar más margen a Codex, aumente la asignación de memoria
del host, contenedor o cgroup. Un límite estricto del sistema operativo puede finalizar Codex en lugar de aplicarle contrapresión.

**La detección de modelos es lenta:** reduzca
`plugins.entries.codex.config.discovery.timeoutMs` o deshabilite la detección.
Consulte la [Referencia del entorno de Codex](/es/plugins/codex-harness-reference#model-discovery).

**El transporte WebSocket falla de inmediato:** compruebe `appServer.url`,
`authToken`, los encabezados y que el app-server remoto utilice la misma versión del protocolo
del app-server de Codex. El transporte WebSocket de Codex sigue siendo experimental
y no cuenta con soporte; prefiera stdio administrado o el socket de control Unix local.

**Las herramientas nativas de shell o de aplicación de parches se bloquean con `Native hook relay
unavailable`:** el hilo de Codex aún intenta utilizar un identificador de retransmisión de hooks nativos
que OpenClaw ya no tiene registrado. Este es un problema del transporte de hooks
nativos de Codex, no un fallo del backend ACP, del proveedor, de GitHub ni de los comandos de shell.
Inicie una sesión nueva en el chat afectado con `/new` o `/reset`
y vuelva a intentar un comando inofensivo. Si funciona una vez, pero la siguiente llamada a una herramienta nativa
vuelve a fallar, considere `/new` solo como una solución provisional: copie el
prompt en una sesión nueva después de reiniciar el app-server de Codex o el
Gateway de OpenClaw para que se descarten los hilos antiguos y se vuelvan a crear los registros de hooks
nativos.

**Las llamadas a herramientas de Codex crean demasiados procesos de hooks de corta duración:** establezca
`plugins.entries.codex.config.appServer.loopDetectionPreToolUseRelay: false`
y reinicie el Gateway. Esto deshabilita únicamente el subproceso `PreToolUse` de Codex
utilizado para detectar bucles de OpenClaw y su marcador de ausencia de políticas. Las retransmisiones obligatorias de
`before_tool_call` y de políticas de herramientas de confianza permanecen habilitadas.

**Un modelo que no es de Codex utiliza el entorno integrado:** es lo esperado, salvo que la política del entorno de ejecución
del proveedor o del modelo lo dirija a otro entorno. Las referencias simples de proveedores
que no son de OpenAI permanecen en la ruta normal de su proveedor en el modo `auto`.

**Computer Use está instalado, pero las herramientas no se ejecutan:** compruebe
`/codex computer-use status` desde una sesión nueva. Si una herramienta informa de
`Native hook relay unavailable`, use la recuperación de retransmisión de hooks nativos indicada anteriormente.
Consulte [Computer Use de Codex](/es/plugins/codex-computer-use#troubleshooting).

## Relacionado

- [Referencia del entorno de Codex](/es/plugins/codex-harness-reference)
- [Entorno de ejecución de Codex](/es/plugins/codex-harness-runtime)
- [Supervisión de Codex](/es/plugins/codex-supervision)
- [Plugins nativos de Codex](/es/plugins/codex-native-plugins)
- [Computer Use de Codex](/es/plugins/codex-computer-use)
- [Entornos de ejecución de agentes](/es/concepts/agent-runtimes)
- [Proveedores de modelos](/es/concepts/model-providers)
- [Proveedor OpenAI](/es/providers/openai)
- [Ayuda de OpenAI Codex](https://help.openai.com/en/collections/14937394-codex)
- [Plugins de entorno de agentes](/es/plugins/sdk-agent-harness)
- [Hooks de plugins](/es/plugins/hooks)
- [Exportación de diagnósticos](/es/gateway/diagnostics)
- [Estado](/es/cli/status)
- [Pruebas](/es/help/testing-live#live-codex-app-server-harness-smoke)
