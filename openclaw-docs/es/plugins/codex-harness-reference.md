---
read_when:
    - Necesitas todos los campos de configuración del entorno de Codex
    - Está cambiando el comportamiento de transporte, autenticación, descubrimiento o tiempo de espera de app-server
    - Está depurando el inicio del entorno de Codex, la detección de modelos o el aislamiento del entorno
summary: Referencia de configuración, autenticación, detección y servidor de aplicaciones para el arnés de Codex
title: Referencia del arnés de Codex
x-i18n:
    generated_at: "2026-07-26T05:47:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 149f065f5bef18d0f491c97facc4b5991afc3f7e1077abdc7a4b49f506eac3e0
    source_path: plugins/codex-harness-reference.md
    workflow: 16
---

Esta referencia abarca la configuración detallada del Plugin oficial `codex`.
Para las decisiones de configuración y enrutamiento, comience por
[arnés de Codex](/es/plugins/codex-harness).

## Superficie de configuración del Plugin

Todos los ajustes del arnés de Codex se encuentran en `plugins.entries.codex.config`.

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          discovery: {
            enabled: true,
            timeoutMs: 2500,
          },
          appServer: {
            mode: "guardian",
          },
        },
      },
    },
  },
}
```

Campos de nivel superior:

| Campo                      | Valor predeterminado                  | Significado                                                                                                                                        |
| -------------------------- | ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `discovery`                | habilitado                  | Ajustes de detección de modelos para `model/list` del servidor de aplicaciones de Codex.                                                                                    |
| `appServer`                | servidor de aplicaciones stdio administrado | Ajustes de transporte, comando, autenticación, aprobación, entorno aislado y tiempo de espera. El arnés normal usa de forma predeterminada el estado con ámbito de agente.                        |
| `codexDynamicToolsLoading` | `"searchable"`           | Use `"direct"` para colocar las herramientas dinámicas de OpenClaw directamente en el contexto inicial de herramientas de Codex.                                                       |
| `codexDynamicToolsExclude` | `[]`                     | Nombres adicionales de herramientas dinámicas de OpenClaw que se deben omitir en los turnos del servidor de aplicaciones de Codex.                                                                    |
| `codexPlugins`             | deshabilitado                 | Compatibilidad nativa con plugins/aplicaciones de Codex, incluido el acceso opcional a aplicaciones de cuentas conectadas. Consulte [Plugins nativos de Codex](/es/plugins/codex-native-plugins). |
| `computerUse`              | deshabilitado                 | Configuración de Codex Computer Use. Consulte [Codex Computer Use](/es/plugins/codex-computer-use).                                                               |
| `sessionCatalog`           | habilitado                  | Detección nativa de sesiones de Codex para la barra lateral. Establezca `enabled: false` para deshabilitar la detección sin deshabilitar el proveedor ni el arnés.           |
| `supervision`              | deshabilitado                 | Política de transcripción y control de escritura de sesiones nativas orientada al agente. Consulte [Supervisión de Codex](/es/plugins/codex-supervision).                          |

## Supervisión

De forma predeterminada, la detección de sesiones nativas muestra las sesiones de Codex no archivadas del equipo del Gateway
y de los nodos emparejados que hayan optado por participar. Deshabilite únicamente ese catálogo con:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          sessionCatalog: {
            enabled: false,
          },
        },
      },
    },
  },
}
```

`supervision` controla por separado las herramientas orientadas al agente:

| Campo                 | Valor predeterminado                 | Significado                                                                                                                                                                                                                                   |
| --------------------- | ----------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `enabled`             | `false`                 | Habilita las herramientas de supervisión de Codex orientadas al agente. Esto no controla el catálogo de sesiones autenticadas del operador.                                                                                                                            |
| `endpoints`           | punto de conexión local integrado | Destinos de puntos de conexión avanzados y de compatibilidad para el agente de supervisión de Codex conservado y las herramientas MCP independientes. El catálogo humano y el flujo de ramas ignoran estos destinos y usan el servidor de aplicaciones de supervisión resuelto desde `appServer`.       |
| `allowRawTranscripts` | `false`                 | Con la supervisión habilitada, permite que el agente autónomo o MCP independiente lea transcripciones y campos de listas derivados de transcripciones. Las lecturas de `codex_threads` solo de metadatos siguen disponibles. No controla la continuación autenticada de la interfaz de control.     |
| `allowWriteControls`  | `false`                 | Con la supervisión habilitada, permite las mutaciones autónomas de bifurcación, cambio de nombre, archivado y desarchivado de `codex_threads`, además de las operaciones independientes de MCP para enviar, dirigir e interrumpir. No omite otras comprobaciones de vinculación, host, estado o confirmación. |

Las entradas de puntos de conexión aceptan estos campos:

| Campo          | Se aplica a    | Significado                                                               |
| -------------- | ------------- | --------------------------------------------------------------------- |
| `id`           | todos           | Identificador estable del punto de conexión.                                                   |
| `label`        | todos           | Etiqueta de visualización opcional.                                               |
| `transport`    | todos           | `"stdio-proxy"` o `"websocket"`.                                     |
| `command`      | `stdio-proxy` | Comando opcional del servidor de aplicaciones.                                          |
| `args`         | `stdio-proxy` | Argumentos opcionales del comando.                                           |
| `cwd`          | `stdio-proxy` | Directorio de trabajo opcional del proceso secundario.                             |
| `url`          | `websocket`   | URL obligatoria de WebSocket o de un socket local compatible.                     |
| `authTokenEnv` | `websocket`   | Variable de entorno opcional cuyo valor autentica el punto de conexión. |

