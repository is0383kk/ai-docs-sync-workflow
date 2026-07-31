---
read_when: Browser control fails on Linux, especially with snap Chromium
summary: Solucionar los problemas de inicio de CDP de Chrome/Brave/Edge/Chromium para el control del navegador de OpenClaw en Linux
title: Solución de problemas del navegador
x-i18n:
    generated_at: "2026-07-26T05:57:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8e5db2da2d43129862f0c005213df828f6eae81f5561e57d41795ea90787822a
    source_path: tools/browser-linux-troubleshooting.md
    workflow: 16
---

## Problema: no se pudo iniciar Chrome CDP en el puerto 18800

```json
{ "error": "Error: No se pudo iniciar Chrome CDP en el puerto 18800 para el perfil \"openclaw\"." }
```

### Causa raíz

En Ubuntu y la mayoría de las distribuciones de Linux, `apt install chromium` instala un envoltorio de snap,
no un navegador real:

```text
Nota: se selecciona 'chromium-browser' en lugar de 'chromium'
chromium-browser ya es la versión más reciente (2:1snap1-0ubuntu2).
```

El confinamiento de AppArmor de snap interfiere en la forma en que OpenClaw inicia y supervisa
el proceso del navegador.

Otros fallos habituales de inicio en Linux:

- `The profile appears to be in use by another Chromium process`: archivos de bloqueo
  `Singleton*` obsoletos en el directorio del perfil administrado. OpenClaw elimina
  estos bloqueos y vuelve a intentarlo una vez cuando el bloqueo apunta a un proceso inactivo o
  de otro host.
- `Missing X server or $DISPLAY`: se solicitó explícitamente un navegador visible
  en un host sin sesión de escritorio. Los perfiles locales administrados usan de forma alternativa
  el modo sin interfaz gráfica en Linux cuando `DISPLAY` y `WAYLAND_DISPLAY` no están definidos.
  Si se ha definido `OPENCLAW_BROWSER_HEADLESS=0`, `browser.headless: false` o
  `browser.profiles.<name>.headless: false`, elimine esa configuración que fuerza el modo con interfaz gráfica, defina
  `OPENCLAW_BROWSER_HEADLESS=1`, inicie `Xvfb`, ejecute
  `openclaw browser start --headless` para realizar un inicio administrado único, o ejecute
  OpenClaw en una sesión de escritorio real.

### Solución 1: instalar Google Chrome (recomendado)

```bash
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
sudo dpkg -i google-chrome-stable_current_amd64.deb
sudo apt --fix-broken install -y  # si hay errores de dependencias
```

Actualice `~/.openclaw/openclaw.json`:

```json
{
  "browser": {
    "enabled": true,
    "executablePath": "/usr/bin/google-chrome-stable",
    "headless": true,
    "noSandbox": true
  }
}
```

### Solución 2: usar Chromium de snap en modo de solo conexión

Si debe conservar Chromium de snap, configure OpenClaw para que se conecte a un
navegador iniciado manualmente en lugar de iniciarlo:

```json
{
  "browser": {
    "enabled": true,
    "attachOnly": true,
    "headless": true,
    "noSandbox": true
  }
}
```

Inicie Chromium manualmente:

```bash
chromium-browser --headless --no-sandbox --disable-gpu \
  --remote-debugging-port=18800 \
  --user-data-dir=$HOME/.openclaw/browser/openclaw/user-data \
  about:blank &
```

Opcionalmente, configúrelo para que se inicie automáticamente con un servicio de usuario de systemd:

```ini
# ~/.config/systemd/user/openclaw-browser.service
[Unit]
Description=Navegador de OpenClaw (Chrome CDP)
After=network.target

[Service]
ExecStart=/snap/bin/chromium --headless --no-sandbox --disable-gpu --remote-debugging-port=18800 --user-data-dir=%h/.openclaw/browser/openclaw/user-data about:blank
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

```bash
systemctl --user enable --now openclaw-browser.service
```

### Verificar que el navegador funciona

```bash
curl -s http://127.0.0.1:18791/ | jq '{running, pid, chosenBrowser}'
curl -s -X POST http://127.0.0.1:18791/start
curl -s http://127.0.0.1:18791/tabs
```

### Referencia de configuración

| Opción                      | Descripción                                                          | Valor predeterminado                                                            |
| --------------------------- | -------------------------------------------------------------------- | ------------------------------------------------------------------ |
| `browser.enabled`           | Habilitar el control del navegador                                               | `true`                                                             |
| `browser.executablePath`    | Ruta al ejecutable de un navegador basado en Chromium (Chrome/Brave/Edge/Chromium) | detección automática (se prefiere el navegador predeterminado del SO si está basado en Chromium) |
| `browser.headless`          | Ejecutar sin interfaz gráfica                                                      | `false`                                                            |
| `OPENCLAW_BROWSER_HEADLESS` | Configuración por proceso que prevalece sobre el modo sin interfaz gráfica del navegador local administrado         | sin definir                                                              |
| `browser.noSandbox`         | Añadir el indicador `--no-sandbox` (necesario para algunas configuraciones de Linux)               | `false`                                                            |
| `browser.attachOnly`        | No iniciar un navegador; conectarse únicamente a uno existente              | `false`                                                            |

En Raspberry Pi, hosts VPS antiguos o almacenamiento lento, use un navegador iniciado
manualmente con `attachOnly` cuando Chrome necesite más tiempo para exponer su endpoint HTTP de CDP
o estar listo del que permite el plazo límite del navegador administrado.

### Problema: no se encontraron pestañas de Chrome para profile="user"

Se está usando el perfil `user` (`existing-session` / Chrome MCP) y no hay
pestañas abiertas a las que conectarse.

Opciones para solucionarlo:

1. Use en su lugar el navegador administrado:
   `openclaw browser --browser-profile openclaw start` (o defina
   `browser.defaultProfile: "openclaw"`).
2. Mantenga Chrome local en ejecución con al menos una pestaña abierta y vuelva a intentarlo con
   `--browser-profile user`.

Notas:

- `user` solo funciona en el host. En servidores Linux, contenedores o hosts remotos, se recomienda usar
  perfiles CDP.
- `user` y otros perfiles `existing-session` comparten las limitaciones actuales de Chrome MCP:
  solo acciones basadas en referencias, un archivo por carga, sin sustituciones de `timeoutMs`
  en cuadros de diálogo, sin `wait --load networkidle` y sin `responsebody`, exportación a PDF,
  interceptación de descargas ni acciones por lotes.
- Los perfiles locales del controlador `openclaw` asignan automáticamente `cdpPort`/`cdpUrl`; establézcalos
  manualmente solo para CDP remoto.
- Los perfiles CDP remotos aceptan `http://`, `https://`, `ws://` y `wss://`.
  Use HTTP(S) para la detección de `/json/version`, o WS(S) cuando el servicio del navegador
  proporcione una URL directa del socket de DevTools.

## Contenido relacionado

- [Navegador](/es/tools/browser)
- [Inicio de sesión en el navegador](/es/tools/browser-login)
- [Solución de problemas del navegador con CDP remoto de WSL2 para Windows](/es/tools/browser-wsl2-windows-remote-cdp-troubleshooting)
