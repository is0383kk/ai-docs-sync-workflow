---
read_when:
    - Añadir/cambiar endpoints
    - Depuración de solicitudes entre la CLI y el registro
summary: Referencia de la API HTTP (endpoints públicos y de la CLI, y autenticación).
x-i18n:
    generated_at: "2026-07-26T04:32:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5b180bbd56d20a3d88c1fe74ccab0fd0ecbe0e8c9624cd1afd2070a2ca1f7fb3
    source_path: clawhub/http-api.md
    workflow: 16
---

# API HTTP

URL base: `https://clawhub.ai` (predeterminada).

Todas las rutas v1 se encuentran bajo `/api/v1/...`.
Las rutas heredadas `/api/...` y `/api/cli/...` se mantienen por compatibilidad (consulte `DEPRECATIONS.md`).
OpenAPI: `/api/v1/openapi.json`.

## Reutilización del catálogo público

Los directorios de terceros pueden usar los endpoints públicos de lectura para enumerar o buscar Skills de ClawHub. Almacene los resultados en caché, respete `429`/`Retry-After`, dirija a los usuarios al listado canónico de ClawHub (`https://clawhub.ai/<owner>/skills/<slug>`) y evite dar a entender que ClawHub respalda el sitio de terceros. No intente replicar contenido oculto, privado o bloqueado por moderación fuera de la superficie de la API pública.

Los accesos directos mediante slugs web se resuelven entre familias del registro, pero los clientes de la API deben usar
las URL canónicas devueltas por los endpoints de lectura en lugar de reconstruir la precedencia
de las rutas.

## Límites de velocidad

Modelo de aplicación:

- Solicitudes anónimas: se aplica por IP.
- Solicitudes autenticadas (token Bearer válido): se aplica por grupo de usuario.
- Si el token falta o no es válido, el comportamiento recurre a la aplicación por IP.
- Los endpoints de escritura autenticados no deben devolver únicamente `Unauthorized` cuando
  el servidor conoce el motivo. Los tokens ausentes, los tokens no válidos o revocados y
  las cuentas eliminadas, bloqueadas o deshabilitadas deben recibir texto procesable para que los clientes
  CLI puedan indicar a los usuarios qué los bloqueó.

- Lectura: 3000/min por IP, 12000/min por clave
- Escritura: 300/min por IP, 3000/min por clave
- Descarga: 1200/min por IP, 6000/min por clave (endpoints de descarga)

Encabezados:

- Compatibilidad heredada: `X-RateLimit-Limit`, `X-RateLimit-Reset`
- Estandarizados: `RateLimit-Limit`, `RateLimit-Reset`
- En `429`: `X-RateLimit-Remaining: 0` y `RateLimit-Remaining: 0`
- En `429`: `Retry-After`

Semántica de los encabezados:

- `X-RateLimit-Reset`: segundos absolutos desde la época Unix
- `RateLimit-Reset`: segundos hasta el restablecimiento (demora)
- `X-RateLimit-Remaining` / `RateLimit-Remaining`: presupuesto restante exacto cuando está presente.
  Las solicitudes fragmentadas correctas omiten este encabezado en lugar de devolver un valor global aproximado.
- `Retry-After`: segundos de espera antes de volver a intentarlo (demora) en `429`

Ejemplo de respuesta `429`:

```http
HTTP/2 429
content-type: text/plain; charset=utf-8
x-ratelimit-limit: 20
x-ratelimit-remaining: 0
x-ratelimit-reset: 1771404540
ratelimit-limit: 20
ratelimit-remaining: 0
ratelimit-reset: 34
retry-after: 34

Límite de velocidad superado
```

Directrices para clientes:

- Si existe `Retry-After`, espere esa cantidad de segundos antes de volver a intentarlo.
- Use una espera exponencial con variación aleatoria para evitar reintentos sincronizados.
- Si falta `Retry-After`, recurra a `RateLimit-Reset` (o calcúlelo a partir de `X-RateLimit-Reset`).

Origen de la IP:

- Usa encabezados de IP de cliente de confianza, incluido `cf-connecting-ip`, solo cuando el
  despliegue habilita explícitamente los encabezados reenviados de confianza.
- ClawHub usa encabezados de reenvío de confianza para identificar las IP de los clientes en el perímetro.
- Si no hay disponible ninguna IP de cliente de confianza, las solicitudes anónimas usan grupos alternativos
  cuyo ámbito se limita exclusivamente al tipo de límite de velocidad. Estos grupos alternativos no incluyen
  rutas, slugs, nombres de paquetes, versiones, cadenas de consulta ni otros
  parámetros de artefactos proporcionados por el solicitante.

## Respuestas de error

Las respuestas de error públicas de v1 son texto sin formato con `content-type: text/plain; charset=utf-8`.
Esto incluye errores de validación (`400`), recursos públicos ausentes (`404`), errores de autenticación y
permisos (`401`/`403`), límites de velocidad (`429`) y descargas bloqueadas. Los clientes
deben leer el cuerpo de la respuesta como una cadena legible para humanos. Los parámetros de consulta desconocidos se
ignoran por compatibilidad, pero los parámetros de consulta reconocidos con valores no válidos devuelven
`400`.

## Endpoints públicos (sin autenticación)

### `GET /api/v1/search`

Parámetros de consulta:

- `q` (obligatorio): cadena de consulta
- `limit` (opcional): entero
- `highlightedOnly` (opcional): `true` para filtrar las Skills destacadas
- `nonSuspiciousOnly` (opcional): `true` para ocultar las Skills sospechosas (`flagged.suspicious`)
- `nonSuspicious` (opcional): alias heredado de `nonSuspiciousOnly`

Respuesta:

```json
{
  "results": [
    {
      "score": 0.123,
      "slug": "gifgrep",
      "displayName": "GifGrep",
      "summary": "…",
      "version": "1.2.3",
      "updatedAt": 1730000000000,
      "ownerHandle": "openclaw",
      "owner": {
        "handle": "openclaw",
        "displayName": "OpenClaw",
        "image": "https://example.com/avatar.png"
      }
    }
  ]
}
```

Notas:

- Los resultados se devuelven en orden de relevancia (similitud de incrustaciones + aumentos por coincidencia exacta de tokens de slug/nombre + una pequeña ponderación previa de popularidad).
- La relevancia tiene más peso que la popularidad. Una coincidencia precisa de un token de slug o nombre para mostrar puede superar a una coincidencia menos precisa con mucha más interacción.
- El texto ASCII se divide en tokens en los límites de palabras y signos de puntuación. Por ejemplo, `personal-map` contiene un token independiente `map`, mientras que `amap-jsapi-skill` contiene `amap`, `jsapi` y `skill`; por lo tanto, buscar `map` otorga a `personal-map` una coincidencia léxica más fuerte que a `amap-jsapi-skill`.
- La popularidad se escala logarítmicamente y tiene un límite máximo. Las Skills con mucha interacción pueden clasificarse por debajo cuando el texto de la consulta presenta una coincidencia más débil.
- Un estado de moderación sospechoso u oculto puede eliminar una Skill de la búsqueda pública según los filtros del solicitante y el estado de moderación actual.

Directrices de visibilidad para editores:

- Incluya los términos que los usuarios buscarán literalmente en el nombre para mostrar, el resumen y las etiquetas. Use un token de slug independiente solo cuando también sea una identidad estable que se desee conservar.
- No cambie el nombre de un slug solo para favorecer una consulta, a menos que el nuevo slug sea un mejor nombre canónico a largo plazo. Los slugs anteriores se convierten en alias de redirección, pero la URL canónica, el slug mostrado y los futuros resúmenes de búsqueda usan el nuevo slug.
- Los alias de cambio de nombre conservan la resolución de URL anteriores y de instalaciones que se resuelven mediante el registro, pero la clasificación de búsqueda se basa en los metadatos canónicos de la Skill después de que se indexa el cambio de nombre. Las estadísticas existentes permanecen asociadas a la Skill.
- Si una Skill está inesperadamente invisible, compruebe primero el estado de moderación con `clawhub inspect @owner/slug` tras iniciar sesión antes de cambiar los metadatos relacionados con la clasificación.

### `GET /api/v1/skills`

Parámetros de consulta:

- `limit` (opcional): entero (1–200)
- `cursor` (opcional): cursor de paginación para cualquier ordenación distinta de `trending`
- `sort` (opcional): `updated` (predeterminado), `recommended` (alias: `default`), `createdAt` (alias: `newest`), `downloads`, `stars` (alias: `rating`), los alias de instalación heredados `installsCurrent`/`installs`/`installsAllTime` se asignan a `downloads`, `trending`
- `nonSuspiciousOnly` (opcional): `true` para ocultar las Skills sospechosas (`flagged.suspicious`)
- `nonSuspicious` (opcional): alias heredado de `nonSuspiciousOnly`

Los valores no válidos de `sort` devuelven `400`.

Notas:

- `recommended` usa señales de interacción y actualidad.
- `trending` clasifica según las instalaciones de los últimos 7 días (basadas en telemetría).
- `createdAt` es estable para los rastreos de Skills nuevas; `updated` cambia cuando se vuelven a publicar Skills existentes.
- Cuando `nonSuspiciousOnly=true`, las ordenaciones basadas en cursores pueden devolver menos de `limit` elementos en una página porque las Skills sospechosas se filtran después de recuperar la página.
- Use `nextCursor` para continuar la paginación cuando esté presente. Una página corta no implica por sí sola el final de los resultados.

Respuesta:

```json
{
  "items": [
    {
      "slug": "gifgrep",
      "displayName": "GifGrep",
      "summary": "…",
      "topics": ["Productivity"],
      "tags": { "latest": "1.2.3" },
      "stats": {},
      "createdAt": 0,
      "updatedAt": 0,
      "latestVersion": { "version": "1.2.3", "createdAt": 0, "changelog": "…" },
      "metadata": { "os": ["macos"], "systems": ["aarch64-darwin"] }
    }
  ],
  "nextCursor": null
}
```

### `GET /api/v1/skills/{slug}`

Respuesta:

```json
{
  "skill": {
    "slug": "gifgrep",
    "displayName": "GifGrep",
    "summary": "…",
    "topics": ["Productivity"],
    "tags": { "latest": "1.2.3" },
    "stats": {},
    "createdAt": 0,
    "updatedAt": 0
  },
  "latestVersion": { "version": "1.2.3", "createdAt": 0, "changelog": "…" },
  "metadata": { "os": ["macos"], "systems": ["aarch64-darwin"] },
  "owner": { "handle": "steipete", "displayName": "Peter", "image": null },
  "moderation": {
    "isSuspicious": false,
    "isMalwareBlocked": false,
    "verdict": "clean",
    "reasonCodes": [],
    "summary": null,
    "engineVersion": "v2.0.0",
    "updatedAt": 0
  }
}
```

Notas:

- Los slugs anteriores creados mediante flujos de cambio de nombre o fusión por parte del propietario se resuelven a la Skill canónica.
- `metadata.os`: restricciones del sistema operativo declaradas en el frontmatter de la Skill (p. ej., `["macos"]`, `["linux"]`). `null` si no se declaran.
- `metadata.systems`: objetivos de sistema Nix (p. ej., `["aarch64-darwin", "x86_64-linux"]`). `null` si no se declaran.
- `metadata` es `null` si la Skill no tiene metadatos de plataforma.
- `moderation` solo se incluye cuando la Skill está marcada o su propietario la está viendo.

### `GET /api/v1/skills/{slug}/moderation`

Devuelve el estado de moderación estructurado.

Respuesta:

```json
{
  "moderation": {
    "isSuspicious": true,
    "isMalwareBlocked": false,
    "verdict": "suspicious",
    "reasonCodes": ["suspicious.dynamic_code_execution"],
    "summary": "Detectado: suspicious.dynamic_code_execution",
    "engineVersion": "v2.0.0",
    "updatedAt": 0,
    "legacyReason": null,
    "evidence": [
      {
        "code": "suspicious.dynamic_code_execution",
        "severity": "critical",
        "file": "index.ts",
        "line": 3,
        "message": "Se detectó la ejecución dinámica de código.",
        "evidence": ""
      }
    ]
  }
}
```

Notas:

- Los propietarios y moderadores pueden acceder a los detalles de moderación de las Skills ocultas.
- Los solicitantes públicos solo reciben `200` para las Skills visibles que ya estén marcadas.
- Las pruebas se censuran para los solicitantes públicos y solo incluyen fragmentos sin procesar para los propietarios o moderadores.

### `POST /api/v1/skills/{slug}/report`

Informa sobre una Skill para que la revisen los moderadores. Los informes corresponden a la Skill en su conjunto, pueden vincularse
opcionalmente a una versión y alimentan la cola de informes de Skills.

Autenticación:

- Requiere un token de API.

Solicitud:

```json
{ "reason": "Paso de instalación sospechoso", "version": "1.2.3" }
```

Respuesta:

```json
{
  "ok": true,
  "reported": true,
  "alreadyReported": false,
  "reportId": "skillReports:...",
  "skillId": "skills:...",
  "reportCount": 1
}
```

### `GET /api/v1/skills/-/reports`

Endpoint de moderación/administración para la recepción de informes de Skills.

Parámetros de consulta:

- `status` (opcional): `open` (predeterminado), `confirmed`, `dismissed` o `all`
- `limit` (opcional): entero (1-200)
- `cursor` (opcional): cursor de paginación

Respuesta:

```json
{
  "items": [
    {
      "reportId": "skillReports:...",
      "skillId": "skills:...",
      "skillVersionId": "skillVersions:...",
      "slug": "gifgrep",
      "displayName": "GifGrep",
      "version": "1.2.3",
      "reason": "Paso de instalación sospechoso",
      "status": "open",
      "createdAt": 1730000000000,
      "reporter": {
        "userId": "users:...",
        "handle": "reporter",
        "displayName": "Denunciante"
      },
      "triagedAt": null,
      "triagedBy": null,
      "triageNote": null
    }
  ],
  "nextCursor": null,
  "done": true
}
```

### `POST /api/v1/skills/-/reports/{reportId}/triage`

Endpoint para moderadores y administradores destinado a resolver o reabrir informes de Skills.

Solicitud:

```json
{ "status": "confirmed", "note": "Revisada y ocultada la versión afectada.", "finalAction": "hide" }
```

`note` es obligatorio para `confirmed` y `dismissed`; puede omitirse al
volver a establecer `status` en `open`. Pase `finalAction: "hide"` con un informe
clasificado para ocultar la Skill en el mismo flujo de trabajo auditable.

### `GET /api/v1/skills/{slug}/versions`

Parámetros de consulta:

- `limit` (opcional): entero
- `cursor` (opcional): cursor de paginación

### `GET /api/v1/skills/{slug}/versions/{version}`

Devuelve los metadatos de la versión y la lista de archivos.

- `version.security` incluye el estado normalizado de verificación del análisis y los detalles de los analizadores
  (VirusTotal + LLM), cuando están disponibles.

### `GET /api/v1/skills/{slug}/scan`

Devuelve los detalles de verificación del análisis de seguridad de una versión de una Skill.

Parámetros de consulta:

- `version` (opcional): cadena de versión específica.
- `tag` (opcional): resuelve una versión etiquetada (por ejemplo, `latest`).

Notas:

- Si no se proporcionan ni `version` ni `tag`, utiliza la versión más reciente.
- Incluye el estado normalizado de verificación y los detalles específicos de cada analizador.
- `security.hasScanResult` es `true` solo cuando un analizador produjo un veredicto definitivo (`clean`, `suspicious` o `malicious`).
- `moderation` es una instantánea actual de moderación a nivel de Skill derivada de la versión más reciente.
- Al consultar una versión histórica, compruebe `moderation.matchesRequestedVersion` y `moderation.sourceVersion` antes de considerar que `moderation` y `security` pertenecen al mismo contexto de versión.

### `POST /api/v1/skills/-/scan`

Endpoint autenticado de envío para nuevos trabajos de ClawScan.

Ya no se admiten los análisis de cargas locales. Las solicitudes que utilizan
`multipart/form-data` o `{ "source": { "kind": "upload" } }` devuelven `410`.

Los análisis publicados utilizan JSON:

```json
{
  "source": { "kind": "published", "slug": "gifgrep", "version": "1.2.3" },
  "update": false
}
```

Notas:

- Las cargas de las solicitudes de análisis y los informes descargables caducan en el almacén de solicitudes de análisis una vez transcurrido el período de retención.
- Los análisis publicados requieren acceso de administración del propietario o publicador, o autoridad de moderador o administrador de la plataforma.
- Los análisis publicados solo escriben los resultados cuando se usa `update: true` y el análisis finaliza correctamente.
- La respuesta es `202` con `{ "ok": true, "scanId": "...", "jobId": "...", "status": "queued", "sourceKind": "published", "update": false, "queue": { "queuedAhead": 0, "queuedAheadIsEstimate": false, "position": 1, "running": 0, "runningIsEstimate": false, "note": "Scans are asynchronous and may take time to complete." } }`.
- Los trabajos de análisis son asíncronos. Las solicitudes de análisis manuales tienen prioridad sobre el trabajo normal de publicación y procesamiento pendiente, pero su finalización sigue dependiendo de la disponibilidad de los trabajadores.

### `GET /api/v1/skills/-/scan/{scanId}`

Endpoint autenticado de consulta para un análisis enviado.

- Devuelve el estado en cola, en ejecución, completado correctamente o fallido.
- Devuelve `queue.queuedAhead` y `queue.position` mientras está en cola para que los clientes puedan mostrar cuántos análisis manuales prioritarios preceden a la solicitud. Las colas muy grandes están limitadas y se notifican mediante `queuedAheadIsEstimate: true`.
- Cuando está disponible, `report` contiene las secciones `clawscan`, `skillspector`, `staticAnalysis` y `virustotal`.
- Los trabajos de análisis fallidos devuelven `status: "failed"` con `lastError`.

### `GET /api/v1/skills/-/scan/{scanId}/download`

Endpoint autenticado del archivo de informes.

- Requiere un análisis completado correctamente; los análisis que no han alcanzado un estado terminal devuelven `409`.
- Devuelve un archivo ZIP con `manifest.json`, `clawscan.json`, `skillspector.json`, `static-analysis.json`, `virustotal.json` y `README.md`.

### `GET /api/v1/skills/-/scan/download/{name}?version=<version>&kind=skill|plugin`

Endpoint autenticado del archivo de informes almacenados para las versiones enviadas.

- Requiere acceso de administración del propietario o publicador de la Skill o el Plugin, o autoridad de moderador o administrador de la plataforma.
- Devuelve los resultados de análisis almacenados para la versión exacta enviada, incluidas las versiones bloqueadas u ocultas.
- `kind` utiliza `skill` de forma predeterminada; use `kind=plugin` para los análisis de Plugins o paquetes.
- Devuelve la misma estructura ZIP que las descargas de solicitudes de análisis.

### `POST /api/v1/skills/-/scan/batch`

Ruta canónica de repetición de análisis por lotes exclusiva para administradores. Acepta la misma estructura de carga que la ruta heredada `POST /api/v1/skills/-/rescan-batch`.

### `POST /api/v1/skills/-/scan/batch/status`

Ruta canónica de estado de lotes exclusiva para administradores. Acepta `{ "jobIds": ["..."] }` y devuelve los mismos contadores agregados que la ruta heredada `POST /api/v1/skills/-/rescan-batch/status`.

### `GET /api/v1/skills/{slug}/verify`

Devuelve el contenedor de verificación de la tarjeta de Skill utilizado por `clawhub skill verify`.

Parámetros de consulta:

- `version` (opcional): cadena de versión específica.
- `tag` (opcional): resuelve una versión etiquetada (por ejemplo, `latest`).

Notas:

- `ok` es `true` solo cuando la versión seleccionada tiene una tarjeta de Skill generada, la moderación no la ha bloqueado por malware y la verificación de ClawScan no detecta problemas.
- La identidad de la Skill, la identidad del publicador y los metadatos de la versión seleccionada son campos de nivel superior del contenedor (`slug`, `displayName`, `publisherHandle`, `version`, `resolvedFrom`, `tag`, `createdAt`) para que la automatización de shell pueda leerlos sin desempaquetar contenedores anidados.
- `security` es el veredicto de nivel superior de ClawScan o seguridad. La automatización debe basarse en `ok`, `decision`, `reasons` y `security.status`.
- `security.signals` contiene pruebas complementarias de los analizadores, como `staticScan`, `virusTotal` y `skillSpector`.
- `security.signals.dependencyRegistry` se conserva para mantener la compatibilidad con las respuestas de v1, pero el analizador de existencia en el registro de dependencias se ha retirado y esta clave siempre es `null`.
- `provenance` es `server-resolved-github-import` solo cuando ClawHub resolvió y almacenó un repositorio, una referencia, un commit y una ruta de GitHub durante la publicación o importación; de lo contrario, es `unavailable`.

### `POST /api/v1/skills/-/security-verdicts`

Devuelve los veredictos compactos de seguridad actuales de versiones exactas de Skills. Este
endpoint de colección está destinado a clientes que ya saben qué versiones instaladas
de Skills de ClawHub necesitan mostrar, como la interfaz de control de OpenClaw.

Solicitud:

```json
{
  "items": [{ "slug": "gifgrep", "version": "1.2.3" }]
}
```

Notas:

- `items` debe contener entre 1 y 100 pares únicos de `{ slug, version }`.
- Los resultados corresponden a cada elemento; la ausencia de una Skill o versión no hace que falle toda la respuesta.
- La respuesta solo contiene información de seguridad. No incluye datos de la tarjeta de Skill, el estado de la tarjeta generada, listas de archivos de artefactos ni cargas detalladas de los analizadores.
- `security.signals` solo contiene pruebas complementarias relativas al estado; use `/scan` o la página de auditoría de seguridad de ClawHub para consultar todos los detalles de los analizadores.
- `security.signals.dependencyRegistry` se conserva para mantener la compatibilidad con las respuestas de v1, pero el analizador de existencia en el registro de dependencias se ha retirado y esta clave siempre es `null`.
- La ausencia de una tarjeta de Skill no afecta a `ok`, `decision` ni `reasons` en este endpoint; los clientes deben leer localmente la `skill-card.md` instalada cuando necesiten el contenido de la tarjeta.
- Use `/verify` cuando necesite el contenedor de verificación de la tarjeta de Skill de una sola Skill, `/card` cuando necesite el Markdown de la tarjeta generada y `/scan` cuando necesite datos detallados de los analizadores.

Respuesta:

```json
{
  "schema": "clawhub.skill.security-verdicts.v1",
  "items": [
    {
      "ok": true,
      "decision": "pass",
      "reasons": [],
      "requestedSlug": "gifgrep",
      "slug": "gifgrep",
      "displayName": "GifGrep",
      "publisherHandle": "steipete",
      "publisherDisplayName": "Peter",
      "requestedVersion": "1.2.3",
      "version": "1.2.3",
      "createdAt": 0,
      "checkedAt": 0,
      "skillUrl": "https://clawhub.ai/steipete/skills/gifgrep",
      "securityAuditUrl": "https://clawhub.ai/steipete/skills/gifgrep/security-audit?version=1.2.3",
      "security": {
        "status": "clean",
        "passed": true,
        "signals": {
          "staticScan": { "status": "clean", "reasonCodes": [] },
          "virusTotal": null,
          "skillSpector": null,
          "dependencyRegistry": null
        }
      }
    },
    {
      "ok": false,
      "decision": "fail",
      "reasons": ["version.not_found"],
      "requestedSlug": "missing-version",
      "requestedVersion": "1.0.0",
      "error": { "code": "version_not_found", "message": "Versión no encontrada" },
      "security": null
    }
  ]
}
```

### `GET /api/v1/skills/{slug}/file`

Devuelve los bytes exactos del archivo almacenado como descarga. Añada `preview=1` para solicitar una vista previa
de texto escapado y con tamaño limitado; se puede obtener una vista previa de cualquier archivo con bytes UTF-8 válidos,
independientemente de su extensión o sus metadatos MIME.

Parámetros de consulta:

- `path` (obligatorio)
- `version` (opcional)
- `tag` (opcional)
- `preview=1` (opcional; devuelve `text/plain` o `415` cuando los bytes no son UTF-8 válidos)

Notas:

- Utiliza de forma predeterminada la versión más reciente.
- Límite de descarga sin procesar: 10MB.
- Límite de vista previa de texto: 200KB.

### `GET /api/v1/packages`

Endpoint de catálogo unificado para:

- Skills
- Plugins de código
- Plugins de paquete

Parámetros de consulta:

- `limit` (opcional): entero (1–100)
- `cursor` (opcional): cursor de paginación
- `family` (opcional): `skill`, `code-plugin` o `bundle-plugin`
- `channel` (opcional): `official`, `community` o `private`
- `isOfficial` (opcional): `true` o `false`
- `sort` (opcional): `updated` (predeterminado), `recommended`, `trending`, `downloads`, alias heredado `installs`
- `category` (opcional): filtro por categoría de Plugin. Solo se admite cuando la
  solicitud está limitada a paquetes de Plugins (`/api/v1/plugins`,
  `/api/v1/code-plugins`, `/api/v1/bundle-plugins` o endpoints de paquetes con
  `family=code-plugin`/`family=bundle-plugin`). Las categorías controladas y
  los alias de filtros heredados de v1 se documentan en `GET /api/v1/plugins`.

