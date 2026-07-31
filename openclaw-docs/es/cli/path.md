---
read_when:
    - Quieres leer o escribir una hoja dentro de un archivo del espacio de trabajo desde la terminal
    - Estás creando scripts que interactúan con el estado del espacio de trabajo y quieres un esquema de direccionamiento estable e independiente del tipo
    - Estás depurando una ruta `oc://` (valida la sintaxis y comprueba a qué se resuelve)
summary: Referencia de la CLI para `openclaw path` (inspeccionar y editar archivos del espacio de trabajo mediante el esquema de direccionamiento `oc://`)
title: Ruta
x-i18n:
    generated_at: "2026-07-26T05:09:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7afe5bd1c3a5fca8dd22c7d807e390e751ae7e895c54bf0e10e2734f3889436c
    source_path: cli/path.md
    workflow: 16
---

# `openclaw path`

Acceso mediante shell al esquema de direccionamiento `oc://`: una sintaxis de rutas despachada por tipo
para inspeccionar y editar archivos direccionables del espacio de trabajo (markdown, jsonc,
jsonl, yaml/yml/lobster). Quienes alojan sus propias instancias, los autores de plugins y las extensiones de editores
la utilizan para leer, buscar o actualizar una ubicación específica sin tener que crear manualmente un
analizador para cada archivo.

`path` se proporciona mediante el plugin opcional incluido `oc-path`. Actívelo antes del
primer uso:

```bash
openclaw plugins enable oc-path
```

Los verbos de la CLI reflejan el modelo de direccionamiento:

- `resolve` es concreto y devuelve una sola coincidencia.
- `find` es el verbo de múltiples coincidencias para comodines, uniones, predicados y
  expansión posicional.
- `set` solo acepta rutas concretas o marcadores de inserción; los patrones comodín
  se rechazan antes de escribir.
- `validate` analiza una ruta sin acceder al sistema de archivos.
- `emit` procesa un archivo de ida y vuelta mediante análisis + emisión (diagnóstico de fidelidad de bytes).

## Por qué usarlo

El estado de OpenClaw se distribuye entre archivos markdown editados por personas, configuración JSONC
con comentarios, registros JSONL de solo anexado y archivos YAML de flujos de trabajo o especificaciones. Los scripts, hooks
y agentes suelen necesitar un pequeño valor de esos archivos: una clave de frontmatter, una
opción de un plugin, un campo de un registro, un paso de YAML o un elemento de viñeta bajo una
sección con nombre.

`openclaw path` proporciona a esos consumidores una dirección estable en lugar de un
grep, una expresión regular o un analizador específico para cada tipo de archivo. La misma ruta `oc://` se puede validar,
resolver, buscar, ejecutar en modo de prueba y escribir desde el terminal, lo que mantiene la automatización
específica revisable y reproducible. Conserva el resto del archivo, por lo que
escribir una hoja no altera sus comentarios, finales de línea ni el
formato cercano.

Úselo cuando lo que busca tenga una dirección lógica, pero la forma del archivo
varíe:

- Un hook lee una opción de JSONC con comentarios sin perderlos al
  volver a escribir el valor.
- Un script de mantenimiento encuentra todos los campos de evento coincidentes en un registro JSONL
  sin cargar el registro completo en un analizador personalizado.
- Un editor salta a una sección o elemento de viñeta de markdown mediante su slug y después representa
  la línea exacta que se resolvió.
- Un agente ejecuta en modo de prueba una pequeña edición del espacio de trabajo antes de aplicarla, con los
  bytes modificados visibles durante la revisión.

No use `openclaw path` para ediciones ordinarias de archivos completos, migraciones complejas de configuración ni
escrituras específicas de memoria; para ellas debe utilizarse el comando o plugin propietario. `path`
está pensado para operaciones pequeñas sobre archivos direccionables en las que un comando de terminal repetible
es preferible a otro analizador específico.

## Cómo se usa

Leer un valor de un archivo de configuración editado por personas:

```bash
openclaw path resolve 'oc://config.jsonc/plugins/github/enabled'
```

Previsualizar una escritura sin modificar el disco:

```bash
openclaw path set 'oc://config.jsonc/plugins/github/enabled' 'true' --dry-run
```

Buscar registros coincidentes en un registro JSONL de solo anexado:

```bash
openclaw path find 'oc://session.jsonl/[event=tool_call]/name'
```

