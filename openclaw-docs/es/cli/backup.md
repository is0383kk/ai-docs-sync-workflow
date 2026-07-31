---
read_when:
    - Se necesita un archivo de copia de seguridad de primera clase para el estado local de OpenClaw
    - Necesita una instantánea compacta y verificada de una base de datos SQLite de OpenClaw.
    - Quieres previsualizar qué rutas se incluirían antes de restablecer o desinstalar
summary: Referencia de la CLI para `openclaw backup` (archivos y snapshots de SQLite)
title: Copia de seguridad
x-i18n:
    generated_at: "2026-07-26T05:05:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: dfb5a118545589b181cede26dab72e9d029d98a1cac5cfccedd9d9cf2c56d3b5
    source_path: cli/backup.md
    workflow: 16
---

# `openclaw backup`

Crea un archivo de copia de seguridad local para el estado, la configuración, los perfiles de autenticación, las credenciales de canales/proveedores, las sesiones y, opcionalmente, los espacios de trabajo de OpenClaw.

```bash
openclaw backup create
openclaw backup create --output ~/Backups
openclaw backup create --dry-run --json
openclaw backup create --verify
openclaw backup create --no-include-workspace
openclaw backup create --only-config
openclaw backup verify ./2026-03-09T08-00-00.000+08-00-openclaw-backup.tar.gz
openclaw backup sqlite create --global --repository ~/Backups/openclaw-sqlite
openclaw backup sqlite create --agent main --repository ~/Backups/openclaw-sqlite
openclaw backup sqlite list --repository ~/Backups/openclaw-sqlite
openclaw backup sqlite verify ~/Backups/openclaw-sqlite/<snapshot-id>
openclaw backup sqlite verify ~/Backups/openclaw-sqlite/<snapshot-id> --scratch ~/Private/openclaw-scratch
openclaw backup sqlite restore ~/Backups/openclaw-sqlite/<snapshot-id> --target ./restored/openclaw.sqlite
```

## Notas

- El archivo incorpora un `manifest.json` con las rutas de origen resueltas y la disposición del archivo.
- La salida predeterminada es un archivo `.tar.gz` con marca de tiempo en el directorio de trabajo actual. Los nombres de archivo con marca de tiempo usan la zona horaria local de la máquina e incluyen el desfase respecto de UTC. Si el directorio de trabajo actual está dentro de un árbol de origen incluido en la copia de seguridad, OpenClaw utiliza el directorio personal como ubicación predeterminada del archivo.
- Los archivos existentes nunca se sobrescriben. Se rechazan las rutas de salida situadas dentro de los árboles de estado o de espacios de trabajo de origen para evitar que se incluyan a sí mismas.
- `openclaw backup verify <archive>` comprueba que el archivo contenga exactamente un manifiesto raíz, rechaza las rutas del archivo que intentan recorrer directorios y los archivos auxiliares de SQLite, confirma que existan todas las cargas útiles declaradas en el manifiesto, valida la estructura de archivo de cada instantánea de SQLite y ejecuta comprobaciones completas de integridad y función en las bases de datos canónicas de OpenClaw. Los esquemas dedicados de plugins permanecen opacos porque pueden requerir capacidades de SQLite definidas por sus propietarios. `openclaw backup create --verify` ejecuta esa validación inmediatamente después de escribir el archivo.
- `openclaw backup create --only-config` crea una copia de seguridad únicamente del archivo de configuración JSON activo.

## Instantáneas de SQLite

Utiliza `openclaw backup sqlite` cuando necesites un artefacto portátil para una sola base de datos SQLite propiedad de OpenClaw en lugar de un archivo amplio del estado.

La creación de instantáneas acepta exactamente un origen con nombre:

| Comando                                                         | Base de datos                    |
| --------------------------------------------------------------- | -------------------------------- |
| `openclaw backup sqlite create --global --repository <dir>`     | Estado compartido de OpenClaw    |
| `openclaw backup sqlite create --agent <id> --repository <dir>` | Una base de datos por cada agente |

El repositorio contiene un directorio por cada instantánea confirmada. Cada directorio de instantánea contiene exactamente:

- `manifest.json`
- `database.sqlite`

La creación de instantáneas verifica la base de datos activa antes de leerla, utiliza la API de copia de seguridad en línea de SQLite para capturar el estado confirmado del WAL sin mantener una transacción de lectura prolongada, cierra la base de datos activa, compacta la copia privada con `VACUUM`, vuelve a verificar la base de datos generada y publica el directorio completado sin sobrescribir rutas existentes. Las instantáneas globales eliminan las filas transitorias de la cola de entrega antes de la compactación para que las cargas útiles eliminadas de la cola no se conserven en las páginas libres.

