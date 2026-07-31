---
read_when:
    - Interpretación de los resultados de la auditoría de seguridad de ClawHub
    - Decidir si instalar una skill o un plugin
    - Explicación del estado de auditoría, el nivel de riesgo o los hallazgos de ClawHub
sidebarTitle: Security Audits
summary: Cómo interpretar los resultados de la auditoría de seguridad de ClawHub antes de instalar una skill o un plugin.
title: Auditorías de seguridad
x-i18n:
    generated_at: "2026-07-26T04:32:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c4178a568c9b8e202da666ed95d2200ad73f931a22c7e473aeaba84545e8bb25
    source_path: clawhub/security-audits.md
    workflow: 16
---

# Auditorías de seguridad

Las auditorías de seguridad de ClawHub ayudan a decidir si una skill o un plugin es lo bastante seguro
para instalarlo. Muestran qué hace una versión, qué autoridad solicita y
si hay algo que merezca atención adicional antes de que pueda acceder a archivos, cuentas,
credenciales, código o servicios externos.

Las auditorías son indicadores de seguridad sólidos, pero no garantizan que una versión esté
libre de riesgos. Se debe actuar siempre con criterio antes de conceder acceso confidencial.

Véanse también [Seguridad](/es/clawhub/security), [Uso aceptable](/es/clawhub/acceptable-usage)
y [Moderación y seguridad de las cuentas](/es/clawhub/moderation).

## Qué comprobar antes de instalar

Antes de instalar, se debe revisar:

- el estado general de la auditoría
- el nivel de riesgo
- todos los hallazgos indicados
- las credenciales, los permisos o las variables de entorno requeridos
- el propietario, el origen, la versión, el registro de cambios, las descargas, las estrellas y otros indicadores de confianza

Se debe instalar únicamente contenido que se comprenda y en el que se confíe.

## Estado de la auditoría

El estado de la auditoría indica cómo reaccionar ante su resultado:

| Estado      | Significado                                                                   |
| ----------- | ------------------------------------------------------------------------- |
| `Pass`      | No se encontró ningún problema visible por encima del riesgo bajo.                                |
| `Review`    | Se deben leer los hallazgos antes de instalar. La versión aún puede ser legítima. |
| `Warn`      | Se debe extremar la precaución. ClawHub encontró un problema de gran impacto o un indicio de advertencia. |
| `Malicious` | No se debe instalar.                                                           |
| `Pending`   | Las auditorías aún no han finalizado.                                             |
| `Error`     | No se pudo completar la auditoría.                                         |

Un resultado `Pass` es tranquilizador, pero no sustituye el criterio propio. Esto es especialmente importante
en el caso de herramientas que pueden publicar contenido, editar datos, ejecutar comandos, leer archivos o
acceder a sistemas de producción.

## Nivel de riesgo

El nivel de riesgo describe el radio de impacto: cuánto poder parece tener la versión si
se usa según lo previsto.

| Nivel de riesgo | Significado                                                                       |
| ---------- | ----------------------------------------------------------------------------- |
| `Low`      | Se encontró poca autoridad confidencial o poco impacto para el usuario.                          |
| `Medium`   | La versión tiene una autoridad significativa, como acceso a cuentas o capacidad para modificar datos. |
| `High`     | La versión tiene una autoridad de gran impacto, hallazgos graves o indicios maliciosos. |

El nivel de riesgo y el estado de la auditoría responden a preguntas diferentes:

- El nivel de riesgo pregunta: «¿Cuánto poder hay aquí?»
- El estado de la auditoría pregunta: «¿Qué se debe hacer con este resultado?»

Por ejemplo, una skill de publicación puede mostrar `Review` con un riesgo `Medium`. Esto no
significa que sea maliciosa. Significa que la skill parece estar alineada con su propósito, pero puede
actuar con una autoridad significativa sobre la cuenta.

## Hallazgos

Los hallazgos explican por qué se mostró un resultado de auditoría. Cada hallazgo suele incluir:

- qué significa
- por qué se señaló
- el contenido pertinente de la skill o del plugin
- una recomendación

Los hallazgos pueden estar etiquetados como `Info`, `Low`, `Medium`, `High` o `Critical`. Los hallazgos de mayor
gravedad contribuyen en mayor medida al nivel de riesgo y al estado de la auditoría.

Los hallazgos de baja confianza se ocultan del resumen público de la auditoría para que la página
se centre en pruebas útiles.

## Qué comprueba ClawHub

ClawHub audita los artefactos de las versiones enviadas, incluidos:

- las instrucciones de la skill o los metadatos del plugin
- las variables de entorno y los permisos declarados
- las instrucciones de instalación y los metadatos del paquete
- los archivos incluidos y los manifiestos de archivos
- los metadatos de compatibilidad y capacidades

La cuestión principal es la coherencia: ¿coinciden el nombre, el resumen, los metadatos, la autoridad
solicitada y el contenido real con lo que los usuarios esperarían razonablemente?

Un comportamiento potente no es malo de forma automática. Muchas herramientas útiles necesitan credenciales,
comandos locales, API de proveedores o instalaciones de paquetes. La auditoría comprueba si ese
poder es esperado, se ha comunicado y es proporcionado.

Las páginas de los artefactos enlazan a la auditoría completa en:

```text
/<owner>/skills/<slug>/security-audit
```

La página de auditoría combina:

1. SkillSpector
2. VirusTotal
3. Análisis de riesgos

## VirusTotal

ClawHub utiliza VirusTotal como telemetría de malware en la pila de auditoría. VirusTotal es un
estándar fiable del sector para la reputación de archivos y el análisis de malware, y nuestra
colaboración permite que ClawHub aporte inteligencia de seguridad más amplia a la revisión de skills y plugins.

VirusTotal resulta especialmente útil para artefactos maliciosos conocidos, detecciones de motores e
indicadores de reputación que complementan la revisión de ClawHub orientada a agentes. Cuando están disponibles
los recuentos de los motores de proveedores, la auditoría los resume en lenguaje sencillo, por
ejemplo:

```text
62/62 proveedores marcaron esta skill como limpia.
```

o:

```text
2/64 proveedores marcaron esta skill como maliciosa, 1/64 la marcaron como sospechosa y 61/64 la marcaron como limpia.
```

Cuando ClawHub no dispone de telemetría de recuentos de proveedores que resumir, la auditoría indica:

```text
Ningún hallazgo de VirusTotal
```

VirusTotal sigue siendo telemetría. No sustituye el análisis de riesgos de ClawHub
basado en los propios artefactos.

## Análisis de riesgos

El análisis de riesgos funciona internamente mediante ClawScan, el sistema de auditorías de seguridad
propio de ClawHub. Revisa cada versión como un artefacto destinado a agentes: instrucciones,
metadatos, permisos declarados, archivos, indicadores de capacidades, indicadores de análisis estático,
hallazgos de SkillSpector, telemetría de VirusTotal y contexto proporcionado por el editor.
Los indicadores de análisis estático constituyen contexto interno para esta revisión; no son una
sección pública independiente de la auditoría ni un veredicto que bloquee la instalación.

El análisis de riesgos utiliza el
[Top 10 de skills agénticas de OWASP](https://owasp.org/www-project-agentic-skills-top-10/)
como marco para riesgos como la inyección de prompts, el uso indebido de herramientas, la exposición de credenciales,
la ejecución no segura, el envenenamiento de la memoria o del contexto y la autonomía excesiva.

ClawScan no considera que una capacidad de aspecto alarmante sea maliciosa de forma automática.
Comprueba si la capacidad se ha comunicado, está alineada con el propósito y es compatible con
el caso de uso declarado de la versión.
