---
read_when:
    - Emparejamiento o reconexión del nodo iOS
    - Activación o solución de problemas del Node directo de Apple Watch
    - Ejecutar la aplicación de iOS desde el código fuente
    - Depuración del descubrimiento del Gateway o de los comandos de canvas
summary: 'Aplicación de nodo para iOS: conexión al Gateway, emparejamiento, lienzo y solución de problemas'
title: Aplicación para iOS
x-i18n:
    generated_at: "2026-07-26T04:42:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2b01a63fa1e2c445f7fb35843536f7f5918e94bfe885dac19c852d7d52d86342
    source_path: platforms/ios.md
    workflow: 16
---

Disponibilidad: las compilaciones de la aplicación para iPhone se distribuyen a través de los canales de Apple cuando están habilitadas para una versión. Las compilaciones de desarrollo local también pueden ejecutarse desde el código fuente.

## Qué hace

- Se conecta a un Gateway mediante WebSocket (LAN o tailnet).
- Expone las capacidades del nodo: Canvas, captura de pantalla, captura de cámara, ubicación, modo Talk, activación por voz y resúmenes opcionales de Salud.
- Recibe comandos `node.invoke` e informa de eventos de estado del nodo.
- Permite explorar en modo de solo lectura el espacio de trabajo del agente seleccionado desde la superficie Agentes (Archivos): navegación por directorios, vistas previas de texto con resaltado de sintaxis, vistas previas de imágenes y exportación mediante la hoja para compartir. No se permiten operaciones de escritura; el Gateway limita el tamaño de las vistas previas.
- Mantiene una pequeña caché sin conexión y de solo lectura de las sesiones de chat y transcripciones recientes de cada Gateway emparejado: al iniciarse en frío, muestra inmediatamente la última transcripción conocida y la actualiza cuando responde el Gateway; los chats recientes permanecen disponibles para su consulta sin conexión; y restablecer u olvidar elimina la caché local protegida.
- Pone en cola los mensajes de texto enviados sin conexión en una bandeja de salida persistente por Gateway (hasta 50): los mensajes en cola aparecen en la transcripción, se envían en orden al reconectarse con reintentos idempotentes, persisten hasta que el historial canónico confirma el envío, vuelven a intentarse con espera exponencial antes de mostrar una acción para reintentar o eliminar y, tras 48 horas sin conexión, caducan en lugar de enviarse; restablecer u olvidar borra la cola junto con la caché.
- Chat es la única superficie de texto y voz. Las acciones de Chat pueden abrir la pantalla completa de Sesiones sin salir de Chat y pueden mostrar u ocultar el razonamiento del asistente y la actividad de las herramientas. Toque el micrófono para dictar un borrador, abra su menú para grabar una nota de voz o use el control Talk integrado para hablar en tiempo real; el control Talk se anima según el nivel del micrófono en directo o de la reproducción mientras escucha o habla.
- **Ajustes -> OpenClaw** abre un asistente específico para configurar el Gateway cuando la conexión del operador dispone de `operator.admin` y el Gateway admite `openclaw.chat`. Su conversación de configuración se mantiene separada del Chat normal, oculta localmente las respuestas secretas y solo pasa a Chat después de tocar **Abrir Chat**.
- Reproduce los mensajes del asistente bajo demanda: mantenga pulsado un mensaje en Chat y seleccione **Escuchar**. La aplicación reproduce los clips `tts.speak` compatibles del Gateway con el proveedor de TTS configurado y recurre a la síntesis de voz del dispositivo cuando el audio del Gateway no está disponible o no puede reproducirse. La reproducción se detiene al cambiar de sesión o al pasar la aplicación a segundo plano.

## Requisitos

- Gateway en ejecución en otro dispositivo (macOS, Linux o Windows mediante WSL2).
- Ruta de red:
  - La misma LAN mediante Bonjour, **o**
  - Tailnet mediante DNS-SD unidifusión (dominio de ejemplo: `openclaw.internal.`), **o**
  - Host/puerto manual (alternativa).

## Inicio rápido (emparejar y conectar)

