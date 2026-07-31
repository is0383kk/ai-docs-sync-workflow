---
read_when:
    - Quieres añadir o eliminar cuentas de canales (Discord, Google Chat, iMessage, Matrix, Signal, Slack, Telegram, WhatsApp y más)
    - Se desea comprobar el estado del canal o seguir los registros del canal en tiempo real
    - Necesita inspeccionar o volver a enviar un evento entrante fallido del canal
summary: Referencia de la CLI para `openclaw channels` (cuentas, estado, mensajes no entregados, capacidades, resolución, registros, inicio/cierre de sesión)
title: Canales
x-i18n:
    generated_at: "2026-07-26T05:34:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8e5b7d674264af51d6fec34c8c95256129d66918b7c4515ac0f2c2bd311f2c3b
    source_path: cli/channels.md
    workflow: 16
---

# `openclaw channels`

Gestiona las cuentas de los canales de chat y su estado de ejecución en el Gateway.

Documentación relacionada:

- Guías de canales: [Canales](/es/channels)
- Configuración del Gateway: [Configuración](/es/gateway/configuration)

## Comandos comunes

```bash
openclaw channels list
openclaw channels list --all
openclaw channels status
openclaw channels capabilities
openclaw channels capabilities --channel discord --target channel:123
openclaw channels resolve --channel slack "#general" "@jane"
openclaw channels logs --channel all
openclaw channels dead-letters list --channel telegram --account default
```

`channels list` muestra únicamente los canales de chat: de forma predeterminada, las cuentas configuradas, con etiquetas de estado `installed`, `configured` y `enabled` por cuenta (`--json` para la salida legible por máquina). Pasa `--all` para mostrar también los canales incluidos que aún no tienen ninguna cuenta configurada y los canales del catálogo instalables que todavía no están en el disco. La autenticación del proveedor y el uso de modelos se gestionan en otros lugares: `openclaw models auth list` para los perfiles de autenticación del proveedor, y `openclaw status` o `openclaw models list` para el uso y la cuota.

## Estado, capacidades, resolución y registros

- `channels status`: `--channel <name>`, `--probe`, `--timeout <ms>` (valor predeterminado: `10000`), `--json`
- `channels capabilities`: `--channel <name>`, `--account <id>` (requiere `--channel`), `--target <dest>` (requiere `--channel`), `--timeout <ms>` (valor predeterminado: `10000`, con un límite máximo de `30000`), `--json`
- `channels resolve <entries...>`: `--channel <name>`, `--account <id>`, `--kind <auto|user|group>` (valor predeterminado: `auto`), `--json`
- `channels logs`: `--channel <name|all>` (valor predeterminado: `all`), `--lines <n>` (valor predeterminado: `200`), `--json`

`channels status --probe` es la ruta en vivo: en un Gateway accesible, ejecuta por cuenta las comprobaciones
`probeAccount` y, opcionalmente, `auditAccount`, por lo que la salida puede incluir el estado
del transporte y resultados de sondeo como `works`, `probe failed`, `audit ok` o `audit failed`.
Si no se puede acceder al Gateway, `channels status` recurre a resúmenes basados únicamente en la configuración
en lugar de mostrar la salida de sondeos en vivo.

## Cartas muertas entrantes

Los eventos entrantes que agotan su política de reintentos permanecen en la base de datos de estado compartida durante el período de retención existente para las entradas fallidas de la cola. Inspecciona una cuenta de canal con:

```bash
openclaw channels dead-letters list --channel telegram --account default
openclaw channels dead-letters list --channel telegram --account default --json
```

La vista de texto muestra los identificadores de eventos, los motivos de los fallos, el número de intentos y el tiempo transcurrido desde los fallos. La salida JSON también incluye la carga útil retenida, los metadatos, el carril y las marcas de tiempo de los intentos para fines de diagnóstico.

Después de corregir el problema subyacente, vuelve a poner en cola un evento con su identificador de evento original:

```bash
openclaw channels dead-letters resubmit <event-id> --channel telegram --account default
```

Ejecuta estos comandos en el host del Gateway para que accedan a la misma base de datos de estado compartida que el entorno de ejecución del canal. El reenvío conserva la carga útil, los metadatos y el carril, pero restablece el contador de intentos y la antigüedad en la cola. Sustituye de forma atómica el marcador de fallo de ese evento, por lo que, si se repite el comando mientras el evento está pendiente o reclamado, este se rechaza en lugar de crear un segundo envío. El canal en ejecución lo recoge durante su siguiente vaciado de entradas. Los eventos completados permanecen en estado terminal y no pueden reenviarse. Las filas fallidas creadas antes de que se añadiera la retención de la carga útil aún pueden aparecer en la lista, pero su reenvío se rechaza porque la carga útil no está disponible.

`openclaw health` informa del número de cartas muertas y de la antigüedad del fallo más antiguo por cuenta de canal. `openclaw doctor` identifica las cuentas afectadas y remite al comando de inspección.

