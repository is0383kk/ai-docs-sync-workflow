---
read_when:
    - Está validando la optimización del rendimiento y del tamaño de los paquetes de mayo de 2026
    - Necesita las cifras que respaldan la publicación del blog sobre el rendimiento y las dependencias de OpenClaw
    - Está cambiando los controles de publicación, el archivo shrinkwrap del paquete o los límites de dependencias de los plugins
summary: Resumen visual y evidencia técnica de la limpieza de rendimiento, tamaño de los paquetes, dependencias y shrinkwrap de mayo de 2026
title: Barrido de rendimiento de la versión
x-i18n:
    generated_at: "2026-07-26T04:50:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9e98ffc9d63e14e078a19368917eb4278695e1426048dc21942f928af145d5e1
    source_path: reference/release-performance-sweep.md
    workflow: 16
---

Esta página recopila las pruebas que respaldan la limpieza de rendimiento,
tamaño de paquetes, dependencias y shrinkwrap de OpenClaw de mayo de 2026. Es el complemento técnico
de la publicación pública del blog.

Aquí se combinan dos auditorías:

- **Revisión del rendimiento de las versiones:** versiones de GitHub desde `v2026.5.28` hasta
  la versión estable `v2026.4.23`, mediante el flujo de trabajo `OpenClaw Performance`,
  `profile=smoke`, en la vía del proveedor simulado. La mayoría de las filas de etiquetas corresponden a una muestra; las
  filas `v2026.5.27` y `v2026.5.28` usan los artefactos más recientes de la rama de versión
  con 3 repeticiones.
- **Contexto anterior de abril:** líneas base publicadas del proveedor simulado
  `clawgrit-reports`, desde `v2026.4.1` hasta `v2026.5.2`, utilizadas únicamente para evitar considerar
  las versiones defectuosas de finales de abril como la línea base pública de rendimiento.
- **Revisión de la huella de instalación:** instalaciones nuevas de `npm install --ignore-scripts`
  en paquetes temporales, con `du -sk node_modules` para el tamaño y un
  recorrido de `node_modules` para contar las instancias de paquetes.
- **Revisión del tamaño del paquete npm:** `npm pack openclaw@<version> --dry-run --json`
  para las versiones publicadas, registrando el tamaño del tarball comprimido, el tamaño descomprimido y
  el número de archivos.

<Warning>
La revisión principal del rendimiento usa una muestra de prueba rápida por etiqueta, excepto las
filas `v2026.5.27` y `v2026.5.28`, que usan los artefactos más recientes de la rama de versión
con 3 repeticiones. El contexto anterior de abril usa medianas publicadas de 3 repeticiones
de `clawgrit-reports`. Los números deben considerarse pruebas de tendencias y
señales para detectar regresiones, no estadísticas de los criterios de aprobación de versiones.
</Warning>

## Resumen

Cobertura de rendimiento: **77 versiones solicitadas**, **74 puntos respaldados por artefactos**
y **3 ejecuciones de CI no disponibles**. Último punto estable medido: `v2026.5.28`.

<CardGroup cols={2}>
  <Card title="Turno estable del agente" icon="gauge">
    **Turno en frío 5.1 veces más rápido**

    - `v2026.4.14`: 9.8s
    - `v2026.5.28`: 1.9s

  </Card>
  <Card title="Paquete publicado" icon="package">
    **Tarball de 17.9MB**

    Paquete estable más reciente, por debajo del máximo de tamaño de paquete de marzo de 43.3MB.

  </Card>
  <Card title="Instalación estable más reciente" icon="hard-drive">
    **Instalación nueva de 361.7MiB**

    Reduce considerablemente el árbol anidado de dependencias de OpenClaw desde el máximo de
    introducción de shrinkwrap de `2026.5.22`, aunque en la auditoría de instalación local
    todavía queda un árbol anidado más pequeño de 259.7MiB.

  </Card>
  <Card title="Grafo de dependencias" icon="boxes">
    **300 paquetes instalados**

    Medidos como raíces únicas de nombre y versión de paquete en una instalación nueva con
    los scripts desactivados; 71 raíces menos que en la versión estable anterior.

  </Card>
