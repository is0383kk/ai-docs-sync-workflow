---
read_when:
    - Se necesitan entornos aislados administrados en la nube en lugar de Docker local
    - Está configurando el plugin OpenShell
    - Debe elegir entre los modos de espacio de trabajo espejo y remoto.
summary: Usar OpenShell como backend de sandbox gestionado para agentes de OpenClaw
title: OpenShell
x-i18n:
    generated_at: "2026-07-26T04:41:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: bf5c33912bd0db759a01cf58ea26712a8ada68c0804bf16f69f1f7cdd496828c
    source_path: gateway/openshell.md
    workflow: 16
---

OpenShell es un backend de entorno aislado gestionado: en lugar de ejecutar contenedores Docker
localmente, OpenClaw delega el ciclo de vida del entorno aislado a la CLI `openshell`, que
aprovisiona entornos remotos y ejecuta comandos mediante SSH.

El plugin reutiliza el mismo transporte SSH y puente del sistema de archivos remoto que el
[backend SSH](/es/gateway/sandboxing#ssh-backend) genérico, y añade el ciclo de vida de OpenShell
(`sandbox create/get/delete/ssh-config`), además de un modo opcional de sincronización del espacio de trabajo
`mirror`.

## Requisitos previos

- Plugin de OpenShell instalado (`openclaw plugins install @openclaw/openshell-sandbox`)
- CLI `openshell` en `PATH` (o una ruta personalizada mediante
  `plugins.entries.openshell.config.command`)
- Una cuenta de OpenShell con acceso a entornos aislados
- Gateway de OpenClaw ejecutándose en el host

## Inicio rápido

```bash
openclaw plugins install @openclaw/openshell-sandbox
```

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
          mode: "remote",
        },
      },
    },
  },
}
```

Reinicie el Gateway. En el siguiente turno del agente, OpenClaw crea un entorno aislado
de OpenShell y dirige a través de él la ejecución de herramientas. Verifíquelo con:

```bash
openclaw sandbox list
openclaw sandbox explain
```

## Modos del espacio de trabajo

Esta es la decisión más importante de OpenShell.

### mirror (predeterminado)

`plugins.entries.openshell.config.mode: "mirror"` mantiene el **espacio de trabajo local
como canónico**:

- Antes de `exec`, OpenClaw sincroniza el espacio de trabajo local con el entorno aislado.
- Después de `exec`, OpenClaw vuelve a sincronizar el espacio de trabajo remoto con el local.
- Las herramientas de archivos utilizan el puente del entorno aislado, pero el entorno local sigue siendo la fuente de verdad
  entre turnos.

Es la mejor opción para flujos de trabajo de desarrollo: las ediciones locales realizadas fuera de OpenClaw aparecen en la
siguiente ejecución, y el entorno aislado se comporta de forma similar al backend de Docker.

Desventaja: coste de carga y descarga en cada turno de ejecución.

### remote

`mode: "remote"` hace que el **espacio de trabajo de OpenShell sea canónico**:

- Durante la primera creación del entorno aislado, OpenClaw inicializa una vez el espacio de trabajo remoto a partir del local.
- A partir de entonces, `exec`, `read`, `write`, `edit` y `apply_patch` operan
  directamente en el espacio de trabajo remoto. OpenClaw **no** vuelve a sincronizar los cambios remotos
  con el entorno local.
- La lectura de contenido multimedia al generar el prompt sigue funcionando (las herramientas de archivos y contenido multimedia leen mediante el
  puente del entorno aislado).

Es la mejor opción para agentes de larga duración y CI: menor sobrecarga por turno y las
ediciones locales del host no pueden sobrescribir silenciosamente el estado remoto.

<Warning>
Las ediciones de archivos realizadas en el host fuera de OpenClaw después de la inicialización no son visibles para el entorno aislado remoto. Ejecute `openclaw sandbox recreate` para volver a inicializarlo.
</Warning>

### Elección de un modo

|                          | `mirror`                   | `remote`                  |
| ------------------------ | -------------------------- | ------------------------- |
| **Espacio de trabajo canónico** | Host local                 | OpenShell remoto          |
| **Dirección de sincronización** | Bidireccional (en cada ejecución) | Inicialización única      |
| **Sobrecarga por turno** | Mayor (carga y descarga) | Menor (operaciones remotas directas) |
| **¿Se ven las ediciones locales?** | Sí, en la siguiente ejecución | No, hasta volver a crear |
| **Opción ideal para**    | Flujos de trabajo de desarrollo | Agentes de larga duración, CI |

## Referencia de configuración

Toda la configuración de OpenShell se encuentra en `plugins.entries.openshell.config`:

| Clave                     | Tipo                     | Valor predeterminado | Descripción                                                                            |
| ------------------------- | ------------------------ | ------------- | -------------------------------------------------------------------------------------- |
| `mode`                    | `"mirror"` o `"remote"` | `"mirror"`    | Modo de sincronización del espacio de trabajo                                           |
| `command`                 | `string`                 | `"openshell"` | Ruta o nombre de la CLI `openshell`                                                    |
| `from`                    | `string`                 | `"openclaw"`  | Origen del entorno aislado para la primera creación                                     |
| `gateway`                 | `string`                 | sin definir   | Nombre del gateway de OpenShell (`--gateway` de nivel superior)                         |
| `gatewayEndpoint`         | `string`                 | sin definir   | Endpoint del gateway de OpenShell (`--gateway-endpoint` de nivel superior)              |
| `policy`                  | `string`                 | sin definir   | ID de política de OpenShell para la creación del entorno aislado                        |
| `providers`               | `string[]`               | `[]`          | Nombres de proveedores asociados durante la creación del entorno aislado (sin duplicados, una marca `--provider` por entrada) |
| `gpu`                     | `boolean`                | `false`       | Solicitar recursos de GPU (`--gpu`)                                             |
| `autoProviders`           | `boolean`                | `true`        | Pasar `--auto-providers` (o `--no-auto-providers` cuando es falso) durante la creación |
| `remoteWorkspaceDir`      | `string`                 | `"/sandbox"`  | Espacio de trabajo principal con escritura dentro del entorno aislado                   |
| `remoteAgentWorkspaceDir` | `string`                 | `"/agent"`    | Ruta de montaje del espacio de trabajo del agente (solo lectura cuando el acceso al espacio de trabajo no es `rw`) |
| `timeoutSeconds`          | `number`                 | `120`         | Tiempo de espera para las operaciones de la CLI `openshell`                          |

`remoteWorkspaceDir` y `remoteAgentWorkspaceDir` deben ser rutas absolutas y
permanecer bajo las raíces gestionadas `/sandbox` o `/agent`; las demás rutas absolutas se
rechazan.

Los ajustes del entorno aislado (`mode`, `scope`, `workspaceAccess`) se encuentran en
`agents.defaults.sandbox`, como con cualquier backend. Consulte
[Entornos aislados](/es/gateway/sandboxing) para ver la matriz completa.

## Ejemplos

### Configuración remota mínima

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "openshell",
      },
    },
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "remote",
        },
      },
    },
  },
}
```

