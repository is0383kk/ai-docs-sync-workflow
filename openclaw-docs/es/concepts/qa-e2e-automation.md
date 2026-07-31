---
doc-schema-version: 1
read_when:
    - Comprender cómo encaja el stack de QA en su conjunto
    - Ampliación de qa-lab, qa-channel o un adaptador de transporte
    - Adición de escenarios de control de calidad respaldados por el repositorio
    - Creación de automatización de control de calidad de mayor realismo en torno al panel del Gateway
summary: 'Descripción general de la pila de control de calidad: qa-lab, qa-channel, escenarios respaldados por el repositorio, canales de transporte en vivo, adaptadores de transporte y generación de informes.'
title: Descripción general del control de calidad
x-i18n:
    generated_at: "2026-07-26T05:08:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 91c34a50e6197195d57228d92b19caff1785ceaa5d82d7c88a1ec0ed76abd635
    source_path: concepts/qa-e2e-automation.md
    workflow: 16
---

La pila privada de control de calidad ejercita OpenClaw de una forma realista, adaptada a los canales, que
una prueba unitaria no puede reproducir.

Componentes:

- `extensions/qa-channel`: canal de mensajes sintético con superficies de mensajes directos, canales, hilos,
  reacciones, ediciones y eliminaciones.
- `extensions/qa-lab`: interfaz de depuración, bus de control de calidad, perfiles de escenarios y adaptadores de
  transporte en vivo para observar la transcripción, inyectar mensajes entrantes
  y exportar un informe de Markdown.
- `qa/`: recursos de inicialización respaldados por el repositorio para la tarea de inicio y los escenarios
  de control de calidad de referencia.
- [Mantis](/es/concepts/mantis): verificación en vivo del antes y el después para errores que
  requieren transportes reales, capturas de pantalla del navegador, estado de la máquina virtual y pruebas del PR.

## Superficie de comandos

Cada flujo de control de calidad se ejecuta bajo `pnpm openclaw qa <subcommand>`. Muchos tienen alias de
script `pnpm qa:*`; ambas formas funcionan.

