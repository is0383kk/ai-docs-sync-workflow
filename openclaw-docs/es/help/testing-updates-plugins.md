---
read_when:
    - Cambiar el comportamiento de actualización, doctor, aceptación de paquetes o instalación de plugins de OpenClaw
    - Preparación o aprobación de una versión candidata
    - Depuración de actualizaciones de paquetes, limpieza de dependencias de plugins o regresiones en la instalación de plugins
sidebarTitle: Update and plugin tests
summary: Cómo valida OpenClaw las rutas de actualización, las migraciones de paquetes y el comportamiento de instalación y actualización de plugins
title: 'Pruebas: actualizaciones y plugins'
x-i18n:
    generated_at: "2026-07-26T05:09:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 96a11fe42472f758d4fd1cc568486e301f7460982fdb547cab8b39de04a8dabe
    source_path: help/testing-updates-plugins.md
    workflow: 16
---

Lista de comprobación para la validación de actualizaciones y plugins: demostrar que el paquete instalable puede
actualizar el estado real del usuario, reparar el estado heredado obsoleto mediante `doctor` y seguir
instalando, cargando, actualizando y desinstalando plugins desde todas las fuentes compatibles.

Para consultar el mapa general del ejecutor de pruebas, véase [Pruebas](/es/help/testing). Para las claves de proveedores
en vivo y las suites que acceden a la red, véase [Pruebas en vivo](/es/help/testing-live).

## Qué protegemos

- Un tarball de paquete está completo, tiene un `dist/postinstall-inventory.json` válido
  y no depende de archivos del repositorio sin empaquetar.
- Un usuario puede pasar de un paquete publicado anterior al paquete candidato
  sin perder la configuración, los agentes, las sesiones, los espacios de trabajo, las listas de plugins permitidos ni
  la configuración de canales.
- `openclaw doctor --fix --non-interactive` controla las rutas de limpieza y reparación
  heredadas. El inicio no debe acumular migraciones de compatibilidad ocultas para el estado
  obsoleto de los plugins.
- Las instalaciones de plugins funcionan desde directorios locales, repositorios git, paquetes npm y la
  ruta del registro de ClawHub.
- Las dependencias npm de los plugins se instalan en un proyecto npm administrado por plugin,
  se analizan antes de establecer la confianza y se eliminan mediante `npm uninstall` durante
  la desinstalación del plugin para que las dependencias elevadas no permanezcan.
- La actualización de un plugin no realiza ninguna operación cuando no ha cambiado nada: los registros de instalación, la fuente
  resuelta, la disposición de las dependencias instaladas y el estado de activación permanecen intactos.

## Verificación local durante el desarrollo

Comience por un alcance reducido:

```bash
pnpm changed:lanes --json
pnpm check:changed
pnpm test:changed
```

Para cambios en la instalación, desinstalación, dependencias o inventario de paquetes de plugins, ejecute también
las pruebas específicas que cubren el punto de integración editado:

```bash
pnpm test src/plugins/uninstall.test.ts src/infra/package-dist-inventory.test.ts test/scripts/package-acceptance-workflow.test.ts
```

Antes de que cualquier carril Docker de paquetes consuma un tarball, verifique el artefacto del paquete:

```bash
pnpm release:check
```

`release:check` ejecuta comprobaciones de divergencias de configuración, documentación y API (esquema de configuración, línea base de la documentación de configuración,
manifiesto del contrato de API y exportaciones del SDK de plugins, versiones e inventario de plugins),
escribe el inventario de distribución del paquete, ejecuta `npm pack --dry-run`, rechaza los
archivos empaquetados prohibidos, instala el tarball en un prefijo temporal, ejecuta postinstall y
realiza pruebas rápidas de los puntos de entrada de los canales incluidos.

## Carriles Docker

Los carriles Docker constituyen la verificación a nivel de producto. Instalan o actualizan un
paquete real dentro de contenedores Linux y verifican el comportamiento mediante comandos de la CLI,
el inicio del Gateway, sondas HTTP, el estado RPC y el estado del sistema de archivos.

Use carriles específicos durante las iteraciones:

```bash
pnpm test:docker:plugins
pnpm test:docker:plugin-lifecycle-matrix
pnpm test:docker:plugin-update
pnpm test:docker:upgrade-survivor
pnpm test:docker:published-upgrade-survivor
pnpm test:docker:update-restart-auth
pnpm test:docker:update-migration
```

Carriles importantes:

- `test:docker:plugins` cubre las pruebas rápidas de instalación de plugins, las instalaciones desde carpetas locales,
el comportamiento de omisión de actualizaciones de carpetas locales, las carpetas locales con
dependencias preinstaladas, las instalaciones de paquetes `file:`, las instalaciones desde git con ejecución mediante la CLI, las actualizaciones
de referencias móviles de git, las instalaciones desde el registro npm con dependencias transitivas
elevadas, las actualizaciones npm sin operaciones, el rechazo de metadatos de paquetes npm malformados,
las instalaciones desde una instancia local de prueba de ClawHub y las actualizaciones sin operaciones, el comportamiento de actualización del marketplace
y la activación e inspección del paquete de Claude. Establezca `OPENCLAW_PLUGINS_E2E_CLAWHUB=0` para
mantener el bloque de ClawHub hermético y sin conexión.
- `test:docker:plugin-lifecycle-matrix` instala el paquete candidato en un contenedor
  vacío y ejecuta un plugin npm durante la instalación, inspección, desactivación, activación,
  actualización explícita, reversión explícita y desinstalación después de eliminar el código del plugin.
  Registra métricas de RSS y CPU por fase.
- `test:docker:plugin-update` valida que un plugin instalado sin cambios
  no se reinstale ni pierda los metadatos de instalación durante `openclaw plugins update`.
- `test:docker:upgrade-survivor` instala el tarball candidato sobre una instancia de prueba
  de un usuario anterior con estado no limpio, ejecuta la actualización del paquete junto con doctor en modo no interactivo y, a continuación, inicia
  un Gateway de bucle invertido y comprueba que se conserve el estado.
- `test:docker:published-upgrade-survivor` instala primero una línea base publicada,
  la configura mediante una receta `openclaw config set` integrada, la actualiza al
  tarball candidato, ejecuta doctor, comprueba la limpieza heredada, inicia el Gateway y
  sondea `/healthz`, `/readyz` y el estado RPC.
- `test:docker:update-restart-auth` instala el paquete candidato, inicia un
  Gateway administrado con autenticación por token, elimina del entorno la autenticación del Gateway del invocador para
  `openclaw update --yes --json` y exige que el comando de actualización candidato
  reinicie el Gateway antes de las sondas normales.
- `test:docker:update-migration` es el carril de actualización publicada con limpieza intensiva. Parte
  de un estado de usuario configurado al estilo de Discord/Telegram, ejecuta doctor en la línea base
  para que las dependencias de los plugins configurados tengan la oportunidad de materializarse, introduce
  residuos heredados de dependencias de plugins para un plugin empaquetado configurado, actualiza al
  tarball candidato y exige que doctor, tras la actualización, elimine las raíces de dependencias heredadas.

Variantes útiles de supervivencia a actualizaciones publicadas:

```bash
OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPEC=openclaw@2026.4.23 \
OPENCLAW_UPGRADE_SURVIVOR_SCENARIO=versioned-runtime-deps \
pnpm test:docker:published-upgrade-survivor

OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPEC=openclaw@latest \
OPENCLAW_UPGRADE_SURVIVOR_SCENARIO=bootstrap-persona \
pnpm test:docker:published-upgrade-survivor
```

Escenarios disponibles: `base`, `acpx-openclaw-tools-bridge`, `feishu-channel`,
`bootstrap-persona`, `channel-post-core-restore`, `plugin-deps-cleanup`,
`configured-plugin-installs`, `stale-source-plugin-shadow`, `tilde-log-path`
y `versioned-runtime-deps`. En ejecuciones agregadas, `OPENCLAW_UPGRADE_SURVIVOR_SCENARIOS=reported-issues`
(alias `far-reaching`) se expande a todos los escenarios, incluida la
migración de instalación de plugins configurados.

La migración completa de actualizaciones está separada intencionadamente del Pipeline de CI de versión completa. Use el
flujo de trabajo manual `Update Migration` cuando la pregunta de la versión sea: «¿pueden todas las
versiones estables publicadas desde 2026.4.23 en adelante actualizarse a este candidato y
limpiar los residuos de dependencias de plugins?»:

```bash
gh workflow run update-migration.yml \
  --ref main \
  -f workflow_ref=main \
  -f package_ref=main \
  -f baselines=all-since-2026.4.23 \
  -f scenarios=plugin-deps-cleanup
```

## Aceptación de paquetes

