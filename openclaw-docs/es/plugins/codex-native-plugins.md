---
read_when:
    - Quieres que los agentes de OpenClaw en modo Codex usen plugins nativos de Codex
    - Está migrando plugins de Codex seleccionados por OpenAI e instalados desde el código fuente
    - Está configurando un plugin de Codex existente en el directorio del espacio de trabajo
    - Se están solucionando problemas de codexPlugins, el inventario de aplicaciones, las acciones destructivas o los diagnósticos de aplicaciones de plugins
summary: Configura plugins nativos de Codex para agentes de OpenClaw en modo Codex
title: Plugins nativos de Codex
x-i18n:
    generated_at: "2026-07-26T04:48:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0b1cfa39838d4dbd1f33a1e5b7f52faec4b033f9fa98ef5c029003177c2e27e5
    source_path: plugins/codex-native-plugins.md
    workflow: 16
---

La compatibilidad nativa con plugins de Codex permite que un agente de OpenClaw en modo Codex use las capacidades de aplicaciones y plugins propias de app-server de Codex dentro del mismo hilo de Codex que gestiona el turno de OpenClaw. Las llamadas a plugins permanecen en la transcripción nativa de Codex; app-server de Codex se encarga de la ejecución de MCP respaldada por aplicaciones. OpenClaw no convierte los plugins de Codex en herramientas dinámicas sintéticas `codex_plugin_*` de OpenClaw.

Use esta página después de que el [entorno de ejecución de Codex](/es/plugins/codex-harness) básico esté funcionando.

## Requisitos

- El entorno de ejecución del agente debe ser el entorno de ejecución nativo de Codex.
- `plugins.entries.codex.enabled` es `true`.
- `plugins.entries.codex.config.codexPlugins.enabled` es `true`.
- El app-server de Codex de destino puede ver el inventario esperado de marketplace, plugins y aplicaciones.
- La migración solo admite plugins `openai-curated` que haya detectado como instalados desde el código fuente en el directorio principal de Codex de origen.
- Los plugins `workspace-directory` configurados manualmente requieren un app-server de Codex cuyo `plugin/list` acepte `marketplaceKinds` y cuyos resúmenes de espacios de trabajo sin ruta incluyan `remotePluginId`. El plugin ya debe estar instalado y habilitado, y las aplicaciones que posee deben estar accesibles en `app/list`.

`codexPlugins` no tiene efecto en las ejecuciones del proveedor de OpenClaw, los enlaces de conversaciones ACP ni otros entornos de ejecución, porque esas rutas nunca crean hilos de app-server de Codex con configuración nativa de `apps`.

La cuenta de Codex del lado de OpenAI, la disponibilidad de aplicaciones y los controles de aplicaciones y plugins del espacio de trabajo proceden de la cuenta de Codex con la sesión iniciada. Consulte [Uso de Codex con su plan de ChatGPT](https://help.openai.com/en/articles/11369540-using-codex-with-your-chatgpt-plan) para conocer el modelo de cuentas y administración de OpenAI.

## Inicio rápido

Previsualice la migración desde el directorio principal de Codex de origen:

```bash
openclaw migrate codex --dry-run
```

Añada `--verify-plugin-apps` para que la migración llame al `app/list` de origen y exija que todas las aplicaciones propias estén presentes, habilitadas y accesibles antes de planificar la activación nativa:

```bash
openclaw migrate codex --dry-run --verify-plugin-apps
```

Aplique la migración cuando el plan sea correcto:

```bash
openclaw migrate apply codex --yes
```

La migración escribe entradas `codexPlugins` explícitas para los plugins aptos y llama a `plugin/install` de app-server de Codex para los plugins seleccionados. Una configuración migrada tiene este aspecto:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          codexPlugins: {
            enabled: true,
            allow_destructive_actions: true,
            plugins: {
              "google-calendar": {
                enabled: true,
                marketplaceName: "openai-curated",
                pluginName: "google-calendar",
              },
            },
          },
        },
      },
    },
  },
}
```

La migración sigue limitada a `openai-curated`. Para usar un plugin `workspace-directory` existente, añádalo manualmente con el `summary.id` exacto y calificado por marketplace que devuelve `plugin/list`. Por ejemplo, si Codex devuelve `example-plugin@workspace-directory`, configure ese valor completo en lugar de su nombre para mostrar:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          codexPlugins: {
            enabled: true,
            plugins: {
              "example-plugin": {
                enabled: true,
                marketplaceName: "workspace-directory",
                pluginName: "example-plugin@workspace-directory",
              },
            },
          },
        },
      },
    },
  },
}
```