Direccionar una instrucción de markdown por sección y elemento en lugar de por número
de línea:

```bash
openclaw path resolve 'oc://AGENTS.md/runtime-safety/openclaw-gateway'
```

Validar una ruta en la Pipeline de CI o en un script de comprobación previa antes de que este lea o
escriba:

```bash
openclaw path validate 'oc://AGENTS.md/tools/$last/risk'
```

Estos comandos están pensados para poder copiarse en scripts de shell. Use `--json` cuando
un consumidor necesite una salida estructurada y `--human` cuando una persona esté inspeccionando
el resultado.

## Cómo funciona

1. Analiza la dirección `oc://` en posiciones: archivo, sección, elemento, campo y una
   consulta de sesión opcional.
2. Elige el adaptador del tipo de archivo según la extensión de destino (`.md`, `.jsonc`,
   `.json`, `.jsonl`, `.ndjson`, `.yaml`, `.yml`, `.lobster`).
3. Resuelve las posiciones según la estructura de ese tipo de archivo: encabezados y
   elementos de markdown, claves de objeto e índices de matriz de JSONC, registros de línea de JSONL o
   nodos de mapa y secuencia de YAML.
4. Para `set`, emite los bytes editados mediante el mismo adaptador, de modo que las partes no modificadas
   del archivo conserven sus comentarios, finales de línea y formato cercano cuando
   el tipo lo admita.

`resolve` y `set` requieren un único destino concreto. `find` es el verbo
exploratorio: expande comodines, uniones, predicados y ordinales en las coincidencias
concretas que pueden inspeccionarse antes de elegir una para escribir.

## Subcomandos

| Subcomando              | Propósito                                                                     |
| ----------------------- | --------------------------------------------------------------------------- |
| `resolve <oc-path>`     | Imprimir la coincidencia concreta en la ruta (o «no encontrado»).                      |
| `find <pattern>`        | Enumerar las coincidencias de una ruta con comodín, unión o predicado.                  |
| `set <oc-path> <value>` | Escribir una hoja o un destino de inserción en una ruta concreta. Admite `--dry-run`.  |
| `validate <oc-path>`    | Solo analizar; imprimir el desglose estructural (archivo, sección, elemento y campo). |
| `emit <file>`           | Procesar un archivo de ida y vuelta mediante análisis + emisión (diagnóstico de fidelidad de bytes).          |

## Opciones globales

| Opción            | Se aplica a                       | Propósito                                                                  |
| --------------- | -------------------------------- | ------------------------------------------------------------------------ |
| `--cwd <dir>`   | `resolve`, `find`, `set`, `emit` | Resolver la posición del archivo respecto a este directorio (valor predeterminado: `process.cwd()`). |
| `--file <path>` | `resolve`, `find`, `set`, `emit` | Sobrescribir la ruta resuelta de la posición del archivo (acceso absoluto).                |
| `--json`        | todos                              | Forzar la salida JSON (valor predeterminado cuando stdout no es un TTY).                    |
| `--human`       | todos                              | Forzar la salida legible para personas (valor predeterminado cuando stdout es un TTY).                       |
| `--value-json`  | `set`                            | Analizar `<value>` como JSON para sustituir hojas de JSON/JSONC/JSONL.           |
| `--dry-run`     | `set`                            | Imprimir los bytes que se escribirían sin escribirlos.                   |
| `--diff`        | `set` (requiere `--dry-run`)     | Imprimir un diff unificado en lugar de los bytes completos.                          |

`validate` solo acepta `--json` / `--human`; no accede al sistema de archivos, por lo que
`--cwd` y `--file` no se aplican.

## Sintaxis de `oc://`

```text
oc://FILE/SECTION/ITEM/FIELD?session=SCOPE
```

Reglas de las posiciones: `field` requiere `item` y `item` requiere `section`. En
las cuatro posiciones:

- **Segmentos entre comillas** — `"a/b.c"` conserva los separadores `/` y `.`. El contenido es
  literal a nivel de bytes; `"` y `\` no se permiten dentro de las comillas. La posición del archivo
  también reconoce las comillas: `oc://"skills/email-drafter"/Tools/$last` trata
  `skills/email-drafter` como una única ruta de archivo.
- **Predicados** — `[k=v]`, `[k!=v]`, `[k<v]`, `[k<=v]`, `[k>v]`, `[k>=v]`.
  Los operadores numéricos requieren que ambos lados puedan convertirse en números finitos.