La página **Sesiones de Codex** usa el servidor de aplicaciones de supervisión del Plugin y muestra
solo las sesiones no archivadas. Sin ajustes de conexión explícitos de `appServer`,
esa conexión se administra mediante stdio en el directorio de inicio del usuario. Las filas locales almacenadas o inactivas pueden crear
un Chat bloqueado al modelo con un historial acotado del usuario y el asistente hasta el último
turno de origen terminal persistido. Su vinculación privada mantiene en esa
conexión la bifurcación de la instantánea, la rama canónica de origen `appServer`, la inyección del historial y los turnos posteriores.
El primer inicio canónico usa el par devuelto por la bifurcación. Las reanudaciones
posteriores omiten las sustituciones de modelo y proveedor de OpenClaw para que Codex restaure el
par persistido del hilo canónico; un cambio nativo independiente puede actualizar ese
par, pero el modelo externo y la cadena de respaldo nunca lo reemplazan. Las filas almacenadas e inactivas
pueden archivarse después de confirmar que no hay otro ejecutor, salvo que otra
vinculación activa de OpenClaw sea propietaria del destino exacto o de uno de sus descendientes
generados no archivados. OpenClaw sigue la paginación de descendientes de Codex y aplica un cierre seguro
ante errores de enumeración, ciclos o agotamiento del límite de seguridad. La confirmación sigue
abarcando los clientes nativos desconocidos y la condición de carrera entre el estado y el archivado. Un Chat supervisado
bloqueado al modelo no puede eliminarse mientras proteja la vinculación nativa.
Los orígenes activos no pueden crear una rama ni archivarse, pero sí puede abrirse un Chat
supervisado existente. Todas las filas de nodos emparejados permanecen en modo de solo lectura; el transporte del nodo
aún no proporciona el ciclo de vida de transmisión necesario para el arnés.

Solo `appServer.homeScope: "user"` cambia qué directorio de inicio de Codex usa un proceso
de arnés administrado; no publica el catálogo de la flota. Habilitar la supervisión no
cambia el valor predeterminado del arnés. En su lugar, la conexión de supervisión independiente
usa de forma predeterminada stdio administrado en el directorio de inicio del usuario cuando no existen ajustes de conexión
explícitos de `appServer`. Los ajustes explícitos se respetan para esa conexión.
Las vinculaciones supervisadas pendientes y confirmadas conservan esa conexión en cada turno;
la supervisión deshabilitada o una desviación de la conexión o del ciclo de vida aplican un cierre seguro en lugar de
recurrir al arnés del directorio de inicio del agente. La conexión predeterminada comparte las sesiones almacenadas
con los clientes nativos de Codex, no el estado de actividad local de sus procesos.

Los ajustes heredados de `plugins.entries.codex-supervisor` se han retirado. Ejecute
`openclaw doctor --fix` para migrar a este bloque la entrada anterior, las definiciones de puntos de conexión, las marcas
de política y las referencias de permitidos/denegados del Plugin. Los valores canónicos explícitos de
`codex.config.supervision` prevalecen en caso de conflicto.

## Transporte del servidor de aplicaciones

Para los turnos normales del arnés, OpenClaw inicia el binario administrado de Codex incluido
con el Plugin oficial (actualmente `@openai/codex` `0.145.0`):

```bash
codex app-server --listen stdio://
```

Esto mantiene la versión del servidor de aplicaciones vinculada al Plugin oficial `codex` en lugar de
a cualquier CLI de Codex independiente que esté instalada localmente. Establezca
`appServer.command` solo cuando se quiera usar intencionadamente otro ejecutable.
Los turnos administrados normales con el directorio de inicio aislado predeterminado del agente prefieren este
paquete fijado incluso cuando hay instalado un paquete de aplicación de escritorio de macOS. Cuando
[Computer Use](/es/plugins/codex-computer-use) está habilitado, o cuando `homeScope` es
`"user"` y puede cargar el estado nativo de Computer Use, el inicio administrado prefiere en su lugar
el binario de la aplicación de escritorio que posee los permisos de macOS necesarios. La misma
regla de prioridad para el escritorio se aplica cuando la configuración efectiva de Codex del directorio de inicio aislado de un agente
habilita Computer Use nativo. Si no hay instalado ningún paquete de aplicación de escritorio, OpenClaw
recurre al binario del paquete fijado.

La transferencia del ejecutable y el aislamiento de la configuración nativa coordinan los clientes dentro de un
mismo proceso del Gateway en ejecución. Reinicie el Gateway después de que otro proceso cambie la
configuración nativa del Plugin de Codex.

La supervisión resuelve una conexión independiente. Sin ajustes de conexión
explícitos de `appServer`, usa stdio administrado con `homeScope: "user"`;
el arnés normal permanece en stdio administrado con `homeScope: "agent"`. Ambos
flujos respetan los ajustes de conexión explícitos. Establezca `homeScope: "user"`
explícitamente cuando el arnés normal deba compartir `$CODEX_HOME` (o `~/.codex`)
con clientes nativos. Una vinculación supervisada privada usa la conexión de supervisión
independientemente del valor predeterminado del arnés normal. Los procesos independientes del servidor de aplicaciones
conservan estados activos y de aprobación separados.