OpenClaw no llama a `plugin/install` ni inicia la autenticación de un plugin `workspace-directory`. Instálelo, habilítelo y autentíquelo en Codex antes de añadir o habilitar la política de OpenClaw. OpenClaw mantiene ocultas las aplicaciones cuando la respuesta omite el marketplace exacto, el identificador del plugin, el identificador de detalles o las pruebas de disponibilidad de la aplicación. Si Codex rechaza la solicitud explícita de `plugin/list` del espacio de trabajo, OpenClaw informa de `marketplace_missing` para cada plugin habilitado del espacio de trabajo y mantiene disponibles los plugins seleccionados que se hayan detectado de forma independiente.

Después de un cambio en `codexPlugins`, las nuevas conversaciones de Codex incorporan automáticamente el conjunto actualizado de aplicaciones. Ejecute `/new` o `/reset` para actualizar la conversación actual. No es necesario reiniciar el Gateway para los cambios de habilitación o deshabilitación de plugins.

## Gestionar plugins desde el chat

`/codex plugins` inspecciona o cambia los plugins nativos de Codex configurados desde el mismo chat en el que se utiliza el entorno de ejecución de Codex:

```text
/codex plugins
/codex plugins list
/codex plugins disable google-calendar
/codex plugins enable google-calendar
```

`/codex plugins` es un alias de `/codex plugins list`. La lista muestra la clave, el estado activado/desactivado, el nombre del plugin de Codex y el marketplace de `plugins.entries.codex.config.codexPlugins.plugins` de cada plugin configurado.

`enable`/`disable` solo escriben en `~/.openclaw/openclaw.json`; nunca editan `~/.codex/config.toml` ni instalan nuevos plugins de Codex. Solo puede ejecutarlos el propietario o un cliente del Gateway con el ámbito `operator.admin`.

Al habilitar un plugin configurado también se activa el interruptor global `codexPlugins.enabled`. Si se escribió un plugin seleccionado como deshabilitado porque la migración devolvió `auth_required`, vuelva a autorizar la aplicación en Codex antes de habilitarla en OpenClaw. Para una entrada `workspace-directory`, habilitarla aquí solo cambia la política de OpenClaw; el plugin y la aplicación ya deben estar activos en Codex.

## Funcionamiento de la configuración nativa de plugins

La integración realiza un seguimiento de tres estados:

| Estado     | Significado                                                                                                                                        |
| ---------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| Instalado  | Codex tiene el paquete del plugin en el entorno de ejecución del app-server de destino.                                                            |
| Habilitado | Codex indica que el plugin está habilitado y la configuración de OpenClaw lo permite para los turnos del entorno de ejecución de Codex.            |
| Accesible  | El app-server de Codex confirma que las entradas de aplicaciones del plugin están disponibles para la cuenta activa y se corresponden con la identidad configurada del plugin. |

Para los plugins `openai-curated`, la migración es el paso persistente de instalación y determinación de aptitud:

