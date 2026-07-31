---
read_when:
    - Añadir compatibilidad con nodos de ubicación o una interfaz de permisos
    - Diseño de los permisos de ubicación o del comportamiento en primer plano de Android
summary: Comando de ubicación para nodos, modos de permisos de la plataforma y configuración de GeoClue en Linux
title: Comando de ubicación
x-i18n:
    generated_at: "2026-07-26T05:18:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 644229c1eafc8fc7b59bc23ba01d4ba95687ea66c4f9bd4a4cda98a87f2b6085
    source_path: nodes/location-command.md
    workflow: 16
---

## TL;DR

- `location.get` es un comando de Node, invocado mediante `node.invoke` o `openclaw nodes location get`.
- Desactivado de forma predeterminada.
- Las compilaciones de terceros para Android usan un selector: Desactivado / Mientras se usa / Siempre. Las compilaciones de Play mantienen Desactivado / Mientras se usa.
- La ubicación precisa tiene un interruptor independiente.

## Por qué un selector (y no solo un interruptor)

Los permisos de ubicación del sistema operativo tienen varios niveles. La ubicación precisa también es una concesión independiente del sistema operativo («Precisa» en iOS 14+, «fina» frente a «aproximada» en Android). El selector de la aplicación determina el modo solicitado, pero el sistema operativo sigue decidiendo la concesión real.

## Modelo de configuración

Por dispositivo Node:

- `location.enabledMode`: `off | whileUsing | always`
- `location.preciseEnabled`: bool

Comportamiento de la interfaz:

- Al seleccionar `whileUsing`, se solicita permiso en primer plano.
- Al seleccionar `always` en la compilación de terceros para Android, primero se solicita permiso en primer plano, se explica el acceso en segundo plano y, a continuación, se abre la configuración de la aplicación de Android para la concesión independiente **Allow all the time**.
- Las compilaciones de Android para Play no declaran el permiso de ubicación en segundo plano ni muestran `always`.
- Si el sistema operativo deniega el nivel solicitado, la aplicación vuelve al nivel concedido más alto y muestra el estado.

## Correspondencia de permisos (node.permissions)

Opcional. El Node de macOS informa de `location` mediante el mapa `permissions` en `node.list`/`node.describe`; iOS/Android pueden omitirlo.

## Comando: `location.get`

Se llama mediante `node.invoke` o con el asistente de la CLI:

```bash
openclaw nodes location get --node <idOrNameOrIp>
openclaw nodes location get --node <idOrNameOrIp> --accuracy precise --max-age 15000 --location-timeout 10000
```

Parámetros:

```json
{
  "timeoutMs": 10000,
  "maxAgeMs": 15000,
  "desiredAccuracy": "coarse|balanced|precise"
}
```

Las opciones de la CLI se corresponden directamente: `--location-timeout` -> `timeoutMs`, `--max-age` -> `maxAgeMs`, `--accuracy` -> `desiredAccuracy`.

Contenido de la respuesta:

```json
{
  "lat": 48.20849,
  "lon": 16.37208,
  "accuracyMeters": 12.5,
  "altitudeMeters": 182.0,
  "speedMps": 0.0,
  "headingDeg": 270.0,
  "timestamp": "2026-01-03T12:34:56.000Z",
  "isPrecise": true,
  "source": "gps|wifi|cell|unknown"
}
```

Errores (códigos estables):

- `LOCATION_DISABLED`: el selector está desactivado.
- `LOCATION_PERMISSION_REQUIRED`: falta el permiso para el modo solicitado.
- `LOCATION_BACKGROUND_UNAVAILABLE`: la aplicación está en segundo plano, pero solo se ha concedido Mientras se usa.
- `LOCATION_TIMEOUT`: no se obtuvo una ubicación a tiempo.
- `LOCATION_UNAVAILABLE`: fallo del sistema o ausencia de proveedores.

## Comportamiento en segundo plano

- Las compilaciones de terceros para Android aceptan `location.get` en segundo plano solo cuando el usuario ha seleccionado `Always` y Android ha concedido la ubicación en segundo plano. El servicio persistente de Node existente añade el tipo de servicio `location` e indica `Location: Always` mientras está activo.
- Las compilaciones de Android para Play y el modo `While Using` deniegan `location.get` cuando la aplicación está en segundo plano.
- Otras plataformas de Node pueden comportarse de forma diferente.

## Host de Node para Linux

El Plugin de Node para Linux incluido añade `location.get` al servicio `openclaw node` de la CLI, incluidos los hosts sin interfaz gráfica que no tienen la aplicación de escritorio para Linux. La ubicación está desactivada de forma predeterminada. Actívela en la entrada del Plugin y, a continuación, reinicie el servicio de Node:

```json5
{
  plugins: {
    entries: {
      "linux-node": {
        config: {
          location: { enabled: true },
        },
      },
    },
  },
}
```

Instale GeoClue2 y su demostración `where-am-i` (`geoclue-2-demo` en Debian y Ubuntu). La política de GeoClue y el agente de autorización del host deben permitir el acceso al usuario del servicio de Node.

El Plugin usa `where-am-i` en lugar de una secuencia de llamadas a `busctl`. GeoClue vincula la creación del cliente, las propiedades, el inicio, las actualizaciones y la detención a una única conexión de cliente D-Bus; la demostración mantiene unido ese ciclo de vida, mientras que los subprocesos independientes de `busctl` no lo hacen. No se añade ninguna dependencia de npm.

Linux asigna `coarse`, `balanced` y `precise` a los niveles de precisión de GeoClue `4`, `6` y `8`. Valida `maxAgeMs` con respecto a la marca de tiempo devuelta. La demostración de GeoClue no expone el proveedor seleccionado, por lo que `source` es `unknown`; `isPrecise` es verdadero solo cuando la precisión indicada es de 100 metros o mejor.

Linux utiliza los mismos errores estables: `LOCATION_DISABLED`, `LOCATION_TIMEOUT` y `LOCATION_UNAVAILABLE`.

## Integración con modelos y herramientas

- Herramienta del agente: la acción `location_get` de la herramienta `nodes` (se requiere un Node).
- CLI: `openclaw nodes location get --node <id>`.
- Directrices del agente: llamar solo cuando el usuario haya habilitado la ubicación y comprenda el alcance.

## Texto de la experiencia de usuario (sugerido)

- Desactivado: «El uso compartido de la ubicación está deshabilitado».
- Mientras se usa: «Solo cuando OpenClaw está abierto».
- Siempre: «Permitir las consultas de ubicación solicitadas mientras OpenClaw está en segundo plano».
- Precisa: «Usar la ubicación GPS precisa. Desactive esta opción para compartir la ubicación aproximada».

## Temas relacionados

- [Descripción general de los Nodes](/es/nodes)
- [Análisis de ubicaciones de canales](/es/channels/location)
- [Captura de cámara](/es/nodes/camera)
- [Modo de conversación](/es/nodes/talk)
