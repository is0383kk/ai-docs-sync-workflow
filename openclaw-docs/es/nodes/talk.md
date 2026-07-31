---
read_when:
    - Implementación del modo Conversación en macOS/iOS/Android
    - Cambiar el comportamiento de voz/TTS/interrupción
summary: 'Modo de conversación: conversaciones de voz continuas mediante STT/TTS local y voz en tiempo real'
title: Modo de conversación
x-i18n:
    generated_at: "2026-07-26T04:41:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b21319eee169ba898331f87279a2b2a5170441131a1e9cdc85c15b268d165e21
    source_path: nodes/talk.md
    workflow: 16
---

El modo de conversación abarca cinco modalidades de ejecución:

- **Conversación nativa en macOS/iOS/Android**: reconocimiento de voz nativo, chat del Gateway y TTS de `talk.speak`. El reconocimiento de Apple Speech en macOS/iOS puede utilizar servicios de red; el comportamiento en Android depende del servicio de voz instalado. Los nodos anuncian la capacidad `talk` y declaran qué comandos `talk.*` admiten.
- **Conversación en iOS (en tiempo real)**: WebRTC gestionado por el cliente para las configuraciones en tiempo real de OpenAI que seleccionan el transporte `webrtc` u omiten el transporte. Las configuraciones en tiempo real que especifican `gateway-relay`, `provider-websocket` y las que no son de OpenAI permanecen en el relé gestionado por el Gateway; las configuraciones que no son en tiempo real utilizan el bucle de voz nativo.
- **Conversación en el navegador**: `talk.client.create` para sesiones `webrtc`/`provider-websocket` gestionadas por el cliente, o `talk.session.create` para sesiones `gateway-relay` gestionadas por el Gateway. `managed-room` está reservado para la transferencia al Gateway y las salas de comunicación mediante pulsar para hablar.
- **Conversación en Android (en tiempo real)**: actívela con `talk.realtime.mode: "realtime"` y `talk.realtime.transport: "gateway-relay"`. De lo contrario, Android sigue utilizando el reconocimiento de voz nativo, el chat del Gateway y `talk.speak`.
- **Clientes solo de transcripción**: `talk.session.create({ mode: "transcription", transport: "gateway-relay", brain: "none" })`, seguido de `talk.session.appendAudio`, `talk.session.cancelTurn` y `talk.session.close` para subtítulos o dictado sin una respuesta de voz del asistente. Las notas de voz cargadas para un único uso siguen utilizando la ruta de audio de [comprensión multimedia](/es/nodes/media-understanding).

La conversación nativa es un bucle continuo: escucha la voz, envía la transcripción al modelo mediante la sesión activa, espera la respuesta y la reproduce mediante el proveedor de conversación configurado (`talk.speak`).

La conversación en tiempo real gestionada por el cliente reenvía las llamadas a herramientas del proveedor mediante `talk.client.toolCall` en lugar de llamar directamente a `chat.send`. Mientras haya una consulta en tiempo real activa, los clientes pueden llamar a `talk.client.steer` o `talk.session.steer` para clasificar la entrada hablada como `status`, `steer`, `cancel` o `followup`. Las instrucciones de redirección aceptadas se ponen en cola en la ejecución integrada activa; las rechazadas devuelven un motivo como `no_active_run`, `not_streaming` o `compacting`.

Las intervenciones finalizadas del usuario y del asistente en tiempo real siempre se añaden de inmediato a la sesión activa del agente, por lo que los turnos posteriores de chat y voz comparten un mismo historial. Los transportes gestionados por el cliente notifican sus transcripciones finalizadas con identificadores de entrada estables; las sesiones de relé del Gateway añaden los mismos eventos en el servidor. Las sesiones del proveedor también reciben el contexto limitado del perfil en tiempo real utilizado por la voz de Discord.

