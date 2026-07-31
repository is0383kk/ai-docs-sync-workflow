---
read_when:
    - Quieres que un agente de OpenClaw se una a una reunión de Microsoft Teams
    - Está configurando Chrome, BlackHole o SoX para la comunicación bidireccional en reuniones de Teams
summary: 'Plugin de reuniones de Microsoft Teams: únete a reuniones de trabajo o de consumidores como invitado mediante el navegador Chrome'
title: Plugin de reuniones de Microsoft Teams
x-i18n:
    generated_at: "2026-07-26T04:50:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6f84e58d478185d026dd79a02a8500af48f51689ef6865d56badb0e27c6d2814
    source_path: plugins/teams-meetings.md
    workflow: 16
---

El plugin `teams-meetings` se une como invitado a enlaces de Microsoft Teams en el perfil de Chrome de OpenClaw. Acepta enlaces de trabajo bajo `teams.microsoft.com/l/meetup-join/...` y enlaces de consumidores bajo `teams.live.com/meet/...`. No crea reuniones, no se conecta por teléfono, no llama a Microsoft Graph ni captura grabaciones de audio o vídeo.

## Configuración

La respuesta de voz usa los mismos requisitos previos de audio local que el [plugin de Google Meet](/es/plugins/google-meet): macOS, el dispositivo de audio virtual `BlackHole 2ch` y SoX.

```bash
brew install blackhole-2ch sox
sudo reboot
system_profiler SPAudioDataType | grep -i BlackHole
command -v sox
```

El plugin está incluido y habilitado de forma predeterminada. Añada una entrada solo para personalizarlo y, a continuación, compruebe la configuración:

```json5
{
  plugins: {
    entries: {
      "teams-meetings": {
        config: {
          defaultMode: "agent",
          chrome: { guestName: "OpenClaw Agent" },
        },
      },
    },
  },
}
```

Ejecute `openclaw plugins disable teams-meetings` si no desea que el plugin esté activo.

```bash
openclaw teamsmeetings setup
openclaw teamsmeetings join 'https://teams.microsoft.com/l/meetup-join/...'
```

Use `chromeNode.node` para ejecutar Chrome, BlackHole y SoX en un nodo macOS emparejado. El nodo debe permitir `teamsmeetings.chrome` y `browser.proxy`.

## Modos

| Modo         | Comportamiento                                                                    |
| ------------ | --------------------------------------------------------------------------- |
| `agent`      | La transcripción en tiempo real consulta al agente de OpenClaw configurado; TTS responde. |
| `bidi`       | Un modelo de voz en tiempo real escucha y responde directamente.                        |
| `transcribe` | Unión de solo observación con instantáneas de la transcripción de subtítulos en directo.                   |

Los subtítulos en directo de Teams se habilitan después de la admisión en todos los modos para que OpenClaw pueda
conservar notas atribuidas a los hablantes. La acción `transcript` sigue devolviendo
solo el búfer en directo limitado para las sesiones `transcribe`. Al salir, OpenClaw almacena
la transcripción persistente y el resumen derivado en la base de datos de estado compartida; se pueden enumerar
o exportar con [`openclaw transcripts`](/es/cli/transcripts).

Las notas automáticas están habilitadas de forma predeterminada. Establezca `transcripts.enabled: false` para
deshabilitar globalmente las notas persistentes; el modo explícito `transcribe` sigue exponiendo solo
su tramo final en directo limitado.

## Límites de la unión como invitado

El adaptador del navegador descarta la pantalla intermedia de la aplicación, rellena el nombre del invitado, apaga la cámara, configura el micrófono para el modo seleccionado y hace clic en el botón de unión. El estado durante la llamada usa el control para colgar; los estados de sala de espera, inicio de sesión del inquilino y permisos del dispositivo devuelven motivos explícitos que requieren una acción manual. Se admiten las redirecciones del iniciador de reuniones para consumidores y las etiquetas `BlackHole 2ch (Virtual)` que muestra Chrome.

La política del inquilino de Teams puede exigir el inicio de sesión, la verificación del correo electrónico o la admisión por parte del organizador. Complete ese paso en el perfil de Chrome de OpenClaw y, a continuación, vuelva a consultar el estado o a intentar la intervención de voz. El plugin no elude la política del inquilino.

El cliente web de Teams para consumidores se ha validado en vivo para la pantalla intermedia de la aplicación, la introducción del nombre de invitado, los controles de micrófono y cámara previos a la unión, la unión, la admisión desde la sala de espera, los permisos multimedia, la detección de llamada en curso, los subtítulos en directo, el enrutamiento de entrada y salida de BlackHole, la salida y la detección posterior a la llamada. Los inquilinos de trabajo pueden imponer políticas diferentes de inicio de sesión, verificación del correo electrónico, admisión y confirmación de salida; complete cualquier acción manual indicada en el perfil de Chrome de OpenClaw.

## Superficie de herramientas y del Gateway

La herramienta de agente `teams_meetings` admite `join`, `leave`, `status`, `transcript` y `speak`. Los métodos del Gateway usan el prefijo `teamsmeetings.*`. El comando del nodo es `teamsmeetings.chrome`.

## Contenido relacionado

- [Descripción general de los plugins de reuniones](/es/plugins/meeting-plugins)
- [Canal de Microsoft Teams](/es/channels/msteams)
