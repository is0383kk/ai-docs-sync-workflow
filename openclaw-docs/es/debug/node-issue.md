---
read_when:
    - Investigación de un fallo del cargador de tsx/esbuild que menciona la ausencia del auxiliar __name
summary: Fallo histórico de Node + tsx «__name is not a function» y su causa
title: Fallo de Node + tsx
x-i18n:
    generated_at: "2026-07-26T04:39:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 97d2f62d24860cee65753027ba84c14c8d4ffb910ee17bb0032cf0409c427589
    source_path: debug/node-issue.md
    workflow: 16
---

# Fallo de Node + tsx «\_\_name is not a function»

## Estado

Resuelto. Este fallo no se reproduce en la versión actual de `tsx` fijada en
`package.json` (`4.22.3`) ni en las versiones actuales de Node. Se conserva aquí por si una
actualización futura de `tsx`/esbuild lo reintroduce.

## Síntoma original

La ejecución de los scripts de desarrollo de OpenClaw mediante `tsx` fallaba al iniciarse con:

```text
[openclaw] No se pudo iniciar la CLI: TypeError: __name is not a function
    at createSubsystemLogger (src/logging/subsystem.ts)
    at <caller> (src/agents/auth-profiles/constants.ts)
```

Se omiten los números de línea; ambos archivos han cambiado desde el fallo original
y las líneas específicas ya no coinciden.

Esto apareció después de que los scripts de desarrollo cambiaran de Bun a `tsx` (`2871657e`,
2026-01-06) para que Bun fuera opcional. La ruta equivalente basada en Bun no fallaba.
Se observó originalmente en Node v25.3.0 en macOS; también se consideró probable que
afectara a otras plataformas que ejecutaran Node 25.

## Causa

`tsx` transforma TS/ESM mediante esbuild con `keepNames: true` codificado de forma fija en
sus opciones de transformación. Esa configuración hace que esbuild envuelva las declaraciones
de funciones y clases con nombre en una llamada a un asistente `__name` para que `fn.name` se conserve durante la minificación
y el empaquetado. El fallo significa que el asistente faltaba o quedaba oculto en el punto de
llamada de ese módulo en la combinación afectada de `tsx`/Node, por lo que `__name(...)`
generaba una excepción en lugar de devolver el valor envuelto.

## Comprobación actual de reproducción

```bash
node --version
pnpm install
node --import tsx src/entry.ts status
```

Reproducción mínima aislada (carga únicamente el módulo del seguimiento de pila original):

```bash
node --import tsx scripts/repro/tsx-name-repro.ts
```

Actualmente, ambos comandos finalizan correctamente. Si alguno vuelve a generar `__name is not a
function`, capture la versión exacta de Node, la versión de `tsx`
(`node_modules/tsx/package.json`) y el seguimiento de pila completo antes de notificarlo al proyecto original.

## Soluciones alternativas (si el fallo reaparece)

- Ejecute los scripts de desarrollo con Bun en lugar de `node --import tsx`.
- Ejecute `pnpm tsgo` para comprobar los tipos y, a continuación, ejecute la salida compilada en lugar del
  código fuente mediante `tsx`:

  ```bash
  pnpm tsgo
  node openclaw.mjs status
  ```

- Pruebe una versión diferente de `tsx` (`pnpm add -D tsx@<version>` es un cambio de dependencia
  y requiere aprobación según la política del repositorio) para determinar mediante bisección si la versión de esbuild
  que incluye ha reintroducido el error.
- Pruebe con una versión principal o secundaria diferente de Node para determinar si el fallo es específico
  de la versión.

## Referencias

- [https://esbuild.github.io/api/#keep-names](https://esbuild.github.io/api/#keep-names)
- [https://github.com/evanw/esbuild/issues/1031](https://github.com/evanw/esbuild/issues/1031)

## Relacionado

- [Instalación de Node.js](/es/install/node)
- [Solución de problemas del Gateway](/es/gateway/troubleshooting)
