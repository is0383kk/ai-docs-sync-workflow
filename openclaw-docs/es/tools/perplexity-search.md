---
read_when:
    - Quieres usar Perplexity Search para búsquedas web
    - Necesita configurar PERPLEXITY_API_KEY u OPENROUTER_API_KEY
summary: Compatibilidad de la API de búsqueda de Perplexity y Sonar/OpenRouter con web_search
title: Búsqueda de Perplexity
x-i18n:
    generated_at: "2026-07-26T04:56:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a7ca97355110e70a05f1d57acab475dda8dec89393804df40c6e9be5e30780e8
    source_path: tools/perplexity-search.md
    workflow: 16
---

OpenClaw admite la API de búsqueda de Perplexity como proveedor de `web_search`. Devuelve resultados estructurados con los campos `title`, `url` y `snippet`.

Por compatibilidad, OpenClaw también admite configuraciones heredadas de Perplexity Sonar/OpenRouter. Si se usa `OPENROUTER_API_KEY`, una clave `sk-or-...` en `plugins.entries.perplexity.config.webSearch.apiKey`, o se establece `plugins.entries.perplexity.config.webSearch.baseUrl` / `model`, el proveedor cambia a la ruta de finalizaciones de chat y devuelve respuestas sintetizadas por IA con citas en lugar de resultados estructurados de la API de búsqueda.

## Instalar el plugin

Instale el plugin oficial y, a continuación, reinicie el Gateway:

```bash
openclaw plugins install @openclaw/perplexity-plugin
openclaw gateway restart
```

## Obtener una clave de API de Perplexity

