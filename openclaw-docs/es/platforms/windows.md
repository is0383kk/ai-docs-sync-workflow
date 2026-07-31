---
read_when:
    - Instalación de OpenClaw en Windows
    - Elegir entre Windows Hub, Windows nativo y WSL2
    - Configuración de la aplicación complementaria de Windows o del modo Node de Windows
summary: 'Compatibilidad con Windows: Hub de Windows, CLI y Gateway nativos, configuración del Gateway en WSL2, modo Node y solución de problemas'
title: Windows
x-i18n:
    generated_at: "2026-07-26T04:47:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c231b81971e1df9f3ee4de1b102c25328c242109331c6465dc802ec003af722b
    source_path: platforms/windows.md
    workflow: 16
---

OpenClaw incluye una aplicación complementaria nativa **Windows Hub**, además de compatibilidad con la CLI en Windows.
Use Windows Hub como aplicación de escritorio con configuración, estado en la bandeja, chat, diagnósticos de Command
Center y capacidades del Node de Windows. Use el instalador de PowerShell
directamente para la CLI y el Gateway. Use WSL2 para obtener el entorno de ejecución del Gateway
más compatible con Linux.

## Recomendado: Windows Hub

Windows Hub es la aplicación complementaria nativa de WinUI para Windows 10 20H2+ y
Windows 11. Se instala sin privilegios de administrador e incluye instaladores x64
y ARM64 firmados en su propia página de versiones.

