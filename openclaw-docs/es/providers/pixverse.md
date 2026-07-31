---
read_when:
    - Quieres usar la generación de vídeo de PixVerse en OpenClaw
    - Necesitas configurar la clave de API y las variables de entorno de PixVerse.
    - Se desea establecer PixVerse como el proveedor de vídeo predeterminado
summary: Configuración de la generación de vídeos de PixVerse en OpenClaw
title: PixVerse
x-i18n:
    generated_at: "2026-07-26T04:56:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3dba881e877e3da4677a40dff736cb46de114337a1e0338ef8220dcd8e616f46
    source_path: providers/pixverse.md
    workflow: 16
---

OpenClaw proporciona `pixverse` como Plugin externo oficial para la generación alojada de vídeos de PixVerse. El Plugin registra el proveedor `pixverse` con el contrato `videoGenerationProviders`.

| Propiedad          | Valor                                                                |
| ------------------ | -------------------------------------------------------------------- |
| Id. del proveedor  | `pixverse`                                                   |
| Paquete del Plugin | `@openclaw/pixverse-provider`                                                   |
| Variable de entorno de autenticación | `PIXVERSE_API_KEY`                                  |
| Indicador de incorporación | `--auth-choice pixverse-api-key`                                          |
| Indicador directo de la CLI | `--pixverse-api-key <key>`                                         |
| API                | API v2 de la plataforma PixVerse (envío mediante `video_id` y consulta periódica del resultado) |
| Modelo predeterminado | `pixverse/v6`                                                |
| Región predeterminada de la API | Internacional                                            |

## Primeros pasos

<Steps>
  <Step title="Instalar el Plugin">
    ```bash
    openclaw plugins install @openclaw/pixverse-provider
    openclaw gateway restart
    ```
  </Step>
  <Step title="Establecer la clave de API">
    ```bash
    openclaw onboard --auth-choice pixverse-api-key
    ```

    El asistente solicita el endpoint Internacional o CN (consulte la región de la API
    más adelante) antes de escribir `region` y `baseUrl` en la configuración del proveedor.
    Las ejecuciones no interactivas (clave procedente de `--pixverse-api-key` o `PIXVERSE_API_KEY`)
    usan Internacional de forma predeterminada.

    La incorporación también establece `agents.defaults.mediaModels.video.primary` en
    `pixverse/v6` cuando todavía no se ha configurado ningún modelo de vídeo predeterminado.

  </Step>
  <Step title="Cambiar un proveedor de vídeo predeterminado existente (opcional)">
    ```bash
    openclaw config set agents.defaults.mediaModels.video.primary "pixverse/v6"
    ```
  </Step>
  <Step title="Generar un vídeo">
    Solicite al agente que genere un vídeo. PixVerse se utilizará automáticamente.
  </Step>
</Steps>

## Modos y modelos compatibles

El proveedor expone los modelos de generación de PixVerse mediante la herramienta de vídeo compartida de OpenClaw.

| Modo            | Modelos              | Entrada de referencia   |
| --------------- | -------------------- | ----------------------- |
| Texto a vídeo   | `v6` (predeterminado), `c1` | Ninguna                 |
| Imagen a vídeo  | `v6` (predeterminado), `c1` | 1 imagen local o remota |

Las referencias a imágenes locales se cargan en PixVerse antes de la solicitud de imagen a vídeo. Las URL de imágenes remotas se transfieren al endpoint de carga de imágenes de PixVerse como `image_url`.

| Opción          | Valores compatibles                                                                                                              |
| --------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| Duración        | 1-15 segundos (valor predeterminado: 5)                                                                                          |
| Resolución      | `360P`, `540P`, `720P`, `1080P` (valor predeterminado: `540P`; las solicitudes `480P` se asignan a `540P`) |
| Relación de aspecto | `16:9` (predeterminada), `4:3`, `1:1`, `3:4`, `9:16`, `2:3`, `3:2`, `21:9`; solo para texto a vídeo; imagen a vídeo utiliza la imagen de origen |
| Audio generado  | `audio: true`                                                                                                               |

<Note>
La generación de plantillas de imagen de PixVerse todavía no se expone mediante `image_generate`. Esa API se basa en identificadores de plantilla, mientras que el contrato compartido de generación de imágenes de OpenClaw no dispone actualmente de un conjunto tipado de opciones específico de PixVerse.
</Note>

## Opciones del proveedor

El proveedor de vídeo acepta estas claves opcionales específicas del proveedor:

| Opción                               | Tipo   | Efecto                                        |
| ------------------------------------ | ------ | --------------------------------------------- |
| `seed`                   | number | Semilla determinista, de 0 a 2147483647       |
| `negativePrompt` / `negative_prompt` | string | Prompt negativo                           |
| `quality`                   | string | Calidad de PixVerse, como `720p`  |
| `motionMode` / `motion_mode` | string | Modo de movimiento de imagen a vídeo (valor predeterminado: `normal`) |
| `cameraMovement` / `camera_movement` | string | Preajuste de movimiento de cámara de PixVerse |
| `templateId` / `template_id` | number | Id. de la plantilla de PixVerse activada  |

## Configuración

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "pixverse/v6",
      },
    },
  },
}
```

## Configuración avanzada

<AccordionGroup>
  <Accordion title="Región de la API">
    | Valor de la región | URL base de la API de PixVerse                 |
    | ------------------ | ---------------------------------------------- |
    | `international` | `https://app-api.pixverse.ai/openapi/v2`                             |
    | `cn` | `https://app-api.pixverseai.cn/openapi/v2`                             |

    Establezca `models.providers.pixverse.region` manualmente cuando la clave pertenezca a una
    región específica de la plataforma PixVerse, o ejecute
    `openclaw onboard --auth-choice pixverse-api-key` para elegir una en el
    asistente de configuración:

    ```json5
    {
      models: {
        providers: {
          pixverse: {
            region: "cn", // "international" or "cn"
            baseUrl: "https://app-api.pixverseai.cn/openapi/v2",
            models: [],
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="URL base personalizada">
    Establezca `models.providers.pixverse.baseUrl` únicamente cuando se enrute mediante un proxy compatible de confianza.
    `baseUrl` tiene prioridad sobre `region`.

    ```json5
    {
      models: {
        providers: {
          pixverse: {
            baseUrl: "https://app-api.pixverse.ai/openapi/v2",
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Consulta periódica de tareas">
    PixVerse devuelve un `video_id` en la solicitud de generación. OpenClaw consulta
    `/openapi/v2/video/result/{video_id}` cada 5 segundos hasta que la tarea
    se completa correctamente, falla o alcanza el tiempo de espera (valor predeterminado: 5 minutos; se puede sustituir mediante
    `agents.defaults.mediaModels.video.timeoutMs`).
  </Accordion>
</AccordionGroup>

## Contenido relacionado

<CardGroup cols={2}>
  <Card title="Generación de vídeo" href="/es/tools/video-generation" icon="video">
    Parámetros de la herramienta compartida, selección del proveedor y comportamiento asíncrono.
  </Card>
  <Card title="Referencia de configuración" href="/es/gateway/config-agents#agent-defaults" icon="gear">
    Configuración predeterminada del agente, incluido el modelo de generación de vídeo.
  </Card>
</CardGroup>
