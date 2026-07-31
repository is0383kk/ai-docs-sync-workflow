---
doc-schema-version: 1
read_when:
    - Ejecución o repetición de la validación completa de la versión
    - Comparación de los perfiles de validación de versiones estable y completa
    - Depuración de fallos en las etapas de validación de versiones
summary: Etapas de validación completa de la versión, flujos de trabajo secundarios, perfiles de versión, identificadores de reejecución y pruebas
title: Validación completa de la versión
x-i18n:
    generated_at: "2026-07-26T04:57:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ddf165d5515f4b9bb11d239382649d332d20bb8a32bd4492ae99092fb5ee2216
    source_path: reference/full-release-validation.md
    workflow: 16
---

`Full Release Validation` es el marco general de validación del producto para la versión. La mayor parte del trabajo
se realiza en flujos de trabajo secundarios para que se pueda volver a ejecutar una máquina con errores sin reiniciar
toda la versión. Ejecute la preparación de la versión antes de fijar el SHA del código; esta
actualiza la salida de configuración regional de la interfaz de control cuando el bot en segundo plano aún no la ha incorporado
y, a continuación, aplica la misma comprobación estricta de cero mecanismos de respaldo utilizada por el Pipeline de CI de la versión.

Fije la confirmación previa al registro de cambios con el producto completo como **SHA del código** y, a continuación, ejecute:

```bash
pnpm ci:full-release \
  --sha <code-sha> \
  --target-ref release/YYYY.M.PATCH
```

`provider` también acepta `anthropic` o `minimax` para la incorporación entre sistemas operativos y el
turno integral del agente. El asistente infiere el perfil `beta` a partir de las versiones
alfa/beta del paquete y `stable` en caso contrario. Pase entradas alternativas del flujo de trabajo con
`-f key=value`; use `-f release_profile=full` únicamente para la revisión exhaustiva de avisos.

El asistente crea una referencia `release-ci/*` temporal fijada a un único SHA de flujo de trabajo
`origin/main` de confianza, pasa el SHA de destino únicamente como `ref` candidato
y elimina la referencia temporal después de la validación. Cada flujo secundario iniciado debe
informar del mismo SHA de flujo de trabajo. Pase
`-f reuse_evidence=false` para forzar una ejecución nueva o
`--workflow-sha <trusted-main-sha>` para seleccionar una confirmación anterior del flujo de trabajo que aún sea
accesible desde el `origin/main` actual. El flujo de trabajo nunca crea ni actualiza
por sí mismo las referencias del repositorio.

## Excepción para la versión estable ampliada

La publicación estable ampliada requiere una ejecución en la que tanto el flujo de trabajo como el destino sean
la rama canónica:

```bash
gh workflow run full-release-validation.yml \
  --ref extended-stable/YYYY.M.33 \
  -f ref=extended-stable/YYYY.M.33 \
  -f release_profile=stable
```

No use `pnpm ci:full-release` ni `release-ci/*`. La publicación vincula la
rama, el SHA de cabecera/destino, el `workflowRef` del manifiesto, el ID y el intento de la ejecución con la rama
canónica y la confirmación de la versión.

Aplique retroportaciones a los fallos del producto; realice la reparación más pequeña que preserve el comportamiento para
las herramientas del destino fijado; vuelva a intentar los fallos del proveedor, de aprobación o del ejecutor sin
cambiar el código fuente. Cualquier cambio de rama requiere una ejecución nueva completa. No omita comportamientos
obligatorios del paquete, instalador, actualización, canal o entorno real porque el destino sea antiguo.

Para una versión normal, cuando el SHA del código esté en verde, genere y confirme únicamente
`CHANGELOG.md`. Esta nueva confirmación es el **SHA de la versión**. Ejecute el mismo asistente para
el SHA de la versión. Las pruebas del producto solo se reutilizan cuando GitHub demuestra que el SHA de la versión
desciende del SHA del código y que el conjunto completo de rutas modificadas es exactamente
`CHANGELOG.md`; la comprobación preliminar de npm y la aceptación del paquete/instalación siguen ejecutándose en el
SHA de la versión.

`release_profile=stable` y `release_profile=full` siempre ejecutan la prueba exhaustiva
prolongada en entorno real/Docker. Pase `run_release_soak=true` para incluir las mismas vías de prueba prolongada
con el perfil `beta`. La publicación estable rechaza un manifiesto de validación
que no contenga esta prueba prolongada ni pruebas bloqueantes del rendimiento del producto.