- Durante la planificación, OpenClaw lee los detalles de `plugin/read` de Codex de origen y comprueba que la cuenta del app-server de Codex de origen sea una cuenta con suscripción a ChatGPT. Una respuesta que indique una cuenta que no sea de ChatGPT, o la ausencia de una respuesta de cuenta, omite los plugins respaldados por aplicaciones con `codex_subscription_required`.
- De forma predeterminada, la migración omite la llamada de origen a `app/list`: los plugins de origen respaldados por aplicaciones que superan la comprobación de la cuenta se planifican sin verificar la accesibilidad de las aplicaciones de origen, y los fallos de transporte de la consulta de la cuenta hacen que se omitan con `codex_account_unavailable`.
- Con `--verify-plugin-apps`, la migración toma una instantánea nueva de `app/list` de origen y exige que todas las aplicaciones propias estén presentes, habilitadas y accesibles antes de planificar la activación nativa. En ese caso, los fallos de transporte de la consulta de la cuenta pasan a la comprobación del inventario de aplicaciones de origen en lugar de provocar una omisión inmediata.

Para los plugins `workspace-directory`, la configuración se realiza fuera de OpenClaw. OpenClaw solo consulta ese marketplace cuando se ha configurado al menos una entrada habilitada del espacio de trabajo, resuelve cada plugin mediante el `summary.id` exacto y reutiliza las comprobaciones existentes de propiedad de `plugin/read` y disponibilidad de `app/list`. Un plugin no instalado, deshabilitado, inaccesible o sin autenticar no expone ninguna aplicación; OpenClaw no intenta instalarlo ni autenticarlo.

El inventario de aplicaciones en tiempo de ejecución es la comprobación de accesibilidad de la sesión de destino tanto para los plugins seleccionados migrados como para los plugins del espacio de trabajo configurados manualmente. La configuración de la sesión del entorno de ejecución de Codex calcula una configuración restrictiva de aplicaciones del hilo a partir de las aplicaciones habilitadas y accesibles de los plugins; no se vuelve a calcular en cada turno, por lo que `/codex plugins enable`/`disable` solo afectan a las nuevas conversaciones de Codex. Use `/new` o `/reset` para aplicar el cambio a la conversación actual.

## Límite de compatibilidad de V1

- Solo los plugins `openai-curated` ya instalados en el inventario del app-server de Codex de origen son aptos para la migración.
- El entorno de ejecución también admite entradas `workspace-directory` explícitas en compilaciones de app-server cuyo `plugin/list` implemente `marketplaceKinds` y devuelva `remotePluginId` para resúmenes de espacios de trabajo sin ruta. Estas entradas deben usar su `summary.id` exacto y calificado por marketplace, y ya deben estar instaladas, habilitadas y tener sus aplicaciones accesibles. Una solicitud rechazada de listado del espacio de trabajo produce el diagnóstico existente `marketplace_missing` por plugin; la ausencia de pruebas del marketplace, del plugin, de los detalles o de la aplicación no expone ninguna aplicación del espacio de trabajo. El inventario seleccionado de la solicitud de listado predeterminada sigue siendo utilizable.
- Los plugins de origen respaldados por aplicaciones deben superar la comprobación de suscripción durante la migración. `--verify-plugin-apps` añade la comprobación del inventario de aplicaciones de origen. Las cuentas que no superen la comprobación de suscripción y, en el modo de verificación, las aplicaciones de origen inaccesibles, deshabilitadas o ausentes, o los fallos al actualizar el inventario de aplicaciones, se notifican como elementos manuales omitidos en lugar de entradas de configuración habilitadas. Los detalles de plugins que no se puedan leer se omiten antes de la comprobación del inventario de aplicaciones.
- La migración escribe identidades explícitas de plugins (`marketplaceName` y `pluginName`); no escribe rutas de caché locales `marketplacePath`.
- `codexPlugins.enabled` es el único interruptor global de habilitación; no existe ningún comodín ni clave de configuración `plugins["*"]` que conceda autoridad de instalación arbitraria.
- Los marketplaces no seleccionados, los paquetes de plugins almacenados en caché, los hooks y los archivos de configuración de Codex se conservan en el informe de migración para su revisión manual, pero no se activan automáticamente. El entorno de ejecución acepta entradas `workspace-directory` configuradas manualmente; los demás marketplaces siguen sin ser compatibles.

