---
read_when:
    - Configuración de OpenClaw en una Raspberry Pi
    - Ejecución de OpenClaw en dispositivos ARM
    - Creación de una IA personal económica y siempre activa
summary: Aloja OpenClaw en una Raspberry Pi para un autoalojamiento siempre activo
title: Raspberry Pi
x-i18n:
    generated_at: "2026-07-26T05:18:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 60f8f3b23577155658d410993937ebe7c34c21f71c1bd7d9b0c453f15c4aa024
    source_path: install/raspberry-pi.md
    workflow: 16
---

Ejecute un Gateway de OpenClaw persistente y siempre activo en una Raspberry Pi. Puesto que la Pi funciona únicamente como Gateway (los modelos se ejecutan en la nube mediante una API), incluso una Pi modesta gestiona bien la carga de trabajo; el coste habitual del hardware es de **$35-80 en un único pago**, sin cuotas mensuales.

## Compatibilidad de hardware

| Modelo de Pi | RAM    | ¿Funciona? | Notas                                      |
| ------------ | ------ | ---------- | ------------------------------------------ |
| Pi 5         | 4/8 GB | Óptimo     | El más rápido; recomendado.                |
| Pi 4         | 4 GB   | Bien       | La opción ideal para la mayoría de usuarios. |
| Pi 4         | 2 GB   | Aceptable  | Añada espacio de intercambio.              |
| Pi 4         | 1 GB   | Limitado   | Posible con espacio de intercambio y una configuración mínima. |
| Pi 3B+       | 1 GB   | Lento      | Funciona, pero con lentitud.               |
| Pi Zero 2 W  | 512 MB | No         | No se recomienda.                          |

**Mínimo:** 1 GB de RAM, 1 núcleo, 500 MB de espacio libre en disco y un SO de 64 bits.
**Recomendado:** 2 GB o más de RAM, tarjeta SD de 16 GB o más (o SSD USB) y Ethernet.

## Requisitos previos

- Raspberry Pi 4 o 5 con 2 GB o más de RAM (se recomiendan 4 GB)
- Tarjeta microSD (16 GB o más) o SSD USB (mejor rendimiento)
- Fuente de alimentación oficial de Pi
- Conexión de red (Ethernet o WiFi)
- Raspberry Pi OS de 64 bits (obligatorio; no utilice la versión de 32 bits)
- Unos 30 minutos

## Configuración

