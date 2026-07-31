---
read_when:
    - Se busca un bot asistente personal para Zalo con inicio de sesión mediante código QR
    - Está instalando o solucionando problemas del plugin de canal openclaw-zaloclawbot
summary: Configuración del canal Zalo ClawBot mediante el plugin externo openclaw-zaloclawbot
title: ClawBot de Zalo
x-i18n:
    generated_at: "2026-07-26T05:07:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 76c9f79d114856b86026a5e4b98a43f451b0d3f16dd41a67e9226da4f8b37b33
    source_path: channels/zaloclawbot.md
    workflow: 16
---

OpenClaw se conecta a Zalo ClawBot mediante el plugin externo `@zalo-platforms/openclaw-zaloclawbot` incluido en el catálogo. El inicio de sesión utiliza un código QR de una Zalo Mini App; el id del plugin en la configuración es `openclaw-zaloclawbot`.

## Compatibilidad

| Versión del plugin | Versión de OpenClaw | dist-tag de npm | Estado        |
| -------------- | ---------------- | ------------ | ------------- |
| 0.1.4          | >=2026.4.10      | `latest`     | Activo / Beta |

## Requisitos previos

- Node.js >= 22
- [OpenClaw](https://docs.openclaw.ai/install) instalado (CLI `openclaw` disponible)
- Una cuenta de Zalo en un dispositivo móvil para escanear el código QR de inicio de sesión

## Instalación con el asistente de incorporación (recomendado)

```bash
openclaw onboard
```

Seleccione **Zalo ClawBot** en el menú de canales. El asistente instala el plugin desde el catálogo oficial (con verificación de integridad), muestra el código QR de inicio de sesión en la terminal y completa la configuración del canal una vez que se escanea con la aplicación Zalo.

## Instalación manual

Para añadir el canal a un Gateway que ya haya completado la incorporación:

### 1. Instalar el plugin

```bash
openclaw plugins install "@zalo-platforms/openclaw-zaloclawbot@0.1.4"
```

Utilice la versión fijada exacta para que OpenClaw verifique el paquete con el hash de integridad del catálogo durante la instalación.

### 2. Habilitar el plugin en la configuración

```bash
openclaw config set plugins.entries.openclaw-zaloclawbot.enabled true
```

### 3. Generar un código QR e iniciar sesión

```bash
openclaw channels login --channel openclaw-zaloclawbot
```

Escanee el código QR mostrado en la terminal con la aplicación móvil Zalo, acepte las Condiciones de uso dentro de la Zalo Mini App y autorice la sesión.

### 4. Reiniciar el Gateway

```bash
openclaw gateway restart
```

## Funcionamiento

A diferencia del canal estándar de Zalo, que requiere registrar una cuenta oficial de Zalo (OA) propia y configurar credenciales estáticas de desarrollador, Zalo ClawBot es un **asistente personal vinculado al propietario** que opera en una infraestructura oficial compartida:

1. **Incorporación:** el código QR dirige a una Zalo Mini App que vincula directamente con el ID de usuario de Zalo un bot privado recién aprovisionado bajo una OA oficial compartida.
2. **Privacidad vinculada al propietario:** el bot solo se comunica con su propietario. Los mensajes de otros usuarios se descartan en el nivel de la plataforma.
3. **Ruta de API oficial:** el plugin utiliza las API de Zalo Bot Platform, no la automatización del navegador ni de sesiones web.

## Funcionamiento interno

El plugin se comunica con Zalo mediante un bucle persistente de sondeo largo (`getUpdates`). Los Webhooks están deshabilitados de forma predeterminada para las ejecuciones locales del Gateway en equipos de escritorio o terminales. Los mensajes se procesan en el cliente y se asignan al entorno de ejecución del agente local.

El plugin gestiona las credenciales del bot en el directorio de estado de OpenClaw. Trate ese directorio como información confidencial y aplíquele la misma política de control de acceso y copias de seguridad que al resto del estado de OpenClaw.

El entorno de ejecución de este plugin reside por completo en el paquete externo `@zalo-platforms/openclaw-zaloclawbot`; los detalles de comportamiento que aparecen a continuación, más allá de la instalación y la configuración, proceden de los mantenedores del plugin y no se han verificado con el código fuente del núcleo de OpenClaw.

## Solución de problemas

- **Tiempo de espera agotado al iniciar sesión mediante QR:** el token de inicio de sesión (`zbsk`) caduca después de 5 minutos por motivos de seguridad. Si el código QR caduca antes de escanearlo, vuelva a ejecutar el comando de inicio de sesión para generar uno nuevo.
- **El Gateway no se carga:** confirme que la versión del host de OpenClaw sea `2026.4.10` o posterior. Las versiones anteriores no admiten el registro de instalación de plugins npm externos que requiere este ID.

## Temas relacionados

- [Descripción general de los canales](/es/channels) - todos los canales compatibles
- [Zalo](/es/channels/zalo) - el canal integrado de Zalo Bot Creator / Marketplace
- [Emparejamiento](/es/channels/pairing) - autenticación por mensaje directo y flujo de emparejamiento
- [Plugins](/es/tools/plugin) - instalación y gestión de plugins
