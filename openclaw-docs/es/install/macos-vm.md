---
read_when:
    - Quieres aislar OpenClaw de tu entorno principal de macOS
    - Se desea integrar iMessage en un entorno aislado
    - Quiere un entorno macOS restablecible que pueda clonar
    - Quieres comparar las opciones de máquinas virtuales macOS locales y alojadas
summary: Ejecuta OpenClaw en una máquina virtual macOS aislada (local o alojada) cuando necesites aislamiento o iMessage
title: Máquinas virtuales de macOS
x-i18n:
    generated_at: "2026-07-26T04:45:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7e6b963faaf40f65adce1081715bc295059b8bed278a8c71a05a86e04ad7a7a5
    source_path: install/macos-vm.md
    workflow: 16
---

## Opción predeterminada recomendada (la mayoría de los usuarios)

- **VPS Linux pequeño** para un Gateway siempre activo y de bajo coste. Consulte [Alojamiento en VPS](/es/vps).
- **Hardware dedicado** (Mac mini o equipo Linux) si se desea un control total y una **IP residencial** para la automatización del navegador. Muchos sitios bloquean las IP de centros de datos, por lo que la navegación local suele funcionar mejor.
- **Híbrido**: mantenga el Gateway en un VPS económico y conecte el Mac como **Node** cuando necesite automatizar el navegador o la interfaz de usuario. Consulte [Nodos](/es/nodes) y [Gateway remoto](/es/gateway/remote).

Utilice una máquina virtual macOS únicamente cuando necesite específicamente funciones exclusivas de macOS, como iMessage, o desee un aislamiento estricto respecto del Mac de uso diario.

## Opciones de máquinas virtuales macOS

### Máquina virtual local en un Mac con Apple Silicon (Lume)

