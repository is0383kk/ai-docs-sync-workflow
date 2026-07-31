---
read_when:
    - Desea ejecutar el Gateway en un servidor Linux o un VPS en la nube
    - Necesita un resumen rápido de las guías de alojamiento
    - Quieres una optimización genérica de servidores Linux para OpenClaw
sidebarTitle: Linux Server
summary: Ejecuta OpenClaw en un servidor Linux o VPS en la nube — selector de proveedor, arquitectura y ajustes
title: Servidor Linux
x-i18n:
    generated_at: "2026-07-26T05:26:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 634a246850ab8b854c2c799688fd368ebed3a02124baa85bf38d5ff6ef8cec64
    source_path: vps.md
    workflow: 16
---

Ejecuta el Gateway de OpenClaw en cualquier servidor Linux o VPS en la nube. Esta página ayuda a
elegir un proveedor, explica cómo funcionan los despliegues en la nube y abarca ajustes genéricos de Linux
aplicables en cualquier entorno.

## Elegir un proveedor

<CardGroup cols={2}>
  <Card title="Azure" href="/es/install/azure">Máquina virtual Linux</Card>
  <Card title="DigitalOcean" href="/es/install/digitalocean">VPS de pago sencillo</Card>
  <Card title="exe.dev" href="/es/install/exe-dev">Máquina virtual con proxy HTTPS</Card>
  <Card title="Fly.io" href="/es/install/fly">Fly Machines</Card>
  <Card title="GCP" href="/es/install/gcp">Compute Engine</Card>
  <Card title="Hetzner" href="/es/install/hetzner">Docker en un VPS de Hetzner</Card>
  <Card title="Hostinger" href="/es/install/hostinger">VPS con configuración en un clic</Card>
  <Card title="Northflank" href="/es/install/northflank">Configuración en un clic desde el navegador</Card>
  <Card title="Oracle Cloud" href="/es/install/oracle">Nivel ARM Always Free</Card>
  <Card title="Railway" href="/es/install/railway">Configuración en un clic desde el navegador</Card>
  <Card title="Raspberry Pi" href="/es/install/raspberry-pi">Alojamiento propio en ARM</Card>
</CardGroup>

