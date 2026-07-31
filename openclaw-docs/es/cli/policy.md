---
read_when:
    - Se desea comprobar la configuración de OpenClaw con un archivo policy.jsonc creado previamente.
    - Se quieren hallazgos de políticas en el lint de doctor
    - Necesita un hash de certificación de políticas como evidencia de auditoría
summary: Referencia de la CLI para las comprobaciones de conformidad de `openclaw policy`
title: Política
x-i18n:
    generated_at: "2026-07-26T04:36:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 63e4faeab8dd6535e3d517439d3f58cdc167b6b7fade808a6482742ec9b5acf1
    source_path: cli/policy.md
    workflow: 16
---

# `openclaw policy`

`openclaw policy` lo proporciona el Plugin Policy incluido. Es una capa de
conformidad empresarial sobre la configuración existente de OpenClaw, no un segundo
sistema de configuración. Los requisitos se definen en `policy.jsonc`; OpenClaw observa el
espacio de trabajo activo como evidencia; Policy informa de las desviaciones mediante `doctor --lint`. Policy
no impone llamadas a herramientas ni reescribe el comportamiento del entorno de ejecución en el momento de la solicitud,
y tampoco certifica almacenes de credenciales por agente como `auth-profiles.json`.

Policy comprueba los canales configurados, los servidores MCP, los proveedores de modelos, la
postura de SSRF de la red, el acceso de entrada/canales, la exposición del Gateway y la postura de comandos de los nodos,
las sondas de enrutamiento de mensajes definidas,
el acceso al espacio de trabajo de los agentes, la postura del entorno aislado, la postura de tratamiento de datos, la postura de los
proveedores de secretos/perfiles de autenticación y los metadatos de las herramientas gobernadas (`TOOLS.md`). Se utiliza
cuando un espacio de trabajo necesita una declaración duradera y verificable, como «Telegram no debe
estar habilitado» o «las herramientas gobernadas deben declarar metadatos de riesgo y propietario». Si
solo se necesita comportamiento local sin certificación ni detección de desviaciones, basta con la
configuración normal.

## Inicio rápido

```bash
openclaw plugins enable policy
```

El Plugin permanece habilitado incluso cuando falta `policy.jsonc`, para que doctor pueda
informar de la ausencia del artefacto en lugar de omitir silenciosamente las comprobaciones.

`policy.jsonc` se define manualmente; no se genera a partir de la configuración actual. Cada
sección de nivel superior es un espacio de nombres de reglas: una comprobación solo se ejecuta cuando contiene
una regla concreta (las secciones o claves no compatibles generan
`policy/policy-jsonc-invalid` en lugar de ignorarse silenciosamente). Ejemplo
mínimo que abarca todas las secciones compatibles:

```jsonc
{
  "channels": {
    "denyRules": [
      {
        "id": "no-telegram",
        "when": { "provider": "telegram" },
        "reason": "Telegram no está aprobado para este espacio de trabajo.",
      },
    ],
  },
  "mcp": {
    "servers": {
      "allow": ["docs"],
      "deny": ["untrusted"],
    },
  },
  "models": {
    "providers": {
      "allow": ["openai", "anthropic"],
      "deny": ["openrouter"],
    },
  },
  "network": {
    "privateNetwork": {
      "allow": false,
    },
  },
  "routing": {
    "requireBindings": true,
    "requireConfiguredChannels": true,
    "probes": [
      {
        "id": "family-dm",
        "route": {
          "channel": "imessage",
          "peer": { "kind": "direct", "id": "+15555550123" },
        },
        "expect": {
          "agentId": "family",
          "matchedBy": ["binding.peer"],
        },
      },
    ],
  },
  "ingress": {
    "session": {
      "requireDmScope": "per-channel-peer",
    },
    "channels": {
      "allowDmPolicies": ["pairing", "allowlist", "disabled"],
      "denyOpenGroups": true,
      "requireMentionInGroups": true,
    },
  },
  "gateway": {
    "exposure": {
      "allowNonLoopbackBind": false,
      "allowTailscaleFunnel": false,
    },
    "auth": {
      "requireAuth": true,
      "requireExplicitRateLimit": true,
    },
    "controlUi": {
      "allowInsecure": false,
    },
    "remote": {
      "allow": false,
    },
    "http": {
      "denyEndpoints": ["chatCompletions", "responses"],
      "requireUrlAllowlists": true,
    },
    "nodes": {
      "denyCommands": ["system.run"],
    },
  },
  "agents": {
    "workspace": {
      "allowedAccess": ["none", "ro"],
      "denyTools": ["exec", "process", "write", "edit", "apply_patch"],
    },
  },
  "dataHandling": {
    "sensitiveLogging": {
      "requireRedaction": true,
    },
    "telemetry": {
      "denyContentCapture": true,
    },
    "retention": {
      "requireSessionMaintenance": true,
    },
    "memory": {
      "denySessionTranscriptIndexing": true,
    },
  },
  "secrets": {
    "requireManagedProviders": true,
    "denySources": ["exec"],
    "allowInsecureProviders": false,
  },
  "auth": {
    "profiles": {
      "requireMetadata": ["provider", "mode"],
      "allowModes": ["api_key", "token"],
    },
  },
  "execApprovals": {
    "requireFile": true,
    "defaults": { "allowSecurity": ["deny"] },
    "agents": {
      "allowSecurity": ["deny", "allowlist"],
      "allowAutoAllowSkills": false,
      "allowlist": { "expected": ["deploy", "status"] },
    },
  },
  "tools": {
    "requireMetadata": ["risk", "sensitivity", "owner"],
    "profiles": {
      "allow": ["messaging", "minimal"],
    },
    "fs": {
      "requireWorkspaceOnly": true,
    },
    "exec": {
      "allowSecurity": ["deny", "allowlist"],
      "requireAsk": ["always"],
      "allowHosts": ["sandbox"],
    },
    "elevated": {
      "allow": false,
    },
    "denyTools": ["group:runtime", "group:fs"],
  },
}
```

Notas transversales que no resultan evidentes en las tablas de reglas siguientes:

- Omitir `gateway.bind` al denegar vinculaciones que no sean de bucle local significa que se acepta
  el valor predeterminado del entorno de ejecución; se debe establecer `gateway.bind: "loopback"` para una conformidad estricta.
- Para un agente de solo lectura, se debe establecer `mode` del entorno aislado en `all` o `non-main` en los
  valores predeterminados o el agente correspondientes, y `workspaceAccess` en `none` o `ro`. Un modo de
  entorno aislado ausente o `off` no satisface una política de solo lectura.
- `agents.workspace.denyTools` acepta `exec`, `process`, `write`, `edit`,
  `apply_patch`. Los grupos de denegación de herramientas de configuración `group:fs` (mutación de archivos) y
  `group:runtime` (shell/proceso) satisfacen la postura equivalente.
- Las comprobaciones de aprobaciones de ejecución leen el artefacto activo `exec-approvals.json` solo cuando
  existe una regla `execApprovals`; un artefacto ausente o no válido constituye
  evidencia no observable, no una aprobación sintética.
- La evidencia de secretos y perfiles de autenticación registra únicamente la postura del proveedor/origen y
  los metadatos de SecretRef, nunca valores sin procesar. Policy no lee ni certifica
  almacenes de credenciales por agente como `auth-profiles.json`.
