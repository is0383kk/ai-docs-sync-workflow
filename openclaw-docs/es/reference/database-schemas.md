---
read_when:
    - Diagnóstico de un error de esquema de base de datos más reciente
    - Comprobación de la compatibilidad de la base de datos antes de una actualización o reversión a una versión anterior
    - Recuperación de una base de datos para una versión anterior de OpenClaw
summary: Ubicaciones de las bases de datos SQLite de OpenClaw, versiones de esquema, comprobaciones de integridad y recuperación tras una reversión a una versión anterior
title: Esquemas de bases de datos
x-i18n:
    generated_at: "2026-07-26T04:53:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 73993e2c593ba460784108aedef70bbfb499e525c709d6d6bdd956ccf93e0ddc
    source_path: reference/database-schemas.md
    workflow: 16
---

OpenClaw almacena el estado del plano de control en una base de datos SQLite global y los datos de cada agente en una base de datos SQLite por agente. Las migraciones de esquema se ejecutan hacia delante cuando se abre una base de datos. Las compilaciones anteriores de OpenClaw rechazan las bases de datos escritas con un esquema más reciente.

## Estructura de las bases de datos

| Ámbito                  | Ruta predeterminada                                        | Contenido                                                                                                                |
| ----------------------- | ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| Plano de control global | `~/.openclaw/state/openclaw.sqlite`                                         | Estado de configuración compartido, registros, aprobaciones, estado de plugins y estado de ejecución compartido          |
| Plano de datos por agente | `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`                                       | Sesiones, transcripciones, índices de memoria, estado de autenticación, estado de conversación y estado de ejecución del agente |

Algunas funciones de gran volumen o específicas del ciclo de vida utilizan almacenes SQLite dedicados, incluidos el registro de tareas y los datos de trayectorias.

## Contrato de versionado

Cada base de datos registra su esquema en dos lugares:

- `PRAGMA user_version` es la versión del esquema SQLite.
- La fila principal `schema_meta` registra `role`, `agent_id`, `schema_version` y `app_version`. `app_version` es la compilación de OpenClaw que escribió por última vez los metadatos del esquema.

OpenClaw aplica migraciones únicamente hacia delante cuando abre una base de datos compatible más antigua. Rechaza las bases de datos cuyo `user_version` sea más reciente que la compilación en ejecución e informa de un error `newer schema version`. El Gateway comprueba todas las bases de datos registradas antes de iniciarse. `openclaw update` también rechaza un paquete o destino de código fuente cuya compatibilidad de esquema declarada sea anterior a la de una base de datos en disco. No es posible realizar la comprobación previa de los paquetes de destino publicados antes de que se añadieran los metadatos del esquema.

La instalación manual de OpenClaw mediante npm omite la protección del actualizador. Las comprobaciones de apertura de la base de datos siguen rechazando las compilaciones incompatibles.

## Historial del esquema de agentes