1. Cree una cuenta de Perplexity en [perplexity.ai/settings/api](https://www.perplexity.ai/settings/api).
2. Genere una clave de API en el panel.
3. Guarde la clave en la configuración o establezca `PERPLEXITY_API_KEY` en el entorno del Gateway.

## Compatibilidad con OpenRouter

Si ya se usaba OpenRouter para Perplexity Sonar, conserve `provider: "perplexity"` y establezca `OPENROUTER_API_KEY` en el entorno del Gateway, o guarde una clave `sk-or-...` en `plugins.entries.perplexity.config.webSearch.apiKey`.

Controles de compatibilidad opcionales:

- `plugins.entries.perplexity.config.webSearch.baseUrl`
- `plugins.entries.perplexity.config.webSearch.model`

## Ejemplos de configuración

### API de búsqueda nativa de Perplexity

```json5
{
  plugins: {
    entries: {
      perplexity: {
        config: {
          webSearch: {
            apiKey: "pplx-...",
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "perplexity",
      },
    },
  },
}
```

### Compatibilidad con OpenRouter / Sonar

```json5
{
  plugins: {
    entries: {
      perplexity: {
        config: {
          webSearch: {
            apiKey: "<openrouter-api-key>",
            baseUrl: "https://openrouter.ai/api/v1",
            model: "perplexity/sonar-pro",
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "perplexity",
      },
    },
  },
}
```

## Dónde establecer la clave

**Mediante la configuración:** ejecute `openclaw configure --section web`. Esto guarda la clave en `~/.openclaw/openclaw.json`, bajo `plugins.entries.perplexity.config.webSearch.apiKey`. Ese campo también acepta objetos SecretRef.

**Mediante el entorno:** establezca `PERPLEXITY_API_KEY` o `OPENROUTER_API_KEY` en el entorno del proceso del Gateway. Para una instalación del Gateway, colóquelo en `~/.openclaw/.env` (o en el entorno del servicio). Consulte [Variables de entorno](/es/help/faq#env-vars-and-env-loading).

Si `provider: "perplexity"` está configurado y el SecretRef de la clave de Perplexity no se puede resolver y no existe una alternativa en el entorno, el inicio o la recarga fallan de inmediato.

## Parámetros de la herramienta

Estos parámetros se aplican a la ruta de la API de búsqueda nativa de Perplexity.

<ParamField path="query" type="string" required>
Consulta de búsqueda.
</ParamField>

<ParamField path="count" type="number" default="5">
Número de resultados que se devolverán (1-10).
</ParamField>

<ParamField path="country" type="string">
Código de país ISO de 2 letras (p. ej., `US`, `DE`).
</ParamField>

<ParamField path="language" type="string">
Código de idioma ISO 639-1 (p. ej., `en`, `de`, `fr`).
</ParamField>

<ParamField path="freshness" type="'day' | 'week' | 'month' | 'year'">
Filtro temporal: `day` corresponde a 24 horas.
</ParamField>

<ParamField path="date_after" type="string">
Solo resultados publicados después de esta fecha (`YYYY-MM-DD`).
</ParamField>

<ParamField path="date_before" type="string">
Solo resultados publicados antes de esta fecha (`YYYY-MM-DD`).
</ParamField>

<ParamField path="domain_filter" type="string[]">
Matriz de dominios permitidos o denegados (máximo 20).
</ParamField>

<ParamField path="max_tokens" type="number" default="25000">
Presupuesto total de contenido (máximo 1000000).
</ParamField>

<ParamField path="max_tokens_per_page" type="number" default="2048">
Límite de tokens por página.
</ParamField>

Para la ruta de compatibilidad heredada de Sonar/OpenRouter:

- Se aceptan `query`, `count` y `freshness`.
- `count` solo se admite por compatibilidad en esa ruta; la respuesta sigue siendo una única respuesta sintetizada con citas, en lugar de una lista de N resultados.
- Los filtros exclusivos de la API de búsqueda (`country`, `language`, `date_after`, `date_before`, `domain_filter`, `max_tokens`, `max_tokens_per_page`) devuelven errores explícitos.

**Ejemplos:**

```javascript
// Búsqueda específica por país e idioma
await web_search({
  query: "energía renovable",
  country: "DE",
  language: "de",
});

// Resultados recientes (última semana)
await web_search({
  query: "noticias sobre IA",
  freshness: "week",
});

// Búsqueda por intervalo de fechas
await web_search({
  query: "avances en IA",
  date_after: "2024-01-01",
  date_before: "2024-06-30",
});

// Filtrado de dominios (lista de permitidos)
await web_search({
  query: "investigación climática",
  domain_filter: ["nature.com", "science.org", ".edu"],
});

// Filtrado de dominios (lista de denegados: usar el prefijo -)
await web_search({
  query: "reseñas de productos",
  domain_filter: ["-reddit.com", "-pinterest.com"],
});

// Extracción de más contenido
await web_search({
  query: "investigación detallada sobre IA",
  max_tokens: 50000,
  max_tokens_per_page: 4096,
});
```

### Reglas del filtro de dominios

- Máximo de 20 dominios por filtro.
- No se pueden mezclar entradas de la lista de permitidos y de la lista de denegados en la misma solicitud.
- Use el prefijo `-` para las entradas de la lista de denegados (p. ej., `["-reddit.com"]`).

## Notas

- La API de búsqueda de Perplexity devuelve resultados estructurados de búsqueda web (`title`, `url`, `snippet`).
- OpenRouter, o un valor explícito de `plugins.entries.perplexity.config.webSearch.baseUrl` / `model`, vuelve a cambiar Perplexity a las finalizaciones de chat de Sonar por compatibilidad.
- La compatibilidad con Sonar/OpenRouter devuelve una única respuesta sintetizada con citas, no filas de resultados estructurados.
- Los resultados se almacenan en caché durante 15 minutos de forma predeterminada (configurable mediante `cacheTtlMinutes`).

## Contenido relacionado

<CardGroup cols={2}>
  <Card title="Descripción general de la búsqueda web" href="/es/tools/web" icon="globe">
    Todos los proveedores y las reglas de detección automática.
  </Card>
  <Card title="Búsqueda con Brave" href="/es/tools/brave-search" icon="shield">
    Resultados estructurados con filtros de país e idioma.
  </Card>
  <Card title="Búsqueda con Exa" href="/es/tools/exa-search" icon="magnifying-glass">
    Búsqueda neuronal con extracción de contenido.
  </Card>
  <Card title="Documentación de la API de búsqueda de Perplexity" href="https://docs.perplexity.ai/docs/search/quickstart" icon="arrow-up-right-from-square">
    Guía de inicio rápido y referencia oficiales de la API de búsqueda de Perplexity.
  </Card>
</CardGroup>