No copies archivos activos `.sqlite`, `-wal`, `-shm` ni `-journal` como artefactos de portabilidad. Copia únicamente directorios de instantáneas completados.

Las instantáneas de SQLite pueden contener perfiles de autenticación, estado de sesiones, estado de plugins y otros registros confidenciales. Protege los repositorios con los mismos permisos, cifrado, política de retención y restricciones de destino que el directorio de estado activo de OpenClaw.

### Verificación y restauración

```bash
openclaw backup sqlite verify <snapshot-directory>
openclaw backup sqlite restore <snapshot-directory> --target <new-database-path>
```

La verificación comprueba la estructura estricta del manifiesto, el tamaño y el SHA-256 del artefacto, la integridad de SQLite, las claves externas, la versión del esquema, la función y el propietario de la base de datos, y las definiciones de los índices propiedad de OpenClaw.

La verificación valida una copia privada anclada al contenido para que las condiciones de carrera en los nombres de ruta no puedan sustituir los bytes que inspecciona SQLite. De forma predeterminada, esa copia temporal se crea junto al repositorio de instantáneas y se elimina antes de que termine el comando. La raíz de preparación y su cadena de ancestros deben impedir que otros usuarios la sustituyan. Las raíces POSIX deben pertenecer al usuario actual y no permitir escritura al grupo ni a otros usuarios; se aceptan ancestros con bit adhesivo, como `/tmp`, para elementos secundarios propiedad del usuario. Se rechazan las concesiones de ACL de macOS que expongan la preparación o permitan sustituirla. Las raíces y los ancestros de Windows deben pertenecer al usuario actual o a una entidad principal de confianza del sistema operativo, con ACL que impidan el acceso de entidades no confiables a la preparación. Para un montaje de solo lectura o un recurso compartido de red, proporciona `--scratch <existing-private-directory>` en un almacenamiento con controles equivalentes de cifrado y destino.

La creación de instantáneas aplica al repositorio las mismas comprobaciones de propietario, ACL, ancestros e identidad de ruta antes de preparar o publicar los bytes de la base de datos.

La restauración repite la verificación y escribe únicamente en un destino nuevo. Rechaza un destino existente o un archivo auxiliar `-wal`, `-shm` o `-journal`, y nunca sustituye en el mismo lugar una base de datos activa de OpenClaw. El directorio padre del destino tiene los mismos requisitos de seguridad de ruta que el área temporal de verificación. La activación de una base de datos restaurada sigue siendo un paso explícito del operador con el sistema fuera de línea.

Los repositorios de instantáneas son directorios locales. La programación, la carga, la retención, los paquetes WAL incrementales, la conmutación por error y el comportamiento de restauración durante el arranque quedan intencionadamente fuera de este comando.

## Contenido de la copia de seguridad

`openclaw backup create` planifica los orígenes a partir de la instalación local de OpenClaw:

- El directorio de estado (normalmente `~/.openclaw`)
- La ruta del archivo de configuración activo
- El directorio `credentials/` resuelto cuando existe fuera del directorio de estado
- Los directorios de espacios de trabajo detectados a partir de la configuración actual, salvo que proporciones `--no-include-workspace`

Los perfiles de autenticación y otros estados de ejecución por agente residen en SQLite dentro del directorio de estado (`agents/<agentId>/agent/openclaw-agent.sqlite`), por lo que la entrada de copia de seguridad del estado los incluye automáticamente.

`--only-config` omite la detección del estado, del directorio de credenciales y de los espacios de trabajo, y archiva únicamente la ruta del archivo de configuración activo.

OpenClaw canonicaliza las rutas antes de crear el archivo: si la configuración, el directorio de credenciales o un espacio de trabajo ya se encuentran dentro del directorio de estado, no se duplican como orígenes independientes de nivel superior en la copia de seguridad. Las rutas inexistentes se omiten.

Durante la creación del archivo, OpenClaw excluye las rutas conocidas que pueden modificarse en tiempo real antes de que `tar` las lea. Esto evita condiciones de carrera entre el tamaño registrado de un archivo y las escrituras simultáneas. El filtro aplica las siguientes reglas relativas al estado en cada directorio de estado incluido en la copia de seguridad:

| Ámbito relativo al estado                       | Sufijos de archivo omitidos        |
| ----------------------------------------------- | ---------------------------------- |
| `sessions/**`                                | `.jsonl`, `.log`              |
| `agents/<agentId>/sessions/**`               | `.jsonl`, `.log`              |
| `cron/runs/**`                               | `.jsonl`, `.log`              |
| `logs/**`                                    | `.jsonl`, `.log`              |
| `delivery-queue/**`                          | `.json`, `.delivered`, `.tmp` |
| `session-delivery-queue/**`                  | `.json`, `.delivered`, `.tmp` |
| Cualquier ruta dentro del directorio de estado incluido en la copia de seguridad | `.sock`, `.pid`, `.tmp`       |

Estas reglas no filtran los archivos de espacios de trabajo situados fuera del directorio de estado. También omiten los archivos completados de transcripciones y registros que coincidan con la tabla, por lo que esos registros deben conservarse por separado cuando sea necesario. El campo `skippedVolatileCount` del resultado JSON indica cuántos archivos se omitieron intencionadamente.

Las bases de datos SQLite situadas bajo el directorio de estado se capturan con la API de copia de seguridad en línea de SQLite y se compactan fuera de línea con `VACUUM` para impedir que los restos de páginas eliminadas entren en el archivo; los archivos WAL/SHM activos no se copian. Si una base de datos propiedad de un plugin requiere capacidades de SQLite definidas por su propietario que no están disponibles, la operación falla de forma segura en lugar de recurrir a una copia directa del archivo. Los archivos SQLite incluidos mediante copias de seguridad de espacios de trabajo se copian como archivos del espacio de trabajo y no están cubiertos por la garantía de compactación.

Se incluyen los archivos de código fuente y manifiestos de plugins instalados bajo el árbol `extensions/` del directorio de estado, pero se omiten sus árboles de dependencias `node_modules/` anidados porque son artefactos de instalación que pueden reconstruirse. Después de restaurar un archivo, utiliza `openclaw plugins update <id>` o vuelve a instalarlo con `openclaw plugins install <spec> --force` si un plugin restaurado informa de dependencias ausentes.

También se omiten las raíces de ejecución administradas por el instalador y que pueden reconstruirse dentro del directorio de estado: `dev/`, `git/`, `npm/`, la raíz heredada `npm-runtime/` y `tools/`. Estas contienen copias de trabajo administradas, árboles de paquetes y entornos de ejecución descargados, no el estado autoritativo del usuario; vuelve a instalar o actualiza el entorno de ejecución o plugin correspondiente después de la restauración. Se seguirá incluyendo cualquier archivo de configuración, directorio de credenciales o espacio de trabajo configurado explícitamente dentro de una de estas raíces.

## Comportamiento ante una configuración no válida

`openclaw backup` omite la comprobación previa normal de la configuración para que pueda seguir siendo útil durante la recuperación. La detección de espacios de trabajo depende de una configuración válida, por lo que `openclaw backup create` falla de inmediato cuando el archivo de configuración existe pero no es válido y la copia de seguridad de espacios de trabajo sigue habilitada.

Para realizar una copia de seguridad parcial en esa situación, vuelve a ejecutar con `--no-include-workspace`: mantiene dentro del ámbito el estado, la configuración y el directorio externo de credenciales, pero omite por completo la detección de espacios de trabajo.

`--only-config` también funciona cuando la configuración está mal formada, ya que no analiza la configuración para detectar espacios de trabajo.

## Tamaño y rendimiento

OpenClaw no impone un tamaño máximo integrado para las copias de seguridad ni un límite de tamaño por archivo. Si una escritura del archivo no produce datos durante cinco minutos, falla y elimina el archivo temporal parcial en lugar de quedar bloqueada indefinidamente. Por lo demás, los límites prácticos dependen de:

- El espacio disponible para la escritura del archivo temporal y el archivo final
- El tiempo necesario para recorrer árboles grandes de espacios de trabajo y comprimirlos en un `.tar.gz`
- El tiempo necesario para volver a analizar el archivo con `--verify` o `openclaw backup verify`
- El comportamiento del sistema de archivos de destino: OpenClaw requiere una publicación mediante enlace físico sin sobrescritura para que la ruta final del archivo nunca exponga una copia en curso; los sistemas de archivos incompatibles fallan con un error que indica cómo actuar

Si la confirmación de durabilidad del directorio final falla después de la publicación, el comando informa del fallo, pero conserva la entrada final completa para no arriesgarse a eliminar una sustitución simultánea.

Los espacios de trabajo grandes suelen ser el principal factor que determina el tamaño del archivo. Utiliza `--no-include-workspace` para obtener una copia de seguridad más pequeña y rápida, o `--only-config` para obtener el archivo más pequeño.

## Contenido relacionado

- [Referencia de la CLI](/es/cli)
