---
read_when:
    - Quieres realizar búsquedas web sin una clave de API
    - Quieres la API de búsqueda de pago de Parallel
    - Quiere extractos densos ordenados según su eficiencia como contexto para LLM.
summary: Búsqueda paralela -- extractos densos de fuentes web optimizados para LLM
title: Búsqueda paralela
x-i18n:
    generated_at: "2026-07-26T05:33:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: eff693f286015b287bbdacf44f11ff6f07f2f7d2605ef6f09259e7402b40515e
    source_path: tools/parallel-search.md
    workflow: 16
---

El plugin Parallel proporciona dos proveedores de [Parallel](https://parallel.ai/) `web_search`,
ambos devuelven fragmentos clasificados y optimizados para LLM de un índice web
creado para agentes de IA:

| Proveedor                  | id              | Autenticación                                                                                      |
| -------------------------- | --------------- | -------------------------------------------------------------------------------------------------- |
| Parallel Search (gratuito) | `parallel-free` | Ninguna: [Search MCP](https://docs.parallel.ai/integrations/mcp/search-mcp) gratuito de Parallel |
| Parallel Search            | `parallel`      | `PARALLEL_API_KEY`: API de búsqueda de pago, límites de frecuencia más altos y ajuste de objetivos |

Establezca `tools.web.search.provider` en `parallel-free` o `parallel` para seleccionar
uno explícitamente; ninguno se detecta automáticamente.

<Note>
  Los modelos directos de OpenAI Responses (`api: "openai-responses"`, proveedor
  `openai`, URL base de la API oficial) utilizan automáticamente la búsqueda web
  nativa alojada de OpenAI cuando `tools.web.search.provider` no está definido, está vacío, es `"auto"`
  o `"openai"`, por lo que omiten Parallel de forma predeterminada. Establezca
  `tools.web.search.provider` en `parallel-free` o `parallel` para dirigirlos
  a través de Parallel en su lugar. Consulte la [descripción general de la búsqueda web](/es/tools/web).
</Note>

## Instalar el plugin

```bash
openclaw plugins install @openclaw/parallel-plugin
openclaw gateway restart
```

## Clave de API (proveedor de pago)

`parallel-free` no necesita ninguna clave, pero aun así debe seleccionarse explícitamente. El proveedor
de pago `parallel` necesita una clave de API:

<Steps>
  <Step title="Crear una cuenta">
    Regístrese en [platform.parallel.ai](https://platform.parallel.ai) y
    genere una clave de API desde su panel.
  </Step>
  <Step title="Almacenar la clave">
    Establezca `PARALLEL_API_KEY` en el entorno del Gateway o configúrela mediante:

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
      parallel: {
        config: {
          webSearch: {
            apiKey: "par-...", // opcional si PARALLEL_API_KEY está definido
            baseUrl: "https://api.parallel.ai", // opcional; OpenClaw añade /v1/search
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        // "parallel-free" para el Search MCP gratuito o "parallel" para el
        // proveedor de pago respaldado por API que se muestra aquí.
        provider: "parallel",
      },
    },
  },
}
```

**Alternativa mediante variable de entorno:** establezca `PARALLEL_API_KEY` en el entorno
del Gateway. Para una instalación del Gateway, colóquela en `~/.openclaw/.env`.

## Sustitución de la URL base

Se aplica solo al proveedor de pago `parallel`; `parallel-free` siempre utiliza
`https://search.parallel.ai/mcp` e ignora esta configuración.

Establezca `plugins.entries.parallel.config.webSearch.baseUrl` para dirigir las solicitudes
de pago a través de un proxy compatible o un punto de conexión alternativo (por ejemplo,
Cloudflare AI Gateway). OpenClaw normaliza los hosts sin esquema anteponiendo
`https://` y añade `/v1/search` a menos que la ruta ya termine así. El
punto de conexión resuelto forma parte de la clave de caché de búsqueda, por lo que nunca
se comparten resultados de puntos de conexión diferentes.

## Parámetros de la herramienta

Ambos proveedores exponen la estructura de búsqueda nativa de Parallel para que el modelo complete un
objetivo en lenguaje natural junto con unas pocas consultas breves por palabras clave: la combinación
que Parallel [recomienda](https://docs.parallel.ai/search/best-practices) para obtener
los mejores resultados.

<ParamField path="objective" type="string" required>
Descripción en lenguaje natural de la pregunta o el objetivo subyacente (máximo de 5000
caracteres). Debe ser autosuficiente.
</ParamField>

<ParamField path="search_queries" type="string[]" required>
Consultas de búsqueda concisas por palabras clave, de 3 a 6 palabras cada una (de 1 a 5 entradas, máximo de 200
caracteres cada una). Proporcione de 2 a 3 consultas diversas para obtener los mejores resultados.
</ParamField>

<ParamField path="count" type="number">
Resultados que se devolverán (1-40).
</ParamField>

<ParamField path="session_id" type="string">
Identificador opcional de sesión de Parallel procedente de `sessionId` de un resultado anterior. Páselo en
las búsquedas posteriores de la misma tarea para que Parallel agrupe las llamadas relacionadas y
mejore los resultados posteriores. Máximo de 1000 caracteres en `parallel`; el Search MCP
gratuito `parallel-free` lo limita a 100. Un identificador que supere el límite se descarta
(en la versión de pago) o se genera uno nuevo (en la gratuita).
</ParamField>

<ParamField path="client_model" type="string">
Identificador opcional del modelo que realiza la llamada (p. ej., `claude-opus-4-7`,
`gpt-5.6-sol`), con un máximo de 100 caracteres. Permite que Parallel adapte la configuración predeterminada a las
capacidades del modelo. Pase el identificador exacto del modelo activo; no lo acorte a un
alias de familia.
</ParamField>

## Notas

- Parallel clasifica y comprime los resultados según su utilidad para el razonamiento de los LLM, no para que las personas
  accedan a ellos mediante clics; se deben esperar fragmentos densos por resultado en lugar del contenido
  de la página completa.
- Los fragmentos de resultados se devuelven como la matriz `excerpts` y también se concatenan en
  `description` para mantener la compatibilidad con el contrato genérico `web_search`.
- Ambos proveedores devuelven un `session_id`; OpenClaw lo expone como `sessionId` en
  la carga útil de la herramienta para que los llamadores puedan agrupar las búsquedas posteriores. Un
  identificador de sesión generado por Parallel (uno que el llamador no haya proporcionado) se excluye
  de la entrada de caché, ya que las tareas no relacionadas con consultas idénticas no deben
  heredarlo.
- `searchId`, `warnings` y `usage` de Parallel se transfieren cuando
  están presentes.
- OpenClaw siempre reenvía a Parallel un recuento de resultados resuelto como
  `advanced_settings.max_results` (`parallel`) o aplica `count`
  en el lado del cliente después de la respuesta de tamaño fijo de Parallel (`parallel-free`). El
  argumento `count` del llamador tiene prioridad, seguido de `tools.web.search.maxResults`; de lo contrario,
  se utiliza el valor predeterminado genérico `web_search` de OpenClaw (5); la API de Parallel
  utiliza 10 de forma predeterminada.
- Los resultados se almacenan en caché durante 15 minutos de forma predeterminada (`cacheTtlMinutes`).
- `parallel-free` genera un `session_id` nuevo en cada llamada mediante su protocolo de enlace MCP
  cuando el llamador no proporciona uno; `parallel` lo deja sin definir en ese
  caso.

## Relacionado

- [Descripción general de la búsqueda web](/es/tools/web): todos los proveedores y la detección automática
- [Búsqueda de Exa](/es/tools/exa-search): búsqueda neuronal con extracción de contenido
- [Perplexity Search](/es/tools/perplexity-search): resultados estructurados con filtrado por dominio
