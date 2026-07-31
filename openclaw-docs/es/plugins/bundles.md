---
read_when:
    - Quiere instalar un paquete compatible con Codex, Claude o Cursor
    - Es necesario comprender cómo OpenClaw asigna el contenido de los paquetes a funciones nativas
    - Está depurando la detección de paquetes o funcionalidades faltantes
summary: Instalar y usar paquetes de Codex, Claude y Cursor como plugins de OpenClaw
title: Paquetes de Plugins
x-i18n:
    generated_at: "2026-07-26T05:12:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d44006866238f53ee2e3e8126cc4f7ed6f7413534257775f7904c9b877778c59
    source_path: plugins/bundles.md
    workflow: 16
---

OpenClaw puede instalar plugins de tres ecosistemas externos: **Codex**, **Claude**
y **Cursor**. Estos se denominan **paquetes**: conjuntos de contenido y metadatos que
OpenClaw asigna a funciones nativas como Skills, hooks y herramientas MCP.

<Info>
  Los paquetes **no** son lo mismo que los plugins nativos de OpenClaw. Los plugins nativos se ejecutan
  dentro del proceso y pueden registrar cualquier capacidad. Los paquetes son conjuntos de contenido con
  una asignación selectiva de funciones y un límite de confianza más restringido.
</Info>

## Por qué existen los paquetes

Muchos plugins útiles se publican en formato de Codex, Claude o Cursor. En lugar
de exigir que sus autores los reescriban como plugins nativos de OpenClaw, OpenClaw
detecta estos formatos y asigna el contenido compatible al conjunto de funciones
nativas. Se puede instalar un paquete de comandos de Claude o un paquete de Skills de Codex y usarlo
de inmediato.

## Instalar un paquete

<Steps>
  <Step title="Instalar desde un directorio, archivo o marketplace">
    ```bash
    # Directorio local
    openclaw plugins install ./my-bundle

    # Archivo
    openclaw plugins install ./my-bundle.tgz

    # Marketplace de Claude
    openclaw plugins marketplace list <source>
    openclaw plugins install <plugin> --marketplace <source>
    ```

    `<source>` es una ruta o repositorio de marketplace local, o una fuente de git/GitHub.

  </Step>

  <Step title="Verificar la detección">
    ```bash
    openclaw plugins list
    openclaw plugins inspect <id>
    ```

    Los paquetes muestran `Format: bundle` y un valor `Bundle format:` de `codex`,
    `claude` o `cursor`.

  </Step>

  <Step title="Reiniciar y usar">
    ```bash
    openclaw gateway restart
    ```

    Las funciones asignadas (Skills, hooks, herramientas MCP y valores predeterminados de LSP) están disponibles en la siguiente sesión.

  </Step>
</Steps>

## Qué asigna OpenClaw desde los paquetes

Actualmente, no todas las funciones de los paquetes se ejecutan en OpenClaw. A continuación se indica qué funciona y qué
se detecta, pero todavía no está conectado.

### Compatibilidad actual

| Función              | Cómo se asigna                                                                                                       | Se aplica a      |
| -------------------- | -------------------------------------------------------------------------------------------------------------------- | ---------------- |
| Contenido de Skills  | Las raíces de Skills del paquete se cargan como Skills normales de OpenClaw                                          | Todos los formatos |
| Comandos             | `commands/` y `.cursor/commands/` se tratan como raíces de Skills                                              | Claude, Cursor   |
| Paquetes de hooks    | Diseños de `HOOK.md` + `handler.ts` al estilo de OpenClaw                                             | Codex            |
| Herramientas MCP     | La configuración MCP del paquete se combina con los ajustes de OpenClaw integrado; se cargan servidores stdio y HTTP compatibles | Todos los formatos |
| Servidores LSP       | `.lsp.json` de Claude y `lspServers` declarados en el manifiesto se combinan con los valores predeterminados de LSP de OpenClaw integrado | Claude           |
| Ajustes              | `settings.json` de Claude se importa como valores predeterminados de OpenClaw integrado                            | Claude           |