En el primer inicio, la aplicación muestra una breve explicación del emparejamiento y una
página de permisos (notificaciones, cámara, micrófono, fotos, contactos,
calendario, recordatorios y ubicación). Todos los permisos son opcionales y pueden modificarse
más adelante en **Ajustes** -> **Permisos** o en la aplicación Ajustes de iOS.

1. Inicie un Gateway autenticado con una ruta accesible desde el teléfono. Tailscale
   Serve es la ruta remota recomendada:

```bash
openclaw gateway --port 18789 --tailscale serve
```

Para una configuración de confianza en la misma LAN, use en su lugar un `gateway.bind: "lan"`
autenticado. La vinculación predeterminada a la interfaz de bucle invertido no es accesible desde un teléfono. Si el
Gateway aún no se ha configurado, ejecute primero `openclaw onboard` para que la creación
del código de configuración disponga de una ruta de autenticación mediante token o contraseña.

2. Abra la [interfaz de control](/es/web/control-ui), seleccione **Nodes** y haga clic en
   **Pair mobile device** en la página **Devices**. Se recomienda el acceso completo
   y está seleccionado de forma predeterminada; elija Limited access solo cuando quiera omitir
   los controles administrativos del Gateway y, a continuación, haga clic en **Create setup code**.

3. En la aplicación para iOS, abra **Ajustes** -> **Gateway**, escanee el código QR (o pegue
   el código de configuración) y conéctese.

   Si el código de configuración contiene rutas tanto de LAN como de Tailscale Serve, la aplicación
   las comprueba en orden y guarda el primer punto de conexión accesible.

   Los Gateways emparejados permanecen en la lista **Gateways**. La marca de verificación identifica
   el Gateway enfocado; use el control con forma de rayo de otra fila para mantener conectada
   simultáneamente su sesión de operador. Cambiar el foco no
   desconecta otros Gateways habilitados. Solo el Gateway enfocado recibe la
   sesión de nodo del iPhone que contiene las capacidades, por lo que la cámara, la pantalla, la ubicación y
   otros comandos del dispositivo siempre tienen un único propietario inequívoco. iOS puede suspender
   estas conexiones en primer plano cuando la aplicación pasa a segundo plano.

4. La aplicación oficial se conecta automáticamente. Si **Pending approval** muestra una
   solicitud, revise su rol y sus ámbitos antes de aprobarla.

   **Ajustes → Gateway** muestra si la conexión de operador guardada tiene acceso
   **Completo** o **Limitado**. La configuración `ws://` de LAN en texto sin cifrar se limita automáticamente
   por seguridad del token al portador. Si está limitada, configure `wss://` o
   Tailscale Serve, escanee un nuevo código de acceso completo desde la interfaz de control o `openclaw qr`
   y vuelva a conectarse para habilitar los ajustes y las actualizaciones.

El botón de la interfaz de control requiere una sesión ya emparejada con `operator.admin`.
Como alternativa desde la terminal, elija un Gateway detectado en la aplicación para iOS (o habilite
Host manual e introduzca el host/puerto) y, a continuación, apruebe la solicitud en el host del Gateway:

```bash
openclaw devices list
openclaw devices approve <requestId>
```

Si la aplicación vuelve a intentar el emparejamiento con datos de autenticación modificados (rol/ámbitos/clave pública), la solicitud pendiente anterior se sustituye y se crea un nuevo `requestId`. Ejecute de nuevo `openclaw devices list` antes de aprobarla.

Opcional: si el nodo de iOS siempre se conecta desde una subred estrictamente controlada, puede habilitar la aprobación automática del nodo durante el primer emparejamiento mediante CIDR explícitos o direcciones IP exactas:

```json5
{
  gateway: {
    nodes: {
      pairing: {
        autoApproveCidrs: ["192.168.1.0/24"],
      },
    },
  },
}
```

Esta opción está deshabilitada de forma predeterminada. Solo se aplica a emparejamientos `role: node` nuevos sin ámbitos solicitados. El emparejamiento de operadores o navegadores, así como cualquier cambio de rol, ámbito, metadatos o clave pública, siguen requiriendo aprobación manual.

5. Verifique la conexión:

```bash
openclaw nodes status
openclaw gateway call node.list --params "{}"
```

## Resúmenes de Salud

