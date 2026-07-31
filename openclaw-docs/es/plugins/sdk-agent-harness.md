---
read_when:
    - Está cambiando el runtime del agente integrado o el registro del arnés.
    - Está registrando un entorno de agente desde un plugin incluido o de confianza.
    - Necesita comprender cómo se relaciona el plugin Codex con los proveedores de modelos
sidebarTitle: Agent Harness
summary: Superficie experimental del SDK para plugins que sustituyen el ejecutor de agente integrado de bajo nivel
title: Plugins del entorno de agentes
x-i18n:
    generated_at: "2026-07-26T05:51:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4ff4e41a46ba0074fc6c8bf46da813b58d074f5e6c5c1d236d7ab78e824bdc02
    source_path: plugins/sdk-agent-harness.md
    workflow: 16
---

Un **arnés de agente** es el ejecutor de bajo nivel de un turno preparado de un agente de OpenClaw. No es un proveedor de modelos, ni un canal, ni un registro de herramientas. Para consultar el modelo mental orientado al usuario, véase [Runtimes de agentes](/es/concepts/agent-runtimes).

Utilice esta superficie solo para plugins nativos incluidos o de confianza. El contrato sigue siendo experimental porque los tipos de parámetros reflejan intencionadamente el ejecutor integrado actual.

## Cuándo utilizar un arnés

Registre un arnés de agente cuando una familia de modelos tenga su propio runtime de sesión nativo y el transporte normal de proveedores de OpenClaw sea una abstracción inadecuada:

- un servidor nativo de agentes de programación que gestiona hilos y Compaction
- una CLI o un daemon local que debe transmitir eventos nativos de planificación, razonamiento y herramientas
- un runtime de modelos que necesita su propio identificador de reanudación además de la transcripción de sesión de OpenClaw

**No** registre un arnés únicamente para añadir una nueva API de LLM. Para las API de modelos HTTP o WebSocket normales, cree un [plugin de proveedor](/es/plugins/sdk-provider-plugins).

## Qué sigue gestionando el núcleo

Antes de seleccionar un arnés, OpenClaw ya ha resuelto:

- el proveedor y el modelo
- el estado de autenticación del runtime, salvo que el arnés declare que gestiona la inicialización de la autenticación
- el nivel de razonamiento y el presupuesto de contexto
- el archivo de transcripción o sesión de OpenClaw
- el espacio de trabajo, el entorno aislado y la política de herramientas
- las devoluciones de llamada de respuesta del canal y las devoluciones de llamada de transmisión
- la política de respaldo y cambio en vivo de modelos

Un arnés ejecuta un intento preparado; no elige proveedores, sustituye la entrega del canal ni cambia de modelo de forma silenciosa.

### Inicialización de autenticación gestionada por el arnés

De forma predeterminada, el núcleo resuelve las credenciales del proveedor antes de llamar a un arnés. Un arnés de confianza que pueda autenticarse mediante su propio runtime nativo puede establecer `authBootstrap: "harness"` en su registro estático `AgentHarness`. A continuación, el núcleo omite su inicialización genérica de credenciales del proveedor y el error por falta de credenciales para cada intento reclamado por ese arnés.

El núcleo sigue reenviando un perfil de autenticación de OpenClaw compatible, seleccionado explícitamente u ordenado, y su almacén con ámbito definido cuando existen. El arnés debe resolver ese perfil o sus credenciales nativas antes de emitir solicitudes al modelo, mantener los secretos limitados al intento y mostrar errores de autenticación que permitan actuar. No establezca esta capacidad en un arnés que solo gestione la autenticación algunas veces.

### Artefactos verificados del runtime de configuración

Un arnés local que pueda proporcionar inferencia para la configuración inicial debe certificar la implementación que completó la prueba. Cuando
`params.captureRuntimeArtifact` sea verdadero, devuelva un
`result.runtimeArtifact` opaco con un identificador estable y una huella digital del contenido. Registre una capacidad `runtimeArtifact.validate(...)` correspondiente que vuelva a comprobar esa vinculación sin cargar otro arnés ni analizar plugins no relacionados.

