---
read_when:
    - Quieres que los agentes muestren las modificaciones de código o Markdown como diferencias.
    - Se necesita una URL de visor lista para el lienzo o un archivo de diferencias renderizado
    - Necesita artefactos de diferencias controlados y temporales con valores predeterminados seguros
sidebarTitle: Diffs
summary: Visor de diferencias de solo lectura y renderizador de archivos para agentes (herramienta de Plugin opcional)
title: Diferencias
x-i18n:
    generated_at: "2026-07-26T05:23:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: baeb5dd1277120e57178f092e3ae1616edd3389a54721c929d8711301535d302
    source_path: tools/diffs.md
    workflow: 16
---

`diffs` es una herramienta opcional de un plugin incluido que convierte texto anterior/posterior o un parche unificado en un artefacto de diferencias de solo lectura. También antepone instrucciones breves para el agente al prompt del sistema e incluye una Skill complementaria con instrucciones más completas.

Entrada: texto `before` + `after`, o un `patch` unificado (mutuamente excluyentes).

Salida: una URL del visor del Gateway para su presentación en el lienzo, una ruta de archivo PNG/PDF renderizado para su entrega mediante mensajes, o ambas.

## Inicio rápido

<Steps>
  <Step title="Instalar el plugin">
    ```bash
    openclaw plugins install diffs
    ```
  </Step>
  <Step title="Habilitar el plugin">
    ```json5
    {
      plugins: {
        entries: {
          diffs: {
            enabled: true,
          },
        },
      },
    }
    ```
  </Step>
  <Step title="Elegir un modo">
    <Tabs>
      <Tab title="view">
        Flujos centrados en el lienzo: los agentes llaman a `diffs` con `mode: "view"` y abren `details.viewerUrl` con `canvas present`.
      </Tab>
      <Tab title="file">
        Entrega de archivos por chat: los agentes llaman a `diffs` con `mode: "file"` y envían `details.filePath` con `message` mediante `path` o `filePath`.
      </Tab>
      <Tab title="both">
        Combinado (predeterminado): los agentes llaman a `diffs` con `mode: "both"` para obtener ambos artefactos en una sola llamada.
      </Tab>
    </Tabs>
  </Step>
</Steps>

## Deshabilitar las instrucciones integradas del sistema

Para conservar la herramienta, pero eliminar las instrucciones antepuestas al prompt del sistema, establezca `plugins.entries.diffs.hooks.allowPromptInjection` en `false`:

```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
        hooks: {
          allowPromptInjection: false,
        },
      },
    },
  },
}
```

Esto bloquea el hook `before_prompt_build` del plugin, pero mantiene disponibles la herramienta y la Skill. Para deshabilitar tanto las instrucciones como la herramienta, deshabilite el plugin.

## Referencia de entrada de la herramienta

Todos los campos son opcionales, salvo que se indique lo contrario.

<ParamField path="before" type="string">
  Texto original. Obligatorio junto con `after` cuando se omite `patch`.
</ParamField>
<ParamField path="after" type="string">
  Texto actualizado. Obligatorio junto con `before` cuando se omite `patch`.
</ParamField>
<ParamField path="patch" type="string">
  Texto de diferencias unificado. Mutuamente excluyente con `before` y `after`.
</ParamField>
<ParamField path="path" type="string">
  Nombre de archivo mostrado para el modo anterior/posterior.
</ParamField>
<ParamField path="lang" type="string">
  Indicación de idioma que anula el valor predeterminado para el modo anterior/posterior. Los valores desconocidos y los idiomas que no pertenecen al conjunto predeterminado del visor recurren a texto sin formato, a menos que esté instalado el plugin Diff Viewer Language Pack.
</ParamField>
<ParamField path="title" type="string">
  Título del visor que anula el valor predeterminado.
