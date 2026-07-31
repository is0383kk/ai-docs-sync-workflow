---
permalink: /security/formal-verification/
read_when:
    - Revisión de las garantías o limitaciones del modelo formal de seguridad
    - Reproducción o actualización de las comprobaciones del modelo de seguridad de TLA+/TLC
summary: Modelos de seguridad verificados automáticamente para las rutas de mayor riesgo de OpenClaw.
title: Verificación formal (modelos de seguridad)
x-i18n:
    generated_at: "2026-07-26T04:51:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 185ee5c1cff7325f10827330c0c7e55ddc3ca40caf6088d4c930ae5e090d6b27
    source_path: security/formal-verification.md
    workflow: 16
---

Los modelos formales de seguridad de OpenClaw (TLA+/TLC actualmente) proporcionan un argumento verificado por máquina de que determinadas rutas de máximo riesgo —autorización, aislamiento de sesiones, control de acceso a herramientas y seguridad ante configuraciones incorrectas— aplican la política prevista, bajo supuestos explícitos.

> Nota: algunos enlaces antiguos pueden hacer referencia al nombre anterior del proyecto.

## Qué es esto

Un conjunto ejecutable de pruebas de regresión de seguridad orientadas a atacantes:

- Cada afirmación cuenta con una comprobación de modelo ejecutable sobre un espacio de estados finito.
- Muchas afirmaciones cuentan con un modelo negativo asociado que genera una traza de contraejemplo para una clase realista de errores.

Esto **no** demuestra que OpenClaw sea seguro en todos los aspectos ni verifica la implementación completa en TypeScript.

## Dónde se encuentran los modelos

