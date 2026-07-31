---
read_when:
    - Quieres una cronología de tu día al estilo de Dayflow en la interfaz de control
    - Está habilitando o configurando el plugin Logbook incluido
    - Se buscan resúmenes para las reuniones diarias o recordar lo ocurrido durante el día basándose en la actividad en pantalla
summary: Diario de trabajo automático opcional creado a partir de capturas de pantalla periódicas
title: Plugin de registro
x-i18n:
    generated_at: "2026-07-26T05:20:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 19197e580421dfe81f82f8599578e4c68a15004813bb2b6c3de761c14f426b08
    source_path: plugins/logbook.md
    workflow: 16
---

El plugin Logbook convierte la actividad de la pantalla en un diario de trabajo automático. 
Captura instantáneas periódicas de la pantalla desde un Node emparejado, las resume como
observaciones con marcas de tiempo y crea tarjetas de cronología en la
[interfaz de control](/es/web/control-ui). También puede generar notas para la reunión diaria y
responder preguntas sobre un día registrado.

El estado propiedad de OpenClaw permanece en el Gateway, en `<state-dir>/logbook/`, pero
el procesamiento del modelo no es necesariamente local. Las capturas de pantalla muestreadas se envían a la
ruta de visión configurada; las observaciones y el texto de la cronología se envían al modelo
predeterminado del agente. Utilice rutas de modelos locales para ambas etapas si el contenido de la pantalla y
el texto de actividad derivado deben permanecer en la máquina.

Logbook está incluido y deshabilitado de forma predeterminada. Al habilitar el plugin, se permite que el
Gateway capture la pantalla porque `captureEnabled` tiene como valor predeterminado `true`.

## Antes de comenzar

Se necesita:

- Un Node conectado que exponga `screen.snapshot` o `logbook.snapshot`. El
  Node de la aplicación para macOS necesita permiso de grabación de pantalla. Un host de Node de macOS sin interfaz gráfica
  (`openclaw node host run`) obtiene el comando `logbook.snapshot` proporcionado por el plugin,
  respaldado por la herramienta del sistema `screencapture`.
- El plugin Codex incluido debe estar habilitado y autenticado. Actualmente, Codex proporciona
  el contrato de extracción estructurada de imágenes que requiere Logbook. Inicie sesión con
  `openclaw models auth login --provider openai`; consulte el
  [entorno de ejecución de Codex](/es/plugins/codex-harness) para conocer otras vías de autenticación.
- Un modelo predeterminado del agente que funcione. Logbook lo utiliza para sintetizar tarjetas, notas
  para la reunión diaria y preguntas y respuestas sobre el día después de la fase de visión.

## Inicio rápido

Habilite los plugins Codex y Logbook:

```bash
openclaw plugins enable codex
openclaw plugins enable logbook
```

