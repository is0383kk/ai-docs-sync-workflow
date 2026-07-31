---
read_when:
    - Se busca un modelo mental rápido para gestionar las zonas horarias
    - Se está decidiendo dónde establecer o sobrescribir una zona horaria
summary: 'Dónde aparecen las zonas horarias en OpenClaw: sobres, cargas útiles de herramientas y prompt del sistema'
title: Zonas horarias
x-i18n:
    generated_at: "2026-07-26T04:37:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9d1620b4b2cedba89bd6ab4392018cd48d0ef92a6abc1744011d482557e2c4fc
    source_path: concepts/timezone.md
    workflow: 16
---

OpenClaw estandariza las marcas de tiempo para que el modelo vea una **única hora de referencia** en lugar de una mezcla de relojes locales de los proveedores. Tres superficies muestran zonas horarias, cada una con su propio propósito:

## Tres superficies de zonas horarias

| Superficie            | Qué muestra                                                                                                            | Valor predeterminado                                    | Configuración mediante                                  |
| --------------------- | ---------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------- | ------------------------------------------------------- |
| Sobres de mensajes    | Envuelve los mensajes entrantes de los canales: `[Signal +1555 Sun 2026-01-18 00:19:42 PST] hello`                                                     | Hora local del host                                     | `agents.defaults.envelopeTimezone`                                      |
| Cargas útiles de herramientas | Las herramientas de canal del tipo `readMessages` devuelven la hora sin procesar del proveedor junto con `timestampMs` / `timestampUtc` normalizados | Los campos UTC siempre están presentes                  | No es configurable; conserva las marcas de tiempo nativas del proveedor |
| Prompt del sistema    | Un pequeño bloque `Current Date & Time` con **solo la zona horaria** (sin valor de reloj, para mantener estable la caché) | Zona horaria del host si `userTimezone` no está definido | `agents.defaults.userTimezone`                                      |

El prompt del sistema omite deliberadamente la hora actual para mantener estable la caché de prompts entre turnos. Cuando el agente necesita la hora actual, llama a `session_status`.

## Configuración de la zona horaria del usuario

```json5
{
  agents: {
    defaults: {
      userTimezone: "America/Chicago",
    },
  },
}
```

Si `userTimezone` no está definido, OpenClaw determina la zona horaria del host en tiempo de ejecución mediante `Intl.DateTimeFormat().resolvedOptions().timeZone` (sin escribir en la configuración). `agents.defaults.timeFormat` (`auto` | `12` | `24`) controla la representación de 12 h/24 h en los sobres y las superficies posteriores, no en la sección del prompt del sistema.

## Valores de zona horaria de los sobres

`agents.defaults.envelopeTimezone` acepta:

- `"local"` (valor predeterminado) o `"host"`: la zona horaria de la máquina host.
- `"utc"` o `"gmt"`: UTC.
- `"user"`: el valor resuelto de `agents.defaults.userTimezone` (si no está definido, se utiliza la zona horaria del host).
- Cualquier cadena de zona IANA explícita, por ejemplo, `"Europe/Vienna"`.

## Cuándo sustituir el valor predeterminado

- **Use `"utc"`** para obtener marcas de tiempo estables entre hosts de distintas regiones o para que coincidan con la salida de diagnósticos y registros alineada con UTC.
- **Use `"user"`** para mantener los sobres alineados con la zona horaria configurada para el usuario, independientemente de la zona en la que se ejecute el host del Gateway.
- **Use una zona IANA fija** cuando el host del Gateway esté en una zona, pero el sobre deba mostrarse siempre en otra, independientemente de las migraciones del host.
- **Establezca `envelopeTimestamp: "off"`** cuando el contexto de las marcas de tiempo no sea útil para la conversación. Esto elimina las marcas de tiempo absolutas de los sobres, los prefijos de prompts enviados directamente al agente y los prefijos integrados en la entrada del modelo.

Para consultar la referencia completa del comportamiento, ejemplos por proveedor y el formato del tiempo transcurrido, consulte [Fecha y hora](/es/date-time).

## Contenido relacionado

- [Fecha y hora](/es/date-time): comportamiento completo de sobres, herramientas y prompts, además de ejemplos.
- [Heartbeat](/es/gateway/heartbeat): las horas activas usan la zona horaria para la programación.
- [Trabajos de Cron](/es/automation/cron-jobs): las expresiones cron usan la zona horaria para la programación.
