---
read_when: You want a dedicated explanation of sandboxing or need to tune agents.defaults.sandbox.
sidebarTitle: Sandboxing
status: active
summary: 'Cómo funciona el aislamiento de OpenClaw: modos, ámbitos, acceso al espacio de trabajo e imágenes'
title: Aislamiento en entorno seguro
x-i18n:
    generated_at: "2026-07-26T05:08:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a3668dc512a8ff30732290ee68e9dd29a3a2e9c106e6e39077a97bfbd90098f7
    source_path: gateway/sandboxing.md
    workflow: 16
---

OpenClaw puede ejecutar herramientas dentro de un backend de entorno aislado para reducir el radio de impacto. El aislamiento está desactivado de forma predeterminada y se controla mediante `agents.defaults.sandbox` (global) o `agents.entries.*.sandbox` (por agente). El proceso Gateway siempre permanece en el host; cuando se habilita el aislamiento, solo la ejecución de herramientas se traslada al entorno aislado.

<Note>
Este no es un límite de seguridad perfecto, pero restringe considerablemente el acceso al sistema de archivos y a los procesos cuando el modelo hace algo absurdo.
</Note>

## Qué se ejecuta en el entorno aislado

- Ejecución de herramientas: `exec`, `read`, `write`, `edit`, `apply_patch`, `process`, etc.
- El navegador opcional del entorno aislado (`agents.defaults.sandbox.browser`).

No se ejecutan en el entorno aislado:

- El propio proceso Gateway.
- Cualquier herramienta a la que se permita explícitamente ejecutarse fuera del entorno aislado mediante `tools.elevated`. La ejecución elevada omite el aislamiento y se ejecuta en la ruta de escape configurada (`gateway` de forma predeterminada, o `node` cuando el destino de ejecución es `node`). Si el aislamiento está desactivado, `tools.elevated` no cambia nada, ya que la ejecución se realiza en el host. Consulte [Modo elevado](/es/tools/elevated).

## Modos, ámbito y backend

Tres ajustes independientes controlan el comportamiento del entorno aislado:

| Ajuste | Clave                               | Valores                       | Valor predeterminado  |
| ------- | --------------------------------- | ---------------------------- | -------- |
| Modo    | `agents.defaults.sandbox.mode`    | `off`, `non-main`, `all`     | `off`    |
| Ámbito   | `agents.defaults.sandbox.scope`   | `agent`, `session`, `shared` | `agent`  |
| Backend | `agents.defaults.sandbox.backend` | `docker`, `ssh`, `openshell` | `docker` |

El **modo** controla cuándo se aplica el aislamiento:

- `off`: sin aislamiento.
- `non-main`: ejecuta en un entorno aislado todas las sesiones excepto la sesión principal del agente. La clave de la sesión principal siempre es `agent:<agentId>:main` (o `global` cuando `session.scope` es `"global"`); no se puede configurar. Las sesiones de grupos o canales usan sus propias claves, por lo que siempre se consideran no principales y se ejecutan en un entorno aislado.
- `all`: todas las sesiones se ejecutan en un entorno aislado.

El **ámbito** controla cuántos contenedores o entornos se crean:

- `agent`: un contenedor por agente.
- `session`: un contenedor por sesión.
- `shared`: un contenedor compartido por todas las sesiones aisladas (las anulaciones por agente `docker`/`ssh`/`browser` se ignoran en este ámbito).

El **backend** controla qué entorno de ejecución ejecuta las herramientas aisladas. La configuración específica de SSH se encuentra en `agents.defaults.sandbox.ssh`; la configuración específica de OpenShell se encuentra en `plugins.entries.openshell.config`.

|                     | Docker                           | SSH                            | OpenShell                                           |
| ------------------- | -------------------------------- | ------------------------------ | --------------------------------------------------- |
| **Dónde se ejecuta**   | Contenedor local                  | Cualquier host accesible mediante SSH        | Entorno aislado administrado por OpenShell                           |
| **Configuración**           | `scripts/sandbox-setup.sh`       | Clave SSH + host de destino          | Plugin de OpenShell habilitado                            |
| **Modelo del espacio de trabajo** | Montaje enlazado o copia               | Remoto canónico (se inicializa una vez)   | `mirror` o `remote`                                |
| **Control de red** | `docker.network` (valor predeterminado: ninguno) | Depende del host remoto         | Depende de OpenShell                                |
| **Navegador aislado** | Compatible                        | No compatible                  | Aún no compatible                                   |
| **Montajes enlazados**     | `docker.binds`                   | N/D                            | N/D                                                 |
| **Ideal para**        | Desarrollo local, aislamiento completo        | Delegar la ejecución a una máquina remota | Entornos aislados remotos administrados con sincronización bidireccional opcional |