La aceptación del paquete normalmente compila el archivo tar candidato a partir de la
`ref` resuelta, incluidas las ejecuciones con SHA completo iniciadas mediante `pnpm ci:full-release`. Después de una
publicación beta, pase `release_package_spec=openclaw@YYYY.M.PATCH-beta.N` para reutilizar
el paquete npm publicado en las comprobaciones de la versión, la aceptación del paquete, las pruebas entre sistemas operativos,
la ruta de versión de Docker y el paquete de Telegram. Use `package_acceptance_package_spec`
únicamente cuando la aceptación del paquete deba demostrar intencionadamente un paquete diferente.
La vía del paquete en entorno real del Plugin de Codex sigue el mismo estado: los valores
`release_package_spec` publicados derivan `codex_plugin_spec=npm:@openclaw/codex@<version>`;
las ejecuciones de SHA/artefacto empaquetan `extensions/codex` desde la referencia seleccionada; y los operadores
pueden establecer `codex_plugin_spec` directamente para fuentes del Plugin
`npm:`, `npm-pack:` o `git:`. La vía concede la aprobación explícita de instalación de la CLI de Codex requerida por
ese Plugin y, a continuación, ejecuta la comprobación preliminar de la CLI de Codex y turnos del agente de OpenAI en la misma sesión.
Su turno final sin reintentos y con razonamiento medio envía progreso visible con
`final` de Codex omitido, lee entradas aleatorias del espacio de trabajo, escribe su artefacto exacto
y envía una finalización explícita. Esto detecta la regresión de v2026.7.1 en la que un
envío normal de progreso finalizaba el turno.

## Etapas de nivel superior

Para `rerun_group=all`, primero se ejecuta un trabajo
`Check for reusable validation evidence`. Busca la validación completa correcta anterior más reciente con el mismo perfil de
versión, la misma configuración efectiva de prueba prolongada y las mismas entradas de validación. Las repeticiones del destino exacto usan
`exact-target-full-validation-v1`. Un descendiente cuyo delta completo sea exactamente
`CHANGELOG.md` usa `changelog-only-release-v1`; se omiten todas las vías del producto
y el verificador vuelve a comprobar de forma independiente la comparación de confirmaciones de GitHub, el
artefacto principal inmutable, las ejecuciones secundarias y los registros de inicio. Cualquier otro cambio en el destino requiere
una validación nueva del SHA del código. Pase `reuse_evidence=false` para forzar una ejecución completa
nueva. La reutilización de pruebas solo se ejecuta desde `main` o una referencia
`release-ci/*` canónica fijada a un SHA cuya confirmación del flujo de trabajo permanezca en el linaje de confianza
`main`; otras referencias del flujo de trabajo ejecutan desde cero las vías seleccionadas.

La validación nueva orientada a paquetes prepara un archivo tar inmutable y un artefacto de imagen
Docker antes de iniciar la versión preliminar de plugins y las comprobaciones de versión de OpenClaw.
Ambos flujos secundarios verifican el mismo SHA del paquete, los ID de artefactos, los resúmenes de servicio,
el intento de ejecución del productor y el resumen del archivo de Docker antes de usarlos. La capa básica de Docker,
independiente del paquete, utiliza una caché GHCR direccionada por contenido; las imágenes específicas
del candidato siguen siendo artefactos inmutables de GitHub. Las ejecuciones específicas con una especificación explícita
de paquete publicado conservan en su lugar la ruta del paquete existente.

También para `rerun_group=all`, un trabajo `Verify Docker runtime image assets` compila
el destino de Docker `runtime-assets` con
`OPENCLAW_EXTENSIONS=diagnostics-otel,codex`. Se ejecuta en paralelo con las
demás etapas y el verificador general exige que se complete; las vías ya no esperan
a que termine antes de iniciarse. Un `rerun_group` más limitado omite esta comprobación preliminar.

