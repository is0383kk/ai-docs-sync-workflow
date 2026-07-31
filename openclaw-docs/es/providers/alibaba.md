---
read_when:
    - Quieres usar la generación de vídeo de Alibaba Wan en OpenClaw
    - Se necesita configurar una clave de API de Model Studio o DashScope para generar vídeos
summary: Generación de vídeo con Wan de Alibaba Model Studio en OpenClaw
title: Alibaba Model Studio
x-i18n:
    generated_at: "2026-07-26T04:54:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cb74e2361500ccfbc5d3c4f2d08c3b62aacba8c79c704570952e2181abacf9fb
    source_path: providers/alibaba.md
    workflow: 16
---

El Plugin incluido `alibaba` registra un proveedor de generación de vídeo para los modelos Wan en Alibaba Model Studio (el nombre internacional de DashScope). Está habilitado de forma predeterminada; solo se necesita una clave de API.

| Propiedad          | Valor                                                                           |
| ------------------ | ------------------------------------------------------------------------------- |
| Id. del proveedor  | `alibaba`                                                              |
| Plugin             | incluido, `enabledByDefault: true`                                                    |
| Variables de entorno de autenticación | `MODELSTUDIO_API_KEY` → `DASHSCOPE_API_KEY` → `QWEN_API_KEY` (se usa la primera coincidencia) |
| Indicador de incorporación | `--auth-choice alibaba-model-studio-api-key`                                                      |
| Indicador directo de la CLI | `--alibaba-model-studio-api-key <key>`                                                     |
| Modelo predeterminado | `alibaba/wan2.6-t2v`                                                           |
| URL base predeterminada | `https://dashscope-intl.aliyuncs.com`                                                        |

## Primeros pasos

<Steps>
  <Step title="Establecer una clave de API">
    Almacene la clave para el proveedor `alibaba` mediante la incorporación:

    ```bash
    openclaw onboard --auth-choice alibaba-model-studio-api-key
    ```

    O pase la clave directamente:

    ```bash
    openclaw onboard --alibaba-model-studio-api-key <your-key>
    ```

    O exporte una de las variables de entorno aceptadas antes de iniciar el Gateway:

    ```bash
    export MODELSTUDIO_API_KEY=sk-...
    # o DASHSCOPE_API_KEY=...
    # o QWEN_API_KEY=...
    ```

  </Step>
  <Step title="Establecer un modelo de vídeo predeterminado">
    ```json5
    {
      agents: {
        defaults: {
          videoGenerationModel: {
            primary: "alibaba/wan2.6-t2v",
          },
        },
      },
    }
    ```
  </Step>
  <Step title="Verificar que el proveedor esté configurado">
    ```bash
    openclaw models list --provider alibaba
    ```

    La lista incluye los cinco modelos Wan incluidos. Si no se puede resolver `MODELSTUDIO_API_KEY`, `openclaw models status --json` informa de la credencial que falta en `auth.unusableProfiles`.

  </Step>
</Steps>

<Note>
  El Plugin de Alibaba y el [Plugin de Qwen](/es/providers/qwen) se autentican con DashScope y aceptan variables de entorno que se solapan. Use los identificadores de modelo `alibaba/...` para la interfaz dedicada de vídeo de Wan; use los identificadores `qwen/...` para el chat, las incrustaciones o la comprensión multimedia de Qwen.
</Note>

## Modelos Wan integrados

| Referencia del modelo      | Modo                              |
| -------------------------- | --------------------------------- |
| `alibaba/wan2.6-t2v`         | Texto a vídeo (predeterminado)    |
| `alibaba/wan2.6-i2v`         | Imagen a vídeo                    |
| `alibaba/wan2.6-r2v`         | Referencia a vídeo                |
| `alibaba/wan2.6-r2v-flash`         | Referencia a vídeo (rápido)       |
| `alibaba/wan2.7-r2v`         | Referencia a vídeo                |

## Capacidades y límites

Los tres modos comparten el mismo límite de cantidad y duración de vídeos por solicitud; solo difiere la estructura de entrada.

