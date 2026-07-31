---
read_when:
    - Uso o explicación de los paneles de sesiones en la interfaz de control
    - Decidir qué pueden hacer los agentes en un tablero y qué requiere una concesión del operador
summary: 'Paneles de sesión: widgets, tableros y pestañas creados por agentes, y el chat acoplado'
title: Paneles de sesiones
x-i18n:
    generated_at: "2026-07-26T04:57:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3babbc859e261aa959740ea778b44fdc1a07bce8ce7628cbabcfbc5fa207a0ce
    source_path: web/dashboards.md
    workflow: 16
---

Cada hilo de la interfaz de control tiene dos caras: la conversación habitual y un
**panel** —una cuadrícula de widgets en vivo que el agente crea—. Un hilo sin
widgets es simplemente un chat. En cuanto se fija un widget, aparece un selector
**Chat | Panel** en el encabezado y el panel se convierte en la superficie principal,
con el chat acoplado a su lado.

No hay nada que preparar ni ninguna aplicación independiente que configurar: los paneles son una
función principal, pertenecen al hilo, se almacenan con el agente y se conservan tras
`/new` y `/reset` (el contexto de la conversación se borra; el panel permanece).

## Crear un panel mediante una solicitud

Pida al agente lo que quiera ver:

> Crea un widget llamado revenue-graph: un gráfico de barras interactivo de los ingresos
> mensuales. Añade los botones "Barras" y "Tendencia" para cambiar de vista. Fíjalo en mi
> panel.

El agente primero representa el widget en línea en el chat, para que pueda verlo
antes de colocarlo en otro lugar. A partir de ahí:

- **Puede fijarlo**: pase el cursor sobre un widget en línea y elija **Fijar en el panel**.
- **O el agente puede fijarlo** directamente cuando se lo pida y actualizarlo más adelante por
  su nombre; los widgets tienen nombres estables, por lo que «actualiza revenue-graph con las
  cifras de junio» sustituye el contenido en el mismo lugar mientras el panel permanece intacto.

Los widgets son pequeñas aplicaciones independientes (HTML/JS/SVG en un entorno aislado estricto). Los botones
y selectores de vista dentro de un widget funcionan de inmediato; para cambiar la vista de un gráfico
nunca hace falta el agente.

## El panel

- **Cuadrícula fluida.** Arrastre los widgets por su controlador; todo se redistribuye y
  se compacta automáticamente. Cambie el tamaño mediante el controlador o elija un tamaño predefinido (pequeño,
  mediano, grande, extragrande) en el menú del widget. Nadie coloca píxeles:
  ni usted ni el agente.
- **Pestañas.** Un panel puede tener varias páginas; por ejemplo, una pestaña de resumen y otra
  específica con un único widget grande. Cada pestaña recuerda su propia posición del chat
  acoplado.
- **Chat acoplado.** En la vista del panel, la conversación se acopla a la
  izquierda, a la derecha o en la parte inferior, cambia de tamaño como la barra lateral y puede ocultarse
  por completo; el agente seguirá escuchándole cuando vuelva a mostrarla.
- **Paridad con el agente.** Todo lo que usted puede hacer también puede hacerlo el agente con su
  herramienta `dashboard`: añadir, actualizar, mover, cambiar el tamaño y eliminar widgets, gestionar
  pestañas, cambiar la pestaña visible y mover u ocultar el chat acoplado. Pida «coloca el
  chat a la izquierda y muestra la pestaña de finanzas» y observe cómo sucede.

## Qué pueden hacer los widgets

Un widget que solo representa contenido no necesita aprobación: aparece al instante, exactamente
como los widgets en línea del chat, y su acceso a la red está completamente deshabilitado.

Los widgets que quieran **acceso** deben declararlo, y puede concedérselo una vez por widget
con un solo toque:

- **Red** (`net`): obtiene datos directamente desde los orígenes HTTPS declarados a través del entorno aislado;
  por ejemplo, una tarjeta meteorológica que se actualiza mediante una API.
- **Datos del Gateway** (`data`): fuentes de solo lectura, como sesiones, uso o estado de Cron,
  resueltas por el Gateway; el widget nunca conserva su token.
- **Automatización** (`actions`): activa una tarea de Cron específica, de modo que un botón pueda ejecutar
  una tarea real (que puede usar un modelo más pequeño) sin reactivar la conversación
  principal.
- **Solicitud** (`prompt`): envía mensajes a su hilo sin la confirmación
  por cada clic que requieren los widgets no aprobados.

Los plugins habilitados pueden añadir sus propias fuentes de solo lectura y acciones con nombre a estas listas de capacidades; al deshabilitar el plugin, se eliminan esas integraciones.

Los permisos concedidos están vinculados a los bytes exactos y a la revisión del widget que se hayan revisado. Si el
agente cambia el widget y solicita _más_ de lo aprobado, vuelve al estado
pendiente; la actualización del contenido con los mismos permisos conserva la concesión.
Las interacciones con el widget que el agente deba conocer (los filtros seleccionados y las vistas
cambiadas) le llegan discretamente como avisos de sesión; se mantiene informado sin
ser interrumpido.

## Aplicaciones MCP en el panel

Si el Gateway tiene servidores MCP configurados, las aplicaciones MCP interactivas que aparecen
en el chat pueden fijarse como cualquier widget. Las aplicaciones fijadas vuelven a activarse en el
panel con sesiones nuevas; de forma predeterminada, son solo de visualización, y conceder al
widget las herramientas del servidor que haya declarado las hace completamente interactivas, con la misma
aprobación mediante un solo toque y vinculada a la revisión que se aplica a todo lo demás.

## Conviene saberlo

- Al restablecer un hilo que tiene un panel, se solicita confirmación y se conserva el
  panel.
- Al eliminar un hilo, se elimina su panel.
- Los paneles residen en el Gateway (en la base de datos del agente propietario) y aparecen en
  todos los dispositivos desde los que se conecte.
- El modelo de seguridad, los detalles de almacenamiento y la justificación del diseño se encuentran en
  [Arquitectura del panel](/es/web/dashboard-architecture), incluidas las
  concesiones documentadas del entorno aislado.
