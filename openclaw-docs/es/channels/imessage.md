---
read_when:
    - Configuración de la compatibilidad con iMessage
    - Depuración del envío y la recepción de iMessage
summary: Compatibilidad nativa con iMessage mediante imsg (JSON-RPC a través de stdio), con acciones de API privada para respuestas, reacciones tapback, efectos, encuestas, archivos adjuntos y gestión de grupos. Opción preferida para nuevas configuraciones de iMessage en OpenClaw cuando se cumplen los requisitos del host.
title: iMessage
x-i18n:
    generated_at: "2026-07-26T05:00:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f3e8b1a65c76b25d03615c06a976f86a8af555cd96d5bfdb10cef9c955893ddc
    source_path: channels/imessage.md
    workflow: 16
---

<Note>
Para la implementación habitual de iMessage con OpenClaw, ejecute el Gateway y `imsg` en el mismo host de macOS con sesión iniciada en Mensajes. Si el Gateway se ejecuta en otro lugar, configure `channels.imessage.cliPath` para que apunte a un wrapper SSH transparente que ejecute `imsg` en el Mac.

**La recuperación de mensajes entrantes es automática.** Después de reiniciar el puente o el Gateway, iMessage reproduce los mensajes perdidos mientras no estaba disponible y suprime la «avalancha de mensajes pendientes» obsoletos que Apple puede enviar después de una recuperación de Push, eliminando duplicados para que nada se procese dos veces. No hay ninguna configuración que habilitar; consulte [Recuperación de mensajes entrantes tras reiniciar el puente o el Gateway](#inbound-recovery-after-a-bridge-or-gateway-restart).
</Note>

<Warning>
Se eliminó la compatibilidad con BlueBubbles. Migre las configuraciones de `channels.bluebubbles` a `channels.imessage`; OpenClaw solo admite iMessage mediante `imsg`. Comience por [Eliminación de BlueBubbles y la ruta de iMessage con imsg](/es/announcements/bluebubbles-imessage) para consultar el anuncio breve, o por [Migración desde BlueBubbles](/es/channels/imessage-from-bluebubbles) para consultar la tabla de migración completa.
</Warning>

Estado: integración nativa con una CLI externa. El Gateway inicia `imsg rpc` y se comunica mediante JSON-RPC a través de la entrada y salida estándar, sin un daemon ni un puerto independientes. Se recomienda encarecidamente el modo de API privada para disponer de un canal de iMessage completo; las respuestas, los tapbacks, los efectos, las encuestas, las respuestas a archivos adjuntos y las acciones de grupo requieren `imsg launch` y una comprobación correcta de la API privada.

Para la configuración local habitual, el asistente de configuración de OpenClaw puede ofrecer una instalación o actualización de `imsg` mediante Homebrew confirmada por el usuario en el Mac con sesión iniciada en Mensajes. La configuración manual y las topologías con un wrapper SSH siguen estando a cargo del operador: instale o actualice `imsg` en el mismo contexto de usuario que ejecutará el Gateway o el wrapper.

<CardGroup cols={3}>
  <Card title="Acciones de la API privada" icon="wand-sparkles" href="#private-api-actions">
    Respuestas, tapbacks, efectos, encuestas, archivos adjuntos y gestión de grupos.
  </Card>
  <Card title="Emparejamiento" icon="link" href="/es/channels/pairing">
    Los mensajes directos de iMessage usan de forma predeterminada el modo de emparejamiento.
  </Card>
  <Card title="Mac remoto" icon="terminal" href="#remote-mac-over-ssh">
    Utilice un wrapper SSH cuando el Gateway no se ejecute en el Mac de Mensajes.
  </Card>
  <Card title="Referencia de configuración" icon="settings" href="/es/gateway/config-channels#imessage">
    Referencia completa de los campos de iMessage.
  </Card>
</CardGroup>

## Configuración rápida

<Tabs>
  <Tab title="Mac local (ruta rápida)">
    <Steps>
      <Step title="Instalar y verificar imsg">

```bash
brew install steipete/tap/imsg
brew update && brew upgrade imsg
imsg rpc --help
imsg launch
openclaw channels status --probe
```

        Cuando el asistente de configuración local detecta que falta el comando predeterminado `imsg`, puede solicitar la instalación de `steipete/tap/imsg` mediante Homebrew. Si detecta una instalación de `imsg` gestionada por Homebrew, puede solicitar que se reinstale o actualice. Los wrappers personalizados de `cliPath` no se modifican.

      </Step>

      <Step title="Configurar OpenClaw">

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "/usr/local/bin/imsg",
      dbPath: "/Users/user/Library/Messages/chat.db",
    },
  },
}
```

      </Step>

      <Step title="Iniciar el Gateway">

```bash
openclaw gateway
```

      </Step>

      <Step title="Aprobar el primer emparejamiento por mensaje directo (dmPolicy predeterminada)">

```bash
openclaw pairing list imessage
openclaw pairing approve imessage <CODE>
```

        Las solicitudes de emparejamiento caducan después de 1 hora.
      </Step>
    </Steps>

  </Tab>

  <Tab title="Mac remoto mediante SSH">
    La mayoría de las configuraciones no necesitan SSH. Utilice esta topología solo cuando el Gateway no pueda ejecutarse en el Mac con sesión iniciada en Mensajes. OpenClaw solo requiere un `cliPath` compatible con la entrada y salida estándar, por lo que puede configurar `cliPath` para que apunte a un script wrapper que se conecte mediante SSH a un Mac remoto y ejecute `imsg`.
    Instale y actualice `imsg` en ese Mac remoto, no en el host del Gateway:

```bash
ssh messages-mac 'brew install steipete/tap/imsg && brew update && brew upgrade imsg'
```

```bash
#!/usr/bin/env bash
exec ssh -T messages-mac imsg "$@"
```

    Configuración recomendada cuando los archivos adjuntos están habilitados:

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "~/.openclaw/scripts/imsg-ssh",
      remoteHost: "user@gateway-host", // se usa para obtener archivos adjuntos mediante SCP
      includeAttachments: true,
      // Opcional: raíces adicionales permitidas para archivos adjuntos (combinadas con la predeterminada
      // /Users/*/Library/Messages/Attachments).
      attachmentRoots: ["/Users/*/Library/Messages/Attachments"],
      remoteAttachmentRoots: ["/Users/*/Library/Messages/Attachments"],
    },
  },
}
```

    Si `remoteHost` no está configurado, OpenClaw intenta detectarlo automáticamente analizando el script wrapper SSH.
    `remoteHost` debe ser `host` o `user@host` (sin espacios ni opciones de SSH); los valores no seguros se ignoran.
    OpenClaw utiliza una comprobación estricta de la clave del host para SCP, por lo que la clave del host de retransmisión ya debe existir en `~/.ssh/known_hosts`.
    Las rutas de los archivos adjuntos se validan con respecto a las raíces permitidas (`attachmentRoots` / `remoteAttachmentRoots`).

