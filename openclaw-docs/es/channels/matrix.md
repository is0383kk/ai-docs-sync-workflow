---
read_when:
    - Configuración de Matrix en OpenClaw
    - Configuración del E2EE y la verificación de Matrix
summary: Estado de compatibilidad, configuración y ejemplos de configuración de Matrix
title: Matrix
x-i18n:
    generated_at: "2026-07-26T05:01:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: aa84c7d9d9019040a3fec3cfaabb78590006a4a2dd4bb95836f2cf37072777c5
    source_path: channels/matrix.md
    workflow: 16
---

Matrix es un plugin de canal descargable (`@openclaw/matrix`) basado en el `matrix-js-sdk` oficial. Admite mensajes directos, salas, hilos, contenido multimedia, reacciones, encuestas, ubicación y E2EE.

## Instalación

```bash
openclaw plugins install @openclaw/matrix
```

Las especificaciones de plugins sin calificar prueban primero ClawHub y, después, recurren a npm. Fuerce un origen con `openclaw plugins install clawhub:@openclaw/matrix` o `npm:@openclaw/matrix`. Desde un checkout local: `openclaw plugins install ./path/to/local/matrix-plugin`.

`plugins install` registra y habilita el plugin; no se necesita un paso `enable` independiente. El canal no hará nada hasta que se configure como se indica a continuación. Consulte [Plugins](/es/tools/plugin) para conocer las reglas generales de instalación.

## Configuración

1. Cree una cuenta de Matrix en su servidor doméstico.
2. Configure `channels.matrix` con `homeserver` + `accessToken`, o `homeserver` + `userId` + `password`.
3. Reinicie el Gateway.
4. Inicie un mensaje directo con el bot o invítelo a una sala. Las invitaciones nuevas solo se aceptan cuando [`autoJoin`](#auto-join) lo permite.

### Configuración interactiva

```bash
openclaw channels add
openclaw configure --section channels
```

El asistente solicita la URL del servidor doméstico, el método de autenticación (token o contraseña), el ID de usuario (solo para la autenticación con contraseña), un nombre de dispositivo opcional, si se debe habilitar E2EE y el acceso a salas/unión automática. Si ya existen variables de entorno `MATRIX_*` coincidentes y la cuenta no tiene una autenticación guardada, el asistente ofrece un método abreviado mediante variables de entorno. Resuelva los nombres de las salas antes de guardar una lista de permitidos con `openclaw channels resolve --channel matrix "Project Room"`. Al habilitar E2EE en el asistente, se ejecuta la misma inicialización que en [`openclaw matrix encryption setup`](#encryption-and-verification).

### Configuración mínima

Basada en token:

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      dm: { policy: "pairing" },
    },
  },
}
```

Basada en contraseña (el token se almacena en caché después del primer inicio de sesión):

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      userId: "@bot:example.org",
      password: "replace-me", // pragma: allowlist secret
      deviceName: "OpenClaw Gateway",
    },
  },
}
```

### Unión automática

El valor predeterminado de `channels.matrix.autoJoin` es `"off"`: el bot no aparecerá en salas o mensajes directos nuevos procedentes de invitaciones nuevas hasta que se una manualmente. OpenClaw no puede determinar en el momento de la invitación si se trata de un mensaje directo o un grupo, por lo que todas las invitaciones pasan primero por `autoJoin`; `dm.policy` solo se aplica después, una vez que el bot se ha unido y la sala se ha clasificado.

<Warning>
Configure `autoJoin: "allowlist"` junto con `autoJoinAllowlist` para restringir las invitaciones aceptadas, o `autoJoin: "always"` para aceptar todas las invitaciones.

`autoJoinAllowlist` solo acepta `!roomId:server`, `#alias:server` o `*`. Los nombres simples de salas se rechazan; los alias se resuelven mediante el servidor doméstico, no mediante el estado que afirma tener la sala de la invitación.
</Warning>

```json5
{
  channels: {
    matrix: {
      autoJoin: "allowlist",
      autoJoinAllowlist: ["!ops:example.org", "#support:example.org"],
      groups: {
        "!ops:example.org": { requireMention: true },
      },
    },
  },
}
```

### Formatos de destino de las listas de permitidos

- Mensajes directos (`dm.allowFrom`, `groupAllowFrom`, `groups.<room>.users`): use `@user:server`. Los nombres para mostrar se ignoran de forma predeterminada (son mutables); configure `dangerouslyAllowNameMatching: true` únicamente para ofrecer compatibilidad explícita con nombres para mostrar.
- Claves de la lista de permitidos de salas (`groups`, alias heredado `rooms`): use `!room:server` o `#alias:server`. Los nombres simples se ignoran a menos que se configure `dangerouslyAllowNameMatching: true`.
- Listas de permitidos de invitaciones (`autoJoinAllowlist`): use `!room:server`, `#alias:server` o `*`. Los nombres simples siempre se rechazan.

### Normalización del ID de cuenta

El asistente convierte un nombre descriptivo en un ID de cuenta normalizado (`Ops Bot` -> `ops-bot`). Los signos de puntuación se escapan en hexadecimal en los nombres de variables de entorno con ámbito para que no puedan producirse colisiones entre cuentas: `-` (0x2D) se convierte en `_X2D_`, por lo que `ops-prod` se asigna al prefijo de entorno `MATRIX_OPS_X2D_PROD_`.

### Credenciales almacenadas en caché

Matrix almacena en caché las credenciales de las cuentas en el estado compartido del plugin `state/openclaw.sqlite`. Cuando existen credenciales almacenadas en caché, OpenClaw considera que Matrix está configurado incluso sin un `accessToken` en el archivo de configuración; esto abarca la configuración, `openclaw doctor` y las comprobaciones del estado del canal. Durante las actualizaciones, se importan los archivos `~/.openclaw/credentials/matrix/credentials*.json` retirados mediante `openclaw doctor --fix`, se verifican las filas de SQLite y, después, se archivan los archivos.

### Variables de entorno