Notas:

- Los valores no válidos de `family`, `channel`, `isOfficial`, `featured`,
  `highlightedOnly` o `sort` devuelven `400`. Los parámetros de consulta desconocidos se ignoran.
- `GET /api/v1/code-plugins` y `GET /api/v1/bundle-plugins` siguen siendo alias de familia fija.
- Las entradas de Skills siguen respaldadas por el registro de Skills y solo pueden publicarse mediante `POST /api/v1/skills`.
- `POST /api/v1/packages` sigue siendo exclusivo para versiones de Plugins de código y Plugins de paquete.
- Las llamadas anónimas solo pueden ver los canales públicos de paquetes.
- Las llamadas autenticadas pueden ver en los resultados de listas y búsquedas los paquetes privados de los publicadores a los que pertenecen.
- `channel=private` solo devuelve los paquetes que la llamada autenticada puede leer.

### `GET /api/v1/packages/search`

Búsqueda unificada en el catálogo de Skills y paquetes de Plugins.

Parámetros de consulta:

- `q` (obligatorio): cadena de consulta
- `limit` (opcional): entero (1–100)
- `family` (opcional): `skill`, `code-plugin` o `bundle-plugin`
- `channel` (opcional): `official`, `community` o `private`
- `isOfficial` (opcional): `true` o `false`
- `category` (opcional): filtro de categoría de plugins. Solo se admite cuando la
  solicitud se limita a paquetes de plugins. Las categorías controladas y los alias
  de filtro heredados de v1 se documentan en `GET /api/v1/plugins`.

Notas:

- Los valores no válidos de `family`, `channel`, `isOfficial`, `featured` o
  `highlightedOnly` devuelven `400`. Los parámetros de consulta desconocidos se ignoran.
- Los solicitantes anónimos solo ven los canales de paquetes públicos.
- Los solicitantes autenticados pueden buscar paquetes privados de los publicadores a los que pertenecen.
- `channel=private` solo devuelve los paquetes que el solicitante autenticado puede leer.

### `GET /api/v1/plugins`

Exploración del catálogo solo para plugins en paquetes de plugins de código y plugins de paquete.

Parámetros de consulta:

- `limit` (opcional): entero (1-100)
- `cursor` (opcional): cursor de paginación
- `isOfficial` (opcional): `true` o `false`
- `sort` (opcional): `recommended` (predeterminado), `trending`, `downloads`, `updated`, alias heredado `installs`
- `category` (opcional): filtro de categoría de plugins. Valores actuales:
  `channels`, `models`, `memory`, `context`, `voice`, `media`, `web`,
  `tools`, `runtime`, `gateway`, `security`, `other`.

Los alias de filtro heredados de v1 siguen aceptándose en los endpoints de lectura:

- `mcp-tooling`, `data` y `automation` se resuelven como `tools`.
- `observability` y `deployment` se resuelven como `gateway`.
- `dev-tools` se resuelve como `runtime`.

`trending` es una clasificación de instalaciones/descargas de siete días y no utiliza totales históricos.
En el endpoint unificado `/api/v1/packages` se limita a plugins; use
`/api/v1/skills?sort=trending` para el catálogo de Skills.

Los alias heredados no se aceptan como valores de categoría almacenados o declarados por el autor.

### `GET /api/v1/skills/export`

Exportación masiva de las Skills públicas más recientes para análisis sin conexión.

Autenticación:

- Se requiere un token de API.

Parámetros de consulta:

- `startDate` (obligatorio): límite inferior en milisegundos Unix para `updatedAt` de la Skill.
- `endDate` (obligatorio): límite superior en milisegundos Unix para `updatedAt` de la Skill.
- `limit` (opcional): entero (1-250), valor predeterminado `250`.
- `cursor` (opcional): cursor de paginación de la respuesta anterior.

Respuesta:

- Cuerpo: archivo ZIP.
- Cada Skill exportada tiene como raíz `{publisher}/{slug}/`.
- Las Skills alojadas incluyen los archivos de la versión almacenada más reciente y se enumeran en
  `_manifest.json` con `sourceRef: "public-clawhub"`.
- Las Skills actuales respaldadas por GitHub con un análisis `clean` o `suspicious` incluyen
  `_source_handoff.json` con `sourceRef: "public-github"`, repositorio, confirmación, ruta,
  hash de contenido y URL del archivo. No incluyen archivos fuente alojados en ClawHub.
- Cada Skill incluye `_export_skill_meta.json`.
- `_manifest.json` siempre se incluye en la raíz del ZIP.
- `_errors.json` se incluye cuando no se pudieron exportar Skills o archivos
  individuales.

Encabezados:

- `X-Next-Cursor`
- `X-Has-More`
- `X-Total-Returned`
- `X-Date-Range`
- `X-Export-Errors`

### `GET /api/v1/plugins/export`

Exportación masiva de las versiones públicas más recientes de plugins para análisis sin conexión.

Autenticación:

- Se requiere un token de API.

Parámetros de consulta:

- `startDate` (obligatorio): límite inferior en milisegundos Unix para `updatedAt` del plugin.
- `endDate` (obligatorio): límite superior en milisegundos Unix para `updatedAt` del plugin.
- `limit` (opcional): entero (1-250), valor predeterminado `250`.
- `cursor` (opcional): cursor de paginación de la respuesta anterior.
- `family` (opcional): `code-plugin` o `bundle-plugin`. Si se omite, incluye ambas
  familias de plugins.

Respuesta:

- Cuerpo: archivo ZIP.
- Cada plugin exportado tiene como raíz `{family}/{packageName}/`.
- Cada plugin exportado incluye los archivos almacenados de la versión más reciente.
- Los metadatos de exportación de cada plugin se almacenan en
  `__clawhub_export/{family}/{packageName}/plugin_meta.json`.
- `_manifest.json` siempre se incluye en la raíz del ZIP.
- `_errors.json` se incluye cuando no se pudieron exportar plugins o archivos
  individuales.

Encabezados:

- `X-Next-Cursor`
- `X-Has-More`
- `X-Total-Returned`
- `X-Date-Range`
- `X-Export-Errors`

### `GET /api/v1/plugins/search`

Búsqueda solo de plugins en paquetes de plugins de código y plugins de paquete.

Parámetros de consulta:

- `q` (obligatorio): cadena de consulta
- `limit` (opcional): entero (1-100)
- `isOfficial` (opcional): `true` o `false`
- `category` (opcional): filtro de categoría de plugins. Valores actuales:
  `channels`, `models`, `memory`, `context`, `voice`, `media`, `web`,
  `tools`, `runtime`, `gateway`, `security`, `other`.

Notas:

- También se aceptan los alias de filtro heredados de v1 documentados en `GET /api/v1/plugins`.
- El filtrado por categoría es un filtro real de la API respaldado por filas de resumen
  de categorías de plugins, no una reescritura de la consulta de búsqueda.
- Los resultados se devuelven por orden de relevancia y actualmente no se paginan.
- Los controles de ordenación de la interfaz del navegador para la búsqueda de plugins reordenan los resultados de relevancia cargados,
  de acuerdo con el comportamiento de exploración actual de `/skills`.

### `GET /api/v1/packages/{name}`

Devuelve los metadatos detallados del paquete.

Notas:

- Las Skills también pueden resolverse mediante esta ruta en el catálogo unificado.
- Los paquetes privados devuelven `404` a menos que el solicitante pueda leer el publicador propietario.

### `DELETE /api/v1/packages/{name}`

Elimina de forma reversible un paquete y todas sus versiones.

Notas:

- Requiere un token de API del propietario del paquete, un propietario/administrador de la organización publicadora,
  un moderador de la plataforma o un administrador de la plataforma.

### `GET /api/v1/packages/{name}/versions`

Devuelve el historial de versiones.

Parámetros de consulta:

- `limit` (opcional): entero (1–100)
- `cursor` (opcional): cursor de paginación

Notas:

- Los paquetes privados devuelven `404` a menos que el solicitante pueda leer el publicador propietario.

### `GET /api/v1/packages/{name}/versions/{version}`