## Backend de Docker

Docker es el backend predeterminado una vez que se habilita el aislamiento. Ejecuta las herramientas y los navegadores aislados localmente a través del socket del daemon de Docker (`/var/run/docker.sock`); el aislamiento procede de los espacios de nombres de Docker.

Valores predeterminados: `network: "none"` (sin salida), `readOnlyRoot: true`, `capDrop: ["ALL"]`, imagen `openclaw-sandbox:bookworm-slim`.

Para exponer las GPU del host, establezca `agents.defaults.sandbox.docker.gpus` (o la anulación por agente) en un valor como `"all"` o `"device=GPU-uuid"`. Este valor se pasa a la opción `--gpus` de Docker y requiere un entorno de ejecución de host compatible, como NVIDIA Container Toolkit.

<Warning>
**Docker fuera de Docker (DooD): restricciones**

Si se despliega el Gateway de OpenClaw como un contenedor de Docker, este organiza contenedores aislados hermanos mediante el socket de Docker del host (DooD). Esto introduce una restricción de asignación de rutas:

- **La configuración requiere rutas del host**: `openclaw.json` `workspace` debe contener la **ruta absoluta del host** (p. ej., `/home/user/.openclaw/workspaces`), no la ruta interna del contenedor de Gateway. El daemon de Docker evalúa las rutas en relación con el espacio de nombres del sistema operativo host, no con el espacio de nombres propio de Gateway.
- **Se requiere una asignación de volúmenes coincidente**: el proceso Gateway también escribe archivos de Heartbeat y de puente en esa ruta `workspace`. Asigne al contenedor de Gateway un volumen idéntico (`-v /home/user/.openclaw:/home/user/.openclaw`) para que la misma ruta del host también se resuelva correctamente desde el interior del contenedor de Gateway. Las asignaciones que no coinciden aparecen como `EACCES` cuando Gateway intenta escribir su Heartbeat.
- **Modo de código de Codex**: cuando un entorno aislado de OpenClaw está activo, OpenClaw deshabilita durante ese turno el Modo de código nativo del servidor de aplicaciones de Codex, los servidores MCP del usuario y la ejecución de plugins respaldados por aplicaciones (estos se ejecutan desde el proceso del servidor de aplicaciones en el host de Gateway, no desde el backend de entorno aislado de OpenClaw), a menos que la política de herramientas del entorno aislado exponga las herramientas necesarias y se habilite la ruta experimental del servidor de ejecución del entorno aislado. El acceso al shell se enruta entonces mediante herramientas respaldadas por el entorno aislado de OpenClaw, como `sandbox_exec` y `sandbox_process`. No monte el socket de Docker del host en contenedores aislados de agentes ni en entornos aislados personalizados de Codex. Consulte [Entorno de pruebas de Codex](/es/plugins/codex-harness) para conocer el comportamiento completo.

En hosts Ubuntu/AppArmor con el modo de entorno aislado de Docker habilitado, la ejecución del shell `workspace-write` del servidor de aplicaciones de Codex necesita espacios de nombres de usuario sin privilegios dentro del contenedor aislado, y puede fallar antes de que se inicie el shell cuando el usuario del servicio no puede crearlos. También se necesita un espacio de nombres de red sin privilegios cuando la salida del entorno aislado de Docker está deshabilitada (`network: "none"`, el valor predeterminado). Síntomas habituales: `bwrap: setting up uid map: Permission denied` y `bwrap: loopback: Failed RTM_NEWADDR: Operation not permitted`. Ejecute `openclaw doctor`; si informa de un error en la prueba del espacio de nombres bwrap de Codex, se recomienda usar un perfil de AppArmor que conceda los espacios de nombres necesarios al proceso del servicio OpenClaw. `kernel.apparmor_restrict_unprivileged_userns=0` es una alternativa para todo el host que conlleva concesiones de seguridad; úsela únicamente cuando esa postura del host sea aceptable.
</Warning>