Las ejecuciones de consulta originadas por voz requieren una confirmación oral nueva y exacta antes de realizar acciones de gran impacto, como enviar mensajes, controlar nodos, ejecutar acciones en el navegador o el equipo, modificar servicios, ejecutar comandos destructivos del shell o publicar. La confirmación solo se aplica a los argumentos exactos de la herramienta bloqueada y se consume una vez; las ejecuciones simultáneas no relacionadas no se ven afectadas. Cuando finaliza una llamada, OpenClaw puede enviar a la última ubicación de entrega de la sesión que no sea WebChat un resumen compacto de **Cambios de la llamada de voz** correspondiente a las herramientas que realizaron modificaciones.

La conversación solo de transcripción emite el mismo contenedor de eventos de conversación que las sesiones en tiempo real y STT/TTS, pero utiliza `mode: "transcription"` y `brain: "none"`. Todas las sesiones de conversación difunden eventos por el canal `talk.event`; los clientes se suscriben a él para recibir actualizaciones parciales y finales de las transcripciones (`transcript.delta`/`transcript.done`) y otros datos de telemetría de la sesión.

La conversación con vídeo en el navegador está disponible para WebRTC de OpenAI Realtime y las sesiones WebSocket de proveedor de Google Live. OpenAI recibe una única imagen JPEG limitada cuando
`describe_view` solicita contexto visual; no recibe una transmisión continua
de la cámara. Google Live recibe fotogramas JPEG limitados directamente desde el
navegador, a un máximo de un fotograma por segundo, mientras que `describe_view` informa del
estado de la transmisión de la cámara. En ambos casos, los fotogramas de la cámara eluden el Gateway y,
al detener la conversación, se liberan las pistas de la cámara y del micrófono.

## Comportamiento (macOS)

- Superposición siempre visible mientras el modo de conversación está activado.
- Transiciones de fase **Escuchando &rarr; Pensando &rarr; Hablando**.
- Tras una pausa breve (intervalo de silencio), se envía la transcripción actual.
- Las respuestas se escriben en WebChat (igual que al escribir).
- **Interrumpir al detectar voz** (activado de forma predeterminada): si el usuario habla mientras el asistente está hablando, la reproducción se detiene y se registra la marca temporal de la interrupción para la siguiente instrucción.

## Directivas de voz en las respuestas

El asistente puede anteponer a una respuesta una sola línea JSON para controlar la voz:

```json
{ "voice": "<voice-id>", "once": true }
```

Reglas:

- Solo la primera línea que no esté vacía; la línea JSON se elimina antes de la reproducción mediante TTS.
- Las claves desconocidas se ignoran.
- `once: true` se aplica únicamente a la respuesta actual; sin esta opción, la voz se convierte en el nuevo valor predeterminado del modo de conversación.

Claves admitidas: `voice` / `voice_id` / `voiceId`, `model` / `model_id` / `modelId`, `speed`, `rate` (palabras por minuto), `stability`, `similarity`, `style`, `speakerBoost`, `seed`, `normalize`, `lang`, `output_format`, `latency_tier`, `once`.

## Configuración (`~/.openclaw/openclaw.json`)

```json5
{
  talk: {
    provider: "elevenlabs",
    providers: {
      elevenlabs: {
        voiceId: "elevenlabs_voice_id",
        modelId: "eleven_v3",
        outputFormat: "mp3_44100_128",
        apiKey: "elevenlabs_api_key",
      },
      mlx: {
        modelId: "mlx-community/Soprano-80M-bf16",
      },
      system: {},
    },
    speechLocale: "ru-RU",
    silenceTimeoutMs: 1500,
    interruptOnSpeech: true,
    realtime: {
      provider: "openai",
      providers: {
        openai: {
          apiKey: "openai_api_key",
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
        },
      },
      instructions: "Speak warmly and keep answers brief.",
      mode: "realtime",
      transport: "webrtc",
      brain: "agent-consult",
    },
  },
}
```