Configure un modelo de visión explícito para que el inicio sea determinista:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
      logbook: {
        enabled: true,
        config: {
          visionModel: "codex/gpt-5.6-sol",
        },
      },
    },
  },
}
```

Si utiliza `plugins.allow`, incluya tanto `codex` como `logbook`. Reinicie el
Gateway después de cambiar la configuración del plugin; a continuación, inspeccione los registros
y abra el panel:

```bash
openclaw gateway restart
openclaw plugins inspect logbook --runtime --json
openclaw nodes status --connected
openclaw nodes describe --node <idOrNameOrIp>
openclaw dashboard
```

La descripción del Node debe incluir `screen.snapshot` o `logbook.snapshot`.
Los Nodes sin interfaz gráfica anuncian `logbook.snapshot` solo después de activar el plugin.
Consulte [Solución de problemas de Nodes](/es/nodes/troubleshooting) si falta el comando.

La pestaña Logbook solo aparece si el plugin está habilitado y hay una sesión de la
interfaz de control con `operator.write`. La fila de estado debe mostrar **Capturando** sin errores.
Aparece una tarjeta de cronología cuando se cierra la ventana de análisis, o se puede seleccionar
**Analizar ahora** después de capturar actividad.

## Cómo funciona

1. **Captura**: cada `captureIntervalSeconds` (30 s de forma predeterminada), Logbook invoca
   el comando de captura del Node seleccionado y almacena un fotograma JPEG escalado.
   Los fotogramas consecutivos idénticos se marcan como inactivos y se excluyen del análisis.
2. **Observación**: cuando transcurre una ventana de análisis (15 minutos de forma predeterminada), el
   plugin toma muestras de hasta 16 fotogramas activos y los envía al modelo de visión,
   que devuelve observaciones de actividad con marcas de tiempo («VS Code: editando
   store.ts, corrigiendo un error de tipo»). Un intervalo sin capturas superior a dos minutos o
   la medianoche local también cierra la ventana actual.
3. **Síntesis**: las observaciones y los últimos 45 minutos de tarjetas existentes se
   revisan para crear tarjetas de cronología (de 10 a 60 minutos cada una) con un título, un resumen,
   una categoría, la aplicación principal y cualquier distracción breve.
4. **Depuración**: se eliminan los fotogramas con más de `retentionDays` días de antigüedad (14 de forma predeterminada).
   Se conservan las tarjetas, las observaciones y las reuniones diarias almacenadas en caché.

Los límites de los días y los relojes de la cronología utilizan la zona horaria local del Gateway, no la
zona horaria del navegador. Los fotogramas y la base de datos SQLite de la cronología se encuentran en
`<state-dir>/logbook/`.

## Flujo de modelos y datos

Logbook utiliza dos rutas de modelos independientes:

| Etapa            | Datos enviados                                                 | Ruta del modelo                                                       |
| ---------------- | --------------------------------------------------------- | ----------------------------------------------------------------- |
| Observación          | Hasta 16 fotogramas JPEG muestreados y sus horas de captura     | `visionModel` o una entrada de Codex `tools.media` compatible y prestada |
| Síntesis de tarjetas | Observaciones con marcas de tiempo y tarjetas recientes de la cronología        | Modelo predeterminado del agente mediante el entorno de ejecución LLM del plugin                |
| Generación de reunión diaria | Tarjetas del día seleccionado y del día anterior               | Modelo predeterminado del agente mediante el entorno de ejecución LLM del plugin                |
| Preguntas sobre el día     | La pregunta, las tarjetas del día seleccionado y las observaciones recientes | Modelo predeterminado del agente mediante el entorno de ejecución LLM del plugin                |

La base de datos SQLite completa no se envía a ninguno de los modelos. Las capturas de pantalla sin procesar solo se envían
a la etapa de observación; la síntesis de tarjetas, la reunión diaria y las preguntas y respuestas reciben texto
derivado.

## Configuración

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
      logbook: {
        enabled: true,
        config: {
          captureEnabled: true,
          captureIntervalSeconds: 30,
          analysisIntervalMinutes: 15,
          nodeId: "my-mac",
          screenIndex: 0,
          maxWidth: 1440,
          visionModel: "codex/gpt-5.6-sol",
          retentionDays: 14,
        },
      },
    },
  },
}
```

Todas las claves de configuración de Logbook son opcionales. Los valores numéricos se redondean a enteros
y se limitan al intervalo admitido.

| Clave                       | Valor predeterminado | Intervalo o valores         | Comportamiento                                                                                     |
| ------------------------- | ------- | ----------------------- | -------------------------------------------------------------------------------------------- |
| `captureEnabled`          | `true`  | booleano                 | Interruptor maestro persistente para nuevas instantáneas; la cronología sigue disponible cuando es `false`      |
| `captureIntervalSeconds`  | `30`    | `5`-`600`               | Retraso entre intentos de captura                                                               |
| `analysisIntervalMinutes` | `15`    | `3`-`120`               | Ventana de observación objetivo; los intervalos y la medianoche pueden cerrarla antes                            |
| `nodeId`                  | sin establecer   | id o nombre visible del Node | Fija la captura a un Node conectado; la comparación no distingue mayúsculas de minúsculas                             |
| `screenIndex`             | `0`     | `0`-`16`                | Índice de pantalla basado en cero                                                                     |
| `maxWidth`                | `1440`  | `480`-`3840`            | Límite solicitado para el tamaño de captura; macOS sin interfaz gráfica lo aplica a la dimensión más grande               |
| `visionModel`             | sin establecer   | `provider/model`        | Ruta estructurada explícita; las referencias mal formadas pausan el análisis y los proveedores no compatibles hacen que fallen los lotes |
| `retentionDays`           | `14`    | `1`-`365`               | Elimina los fotogramas antiguos; las tarjetas, las observaciones y las reuniones diarias se conservan                                 |