<Warning>
Cualquier wrapper de `cliPath` o proxy SSH que coloque delante de `imsg` DEBE comportarse como una canalización transparente de entrada y salida estándar para JSON-RPC de larga duración. OpenClaw intercambia pequeños mensajes JSON-RPC delimitados por saltos de línea mediante la entrada y salida estándar del wrapper durante toda la vida útil del canal:

- Reenvíe cada fragmento o línea de la entrada estándar **en cuanto haya bytes disponibles**; no espere al final del archivo.
- Reenvíe rápidamente cada fragmento o línea de la salida estándar en sentido inverso.
- Conserve los saltos de línea.
- Evite las lecturas bloqueantes de tamaño fijo (`read(4096)`, `cat | buffer`, el `read` predeterminado del shell) que pueden privar de datos a los marcos pequeños.
- Mantenga la salida de errores separada del flujo de salida estándar de JSON-RPC.

Un wrapper que almacene la entrada estándar en un búfer hasta que se llene un bloque grande producirá síntomas similares a una interrupción de iMessage — `imsg rpc timeout (chats.list)` o reinicios reiterados del canal — aunque `imsg rpc` funcione correctamente. `ssh -T host imsg "$@"` (arriba) es seguro porque reenvía los argumentos de `cliPath` de OpenClaw, como `rpc` y `--db`. Las canalizaciones como `ssh host imsg | grep -v '^DEBUG'` NO son seguras: las herramientas con búfer por líneas aún pueden retener marcos; utilice `stdbuf -oL -eL` en cada etapa si debe aplicar un filtro.
</Warning>

  </Tab>
</Tabs>

## Requisitos y permisos (macOS)

