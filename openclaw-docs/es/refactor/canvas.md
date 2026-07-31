---
read_when:
    - Traslado de la propiedad del host, las herramientas, los comandos, la documentación o el protocolo de Canvas
    - Auditando si Canvas sigue siendo propiedad del núcleo
    - Preparación o revisión del PR del plugin experimental Canvas
summary: Lista de verificación de planificación y auditoría para trasladar Canvas fuera del núcleo y convertirlo en un plugin experimental incluido.
title: Refactorización del plugin Canvas
x-i18n:
    generated_at: "2026-07-26T04:52:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ead3f865ea80acb1e47f45a5ab07acf19a6470035c00c81006b2b1230bedd71e
    source_path: refactor/canvas.md
    workflow: 16
---

# Refactorización del plugin Canvas

Canvas es experimental y se usa poco. Debe tratarse como un plugin incluido, no como una función del núcleo. El núcleo puede conservar la infraestructura genérica del Gateway, Node, HTTP, autenticación, configuración y cliente nativo, pero el comportamiento específico de Canvas debe residir en `extensions/canvas`.

## Objetivo

Trasladar la responsabilidad de Canvas a `extensions/canvas` y preservar el comportamiento actual de nodos emparejados:

- la herramienta `canvas` orientada al agente la registra el plugin Canvas
- los comandos de Node de Canvas solo se permiten cuando los registra el plugin Canvas
- los archivos de host y código fuente de A2UI residen en el plugin Canvas
- la materialización de documentos de Canvas reside en el plugin Canvas
- la implementación del comando de la CLI reside en el plugin Canvas o delega mediante un barrel de runtime propiedad del plugin
- la documentación y el inventario de plugins describen Canvas como experimental y respaldado por un plugin

## Objetivos excluidos

- No se debe rediseñar la interfaz de usuario de Canvas de la aplicación nativa en esta refactorización.
- No se debe eliminar la compatibilidad del protocolo o cliente de Canvas con iOS, Android o macOS, salvo que una decisión de producto independiente determine que Canvas debe eliminarse.
- No se debe crear un framework amplio de servicios para plugins solo para Canvas, salvo que al menos otro plugin incluido necesite la misma interfaz.

## Estado actual de la rama

Completado:

- Se añadió el paquete del plugin incluido en `extensions/canvas`.
- Se añadió `extensions/canvas/openclaw.plugin.json`.
- Se trasladó la herramienta `canvas` del agente de `src/agents/tools/canvas-tool.ts` a `extensions/canvas/src/tool.ts`.
- Se eliminó del núcleo el registro de `createCanvasTool` en `src/agents/openclaw-tools.ts`.
- Se trasladó la implementación del host de Canvas de `src/canvas-host` a `extensions/canvas/src/host`.
- Se mantuvo `extensions/canvas/runtime-api.ts` como barrel de compatibilidad propiedad del plugin para pruebas, empaquetado y utilidades públicas externas de Canvas.
- Se trasladó la materialización de documentos de Canvas de `src/gateway/canvas-documents.ts` a `extensions/canvas/src/documents.ts`.
- Se trasladaron la implementación de la CLI de Canvas y las utilidades JSONL de A2UI a `extensions/canvas/src/cli.ts`.
- Se trasladaron la URL del host de Canvas y las utilidades de capacidades con ámbito a `extensions/canvas/src`.
- Se retiraron los valores predeterminados de los comandos de Node de Canvas de las listas codificadas en el núcleo y se trasladaron a `nodeInvokePolicies` del plugin.
- Se añadió la configuración del host de Canvas propiedad del plugin en `plugins.entries.canvas.config.host`.
- Se trasladó el servicio HTTP de Canvas y A2UI detrás del registro de rutas HTTP del plugin Canvas.
- Se añadió el despacho genérico de actualizaciones WebSocket de plugins para las rutas HTTP propiedad de plugins.
- Se sustituyeron la URL del host de Gateway específica de Canvas y la autenticación de capacidades de Node por una superficie genérica de plugins alojados y utilidades de capacidades de Node.
- Se añadieron resolutores de medios alojados propiedad del plugin para que las URL de documentos de Canvas se resuelvan mediante el plugin Canvas, en lugar de que el núcleo importe los componentes internos de documentos de Canvas.
- Se añadió `api.registerNodeCliFeature(...)` para que Canvas pueda declarar `openclaw nodes canvas` como una función de Node propiedad del plugin sin especificar manualmente la ruta del comando principal.
- Se eliminaron las importaciones de producción de `src/**` correspondientes a `extensions/canvas/runtime-api.js`.
- Se trasladó el código fuente del paquete A2UI de `apps/shared/OpenClawKit/Tools/CanvasA2UI` a `extensions/canvas/src/host/a2ui-app`.
- Se trasladó la implementación de compilación y copia de A2UI a `extensions/canvas/scripts`, y se sustituyó la integración de compilación raíz por hooks genéricos de recursos de plugins incluidos.
- Se eliminó el alias heredado de configuración de nivel superior `canvasHost` del runtime.
- Se mantuvo la migración de doctor de Canvas para que `openclaw doctor --fix` reescriba las configuraciones antiguas de `canvasHost` como `plugins.entries.canvas.config.host`.
- Se eliminó la compatibilidad del protocolo de Canvas para agentes antiguos tras el protocolo v4 del Gateway. Ahora los clientes nativos y los gateways utilizan únicamente `pluginSurfaceUrls.canvas` junto con `node.pluginSurface.refresh`; la ruta obsoleta formada por `canvasHostUrl`, `canvasCapability` y `node.canvas.capability.refresh` no es compatible intencionadamente en esta refactorización experimental.
- Se actualizó el inventario generado de plugins para incluir Canvas.
- Se añadió documentación de referencia del plugin en `docs/plugins/reference/canvas.md`.