#### Contenido de Skills

- Las raíces de Skills del paquete se cargan como raíces de Skills normales de OpenClaw.
- Las raíces `commands/` de Claude se tratan como raíces de Skills adicionales.
- Las raíces `.cursor/commands/` de Cursor se tratan como raíces de Skills adicionales.

Tanto los archivos Markdown de comandos de Claude como los comandos Markdown de Cursor funcionan mediante el
cargador normal de Skills de OpenClaw.

#### Paquetes de hooks

Las raíces de hooks del paquete funcionan **solo** cuando utilizan el diseño normal de paquetes de hooks
de OpenClaw: `HOOK.md` más `handler.ts` o `handler.js`. Actualmente, este es principalmente
el caso compatible con Codex.

#### MCP para OpenClaw integrado

- Los paquetes habilitados pueden aportar configuración de servidores MCP.
- OpenClaw combina la configuración MCP del paquete con los ajustes efectivos de OpenClaw
  integrado como `mcpServers`.
- OpenClaw expone las herramientas MCP compatibles del paquete durante los turnos del agente
  de OpenClaw integrado mediante el inicio de servidores stdio o la conexión a servidores HTTP.
- Los perfiles de herramientas `coding` y `messaging` incluyen de forma
  predeterminada las herramientas MCP del paquete; se puede usar `tools.deny: ["bundle-mcp"]` para excluirlas de un agente o Gateway.
- Los ajustes del agente integrado locales del proyecto siguen aplicándose después de los valores predeterminados del paquete, por lo que
  los ajustes del espacio de trabajo pueden anular las entradas MCP del paquete cuando sea necesario.
- Los catálogos de herramientas MCP del paquete se ordenan de forma determinista antes del registro, para que
  los cambios en el orden de `listTools()` en el origen no alteren continuamente los bloques de herramientas de la caché de prompts.

##### Transportes

Los servidores MCP pueden utilizar el transporte stdio o HTTP.

**Stdio** inicia un proceso secundario:

```json
{
  "mcp": {
    "servers": {
      "my-server": {
        "command": "node",
        "args": ["server.js"],
        "env": { "PORT": "3000" }
      }
    }
  }
}
```

**HTTP** se conecta a un servidor MCP en ejecución y utiliza de forma predeterminada `sse`, salvo que
se solicite `streamable-http`:

```json
{
  "mcp": {
    "servers": {
      "my-server": {
        "url": "http://localhost:3100/mcp",
        "transport": "streamable-http",
        "headers": {
          "Authorization": "Bearer ${MY_SECRET_TOKEN}"
        },
        "connectionTimeoutMs": 30000
      }
    }
  }
}
```

- `transport` acepta `"streamable-http"` o `"sse"`; si se omite, el valor predeterminado es `sse`.
- `type: "http"` es una estructura descendente nativa de la CLI; se debe usar `transport: "streamable-http"` en la configuración de OpenClaw. `openclaw mcp set` y `openclaw doctor --fix` normalizan el alias común.
- Solo se permiten los esquemas de URL `http:` y `https:`.
- Los valores `headers` admiten la interpolación de `${ENV_VAR}`.
- Se rechaza una entrada de servidor que contenga tanto `command` como `url`.
- Las credenciales de URL (información de usuario y parámetros de consulta) se ocultan en las descripciones
  de las herramientas y los registros.
- `connectionTimeoutMs` anula el tiempo de espera de conexión predeterminado de 30 segundos para
  los transportes stdio y HTTP. El tiempo de espera de las solicitudes es de 60 segundos de forma predeterminada y
  se puede anular con `requestTimeoutMs`.

##### Nombres de herramientas

