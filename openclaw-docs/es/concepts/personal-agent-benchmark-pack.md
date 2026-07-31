---
read_when:
    - Ejecución de comprobaciones locales de fiabilidad del agente personal
    - Ampliación del catálogo de escenarios de QA respaldado por el repositorio
    - Verificación de recordatorios, respuestas, memoria, censura, seguimiento seguro de herramientas, estado de tareas, diagnósticos seguros para compartir, afirmaciones de finalización respaldadas por pruebas y recuperación ante fallos
summary: Escenarios locales de qa-channel para comprobar flujos de trabajo de asistentes personales que preservan la privacidad.
title: Paquete de pruebas de rendimiento para agentes personales
x-i18n:
    generated_at: "2026-07-26T04:35:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 35da45e4b22b1044a777fa8d6bce87f9ace377950dd0af3f2419b40cfe4d9be6
    source_path: concepts/personal-agent-benchmark-pack.md
    workflow: 16
---

El paquete de referencia de agentes personales es un pequeño conjunto de escenarios de QA respaldado por un repositorio para
flujos de trabajo locales de asistentes personales. No es una referencia genérica de modelos y
no necesita un ejecutor nuevo: reutiliza la pila privada de QA ([descripción general de QA](/es/concepts/qa-e2e-automation)),
el [canal de QA](/es/channels/qa-channel) sintético y el catálogo YAML
`qa/scenarios` existente.

## Escenarios

Diez escenarios, definidos en `qa/scenarios/personal/*.yaml`:

| Id. del escenario                          | Comprobaciones                                                                                |
| ------------------------------------------ | --------------------------------------------------------------------------------------------- |
| `personal-reminder-roundtrip`              | Recordatorios personales ficticios mediante entrega de Cron local                             |
| `personal-channel-thread-reply`            | Enrutamiento de mensajes directos y respuestas en hilos ficticios mediante `qa-channel`       |
| `personal-memory-preference-recall`        | Recuperación de preferencias ficticias desde los archivos de memoria temporales del espacio de trabajo de QA |
| `personal-redaction-no-secret-leak`        | Comprobaciones para no repetir secretos ficticios                                             |
| `personal-tool-safety-followthrough`       | Continuación segura mediante una herramienta de lectura tras un breve turno de tipo aprobación |
| `personal-approval-denial-stop`            | Comportamiento de detención ante la denegación de aprobación para una solicitud sensible de lectura local |
| `personal-task-followthrough-status`       | Informes del estado de tareas respaldados por pruebas que mantienen separados los estados pendiente, bloqueado y completado |
| `personal-share-safe-diagnostics-artifact` | Artefactos de diagnóstico seguros para compartir que conservan un estado útil y omiten el contenido personal sin procesar |
| `personal-no-fake-progress`                | Afirmaciones de finalización respaldadas por pruebas que evitan avances ficticios antes de que existan pruebas locales |
| `personal-failure-recovery`                | Recuperación ante fallos que informa del estado parcial y mantiene claros los límites de los reintentos |

Los metadatos del paquete legibles por máquina (lista de identificadores, título y descripción) se encuentran en
`extensions/qa-lab/src/scenario-packs.ts` como `QA_PERSONAL_AGENT_SCENARIO_IDS`.
Ejecute el paquete con `--pack personal-agent`:

```bash
OPENCLAW_ENABLE_PRIVATE_QA_CLI=1 pnpm openclaw qa suite \
  --provider-mode mock-openai \
  --pack personal-agent \
  --concurrency 1
```

`--pack` es acumulativo con indicadores `--scenario` repetidos. Los escenarios explícitos se ejecutan
primero y, después, los escenarios del paquete se ejecutan en el orden de `QA_PERSONAL_AGENT_SCENARIO_IDS`,
eliminando los duplicados.

El paquete está dirigido a `qa-channel` con `mock-openai` u otra vía de proveedor de QA
local. No lo dirija a servicios de chat activos ni a cuentas personales reales.

## Modelo de privacidad

Los escenarios utilizan únicamente usuarios ficticios, preferencias ficticias, secretos ficticios y el
espacio de trabajo temporal del Gateway de QA creado por el conjunto. No deben leer ni
escribir la memoria, las sesiones, las credenciales, los agentes de inicio, las configuraciones
globales ni el estado activo del Gateway de usuarios reales de OpenClaw.

Los artefactos permanecen en el directorio de artefactos existente del conjunto de QA y se tratan
como resultados de pruebas. Las comprobaciones de ocultación utilizan marcadores ficticios para que sea seguro
inspeccionar los fallos y registrarlos en incidencias.

## Ampliación del paquete

Añada nuevos casos `.yaml` en `qa/scenarios/personal/` y, después, añada el identificador del escenario
a `QA_PERSONAL_AGENT_SCENARIO_IDS`. Mantenga cada caso pequeño, local y determinista
en `mock-openai`, y centrado en un único comportamiento del asistente personal.

Buenos candidatos para continuar: comprobaciones de exportación de trayectorias ocultadas y comprobaciones
de flujos de trabajo de plugins exclusivamente locales.

Evite añadir un nuevo ejecutor, plugin, dependencia, transporte activo o evaluador de modelos
hasta que el catálogo de escenarios tenga suficientes casos estables para justificar esa superficie.
