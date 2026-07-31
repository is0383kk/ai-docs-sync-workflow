---
read_when:
    - Reclamar una organización, marca, ámbito de paquete, identificador de propietario, slug de Skill o espacio de nombres de paquete
    - Resolver un espacio de nombres que ya está reclamado o reservado
    - Decidir si usar un informe, una apelación o una reclamación de espacio de nombres
sidebarTitle: Org and Namespace Claims
summary: Cómo solicitar una revisión de ClawHub para disputas de propiedad de organizaciones, marcas, identificadores de propietarios, ámbitos de paquetes, slugs de Skills o espacios de nombres.
title: Reivindicaciones de organización y espacio de nombres
x-i18n:
    generated_at: "2026-07-26T05:01:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 77a4d8090b55298c401154d116d93d4f8139d40983a45982288d8e48bcea40fb
    source_path: clawhub/namespace-claims.md
    workflow: 16
---

# Reclamaciones de organizaciones y espacios de nombres

ClawHub utiliza identificadores de propietarios, identificadores de organizaciones, slugs de Skills, nombres de paquetes de plugins y
ámbitos de paquetes como espacios de nombres públicos. Si un espacio de nombres parece pertenecer a un
proyecto, marca, ecosistema de paquetes u organización del mundo real, pero ya está
reclamado, reservado, resulta engañoso o está en disputa en ClawHub, solicite al personal que lo revise
mediante el
[formulario de incidencia para reclamaciones de organizaciones o espacios de nombres](https://github.com/openclaw/clawhub/issues/new?template=org-namespace-claim.yml).

Utilice esta vía para la revisión pública y no confidencial de la titularidad. No utilice los
informes del producto ni el formulario de apelación de cuentas para reclamar espacios de nombres.

## Cuándo presentar una reclamación

Presente una reclamación de espacio de nombres cuando considere que el personal de ClawHub debe revisar si un
espacio de nombres debe reservarse, transferirse, renombrarse, ocultarse, ponerse en cuarentena, recibir un alias
o modificarse de otro modo debido a su titularidad en el mundo real.

Algunos ejemplos son:

- un identificador de organización que coincide con su organización de GitHub, proyecto, empresa o comunidad
- un ámbito de paquete como `@example-org/*` que solo debería publicarse bajo el
  propietario correspondiente de ClawHub
- un slug de Skill o nombre de paquete de plugin que parece suplantar a un proyecto
- una disputa relacionada con una marca, marca comercial, cambio de nombre de un proyecto o historial de un paquete
- un propietario eliminado, inactivo o inaccesible que bloquea al titular legítimo del espacio de nombres

Si la publicación es insegura, maliciosa o engañosa más allá de la disputa de titularidad,
siga también las directrices pertinentes de moderación o seguridad. El formulario de reclamación de
espacios de nombres sirve para revisar la titularidad, no para divulgar vulnerabilidades urgentes.

## Antes de presentar la reclamación

Primero, confirme que publica con el propietario que corresponde al espacio de nombres.
En el caso de los paquetes de plugins, los nombres con ámbito como `@example-org/example-plugin` deben
publicarse con el propietario `example-org` correspondiente.

Si puede gestionar al propietario actual, corrija directamente el espacio de nombres publicando,
renombrando, transfiriendo, ocultando o eliminando el recurso afectado. Presente una reclamación
cuando no pueda gestionar al propietario actual o cuando sea necesario que el personal resuelva una
disputa.

## Pruebas que deben incluirse

Utilice pruebas públicas y no confidenciales. Entre las pruebas útiles se incluyen:

- historial de organizaciones, repositorios, versiones o responsables de mantenimiento en GitHub
- documentación oficial del proyecto que indique el espacio de nombres
- pruebas de un dominio o de un dominio de correo electrónico oficial
- control del ámbito en npm, PyPI, crates.io u otro registro de paquetes
- pruebas de titularidad de una marca comercial, marca o proyecto que puedan comentarse
  públicamente de forma segura
- historial del repositorio de origen, historial del paquete o avisos públicos de cambio de nombre
- enlaces al propietario, Skill, plugin, paquete o incidencia de ClawHub en disputa

Explique qué demuestra cada enlace. El personal debe poder comprender la
relación sin necesitar credenciales privadas ni secretos.

## Qué no debe incluirse

No publique secretos ni pruebas privadas en una incidencia pública de GitHub. No incluya:

- tokens de API, claves de firma o credenciales
- tokens de desafío de DNS
- archivos legales privados o contratos
- documentos personales de identidad
- correos electrónicos privados, informes de seguridad privados o datos confidenciales de clientes

El formulario de reclamación pregunta si las pruebas confidenciales requieren un canal privado con el personal.
Utilice esa opción en lugar de publicar material confidencial.

## Resultados posibles

Según las pruebas y el riesgo, el personal de ClawHub puede reservar un espacio de nombres,
transferir su titularidad, renombrar un recurso, ocultar o poner en cuarentena una publicación existente,
añadir un alias o una redirección, solicitar más pruebas o rechazar la solicitud.

La revisión del espacio de nombres no garantiza que se transfieran todos los nombres coincidentes.
El personal valora las pruebas públicas, el uso existente, el riesgo de seguridad y el impacto para los usuarios.

## Documentación relacionada

- [Publicación](/es/clawhub/publishing)
- [Solución de problemas](/es/clawhub/troubleshooting#publish-fails-because-a-namespace-is-claimed-or-reserved)
- [Moderación y seguridad de las cuentas](/es/clawhub/moderation)
- [Seguridad](/es/clawhub/security)
