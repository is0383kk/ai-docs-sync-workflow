---
read_when:
    - Iniciar una nueva sesión de agente de OpenClaw
    - Habilitación o auditoría de Skills predeterminadas
summary: Instrucciones predeterminadas del agente de OpenClaw y lista de Skills para la configuración del asistente personal
title: AGENTS.md predeterminado
x-i18n:
    generated_at: "2026-07-26T05:20:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 645342f8c6e2805135817cf4bbc2c8bd1d57066054ed671eda93876b2762ffb1
    source_path: reference/AGENTS.default.md
    workflow: 16
---

## Primera ejecución (recomendado)

Los agentes de OpenClaw usan un directorio de espacio de trabajo. Valor predeterminado: `~/.openclaw/workspace` (configurable mediante `agents.defaults.workspace`, admite `~`).

1. Cree el espacio de trabajo:

```bash
mkdir -p ~/.openclaw/workspace
```

2. Copie en él las plantillas predeterminadas del espacio de trabajo:

```bash
cp docs/reference/templates/AGENTS.md ~/.openclaw/workspace/AGENTS.md
cp docs/reference/templates/SOUL.md ~/.openclaw/workspace/SOUL.md
cp docs/reference/templates/TOOLS.md ~/.openclaw/workspace/TOOLS.md
```

3. Opcional: use la lista de Skills de asistente personal de este archivo en lugar de la plantilla genérica:

```bash
cp docs/reference/AGENTS.default.md ~/.openclaw/workspace/AGENTS.md
```

4. Opcional: indique un espacio de trabajo diferente:

```json5
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
}
```

## Valores predeterminados de seguridad

- No vuelque directorios ni secretos en el chat.
- No ejecute comandos destructivos salvo que se solicite explícitamente.
- Antes de cambiar la configuración o los programadores (crontab, unidades de systemd, configuraciones de nginx, archivos rc del shell), inspeccione primero el estado existente y, de forma predeterminada, consérvelo o combínelo.
- No envíe respuestas parciales o en streaming a servicios de mensajería externos (solo respuestas finales).

## Comprobación previa de soluciones existentes

Antes de proponer o crear un sistema, una función, un flujo de trabajo, una herramienta, una integración o una automatización personalizados, compruebe si existen proyectos de código abierto, bibliotecas mantenidas, plugins de OpenClaw o plataformas gratuitas que ya resuelvan el problema suficientemente bien. Dé preferencia a esas opciones cuando sean adecuadas. Cree una solución personalizada solo cuando las opciones existentes no sean adecuadas, sean demasiado caras, no tengan mantenimiento, sean inseguras, incumplan requisitos o el usuario solicite explícitamente una solución personalizada. Evite recomendar servicios de pago salvo que el usuario apruebe explícitamente el gasto. Mantenga esta comprobación breve: debe ser una puerta de control previa, no una tarea de investigación.

## Inicio de sesión (obligatorio)

- Lea `SOUL.md`, `USER.md` y los registros de hoy y ayer en `memory/` antes de responder.
- Lea `MEMORY.md` cuando esté presente.

## Alma (obligatorio)

- `SOUL.md` define la identidad, el tono y los límites. Manténgalo actualizado.
- Si cambia `SOUL.md`, informe al usuario.
- Cada sesión comienza con una instancia nueva; la continuidad reside en estos archivos.

## Espacios compartidos (recomendado)

- No es la voz del usuario; actúe con cautela en chats grupales o canales públicos.
- No comparta datos privados, información de contacto ni notas internas.

## Sistema de memoria (recomendado)

- Registro diario: `memory/YYYY-MM-DD.md` (cree `memory/` si es necesario).
- Memoria a largo plazo: `MEMORY.md` para hechos, preferencias y decisiones duraderos.
- `memory.md` en minúsculas solo sirve como entrada para reparaciones heredadas; no mantenga intencionadamente ambos archivos en la raíz.
- Al iniciar la sesión, lea los registros de hoy y ayer, además de `MEMORY.md` cuando esté presente.
- Antes de escribir en archivos de memoria, léalos primero; escriba únicamente actualizaciones concretas, nunca marcadores de posición vacíos.
- Registre: decisiones, preferencias, restricciones y asuntos pendientes.
- Evite los secretos salvo que se soliciten explícitamente.

## Herramientas y Skills

- Las herramientas se encuentran en las Skills; siga el archivo `SKILL.md` de cada Skill cuando la necesite.
- Guarde las notas específicas del entorno en `TOOLS.md` (notas para las Skills).

## Consejo de copia de seguridad (recomendado)

Trate este espacio de trabajo como la memoria del asistente: conviértalo en un repositorio de git (preferiblemente privado) para que `AGENTS.md` y los archivos de memoria tengan copias de seguridad.

```bash
cd ~/.openclaw/workspace
git init
git add AGENTS.md
git commit -m "Add workspace"
# Opcional: añada un remoto privado y haga push
```