## Inventario y propiedad de aplicaciones

OpenClaw lee el inventario de aplicaciones de Codex mediante `app/list` de app-server, lo almacena en memoria durante una hora y actualiza de forma asíncrona las entradas obsoletas o ausentes. La caché es local al proceso; reiniciar la CLI o el Gateway la elimina, y OpenClaw la reconstruye a partir de la siguiente lectura de `app/list`.

La migración y el entorno de ejecución usan claves de caché distintas:

- La verificación de la migración de origen usa el directorio principal y las opciones de inicio de Codex de origen. Solo se ejecuta con `--verify-plugin-apps` y fuerza un recorrido nuevo de `app/list` de origen para esa ejecución de planificación.
- La configuración del entorno de ejecución de destino usa la identidad del app-server de Codex del agente de destino al crear la configuración de aplicaciones del hilo. La activación de plugins seleccionados invalida esa clave de caché de destino y después fuerza su actualización tras `plugin/install`. La configuración de `workspace-directory` nunca ejecuta esta ruta de activación.

Una aplicación de plugin solo se expone cuando OpenClaw puede vincularla con el plugin configurado mediante una propiedad estable: un identificador exacto de la aplicación incluido en los detalles del plugin, un nombre conocido de servidor MCP o metadatos estables únicos. La propiedad basada únicamente en el nombre para mostrar o que sea ambigua se excluye hasta que la siguiente actualización del inventario demuestre la propiedad.

## Aplicaciones de cuentas conectadas

