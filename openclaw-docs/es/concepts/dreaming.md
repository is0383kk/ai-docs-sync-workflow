---
read_when:
    - Quieres que la promoción de memoria se ejecute automáticamente
    - Quiere comprender qué hace cada fase de Dreaming
    - Quiere ajustar la consolidación sin sobrecargar MEMORY.md
sidebarTitle: Dreaming
summary: Consolidación de memoria en segundo plano con fases ligera, profunda y REM, además de un diario de sueños
title: Dreaming
x-i18n:
    generated_at: "2026-07-26T04:38:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 501ab42cfdfa0216c308896aa8c1719b06b49d64a62afdb004e097102a376eac
    source_path: concepts/dreaming.md
    workflow: 16
---

Dreaming es el sistema de consolidación de memoria en segundo plano de `memory-core`. Transfiere las señales sólidas de corto plazo a la memoria duradera, a la vez que mantiene el proceso explicable y revisable.

<Note>
Dreaming es **opcional** y está desactivado de forma predeterminada.
</Note>

## Qué escribe Dreaming

- **Estado de la máquina** en `memory/.dreams/` (almacén de recuperación, señales de fase, puntos de control de ingesta, bloqueos).
- **Salida legible por humanos** en `DREAMS.md` (o en un `dreams.md` existente) y archivos opcionales de informes de fase en `memory/dreaming/<phase>/YYYY-MM-DD.md`.

La promoción a largo plazo sigue escribiendo únicamente en `MEMORY.md`.

## Modelo de fases

Dreaming ejecuta tres fases cooperativas por cada barrido, en este orden: ligera -> REM -> profunda. Son fases internas de implementación, no modos independientes configurados por el usuario.

| Fase    | Propósito                                         | Escritura duradera       |
| ------- | ------------------------------------------------- | ------------------------ |
| Ligera  | Ordenar y preparar material reciente a corto plazo | No                       |
| REM     | Reflexionar sobre temas e ideas recurrentes       | No                       |
| Profunda | Puntuar y promover candidatos duraderos          | Sí (`MEMORY.md`)  |

<AccordionGroup>
  <Accordion title="Fase ligera">
    - Lee el estado reciente de recuperación a corto plazo, los archivos diarios de memoria y las transcripciones de sesiones censuradas cuando están disponibles.
    - Elimina señales duplicadas y prepara líneas candidatas.
    - Escribe un bloque `## Light Sleep` administrado cuando el almacenamiento incluye salida en línea.
    - Registra señales de refuerzo para la clasificación profunda posterior.
    - Nunca escribe en `MEMORY.md`.

  </Accordion>
  <Accordion title="Fase REM">
    - Genera resúmenes de temas y reflexiones a partir de trazas recientes a corto plazo.
    - Escribe un bloque `## REM Sleep` administrado cuando el almacenamiento incluye salida en línea.
    - Registra las señales de refuerzo REM utilizadas por la clasificación profunda.
    - Nunca escribe en `MEMORY.md`.

  </Accordion>
  <Accordion title="Fase profunda">
    - Clasifica candidatos mediante puntuación ponderada y umbrales de validación (`minScore`, `minRecallCount` y `minUniqueQueries` deben superarse).
    - Rehidrata fragmentos desde archivos diarios activos antes de escribir, por lo que se omiten los fragmentos obsoletos o eliminados.
    - Añade las entradas promovidas a `MEMORY.md`.
    - Escribe un resumen `## Deep Sleep` en `DREAMS.md` y, opcionalmente, en `memory/dreaming/deep/YYYY-MM-DD.md`.

  </Accordion>
</AccordionGroup>

## Ingesta de transcripciones de sesiones

Dreaming puede ingerir transcripciones de sesiones censuradas en el corpus de Dreaming. Cuando están disponibles, las transcripciones alimentan la fase ligera junto con las señales de memoria diaria y las trazas de recuperación. El contenido personal y confidencial se censura antes de la ingesta.

## Diario de sueños

Dreaming mantiene un **Diario de sueños** narrativo en `DREAMS.md`. Cuando cada fase dispone de material suficiente, `memory-core` ejecuta en segundo plano, según el mejor esfuerzo posible, un turno de subagente y añade una breve entrada al diario mediante el modelo predeterminado del entorno de ejecución, salvo que se configure `dreaming.model`. Si el modelo configurado no está disponible, la ejecución del diario vuelve a intentarse una vez con el modelo predeterminado de la sesión; los errores de confianza o de lista de permitidos no se vuelven a intentar y permanecen visibles en los registros, en lugar de recurrir silenciosamente a una entrada genérica del diario.

