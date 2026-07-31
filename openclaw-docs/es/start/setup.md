---
read_when:
    - Configuración de una máquina nueva
    - Quieres «lo último y lo mejor» sin estropear tu configuración personal
summary: Flujos de trabajo avanzados de configuración y desarrollo para OpenClaw
title: Configuración
x-i18n:
    generated_at: "2026-07-26T04:52:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c40d6d2bf2814465f3cc49c65d4c1498671420af728ce8012d13af3fba67025a
    source_path: start/setup.md
    workflow: 16
---

<Note>
Si realiza la configuración por primera vez, comience por [Primeros pasos](/es/start/getting-started).
Para obtener detalles sobre la incorporación, consulte [Incorporación (CLI)](/es/start/wizard).
</Note>

## Resumen

Elija un flujo de configuración según la frecuencia con la que desee recibir actualizaciones y si quiere ejecutar el Gateway personalmente:

- **La personalización se mantiene fuera del repositorio:** conserve la configuración y el espacio de trabajo en `~/.openclaw/openclaw.json` y `~/.openclaw/workspace/` para que las actualizaciones del repositorio no los modifiquen.
- **Flujo estable (recomendado para la mayoría):** instale la aplicación para macOS y deje que ejecute el Gateway incluido.
- **Flujo de última generación (desarrollo):** ejecute el Gateway personalmente mediante `pnpm gateway:watch` y, después, permita que la aplicación para macOS se conecte en modo Local.

## Requisitos previos (desde el código fuente)

- Se recomienda Node 24.15+ (Node 22 LTS, actualmente `22.22.3+`, aún es compatible)
- Se requiere `pnpm` para trabajar con copias del código fuente. OpenClaw carga los plugins incluidos desde los
  paquetes `extensions/*` del espacio de trabajo de pnpm en modo de desarrollo, por lo que `npm install` en la raíz
  no prepara todo el árbol del código fuente.
- Docker (opcional; solo para configuración en contenedores y pruebas e2e; consulte [Docker](/es/install/docker))

## Estrategia de personalización (para que las actualizaciones no causen problemas)

Si desea una configuración «100 % adaptada» _y_ actualizaciones sencillas, conserve la personalización en:

- **Configuración:** `~/.openclaw/openclaw.json` (similar a JSON/JSON5)
- **Espacio de trabajo:** `~/.openclaw/workspace` (Skills, instrucciones, memorias; conviértalo en un repositorio Git privado)

Inicialice una sola vez las carpetas de configuración y del espacio de trabajo, sin ejecutar el asistente de incorporación completo:

```bash
openclaw setup --baseline
```

¿Aún no hay una instalación global? Ejecútelo desde este repositorio:

```bash
pnpm openclaw setup --baseline
```

(`openclaw setup` sin `--baseline` es un alias de `openclaw onboard` y ejecuta el asistente interactivo completo).

## Ejecutar el Gateway desde este repositorio

Después de `pnpm build`, puede ejecutar directamente la CLI empaquetada:

```bash
node openclaw.mjs gateway --port 18789 --verbose
```

## Flujo estable (primero la aplicación para macOS)

1. Instale e inicie **OpenClaw.app** (barra de menús).
2. Complete la lista de comprobación de incorporación y permisos (solicitudes de TCC).
3. Asegúrese de que el Gateway esté en modo **Local** y en ejecución (la aplicación lo gestiona).
4. Vincule las plataformas (por ejemplo, WhatsApp):

```bash
openclaw channels login
```

5. Comprobación rápida:

```bash
openclaw health
```

Si la incorporación no está disponible en la compilación:

- Ejecute `openclaw setup`, después `openclaw channels login` y, a continuación, inicie el Gateway manualmente (`openclaw gateway`).

## Flujo de última generación (Gateway en una terminal)

Objetivo: trabajar en el Gateway de TypeScript, disponer de recarga en caliente y mantener conectada la interfaz de la aplicación para macOS.

### 0) (Opcional) Ejecutar también la aplicación para macOS desde el código fuente

Si también desea usar la versión de última generación de la aplicación para macOS:

```bash
./scripts/restart-mac.sh
```

### 1) Iniciar el Gateway de desarrollo

```bash
pnpm install
# Solo la primera ejecución (o después de restablecer la configuración o el espacio de trabajo local de OpenClaw)
pnpm openclaw setup
pnpm gateway:watch
```