| Comando                                             | Propósito                                                                                                                                                                                                                                                             |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `qa run`                                            | Autocomprobación de control de calidad incluida sin `--qa-profile`; ejecutor de perfiles de madurez respaldados por la taxonomía con `--qa-profile smoke-ci`, `--qa-profile release` o `--qa-profile all`.                                                                                                  |
| `qa suite`                                          | Ejecuta escenarios respaldados por el repositorio en el carril del Gateway de control de calidad. `--runner multipass` utiliza una máquina virtual Linux desechable en lugar del host.                                                                                                                                         |
| `qa coverage`                                       | Muestra el inventario YAML de cobertura de escenarios (`--json` para salida procesable por máquinas; `--match <query>` para buscar escenarios de un comportamiento modificado; `--tools` para la cobertura de accesorios de herramientas en tiempo de ejecución).                                                                                  |
| `qa parity-report`                                  | Compara dos archivos `qa-suite-summary.json` para una puerta de paridad del eje de modelos, o utiliza `--runtime-axis --token-efficiency` para escribir informes de paridad en tiempo de ejecución y eficiencia de tokens entre Codex y OpenClaw.                                                                          |
| `qa confidence-report`                              | Clasifica los artefactos de prueba de control de calidad con respecto a un manifiesto en un informe de confianza sin elementos desconocidos.                                                                                                                                                                               |
| `qa confidence-self-test`                           | Escribe canarios de control negativo inicializados que demuestran que la puerta de confianza detecta desviaciones.                                                                                                                                                                                   |
| `qa jsonl-replay`                                   | Reproduce transcripciones JSONL seleccionadas mediante el arnés de reproducción de paridad en tiempo de ejecución.                                                                                                                                                                                         |
| `qa character-eval`                                 | Ejecuta el escenario de control de calidad de personajes en varios modelos en vivo con un informe evaluado. Consulta [Informes](#reporting).                                                                                                                                                        |
| `qa manual`                                         | Ejecuta un prompt único en el carril del proveedor/modelo seleccionado.                                                                                                                                                                                                      |
| `qa ui`                                             | Inicia la interfaz de depuración de control de calidad y el bus local de control de calidad (alias: `pnpm qa:lab:ui`).                                                                                                                                                                                                |
| `qa docker-build-image`                             | Compila la imagen Docker de control de calidad preconfigurada.                                                                                                                                                                                                                                 |
| `qa docker-scaffold`                                | Escribe un esqueleto de docker-compose para el panel de control de calidad y el carril del Gateway.                                                                                                                                                                                                |
| `qa up`                                             | Compila el sitio de control de calidad, inicia la pila respaldada por Docker y muestra la URL (alias: `pnpm qa:lab:up`; la variante `:fast` añade `--use-prebuilt-image --bind-ui-dist --skip-ui-build`).                                                                                              |
| `qa aimock`                                         | Inicia únicamente el servidor del proveedor AIMock.                                                                                                                                                                                                                              |
| `qa mock-openai`                                    | Inicia únicamente el servidor del proveedor `mock-openai` con reconocimiento de escenarios.                                                                                                                                                                                                        |
| `qa credentials doctor` / `add` / `list` / `remove` | Gestiona el grupo compartido de credenciales de Convex.                                                                                                                                                                                                                           |
| `qa discord`                                        | Carril de transporte en vivo con un canal real de un servidor privado de Discord.                                                                                                                                                                                                   |
| `qa matrix`                                         | Perfiles Matrix de QA Lab con un servidor doméstico Tuwunel desechable. Consulta [Carriles de pruebas de humo de Matrix](#matrix-smoke-lanes).                                                                                                                                                      |
| `qa slack`                                          | Carril de transporte en vivo con un canal privado real de Slack.                                                                                                                                                                                                           |
| `qa telegram`                                       | Carril de transporte en vivo con un grupo privado real de Telegram.                                                                                                                                                                                                          |
| `qa whatsapp`                                       | Carril de transporte en vivo con cuentas reales de WhatsApp Web.                                                                                                                                                                                                             |
| `qa mantis`                                         | Ejecutor de verificación del antes y el después para errores de transporte en vivo, con pruebas de reacciones de estado de Discord, pruebas de humo de escritorio/navegador de Crabbox y pruebas de humo de Slack en VNC. Consulta [Mantis](/es/concepts/mantis) y la [Guía operativa de Mantis para Slack Desktop](/es/concepts/mantis-slack-desktop-runbook). |

### `qa run` respaldado por perfiles

`qa run` respaldado por perfiles lee la pertenencia de `taxonomy.yaml` y, a continuación, despacha
los escenarios resueltos mediante `qa suite`. `--surface` y `--category` filtran
el perfil seleccionado en lugar de definir carriles independientes. El
`qa-evidence.json` resultante incluye un resumen de la tabla de puntuación del perfil con los recuentos
de categorías seleccionadas y los ID de cobertura faltantes; las entradas de pruebas individuales siguen siendo la
fuente de verdad para las pruebas, las funciones de cobertura y los resultados. Los ID de cobertura
de características de la taxonomía son objetivos de prueba exactos, no alias: la cobertura principal de escenarios
satisface los ID coincidentes, mientras que la cobertura secundaria sigue siendo orientativa. Cada ID de cobertura
es exactamente `taxonomy-surface.feature`, utilizando el ID corto de la superficie de
`taxonomy.yaml`. El campo `surface` independiente de un escenario es una etiqueta de ejecución/informes
(por ejemplo, `channel` o `runtime-tool`); no define la propiedad
de la taxonomía.

Las pruebas reducidas omiten el `execution` de cada entrada y establecen `evidenceMode: "slim"`;
`smoke-ci` utiliza el modo reducido de forma predeterminada y `--evidence-mode full` restaura las entradas completas:

```bash
pnpm openclaw qa run \
  --qa-profile smoke-ci \
  --category channels.conversation-routing-and-delivery \
  --provider-mode mock-openai \
  --output-dir .artifacts/qa-e2e/smoke-ci-profile-dispatch
```

Utiliza `smoke-ci` para obtener pruebas deterministas de perfiles con proveedores de modelos simulados y
servidores de proveedores locales de Crabline. Utiliza `release` para las pruebas de Stable/LTS con
canales en vivo. Utiliza `all` únicamente para ejecuciones explícitas de pruebas de la taxonomía completa; este
selecciona todas las categorías de madurez activas y se puede despachar mediante el flujo de trabajo
`QA
Profile Evidence` de GitHub Actions con `qa_profile=all`. Cuando un
comando también necesite un perfil raíz de OpenClaw, coloca el perfil raíz antes del
comando de control de calidad:

```bash
pnpm openclaw --profile work qa run --qa-profile smoke-ci
```

## Flujo del operador

El flujo actual del operador de control de calidad es un sitio de control de calidad con dos paneles:

- Izquierda: panel del Gateway (interfaz de control) con el agente.
- Derecha: QA Lab, que muestra la transcripción similar a Slack y el plan del escenario.

Ejecútalo con:

```bash
pnpm qa:lab:up
```

Esto compila el sitio de control de calidad, inicia el carril del Gateway respaldado por Docker y expone
la página de QA Lab, donde un operador o un bucle de automatización puede asignar al agente una
misión de control de calidad, observar el comportamiento real del canal y registrar qué funcionó, qué falló o qué
permaneció bloqueado.

Para iterar más rápidamente en la interfaz de QA Lab sin volver a compilar la imagen Docker cada vez,
inicia la pila con un paquete de QA Lab montado mediante enlace:

```bash
pnpm openclaw qa docker-build-image
pnpm qa:lab:build
pnpm qa:lab:up:fast
pnpm qa:lab:watch
```

`qa:lab:up:fast` mantiene los servicios de Docker en una imagen precompilada y
monta mediante enlace `extensions/qa-lab/web/dist` en el contenedor `qa-lab`.
`qa:lab:watch` recompila ese paquete cuando hay cambios y el navegador se recarga automáticamente
cuando cambia el hash del recurso de QA Lab.

### Pruebas de humo de observabilidad

<Note>
El control de calidad de observabilidad sigue estando disponible únicamente en el checkout del código fuente. El paquete tar de npm omite
intencionadamente QA Lab (y `qa-channel`), por lo que los carriles de lanzamiento de Docker
del paquete no ejecutan comandos `qa`. Ejecútalos desde un checkout del código fuente compilado cuando
se modifique la instrumentación de diagnóstico.
</Note>

| Alias                                   | Qué ejecuta                                                                                                                            |
| --------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `pnpm qa:otel:smoke`                    | Receptor local de OpenTelemetry más el escenario `otel-trace-smoke` con `diagnostics-otel` habilitado.                                      |
| `pnpm qa:otel:collector-smoke`          | La misma vía detrás de un contenedor Docker real de OpenTelemetry Collector. Úsela al cambiar la conexión de endpoints o la compatibilidad con el recopilador/OTLP. |
| `pnpm qa:prometheus:smoke`              | El escenario `docker-prometheus-smoke` con `diagnostics-prometheus` habilitado.                                                           |
| `pnpm qa:observability:smoke`           | `qa:otel:smoke` seguido de `qa:prometheus:smoke`.                                                                                      |
| `pnpm qa:observability:collector-smoke` | `qa:otel:collector-smoke` seguido de `qa:prometheus:smoke`.                                                                            |

`qa:otel:smoke` inicia un receptor OTLP/HTTP local, ejecuta un turno mínimo
del agente del canal de QA y, a continuación, comprueba que se exporten trazas,
métricas y registros. Decodifica los intervalos de traza protobuf exportados y
comprueba la estructura crítica para la versión:
`openclaw.run`, `openclaw.harness.run`, un intervalo de llamada al modelo con la
convención semántica GenAI más reciente, `openclaw.context.assembled` y `openclaw.message.delivery`
deben estar presentes. La prueba de humo fuerza
`OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental`, por lo que el intervalo de llamada
al modelo debe usar el nombre `{gen_ai.operation.name} {gen_ai.request.model}`; las llamadas
al modelo no deben exportar `StreamAbandoned` en los turnos correctos; los
identificadores de diagnóstico sin procesar y los atributos `openclaw.content.*`
deben mantenerse fuera de la traza. El prompt del escenario solicita al modelo
que responda con un marcador fijo y que no revele una cadena secreta fija; las
cargas OTLP sin procesar no deben contener ninguno de ellos ni la clave de sesión
de QA derivada del identificador del escenario. Escribe `otel-smoke-summary.json`
junto a los artefactos del conjunto de QA.

`qa:prometheus:smoke` verifica que se rechacen las extracciones no autenticadas y,
a continuación, comprueba que la extracción autenticada incluya las familias
de métricas críticas para la versión sin contenido del prompt, contenido de la
respuesta, identificadores de diagnóstico sin procesar, tokens de autenticación
ni rutas locales.

### Vías de prueba de humo de Matrix

Para una vía de prueba de humo de Matrix con transporte real que no requiera
credenciales del proveedor de modelos, ejecute el perfil de versión con el
proveedor OpenAI simulado determinista:

```bash
pnpm openclaw qa matrix --provider-mode mock-openai --profile release
```

Para la vía del proveedor avanzado en vivo, proporcione explícitamente
credenciales compatibles con OpenAI:

```bash
OPENCLAW_LIVE_OPENAI_KEY="${OPENAI_API_KEY}" \
  pnpm openclaw qa matrix --provider-mode live-frontier --profile release
```

El comando simple `pnpm openclaw qa matrix` ejecuta el perfil `all`
completo y continúa tras los fallos de los escenarios. Use
`--fail-fast` para un ciclo de retroalimentación más corto o repita
`--scenario <id>` para seleccionar escenarios individuales; los identificadores
explícitos de escenarios tienen prioridad sobre `--profile`.

| Perfil       | Escenarios | Finalidad                                                                                                                                |
| ------------ | ---------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `all`        | 93        | Catálogo completo (predeterminado).                                                                                                      |
| `release`    | 2         | Referencia crítica para la versión del canal y recarga en vivo de la lista de permitidos.                                                |
| `fast`       | 12        | Cobertura específica de hilos, reacciones, aprobaciones, políticas, control de bots y respuestas cifradas.                              |
| `transport`  | 50        | Hilos, enrutamiento de mensajes directos/salas, unión automática, aprobaciones, reacciones, reinicios, política de menciones/listas de permitidos, ediciones y ordenación de varios actores. |
| `media`      | 7         | Cobertura de imágenes, imágenes generadas, voz, archivos adjuntos, medios no compatibles y medios cifrados.                             |
| `e2ee-smoke` | 8         | Cobertura mínima de respuestas cifradas, hilos, arranque, recuperación, reinicio, censura y fallos.                                      |
| `e2ee-deep`  | 18        | Pérdida de estado, copias de seguridad, recuperación de claves, higiene de dispositivos y verificación SAS/QR/mensajes directos.         |
| `e2ee-cli`   | 9         | Comandos de `openclaw matrix encryption setup`, clave de recuperación, varias cuentas, ida y vuelta del Gateway y autoverificación mediante el arnés. |

La pertenencia a los perfiles y los requisitos del canal residen junto a los
escenarios declarativos de Matrix en `qa/scenarios/channels/`. La ejecución elige el
controlador del canal. Sus implementaciones en vivo residen en
`extensions/qa-lab/src/live-transports/matrix/scenarios/`.

El adaptador aprovisiona un servidor doméstico Tuwunel desechable en Docker
(imagen predeterminada `ghcr.io/matrix-construct/tuwunel:v1.5.1`, nombre del servidor
`matrix-qa.test`, puerto `28008`), registra usuarios temporales
para el controlador, el sistema bajo prueba y el observador, prepara las salas
necesarias y registra el límite de solicitudes/respuestas censurado. A
continuación, ejecuta el Plugin real de Matrix dentro de un Gateway de QA
secundario limitado a ese transporte (sin `qa-channel`) y desmonta el
entorno.

Opciones habituales:

| Opción                   | Valor predeterminado | Finalidad                                                                             |
| ------------------------ | -------------------- | ------------------------------------------------------------------------------------ |
| `--profile <profile>`    | `all`             | Seleccionar uno de los perfiles anteriores.                                          |
| `--scenario <id>`        | -                 | Seleccionar un escenario; se puede repetir.                                          |
| `--fail-fast`            | desactivado       | Detenerse después de la primera comprobación o escenario fallido.                    |
| `--allow-failures`       | desactivado       | Escribir artefactos sin devolver un código de salida de error por fallos de escenarios. |
| `--provider-mode <mode>` | `live-frontier`   | Usar `mock-openai` para el envío determinista o `live-frontier` para un proveedor en vivo. |
| `--model <ref>`          | valor del proveedor | Establecer la referencia principal de `provider/model`.                            |
| `--alt-model <ref>`      | valor del proveedor | Establecer el modelo alternativo que usan los escenarios que cambian de modelo.      |
| `--fast`                 | desactivado       | Habilitar el modo rápido del proveedor cuando sea compatible.                        |
| `--output-dir <path>`    | generado           | Elegir el directorio de informes; las rutas relativas se resuelven respecto a `--repo-root`. |
| `--repo-root <path>`     | directorio actual | Ejecutar desde un directorio de trabajo neutro.                                      |
| `--sut-account <id>`     | `sut`             | Seleccionar el identificador de cuenta de Matrix en la configuración del Gateway secundario. |

QA de Matrix no arrienda credenciales compartidas de Matrix: el adaptador crea
usuarios desechables localmente, por lo que no acepta `--credential-source` ni
`--credential-role`. Reemplace la imagen del servidor doméstico con
`OPENCLAW_QA_MATRIX_TUWUNEL_IMAGE`; ajuste las comprobaciones negativas de ausencia de respuesta
con `OPENCLAW_QA_MATRIX_NO_REPLY_WINDOW_MS` (valor predeterminado `8000`, limitado al
tiempo de espera del escenario activo). El comando de ejecución única suele
forzar una salida limpia después de vaciar los artefactos porque los
identificadores nativos criptográficos de Matrix pueden sobrevivir a la
limpieza; establezca `OPENCLAW_QA_MATRIX_DISABLE_FORCE_EXIT=1` solo para un arnés de pruebas directo
que necesite que el comando devuelva el control.

Cada ejecución escribe los artefactos habituales de QA Lab en el directorio de
salida seleccionado: `qa-suite-report.md`, `qa-suite-summary.json` y
`qa-evidence.json`. Si la limpieza falla, ejecute el comando de recuperación
`docker compose ... down --remove-orphans` mostrado. En ejecutores lentos, aumente el intervalo sin
respuesta; en una Pipeline de CI rápida, un intervalo menor puede acortar las
comprobaciones negativas.

Los escenarios abarcan comportamientos de transporte que las pruebas unitarias
no pueden demostrar de extremo a extremo: control de menciones, políticas de
permisos para bots, listas de permitidos, respuestas de nivel superior y en
hilos, enrutamiento de mensajes directos, gestión de reacciones, supresión de
ediciones entrantes, deduplicación de repeticiones tras reinicios, recuperación
de interrupciones del servidor doméstico, entrega de metadatos de aprobación,
gestión de medios y flujos de arranque, recuperación y verificación de E2EE de
Matrix. El perfil de CLI de E2EE también ejecuta `openclaw matrix encryption setup` y comandos
de verificación mediante el mismo servidor doméstico desechable antes de
comprobar las respuestas del Gateway.

`matrix-room-block-streaming` y `subagent-thread-spawn` siguen disponibles mediante la selección
explícita de `--scenario`, pero permanecen fuera del perfil predeterminado
`all`.

La Pipeline de CI usa la misma superficie de comandos en
`.github/workflows/qa-live-transports-convex.yml`. Las ejecuciones programadas y de versión ejecutan los
escenarios de versión. Los envíos manuales de `matrix_profile=all` distribuyen
los perfiles `transport`, `media`, `e2ee-smoke`,
`e2ee-deep` y `e2ee-cli`; los envíos específicos seleccionan
`fast`, `release` o `transport` en un trabajo.

### Escenarios Mantis de Discord

Discord también tiene escenarios opcionales exclusivos de Mantis para
reproducir errores. Use `--scenario discord-status-reactions-tool-only` para la cronología explícita de
reacciones de estado o `--scenario discord-thread-reply-filepath-attachment` para crear un hilo real de Discord y
verificar que `message.thread-reply` conserve un archivo adjunto
`filePath`. Estos escenarios quedan fuera de la vía predeterminada de
Discord en vivo porque son sondas de reproducción antes/después, en lugar de
una cobertura amplia de pruebas de humo. El flujo de trabajo Mantis de archivos
adjuntos en hilos también puede añadir un vídeo testigo de Discord Web con una
sesión iniciada cuando `MANTIS_DISCORD_VIEWER_CHROME_PROFILE_DIR` o `MANTIS_DISCORD_VIEWER_CHROME_PROFILE_TGZ_B64` están
configurados en el entorno de QA. Ese perfil de visualización solo sirve para
la captura visual; la decisión de aprobación o fallo sigue procediendo del
oráculo REST de Discord.

Para las demás vías de prueba de humo con transporte real:

```bash
pnpm openclaw qa discord
pnpm openclaw qa slack
pnpm openclaw qa telegram
pnpm openclaw qa whatsapp
```

Se dirigen a un canal real preexistente con dos bots o cuentas (controlador +
sistema bajo prueba). Las variables de entorno necesarias, las listas de
escenarios, los artefactos de salida y el conjunto de credenciales de Convex
para esos cuatro transportes se documentan en la
[referencia de QA de Discord, Slack, Telegram y WhatsApp](#discord-slack-telegram-and-whatsapp-qa-reference)
que aparece más adelante.

### Ejecutores de escritorio de Slack y tareas visuales de Mantis

Para una ejecución completa en una máquina virtual de escritorio de Slack con
rescate mediante VNC, ejecute:

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --gateway-setup \
  --scenario slack-canary \
  --keep-lease
```

Ese comando reserva una máquina Crabbox de escritorio/navegador, ejecuta el
carril en vivo de Slack dentro de la máquina virtual, abre Slack Web en el navegador VNC, captura el escritorio
y copia `slack-qa/`, `slack-desktop-smoke.png` y
`slack-desktop-smoke.mp4` (cuando la captura de vídeo está disponible) de vuelta al
directorio de artefactos de Mantis. Las reservas de escritorio/navegador de Crabbox proporcionan de antemano las
herramientas de captura y los paquetes auxiliares de navegador/compilación nativa, por lo que el escenario
solo debería instalar alternativas en reservas antiguas. Mantis informa de los tiempos totales y
por fase en `mantis-slack-desktop-smoke-report.md` para que las ejecuciones lentas muestren
si el tiempo se dedicó a preparar la reserva, adquirir credenciales, configurar el entorno remoto o
copiar artefactos. Reutilice `--lease-id <cbx_...>` después de iniciar sesión manualmente
en Slack Web mediante VNC; las reservas reutilizadas también mantienen activa la caché del almacén pnpm
de Crabbox. El valor predeterminado `--hydrate-mode source` realiza la verificación desde un checkout del código fuente y
ejecuta la instalación/compilación dentro de la máquina virtual. Use `--hydrate-mode prehydrated` solo cuando
el espacio de trabajo remoto reutilizado ya tenga `node_modules` y un `dist/` compilado;
ese modo omite el costoso paso de instalación/compilación y falla de forma segura cuando el
espacio de trabajo no está listo. Con `--gateway-setup`, Mantis mantiene en ejecución un
Gateway de Slack de OpenClaw persistente dentro de la máquina virtual en el puerto `38973`; sin esta opción, el
comando ejecuta el carril normal de QA de Slack de bot a bot y termina después de capturar los
artefactos.

Para demostrar la interfaz de aprobación nativa de Slack con pruebas del escritorio, ejecute el modo de
puntos de control de aprobación de Mantis:

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --approval-checkpoints \
  --credential-source convex \
  --credential-role maintainer
```

Este modo es mutuamente excluyente con `--gateway-setup`. Ejecuta los escenarios de
aprobación de Slack, rechaza los identificadores de escenarios que no sean de aprobación, espera en cada estado de
aprobación pendiente y resuelto, renderiza el mensaje observado de la API de Slack en
`approval-checkpoints/<scenario>-pending.png` y
`approval-checkpoints/<scenario>-resolved.png` y, a continuación, falla si algún punto de control,
prueba de mensaje, confirmación o captura de pantalla renderizada falta o
está vacío. Las reservas de CI en frío todavía pueden mostrar el inicio de sesión de Slack en
`slack-desktop-smoke.png`; las imágenes de los puntos de control de aprobación son la prueba
visual de este carril.

La ejecución predeterminada de puntos de control conserva los dos escenarios estándar de aprobación de Slack.
Para capturar cualquiera de las rutas de aprobación opcionales de Codex, selecciónela explícitamente con
`--scenario slack-codex-approval-exec-native` o
`--scenario slack-codex-approval-plugin-native`; Mantis acepta ambas y genera
el mismo par de capturas de pantalla pendiente/resuelta. El ejecutor amplía los plazos de sus puntos de control
y comandos remotos para cada ruta de Codex seleccionada, de modo que pueda finalizar la secuencia completa de
aprobación, finalización del agente y actualización resuelta.

La lista de comprobación del operador, el comando de ejecución del flujo de trabajo de GitHub, el contrato de comentarios
de pruebas, la tabla de decisiones del modo de hidratación, la interpretación de tiempos y los pasos para
gestionar fallos se encuentran en
[Manual de ejecución de escritorio de Slack con Mantis](/es/concepts/mantis-slack-desktop-runbook).

Para una tarea de escritorio de estilo agente/visión artificial, ejecute:

```bash
pnpm openclaw qa mantis visual-task \
  --browser-url https://example.net \
  --expect-text "Example Domain" \
  --vision-model openai/gpt-5.6-luna
```

`visual-task` reserva o reutiliza una máquina Crabbox de escritorio/navegador, inicia
`crabbox record --while`, controla el navegador visible mediante un
`visual-driver` anidado, captura `visual-task.png`, ejecuta `openclaw infer image
describe` con la captura de pantalla cuando se
selecciona `--vision-mode image-describe` y escribe `visual-task.mp4`, `mantis-visual-task-summary.json`,
`mantis-visual-task-driver-result.json` y
`mantis-visual-task-report.md`. Cuando se establece `--expect-text`, el prompt de visión
solicita un veredicto JSON estructurado (`visible`, `evidence`, `reason`)
y solo se supera cuando el modelo informa de `visible: true` con pruebas que
citan el texto esperado; una respuesta `visible: false` que se limite a citar el
texto objetivo sigue sin superar la aserción. Use `--vision-mode metadata` para una
prueba de humo sin modelo que demuestre el funcionamiento del escritorio, el navegador, las capturas de pantalla y el vídeo
sin llamar a un proveedor de comprensión de imágenes. La grabación es un
artefacto obligatorio para `visual-task`; si Crabbox no graba ningún
`visual-task.mp4` que no esté vacío, la tarea falla aunque el controlador visual haya superado la prueba. En caso de
fallo, Mantis conserva la reserva para VNC, salvo que la tarea ya se hubiera superado
y no se hubiera establecido `--keep-lease`.

### Comprobación del estado del conjunto de credenciales

Antes de usar credenciales en vivo compartidas, ejecute:

```bash
pnpm openclaw qa credentials doctor
```

El diagnóstico comprueba el entorno del intermediario de Convex (`OPENCLAW_QA_CONVEX_SITE_URL`,
`OPENCLAW_QA_CONVEX_ENDPOINT_PREFIX`), valida la configuración de los endpoints, informa
solo del estado establecido/ausente de `OPENCLAW_QA_CONVEX_SECRET_CI` y
`OPENCLAW_QA_CONVEX_SECRET_MAINTAINER` y verifica la accesibilidad de administración/listado
cuando está presente el secreto del mantenedor.

## Cobertura canónica de escenarios

El archivo raíz `taxonomy.yaml` define identificadores de cobertura semántica. Los archivos YAML de escenarios
en `qa/scenarios/` asignan cada escenario a esos identificadores y controlan los metadatos
de ejecución: `channel` es el único requisito de canal y `profiles` declara
la pertenencia a ejecuciones con nombre. El controlador de canal es una opción intercambiable de implementación
en el nivel de ejecución. Los ejecutores de TypeScript
consultan ese catálogo; no mantienen inventarios paralelos de escenarios ni de cobertura.

La salida estática de `qa coverage` informa de la correspondencia entre la taxonomía y los escenarios. Las
pruebas reales proceden de `qa-evidence.json`, que registra el escenario ejecutado,
los identificadores de cobertura, el canal, el controlador realmente utilizado y el resultado. El canal y el controlador son
dimensiones de los informes, no vocabularios adicionales de identificadores de cobertura ni ejes de
elegibilidad de escenarios.

Para un carril de máquina virtual Linux desechable sin incorporar Docker a la ruta de QA, ejecute:

```bash
pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline
```

Esto inicia un invitado Multipass nuevo, instala las dependencias, compila OpenClaw
dentro del invitado, ejecuta `qa suite` y, a continuación, copia el informe y el
resumen normales de QA de vuelta a `.artifacts/qa-e2e/...` en el host. Reutiliza el mismo
comportamiento de selección de escenarios que `qa suite` en el host.

Las ejecuciones de la suite en el host y en Multipass ejecutan varios escenarios seleccionados en
paralelo y, de forma predeterminada, utilizan procesos de trabajo del Gateway aislados. `qa-channel` tiene de forma predeterminada
una concurrencia de 4, limitada por el número de escenarios seleccionados. Use `--concurrency
<count>` para ajustar el número de procesos de trabajo o `--concurrency 1` para una ejecución en serie.
Use `--pack personal-agent` para ejecutar el paquete de pruebas comparativas del asistente personal (10
escenarios). El selector de paquetes es acumulativo con los indicadores `--scenario` repetidos:
primero se ejecutan los escenarios explícitos y, después, los escenarios del paquete se ejecutan en el orden del paquete,
eliminando los duplicados. Use `--pack observability` para seleccionar conjuntamente los escenarios
`otel-trace-smoke` y `docker-prometheus-smoke` cuando un
ejecutor de QA personalizado ya proporcione la configuración del recopilador de OpenTelemetry.

El comando termina con un código distinto de cero cuando falla algún escenario. Use `--allow-failures`
cuando quiera generar artefactos sin un código de salida de error.

Las ejecuciones en vivo reenvían las entradas de autenticación de QA compatibles que resultan prácticas para el
invitado: claves del proveedor basadas en variables de entorno, la ruta de configuración del proveedor en vivo de QA y
`CODEX_HOME` cuando está presente. Mantenga `--output-dir` bajo la raíz del repositorio para que el
invitado pueda volver a escribir mediante el espacio de trabajo montado.

## Referencia de QA de Discord, Slack, Telegram y WhatsApp

El adaptador de Matrix utiliza el carril desechable basado en Docker documentado anteriormente.
Discord, Slack, Telegram y WhatsApp se ejecutan con transportes reales
preexistentes, por lo que su referencia se encuentra aquí.

### Indicadores compartidos de la CLI

Estos carriles se registran mediante
`extensions/qa-lab/src/live-transports/shared/live-transport-cli.ts` y
aceptan los mismos indicadores:

| Indicador                             | Valor predeterminado                                | Descripción                                                                                                                                     |
| ------------------------------------- | -------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `--scenario <id>`                     | -                                                  | Ejecuta solo este escenario. Se puede repetir.                                                                                                  |
| `--output-dir <path>`                 | `<repo>/.artifacts/qa-e2e/<transport>-<timestamp>` | Lugar donde se escriben los informes, resúmenes, pruebas, artefactos específicos del transporte y el registro de salida. Las rutas relativas se resuelven respecto de `--repo-root`. |
| `--repo-root <path>`                  | `process.cwd()`                                    | Raíz del repositorio cuando se invoca desde un directorio de trabajo neutral.                                                                   |
| `--sut-account <id>`                  | `sut`                                              | Identificador de cuenta temporal en la configuración del Gateway de QA.                                                                         |
| `--provider-mode <mode>`              | `live-frontier`                                    | `mock-openai`, `aimock` o `live-frontier`.                                                                                                    |
| `--model <ref>` / `--alt-model <ref>` | valor predeterminado del proveedor                  | Referencias de los modelos principal/alternativo.                                                                                               |
| `--fast`                              | desactivado                                         | Modo rápido del proveedor cuando sea compatible.                                                                                                |
| `--credential-source <env\|convex>`   | `env`                                              | Consulte [Conjunto de credenciales de Convex](#convex-credential-pool).                                                                         |
| `--credential-role <maintainer\|ci>`  | `ci` en CI, `maintainer` en caso contrario                 | Rol utilizado cuando `--credential-source convex`.                                                                                                    |
| `--allow-failures`                    | desactivado                                         | Escribe artefactos sin devolver un código de salida de error cuando fallan los escenarios.                                                       |

Cada carril termina con un código distinto de cero si falla algún escenario. `--allow-failures` escribe
los artefactos sin establecer un código de salida de error. Telegram también acepta
`--list-scenarios` para mostrar los identificadores de escenarios disponibles y salir; los demás carriles
no ofrecen ese indicador.

### QA de Telegram

```bash
pnpm openclaw qa telegram
```

Se dirige a un grupo privado real de Telegram con dos bots distintos (controlador +
SUT). El bot SUT debe tener un nombre de usuario de Telegram; la observación de bot a bot funciona
mejor cuando ambos bots tienen **Bot-to-Bot Communication Mode** habilitado en
`@BotFather`.

Entorno obligatorio cuando `--credential-source env`:

- `OPENCLAW_QA_TELEGRAM_GROUP_ID` - identificador numérico del chat (cadena).
- `OPENCLAW_QA_TELEGRAM_DRIVER_BOT_TOKEN`
- `OPENCLAW_QA_TELEGRAM_SUT_BOT_TOKEN`

El perfil `release` selecciona los escenarios YAML mantenidos de Telegram; `all`
añade comprobaciones opcionales de estrés de sesiones, uso, cadenas de respuestas y streaming. Los valores explícitos de
`--scenario` sustituyen el perfil.

- `channel-canary`
- `channel-mention-gating`
- `telegram-help-command`
- `telegram-commands-command`
- `telegram-tools-compact-command`
- `telegram-whoami-command`
- `telegram-status-command`
- `telegram-repeated-command-authorization`
- `telegram-other-bot-command-gating`
- `telegram-context-command`
- `telegram-current-session-status-tool`
- `telegram-tool-only-usage-footer`
- `telegram-reply-chain-exact-marker`
- `telegram-stream-final-single-message`
- `telegram-long-final-reuses-preview`
- `telegram-long-final-three-chunks`

El perfil `release` siempre abarca el canary, el control por mención, las respuestas de comandos nativos, el direccionamiento de comandos y las respuestas de bot a bot en grupos. `mock-openai`
también incluye la comprobación determinista de la vista previa final larga.
`telegram-current-session-status-tool` y
`telegram-tool-only-usage-footer` siguen siendo opcionales: el primero solo es estable
cuando se ejecuta directamente después del canary, y el segundo es una prueba con Telegram real
del pie de página `/usage` en respuestas que solo contienen herramientas. Use `pnpm openclaw qa telegram
--list-scenarios --provider-mode mock-openai` para imprimir la división
actual entre valores predeterminados y opcionales con referencias de regresión. Use `--profile all` para cada
escenario del adaptador en vivo de Telegram.

Artefactos de salida:

- `qa-suite-report.md`
- `qa-suite-summary.json`
- `qa-evidence.json` - entradas de evidencia para las comprobaciones del transporte en vivo,
  incluidos los campos de perfil, cobertura, proveedor, canal, artefactos, resultado y RTT.

Las ejecuciones de paquete de Telegram utilizan el mismo contrato de credenciales de Telegram. La medición
repetida de RTT forma parte del carril en vivo normal de Telegram del paquete; la distribución
de RTT se incorpora en `qa-evidence.json` bajo `result.timing` para la
comprobación de RTT seleccionada.

```bash
OPENCLAW_QA_CREDENTIAL_SOURCE=convex \
pnpm test:docker:npm-telegram-live
```

Cuando se establece `OPENCLAW_QA_CREDENTIAL_SOURCE=convex`, el contenedor de ejecución en vivo del paquete
obtiene en concesión una credencial `kind: "telegram"`, exporta las variables de entorno del grupo, controlador y bot
SUT concedidos a la ejecución del paquete instalado, mantiene activa la concesión mediante Heartbeat y la libera
al cerrarse. El contenedor del paquete utiliza de forma predeterminada 20 comprobaciones de RTT de
`channel-canary`, un tiempo de espera de RTT de 30s y el rol de Convex
`maintainer` fuera de CI cuando se selecciona Convex. Modifique
`OPENCLAW_NPM_TELEGRAM_RTT_SAMPLES`, `OPENCLAW_NPM_TELEGRAM_RTT_TIMEOUT_MS`
o `OPENCLAW_NPM_TELEGRAM_RTT_MAX_FAILURES` para ajustar la medición de RTT sin
crear un comando de RTT independiente ni un formato de resumen específico de Telegram.

### QA de Discord

```bash
pnpm openclaw qa discord
```

Se dirige a un canal privado real de un servidor de Discord con dos bots: un bot controlador
gestionado por el arnés y un bot SUT iniciado por el Gateway secundario de OpenClaw
mediante el Plugin de Discord incluido. Verifica la gestión de menciones del canal, que
el bot SUT haya registrado el comando nativo `/help` en Discord y
los escenarios de evidencia opcionales de Mantis.

Variables de entorno obligatorias cuando `--credential-source env`:

- `OPENCLAW_QA_DISCORD_GUILD_ID`
- `OPENCLAW_QA_DISCORD_CHANNEL_ID`
- `OPENCLAW_QA_DISCORD_DRIVER_BOT_TOKEN`
- `OPENCLAW_QA_DISCORD_SUT_BOT_TOKEN`
- `OPENCLAW_QA_DISCORD_SUT_APPLICATION_ID` - debe coincidir con el id. de usuario del bot SUT
  devuelto por Discord (de lo contrario, el carril falla inmediatamente).

Opcional:

- `OPENCLAW_QA_DISCORD_VOICE_CHANNEL_ID` selecciona el canal de voz/escenario para
  `discord-voice-autojoin`; sin esta variable, el escenario selecciona el primer canal
  de voz/escenario visible para el bot SUT.

Escenarios del módulo YAML de Discord (`qa/scenarios/channels/discord-*.yaml`):

- `discord-canary`
- `discord-mention-gating`
- `discord-native-help-command-registration`
- `discord-voice-autojoin` - escenario de voz opcional. Se ejecuta por sí solo, activa
  `channels.discord.voice.autoJoin` y verifica que el estado de voz actual del bot SUT
  en Discord corresponda al canal de voz/escenario de destino. Las credenciales de Discord de Convex
  pueden incluir el valor opcional `voiceChannelId`; de lo contrario, el adaptador de ejecución
  detecta el primer canal de voz/escenario visible del servidor.
- `discord-status-reactions-tool-only` - escenario de Mantis opcional. Se ejecuta por
  sí solo porque cambia el SUT a respuestas siempre activas y solo de herramientas en el servidor
  mediante `messages.statusReactions.enabled=true`, y después captura una cronología de
  reacciones REST junto con artefactos visuales HTML/PNG. Los informes anterior/posterior de
  Mantis también conservan como `baseline.mp4`
  y `candidate.mp4` los artefactos MP4 proporcionados por el escenario.
- `discord-thread-reply-filepath-attachment` - escenario de Mantis opcional; consulte
  [Escenarios de Mantis en Discord](#discord-mantis-scenarios).

Ejecute explícitamente el escenario de unión automática a voz de Discord:

```bash
pnpm openclaw qa discord \
  --scenario discord-voice-autojoin \
  --provider-mode mock-openai
```

Ejecute explícitamente el escenario de reacciones de estado de Mantis:

```bash
pnpm openclaw qa discord \
  --scenario discord-status-reactions-tool-only \
  --provider-mode live-frontier \
  --model openai/gpt-5.6-luna \
  --alt-model openai/gpt-5.6-luna \
  --fast
```

Artefactos de salida:

- `qa-suite-report.md`
- `qa-suite-summary.json`
- `qa-evidence.json` - entradas de evidencia para las comprobaciones del transporte en vivo.
- `discord-qa-reaction-timelines.json` y
  `discord-status-reactions-tool-only-timeline.png` cuando se ejecuta el escenario
  de reacciones de estado.

### QA de Slack

```bash
pnpm openclaw qa slack
```

Se dirige a un canal privado real de Slack con dos bots distintos: un bot controlador
gestionado por el arnés y un bot SUT iniciado por el Gateway secundario de OpenClaw
mediante el Plugin de Slack incluido.

Variables de entorno obligatorias cuando `--credential-source env`:

- `OPENCLAW_QA_SLACK_CHANNEL_ID`
- `OPENCLAW_QA_SLACK_DRIVER_BOT_TOKEN`
- `OPENCLAW_QA_SLACK_SUT_BOT_TOKEN`
- `OPENCLAW_QA_SLACK_SUT_APP_TOKEN`

Opcional:

- `OPENCLAW_QA_SLACK_APPROVAL_CHECKPOINT_DIR` activa puntos de control de aprobación visual
  para Mantis. El adaptador escribe `<scenario>.pending.json` y
  `<scenario>.resolved.json`, y después espera los archivos `.ack.json` correspondientes.
- `OPENCLAW_QA_SLACK_APPROVAL_CHECKPOINT_TIMEOUT_MS` modifica el tiempo de espera de
  confirmación de los puntos de control. El valor predeterminado es `120000`.

Escenarios YAML canónicos expuestos mediante el adaptador en vivo de Slack:

- `thread-follow-up`
- `thread-isolation`

Escenarios del módulo YAML de Slack (`qa/scenarios/channels/slack-*.yaml`):

- `slack-canary`
- `slack-mention-gating`
- `slack-allowlist-block`
- `slack-channel-disabled-warning` - prueba opcional con Slack real que confirma que un
  canal desactivado configurado emite una advertencia estructurada sin responder.
- `slack-top-level-reply-shape`
- `slack-restart-resume`
- `slack-progress-commentary-true`, `slack-progress-commentary-false`,
  `slack-progress-commentary-omitted` y
  `slack-progress-commentary-verbose-dedupe` - pruebas opcionales con Slack real para
  controles independientes de comentarios/progreso de herramientas, el valor predeterminado heredado
  cuando se omite la clave y el comportamiento de entrega única cuando está activado el progreso detallado persistente.
- `slack-reaction-glyph-native` - escenario opcional de reacción de la herramienta de mensajes en vivo.
  Indica al agente que envíe el glifo exacto `✅` y confirma que Slack almacenó
  `white_check_mark` para el bot SUT en el mensaje de destino.
- `slack-chart-presentation-native` - escenario opcional de gráfico portátil que
  verifica el bloque nativo `data_visualization` y el texto accesible exacto.
- `slack-table-presentation-native` - escenario opcional de tabla portátil que
  verifica el bloque nativo `data_table`, las filas exactas y el texto accesible.
- `slack-table-invalid-blocks-fallback` - escenario opcional de transporte directo
  que envía, mediante la ruta de envío de Slack de producción,
  una tabla sin procesar, estructuralmente legible y por encima del límite, con 101 filas de datos
  además de su encabezado, demuestra que el propio Slack devuelve `invalid_blocks`
  y verifica que la alternativa almacenada con el formato desactivado está completa y no contiene
  ningún bloque de datos nativo. Los detalles del escenario solo conservan evidencia segura de
  códigos de error, recuentos y valores booleanos.
- `slack-approval-exec-native` - escenario opcional de aprobación nativa de ejecución de Slack.
  Solicita una aprobación de ejecución mediante el Gateway, verifica que el mensaje de Slack
  tenga botones de aprobación nativos, la resuelve y verifica la actualización resuelta de Slack.
- `slack-approval-plugin-native` - escenario opcional de aprobación nativa de Plugin de Slack.
  Activa conjuntamente el reenvío de aprobaciones de ejecución y de Plugin para que los eventos del Plugin
  no queden suprimidos por el enrutamiento de aprobaciones de ejecución, y después verifica la misma
  ruta de la interfaz de usuario nativa de Slack pendiente/resuelta.
- `slack-codex-approval-exec-native` - escenario opcional de aprobación de comandos de Codex Guardian.
  Activa el Plugin de Codex en modo Guardian, enruta un turno de agente del Gateway
  originado en Slack mediante el arnés del servidor de aplicaciones de Codex,
  espera la solicitud de aprobación nativa del Plugin de Slack para
  `openclaw-codex-app-server`, la resuelve y verifica que el turno de Codex
  finalice con los marcadores esperados de salida de comandos y del asistente.
- `slack-codex-approval-plugin-native` - escenario opcional de aprobación de archivos de Codex Guardian.
  Usa una instrucción `apply_patch` fuera del espacio de trabajo para que Codex emita
  la ruta de aprobación de cambios de archivos del servidor de aplicaciones, y después verifica la misma ruta
  de aprobación nativa pendiente/resuelta de Slack, el marcador final del asistente y el contenido exacto del archivo
  antes de la limpieza.

Los escenarios de aprobación de Codex requieren un `openai/*` o `codex/*` `--model`, las
credenciales habituales del modelo en vivo y la autenticación de Codex o mediante clave de API aceptada por el Plugin de Codex.
Los detalles del escenario incluyen el método del servidor de aplicaciones de Codex, la clave del modelo de Codex
seleccionado, el estado final del turno de Codex y la verificación del marcador de operación, junto con
los metadatos censurados de aprobación de Slack.

Artefactos de salida:

- `qa-suite-report.md`
- `qa-suite-summary.json`
- `qa-evidence.json` - entradas de evidencia para las comprobaciones del transporte en vivo.
- `approval-checkpoints/` - solo cuando Mantis establece
  `OPENCLAW_QA_SLACK_APPROVAL_CHECKPOINT_DIR`; contiene el JSON del punto de control,
  el JSON de confirmación y capturas de pantalla pendientes/resueltas.

#### Configuración del espacio de trabajo de Slack

El carril necesita dos aplicaciones distintas de Slack en un espacio de trabajo, además de un canal del que ambos
bots sean miembros:

- `channelId` - el id. `Cxxxxxxxxxx` de un canal al que se haya invitado a ambos bots.
  Use un canal dedicado; el carril publica en cada ejecución.
- `driverBotToken` - token de bot (`xoxb-...`) de la aplicación **Driver**.
- `sutBotToken` - token de bot (`xoxb-...`) de la aplicación **SUT**, que debe ser una
  aplicación de Slack distinta de la del controlador para que su id. de usuario de bot sea diferente.
- `sutAppToken` - token de nivel de aplicación (`xapp-...`) de la aplicación SUT con
  `connections:write`, utilizado por Socket Mode para que la aplicación SUT pueda recibir eventos.

Es preferible usar un espacio de trabajo de Slack dedicado a QA en lugar de reutilizar uno
de producción.

El manifiesto de SUT que aparece a continuación restringe deliberadamente la instalación
de producción del Plugin de Slack incluido (`extensions/slack/src/setup-shared.ts:12`) a los
permisos y eventos que cubre el conjunto de QA en vivo de Slack. Para consultar la configuración
del canal de producción tal como la ven los usuarios, consulte
[Configuración rápida del canal de Slack](/es/channels/slack#quick-setup); el par de QA Driver/SUT
se mantiene separado deliberadamente porque el carril necesita dos id. de usuario de bot distintos
en un mismo espacio de trabajo.

**1. Cree la aplicación Driver**

Vaya a [api.slack.com/apps](https://api.slack.com/apps) → _Create New App_ →
_From a manifest_ → seleccione el espacio de trabajo de QA, pegue el siguiente manifiesto
y después seleccione _Install to Workspace_:

```json
{
  "display_information": {
    "name": "OpenClaw QA Driver",
    "description": "Bot controlador de pruebas para el carril en vivo de QA de Slack de OpenClaw"
  },
  "features": {
    "bot_user": {
      "display_name": "OpenClaw QA Driver",
      "always_online": true
    }
  },
  "oauth_config": {
    "scopes": {
      "bot": ["chat:write", "channels:history", "groups:history", "users:read"]
    }
  },
  "settings": {
    "socket_mode_enabled": false
  }
}
```

Copie el _Bot User OAuth Token_ (`xoxb-...`); este se convierte en
`driverBotToken`. El controlador solo necesita publicar mensajes e identificarse;
no necesita eventos ni Socket Mode.

**2. Cree la aplicación SUT**

Repita _Create New App → From a manifest_ en el mismo espacio de trabajo. Esta aplicación de QA
utiliza deliberadamente una versión más restringida del manifiesto de producción del Plugin de Slack
incluido (`extensions/slack/src/setup-shared.ts:12`): se omiten los
ámbitos y eventos de reacciones porque el conjunto de QA en vivo de Slack aún no cubre
la gestión de reacciones.

```json
{
  "display_information": {
    "name": "OpenClaw QA SUT",
    "description": "Conector OpenClaw QA SUT para OpenClaw"
  },
  "features": {
    "bot_user": {
      "display_name": "OpenClaw QA SUT",
      "always_online": true
    },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    }
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "emoji:read",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "mpim:history",
        "mpim:read",
        "mpim:write",
        "pins:read",
        "pins:write",
        "usergroups:read",
        "users:read"
      ]
    }
  },
  "settings": {
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed"
      ]
    }
  }
}
```

Después de que Slack cree la aplicación, haga dos cosas en su página de configuración:

- _Install to Workspace_ → copie el _Bot User OAuth Token_ → este se convierte en
  `sutBotToken`.
- _Basic Information → App-Level Tokens → Generate Token and Scopes_ → añada
  el ámbito `connections:write` → guarde → copie el valor `xapp-...` → este
  se convierte en `sutAppToken`.

Compruebe que los dos bots tengan identificadores de usuario distintos llamando a `auth.test` con cada
token. El entorno de ejecución distingue el controlador y el SUT por el identificador de usuario; reutilizar una aplicación
para ambos hará que el control de menciones falle inmediatamente.

**3. Crear el canal**

En el espacio de trabajo de QA, cree un canal (p. ej., `#openclaw-qa`) e invite a ambos
bots desde el canal:

```text
/invite @OpenClaw QA Driver
/invite @OpenClaw QA SUT
```

Copie el identificador `Cxxxxxxxxxx` de _channel info → About → Channel ID_; este
se convierte en `channelId`. Un canal público funciona; si utiliza un canal privado,
ambas aplicaciones ya tienen `groups:history`, por lo que las lecturas del historial del arnés
también se realizarán correctamente.

**4. Registrar las credenciales**

Hay dos opciones. Utilice variables de entorno para depurar en una sola máquina (establezca las cuatro
variables `OPENCLAW_QA_SLACK_*` y pase `--credential-source env`) o inicialice
el grupo compartido de Convex para que la Pipeline de CI y otros mantenedores puedan arrendarlas.

Para el grupo de Convex, escriba los cuatro campos en un archivo JSON:

```json
{
  "channelId": "Cxxxxxxxxxx",
  "driverBotToken": "xoxb-...",
  "sutBotToken": "xoxb-...",
  "sutAppToken": "xapp-..."
}
```

Con `OPENCLAW_QA_CONVEX_SITE_URL` y `OPENCLAW_QA_CONVEX_SECRET_MAINTAINER`
exportados en el shell, registre y compruebe:

```bash
pnpm openclaw qa credentials add \
  --kind slack \
  --payload-file slack-creds.json \
  --note "QA Slack pool seed"

pnpm openclaw qa credentials list --kind slack --status all --json
```

Se esperan `count: 1`, `status: "active"` y ningún campo `lease`.

**5. Verificar de extremo a extremo**

Ejecute la vía localmente para confirmar que ambos bots puedan comunicarse entre sí mediante el
intermediario:

```bash
pnpm openclaw qa slack \
  --credential-source convex \
  --credential-role maintainer \
  --output-dir .artifacts/qa-e2e/slack-local
```

Una ejecución correcta se completa en bastante menos de 30 segundos y `qa-suite-report.md`
muestra tanto `slack-canary` como `slack-mention-gating` con el estado `pass`. Si la
vía se bloquea durante ~90 segundos y termina con `Convex credential pool exhausted
for kind "slack"`, el grupo está vacío o todas las filas están arrendadas; `qa
credentials list --kind slack --status all --json` indicará cuál es el caso.

### QA de WhatsApp

```bash
pnpm openclaw qa whatsapp
```

Se dirige a dos cuentas dedicadas de WhatsApp Web: una cuenta controladora gestionada por
el arnés y una cuenta SUT iniciada por el Gateway secundario de OpenClaw mediante
el plugin de WhatsApp incluido.

Entorno requerido cuando `--credential-source env`:

- `OPENCLAW_QA_WHATSAPP_DRIVER_PHONE_E164`
- `OPENCLAW_QA_WHATSAPP_SUT_PHONE_E164`
- `OPENCLAW_QA_WHATSAPP_DRIVER_AUTH_ARCHIVE_BASE64`
- `OPENCLAW_QA_WHATSAPP_SUT_AUTH_ARCHIVE_BASE64`

Opcional:

- `OPENCLAW_QA_WHATSAPP_GROUP_JID` habilita escenarios de grupo como
  `whatsapp-mention-gating`, `whatsapp-group-pending-history-context`,
  `whatsapp-broadcast-group-fanout`, `whatsapp-group-activation-always`,
  `whatsapp-group-reply-to-bot-triggers`, escenarios de acciones, medios y encuestas de grupo,
  y `whatsapp-group-allowlist-block`.

Escenarios YAML de WhatsApp (`qa/scenarios/channels/whatsapp-*.yaml`):

- Línea base y control de grupos: `whatsapp-canary`, `whatsapp-pairing-block`,
  `whatsapp-mention-gating`, `whatsapp-group-pending-history-context`,
  `whatsapp-group-activation-always`, `whatsapp-group-reply-to-bot-triggers`,
  `whatsapp-top-level-reply-shape`, `whatsapp-restart-resume`,
  `whatsapp-group-allowlist-block`.
- Comandos nativos: `whatsapp-help-command`, `whatsapp-status-command`,
  `whatsapp-commands-command`, `whatsapp-tools-compact-command`,
  `whatsapp-whoami-command`, `whatsapp-context-command`,
  `whatsapp-native-new-command`.
- Comportamiento de respuesta y salida final: `whatsapp-tool-only-usage-footer`,
  `whatsapp-reply-to-message`, `whatsapp-group-reply-to-message`,
  `whatsapp-reply-to-mode-batched`, `whatsapp-reply-context-isolation`,
  `whatsapp-reply-delivery-shape`, `whatsapp-stream-final-message-accounting`.
- Acciones de mensajes de la ruta del usuario: `whatsapp-agent-message-action-react` comienza
  desde un mensaje directo real del controlador, permite que el modelo llame a la herramienta `message` y
  observa la reacción nativa de WhatsApp. `whatsapp-agent-message-action-upload-file`
  utiliza el mismo enfoque para `message(action=upload-file)` y observa
  contenido multimedia nativo de WhatsApp. `whatsapp-group-agent-message-action-react` y
  `whatsapp-group-agent-message-action-upload-file` demuestran las mismas
  acciones visibles para el usuario en un grupo real de WhatsApp.
- Distribución en grupos: `whatsapp-broadcast-group-fanout` comienza con un mensaje de grupo de
  WhatsApp que contiene una mención y comprueba respuestas visibles distintas de `main`
  y `qa-second`.
- Activación de grupos: `whatsapp-group-activation-always` cambia una sesión de grupo real
  a `/activation always`, demuestra que un mensaje de grupo sin mención activa
  al agente y después restaura `/activation mention`.
  `whatsapp-group-reply-to-bot-triggers` prepara una respuesta del bot, envía una
  respuesta nativa que la cita sin una mención explícita y comprueba que el agente
  se activa a partir del contexto de esa respuesta.
- Medios entrantes y mensajes estructurados: `whatsapp-inbound-image-caption`,
  `whatsapp-audio-preflight`, `whatsapp-inbound-structured-messages`,
  `whatsapp-group-audio-gating`, `whatsapp-inbound-reaction-no-trigger`.
  Estos envían eventos reales de imágenes, audio, documentos, ubicaciones, contactos,
  adhesivos y reacciones de WhatsApp mediante el controlador.
- Sondeos directos del contrato del Gateway: `whatsapp-outbound-media-matrix`,
  `whatsapp-outbound-document-preserves-filename`, `whatsapp-outbound-poll`,
  `whatsapp-outbound-send-serialization`,
  `whatsapp-group-outbound-media`, `whatsapp-group-outbound-poll`,
  `whatsapp-message-actions`, `whatsapp-reply-context-isolation`,
  `whatsapp-reply-delivery-shape`. Estos omiten deliberadamente las solicitudes al modelo
  y demuestran de forma determinista los contratos de `send`, `poll` y
  `message.action` del Gateway y el canal.
- Cobertura del control de acceso: `whatsapp-access-control-dm-open`,
  `whatsapp-access-control-dm-disabled`, `whatsapp-access-control-group-open`,
  `whatsapp-access-control-group-disabled`, `whatsapp-group-allowlist-block`.
- Aprobaciones nativas: `whatsapp-approval-exec-deny-native`,
  `whatsapp-approval-exec-native`, `whatsapp-approval-exec-reaction-native`,
  `whatsapp-approval-exec-group-reaction-native`,
  `whatsapp-approval-plugin-native`.
- Reacciones de estado: `whatsapp-status-reactions`,
  `whatsapp-status-reaction-lifecycle`.

Actualmente, el catálogo contiene 52 escenarios. La vía predeterminada `live-frontier`
se mantiene reducida a 8 escenarios para ofrecer una cobertura de humo rápida. La vía predeterminada `mock-openai`
ejecuta 39 escenarios de forma determinista mediante el transporte real de WhatsApp,
simulando únicamente la salida del modelo; los escenarios de aprobación y algunas
comprobaciones más pesadas o bloqueantes siguen siendo explícitos mediante el identificador del escenario.

El controlador de QA de WhatsApp observa eventos en vivo estructurados (`text`, `media`,
`location`, `reaction` y `poll`) y puede enviar activamente medios, encuestas,
contactos, ubicaciones y adhesivos. QA Lab importa ese controlador mediante la
superficie del paquete `@openclaw/whatsapp/api.js`, en lugar de acceder a archivos privados
del entorno de ejecución de WhatsApp. Para las observaciones de grupo, `fromJid` es el JID del grupo,
mientras que `participantJid` y `fromPhoneE164` identifican al participante remitente.
El contenido de los mensajes se oculta de forma predeterminada. Los sondeos directos del Gateway sobre encuestas, carga de archivos,
medios, encuestas de grupo, medios de grupo y forma de las respuestas son comprobaciones
del contrato de transporte/API; no se consideran una prueba de que una solicitud del usuario hiciera que el
agente eligiera la misma acción. La prueba de las acciones de la ruta del usuario procede de escenarios
como `whatsapp-agent-message-action-react` y
`whatsapp-group-agent-message-action-react`, en los que el controlador envía un mensaje normal de
WhatsApp y QA Lab observa el artefacto nativo resultante de WhatsApp.
Los detalles de los escenarios de WhatsApp incluyen el enfoque de cada escenario (`user-path`,
`direct-gateway` o `native-approval`) para evitar que las pruebas se confundan con un
contrato más sólido de lo que realmente demuestran.

Artefactos de salida:

- `qa-suite-report.md`
- `qa-suite-summary.json`
- `qa-evidence.json`: entradas de pruebas para las comprobaciones del transporte en vivo.

### Grupo de credenciales de Convex

Las vías de Discord, Slack, Telegram y WhatsApp pueden arrendar credenciales de un
grupo compartido de Convex en lugar de leer las variables de entorno anteriores. Pase
`--credential-source convex` (o establezca `OPENCLAW_QA_CREDENTIAL_SOURCE=convex`);
QA Lab adquiere un arrendamiento exclusivo, le envía Heartbeat durante toda la
ejecución y lo libera al apagarse. Los tipos del grupo son `"discord"`, `"slack"`,
`"telegram"` y `"whatsapp"`.

Formatos de carga útil que el intermediario valida en `admin/add`:

- Discord (`kind: "discord"`): `{ guildId: string, channelId: string,
driverBotToken: string, sutBotToken: string, sutApplicationId: string }`.
- Telegram (`kind: "telegram"`): `{ groupId: string, driverToken: string,
sutToken: string }`; `groupId` debe ser una cadena numérica de identificador de chat.
- Usuario real de Telegram (`kind: "telegram-user"`): `{ groupId: string, sutToken:
string, testerUserId: string, testerUsername: string, telegramApiId:
string, telegramApiHash: string, tdlibDatabaseEncryptionKey: string,
tdlibArchiveBase64: string, tdlibArchiveSha256: string,
desktopTdataArchiveBase64: string, desktopTdataArchiveSha256: string }`;
  solo para las pruebas de Telegram Desktop de Mantis. Las vías genéricas de QA Lab no deben adquirir
  este tipo.
- WhatsApp (`kind: "whatsapp"`): `{ driverPhoneE164: string, sutPhoneE164:
string, driverAuthArchiveBase64: string, sutAuthArchiveBase64: string,
groupJid?: string }`; los números de teléfono deben ser cadenas E.164 distintas.

El flujo de pruebas de Telegram Desktop de Mantis mantiene un arrendamiento exclusivo de Convex
`telegram-user` tanto para el controlador de la CLI de TDLib como para el
testigo de Telegram Desktop y después lo libera tras publicar las pruebas.

Cuando un pull request necesita una comparación visual determinista, Mantis puede utilizar la misma respuesta
del modelo simulado en `main` y en la cabecera del pull request mientras cambia el formateador
o la capa de entrega de Telegram. Los valores predeterminados de captura están ajustados para los comentarios de los pull requests: clase
estándar de Crabbox, grabación de escritorio a 24fps, GIF de movimiento a 24fps y ancho de vista previa
de 1920px. Los comentarios del antes y el después deben publicar un paquete limpio que contenga
únicamente los GIF previstos.

Las vías de Slack también pueden utilizar el grupo. Actualmente, las comprobaciones del formato de carga útil de Slack se encuentran
en el ejecutor de QA de Slack en lugar del intermediario; utilice `{ channelId: string,
driverBotToken: string, sutBotToken: string, sutAppToken: string }`, con un
identificador de canal de Slack como `Cxxxxxxxxxx`. Consulte
[Configurar el espacio de trabajo de Slack](#setting-up-the-slack-workspace) para el aprovisionamiento
de aplicaciones y ámbitos.

Las variables de entorno operativas y el contrato del punto de conexión del intermediario de Convex se encuentran en
[Pruebas → Credenciales compartidas de Telegram mediante Convex](/es/help/testing#shared-telegram-credentials-via-convex-v1)
(el nombre de la sección es anterior al grupo multicanal; la semántica del arrendamiento se
comparte entre los distintos tipos).

## Datos iniciales respaldados por el repositorio

Los recursos iniciales se encuentran en `qa/`:

- `qa/scenarios/index.yaml`
- `qa/scenarios/<theme>/*.yaml`

Estos se incluyen intencionadamente en git para que el plan de QA sea visible tanto para las personas como para
el agente.

`qa-lab` sigue siendo un ejecutor genérico de escenarios YAML. Cada archivo YAML de escenario es la
fuente de verdad para una ejecución de prueba y debe definir:

- `title` de nivel superior
- metadatos de `scenario`
- metadatos opcionales de categoría, capacidad, vía y riesgo en `scenario`
- referencias a documentación y código en `scenario`
- requisitos opcionales de plugins en `scenario`
- parche opcional de configuración del Gateway en `scenario`
- `flow` ejecutable de nivel superior para escenarios de flujo, o
  `scenario.execution.kind` / `scenario.execution.path` para escenarios de Vitest y
  Playwright

La superficie de runtime reutilizable que sustenta `flow` se mantiene genérica y
transversal. Por ejemplo, los escenarios YAML pueden combinar auxiliares del lado
del transporte con auxiliares del lado del navegador que controlan la interfaz de control integrada mediante
la unión `browser.request` del Gateway sin añadir un ejecutor para casos especiales.

Los archivos de escenarios deben agruparse por capacidad del producto en lugar de por carpeta
del árbol de fuentes. Mantenga estables los identificadores de escenarios cuando se muevan los archivos; use `docsRefs` y
`codeRefs` para la trazabilidad de la implementación.

La lista de referencia debe ser lo bastante amplia para abarcar:

- mensajes directos y chat de canales
- comportamiento de los hilos
- ciclo de vida de las acciones de mensajes
- retrollamadas de Cron
- recuperación de memoria
- cambio de modelo
- transferencia a subagentes
- lectura del repositorio y de la documentación
- una tarea de compilación pequeña, como Lobster Invaders

## Vías de simulación de proveedores

`qa suite` tiene dos vías locales de simulación de proveedores:

- `mock-openai` es la simulación de OpenClaw que reconoce los escenarios. Sigue siendo la vía
  de simulación determinista predeterminada para el control de calidad respaldado por el repositorio y las puertas de paridad.
- `aimock` inicia un servidor de proveedor respaldado por AIMock para la cobertura experimental
  de protocolos, datos de prueba, grabación/reproducción y caos. Es una adición y
  no sustituye al despachador de escenarios `mock-openai`.

La implementación de las vías de proveedores se encuentra en `extensions/qa-lab/src/providers/`.
Cada proveedor posee sus valores predeterminados, el inicio del servidor local, la configuración del modelo del Gateway,
las necesidades de preparación de perfiles de autenticación y los indicadores de capacidades reales/simuladas. El código compartido de la suite y
del Gateway se enruta mediante el registro de proveedores en lugar de bifurcarse según
los nombres de los proveedores.

## Adaptadores de transporte

`qa-lab` posee una unión de transporte genérica para los escenarios de control de calidad YAML. `qa-channel` es
el valor predeterminado sintético. `crabline` inicia servidores locales con la forma de los proveedores y
ejecuta los plugins de canales normales de OpenClaw contra ellos. `live` se reserva para
credenciales reales de proveedores y canales externos.

En el ámbito de la arquitectura, la división es:

- `qa-lab` se encarga de la ejecución genérica de escenarios, la concurrencia de procesos de trabajo, la escritura
  de artefactos y la generación de informes.
- El adaptador de transporte se encarga de la configuración del Gateway, la disponibilidad, la observación
  entrante y saliente, las acciones de transporte y el estado de transporte normalizado.
- Los archivos de escenarios YAML de `qa/scenarios/` definen la ejecución de las pruebas; `qa-lab`
  proporciona la superficie de runtime reutilizable que los ejecuta.

### Añadir un canal

Añadir un canal al sistema de control de calidad YAML requiere la implementación del canal
más un paquete de escenarios que ejercite el contrato del canal. Para la cobertura
de CI de pruebas de humo, añada el servidor de proveedor local de Crabline correspondiente y expóngalo
mediante el controlador `crabline`.

No añada una nueva raíz de comandos de control de calidad de nivel superior cuando el host compartido `qa-lab`
pueda gestionar el flujo.

`qa-lab` se encarga de los mecanismos compartidos del host:

- la raíz de comandos `openclaw qa`
- inicio y finalización de la suite
- concurrencia de procesos de trabajo
- escritura de artefactos
- generación de informes
- ejecución de escenarios
- alias de compatibilidad para escenarios `qa-channel` anteriores

Los plugins de ejecución se encargan del contrato de transporte:

- cómo se monta `openclaw qa <runner>` bajo la raíz compartida `qa`
- cómo se configura el Gateway para ese transporte
- cómo se comprueba la disponibilidad
- cómo se inyectan los eventos entrantes
- cómo se observan los mensajes salientes
- cómo se exponen las transcripciones y el estado de transporte normalizado
- cómo se ejecutan las acciones respaldadas por el transporte
- cómo se gestionan el restablecimiento o la limpieza específicos del transporte

Los requisitos mínimos de adopción para un canal nuevo:

1. Mantenga `qa-lab` como responsable de la raíz compartida `qa`.
2. Implemente el ejecutor de transporte en la unión del host compartido `qa-lab`.
3. Mantenga los mecanismos específicos del transporte dentro del plugin de ejecución o del arnés
   del canal.
4. Monte el ejecutor como `openclaw qa <runner>` en lugar de registrar una
   raíz de comandos competidora. Los plugins de ejecución deben declarar `qaRunners` en
   `openclaw.plugin.json` y exportar un array `qaRunnerCliRegistrations`
   correspondiente desde `runtime-api.ts`. Mantenga `runtime-api.ts` ligero; la CLI diferida y
   la ejecución del ejecutor deben permanecer tras puntos de entrada separados. Un
   `adapterFactory` opcional expone el transporte a los escenarios compartidos sin cambiar
   el catálogo de escenarios existente del comando. Las particiones del mismo canal son secuenciales,
   salvo que la factoría declare que cada instancia posee credenciales aisladas o
   servidores desechables, estado del Gateway y rutas de artefactos.
5. Cree o adapte escenarios YAML en los directorios temáticos `qa/scenarios/`.
6. Use los auxiliares genéricos de escenarios para los escenarios nuevos.
7. Mantenga en funcionamiento los alias de compatibilidad existentes, salvo que el repositorio lleve a cabo una
   migración intencionada.

La regla de decisión es estricta:

- Si un comportamiento puede expresarse una sola vez en `qa-lab`, colóquelo en `qa-lab`.
- Si un comportamiento depende del transporte de un canal, manténgalo en ese plugin
  de ejecución o en el arnés del plugin.
- Si un escenario necesita una capacidad nueva que pueda usar más de un canal,
  añada un auxiliar genérico en lugar de una rama específica del canal en `suite.ts`.
- Si un comportamiento solo tiene sentido para un transporte, mantenga el escenario
  específico del transporte y hágalo explícito en el contrato del escenario.

### Nombres de auxiliares de escenarios

Auxiliares genéricos preferidos para escenarios nuevos:

- `waitForTransportReady`
- `waitForChannelReady`
- `injectInboundMessage`
- `injectOutboundMessage`
- `waitForTransportOutboundMessage`
- `waitForChannelOutboundMessage`
- `waitForNoTransportOutbound`
- `getTransportSnapshot`
- `readTransportMessage`
- `readTransportTranscript`
- `formatTransportTranscript`
- `resetTransport`

Los alias de compatibilidad siguen disponibles para los escenarios existentes:
`waitForQaChannelReady`, `waitForOutboundMessage`, `waitForNoOutbound`,
`formatConversationTranscript`, `resetBus`, pero la creación de escenarios nuevos
debe usar los nombres genéricos. Los alias existen para evitar una migración
inmediata de todo el sistema, no como modelo de cara al futuro.

## Informes

`qa-lab` exporta un informe de protocolo en Markdown a partir de la cronología observada del bus.
El informe debe responder:

- Qué funcionó
- Qué falló
- Qué permaneció bloqueado
- Qué escenarios de seguimiento merece la pena añadir

Para consultar el inventario de escenarios disponibles, útil al dimensionar el trabajo de seguimiento
o conectar un transporte nuevo, ejecute `pnpm openclaw qa coverage` (añada `--json`
para obtener una salida legible por máquinas). Al elegir una prueba específica para un
comportamiento o una ruta de archivo modificados, ejecute `pnpm openclaw qa coverage --match <query>`. El
informe de coincidencias busca en los metadatos de los escenarios, las referencias de documentación, las referencias de código, los identificadores de cobertura,
los plugins y los requisitos de proveedores, y luego muestra los objetivos `qa suite
--scenario ...` coincidentes.

Cada ejecución de `qa suite` escribe los artefactos de nivel superior `qa-evidence.json`,
`qa-suite-summary.json` y `qa-suite-report.md` para el conjunto de
escenarios seleccionado. Los escenarios que declaran `execution.kind: vitest` o
`execution.kind: playwright` ejecutan la ruta de prueba correspondiente y también escriben
registros por escenario. Los escenarios que declaran `execution.kind: script` ejecutan el
productor de evidencias en `execution.path` mediante `node --import tsx` (con
`${outputDir}` y `${scenarioId}` expandidos en `execution.args`); el
productor escribe su propio `qa-evidence.json`, cuyas entradas se importan en
la salida de la suite y cuyas rutas de artefactos se resuelven con respecto a ese
`qa-evidence.json` del productor. Cuando se llega a `qa suite` mediante `qa run
--qa-profile`, el mismo `qa-evidence.json` también incluye el resumen de la tabla
de puntuaciones del perfil para las categorías taxonómicas seleccionadas.

Trate la salida de cobertura como una ayuda para el descubrimiento, no como un sustituto de las puertas; el
escenario seleccionado sigue necesitando el modo de proveedor, el transporte real,
Multipass, Testbox o la vía de lanzamiento adecuados para el comportamiento sometido a prueba. Para
obtener el contexto de la tabla de puntuaciones, consulte [Tabla de puntuaciones de madurez](/es/maturity/scorecard).

Para las comprobaciones de carácter y estilo, ejecute el mismo escenario con varias referencias de modelos
reales y escriba un informe Markdown evaluado:

```bash
pnpm openclaw qa character-eval \
  --model openai/gpt-5.6-luna,thinking=medium,fast \
  --model openai/gpt-5.2,thinking=xhigh \
  --model openai/gpt-5,thinking=xhigh \
  --model anthropic/claude-opus-4-8,thinking=high \
  --model anthropic/claude-sonnet-4-6,thinking=high \
  --model zai/glm-5.1,thinking=high \
  --model moonshot/kimi-k2.5,thinking=high \
  --model google/gemini-3.1-pro-preview,thinking=high \
  --judge-model openai/gpt-5.6-sol,thinking=xhigh,fast \
  --judge-model anthropic/claude-opus-4-8,thinking=high \
  --blind-judge-models \
  --concurrency 16 \
  --judge-concurrency 16
```

El comando ejecuta procesos secundarios locales del Gateway de control de calidad, no Docker. Los escenarios de
evaluación de carácter deben establecer la personalidad mediante `SOUL.md` y, a continuación, ejecutar interacciones normales
del usuario, como conversaciones, ayuda sobre el espacio de trabajo y pequeñas tareas con archivos. No se debe
informar al modelo candidato de que está siendo evaluado. El comando conserva
cada transcripción completa, registra estadísticas básicas de la ejecución y, a continuación, solicita a los modelos evaluadores en
modo rápido y con razonamiento `xhigh`, cuando sea compatible, que clasifiquen las ejecuciones por
naturalidad, estilo y humor. Use `--blind-judge-models` al comparar
proveedores: el prompt del evaluador sigue recibiendo cada transcripción y estado de ejecución, pero
las referencias de los candidatos se sustituyen por etiquetas neutras como `candidate-01`; el
informe vuelve a asociar las clasificaciones con las referencias reales después del análisis.

Las ejecuciones candidatas usan de forma predeterminada el nivel de razonamiento `high`, con `medium` para GPT-5.6 Luna y
`xhigh` para las referencias de evaluación de OpenAI anteriores que lo admiten. Sobrescriba un
candidato concreto en línea con `--model provider/model,thinking=<level>`; las opciones
en línea también admiten `fast`, `no-fast` y `fast=<bool>`. `--thinking
<level>` sigue estableciendo un valor de reserva global, y la forma anterior `--model-thinking
<provider/model=level>` se conserva por compatibilidad. Las referencias candidatas
de OpenAI usan el modo rápido de forma predeterminada para que se utilice el procesamiento prioritario cuando el proveedor
lo admita. Pase `--fast` solo cuando quiera forzar la activación del modo rápido para
todos los modelos candidatos. Las duraciones de los candidatos y los evaluadores se registran en el
informe para el análisis comparativo, pero los prompts de los evaluadores indican explícitamente que no deben clasificar
por velocidad. Tanto las ejecuciones de los modelos candidatos como las de los evaluadores usan una concurrencia predeterminada de 16.
Reduzca `--concurrency` o `--judge-concurrency` cuando los límites del proveedor o la presión sobre el
Gateway local generen demasiado ruido en una ejecución.

Cuando no se pasa ningún `--model` candidato, la evaluación de carácter usa de forma predeterminada
`openai/gpt-5.6-luna`, `openai/gpt-5.2`, `openai/gpt-5`,
`anthropic/claude-opus-4-8`, `anthropic/claude-sonnet-4-6`, `zai/glm-5.1`,
`moonshot/kimi-k2.5` y `google/gemini-3.1-pro-preview`. Cuando no se pasa ningún
`--judge-model`, los evaluadores predeterminados son
`openai/gpt-5.6-sol,thinking=xhigh,fast` y
`anthropic/claude-opus-4-8,thinking=high`.

## Documentación relacionada

- [Tabla de puntuaciones de madurez](/es/maturity/scorecard)
- [Paquete de pruebas comparativas para agentes personales](/es/concepts/personal-agent-benchmark-pack)
- [Canal de control de calidad](/es/channels/qa-channel)
- [Pruebas](/es/help/testing)
- [Panel de control](/es/web/dashboard)