Devuelve una versión del paquete, incluidos los metadatos de archivos, la compatibilidad,
la verificación, los metadatos del artefacto y los datos del análisis.

Notas:

- `version.artifact.kind` es `legacy-zip` para los archivos de paquetes del sistema anterior o
  `npm-pack` para las versiones respaldadas por ClawPack.
- Las versiones de ClawPack incluyen los campos compatibles con npm `npmIntegrity`, `npmShasum` y
  `npmTarballName`.
- `version.sha256hash` son metadatos de compatibilidad obsoletos para clientes antiguos. Generan
  el hash de los bytes ZIP exactos devueltos por `/api/v1/packages/{name}/download`.
  Los clientes modernos deben usar `version.artifact.sha256`, que identifica el
  artefacto canónico de la versión.
- `version.vtAnalysis`, `version.llmAnalysis` y `version.staticScan` se
  incluyen cuando existen datos del análisis.
- Los paquetes privados devuelven `404` a menos que el solicitante pueda leer el publicador propietario.

### `GET /api/v1/packages/{name}/versions/{version}/security`

Devuelve el resumen exacto de seguridad y confianza de la versión del paquete para los
clientes de instalación. Esta es la superficie pública de consumo de OpenClaw para decidir si
se puede instalar una versión resuelta.

Autenticación:

- Endpoint público de lectura. No se requiere ningún token de propietario, publicador,
  moderador ni administrador.

Respuesta:

```json
{
  "package": {
    "name": "@openclaw/example-plugin",
    "displayName": "Plugin de ejemplo",
    "family": "code-plugin"
  },
  "release": {
    "releaseId": "packageReleases:...",
    "version": "1.2.3",
    "artifactKind": "npm-pack",
    "artifactSha256": "0123456789abcdef...",
    "npmIntegrity": "sha512-...",
    "npmShasum": "0123456789abcdef0123456789abcdef01234567",
    "npmTarballName": "example-plugin-1.2.3.tgz",
    "createdAt": 1730000000000
  },
  "trust": {
    "scanStatus": "malicious",
    "moderationState": "quarantined",
    "blockedFromDownload": true,
    "reasons": ["manual:quarantined", "scan:malicious"],
    "pending": false,
    "stale": false
  }
}
```

Campos de respuesta:

- `package.name`, `package.displayName` y `package.family` identifican el
  paquete resuelto del registro.
- `release.releaseId`, `release.version` y `release.createdAt` identifican la
  versión exacta que se evaluó.
- `release.artifactKind`, `release.artifactSha256`, `release.npmIntegrity`,
  `release.npmShasum` y `release.npmTarballName` están presentes cuando se conocen
  para el artefacto de la versión.
- `trust.scanStatus` es el estado de confianza efectivo derivado de los datos del analizador
  y la moderación manual de la versión.
- `trust.moderationState` admite valores nulos. Es `null` cuando no existe moderación manual
  de la versión.
- `trust.blockedFromDownload` es la señal de bloqueo de instalación. OpenClaw y otros
  clientes de instalación deben bloquearla cuando este valor sea `true`, en lugar de
  volver a derivar las reglas de bloqueo a partir de los campos del analizador o de moderación.
- `trust.reasons` es la lista de explicaciones para el usuario y de auditoría. Los códigos de motivo
  son cadenas compactas y estables, como `manual:quarantined`, `scan:malicious`
  y `package:malicious`.
- `trust.pending` significa que una o más entradas de confianza aún están pendientes de completarse.
- `trust.stale` significa que el resumen de confianza se calculó a partir de entradas obsoletas y
  debe considerarse que requiere una actualización antes de tomar una decisión de autorización con un alto grado de confianza.

Notas:

- Este endpoint es específico de la versión. Los clientes deben llamarlo después de resolver la
  versión del paquete que pretenden instalar, no solo después de leer los metadatos
  más recientes del paquete.
- Los paquetes privados devuelven `404` a menos que el solicitante pueda leer el publicador propietario.
- Este endpoint es deliberadamente más limitado que los endpoints de moderación
  de propietarios/moderadores. Expone la decisión de instalación y la explicación pública, pero no
  las identidades de los denunciantes, el contenido de las denuncias, las pruebas privadas ni los plazos
  internos de revisión.

### `GET /api/v1/packages/{name}/versions/{version}/artifact`

Devuelve los metadatos explícitos del solucionador de artefactos para una versión del paquete.

Notas:

- Las versiones heredadas de paquetes devuelven un artefacto `legacy-zip` y un
  `downloadUrl` ZIP heredado.
- Las versiones de ClawPack devuelven un artefacto `npm-pack`, campos de integridad de npm, un
  `tarballUrl` y la URL de compatibilidad ZIP heredada.
- Esta es la superficie del solucionador de OpenClaw; evita deducir el formato del archivo a partir
  de una URL compartida.

### `GET /api/v1/packages/{name}/versions/{version}/artifact/download`

Descarga el artefacto de la versión mediante la ruta explícita del solucionador.

Notas:

- Las versiones de ClawPack transmiten exactamente los bytes `.tgz` del paquete npm subido.
- Las versiones ZIP heredadas redirigen a `/api/v1/packages/{name}/download?version=`.
- Usa el límite de frecuencia de descargas.

### `GET /api/v1/packages/{name}/readiness`

Devuelve la preparación calculada para el consumo futuro de OpenClaw.

Las comprobaciones de preparación abarcan:

- estado del canal oficial
- disponibilidad de la versión más reciente
- disponibilidad del artefacto npm-pack de ClawPack
- resumen del artefacto
- procedencia del repositorio de origen y del commit
- metadatos de compatibilidad con OpenClaw
- destinos de host
- estado del análisis

Respuesta:

```json
{
  "package": {
    "name": "@openclaw/example-plugin",
    "displayName": "Plugin de ejemplo",
    "family": "code-plugin",
    "isOfficial": true,
    "latestVersion": "1.2.3"
  },
  "ready": false,
  "checks": [
    {
      "id": "clawpack",
      "label": "Artefacto de ClawPack",
      "status": "fail",
      "message": "La versión más reciente solo está disponible como ZIP heredado."
    }
  ],
  "blockers": ["clawpack"]
}
```

### `GET /api/v1/packages/migrations`

Endpoint para moderadores que permite enumerar las filas de migración de plugins oficiales de OpenClaw.

Autenticación:

- Requiere un token de API de un usuario moderador o administrador.

Parámetros de consulta:

- `phase` (opcional): `planned`, `published`, `clawpack-ready`,
  `legacy-zip-only`, `metadata-ready`, `blocked`, `ready-for-openclaw` o
  `all` (valor predeterminado).
- `limit` (opcional): entero (1-100)
- `cursor` (opcional): cursor de paginación

Respuesta:

```json
{
  "items": [
    {
      "migrationId": "officialPluginMigrations:...",
      "bundledPluginId": "core.search",
      "packageName": "@openclaw/search-plugin",
      "packageId": "packages:...",
      "owner": "platform",
      "sourceRepo": "openclaw/openclaw",
      "sourcePath": "plugins/search",
      "sourceCommit": "abc123",
      "phase": "blocked",
      "blockers": ["ClawPack ausente"],
      "hostTargetsComplete": true,
      "scanClean": false,
      "moderationApproved": false,
      "runtimeBundlesReady": false,
      "notes": null,
      "createdAt": 1760000000000,
      "updatedAt": 1760000000000
    }
  ],
  "nextCursor": null,
  "done": true
}
```

### `POST /api/v1/packages/migrations`

Endpoint para administradores que permite crear o actualizar una fila de migración de un plugin oficial.

Autenticación:

- Requiere un token de API de un usuario administrador.

Cuerpo de la solicitud:

```json
{
  "bundledPluginId": "core.search",
  "packageName": "@openclaw/search-plugin",
  "owner": "platform",
  "sourceRepo": "openclaw/openclaw",
  "sourcePath": "plugins/search",
  "sourceCommit": "abc123",
  "phase": "blocked",
  "blockers": ["ClawPack ausente"],
  "hostTargetsComplete": true,
  "scanClean": false,
  "moderationApproved": false,
  "runtimeBundlesReady": false,
  "notes": "a la espera de la carga del publicador"
}
```

Notas:

- `bundledPluginId` se normaliza a minúsculas y es la clave estable de inserción o actualización.
- `packageName` se normaliza como nombre de npm; el paquete puede estar ausente en las migraciones
  planificadas.
