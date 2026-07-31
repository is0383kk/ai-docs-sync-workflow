---
read_when:
    - Quieres actualizar de forma segura una copia de trabajo del código fuente
    - Está depurando la salida o las opciones de `openclaw update`
    - Necesita comprender el comportamiento abreviado de `--update`
summary: Referencia de la CLI para `openclaw update` (actualización de fuente relativamente segura + reinicio automático del Gateway)
title: Actualizar
x-i18n:
    generated_at: "2026-07-26T05:04:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b46696f6b9cba5c318f870bcb6c5ea8e0652940968da2ad85e86709fe4c11146
    source_path: cli/update.md
    workflow: 16
---

# `openclaw update`

Actualiza OpenClaw y cambia entre los canales stable/extended-stable/beta/dev.

Si se instaló mediante **npm/pnpm/bun** (instalación global, sin metadatos de git),
las actualizaciones siguen el flujo del gestor de paquetes descrito en
[Actualización](/es/install/updating).

## Uso

```bash
openclaw update
openclaw update status
openclaw update repair
openclaw update wizard
openclaw update --channel extended-stable
openclaw update --channel beta
openclaw update --channel dev
openclaw update --tag beta
openclaw update --tag main
openclaw update --dry-run
openclaw update --no-restart
openclaw update --yes
openclaw update --acknowledge-clawhub-risk
openclaw update --json
openclaw --update
```

`openclaw --update` se reescribe como `openclaw update` (útil para shells y
scripts de lanzamiento).

## Opciones

| Indicador                                        | Descripción                                                                                                                                                                                                                                                                                                                                  |
| ------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--no-restart`                                   | Omite el reinicio del servicio Gateway después de una actualización correcta. Las actualizaciones mediante el gestor de paquetes que sí reinician verifican que el servicio reiniciado indique la versión esperada antes de que el comando se complete correctamente.                                                                         |
| `--channel <stable\|extended-stable\|beta\|dev>` | Establece el canal de actualización y lo conserva después de que la actualización del núcleo se complete correctamente. Extended-stable solo está disponible mediante paquetes.                                                                                                                                                               |
| `--tag <dist-tag\|version\|spec>`                | Reemplaza el destino del paquete solo para esta actualización. No se puede combinar con un canal `extended-stable` efectivo, cuyo destino exacto verificado es obligatorio. Para otras instalaciones de paquetes, `main` se asigna a `github:openclaw/openclaw#main`; las especificaciones de origen de GitHub/git se empaquetan en un tarball temporal antes de la instalación global preparada con npm. |
| `--dry-run`                                      | Muestra una vista previa de las acciones previstas (flujo de canal/etiqueta/destino/reinicio) sin escribir la configuración, instalar, sincronizar plugins ni reiniciar.                                                                                                                                                                       |
| `--json`                                         | Imprime JSON `UpdateRunResult` legible por máquina. Incluye `postUpdate.plugins.warnings` cuando un plugin administrado necesita reparación, detalles de la alternativa de plugins del canal beta y `postUpdate.plugins.integrityDrifts` cuando se detecta una divergencia en los artefactos de plugins de npm durante la sincronización posterior a la actualización. |
| `--timeout <seconds>`                            | Tiempo de espera por paso. Valor predeterminado: `1800`.                                                                                                                                                                                                                                                                          |
| `--yes`                                          | Omite las solicitudes de confirmación (por ejemplo, la confirmación de una reversión de versión).                                                                                                                                                                                                                                             |
| `--acknowledge-clawhub-risk`                     | Permite que la sincronización de plugins posterior a la actualización continúe pese a las advertencias de confianza de la comunidad de ClawHub sin una solicitud interactiva. Sin esta opción, las versiones riesgosas de la comunidad se omiten y permanecen sin cambios cuando OpenClaw no puede solicitar confirmación. Los paquetes oficiales de ClawHub y las fuentes de plugins incluidas omiten esta solicitud. |