Las continuaciones verificadas de OpenClaw también pasan `params.expectedRuntimeArtifact`.
El arnés debe compararlo con el proceso nativo exacto que adquirió y generar un error antes de iniciar o reanudar un hilo nativo si no coinciden. Los turnos ordinarios del agente omiten ambos campos, por lo que el cálculo de hashes del contenido permanece fuera de la ruta crítica normal de las solicitudes. Los arneses remotos o WebSocket necesitan un contrato de certificación del servidor antes de poder participar; una cadena de versión por sí sola no constituye la identidad de un artefacto.

El intento preparado también incluye `params.runtimePlan`, un paquete de políticas gestionado por OpenClaw para las decisiones del runtime que deben seguir siendo comunes entre OpenClaw y los arneses nativos:

- `runtimePlan.tools.normalize(...)` y `runtimePlan.tools.logDiagnostics(...)`
  para la política de esquemas de herramientas adaptada al proveedor
- `runtimePlan.transcript.resolvePolicy(...)` para la limpieza de transcripciones y
  la política de reparación de llamadas a herramientas
- `runtimePlan.delivery.isSilentPayload(...)` para la `NO_REPLY` compartida y la supresión
  de entrega de contenido multimedia
- `runtimePlan.outcome.classifyRunResult(...)` para la clasificación del respaldo
  de modelos
- `runtimePlan.observability` para los metadatos resueltos del proveedor, modelo y arnés

Los arneses pueden utilizar el plan para tomar decisiones que deban coincidir con el comportamiento de OpenClaw, pero deben tratarlo como estado del intento gestionado por el host: no lo modifique ni lo utilice para cambiar de proveedor o modelo dentro de un turno.

### Contrato de transporte de solicitudes

`supports(ctx)` recibe el transporte del modelo resuelto en `ctx.modelProvider`.
Dos datos sin secretos gestionados por el proveedor describen la ruta seleccionada:

- `runtimePolicy.compatibleIds` enumera los identificadores de runtime que el proveedor declara
  compatibles con esa ruta concreta. La ausencia de una política significa que el proveedor no
  declaró compatibilidad en el nivel de ruta; no es un permiso para presuponer la compatibilidad.
- `requestTransportOverrides: "none"` significa que no debe reproducirse ninguna
  modificación de solicitud del proveedor o modelo definida expresamente. `"present"` significa que existen encabezados definidos expresamente, transporte de autenticación,
  proxy, TLS, comportamiento de servicio local o red privada, o parámetros de
  solicitud. El dato no expone esos valores.

Devuelva `{ supported: false, reason }` cuando el arnés no pueda reproducir el transporte preparado. No deduzca la compatibilidad leyendo la configuración sin procesar después de la selección.
Cuando la preparación de la autenticación genere varias rutas de reintento, un mismo arnés debe admitirlas todas antes del envío. La selección implícita utiliza OpenClaw si ningún plugin puede gestionar el conjunto completo; una selección de plugin explícita o persistente produce un error de forma segura.

## Registrar un arnés

**Importación:** `openclaw/plugin-sdk/agent-harness`

```typescript
import type { AgentHarness } from "openclaw/plugin-sdk/agent-harness";
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

const myHarness: AgentHarness = {
  id: "my-harness",
  label: "Mi arnés de agente nativo",

  supports(ctx) {
    const routeSupportsHarness =
      ctx.modelProvider?.runtimePolicy?.compatibleIds.includes("my-harness") === true;
    const canReproduceRequest = ctx.modelProvider?.requestTransportOverrides !== "present";
    return ctx.provider === "my-provider" && routeSupportsHarness && canReproduceRequest
      ? { supported: true, priority: 100 }
      : { supported: false, reason: "la ruta efectiva no es compatible con el arnés" };
  },

  async runAttempt(params) {
    // Inicie o reanude el hilo nativo.
    // Utilice params.prompt, params.tools, params.images, params.onPartialReply,
    // params.onAgentEvent y los demás campos del intento preparado.
    return await runMyNativeTurn(params);
  },
};

export default definePluginEntry({
  id: "my-native-agent",
  name: "Mi agente nativo",
  description: "Ejecuta los modelos seleccionados mediante un daemon de agente nativo.",
  register(api) {
    api.registerAgentHarness(myHarness);
  },
});
```