Para pruebas que no sean de producción con un servidor de aplicaciones que ya se esté ejecutando, está disponible el transporte
WebSocket:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            transport: "websocket",
            url: "ws://gateway-host:39175",
            authToken: "${CODEX_APP_SERVER_TOKEN}",
            requestTimeoutMs: 60000,
          },
        },
      },
    },
  },
}
```

Codex clasifica el transporte WebSocket como experimental y no compatible. Para
cargas de trabajo de producción, se recomienda stdio administrado o el socket de control Unix local.

Campos de `appServer`:

| Campo                                         | Valor predeterminado                                    | Significado                                                                                                                                                                                                                                                                                                                                                                                         |
| --------------------------------------------- | ------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `transport`                                   | `"stdio"`                                              | `"stdio"` inicia Codex; `"unix"` explícito se conecta al socket de control local; `"websocket"` se conecta a `url`.                                                                                                                                                                                                                                                                                |
| `homeScope`                                   | `"agent"`                                              | `"agent"` aísla el estado ordinario del arnés por agente de OpenClaw. `"user"` es una adhesión explícita que comparte el `$CODEX_HOME` o `~/.codex` nativo, utiliza la autenticación nativa y habilita la gestión de hilos solo para el propietario. El ámbito de usuario admite stdio local o transporte Unix. Para la conexión de supervisión independiente, un valor no establecido se resuelve como `"user"` para stdio o Unix y como `"agent"` para WebSocket.     |
| `command`                                     | binario de Codex gestionado                             | Ejecutable para el transporte stdio. Déjelo sin establecer para utilizar el binario gestionado.                                                                                                                                                                                                                                                                                                                          |
| `args`                                        | `["app-server", "--listen", "stdio://"]`               | Argumentos para el transporte stdio.                                                                                                                                                                                                                                                                                                                                                                  |
| `url`                                         | no establecido                                         | URL del App Server WebSocket o URL `unix://`. Una ruta Unix explícita vacía selecciona el socket de control canónico del directorio personal del usuario.                                                                                                                                                                                                                                                                          |
| `authToken`                                   | no establecido                                         | Token de portador para el transporte WebSocket. Acepta una cadena literal o SecretInput, como `${CODEX_APP_SERVER_TOKEN}`.                                                                                                                                                                                                                                                                              |
| `headers`                                     | `{}`                                                   | Encabezados WebSocket adicionales. Los valores de los encabezados aceptan cadenas literales o valores SecretInput, por ejemplo, `x-codex-client-session-token: "${CODEX_CLIENT_SESSION_TOKEN}"`.                                                                                                                                                                                                                               |
| `clearEnv`                                    | `[]`                                                   | Nombres de variables de entorno adicionales que se eliminan del proceso app-server stdio iniciado después de que OpenClaw construye su entorno heredado.                                                                                                                                                                                                                                                             |
| `remoteWorkspaceRoot`                         | no establecido                                         | Raíz remota del espacio de trabajo del app-server de Codex. Cuando se establece, OpenClaw infiere la raíz del espacio de trabajo local a partir del espacio de trabajo resuelto de OpenClaw, conserva el sufijo del cwd actual bajo esta raíz remota y envía a Codex únicamente el cwd final del app-server. Si el cwd está fuera de la raíz resuelta del espacio de trabajo de OpenClaw, OpenClaw aplica un cierre seguro en lugar de enviar una ruta local del Gateway al app-server remoto. |
| `loopDetectionPreToolUseRelay`                | `true`                                                 | Instala el subproceso `PreToolUse` de Codex que se utiliza únicamente para la detección de bucles de OpenClaw y su marcador explícito de ausencia de política. Establezca `false` para reducir la expansión de procesos por herramienta. Los hooks de Plugin previos a la herramienta y la política de herramientas de confianza siguen instalando su retransmisor requerido.                                                                                                                                         |
| `requestTimeoutMs`                            | `60000`                                                | Tiempo de espera para las llamadas al plano de control del app-server.                                                                                                                                                                                                                                                                                                                                                     |
| `turnCompletionIdleTimeoutMs`                 | `60000`                                                | Intervalo de inactividad después de que Codex acepta un turno o tras una solicitud del app-server limitada al turno mientras OpenClaw espera `turn/completed`.                                                                                                                                                                                                                                                                    |
| `turnAssistantCompletionIdleTimeoutMs`        | `10000`                                                | Intervalo de inactividad después de que un elemento final o no comentarial del asistente, o la finalización sin procesar del asistente previa a una herramienta, habilita la liberación de la salida del asistente mientras OpenClaw sigue esperando `turn/completed`. Aumentarlo da a Codex más tiempo para emitir `turn/completed` antes de que OpenClaw interrumpa y libere el carril de la sesión.                                                                                            |
| `postToolRawAssistantCompletionIdleTimeoutMs` | `300000`                                               | Guarda de inactividad de finalización y progreso que se utiliza después de una transferencia a una herramienta, la finalización de una herramienta nativa, el progreso sin procesar del asistente posterior a una herramienta, la finalización del razonamiento sin procesar o el progreso del razonamiento mientras OpenClaw espera `turn/completed`. Utilícela para cargas de trabajo de confianza o intensivas en las que la síntesis posterior a una herramienta pueda permanecer legítimamente inactiva durante más tiempo que el presupuesto de liberación final del asistente.                                |
| `mode`                                        | `"yolo"` salvo que los requisitos locales de Codex no permitan YOLO | Preajuste para la ejecución YOLO o revisada por un guardián.                                                                                                                                                                                                                                                                                                                                                 |
| `approvalPolicy`                              | `"never"` o una política de aprobación permitida del guardián       | Política de aprobación nativa de Codex enviada al iniciar y reanudar el hilo, y durante el turno.                                                                                                                                                                                                                                                                                                                            |
| `sandbox`                                     | `"danger-full-access"` o un sandbox permitido del guardián  | Modo de sandbox nativo de Codex enviado al iniciar y reanudar el hilo. Los sandboxes activos de OpenClaw restringen los turnos `danger-full-access` al `workspace-write` de Codex; la marca de red del turno sigue la salida del sandbox de OpenClaw.                                                                                                                                                                                       |
| `approvalsReviewer`                           | `"user"` o un revisor permitido del guardián               | Utilice `"auto_review"` para permitir que Codex revise las solicitudes de aprobación nativas cuando esté permitido.                                                                                                                                                                                                                                                                                                                   |
| `defaultWorkspaceDir`                         | directorio del proceso actual                           | Espacio de trabajo utilizado por `/codex bind` cuando se omite `--cwd`.                                                                                                                                                                                                                                                                                                                                        |
| `serviceTier`                                 | no establecido                                         | Nivel de servicio opcional del app-server de Codex. `"priority"` habilita el enrutamiento en modo rápido, `"flex"` solicita procesamiento flexible y `null` elimina la anulación. El valor heredado `"fast"` se acepta como `"priority"`.                                                                                                                                                                                                 |
| `networkProxy`                                | deshabilitado                                          | Habilita de forma opcional la conectividad de red del perfil de permisos de Codex para los comandos del app-server. OpenClaw define la configuración `permissions.<profile>.network` seleccionada y la elige mediante `default_permissions` en lugar de enviar `sandbox`.                                                                                                                                                                             |
| `experimental.sandboxExecServer`              | `false`                                                | Opción de participación voluntaria en la versión preliminar que registra un entorno de Codex respaldado por un entorno aislado de OpenClaw en el servidor de aplicaciones de Codex compatible, para que la ejecución nativa de Codex pueda realizarse dentro del entorno aislado activo de OpenClaw.                                                                                                                                                                                                            |

