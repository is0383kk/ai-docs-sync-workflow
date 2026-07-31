---
read_when:
    - Comparte un agente de OpenClaw con otros operadores
    - Es necesario comprender los indicadores de propietario y presencia de la sesión.
    - Está decidiendo si un único agente compartido proporciona suficiente aislamiento
summary: Cómo funcionan la propiedad de las sesiones y la presencia cuando varias personas operan un agente
title: Modo multiusuario
x-i18n:
    generated_at: "2026-07-26T05:11:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c6a5a0e37b8dbeb2ebb7f32c3518acc6f3995dbfc09102f4d58c85e9cd62dfc2
    source_path: concepts/multi-user.md
    workflow: 16
---

El modo multiusuario permite que varias personas de confianza operen el mismo agente de OpenClaw. Añade propiedad de las sesiones, presencia en tiempo real y filtrado por creador para que un equipo pueda saber quién inició el trabajo y quién lo está observando en ese momento.

## Límite de confianza

Todas las personas que pueden operar un agente pueden hacer que este realice cualquier acción que esté a su alcance. La propiedad de las sesiones, la visibilidad en la barra lateral y los indicadores de presencia son funciones de usabilidad, no límites de seguridad.

Si las personas no deben acceder a las sesiones, herramientas, credenciales o archivos de otras, asígneles agentes distintos o límites de confianza separados de Gateway o del host. No confíe en los avatares de propietarios ni en los filtros como mecanismos de aislamiento.

## Propiedad y presencia

Las sesiones nuevas registran un `createdActor` de escritura única cuando la ruta de creación puede demostrar quién las originó. Las personas autenticadas usan su identificador de perfil persistente de Gateway; los agentes solicitantes y las rutas del sistema usan el mismo campo de actor. Las sesiones creadas sin un actor demostrado permanecen sin atribución.

Los nombres para mostrar de las personas se obtienen del perfil actual de Gateway cuando se devuelven las filas de las sesiones. OpenClaw no almacena etiquetas en las entradas de sesión, por lo que cambiar el nombre de un perfil actualiza la interfaz de propiedad sin reescribir el historial de sesiones.

La aplicación web mantiene visualmente diferenciadas la propiedad y la presencia:

- Un avatar de propietario sólido es permanente durante toda la vida de esa sesión.
- Los avatares de presencia con anillo o translúcidos muestran a las personas que están conectadas u observando en ese momento.
- El filtro de personas de la barra lateral muestra las sesiones creadas por una identidad y conserva los grupos personalizados existentes.

Cuando aparecen menos de dos creadores distintos en la lista de sesiones cargada, OpenClaw oculta todos los elementos visuales de propiedad y del filtro de personas. Por lo tanto, un Gateway de un solo usuario mantiene el mismo aspecto.

## Borradores

Inicie una sesión como borrador para mantener el trabajo en curso fuera de las barras laterales de los compañeros hasta que se publique. Los borradores nunca se ocultan a los administradores, que ven los borradores de otras personas con un indicador de fantasma atenuado. Esta es una función de coordinación, no un límite de seguridad.

## Atribución de turnos

La atribución del remitente de los turnos se realiza en la medida de lo posible. La intervención puede combinar entradas en un turno activo, por lo que la transcripción no siempre puede representar la contribución de cada persona como un turno independiente.

## Contenido relacionado

- [La sesión principal](/es/concepts/main-session)
- [Gestión de sesiones](/es/concepts/session)
- [Presencia](/es/concepts/presence)
- [Seguridad de Gateway](/es/gateway/security)
