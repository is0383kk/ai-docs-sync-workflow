---
read_when:
    - Buscando el estado de la aplicación complementaria para Linux
    - Activación de la cámara, la ubicación o las notificaciones en un host Node con Linux
    - Planificación de la cobertura de plataformas o de las contribuciones
    - Depuración de cierres por OOM en Linux o de salidas con código 137 en un VPS o contenedor
summary: Compatibilidad con Linux + estado de la aplicación complementaria
title: Aplicación para Linux
x-i18n:
    generated_at: "2026-07-26T04:46:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fe55d3ec63fcf8291a24126c04638f005c03c3d44ff84a26a925e931066b01cc
    source_path: platforms/linux.md
    workflow: 16
---

El Gateway es totalmente compatible con Linux y requiere Node. Bun puede seguir utilizándose
como instalador de dependencias o ejecutor de scripts de paquetes, pero no puede ejecutar OpenClaw
porque no proporciona `node:sqlite`.

## Aplicación complementaria de escritorio

La aplicación complementaria de OpenClaw para Linux es una aplicación de escritorio Tauri para un Gateway local. Esta:

- instala la CLI de OpenClaw y el entorno de ejecución de Node administrado cuando no están presentes; las compilaciones de lanzamiento instalan automáticamente el canal estable, mientras que las compilaciones de desarrollo solicitan primero el canal
- se conecta a un Gateway en buen estado antes de intentar modificar el servicio
- delega las operaciones de instalación, inicio, detención y reinicio al servicio de usuario de systemd administrado por la CLI
- descubre Gateways Bonjour cercanos y abre la interfaz de control de cada uno en una ventana con ámbito de ruta, para que varios
  paneles de Gateway puedan permanecer conectados y utilizarse simultáneamente
- abre la interfaz de control servida por el Gateway con su URL de autenticación resuelta
- abre la interfaz de control en modo de incorporación tras la instalación de la primera ejecución, lo que
  permite importar las memorias detectadas de Claude Code, Codex o Hermes al
  espacio de trabajo del agente (la misma importación sigue disponible posteriormente en
  Settings → Import Memory)
- renderiza Canvas controlado por el agente y contenido A2UI incluido para un host de nodo de la CLI ubicado en el mismo equipo
- permanece disponible en la bandeja del sistema cuando se cierra su ventana