`appServer.networkProxy` es explícito porque cambia el contrato del entorno aislado de Codex. Cuando está habilitado, OpenClaw también establece `features.network_proxy.enabled` y
`default_permissions` en la configuración del hilo de Codex para que el perfil de permisos generado pueda iniciar la red gestionada por Codex. De forma predeterminada, OpenClaw genera un nombre de perfil `openclaw-network-<fingerprint>` resistente a colisiones a partir del cuerpo del perfil; use `profileName` solo cuando se requiera un nombre local estable.

```js
export default {
  plugins: {
    entries: {
      codex: {
        config: {
          appServer: {
            sandbox: "workspace-write",
            networkProxy: {
              enabled: true,
              domains: {
                "api.openai.com": "allow",
                "blocked.example.com": "deny",
              },
              allowUpstreamProxy: true,
              proxyUrl: "http://127.0.0.1:3128",
            },
          },
        },
      },
    },
  },
};
```

Si el entorno de ejecución normal del servidor de aplicaciones fuera `danger-full-access`, habilitar
`networkProxy` utiliza en su lugar acceso al sistema de archivos de tipo espacio de trabajo para el perfil de permisos generado. La aplicación de restricciones de red gestionada por Codex constituye una red aislada, por lo que un perfil de acceso completo no protegería el tráfico saliente.

El plugin bloquea los protocolos de enlace de servidores de aplicaciones antiguos, más recientes pero no validados, de prelanzamiento, con sufijos de compilación o sin versión. El servidor de aplicaciones de Codex debe indicar una versión estable desde `0.143.0` hasta la versión `0.145.0` incluida.

OpenClaw considera remotas las URL WebSocket de servidores de aplicaciones que no sean de bucle invertido y exige autenticación WebSocket con identidad mediante `appServer.authToken` o un encabezado `Authorization`. `appServer.authToken` y cada valor de `appServer.headers.*` pueden ser un SecretInput; el entorno de ejecución de secretos resuelve las SecretRefs y la notación abreviada de variables de entorno antes de que OpenClaw cree las opciones de inicio del servidor de aplicaciones, y las SecretRefs estructuradas sin resolver provocan un fallo antes de que se envíe cualquier token o encabezado. Cuando se configuran plugins nativos de Codex, OpenClaw utiliza el plano de control de plugins del servidor de aplicaciones conectado para instalar o actualizar esos plugins y, a continuación, actualiza el inventario de aplicaciones para que las aplicaciones pertenecientes a plugins estén visibles para el hilo de Codex. `app/list` sigue siendo la fuente autoritativa del inventario y los metadatos, pero la política de OpenClaw decide si `thread/start` envía `config.apps[appId].enabled = true` para una aplicación accesible incluida en la lista, aunque Codex la marque actualmente como deshabilitada. Los identificadores de aplicaciones desconocidos o ausentes siguen provocando un fallo cerrado; esta ruta solo activa plugins del mercado mediante `plugin/install` y actualiza el inventario. Conecte OpenClaw únicamente a servidores de aplicaciones remotos en los que confíe para aceptar instalaciones de plugins gestionadas por OpenClaw y actualizaciones del inventario de aplicaciones.

## Modos de aprobación y entorno aislado

Las sesiones locales del servidor de aplicaciones mediante stdio utilizan el modo YOLO de forma predeterminada:
`approvalPolicy: "never"`, `approvalsReviewer: "user"` y
`sandbox: "danger-full-access"`. Esta postura de operador local de confianza permite que los turnos y Heartbeat desatendidos de OpenClaw progresen sin solicitudes de aprobación nativas que nadie esté presente para responder.

Si el archivo local de requisitos del sistema de Codex no permite valores implícitos de aprobación YOLO, revisor o entorno aislado, OpenClaw considera guardian el valor predeterminado implícito y selecciona permisos de guardian permitidos. `tools.exec.mode: "auto"`
también obliga a que las aprobaciones de Codex sean revisadas por guardian y no conserva las sustituciones heredadas no seguras `approvalPolicy: "never"` ni `sandbox: "danger-full-access"`;
establezca `tools.exec.mode: "full"` para adoptar intencionadamente una postura sin aprobaciones.
Las entradas `[[remote_sandbox_config]]` que coincidan con el nombre de host en el mismo archivo de requisitos se respetan al decidir el valor predeterminado del entorno aislado.

