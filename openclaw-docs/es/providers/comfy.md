---
read_when:
    - Quieres usar flujos de trabajo locales de ComfyUI con OpenClaw
    - Quieres usar Comfy Cloud con flujos de trabajo de imágenes, vídeo o música
    - Necesitas las claves de configuración del plugin comfy incluido.
summary: Configuración de generación de imágenes, vídeos y música mediante flujos de trabajo de ComfyUI en OpenClaw
title: ComfyUI
x-i18n:
    generated_at: "2026-07-26T04:48:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 74150d202a422de8e0f4b2b82d5d12bd42eb46991e8ef688832208e1a2ff7793
    source_path: providers/comfy.md
    workflow: 16
---

OpenClaw incluye un plugin `comfy` integrado para ejecuciones de ComfyUI basadas en flujos de trabajo. El
plugin se basa por completo en flujos de trabajo: OpenClaw no asigna controles genéricos de `size`,
`aspectRatio`, `resolution`, `durationSeconds` ni de tipo TTS a
su grafo.

| Propiedad             | Detalle                                                                           |
| --------------------- | --------------------------------------------------------------------------------- |
| Proveedor             | `comfy`                                                                |
| Modelo                | `comfy/workflow`                                                                |
| Herramientas compartidas | `image_generate`, `video_generate`, `music_generate`                     |
| Autenticación         | Ninguna para ComfyUI local; `COMFY_API_KEY` o `COMFY_CLOUD_API_KEY` para Comfy Cloud |
| API                   | ComfyUI `/prompt` / `/history` / `/view`; Comfy Cloud `/api/*` |

## Funcionalidades compatibles

- Generación y edición de imágenes a partir de un JSON de flujo de trabajo (la edición admite 1 imagen de referencia cargada)
- Generación de vídeo a partir de un JSON de flujo de trabajo, de texto a vídeo o de imagen a vídeo (1 imagen de referencia)
- Generación de música/audio mediante la herramienta compartida `music_generate`, con 1 imagen de referencia opcional
- Descarga de la salida desde un nodo configurado o desde todos los nodos de salida coincidentes cuando no se configura ninguno

## Primeros pasos

Elija entre ejecutar ComfyUI en su propio equipo o utilizar Comfy Cloud.