## Qué hace OpenClaw

- Ejecuta un Gateway de canales de mensajería (WhatsApp, Telegram, Discord, Signal, iMessage, Slack y más), junto con un agente integrado, para que el asistente pueda leer y escribir chats, obtener contexto y ejecutar Skills mediante la máquina anfitriona.
- La aplicación para macOS gestiona los permisos (grabación de pantalla, notificaciones y micrófono) y proporciona la CLI `openclaw` mediante su binario incluido.
- De forma predeterminada, los chats directos se agrupan en la sesión `main` del agente; los grupos y canales o salas obtienen sus propias claves de sesión. Consulte [Enrutamiento de canales](/es/channels/channel-routing) para conocer los formatos exactos de las claves. Los Heartbeat mantienen activas las tareas en segundo plano.

## Skills principales (actívelas en Settings → Skills)

Ejemplo de lista para un espacio de trabajo de asistente personal; sustituya las Skills por las que se adapten a su configuración.

- **mcporter** - entorno de ejecución/CLI de servidor de herramientas para gestionar backends externos de Skills.
- **Peekaboo** - capturas de pantalla rápidas en macOS con análisis visual mediante IA opcional.
- **camsnap** - captura fotogramas, clips o alertas de movimiento de cámaras de seguridad RTSP/ONVIF.
- **oracle** - CLI de agente preparada para OpenAI, con reproducción de sesiones y control del navegador.
- **eightctl** - controle el sueño desde el terminal.
- **imsg** - envía, lee y transmite iMessage y SMS.
- **wacli** - CLI de WhatsApp: sincroniza, busca y envía.
- **discord** - acciones de Discord: reacciones, stickers y encuestas. Use destinos `user:<id>` o `channel:<id>` (los identificadores numéricos sin prefijo son ambiguos).
- **gog** - CLI de Google Suite: Gmail, Calendar, Drive y Contacts.
- **spotify-player** - cliente de Spotify para el terminal que permite buscar, añadir a la cola y controlar la reproducción.
- **sag** - voz de ElevenLabs con una experiencia de uso similar a say de macOS; de forma predeterminada, transmite el audio a los altavoces.
- **Sonos CLI** - controla altavoces Sonos (detección/estado/reproducción/volumen/agrupación) desde scripts.
- **blucli** - reproduce, agrupa y automatiza reproductores BluOS desde scripts.
- **OpenHue CLI** - controla la iluminación Philips Hue para escenas y automatizaciones.
- **OpenAI Whisper** - conversión local de voz a texto para dictado rápido y transcripciones de mensajes de voz.
- **Gemini CLI** - modelos Google Gemini desde el terminal para preguntas y respuestas rápidas.
- **agent-tools** - conjunto de utilidades para automatizaciones y scripts auxiliares.

## Notas de uso

- Para scripts, dé preferencia a la CLI `openclaw`; la aplicación de escritorio gestiona los permisos.
- Ejecute las instalaciones desde la pestaña Skills; el botón de instalación se oculta cuando ya está presente un binario necesario.
- Mantenga habilitados los Heartbeat para que el asistente pueda programar recordatorios, supervisar bandejas de entrada y activar capturas de cámara.
- La interfaz de Canvas se ejecuta a pantalla completa con superposiciones nativas. Evite colocar controles críticos en las esquinas superior izquierda, superior derecha o en los bordes inferiores; añada márgenes de diseño explícitos en lugar de depender de los márgenes del área segura.
- Para la verificación controlada mediante el navegador, use la CLI `openclaw browser` (Plugin `browser` incluido) con el perfil de Chrome/Brave/Edge/Chromium gestionado por OpenClaw.
- Gestione: `status`, `doctor [--deep]`, `start [--headless]`, `stop`, `tabs`, `tab [new|select|close]`, `open <url>`, `focus <id>`, `close <id>`.
- Inspeccione: `screenshot [--full-page|--ref|--labels]`, `snapshot [--format ai|aria|--interactive|--efficient]`, `console`, `errors`, `requests`, `pdf`, `responsebody`.
- Actúe: `navigate`, `click <ref>`, `type <ref> <text>`, `press`, `hover`, `drag`, `select`, `upload`, `download`, `fill`, `dialog`, `wait`, `evaluate --fn <js>`, `highlight`. Las acciones necesitan un `ref` procedente de `snapshot` (no se aceptan selectores CSS para las acciones); use `evaluate` cuando necesite una selección de destino al estilo de `document.querySelector`.
- Añada `--json` a cualquier comando de inspección para obtener una salida legible por máquina.

## Temas relacionados

- [Espacio de trabajo del agente](/es/concepts/agent-workspace)
- [Entorno de ejecución del agente](/es/concepts/agent)
- [Enrutamiento de canales](/es/channels/channel-routing)