</CardGroup>

## Qué cambió en 5.28

La limpieza entre `v2026.5.27` y `v2026.5.28` redujo el grafo de instalación
predeterminado en lugar de eliminar las propias capacidades.

<CardGroup cols={2}>
  <Card title="Grafo raíz predeterminado" icon="git-branch">
    Las raíces únicas de nombre y versión de paquete bajaron de **371** a **300**. Las instancias
    de paquetes bajaron de **372** a **301**.
  </Card>
  <Card title="Árbol anidado" icon="unplug">
    El `openclaw/node_modules` anidado bajó de **656.1MiB** a **259.7MiB** en
    la misma auditoría de instalación local.
  </Card>
  <Card title="Conos nativos opcionales" icon="cpu">
    El cono de paquetes nativos multiplataforma de `@napi-rs/canvas` dejó de incluirse en
    la instalación predeterminada.
  </Card>
  <Card title="Superficie de la cadena de suministro" icon="shield">
    Menos paquetes predeterminados implican menos tarballs, mantenedores, binarios nativos,
    comportamientos durante la instalación y rutas de actualización transitivas en las que confiar de forma predeterminada.
  </Card>
</CardGroup>

<Tip>
Shrinkwrap no era el problema por sí solo. El problema era la estructura deficiente del paquete.
`v2026.5.28` sigue distribuyendo shrinkwrap, pero el árbol anidado de dependencias es mucho
más pequeño y la ramificación multiplataforma de canvas ha desaparecido en la auditoría local.
</Tip>

## Cifras principales

No se deben usar las filas defectuosas de finales de abril como líneas base públicas de rendimiento.
`v2026.4.23` y `v2026.4.29` son pruebas útiles de regresiones, pero las grandes
diferencias del tipo `14x` describen principalmente la recuperación de una línea de versiones deficiente.

Para la narrativa del blog, debe usarse la línea base publicada de principios de abril como referencia de escala.
La línea base es `v2026.4.14` de la ejecución publicada del proveedor simulado
`clawgrit-reports` (3 repeticiones; esa ejecución solo falló porque no se emitió
la cronología de diagnóstico, por lo que las medianas en frío, en caliente y de RSS siguen siendo útiles
como escala aproximada). Esto debe considerarse contexto narrativo, no una estadística
de los criterios de aprobación de versiones.

| Métrica          | Línea base de principios de abril | `v2026.5.28` |                    Diferencia |
| --------------- | ---------------------: | -----------: | -----------------------: |
| Turno en frío del agente |                9,819ms |      1,908ms | 80.6% menos, 5.1 veces más rápido |
| Turno en caliente del agente |                7,458ms |      1,870ms | 74.9% menos, 4.0 veces más rápido |
| RSS máximo del agente  |                686.2MB |      581.0MB |              15.3% menos |

Dentro de la revisión de mayo, la fila más reciente de la rama de versión cambió sustancialmente respecto de
`v2026.5.2`:

| Métrica          | `v2026.5.2` | `v2026.5.28` |       Diferencia |
| --------------- | ----------: | -----------: | ----------: |
| Turno en frío del agente |     3,897ms |      1,908ms | 51.0% menos |
| Turno en caliente del agente |     3,610ms |      1,870ms | 48.2% menos |
| RSS máximo del agente  |     613.7MB |      581.0MB |  5.3% menos |

En comparación con la versión estable anterior:

| Métrica          | `v2026.5.27` | `v2026.5.28` |       Diferencia |
| --------------- | -----------: | -----------: | ----------: |
| Turno en frío del agente |      2,231ms |      1,908ms | 14.5% menos |
| Turno en caliente del agente |      2,226ms |      1,870ms | 16.0% menos |
| RSS máximo del agente  |      649.0MB |      581.0MB | 10.5% menos |

### Huella de instalación