- Esto solo registra la preparación de la migración. No modifica OpenClaw ni genera
  ClawPacks.

### `GET /api/v1/packages/moderation/queue`

Endpoint para moderadores y administradores destinado a las colas de revisión de versiones de paquetes.

Autenticación:

- Requiere un token de API de un usuario moderador o administrador.

Parámetros de consulta:

- `status` (opcional): `open` (valor predeterminado), `blocked`, `manual` o `all`
- `limit` (opcional): entero (1-100)
- `cursor` (opcional): cursor de paginación

Significados de los estados:

- `open`: versiones sospechosas, maliciosas, pendientes, en cuarentena, revocadas o denunciadas.
- `blocked`: versiones en cuarentena, revocadas o maliciosas.
- `manual`: cualquier versión con una anulación manual de moderación.
- `all`: cualquier versión con una anulación manual, un estado de análisis no limpio o una denuncia del paquete.

Respuesta:

```json
{
  "items": [
    {
      "packageId": "packages:...",
      "releaseId": "packageReleases:...",
      "name": "@openclaw/example-plugin",
      "displayName": "Plugin de ejemplo",
      "family": "code-plugin",
      "channel": "community",
      "isOfficial": false,
      "version": "1.2.3",
      "createdAt": 1730000000000,
      "artifactKind": "npm-pack",
      "scanStatus": "malicious",
      "moderationState": "quarantined",
      "moderationReason": "revisión manual",
      "sourceRepo": "openclaw/example-plugin",
      "sourceCommit": "abc123",
      "reportCount": 2,
      "lastReportedAt": 1730000001000,
      "reasons": ["manual:quarantined", "scan:malicious", "reports:2"]
    }
  ],
  "nextCursor": null,
  "done": true
}
```

### `POST /api/v1/packages/{name}/report`

Denuncia un paquete para que lo revise un moderador. Las denuncias corresponden al paquete y pueden
estar vinculadas opcionalmente a una versión. Se incorporan a la cola de moderación, pero por sí solas no ocultan
ni bloquean automáticamente las descargas; los moderadores deben usar la moderación de versiones para
aprobar, poner en cuarentena o revocar artefactos.

Autenticación:

- Requiere un token de API.

Solicitud:

```json
{ "reason": "Binario nativo sospechoso", "version": "1.2.3" }
```

Respuesta:

```json
{
  "ok": true,
  "reported": true,
  "alreadyReported": false,
  "packageId": "packages:...",
  "releaseId": "packageReleases:...",
  "reportCount": 1
}
```

### `GET /api/v1/packages/reports`

Endpoint para moderadores y administradores destinado a la recepción de denuncias de paquetes.

Autenticación:

- Requiere un token de API de un usuario moderador o administrador.

Parámetros de consulta:

- `status` (opcional): `open` (valor predeterminado), `confirmed`, `dismissed` o `all`
- `limit` (opcional): entero (1-100)
- `cursor` (opcional): cursor de paginación

Respuesta:

```json
{
  "items": [
    {
      "reportId": "packageReports:...",
      "packageId": "packages:...",
      "releaseId": "packageReleases:...",
      "name": "@openclaw/example-plugin",
      "displayName": "Plugin de ejemplo",
      "family": "code-plugin",
      "version": "1.2.3",
      "reason": "Binario nativo sospechoso",
      "status": "open",
      "createdAt": 1730000000000,
      "reporter": {
        "userId": "users:...",
        "handle": "reporter",
        "displayName": "Denunciante"
      },
      "triagedAt": null,
      "triagedBy": null,
      "triageNote": null
    }
  ],
  "nextCursor": null,
  "done": true
}
```

### `GET /api/v1/packages/{name}/moderation`

Endpoint para propietarios y moderadores destinado a la visibilidad de la moderación de paquetes.

Autenticación:

- Requiere un token de API del propietario del paquete, un miembro del publicador, un moderador o
  un usuario administrador.

Respuesta:

```json
{
  "package": {
    "packageId": "packages:...",
    "name": "@openclaw/example-plugin",
    "displayName": "Plugin de ejemplo",
    "family": "code-plugin",
    "channel": "community",
    "isOfficial": false,
    "reportCount": 2,
    "lastReportedAt": 1730000001000,
    "scanStatus": "malicious"
  },
  "latestRelease": {
    "releaseId": "packageReleases:...",
    "version": "1.2.3",
    "artifactKind": "npm-pack",
    "scanStatus": "malicious",
    "moderationState": "quarantined",
    "moderationReason": "revisión manual",
    "blockedFromDownload": true,
    "reasons": ["manual:quarantined", "scan:malicious", "reports:2"],
    "createdAt": 1730000000000
  }
}
```

### `POST /api/v1/packages/reports/{reportId}/triage`

Endpoint para moderadores y administradores que permite resolver o reabrir denuncias de paquetes.

Solicitud:

```json
{
  "status": "confirmed",
  "note": "Se revisó y puso en cuarentena la versión afectada.",
  "finalAction": "quarantine"
}
```

`note` es obligatorio para `confirmed` y `dismissed`; puede omitirse al
volver a establecer `status` en `open`. Pase `finalAction: "quarantine"` o
`finalAction: "revoke"` con una denuncia confirmada para aplicar la moderación de la versión en el
mismo flujo de trabajo auditable.

Respuesta:

```json
{
  "ok": true,
  "reportId": "packageReports:...",
  "packageId": "packages:...",
  "status": "confirmed",
  "reportCount": 0
}
```

### `POST /api/v1/packages/{name}/versions/{version}/moderation`

Endpoint para moderadores y administradores destinado a la revisión de versiones de paquetes.

Solicitud:

```json
{ "state": "quarantined", "reason": "Carga nativa sospechosa." }
```

Estados compatibles:

- `approved`: revisada manualmente y permitida.
- `quarantined`: bloqueada a la espera de seguimiento.
- `revoked`: bloqueada después de que una versión se considerara fiable anteriormente.

Las versiones en cuarentena y revocadas devuelven `403` desde las rutas de descarga de artefactos.
Cada cambio escribe una entrada en el registro de auditoría.

### `GET /api/v1/packages/{name}/file`

Devuelve como descarga los bytes exactos del archivo almacenado del paquete. Añada `preview=1` para solicitar la misma vista previa
de texto UTF-8 limitada que se usa para los archivos de Skills.

Parámetros de consulta:

- `path` (obligatorio)
- `version` (opcional)
- `tag` (opcional)
- `preview=1` (opcional; devuelve `text/plain` o `415` cuando los bytes no son UTF-8 válido)

Notas:

- El valor predeterminado es la versión más reciente.
- Usa el límite de frecuencia de lectura, no el de descargas.
- Límite de descarga sin procesar: 10MB.
- Límite de la vista previa de texto: 200KB; los archivos opacos devuelven `415` solo para las solicitudes de vista previa.
- Los análisis pendientes de VirusTotal no bloquean las lecturas; las versiones maliciosas pueden seguir reteniéndose en otros lugares.
- Los paquetes privados devuelven `404` salvo que el solicitante pueda leer el publicador propietario.

### `GET /api/v1/packages/{name}/download`

Descarga el archivo ZIP determinista heredado de una versión del paquete.

Parámetros de consulta:

- `version` (opcional)
- `tag` (opcional)

Notas:

- El valor predeterminado es la versión más reciente.
- Skills redirige a `GET /api/v1/download`.
- Los archivos de plugins y paquetes son archivos zip con una raíz `package/` para que los clientes antiguos de OpenClaw
  sigan funcionando.
- Esta ruta permanece exclusivamente en formato ZIP. No transmite archivos `.tgz` de ClawPack.
- Las respuestas incluyen las cabeceras `ETag`, `Digest`, `X-ClawHub-Artifact-Type` y
  `X-ClawHub-Artifact-Sha256` para las comprobaciones de integridad del resolvedor.
- Los metadatos exclusivos del registro no se insertan en el archivo descargado.
- Los análisis pendientes de VirusTotal no bloquean las descargas; las versiones maliciosas devuelven `403`.
- Los paquetes privados devuelven `404` salvo que el solicitante sea el propietario.

### `GET /api/npm/{package}`

Devuelve un packument compatible con npm para las versiones de paquetes respaldadas por ClawPack.

Notas:

- Solo se enumeran las versiones que tienen tarballs npm-pack de ClawPack subidos.
- Las versiones heredadas disponibles únicamente como ZIP se omiten intencionadamente.
- `dist.tarball`, `dist.integrity` y `dist.shasum` usan campos compatibles con
  npm para que los usuarios puedan dirigir npm al espejo si así lo desean.