- Se debe haber iniciado sesión en Mensajes en el Mac que ejecuta `imsg`.
- Se requiere acceso total al disco para el contexto de proceso que ejecuta OpenClaw/`imsg` (acceso a la base de datos de Mensajes).
- Se requiere permiso de automatización para enviar mensajes mediante Messages.app.
- Para las acciones avanzadas (reaccionar / editar / anular el envío / responder en un hilo / efectos / encuestas / operaciones de grupo), se debe deshabilitar la Protección de Integridad del Sistema; consulte [Habilitación de la API privada de imsg](#enabling-the-imsg-private-api). El envío y la recepción básicos de texto y contenido multimedia funcionan sin deshabilitarla.

<Tip>
Los permisos se conceden por contexto de proceso. Si el Gateway se ejecuta sin interfaz gráfica (LaunchAgent/SSH), ejecute una vez un comando interactivo en ese mismo contexto para activar las solicitudes de permisos:

```bash
imsg chats --limit 1
# o
imsg send <handle> "prueba"
```

</Tip>

<Accordion title="Los envíos mediante el wrapper SSH fallan con AppleEvents -1743">
  Una configuración mediante SSH remoto puede leer chats, superar `channels status --probe` y procesar mensajes entrantes, mientras que los envíos salientes continúan fallando con un error de autorización de AppleEvents:

```text
No se dispone de autorización para enviar eventos de Apple a Mensajes. (-1743)
```

Compruebe la base de datos TCC del usuario del Mac con sesión iniciada o System Settings > Privacy & Security > Automation. Si la entrada de automatización está registrada para `/usr/libexec/sshd-keygen-wrapper` en lugar de para `imsg` o el proceso del shell local, es posible que macOS no muestre un selector de Mensajes utilizable para ese cliente SSH del lado del servidor:

```text
kTCCServiceAppleEvents | /usr/libexec/sshd-keygen-wrapper | auth_value=0 | com.apple.MobileSMS
```

En ese estado, repetir `tccutil reset AppleEvents` o volver a ejecutar `imsg send` mediante el mismo wrapper SSH puede seguir fallando porque el contexto de proceso que necesita la automatización de Mensajes es el wrapper SSH, no una aplicación a la que la interfaz pueda conceder acceso.

Utilice en su lugar uno de los contextos de proceso de `imsg` compatibles:

- Ejecute el Gateway, o al menos el puente de `imsg`, en la sesión local del usuario que ha iniciado sesión en Mensajes.
- Inicie el Gateway mediante un LaunchAgent para ese usuario después de conceder acceso total al disco y automatización desde la misma sesión.
- Si mantiene la topología SSH de dos usuarios, compruebe que un envío saliente real mediante `imsg send` se complete correctamente a través del wrapper exacto antes de habilitar el canal. Si no se le puede conceder automatización, cambie la configuración a una instalación de `imsg` con un solo usuario en lugar de depender del wrapper SSH para los envíos.

</Accordion>

## Habilitación de la API privada de imsg

`imsg` incluye dos modos operativos. Para OpenClaw, el modo de API privada es la configuración recomendada porque proporciona al canal las acciones nativas de iMessage que los usuarios esperan. El modo básico sigue siendo útil para instalaciones de bajo riesgo, verificaciones iniciales o hosts en los que no se pueda deshabilitar SIP.

- **Modo básico** (predeterminado, no requiere cambios en SIP): texto y contenido multimedia salientes mediante `send`, supervisión e historial de mensajes entrantes y lista de chats. Esto es lo que se obtiene de forma inmediata con una instalación nueva de `brew install steipete/tap/imsg` y los permisos estándar de macOS indicados anteriormente.
- **Modo de API privada**: `imsg` inyecta una biblioteca auxiliar dinámica en `Messages.app` para llamar a funciones internas de `IMCore`. Esto habilita `react`, `edit`, `unsend`, `reply` (en hilos), `sendWithEffect`, `poll` y `poll-vote` (encuestas nativas de Mensajes), `renameGroup`, `setGroupIcon`, `addParticipant`, `removeParticipant`, `leaveGroup`, además de indicadores de escritura y confirmaciones de lectura.

La superficie de acciones recomendada en esta página requiere el modo de API privada. El archivo README de `imsg` indica explícitamente el requisito:

> Las funciones avanzadas, como `read`, `typing`, `launch`, el envío enriquecido mediante el puente, la modificación de mensajes y la gestión de chats, son opcionales. Requieren que SIP esté deshabilitado y que se inyecte una biblioteca auxiliar dinámica en `Messages.app`. `imsg launch` se niega a realizar la inyección cuando SIP está habilitado.

La técnica de inyección del componente auxiliar utiliza la propia biblioteca dinámica de `imsg` para acceder a las API privadas de Mensajes. La ruta de iMessage de OpenClaw no utiliza ningún servidor de terceros ni el entorno de ejecución de BlueBubbles.

<Warning>
**Deshabilitar SIP supone una contrapartida real en materia de seguridad.** SIP es una de las principales protecciones de macOS contra la ejecución de código del sistema modificado; deshabilitarlo en todo el sistema amplía la superficie de ataque y puede provocar efectos secundarios. En particular, **deshabilitar SIP en los Mac con Apple Silicon también deshabilita la capacidad de instalar y ejecutar aplicaciones de iOS en el Mac**.

Considérelo una decisión operativa deliberada, especialmente en un Mac personal principal. Para utilizar iMessage con OpenClaw con calidad de producción, es preferible usar un Mac dedicado o un usuario bot de macOS en el que resulte aceptable habilitar el puente. Si el modelo de amenazas no permite que SIP esté deshabilitado en ningún equipo, el iMessage incluido queda limitado al modo básico: solo envío y recepción de texto y contenido multimedia, sin reacciones / edición / anulación del envío / efectos / operaciones de grupo.
</Warning>

### Configuración

1. **Instale (o actualice) `imsg`** en el Mac que ejecuta Messages.app:

   ```bash
   brew install steipete/tap/imsg
   brew update && brew upgrade imsg
   imsg --version
   imsg status --json
   ```

   La salida de `imsg status --json` informa sobre `bridge_version`, `rpc_methods` y el valor de `selectors` de cada método, para que pueda ver qué admite la compilación actual antes de comenzar.

2. **Desactive la Protección de Integridad del Sistema y, en versiones modernas de macOS, la validación de bibliotecas.** Para inyectar una dylib auxiliar que no sea de Apple en el componente firmado por Apple `Messages.app`, es necesario desactivar SIP **y** relajar la validación de bibliotecas. El paso de SIP en el modo Recuperación depende de la versión de macOS:
   - **macOS 10.13-10.15 (Sierra-Catalina):** desactive la validación de bibliotecas mediante Terminal, reinicie en el modo Recuperación, ejecute `csrutil disable` y reinicie.
   - **macOS 11+ (Big Sur y posteriores), Intel:** acceda al modo Recuperación (o Recuperación por Internet), ejecute `csrutil disable` y reinicie.
   - **macOS 11+, Apple Silicon:** use la secuencia de inicio con el botón de encendido para acceder a Recuperación; en versiones recientes de macOS, mantenga pulsada la tecla **Left Shift** al hacer clic en Continue y, a continuación, ejecute `csrutil disable`. Las configuraciones de máquinas virtuales siguen un flujo distinto, por lo que primero debe crear una instantánea de la VM.

   **En macOS 11 y posteriores, `csrutil disable` por sí solo no suele ser suficiente.** Apple sigue aplicando la validación de bibliotecas a `Messages.app` como binario de plataforma, por lo que se rechaza un auxiliar con firma ad hoc (`Library Validation failed: ... platform binary, but mapped file is not`) incluso con SIP desactivado. Después de desactivar SIP, desactive también la validación de bibliotecas y reinicie:

   ```bash
   sudo defaults write /Library/Preferences/com.apple.security.libraryvalidation.plist DisableLibraryValidation -bool true
   ```

   **macOS 26 (Tahoe), verificado en 26.5.1:** desactivar SIP **junto con** el comando `DisableLibraryValidation` anterior basta para inyectar el auxiliar desde 26.0 hasta 26.5.x. **No se requieren argumentos de arranque.** El plist es el factor decisivo y el paso omitido con mayor frecuencia cuando falla la inyección en Tahoe:
   - **Con el plist:** `imsg launch` realiza la inyección y `imsg status` informa de `advanced_features: true`.
   - **Sin el plist (incluso con SIP desactivado):** `imsg launch` falla con `Failed to launch: Timeout waiting for Messages.app to initialize`. AMFI rechaza el auxiliar con firma ad hoc durante la carga, por lo que el puente nunca queda listo y el inicio agota el tiempo de espera. Ese tiempo de espera agotado es el síntoma que encuentra la mayoría de las personas en Tahoe; la solución es el plist anterior, no una medida más drástica.

   Si la inyección de `imsg launch` o determinados `selectors` comienzan a devolver falso después de una actualización de macOS, esta restricción suele ser la causa. Compruebe el estado de SIP y de la validación de bibliotecas antes de suponer que el propio paso de SIP ha fallado. Si esos ajustes son correctos y el puente todavía no puede realizar la inyección, recopile `imsg status --json` junto con la salida de `imsg launch` e informe al proyecto `imsg` en lugar de debilitar otros controles de seguridad de todo el sistema.

3. **Inyecte el auxiliar.** Con SIP desactivado y la sesión iniciada en Messages.app:

   ```bash
   imsg launch
   ```

   `imsg launch` se niega a realizar la inyección si SIP sigue activado, por lo que esto también sirve para confirmar que se completó el paso 2.

4. **Verifique el puente desde OpenClaw:**

   ```bash
   openclaw channels status --probe
   ```

   La entrada de iMessage debería informar de `works`, y `imsg status --json | jq '{rpc_methods, selectors}'` debería mostrar las capacidades expuestas por su compilación de macOS. La creación de encuestas requiere `selectors.pollPayloadMessage`; votar requiere tanto `selectors.pollVoteMessage` como el método RPC `poll.vote`. El Plugin de OpenClaw anuncia únicamente las acciones compatibles con la comprobación almacenada en caché, mientras que una caché vacía mantiene una actitud optimista y realiza la comprobación en el primer envío.

Si `openclaw channels status --probe` informa que el canal está `works`, pero determinadas acciones generan "iMessage `<action>` requires the imsg private API bridge" durante el envío, vuelva a ejecutar `imsg launch`; el auxiliar puede desconectarse (por un reinicio de Messages.app, una actualización del sistema operativo, etc.) y el estado `available: true` almacenado en caché seguirá anunciando acciones hasta que la siguiente comprobación lo actualice.

### Cuando SIP permanece activado

Si desactivar SIP no es aceptable para su modelo de amenazas:

- `imsg` recurre al modo básico: solo texto, contenido multimedia y recepción.
- El Plugin de OpenClaw sigue anunciando el envío de texto/contenido multimedia y la supervisión de mensajes entrantes; oculta `react`, `edit`, `unsend`, `reply`, `sendWithEffect` y las operaciones de grupo en la superficie de acciones (según la restricción de capacidades de cada método).
- Puede ejecutar un Mac independiente que no use Apple Silicon (o un Mac dedicado al bot) con SIP desactivado para la carga de trabajo de iMessage, mientras mantiene SIP activado en sus dispositivos principales. Consulte [Usuario de macOS dedicado al bot (identidad de iMessage independiente)](#deployment-patterns) más adelante.

## Control de acceso y enrutamiento

<Tabs>
  <Tab title="Política de mensajes directos">
    `channels.imessage.dmPolicy` controla los mensajes directos:

    - `pairing` (valor predeterminado)
    - `allowlist` (requiere al menos una entrada `allowFrom`)
    - `open` (requiere que `allowFrom` incluya `"*"`)
    - `disabled`

    Campo de lista de permitidos: `channels.imessage.allowFrom`.

    Las entradas de la lista de permitidos deben identificar a los remitentes: identificadores o grupos estáticos de acceso de remitentes (`accessGroup:<name>`). Use `channels.imessage.groupAllowFrom` para destinos de chat como `chat_id:*`, `chat_guid:*` o `chat_identifier:*`; use `channels.imessage.groups` para claves numéricas `chat_id` del registro.

  </Tab>

  <Tab title="Política de grupos y menciones">
    `channels.imessage.groupPolicy` controla la gestión de grupos:

    - `allowlist` (valor predeterminado)
    - `open`
    - `disabled`

    Lista de remitentes permitidos del grupo: `channels.imessage.groupAllowFrom`.

    Las entradas `groupAllowFrom` también pueden hacer referencia a grupos estáticos de acceso de remitentes (`accessGroup:<name>`).

    Comportamiento alternativo en tiempo de ejecución: si `groupAllowFrom` no está definido, las comprobaciones de remitentes de grupos de iMessage usan `allowFrom`; defina `groupAllowFrom` cuando la admisión de mensajes directos y de grupos deba ser diferente. Un `groupAllowFrom: []` explícitamente vacío no recurre al valor alternativo: bloquea a todos los remitentes de grupos bajo `allowlist`.
    Nota sobre el tiempo de ejecución: si `channels.imessage` falta por completo, el entorno de ejecución recurre a `groupPolicy="allowlist"` y registra una advertencia (incluso si `channels.defaults.groupPolicy` está definido).

    <Warning>
    El enrutamiento de grupos bajo `groupPolicy: "allowlist"` aplica **dos** restricciones consecutivas:

    1. **Lista de remitentes permitidos** (`channels.imessage.groupAllowFrom`): identificador, `accessGroup:<name>`, `chat_guid`, `chat_identifier` o `chat_id`. Una lista efectiva vacía (sin `groupAllowFrom` ni valor alternativo `allowFrom`) bloquea a todos los remitentes de grupos.
    2. **Registro de grupos** (`channels.imessage.groups`): se aplica cuando el mapa contiene entradas; el chat debe coincidir con una entrada explícita por `chat_id` o con el comodín `groups: { "*": { ... } }`. Cuando `groups` está vacío o no existe, la lista de remitentes permitidos determina por sí sola la admisión.

    Si no se configura ninguna lista efectiva de remitentes de grupos permitidos, todos los mensajes de grupo se descartan antes de la restricción del registro. Cada restricción tiene su propia señal de nivel `warn` en el nivel de registro predeterminado, y cada una indica una solución diferente:

    - una vez por cuenta al iniciar, cuando la lista efectiva de remitentes de grupos permitidos está vacía: `imessage: groupPolicy="allowlist" for account "<id>" but no group sender allowlist is configured ...`; corríjalo definiendo `channels.imessage.groupAllowFrom` (o `allowFrom`); añadir únicamente entradas `groups` hace que la restricción 1 siga bloqueando a todos los remitentes.
    - una vez por `chat_id` en tiempo de ejecución, cuando un remitente supera la restricción 1, pero el chat no aparece en un registro `groups` con entradas: `imessage: dropping group message from chat_id=<id> ...`; corríjalo añadiendo ese `chat_id` (o `"*"`) bajo `channels.imessage.groups`.

    Los mensajes directos no se ven afectados: siguen una ruta de código distinta.

    Configuración recomendada para el flujo de grupos bajo `groupPolicy: "allowlist"`:

    ```json5
    {
      channels: {
        imessage: {
          groupPolicy: "allowlist",
          groupAllowFrom: ["+15555550123"],
          groups: { "*": { "requireMention": true } },
        },
      },
    }
    ```

    `groupAllowFrom` por sí solo admite a esos remitentes en cualquier grupo; añada el bloque `groups` para delimitar qué chats están permitidos (y definir opciones por chat como `requireMention`).
    </Warning>

    Restricción de menciones para grupos:

    - iMessage no incluye metadatos nativos de menciones
    - la detección de menciones usa patrones de expresiones regulares (`agents.entries.*.groupChat.mentionPatterns`, con `messages.groupChat.mentionPatterns` como alternativa)
    - sin patrones configurados, no se puede aplicar la restricción de menciones
    - los comandos de control de remitentes autorizados omiten la restricción de menciones

    `systemPrompt` por grupo:

    Cada entrada bajo `channels.imessage.groups.*` acepta una cadena opcional `systemPrompt`, que se inyecta en el prompt del sistema del agente en cada turno que gestiona un mensaje de ese grupo. La resolución refleja `channels.whatsapp.groups`:

    1. **Prompt del sistema específico del grupo** (`groups["<chat_id>"].systemPrompt`): se usa cuando la entrada específica del grupo existe en el mapa **y** su clave `systemPrompt` está definida. Si `systemPrompt` es una cadena vacía (`""`), se suprime el comodín y no se aplica ningún prompt del sistema a ese grupo.
    2. **Prompt del sistema comodín para grupos** (`groups["*"].systemPrompt`): se usa cuando la entrada específica del grupo no está presente en el mapa o cuando existe, pero no define ninguna clave `systemPrompt`.

    ```json5
    {
      channels: {
        imessage: {
          groupPolicy: "allowlist",
          groupAllowFrom: ["+15555550123"],
          groups: {
            "*": { systemPrompt: "Use la ortografía británica." },
            "8421": {
              requireMention: true,
              systemPrompt: "Este es el chat de rotación de guardia. Limite las respuestas a menos de 3 frases.",
            },
            "9907": {
              // supresión explícita: el comodín "Use la ortografía británica." no se aplica aquí
              systemPrompt: "",
            },
          },
        },
      },
    }
    ```

    Los prompts por grupo solo se aplican a los mensajes de grupo; los mensajes directos no se ven afectados.

  </Tab>

  <Tab title="Sesiones y respuestas deterministas">
    - Los mensajes directos usan enrutamiento directo; los grupos usan enrutamiento de grupos.
    - Con el valor predeterminado `session.dmScope=main`, los mensajes directos de iMessage se consolidan en la sesión principal del agente.
    - Las sesiones de grupo están aisladas (`agent:<agentId>:imessage:group:<chat_id>`).
    - Las respuestas se devuelven a iMessage mediante los metadatos del canal y del destino de origen.

    Comportamiento de hilos similares a grupos:

    Algunos hilos de iMessage con varios participantes pueden llegar con `is_group=false`.
    Si ese `chat_id` está configurado explícitamente bajo `channels.imessage.groups`, OpenClaw lo trata como tráfico de grupo (restricciones de grupo y aislamiento de la sesión de grupo).

  </Tab>
</Tabs>

## Vinculaciones de conversaciones ACP

Los chats de iMessage pueden vincularse a sesiones ACP.

Flujo rápido para operadores:

- Ejecute `/acp spawn codex --bind here` dentro del mensaje directo o del chat de grupo permitido.
- Los mensajes futuros de esa misma conversación de iMessage se enrutarán a la sesión ACP creada.
- `/new` y `/reset` restablecen en el mismo lugar la sesión ACP vinculada.
- `/acp close` cierra la sesión ACP y elimina la vinculación.

Las vinculaciones persistentes configuradas usan entradas `bindings[]` de nivel superior con `type: "acp"` y `match.channel: "imessage"`.

`match.peer.id` puede usar:

- un identificador normalizado de mensaje directo, como `+15555550123` o `user@example.com`
- `chat_id:<id>` (recomendado para vinculaciones de grupos estables)
- `chat_guid:<guid>`
- `chat_identifier:<identifier>`

Ejemplo:

```json5
{
  agents: {
    list: [
      {
        id: "codex",
        runtime: {
          type: "acp",
          acp: { agent: "codex", backend: "acpx", mode: "persistent" },
        },
      },
    ],
  },
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "imessage",
        accountId: "default",
        peer: { kind: "group", id: "chat_id:123" },
      },
      acp: { label: "codex-group" },
    },
  ],
}
```

Consulte [Agentes ACP](/es/tools/acp-agents) para conocer el comportamiento compartido de las vinculaciones ACP.

## Patrones de implementación

<AccordionGroup>
  <Accordion title="Usuario de macOS dedicado al bot (identidad de iMessage independiente)">
    Use un Apple ID y un usuario de macOS dedicados para aislar el tráfico del bot de su perfil personal de Messages.

    Flujo habitual:

    1. Cree/inicie sesión con un usuario de macOS dedicado.
    2. Inicie sesión en Mensajes con el Apple ID del bot en ese usuario.
    3. Instale `imsg` en ese usuario.
    4. Cree un contenedor SSH para que OpenClaw pueda ejecutar `imsg` en el contexto de ese usuario.
    5. Dirija `channels.imessage.accounts.<id>.cliPath` y `.dbPath` a ese perfil de usuario.

    La primera ejecución puede requerir aprobaciones en la interfaz gráfica (Automatización + Acceso total al disco) en la sesión de ese usuario bot.

  </Accordion>

  <Accordion title="Mac remoto mediante Tailscale (ejemplo)">
    Topología habitual:

    - El gateway se ejecuta en Linux/VM
    - iMessage + `imsg` se ejecutan en un Mac de su tailnet
    - El contenedor `cliPath` usa SSH para ejecutar `imsg`
    - `remoteHost` habilita la obtención de archivos adjuntos mediante SCP

    Ejemplo:

    ```json5
    {
      channels: {
        imessage: {
          enabled: true,
          cliPath: "~/.openclaw/scripts/imsg-ssh",
          remoteHost: "bot@mac-mini.tailnet-1234.ts.net",
          includeAttachments: true,
          dbPath: "/Users/bot/Library/Messages/chat.db",
        },
      },
    }
    ```

    ```bash
    #!/usr/bin/env bash
    exec ssh -T bot@mac-mini.tailnet-1234.ts.net imsg "$@"
    ```

    Use claves SSH para que tanto SSH como SCP sean no interactivos.
    Asegúrese primero de que la clave del host sea de confianza (por ejemplo, `ssh bot@mac-mini.tailnet-1234.ts.net`) para que se rellene `known_hosts`.

  </Accordion>

  <Accordion title="Patrón de varias cuentas">
    iMessage admite configuración por cuenta en `channels.imessage.accounts`.

    Cada cuenta puede sustituir campos como `cliPath`, `dbPath`, `allowFrom`, `groupPolicy`, `mediaMaxMb`, la configuración del historial y las listas de permitidos de raíces de archivos adjuntos.

  </Accordion>

  <Accordion title="Historial de mensajes directos">
    Establezca `channels.imessage.dmHistoryLimit` para inicializar las nuevas sesiones de mensajes directos con el historial reciente de `imsg` decodificado de esa conversación. Use `channels.imessage.dms["<sender>"].historyLimit` para sustituciones por remitente, incluido `0` para deshabilitar el historial de un remitente.

    El historial de MD de iMessage se obtiene bajo demanda de `imsg`. Si `dmHistoryLimit` se deja sin establecer, se deshabilita la inicialización global del historial de MD, pero un valor positivo de `channels.imessage.dms["<sender>"].historyLimit` por remitente sigue habilitando la inicialización para ese remitente.

  </Accordion>
</AccordionGroup>

## Contenido multimedia, fragmentación y destinos de entrega

<AccordionGroup>
  <Accordion title="Archivos adjuntos y contenido multimedia">
    - la ingesta de archivos adjuntos entrantes está **desactivada de forma predeterminada**; establezca `channels.imessage.includeAttachments: true` para reenviar fotos, notas de voz, vídeos y otros archivos adjuntos al agente. Si está deshabilitada, los iMessages que solo contienen archivos adjuntos se descartan antes de llegar al agente y es posible que no generen ninguna línea de registro `Inbound message`.
    - las rutas remotas de archivos adjuntos se pueden obtener mediante SCP cuando se establece `remoteHost`
    - las rutas de archivos adjuntos deben coincidir con las raíces permitidas:
      - `channels.imessage.attachmentRoots` (local)
      - `channels.imessage.remoteAttachmentRoots` (modo SCP remoto)
      - las raíces configuradas amplían el patrón de raíz predeterminado `/Users/*/Library/Messages/Attachments` (se combinan, no se sustituyen)
    - SCP usa una comprobación estricta de la clave del host (`StrictHostKeyChecking=yes`)
    - el tamaño del contenido multimedia saliente usa `channels.imessage.mediaMaxMb` (valor predeterminado: 16 MB)

  </Accordion>

  <Accordion title="Texto saliente y fragmentación">
    - límite de fragmento de texto: `channels.imessage.textChunkLimit` (valor predeterminado: 4000)
    - modo de fragmentación: `channels.imessage.streaming.chunkMode`
      - `length` (valor predeterminado)
      - `newline` (división prioritaria por párrafos)
    - la negrita, cursiva, subrayado y tachado de Markdown saliente se convierten en texto con estilo nativo (los destinatarios con macOS 15+ ven el formato; los destinatarios con versiones anteriores ven texto sin formato y sin los marcadores); las tablas Markdown se convierten según el modo de tablas Markdown del canal
    - `channels.imessage.sendTransport` (`auto` de forma predeterminada, `bridge`, `applescript`) selecciona cómo `imsg` realiza los envíos

  </Accordion>

  <Accordion title="Formatos de direccionamiento">
    Destinos explícitos preferidos:

    - `chat_id:123` (recomendado para un enrutamiento estable)
    - `chat_guid:...`
    - `chat_identifier:...`

    También se admiten destinos de identificador:

    - `imessage:+1555...`
    - `sms:+1555...`
    - `user@example.com`

    ```bash
    imsg chats --limit 20
    ```

  </Accordion>
</AccordionGroup>

## Acciones de la API privada

Cuando `imsg launch` está en ejecución y `openclaw channels status --probe` informa de `privateApi.available: true`, la herramienta de mensajes puede usar acciones nativas de iMessage además de los envíos de texto normales.

Todas las acciones están habilitadas de forma predeterminada; use `channels.imessage.actions` para desactivar acciones individuales:

```json5
{
  channels: {
    imessage: {
      actions: {
        reactions: true,
        edit: true,
        unsend: true,
        reply: true,
        sendWithEffect: true,
        sendAttachment: true,
        renameGroup: true,
        setGroupIcon: true,
        addParticipant: true,
        removeParticipant: true,
        leaveGroup: true,
        polls: true,
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="Acciones disponibles">
    - **react**: Añade/elimina reacciones Tapback de iMessage (`messageId`, `emoji`, `remove`). Las reacciones Tapback admitidas corresponden a amor, me gusta, no me gusta, risa, énfasis y pregunta. Si se elimina sin un emoji, se borra cualquier reacción Tapback establecida.
    - **reply**: Envía una respuesta en un hilo a un mensaje existente (`messageId`, `text` o `message`, además de `chatGuid`, `chatId`, `chatIdentifier` o `to`). La respuesta con archivo adjunto también necesita una compilación de `imsg` cuyo `send-rich` admita `--file`.
    - **sendWithEffect**: Envía texto con un efecto de iMessage (`text` o `message`, `effect` o `effectId`). Nombres abreviados: slam, loud, gentle, invisibleink, confetti, lasers, fireworks, balloon, heart, echo, happybirthday, shootingstar, sparkles, spotlight.
    - **edit**: Edita un mensaje enviado en las versiones compatibles de macOS/API privada (`messageId`, `text` o `newText`). Solo se pueden editar los mensajes enviados por el propio gateway.
    - **unsend**: Retira un mensaje enviado en las versiones compatibles de macOS/API privada (`messageId`). Solo se pueden retirar los mensajes enviados por el propio gateway.
    - **upload-file**: Envía contenido multimedia/archivos (`buffer` como base64 o un `media`/`path`/`filePath` hidratado, `filename`, `asVoice` opcional). Alias heredado: `sendAttachment`.
    - **renameGroup**, **setGroupIcon**, **addParticipant**, **removeParticipant**, **leaveGroup**: Gestionan chats grupales cuando el destino actual es una conversación grupal. Estas acciones modifican la identidad de Mensajes del host, por lo que requieren un remitente propietario o un cliente de Gateway `operator.admin`.
    - **poll**: Crea una encuesta nativa de Mensajes de Apple (`pollQuestion`, `pollOption` repetido entre 2 y 12 veces, además de `chatGuid`, `chatId`, `chatIdentifier` o `to`). Los destinatarios con iOS/iPadOS/macOS 26+ pueden verla y votar de forma nativa; las versiones anteriores del sistema operativo reciben el texto alternativo "Se envió una encuesta". Requiere `selectors.pollPayloadMessage`.
    - **poll-vote**: Vota en una encuesta existente (`pollId` o `messageId`, además de exactamente uno de `pollOptionIndex`, `pollOptionId` o `pollOptionText`). Requiere `selectors.pollVoteMessage` y el método RPC `poll.vote`.

    Las encuestas entrantes aceptadas se presentan al agente con la pregunta, las etiquetas numeradas de las opciones, los recuentos de votos y el ID del mensaje de la encuesta que necesita `poll-vote`.

  </Accordion>

  <Accordion title="ID de mensajes">
    El contexto de iMessage entrante incluye tanto valores abreviados de `MessageSid` como GUID completos de mensajes (`MessageSidFull`), cuando están disponibles. Los ID abreviados se limitan a la caché reciente de respuestas respaldada por SQLite y se comprueban con el chat actual antes de usarlos. Si caduca un ID abreviado, vuelva a intentarlo con su `MessageSidFull` y use como destino la conversación que lo proporcionó. Los ID completos no eluden la vinculación con la conversación o la cuenta, así que sustituya un ID de otro chat por uno del destino actual. Las llamadas delegadas remotas pueden rechazar ID completos obsoletos cuando no hay pruebas disponibles de la conversación actual.

  </Accordion>

  <Accordion title="Detección de capacidades">
    OpenClaw oculta las acciones de la API privada solo cuando el estado almacenado en caché del sondeo indica que el puente no está disponible. Si el estado es desconocido, las acciones permanecen visibles y su ejecución inicia los sondeos de forma diferida para que la primera acción pueda completarse correctamente después de `imsg launch` sin una actualización manual independiente del estado.

  </Accordion>

  <Accordion title="Confirmaciones de lectura e indicador de escritura">
    Cuando el puente de la API privada está activo, los chats entrantes aceptados se marcan como leídos y los chats directos muestran una burbuja de escritura en cuanto se acepta el turno, mientras el agente prepara el contexto y genera la respuesta. Deshabilite el marcado como leído con:

    ```json5
    {
      channels: {
        imessage: {
          sendReadReceipts: false,
        },
      },
    }
    ```

    Las compilaciones antiguas de `imsg` anteriores a la lista de capacidades por método desactivan silenciosamente el indicador de escritura y las confirmaciones de lectura; OpenClaw registra una advertencia única por reinicio para que se pueda atribuir la ausencia de la confirmación.

  </Accordion>

  <Accordion title="Reacciones Tapback entrantes">
    OpenClaw se suscribe a las reacciones Tapback de iMessage y enruta las reacciones aceptadas como eventos del sistema en lugar de texto de mensaje normal, por lo que una reacción Tapback de un usuario no activa un bucle de respuesta normal.

    El modo de notificación se controla mediante `channels.imessage.reactionNotifications`:

    - `"own"` (valor predeterminado): notifica solo cuando los usuarios reaccionan a mensajes creados por el bot.
    - `"all"`: notifica todas las reacciones Tapback entrantes de remitentes autorizados.
    - `"off"`: ignora las reacciones Tapback entrantes.

    Las sustituciones por cuenta usan `channels.imessage.accounts.<id>.reactionNotifications`.

  </Accordion>

  <Accordion title="Reacciones de aprobación (👍 / 👎)">
    Cuando `approvals.exec.enabled` o `approvals.plugin.enabled` es true y la solicitud se enruta a iMessage, el gateway entrega una solicitud de aprobación de forma nativa y acepta una reacción Tapback para resolverla:

    - `👍` (reacción Tapback Me gusta) → `allow-once`
    - `👎` (reacción Tapback No me gusta) → `deny`
    - `allow-always` sigue siendo una alternativa manual: envíe `/approve <id> allow-always` como respuesta normal.

    El procesamiento de reacciones requiere que el identificador del usuario que reacciona sea un aprobador explícito. La lista de aprobadores se lee de `channels.imessage.allowFrom` (o `channels.imessage.accounts.<id>.allowFrom`); añada el número de teléfono del usuario en formato E.164 o el correo electrónico de su Apple ID (los destinos de chat como `chat_id:*` no son entradas de aprobador válidas). La entrada comodín `"*"` se respeta, pero permite que cualquier remitente apruebe; una lista de aprobadores vacía deshabilita por completo el acceso directo mediante reacciones. El acceso directo mediante reacciones omite intencionadamente `reactionNotifications`, `dmPolicy` y `groupAllowFrom`, porque la lista de permitidos de aprobadores explícitos es el único control relevante para resolver la aprobación.

    La autorización del comando de texto `/approve` sigue la misma lista: cuando `channels.imessage.allowFrom` no está vacío, `/approve <id> <decision>` se autoriza según esa lista de aprobadores (no según la lista de permitidos de MD más amplia), y los remitentes permitidos en la lista de permitidos de MD pero que no estén en `allowFrom` reciben una denegación explícita. Cuando `allowFrom` está vacío, la alternativa del mismo chat continúa vigente y `/approve` autoriza a cualquier persona permitida por la lista de permitidos de MD. Añada a todos los operadores que deban aprobar —mediante `/approve` o mediante reacciones— a `allowFrom`.

    Notas para operadores:
    - La vinculación de la reacción se almacena tanto en memoria como en el almacén persistente con claves del Gateway (con un TTL que coincide con el vencimiento de la aprobación), y el Gateway también consulta periódicamente las solicitudes pendientes en busca de tapbacks, por lo que un tapback que llegue poco después de reiniciar el Gateway seguirá resolviendo la aprobación.
    - El tapback `is_from_me=true` del propio operador (por ejemplo, desde un dispositivo Apple enlazado) resuelve la aprobación cuando ese identificador es un aprobador explícito.
    - Las solicitudes de aprobación solo se enrutan a una conversación grupal cuando se configuran aprobadores explícitos; de lo contrario, cualquier miembro del grupo podría aprobar.
    - Los tapbacks heredados con estilo de texto (`Liked "…"` como texto sin formato de clientes Apple muy antiguos) no pueden resolver aprobaciones porque no incluyen un GUID de mensaje; la resolución mediante reacciones requiere los metadatos estructurados de tapback que emiten los clientes actuales de macOS / iOS.

  </Accordion>

  <Accordion title="Reacciones a preguntas (1️⃣ / 2️⃣ / 3️⃣ / 4️⃣)">
    Para una solicitud `ask_user` con una pregunta no secreta, de selección única y entre una y cuatro opciones, OpenClaw añade opciones de emojis numerados. Reaccione a la solicitud entregada con el número correspondiente para responderla. La reacción debe incluir el GUID estable del mensaje creado por el bot; a continuación, OpenClaw asigna el número a la opción canónica mediante el Gateway. Los toques obsoletos o duplicados se ignoran.

    Las solicitudes con varias preguntas, selección múltiple o texto libre siguen admitiendo únicamente respuestas de texto. Las reacciones a preguntas siguen las reglas habituales de admisión de mensajes directos y grupos de iMessage. Se reconocen incluso cuando el valor general `reactionNotifications` es `"off"`, sin convertir reacciones no relacionadas en eventos del agente.

  </Accordion>
</AccordionGroup>

## Escrituras de configuración

iMessage permite de forma predeterminada las escrituras de configuración iniciadas por el canal (para `/config set|unset` cuando `commands.config: true`).

Para desactivarlas:

```json5
{
  channels: {
    imessage: {
      configWrites: false,
    },
  },
}
```

<a id="coalescing-split-send-dms-command--url-in-one-composition"></a>

## Combinación de mensajes directos enviados por separado (comando + URL en una sola composición)

Apple puede almacenar un comando y la vista previa de su URL como filas físicas `chat.db` independientes. `imsg` 0.13.1 y versiones posteriores combinan esas filas antes de que la supervisión, el historial o la búsqueda devuelvan el mensaje, por lo que OpenClaw recibe un único mensaje entrante lógico sin añadir latencia específica del canal a los mensajes directos.

No se necesita ninguna opción de combinación de iMessage. `openclaw doctor --fix` elimina la clave retirada `channels.imessage.coalesceSameSenderDms`. El retraso genérico `messages.inbound` sigue disponible cuando se desea agrupar intencionadamente mensajes de texto rápidos en un canal.

Si los envíos de comando más URL llegan como turnos separados del agente, actualice `imsg` en el Mac con Mensajes:

```bash
brew update && brew upgrade imsg
```

## Recuperación de mensajes entrantes tras reiniciar un puente o el Gateway

iMessage recupera los mensajes perdidos mientras el Gateway estaba inactivo y, al mismo tiempo, suprime la antigua «avalancha de mensajes pendientes» que Apple puede liberar tras recuperar Push. El comportamiento predeterminado está siempre activado y se basa en una entrada duradera y un límite de antigüedad.

- **Protección duradera contra repeticiones.** Antes de avanzar el cursor de recuperación, OpenClaw registra cada fila sin procesar en la cola de entrada SQLite compartida, con su GUID de Apple como identificador del evento. Una fila completada conserva una marca de eliminación durante unas 4 horas, con un límite de 10,000 entradas, por lo que una repetición con el mismo GUID se descarta incluso después de un reinicio. Una fila pendiente sigue siendo recuperable hasta que el envío la adopta.
- **Recuperación tras un periodo de inactividad.** Al iniciarse, el monitor recuerda el último rowid de `chat.db` admitido de forma duradera (un cursor persistente por cuenta) y lo pasa a `imsg watch.subscribe` como `since_rowid`, de modo que imsg repite las filas que aún no se habían registrado y luego continúa siguiendo los eventos en directo. Las filas registradas antes de un fallo se reanudan desde SQLite. La repetición se limita a las 500 filas más recientes y a mensajes de hasta ~2 horas de antigüedad, y las marcas de eliminación por GUID descartan todo lo ya procesado.
- **Límite de antigüedad para mensajes pendientes obsoletos.** Las filas posteriores al límite de inicio son realmente eventos en directo; si la fecha de envío de una fila es más de ~15 minutos anterior a su llegada, se considera parte de los mensajes pendientes liberados por Push y se suprime. Las filas repetidas (en el límite o antes de él) utilizan en cambio la ventana de recuperación más amplia, por lo que se entrega un mensaje perdido recientemente, pero no el historial antiguo.

La recuperación funciona tanto con configuraciones `cliPath` locales como remotas, porque la repetición de `since_rowid` se ejecuta mediante la misma conexión RPC `imsg`. La diferencia es la ventana: cuando el Gateway puede leer `chat.db` (en local), fija el límite inicial de rowid, limita el intervalo de repetición y entrega mensajes perdidos de hasta un par de horas de antigüedad. Mediante un `cliPath` SSH remoto no puede leer la base de datos, por lo que la repetición no tiene límite y todas las filas utilizan el límite de antigüedad de eventos en directo: aun así, recupera los mensajes perdidos recientemente y suprime los mensajes pendientes antiguos, pero con la ventana en directo más estrecha. Ejecute el Gateway en el Mac con Mensajes para disponer de la ventana de recuperación más amplia.

### Señal visible para el operador

Los mensajes pendientes suprimidos se registran en el nivel predeterminado y nunca se descartan silenciosamente (la marca `recovery` muestra qué ventana se aplicó):

```text
imessage: se suprimieron mensajes entrantes pendientes obsoletos account=<id> sent=<iso> recovery=<bool> (<N> suprimidos desde el inicio)
```

### Migración

`channels.imessage.catchup.*` está obsoleto: la recuperación tras periodos de inactividad es automática y no necesita configuración en instalaciones nuevas. Las configuraciones existentes con `catchup.enabled: true` se siguen respetando como perfil de compatibilidad para la ventana de repetición de recuperación. Los bloques de recuperación desactivados (`enabled: false` o sin `enabled: true`) se han retirado; `openclaw doctor --fix` los elimina.

## Solución de problemas

<AccordionGroup>
  <Accordion title="No se encuentra imsg o RPC no es compatible">
    Valide el binario y la compatibilidad con RPC:

    ```bash
    imsg rpc --help
    imsg status --json
    openclaw channels status --probe
    ```

    Si la comprobación indica que RPC no es compatible, actualice `imsg`. Si las acciones de la API privada no están disponibles, ejecute `imsg launch` en la sesión del usuario de macOS que ha iniciado sesión y vuelva a realizar la comprobación. Si el Gateway no se ejecuta en macOS, utilice la configuración de Mac remoto mediante SSH descrita anteriormente en lugar de la ruta local predeterminada `imsg`.

  </Accordion>

  <Accordion title="Los mensajes se envían, pero los iMessage entrantes no llegan">
    Primero compruebe si el mensaje llegó al Mac local. Si `chat.db` no cambia, OpenClaw no puede recibir el mensaje aunque `imsg status --json` indique que el puente funciona correctamente.

```bash
imsg chats --limit 10 --json
imsg watch --chat-id <chat-id> --json
sqlite3 ~/Library/Messages/chat.db \
  "select datetime(max(date)/1000000000 + 978307200, 'unixepoch', 'localtime'), max(ROWID) from message;"
```

    Si los mensajes enviados desde el teléfono no crean filas nuevas, repare la capa de Mensajes y Apple Push de macOS antes de cambiar la configuración de OpenClaw. Suele bastar con actualizar los servicios una vez:

```bash
launchctl kickstart -k system/com.apple.apsd
launchctl kickstart -k gui/$(id -u)/com.apple.CommCenter
launchctl kickstart -k gui/$(id -u)/com.apple.identityservicesd
launchctl kickstart -k gui/$(id -u)/com.apple.imagent
imsg launch
openclaw gateway restart
```

    Envíe un iMessage nuevo desde el teléfono y confirme que aparece una fila `chat.db` nueva o un evento `imsg watch` antes de depurar las sesiones de OpenClaw. No ejecute este procedimiento como un bucle periódico para reiniciar el puente; repetir `imsg launch` junto con reinicios del Gateway durante el trabajo activo puede interrumpir las entregas y dejar bloqueadas ejecuciones de canal en curso.

  </Accordion>

  <Accordion title="El Gateway no se ejecuta en macOS">
    El `cliPath: "imsg"` predeterminado debe ejecutarse en el Mac que tiene iniciada la sesión de Mensajes. En Linux o Windows, establezca `channels.imessage.cliPath` en un script contenedor que se conecte mediante SSH a ese Mac y ejecute `imsg "$@"`.

```bash
#!/usr/bin/env bash
exec ssh -T messages-mac imsg "$@"
```

    A continuación, ejecute:

```bash
openclaw channels status --probe --channel imessage
```

  </Accordion>

  <Accordion title="Los mensajes directos se ignoran">
    Compruebe:

    - `channels.imessage.dmPolicy`
    - `channels.imessage.allowFrom`
    - las aprobaciones de vinculación (`openclaw pairing list imessage`)

  </Accordion>

  <Accordion title="Los mensajes de grupo se ignoran">
    Compruebe:

    - `channels.imessage.groupPolicy`
    - `channels.imessage.groupAllowFrom`
    - el comportamiento de la lista de permitidos de `channels.imessage.groups`
    - la configuración de patrones de mención (`agents.entries.*.groupChat.mentionPatterns`)

  </Accordion>

  <Accordion title="Los archivos adjuntos remotos fallan">
    Compruebe:

    - `channels.imessage.remoteHost`
    - `channels.imessage.remoteAttachmentRoots`
    - la autenticación por clave SSH/SCP desde el host del Gateway
    - que la clave del host exista en `~/.ssh/known_hosts` en el host del Gateway
    - que la ruta remota se pueda leer en el Mac que ejecuta Mensajes

  </Accordion>

  <Accordion title="Se omitieron las solicitudes de permisos de macOS">
    Vuelva a ejecutar los comandos en un terminal gráfico interactivo dentro del mismo contexto de usuario y sesión, y apruebe las solicitudes:

    ```bash
    imsg chats --limit 1
    imsg send <handle> "test"
    ```

    Confirme que se han concedido Acceso total al disco y Automatización al contexto de proceso que ejecuta OpenClaw/`imsg`.

  </Accordion>
</AccordionGroup>

## Referencias de configuración

- [Referencia de configuración: iMessage](/es/gateway/config-channels#imessage)
- [Configuración del Gateway](/es/gateway/configuration)
- [Vinculación](/es/channels/pairing)

## Contenido relacionado

- [Descripción general de los canales](/es/channels) — todos los canales compatibles
- [Eliminación de BlueBubbles y la ruta de iMessage mediante imsg](/es/announcements/bluebubbles-imessage) — anuncio y resumen de la migración
- [Migración desde BlueBubbles](/es/channels/imessage-from-bluebubbles) — tabla de conversión de la configuración y transición paso a paso
- [Vinculación](/es/channels/pairing) — autenticación de mensajes directos y flujo de vinculación
- [Grupos](/es/channels/groups) — comportamiento de los chats grupales y control mediante menciones
- [Enrutamiento de canales](/es/channels/channel-routing) — enrutamiento de sesiones para mensajes
- [Seguridad](/es/gateway/security) — modelo de acceso y refuerzo
