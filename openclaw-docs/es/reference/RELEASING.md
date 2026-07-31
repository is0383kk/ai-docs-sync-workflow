---
doc-schema-version: 1
read_when:
    - Buscando definiciones públicas de canales de lanzamiento
    - Ejecución de la validación de versiones o la aceptación de paquetes
    - Buscando la nomenclatura y la cadencia de las versiones
summary: Canales de lanzamiento, lista de comprobación del operador, entornos de validación, nomenclatura de versiones y cadencia
title: Política de versiones
x-i18n:
    generated_at: "2026-07-26T05:28:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: de2429f039bb42deabdcfe280b7d91afac3bae3dc24714203ab7a67672dcc10c
    source_path: reference/RELEASING.md
    workflow: 16
---

OpenClaw expone cuatro canales de actualización orientados al usuario:

- stable: la versión regular promocionada en npm `latest`
- extended-stable: la línea de mantenimiento `.33+` del último mes completado en
  npm `extended-stable`
- beta: etiquetas de versión preliminar en npm `beta`
- dev: la cabecera móvil de `main`

Extended-stable distribuye el Gateway, los plugins oficiales de npm y las
imágenes de Docker del mes anterior sin modificar los selectores regulares `latest` ni `main`.

Las compilaciones alfa de Tideclaw constituyen una vía interna de versiones preliminares independiente (dist-tag de npm `alpha`), descrita en [Entradas del flujo de trabajo de NPM](#npm-workflow-inputs) y [Cajas de prueba de versiones](#release-test-boxes).

## Nomenclatura de versiones

- Versión mensual extended-stable del Gateway: `YYYY.M.PATCH`, con `PATCH >= 33`, etiqueta de git `vYYYY.M.PATCH`
- Versión final diaria/regular: `YYYY.M.PATCH`, con `PATCH < 33`, etiqueta de git `vYYYY.M.PATCH`
- Versión regular de corrección alternativa: `YYYY.M.PATCH-N`, etiqueta de git `vYYYY.M.PATCH-N`
- Versión preliminar beta: `YYYY.M.PATCH-beta.N`, etiqueta de git `vYYYY.M.PATCH-beta.N`
- Versión preliminar alfa: `YYYY.M.PATCH-alpha.N`, etiqueta de git `vYYYY.M.PATCH-alpha.N`
- Nunca se deben rellenar con ceros el mes ni el parche
- `PATCH` es un número secuencial del ciclo mensual de versiones, no un día del calendario. Las versiones finales regulares y beta avanzan el ciclo actual; las etiquetas exclusivamente alfa nunca consumen ni avanzan el número de parche beta/regular, por lo que deben ignorarse las etiquetas heredadas exclusivamente alfa con números de parche superiores al seleccionar un ciclo beta o regular.
- Las compilaciones alfa/nocturnas usan el siguiente ciclo de parche aún no publicado e incrementan únicamente `alpha.N` en compilaciones repetidas. Cuando ese parche ya tiene una beta, las nuevas compilaciones alfa pasan al parche siguiente.
- Las versiones de npm son inmutables: nunca se debe eliminar, volver a publicar ni reutilizar una etiqueta publicada. Se debe crear el siguiente número de versión preliminar o el siguiente parche mensual.
- `latest` continúa siguiendo la línea npm regular/diaria actual; `beta` es el destino de instalación beta actual
- `extended-stable` representa la distribución compatible del Gateway del mes anterior, comenzando en el parche `33`; el parche `34` y los posteriores son versiones de mantenimiento de esa línea mensual
- Las versiones finales regulares y las correcciones regulares se publican de forma predeterminada en npm `beta`; los operadores de versiones pueden elegir explícitamente `latest` o promocionar posteriormente una compilación beta examinada
- Gateway extended-stable publica el núcleo, todos los plugins oficiales publicables en npm
  y sus imágenes de Docker con una única versión exacta; véase el flujo de trabajo específico a continuación.
- Cada versión final regular distribuye conjuntamente el paquete de npm, la aplicación de macOS, el APK independiente firmado de Android y los instaladores firmados de Windows Hub. Normalmente, las versiones beta validan y publican primero la ruta de npm/paquetes, mientras que la compilación, firma, notarización y promoción de aplicaciones nativas se reservan para la versión final regular, salvo que se soliciten explícitamente.

## Cadencia de publicación

- Las versiones avanzan primero por beta; stable solo viene después de validar la beta más reciente
- Normalmente, los mantenedores crean las versiones desde una rama `release/YYYY.M.PATCH` creada a partir del `main` actual, de modo que la validación y las correcciones de versiones no bloqueen el nuevo desarrollo en `main`
- Si se ha enviado o publicado una etiqueta beta y necesita una corrección, los mantenedores crean la siguiente etiqueta `-beta.N` en lugar de eliminar o volver a crear la anterior
- El procedimiento detallado de publicación, las aprobaciones, las credenciales y las notas de recuperación son exclusivos para mantenedores

## Publicación mensual extended-stable del Gateway

Para el mes completado `YYYY.M`, se debe crear `extended-stable/YYYY.M.33` y publicar
`.33+` desde esa rama. La etiqueta, la rama, el checkout, la versión del paquete, la comprobación previa y
la validación deben identificar un único commit. Antes de `.33`, el `main` protegido debe contener
una versión final de un mes posterior inferior al parche `33`; los parches de mantenimiento posteriores siguen
siendo aptos.

### Preparar y estabilizar el candidato

Se debe auditar el intervalo no auditado de la línea principal, conciliar el trabajo privado de seguridad, aprobar un
conjunto acotado de backports e integrar un único PR coordinado. No se debe enviar directamente a la
rama canónica.

En la rama canónica, se debe establecer `YYYY.M.P`, ejecutar `pnpm release:prep` y exigir
esa versión en todos los plugins oficiales publicables. A partir del registro aprobado,
se debe generar y confirmar una sección `## YYYY.M.P` completa con `### Highlights`,
`### Changes` y `### Fixes`, citando los PR originales fusionados de `main` para los
backports equivalentes. La comprobación previa rechaza una sección ausente o vacía.

Se debe trasladar la unidad completa del canal de publicación de Docker de la rama principal actual: flujo de trabajo, promotor,
política, clasificador compartido, pruebas y validación del flujo de trabajo. GitHub carga los flujos de trabajo de las etiquetas
desde el commit etiquetado; una copia incompleta puede fallar después de la compilación o
modificar los alias regulares. Se deben ejecutar comprobaciones específicas.

Se debe congelar el SHA completo de la punta de la rama. Antes de etiquetarlo, se deben comprobar previamente sus bytes exactos de npm
y ejecutar la Validación completa de la versión con ese SHA:

```bash
RELEASE_SHA="$(git rev-parse HEAD)"

gh workflow run openclaw-npm-release.yml \
  --ref extended-stable/YYYY.M.33 \
  -f tag="$RELEASE_SHA" \
  -f preflight_only=true \
  -f npm_dist_tag=extended-stable

gh workflow run full-release-validation.yml \
  --ref extended-stable/YYYY.M.33 \
  -f ref=extended-stable/YYYY.M.33 \
  -f release_profile=stable
```

La forma SHA solo sirve para la comprobación previa. La validación debe ejecutarse en la rama canónica; la publicación
vincula la referencia de su flujo de trabajo, el SHA de cabecera/destino, el ID de ejecución y el intento. Se deben guardar ambos ID y
el `run_attempt` correcto; se deben rechazar las pruebas de `release-ci/*`.

Los fallos deben clasificarse antes de editar:

- Producto: integrar otro PR de backport aprobado.
- Herramientas del destino congelado: trasladar únicamente la reparación de compatibilidad mínima que
  pruebe el producto anterior sin modificarlo.
- Proveedor, aprobación, ejecutor o servicio: mantener el candidato sin cambios y usar
  la ruta de reintento acotada.

Cualquier cambio en la rama invalida ambas barreras. Cuando se superen, se debe exigir que la punta aún
sea igual a `RELEASE_SHA` y, después, enviar la etiqueta firmada `vYYYY.M.P`. Los cambios posteriores requieren el siguiente
parche; nunca se debe mover ni eliminar la etiqueta. Su envío inicia `Docker Release`.

### Publicar los paquetes de npm

Se deben publicar todos los plugins oficiales publicables en npm desde el mismo SHA y guardar el
ID de la ejecución correcta:

```bash
RELEASE_SHA="$(git rev-parse HEAD)"
gh workflow run plugin-npm-release.yml \
  --ref extended-stable/YYYY.M.33 \
  -f publish_scope=all-publishable \
  -f ref="$RELEASE_SHA" \
  -f npm_dist_tag=extended-stable
```

El flujo de trabajo abarca todos los paquetes `all-publishable`, incluidos los que no han cambiado,
y verifica cada versión y selector exactos. Las repeticiones reutilizan las versiones publicadas.

A continuación, se debe publicar el tarball del núcleo preparado con las tres identidades de ejecución guardadas:

```bash
gh workflow run openclaw-npm-release.yml \
  --ref extended-stable/YYYY.M.33 \
  -f tag=vYYYY.M.P \
  -f preflight_only=false \
  -f npm_dist_tag=extended-stable \
  -f preflight_run_id=<npm-preflight-run-id> \
  -f full_release_validation_run_id=<full-validation-run-id> \
  -f full_release_validation_run_attempt=<full-validation-run-attempt> \
  -f plugin_npm_run_id=<plugin-npm-run-id>
```

Solo para ensayos fuera de producción, se debe añadir
`-f bypass_extended_stable_guard=true` a la comprobación previa y a la publicación. Solo omite la
protección del mes, nunca las comprobaciones de referencia canónica, igualdad de SHA/etiqueta/versión, procedencia,
aprobación o lectura posterior. Nunca debe usarse en producción.

### Verificar y recuperar

Desde otro checkout limpio del `main` actual, no desde la rama congelada, se debe ejecutar:

```bash
node --import tsx scripts/openclaw-npm-postpublish-verify.ts YYYY.M.P
npm view openclaw@YYYY.M.P version --userconfig "$(mktemp)"
npm view openclaw@extended-stable version --userconfig "$(mktemp)"
```

Se deben exigir firmas y procedencia de npm para la rama canónica, además de la vinculación de la publicación,
la comprobación previa y el resumen del tarball con el SHA de la versión. Ambos comandos deben
devolver `YYYY.M.P`. Se deben verificar todos los paquetes del núcleo preparados y cada plugin oficial `all-publishable`
con su versión y selector exactos.

Si solo falla el selector raíz, se debe usar el comando de reparación
`npm dist-tag add openclaw@YYYY.M.P extended-stable` generado e impreso en
el resumen del flujo de trabajo. Los selectores de plugins existentes u otros selectores del núcleo preparado
deben repararse mediante herramientas aprobadas con credenciales aisladas; el origen OIDC no puede
modificarlos. Nunca se debe volver a publicar una versión inmutable.

Se debe exigir que `Docker Release` verifique las imágenes exactas predeterminadas, reducidas, de navegador y de arquitectura
en GHCR y Docker Hub, incluidas las certificaciones y versiones de plataforma. Solo debe avanzar
`extended-stable`, `extended-stable-slim` y `extended-stable-browser`
por resumen; los alias regulares permanecen sin cambios y se rechaza la reversión automática.

Para reparar alias, se debe ejecutar `Docker Channel Promotion`, sujeto a aprobación, desde el
`main` actual con la etiqueta. Repite las comprobaciones de resumen, certificación y plataforma, permite
una reversión explícita y nunca vuelve a compilar las imágenes.

Slack, Discord y Codex son las superficies de asistencia documentadas inicialmente, no una
lista de permitidos para las versiones: se distribuyen todos los plugins oficiales publicables en npm. Solo la
lista de comprobación regular controla beta/`latest`, GitHub Releases, ClawHub, las aplicaciones nativas, los dispositivos móviles,
el sitio web y los dist-tags privados; esos pasos no deben ejecutarse para esta ruta del Gateway.

## Lista de comprobación del operador de versiones regulares

Esta lista de comprobación representa públicamente el flujo de publicación. Las credenciales privadas, la firma, la notarización, la recuperación de dist-tags y los detalles de reversión de emergencia permanecen en el manual de publicación exclusivo para mantenedores.

1. Se debe comenzar desde el `main` actual: obtener lo más reciente, confirmar que el commit de destino se haya enviado y confirmar que la CI de `main` esté suficientemente verde como para crear una rama.
2. Se debe crear `release/YYYY.M.PATCH` desde ese commit. Los backports son opcionales; solo se debe aplicar el conjunto seleccionado por el operador. Se deben incrementar todas las ubicaciones de versión necesarias, ejecutar `pnpm release:prep`, finalizar las correcciones de la versión y los forward-ports necesarios, y revisar `src/plugins/compat/registry.ts` junto con `src/commands/doctor/shared/deprecation-compat.ts`.
3. Se debe congelar el commit previo al registro de cambios con el producto completo como **SHA del código**. Se debe ejecutar la comprobación previa determinista del código fuente y, después, usar `node scripts/full-release-validation-at-sha.mjs --sha <code-sha> --target-ref release/YYYY.M.PATCH`. Esto fija las herramientas de confianza del flujo de trabajo mientras toda la matriz de Vitest, Docker, QA, paquetes y rendimiento apunta al SHA exacto del código.
4. Los fallos deben clasificarse antes de editar. Un fallo del producto o del código crea un nuevo SHA del código y exige una validación completa correcta para ese SHA. Un fallo del flujo de trabajo, del entorno de pruebas, de las credenciales, de la aprobación o de la infraestructura se repara en su superficie propietaria y se vuelve a ejecutar con el mismo SHA del código.
5. Solo después de que el SHA del código esté verde, se debe generar la sección superior `CHANGELOG.md` a partir de los PR fusionados y los commits directos desde la última etiqueta publicada accesible. Las entradas deben estar orientadas al usuario y sin duplicados. Cuando una etiqueta publicada divergente o un forward-port posterior vuelva a asociar PR ya publicados, se debe pasar explícitamente como `--shipped-ref`.
6. Solo se debe confirmar `CHANGELOG.md`. Este commit es el **SHA de la versión**. La diferencia completa entre el SHA del código y el SHA de la versión debe ser exactamente `CHANGELOG.md`; cualquier otra ruta modificada devuelve la versión al paso 2.
7. Se debe ejecutar la Validación completa de la versión fijada por SHA para el SHA de la versión con la reutilización de pruebas habilitada. El proceso principal ligero debe registrar `changelog-only-release-v1`, apuntar al SHA del código que está verde y no iniciar ninguna vía secundaria del producto. Esto reutiliza las pruebas del producto; no reutiliza los bytes de los paquetes.
8. Se debe ejecutar `OpenClaw NPM Release` con `preflight_only=true` sobre el SHA o la etiqueta de la versión. Se debe guardar el `preflight_run_id` correcto. Esto compila y comprueba los bytes exactos de los paquetes que incluyen el registro de cambios final.
9. Se debe etiquetar el SHA de la versión y, después, ejecutar el asistente del candidato con el proceso principal de validación correcto del SHA de la versión y la comprobación previa de npm, en lugar de volver a iniciar cualquiera de ellos:

   ```bash
   pnpm release:candidate -- \
     --tag vYYYY.M.PATCH-beta.N \
     --full-release-run <release-sha-validation-run-id> \
     --npm-preflight-run <preflight-run-id> \
     --skip-dispatch
   ```

   Para una versión estable, pase también `--windows-node-tag vX.Y.Z`. La herramienta auxiliar verifica la procedencia de las notas de la versión, los bytes de la comprobación preliminar de npm, las pruebas de instalación y actualización de Parallels, la prueba del paquete de Telegram y los planes de publicación de plugins; a continuación, muestra el comando de publicación.

   `OpenClaw Release Publish` envía los paquetes de plugins seleccionados o todos los publicables a npm y, en paralelo, el mismo conjunto a ClawHub; después, una vez que la publicación de los plugins en npm se completa correctamente, promociona el artefacto preparado de comprobación preliminar de npm de OpenClaw con la dist-tag correspondiente. El checkout de la versión permanece como raíz del producto y los datos, mientras que la planificación y la verificación final se ejecutan desde el checkout exacto y de confianza del código fuente del flujo de trabajo, de modo que un commit de una versión anterior no pueda utilizar silenciosamente herramientas de publicación obsoletas. Antes de iniciar cualquier proceso secundario de publicación, renderiza y almacena en caché el cuerpo exacto de la versión de GitHub. Cuando la sección coincidente completa `CHANGELOG.md` se ajusta al límite de 125,000 caracteres de GitHub y al límite de seguridad correspondiente de 125,000 bytes del renderizador, la página contiene esa sección exacta `## YYYY.M.PATCH`, incluido su encabezado. Cuando la sección de origen no cabe, la página conserva las notas editoriales agrupadas exactas y sustituye el registro de contribuciones sobredimensionado por un enlace estable al registro completo en `CHANGELOG.md`, fijado a la etiqueta; nunca se publican registros parciales ni viñetas truncadas. El flujo de trabajo elige ese cuerpo completo o compacto antes de añadir `### Release verification`; si la cola de pruebas superara el límite, conserva el cuerpo canónico y utiliza en su lugar las pruebas adjuntas inmutables. Las versiones estables publicadas en npm `latest` se convierten en la versión más reciente de GitHub, mientras que las versiones estables de mantenimiento conservadas en npm `beta` se crean con GitHub `latest=false`. El flujo de trabajo también carga en la versión de GitHub las pruebas de dependencias de la comprobación preliminar, el manifiesto de validación completa y las pruebas de verificación del registro posteriores a la publicación para responder a incidentes posteriores a la versión. Muestra inmediatamente los ID de las ejecuciones secundarias, aprueba automáticamente las barreras del entorno de publicación que el token del flujo de trabajo puede aprobar, resume los trabajos secundarios fallidos con los finales de sus registros, crea por adelantado el borrador de la página de la versión de GitHub y promociona simultáneamente los recursos de Windows y Android mientras se publica OpenClaw en npm, completa la página de la versión y las pruebas de dependencias cuando esas etapas finalizan correctamente, espera a ClawHub siempre que se publique OpenClaw en npm, ejecuta después el verificador beta de la rama principal de confianza y carga pruebas posteriores a la publicación para la versión de GitHub, el paquete npm, los paquetes npm de plugins seleccionados, los paquetes de ClawHub seleccionados, los ID de las ejecuciones de los flujos de trabajo secundarios y el ID opcional de la ejecución de NPM Telegram. El verificador de arranque de ClawHub exige la ruta y el SHA exactos del flujo de trabajo de la rama principal de confianza, los intentos de ejecución del productor y del terminal, el SHA de la versión, el conjunto de paquetes solicitado, la tupla inmutable del artefacto del paquete y el artefacto terminal de lectura del registro; no se acepta una ejecución correcta heredada de la referencia de la versión.

   A continuación, ejecute la aceptación del paquete posterior a la publicación con el paquete publicado `openclaw@YYYY.M.PATCH-beta.N` o `openclaw@beta`. Si una versión preliminar enviada o publicada necesita una corrección, cree el siguiente número de versión preliminar correspondiente; nunca elimine ni reescriba el anterior.

10. Tras un intento de publicación fallido, mantenga sin cambios el SHA de la versión, salvo que el fallo demuestre un defecto del producto o del registro de cambios. Reanude los procesos secundarios y artefactos inmutables que se hayan completado correctamente; nunca reconstruya ni vuelva a publicar una versión de paquete que ya se haya completado correctamente.
11. Para una versión estable, continúe solo después de que la beta o la candidata a versión examinada disponga de las pruebas de validación requeridas. La publicación estable en npm también se realiza mediante `OpenClaw Release Publish`, reutilizando el artefacto correcto de la comprobación preliminar mediante `preflight_run_id`. La preparación de la versión estable para macOS también exige los archivos empaquetados `.zip`, `.dmg`, `.dSYM.zip` y el archivo `appcast.xml` actualizado en `main`; el flujo de trabajo de publicación de macOS publica automáticamente el appcast firmado en el recurso público `main` después de verificar los recursos de la versión, o abre o actualiza un pull request del appcast si la protección de la rama bloquea el envío directo. La preparación estable de Windows Hub exige los recursos firmados `OpenClawCompanion-Setup-x64.exe`, `OpenClawCompanion-Setup-arm64.exe` y `OpenClawCompanion-SHA256SUMS.txt` en la versión de GitHub de OpenClaw. Pase la etiqueta exacta de la versión firmada `openclaw/openclaw-windows-node` como `windows_node_tag` y su mapa de resúmenes de instaladores aprobado para la candidata como `windows_node_installer_digests`; `OpenClaw Release Publish` conserva el borrador de la versión, envía `Windows Node Release` y verifica los tres recursos antes de la publicación.
12. Después de la publicación, ejecute el verificador posterior a la publicación de npm, la prueba E2E independiente y opcional de Telegram con el paquete npm publicado cuando se necesite una prueba del canal posterior a la publicación, la promoción de la dist-tag cuando sea necesaria, verifique la página generada de la versión de GitHub, ejecute los pasos del anuncio de la versión y, a continuación, complete el [cierre estable de la rama principal](#stable-main-closeout) antes de considerar finalizada una versión estable.

## Cierre estable de la rama principal

La publicación estable no está completa hasta que `main` contenga el estado real de la versión publicada.

1. Comience desde la última versión nueva de `main`. Audite `release/YYYY.M.PATCH` respecto a ella e incorpore hacia delante las correcciones reales que falten en `main`. No fusione ciegamente en la versión más reciente de `main` los adaptadores de compatibilidad, pruebas o validación exclusivos de la versión.
2. Para la ruta normal, establezca `main` en la versión estable publicada. Un cierre tardío puede utilizar `main` después de que haya avanzado a un CalVer estable posterior de OpenClaw; no revierta a una versión anterior un ciclo de publicación ya iniciado únicamente para cerrar la versión anterior. El validador continúa exigiendo la sección exacta del registro de cambios y la entrada del appcast de la versión publicada, y registra la versión y el SHA reales de `main`. Ejecute `pnpm release:prep` después de cualquier cambio de la versión raíz y, a continuación, `pnpm deps:shrinkwrap:generate`.
3. Haga que la sección `## YYYY.M.PATCH` de `CHANGELOG.md` en `main` coincida exactamente con la rama etiquetada de la versión. Incluya la actualización estable de `appcast.xml` cuando la versión para Mac haya publicado una.
4. No añada `YYYY.M.PATCH+1`, una versión beta ni una sección futura vacía del registro de cambios a `main` hasta que el operador inicie explícitamente ese ciclo de publicación.
5. Ejecute `pnpm release:generated:check`, `pnpm deps:shrinkwrap:check` y `OPENCLAW_TESTBOX=1 pnpm check:changed`. Envíe los cambios y, después, verifique que `origin/main` contenga la versión publicada y el registro de cambios antes de considerar finalizada la versión estable.
6. Mantenga actualizadas las variables del repositorio `RELEASE_ROLLBACK_DRILL_ID` y `RELEASE_ROLLBACK_DRILL_DATE` después de cada simulacro privado de reversión.

`OpenClaw Stable Main Closeout` se inicia a partir del envío de `main` que contiene la versión publicada, el registro de cambios y el appcast tras la publicación estable. Lee las pruebas inmutables posteriores a la publicación para vincular la etiqueta publicada con sus ejecuciones de validación completa de la versión y publicación; después, verifica el estado estable de la rama principal, la versión, el periodo de observación estable obligatorio y las pruebas de rendimiento bloqueantes. Adjunta un manifiesto inmutable de cierre y su suma de comprobación a la versión de GitHub. El desencadenador automático por envío omite las versiones heredadas anteriores a las pruebas inmutables posteriores a la publicación y nunca considera esa omisión como un cierre completado.

Un cierre completo exige ambos recursos y una suma de comprobación coincidente. Un manifiesto parcial reproduce el SHA registrado en `main` y el simulacro de reversión para regenerar bytes idénticos; después, adjunta la suma de comprobación que falta. Un par no válido, o una suma de comprobación sin manifiesto, continúa bloqueando el proceso. Una ejecución desencadenada por un envío sin variables de repositorio del simulacro de reversión se omite sin completar el cierre; la ausencia de un registro del simulacro, o uno con más de 90 días de antigüedad, sigue bloqueando el cierre manual respaldado por pruebas. Los comandos privados de recuperación permanecen en el manual de ejecución exclusivo para mantenedores. Utilice el envío manual únicamente para reparar o reproducir un cierre estable respaldado por pruebas.

Si el proceso principal de publicación de la versión falló únicamente después de adjuntar las pruebas inmutables de npm y los plugins, repare y publique primero todos los recursos de las plataformas estables. Después, un mantenedor puede enviar manualmente el cierre con `allow_failed_publish_recovery=true`; este modo solo acepta un proceso principal fallido y completado, y además exige los contratos exactos de los recursos de Android y Windows, los resúmenes SHA-256 de GitHub, la verificación de las sumas de comprobación, la procedencia de Android y una promoción correcta de Windows enviada por el proceso principal, cuyas comprobaciones de Authenticode y resúmenes aprobados para la candidata coincidan con los instaladores publicados, junto con las comprobaciones normales de macOS y el appcast. El cierre automático desencadenado por un envío nunca habilita este modo de recuperación.

Una etiqueta heredada de corrección alternativa solo puede reutilizar las pruebas del paquete base cuando la etiqueta de corrección se resuelva al mismo commit de origen que la etiqueta estable base. Su versión de Android reutiliza el APK verificado de la etiqueta base y añade la procedencia de la etiqueta de corrección. Una corrección con un origen distinto debe publicar y verificar sus propias pruebas del paquete y utilizar un valor superior de Android `versionCode`.

## Comprobación preliminar de la versión

- Ejecute `pnpm check:test-types` antes de la comprobación preliminar de la versión para que las pruebas de TypeScript sigan cubiertas fuera de la barrera local más rápida `pnpm check`.
- Ejecute `pnpm check:architecture` antes de la comprobación preliminar de la versión para que las comprobaciones más amplias de ciclos de importación y límites de arquitectura estén correctas fuera de la barrera local más rápida.
- Ejecute `pnpm build && pnpm ui:build` antes de `pnpm release:check` para que los artefactos esperados de la versión `dist/*` y el paquete de Control UI existan durante el paso de validación del paquete.
- Ejecute `pnpm release:prep` después de incrementar la versión raíz y antes de etiquetar. Ejecuta todos los generadores deterministas de versiones que suelen quedar desactualizados después de un cambio de versión, configuración o API: versiones de plugins, archivos shrinkwrap de npm, inventario de plugins, esquema de configuración base, metadatos de configuración de los canales incluidos, referencia de la documentación de configuración, exportaciones del SDK de plugins, manifiesto del contrato de la API del SDK de plugins y paquetes de configuraciones regionales de Control UI. También bloquea el proceso hasta que las traducciones de las aplicaciones nativas y los recursos de configuraciones regionales generados para las plataformas coincidan con el inventario de origen; si están retrasados, espere a `Native App Locale Refresh` o envíelo antes de inmovilizar el SHA del código. `pnpm release:check` vuelve a ejecutar esas comprobaciones en modo de verificación —incluidas las barreras estrictas de configuraciones regionales y el presupuesto de superficie del SDK de plugins— y comunica en una sola pasada todos los fallos por divergencias generadas antes de ejecutar las comprobaciones de publicación de los paquetes.
- De forma predeterminada, la sincronización de versiones de plugins actualiza a la versión de OpenClaw el paquete publicable de tiempo de ejecución `@openclaw/ai`, las versiones de los paquetes oficiales de plugins y los límites mínimos existentes de `openclaw.compat.pluginApi`. Trate ese campo como el límite mínimo de la API del SDK o del tiempo de ejecución del plugin, no simplemente como una copia de la versión del paquete: para las versiones exclusivas de plugins que permanezcan intencionadamente compatibles con hosts anteriores de OpenClaw, mantenga el límite mínimo en la API del host compatible más antiguo y documente esa elección en las pruebas de la versión del plugin.
- Ejecute el flujo de trabajo manual `Full Release Validation` antes de aprobar la versión para iniciar todos los entornos de pruebas previos a la publicación desde un único punto de entrada. Acepta una rama, una etiqueta o un SHA completo de commit, envía manualmente `CI` y envía `OpenClaw Release Checks` para las vías de pruebas rápidas de instalación, aceptación de paquetes, comprobaciones de paquetes entre sistemas operativos, paridad de QA Lab, Matrix y Telegram. Las ejecuciones estables y completas siempre incluyen pruebas exhaustivas en vivo y E2E, así como un periodo de observación mediante Docker de la ruta de publicación; `run_release_soak=true` se conserva para un periodo de observación beta explícito. La aceptación de paquetes proporciona la prueba E2E canónica de Telegram para el paquete durante la validación de la candidata, lo que evita un segundo proceso de consulta en vivo simultáneo.

  Proporcione `release_package_spec` después de publicar una beta para reutilizar el paquete npm publicado en las comprobaciones de la versión, la aceptación de paquetes y la prueba E2E del paquete de Telegram sin reconstruir el archivo tar de la versión. Proporcione `npm_telegram_package_spec` únicamente cuando Telegram deba utilizar un paquete publicado distinto del utilizado en el resto de la validación de la versión. Proporcione `package_acceptance_package_spec` cuando la aceptación de paquetes deba utilizar un paquete publicado distinto del especificado para la versión. Proporcione `evidence_package_spec` cuando el informe de pruebas de la versión deba demostrar que la validación coincide con un paquete npm publicado sin obligar a ejecutar la prueba E2E de Telegram.

  ```bash
  node scripts/full-release-validation-at-sha.mjs \
    --sha <code-sha> \
    --target-ref release/YYYY.M.PATCH
  ```

- Ejecute el flujo de trabajo manual `Package Acceptance` cuando necesite una prueba por un canal secundario para un paquete candidato mientras continúa el trabajo de publicación. Use `source=npm` para `openclaw@beta`, `openclaw@latest` o una versión de publicación exacta; `source=ref` para empaquetar una rama/etiqueta/SHA de `package_ref` de confianza con el arnés `workflow_ref` actual; `source=url` para un tarball HTTPS público con un SHA-256 obligatorio y una política estricta de URL públicas; `source=trusted-url` para una política de fuente de confianza con nombre que use el `trusted_source_id` obligatorio y SHA-256; o `source=artifact` para un tarball cargado por otra ejecución de GitHub Actions.

  El flujo de trabajo resuelve el candidato como `package-under-test`, reutiliza el programador de publicaciones E2E de Docker con ese tarball y puede ejecutar el control de calidad de Telegram con el mismo tarball mediante `telegram_mode=mock-openai` o `telegram_mode=live-frontier`. Cuando los carriles de Docker seleccionados incluyen `published-upgrade-survivor`, el artefacto del paquete es el candidato y `published_upgrade_survivor_baseline` selecciona la línea base publicada. `update-restart-auth` usa el paquete candidato tanto como CLI instalada como paquete sometido a prueba para ejercitar la ruta de reinicio administrado del comando de actualización del candidato.

  Ejemplo:

  ```bash
  gh workflow run package-acceptance.yml --ref main -f workflow_ref=main -f source=npm -f package_spec=openclaw@beta -f suite_profile=product -f published_upgrade_survivor_baseline=openclaw@2026.4.26 -f telegram_mode=mock-openai
  ```

  Perfiles habituales:
  - `smoke`: carriles de instalación/canal/agente, red del Gateway y recarga de configuración
  - `package`: carriles de paquete/actualización/reinicio/plugin nativos del artefacto, sin OpenWebUI ni ClawHub en vivo
  - `product`: perfil de paquete más canales MCP, limpieza de Cron/subagentes, búsqueda web de OpenAI y OpenWebUI
  - `full`: segmentos de la ruta de publicación de Docker con OpenWebUI
  - `custom`: selección exacta de `docker_lanes` para una repetición enfocada

- Ejecute directamente el flujo de trabajo manual `CI` cuando solo necesite una cobertura de CI normal y determinista para el candidato de publicación. Las ejecuciones manuales de CI omiten el alcance basado en cambios y fuerzan los fragmentos de Linux Node, los fragmentos de plugins incluidos, los fragmentos de contratos de plugins y canales, la compatibilidad con Node 22, `check-*`, `check-additional-*`, las comprobaciones rápidas de artefactos compilados, las comprobaciones de documentación, las Skills de Python, Windows, macOS y los carriles de i18n de la interfaz de control. Las ejecuciones manuales independientes de CI solo ejecutan Android cuando se inician con `include_android=true`; `Full Release Validation` pasa esa entrada a su CI secundaria.

  ```bash
  gh workflow run ci.yml --ref release/YYYY.M.PATCH -f include_android=true
  ```

- Ejecute `pnpm qa:otel:smoke` al validar la telemetría de publicación. Ejercita el laboratorio de control de calidad mediante un receptor OTLP/HTTP local y verifica la exportación de trazas, métricas y registros, además de los atributos de traza acotados y la censura de contenido e identificadores, sin requerir Opik, Langfuse ni otro recopilador externo.
- Ejecute `pnpm qa:otel:collector-smoke` al validar la compatibilidad del recopilador. Enruta la misma exportación OTLP del laboratorio de control de calidad a través de un contenedor Docker real de OpenTelemetry Collector antes de las aserciones del receptor local.
- Ejecute `pnpm qa:prometheus:smoke` al validar la extracción protegida de Prometheus. Ejercita el laboratorio de control de calidad, rechaza las extracciones no autenticadas y verifica que las familias de métricas críticas para la publicación no contengan contenido de solicitudes, identificadores sin procesar, tokens de autenticación ni rutas locales.
- Ejecute `pnpm qa:observability:smoke` para ejecutar consecutivamente los carriles de comprobación rápida de OpenTelemetry y Prometheus desde el checkout del código fuente.
- Ejecute `pnpm release:check` antes de cada publicación etiquetada.
- La comprobación preliminar `OpenClaw NPM Release` genera pruebas de publicación de las dependencias antes de empaquetar el tarball de npm. La puerta de vulnerabilidades de los avisos de npm bloquea la publicación. Los informes de riesgo del manifiesto transitivo, de propiedad/superficie de instalación de dependencias y de cambios en las dependencias solo constituyen pruebas de publicación. El informe de cambios en las dependencias compara el candidato de publicación con la etiqueta de publicación accesible anterior. La comprobación preliminar carga las pruebas de dependencias como `openclaw-release-dependency-evidence-<tag>` y también las incorpora en `dependency-evidence/` dentro del artefacto preparado de comprobación preliminar de npm. La ruta de publicación real reutiliza ese artefacto de comprobación preliminar y después adjunta las mismas pruebas a la publicación de GitHub como `openclaw-<version>-dependency-evidence.zip`.
- Ejecute `OpenClaw Release Publish` para la secuencia de publicación con cambios después de que exista la etiqueta. Inicie las publicaciones beta y estables habituales desde `main` de confianza; la etiqueta de publicación sigue seleccionando el commit de destino exacto y puede apuntar a `release/YYYY.M.PATCH`. Las publicaciones alfa de Tideclaw permanecen en su rama alfa correspondiente. Pase el `preflight_run_id` de npm de OpenClaw correcto, el `full_release_validation_run_id` correcto y el `full_release_validation_run_attempt` exacto, y mantenga el alcance predeterminado de publicación de plugins `all-publishable`, salvo que ejecute deliberadamente una reparación enfocada. El flujo de trabajo serializa la publicación de plugins en npm, la publicación de plugins en ClawHub y la publicación de OpenClaw en npm para que el paquete principal no se publique antes que sus plugins externalizados; la promoción de Windows y Android se ejecuta simultáneamente con la publicación del paquete principal en npm usando la página de publicación en borrador. Las repeticiones de la publicación se pueden reanudar: una versión principal ya publicada en npm omite la ejecución principal después de que el flujo de trabajo demuestre que el tarball del registro coincide con el artefacto de comprobación preliminar de la etiqueta, y la promoción de Windows/Android se omite cuando la publicación ya contiene el contrato de artefactos verificado, por lo que un reintento solo repite las etapas fallidas. Las reparaciones enfocadas exclusivamente en plugins requieren `plugin_publish_scope=selected` y una lista de plugins que no esté vacía. Las ejecuciones `all-publishable` exclusivas de plugins requieren pruebas completas e inmutables de la comprobación preliminar y de la validación completa de la publicación; se rechazan las pruebas parciales.
- La `OpenClaw Release Publish` estable requiere un `windows_node_tag` exacto después de que exista la publicación `openclaw/openclaw-windows-node` correspondiente que no sea una versión preliminar, además del mapa `windows_node_installer_digests` aprobado para el candidato. Antes de iniciar cualquier publicación secundaria, verifica que esa publicación de origen esté publicada, no sea una versión preliminar, contenga los instaladores x64/ARM64 obligatorios y siga coincidiendo con ese mapa aprobado. Después inicia `Windows Node Release` mientras la publicación de OpenClaw aún está en borrador, conservando sin cambios el mapa fijado de resúmenes de los instaladores. El flujo de trabajo secundario descarga los instaladores firmados de Windows Hub desde esa etiqueta exacta, los coteja con los resúmenes fijados, verifica en un ejecutor de Windows que sus firmas Authenticode usan el firmante esperado de OpenClaw Foundation, escribe un manifiesto SHA-256 y carga los instaladores y el manifiesto en la publicación canónica de OpenClaw en GitHub; después vuelve a descargar los artefactos promocionados y verifica su pertenencia al manifiesto y sus hashes. El flujo de trabajo principal verifica el contrato actual de los artefactos x64, ARM64 y de suma de comprobación antes de la publicación. La recuperación directa rechaza los nombres inesperados de artefactos `OpenClawCompanion-*` antes de sustituir los artefactos esperados del contrato por los bytes fijados del origen.

  Inicie manualmente `Windows Node Release` solo para recuperación y pase siempre una etiqueta exacta, nunca `latest`, además del mapa JSON explícito `expected_installer_digests` de la publicación de origen aprobada. Los enlaces de descarga del sitio web deben apuntar a las URL exactas de los artefactos de publicación de OpenClaw para la publicación estable actual, o a `releases/latest/download/...` solo después de verificar que la redirección de la versión más reciente de GitHub apunta a esa misma publicación; no enlace únicamente a la página de publicación del repositorio complementario.

- Las comprobaciones de la versión ahora se ejecutan en un flujo de trabajo manual independiente: `OpenClaw Release Checks`. También ejecuta la vía de paridad simulada de QA Lab, además del perfil de lanzamiento de Matrix y la vía de QA de Telegram, antes de aprobar la versión. Las vías en vivo usan el entorno `qa-live-shared`; Telegram también usa concesiones de credenciales de CI de Convex. Ejecute el flujo de trabajo manual `QA-Lab - All Lanes` con `matrix_profile=all` cuando quiera incluir todos los escenarios de Matrix mantenidos; el flujo de trabajo distribuye esa selección entre los perfiles de transporte, contenido multimedia y E2EE para mantener la comprobación completa dentro de los tiempos de espera de cada trabajo.
- La validación del entorno de ejecución para la instalación y actualización entre sistemas operativos forma parte de los flujos públicos `OpenClaw Release Checks` y `Full Release Validation`, que invocan directamente el flujo de trabajo reutilizable `.github/workflows/openclaw-cross-os-release-checks-reusable.yml`. Esta separación es intencionada: mantiene la ruta real de publicación de npm breve, determinista y centrada en los artefactos, mientras que las comprobaciones en vivo más lentas permanecen en su propia vía para que no retrasen ni bloqueen la publicación.
- Las comprobaciones de la versión que contienen secretos deben iniciarse mediante `Full Release Validation` o desde la referencia del flujo de trabajo `main`/release para que la lógica del flujo y los secretos permanezcan controlados.
- `OpenClaw Release Checks` acepta una rama, una etiqueta o un SHA de confirmación completo, siempre que la confirmación resuelta sea accesible desde una rama o etiqueta de versión de OpenClaw.
- La comprobación preliminar de `OpenClaw NPM Release`, destinada únicamente a la validación, también acepta el SHA completo actual de 40 caracteres de la confirmación de la rama del flujo de trabajo sin exigir una etiqueta enviada. Esa ruta mediante SHA es solo para validación y no puede promoverse a una publicación real. En el modo SHA, el flujo de trabajo sintetiza `v<package.json version>` únicamente para comprobar los metadatos del paquete; la publicación real sigue requiriendo una etiqueta de versión real.
- Ambos flujos de trabajo mantienen la ruta real de publicación y promoción en ejecutores alojados en GitHub, mientras que la ruta de validación que no realiza cambios puede usar los ejecutores Linux de mayor tamaño de Blacksmith.
- Ese flujo de trabajo ejecuta `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_CACHE_TEST=1 pnpm test:live:cache` usando los secretos del flujo de trabajo `OPENAI_API_KEY` y `ANTHROPIC_API_KEY`.
- La comprobación preliminar de la versión de npm ya no espera a la vía independiente de comprobaciones de la versión.
- Antes de etiquetar localmente una versión candidata, ejecute `RELEASE_TAG=vYYYY.M.PATCH-beta.N pnpm release:fast-pretag-check`. El asistente ejecuta las protecciones rápidas de la versión, las comprobaciones de publicación del plugin en npm/ClawHub, la compilación, la compilación de la interfaz de usuario y `release:openclaw:npm:check` en el orden que permite detectar errores comunes que bloquearían la aprobación antes de que comience el flujo de publicación de GitHub.
- Ejecute `RELEASE_TAG=vYYYY.M.PATCH node --import tsx scripts/openclaw-npm-release-check.ts` (o la etiqueta correspondiente de versión preliminar/corrección) antes de la aprobación.
- Después de publicar en npm, ejecute `node --import tsx scripts/openclaw-npm-postpublish-verify.ts YYYY.M.PATCH` (o la versión beta/corrección correspondiente) para verificar la ruta de instalación desde el registro publicado en un prefijo temporal nuevo.
- Después de publicar una beta, ejecute `OPENCLAW_NPM_TELEGRAM_PACKAGE_SPEC=openclaw@YYYY.M.PATCH-beta.N OPENCLAW_NPM_TELEGRAM_CREDENTIAL_SOURCE=convex OPENCLAW_NPM_TELEGRAM_CREDENTIAL_ROLE=ci pnpm test:docker:npm-telegram-live` para verificar la incorporación desde el paquete instalado, la configuración de Telegram y las pruebas E2E reales de Telegram con el paquete de npm publicado mediante el conjunto compartido de credenciales de Telegram concedidas. Para ejecuciones locales puntuales, los mantenedores pueden omitir las variables de Convex y proporcionar directamente las tres credenciales de entorno `OPENCLAW_QA_TELEGRAM_*`.
- Para ejecutar la comprobación de humo beta completa posterior a la publicación desde el equipo de un mantenedor, use `pnpm release:beta-smoke -- --beta betaN`. El asistente ejecuta la validación de actualización de npm y destino nuevo en Parallels, inicia `NPM Telegram Beta E2E`, consulta periódicamente la ejecución exacta del flujo de trabajo, descarga el artefacto e imprime el informe de Telegram.
- Los mantenedores pueden ejecutar la misma comprobación posterior a la publicación desde GitHub Actions mediante el flujo de trabajo manual `NPM Telegram Beta E2E`. Es intencionadamente solo manual y no se ejecuta con cada fusión.
- La automatización de versiones para mantenedores usa primero la comprobación preliminar y después la promoción:
  - La publicación real en npm debe superar correctamente una `preflight_run_id` de npm.
  - La orquestación y la comprobación preliminar de publicaciones beta y estables normales usan `main` de confianza con la etiqueta de destino exacta. La publicación y la comprobación preliminar alfa de Tideclaw usan la rama alfa correspondiente.
  - Las versiones estables de npm usan `beta` de forma predeterminada; la publicación estable en npm puede dirigirse explícitamente a `latest` mediante una entrada del flujo de trabajo.
  - La modificación del dist-tag de npm basada en tokens reside en `openclaw/releases/.github/workflows/openclaw-npm-dist-tags.yml` porque `npm dist-tag add` todavía necesita `NPM_TOKEN`, mientras que el repositorio de origen mantiene la publicación exclusivamente mediante OIDC.
  - El flujo público `macOS Release` es solo para validación; cuando una etiqueta existe únicamente en una rama de versión, pero el flujo de trabajo se inicia desde `main`, establezca `public_release_branch=release/YYYY.M.PATCH`.
  - La publicación real de macOS debe superar correctamente las `preflight_run_id` y `validate_run_id` de macOS.
  - Las rutas de publicación reales promueven los artefactos preparados en lugar de volver a compilarlos.
- Para versiones estables de corrección como `YYYY.M.PATCH-N`, el verificador posterior a la publicación también comprueba la misma ruta de actualización con prefijo temporal desde `YYYY.M.PATCH` hasta `YYYY.M.PATCH-N`, para que las correcciones de versiones no puedan dejar silenciosamente instalaciones globales anteriores con la carga útil de la versión estable base.
- La comprobación preliminar de la versión de npm falla de forma segura, salvo que el archivo tar incluya tanto `dist/control-ui/index.html` como una carga útil `dist/control-ui/assets/` no vacía, para evitar volver a distribuir un panel del navegador vacío.
- La verificación posterior a la publicación también comprueba que los puntos de entrada de los plugins publicados y los metadatos del paquete estén presentes en la disposición instalada del registro. Una versión que distribuya cargas útiles del entorno de ejecución de plugins ausentes no supera el verificador posterior a la publicación y no puede promoverse a `latest`.
- `pnpm test:install:smoke` también aplica el presupuesto de `unpackedSize` de npm pack al archivo tar de actualización candidato, para que las pruebas E2E del instalador detecten aumentos accidentales del paquete antes de la ruta de publicación de la versión.
- Si el trabajo de la versión modificó la planificación de CI, los manifiestos de tiempos de extensiones o las matrices de pruebas de extensiones, regenere y revise las salidas de la matriz `plugin-prerelease-extension-shard`, propiedad del planificador, desde `.github/workflows/plugin-prerelease.yml` antes de la aprobación, para que las notas de la versión no describan una disposición obsoleta de CI.
- La preparación de una versión estable para macOS también incluye las superficies del actualizador: la versión de GitHub debe terminar con los archivos empaquetados `.zip`, `.dmg` y `.dSYM.zip`; `appcast.xml` en `main` debe apuntar al nuevo archivo zip estable después de la publicación (el flujo de publicación de macOS lo confirma automáticamente o abre un pull request del appcast cuando se bloquea el envío directo); la aplicación empaquetada debe conservar un identificador de paquete que no sea de depuración, una URL de fuente de Sparkle no vacía y un `CFBundleVersion` igual o superior al mínimo canónico de compilación de Sparkle para esa versión.

## Entornos de prueba de versiones

`Full Release Validation` es el modo en que los operadores inician la matriz completa del producto desde un único punto de entrada. Use el asistente para que cada flujo de trabajo secundario se ejecute desde una rama temporal fijada a un único SHA de flujo de trabajo `main` de confianza, mientras la confirmación solicitada sigue siendo la candidata sometida a prueba:

```bash
pnpm ci:full-release \
  --sha <code-sha> \
  --target-ref release/YYYY.M.PATCH
```

El asistente obtiene el `origin/main` actual, envía `release-ci/<workflow-sha>-...` en esa confirmación de flujo de trabajo de confianza, deduce `beta` a partir de las versiones alfa/beta del paquete y `stable` en los demás casos, inicia `Full Release Validation` desde la rama temporal con `ref=<target-sha>`, verifica que cada `headSha` de los flujos de trabajo secundarios coincida con el SHA fijado del flujo de trabajo principal y, después, elimina la rama temporal. Proporcione `-f reuse_evidence=false` para forzar una ejecución nueva, `-f release_profile=full` para el análisis consultivo amplio o `--workflow-sha <trusted-main-sha>` para fijar una confirmación anterior que todavía sea accesible desde el `origin/main` actual. El propio flujo de trabajo nunca escribe referencias del repositorio. Esto mantiene disponibles las herramientas de publicación exclusivas de main sin añadir confirmaciones de herramientas a la candidata y evita validar accidentalmente una ejecución secundaria más reciente de `main`.

Cuando el SHA del código esté en verde, confirme únicamente `CHANGELOG.md` y ejecute el mismo asistente con el SHA de la versión:

```bash
pnpm ci:full-release \
  --sha <release-sha> \
  --target-ref release/YYYY.M.PATCH
```

El segundo flujo principal reutiliza la evidencia del producto únicamente cuando GitHub demuestra que el SHA de la versión desciende del SHA del código y que el conjunto completo de rutas modificadas es exactamente `CHANGELOG.md`. Registra `changelog-only-release-v1` y no inicia ningún flujo secundario del producto. La comprobación preliminar de npm y la aceptación del paquete/instalación siguen ejecutándose en el SHA de la versión porque los bytes de su archivo tar cambiaron.

Para un SHA de código nuevo, el flujo de trabajo resuelve el destino, inicia manualmente `CI` y después inicia `OpenClaw Release Checks`. `OpenClaw Release Checks` distribuye las comprobaciones de humo de instalación, las comprobaciones de versión entre sistemas operativos, la cobertura en vivo/E2E de la ruta de versión en Docker cuando se activa la ejecución prolongada, la aceptación del paquete con las pruebas E2E canónicas del paquete de Telegram, la paridad de QA Lab, Matrix en vivo y Telegram en vivo. Una ejecución completa/total solo es aceptable cuando el resumen de `Full Release Validation` muestra `normal_ci`, `plugin_prerelease` y `release_checks` como correctos, salvo que una repetición de ejecución específica haya omitido intencionadamente el flujo secundario independiente `Plugin Prerelease`. Use el flujo secundario independiente `npm-telegram` solo para repetir específicamente las pruebas del paquete publicado con `release_package_spec` o `npm_telegram_package_spec`. El resumen final del verificador incluye tablas de los trabajos más lentos de cada ejecución secundaria, para que el responsable de la versión pueda ver la ruta crítica actual sin descargar los registros.

El flujo secundario de rendimiento del producto solo genera artefactos en esta ruta de publicación. El
flujo general lo inicia con `publish_reports=false`, y la validación se rechaza
salvo que su protección de solo artefactos demuestre que el publicador de informes de Clawgrit permaneció
omitido.

Consulte [Validación completa de la versión](/es/reference/full-release-validation) para conocer la matriz completa de etapas, los nombres exactos de los trabajos del flujo de trabajo, las diferencias entre los perfiles estable y completo, los artefactos y los identificadores para repetir ejecuciones específicas.

Los flujos de trabajo secundarios se inician desde la referencia de confianza fijada mediante SHA que ejecuta `Full Release Validation`. Cada ejecución secundaria debe usar el SHA exacto del flujo de trabajo principal. No use inicios directos de `--ref main -f ref=<sha>` como comprobación de la versión; use `pnpm ci:full-release --sha <target-sha> --target-ref release/YYYY.M.PATCH`.

Use `release_profile` para seleccionar la amplitud en vivo/de proveedores:

- `beta`: ruta crítica de versión más rápida para OpenAI/núcleo en vivo y Docker
- `stable`: cobertura beta y estable de proveedores/backends para aprobar la versión
- `full`: cobertura estable más una cobertura consultiva amplia de proveedores/contenido multimedia

La validación estable y completa siempre ejecuta el análisis exhaustivo en vivo/E2E, la ruta de versión en Docker y el análisis acotado de supervivencia a actualizaciones publicadas antes de la promoción. Use `run_release_soak=true` para solicitar ese mismo análisis para una beta. Dicho análisis cubre los cuatro paquetes estables más recientes, además de las líneas base fijadas `2026.4.23` y `2026.5.2`, así como la cobertura anterior de `2026.4.15`; se eliminan las líneas base duplicadas y cada una se divide en su propio trabajo de ejecutor de Docker.

`OpenClaw Release Checks` usa la referencia de flujo de trabajo de confianza para resolver una vez la referencia de destino como `release-package-under-test` y reutiliza ese artefacto en las comprobaciones entre sistemas operativos, la aceptación del paquete y las comprobaciones de Docker de la ruta de versión cuando se ejecuta la prueba prolongada. Esto mantiene todos los entornos orientados a paquetes sobre los mismos bytes y evita repetir las compilaciones del paquete. Cuando una beta ya esté en npm, establezca `release_package_spec=openclaw@YYYY.M.PATCH-beta.N` para que las comprobaciones de la versión descarguen una vez el paquete distribuido, extraigan su SHA del código fuente de compilación desde `dist/build-info.json` y reutilicen ese artefacto en las vías entre sistemas operativos, de aceptación del paquete, de Docker de la ruta de versión y del paquete de Telegram.

La comprobación de humo de instalación de OpenAI entre sistemas operativos usa `OPENCLAW_CROSS_OS_OPENAI_MODEL` cuando está definida la variable del repositorio o la organización; en caso contrario, usa `openai/gpt-5.6-luna`, porque esta vía verifica la instalación del paquete, la incorporación, el inicio del Gateway y un turno de agente en vivo, en lugar de evaluar comparativamente el modelo con mayores capacidades. La matriz más amplia de proveedores en vivo sigue siendo el lugar destinado a la cobertura específica de cada modelo.

Use estas variantes según la etapa de la versión:

```bash
# Valida el SHA de código del producto completo.
pnpm ci:full-release \
  --sha <code-sha> \
  --target-ref release/YYYY.M.PATCH

# Valida el SHA de la versión que solo contiene el registro de cambios reutilizando la evidencia del producto del SHA de código.
pnpm ci:full-release \
  --sha <release-sha> \
  --target-ref release/YYYY.M.PATCH

# Después de publicar una beta, añade el E2E de Telegram del paquete publicado.
pnpm ci:full-release \
  --sha <release-sha> \
  --target-ref release/YYYY.M.PATCH \
  -f release_package_spec=openclaw@YYYY.M.PATCH-beta.N \
  -f evidence_package_spec=openclaw@YYYY.M.PATCH-beta.N \
  -f npm_telegram_provider_mode=mock-openai
```

No utilice el conjunto completo como primera repetición después de una corrección específica. Si falla una caja, utilice para la siguiente prueba el flujo de trabajo secundario, trabajo, carril de Docker, perfil de paquete, proveedor de modelos o carril de QA que haya fallado. Vuelva a ejecutar el conjunto completo únicamente cuando la corrección haya cambiado la orquestación compartida de la versión o haya dejado obsoleta la evidencia anterior de todas las cajas. El verificador final del conjunto vuelve a comprobar los identificadores registrados de las ejecuciones de los flujos de trabajo secundarios; por tanto, después de repetir correctamente un flujo de trabajo secundario, repita únicamente el trabajo principal `Verify full validation` que falló.

`rerun_group=all` puede reutilizar una ejecución anterior correcta del conjunto cuando coincidan el perfil de la versión,
la configuración efectiva de la prueba prolongada y las entradas de validación, y cuando el SHA de destino
sea idéntico o el nuevo destino sea un descendiente cuyo conjunto completo de rutas modificadas
sea exactamente `CHANGELOG.md`. La reutilización del destino exacto registra
`exact-target-full-validation-v1`; el SHA de la versión posterior a la validación registra
`changelog-only-release-v1`. Este último reutiliza únicamente la validación del producto. La comprobación
preliminar de npm, los bytes del paquete, la procedencia de las notas de la versión y la aceptación
de instalación/actualización deben seguir ejecutándose con el SHA de la versión. Cualquier cambio del destino
perteneciente a la versión, el código fuente, los archivos generados, las dependencias, el paquete o el flujo de trabajo requiere un nuevo SHA de código
y una validación completa nueva. Las ejecuciones más recientes del conjunto para la misma referencia `release/*` y
el mismo grupo de repetición sustituyen automáticamente a las que están en curso. Pase
`reuse_evidence=false` para forzar una nueva ejecución completa.

Para una recuperación acotada, pase `rerun_group` al conjunto. `all` es la ejecución real de la candidata a versión, `ci` ejecuta únicamente el flujo secundario de CI normal, `plugin-prerelease` ejecuta únicamente el flujo secundario de plugins exclusivo de la versión, `release-checks` ejecuta todas las cajas de la versión y los grupos de versión más específicos son `install-smoke`, `cross-os`, `live-e2e`, `package`, `qa`, `qa-parity`, `qa-live` y `npm-telegram`. Las repeticiones específicas de `npm-telegram` requieren `release_package_spec` o `npm_telegram_package_spec`; las ejecuciones completas/totales usan el E2E canónico de Telegram del paquete dentro de Package Acceptance. Las repeticiones específicas entre sistemas operativos pueden añadir `cross_os_suite_filter=windows/packaged-upgrade` u otro filtro de sistema operativo/conjunto. Los fallos de las comprobaciones de versión de QA bloquean la validación normal de la versión, incluida la desviación de las herramientas dinámicas de OpenClaw en el carril del par de entornos de ejecución principal. Las ejecuciones alfa de Tideclaw aún pueden tratar como consultivos los carriles de comprobación de versión que no afecten a la seguridad del paquete. Con `release_profile=beta`, los conjuntos de proveedores en vivo `Run repo/live E2E validation` son consultivos (advertencias, no bloqueos); los perfiles estable y completo mantienen su carácter bloqueante. Cuando `live_suite_filter` solicita explícitamente un carril en vivo de QA sujeto a control, como Discord, WhatsApp o Slack, debe estar habilitada la variable de repositorio `OPENCLAW_RELEASE_QA_*_LIVE_CI_ENABLED` correspondiente; de lo contrario, falla la captura de entradas en vez de omitir silenciosamente el carril.

### Vitest

La caja de Vitest es el flujo de trabajo secundario manual `CI`. La CI manual omite deliberadamente el ámbito basado en cambios y fuerza el grafo normal de pruebas para la candidata a versión: fragmentos de Linux Node, fragmentos de plugins incluidos, fragmentos de contratos de plugins y canales, compatibilidad con Node 22, `check-*`, `check-additional-*`, comprobaciones rápidas de artefactos compilados, comprobaciones de documentación, Skills de Python, Windows, macOS e i18n de Control UI. Android se incluye cuando `Full Release Validation` ejecuta la caja porque el conjunto pasa `include_android=true`; la CI manual independiente requiere `include_android=true` para cubrir Android.

Utilice esta caja para responder «¿el árbol de código fuente superó el conjunto completo de pruebas normales?». No equivale a la validación del producto en la ruta de la versión. Evidencia que debe conservarse:

- resumen de `Full Release Validation` que muestra la URL de la ejecución de `CI` iniciada
- ejecución de `CI` correcta en el SHA de destino exacto
- nombres de fragmentos fallidos o lentos de los trabajos de CI al investigar regresiones
- artefactos de tiempos de Vitest, como `.artifacts/vitest-shard-timings.json`, cuando una ejecución requiere un análisis de rendimiento

Ejecute directamente la CI manual solo cuando la versión necesite una CI normal determinista, pero no las cajas de Docker, QA Lab, ejecución en vivo, sistemas operativos cruzados o paquetes. Utilice el primer comando para una CI directa sin Android. Añada `include_android=true` cuando la CI directa de la candidata a versión deba cubrir Android:

```bash
gh workflow run ci.yml --ref main -f target_ref=release/YYYY.M.PATCH
gh workflow run ci.yml --ref main -f target_ref=release/YYYY.M.PATCH -f include_android=true
```

### Docker

La caja de Docker se encuentra en `OpenClaw Release Checks` mediante `openclaw-live-and-e2e-checks-reusable.yml`, además del flujo de trabajo en modo de versión `install-smoke`. Valida la candidata a versión mediante entornos Docker empaquetados, en lugar de limitarse a pruebas en el nivel del código fuente.

La cobertura de Docker para la versión incluye:

- comprobación rápida de instalación completa con la comprobación lenta de instalación global de Bun habilitada
- preparación/reutilización de la imagen de comprobación rápida del Dockerfile raíz por SHA de destino, con los trabajos de comprobación rápida de QR, raíz/Gateway e instalador/Bun ejecutados como fragmentos independientes de comprobación de instalación
- carriles E2E del repositorio
- segmentos de Docker de la ruta de la versión: `core`, `package-update-openai`, `package-update-anthropic`, `package-update-core`, `plugins-runtime-plugins`, `plugins-runtime-services`, de `plugins-runtime-install-a` a `plugins-runtime-install-h` y `openwebui`
- cobertura de OpenWebUI en un ejecutor dedicado con disco de gran capacidad cuando se solicite
- carriles divididos de instalación/desinstalación de plugins incluidos, de `bundled-plugin-install-uninstall-0` a `bundled-plugin-install-uninstall-23`
- conjuntos de proveedores en vivo/E2E y cobertura de modelos en vivo de Docker cuando las comprobaciones de versión incluyen conjuntos en vivo

Utilice los artefactos de Docker antes de repetir la ejecución. El planificador de la ruta de la versión carga `.artifacts/docker-tests/` con registros de los carriles, `summary.json`, `failures.json`, tiempos de las fases, el JSON del plan del planificador y comandos de repetición. Para una recuperación específica, utilice `docker_lanes=<lane[,lane]>` en el flujo de trabajo reutilizable en vivo/E2E en lugar de repetir todos los segmentos de la versión. Los comandos de repetición generados incluyen las entradas anteriores de `package_artifact_run_id` y de las imágenes Docker preparadas cuando están disponibles, por lo que un carril fallido puede reutilizar el mismo tarball y las mismas imágenes de GHCR.

### QA Lab

La caja de QA Lab también forma parte de `OpenClaw Release Checks`. Es el control de la versión para el comportamiento agéntico y en el nivel de canal, independiente de Vitest y de la mecánica de paquetes de Docker.

La cobertura de QA Lab para la versión incluye:

- carril de paridad simulada que compara el carril candidato de OpenAI con la referencia `anthropic/claude-opus-4-8` mediante el paquete de paridad agéntica
- perfil de versión del adaptador en vivo de Matrix mediante el entorno `qa-live-shared`
- carril de QA en vivo de Telegram mediante arrendamientos de credenciales de CI de Convex
- `pnpm qa:otel:smoke`, `pnpm qa:otel:collector-smoke`, `pnpm qa:prometheus:smoke` o `pnpm qa:observability:smoke` cuando la telemetría de la versión requiera una prueba local explícita

Utilice esta caja para responder «¿la versión se comporta correctamente en los escenarios de QA y los flujos de canales en vivo?». Conserve las URL de los artefactos de los carriles de paridad, Matrix y Telegram al aprobar la versión. La cobertura completa de Matrix sigue disponible como una ejecución manual fragmentada de QA Lab, en lugar de ser el carril crítico predeterminado de la versión.

### Paquete

La caja de paquetes es el control del producto instalable. Está respaldada por `Package Acceptance` y el solucionador `scripts/resolve-openclaw-package-candidate.mjs`. El solucionador normaliza una candidata en el tarball `package-under-test` que consume Docker E2E, valida el inventario del paquete, registra la versión y el SHA-256 del paquete y mantiene la referencia del arnés del flujo de trabajo separada de la referencia del código fuente del paquete.

Orígenes de candidatas admitidos:

- `source=npm`: `openclaw@beta`, `openclaw@latest` o una versión exacta de OpenClaw
- `source=ref`: empaquetar una rama, etiqueta o SHA de confirmación completo `package_ref` de confianza con el arnés `workflow_ref` seleccionado
- `source=url`: descargar un `.tgz` HTTPS público con el `package_sha256` obligatorio; se rechazan las credenciales en la URL, los puertos HTTPS no predeterminados, los nombres de host o direcciones resueltas privados/internos/de uso especial y las redirecciones no seguras
- `source=trusted-url`: descargar un `.tgz` HTTPS con los `package_sha256` y `trusted_source_id` obligatorios desde una política con nombre en `.github/package-trusted-sources.json`; utilice esta opción para réplicas empresariales propiedad de mantenedores o repositorios de paquetes privados, en lugar de añadir a `source=url` una omisión de red privada en el nivel de entrada
- `source=artifact`: reutilizar un `.tgz` cargado por otra ejecución de GitHub Actions

`OpenClaw Release Checks` ejecuta Package Acceptance con `source=artifact`, el artefacto preparado del paquete de la versión, `suite_profile=custom`, `docker_lanes=doctor-switch update-channel-switch skill-install update-corrupt-plugin upgrade-survivor published-upgrade-survivor root-managed-vps-upgrade update-restart-auth plugins-offline plugin-update plugin-binding-command-escape`, `telegram_mode=mock-openai`. Package Acceptance conserva la migración, la actualización, la actualización de VPS administrados por el usuario raíz, el reinicio tras actualizar con autenticación configurada, la instalación en vivo de Skills de ClawHub, la limpieza de dependencias obsoletas de plugins, los accesorios de plugins sin conexión, la actualización de plugins, el refuerzo del escape de vinculaciones de comandos de plugins y la QA del paquete de Telegram con el mismo tarball resuelto. Las comprobaciones bloqueantes de la versión utilizan como referencia predeterminada el paquete publicado más reciente; el perfil beta con `run_release_soak=true`, `release_profile=stable` o `release_profile=full` amplía la comprobación de supervivencia a las actualizaciones publicadas a `last-stable-4`, además de las referencias fijadas `2026.4.23`, `2026.5.2` y `2026.4.15` con escenarios `reported-issues`. Utilice Package Acceptance con `source=npm` para una candidata ya publicada, `source=ref` para un tarball local de npm respaldado por un SHA antes de publicarlo, `source=trusted-url` para una réplica empresarial/privada propiedad de mantenedores o `source=artifact` para un tarball preparado y cargado por otra ejecución de GitHub Actions.

Es el reemplazo nativo de GitHub para la mayor parte de la cobertura de paquetes/actualizaciones que antes requería Parallels. Las comprobaciones de versión entre sistemas operativos siguen siendo importantes para la incorporación, el instalador y el comportamiento específico de cada plataforma, pero la validación del producto para paquetes/actualizaciones debe preferir Package Acceptance.

La lista de comprobación canónica para validar actualizaciones y plugins es [Pruebas de actualizaciones y plugins](/es/help/testing-updates-plugins). Utilícela para decidir qué carril local, de Docker, de Package Acceptance o de comprobación de versión demuestra un cambio de instalación/actualización de plugins, limpieza de doctor o migración de un paquete publicado. La migración exhaustiva de actualizaciones publicadas desde cada paquete estable `2026.4.23+` es un flujo de trabajo manual independiente `Update Migration`, no forma parte de la CI completa de la versión.

La tolerancia heredada de aceptación de paquetes está deliberadamente limitada en el tiempo. Los paquetes hasta `2026.4.25` pueden utilizar la ruta de compatibilidad para las carencias de metadatos ya publicadas en npm: entradas privadas del inventario de QA ausentes del tarball, ausencia de `gateway install --wrapper`, archivos de parche ausentes en el accesorio de git derivado del tarball, ausencia persistente de `update.channel`, ubicaciones heredadas de los registros de instalación de plugins, ausencia de persistencia de registros de instalación del marketplace y migración de metadatos de configuración durante `plugins update`. El paquete publicado `2026.4.26` puede advertir sobre archivos de sello de metadatos de compilación local ya publicados. Los paquetes posteriores deben satisfacer los contratos modernos de paquetes; esas mismas carencias hacen que falle la validación de la versión.

Utilice perfiles más amplios de Package Acceptance cuando la pregunta de la versión se refiera a un paquete instalable real:

```bash
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=npm \
  -f package_spec=openclaw@beta \
  -f suite_profile=product \
  -f published_upgrade_survivor_baseline=openclaw@2026.4.26
```

Perfiles de paquete habituales:

- `smoke`: vías rápidas de instalación de paquetes/canales/agentes, red del Gateway y recarga de configuración
- `package`: contratos de instalación/actualización/reinicio/paquetes de plugins, además de prueba en vivo de instalación de Skills de ClawHub; esta es la opción predeterminada para la comprobación de versiones
- `product`: `package` más canales MCP, limpieza de cron/subagentes, búsqueda web de OpenAI y OpenWebUI
- `full`: segmentos de la ruta de publicación de Docker con OpenWebUI
- `custom`: lista exacta de `docker_lanes` para repeticiones de ejecución específicas

Para la prueba de Telegram del paquete candidato, habilite `telegram_mode=mock-openai` o `telegram_mode=live-frontier` en Package Acceptance. El flujo de trabajo pasa el tarball `package-under-test` resuelto a la vía de Telegram; el flujo de trabajo independiente de Telegram sigue aceptando una especificación de npm publicada para las comprobaciones posteriores a la publicación.

## Automatización de publicación de versiones regulares

Para la publicación beta, de `latest`, plugins, GitHub Release y plataformas,
`OpenClaw Release Publish` es el punto de entrada normal con mutaciones. La ruta mensual
de estabilidad extendida del Gateway `.33+` no utiliza este orquestador. El
flujo de trabajo regular orquesta los flujos de trabajo de publicación de confianza en el orden que
requiere la versión:

1. Extraiga la etiqueta de la versión y resuelva el SHA de su commit.
2. Verifique que se pueda acceder a la etiqueta desde `main` o `release/*` (o desde una rama alfa de Tideclaw para versiones preliminares alfa).
3. Ejecute `pnpm plugins:sync:check`.
4. Despache `Plugin NPM Release` con `publish_scope=all-publishable` y `ref=<release-sha>`.
5. Despache `Plugin ClawHub Release` con el mismo alcance y SHA.
6. Despache `OpenClaw NPM Release` con la etiqueta de versión, la etiqueta de distribución de npm y el `preflight_run_id` guardado, después de verificar el `full_release_validation_run_id` guardado y el intento de ejecución exacto.
7. Para versiones estables, cree o actualice la versión de GitHub como borrador, despache `Windows Node Release` con el `windows_node_tag` explícito y el `windows_node_installer_digests` aprobado para el candidato, y verifique los activos canónicos del instalador de Windows y sus sumas de comprobación. Despache también `Android Release` para compilar el APK firmado de la etiqueta exacta, junto con la suma de comprobación y la procedencia. Verifique ambos contratos de activos nativos antes de publicar el borrador.

Ejemplo de publicación beta:

```bash
gh workflow run openclaw-release-publish.yml \
  --ref main \
  -f tag=vYYYY.M.PATCH-beta.N \
  -f preflight_run_id=<successful-openclaw-npm-preflight-run-id> \
  -f full_release_validation_run_id=<successful-full-release-validation-run-id> \
  -f full_release_validation_run_attempt=<successful-full-release-validation-run-attempt> \
  -f npm_dist_tag=beta
```

Publicación estable en la etiqueta de distribución beta predeterminada:

```bash
gh workflow run openclaw-release-publish.yml \
  --ref main \
  -f tag=vYYYY.M.PATCH \
  -f windows_node_tag=vX.Y.Z \
  -f windows_node_installer_digests='{"OpenClawCompanion-Setup-x64.exe":"sha256:<approved-x64-sha256>","OpenClawCompanion-Setup-arm64.exe":"sha256:<approved-arm64-sha256>"}' \
  -f preflight_run_id=<successful-openclaw-npm-preflight-run-id> \
  -f full_release_validation_run_id=<successful-full-release-validation-run-id> \
  -f full_release_validation_run_attempt=<successful-full-release-validation-run-attempt> \
  -f npm_dist_tag=beta
```

La promoción estable directamente a `latest` es explícita:

```bash
gh workflow run openclaw-release-publish.yml \
  --ref main \
  -f tag=vYYYY.M.PATCH \
  -f windows_node_tag=vX.Y.Z \
  -f windows_node_installer_digests='{"OpenClawCompanion-Setup-x64.exe":"sha256:<approved-x64-sha256>","OpenClawCompanion-Setup-arm64.exe":"sha256:<approved-arm64-sha256>"}' \
  -f preflight_run_id=<successful-openclaw-npm-preflight-run-id> \
  -f full_release_validation_run_id=<successful-full-release-validation-run-id> \
  -f full_release_validation_run_attempt=<successful-full-release-validation-run-attempt> \
  -f npm_dist_tag=latest
```

Utilice los flujos de trabajo de nivel inferior `Plugin NPM Release` y `Plugin ClawHub Release` únicamente para trabajos específicos de reparación o republicación. `OpenClaw Release Publish` rechaza `plugin_publish_scope=selected` cuando `publish_openclaw_npm=true`, para que el paquete principal no pueda publicarse sin todos los plugins oficiales publicables, incluido `@openclaw/diffs-language-pack`. Para reparar un plugin seleccionado, configure `publish_openclaw_npm=false` con `plugin_publish_scope=selected` y `plugins=@openclaw/name`, o despache directamente el flujo de trabajo secundario.

La inicialización de ClawHub en la primera publicación es la excepción: despache `Plugin ClawHub New`
desde el `main` de confianza y pase el SHA completo de la versión de destino mediante `ref`.
Nunca ejecute el propio flujo de trabajo de inicialización desde la etiqueta o rama de la versión:

```bash
gh workflow run plugin-clawhub-new.yml \
  --ref main \
  -f plugins=@openclaw/name \
  -f ref=<full-40-character-release-sha> \
  -f pretag_validation=true \
  -f dry_run=true
```

La validación previa a la etiqueta requiere `dry_run=true`, rechaza las entradas de etiqueta de versión y ejecución principal,
y solo acepta un destino exacto accesible desde `main` o `release/*`.
No carga credenciales de ClawHub, publica bytes de paquetes ni cambia la configuración
del publicador de confianza. El flujo de trabajo sigue resolviendo el plan del registro en vivo,
extrae y empaqueta el destino únicamente en un trabajo sin secretos, materializa la
cadena de herramientas bloqueada de ClawHub y valida el artefacto inmutable y el
slug/identidad del paquete antes de que exista la etiqueta de versión. Apruebe el entorno
`clawhub-plugin-bootstrap` solo después de que finalicen los trabajos de empaquetado sin secretos;
este trabajo de validación protegido no tiene credenciales ni comandos de mutación.

Una ejecución de prueba aprobada o una inicialización real después de etiquetar debe incluir la
etiqueta de versión exacta, además del identificador, el intento y la rama de la ejecución principal
`OpenClaw Release Publish`. La ejecución principal certifica el SHA de su propio flujo de trabajo y un SHA
de confianza `main` exacto e independiente para `Plugin ClawHub New`; la ejecución
secundaria y cada aprobación de entorno protegido deben coincidir con ese SHA secundario aprobado.
La etiqueta de versión se vuelve a comprobar antes de cada intento de publicación y mutación
del publicador de confianza.

El trabajo de empaquetado
carga un artefacto inmutable cuyo nombre, identificador/resumen del artefacto de Actions,
ejecución/intento del productor, SHA de destino y SHA-256/tamaño del tarball de cada paquete
se transfieren a los trabajos de validación y protegidos. El trabajo protegido extrae únicamente
las herramientas de confianza de `main`, valida la tupla del artefacto mediante
la API de GitHub, descarga por identificador de artefacto exacto, vuelve a calcular el hash
de cada tarball y valida las rutas TAR locales y la identidad del paquete con las reglas de
canonicalización USTAR de la CLI fijada. Después, cada candidato supera la ejecución de prueba
de publicación de la CLI fijada, que finaliza antes de consultar el registro o realizar la autenticación.
El prefiltro del trabajo con credenciales limita los ClawPacks comprimidos a 120 MiB, la carga
total de archivos a 50 MiB, los datos TAR expandidos a 64 MiB y el número de entradas TAR a 10,000.
La reparación del publicador de confianza de paquetes existentes continúa siendo solo de configuración,
pero sigue empaquetando el destino y requiere la etiqueta solicitada, así como la igualdad exacta
de los bytes y metadatos del registro, antes de cambiar la configuración del publicador de confianza.
La verificación posterior a la publicación descarga el artefacto de ClawHub y exige el mismo SHA-256
y tamaño. Una recuperación mediante repetición de trabajos fallidos puede reutilizar el artefacto
de paquete de un intento anterior únicamente cuando el trabajo productor exacto haya finalizado
correctamente. La evidencia final también vincula la versión bloqueada de ClawHub, el SHA-256
del bloqueo y la integridad de npm. Una discrepancia requiere una nueva versión del paquete.

## Entradas del flujo de trabajo de NPM

`OpenClaw NPM Release` acepta estas entradas controladas por el operador:

- `tag`: etiqueta de versión obligatoria, como `v2026.4.2`, `v2026.4.2-1`, `v2026.4.2-beta.1` o `v2026.4.2-alpha.1`; cuando `preflight_only=true`, también puede ser el SHA del commit actual de 40 caracteres completos de la rama del flujo de trabajo para una comprobación previa exclusivamente de validación
- `preflight_only`: `true` solo para validación/compilación/empaquetado, `false` para la ruta de publicación real
- `preflight_run_id`: identificador de una ejecución previa correcta existente, obligatorio en la ruta de publicación real para que el flujo de trabajo reutilice el tarball preparado en lugar de volver a compilarlo
- `full_release_validation_run_id`: identificador de una ejecución correcta de `Full Release Validation` para esta etiqueta/SHA, obligatorio para la publicación real. Las publicaciones beta pueden continuar solo con la comprobación previa, con una advertencia, pero la promoción estable/`latest` sigue requiriéndolo.
- `full_release_validation_run_attempt`: intento de ejecución positivo exacto emparejado con `full_release_validation_run_id`; obligatorio siempre que se proporcione el identificador de ejecución, para que las repeticiones de ejecución no puedan cambiar la evidencia de autorización durante la publicación.
- `release_publish_run_id`: identificador de ejecución aprobado de `OpenClaw Release Publish`; obligatorio cuando este flujo de trabajo lo despacha ese flujo principal (llamadas de publicación real del actor bot)
- `plugin_npm_run_id`: identificador de ejecución correcta con encabezado exacto de `Plugin NPM Release`; obligatorio para una publicación real del núcleo `extended-stable`
- `npm_dist_tag`: etiqueta de destino de npm para la ruta de publicación; acepta `alpha`, `beta`, `latest` o `extended-stable`, y su valor predeterminado es `beta`. El parche final `33` y los posteriores deben usar `extended-stable`; de forma predeterminada, `extended-stable` rechaza los parches anteriores y siempre rechaza las etiquetas no finales.
- `bypass_extended_stable_guard`: booleano exclusivo para pruebas, con valor predeterminado `false`; con `npm_dist_tag=extended-stable`, omite los requisitos mensuales de estabilidad extendida, pero conserva las comprobaciones de identidad de la versión, artefactos, aprobación y lectura posterior.

`Plugin NPM Release` acepta `npm_dist_tag=default` para el comportamiento de versiones
existente o `npm_dist_tag=extended-stable` para la ruta mensual protegida. La
opción de estabilidad extendida requiere `publish_scope=all-publishable`, una entrada
`plugins` vacía, un parche final igual o superior a `33` y la rama canónica
`extended-stable/YYYY.M.33` en su punta exacta. Nunca desplaza los plugins
`latest` ni `beta`. Las versiones nuevas de paquetes reciben `extended-stable` de forma atómica
mediante publicación de confianza OIDC (`npm publish --tag extended-stable`); este
flujo de trabajo de origen no utiliza `npm dist-tag add` autenticado mediante token. Los reintentos
omiten las versiones exactas que ya están presentes en npm y, después, se cierran de forma segura,
a menos que la lectura posterior completa confirme que cada paquete exacto y cada etiqueta
`extended-stable` han convergido.

`OpenClaw Release Publish` acepta estas entradas controladas por el operador:

- `tag`: etiqueta de versión obligatoria; ya debe existir
- `preflight_run_id`: identificador de ejecución previa correcta de `OpenClaw NPM Release`; obligatorio cuando `publish_openclaw_npm=true` o `plugin_publish_scope=all-publishable`
- `full_release_validation_run_id`: identificador de ejecución correcta de `Full Release Validation`; obligatorio cuando `publish_openclaw_npm=true` o `plugin_publish_scope=all-publishable`
- `full_release_validation_run_attempt`: intento positivo exacto emparejado con `full_release_validation_run_id`; obligatorio siempre que se proporcione el identificador de ejecución
- `windows_node_tag`: etiqueta de versión exacta de `openclaw/openclaw-windows-node` que no sea una versión preliminar; obligatoria para la publicación estable de OpenClaw
- `windows_node_installer_digests`: mapa JSON compacto aprobado para el candidato que asigna los nombres actuales de los instaladores de Windows a sus resúmenes `sha256:` fijados; obligatorio para la publicación estable de OpenClaw
- `npm_telegram_run_id`: identificador opcional de una ejecución correcta de `NPM Telegram Beta E2E` para incluirlo en la evidencia final de la versión
- `npm_dist_tag`: etiqueta de destino de npm para el paquete OpenClaw, una de `alpha`, `beta` o `latest`
- `plugin_publish_scope`: su valor predeterminado es `all-publishable`; utilice `selected` únicamente para trabajos específicos de reparación exclusiva de plugins con `publish_openclaw_npm=false`
- `plugins`: nombres de paquetes `@openclaw/*` separados por comas cuando `plugin_publish_scope=selected`
- `publish_openclaw_npm`: su valor predeterminado es `true`; establezca `false` únicamente cuando utilice el flujo de trabajo como orquestador de reparación exclusiva de plugins
- `release_profile`: perfil de cobertura de la versión utilizado para los resúmenes de evidencia de la versión; su valor predeterminado es `from-validation`, que lo lee del manifiesto de validación, o se puede sustituir por `beta`, `stable` o `full`
- `wait_for_clawhub`: su valor predeterminado es `false` para que la disponibilidad de npm no quede bloqueada por el proceso auxiliar de ClawHub; establezca `true` únicamente cuando la finalización del flujo de trabajo deba incluir la finalización de ClawHub

`OpenClaw Release Checks` acepta estas entradas controladas por el operador:

- `ref`: rama, etiqueta o SHA completo del commit que se validará. Las comprobaciones que requieren secretos exigen que el commit resuelto sea accesible desde una rama o etiqueta de versión de OpenClaw.
- `run_release_soak`: habilita las comprobaciones exhaustivas en vivo/E2E, la ruta de lanzamiento de Docker y la prueba prolongada de supervivencia a actualizaciones desde todas las versiones para las comprobaciones de versiones beta. Se activa obligatoriamente mediante `release_profile=stable` y `release_profile=full`.

Reglas:

- Las versiones finales normales y las versiones de corrección inferiores al parche `33` pueden publicarse en `beta` o `latest`. Las versiones finales con el parche `33` o superior deben publicarse en `extended-stable`, y se rechazan las versiones con sufijo de corrección en ese límite.
- Las etiquetas de prelanzamiento beta solo pueden publicarse en `beta`; las etiquetas de prelanzamiento alfa solo pueden publicarse en `alpha`
- Para `OpenClaw NPM Release`, solo se permite introducir el SHA completo del commit cuando `preflight_only=true`
- `OpenClaw Release Checks` y `Full Release Validation` son siempre exclusivamente de validación
- La ruta de publicación real debe usar el mismo `npm_dist_tag` utilizado durante la comprobación previa; el flujo de trabajo verifica esos metadatos antes de continuar con la publicación

## Secuencia normal de lanzamiento beta/última versión estable

Esta secuencia heredada corresponde al lanzamiento orquestado normal, que también abarca los plugins, la versión de GitHub, Windows y el trabajo para otras plataformas. No es la ruta mensual de estabilidad extendida del Gateway `.33+` documentada al principio de esta página.

Al preparar un lanzamiento estable orquestado normal:

1. Ejecute `OpenClaw NPM Release` con `preflight_only=true`. Antes de que exista una etiqueta, puede usar el SHA completo del commit actual de la rama del flujo de trabajo para una ejecución en seco, exclusivamente de validación, del flujo de trabajo de comprobación previa.
2. Elija `npm_dist_tag=beta` para el flujo normal que comienza con una beta, o `latest` solo cuando se quiera realizar intencionadamente una publicación estable directa.
3. Ejecute `Full Release Validation` en la rama de lanzamiento, la etiqueta de versión o el SHA completo del commit cuando se necesite la CI normal junto con cobertura de caché de prompts en vivo, Docker, QA Lab, Matrix y Telegram desde un único flujo de trabajo manual. Si intencionadamente solo se necesita el grafo determinista de pruebas normales, ejecute en su lugar el flujo de trabajo manual `CI` en la referencia de lanzamiento.
4. Seleccione la etiqueta de versión `openclaw/openclaw-windows-node` exacta que no sea de prelanzamiento y cuyos instaladores firmados para x64 y ARM64 deban distribuirse. Guárdela como `windows_node_tag` y guarde su mapa de resúmenes validado como `windows_node_installer_digests`. La herramienta auxiliar de la versión candidata registra ambos valores y los incluye en el comando de publicación que genera.
5. Guarde los valores correctos de `preflight_run_id`, `full_release_validation_run_id` y el `full_release_validation_run_attempt` exacto.
6. Ejecute `OpenClaw Release Publish` desde el `main` de confianza con el mismo `tag`, el mismo `npm_dist_tag`, el `windows_node_tag` seleccionado, su `windows_node_installer_digests` guardado, el `preflight_run_id` guardado, `full_release_validation_run_id` y `full_release_validation_run_attempt`. Publica los plugins externalizados en npm y ClawHub antes de promover el paquete npm de OpenClaw.
7. Si el lanzamiento se publicó en `beta`, use el flujo de trabajo `openclaw/releases/.github/workflows/openclaw-npm-dist-tags.yml` para promover esa versión estable de `beta` a `latest`.
8. Si el lanzamiento se publicó intencionadamente de forma directa en `latest` y `beta` debe adoptar de inmediato la misma compilación estable, use ese mismo flujo de trabajo de lanzamiento para hacer que ambas etiquetas de distribución apunten a la versión estable, o permita que su sincronización autorreparable programada mueva `beta` más adelante.

La modificación de las etiquetas de distribución reside en el repositorio del registro de lanzamientos porque todavía requiere `NPM_TOKEN`, mientras que el repositorio de código fuente mantiene una publicación exclusivamente mediante OIDC. De este modo, tanto la ruta de publicación directa como la ruta de promoción que comienza con una beta permanecen documentadas y visibles para los operadores.

Si un mantenedor debe recurrir a la autenticación local de npm, ejecute cualquier comando de la CLI de 1Password (`op`) únicamente dentro de una sesión de tmux dedicada. No invoque `op` directamente desde el shell principal del agente; mantenerlo dentro de tmux permite observar los mensajes, las alertas y la gestión de OTP, y evita que se repitan las alertas del host.

## Referencias públicas

- [`.github/workflows/full-release-validation.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/full-release-validation.yml)
- [`.github/workflows/package-acceptance.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/package-acceptance.yml)
- [`.github/workflows/openclaw-npm-release.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-npm-release.yml)
- [`.github/workflows/openclaw-release-checks.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-release-checks.yml)
- [`.github/workflows/openclaw-cross-os-release-checks-reusable.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/openclaw-cross-os-release-checks-reusable.yml)
- [`.github/workflows/docker-release.yml`](https://github.com/openclaw/openclaw/blob/main/.github/workflows/docker-release.yml)
- [`scripts/resolve-openclaw-package-candidate.mjs`](https://github.com/openclaw/openclaw/blob/main/scripts/resolve-openclaw-package-candidate.mjs)
- [`scripts/openclaw-npm-release-check.ts`](https://github.com/openclaw/openclaw/blob/main/scripts/openclaw-npm-release-check.ts)
- [`scripts/package-mac-dist.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/package-mac-dist.sh)
- [`scripts/make_appcast.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/make_appcast.sh)

Los mantenedores usan la documentación privada de lanzamientos de [`openclaw/maintainers/release/README.md`](https://github.com/openclaw/maintainers/blob/main/release/README.md) como guía operativa real.

## Contenido relacionado

- [Canales de lanzamiento](/es/install/development-channels)
