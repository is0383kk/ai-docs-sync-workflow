---
read_when:
    - Quiere ejecutar OpenClaw con modelos de GMI Cloud
    - Necesita el identificador, la clave o el endpoint del proveedor GMI.
summary: Usar la API compatible con OpenAI de GMI Cloud con OpenClaw
title: GMI Cloud
x-i18n:
    generated_at: "2026-07-26T05:26:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a21fd2a997f44e1f78d97a0fba24ca2bbc00dd193323da712d650ed4ba105355
    source_path: providers/gmi.md
    workflow: 16
---

GMI Cloud es una plataforma de inferencia alojada para modelos de vanguardia y de pesos abiertos
mediante una API compatible con OpenAI. En OpenClaw es un plugin de proveedor externo oficial:
instálelo una vez, almacene las credenciales mediante la autenticación normal de modelos y use
referencias de modelo como `gmi/google/gemini-3.1-flash-lite`.

Use GMI cuando desee una clave de API para varias familias de modelos alojados, incluidas
las rutas de Anthropic, DeepSeek, Google, Moonshot, OpenAI y Z.AI expuestas por el
catálogo de GMI. Funciona como proveedor secundario para la conmutación por error de modelos, para comparar
rutas alojadas entre proveedores o cuando GMI dispone de un modelo antes que su
proveedor principal. OpenClaw controla el id del proveedor, el perfil de autenticación, los alias,
la base inicial del catálogo de modelos y la URL base; GMI controla la disponibilidad activa de los modelos, la facturación,
los límites de frecuencia y cualquier política de enrutamiento del proveedor.

| Propiedad      | Valor                                    |
| -------------- | ---------------------------------------- |
| Id del proveedor | `gmi` (alias: `gmi-cloud`, `gmicloud`) |
| Paquete       | `@openclaw/gmi-provider`                 |
| Variable de entorno de autenticación | `GMI_API_KEY`                            |
| API           | Compatible con OpenAI (`openai-completions`) |
| URL base      | `https://api.gmi-serving.com/v1`         |
| Modelo predeterminado | `gmi/google/gemini-3.1-flash-lite`       |

## Configuración

Instale el plugin, reinicie el Gateway y, a continuación, cree una clave de API en GMI Cloud
(`https://www.gmicloud.ai/`):

```bash
openclaw plugins install @openclaw/gmi-provider
openclaw gateway restart
```

A continuación, ejecute:

```bash
openclaw onboard --auth-choice gmi-api-key
```

Las configuraciones no interactivas pueden pasar `--gmi-api-key <key>` o establecer:

```bash
export GMI_API_KEY="<your-gmi-api-key>" # pragma: allowlist secret
```

## Cuándo elegir GMI

- Desea un endpoint alojado compatible con OpenAI en lugar de un servidor de modelos local.
- Desea probar varias familias de modelos comerciales y de pesos abiertos mediante una sola
  cuenta de proveedor.
- Desea un proveedor de reserva con un enrutamiento ascendente diferente al de DeepInfra,
  OpenRouter, Together o las API directas de los proveedores.
- Necesita ids de modelos, precios o controles de cuenta específicos de GMI.

Elija en su lugar el proveedor directo del fabricante cuando necesite funciones nativas
que GMI no exponga mediante su ruta compatible con OpenAI. Elija un proveedor local,
como LM Studio, Ollama, SGLang o vLLM, cuando la ubicación de los datos o el control local
de la GPU sean más importantes que la comodidad del alojamiento.

## Modelos

El catálogo del plugin proporciona como base inicial los ids de rutas de GMI Cloud disponibles habitualmente:

| Referencia de modelo               | Entrada      | Contexto  | Salida máxima |
| ---------------------------------- | ------------ | --------- | ------------- |
| `gmi/anthropic/claude-sonnet-4.6`  | texto + imagen | 200,000   | 64,000     |
| `gmi/deepseek-ai/DeepSeek-V3.2`    | texto         | 163,840   | 65,536     |
| `gmi/google/gemini-3.1-flash-lite` | texto + imagen | 1,048,576 | 65,536     |
| `gmi/moonshotai/Kimi-K2.5`         | texto + imagen | 262,144   | 65,536     |
| `gmi/openai/gpt-5.4`               | texto + imagen | 400,000   | 128,000    |
| `gmi/zai-org/GLM-5.1-FP8`          | texto         | 202,752   | 65,536     |

El catálogo es una base inicial, no una promesa de que todas las cuentas puedan llamar a todos los modelos en
todo momento. Enumere lo que informa el proveedor configurado en su entorno:

```bash
openclaw models list --provider gmi
```

## Solución de problemas

- `401` o `403`: compruebe que `GMI_API_KEY` esté establecida para el proceso que ejecuta
  OpenClaw, o vuelva a ejecutar la incorporación para almacenar la clave en el perfil de autenticación del proveedor.
- Errores de modelo desconocido: confirme que el modelo exista en su cuenta de GMI y use la
  referencia `gmi/<route-id>` completa que muestra `openclaw models list --provider gmi`.
- Errores intermitentes del proveedor: pruebe otra ruta de GMI o configure GMI como
  proveedor de reserva en lugar de como único proveedor principal de modelos.

## Contenido relacionado

- [Proveedores de modelos](/es/concepts/model-providers)
- [Todos los proveedores](/es/providers/index)