La aceptación de paquetes es la puerta de paquetes nativa de GitHub. Resuelve un
paquete candidato en un tarball `package-under-test`, registra la versión y el SHA-256 y, a continuación,
ejecuta carriles E2E reutilizables de Docker con ese tarball exacto. La referencia del entorno de
pruebas del flujo de trabajo está separada de la referencia de origen del paquete, por lo que la lógica de pruebas actual puede validar
versiones de confianza anteriores.

Fuentes de candidatos:

- `source=npm`: valida `openclaw@extended-stable`, `openclaw@beta`,
  `openclaw@latest` o una versión publicada exacta.
- `source=ref`: empaqueta una rama, etiqueta o confirmación de confianza con el entorno de
  pruebas actual seleccionado.
- `source=url`: valida un tarball HTTPS público con el valor obligatorio `package_sha256`.
  Esta ruta rechaza credenciales en la URL, puertos HTTPS no predeterminados, nombres de host o
  resultados DNS/IP privados o internos, espacios de direcciones IP de uso especial y redirecciones no seguras.
- `source=trusted-url`: valida un tarball HTTPS con los valores obligatorios
  `package_sha256` y `trusted_source_id` según la política controlada por los responsables de mantenimiento
  en `.github/package-trusted-sources.json`. Use esta opción para réplicas
  empresariales o privadas en lugar de debilitar `source=url` con un modificador de entrada para permitir destinos privados.
  La autenticación mediante portador, cuando está configurada por la política, utiliza el secreto fijo
  `OPENCLAW_TRUSTED_PACKAGE_TOKEN`.
- `source=artifact`: reutiliza un tarball cargado por otra ejecución de Actions.

La validación completa de la versión usa `source=artifact` de forma predeterminada, creado a partir del
SHA de la versión resuelta. Para la verificación posterior a la publicación, proporcione
`package_acceptance_package_spec=openclaw@YYYY.M.PATCH` para que la misma matriz de actualizaciones
se dirija en su lugar al paquete npm publicado.

Las comprobaciones de versiones invocan la aceptación de paquetes con el conjunto de paquete, actualización, reinicio y plugins:

```text
doctor-switch update-channel-switch skill-install update-corrupt-plugin upgrade-survivor published-upgrade-survivor root-managed-vps-upgrade update-restart-auth plugins-offline plugin-update plugin-binding-command-escape
```

Cuando se habilita la estabilización de la versión (forzada para `release_profile=stable` y
`full`), también proporcionan:

```text
published_upgrade_survivor_baselines=last-stable-4 2026.4.23 2026.5.2 2026.4.15
published_upgrade_survivor_scenarios=reported-issues
telegram_mode=mock-openai
```

Esto mantiene la migración de paquetes, el cambio de canal de actualización, la tolerancia a plugins administrados
dañados, la limpieza de dependencias obsoletas de plugins, la cobertura de plugins sin conexión, el
comportamiento de actualización de plugins y el control de calidad de paquetes de Telegram en el mismo artefacto resuelto, sin
hacer que la puerta predeterminada de paquetes de versión recorra todas las versiones publicadas.

`last-stable-4` se resuelve en las cuatro versiones estables más recientes de OpenClaw
publicadas en npm. La aceptación de paquetes de versión fija `2026.4.23` como el primer límite de compatibilidad
de actualización de plugins, `2026.5.2` como límite de cambios en la arquitectura de plugins y
`2026.4.15` como una línea base anterior de actualización publicada de 2026.4.1x; el solucionador
elimina los valores fijados duplicados que ya se encuentren entre las cuatro últimas. Para obtener una cobertura exhaustiva de la migración
de actualizaciones publicadas, use `all-since-2026.4.23` en el flujo de trabajo de migración de actualizaciones
independiente, en lugar del Pipeline de CI de versión completa. `release-history` sigue
disponible para un muestreo manual más amplio cuando también se desee incluir el punto de referencia anterior a la fecha heredada.

Cuando se seleccionan varias líneas base de supervivencia a actualizaciones publicadas, el flujo de trabajo
reutilizable de Docker divide cada línea base en su propia tarea específica del ejecutor. Cada
segmento de línea base sigue ejecutando el conjunto de escenarios seleccionado, pero los registros y artefactos permanecen
separados por línea base y el tiempo total queda limitado por el segmento más lento, en lugar de por una única
tarea serial de gran tamaño.

Ejecute manualmente un perfil de paquete al validar un candidato antes de la versión:

```bash
gh workflow run package-acceptance.yml \
  --ref main \
  -f workflow_ref=main \
  -f source=npm \
  -f package_spec=openclaw@beta \
  -f suite_profile=package \
  -f published_upgrade_survivor_baselines="last-stable-4 2026.4.23 2026.5.2 2026.4.15" \
  -f published_upgrade_survivor_scenarios=reported-issues \
  -f telegram_mode=mock-openai
```