<Steps>
  <Step title="Grabar el SO">
    Utilice **Raspberry Pi OS Lite (64-bit)**; no se necesita un escritorio para un servidor sin monitor.

    1. Descargue [Raspberry Pi Imager](https://www.raspberrypi.com/software/).
    2. Elija el SO: **Raspberry Pi OS Lite (64-bit)**.
    3. En el cuadro de diálogo de configuración, preconfigure:
       - Nombre de host: `gateway-host`
       - Active SSH
       - Establezca el nombre de usuario y la contraseña
       - Configure WiFi (si no utiliza Ethernet)
    4. Grabe la imagen en la tarjeta SD o unidad USB, insértela e inicie la Pi.

  </Step>

  <Step title="Conectarse mediante SSH">
    ```bash
    ssh user@gateway-host
    ```
  </Step>

  <Step title="Actualizar el sistema">
    ```bash
    sudo apt update && sudo apt upgrade -y
    sudo apt install -y git curl build-essential

    # Establecer la zona horaria (importante para cron y los recordatorios)
    sudo timedatectl set-timezone America/Chicago
    ```

  </Step>

  <Step title="Instalar Node.js 24">
    ```bash
    curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
    sudo apt install -y nodejs
    node --version
    ```
  </Step>

  <Step title="Añadir espacio de intercambio (importante con 2 GB o menos)">
    ```bash
    sudo fallocate -l 2G /swapfile
    sudo chmod 600 /swapfile
    sudo mkswap /swapfile
    sudo swapon /swapfile
    echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

    # Reducir la tendencia al intercambio en dispositivos con poca RAM
    echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
    sudo sysctl -p
    ```

  </Step>

  <Step title="Instalar OpenClaw">
    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash
    ```
  </Step>

  <Step title="Ejecutar la incorporación">
    ```bash
    openclaw onboard --install-daemon
    ```

    Siga el asistente. Para dispositivos sin monitor, se recomiendan las claves de API en lugar de OAuth. Telegram es el canal más sencillo para empezar.

  </Step>

  <Step title="Verificar">
    ```bash
    openclaw status
    systemctl --user status openclaw-gateway.service
    journalctl --user -u openclaw-gateway.service -f
    ```
  </Step>

  <Step title="Acceder a la interfaz de control">
    En el ordenador, obtenga una URL del panel desde la Pi:

    ```bash
    ssh user@gateway-host 'openclaw dashboard --no-open'
    ```

    Después, cree un túnel SSH en otro terminal:

    ```bash
    ssh -N -L 18789:127.0.0.1:18789 user@gateway-host
    ```

    Abra la URL mostrada en el navegador local. Para disponer de acceso remoto permanente, consulte la [integración con Tailscale](/es/gateway/tailscale).

  </Step>
</Steps>

## Consejos de rendimiento

**Utilice un SSD USB**; las tarjetas SD son lentas y se desgastan. Un SSD USB mejora considerablemente el rendimiento y soporta más ciclos de escritura; utilícelo para `OPENCLAW_STATE_DIR` si mantiene el SO en la tarjeta SD. Consulte la [guía de arranque USB de Pi](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#usb-mass-storage-boot).

**Active la caché de compilación de módulos**; acelera las invocaciones repetidas de la CLI en hosts Pi de menor potencia. `OPENCLAW_NO_RESPAWN=1` mantiene los reinicios rutinarios del Gateway dentro del proceso, evita transferencias adicionales entre procesos y simplifica el seguimiento del PID en hosts pequeños:

```bash
grep -q 'NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache' ~/.bashrc || cat >> ~/.bashrc <<'EOF' # pragma: allowlist secret
export NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
mkdir -p /var/tmp/openclaw-compile-cache
export OPENCLAW_NO_RESPAWN=1
EOF
source ~/.bashrc
```

Utilice `/var/tmp`, no `/tmp`; algunas distribuciones borran `/tmp` durante el arranque, lo que elimina la caché preparada.

**Reduzca el uso de memoria**; para configuraciones sin monitor, libere memoria de la GPU y desactive los servicios que no se utilicen:

```bash
echo 'gpu_mem=16' | sudo tee -a /boot/config.txt
sudo systemctl disable bluetooth
```

**Configuración adicional de systemd para reinicios estables**; si esta Pi se utiliza principalmente para ejecutar OpenClaw, añada una configuración adicional al servicio:

```bash
systemctl --user edit openclaw-gateway.service
```

```ini
[Service]
Environment=OPENCLAW_NO_RESPAWN=1
Environment=NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
Restart=always
RestartSec=2
TimeoutStartSec=90
```

A continuación, `systemctl --user daemon-reload && systemctl --user restart openclaw-gateway.service`. En una Pi sin monitor, active también la permanencia una vez para que el servicio de usuario continúe ejecutándose después de cerrar la sesión: `sudo loginctl enable-linger "$(whoami)"`.

## Configuración de modelos recomendada

Como la Pi solo ejecuta el Gateway, utilice modelos de API alojados en la nube; no ejecute LLM locales en una Pi, ya que incluso los modelos pequeños son demasiado lentos para resultar útiles:

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "anthropic/claude-sonnet-4-6",
        "fallbacks": ["openai/gpt-5.4-mini"]
      }
    }
  }
}
```

## Notas sobre binarios ARM

La mayoría de las funciones de OpenClaw funcionan en ARM64 sin cambios (Node.js, Telegram, WhatsApp/Baileys y Chromium). Los binarios que ocasionalmente carecen de compilaciones para ARM suelen ser herramientas CLI opcionales de Go o Rust distribuidas mediante Skills. Compruebe la arquitectura con `uname -m` (debería mostrar `aarch64`) y, a continuación, consulte la página de versiones del binario que falte para buscar artefactos `linux-arm64` / `aarch64` antes de recurrir a la compilación desde el código fuente.

## Persistencia y copias de seguridad

El estado de OpenClaw se almacena en:

- `~/.openclaw/` -- `openclaw.json`, `auth-profiles.json` por agente, estado de canales/proveedores y sesiones.
- `~/.openclaw/workspace/` -- espacio de trabajo del agente (SOUL.md, memoria y artefactos).

Estos datos sobreviven a los reinicios y se benefician del uso de un SSD en lugar de una tarjeta SD, tanto en rendimiento como en durabilidad. Cree una instantánea portátil con:

```bash
openclaw backup create
```

## Solución de problemas

**Memoria insuficiente**; compruebe con `free -h` que el espacio de intercambio esté activo. Desactive los servicios que no se utilicen (`sudo systemctl disable cups bluetooth avahi-daemon`). Utilice únicamente modelos basados en API.

**Rendimiento lento**; utilice un SSD USB en lugar de una tarjeta SD. Compruebe si existe limitación de la CPU con `vcgencmd get_throttled` (debería devolver `0x0`).

**El servicio no se inicia**; consulte los registros con `journalctl --user -u openclaw-gateway.service --no-pager -n 100` y ejecute `openclaw doctor --non-interactive`. Si se trata de una Pi sin monitor, compruebe también que la permanencia esté activada: `sudo loginctl enable-linger "$(whoami)"`.

**Problemas con binarios ARM**; si una Skill falla con "exec format error", compruebe si el binario dispone de una compilación para ARM64. Verifique la arquitectura con `uname -m` (debería mostrar `aarch64`).

**Interrupciones de WiFi**; desactive la administración de energía de WiFi: `sudo iwconfig wlan0 power off`.

## Pasos siguientes

- [Canales](/es/channels) -- conecte Telegram, WhatsApp, Discord y otros servicios
- [Configuración del Gateway](/es/gateway/configuration) -- todas las opciones de configuración
- [Actualización](/es/install/updating) -- mantenga OpenClaw actualizado

## Contenido relacionado

- [Descripción general de la instalación](/es/install)
- [Servidor Linux](/es/vps)
- [Plataformas](/es/platforms)