Establezca `appServer.mode: "guardian"` para que las aprobaciones de Codex sean revisadas por guardian:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            mode: "guardian",
            serviceTier: "priority",
          },
        },
      },
    },
  },
}
```

El preajuste `guardian` se expande a `approvalPolicy: "on-request"`,
`approvalsReviewer: "auto_review"` y `sandbox: "workspace-write"` cuando esos valores están permitidos. Los campos de política individuales prevalecen sobre `mode`. El valor de revisor anterior `guardian_subagent` todavía se acepta como alias de compatibilidad, pero las configuraciones nuevas deben utilizar `auto_review`.

Cuando hay un entorno aislado de OpenClaw activo, el proceso local del servidor de aplicaciones de Codex sigue ejecutándose en el host del Gateway. Por lo tanto, OpenClaw deshabilita el Code Mode nativo de Codex, los servidores MCP del usuario y la ejecución de plugins respaldada por aplicaciones para ese turno, en lugar de considerar que el aislamiento del lado del host de Codex equivale al backend de entorno aislado de OpenClaw. El acceso al shell se expone mediante herramientas dinámicas respaldadas por el entorno aislado de OpenClaw, como `sandbox_exec` y `sandbox_process`, cuando están disponibles las herramientas habituales de ejecución y procesos.

<Note>
En hosts de entorno aislado de OpenClaw respaldados por Docker (`agents.defaults.sandbox.mode` establecido en
un backend de Docker), `openclaw doctor` comprueba si el host permite los espacios de nombres de usuario sin privilegios (y, cuando la salida de red del entorno aislado de Docker está deshabilitada, los espacios de nombres de red) que el `bwrap` anidado de Codex necesita para ejecutar el shell mediante `workspace-write` dentro del contenedor del entorno aislado. Una comprobación fallida suele manifestarse como `bwrap: setting up uid map: Permission denied` o
`bwrap: loopback: Failed RTM_NEWADDR: Operation not permitted` en
hosts Ubuntu/AppArmor. Corrija la política de espacios de nombres del host indicada para el usuario del servicio OpenClaw y reinicie el Gateway; es preferible utilizar un perfil de AppArmor limitado al proceso del servicio en lugar de la alternativa `kernel.apparmor_restrict_unprivileged_userns=0` para todo el host, y no conceda privilegios más amplios al contenedor de Docker solo para satisfacer el `bwrap` anidado.
</Note>

## Ejecución nativa en entorno aislado

El valor predeterminado estable es el fallo cerrado: el aislamiento activo de OpenClaw deshabilita las superficies de ejecución nativa de Codex que, de otro modo, se ejecutarían desde el host del servidor de aplicaciones de Codex. Use `appServer.experimental.sandboxExecServer: true` solo cuando desee probar la compatibilidad de Codex con entornos remotos mediante el backend de entorno aislado de OpenClaw.
Esta ruta de vista previa funciona con todas las versiones compatibles del servidor de aplicaciones de Codex.

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            experimental: {
              sandboxExecServer: true,
            },
          },
        },
      },
    },
  },
}
```

Cuando la marca está activada y la sesión actual de OpenClaw está aislada, OpenClaw inicia un servidor de ejecución local de bucle invertido respaldado por el entorno aislado activo, lo registra en el servidor de aplicaciones de Codex e inicia el hilo y el turno de Codex con ese entorno perteneciente a OpenClaw. Si el servidor de aplicaciones no puede registrar el entorno, la ejecución falla de forma cerrada en lugar de recurrir silenciosamente a la ejecución en el host.

Esta ruta de vista previa es solo local. Un servidor de aplicaciones WebSocket remoto no puede acceder al servidor de ejecución de bucle invertido a menos que se ejecute en el mismo host, por lo que OpenClaw rechaza esa combinación.

## Aislamiento de autenticación y entorno

En el directorio principal por agente predeterminado, la autenticación se selecciona en este orden:

1. Un perfil explícito de autenticación de Codex de OpenClaw para el agente.
2. La cuenta existente del servidor de aplicaciones en el directorio principal de Codex de ese agente.
3. Solo para inicios locales del servidor de aplicaciones mediante stdio, `CODEX_API_KEY` y después
   `OPENAI_API_KEY`, cuando no hay ninguna cuenta del servidor de aplicaciones y la autenticación de OpenAI sigue siendo necesaria.

Cuando OpenClaw detecta un perfil de autenticación de Codex de tipo suscripción de ChatGPT (tipo de credencial OAuth o token), elimina `CODEX_API_KEY` y `OPENAI_API_KEY` del proceso secundario de Codex iniciado. Esto mantiene las claves de API del Gateway disponibles para incrustaciones o modelos directos de OpenAI sin provocar accidentalmente que los turnos nativos del servidor de aplicaciones de Codex se facturen mediante la API.

Los perfiles explícitos de clave de API de Codex y la alternativa local de clave de entorno mediante stdio utilizan el inicio de sesión del servidor de aplicaciones en lugar de heredar el entorno del proceso secundario. Las conexiones WebSocket del servidor de aplicaciones no reciben la alternativa de clave de API del entorno del Gateway; use un perfil de autenticación explícito o la cuenta propia del servidor de aplicaciones remoto.

Los inicios del servidor de aplicaciones mediante stdio heredan de forma predeterminada el entorno del proceso de OpenClaw.
OpenClaw controla el puente de cuentas del servidor de aplicaciones de Codex y establece `CODEX_HOME` en un directorio por agente dentro del estado de OpenClaw de ese agente. Esto mantiene la configuración, las cuentas, la caché y los datos de los plugins, y el estado de los hilos de Codex limitados al agente de OpenClaw, en lugar de filtrarlos desde el directorio principal personal `~/.codex` del operador.

Establezca `appServer.homeScope: "user"` para compartir el estado nativo de Codex con Codex Desktop y la CLI. Este modo de directorio principal del usuario admite stdio gestionado y transporte Unix explícito. Utiliza `$CODEX_HOME` cuando está establecido y `~/.codex` en caso contrario, incluida la autenticación nativa, la configuración, los plugins y los hilos.
OpenClaw omite su puente de perfiles de autenticación para el servidor de aplicaciones. Los turnos verificados del propietario pueden usar `codex_threads` para enumerar (con un filtro `search` opcional), leer, bifurcar, cambiar el nombre, archivar y desarchivar esos hilos. Bifurque un hilo antes de continuarlo en OpenClaw; los procesos independientes de Codex no coordinan escritores simultáneos para el mismo hilo.

Esa activación voluntaria de `homeScope` se aplica a las sesiones habituales del arnés. Un Chat creado mediante Codex Sessions utiliza en su lugar su conexión privada de supervisión, que conserva la configuración de autenticación y proveedor de la conexión nativa para la rama canónica y futuras reanudaciones.

