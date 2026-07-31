---
read_when:
    - Publicación de Skills
    - Depuración de errores de publicación
summary: Formato de la carpeta de Skills, archivos obligatorios, artefactos complementarios y límites.
x-i18n:
    generated_at: "2026-07-26T04:33:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fdf16a589b8961ccd9181a53a9fa92a358952b9147d22eaf977f23e0b4b4d653
    source_path: clawhub/skill-format.md
    workflow: 16
---

# Formato de Skills

## En disco

Una Skill es una carpeta.

Obligatorio:

- `SKILL.md` (o `skill.md`; también se acepta el formato heredado `skills.md`)

Opcional:

- cualquier archivo normal de apoyo (consulte «Archivos de la Skill»)
- `.clawhubignore` (patrones que se omiten al publicar; formato heredado `.clawdhubignore`)
- `.gitignore` (también se respeta)

## Importación desde GitHub

El importador web de GitHub es más estricto que la publicación y sincronización locales. Solo detecta
archivos `SKILL.md` o archivos heredados `skills.md` en repositorios públicos que no sean forks y pertenezcan
a la cuenta de GitHub con la sesión iniciada. No importa repositorios privados, forks,
repositorios archivados o deshabilitados ni repositorios públicos de terceros.

Metadatos de instalación local (escritos por la CLI):

- `<skill>/.clawhub/origin.json` (formato heredado `.clawdhub`)

Estado de instalación del directorio de trabajo (escrito por la CLI):

- `<workdir>/.clawhub/lock.json` (formato heredado `.clawdhub`)

## `SKILL.md`

- Markdown con frontmatter YAML opcional.
- El servidor extrae los metadatos del frontmatter durante la publicación.
- `description` se utiliza como resumen de la Skill en la interfaz y las búsquedas.

Para que las Agent Skills sean portables, `name` debe coincidir con el directorio principal y utilizar
entre 1 y 64 letras minúsculas, números o guiones. ClawHub mantiene separados el slug enrutable y
el nombre para mostrar en el catálogo, por lo que los nombres existentes de otros clientes siguen
pudiéndose publicar y no se reescriben silenciosamente. Las listas del catálogo pueden acortar visualmente los nombres largos
sin cambiar el nombre almacenado.

## Metadatos del frontmatter

Los metadatos de la Skill se declaran en el frontmatter YAML situado al principio de `SKILL.md`. Esto indica al registro (y al análisis de seguridad) qué necesita la Skill para ejecutarse.

### Frontmatter básico

```yaml
---
name: my-skill
description: Resumen breve de lo que hace esta Skill.
version: 1.0.0
---
```

### Metadatos de ejecución (`metadata.openclaw`)

Declare los requisitos de ejecución de la Skill en `metadata.openclaw` (alias: `metadata.clawdbot`, `metadata.clawdis`).

```yaml
---
name: my-skill
description: Gestiona tareas mediante la API de Todoist.
metadata:
  openclaw:
    requires:
      env:
        - TODOIST_API_KEY
      bins:
        - curl
    primaryEnv: TODOIST_API_KEY
---
```

Utilice `requires.env` para las variables de entorno que deben estar presentes antes de que la Skill pueda ejecutarse. Utilice `envVars` cuando necesite metadatos para cada variable, incluidas variables opcionales con `required: false`.

### Referencia completa de campos

| Campo              | Tipo       | Descripción                                                                                                                                  |
| ------------------ | ---------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `requires.env`     | `string[]` | Variables de entorno obligatorias que espera la Skill.                                                                                           |
| `requires.bins`    | `string[]` | Binarios de la CLI que deben estar instalados en su totalidad.                                                                                                     |
| `requires.anyBins` | `string[]` | Binarios de la CLI de los que debe existir al menos uno.                                                                                                  |
| `requires.config`  | `string[]` | Rutas de archivos de configuración que lee la Skill.                                                                                                          |
| `primaryEnv`       | `string`   | Variable de entorno principal de credenciales de la Skill.                                                                                                  |
| `envVars`          | `array`    | Declaraciones de variables de entorno con `name`, `required` opcional y `description` opcional. Establezca `required: false` para las variables de entorno opcionales. |
| `always`           | `boolean`  | Si es `true`, la Skill está siempre activa (no se necesita una instalación explícita).                                                                              |
| `skillKey`         | `string`   | Sustituye la clave de invocación de la Skill.                                                                                                         |
| `emoji`            | `string`   | Emoji que se muestra para la Skill.                                                                                                                 |
| `homepage`         | `string`   | URL de la página principal o la documentación de la Skill.                                                                                                         |
| `os`               | `string[]` | Restricciones del sistema operativo (por ejemplo, `["macos"]`, `["linux"]`).                                                                                             |
| `install`          | `array`    | Especificaciones de instalación de dependencias (consulte más adelante).                                                                                                  |
| `nix`              | `object`   | Especificación del plugin de Nix (consulte el README).                                                                                                                |
| `config`           | `object`   | Especificación de configuración de Clawdbot (consulte el README).                                                                                                           |