No existe ningún indicador `--verbose`. Se usa `--dry-run` para obtener una vista previa de las acciones previstas,
`--json` para obtener resultados legibles por máquina y `openclaw update status --json`
solo para consultar el canal y la disponibilidad. El nivel de detalle de la consola del Gateway (`--verbose`) y
el nivel de registro de archivos (`logging.level: "debug"`/`"trace"`) son controles independientes; véase
[Registro del Gateway](/es/gateway/logging).

<Note>
En el modo Nix (`OPENCLAW_NIX_MODE=1`), las ejecuciones de `openclaw update` que realizan modificaciones están deshabilitadas. En su lugar, se debe actualizar la fuente de Nix o la entrada del flake de esta instalación; para nix-openclaw, se debe usar la [Guía de inicio rápido](https://github.com/openclaw/nix-openclaw#quick-start) centrada en el agente. `openclaw update status` y `openclaw update --dry-run` permanecen en modo de solo lectura.
</Note>

<Warning>
Las reversiones de versión requieren confirmación porque las versiones anteriores pueden dañar la configuración.
Si la instalación ya migró las sesiones a SQLite, se deben restaurar los artefactos
archivados de transcripciones heredadas antes de iniciar una versión anterior basada en archivos. Véase
[Doctor: Reversión de versión tras la migración de sesiones a SQLite](/es/cli/doctor#downgrading-after-session-sqlite-migration).
</Warning>

## `update status`

Muestra el canal de actualización activo, la etiqueta/rama/SHA de git (solo en checkouts de código fuente)
y la disponibilidad de actualizaciones.

```bash
openclaw update status
openclaw update status --json
openclaw update status --timeout 10
```

| Indicador             | Valor predeterminado | Descripción                              |
| --------------------- | -------------------- | ---------------------------------------- |
| `--json`              | `false`        | Imprime el JSON de estado legible por máquina. |
| `--timeout <seconds>` | `3`            | Tiempo de espera para las comprobaciones.      |

Para las instalaciones de paquetes extended-stable, el estado realiza la misma selección pública
y verificación exacta del paquete que la actualización en primer plano. Puede indicar
`ahead of extended-stable` cuando la versión instalada es más reciente. Los errores JSON
incluyen `registry.reason` (`selector_missing`, `selector_query_failed`,
`exact_package_mismatch` o `unsupported_git_channel`).

## `update repair`

Vuelve a ejecutar la finalización de la actualización después de que el paquete del núcleo ya haya cambiado,
pero las tareas de reparación posteriores no hayan terminado correctamente. Esta es la ruta de recuperación compatible cuando
`openclaw update` instaló el nuevo paquete del núcleo, pero la sincronización de plugins posterior a la actualización del núcleo,
los metadatos de plugins de npm administrados, la actualización del registro o la reparación de Doctor no
convergieron.

```bash
openclaw update repair
openclaw update repair --channel beta
openclaw update repair --acknowledge-clawhub-risk
openclaw update repair --json
```

| Indicador                                        | Descripción                                                                                                                                                                                                                                                         |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--channel <stable\|extended-stable\|beta\|dev>` | Conserva el canal de actualización del núcleo antes de la reparación. Para extended-stable, los plugins oficiales de npm aptos que siguen la intención básica/predeterminada o `latest` usan como destino la versión exacta instalada del núcleo. La reparación de extended-stable se rechaza en checkouts de Git sin modificar la configuración. |
| `--json`                                         | Imprime el JSON de finalización legible por máquina.                                                                                                                                                                                                                |
| `--timeout <seconds>`                            | Tiempo de espera para los pasos de reparación. Valor predeterminado: `1800`.                                                                                                                                                                           |
| `--yes`                                          | Omite las solicitudes de confirmación.                                                                                                                                                                                                                              |
| `--acknowledge-clawhub-risk`                     | Mismo comportamiento que en `openclaw update`.                                                                                                                                                                                                                     |
| `--no-restart`                                   | Se acepta por coherencia; la reparación nunca reinicia el Gateway.                                                                                                                                                                                                  |

`update repair` ejecuta `openclaw doctor --fix`, vuelve a cargar la configuración reparada y
los registros de instalación, sincroniza los plugins rastreados para el canal de actualización activo, actualiza
las instalaciones administradas de plugins de npm, repara las cargas útiles faltantes de los plugins configurados,
actualiza el registro de plugins y escribe los metadatos convergentes de los registros de instalación.
No instala un nuevo paquete del núcleo ni reinicia el Gateway.

## `update wizard`

Flujo interactivo para elegir un canal de actualización y confirmar si se debe reiniciar el
Gateway después (de forma predeterminada, se reinicia). Al seleccionar `dev` sin un checkout de git,
se ofrece la posibilidad de crear uno.

| Indicador             | Valor predeterminado | Descripción                                  |
| --------------------- | -------------------- | -------------------------------------------- |
| `--timeout <seconds>` | `1800`         | Tiempo de espera para cada paso de actualización. |

## Qué hace

Cambiar de canal explícitamente (`--channel ...`) también mantiene alineado el método
de instalación:

- `dev` -> garantiza que exista un checkout de git (de forma predeterminada, `~/openclaw`, o
  `$OPENCLAW_HOME/openclaw` cuando se establece `OPENCLAW_HOME`; se puede reemplazar con
  `OPENCLAW_GIT_DIR`), lo actualiza e instala la CLI global desde ese
  checkout.
- `stable` -> instala desde npm mediante `latest`.
- `extended-stable` -> resuelve el selector público `extended-stable` de npm,
  verifica el paquete exacto seleccionado e instala esa versión exacta. No
  recurre a otro selector y se rechaza para los checkouts de Git.
- `beta` -> da preferencia a la etiqueta de distribución `beta` de npm y recurre a `latest` cuando la versión beta
  no existe o es anterior a la versión estable actual.

### Transferencia del reinicio

El actualizador automático del núcleo del Gateway (cuando está habilitado mediante la configuración) inicia la ruta
de actualización de la CLI fuera del controlador de solicitudes activo del Gateway. Las actualizaciones del gestor de paquetes
`update.run` del plano de control y las actualizaciones supervisadas de checkouts de git usan
la misma transferencia del servicio administrado en lugar de reemplazar el árbol de paquetes o
recompilar `dist/` dentro del proceso activo del Gateway: el Gateway inicia un
proceso auxiliar independiente y finaliza, y ese proceso auxiliar ejecuta `openclaw update --yes --json`
desde fuera del árbol de procesos del Gateway. Si la transferencia no está disponible,
`update.run` devuelve una respuesta estructurada con el comando de shell seguro que se debe ejecutar
manualmente.

Las selecciones de estabilidad extendida almacenadas reciben indicaciones de inicio de solo lectura y de actualización cada 24 horas
cuando `update.checkOnStart` está habilitado. Estas comprobaciones nunca aplican una actualización,
inician un traspaso, reinician el Gateway, usan el retraso o la variación aleatoria de stable, ni usan la
cadencia de sondeo de beta. Las actualizaciones explícitas en primer plano, las actualizaciones sin argumentos en primer plano con
`update.channel: "extended-stable"` almacenado, el estado bajo demanda y su traspaso administrado del
Gateway siguen siendo compatibles.

Cuando hay instalado un servicio Gateway administrado local y el reinicio está habilitado,
las actualizaciones mediante el gestor de paquetes y mediante un checkout de Git detienen el servicio en ejecución antes de
reemplazar el árbol de paquetes o modificar la salida del checkout o de la compilación. A continuación, el actualizador
renueva los metadatos del servicio, reinicia el servicio y verifica el
Gateway reiniciado antes de informar `Gateway: restarted and verified.`.
Además, las actualizaciones mediante el gestor de paquetes verifican que el Gateway reiniciado informe la
versión esperada del paquete; las actualizaciones mediante un checkout de Git verifican el estado del Gateway y
la disponibilidad del servicio después de la recompilación.

Normalmente, las actualizaciones mediante el gestor de paquetes siguen usando el binario de Node registrado en el
servicio administrado. Si ese Node no puede ejecutar la versión de destino, pero el Node de la
CLI actual sí puede y se ha demostrado que el servicio pertenece al paquete que se está actualizando,
una actualización con reinicio habilitado usa el Node actual para la finalización y reescribe
los metadatos del servicio para ese entorno de ejecución. `--no-restart` no puede reparar los metadatos del
servicio, por lo que la misma incompatibilidad del entorno de ejecución detiene el proceso antes de modificar el paquete.

En macOS, la comprobación posterior a la actualización también verifica que el LaunchAgent esté
cargado y en ejecución para el perfil activo y que el puerto de bucle invertido configurado
funcione correctamente. Si el plist está instalado, pero launchd no lo está supervisando, OpenClaw
vuelve a inicializar automáticamente el LaunchAgent y repite las comprobaciones de estado, versión y
disponibilidad del canal (una inicialización nueva carga directamente el trabajo `RunAtLoad`,
por lo que la recuperación no ejecuta inmediatamente `kickstart -k` en el Gateway recién iniciado). Si
el Gateway sigue sin alcanzar un estado correcto, el comando termina con un código distinto de cero e
imprime la ruta del registro de reinicio, además de instrucciones para reiniciar, reinstalar y revertir
el paquete.

Si no se puede ejecutar el reinicio, el comando imprime `Gateway: restart skipped (...)` o
`Gateway: restart failed: ...` con una indicación manual de `openclaw gateway restart`.
Con `--no-restart`, el reemplazo del paquete o la recompilación de Git se siguen ejecutando, pero el
servicio administrado no se detiene ni se reinicia, por lo que el Gateway en ejecución conserva el código
anterior hasta que se reinicie manualmente.

### Formato de respuesta del plano de control

Cuando `update.run` se ejecuta a través del plano de control del Gateway en una instalación
mediante un gestor de paquetes o un checkout de Git supervisado, el controlador informa el inicio del traspaso
por separado de la actualización de la CLI que continúa después de que el Gateway termine:

- `ok: true`, `result.status: "skipped"`,
  `result.reason: "managed-service-handoff-started"` y
  `handoff.status: "started"`: el Gateway creó el traspaso del servicio administrado
  y programó su propio reinicio para que el proceso auxiliar desacoplado pueda ejecutar
  `openclaw update --yes --json` fuera del proceso activo del servicio.
- `ok: false`, `result.reason: "managed-service-handoff-unavailable"` y
  `handoff.status: "unavailable"`: OpenClaw no pudo encontrar un límite de servicio supervisado
  ni una identidad de servicio persistente para realizar un traspaso seguro (por
  ejemplo, el traspaso de systemd requiere la identidad de unidad `OPENCLAW_SYSTEMD_UNIT`,
  no solo indicadores de procesos systemd del entorno). La respuesta incluye
  `handoff.command`, el comando de shell que se debe ejecutar desde fuera del Gateway.
- `ok: false`, `result.reason: "managed-service-handoff-failed"`: el Gateway
  intentó crear el traspaso, pero no pudo iniciar el proceso auxiliar desacoplado.

La carga útil `sentinel` se escribe antes de que el Gateway termine, y el traspaso de la CLI
actualiza ese mismo indicador de reinicio después de que finalicen las comprobaciones de estado
del reinicio del servicio administrado. Durante el traspaso, el indicador puede contener
`stats.reason: "restart-health-pending"` sin continuación de éxito; el
Gateway reiniciado lo sondea y activa la continuación únicamente después de que la CLI haya
verificado el estado del servicio y reescrito el indicador con el resultado final `ok`.
`openclaw status` y `openclaw status --all` muestran una fila `Update restart`
mientras ese indicador está pendiente o ha fallado, y `update.status` actualiza y
devuelve el indicador más reciente.

## Flujo de checkout de Git

### Selección del canal

- `stable`: obtiene la etiqueta no beta más reciente y, a continuación, compila y ejecuta doctor.
- `beta`: prioriza la etiqueta `-beta` más reciente y recurre a la etiqueta stable más reciente
  cuando no existe una beta o esta es más antigua.
- `dev`: obtiene `main` y, a continuación, descarga los cambios y ejecuta rebase.
- `extended-stable`: no es compatible con los checkouts de Git; no se produce ninguna modificación
  del checkout.

### Pasos de actualización

<Steps>
  <Step title="Verificar que el árbol de trabajo esté limpio">
    Requiere que no haya cambios sin confirmar.
  </Step>
  <Step title="Cambiar de canal">
    Cambia al canal seleccionado (etiqueta o rama).
  </Step>
  <Step title="Obtener cambios del repositorio remoto">
    Solo para dev.
  </Step>
  <Step title="Compilación preliminar (solo para dev)">
    Ejecuta la compilación de TypeScript en un árbol de trabajo temporal. Si la punta falla, retrocede hasta 10 commits para encontrar el commit compilable más reciente. Configure `OPENCLAW_UPDATE_PREFLIGHT_LINT=1` para ejecutar también el lint durante esta comprobación preliminar; el lint se ejecuta en un modo en serie restringido porque los hosts de actualización de los usuarios suelen ser más pequeños que los ejecutores de CI.
  </Step>
  <Step title="Ejecutar rebase">
    Ejecuta rebase sobre el commit seleccionado (solo para dev).
  </Step>
  <Step title="Instalar dependencias">
    Usa el gestor de paquetes del repositorio. Para los checkouts de pnpm, el actualizador inicializa `pnpm` bajo demanda (primero mediante `corepack` y después mediante una alternativa temporal `npm install pnpm@11`) en lugar de ejecutar `npm run build` dentro de un espacio de trabajo de pnpm. Si la inicialización de pnpm sigue fallando, el actualizador se detiene anticipadamente con un error específico del gestor de paquetes en lugar de intentar ejecutar `npm run build` en el checkout.
  </Step>
  <Step title="Compilar la interfaz de control">
    Compila el Gateway y la interfaz de control.
  </Step>
  <Step title="Ejecutar doctor">
    `openclaw doctor` se ejecuta como comprobación final de actualización segura.
  </Step>
  <Step title="Sincronizar plugins">
    Sincroniza los plugins con el canal activo. Dev usa plugins incluidos; stable y beta usan npm. Actualiza las instalaciones de plugins con seguimiento.
  </Step>
</Steps>

### Detalles de la sincronización de plugins

En el canal beta, las instalaciones de plugins de npm y ClawHub con seguimiento que siguen la
línea predeterminada/latest intentan primero una versión `@beta` del plugin. Si el plugin no tiene
una versión beta, OpenClaw recurre a la especificación predeterminada/latest registrada e
informa de una advertencia. Para los plugins de npm, OpenClaw también recurre a la alternativa cuando el
paquete beta existe, pero no supera la validación de instalación. Estas advertencias de uso de alternativas no
hacen que falle la actualización del núcleo. Las versiones exactas y las etiquetas explícitas nunca se reescriben.

<Warning>
Si una actualización de un plugin de npm fijada a una versión exacta se resuelve en un artefacto cuya integridad difiere del registro de instalación almacenado, `openclaw update` cancela la actualización de ese artefacto del plugin en lugar de instalarlo. Reinstale o actualice el plugin explícitamente solo después de verificar que confía en el nuevo artefacto.
</Warning>

<Note>
Los fallos de sincronización de plugins posteriores a la actualización que se limitan a un plugin administrado y que la ruta de sincronización puede evitar (por ejemplo, un registro de npm inaccesible para un plugin no esencial) se informan como advertencias después de que la actualización del núcleo se complete correctamente. El resultado JSON mantiene `status: "ok"` para la actualización de nivel superior e informa `postUpdate.plugins.status: "warning"` con orientación de `openclaw update repair` y `openclaw plugins inspect <id> --runtime --json`. Las excepciones inesperadas del actualizador o de la sincronización siguen provocando el fallo del resultado de la actualización. Corrija el error de instalación o actualización del plugin y, a continuación, vuelva a ejecutar `openclaw update repair`. Cuando una actualización fallida deja inutilizable un plugin administrado, OpenClaw deshabilita su entrada de entorno de ejecución y restablece los slots activos sin cambiar la política `plugins.allow` o `plugins.deny` definida por el operador.

Después del paso de sincronización de cada plugin, `openclaw update` ejecuta una pasada obligatoria de **convergencia posterior al núcleo** antes de que el Gateway se reinicie: repara las cargas útiles que faltan de los plugins configurados, valida en el disco cada registro de instalación con seguimiento _activo_ y verifica estáticamente que su `package.json` se pueda analizar (y que exista cualquier `main` declarado explícitamente). Los fallos de esta pasada y una instantánea de configuración no válida devuelven `postUpdate.plugins.status: "error"` y cambian el valor de nivel superior `status` de la actualización a `"error"`, por lo que `openclaw update` termina con un código distinto de cero y el Gateway _no_ se reinicia con un conjunto de plugins sin verificar. El error incluye líneas estructuradas `postUpdate.plugins.warnings[].guidance` que apuntan a `openclaw update repair` y `openclaw plugins inspect <id> --runtime --json`. Las entradas de plugins deshabilitadas y los registros que no sean destinos oficiales de sincronización vinculados a una fuente de confianza se omiten aquí (reflejando la política `skipDisabledPlugins` utilizada por la comprobación de cargas útiles ausentes), por lo que un registro obsoleto de un plugin deshabilitado no puede bloquear una actualización que, por lo demás, sea válida.

Cuando se inicia el Gateway actualizado, la carga de plugins es solo de verificación: el inicio no ejecuta gestores de paquetes ni modifica árboles de dependencias. Los reinicios `update.run` mediante el gestor de paquetes se transfieren a la ruta de servicio administrado de la CLI, por lo que el intercambio de paquetes se realiza fuera del proceso anterior del Gateway y las comprobaciones de estado del servicio determinan si se puede informar que la actualización se ha completado.
</Note>

Después de que una actualización del núcleo de estabilidad extendida se complete correctamente, la integridad y
la convergencia de los plugins posteriores al núcleo se aplican a los plugins oficiales de npm aptos en la versión
exacta instalada del núcleo. Para la intención predeterminada/`latest`, OpenClaw no consulta
`@extended-stable` del plugin ni recurre a `latest` de npm; deriva la versión del paquete
a partir del núcleo instalado. Las versiones fijadas explícitamente, las etiquetas explícitas distintas de `latest`,
los paquetes de terceros y las fuentes que no sean npm conservan su intención existente.

Para las instalaciones mediante gestores de paquetes, `openclaw update` resuelve la versión de destino del paquete
antes de invocar el gestor de paquetes. Las instalaciones globales de npm usan una instalación por etapas:
OpenClaw instala el nuevo paquete en un prefijo temporal de npm,
permite que el paquete candidato valide la versión de Node del host durante `preinstall`,
y verifica allí el inventario empaquetado `dist`. Una protección de finalización empaquetada
permanece fuera de ese inventario hasta que `preinstall` se completa correctamente, por lo que los gestores de paquetes
que omiten los scripts del ciclo de vida también se detienen antes de la activación. En npm 12 y versiones posteriores,
el actualizador aprueba únicamente el ciclo de vida de la versión candidata de OpenClaw; los scripts de
dependencias transitivas permanecen bloqueados. A continuación, OpenClaw intercambia el árbol de paquetes limpio
en el prefijo global real. Si la verificación falla, doctor posterior a la actualización, la sincronización de plugins
y las tareas de reinicio no se ejecutan desde el árbol sospechoso. Incluso cuando la
versión instalada ya coincide con la de destino, el comando renueva la
instalación global del paquete y, a continuación, ejecuta la sincronización de plugins, una renovación de finalización
de los comandos del núcleo y las tareas de reinicio. Esto mantiene los componentes auxiliares empaquetados y los registros
de plugins propiedad del canal alineados con la compilación instalada de OpenClaw, mientras que reserva las
recompilaciones completas de finalización de comandos de plugins para ejecuciones explícitas de
`openclaw completion --write-state`.

## Contenido relacionado

- `openclaw doctor` (ofrece ejecutar primero la actualización en los checkouts de Git)
- [Canales de desarrollo](/es/install/development-channels)
- [Actualización](/es/install/updating)
- [Referencia de la CLI](/es/cli)
