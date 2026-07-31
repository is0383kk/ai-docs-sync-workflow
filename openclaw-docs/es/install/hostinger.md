---
read_when:
    - Configuración de OpenClaw en Hostinger
    - ¿Busca un VPS administrado para OpenClaw?
    - Uso de OpenClaw con 1 clic de Hostinger
summary: Alojar OpenClaw en Hostinger
title: Hostinger
x-i18n:
    generated_at: "2026-07-26T05:17:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7dc49e741f8581928553e2426ed91f92df6e7b0c31dd8780c0d6e891a07be263
    source_path: install/hostinger.md
    workflow: 16
---

Ejecute un Gateway persistente de OpenClaw en [Hostinger](https://www.hostinger.com/openclaw), ya sea como un despliegue administrado con **1-Click** o como una instalación en un **VPS** que administre por cuenta propia.

## Requisitos previos

- Cuenta de Hostinger ([registro](https://www.hostinger.com/openclaw))
- Entre 5 y 10 minutos aproximadamente

## Opción A: OpenClaw con 1-Click

Hostinger se encarga de la infraestructura, Docker y las actualizaciones automáticas. Es la forma más rápida de disponer de una instancia en ejecución.

<Steps>
  <Step title="Comprar e iniciar">
    1. En la [página de OpenClaw de Hostinger](https://www.hostinger.com/openclaw), elija un plan Managed OpenClaw y complete el proceso de compra.

    <Note>
    Durante el proceso de compra, puede seleccionar créditos de **Ready-to-Use AI**, que se compran por adelantado y se integran al instante en OpenClaw; no se necesitan cuentas externas ni claves de API de otros proveedores. Puede empezar a chatear de inmediato. Como alternativa, proporcione su propia clave de Anthropic, OpenAI, Google Gemini o xAI durante la configuración.
    </Note>

  </Step>

  <Step title="Seleccionar un canal de mensajería">
    Elija uno o más canales para conectar:

    - **WhatsApp** -- escanee el código QR que aparece en el asistente de configuración.
    - **Telegram** -- pegue el token del bot obtenido de [BotFather](https://t.me/BotFather).

  </Step>

  <Step title="Completar la instalación">
    Haga clic en **Finish** para desplegar la instancia. Cuando esté lista, acceda al panel de OpenClaw desde **OpenClaw Overview** en hPanel.
  </Step>

</Steps>

## Opción B: OpenClaw en un VPS

Ofrece más control sobre el servidor. Hostinger despliega OpenClaw mediante Docker en su VPS; puede administrarlo a través de **Docker Manager** en hPanel.

<Steps>
  <Step title="Comprar un VPS">
    1. En la [página de OpenClaw de Hostinger](https://www.hostinger.com/openclaw), elija un plan OpenClaw on VPS y complete el proceso de compra.

    <Note>
    Puede seleccionar créditos de **Ready-to-Use AI** durante el proceso de compra; se compran por adelantado y se integran al instante en OpenClaw, por lo que puede empezar a chatear sin cuentas externas ni claves de API de otros proveedores.
    </Note>

  </Step>

  <Step title="Configurar OpenClaw">
    Una vez aprovisionado el VPS, complete los campos de configuración:

    - **Token del Gateway** -- se genera automáticamente; guárdelo para usarlo más adelante.
    - **Número de WhatsApp** -- su número con el código de país (opcional).
    - **Token del bot de Telegram** -- obtenido de [BotFather](https://t.me/BotFather) (opcional).
    - **Claves de API** -- solo son necesarias si no seleccionó créditos de Ready-to-Use AI durante el proceso de compra.

  </Step>

  <Step title="Iniciar OpenClaw">
    Haga clic en **Deploy**. Cuando esté en ejecución, abra el panel de OpenClaw desde hPanel haciendo clic en **Open**.
  </Step>

</Steps>

Los registros, reinicios y actualizaciones se gestionan desde la interfaz Docker Manager en hPanel. Para actualizar, pulse **Update** en Docker Manager a fin de descargar la imagen más reciente.

## Verificar la configuración

Envíe «Hola» a su asistente en el canal que conectó. OpenClaw responderá y le guiará por las preferencias iniciales.

## Solución de problemas

**El panel no se carga** -- Espere unos minutos hasta que el contenedor termine de aprovisionarse y, a continuación, consulte los registros de Docker Manager en hPanel.

**El contenedor de Docker continúa reiniciándose** -- Abra los registros de Docker Manager y busque errores de configuración (tokens ausentes o claves de API no válidas).

**El bot de Telegram no responde** -- Si se requiere el emparejamiento de mensajes directos, los remitentes desconocidos reciben un código de emparejamiento corto en lugar de una respuesta. Apruébelo desde el chat del panel de OpenClaw o con `openclaw pairing approve telegram <CODE>` si tiene acceso mediante shell al contenedor. Consulte [Emparejamiento](/es/channels/pairing).

## Próximos pasos

- [Canales](/es/channels) -- conecte Telegram, WhatsApp, Discord y otros servicios
- [Configuración del Gateway](/es/gateway/configuration) -- todas las opciones de configuración

## Contenido relacionado

- [Descripción general de la instalación](/es/install)
- [Alojamiento en VPS](/es/vps)
- [DigitalOcean](/es/install/digitalocean)