<Tabs>
  <Tab title="Local">
    **Opción ideal para:** ejecutar su propia instancia de ComfyUI en su equipo o LAN.

    <Steps>
      <Step title="Iniciar ComfyUI localmente">
        Asegúrese de que su instancia local de ComfyUI esté en ejecución (el valor predeterminado es `http://127.0.0.1:8188`).
      </Step>
      <Step title="Preparar el JSON del flujo de trabajo">
        Exporte o cree un archivo JSON de flujo de trabajo de ComfyUI. Anote los identificadores de nodo del nodo de entrada del prompt y del nodo de salida que desea que OpenClaw lea.
      </Step>
      <Step title="Configurar el proveedor">
        Establezca `mode: "local"` e indique el archivo del flujo de trabajo. Ejemplo mínimo de imagen:

        ```json5
        {
          plugins: {
            entries: {
              comfy: {
                config: {
                  mode: "local",
                  baseUrl: "http://127.0.0.1:8188",
                  image: {
                    workflowPath: "./workflows/flux-api.json",
                    promptNodeId: "6",
                    outputNodeId: "9",
                  },
                },
              },
            },
          },
        }
        ```
      </Step>
      <Step title="Establecer el modelo predeterminado">
        Configure OpenClaw para que utilice el modelo `comfy/workflow` en la capacidad que haya configurado:

        ```json5
        {
          agents: {
            defaults: {
              imageGenerationModel: {
                primary: "comfy/workflow",
              },
            },
          },
        }
        ```
      </Step>
      <Step title="Verificar">
        ```bash
        openclaw models list --provider comfy
        ```
      </Step>
    </Steps>

  </Tab>

  <Tab title="Comfy Cloud">
    **Opción ideal para:** ejecutar flujos de trabajo en Comfy Cloud sin administrar recursos de GPU locales.

    <Steps>
      <Step title="Obtener una clave de API">
        Regístrese en [comfy.org](https://comfy.org) y genere una clave de API desde el panel de su cuenta.
      </Step>
      <Step title="Establecer la clave de API">
        Proporcione su clave mediante cualquiera de estos métodos:

        ```bash
        # Indicador de incorporación
        openclaw onboard --comfy-api-key "your-key"

        # Variable de entorno (preferida para demonios)
        export COMFY_API_KEY="your-key"

        # Variable de entorno alternativa
        export COMFY_CLOUD_API_KEY="your-key"

        # O integrada en la configuración
        openclaw config set plugins.entries.comfy.config.apiKey "your-key"
        ```
      </Step>
      <Step title="Preparar el JSON del flujo de trabajo">
        Exporte o cree un archivo JSON de flujo de trabajo de ComfyUI. Anote los identificadores de nodo del nodo de entrada del prompt y del nodo de salida.
      </Step>
      <Step title="Configurar el proveedor">
        Establezca `mode: "cloud"` e indique el archivo del flujo de trabajo:

        ```json5
        {
          plugins: {
            entries: {
              comfy: {
                config: {
                  mode: "cloud",
                  image: {
                    workflowPath: "./workflows/flux-api.json",
                    promptNodeId: "6",
                    outputNodeId: "9",
                  },
                },
              },
            },
          },
        }
        ```

        <Tip>
        En el modo de nube, `baseUrl` tiene como valor predeterminado `https://cloud.comfy.org`. Establezca `baseUrl` únicamente para un endpoint de nube personalizado.
        </Tip>
      </Step>
      <Step title="Establecer el modelo predeterminado">
        ```json5
        {
          agents: {
            defaults: {
              imageGenerationModel: {
                primary: "comfy/workflow",
              },
            },
          },
        }
        ```
      </Step>
      <Step title="Verificar">
        ```bash
        openclaw models list --provider comfy
        ```
      </Step>
    </Steps>

  </Tab>
</Tabs>

## Configuración

Comfy admite ajustes de conexión compartidos de nivel superior, además de secciones de flujo de trabajo por capacidad (`image`, `video`, `music`):

```json5
{
  plugins: {
    entries: {
      comfy: {
        config: {
          mode: "local",
          baseUrl: "http://127.0.0.1:8188",
          image: {
            workflowPath: "./workflows/flux-api.json",
            promptNodeId: "6",
            outputNodeId: "9",
          },
          video: {
            workflowPath: "./workflows/video-api.json",
            promptNodeId: "12",
            outputNodeId: "21",
          },
          music: {
            workflowPath: "./workflows/music-api.json",
            promptNodeId: "3",
            outputNodeId: "18",
          },
        },
      },
    },
  },
}
```

### Claves compartidas

| Clave                 | Tipo                   | Descripción                                                                           |
| --------------------- | ---------------------- | ------------------------------------------------------------------------------------- |
| `mode`    | `"local"` o `"cloud"` | Modo de conexión. El valor predeterminado es `"local"`.            |
| `baseUrl`    | cadena                 | El valor predeterminado es `http://127.0.0.1:8188` para local o `https://cloud.comfy.org` para la nube. |
| `apiKey`    | cadena                 | Clave integrada opcional, alternativa a las variables de entorno `COMFY_API_KEY` / `COMFY_CLOUD_API_KEY`. |
| `allowPrivateNetwork`    | booleano               | Permite una `baseUrl` privada/LAN en modo de nube o un FQDN DNS privado local. |

<Note>
En el modo `local`, los literales de IP de bucle invertido/privadas y los nombres de servicio de una sola etiqueta, como `http://comfyui:8188`, funcionan sin `allowPrivateNetwork`. Los FQDN DNS privados que parecen públicos, como `https://comfy.local.example.com`, requieren `allowPrivateNetwork: true`. La confianza en orígenes privados permanece limitada al esquema, nombre de host y puerto configurados; las redirecciones locales no pueden abandonar el nombre de host configurado, mientras que las redirecciones de nube a CDN públicas se comprueban con la política SSRF predeterminada.
</Note>

### Claves por capacidad

Estas claves se aplican dentro de las secciones `image`, `video` o `music`:

| Clave                        | Obligatoria | Valor predeterminado | Descripción                                                               |
| ---------------------------- | ----------- | -------------------- | ------------------------------------------------------------------------- |
| `workflow` o `workflowPath` | Sí | --             | JSON de flujo de trabajo integrado o ruta al archivo JSON de flujo de trabajo de ComfyUI. |
| `promptNodeId`           | Sí          | --                   | Identificador del nodo que recibe el prompt de texto.                     |
| `promptInputName`           | No          | `"text"`   | Nombre de la entrada en el nodo del prompt.                               |
| `outputNodeId`           | No          | --                   | Identificador del nodo del que se lee la salida. Si se omite, se utilizan todos los nodos de salida coincidentes. |
| `pollIntervalMs`           | No          | `1500`   | Intervalo de sondeo en milisegundos para la finalización del trabajo.     |
| `timeoutMs`           | No          | `300000`   | Tiempo de espera en milisegundos para la ejecución del flujo de trabajo.  |

Las secciones `image` y `video` también admiten un nodo de entrada de imagen de referencia:

| Clave                 | Obligatoria                                  | Valor predeterminado | Descripción                                              |
| --------------------- | -------------------------------------------- | -------------------- | -------------------------------------------------------- |
| `inputImageNodeId`    | Sí (al proporcionar una imagen de referencia) | --                 | Identificador del nodo que recibe la imagen de referencia cargada. |
| `inputImageInputName`    | No                                           | `"image"` | Nombre de la entrada en el nodo de imagen.               |

`apiKey` acepta una cadena literal o un objeto de [referencia a secreto](/es/gateway/configuration-reference#secrets).

## Detalles del flujo de trabajo

<AccordionGroup>
  <Accordion title="Flujos de trabajo de imágenes">
    Establezca el modelo de imagen predeterminado en `comfy/workflow`:

    ```json5
    {
      agents: {
        defaults: {
          imageGenerationModel: {
            primary: "comfy/workflow",
          },
        },
      },
    }
    ```

    **Ejemplo de edición con una imagen de referencia:**

    Para habilitar la edición de imágenes con una imagen de referencia cargada, añada `inputImageNodeId` a la configuración de imagen:

    ```json5
    {
      plugins: {
        entries: {
          comfy: {
            config: {
              image: {
                workflowPath: "./workflows/edit-api.json",
                promptNodeId: "6",
                inputImageNodeId: "7",
                inputImageInputName: "image",
                outputNodeId: "9",
              },
            },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Flujos de trabajo de vídeo">
    Establezca el modelo de vídeo predeterminado en `comfy/workflow`:

    ```json5
    {
      agents: {
        defaults: {
          videoGenerationModel: {
            primary: "comfy/workflow",
          },
        },
      },
    }
    ```

    Los flujos de trabajo de vídeo de Comfy admiten la conversión de texto a vídeo y de imagen a vídeo mediante el grafo configurado.

    <Note>
    OpenClaw no pasa vídeos de entrada a los flujos de trabajo de Comfy. Solo se admiten como entradas prompts de texto e imágenes de referencia individuales.
    </Note>

  </Accordion>

  <Accordion title="Flujos de trabajo de música">
    El plugin integrado registra un proveedor de generación de música para salidas de audio o música definidas por el flujo de trabajo, disponible mediante la herramienta compartida `music_generate`. Acepta una imagen de referencia opcional (hasta 1):

    ```text
    /tool music_generate prompt="Bucle cálido de sintetizador ambiental con una suave textura de cinta"
    ```

    Utilice la sección de configuración `music` para indicar el JSON del flujo de trabajo de audio y el nodo de salida.

  </Accordion>

  <Accordion title="Compatibilidad con versiones anteriores">
    La configuración de imagen de nivel superior existente (sin la sección `image` anidada) sigue funcionando:

    ```json5
    {
      plugins: {
        entries: {
          comfy: {
            config: {
              workflowPath: "./workflows/flux-api.json",
              promptNodeId: "6",
              outputNodeId: "9",
            },
          },
        },
      },
    }
    ```

    OpenClaw trata esa estructura heredada como la configuración del flujo de trabajo de imágenes. No es necesario migrar inmediatamente, pero se recomiendan las secciones anidadas `image` / `video` / `music` para las nuevas configuraciones. Si solo se utiliza la generación de imágenes, la configuración plana heredada y la nueva sección anidada `image` son funcionalmente equivalentes.

  </Accordion>

  <Accordion title="Pruebas en vivo">
    El Plugin incluido dispone de cobertura de pruebas en vivo opcional:

    ```bash
    OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts
    ```

    La prueba en vivo omite los casos individuales de imágenes, vídeos o música, a menos que esté configurada la sección correspondiente del flujo de trabajo de Comfy.

  </Accordion>
</AccordionGroup>

## Contenido relacionado

<CardGroup cols={2}>
  <Card title="Generación de imágenes" href="/es/tools/image-generation" icon="image">
    Configuración y uso de la herramienta de generación de imágenes.
  </Card>
  <Card title="Generación de vídeos" href="/es/tools/video-generation" icon="video">
    Configuración y uso de la herramienta de generación de vídeos.
  </Card>
  <Card title="Generación de música" href="/es/tools/music-generation" icon="music">
    Configuración de la herramienta de generación de música y audio.
  </Card>
  <Card title="Directorio de proveedores" href="/es/providers/index" icon="layers">
    Descripción general de todos los proveedores y referencias de modelos.
  </Card>
  <Card title="Referencia de configuración" href="/es/gateway/config-agents#agent-defaults" icon="gear">
    Referencia completa de la configuración, incluidos los valores predeterminados de los agentes.
  </Card>
</CardGroup>