| Etapa                   | Detalles                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Resolución del destino       | **Trabajo:** `Resolve target ref`<br />**Flujo de trabajo secundario:** ninguno<br />**Demuestra:** resuelve la rama, etiqueta o SHA completo de confirmación de la versión y registra las entradas seleccionadas.<br />**Repetición:** vuelva a ejecutar el marco general si falla.                                                                                                                                                                                                                                                                                                            |
| Candidato compartido        | **Trabajo:** `Prepare shared release candidate`<br />**Flujo de trabajo secundario:** `OpenClaw Live And E2E Checks (Reusable)`<br />**Demuestra:** empaqueta y valida un paquete con un SHA exacto, compila una imagen funcional de Docker y registra tuplas inmutables de artefactos del paquete y de la imagen para ambos flujos de trabajo secundarios orientados a paquetes.<br />**Repetición:** vuelva a ejecutar el grupo afectado de paquete, versión preliminar de plugins, pruebas entre sistemas operativos o entorno real/E2E.                                                                                                                 |
| Comprobación preliminar de recursos de Docker | **Trabajo:** `Verify Docker runtime image assets`<br />**Flujo de trabajo secundario:** ninguno<br />**Demuestra:** el destino de compilación de Docker `runtime-assets` sigue completándose correctamente antes de que se inicie cualquier otra etapa. Solo se ejecuta para `rerun_group=all`.<br />**Repetición:** vuelva a ejecutar el marco general con `rerun_group=all`.                                                                                                                                                                                                                                         |
| Vitest y CI normal    | **Trabajo:** `Run normal full CI`<br />**Flujo de trabajo secundario:** `CI`<br />**Demuestra:** el gráfico completo de CI manual respecto a la referencia de destino, incluidas las vías de Node en Linux, los fragmentos de plugins incluidos, los fragmentos de contratos de plugins y canales, la compatibilidad con Node 22, `check-*`, `check-additional-*`, las comprobaciones básicas de artefactos compilados, las comprobaciones de documentación, las Skills de Python, Windows, macOS, la internacionalización de la interfaz de control y Android mediante el marco general.<br />**Repetición:** `rerun_group=ci`.                                                                                          |
| Versión preliminar de plugins       | **Trabajo:** `Run plugin prerelease validation`<br />**Flujo de trabajo secundario:** `Plugin Prerelease`<br />**Demuestra:** comprobaciones estáticas de plugins exclusivas de la versión, cobertura agéntica de plugins, fragmentos de lotes completos de plugins, vías de Docker para la versión preliminar de plugins y un artefacto `plugin-inspector-advisory` no bloqueante para el triaje de compatibilidad.<br />**Repetición:** `rerun_group=plugin-prerelease`.                                                                                                                                                          |
| Comprobaciones de la versión          | **Trabajo:** `Run release/live/Docker/QA validation`<br />**Flujo de trabajo secundario:** `OpenClaw Release Checks`<br />**Demuestra:** comprobación básica de instalación, comprobaciones de paquetes entre sistemas operativos, aceptación del paquete, paridad de QA Lab, Matrix y Telegram en entorno real, además de vías de aviso controladas para Discord, WhatsApp y Slack. Los perfiles estable y completo también ejecutan conjuntos exhaustivos en entorno real/E2E y fragmentos de la ruta de versión de Docker; la versión beta puede incluirlos mediante `run_release_soak=true`.<br />**Repetición:** `rerun_group=release-checks` o un identificador más limitado de comprobaciones de la versión.              |
| Paquete de Telegram        | **Trabajo:** `Run package Telegram E2E`<br />**Flujo de trabajo secundario:** `NPM Telegram Beta E2E`<br />**Demuestra:** una prueba E2E específica de Telegram con el paquete publicado cuando se establece `release_package_spec` o `npm_telegram_package_spec`. La validación completa del candidato utiliza en su lugar la prueba E2E canónica de Telegram de aceptación del paquete.<br />**Repetición:** `rerun_group=npm-telegram` con `release_package_spec` o `npm_telegram_package_spec`.                                                                                                              |
| Rendimiento del producto     | **Trabajo:** `Run product performance evidence`<br />**Flujo de trabajo secundario:** `OpenClaw Performance`<br />**Demuestra:** ejecución de rendimiento del perfil de versión (`profile=release`, `repeat=3`, `fail_on_regression=true`, `publish_reports=false`) respecto al SHA de destino. La salida de Kova permanece en los artefactos del flujo de trabajo y el flujo secundario debe demostrar que se omitió su publicador de informes. Obligatorio (bloqueante) únicamente para `rerun_group=all` o `rerun_group=performance`; no es obligatorio para grupos de repetición más limitados.<br />**Repetición:** `rerun_group=performance`. |
| Verificador general       | **Trabajo:** `Verify full validation`<br />**Flujo de trabajo secundario:** ninguno<br />**Demuestra:** vuelve a comprobar las conclusiones registradas de las ejecuciones secundarias y añade tablas de los trabajos más lentos de los flujos de trabajo secundarios.<br />**Repetición:** vuelva a ejecutar únicamente este trabajo después de volver a ejecutar correctamente un flujo secundario con errores.                                                                                                                                                                                                                                                                 |