<Note>
El diario está destinado a la lectura humana en la interfaz de Dreams, no es una fuente de promoción. Los artefactos de diarios e informes se excluyen de la promoción a corto plazo; solo los fragmentos de memoria fundamentados pueden promoverse a `MEMORY.md`.
</Note>

También existe una vía de relleno histórico fundamentado para tareas de revisión y recuperación:

<AccordionGroup>
  <Accordion title="Comandos de relleno">
    - `memory rem-harness --path ... --grounded` muestra una vista previa de la salida fundamentada del diario a partir de notas históricas de `YYYY-MM-DD.md`.
    - `memory rem-backfill --path ...` escribe entradas fundamentadas y reversibles del diario en `DREAMS.md`.
    - `memory rem-backfill --path ... --stage-short-term` prepara candidatos duraderos fundamentados en el mismo almacén de evidencias a corto plazo que utiliza la fase profunda normal.
    - `memory rem-backfill --rollback` y `--rollback-short-term` eliminan esos artefactos de relleno preparados sin modificar las entradas normales del diario ni la recuperación activa a corto plazo.

  </Accordion>
</AccordionGroup>

La interfaz de control presenta el mismo flujo de relleno y restablecimiento del diario en la pestaña Memory del agente (página Agents), para que sea posible inspeccionar los resultados en la escena de sueños antes de decidir si los candidatos fundamentados merecen ser promovidos. Una vía diferenciada de escenas fundamentadas muestra qué entradas preparadas a corto plazo proceden de la reproducción histórica y qué elementos promovidos fueron impulsados por contenido fundamentado, y permite borrar únicamente las entradas preparadas que son exclusivamente fundamentadas sin modificar el estado activo a corto plazo.

## Señales de clasificación profunda

La clasificación profunda utiliza seis señales base ponderadas, además del refuerzo de fase:

| Señal                 | Peso | Descripción                                                     |
| --------------------- | ---- | --------------------------------------------------------------- |
| Relevancia            | 0.30 | Calidad media de recuperación de la entrada                      |
| Frecuencia            | 0.24 | Cantidad de señales a corto plazo acumuladas por la entrada      |
| Diversidad de consultas | 0.15 | Contextos distintos de consulta/día en los que apareció        |
| Actualidad            | 0.15 | Puntuación de vigencia con decaimiento temporal                  |
| Consolidación         | 0.10 | Intensidad de la recurrencia durante varios días                 |
| Riqueza conceptual    | 0.06 | Densidad de etiquetas conceptuales del fragmento o la ruta       |

Las coincidencias de las fases ligera y REM añaden un pequeño refuerzo con decaimiento temporal procedente de `memory/.dreams/phase-signals.json`.

Los resultados de pruebas paralelas pueden superponerse a la puntuación base como señal de revisión antes de cualquier escritura duradera: una prueba útil proporciona al candidato un pequeño refuerzo acotado, una prueba neutra mantiene su aplazamiento y una prueba perjudicial lo marca como rechazado para esa pasada de puntuación. Esta señal es solo informativa: puede cambiar el orden de los candidatos o los metadatos de revisión, pero nunca escribe en `MEMORY.md` ni promueve por sí sola a un candidato.

### Cobertura del informe de pruebas paralelas de QA

QA Lab incluye un escenario solo informativo para explorar cómo una futura prueba paralela de Dreaming podría revisar una memoria candidata antes de su promoción: un agente compara una respuesta de referencia con otra respuesta que puede utilizar la memoria candidata y, a continuación, escribe un informe local con un veredicto, un motivo e indicadores de riesgo. Esta cobertura se limita a QA: verifica que el artefacto del informe permanezca separado de `MEMORY.md` y que el agente nunca afirme que el candidato fue promovido. No añade un comportamiento de pruebas paralelas en producción ni modifica el motor de promoción de la fase profunda.

El ejecutor de pruebas paralelas `memory-core` mantiene el mismo contrato solo informativo para las rutas de código que necesitan un artefacto estable. Acepta el candidato, la instrucción de prueba, el resultado de referencia, el resultado del candidato, el veredicto, el motivo, los indicadores de riesgo y las referencias de evidencia, y después escribe un informe con `promotion action: report-only`. Los veredictos útiles se asignan a una recomendación `promote`, los veredictos neutros se asignan a `defer` y los veredictos perjudiciales se asignan a `reject`; ninguno de ellos escribe en `MEMORY.md` ni aplica la promoción de la fase profunda.

## Programación

Cuando está activado, `memory-core` administra automáticamente una tarea Cron para un barrido completo de Dreaming, sin duplicados entre el espacio de trabajo principal del entorno de ejecución y cualquier espacio de trabajo de agente configurado, de modo que la expansión de espacios de trabajo de subagentes no excluya el `DREAMS.md` ni el estado de memoria del agente principal.