| Modo                | Máx. de vídeos de salida | Máx. de imágenes de entrada | Máx. de vídeos de entrada | Duración máx. | Controles compatibles                                      |
| ------------------- | ------------------------ | --------------------------- | ------------------------- | ------------- | ---------------------------------------------------------- |
| Texto a vídeo       | 1                        | n/d                         | n/d                       | 10 s          | `size`, `aspectRatio`, `resolution`, `audio`, `watermark` |
| Imagen a vídeo      | 1                        | 1                           | n/d                       | 10 s          | `size`, `aspectRatio`, `resolution`, `audio`, `watermark` |
| Referencia a vídeo  | 1                        | n/d                         | 4                         | 10 s          | `size`, `aspectRatio`, `resolution`, `audio`, `watermark` |

Una solicitud que omite `durationSeconds` obtiene el valor predeterminado aceptado de DashScope de **5 segundos**. Establezca `durationSeconds` explícitamente en la [herramienta de generación de vídeo](/es/tools/video-generation) para ampliarlo hasta 10 s.

<Warning>
  Las entradas de imágenes y vídeos de referencia deben ser URL `http(s)` remotas; los modos de referencia de DashScope rechazan las rutas de archivos locales. Cárguelos primero en un almacenamiento de objetos o use el flujo de la [herramienta multimedia](/es/tools/media-overview), que ya genera una URL pública.
</Warning>

## Configuración avanzada

<AccordionGroup>
  <Accordion title="Anular la URL base de DashScope">
    El proveedor utiliza de forma predeterminada el punto de conexión internacional de DashScope. Para usar el punto de conexión de la región de China:

    ```json5
    {
      models: {
        providers: {
          alibaba: {
            baseUrl: "https://dashscope.aliyuncs.com",
          },
        },
      },
    }
    ```

    El proveedor elimina las barras diagonales finales antes de construir las URL de las tareas AIGC.

  </Accordion>

  <Accordion title="Prioridad de las variables de entorno de autenticación">
    OpenClaw resuelve la clave de API de Alibaba a partir de las variables de entorno en este orden y toma el primer valor que no esté vacío:

    1. `MODELSTUDIO_API_KEY`
    2. `DASHSCOPE_API_KEY`
    3. `QWEN_API_KEY`

    Las entradas `auth.profiles` configuradas (establecidas mediante `openclaw models auth login`) prevalecen sobre la resolución de variables de entorno. Consulte [Perfiles de autenticación en las preguntas frecuentes sobre modelos](/es/help/faq-models#auth-profiles-what-they-are-and-how-to-manage-them) para obtener información sobre la rotación de perfiles, el tiempo de espera y los mecanismos de anulación.

  </Accordion>

  <Accordion title="Relación con el Plugin de Qwen">
    Ambos Plugins incluidos se comunican con DashScope y aceptan claves de API que se solapan. Use:

    - `alibaba/wan*.*` identificadores para el proveedor dedicado de vídeo de Wan documentado en esta página.
    - `qwen/*` identificadores para el chat, las incrustaciones y la comprensión multimedia de Qwen (consulte [Qwen](/es/providers/qwen)).

    Establecer `MODELSTUDIO_API_KEY` una vez autentica ambos Plugins, ya que la lista de variables de entorno de autenticación se solapa intencionadamente; no es necesario incorporar cada Plugin por separado.

  </Accordion>
</AccordionGroup>

## Contenido relacionado

<CardGroup cols={2}>
  <Card title="Generación de vídeo" href="/es/tools/video-generation" icon="video">
    Parámetros compartidos de la herramienta de vídeo y selección de proveedores.
  </Card>
  <Card title="Qwen" href="/es/providers/qwen" icon="microchip">
    Configuración del chat, las incrustaciones y la comprensión multimedia de Qwen con la misma autenticación de DashScope.
  </Card>
  <Card title="Referencia de configuración" href="/es/gateway/config-agents#agent-defaults" icon="gear">
    Valores predeterminados de los agentes y configuración de modelos.
  </Card>
  <Card title="Preguntas frecuentes sobre modelos" href="/es/help/faq-models" icon="circle-question">
    Perfiles de autenticación, cambio de modelos y resolución de errores de «no profile».
  </Card>
</CardGroup>