El marco general siempre inicia el rendimiento del producto en modo de solo artefactos.
`OpenClaw Performance` permite publicar informes únicamente en ejecuciones programadas o en un
inicio manual que establezca explícitamente `publish_reports=true`. La protección de solo
artefactos debe completarse correctamente, lo que demuestra que el trabajo del publicador permaneció omitido.
Las pruebas nuevas y reutilizadas registran
`controls.performanceReportPublication=artifact-only`; el verificador y el selector de reutilización
rechazan las pruebas que no contengan la demostración normalizada correspondiente del flujo secundario de rendimiento.

El verificador carga el manifiesto canónico como
`full-release-validation-<run-id>-<run-attempt>`. Las herramientas de evidencia validan
el ID del artefacto, el resumen, la ejecución productora y el intento antes de descargar ese
ID de artefacto exacto. Limitan el ZIP descargado, verifican sus bytes con respecto al resumen
`sha256:` de REST y transmiten la única entrada de manifiesto acotada permitida sin
extraer el archivo. Se mantiene temporalmente un alias de nombre estable para los consumidores
de publicación más antiguos. El verificador siempre prefiere el artefacto calificado por intento;
como transición, acepta el nombre estable únicamente para un productor de manifiesto v2 del intento 1.
Rechaza ese nombre heredado para intentos posteriores y para el manifiesto v3.

Para `ref=main` con `rerun_group=all`, para referencias `release/*` y para referencias
alfa de Tideclaw, una ejecución general más reciente sustituye a una anterior con la misma referencia y
el mismo grupo de reejecución. Cuando se cancela la ejecución principal, su monitor cancela cualquier flujo de trabajo
secundario que ya haya lanzado. Las ejecuciones de validación de etiquetas y SHA fijados no se
cancelan entre sí.

## Etapas de las comprobaciones de la versión

`OpenClaw Release Checks` es el flujo de trabajo secundario más grande. Resuelve el objetivo
una vez y valida el artefacto de paquete compartido de la ejecución general cuando está disponible. Un
lanzamiento directo o específico prepara su propio artefacto `release-package-under-test`
cuando lo necesitan las etapas relacionadas con paquetes o Docker.

