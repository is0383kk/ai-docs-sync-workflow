---
read_when:
    - Revisión de cargas para detectar abusos o infracciones de las políticas
    - Redacción de documentación de moderación o guías operativas para revisores
    - Decidir si una skill debe ocultarse o si se debe bloquear a un usuario
sidebarTitle: Acceptable Usage
summary: 'Política del marketplace: qué permite ClawHub y qué no alojará.'
title: Uso aceptable
x-i18n:
    generated_at: "2026-07-26T04:31:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ace357e7a3e9f4d242f113ad791b254e94ae8a841dd9a864a77c5bac15713132
    source_path: clawhub/acceptable-usage.md
    workflow: 16
---

# Uso aceptable

ClawHub aloja Skills, plugins, paquetes y metadatos del mercado para OpenClaw.
Utilice esta página para decidir si el contenido o el comportamiento de publicación corresponde a
ClawHub.

Estas reglas se aplican a lo que hace una publicación, lo que pide ejecutar a los usuarios, cómo se
presenta y cómo utilizan los editores las funciones de descubrimiento, instalación y
confianza de ClawHub. Para conocer los estados de moderación y la situación de las cuentas, consulte
[Moderación y seguridad de las cuentas](/es/clawhub/moderation). Para reclamaciones de derechos de autor u otros derechos,
consulte [Solicitudes de derechos sobre el contenido](/es/clawhub/content-rights).

## Contenido permitido

ClawHub acepta contenido útil, comprensible y publicado de
buena fe.

| Categoría                                         | Permitido cuando                                                                                                                      |
| ------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------- |
| Productividad de los desarrolladores                           | La publicación ayuda a los usuarios a crear, probar, migrar, depurar, documentar u operar software.                                               |
| Flujos de trabajo de interfaz de usuario, datos y automatización               | El alcance es claro, las credenciales necesarias son explícitas y las acciones de riesgo incluyen procesos de revisión, simulación, vista previa o confirmación. |
| Seguridad defensiva, moderación y revisión de abusos | La herramienta se presenta para revisiones autorizadas, conserva las pruebas y mantiene claros los límites de aprobación humana.                          |
| Flujos de trabajo personales o de equipo                       | El flujo de trabajo utiliza cuentas basadas en el consentimiento, una configuración transparente y permisos explícitos.                                            |
| Catálogos mantenidos                              | Cada publicación es distinta, útil, está descrita con precisión y recibe un mantenimiento razonable.                                                |

El contexto importa. Un mismo tema puede ser aceptable en un entorno defensivo limitado o
basado en el consentimiento e inaceptable cuando se presenta como un flujo de trabajo para cometer abusos.

## Contenido no permitido

ClawHub no aloja contenido cuyo propósito principal sea el abuso, el engaño, la ejecución
insegura o la vulneración de derechos.

| Categoría                                                    | No permitido                                                                                                                                                                                                                                                                                                   |
| ----------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Acceso no autorizado o elusión de la seguridad                      | Elusión de la autenticación, apropiación de cuentas, abuso de límites de frecuencia, apropiación de llamadas en vivo o agentes, robo de sesiones reutilizables o aprobación automática de procesos de vinculación para usuarios no autorizados.                                                                                                                                                   |
| Abuso de plataformas y evasión de prohibiciones                              | Cuentas encubiertas después de una prohibición, preparación o explotación masiva de cuentas, interacción falsa, automatización de múltiples cuentas, publicación masiva, bots de spam o automatización creada para evitar la detección.                                                                                                                                          |
| Fraude, estafas y flujos de trabajo financieros engañosos             | Certificados o facturas falsos, flujos de pago engañosos, contacto para estafas, pruebas sociales falsas, flujos de trabajo con identidades sintéticas para cometer fraudes o herramientas para gastar o efectuar cargos sin una aprobación humana clara.                                                                                                                    |
| Enriquecimiento invasivo de la privacidad o vigilancia                 | Extracción de contactos para enviar spam, divulgación de datos personales, acoso, extracción de clientes potenciales combinada con contacto no solicitado, supervisión encubierta, comparación biométrica sin consentimiento o uso de datos filtrados o volcados de brechas de seguridad.                                                                                                                  |
| Suplantación o manipulación de identidad sin consentimiento       | Intercambio de rostros, gemelos digitales, influentes clonados, identidades falsas u otras herramientas utilizadas para suplantar o engañar.                                                                                                                                                                                                 |
| Contenido sexual explícito o generación para adultos con la seguridad desactivada | Generación de imágenes, vídeos o contenido NSFW; envoltorios de contenido para adultos en torno a API de terceros; o publicaciones cuyo propósito principal sea el contenido sexual explícito.                                                                                                                                                       |
| Requisitos de ejecución ocultos, inseguros o engañosos        | Comandos de instalación ofuscados, instaladores que canalizan contenido al intérprete de comandos, como contenido descargado ejecutado con `sh` o `bash` sin que pueda revisarse claramente, requisitos no declarados de secretos o claves privadas, ejecución remota de `npx @latest` sin que pueda revisarse claramente o metadatos que ocultan lo que la publicación necesita realmente para ejecutarse. |
| Material que infringe derechos de autor u otros derechos           | Republicar Skills, plugins, documentación, recursos de marca o código propietario de otra persona sin permiso; infringir los términos de una licencia; o suplantar al autor o editor original.                                                                                                                            |