- La evidencia de tratamiento de datos solo representa la postura en el nivel de configuración (modo de ocultación,
  opción de captura de telemetría, modo de mantenimiento de sesiones y configuración de indexación
  de transcripciones). No inspecciona registros, exportaciones de telemetría, transcripciones ni
  archivos de memoria, y un resultado limpio no demuestra que no contengan datos personales ni
  secretos.
- Las sondas de enrutamiento reutilizan el solucionador de vinculaciones del entorno de ejecución de OpenClaw. La evidencia de enrutamiento
  solo registra el identificador de la sonda, el agente resuelto, el tipo de coincidencia y los metadatos de vinculación
  ocultados. Nunca registra identificadores de interlocutores, cuentas, servidores, equipos ni roles.
  Añadir una sección de enrutamiento cambia deliberadamente los hashes de la política y la certificación;
  las políticas sin enrutamiento conservan la estructura de evidencia existente.

### Referencia de reglas de Policy

Todas las reglas siguientes son opcionales; una comprobación solo se ejecuta cuando la regla está presente. El
estado observado corresponde a la configuración o los metadatos del espacio de trabajo existentes de OpenClaw.

#### Superposiciones con ámbito

Se debe utilizar `scopes.<scopeName>` cuando agentes o canales concretos necesiten una política
más estricta que la base de nivel superior. El nombre del ámbito es solo una etiqueta; la coincidencia utiliza el
selector incluido en el ámbito. Las superposiciones son aditivas: la regla global sigue ejecutándose
y la regla con ámbito puede añadir su propio hallazgo sobre la misma evidencia.

| Selector     | Secciones compatibles                                                             | Cuándo utilizarlo                                          |
| ------------ | ------------------------------------------------------------------------------ | ------------------------------------------------- |
| `agentIds`   | `tools`, `agents.workspace`, `sandbox`, `dataHandling.memory`, `execApprovals` | Uno o más agentes del entorno de ejecución necesitan reglas más estrictas.   |
| `channelIds` | `ingress.channels`                                                             | Uno o más canales necesitan reglas de entrada más estrictas. |

Si no existe una entrada `agentIds` en `agents.entries.*`, OpenClaw evalúa
la regla con ámbito respecto de la postura global/predeterminada heredada para ese identificador de
agente del entorno de ejecución en lugar de omitirla.

```jsonc
{
  "tools": {
    "exec": {
      "allowHosts": ["sandbox", "node"],
    },
  },
  "sandbox": {
    "requireMode": ["all", "non-main"],
  },
  "scopes": {
    "release-workspace": {
      "agentIds": ["release-agent", "review-agent"],
      "agents": {
        "workspace": {
          "allowedAccess": ["none", "ro"],
        },
      },
    },
    "release-lockdown": {
      "agentIds": ["release-agent"],
      "tools": {
        "exec": {
          "allowHosts": ["sandbox"],
          "allowSecurity": ["deny", "allowlist"],
          "requireAsk": ["always"],
        },
        "denyTools": ["exec", "process", "write", "edit", "apply_patch"],
      },
      "sandbox": {
        "requireMode": ["all"],
        "allowBackends": ["docker"],
      },
      "dataHandling": {
        "memory": {
          "denySessionTranscriptIndexing": true,
        },
      },
    },
    "shell-sandbox": {
      "agentIds": ["shell-agent"],
      "sandbox": {
        "allowBackends": ["openshell"],
        "containers": {
          "requireReadOnlyMounts": false,
        },
      },
    },
    "telegram-ingress": {
      "channelIds": ["telegram"],
      "ingress": {
        "channels": {
          "allowDmPolicies": ["pairing"],
          "denyOpenGroups": true,
          "requireMentionInGroups": true,
        },
      },
    },
  },
}
```

El mismo agente puede aparecer en varios ámbitos si cada uno gobierna un
campo diferente, como en el ejemplo anterior. Un campo con ámbito repetido para el mismo agente debe ser igual de
restrictivo o más; las declaraciones duplicadas menos restrictivas se rechazan (las listas de permitidos son
subconjuntos, las listas de denegados son superconjuntos y los valores booleanos obligatorios son fijos).

Las reglas de postura de contenedores (`sandbox.containers.*`) solo se comprueban respecto de
la evidencia que puede exponer el backend del entorno aislado del agente coincidente. Si un backend no puede
observar una regla habilitada para él, Policy informa de
`policy/sandbox-container-posture-unobservable` en lugar de aprobarla; las reglas de
contenedores deben limitarse a los grupos de agentes que utilicen un backend capaz de exponerlas.

`ingress.session.requireDmScope` de nivel superior sigue siendo global; `session.dmScope`
no es evidencia atribuible a un canal, por lo que no se le puede aplicar un ámbito mediante `channelIds`.

Todos los ámbitos presentes en `policy.jsonc` deben ser válidos y aplicables.

#### Canales

| Campo de Policy                         | Estado observado                          | Cuándo utilizarlo                                                     |
| ------------------------------------ | --------------------------------------- | ------------------------------------------------------------ |
| `channels.denyRules[].when.provider` | Proveedor y estado habilitado de `channels.*` | Denegar canales configurados de un proveedor como `telegram`. |
| `channels.denyRules[].reason`        | Mensaje del hallazgo y contexto de la sugerencia de reparación | Explicar por qué se deniega el proveedor.                          |

#### Servidores MCP

| Campo de Policy        | Estado observado      | Cuándo utilizarlo                                                   |
| ------------------- | ------------------- | ---------------------------------------------------------- |
| `mcp.servers.allow` | Identificadores de `mcp.servers.*` | Exigir que todos los servidores MCP configurados estén en una lista de permitidos. |
| `mcp.servers.deny`  | Identificadores de `mcp.servers.*` | Denegar identificadores específicos de servidores MCP configurados.                   |

#### Proveedores de modelos

| Campo de Policy             | Estado observado                                   | Cuándo utilizarlo                                                                        |
| ------------------------ | ------------------------------------------------ | ------------------------------------------------------------------------------- |
| `models.providers.allow` | Identificadores de `models.providers.*` y referencias de modelos seleccionadas | Exigir que los proveedores configurados y las referencias de modelos seleccionadas utilicen proveedores aprobados. |
| `models.providers.deny`  | Identificadores de `models.providers.*` y referencias de modelos seleccionadas | Denegar proveedores configurados y referencias de modelos seleccionadas por identificador de proveedor.               |

#### Red

| Campo de política              | Estado observado                    | Uso                                                               |
| ------------------------------ | ----------------------------------- | ----------------------------------------------------------------- |
| `network.privateNetwork.allow` | Vías de escape de SSRF hacia redes privadas | Establecer en `false` para exigir que el acceso a redes privadas permanezca deshabilitado. |

#### Enrutamiento de mensajes

| Campo de política                   | Estado observado                                             | Uso                                                                    |
| ----------------------------------- | ------------------------------------------------------------ | ---------------------------------------------------------------------- |
| `routing.requireBindings`           | Enlaces de rutas de canales, excluidos los enlaces de ACP    | Exigir al menos un enlace de enrutamiento de mensajes.                 |
| `routing.requireConfiguredChannels` | Identificadores de canal de enlaces e identificadores `channels.*` configurados | Detectar identificadores de canal de enlaces obsoletos o mal escritos. |
| `routing.probes[].route`            | El resolutor público de rutas de OpenClaw                    | Describir una ruta entrante representativa sin enviar un mensaje.      |
| `routing.probes[].expect.agentId`   | Identificador de agente resuelto                             | Exigir que la ruta llegue al agente revisado.                          |
| `routing.probes[].expect.matchedBy` | Tipo de coincidencia del resolutor                           | Exigir la especificidad revisada del enlace de par, cuenta, canal u otro tipo. |