| Etapa                    | Detalles                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Objetivo de la versión           | **Trabajo:** `Resolve target ref`<br />**Flujo de trabajo subyacente:** ninguno<br />**Pruebas:** referencia seleccionada, SHA esperado opcional, perfil, grupo de reejecución y filtro específico del conjunto de pruebas en vivo.<br />**Reejecución:** `rerun_group=release-checks`.                                                                                                                                                                                                                                                                                                                                                             |
| Artefacto de paquete         | **Trabajo:** `Prepare release package artifact`<br />**Flujo de trabajo subyacente:** ninguno<br />**Pruebas:** valida la tupla inmutable del paquete de la ejecución general o empaqueta un tarball candidato para un lanzamiento directo o específico de Comprobaciones de la versión y, después, lo pone a disposición de las comprobaciones posteriores relacionadas con paquetes.<br />**Reejecución:** el grupo afectado de paquetes, multiplataforma o en vivo/E2E.                                                                                                                                                                                                                                |
| Prueba de humo de instalación            | **Trabajo:** `Run install smoke`<br />**Flujo de trabajo subyacente:** `Install Smoke`<br />**Pruebas:** ruta completa de instalación con reutilización de la imagen de prueba de humo del Dockerfile raíz, instalación del paquete mediante QR, pruebas de humo de Docker raíz y del Gateway, pruebas del instalador con Docker y prueba de humo del proveedor de imágenes con instalación global mediante Bun.<br />**Reejecución:** `rerun_group=install-smoke`.                                                                                                                                                                                                                                                           |
| Multiplataforma                 | **Trabajo:** `cross_os_release_checks`<br />**Flujo de trabajo subyacente:** `OpenClaw Cross-OS Release Checks (Reusable)`<br />**Pruebas:** carriles de instalación nueva y actualización en Linux, Windows y macOS para el proveedor y el modo seleccionados, mediante el tarball candidato y un paquete de referencia.<br />**Reejecución:** `rerun_group=cross-os`.                                                                                                                                                                                                                                                                 |
| E2E del repositorio y en vivo        | **Trabajo:** `Run repo/live E2E validation`<br />**Flujo de trabajo subyacente:** `OpenClaw Live And E2E Checks (Reusable)`<br />**Pruebas:** E2E del repositorio, caché en vivo, transmisión por websocket de OpenAI, fragmentos de proveedores en vivo nativos y plugins, y bancos de pruebas en vivo de modelos, backends y gateways respaldados por Docker seleccionados mediante `release_profile`.<br />**Ejecuciones:** `run_release_soak=true`, `release_profile=full` o `rerun_group=live-e2e` específico.<br />**Reejecución:** `rerun_group=live-e2e`, opcionalmente con `live_suite_filter`.                                                                                |
| Ruta de versión de Docker      | **Trabajo:** `Run Docker release-path validation`<br />**Flujo de trabajo subyacente:** `OpenClaw Live And E2E Checks (Reusable)`<br />**Pruebas:** fragmentos de Docker de la ruta de versión con respecto al artefacto de paquete compartido.<br />**Ejecuciones:** `run_release_soak=true`, `release_profile=full` o `rerun_group=live-e2e` específico.<br />**Reejecución:** `rerun_group=live-e2e`.                                                                                                                                                                                                                                     |
| Aceptación del paquete       | **Trabajo:** `Run package acceptance`<br />**Flujo de trabajo subyacente:** `Package Acceptance`<br />**Pruebas:** fixtures sin conexión de paquetes de plugins, actualización de plugins, el E2E canónico del paquete de Telegram con OpenAI simulado y comprobaciones de supervivencia a actualizaciones publicadas con el mismo tarball. Las comprobaciones de versión bloqueantes utilizan como referencia predeterminada la última versión publicada; las comprobaciones prolongadas (`run_release_soak=true`) se amplían a las últimas 4 versiones estables de npm más 3 versiones históricas fijadas (`2026.4.23`, `2026.5.2`, `2026.4.15`) y se ejecutan con fixtures de actualización de incidencias notificadas.<br />**Reejecución:** `rerun_group=package`. |
| Cuadro de indicadores de madurez       | **Trabajo:** `Render maturity scorecard release docs`<br />**Flujo de trabajo subyacente:** `maturity-scorecard.yml`<br />**Pruebas:** genera la documentación orientativa del cuadro de indicadores de madurez con respecto a la referencia objetivo. Solo se ejecuta cuando se proporciona `run_maturity_scorecard=true`.<br />**Reejecución:** `rerun_group=qa` con `run_maturity_scorecard=true`.                                                                                                                                                                                                                                                           |
| Paridad de QA                | **Trabajo:** `Run QA Lab parity lane` y `Run QA Lab parity report`<br />**Flujo de trabajo subyacente:** trabajos directos<br />**Pruebas:** paquetes de paridad agéntica del candidato y de referencia y, después, el informe de paridad.<br />**Reejecución:** `rerun_group=qa-parity` o `rerun_group=qa`.                                                                                                                                                                                                                                                                                                                         |
| Paridad del entorno de ejecución de QA        | **Trabajo:** `Verify QA Lab runtime-pair lanes`<br />**Flujo de trabajo subyacente:** trabajo directo<br />**Pruebas:** el carril canónico del núcleo `openclaw`/`codex` (`pnpm openclaw qa suite --runtime-pair openclaw,codex --runtime-pair-lane core`) y, con `run_release_soak=true`, el carril de prueba prolongada. Orientativo: los trabajos de carriles individuales no bloquean el verificador de comprobaciones de la versión.<br />**Reejecución:** `rerun_group=qa-parity` o `rerun_group=qa`.                                                                                                                                                             |
| Cobertura de herramientas del entorno de ejecución de QA | **Trabajo:** `Enforce QA Lab runtime tool coverage`<br />**Flujo de trabajo subyacente:** trabajo directo<br />**Pruebas:** desviación dinámica de herramientas entre `openclaw` y `codex` en el carril canónico de pares de entornos de ejecución del núcleo (`pnpm openclaw qa coverage --tools`), mediante la salida de ese carril. Bloqueante: este trabajo no permite anulación por su carácter orientativo.<br />**Reejecución:** `rerun_group=qa-parity` o `rerun_group=qa`.                                                                                                                                                                                                     |
| Matrix en vivo de QA           | **Trabajo:** `Run QA Live Matrix profile`<br />**Flujo de trabajo subyacente:** flujo de trabajo reutilizable `QA-Lab - All Lanes`<br />**Pruebas:** escenarios YAML con paridad demostrada mediante el adaptador compartido de Matrix en vivo en el entorno `qa-live-shared`.<br />**Reejecución:** `rerun_group=qa-live` o `rerun_group=qa`; use `live_suite_filter=qa-live-matrix` para una reejecución específica de Matrix.                                                                                                                                                                                                                    |
| Telegram en vivo de QA         | **Trabajo:** `Run QA Lab live Telegram lane`<br />**Flujo de trabajo subyacente:** lanzamiento de confianza `OpenClaw Release Telegram QA`<br />**Pruebas:** QA de Telegram en vivo con concesiones de credenciales de CI de Convex.<br />**Reejecución:** `rerun_group=qa-live` o `rerun_group=qa`.                                                                                                                                                                                                                                                                                                                                 |
| Discord en vivo de QA          | **Trabajo:** `Run QA Lab live Discord lane`<br />**Flujo de trabajo subyacente:** trabajo orientativo directo<br />**Pruebas:** QA de Discord en vivo con concesiones de credenciales de CI de Convex cuando `OPENCLAW_RELEASE_QA_DISCORD_LIVE_CI_ENABLED` está habilitado.<br />**Reejecución:** `rerun_group=qa-live` con `live_suite_filter=qa-live-discord`.                                                                                                                                                                                                                                                                            |
| WhatsApp en vivo de QA         | **Trabajo:** `Run QA Lab live WhatsApp lane`<br />**Flujo de trabajo subyacente:** trabajo orientativo directo<br />**Pruebas:** QA de WhatsApp en vivo con concesiones de credenciales de CI de Convex cuando `OPENCLAW_RELEASE_QA_WHATSAPP_LIVE_CI_ENABLED` está habilitado.<br />**Reejecución:** `rerun_group=qa-live` con `live_suite_filter=qa-live-whatsapp`.                                                                                                                                                                                                                                                                        |
| Slack en vivo de QA            | **Trabajo:** `Run QA Lab live Slack lane`<br />**Flujo de trabajo subyacente:** trabajo orientativo directo<br />**Pruebas:** QA de Slack en vivo con concesiones de credenciales de CI de Convex cuando `OPENCLAW_RELEASE_QA_SLACK_LIVE_CI_ENABLED` está habilitado.<br />**Reejecución:** `rerun_group=qa-live` con `live_suite_filter=qa-live-slack`.                                                                                                                                                                                                                                                                                    |
| Verificador de la versión         | **Trabajo:** `Verify release checks`<br />**Flujo de trabajo subyacente:** ninguno<br />**Pruebas:** trabajos obligatorios de comprobación de la versión para el grupo de reejecución seleccionado.<br />**Reejecución:** volver a ejecutar después de que se superen los trabajos secundarios específicos.                                                                                                                                                                                                                                                                                                                                                                                   |

