---
read_when:
    - Está creando o validando un manifiesto CLAW.md
    - Quieres previsualizar o añadir un agente desde un Claw
    - Necesita inspeccionar la propiedad, la deriva o el comportamiento de limpieza de Claw
summary: Crear, añadir, actualizar y eliminar paquetes experimentales de agentes Claw
title: Garras
x-i18n:
    generated_at: "2026-07-26T04:32:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: da4b52bdee2b4cf4898677aadeeabb2c0cf98e7c3c53cec6f0b4c6d0b8ab3ae5
    source_path: cli/claws.md
    workflow: 16
---

# `openclaw claws`

Un Claw es una configuración versionada para un nuevo agente de OpenClaw. Puede describir la
identidad portátil del agente, los archivos del espacio de trabajo, las Skills, los plugins, los servidores MCP y
los trabajos de Cron. La configuración del agente específica del entorno de ejecución puede incluirse en un
perfil de paquete referenciado. Un Claw no sustituye ni modifica un agente existente.

Los Claws son experimentales. Su esquema, la salida de los comandos y su ciclo de vida pueden cambiar.
Habilite explícitamente la superficie de comandos:

```bash
export OPENCLAW_EXPERIMENTAL_CLAWS=1
```

La CLI actual lee un directorio de paquete local, `CLAW.md` o un manifiesto JSON agrupado.
La publicación, búsqueda e instalación de Claws completos mediante ClawHub forman una
vía de registro independiente y todavía no forman parte de esta superficie de comandos.

## Crear un paquete de Claw

Un paquete contiene `package.json`, un manifiesto `CLAW.md` y cualquier perfil o
archivo auxiliar del espacio de trabajo al que haga referencia dicho manifiesto:

```json
{
  "name": "@acme/incident-triage-claw",
  "version": "1.0.0",
  "type": "module",
  "openclaw": { "claw": "CLAW.md" }
}
```

`CLAW.md` comienza con frontmatter YAML. Su cuerpo Markdown describe el Claw
para las personas y no forma parte de la configuración del agente:

```md
---
schemaVersion: 1
agent:
  id: incident-triage
  name: Clasificación de incidentes
metadata:
  openclaw.config: profiles/openclaw.yml
workspace:
  bootstrapFiles: {}
packages: []
mcpServers: {}
cronJobs: []
---

# Clasificación de incidentes

Crea un agente para revisar y derivar incidentes.
```

`metadata` es un mapa de cadenas a cadenas para indicaciones portátiles destinadas a los consumidores. La clave
`openclaw.config` de OpenClaw apunta a un perfil YAML opcional relativo al paquete. El
valor predeterminado exportado es `profiles/openclaw.yml`; el puntero es normativo, por lo que un
paquete puede elegir otra ruta relativa segura `.yml` o `.yaml`.

```yaml
schemaVersion: 1
agent:
  tools:
    profile: coding
    alsoAllow: [cron]
    deny: [exec]
    fs:
      workspaceOnly: true
  memory:
    search:
      enabled: true
      rememberAcrossConversations: true
      sources: [memory, sessions]
```

Este perfil existe únicamente dentro del paquete de Claw. OpenClaw lo valida y lo utiliza
al inspeccionar, añadir, actualizar y exportar ese Claw; no se copia
a la ruta de configuración normal de OpenClaw del usuario. Otros entornos de ejecución pueden ignorar
la clave de metadatos con espacio de nombres y consumir los campos portátiles del manifiesto.

El mismo esquema estricto de la versión 1 sigue aceptando manifiestos JSON agrupados.
El JSON agrupado utiliza el mismo puntero `metadata.openclaw.config` en lugar de
incrustar una segunda copia del perfil de OpenClaw. Los fragmentos de esquema restantes
de esta página utilizan JSON, con claves equivalentes disponibles en el frontmatter de `CLAW.md`.