En un Chat supervisado y bloqueado a un modelo, `codex_threads` no puede adjuntar una bifurcación distinta ni archivar el hilo nativo vinculado al Chat. La enumeración y la lectura exclusiva de metadatos siguen disponibles. Las lecturas de la transcripción sin procesar requieren `allowRawTranscripts`; cuando está deshabilitado, también se rechaza la búsqueda en listas porque la búsqueda nativa puede encontrar coincidencias en las vistas previas de transcripciones. Cambiar el nombre, desarchivar, realizar una bifurcación independiente y archivar un hilo no relacionado que no pertenezca a otro Chat de OpenClaw requiere
`allowWriteControls`. Ninguna de las opciones omite un vínculo bloqueado.

OpenClaw no reescribe `HOME` para los inicios locales normales del servidor de aplicaciones.
Los subprocesos ejecutados por Codex, como `openclaw`, `gh`, `git`, las CLI en la nube y los comandos de shell, ven el directorio principal normal del proceso y pueden encontrar la configuración y los tokens del directorio principal del usuario. Codex también puede detectar `$HOME/.agents/skills` y
`$HOME/.agents/plugins/marketplace.json`; esa detección de `.agents` se comparte intencionadamente con el directorio principal del operador y es independiente del estado aislado de `~/.codex`.

En el ámbito predeterminado del agente, los plugins de OpenClaw y las instantáneas de Skills de OpenClaw siguen fluyendo mediante el registro de plugins y el cargador de Skills propios de OpenClaw; los recursos personales `~/.codex` de Codex no. Si existen Skills o plugins útiles de la CLI de Codex en un directorio principal de Codex que deban formar parte de un agente aislado de OpenClaw, realice explícitamente su inventario:

```bash
openclaw migrate codex --dry-run
openclaw migrate apply codex --yes
```