- Los packuments de paquetes con ámbito admiten tanto `/api/npm/@scope/name` como la ruta de solicitud
  codificada `/api/npm/@scope%2Fname` de npm.

### `GET /api/npm/{package}/-/{tarball}.tgz`

Transmite exactamente los bytes del tarball de ClawPack subido para los clientes del espejo de npm.

Notas:

- Usa el límite de frecuencia de descargas.
- Las cabeceras de descarga incluyen el SHA-256 de ClawHub, además de los metadatos de integridad y shasum de npm.
- Las comprobaciones de moderación y de acceso a paquetes privados siguen aplicándose.

### `GET /api/v1/resolve`

La CLI lo usa para asignar una huella digital local a una versión conocida.

Parámetros de consulta:

- `slug` (obligatorio)
- `hash` (obligatorio): sha256 hexadecimal de 64 caracteres de la huella digital del paquete

Respuesta:

```json
{ "slug": "gifgrep", "match": { "version": "1.2.2" }, "latestVersion": { "version": "1.2.3" } }
```

### `GET /api/v1/download`

Descarga un ZIP de una versión alojada de una skill o devuelve una transferencia a la fuente de GitHub para una
skill actual respaldada por GitHub con un análisis `clean` o `suspicious` y sin una versión
alojada.

Parámetros de consulta:

- `slug` (obligatorio)
- `version` (opcional): cadena semver
- `tag` (opcional): nombre de etiqueta (p. ej., `latest`)

Notas:

- Si no se proporciona `version` ni `tag`, se utiliza la versión más reciente.
- Las versiones eliminadas de forma lógica devuelven `410`.
- Las transferencias de skills respaldadas por GitHub no actúan como proxy ni replican bytes. La respuesta JSON
  incluye `sourceRef: "public-github"`, `repo`, `commit`, `path`, `contentHash`
  y `archiveUrl`; el análisis y el estado actual constituyen una condición de acceso y no se incluyen como metadatos
  de la carga útil de éxito.
- Las estadísticas de descarga se cuentan como identidades únicas por día UTC (`userId` cuando el token de API es válido; de lo contrario, la IP).

## Endpoints de autenticación (token Bearer)

Todos los endpoints requieren:

```
Authorization: Bearer clh_...
```

### `GET /api/v1/whoami`

Valida el token y devuelve el identificador del usuario.

### `POST /api/v1/skills`

Publica una versión nueva.

- Opción preferida: `multipart/form-data` con JSON `payload` y blobs `files[]`.
- También se acepta un cuerpo JSON con `files` (basado en storageId).
- Campo opcional de la carga útil: `ownerHandle`. Cuando está presente, la API resuelve ese
  publicador en el servidor y exige que el actor tenga acceso de publicador.
- Campo opcional de la carga útil: `migrateOwner`. Cuando `true` tiene el valor `ownerHandle`, una
  skill existente puede transferirse a ese propietario si el actor es administrador o propietario tanto del publicador
  actual como del publicador de destino. Sin esta aceptación explícita, se rechazan los cambios de
  propietario.

### `POST /api/v1/packages`

Publica una versión de un plugin de código o un plugin de paquete.

- Requiere autenticación mediante token Bearer.
- Requiere `multipart/form-data`.
- Los campos de formulario permitidos son `payload`, blobs `files` repetidos o una referencia a un único
  tarball `clawpack`. `clawpack` puede ser un blob `.tgz` o un identificador de almacenamiento devuelto por
  el flujo de URL de carga. Las publicaciones preparadas mediante identificador de almacenamiento también deben incluir el
  `clawpackUploadTicket` devuelto con esa URL de carga.
- Utilice `files` o `clawpack`, nunca ambos en la misma solicitud.
- Se rechazan los cuerpos JSON y los metadatos `payload.files` / `payload.artifact`
  proporcionados por el invocador.
- Las solicitudes directas de publicación multipart están limitadas a 18MB. Los tarballs de ClawPack pueden
  utilizar el flujo de URL de carga hasta el límite de 120MB por tarball.
- Campo opcional de la carga útil: `ownerHandle`. Cuando está presente, solo los administradores pueden publicar en nombre de ese propietario.

Aspectos destacados de la validación:

- `family` debe ser `code-plugin` o `bundle-plugin`.
- Los paquetes de plugins requieren `openclaw.plugin.json`. Las cargas `.tgz` de ClawPack deben
  contenerlo en `package/openclaw.plugin.json`.
- Los plugins de código requieren `package.json`, metadatos del repositorio fuente, metadatos del commit
  fuente, metadatos del esquema de configuración, `openclaw.compat.pluginApi` y
  `openclaw.build.openclawVersion`.
- `openclaw.hostTargets` y `openclaw.environment` son metadatos opcionales.
- Solo el publicador de la organización `openclaw` y los publicadores personales de los miembros actuales de la organización `openclaw`
  pueden publicar en el canal `official`.
- Las publicaciones en nombre de terceros siguen validando la aptitud para el canal oficial con respecto a la cuenta del propietario de destino.

### `DELETE /api/v1/skills/{slug}` / `POST /api/v1/skills/{slug}/undelete`

Elimina de forma lógica o restaura una skill (propietario, moderador o administrador).

Cuerpo JSON opcional:

```json
{ "reason": "Retenida para moderación a la espera de una revisión legal." }
```

Cuando está presente, `reason` se almacena como nota de moderación de la skill y se copia en el registro de auditoría.
Las eliminaciones lógicas iniciadas por el propietario reservan el slug durante 30 días; después, otro
publicador puede reclamarlo. La respuesta de eliminación incluye `slugReservedUntil` cuando se aplica este vencimiento.
Las ocultaciones realizadas por moderadores o administradores y las eliminaciones por seguridad no vencen de esta manera.

Respuesta de eliminación:

```json
{ "ok": true, "slugReservedUntil": 1730000000000 }
```

Códigos de estado:

- `200`: correcto
- `401`: no autorizado
- `403`: prohibido
- `404`: skill o usuario no encontrado
- `500`: error interno del servidor

### `POST /api/v1/users/publisher`

Solo para administradores. Garantiza que exista un publicador de organización para un identificador. Si el identificador aún apunta a un
usuario compartido o publicador personal heredado, el endpoint lo migra primero a un publicador de organización.
Para una organización recién creada, proporcione `memberHandle`; el administrador que realiza la acción no se añade como miembro.
`memberRole` tiene como valor predeterminado `owner`.

- Cuerpo: `{ "handle": "openclaw", "displayName": "OpenClaw", "memberHandle": "alice", "memberRole": "owner", "trusted": true }`
- Respuesta: `{ "ok": true, "publisherId": "...", "handle": "openclaw", "created": true, "migrated": false, "trusted": true, "member": { "userId": "...", "handle": "alice", "role": "owner" } }`

### `POST /api/v1/publishers`

Creación autenticada y autoservicio de un publicador de organización. Crea un publicador de organización nuevo y añade al
invocador como propietario. Este endpoint no migra identificadores existentes de usuarios o publicadores personales ni
marca al publicador como de confianza u oficial.

- Cuerpo: `{ "handle": "opik", "displayName": "Opik" }`
- Respuesta: `{ "ok": true, "publisherId": "...", "handle": "opik", "created": true, "trusted": false }`
- Devuelve `409` cuando el identificador ya está en uso por un publicador, usuario o publicador personal.

### `POST /api/v1/users/reserve`

Solo para administradores. Reserva slugs raíz y nombres de paquetes para su propietario legítimo sin publicar una
versión. Los nombres de paquetes se convierten en paquetes marcadores de posición privados sin filas de versiones, de modo que el mismo
propietario pueda publicar posteriormente la versión real del plugin de código o del plugin de paquete con ese nombre.

- Cuerpo: `{ "handle": "openclaw", "slugs": ["diffs"], "packageNames": ["@openclaw/diffs"], "reason": "reserved for official OpenClaw plugin" }`
- Respuesta: `{ "ok": true, "succeeded": 2, "failed": 0, "results": [{ "kind": "slug", "name": "diffs", "ok": true, "action": "reserved" }] }`

### `POST /api/v1/users/publisher-recovery`

Solo para administradores. Recupera un publicador personal para una identidad principal de OAuth de GitHub de reemplazo verificada
sin editar las filas de cuentas de Convex Auth. La solicitud debe especificar los identificadores inmutables de las cuentas del proveedor de GitHub
de ambas identidades; los identificadores mutables solo se utilizan como protección orientada al operador.

