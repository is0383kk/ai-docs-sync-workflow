---
read_when:
    - Despliegue de OpenClaw en Northflank
    - Quiere un despliegue en la nube con un solo clic y una interfaz de control basada en navegador.
summary: Despliega OpenClaw en Northflank con una plantilla de un solo clic
title: Northflank
x-i18n:
    generated_at: "2026-07-26T04:41:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 16bb96fdf470999e15e163b6227d228ce8b60b9a172eb74cadc87bddd3955957
    source_path: install/northflank.mdx
    workflow: 16
---

Implementa OpenClaw en Northflank con una plantilla de un solo clic y accede mediante la interfaz web de control. Esta es la forma más sencilla de hacerlo «sin terminal en el servidor»: Northflank ejecuta el Gateway por ti.

## Cómo empezar

1. Haz clic en [Implementar OpenClaw](https://northflank.com/stacks/deploy-openclaw) para abrir la plantilla.
2. Crea una [cuenta en Northflank](https://app.northflank.com/signup) si aún no tienes una.
3. Haz clic en **Deploy OpenClaw now**.
4. Establece la variable de entorno obligatoria: `OPENCLAW_GATEWAY_TOKEN` (usa un valor aleatorio seguro).
5. Haz clic en **Deploy stack** para compilar y ejecutar la plantilla de OpenClaw.
6. Espera a que finalice la implementación y, a continuación, haz clic en **View resources**.
7. Abre el servicio de OpenClaw.
8. Abre la URL pública de OpenClaw en `/openclaw` y conéctate con el secreto compartido configurado. Esta plantilla usa `OPENCLAW_GATEWAY_TOKEN` de forma predeterminada; si lo sustituyes por autenticación mediante contraseña, usa esa contraseña en su lugar.

## Qué se obtiene

- Gateway de OpenClaw alojado + interfaz de control
- Almacenamiento persistente mediante un volumen de Northflank (`/data`) para que `openclaw.json`, los `auth-profiles.json` de cada agente, el estado de los canales y proveedores, las sesiones y el espacio de trabajo se conserven tras nuevas implementaciones

## Conectar un canal

Usa la interfaz de control en `/openclaw` o ejecuta `openclaw onboard` mediante SSH para obtener instrucciones sobre cómo configurar canales:

- [Telegram](/es/channels/telegram) (la opción más rápida, solo requiere un token de bot)
- [Discord](/es/channels/discord)
- [Todos los canales](/es/channels)

## Pasos siguientes

- Configura los canales de mensajería: [Canales](/es/channels)
- Configura el Gateway: [Configuración del Gateway](/es/gateway/configuration)
- Mantén OpenClaw actualizado: [Actualización](/es/install/updating)
