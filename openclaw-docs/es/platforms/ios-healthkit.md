---
read_when:
    - Activación de resúmenes de HealthKit en un nodo de iOS
    - Invocación de health.summary o solución de problemas de métricas de estado ausentes
    - Revisión de qué datos de salud pueden salir de un dispositivo iOS
summary: Activar e invocar resúmenes de HealthKit sujetos a controles de privacidad desde un Node de iOS
title: Resúmenes de HealthKit
x-i18n:
    generated_at: "2026-07-26T04:45:09Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b8ac13d2870c55e2083a5e3a14c3d04238c2780a9e83d091f31923eb738476af
    source_path: platforms/ios-healthkit.md
    workflow: 16
---

# Resúmenes de HealthKit

OpenClaw puede solicitar un resumen de solo lectura del día natural actual a un
nodo iPhone o iPad conectado. El dispositivo calcula el agregado en el propio dispositivo y devuelve
únicamente los pasos, la duración del sueño, la frecuencia cardíaca media en reposo y el
número y la duración de los entrenamientos. No se admiten muestras individuales de HealthKit,
fuentes, metadatos, historiales clínicos, ingesta en segundo plano ni operaciones
de escritura.

Esta función está desactivada de forma predeterminada. Requiere un consentimiento independiente en el dispositivo iOS y
autorización en el Gateway.

## Requisitos

- Un iPhone o iPad que ejecute la aplicación de OpenClaw para iOS y en el que HealthKit indique que los datos de salud están
  disponibles.
- Un nodo iOS conectado y aprobado. Consulte [Configuración de la aplicación para iOS](/es/platforms/ios).
- Un Gateway actualizado que pueda comunicarse con el nodo iOS.
- Datos de Salud legibles para todas las métricas que se espera ver. Un Apple Watch puede
  aportar datos al repositorio de Apple Health, pero la aplicación de OpenClaw para watchOS
  no es necesaria para los resúmenes de HealthKit.

## Habilitar el acceso

### 1. Autorizar el comando del Gateway

Añada `health.summary` al array `gateway.nodes.commands.allow` existente en
`openclaw.json`. Conserve todos los comandos que ya estén presentes:

```json5
{
  gateway: {
    nodes: {
      commands: { allow: ["health.summary"] },
    },
  },
}
```