Los agentes gestionados por el propietario pueden aceptar todas las aplicaciones que ya estén conectadas a su cuenta de Codex sin requerir un paquete de plugin correspondiente:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          codexPlugins: {
            enabled: true,
            allow_all_plugins: true,
            allow_destructive_actions: "auto",
          },
        },
      },
    },
  },
}
```

`allow_all_plugins: true` toma una instantánea completa de `app/list` al establecerse un nuevo hilo nativo de Codex y solo admite las aplicaciones marcadas como accesibles para esa cuenta. No instala, autentica ni habilita aplicaciones globalmente. Los hilos existentes conservan su conjunto persistente de aplicaciones; use `/new`, `/reset` o reinicie el Gateway para incorporar las aplicaciones conectadas o revocadas recientemente.

Las aplicaciones de la cuenta heredan el valor global `codexPlugins.allow_destructive_actions`,
que acepta `true`, `false`, `"auto"` o `"ask"`. La política explícita por plugin
anula la política global para los identificadores de aplicaciones coincidentes. Los errores de inventario
aplican un cierre restrictivo en lugar de recurrir a un valor predeterminado sin restricciones.

## Configuración de aplicaciones del hilo

OpenClaw inyecta un parche restrictivo de `config.apps` para el hilo de Codex:
`_default` está deshabilitado y solo se habilitan las aplicaciones propiedad de plugins configurados y habilitados o
las aplicaciones de cuenta accesibles admitidas por `allow_all_plugins`.

El valor de `destructive_enabled` de cada aplicación procede de la política `allow_destructive_actions` global efectiva o
por plugin; `true`, `"auto"` y `"ask"`
establecen `destructive_enabled: true`, y `false` lo establece en `false`. Codex sigue
aplicando los metadatos de herramientas destructivas de las anotaciones de herramientas nativas de la aplicación.
`_default` se deshabilita con `open_world_enabled: false`; las aplicaciones de plugins habilitados
obtienen `open_world_enabled: true`. OpenClaw no expone una opción independiente
de política de mundo abierto a nivel de plugin ni mantiene listas de denegación
de nombres de herramientas destructivas por plugin.

El modo de aprobación de herramientas utiliza de forma predeterminada la aprobación automática para las aplicaciones admitidas, por lo que las herramientas
de lectura no destructivas se ejecutan sin solicitar aprobación en el mismo hilo. Las herramientas destructivas permanecen
controladas por la política `destructive_enabled` de cada aplicación.

## Política de acciones destructivas

Las solicitudes destructivas de los plugins se permiten de forma predeterminada para los plugins de Codex
configurados, mientras que los esquemas no seguros y la propiedad ambigua aplican un cierre restrictivo:

- El valor global `allow_destructive_actions` es `true` de forma predeterminada.
- El valor `allow_destructive_actions` por plugin anula la política global para
  ese plugin.
- `false`: OpenClaw devuelve un rechazo determinista.
- `true`: OpenClaw acepta automáticamente solo los esquemas seguros que puede asignar a una respuesta
  de aprobación, como un campo booleano de aprobación.
- `"auto"`: OpenClaw expone las acciones destructivas de los plugins a Codex y, después,
  convierte las solicitudes de aprobación de MCP cuya propiedad se ha demostrado en aprobaciones de
  plugins de OpenClaw antes de devolver la respuesta de aprobación de Codex.
- `"ask"`: OpenClaw utiliza el mismo control de escritura y acciones destructivas de Codex que
  `"auto"`, borra las anulaciones persistentes de aprobación por herramienta de Codex para la aplicación
  antes de que se inicie el hilo y solo ofrece aprobación o denegación para una única ocasión, de modo que
  las aprobaciones persistentes no puedan suprimir las solicitudes posteriores de acciones de escritura. Para cada
  aplicación admitida que utilice `"ask"`, OpenClaw selecciona el revisor de aprobaciones humanas de Codex
  para esa aplicación, de modo que Codex envíe sus solicitudes de aprobación a
  OpenClaw; las demás aplicaciones y las aprobaciones del hilo no relacionadas con aplicaciones conservan el
  revisor y la política configurados.
- Si falta la identidad del plugin, la propiedad es ambigua, falta el identificador
  del turno o no coincide, o el esquema de solicitud no es seguro, se rechaza la acción en lugar de solicitar aprobación.

## Solución de problemas

| Código                                            | Significado                                                                                                                                    | Solución                                                                                                                           |
| ------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| `auth_required`                                | La migración instaló el plugin, pero una de sus aplicaciones aún necesita autenticación. La entrada se escribe deshabilitada hasta que se vuelva a autorizar. | Vuelva a autorizar la aplicación en Codex y, después, habilite el plugin en OpenClaw.                                              |
| `app_inaccessible`, `app_disabled`, `app_missing` | Con `--verify-plugin-apps`, el inventario de aplicaciones de Codex de origen no mostraba todas las aplicaciones propias como presentes, habilitadas y accesibles. | Vuelva a autorizar o habilite la aplicación en Codex y, después, vuelva a ejecutar la migración con `--verify-plugin-apps`.            |
| `app_inventory_unavailable`                                | Se solicitó una verificación estricta de las aplicaciones de origen, pero falló la actualización del inventario de aplicaciones de Codex de origen. | Corrija el acceso al servidor de aplicaciones de Codex de origen o vuelva a intentarlo sin `--verify-plugin-apps` para aceptar el plan más rápido sujeto a la cuenta. |
| `codex_subscription_required`                                | La cuenta del servidor de aplicaciones de Codex de origen no era una cuenta con suscripción de ChatGPT.                                        | Inicie sesión en la aplicación de Codex con autenticación de suscripción y, después, vuelva a ejecutar la migración.               |
| `codex_account_unavailable`                                | No se pudo leer la cuenta del servidor de aplicaciones de Codex de origen.                                                                     | Corrija la autenticación del servidor de aplicaciones de Codex de origen o vuelva a ejecutar con `--verify-plugin-apps` para que el inventario de aplicaciones de origen determine la elegibilidad. |
| `marketplace_missing`, `plugin_missing`            | El marketplace o el plugin exacto no están disponibles; es posible que se haya rechazado la solicitud explícita del catálogo del espacio de trabajo; las aplicaciones del espacio de trabajo aplican un cierre restrictivo. | Verifique el contrato compatible del servidor de aplicaciones y el identificador exacto que se describen a continuación.          |
| `plugin_detail_unavailable`                                | OpenClaw no pudo leer los detalles de propiedad del plugin.                                                                                    | Inspeccione las respuestas `plugin/list` y `plugin/read` del servidor de aplicaciones de destino.                       |
| `plugin_disabled`                                | Codex indica que el plugin está instalado, pero deshabilitado.                                                                                 | La activación seleccionada puede corregirlo; habilite un plugin del espacio de trabajo en Codex antes de volver a intentarlo.      |
| `plugin_activation_failed`                                | La activación del plugin no se completó.                                                                                                       | Utilice el diagnóstico adjunto para distinguir entre errores del marketplace, de autenticación, de actualización o de preparación del espacio de trabajo. |
| `app_inventory_missing`, `app_inventory_stale`            | El estado de preparación de la aplicación procedía de una caché vacía u obsoleta.                                                              | OpenClaw programa automáticamente una actualización asíncrona; las aplicaciones de plugins permanecen excluidas hasta que se conozcan su propiedad y estado de preparación. |
| `app_ownership_ambiguous`                                | El inventario de aplicaciones solo encontró una coincidencia por nombre para mostrar.                                                         | La aplicación permanece oculta en el hilo de Codex hasta que una actualización posterior demuestre su propiedad.                   |

**El plugin del espacio de trabajo está instalado, pero no es visible:** confirme que el resultado
`plugin/list` del espacio de trabajo indique que el identificador configurado exacto está instalado y habilitado;
después, confirme que `app/list` indique que todas las aplicaciones propias son accesibles para la misma cuenta de
Codex. OpenClaw puede habilitar una aplicación accesible para el hilo incluso cuando el
inventario de la cuenta indique actualmente que esa aplicación está deshabilitada. Si se cambió ese estado después de que el Gateway almacenara en caché el inventario de
aplicaciones, espere a que se actualice la caché de una hora o reinicie el Gateway y, después, utilice
`/new` o `/reset`. OpenClaw no corrige ni autentica plugins del espacio de trabajo.
Si se rechaza la solicitud explícita de la lista del espacio de trabajo, cada entrada habilitada del espacio de trabajo
indica `marketplace_missing`; las entradas seleccionadas no relacionadas siguen procesándose
a partir de la respuesta de la lista predeterminada.

Para `plugin_detail_unavailable`, un resumen del espacio de trabajo sin ruta debe incluir
`remotePluginId`; OpenClaw mantiene ocultas las aplicaciones propias cuando ese selector o el
resultado posterior de `plugin/read` no están disponibles. Para
`plugin_activation_failed`, los plugins seleccionados pueden indicar un error del marketplace, de autenticación o
de actualización posterior a la instalación. Un plugin del espacio de trabajo indica este código cuando
aún no está activo; instálelo, habilítelo y autentíquelo fuera de OpenClaw.

**La configuración cambió, pero el agente no puede ver el plugin:** ejecute `/codex plugins
list` para confirmar el estado configurado y, después, `/new` o `/reset`. Los enlaces existentes de
hilos de Codex conservan la configuración de aplicaciones con la que se iniciaron hasta que OpenClaw
establezca una nueva sesión del entorno de ejecución o sustituya un enlace obsoleto.

**Se rechaza la acción destructiva:** compruebe los valores globales y por plugin de
`allow_destructive_actions`. Incluso con `true`, `"auto"` o `"ask"`,
los esquemas de solicitud no seguros y la identidad ambigua del plugin siguen aplicando un cierre restrictivo.

## Temas relacionados

- [Entorno de ejecución de Codex](/es/plugins/codex-harness)
- [Referencia del entorno de ejecución de Codex](/es/plugins/codex-harness-reference)
- [Runtime del entorno de ejecución de Codex](/es/plugins/codex-harness-runtime)
- [Referencia de configuración](/es/gateway/configuration-reference#codex-harness-plugin-config)
- [CLI de migración](/es/cli/migrate)
