---
read_when:
    - Quieres que los agentes de OpenClaw en modo Codex usen Codex Computer Use
    - Está decidiendo entre Codex Computer Use, PeekabooBridge y el MCP directo de cua-driver
    - Se está configurando computerUse para el plugin de Codex incluido
    - Está solucionando problemas con el estado o la instalación del uso del ordenador de /codex
summary: Configurar Codex Computer Use para agentes de OpenClaw en modo Codex
title: Uso de computadoras con Codex
x-i18n:
    generated_at: "2026-07-26T05:19:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6b11d00c74bc2990a4e33b6ffe23209ed76a1e10180ce5950dbb5073ea57ad05
    source_path: plugins/codex-computer-use.md
    workflow: 16
---

Computer Use es un plugin MCP nativo de Codex para controlar el escritorio local. OpenClaw
no incluye la aplicación de escritorio, no ejecuta por sí mismo acciones de escritorio ni elude
los permisos de Codex. El plugin `codex` incluido solo prepara Codex app-server:
habilita la compatibilidad con plugins de Codex, busca o instala el plugin Computer Use
configurado, comprueba que el servidor MCP `computer-use` esté disponible y después permite
que Codex gestione las llamadas nativas a herramientas MCP durante los turnos en modo Codex.

Use esta página cuando OpenClaw ya utilice el arnés nativo de Codex. Para configurar
el propio entorno de ejecución, consulte [Arnés de Codex](/es/plugins/codex-harness).

Esto es distinto de la [herramienta informática integrada basada en nodos](/es/nodes/computer-use) de OpenClaw. Use la herramienta integrada cuando el mismo contrato de agente deba controlar un Mac emparejado, tanto si el agente se ejecuta en el Gateway como en otro nodo. Use Codex Computer Use cuando Codex app-server deba gestionar la instalación local de MCP, los permisos y las llamadas nativas a herramientas.

## OpenClaw.app y Peekaboo

La integración de Peekaboo de OpenClaw.app es independiente de Codex Computer Use. La
aplicación para macOS puede alojar un socket PeekabooBridge para que la CLI `peekaboo` reutilice las
concesiones locales de Accesibilidad y Grabación de pantalla de la aplicación para las propias
herramientas de automatización de Peekaboo. Ese puente no instala ni actúa como proxy de Codex Computer Use, y
Codex Computer Use no realiza llamadas a través del socket PeekabooBridge.

Use [Puente de Peekaboo](/es/platforms/mac/peekaboo) cuando quiera que OpenClaw.app sea
un host consciente de los permisos para la automatización con la CLI de Peekaboo. Use esta página cuando un
agente de OpenClaw en modo Codex deba tener disponible el plugin MCP nativo
`computer-use` de Codex antes de que comience el turno.

## Aplicación para iOS

La aplicación para iOS es independiente de Codex Computer Use. No instala ni actúa como proxy
del servidor MCP `computer-use` de Codex y no es un backend de control de escritorio.
En su lugar, la aplicación para iOS se conecta como un nodo de OpenClaw y expone
funciones móviles mediante comandos de nodo como `canvas.*`, `camera.*`, `screen.*`,
`location.*` y `talk.*`.

Use [iOS](/es/platforms/ios) cuando quiera que un agente controle un nodo iPhone
a través del Gateway. Use esta página cuando un agente en modo Codex deba controlar el
escritorio local de macOS mediante el plugin nativo Computer Use de Codex.

## MCP cua-driver directo

Codex Computer Use no es la única forma de exponer el control del escritorio. Si quiere que
los entornos de ejecución gestionados por OpenClaw llamen directamente al controlador de TryCua, use el servidor
`cua-driver mcp` original mediante el registro MCP de OpenClaw en lugar del
flujo de marketplace específico de Codex.

Después de instalar `cua-driver`, solicítele el comando de OpenClaw:

```bash
cua-driver mcp-config --client openclaw
```

o registre directamente el servidor stdio:

```bash
openclaw mcp set cua-driver '{"command":"cua-driver","args":["mcp"]}'
```

