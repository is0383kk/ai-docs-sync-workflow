---
read_when:
    - Implementación de hooks de runtime de proveedores, ciclo de vida de canales o paquetes de paquetes
    - Depuración del orden de carga de plugins o del estado del registro
    - Añadir una nueva capacidad de Plugin o un Plugin de motor de contexto
summary: 'Aspectos internos de la arquitectura de Plugin: Pipeline de carga, registro, hooks de runtime, rutas HTTP y tablas de referencia'
title: Detalles internos de la arquitectura de plugins
x-i18n:
    generated_at: "2026-07-26T05:47:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 278ac23a9454ab69407c59fa197e75756fa0dc5880fcae6c3eecc15bd4733a09
    source_path: plugins/architecture-internals.md
    workflow: 16
---

Para conocer el modelo público de capacidades, las estructuras de los plugins y los contratos de propiedad/ejecución, consulte [Arquitectura de plugins](/es/plugins/architecture). Esta página aborda los mecanismos internos: Pipeline de carga, registro, hooks de tiempo de ejecución, rutas HTTP del Gateway, rutas de importación y tablas de esquemas.

## Pipeline de carga

Al iniciarse, OpenClaw hace aproximadamente lo siguiente:

1. descubre las raíces de plugins candidatas
2. lee los manifiestos de paquetes nativos o compatibles y los metadatos de paquetes
3. rechaza los candidatos no seguros
4. normaliza la configuración de los plugins (`plugins.enabled`, `allow`, `deny`, `entries`,
   `slots`, `load.paths`)
5. decide la habilitación de cada candidato
6. carga los módulos nativos habilitados: los módulos incluidos y compilados usan un cargador nativo;
   el código fuente TypeScript local de terceros usa el mecanismo de reserva de emergencia Jiti
7. llama a los hooks nativos `register(api)` y recopila los registros en el registro de plugins
8. expone el registro a los comandos y las superficies de tiempo de ejecución

Las comprobaciones de seguridad se ejecutan **antes** de la ejecución en tiempo de ejecución. El descubrimiento bloquea un candidato cuando:

- su punto de entrada resuelto sale de la raíz del plugin
- su ruta (o su directorio raíz) permite la escritura a cualquier usuario
- para los plugins no incluidos, la propiedad de la ruta no coincide con el uid actual (o root)

En los directorios incluidos que permiten la escritura a cualquier usuario, primero se intenta realizar una reparación local mediante `chmod` (las instalaciones npm/globales pueden distribuir directorios de paquetes con `0777`) antes de volver a ejecutar la comprobación; las comprobaciones de propiedad se omiten por completo para el origen incluido.

Los candidatos bloqueados siguen incluyendo el id de su plugin en el diagnóstico emitido cuando se conoce (incluidos los ids resueltos a partir de un manifiesto dentro de un directorio rechazado por otros motivos), de modo que la configuración que hace referencia a ese id muestra un plugin bloqueado vinculado a una advertencia de seguridad de la ruta, en lugar de un error no relacionado de «plugin desconocido».

### Comportamiento basado primero en el manifiesto

El manifiesto es la fuente de verdad del plano de control. OpenClaw lo utiliza para:

- identificar el plugin
- descubrir los canales, Skills, esquemas de configuración o capacidades del paquete declarados
- validar `plugins.entries.<id>.config`
- ampliar las etiquetas y los textos de marcador de posición de la interfaz de control
- mostrar los metadatos de instalación y catálogo
- conservar descriptores ligeros de activación y configuración sin cargar el tiempo de ejecución del plugin

Para los plugins nativos, el módulo de tiempo de ejecución es la parte del plano de datos. Registra el comportamiento real, como hooks, herramientas, comandos o flujos de proveedores.

Los bloques opcionales `activation` y `setup` del manifiesto permanecen en el plano de control. Son descriptores exclusivamente de metadatos para planificar la activación y descubrir la configuración; no sustituyen el registro en tiempo de ejecución, `register(...)` ni `setupEntry`. Los consumidores de activación en vivo utilizan las indicaciones de comandos, canales y proveedores del manifiesto para limitar la carga de plugins antes de una materialización más amplia del registro:

- la carga de la CLI se limita a los plugins que poseen el comando principal solicitado
- la configuración del canal o resolución del plugin se limita a los plugins que poseen el id de canal solicitado
- la configuración explícita o resolución en tiempo de ejecución del proveedor se limita a los plugins que poseen el id de proveedor solicitado
- la planificación del inicio del Gateway utiliza `activation.onStartup` para las importaciones explícitas de inicio; los plugins sin metadatos de inicio solo se cargan mediante activadores de activación más específicos

El planificador de activación expone tanto una API que solo contiene ids para los consumidores existentes como una API de planificación para los diagnósticos. Las entradas del plan indican por qué se seleccionó un plugin y distinguen las indicaciones explícitas de `activation.*` de la alternativa basada en la propiedad del manifiesto:

| Motivo (de las indicaciones de `activation.*`)   | Motivo (de la propiedad del manifiesto)                                                             |
| ------------------------------------ | -------------------------------------------------------------------------------------------- |
| `activation-agent-harness-hint`      | —                                                                                            |
| `activation-capability-hint`         | —                                                                                            |
| `activation-channel-hint`            | `manifest-channel-owner` (`channels`)                                                        |
| `activation-command-hint`            | `manifest-command-alias` (`commandAliases`)                                                  |
| `activation-provider-hint`           | `manifest-provider-owner` (`providers`), `manifest-setup-provider-owner` (`setup.providers`) |
| `activation-route-hint`              | —                                                                                            |
| — (el activador de hook no tiene una variante de indicación) | `manifest-hook-owner` (`hooks`), `manifest-tool-contract` (`contracts.tools`)                |

Esta separación de motivos es el límite de compatibilidad: los metadatos de plugins existentes siguen funcionando, mientras que el código nuevo puede detectar indicaciones amplias o comportamientos alternativos sin cambiar la semántica de carga en tiempo de ejecución.

Las precargas en tiempo de ejecución durante las solicitudes que piden el ámbito amplio `all` siguen derivando un conjunto explícito de ids de plugins efectivos a partir de la configuración, la planificación del inicio, los canales configurados, los slots y las reglas de habilitación automática (`resolveEffectivePluginIds` en `src/plugins/effective-plugin-ids.ts`). Si ese conjunto derivado está vacío, OpenClaw mantiene el ámbito vacío en lugar de ampliarlo a todos los plugins detectables.

El descubrimiento de configuración prefiere ids pertenecientes a descriptores, como `setup.providers` y `setup.cliBackends`, para limitar los plugins candidatos antes de recurrir a `setup-api` para los plugins que aún necesitan hooks de tiempo de ejecución durante la configuración. Las listas de configuración de proveedores utilizan `providerAuthChoices` del manifiesto, las opciones de configuración derivadas de descriptores y los metadatos del catálogo de instalación sin cargar el tiempo de ejecución del proveedor. Un `setup.requiresRuntime: false` explícito es un punto de corte exclusivo del descriptor; si se omite `requiresRuntime`, se conserva la alternativa de la API de configuración heredada por compatibilidad. Si más de un plugin descubierto reclama el mismo proveedor de configuración normalizado o id de backend de la CLI, la búsqueda de configuración rechaza el propietario ambiguo en lugar de depender del orden de descubrimiento. Cuando se ejecuta el tiempo de ejecución de configuración, los diagnósticos del registro notifican las discrepancias entre `setup.providers` / `setup.cliBackends` y los proveedores o backends de la CLI registrados realmente por la API de configuración, sin bloquear los plugins heredados.

### Límite de caché de plugins

OpenClaw no almacena en caché los resultados del descubrimiento de plugins ni los datos directos del registro de manifiestos mediante intervalos de reloj. Las instalaciones, las modificaciones de manifiestos y los cambios en las rutas de carga deben hacerse visibles en la siguiente lectura explícita de metadatos o reconstrucción de una instantánea. El analizador de archivos de manifiesto mantiene una caché limitada de firmas de archivo cuya clave combina la ruta del manifiesto abierto con el dispositivo/inodo, el tamaño y mtime/ctime; esa caché solo evita volver a analizar bytes sin cambios y no debe almacenar en caché respuestas de descubrimiento, registro, propiedad ni política.

La ruta rápida y segura para los metadatos es la propiedad explícita de los objetos, no una caché oculta. Las rutas críticas del inicio del Gateway deben pasar el `PluginMetadataSnapshot` actual, el `PluginLookUpTable` derivado o un registro explícito de manifiestos a través de la cadena de llamadas. La validación de la configuración, la habilitación automática durante el inicio, el arranque de plugins y la selección de proveedores pueden reutilizar esos objetos mientras representen la configuración y el inventario de plugins actuales. La búsqueda de configuración sigue reconstruyendo los metadatos del manifiesto bajo demanda, salvo que la ruta de configuración específica reciba un registro explícito de manifiestos; esto debe mantenerse como alternativa de ruta no crítica en lugar de añadir cachés de búsqueda ocultas. Cuando cambie la entrada, se debe reconstruir y sustituir la instantánea en lugar de modificarla o conservar copias históricas. Las vistas del registro de plugins activo y los auxiliares de arranque de los canales incluidos deben volver a calcularse a partir del registro o la raíz actuales. Los mapas de corta duración son aceptables dentro de una única llamada para desduplicar trabajo o proteger contra la reentrada; no deben convertirse en cachés de metadatos del proceso.