Ejecute OpenClaw en una máquina virtual macOS aislada en su Mac con Apple Silicon mediante [Lume](https://cua.ai/docs/lume). Esto proporciona:

- Un entorno macOS completo y aislado (el sistema anfitrión se mantiene limpio)
- Compatibilidad con iMessage mediante `imsg`; la ruta local predeterminada no es posible en Linux/Windows
- Restablecimiento instantáneo mediante la clonación de máquinas virtuales
- Sin costes adicionales de hardware ni de servicios en la nube

### Proveedores de Mac alojados (nube)

Si se desea usar macOS en la nube, también sirven los proveedores de Mac alojados:

- [MacStadium](https://www.macstadium.com/) (Mac alojados)
- También sirven otros proveedores de Mac alojados; siga su documentación sobre máquinas virtuales y SSH

Cuando tenga acceso SSH a una máquina virtual macOS, continúe en [Instalar OpenClaw](#6-install-openclaw) a continuación.

## Ruta rápida (Lume, usuarios con experiencia)

1. Instale Lume.
2. `lume create openclaw --os macos --ipsw latest`
3. Complete el Asistente de Configuración y active Remote Login (SSH).
4. `lume run openclaw --no-display`
5. Acceda mediante SSH, instale OpenClaw y configure los canales.
6. Listo.

## Requisitos (Lume)

- Mac con Apple Silicon (M1/M2/M3/M4)
- macOS Sequoia o posterior en el sistema anfitrión
- ~60 GB de espacio libre en disco por máquina virtual
- ~20 minutos

## 1) Instalar Lume

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/trycua/cua/main/libs/lume/scripts/install.sh)"
```

Si `~/.local/bin` no está en PATH:

```bash
echo 'export PATH="$PATH:$HOME/.local/bin"' >> ~/.zshrc && source ~/.zshrc
```

Verifique:

```bash
lume --version
```

Documentación: [Instalación de Lume](https://cua.ai/docs/lume/guide/getting-started/installation)

## 2) Crear la máquina virtual macOS

```bash
lume create openclaw --os macos --ipsw latest
```

Esto descarga macOS y crea la máquina virtual. Se abre automáticamente una ventana VNC.

<Note>
La descarga puede tardar según la conexión.
</Note>

## 3) Completar el Asistente de Configuración

En la ventana VNC:

1. Seleccione el idioma y la región.
2. Omita Apple ID (o inicie sesión si desea usar iMessage más adelante).
3. Cree una cuenta de usuario (recuerde el nombre de usuario y la contraseña).
4. Omita todas las funciones opcionales.

Una vez completada la configuración:

1. Active SSH: System Settings -> General -> Sharing y active "Remote Login".
2. Para usar la máquina virtual sin interfaz gráfica, active el inicio de sesión automático: System Settings -> Users & Groups, seleccione "Automatically log in as:" y elija el usuario de la máquina virtual.

## 4) Obtener la dirección IP de la máquina virtual

```bash
lume get openclaw
```

Busque la dirección IP (normalmente `192.168.64.x`).

## 5) Acceder a la máquina virtual mediante SSH

```bash
ssh youruser@192.168.64.X
```

Sustituya `youruser` por la cuenta creada y la IP por la de la máquina virtual.

## 6) Instalar OpenClaw

Dentro de la máquina virtual:

```bash
npm install -g openclaw@latest
openclaw onboard --install-daemon
```

Siga las indicaciones de incorporación para configurar el proveedor de modelos (Anthropic, OpenAI, etc.).

## 7) Configurar los canales

Edite el archivo de configuración:

```bash
nano ~/.openclaw/openclaw.json
```

Añada los canales:

```json5
{
  channels: {
    telegram: {
      botToken: "YOUR_BOT_TOKEN",
    },
    whatsapp: {
      dmPolicy: "allowlist",
      allowFrom: ["+15551234567"],
    },
  },
}
```

Después, inicie sesión en WhatsApp (escanee el código QR):

```bash
openclaw channels login
```

## 8) Ejecutar la máquina virtual sin interfaz gráfica

Detenga la máquina virtual y reiníciela sin pantalla:

```bash
lume stop openclaw
lume run openclaw --no-display
```

La máquina virtual se ejecuta en segundo plano; el daemon de OpenClaw mantiene el Gateway en funcionamiento. Para comprobar el estado:

```bash
ssh youruser@192.168.64.X "openclaw status"
```

## Adicional: integración con iMessage

Esta es la función estrella de ejecutar OpenClaw en macOS. Utilice [iMessage](/es/channels/imessage) con `imsg` para añadir Mensajes a OpenClaw.

Dentro de la máquina virtual:

1. Inicie sesión en Mensajes.
2. Instale `imsg`.
3. Conceda acceso completo al disco y permiso de automatización al proceso que ejecuta OpenClaw/`imsg`.
4. Verifique la compatibilidad con RPC mediante `imsg rpc --help`.

Añada lo siguiente a la configuración de OpenClaw:

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "imsg",
      dbPath: "~/Library/Messages/chat.db",
    },
  },
}
```

Reinicie el Gateway. Ahora el agente puede enviar y recibir mensajes de iMessage. Detalles completos de configuración: [Canal de iMessage](/es/channels/imessage).

## Guardar una imagen maestra

Antes de continuar con la personalización, cree una instantánea del estado limpio:

```bash
lume stop openclaw
lume clone openclaw openclaw-golden
```

Restablézcala en cualquier momento:

```bash
lume stop openclaw && lume delete openclaw
lume clone openclaw-golden openclaw
lume run openclaw --no-display
```

## Ejecución 24/7

Mantenga la máquina virtual en funcionamiento:

- Manteniendo el Mac conectado a la corriente
- Desactivando la suspensión en System Settings -> Energy Saver
- Utilizando `caffeinate` si es necesario

Para un funcionamiento realmente continuo, considere un Mac mini dedicado o un VPS pequeño. Consulte [Alojamiento en VPS](/es/vps).

## Solución de problemas

| Problema                           | Solución                                                                                                  |
| ---------------------------------- | --------------------------------------------------------------------------------------------------------- |
| No se puede acceder a la máquina virtual mediante SSH | Compruebe que "Remote Login" esté activado en System Settings de la máquina virtual                       |
| No aparece la IP de la máquina virtual              | Espere a que la máquina virtual termine de arrancar y vuelva a ejecutar `lume get openclaw`                |
| No se encuentra el comando Lume                    | Añada `~/.local/bin` a PATH                                                                           |
| No se escanea el QR de WhatsApp                    | Asegúrese de haber iniciado sesión en la máquina virtual (no en el sistema anfitrión) al ejecutar `openclaw channels login` |

## Documentación relacionada

- [Alojamiento en VPS](/es/vps)
- [Nodos](/es/nodes)
- [Gateway remoto](/es/gateway/remote)
- [Canal de iMessage](/es/channels/imessage)
- [Inicio rápido de Lume](https://cua.ai/docs/lume/guide/getting-started/quickstart)
- [Referencia de la CLI de Lume](https://cua.ai/docs/lume/reference/cli-reference)
- [Configuración desatendida de máquinas virtuales](https://cua.ai/docs/lume/guide/fundamentals/unattended-setup) (avanzado)
- [Aislamiento con Docker](/es/install/docker) (método de aislamiento alternativo)