Esa vía mantiene intacta la superficie de herramientas MCP original, incluidos los esquemas
del controlador y las respuestas MCP estructuradas. Úsela cuando quiera que el controlador CUA
esté disponible como un servidor MCP normal de OpenClaw. Use la configuración de Codex Computer Use de
esta página cuando Codex app-server deba gestionar la instalación del plugin, las recargas de MCP
y las llamadas nativas a herramientas dentro de los turnos en modo Codex.

El controlador de CUA distribuye compilaciones preliminares para macOS, Windows (x64 y ARM64) y
Linux (x64 y ARM64, nivel de vista previa). Sigue requiriendo los permisos locales del
sistema operativo que solicita su aplicación, como Accesibilidad y Grabación de pantalla en
macOS. OpenClaw no instala `cua-driver`, no concede esos permisos ni
elude el modelo de seguridad del controlador original.

## Configuración rápida

Establezca `plugins.entries.codex.config.computerUse` cuando los turnos en modo Codex deban tener
Computer Use disponible antes de que se inicie un hilo. `autoInstall: true` habilita
Computer Use y permite que OpenClaw lo instale o vuelva a habilitar antes del turno:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          computerUse: {
            autoInstall: true,
          },
        },
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.6-sol",
    },
  },
}
```

Con esta configuración, OpenClaw comprueba Codex app-server antes de cada turno en modo
Codex. Si falta Computer Use, pero Codex app-server ya ha detectado
un marketplace que permite su instalación, OpenClaw solicita a Codex app-server que instale o
vuelva a habilitar el plugin y recargue los servidores MCP. Antes de iniciar un
Codex app-server aislado en macOS, la instalación automática también copia la aplicación de servicio oficial
y firmada de Computer Use desde el paquete de la aplicación de escritorio seleccionada al
directorio `computer-use` de ese entorno principal de Codex cuando falta el cliente nativo.
En macOS, cuando no hay ningún
marketplace correspondiente registrado y existe un paquete estándar de la aplicación de escritorio, OpenClaw
también intenta registrar el marketplace de Codex incluido desde
`/Applications/ChatGPT.app/Contents/Resources/plugins/openai-bundled`, y conserva
`/Applications/Codex.app/Contents/Resources/plugins/openai-bundled` como alternativa
para instalaciones independientes heredadas. Si la configuración sigue sin poder hacer que el
servidor MCP esté disponible, el turno falla antes de que se inicie el hilo.
Los fallos estrictos de disponibilidad son fallos de comprobación previa del arnés, por lo que la alternativa
de modelo no repite la misma secuencia local de comprobación para cada modelo candidato.

Después de cambiar la configuración de Computer Use, use `/new` o `/reset` en el chat
afectado antes de realizar pruebas si ya se ha iniciado un hilo de Codex.

En macOS, el inicio gestionado de Computer Use da preferencia al binario de la aplicación de escritorio en
`/Applications/ChatGPT.app/Contents/Resources/codex` y después recurre
a `/Applications/Codex.app/Contents/Resources/codex` para instalaciones
independientes heredadas. Esto también se aplica a los comandos puntuales de estado e
instalación de Computer Use que inician su propio cliente. Mantiene el control del escritorio bajo
el paquete de la aplicación que posee los permisos locales de macOS. Si la aplicación de escritorio no está
instalada, OpenClaw recurre al binario gestionado de Codex instalado junto al
plugin. Los turnos gestionados normales de Codex con el entorno principal aislado predeterminado del agente dan preferencia
a ese paquete fijado para que una aplicación de escritorio antigua no pueda eclipsar la compatibilidad con los
modelos actuales. Los entornos principales del ámbito del usuario siguen dando preferencia al escritorio porque pueden cargar el estado
nativo de Computer Use. Un entorno principal aislado de agente cuya configuración efectiva de Codex habilite
Computer Use también sigue dando preferencia al escritorio. La configuración explícita
`appServer.command` o `OPENCLAW_CODEX_APP_SERVER_BIN` continúa teniendo prioridad sobre
esta selección gestionada.

OpenClaw serializa las lecturas de la configuración nativa de Codex y la instalación de Computer Use
dentro de un mismo Gateway en ejecución. Un proceso de Codex independiente u otro Gateway no forma
parte de ese bloqueo. Después de cambiar la configuración nativa de plugins de Codex fuera del
Gateway, reinicie el Gateway e inicie un chat nuevo antes de depender de la nueva
selección.

## Comandos

Use los comandos `/codex computer-use` desde cualquier superficie de chat donde esté
disponible la superficie de comandos del plugin `codex`. Son comandos de chat y del entorno de ejecución
de OpenClaw, no subcomandos de la CLI `openclaw codex ...`:

```text
/codex computer-use status
/codex computer-use install
/codex computer-use install --source <marketplace-source>
/codex computer-use install --marketplace-path <path>
/codex computer-use install --marketplace <name>
```

`status` es la acción predeterminada y es de solo lectura: no añade fuentes de
marketplace, no instala plugins ni habilita la compatibilidad con plugins de Codex. Si ninguna configuración habilita
Computer Use, `status` puede indicar que está deshabilitado incluso después de un comando puntual de
instalación.

`install` habilita la compatibilidad de Codex app-server con plugins, añade opcionalmente una
fuente de marketplace configurada, instala o vuelve a habilitar el plugin configurado
mediante Codex app-server, recarga los servidores MCP y verifica que el servidor MCP
exponga herramientas. Como la instalación modifica recursos de confianza del host,
solo un propietario o un cliente de Gateway `operator.admin` puede ejecutar `install`. Los demás
remitentes autorizados pueden seguir usando el comando de solo lectura `status`,
incluso con valores de sustitución.

Las versiones anteriores aceptaban valores puntuales de sustitución de identidad `--plugin`, `--server` y `--mcp-server`.
En su lugar, configure de forma persistente `computerUse.pluginName` y
`computerUse.mcpServerName`. Cuando se usa una marca de identidad heredada,
el comando identifica el ajuste exacto que debe conservarse y repite la
acción solicitada junto con cualquier marca de marketplace compatible en sus indicaciones de migración.

## Opciones de marketplace

OpenClaw usa la misma API de app-server que expone el propio Codex. Los
campos de marketplace determinan dónde debe encontrar Codex `computer-use`.

| Campo                | Cuándo usarlo                                                        | Compatibilidad con la instalación                                          |
| -------------------- | --------------------------------------------------------------- | -------------------------------------------------------- |
| Sin campo de marketplace | Se quiere que Codex app-server use los marketplaces que ya conoce. | Sí, cuando app-server devuelve un marketplace local.        |
| `marketplaceSource`  | Se dispone de una fuente de marketplace de Codex que app-server puede añadir.         | Sí, para `/codex computer-use install` explícito.         |
| `marketplacePath`    | Ya se conoce la ruta local del archivo de marketplace en el host.   | Sí, para la instalación explícita y la instalación automática al inicio del turno.   |
| `marketplaceName`    | Se quiere seleccionar por nombre un marketplace ya registrado.  | Sí, solo cuando el marketplace seleccionado tiene una ruta local. |

Los entornos principales nuevos de Codex pueden necesitar un breve momento para inicializar sus
marketplaces oficiales. Durante la instalación, OpenClaw consulta `plugin/list` durante un máximo de
`marketplaceDiscoveryTimeoutMs` milisegundos (60 segundos de forma predeterminada).

Si varios marketplaces conocidos contienen Computer Use, OpenClaw da preferencia a
`openai-bundled`, después a `openai-curated` y, por último, a `local`. Las coincidencias ambiguas
desconocidas producen un fallo seguro y solicitan establecer `marketplaceName` o
`marketplacePath`.

## Marketplace incluido para macOS

Las compilaciones actuales de ChatGPT para escritorio incluyen Computer Use aquí; las compilaciones independientes
heredadas de Codex para escritorio usan la misma disposición bajo `Codex.app`:

```text
/Applications/ChatGPT.app/Contents/Resources/plugins/openai-bundled/plugins/computer-use
/Applications/Codex.app/Contents/Resources/plugins/openai-bundled/plugins/computer-use
```

Cuando `computerUse.autoInstall` es true y no hay registrado ningún marketplace que contenga
`computer-use`, OpenClaw intenta añadir la primera raíz estándar de
marketplace incluido que exista:

```text
/Applications/ChatGPT.app/Contents/Resources/plugins/openai-bundled
/Applications/Codex.app/Contents/Resources/plugins/openai-bundled
```

También se puede registrar explícitamente desde un shell con Codex:

```bash
codex plugin marketplace add /Applications/ChatGPT.app/Contents/Resources/plugins/openai-bundled
```

Si se usa una ruta no estándar de la aplicación Codex, ejecute `/codex computer-use install
--source <marketplace-root>` una vez o establezca `computerUse.marketplacePath` en una
ruta de archivo de marketplace local. Use `--marketplace-path` solo cuando disponga de la
ruta del archivo JSON del marketplace, no de la raíz del marketplace incluido.

### Caché compartida de plugins

El valor predeterminado `pluginCacheMode: "independent"` deja sin gestionar cada entorno principal de Codex y su
caché de plugins. Establezca `pluginCacheMode: "shared"` para copiar el plugin Computer Use
incluido en la caché de plugins detectable del entorno principal activo de Codex
antes de iniciar app-server. El modo compartido conserva las versiones antiguas en caché porque
los clientes de Codex en ejecución aún pueden hacer referencia a sus directorios versionados de plugins; si
falla una copia de sustitución, también se conserva la caché activa. La configuración explícita
`marketplaceName` o `marketplacePath` deshabilita esta
conciliación para que OpenClaw no sustituya esa selección.

## Límite del catálogo remoto

Codex app-server puede enumerar y leer entradas de catálogo exclusivamente remotas, pero actualmente
no admite `plugin/install` remoto. Esto significa que `marketplaceName`
puede seleccionar un marketplace exclusivamente remoto para las comprobaciones de estado, pero las instalaciones y
rehabilitaciones siguen necesitando un marketplace local mediante `marketplaceSource` o
`marketplacePath`.

Si el estado indica que el plugin está disponible en un marketplace remoto de Codex, pero
la instalación remota no es compatible, ejecute la instalación con una fuente o ruta local:

```text
/codex computer-use install --source <marketplace-source>
/codex computer-use install --marketplace-path <path>
```

## Referencia de configuración

| Campo                           | Valor predeterminado        | Significado                                                                        |
| ------------------------------- | -------------- | ------------------------------------------------------------------------------ |
| `enabled`                       | inferido       | Requiere Computer Use. El valor predeterminado es true cuando se establece otro campo de Computer Use. |
| `autoInstall`                   | false          | Aprovisiona el cliente nativo e instala o vuelve a habilitar el plugin al inicio del turno. |
| `marketplaceDiscoveryTimeoutMs` | 60000          | Tiempo que la instalación espera a que el app-server de Codex detecte el marketplace.             |
| `liveTestTimeoutMs`             | 60000          | Tiempo de espera del hilo temporal de disponibilidad y sus solicitudes de limpieza.           |
| `toolCallTimeoutMs`             | 60000          | Tiempo de espera de la llamada a la herramienta de disponibilidad `list_apps` de Computer Use.                  |
| `healthCheckEnabled`            | false          | Ejecuta comprobaciones periódicas de disponibilidad mientras el cliente propietario del app-server esté activo.    |
| `healthCheckIntervalMinutes`    | 60             | Frecuencia de comprobación; los valores aceptados son 30, 60, 120 o 240 minutos.                |
| `pluginCacheMode`               | `independent`  | Usa `shared` para actualizar la caché del directorio principal de Codex desde el plugin de escritorio incluido.  |
| `strictReadiness`               | false          | Detiene el inicio si falla una comprobación en vivo, en lugar de continuar con una advertencia.      |
| `autoRepair`                    | false          | Finaliza los procesos secundarios MCP obsoletos de Computer Use del ámbito correspondiente y reintenta una vez la comprobación fallida.     |
| `marketplaceSource`             | sin establecer          | Cadena de origen que se pasa a `marketplace/add` del app-server de Codex.                    |
| `marketplacePath`               | sin establecer          | Ruta local al archivo del marketplace de Codex que contiene el plugin.                       |
| `marketplaceName`               | sin establecer          | Nombre del marketplace de Codex registrado que se seleccionará.                                   |
| `pluginName`                    | `computer-use` | Nombre del plugin en el marketplace de Codex.                                                 |
| `mcpServerName`                 | `computer-use` | Nombre del servidor MCP expuesto por el plugin instalado.                               |

La instalación automática al inicio del turno rechaza deliberadamente los valores configurados de `marketplaceSource`.
Añadir un origen nuevo es una operación de configuración explícita, por lo que se debe usar
`/codex computer-use install --source <marketplace-source>` una vez y, después, permitir que
`autoInstall` gestione las reactivaciones futuras desde los marketplaces locales detectados.
La instalación automática al inicio del turno puede usar un `marketplacePath` configurado, porque
ya es una ruta local del host.

Cada campo también acepta una sustitución mediante una variable de entorno, que se comprueba cuando la
clave de configuración correspondiente no está establecida:

| Campo                           | Variable de entorno                                                        |
| ------------------------------- | -------------------------------------------------------------- |
| `enabled`                       | `OPENCLAW_CODEX_COMPUTER_USE`                                  |
| `autoInstall`                   | `OPENCLAW_CODEX_COMPUTER_USE_AUTO_INSTALL`                     |
| `marketplaceDiscoveryTimeoutMs` | `OPENCLAW_CODEX_COMPUTER_USE_MARKETPLACE_DISCOVERY_TIMEOUT_MS` |
| `liveTestTimeoutMs`             | `OPENCLAW_CODEX_COMPUTER_USE_LIVE_TEST_TIMEOUT_MS`             |
| `toolCallTimeoutMs`             | `OPENCLAW_CODEX_COMPUTER_USE_TOOL_CALL_TIMEOUT_MS`             |
| `healthCheckEnabled`            | `OPENCLAW_CODEX_COMPUTER_USE_HEALTH_CHECK_ENABLED`             |
| `healthCheckIntervalMinutes`    | `OPENCLAW_CODEX_COMPUTER_USE_HEALTH_CHECK_INTERVAL_MINUTES`    |
| `pluginCacheMode`               | `OPENCLAW_CODEX_COMPUTER_USE_PLUGIN_CACHE_MODE`                |
| `strictReadiness`               | `OPENCLAW_CODEX_COMPUTER_USE_STRICT_READINESS`                 |
| `autoRepair`                    | `OPENCLAW_CODEX_COMPUTER_USE_AUTO_REPAIR`                      |
| `marketplaceSource`             | `OPENCLAW_CODEX_COMPUTER_USE_MARKETPLACE_SOURCE`               |
| `marketplacePath`               | `OPENCLAW_CODEX_COMPUTER_USE_MARKETPLACE_PATH`                 |
| `marketplaceName`               | `OPENCLAW_CODEX_COMPUTER_USE_MARKETPLACE_NAME`                 |
| `pluginName`                    | `OPENCLAW_CODEX_COMPUTER_USE_PLUGIN_NAME`                      |
| `mcpServerName`                 | `OPENCLAW_CODEX_COMPUTER_USE_MCP_SERVER_NAME`                  |

## Qué comprueba OpenClaw

OpenClaw informa internamente de un motivo de configuración estable y da formato al
estado visible para el usuario en el chat:

| Motivo                       | Significado                                                | Siguiente paso                                     |
| ---------------------------- | ------------------------------------------------------ | --------------------------------------------- |
| `disabled`                   | `computerUse.enabled` se resolvió como false.               | Establecer `enabled` u otro campo de Computer Use.  |
| `marketplace_missing`        | No había ningún marketplace coincidente disponible.                 | Configurar el origen, la ruta o el nombre del marketplace.  |
| `plugin_not_installed`       | El marketplace existe, pero el plugin no está instalado.   | Ejecutar la instalación o habilitar `autoInstall`.          |
| `plugin_disabled`            | El plugin está instalado, pero deshabilitado en la configuración de Codex.      | Ejecutar la instalación para volver a habilitarlo.                  |
| `remote_install_unsupported` | El marketplace seleccionado es exclusivamente remoto.                   | Usar `marketplaceSource` o `marketplacePath`. |
| `mcp_missing`                | El plugin está habilitado, pero el servidor MCP no está disponible.  | Comprobar Computer Use de Codex y los permisos del sistema operativo.  |
| `ready`                      | El plugin y las herramientas MCP están disponibles.                    | Iniciar el turno en modo Codex.                    |
| `check_failed`               | Una solicitud al app-server de Codex falló durante la comprobación de estado. | Comprobar la conectividad y los registros del app-server.       |
| `auto_install_blocked`       | La configuración al inicio del turno tendría que añadir un origen nuevo.       | Ejecutar primero la instalación explícita.                   |

La salida del chat incluye el estado del plugin, el estado del servidor MCP, el marketplace,
las herramientas cuando están disponibles y el mensaje específico del paso de configuración que falla.

## Permisos de macOS

Esta ruta de Computer Use propiedad de Codex se ejecuta en macOS, donde el servidor MCP puede necesitar
permisos locales del sistema operativo antes de poder inspeccionar o controlar aplicaciones. (Para el control
de escritorio multiplataforma en hosts Node con Windows y Linux, consultar el
[ejecutor cua-computer](/es/nodes/computer-use#windows-and-linux-experimental-via-cua-driver).)
Si OpenClaw indica que Computer Use está instalado, pero el servidor MCP no está disponible,
se debe comprobar primero la configuración de Computer Use en Codex:

- El app-server de Codex se ejecuta en el mismo host donde debe producirse el control del escritorio.
- El plugin de Computer Use está habilitado en la configuración de Codex.
- El servidor MCP `computer-use` aparece en el estado de MCP del app-server de Codex.
- macOS ha concedido los permisos necesarios para la aplicación de control del escritorio.
- La sesión actual del host puede acceder al escritorio que se está controlando.

OpenClaw aplica deliberadamente un cierre seguro cuando `computerUse.enabled` es true. Un
turno en modo Codex no debe continuar de forma silenciosa sin las herramientas nativas de escritorio
que exige la configuración.

## Solución de problemas

**El estado indica que no está instalado.** Ejecutar `/codex computer-use install`. Si no se
detecta el marketplace, proporcionar `--source` o `--marketplace-path`.

**El estado indica que está instalado, pero deshabilitado.** Ejecutar `/codex computer-use install`
de nuevo. La instalación del app-server de Codex vuelve a escribir la configuración del plugin como habilitada.

**El estado indica que la instalación remota no es compatible.** Usar un origen o una ruta
de marketplace local. Las entradas de catálogo exclusivamente remotas pueden inspeccionarse, pero no
instalarse mediante la API actual del app-server.

**El estado indica que el servidor MCP no está disponible.** Volver a ejecutar la instalación una vez para que se
recarguen los servidores MCP. Si continúa sin estar disponible, corregir la aplicación Computer Use de Codex,
el estado de MCP del app-server de Codex o los permisos de macOS.

**El estado o una comprobación agota el tiempo de espera en `computer-use.list_apps`.** El plugin y
el servidor MCP están presentes, pero el puente local de Computer Use no respondió.
Cerrar o reiniciar Computer Use de Codex, volver a iniciar Codex Desktop si es necesario y, después,
reintentar en una sesión nueva de OpenClaw. Si el host ejecutó anteriormente Computer Use
mediante un app-server de Codex administrado más antiguo, actualizar el plugin instalado desde
el marketplace incluido con la aplicación de escritorio (usar la ruta `Codex.app` para instalaciones
independientes de Codex Desktop):

```text
/codex computer-use install --source /Applications/ChatGPT.app/Contents/Resources/plugins/openai-bundled
```

**Una herramienta de Computer Use indica `Native hook relay unavailable`.** El
enlace de herramientas nativo de Codex no pudo acceder a un relé activo de OpenClaw mediante el
puente local ni la alternativa del Gateway. Iniciar una sesión nueva de OpenClaw con `/new`
o `/reset`. Si funciona una vez y vuelve a fallar en una llamada posterior a una herramienta,
`/new` solo está limpiando el intento actual; reiniciar el app-server de Codex o el
Gateway de OpenClaw para descartar los hilos y registros de enlaces antiguos y, después,
reintentar en una sesión nueva.

**La instalación automática al inicio del turno rechaza un origen.** Es intencionado. Añadir primero el
origen con `/codex computer-use install --source
<marketplace-source>` explícito y, después, la instalación automática al inicio de turnos futuros podrá usar el
marketplace local detectado.

## Contenido relacionado

- [Entorno de pruebas de Codex](/es/plugins/codex-harness)
- [Puente Peekaboo](/es/platforms/mac/peekaboo)
- [Aplicación para iOS](/es/platforms/ios)