### Especificaciones de instalación

Si la Skill necesita que se instalen dependencias, declárelas en la matriz `install`:

```yaml
metadata:
  openclaw:
    install:
      - kind: brew
        formula: jq
        bins: [jq]
      - kind: node
        package: typescript
        bins: [tsc]
```

Tipos de instalación compatibles: `brew`, `node`, `go`, `uv`.

### Variables de entorno opcionales

Declare las variables de entorno opcionales en `metadata.openclaw.envVars` y establezca `required: false`. No añada entradas opcionales a `requires.env`, porque `requires.env` significa que la Skill no puede ejecutarse sin ellas.

```yaml
metadata:
  openclaw:
    primaryEnv: TODOIST_API_KEY
    envVars:
      - name: TODOIST_API_KEY
        required: true
        description: Token de la API de Todoist utilizado para solicitudes autenticadas.
      - name: TODOIST_PROJECT_ID
        required: false
        description: ID de proyecto predeterminado opcional cuando no se especifica ninguno.
```

### Por qué es importante

El análisis de seguridad de ClawHub comprueba que lo declarado por la Skill coincida con lo que realmente hace. Si el código hace referencia a `TODOIST_API_KEY`, pero el frontmatter no lo declara en `requires.env`, `primaryEnv` o `envVars`, el análisis señalará una discrepancia en los metadatos. Mantener las declaraciones precisas ayuda a que la Skill supere la revisión y a que los usuarios entiendan qué están instalando.

### Ejemplo: frontmatter completo

```yaml
---
name: todoist-cli
description: Gestiona tareas, proyectos y etiquetas de Todoist desde la línea de comandos.
version: 1.2.0
metadata:
  openclaw:
    requires:
      env:
        - TODOIST_API_KEY
      bins:
        - curl
    primaryEnv: TODOIST_API_KEY
    envVars:
      - name: TODOIST_API_KEY
        required: true
        description: Token de la API de Todoist.
      - name: TODOIST_PROJECT_ID
        required: false
        description: ID de proyecto predeterminado opcional.
    emoji: "\u2705"
    homepage: https://github.com/example/todoist-cli
---
```

## Archivos de la Skill

La publicación acepta todos los archivos normales de la carpeta de la Skill, independientemente de su extensión. Se siguen aplicando los archivos ignorados,
las rutas ocultas, los enlaces simbólicos, los metadatos de macOS y los límites de tamaño del servidor.

- Los archivos de tamaño limitado que contienen UTF-8 válido pueden previsualizarse como texto sin formato con caracteres de escape y se incluyen
  en el análisis de texto limitado.
- Los demás archivos conservan sus bytes exactos y están disponibles para su descarga.
- Los escáneres de seguridad reciben el artefacto almacenado completo; la detección de texto afecta a la representación y
  al análisis, no constituye una lista de formatos permitidos para la carga.

Límites (del servidor):

- Tamaño total del paquete: 50MB.
- El texto de la incrustación incluye `SKILL.md` y hasta aproximadamente 40 archivos UTF-8 de tamaño limitado (límite aplicado en la medida de lo posible).

## Slugs

- Se derivan del nombre de la carpeta de forma predeterminada.
- Los ámbitos de los paquetes deben coincidir exactamente con el identificador del publicador de ClawHub. Los identificadores de publicador pueden utilizar letras minúsculas, números, guiones, puntos y guiones bajos; deben comenzar y terminar con una letra minúscula o un número.
- Los slugs de los paquetes deben estar en minúsculas y ser compatibles con npm; por ejemplo, `@example.tools/demo-plugin` o `demo-plugin`.

## Control de versiones y etiquetas

- Cada publicación crea una versión nueva (semver).
- Las etiquetas son punteros de cadena a una versión; `latest` se utiliza habitualmente.

## Licencia

- Todas las Skills publicadas en ClawHub se distribuyen bajo la licencia `MIT-0`.
- Cualquiera puede utilizar, modificar y redistribuir las Skills publicadas, incluso con fines comerciales.
- No se requiere atribución.
- No añada condiciones de licencia incompatibles en `SKILL.md`; ClawHub no admite anulaciones de licencia por Skill.

## Skills de pago

- ClawHub no admite Skills de pago, precios por Skill, muros de pago ni reparto de ingresos.
- No añada metadatos de precios a `SKILL.md`; no forman parte del formato de las Skills y no harán que una Skill publicada sea de pago.
- Si la Skill se integra con un servicio de terceros de pago, documente claramente el coste externo y la cuenta necesaria en las instrucciones de la Skill y las declaraciones de entorno (`requires.env` para variables obligatorias o `envVars` con `required: false` para variables opcionales).