## Fragmentos de la ruta de lanzamiento de Docker

La etapa de la ruta de lanzamiento de Docker ejecuta estos fragmentos cuando `live_suite_filter` está
vacío:

| Fragmento                                                       | Cobertura                                                                                                                                    |
| --------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `core`                                                          | Carriles principales de pruebas de humo de la ruta de lanzamiento de Docker.                                                                 |
| `package-update-openai`                                         | Comportamiento de instalación/actualización del paquete OpenAI, instalación bajo demanda de Codex, seguimiento del progreso en vivo del Plugin Codex y llamadas a herramientas de Chat Completions. |
| `package-update-anthropic`                                      | Comportamiento de instalación y actualización del paquete Anthropic.                                                                         |
| `package-update-core`                                           | Comportamiento de paquetes y actualizaciones independiente del proveedor.                                                                    |
| `plugins-runtime-plugins`                                       | Carriles de ejecución de Plugins que ejercitan el comportamiento de los Plugins.                                                             |
| `plugins-runtime-services`                                      | Carriles de ejecución de Plugins respaldados por servicios y en vivo.                                                                        |
| `plugins-runtime-install-a` a `plugins-runtime-install-h` | Lotes de instalación/ejecución de Plugins divididos para la validación paralela del lanzamiento.                                               |
| `openwebui`                                                     | Prueba de humo de compatibilidad con OpenWebUI aislada en un ejecutor dedicado con un disco de gran capacidad cuando se solicita.              |

Use `docker_lanes=<lane[,lane]>` de forma específica en el flujo de trabajo reutilizable en vivo/E2E cuando
solo haya fallado un carril de Docker. Los artefactos del lanzamiento incluyen comandos de
reejecución por carril con entradas para reutilizar el artefacto del paquete y la imagen cuando estén disponibles.

## Perfiles de lanzamiento

`release_profile` controla principalmente la amplitud de las pruebas en vivo/de proveedores dentro de las comprobaciones de lanzamiento.
No elimina la CI completa normal, la versión preliminar de Plugins, las pruebas de humo de instalación, la aceptación de
paquetes ni QA Lab. Los perfiles estable y completo siempre ejecutan una cobertura exhaustiva
E2E del repositorio/en vivo y de pruebas prolongadas de la ruta de lanzamiento de Docker. El perfil beta puede habilitarla con
`run_release_soak=true`. La aceptación de paquetes proporciona la prueba E2E canónica del paquete en
Telegram para cada candidato completo, por lo que el flujo general no duplica ese
sondeador en vivo.

