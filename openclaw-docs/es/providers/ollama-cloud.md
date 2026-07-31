---
read_when:
    - Quieres usar modelos de Ollama alojados sin un servidor de Ollama local
    - Necesita el id, la clave o el endpoint del proveedor ollama-cloud
summary: Usa Ollama Cloud directamente con OpenClaw
title: Ollama Cloud
x-i18n:
    generated_at: "2026-07-26T04:56:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 966e5237e37134cef109979079db390e9844714001e921e7976dc8ca7f58bcc4
    source_path: providers/ollama-cloud.md
    workflow: 16
---

Ollama Cloud es la API de modelos alojada de Ollama. El proveedor `ollama-cloud` la invoca
directamente en `https://ollama.com` mediante la API nativa `/api/chat` de Ollama, sin
un servidor Ollama local ni una aplicación Ollama local con sesión iniciada en modo de nube. Use referencias
de modelos como `ollama-cloud/kimi-k2.6`.

OpenClaw registra `ollama-cloud` como su propio id de proveedor para que
las credenciales exclusivas de la nube, la detección en vivo del catálogo y la selección de modelos no se mezclen con
un host `ollama` local. Para Ollama local, el enrutamiento híbrido entre nube y entorno local,
los embeddings y los detalles de hosts personalizados, consulte [Ollama](/es/providers/ollama).

## Configuración

Cree una clave de API de Ollama Cloud en [ollama.com/settings/keys](https://ollama.com/settings/keys) y, a continuación, ejecute:

```bash
openclaw onboard --auth-choice ollama-cloud
```

O configure:

```bash
export OLLAMA_API_KEY="<your-ollama-cloud-api-key>" # pragma: allowlist secret
```

La incorporación no interactiva acepta la clave directamente:

```bash
openclaw onboard --auth-choice ollama-cloud --ollama-cloud-api-key "<key>"
```

La incorporación establece el modelo predeterminado en `ollama-cloud/kimi-k2.5:cloud`.

## Valores predeterminados

- Proveedor: `ollama-cloud`
- URL base: `https://ollama.com`
- Variable de entorno: `OLLAMA_API_KEY`
- Estilo de API: `/api/chat` nativa de Ollama
- Modelo predeterminado de incorporación: `ollama-cloud/kimi-k2.5:cloud`

## Cuándo elegir Ollama Cloud

- Desea modelos de Ollama alojados sin ejecutar `ollama serve` localmente.
- Desea la misma estructura de la API de chat nativa de Ollama que OpenClaw usa para Ollama
  local, pero dirigida a `https://ollama.com`.
- Desea una ruta sencilla en la nube para modelos que ya están en el catálogo
  alojado de Ollama.
- No necesita descargar modelos localmente, controlar la GPU local ni realizar inferencia solo en la LAN.

Use [Ollama](/es/providers/ollama) en su lugar cuando desee un enrutamiento solo local o
entre la nube y el entorno local mediante un host Ollama con sesión iniciada. Use un
proveedor compatible con OpenAI en su lugar cuando necesite la semántica de `/v1/chat/completions`
o funciones específicas del proveedor al estilo de OpenAI.

## Modelos

El proveedor requiere una clave de API; sin ella, permanece inactivo. Con una clave,
OpenClaw detecta en vivo los modelos de Ollama Cloud en el catálogo alojado:

```bash
openclaw models list --provider ollama-cloud
openclaw models set ollama-cloud/kimi-k2.6
```

Los ids alojados del catálogo en vivo incluyen `deepseek-v4-flash`, `glm-5`,
`gpt-oss:20b`, `kimi-k2.6` y `minimax-m2.7`. Cuando la detección en vivo no devuelve
nada, OpenClaw recurre a las filas incluidas `kimi-k2.5:cloud`,
`minimax-m2.7:cloud`, `glm-5.1:cloud` y `glm-5.2:cloud`.

Los ids de modelos son ids del catálogo de la nube, no nombres de descarga local. Si un nombre de modelo funciona en
un host Ollama local, pero no aparece en el catálogo alojado, use en su lugar el proveedor `ollama`
con ese host local.

## Prueba en vivo

Para las pruebas de humo de Ollama Cloud con clave de API, dirija la prueba en vivo de Ollama al
endpoint alojado y elija un modelo de su catálogo actual:

```bash
export OLLAMA_API_KEY="<your-ollama-cloud-api-key>" # pragma: allowlist secret

OPENCLAW_LIVE_TEST=1 \
OPENCLAW_LIVE_OLLAMA=1 \
OPENCLAW_LIVE_OLLAMA_BASE_URL=https://ollama.com \
OPENCLAW_LIVE_OLLAMA_MODEL=kimi-k2.6 \
pnpm test:live -- extensions/ollama/ollama.live.test.ts
```

La prueba de humo en la nube ejecuta texto, streaming nativo y búsqueda web; configure
`OPENCLAW_LIVE_OLLAMA_WEB_SEARCH=0` para omitir la búsqueda web. Omite los embeddings de forma
predeterminada para `https://ollama.com` porque es posible que las claves de API de Ollama Cloud no
autoricen `/api/embed`; fuércelos con `OPENCLAW_LIVE_OLLAMA_EMBEDDINGS=1`.

## Solución de problemas

- Errores `Ollama Cloud requires an API key` / `Set OLLAMA_API_KEY`: proporcione una
  clave de API real de la nube. El marcador local `ollama-local` es solo para hosts Ollama
  locales o privados.
- Errores de modelo desconocido: ejecute `openclaw models list --provider ollama-cloud` y
  copie exactamente el id del modelo alojado.
- Problemas con llamadas a herramientas o JSON sin procesar en hosts Ollama personalizados: compruebe si está
  usando accidentalmente una URL `/v1` compatible con OpenAI. Las rutas de Ollama deben usar
  la URL base nativa sin el sufijo `/v1`.

## Contenido relacionado

- [Ollama](/es/providers/ollama)
- [Proveedores de modelos](/es/concepts/model-providers)
- [Todos los proveedores](/es/providers/index)
