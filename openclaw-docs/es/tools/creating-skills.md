---
read_when:
    - Está creando una nueva skill personalizada
    - Necesita un flujo de trabajo inicial rápido para Skills basadas en SKILL.md
    - Quieres usar Skill Workshop para proponer una skill para que un agente la revise.
sidebarTitle: Creating skills
summary: Crea, prueba y publica Skills personalizadas del espacio de trabajo mediante SKILL.md para tus agentes de OpenClaw.
title: Creación de Skills
x-i18n:
    generated_at: "2026-07-26T05:00:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cba2aa863ebd083d4592e8a764dbdc2c30a0dd8aff49d273927e82df0069bc81
    source_path: tools/creating-skills.md
    workflow: 16
---

Skills enseña al agente cómo y cuándo usar herramientas. Cada skill es un directorio
que contiene un archivo `SKILL.md` con frontmatter YAML e instrucciones en Markdown.
OpenClaw carga las skills desde varias raíces según un [orden de precedencia](/es/tools/skills#loading-order) definido.

## Crear la primera skill

<Steps>
  <Step title="Crear el directorio de la skill">
    Las skills se encuentran en la carpeta `skills/` del espacio de trabajo:

    ```bash
    mkdir -p ~/.openclaw/workspace/skills/hello-world
    ```

    Las skills se pueden agrupar en subcarpetas para organizarlas; la skill sigue
    recibiendo el nombre del frontmatter `SKILL.md`, no de la ruta de la carpeta:

    ```bash
    mkdir -p ~/.openclaw/workspace/skills/personal/hello-world
    # el nombre de la skill sigue siendo "hello-world" y se invoca como /hello-world
    ```

  </Step>

  <Step title="Escribir SKILL.md">
    El frontmatter define los metadatos; el cuerpo proporciona instrucciones al agente.

    ```markdown
    ---
    name: hello-world
    description: Una skill sencilla que muestra un saludo.
    ---

    # Hola, mundo

    Cuando el usuario solicite un saludo, usa la herramienta `exec` para ejecutar:

    ```bash
    echo "¡Hola desde tu skill personalizada!"
    ```
    ```

    Reglas de nomenclatura:
    - Usa letras minúsculas, dígitos y guiones para `name`.
    - Mantén alineados el nombre del directorio y el valor `name` del frontmatter.
    - `description` se muestra al agente y en la detección de comandos con barra;
      debe ocupar una sola línea y tener menos de 160 caracteres.

  </Step>

  <Step title="Verificar que se haya cargado la skill">
    ```bash
    openclaw skills list
    ```

    De forma predeterminada, OpenClaw supervisa los archivos `SKILL.md` en las raíces de skills. Si el
    supervisor está deshabilitado o se continúa una sesión existente, se debe iniciar una
    nueva para que el agente reciba la lista actualizada:

    ```bash
    # Desde el chat: archiva la sesión actual e inicia una nueva
    /new

    # O reinicia el Gateway
    openclaw gateway restart
    ```

  </Step>

  <Step title="Probarla">
    ```bash
    openclaw agent --message "dame un saludo"
    ```

    También se puede abrir un chat y pedírselo directamente al agente. Usa `/skill hello-world` para
    invocarla explícitamente por su nombre.

  </Step>
</Steps>

## Referencia de SKILL.md

### Campos obligatorios

| Campo         | Descripción                                                     |
| ------------- | --------------------------------------------------------------- |
| `name`        | Identificador único que usa letras minúsculas, dígitos y guiones        |
| `description` | Descripción de una línea que se muestra al agente y en la salida de detección |

### Claves opcionales del frontmatter

| Campo                      | Valor predeterminado | Descripción                                                                      |
| -------------------------- | ------- | -------------------------------------------------------------------------------- |
| `user-invocable`           | `true`  | Expone la skill como comando con barra para el usuario                                         |
| `disable-model-invocation` | `false` | Mantiene la skill fuera del prompt del sistema del agente (aun así, se ejecuta mediante `/skill`)        |
| `command-dispatch`         | —       | Se establece en `tool` para dirigir el comando con barra directamente a una herramienta, omitiendo el modelo |
| `command-tool`             | —       | Nombre de la herramienta que se invoca cuando se establece `command-dispatch: tool`                         |
| `command-arg-mode`         | `raw`   | Para el despacho a herramientas, reenvía a la herramienta la cadena de argumentos sin procesar                      |
| `homepage`                 | —       | URL que se muestra como "Website" en la interfaz de Skills de macOS                                    |

Para consultar los campos de restricción (`requires.bins`, `requires.env`, etc.), véase
[Skills — Restricciones](/es/tools/skills#gating).

### Uso de `{baseDir}`

Haz referencia a archivos dentro del directorio de la skill sin codificar las rutas de forma fija; el
agente resuelve `{baseDir}` con respecto al directorio de la propia skill:

```markdown
Ejecuta el script auxiliar ubicado en `{baseDir}/scripts/run.sh`.
```

## Añadir activación condicional

Restringe la skill para que solo se cargue cuando sus dependencias estén disponibles:

```markdown
---
name: gemini-search
description: Busca mediante la CLI de Gemini.
metadata: { "openclaw": { "requires": { "bins": ["gemini"] }, "primaryEnv": "GEMINI_API_KEY" } }
---
```

<AccordionGroup>
  <Accordion title="Opciones de restricción">
    | Clave | Descripción |
    | --- | --- |
    | `requires.bins` | Todos los binarios deben existir en `PATH` |
    | `requires.anyBins` | Al menos un binario debe existir en `PATH` |
    | `requires.env` | Cada variable de entorno debe existir en el proceso o en la configuración |
    | `requires.config` | Cada ruta `openclaw.json` debe tener un valor verdadero |
    | `os` | Filtro de plataforma: `["darwin"]`, `["linux"]`, `["win32"]` |
    | `always` | Establece `true` para omitir todas las restricciones e incluir siempre la skill |

    Referencia completa: [Skills — Restricciones](/es/tools/skills#gating).

  </Accordion>
  <Accordion title="Entorno y claves de API">
    Vincula una clave de API a una entrada de skill en `openclaw.json`:

    ```json5
    {
      skills: {
        entries: {
          "gemini-search": {
            enabled: true,
            apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" },
          },
        },
      },
    }
    ```

    La clave se inyecta en el proceso host solo durante ese turno del agente.
    No llega al entorno aislado; véanse las
    [variables de entorno en entornos aislados](/es/tools/skills-config#sandboxed-skills-and-env-vars).

  </Accordion>
</AccordionGroup>

## Proponer mediante Skill Workshop

Para skills redactadas por el agente, o cuando se desea que el operador las revise antes de que una skill
entre en funcionamiento, utiliza propuestas de [Skill Workshop](/es/tools/skill-workshop) en lugar de escribir
`SKILL.md` directamente.

```bash
# Propón una skill completamente nueva
openclaw skills workshop propose-create \
  --name "hello-world" \
  --description "Una skill sencilla que muestra un saludo." \
  --proposal ./PROPOSAL.md

# Propón una actualización de una skill existente
openclaw skills workshop propose-update hello-world \
  --proposal ./PROPOSAL.md \
  --description "Skill de saludo actualizada"
```

Usa `--proposal-dir` cuando la propuesta incluya archivos auxiliares:

```bash
openclaw skills workshop propose-create \
  --name "hello-world" \
  --description "Una skill sencilla que muestra un saludo." \
  --proposal-dir ./hello-world-proposal/
```

El directorio debe contener `PROPOSAL.md` en su raíz. Los archivos auxiliares se colocan en
`assets/`, `examples/`, `references/`, `scripts/` o `templates/`.

Después de la revisión:

```bash
openclaw skills workshop inspect <proposal-id>
openclaw skills workshop apply <proposal-id>
```

Consulta [Skill Workshop](/es/tools/skill-workshop) para conocer el ciclo de vida completo de las propuestas.

## Publicar en ClawHub

<Steps>
  <Step title="Comprobar que SKILL.md esté completo">
    Asegúrate de que estén definidos `name`, `description` y todos los campos de restricción `metadata.openclaw`
    necesarios. Añade una URL `homepage` si hay una página del proyecto.
  </Step>
  <Step title="Instalar la CLI independiente de ClawHub e iniciar sesión">
    ```bash
    npm i -g clawhub
    clawhub login
    ```
  </Step>
  <Step title="Publicar">
    ```bash
    clawhub skill publish ./path/to/hello-world
    ```

    Añade `--version <version>` o `--owner <owner>` para reemplazar la versión
    inferida o publicar bajo un propietario específico. Consulta
    [ClawHub — Publicación](/es/clawhub/publishing) y
    [CLI de ClawHub](/es/clawhub/cli) para conocer el flujo completo, el ámbito de los propietarios y otros
    comandos de mantenimiento (`clawhub sync`, `clawhub skill rename`, ...).

  </Step>
</Steps>

## Prácticas recomendadas

<Tip>
  - **Sé conciso**: indica al modelo *qué* debe hacer, no cómo ser una IA.
  - **La seguridad ante todo**: si la skill usa `exec`, asegúrate de que los prompts no permitan
    la inyección arbitraria de comandos desde entradas que no sean de confianza.
  - **Prueba localmente**: usa `openclaw agent --message "..."` antes de compartir.
  - **Usa ClawHub**: explora las skills de la comunidad en [clawhub.ai](https://clawhub.ai)
    antes de crear una desde cero.
</Tip>

## Contenido relacionado

<CardGroup cols={2}>
  <Card title="Referencia de Skills" href="/es/tools/skills" icon="puzzle-piece">
    Orden de carga, restricciones, listas de permitidos y formato de SKILL.md.
  </Card>
  <Card title="Skill Workshop" href="/es/tools/skill-workshop" icon="flask">
    Cola de propuestas para skills redactadas por agentes.
  </Card>
  <Card title="Configuración de Skills" href="/es/tools/skills-config" icon="gear">
    Esquema de configuración completo de `skills.*`.
  </Card>
  <Card title="ClawHub" href="/es/clawhub" icon="cloud">
    Explora y publica skills en el registro público.
  </Card>
  <Card title="Creación de plugins" href="/es/plugins/building-plugins" icon="plug">
    Los plugins pueden incluir skills junto con las herramientas que documentan.
  </Card>
</CardGroup>