Los identificadores de sondeo deben ser únicos. Una ruta admite `channel`, `accountId` opcional,
`peer`, `parentPeer`, `guildId`, `teamId` y `memberRoleIds`. Los tipos de par son
`direct`, `group` y `channel`. `matchedBy` puede contener uno o más tipos de
coincidencia en tiempo de ejecución, incluidos `binding.peer`, `binding.account`, `binding.channel`
o `default`.

Las comprobaciones de enrutamiento son únicamente comprobaciones de conformidad. No modifican el inicio,
la entrega de mensajes, la precedencia de los enlaces ni el comportamiento de reserva. Los hallazgos requieren
la revisión del operador, ya que modificar automáticamente un enlace podría redirigir
mensajes privados.

#### Acceso de entrada y a canales

| Campo de política                        | Estado observado                                                     | Uso                                                                |
| ---------------------------------------- | -------------------------------------------------------------------- | ------------------------------------------------------------------ |
| `ingress.session.requireDmScope`          | `session.dmScope`                                                    | Exigir un ámbito revisado de aislamiento de mensajes directos.     |
| `ingress.channels.allowDmPolicies`        | `channels.*.dmPolicy` y campos heredados de política de MD del canal    | Permitir únicamente políticas revisadas de mensajes directos del canal. |
| `ingress.channels.denyOpenGroups`         | Política de entrada de canales, cuentas y grupos                     | Denegar la entrada abierta de grupos para canales y cuentas configurados. |
| `ingress.channels.requireMentionInGroups` | Configuración de puertas de menciones de canales, cuentas, grupos, servidores y elementos anidados | Exigir puertas de menciones cuando la entrada de grupos esté abierta o condicionada a menciones. |

#### Gateway

| Campo de política                      | Estado observado                                  | Uso                                                                                             |
| -------------------------------------- | ------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| `gateway.exposure.allowNonLoopbackBind` | `gateway.bind`                                | Establecer en `false` para exigir que Gateway se vincule a la interfaz de bucle local. |
| `gateway.exposure.allowTailscaleFunnel` | Postura de Gateway para serve/funnel de Tailscale | Establecer en `false` para denegar la exposición mediante Tailscale Funnel.           |
| `gateway.auth.requireAuth`              | `gateway.auth.mode`                               | Establecer en `true` para rechazar la autenticación deshabilitada de Gateway.        |
| `gateway.auth.requireExplicitRateLimit` | `gateway.auth.rateLimit`                          | Establecer en `true` para exigir una configuración explícita del límite de frecuencia de autenticación. |
| `gateway.controlUi.allowInsecure`       | Opciones de autenticación, dispositivo u origen inseguros de la interfaz de control | Establecer en `false` para denegar las opciones de exposición insegura de la interfaz de control. |
| `gateway.remote.allow`                  | Modo/configuración remotos de Gateway             | Establecer en `false` para denegar el modo remoto de Gateway.                          |
| `gateway.http.denyEndpoints`            | Endpoints de la API HTTP de Gateway               | Denegar identificadores de endpoint como `chatCompletions` o `responses`.                |
| `gateway.http.requireUrlAllowlists`     | Entradas de obtención de URL mediante HTTP de Gateway | Establecer en `true` para exigir listas de permitidos de URL en las entradas de obtención de URL. |
| `gateway.nodes.denyCommands`            | `gateway.nodes.commands.deny`                     | Exigir que los identificadores exactos de comandos de Node, como `system.run`, estén denegados en la configuración de OpenClaw. |

`gateway.nodes.denyCommands` es una regla exacta y sensible a mayúsculas y minúsculas de superconjunto de denegación de políticas.
Se utiliza cuando la política debe demostrar que los comandos privilegiados de Node están explícitamente
denegados por la configuración de OpenClaw. Una implementación que permita intencionadamente un comando privilegiado
de Node debe actualizar `policy.jsonc` después de la revisión, en lugar de depender únicamente de
`gateway.nodes.commands.allow`.

#### Espacio de trabajo del agente

| Campo de política                | Estado observado                                                                           | Uso                                                                                              |
| -------------------------------- | ------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------ |
| `agents.workspace.allowedAccess` | `agents.defaults.sandbox.workspaceAccess` y `agents.entries.*.sandbox.workspaceAccess` | Permitir únicamente valores de acceso al espacio de trabajo del entorno aislado como `none` o `ro`. |
| `agents.workspace.denyTools`     | Configuración global y por agente de denegación de herramientas                             | Exigir que las herramientas de mutación (`exec`, `process`, `write`, `edit`, `apply_patch`) estén denegadas. |

#### Postura del entorno aislado

| Campo de política                                    | Estado observado                                           | Uso                                                                  |
| ---------------------------------------------------- | ---------------------------------------------------------- | -------------------------------------------------------------------- |
| `sandbox.requireMode`                                 | `agents.defaults.sandbox.mode` y modo por agente                       | Permitir únicamente modos de entorno aislado revisados como `all` o `non-main`. |
| `sandbox.allowBackends`                               | `agents.defaults.sandbox.backend` y backend por agente                    | Permitir únicamente backends de entorno aislado revisados como `docker`. |
| `sandbox.containers.denyHostNetwork`                  | Modo de red del entorno aislado/navegador basado en contenedores | Denegar el modo de red del host.                                     |
| `sandbox.containers.denyContainerNamespaceJoin`       | Modo de red del entorno aislado/navegador basado en contenedores | Denegar la incorporación al espacio de nombres de red de otro contenedor. |
| `sandbox.containers.requireReadOnlyMounts`            | Modo de montaje del entorno aislado/navegador basado en contenedores | Exigir que los montajes sean de solo lectura.                        |
| `sandbox.containers.denyContainerRuntimeSocketMounts` | Destinos de montaje del entorno aislado/navegador basado en contenedores | Denegar los montajes de sockets del entorno de ejecución de contenedores. |
| `sandbox.containers.denyUnconfinedProfiles`           | Postura del perfil de seguridad del contenedor             | Denegar perfiles de seguridad de contenedores sin restricciones.     |
| `sandbox.browser.requireCdpSourceRange`               | Intervalo de origen de CDP del navegador del entorno aislado | Exigir que la exposición de CDP del navegador declare un intervalo de origen. |

La política trata la ausencia de `sandbox.mode` como su valor predeterminado implícito `off`, por lo que
`sandbox.requireMode` indica que un entorno aislado nuevo o sin configurar está fuera de una
lista de permitidos como `["all"]`.

#### Tratamiento de datos