El nodo de iOS puede devolver un agregado opcional y de solo lectura de HealthKit correspondiente al día
natural actual. El consentimiento del dispositivo iOS y la autorización explícita de comandos del Gateway son
controles independientes. Consulte [Resúmenes de HealthKit](/es/platforms/ios-healthkit) para obtener información sobre
la configuración, la invocación, los campos de la carga útil, el comportamiento de privacidad y la resolución de problemas.

De forma predeterminada, la aplicación complementaria del Apple Watch sigue usando el enlace existente con el iPhone y
no necesita un emparejamiento independiente con el Gateway. Empareje el Watch con el iPhone en
la aplicación Watch de Apple, instale OpenClaw desde **Watch app -> My Watch -> Available
Apps** y, a continuación, abra OpenClaw una vez en ambos dispositivos.

## Revisar las aprobaciones de comandos

Una conexión de operador con `operator.admin`, o una conexión
`operator.approvals` emparejada a la que el Gateway se dirija explícitamente, puede revisar
las solicitudes de ejecución pendientes en el iPhone. La tarjeta de aprobación muestra la vista previa
depurada del comando del Gateway, la advertencia, el contexto del host, la caducidad y solo las
decisiones que ofrece esa solicitud. El Apple Watch emparejado recibe la misma
solicitud segura para el revisor mediante el enlace existente con el iPhone y ofrece el subconjunto compacto
de decisiones de permitir una vez o denegar. El modo de conexión directa del Watch al Gateway no transmite
solicitudes de aprobación.

El estado de aprobación se comparte con la interfaz de control y las superficies de chat compatibles. La
primera respuesta confirmada prevalece. El iPhone y el Watch obtienen el registro
terminal canónico del Gateway después de que otra superficie resuelva la solicitud, tras una
notificación remota de resolución y siempre que pueda haberse perdido una confirmación
de resolución. Las acciones permanecen deshabilitadas hasta que esa nueva lectura confirme si la
solicitud sigue pendiente.

La propiedad de la aprobación está vinculada al Gateway seleccionado. Cambiar de Gateway no puede
aplicar una solicitud antigua a la conexión sustituta. Los Gateways anteriores a los
métodos de aprobación unificados recurren a los métodos específicos de ejecución incluidos en la versión;
el estado terminal conservado y los resultados más completos entre superficies requieren un
Gateway actualizado.

## Responder a las preguntas del agente

Chat muestra las preguntas pendientes del Gateway como tarjetas nativas para las conexiones de operador
con `operator.questions` (o `operator.admin`). Las tarjetas admiten opciones de selección
única y múltiple, descripciones de opciones, respuestas de texto libre en **Otra opción** y una
cuenta atrás hasta la caducidad. Las reconexiones vuelven a cargar las preguntas pendientes desde el Gateway. Una tarjeta
se bloquea cuando este dispositivo la responde, otra superficie la responde primero o la
pregunta caduca o se cancela.

## Nodo directo opcional de Apple Watch

El modo directo proporciona al reloj su propia identidad de nodo firmada y conexión con el Gateway.
Los comandos de nodo compatibles siguen funcionando mediante la red Wi-Fi o celular del reloj mientras
OpenClaw está activo, incluso cuando el iPhone emparejado no está disponible.

Requisitos:

- El iPhone está conectado al Gateway con el ámbito `operator.admin`.
- El código de configuración anuncia un punto de conexión `wss://` del Gateway con un certificado de confianza
  para watchOS; el reloj consulta periódicamente el origen `https://` correspondiente. No se admiten HTTP sin cifrar ni
  certificados autofirmados o cuya confianza se base únicamente en la huella digital. Consulte [Emparejamiento
  gestionado por el Gateway](/es/gateway/pairing) para configurar el punto de conexión. Las rutas de bucle invertido, exclusivas del iPhone
  y exclusivas de tailnet no son accesibles de forma independiente desde el reloj.