| Métrica                                          |  Línea base | `v2026.5.28` |       Diferencia |
| ----------------------------------------------- | --------: | -----------: | ----------: |
| Tamaño de instalación desde el máximo de `2026.5.22`              | 1,020.6MB |     361.7MiB | 64.6% menos |
| Tamaño de instalación desde la versión más reciente `2026.5.27`    |  767.1MiB |     361.7MiB | 52.8% menos |
| Dependencias desde el máximo mensual `2026.2.26`      |       645 |          300 | 53.5% menos |
| Dependencias desde la versión más reciente `2026.5.27`    |       371 |          300 | 19.1% menos |
| `openclaw/node_modules` anidado desde `2026.5.22` |   911.8MB |     259.7MiB | 71.5% menos |
| `openclaw/node_modules` anidado desde `2026.5.27` |  656.1MiB |     259.7MiB | 60.4% menos |

### Tamaño del paquete npm

| Versión     | Tarball comprimido | Paquete descomprimido |  Archivos | Notas                             |
| ----------- | -----------------: | ---------------: | -----: | --------------------------------- |
| `2026.1.30` |             12.8MB |           33.5MB |  4,607 | paquete inicial con cambio de marca           |
| `2026.2.26` |             23.6MB |           82.9MB | 10,125 | crecimiento de funcionalidades                    |
| `2026.3.31` |             43.3MB |          182.6MB | 21,037 | máximo de tamaño del paquete           |
| `2026.4.29` |             22.9MB |           74.6MB |  9,309 | reducción visible del paquete           |
| `2026.5.12` |             23.4MB |           80.1MB | 12,035 | separación importante de plugins externos       |
| `2026.5.22` |             17.2MB |           76.9MB | 12,386 | documentación y recursos excluidos del paquete |
| `2026.5.27` |             17.8MB |           79.0MB | 12,509 | paquete estable anterior           |
| `2026.5.28` |             17.9MB |           81.0MB |  9,082 | paquete estable más reciente             |

`2026.5.12` es el hito visible de extracción de plugins en el registro de cambios:
Amazon Bedrock, Bedrock Mantle, Slack, el entorno aislado OpenShell, Anthropic Vertex,
Matrix y WhatsApp salieron de la ruta de dependencias del núcleo, de modo que sus conos de
dependencias se instalan con esos plugins en lugar de con cada instalación del núcleo.

## Resumen de turnos del agente Kova

La línea estable de abril contiene dos historias diferentes. A principios de abril era lenta,
pero reconocible. A finales de abril se convirtió en un precipicio de regresiones. `v2026.5.2` es el punto en el que
la vía del proveedor simulado entra por primera vez en el intervalo de 3-5s y empieza a aprobar
de forma consistente en la revisión proporcionada.

Contexto publicado anteriormente:

| Versión      | Kova | Turno en frío | Turno en caliente | RSS máximo del agente |
| ------------ | ---- | --------: | --------: | -------------: |
| `v2026.4.10` | FALLA |  11,031ms |   7,962ms |        679.0MB |
| `v2026.4.12` | FALLA |  11,965ms |   8,289ms |        713.5MB |
| `v2026.4.14` | FALLA |   9,819ms |   7,458ms |        686.2MB |
| `v2026.4.20` | FALLA |  22,314ms |  18,811ms |        810.8MB |
| `v2026.4.22` | FALLA |   9,630ms |   7,459ms |        743.0MB |

Revisión proporcionada:

| Versión             | Kova | Turno en frío | Turno en caliente | RSS máximo del agente |
| ------------------- | ---- | --------: | --------: | -------------: |
| `v2026.4.23`        | FALLA |  47,847ms |   8,010ms |      1,082.7MB |
| `v2026.4.24`        | FALLA |  48,264ms |  25,483ms |        996.0MB |
| `v2026.4.25`        | FALLA |  81,080ms |  59,172ms |      1,113.9MB |
| `v2026.4.26`        | FALLA |  76,771ms |  54,941ms |      1,140.8MB |
| `v2026.4.27`        | FALLA |  60,902ms |  33,699ms |      1,156.0MB |
| `v2026.4.29`        | FALLA |  94,031ms |  57,334ms |      3,613.7MB |
| `v2026.5.2`         | APRUEBA |   3,897ms |   3,610ms |        613.7MB |
| `v2026.5.7`         | APRUEBA |   3,923ms |   3,693ms |        654.1MB |
| `v2026.5.12`        | APRUEBA |   7,248ms |   6,629ms |        834.8MB |
| `v2026.5.18`        | APRUEBA |   3,301ms |   2,913ms |        630.3MB |
| `v2026.5.20`        | APRUEBA |   3,413ms |   2,952ms |        643.2MB |
| `v2026.5.22`        | APRUEBA |   4,494ms |   4,093ms |        654.3MB |
| `v2026.5.26`        | APRUEBA |   2,626ms |   2,282ms |        660.4MB |
| `v2026.5.27-beta.1` | APRUEBA |   2,575ms |   2,217ms |        635.3MB |
| `v2026.5.27`        | APRUEBA |   2,231ms |   2,226ms |        649.0MB |
| `v2026.5.28`        | APRUEBA |   1,908ms |   1,870ms |        581.0MB |

## Sondeos de código fuente

Los sondeos de código fuente se omitieron para 17 referencias antiguas correctas porque esos árboles
de código fuente todavía no tenían los puntos de entrada necesarios para los sondeos. Las métricas de turnos del agente
siguen existiendo para esas referencias.

Puntos representativos de los sondeos de código fuente:

| Versión             | p50 predeterminado de `readyz` | p50 de `readyz` con 50 plugins | p50 de estado de la CLI | RSS máximo de plugins |
| ------------------- | -------------------: | ----------------------: | -------------: | -------------: |
| `v2026.4.29`        |              2,819ms |                 2,618ms |        1,679ms |        389.0MB |
| `v2026.5.2`         |              2,324ms |                 2,013ms |        1,384ms |        377.2MB |
| `v2026.5.7`         |              1,649ms |                 1,540ms |        1,175ms |        387.6MB |
| `v2026.5.18`        |              1,942ms |                 1,927ms |          607ms |        426.5MB |
| `v2026.5.20`        |              1,966ms |                 1,987ms |          621ms |        455.0MB |
| `v2026.5.22`        |              2,081ms |                 1,884ms |        5,095ms |        444.2MB |
| `v2026.5.26`        |              1,546ms |                 1,634ms |          656ms |        400.4MB |
| `v2026.5.27-beta.1` |              1,462ms |                 1,548ms |          548ms |        394.0MB |
| `v2026.5.27`        |              1,491ms |                 1,571ms |          553ms |        401.5MB |
| `v2026.5.28`        |              1,457ms |                 1,474ms |          623ms |        386.1MB |

El pico de estado de la CLI de `v2026.5.22` es visible en esta tabla aunque la
vía de turnos del agente siguiera aprobando. Deben conservarse los sondeos de código fuente al investigar
regresiones específicas de la CLI o del Gateway.

## Auditoría de la huella de instalación

Las muestras de dependencias usan una versión estable por mes, además del
evento de introducción de shrinkwrap de `2026.5.22` y la versión más reciente de `2026.5.28`.

| Punto              | Dependencias instaladas | Instalación limpia | Paquete de OpenClaw | `openclaw/node_modules` anidado | Shrinkwrap raíz | Comportamiento de instalación de Canvas                   |
| ------------------ | -------------: | ------------: | ---------------: | -----------------------------: | --------------- | ----------------------------------------- |
| Ene `2026.1.30`    |            605 |       438.4MB |           45.8MB |                          2.4MB | no              | envoltorio de nivel superior + `darwin-arm64`        |
| Feb `2026.2.26`    |            645 |       575.7MB |          110.1MB |                          3.5MB | no              | envoltorio de nivel superior + `darwin-arm64`        |
| Mar `2026.3.31`    |            438 |       584.1MB |          234.8MB |                            0MB | no              | envoltorio de nivel superior + `darwin-arm64`        |
| Abr `2026.4.29`    |            392 |       335.0MB |           97.4MB |                            0MB | no              | ninguno instalado                            |
| `2026.5.22`        |            401 |     1,020.6MB |        1,020.4MB |                        911.8MB | sí             | anidados: los 12 paquetes de `@napi-rs/canvas` |
| May `2026.5.26`    |            371 |       767.5MB |          767.4MB |                        656.4MB | sí             | anidados: los 12 paquetes de `@napi-rs/canvas` |
| `2026.5.27`        |            371 |      767.1MiB |         766.9MiB |                       656.1MiB | sí             | anidados: los 12 paquetes de `@napi-rs/canvas` |
| Más reciente `2026.5.28` |            300 |      361.7MiB |         361.6MiB |                       259.7MiB | sí             | ninguno instalado                            |