| Campo de política                                  | Estado observado                                                                                     | Uso                                                                    |
| -------------------------------------------------- | ---------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| `dataHandling.sensitiveLogging.requireRedaction`    | `logging.redactSensitive`                                                                          | Establecer en `true` para rechazar `logging.redactSensitive: "off"`.     |
| `dataHandling.telemetry.denyContentCapture`         | `diagnostics.otel.captureContent`                                                                  | Establecer en `true` para rechazar la captura de contenido de telemetría. |
| `dataHandling.retention.requireSessionMaintenance`  | `session.maintenance.mode`                                                                         | Establecer en `true` para exigir el modo efectivo de mantenimiento de sesiones `enforce`. |
| `dataHandling.memory.denySessionTranscriptIndexing` | `memory.qmd.sessions.enabled`, `memory.search.experimental.sessionMemory` y anulaciones por agente | Establecer en `true` para rechazar la indexación en memoria de transcripciones de sesiones. |

#### Secretos

| Campo de política                 | Estado observado                                               | Uso                                                                         |
| --------------------------------- | -------------------------------------------------------------- | --------------------------------------------------------------------------- |
| `secrets.requireManagedProviders` | SecretRefs de configuración y declaraciones `secrets.providers.*` | Establecer en `true` para exigir que las SecretRefs apunten a proveedores declarados. |
| `secrets.denySources`             | Orígenes de proveedores de secretos y orígenes de SecretRef    | Denegar orígenes como `exec`, `file` u otro nombre de origen configurado. |
| `secrets.allowInsecureProviders`  | Indicadores de postura insegura de proveedores de secretos     | Establecer en `false` para rechazar proveedores que habiliten una postura insegura. |

#### Aprobaciones de ejecución

Las comprobaciones de aprobaciones de ejecución leen el artefacto `exec-approvals.json` del entorno de ejecución:
`~/.openclaw/exec-approvals.json` de forma predeterminada, o
`$OPENCLAW_STATE_DIR/exec-approvals.json` cuando se establece `OPENCLAW_STATE_DIR`.
Las reglas de postura incluidas en `execApprovals.defaults.*` o `execApprovals.agents.*`
requieren evidencia legible del artefacto; un artefacto ausente o no válido se notifica como
evidencia no observable en lugar de considerarse aprobado mediante el mejor esfuerzo. Una vez que es legible, los
campos omitidos heredan los valores predeterminados del entorno de ejecución: la ausencia de `defaults.security` equivale a `full`, y
la ausencia de seguridad del agente hereda ese valor predeterminado. La evidencia incluye `defaults`,
`agents.*`, `agents.*.allowlist[].pattern`, `argPattern` opcional, la postura efectiva
de `autoAllowSkills` y el origen de la entrada; nunca la ruta o el token del socket,
`commandText`, `lastUsedCommand`, las rutas resueltas ni las marcas de tiempo.

| Campo de política                          | Estado observado                                                                       | Usar cuando                                                                                          |
| ------------------------------------------ | --------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `execApprovals.requireFile`                         | Ruta de `exec-approvals.json` del entorno de ejecución activo                              | Establézcalo en `true` para exigir que el artefacto de aprobaciones exista y se analice. |
| `execApprovals.defaults.allowSecurity`                         | `defaults.security`, con valor predeterminado `full`                         | Permitir solo modos de seguridad de aprobación predeterminados que estén aprobados.                  |
| `execApprovals.agents.allowSecurity`                         | `agents.*.security`, con herencia de los valores predeterminados                         | Permitir solo modos efectivos de seguridad de aprobación por agente que estén aprobados.            |
| `execApprovals.agents.allowAutoAllowSkills`                         | `defaults.autoAllowSkills` y `agents.*.autoAllowSkills`, con herencia de los valores del entorno de ejecución | Establézcalo en `false` para exigir listas de permitidos manuales estrictas sin aprobación implícita de la CLI de las Skills. |
| `execApprovals.agents.allowlist.expected`                         | Patrón agregado `agents.*.allowlist[]` y entradas opcionales de argPattern                  | Exigir que la lista de permitidos de aprobaciones coincida con el conjunto de patrones revisado.     |

Ejemplo: exigir el artefacto de aprobaciones, denegar los valores predeterminados permisivos y permitir
solo una postura revisada de aprobación de ejecución para los agentes seleccionados.

```jsonc
{
  "execApprovals": {
    "requireFile": true,
    "defaults": {
      // Modos de seguridad: "deny", "allowlist" o "full".
      // Este valor predeterminado solo permite la postura de denegación restringida.
      "allowSecurity": ["deny"],
    },
  },
  "scopes": {
    "restricted-shell": {
      "agentIds": ["family-agent", "groups-agent"],
      "execApprovals": {
        "agents": {
          // Los agentes seleccionados pueden usar la postura de lista de permitidos revisada, pero no "full".
          "allowSecurity": ["allowlist"],
          // false significa que las CLI de las Skills deben aparecer en la lista de permitidos revisada en lugar de
          // recibir aprobación implícita mediante autoAllowSkills.
          "allowAutoAllowSkills": false,
          "allowlist": {
            "expected": [
              // Entrada simple: patrón exacto del ejecutable revisado sin argPattern.
              "travel-hub",
              // Entrada restringida: patrón más expresión regular de argumentos revisada.
              { "pattern": "calendar-cli", "argPattern": "^sync\\b" },
              "/bin/date",
            ],
          },
        },
      },
    },
  },
}
```

#### Perfiles de autenticación

| Campo de política              | Estado observado                               | Usar cuando                                                                                                          |
| ------------------------------ | ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| `auth.profiles.requireMetadata`             | Metadatos de proveedor y modo `auth.profiles.*` | Exigir claves de metadatos como `provider` y `mode` en los perfiles de autenticación de la configuración. |
| `auth.profiles.allowModes`             | `auth.profiles.*.mode`                             | Permitir solo modos compatibles de perfiles de autenticación como `api_key`, `aws-sdk`, `oauth` o `token`. |

#### Metadatos de herramientas

| Campo de política              | Estado observado                              | Usar cuando                                                                                                   |
| ------------------------------ | --------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| `tools.requireMetadata`             | Declaraciones `TOOLS.md` gobernadas   | Exigir que las herramientas gobernadas declaren claves de metadatos como `risk`, `sensitivity` o `owner`. |

#### Postura de las herramientas

| Campo de política              | Estado observado                                               | Usar cuando                                                                                                           |
| ------------------------------ | -------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| `tools.profiles.allow`             | `tools.profile` y `agents.entries.*.tools.profile`                        | Permitir solo identificadores de perfiles de herramientas como `minimal`, `messaging` o `coding`. |
| `tools.fs.requireWorkspaceOnly`             | `tools.fs.workspaceOnly` y anulaciones `tools.fs` por agente | Establézcalo en `true` para exigir que la postura de las herramientas del sistema de archivos se limite al espacio de trabajo. |
| `tools.exec.allowSecurity`             | `tools.exec.security` y seguridad de ejecución por agente         | Permitir solo modos de seguridad de ejecución como `deny` o `allowlist`.                           |
| `tools.exec.requireAsk`             | `tools.exec.ask` y modo de solicitud de ejecución por agente | Exigir una postura de aprobación como `always`.                                                             |
| `tools.exec.allowHosts`             | `tools.exec.host` y enrutamiento del host de ejecución por agente | Permitir solo modos de enrutamiento del host de ejecución como `sandbox`.                                |
| `tools.elevated.allow`             | `tools.elevated.enabled` y postura elevada por agente                | Establézcalo en `false` para exigir que el modo elevado de las herramientas permanezca desactivado.       |
| `tools.alsoAllow.expected`             | `tools.alsoAllow` y `tools.alsoAllow` por agente             | Exigir entradas `alsoAllow` exactas e informar de concesiones adicionales de herramientas faltantes o inesperadas. |
| `tools.denyTools`             | `tools.deny` y `agents.entries.*.tools.deny`                        | Exigir que las listas configuradas de denegación de herramientas incluyan identificadores o grupos de herramientas como `group:runtime` y `group:fs`. |