`authBootstrap` se ha omitido intencionadamente de este ejemplo genérico. Añada
`authBootstrap: "harness"` solo cuando el arnés cumpla el contrato anterior.

### Ejecución delegada

El propietario de un arnés puede establecer `delegatedExecutionPluginIds` con los identificadores de los plugins de confianza que necesiten ejecutar una sesión existente vinculada a un modelo, como un transporte de voz que continúe una conversación respaldada por Codex. Esto constituye el consentimiento estático del propietario, no una lista de permitidos del núcleo. Mantenga la lista limitada.

Los delegados solo reciben la admisión del trabajo y la ejecución integrada. OpenClaw exige la clave de sesión, la ruta del almacén y el identificador de sesión almacenados exactos; `modelSelectionLocked:
true`; y valores `agentHarnessId` y `agentHarnessRuntimeOverride` coincidentes.
La ejecución queda entonces limitada mediante el propietario del arnés. La creación, modificación, restablecimiento, eliminación y archivado de sesiones, así como la mutación del Gateway, siguen siendo exclusivos del propietario.

## Política de selección

OpenClaw elige un arnés después de resolver el proveedor y el modelo:

1. La política de runtime específica del modelo tiene prioridad.
2. La política de runtime específica del proveedor se aplica a continuación.
3. `auto` pregunta a los arneses registrados si admiten la ruta efectiva resuelta.
   Los prefijos de proveedor o modelo por sí solos nunca seleccionan un arnés.
4. Si ningún arnés registrado coincide, OpenClaw utiliza su runtime integrado.

Los errores de los arneses de plugins se muestran como errores de ejecución. En el modo `auto`, el respaldo integrado solo se aplica cuando ningún arnés de plugin registrado admite el proveedor o modelo resuelto. Una vez que un arnés de plugin ha reclamado una ejecución, OpenClaw no vuelve a ejecutar ese mismo turno mediante otro runtime, porque esto puede cambiar la semántica de autenticación o del runtime, o duplicar efectos secundarios.

La política de runtime configurada sigue determinando el runtime deseado. Un `agentHarnessId` de sesión persistente conserva la propiedad de su transcripción nativa mientras la preparación de la ruta y la autenticación sigue pendiente. Ninguno de ellos convierte una ruta incompatible en compatible: cuando existen los datos preparados, el arnés seleccionado o fijado debe admitirlos, o la ejecución produce un error de forma segura. `/status` muestra el runtime efectivo seleccionado a partir de la política, la propiedad persistente y la compatibilidad de la ruta.
El estado preparado es explícito: si falta `runtimePolicy`, permanece sin declarar en lugar de deducirse de los campos de transporte que estén presentes.
Cuando la autenticación gestionada por el arnés deja varias rutas físicas sin resolver, el dato de compatibilidad preparado es la intersección de sus identificadores de runtime compatibles e informa de modificaciones de solicitud si algún candidato las tiene. Por tanto, un solo candidato sin declarar hace que la compatibilidad nativa quede vacía; `preparedAuth.source: "harness"` es un propietario de autenticación, no un permiso para deducir la compatibilidad de la ruta.

Si el arnés seleccionado resulta inesperado, active el registro de depuración `agents/harness` e inspeccione el registro estructurado `agent harness selected` del Gateway: incluye el identificador del arnés seleccionado, el motivo de la selección, la política de runtime y respaldo y, en el modo `auto`, el resultado de compatibilidad de cada candidato de plugin.

