---
doc-schema-version: 1
read_when:
    - Quiere encontrar plugins de OpenClaw de terceros
    - Se desea publicar o incluir un Plugin propio en ClawHub
summary: Buscar y publicar plugins de OpenClaw mantenidos por la comunidad
title: Plugins de la comunidad
x-i18n:
    generated_at: "2026-07-26T05:19:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6a9eb477f20da8171a35c22ea6b112d77ff4afe0878f60314c052746aef4e0ac
    source_path: plugins/community.md
    workflow: 16
---

Los plugins de la comunidad son paquetes de terceros que amplían OpenClaw con
canales, herramientas, proveedores, hooks u otras capacidades. Utilice
[ClawHub](/es/clawhub) como principal medio de descubrimiento de plugins públicos de la
comunidad.

## Buscar plugins

Busque en ClawHub desde la CLI:

```bash
openclaw plugins search "calendar"
```

Instale un plugin de ClawHub con un prefijo de origen explícito:

```bash
openclaw plugins install clawhub:<package-name>
```

npm sigue siendo una vía compatible de instalación directa durante la transición del lanzamiento:

```bash
openclaw plugins install npm:<package-name>
```

Consulte [Gestionar plugins](/es/plugins/manage-plugins) para ver ejemplos habituales de instalación, actualización,
inspección y desinstalación. Consulte [`openclaw plugins`](/es/cli/plugins) para ver
la referencia completa de comandos y las reglas de selección de origen.

## Publicar plugins

Publique los plugins públicos de la comunidad en ClawHub para que los usuarios de OpenClaw puedan descubrirlos
e instalarlos. ClawHub gestiona el listado activo de paquetes, el historial de versiones,
el estado del análisis y las indicaciones de instalación; la documentación no mantiene un catálogo
estático de plugins de terceros.

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

Antes de publicar, asegúrese de que el plugin tenga metadatos del paquete, un
manifiesto del plugin, documentación de configuración y un responsable de mantenimiento claramente definido. ClawHub valida el ámbito
del propietario, el nombre del paquete, la versión, los límites de archivos y los metadatos de origen antes de
crear una versión; después, mantiene las versiones nuevas ocultas de las interfaces normales de instalación y
descarga hasta que finalicen la revisión y la verificación.

Lista de comprobación antes de publicar:

| Requisito            | Motivo                                              |
| -------------------- | --------------------------------------------------- |
| Publicado en ClawHub | Los usuarios necesitan que funcionen las indicaciones de `openclaw plugins install` |
| Repositorio público de GitHub | Revisión del código fuente, seguimiento de incidencias y transparencia |
| Documentación de configuración y uso | Los usuarios necesitan saber cómo configurarlo |
| Mantenimiento activo | Actualizaciones recientes o gestión ágil de incidencias |

Contrato completo de publicación:

- [Publicación en ClawHub](/es/clawhub/publishing) - propietarios, ámbitos, versiones,
  revisión, validación de paquetes y transferencia de paquetes
- [Creación de plugins](/es/plugins/building-plugins) - la estructura del paquete del plugin
  y el flujo de trabajo de la primera publicación
- [Manifiesto del plugin](/es/plugins/manifest) - campos del manifiesto nativo del plugin

## Temas relacionados

- [Plugins](/es/tools/plugin) - instalar, configurar, reiniciar y solucionar problemas
- [Gestionar plugins](/es/plugins/manage-plugins) - ejemplos de comandos
- [Publicación en ClawHub](/es/clawhub/publishing) - reglas de publicación y versiones