Para la carga de plugins, la capa de caché persistente corresponde a la carga en tiempo de ejecución. Puede reutilizar el estado del cargador cuando se cargan realmente el código o los artefactos instalados, como:

- `PluginLoaderCacheState` y registros activos de tiempo de ejecución compatibles
- cachés de jiti/módulos y cachés de cargadores de superficies públicas utilizadas para evitar importar repetidamente la misma superficie de tiempo de ejecución
- cachés del sistema de archivos para los artefactos de plugins instalados
- mapas de corta duración por llamada para normalizar rutas o resolver duplicados

Esas cachés son detalles de implementación del plano de datos. No deben responder preguntas del plano de control como «¿qué plugin posee este proveedor?», salvo que el consumidor haya solicitado deliberadamente la carga en tiempo de ejecución.

No se deben añadir cachés persistentes ni basadas en intervalos de reloj para:

- resultados del descubrimiento
- registros directos de manifiestos
- registros de manifiestos reconstruidos a partir del índice de plugins instalados
- la búsqueda del propietario del proveedor, la supresión de modelos, la política de proveedores o los metadatos de artefactos públicos
- cualquier otra respuesta derivada de manifiestos en la que un cambio en el manifiesto, el índice instalado o la ruta de carga deba ser visible en la siguiente lectura de metadatos

Los consumidores que reconstruyen metadatos de manifiestos a partir del índice persistente de plugins instalados reconstruyen ese registro bajo demanda. El índice instalado es un estado duradero del plano de origen; no es una caché de metadatos oculta dentro del proceso.

## Modelo de registro

Los plugins cargados no modifican directamente variables globales arbitrarias del núcleo. Se registran en un registro central de plugins (`PluginRegistry` en `src/plugins/registry-types.ts`), que realiza el seguimiento de los registros de plugins (identidad, fuente, origen, estado y diagnósticos), además de matrices para cada capacidad: herramientas, hooks heredados y hooks tipados, canales, proveedores, controladores RPC del Gateway, rutas HTTP, registradores de la CLI, servicios en segundo plano, comandos pertenecientes a plugins y muchas otras familias tipadas de proveedores (voz, embeddings, generación de imágenes/vídeos/música, obtención/búsqueda web, arneses de agentes, acciones de sesión, etc.).

A continuación, las funciones del núcleo leen ese registro en lugar de comunicarse directamente con los módulos de plugins. Esto mantiene la carga unidireccional:

- módulo del plugin -> registro en el registro
- tiempo de ejecución del núcleo -> consumo del registro

Esta separación es importante para la mantenibilidad. Significa que la mayoría de las superficies del núcleo solo necesitan un punto de integración: «leer el registro», no «crear un caso especial para cada módulo de plugin».

## Callbacks de vinculación de conversaciones

Los plugins que vinculan una conversación pueden reaccionar cuando se resuelve una aprobación.

Utilice `api.onConversationBindingResolved(...)` para recibir un callback después de que una solicitud de vinculación se apruebe o deniegue:

```ts
export default {
  id: "my-plugin",
  register(api) {
    api.onConversationBindingResolved(async (event) => {
      if (event.status === "approved") {
        // Ahora existe una vinculación para este plugin y esta conversación.
        console.log(event.binding?.conversationId);
        return;
      }

      // La solicitud se denegó; borra cualquier estado local pendiente.
      console.log(event.request.conversation.conversationId);
    });
  },
};
```

Campos de la carga útil del callback:

- `status`: `"approved"` o `"denied"`
- `decision`: `"allow-once"`, `"allow-always"` o `"deny"`
- `binding`: la vinculación resuelta para las solicitudes aprobadas
- `request`: el resumen de la solicitud original, la indicación de desvinculación, el id del remitente y los metadatos de la conversación

Este callback es exclusivamente de notificación. No cambia quién tiene permiso para vincular una conversación y se ejecuta después de que finalice el procesamiento de la aprobación por parte del núcleo.

## Hooks de tiempo de ejecución del proveedor

Los plugins de proveedores tienen tres capas:

- **Metadatos del manifiesto** para búsquedas ligeras previas al tiempo de ejecución:
  `setup.providers[].envVars`, `providerAuthAliases`, `providerAuthChoices`
  y `channelConfigs`.