## Comportamiento no permitido en el mercado

ClawHub también revisa cómo utilizan los editores el mercado. No utilice ClawHub para
manipular el descubrimiento, las métricas, las señales de confianza, los sistemas de moderación ni la
atención de los usuarios.

El comportamiento no permitido en el mercado incluye:

- publicar de forma masiva grandes cantidades de publicaciones de poco esfuerzo, duplicadas, provisionales o
  generadas automáticamente que no parezcan aportar valor real a los usuarios
- saturar las superficies de búsqueda o categorías con Skills o plugins casi idénticos
- publicar cientos de publicaciones con poco o ningún uso, mantenimiento, claridad sobre el código fuente
  o diferenciación significativa
- aumentar artificialmente las instalaciones, descargas, estrellas u otras métricas de
  interacción mediante automatización, ciclos de autoinstalación, cuentas falsas, actividad
  coordinada, interacción pagada u otros comportamientos no orgánicos
- crear o rotar cuentas para eludir la moderación, las prohibiciones, los límites para editores o la
  revisión del mercado
- engañar a los usuarios sobre la propiedad, el código fuente, las capacidades, la postura de seguridad,
  los requisitos de instalación o la afiliación con otro proyecto o editor
- subir repetidamente contenido que ya se haya ocultado, eliminado o bloqueado
  sin corregir el problema subyacente

La publicación de grandes volúmenes no constituye automáticamente un abuso. Los catálogos grandes son aceptables
cuando las publicaciones presentan diferencias significativas, están descritas con precisión, reciben mantenimiento
y las utilizan usuarios reales. Los catálogos grandes se convierten en un problema de confianza y seguridad cuando
el volumen se combina con publicaciones de escaso contenido, duplicadas, engañosas, sin mantenimiento o
promocionadas artificialmente.

## Derechos sobre el contenido

Si considera que algún contenido de ClawHub infringe sus derechos de autor u otros derechos, utilice
[Solicitudes de derechos sobre el contenido](/es/clawhub/content-rights). No utilice los informes normales del mercado
para reclamaciones de derechos de autor u otros derechos, salvo que la publicación también sea insegura,
maliciosa o engañosa.

## Revisión y aplicación de medidas

ClawHub puede utilizar comprobaciones automatizadas, señales estadísticas de abuso, informes de usuarios y
revisiones del personal para identificar contenido inseguro o comportamientos de publicación abusivos. Una señal
no demuestra por sí sola que exista abuso; ayuda a ClawHub a decidir qué debe revisarse.

Podemos:

- ocultar, retener, retirar, eliminar de forma reversible o, cuando el tipo de recurso lo permita,
  eliminar de forma permanente las publicaciones que infrinjan las reglas
- bloquear las descargas o instalaciones de versiones inseguras
- revocar tokens de API
- eliminar de forma reversible el contenido asociado
- restringir el acceso a la publicación
- prohibir el acceso a los infractores reincidentes o graves

No garantizamos que se emita una advertencia antes de aplicar medidas contra abusos evidentes. Consulte
[Moderación y seguridad de las cuentas](/es/clawhub/moderation) para obtener información sobre informes, retenciones por moderación,
publicaciones ocultas, prohibiciones y situación de las cuentas.