Los modelos se mantienen en un repositorio independiente: [vignesh07/openclaw-formal-models](https://github.com/vignesh07/openclaw-formal-models).

<Note>
Actualmente no se puede acceder a ese repositorio (GitHub devuelve "Repository not found" en el momento de redactar este documento). Si sigue sin funcionar, consulte en los canales de mantenedores de OpenClaw cuál es la ubicación actual antes de suponer que los modelos se eliminaron.
</Note>

## Consideraciones

- Estos son modelos, no la implementación completa en TypeScript; es posible que existan divergencias entre el modelo y el código.
- Los resultados están limitados por el espacio de estados que explora TLC. Un resultado satisfactorio no implica seguridad más allá de los supuestos y límites modelados.
- Algunas afirmaciones dependen de supuestos explícitos sobre el entorno (por ejemplo, un despliegue correcto y entradas de configuración correctas).

## Reproducción de los resultados

Clone el repositorio de modelos y ejecute TLC:

```bash
git clone https://github.com/vignesh07/openclaw-formal-models
cd openclaw-formal-models

# Se requiere Java 11+ (TLC se ejecuta en la JVM).
# El repositorio incluye una versión fijada de tla2tools.jar y proporciona bin/tlc, además de objetivos de Make.

make <target>
```

Todavía no existe una integración de CI con este repositorio; una futura iteración podría añadir modelos ejecutados mediante CI con artefactos públicos (trazas de contraejemplos y registros de ejecución) o un flujo de trabajo alojado de «ejecutar este modelo» para comprobaciones acotadas pequeñas.

## Afirmaciones y objetivos

### Exposición del Gateway y configuración incorrecta de un Gateway abierto

**Afirmación:** vincular más allá de la interfaz de bucle local sin autenticación puede posibilitar una vulneración remota y aumentar la exposición; según los supuestos del modelo, un token o una contraseña bloquean a los atacantes no autenticados.

| Resultado         | Objetivos                                                          |
| -------------- | ---------------------------------------------------------------- |
| Satisfactorio          | `make gateway-exposure-v2`, `make gateway-exposure-v2-protected` |
| Fallido (esperado) | `make gateway-exposure-v2-negative`                              |

Consulte también `docs/gateway-exposure-matrix.md` en el repositorio de modelos.

### Pipeline de ejecución de Node (capacidad de máximo riesgo)

**Afirmación:** `exec host=node` requiere (a) una lista de comandos de Node permitidos junto con los comandos declarados y (b) aprobación en tiempo real cuando esté configurada; en el modelo, las aprobaciones se tokenizan para evitar su reutilización.

| Resultado         | Objetivos                                                         |
| -------------- | --------------------------------------------------------------- |
| Satisfactorio          | `make nodes-pipeline`, `make approvals-token`                   |
| Fallido (esperado) | `make nodes-pipeline-negative`, `make approvals-token-negative` |

### Almacén de emparejamiento (control de acceso a mensajes directos)

**Afirmación:** las solicitudes de emparejamiento respetan el TTL y los límites de solicitudes pendientes.

| Resultado         | Objetivos                                              |
| -------------- | ---------------------------------------------------- |
| Satisfactorio          | `make pairing`, `make pairing-cap`                   |
| Fallido (esperado) | `make pairing-negative`, `make pairing-cap-negative` |

### Control de acceso de entrada (menciones y elusión mediante comandos de control)

**Afirmación:** en contextos de grupo que requieran una mención, un comando de control no autorizado no puede eludir el control de acceso mediante menciones.

| Resultado         | Objetivos                        |
| -------------- | ------------------------------ |
| Satisfactorio          | `make ingress-gating`          |
| Fallido (esperado) | `make ingress-gating-negative` |

### Enrutamiento y aislamiento de claves de sesión

**Afirmación:** los mensajes directos de interlocutores distintos no se agrupan en la misma sesión, salvo que estén vinculados o configurados explícitamente.

| Resultado         | Objetivos                           |
| -------------- | --------------------------------- |
| Satisfactorio          | `make routing-isolation`          |
| Fallido (esperado) | `make routing-isolation-negative` |

## Modelos v1++: concurrencia, reintentos y corrección de trazas

Modelos posteriores que mejoran la fidelidad en torno a modos de fallo reales: actualizaciones no atómicas, reintentos y distribución de mensajes.

### Concurrencia e idempotencia del almacén de emparejamiento

**Afirmación:** el almacén de emparejamiento aplica `MaxPending` y la idempotencia incluso cuando las operaciones se intercalan: la comprobación seguida de escritura debe ser atómica o estar bloqueada, y la actualización no debe crear duplicados. En concreto: las solicitudes simultáneas no pueden superar `MaxPending` para un canal, y las solicitudes o actualizaciones repetidas para el mismo `(channel, sender)` no crean filas pendientes activas duplicadas.

| Resultado         | Objetivos                                                                                                                                                                     |
| -------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Satisfactorio          | `make pairing-race` (comprobación atómica o bloqueada del límite), `make pairing-idempotency`, `make pairing-refresh`, `make pairing-refresh-race`                                              |
| Fallido (esperado) | `make pairing-race-negative` (condición de carrera por límite entre inicio y confirmación no atómicos), `make pairing-idempotency-negative`, `make pairing-refresh-negative`, `make pairing-refresh-race-negative` |

### Correlación e idempotencia de trazas de entrada

**Afirmación:** la ingesta conserva la correlación de trazas durante la distribución y es idempotente ante los reintentos del proveedor. Cuando un evento externo se convierte en varios mensajes internos, cada parte conserva la misma identidad de traza o evento; los reintentos no provocan un procesamiento duplicado; si faltan los identificadores de evento del proveedor, la deduplicación utiliza como alternativa una clave segura (por ejemplo, el identificador de traza) para evitar descartar eventos distintos.

| Resultado         | Objetivos                                                                                                                                     |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| Satisfactorio          | `make ingress-trace`, `make ingress-trace2`, `make ingress-idempotency`, `make ingress-dedupe-fallback`                                     |
| Fallido (esperado) | `make ingress-trace-negative`, `make ingress-trace2-negative`, `make ingress-idempotency-negative`, `make ingress-dedupe-fallback-negative` |

### Precedencia de dmScope en el enrutamiento e identityLinks

**Afirmación:** la precedencia de `dmScope` y los vínculos de identidad se comportan de forma determinista: el ámbito predeterminado `main` comparte una única sesión continua entre los mensajes directos de un solo propietario (la configuración predeterminada del agente personal), mientras que cualquier ámbito de aislamiento configurado (`per-peer`, `per-channel-peer`, `per-account-channel-peer`) mantiene las sesiones de mensajes directos estrictamente separadas. Las anulaciones de `dmScope` específicas del canal prevalecen sobre los valores predeterminados globales; `identityLinks` agrupa sesiones únicamente dentro de grupos vinculados explícitamente, no entre interlocutores no relacionados. Se espera que las bandejas de entrada multiusuario adopten un ámbito de aislamiento (la auditoría de seguridad del entorno de ejecución lo recomienda cuando detecta tráfico de mensajes directos de varios usuarios).

| Resultado         | Objetivos                                                                   |
| -------------- | ------------------------------------------------------------------------- |
| Satisfactorio          | `make routing-precedence`, `make routing-identitylinks`                   |
| Fallido (esperado) | `make routing-precedence-negative`, `make routing-identitylinks-negative` |

## Contenido relacionado

- [Modelo de amenazas](/es/security/THREAT-MODEL-ATLAS)
- [Contribución al modelo de amenazas](/es/security/CONTRIBUTING-THREAT-MODEL)
- [Respuesta ante incidentes](/es/security/incident-response)