`gateway:watch` inicia o reinicia el proceso de supervisión del Gateway en una sesión de tmux
con nombre (`openclaw-gateway-watch-main`) y se conecta automáticamente desde terminales
interactivas. Los shells no interactivos permanecen desconectados y muestran
`tmux attach -t openclaw-gateway-watch-main`; use
`OPENCLAW_GATEWAY_WATCH_ATTACH=0 pnpm gateway:watch` para mantener desconectada una ejecución
interactiva, o `pnpm gateway:watch:raw` para el modo de supervisión en primer plano. El supervisor
detiene el servicio de Gateway instalado del perfil activo antes de tomar el control de su
puerto configurado o predeterminado, lo que evita que el supervisor de servicios sustituya
el proceso del código fuente. El servicio permanece instalado; ejecute `pnpm openclaw gateway start`
cuando termine la supervisión. El panel de tmux permanece disponible tras un fallo de inicio
para que otra terminal u otro agente puedan conectarse o capturar sus registros. El supervisor
recarga el proceso cuando se producen cambios relevantes en el código fuente, la configuración y los metadatos de los plugins incluidos. Si el
Gateway supervisado se cierra durante el inicio, `gateway:watch` ejecuta
`openclaw doctor --fix --non-interactive` una vez y vuelve a intentarlo; establezca
`OPENCLAW_GATEWAY_WATCH_AUTO_DOCTOR=0` para desactivar esa reparación exclusiva del entorno de desarrollo.
`pnpm gateway:watch` no recompila `dist/control-ui`, por lo que debe volver a ejecutar `pnpm ui:build` después de modificar `ui/` o usar `pnpm ui:dev` mientras desarrolla la interfaz de control.

### 2) Conectar la aplicación para macOS al Gateway en ejecución

En **OpenClaw.app**:

- Connection Mode: **Local**
  La aplicación se conectará al Gateway en ejecución en el puerto configurado.

### 3) Verificar

- El estado del Gateway en la aplicación debe mostrar **"Using existing gateway …"**
- O mediante la CLI:

```bash
openclaw health
```

### Errores comunes

- **Puerto incorrecto:** el WS del Gateway usa de forma predeterminada `ws://127.0.0.1:18789`; mantenga la aplicación y la CLI en el mismo puerto.
- **Ubicación del estado:**
  - Estado de los canales y proveedores: `~/.openclaw/credentials/`
  - Perfiles de autenticación de modelos: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
  - Sesiones y transcripciones: `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`
  - Artefactos de sesiones heredados o archivados: `~/.openclaw/agents/<agentId>/sessions/`
  - Registros: `/tmp/openclaw/`

## Mapa de almacenamiento de credenciales

Utilice esta información al depurar la autenticación o decidir qué debe incluirse en las copias de seguridad:

- **WhatsApp**: `~/.openclaw/credentials/whatsapp/<accountId>/creds.json`
- **Token del bot de Telegram**: configuración/entorno o `channels.telegram.tokenFile` (solo archivos normales; se rechazan los enlaces simbólicos)
- **Token del bot de Discord**: configuración/entorno o SecretRef (proveedores de entorno/archivo/ejecución)
- **Tokens de Slack**: configuración/entorno (`channels.slack.*`)
- **Listas de permitidos para el emparejamiento**:
  - `~/.openclaw/credentials/<channel>-allowFrom.json` (cuenta predeterminada)
  - `~/.openclaw/credentials/<channel>-<accountId>-allowFrom.json` (cuentas no predeterminadas)
- **Perfiles de autenticación de modelos**: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
- **Contenido de secretos respaldado por archivos (opcional)**: `~/.openclaw/secrets.json`
- **Importación de OAuth heredada**: `~/.openclaw/credentials/oauth.json`
  Más información: [Seguridad](/es/gateway/security#credential-storage-map).

## Actualización (sin estropear la configuración)

- Mantenga `~/.openclaw/workspace` y `~/.openclaw/` como «sus propios archivos»; no coloque instrucciones ni configuraciones personales en el repositorio `openclaw`.
- Actualización del código fuente: `git pull` + `pnpm install` + siga usando `pnpm gateway:watch`.

## Linux (servicio de usuario de systemd)

Las instalaciones en Linux utilizan un servicio de **usuario** de systemd. De forma predeterminada, systemd detiene los
servicios de usuario al cerrar sesión o durante la inactividad, lo que finaliza el Gateway. La incorporación intenta habilitar
la permanencia de la sesión (puede solicitar sudo). Si aún está desactivada, ejecute:

```bash
sudo loginctl enable-linger $USER
```

Para servidores siempre activos o multiusuario, considere usar un servicio de **sistema** en lugar de un
servicio de usuario (no es necesario mantener la sesión activa). Consulte la [guía operativa del Gateway](/es/gateway) para obtener información sobre systemd.

## Documentación relacionada

- [Guía operativa del Gateway](/es/gateway) (opciones, supervisión, puertos)
- [Configuración del Gateway](/es/gateway/configuration) (esquema de configuración y ejemplos)
- [Discord](/es/channels/discord) y [Telegram](/es/channels/telegram) (etiquetas de respuesta y ajustes de replyToMode)
- [Configuración del asistente de OpenClaw](/es/start/openclaw)
- [Aplicación para macOS](/es/platforms/macos) (ciclo de vida del Gateway)