El perfil de paquete de OpenClaw puede seleccionar cualquier perfil de herramientas integrado registrado por
la versión de OpenClaw en ejecución y, después, perfeccionarlo con `alsoAllow`, `deny` y
`tools.fs.workspaceOnly: true`. Un Claw no puede establecer ese campo en `false` y
debilitar el confinamiento del sistema de archivos del host. `tools.allow` sigue disponible como una
lista de permitidos explícita, pero no puede combinarse con `alsoAllow`. Un Claw también puede establecer
`memory.search.enabled`, elegir las fuentes portátiles `memory` y `sessions`
y habilitar la memoria entre conversaciones mediante `rememberAcrossConversations`.
Declarar la fuente `sessions` requiere dicha habilitación.
La política del host sigue restringiendo esta configuración, y los Claws no incluyen definiciones
de perfiles personalizadas, proveedores, credenciales, vinculaciones ni rutas de memoria locales.
El perfil referenciado está limitado a 256 KiB, debe ser YAML compatible con JSON, no puede
utilizar alias, anclas, etiquetas ni claves de combinación y debe ser un archivo normal,
sin enlaces simbólicos ni enlaces físicos, dentro del paquete.

Las rutas del paquete y del espacio de trabajo deben permanecer dentro de la raíz del paquete. Los manifiestos están
limitados a 1 MiB, los metadatos del paquete a 256 KiB y las fuentes del espacio de trabajo aplican
límites independientes por archivo y agregados. Las fuentes del espacio de trabajo también rechazan
directorios superiores con enlaces simbólicos.

Los archivos del espacio de trabajo se declaran mediante una ruta y se leen desde archivos auxiliares del paquete. Los archivos
de arranque, como `SOUL.md`, utilizan entradas con nombre; los archivos adicionales utilizan fuentes
relativas al paquete y destinos relativos al espacio de trabajo:

```json
{
  "workspace": {
    "bootstrapFiles": {
      "SOUL.md": { "source": "workspace/SOUL.md" }
    },
    "files": [
      {
        "source": "workspace/reference/policy.md",
        "path": "reference/policy.md"
      }
    ]
  }
}
```

Las Skills y los plugins utilizan versiones exactas de ClawHub:

```json
{
  "packages": [
    {
      "kind": "skill",
      "source": "clawhub",
      "ref": "incident-triage",
      "version": "1.0.0"
    },
    {
      "kind": "plugin",
      "source": "clawhub",
      "ref": "@acme/audit-plugin",
      "version": "2.0.0"
    }
  ]
}
```

La ejecución de prueba utiliza las rutas de comprobación previa existentes de Skills y plugins para resolver el
artefacto exacto, la integridad y cualquier advertencia de confianza de ClawHub antes del consentimiento. La
advertencia permanece visible en el plan vinculado a la integridad. La aplicación instala los artefactos que faltan
o reutiliza los que coinciden y registra si el Claw introdujo cada recurso o hizo referencia
a él. Los plugins siguen siendo capacidades de OpenClaw para todo el proceso, en lugar de
instalaciones por agente.

Los trabajos de Cron declaran tareas programadas para el nuevo agente:

```json
{
  "cronJobs": [
    {
      "id": "daily-summary",
      "name": "Resumen diario de incidentes",
      "schedule": { "cron": "0 9 * * *", "timezone": "UTC" },
      "session": "isolated",
      "message": "Resume los incidentes activos."
    }
  ]
}
```

Los Claws utilizan el programador existente del Gateway y vinculan los trabajos creados al nuevo
agente. La vista previa, la procedencia, el estado y la eliminación abarcan esos trabajos sin
cambiar el comportamiento de los comandos de Cron ordinarios. La eliminación vuelve a leer el trabajo activo
mediante el Gateway y lo conserva cuando su definición gestionada ha cambiado después
de la planificación.

Las declaraciones MCP utilizan el modelo de configuración `mcp.servers` existente:

```json
{
  "mcpServers": {
    "statuspage": {
      "command": "npx",
      "args": ["--yes", "@acme/statuspage-mcp@1.0.0"],
      "env": { "STATUSPAGE_TOKEN": "${STATUSPAGE_TOKEN}" }
    }
  }
}
```

Las referencias de entorno siguen siendo referencias; los Claws no incrustan valores de secretos
resueltos. Una declaración sin colisiones pasa a estar gestionada, mientras que se hace referencia a una declaración existente
exacta o compartida. La vista previa, la procedencia, el estado, la exportación y la
eliminación siguen la misma política de propiedad que los demás recursos de Claw.