Superficies conocidas de Canvas que aún pertenecen al núcleo:

- Los controladores de Canvas de la aplicación nativa en `apps/` todavía consumen intencionadamente la superficie del plugin Canvas
- los controladores del protocolo y cliente de Canvas de la aplicación nativa en `apps/`
- la salida del artefacto publicado todavía utiliza `dist/canvas-host/a2ui` para permitir la búsqueda retrocompatible durante el runtime, pero el paso de copia ahora pertenece al plugin

## Estructura objetivo

`extensions/canvas` debe ser responsable de:

- el manifiesto del plugin y los metadatos del paquete
- el registro de herramientas del agente
- la política de comandos de invocación de Node
- el host de Canvas y el runtime de A2UI
- el código fuente del paquete A2UI de Canvas y los scripts de compilación y copia de recursos
- la creación de documentos de Canvas y la resolución de recursos
- la implementación de la CLI de Canvas
- la página de documentación de Canvas y la entrada del inventario de plugins

El núcleo solo debe ser responsable de las interfaces genéricas:

- el descubrimiento y registro de plugins
- el registro genérico de herramientas de agentes
- el registro genérico de políticas de invocación de Node
- el despacho genérico de HTTP/autenticación del Gateway y actualizaciones WebSocket
- la resolución genérica de URL de superficies de plugins alojados
- el registro genérico de resolutores de medios alojados
- el transporte genérico de capacidades de Node
- la infraestructura genérica de configuración
- el descubrimiento genérico de hooks de recursos de plugins incluidos

Las aplicaciones nativas pueden conservar los controladores de comandos de Canvas como clientes del protocolo. No son las propietarias del runtime del plugin.

## Pasos de migración

1. Tratar `plugins.entries.canvas.config.host` como la superficie de configuración propiedad del plugin.
2. Actualizar la documentación para describir Canvas como un plugin incluido experimental.
3. Ejecutar pruebas específicas de Canvas, comprobaciones del inventario de plugins, comprobaciones de la API del SDK de plugins y las barreras de compilación y tipos afectadas por los límites del runtime.

## Lista de comprobación de auditoría

Antes de considerar completa la refactorización:

- `rg "src/canvas-host|../canvas-host"` no devuelve ninguna importación activa de código fuente.
- `rg "canvas-tool|createCanvasTool" src` no encuentra ninguna implementación de herramientas de Canvas propiedad del núcleo.
- `rg "canvas.present|canvas.snapshot|canvas.a2ui" src/gateway` no encuentra valores predeterminados de listas de permitidos codificados fuera de las pruebas genéricas de políticas de plugins.
- `rg "extensions/canvas/runtime-api" src --glob '!**/*.test.ts'` está vacío.
- `rg "canvas-documents" src` está vacío.
- `rg "registerNodesCanvasCommands|nodes-canvas" src` está vacío; el plugin Canvas registra `openclaw nodes canvas` mediante metadatos anidados de la CLI del plugin.
- `rg "createCanvasHostHandler|handleA2uiHttpRequest" src/gateway` no devuelve ninguna responsabilidad del runtime del Gateway.
- `rg "apps/shared/OpenClawKit/Tools/CanvasA2UI|canvas-a2ui-copy|extensions/canvas/src/host/a2ui" scripts .github package.json` solo encuentra wrappers de compatibilidad o rutas propiedad del plugin.
- `pnpm plugins:inventory:check` se ejecuta correctamente.
- `pnpm plugin-sdk:api:check` se ejecuta correctamente, o los registros generados del contrato de la API se actualizan y revisan intencionadamente.
- Las pruebas específicas de Canvas se ejecutan correctamente.
- Las pruebas de carriles modificados para las rutas del host de Canvas y A2UI se ejecutan correctamente.
- El cuerpo del PR indica explícitamente que Canvas es experimental y está respaldado por un plugin.

## Comandos de verificación

Utilice comprobaciones locales específicas durante la iteración:

```sh
pnpm test extensions/canvas/src/host/server.test.ts extensions/canvas/src/host/server.state-dir.test.ts extensions/canvas/src/host/file-resolver.test.ts
pnpm test src/gateway/server.plugin-node-capability-auth.test.ts src/gateway/server-import-boundary.test.ts
pnpm test extensions/canvas/src/config-migration.test.ts src/commands/doctor-legacy-config.migrations.test.ts
pnpm test test/scripts/changed-lanes.test.ts test/scripts/build-all.test.ts extensions/canvas/scripts/bundle-a2ui.test.ts test/scripts/bundled-plugin-assets.test.ts extensions/canvas/scripts/copy-a2ui.test.ts src/infra/run-node.test.ts
pnpm tsgo:extensions
pnpm plugins:inventory:check
pnpm plugin-sdk:api:check
```

Ejecute `pnpm build` antes del push si cambian el barrel del runtime, las importaciones diferidas, el empaquetado o las superficies publicadas del plugin.