- El uso de la red celular requiere un Apple Watch compatible con conexión celular y con el servicio activo.
- OpenClaw está activo en el reloj. Apple no permite que las aplicaciones ordinarias de watchOS
  mantengan conexiones WebSocket/TCP genéricas, por lo que el nodo directo usa consultas HTTPS
  breves y vuelve a conectarse cuando la aplicación regresa a primer plano. Consulte la
  [guía de Apple sobre redes de bajo nivel en watchOS](https://developer.apple.com/documentation/technotes/tn3135-low-level-networking-on-watchOS).

Configuración:

1. En el iPhone, abra **Ajustes -> Apple Watch**.
2. Toque **Habilitar la conexión directa con el Gateway**.
3. Abra OpenClaw en el reloj antes de que caduque el código de configuración de corta duración.
4. Verifique la fila independiente del Apple Watch con `openclaw nodes status`.

El código de configuración contiene una credencial de arranque de corta duración y exclusiva para el nodo; trátela
como una contraseña hasta que caduque. Nunca contiene la contraseña ni el token del Gateway
guardados en el iPhone. Tras el emparejamiento, el reloj almacena su propio token de dispositivo y
elimina la credencial de arranque. El modo directo solo cubre los comandos siguientes.
Chat, Talk, las aprobaciones y el flujo de notificaciones `watch.*` existente siguen siendo
funciones transmitidas mediante el iPhone y continúan requiriendo el iPhone emparejado.

Comandos directos del nodo de watchOS:

| Superficie    | Comandos                       | Notas                                                        |
| ------------- | ------------------------------ | ------------------------------------------------------------ |
| Dispositivo   | `device.info`, `device.status` | Identidad del Watch, batería, estado térmico, almacenamiento y red. |
| Notificaciones | `system.notify`                | Mientras la aplicación está activa; requiere permiso en el reloj.  |

watchOS no ofrece WebKit a las aplicaciones de terceros, por lo que el nodo directo del reloj
no anuncia comandos de Canvas.

## Notificaciones push mediante retransmisión para compilaciones oficiales

Las compilaciones oficiales distribuidas para iOS usan un retransmisor push externo en lugar de publicar el token APNs sin procesar en el Gateway. Las compilaciones oficiales de App Store procedentes del canal público de versiones usan el retransmisor alojado en `https://ios-push-relay.openclaw.ai`; esta URL base está integrada en el código para la distribución mediante App Store y no admite ningún valor de sustitución.

Las implementaciones con un retransmisor personalizado requieren una ruta de compilación e implementación de iOS deliberadamente independiente, cuya URL del retransmisor coincida con la URL del retransmisor del Gateway. El canal de versiones de App Store nunca acepta una URL de retransmisor personalizada. Si se usa una compilación con retransmisor personalizado, configure la URL coincidente del retransmisor del Gateway:

```json5
{
  gateway: {
    push: {
      apns: {
        relay: {
          baseUrl: "https://relay.example.com",
        },
      },
    },
  },
}
```

Cómo funciona el flujo:

- La aplicación iOS se registra en el relé mediante App Attest y un JWS de transacción de la aplicación de StoreKit.
- El relé devuelve un identificador de relé opaco junto con una autorización de envío limitada al registro.
- La aplicación iOS obtiene la identidad del gateway emparejado (`gateway.identity.get`) y la incluye en el registro del relé, de modo que el registro respaldado por el relé queda delegado en ese gateway específico.
- La aplicación reenvía ese registro respaldado por el relé al gateway emparejado mediante `push.apns.register`.
- El gateway utiliza ese identificador de relé almacenado para `push.test`, activaciones en segundo plano y avisos de activación.
- Si posteriormente la aplicación se conecta a otro gateway o a una compilación con una URL base de relé diferente, actualiza el registro del relé en lugar de reutilizar la vinculación anterior.

Lo que el gateway **no** necesita para esta ruta: ningún token de relé para todo el despliegue ni una clave directa de APNs para los envíos oficiales respaldados por el relé de la App Store.

Flujo esperado para el operador:

1. Instalar la aplicación oficial para iOS.
2. Opcional: establecer `gateway.push.apns.relay.baseUrl` en el gateway únicamente cuando se utilice deliberadamente una compilación personalizada con un relé independiente.
3. Emparejar la aplicación con el gateway y dejar que termine de conectarse.
4. La aplicación publica `push.apns.register` cuando dispone de un token de APNs, la sesión del operador está conectada y el registro del relé se completa correctamente.
5. A partir de ese momento, `push.test`, las activaciones por reconexión y los avisos de activación pueden utilizar el registro almacenado respaldado por el relé.

## Señales de actividad en segundo plano

Cuando iOS activa la aplicación mediante una notificación push silenciosa, una actualización en segundo plano o un evento de cambio significativo de ubicación, la aplicación intenta realizar una breve reconexión del nodo y luego llama a `node.event` con `event: "node.presence.alive"`. El gateway registra esto como `lastSeenAtMs`/`lastSeenReason` en los metadatos del nodo/dispositivo emparejado únicamente después de conocer la identidad autenticada del dispositivo del nodo.

La aplicación considera que una activación en segundo plano se ha registrado correctamente solo cuando la respuesta del gateway incluye `handled: true`. Los gateways anteriores pueden confirmar `node.event` con `{ "ok": true }`; esa respuesta es compatible, pero no cuenta como una actualización persistente de la última actividad.

Nota de compatibilidad:

- `OPENCLAW_APNS_RELAY_BASE_URL` sigue funcionando como anulación temporal mediante una variable de entorno para el gateway (`gateway.push.apns.relay.baseUrl` es la ruta que prioriza la configuración).
- El modo push de la compilación publicada en la App Store codifica de forma fija el host del relé alojado y nunca lee una anulación de la URL del relé; la variable de entorno de compilación `OPENCLAW_PUSH_RELAY_BASE_URL` solo afecta a los modos de compilación local o de entorno aislado de iOS.

## Flujo de autenticación y confianza

El relé existe para aplicar dos restricciones que el uso directo de APNs desde el gateway no puede proporcionar a las compilaciones oficiales de iOS:

- Solo las compilaciones genuinas de OpenClaw para iOS distribuidas mediante Apple pueden utilizar el relé alojado.
- Un gateway solo puede enviar notificaciones push respaldadas por el relé a dispositivos iOS que se hayan emparejado con ese gateway específico.

Paso a paso:

1. `iOS app -> gateway`: la aplicación se empareja con el gateway mediante el flujo normal de autenticación del Gateway, lo que le proporciona una sesión de nodo autenticada y una sesión de operador autenticada. La sesión del operador llama a `gateway.identity.get`.
2. `iOS app -> relay`: la aplicación llama a los endpoints de registro del relé mediante HTTPS con una prueba de App Attest y un JWS de transacción de la aplicación de StoreKit. El relé valida el ID del paquete, la prueba de App Attest y la prueba de distribución de Apple, y exige la ruta de distribución oficial/de producción. Esto impide que las compilaciones locales de Xcode/desarrollo utilicen el relé alojado, ya que una compilación local no puede proporcionar una prueba válida de distribución oficial de Apple.
3. `gateway identity delegation`: antes de registrarse en el relé, la aplicación obtiene la identidad del gateway emparejado desde `gateway.identity.get` y la incluye en la carga útil de registro del relé. El relé devuelve un identificador de relé y una autorización de envío limitada al registro y delegada en esa identidad del gateway.
4. `gateway -> relay`: el gateway almacena el identificador de relé y la autorización de envío procedentes de `push.apns.register`. Durante `push.test`, las activaciones por reconexión y los avisos de activación, el gateway firma la solicitud de envío con su propia identidad de dispositivo; el relé verifica tanto la autorización de envío almacenada como la firma del gateway con respecto a la identidad del gateway delegada durante el registro. Otro gateway no puede reutilizar ese registro almacenado, aunque consiga obtener el identificador de algún modo.
5. `relay -> APNs`: el relé posee las credenciales de APNs de producción y el token de APNs sin procesar de la compilación oficial. El gateway nunca almacena el token de APNs sin procesar de las compilaciones oficiales respaldadas por el relé; el relé envía la notificación push final a APNs en nombre del gateway emparejado.

Motivo de este diseño: mantener las credenciales de APNs de producción fuera de los gateways de los usuarios, evitar almacenar en el gateway los tokens de APNs sin procesar de las compilaciones oficiales, permitir el uso del relé alojado únicamente a las compilaciones oficiales de OpenClaw para iOS e impedir que un gateway envíe notificaciones push de activación a dispositivos iOS pertenecientes a otro gateway.

Las compilaciones locales/manuales siguen utilizando APNs directamente. Si se prueban esas compilaciones sin el relé, el gateway sigue necesitando credenciales directas de APNs:

```bash
export OPENCLAW_APNS_TEAM_ID="TEAMID"
export OPENCLAW_APNS_KEY_ID="KEYID"
export OPENCLAW_APNS_PRIVATE_KEY_P8="$(cat /path/to/AuthKey_KEYID.p8)"
```

Estas son variables de entorno de ejecución del host del gateway, no ajustes de Fastlane. `apps/ios/fastlane/.env` solo almacena la autenticación de App Store Connect, como `APP_STORE_CONNECT_KEY_ID` y `APP_STORE_CONNECT_ISSUER_ID`; no configura la entrega directa mediante APNs para las compilaciones locales de iOS.

Almacenamiento recomendado en el host del gateway, coherente con las demás credenciales de proveedores ubicadas en `~/.openclaw/credentials/`:

```bash
mkdir -p ~/.openclaw/credentials/apns
chmod 700 ~/.openclaw/credentials/apns
mv /path/to/AuthKey_KEYID.p8 ~/.openclaw/credentials/apns/AuthKey_KEYID.p8
chmod 600 ~/.openclaw/credentials/apns/AuthKey_KEYID.p8
export OPENCLAW_APNS_PRIVATE_KEY_PATH="$HOME/.openclaw/credentials/apns/AuthKey_KEYID.p8"
```

No se debe confirmar el archivo `.p8` en el repositorio ni colocarlo dentro de la copia de trabajo del repositorio.

## Rutas de detección

### Bonjour (LAN)

La aplicación iOS busca `_openclaw-gw._tcp` en `local.` y, cuando está configurado, en el mismo dominio de detección DNS-SD de área extensa. Los gateways de la misma LAN aparecen automáticamente desde `local.`; la detección entre redes puede utilizar el dominio de área extensa configurado sin cambiar el tipo de señal.

### Tailnet (entre redes)

Si mDNS está bloqueado, se debe utilizar una zona DNS-SD unicast (elija un dominio; por ejemplo: `openclaw.internal.`) y el DNS dividido de Tailscale. Consulte [Bonjour](/es/gateway/bonjour) para ver el ejemplo de CoreDNS.

### Host/puerto manual

En Settings, active **Manual Host** e introduzca el host y el puerto del gateway (valor predeterminado: `18789`).

## Varios gateways

La aplicación mantiene un registro de todos los gateways con los que se ha emparejado, por lo que es posible cambiar entre ellos sin volver a realizar el emparejamiento:

- **Settings -> Gateway** muestra una lista de **Paired Gateways** con el gateway activo marcado. Toque una entrada para cambiar; la aplicación cierra las sesiones actuales y vuelve a conectarse al gateway seleccionado. Cuando hay más de un gateway emparejado, aparece un menú de cambio rápido junto a la fila de conexión.
- Las credenciales, las decisiones de confianza de TLS, las preferencias específicas de cada gateway y el historial de chat almacenado en caché se guardan por separado para cada gateway. Al cambiar de gateway nunca se mezclan sus estados, y el registro de notificaciones push sigue al gateway activo.
- Deslice un gateway emparejado (o utilice su menú contextual) para seleccionar **Forget**, lo que elimina sus credenciales, tokens de dispositivo, anclaje TLS y chats almacenados en caché.
- Los gateways detectados deben estar visibles en la red para poder cambiar a ellos; los gateways manuales vuelven a conectarse mediante el host y el puerto guardados.

## Canvas + A2UI

El nodo iOS representa un canvas WKWebView. Utilice `node.invoke` para controlarlo:

```bash
openclaw nodes invoke --node "iOS Node" --command canvas.navigate --params '{"url":"http://<gateway-host>:18789/__openclaw__/canvas/"}'
```

Notas:

- El host de canvas del Gateway sirve `/__openclaw__/canvas/` y `/__openclaw__/a2ui/` desde el servidor HTTP del Gateway (el mismo puerto que `gateway.port`, valor predeterminado: `18789`).
- El nodo iOS mantiene la estructura integrada como vista conectada predeterminada. `canvas.a2ui.push` y `canvas.a2ui.reset` utilizan la página A2UI incluida y perteneciente a la aplicación.
- Las páginas A2UI remotas del Gateway son de solo representación en iOS; las acciones de botones A2UI nativos solo se aceptan desde páginas incluidas y pertenecientes a la aplicación.
- Para volver a la estructura integrada, utilice `canvas.navigate` y `{"url":""}`.

## Relación con Computer Use

La aplicación iOS es una superficie de nodo móvil, no un backend de Codex Computer Use. Codex Computer Use y `cua-driver mcp` controlan un escritorio macOS local mediante herramientas MCP; la aplicación iOS expone capacidades del iPhone mediante comandos de nodo de OpenClaw como `canvas.*`, `camera.*`, `screen.*`, `location.*` y `talk.*`.

Los agentes pueden seguir operando la aplicación iOS mediante OpenClaw invocando comandos de nodo, pero esas llamadas pasan por el protocolo de nodos del gateway y están sujetas a los límites de primer y segundo plano de iOS. Utilice [Codex Computer Use](/es/plugins/codex-computer-use) para controlar el escritorio local y esta página para consultar las capacidades de los nodos iOS.

### Evaluación/instantánea del canvas

```bash
openclaw nodes invoke --node "iOS Node" --command canvas.eval --params '{"javaScript":"(() => { const {ctx} = window.__openclaw; ctx.clearRect(0,0,innerWidth,innerHeight); ctx.lineWidth=6; ctx.strokeStyle=\"#ff2d55\"; ctx.beginPath(); ctx.moveTo(40,40); ctx.lineTo(innerWidth-40, innerHeight-40); ctx.stroke(); return \"ok\"; })()"}'
```

```bash
openclaw nodes invoke --node "iOS Node" --command canvas.snapshot --params '{"maxWidth":900,"format":"jpeg"}'
```

## Activación por voz + modo de conversación

- La activación por voz y el modo de conversación están disponibles en Settings.
- La conversación en tiempo real de OpenAI utiliza WebRTC propiedad del cliente cuando `talk.realtime.transport` es `webrtc`; una configuración explícita de `gateway-relay` sigue siendo propiedad del Gateway. Consulte [Modo de conversación](/es/nodes/talk).
- Los nodos iOS compatibles con la conversación anuncian la capacidad `talk` y pueden declarar `talk.ptt.start`, `talk.ptt.stop`, `talk.ptt.cancel` y `talk.ptt.once`; el Gateway permite de forma predeterminada esos comandos de pulsar para hablar en los nodos de confianza compatibles con la conversación.
- iOS puede suspender el audio en segundo plano; las funciones de voz deben considerarse sujetas a disponibilidad cuando la aplicación no esté activa.

## Errores comunes

- `NODE_BACKGROUND_UNAVAILABLE`: lleve la aplicación iOS al primer plano (los comandos de canvas, cámara y pantalla lo requieren).
- `A2UI_HOST_UNAVAILABLE`: no se pudo acceder a la página A2UI incluida desde la WebView de la aplicación; mantenga la aplicación en primer plano en la pestaña Screen y vuelva a intentarlo.
- La solicitud de emparejamiento nunca aparece: ejecute `openclaw devices list` y apruébela manualmente.
- El Watch no muestra el estado del iPhone: confirme que el iPhone informa de `watchPaired: true`
  y `watchAppInstalled: true` en `watch.status`. Si el emparejamiento es falso, empareje el
  Watch en la aplicación Watch de Apple. Si la instalación es falsa, instale la aplicación complementaria
  desde **My Watch -> Available Apps**. Después de cualquiera de estos cambios, abra OpenClaw una vez en el
  Watch; la disponibilidad inmediata sigue requiriendo que ambas aplicaciones estén en ejecución,
  mientras que las actualizaciones en cola pueden llegar más tarde en segundo plano.
- La reconexión falla después de reinstalar: se borró el token de emparejamiento del Keychain; vuelva a emparejar el nodo.

## Documentación relacionada

- [Emparejamiento](/es/channels/pairing)
- [Detección](/es/gateway/discovery)
- [Bonjour](/es/gateway/bonjour)