No uses `openclaw sessions`, `sessions.list` del Gateway ni la herramienta
`sessions_list` del agente como indicador del estado del socket del canal. Esas superficies informan de
filas de conversaciones almacenadas, no del estado del entorno de ejecución del proveedor. Después de reiniciar
un proveedor de Discord, una cuenta conectada pero inactiva puede estar en buen estado aunque no aparezca ninguna
fila de sesión de Discord hasta el siguiente evento de conversación entrante o saliente.

## Añadir o eliminar cuentas

```bash
openclaw channels add --channel telegram --token <bot-token>
openclaw channels add --channel nostr --private-key "$NOSTR_PRIVATE_KEY"
openclaw channels remove --channel telegram --delete
```

<Tip>
`openclaw channels add telegram --help` o `openclaw channels add --channel telegram --help` muestra únicamente las opciones de configuración de Telegram. `openclaw channels add --help` muestra únicamente el contenedor de comandos compartido.
</Tip>

`channels remove` solo funciona con plugins de canal instalados o configurados. Usa primero `channels add` para los canales instalables del catálogo. Sin `--delete`, solicita desactivar la cuenta y conserva su configuración; `--delete` elimina las entradas de configuración sin pedir confirmación.
En los plugins de canal respaldados por un entorno de ejecución, `channels remove` también solicita al Gateway en ejecución que detenga la cuenta seleccionada antes de actualizar la configuración, de modo que desactivar o eliminar una cuenta no deje activo el receptor anterior hasta el reinicio.

El contenedor de control compartido solo contiene `--channel`, `--account` y el valor opcional `--name` para mostrar la cuenta. Cada plugin de canal moderno controla sus credenciales, su transporte y la semántica específica de su proveedor. Una vez seleccionado un canal mediante un identificador posicional o `--channel <id>`, la CLI crea únicamente las opciones de ese canal a partir de los metadatos del paquete del plugin incluido o instalado, sin cargar el código de ejecución del canal.

Las opciones de aspecto común, como `--token`, `--url` o `--use-env`, siguen perteneciendo al canal cuando las gestiona un contrato moderno. Cuando un plugin de terceros seleccionado todavía usa el adaptador de configuración compartido heredado, el núcleo registra únicamente el conjunto publicado de opciones de compatibilidad para ese canal, junto con su `cliAddOptions` heredado. Los campos heredados no relacionados no se filtran a otros canales, y un canal moderno seleccionado rechaza las opciones de compatibilidad que no haya declarado.

Estos son algunos ejemplos de opciones pertenecientes a canales:

| Canal       | Opciones                                                                                             |
| ----------- | ---------------------------------------------------------------------------------------------------- |
| Google Chat | `--webhook-path`, `--webhook-url`, `--audience-type`, `--audience`                                   |
| iMessage    | `--cli-path`, `--db-path`, `--service`, `--region`                                                   |
| Matrix      | `--homeserver`, `--user-id`, `--access-token`, `--password`, `--device-name`, `--initial-sync-limit` |
| Nostr       | `--private-key`, `--relay-urls`                                                                      |
| Signal      | `--signal-number`, `--signal-transport`, `--cli-path`, `--http-url`, `--http-host`, `--http-port`    |
| Tlon        | `--ship`, `--url`, `--code`, `--group-channels`, `--dm-allowlist`, `--auto-discover-channels`        |
| WhatsApp    | `--auth-dir`                                                                                         |

Si es necesario instalar un plugin de canal durante un comando de adición basado en opciones, OpenClaw usa la fuente de instalación predeterminada del canal sin abrir la solicitud interactiva de instalación del plugin.

Tanto la configuración guiada como la basada en opciones pasan por el analizador, la validación, la resolución de cuentas, el escritor de configuración y los enlaces posteriores a la escritura del canal seleccionado. Las opciones no compatibles generan el error de configuración del canal propietario, en lugar de aceptarse mediante un contenedor global de entradas.

Cuando se ejecuta `openclaw channels add` sin opciones directas de cuenta, credenciales o configuración del canal, el asistente interactivo puede solicitar información. Tanto un identificador posicional de canal como `--channel <id>` preseleccionan ese canal sin omitir las indicaciones:

```bash
openclaw channels add telegram
openclaw channels add --channel telegram
```

El asistente puede solicitar:

- identificadores de cuenta para cada canal seleccionado
- nombres visibles opcionales para esas cuentas
- `Route these channel accounts to agents now?`

Si se confirma la vinculación en ese momento, el asistente pregunta qué agente debe controlar cada cuenta de canal configurada y escribe vinculaciones de enrutamiento con ámbito de cuenta.

También se pueden gestionar más adelante las mismas reglas de enrutamiento mediante `openclaw agents bindings`, `openclaw agents bind` y `openclaw agents unbind` (consulta [agentes](/es/cli/agents)).