| Perfil   | Uso previsto                      | Cobertura en vivo/de proveedores incluida                                                                                                                                                                  |
| -------- | --------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `beta`   | Prueba de humo crítica para el lanzamiento más rápida. | Ruta en vivo principal de OpenAI, modelos en vivo de Docker para OpenAI, núcleo del Gateway nativo, perfil nativo del Gateway de OpenAI, Plugin nativo de OpenAI y Gateway en vivo de Docker para OpenAI. |
| `stable` | Perfil predeterminado de aprobación del lanzamiento. | `beta` más pruebas de humo de Anthropic, Google, MiniMax, backend, arnés nativo de pruebas en vivo, backend de CLI en vivo de Docker, enlace ACP de Docker, arnés Codex de Docker, anuncio de subagentes de Docker y un fragmento de pruebas de humo de OpenCode Go. |
| `full`   | Barrido consultivo amplio.       | `stable` más proveedores consultivos, fragmentos de Plugins en vivo y fragmentos multimedia en vivo.                                                                                              |

## Adiciones exclusivas del perfil completo

`stable` omite estas suites y `full` las incluye:

| Área                             | Cobertura exclusiva del perfil completo                                                                                     |
| -------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| Modelos en vivo de Docker        | OpenCode Go, OpenRouter, xAI, Z.ai y Fireworks.                                                                              |
| Gateway en vivo de Docker        | Proveedores consultivos divididos en fragmentos DeepSeek/Fireworks, OpenCode Go/OpenRouter y xAI/Z.ai.                      |
| Perfiles de proveedores del Gateway nativo | Fragmentos completos de Anthropic Opus y Sonnet/Haiku, Fireworks, DeepSeek, fragmentos completos de modelos OpenCode Go, OpenRouter, xAI y Z.ai. |
| Fragmentos de Plugins nativos en vivo | Plugins A-K, L-N, otros O-Z, Moonshot y xAI.                                                                                |
| Fragmentos multimedia nativos en vivo | Audio, música de Google, música de MiniMax y grupos de vídeo A-D.                                                          |

`stable` incluye `native-live-src-gateway-profiles-anthropic-smoke` y
`native-live-src-gateway-profiles-opencode-go-smoke`; `full` utiliza en su lugar los fragmentos más amplios
de modelos Anthropic y OpenCode Go. Las reejecuciones específicas aún pueden usar los
identificadores agregados `native-live-src-gateway-profiles-anthropic` o
`native-live-src-gateway-profiles-opencode-go`.

## Reejecuciones específicas

Use `rerun_group` para evitar repetir entornos de lanzamiento no relacionados:

| Identificador       | Alcance                                                                                         |
| ------------------- | ----------------------------------------------------------------------------------------------- |
| `all`               | Todas las etapas de validación completa del lanzamiento.                                        |
| `ci`                | Solo el flujo secundario manual de CI completa.                                                 |
| `plugin-prerelease` | Solo el flujo secundario de la versión preliminar de Plugins.                                    |
| `release-checks`    | Todas las etapas de comprobaciones de lanzamiento de OpenClaw.                                  |
| `install-smoke`     | Desde las pruebas de humo de instalación hasta las comprobaciones de lanzamiento.               |
| `cross-os`          | Comprobaciones de lanzamiento entre sistemas operativos.                                        |
| `live-e2e`          | Validación E2E del repositorio/en vivo y de la ruta de lanzamiento de Docker.                    |
| `package`           | Aceptación de paquetes.                                                                         |
| `qa`                | Paridad de QA más carriles de QA en vivo.                                                       |
| `qa-parity`         | Solo los carriles y el informe de paridad de QA.                                                 |
| `qa-live`           | Matrix/Telegram de QA en vivo más los carriles condicionados de Discord, WhatsApp y Slack cuando están habilitados. |
| `npm-telegram`      | Prueba E2E de Telegram con el paquete publicado; requiere `release_package_spec` o `npm_telegram_package_spec`. |
| `performance`       | Solo evidencia del rendimiento del producto.                                                    |

Use `live_suite_filter` con `rerun_group=live-e2e` cuando falle una suite en vivo.
Los identificadores de filtro válidos se definen en el flujo de trabajo reutilizable en vivo/E2E, incluidos
`docker-live-models`, `live-gateway-docker`,
`live-gateway-anthropic-docker`, `live-gateway-google-docker`,
`live-gateway-minimax-docker`, `live-gateway-advisory-docker`,
`live-cli-backend-docker`, `live-acp-bind-docker` y
`live-codex-harness-docker`.

