---
read_when:
    - Empaquetado de OpenClaw.app
    - Depuración del servicio launchd del Gateway en macOS
    - Instalación de la CLI del Gateway para macOS
summary: Entorno de ejecución del Gateway en macOS (servicio externo de launchd)
title: Gateway en macOS
x-i18n:
    generated_at: "2026-07-26T05:11:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 30c1ae14d8f8eaab73d0e2b725292d7411c2c8b5e0e0c32ad13989c01340d054
    source_path: platforms/mac/bundled-gateway.md
    workflow: 16
---

OpenClaw.app no incluye Node ni el entorno de ejecución del Gateway. La aplicación para macOS
requiere una instalación **externa** de la CLI `openclaw`, no inicia el Gateway como
proceso secundario y administra un servicio launchd por usuario para mantener el Gateway
en ejecución (o se conecta a un Gateway local que ya esté en ejecución).

## Configuración automática

En un Mac nuevo, seleccione **Este Mac** durante la incorporación. La aplicación ejecuta su
script de instalación firmado e incluido antes del asistente del Gateway: instala un
entorno de ejecución de Node en el espacio del usuario y la CLI `openclaw` correspondiente en `~/.openclaw`,
y después instala e inicia el servicio launchd por usuario. Esta vía no requiere
Terminal, Homebrew ni acceso de administrador.

La aplicación solo incluye el script de instalación, no la carga útil de Node ni del Gateway;
la configuración requiere conexión a Internet para descargar el entorno de ejecución y el paquete
de OpenClaw correspondiente.

## Recuperación manual

Se recomienda Node 24.15+ para una instalación manual; Node 22.22.3+ también funciona. Instale
`openclaw` globalmente:

```bash
npm install -g openclaw@<version>
```

Use **Reintentar configuración** después de que falle la configuración automática. Si sigue fallando,
instale manualmente la CLI con el comando anterior y seleccione **Comprobar de nuevo**
durante la incorporación.

## Launchd (Gateway como LaunchAgent)

Etiqueta: `ai.openclaw.gateway` (perfil predeterminado) o `ai.openclaw.<profile>`
para un perfil con nombre.

Ubicación del plist (por usuario): `~/Library/LaunchAgents/ai.openclaw.gateway.plist`
(o `ai.openclaw.<profile>.plist`).

La aplicación para macOS administra la instalación y actualización del LaunchAgent para el perfil predeterminado en
modo local. La CLI también puede instalarlo directamente: `openclaw gateway install`
(los perfiles con nombre se seleccionan mediante la variable de entorno `OPENCLAW_PROFILE`).

Comportamiento:

- «OpenClaw activo» activa o desactiva el LaunchAgent.
- Salir de la aplicación **no** detiene el Gateway (launchd lo mantiene activo).
- Si ya hay un Gateway en ejecución en el puerto configurado, la aplicación se conecta a
  él en lugar de iniciar uno nuevo.

Registro:

- Salida estándar de launchd: `~/Library/Logs/openclaw/gateway.log` (los perfiles usan
  `gateway-<profile>.log`)
- Salida de error estándar de launchd: suprimida
- Si el host entra en un bucle con mensajes `EADDRINUSE` repetidos o reinicios rápidos, compruebe si hay
  LaunchAgents `ai.openclaw.gateway` / `ai.openclaw.node` duplicados y la
  solución alternativa del marcador de launchd en
  [Solución de problemas del Gateway](/es/gateway/troubleshooting#macos-launchd-supervisor-loop-with-duplicate-gatewaynode-launchagents).

## Compatibilidad de versiones

La aplicación para macOS compara la versión del Gateway con su propia versión. La incorporación
ejecuta automáticamente la configuración administrada cuando falta una CLI existente o esta es
incompatible. Use **Reintentar configuración** para repetir la instalación o **Comprobar de nuevo**
después de reparar una CLI externa.

## Directorio de estado en macOS

Mantenga el estado de OpenClaw en un disco local no sincronizado. Evite iCloud Drive y otras
carpetas sincronizadas con la nube; la latencia de sincronización y los bloqueos de archivos pueden afectar a las sesiones,
las credenciales y el estado del Gateway.

Establezca `OPENCLAW_STATE_DIR` en una ruta local solo cuando necesite reemplazar el valor predeterminado.
`openclaw doctor` advierte sobre rutas de estado habituales sincronizadas con la nube y recomienda
volver a mover el estado al almacenamiento local. Consulte
[variables de entorno](/es/help/environment#path-related-env-vars) y
[Doctor](/es/gateway/doctor).

## Depuración de la conectividad de la aplicación

Use la CLI de depuración de macOS desde un repositorio de código fuente para probar el mismo
protocolo de enlace WebSocket y la misma lógica de descubrimiento del Gateway que utiliza la aplicación:

```bash
cd apps/macos
swift run openclaw-mac connect --json
swift run openclaw-mac discover --timeout 3000 --json
```

`connect` acepta `--url`, `--token`, `--timeout`, `--probe` y `--json`
(además de reemplazos de la identidad del cliente; ejecútelo con `--help` para ver la lista completa).
`discover` acepta `--timeout`, `--json` y `--include-local`. Compare
la salida del descubrimiento con `openclaw gateway discover --json` cuando necesite
distinguir los problemas de descubrimiento de la CLI de los problemas de conexión de la aplicación.

## Comprobación rápida

```bash
openclaw --version

OPENCLAW_SKIP_CHANNELS=1 \
OPENCLAW_SKIP_CANVAS_HOST=1 \
openclaw gateway --port 18999 --bind loopback
```

Después:

```bash
openclaw gateway call health --url ws://127.0.0.1:18999 --timeout 3000
```

## Relacionado

- [Aplicación para macOS](/es/platforms/macos)
- [Guía operativa del Gateway](/es/gateway)