| Clave                                    | Predeterminado                             | Notas                                                                                                                                                                                                                                                                      |
| ---------------------------------------- | ------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `provider`                               | -                                          | Proveedor de TTS de Active Talk. Use `elevenlabs`, `mlx` o `system` para las rutas de reproducción local de macOS.                                                                                                                                                                             |
| `providers.<id>.voiceId`                 | -                                          | ElevenLabs recurre a `ELEVENLABS_VOICE_ID` / `SAG_VOICE_ID`, o a la primera voz disponible con una clave de API.                                                                                                                                                             |
| `speechLocale`                           | valor predeterminado del dispositivo       | Configuración regional BCP 47 para el reconocimiento de voz nativo de Android, iOS y macOS. Apple Speech puede usar servicios de red; Android también reenvía el componente de idioma a la transcripción de entrada en tiempo real.                                                                                  |
| `providers.elevenlabs.modelId`           | `eleven_v3`                                |                                                                                                                                                                                                                                                                            |
| `providers.mlx.modelId`                  | `mlx-community/Soprano-80M-bf16`           |                                                                                                                                                                                                                                                                            |
| `providers.elevenlabs.apiKey`            | -                                          | Recurre a `ELEVENLABS_API_KEY` (o al perfil de shell del Gateway, si está disponible).                                                                                                                                                                                                |
| `silenceTimeoutMs`                       | `700` ms macOS/Android, `900` ms iOS       | Intervalo de pausa antes de que Talk envíe la transcripción.                                                                                                                                                                                                                             |
| `interruptOnSpeech`                      | `true`                                     |                                                                                                                                                                                                                                                                            |
| `outputFormat`                           | `pcm_44100` macOS/iOS, `pcm_24000` Android | Establezca `mp3_*` para forzar la transmisión de MP3.                                                                                                                                                                                                                                        |
| `consultThinkingLevel`                   | sin establecer                            | Sustitución del nivel de razonamiento para la ejecución del agente que procesa las llamadas `openclaw_agent_consult` en tiempo real.                                                                                                                                                                                  |
| `consultFastMode`                        | sin establecer                            | Sustitución del modo rápido para las llamadas `openclaw_agent_consult` en tiempo real.                                                                                                                                                                                                            |
| `realtime.provider`                      | -                                          | `openai` para WebRTC, `google` para el WebSocket del proveedor o un proveedor exclusivo de puente mediante la retransmisión del Gateway.                                                                                                                                                                     |
| `realtime.providers.<id>`                | -                                          | Configuración en tiempo real propiedad del proveedor. Los navegadores solo reciben credenciales de sesión efímeras/restringidas, nunca una clave de API estándar.                                                                                                                                                 |
| `realtime.providers.openai.speakerVoice` | `alloy`                                    | Id. de voz integrado de OpenAI Realtime (la clave anterior `voice` aún funciona, pero está obsoleta). Voces actuales de `gpt-realtime-2.1`: `alloy`, `ash`, `ballad`, `cedar`, `coral`, `echo`, `marin`, `sage`, `shimmer`, `verse`; se recomiendan `marin` y `cedar` para obtener la mejor calidad. |
| `realtime.transport`                     | -                                          | `webrtc`: WebRTC de OpenAI controlado por el cliente en iOS y en el navegador. `provider-websocket`: controlado por el navegador; permanece en la retransmisión del Gateway en iOS. `gateway-relay`: mantiene el audio del proveedor en el Gateway; Android solo usa el modo en tiempo real con este transporte.                                  |
| `realtime.brain`                         | -                                          | `agent-consult` enruta las llamadas de herramientas en tiempo real mediante la política del Gateway; `direct-tools` ofrece compatibilidad heredada con herramientas directas; `none` se usa para la transcripción/orquestación externa.                                                                                                 |
| `realtime.consultRouting`                | -                                          | `provider-direct` conserva la respuesta directa del proveedor cuando omite `openclaw_agent_consult`; `force-agent-consult` enruta en su lugar las transcripciones finalizadas del usuario mediante OpenClaw.                                                                                          |
| `realtime.instructions`                  | -                                          | Añade instrucciones del sistema orientadas al proveedor al prompt en tiempo real integrado de OpenClaw (estilo/tono de voz); se mantienen las indicaciones predeterminadas de `openclaw_agent_consult`.                                                                                                                |

