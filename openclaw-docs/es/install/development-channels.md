---
read_when:
    - Quieres cambiar entre stable/extended-stable/beta/dev
    - Quieres fijar una versión, etiqueta o SHA específicos
    - Estás etiquetando o publicando versiones preliminares
sidebarTitle: Release Channels
summary: 'Canales estable, estable extendido, beta y de desarrollo: semántica, cambio, fijación y etiquetado'
title: Canales de lanzamiento
x-i18n:
    generated_at: "2026-07-26T05:12:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a99e31f5121c0ab8696e638cb10a7ce16e8f32c81e4b2bef1f703eef71191494
    source_path: install/development-channels.md
    workflow: 16
---

OpenClaw incluye cuatro canales de actualización:

- **estable**: dist-tag de npm `latest`. Recomendado para la mayoría de los usuarios.
- **estable extendido**: dist-tag de npm `extended-stable`. Un canal de paquetes completamente nuevo, correspondiente a un mes compatible anterior.
  Solo está disponible como paquete y la instalación
  se realiza únicamente en primer plano. Una selección almacenada recibe avisos de actualización de solo lectura cuando
  `update.checkOnStart` está habilitado, pero nunca los aplica automáticamente.
- **beta**: dist-tag de npm `beta`. Recurre a `latest` cuando `beta` no está disponible
  o es anterior a la versión estable actual.
- **dev**: extremo móvil de `main` (git). dist-tag de npm `dev` cuando se publica. `main`
  está destinado a la experimentación y el desarrollo activo; puede contener
  funciones incompletas o cambios incompatibles. No lo ejecute en gateways de producción.

Las compilaciones estables suelen publicarse primero en **beta**, se validan allí y después
se promocionan a **latest** sin incrementar la versión. Los responsables de mantenimiento también pueden publicarlas
directamente en `latest`. Los dist-tags son la fuente de verdad para las instalaciones mediante npm.

## Cambio de canal

```bash
openclaw update --channel stable
openclaw update --channel extended-stable
openclaw update --channel beta
openclaw update --channel dev
```

`--channel` conserva la elección en `update.channel` dentro de la configuración y controla ambas
rutas de instalación:

| Canal             | Instalaciones mediante npm/paquetes                                                                                                                                                    | Instalaciones mediante git                                                                                                                                         |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `stable`          | dist-tag `latest`                                                                                                                                                                      | etiqueta de git estable más reciente (excluye `-alpha.N`, `-beta.N`, `-rc.N`, `-dev.N`, `-next.N`, `-preview.N`, `-canary.N`, `-nightly.N` y otros sufijos de versión preliminar con nombre) |
| `extended-stable` | resuelve el selector público de npm `extended-stable`, verifica el paquete exacto seleccionado e instala esa versión exacta. Finaliza de forma segura sin recurrir a `latest`, `beta` ni `dev`. | no compatible: OpenClaw no modifica el checkout y solicita que se utilice una instalación mediante paquetes                                                        |
| `beta`            | dist-tag `beta`, recurriendo a `latest` cuando `beta` no está disponible o es anterior                                                                                                              | etiqueta de git beta más reciente; recurre a la etiqueta de git estable más reciente cuando la beta no está disponible o es anterior                              |
| `dev`             | dist-tag `dev` (poco frecuente; la mayoría de los usuarios de dev ejecutan instalaciones mediante git)                                                                                                                   | obtiene los cambios, reorganiza el checkout mediante rebase sobre la rama ascendente `main`, compila y reinstala la CLI global                         |

Para las instalaciones mediante git de `dev`, el checkout predeterminado es `~/openclaw` (o
`$OPENCLAW_HOME/openclaw` cuando `OPENCLAW_HOME` está definido); se puede sustituir con
`OPENCLAW_GIT_DIR`.

<Tip>
Para mantener stable y dev en paralelo, utilice dos checkouts independientes y dirija cada gateway al suyo.
</Tip>

## Selección puntual de una versión o etiqueta

Utilice `--tag` para seleccionar un dist-tag, una versión o una especificación de paquete concretos para una
única actualización **sin** cambiar el canal conservado:

```bash
# Instalar una versión específica
openclaw update --tag 2026.4.1-beta.1

# Instalar desde el dist-tag beta (una sola vez, no se conserva)
openclaw update --tag beta

# Cambiar al checkout móvil main de GitHub (persistente)
openclaw update --channel dev

# Instalar una especificación concreta de paquete npm
openclaw update --tag openclaw@2026.4.1-beta.1

# Instalar una vez desde main de GitHub sin conservar el canal
openclaw update --tag main
```

