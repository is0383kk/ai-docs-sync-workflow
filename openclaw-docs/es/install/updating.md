---
read_when:
    - Actualización de OpenClaw
    - Algo deja de funcionar después de una actualización
summary: Actualizar OpenClaw de forma segura (instalación global o desde el código fuente), además de una estrategia de reversión
title: Actualizando
x-i18n:
    generated_at: "2026-07-26T04:41:23Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 83444d56e0aa34f47830610538b0c3012903abb812bfe0fffb8163a5db9ac2db
    source_path: install/updating.md
    workflow: 16
---

Mantenga OpenClaw actualizado.

Para reemplazar imágenes de Docker, Podman y Kubernetes, consulte
[Actualización de imágenes de contenedor](/es/install/docker#upgrading-container-images). El
Gateway ejecuta tareas de actualización seguras para el inicio antes de indicar que está listo y finaliza si el
estado montado requiere reparación manual.

## Recomendado: `openclaw update`

Detecta el tipo de instalación (npm, pnpm, Bun o git), obtiene la versión más reciente, ejecuta `openclaw doctor` y reinicia el Gateway.

```bash
openclaw update
```

Cambie de canal o seleccione una versión específica:

```bash
openclaw update --channel beta
openclaw update --channel extended-stable
openclaw update --channel dev
openclaw update --dry-run   # vista previa sin aplicar
```

`openclaw update` no tiene la opción `--verbose` (el instalador sí). Para el diagnóstico, use
`--dry-run` para obtener una vista previa de las acciones previstas, `--json` para obtener resultados estructurados o
`openclaw update status --json` para inspeccionar el canal y el estado de disponibilidad.

`--channel beta` prefiere la etiqueta de distribución beta de npm, pero recurre a stable/latest
cuando falta la etiqueta beta o su versión es anterior a la versión estable
más reciente. En su lugar, use `--tag beta` para realizar una actualización puntual del paquete fijada a la
etiqueta de distribución beta sin procesar de npm.

`--channel extended-stable` funciona solo con paquetes y la instalación continúa
ejecutándose únicamente en primer plano. OpenClaw lee el selector público `extended-stable` de npm,
verifica el paquete exacto seleccionado e instala esa versión exacta. Si faltan datos
del registro o son incoherentes, el proceso falla de forma segura; nunca recurre a `latest`.
Si la versión seleccionada es anterior a la versión instalada, se sigue aplicando
la confirmación de degradación habitual. La CLI conserva el canal después de una
actualización correcta del núcleo; una ejecución directa de `npm install -g openclaw@extended-stable`
no actualiza `update.channel`.
Después de sustituir el núcleo, los plugins oficiales de npm aptos con intención
vacía/predeterminada o `latest` convergen en esa versión exacta del núcleo. Las versiones fijadas exactamente y las etiquetas explícitas
distintas de `latest`, los plugins de terceros y las fuentes que no son de npm permanecen sin cambios.
Las instalaciones de catálogo creadas por las versiones actuales de OpenClaw conservan esa intención
predeterminada. Los registros anteriores que solo contienen una versión exacta permanecen fijados porque
OpenClaw no puede distinguir de forma segura una fijación automática antigua de una fijación del usuario; ejecute
`openclaw plugins update @openclaw/name` una vez en el canal extended-stable
para que ese plugin vuelva a seguir exactamente la versión del núcleo.

`--channel dev` proporciona un checkout persistente y móvil de `main` en GitHub. Para una actualización
puntual de un paquete, `--tag main` se asigna a la especificación de paquete
`github:openclaw/openclaw#main` y la instala directamente mediante el gestor de paquetes de destino (npm/pnpm/bun).

Para los plugins administrados, la ausencia de una versión beta es una advertencia, no un fallo: la
actualización del núcleo puede completarse correctamente mientras un plugin recurre a su versión
predeterminada/más reciente registrada.

Consulte [Canales de publicación](/es/install/development-channels) para conocer la semántica de los canales.

## Cambiar entre instalaciones de npm y git

Use canales para cambiar el tipo de instalación. El actualizador conserva el estado, la configuración,
las credenciales y el espacio de trabajo en `~/.openclaw`; solo cambia qué instalación
del código de OpenClaw utilizan la CLI y el Gateway.

```bash
# instalación del paquete npm -> checkout de git editable
openclaw update --channel dev

# checkout de git -> instalación del paquete npm
openclaw update --channel stable
```

Obtenga primero una vista previa del cambio de modo de instalación:

```bash
openclaw update --channel dev --dry-run
openclaw update --channel stable --dry-run
```

`dev` garantiza que haya un checkout de git, lo compila e instala la CLI global desde ese
checkout. Los canales `stable`, `extended-stable` y `beta` usan instalaciones de
paquetes. Extended-stable se rechaza en un checkout de git sin modificarlo ni
convertirlo. Si el Gateway ya está instalado, `openclaw update` actualiza
los metadatos del servicio y lo reinicia, a menos que se proporcione `--no-restart`.

En instalaciones de paquetes con un servicio Gateway administrado, `openclaw update` utiliza como destino
la raíz del paquete que usa dicho servicio. Si el comando del shell `openclaw` procede
de una instalación diferente, el actualizador muestra ambas raíces y la ruta de Node
del servicio administrado, y comprueba esa versión de Node con respecto al requisito
`engines.node` de la versión de destino antes de sustituir el paquete.

## Servidores con checkout del código fuente (script de referencia)

Los equipos que ejecuten un Gateway directamente desde un checkout de git en un servidor pueden actualizarlo
con `scripts/update-gateway.sh` desde dentro de dicho checkout. Es la referencia
para una actualización eficiente de un servidor desde el código fuente: restaura las salidas de compilación versionadas que
`pnpm build` reescribe, falla de forma segura ante cualquier otro cambio local, avanza rápidamente
`main` (o reorganiza una rama local del servidor sobre `origin/main`), instala
las dependencias, realiza una compilación limpia y reinicia el Gateway.

```bash
ssh you@server 'cd /path/to/openclaw && scripts/update-gateway.sh'
```

Sobrescriba el reinicio para unidades de servicio personalizadas u omítalo por completo:

```bash
OPENCLAW_UPDATE_RESTART_CMD='systemctl --user restart openclaw-gateway.service' scripts/update-gateway.sh
OPENCLAW_UPDATE_RESTART_CMD='' scripts/update-gateway.sh
```

Para una instalación sencilla del código fuente de un solo usuario, se recomienda `openclaw update --channel dev`
en su lugar: administra el checkout, la compilación y el reinicio del Gateway.

## Alternativa: volver a ejecutar el instalador

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

Añada `--no-onboard` para omitir la incorporación. Para forzar un tipo de instalación específico, proporcione
`--install-method git --no-onboard` o `--install-method npm --no-onboard`.

Si `openclaw update` falla después de la fase de instalación del paquete npm, vuelva a ejecutar el
instalador. No llama al actualizador; ejecuta directamente la instalación global del paquete
y puede recuperar una instalación de npm actualizada parcialmente.

```bash
curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method npm
```

Fije la recuperación a una versión o etiqueta de distribución específica con `--version`:

```bash
curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method npm --version <version-or-dist-tag>
```

## Alternativa: npm, pnpm o bun de forma manual

```bash
npm i -g openclaw@latest
```

Se recomienda `openclaw update` para instalaciones supervisadas: puede coordinar la sustitución
del paquete con el servicio Gateway en ejecución. Si actualiza manualmente una instalación
supervisada, detenga primero el Gateway administrado. Los gestores de paquetes sustituyen los archivos
en el mismo lugar y, de lo contrario, un Gateway en ejecución podría intentar cargar archivos del núcleo o de plugins
en plena sustitución. Reinicie el Gateway cuando termine el gestor de paquetes para que utilice
la nueva instalación.

En una instalación global del sistema Linux propiedad de root, si `openclaw update` falla con
`EACCES`, realice la recuperación con el npm del sistema y mantenga el Gateway detenido durante la
sustitución manual. Use las mismas opciones de perfil o variables de entorno que utiliza normalmente para
ese Gateway. Sustituya `/usr/bin/npm` por el npm del sistema que controla el
prefijo global propiedad de root en el host:

```bash
openclaw gateway stop
sudo /usr/bin/npm i -g openclaw@latest
openclaw gateway install --force
openclaw gateway restart
```

A continuación, verifique:

```bash
openclaw --version
curl -fsS http://127.0.0.1:18789/readyz
openclaw plugins list --json
openclaw gateway status --deep --json
openclaw doctor --lint --json
```

Cuando `openclaw update` administra una instalación global de npm, primero instala el destino
en un prefijo temporal de npm. El paquete candidato valida la versión de Node
del host durante `preinstall`; solo entonces OpenClaw verifica el inventario empaquetado
`dist` y sustituye el árbol limpio del paquete en el prefijo global real. Se omite
una protección de finalización empaquetada del inventario esperado y solo se elimina
después de que `preinstall` se complete correctamente, por lo que los scripts de ciclo de vida omitidos también fallan antes de la
sustitución. En npm 12 y versiones posteriores, el actualizador solo autoriza el ciclo de vida
del OpenClaw candidato; los scripts de dependencias transitivas permanecen bloqueados. Esto evita que npm
superponga un paquete nuevo sobre archivos obsoletos del anterior. Si el comando de
instalación falla, OpenClaw vuelve a intentarlo una vez con `--omit=optional`, lo que resulta útil en hosts
donde las dependencias opcionales nativas no pueden compilarse.

Los comandos de actualización de npm y de actualización de plugins administrados por OpenClaw también borran la
cuarentena de la cadena de suministro `min-release-age` de npm (o la clave de configuración anterior `before`)
para el proceso secundario de npm. Esa política existe como protección general, pero una
actualización explícita de OpenClaw significa «instalar ahora la versión seleccionada».

```bash
pnpm add -g openclaw@latest
```

Si pnpm 11 instaló OpenClaw 2026.7.1, ejecute ese comando manual una vez. Esa
versión es anterior al diseño aislado de paquetes globales de pnpm 11, por lo que su actualizador puede
confundir otra instalación de npm con la CLI en ejecución. Las versiones posteriores conservan
la propiedad de pnpm y siguen la raíz del paquete de sustitución durante las actualizaciones. También
utilizan el directorio bin global indicado por el gestor propietario y se detienen antes
de realizar modificaciones cuando el comando pnpm disponible indica otra raíz global o versión principal,
o cuando el paquete invocador está huérfano o no es la única instalación activa de OpenClaw
allí.

Si OpenClaw comparte un grupo de instalación global de pnpm 11 con otro paquete, el
actualizador automático se detiene antes de cambiar el grupo. Actualice manualmente el grupo
original separado por comas para mantener intactos sus paquetes asociados y su política de
compilación.

```bash
bun add -g openclaw@latest
```

### Temas avanzados sobre la instalación con npm

<AccordionGroup>
  <Accordion title="Árbol de paquetes de solo lectura">
    OpenClaw trata las instalaciones globales empaquetadas como de solo lectura durante la ejecución, incluso cuando el usuario actual puede escribir en el directorio global de paquetes. Las instalaciones de paquetes de plugins se encuentran en raíces de npm/git propiedad de OpenClaw bajo el directorio de configuración del usuario, y el inicio del Gateway no modifica el árbol de paquetes de OpenClaw.

    Algunas configuraciones de npm en Linux instalan los paquetes globales en directorios propiedad de root, como `/usr/lib/node_modules/openclaw`. OpenClaw admite este diseño porque los comandos de instalación y actualización de plugins escriben fuera de ese directorio global de paquetes.

  </Accordion>
  <Accordion title="Unidades systemd reforzadas">
    Conceda a OpenClaw acceso de escritura a sus raíces de configuración y estado para que las instalaciones explícitas de plugins, las actualizaciones de plugins y la limpieza de doctor puedan conservar sus cambios:

    ```ini
    ReadWritePaths=/var/lib/openclaw /home/openclaw/.openclaw /tmp
    ```

  </Accordion>
  <Accordion title="Comprobación previa del espacio en disco">
    Antes de las actualizaciones de paquetes y las instalaciones explícitas de plugins, OpenClaw intenta comprobar el espacio en disco disponible del volumen de destino de forma orientativa. Si hay poco espacio, se genera una advertencia con la ruta comprobada, pero no se bloquea la actualización porque las cuotas del sistema de archivos, las instantáneas y los volúmenes de red pueden cambiar después de la comprobación. La instalación real del gestor de paquetes y la verificación posterior a la instalación siguen siendo la fuente definitiva.
  </Accordion>
</AccordionGroup>

## Actualizador automático

Desactivado de forma predeterminada. Actívelo en `~/.openclaw/openclaw.json`:

```json5
{
  update: {
    channel: "stable",
    auto: {
      enabled: true,
    },
  },
}
```

| Canal             | Comportamiento                                                                                                                   |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `stable`          | Se aplica después de un retraso integrado con una variación determinista para un despliegue escalonado.                          |
| `extended-stable` | Comprueba si existe un aviso de actualización de solo lectura al iniciar y cada 24 horas cuando `checkOnStart` está activado. Nunca se aplica automáticamente. |
| `beta`            | Realiza comprobaciones en un intervalo integrado y se aplica inmediatamente.                                                     |
| `dev`             | No se aplica automáticamente. Use `openclaw update` manualmente.                                                                |

El Gateway también registra una sugerencia de actualización al iniciarse (se desactiva con
`update.checkOnStart: false`). Las selecciones de estabilidad extendida almacenadas usan esta
ruta de sugerencias de solo lectura y el intervalo existente de 24 horas, pero nunca invocan
la instalación automática, la transferencia, el reinicio, el retraso o la fluctuación de la versión estable ni el sondeo de la versión beta.
Para revertir a una versión anterior o recuperarse de un incidente, establezca `OPENCLAW_NO_AUTO_UPDATE=1` en el entorno del Gateway para bloquear las aplicaciones automáticas incluso cuando `update.auto.enabled` esté configurado. Las sugerencias de actualización al inicio pueden seguir ejecutándose a menos que también se desactive `update.checkOnStart`.

Las actualizaciones del gestor de paquetes solicitadas mediante el plano de control activo del Gateway
(`update.run`) no sustituyen el árbol de paquetes dentro del proceso del Gateway
en ejecución. En instalaciones de servicios administrados, el Gateway inicia una transferencia desacoplada,
sale y permite que la ruta normal de la CLI `openclaw update --yes --json` detenga el
servicio, sustituya el paquete, actualice los metadatos del servicio, reinicie, verifique la
versión y la accesibilidad del Gateway, y recupere, cuando sea posible, un LaunchAgent de macOS
instalado pero no cargado. Si el Gateway no puede realizar esa transferencia de forma segura,
`update.run` muestra un comando de shell seguro en lugar de ejecutar el gestor de
paquetes dentro del proceso.

La tarjeta de actualización de la barra lateral de la interfaz de control muestra **Actualizar Gateway** cuando inicia
directamente este flujo `update.run`. Esto abarca la interfaz de control alojada en el navegador, los
Gateways remotos y los Gateways locales administrados manualmente.

En la aplicación firmada para macOS, un Gateway local propiedad de la aplicación cambia esa tarjeta a
**Actualizar aplicación para Mac + Gateway**. Sparkle actualiza primero la aplicación; después de volver a iniciarla, la
aplicación ejecuta `openclaw update --tag <app-version> --json`, reinicia su Gateway
y verifica el estado en una ventana de progreso similar a la de configuración. La ventana aparece solo
cuando ese Gateway administrado necesita una actualización, reparación o instalación; las actualizaciones exclusivas de la aplicación se reinician
directamente en la aplicación. Los detalles de los fallos permanecen visibles con las acciones Reintentar, [Guía de actualización](/es/install/updating) y
[Discord](https://discord.gg/clawd). La aplicación nunca utiliza esta ruta coordinada
para un Gateway remoto o administrado externamente, nunca revierte un Gateway más reciente
a una versión anterior y nunca anula una fijación del canal `extended-stable`.

Cuando la actualización se completa correctamente, la aplicación pone en cola un evento de bienvenida único para la
sesión directa de nivel superior más reciente con una interacción real de usuario o canal. Las ejecuciones de Cron,
los Heartbeat y las actualizaciones de sesión exclusivamente en segundo plano no cambian esa selección. En
modo remoto, la aplicación solo actualiza el entorno de ejecución de su Node local de Mac y envía el evento
únicamente cuando el Gateway remoto conectado es al menos tan reciente como la aplicación.

## Después de actualizar

<Steps>

### Ejecutar doctor

```bash
openclaw doctor
```

Migra la configuración, audita las políticas de mensajes directos y comprueba el estado del Gateway. Detalles: [Doctor](/es/gateway/doctor)

### Reiniciar el Gateway

```bash
openclaw gateway restart
```

### Verificar

```bash
openclaw health
```

</Steps>

## Reversión

La reversión tiene dos niveles:

1. Reinstalar código anterior de OpenClaw conservando el estado actual.
2. Restaurar el estado anterior a la actualización solo cuando el código anterior no pueda usar una
   configuración o base de datos migrada.

Comience con una reversión únicamente del código. Restaurar el estado descarta los cambios realizados después
de la copia de seguridad.

### Antes de actualizar: crear una copia de seguridad verificada

`openclaw update` conserva una copia automática de la configuración previa a la actualización, pero no
crea un punto de recuperación completo del estado. Antes de una actualización importante, cree uno
explícitamente:

```bash
mkdir -p ~/Backups/openclaw
openclaw backup create --output ~/Backups/openclaw --verify
```

El manifiesto del archivo registra la versión de OpenClaw y las rutas de origen incluidas
en la copia de seguridad. El archivo puede contener credenciales, perfiles de autenticación y el estado de los
canales, por lo que debe almacenarse con permisos exclusivos del propietario y la misma protección que el
directorio de estado activo. Consulte [Copia de seguridad](/es/cli/backup) para conocer los archivos incluidos y los omitidos
intencionadamente.

Para obtener un punto de recuperación idéntico byte por byte que incluya los artefactos volátiles omitidos por
el archivo portátil, detenga el Gateway y use una instantánea del sistema de archivos, volumen o máquina virtual
proporcionada por su plataforma.

### Revertir la instalación de un paquete

Enumere las versiones publicadas y, a continuación, previsualice e instale la versión válida conocida:

```bash
npm view openclaw versions --json
openclaw update --tag <known-good-version> --dry-run
openclaw update --tag <known-good-version>
```

Se prefiere `openclaw update --tag` a una instalación directa mediante el gestor de paquetes. Esta opción
detecta la reversión a una versión anterior, solicita confirmación, ejecuta la convergencia administrada de plugins
y las comprobaciones de compatibilidad con el destino instalado, actualiza los metadatos del servicio,
reinicia el Gateway y verifica la versión en ejecución. Si el canal almacenado es
`extended-stable`, use
`--channel stable --tag <known-good-version>`, ya que las etiquetas puntuales exactas no pueden
combinarse con el selector `extended-stable`.

Las actualizaciones de paquetes preparan y verifican el candidato antes de activarlo. Si falla el
intercambio del sistema de archivos o la sustitución del enlace de comandos, OpenClaw restaura automáticamente
el paquete anterior. Tras un intercambio correcto, un fallo posterior en el estado del Gateway
indica la versión anterior y proporciona instrucciones de reversión manual en lugar de
volver a sustituir automáticamente el paquete.

Si la ruta de actualización de la CLI no está disponible, use el mismo gestor de paquetes y el mismo
ámbito de instalación que administra el Gateway actual:

```bash
openclaw gateway stop
npm i -g openclaw@<known-good-version>
openclaw gateway install --force
openclaw gateway restart
```

Sustituya `npm` por `pnpm` o `bun` cuando ese gestor sea el propietario de la instalación. Durante
la recuperación de un incidente, impida que un actualizador automático habilitado aplique inmediatamente una
versión más reciente estableciendo `OPENCLAW_NO_AUTO_UPDATE=1` en el entorno del Gateway.

### Revertir un checkout del código fuente

Use un checkout limpio y seleccione una etiqueta o un commit válido conocido:

```bash
git fetch --all --tags
git checkout --detach <known-good-tag-or-commit>
pnpm install && pnpm build
openclaw gateway restart
```

Para volver a la versión más reciente: `git checkout main && git pull`.

El actualizador devuelve automáticamente un checkout de Git a su rama y
SHA anteriores cuando la instalación de dependencias, la compilación, la compilación de la interfaz de usuario o doctor fallan después de iniciarse una
actualización de Git. El checkout manual sigue siendo necesario cuando se elige intencionadamente
un commit anterior.

### Reversión a una versión anterior tras la migración de sesiones a SQLite

Antes de iniciar una versión anterior de OpenClaw basada en archivos, use la CLI actual para
restaurar los artefactos archivados de transcripciones heredadas:

```bash
openclaw gateway stop
openclaw doctor --session-sqlite restore --session-sqlite-all-agents
```

Esto no elimina los datos de SQLite. Las sesiones creadas después de la migración a SQLite
solo existen en SQLite y no aparecerán en el entorno de ejecución anterior. Consulte
[Reversión a una versión anterior tras la migración de sesiones a SQLite](/es/cli/doctor#downgrading-after-session-sqlite-migration).

### Restaurar el estado solo cuando sea necesario

Si el código anterior no puede leer una configuración o un esquema de base de datos más reciente, detenga el
Gateway y restaure la instantánea verificada del sistema de archivos, volumen o máquina virtual anterior a la actualización.
Conserve el estado actual por separado antes de restaurar, ya que esto elimina
los cambios realizados después de la instantánea.

Los archivos amplios `openclaw backup create` permiten su creación y verificación, pero
no la activación in situ de todo el archivo. Extraiga un archivo amplio en un directorio
de preparación y use su asignación de origen a archivo `manifest.json` para una restauración
sin conexión. Del mismo modo, `openclaw backup sqlite restore` escribe una base de datos verificada
en un destino nuevo; la activación de ese destino sigue siendo un paso explícito del operador
sin conexión.

### Verificar la reversión

```bash
openclaw --version
openclaw health
openclaw plugins list --json
openclaw gateway status --deep --json
openclaw doctor --lint --json
```

## En caso de bloqueo

- Ejecute de nuevo `openclaw doctor` y lea detenidamente el resultado.
- Para `openclaw update --channel dev` en checkouts del código fuente, el actualizador prepara automáticamente `pnpm` cuando es necesario. Si aparece un error de preparación de pnpm/corepack, instale `pnpm` manualmente (o vuelva a habilitar `corepack`) y ejecute de nuevo la actualización.
- Consulte: [Solución de problemas](/es/gateway/troubleshooting)
- Pregunte en Discord: [https://discord.gg/clawd](https://discord.gg/clawd)

## Relacionado

- [Descripción general de la instalación](/es/install): todos los métodos de instalación.
- [Doctor](/es/gateway/doctor): comprobaciones de estado después de las actualizaciones.
- [Migración](/es/install/migrating): guías de migración entre versiones principales.
