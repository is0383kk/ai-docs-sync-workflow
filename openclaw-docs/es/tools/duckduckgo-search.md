---
read_when:
    - Quieres un proveedor de búsqueda web que no requiera una clave de API
    - Quieres usar DuckDuckGo para web_search
    - Se desea un proveedor de búsqueda sin clave seleccionado explícitamente
summary: Búsqueda web de DuckDuckGo -- proveedor sin clave (experimental, basado en HTML)
title: Búsqueda de DuckDuckGo
x-i18n:
    generated_at: "2026-07-26T05:33:03Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 84e90532de276dcb3f73c67015dffe5f5a62be673e44a19053b2b1dfcb0986ac
    source_path: tools/duckduckgo-search.md
    workflow: 16
---

OpenClaw admite DuckDuckGo como proveedor `web_search` **sin clave**. No se requiere ninguna clave de API ni cuenta.

<Warning>
  DuckDuckGo es una integración **experimental y no oficial** que extrae datos de las páginas de búsqueda HTML sin JavaScript de DuckDuckGo; no es una API oficial. Es posible que se produzcan fallos ocasionales debido a páginas de comprobación contra bots o cambios en el HTML.
</Warning>

## Configuración

DuckDuckGo nunca se selecciona automáticamente, ya que la detección automática solo tiene en cuenta proveedores con credenciales utilizables. Se debe configurar explícitamente:

<Steps>
  <Step title="Configurar">
    ```bash
    openclaw configure --section web
    # Seleccione "duckduckgo" como proveedor
    ```
  </Step>
</Steps>

## Configuración

Configure el proveedor directamente en la configuración:

```json5
{
  tools: {
    web: {
      search: {
        provider: "duckduckgo",
      },
    },
  },
}
```

Configuración opcional del Plugin para la región y SafeSearch:

```json5
{
  plugins: {
    entries: {
      duckduckgo: {
        config: {
          webSearch: {
            region: "us-en", // Código de región de DuckDuckGo
            safeSearch: "moderate", // "strict", "moderate" u "off"
          },
        },
      },
    },
  },
}
```

## Parámetros de la herramienta

<ParamField path="query" type="string" required>
Consulta de búsqueda.
</ParamField>

<ParamField path="count" type="number" default="5">
Resultados que se devolverán (1-10).
</ParamField>

<ParamField path="region" type="string">
Código de región de DuckDuckGo (p. ej., `us-en`, `uk-en`, `de-de`).
</ParamField>

<ParamField path="safeSearch" type="'strict' | 'moderate' | 'off'" default="moderate">
Nivel de SafeSearch.
</ParamField>

Los parámetros de la herramienta `region` y `safeSearch` anulan los valores de configuración del Plugin indicados anteriormente para cada consulta.

## Notas

- **Sin clave de API**: funciona una vez que se selecciona DuckDuckGo como proveedor `web_search`.
- **Experimental**: extrae datos de las páginas de búsqueda HTML sin JavaScript de DuckDuckGo; no es una API ni un SDK oficial. Los resultados dependen de la estructura de la página, que puede cambiar sin previo aviso.
- **Riesgo de comprobaciones contra bots**: DuckDuckGo puede mostrar CAPTCHA o bloquear solicitudes cuando el uso es intensivo o automatizado.
- **Solo selección explícita**: la detección automática de OpenClaw solo tiene en cuenta proveedores con credenciales utilizables, por lo que un proveedor sin clave como DuckDuckGo nunca se elige automáticamente; se debe configurar `provider: "duckduckgo"`.
- **SafeSearch utiliza `moderate` de forma predeterminada** cuando no está configurado.

<Tip>
  Para entornos de producción, considere [Brave Search](/es/tools/brave-search) (con nivel gratuito disponible) u otro proveedor respaldado por una API.
</Tip>

## Contenido relacionado

- [Descripción general de la búsqueda web](/es/tools/web): todos los proveedores y la detección automática
- [Brave Search](/es/tools/brave-search): resultados estructurados con nivel gratuito
- [Exa Search](/es/tools/exa-search): búsqueda neuronal con extracción de contenido
