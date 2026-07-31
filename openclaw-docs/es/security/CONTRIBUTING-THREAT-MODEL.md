---
read_when:
    - Quieres contribuir con hallazgos de seguridad o escenarios de amenazas
    - Revisión o actualización del modelo de amenazas
summary: Cómo contribuir al modelo de amenazas de OpenClaw
title: Contribuir al modelo de amenazas
x-i18n:
    generated_at: "2026-07-26T05:56:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4e2e5cd95e8a2bf5ee4bd167afedfadf9aa876e4260e2d0bfb5f414cd4255410
    source_path: security/CONTRIBUTING-THREAT-MODEL.md
    workflow: 16
---

El [modelo de amenazas](/es/security/THREAT-MODEL-ATLAS) es un documento vivo. Se aceptan contribuciones de cualquier persona; no se necesitan conocimientos de seguridad ni de MITRE ATLAS.

<Note>
Esto sirve para añadir contenido al modelo de amenazas, no para informar sobre vulnerabilidades activas. Si se ha encontrado una vulnerabilidad explotable, deben seguirse las instrucciones de divulgación responsable de la [página de confianza](https://trust.openclaw.ai).
</Note>

## Formas de contribuir

**Añadir una amenaza.** Abra una incidencia en [openclaw/trust](https://github.com/openclaw/trust/issues) y describa el escenario de ataque con sus propias palabras. Resulta útil, aunque no es obligatorio, incluir:

- El escenario de ataque y cómo podría explotarse.
- Los componentes afectados (CLI, Gateway, canales, ClawHub, servidores MCP, etc.).
- Su estimación de la gravedad (baja / media / alta / crítica).
- Enlaces a investigaciones relacionadas, CVE o ejemplos reales.

Durante la revisión, los responsables asignan la correspondencia con ATLAS, el ID de la amenaza y el nivel de riesgo.

**Sugerir una mitigación.** Abra una incidencia o un PR que haga referencia a la amenaza. La propuesta debe ser específica y aplicable: «limitación de frecuencia por remitente de 10 mensajes/minuto en el Gateway» resulta más útil que «implementar una limitación de frecuencia».

**Proponer una cadena de ataque.** Las cadenas de ataque muestran cómo se combinan varias amenazas para crear un escenario realista. Describa los pasos y cómo los encadenaría un atacante; una narración breve es preferible a una plantilla formal.

**Corregir o mejorar el contenido existente.** Erratas, aclaraciones, información desactualizada o mejores ejemplos: se aceptan PR sin necesidad de abrir una incidencia.

## Referencia del marco

Las amenazas se asocian con [MITRE ATLAS](https://atlas.mitre.org/) (panorama de amenazas adversarias para sistemas de IA), un marco para amenazas específicas de IA/ML, como la inyección de instrucciones, el uso indebido de herramientas y la explotación de agentes. No es necesario conocer ATLAS para contribuir; los responsables asocian los envíos durante la revisión.

**ID de amenazas.** Cada amenaza recibe un ID como `T-EXEC-003`, asignado por los responsables durante la revisión.

| Código  | Categoría                                             |
| ------- | ----------------------------------------------------- |
| RECON   | Reconocimiento: recopilación de información           |
| ACCESS  | Acceso inicial: obtención de acceso                    |
| EXEC    | Ejecución: realización de acciones maliciosas         |
| PERSIST | Persistencia: mantenimiento del acceso                 |
| EVADE   | Evasión de defensas: elusión de la detección           |
| DISC    | Descubrimiento: obtención de información sobre el entorno |
| EXFIL   | Exfiltración: robo de datos                            |
| IMPACT  | Impacto: daños o interrupciones                        |

**Niveles de riesgo.** Si no se tiene certeza sobre el nivel, basta con describir el impacto; los responsables lo evaluarán.

| Nivel        | Significado                                                          |
| ------------ | -------------------------------------------------------------------- |
| **Crítico**  | Compromiso total del sistema, o alta probabilidad + impacto crítico   |
| **Alto**     | Daños significativos probables, o probabilidad media + impacto crítico |
| **Medio**    | Riesgo moderado, o baja probabilidad + impacto alto                   |
| **Bajo**     | Impacto improbable y limitado                                        |

## Proceso de revisión

1. **Triaje**: los nuevos envíos se revisan en un plazo de 48 horas.
2. **Evaluación**: los responsables verifican la viabilidad, asignan la correspondencia con ATLAS y el ID de la amenaza, y validan el nivel de riesgo.
3. **Documentación**: revisión del formato y la integridad.
4. **Fusión**: se añade al modelo de amenazas y a la visualización.

## Recursos

- [Sitio web de ATLAS](https://atlas.mitre.org/)
- [Técnicas de ATLAS](https://atlas.mitre.org/techniques/)
- [Estudios de caso de ATLAS](https://atlas.mitre.org/studies/)

## Contacto

- **Vulnerabilidades de seguridad:** consulte la [página de confianza](https://trust.openclaw.ai) para obtener instrucciones sobre cómo informar de ellas, o `security@openclaw.ai`.
- **Preguntas sobre el modelo de amenazas:** abra una incidencia en [openclaw/trust](https://github.com/openclaw/trust/issues).
- **Chat general:** canal `#security` de Discord.

## Reconocimiento

Las personas que contribuyen al modelo de amenazas reciben reconocimiento en los agradecimientos del modelo de amenazas, las notas de la versión y el salón de la fama de seguridad de OpenClaw por sus contribuciones significativas.

## Contenido relacionado

- [Modelo de amenazas](/es/security/THREAT-MODEL-ATLAS)
- [Respuesta ante incidentes](/es/security/incident-response)
- [Verificación formal](/es/security/formal-verification)