- **Uniones** — `{a,b,c}` coincide con cualquiera de las alternativas.
- **Comodines** — `*` (un solo subsegmento) y `**` (cero o más,
  recursivo). `find` los acepta; `resolve` y `set` los rechazan por ser
  ambiguos.
- **Posicional** — `$first` / `$last` se resuelven como el primer o último índice o
  clave declarada.
- **Ordinal** — `#N` para la enésima coincidencia según el orden del documento.
- **Marcadores de inserción** — `+`, `+key`, `+nnn` para la inserción por clave o índice
  (se usan con `set`).
- **Ámbito de sesión** — `?session=cron-daily`, etc. Es independiente del anidamiento de posiciones.
  Los valores de sesión son sin procesar y no se decodifican por porcentaje; no pueden contener caracteres de
  control ni delimitadores reservados de consulta (`?`, `&`, `%`).

Se rechazan los caracteres reservados (`?`, `&`, `%`) fuera de segmentos
entre comillas, de predicado o de unión. Los caracteres de control (U+0000-U+001F, U+007F) se
rechazan en cualquier lugar, incluido el valor de consulta `session`.

`formatOcPath(parseOcPath(path)) === path` está garantizado para las rutas canónicas.
Los parámetros de consulta no canónicos se ignoran, excepto el primer valor no vacío de
`session=`.

Límites estrictos: una ruta está limitada a 4096 bytes, a un máximo de 4 posiciones (archivo/sección/elemento/
campo), a un máximo de 64 subsegmentos separados por puntos por posición y a un máximo de 256 niveles
de recorrido anidado para rutas JSON profundas. Por separado, cualquier entrada de archivo JSONC/JSON
superior a 16 MiB se rechaza con un diagnóstico de análisis en lugar de analizarse, para
cualquier verbo que cargue ese archivo.

## Direccionamiento según el tipo de archivo

| Tipo          | Extensiones de archivo             | Modelo de direccionamiento                                                                                    |
| ------------- | --------------------------- | --------------------------------------------------------------------------------------------------- |
| Markdown      | `.md`                       | Secciones H2 por slug, elementos de viñeta por slug o `#N`, frontmatter mediante `[frontmatter]`.                 |
| JSONC/JSON    | `.jsonc`, `.json`           | Claves de objeto e índices de matriz; los puntos dividen subsegmentos anidados, salvo cuando están entre comillas.                        |
| JSONL         | `.jsonl`, `.ndjson`         | Direcciones de línea de nivel superior (`L1`, `L2`, `$first`, `$last`) y, después, descenso al estilo JSONC dentro de la línea. |
| YAML/.lobster | `.yaml`, `.yml`, `.lobster` | Claves de mapa e índices de secuencia; los comentarios y el estilo de flujo se gestionan mediante la API de documentos YAML.        |

`resolve` devuelve una coincidencia estructurada: `root`, `node`, `leaf` o
`insertion-point`, con un número de línea basado en 1. Los valores de hoja se presentan como
texto más un `leafType`, para que los autores de plugins puedan representar vistas previas sin
depender de la forma del AST de cada tipo.

## Contrato de mutación

`set` escribe un destino concreto:

- Los valores del frontmatter de Markdown y los campos de elemento `- key: value` son hojas de
  cadena. Las inserciones de Markdown anexan secciones, claves del frontmatter o elementos de
  sección y representan una estructura Markdown canónica para el archivo modificado. Los cuerpos
  de las secciones no se pueden escribir en su totalidad mediante `set`.
- Las escrituras de hojas JSONC convierten el valor de cadena al tipo de hoja existente
  (`string`, `number` finito, `true`/`false` o `null`). Use `--value-json`
  cuando un reemplazo de hoja JSONC/JSON/JSONL deba analizar `<value>` como JSON y
  pueda cambiar de estructura, como al sustituir una forma abreviada de referencia a secreto en
  forma de cadena por un objeto. Las inserciones de objetos y matrices JSONC analizan `<value>` como JSON y usan
  la ruta de edición `jsonc-parser` para las escrituras de hojas ordinarias, conservando los comentarios
  y el formato cercano.
- Las escrituras de hojas JSONL realizan la conversión como JSONC dentro de una línea. El reemplazo
  de líneas completas y la anexión analizan `<value>` como JSON. El JSONL representado conserva la convención
  predominante de finales de línea LF/CRLF del archivo (por voto mayoritario entre los
  saltos de línea del archivo, por lo que un archivo mayoritariamente CRLF se mantiene en CRLF incluso con algunos LF aislados).
