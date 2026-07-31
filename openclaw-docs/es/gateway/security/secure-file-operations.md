---
read_when:
    - Cambiar el acceso a archivos, la extracción de archivos comprimidos, el almacenamiento del espacio de trabajo o los auxiliares del sistema de archivos de los plugins
summary: Cómo gestiona OpenClaw de forma segura el acceso a archivos locales y por qué el asistente opcional de Python fs-safe está desactivado de forma predeterminada
title: Operaciones seguras con archivos
x-i18n:
    generated_at: "2026-07-26T05:09:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5c8edf36ddbb8c8bc1edc52ecdf481affe5395d1779c679a40439167dfe70299
    source_path: gateway/security/secure-file-operations.md
    workflow: 16
---

OpenClaw usa [`@openclaw/fs-safe`](https://github.com/openclaw/fs-safe) para operaciones de archivos locales sensibles para la seguridad: lecturas y escrituras limitadas a una raíz, reemplazo atómico, extracción de archivos, espacios de trabajo temporales, estado JSON y manejo de archivos de secretos.

Es una **medida de protección de biblioteca** para código de confianza de OpenClaw que recibe nombres de rutas que no son de confianza, no un entorno aislado. Los permisos del sistema de archivos del host, los usuarios del sistema operativo, los contenedores y la política del agente o de las herramientas siguen definiendo el alcance real del impacto.

## Valor predeterminado: sin auxiliar de Python

OpenClaw establece el auxiliar POSIX de Python de fs-safe como **desactivado** de forma predeterminada:

- el Gateway no debe iniciar un proceso auxiliar persistente de Python salvo que un operador lo habilite;
- la mayoría de las instalaciones no necesitan el refuerzo adicional contra la modificación de directorios superiores;
- deshabilitar Python mantiene predecible el comportamiento en tiempo de ejecución en entornos de escritorio, Docker, CI y aplicaciones empaquetadas.

OpenClaw solo cambia el valor _predeterminado_. Una configuración explícita siempre tiene prioridad:

```bash
# Comportamiento predeterminado de OpenClaw: alternativas de fs-safe solo con Node.
OPENCLAW_FS_SAFE_PYTHON_MODE=off

# Habilita el auxiliar cuando esté disponible y utiliza la alternativa si no lo está.
OPENCLAW_FS_SAFE_PYTHON_MODE=auto

# Interrumpe la operación de forma segura si el auxiliar no puede iniciarse.
OPENCLAW_FS_SAFE_PYTHON_MODE=require

# Ruta explícita opcional al intérprete.
OPENCLAW_FS_SAFE_PYTHON=/usr/bin/python3
```

Los nombres genéricos de variables de entorno de fs-safe también funcionan: `FS_SAFE_PYTHON_MODE` y `FS_SAFE_PYTHON`.

Use `require` (no `auto`) cuando el auxiliar forme parte de su estrategia de seguridad; `auto` recurre silenciosamente al comportamiento basado solo en Node si el auxiliar no puede iniciarse.

## Qué sigue protegido sin Python

Con el auxiliar desactivado, OpenClaw sigue contando con las medidas de protección de fs-safe basadas solo en Node:

- rechaza escapes de rutas relativas (`..`), rutas absolutas y separadores de ruta donde solo se permiten nombres simples;
- resuelve las operaciones mediante un identificador de raíz de confianza en lugar de comprobaciones específicas de `path.resolve(...).startsWith(...)`;
- rechaza patrones de enlaces simbólicos y enlaces físicos en las API que exigen esa política;
- abre archivos con comprobaciones de identidad cuando la API devuelve o consume su contenido;
- escribe archivos de estado y configuración mediante un archivo temporal hermano y un cambio de nombre atómico;
- aplica límites de bytes a las lecturas y la extracción de archivos;
- aplica modos de archivo privados a los secretos y archivos de estado cuando la API lo exige.

Esto cubre el modelo de amenazas habitual de OpenClaw: código de confianza del Gateway que maneja entradas de rutas no confiables provenientes de modelos, plugins o canales dentro de un único límite de confianza del operador.

## Qué añade Python

En POSIX, el auxiliar opcional mantiene un proceso persistente de Python y usa operaciones del sistema de archivos relativas a descriptores de archivo para las modificaciones de directorios superiores: cambio de nombre, eliminación, creación de directorios, consulta/listado y algunas rutas de escritura.

Esto reduce las ventanas de condiciones de carrera con el mismo UID en las que otro proceso sustituye un directorio superior entre la validación y la modificación: una defensa en profundidad en hosts donde procesos locales no confiables pueden modificar los mismos directorios en los que opera OpenClaw.

Si su implementación presenta ese riesgo y se garantiza la disponibilidad de Python, establezca:

```bash
OPENCLAW_FS_SAFE_PYTHON_MODE=require
```

## Directrices para plugins y el núcleo

- El acceso a archivos destinado a plugins debe realizarse mediante los auxiliares `openclaw/plugin-sdk/*`, no mediante `fs` directamente, cuando una ruta provenga de un mensaje, la salida de un modelo, la configuración o la entrada de un plugin.
- El código del núcleo debe usar los envoltorios de fs-safe ubicados en `src/infra/*` para que la política de procesos de OpenClaw se aplique de manera coherente.
- La extracción de archivos debe usar los auxiliares de archivos de fs-safe con límites explícitos de tamaño, número de entradas, enlaces y destino.
- Los secretos deben usar los auxiliares de secretos de OpenClaw o los auxiliares de secretos y estado privado de fs-safe; no implemente manualmente comprobaciones de modos en torno a `fs.writeFile`.
- Para aislar usuarios locales hostiles, no dependa únicamente de fs-safe. Ejecute gateways separados bajo distintos usuarios o hosts del sistema operativo, o use un entorno aislado.

Relacionado: [Seguridad](/es/gateway/security), [Aislamiento](/es/gateway/sandboxing), [Aprobaciones de ejecución](/es/tools/exec-approvals), [Secretos](/es/gateway/secrets).