Para un canario de estabilidad extendida publicado, establezca
`package_spec=openclaw@extended-stable`. La aceptación de paquetes resuelve ese
selector en un tarball exacto antes de ejecutar los carriles Docker.

Use `suite_profile=product` cuando la cuestión de la versión incluya canales MCP,
la limpieza de Cron o subagentes, la búsqueda web de OpenAI u OpenWebUI. Use `suite_profile=full`
solo cuando necesite una cobertura completa mediante Docker de la ruta de publicación.

## Valor predeterminado de la versión

Para los candidatos a versión, el conjunto de verificación predeterminado es:

1. `pnpm check:changed` y `pnpm test:changed` para regresiones a nivel de código fuente.
2. `pnpm release:check` para la integridad del artefacto del paquete.
3. El perfil `package` de aceptación de paquetes o los carriles personalizados de paquetes
   de comprobación de versiones para los contratos de instalación, actualización, reinicio y plugins.
4. Comprobaciones de versiones en varios sistemas operativos para el instalador, la incorporación y el comportamiento
   específicos de cada sistema operativo y plataforma.
5. Suites en vivo solo cuando la superficie modificada afecte al comportamiento del proveedor o del servicio
   alojado.

En los equipos de los responsables de mantenimiento, las puertas amplias y la verificación de productos mediante Docker o paquetes deben ejecutarse
en Testbox, salvo que se realice explícitamente una verificación local.

## Compatibilidad heredada

La flexibilidad de compatibilidad es limitada y temporal:

- Los paquetes hasta `2026.4.25`, incluido `2026.4.25-beta.*`, pueden tolerar
  carencias ya publicadas en los metadatos de paquetes durante la aceptación de paquetes.
- El paquete publicado `2026.4.26` puede emitir advertencias por archivos locales de sello de metadatos
  de compilación que ya se hayan publicado.
- Los paquetes posteriores deben satisfacer los contratos modernos. Las mismas carencias producen errores en lugar de
  advertencias u omisiones.

No añada nuevas migraciones de inicio para estas formas antiguas. Añada o amplíe una reparación de doctor
y, a continuación, compruébela con `upgrade-survivor`, `published-upgrade-survivor` o
`update-restart-auth` cuando el comando de actualización controle el reinicio.

## Añadir cobertura

Al cambiar el comportamiento de actualización o de los plugins, añada cobertura en la capa más baja que
pueda fallar por el motivo correcto:

- Lógica pura de rutas o metadatos: prueba unitaria junto al código fuente.
- Inventario de paquetes o comportamiento de archivos empaquetados: `package-dist-inventory` o prueba del
  verificador de tarballs.
- Comportamiento de instalación/actualización de la CLI: aserción o fixture del carril de Docker.
- Comportamiento de migración de versiones publicadas: escenario `published-upgrade-survivor`.
- Comportamiento de reinicio gestionado por la actualización: `update-restart-auth`.
- Comportamiento del registro o de la fuente de paquetes: fixture `test:docker:plugins` o servidor
  de fixtures de ClawHub.
- Comportamiento de la disposición o limpieza de dependencias: verifique tanto la ejecución en tiempo de ejecución como el
  límite del sistema de archivos. Las dependencias de npm pueden elevarse dentro del proyecto npm
  gestionado del plugin, por lo que las pruebas deben demostrar que ese proyecto se analiza y limpia
  en lugar de suponer que solo se procesa el árbol `node_modules` local del paquete del plugin.

Mantenga los nuevos fixtures de Docker herméticos de forma predeterminada. Utilice registros de fixtures locales y
paquetes falsos, a menos que el objetivo de la prueba sea el comportamiento del registro en vivo.

## Triaje de fallos

Comience por la identidad del artefacto:

- Resumen de `resolve_package` de Package Acceptance: fuente, versión, SHA-256 y
  nombre del artefacto.
- Artefactos de Docker: `.artifacts/docker-tests/**/summary.json`,
  `failures.json`, registros de los carriles y comandos de repetición.
- Resumen de supervivencia a la actualización: `.artifacts/upgrade-survivor/summary.json`,
  incluida la versión de referencia, la versión candidata, el escenario, los tiempos de las fases y
  la cobertura de las recetas de configuración.

Dé prioridad a repetir el carril exacto que falló con el mismo artefacto de paquete en lugar de
repetir todo el conjunto de la versión.