</ParamField>
<ParamField path="mode" type='"view" | "file" | "both"'>
  Modo de salida. El valor predeterminado es el del plugin `defaults.mode` (`both`). Alias obsoleto: `"image"` se comporta de forma idéntica a `"file"`.
</ParamField>
<ParamField path="theme" type='"light" | "dark"'>
  Tema del visor. El valor predeterminado es el del plugin `defaults.theme`.
</ParamField>
<ParamField path="layout" type='"unified" | "split"'>
  Diseño de las diferencias. El valor predeterminado es el del plugin `defaults.layout`.
</ParamField>
<ParamField path="expandUnchanged" type="boolean">
  Expande las secciones sin cambios cuando está disponible el contexto completo. Opción disponible solo por llamada (no es una clave predeterminada del plugin).
</ParamField>
<ParamField path="fileFormat" type='"png" | "pdf"'>
  Formato del archivo renderizado. El valor predeterminado es el del plugin `defaults.fileFormat`.
</ParamField>
<ParamField path="fileQuality" type='"standard" | "hq" | "print"'>
  Preajuste de calidad para la renderización en PNG/PDF.
</ParamField>
<ParamField path="fileScale" type="number">
  Escala del dispositivo que anula el valor predeterminado (`1`-`4`).
</ParamField>
<ParamField path="fileMaxWidth" type="number">
  Anchura máxima de renderización en píxeles CSS (`640`-`2400`).
</ParamField>
<ParamField path="ttlSeconds" type="number" default="1800">
  TTL del artefacto en segundos para las salidas del visor y de archivos independientes. Máximo: `21600`.
</ParamField>
<ParamField path="baseUrl" type="string">
  Origen de la URL del visor que anula el valor predeterminado. Anula el valor `viewerBaseUrl` del plugin. Debe ser `http` o `https`, sin consulta ni hash.
</ParamField>

<AccordionGroup>
  <Accordion title="Validación y límites">
    - `before`/`after`: máximo de 512 KiB cada uno.
    - `patch`: máximo de 2 MiB.
    - `path`: máximo de 2048 bytes.
    - `lang`: máximo de 128 bytes.
    - `title`: máximo de 1024 bytes.
    - Límite de complejidad del parche: máximo de 128 archivos y 120000 líneas en total.
    - Se rechaza `patch` junto con `before`/`after`.
    - Límites de seguridad de los archivos renderizados (PNG y PDF):
      - `fileQuality: "standard"`: máximo de 8 MP (8,000,000 píxeles renderizados).
      - `fileQuality: "hq"`: máximo de 14 MP.
      - `fileQuality: "print"`: máximo de 24 MP.
      - El PDF también tiene un límite de 50 páginas.

  </Accordion>
</AccordionGroup>

## Resaltado de sintaxis

Idiomas integrados:

`javascript`, `typescript`, `tsx`, `jsx`, `json`, `markdown`, `yaml`, `css`, `html`, `sh`, `python`, `go`, `rust`, `java`, `c`, `cpp`, `csharp`, `php`, `sql`, `docker`, `ruby`, `swift`, `kotlin`, `r`, `dart`, `lua`, `powershell`, `xml` y `toml`.

Los alias habituales (`js`, `ts`, `bash`, `md`, `yml`, `c++`, `dockerfile`, `rb`, `kt`, `ps1`, etc.) se normalizan a esos idiomas.

Instale el plugin Diff Viewer Language Pack para disponer de más idiomas (Astro, Vue, Svelte, MDX, GraphQL, Terraform/HCL, Nix, Clojure, Elixir, Haskell, OCaml, Scala, Zig, Solidity, Verilog/VHDL, Fortran, MATLAB, LaTeX, Mermaid, Sass/Less/SCSS, Nginx, Apache, CSV, dotenv, INI, diff y más):

```bash
openclaw plugins install clawhub:@openclaw/diffs-language-pack
```

