---
read_when:
    - Creación o depuración de plugins nativos de OpenClaw
    - Comprender el modelo de capacidades de los plugins o los límites de responsabilidad
    - Trabajar en el pipeline de carga o el registro de plugins
    - Implementación de hooks de runtime de proveedores o plugins de canal
sidebarTitle: Internals
summary: 'Aspectos internos de los Plugins: modelo de capacidades, propiedad, contratos, pipeline de carga y auxiliares de tiempo de ejecución'
title: Detalles internos del Plugin
x-i18n:
    generated_at: "2026-07-26T04:45:39Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d47551b1bc2f71ce2ade3dfdd14bff8ee187616c3807f8101c1a3236e1443cc1
    source_path: plugins/architecture.md
    workflow: 16
---

Esta es la **referencia detallada de arquitectura** del sistema de plugins de OpenClaw. Para consultar guías prácticas, se recomienda comenzar con una de las páginas específicas siguientes.

<CardGroup cols={2}>
  <Card title="Instalar y usar plugins" icon="plug" href="/es/tools/plugin">
    Guía para usuarios finales sobre cómo añadir, habilitar y solucionar problemas de los plugins.
  </Card>
  <Card title="Crear plugins" icon="rocket" href="/es/plugins/building-plugins">
    Tutorial para crear el primer plugin con el manifiesto funcional más pequeño.
  </Card>
  <Card title="Plugins de canal" icon="comments" href="/es/plugins/sdk-channel-plugins">
    Crear un plugin de canal de mensajería.
  </Card>
  <Card title="Plugins de proveedor" icon="microchip" href="/es/plugins/sdk-provider-plugins">
    Crear un plugin de proveedor de modelos.
  </Card>
  <Card title="Descripción general del SDK" icon="book" href="/es/plugins/sdk-overview">
    Referencia del mapa de importaciones y la API de registro.
  </Card>
</CardGroup>

## Modelo público de capacidades

Las capacidades constituyen el modelo público de **plugins nativos** en OpenClaw. Cada plugin nativo de OpenClaw se registra con uno o más tipos de capacidad:

| Capacidad                    | Método de registro                              | Plugins de ejemplo                                           |
| ---------------------------- | ----------------------------------------------- | ------------------------------------------------------------ |
| Inferencia de texto          | `api.registerProvider(...)`                              | `anthropic`, `openai`                       |
| Backend de inferencia de CLI | `api.registerCliBackend(...)`                              | `anthropic`, `openai`                       |
| Embeddings                   | `api.registerEmbeddingProvider(...)`                              | Plugins vectoriales propiedad del proveedor                  |
| Voz                          | `api.registerSpeechProvider(...)`                              | `elevenlabs`, `microsoft`                       |
| Transcripción en tiempo real | `api.registerRealtimeTranscriptionProvider(...)`                              | `openai`                                           |
| Voz en tiempo real           | `api.registerRealtimeVoiceProvider(...)`                              | `google`, `openai`                       |
| Comprensión multimedia       | `api.registerMediaUnderstandingProvider(...)`                              | `google`, `openai`                       |
| Fuente de transcripciones    | `api.registerTranscriptSourceProvider(...)`                              | `discord`, `google-meet`, `teams-meetings`, `zoom-meetings` |
| Generación de imágenes       | `api.registerImageGenerationProvider(...)`                              | `fal`, `google`, `openai`  |
| Generación de música         | `api.registerMusicGenerationProvider(...)`                              | `fal`, `google`, `minimax`  |
| Generación de vídeo          | `api.registerVideoGenerationProvider(...)`                              | `fal`, `google`, `qwen`  |
| Obtención web                | `api.registerWebFetchProvider(...)`                              | `firecrawl`                                           |
| Búsqueda web                 | `api.registerWebSearchProvider(...)`                              | `brave`, `firecrawl`, `google`  |
| Canal/mensajería             | `api.registerChannel(...)`                              | `matrix`, `msteams`                       |
| Detección del Gateway        | `api.registerGatewayDiscoveryService(...)`                              | `bonjour`                                           |

<Note>
Un plugin que registra cero capacidades, pero proporciona hooks, herramientas, servicios de detección o servicios en segundo plano, es un plugin **heredado basado únicamente en hooks**. Este patrón sigue siendo totalmente compatible.
</Note>

### Postura sobre la compatibilidad externa