Al añadir una cuenta no predeterminada a un canal que todavía utiliza ajustes de nivel superior para una sola cuenta, OpenClaw traslada esos valores de nivel superior al mapa de cuentas del canal antes de escribir la nueva cuenta. El traslado reutiliza una cuenta con nombre existente cuando el canal tiene exactamente una o cuando `defaultAccount` apunta a una; de lo contrario, los valores se almacenan en `channels.<channel>.accounts.default`.

El comportamiento del enrutamiento se mantiene coherente:

- Las vinculaciones existentes exclusivas del canal (sin `accountId`) siguen coincidiendo con la cuenta predeterminada.
- `channels add` no crea ni reescribe automáticamente vinculaciones en el modo no interactivo.
- La configuración interactiva puede añadir opcionalmente vinculaciones con ámbito de cuenta.

Si la configuración ya estaba en un estado mixto (con cuentas con nombre y valores de nivel superior para una sola cuenta aún definidos), ejecuta `openclaw doctor --fix` para mover los valores con ámbito de cuenta a la cuenta trasladada elegida para ese canal.

## Inicio y cierre de sesión (interactivo)

```bash
openclaw channels login --channel whatsapp
openclaw channels logout --channel whatsapp
```

- `channels login` admite `--account <id>` y `--verbose`; `channels logout` admite `--account <id>`.
- `channels login` y `logout` pueden deducir el canal cuando solo un canal configurado admite esa acción; si hay varios, pasa `--channel`.
- `channels logout` da prioridad a la ruta en vivo del Gateway cuando está accesible, por lo que el cierre de sesión detiene cualquier receptor activo antes de borrar el estado de autenticación del canal. Si no se puede acceder a un Gateway local, recurre a la limpieza de autenticación local; con `gateway.mode: "remote"`, el error del Gateway hace que el comando falle.
- Después de iniciar sesión correctamente, la CLI solicita a un Gateway local accesible que inicie la cuenta; en el modo remoto, guarda la autenticación localmente e indica que el entorno de ejecución remoto no se ha reiniciado.
- Ejecuta `channels login` desde un terminal en el host del Gateway. La herramienta `exec` del agente bloquea este flujo de inicio de sesión interactivo; para iniciar sesión desde el chat, se deben usar las herramientas de inicio de sesión del agente nativas del canal, como `whatsapp_login`, cuando estén disponibles.

## Solución de problemas

- Ejecuta `openclaw status --deep` para realizar un sondeo amplio.
- Usa `openclaw doctor` para obtener correcciones guiadas.
- `openclaw channels status` recurre a resúmenes basados únicamente en la configuración cuando no se puede acceder al Gateway. Si las credenciales de un canal compatible se configuran mediante SecretRef, pero no están disponibles en la ruta del comando actual, informa de que esa cuenta está configurada e incluye notas sobre su estado degradado, en lugar de mostrarla como no configurada.

## Sondeo de capacidades

Obtén indicaciones sobre las capacidades del proveedor (intenciones y ámbitos cuando estén disponibles), además de la compatibilidad estática con las funciones:

```bash
openclaw channels capabilities
openclaw channels capabilities --channel discord --target channel:123
```

Notas:

- `--channel` es opcional; omítalo para enumerar todos los canales (incluidos los proporcionados por plugins).
- `--account` solo es válido con `--channel`.
- `--target` acepta `channel:<id>` o un id. numérico de canal sin procesar y solo se aplica a Discord. Para los canales de voz de Discord, la comprobación de permisos señala la ausencia de `ViewChannel`, `Connect`, `Speak`, `SendMessages` y `ReadMessageHistory`.
- Las comprobaciones son específicas del proveedor: identidad del bot de Discord + intents, además de permisos opcionales del canal; bot de Slack + ámbitos de usuario; indicadores del bot de Telegram + webhook; versión del daemon de Signal; token de aplicación de Microsoft Teams + roles/ámbitos de Graph (anotados cuando se conocen). Los canales sin comprobaciones indican `Probe: unavailable`.

## Resolver nombres a identificadores

Resuelva los nombres de canales/usuarios a identificadores mediante el directorio del proveedor:

```bash
openclaw channels resolve --channel slack "#general" "@jane"
openclaw channels resolve --channel discord "My Server/#support" "@someone"
openclaw channels resolve --channel matrix "Project Room"
```

Notas:

- Use `--kind user|group|auto` para forzar el tipo de destino.
- La resolución da preferencia a las coincidencias activas cuando varias entradas comparten el mismo nombre.
- `channels resolve` es de solo lectura. Si una cuenta seleccionada está configurada mediante SecretRef, pero esa credencial no está disponible en la ruta del comando actual, el comando devuelve resultados degradados sin resolver con notas en lugar de cancelar toda la ejecución.
- `channels resolve` no instala plugins de canal. Use `channels add --channel <name>` antes de resolver nombres para un canal instalable del catálogo.

## Contenido relacionado

- [Referencia de la CLI](/es/cli)
- [Descripción general de los canales](/es/channels)