## Inspeccionar y obtener una vista previa

Valide la fuente sin planificar cambios locales:

```bash
openclaw claws inspect ./incident-triage.claw.json
```

Obtenga una vista previa de todas las acciones propuestas del ciclo de vida:

```bash
openclaw claws add ./incident-triage.claw.json --dry-run --json
```

El plan muestra el agente y el espacio de trabajo derivados, cada acción propuesta,
los requisitos previos, los bloqueos, las distintas ampliaciones de capacidades y un resumen
`planIntegrity`. Los registros de capacidades muestran el efecto exacto del paquete, MCP, trabajo programado, entorno aislado,
herramienta o Heartbeat. Revise el plan antes de crear el agente:

```bash
openclaw claws add ./incident-triage.claw.json \
  --yes \
  --plan-integrity <SHA256_FROM_DRY_RUN>
```

`--yes` por sí solo no es suficiente. OpenClaw vuelve a generar el plan y rechaza el consentimiento
cuando la fuente, el destino o la configuración activa han cambiado después de la vista previa. Utilice
`--agent-id` o `--workspace` tanto durante la vista previa como durante la aplicación cuando los valores
predeterminados del paquete entren en conflicto con el estado local. Para perfiles desechables y validación paralela,
proporcione un `--workspace` explícito; `OPENCLAW_STATE_DIR` reubica el estado de ejecución, pero
no cambia la ubicación predeterminada del espacio de trabajo.

Añadir un Claw crea el nuevo agente y la configuración del espacio de trabajo, escribe los archivos
declarados del espacio de trabajo, instala o reutiliza los artefactos de Skills y plugins declarados y
registra la procedencia del paquete, MCP y Cron. Los archivos existentes no se sobrescriben
y los reintentos fallan de forma segura cuando el contenido gestionado ha divergido.

## Inspeccionar el estado instalado

```bash
openclaw claws status
openclaw claws status incident-triage --json
openclaw doctor
```

`status` compara el agente instalado y su procedencia registrada del espacio de trabajo, paquete, MCP
y Cron con el estado actual. Informa de instalaciones incompletas, recursos
ausentes y divergencias sin cambiar el estado local. `openclaw doctor` añade
diagnósticos específicos de Claw para registros de propiedad incompletos, archivos gestionados
no seguros y trabajos de Cron que no pueden corroborarse con el inventario activo del Gateway.

La procedencia de Claw distingue dos relaciones:

- **Gestionado:** el Claw introdujo y gestiona actualmente el recurso. Es un
  candidato para la limpieza cuando no presenta cambios y no queda ningún propietario en conflicto.
- **Referenciado:** el recurso existía de forma independiente o se comparte. La eliminación
  libera la referencia de este Claw y conserva el recurso de forma predeterminada.

Esto no es un recuento de referencias. Los comandos ordinarios de plugins, Skills y agentes mantienen
su comportamiento existente; los Claws añaden además procedencia y operaciones protegidas del ciclo de vida.

## Actualizar un Claw instalado

De forma predeterminada, la actualización utiliza la fuente registrada cuando se añadió el Claw. Utilice
`--from` cuando dicha fuente se haya movido o al probar otro directorio de paquete:

```bash
openclaw claws update incident-triage --dry-run --json
openclaw claws update incident-triage \
  --from ./incident-triage-next \
  --dry-run --json
```

El plan compara la procedencia actual y el estado activo con el manifiesto de destino.
Informa de cambios en el agente, el espacio de trabajo, el paquete, MCP, Cron y la propiedad,
incluidas las ampliaciones de capacidades y los bloqueos. Las ampliaciones de capacidades tienen
registros independientes legibles por máquina y líneas `!` con los efectos exactos censurados en
la salida para personas. Se incluyen la integridad resuelta del paquete, la identidad de instalación y cualquier
advertencia de confianza. Eliminar una declaración de paquete libera la relación de este Claw
sin desinstalar el artefacto durante la actualización. La confirmación exacta final de
`planIntegrity` vincula tanto ese conjunto revelado como los cambios de contenido
ordinarios. Los hosts pueden utilizar los mismos registros para un cuadro de diálogo independiente o una
revisión agregada de varios agentes. Aplique el plan exacto revisado con consentimiento
explícito:

```bash
openclaw claws update incident-triage \
  --yes \
  --plan-integrity <SHA256_FROM_DRY_RUN>
```

OpenClaw vuelve a generar el plan y realiza una comparación e intercambio del estado gestionado antes de cada
mutación. Las declaraciones de paquetes eliminadas liberan relaciones de dependencia sin
desinstalar artefactos. Los cambios de Cron vuelven a leer la definición activa del programador y
se detienen si el operador ha introducido divergencias. Los instaladores de paquetes, los escritores de configuración de fuentes y el programador del Gateway
no forman una sola transacción. Si no puede demostrarse la compensación después de una mutación
externa, OpenClaw informa del código de error `update_partial` con
`status: partial` estructurado, conserva la procedencia incierta
y se detiene. Inspeccione `claws status`, el recurso afectado y `openclaw doctor`;
después, vuelva a obtener una vista previa antes de reintentar o eliminar cualquier elemento.

## Eliminar un Claw instalado

Obtenga una vista previa de la eliminación antes de seleccionar la limpieza:

```bash
openclaw claws remove incident-triage --dry-run --json
openclaw claws remove incident-triage \
  --yes \
  --plan-integrity <SHA256_FROM_DRY_RUN>
```

De forma predeterminada, se elimina el estado gestionado apto y se libera el estado referenciado.
Los archivos modificados y los recursos con otro propietario actual se conservan o
se bloquean. Las opciones de limpieza forman parte del resumen del plan; `--yes` nunca
las amplía. Los plugins instalados globalmente se conservan mientras se libera la referencia de este Claw;
utilice por separado el ciclo de vida ordinario de plugins cuando pretenda
desinstalar un plugin para todo el proceso.

Para eliminar referencias sin cambios introducidas por el Claw que no tengan ningún otro propietario
actual, incluya `--remove-unused` tanto en la vista previa como en la aplicación. Para seleccionar recursos
referenciados exactos, repita `--remove-referenced`:

```bash
openclaw claws remove incident-triage \
  --dry-run \
  --remove-referenced 'plugin:@acme/audit-plugin@2.0.0'
```

Utilice `--force-referenced` únicamente después de revisar los dependientes mostrados,
los propietarios independientes y el origen preexistente. Permite la limpieza seleccionada pese
a esos conflictos; no omite el consentimiento de integridad del plan.

## Exportar un agente instalado

La exportación crea un directorio de paquete nuevo y falla si el destino ya existe o
el estado administrado presenta desviaciones:

```bash
openclaw claws export incident-triage --out ./incident-triage-export --json
```

El resultado contiene `package.json`, `CLAW.md` canónico y archivos auxiliares
del espacio de trabajo administrado. Es un paquete Claw portátil, no una copia de seguridad de toda la instancia: se excluyen los
agentes, las credenciales, las sesiones y el estado local sin propietario que no estén relacionados.

## Referencia de comandos

| Comando                             | Propósito                                                        |
| ----------------------------------- | ---------------------------------------------------------------- |
| `claws inspect <source>`            | Validar un directorio de paquete o un manifiesto agrupado.        |
| `claws add <source>`                | Previsualizar o crear un agente nuevo y un espacio de trabajo.    |
| `claws status [claw-or-agent]`      | Informar sobre el estado instalado, la propiedad y las desviaciones. |
| `claws update <claw-or-agent>`      | Previsualizar o aplicar cambios desde la fuente seleccionada.     |
| `claws remove <claw-or-agent>`      | Previsualizar o eliminar el agente y los recursos aptos.          |
| `claws export <agent> --out <path>` | Crear un paquete portátil a partir de un agente instalado.        |

Use `--json` para obtener resultados experimentales legibles por máquina.

## Véase también

- [Agentes](/es/cli/agents)
- [Skills](/es/tools/skills)
- [Plugins](/es/tools/plugin)
- [Tareas de Cron](/es/automation/cron-jobs)
- [Configuración de MCP](/es/gateway/configuration-reference#mcp)