Actualmente, el modelo de capacidades está integrado en el núcleo y lo utilizan los plugins incluidos y nativos, pero la compatibilidad de los plugins externos aún exige un criterio más estricto que «se exporta, por lo tanto está congelado».

| Situación del plugin                              | Orientación                                                                                                                 |
| ------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| Plugins externos existentes                       | Mantener en funcionamiento las integraciones basadas en hooks; esta es la referencia de compatibilidad.                     |
| Nuevos plugins incluidos o nativos                | Preferir el registro explícito de capacidades frente a accesos específicos de proveedores o nuevos diseños basados únicamente en hooks. |
| Plugins externos que adoptan el registro de capacidades | Permitido, pero las superficies auxiliares específicas de cada capacidad deben considerarse en evolución, salvo que la documentación las marque como estables. |

El registro de capacidades es la dirección prevista. Durante la transición, los hooks heredados siguen siendo la vía más segura para evitar incompatibilidades en los plugins externos. No todas las subrutas auxiliares exportadas son equivalentes; se deben preferir los contratos específicos documentados frente a exportaciones auxiliares circunstanciales.

### Formas de los plugins

OpenClaw clasifica cada plugin cargado en una forma según su comportamiento real de registro, no solo según los metadatos estáticos:

<AccordionGroup>
  <Accordion title="plain-capability">
    Registra exactamente un tipo de capacidad (por ejemplo, un plugin exclusivo de proveedor como `arcee` o `chutes`).
  </Accordion>
  <Accordion title="hybrid-capability">
    Registra varios tipos de capacidad (por ejemplo, `openai` posee la inferencia de texto, la voz, la comprensión multimedia y la generación de imágenes).
  </Accordion>
  <Accordion title="hook-only">
    Registra únicamente hooks (tipados o personalizados), sin capacidades, herramientas, comandos ni servicios.
  </Accordion>
  <Accordion title="non-capability">
    Registra herramientas, comandos, servicios o rutas, pero no capacidades.
  </Accordion>
</AccordionGroup>