`talk.catalog` expone los identificadores canónicos de los proveedores y los alias del registro, los modos, transportes, estrategias de razonamiento, formatos de audio en tiempo real e indicadores de capacidad válidos de cada proveedor, así como el resultado de disponibilidad seleccionado durante la ejecución. Los clientes Talk propios deben consultar ese catálogo en lugar de mantener localmente los alias de proveedores; un Gateway antiguo que omita la disponibilidad del grupo debe considerarse no verificado, en lugar de definitivamente no configurado. Los proveedores de transcripción por transmisión se descubren mediante `talk.catalog.transcription`; la retransmisión actual del Gateway usa la configuración del proveedor de transmisión de Voice Call hasta que se publique una superficie de configuración específica para la transcripción de Talk.

## Interfaz de macOS

- Interruptor de la barra de menús: **Talk**
- Pestaña de configuración: grupo **Talk Mode** (id. de voz + interruptor de interrupción)
- Superposición: el orbe representa la forma de onda universal de Talk (compartida con iOS, watchOS y Android). Durante la escucha, sigue el nivel del micrófono en directo; durante el habla, sigue la envolvente real de reproducción de TTS; durante el razonamiento, respira suavemente. Haga clic en el orbe para pausar/reanudar, haga doble clic para detener el habla y haga clic en X para salir del modo Talk.

## Interfaz de Android

- La navegación principal de Android consta de **Home**, **Chat** y **Settings**. La entrada de voz
  se encuentra en el editor de Chat, en lugar de en una pestaña Voice independiente.
- Toque el micrófono del editor para usar el dictado en el dispositivo. Manténgalo pulsado para grabar
  un archivo adjunto de nota de voz. Inicie Talk continuo desde la forma de onda de Talk.
- El dictado, la grabación de notas de voz y Talk son rutas de micrófono
  mutuamente excluyentes; iniciar una detiene o bloquea las demás.
- Talk en tiempo real prefiere el micrófono de unos auriculares Bluetooth Classic o BLE
  conectados; si se desconectan, la aplicación solicita otra entrada de auriculares o
  recurre al micrófono predeterminado y restablece la preferencia predeterminada cuando
  se detiene la captura.
- El dictado y la grabación de notas de voz se detienen cuando la aplicación deja de estar en primer plano o
  el usuario sale de Chat.
- Talk Mode continúa ejecutándose hasta que se desactiva o el Node se desconecta y, mientras está activo, usa el tipo de servicio en primer plano de micrófono de Android.
- Android admite los formatos de salida `pcm_16000`, `pcm_22050`, `pcm_24000` y `pcm_44100` para la transmisión `AudioTrack` de baja latencia.

## Notas

- Requiere permisos de voz y micrófono.
- Talk nativo usa la sesión activa del Gateway y solo recurre al sondeo del historial cuando los eventos de respuesta no están disponibles.
- El Gateway resuelve la reproducción de Talk mediante `talk.speak` usando el proveedor activo de Talk. Android solo recurre al TTS local del sistema cuando ese RPC no está disponible.
- La reproducción local de MLX en macOS usa el asistente `openclaw-mlx-tts` incluido cuando está presente, o un ejecutable en `PATH`. Establezca `OPENCLAW_MLX_TTS_BIN` para apuntar a un binario de asistente personalizado durante el desarrollo.
- Intervalos de valores de las directivas de voz (ElevenLabs): `stability`, `similarity` y `style` aceptan `0..1`; `speed` acepta `0.5..2`; `latency_tier` acepta `0..4`.

## Relacionado

- [Activación por voz](/es/nodes/voicewake)
- [Audio y notas de voz](/es/nodes/audio)
- [Comprensión de contenido multimedia](/es/nodes/media-understanding)
