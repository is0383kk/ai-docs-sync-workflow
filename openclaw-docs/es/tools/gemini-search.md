---
read_when:
    - Quieres usar Gemini para web_search
    - Necesita una GEMINI_API_KEY o models.providers.google.apiKey
    - Quieres fundamentación con Google Search
summary: Búsqueda web de Gemini con fundamentación en Google Search
title: Búsqueda de Gemini
x-i18n:
    generated_at: "2026-07-26T05:33:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4c7cb55fb185adfda01ab6b3c6434ab6e3ee31162733c752d4c81328bce9a6cd
    source_path: tools/gemini-search.md
    workflow: 16
---

OpenClaw admite modelos Gemini con
[fundamentación de Google Search](https://ai.google.dev/gemini-api/docs/grounding) integrada,
que devuelve respuestas sintetizadas por IA respaldadas por resultados de Google Search en vivo con
citas.

## Obtener una clave de API

<Steps>
  <Step title="Crear una clave">
    Ir a [Google AI Studio](https://aistudio.google.com/apikey) y crear una
    clave de API.
  </Step>
  <Step title="Almacenar la clave">
    Establecer `GEMINI_API_KEY` en el entorno del Gateway, reutilizar
    `models.providers.google.apiKey` o configurar una clave dedicada para búsquedas web mediante:

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
      google: {
        config: {
          webSearch: {
            apiKey: "AIza...", // opcional si se establece GEMINI_API_KEY o models.providers.google.apiKey
            baseUrl: "https://generativelanguage.googleapis.com/v1beta", // opcional; recurre a models.providers.google.baseUrl
            model: "gemini-2.5-flash", // predeterminado
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "gemini",
      },
    },
  },
}
```

**Precedencia de credenciales:** la búsqueda web de Gemini utiliza primero
`plugins.entries.google.config.webSearch.apiKey`, después `GEMINI_API_KEY`
y, por último, `models.providers.google.apiKey`. Para las URL base, la configuración dedicada
`plugins.entries.google.config.webSearch.baseUrl` tiene precedencia sobre
`models.providers.google.baseUrl`.

Para una instalación del Gateway, colocar las claves de entorno en `~/.openclaw/.env`.

## Cómo funciona

A diferencia de los proveedores de búsqueda tradicionales, que devuelven una lista de enlaces y fragmentos,
Gemini utiliza la fundamentación de Google Search para generar respuestas sintetizadas por IA con
citas integradas. Los resultados incluyen tanto la respuesta sintetizada como las URL de las
fuentes.

- Las URL de citas de la fundamentación de Gemini se resuelven automáticamente desde las URL de
  redirección de Google hasta las URL directas mediante una solicitud HEAD a través de la ruta de
  obtención protegida contra SSRF de OpenClaw (seguimiento de redirecciones, validación de http/https).
- La resolución de redirecciones utiliza valores predeterminados estrictos contra SSRF, por lo que se bloquean las redirecciones a
  destinos privados o internos.

## Parámetros compatibles

La búsqueda de Gemini admite `query`, `freshness`, `date_after` y `date_before`.

`count` se acepta para mantener la compatibilidad compartida con `web_search`, pero la fundamentación de Gemini
sigue devolviendo una única respuesta sintetizada con citas en lugar de una lista
de N resultados.

`freshness` acepta `day`, `week`, `month`, `year` y los atajos compartidos
`pd`, `pw`, `pm` y `py`. `day`/`pd` añade una instrucción de actualidad a la consulta de Gemini
en lugar de un intervalo estricto de 24 horas. `week`, `month`, `year` y los intervalos explícitos
`date_after`/`date_before` establecen
`timeRangeFilter` de la fundamentación de Google Search de Gemini. `country`, `language` y `domain_filter` no son compatibles.

## Selección del modelo

El modelo predeterminado es `gemini-2.5-flash` (rápido y rentable). Se puede utilizar cualquier modelo de
Gemini que admita fundamentación mediante
`plugins.entries.google.config.webSearch.model`.

## Anulaciones de la URL base

Establecer `plugins.entries.google.config.webSearch.baseUrl` cuando la búsqueda web de Gemini
deba enrutarse a través de un proxy del operador o un endpoint personalizado compatible con Gemini. Si
no se establece, la búsqueda web de Gemini reutiliza `models.providers.google.baseUrl`. Un valor simple de
`https://generativelanguage.googleapis.com` se normaliza como
`https://generativelanguage.googleapis.com/v1beta`; las rutas de proxy personalizadas se conservan
tal como se proporcionan después de eliminar las barras diagonales finales.

## Relacionado

- [Descripción general de la búsqueda web](/es/tools/web) -- todos los proveedores y detección automática
- [Brave Search](/es/tools/brave-search) -- resultados estructurados con fragmentos
- [Perplexity Search](/es/tools/perplexity-search) -- resultados estructurados + extracción de contenido