- Las escrituras de hojas YAML convierten al tipo escalar existente (`string`, `number`
  finito, `true`/`false` o `null`). Las inserciones YAML usan la API de documentos
  del paquete incluido `yaml` para actualizar mapas y secuencias. Los documentos YAML
  malformados con errores del analizador se rechazan antes de la modificación con
  `parse-error`.

Use `--dry-run` antes de las escrituras visibles para el usuario cuando importen los bytes exactos. Las ediciones
JSONC y YAML modifican el documento existente (mediante `jsonc-parser` o la API de documentos
`yaml`), por lo que los bytes no afectados suelen conservarse; Markdown reconstruye el archivo
a partir de su estructura analizada en cada edición, lo que puede normalizar el formato
accidental fuera de la hoja modificada. Añada `--diff` cuando desee que la vista previa
sea un parche enfocado del antes y el después, en lugar del archivo representado completo.

## Ejemplos

```bash
# Validar una ruta (sin acceso al sistema de archivos)
openclaw path validate 'oc://AGENTS.md/Tools/$last/risk'

# Leer una hoja
openclaw path resolve 'oc://gateway.jsonc/version'

# Buscar con comodines
openclaw path find 'oc://session.jsonl/*/event' --file ./logs/session.jsonl

# Simular una escritura
openclaw path set 'oc://gateway.jsonc/version' '2.0' --dry-run

# Simular una escritura como diff unificado
openclaw path set 'oc://gateway.jsonc/version' '2.0' --dry-run --diff

# Aplicar la escritura
openclaw path set 'oc://gateway.jsonc/version' '2.0'

# Recorrido de ida y vuelta con fidelidad de bytes (diagnóstico)
openclaw path emit ./AGENTS.md
```

Más ejemplos de gramática:

```bash
# Entrecomillar claves que contengan / o .
openclaw path resolve 'oc://config.jsonc/agents.defaults.models/"anthropic/claude-opus-4-7"/alias'

# Las rutas JSON/JSONC profundas pueden usar segmentos con barras; se normalizan como subsegmentos separados por puntos
openclaw path set 'oc://openclaw.json/agents/list/0/tools/exec/security' 'allowlist' --dry-run

# Reemplazar una hoja JSONC por un objeto analizado
openclaw path set 'oc://openclaw.json/gateway/auth/token' '{"source":"file","provider":"secrets","id":"/test"}' --value-json --dry-run

# Buscar mediante predicado entre los hijos JSONC
openclaw path find 'oc://config.jsonc/plugins/[enabled=true]/id'

# Insertar en una matriz JSONC
openclaw path set 'oc://config.jsonc/items/+1' '{"id":"new","enabled":true}' --dry-run

# Insertar una clave de objeto JSONC
openclaw path set 'oc://config.jsonc/plugins/+github' '{"enabled":true}' --dry-run

# Anexar un evento JSONL
openclaw path set 'oc://session.jsonl/+' '{"event":"checkpoint","ok":true}' --file ./logs/session.jsonl

# Resolver la última línea de valor JSONL
openclaw path resolve 'oc://session.jsonl/$last/event' --file ./logs/session.jsonl

# Resolver un paso de un flujo de trabajo YAML
openclaw path resolve 'oc://workflow.yaml/steps/0/id'

# Actualizar un escalar YAML
openclaw path set 'oc://workflow.yaml/steps/$last/id' 'classify-renamed' --dry-run

# Referenciar el frontmatter de Markdown
openclaw path resolve 'oc://AGENTS.md/[frontmatter]/name'

# Insertar frontmatter de Markdown
openclaw path set 'oc://AGENTS.md/[frontmatter]/+description' 'Agent instructions' --dry-run

# Buscar campos de elementos de Markdown
openclaw path find 'oc://SKILL.md/Tools/*/send_email'

# Validar una ruta con ámbito de sesión
openclaw path validate 'oc://AGENTS.md/Tools/$last/risk?session=cron-daily'
```

## Recetas por tipo de archivo

Los mismos cinco verbos funcionan con todos los tipos; el esquema de direccionamiento
selecciona el comportamiento según la extensión del archivo.

### Markdown