Las versiones estables compiladas desde `main` incluyen paquetes `.deb` y AppImage como recursos en la
[versión de GitHub](https://github.com/openclaw/openclaw/releases) correspondiente a la etiqueta,
con los nombres `OpenClaw-<version>-amd64.deb` y `OpenClaw-<version>-amd64.AppImage`,
junto con un archivo de suma de comprobación `SHA256SUMS.linux-app.txt`. Descargue
`.deb` e instálelo con `sudo apt install ./OpenClaw-<version>-amd64.deb`,
o marque el AppImage como ejecutable y ejecútelo directamente. El entorno de ejecución de AppImage
necesita FUSE 2 (`sudo apt install libfuse2`, o `libfuse2t64` en Ubuntu 24.04+);
sin él, ejecute el AppImage con `APPIMAGE_EXTRACT_AND_RUN=1`.

También se pueden compilar los mismos paquetes desde una copia de trabajo del código fuente:

```bash
cd apps/linux/src-tauri
pnpm dlx @tauri-apps/cli@2.11.4 build --bundles deb,appimage
```

El flujo de trabajo de CI `Linux App` carga los mismos paquetes como el
artefacto `openclaw-linux-companion` para los pull requests que afectan a la aplicación y para las
ejecuciones manuales. Consulte `apps/linux/README.md` en el repositorio para conocer las dependencias
de compilación de Linux y los comandos de desarrollo.

### Chat rápido

Abra el chat rápido con `Ctrl+Shift+Space` o el elemento **Quick Chat** de la bandeja. La ficha del agente
muestra el avatar, emoji o monograma configurado; selecciónela para cambiar de agente.
Los mensajes utilizan la sesión principal del agente seleccionado y respetan el ámbito global de sesión.
El cliente nativo de Rust posee una identidad persistente de dispositivo Ed25519. Utiliza el
token compartido o la contraseña de la transferencia de la CLI solo para iniciar el emparejamiento y, después, almacena y
prioriza el token de dispositivo emitido por el Gateway en conexiones posteriores. La identidad y el
token de dispositivo se encuentran en el directorio de configuración de la aplicación, en un archivo con modo `0600`; la WebView del chat
rápido no recibe ni las credenciales ni el WebSocket.

Cuando la conexión nativa no está disponible, el chat rápido muestra **No se puede acceder al Gateway — reintentando**
y desactiva el envío hasta que se restablezca la conexión. Un dispositivo remoto
que ha alcanzado la fase de emparejamiento muestra en su lugar **Apruebe este dispositivo en el panel
(Nodes)**, con un ID corto del dispositivo cuando el Gateway proporciona uno. Un
Gateway que requiere una credencial compartida ausente muestra **El Gateway requiere una
credencial — abra el panel en el host del Gateway**; en ese estado no hay ninguna solicitud de emparejamiento
pendiente de aprobación. Las instrucciones de corrección proporcionadas por el servidor
sustituyen estos avisos alternativos cuando son más específicas.
Para los Gateways TLS, la CLI entrega a la aplicación la huella digital SHA-256
del certificado del Gateway; el cliente nativo fija ese certificado e informa de **Error de
confianza TLS del Gateway — compruebe la huella digital del certificado** de forma independiente de la interrupción.
Los Gateways cuyo secreto compartido está configurado mediante un SecretRef lo omiten de la
transferencia de la CLI. Las instalaciones emparejadas existentes siguen funcionando mediante el token de dispositivo
almacenado, pero una instalación nueva no puede crear una solicitud de emparejamiento pendiente con autenticación
por secreto compartido sin esa credencial de arranque.
El canje del código de configuración y de `bootstrapToken` requiere una interfaz de producto específica y sigue
pendiente; el chat rápido no intenta ninguno de los dos flujos.

En X11, utilice el engranaje del chat rápido para registrar o restablecer un atajo personalizado. El
control de la bandeja **Quick Chat shortcut** lo activa o desactiva sin desactivar el
elemento normal **Quick Chat** de la bandeja. Los atajos globales no están disponibles en Wayland, por lo que
la configuración de atajos se oculta y el elemento de la bandeja sigue siendo el punto de entrada.
Tras aceptar un envío, el chat rápido permanece abierto y transmite la respuesta
de texto sin formato del agente seleccionado debajo del editor. Pulse `Esc` para cerrar la barra y su respuesta;
`Ctrl+Enter` sigue abriendo el panel.

### Canvas

Canvas para Linux utiliza dos procesos que cooperan entre sí. `openclaw node run` sigue siendo la única conexión de nodo del Gateway; el Plugin `linux-canvas` incluido reenvía las llamadas `canvas.*` a la aplicación de escritorio en ejecución mediante un socket Unix exclusivo del usuario. La aplicación posee una única ventana WebView bajo demanda, incluido el renderizador A2UI integrado y el puente de acciones de vuelta al agente.

El Plugin está activado de forma predeterminada. Solo anuncia Canvas cuando el socket de escritorio existe en `$XDG_RUNTIME_DIR/openclaw-canvas.sock`, o en `/tmp/openclaw-canvas-$UID.sock` cuando `XDG_RUNTIME_DIR` no está disponible. Desactívelo con `plugins.entries.linux-canvas.enabled: false`. En un servidor Linux sin interfaz gráfica y sin la aplicación de escritorio, Canvas no se anuncia.

Linux v1 utiliza una sola ventana de Canvas. Las páginas HTTP y HTTPS se pueden renderizar, pero las acciones A2UI solo se aceptan desde el renderizador incluido.

## Alternativa mediante CLI y SSH

La CLI sigue siendo la opción más sencilla para un servidor sin interfaz gráfica, un VPS o un Gateway remoto:

1. Instale Node 24.15+ (recomendado), Node 22.22.3+ (LTS) o Node 25.9+.
2. `npm i -g openclaw@latest`
3. `openclaw onboard --install-daemon`
4. Desde el portátil: `ssh -N -L 18789:127.0.0.1:18789 <user>@<host>`
5. Abra `http://127.0.0.1:18789/` y autentíquese con el secreto compartido configurado
   (token de forma predeterminada; contraseña si `gateway.auth.mode` es `"password"`).

Guía completa del servidor: [Servidor Linux](/es/vps). Ejemplo de VPS paso a paso:
[exe.dev](/es/install/exe-dev).

## Capacidades de Node

El Plugin de Node para Linux incluido proporciona a la CLI capacidades de dispositivo del servicio `openclaw node` sin requerir la aplicación de escritorio. Los comandos se anuncian al Gateway solo cuando su capacidad está activada y existe la herramienta local necesaria.

| Capacidad                               | Valor predeterminado | Requisito                                                             |
| --------------------------------------- | -------------------- | --------------------------------------------------------------------- |
| Notificaciones de escritorio (`system.notify`) | Activado             | `notify-send` de libnotify y una sesión de notificaciones de escritorio |
| Fotos y clips de la cámara (`camera.*`)    | Desactivado          | FFmpeg, acceso a la cámara V4L2 y PulseAudio o PipeWire para el audio de los clips |
| Ubicación (`location.get`)                   | Desactivado          | GeoClue2 y su demostración `where-am-i`                         |

Configure el Plugin en `openclaw.json`:

```json5
{
  plugins: {
    entries: {
      "linux-node": {
        config: {
          notify: { enabled: true },
          camera: { enabled: true },
          location: { enabled: true },
        },
      },
    },
  },
}
```

Reinicie el servicio del nodo después de cambiar esta configuración. La disponibilidad se determina una vez por proceso y el anuncio del nodo se vuelve a generar al reiniciarlo.

El Gateway aprueba la superficie de comandos y capacidades del nodo por separado del emparejamiento del dispositivo. En el primer inicio, o después de activar más capacidades, apruebe la superficie pendiente:

```bash
openclaw nodes pending
openclaw nodes approve <requestId>
```

Un nodo puede estar conectado y emparejado con el dispositivo mientras sus valores efectivos `caps` y `commands` permanecen vacíos hasta que finalice esta aprobación.

Los dispositivos de cámara deben ser legibles por el usuario del servicio, normalmente mediante el grupo `video`. Los clips de cámara utilizan la fuente predeterminada de PulseAudio o PipeWire cuando `includeAudio` es true; el audio del micrófono existe únicamente como esa pista del clip, no como un comando independiente. La ubicación requiere que la política GeoClue del host permita al usuario del servicio del nodo utilizarla.

`camera.snap` y `camera.clip` también requieren la activación explícita del Gateway mediante `gateway.nodes.commands.allow`. Consulte [Captura con la cámara](/es/nodes/camera) y [Comando de ubicación](/es/nodes/location-command) para conocer las cargas útiles, los límites y los errores.

## Instalación

- [Primeros pasos](/es/start/getting-started)
- [Instalación y actualizaciones](/es/install/updating)
- Opcional: [Flujo de trabajo de paquetes con Bun](/es/install/bun), [Nix](/es/install/nix), [Docker](/es/install/docker)

## Servicio del Gateway (systemd)

Instálelo con una de las siguientes opciones:

```bash
openclaw onboard --install-daemon
openclaw gateway install
openclaw configure   # select "Gateway service" when prompted
```

Repare o migre una instalación existente:

```bash
openclaw doctor
```

`openclaw gateway install` genera de forma predeterminada una unidad de **usuario** de systemd. La guía
completa del servicio, incluida la variante de unidad de nivel de **sistema** para hosts compartidos o
siempre activos, se encuentra en el [manual operativo del Gateway](/es/gateway#supervision-and-service-lifecycle).

Escriba una unidad manualmente solo para una configuración personalizada. Ejemplo mínimo de unidad de usuario
(`~/.config/systemd/user/openclaw-gateway[-<profile>].service`):

```ini
[Unit]
Description=OpenClaw Gateway (profile: <profile>, v<version>)
After=network-online.target
Wants=network-online.target
StartLimitBurst=5
StartLimitIntervalSec=60

[Service]
ExecStart=/usr/local/bin/openclaw gateway --port 18789
Restart=always
RestartSec=5
RestartPreventExitStatus=78
TimeoutStopSec=30
TimeoutStartSec=30
SuccessExitStatus=0 143
OOMPolicy=continue
KillMode=control-group

[Install]
WantedBy=default.target
```

Las unidades escritas manualmente no heredan el ajuste adaptativo del montón que `openclaw gateway install` escribe para los servicios de Gateway administrados. Se recomienda utilizar el instalador administrado o establecer un límite explícito del montón en el supervisor personalizado tras tener en cuenta el margen de memoria nativa.

Actívelo:

```bash
systemctl --user enable --now openclaw-gateway[-<profile>].service
```

## Presión de memoria y finalizaciones por OOM

En Linux, el kernel elige una víctima de OOM cuando un host, una máquina virtual o el cgroup de un contenedor
se queda sin memoria. El Gateway es una mala víctima porque mantiene sesiones
y conexiones de canales de larga duración, por lo que OpenClaw favorece que los procesos secundarios
transitorios se finalicen primero cuando sea posible.

Para los procesos secundarios de Linux que cumplen los requisitos, OpenClaw envuelve el comando en un pequeño
shim `/bin/sh` que eleva el valor `oom_score_adj` del propio proceso secundario a `1000` y, a continuación,
aplica `exec` al comando real. Esto no requiere privilegios: un proceso siempre puede elevar
su propia puntuación de OOM.

Superficies de procesos secundarios cubiertas:

- Procesos secundarios de comandos administrados por el supervisor
- Procesos secundarios del shell PTY
- Procesos secundarios del servidor stdio de MCP
- Procesos de navegador/Chrome iniciados por OpenClaw (mediante el entorno de ejecución de procesos del SDK del Plugin)

El envoltorio solo funciona en Linux y se omite cuando `/bin/sh` no está disponible, o cuando
el entorno del proceso secundario establece `OPENCLAW_CHILD_OOM_SCORE_ADJ` en `0`, `false`, `no` o
`off`.

Verifique un proceso secundario:

```bash
cat /proc/<child-pid>/oom_score_adj
```

El valor esperado para los procesos secundarios cubiertos es `1000`; el propio proceso del Gateway
mantiene su puntuación normal (normalmente `0`).

El valor `OOMPolicy=continue` de la unidad de systemd mantiene activo el servicio del Gateway cuando
el eliminador de OOM selecciona un proceso secundario transitorio, en lugar de marcar toda la
unidad como fallida y reiniciar todos los canales; el proceso secundario o la sesión que falla informa de su
propio error.

Esto no sustituye al ajuste normal de memoria. Si un VPS o contenedor finaliza repetidamente
procesos secundarios, aumente el límite de memoria, reduzca la concurrencia o añada controles
de recursos más estrictos (`MemoryMax=` de systemd, límites de memoria del contenedor).

## Contenido relacionado

- [Descripción general de la instalación](/es/install)
- [Servidor Linux](/es/vps)
- [Raspberry Pi](/es/install/raspberry-pi)
- [Guía operativa del Gateway](/es/gateway)
- [Configuración del Gateway](/es/gateway/configuration)
