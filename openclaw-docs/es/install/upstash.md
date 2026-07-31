---
read_when:
    - Despliegue de OpenClaw en Upstash Box
    - Se necesita un entorno Linux gestionado para OpenClaw con acceso al panel mediante un túnel SSH
summary: Aloja OpenClaw en Upstash Box con mantenimiento de actividad y acceso mediante túnel SSH
title: Upstash Box
x-i18n:
    generated_at: "2026-07-26T04:45:39Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 29232c43e0e4940b7445ab8896c9ccd3e81d0fdbdd522d7f50cb8c8057ac18f0
    source_path: install/upstash.md
    workflow: 16
---

Ejecuta un Gateway de OpenClaw persistente en Upstash Box, un entorno Linux administrado
con compatibilidad de ciclo de vida keep-alive.

Usa un túnel SSH para acceder al panel. No expongas el puerto del Gateway directamente
a la Internet pública.

## Requisitos previos

- Cuenta de Upstash
- Upstash Box con keep-alive
- Cliente SSH en la máquina local

## Crear un Box

Crea un Box con keep-alive en la consola de Upstash. Anota el ID del Box (por ejemplo,
`right-flamingo-14486`) y la clave de API del Box.

Upstash mantiene su guía actual para OpenClaw Box en
[Configuración de OpenClaw](https://upstash.com/docs/box/guides/openclaw-setup).

## Conectarse mediante un túnel SSH

Reenvía el puerto del panel de OpenClaw a la máquina local. Usa la clave de API del Box
como contraseña SSH cuando se solicite:

```bash
ssh -o ServerAliveInterval=15 -o ServerAliveCountMax=3 -L 18789:127.0.0.1:18789 <box-id>@us-east-1.box.upstash.com
```

Las opciones de keep-alive reducen las desconexiones del túnel por inactividad durante la incorporación.

## Instalar OpenClaw

Dentro del Box:

```bash
sudo npm install -g openclaw
```

## Ejecutar la incorporación

```bash
openclaw onboard --install-daemon
```

Sigue las indicaciones. Copia la URL y el token del panel cuando finalice la incorporación.

## Iniciar el Gateway

Configura el Gateway para la red del Box e inícialo en segundo plano:

```bash
openclaw config set gateway.bind lan
nohup openclaw gateway > gateway.log 2>&1 &
```

Con el túnel SSH activo, abre localmente la URL del panel:

```text
http://127.0.0.1:18789/#token=<your-token>
```

## Reinicio automático

Configura este comando como script de inicio del Box para que el Gateway se reinicie cuando se
inicie el Box:

```bash
nohup openclaw gateway > gateway.log 2>&1 &
```

## Solución de problemas

Si SSH se bloquea durante la incorporación, vuelve a conectarte con una configuración SSH limpia y
opciones de keep-alive:

```bash
ssh -F /dev/null -o ControlMaster=no -o ServerAliveInterval=15 -o ServerAliveCountMax=3 -L 18789:127.0.0.1:18789 <box-id>@us-east-1.box.upstash.com
```

Esto omite la configuración local obsoleta de `~/.ssh/config` y mantiene activo el túnel
durante los periodos de inactividad de la red.

## Contenido relacionado

- [Acceso remoto](/es/gateway/remote)
- [Seguridad del Gateway](/es/gateway/security)
- [Actualizar OpenClaw](/es/install/updating)