### Navegador aislado

- El navegador aislado se inicia automáticamente (garantiza que CDP sea accesible) cuando la herramienta de navegador lo necesita. Configúrelo mediante `agents.defaults.sandbox.browser.autoStart` (valor predeterminado: `true`) y `autoStartTimeoutMs` (valor predeterminado: 12s).
- Los contenedores de navegadores aislados usan una red de Docker dedicada (`openclaw-sandbox-browser`) en lugar de la red global `bridge`. Configúrela mediante `agents.defaults.sandbox.browser.network`.
- `agents.defaults.sandbox.browser.cdpSourceRange` restringe la entrada de CDP en el borde del contenedor mediante una lista de CIDR permitidos (por ejemplo, `172.21.0.1/32`).
- El acceso de observador de noVNC está protegido mediante contraseña de forma predeterminada; OpenClaw emite una URL con token de corta duración que sirve una página de arranque local y abre noVNC con la contraseña en el fragmento de la URL (no en la cadena de consulta ni en los registros de cabeceras).
- `agents.defaults.sandbox.browser.allowHostControl` (valor predeterminado: `false`) permite que las sesiones aisladas seleccionen explícitamente el navegador del host.
- Las listas de permitidos opcionales controlan `target: "custom"`: `allowedControlUrls`, `allowedControlHosts`, `allowedControlPorts`.

## Backend de SSH

Use `backend: "ssh"` para ejecutar en un entorno aislado `exec`, las herramientas de archivos y las lecturas de medios en cualquier máquina accesible mediante SSH.

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "ssh",
        scope: "session",
        workspaceAccess: "rw",
        ssh: {
          target: "user@gateway-host:22",
          workspaceRoot: "/tmp/openclaw-sandboxes",
          strictHostKeyChecking: true,
          updateHostKeys: true,
          identityFile: "~/.ssh/id_ed25519",
          certificateFile: "~/.ssh/id_ed25519-cert.pub",
          knownHostsFile: "~/.ssh/known_hosts",
          // O use SecretRefs / contenido en línea en lugar de archivos locales:
          // identityData: { source: "env", provider: "default", id: "SSH_IDENTITY" },
          // certificateData: { source: "env", provider: "default", id: "SSH_CERTIFICATE" },
          // knownHostsData: { source: "env", provider: "default", id: "SSH_KNOWN_HOSTS" },
        },
      },
    },
  },
}
```

Valores predeterminados: `command: "ssh"`, `workspaceRoot: "/tmp/openclaw-sandboxes"`, `strictHostKeyChecking: true`, `updateHostKeys: true`.

- **Ciclo de vida**: OpenClaw crea una raíz remota por ámbito en `sandbox.ssh.workspaceRoot`. La primera vez que se usa después de crearla o volver a crearla, inicializa una vez ese espacio de trabajo remoto desde el espacio de trabajo local. Después, `exec`, `read`, `write`, `edit`, `apply_patch`, las lecturas de medios de solicitudes y la preparación de medios entrantes se ejecutan directamente en el espacio de trabajo remoto mediante SSH. OpenClaw no sincroniza automáticamente los cambios remotos con el espacio de trabajo local.
- **Material de autenticación**: `identityFile`/`certificateFile`/`knownHostsFile` hacen referencia a archivos locales existentes. `identityData`/`certificateData`/`knownHostsData` aceptan cadenas en línea o SecretRefs, que se resuelven mediante la instantánea normal del entorno de ejecución de secretos, se escriben en archivos temporales con el modo `0600` y se eliminan cuando termina la sesión SSH. Si se definen tanto una variante `*File` como una variante `*Data` para el mismo elemento, `*Data` prevalece durante esa sesión.
- **Consecuencias del modelo remoto canónico**: el espacio de trabajo SSH remoto se convierte en el estado real del entorno aislado después de la inicialización inicial. Las ediciones locales del host realizadas fuera de OpenClaw después de la inicialización no son visibles de forma remota hasta que se vuelve a crear el entorno aislado. `openclaw sandbox recreate` elimina la raíz remota por ámbito y vuelve a inicializarla desde el entorno local en el siguiente uso. Este backend no admite el aislamiento del navegador y los ajustes `sandbox.docker.*` no se aplican.

## Backend de OpenShell

Use `backend: "openshell"` para ejecutar herramientas en un entorno remoto administrado por OpenShell. OpenShell reutiliza el mismo transporte SSH y el mismo puente del sistema de archivos remoto que el backend SSH genérico, y añade el ciclo de vida de OpenShell (`sandbox create/get/delete/ssh-config`) y un modo opcional de sincronización del espacio de trabajo `mirror`.

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "openshell",
        scope: "session",
        workspaceAccess: "rw",
      },
    },
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "remote", // reflejo | remoto
        },
      },
    },
  },
}
```