```text
<!-- frontmatter.md -->
---
name: redactor
description: agente de redacción de correos electrónicos
tier: núcleo
---
## Herramientas
- gh: CLI de GitHub
- curl: cliente HTTP
- send_email: habilitado
```

```bash
$ openclaw path resolve 'oc://x.md/[frontmatter]/tier' --file frontmatter.md --human
hoja @ L4: "core" (cadena)

$ openclaw path resolve 'oc://x.md/tools/gh/gh' --file frontmatter.md --human
hoja @ L9: "CLI de GitHub" (cadena)

$ openclaw path find 'oc://x.md/tools/*' --file frontmatter.md --human
3 coincidencias para oc://x.md/tools/*:
  oc://x.md/tools/gh           →  nodo @ L9 [md-item]
  oc://x.md/tools/curl         →  nodo @ L10 [md-item]
  oc://x.md/tools/send-email   →  nodo @ L11 [md-item]
```

El predicado `[frontmatter]` referencia el bloque de frontmatter YAML; `tools`
coincide con el encabezado `## Tools` mediante su slug, y las hojas de elementos conservan su forma de slug
incluso cuando la fuente usa guiones bajos (`send_email` se convierte en `send-email`).

### JSONC

```text
// config.jsonc
{
  "plugins": {
    "github": {"enabled": true, "role": "vcs"},
    "slack":  {"enabled": false, "role": "chat"}
  }
}
```

```bash
$ openclaw path resolve 'oc://config.jsonc/plugins/github/enabled' --file config.jsonc --human
hoja @ L4: "true" (booleano)

$ openclaw path set 'oc://config.jsonc/plugins/slack/enabled' 'true' --file config.jsonc --dry-run
--dry-run: escribiría 142 bytes en /…/config.jsonc
{
  "plugins": {
    "github": {"enabled": true, "role": "vcs"},
    "slack":  {"enabled": true, "role": "chat"}
  }
}
```

Las ediciones JSONC pasan por `jsonc-parser`, por lo que los comentarios y los espacios en blanco sobreviven a
`set`. Ejecute primero con `--dry-run` para inspeccionar los bytes antes de confirmar.
Los archivos `.json` usan el mismo adaptador y la misma ruta de edición que `.jsonc`.

### JSONL

```text
{"event":"start","userId":"u1","ts":1}
{"event":"action","userId":"u1","ts":2}
{"event":"end","userId":"u1","ts":3}
```

```bash
$ openclaw path find 'oc://session.jsonl/[event=action]/userId' --file session.jsonl --human
1 coincidencia para oc://session.jsonl/[event=action]/userId:
  oc://session.jsonl/L2/userId  →  hoja @ L2: "u1" (cadena)

$ openclaw path resolve 'oc://session.jsonl/L2/ts' --file session.jsonl --human
hoja @ L2: "2" (número)
```

Cada línea es un registro. Referencie mediante predicado (`[event=action]`) cuando
no conozca el número de línea, o mediante el segmento canónico `LN` cuando sí lo conozca.
Los archivos `.ndjson` usan el mismo adaptador que `.jsonl`.

### YAML

```text
# workflow.yaml
name: inbox-triage
steps:
  - id: fetch
    command: gmail.search
  - id: classify
    command: openclaw.invoke
```

```bash
$ openclaw path resolve 'oc://workflow.yaml/steps/0/id' --file workflow.yaml --human
hoja @ L3: "fetch" (cadena)

$ openclaw path set 'oc://workflow.yaml/steps/$last/id' 'classify-renamed' --file workflow.yaml --dry-run
--dry-run: escribiría 99 bytes en /…/workflow.yaml
name: inbox-triage
steps:
  - id: fetch
    command: gmail.search
  - id: classify-renamed
    command: openclaw.invoke
```

YAML usa la API `Document` del paquete `yaml` en lugar de un
analizador creado manualmente, por lo que los recorridos ordinarios de análisis y emisión conservan los comentarios y la
estructura de creación, mientras que las rutas resueltas usan el mismo modelo de clave de mapa/índice de secuencia que
JSONC. El mismo adaptador gestiona los archivos `.yaml`, `.yml` y `.lobster`.

## Referencia de subcomandos

### `resolve <oc-path>`

Lee una sola hoja o nodo. Los comodines se rechazan; use `find` para ellos.
Finaliza con `0` si hay una coincidencia, `1` si no hay ninguna sin que se produzca un error, y `2` si hay un error de análisis o un
patrón rechazado.

