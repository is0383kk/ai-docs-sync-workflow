---
read_when:
    - Se desea usar Exa para web_search
    - Necesita una EXA_API_KEY
    - Se necesita búsqueda neuronal o extracción de contenido
summary: 'Búsqueda de Exa AI: búsqueda neuronal y por palabras clave con extracción de contenido'
title: Búsqueda de Exa
x-i18n:
    generated_at: "2026-07-26T05:57:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3ddfd6fb471f92e705facf5a2d02361c1a343b9032fa8e0a7b135af634df65b7
    source_path: tools/exa-search.md
    workflow: 16
---

[Exa AI](https://exa.ai/) es un proveedor de `web_search` con modos de búsqueda neuronal, por palabras clave e
híbrida, además de extracción de contenido integrada (fragmentos destacados, texto y
resúmenes).

## Instalar el plugin

```bash
openclaw plugins install @openclaw/exa-plugin
openclaw gateway restart
```

## Obtener una clave de API

<Steps>
  <Step title="Crear una cuenta">
    Regístrese en [exa.ai](https://exa.ai/) y genere una clave de API desde el
    panel de control.
  </Step>
  <Step title="Guardar la clave">
    Establezca `EXA_API_KEY` en el entorno del Gateway o configúrela mediante:

    ```bash
    openclaw configure --section web
    ```

  </Step>
</Steps>

## Configuración

```json5
{
  plugins: {
    entries: {
      exa: {
        config: {
          webSearch: {
            apiKey: "exa-...", // opcional si EXA_API_KEY está establecida
            baseUrl: "https://api.exa.ai", // opcional; OpenClaw añade /search
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "exa",
      },
    },
  },
}
```

**Alternativa mediante variable de entorno:** establezca `EXA_API_KEY` en el entorno del Gateway. Para
una instalación del Gateway, colóquela en `~/.openclaw/.env`. Consulte
[Variables de entorno](/es/help/faq#env-vars-and-env-loading).

## Sustituir la URL base

Establezca `plugins.entries.exa.config.webSearch.baseUrl` para dirigir las solicitudes de búsqueda de Exa
a través de un proxy compatible o un endpoint alternativo. OpenClaw
normaliza los hosts sin esquema anteponiendo `https://` y añade `/search`, a menos que
la ruta ya termine así. El endpoint resuelto forma parte de la clave de la caché de
búsqueda, por lo que los resultados de distintos endpoints nunca se comparten.

## Parámetros de la herramienta

<ParamField path="query" type="string" required>
Consulta de búsqueda.
</ParamField>

<ParamField path="count" type="number" default="5">
Resultados que se devolverán (1-100, sujeto a los límites del tipo de búsqueda de Exa).
</ParamField>

<ParamField path="type" type="'auto' | 'neural' | 'fast' | 'deep' | 'deep-reasoning' | 'instant'">
Modo de búsqueda.
</ParamField>

<ParamField path="freshness" type="'day' | 'week' | 'month' | 'year'">
Filtro temporal. No se puede combinar con `date_after`/`date_before`.
</ParamField>

<ParamField path="date_after" type="string">
Resultados posteriores a esta fecha (`YYYY-MM-DD`).
</ParamField>

<ParamField path="date_before" type="string">
Resultados anteriores a esta fecha (`YYYY-MM-DD`).
</ParamField>

<ParamField path="contents" type="object">
Opciones de extracción de contenido (véase más adelante).
</ParamField>

### Extracción de contenido

Pase un objeto `contents` para controlar el contenido extraído en los resultados:

```javascript
await web_search({
  query: "explicación de la arquitectura de transformadores",
  type: "neural",
  contents: {
    text: true, // texto completo de la página
    highlights: { numSentences: 3 }, // frases clave
    summary: true, // resumen generado por IA
  },
});
```

| Opción de contenido | Tipo                                                                  | Descripción                     |
| ------------------- | --------------------------------------------------------------------- | ------------------------------- |
| `text`          | `boolean \| { maxCharacters }`                                        | Extraer el texto completo de la página |
| `highlights`    | `boolean \| { maxCharacters, query, numSentences, highlightsPerUrl }` | Extraer frases clave            |
| `summary`       | `boolean \| { query }`                                                | Resumen generado por IA         |

Si se omite `contents`, Exa utiliza de forma predeterminada `{ highlights: true }` para que los resultados
incluyan extractos de frases clave. Las descripciones de los resultados se obtienen primero de los fragmentos destacados,
después del resumen y, por último, del texto completo, según cuál esté disponible primero. Los resultados
también conservan los campos sin procesar `highlightScores` y `summary` de la respuesta de la API de Exa
cuando están disponibles.

### Modos de búsqueda

| Modo             | Descripción                                      |
| ---------------- | ------------------------------------------------ |
| `auto`           | Exa elige el mejor modo (predeterminado)         |
| `neural`         | Búsqueda semántica/basada en el significado      |
| `fast`           | Búsqueda rápida por palabras clave               |
| `deep`           | Búsqueda profunda y exhaustiva                   |
| `deep-reasoning` | Búsqueda profunda con razonamiento                |
| `instant`        | Resultados más rápidos                            |

## Notas

- `count` acepta hasta 100, sujeto a los límites del tipo de búsqueda de Exa.
- Los resultados se almacenan en caché durante 15 minutos de forma predeterminada. Configure los valores compartidos
  `tools.web.search.cacheTtlMinutes` (minutos) y
  `tools.web.search.timeoutSeconds` (30 s de forma predeterminada) para cambiar el almacenamiento en caché y
  el tiempo de espera de las solicitudes de todos los proveedores de `web_search`, incluido Exa.

## Contenido relacionado

- [Descripción general de la búsqueda web](/es/tools/web) -- todos los proveedores y detección automática
- [Brave Search](/es/tools/brave-search) -- resultados estructurados con filtros de país/idioma
- [Perplexity Search](/es/tools/perplexity-search) -- resultados estructurados con filtrado por dominio