## Ejecutar comprobaciones

Ejecute comprobaciones exclusivamente de políticas durante la creación:

```bash
openclaw policy check
openclaw policy check --json
openclaw policy check --severity-min error
```

`policy check` ejecuta únicamente el conjunto de comprobaciones de políticas y emite pruebas, hallazgos
y hashes de atestación. Los mismos hallazgos también aparecen en
`openclaw doctor --lint` cuando el Plugin de políticas está habilitado.

Compare un archivo de políticas del operador con una línea base creada:

```bash
openclaw policy compare --baseline official.policy.jsonc
openclaw policy compare --baseline official.policy.jsonc --policy policy.jsonc --json
```

`policy compare` comprueba la sintaxis de un archivo de políticas con respecto a la sintaxis de otro archivo de políticas; no
inspecciona el estado del entorno de ejecución, las pruebas, las credenciales ni los secretos. Utiliza los mismos
metadatos de reglas que rigen las superposiciones con ámbito: las listas de permitidos deben mantenerse iguales o
ser más restrictivas, las listas de denegación deben mantenerse iguales o ser más amplias, los booleanos obligatorios deben conservar
su valor, las cadenas ordenadas solo pueden avanzar hacia el extremo más estricto del
orden configurado y las listas exactas deben coincidir. La línea base puede ser una
política creada por la organización; la política comprobada puede añadir valores más estrictos o
reglas adicionales. Una regla comprobada de nivel superior puede satisfacer una regla de línea base con ámbito cuando
es igual o más restrictiva. Los nombres de los ámbitos no tienen que coincidir entre
archivos; la comparación se determina por el selector (`agentIds`/`channelIds`) y el campo.
En las pruebas de enrutamiento, cada identificador de prueba de la línea base debe conservar la misma ruta
y el mismo agente esperado. Una política comprobada puede añadir pruebas o restringir `matchedBy`, pero
eliminar una prueba, cambiar su ruta o agente, o ampliar los tipos de coincidencia que acepta
es menos restrictivo.

Comparación sin hallazgos (`--json`):

```json
{
  "ok": true,
  "baselinePath": "official.policy.jsonc",
  "policyPath": "policy.jsonc",
  "rulesChecked": 3,
  "findings": []
}
```

La salida sin hallazgos de `policy check --json` incluye hashes estables que un operador o
supervisor puede registrar:

```json
{
  "ok": true,
  "attestation": {
    "policy": {
      "path": "policy.jsonc",
      "hash": "sha256:..."
    },
    "workspace": {
      "scope": "policy",
      "hash": "sha256:..."
    },
    "findingsHash": "sha256:...",
    "attestationHash": "sha256:..."
  },
  "checksRun": 5,
  "checksSkipped": 0,
  "findings": []
}
```

## Configurar la política

La configuración de políticas se encuentra en `plugins.entries.policy.config`.

```jsonc
{
  "plugins": {
    "entries": {
      "policy": {
        "enabled": true,
        "config": {
          "enabled": true,
          "path": "policy.jsonc",
          "workspaceRepairs": false,
          "expectedHash": "sha256:...",
          "expectedAttestationHash": "sha256:...",
        },
      },
    },
  },
}
```

| Ajuste                      | Finalidad                                                                        |
| --------------------------- | -------------------------------------------------------------------------------- |
| `enabled`          | Habilitar las comprobaciones de políticas incluso antes de que exista `policy.jsonc`. |
| `workspaceRepairs`          | Permitir que `doctor --fix` edite los ajustes del espacio de trabajo administrados por políticas. |
| `expectedHash`          | Bloqueo opcional mediante hash para el artefacto de políticas aprobado.           |
| `expectedAttestationHash`          | Bloqueo opcional mediante hash para la última comprobación de políticas aceptada sin hallazgos. |
| `path`          | Ubicación del artefacto de políticas relativa al espacio de trabajo.              |

Establezca `plugins.entries.policy.config.enabled` en `false` para deshabilitar las comprobaciones de
políticas de un espacio de trabajo y mantener instalado el Plugin.

## Aceptar el estado de la política

Ejemplo de salida JSON:

```json
{
  "ok": true,
  "attestation": {
    "checkedAt": "2026-05-10T20:00:00.000Z",
    "policy": {
      "path": "policy.jsonc",
      "hash": "sha256:..."
    },
    "workspace": {
      "scope": "policy",
      "hash": "sha256:..."
    },
    "findingsHash": "sha256:...",
    "attestationHash": "sha256:..."
  },
  "evidence": {
    "channels": [
      {
        "id": "telegram",
        "provider": "telegram",
        "source": "oc://openclaw.config/channels/telegram",
        "enabled": false
      }
    ],
    "mcpServers": [
      {
        "id": "docs",
        "transport": "stdio",
        "source": "oc://openclaw.config/mcp/servers/docs",
        "command": "npx"
      }
    ],
    "modelProviders": [
      {
        "id": "openai",
        "source": "oc://openclaw.config/models/providers/openai"
      }
    ],
    "modelRefs": [
      {
        "ref": "openai/gpt-5.6-sol",
        "provider": "openai",
        "model": "gpt-5.6-sol",
        "source": "oc://openclaw.config/agents/defaults/model"
      }
    ],
    "network": [
      {
        "id": "browser-private-network",
        "source": "oc://openclaw.config/browser/ssrfPolicy/dangerouslyAllowPrivateNetwork",
        "value": false
      }
    ],
    "gatewayExposure": [
      {
        "id": "gateway-bind",
        "kind": "bind",
        "source": "oc://openclaw.config/gateway/bind",
        "value": "loopback",
        "nonLoopback": false,
        "explicit": true
      }
    ],
    "agentWorkspace": [
      {
        "id": "agents-defaults-workspace-access",
        "kind": "workspaceAccess",
        "source": "oc://openclaw.config/agents/defaults/sandbox/workspaceAccess",
        "scope": "defaults",
        "value": "ro",
        "sandboxMode": "all",
        "sandboxModeSource": "oc://openclaw.config/agents/defaults/sandbox/mode",
        "sandboxEnabled": true,
        "explicit": true
      },
      {
        "id": "agents-defaults-tool-exec",
        "kind": "toolDeny",
        "source": "oc://openclaw.config/tools/deny",
        "scope": "defaults",
        "tool": "exec",
        "denied": true,
        "explicit": true
      }
    ],
    "secrets": [
      {
        "id": "vault",
        "kind": "provider",
        "source": "oc://openclaw.config/secrets/providers/vault",
        "providerSource": "env"
      },
      {
        "id": "oc://openclaw.config/models/providers/openai/apiKey",
        "kind": "input",
        "source": "oc://openclaw.config/models/providers/openai/apiKey",
        "provenance": "secretRef",
        "refSource": "env",
        "refProvider": "vault"
      }
    ],
    "authProfiles": [
      {
        "id": "github",
        "source": "oc://openclaw.config/auth/profiles/github",
        "validMetadata": true,
        "provider": "github",
        "mode": "token"
      }
    ],
    "tools": [
      {
        "id": "deploy",
        "source": "oc://TOOLS.md/tools/deploy",
        "line": 12,
        "risk": "critical",
        "sensitivity": "restricted",
        "capabilities": ["IRREVERSIBLE_EXTERNAL"]
      }
    ]
  },
  "checksRun": 30,
  "checksSkipped": 0,
  "findings": []
}
```

