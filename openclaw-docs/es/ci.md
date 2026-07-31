---
read_when:
    - Necesita comprender por qué una tarea de CI se ejecutó o no se ejecutó.
    - Está depurando una comprobación fallida de GitHub Actions
    - Se está coordinando una ejecución o repetición de la validación de una versión
    - Está cambiando el despacho de ClawSweeper o el reenvío de actividad de GitHub
summary: Grafo de trabajos de CI, controles de alcance, conjuntos de publicación y comandos locales equivalentes
title: Pipeline de CI
x-i18n:
    generated_at: "2026-07-26T04:31:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f9de5b527354f3cc9eed3813e961116f3834c61bd72b29c92f762c46722815df
    source_path: ci.md
    workflow: 16
---

La CI de OpenClaw se ejecuta con los envíos a `main` (las rutas de Markdown y `docs/**` se ignoran
en el desencadenador), en cada pull request que no sea borrador y mediante ejecución manual.
Los envíos canónicos a `main` se ejecutan de uno en uno: el grupo de concurrencia `CI` permite ejecutar
un ciclo de integración completo mientras GitHub conserva únicamente el envío pendiente más reciente.
Las nuevas fusiones sustituyen esa ejecución pendiente en lugar de cancelar el trabajo que ya
haya registrado una matriz de Blacksmith. Los pull requests siguen cancelando los encabezados reemplazados,
y las ejecuciones manuales usan grupos aislados. `preflight` clasifica las diferencias y
desactiva las fases costosas cuando solo han cambiado áreas no relacionadas. Las ejecuciones manuales
de `workflow_dispatch` omiten deliberadamente la delimitación inteligente y distribuyen
el grafo completo para las versiones candidatas y la validación amplia. Las fases de Android siguen
siendo opcionales mediante `include_android` (o la entrada `release_gate`). La cobertura de plugins
exclusiva para versiones reside en el flujo de trabajo independiente
[`Plugin Prerelease`](#plugin-prerelease) y solo se ejecuta desde
[`Full Release Validation`](#full-release-validation) o mediante una
ejecución manual explícita.

## Descripción general del Pipeline

| Trabajo                            | Propósito                                                                                                                                                                                                             | Cuándo se ejecuta                                      |
| ---------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------ |
| `preflight`                        | Detectar los ámbitos modificados y crear el manifiesto de CI; en `main` canónicos relevantes para Node, actualizar y mantener la instantánea de dependencias antes de la distribución                            | Siempre en envíos y pull requests que no sean borrador |
| `security-fast`                    | Detección de claves privadas, auditoría de flujos de trabajo modificados mediante `zizmor` y auditoría del archivo de bloqueo de producción                                                                      | Siempre en envíos y pull requests que no sean borrador |
| `pnpm-store-warmup`                | Preparar la caché de Actions fijada por el archivo de bloqueo para pull requests y ejecuciones manuales sin bloquear los fragmentos de Node para Linux                                                                 | Cuando se seleccionan fases de Node o comprobación de documentación fuera de main |
| `build-artifacts`                  | Compilar `dist/` y la interfaz de control, ejecutar comprobaciones rápidas de la CLI compilada y comprobar la memoria de inicio y los artefactos compilados integrados                                          | Cambios relevantes para Node                           |
| `control-ui-i18n`                  | Verificar los paquetes de configuración regional, metadatos y memoria de traducción generados de la interfaz de control; informativo en ejecuciones automáticas y bloqueante en la CI manual de versiones             | Cambios relevantes para la i18n de la interfaz de control y CI manual |
| `checks-fast-core`                 | Fases rápidas de corrección en Linux: ajuste progresivo del máximo de líneas de la línea base de supresión, componentes integrados + protocolo, iniciador de Bun y tarea rápida de enrutamiento de CI                 | Cambios relevantes para Node                           |
| `qa-smoke-ci-profile`              | Dos partes equilibradas y autocontenidas del conjunto representativo acotado y automático de pruebas rápidas de control de calidad; la cobertura completa de la taxonomía sigue disponible mediante perfiles explícitos de control de calidad | Cambios relevantes para Node                           |
| `checks-fast-contracts-plugins-*`  | Dos fragmentos ponderados de contratos de plugins                                                                                                                                                                     | Cambios relevantes para Node                           |
| `checks-fast-contracts-channels-*` | Dos fragmentos ponderados de contratos de canales                                                                                                                                                                     | Cambios relevantes para Node                           |
| `checks-node-*`                    | Pruebas de Node para los objetivos modificados en pull requests; fragmentos completos del núcleo en `main` y en ejecuciones manuales, de versiones y de reserva amplia                                       | Cambios relevantes para Node                           |
| `check-*`                          | Equivalente fragmentado de la puerta local principal: protecciones, shrinkwrap, metadatos de configuración de canales integrados, tipos de producción, lint, dependencias y tipos de pruebas                         | Cambios relevantes para Node                           |
| `check-additional-*`               | Franjas de comprobación de límites (incluida la desviación de instantáneas de prompts), límites del descriptor de acceso de sesiones, el lector de transcripciones y las transacciones de SQLite, grupos de lint de extensiones, compilación/canario de límites de paquetes y arquitectura de la topología de ejecución | Cambios relevantes para Node                           |
| `checks-node-compat-node22`        | Fase de compilación y comprobación rápida de compatibilidad con Node 22                                                                                                                                                 | Ejecución manual de CI para versiones                  |
| `check-docs`                       | Comprobaciones de formato, lint y enlaces rotos de la documentación                                                                                                                                                    | Cuando cambia la documentación (pull requests y ejecución manual) |
| `native-i18n`                      | Verificar la extracción de fuentes nativas y la seguridad de la localización en pull requests de código fuente; exigir la paridad completa de traducciones y recursos generados por plataforma en pull requests generados y CI manual | Cambios relevantes para la i18n nativa                 |
| `skills-python`                    | Ruff + pytest para Skills respaldadas por Python                                                                                                                                                                      | Cambios relevantes para Skills de Python               |
| `checks-windows`                   | Pruebas de procesos y rutas específicas de Windows, además de regresiones compartidas de especificadores de importación del entorno de ejecución                                                                      | Cambios relevantes para Windows                        |
| `macos-node`                       | Pruebas específicas de TypeScript para macOS: launchd, Homebrew, rutas del entorno de ejecución, scripts de empaquetado y contenedor de grupos de procesos                                                          | Cambios relevantes para macOS                          |
| `macos-swift`                      | Lint y compilación de Swift para la aplicación de macOS, además de pruebas de la aplicación y del paquete compartido OpenClawKit                                                                                       | Cambios relevantes para macOS                          |
| `ios-build`                        | Generación del proyecto de Xcode y compilación de la aplicación de iOS para el simulador                                                                                                                             | Cambios en la aplicación de iOS, el kit compartido de aplicaciones o Swabble |
| `android`                          | Pruebas unitarias de Android para ambas variantes y compilación de un APK de depuración                                                                                                                              | Cambios relevantes para Android                        |
| `openclaw/ci-gate`                 | Agregado final: requiere comprobaciones preliminares y seguridad; solo acepta omisiones en fases posteriores deshabilitadas por el manifiesto                                                                          | Cada ejecución de CI que no sea borrador               |
| `test-performance-agent`           | Flujo de trabajo independiente: optimización diaria de pruebas lentas de Codex después de actividad de confianza                                                                                                       | Éxito de la CI principal o ejecución manual            |
| `openclaw-performance`             | Flujo de trabajo independiente: informes diarios o bajo demanda sobre el rendimiento del entorno de ejecución Kova, con proveedor simulado, perfilado profundo y fases en vivo de GPT 5.6                           | Ejecución programada y manual                          |

Los flujos de trabajo independientes de Periphery exigen que no haya hallazgos de código muerto en las aplicaciones de iOS y macOS. El flujo de trabajo compartido de OpenClawKit analiza ambos consumidores en paralelo y solo informa de una declaración cuando Periphery emite el mismo USR de Swift desde ambas compilaciones. Su contrato de esquema `OpenClawProtocol/GatewayModels.swift` generado se conserva como código perteneciente al generador, en lugar de tratarse como código muerto local de la aplicación.

## Orden de interrupción rápida

1. `preflight` decide qué fases existen. La lógica de `docs-scope` y `changed-scope` corresponde a pasos dentro de este trabajo, no a trabajos independientes. `main` canónico comienza inmediatamente, pero su grupo de concurrencia admite una sola ejecución completa y combina los envíos posteriores en una única ejecución pendiente con el envío más reciente. Los envíos a main relevantes para Node también serializan aquí el único escritor del disco de dependencias y el mantenimiento de su tamaño antes de que los trabajos posteriores puedan montar la clave; Blacksmith puede exponer una confirmación nueva únicamente a una ejecución posterior del flujo de trabajo, por lo que los consumidores de la misma ejecución conservan la alternativa local comprobada mediante marcador.
2. `security-fast`, `check-*`, `check-additional-*`, `check-docs` y `skills-python` fallan rápidamente sin esperar a los trabajos más pesados de la matriz de artefactos y plataformas.
3. `build-artifacts` y las comprobaciones de configuración regional se ejecutan en paralelo con las fases rápidas de Linux. Los pull requests de código fuente de la interfaz de control y las aplicaciones nativas excluyen las instantáneas y los recursos de configuración regional generados; sus flujos de trabajo de actualización serializados reparan y fusionan automáticamente pull requests generados y aislados en segundo plano. La CI del código fuente sigue bloqueando los inventarios de fuentes obsoletos y las llamadas de localización no seguras. Los pull requests generados, la CI manual y la preparación de versiones exigen la paridad completa de las traducciones y los recursos generados por plataforma. Las ramas canónicas `release/YYYY.M.PATCH` pueden incluir reparaciones de configuración regional para la preparación de versiones junto con el resto de la salida generada para la versión.
4. Después se distribuyen las fases más pesadas de plataformas y entornos de ejecución: `checks-fast-core`, `checks-fast-contracts-plugins-*`, `checks-fast-contracts-channels-*`, `checks-node-*`, `checks-windows`, `macos-node`, `macos-swift`, `ios-build` y `android`.
5. `openclaw/ci-gate` espera a todas las fases seleccionadas. Las comprobaciones preliminares y de seguridad deben completarse correctamente; los trabajos posteriores solo pueden omitirse cuando el manifiesto no los haya seleccionado. Una fase seleccionada que falle o se cancele provoca el fallo del agregado.

El coordinador de fusiones puede reutilizar un `openclaw/ci-gate` autenticado y correcto
para el mismo encabezado de pull request durante un máximo de 24 horas. Esto evita reescribir una
rama de colaborador después de cambios no relacionados en `main`. El resultado reutilizable no
sustituye la comprobación independiente y estricta de fusión de prueba, perteneciente a la aplicación, con respecto al `main` actual.
Una ejecución posterior pendiente o fallida no elimina un resultado correcto anterior para
ese encabezado sin cambios durante el periodo de vigencia.

El conjunto de reglas de la rama predeterminada requiere la comprobación `openclaw/ci-gate`, propiedad de GitHub Actions. Los responsables de mantenimiento y administradores del repositorio disponen de una omisión de emergencia auditada, destinada únicamente a integraciones directas firmadas mediante avance rápido; el conjunto de reglas de la organización sigue bloqueando la eliminación y las actualizaciones que no sean de avance rápido. Las integraciones normales de pull requests deben seguir utilizando la puerta de control en lugar de omitir una Pipeline de CI fallida. La comprobación estricta independiente de integración de pruebas, propiedad de la aplicación, sigue vinculando la cabecera al `main` actual.

GitHub puede marcar como `cancelled` los trabajos de pull requests reemplazados cuando se integra una cabecera más reciente. Debe considerarse ruido de la Pipeline de CI, salvo que también falle la ejecución más reciente del mismo pull request. Las ejecuciones canónicas de `main` no se cancelan después de su admisión; cuando llega tráfico de integración, GitHub sustituye únicamente la ejecución pendiente más antigua por la punta más reciente. Los trabajos de matriz utilizan `fail-fast: false`, y `build-artifacts` informa directamente de los fallos del canal integrado, del límite de compatibilidad del núcleo y de la supervisión del Gateway, en lugar de poner en cola pequeños trabajos de verificación. La clave de concurrencia automática de la Pipeline de CI tiene control de versiones (`CI-v7-*`), de modo que un proceso zombi del lado de GitHub en un grupo de cola antiguo no pueda bloquear indefinidamente las ejecuciones más recientes de la rama principal. Las ejecuciones manuales de la suite completa utilizan `CI-manual-v1-*` y no cancelan las ejecuciones en curso. La protección de memoria de inicio de la lista de plugins mantiene un límite de 350 MiB en Linux Blacksmith autohospedado y permite 425 MiB en Linux hospedado por GitHub, cuya referencia de RSS es mayor para la misma CLI compilada.

Se puede utilizar `pnpm ci:timings`, `pnpm ci:timings:recent` o `node scripts/ci-run-timings.mjs <run-id>` para resumir el tiempo transcurrido, el tiempo en cola, los trabajos más lentos, los fallos y la barrera de distribución `pnpm-store-warmup` de GitHub Actions. El trabajo `ci-timings-summary` dentro del flujo de trabajo existe en `ci.yml`, pero actualmente está deshabilitado (`if: false`); en su lugar, debe ejecutarse localmente el asistente de tiempos. Para medir los tiempos de compilación, consulte el paso `Build dist` del trabajo `build-artifacts`: `pnpm build:ci-artifacts` muestra `[build-all] phase timings:` e incluye `ui:build`; el trabajo también carga el artefacto `startup-memory`.

## Contexto y evidencias del pull request

Los pull requests de colaboradores externos ejecutan una puerta de contexto y evidencias del pull request desde
`.github/workflows/real-behavior-proof.yml`. El flujo de trabajo obtiene la revisión de confianza
del flujo de trabajo (`github.workflow_sha`) y evalúa únicamente el cuerpo del pull request;
no ejecuta código de la rama del colaborador.

La puerta se aplica a los autores de pull requests que no sean propietarios,
miembros o colaboradores del repositorio, ni bots. Se supera cuando el cuerpo del pull request contiene
secciones `What Problem This Solves` y `Evidence` redactadas por el autor. Las evidencias pueden consistir en una prueba específica,
un resultado de la Pipeline de CI, una captura de pantalla, una grabación, una salida del terminal, una observación en vivo,
un registro censurado o un enlace a un artefacto. El cuerpo proporciona la intención y una validación útil;
los revisores inspeccionan el código, las pruebas y la Pipeline de CI para evaluar la corrección.

Cuando la comprobación falla, se debe actualizar el cuerpo del pull request en lugar de enviar otro commit de código.

## Alcance y enrutamiento

La lógica de alcance se encuentra en `scripts/ci-changed-scope.mjs` y está cubierta por pruebas unitarias en `src/scripts/ci-changed-scope.test.ts`. La ejecución manual omite la detección de cambios de alcance y hace que el manifiesto de comprobaciones preliminares actúe como si todas las áreas con alcance hubieran cambiado.

Flujos de trabajo de Periphery independientes para iOS y macOS aplican una política de cero hallazgos de código muerto. Cada uno se ejecuta únicamente cuando un pull request que no sea borrador afecta a su alcance de análisis nativo o cuando se ejecuta manualmente.

- **Las modificaciones del flujo de trabajo de la Pipeline de CI** validan el grafo de la Pipeline de CI de Node, el análisis de los flujos de trabajo y el carril de Windows (`ci.yml` lo ejecuta), pero no fuerzan por sí solas compilaciones nativas de iOS, Android o macOS; esos carriles de plataforma siguen limitados a los cambios en el código fuente de la plataforma.
- **Comprobación de coherencia del flujo de trabajo** ejecuta `actionlint`, `zizmor` sobre todos los archivos YAML de flujos de trabajo, la protección de interpolación de acciones compuestas y la protección de marcadores de conflicto. El trabajo `security-fast`, limitado al pull request, también ejecuta `zizmor` sobre los archivos de flujo de trabajo modificados, de modo que los hallazgos de seguridad del flujo de trabajo provoquen un fallo anticipado en el grafo principal de la Pipeline de CI.
- **La documentación en envíos a `main`** se comprueba mediante el flujo de trabajo independiente `Docs`, con el mismo espejo de documentación de ClawHub utilizado por la Pipeline de CI, por lo que los envíos combinados de código y documentación no ponen también en cola el fragmento `check-docs` de la Pipeline de CI. Los pull requests y la Pipeline de CI manual siguen ejecutando `check-docs` desde la Pipeline de CI cuando cambia la documentación.
- **La PTY de la TUI** se ejecuta en el fragmento `checks-node-core-runtime-tui-pty` de Node para Linux cuando hay cambios en la TUI. El fragmento ejecuta `test/vitest/vitest.tui-pty.config.ts` con `OPENCLAW_TUI_PTY_INCLUDE_LOCAL=1`, por lo que cubre tanto el carril determinista de recursos de prueba `TuiBackend` como la prueba de humo `tui --local`, más lenta, que simula únicamente el endpoint externo del modelo.
- **Las modificaciones exclusivas del enrutamiento de la Pipeline de CI, el pequeño conjunto de recursos de pruebas del núcleo que la tarea rápida ejecuta directamente y las modificaciones limitadas de los asistentes de contratos de plugins** utilizan una ruta de manifiesto rápida exclusiva de Node: `preflight`, `security-fast` y únicamente los carriles rápidos afectados por el cambio: una sola tarea de enrutamiento de la Pipeline de CI `checks-fast-core`, los dos fragmentos de contratos de plugins o ambos. Esa ruta omite los artefactos de compilación, la compatibilidad con Node 22, los contratos de canales, todos los fragmentos del núcleo, los fragmentos de plugins integrados y las matrices de protección adicionales.
- **Las comprobaciones de Node en Windows** se limitan a los adaptadores de procesos y rutas específicos de Windows, los asistentes de ejecución de npm/pnpm/UI, la configuración del gestor de paquetes y las superficies del flujo de trabajo de la Pipeline de CI que ejecutan ese carril; los cambios no relacionados en el código fuente, los plugins, las pruebas de humo de instalación y los cambios exclusivos de pruebas permanecen en los carriles de Node para Linux.

Las familias de pruebas de Node más lentas se dividen o equilibran para que cada trabajo sea pequeño sin reservar ejecutores en exceso:

- Los contratos de Plugin y los contratos de canal se ejecutan cada uno como dos fragmentos ponderados respaldados por Blacksmith, con el mecanismo alternativo estándar del runner de GitHub.
- Las vías rápidas/de soporte de pruebas unitarias del núcleo se ejecutan por separado; la infraestructura de ejecución del núcleo se divide en fragmentos de dominio de procesos, compartidos, hooks, secretos y tres de Cron.
- La respuesta automática se ejecuta mediante workers equilibrados, con el subárbol de respuestas dividido en fragmentos de ejecutor de agentes, comandos, despacho, sesión y enrutamiento de estado.
- Las configuraciones del Gateway/servidor agéntico (plano de control) se dividen en vías de chat, autenticación, modelo, HTTP/Plugin, ejecución e inicio, en lugar de esperar los artefactos compilados.
- La Pipeline de CI normal empaqueta únicamente los fragmentos aislados de infraestructura con patrones de inclusión en lotes deterministas de un máximo de 64 archivos de prueba, lo que reduce la matriz de Node sin combinar los conjuntos no aislados de comandos/Cron, agents-core con estado o Gateway/servidor. Los conjuntos fijos pesados permanecen en 8 vCPU, mientras que las vías agrupadas y de menor peso usan 4 vCPU.
- Los pull requests del repositorio canónico reutilizan el resolutor de pruebas modificadas con el diff sintético del árbol fusionado. Los cambios precisos ejecutan un trabajo dirigido de Node; cada archivo de prueba seleccionado obtiene su propio proceso para mantener intacto el aislamiento de los conjuntos con estado. El planificador combina las pruebas hermanas con los dependientes del grafo de importaciones y recurre al plan compacto existente de conjunto completo de 14 trabajos para cambios en paquetes del workspace, paquetes/archivo de bloqueo, arnés compartido, configuración dividida, archivos renombrados o eliminados, cambios en contratos públicos de extensiones, pruebas con configuración especial de fragmentos, destinos resueltos parcialmente o vacíos, planes de rutas o destinos sobredimensionados y errores del planificador. Los planes dirigidos siempre conservan la comprobación completa del límite de artefactos compilados porque sus escáneres del repositorio no pueden derivarse de las importaciones. Las ejecuciones `main` usan el mismo conjunto compacto completo: los eventos de push intermedios pendientes pueden agruparse, por lo que la ejecución superviviente más reciente debe validar todo el árbol de integración, no solo el diff de su último push individual. Los despachos manuales y las comprobaciones de lanzamiento conservan la matriz completa con nombres por fragmento.
- La matriz completa de Node admite primero las herramientas en serie sistemáticamente lentas, los fragmentos de comandos de respuesta automática y el escritor amplio de caché de core-fast. Esto mantiene el límite de 28 trabajos e impide que el trabajo de la ruta crítica y la semilla de transformación de la siguiente ejecución se desplacen a una oleada posterior.
- Las pruebas generales de navegador, QA, medios y plugins diversos usan sus configuraciones de Vitest dedicadas en lugar del contenedor común compartido de plugins. Los fragmentos con patrones de inclusión registran entradas de tiempo mediante el nombre del fragmento de la Pipeline de CI, para que `.artifacts/vitest-shard-timings.json` pueda distinguir una configuración completa de un fragmento filtrado.
- Los trabajos de fragmentos de Node en Linux conservan la caché experimental de módulos del sistema de archivos de Vitest mediante la API de caché de Actions original, que Blacksmith acelera de forma transparente en sus runners. Cada fragmento de la Pipeline de CI es de solo restauración y descomprime la semilla protegida en su propia raíz local del runner; después, el contenedor del fragmento proporciona subdirectorios activos independientes a los procesos simultáneos de Vitest. Solo el calentador diario que no se cancela o el que se despacha explícitamente guarda un nuevo archivo inmutable, por lo que los pull requests no pueden publicar transformaciones ni generar familias de caché por PR. Una huella digital de las entradas de transformación elimina las generaciones incompatibles de archivos de bloqueo, paquetes, tsconfig y configuraciones de Vitest. El escritor protegido examina y reduce su caché al 75% después de que supere los 2 GiB. Vitest calcula hashes del id del módulo, el contenido fuente, el entorno y la configuración de transformación resuelta, por lo que los cambios parciales ordinarios del código fuente mantienen calientes las entradas sin cambios, mientras que los módulos modificados producen fallos de caché de forma segura. Los prefijos generales de restauración enlazan las ejecuciones de los flujos de trabajo; la LRU normal de la caché de Actions y la expulsión por inactividad limitan los archivos inmutables antiguos.
- Los trabajos de confianza de Node en Linux también vinculan el almacén de pnpm y `node_modules` desde un disco de dependencias protegido por cada línea de Node compatible. Los manifiestos de paquetes, la configuración de instalación, la plataforma del runner y el parche exacto de Node quedan fuera de la clave del disco; una huella digital exacta de la ejecución y de las entradas de instalación determina si un trabajo reutiliza el árbol o reinstala y actualiza el mismo disco. Los manifiestos se canonizan antes de calcular el hash. Los hooks raíz directos auditados conservan únicamente los scripts del ciclo de vida de instalación de pnpm, por lo que los cambios de formato y de scripts ordinarios de prueba/compilación mantienen caliente el árbol de dependencias; las variaciones no auditadas de los hooks del ciclo de vida producen un cierre seguro hasta que sus entradas fuente se incorporan al contrato de la huella digital. Los cambios en dependencias, gestor de paquetes, código fuente de hooks y archivo de bloqueo siempre invalidan la instantánea. Una huella digital coincidente es necesaria, pero no suficiente: la configuración también comprueba el archivo del importador y las sumas de comprobación de los manifiestos, y después verifica las dependencias del archivo de bloqueo respaldadas por el registro que conserva postinstall con respecto a los manifiestos de paquetes que Node resuelve desde sus importadores. Si el contenido del importador falta o está obsoleto, se recurre a una instalación nueva en lugar de servir el izado raíz. Un pull request cuya instantánea de solo lectura no sea utilizable desvincula el workspace e instala en el almacenamiento local del runner, lo que evita escrituras lentas en un clon que no puede publicar. Las instalaciones en frío persistentes desactivan los reintentos internos de obtención de pnpm y realizan hasta tres intentos completos de instalación acotados desde el almacén que se calienta progresivamente; un tiempo de espera agotado sigue siendo un fallo. Tras una restauración validada por contenido o una instalación con el archivo de bloqueo congelado, la configuración desactiva la comprobación redundante de dependencias previa a la ejecución de pnpm: el repositorio elimina intencionadamente los `node_modules` locales de los plugins, que pnpm considera obsoletos y repara mediante instalaciones implícitas simultáneas no seguras durante la distribución de fragmentos. La comprobación previa canónica de main es el único escritor y mide el almacén en cada actualización; solo ejecuta `pnpm store prune` cuando las versiones retiradas de paquetes hacen que supere los 8 GiB. La publicación de instantáneas de Blacksmith es asíncrona incluso después de que finalice un trabajo escritor, por lo que la primera ejecución tras una clave o huella digital nueva puede continuar en frío; las restauraciones posteriores, validadas por contenido y con un marcador exacto, constituyen la prueba del despliegue. Los trabajos obligatorios de la Pipeline de CI y los pull requests obtienen clones desechables, por lo que los cambios de dependencias no crean discos nuevos, instantáneas competidoras ni un bloqueo de caché capaz de cancelar compilaciones.
- Los trabajos de fragmentos de Node y de artefactos de compilación también restauran la caché de compilación portátil en disco de Node mediante cachés inmutables de Actions. Los espacios de nombres independientes `test` y `build` impiden que sus escritores sustituyan los archivos del otro: el calentador de pruebas programado posee la semilla protegida de pruebas, mientras que `build-artifacts` puede publicar como máximo un archivo protegido de compilación por día UTC desde pushes de confianza de `main`. Los trabajos de PR y las pruebas ordinarias solo leen instantáneas protegidas, por lo que el bytecode de ramas de funcionalidades nunca entra en la semilla compartida y el tráfico de PR no crea archivos de caché. Esto reutiliza el bytecode de V8 para la orquestación cargada por Node, las herramientas de compilación y las dependencias externas entre distintas rutas de checkout, incluso cuando solo cambia una parte del grafo de código fuente. Los procesos secundarios de Vitest desactivan una caché de compilación heredada porque la cobertura puede activarse dentro de configuraciones dinámicas y la cobertura de V8 puede perder precisión en la posición del código fuente cuando los scripts se deserializan desde bytecode.
- El trabajo de artefactos de compilación también conserva las salidas de los pasos `build-all` con huellas digitales de contenido. Las declaraciones del SDK de plugins compiladas por la propia Pipeline de CI calculan el hash de todo el grafo de código fuente TypeScript/JSON propiedad del repositorio, excluyen los directorios instalados y generados y restauran tanto las declaraciones planas como los puentes de paquetes después de que `tsdown` borre `dist`. Los cambios en la documentación, los flujos de trabajo, los plugins y otros elementos externos a ese grafo pueden reutilizar la instantánea de declaraciones; los cambios en el código fuente la vuelven a compilar antes de ejecutar la comprobación de exportaciones.
- Las compilaciones completas de declaraciones dividen `tsdown` en grupos de IA, paquetes del workspace y unificados. Cada grupo almacena en caché solo las declaraciones y, aun así, vuelve a compilar el JavaScript de ejecución antes de restaurarlas. Por tanto, los cambios en el núcleo o en plugins invalidan únicamente el gran grafo unificado, mientras que los cambios en paquetes del workspace invalidan de forma conservadora todos los grupos de declaraciones dependientes. Las compilaciones públicas completas suelen usar una caché inmutable de Actions; las claves generales de restauración sirven como semilla para cambios parciales, las huellas digitales de contenido por grupo rechazan los datos obsoletos y la cuota de caché de GitHub expulsa las generaciones antiguas. En cambio, la vía semanal de Node 22 publica un artefacto de 14 días después de ejecuciones correctas de `main` y solo restaura artefactos cuya identidad inmutable de productor se resuelva a ese flujo de trabajo en `main`, lo que evita la rotación de cuota sin permitir que el código de PR escriba en una caché compartida. Las declaraciones de QA privada nunca se conservan en cachés de Actions porque los espacios de nombres de caché no son límites de confidencialidad.
- `check-additional-*` distribuye la lista complementaria de comprobaciones de límites (`scripts/run-additional-boundary-checks.mjs`) entre un fragmento con uso intensivo de prompts (`check-additional-boundaries-a`, que incluye la comprobación de desviaciones de instantáneas de prompts de Codex) y un fragmento combinado para las franjas restantes (`check-additional-boundaries-bcd`); cada uno ejecuta comprobaciones independientes simultáneamente e imprime los tiempos de cada comprobación. El trabajo de compilación/canario de límites de paquetes permanece agrupado, y la arquitectura de topología de ejecución se ejecuta por separado de la cobertura de observación del Gateway integrada en `build-artifacts`.
- En el runner de compilación autohospedado de 32 vCPU, la observación del Gateway, las pruebas de canales y el fragmento de límites de soporte del núcleo se inician juntos dentro de `build-artifacts` después de que `dist/` y `dist-runtime/` ya estén compilados. Las ejecuciones alternativas alojadas en GitHub mantienen la observación del Gateway en serie para que la contención de pocos núcleos no agote su plazo de disponibilidad.

Una vez admitida, la Pipeline de CI canónica de Linux permite hasta 28 trabajos simultáneos de pruebas de Node y
12 para las vías rápidas/de comprobación más pequeñas; Windows y Android permanecen en dos porque
esos grupos de runners son más reducidos. Los lotes compactos de configuraciones completas se ejecutan con un
tiempo de espera de lote de 120 minutos, mientras que los grupos con patrones de inclusión comparten el mismo
presupuesto acotado de trabajos.

La Pipeline de CI de Android ejecuta tanto `testPlayDebugUnitTest` como `testThirdPartyDebugUnitTest` y después compila el APK de depuración Play. La variante de terceros no tiene un conjunto de código fuente ni un manifiesto independientes; su vía de pruebas unitarias sigue compilando la variante con las marcas BuildConfig de SMS/registro de llamadas, a la vez que evita un trabajo duplicado de empaquetado del APK de depuración en cada push relevante para Android. Cada tarea actual de Gradle tiene un disco persistente protegido; los trabajos de PR usan clones desechables, mientras que las ejecuciones protegidas actualizan en el mismo lugar las entradas de Gradle direccionadas por contenido.

Las claves de discos persistentes de Blacksmith están delimitadas deliberadamente por las dimensiones de ejecución o tarea compatibles, nunca por el número de PR, commit, ejecución, rama o hash de dependencias. Las cachés de transformación y compilación en tiempo de ejecución usan la caché de Actions en lugar de discos persistentes porque los archivos inmutables ofrecen resultados verificables de restauración/guardado y evitan fallos de promoción de instantáneas mutables. Tras una migración de versión de clave persistente, se añaden únicamente las identidades exactas obsoletas de clave, arquitectura y región a `.github/retired-sticky-disks.json`, se despacha `Sticky Disk Cleanup` desde `main` con las mismas dimensiones y confirmación, se verifica la eliminación y después se quitan esas entradas. El flujo de trabajo enruta las identidades ARM a un runner ARM, rechaza las discrepancias de región del runner, usa la acción de eliminación por clave exacta de Blacksmith y nunca elimina las cachés del constructor de Docker ni prefijos comodín. Los archivos de caché de Actions usan la LRU normal y la expulsión por inactividad.

El fragmento `check-dependencies` ejecuta comprobaciones de Knip de dependencias de producción, archivos no utilizados y exportaciones no utilizadas. La comprobación de archivos no utilizados falla cuando un PR añade un archivo nuevo no utilizado y no revisado o deja una entrada obsoleta en la lista de permitidos, al tiempo que conserva las superficies intencionadas de plugins dinámicos, elementos generados, compilación, pruebas en vivo y puentes de paquetes que Knip no puede resolver estáticamente. La comprobación de exportaciones no utilizadas excluye los archivos de soporte de pruebas y falla ante cada exportación de producción no utilizada; los consumidores dinámicos intencionados deben modelarse en `config/knip.config.ts`. Los destinos históricos ejecutan la comprobación de exportaciones cuando la proporcionan y, de lo contrario, conservan su mecanismo alternativo anterior de código muerto.

## Reenvío de actividad de ClawSweeper

`.github/workflows/clawsweeper-dispatch.yml` es el puente del lado de destino que conecta la actividad del repositorio de OpenClaw con ClawSweeper. No obtiene ni ejecuta código no confiable de pull requests. El flujo de trabajo crea un token de GitHub App a partir de `CLAWSWEEPER_APP_PRIVATE_KEY` y, a continuación, envía cargas útiles compactas de `repository_dispatch` a `openclaw/clawsweeper`.

El flujo de trabajo tiene cuatro vías:

- `clawsweeper_item` para solicitudes específicas de revisión de incidencias y pull requests;
- `clawsweeper_comment` para comandos explícitos de ClawSweeper en comentarios de incidencias;
- `clawsweeper_commit_review` para solicitudes de revisión a nivel de commit en envíos `main`;
- `github_activity` para la actividad general de GitHub que el agente ClawSweeper puede inspeccionar.

La vía `github_activity` reenvía únicamente metadatos normalizados: tipo de evento, acción, actor, repositorio, número del elemento, URL, título, estado y extractos breves de comentarios o revisiones cuando estén presentes. Evita intencionadamente reenviar el cuerpo completo del webhook. El flujo de trabajo receptor en `openclaw/clawsweeper` es `.github/workflows/github-activity.yml`, que publica el evento normalizado en el enlace del Gateway de OpenClaw para el agente ClawSweeper.

La actividad general es observación, no entrega predeterminada. El agente ClawSweeper recibe el destino de Discord en su prompt y solo debe publicar en `#clawsweeper` cuando el evento sea sorprendente, requiera alguna acción, implique riesgos o resulte útil para las operaciones. Las aperturas y ediciones rutinarias, la actividad irrelevante de bots, el ruido de webhooks duplicados y el tráfico normal de revisiones deben dar como resultado `NO_REPLY`.

Los títulos, comentarios, cuerpos, textos de revisión, nombres de ramas y mensajes de commit de GitHub deben tratarse como datos no confiables en todo este recorrido. Constituyen datos de entrada para el resumen y la clasificación, no instrucciones para el flujo de trabajo ni para el entorno de ejecución del agente.

## Ejecuciones manuales

Las ejecuciones manuales de CI utilizan el mismo grafo de trabajos que la CI normal, pero activan obligatoriamente todas las vías cuyo ámbito no sea Android: fragmentos de Node en Linux, fragmentos de plugins incluidos, fragmentos de contratos de plugins y canales, compatibilidad con Node 22, `check-*`, `check-additional-*`, comprobaciones rápidas de artefactos compilados, comprobaciones de documentación, Skills de Python, Windows, macOS, compilación de iOS e i18n de la interfaz de control y de la aplicación nativa. Los PR automáticos de código fuente verifican el inventario de extracción nativa y la seguridad de la localización de Android y Apple sin exigir resultados traducidos o generados por la plataforma en el mismo PR. El flujo de trabajo serializado de actualización de configuraciones regionales de aplicaciones nativas vuelve a generar esos artefactos en un único PR aislado y habilita la fusión automática de la cabecera exacta cuando se superan las comprobaciones requeridas. La paridad nativa completa sigue siendo bloqueante para los PR de artefactos generados, la CI manual, la validación completa de versiones y la preparación de versiones. La paridad de configuraciones regionales de la interfaz de control sigue siendo informativa en los PR automáticos y las ejecuciones de `main`, y bloqueante en la CI manual o de versiones. Las ejecuciones manuales independientes de CI solo ejecutan Android con `include_android=true` (la entrada `release_gate` también fuerza Android); el proceso global de versiones completas habilita Android mediante `include_android=true`. Las comprobaciones estáticas de prelanzamiento de plugins, el fragmento exclusivo de versiones `agentic-plugins`, el barrido completo por lotes de extensiones y las vías Docker de prelanzamiento de plugins quedan excluidos de la CI. El conjunto de prelanzamiento de Docker solo se ejecuta cuando `Full Release Validation` inicia el flujo de trabajo independiente `Plugin Prerelease` con la puerta de validación de versiones habilitada.

Las comprobaciones del máximo de líneas de los PR obtienen la referencia base del árbol de fusión sintético extraído y verifican su commit padre de cabecera frente a la cabecera del evento. Las ejecuciones manuales utilizan un grupo de concurrencia único para que otro envío o ejecución de PR en la misma referencia no cancele un conjunto completo de pruebas de una versión candidata. La entrada opcional `target_ref` permite que un invocador de confianza ejecute ese grafo sobre una rama, etiqueta o SHA completo de commit mientras utiliza el archivo de flujo de trabajo de la referencia de ejecución seleccionada; la referencia base del máximo de líneas se compara con la base de fusión del destino respecto a la cabecera de la rama predeterminada resuelta para esa ejecución. La entrada `release_gate` es una alternativa de mantenimiento basada en un SHA exacto para la CI de PR bloqueada por falta de capacidad: exige que `target_ref` sea un SHA completo de commit que coincida con la cabecera de la rama ejecutada y que `pull_request_number` identifique el PR abierto cuyo árbol de fusión se valida.

```bash
gh workflow run ci.yml --ref release/YYYY.M.PATCH
gh workflow run ci.yml --ref main -f target_ref=<branch-or-sha> -f include_android=true
gh workflow run full-release-validation.yml --ref main -f ref=<branch-or-sha>
```

Las ejecuciones de estabilidad extendida del Gateway realizan la comprobación previa de npm, la validación completa de versiones y la publicación npm de plugins desde `extended-stable/YYYY.M.33`; la publicación del núcleo utiliza esos tres
identificadores de ejecución junto con el intento de validación. La evidencia de `release-ci/*` no es válida porque
la publicación vincula todas las ejecuciones con la rama canónica y el SHA de la versión. La etiqueta
publica imágenes del Gateway y únicamente los alias `extended-stable*`; esta ruta omite
el orquestador habitual y sus superficies de ClawHub, aplicaciones nativas, GitHub Release, sitio web
y etiquetas de distribución privadas. Consulte [Publicación mensual de estabilidad extendida del Gateway](/es/reference/RELEASING#monthly-gateway-extended-stable-publication)
para conocer los comandos y el procedimiento de recuperación.

## Ejecutores

| Ejecutor                          | Trabajos                                                                                                                                                                                                                                                                              |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ubuntu-24.04`                  | `security-fast`, ejecución manual de CI y alternativas para repositorios no canónicos, el agregado de comprobación rápida de QA, análisis de seguridad y calidad de CodeQL, comprobación de coherencia de flujos de trabajo, etiquetador, respuesta automática, el flujo de trabajo independiente de documentación y todo el flujo de trabajo de comprobación rápida de instalación                                |
| `blacksmith-4vcpu-ubuntu-2404`  | `preflight`, `pnpm-store-warmup`, `native-i18n`, `checks-fast-core` excepto la CI de comprobación rápida de QA, fragmentos de contratos de plugins y canales, la mayoría de los fragmentos de Node en Linux incluidos o de menor carga, vías `check-*` excepto `check-lint`, fragmentos seleccionados de `check-additional-*`, `check-docs` y `skills-python` |
| `blacksmith-8vcpu-ubuntu-2404`  | Conjuntos pesados conservados de Node en Linux, fragmentos `check-additional-*` con uso intensivo de límites o extensiones y `android`                                                                                                                                                                             |
| `blacksmith-16vcpu-ubuntu-2404` | Fragmentos automáticos de CI de comprobación rápida de QA, `build-artifacts` en CI y Testbox, y `check-lint` (lo bastante sensibles a la CPU como para que 8 vCPU costaran más de lo que ahorraban)                                                                                                                                  |
| `blacksmith-8vcpu-windows-2025` | `checks-windows`                                                                                                                                                                                                                                                                  |
| `blacksmith-6vcpu-macos-15`     | `macos-node` en `openclaw/openclaw`; las bifurcaciones recurren a `macos-15`                                                                                                                                                                                                                |
| `blacksmith-12vcpu-macos-26`    | `macos-swift` y `ios-build` en `openclaw/openclaw`; las bifurcaciones recurren a `macos-26`                                                                                                                                                                                               |

## Presupuesto de registro de ejecutores

El límite actual de registro de ejecutores de GitHub de OpenClaw indica 10,000 registros de ejecutores
autohospedados por cada 5 minutos en `ghx api rate_limit`. Vuelva a comprobar
`actions_runner_registration` antes de cada ajuste, ya que GitHub puede modificar
este límite. El límite se comparte entre todos los registros de ejecutores de Blacksmith de la
organización `openclaw`, por lo que añadir otra instalación de Blacksmith no añade
un nuevo límite.

Las etiquetas de Blacksmith deben tratarse como el recurso escaso para controlar las ráfagas. Los trabajos que
solo enrutan, notifican, resumen, seleccionan fragmentos o ejecutan análisis breves de CodeQL deben
permanecer en ejecutores alojados en GitHub, salvo que tengan necesidades específicas de Blacksmith
demostradas mediante mediciones. Cualquier nueva matriz de Blacksmith, un `max-parallel` mayor o un flujo de trabajo
de alta frecuencia debe mostrar su número de registros en el peor caso y mantener el objetivo
de toda la organización por debajo de aproximadamente el 60% del límite activo. Con el límite actual de
10,000 registros, esto supone un objetivo operativo de 6,000 registros, lo que deja margen para
repositorios concurrentes, reintentos y solapamientos de ráfagas.

El plan de PR basado en destinos modificados reduce la ráfaga habitual de pruebas de Node de 14 registros de Blacksmith a uno. Los PR de riesgo amplio conservan la alternativa compacta de 14 registros, por lo que el peor caso no aumenta.

La CI del repositorio canónico mantiene Blacksmith como ruta predeterminada de ejecutores para las ejecuciones normales de envíos y pull requests. `workflow_dispatch` y las ejecuciones de repositorios no canónicos utilizan ejecutores alojados en GitHub, pero actualmente las ejecuciones canónicas normales no comprueban el estado de las colas de Blacksmith ni recurren automáticamente a etiquetas alojadas en GitHub cuando Blacksmith no está disponible.

## Límites de reducción de superficies

Dos presupuestos que solo permiten reducciones protegen la superficie de configuración. Ambos hacen que la CI falle si aumenta
hasta que el archivo de presupuesto se actualice de forma consciente en el mismo PR, y ambos exigen
reducir el límite cuando una limpieza disminuye el recuento real.

- `config/env-var-count-budget.txt` limita el número de nombres `OPENCLAW_*` distintos
  en el código fuente de producción dentro de `src/`, `packages/` y `extensions/`
  (se excluyen las pruebas y QA Lab). Se comprueba mediante `node scripts/check-env-var-count.mjs`.
  Al eliminar variables de entorno, reduzca el número en el mismo PR. Añadir una es una
  decisión sobre la superficie de configuración: justifíquela en el cuerpo del PR.
- `docs/.generated/config-baseline.counts.json` limita por categoría
  (núcleo/canal/plugin) el número de entradas del esquema `openclaw.json`. Se comprueba mediante
  `pnpm config:docs:check`; vuelva a generarlo con `pnpm config:docs:gen` después de cualquier
  cambio del esquema.

## Equivalentes locales

```bash
pnpm changed:lanes                            # inspeccionar el clasificador local de carriles modificados para origin/main...HEAD
pnpm check:changed                            # comprobación local inteligente: formato/typecheck/lint/guardas modificados por carril de límite
pnpm check                                    # comprobación local rápida: tsgo de producción + lint fragmentado + guardas rápidas en paralelo
pnpm check:test-types
pnpm check:timed                              # la misma comprobación con tiempos por etapa
pnpm build:strict-smoke
pnpm check:architecture
pnpm test:gateway:watch-regression
OPENCLAW_TUI_PTY_INCLUDE_LOCAL=1 node scripts/run-vitest.mjs run --config test/vitest/vitest.tui-pty.config.ts
pnpm test                                     # pruebas de vitest
pnpm test:changed                             # objetivos de Vitest modificados, económicos e inteligentes
pnpm test:ui                                  # conjunto de pruebas unitarias/de navegador de la interfaz de control
pnpm ui:i18n:check                            # paridad generada de configuraciones regionales de la interfaz de control (comprobación de lanzamiento)
pnpm native:i18n:baseline                     # actualizar el inventario de extracción nativa propiedad del código fuente
pnpm native:i18n:verify                       # inventario del código fuente + seguridad de localización de Android/Apple
pnpm native:i18n:check                        # paridad estricta de traducciones/elementos generados por plataforma (comprobación de lanzamiento)
pnpm test:channels
pnpm test:contracts:channels
pnpm check:docs                               # formato de documentación + lint + enlaces rotos
pnpm build                                    # compilar dist cuando importen las comprobaciones de artefactos/smoke de CI
pnpm ios:build                                # generar y compilar el proyecto de la aplicación iOS
pnpm ci:timings                               # resumir la ejecución de CI más reciente de un push a origin/main
pnpm ci:timings:recent                        # comparar ejecuciones recientes correctas de CI en main
node scripts/ci-run-timings.mjs <run-id>      # resumir el tiempo transcurrido, el tiempo en cola y los trabajos más lentos
node scripts/ci-run-timings.mjs --latest-main # ignorar el ruido de incidencias/comentarios y elegir la CI de un push a origin/main
node scripts/ci-run-timings.mjs --recent 10   # comparar ejecuciones recientes correctas de CI en main
pnpm test:perf:groups --full-suite --allow-failures --output .artifacts/test-perf/baseline-before.json
pnpm test:perf:groups:compare .artifacts/test-perf/baseline-before.json .artifacts/test-perf/after-agent.json
pnpm test:startup:memory
pnpm test:extensions:memory -- --json .artifacts/openclaw-performance/source/mock-provider/extension-memory.json
pnpm perf:kova:summary --report .artifacts/kova/reports/mock-provider/report.json --output .artifacts/kova/summary.md
```

## Rendimiento de OpenClaw

`OpenClaw Performance` es el flujo de trabajo de rendimiento del producto/entorno de ejecución. Se ejecuta diariamente en `main` y puede iniciarse manualmente:

```bash
gh workflow run openclaw-performance.yml --ref main -f profile=diagnostic -f repeat=3
gh workflow run openclaw-performance.yml --ref main -f profile=smoke -f repeat=1 -f deep_profile=true -f live_openai_candidate=true
gh workflow run openclaw-performance.yml --ref main -f target_ref=v2026.5.2 -f profile=diagnostic -f repeat=3
```

El inicio manual normalmente evalúa el rendimiento de la referencia del flujo de trabajo. Establezca `target_ref` para evaluar una etiqueta de lanzamiento u otra rama con la implementación actual del flujo de trabajo. Las rutas de informes publicados y los punteros más recientes se indexan mediante la referencia probada, y cada `index.md` registra la referencia/SHA probada, la referencia/SHA del flujo de trabajo, la referencia de Kova, el perfil, el modo de autenticación del carril, el modelo, el número de repeticiones y los filtros de escenarios.

El flujo de trabajo instala OCM desde un lanzamiento fijado y Kova desde `openclaw/Kova` en la entrada fijada `kova_ref`, y luego ejecuta tres carriles:

- `mock-provider`: escenarios de diagnóstico de Kova contra un entorno de ejecución compilado localmente con autenticación falsa determinista compatible con OpenAI.
- `mock-deep-profile`: generación de perfiles de CPU/montículo/trazas para puntos críticos del inicio, Gateway y turnos del agente. Se ejecuta según la programación o, al iniciarse manualmente, con `deep_profile=true`.
- `live-openai-candidate`: un turno real del agente OpenAI `openai/gpt-5.6-luna`, que se omite cuando `OPENAI_API_KEY` no está disponible. Se ejecuta según la programación o, al iniciarse manualmente, con `live_openai_candidate=true`.

El carril del proveedor simulado también ejecuta sondas de código fuente nativas de OpenClaw tras la pasada de Kova: tiempo de arranque y memoria del Gateway para los casos de inicio predeterminado, con canal omitido, con enlace interno y con cincuenta plugins; RSS de importación de plugins incluidos, bucles repetidos de saludo `channel-chat-baseline` con OpenAI simulado, comandos de inicio de la CLI contra el Gateway arrancado y la sonda de rendimiento smoke del estado de SQLite. Cuando está disponible el informe de código fuente del proveedor simulado publicado anteriormente para la referencia probada, el resumen del código fuente compara los valores actuales de RSS y montículo con esa línea base y marca los grandes aumentos de RSS como `watch`. El resumen Markdown de la sonda de código fuente se encuentra en `source/index.md` dentro del paquete de informes, junto al JSON sin procesar.

Cada carril carga su artefacto completo de GitHub, incluidos los paquetes de CPU, montículo, trazas y diagnóstico comprimido. Un trabajo de publicación independiente descarga y valida esos artefactos; después genera un token de corta duración de la aplicación de GitHub ClawSweeper limitado únicamente al contenido de `openclaw/clawgrit-reports` y lo pasa solo al paso de push de Git. Confirma `report.json`, `report.md`, `index.md`, los artefactos de las sondas de código fuente y los metadatos/sumas de comprobación del paquete en `openclaw-performance/<tested-ref>/<run-id>-<attempt>/<lane>/`; el archivo de diagnóstico completo permanece en el artefacto enlazado de Actions. El publicador rechaza cualquier archivo de informe de más de 50 MB antes de intentar un push. El puntero actual de la referencia probada es `openclaw-performance/<tested-ref>/latest-<lane>.json`. Las ejecuciones programadas y los inicios manuales `profile=release` fallan si falla la creación del token de la aplicación o la publicación del informe. En los inicios manuales que no sean de lanzamiento, la publicación sigue siendo informativa y se conservan los artefactos de GitHub cuando falla la autenticación o la publicación. La línea base anterior del código fuente se obtiene anónimamente del repositorio público de informes, por lo que obtenerla correctamente no demuestra la autenticación del publicador.

## Validación completa del lanzamiento

`Full Release Validation` es el flujo de trabajo paraguas manual para «ejecutarlo todo antes del lanzamiento». Acepta una rama, una etiqueta o un SHA de commit completo; inicia el flujo de trabajo manual `CI` con ese objetivo (incluido Android), inicia `Plugin Prerelease` para las pruebas exclusivas del lanzamiento de plugins/paquetes/elementos estáticos/Docker, inicia `OpenClaw Performance` contra el SHA objetivo e inicia `OpenClaw Release Checks` para la prueba smoke de instalación, la aceptación de paquetes, las comprobaciones de paquetes entre sistemas operativos, la paridad de QA Lab, Matrix, Telegram y los carriles sujetos a condiciones de Discord, WhatsApp y Slack (la representación informativa de la tabla de madurez puede habilitarse mediante `run_maturity_scorecard`). Los perfiles estable y completo siempre incluyen cobertura exhaustiva en vivo/E2E y de pruebas prolongadas de la ruta de lanzamiento de Docker; el perfil beta puede habilitarla mediante `run_release_soak=true`. La prueba E2E canónica de Telegram del paquete se ejecuta dentro de la aceptación de paquetes, por lo que un candidato completo no inicia un sondeador en vivo duplicado. Después de publicar, pase `release_package_spec` para reutilizar el paquete npm publicado en las comprobaciones de lanzamiento, la aceptación de paquetes, Docker, las comprobaciones entre sistemas operativos y Telegram sin volver a compilarlo. Use `npm_telegram_package_spec` solo para volver a ejecutar de forma específica Telegram con el paquete publicado. El carril en vivo del paquete del plugin Codex usa de forma predeterminada el mismo estado seleccionado: el `release_package_spec=openclaw@<tag>` publicado deriva `codex_plugin_spec=npm:@openclaw/codex@<tag>`, mientras que las ejecuciones mediante SHA/artefacto empaquetan `extensions/codex` desde la referencia seleccionada. Establezca `codex_plugin_spec` explícitamente para fuentes personalizadas del plugin, como las especificaciones `npm:`, `npm-pack:` o `git:`. Su prueba en vivo del agente envía progreso visible, continúa con lecturas aleatorias del espacio de trabajo y una escritura exacta del artefacto, y luego envía la finalización.

Consulte [Validación completa del lanzamiento](/es/reference/full-release-validation) para ver la
matriz de etapas, los nombres exactos de los trabajos del flujo de trabajo, las diferencias entre perfiles, los artefactos y los
manejadores de repeticiones específicas.

`OpenClaw Release Publish` es el flujo de trabajo manual de lanzamiento que realiza modificaciones. Inicie
las publicaciones beta y estables normales desde `main` de confianza después de que exista la etiqueta
de lanzamiento y de que la comprobación preliminar de npm de OpenClaw haya finalizado correctamente (la comprobación preliminar ejecuta
`pnpm plugins:sync:check` entre sus comprobaciones). La etiqueta sigue seleccionando el
commit exacto del lanzamiento, incluido un commit en `release/YYYY.M.PATCH`; las publicaciones alfa
de Tideclaw siguen usando su rama alfa correspondiente. Requiere el
`preflight_run_id` guardado y un
`full_release_validation_run_id` correcto, así como su
`full_release_validation_run_attempt` exacto; inicia `Plugin NPM Release` para todos
los paquetes de plugins publicables, inicia `Plugin ClawHub Release` para el mismo
SHA de lanzamiento y solo entonces inicia `OpenClaw NPM Release`. La publicación estable también
requiere un `windows_node_tag` exacto; el flujo de trabajo verifica el lanzamiento del código fuente
de Windows y compara sus instaladores x64/ARM64 con la entrada
`windows_node_installer_digests` aprobada para el candidato antes de iniciar cualquier publicación secundaria, y luego promociona
y verifica los mismos resúmenes fijados de los instaladores, junto con el contrato exacto
del recurso complementario y la suma de comprobación, antes de publicar el borrador del lanzamiento de GitHub.
Las reparaciones específicas solo de plugins usan `plugin_publish_scope=selected` con una
lista de paquetes no vacía. Las ejecuciones `all-publishable` solo de plugins requieren las mismas pruebas
inmutables de la comprobación preliminar de npm y de la validación completa del lanzamiento que una publicación del núcleo.

```bash
gh workflow run openclaw-release-publish.yml \
  --ref main \
  -f tag=vYYYY.M.PATCH-beta.N \
  -f preflight_run_id=<successful-openclaw-npm-preflight-run-id> \
  -f full_release_validation_run_id=<successful-full-release-validation-run-id> \
  -f full_release_validation_run_attempt=<successful-full-release-validation-run-attempt> \
  -f npm_dist_tag=beta
```

Para obtener pruebas de un commit fijado en una rama que cambia rápidamente, use el asistente en lugar de
`gh workflow run ... --ref main -f ref=<sha>`:

```bash
pnpm ci:full-release --sha <full-sha>
```

Las referencias de inicio de los flujos de trabajo de GitHub deben ser ramas o etiquetas, no SHA de commits sin procesar. El
asistente envía una rama temporal `release-ci/<sha>-...` en un SHA de flujo de trabajo
`main` de confianza, pasa el SHA objetivo solicitado mediante la entrada `ref` del flujo de trabajo,
reutiliza pruebas estrictas del objetivo exacto cuando están disponibles, verifica que el
`headSha` de cada flujo de trabajo secundario coincida con el SHA del flujo de trabajo de confianza y elimina la rama temporal
cuando finaliza la ejecución. Pase `-f reuse_evidence=false` para forzar una
validación nueva. El verificador paraguas también falla si algún flujo de trabajo secundario se ejecutó con un
SHA de flujo de trabajo diferente.

`release_profile` controla la amplitud de las pruebas en vivo/de proveedores que se pasa a las comprobaciones de lanzamiento. Los
flujos de trabajo manuales de lanzamiento usan `stable` de forma predeterminada; use `full` solo cuando
se desee intencionadamente la matriz informativa amplia de proveedores/medios. Las comprobaciones de lanzamientos
estables y completos siempre ejecutan las pruebas exhaustivas en vivo/E2E y las pruebas prolongadas de la ruta de lanzamiento de Docker;
el perfil beta puede habilitarlas mediante `run_release_soak=true`.

- `beta` conserva los carriles críticos para el lanzamiento más rápidos de OpenAI/núcleo.
- `stable` añade el conjunto estable de proveedores/backends.
- `full` ejecuta la matriz informativa amplia de proveedores/medios.

El flujo paraguas registra los identificadores de las ejecuciones secundarias iniciadas, y el trabajo final `Verify full validation` vuelve a comprobar las conclusiones actuales de dichas ejecuciones y añade tablas de los trabajos más lentos de cada una. Si se vuelve a ejecutar un flujo de trabajo secundario y pasa a ser correcto, vuelva a ejecutar únicamente el trabajo verificador principal para actualizar el resultado del flujo paraguas y el resumen de tiempos.

Para la recuperación, tanto `Full Release Validation` como `OpenClaw Release Checks` aceptan `rerun_group`. Use `all` para una versión candidata, `ci` solo para el proceso secundario normal de CI completa, `plugin-prerelease` solo para el proceso secundario de versión preliminar del plugin, `performance` solo para el proceso secundario de rendimiento de OpenClaw, `release-checks` para todos los procesos secundarios de la versión o un grupo más específico: `install-smoke`, `cross-os`, `live-e2e`, `package`, `qa`, `qa-parity`, `qa-live` o `npm-telegram` en el flujo general. Esto mantiene acotada la repetición de una máquina de versión fallida después de una corrección específica. Para un único carril multiplataforma fallido, combine `rerun_group=cross-os` con `cross_os_suite_filter`, por ejemplo, `windows/packaged-upgrade`; los comandos multiplataforma largos emiten líneas de Heartbeat y los resúmenes de actualización empaquetada incluyen los tiempos de cada fase. Los carriles de QA seleccionados de Matrix y Telegram bloquean la validación normal de la versión, al igual que la puerta de cobertura de herramientas del par de runtimes del núcleo. La paridad de QA, la paridad de runtimes y los carriles en vivo con puerta de Discord, WhatsApp y Slack son informativos.

`OpenClaw Release Checks` utiliza la referencia de flujo de trabajo de confianza para resolver una vez la referencia seleccionada en un archivo tar `release-package-under-test` y después pasa ese artefacto a las comprobaciones multiplataforma y a la aceptación de paquetes, además de al flujo de trabajo de Docker de la ruta de versión en vivo/E2E cuando se ejecuta la cobertura de estabilidad. Esto mantiene coherentes los bytes del paquete entre las máquinas de versión y evita volver a empaquetar el mismo candidato en varios trabajos secundarios. Para el carril en vivo del plugin npm de Codex, las comprobaciones de versión pasan una especificación de plugin publicada coincidente derivada de `release_package_spec`, pasan `codex_plugin_spec` proporcionado por el operador o dejan la entrada en blanco para que el script de Docker empaquete el plugin de Codex del checkout seleccionado.

Las ejecuciones duplicadas de `Full Release Validation` para `ref=main` y `rerun_group=all`
reemplazan al flujo general anterior. El monitor principal cancela cualquier flujo de trabajo secundario que
ya haya despachado cuando se cancela el principal, de modo que la validación más reciente de main
no quede detrás de una ejecución obsoleta de dos horas de comprobación de la versión. La validación de ramas/etiquetas
de versión y los grupos de repetición específicos conservan `cancel-in-progress: false`.

## Fragmentos en vivo y E2E

El proceso secundario en vivo/E2E de la versión conserva una amplia cobertura nativa de `pnpm test:live`, pero la ejecuta como fragmentos con nombre mediante `scripts/test-live-shard.mjs` en lugar de un único trabajo en serie:

- `native-live-src-agents` y `native-live-src-agents-zai-coding`
- `native-live-src-gateway-core`
- trabajos de `native-live-src-gateway-profiles` filtrados por proveedor
- `native-live-src-gateway-backends`
- `native-live-src-infra`
- `native-live-test`
- `native-live-extensions-a-k`
- `native-live-extensions-l-n`
- `native-live-extensions-moonshot`
- `native-live-extensions-openai`
- `native-live-extensions-o-z-other`
- `native-live-extensions-xai`
- fragmentos separados de audio/vídeo multimedia y fragmentos de música filtrados por proveedor

Esto conserva la misma cobertura de archivos y facilita la repetición y el diagnóstico de los fallos lentos de proveedores en vivo. Los nombres de fragmento agregados `native-live-src-gateway`, `native-live-extensions-o-z`, `native-live-extensions-media` y `native-live-extensions-media-music` siguen siendo válidos para repeticiones manuales únicas.

Los fragmentos multimedia nativos en vivo se ejecutan en `ghcr.io/openclaw/openclaw-live-media-runner:ubuntu-24.04`, creado por el flujo de trabajo `Live Media Runner Image`. Esa imagen preinstala `ffmpeg` y `ffprobe`; los trabajos multimedia solo verifican los binarios antes de la configuración. Mantenga los conjuntos de pruebas en vivo respaldados por Docker en ejecutores normales de Blacksmith: los trabajos de contenedor no son el lugar adecuado para iniciar pruebas de Docker anidadas.

Los fragmentos de modelos/backends en vivo respaldados por Docker utilizan una imagen `ghcr.io/openclaw/openclaw-live-test:<sha>-<extensions>` compartida independiente por cada commit seleccionado. El flujo de trabajo de versión en vivo crea y publica esa imagen una sola vez; después, los fragmentos de modelos en vivo de Docker, Gateway dividido por proveedor, backend de CLI, enlace de ACP y arnés de Codex se ejecutan con `OPENCLAW_SKIP_DOCKER_BUILD=1`. Los fragmentos de Gateway en Docker llevan límites `timeout` explícitos a nivel de script, inferiores al tiempo de espera del trabajo del flujo de trabajo, para que un contenedor bloqueado o una ruta de limpieza fallen rápidamente en lugar de consumir todo el presupuesto de comprobación de la versión. Si esos fragmentos reconstruyen por separado el destino completo de Docker desde el código fuente, la ejecución de la versión está mal configurada y desperdiciará tiempo real en compilaciones de imagen duplicadas.

## Aceptación de paquetes

Use `Package Acceptance` cuando la pregunta sea «¿funciona como producto este paquete instalable de OpenClaw?». Es diferente de la CI normal: la CI normal valida el árbol de código fuente, mientras que la aceptación de paquetes valida un único archivo tar mediante el mismo arnés E2E de Docker que utilizan los usuarios después de instalar o actualizar.

### Trabajos

1. `resolve_package` hace checkout de `workflow_ref`, resuelve un candidato de paquete, escribe `.artifacts/docker-e2e-package/openclaw-current.tgz`, escribe `.artifacts/docker-e2e-package/package-candidate.json`, carga ambos como el artefacto `package-under-test` e imprime el origen, la referencia del flujo de trabajo, la referencia del paquete, la versión, el SHA-256 y el perfil en el resumen del paso de GitHub.
2. `package_integrity` descarga el artefacto `package-under-test` y aplica el contrato del archivo tar del paquete público con `scripts/check-openclaw-package-tarball.mjs`.
3. `docker_acceptance` llama a `openclaw-live-and-e2e-checks-reusable.yml` con el SHA de origen resuelto del paquete (con `workflow_ref` como alternativa) y `package_artifact_name=package-under-test`. El flujo de trabajo reutilizable descarga ese artefacto, valida el inventario del archivo tar, prepara imágenes de Docker basadas en el resumen del paquete cuando es necesario y ejecuta los carriles de Docker seleccionados con ese paquete en lugar de empaquetar el checkout del flujo de trabajo. Cuando un perfil selecciona varios `docker_lanes` específicos, el flujo de trabajo reutilizable prepara una sola vez el paquete y las imágenes compartidas y, después, distribuye esos carriles como trabajos de Docker específicos en paralelo con artefactos únicos.
4. `package_telegram` llama opcionalmente a `NPM Telegram Beta E2E`. Se ejecuta cuando `telegram_mode` no es `none` e instala el mismo artefacto `package-under-test` cuando la aceptación de paquetes ha resuelto uno; el despacho independiente de Telegram todavía puede instalar una especificación npm publicada.
5. `summary` hace que el flujo de trabajo falle si fallan la resolución del paquete, la integridad, la aceptación de Docker o el carril opcional de Telegram. La entrada `advisory` rebaja los fallos de aceptación a advertencias para los invocadores informativos.

### Orígenes de candidatos

- `source=npm` solo acepta `openclaw@extended-stable`, `openclaw@beta`, `openclaw@latest` o una versión exacta de OpenClaw, como `openclaw@2026.4.27-beta.2`. Use esta opción para la aceptación de versiones estables extendidas, preliminares o estables publicadas.
- `source=ref` empaqueta una rama, etiqueta o SHA completo de commit `package_ref` de confianza. El resolutor obtiene las ramas/etiquetas de OpenClaw, verifica que el commit seleccionado sea accesible desde el historial de ramas del repositorio o desde una etiqueta de versión, instala las dependencias en un árbol de trabajo desacoplado y lo empaqueta con `scripts/package-openclaw-for-docker.mjs`.
- `source=url` descarga un `.tgz` HTTPS público; se requiere `package_sha256`. Esta ruta rechaza las credenciales en URL, los puertos HTTPS no predeterminados, los nombres de host o las IP resueltas privados, internos o de uso especial, y las redirecciones que queden fuera de la misma política de seguridad pública.
- `source=trusted-url` descarga un `.tgz` HTTPS desde una política de origen de confianza con nombre en `.github/package-trusted-sources.json`; se requieren `package_sha256` y `trusted_source_id`. Use esta opción solo para espejos empresariales propiedad de los mantenedores o repositorios de paquetes privados que necesiten hosts, puertos, prefijos de ruta, hosts de redirección o resolución de red privada configurados. Si la política declara autenticación por token al portador, el flujo de trabajo utiliza el secreto fijo `OPENCLAW_TRUSTED_PACKAGE_TOKEN`; las credenciales incrustadas en la URL siguen rechazándose.
- `source=artifact` descarga un `.tgz` desde `artifact_run_id` y `artifact_name`; `package_sha256` es opcional, pero debe proporcionarse para los artefactos compartidos externamente.

Mantenga `workflow_ref` y `package_ref` separados. `workflow_ref` es el código de confianza del flujo de trabajo/arnés que ejecuta la prueba. `package_ref` es el commit de origen que se empaqueta cuando `source=ref`. Esto permite que el arnés de pruebas actual valide commits de origen de confianza anteriores sin ejecutar la lógica de flujos de trabajo antigua.

### Perfiles de conjuntos de pruebas

- `smoke` — `npm-onboard-channel-agent`, `gateway-network`, `config-reload`
- `package` — `npm-onboard-channel-agent`, `doctor-switch`, `update-channel-switch`, `skill-install`, `update-corrupt-plugin`, `upgrade-survivor`, `published-upgrade-survivor`, `root-managed-vps-upgrade`, `update-restart-auth`, `plugins-offline`, `plugin-update`
- `product` — el conjunto `package` con cobertura `plugins` en vivo en lugar de `plugins-offline`, además de `mcp-channels`, `cron-mcp-cleanup`, `openai-web-search-minimal`, `openwebui`
- `full` — segmentos completos de la ruta de versión de Docker con OpenWebUI
- `custom` — `docker_lanes` exacto; requerido cuando `suite_profile=custom`

El perfil `package` utiliza cobertura de plugins sin conexión para que la validación de paquetes publicados no dependa de la disponibilidad en vivo de ClawHub. El carril opcional de Telegram reutiliza el artefacto `package-under-test` en `NPM Telegram Beta E2E`, y conserva la ruta de especificación npm publicada para los despachos independientes.

Para consultar la política específica de pruebas de actualizaciones y plugins, incluidos los comandos locales,
los carriles de Docker, las entradas de aceptación de paquetes, los valores predeterminados de versión y el diagnóstico de fallos,
consulte [Pruebas de actualizaciones y plugins](/es/help/testing-updates-plugins).

Las comprobaciones de versión llaman a la aceptación de paquetes con `source=artifact`, el artefacto de paquete de versión preparado, `suite_profile=custom`, `docker_lanes='doctor-switch update-channel-switch skill-install update-corrupt-plugin upgrade-survivor published-upgrade-survivor root-managed-vps-upgrade update-restart-auth plugins-offline plugin-update plugin-binding-command-escape'` y `telegram_mode=mock-openai`. Esto mantiene la migración de paquetes, la actualización, la instalación en vivo de Skills de ClawHub, la limpieza de dependencias de plugins obsoletas, la reparación de instalaciones de plugins configurados, las pruebas de plugins sin conexión, la actualización de plugins y la prueba de Telegram en el mismo archivo tar de paquete resuelto. Establezca `release_package_spec` en la validación completa de la versión o en las comprobaciones de versión de OpenClaw después de publicar una beta para ejecutar la misma matriz con el paquete npm distribuido sin reconstruirlo; establezca `package_acceptance_package_spec` solo cuando la aceptación de paquetes necesite un paquete diferente del resto de la validación de la versión. Las comprobaciones multiplataforma de la versión siguen cubriendo la incorporación, el instalador y el comportamiento específico de cada sistema operativo; la validación del producto en cuanto a paquetes y actualizaciones debe comenzar con la aceptación de paquetes.

El carril de Docker `published-upgrade-survivor` valida una base de referencia de paquete publicado por ejecución en la ruta de versión bloqueante. En la aceptación de paquetes, el archivo tar `package-under-test` resuelto siempre es el candidato y `published_upgrade_survivor_baseline` selecciona la base de referencia publicada alternativa, cuyo valor predeterminado es `openclaw@latest`; los comandos de repetición de carriles fallidos conservan esa base de referencia. La validación completa de la versión con `run_release_soak=true` o `release_profile=full` establece `published_upgrade_survivor_baselines='last-stable-4 2026.4.23 2026.5.2 2026.4.15'` y `published_upgrade_survivor_scenarios=reported-issues` para abarcar las cuatro versiones estables más recientes de npm, además de versiones límite fijadas de compatibilidad de plugins y casos de prueba basados en incidencias para la configuración de Feishu, archivos de arranque/persona conservados, instalaciones configuradas de plugins de OpenClaw, rutas de registro con virgulilla y raíces obsoletas de dependencias de plugins heredados. Las selecciones de supervivencia de actualizaciones publicadas con varias bases de referencia se dividen por base en trabajos independientes y específicos de ejecutores de Docker. El flujo de trabajo independiente `Update Migration` utiliza el carril de Docker `update-migration` con bases de referencia `all-since-2026.4.23` y escenarios `plugin-deps-cleanup` cuando se necesita una limpieza exhaustiva de las actualizaciones publicadas, no la amplitud normal de la CI completa de versión. Las ejecuciones agregadas locales pueden pasar especificaciones exactas de paquetes con `OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPECS`, conservar un solo carril con `OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPEC`, como `openclaw@2026.4.15`, o establecer `OPENCLAW_UPGRADE_SURVIVOR_SCENARIOS` para la matriz de escenarios. El carril publicado configura la base de referencia con una receta de comandos `openclaw config set` incorporada, registra los pasos de la receta en `summary.json` y comprueba `/healthz`, `/readyz`, además del estado RPC después de iniciar el Gateway. Los carriles de instalación nueva mediante paquete e instalador de Windows también verifican que un paquete instalado pueda importar una sustitución del control del navegador desde una ruta absoluta de Windows sin procesar. La prueba de humo multiplataforma de turnos del agente de OpenAI usa de forma predeterminada `OPENCLAW_CROSS_OS_OPENAI_MODEL` cuando está establecido y, en caso contrario, `openai/gpt-5.6-luna`, de modo que la prueba de instalación y Gateway utilice el nivel de pruebas GPT-5.6 de menor coste.

### Periodos de compatibilidad heredada

La aceptación de paquetes tiene ventanas acotadas de compatibilidad heredada para paquetes ya publicados. Los paquetes hasta `2026.4.25`, incluido `2026.4.25-beta.*`, pueden usar la ruta de compatibilidad:

- las entradas conocidas de control de calidad privado en `dist/postinstall-inventory.json` pueden apuntar a archivos omitidos del tarball;
- `doctor-switch` puede omitir el subcaso de persistencia `gateway install --wrapper` cuando el paquete no expone esa opción;
- `update-channel-switch` puede eliminar del fixture de git falso derivado del tarball los `patchedDependencies` de pnpm que falten y puede registrar los `update.channel` persistidos que falten;
- las pruebas de humo de plugins pueden leer ubicaciones heredadas de registros de instalación o aceptar que falte la persistencia del registro de instalación del marketplace;
- `plugin-update` puede permitir la migración de metadatos de configuración, pero sigue exigiendo que el registro de instalación y el comportamiento sin reinstalación permanezcan sin cambios.

El paquete `2026.4.26` publicado también puede emitir advertencias por archivos locales de sello de metadatos de compilación que ya se distribuyeron, y los paquetes hasta `2026.5.20` pueden emitir una advertencia en lugar de fallar cuando falta `npm-shrinkwrap.json`. Los paquetes posteriores deben cumplir los contratos modernos; las mismas condiciones producen un fallo en lugar de una advertencia u omisión.

### Ejemplos

```bash
# Validar el paquete beta actual con cobertura de nivel de producto.
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=npm \
  -f package_spec=openclaw@beta \
  -f suite_profile=product \
  -f telegram_mode=mock-openai

# Validar el paquete extended-stable publicado con cobertura de paquete.
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=npm \
  -f package_spec=openclaw@extended-stable \
  -f suite_profile=package \
  -f telegram_mode=mock-openai

# Empaquetar y validar una rama de versión con el arnés actual.
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=ref \
  -f package_ref=release/YYYY.M.PATCH \
  -f suite_profile=package \
  -f telegram_mode=mock-openai

# Validar una URL de tarball. SHA-256 es obligatorio para source=url.
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=url \
  -f package_url=https://example.com/openclaw-current.tgz \
  -f package_sha256=<64-char-sha256> \
  -f suite_profile=smoke

# Validar un tarball de una política de réplica privada de confianza con nombre.
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=trusted-url \
  -f trusted_source_id=enterprise-artifactory \
  -f package_url=https://packages.example.internal:8443/artifactory/openclaw/openclaw-current.tgz \
  -f package_sha256=<64-char-sha256> \
  -f suite_profile=smoke

# Reutilizar un tarball cargado por otra ejecución de Actions.
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=artifact \
  -f artifact_run_id=<run-id> \
  -f artifact_name=package-under-test \
  -f suite_profile=custom \
  -f docker_lanes='install-e2e plugin-update'
```

Al depurar una ejecución fallida de aceptación de paquetes, se debe comenzar por el resumen `resolve_package` para confirmar el origen, la versión y el SHA-256 del paquete. Después, se debe inspeccionar la ejecución secundaria `docker_acceptance` y sus artefactos de Docker: `.artifacts/docker-tests/**/summary.json`, `failures.json`, registros de lanes, tiempos de fases y comandos de reejecución. Es preferible volver a ejecutar el perfil de paquete fallido o las lanes de Docker exactas en lugar de volver a ejecutar la validación completa de la versión.

## Prueba de humo de instalación

El workflow `Install Smoke` ya no se ejecuta en pull requests ni en pushes de `main`. Su contenedor nocturno/manual y la validación de versiones llaman al núcleo de solo lectura `install-smoke-reusable.yml`, y cada ejecución recorre la ruta completa de prueba de humo de instalación en runners alojados en GitHub:

- La imagen de prueba de humo del Dockerfile raíz se compila una vez por SHA de destino, se vincula a la revisión del workflow y al intento del productor en un artefacto inmutable y, después, la cargan la prueba de humo de CLI, la prueba de humo de CLI de eliminación del espacio de trabajo compartido de los agentes, el E2E de red del Gateway del contenedor y la prueba de humo del argumento de compilación del plugin `matrix` incluido. La prueba de humo del plugin verifica la replicación de la instalación de dependencias de ejecución y que el plugin se cargue sin diagnósticos de escape de entrada.
- La instalación del paquete QR y las pruebas de humo de Docker del instalador y la actualización —incluidas las lanes del instalador de Rocky Linux y una lane de actualización con una referencia base de npm `update_baseline_version` configurable— se ejecutan como trabajos separados, para que el trabajo del instalador no tenga que esperar detrás de las pruebas de humo de la imagen raíz.

La lenta prueba de humo del proveedor de imágenes de instalación global de Bun se controla por separado mediante `run_bun_global_install_smoke`. Se ejecuta según la programación nocturna, está activada de forma predeterminada para las llamadas al workflow desde las comprobaciones de versiones y los envíos manuales de `Install Smoke` pueden habilitarla. La CI normal de pull requests sigue ejecutando la lane rápida de regresión del iniciador de Bun para cambios relevantes para Node. Las pruebas de Docker de QR y del instalador conservan sus propios Dockerfiles centrados en la instalación.

## E2E local de Docker

`pnpm test:docker:all` precompila una imagen compartida de pruebas en vivo, empaqueta OpenClaw una vez como tarball de npm y compila dos imágenes `scripts/e2e/Dockerfile` compartidas:

- un runner básico de Node/Git para las lanes del instalador, actualización y dependencias de plugins;
- una imagen funcional que instala el mismo tarball en `/app` para las lanes de funcionalidad normales.

Las definiciones de las lanes de Docker se encuentran en `scripts/lib/docker-e2e-scenarios.mjs`, la lógica del planificador se encuentra en `scripts/lib/docker-e2e-plan.mjs` y el runner solo ejecuta el plan seleccionado. El programador selecciona la imagen por lane mediante `OPENCLAW_DOCKER_E2E_BARE_IMAGE` y `OPENCLAW_DOCKER_E2E_FUNCTIONAL_IMAGE` y, después, ejecuta las lanes mediante `OPENCLAW_SKIP_DOCKER_BUILD=1`.

### Parámetros ajustables

| Variable                               | Valor predeterminado | Propósito                                                                                       |
| -------------------------------------- | ------- | --------------------------------------------------------------------------------------------- |
| `OPENCLAW_DOCKER_ALL_PARALLELISM`      | 10      | Cantidad de slots del grupo principal para las lanes normales.                                                        |
| `OPENCLAW_DOCKER_ALL_TAIL_PARALLELISM` | 10      | Cantidad de slots del grupo final sensible al proveedor.                                                      |
| `OPENCLAW_DOCKER_ALL_LIVE_LIMIT`       | 9       | Límite de lanes en vivo simultáneas para que los proveedores no apliquen limitación.                                        |
| `OPENCLAW_DOCKER_ALL_NPM_LIMIT`        | 5       | Límite de lanes simultáneas de instalación de npm.                                                              |
| `OPENCLAW_DOCKER_ALL_SERVICE_LIMIT`    | 7       | Límite de lanes simultáneas de varios servicios.                                                            |
| `OPENCLAW_DOCKER_ALL_START_STAGGER_MS` | 2000    | Intervalo entre inicios de lanes para evitar oleadas de creación del daemon de Docker; se debe establecer `0` para no aplicar ningún intervalo.     |
| `OPENCLAW_DOCKER_ALL_LANE_TIMEOUT_MS`  | 7200000 | Tiempo de espera alternativo por lane (120 minutos); las lanes en vivo/finales seleccionadas usan límites más estrictos.           |
| `OPENCLAW_DOCKER_ALL_DRY_RUN`          | sin establecer   | `1` imprime el plan del programador sin ejecutar las lanes.                                          |
| `OPENCLAW_DOCKER_ALL_LANES`            | sin establecer   | Lista de lanes exactas separadas por comas; omite la prueba de humo de limpieza para que los agentes puedan reproducir una lane fallida. |

Una lane más pesada que su límite efectivo todavía puede iniciarse desde un grupo vacío y, después, ejecutarse sola hasta que libere capacidad. El agregado local realiza comprobaciones previas de Docker, elimina los contenedores E2E obsoletos de OpenClaw, emite el estado de las lanes activas, conserva los tiempos de las lanes para ordenarlas de mayor a menor duración y, de forma predeterminada, deja de programar nuevas lanes agrupadas tras el primer fallo.

### Workflow reutilizable en vivo/E2E

El workflow reutilizable en vivo/E2E consulta a `scripts/test-docker-all.mjs --plan-json` qué paquete, tipo de imagen, imagen en vivo, lane y cobertura de credenciales se requieren. Después, `scripts/docker-e2e.mjs` convierte ese plan en salidas y resúmenes de GitHub. Empaqueta OpenClaw mediante `scripts/package-openclaw-for-docker.mjs`, descarga un artefacto de paquete de la ejecución actual o descarga un artefacto de paquete desde `package_artifact_run_id` y, después, valida el inventario del tarball. La ruta predeterminada `no-push-artifact` compila imágenes básicas/funcionales etiquetadas con el resumen del paquete mediante la caché de capas de Docker de Blacksmith, empaqueta los bytes exactos de la imagen en un artefacto inmutable del workflow y hace que cada consumidor verifique y cargue ese artefacto. En su lugar, `existing-only` exige referencias explícitas de GHCR `docker_e2e_bare_image`/`docker_e2e_functional_image` y nunca compila ni publica. Esas descargas del registro usan un tiempo de espera acotado de 180 segundos por intento para que un flujo bloqueado se reintente rápidamente en lugar de consumir la mayor parte de la ruta crítica de CI. Tras una validación programada correcta, `openclaw-scheduled-live-checks.yml` pasa el manifiesto inmutable de imágenes probadas al publicador independiente con permisos de escritura de paquetes; las llamadas de solo lectura de versiones y versiones preliminares nunca atraviesan ese escritor.

### Fragmentos de la ruta de versión

La cobertura de Docker de versiones ejecuta trabajos fragmentados más pequeños mediante `OPENCLAW_SKIP_DOCKER_BUILD=1`, de modo que cada fragmento verifique y cargue únicamente el tipo de imagen respaldado por artefactos que necesita —o lo descargue con la reutilización explícita `existing-only`— y ejecute varias lanes mediante el mismo programador ponderado:

- `OPENCLAW_DOCKER_ALL_PROFILE=release-path`
- `OPENCLAW_DOCKER_ALL_CHUNK=core | package-update-openai | package-update-anthropic | package-update-core | plugins-runtime-plugins | plugins-runtime-services | plugins-runtime-install-a..h | openwebui`

Los fragmentos actuales de Docker para versiones son `core`, `package-update-openai`, `package-update-anthropic`, `package-update-core`, `plugins-runtime-plugins`, `plugins-runtime-services`, desde `plugins-runtime-install-a` hasta `plugins-runtime-install-h` y `openwebui`. `package-update-openai` incluye la lane de paquete del plugin Codex en vivo, que instala el paquete candidato de OpenClaw, instala el plugin Codex desde `codex_plugin_spec` o desde un tarball de la misma referencia con aprobación explícita para instalar la CLI de Codex, ejecuta las comprobaciones previas de la CLI de Codex y turnos del agente en la misma sesión y, después, ejecuta un turno de razonamiento medio sin reintentos que envía el progreso, lee entradas aleatorias del espacio de trabajo, escribe su artefacto exacto y envía la finalización. `plugins-runtime-core`, `plugins-runtime` y `plugins-integrations` siguen siendo alias agregados de plugins/entorno de ejecución. El alias de lane `install-e2e` sigue siendo el alias agregado de reejecución manual para ambas lanes de instaladores de proveedores.

OpenWebUI se ejecuta como un fragmento `openwebui` independiente en un runner de Blacksmith dedicado con disco de gran capacidad siempre que lo solicite la cobertura estable o completa de la ruta de versión, incluso cuando el workflow reutilizable dirige los trabajos compatibles a runners alojados en GitHub. Mantener separada la descarga de la imagen externa evita que la imagen de gran tamaño compita con las imágenes compartidas de paquetes y plugins en `plugins-runtime-services`; los fragmentos agregados heredados de plugins/entorno de ejecución siguen incluyendo OpenWebUI para reejecuciones manuales compatibles. Las lanes de actualización de canales incluidos vuelven a intentarlo una vez ante fallos transitorios de red de npm.

Cada fragmento carga `.artifacts/docker-tests/` con registros de lanes, tiempos, `summary.json`, `failures.json`, tiempos de fases, el JSON del plan del programador, tablas de lanes lentas y comandos de reejecución por lane. La entrada `docker_lanes` del workflow ejecuta las lanes seleccionadas con las imágenes preparadas para esa ejecución en lugar de los trabajos por fragmentos, lo que mantiene la depuración de lanes fallidas limitada a un único trabajo de Docker específico; si una lane seleccionada es una lane de Docker en vivo, el trabajo específico compila localmente la imagen de pruebas en vivo para esa reejecución. El asistente de reejecución valida el SHA de destino seleccionado exacto del artefacto de fallo y el envío manual vuelve a empaquetar esa referencia, porque la tupla interna de paquetes del workflow reutilizable no forma parte del esquema `workflow_dispatch`. Los comandos generados incluyen entradas de imágenes preparadas y `shared_image_policy=existing-only` solo cuando esas entradas están respaldadas por GHCR; se omiten las etiquetas de artefactos locales del runner para que un runner nuevo las vuelva a compilar. Una sustitución explícita del destino descarta las referencias de imágenes de GHCR recuperadas, salvo que el artefacto demuestre que coinciden con la sustitución. También se omiten las referencias de definiciones de workflows generadas por artefactos porque se eliminan las ramas temporales de versiones completas; el envío usa la rama predeterminada del repositorio, salvo que el operador la sustituya explícitamente.

```bash
pnpm test:docker:rerun <run-id>      # descargar artefactos de Docker e imprimir comandos de reejecución específica combinados/por lane
pnpm test:docker:timings <summary>   # resúmenes de lanes lentas y de la ruta crítica de fases
```

El workflow programado en vivo/E2E ejecuta diariamente el conjunto completo de Docker de la ruta de versión y, tras completarse correctamente, invoca al publicador explícito para los artefactos exactos de imágenes probadas.

## Versión preliminar de plugins

`Plugin Prerelease` ofrece una cobertura de producto/paquete más costosa, por lo que es un flujo de trabajo independiente que activa `Full Release Validation` o un operador explícito. Los pull requests normales, los pushes de `main` y las ejecuciones manuales independientes de la CI mantienen desactivada esa suite. Distribuye las pruebas de plugins incluidos entre ocho workers de extensiones; esos trabajos de fragmentos de extensiones ejecutan hasta dos grupos de configuración de plugins a la vez, con un worker de Vitest por grupo y un heap de Node más grande, para que los lotes de plugins con muchas importaciones no creen trabajos de CI adicionales. La ruta de prelanzamiento de Docker exclusiva para lanzamientos (habilitada mediante la entrada `full_release_validation`) agrupa los carriles de Docker seleccionados en grupos de cuatro para evitar reservar decenas de runners para trabajos de uno a tres minutos. El flujo de trabajo también carga un artefacto informativo `plugin-inspector-advisory` desde `@openclaw/plugin-inspector`; los hallazgos del inspector sirven como datos para el triaje y no modifican la puerta de bloqueo de prelanzamiento de plugins.

## QA Lab

QA Lab tiene carriles de CI dedicados fuera del flujo de trabajo principal de alcance inteligente. La paridad agéntica está integrada en los amplios arneses de QA y lanzamiento, no en un flujo de trabajo de PR independiente. Use `Full Release Validation` con `rerun_group=qa-parity` cuando la paridad deba formar parte de una ejecución de validación amplia.

- El flujo de trabajo `QA-Lab - All Lanes` se ejecuta cada noche en `main` y mediante activación manual; distribuye trabajos de paridad simulada y trabajos en vivo de Matrix, Telegram, Discord, WhatsApp y Slack. Los trabajos en vivo usan el entorno `qa-live-shared`; Telegram, Discord, WhatsApp y Slack usan arrendamientos de Convex, mientras que Matrix aprovisiona credenciales locales desechables.

Las comprobaciones de lanzamiento ejecutan carriles de transporte en vivo de Matrix y Telegram con el proveedor simulado determinista y modelos aptos para simulación (`mock-openai/gpt-5.6-luna` y `mock-openai/gpt-5.6-luna-alt`), de modo que el contrato del canal quede aislado de la latencia del modelo en vivo y del inicio normal del plugin del proveedor. El gateway de transporte en vivo desactiva la búsqueda en memoria porque la paridad de QA cubre por separado el comportamiento de la memoria; la conectividad del proveedor está cubierta por las suites independientes de modelo en vivo, proveedor nativo y proveedor de Docker.

Las puertas programadas y de lanzamiento de Matrix usan el host compartido de la suite de QA Lab y el adaptador en vivo con los escenarios de lanzamiento. El valor predeterminado de la CLI y la entrada manual del flujo de trabajo siguen siendo `all`; las activaciones manuales de `all` distribuyen los perfiles `transport`, `media`, `e2ee-smoke`, `e2ee-deep` y `e2ee-cli`, para que la prueba de 93 escenarios se mantenga dentro de los tiempos de espera de cada trabajo. Las activaciones manuales específicas seleccionan `fast`, `release` o `transport` en un solo trabajo.

`OpenClaw Release Checks` también ejecuta los carriles de QA Lab críticos para el lanzamiento antes de su aprobación; su puerta de paridad de QA ejecuta los paquetes candidato y de referencia como trabajos de carriles paralelos y, después, descarga ambos artefactos en un pequeño trabajo de informes para realizar la comparación final de paridad.

Para los PR normales, siga las pruebas de CI/comprobaciones de alcance específico en lugar de tratar la paridad como un estado obligatorio.

## CodeQL

El flujo de trabajo `CodeQL` es intencionadamente un escáner de seguridad limitado de primera pasada, no un análisis completo del repositorio. Las ejecuciones diarias, manuales, por push de `main` y de protección de pull requests que no sean borradores analizan el código de los flujos de trabajo de Actions, además de las superficies de JavaScript/TypeScript de mayor riesgo, mediante consultas de seguridad de alta confianza filtradas por `security-severity` alto/crítico.

La protección de pull requests se mantiene ligera: solo se inicia para cambios en `.github/actions`, `.github/codeql`, `.github/workflows`, `packages`, `scripts`, `src` o rutas de ejecución de plugins incluidos que controlan procesos, y ejecuta la misma matriz de seguridad de alta confianza que el flujo de trabajo programado. CodeQL para Android y macOS queda fuera de los valores predeterminados de los PR.

### Categorías de seguridad

| Categoría                                          | Superficie                                                                                                                             |
| ------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| `/codeql-security-high/core-auth-secrets`         | Línea base de autenticación, secretos, entorno aislado, cron y gateway                                                                                  |
| `/codeql-security-high/channel-runtime-boundary`  | Contratos de implementación de canales principales, además del entorno de ejecución del plugin de canal, el gateway, el SDK de plugins, los secretos y los puntos de contacto de auditoría              |
| `/codeql-security-high/network-ssrf-boundary`     | Superficies principales de SSRF, análisis de IP, protección de red, obtención web y políticas de SSRF del SDK de plugins                                                |
| `/codeql-security-high/mcp-process-tool-boundary` | Servidores MCP, ayudantes de ejecución de procesos, entrega saliente y puertas de ejecución de herramientas del agente                                           |
| `/codeql-security-high/process-exec-boundary`     | Shell local, ayudantes para iniciar procesos, entornos de ejecución de plugins incluidos que controlan subprocesos y código de enlace de scripts de flujos de trabajo                             |
| `/codeql-security-high/plugin-trust-boundary`     | Superficies de confianza de instalación, cargador, manifiesto, registro, instalación mediante el gestor de paquetes, carga de fuentes y contrato de paquetes del SDK de plugins |

### Fragmentos de seguridad específicos de la plataforma

- `CodeQL Android Critical Security` — fragmento programado de seguridad de Android. Compila manualmente la aplicación Android para CodeQL en el runner Linux de Blacksmith más pequeño aceptado por la validación del flujo de trabajo. Se carga en `/codeql-critical-security/android`.
- `CodeQL macOS Critical Security` — fragmento de seguridad semanal/manual de macOS. Compila manualmente la aplicación para macOS para CodeQL en Blacksmith macOS, excluye de los SARIF cargados los resultados de compilación de dependencias y los carga en `/codeql-critical-security/macos`. Se mantiene fuera de los valores predeterminados diarios porque la compilación de macOS domina el tiempo de ejecución incluso cuando no hay problemas.

### Categorías de calidad crítica

`CodeQL Critical Quality` es el fragmento equivalente que no es de seguridad. Ejecuta únicamente consultas de calidad de JavaScript/TypeScript que no sean de seguridad y tengan gravedad de error sobre superficies limitadas de alto valor, en runners Linux alojados por GitHub, para que los análisis de calidad no consuman el presupuesto de registro de runners de Blacksmith. Su protección de pull requests es intencionadamente más reducida que el perfil programado: los PR que no sean borradores ejecutan solo los fragmentos correspondientes a las superficies que modifican, entre trece fragmentos enrutables para PR: `agent-runtime-boundary`, `channel-runtime-boundary`, `config-boundary`, `core-auth-secrets`, `gateway-runtime-boundary`, `mcp-process-runtime-boundary`, `memory-runtime-boundary`, `network-runtime-boundary`, `plugin-boundary`, `plugin-sdk-package-contract`, `plugin-sdk-reply-runtime`, `provider-runtime-boundary` y `session-diagnostics-boundary`. `ui-control-plane` y `web-media-runtime-boundary` quedan fuera de las ejecuciones de PR. Los cambios en la configuración de CodeQL y en el flujo de trabajo de calidad ejecutan el conjunto completo de fragmentos de PR (el fragmento del entorno de ejecución de red se activa según sus propios archivos de configuración de CodeQL y las rutas de código fuente que controlan la red).

La activación manual acepta:

```text
profile=all|agent-runtime-boundary|config-boundary|core-auth-secrets|channel-runtime-boundary|gateway-runtime-boundary|memory-runtime-boundary|mcp-process-runtime-boundary|network-runtime-boundary|plugin-boundary|plugin-sdk-package-contract|plugin-sdk-reply-runtime|provider-runtime-boundary|session-diagnostics-boundary
```

Los perfiles limitados son mecanismos de aprendizaje e iteración para ejecutar un fragmento de calidad de forma aislada.

| Categoría                                                | Superficie                                                                                                                                                           |
| ------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `/codeql-critical-quality/core-auth-secrets`            | Código de los límites de seguridad de autenticación, secretos, entorno aislado, cron y gateway                                                                                                  |
| `/codeql-critical-quality/config-boundary`              | Contratos de esquema, migración, normalización y E/S de la configuración                                                                                                         |
| `/codeql-critical-quality/gateway-runtime-boundary`     | Esquemas del protocolo del gateway y contratos de métodos del servidor                                                                                                              |
| `/codeql-critical-quality/channel-runtime-boundary`     | Contratos de implementación de canales principales y plugins de canal incluidos                                                                                                  |
| `/codeql-critical-quality/agent-runtime-boundary`       | Ejecución de comandos, envío a modelos/proveedores, envío de respuestas automáticas y colas, y contratos del entorno de ejecución del plano de control de ACP                                               |
| `/codeql-critical-quality/mcp-process-runtime-boundary` | Servidores MCP y puentes de herramientas, ayudantes de supervisión de procesos y contratos de entrega saliente                                                                        |
| `/codeql-critical-quality/memory-runtime-boundary`      | SDK del host de memoria, fachadas del entorno de ejecución de memoria, alias de memoria del SDK de plugins, código de enlace para activar el entorno de ejecución de memoria y comandos de diagnóstico de memoria                                    |
| `/codeql-critical-quality/network-runtime-boundary`     | Paquete de políticas de red, entorno de ejecución de sockets sin procesar y captura de proxy, túnel SSH, bloqueo del gateway, socket JSONL y superficies de transporte push                                 |
| `/codeql-critical-quality/session-diagnostics-boundary` | Componentes internos de la cola de respuestas, colas de entrega de sesiones, ayudantes para vinculación/entrega de sesiones salientes, superficies de paquetes de eventos/registros de diagnóstico y contratos de la CLI de diagnóstico de sesiones |
| `/codeql-critical-quality/plugin-sdk-reply-runtime`     | Envío de respuestas entrantes del SDK de plugins, ayudantes de carga útil/fragmentación/entorno de ejecución de respuestas, opciones de respuesta de canales, colas de entrega y ayudantes de vinculación de sesiones/hilos             |
| `/codeql-critical-quality/provider-runtime-boundary`    | Normalización del catálogo de modelos, autenticación y detección de proveedores, registro del entorno de ejecución de proveedores, valores predeterminados/catálogos de proveedores y registros de web/búsqueda/obtención/incrustaciones    |
| `/codeql-critical-quality/ui-control-plane`             | Inicio de la interfaz de control, persistencia local, flujos de control del gateway y contratos del entorno de ejecución del plano de control de tareas                                                          |
| `/codeql-critical-quality/web-media-runtime-boundary`   | Contratos principales del entorno de ejecución para obtención/búsqueda web, E/S multimedia, comprensión multimedia, generación de imágenes y generación multimedia                                                    |
| `/codeql-critical-quality/plugin-boundary`              | Contratos del cargador, registro, superficie pública y puntos de entrada del SDK de plugins                                                                                             |
| `/codeql-critical-quality/plugin-sdk-package-contract`  | Código fuente publicado del SDK de plugins del paquete y ayudantes para contratos de paquetes de plugins                                                                                      |

La calidad se mantiene separada de la seguridad para que los hallazgos de calidad puedan programarse, medirse, desactivarse o ampliarse sin ocultar la señal de seguridad. La ampliación de CodeQL para Swift, Python y plugins incluidos solo debe volver a incorporarse como trabajo posterior de alcance específico o fragmentado después de que los perfiles limitados tengan un tiempo de ejecución y una señal estables.

## Flujos de trabajo de mantenimiento

### Agente de documentación

El flujo de trabajo `Docs Agent` es un carril de mantenimiento de Codex controlado por eventos para mantener la documentación existente alineada con los cambios incorporados recientemente. No tiene una programación independiente: una ejecución de CI correcta causada por un push que no sea de un bot en `main` puede activarlo, y una activación manual puede ejecutarlo directamente. Las invocaciones mediante ejecuciones de flujos de trabajo se omiten cuando `main` ha avanzado o cuando se creó otra ejecución no omitida del agente de documentación durante la última hora. Cuando se ejecuta, revisa el intervalo de commits desde el SHA de origen de la ejecución anterior no omitida del agente de documentación hasta el `main` actual, por lo que una ejecución por hora puede abarcar todos los cambios de la rama principal acumulados desde la última revisión de la documentación.

### Agente de rendimiento de pruebas

El flujo de trabajo `Test Performance Agent` es una vía de mantenimiento de Codex basada en eventos para pruebas lentas. No tiene una programación pura: una ejecución correcta de CI de un push que no sea de un bot en `main` puede activarlo, pero se omite si otra invocación de ejecución de flujo de trabajo ya se ejecutó o está ejecutándose ese día UTC. La ejecución manual omite ese control de actividad diaria. La vía genera un informe de rendimiento agrupado de Vitest para la suite completa, permite que Codex realice únicamente pequeñas correcciones de rendimiento de las pruebas que conserven la cobertura, en lugar de refactorizaciones amplias, vuelve a ejecutar el informe de la suite completa y rechaza los cambios que reduzcan el recuento de pruebas correctas de referencia. El informe agrupado registra el tiempo de reloj por configuración y el RSS máximo en Linux y macOS, por lo que la comparación anterior/posterior muestra las diferencias de memoria de las pruebas junto a las diferencias de duración. Si la referencia tiene pruebas fallidas, Codex solo puede corregir fallos evidentes, y el informe de la suite completa posterior al agente debe completarse correctamente antes de confirmar cualquier cambio. Cuando `main` avanza antes de que llegue el push del bot, la vía reorganiza mediante rebase el parche validado, vuelve a ejecutar `pnpm check:changed` y reintenta el push; los parches obsoletos con conflictos se omiten. Utiliza Ubuntu alojado en GitHub para que la acción de Codex pueda mantener la misma postura de seguridad de eliminación de sudo que el agente de documentación.

### PR duplicados después de la fusión

El flujo de trabajo `Duplicate PRs After Merge` es un flujo de trabajo manual para mantenedores destinado a limpiar duplicados después de la integración. De forma predeterminada, realiza una ejecución de prueba y solo cierra los PR enumerados explícitamente cuando `apply=true`. Antes de modificar GitHub, verifica que el PR integrado esté fusionado y que cada duplicado tenga un issue referenciado compartido o fragmentos modificados que se solapen.

```bash
gh workflow run duplicate-after-merge.yml \
  -f landed_pr=70532 \
  -f duplicate_prs='70530,70592' \
  -f apply=true
```

## Controles locales y enrutamiento de cambios

### Trinquete del recuento de referencia de configuración

`pnpm config:docs:check` rechaza el crecimiento no documentado de la superficie de configuración y las instantáneas de recuentos dañadas u obsoletas. Cuando un cambio de producto revisado añada intencionadamente rutas de esquema, ejecute `pnpm config:docs:gen`, inspeccione las diferencias de recuentos de núcleo/canal/plugin y los archivos SHA-256 generados, y confirme el aumento consciente de la referencia junto con el esquema, la ayuda, las etiquetas, la migración y las pruebas. No edite manualmente el archivo de recuentos para eludir el trinquete.

Los autores de configuraciones también deben asignar niveles a las nuevas hojas para Ajustes. Añada `advanced: false` o
`advanced: true` a la hoja, o coloque la clave debajo de un antecesor cuyo nivel
deban heredar todos los descendientes. Las raíces sin clasificar hacen que falle la prueba
de calidad del esquema con fragmentos para copiar y pegar; las rutas sin antecesor se consideran avanzadas de forma predeterminada.
La instantánea seleccionada de hojas comunes hace visibles en
la revisión los cambios intencionados de nivel.

La lógica local de vías modificadas se encuentra en `scripts/changed-lanes.mjs` y la ejecuta `scripts/check-changed.mjs`. Ese control local es más estricto con los límites arquitectónicos que el ámbito general de la plataforma de CI:

- los cambios de producción del núcleo ejecutan la comprobación de tipos de producción y pruebas del núcleo, además del lint y los controles del núcleo;
- los cambios solo en pruebas del núcleo ejecutan únicamente la comprobación de tipos de pruebas del núcleo, además del lint del núcleo;
- los cambios de producción de extensiones ejecutan la comprobación de tipos de producción y pruebas de extensiones, además del lint de extensiones;
- los cambios solo en pruebas de extensiones ejecutan la comprobación de tipos de pruebas de extensiones, además del lint de extensiones;
- los cambios del SDK público de Plugin o de contratos de plugins amplían el alcance a la comprobación de tipos de extensiones porque estas dependen de esos contratos del núcleo (los barridos de extensiones de Vitest siguen siendo trabajo de pruebas explícito);
- los incrementos de versión que solo afectan a metadatos de versión ejecutan comprobaciones específicas de versión/configuración/dependencias raíz;
- los cambios desconocidos de raíz/configuración aplican un comportamiento seguro y ejecutan todas las vías de comprobación.

El enrutamiento local de pruebas modificadas se encuentra en `scripts/test-projects.test-support.mjs` y es intencionadamente más económico que `check:changed`: las ediciones directas de pruebas ejecutan esas mismas pruebas; las ediciones de código fuente prefieren asignaciones explícitas, después pruebas hermanas y dependientes del grafo de importación. La configuración compartida de entrega en salas de grupo es una de las asignaciones explícitas: los cambios en la configuración de respuestas visibles del grupo, el modo de entrega de respuestas del código fuente o el prompt del sistema de la herramienta de mensajes se enrutan a través de las pruebas principales de respuesta y de las regresiones de entrega de Discord y Slack, para que un cambio predeterminado compartido falle antes del primer push del PR. Use `OPENCLAW_TEST_CHANGED_BROAD=1 pnpm test:changed` solo cuando el cambio afecte al arnés de forma tan amplia que el conjunto asignado económico no sea un indicador fiable.

## Validación en Testbox

Crabbox es el contenedor de cajas remotas propiedad del repositorio para la verificación en Linux por parte de mantenedores. Las sesiones de
agentes mantienen una o unas pocas pruebas específicas y comprobaciones estáticas económicas en local solo para
código fuente de confianza cuando la instalación de dependencias existente está lista. Utilizan Crabbox para suites más grandes y
trabajo de alta intensidad computacional, como compilaciones, comprobaciones de tipos, despliegue del lint,
Docker, vías de paquetes, E2E, verificación en vivo y paridad con CI. La verificación intensiva de mantenedores de confianza
utiliza `blacksmith-testbox` de forma predeterminada, y `.crabbox.yaml` ahora también lo utiliza de forma predeterminada. Su flujo de trabajo
configurado incorpora credenciales del proveedor y del agente, por lo que el código no confiable de colaboradores o
forks debe usar CI de forks sin secretos o Crabbox directo y saneado en AWS.
Las ejecuciones saneadas en AWS establecen `CRABBOX_ENV_ALLOW=CI`, pasan
`--no-hydrate` y usan un `HOME` remoto temporal nuevo; esto evita que la lista de permitidos
`OPENCLAW_*` del repositorio y los perfiles de autenticación existentes lleguen al código no confiable.
Utilizan un arrendamiento recién preparado dedicado a ese código no confiable, nunca un
arrendamiento de confianza o hidratado previamente. Inicie un binario de Crabbox de confianza instalado
desde un checkout limpio y de confianza de `main`, y obtenga solo el PR remoto con
`--fresh-pr`; nunca ejecute localmente el contenedor ni la configuración del checkout no confiable.
Anule `CRABBOX_AWS_INSTANCE_PROFILE` y aplique un fallo seguro a menos que el valor resuelto de
`aws.instanceProfile` esté vacío. Antes de cualquier instalación/prueba, utilice herramientas de confianza
con rutas absolutas para exigir un token de IMDSv2, demostrar que el endpoint de credenciales de IAM
devuelve 404 y comparar el valor remoto de `git rev-parse HEAD` con el SHA completo
de la cabecera del PR revisado. Vincule el arrendamiento a ese SHA, y deténgalo y vuelva a prepararlo si cambia la cabecera.
Cargue el archivo de confianza `scripts/crabbox-untrusted-bootstrap.sh` desde un `main` limpio
junto con `--fresh-pr`; instala versiones fijadas de Node/pnpm, verifica el SHA y
la versión fijada del gestor de paquetes, aísla `HOME`, instala las dependencias y, después, ejecuta la
prueba solicitada.
Anule todas las sustituciones de `CRABBOX_TAILSCALE*`, fuerce `--network public
--tailscale=false`, borre los indicadores de nodo de salida/LAN y exija que `crabbox inspect`
informe de redes públicas sin estado de Tailscale antes de cargar cualquier script.
La capacidad propia de AWS/Hetzner también sigue siendo la alternativa para interrupciones de Blacksmith,
problemas de cuota o pruebas explícitas con capacidad propia.

Los agentes no preparan recursos con antelación para trabajo previsto. Adquiera una Testbox cuando
esté listo el primer comando intensivo, reutilice el id `tbx_...` devuelto para comandos intensivos
posteriores, sincronice el checkout actual en cada ejecución y deténgala antes de la entrega.

Las ejecuciones de Blacksmith respaldadas por Crabbox preparan, reclaman, sincronizan, ejecutan, generan informes y limpian
Testboxes de un solo uso. La comprobación de integridad de sincronización integrada falla rápidamente cuando
`git status --short` en la caja sincronizada muestra al menos 200 eliminaciones de archivos con seguimiento,
lo que detecta la desaparición de archivos raíz como `pnpm-lock.yaml`. Para los PR que
eliminen intencionadamente una gran cantidad de archivos, establezca `CRABBOX_ALLOW_MASS_DELETIONS=1` para el comando remoto.

Crabbox también finaliza una invocación local de la CLI de Blacksmith que permanezca en la
fase de sincronización durante más de cinco minutos sin salida posterior a la sincronización. Establezca
`CRABBOX_BLACKSMITH_SYNC_TIMEOUT_MS=0` para desactivar ese control o use un valor mayor
en milisegundos para diferencias locales inusualmente grandes.

Antes de una primera ejecución, compruebe el contenedor desde la raíz del repositorio:

```bash
pnpm crabbox:run -- --help | sed -n '1,120p'
```

El contenedor del repositorio rechaza un binario de Crabbox obsoleto que no anuncie el proveedor seleccionado, y las ejecuciones respaldadas por Blacksmith requieren Crabbox 0.22.0 o una versión posterior para que el contenedor obtenga el comportamiento actual de sincronización, cola y limpieza de Testbox. En worktrees de Codex o checkouts enlazados/dispersos, evite el script local `pnpm crabbox:run`, porque pnpm puede reconciliar las dependencias antes de que se inicie Crabbox; en su lugar, invoque directamente el contenedor de Node:

```bash
node scripts/crabbox-wrapper.mjs run --provider blacksmith-testbox --timing-json --shell -- "pnpm test <path-or-filter>"
```

Al usar el checkout hermano, vuelva a compilar el binario local ignorado antes de realizar trabajos de medición o verificación:

```bash
version="$(git -C ../crabbox describe --tags --always --dirty | sed 's/^v//')" \
  && go build -C ../crabbox -trimpath -ldflags "-s -w -X github.com/openclaw/crabbox/internal/cli.version=${version}" -o bin/crabbox ./cmd/crabbox
```

El bloque `blacksmith:` de `.crabbox.yaml` ya fija los valores predeterminados de organización, flujo de trabajo, trabajo y referencia, por lo que los indicadores explícitos siguientes son opcionales. Control modificado:

```bash
pnpm crabbox:run -- --provider blacksmith-testbox \
  --blacksmith-org openclaw \
  --blacksmith-workflow .github/workflows/ci-check-testbox.yml \
  --blacksmith-job check \
  --blacksmith-ref main \
  --idle-timeout 90m \
  --ttl 240m \
  --timing-json \
  --shell -- \
  "corepack pnpm check:changed"
```

Nueva ejecución de una prueba específica en Testbox cuando las dependencias locales no estén disponibles o el
objetivo se ramifique:

```bash
pnpm crabbox:run -- --provider blacksmith-testbox \
  --idle-timeout 90m \
  --ttl 240m \
  --timing-json \
  --shell -- \
  "corepack pnpm test <path-or-filter>"
```

Suite completa:

```bash
pnpm crabbox:run -- --provider blacksmith-testbox \
  --idle-timeout 90m \
  --ttl 240m \
  --timing-json \
  --shell -- \
  "corepack pnpm test"
```

Lea el resumen JSON final. Los campos útiles son `provider`, `leaseId`,
`syncDelegated`, `exitCode`, `commandMs` y `totalMs`. Para las ejecuciones delegadas
de Blacksmith Testbox, el código de salida del contenedor de Crabbox y el resumen JSON son el
resultado del comando. La ejecución vinculada de GitHub Actions gestiona la incorporación de credenciales y el mantenimiento activo; puede
finalizar como `cancelled` cuando Testbox se detiene externamente después de que el comando SSH
ya haya terminado. Trátelo como un artefacto de limpieza/estado, salvo que
el valor `exitCode` del contenedor sea distinto de cero o que la salida del comando muestre una prueba fallida.
Las ejecuciones de Crabbox de un solo uso respaldadas por Blacksmith deberían detener Testbox automáticamente;
si una ejecución se interrumpe o la limpieza no está clara, inspeccione las cajas activas y detenga solo
las que haya creado:

```bash
blacksmith testbox list --all
blacksmith testbox status --id <tbx_id>
blacksmith testbox stop --id <tbx_id>
```

Use la reutilización solo cuando necesite intencionadamente varios comandos en la misma caja con credenciales incorporadas:

```bash
node scripts/crabbox-wrapper.mjs run --provider blacksmith-testbox --id <tbx_id> --timing-json --shell -- "corepack pnpm test <path-or-filter>"
pnpm crabbox:stop -- <tbx_id>
```

Reutilice el arrendamiento, no código fuente obsoleto. Omita `--no-sync` para que cada ejecución cargue el
checkout actual; úselo solo para volver a ejecutar intencionadamente un árbol sin cambios y ya sincronizado.
El código no confiable de colaboradores/forks debe usar
`CRABBOX_ENV_ALLOW=CI`, `--provider aws --no-hydrate` y un `HOME` remoto
temporal nuevo para cada comando; instale las dependencias dentro de ese comando
saneado antes de realizar las pruebas. Reutilice únicamente un arrendamiento recién preparado y dedicado al
mismo código no confiable; nunca uno de confianza o hidratado previamente. Nunca
ejecute localmente el contenedor ni la configuración del checkout no confiable: inicie el binario de Crabbox
de confianza instalado desde un `main` limpio y de confianza, y pase `--fresh-pr` en cada
ejecución. Mantenga `CRABBOX_AWS_INSTANCE_PROFILE` sin establecer, rechace un perfil de instancia resuelto
que no esté vacío, exija una prueba remota de IMDS de confianza sin rol y verifique el
SHA de cabecera revisado antes de la instalación/prueba. Vincule el arrendamiento a ese SHA; deténgalo y
vuelva a prepararlo después de cualquier cambio en la cabecera. Si no existe un PR remoto, use CI de forks sin secretos.
Nunca seleccione `hydrate-github` ni el flujo de trabajo de Blacksmith con credenciales
incorporadas para código no confiable.

Si Crabbox es la capa que está averiada, pero Blacksmith funciona, use Blacksmith
directamente solo para diagnósticos como `list`, `status` y la limpieza. Corrija la
ruta de Crabbox antes de considerar una ejecución directa de Blacksmith como verificación de mantenedor.

Si `blacksmith testbox list --all` y `blacksmith testbox status` funcionan, pero los nuevos
calentamientos permanecen `queued` sin IP ni URL de ejecución de Actions después de un par de minutos,
considérelo presión del proveedor Blacksmith, la cola, la facturación o los límites de la organización. Detenga los
identificadores en cola que haya creado, evite iniciar más Testboxes y traslade la prueba a la
ruta de capacidad propia de Crabbox que se indica a continuación mientras alguien comprueba el panel de Blacksmith,
la facturación y los límites de la organización.

Escale a la capacidad propia de Crabbox solo cuando Blacksmith no esté disponible, tenga limitaciones de cuota, carezca del entorno necesario o la capacidad propia sea explícitamente el objetivo:

```bash
CRABBOX_CAPACITY_REGIONS=eu-west-1,eu-west-2,eu-central-1,us-east-1,us-west-2 \
  pnpm crabbox:warmup -- --provider aws --class standard --market on-demand --idle-timeout 90m
pnpm crabbox:hydrate -- --provider aws --id <cbx_id-or-slug>
pnpm crabbox:run -- --provider aws --id <cbx_id-or-slug> --timing-json --shell -- "pnpm check:changed"
pnpm crabbox:stop -- --provider aws <cbx_id-or-slug>
```

Cuando AWS esté bajo presión, evite `class=beast` a menos que la tarea realmente necesite CPU de clase 48xlarge. Una solicitud `beast` comienza con 192 vCPU y es la forma más fácil de alcanzar la cuota regional de EC2 Spot o Standard bajo demanda. El `.crabbox.yaml` propiedad del repositorio utiliza de forma predeterminada `class: standard`, el mercado bajo demanda y `capacity.hints: true`, de modo que los arrendamientos de AWS intermediados muestren la región y el mercado seleccionados, la presión de cuota, la alternativa de Spot y las advertencias de clases con alta presión. Use `fast` para comprobaciones generales más exigentes, `large` solo cuando standard/fast no sean suficientes y `beast` solo para flujos excepcionales que requieran mucha CPU, como conjuntos de pruebas completos o matrices de Docker para todos los plugins, validaciones explícitas de versiones o bloqueos, o perfiles de rendimiento con una gran cantidad de núcleos. No use `beast` para `pnpm check:changed`, pruebas específicas, trabajo exclusivo de documentación, lint o comprobación de tipos ordinarios, reproducciones E2E pequeñas ni la clasificación de interrupciones de Blacksmith. Use `--market on-demand` para el diagnóstico de capacidad, de modo que las fluctuaciones del mercado Spot no se mezclen con la señal.

`.crabbox.yaml` controla los valores predeterminados del proveedor, la sincronización y la hidratación de GitHub Actions. La sincronización de Crabbox nunca transfiere `.git`, por lo que el checkout hidratado de Actions conserva sus propios metadatos remotos de Git en lugar de sincronizar los remotos y almacenes de objetos locales del mantenedor; además, la configuración del repositorio excluye los artefactos locales de ejecución y compilación (como `.artifacts` y los informes de pruebas) que nunca deben transferirse. `.github/workflows/crabbox-hydrate.yml` controla el checkout, la configuración de Node/pnpm, la obtención de `origin/main` y la transferencia del entorno sin secretos para los comandos `crabbox run --id <cbx_id>` de la nube propia.

## Relacionado

- [Descripción general de la instalación](/es/install)
- [Canales de desarrollo](/es/install/development-channels)