Sin el paquete, los idiomas no compatibles se siguen renderizando como texto sin formato legible. Consulte [plugin Diffs Language Pack](/es/plugins/reference/diffs-language-pack) e [idiomas de Shiki](https://shiki.style/languages) para ver el catálogo del proyecto de origen.

## Contrato de detalles de salida

Todos los resultados correctos incluyen `changed`: una entrada anterior/posterior idéntica devuelve `false` sin crear ningún artefacto; los resultados renderizados devuelven `true`.

<AccordionGroup>
  <Accordion title="Campos del visor (modos view y both)">
    - `changed`
    - `artifactId`
    - `viewerUrl`
    - `viewerPath`
    - `title`
    - `expiresAt`
    - `inputKind`
    - `fileCount`
    - `mode`
    - `context` (`agentId`, `sessionId`, `messageChannel`, `agentAccountId` cuando estén disponibles)

  </Accordion>
  <Accordion title="Campos de archivo (modos file y both)">
    - `changed`
    - `artifactId`
    - `expiresAt`
    - `filePath`
    - `path` (el mismo valor que `filePath`, para mantener la compatibilidad con la herramienta de mensajes)
    - `fileBytes`
    - `fileFormat`
    - `fileQuality`
    - `fileScale`
    - `fileMaxWidth`

  </Accordion>
</AccordionGroup>

| Modo     | Devuelve                                                                                         |
| -------- | ----------------------------------------------------------------------------------------------- |
| `"view"` | Solo campos del visor.                                                                             |
| `"file"` | Solo campos de archivo, sin artefacto del visor.                                                           |
| `"both"` | Campos del visor y campos de archivo. Si falla la renderización del archivo, el visor se sigue devolviendo con `fileError`. |

### Secciones sin cambios contraídas

El visor muestra filas como `N unmodified lines`. Los controles para expandir solo aparecen cuando las diferencias renderizadas contienen datos de contexto expandibles (algo habitual con entradas anterior/posterior). Muchos parches unificados omiten los cuerpos de contexto de sus fragmentos, por lo que la fila puede aparecer sin un control para expandir; es el comportamiento esperado, no un error. `expandUnchanged` solo se aplica cuando existe contexto expandible.

### Navegación entre varios archivos

Los parches que afectan a más de un archivo comienzan con una tarjeta de resumen de los archivos modificados: recuentos totales de `+N` / `-N`, recuentos por archivo, insignias de archivos añadidos/eliminados/renombrados y enlaces de anclaje para ir a cada archivo. Los archivos PNG/PDF renderizados conservan los recuentos del encabezado de cada archivo, pero omiten los controles interactivos para cambiar de vista, ya que no funcionan en un archivo estático.

## Valores predeterminados del plugin

Establezca los valores predeterminados del plugin en `~/.openclaw/openclaw.json`:

```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
        config: {
          defaults: {
            fontFamily: "Fira Code",
            fontSize: 15,
            lineSpacing: 1.6,
            layout: "unified",
            showLineNumbers: true,
            diffIndicators: "bars",
            wordWrap: true,
            background: true,
            theme: "dark",
            fileFormat: "png",
            fileQuality: "standard",
            fileScale: 2,
            fileMaxWidth: 960,
            mode: "both",
            ttlSeconds: 21600,
          },
        },
      },
    },
  },
}
```

Claves `defaults` compatibles: `fontFamily`, `fontSize`, `lineSpacing`, `layout`, `showLineNumbers`, `diffIndicators`, `wordWrap`, `background`, `theme`, `fileFormat`, `fileQuality`, `fileScale`, `fileMaxWidth`, `mode`, `ttlSeconds`. Los parámetros explícitos de la llamada a la herramienta anulan estos valores.

### Configuración persistente de la URL del visor

<ParamField path="viewerBaseUrl" type="string">
  Alternativa propiedad del plugin para los enlaces del visor devueltos cuando una llamada a la herramienta no proporciona `baseUrl`. Debe ser `http` o `https`, sin consulta ni hash.
</ParamField>

```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
        config: {
          viewerBaseUrl: "https://gateway.example.com/openclaw",
        },
      },
    },
  },
}
```

## Configuración de seguridad

<ParamField path="security.allowRemoteViewer" type="boolean" default="false">
  `false`: se deniegan las solicitudes que no proceden de la interfaz de bucle invertido dirigidas a las rutas del visor. `true`: se permiten los visores remotos si la ruta con token es válida.
</ParamField>

```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
        config: {
          security: {
            allowRemoteViewer: false,
          },
        },
      },
    },
  },
}
```

## Ciclo de vida y almacenamiento de los artefactos

- El HTML y los metadatos del visor se almacenan en la base de datos compartida `state/openclaw.sqlite`, dentro del espacio de nombres de blobs del plugin Diffs. El HTML se comprime con gzip; SQLite almacena únicamente un hash SHA-256 del token aleatorio de la URL, no el token en sí.
- Los archivos PNG/PDF renderizados siguen siendo materializaciones temporales en `$TMPDIR/openclaw-diffs`, porque la entrega mediante canales requiere una ruta de archivo. SQLite gestiona sus metadatos de caducidad; no se escriben archivos JSON auxiliares.
- TTL predeterminado de los artefactos: 30 minutos. TTL máximo aceptado: 6 horas.
- La limpieza se ejecuta de forma oportunista después de cada llamada de creación de artefactos. Primero se eliminan las filas caducadas de SQLite y, después, cualquier directorio PNG/PDF correspondiente.
- Un barrido de respaldo elimina las carpetas temporales sin filas asociadas que tengan más de 24 horas. Las cachés heredadas `meta.json`, `file-meta.json` y `viewer.html` no se importan ni se leen.

## URL del visor y comportamiento de red

Ruta del visor: `/plugins/diffs/view/{artifactId}/{token}`

Recursos del visor:

- `/plugins/diffs/assets/viewer.js`
- `/plugins/diffs/assets/viewer-runtime.js`
- `/plugins/diffs-language-pack/assets/viewer.js` (solo cuando el diff utiliza un idioma del paquete de idiomas)

El documento del visor resuelve estos recursos con respecto a la URL del visor, por lo que un prefijo de ruta opcional `baseUrl` también se aplica a las solicitudes de recursos.

Orden de resolución de URL: `baseUrl` de la llamada a la herramienta (tras una validación estricta) -> `viewerBaseUrl` del plugin -> valor predeterminado de bucle invertido `127.0.0.1`. Si el modo de enlace del Gateway es `custom` y se ha establecido `gateway.customBindHost`, se utiliza ese host en lugar del bucle invertido.

Reglas de `baseUrl`: debe ser `http://` o `https://`; se rechazan la consulta y el hash; se permite el origen con una ruta base opcional.

## Modelo de seguridad

<AccordionGroup>
  <Accordion title="Refuerzo del visor">
    - Solo bucle invertido de forma predeterminada.
    - Rutas del visor con tokens y validación estricta de los patrones de ID y token.
    - CSP de la respuesta del visor: `default-src 'none'`; scripts y recursos únicamente del propio origen; sin `connect-src` salientes.
    - Limitación de intentos fallidos remotos cuando el acceso remoto está habilitado: 40 fallos en 60 segundos activan un bloqueo de 60 segundos (`429 Too Many Requests`).

  </Accordion>
  <Accordion title="Refuerzo de la renderización de archivos">
    - El enrutamiento de solicitudes del navegador para capturas de pantalla se deniega de forma predeterminada.
    - Solo se permiten recursos locales del visor procedentes de `http://127.0.0.1/plugins/diffs/assets/*`.
    - Las solicitudes de red externas están bloqueadas.

  </Accordion>
</AccordionGroup>

## Requisitos del navegador para el modo de archivo

`mode: "file"` y `mode: "both"` necesitan un navegador compatible con Chromium.

Orden de resolución:

<Steps>
  <Step title="Configuración">
    `browser.executablePath` en la configuración de OpenClaw.
  </Step>
  <Step title="Variables de entorno">
    - `OPENCLAW_BROWSER_EXECUTABLE_PATH`
    - `BROWSER_EXECUTABLE_PATH`
    - `PLAYWRIGHT_CHROMIUM_EXECUTABLE_PATH`

  </Step>
  <Step title="Alternativa de la plataforma">
    Rutas de instalación habituales y búsquedas de `PATH` para Chrome, Chromium, Edge y Brave.
  </Step>
</Steps>

Texto de error habitual: `Diff PNG/PDF rendering requires a Chromium-compatible browser...`. Para solucionarlo, instale Chrome, Chromium, Edge o Brave, o establezca una de las opciones de ruta del ejecutable indicadas anteriormente.

## Solución de problemas

<AccordionGroup>
  <Accordion title="Errores de validación de entrada">
    - `Provide patch or both before and after text.` -- incluya tanto `before` como `after`, o proporcione `patch`.
    - `Provide either patch or before/after input, not both.` -- no mezcle modos de entrada.
    - `Invalid baseUrl: ...` -- utilice un origen `http(s)` con una ruta opcional, sin consulta ni hash.
    - `{field} exceeds maximum size (...)` -- reduzca el tamaño de la carga útil.
    - Rechazo de un parche grande -- reduzca el número de archivos del parche o el total de líneas.

  </Accordion>
  <Accordion title="Accesibilidad del visor">
    - La URL del visor se resuelve como `127.0.0.1` de forma predeterminada.
    - Para el acceso remoto, establezca `viewerBaseUrl` en el plugin, pase `baseUrl` en cada llamada o utilice `gateway.bind=custom` con `gateway.customBindHost`.
    - Si `gateway.trustedProxies` incluye el bucle invertido para un proxy del mismo host (por ejemplo, Tailscale Serve), las solicitudes directas al visor mediante el bucle invertido sin encabezados de IP del cliente reenviados se cierran de forma segura por diseño.
    - Para esa topología de proxy, se recomienda `mode: "file"`/`"both"` para un archivo adjunto, o habilitar intencionadamente `security.allowRemoteViewer` junto con `viewerBaseUrl` en el plugin/un `baseUrl` del proxy para obtener un enlace compartible del visor.
    - Habilite `security.allowRemoteViewer` únicamente cuando se pretenda permitir el acceso externo al visor.

  </Accordion>
  <Accordion title="La fila de líneas sin modificar no tiene botón de expansión">
    Es lo esperado para una entrada de parche que carece de contexto ampliable; no se trata de un fallo del visor.
  </Accordion>
  <Accordion title="Artefacto no encontrado">
    - El artefacto ha caducado debido al TTL.
    - El token o la ruta han cambiado.
    - La limpieza ha eliminado datos obsoletos.

  </Accordion>
</AccordionGroup>

## Orientación operativa

- Se recomienda `mode: "view"` para revisiones interactivas locales en el lienzo.
- Se recomienda `mode: "file"` para canales de chat salientes que necesiten un archivo adjunto.
- Mantenga `allowRemoteViewer` deshabilitado, salvo que el despliegue requiera URL remotas del visor.
- Establezca un `ttlSeconds` corto y explícito para diffs sensibles.
- Evite enviar secretos en la entrada del diff cuando no sea necesario.
- Si el canal comprime las imágenes de forma agresiva (por ejemplo, Telegram o WhatsApp), se recomienda la salida PDF (`fileFormat: "pdf"`).

<Note>
Motor de renderización de diffs desarrollado por [Diffs](https://diffs.com).
</Note>

## Contenido relacionado

- [Navegador](/es/tools/browser)
- [Plugins](/es/tools/plugin)
- [Descripción general de las herramientas](/es/tools)
