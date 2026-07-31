---
read_when:
    - Trabajo en las vías de activación por voz o PTT
summary: Modos de activación por voz y pulsar para hablar, además de detalles de enrutamiento en la aplicación para Mac
title: Activación por voz (macOS)
x-i18n:
    generated_at: "2026-07-26T05:11:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d3b2a01ee997b4158bf88b9ef54b1e523503722620f943d594323516619e7502
    source_path: platforms/mac/voicewake.md
    workflow: 16
---

# Activación por voz y pulsar para hablar

## Requisitos

La activación por voz y la función de pulsar para hablar requieren macOS 26 o posterior. En versiones anteriores de macOS, los controles se ocultan de la página de configuración de voz, que muestra en su lugar el requisito de macOS 26.

La activación por voz requiere que Apple Speech admita el reconocimiento en el dispositivo para el idioma seleccionado. La aplicación se niega a iniciar la escucha pasiva de la palabra de activación cuando ese requisito de funcionamiento exclusivamente local no está disponible; nunca recurre al reconocimiento por red. Pulsar para hablar, el modo de conversación y el dictado de chat rápido son acciones explícitas del usuario y pueden utilizar los servicios de red de Apple Speech para ofrecer una cobertura de idiomas más amplia.

## Modos

- **Modo de palabra de activación** (predeterminado): un reconocedor de voz en el dispositivo y siempre activo espera los tokens de activación (`swabbleTriggerWords`). Cuando detecta una coincidencia, inicia la captura, muestra la superposición con el texto parcial y envía automáticamente después de un periodo de silencio.
- **Pulsar para hablar (mantener pulsada la tecla Opción derecha)**: mantenga pulsada la tecla Opción derecha para iniciar la captura de inmediato, sin necesidad de una palabra de activación. La superposición aparece mientras se mantiene pulsada; al soltarla, se finaliza y reenvía el texto tras una breve demora para que pueda editarlo.

## Comportamiento en tiempo de ejecución (palabra de activación)

- El reconocedor reside en `VoiceWakeRuntime`.
- La activación solo se produce cuando hay una pausa significativa entre la palabra de activación y la siguiente palabra (`triggerPauseWindow` = 0.55s). La superposición o el sonido pueden iniciarse durante la pausa, incluso antes de que comience el comando.
- Intervalos de silencio: 2.0s (`silenceWindow`) cuando el habla continúa, 5.0s (`triggerOnlySilenceWindow`) si solo se ha oído la palabra de activación.
- Límite estricto: 120s (`captureHardStop`) para evitar sesiones descontroladas.
- Antirrebote entre sesiones: 350ms (`debounceAfterSend`) después de un envío.
- La superposición se controla mediante `VoiceWakeOverlayController`, con colores distintos para el texto confirmado y el texto provisional.
- Tras el envío, el reconocedor se reinicia de forma limpia para escuchar la siguiente palabra de activación.

## Invariantes del ciclo de vida

- Si la activación por voz está habilitada y se han concedido los permisos, el reconocedor de la palabra de activación permanece a la escucha, excepto durante una captura activa de pulsar para hablar.
- Al cerrar la superposición, incluso manualmente mediante el botón X, siempre se reanuda el reconocedor: `VoiceSessionCoordinator.overlayDidDismiss` llama a `VoiceWakeRuntime.refresh(state:)` en todas las rutas de cierre. Consulte [Superposición de voz](/es/platforms/mac/voice-overlay) para obtener información sobre el modelo de sesión y tokens.

## Detalles de pulsar para hablar

- La detección de la tecla de acceso rápido utiliza un monitor global `.flagsChanged` para la tecla Opción derecha (`keyCode 61` + `.option`). Solo observa los eventos, nunca los intercepta.
- La captura reside en `VoicePushToTalk`: inicia Speech de inmediato, transmite los resultados parciales a la superposición y llama a `VoiceWakeForwarder` al soltar la tecla.
- Al iniciar pulsar para hablar, se pausa el entorno de ejecución de la palabra de activación para evitar capturas de audio simultáneas; se reinicia automáticamente al soltar la tecla.
- Permisos: requiere acceso al micrófono y a Speech; para recibir eventos de teclado se necesita la aprobación de accesibilidad o monitorización de entrada.
- Teclados externos: algunos no exponen la tecla Opción derecha como se espera. Ofrezca un atajo alternativo si los usuarios informan de fallos de detección.

## Configuración visible para el usuario

- Interruptor **Activación por voz**: habilita el entorno de ejecución de la palabra de activación.
- **Mantener pulsada la tecla Opción derecha para hablar**: habilita el monitor de pulsar para hablar.
- Si el idioma seleccionado no admite el reconocimiento en el dispositivo en este Mac, la activación por voz permanece deshabilitada, mientras que pulsar para hablar y el modo de conversación siguen disponibles.
- Selectores de idioma y micrófono, un medidor de nivel en tiempo real, una tabla de palabras de activación y una herramienta de prueba (exclusivamente local, nunca reenvía).
- El selector de micrófono conserva la última selección si se desconecta un dispositivo, muestra un aviso de desconexión y utiliza temporalmente el dispositivo predeterminado del sistema hasta que el dispositivo vuelva a estar disponible.
- **Sonidos**: se reproducen al detectar la palabra de activación y al enviar, usando de forma predeterminada el sonido del sistema "Glass" de macOS. Seleccione cualquier archivo que pueda cargar `NSSound` (por ejemplo, MP3/WAV/AIFF) para cada evento, o elija **Sin sonido**.

## Comportamiento del reenvío

- Al reenviar, `VoiceWakeForwarder.selectedSessionOptions` selecciona la clave de sesión activa de WebChat si se ha establecido una; de lo contrario, utiliza la clave de sesión principal del Gateway.
- Busca esa sesión mediante `sessions.list` y obtiene el canal y el destino de entrega a partir del contexto de entrega de la sesión (recurriendo primero a su último canal y destino y, después, a una clave de sesión analizada); si no se puede resolver nada, utiliza WebChat de forma predeterminada.
- Si la entrega falla, el error se registra (categoría `voicewake.forward`) y la ejecución sigue siendo visible mediante WebChat o los registros de sesión.

## Carga útil del reenvío

- `VoiceWakeForwarder.prefixedTranscript(_:)` antepone a la transcripción una línea con una indicación del equipo (el nombre de host resuelto o, si no está disponible, "este Mac"), compartida entre las rutas de la palabra de activación y de pulsar para hablar.

## Verificación rápida

- Active pulsar para hablar, mantenga pulsada la tecla Opción derecha, hable y suéltela: la superposición debería mostrar los resultados parciales y después enviarlos.
- Mientras se mantiene pulsada, las orejas de la barra de menús deberían permanecer ampliadas (`triggerVoiceEars(ttl: nil)`); vuelven a su tamaño normal al soltarla.

## Contenido relacionado

- [Activación por voz](/es/nodes/voicewake)
- [Superposición de voz](/es/platforms/mac/voice-overlay)
- [Aplicación para macOS](/es/platforms/macos)