| Versión | Cambio                                                                                                                                                                                                                                                         | Primera versión                                        |
| ------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------ |
| 1       | Almacén inicial por agente ([#88349](https://github.com/openclaw/openclaw/pull/88349))                                                                                                                                                                         | `v2026.5.30-beta.1`, estable hasta `v2026.7.1`   |
| 2       | Identidad del índice de memoria ([#104449](https://github.com/openclaw/openclaw/pull/104449))                                                                                                                                                                  | `v2026.7.2-beta.1`                                     |
| 4       | Sesiones y transcripciones trasladadas a SQLite ([#98236](https://github.com/openclaw/openclaw/pull/98236))                                                                                                                                                    | `v2026.7.2-beta.1`                                     |
| 5-6     | Actualidad del terminal y ciclo de vida del estado ([#104859](https://github.com/openclaw/openclaw/pull/104859))                                                                                                                                               | `v2026.7.2-beta.1`                                     |
| 7       | Proyección del estado del ciclo de vida por entrada ([#106151](https://github.com/openclaw/openclaw/pull/106151))                                                                                                                                              | `v2026.7.2-beta.1`                                     |
| 8       | Procedencia de la sesión por transcripción ([#106766](https://github.com/openclaw/openclaw/pull/106766))                                                                                                                                                       | `v2026.7.2-beta.2`                                     |
| 9       | Tablas `STRICT` ([#108663](https://github.com/openclaw/openclaw/pull/108663))                                                                                                                                                                       | `v2026.7.2-beta.2`                                     |
| 10      | Rutas de transcripciones activas materializadas ([#108851](https://github.com/openclaw/openclaw/pull/108851))                                                                                                                                                  | Sin publicar                                           |
| 11      | Arrendamientos, entrega duradera, direcciones de conversación y resultados de Heartbeat ([#109636](https://github.com/openclaw/openclaw/pull/109636), [#95838](https://github.com/openclaw/openclaw/pull/95838), [#109999](https://github.com/openclaw/openclaw/pull/109999)) | Sin publicar                                      |

La versión 3 fue un paso de desarrollo no publicado que se integró en la versión 4.

## Historial del esquema de estado

| Versión | Cambio                                                                                                                    | Primera versión       |
| ------- | ------------------------------------------------------------------------------------------------------------------------- | --------------------- |
| 1       | Base de datos inicial de estado compartido                                                                                | `v2026.5.30-beta.1`    |
| 2       | Eventos de auditoría de mensajes solo con metadatos ([#103903](https://github.com/openclaw/openclaw/pull/103903))          | `v2026.7.2-beta.1`    |
| 3       | Tablas `STRICT` y refuerzo contra desviaciones del esquema ([#108663](https://github.com/openclaw/openclaw/pull/108663)) | `v2026.7.2-beta.2` |
| 4       | La procedencia de supervisión de sesiones sustituye las filas centinela codificadas                                       | Sin publicar          |

## Comprobaciones de integridad

| Cuándo                                      | Comprobación                                                                 |
| ------------------------------------------- | --------------------------------------------------------------------------- |
| En cada apertura                            | Validar la tabla `schema_meta` y la fila principal de metadatos        |
| Antes de una migración pendiente            | Ejecutar un análisis completo de integridad, claves externas, roles, esquema e índices |
| Verificador en segundo plano del Gateway    | Ejecutar el análisis completo aproximadamente una vez al día y registrar los resultados |
| Doctor, verificación de copias de seguridad y Compaction | Ejecutar el análisis completo antes de aceptar o reescribir la base de datos |

La comprobación previa del Gateway solo lee las cabeceras del esquema. El verificador en segundo plano se encarga del análisis completo, más lento, de las bases de datos que no necesitan migración.
Las decisiones de cuarentena solo se almacenan en un almacén `openclaw-quarantine.sqlite` dedicado, por lo que sobreviven a los daños en las bases de datos puestas en cuarentena. Los resultados de la verificación se registran.

## Solución de problemas

### Por qué no es posible volver atrás después de actualizar a 2026.7.2

Todas las versiones hasta `v2026.7.1` utilizaron el esquema de agentes 1 y el esquema de estado 1. La serie de versiones 2026.7.2 (a partir de `v2026.7.2-beta.1`) migra las bases de datos hacia delante durante el primer inicio. Esta migración es unidireccional: los datos se reescriben con el esquema más reciente, y la instalación posterior de una versión anterior de OpenClaw no la deshace. La compilación anterior se niega a iniciarse con un error `newer schema version` que identifica la compilación propietaria de la base de datos.

Cambiar el binario a una versión anterior nunca revierte los datos. Si es imprescindible ejecutar una versión anterior a 2026.7.2 después de actualizar, existen tres opciones:

1. Restaurar una copia de seguridad creada antes de la actualización. [Crear y verificar copias de seguridad](/es/cli/backup) antes de realizar actualizaciones importantes.
2. Ejecutar la compilación anterior con un directorio de estado independiente (`OPENCLAW_STATE_DIR`). Comenzará desde cero; los datos migrados permanecerán intactos para cuando se vuelva a la compilación más reciente.
3. Seguir el procedimiento manual para volver a una versión anterior que se indica a continuación. No se admite y conlleva riesgo de pérdida de datos si no se dispone de una copia de seguridad verificada.

Desde 2026.7.2, `openclaw update` se niega a instalar una versión que no pueda abrir las bases de datos actuales, por lo que el actualizador no provocará esta situación. La instalación manual de una versión anterior mediante npm omite esta protección; las bases de datos siguen rechazando el binario anterior, pero solo después de instalarlo.

### El Gateway se niega a iniciarse con un error de versión de esquema más reciente

Una compilación más reciente de OpenClaw escribió las bases de datos, y la compilación en ejecución es anterior. El error y el registro de inicio del Gateway identifican la compilación propietaria de la base de datos (`app_version`). Instalar esa versión o una más reciente, o utilizar una de las opciones anteriores. No modificar la base de datos para silenciar el error.

### Una base de datos se pone en cuarentena tras fallar la verificación de integridad

El verificador en segundo plano ha demostrado que el archivo está dañado, y ahora cada apertura falla inmediatamente en lugar de volver a analizarlo. Restaurar la base de datos desde una copia de seguridad o repararla y, a continuación, ejecutar `openclaw doctor --fix` para borrar el registro de cuarentena. Doctor informa de un error explícito si no puede borrar el propio registro de cuarentena; volver a ejecutarlo hasta que indique que no hay problemas.

## No se admite volver a versiones anteriores

Las regresiones manuales del esquema están destinadas a agentes y operadores que acepten el riesgo. [Crear y verificar una copia de seguridad](/es/cli/backup) antes de editar cualquier base de datos. Detener el Gateway y todos los procesos que puedan abrir la base de datos.

El procedimiento general es:

1. Leer el esquema y las migraciones de la versión de destino.
2. En una sola transacción, eliminar todas las tablas, índices, desencadenadores y columnas introducidos después de la versión de destino.
3. Establecer `PRAGMA user_version` y `schema_meta.schema_version` en la versión de destino.
4. Ejecutar la verificación completa de la base de datos de la versión de destino antes de iniciar el Gateway.

### Ejemplo: esquema de agentes 11 a 9

El esquema 10 añadió la proyección de transcripciones activas. El esquema 11 añadió arrendamientos, entrega duradera, estado de las direcciones de conversación y resultados de Heartbeat. La coordinación de QMD utiliza filas en `state_leases`; no existe una tabla QMD independiente que deba conservarse.

Ejecutar SQL equivalente en cada base de datos por agente afectada después de inspeccionar el esquema exacto que la escribió:

```sql
BEGIN IMMEDIATE;

DROP TABLE IF EXISTS heartbeat_outcomes;
DROP TABLE IF EXISTS conversation_deliveries;
DROP TABLE IF EXISTS state_leases;
DROP TABLE IF EXISTS session_transcript_active_events;

ALTER TABLE session_transcript_index_state DROP COLUMN active_event_count;
ALTER TABLE session_transcript_index_state DROP COLUMN active_message_count;
ALTER TABLE conversations DROP COLUMN delivery_target;

PRAGMA user_version = 9;
UPDATE schema_meta
SET schema_version = 9,
    updated_at = unixepoch('now') * 1000
WHERE meta_key = 'primary';

COMMIT;
```

Esto descarta el estado de las versiones 10-11, incluidas las operaciones de entrega en curso, los arrendamientos, los resultados de Heartbeat y la proyección derivada de transcripciones activas. Si el cambio a una versión anterior sale mal, se deberá restaurar la copia de seguridad verificada.