Windows Hub se publica de manera independiente de la CLI y el Gateway de OpenClaw. Descargue
el instalador estable más reciente de Hub desde la
[página de versiones de Windows Hub](https://github.com/openclaw/openclaw-windows-node/releases/latest)
o directamente mediante `releases/latest/download`:

- [OpenClawCompanion-Setup-x64.exe](https://github.com/openclaw/openclaw-windows-node/releases/latest/download/OpenClawCompanion-Setup-x64.exe)
- [OpenClawCompanion-Setup-arm64.exe](https://github.com/openclaw/openclaw-windows-node/releases/latest/download/OpenClawCompanion-Setup-arm64.exe)

Si uno de los enlaces anteriores devuelve un error 404, visite la [página de versiones de Windows Hub](https://github.com/openclaw/openclaw-windows-node/releases)
y abra la versión estable más reciente de Windows Hub. Las versiones estables normales de OpenClaw
también replican una compilación fijada de Windows Hub, validada para la versión; esa réplica puede estar
por detrás de una versión independiente más reciente de Hub.

Después de la instalación, inicie **OpenClaw Companion** desde el menú Inicio o la
bandeja del sistema. El instalador también añade accesos directos para Gateway Setup, Chat, Settings,
Check for Updates y la desinstalación.

### Qué incluye Windows Hub

- Estado en la bandeja del sistema e inicio al iniciar sesión.
- Configuración inicial de un Gateway WSL local administrado por la aplicación.
- Configuración de conexiones para Gateways locales, remotos y con túneles SSH.
- Ventana de chat nativa y acceso a la interfaz de control en el navegador.
- Diagnósticos de Command Center para sesiones, uso, canales, Nodes, emparejamiento
  y comandos de reparación.
- Modo Node de Windows para que el agente controle el lienzo, la pantalla, la cámara,
  las notificaciones, el estado del dispositivo, la voz y `system.run` de forma controlada.
- Modo de servidor MCP local para clientes MCP como Claude Desktop, Claude Code
  y Cursor.

### Primer inicio

En el primer inicio, Windows Hub abre la configuración cuando no hay ningún
Gateway guardado que se pueda utilizar. La vía más rápida es **Set up locally**, que aprovisiona una
distribución WSL `OpenClawGateway` administrada por la aplicación, instala el Gateway en ella y
empareja la aplicación. Esto no exporta ni modifica la distribución Ubuntu existente.

Elija **Advanced setup** o abra la pestaña Connections si ya dispone de un
Gateway. Puede conectarse a:

- un Gateway local en este PC
- un Gateway WSL en este PC
- un Gateway remoto mediante URL y token o código de configuración
- un Gateway al que se accede mediante un túnel SSH

Cuando finaliza la configuración, el icono de la bandeja se vuelve verde. Abra **Command Center** desde
la bandeja para confirmar la conexión, el emparejamiento, el estado del Node y el estado de los canales.

## Modo Node de Windows

Windows Hub puede registrarse como un Node de OpenClaw para que el agente pueda usar las
capacidades nativas de Windows declaradas a través del Gateway. Los comandos del Node deben estar
declarados por el Node y permitidos por la política del Gateway antes de ejecutarse; consulte
[Nodes](/es/nodes#command-policy) para conocer el modelo completo de permisos y denegaciones.

Comandos habituales:

| Familia | Comandos                                                                             |
| ------ | ------------------------------------------------------------------------------------ |
| Lienzo | `canvas.present`, `canvas.hide`, `canvas.navigate`, `canvas.eval`, `canvas.snapshot` |
| Pantalla | `screen.snapshot`; `screen.record` requiere consentimiento explícito                          |
| Cámara | `camera.list`; `camera.snap`, `camera.clip` requieren consentimiento explícito                  |
| Sistema | `system.notify`, `system.run`, `system.run.prepare`, `system.which`                  |
| Dispositivo | `location.get`, `device.info`, `device.status`                                       |
| Voz   | `talk.ptt.start`, `talk.ptt.stop`, `talk.ptt.cancel`, `talk.ptt.once`, `talk.speak`  |

El modo Node requiere el emparejamiento con el Gateway. Si la aplicación muestra una solicitud de emparejamiento,
apruébela desde el host del Gateway:

```powershell
openclaw devices list
openclaw devices approve <requestId>
openclaw nodes status
```

El Gateway solo reenvía los comandos que declara el Node y que permite la política
del servidor. Los comandos que afectan a la privacidad, como `screen.record`, `camera.snap`
y `camera.clip`, necesitan un consentimiento explícito mediante `gateway.nodes.commands.allow`.

## Modo MCP local

Windows Hub puede exponer el mismo registro de capacidades nativas de Windows como servidor
MCP local en la interfaz de bucle invertido, para que los clientes MCP locales puedan controlar las capacidades de Windows
sin que se esté ejecutando un Gateway de OpenClaw.

Actívelo en Settings de Windows Hub, en la sección para desarrolladores o avanzada. La
aplicación muestra el punto de conexión de bucle invertido y el token de portador cuando se activa el servidor.

Matriz de modos:

| Modo Node | Servidor MCP | Comportamiento                           |
| --------- | ---------- | ---------------------------------- |
| desactivado       | desactivado        | Aplicación de escritorio solo para el operador          |
| activado        | desactivado        | Node de Windows conectado al Gateway     |
| desactivado       | activado         | Solo servidor MCP local              |
| activado        | activado         | Node del Gateway y servidor MCP local |

## CLI y Gateway nativos de Windows

Para un uso centrado en el terminal, instale OpenClaw desde PowerShell:

```powershell
iwr -useb https://openclaw.ai/install.ps1 | iex
```

Verifique:

```powershell
openclaw --version
openclaw doctor
openclaw gateway status --json
```

El inicio administrado usa las tareas programadas de Windows cuando están disponibles. La tarea conserva
el script legible `gateway.cmd` en el directorio de estado de OpenClaw, pero lo inicia
mediante un contenedor WScript `gateway.vbs` generado, para que el Gateway en segundo plano
no abra una ventana de consola visible. Si se deniega la creación de la tarea, OpenClaw
recurre a un elemento de inicio de sesión por usuario en la carpeta Inicio.

Instale el servicio del Gateway:

```powershell
openclaw gateway install
openclaw gateway status --json
```

Para usar solo la CLI sin un servicio administrado del Gateway:

```powershell
openclaw onboard --non-interactive --skip-health
openclaw gateway run
```

## Gateway WSL2

WSL2 sigue siendo el entorno de ejecución del Gateway más compatible con Linux en Windows. Windows
Hub puede configurar automáticamente un Gateway WSL administrado por la aplicación, o se puede instalar manualmente
en una distribución propia.

Configuración manual:

```powershell
wsl --install
# O elija explícitamente una distribución:
wsl --list --online
wsl --install -d Ubuntu-24.04
```

Active systemd dentro de WSL:

```bash
sudo tee /etc/wsl.conf >/dev/null <<'EOF'
[boot]
systemd=true
EOF
```

Reinicie WSL desde PowerShell:

```powershell
wsl --shutdown
```

Después, instale OpenClaw dentro de WSL mediante la guía de inicio rápido de Linux:

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
openclaw gateway status
```

## Inicio automático del Gateway antes de iniciar sesión en Windows

Para configuraciones WSL sin interfaz gráfica, asegúrese de que se ejecute toda la cadena de arranque, incluso cuando nadie
inicie sesión en Windows.

Dentro de WSL:

```bash
sudo apt-get install -y dbus-x11
sudo loginctl enable-linger "$(whoami)"
openclaw gateway install
```

En PowerShell como administrador:

```powershell
schtasks /create /tn "WSL Boot" /tr "wsl.exe -d Ubuntu --exec dbus-launch true" /sc onstart /ru "$env:USERNAME"
```

Sustituya `Ubuntu` por el nombre de la distribución obtenido mediante:

```powershell
wsl --list --verbose
```

<Note>
Dos cambios respecto a las instrucciones anteriores:

- **`dbus-launch true` en lugar de `/bin/true`**: en WSL >= 2.6.1.0, una
  regresión ([microsoft/WSL #13416](https://github.com/microsoft/WSL/issues/13416))
  cierra la distribución por inactividad entre 15 y 20 segundos después de que el último cliente salga, incluso
  con la permanencia activada. `dbus-launch true` mantiene activo un proceso secundario de init
  como solución provisional (debate de la comunidad, [microsoft/WSL #9245](https://github.com/microsoft/WSL/discussions/9245)).
- **`/ru "$env:USERNAME"` en lugar de `/ru SYSTEM`**: las distribuciones WSL por usuario (la
  configuración predeterminada) no son visibles para la cuenta SYSTEM, por lo que la tarea parece
  ejecutarse, pero la distribución nunca se inicia. Ejecutarla con su propia cuenta evita
  este problema; Windows solicita la contraseña al crear la tarea.

</Note>

Después de reiniciar, verifique desde WSL:

```bash
systemctl --user is-enabled openclaw-gateway.service
systemctl --user status openclaw-gateway.service --no-pager
```

## Exponer servicios WSL en la LAN

WSL tiene su propia red virtual. Si otro equipo debe acceder a un servicio
dentro de WSL, reenvíe un puerto de Windows a la dirección IP actual de WSL. La dirección IP de WSL puede
cambiar después de reiniciar, así que actualice la regla de reenvío cuando sea necesario.

Ejemplo en PowerShell como administrador:

```powershell
$Distro = "Ubuntu-24.04"
$ListenPort = 2222
$TargetPort = 22

$WslIp = (wsl -d $Distro -- hostname -I).Trim().Split(" ")[0]
if (-not $WslIp) { throw "No se encontró la dirección IP de WSL." }

netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=$ListenPort `
  connectaddress=$WslIp connectport=$TargetPort

New-NetFirewallRule -DisplayName "WSL SSH $ListenPort" -Direction Inbound `
  -Protocol TCP -LocalPort $ListenPort -Action Allow
```

Notas:

- Las conexiones SSH desde otro equipo apuntan a la dirección IP del host de Windows; por ejemplo, `ssh user@windows-host -p 2222`.
- Los Nodes remotos deben apuntar a una URL de Gateway accesible, no a `127.0.0.1`.
- Use `listenaddress=0.0.0.0` para el acceso mediante LAN y `127.0.0.1` solo para el acceso local.

## Solución de problemas

### El icono de la bandeja no aparece

Busque `OpenClaw.Tray.WinUI.exe` en el Administrador de tareas. Si está en ejecución, abra el
área de iconos ocultos de la bandeja y fíjelo. Si no lo está, inicie **OpenClaw Companion** desde
el menú Inicio.

### La configuración local falla

Abra el registro de configuración desde Windows Hub o examine:

```powershell
notepad "$env:LOCALAPPDATA\OpenClawTray\Logs\Setup\easy-setup-latest.txt"
```

Causas habituales: WSL desactivado, virtualización bloqueada, estado obsoleto de WSL
administrado por la aplicación o un fallo de red durante la instalación del paquete del Gateway.

### La aplicación indica que se requiere emparejamiento

Apruebe la solicitud del operador o del Node desde el Gateway:

```powershell
openclaw devices list
openclaw devices approve <requestId>
```

Si el dispositivo ya tenía un token, vuelva a conectarse desde la pestaña Connections después
de la aprobación.

### El chat web no puede acceder a un Gateway remoto

El chat web remoto necesita HTTPS o localhost. En el caso de certificados autofirmados, confíe
en el certificado en Windows o use un túnel SSH hacia una URL de localhost.

### Los comandos de `screen.snapshot`, cámara o audio fallan

Confirme los permisos de Windows para la cámara, el micrófono, la captura de pantalla y
las notificaciones. Las instalaciones empaquetadas declaran las capacidades protegidas, pero
Windows puede seguir solicitando permiso la primera vez que un comando las utiliza.

### Falla la conectividad con Git o GitHub

Algunas redes bloquean o limitan HTTPS hacia GitHub. Si `git clone` o
`gh auth login` falla, pruebe otra red, una VPN o un proxy HTTP/HTTPS.

Para la autenticación basada en token de `gh` en la sesión actual:

```powershell
$env:GH_TOKEN="<your-token>"
gh auth status
gh auth setup-git
```

Nunca confirme tokens en el repositorio ni los pegue en incidencias o pull requests.

## Contenido relacionado

- [Descripción general de la instalación](/es/install)
- [Configuración de Node.js](/es/install/node)
- [Nodes](/es/nodes)
- [Interfaz de control](/es/web/control-ui)
- [Configuración del Gateway](/es/gateway/configuration)