- **Hooks durante la configuración**: `catalog` más `applyConfigDefaults`.
- **Hooks de tiempo de ejecución**: más de 40 hooks opcionales que abarcan la autenticación, la resolución de modelos, el encapsulado de flujos, los niveles de razonamiento, la política de reproducción y los endpoints de uso. Consulte [Orden y uso de los hooks](#hook-order-and-usage).

OpenClaw sigue siendo responsable del bucle genérico del agente, la conmutación por error, la gestión de transcripciones y la
política de herramientas. Estos hooks constituyen la superficie de extensión para el comportamiento
específico del proveedor sin necesidad de un transporte de inferencia totalmente personalizado.

Use `setup.providers[].envVars` del manifiesto cuando el proveedor tenga
credenciales basadas en variables de entorno que las rutas genéricas de autenticación, estado y selector de modelos deban detectar sin
cargar el entorno de ejecución del plugin. Use
`providerAuthAliases` del manifiesto cuando un identificador de proveedor deba reutilizar las variables de entorno, los perfiles de autenticación,
la autenticación respaldada por configuración y la opción de incorporación mediante clave de API de otro identificador de proveedor. Use
`providerAuthChoices` del manifiesto cuando las superficies de CLI de incorporación y elección de autenticación deban conocer el
identificador de opción del proveedor, las etiquetas de grupo y la configuración sencilla de autenticación mediante una sola opción,
sin cargar el entorno de ejecución del proveedor. Reserve
`envVars` del entorno de ejecución del proveedor para indicaciones dirigidas a operadores, como las etiquetas de incorporación o las variables
de configuración del identificador y el secreto de cliente de OAuth.

Describa la configuración y autenticación del canal basadas en variables de entorno mediante los descriptores de
`channelConfigs.<id>.schema` y configuración correspondientes.

### Orden y uso de los hooks

Para los plugins de modelos y proveedores, OpenClaw llama a los hooks aproximadamente en este orden.
La columna «Cuándo usarlo» es la guía rápida para tomar decisiones.
Los campos de proveedor exclusivos para compatibilidad que OpenClaw ya no invoca, como
`ProviderPlugin.capabilities` y `suppressBuiltInModel`, no se
incluyen aquí de forma intencionada.

| Hook                              | Qué hace                                                                                                   | Cuándo usarlo                                                                                                                                   |
| --------------------------------- | -------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `catalog`                         | Publica la configuración del proveedor en `models.providers` durante la generación de `models.json`                                | El proveedor posee un catálogo o valores predeterminados de URL base                                                                                                  |
| `applyConfigDefaults`             | Aplica los valores predeterminados de configuración global propiedad del proveedor durante la materialización de la configuración                                      | Los valores predeterminados dependen del modo de autenticación, el entorno o la semántica de la familia de modelos del proveedor                                                                         |
| _(búsqueda de modelos integrada)_         | OpenClaw prueba primero la ruta normal del registro o catálogo                                                          | _(no es un hook de plugin)_                                                                                                                         |
| `normalizeModelId`                | Normaliza los alias heredados o preliminares de identificadores de modelos antes de la búsqueda                                                     | El proveedor se encarga de depurar los alias antes de la resolución canónica del modelo                                                                                 |
| `normalizeTransport`              | Normaliza los `api` / `baseUrl` de la familia del proveedor antes del ensamblaje genérico del modelo                                      | El proveedor se encarga de depurar el transporte para identificadores de proveedor personalizados de la misma familia de transporte                                                          |
| `normalizeConfig`                 | Normaliza `models.providers.<id>` antes de la resolución del entorno de ejecución o proveedor                                           | El proveedor necesita una depuración de la configuración que debe residir en el plugin; los auxiliares integrados de la familia de Google también respaldan las entradas de configuración de Google compatibles   |
| `applyNativeStreamingUsageCompat` | Aplica reescrituras de compatibilidad del uso de streaming nativo a los proveedores de configuración                                               | El proveedor necesita correcciones de los metadatos de uso de streaming nativo determinadas por el endpoint                                                                          |
| `resolveConfigApiKey`             | Resuelve la autenticación mediante marcadores de entorno para los proveedores de configuración antes de cargar la autenticación del entorno de ejecución                                       | Los proveedores exponen sus propios hooks para resolver claves de API mediante marcadores de entorno                                                                                |
| `resolveSyntheticAuth`            | Expone la autenticación local, autoalojada o respaldada por la configuración sin conservar texto sin cifrar                                   | El proveedor puede funcionar con un marcador de credencial sintético o local                                                                                 |
| `resolveExternalAuthProfiles`     | Superpone perfiles de autenticación externos propiedad del proveedor; el valor predeterminado de `persistence` es `runtime-only` para credenciales propiedad de la CLI o la aplicación | El proveedor reutiliza credenciales de autenticación externas sin conservar los tokens de actualización copiados; declara `contracts.externalAuthProviders` en el manifiesto |
| `shouldDeferSyntheticProfileAuth` | Reduce la prioridad de los marcadores sintéticos de perfiles almacenados frente a la autenticación respaldada por el entorno o la configuración                                      | El proveedor almacena perfiles de marcador sintéticos que no deben prevalecer                                                                 |
| `resolveDynamicModel`             | Respaldo síncrono para identificadores de modelos propiedad del proveedor que aún no están en el registro local                                       | El proveedor acepta identificadores arbitrarios de modelos del servicio ascendente                                                                                                 |
| `prepareDynamicModel`             | Precalentamiento asíncrono; después, `resolveDynamicModel` vuelve a ejecutarse                                                           | El proveedor necesita metadatos de red antes de resolver identificadores desconocidos                                                                                  |
| `normalizeResolvedModel`          | Reescritura final antes de que el ejecutor integrado use el modelo resuelto                                               | El proveedor necesita reescrituras de transporte, pero sigue usando un transporte del núcleo                                                                             |
| `normalizeToolSchemas`            | Normaliza los esquemas de herramientas antes de que el ejecutor integrado los procese                                                    | El proveedor necesita depurar los esquemas de la familia de transporte                                                                                                |
| `inspectToolSchemas`              | Expone diagnósticos de esquemas propiedad del proveedor después de la normalización                                                  | El proveedor quiere advertencias sobre palabras clave sin incorporar reglas específicas del proveedor al núcleo                                                                 |
| `resolveReasoningOutputMode`      | Selecciona el contrato de salida de razonamiento nativo o etiquetado                                                              | El proveedor necesita una salida etiquetada de razonamiento y resultado final en lugar de campos nativos                                                                         |
| `prepareExtraParams`              | Normaliza los parámetros de solicitud antes de los envoltorios genéricos de opciones de streaming                                              | El proveedor necesita parámetros de solicitud predeterminados o la depuración de parámetros específicos del proveedor                                                                           |
| `createStreamFn`                  | Sustituye por completo la ruta normal de streaming por un transporte personalizado                                                   | El proveedor necesita un protocolo de comunicación personalizado, no solo un envoltorio                                                                                     |
| `wrapStreamFn`                    | Envoltorio de streaming después de aplicar los envoltorios genéricos                                                              | El proveedor necesita envoltorios de compatibilidad para encabezados, cuerpo o modelo de la solicitud sin un transporte personalizado                                                          |
| `resolveTransportTurnState`       | Adjunta encabezados o metadatos de transporte nativos por turno                                                           | El proveedor quiere que los transportes genéricos envíen la identidad de turno nativa del proveedor                                                                       |
| `resolveWebSocketSessionPolicy`   | Adjunta encabezados nativos de WebSocket o una política de tiempo de espera de la sesión                                                    | El proveedor quiere que los transportes WS genéricos ajusten los encabezados de sesión o la política de respaldo                                                               |
| `formatApiKey`                    | Formateador de perfiles de autenticación: el perfil almacenado se convierte en la cadena `apiKey` del entorno de ejecución                                     | El proveedor almacena metadatos de autenticación adicionales y necesita un formato de token personalizado para el entorno de ejecución                                                                    |
| `refreshOAuth`                    | Sustitución de la actualización de OAuth para endpoints de actualización personalizados o políticas ante errores de actualización                                  | El proveedor no es compatible con los actualizadores compartidos de OpenClaw                                                                                          |
| `buildAuthDoctorHint`             | Sugerencia de reparación añadida cuando falla la actualización de OAuth                                                                  | El proveedor necesita instrucciones de reparación de la autenticación bajo su responsabilidad tras un error de actualización                                                                      |
| `matchesContextOverflowError`     | Detector de desbordamiento de la ventana de contexto propiedad del proveedor                                                                 | El proveedor presenta errores de desbordamiento sin procesar que las heurísticas genéricas no detectarían                                                                                |
| `classifyFailoverReason`          | Clasificación de motivos de conmutación por error propiedad del proveedor                                                                  | El proveedor puede asignar errores sin procesar de API o transporte a límites de frecuencia, sobrecarga, etc.                                                                          |
| `isCacheTtlEligible`              | Política de caché de prompts para proveedores de proxy o red de retorno                                                               | El proveedor necesita controlar el TTL de la caché específicamente para el proxy                                                                                                |
| `buildMissingAuthMessage`         | Sustitución del mensaje genérico de recuperación por ausencia de autenticación                                                      | El proveedor necesita una sugerencia específica para recuperarse de la ausencia de autenticación                                                                                 |
| `augmentModelCatalog`             | Filas sintéticas o finales del catálogo añadidas después del descubrimiento (obsoleto; véase más abajo)                                  | El proveedor necesita filas sintéticas de compatibilidad futura en `models list` y selectores                                                                     |
| `resolveThinkingProfile`          | Conjunto de niveles, etiquetas visibles y valor predeterminado de `/think` específicos del modelo                                                 | El proveedor expone una escala personalizada de razonamiento o una etiqueta binaria para los modelos seleccionados                                                                 |
| `isBinaryThinking`                | Hook de compatibilidad para activar o desactivar el razonamiento                                                                     | El proveedor solo expone el razonamiento binario activado o desactivado                                                                                                  |
| `supportsXHighThinking`           | Hook de compatibilidad con el razonamiento `xhigh`                                                                   | El proveedor quiere `xhigh` solo en un subconjunto de modelos                                                                                             |
| `resolveDefaultThinkingLevel`     | Hook de compatibilidad con el nivel predeterminado de `/think`                                                                      | El proveedor controla la política predeterminada de `/think` para una familia de modelos                                                                                      |
| `isModernModelRef`                | Detector de modelos modernos para filtros de perfiles en vivo y selección de pruebas de humo                                              | El proveedor controla la coincidencia de modelos preferidos para pruebas en vivo o de humo                                                                                             |
| `prepareRuntimeAuth`              | Intercambia una credencial configurada por el token o la clave reales del entorno de ejecución justo antes de la inferencia                       | El proveedor necesita un intercambio de tokens o una credencial de solicitud de corta duración                                                                             |
| `resolveUsageAuth`                | Resuelve las credenciales de uso o facturación para `/usage` y superficies de estado relacionadas                                     | El proveedor necesita un análisis personalizado del token de uso o cuota, o una credencial de uso diferente                                                               |
| `fetchUsageSnapshot`              | Obtiene y normaliza instantáneas de uso o cuota específicas del proveedor después de resolver la autenticación                             | El proveedor necesita un endpoint de uso específico o un analizador de la carga útil                                                                           |
| `createEmbeddingProvider`         | Crear un adaptador de embeddings para memoria/búsqueda gestionado por el proveedor                                                     | El comportamiento de los embeddings de memoria corresponde al plugin del proveedor                                                                                    |
| `buildReplayPolicy`               | Devolver una política de reproducción que controle el tratamiento de la transcripción para el proveedor                                        | El proveedor necesita una política de transcripción personalizada (por ejemplo, la eliminación de bloques de razonamiento)                                                               |
| `sanitizeReplayHistory`           | Reescribir el historial de reproducción tras la limpieza genérica de la transcripción                                                        | El proveedor necesita reescrituras de reproducción específicas que vayan más allá de los asistentes de Compaction compartidos                                                             |
| `validateReplayTurns`             | Realizar la validación o reestructuración final del turno de reproducción antes del ejecutor integrado                                           | El transporte del proveedor necesita una validación de turnos más estricta tras el saneamiento genérico                                                                    |
| `onModelSelected`                 | Ejecutar efectos secundarios posteriores a la selección gestionados por el proveedor                                                                 | El proveedor necesita telemetría o un estado gestionado por él cuando un modelo pasa a estar activo                                                                  |

`normalizeModelId`, `normalizeTransport` y `normalizeConfig` comprueban primero el
plugin del proveedor coincidente y luego continúan con otros plugins de proveedores
compatibles con hooks hasta que uno cambie realmente el identificador del modelo o
el transporte/la configuración. Esto permite que los adaptadores de proveedor de
alias/compatibilidad sigan funcionando sin exigir que el llamador sepa qué plugin
incluido es responsable de la reescritura. Si ningún hook de proveedor reescribe una
entrada de configuración compatible de la familia Google, el normalizador de
configuración de Google incluido sigue aplicando esa limpieza de compatibilidad.

Si el proveedor necesita un protocolo de comunicación totalmente personalizado o
un ejecutor de solicitudes personalizado, se trata de una clase de extensión
diferente. Estos hooks son para comportamientos de proveedores que siguen
ejecutándose en el bucle de inferencia normal de OpenClaw.

`resolveUsageAuth` decide si OpenClaw debe llamar a `fetchUsageSnapshot` o
recurrir a la resolución genérica de credenciales para las superficies de uso/estado.
Devuelva `{ token, accountId?, subscriptionType?, rateLimitTier? }` cuando el proveedor
tenga una credencial de uso (los metadatos opcionales del plan pasan a
`fetchUsageSnapshot`), devuelva
`{ handled: true }` cuando la autenticación de uso gestionada por el proveedor haya
procesado la solicitud y deba suprimir la alternativa genérica de clave de API/OAuth,
y devuelva `null` o `undefined`
cuando el proveedor no haya gestionado la autenticación de uso.

Declare las credenciales de organización o facturación en el manifiesto
`providerUsageAuthEnvVars`. Esto permite que las superficies genéricas de descubrimiento y
eliminación de secretos las reconozcan sin convertirlas en candidatas para la
autenticación de inferencia.

### Ejemplo de proveedor

```ts
api.registerProvider({
  id: "example-proxy",
  label: "Example Proxy",
  auth: [],
  catalog: {
    order: "simple",
    run: async (ctx) => {
      const apiKey = ctx.resolveProviderApiKey("example-proxy").apiKey;
      if (!apiKey) {
        return null;
      }
      return {
        provider: {
          baseUrl: "https://proxy.example.com/v1",
          apiKey,
          api: "openai-completions",
          models: [{ id: "auto", name: "Auto" }],
        },
      };
    },
  },
  resolveDynamicModel: (ctx) => ({
    id: ctx.modelId,
    name: ctx.modelId,
    provider: "example-proxy",
    api: "openai-completions",
    baseUrl: "https://proxy.example.com/v1",
    reasoning: false,
    input: ["text"],
    cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
    contextWindow: 128000,
    maxTokens: 8192,
  }),
  prepareRuntimeAuth: async (ctx) => {
    const exchanged = await exchangeToken(ctx.apiKey);
    return {
      apiKey: exchanged.token,
      baseUrl: exchanged.baseUrl,
      expiresAt: exchanged.expiresAt,
    };
  },
  resolveUsageAuth: async (ctx) => {
    const auth = await ctx.resolveOAuthToken();
    return auth ? { token: auth.token } : null;
  },
  fetchUsageSnapshot: async (ctx) => {
    return await fetchExampleProxyUsage(ctx.token, ctx.timeoutMs, ctx.fetchFn);
  },
});
```

### Ejemplos integrados

Los plugins de proveedores incluidos combinan los hooks anteriores para adaptarse a
las necesidades de catálogo, autenticación, razonamiento, reproducción y uso de cada
proveedor. El conjunto de hooks autoritativo reside en cada plugin bajo
`extensions/`; esta página ilustra las estructuras en lugar de reproducir la
lista.

<AccordionGroup>
  <Accordion title="Proveedores de catálogo de paso directo">
    OpenRouter, Kilocode, Z.AI y xAI registran `catalog` junto con
    `resolveDynamicModel` / `prepareDynamicModel` para poder presentar los
    identificadores de modelos ascendentes antes que el catálogo estático de OpenClaw.
  </Accordion>
  <Accordion title="Proveedores de OAuth y endpoints de uso">
    GitHub Copilot, Gemini CLI, ChatGPT Codex, MiniMax, Xiaomi y z.ai combinan
    `prepareRuntimeAuth` o `formatApiKey` con `resolveUsageAuth` +
    `fetchUsageSnapshot` para gestionar el intercambio de tokens y la integración
    de `/usage`.
  </Accordion>
  <Accordion title="Familias de limpieza de reproducciones y transcripciones">
    Las familias compartidas con nombre (`google-gemini`, `passthrough-gemini`,
    `anthropic-by-model`, `hybrid-anthropic-openai`) permiten que los proveedores adopten
    la política de transcripciones mediante `buildReplayPolicy`, en lugar de que
    cada plugin vuelva a implementar la limpieza.
  </Accordion>
  <Accordion title="Proveedores solo de catálogo">
    `byteplus`, `cloudflare-ai-gateway`, `huggingface`, `kimi-coding`, `nvidia`,
    `qianfan`, `synthetic`, `together`, `venice`, `vercel-ai-gateway` y
    `volcengine` solo registran `catalog` y utilizan el bucle de inferencia compartido.
  </Accordion>
  <Accordion title="Ayudantes de flujo específicos de Anthropic">
    Los encabezados beta, `/fast` / `serviceTier` y `context1m` residen en la
    interfaz pública `api.ts` / `contract-api.ts` del plugin de Anthropic
    (`wrapAnthropicProviderStream`, `resolveAnthropicBetas`,
    `resolveAnthropicFastMode`, `resolveAnthropicServiceTier`), en lugar de
    en el SDK genérico.
  </Accordion>
</AccordionGroup>

## Ayudantes de tiempo de ejecución

Los plugins pueden acceder a determinados ayudantes del núcleo mediante `api.runtime`. Para TTS:

```ts
const clip = await api.runtime.tts.textToSpeech({
  text: "Hello from OpenClaw",
  cfg: api.config,
});

const result = await api.runtime.tts.textToSpeechTelephony({
  text: "Hello from OpenClaw",
  cfg: api.config,
});

const voices = await api.runtime.tts.listVoices({
  provider: "elevenlabs",
  cfg: api.config,
});
```

Notas:

- `textToSpeech` devuelve la carga útil de salida TTS normal del núcleo para superficies de archivos/notas de voz.
- Utiliza la configuración `tts` y la selección de proveedor del núcleo.
- Devuelve un búfer de audio PCM y la frecuencia de muestreo. Los plugins deben remuestrear/codificar para los proveedores.
- `listVoices` es opcional para cada proveedor. Se utiliza para selectores de voz o flujos de configuración gestionados por el proveedor.
- El núcleo pasa un plazo de solicitud resuelto a los hooks `listVoices` del proveedor; los ajustes de tiempo de espera específicos del proveedor pueden reemplazarlo.
- Las listas de voces pueden incluir metadatos más completos, como configuración regional, género y etiquetas de personalidad, para selectores conscientes del proveedor.
- OpenAI y ElevenLabs admiten actualmente telefonía. Microsoft no.

Los plugins también pueden registrar proveedores de voz mediante `api.registerSpeechProvider(...)`.

```ts
api.registerSpeechProvider({
  id: "acme-speech",
  label: "Acme Speech",
  isConfigured: ({ config }) => Boolean(config.messages?.tts),
  synthesize: async (req) => {
    return {
      audioBuffer: Buffer.from([]),
      outputFormat: "mp3",
      fileExtension: ".mp3",
      voiceCompatible: false,
    };
  },
});
```

Notas:

- Mantenga en el núcleo la política de TTS, las alternativas y la entrega de respuestas.
- Utilice proveedores de voz para el comportamiento de síntesis gestionado por el proveedor.
- La entrada heredada `edge` de Microsoft se normaliza al identificador de proveedor `microsoft`.
- El modelo de propiedad preferido está orientado a empresas: un plugin de proveedor puede gestionar
  proveedores de texto, voz, imágenes y medios futuros a medida que OpenClaw añada esos
  contratos de capacidades.

Para la comprensión de imágenes/audio/vídeo, los plugins registran un proveedor tipado
de comprensión multimedia en lugar de un contenedor genérico de clave/valor:

```ts
api.registerMediaUnderstandingProvider({
  id: "google",
  capabilities: ["image", "audio", "video"],
  describeImage: async (req) => ({ text: "..." }),
  transcribeAudio: async (req) => ({ text: "..." }),
  describeVideo: async (req) => ({ text: "..." }),
});
```

Notas:

- Mantenga en el núcleo la orquestación, las alternativas, la configuración y la conexión con canales.
- Mantenga el comportamiento del proveedor en el plugin del proveedor.
- La ampliación aditiva debe conservar los tipos: nuevos métodos opcionales, nuevos campos
  de resultados opcionales y nuevas capacidades opcionales.
- La generación de vídeo ya sigue el mismo patrón:
  - el núcleo gestiona el contrato de capacidades y el ayudante de tiempo de ejecución
  - los plugins de proveedores registran `api.registerVideoGenerationProvider(...)`
  - los plugins de funcionalidades/canales consumen `api.runtime.videoGeneration.*`

Para los ayudantes de tiempo de ejecución de comprensión multimedia, los plugins pueden llamar a:

```ts
const image = await api.runtime.mediaUnderstanding.describeImageFile({
  filePath: "/tmp/inbound-photo.jpg",
  cfg: api.config,
  agentDir: "/tmp/agent",
});

const video = await api.runtime.mediaUnderstanding.describeVideoFile({
  filePath: "/tmp/inbound-video.mp4",
  cfg: api.config,
});

const extraction = await api.runtime.mediaUnderstanding.extractStructuredWithModel({
  provider: "codex",
  model: "gpt-5.6-sol",
  input: [
    {
      type: "image",
      buffer: receiptImageBuffer,
      fileName: "receipt.png",
      mime: "image/png",
    },
    { type: "text", text: "Use the printed fields as the source of truth." },
  ],
  instructions: "Return entities and searchable tags.",
  schemaName: "example.evidence",
  jsonSchema: {
    type: "object",
    properties: {
      entities: { type: "array", items: { type: "string" } },
      tags: { type: "array", items: { type: "string" } },
    },
  },
  cfg: api.config,
});
```

Para la transcripción de audio, los plugins pueden utilizar el tiempo de ejecución de
comprensión multimedia o el alias STT anterior:

```ts
const { text } = await api.runtime.mediaUnderstanding.transcribeAudioFile({
  filePath: "/tmp/inbound-audio.ogg",
  cfg: api.config,
  // Optional when MIME cannot be inferred reliably:
  mime: "audio/ogg",
});
```

Notas:

- `api.runtime.mediaUnderstanding.*` es la superficie compartida preferida para
  la comprensión de imágenes/audio/vídeo.
- `extractStructuredWithModel(...)` es la interfaz orientada a plugins para la extracción acotada,
  centrada primero en imágenes y gestionada por el proveedor. Incluya al menos una entrada de imagen;
  las entradas de texto son contexto complementario. Los plugins de producto gestionan sus rutas y
  esquemas, mientras que OpenClaw gestiona el límite entre el proveedor y el tiempo de ejecución.
- Utiliza la configuración de audio de comprensión multimedia del núcleo (`tools.media.audio`) y el orden de alternativas de proveedores.
- Devuelve `{ text: undefined }` cuando no se produce ninguna salida de transcripción (por ejemplo, una entrada omitida/no compatible).

Los plugins también pueden iniciar ejecuciones de subagentes en segundo plano mediante `api.runtime.subagent`:

```ts
const result = await api.runtime.subagent.run({
  sessionKey: "agent:main:subagent:search-helper",
  message: "Expand this query into focused follow-up searches.",
  toolsAlsoAllow: ["my_plugin_progress"],
  provider: "openai",
  model: "gpt-4.1-mini",
  deliver: false,
});
```

Notas:

- `provider` y `model` son reemplazos opcionales por ejecución, no cambios persistentes de la sesión.
- `toolsAlsoAllow` acepta nombres de herramientas exactos y con propietario único registrados por el plugin llamador. Se rechazan los nombres del núcleo y los ambiguos. Se añade al perfil normal, pero las listas de permisos y denegaciones del operador siguen siendo autoritativas.
- OpenClaw solo respeta esos campos de reemplazo para llamadores de confianza.
- Para las ejecuciones alternativas gestionadas por plugins, los operadores deben habilitarlas explícitamente con `plugins.entries.<id>.subagent.allowModelOverride: true`.
- Utilice `plugins.entries.<id>.subagent.allowedModels` para restringir los plugins de confianza a destinos canónicos `provider/model` específicos, o `"*"` para permitir explícitamente cualquier destino.
- Las ejecuciones de subagentes de plugins que no son de confianza siguen funcionando, pero las solicitudes de reemplazo se rechazan en lugar de recurrir silenciosamente a una alternativa.
- Las sesiones de subagentes creadas por plugins se etiquetan con el identificador del plugin creador. El mecanismo alternativo `api.runtime.subagent.deleteSession(...)` solo puede eliminar esas sesiones propias; la eliminación arbitraria de sesiones sigue requiriendo una solicitud del Gateway con ámbito de administrador.

Para la búsqueda web, los plugins pueden utilizar el ayudante de tiempo de ejecución
compartido en lugar de acceder al cableado de herramientas del agente:

```ts
const providers = api.runtime.webSearch.listProviders({
  config: api.config,
});

const result = await api.runtime.webSearch.search({
  config: api.config,
  args: {
    query: "OpenClaw plugin runtime helpers",
    count: 5,
  },
});
```

Los plugins también pueden registrar proveedores de búsqueda web mediante
`api.registerWebSearchProvider(...)`.

Notas:

- Mantenga en el núcleo la selección de proveedores, la resolución de credenciales y la semántica compartida de las solicitudes.
- Utilice proveedores de búsqueda web para transportes de búsqueda específicos del proveedor.
- `api.runtime.webSearch.*` es la superficie compartida preferida para plugins de funcionalidades/canales que necesitan comportamiento de búsqueda sin depender del contenedor de herramientas del agente.

### `api.runtime.imageGeneration`

```ts
const result = await api.runtime.imageGeneration.generate({
  config: api.config,
  args: { prompt: "A friendly lobster mascot", size: "1024x1024" },
});

const providers = api.runtime.imageGeneration.listProviders({
  config: api.config,
});
```

- `generate(...)`: genera una imagen mediante la cadena de proveedores de generación de imágenes configurada.
- `listProviders(...)`: enumera los proveedores de generación de imágenes disponibles y sus capacidades.

## Rutas HTTP del Gateway

Los plugins pueden exponer endpoints HTTP con `api.registerHttpRoute(...)`.

```ts
api.registerHttpRoute({
  path: "/acme/webhook",
  auth: "plugin",
  match: "exact",
  handler: async (_req, res) => {
    res.statusCode = 200;
    res.end("ok");
    return true;
  },
});
```

Campos de la ruta:

- `path`: ruta dentro del servidor HTTP del Gateway.
- `auth`: obligatorio, `"gateway"` o `"plugin"`. Use `"gateway"` para exigir la autenticación normal del Gateway, o `"plugin"` para la autenticación o verificación de Webhooks gestionada por el plugin.
- `match`: opcional. `"exact"` (predeterminado) o `"prefix"`.
- `handleUpgrade`: controlador opcional para solicitudes de actualización a WebSocket en la misma ruta.
- `replaceExisting`: opcional. Permite que el mismo plugin sustituya su propio registro de ruta existente.
- `handler`: devuelve `true` cuando la ruta haya gestionado la solicitud.

Notas:

- `api.registerHttpHandler(...)` se eliminó y provocará un error al cargar el plugin. Use `api.registerHttpRoute(...)` en su lugar.
- Las rutas de plugins deben declarar `auth` explícitamente.
- Los conflictos exactos de `path + match` se rechazan salvo que se use `replaceExisting: true`, y un plugin no puede sustituir la ruta de otro plugin.
- Las rutas superpuestas con distintos niveles de `auth` se rechazan. Mantenga las cadenas de continuidad `exact`/`prefix` únicamente en el mismo nivel de autenticación.
- Las rutas `auth: "plugin"` **no** reciben automáticamente ámbitos de ejecución del operador. Están destinadas a Webhooks o a la verificación de firmas gestionados por el plugin, no a llamadas privilegiadas a los auxiliares del Gateway.
- Las rutas `auth: "gateway"` se ejecutan dentro del ámbito de ejecución de una solicitud del Gateway. La superficie predeterminada (`gatewayRuntimeScopeSurface: "write-default"`) es intencionadamente conservadora:
  - la autenticación de portador mediante secreto compartido (`gateway.auth.mode = "token"` / `"password"`) y cualquier método de autenticación que no sea de proxy de confianza obtienen un único ámbito `operator.write`, incluso si el llamador envía `x-openclaw-scopes`
  - los llamadores `trusted-proxy` sin un encabezado `x-openclaw-scopes` explícito también conservan la superficie heredada limitada a `operator.write`
  - los llamadores `trusted-proxy` que sí envían `x-openclaw-scopes` obtienen en su lugar los ámbitos declarados
  - una ruta puede optar por `gatewayRuntimeScopeSurface: "trusted-operator"` para respetar siempre `x-openclaw-scopes` en los modos de autenticación que incluyen identidad (y recurrir al conjunto completo de ámbitos predeterminados de la CLI cuando el encabezado no está presente)
- Las pestañas externas aisladas de la interfaz de control respaldadas por rutas `auth: "gateway"` usan una concesión de cookie firmada y de corta duración, emitida únicamente mediante un arranque autenticado; las pestañas con autenticación de plugin conservan su ruta directa de iframe. Antes del montaje, el elemento principal ejecuta una comprobación propiedad de la ruta dentro del mismo entorno aislado opaco y bloquea el acceso cuando la política de privacidad del navegador impide usar la cookie. La concesión está vinculada al plugin propietario, a la raíz de la ruta coincidente y a la generación de autenticación actual; el nombre aleatorio por proceso de su cookie evita que Gateways de confianza del mismo host se sobrescriban entre sí, pero las cookies nunca aíslan los puertos TCP. Por tanto, el nombre de host del Gateway constituye un único límite de credenciales: no aloje conjuntamente servicios que no confíen entre sí en ese nombre de host, ni siquiera en otros puertos. El enrutamiento rechaza reutilizarla en una ruta anidada propiedad de otro plugin. Como los descendientes del entorno aislado son sitios distintos a efectos de las cookies, la concesión solo acepta `GET` y `HEAD` con `operator.read`; las mutaciones y actualizaciones a WebSocket permanecen en superficies con autenticación explícita del Gateway. La cookie no puede usar CHIPS intencionadamente: los navegadores actuales incluyen un bit de ancestro entre sitios en la clave de partición, por lo que los marcos aislados opacos anidados perderían el acceso a los recursos de la misma ruta. La cookie requiere un contexto seguro y permiso del navegador para usar cookies entre sitios, por lo que las pestañas externas con autenticación del Gateway no están disponibles en orígenes LAN con HTTP simple ni cuando se bloquean por completo las cookies de terceros; use HTTPS/Tailscale Serve o un bucle local de confianza para el navegador con una política de cookies compatible.
- La concesión impide la divulgación del token de portador del Gateway y la reutilización accidental de rutas o ámbitos; no crea un límite de seguridad entre plugins nativos. El código de los plugins nativos y el contenido de la interfaz que sirven siguen formando parte del mismo límite de confianza del plugin dentro del proceso.
- Regla práctica: no suponga que una ruta de plugin con autenticación del Gateway sea implícitamente una superficie administrativa. Si la ruta necesita un comportamiento exclusivo para administradores, opte por la superficie de ámbitos `trusted-operator`, exija un modo de autenticación que incluya identidad y documente el contrato explícito del encabezado `x-openclaw-scopes`.
- Después de encontrar la ruta y autenticar la solicitud, los controladores normales participan en la admisión de trabajo raíz del Gateway. Un Gateway preparado o en proceso de reinicio devuelve `503` antes de invocar el controlador. La única excepción limitada es una ruta `auth: "gateway"` autorizada por el manifiesto que también opte por la superficie específica de la ruta `trusted-operator`; permanece accesible para que el enrutamiento del control de suspensión no quede bloqueado, mientras que las demás rutas normales del mismo plugin permanecen detrás del límite de admisión. La propiedad de `handleUpgrade` de WebSocket utiliza el mismo límite de admisión atómico; una vez que el controlador acepta un socket, su ciclo de vida posterior pertenece al plugin y este límite no realiza su seguimiento.

## Rutas de importación del SDK de plugins

Use subrutas específicas del SDK en lugar del barrel raíz monolítico
`openclaw/plugin-sdk` al crear plugins nuevos. Subrutas principales:

| Subruta                            | Finalidad                                      |
| ---------------------------------- | -------------------------------------------- |
| `openclaw/plugin-sdk/plugin-entry` | Primitivas de registro de plugins               |
| `openclaw/plugin-sdk/channel-core` | Auxiliares de entrada y compilación de canales                  |
| `openclaw/plugin-sdk/core`         | Auxiliares compartidos genéricos y contrato general |

Los plugins de canal eligen entre una familia de interfaces específicas: `channel-setup`,
`setup-runtime`, `setup-tools`, `channel-pairing`,
`channel-contract`, `channel-feedback`, `channel-inbound`, `channel-outbound`,
`command-auth`, `secret-input`, `webhook-ingress`,
`channel-targets` y `channel-actions`. El comportamiento de aprobación debe consolidarse
en un único contrato `approvalCapability` en lugar de mezclarse entre campos
de plugins no relacionados. Consulte [Plugins de canal](/es/plugins/sdk-channel-plugins).

Los auxiliares de ejecución y configuración se encuentran en subrutas específicas de `*-runtime`
correspondientes (`approval-runtime`, `agent-runtime`, `lazy-runtime`, `directory-runtime`,
`text-runtime`, `runtime-store`, `system-event-runtime`, `heartbeat-runtime`,
`channel-activity-runtime`, etc.). Prefiera `config-contracts`,
`plugin-config-runtime`, `runtime-config-snapshot` y `config-mutation`
en lugar del amplio barrel de compatibilidad `config-runtime`.

<Info>
`openclaw/plugin-sdk/channel-lifecycle`, las pequeñas fachadas auxiliares de canales,
`openclaw/plugin-sdk/config-runtime` y `openclaw/plugin-sdk/infra-runtime`
son adaptadores de compatibilidad obsoletos para plugins antiguos. El código nuevo debe importar
primitivas genéricas más específicas en su lugar.
</Info>

Puntos de entrada internos del repositorio (por raíz de paquete de plugin incluido):

- `index.js` — punto de entrada del plugin incluido
- `api.js` — barrel de auxiliares y tipos
- `runtime-api.js` — barrel exclusivo de ejecución
- `setup-entry.js` — punto de entrada de configuración del plugin

Los plugins externos solo deben importar subrutas de `openclaw/plugin-sdk/*`. Nunca
importe el `src/*` del paquete de otro plugin desde el núcleo ni desde otro plugin.
Los puntos de entrada cargados mediante fachadas prefieren la instantánea activa de la configuración de ejecución cuando
existe y, en caso contrario, recurren al archivo de configuración resuelto en el disco.

Existen subrutas específicas de capacidades como `image-generation`, `media-understanding`
y `speech` porque los plugins incluidos las usan actualmente. No son
automáticamente contratos externos inmutables a largo plazo; consulte la página de referencia
del SDK correspondiente antes de depender de ellas.

## Esquemas de la herramienta de mensajes

Los plugins deben ser propietarios de las contribuciones al esquema `describeMessageTool(...)`
específicas del canal para primitivas que no sean mensajes, como reacciones, lecturas y encuestas.
La presentación compartida de envíos debe usar el contrato genérico `MessagePresentation`
en lugar de campos de botones, componentes, bloques o tarjetas nativos del proveedor.
Consulte [Presentación de mensajes](/es/plugins/message-presentation) para conocer el contrato,
las reglas de degradación, la asignación de proveedores y la lista de comprobación para autores de plugins.

Los plugins con capacidad de envío declaran lo que pueden representar mediante capacidades de mensajes:

- `presentation` para bloques de presentación semántica (`text`, `context`,
  `divider`, `chart`, `table`, `buttons`, `select`)
- `delivery-pin` para solicitudes de entrega fijada

El núcleo decide si representa la presentación de forma nativa o la degrada a texto.
No exponga vías de escape de interfaz nativas del proveedor desde la herramienta genérica de mensajes.
Los auxiliares obsoletos del SDK para esquemas nativos heredados siguen exportándose para plugins
de terceros existentes, pero los plugins nuevos no deben usarlos.

## Resolución de destinos de canales

Los plugins de canal deben ser propietarios de la semántica de destino específica del canal. Mantenga genérico
el host de salida compartido y use la superficie del adaptador de mensajería para las reglas del proveedor:

- `messaging.inferTargetChatType({ to })` decide si un destino normalizado
  debe tratarse como `direct`, `group` o `channel` antes de buscarlo en el directorio.
- `messaging.targetResolver.looksLikeId(raw, normalized)` indica al núcleo si una
  entrada debe pasar directamente a una resolución similar a un identificador en lugar de buscar en el directorio.
- `messaging.targetResolver.reservedLiterals` enumera las palabras sin formato que son
  referencias de canal o sesión para ese proveedor. La resolución conserva las entradas
  configuradas del directorio antes de rechazar los literales reservados y, después, se bloquea
  si no encuentra una coincidencia en el directorio.
- `messaging.targetResolver.resolveTarget(...)` es la alternativa del plugin cuando
  el núcleo necesita una resolución final propiedad del proveedor tras la normalización o después de no encontrar
  una coincidencia en el directorio.
- `messaging.resolveOutboundSessionRoute(...)` controla la construcción de rutas
  de sesión específicas del proveedor una vez resuelto el destino.

División recomendada:

- Use `inferTargetChatType` para las decisiones de categoría que deban tomarse antes de
  buscar pares o grupos.
- Use `looksLikeId` para comprobar si «esto debe tratarse como un identificador de destino explícito o nativo».
- Use `resolveTarget` como alternativa de normalización específica del proveedor, no para
  búsquedas amplias en el directorio.
- Mantenga los identificadores nativos del proveedor, como identificadores de chat, de hilos, JID, nombres de usuario e identificadores
  de salas, dentro de los valores `target` o de parámetros específicos del proveedor, no en campos
  genéricos del SDK.

## Directorios respaldados por la configuración

Los plugins que derivan entradas de directorio de la configuración deben conservar esa lógica en el
plugin y reutilizar los auxiliares compartidos de
`openclaw/plugin-sdk/directory-runtime`.

Use esto cuando un canal necesite pares o grupos respaldados por la configuración, como:

- pares de mensajes directos controlados mediante una lista de permitidos
- asignaciones configuradas de canales o grupos
- alternativas de directorio estáticas limitadas a una cuenta

Los auxiliares compartidos de `directory-runtime` solo gestionan operaciones genéricas:

- filtrado de consultas
- aplicación de límites
- auxiliares de desduplicación y normalización
- creación de `ChannelDirectoryEntry[]`

La inspección de cuentas y la normalización de identificadores específicas del canal deben permanecer en la
implementación del plugin.

## Catálogos de proveedores

Los plugins de proveedores pueden definir catálogos de modelos para inferencia con
`registerProvider({ catalog: { run(...) { ... } } })`.

`catalog.run(...)` devuelve la misma estructura que OpenClaw escribe en
`models.providers`:

- `{ provider }` para una entrada de proveedor
- `{ providers }` para varias entradas de proveedor

Use `catalog` cuando el plugin sea propietario de identificadores de modelos específicos del proveedor, valores predeterminados de la URL base
o metadatos de modelos sujetos a autenticación.

`catalog.order` controla cuándo se combina el catálogo de un plugin con respecto a los proveedores implícitos
integrados de OpenClaw:

- `simple`: proveedores simples basados en claves de API o variables de entorno
- `profile`: proveedores que aparecen cuando existen perfiles de autenticación
- `paired`: proveedores que sintetizan varias entradas de proveedor relacionadas
- `late`: última pasada, después de los demás proveedores implícitos

Los proveedores posteriores prevalecen en caso de colisión de claves, por lo que los plugins pueden sobrescribir intencionadamente una
entrada de proveedor integrada con el mismo identificador de proveedor.

Los plugins también pueden publicar filas de modelos de solo lectura mediante
`api.registerModelCatalogProvider({ provider, kinds, staticCatalog, liveCatalog
})`. Esta es la vía futura para las superficies de lista/ayuda/selector y admite
filas `text`, `voice`, `image_generation`, `video_generation` y `music_generation`.
Los plugins de proveedores siguen siendo responsables de las llamadas activas a endpoints, el intercambio de tokens y
la asignación de respuestas del proveedor; el núcleo es responsable de la forma común de las filas, las etiquetas de origen y
el formato de la ayuda de las herramientas multimedia. Los registros de proveedores de generación multimedia sintetizan
automáticamente filas estáticas del catálogo a partir de `defaultModel`, `models` y
`capabilities`.

Compatibilidad:

- `discovery` sigue funcionando como alias heredado, pero emite una advertencia de obsolescencia
- si se registran tanto `catalog` como `discovery`, OpenClaw usa `catalog`
  y emite una advertencia
- `augmentModelCatalog` está obsoleto; los proveedores incluidos deben publicar
  filas complementarias mediante `registerModelCatalogProvider`

## Inspección de canales de solo lectura

Si el plugin registra un canal, se recomienda implementar
`plugin.config.inspectAccount(cfg, accountId)` junto con `resolveAccount(...)`.

Motivos:

- `resolveAccount(...)` es la ruta de ejecución. Puede asumir que las credenciales
  están completamente materializadas y fallar de inmediato cuando faltan secretos obligatorios.
- Las rutas de comandos de solo lectura, como `openclaw status`, `openclaw status --all`,
  `openclaw channels status`, `openclaw channels resolve` y los flujos de reparación de doctor/configuración,
  no deberían necesitar materializar credenciales de ejecución solo para
  describir la configuración.

Comportamiento recomendado de `inspectAccount(...)`:

- Devuelva únicamente el estado descriptivo de la cuenta.
- Conserve `enabled` y `configured`.
- Incluya campos de origen/estado de las credenciales cuando corresponda, como:
  - `tokenSource`, `tokenStatus`
  - `botTokenSource`, `botTokenStatus`
  - `appTokenSource`, `appTokenStatus`
  - `signingSecretSource`, `signingSecretStatus`
- No es necesario devolver los valores sin procesar de los tokens solo para informar sobre la disponibilidad
  de solo lectura. Devolver `tokenStatus: "available"` (y el campo de origen
  correspondiente) es suficiente para los comandos de estado.
- Use `configured_unavailable` cuando una credencial esté configurada mediante SecretRef, pero
  no esté disponible en la ruta de comandos actual.

Esto permite que los comandos de solo lectura informen «configurada, pero no disponible en esta ruta
de comandos» en lugar de bloquearse o indicar erróneamente que la cuenta no está configurada.

## Paquetes de plugins

Un directorio de plugin puede incluir un `package.json` con `openclaw.extensions`:

```json
{
  "name": "my-pack",
  "openclaw": {
    "extensions": ["./src/safety.ts", "./src/tools.ts"],
    "setupEntry": "./src/setup-entry.ts"
  }
}
```

Cada entrada se convierte en un plugin. Si el paquete enumera varias extensiones, el identificador del plugin
se convierte en `<manifestOrPackageName>/<fileBase>` (el identificador del manifiesto prevalece cuando
está presente; de lo contrario, se usa el nombre `package.json` sin ámbito).

Si el plugin importa dependencias de npm, instálelas en ese directorio para que
`node_modules` esté disponible (`npm install` / `pnpm install`).

Medida de seguridad: cada entrada `openclaw.extensions` debe permanecer dentro del directorio del plugin
después de resolver los enlaces simbólicos. Se rechazan las entradas que escapen del directorio del paquete.

Nota de seguridad: `openclaw plugins install` instala las dependencias del plugin con un
`npm install --omit=dev --ignore-scripts` local del proyecto (sin scripts del ciclo de vida
ni dependencias de desarrollo durante la ejecución), e ignora la configuración global heredada de instalación de npm.
Mantenga los árboles de dependencias de los plugins como «JS/TS puro» y evite paquetes que requieran
compilaciones `postinstall`.

Opcional: `openclaw.setupEntry` puede apuntar a un módulo ligero exclusivo para la configuración.
Cuando OpenClaw necesita superficies de configuración para un plugin de canal deshabilitado, o
cuando un plugin de canal está habilitado pero aún no está configurado, carga `setupEntry`
en lugar de la entrada completa del plugin. Esto reduce la carga del inicio y la configuración
cuando la entrada principal del plugin también conecta herramientas, hooks u otro código exclusivo
de la ejecución.

Opcional: `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen`
puede incorporar un plugin de canal a la misma ruta `setupEntry` durante la fase de inicio
anterior a la escucha del gateway, incluso cuando el canal ya está configurado.

Use esta opción únicamente cuando `setupEntry` cubra por completo la superficie de inicio que debe existir
antes de que el gateway comience a escuchar. En la práctica, esto significa que la entrada de configuración
debe registrar todas las capacidades propiedad del canal de las que depende el inicio, como:

- el propio registro del canal
- todas las rutas HTTP que deban estar disponibles antes de que el gateway comience a escuchar
- todos los métodos, herramientas o servicios del gateway que deban existir durante ese mismo periodo

Si la entrada completa sigue siendo propietaria de alguna capacidad de inicio obligatoria, no habilite
esta opción. Mantenga el plugin con el comportamiento predeterminado y permita que OpenClaw cargue la
entrada completa durante el inicio.

Los canales incluidos también pueden publicar auxiliares de superficie de contrato exclusivos para la configuración que el núcleo
puede consultar antes de cargar la ejecución completa del canal. La superficie actual de
promoción de configuración es:

- `singleAccountKeysToMove`
- `namedAccountPromotionKeys`
- `resolveSingleAccountPromotionTarget(...)`

El núcleo usa esa superficie cuando necesita promover la configuración heredada de un canal de una sola cuenta
a `channels.<id>.accounts.*` sin cargar la entrada completa del plugin.
Matrix es el ejemplo incluido actual: mueve únicamente las claves de autenticación/inicialización a una
cuenta promovida con nombre cuando ya existen cuentas con nombre, y puede conservar una
clave configurada de cuenta predeterminada no canónica en lugar de crear siempre
`accounts.default`.

Esos adaptadores de parches de configuración mantienen diferido el descubrimiento de superficies de contrato incluidas. El tiempo
de importación sigue siendo reducido; la superficie de promoción solo se carga en el primer uso, en lugar de
volver a ejecutar el inicio del canal incluido al importar el módulo.

Cuando esas superficies de inicio incluyan métodos RPC del gateway, manténgalos bajo un
prefijo específico del plugin. Los espacios de nombres administrativos del núcleo (`config.*`,
`exec.approvals.*`, `wizard.*`, `update.*`) permanecen reservados y siempre se resuelven
como `operator.admin`, incluso si un plugin solicita un ámbito más restringido.

Ejemplo:

```json
{
  "name": "@scope/my-channel",
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "startup": {
      "deferConfiguredChannelFullLoadUntilAfterListen": true
    }
  }
}
```

### Metadatos del catálogo de canales

Los plugins de canales pueden anunciar metadatos de configuración/descubrimiento mediante `openclaw.channel` y
sugerencias de instalación mediante `openclaw.install`. Esto evita que el catálogo del núcleo contenga datos.

Ejemplo:

```json
{
  "name": "@openclaw/nextcloud-talk",
  "openclaw": {
    "extensions": ["./index.ts"],
    "channel": {
      "id": "nextcloud-talk",
      "label": "Nextcloud Talk",
      "selectionLabel": "Nextcloud Talk (alojamiento propio)",
      "docsPath": "/channels/nextcloud-talk",
      "docsLabel": "nextcloud-talk",
      "blurb": "Chat con alojamiento propio mediante bots de webhook de Nextcloud Talk.",
      "order": 65,
      "aliases": ["nc-talk", "nc"]
    },
    "install": {
      "npmSpec": "@openclaw/nextcloud-talk",
      "localPath": "<bundled-plugin-local-path>",
      "defaultChoice": "npm"
    }
  }
}
```

Campos útiles de `openclaw.channel` adicionales al ejemplo mínimo:

- `detailLabel`: etiqueta secundaria para superficies de catálogo/estado más completas
- `docsLabel`: sobrescribe el texto del enlace a la documentación
- `preferOver`: identificadores de plugins/canales de menor prioridad que esta entrada del catálogo debe superar
- `selectionDocsPrefix`, `selectionDocsOmitLabel`, `selectionExtras`: controles de texto de la superficie de selección
- `markdownCapable`: marca el canal como compatible con Markdown para las decisiones de formato saliente
- `exposure.configured`: oculta el canal en las superficies de listado de canales configurados cuando se establece en `false`
- `exposure.setup`: oculta el canal en los selectores interactivos de configuración cuando se establece en `false`
- `exposure.docs`: marca el canal como interno/privado para las superficies de navegación de la documentación
- `quickstartAllowFrom`: incorpora el canal al flujo estándar de inicio rápido `allowFrom`
- `forceAccountBinding`: exige una vinculación explícita de la cuenta incluso cuando solo existe una
- `preferSessionLookupForAnnounceTarget`: prioriza la búsqueda de sesiones al resolver los destinos de anuncios

OpenClaw también puede combinar **catálogos de canales externos** (por ejemplo, una exportación del
registro MPM). Coloque un archivo JSON en una de estas ubicaciones:

- `~/.openclaw/mpm/plugins.json`
- `~/.openclaw/mpm/catalog.json`
- `~/.openclaw/plugins/catalog.json`

O haga que `OPENCLAW_PLUGIN_CATALOG_PATHS` (o `OPENCLAW_MPM_CATALOG_PATHS`) apunte a
uno o varios archivos JSON (delimitados por comas, puntos y coma o `PATH`). Cada archivo debe
contener `{ "entries": [ { "name": "@scope/pkg", "openclaw": { "channel": {...}, "install": {...} } } ] }`. El analizador también acepta `"packages"` o `"plugins"` como alias heredados de la clave `"entries"`.

Las entradas generadas del catálogo de canales y las entradas del catálogo de instalación de proveedores exponen
datos normalizados del origen de instalación junto al bloque `openclaw.install` sin procesar. Los
datos normalizados identifican si la especificación de npm es una versión exacta o un
selector flotante, si están presentes los metadatos de integridad esperados y si también hay disponible una
ruta de origen local. Cuando se conoce la identidad del catálogo/paquete, los
datos normalizados advierten si el nombre del paquete npm analizado difiere de dicha identidad.
También advierten cuando `defaultChoice` no es válido o apunta a un origen que no está
disponible, y cuando existen metadatos de integridad de npm sin un origen npm
válido. Los consumidores deben tratar `installSource` como un campo opcional aditivo para que
las entradas creadas manualmente y los adaptadores de catálogos no tengan que sintetizarlo.
Esto permite que la incorporación y los diagnósticos expliquen el estado del plano de origen sin
importar la ejecución del plugin.

Las entradas npm externas oficiales deben priorizar un `npmSpec` exacto junto con
`expectedIntegrity`. Los nombres de paquetes sin versión y las etiquetas de distribución siguen funcionando por
compatibilidad, pero muestran advertencias del plano de origen para que el catálogo pueda avanzar
hacia instalaciones fijadas y verificadas mediante integridad sin interrumpir los plugins existentes.
Cuando la incorporación instala desde una ruta de catálogo local, registra una entrada administrada
en el índice de plugins con `source: "path"` y un
`sourcePath` relativo al espacio de trabajo cuando sea posible. La ruta de carga operativa absoluta permanece en
`plugins.load.paths`; el registro de instalación evita duplicar rutas de la estación de trabajo local
en la configuración persistente. Esto mantiene las instalaciones de desarrollo local visibles para
los diagnósticos del plano de origen sin añadir una segunda superficie de divulgación de rutas
del sistema de archivos sin procesar. La tabla SQLite persistente `installed_plugin_index` es la
fuente de verdad de la instalación y puede actualizarse sin cargar los módulos de ejecución del plugin.
Su mapa `installRecords` es persistente incluso cuando falta el manifiesto de un plugin o
no es válido; su carga útil `plugins` es una vista reconstruible del manifiesto.

## Plugins del motor de contexto

Los plugins del motor de contexto son responsables de la orquestación del contexto de sesión para la ingesta, el ensamblaje
y la Compaction. Regístrelos desde el plugin con
`api.registerContextEngine(id, factory)` y, a continuación, seleccione el motor activo con
`plugins.slots.contextEngine`.

Use esta opción cuando el plugin necesite reemplazar o ampliar el pipeline de contexto
predeterminado, en lugar de limitarse a añadir búsqueda en memoria o hooks.

```ts
import { buildMemorySystemPromptAddition } from "openclaw/plugin-sdk/core";

export default function (api) {
  api.registerContextEngine("lossless-claw", (ctx) => ({
    info: { id: "lossless-claw", name: "Lossless Claw", ownsCompaction: true },
    async ingest() {
      return { ingested: true };
    },
    async assemble({ messages, sessionKey, availableTools, citationsMode }) {
      return {
        messages,
        estimatedTokens: 0,
        systemPromptAddition: buildMemorySystemPromptAddition({
          availableTools: availableTools ?? new Set(),
          citationsMode,
          agentSessionKey: sessionKey,
        }),
      };
    },
    async compact() {
      return { ok: true, compacted: false };
    },
  }));
}
```

La fábrica `ctx` expone valores opcionales `config`, `agentDir` y `workspaceDir`
para la inicialización durante la construcción.

El host completa la preparación asíncrona registrada del prompt de memoria antes de llamar a
`assemble()` de un motor no heredado. `buildMemorySystemPromptAddition(...)` permanece
síncrono y lee esa instantánea inmutable de la ejecución mientras `assemble()` está activo.
Pase sin cambios el contexto proporcionado de herramientas y citas para que la instantánea
no pueda cruzar los límites de la ejecución.

`assemble()` puede devolver `contextProjection` cuando el arnés activo tiene un
hilo persistente del backend. Omítalo para la proyección heredada por turno. Devuelva
`{ mode: "thread_bootstrap", epoch }` cuando el contexto ensamblado deba
inyectarse una vez en un hilo del backend y reutilizarse hasta que cambie la época. Cambie
la época después de que cambie el contexto semántico del motor, por ejemplo, tras una
pasada de Compaction gestionada por el motor. Los hosts pueden conservar los metadatos de llamadas a herramientas, la forma
de la entrada y los resultados censurados de las herramientas en una proyección de arranque del hilo para que los
hilos nuevos del backend mantengan la continuidad de las herramientas sin copiar cargas
sin procesar que contengan secretos.

Si el motor **no** controla el algoritmo de Compaction, mantenga `compact()`
implementado y deléguelo explícitamente:

```ts
import {
  buildMemorySystemPromptAddition,
  delegateCompactionToRuntime,
} from "openclaw/plugin-sdk/core";

export default function (api) {
  api.registerContextEngine("my-memory-engine", (ctx) => ({
    info: {
      id: "my-memory-engine",
      name: "My Memory Engine",
      ownsCompaction: false,
    },
    async ingest() {
      return { ingested: true };
    },
    async assemble({ messages, sessionKey, availableTools, citationsMode }) {
      return {
        messages,
        estimatedTokens: 0,
        systemPromptAddition: buildMemorySystemPromptAddition({
          availableTools: availableTools ?? new Set(),
          citationsMode,
          agentSessionKey: sessionKey,
        }),
      };
    },
    async compact(params) {
      return await delegateCompactionToRuntime(params);
    },
  }));
}
```

## Añadir una capacidad nueva

Cuando un plugin necesite un comportamiento que no encaje en la API actual, no eluda
el sistema de plugins accediendo de forma privada a sus componentes internos. Añada la capacidad que falta.

Secuencia recomendada:

1. **Defina el contrato del núcleo.** Decida qué comportamiento compartido debe controlar el núcleo:
   políticas, mecanismo alternativo, combinación de configuración, ciclo de vida, semántica orientada a canales y
   forma del asistente de tiempo de ejecución.
2. **Añada superficies tipadas de registro y tiempo de ejecución de plugins.** Amplíe
   `OpenClawPluginApi` y/o `api.runtime` con la superficie tipada de capacidad útil
   más pequeña.
3. **Conecte el núcleo y los consumidores de canales/funcionalidades.** Los canales y plugins de funcionalidades
   deben consumir la nueva capacidad a través del núcleo, no importando directamente una
   implementación de un proveedor.
4. **Registre las implementaciones de los proveedores.** A continuación, los plugins de los proveedores registran sus
   backends para la capacidad.
5. **Añada cobertura del contrato.** Añada pruebas para que la propiedad y la forma del registro
   permanezcan explícitas con el tiempo.

Así es como OpenClaw mantiene criterios definidos sin quedar codificado de forma rígida según la
visión de un solo proveedor. Consulte el [Recetario de capacidades](/es/plugins/adding-capabilities)
para ver una lista de comprobación concreta de archivos y un ejemplo desarrollado.

### Lista de comprobación de capacidades

Cuando añada una capacidad nueva, la implementación normalmente debe abarcar conjuntamente estas
superficies:

- tipos de contratos del núcleo en `src/<capability>/types.ts`
- ejecutor o asistente de tiempo de ejecución del núcleo en `src/<capability>/runtime.ts`
- superficie de registro de la API de plugins en `src/plugins/types.ts`
- conexión del registro de plugins en `src/plugins/registry.ts`
- exposición del tiempo de ejecución del plugin en `src/plugins/runtime/*` cuando los plugins de funcionalidades o canales
  necesiten consumirla
- asistentes de captura y pruebas en `src/test-utils/plugin-registration.ts`
- aserciones de propiedad y contrato en `src/plugins/contracts/registry.ts`
- documentación para operadores y plugins en `docs/`

Si falta alguna de esas superficies, normalmente indica que la capacidad
aún no está completamente integrada.

### Plantilla de capacidad

Patrón mínimo:

```ts
// core contract
export type VideoGenerationProviderPlugin = {
  id: string;
  label: string;
  generateVideo: (req: VideoGenerationRequest) => Promise<VideoGenerationResult>;
};

// plugin API
api.registerVideoGenerationProvider({
  id: "openai",
  label: "OpenAI",
  async generateVideo(req) {
    return await generateOpenAiVideo(req);
  },
});

// shared runtime helper for feature/channel plugins
const clip = await api.runtime.videoGeneration.generate({
  prompt: "Show the robot walking through the lab.",
  cfg,
});
```

Patrón de prueba del contrato (`src/plugins/contracts/registry.ts` expone búsquedas de
propiedad como `providerContractPluginIds`; las pruebas verifican que la lista
`contracts.videoGenerationProviders` de un plugin coincida con lo que realmente registra):

```ts
expect(pluginManifest.contracts?.videoGenerationProviders).toEqual(["openai"]);
```

Esto mantiene una regla sencilla:

- el núcleo controla el contrato y la orquestación de la capacidad
- los plugins de los proveedores controlan sus implementaciones
- los plugins de funcionalidades y canales consumen los asistentes de tiempo de ejecución
- las pruebas de contratos mantienen explícita la propiedad

## Contenido relacionado

- [Arquitectura de plugins](/es/plugins/architecture) — modelo público de capacidades y formas
- [Subrutas del SDK de plugins](/es/plugins/sdk-subpaths)
- [Configuración del SDK de plugins](/es/plugins/sdk-setup)
- [Creación de plugins](/es/plugins/building-plugins)