Si un despliegue necesita aislamiento adicional del entorno, añada esas variables a `appServer.clearEnv`:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            clearEnv: ["CODEX_API_KEY", "OPENAI_API_KEY"],
          },
        },
      },
    },
  },
}
```

`appServer.clearEnv` solo afecta al proceso secundario iniciado del servidor de aplicaciones de Codex.
OpenClaw elimina `CODEX_HOME` y `HOME` de esta lista durante la normalización del inicio local: `CODEX_HOME` sigue apuntando al ámbito seleccionado del agente o usuario, y `HOME` continúa heredándose para que los subprocesos puedan utilizar el estado normal del directorio principal del usuario.

## Herramientas dinámicas

Las herramientas dinámicas de Codex utilizan de forma predeterminada la carga `searchable`, expuesta en el espacio de nombres `openclaw` con `deferLoading: true`. OpenClaw normalmente no expone herramientas dinámicas que dupliquen las operaciones nativas de Codex sobre el espacio de trabajo o la propia superficie de búsqueda de herramientas de Codex:

- `read`
- `write`
- `edit`
- `apply_patch`
- `exec`
- `process`
- `update_plan`
- `tool_call`
- `tool_describe`
- `tool_search`
- `tool_search_code`

Cuando una lista de permitidos finita del entorno de ejecución deshabilita el Code Mode nativo, OpenClaw envía una selección vacía del entorno de ejecución. En ese caso directo y sin aislamiento, OpenClaw conserva sus herramientas `exec` y `process`, filtradas según la política, como alternativa para el shell. Las listas de permitidos del entorno de ejecución y `codexDynamicToolsExclude` siguen siendo aplicables.

La mayoría de las herramientas de integración restantes de OpenClaw, como mensajería, contenido multimedia, cron,
navegador, nodos, gateway, `heartbeat_respond` y `web_search`, están disponibles
mediante la búsqueda de herramientas de Codex en ese espacio de nombres. Esto mantiene más pequeño el contexto
inicial del modelo. Un pequeño conjunto de herramientas permanece disponible para llamadas directas independientemente de
`codexDynamicToolsLoading`, porque la búsqueda de herramientas de Codex puede no estar disponible o
resolver un universo exclusivo de conectores: `agents_list`, `sessions_spawn` y
`sessions_yield`. Las instrucciones para desarrolladores siguen orientando a los subagentes normales de Codex
hacia `spawn_agent` nativo para el trabajo de subagentes nativos de Codex, mientras que
`sessions_spawn` permanece disponible para la delegación explícita de OpenClaw o ACP.
Las respuestas de origen que solo usan herramientas de mensajería también permanecen directas, ya que se trata de un
contrato de control de turnos.

El Modo Código de Codex proyecta como texto los resultados genéricos de herramientas dinámicas de OpenClaw. Analice un
resultado JSON antes de leer sus campos. Las llamadas dinámicas anidadas se serializan mediante el
entorno de ejecución de Codex, por lo que `Promise.all` no las envía de forma simultánea; use un
bucle de lanzamiento secuencial acotado al iniciar procesos secundarios recopiladores.

Las herramientas marcadas con `catalogMode: "direct-only"`, incluida la herramienta `computer` de
OpenClaw, se agrupan bajo `openclaw_direct`. OpenClaw añade ese espacio de nombres a
la lista `code_mode.direct_only_tool_namespaces` de Codex sin reemplazar las entradas
proporcionadas por el operador. Por lo tanto, Codex expone esas herramientas como
`DirectModelOnly` en hilos normales y exclusivos del modo código, en lugar de encaminarlas
a través de llamadas anidadas `tools.*` del Modo Código. Este límite es necesario para
los resultados que contienen imágenes: la serialización anidada del Modo Código convierte la salida de imágenes en
texto, lo que descartaría la captura de pantalla necesaria para la siguiente acción en el equipo.

Establezca `codexDynamicToolsLoading: "direct"` solo al conectarse a un servidor de aplicaciones
Codex personalizado que no pueda buscar herramientas dinámicas diferidas o al depurar
la carga completa de herramientas.

## Tiempos de espera

Las llamadas a herramientas dinámicas propiedad de OpenClaw se limitan de forma independiente de
`appServer.requestTimeoutMs`. Cada solicitud `item/tool/call` de Codex usa el
primer tiempo de espera disponible en este orden:

- Un argumento positivo `timeoutMs` por llamada.
- Para `image_generate`, `agents.defaults.mediaModels.image.timeoutMs`.
- Para `image_generate` sin un tiempo de espera configurado, el valor predeterminado de 120 segundos
  para la generación de imágenes.
- Para la herramienta `image` de comprensión multimedia, el valor `timeoutSeconds`
  de la entrada `tools.media.models[]` seleccionada compatible con imágenes,
  convertido a milisegundos, o el valor predeterminado de 60 segundos para contenido multimedia. Para la comprensión
  de imágenes, esto se aplica a la solicitud en sí y no se reduce por
  el trabajo de preparación anterior.
- Para la herramienta `message`, un presupuesto externo fijo de 600 segundos que abarca la entrega del Gateway y la conciliación acotada con la misma clave.
- El valor predeterminado de 90 segundos para herramientas dinámicas.

Este mecanismo de vigilancia es el presupuesto externo dinámico de `item/tool/call`. Los tiempos de espera de
solicitudes específicos del proveedor se ejecutan dentro de esa llamada y conservan su propia semántica de tiempo de espera.
Los presupuestos de herramientas dinámicas tienen un límite de 600000 ms. `agents_wait` añade 30000 ms de
margen externo para la finalización, y el cliente del servidor de aplicaciones permite 660000 ms para que
el resultado estructurado de la espera pueda llegar a Codex. Cuando se agota el tiempo de espera, OpenClaw cancela la señal de la herramienta
cuando es compatible y devuelve a Codex una respuesta fallida de la herramienta dinámica para que
el turno pueda continuar, en lugar de dejar la sesión en `processing`.

Después de que Codex acepte un turno y después de que OpenClaw responda a una solicitud del servidor de aplicaciones
con alcance de turno, el arnés espera que Codex avance en el turno actual
y termine finalmente el turno nativo con `turn/completed`. Si el
servidor de aplicaciones permanece inactivo durante `appServer.turnCompletionIdleTimeoutMs`, OpenClaw
intenta interrumpir el turno de Codex, registra un tiempo de espera de diagnóstico y
libera el canal de sesión de OpenClaw para que los mensajes posteriores del chat no queden en cola
detrás de un turno nativo obsoleto.

La mayoría de las notificaciones no terminales del mismo turno desactivan ese breve mecanismo de vigilancia
porque Codex ha demostrado que el turno sigue activo. Las transferencias de herramientas usan un presupuesto
de inactividad posterior a la herramienta más largo: después de que OpenClaw devuelva una respuesta `item/tool/call`,
después de que se completen elementos de herramientas nativas como `commandExecution`, después de finalizaciones
`custom_tool_call_output` sin procesar y después del progreso del asistente
sin procesar posterior a la herramienta, las finalizaciones de razonamiento sin procesar o el progreso del razonamiento. El mecanismo de protección usa
`appServer.postToolRawAssistantCompletionIdleTimeoutMs` cuando está configurado y
utiliza cinco minutos de forma predeterminada en caso contrario. Ese mismo presupuesto posterior a la herramienta también amplía
el mecanismo de vigilancia del progreso durante el intervalo de síntesis silencioso antes de que Codex emita el
siguiente evento del turno actual. Las finalizaciones de razonamiento, las finalizaciones `agentMessage`
de comentarios y el progreso del razonamiento o del asistente sin procesar anterior a la herramienta pueden ir seguidos
de una respuesta final automática, por lo que usan el mecanismo de protección de respuesta posterior al progreso
en lugar de liberar inmediatamente el canal de sesión. Solo los elementos `agentMessage`
completados finales o que no son comentarios y las finalizaciones del asistente sin procesar anteriores a la herramienta activan la
liberación de la salida del asistente: si Codex permanece inactivo después sin `turn/completed`,
OpenClaw intenta interrumpir el turno nativo y libera el canal de
sesión. Los fallos del servidor de aplicaciones stdio que permiten una reproducción segura, incluidos los tiempos de espera
por inactividad al completar el turno sin indicios del asistente, herramientas, elementos activos ni efectos secundarios, se
reintentan una vez mediante un nuevo intento del servidor de aplicaciones. Los tiempos de espera no seguros siguen retirando el
cliente del servidor de aplicaciones bloqueado y liberan el canal de sesión de OpenClaw. También
eliminan la vinculación obsoleta del hilo nativo en lugar de reproducirse
automáticamente. Los tiempos de espera de supervisión de finalización muestran texto de tiempo de espera específico de Codex:
los casos seguros para reproducción indican que la respuesta puede estar incompleta, mientras que los casos no seguros indican
al usuario que verifique el estado actual antes de volver a intentarlo. Los diagnósticos públicos de tiempo de espera
incluyen campos estructurales como el último método de notificación del servidor de aplicaciones,
el id/tipo/rol del elemento de respuesta sin procesar del asistente, los recuentos de solicitudes y elementos activos y
el estado de vigilancia activado. Cuando la última notificación es un elemento de respuesta sin procesar del asistente,
también incluyen una vista previa acotada del texto del asistente. No
incluyen el contenido sin procesar de solicitudes ni herramientas.

## Detección de modelos

De forma predeterminada, el Plugin Codex solicita al servidor de aplicaciones los modelos disponibles. La
disponibilidad de los modelos pertenece al servidor de aplicaciones de Codex, por lo que la lista puede cambiar cuando
OpenClaw actualiza la versión incluida de `@openai/codex` o cuando una implementación
dirige `appServer.command` a un binario distinto de Codex. La disponibilidad también
puede depender de la cuenta. Use `/codex models` en un gateway en ejecución para ver el catálogo
activo de ese arnés y esa cuenta.

Si la detección falla o agota el tiempo de espera, OpenClaw usa un catálogo alternativo incluido:

| Id. del modelo       | Nombre para mostrar | Niveles de razonamiento        |
| -------------- | ------------ | ------------------------ |
| `gpt-5.5`      | gpt-5.5      | bajo, medio, alto, muy alto |
| `gpt-5.4-mini` | GPT-5.4-Mini | bajo, medio, alto, muy alto |

<Note>
El arnés incluido actualmente es `@openai/codex` `0.145.0`. Una comprobación `model/list`
del servidor de aplicaciones incluido devolvió estas filas públicas del selector:

| Id. del modelo        | Modalidades de entrada | Niveles de razonamiento                    |
| --------------- | ---------------- | ------------------------------------ |
| `gpt-5.6-sol`   | texto, imagen      | bajo, medio, alto, muy alto, máximo, ultra |
| `gpt-5.6-terra` | texto, imagen      | bajo, medio, alto, muy alto, máximo, ultra |
| `gpt-5.6-luna`  | texto, imagen      | bajo, medio, alto, muy alto, máximo        |
| `gpt-5.5`       | texto, imagen      | bajo, medio, alto, muy alto             |
| `gpt-5.2`       | texto, imagen      | bajo, medio, alto, muy alto             |

El catálogo del servidor de aplicaciones puede informar de `ultra`; los controles de razonamiento de OpenClaw actualmente
exponen niveles hasta `max`.

Las filas activas del selector dependen de la cuenta y pueden cambiar según la cuenta, el catálogo de Codex
o la versión incluida; ejecute `/codex models` para obtener la lista actual en lugar de
depender de una tabla correspondiente a un momento concreto. También pueden aparecer modelos ocultos en el
catálogo del servidor de aplicaciones para flujos internos o especializados sin ser opciones normales
del selector de modelos.
</Note>

Ajuste la detección en `plugins.entries.codex.config.discovery`:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          discovery: {
            enabled: true,
            timeoutMs: 2500,
          },
        },
      },
    },
  },
}
```