**AWS (EC2 / Lightsail / nivel gratuito)** también funciona bien.
Hay disponible una guía en vídeo de la comunidad en
[x.com/techfrenAJ/status/2014934471095812547](https://x.com/techfrenAJ/status/2014934471095812547)
(recurso de la comunidad; podría dejar de estar disponible).

## Cómo funcionan las configuraciones en la nube

- El **Gateway se ejecuta en el VPS** y gestiona el estado y el espacio de trabajo.
- La conexión se realiza desde un portátil o teléfono mediante la **interfaz de control** o **Tailscale/SSH**.
- Se debe tratar el VPS como la fuente de verdad y realizar **copias de seguridad** periódicas del estado y el espacio de trabajo.
- Configuración segura predeterminada: mantener el Gateway en la interfaz de bucle invertido y acceder mediante un túnel SSH o Tailscale Serve.
  Si se vincula a `lan` o `tailnet`, el Gateway requiere un secreto compartido
  (`gateway.auth.token` o `gateway.auth.password`), salvo que la autenticación se delegue en un
  proxy de confianza.

Páginas relacionadas: [Acceso remoto al Gateway](/es/gateway/remote), [Centro de plataformas](/es/platforms).

## Proteger primero el acceso administrativo

Antes de instalar OpenClaw en un VPS público, se debe decidir cómo se administrará
el propio servidor.

- Para el acceso administrativo exclusivo desde la red Tailscale: instalar primero Tailscale, unir el VPS a la
  red Tailscale, verificar una segunda sesión SSH mediante la IP de Tailscale o el nombre de MagicDNS
  y, después, restringir el acceso SSH público.
- Sin Tailscale: aplicar una protección equivalente a la ruta SSH antes de
  exponer más servicios.
- Esto es independiente del acceso al Gateway. OpenClaw puede seguir vinculado a la
  interfaz de bucle invertido y se puede usar un túnel SSH o Tailscale Serve para el panel.

Las opciones del Gateway específicas de Tailscale se describen en [Tailscale](/es/gateway/tailscale).

## Agente empresarial compartido en un VPS

Ejecutar un único agente para un equipo es una configuración válida cuando todos los usuarios pertenecen al
mismo límite de confianza y el agente se utiliza exclusivamente para fines empresariales.

- Se debe mantener en un entorno de ejecución dedicado (VPS/máquina virtual/contenedor y usuario/cuentas del sistema operativo exclusivos).
- No se deben iniciar sesiones en ese entorno de ejecución con cuentas personales de Apple/Google ni perfiles personales de navegador o gestor de contraseñas.
- Si los usuarios pueden actuar de forma maliciosa entre sí, se deben separar por Gateway/host/usuario del sistema operativo.

Detalles del modelo de seguridad: [Seguridad](/es/gateway/security).

## Uso de nodos con un VPS

Es posible mantener el Gateway en la nube y emparejar **nodos** en dispositivos locales
(Mac/iOS/Android/sin interfaz). Los nodos proporcionan capacidades locales de pantalla/cámara/lienzo y `system.run`,
mientras el Gateway permanece en la nube.

Documentación: [Nodos](/es/nodes), [CLI de nodos](/es/cli/nodes).

## Ajustes de inicio para máquinas virtuales pequeñas y hosts ARM

Si los comandos de la CLI parecen lentos en máquinas virtuales de baja potencia (o hosts ARM), se puede habilitar la caché de compilación de módulos de Node:

```bash
grep -q 'NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache' ~/.bashrc || cat >> ~/.bashrc <<'EOF'
export NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
mkdir -p /var/tmp/openclaw-compile-cache
export OPENCLAW_NO_RESPAWN=1
EOF
source ~/.bashrc
```

- `NODE_COMPILE_CACHE` mejora los tiempos de inicio de los comandos repetidos; la primera ejecución prepara la caché.
- `OPENCLAW_NO_RESPAWN=1` mantiene los reinicios habituales del Gateway dentro del mismo proceso, lo que evita transferencias adicionales entre procesos y simplifica el seguimiento del PID en hosts pequeños.
- Para obtener información específica sobre Raspberry Pi, consultar [Raspberry Pi](/es/install/raspberry-pi).

### Lista de comprobación de ajustes de systemd (opcional)

Para hosts de máquinas virtuales que utilicen `systemd`, se recomienda considerar:

- Variables de entorno del servicio para una ruta de inicio estable: `OPENCLAW_NO_RESPAWN=1` y
  `NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache`
- Comportamiento de reinicio explícito: `Restart=always`, `RestartSec=2`, `TimeoutStartSec=90`
- Discos respaldados por SSD para las rutas de estado y caché, a fin de reducir las penalizaciones de arranque en frío por operaciones de E/S aleatorias.

La ruta estándar `openclaw onboard --install-daemon` instala una unidad de usuario de
systemd; se puede editar con:

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

Si se ha instalado deliberadamente una unidad del sistema, se debe editar mediante
`sudo systemctl edit openclaw-gateway.service`.

Cómo ayudan las políticas de `Restart=` a la recuperación automatizada:
[systemd puede automatizar la recuperación de servicios](https://www.redhat.com/en/blog/systemd-automate-recovery).

Para obtener información sobre el comportamiento de OOM en Linux, la selección de procesos secundarios que se finalizarán y los
diagnósticos de `exit 137`, consultar [Presión de memoria y finalizaciones por OOM en Linux](/es/platforms/linux#memory-pressure-and-oom-kills).

## Contenido relacionado

- [Descripción general de la instalación](/es/install)
- [DigitalOcean](/es/install/digitalocean)
- [Fly.io](/es/install/fly)
- [Hetzner](/es/install/hetzner)