Sin `nodeId`, Logbook prefiere un Node de aplicación conectado que exponga
`screen.snapshot` y, a continuación, recurre a un Node sin interfaz gráfica que exponga
`logbook.snapshot`. En una configuración sin fijar, un Node que falla pasa detrás de otros
Nodes aptos. El control de pausa del panel solo se aplica a la sesión y se restablece cuando se
reinicia el Gateway; utilice `captureEnabled: false` para una detención persistente.

### Selección del modelo de visión

Logbook resuelve el modelo de observación en este orden:

1. `plugins.entries.logbook.config.visionModel`
2. la primera entrada de Codex compatible con imágenes en `tools.media.models`

Se omiten otros proveedores multimedia porque actualmente no exponen el
contrato de extracción estructurada que requiere Logbook. Establecer
`tools.media.image.enabled: false` deshabilita los valores multimedia predeterminados prestados, pero un
`visionModel` explícito de Logbook sigue siendo aplicable.

## Pestaña del panel

- **Cronología**: tarjetas expandibles por actividad con colores de categoría, la aplicación
  principal, etiquetas de distracciones y un fotograma clave de instantánea.
- **Resumen del día**: proporción de concentración, desglose por categorías y aplicaciones principales.
- **Reunión diaria**: convierte el día de ayer y el de hoy en una actualización lista para pegar.
- **Preguntas sobre el día**: preguntas en lenguaje natural respondidas a partir de la cronología
  registrada («¿cuándo revisé el pull request del Gateway?»).
- **Analizar ahora**: cierra inmediatamente la ventana de captura actual en lugar de
  esperar al intervalo de análisis.

## Métodos del Gateway

Logbook registra estos métodos RPC del Gateway:

| Método                | Parámetros               | Ámbito            | Resultado                                                                   |
| --------------------- | ------------------------ | ---------------- | ------------------------------------------------------------------------ |
| `logbook.status`      | ninguno                     | `operator.read`  | Estado de captura, análisis, modelo, Node, día del Gateway y zona horaria del Gateway |
| `logbook.days`        | ninguno                     | `operator.read`  | Días con recuentos de tarjetas de cronología y límites temporales de las tarjetas                      |
| `logbook.timeline`    | `{ day?: "YYYY-MM-DD" }` | `operator.read`  | Tarjetas derivadas y estadísticas del día; utiliza de forma predeterminada el día actual del Gateway  |
| `logbook.frames`      | `{ startMs, endMs }`     | `operator.write` | Metadatos de fotogramas en el intervalo solicitado de milisegundos desde la época                  |
| `logbook.frame`       | `{ frameId }`            | `operator.write` | Un fotograma JPEG sin procesar como base64                                             |
| `logbook.standup`     | `{ day?, refresh? }`     | `operator.write` | Texto de la reunión diaria almacenado en caché o regenerado para un día                             |
| `logbook.ask`         | `{ day?, question }`     | `operator.write` | Respuesta basada en la cronología para un día                                       |
| `logbook.capture.set` | `{ paused }`             | `operator.write` | Estado de pausa solo para la sesión y estado actualizado                              |
| `logbook.analyze.now` | ninguno                     | `operator.write` | Inicia el análisis pendiente o devuelve el motivo por el que no se pudo iniciar          |