`health.summary` está clasificado como altamente sensible para la privacidad y nunca está permitido de forma predeterminada por la
plataforma iOS. Una entrada en `gateway.nodes.commands.deny` prevalece sobre la
entrada de permiso. Consulte [Política de comandos de Node](/es/nodes#command-policy).

### 2. Habilitar el uso compartido en el dispositivo iOS

En la aplicación para iOS:

1. Abra **Settings -> Permissions** y busque **Apple Health Summaries** en la sección
   **Apple Health**, siempre visible.
2. Pulse **Enable Apple Health Summaries**.
3. Lea la información y, a continuación, elija qué categorías de Salud puede leer OpenClaw
   en la hoja de permisos de Apple.

El interruptor registra la elección explícita de compartir con OpenClaw. No implica
que Apple haya concedido acceso a todas las categorías solicitadas.

Al habilitar los resúmenes de Salud, se añade `health.summary` a la superficie de comandos declarada
por el nodo. Apruebe la actualización de emparejamiento del nodo resultante:

```bash
openclaw nodes pending
openclaw nodes approve <requestId>
```

A continuación, compruebe que el dispositivo iOS conectado exponga un comando `health.summary`
efectivo:

```bash
openclaw nodes describe --node "<iOS device name>"
```

## Solicitar el resumen de hoy

Solo se admite `today`. Abarca desde la medianoche local hasta el momento de la solicitud,
según el calendario y la zona horaria actuales del dispositivo iOS.

```bash
openclaw nodes invoke \
  --node "<iOS device name>" \
  --command health.summary \
  --params '{"period":"today"}' \
  --json
```

Los agentes pueden ejecutar el mismo comando con la herramienta `nodes`:

```json
{
  "action": "invoke",
  "node": "<iOS device name>",
  "invokeCommand": "health.summary",
  "invokeParamsJson": "{\"period\":\"today\"}"
}
```

La carga útil del resumen contiene:

| Campo                    | Significado                                       |
| ------------------------ | --------------------------------------------- |
| `period`                 | Siempre `today`                                |
| `startISO`               | Inicio local del día, codificado como instante ISO |
| `endISO`                 | Hora de la solicitud, codificada como instante ISO       |
| `timeZoneIdentifier`     | Identificador de zona horaria del dispositivo iOS               |
| `stepCount`              | Pasos acumulados redondeados                      |
| `sleepDurationMinutes`   | Tiempo de sueño sin duplicados, limitado al día de hoy    |
| `restingHeartRateBpm`    | Frecuencia cardíaca media en reposo                    |
| `workoutCount`           | Entrenamientos que comenzaron hoy                   |
| `workoutDurationMinutes` | Duración total de esos entrenamientos              |

Los campos de métricas son opcionales y se omiten cuando HealthKit no devuelve ningún
valor legible. Las fases del sueño y las fuentes superpuestas se combinan antes de
calcular la duración, por lo que un mismo minuto no se cuenta dos veces.

## Comportamiento de privacidad

- La agregación se realiza en el dispositivo iOS. Las muestras sin procesar no salen del dispositivo.
- El agregado solicitado sale del dispositivo a través del Gateway. Cuando un agente
  lo solicita, el agregado llega al proveedor de IA configurado y puede permanecer
  en el historial del chat. Una invocación directa desde la CLI lo devuelve al operador de la CLI.
- OpenClaw solo solicita acceso de lectura. No puede añadir ni modificar datos de Salud.
- OpenClaw solo lee HealthKit cuando se invoca `health.summary`. No hay
  ingesta de datos de salud en segundo plano.
- HealthKit no revela deliberadamente si se ha denegado el acceso de lectura. La
  ausencia de una métrica puede indicar acceso denegado, que no hay muestras coincidentes o que el
  tipo de datos no está disponible. OpenClaw no puede distinguir entre esos casos.
- El resumen proporciona contexto personal de salud y actividad física, no un diagnóstico ni
  asesoramiento médico.

Para dejar de compartir, vuelva a **Apple Health Summaries** y pulse **Turn Off Summaries**.
El dispositivo iOS eliminará entonces la capacidad de Salud y el comando `health.summary` de la superficie
del nodo. También puede eliminar `health.summary` de
`gateway.nodes.commands.allow` para cerrar el acceso desde el Gateway.

## Solución de problemas

### El nodo no declara el comando

Confirme que los resúmenes de Apple Health estén habilitados en la aplicación para iOS y que el dispositivo esté conectado.
Ejecute `openclaw nodes pending` y apruebe cualquier actualización de capacidades; a continuación, vuelva a inspeccionar
`openclaw nodes describe --node "<iOS device name>"`.

### El comando requiere una activación explícita

Añada `health.summary` a `gateway.nodes.commands.allow`. Compruebe también que
`gateway.nodes.commands.deny` no lo contenga; la lista de denegación prevalece.

### `HEALTH_ACCESS_DISABLED`

El interruptor de uso compartido de la aplicación está desactivado. Habilite **Apple Health Summaries** en
**Settings -> Permissions -> Apple Health** en el dispositivo iOS.

### El resumen se obtiene correctamente, pero faltan métricas

Abra la aplicación Salud de Apple y confirme que existan datos para hoy. Revise
el acceso de OpenClaw en los ajustes de Salud de Apple, pero no interprete un resultado vacío
como prueba de que se denegó el acceso: HealthKit oculta intencionadamente esa distinción.

### Los intervalos anteriores fallan

El comando solo acepta `{"period":"today"}`. No se admiten resúmenes
de varios días ni históricos.

## Temas relacionados

- [Aplicación para iOS](/es/platforms/ios)
- [Nodes](/es/nodes)
- [Referencia de configuración del Gateway](/es/gateway/configuration-reference#gateway)
- [Auditoría de seguridad](/es/gateway/security)