`attestation.policy.hash` identifica el artefacto de reglas creado. `evidence`
registra el estado observado de OpenClaw utilizado por las comprobaciones, y
`workspace.hash` identifica esa carga útil de evidencia. `findingsHash` identifica
el conjunto exacto de hallazgos. `checkedAt` registra cuándo se ejecutó la comprobación.
`attestationHash` identifica la declaración estable (hash de la política, hash de la evidencia,
hash de los hallazgos y estado limpio/con cambios) y excluye deliberadamente `checkedAt`,
por lo que el mismo estado de la política siempre produce el mismo hash de atestación. En conjunto,
estos cuatro valores forman la tupla de auditoría de una comprobación de política.

Si un Gateway o supervisor utiliza la política para bloquear, aprobar o anotar una
acción en tiempo de ejecución, debe registrar el hash de atestación de la última
comprobación limpia. `checkedAt` permanece en la salida JSON para los registros de auditoría, pero no forma parte del
hash estable.

Ciclo de vida para aceptar el estado de la política:

1. Cree o revise `policy.jsonc`.
2. Ejecute `openclaw policy check --json`.
3. Si está limpio, registre `attestation.policy.hash` como `expectedHash`.
4. Registre `attestation.attestationHash` como `expectedAttestationHash`.
5. Vuelva a ejecutar `openclaw doctor --lint` en la Pipeline de CI o en las puertas de lanzamiento.

Si las reglas de la política cambian intencionadamente, actualice ambos hashes aceptados a partir de una
comprobación limpia. Si solo cambia la configuración del espacio de trabajo (la política permanece igual),
normalmente solo cambia `expectedAttestationHash`.

Habilitar o actualizar las reglas de `agents.workspace` añade evidencia de `agentWorkspace`
al hash del espacio de trabajo y al hash de atestación; revise la nueva evidencia y
actualice los hashes de atestación aceptados después de habilitarlas. Habilitar o actualizar
las reglas de postura de las herramientas añade evidencia de `toolPosture` de la misma manera.

`openclaw policy watch` vuelve a ejecutar la comprobación e informa cuando la evidencia actual ya
no coincide con `expectedAttestationHash`:

```bash
openclaw policy watch --json
```

Utilice `--once` en la CI o en scripts que necesiten una única evaluación de desviaciones. Sin
`--once`, sondea cada dos segundos de forma predeterminada; utilice `--interval-ms` para cambiar
el intervalo.

## Hallazgos