Los métodos de lectura devuelven el estado operativo o texto derivado. Los píxeles de las
capturas de pantalla sin procesar, las acciones que incurren en gastos del modelo y las mutaciones en tiempo de ejecución requieren
`operator.write`. La pestaña de la interfaz de control también requiere `operator.write` porque
expone esas acciones y vistas previas de los fotogramas sin procesar; un cliente de solo lectura aún puede invocar
directamente los métodos de texto derivado.

## Notas sobre privacidad

- Las capturas pueden contener cualquier elemento visible en pantalla, incluidos secretos. Los fotogramas nunca
  salen del equipo, excepto como entrada muestreada para el modelo de observación
  configurado.
- Las observaciones, las tarjetas recientes y las preguntas pueden salir del equipo a través del
  modelo de agente predeterminado durante la síntesis de tarjetas, la generación de reuniones diarias o las preguntas y respuestas. Aplique
  la política de tratamiento de datos del proveedor a ambas rutas de modelos.
- Utilice rutas locales tanto para el modelo de observación estructurada como para el modelo de agente
  predeterminado cuando necesite un pipeline completamente local.
- Los fotogramas, la base de datos de la línea temporal y las capturas temporales se escriben con
  permisos de archivo exclusivos para el propietario.
- Añadir `screen.snapshot` a `gateway.nodes.commands.deny` es el
  interruptor de desactivación de la captura de pantalla: bloquea tanto la captura del nodo de la aplicación como el
  propio comando `logbook.snapshot` de Logbook.
- Configurar `tools.media.image.enabled: false` también impide que Logbook tome prestados
  los modelos de imágenes multimedia para el análisis; en ese caso, solo se utiliza un `visionModel` explícito en la
  configuración del plugin.

## Solución de problemas

### Falta la pestaña Logbook

Compruebe las tres condiciones:

1. `openclaw plugins list --enabled` incluye `logbook`.
2. El Gateway se reinició después del cambio en el plugin o en la lista de permitidos.
3. La conexión de la interfaz de control tiene `operator.write`; las sesiones de solo lectura no
   reciben el descriptor de la pestaña interactiva.

Si se establece `plugins.allow`, debe incluir tanto `logbook` como `codex` para la
configuración recomendada.

### La captura informa de un error

```bash
openclaw nodes status --connected
openclaw nodes describe --node <idOrNameOrIp>
openclaw logs --follow
```

- Confirme que el nodo exponga `screen.snapshot` o `logbook.snapshot`.
- Conceda el permiso Screen Recording en el Mac de captura.
- Si se configura `nodeId`, confirme que coincida con el identificador o el nombre para mostrar del nodo.
- Compruebe que `gateway.nodes.commands.deny` no contenga
  `screen.snapshot`.

Tras tres errores consecutivos, Logbook espera durante diez ciclos de captura y
vuelve a intentarlo. Una configuración sin fijar puede cambiar a otro nodo apto.

### Las capturas se realizan correctamente, pero no aparecen tarjetas

- El estado **Falta el modelo** significa que no se encontró ninguna ruta de visión estructurada
  compatible. Habilite y autentique el plugin Codex, o establezca un
  `visionModel` explícito válido. Los fotogramas capturados permanecen pendientes mientras falta el modelo y
  pueden analizarse después de corregir la configuración.
- Espere a `analysisIntervalMinutes` o seleccione **Analizar ahora** después de que
  se haya capturado actividad.
- Los fotogramas idénticos consecutivos constituyen evidencia de inactividad y no entran en los lotes
  de análisis. Cambie la pantalla visible antes de realizar la prueba.
- Si el lote más reciente muestra un error, corrija el problema del modelo o de autenticación y seleccione
  **Analizar ahora**. Los lotes fallidos solo se vuelven a intentar mediante esa acción explícita para
  evitar gastos reiterados del modelo.

## Contenido relacionado

- [Gestionar plugins](/es/plugins/manage-plugins)
- [Arnés de Codex](/es/plugins/codex-harness)
- [Comprensión multimedia](/es/nodes/media-understanding)
- [Nodos](/es/nodes)
- [Solución de problemas de nodos](/es/nodes/troubleshooting)
- [Interfaz de control](/es/web/control-ui)
