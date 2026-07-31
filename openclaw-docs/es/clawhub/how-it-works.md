---
read_when:
    - Comprender los listados, las versiones, las instalaciones, la publicación y la moderación
summary: Cómo funcionan los listados, las versiones, las instalaciones, la publicación, los análisis y las actualizaciones de ClawHub.
x-i18n:
    generated_at: "2026-07-26T05:33:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 747079343899e42d00f84b00c553447abe0b83f2c4f1c9cdbf54725e34779eaf
    source_path: clawhub/how-it-works.md
    workflow: 16
---

# Cómo funciona ClawHub

ClawHub es la capa de registro para las Skills y los plugins de OpenClaw. Proporciona a los usuarios un
lugar donde descubrir paquetes, a los publicadores un lugar donde publicar versiones y
a OpenClaw los metadatos suficientes para instalar y actualizar esos paquetes de forma segura.

## Registros del registro

Cada ficha pública es un registro del registro que contiene:

- un propietario y un slug o nombre de paquete
- una o más versiones publicadas
- metadatos, resumen, archivos y atribución de la fuente
- registro de cambios e información de etiquetas, como `latest`
- señales de descargas, instalaciones y estrellas
- estado del análisis de seguridad y la moderación

La página de la ficha es el lugar canónico donde los usuarios pueden examinar lo que una Skill o un
plugin afirma hacer antes de instalarlo.

## Skills

Una Skill es un paquete de texto con versiones centrado en `SKILL.md`. Puede incluir
archivos auxiliares, ejemplos, plantillas y scripts.

ClawHub lee el frontmatter de `SKILL.md` para conocer el nombre de la Skill,
su descripción, requisitos, variables de entorno y metadatos. La precisión de los
metadatos es importante porque ayuda a los usuarios a decidir si deben instalar la Skill y
ayuda a los análisis automatizados a detectar discrepancias entre el comportamiento declarado y el observado.

Consulte [Formato de las Skills](/es/clawhub/skill-format).

## Plugins

Los plugins son extensiones empaquetadas de OpenClaw. ClawHub almacena los metadatos de los paquetes,
la información de compatibilidad, los enlaces a las fuentes, los artefactos y los registros de versiones.

Cuando OpenClaw instala un plugin desde ClawHub, comprueba los metadatos de compatibilidad
anunciados antes de instalarlo. Los registros de paquetes pueden incluir compatibilidad con la API,
versión mínima del Gateway, hosts de destino, requisitos del entorno y resúmenes
criptográficos de los artefactos.

Utilice una fuente de instalación explícita de ClawHub cuando quiera que el registro sea la
fuente de referencia:

```bash
openclaw plugins install clawhub:<package>
```

## Publicación

La publicación crea un nuevo registro de versión inmutable. Los publicadores utilizan la CLI
`clawhub` para los flujos de trabajo autenticados del registro:

```bash
clawhub skill publish ./my-skill
clawhub package publish <source> --family code-plugin --dry-run
clawhub package publish <source> --family code-plugin
```

Utilice ejecuciones de prueba para previsualizar la carga útil resuelta antes de subirla. A continuación, las páginas
públicas muestran los metadatos publicados, los archivos, la atribución de la fuente y el estado del análisis.

## Instalaciones y actualizaciones

Los comandos de instalación de OpenClaw utilizan ClawHub como fuente de paquetes:

```bash
openclaw skills install @openclaw/demo
openclaw plugins install clawhub:<package>
```

OpenClaw registra los metadatos de la fuente de instalación para que las actualizaciones puedan resolver posteriormente el mismo
paquete del registro. La CLI de ClawHub también admite flujos de trabajo de instalación y
actualización directa de Skills para los usuarios que quieran carpetas de Skills gestionadas por el registro fuera de un
espacio de trabajo completo de OpenClaw.

## Estado de seguridad

ClawHub permite la publicación abierta, pero las versiones siguen estando sujetas a controles de subida,
comprobaciones automatizadas, informes de los usuarios y medidas de los moderadores.

Las páginas públicas muestran resúmenes de los análisis cuando están disponibles. El contenido retenido, oculto
o bloqueado puede desaparecer de la búsqueda pública y de los flujos de instalación, aunque siga siendo
visible para el propietario con fines de diagnóstico.

Consulte [Seguridad](/es/clawhub/security), [Auditorías de seguridad](/es/clawhub/security-audits),
[Moderación y seguridad de las cuentas](/es/clawhub/moderation) y
[Uso aceptable](/es/clawhub/acceptable-usage).

## Acceso a la API

ClawHub ofrece API públicas de lectura para el descubrimiento, la búsqueda, los detalles de los paquetes y las
descargas. Los catálogos de terceros pueden utilizar estas API si incluyen un enlace a la
ficha canónica de ClawHub, respetan los límites de frecuencia y evitan dar a entender que cuentan con su respaldo.

Consulte [API pública](/es/clawhub/api) y [API HTTP](/es/clawhub/http-api).