| Id. de comprobación                                     | Hallazgo                                                                          |
| -------------------------------------------------------- | --------------------------------------------------------------------------------- |
| `policy/policy-jsonc-missing`                            | La política está habilitada, pero falta `policy.jsonc`.                           |
| `policy/policy-jsonc-invalid`                            | La política no se puede analizar o contiene entradas de reglas con formato incorrecto. |
| `policy/policy-hash-mismatch`                            | La política no coincide con el valor configurado de `expectedHash`.                |
| `policy/attestation-hash-mismatch`                       | La evidencia actual de la política ya no coincide con la certificación aceptada.  |
| `policy/policy-conformance-invalid`                      | Un archivo de política de referencia o comprobado tiene una sintaxis de comparación no válida. |
| `policy/policy-conformance-missing`                      | A un archivo de política comprobado le falta una regla exigida por el archivo de política de referencia. |
| `policy/policy-conformance-weaker`                       | Un archivo de política comprobado tiene un valor menos restrictivo que el archivo de política de referencia. |
| `policy/channels-denied-provider`                        | Un canal habilitado coincide con una regla de denegación de canales.               |
| `policy/mcp-denied-server`                               | La política deniega un servidor MCP configurado.                                   |
| `policy/mcp-unapproved-server`                           | Un servidor MCP configurado no está en la lista de permitidos.                     |
| `policy/models-denied-provider`                          | Un proveedor de modelos o una referencia de modelo configurados usan un proveedor denegado. |
| `policy/models-unapproved-provider`                      | Un proveedor de modelos o una referencia de modelo configurados no están en la lista de permitidos. |
| `policy/network-private-access-enabled`                  | Se ha habilitado una vía de escape de SSRF para redes privadas cuando la política la deniega. |
| `policy/routing-bindings-required`                       | La política exige una vinculación de ruta de canal, pero no hay ninguna configurada. |
| `policy/routing-binding-channel-unconfigured`            | Una vinculación de ruta nombra un canal que no está presente en `channels.*`.     |
| `policy/routing-agent-mismatch`                          | Una ruta definida se resuelve a un agente diferente.                               |
| `policy/routing-match-kind-mismatch`                     | Una ruta definida coincide con una especificidad de vinculación inesperada.        |
| `policy/ingress-dm-policy-unapproved`                    | Una política de mensajes directos de un canal no está en la lista de permitidos de la política. |
| `policy/ingress-dm-scope-unapproved`                     | `session.dmScope` no coincide con el ámbito de aislamiento de mensajes directos exigido por la política. |
| `policy/ingress-open-groups-denied`                      | Una política de grupos de un canal es `open` mientras la política deniega la entrada abierta de grupos. |
| `policy/ingress-group-mention-required`                  | Una entrada de canal o grupo deshabilita los requisitos de mención cuando la política los exige. |
| `policy/gateway-non-loopback-bind`                       | La configuración de vinculación del Gateway permite la exposición fuera de la interfaz de bucle invertido cuando la política la deniega. |
| `policy/gateway-auth-disabled`                           | La autenticación del Gateway está deshabilitada cuando la política la exige.       |
| `policy/gateway-rate-limit-missing`                      | La configuración del límite de frecuencia de autenticación del Gateway no es explícita cuando la política lo exige. |
| `policy/gateway-control-ui-insecure`                     | Están habilitadas las opciones de exposición no segura de la interfaz de control del Gateway. |
| `policy/gateway-tailscale-funnel`                        | La exposición mediante Tailscale Funnel del Gateway está habilitada cuando la política la deniega. |
| `policy/gateway-remote-enabled`                          | El modo remoto del Gateway está activo cuando la política lo deniega.              |
| `policy/gateway-http-endpoint-enabled`                   | Un endpoint de la API HTTP del Gateway está habilitado aunque la política lo deniega. |
| `policy/gateway-http-url-fetch-unrestricted`             | La entrada de obtención de URL por HTTP del Gateway carece de una lista de URL permitidas obligatoria. |
| `policy/gateway-node-command-denied`                     | Un comando de Node denegado por la política no está denegado por la configuración de OpenClaw. |
| `policy/agents-workspace-access-denied`                  | El modo de entorno aislado del agente o el acceso al espacio de trabajo no están en la lista de permitidos de la política. |
| `policy/agents-tool-not-denied`                          | La configuración de un agente o la predeterminada no deniega una herramienta que la política exige denegar. |
| `policy/tools-profile-unapproved`                        | Un perfil de herramientas global o por agente configurado no está en la lista de permitidos. |
| `policy/tools-fs-workspace-only-required`                | Las herramientas del sistema de archivos no están configuradas para limitar las rutas únicamente al espacio de trabajo. |
| `policy/tools-exec-security-unapproved`                  | El modo de seguridad de ejecución no está en la lista de permitidos de la política. |
| `policy/tools-exec-ask-unapproved`                       | El modo de solicitud de ejecución no está en la lista de permitidos de la política. |
| `policy/tools-exec-host-unapproved`                      | El enrutamiento del host de ejecución no está en la lista de permitidos de la política. |
| `policy/tools-elevated-enabled`                          | El modo de herramientas con privilegios elevados está habilitado cuando la política lo deniega. |
| `policy/tools-also-allow-missing`                        | A una lista `alsoAllow` configurada le falta una entrada exigida por la política. |
| `policy/tools-also-allow-unexpected`                     | Una lista `alsoAllow` configurada incluye una entrada que la política no contempla. |
| `policy/tools-required-deny-missing`                     | Una lista de denegación de herramientas global o por agente no incluye una herramienta que debe denegarse. |
| `policy/sandbox-mode-unapproved`                         | El modo de entorno aislado no está en la lista de permitidos de la política.       |
| `policy/sandbox-backend-unapproved`                      | El backend del entorno aislado no está en la lista de permitidos de la política.   |
| `policy/sandbox-container-posture-unobservable`          | Una regla de configuración de contenedores está habilitada para un backend que no puede observarla. |
| `policy/sandbox-container-host-network-denied`           | Un entorno aislado o navegador basado en contenedores usa el modo de red del host. |
| `policy/sandbox-container-namespace-join-denied`         | Un entorno aislado o navegador basado en contenedores se une al espacio de nombres de otro contenedor. |
| `policy/sandbox-container-mount-mode-required`           | Un montaje de un entorno aislado o navegador basado en contenedores no es de solo lectura. |
| `policy/sandbox-container-runtime-socket-mount`          | Un montaje de un entorno aislado o navegador basado en contenedores expone el socket del entorno de ejecución de contenedores. |
| `policy/sandbox-container-unconfined-profile`            | El perfil del entorno aislado de contenedores no está confinado cuando la política lo deniega. |
| `policy/sandbox-browser-cdp-source-range-missing`        | Falta el intervalo de origen de CDP del navegador del entorno aislado cuando la política exige uno. |
| `policy/data-handling-redaction-disabled`                | La ocultación de datos sensibles en los registros está deshabilitada cuando la política la exige. |
| `policy/data-handling-telemetry-content-capture`         | La captura de contenido de telemetría está habilitada cuando la política la deniega. |
| `policy/data-handling-session-retention-not-enforced`    | El mantenimiento de retención de sesiones no se aplica cuando la política lo exige. |
| `policy/data-handling-session-transcript-memory-enabled` | La indexación en memoria de las transcripciones de sesiones está habilitada cuando la política la deniega. |
| `policy/secrets-unmanaged-provider`                      | Un SecretRef de la configuración hace referencia a un proveedor no declarado en `secrets.providers`. |
| `policy/secrets-denied-provider-source`                  | Un proveedor de secretos o SecretRef de la configuración usa una fuente denegada por la política. |
| `policy/secrets-insecure-provider`                       | Un proveedor de secretos habilita una configuración no segura cuando la política la deniega. |
| `policy/auth-profile-invalid-metadata`                   | A un perfil de autenticación de la configuración le faltan metadatos válidos de proveedor o modo. |
| `policy/auth-profile-unapproved-mode`                    | El modo de un perfil de autenticación de la configuración no está en la lista de permitidos de la política. |
| `policy/exec-approvals-missing`                          | La política exige `exec-approvals.json`, pero falta el artefacto.                  |
| `policy/exec-approvals-invalid`                          | No se puede analizar el artefacto configurado de aprobaciones de ejecución.        |
| `policy/exec-approvals-default-security-unapproved`      | Los valores predeterminados de aprobación de ejecución usan un modo de seguridad que no está en la lista de permitidos de la política. |
| `policy/exec-approvals-agent-security-unapproved`        | El modo efectivo de seguridad de aprobación de ejecución de un agente no está en la lista de permitidos. |
| `policy/exec-approvals-auto-allow-skills-enabled`        | Un agente de aprobación de ejecución permite automáticamente de forma implícita las CLI de Skills cuando la política lo deniega. |
| `policy/exec-approvals-allowlist-missing`                | A la lista de permitidos de aprobaciones le falta un patrón exigido por la política. |
| `policy/exec-approvals-allowlist-unexpected`             | La lista de permitidos de aprobaciones incluye un patrón que la política no contempla. |
| `policy/tools-missing-risk-level`                        | A una declaración de herramienta sometida a gobernanza le faltan metadatos de riesgo. |
| `policy/tools-unknown-risk-level`                        | Una declaración de herramienta sometida a gobernanza usa un valor de riesgo desconocido. |
| `policy/tools-missing-sensitivity-token`                 | A una declaración de herramienta sometida a gobernanza le faltan metadatos de sensibilidad. |
| `policy/tools-missing-owner`                             | A una declaración de herramienta sometida a gobernanza le faltan metadatos de propietario. |
| `policy/tools-unknown-sensitivity-token`                 | Una declaración de herramienta sometida a gobernanza usa un valor de sensibilidad desconocido. |

Un hallazgo puede incluir tanto `target` (el elemento observado del espacio de trabajo que
no cumple los requisitos) como `requirement` (la regla definida que hizo que se considerara un hallazgo).
Actualmente, ambos son cadenas de dirección `oc://`, pero los nombres de los campos describen la función
en la política y no el formato de la dirección.

Ejemplos de hallazgos:

```json
{
  "checkId": "policy/channels-denied-provider",
  "severity": "error",
  "message": "El canal 'telegram' usa el proveedor denegado 'telegram'.",
  "source": "policy",
  "path": "configuración de openclaw",
  "ocPath": "oc://openclaw.config/channels/telegram",
  "target": "oc://openclaw.config/channels/telegram",
  "requirement": "oc://policy.jsonc/channels/denyRules/#0",
  "fixHint": "Telegram no está aprobado para este espacio de trabajo."
}
```

```json
{
  "checkId": "policy/tools-missing-risk-level",
  "severity": "error",
  "message": "La herramienta 'deploy' de TOOLS.md no tiene una clasificación de riesgo explícita.",
  "source": "policy",
  "path": "TOOLS.md",
  "line": 12,
  "ocPath": "oc://TOOLS.md/tools/deploy",
  "target": "oc://TOOLS.md/tools/deploy",
  "requirement": "oc://policy.jsonc/tools/requireMetadata"
}
```

```json
{
  "checkId": "policy/mcp-unapproved-server",
  "severity": "error",
  "message": "El servidor MCP 'remote' no está en la lista de permitidos de la política.",
  "source": "policy",
  "path": "configuración de openclaw",
  "ocPath": "oc://openclaw.config/mcp/servers/remote",
  "target": "oc://openclaw.config/mcp/servers/remote",
  "requirement": "oc://policy.jsonc/mcp/servers/allow"
}
```

