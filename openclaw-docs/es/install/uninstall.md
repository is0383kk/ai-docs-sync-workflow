---
read_when:
    - Quieres eliminar OpenClaw de un equipo
    - El servicio del Gateway sigue ejecutándose después de la desinstalación
summary: Desinstalar OpenClaw por completo (CLI, servicio, estado, espacio de trabajo)
title: Desinstalar
x-i18n:
    generated_at: "2026-07-26T04:44:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 84f01dc11defe6f19c89232375e48bad383b2e71379f47f43e759d3d7bb908b5
    source_path: install/uninstall.md
    workflow: 16
---

Dos rutas:

- **Ruta sencilla** si `openclaw` todavía está instalado.
- **Eliminación manual del servicio** si la CLI ya no está disponible, pero el servicio sigue ejecutándose.

## Ruta sencilla (la CLI sigue instalada)

Recomendación: usar el desinstalador integrado:

```bash
openclaw uninstall
```

La eliminación del estado conserva los directorios de espacio de trabajo configurados, a menos que también se seleccione `--workspace`.

Vista previa de lo que se eliminará (seguro):

```bash
openclaw uninstall --dry-run --all
```

Sin interacción (automatización / npx). Usar con precaución y solo después de confirmar los ámbitos:

```bash
openclaw uninstall --all --yes --non-interactive
npx -y openclaw uninstall --all --yes --non-interactive
```

Opciones: `--service`, `--state`, `--workspace`, `--app` seleccionan ámbitos individuales; `--all` selecciona los cuatro.

Pasos manuales (mismo resultado):

1. Detener el servicio del Gateway:

```bash
openclaw gateway stop
```

2. Desinstalar el servicio del Gateway (launchd/systemd/schtasks):

```bash
openclaw gateway uninstall
```

3. Eliminar el estado y la configuración:

```bash
rm -rf "${OPENCLAW_STATE_DIR:-$HOME/.openclaw}"
```

Si se estableció `OPENCLAW_CONFIG_PATH` en una ubicación personalizada fuera del directorio de estado, eliminar también ese archivo.
Si se desea conservar un espacio de trabajo dentro del directorio de estado, como `~/.openclaw/workspace`, moverlo a otra ubicación antes de ejecutar `rm -rf` o eliminar selectivamente el contenido del estado.

4. Eliminar el espacio de trabajo (opcional; elimina los archivos del agente):

```bash
rm -rf ~/.openclaw/workspace
```

5. Eliminar la instalación de la CLI (elegir el método utilizado):

```bash
npm rm -g openclaw
pnpm remove -g openclaw
bun remove -g openclaw
```

6. Si se instaló la aplicación para macOS:

```bash
rm -rf /Applications/OpenClaw.app
```

Notas:

- Si se usaron perfiles (`--profile` / `OPENCLAW_PROFILE`), repetir el paso 3 para cada directorio de estado (los valores predeterminados son `~/.openclaw-<profile>`).
- En modo remoto, el directorio de estado se encuentra en el **host del Gateway**, por lo que también deben ejecutarse allí los pasos 1-4.

## Eliminación manual del servicio (CLI no instalada)

Usar este método si el servicio del Gateway sigue ejecutándose, pero falta `openclaw`.

### macOS (launchd)

La etiqueta predeterminada es `ai.openclaw.gateway` (o `ai.openclaw.<profile>` con un perfil):

```bash
launchctl bootout gui/$UID/ai.openclaw.gateway
rm -f ~/Library/LaunchAgents/ai.openclaw.gateway.plist
```

Si se usó un perfil, sustituir la etiqueta y el nombre del archivo plist por `ai.openclaw.<profile>`.

### Linux (unidad de usuario de systemd)

El nombre predeterminado de la unidad es `openclaw-gateway.service` (o `openclaw-gateway-<profile>.service`). Es posible que todavía exista una unidad `clawdbot-gateway.service` anterior al cambio de nombre en equipos actualizados desde instalaciones muy antiguas; `openclaw uninstall` / `openclaw gateway uninstall` la detecta y elimina automáticamente.

```bash
systemctl --user disable --now openclaw-gateway.service
rm -f ~/.config/systemd/user/openclaw-gateway.service
systemctl --user daemon-reload
```

### Windows (tarea programada)

El nombre predeterminado de la tarea es `OpenClaw Gateway` (o `OpenClaw Gateway (<profile>)`).
La tarea inicia un script `gateway.vbs` sin ventana dentro del directorio de estado, que a su vez
ejecuta `gateway.cmd`; eliminar ambos.

```powershell
schtasks /Delete /F /TN "OpenClaw Gateway"
Remove-Item -Force "$env:USERPROFILE\.openclaw\gateway.cmd" -ErrorAction SilentlyContinue
Remove-Item -Force "$env:USERPROFILE\.openclaw\gateway.vbs" -ErrorAction SilentlyContinue
```

Si se usó un perfil, eliminar el nombre de tarea correspondiente y los archivos `gateway.cmd` /
`gateway.vbs` de `~\.openclaw-<profile>`.

## Instalación normal frente a repositorio de código fuente

### Instalación normal (install.sh / npm / pnpm / bun)

Si se usó `https://openclaw.ai/install.sh` o `install.ps1`, la CLI se instaló con `npm install -g openclaw@latest`.
Eliminarla con `npm rm -g openclaw` (o `pnpm remove -g` / `bun remove -g` si se instaló de ese modo).

### Repositorio de código fuente (git clone)

Si se ejecuta desde una copia local del repositorio (`git clone` + `openclaw ...` / `bun run openclaw ...`):

1. Desinstalar el servicio del Gateway **antes** de eliminar el repositorio (usar la ruta sencilla anterior o la eliminación manual del servicio).
2. Eliminar el directorio del repositorio.
3. Eliminar el estado y el espacio de trabajo como se muestra anteriormente.

## Contenido relacionado

- [Descripción general de la instalación](/es/install)
- [Guía de migración](/es/install/migrating)