`mode: "mirror"` (predeterminado) mantiene canónico el espacio de trabajo local: OpenClaw sincroniza el contenido local con el entorno aislado antes de `exec` y lo vuelve a sincronizar después. `mode: "remote"` inicializa una vez el espacio de trabajo remoto a partir del local y luego ejecuta `exec`/`read`/`write`/`edit`/`apply_patch` directamente en el espacio de trabajo remoto sin sincronizarlo de vuelta; las ediciones locales posteriores a la inicialización no son visibles hasta que se ejecuta `openclaw sandbox recreate`. Con `scope: "agent"` o `scope: "shared"`, ese espacio de trabajo remoto se comparte en el mismo ámbito. Limitaciones actuales: todavía no se admite el navegador del entorno aislado y `sandbox.docker.binds` no se aplica a este backend.

`openclaw sandbox list`/`recreate`/la depuración tratan los runtimes de OpenShell igual que los runtimes de Docker; la lógica de depuración tiene en cuenta el backend.

Para consultar todos los requisitos previos, la referencia de configuración, la comparación de modos del espacio de trabajo y los detalles del ciclo de vida, véase [OpenShell](/es/gateway/openshell).

## Acceso al espacio de trabajo

`agents.defaults.sandbox.workspaceAccess` controla qué puede ver el entorno aislado:

| Valor            | Comportamiento                                                                                  |
| ---------------- | ----------------------------------------------------------------------------------------- |
| `none` (predeterminado) | Las herramientas ven un espacio de trabajo aislado bajo `~/.openclaw/sandboxes`.                    |
| `ro`             | Monta el espacio de trabajo del agente en modo de solo lectura en `/agent` (deshabilita `write`/`edit`/`apply_patch`). |
| `rw`             | Monta el espacio de trabajo del agente en modo de lectura y escritura en `/workspace`.                                    |

Con el backend de OpenShell, el modo `mirror` sigue usando el espacio de trabajo local como fuente canónica entre turnos de ejecución, el modo `remote` usa el espacio de trabajo remoto de OpenShell como canónico después de la inicialización y `workspaceAccess: "ro"`/`"none"` siguen restringiendo el comportamiento de escritura de la misma manera.

Los archivos multimedia entrantes se copian en el espacio de trabajo activo del entorno aislado (`media/inbound/*`).

<Note>
**Skills**: la herramienta `read` tiene como raíz el entorno aislado. Con `workspaceAccess: "none"`, OpenClaw replica las Skills aptas en el espacio de trabajo del entorno aislado (`.../skills`) para que se puedan leer. Con `"rw"`, las Skills del espacio de trabajo se pueden leer desde `/workspace/skills`, y las Skills aptas administradas, integradas o de plugins se materializan en la ruta generada de solo lectura `/workspace/.openclaw/sandbox-skills/skills`.
</Note>

## Varias carpetas para un agente

Utilice montajes enlazados de Docker cuando un agente en un entorno aislado necesite más carpetas además de su espacio de trabajo principal. Cada entrada asigna una carpeta del host a una ruta del contenedor con un modo de acceso explícito:

```text
directorio-del-host:directorio-del-contenedor:ro
directorio-del-host:directorio-del-contenedor:rw
```

- `ro` hace que la carpeta montada sea de solo lectura dentro del entorno aislado.
- `rw` permite que las herramientas y los procesos del entorno aislado modifiquen la carpeta del host.
- La ruta del contenedor es la ruta que utiliza el agente. Las rutas del host no se exponen automáticamente.

