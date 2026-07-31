---
read_when:
    - Quiere que un agente de OpenClaw se una a una reunión por vídeo
    - Está eligiendo entre los plugins de reuniones de Google Meet, Microsoft Teams y Zoom
    - Necesita la configuración compartida de Chrome, BlackHole, SoX o del modo de reunión
summary: Elige y configura la participación en reuniones de Google Meet, Microsoft Teams o Zoom
title: Plugins de reuniones
x-i18n:
    generated_at: "2026-07-26T04:43:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f41488de018402e3d5cfd01fa5351cdb6107412477d5d54e2d9e186e0fc8ee94
    source_path: plugins/meeting-plugins.md
    workflow: 16
---

OpenClaw tiene plugins independientes para Google Meet, las reuniones de Microsoft Teams y Zoom. Los tres pueden unirse mediante Chrome, usar los mismos modos de participación y ejecutar Chrome en el host del Gateway o en un nodo emparejado. Sus URL de plataforma, modelo de instalación y capacidades adicionales son diferentes.

Estos plugins participan en reuniones. Son independientes de los canales de mensajería, como el [canal de Microsoft Teams](/es/channels/msteams), y del [plugin de llamadas de voz](/es/plugins/voice-call).

## Elegir un plugin

| Plataforma      | Plugin                                      | Enlaces de reunión aceptados                                                                                  | Instalación                                           | Formas de participación                                  | Capacidades específicas de la plataforma                                                                                   |
| --------------- | ------------------------------------------- | ------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------- | -------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| Google Meet     | [`google-meet`](/es/plugins/google-meet)       | `meet.google.com/...`                                                                                       | Instalar desde npm o ClawHub; activado de forma predeterminada | Chrome local, Chrome en un nodo emparejado o acceso telefónico mediante Twilio | Puede crear reuniones mediante la API de Meet o un navegador con una sesión iniciada; puede leer artefactos compatibles de Meet mediante OAuth |
| Microsoft Teams | [`teams-meetings`](/es/plugins/teams-meetings) | Enlaces de trabajo en `teams.microsoft.com/l/meetup-join/...` y enlaces para consumidores en `teams.live.com/meet/...` | Incluido; activado de forma predeterminada            | Chrome local o Chrome en un nodo emparejado              | Acceso como invitado a reuniones de trabajo y para consumidores                                                          |
| Zoom            | [`zoom-meetings`](/es/plugins/zoom-meetings)   | `zoom.us/j/...` y subdominios de cuenta como `example.zoom.us/j/...`                                      | Incluido; activado de forma predeterminada            | Chrome local o Chrome en un nodo emparejado              | Acceso como invitado mediante Zoom Web App                                                                                 |

Elija Google Meet cuando necesite crear reuniones, utilizar artefactos de la API de Google o acceder por teléfono mediante Twilio. Elija Teams o Zoom para la participación directa como invitado mediante el navegador en esas plataformas. Los plugins de Teams y Zoom no crean reuniones, no acceden por teléfono, no llaman a la API del proveedor ni capturan grabaciones de audio o vídeo.

## Elegir un modo

Los tres plugins comparten los mismos modos:

| Modo         | Comportamiento                                                                                                     | Requisitos de audio                                              |
| ------------ | ------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------- |
| `agent`      | La transcripción en tiempo real se envía al agente de OpenClaw configurado; el TTS habitual de OpenClaw reproduce la respuesta. | La respuesta de voz de Chrome requiere el puente de BlackHole y SoX. |
| `bidi`       | Un modelo de voz en tiempo real escucha y responde directamente.                                                   | La respuesta de voz de Chrome requiere el puente de BlackHole y SoX. |
| `transcribe` | Se une solo como observador y expone una transcripción acotada de subtítulos en directo cuando la plataforma proporciona subtítulos. | No se requiere el puente de respuesta de voz de BlackHole ni SoX.   |

Use `transcribe` cuando el agente solo necesite el texto de la reunión. Use `agent` para el razonamiento y las herramientas habituales de OpenClaw. Use `bidi` cuando la voz directa de baja latencia sea más importante que encaminar cada turno mediante el agente habitual.

La transcripción en directo acotada permanece disponible únicamente en el modo `transcribe`. En los
tres modos, las incorporaciones mediante navegador también conservan las filas de subtítulos completadas y un
resumen derivado en la base de datos de estado compartida. Al salir de la reunión, se finalizan los
subtítulos visibles y se escribe el resumen; use [`openclaw transcripts`](/es/cli/transcripts)
para enumerarlo, inspeccionarlo o exportarlo. Esta ruta de notas persistentes no modifica la transcripción en directo
consultada por el agente ni crea una grabación de audio o vídeo.

Las notas automáticas están activadas de forma predeterminada. Establezca `transcripts.enabled: false` para desactivar
globalmente las notas persistentes. Una sesión `transcribe` seleccionada explícitamente conserva su
tramo final acotado de subtítulos en directo sin escribir filas persistentes. La disponibilidad de los subtítulos
sigue dependiendo de la plataforma de reuniones, la cuenta, el idioma y la política del anfitrión.

## Preparar Chrome y el audio

Chrome puede ejecutarse en el host del Gateway o en un nodo emparejado. Un nodo de Chrome remoto debe permitir `browser.proxy` además del comando de la plataforma:

| Plugin          | Comando del nodo       |
| --------------- | ---------------------- |
| Google Meet     | `googlemeet.chrome`    |
| Microsoft Teams | `teamsmeetings.chrome` |
| Zoom            | `zoommeetings.chrome`  |

Para el modo `agent` o `bidi` mediante Chrome, ejecute Chrome en macOS e instale las dependencias de audio compartidas en ese mismo host:

```bash
brew install blackhole-2ch sox
sudo reboot
system_profiler SPAudioDataType | grep -i BlackHole
command -v sox
```

El host del Gateway sigue alojando el agente de OpenClaw y las credenciales del modelo cuando Chrome se ejecuta en un nodo emparejado. Configure un proveedor de transcripción en tiempo real y el TTS de OpenClaw para el modo `agent`, o un proveedor de voz en tiempo real para el modo `bidi`. Las guías de las plataformas contienen las opciones del proveedor y de los comandos de audio.

## Instalar o desactivar plugins

Instale Google Meet por separado; queda activado de forma predeterminada después de la instalación. Las reuniones de Teams y Zoom se incluyen con OpenClaw y están activadas de forma predeterminada:

```bash
# Solo Google Meet
openclaw plugins install npm:@openclaw/google-meet
```

Desactive los plugins de reuniones que no utilice:

```bash
openclaw plugins disable google-meet
openclaw plugins disable teams-meetings
openclaw plugins disable zoom-meetings
```

Reinicie el Gateway si la ruta de administración de plugins no lo reinicia automáticamente. A continuación, ejecute la comprobación de configuración de la plataforma antes de unirse.

## Verificar y unirse

| Plataforma      | Comprobación de configuración | Comando para unirse                                                          |
| --------------- | ------------------------------ | ----------------------------------------------------------------------------- |
| Google Meet     | `openclaw googlemeet setup`    | `openclaw googlemeet join 'https://meet.google.com/abc-defg-hij'`             |
| Microsoft Teams | `openclaw teamsmeetings setup` | `openclaw teamsmeetings join 'https://teams.microsoft.com/l/meetup-join/...'` |
| Zoom            | `openclaw zoommeetings setup`  | `openclaw zoommeetings join 'https://zoom.us/j/1234567890'`                   |

Considere cualquier fallo en la comprobación de configuración como un impedimento para ese transporte y modo. Para una prueba rápida de solo observación, seleccione el modo `transcribe` y confirme que el estado indique una sesión con una llamada en curso antes de esperar texto de subtítulos.

Para las pruebas rápidas de respuesta de voz, verificar el habla requiere algo más que la aceptación de bytes por parte del comando de reproducción. El puente compartido de pares de comandos correlaciona una huella de forma de onda acotada de la generación de salida actual con el audio que regresa por la ruta de captura del micrófono de BlackHole; Google Meet, Teams y Zoom no indican `speechOutputVerified: true` cuando solo aumenta el contador de bytes de salida o hay audio no relacionado de otros participantes.

## Gestionar las solicitudes de políticas de la plataforma

La automatización del navegador gestiona el nombre habitual del invitado, los controles de cámara y micrófono previos a la incorporación, así como los controles para unirse, permanecer en la llamada y salir. No elude las políticas de la plataforma ni del organizador.

- Google Meet puede requerir iniciar sesión en Google, la admisión del anfitrión o una decisión sobre permisos del navegador.
- Microsoft Teams puede requerir iniciar sesión en el inquilino, verificar el correo electrónico o recibir la admisión del organizador.
- Zoom puede requerir autenticación, verificación del correo electrónico, un código de acceso, completar un CAPTCHA o la admisión del anfitrión; una cuenta también puede desactivar la incorporación mediante navegador.

Cuando el resultado de una incorporación o del estado indique `manualActionRequired`, complete el paso indicado en el mismo perfil de Chrome de OpenClaw antes de volver a intentarlo. Abrir repetidamente pestañas nuevas no resuelve un bloqueo de cuenta, inquilino, sala de espera o CAPTCHA.

Únase únicamente a reuniones en las que el operador esté autorizado a añadir un agente. Informe a los participantes cuando las políticas locales o las normas de consentimiento exijan revelar la participación automatizada, la transcripción o el habla sintetizada.

## Chat de voz de Discord

Los [canales de voz de Discord](/es/channels/discord#voice-channels) proporcionan conversaciones nativas en tiempo real, exclusivamente de audio y sin automatización de reuniones mediante navegador. OpenClaw puede unirse a un canal de voz, escuchar, encaminar los turnos mediante un agente de OpenClaw o un modelo de voz en tiempo real y reproducir las respuestas. No envía ni recibe vídeo de cámara ni comparte la pantalla, aunque las personas utilicen vídeo en el mismo canal de Discord, por lo que la voz de Discord es una superficie relacionada de conversación en directo, no un cuarto plugin de reuniones mediante navegador.

## Guías de las plataformas

- [Plugin de Google Meet](/es/plugins/google-meet)
- [Plugin de reuniones de Microsoft Teams](/es/plugins/teams-meetings)
- [Plugin de reuniones de Zoom](/es/plugins/zoom-meetings)
- [Administrar plugins](/es/plugins/manage-plugins)
- [Control del navegador](/es/tools/browser)