El plugin Codex incluido registra `codex` como identificador de su arnés. El núcleo lo trata como un identificador ordinario de arnés de plugin; los alias específicos de Codex deben residir en el plugin o en la configuración del operador, no en el selector de runtime compartido.

## Asociación de proveedor y arnés

La mayoría de los arneses también deben registrar un proveedor. El proveedor hace que las referencias de modelos, el estado de autenticación, los metadatos del modelo y la selección de `/model` sean visibles para el resto de OpenClaw. A continuación, el arnés reclama ese proveedor en `supports(...)`.

El plugin Codex incluido sigue este patrón:

- referencias de modelos de usuario preferidas: `openai/gpt-5.6-sol`
- referencias de compatibilidad: las referencias heredadas `codex/gpt-*` siguen siendo válidas, pero las configuraciones nuevas no deben utilizarlas como referencias normales de proveedor o modelo
- identificador del arnés: `codex`
- autenticación: disponibilidad sintética del proveedor, porque el arnés Codex gestiona el inicio de sesión y la sesión nativos de Codex
- solicitud al servidor de aplicaciones: OpenClaw envía el identificador simple del modelo a Codex y permite que el arnés se comunique con el protocolo nativo del servidor de aplicaciones

El plugin Codex es aditivo. Cuando la política de runtime no está definida o es `auto`, OpenAI solo puede seleccionar Codex cuando su contrato de ruta gestionado por el proveedor declara compatible `codex`: una ruta oficial exacta de Platform Responses o ChatGPT Responses mediante HTTPS sin ninguna modificación de solicitud definida expresamente. El prefijo `openai/*` por sí solo nunca selecciona Codex. Los endpoints personalizados, los adaptadores de Completions y el comportamiento de solicitudes definido expresamente permanecen en OpenClaw. Se rechazan los endpoints HTTP oficiales sin cifrar. Las referencias `codex/gpt-*` antiguas siguen siendo entradas de compatibilidad. Véase
[Runtime de agente implícito de OpenAI](/es/providers/openai#implicit-agent-runtime).

Para consultar la configuración del operador, ejemplos de prefijos de modelos y configuraciones exclusivas de Codex, véase
[Arnés de Codex](/es/plugins/codex-harness).

El plugin Codex aplica la versión mínima del servidor de aplicaciones documentada en
[Arnés de Codex](/es/plugins/codex-harness). Comprueba el protocolo de enlace de inicialización y bloquea los servidores antiguos o sin versión, de modo que OpenClaw solo se ejecuta con la superficie del protocolo que ha probado.

### Middleware de resultados de herramientas

Los plugins incluidos y los plugins instalados habilitados explícitamente cuyos contratos de manifiesto coincidan pueden adjuntar middleware de resultados de herramientas independiente del runtime mediante
`api.registerAgentToolResultMiddleware(...)` cuando su manifiesto declare los
identificadores de runtime de destino en `contracts.agentToolResultMiddleware`. Esta superficie de confianza sirve para transformaciones asíncronas de resultados de herramientas que deben ejecutarse antes de que OpenClaw o Codex devuelvan la salida de la herramienta al modelo.

Los plugins integrados heredados todavía pueden usar
`api.registerCodexAppServerExtensionFactory(...)` para middleware exclusivo del servidor de aplicaciones de Codex,
pero las nuevas transformaciones de resultados deben usar la API neutral respecto al entorno de ejecución. El
hook `api.registerEmbeddedExtensionFactory(...)`, exclusivo del ejecutor integrado, se ha
eliminado; las transformaciones de resultados de herramientas integradas deben usar middleware neutral respecto al entorno de ejecución.

### Clasificación del resultado terminal

Los arneses nativos que controlan su propia proyección de protocolo pueden usar
`classifyAgentHarnessTerminalOutcome(...)` de
`openclaw/plugin-sdk/agent-harness-runtime` cuando un turno completado no haya producido
texto visible del asistente. El auxiliar devuelve `empty`, `reasoning-only` o
`planning-only` para que la política de reserva de OpenClaw pueda decidir si se reintenta con un
modelo diferente. `planning-only` requiere el campo `planText` explícito
del arnés; OpenClaw no lo infiere de la prosa del asistente. El auxiliar
deja intencionadamente sin clasificar los errores de prompt, los turnos en curso y las
respuestas silenciosas intencionadas, como `NO_REPLY`.

### Efectos secundarios al finalizar el agente

Los arneses nativos deben llamar a `runAgentEndSideEffects(...)` desde
`openclaw/plugin-sdk/agent-harness-runtime` después de finalizar un intento. Este
despacha el hook portátil `agent_end` y la captura de investigación de OpenClaw
sin retrasar las respuestas interactivas. Use `awaitAgentEndSideEffects(...)` para
ejecuciones locales no interactivas en las que el intento no deba resolverse hasta que esos
efectos secundarios finalicen. Ambos auxiliares aceptan la misma carga `{ event, ctx }` que
`runAgentHarnessAgentEndHook(...)`; sus fallos no alteran el resultado del
intento completado.

### Entrada del usuario y superficies de herramientas

Los arneses nativos que expongan una solicitud de entrada del usuario a nivel del entorno de ejecución deben usar los
auxiliares de entrada del usuario de `openclaw/plugin-sdk/agent-harness-runtime` para dar formato
al prompt, entregarlo a través de la ruta de respuesta bloqueante de OpenClaw y normalizar
las respuestas de selección o de formato libre a la forma de respuesta nativa del entorno de ejecución. El
auxiliar mantiene coherente la presentación del canal y la TUI, mientras cada arnés conserva su
propio análisis del protocolo y el ciclo de vida de las solicitudes pendientes.

Los arneses nativos que necesiten un enrutamiento compacto de herramientas similar a PI deben usar
`createAgentHarnessToolSurfaceRuntime(...)` de
`openclaw/plugin-sdk/agent-harness-tool-runtime`. Este controla
la selección del control de búsqueda de herramientas/modo de código, los valores predeterminados reducidos para modelos locales,
el filtrado de esquemas compatible con el entorno de ejecución, la ejecución oculta del catálogo, la
hidratación de directorios y la limpieza del catálogo. Los arneses siguen controlando su conversión de herramientas
específica del SDK y la devolución de llamada de ejecución nativa.

### Modo de arnés nativo de Codex

El arnés integrado `codex` es el modo nativo de Codex para los turnos de agente
integrados de OpenClaw. Active primero el plugin integrado `codex` e incluya `codex` en
`plugins.allow` si la configuración usa una lista de permitidos restrictiva. Las configuraciones nativas del servidor de aplicaciones
deben usar `openai/gpt-*`; los turnos del agente de OpenAI seleccionan el arnés de Codex
solo cuando la ruta efectiva declara compatibilidad con Codex. Las referencias heredadas a modelos de Codex
deben repararse con `openclaw doctor --fix`, y las referencias heredadas a modelos `codex/*`
siguen siendo alias de compatibilidad para el arnés nativo.

Cuando se ejecuta este modo, Codex controla el id. de hilo nativo, el comportamiento de reanudación,
la Compaction y la ejecución del servidor de aplicaciones. OpenClaw sigue controlando el canal de chat,
la réplica visible de la transcripción, la política de herramientas, las aprobaciones, la entrega de contenido multimedia y la selección
de sesiones. Use el proveedor/modelo `agentRuntime.id: "codex"` cuando necesite
demostrar que solo la ruta del servidor de aplicaciones de Codex puede hacerse cargo de la ejecución. Los entornos de ejecución de plugins
explícitos adoptan un comportamiento cerrado ante fallos; los fallos de selección y de ejecución del servidor de aplicaciones
de Codex no se reintentan mediante otro entorno de ejecución.

## Rigurosidad del entorno de ejecución

De forma predeterminada, OpenClaw usa la política de entorno de ejecución de proveedor/modelo `auto`: los
arneses de plugins registrados pueden hacerse cargo de las rutas efectivas compatibles, y el entorno de ejecución
integrado gestiona el turno cuando ninguno coincide. Un prefijo de proveedor/modelo por sí solo nunca
selecciona un arnés. Use un entorno de ejecución de plugin de proveedor/modelo explícito, como
`agentRuntime.id: "codex"`, cuando la ausencia de selección de arnés deba producir un fallo
en lugar de enrutar a través del entorno de ejecución integrado. La selección explícita no hace que una
ruta incompatible pase a ser compatible. Los fallos del arnés de plugin seleccionado siempre
producen un fallo definitivo. Esto no bloquea un
`agentRuntime.id: "openclaw"` de proveedor/modelo explícito.

Para ejecuciones integradas exclusivas de Codex:

```json
{
  "models": {
    "providers": {
      "openai": {
        "agentRuntime": {
          "id": "codex"
        }
      }
    }
  },
  "agents": {
    "defaults": {
      "model": "openai/gpt-5.6-sol"
    }
  }
}
```

Si desea un backend de CLI para un modelo canónico, coloque el entorno de ejecución en la
entrada de ese modelo:

```json
{
  "agents": {
    "defaults": {
      "model": "anthropic/claude-opus-5",
      "models": {
        "anthropic/claude-opus-5": {
          "agentRuntime": {
            "id": "claude-cli"
          }
        }
      }
    }
  }
}
```

Las anulaciones por agente usan la misma forma con ámbito de modelo:

```json
{
  "agents": {
    "list": [
      {
        "id": "codex-only",
        "model": "openai/gpt-5.6-sol",
        "models": {
          "openai/gpt-5.6-sol": {
            "agentRuntime": { "id": "codex" }
          }
        }
      }
    ]
  }
}
```

Los ejemplos heredados de entorno de ejecución para todo el agente, como este, se ignoran:

```json
{
  "agents": {
    "defaults": {
      "agentRuntime": {
        "id": "codex"
      }
    }
  }
}
```

Con un entorno de ejecución de plugin explícito, una sesión falla de forma anticipada cuando el
arnés solicitado no está registrado, no admite el proveedor/modelo resuelto o
falla antes de producir efectos secundarios del turno. Esto es intencionado para implementaciones exclusivas
de Codex y para pruebas en vivo que deban demostrar que la ruta del servidor de aplicaciones de Codex
está realmente en uso.

Esta configuración solo controla el arnés de agente integrado. No desactiva
el enrutamiento de modelos específico del proveedor para imágenes, vídeo, música, TTS, PDF u otros tipos.

## Sesiones nativas y réplica de la transcripción

Un arnés puede conservar un id. de sesión nativo, un id. de hilo o un token de reanudación
del lado del daemon. Mantenga ese vínculo asociado explícitamente con la sesión de OpenClaw y
siga replicando la salida del asistente y de las herramientas visible para el usuario en la
transcripción de OpenClaw.

La transcripción de OpenClaw sigue siendo la capa de compatibilidad para:

- el historial de sesiones visible en el canal
- la búsqueda e indexación de transcripciones
- el cambio de vuelta al arnés integrado de OpenClaw en un turno posterior
- el comportamiento genérico de `/new`, `/reset` y eliminación de sesiones

Si el arnés almacena un vínculo auxiliar, implemente `reset(...)` para que OpenClaw
pueda borrarlo cuando se restablezca la sesión de OpenClaw propietaria.

## Resultados de herramientas y contenido multimedia

El núcleo construye la lista de herramientas de OpenClaw y la pasa al
intento preparado. Cuando un arnés ejecuta una llamada de herramienta dinámica, devuelva el resultado
de la herramienta mediante la forma del resultado del arnés en lugar de enviar el contenido multimedia del canal
directamente.

Esto mantiene las salidas de texto, imágenes, vídeo, música, TTS, aprobaciones y herramientas
de mensajería en la misma ruta de entrega que las ejecuciones respaldadas por OpenClaw.

Establezca `AgentHarnessAttemptResult.hostOwnedToolMediaUrls` solo para artefactos nativos
que el entorno de ejecución de confianza del arnés haya creado y conservado por sí mismo. Cada entrada también debe
aparecer en `toolMediaUrls`. Nunca incluya contenido multimedia seleccionado por el modelo de herramientas dinámicas o
herramientas de OpenClaw. En las rutas `message_tool_only`, esta procedencia restringida permite que
los artefactos nativos del entorno de ejecución sobrevivan a la supresión de la respuesta de origen; la política normal de envío
y la admisión en salas ambientales siguen siendo aplicables.

### Resultados terminales de herramientas

`AgentHarnessAttemptParams.observeToolTerminal` es el acumulador de resultados
terminales controlado por el host. Un arnés que ejecute herramientas dinámicas de OpenClaw o herramientas
nativas debe llamarlo cuando cada herramienta alcance un resultado terminal, antes de
finalizar el resultado del intento. Los arneses que no ejecuten herramientas no necesitan
llamarlo.

Informe de los hechos desde el límite de ejecución:

- Pase el id. de llamada del protocolo cuando exista, el nombre canónico de la herramienta y los
  argumentos que hayan llegado realmente a la herramienta después de la preparación o las reescrituras de hooks.
- Establezca `executionStarted: false` cuando la validación, la aprobación u otra protección
  haya detenido la llamada antes de que comenzara la implementación de la herramienta. Una vez que pueda
  haberse producido el despacho, informe `true` de forma conservadora.
- Informe `outcome: "success"` o `outcome: "failure"`. Incluya los campos estructurados
  de fallo disponibles en el entorno de ejecución en lugar de inferir el fallo a partir del
  texto mostrado.
- Use `nativeMutation` solo para herramientas nativas que no usen una definición de herramienta de OpenClaw.
  Proporcione allí los datos de mutación y repetición controlados por el protocolo; no copie
  el clasificador de mutaciones de OpenClaw en el arnés.

La devolución de llamada devuelve la resolución canónica de esa llamada. Transfiera su
`lastToolError` a `AgentHarnessAttemptResult` y use sus datos de ejecución,
argumentos y efectos secundarios en la proyección del arnés en lugar de derivar
un estado paralelo. El host conserva un fallo de mutación sin resolver aunque haya herramientas
no relacionadas que se ejecuten correctamente y solo lo borra después de que la acción correspondiente se ejecute correctamente.

La devolución de llamada sigue siendo opcional para mantener la compatibilidad del código fuente con arneses experimentales
anteriores. Opcional no significa que un arnés que ejecute herramientas pueda ignorarla:
sin informes terminales, OpenClaw no puede preservar el estado real de los fallos de herramientas con mutaciones
entre llamadas de herramientas posteriores, incluida la finalización silenciosa de Heartbeat.

### Finalización de herramientas completadas

OpenClaw puede necesitar una respuesta visible final después de que un arnés haya completado todas las
llamadas de herramientas, pero su turno nativo haya terminado sin texto del asistente. Un arnés puede optar
por esa recuperación implementando `finalizeSettledTurn({ attempt,
settledAttempt })`.

La devolución de llamada es una capacidad independiente, no otro intento ordinario. Debe:

- usar la transcripción nativa restringida exacta o una transcripción completa de la aplicación
  inmovilizada hasta el límite del resultado de herramienta completado;
- no exponer herramientas, capacidades de concesión de permisos o de entrada del usuario, hooks de ejecución
  nativos, agentes, Skills, memoria, programación, extensiones ni control remoto;
- enviar únicamente el prompt de finalización proporcionado por el host; y
- adoptar un comportamiento cerrado ante fallos si la estrategia seleccionada de transcripción/aislamiento no puede aplicar
  esas restricciones.

OpenClaw invoca la devolución de llamada una vez como suboperación terminal, fuera del
intento ordinario y del bucle de reintentos. Un fallo termina la ejecución con la
advertencia de turno incompleto que tiene en cuenta los efectos secundarios; no puede entrar en las rutas ordinarias
de rotación de autenticación/perfiles, reserva de modelos, recuperación de contexto, continuación de
Compaction ni revisión solicitada por hooks. La finalización también omite la mutación del prompt por plugins,
`before_agent_run`, la entrada/salida del LLM, la revisión terminal y los
hooks `agent_end`. Los diagnósticos del núcleo siguen registrando la operación y su fallo.

La devolución de llamada devuelve `AgentHarnessSettledTurnFinalizationResult`, no un
resultado de intento ordinario. Sus campos públicos se limitan al mensaje completado
del asistente, el uso de la llamada de finalización, los metadatos de propiedad de la transcripción y
la traza de diagnóstico. El estado de herramientas, entrega, contenido multimedia, creación, ciclo de vida,
repetición, sesión y reserva no puede cruzar este límite del resultado. Los campos desconocidos y las llamadas
a herramientas del asistente adoptan un comportamiento cerrado ante fallos.

Un arnés que reutilice internamente su motor de intentos completo puede llamar a
`projectSettledTurnFinalizationAttemptResult(...)` antes de devolver el resultado. El auxiliar
rechaza las evidencias canónicas de fallo, herramientas, entrega, repetición y ciclo de vida, y después
proyecta únicamente el resultado restringido. Es una defensa en profundidad posterior al aislamiento nativo,
no un sustituto de eliminar la superficie de capacidades nativa.

Un arnés basado en proyecciones debe colocar el contexto completo en
`settledAttempt.settledTurnFinalizationContext` con
`source: "openclaw-transcript"`. Debe capturar la rama activa después de que se
replique el turno completado, demostrar que el prompt actual y cada llamada/resultado de herramienta
actual estén presentes hasta ese límite e inmovilizar la matriz de mensajes resultante
antes de devolver el intento. El finalizador debe rechazar un contexto ausente,
no compatible, ambiguo o demasiado grande. No debe truncar mensajes,
descartar el historial anterior ni describir esta transcripción de la aplicación como historial nativo
exacto. Los arneses que reanudan una sesión nativa restringida no necesitan este
campo de proyección.

No implemente esta devolución de llamada invocando `runAttempt` con una indicación
`disableTools` aproximada. El propietario del arnés debe aplicar el límite completo
de capacidades nativas. OpenClaw no proporciona una reserva genérica porque no
puede certificar que un entorno de ejecución nativo arbitrario haya respetado esas restricciones.

El callback sigue siendo opcional para mantener la compatibilidad con harnesses experimentales de terceros. Cuando el harness seleccionado lo omite, OpenClaw conserva el error existente de turno incompleto en lugar de arriesgarse a repetir efectos secundarios.

## Limitaciones actuales

- La ruta pública de importación es genérica, pero algunos alias de tipos de intento/resultado aún conservan nombres heredados por compatibilidad.
- La instalación de harnesses de terceros es experimental. Se recomienda usar plugins de proveedor hasta que se necesite un entorno de ejecución de sesión nativo.
- Se admite el cambio de harness entre turnos. No se debe cambiar de harness en medio de un turno una vez que hayan comenzado las herramientas nativas, las aprobaciones, el texto del asistente o los envíos de mensajes.

## Contenido relacionado

- [Descripción general del SDK](/es/plugins/sdk-overview)
- [Ayudantes del entorno de ejecución](/es/plugins/sdk-runtime)
- [Plugins de proveedor](/es/plugins/sdk-provider-plugins)
- [Harness de Codex](/es/plugins/codex-harness)
- [Proveedores de modelos](/es/concepts/model-providers)