Notas:

- `--tag` se aplica **solo a instalaciones mediante paquetes (npm)**; las instalaciones mediante git lo ignoran.
- La etiqueta no se conserva; la siguiente ejecución de `openclaw update` utiliza el canal
  configurado.
- `--tag main` se asigna a la especificación compatible con npm `github:openclaw/openclaw#main`
  para esa única ejecución. Para una instalación móvil persistente de `main`, utilice
  `openclaw update --channel dev` (las instalaciones mediante paquetes cambian a un checkout de git)
  o vuelva a realizar la instalación con el método git del instalador:
  `curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git --version main`.
  La ruta de instalación mediante npm rechaza directamente los destinos de origen de GitHub/git y
  remite al método git.
- Protección contra versiones anteriores: si la versión de destino es anterior a la versión
  actual, OpenClaw solicita confirmación (omítala con `--yes`).
- El canal estable extendido siempre utiliza su destino de paquete exacto y verificado. No es un
  alias puntual de `--tag extended-stable`, y `--tag` no puede combinarse
  con un canal estable extendido efectivo.
- `--channel beta` difiere de `--tag beta`: el flujo del canal puede recurrir
  a stable/latest cuando beta no está disponible o es anterior, mientras que `--tag beta` siempre
  selecciona el dist-tag `beta` sin procesar para esa única ejecución.

## Simulación

Previsualice lo que haría `openclaw update` sin realizar cambios:

```bash
openclaw update --dry-run
openclaw update --channel beta --dry-run
openclaw update --tag 2026.4.1-beta.1 --dry-run
openclaw update --dry-run --json
```

La simulación muestra el canal efectivo, la versión de destino, las acciones previstas
y si sería necesaria una confirmación para instalar una versión anterior.

## Plugins y canales

Cambiar de canal con `openclaw update` también sincroniza los orígenes de los plugins:

- `dev` devuelve los plugins instalados que tienen un equivalente incluido a
  su origen incluido (checkout de git).
- `stable` y `beta` restauran los paquetes de plugins
  instalados mediante npm o ClawHub.
- `extended-stable` resuelve los plugins oficiales de npm aptos con una intención
  simple/predeterminada o `latest` a la versión exacta instalada del núcleo. No consulta
  las etiquetas `@extended-stable` de los plugins durante la ejecución.
- Los plugins instalados mediante npm se actualizan después de que finaliza la actualización del núcleo.

## Comprobación del estado actual

```bash
openclaw update status
```

Muestra el canal activo (con el origen que lo determinó: configuración, etiqueta de git,
rama de git, versión instalada o valor predeterminado), el tipo de instalación (git o paquete),
la versión actual y la disponibilidad de actualizaciones.

## Prácticas recomendadas para las etiquetas

- Etiquete las versiones en las que deben situarse los checkouts de git: `vYYYY.M.PATCH` para stable,
  `vYYYY.M.PATCH-beta.N` para beta. Los sufijos de versión preliminar con nombre, como
  `-alpha.N`, `-rc.N` y `-next.N`, no son destinos estables ni beta.
- Las etiquetas estables numéricas heredadas, como `vYYYY.M.PATCH-1` y `v1.0.1-1`, todavía
  se reconocen como etiquetas de git estables por compatibilidad.
- `vYYYY.M.PATCH.beta.N` (separado por puntos) también se reconoce por compatibilidad;
  se recomienda `-beta.N`.
- Mantenga las etiquetas inmutables: nunca mueva ni reutilice una etiqueta.
- Los dist-tags de npm siguen siendo la fuente de verdad para las instalaciones mediante npm:
  - `latest` -> estable
  - `extended-stable` -> versión de paquete correspondiente a un mes compatible anterior
  - `beta` -> compilación candidata o compilación estable publicada primero como beta
  - `dev` -> instantánea de main (opcional)

## Disponibilidad de la aplicación para macOS

Es posible que las compilaciones beta y dev **no** incluyan una versión de la aplicación para macOS. No supone ningún problema:

- La etiqueta de git y el dist-tag de npm pueden publicarse de forma independiente.
- Indique «no hay compilación para macOS en esta beta» en las notas de la versión o en el registro de cambios.

## Contenido relacionado

- [Actualización](/es/install/updating)
- [Funcionamiento interno del instalador](/es/install/installer)
