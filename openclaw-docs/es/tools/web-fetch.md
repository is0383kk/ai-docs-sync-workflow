---
read_when:
    - Se desea obtener una URL y extraer contenido legible
    - Debe configurar web_fetch o su alternativa Firecrawl
    - Quieres comprender los límites y el almacenamiento en caché de web_fetch
sidebarTitle: Web Fetch
summary: 'herramienta web_fetch: obtención mediante HTTP con extracción de contenido legible'
title: Obtención web
x-i18n:
    generated_at: "2026-07-26T04:57:26Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ddf312245064672dcf489e8714740fa3e034827e16b33be8fb6a87db04f19ef8
    source_path: tools/web-fetch.md
    workflow: 16
---

`web_fetch` realiza una solicitud HTTP GET simple y extrae contenido legible (de HTML a
markdown o texto). **No** ejecuta JavaScript. Para sitios que dependen en gran medida de JS o
páginas protegidas mediante inicio de sesión, use en su lugar el [navegador web](/es/tools/browser).

## Inicio rápido

Está habilitado de forma predeterminada y no requiere configuración:

```javascript
await web_fetch({ url: "https://example.com/article" });
```

## Parámetros de la herramienta

<ParamField path="url" type="string" required>
URL que se obtendrá. Solo `http(s)`.
</ParamField>

<ParamField path="extractMode" type="'markdown' | 'text'" default="markdown">
Formato de salida tras extraer el contenido principal.
</ParamField>

<ParamField path="maxChars" type="number">
Trunca la salida a esta cantidad de caracteres. Se limita a `tools.web.fetch.maxCharsCap`.
</ParamField>

## Resultado

`web_fetch` devuelve un resultado estructurado cerrado con estos campos:

- Metadatos de la solicitud: `url`, `finalUrl`, `status`, `extractMode` y `extractor`
- Metadatos opcionales de la respuesta: `contentType`, `title` y `warning` (se omiten si no están presentes)
- Metadatos del contenido encapsulado: `externalContent`, `truncated`, `length`, `rawLength`,
  `fetchedAt`, `tookMs` y `text`
- `cached: true` opcional cuando se produce un acierto de caché
- `spill: { path, chars, truncated? }` opcional cuando se escribió contenido truncado
  en un archivo temporal privado; `truncated` solo está presente cuando ese archivo contiene
  contenido parcial de la fuente

`length` es la longitud encapsulada de `text`. `rawLength` es la longitud del contenido extraído
antes de encapsular el contenido externo.

## Funcionamiento

<Steps>
  <Step title="Obtención">
    Envía una solicitud HTTP GET con un User-Agent similar al de Chrome y el encabezado
    `Accept-Language`. Bloquea nombres de host privados o internos y vuelve a comprobar las redirecciones.
  </Step>
  <Step title="Extracción">
    Ejecuta Readability (extracción del contenido principal) sobre la respuesta HTML.
  </Step>
  <Step title="Alternativa (opcional)">
    Si Readability falla y hay disponible un proveedor de obtención, vuelve a intentarlo mediante
    dicho proveedor (por ejemplo, el modo de evasión de bots de Firecrawl).
  </Step>
  <Step title="Caché">
    Los resultados se almacenan en caché durante 15 minutos (configurable) para reducir las
    obtenciones repetidas de la misma URL.
  </Step>
</Steps>

## Actualizaciones de progreso

`web_fetch` emite una línea pública de progreso únicamente si la obtención sigue pendiente
después de cinco segundos:

```text
Obteniendo el contenido de la página...
```

Los aciertos rápidos de caché y las respuestas de red rápidas finalizan antes de que se active el temporizador, por lo que
nunca muestran una línea de progreso. Al cancelar la llamada se borra el temporizador. La
línea de progreso es únicamente un estado de la interfaz de usuario del canal y nunca contiene el contenido obtenido de la página.

## Configuración

```json5
{
  tools: {
    web: {
      fetch: {
        enabled: true, // valor predeterminado: true
        provider: "firecrawl", // opcional; omitir para la detección automática
        maxChars: 20000, // caracteres de salida predeterminados; limitados por maxCharsCap
        maxCharsCap: 20000, // límite estricto para el parámetro maxChars
        maxResponseBytes: 750000, // tamaño máximo de descarga antes del truncamiento (32000-10000000)
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
        maxRedirects: 3,
        useTrustedEnvProxy: false, // permitir que un proxy de entorno HTTP(S) de confianza resuelva el DNS
        readability: true, // usar la extracción de Readability
        userAgent: "Mozilla/5.0 ...", // sustituir el User-Agent
        ssrfPolicy: {
          allowRfc2544BenchmarkRange: true, // activación explícita para proxies de IP falsas de confianza que usen 198.18.0.0/15
          allowIpv6UniqueLocalRange: true, // activación explícita para proxies de IP falsas de confianza que usen fc00::/7
        },
      },
    },
  },
}
```

## Alternativa de Firecrawl

Si la extracción de Readability falla, `web_fetch` puede recurrir a
[Firecrawl](/es/tools/firecrawl) para evadir bots y mejorar la extracción:

```json5
{
  tools: {
    web: {
      fetch: {
        provider: "firecrawl", // opcional; omitir para la detección automática a partir de las credenciales disponibles
      },
    },
  },
  plugins: {
    entries: {
      firecrawl: {
        enabled: true,
        config: {
          webFetch: {
            // apiKey: "fc-...", // opcional; omitir para el acceso inicial sin clave
            baseUrl: "https://api.firecrawl.dev",
            onlyMainContent: true,
            maxAgeMs: 172800000, // duración de la caché (2 días)
            timeoutSeconds: 60,
          },
        },
      },
    },
  },
}
```

`plugins.entries.firecrawl.config.webFetch.apiKey` es opcional y admite objetos SecretRef.
La configuración heredada `tools.web.fetch.firecrawl.*` se migra automáticamente a
`plugins.entries.firecrawl.config.webFetch` mediante `openclaw doctor --fix`.

<Note>
  Si configura una SecretRef de clave de API de Firecrawl y esta no se puede resolver sin una
  alternativa de entorno `FIRECRAWL_API_KEY`, el inicio del Gateway falla de inmediato.
</Note>

<Note>
  Las sustituciones de `baseUrl` de Firecrawl están restringidas: el tráfico alojado utiliza
  `https://api.firecrawl.dev`; las sustituciones autoalojadas deben apuntar a puntos de conexión privados o
  internos, y `http://` solo se acepta para esos destinos privados.
</Note>

Comportamiento actual en tiempo de ejecución:

- `tools.web.fetch.provider` selecciona explícitamente el proveedor alternativo de obtención.
- Si se omite `provider`, OpenClaw detecta automáticamente el primer proveedor de obtención web
  listo a partir de las credenciales configuradas. Las llamadas `web_fetch` no aisladas pueden usar
  plugins instalados que declaren `contracts.webFetchProviders` y registren un
  proveedor correspondiente en tiempo de ejecución. Actualmente, el plugin oficial de Firecrawl proporciona esta
  alternativa.
- Las llamadas `web_fetch` aisladas permiten proveedores incluidos y proveedores instalados
  cuya procedencia oficial de npm o ClawHub esté verificada. Actualmente, esto permite el
  plugin oficial de Firecrawl; los plugins externos de obtención de terceros permanecen excluidos.
- Si Readability está deshabilitado, `web_fetch` pasa directamente al proveedor
  alternativo seleccionado. Si no hay ningún proveedor disponible, falla de forma cerrada.

## Proxy de entorno de confianza

Si la implementación requiere que `web_fetch` pase por un proxy HTTP(S)
de salida de confianza, establezca `tools.web.fetch.useTrustedEnvProxy: true`.

En este modo, OpenClaw sigue aplicando comprobaciones SSRF basadas en el nombre de host antes de enviar
la solicitud, pero permite que el proxy resuelva el DNS en lugar de realizar la
fijación de DNS local. Habilite esta opción únicamente cuando el proxy esté controlado por el operador y aplique
la política de salida después de la resolución de DNS.

<Note>
  Si no se ha configurado ninguna variable de entorno de proxy HTTP(S), o el host de destino está excluido mediante
  `NO_PROXY`, `web_fetch` recurre a la ruta estricta normal con fijación de DNS
  local.
</Note>

## Límites y seguridad

- `maxChars` se limita a `tools.web.fetch.maxCharsCap` (valor predeterminado: `20000`)
- El cuerpo de la respuesta se limita a `maxResponseBytes` (valor predeterminado: `750000`, limitado a
  32000-10000000) antes del análisis; las respuestas demasiado grandes se truncan con una advertencia
- Se bloquean los nombres de host privados o internos
- `tools.web.fetch.ssrfPolicy.allowRfc2544BenchmarkRange` y
  `tools.web.fetch.ssrfPolicy.allowIpv6UniqueLocalRange` son activaciones explícitas específicas
  para conjuntos de proxies de IP falsas de confianza; déjelas sin establecer a menos que el proxy sea propietario
  de esos intervalos sintéticos y aplique su propia política de destinos
- Las redirecciones se comprueban y se limitan mediante `maxRedirects` (valor predeterminado: `3`)
- `useTrustedEnvProxy` requiere una activación explícita y solo debe habilitarse para
  proxies controlados por el operador que sigan aplicando la política de salida después de la resolución de
  DNS
- `web_fetch` funciona con el mejor esfuerzo; algunos sitios necesitan el [navegador web](/es/tools/browser)

## Perfiles de herramientas

Si utiliza perfiles de herramientas o listas de permitidos, añada `web_fetch` o `group:web`:

```json5
{
  tools: {
    allow: ["web_fetch"],
    // o bien: allow: ["group:web"]  (incluye web_fetch, web_search y x_search)
  },
}
```

## Contenido relacionado

- [Búsqueda web](/es/tools/web): busca en la web con varios proveedores
- [Navegador web](/es/tools/browser): automatización completa del navegador para sitios que dependen en gran medida de JS
- [Firecrawl](/es/tools/firecrawl): herramientas de búsqueda y extracción de Firecrawl
