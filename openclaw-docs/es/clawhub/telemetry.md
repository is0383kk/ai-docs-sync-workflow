---
read_when:
    - Trabajando en los controles de telemetría y privacidad
    - Preguntas sobre qué datos se recopilan
summary: Telemetría de instalación recopilada por la CLI de ClawHub y cómo desactivarla.
x-i18n:
    generated_at: "2026-07-26T05:02:03Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a02bb1c76fea3105255235f6314ade73f260f692d6eb1b41f8001dc84db6ded7
    source_path: clawhub/telemetry.md
    workflow: 16
---

# Telemetría

ClawHub utiliza telemetría mínima de la CLI para calcular los recuentos agregados de instalaciones de Skills y plugins.

## Cuándo se recopila la telemetría

La telemetría solo se envía cuando:

- Se ha iniciado sesión en la CLI.
- Se ejecuta `clawhub install <slug>` o se completa una instalación autenticada de
  `openclaw plugins install clawhub:<package>`.
- La telemetría **no está deshabilitada** (consulte «Cómo deshabilitarla» más adelante).

Si no se ha iniciado sesión, no se informa de nada.

## Qué recopilamos

Después de instalar una Skill o un plugin y conservar localmente su registro de instalación, la CLI
envía un único evento de instalación sin garantía de entrega.

El evento incluye:

- El slug de la Skill instalada o el nombre canónico del paquete del plugin.
- `version`: la versión instalada, cuando se conoce.

### Qué _no_ recopilamos

- Ninguna ruta de carpeta ni identificador derivado de carpetas.
- Ningún contenido de archivo.
- Ningún registro por ejecución, prompt ni otra salida de la CLI.

## Recuentos de instalaciones

Para las Skills, ClawHub mantiene:

- `installsAllTime`: usuarios únicos que han informado de al menos una instalación de la Skill mediante la CLI.
- `installsCurrent`: usuarios únicos que han informado de una instalación y no han eliminado sus
  datos de telemetría.

Para los plugins, ClawHub cuenta la primera instalación correcta que informa cada usuario para cada paquete.
Las instalaciones repetidas y las actualizaciones renuevan la versión registrada sin aumentar el recuento
agregado de instalaciones.

## Transparencia y controles del usuario

Solo se muestran **contadores agregados de instalaciones**.

Al eliminar la cuenta, también se eliminan los datos de telemetría y se retira su contribución de los
contadores de instalaciones.

## Cómo deshabilitar la telemetría

Establezca la variable de entorno:

```bash
export CLAWHUB_DISABLE_TELEMETRY=1
```

Con esta variable establecida, la CLI no enviará telemetría de instalaciones.
