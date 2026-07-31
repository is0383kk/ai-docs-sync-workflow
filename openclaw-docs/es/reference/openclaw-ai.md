---
read_when:
    - Quieres reutilizar los transportes de modelos de OpenClaw en otra aplicación
    - Está modificando packages/ai o los puertos del host de transporte de IA
    - Está revisando qué publica la versión de OpenClaw en npm además del paquete raíz
summary: 'El paquete npm @openclaw/ai: transportes de modelos reutilizables, entornos de ejecución aislados y puertos de políticas del host'
title: Paquete @openclaw/ai
x-i18n:
    generated_at: "2026-07-26T05:28:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 610057caae0a9bbf9f74074cda75fc40c0b9aa9d3441f8263151f08f1a3f35a8
    source_path: reference/openclaw-ai.md
    workflow: 16
---

`@openclaw/ai` es la forma de biblioteca publicable de la capa de ejecución
de modelos de OpenClaw: contratos neutrales respecto al proveedor para mensajes/herramientas/flujos, validación, diagnósticos,
flujos de eventos, un registro de runtime aislado y adaptadores de carga diferida para las ocho
familias de API integradas (Anthropic Messages, OpenAI Completions, OpenAI
Responses, Azure OpenAI Responses, ChatGPT/Codex Responses, Google Generative
AI, Google Vertex, Mistral Conversations).

Se publica junto con el paquete raíz `openclaw` en cada versión, fijado a
la misma versión, con su propio `npm-shrinkwrap.json` para que su árbol de
dependencias transitivas quede bloqueado durante la instalación. Instalar `openclaw` instala
automáticamente el `@openclaw/ai` correspondiente; quienes consumen la biblioteca pueden depender de él
directamente sin código de la aplicación OpenClaw.

## Inicio rápido

```js
import { createLlmRuntime } from "@openclaw/ai";
import { registerBuiltInApiProviders } from "@openclaw/ai/providers";

const runtime = createLlmRuntime();
registerBuiltInApiProviders(runtime.registry);

const stream = runtime.streamSimple(model, { messages }, { apiKey });
for await (const event of stream) {
  if (event.type === "text_delta") process.stdout.write(event.delta);
}
const result = await stream.result();
```

Hay una versión ejecutable en el repositorio, en `examples/ai-chat`.

## Contrato de diseño

- **Con ámbito de instancia de forma predeterminada.** Importar el paquete no registra nada
  globalmente. `createApiRegistry()` / `createLlmRuntime()` devuelven instancias
  aisladas; `registerBuiltInApiProviders(registry)` habilita los transportes
  integrados en un registro. Los módulos del SDK del proveedor se cargan de forma diferida la primera vez que se usan.
- **La política del host se inyecta, no se incluye.** La protección de fetch para solicitudes (por
  ejemplo, la política de SSRF), la ocultación de secretos en el texto de repetición de resultados de herramientas, los
  valores predeterminados de herramientas estrictas de OpenAI y el registro de diagnósticos son puertos `AiTransportHost`
  configurados con `configureAiTransportHost`. Los valores predeterminados de la biblioteca son inertes;
  OpenClaw instala sus implementaciones reales en su fachada de flujos.
- **Una única identidad de flujo de eventos.** `@openclaw/ai/event-stream` es el constructor
  canónico de `EventStream` compartido por el núcleo de OpenClaw, agent-core y los consumidores
  externos.
- **Las subrutas `internal/*` no forman parte de la API.** Existen para la propia
  aplicación OpenClaw y no ofrecen ninguna garantía de semver.
- Los identificadores de proveedores, las credenciales, los catálogos de modelos, los reintentos y la conmutación por error siguen siendo
  responsabilidad de la aplicación. OpenClaw añade esas capas alrededor de este paquete; quien
  consume la biblioteca proporciona directamente un objeto `Model` y las opciones.

## Exportaciones de subrutas

| Subruta          | Contenido                                                                       |
| ---------------- | ------------------------------------------------------------------------------ |
| `.`              | Contratos, `createApiRegistry`, `createLlmRuntime`, `configureAiTransportHost` |
| `./providers`    | `registerBuiltInApiProviders`, `resetApiProviders`                             |
| `./types`        | Tipos de modelos/mensajes/herramientas/flujos                                  |
| `./validation`   | Validación de argumentos de herramientas                                       |
| `./diagnostics`  | Contratos de diagnóstico                                                       |
| `./event-stream` | Implementación compartida de `EventStream`                                |
| `./internal/*`   | Uso interno de OpenClaw, sin garantía de semver                                |
