---
read_when:
    - Fallan los comandos de la CLI de ClawHub o del registro de OpenClaw
    - No se puede instalar, publicar ni actualizar un paquete
summary: Solución de problemas de inicio de sesión, instalación, publicación, actualización y API de ClawHub.
x-i18n:
    generated_at: "2026-07-26T05:05:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fc789fcc891cf8c44b5d1a10d38a4e6dd4dec9474d8d13f8058ea1c3392a9f91
    source_path: clawhub/troubleshooting.md
    workflow: 16
---

# Solución de problemas

## `clawhub login` abre un navegador, pero nunca finaliza

La CLI inicia un servidor local de devolución de llamada de corta duración durante el inicio de sesión mediante el navegador.

- Asegúrese de que el navegador pueda acceder a `http://127.0.0.1:<port>/callback`.
- Compruebe las reglas locales del firewall, la VPN y el proxy si la devolución de llamada nunca llega.
- En entornos sin interfaz gráfica, cree un token de API en la interfaz web de ClawHub y ejecute:

```bash
clawhub login --token clh_...
```

## `whoami` o `publish` devuelve `Unauthorized` (401)

- Vuelva a iniciar sesión con `clawhub login`.
- Si utiliza una ruta de configuración personalizada, confirme que `CLAWHUB_CONFIG_PATH` apunte al
  archivo que contiene su token actual.
- Si utiliza un token de API, confirme que no se haya revocado en la interfaz web.

## La búsqueda o la instalación devuelve `Rate limit exceeded` (429)

Consulte la información sobre reintentos en la respuesta:

- `Retry-After`: segundos que se deben esperar antes de volver a intentarlo.
- `RateLimit-Limit`: límite aplicado a esta solicitud.
- `RateLimit-Remaining`: presupuesto restante exacto cuando el encabezado está presente. En `429`, es `0`.
- `RateLimit-Reset` o `X-RateLimit-Reset`: momento del restablecimiento.

Si muchos usuarios comparten una misma IP de salida, se pueden alcanzar los límites de IP anónima aunque cada
persona solo envíe unas pocas solicitudes. Inicie sesión cuando sea posible y vuelva a intentarlo tras el
retraso indicado.

## La búsqueda o la instalación falla detrás de un proxy

La CLI respeta las variables de proxy estándar:

```bash
export HTTPS_PROXY=http://proxy.example.com:3128
clawhub search "my query"
```

Los nombres admitidos incluyen `HTTPS_PROXY`, `HTTP_PROXY`, `https_proxy` y
`http_proxy`.

## Una habilidad no aparece en la búsqueda

- Compruebe el slug exacto o la página del propietario si los conoce.
- Confirme que la versión sea pública y que no esté retenida por el análisis o la moderación.
- Si la habilidad le pertenece, inicie sesión e inspecciónela:

```bash
clawhub inspect @openclaw/demo
```

Los diagnósticos visibles para el propietario pueden explicar el estado del análisis, del control de carga o de la moderación.

## La publicación falla porque faltan metadatos obligatorios

Para las habilidades, compruebe el frontmatter de `SKILL.md`. Se deben declarar las variables de entorno y
las herramientas obligatorias para que los usuarios y los analizadores puedan comprender el paquete.

Para los plugins, compruebe los metadatos de compatibilidad de `package.json`. Las publicaciones de plugins
de código necesitan campos de compatibilidad con OpenClaw como `openclaw.compat.pluginApi` y
`openclaw.build.openclawVersion`.

Obtenga primero una vista previa de la carga útil de publicación:

```bash
clawhub package publish <source> --family code-plugin --dry-run
```

## La publicación falla con un error del propietario o del origen de GitHub

ClawHub utiliza la identidad de GitHub y la atribución del origen para vincular los paquetes con quienes
los publican.

- Asegúrese de haber iniciado sesión con la cuenta de GitHub propietaria del paquete o que pueda publicarlo.
- Compruebe que la URL de origen sea pública o accesible para ClawHub.
- Para orígenes de GitHub, utilice `owner/repo`, `owner/repo@ref` o una URL completa de GitHub.

## La publicación falla porque un espacio de nombres está reclamado o reservado

Si una publicación falla porque el identificador del propietario, el espacio de nombres de la organización, el ámbito del paquete, el slug de la habilidad
o el nombre del paquete ya está reclamado o reservado, confirme primero que está
publicando con el propietario correspondiente al espacio de nombres. Para los paquetes de plugins,
los nombres con ámbito como `@example-org/example-plugin` deben publicarse con el
propietario `example-org` correspondiente.

Si considera que su organización, proyecto o marca es el propietario legítimo del espacio de nombres, pero
no puede gestionar el propietario actual de ClawHub, abra una
[incidencia de reclamación de organización o espacio de nombres](https://github.com/openclaw/clawhub/issues/new?template=org-namespace-claim.yml)
con pruebas públicas y no confidenciales. Consulte
[Reclamaciones de organizaciones y espacios de nombres](/es/clawhub/namespace-claims) para obtener orientación sobre las pruebas y saber qué
no se debe incluir en incidencias públicas.

## `sync` indica que no se encontraron habilidades

`sync` busca carpetas que contengan `SKILL.md` o `skill.md`.

Indique las raíces que desea analizar:

```bash
clawhub sync --root /path/to/skills
```

Obtenga primero una vista previa si no sabe con certeza qué se publicará:

```bash
clawhub sync --all --dry-run --no-input
```

## `update` se niega a continuar debido a cambios locales

Los archivos locales no coinciden con ninguna versión conocida por ClawHub. Elija una opción:

- Conserve las modificaciones locales y omita la actualización.
- Sobrescriba con la versión publicada:

```bash
clawhub update @openclaw/demo --force
```

- Publique la copia modificada como un slug nuevo o una bifurcación.

## La instalación de un plugin falla en OpenClaw

- Utilice un origen explícito de ClawHub:

```bash
openclaw plugins install clawhub:<package>
```

- Compruebe en la página de detalles del paquete el estado del análisis y los metadatos de compatibilidad.
- Confirme que su versión de OpenClaw satisfaga el intervalo de compatibilidad
  anunciado por el paquete.
- Si el paquete está oculto, retenido o bloqueado, es posible que no se pueda instalar hasta que
  el propietario resuelva el problema.

## Las solicitudes a la API pública fallan

- Respete los encabezados de reintento de `429` y almacene en caché las respuestas públicas de listado y búsqueda.
- Dirija a los usuarios al listado canónico de ClawHub.
- No replique contenido oculto, privado, retenido o bloqueado por moderación fuera de la
  superficie de la API pública.

Consulte [API HTTP](/es/clawhub/http-api) para obtener detalles de los endpoints.