| Ajuste                 | Valor predeterminado |
| ---------------------- | -------------------- |
| `dreaming.frequency`     | `0 3 * * *`   |
| `dreaming.model`     | modelo predeterminado |

## Inicio rápido

<Tabs>
  <Tab title="Activar Dreaming">
    ```json
    {
      "plugins": {
        "entries": {
          "memory-core": {
            "config": {
              "dreaming": {
                "enabled": true
              }
            }
          }
        }
      }
    }
    ```
  </Tab>
  <Tab title="Cadencia de barrido personalizada">
    ```json
    {
      "plugins": {
        "entries": {
          "memory-core": {
            "config": {
              "dreaming": {
                "enabled": true,
                "timezone": "America/Los_Angeles",
                "frequency": "0 */6 * * *"
              }
            }
          }
        }
      }
    }
    ```
  </Tab>
</Tabs>

## Comando con barra diagonal

```text
/dreaming status
/dreaming on
/dreaming off
/dreaming help
```

`/dreaming on` y `/dreaming off` requieren el estado de propietario para las llamadas desde canales o `operator.admin` para los clientes del Gateway. `/dreaming status` y `/dreaming help` son de solo lectura.

## Flujo de trabajo de la CLI

<Tabs>
  <Tab title="Vista previa o aplicación de la promoción">
    ```bash
    openclaw memory promote
    openclaw memory promote --apply
    openclaw memory promote --limit 5
    openclaw memory status --deep
    ```

    La operación manual `memory promote` utiliza de forma predeterminada los umbrales de la fase profunda, salvo que se sobrescriban mediante indicadores de la CLI.

  </Tab>
  <Tab title="Explicar la promoción">
    Explica por qué un candidato específico se promovería o no:

    ```bash
    openclaw memory promote-explain "router vlan"
    openclaw memory promote-explain "router vlan" --json
    ```

  </Tab>
  <Tab title="Vista previa del entorno de pruebas REM">
    Muestra una vista previa de las reflexiones REM, las verdades candidatas y la salida de promoción profunda sin escribir nada:

    ```bash
    openclaw memory rem-harness
    openclaw memory rem-harness --json
    ```

  </Tab>
</Tabs>

## Valores predeterminados clave

Todos los ajustes se encuentran en `plugins.entries.memory-core.config.dreaming`.

<ParamField path="enabled" type="boolean" default="false">
  Activa o desactiva el barrido de Dreaming.
</ParamField>
<ParamField path="frequency" type="string" default="0 3 * * *">
  Cadencia Cron del barrido completo de Dreaming.
</ParamField>
<ParamField path="model" type="string">
  Sustitución opcional del modelo del subagente del Diario de sueños. Utilice un valor canónico de `provider/model` cuando también configure una lista de permitidos `allowedModels` para subagentes.
</ParamField>
<ParamField path="phases.deep.maxPromotedSnippetTokens" type="number" default="160">
  Cantidad máxima estimada de tokens que se conserva de cada fragmento de recuperación a corto plazo promovido a `MEMORY.md`. La procedencia de la clasificación permanece visible.
</ParamField>

<Warning>
`dreaming.model` requiere `plugins.entries.memory-core.subagent.allowModelOverride: true`. Para restringirlo, configure también `plugins.entries.memory-core.subagent.allowedModels`. El reintento automático solo abarca los errores de modelo no disponible; los errores de confianza o de lista de permitidos permanecen visibles en los registros, en lugar de recurrir silenciosamente a otra opción.
</Warning>

<Note>
La mayoría de las políticas de fase, los umbrales y el comportamiento de almacenamiento son detalles internos de implementación. Consulte la [referencia de configuración de memoria](/es/reference/memory-config#dreaming) para obtener la lista completa de claves.
</Note>

## Interfaz de Dreams

Cuando está activada, la pestaña **Dreams** del Gateway muestra:

- el estado actual de activación de Dreaming
- el estado de cada fase y la presencia del barrido administrado
- los recuentos a corto plazo, fundamentados, de señales y de elementos promovidos hoy
- la hora de la próxima ejecución programada
- una vía diferenciada de escenas fundamentadas para las entradas preparadas de reproducción histórica
- un lector ampliable del Diario de sueños respaldado por `doctor.memory.dreamDiary`

## Contenido relacionado

- [Memoria](/es/concepts/memory)
- [CLI de memoria](/es/cli/memory)
- [Referencia de configuración de memoria](/es/reference/memory-config)
- [Búsqueda en la memoria](/es/concepts/memory-search)