### Límite de shrinkwrap

`2026.5.20` se publicó sin shrinkwrap raíz y sin un gran árbol anidado de
dependencias de OpenClaw. `2026.5.22` introdujo el shrinkwrap raíz e instaló 911.8MB
bajo el `openclaw/node_modules` anidado. `2026.5.28` mantiene el shrinkwrap y todavía
instala 259.7MiB bajo el `openclaw/node_modules` anidado, pero ya no instala
ningún paquete de `@napi-rs/canvas` en la auditoría local de instalación limpia.

La inspección del tarball publicado verifica el límite:

| Versión     | ¿Versión estable publicada? | `npm-shrinkwrap.json` raíz | Notas                                 |
| ----------- | ----------------- | -------------------------- | ------------------------------------- |
| `2026.5.20` | sí               | no                         | última versión estable antes del shrinkwrap |
| `2026.5.21` | no                | n/d                        | no hay versión estable en npm                 |
| `2026.5.22` | sí               | sí                        | se introdujo el shrinkwrap                 |
| `2026.5.23` | no                | n/d                        | no hay versión estable en npm                 |
| `2026.5.24` | no                | n/d                        | no hay versión estable en npm                 |
| `2026.5.25` | no                | n/d                        | no hay versión estable en npm                 |
| `2026.5.26` | sí               | sí                        | el árbol de dependencias anidado sigue presente  |
| `2026.5.27` | sí               | sí                        | el árbol de dependencias anidado sigue presente  |
| `2026.5.28` | sí               | sí                        | el árbol de dependencias anidado es mucho más pequeño   |

La distinción importante: **el shrinkwrap en sí no es el problema**.
`v2026.5.28` todavía incluye el shrinkwrap raíz. El problema era la estructura del paquete
que hacía que npm materializara un gran árbol anidado de dependencias de OpenClaw y los 12
paquetes de plataforma de `@napi-rs/canvas`. El árbol anidado es más pequeño en `v2026.5.28`,
y la distribución entre plataformas de Canvas ya no aparece en la auditoría local.

Para obtener una explicación sencilla de shrinkwrap y de las comprobaciones de paquetes
destinadas a responsables de mantenimiento, consulte [shrinkwrap de npm](/es/gateway/security/shrinkwrap).

## Interpretación de la cadena de suministro

El recuento de dependencias es una métrica de seguridad operativa, no solo una
métrica del tamaño de instalación. Cada paquete amplía el conjunto de responsables de mantenimiento,
tarballs, actualizaciones transitivas, binarios nativos opcionales y comportamientos durante la instalación
en los que los operadores deben confiar.

La dirección de la depuración es:

- mantener las capacidades pesadas y opcionales fuera de la instalación predeterminada del núcleo
- hacer que los paquetes de plugins sean propietarios de su grafo de dependencias de ejecución
- evitar la reparación mediante el gestor de paquetes durante el inicio del Gateway
- preservar las instalaciones deterministas sin provocar la materialización de paquetes
  nativos para todas las plataformas
- mantener deshabilitados los scripts de instalación en las rutas de aceptación y medición de paquetes
- detectar árboles de dependencias anidados y expansiones descontroladas de dependencias nativas opcionales antes
  de publicar

Documentación relacionada:

- [Resolución de dependencias de plugins](/es/plugins/dependency-resolution)
- [Inventario de plugins](/es/plugins/plugin-inventory)
- [Validación completa de versiones](/es/reference/full-release-validation)
