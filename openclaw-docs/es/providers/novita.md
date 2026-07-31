---
read_when:
    - Quiere ejecutar OpenClaw con modelos de NovitaAI
    - Necesita el id, la clave o el endpoint del proveedor Novita
summary: Usa la API compatible con OpenAI de NovitaAI con OpenClaw
title: NovitaAI
x-i18n:
    generated_at: "2026-07-26T04:51:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 83e0e43e68d85d73e790023858a49f971b683129dbbdf6092fbd8bba4d8da331
    source_path: providers/novita.md
    workflow: 16
---

NovitaAI es un proveedor de infraestructura de IA alojada con una API compatible con OpenAI.
Se distribuye como proveedor incluido de OpenClaw (sin instalar un Plugin por separado), por lo que
las credenciales siguen el flujo normal de autenticación de modelos y las referencias de modelos tienen el formato
`novita/deepseek/deepseek-v3-0324`.

## Configuración

Cree una clave de API en [novita.ai/settings/key-management](https://novita.ai/settings/key-management) y, a continuación, ejecute:

```bash
openclaw onboard --auth-choice novita-api-key
```

O establezca:

```bash
export NOVITA_API_KEY="<your-novita-api-key>" # pragma: allowlist secret
```

## Valores predeterminados

| Configuración       | Valor                              |
| ------------- | ---------------------------------- |
| Id. del proveedor   | `novita`                           |
| Alias       | `novita-ai`, `novitaai`            |
| URL base      | `https://api.novita.ai/openai/v1`  |
| Variable de entorno       | `NOVITA_API_KEY`                   |
| Modelo predeterminado | `novita/deepseek/deepseek-v3-0324` |

## Catálogo de modelos incluido

- `novita/moonshotai/kimi-k2.5`
- `novita/minimax/minimax-m2.7`
- `novita/zai-org/glm-5`
- `novita/deepseek/deepseek-v3-0324`
- `novita/deepseek/deepseek-r1-0528`
- `novita/qwen/qwen3-235b-a22b-fp8`

Este es un punto de partida, no un catálogo actualizado en tiempo real. La cuenta, la región o
la oferta actual de Novita pueden añadir, eliminar o restringir rutas. Compruébelo antes de
establecer un valor predeterminado a largo plazo:

```bash
openclaw models list --provider novita
```

## Cuándo elegir Novita

- Acceso alojado a modelos de pesos abiertos con una API compatible con OpenAI.
- Rutas de las familias DeepSeek, Kimi, MiniMax, GLM o Qwen mediante una única cuenta
  de proveedor.
- Otra ruta de respaldo alojada junto con DeepInfra, GMI, OpenRouter o las API directas
  de los proveedores.
- Alojamiento de modelos por parte del proveedor en lugar de mantener infraestructura de LM Studio, Ollama,
  SGLang o vLLM.

Elija un proveedor directo cuando necesite parámetros de solicitud
nativos del proveedor o contratos de soporte. Elija un proveedor local cuando el modelo deba
ejecutarse en su propio hardware o dentro del perímetro de su red.

## Solución de problemas

- `401`/`403`: verifique la clave en la página de gestión de claves de Novita y vuelva a ejecutar
  `openclaw onboard --auth-choice novita-api-key` si el perfil almacenado está
  desactualizado.
- Errores de modelo desconocido: use el `novita/<route-id>` exacto devuelto por
  `openclaw models list --provider novita`.
- Rutas lentas o fallidas: pruebe otra ruta de modelo de Novita o configure Novita como
  proveedor de respaldo para cargas de trabajo que puedan tolerar variaciones específicas
  del proveedor.

## Contenido relacionado

- [Proveedores de modelos](/es/concepts/model-providers)
- [Directorio de proveedores](/es/providers/index)
