---
read_when:
    - Quieres que un agente de OpenClaw se una a una reunión de Zoom
    - Está configurando Chrome, BlackHole o SoX para la comunicación bidireccional en reuniones de Zoom
summary: 'Plugin de reuniones de Zoom: únete a reuniones como invitado desde el navegador Chrome'
title: Plugin de reuniones de Zoom
x-i18n:
    generated_at: "2026-07-26T04:50:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d91e57cccb163f634c6eaee71dd3832fc7b9e783fc5cd02601572b302d0d25e8
    source_path: plugins/zoom-meetings.md
    workflow: 16
---

El plugin `zoom-meetings` se une como invitado a enlaces de reuniones de Zoom mediante la aplicación web de Zoom en el perfil de Chrome de OpenClaw. Acepta enlaces de reuniones en `zoom.us/j/...` y subdominios de cuentas como `example.zoom.us/j/...`. No crea reuniones, no se conecta por teléfono, no utiliza el SDK de reuniones de Zoom ni captura grabaciones de audio o vídeo.

## Configuración

La respuesta por voz utiliza los mismos requisitos previos de audio local que el [plugin de Google Meet](/es/plugins/google-meet): macOS, el dispositivo de audio virtual `BlackHole 2ch` y SoX.

```bash
brew install blackhole-2ch sox
sudo reboot
system_profiler SPAudioDataType | grep -i BlackHole
command -v sox
```

El plugin está incluido y habilitado de forma predeterminada. Añada una entrada solo para personalizarlo y, después, compruebe la configuración:

```json5
{
  plugins: {
    entries: {
      "zoom-meetings": {
        config: {
          defaultMode: "agent",
          chrome: { guestName: "OpenClaw Agent" },
        },
      },
    },
  },
}
```

Ejecute `openclaw plugins disable zoom-meetings` si no desea que el plugin esté activo.

```bash
openclaw zoommeetings setup
openclaw zoommeetings join 'https://zoom.us/j/1234567890'
```

Utilice `chromeNode.node` para ejecutar Chrome, BlackHole y SoX en un Node macOS emparejado. El Node debe permitir `zoommeetings.chrome` y `browser.proxy`.

## Modos

| Modo         | Comportamiento                                                                    |
| ------------ | --------------------------------------------------------------------------- |
| `agent`      | La transcripción en tiempo real consulta al agente de OpenClaw configurado; TTS responde. |
| `bidi`       | Un modelo de voz en tiempo real escucha y responde directamente.                        |
| `transcribe` | Unión solo para observación con instantáneas de la transcripción de subtítulos en directo.                   |

Los subtítulos en directo de Zoom se habilitan después de la admisión en todos los modos para que OpenClaw pueda
conservar las notas de la reunión. La acción `transcript` sigue devolviendo el búfer en directo
limitado solo para las sesiones `transcribe`. Al salir, OpenClaw almacena la
transcripción persistente y el resumen derivado en la base de datos de estado compartida; enumérelos o expórtelos
con [`openclaw transcripts`](/es/cli/transcripts).

Las notas automáticas están habilitadas de forma predeterminada. Establezca `transcripts.enabled: false` para
deshabilitar las notas persistentes de forma global; el modo `transcribe` explícito sigue exponiendo solo
su tramo final en directo limitado.

## Limitaciones de la unión como invitado

El adaptador del navegador elige **Join from browser**, introduce el nombre del invitado, apaga la cámara, configura el micrófono para el modo seleccionado y hace clic en **Join**. La aplicación web de Zoom se ejecuta en `app.zoom.us`; el plugin concede a ese origen permisos de micrófono y selección de altavoz antes de la navegación. El estado durante la llamada utiliza el control Leave de Zoom. Los estados de sala de espera, inicio de sesión, código de acceso, CAPTCHA y permisos del dispositivo devuelven motivos explícitos que requieren una acción manual.

Las políticas del anfitrión y de la cuenta de Zoom pueden deshabilitar la unión desde el navegador, exigir autenticación o verificación por correo electrónico, mostrar un CAPTCHA o requerir la admisión del anfitrión. Complete ese paso en el perfil de Chrome de OpenClaw y, después, vuelva a consultar el estado o a intentar la intervención por voz. El plugin no elude las políticas de Zoom.

La aplicación web de Zoom se ha validado en vivo con una reunión de prueba oficial de Zoom para comprobar la pantalla intermedia de la aplicación, la introducción del nombre del invitado en un iframe, los controles del micrófono y la cámara previos a la unión, la unión, los permisos multimedia del navegador y de macOS, la detección durante la llamada, la habilitación de subtítulos en directo y la detección de la finalización por parte del anfitrión. Los estados de sala de espera y autenticación dependen de las políticas del anfitrión y mantienen alternativas basadas en texto cuando no hay disponible un identificador estable del DOM.

## Superficie de herramientas y del Gateway

La herramienta de agente `zoom_meetings` admite `join`, `leave`, `status`, `transcript` y `speak`. Los métodos del Gateway utilizan el prefijo `zoommeetings.*`. El comando del Node es `zoommeetings.chrome`.

## Contenido relacionado

- [Descripción general de los plugins de reuniones](/es/plugins/meeting-plugins)