OpenClaw registra las herramientas MCP de los paquetes con nombres seguros para el proveedor con el formato
`serverName__toolName`. Por ejemplo, un servidor con la clave `"vigil-harbor"` que expone una
herramienta `memory_search` se registra como `vigil-harbor__memory_search`.

- Los caracteres que no pertenecen a `A-Za-z0-9_-` se sustituyen por `-`.
- Los fragmentos que comenzarían con un carácter que no sea una letra reciben un prefijo alfabético, por lo que las claves
  numéricas de servidores como `12306` se convierten en prefijos de herramientas seguros para el proveedor.
- Los prefijos de servidor están limitados a 30 caracteres.
- Los nombres completos de las herramientas están limitados a 64 caracteres.
- Los nombres de servidor vacíos utilizan `mcp` como alternativa.
- Los nombres saneados que colisionan se distinguen mediante sufijos numéricos.
- El orden final de las herramientas expuestas es determinista según el nombre seguro, lo que mantiene estables
  en caché los turnos repetidos del agente integrado.
- El filtrado de perfiles trata todas las herramientas de un servidor MCP del paquete como
  propiedad del plugin `bundle-mcp`, por lo que las listas de elementos permitidos o denegados del perfil pueden hacer referencia
  a los nombres individuales de las herramientas expuestas o a la clave de plugin `bundle-mcp`.

#### Ajustes de OpenClaw integrado

`settings.json` de Claude se importa como ajustes predeterminados de OpenClaw integrado cuando
el paquete está habilitado. OpenClaw sanea las claves de anulación del shell antes de aplicarlas:

- `shellPath`
- `shellCommandPrefix`

#### LSP de OpenClaw integrado

- Los paquetes de Claude habilitados pueden aportar configuración de servidores LSP.
- OpenClaw carga `.lsp.json` junto con las rutas `lspServers` declaradas en el manifiesto.
- La configuración LSP del paquete se combina con los valores predeterminados efectivos de LSP
  de OpenClaw integrado.
- Actualmente, solo se pueden ejecutar los servidores LSP compatibles basados en stdio; los transportes
  no compatibles siguen apareciendo en `openclaw plugins inspect <id>`.

### Detectado, pero no ejecutado

Estos elementos se reconocen y aparecen en los diagnósticos, pero OpenClaw no los ejecuta:

- `agents` de Claude, automatización `hooks/hooks.json`, `outputStyles`
- `.cursor/agents`, `.cursor/hooks.json`, `.cursor/rules` de Cursor
- Metadatos `.app.json` de Codex más allá de los informes de capacidades

## Formatos de paquetes

<AccordionGroup>
  <Accordion title="Paquetes de Codex">
    Marcadores: `.codex-plugin/plugin.json`

    Contenido opcional: `skills/`, `hooks/`, `.mcp.json`, `.app.json`

    Los paquetes de Codex se adaptan mejor a OpenClaw cuando utilizan raíces de Skills y directorios de
    paquetes de hooks al estilo de OpenClaw (`HOOK.md` + `handler.ts`).

  </Accordion>

  <Accordion title="Paquetes de Claude">
    Dos modos de detección:

    - **Basado en manifiesto:** `.claude-plugin/plugin.json`
    - **Sin manifiesto:** diseño predeterminado de Claude (`skills/`, `commands/`, `agents/`, `hooks/`, `.mcp.json`, `.lsp.json`, `settings.json`)

    Comportamiento específico de Claude:

    - `commands/` se trata como contenido de Skills
    - `settings.json` se importa en los ajustes de OpenClaw integrado (se sanean las claves de anulación del shell)
    - `.mcp.json` expone herramientas stdio compatibles a OpenClaw integrado
    - `.lsp.json` y las rutas `lspServers` declaradas en el manifiesto se cargan en los valores predeterminados de LSP de OpenClaw integrado
    - `hooks/hooks.json` se detecta, pero no se ejecuta
    - Las rutas de componentes personalizadas del manifiesto son aditivas; amplían los valores predeterminados, no los sustituyen

  </Accordion>

  <Accordion title="Paquetes de Cursor">
    Marcadores: `.cursor-plugin/plugin.json`

    Contenido opcional: `skills/`, `.cursor/commands/`, `.cursor/agents/`, `.cursor/rules/`, `.cursor/hooks.json`, `.mcp.json`

    - `.cursor/commands/` se trata como contenido de Skills
    - `.cursor/rules/`, `.cursor/agents/` y `.cursor/hooks.json` solo se detectan

  </Accordion>