```json
{
  "checkId": "policy/models-unapproved-provider",
  "severity": "error",
  "message": "La referencia de modelo 'anthropic/claude-sonnet-4.7' usa el proveedor no aprobado 'anthropic'.",
  "source": "policy",
  "path": "configuración de openclaw",
  "ocPath": "oc://openclaw.config/agents/defaults/model/fallbacks/#0",
  "target": "oc://openclaw.config/agents/defaults/model/fallbacks/#0",
  "requirement": "oc://policy.jsonc/models/providers/allow"
}
```

```json
{
  "checkId": "policy/network-private-access-enabled",
  "severity": "error",
  "message": "La opción de red 'browser-private-network' permite el acceso a la red privada.",
  "source": "policy",
  "path": "configuración de openclaw",
  "ocPath": "oc://openclaw.config/browser/ssrfPolicy/dangerouslyAllowPrivateNetwork",
  "target": "oc://openclaw.config/browser/ssrfPolicy/dangerouslyAllowPrivateNetwork",
  "requirement": "oc://policy.jsonc/network/privateNetwork/allow"
}
```

```json
{
  "checkId": "policy/gateway-non-loopback-bind",
  "severity": "error",
  "message": "La configuración de enlace del Gateway 'gateway-bind' permite la exposición fuera de la interfaz de bucle invertido.",
  "source": "policy",
  "path": "openclaw config",
  "ocPath": "oc://openclaw.config/gateway/bind",
  "target": "oc://openclaw.config/gateway/bind",
  "requirement": "oc://policy.jsonc/gateway/exposure/allowNonLoopbackBind"
}
```

```json
{
  "checkId": "policy/gateway-node-command-denied",
  "severity": "error",
  "message": "El comando de Node del Gateway 'system.run' está denegado por la política, pero no por la configuración de OpenClaw.",
  "source": "policy",
  "path": "openclaw config",
  "ocPath": "oc://openclaw.config/gateway/nodes/commands/deny",
  "target": "oc://openclaw.config/gateway/nodes/commands/deny",
  "requirement": "oc://policy.jsonc/gateway/nodes/denyCommands",
  "fixHint": "Añada 'system.run' a gateway.nodes.commands.deny o actualice la política después de revisarla."
}
```

```json
{
  "checkId": "policy/agents-workspace-access-denied",
  "severity": "error",
  "message": "La política no permite el valor 'rw' de workspaceAccess del entorno aislado de agents.defaults.",
  "source": "policy",
  "path": "openclaw config",
  "ocPath": "oc://openclaw.config/agents/defaults/sandbox/workspaceAccess",
  "target": "oc://openclaw.config/agents/defaults/sandbox/workspaceAccess",
  "requirement": "oc://policy.jsonc/agents/workspace/allowedAccess"
}
```

## Reparación

`doctor --lint` y `policy check` son de solo lectura.

`doctor --fix` solo modifica la configuración del espacio de trabajo gestionada por políticas cuando
`workspaceRepairs` está habilitado explícitamente; de lo contrario, las comprobaciones indican lo que
repararían y no modifican la configuración.

En esta versión, la reparación puede deshabilitar los canales denegados por `channels.denyRules` y
aplicar las reparaciones automáticas de restricción que se enumeran a continuación. Habilite `workspaceRepairs`
solo después de revisar el archivo de políticas, porque una regla válida puede modificar
la configuración del espacio de trabajo:

- establecer `tools.elevated.enabled=false` cuando una política global prohíbe las herramientas con privilegios elevados
- añadir los identificadores de herramientas de denegación obligatoria que falten a `tools.deny` o
  `agents.entries.*.tools.deny` cuando la política exija denegar esas herramientas
- establecer en `false` los conmutadores `gateway.controlUi.*` que no sean seguros
- establecer `gateway.mode=local` cuando la política deniegue el modo de Gateway remoto
- establecer las rutas `gateway.http.endpoints.*.enabled` notificadas en `false` cuando la política
  deniegue los endpoints de la API HTTP del Gateway
- establecer las rutas `groupPolicy` notificadas de entrada de canales en `allowlist` cuando la política
  deniegue la entrada abierta de grupos
- establecer las rutas `requireMention` notificadas de entrada de canales en `true` cuando la política
  exija menciones de grupo
- establecer `logging.redactSensitive=tools` cuando la política exija la censura de
  datos confidenciales en los registros
- establecer `diagnostics.otel.captureContent=false`, o
  `diagnostics.otel.captureContent.enabled=false` para la configuración de captura de telemetría
  en formato de objeto, cuando la política deniegue la captura del contenido de telemetría

Las reparaciones de herramientas con privilegios elevados y ámbito limitado son solo de detección. Las reparaciones de tratamiento de datos con ámbito limitado
también se omiten cuando el hallazgo notifica una configuración compartida de registros o telemetría,
porque modificar la configuración compartida afectaría a más elementos que el objetivo de la política
con ámbito limitado.

Las reparaciones de denegación obligatoria con ámbito limitado se omiten cuando el hallazgo notifica
la configuración raíz heredada `tools.deny`, porque añadir la herramienta obligatoria a la configuración raíz afectaría
a más elementos que el objetivo de la política con ámbito limitado. Las reparaciones de denegación obligatoria locales del agente pueden actualizar
la ruta `agents.entries.*.tools.deny` notificada.

Las reparaciones de entrada de canales con ámbito limitado se omiten cuando el hallazgo notifica
la configuración heredada `channels.defaults.*`, porque modificar el valor predeterminado compartido del canal afectaría
a más elementos que el objetivo de la política con ámbito limitado. Los hallazgos de listas de permitidos para la obtención de URL mediante HTTP del Gateway
siguen requiriendo intervención manual porque la reparación automática no puede elegir los valores correctos
de la lista de permitidos de URL de endpoints.

Los hallazgos relativos al enlace del Gateway y a los comandos de Node siguen requiriendo revisión. Cuando
`policy/gateway-non-loopback-bind` o `policy/gateway-node-command-denied`
pueden asignarse a una ruta de configuración, `doctor --fix` notifica el cambio propuesto de
`gateway.bind` o `gateway.nodes.commands.deny` como orientación de vista previa
omitida. No aplica el cambio y el hallazgo no se considera
reparado hasta que un operador revise y actualice la configuración o la política.

```jsonc
{
  "plugins": {
    "entries": {
      "policy": {
        "config": {
          "workspaceRepairs": true,
        },
      },
    },
  },
}
```

## Códigos de salida

| Comando          | `0`                                                    | `1`                                                                 | `2`                          |
| ---------------- | ------------------------------------------------------ | ------------------------------------------------------------------- | ---------------------------- |
| `policy check`   | No hay hallazgos en el umbral.                          | Uno o más hallazgos alcanzaron el umbral.                             | Error de argumentos o de ejecución. |
| `policy compare` | El archivo de políticas es al menos tan estricto como la referencia. | El archivo de políticas no es válido, no existe o es menos estricto que las reglas de referencia. | Error de argumentos o de ejecución. |
| `policy watch`   | No hay hallazgos y el hash aceptado está actualizado.              | Existen hallazgos o la atestación aceptada está obsoleta.                    | Error de argumentos o de ejecución. |

## Temas relacionados

- [Modo de lint de Doctor](/es/cli/doctor#lint-mode)
- [CLI de rutas](/es/cli/path)