Para una reejecución específica de un transporte de QA, establezca `rerun_group=qa-live` y use el
selector canónico `qa-live-matrix`, `qa-live-telegram`, `qa-live-discord`,
`qa-live-whatsapp` o `qa-live-slack`.

El identificador `live-gateway-advisory-docker` es un identificador agregado de reejecución para sus
tres fragmentos de proveedores, por lo que sigue distribuyéndose a todos los trabajos consultivos del Gateway de Docker.

Use `cross_os_suite_filter` con `rerun_group=cross-os` cuando falle un carril
entre sistemas operativos. El filtro acepta un identificador de sistema operativo, un identificador de suite o un par sistema operativo/suite, por
ejemplo, `windows/packaged-upgrade`, `windows` o `packaged-fresh`. Los resúmenes
entre sistemas operativos incluyen tiempos por fase para los carriles de actualización de paquetes, y los comandos
de larga duración imprimen líneas de Heartbeat para que una actualización bloqueada sea visible antes del
tiempo de espera del trabajo.

Los fallos de las comprobaciones de lanzamiento de QA bloquean la validación normal del lanzamiento solo para los carriles seleccionados
de cobertura de herramientas de Matrix, Telegram y el entorno de ejecución de QA. La paridad de QA, la paridad
del entorno de ejecución y los carriles condicionados en vivo de Discord, WhatsApp y Slack son consultivos y
publican artefactos de estado sin bloquear el verificador del lanzamiento. Las ejecuciones alfa de Tideclaw
aún pueden tratar como consultivos los carriles de comprobación del lanzamiento que no afectan a la seguridad de los paquetes. Con
`release_profile=beta`, las suites de proveedores en vivo `Run repo/live E2E validation`
son consultivas: las implementaciones de modelos de terceros cambian durante un lanzamiento, por lo que
beta presenta sus fallos como advertencias, mientras que los perfiles estable y completo
los mantienen como bloqueantes. Cuando
`live_suite_filter` solicita explícitamente un carril de QA en vivo condicionado, como Discord,
WhatsApp o Slack, debe habilitarse la variable de repositorio `OPENCLAW_RELEASE_QA_*_LIVE_CI_ENABLED`
correspondiente; de lo contrario, la captura de entradas falla en lugar de omitir el carril silenciosamente.
Vuelva a ejecutar `rerun_group=qa`, `qa-parity` o `qa-live` cuando
se necesite evidencia de QA actualizada.

## Evidencia que se debe conservar

Conserve el resumen `Full Release Validation` como índice del lanzamiento. Incluye enlaces a
los identificadores de las ejecuciones secundarias y tablas de los trabajos más lentos. En caso de fallos, inspeccione primero el flujo de trabajo
secundario y, después, vuelva a ejecutar el identificador correspondiente más específico de los anteriores.

Para un lanzamiento normal, registre tanto el SHA del código como el SHA del lanzamiento, la política de reutilización
y el conjunto de rutas modificadas, la ejecución principal correcta del SHA del código y la ejecución principal ligera
del SHA del lanzamiento. Para extended-stable, registre la rama canónica, el SHA exacto del lanzamiento,
el identificador y el intento de la nueva ejecución principal, la referencia del flujo de trabajo, cada ejecución secundaria y cualquier
reparación de compatibilidad del destino congelado u omisión intencionada.

Artefactos útiles:

- `release-package-under-test` de `OpenClaw Release Checks`
- Artefactos de la ruta de lanzamiento de Docker en `.artifacts/docker-tests/`
- Artefactos de aceptación de paquetes `package-under-test` y de aceptación de Docker
- Artefactos de comprobación del lanzamiento entre sistemas operativos para cada sistema operativo y suite
- Artefactos de paridad de QA, paridad del entorno de ejecución y los seleccionados de Matrix, Telegram, Discord, WhatsApp
  o Slack

## Archivos de flujo de trabajo

- `.github/workflows/full-release-validation.yml`
- `.github/workflows/openclaw-release-checks.yml`
- `.github/workflows/openclaw-live-and-e2e-checks-reusable.yml`
- `.github/workflows/plugin-prerelease.yml`
- `.github/workflows/install-smoke.yml`
- `.github/workflows/install-smoke-reusable.yml`
- `.github/workflows/openclaw-cross-os-release-checks-reusable.yml`
- `.github/workflows/package-acceptance.yml`
- `.github/workflows/openclaw-performance.yml`
- `.github/workflows/npm-telegram-beta-e2e.yml`