</AccordionGroup>

## Precedencia de detección

OpenClaw comprueba primero el formato de plugin nativo:

1. `openclaw.plugin.json` o un `package.json` válido con `openclaw.extensions`: se trata como un **plugin nativo**
2. Marcadores de paquete (`.codex-plugin/`, `.claude-plugin/` o el diseño predeterminado de Claude/Cursor): se trata como un **paquete**

Si un directorio contiene ambos, OpenClaw utiliza la ruta nativa. Esto evita que
los paquetes con formato dual se instalen parcialmente como paquetes.

## Dependencias de tiempo de ejecución y limpieza

- Los paquetes compatibles de terceros no reciben la reparación de `npm install` al iniciarse. Deben
  instalarse mediante `openclaw plugins install` e incluir todo lo
  necesario en el directorio del plugin instalado.
- Los plugins incluidos propiedad de OpenClaw se distribuyen de forma ligera en el núcleo o
  se pueden descargar mediante el instalador de plugins. El inicio del Gateway nunca ejecuta un
  gestor de paquetes para ellos.
- `openclaw doctor --fix` elimina los registros obsoletos de instalaciones locales de plugins incluidos
  y puede recuperar plugins descargables que falten en el índice local de plugins
  cuando la configuración todavía haga referencia a ellos.

## Seguridad

Los paquetes tienen un límite de confianza más restringido que los plugins nativos:

- OpenClaw **no** carga módulos arbitrarios de tiempo de ejecución de paquetes dentro del proceso.
- Las rutas de Skills y paquetes de hooks deben permanecer dentro de la raíz del plugin (se comprueban los límites).
- Los archivos de ajustes se leen con las mismas comprobaciones de límites.
- Los servidores MCP stdio compatibles pueden iniciarse como subprocesos.

Esto hace que los paquetes sean más seguros de forma predeterminada, pero aun así se deben tratar los paquetes
de terceros como contenido de confianza para las funciones que exponen.

## Solución de problemas

<AccordionGroup>
  <Accordion title="El paquete se detecta, pero las capacidades no se ejecutan">
    Ejecute `openclaw plugins inspect <id>`. Si una capacidad aparece en la lista, pero está marcada como
    no conectada, se trata de una limitación del producto, no de una instalación defectuosa.
  </Accordion>

  <Accordion title="Los archivos de comandos de Claude no aparecen">
    Asegúrese de que el paquete esté habilitado y de que los archivos Markdown se encuentren dentro de una raíz
    `commands/` o `skills/` detectada.
  </Accordion>

  <Accordion title="La configuración de Claude no se aplica">
    Solo se admite la configuración integrada de OpenClaw procedente de `settings.json`. OpenClaw
    no trata la configuración del paquete como parches de configuración sin procesar.
  </Accordion>

  <Accordion title="Los hooks de Claude no se ejecutan">
    `hooks/hooks.json` solo se detecta. Si necesita hooks ejecutables, use la
    estructura de paquetes de hooks de OpenClaw o distribuya un plugin nativo.
  </Accordion>
</AccordionGroup>

## Temas relacionados

- [Instalar y configurar plugins](/es/tools/plugin)
- [Creación de plugins](/es/plugins/building-plugins) - cree un plugin nativo
- [Manifiesto de plugin](/es/plugins/manifest) - esquema del manifiesto nativo