Se puede usar `openclaw plugins inspect <id>` para consultar la forma y el desglose de capacidades de un plugin. Para obtener más información, véase la [referencia de la CLI](/es/cli/plugins#inspect).

### Señales de compatibilidad

`openclaw doctor`, `openclaw plugins inspect <id>`, `openclaw status --all` y `openclaw plugins doctor` muestran estos avisos de compatibilidad:

| Señal                                            | Significado                                                                                                           |
| ------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------- |
| **configuración válida**                         | La configuración se analiza correctamente y los plugins se resuelven                                                 |
| **solo hooks** (información)                     | El plugin solo registra hooks; es una vía compatible, pero aún no se ha migrado al registro de capacidades           |
| **API de embeddings de memoria obsoleta** (advertencia) | Un plugin no incluido utiliza la antigua API de proveedor de embeddings específica de memoria en lugar de `registerEmbeddingProvider` |
| **error grave**                                  | La configuración no es válida o no se pudo cargar el plugin                                                          |

Actualmente, ninguna de las señales informativas o de advertencia impide que el plugin funcione. Estas señales también aparecen en `openclaw status --all` y `openclaw plugins doctor`.

## Descripción general de la arquitectura

El sistema de plugins de OpenClaw tiene cuatro capas:

<Steps>
  <Step title="Manifiesto y detección">
    OpenClaw encuentra plugins candidatos en las rutas configuradas, las raíces de los espacios de trabajo, las raíces globales de plugins y los plugins incluidos. La detección lee primero los manifiestos nativos `openclaw.plugin.json` y los manifiestos de paquetes compatibles.
  </Step>
  <Step title="Habilitación y validación">
    El núcleo decide si un plugin detectado está habilitado, deshabilitado, bloqueado o seleccionado para un espacio exclusivo, como la memoria.
  </Step>
  <Step title="Carga en tiempo de ejecución">
    Los plugins nativos de OpenClaw se cargan dentro del proceso y registran sus capacidades en un registro central. El JavaScript empaquetado se carga mediante `require`; el código fuente TypeScript local de terceros utiliza Jiti como alternativa de emergencia. Los paquetes compatibles se normalizan como registros del registro sin importar código de tiempo de ejecución.
  </Step>
  <Step title="Consumo de superficies">
    El resto de OpenClaw consulta el registro para exponer herramientas, canales, configuración de proveedores, hooks, rutas HTTP, comandos de la CLI y servicios.
  </Step>
</Steps>

En el caso específico de la CLI de plugins, la detección de comandos raíz se divide en dos fases:

- los metadatos del momento del análisis proceden de `registerCli(..., { descriptors: [...] })`
- el módulo real de la CLI del plugin puede permanecer en carga diferida y registrarse en la primera invocación

Esto mantiene el código de la CLI propiedad del plugin dentro del propio plugin, a la vez que permite que OpenClaw reserve los nombres de los comandos raíz antes del análisis.

El límite de diseño importante es el siguiente:

- la validación del manifiesto y la configuración debe funcionar a partir de los **metadatos del manifiesto y del esquema** sin ejecutar el código del plugin
- la detección de capacidades nativas puede cargar código de entrada de plugins de confianza para crear una instantánea del registro que no active nada
- el comportamiento nativo en tiempo de ejecución procede de la ruta `register(api)` del módulo del plugin con `api.registrationMode === "full"`

Esta separación permite que OpenClaw valide la configuración, explique la ausencia o deshabilitación de plugins y genere indicaciones para la interfaz de usuario y el esquema antes de que se active el entorno de ejecución completo.

### Instantánea de metadatos de plugins y tabla de búsqueda

Al iniciarse, el Gateway crea una instancia de `PluginMetadataSnapshot` para la instantánea de configuración actual. La instantánea solo contiene metadatos: almacena el índice de plugins instalados, el registro de manifiestos, los diagnósticos de manifiestos, los mapas de propietarios, un normalizador de identificadores de plugins y los registros de manifiestos. No contiene módulos de plugins cargados, SDK de proveedores, contenido de paquetes ni exportaciones del entorno de ejecución.

La validación de la configuración compatible con plugins, la habilitación automática al inicio y el arranque de plugins del Gateway utilizan esta instantánea en lugar de reconstruir por separado los metadatos de manifiestos e índices. `PluginLookUpTable` se deriva de la misma instantánea y añade el plan de plugins de inicio correspondiente a la configuración actual del entorno de ejecución.

Tras el inicio, el Gateway conserva la instantánea actual de metadatos como un producto reemplazable del entorno de ejecución. La detección repetida de proveedores en tiempo de ejecución puede reutilizar esa instantánea, en lugar de reconstruir el índice de instalaciones y el registro de manifiestos en cada pasada por el catálogo de proveedores. La instantánea se elimina o se sustituye cuando se apaga el Gateway, cambia la configuración o el inventario de plugins, o se escribe en el índice de instalaciones; cuando no existe una instantánea actual compatible, las llamadas recurren a la ruta en frío de manifiestos e índices. Las comprobaciones de compatibilidad deben incluir las raíces de detección de plugins, como `plugins.load.paths`, y el espacio de trabajo predeterminado del agente, porque los plugins del espacio de trabajo forman parte del alcance de los metadatos.

La instantánea y la tabla de búsqueda mantienen en la ruta rápida las decisiones repetidas del inicio:

- propiedad de los canales
- inicio diferido de los canales
- identificadores de los plugins de inicio
- propiedad de los proveedores y los backends de la CLI
- propiedad del proveedor de configuración, los alias de comandos, el proveedor del catálogo de modelos y los contratos de manifiesto
- validación del esquema de configuración de plugins y del esquema de configuración de canales
- decisiones de habilitación automática al inicio

El límite de seguridad es la sustitución de la instantánea, no su mutación. La instantánea debe reconstruirse cuando cambien la configuración, el inventario de plugins, los registros de instalación o la política persistente del índice. No debe tratarse como un registro global mutable de uso general ni se deben conservar instantáneas históricas sin límite. La carga de plugins en tiempo de ejecución permanece separada de las instantáneas de metadatos para evitar que un estado obsoleto del entorno de ejecución quede oculto tras una caché de metadatos.

La regla de caché se documenta en [Aspectos internos de la arquitectura de plugins](/es/plugins/architecture-internals#plugin-cache-boundary): los metadatos de manifiestos y detección se mantienen actualizados, salvo que una llamada conserve una instantánea, una tabla de búsqueda o un registro de manifiestos explícitos para el flujo actual. Las cachés ocultas de metadatos y los TTL basados en el tiempo de reloj no forman parte de la carga de plugins. Solo pueden conservarse las cachés del cargador del entorno de ejecución, de módulos y de artefactos de dependencias después de que se hayan cargado realmente el código o los artefactos instalados.

Algunos llamadores de rutas frías todavía reconstruyen los registros de manifiestos directamente a partir del índice persistente de plugins instalados, en lugar de recibir un Gateway `PluginLookUpTable`. Esa ruta ahora reconstruye el registro bajo demanda; cuando un llamador ya disponga de ella, es preferible pasar la tabla de consulta actual o un registro de manifiestos explícito a través de los flujos de ejecución.

### Planificación de la activación

La planificación de la activación forma parte del plano de control. Los llamadores pueden consultar qué plugins son pertinentes para un comando, proveedor, canal, ruta, arnés de agente o capacidad concretos antes de cargar registros de ejecución más amplios.

El planificador mantiene la compatibilidad con el comportamiento actual de los manifiestos:

- `activation.*` los campos son indicaciones explícitas para el planificador
- `providers`, `channels`, `commandAliases`, `setup.providers`, `contracts.tools` y los hooks siguen siendo el mecanismo alternativo de propiedad del manifiesto
- la API del planificador que solo devuelve identificadores sigue disponible para los llamadores existentes
- la API del plan informa de etiquetas de motivo para que los diagnósticos puedan distinguir las indicaciones explícitas del mecanismo alternativo de propiedad

<Warning>
No se debe tratar `activation` como un hook de ciclo de vida ni como sustituto de `register(...)`. Son metadatos utilizados para limitar la carga. Es preferible usar los campos de propiedad cuando ya describan la relación; `activation` debe usarse únicamente para indicaciones adicionales del planificador.
</Warning>

### Plugins de canal y la herramienta de mensajes compartida

Los plugins de canal no necesitan registrar una herramienta independiente para enviar, editar o reaccionar en las acciones normales de chat. OpenClaw mantiene una única herramienta compartida `message` en el núcleo, y los plugins de canal se encargan de la detección y ejecución específicas del canal que hay detrás de ella.

El límite actual es el siguiente:

- el núcleo se encarga del host de la herramienta compartida `message`, la conexión con los prompts, el registro de sesiones e hilos y el despacho de la ejecución
- los plugins de canal se encargan de la detección de acciones dentro del ámbito, la detección de capacidades y cualquier fragmento de esquema específico del canal
- los plugins de canal se encargan de la gramática de conversación de sesión específica del proveedor, como la forma en que los identificadores de conversación codifican los identificadores de hilo o se heredan de las conversaciones principales
- los plugins de canal ejecutan la acción final mediante su adaptador de acciones

Para los plugins de canal, la superficie del SDK es `ChannelMessageActionAdapter.describeMessageTool(...)`. Esa llamada de detección unificada permite que un plugin devuelva conjuntamente sus acciones visibles, capacidades y contribuciones al esquema, de modo que esas partes no pierdan coherencia entre sí.

Los nombres de las acciones de mensajes utilizan deliberadamente un vocabulario cerrado y controlado por el núcleo para que todos los transportes puedan representar todas las acciones. Los plugins añaden nombres de acciones mediante un pull request al núcleo; el registro durante la ejecución no se admite de forma intencionada.

Cuando un parámetro específico de un canal de la herramienta de mensajes contiene una fuente multimedia, como una ruta local o una URL multimedia remota, el plugin también debe devolver `mediaSourceParams` desde `describeMessageTool(...)`. El núcleo utiliza esa lista explícita para aplicar la normalización de rutas del entorno aislado y las indicaciones de acceso a contenido multimedia saliente sin codificar de forma rígida los nombres de parámetros que pertenecen al plugin. En este punto, es preferible usar mapas limitados por acción y no una única lista plana para todo el canal, de modo que un parámetro multimedia exclusivo del perfil no se normalice en acciones no relacionadas como `send`.

El núcleo pasa el ámbito de ejecución a ese paso de detección. Los campos importantes incluyen:

- `accountId`
- `currentChannelId`
- `currentThreadTs`
- `currentMessageId`
- `sessionKey`
- `sessionId`
- `agentId`
- entrada de confianza `requesterSenderId`

Esto es importante para los plugins sensibles al contexto. Un canal puede ocultar o mostrar acciones de mensajes en función de la cuenta activa, la sala, el hilo o el mensaje actuales, o la identidad de confianza del solicitante, sin codificar de forma rígida bifurcaciones específicas del canal en la herramienta principal `message`.

Por este motivo, los cambios de enrutamiento del ejecutor integrado siguen siendo trabajo del plugin: el ejecutor es responsable de reenviar la identidad actual del chat y de la sesión al límite de detección del plugin para que la herramienta compartida `message` exponga la superficie correcta, propiedad del canal, durante el turno actual.

Para los auxiliares de ejecución que pertenecen al canal, los plugins de canal deben mantener el entorno de ejecución dentro de sus propios módulos de plugin. El núcleo ya no se encarga de los entornos de ejecución de acciones de mensajes de Discord, Slack, Telegram o WhatsApp en `src/agents/tools`. No se publican subrutas independientes `plugin-sdk/*-action-runtime`, y esos plugins deben importar directamente su propio código de ejecución local desde los módulos que les pertenecen.

El mismo límite se aplica en general a los puntos de integración del SDK que llevan el nombre de un proveedor: el núcleo no debe importar módulos de conveniencia específicos de canales para Discord, Signal, Slack, WhatsApp ni plugins similares. Si el núcleo necesita un comportamiento, debe consumir el módulo `api.ts` / `runtime-api.ts` del propio plugin incluido o convertir la necesidad en una capacidad genérica y limitada del SDK compartido.

Los plugins incluidos siguen la misma regla. El `runtime-api.ts` de un plugin incluido no debe volver a exportar su propia fachada de marca `openclaw/plugin-sdk/<plugin-id>`. Esas fachadas de marca siguen siendo capas de compatibilidad para plugins externos y consumidores antiguos, pero los plugins incluidos deben usar exportaciones locales junto con subrutas genéricas y limitadas del SDK, como `openclaw/plugin-sdk/channel-policy`, `openclaw/plugin-sdk/runtime-store` o `openclaw/plugin-sdk/webhook-ingress`. El código nuevo no debe añadir fachadas del SDK específicas de un identificador de plugin, salvo que lo exija el límite de compatibilidad de un ecosistema externo existente.

En el caso específico de las encuestas, existen dos rutas de ejecución:

- `outbound.sendPoll` es la base compartida para los canales que se ajustan al modelo común de encuestas
- `actions.handleAction("poll")` es la ruta preferida para la semántica de encuestas específica de un canal o para parámetros de encuesta adicionales

El núcleo ahora pospone el análisis compartido de encuestas hasta que el despacho de encuestas del plugin rechaza la acción, de modo que los controladores de encuestas pertenecientes al plugin puedan aceptar campos de encuesta específicos del canal sin que el analizador genérico de encuestas los bloquee primero.

Consulte [Detalles internos de la arquitectura de plugins](/es/plugins/architecture-internals) para conocer la secuencia de inicio completa.

## Modelo de propiedad de capacidades

OpenClaw trata un plugin nativo como el límite de propiedad de una **empresa** o una **funcionalidad**, no como una colección indiscriminada de integraciones no relacionadas.

Esto significa lo siguiente:

- por lo general, un plugin de empresa debe encargarse de todas las superficies de esa empresa orientadas a OpenClaw
- por lo general, un plugin de funcionalidad debe encargarse de toda la superficie de la funcionalidad que introduce
- los canales deben consumir capacidades compartidas del núcleo en lugar de volver a implementar de forma improvisada el comportamiento de los proveedores

<AccordionGroup>
  <Accordion title="Proveedor con múltiples capacidades">
    `google` se encarga de la inferencia de texto, el backend de CLI, las incrustaciones, la voz, la voz en tiempo real, la comprensión multimedia, la generación de imágenes, música y vídeo, y la búsqueda web. `openai` se encarga de la inferencia de texto, las incrustaciones, la voz, la transcripción en tiempo real, la voz en tiempo real, la comprensión multimedia y la generación de imágenes y vídeo. `minimax` se encarga de la inferencia de texto, además de la comprensión multimedia, la voz, la generación de imágenes, música y vídeo, y la búsqueda web.
  </Accordion>
  <Accordion title="Proveedor con una sola capacidad">
    `arcee` y `chutes` se encargan únicamente de la inferencia de texto; `microsoft` se encarga únicamente de la voz. Un plugin de proveedor puede mantener este alcance reducido hasta que necesite abarcar una mayor parte de la superficie de ese proveedor.
  </Accordion>
  <Accordion title="Plugin de funcionalidad">
    `voice-call` se encarga del transporte de llamadas, las herramientas, la CLI, las rutas y el puente de transmisiones multimedia de Twilio, pero consume las capacidades compartidas de voz, transcripción en tiempo real y voz en tiempo real en lugar de importar directamente los plugins de proveedores.
  </Accordion>
</AccordionGroup>

El estado final previsto es el siguiente:

- la superficie de un proveedor orientada a OpenClaw reside en un solo plugin, aunque abarque modelos de texto, voz, imágenes y vídeo
- otros proveedores pueden hacer lo mismo con su propia superficie
- a los canales no les importa qué plugin de proveedor se encarga del proveedor; consumen el contrato de capacidad compartido que expone el núcleo

Esta es la distinción fundamental:

- **plugin** = límite de propiedad
- **capacidad** = contrato del núcleo que varios plugins pueden implementar o consumir

Por tanto, si OpenClaw añade un dominio nuevo, como el vídeo, la primera pregunta no es «¿qué proveedor debe codificar de forma rígida el manejo del vídeo?». La primera pregunta es «¿cuál es el contrato de capacidad de vídeo del núcleo?». Una vez que exista ese contrato, los plugins de proveedores pueden registrarse en él y los plugins de canales o funcionalidades pueden consumirlo.

Si la capacidad todavía no existe, lo adecuado suele ser:

<Steps>
  <Step title="Definir la capacidad">
    Definir en el núcleo la capacidad que falta.
  </Step>
  <Step title="Exponerla mediante el SDK">
    Exponerla de forma tipada mediante la API y el entorno de ejecución del plugin.
  </Step>
  <Step title="Conectar los consumidores">
    Conectar los canales y las funcionalidades con esa capacidad.
  </Step>
  <Step title="Implementaciones de proveedores">
    Permitir que los plugins de proveedores registren implementaciones.
  </Step>
</Steps>

Esto mantiene explícita la propiedad y evita, al mismo tiempo, que el comportamiento del núcleo dependa de un único proveedor o de una ruta de código específica de un plugin creada para un caso aislado.

### Capas de capacidades

Utilice este modelo mental para decidir dónde debe residir el código:

<Tabs>
  <Tab title="Capa de capacidades del núcleo">
    Orquestación compartida, políticas, mecanismos alternativos, reglas de combinación de configuración, semántica de entrega y contratos tipados.
  </Tab>
  <Tab title="Capa de plugins de proveedores">
    API específicas del proveedor, autenticación, catálogos de modelos, síntesis de voz, generación de imágenes, backends de vídeo y puntos de conexión de uso.
  </Tab>
  <Tab title="Capa de plugins de canales y funcionalidades">
    Integración de Discord, Slack, llamadas de voz, etc., que consume capacidades del núcleo y las presenta en una superficie.
  </Tab>
</Tabs>

Por ejemplo, la conversión de texto a voz sigue esta estructura:

- el núcleo se encarga de la política de conversión de texto a voz en el momento de responder, el orden de los mecanismos alternativos, las preferencias y la entrega al canal
- `elevenlabs`, `google`, `microsoft` y `openai` se encargan de las implementaciones de síntesis
- `voice-call` consume el auxiliar de ejecución de conversión de texto a voz para telefonía

Debe preferirse este mismo patrón para capacidades futuras.

### Ejemplo de plugin de empresa con múltiples capacidades

Un plugin de empresa debe percibirse como una unidad coherente desde el exterior. Si OpenClaw dispone de contratos compartidos para modelos, voz, transcripción en tiempo real, voz en tiempo real, comprensión multimedia, generación de imágenes, generación de vídeo, obtención de contenido web y búsqueda web, un proveedor puede encargarse de todas sus superficies en un único lugar:

```ts
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { exampleAiMedia } from "./exampleai-media.js";

export default definePluginEntry({
  id: "exampleai",
  name: "ExampleAI",
  description: "Modelos y capacidades multimedia de ExampleAI.",
  register(api) {
    api.registerProvider({
      id: "exampleai",
      // hooks de autenticación, catálogo de modelos y entorno de ejecución
    });

    api.registerSpeechProvider({
      id: "exampleai",
      // configuración de voz del proveedor — implementar directamente la interfaz SpeechProviderPlugin
    });

    api.registerMediaUnderstandingProvider({
      id: "exampleai",
      capabilities: ["image", "audio", "video"],
      describeImage: (req) => exampleAiMedia.describeImage(req),
      transcribeAudio: (req) => exampleAiMedia.transcribeAudio(req),
      describeVideo: (req) => exampleAiMedia.describeVideo(req),
    });

    api.registerWebSearchProvider({
      id: "exampleai-search",
      createTool() {
        // Devolver la herramienta de búsqueda web que pertenece al proveedor.
      },
    });
  },
});
```

Lo importante no son los nombres exactos de los auxiliares. Lo importante es la estructura:

- un solo plugin se encarga de la superficie del proveedor
- el núcleo sigue encargándose de los contratos de capacidades
- la traducción de solicitudes del proveedor y los auxiliares HTTP permanecen en el plugin del proveedor
- los canales y los plugins de funcionalidades consumen auxiliares `api.runtime.*`, no código del proveedor
- las pruebas de contrato pueden verificar que el plugin haya registrado las capacidades de las que afirma encargarse

### Ejemplo de capacidad: comprensión de vídeo

OpenClaw ya trata la comprensión de imágenes, audio y vídeo como una única capacidad compartida. Allí se aplica el mismo modelo de propiedad:

<Steps>
  <Step title="Core define el contrato">
    Core define el contrato de comprensión de medios.
  </Step>
  <Step title="Los plugins de proveedores se registran">
    Los plugins de proveedores registran `describeImage`, `transcribeAudio` y `describeVideo` según corresponda.
  </Step>
  <Step title="Los consumidores usan el comportamiento compartido">
    Los canales y los plugins de funcionalidades consumen el comportamiento compartido de Core en lugar de conectarse directamente al código del proveedor.
  </Step>
</Steps>

Esto evita incorporar en Core las suposiciones sobre vídeo de un único proveedor. El plugin es responsable de la superficie del proveedor; Core es responsable del contrato de capacidad y del comportamiento de respaldo.

La generación de vídeo ya utiliza esa misma secuencia: Core es responsable del contrato de capacidad tipado y del asistente de tiempo de ejecución, y los plugins de proveedores registran implementaciones de `api.registerVideoGenerationProvider(...)` conforme a él.

¿Se necesita una lista de verificación concreta para el despliegue? Consulte el [Recetario de capacidades](/es/plugins/adding-capabilities).

## Contratos y aplicación

La superficie de la API de plugins está tipada y centralizada de forma intencionada en `OpenClawPluginApi`. Ese contrato define los puntos de registro compatibles y los asistentes de tiempo de ejecución en los que puede apoyarse un plugin.

Por qué es importante:

- los autores de plugins disponen de un único estándar interno estable
- Core puede rechazar la propiedad duplicada, como cuando dos plugins registran el mismo id de proveedor
- el inicio puede mostrar diagnósticos procesables para registros con formato incorrecto
- las pruebas de contrato pueden hacer cumplir la propiedad de los plugins incluidos y evitar divergencias silenciosas

Hay dos niveles de aplicación:

<AccordionGroup>
  <Accordion title="Aplicación del registro en tiempo de ejecución">
    El registro de plugins valida los registros a medida que se cargan los plugins. Por ejemplo, los ids de proveedor duplicados, los ids de proveedor de voz duplicados y los registros con formato incorrecto generan diagnósticos de plugins en lugar de un comportamiento indefinido.
  </Accordion>
  <Accordion title="Pruebas de contrato">
    Los plugins incluidos se capturan en registros de contratos durante la ejecución de las pruebas para que OpenClaw pueda verificar explícitamente la propiedad. Actualmente, esto se utiliza para proveedores de modelos, proveedores de voz, proveedores de búsqueda web y la propiedad de los registros incluidos.
  </Accordion>
</AccordionGroup>

El efecto práctico es que OpenClaw sabe de antemano qué plugin es responsable de cada superficie. Esto permite que Core y los canales se compongan sin problemas, porque la propiedad se declara, se tipa y se puede probar, en lugar de ser implícita.

### Qué debe incluirse en un contrato

<Tabs>
  <Tab title="Contratos adecuados">
    - tipados
    - pequeños
    - específicos de una capacidad
    - propiedad de Core
    - reutilizables por varios plugins
    - utilizables por canales y funcionalidades sin conocimiento del proveedor

  </Tab>
  <Tab title="Contratos inadecuados">
    - política específica del proveedor oculta en Core
    - vías de escape puntuales para plugins que eluden el registro
    - código de canal que accede directamente a una implementación de proveedor
    - objetos de tiempo de ejecución ad hoc que no forman parte de `OpenClawPluginApi` ni de `api.runtime`

  </Tab>
</Tabs>

En caso de duda, eleve el nivel de abstracción: defina primero la capacidad y, después, permita que los plugins se conecten a ella.

## Modelo de ejecución

Los plugins nativos de OpenClaw se ejecutan **dentro del proceso** junto con el Gateway. No están aislados. Un plugin nativo cargado tiene el mismo límite de confianza a nivel de proceso que el código de Core.

<Warning>
Implicaciones de los plugins nativos: un plugin puede registrar herramientas, controladores de red, hooks y servicios; un error en un plugin puede bloquear o desestabilizar el Gateway; y un plugin nativo malicioso equivale a la ejecución de código arbitrario dentro del proceso de OpenClaw.
</Warning>

Los paquetes compatibles son más seguros de forma predeterminada porque OpenClaw los trata actualmente como paquetes de metadatos o contenido. En las versiones actuales, esto se refiere principalmente a Skills incluidas.

Utilice listas de permitidos y rutas explícitas de instalación y carga para los plugins no incluidos. Trate los plugins del espacio de trabajo como código para la fase de desarrollo, no como valores predeterminados de producción.

Para los nombres de paquetes incluidos en el espacio de trabajo, mantenga el id del plugin anclado al nombre de npm: `@openclaw/<id>` de forma predeterminada, o un sufijo tipado aprobado como `-provider`, `-plugin`, `-speech`, `-sandbox` o `-media-understanding` cuando el paquete exponga intencionadamente una función de plugin más limitada.

<Note>
**Nota de confianza:** `plugins.allow` confía en los **ids de plugins**, no en la procedencia del código fuente. Un plugin del espacio de trabajo con el mismo id que un plugin incluido reemplaza intencionadamente la copia incluida cuando dicho plugin del espacio de trabajo está habilitado o figura en la lista de permitidos. Esto es normal y útil para el desarrollo local, las pruebas de parches y las correcciones urgentes. La confianza de los plugins incluidos se determina a partir de la instantánea del código fuente —el manifiesto y el código presentes en el disco en el momento de la carga—, no de los metadatos de instalación. Un registro de instalación dañado o sustituido no puede ampliar silenciosamente la superficie de confianza de un plugin incluido más allá de lo que declara el código fuente real.
</Note>

## Límite de exportación

OpenClaw exporta capacidades, no facilidades de implementación.

Mantenga público el registro de capacidades. Reduzca las exportaciones de asistentes que no formen parte del contrato:

- subrutas de asistentes específicas de plugins incluidos
- subrutas de infraestructura de tiempo de ejecución no destinadas a ser una API pública
- asistentes de conveniencia específicos de proveedores
- asistentes de configuración e incorporación que son detalles de implementación

Las subrutas reservadas de asistentes para plugins incluidos se han retirado del mapa de exportaciones generado del SDK. Mantenga los asistentes específicos de cada responsable dentro del paquete del plugin correspondiente; promueva únicamente el comportamiento reutilizable del host a contratos genéricos del SDK, como `plugin-sdk/gateway-runtime`, `plugin-sdk/security-runtime` y las capacidades inyectadas de la API de plugins.

## Detalles internos y referencia

Para consultar el pipeline de carga, el modelo de registro, los hooks de tiempo de ejecución de proveedores, las rutas HTTP del Gateway, los esquemas de herramientas de mensajes, la resolución de destinos de canales, los catálogos de proveedores, los plugins del motor de contexto y la guía para añadir una capacidad nueva, consulte los [Detalles internos de la arquitectura de plugins](/es/plugins/architecture-internals).

## Temas relacionados

- [Creación de plugins](/es/plugins/building-plugins)
- [Manifiesto de plugins](/es/plugins/manifest)
- [Configuración del SDK de plugins](/es/plugins/sdk-setup)