Desactive la detección cuando quiera que el inicio evite consultar Codex y use únicamente
el catálogo alternativo:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          discovery: {
            enabled: false,
          },
        },
      },
    },
  },
}
```

## Archivos de arranque del espacio de trabajo

Codex gestiona `AGENTS.md` por sí mismo mediante la detección nativa de documentación del proyecto.
OpenClaw no escribe archivos sintéticos de documentación de proyectos de Codex ni depende de los
nombres de archivo alternativos de Codex para los archivos de personalidad, porque las alternativas de Codex solo se aplican cuando
falta `AGENTS.md`.

Para mantener la paridad del espacio de trabajo de OpenClaw, el arnés Codex reenvía los demás
archivos de arranque como instrucciones para desarrolladores, pero no de forma idéntica:

- `TOOLS.md` se reenvía como instrucciones para desarrolladores **heredadas** de Codex, por lo que
  los subagentes nativos de Codex generados durante el turno también las reciben.
- `SOUL.md`, `IDENTITY.md` y `USER.md` se reenvían como instrucciones de colaboración
  **con alcance de turno**. Los subagentes nativos de Codex no las heredan,
  lo que evita que sus turnos adopten la personalidad y el
  perfil de usuario del agente principal.
- La lista compacta de Skills cargadas de OpenClaw también se reenvía como instrucciones para desarrolladores de
  colaboración con alcance de turno, de modo que los subagentes nativos de Codex tampoco
  la heredan.
- El contenido de `HEARTBEAT.md` no se inyecta; los turnos de Heartbeat reciben un
  puntero en modo de colaboración para leer el archivo cuando existe y no está
  vacío.
- El contenido de `MEMORY.md` del espacio de trabajo del agente configurado no se pega en
  la entrada de turnos nativa de Codex cuando las herramientas de memoria están disponibles para ese
  espacio de trabajo; cuando existe, el arnés añade un pequeño puntero a la memoria del espacio de trabajo
  a las instrucciones para desarrolladores de colaboración con alcance de turno, y Codex
  debe usar `memory_search` o `memory_get` cuando la memoria persistente sea pertinente.
  Si las herramientas están desactivadas, la búsqueda en memoria no está disponible o el espacio de trabajo
  activo difiere del espacio de trabajo de memoria del agente, `MEMORY.md` usa la
  ruta normal acotada del contexto del turno.
- `BOOTSTRAP.md`, cuando está presente, se reenvía como contexto de referencia de entrada
  del turno de OpenClaw.

## Sustituciones del entorno

Las sustituciones del entorno siguen disponibles para las pruebas locales:

- `OPENCLAW_CODEX_APP_SERVER_BIN`
- `OPENCLAW_CODEX_APP_SERVER_ARGS`
- `OPENCLAW_CODEX_APP_SERVER_MODE=yolo|guardian`
- `OPENCLAW_CODEX_APP_SERVER_APPROVAL_POLICY`
- `OPENCLAW_CODEX_APP_SERVER_SANDBOX`

`OPENCLAW_CODEX_APP_SERVER_BIN` omite el binario administrado cuando
`appServer.command` no está establecido.

Se eliminó `OPENCLAW_CODEX_APP_SERVER_GUARDIAN=1`. Use
`plugins.entries.codex.config.appServer.mode: "guardian"` en su lugar o
`OPENCLAW_CODEX_APP_SERVER_MODE=guardian` para pruebas locales puntuales. Se
prefiere la configuración para implementaciones reproducibles porque mantiene el comportamiento del Plugin en
el mismo archivo revisado que el resto de la configuración del arnés Codex.

## Temas relacionados

- [Arnés Codex](/es/plugins/codex-harness)
- [Entorno de ejecución del arnés Codex](/es/plugins/codex-harness-runtime)
- [Supervisión de Codex](/es/plugins/codex-supervision)
- [Plugins nativos de Codex](/es/plugins/codex-native-plugins)
- [Uso del equipo con Codex](/es/plugins/codex-computer-use)
- [Proveedor OpenAI](/es/providers/openai)
- [Referencia de configuración](/es/gateway/configuration-reference)