```bash
openclaw path resolve 'oc://AGENTS.md/tools/gh/risk' --human
openclaw path resolve 'oc://gateway.jsonc/server/port' --json
```

### `find <pattern>`

Enumera todas las coincidencias de un patrón con comodín, predicado o unión. Finaliza con `0`
si hay al menos una coincidencia y con `1` si no hay ninguna. Los comodines en la posición del archivo se rechazan con
`OC_PATH_FILE_WILDCARD_UNSUPPORTED`; proporcione un archivo concreto (la
expansión mediante patrones en varios archivos es una función posterior).

```bash
openclaw path find 'oc://AGENTS.md/tools/**/risk'
openclaw path find 'oc://session.jsonl/[event=action]/userId'
openclaw path find 'oc://config.jsonc/plugins/{github,slack}/enabled'
```

### `set <oc-path> <value>`

Escribe una hoja. Combínelo con `--dry-run` para previsualizar los bytes que se
escribirían sin modificar el archivo. Añada `--diff` para obtener una vista previa en forma de diff unificado.
Finaliza con `0` tras una escritura correcta, con `1` si el sustrato la rechaza (por ejemplo,
si se activa una protección centinela) y con `2` si se producen errores de análisis.

```bash
openclaw path set 'oc://gateway.jsonc/version' '2.0' --dry-run
openclaw path set 'oc://gateway.jsonc/version' '2.0' --dry-run --diff
openclaw path set 'oc://gateway.jsonc/version' '2.0'
openclaw path set 'oc://AGENTS.md/Tools/+gh/risk' 'low'
```

El marcador de inserción `+key` crea el hijo indicado si aún no
existe; `+nnn` y `+` sin adornos sirven respectivamente para la inserción
indexada y la anexión.

### `validate <oc-path>`

Comprobación solo de análisis. Sin acceso al sistema de archivos. Resulta útil para confirmar que una
ruta de plantilla está bien formada antes de sustituir variables, o para obtener
el desglose estructural durante la depuración:

```bash
$ openclaw path validate 'oc://AGENTS.md/tools/gh' --human
válida: oc://AGENTS.md/tools/gh
  archivo:  AGENTS.md
  sección:  tools
  elemento: gh
```

Finaliza con `0` si es válida, con `1` si no lo es (con `code` y
`message` estructurados) y con `2` si hay errores en los argumentos.

### `emit <file>`

Procesa un archivo de ida y vuelta mediante el analizador y emisor correspondientes a su tipo. La salida debería
ser idéntica byte por byte a la entrada si el archivo es válido; cualquier diferencia indica un
error del analizador o la activación de un centinela. Resulta útil para depurar el comportamiento del sustrato con
entradas reales.

```bash
openclaw path emit ./AGENTS.md
openclaw path emit ./gateway.jsonc --json
```

## Códigos de salida

| Código | Significado                                                                    |
| ---- | -------------------------------------------------------------------------- |
| `0`  | Correcto. (`resolve` / `find`: al menos una coincidencia. `set`: escritura correcta.) |
| `1`  | Sin coincidencias, o el sustrato rechazó `set` (sin error a nivel del sistema).      |
| `2`  | Error de argumento o de análisis.                                                   |

## Modo de salida

`openclaw path` detecta si se usa una TTY: muestra una salida legible para personas en una terminal y JSON cuando
stdout se canaliza o redirige. `--json` y `--human` anulan la
detección automática.

## Notas

- `set` escribe bytes mediante la ruta de emisión del sustrato, que aplica
  automáticamente la protección del centinela de censura. Una hoja que contenga
  `__OPENCLAW_REDACTED__` (literalmente o como subcadena) se rechaza en el momento de la
  escritura.
- El análisis de JSONC y las ediciones de hojas utilizan la dependencia `jsonc-parser`
  local del plugin, por lo que los comentarios y el formato se conservan en las escrituras
  normales de hojas, en lugar de pasar por una ruta de análisis y renderizado posterior
  implementada manualmente.
- `path` no conoce el seguimiento ni la recuperación de la configuración válida
  más reciente (LKG); ese ciclo de vida se gestiona en otro lugar. Si un archivo que se edita
  mediante `path` también está sujeto al seguimiento de LKG, la siguiente lectura
  de la configuración decide si se promociona o se recupera; una edición con
  `path` debe tratarse igual que cualquier otra escritura directa en ese archivo.

## Relacionado

- [Referencia de la CLI](/es/cli)