### Modo mirror con GPU

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "openshell",
        scope: "agent",
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
          mode: "mirror",
          gpu: true,
          providers: ["openai"],
          timeoutSeconds: 180,
        },
      },
    },
  },
}
```

### OpenShell por agente con gateway personalizado

```json5
{
  agents: {
    defaults: {
      sandbox: { mode: "off" },
    },
    list: [
      {
        id: "researcher",
        sandbox: {
          mode: "all",
          backend: "openshell",
          scope: "agent",
          workspaceAccess: "rw",
        },
      },
    ],
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "remote",
          gateway: "lab",
          gatewayEndpoint: "https://lab.example",
          policy: "strict",
        },
      },
    },
  },
}
```

## Gestión del ciclo de vida

```bash
# Enumerar todos los entornos de ejecución aislados (Docker + OpenShell)
openclaw sandbox list

# Inspeccionar la política efectiva
openclaw sandbox explain

# Volver a crear (elimina el espacio de trabajo remoto y vuelve a inicializarlo en el siguiente uso)
openclaw sandbox recreate --all
```

En el modo `remote`, volver a crear es especialmente importante: elimina el espacio de trabajo
remoto canónico de ese ámbito, y el siguiente uso inicializa uno nuevo a partir del
entorno local. En el modo `mirror`, volver a crear restablece principalmente el entorno de ejecución
remoto, ya que el entorno local sigue siendo canónico.

Vuelva a crear el entorno después de cambiar cualquiera de estos elementos:

- `agents.defaults.sandbox.backend`
- `plugins.entries.openshell.config.from`
- `plugins.entries.openshell.config.mode`
- `plugins.entries.openshell.config.policy`

## Refuerzo de la seguridad

El puente del sistema de archivos del modo mirror fija la raíz del espacio de trabajo local y vuelve a comprobar
las rutas canónicas (mediante realpath) antes de cada lectura, escritura, creación de directorio, eliminación y
cambio de nombre, y rechaza los enlaces simbólicos en puntos intermedios de la ruta. Un cambio de enlace simbólico o un espacio de trabajo vuelto a montar
no puede redirigir el acceso a archivos fuera del árbol reflejado.

## Limitaciones actuales

- El navegador del entorno aislado no es compatible con el backend de OpenShell.
- `sandbox.docker.binds` no se aplica a OpenShell; la creación del entorno aislado falla
  si hay enlaces configurados.
- Los ajustes de ejecución específicos de Docker en `sandbox.docker.*` (salvo `env`)
  solo se aplican al backend de Docker.

## Cómo funciona

1. OpenClaw ejecuta `sandbox get` para el nombre del entorno aislado (con cualquier
   `--gateway`/`--gateway-endpoint` configurado); si falla, crea uno con
   `sandbox create`, pasando `--name`, `--from`, `--policy` cuando estén definidos, `--gpu`
   cuando esté habilitado, `--auto-providers`/`--no-auto-providers` y una
   marca `--provider` por cada proveedor configurado.
2. OpenClaw ejecuta `sandbox ssh-config` para el nombre del entorno aislado a fin de obtener los datos
   de conexión SSH.
3. El núcleo escribe la configuración SSH en un archivo temporal y abre una sesión SSH mediante
   el mismo puente del sistema de archivos remoto que el backend SSH genérico.
4. En el modo `mirror`: sincroniza el entorno local con el remoto antes de la ejecución, ejecuta y vuelve a sincronizar después.
5. En el modo `remote`: inicializa una vez durante la creación y luego opera directamente en el espacio de trabajo
   remoto.

## Contenido relacionado

- [Entornos aislados](/es/gateway/sandboxing) - modos, ámbitos y comparación de backends
- [Entorno aislado frente a política de herramientas frente a privilegios elevados](/es/gateway/sandbox-vs-tool-policy-vs-elevated) - depuración de herramientas bloqueadas
- [Entornos aislados y herramientas multiagente](/es/tools/multi-agent-sandbox-tools) - anulaciones por agente
- [CLI de entornos aislados](/es/cli/sandbox) - comandos `openclaw sandbox`