Variables de entorno respaldadas por claves de configuración, que se utilizan cuando la clave de configuración equivalente no está establecida. La cuenta predeterminada utiliza nombres sin prefijo; las cuentas con nombre insertan el token de la cuenta antes del sufijo (consulte [normalización](#account-id-normalization)).

| Cuenta predeterminada       | Cuenta con nombre (`<ID>` = token de la cuenta) |
| --------------------- | -------------------------------------- |
| `MATRIX_HOMESERVER`   | `MATRIX_<ID>_HOMESERVER`               |
| `MATRIX_ACCESS_TOKEN` | `MATRIX_<ID>_ACCESS_TOKEN`             |
| `MATRIX_USER_ID`      | `MATRIX_<ID>_USER_ID`                  |
| `MATRIX_PASSWORD`     | `MATRIX_<ID>_PASSWORD`                 |
| `MATRIX_DEVICE_ID`    | `MATRIX_<ID>_DEVICE_ID`                |
| `MATRIX_DEVICE_NAME`  | `MATRIX_<ID>_DEVICE_NAME`              |

Para la cuenta `ops`, los nombres pasan a ser `MATRIX_OPS_HOMESERVER`, `MATRIX_OPS_ACCESS_TOKEN` y así sucesivamente. `MATRIX_HOMESERVER` (y cualquier variante con ámbito `*_HOMESERVER`) no se puede establecer desde un `.env` del espacio de trabajo; consulte [Archivos `.env` del espacio de trabajo](/es/gateway/security).

<Note>
La clave de recuperación no es una variable de entorno respaldada por la configuración: OpenClaw nunca la lee directamente del entorno. El texto de orientación de la CLI sugiere canalizarla mediante una variable de shell denominada `MATRIX_RECOVERY_KEY` para la cuenta predeterminada, o `MATRIX_RECOVERY_KEY_<ID>` (ID de cuenta en mayúsculas sin formato, sin escape hexadecimal) para una cuenta con nombre; consulte [Verificar este dispositivo con una clave de recuperación](#verify-this-device-with-a-recovery-key).
</Note>

## Ejemplo de configuración

Una base práctica con emparejamiento de mensajes directos, lista de permitidos de salas y E2EE:

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      encryption: true,

      dm: {
        policy: "pairing",
        sessionScope: "per-room",
        threadReplies: "off",
      },

      groupPolicy: "allowlist",
      groupAllowFrom: ["@admin:example.org"],
      groups: {
        "!roomid:example.org": { requireMention: true },
      },

      autoJoin: "allowlist",
      autoJoinAllowlist: ["!roomid:example.org"],
      threadReplies: "inbound",
      replyToMode: "off",
      streaming: { mode: "partial" },
    },
  },
}
```

## Vistas previas en streaming

El streaming de respuestas de Matrix es opcional. `streaming.mode` controla cómo OpenClaw entrega la respuesta del asistente mientras está en curso; `streaming.block.enabled` controla si cada bloque completado se conserva como un mensaje de Matrix independiente.

```json5
{
  channels: {
    matrix: {
      streaming: { mode: "partial" },
    },
  },
}
```

Para conservar las vistas previas de respuestas en directo, pero ocultar las líneas provisionales de herramientas/progreso:

```json5
{
  channels: {
    matrix: {
      streaming: {
        mode: "partial",
        preview: {
          toolProgress: false,
        },
      },
    },
  },
}
```

La configuración completa acepta `{ mode, chunkMode, block, preview, progress }`:

```json5
{
  channels: {
    matrix: {
      streaming: {
        mode: "progress",
        progress: {
          label: "auto", // pick from configured or built-in labels (false to hide)
          labels: ["Thinking", "Writing", "Searching"], // candidates for label: "auto"
          maxLines: 8, // max rolling progress lines (default: 8)
          maxLineChars: 120, // max chars per line before truncation (default: 120)
          toolProgress: true, // show tool/progress activity (default: true)
        },
      },
    },
  },
}
```

- `progress.label`: etiqueta personalizada, `"auto"`/sin establecer para elegir una etiqueta configurada o integrada, o `false` para ocultarla.
- `progress.labels`: candidatos utilizados únicamente cuando `label` es `"auto"` o no está establecido.
- `progress.maxLines`: número máximo de líneas de progreso sucesivas que se conservan en el borrador; las líneas más antiguas que superen este límite se eliminan.
- `progress.maxLineChars`: número máximo de caracteres por línea compacta de progreso antes de truncarla.
- `progress.toolProgress`: cuando es `true` (valor predeterminado), la actividad de herramientas/progreso en directo aparece en el borrador.

| `streaming.mode`  | Comportamiento                                                                                                                                                 |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `"off"` (valor predeterminado) | Espera la respuesta completa y la envía una sola vez.                                                                                                                      |
| `"partial"`       | Edita un mensaje de texto normal en el mismo lugar mientras el modelo escribe el bloque actual. Los clientes estándar pueden notificar la primera vista previa, no la edición final.          |
| `"quiet"`         | Igual que `"partial"`, pero el mensaje es un aviso que no genera notificaciones. Los destinatarios reciben una notificación cuando una regla push por usuario coincide con la edición finalizada (consulte a continuación). |
| `"progress"`      | Envía líneas compactas de progreso individuales mediante un borrador de progreso.                                                                                          |

`streaming.block.enabled` (valor predeterminado `false`) es independiente de `streaming.mode`:

| `streaming.mode`        | `block.enabled: true`                                               | `block.enabled: false` (valor predeterminado)                     |
| ----------------------- | ------------------------------------------------------------------- | ---------------------------------------------------- |
| `"partial"` / `"quiet"` | Borrador en directo para el bloque actual; los bloques completados se conservan como mensajes | Borrador en directo para el bloque actual, finalizado en el mismo lugar |
| `"off"`                 | Un mensaje de Matrix con notificación por cada bloque terminado                     | Un mensaje de Matrix con notificación para la respuesta completa      |

Notas:

- Si una vista previa supera el límite de tamaño por evento de Matrix, OpenClaw detiene el streaming de la vista previa y recurre a entregar únicamente el contenido final.
- Las respuestas multimedia siempre envían los archivos adjuntos normalmente; si una vista previa obsoleta no se puede reutilizar de forma segura, OpenClaw la censura antes de enviar la respuesta multimedia final.
- Las actualizaciones de la vista previa del progreso de herramientas están activadas de forma predeterminada cuando el streaming de vistas previas está activo. Configure `streaming.preview.toolProgress: false` para conservar las ediciones de la vista previa del texto de la respuesta, pero mantener el progreso de las herramientas en la ruta de entrega normal.
- Las ediciones de las vistas previas requieren llamadas adicionales a la API de Matrix. Mantenga `streaming.mode: "off"` para obtener el perfil de límites de frecuencia más conservador.
- Los valores escalares/booleanos heredados de `streaming` y las claves planas `blockStreaming` / `chunkMode` se reescriben con esta estructura anidada mediante `openclaw doctor --fix`.

## Mensajes de voz

Las notas de voz entrantes de Matrix se transcriben antes del control de menciones de la sala, por lo que una nota de voz que diga el nombre del bot puede activar el agente en una sala `requireMention: true`, y el agente recibe la transcripción en lugar de únicamente un marcador de posición de archivo adjunto de audio.

Matrix utiliza el proveedor compartido de contenido multimedia de audio en `tools.media.audio`, como `gpt-4o-mini-transcribe` de OpenAI. Consulte [Descripción general de las herramientas multimedia](/es/tools/media-overview) para obtener información sobre la configuración y los límites del proveedor.

- Los eventos `m.audio` y los eventos `m.file` con un tipo MIME `audio/*` son aptos.
- En salas cifradas, OpenClaw descifra el archivo adjunto mediante la ruta de contenido multimedia de Matrix existente antes de la transcripción.
- La transcripción se marca como generada por una máquina y no confiable en el prompt del agente.
- El archivo adjunto se marca como ya transcrito para que las herramientas multimedia posteriores no vuelvan a transcribirlo.
- Establezca `tools.media.audio.enabled: false` para desactivar globalmente la transcripción de audio.

## Metadatos de aprobación

Los prompts de aprobación nativos de Matrix son eventos `m.room.message` normales con contenido específico de OpenClaw bajo la clave `com.openclaw.approval`. Los clientes estándar siguen mostrando el cuerpo de texto; los clientes compatibles con OpenClaw pueden leer el identificador de aprobación estructurado, el tipo, el estado, las decisiones y los detalles de ejecución o del plugin.

Cuando un prompt es demasiado largo para un solo evento de Matrix, OpenClaw divide el texto visible en fragmentos y adjunta `com.openclaw.approval` únicamente al primer fragmento. Las reacciones para permitir o denegar se vinculan a ese primer evento, por lo que los prompts largos mantienen el mismo destino de aprobación que los prompts de un solo evento.

### Reglas de push en alojamiento propio para vistas previas finalizadas silenciosas

`streaming.mode: "quiet"` solo notifica a los destinatarios una vez finalizado un bloque o turno; una regla de push por usuario debe coincidir con el marcador de vista previa finalizada. Consulte [Reglas de push de Matrix para vistas previas silenciosas](/es/channels/matrix-push-rules) para ver la receta completa.

## Salas de bot a bot

De forma predeterminada, se ignoran los mensajes de Matrix procedentes de otras cuentas de Matrix configuradas en OpenClaw. Use `allowBots` para permitir intencionadamente el tráfico entre agentes:

```json5
{
  channels: {
    matrix: {
      allowBots: "mentions", // true | "mentions"
      groups: {
        "!roomid:example.org": {
          requireMention: true,
        },
      },
    },
  },
}
```

- `allowBots: true` acepta mensajes de otras cuentas de bot de Matrix configuradas en las salas permitidas y en los mensajes directos.
- `allowBots: "mentions"` acepta esos mensajes solo cuando mencionan visiblemente a este bot en las salas; los mensajes directos siguen permitidos en cualquier caso.
- `groups.<room>.allowBots` reemplaza la configuración de la cuenta para una sala.
- Los mensajes aceptados de bots configurados usan la [protección compartida contra bucles de bots](/es/channels/bot-loop-protection). Configure `channels.defaults.botLoopProtection` y, a continuación, reemplácelo por cuenta con `channels.matrix.botLoopProtection` o por sala con `channels.matrix.groups.<room>.botLoopProtection`.
- OpenClaw sigue ignorando los mensajes del mismo identificador de usuario de Matrix para evitar bucles de autorrespuesta.
- Matrix no tiene un indicador de bot nativo; OpenClaw interpreta «escrito por un bot» como «enviado por otra cuenta de Matrix configurada en este Gateway de OpenClaw».

Use listas estrictas de salas permitidas y requisitos de mención al habilitar el tráfico de bot a bot en salas compartidas.

## Cifrado y verificación

En las salas cifradas (E2EE), los eventos de imagen salientes usan `thumbnail_file` para que las vistas previas de las imágenes se cifren junto con el archivo adjunto completo; las salas sin cifrar usan `thumbnail_url` sin cifrar. No se requiere ninguna configuración: el plugin detecta automáticamente el estado de E2EE.

Todos los comandos `openclaw matrix` aceptan `--verbose` (diagnósticos completos), `--json` (salida legible por máquina) y `--account <id>` (configuraciones con varias cuentas). La salida es concisa de forma predeterminada.

### Habilitar el cifrado

```bash
openclaw matrix encryption setup
printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix encryption setup --recovery-key-stdin
```

Inicializa el almacenamiento de secretos y la firma cruzada, crea una copia de seguridad de las claves de sala si es necesario y, a continuación, muestra el estado y los pasos siguientes. Indicadores útiles:

- `--recovery-key-stdin` lee una clave de recuperación desde la entrada estándar sin exponerla en los argumentos del proceso; `--recovery-key <key>` sigue disponible por compatibilidad
- `--force-reset-cross-signing` descarta la identidad de firma cruzada actual y crea una nueva (solo para uso intencionado)

Para una cuenta nueva, habilite E2EE al crearla:

```bash
openclaw matrix account add \
  --homeserver https://matrix.example.org \
  --access-token syt_xxx \
  --enable-e2ee
```

`--encryption` es un alias de `--enable-e2ee`. Configuración manual equivalente:

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_xxx",
      encryption: true,
      dm: { policy: "pairing" },
    },
  },
}
```

### Estado y señales de confianza

```bash
openclaw matrix verify status
openclaw matrix verify status --include-recovery-key --json
```

`verify status` informa de tres señales de confianza independientes (`--verbose` las muestra todas):

- `Locally trusted`: confiable únicamente para este cliente
- `Cross-signing verified`: el SDK informa de la verificación mediante firma cruzada
- `Signed by owner`: firmado con su propia clave de autofirma (solo para diagnóstico)

`Verified by owner` es `yes` únicamente cuando `Cross-signing verified` es `yes`; la confianza local o la firma de un propietario por sí solas no bastan.

`--allow-degraded-local-state` devuelve diagnósticos con el mejor esfuerzo posible sin preparar primero la cuenta de Matrix; resulta útil para comprobaciones sin conexión o con una configuración parcial.

### Verificar este dispositivo con una clave de recuperación

Pase la clave de recuperación mediante la entrada estándar en lugar de proporcionarla en la línea de comandos:

```bash
printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix verify device --recovery-key-stdin
```

El comando informa de tres estados:

- `Recovery key accepted`: Matrix aceptó la clave para el almacenamiento de secretos o la confianza del dispositivo.
- `Backup usable`: la copia de seguridad de las claves de sala puede cargarse con el material de recuperación confiable.
- `Device verified by owner`: este dispositivo tiene plena confianza en la identidad de firma cruzada de Matrix.

Finaliza con un código distinto de cero cuando la confianza plena en la identidad está incompleta, aunque la clave de recuperación haya desbloqueado el material de la copia de seguridad. En ese caso, complete la autoverificación desde otro cliente de Matrix:

```bash
openclaw matrix verify self
```

`verify self` espera a `Cross-signing verified: yes` antes de finalizar correctamente. Use `--timeout-ms <ms>` para ajustar la espera.

La forma de clave literal `openclaw matrix verify device "<recovery-key>"` también funciona, pero la clave queda guardada en el historial del shell.

### Inicializar o reparar la firma cruzada

```bash
openclaw matrix verify bootstrap
```

Es el comando de reparación y configuración para cuentas cifradas. En orden:

- inicializa el almacenamiento de secretos y reutiliza una clave de recuperación existente cuando es posible
- inicializa la firma cruzada y carga las claves públicas que faltan
- marca y firma de forma cruzada el dispositivo actual
- crea una copia de seguridad de las claves de sala en el servidor si todavía no existe

Si el homeserver requiere UIA para cargar las claves de firma cruzada, OpenClaw intenta primero sin autenticación, después `m.login.dummy` y, por último, `m.login.password` (requiere `channels.matrix.password`).

Indicadores útiles:

- `--recovery-key-stdin` (úselo junto con `printf '%s\n' "$MATRIX_RECOVERY_KEY" | ...`) o `--recovery-key <key>`
- `--force-reset-cross-signing` para descartar la identidad de firma cruzada actual (solo de forma intencionada; requiere que la clave de recuperación activa esté almacenada o se proporcione mediante `--recovery-key-stdin`)

### Copia de seguridad de las claves de sala

```bash
openclaw matrix verify backup status
printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix verify backup restore --recovery-key-stdin
```

`backup status` muestra si existe una copia de seguridad en el servidor y si este dispositivo puede descifrarla. `backup restore` importa las claves de sala respaldadas al almacén criptográfico local; omita `--recovery-key-stdin` si la clave de recuperación ya está en el disco.

Para reemplazar una copia de seguridad dañada por una base de referencia nueva (esto implica aceptar la pérdida del historial antiguo irrecuperable y también puede recrear el almacenamiento de secretos si el secreto de la copia de seguridad actual no puede cargarse):

```bash
openclaw matrix verify backup reset --yes
```

Añada `--rotate-recovery-key` únicamente cuando se pretenda que la clave de recuperación anterior deje de desbloquear la nueva base de referencia de la copia de seguridad.

### Enumerar, solicitar y responder a verificaciones

```bash
openclaw matrix verify list
```

Enumera las solicitudes de verificación pendientes de la cuenta seleccionada.

```bash
openclaw matrix verify request --own-user
openclaw matrix verify request --user-id @ops:example.org --device-id ABCDEF
```

Envía una solicitud de verificación desde esta cuenta. `--own-user` solicita la autoverificación (acepte el prompt en otro cliente de Matrix del mismo usuario); `--user-id`/`--device-id`/`--room-id` se dirigen a otra persona. `--own-user` no puede combinarse con los demás indicadores de destino.

Para gestionar el ciclo de vida a un nivel inferior —normalmente mientras se supervisan solicitudes entrantes de otro cliente—, estos comandos actúan sobre una solicitud `<id>` específica (mostrada por `verify list` y `verify request`):

| Comando                                    | Finalidad                                                             |
| ------------------------------------------ | ------------------------------------------------------------------- |
| `openclaw matrix verify accept <id>`       | Aceptar una solicitud entrante                                           |
| `openclaw matrix verify start <id>`        | Iniciar el flujo SAS                                                  |
| `openclaw matrix verify sas <id>`          | Mostrar los emojis o decimales de SAS                                     |
| `openclaw matrix verify confirm-sas <id>`  | Confirmar que SAS coincide con lo que muestra el otro cliente            |
| `openclaw matrix verify mismatch-sas <id>` | Rechazar SAS cuando los emojis o decimales no coincidan              |
| `openclaw matrix verify cancel <id>`       | Cancelar; acepta opcionalmente `--reason <text>` y `--code <matrix-code>` |

`accept`, `start`, `sas`, `confirm-sas`, `mismatch-sas` y `cancel` aceptan `--user-id` y `--room-id` como indicaciones de seguimiento por mensaje directo cuando la verificación está vinculada a una sala específica de mensajes directos.

### Notas sobre varias cuentas

Sin `--account <id>`, los comandos de la CLI de Matrix usan la cuenta predeterminada implícita. Si hay varias cuentas con nombre y no se proporciona `channels.matrix.defaultAccount`, los comandos se niegan a adivinar y solicitan que se elija una. Cuando E2EE está deshabilitado o no está disponible para una cuenta con nombre, los errores señalan la clave de configuración de esa cuenta, por ejemplo, `channels.matrix.accounts.assistant.encryption`.

<AccordionGroup>
  <Accordion title="Comportamiento durante el inicio">
    Con `encryption: true`, `startupVerification` usa `"if-unverified"` de forma predeterminada. Durante el inicio, un dispositivo no verificado solicita la autoverificación en otro cliente de Matrix, omite los duplicados y aplica un período de espera (24 horas de forma predeterminada). Ajústelo con `startupVerificationCooldownHours` o desactívelo con `startupVerification: "off"`.

    Durante el inicio también se ejecuta una inicialización criptográfica conservadora que reutiliza el almacenamiento de secretos y la identidad de firma cruzada actuales. Si el estado de inicialización está dañado, OpenClaw intenta realizar una reparación protegida incluso sin `channels.matrix.password`; si el homeserver requiere UIA con contraseña, el inicio registra una advertencia y el error no es fatal. Se conservan los dispositivos que ya están firmados por el propietario.

    Consulte [Migración de Matrix](/es/channels/matrix-migration) para ver el flujo de actualización completo.

  </Accordion>

  <Accordion title="Avisos de verificación">
    Matrix publica avisos sobre el ciclo de vida de la verificación en la sala estricta de verificación por mensajes directos como mensajes `m.notice`: solicitud, preparación (con instrucciones de «Verificar mediante emojis»), inicio/finalización y detalles de SAS (emojis/decimales) cuando están disponibles.

    Las solicitudes entrantes de otro cliente de Matrix se rastrean y aceptan automáticamente. Para la autoverificación, OpenClaw inicia automáticamente el flujo SAS y confirma su propio lado cuando está disponible la verificación mediante emojis; aún es necesario comparar y confirmar "They match" en el cliente de Matrix.

    Los avisos del sistema de verificación no se reenvían al pipeline de chat del agente.

  </Accordion>

  <Accordion title="Dispositivo de Matrix eliminado o no válido">
    Si `verify status` indica que el dispositivo actual ya no aparece en el homeserver, cree un nuevo dispositivo de Matrix para OpenClaw. Para iniciar sesión con contraseña:

```bash
openclaw matrix account add \
  --account assistant \
  --homeserver https://matrix.example.org \
  --user-id '@assistant:example.org' \
  --password '<password>' \
  --device-name OpenClaw-Gateway
```

    Para la autenticación con token, cree un token de acceso nuevo en el cliente de Matrix o en la interfaz de administración y, a continuación, actualice OpenClaw:

```bash
openclaw matrix account add \
  --account assistant \
  --homeserver https://matrix.example.org \
  --access-token '<token>'
```

    Sustituya `assistant` por el ID de cuenta del comando que falló u omita `--account` para usar la cuenta predeterminada.

  </Accordion>

  <Accordion title="Higiene de dispositivos">
    Los dispositivos antiguos administrados por OpenClaw pueden acumularse. Enumérelos y elimine los obsoletos:

```bash
openclaw matrix devices list
openclaw matrix devices prune-stale
```

  </Accordion>

  <Accordion title="Almacén criptográfico">
    El E2EE de Matrix usa la ruta criptográfica oficial de Rust `matrix-js-sdk` con `fake-indexeddb` como capa de compatibilidad con IndexedDB. El estado criptográfico se conserva en `crypto-idb-snapshot.json` (con permisos de archivo restrictivos).

    El estado cifrado del entorno de ejecución se encuentra en `~/.openclaw/matrix/accounts/<account>/<homeserver>__<user>/<token-hash>/` e incluye el almacén de sincronización, el almacén criptográfico, la clave de recuperación, la instantánea de IDB, las vinculaciones de hilos y el estado de verificación del inicio. Cuando el token cambia, pero la identidad de la cuenta permanece igual, OpenClaw reutiliza la mejor raíz existente para que el estado anterior siga visible.

    Una única raíz antigua basada en el hash del token puede ser una ruta normal de continuidad durante la rotación del token. Si OpenClaw registra `matrix: multiple populated token-hash storage roots detected`, inspeccione el directorio de la cuenta y archive las raíces hermanas obsoletas solo después de confirmar que la raíz activa seleccionada está en buen estado. Es preferible mover las raíces obsoletas a un directorio `_archive/` en lugar de eliminarlas inmediatamente.

  </Accordion>
</AccordionGroup>

## Administración del perfil

```bash
openclaw matrix profile set --name "OpenClaw Assistant"
openclaw matrix profile set --avatar-url https://cdn.example.org/avatar.png
```

Pase ambas opciones en una sola llamada. Matrix acepta directamente las URL de avatar `mxc://`; al pasar `http://`/`https://`, primero se carga el archivo y se almacena la URL `mxc://` resuelta en `channels.matrix.avatarUrl` (o en la anulación por cuenta).

## Hilos

Matrix admite hilos nativos tanto para respuestas automáticas como para envíos mediante la herramienta de mensajes. Dos controles independientes determinan el comportamiento:

### Enrutamiento de sesiones (`sessionScope`)

`dm.sessionScope` determina cómo se asignan las salas de mensajes directos de Matrix a las sesiones de OpenClaw:

- `"per-user"` (predeterminado): todas las salas de mensajes directos con el mismo interlocutor enrutado comparten una sesión.
- `"per-room"`: cada sala de mensajes directos de Matrix obtiene su propia clave de sesión, incluso para el mismo interlocutor.

Las vinculaciones explícitas de conversaciones siempre tienen prioridad sobre `sessionScope`; las salas y los hilos vinculados conservan la sesión de destino elegida.

### Respuestas en hilos (`threadReplies`)

`threadReplies` determina dónde publica el bot su respuesta:

- `"off"`: las respuestas se publican en el nivel superior. Los mensajes entrantes de hilos permanecen en la sesión principal.
- `"inbound"`: responde dentro de un hilo solo cuando el mensaje entrante ya estaba en ese hilo.
- `"always"`: responde dentro de un hilo cuya raíz es el mensaje desencadenante; esa conversación se enruta mediante una sesión correspondiente con ámbito de hilo desde el primer desencadenante en adelante.

`dm.threadReplies` anula este comportamiento solo para los mensajes directos; por ejemplo, permite mantener aislados los hilos de las salas mientras los mensajes directos permanecen sin hilos.

### Herencia de hilos y comandos con barra

- Los mensajes entrantes de hilos incluyen el mensaje raíz del hilo como contexto adicional para el agente.
- Los envíos mediante la herramienta de mensajes heredan automáticamente el hilo actual de Matrix cuando se dirigen a la misma sala (o al mismo usuario de mensajes directos), a menos que se proporcione explícitamente `threadId`.
- La reutilización del usuario de mensajes directos solo se activa cuando los metadatos de la sesión actual confirman el mismo interlocutor de mensajes directos en la misma cuenta de Matrix; de lo contrario, OpenClaw recurre al enrutamiento normal con ámbito de usuario.
- `/focus`, `/unfocus`, `/agents`, `/session idle`, `/session max-age` y el `/acp spawn` vinculado a un hilo funcionan en las salas y los mensajes directos de Matrix.
- El `/focus` de nivel superior crea un nuevo hilo de Matrix y lo vincula a la sesión de destino cuando `threadBindings.spawnSessions` está habilitado.
- Ejecutar `/focus` o `/acp spawn --thread here` dentro de un hilo existente de Matrix vincula ese hilo en el mismo lugar.

Cuando OpenClaw detecta que una sala de mensajes directos de Matrix entra en conflicto con otra sala de mensajes directos en la misma sesión compartida, publica una notificación única `m.notice` que señala la vía de escape `/focus` y sugiere cambiar `dm.sessionScope`. La notificación solo aparece cuando están habilitadas las vinculaciones de hilos.

## Vinculaciones de conversaciones ACP

Las salas, los mensajes directos y los hilos existentes de Matrix pueden convertirse en espacios de trabajo ACP persistentes sin cambiar la superficie de chat.

Flujo rápido para operadores:

- Ejecute `/acp spawn codex --bind here` dentro del mensaje directo, la sala o el hilo existente de Matrix para seguir usándolo.
- En un mensaje directo o una sala de nivel superior, el mensaje directo o la sala actuales permanecen como superficie de chat y los mensajes futuros se enrutan a la sesión ACP iniciada.
- Dentro de un hilo existente, `--bind here` vincula ese hilo en el mismo lugar.
- `/new` y `/reset` restablecen la misma sesión ACP vinculada en el mismo lugar.
- `/acp close` cierra la sesión ACP y elimina la vinculación.

`--bind here` no crea un hilo secundario de Matrix. `threadBindings.spawnSessions` controla `/acp spawn --thread auto|here`, cuando OpenClaw necesita crear o vincular un hilo secundario.

### Configuración de vinculaciones de hilos

Matrix hereda los valores predeterminados globales de `session.threadBindings` y admite anulaciones por canal:

- `threadBindings.enabled`
- `threadBindings.idleHours`
- `threadBindings.maxAgeHours`
- `threadBindings.spawnSessions`: controla la creación de hilos tanto para subagentes como para ACP.
- Las claves obsoletas `threadBindings.spawnSubagentSessions` / `threadBindings.spawnAcpSessions` se migran a `spawnSessions` mediante `openclaw doctor --fix`.
- `threadBindings.defaultSpawnContext`

La creación de sesiones vinculadas a hilos de Matrix está habilitada de forma predeterminada. Establezca `threadBindings.spawnSessions: false` para impedir que `/focus` y `/acp spawn --thread auto|here` de nivel superior creen o vinculen hilos de Matrix. Establezca `threadBindings.defaultSpawnContext: "isolated"` cuando la creación de hilos nativos de subagentes no deba bifurcar la transcripción principal.

## Reacciones

Matrix admite reacciones salientes, notificaciones de reacciones entrantes y reacciones de confirmación.

Las herramientas de reacciones salientes están controladas por `channels.matrix.actions.reactions`:

- `react` añade una reacción a un evento de Matrix.
- `reactions` enumera el resumen actual de reacciones de un evento de Matrix.
- `emoji=""` elimina las reacciones del propio bot en ese evento.
- `remove: true` elimina únicamente la reacción con el emoji especificado del bot.

**Orden de resolución** (prevalece el primer valor definido):

| Configuración                 | Orden                                                                               |
| ----------------------- | ----------------------------------------------------------------------------------- |
| `ackReaction`           | por cuenta -> canal -> `messages.ackReaction` -> emoji de reserva de la identidad del agente   |
| `ackReactionScope`      | por cuenta -> canal -> `messages.ackReactionScope` -> valor predeterminado `"group-mentions"` |
| `reactionNotifications` | por cuenta -> canal -> valor predeterminado `"own"`                                           |

`reactionNotifications: "own"` reenvía los eventos `m.reaction` añadidos cuando se dirigen a mensajes de Matrix creados por el bot; `"off"` deshabilita los eventos del sistema de reacciones. Las eliminaciones de reacciones no se sintetizan como eventos del sistema: Matrix las presenta como censuras, no como eliminaciones independientes de `m.reaction`.

## Contexto del historial

- `channels.matrix.historyLimit` controla cuántos mensajes recientes de la sala se incluyen como `InboundHistory` cuando un mensaje de la sala activa el agente. Recurre a `messages.groupChat.historyLimit`; el valor predeterminado efectivo es `0` si ninguno está configurado (deshabilitado).
- El historial de salas de Matrix se limita a la sala; los mensajes directos siguen usando el historial normal de la sesión.
- El historial de la sala solo incluye mensajes pendientes: OpenClaw almacena temporalmente los mensajes de la sala que aún no han activado una respuesta y, cuando llega una mención u otro desencadenante, crea una instantánea de esa ventana.
- El mensaje desencadenante actual no se incluye en `InboundHistory`; permanece en el cuerpo entrante principal de ese turno.
- Los reintentos del mismo evento de Matrix reutilizan la instantánea original del historial en lugar de avanzar hacia mensajes más recientes de la sala.

## Visibilidad del contexto

Matrix admite el control compartido `contextVisibility` para el contexto complementario de la sala, como el texto de respuestas recuperado, las raíces de hilos y el historial pendiente.

- `contextVisibility: "all"` es el valor predeterminado. El contexto complementario se conserva tal como se recibió.
- `contextVisibility: "allowlist"` filtra el contexto complementario para incluir solo remitentes permitidos por las comprobaciones activas de las listas de permitidos de salas y usuarios.
- `contextVisibility: "allowlist_quote"` se comporta como `allowlist`, pero conserva una respuesta citada explícita.

Esto afecta únicamente a la visibilidad del contexto complementario, no a si el propio mensaje entrante puede activar una respuesta. La autorización del desencadenante sigue procediendo de `groupPolicy`, `groups`, `groupAllowFrom` y la configuración de la política de mensajes directos.

## Política de mensajes directos y salas

```json5
{
  channels: {
    matrix: {
      dm: {
        policy: "allowlist",
        allowFrom: ["@admin:example.org"],
        threadReplies: "off",
      },
      groupPolicy: "allowlist",
      groupAllowFrom: ["@admin:example.org"],
      groups: {
        "!roomid:example.org": { requireMention: true },
      },
    },
  },
}
```

Para silenciar por completo los mensajes directos y mantener operativas las salas, establezca `dm.enabled: false`:

```json5
{
  channels: {
    matrix: {
      dm: { enabled: false },
      groupPolicy: "allowlist",
      groupAllowFrom: ["@admin:example.org"],
    },
  },
}
```

Consulte [Grupos](/es/channels/groups) para conocer el comportamiento del requisito de mención y las listas de permitidos.

Ejemplo de emparejamiento para mensajes directos de Matrix:

```bash
openclaw pairing list matrix
openclaw pairing approve matrix <CODE>
```

Si un usuario de Matrix no aprobado sigue enviando mensajes antes de la aprobación, OpenClaw reutiliza el mismo código de emparejamiento pendiente y puede enviar una respuesta de recordatorio después de un breve tiempo de espera, en lugar de generar un código nuevo.

Consulte [Emparejamiento](/es/channels/pairing) para conocer el flujo compartido de emparejamiento de mensajes directos y la disposición del almacenamiento.

## Reparación de salas directas

Si el estado de los mensajes directos se desincroniza, OpenClaw puede acabar con asignaciones `m.direct` obsoletas que apuntan a antiguas salas individuales en lugar de al mensaje directo activo. Inspeccione la asignación actual de un interlocutor:

```bash
openclaw matrix direct inspect --user-id @alice:example.org
```

Repárela:

```bash
openclaw matrix direct repair --user-id @alice:example.org
```

Ambos comandos aceptan `--account <id>` para configuraciones con varias cuentas. El flujo de reparación:

- da preferencia a un mensaje directo 1:1 estricto que ya esté asignado en `m.direct`
- recurre a cualquier mensaje directo 1:1 estricto al que se haya unido actualmente con ese usuario
- crea una sala directa nueva y vuelve a escribir `m.direct` si no existe ningún mensaje directo en buen estado

No elimina automáticamente las salas antiguas. Selecciona el mensaje directo en buen estado y actualiza la asignación para que los futuros envíos de Matrix, las notificaciones de verificación y otros flujos de mensajes directos se dirijan a la sala correcta.

## Aprobaciones de ejecución

Matrix puede actuar como cliente nativo de aprobación. Configure esta función en `channels.matrix.execApprovals` (o en `channels.matrix.accounts.<account>.execApprovals` para una anulación por cuenta):

- `enabled`: entrega las aprobaciones mediante solicitudes nativas de Matrix. Si no se establece o se usa `"auto"`, se habilita automáticamente cuando puede resolverse al menos un aprobador; establezca `false` para deshabilitarla explícitamente.
- `approvers`: ID de usuario de Matrix (`@owner:example.org`) autorizados para aprobar solicitudes de ejecución. Recurre a `channels.matrix.dm.allowFrom`.
- `target`: dónde se envían las solicitudes. `"dm"` (predeterminado) las envía a los mensajes directos de los aprobadores; `"channel"` las envía a la sala o al mensaje directo de origen; `"both"` las envía a ambos.
- `agentFilter` / `sessionFilter`: listas de permitidos opcionales que determinan qué agentes o sesiones activan la entrega mediante Matrix.

La autorización difiere ligeramente entre los tipos de aprobación:

- **Las aprobaciones de ejecución** usan `execApprovals.approvers` y, como alternativa, `dm.allowFrom`.
- **Las aprobaciones de Plugin** autorizan únicamente mediante `dm.allowFrom`.

Ambos tipos comparten los atajos de reacciones de Matrix y las actualizaciones de mensajes. Las personas encargadas de aprobar ven atajos de reacciones en el mensaje principal de aprobación:

- ✅ permitir una vez
- ❌ denegar
- ♾️ permitir siempre (cuando la política de ejecución efectiva lo permita)

Comandos de barra alternativos: `/approve <id> allow-once`, `/approve <id> allow-always`, `/approve <id> deny`.

Solo las personas identificadas como encargadas de aprobar pueden aprobar o denegar. La entrega al canal de las aprobaciones de ejecución incluye el texto del comando; solo se deben habilitar `channel` o `both` en salas de confianza.

Relacionado: [Aprobaciones de ejecución](/es/tools/exec-approvals).

## Comandos de barra

Los comandos de barra (`/new`, `/reset`, `/model`, `/focus`, `/unfocus`, `/agents`, `/session`, `/acp`, `/approve`, etc.) funcionan directamente en los mensajes directos. En las salas, OpenClaw también reconoce los comandos precedidos por la propia mención de Matrix del bot, por lo que `@bot:server /new` activa la ruta del comando sin una expresión regular personalizada para menciones; esto mantiene al bot receptivo a las publicaciones de estilo de sala `@mention /command` que emiten Element y clientes similares cuando una persona completa con el tabulador el nombre del bot antes de escribir el comando.

Las reglas de autorización siguen aplicándose: quienes envíen comandos deben cumplir las mismas políticas de lista de permitidos o de propietarios para mensajes directos o salas que los mensajes normales.

## Varias cuentas

```json5
{
  channels: {
    matrix: {
      enabled: true,
      defaultAccount: "assistant",
      dm: { policy: "pairing" },
      accounts: {
        assistant: {
          homeserver: "https://matrix.example.org",
          accessToken: "syt_assistant_xxx",
          encryption: true,
        },
        alerts: {
          homeserver: "https://matrix.example.org",
          accessToken: "syt_alerts_xxx",
          dm: {
            policy: "allowlist",
            allowFrom: ["@ops:example.org"],
            threadReplies: "off",
          },
        },
      },
    },
  },
}
```

**Herencia:**

- Los valores `channels.matrix` de nivel superior actúan como valores predeterminados para las cuentas con nombre, salvo que una cuenta los sobrescriba.
- Limite una entrada de sala heredada a una cuenta específica con `groups.<room>.account`. Las entradas sin `account` se comparten entre cuentas; `account: "default"` sigue funcionando cuando la cuenta predeterminada se configura en el nivel superior.

**Selección de la cuenta predeterminada:**

- Establezca `defaultAccount` para elegir la cuenta con nombre que prefieren el enrutamiento implícito, las comprobaciones y los comandos de la CLI.
- Si hay varias cuentas y una se llama literalmente `default`, OpenClaw la usa de forma implícita incluso cuando `defaultAccount` no está establecido.
- Si hay varias cuentas con nombre y no se ha seleccionado ninguna como predeterminada, los comandos de la CLI se niegan a hacer suposiciones; establezca `defaultAccount` o pase `--account <id>`.
- El bloque `channels.matrix.*` de nivel superior solo se considera la cuenta `default` implícita cuando su autenticación está completa (`homeserver` + `accessToken`, o `homeserver` + `userId` + `password`). Las cuentas con nombre siguen siendo detectables mediante `homeserver` + `userId` cuando las credenciales almacenadas en caché cubren la autenticación.

**Promoción:**

- Cuando OpenClaw promueve una configuración de una sola cuenta a varias cuentas durante la reparación o la configuración, conserva la cuenta con nombre existente si hay alguna o si `defaultAccount` ya apunta a una. Solo las claves de autenticación o inicialización de Matrix se trasladan a la cuenta promovida; las claves compartidas de políticas de entrega permanecen en el nivel superior.

Consulte la [Referencia de configuración](/es/gateway/config-channels#multi-account-all-channels) para conocer el patrón compartido de varias cuentas.

## Servidores domésticos privados o de LAN

De forma predeterminada, OpenClaw bloquea los servidores domésticos privados o internos de Matrix como protección contra SSRF, salvo que se habiliten explícitamente para cada cuenta.

Si el servidor doméstico se ejecuta en localhost, una IP de LAN/Tailscale o un nombre de host interno, habilite `network.dangerouslyAllowPrivateNetwork` para esa cuenta:

```json5
{
  channels: {
    matrix: {
      homeserver: "http://matrix-synapse:8008",
      network: {
        dangerouslyAllowPrivateNetwork: true,
      },
      accessToken: "syt_internal_xxx",
    },
  },
}
```

Ejemplo de configuración mediante la CLI:

```bash
openclaw matrix account add \
  --account ops \
  --homeserver http://matrix-synapse:8008 \
  --allow-private-network \
  --access-token syt_ops_xxx
```

Esta habilitación explícita solo permite destinos privados o internos de confianza. Los servidores domésticos públicos sin cifrado, como `http://matrix.example.org:8008`, permanecen bloqueados. Siempre que sea posible, se debe preferir `https://`.

## Tráfico de Matrix mediante proxy

Si el despliegue de Matrix necesita un proxy HTTP(S) saliente explícito, establezca `channels.matrix.proxy`:

```json5
{
  channels: {
    matrix: {
      homeserver: "https://matrix.example.org",
      accessToken: "syt_bot_xxx",
      proxy: "http://127.0.0.1:7890",
    },
  },
}
```

Las cuentas con nombre pueden sobrescribir el valor predeterminado de nivel superior mediante `channels.matrix.accounts.<id>.proxy`. OpenClaw usa la misma configuración de proxy para el tráfico de Matrix en tiempo de ejecución y las comprobaciones de estado de las cuentas.

## Resolución de destinos

Matrix acepta estas formas de destino en cualquier lugar donde OpenClaw solicite una sala o un usuario de destino:

- Usuarios: `@user:server`, `user:@user:server` o `matrix:user:@user:server`
- Salas: `!room:server`, `room:!room:server` o `matrix:room:!room:server`
- Alias: `#alias:server`, `channel:#alias:server` o `matrix:channel:#alias:server`

Los ID de sala de Matrix distinguen entre mayúsculas y minúsculas. Use exactamente las mismas mayúsculas y minúsculas del ID de sala de Matrix al configurar destinos de entrega explícitos, tareas cron, vinculaciones o listas de permitidos. OpenClaw mantiene canónicas las claves internas de sesión para el almacenamiento, por lo que esas claves en minúsculas no son una fuente fiable de ID de entrega de Matrix.

La consulta en vivo del directorio usa la cuenta de Matrix que ha iniciado sesión:

- Las búsquedas de usuarios consultan el directorio de usuarios de Matrix en ese servidor doméstico.
- Las búsquedas de salas aceptan directamente ID y alias explícitos de salas. La búsqueda por nombre entre las salas a las que se ha unido se realiza en la medida de lo posible y solo se aplica a las listas de permitidos de salas en tiempo de ejecución cuando se establece `dangerouslyAllowNameMatching: true`.
- Si el nombre de una sala no se puede resolver como un ID o alias, la resolución de la lista de permitidos en tiempo de ejecución lo ignora.

## Referencia de configuración

Los campos de usuario de tipo lista de permitidos (`groupAllowFrom`, `dm.allowFrom`, `groups.<room>.users`) aceptan ID completos de usuarios de Matrix (la opción más segura). Las entradas que no son ID se ignoran de forma predeterminada. Si se establece `dangerouslyAllowNameMatching: true`, las coincidencias exactas con nombres visibles del directorio de Matrix se resuelven al inicio y cada vez que cambia la lista de permitidos mientras el monitor está en ejecución; las entradas que no se puedan resolver se ignoran en tiempo de ejecución.

Las claves de la lista de permitidos de salas (`groups`, `rooms` heredada) deben ser ID o alias de salas. Las claves que sean nombres simples de salas se ignoran de forma predeterminada; `dangerouslyAllowNameMatching: true` restaura la búsqueda en la medida de lo posible entre los nombres de las salas a las que se ha unido.

### Cuenta y conexión

- `enabled`: habilita o deshabilita el canal.
- `name`: etiqueta visible opcional para la cuenta.
- `defaultAccount`: ID de cuenta preferido cuando hay varias cuentas de Matrix configuradas.
- `accounts`: sobrescrituras por cuenta con nombre. Los valores `channels.matrix` de nivel superior se heredan como valores predeterminados.
- `homeserver`: URL del servidor doméstico, por ejemplo, `https://matrix.example.org`.
- `network.dangerouslyAllowPrivateNetwork`: permite que esta cuenta se conecte a `localhost`, a IP de LAN/Tailscale o a nombres de host internos.
- `proxy`: URL de proxy HTTP(S) opcional para el tráfico de Matrix. Admite sobrescrituras por cuenta.
- `userId`: ID completo de usuario de Matrix (`@bot:example.org`).
- `accessToken`: token de acceso para la autenticación basada en tokens. Se admiten valores de texto sin formato y SecretRef mediante proveedores de entorno, archivo o ejecución ([Gestión de secretos](/es/gateway/secrets)).
- `password`: contraseña para el inicio de sesión basado en contraseña. Se admiten valores de texto sin formato y SecretRef.
- `deviceId`: ID explícito del dispositivo de Matrix.
- `deviceName`: nombre visible del dispositivo usado al iniciar sesión mediante contraseña.
- `avatarUrl`: URL almacenada del avatar propio para la sincronización del perfil y las actualizaciones de `profile set`.
- `initialSyncLimit`: número máximo de eventos recuperados durante la sincronización inicial.

### Cifrado

- `encryption`: habilita E2EE. Valor predeterminado: `false`.
- `startupVerification`: `"if-unverified"` (valor predeterminado cuando E2EE está activado) o `"off"`. Solicita automáticamente la autoverificación al inicio cuando este dispositivo no está verificado.
- `startupVerificationCooldownHours`: tiempo de espera antes de la siguiente solicitud automática al inicio. Valor predeterminado: `24`.

### Acceso y políticas

- `groupPolicy`: `"open"`, `"allowlist"` o `"disabled"`. Valor predeterminado: `"allowlist"`.
- `groupAllowFrom`: lista de ID de usuarios permitidos para el tráfico de salas.
- `mentionPatterns`: patrones de expresiones regulares con ámbito para las menciones en salas. Objeto con `{ mode: "allow"|"deny", allowIn: [roomId, ...], denyIn: [roomId, ...] }`. Controla si los valores `agents.entries.*.groupChat.mentionPatterns` configurados se aplican a cada sala.
- `dm.enabled`: cuando es `false`, ignora todos los mensajes directos. Valor predeterminado: `true`.
- `dm.policy`: `"pairing"` (valor predeterminado), `"allowlist"`, `"open"` o `"disabled"`. Se aplica después de que el bot se haya unido y haya clasificado la sala como un mensaje directo; no afecta al tratamiento de las invitaciones.
- `dm.allowFrom`: lista de ID de usuarios permitidos para el tráfico de mensajes directos.
- `dm.sessionScope`: `"per-user"` (valor predeterminado) o `"per-room"`.
- `dm.threadReplies`: sobrescritura exclusiva para mensajes directos del uso de hilos en las respuestas (`"off"`, `"inbound"`, `"always"`).
- `allowBots`: acepta mensajes de otras cuentas de bot de Matrix configuradas (`true` o `"mentions"`).
- `allowlistOnly`: cuando es `true`, fuerza todas las políticas activas de mensajes directos (excepto `"disabled"`) y las políticas de grupo `"open"` a `"allowlist"`. No cambia las políticas `"disabled"`.
- `dangerouslyAllowNameMatching`: cuando es `true`, permite la búsqueda en el directorio por nombres visibles de Matrix para las entradas de la lista de usuarios permitidos y la búsqueda por nombre de las salas a las que se ha unido para las claves de la lista de salas permitidas. Se deben preferir los ID completos `@user:server` y los ID o alias de salas.
- `autoJoin`: `"always"`, `"allowlist"` o `"off"`. Valor predeterminado: `"off"`. Se aplica a todas las invitaciones de Matrix, incluidas las invitaciones con formato de mensaje directo.
- `autoJoinAllowlist`: salas o alias permitidos cuando `autoJoin` es `"allowlist"`. Las entradas de alias se resuelven en el servidor doméstico, no mediante el estado declarado por la sala que envía la invitación.
- `contextVisibility`: visibilidad del contexto complementario (`"all"` de forma predeterminada, `"allowlist"`, `"allowlist_quote"`).

### Comportamiento de las respuestas

- `replyToMode`: `"off"` (predeterminado), `"first"`, `"all"` o `"batched"`.
- `threadReplies`: `"off"` (el valor predeterminado de nivel superior se resuelve como `"inbound"` salvo que se establezca explícitamente), `"inbound"` o `"always"`.
- `threadBindings`: anulaciones por canal para el enrutamiento y el ciclo de vida de sesiones vinculadas a hilos.
- `streaming`: objeto anidado `{ mode, chunkMode, block: { enabled, coalesce }, preview: { toolProgress }, progress: { label, labels, maxLines, maxLineChars, toolProgress } }`. `mode` es `"off"` (predeterminado), `"partial"`, `"quiet"` o `"progress"`. Las formas escalares/booleanas heredadas se migran mediante `openclaw doctor --fix`.
- `streaming.block.enabled`: cuando es `true`, los bloques completados del asistente se conservan como mensajes de progreso independientes. Valor predeterminado: `false`.
- `markdown`: configuración opcional de renderizado de Markdown para el texto saliente.
- `responsePrefix`: cadena opcional que se antepone a las respuestas salientes.
- `textChunkLimit`: tamaño de los fragmentos salientes en caracteres cuando `streaming.chunkMode: "length"`. Valor predeterminado: `4000`.
- `streaming.chunkMode`: `"length"` (predeterminado, divide según el número de caracteres) o `"newline"` (divide en los límites de línea).
- `historyLimit`: número de mensajes recientes de la sala incluidos como `InboundHistory` cuando un mensaje de la sala activa el agente. Recurre a `messages.groupChat.historyLimit`; valor predeterminado efectivo: `0` (deshabilitado).
- `mediaMaxMb`: límite de tamaño de los archivos multimedia en MB para los envíos salientes y el procesamiento entrante. Valor predeterminado: `20`.

### Configuración de reacciones

- `ackReaction`: anulación de la reacción de confirmación para este canal/cuenta.
- `ackReactionScope`: anulación del ámbito (`"group-mentions"` predeterminado, `"group-all"`, `"direct"`, `"all"`, `"none"`, `"off"`).
- `reactionNotifications`: modo de notificación de reacciones entrantes (`"own"` predeterminado, `"off"`).

### Herramientas y anulaciones por sala

- `actions`: control de acceso a herramientas por acción (`messages`, `reactions`, `pins`, `profile`, `memberInfo`, `channelInfo`, `verification`).
- `groups`: mapa de políticas por sala. La identidad de sesión utiliza el ID estable de la sala después de la resolución. (`rooms` es un alias heredado).
  - `groups.<room>.account`: restringe una entrada de sala heredada a una cuenta específica.
  - `groups.<room>.enabled`: opción por sala. Cuando es `false`, la sala se ignora como si no estuviera en el mapa.
  - `groups.<room>.requireMention`: anulación por sala del requisito de mención a nivel de canal.
  - `groups.<room>.allowBots`: anulación por sala de la configuración a nivel de canal (`true` o `"mentions"`).
  - `groups.<room>.botLoopProtection`: anulación por sala del presupuesto de protección contra bucles entre bots.
  - `groups.<room>.users`: lista de remitentes permitidos por sala.
  - `groups.<room>.tools`: anulaciones por sala para permitir/denegar herramientas.
  - `groups.<room>.autoReply`: anulación por sala del control mediante menciones. `true` deshabilita los requisitos de mención para esa sala; `false` vuelve a exigirlos.
  - `groups.<room>.skills`: filtro de Skills por sala.
  - `groups.<room>.systemPrompt`: fragmento del prompt del sistema por sala.

### Configuración de aprobación de ejecución

- `execApprovals.enabled`: entrega las aprobaciones de ejecución mediante solicitudes nativas de Matrix.
- `execApprovals.approvers`: ID de usuario de Matrix autorizados para aprobar. Recurre a `dm.allowFrom`.
- `execApprovals.target`: `"dm"` (predeterminado), `"channel"` o `"both"`.
- `execApprovals.agentFilter` / `execApprovals.sessionFilter`: listas opcionales de agentes/sesiones permitidos para la entrega.

## Contenido relacionado

- [Descripción general de los canales](/es/channels) - todos los canales compatibles
- [Emparejamiento](/es/channels/pairing) - autenticación por mensaje directo y flujo de emparejamiento
- [Grupos](/es/channels/groups) - comportamiento de los chats grupales y control mediante menciones
- [Enrutamiento de canales](/es/channels/channel-routing) - enrutamiento de sesiones para mensajes
- [Seguridad](/es/gateway/security) - modelo de acceso y refuerzo de seguridad
