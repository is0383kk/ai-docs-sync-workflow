---
read_when:
    - Un usuario informa que los agentes se quedan atascados repitiendo llamadas a herramientas
    - Necesita controlar la protección contra llamadas repetitivas
    - Estás editando las políticas de herramientas y tiempo de ejecución del agente
    - Se producen cancelaciones de `compaction_loop_persisted` después de un reintento por desbordamiento de contexto
summary: Cómo habilitar medidas de protección que detectan bucles repetitivos de llamadas a herramientas
title: Detección de bucles de herramientas
x-i18n:
    generated_at: "2026-07-26T05:32:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 79b5aa1d85e02b8cf46a95b3bcebb255178b91456517cab804cce77b8f3b818e
    source_path: tools/loop-detection.md
    workflow: 16
---

OpenClaw dispone de dos mecanismos de protección que cooperan contra patrones repetitivos de llamadas a herramientas,
ambos configurados en `tools.loopDetection`:

1. **Detección de bucles** (`enabled`) - desactivada de forma predeterminada. Supervisa el historial
   móvil de llamadas a herramientas para detectar patrones repetidos y reintentos de herramientas desconocidas.
2. **Protección posterior a la compactación** - activada siempre que
   `enabled` no sea explícitamente `false`. Se activa después de cada reintento de compactación y
   cancela la ejecución si el agente repite la misma terna `(tool, args, result)`
   dentro de la ventana.

Establezca `tools.loopDetection.enabled: false` para desactivar ambos mecanismos de protección.

## Por qué existe

- Detectar secuencias repetitivas que no producen ningún avance.
- Detectar bucles de alta frecuencia sin resultados (misma herramienta, mismas entradas, errores
  repetidos).
- Detectar patrones específicos de llamadas repetidas para herramientas de sondeo conocidas.
- Interrumpir los ciclos de desbordamiento de contexto -> compactación -> mismo bucle en lugar de permitir que
  se ejecuten indefinidamente.

## Bloque de configuración

Configuración global:

```json5
{
  tools: {
    loopDetection: {
      enabled: false, // interruptor principal de los detectores de historial móvil
    },
  },
}
```

Anulación por agente (opcional, en `agents.entries.*.tools.loopDetection`):

```json5
{
  agents: {
    list: [
      {
        id: "safe-runner",
        tools: {
          loopDetection: {
            enabled: true,
          },
        },
      },
    ],
  },
}
```

La configuración por agente anula la configuración global.

### Comportamiento del campo

| Campo     | Valor predeterminado | Efecto                                                                                            |
| --------- | ------- | ------------------------------------------------------------------------------------------------- |
| `enabled` | `false` | Interruptor principal de los detectores de historial móvil. `false` también desactiva la protección posterior a la compactación. |

Para `exec`, el hash de ausencia de avance compara resultados estables de comandos (estado,
código de salida, indicador de tiempo de espera agotado y salida) e ignora metadatos volátiles de ejecución,
como la duración, el PID, el identificador de sesión y el directorio de trabajo. Los resultados de envío de mensajes
salientes se procesan mediante hash después de eliminar los identificadores volátiles de cada llamada (identificador de mensaje, identificador de archivo y marca de tiempo),
por lo que un resultado de «enviado» no parece idéntico a otro resultado de «enviado»
diferente. Cuando hay un identificador de ejecución disponible, el historial se evalúa únicamente dentro de esa ejecución,
por lo que los ciclos programados de Heartbeat y las ejecuciones nuevas no heredan recuentos de bucles obsoletos
de ejecuciones anteriores.

## Configuración recomendada

- Para modelos más pequeños, establezca `enabled: true`. Los modelos insignia rara vez necesitan la detección mediante historial móvil y pueden
  mantener el interruptor principal en `false` mientras siguen beneficiándose de la
  protección posterior a la compactación.
- Para desactivar todo, incluida la protección posterior a la compactación, establezca
  explícitamente `tools.loopDetection.enabled: false`.

## Protección posterior a la compactación

Después de un reintento de compactación tras un desbordamiento de contexto, el ejecutor activa una
protección de ventana corta para las siguientes llamadas a herramientas. Si el agente emite la misma
terna `(toolName, argsHash, resultHash)` suficientes veces dentro de esa ventana, la protección concluye que la compactación no interrumpió el
bucle y cancela la ejecución con un error `compaction_loop_persisted`.

La protección está controlada por el indicador principal `tools.loopDetection.enabled`, con una
particularidad: permanece **activada cuando el indicador no está establecido o es `true`**, y solo se
desactiva cuando el indicador se establece explícitamente en `false`. Esto es intencionado: la protección
existe para escapar de bucles de compactación que, de otro modo, consumirían una cantidad ilimitada de tokens,
por lo que un usuario sin configuración también recibe la protección.

```json5
{
  tools: {
    loopDetection: {
      // interruptor principal; establézcalo en false para desactivar la protección junto con los detectores móviles
      enabled: true,
    },
  },
}
```

- La protección nunca cancela la ejecución mientras los resultados cambien; solo la activan los resultados
  idénticos byte por byte en toda la ventana.
- Solo se activa inmediatamente después de un reintento de compactación, no en otros
  puntos de una ejecución.

<Note>
  La protección posterior a la compactación se ejecuta siempre que el indicador principal no sea explícitamente `false`, incluso si nunca se ha escrito un bloque `tools.loopDetection`. Para verificarlo, busque `post-compaction guard armed for N attempts` en el registro del Gateway inmediatamente después de un evento de compactación.
</Note>

## Registros y comportamiento esperado

Cuando se detecta un bucle, OpenClaw registra un evento de bucle y advierte o bloquea
el siguiente ciclo de herramientas según la gravedad, lo que protege contra el consumo descontrolado de
tokens y los bloqueos sin impedir el acceso normal a las herramientas.

- Primero aparecen las advertencias.
- El bloqueo se produce cuando un patrón persiste más allá del umbral de advertencia.
- Los umbrales críticos bloquean el siguiente ciclo de herramientas y muestran un motivo claro
  de detección del bucle en el registro de ejecución.
- La protección posterior a la compactación emite errores `compaction_loop_persisted` que indican
  la herramienta responsable y el número de llamadas idénticas.

## Contenido relacionado

<CardGroup cols={2}>
  <Card title="Aprobaciones de ejecución" href="/es/tools/exec-approvals" icon="shield">
    Política de permitir/denegar la ejecución en el shell.
  </Card>
  <Card title="Niveles de razonamiento" href="/es/tools/thinking" icon="brain">
    Niveles de esfuerzo de razonamiento e interacción con la política del proveedor.
  </Card>
  <Card title="Subagentes" href="/es/tools/subagents" icon="users">
    Creación de agentes aislados para limitar comportamientos descontrolados.
  </Card>
  <Card title="Referencia de configuración" href="/es/gateway/config-tools#toolsloopdetection" icon="gear">
    Esquema completo de `tools.loopDetection` y semántica de combinación.
  </Card>
</CardGroup>
