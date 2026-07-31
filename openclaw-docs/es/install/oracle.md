---
read_when:
    - Configuración de OpenClaw en Oracle Cloud
    - Buscando alojamiento VPS gratuito para OpenClaw
    - ¿Desea ejecutar OpenClaw las 24 horas del día, los 7 días de la semana, en un servidor pequeño?
summary: Aloja OpenClaw en el nivel ARM Always Free de Oracle Cloud
title: Oracle Cloud
x-i18n:
    generated_at: "2026-07-26T04:45:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5e1eb95b6bc8ad73e1492a03d8ebe32d89c80e58347614e6ae12d2d3d926d577
    source_path: install/oracle.md
    workflow: 16
---

Ejecute un Gateway de OpenClaw persistente en el nivel ARM **Always Free** de Oracle Cloud (hasta 4 OCPU, 24 GB de RAM y 200 GB de almacenamiento) sin coste alguno.

## Requisitos previos

- Cuenta de Oracle Cloud ([registro](https://www.oracle.com/cloud/free/)); si tiene problemas, consulte la [guía de registro de la comunidad](https://gist.github.com/rssnyder/51e3cfedd730e7dd5f4a816143b25dbd)
- Cuenta de Tailscale (gratuita en [tailscale.com](https://tailscale.com))
- Un par de claves SSH
- Unos 30 minutos

## Configuración

<Steps>
  <Step title="Crear una instancia de OCI">
    1. Inicie sesión en la [consola de Oracle Cloud](https://cloud.oracle.com/).
    2. Vaya a **Compute > Instances > Create Instance**.
    3. Configure lo siguiente:
       - **Name:** `openclaw`
       - **Image:** Ubuntu 24.04 (aarch64)
       - **Shape:** `VM.Standard.A1.Flex` (Ampere ARM)
       - **OCPUs:** 2 (o hasta 4)
       - **Memory:** 12 GB (o hasta 24 GB)
       - **Boot volume:** 50 GB (hasta 200 GB gratis)
       - **SSH key:** añada su clave pública
    4. Haga clic en **Create** y anote la dirección IP pública.

    <Tip>
    Si la creación de la instancia falla con el mensaje "Out of capacity", pruebe otro dominio de disponibilidad o vuelva a intentarlo más tarde. La capacidad del nivel gratuito es limitada.
    </Tip>

  </Step>

  <Step title="Conectarse y actualizar el sistema">
    ```bash
    ssh ubuntu@YOUR_PUBLIC_IP

    sudo apt update && sudo apt upgrade -y
    sudo apt install -y build-essential
    ```

    `build-essential` es necesario para compilar algunas dependencias en ARM.

  </Step>

  <Step title="Configurar el usuario y el nombre de host">
    ```bash
    sudo hostnamectl set-hostname openclaw
    sudo passwd ubuntu
    sudo loginctl enable-linger ubuntu
    ```

    Habilitar la permanencia mantiene los servicios del usuario en ejecución después de cerrar la sesión.

  </Step>

  <Step title="Instalar Tailscale">
    ```bash
    curl -fsSL https://tailscale.com/install.sh | sh
    sudo tailscale up --ssh --hostname=openclaw
    ```

    A partir de ahora, conéctese mediante Tailscale: `ssh ubuntu@openclaw`.

  </Step>

  <Step title="Instalar OpenClaw">
    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash
    source ~/.bashrc
    ```

    Cuando aparezca la pregunta "How do you want to hatch your bot?", seleccione **Do this later**.

  </Step>

  <Step title="Configurar el Gateway">
    Use la autenticación mediante token con Tailscale Serve para obtener acceso remoto seguro.

    ```bash
    openclaw config set gateway.bind loopback
    openclaw config set gateway.auth.mode token
    openclaw doctor --generate-gateway-token
    openclaw config set gateway.tailscale.mode serve
    openclaw config set gateway.trustedProxies '["127.0.0.1"]'

    systemctl --user restart openclaw-gateway.service
    ```

    En este caso, `gateway.trustedProxies=["127.0.0.1"]` solo se utiliza para gestionar la IP reenviada y el cliente local del proxy local de Tailscale Serve. **No** es `gateway.auth.mode: "trusted-proxy"`. Las rutas del visor de diferencias mantienen un comportamiento de cierre seguro en esta configuración: las solicitudes directas del visor `127.0.0.1` sin encabezados reenviados del proxy devuelven `Diff not found`. Use `mode=file` / `mode=both` para los archivos adjuntos, o habilite intencionadamente los visores remotos y establezca `plugins.entries.diffs.config.viewerBaseUrl` (o pase un `baseUrl` del proxy) si necesita enlaces compartibles al visor.

  </Step>

  <Step title="Restringir la seguridad de la VCN">
    Bloquee todo el tráfico excepto Tailscale en el perímetro de la red:

    1. Vaya a **Networking > Virtual Cloud Networks** en la consola de OCI.
    2. Haga clic en su VCN y, después, en **Security Lists > Default Security List**.
    3. **Elimine** todas las reglas de entrada excepto `0.0.0.0/0 UDP 41641` (Tailscale).
    4. Mantenga las reglas de salida predeterminadas (permitir todo el tráfico saliente).

    Esto bloquea SSH en el puerto 22, HTTP, HTTPS y cualquier otro tráfico en el perímetro de la red. A partir de este momento, solo podrá conectarse mediante Tailscale.

  </Step>

  <Step title="Verificar">
    ```bash
    openclaw --version
    systemctl --user status openclaw-gateway.service
    tailscale serve status
    curl http://localhost:18789
    ```

    Acceda a la interfaz de control desde cualquier dispositivo de su tailnet:

    ```
    https://openclaw.<tailnet-name>.ts.net/
    ```

    Sustituya `<tailnet-name>` por el nombre de su tailnet (visible en `tailscale status`).

  </Step>
</Steps>

## Verificar la postura de seguridad

Con la VCN restringida (solo está abierto UDP 41641) y el Gateway vinculado a la interfaz de bucle invertido, el tráfico público queda bloqueado en el perímetro de la red y el acceso administrativo se limita a la tailnet. Esto elimina la necesidad de aplicar varias medidas tradicionales de protección de VPS:

| Medida tradicional                 | ¿Es necesaria?    | Motivo                                                                            |
| ---------------------------------- | ----------------- | --------------------------------------------------------------------------------- |
| Cortafuegos UFW                    | No                | La VCN bloquea el tráfico antes de que llegue a la instancia.                     |
| fail2ban                           | No                | El puerto 22 está bloqueado en la VCN; no existe una superficie de fuerza bruta.  |
| Protección de sshd                 | No                | Tailscale SSH no utiliza sshd.                                                    |
| Deshabilitar el inicio como root   | No                | Tailscale autentica mediante la identidad de la tailnet, no usuarios del sistema. |
| Autenticación solo con claves SSH  | No                | Lo mismo: la identidad de la tailnet sustituye las claves SSH del sistema.        |
| Protección de IPv6                 | Normalmente no    | Depende de la configuración de la VCN/subred; verifique qué se asigna o expone.    |

Se sigue recomendando lo siguiente:

- `chmod 700 ~/.openclaw` para restringir los permisos de los archivos de credenciales.
- `openclaw security audit` para realizar una comprobación de postura específica de OpenClaw.
- Ejecutar periódicamente `sudo apt update && sudo apt upgrade` para aplicar parches del sistema operativo.
- Revisar periódicamente los dispositivos en la [consola de administración de Tailscale](https://login.tailscale.com/admin).

Comandos rápidos de verificación:

```bash
# Confirmar que no hay puertos públicos a la escucha
sudo ss -tlnp | grep -v '127.0.0.1\|::1'

# Verificar que Tailscale SSH está activo
tailscale status | grep -q 'offers: ssh' && echo "Tailscale SSH activo"

# Opcional: deshabilitar sshd por completo tras confirmar que Tailscale SSH funciona
sudo systemctl disable --now ssh
```

## Notas sobre ARM

El nivel Always Free utiliza ARM (`aarch64`). La mayoría de las funciones de OpenClaw funcionan correctamente; una pequeña cantidad de binarios nativos requieren compilaciones para ARM:

- Node.js, Telegram y WhatsApp (Baileys): JavaScript puro, sin problemas.
- La mayoría de los paquetes npm con código nativo: hay artefactos `linux-arm64` precompilados disponibles.
- Herramientas auxiliares opcionales de la CLI (por ejemplo, binarios de Go/Rust distribuidos mediante Skills): compruebe si existe una versión `aarch64` / `linux-arm64` antes de instalarlas.

Verifique la arquitectura con `uname -m` (debe mostrar `aarch64`). Para los binarios sin una compilación para ARM, instálelos desde el código fuente u omítalos.

## Persistencia y copias de seguridad

El estado de OpenClaw se almacena en:

- `~/.openclaw/`: `openclaw.json`, `auth-profiles.json` por agente, estado de canales/proveedores y datos de sesiones.
- `~/.openclaw/workspace/`: el espacio de trabajo del agente (SOUL.md, memoria y artefactos).

Estos datos se conservan tras los reinicios. Para crear una instantánea portátil:

```bash
openclaw backup create
```

## Alternativa: túnel SSH

Si Tailscale Serve no funciona, utilice un túnel SSH desde su equipo local:

```bash
ssh -L 18789:127.0.0.1:18789 ubuntu@openclaw
```

A continuación, abra `http://localhost:18789`.

## Solución de problemas

**La creación de la instancia falla ("Out of capacity")**: las instancias ARM del nivel gratuito son muy populares. Pruebe otro dominio de disponibilidad o vuelva a intentarlo durante las horas de menor demanda.

**Tailscale no se conecta**: ejecute `sudo tailscale up --ssh --hostname=openclaw --reset` para volver a autenticarse.

**El Gateway no se inicia**: ejecute `openclaw doctor --non-interactive` y consulte los registros con `journalctl --user -u openclaw-gateway.service -n 50`.

**Problemas con binarios ARM**: la mayoría de los paquetes npm funcionan en ARM64. Para los binarios nativos, busque versiones `linux-arm64` o `aarch64`. Verifique la arquitectura con `uname -m`.

## Próximos pasos

- [Canales](/es/channels): conecte Telegram, WhatsApp, Discord y otros servicios
- [Configuración del Gateway](/es/gateway/configuration): todas las opciones de configuración
- [Actualización](/es/install/updating): mantenga OpenClaw actualizado

## Contenido relacionado

- [Descripción general de la instalación](/es/install)
- [GCP](/es/install/gcp)
- [Alojamiento en VPS](/es/vps)