El endpoint utiliza de forma predeterminada una ejecución de prueba. Para aplicar la recuperación se requieren `dryRun: false` y
`confirmIdentityVerified: true` después de que el personal verifique de manera independiente la continuidad entre ambas
identidades principales de GitHub. La recuperación se interrumpe de forma segura cuando el publicador personal actual
del usuario de destino tiene skills, paquetes o fuentes de skills de GitHub.
La recuperación también migra los campos `ownerUserId` heredados de las skills del publicador recuperado,
los alias de slugs de skills, los paquetes, las advertencias del inspector de paquetes y las filas derivadas de resúmenes de búsqueda, para que
las rutas de propietario directo coincidan con la nueva autoridad del publicador. Una reserva activa del identificador protegido
para el identificador recuperado también se reasigna al usuario de reemplazo, de modo que la sincronización posterior
del perfil no pueda restaurar la autoridad competidora del usuario anterior. Cada tabla principal está limitada a
100 filas por transacción de aplicación; las recuperaciones más grandes deben usar primero una migración reanudable de propietarios.
Las fuentes de skills de GitHub están vinculadas al publicador y se notifican como comprobadas en lugar de reescribirse.

- Cuerpo: `{ "handle": "gingiris", "nextUserHandle": "gingiris-1031", "previousGitHubProviderAccountId": "123", "nextGitHubProviderAccountId": "456", "reason": "Verified account continuity for issue #2555", "confirmIdentityVerified": true, "dryRun": false }`
- Respuesta: `{ "ok": true, "dryRun": false, "recovered": true, "publisherId": "...", "handle": "gingiris", "previousUser": { "userId": "...", "handle": "gingiris", "nextHandle": "gingiris-recovered", "githubProviderAccountId": "123", "authAccountCount": 1 }, "nextUser": { "userId": "...", "handle": "gingiris-1031", "nextHandle": "gingiris", "githubProviderAccountId": "456", "authAccountCount": 1 }, "retiredPersonalPublisher": null, "resourceOwnerMigration": { "limitPerTable": 100, "skills": 1, "skillSlugAliases": 1, "packages": 0, "packageInspectorWarnings": 0, "githubSourcesChecked": 1, "handleReservations": 1 }, "identityVerified": true, "reason": "Verified account continuity for issue #2555" }`

### Endpoints de gestión de slugs del propietario

- `POST /api/v1/skills/{slug}/rename`
  - Cuerpo: `{ "newSlug": "new-canonical-slug" }`
  - Respuesta: `{ "ok": true, "slug": "new-canonical-slug", "previousSlug": "old-slug" }`
- `POST /api/v1/skills/{slug}/merge`
  - Cuerpo: `{ "targetSlug": "canonical-target-slug" }`
  - Respuesta: `{ "ok": true, "sourceSlug": "old-slug", "targetSlug": "canonical-target-slug" }`

Notas:

- Ambos endpoints requieren autenticación mediante token de API y solo funcionan para el propietario de la skill.
- `rename` conserva el slug anterior como alias de redirección.
- `merge` oculta el listado de origen y redirige el slug de origen al listado de destino.

### Endpoints de transferencia de propiedad

- `POST /api/v1/skills/{slug}/transfer`
  - Cuerpo: `{ "toUserHandle": "target_handle", "message": "optional" }`
  - Respuesta: `{ "ok": true, "transferId": "skillOwnershipTransfers:...", "toUserHandle": "target_handle", "expiresAt": 1730000000000 }`
- `POST /api/v1/skills/{slug}/transfer/accept`
- `POST /api/v1/skills/{slug}/transfer/reject`
- `POST /api/v1/skills/{slug}/transfer/cancel`
  - Respuesta (aceptar/rechazar/cancelar): `{ "ok": true, "skillSlug": "demo-skill?" }`
- `GET /api/v1/transfers/incoming`
- `GET /api/v1/transfers/outgoing`
  - Formato de la respuesta: `{ "transfers": [{ "_id": "...", "skill": { "slug": "demo", "displayName": "Demo" }, "fromUser"|"toUser": { "handle": "..." }, "message": "...", "requestedAt": 0, "expiresAt": 0 }] }`

### `POST /api/v1/users/ban`

Bloquea a un usuario y elimina de forma permanente las skills de su propiedad (solo moderadores o administradores).

Cuerpo:

```json
{ "handle": "user_handle", "reason": "motivo opcional del bloqueo" }
```

o

```json
{ "userId": "users_...", "reason": "motivo opcional del bloqueo" }
```

Respuesta:

```json
{ "ok": true, "alreadyBanned": false, "deletedSkills": 3 }
```

### `POST /api/v1/users/unban`

Desbloquea a un usuario y restaura las skills aptas (solo administradores).

Cuerpo:

```json
{ "handle": "user_handle", "reason": "motivo opcional del desbloqueo" }
```

o

```json
{ "userId": "users_...", "reason": "motivo opcional del desbloqueo" }
```

Respuesta:

```json
{ "ok": true, "alreadyUnbanned": false, "restoredSkills": 3 }
```

### `POST /api/v1/users/reclassify-ban`

Cambia el motivo almacenado de un bloqueo existente sin desbloquear al usuario ni restaurar
el contenido (solo administradores). Utiliza de forma predeterminada una ejecución de prueba, salvo que `dryRun` sea `false`.

Cuerpo:

```json
{ "handle": "user_handle", "reason": "spam de publicaciones masivas", "dryRun": true }
```

o

```json
{ "userId": "users_...", "reason": "spam de publicaciones masivas", "dryRun": false }
```

Respuesta:

```json
{
  "ok": true,
  "dryRun": false,
  "userId": "users_...",
  "handle": "user_handle",
  "previousReason": "bloqueo automático por malware",
  "nextReason": "spam de publicaciones masivas",
  "changed": true
}
```

### `POST /api/v1/users/role`

Cambia el rol de un usuario (solo administradores).

Cuerpo:

```json
{ "handle": "user_handle", "role": "moderator" }
```

o

```json
{ "userId": "users_...", "role": "admin" }
```

Respuesta:

```json
{ "ok": true, "role": "moderator" }
```

### `GET /api/v1/users`

Enumera o busca usuarios (solo administradores).

Parámetros de consulta:

- `q` (opcional): consulta de búsqueda
- `query` (opcional): alias de `q`
- `limit` (opcional): resultados máximos (valor predeterminado: 20; máximo: 200)

Respuesta:

```json
{
  "items": [
    {
      "userId": "users_...",
      "handle": "user_handle",
      "displayName": "Usuario",
      "name": "Usuario",
      "role": "moderator"
    }
  ],
  "total": 1
}
```

### `POST /api/v1/stars/{slug}` / `DELETE /api/v1/stars/{slug}`

Añade o elimina un marcador. La ruta heredada `stars` y los nombres de los campos de respuesta se mantienen
por compatibilidad. Ambos endpoints son idempotentes.

Respuestas:

```json
{ "ok": true, "starred": true, "alreadyStarred": false }
```

```json
{ "ok": true, "unstarred": true, "alreadyUnstarred": false }
```

## Endpoints heredados de la CLI (obsoletos)

Aún se admiten para versiones anteriores de la CLI:

- `GET /api/cli/whoami`
- `POST /api/cli/upload-url`
- `POST /api/cli/publish`
- `POST /api/cli/telemetry/install`
- `POST /api/cli/skill/delete`
- `POST /api/cli/skill/undelete`

Consulte `DEPRECATIONS.md` para conocer el plan de eliminación.

`POST /api/cli/upload-url` devuelve `uploadUrl` y `uploadTicket`. Las publicaciones de paquetes
que preparan un tarball de ClawPack deben enviar el identificador de almacenamiento resultante como
`clawpack` y el tique devuelto como `clawpackUploadTicket`.

## Detección del registro (`/.well-known/clawhub.json`)

La CLI puede detectar la configuración del registro y de autenticación desde el sitio:

- `/.well-known/clawhub.json` (JSON, preferido)
- `/.well-known/clawdhub.json` (heredado)

Esquema:

```json
{ "apiBase": "https://clawhub.ai", "authBase": "https://clawhub.ai", "minCliVersion": "0.0.5" }
```

Si utiliza alojamiento propio, sirva este archivo (o establezca `CLAWHUB_REGISTRY` explícitamente; `CLAWDHUB_REGISTRY` es la opción heredada).
