---
read_when:
    - Quieres usar Ollama para `web_search`
    - Se desea un proveedor de web_search sin clave
    - Quiere usar Ollama Web Search alojado con OLLAMA_API_KEY
    - Necesita orientación para configurar Ollama Web Search
summary: Búsqueda web de Ollama mediante un host local de Ollama o la API alojada de Ollama
title: Búsqueda web de Ollama
x-i18n:
    generated_at: "2026-07-26T05:24:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: edbbd887841339ab4c0c62ab7682a22fe99434a788957a91989fce6942187e9a
    source_path: tools/ollama-search.md
    workflow: 16
---

OpenClaw admite **Ollama Web Search** como proveedor `web_search` incluido,
que devuelve títulos, URL y fragmentos de la API de búsqueda web de Ollama.

De forma predeterminada, Ollama local o autoalojado no necesita una clave de API; requiere un
host de Ollama accesible y `ollama signin`. La búsqueda alojada directa (sin Ollama local) necesita
`baseUrl: "https://ollama.com"` y un `OLLAMA_API_KEY` real.

## Configuración

<Steps>
  <Step title="Iniciar Ollama">
    Asegúrese de que Ollama esté instalado y en ejecución.
  </Step>
  <Step title="Iniciar sesión">
    ```bash
    ollama signin
    ```
  </Step>
  <Step title="Elegir Ollama Web Search">
    ```bash
    openclaw configure --section web
    ```

    Seleccione **Ollama Web Search** como proveedor.

  </Step>
</Steps>

Si ya utiliza Ollama para modelos, Ollama Web Search reutiliza el mismo
host configurado.

<Note>
  OpenClaw nunca selecciona automáticamente Ollama Web Search en lugar de un proveedor
  con credenciales de mayor prioridad; debe elegirlo explícitamente con
  `tools.web.search.provider: "ollama"`.
</Note>

## Configuración

```json5
{
  tools: {
    web: {
      search: {
        provider: "ollama",
      },
    },
  },
}
```

Sustitución opcional del host, limitada únicamente a la búsqueda web:

```json5
{
  plugins: {
    entries: {
      ollama: {
        config: {
          webSearch: {
            baseUrl: "http://ollama-host:11434",
          },
        },
      },
    },
  },
}
```

También puede reutilizar el host ya configurado para el proveedor de modelos de Ollama:

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "http://ollama-host:11434",
      },
    },
  },
}
```

`models.providers.ollama.baseUrl` es la clave canónica; el proveedor de búsqueda web
también acepta `baseURL` en esa ubicación para mantener la compatibilidad con ejemplos de
configuración al estilo del SDK de OpenAI. Si no se establece nada, OpenClaw usa de forma predeterminada
`http://127.0.0.1:11434`.

Ollama Web Search alojado directo (sin Ollama local):

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "https://ollama.com",
        apiKey: "OLLAMA_API_KEY",
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "ollama",
      },
    },
  },
}
```

## Autenticación y enrutamiento de solicitudes

- No existe ningún campo de clave de API específico para la búsqueda web; el proveedor reutiliza
  `models.providers.ollama.apiKey` (o la autenticación correspondiente del proveedor respaldada por variables de entorno)
  cuando el host configurado está protegido mediante autenticación.
- Orden de resolución del host: `plugins.entries.ollama.config.webSearch.baseUrl` →
  `models.providers.ollama.baseUrl` (o `baseURL`) → `http://127.0.0.1:11434`.
- Si el host resuelto es `https://ollama.com`, OpenClaw llama a
  `https://ollama.com/api/web_search` directamente y utiliza la clave de API como autenticación
  de portador.
- En caso contrario, OpenClaw llama primero al endpoint del proxy local
  `/api/experimental/web_search` (que firma y reenvía la solicitud a Ollama
  Cloud) y, después, recurre a `/api/web_search` en el mismo host. Si ambos fallan
  y se ha establecido `OLLAMA_API_KEY`, vuelve a intentarlo una vez con
  `https://ollama.com/api/web_search` y esa clave, sin enviarla
  al host local.
- OpenClaw muestra una advertencia durante la configuración si no se puede acceder a Ollama o no se ha iniciado sesión, pero
  no impide seleccionar el proveedor.

## Contenido relacionado

- [Descripción general de la búsqueda web](/es/tools/web) -- todos los proveedores y la detección automática
- [Ollama](/es/providers/ollama) -- configuración de modelos de Ollama y modos local/en la nube
