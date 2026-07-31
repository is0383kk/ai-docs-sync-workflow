---
read_when:
    - Configuración inicial desde cero
    - Quieres la forma más rápida de poner en marcha un chat
summary: Instala OpenClaw e inicia tu primer chat en cuestión de minutos.
title: Primeros pasos
x-i18n:
    generated_at: "2026-07-26T04:53:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8f50073b059477636b94e128cec90b41dcc21c8bb132e34900e68409cacf70eb
    source_path: start/getting-started.md
    workflow: 16
---

Instala OpenClaw, ejecuta la incorporación y conversa con tu asistente de IA en unos 5
minutos. Al terminar, tendrás un Gateway en ejecución, la autenticación configurada y una
sesión de chat operativa.

## Qué necesitas

- **Node.js 22.22.3+, 24.15+ o 25.9+** (24 es la opción predeterminada recomendada)
- **Una clave de API** de un proveedor de modelos (Anthropic, OpenAI, Google, etc.); la incorporación te la solicitará

<Tip>
Comprueba tu versión de Node con `node --version`.
**Usuarios de Windows:** la aplicación Hub nativa para Windows es la opción de escritorio más sencilla. También se admiten
el instalador de PowerShell y las opciones de Gateway mediante WSL2. Consulta [Windows](/es/platforms/windows).
¿Necesitas instalar Node? Consulta [Configuración de Node](/es/install/node).
</Tip>

## Configuración rápida

<Steps>
  <Step title="Instalar OpenClaw">
    <Tabs>
      <Tab title="macOS / Linux">
        ```bash
        curl -fsSL https://openclaw.ai/install.sh | bash
        ```
        <img
  src="/assets/install-script.svg"
  alt="Proceso del script de instalación"
  className="rounded-lg"
/>
      </Tab>
      <Tab title="Windows (PowerShell)">
        ```powershell
        iwr -useb https://openclaw.ai/install.ps1 | iex
        ```
      </Tab>
    </Tabs>

    <Note>
    Otros métodos de instalación (Docker, Nix, npm): [Instalación](/es/install).
    </Note>

  </Step>
  <Step title="Ejecutar la incorporación">
    ```bash
    openclaw onboard --install-daemon
    ```

    El asistente te guía para elegir un proveedor de modelos, establecer una clave de API
    y configurar el Gateway. QuickStart suele tardar solo unos minutos, pero
    el inicio de sesión en el proveedor, la vinculación de canales, la instalación del daemon, las descargas de red, las Skills
    o los plugins opcionales pueden prolongar la incorporación completa. Omite los
    pasos opcionales y vuelve más tarde con `openclaw configure`.

    Consulta [Incorporación (CLI)](/es/start/wizard) para ver la referencia completa.

  </Step>
  <Step title="Verificar que el Gateway esté en ejecución">
    ```bash
    openclaw gateway status
    ```

    Deberías ver que el Gateway está escuchando en el puerto 18789.

  </Step>
  <Step title="Abrir el panel">
    ```bash
    openclaw dashboard
    ```

    Esto abre la interfaz de control en tu navegador. Si se carga, todo funciona correctamente.

  </Step>
  <Step title="Enviar tu primer mensaje">
    Escribe un mensaje en el chat de la interfaz de control y deberías recibir una respuesta de la IA.

    ¿Prefieres conversar desde tu teléfono? El canal más rápido de configurar es
    [Telegram](/es/channels/telegram) (solo se necesita un token de bot). Consulta [Canales](/es/channels)
    para ver todas las opciones.

  </Step>
</Steps>

<Accordion title="Avanzado: montar una compilación personalizada de la interfaz de control">
  Si mantienes una compilación localizada o personalizada del panel, configura
  `gateway.controlUi.root` para que apunte a un directorio que contenga tus recursos estáticos
  compilados y `index.html`.

```bash
mkdir -p "$HOME/.openclaw/control-ui-custom"
# Copia tus archivos estáticos compilados en ese directorio.
```

A continuación, establece:

```json
{
  "gateway": {
    "controlUi": {
      "enabled": true,
      "root": "$HOME/.openclaw/control-ui-custom"
    }
  }
}
```

Reinicia el Gateway y vuelve a abrir el panel:

```bash
openclaw gateway restart
openclaw dashboard
```

</Accordion>

## Qué hacer a continuación

<Columns>
  <Card title="Conectar un canal" href="/es/channels" icon="message-square">
    Discord, Feishu, iMessage, Matrix, Microsoft Teams, Signal, Slack, Telegram, WhatsApp, Zalo y más.
  </Card>
  <Card title="Vinculación y seguridad" href="/es/channels/pairing" icon="shield">
    Controla quién puede enviar mensajes a tu agente.
  </Card>
  <Card title="Configurar el Gateway" href="/es/gateway/configuration" icon="settings">
    Modelos, herramientas, entorno aislado y configuración avanzada.
  </Card>
  <Card title="Explorar herramientas" href="/es/tools" icon="wrench">
    Navegador, ejecución, búsqueda web, Skills y plugins.
  </Card>
</Columns>

<Accordion title="Avanzado: variables de entorno">
  Si ejecutas OpenClaw como una cuenta de servicio o deseas usar rutas personalizadas:

- `OPENCLAW_HOME` — directorio de inicio para la resolución de rutas internas
- `OPENCLAW_STATE_DIR` — sustituye el directorio de estado
- `OPENCLAW_CONFIG_PATH` — sustituye la ruta del archivo de configuración

Referencia completa: [Variables de entorno](/es/help/environment).
</Accordion>

## Contenido relacionado

- [Descripción general de la instalación](/es/install)
- [Descripción general de los canales](/es/channels)
- [Configuración](/es/start/setup)