Este ejemplo proporciona al agente `research` un espacio de trabajo principal con permisos de escritura, material de referencia de solo lectura en `/reference` y una carpeta de salida independiente con permisos de escritura en `/drafts`:

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        scope: "agent",
      },
    },
    list: [
      {
        id: "research",
        workspace: "/srv/openclaw/research-workspace",
        sandbox: {
          workspaceAccess: "rw",
          docker: {
            binds: ["/srv/shared/reference:/reference:ro", "/srv/shared/drafts:/drafts:rw"],
            // Obligatorio porque estos orígenes están fuera del espacio de trabajo del agente.
            dangerouslyAllowExternalBindSources: true,
          },
        },
      },
    ],
  },
}
```

`workspaceAccess` y los modos de enlace son independientes:

| Configuración                          | Controla                                                                    |
| -------------------------------- | --------------------------------------------------------------------------- |
| `workspaceAccess: "none"`        | Utiliza un espacio de trabajo aislado; no expone el espacio de trabajo del agente.    |
| `workspaceAccess: "ro"`          | Monta el espacio de trabajo del agente en modo de solo lectura en `/agent`.                           |
| `workspaceAccess: "rw"`          | Monta el espacio de trabajo del agente en modo de lectura y escritura en `/workspace`.                      |
| Entrada `docker.binds` `:ro`/`:rw` | Controla únicamente esa carpeta adicional del host en la ruta configurada del contenedor. |

Cambiar `workspaceAccess` no cambia un enlace adicional de `ro` a `rw`, ni viceversa. Los valores globales y por agente de `docker.binds` se combinan. Mantenga `scope: "agent"` o `"session"` para los enlaces por agente; `scope: "shared"` ignora todas las sustituciones de Docker por agente y utiliza únicamente los enlaces globales.

Los montajes enlazados son el límite admitido para varias carpetas porque Docker construye la vista del sistema de archivos del contenedor con aislamiento de montajes, y el modo `ro`/`rw` se aplica a todos los procesos del entorno aislado. Ese límite abarca `exec`, las herramientas del sistema de archivos, los procesos secundarios y las bibliotecas sin duplicar las comprobaciones de autorización de rutas en cada ruta de código de OpenClaw. Una lista de rutas permitidas en el host no puede proporcionar el mismo límite completo cuando un shell o una dependencia permitidos pueden acceder directamente a los archivos.

La opción de participación voluntaria `dangerouslyAllowExternalBindSources` solo permite orígenes situados fuera de las raíces del espacio de trabajo. No deshabilita las comprobaciones de OpenClaw sobre el sistema, las credenciales, el socket de Docker, los directorios superiores con enlaces simbólicos ni los destinos reservados. Utilice la carpeta más pequeña posible, use `ro` salvo que se requieran escrituras y vuelva a crear el entorno aislado después de cambiar los montajes:

```bash
openclaw sandbox recreate --agent research
```

### Otros comportamientos de los enlaces

`agents.defaults.sandbox.docker.binds` configura los montajes globales. El formato es el mismo `host:container:mode` (por ejemplo, `"/home/user/source:/source:rw"`).

`agents.defaults.sandbox.browser.binds` monta directorios adicionales del host únicamente en el contenedor del **navegador del entorno aislado**. Cuando se establece (incluido `[]`), sustituye a `docker.binds` para el contenedor del navegador; cuando se omite, el contenedor del navegador recurre a `docker.binds`.

```json5
{
  agents: {
    defaults: {
      sandbox: {
        docker: {
          binds: ["/home/user/source:/source:ro", "/var/data/myapp:/data:ro"],
        },
      },
    },
    list: [
      {
        id: "build",
        sandbox: {
          docker: {
            binds: ["/mnt/cache:/cache:rw"],
          },
        },
      },
    ],
  },
}
```

<Warning>
**Seguridad de los enlaces**

- Los enlaces omiten el sistema de archivos del entorno aislado: exponen las rutas del host con el modo que se establezca (`:ro` o `:rw`).
- OpenClaw bloquea de forma predeterminada los orígenes de enlace peligrosos: rutas del sistema (`/etc`, `/proc`, `/sys`, `/dev`, `/root`, `/boot`), directorios del socket de Docker (`/run`, `/var/run` y sus variantes `docker.sock`) y raíces habituales de credenciales en el directorio personal (`~/.aws`, `~/.cargo`, `~/.config`, `~/.docker`, `~/.gnupg`, `~/.netrc`, `~/.npm`, `~/.ssh`).
- La validación normaliza la ruta de origen y luego vuelve a resolverla a través del ancestro existente más profundo antes de comprobar de nuevo las rutas bloqueadas y las raíces permitidas, por lo que los escapes mediante un directorio superior con enlace simbólico se rechazan de forma segura incluso si la hoja final aún no existe (por ejemplo, `/workspace/run-link/new-file` sigue resolviéndose como `/var/run/...` si `run-link` apunta allí).
- Los destinos de enlace que ocultan los puntos de montaje reservados del contenedor (`/workspace`, `/agent`) también se bloquean de forma predeterminada; use `agents.defaults.sandbox.docker.dangerouslyAllowReservedContainerTargets: true` para sustituir este comportamiento.
- Los orígenes de enlace situados fuera de las raíces permitidas del espacio de trabajo o del espacio de trabajo del agente se bloquean de forma predeterminada; use `agents.defaults.sandbox.docker.dangerouslyAllowExternalBindSources: true` para sustituir este comportamiento. Las raíces permitidas se canonizan de la misma manera, por lo que una ruta que solo parece estar dentro de la lista de permitidas antes de resolver los enlaces simbólicos se sigue rechazando por estar fuera de las raíces permitidas.
- Los montajes confidenciales (secretos, claves SSH y credenciales de servicios) deben ser `:ro`, salvo que sea absolutamente necesario lo contrario.
- Combine esta opción con `workspaceAccess: "ro"` si solo necesita acceso de lectura al espacio de trabajo; los modos de enlace siguen siendo independientes.
- Consulte [Entorno aislado frente a política de herramientas frente a ejecución elevada](/es/gateway/sandbox-vs-tool-policy-vs-elevated) para saber cómo interactúan los enlaces con la política de herramientas y la ejecución elevada.

</Warning>

## Imágenes y configuración

Imagen predeterminada de Docker: `openclaw-sandbox:bookworm-slim`

<Note>
**Checkout del código fuente frente a instalación mediante npm**

Los scripts auxiliares `scripts/sandbox-setup.sh`, `scripts/sandbox-common-setup.sh` y `scripts/sandbox-browser-setup.sh` solo están disponibles cuando se ejecuta desde un [checkout del código fuente](https://github.com/openclaw/openclaw). No se incluyen en el paquete npm.

Si instaló OpenClaw mediante `npm install -g openclaw`, utilice en su lugar los comandos `docker build` integrados que se muestran a continuación.
</Note>

<Steps>
  <Step title="Compilar la imagen predeterminada">
    Desde un checkout del código fuente:

    ```bash
    scripts/sandbox-setup.sh
    ```

    Desde una instalación mediante npm (no se necesita un checkout del código fuente):

    ```bash
    docker build -t openclaw-sandbox:bookworm-slim - <<'DOCKERFILE'
    FROM debian:bookworm-slim
    ENV DEBIAN_FRONTEND=noninteractive
    RUN apt-get update && apt-get install -y --no-install-recommends \
      bash ca-certificates curl git jq python3 ripgrep \
      && rm -rf /var/lib/apt/lists/*
    RUN useradd --create-home --shell /bin/bash sandbox
    USER sandbox
    WORKDIR /home/sandbox
    CMD ["sleep", "infinity"]
    DOCKERFILE
    ```

    La imagen predeterminada **no** incluye Node. Si una Skill necesita Node (u otros runtimes), cree una imagen personalizada que los incluya o instálelos mediante `sandbox.docker.setupCommand` (requiere salida de red, una raíz con permisos de escritura y el usuario root).

    OpenClaw no sustituye silenciosamente `openclaw-sandbox:bookworm-slim` por `debian:bookworm-slim` cuando falta. Las ejecuciones del entorno aislado dirigidas a la imagen predeterminada fallan de inmediato y muestran instrucciones de compilación hasta que se compile, porque la imagen integrada incluye `python3` para los auxiliares de escritura y edición del entorno aislado.

  </Step>
  <Step title="Opcional: compilar la imagen común">
    Para disponer de una imagen de entorno aislado más funcional con herramientas habituales (por ejemplo, `curl`, `jq`, Node 24, pnpm, `python3` y `git`):

    Desde un checkout del código fuente:

    ```bash
    scripts/sandbox-common-setup.sh
    ```

    Desde una instalación mediante npm, compile primero la imagen predeterminada (véase más arriba) y luego compile sobre ella la imagen común utilizando [`scripts/docker/sandbox/Dockerfile.common`](https://github.com/openclaw/openclaw/blob/main/scripts/docker/sandbox/Dockerfile.common) del repositorio.

    A continuación, establezca `agents.defaults.sandbox.docker.image` en `openclaw-sandbox-common:bookworm-slim`.

  </Step>
  <Step title="Opcional: compilar la imagen del navegador del entorno aislado">
    Desde un checkout del código fuente:

    ```bash
    scripts/sandbox-browser-setup.sh
    ```

    Desde una instalación mediante npm, compile utilizando [`scripts/docker/sandbox/Dockerfile.browser`](https://github.com/openclaw/openclaw/blob/main/scripts/docker/sandbox/Dockerfile.browser) del repositorio.

  </Step>
</Steps>

De forma predeterminada, los contenedores del entorno aislado de Docker se ejecutan **sin red**. Sustituya este comportamiento con `agents.defaults.sandbox.docker.network`.

<AccordionGroup>
  <Accordion title="Valores predeterminados de Chromium para el navegador del entorno aislado">
    La imagen integrada del navegador del entorno aislado aplica indicadores conservadores de inicio de Chromium para cargas de trabajo en contenedores:

    - `--remote-debugging-address=127.0.0.1`
    - `--remote-debugging-port=<derived from OPENCLAW_BROWSER_CDP_PORT>`
    - `--user-data-dir=${HOME}/.chrome`
    - `--no-first-run`
    - `--no-default-browser-check`
    - `--disable-dev-shm-usage`
    - `--disable-background-networking`
    - `--disable-breakpad`
    - `--disable-crash-reporter`
    - `--no-zygote`
    - `--metrics-recording-only`
    - `--password-store=basic`
    - `--use-mock-keychain`
    - `--headless=new` cuando `browser.headless` está habilitado.
    - `--no-sandbox --disable-setuid-sandbox` cuando `browser.noSandbox` está habilitado.
    - `--disable-3d-apis`, `--disable-gpu`, `--disable-software-rasterizer` de forma predeterminada; estas opciones de refuerzo gráfico ayudan a los contenedores sin compatibilidad con GPU. Establezca `OPENCLAW_BROWSER_DISABLE_GRAPHICS_FLAGS=0` si la carga de trabajo necesita WebGL u otras funciones 3D.
    - `--disable-extensions` de forma predeterminada; establezca `OPENCLAW_BROWSER_DISABLE_EXTENSIONS=0` para los flujos que dependan de extensiones.
    - `--renderer-process-limit=2` de forma predeterminada; se controla mediante `OPENCLAW_BROWSER_RENDERER_PROCESS_LIMIT=<N>`, donde `0` conserva el valor predeterminado de Chromium.

    Si se necesita un perfil de entorno de ejecución diferente, utilice una imagen de navegador personalizada y proporcione un punto de entrada propio. Para los perfiles locales de Chromium (sin contenedor), utilice `browser.extraArgs` para añadir opciones de inicio adicionales.

  </Accordion>
  <Accordion title="Valores predeterminados de seguridad de red">
    - `network: "host"` está bloqueado.
    - `network: "container:<id>"` está bloqueado de forma predeterminada (riesgo de eludir las restricciones al unirse al espacio de nombres).
    - Excepción de emergencia: `agents.defaults.sandbox.docker.dangerouslyAllowContainerNamespaceJoin: true`.

  </Accordion>
</AccordionGroup>

Las instalaciones de Docker y el Gateway en contenedor se encuentran aquí: [Docker](/es/install/docker)

Para las implementaciones del Gateway con Docker, `scripts/docker/setup.sh` puede inicializar la configuración del entorno aislado. Establezca `OPENCLAW_SANDBOX=1` (o `true`/`yes`/`on`) para habilitar esa ruta. Sobrescriba la ubicación del socket con `OPENCLAW_DOCKER_SOCKET`. Configuración completa y referencia de variables de entorno: [Docker](/es/install/docker#agent-sandbox).

## setupCommand (configuración única del contenedor)

`setupCommand` se ejecuta **una vez** después de crear el contenedor del entorno aislado (no en cada ejecución). Se ejecuta dentro del contenedor mediante `sh -lc`.

Rutas:

- Global: `agents.defaults.sandbox.docker.setupCommand`
- Por agente: `agents.entries.*.sandbox.docker.setupCommand`

<AccordionGroup>
  <Accordion title="Errores comunes">
    - El valor predeterminado de `docker.network` es `"none"` (sin salida de red), por lo que las instalaciones de paquetes fallarán.
    - `docker.network: "container:<id>"` requiere `dangerouslyAllowContainerNamespaceJoin: true` y solo debe utilizarse como medida de emergencia.
    - `readOnlyRoot: true` impide las escrituras; establezca `readOnlyRoot: false` o prepare una imagen personalizada.
    - `user` debe ser el usuario raíz para instalar paquetes (omita `user` o establezca `user: "0:0"`).
    - La ejecución en el entorno aislado **no** hereda `process.env` del host. Utilice `agents.defaults.sandbox.docker.env` (o una imagen personalizada) para las claves de API de Skills.
    - Los valores de `agents.defaults.sandbox.docker.env` se pasan como variables de entorno explícitas del contenedor Docker. Cualquier persona con acceso al daemon de Docker puede inspeccionarlos mediante comandos de metadatos de Docker como `docker inspect`. Utilice una imagen personalizada, un archivo de secretos montado u otra vía de entrega de secretos si esta exposición de metadatos no es aceptable.

  </Accordion>
</AccordionGroup>

## Política de herramientas y vías de escape

Las políticas de autorización y denegación de herramientas siguen aplicándose antes que las reglas del entorno aislado. Si una herramienta está denegada globalmente o para un agente, el aislamiento no vuelve a habilitarla.

`tools.elevated` es una vía de escape explícita que ejecuta `exec` fuera del entorno aislado (`gateway` de forma predeterminada, o `node` cuando el destino de ejecución es `node`). Las directivas `/exec` solo se aplican a remitentes autorizados y persisten por sesión; para deshabilitar por completo `exec`, utilice la denegación de la política de herramientas (consulte [Entorno aislado frente a política de herramientas y modo elevado](/es/gateway/sandbox-vs-tool-policy-vs-elevated)).

Depuración:

- `openclaw sandbox list` muestra los contenedores de los entornos aislados, su estado, la coincidencia de imagen, la antigüedad, el tiempo de inactividad y la sesión o el agente asociados.
- `openclaw sandbox explain [--session <key>] [--agent <id>]` inspecciona el modo efectivo del entorno aislado, el espacio de trabajo del host, el directorio de trabajo del entorno de ejecución, los montajes de Docker, la política de herramientas y las claves de configuración para corregir problemas. Su campo `workspaceRoot` continúa siendo la raíz configurada del entorno aislado; `effectiveHostWorkspaceRoot` muestra dónde se encuentra realmente el espacio de trabajo activo.
- `openclaw sandbox recreate [--all | --session <key> | --agent <id>] [--browser] [--force]` elimina los contenedores y entornos para que se vuelvan a crear con la configuración actual la próxima vez que se utilicen.
- Consulte [Entorno aislado frente a política de herramientas y modo elevado](/es/gateway/sandbox-vs-tool-policy-vs-elevated) para entender por qué algo está bloqueado.

## Sobrescrituras multiagente

Cada agente puede sobrescribir el entorno aislado y las herramientas: `agents.entries.*.sandbox` y `agents.entries.*.tools` (además de `agents.entries.*.tools.sandbox.tools` para la política de herramientas del entorno aislado). Consulte [Entorno aislado y herramientas multiagente](/es/tools/multi-agent-sandbox-tools) para conocer la precedencia.

## Ejemplo mínimo de habilitación

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",
        scope: "session",
        workspaceAccess: "none",
      },
    },
  },
}
```

## Temas relacionados

- [Entorno aislado y herramientas multiagente](/es/tools/multi-agent-sandbox-tools) -- sobrescrituras por agente y precedencia
- [OpenShell](/es/gateway/openshell) -- configuración del backend de entorno aislado administrado, modos del espacio de trabajo y referencia de configuración
- [Configuración del entorno aislado](/es/gateway/config-agents#agentsdefaultssandbox)
- [Entorno aislado frente a política de herramientas y modo elevado](/es/gateway/sandbox-vs-tool-policy-vs-elevated) -- depuración de «¿por qué está bloqueado?»
- [Seguridad](/es/gateway/security)
