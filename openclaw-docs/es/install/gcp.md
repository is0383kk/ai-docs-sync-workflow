---
read_when:
    - Quieres que OpenClaw funcione 24/7 en GCP
    - Quieres un Gateway de nivel de producción y siempre activo en tu propia máquina virtual
    - Quieres tener control total sobre la persistencia, los binarios y el comportamiento de reinicio
summary: Ejecutar OpenClaw Gateway 24/7 en una VM de GCP Compute Engine (Docker) con estado persistente
title: GCP
x-i18n:
    generated_at: "2026-07-26T05:10:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6ca46b2ee78731162261cae6ea5a26b718be6035b998fa92e4ee5c9ea2e7ae07
    source_path: install/gcp.md
    workflow: 16
---

Ejecute un Gateway de OpenClaw persistente en una VM de GCP Compute Engine mediante Docker, con estado duradero, binarios integrados y un comportamiento de reinicio seguro.

El precio varía según el tipo de máquina y la región; elija la VM más pequeña que se adapte a su carga de trabajo y amplíela si se producen errores por falta de memoria.

Se puede acceder al Gateway mediante el reenvío de puertos SSH desde su portátil o mediante la exposición directa del puerto si gestiona personalmente el cortafuegos y los tokens.

Esta guía utiliza Debian en GCP Compute Engine. Ubuntu también funciona; adapte los paquetes según corresponda. Para consultar el flujo genérico de Docker, véase [Docker](/es/install/docker).

## Qué se necesita

- Cuenta de GCP (`e2-micro` cumple los requisitos del nivel gratuito)
- CLI de `gcloud` o [Cloud Console](https://console.cloud.google.com)
- Acceso SSH desde su portátil
- Docker y Docker Compose
- Credenciales de autenticación del modelo
- Credenciales opcionales de proveedores (QR de WhatsApp, token de bot de Telegram, OAuth de Gmail)
- ~20-30 minutos

## Ruta rápida

1. Cree un proyecto de GCP y habilite la facturación y la API de Compute Engine
2. Cree una VM de Compute Engine (`e2-small`, Debian 12, 20GB)
3. Conéctese por SSH a la VM e instale Docker
4. Clone el repositorio de OpenClaw
5. Cree directorios persistentes en el host
6. Configure `.env` y `docker-compose.yml`
7. Integre los binarios necesarios, compile e inicie

<Steps>
  <Step title="Instalar la CLI de gcloud (o usar Console)">
    Instálela desde [cloud.google.com/sdk/docs/install](https://cloud.google.com/sdk/docs/install) y, a continuación, ejecute:

    ```bash
    gcloud init
    gcloud auth login
    ```

    Como alternativa, realice todos los pasos siguientes mediante la interfaz web de [Cloud Console](https://console.cloud.google.com).

  </Step>

  <Step title="Crear un proyecto de GCP">
    ```bash
    gcloud projects create my-openclaw-project --name="OpenClaw Gateway"
    gcloud config set project my-openclaw-project
    gcloud services enable compute.googleapis.com
    ```

    Habilite la facturación en [console.cloud.google.com/billing](https://console.cloud.google.com/billing) (es obligatoria para Compute Engine).

    Equivalente en Console: IAM & Admin > Create Project, habilite la facturación y, a continuación, APIs & Services > Enable APIs > "Compute Engine API" > Enable.

  </Step>

  <Step title="Crear la VM">
    | Tipo      | Especificaciones          | Coste                        | Notas                                                     |
    | --------- | ------------------------- | ---------------------------- | --------------------------------------------------------- |
    | e2-medium | 2 vCPU, 4GB de RAM        | ~$25/mes                     | La opción más fiable para compilaciones locales de Docker |
    | e2-small  | 2 vCPU, 2GB de RAM        | ~$12/mes                     | Mínimo recomendado para una compilación de Docker         |
    | e2-micro  | 2 vCPU (compartidas), 1GB de RAM | Cumple los requisitos del nivel gratuito | Suele fallar por falta de memoria al compilar con Docker (salida 137) |

    ```bash
    gcloud compute instances create openclaw-gateway \
      --zone=us-central1-a \
      --machine-type=e2-small \
      --boot-disk-size=20GB \
      --image-family=debian-12 \
      --image-project=debian-cloud
    ```

  </Step>

  <Step title="Conectarse por SSH a la VM">
    ```bash
    gcloud compute ssh openclaw-gateway --zone=us-central1-a
    ```

    Console: haga clic en "SSH" junto a la VM en el panel de Compute Engine.

    La propagación de claves SSH puede tardar 1-2 minutos después de crear la VM; espere y vuelva a intentarlo si se rechaza la conexión.

  </Step>

  <Step title="Instalar Docker (en la VM)">
    ```bash
    sudo apt-get update
    sudo apt-get install -y git curl ca-certificates
    curl -fsSL https://get.docker.com | sudo sh
    sudo usermod -aG docker $USER
    ```

    Cierre la sesión y vuelva a iniciarla para que el cambio de grupo surta efecto; después, vuelva a conectarse por SSH:

    ```bash
    exit
    ```

    ```bash
    gcloud compute ssh openclaw-gateway --zone=us-central1-a
    ```

    Compruebe la instalación:

    ```bash
    docker --version
    docker compose version
    ```

  </Step>

  <Step title="Clonar el repositorio de OpenClaw">
    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    ```

    Esta guía compila una imagen personalizada para que los binarios integrados se conserven tras los reinicios.

  </Step>

  <Step title="Crear directorios persistentes en el host">
    Los contenedores de Docker son efímeros; todo el estado de larga duración debe residir en el host.

    ```bash
    mkdir -p ~/.openclaw
    mkdir -p ~/.openclaw/workspace
    ```

  </Step>

  <Step title="Configurar las variables de entorno">
    Cree `.env` en la raíz del repositorio:

    ```bash
    OPENCLAW_IMAGE=openclaw:latest
    OPENCLAW_GATEWAY_TOKEN=
    OPENCLAW_GATEWAY_BIND=lan
    OPENCLAW_GATEWAY_PORT=18789

    OPENCLAW_CONFIG_DIR=/home/$USER/.openclaw
    OPENCLAW_WORKSPACE_DIR=/home/$USER/.openclaw/workspace

    GOG_KEYRING_PASSWORD=
    XDG_CONFIG_HOME=/home/node/.openclaw
    ```

    Establezca `OPENCLAW_GATEWAY_TOKEN` para gestionar el token estable del Gateway mediante
    `.env`; de lo contrario, configure `gateway.auth.token` antes de depender de clientes
    entre reinicios. Si no se establece ninguno, OpenClaw utiliza un token exclusivo del tiempo de ejecución para
    ese inicio. Genere una contraseña para el almacén de claves de `GOG_KEYRING_PASSWORD`:

    ```bash
    openssl rand -hex 32
    ```

    **No confirme este archivo en el repositorio.** Contiene variables de entorno del contenedor y del tiempo de ejecución, como
    `OPENCLAW_GATEWAY_TOKEN`. La autenticación OAuth o mediante claves de API de los proveedores se almacena en el
    `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` montado.

  </Step>

  <Step title="Configuración de Docker Compose">
    Cree o actualice `docker-compose.yml`:

    ```yaml
    services:
      openclaw-gateway:
        image: ${OPENCLAW_IMAGE}
        build: .
        restart: unless-stopped
        env_file:
          - .env
        environment:
          - HOME=/home/node
          - NODE_ENV=production
          - TERM=xterm-256color
          - OPENCLAW_GATEWAY_BIND=${OPENCLAW_GATEWAY_BIND}
          - OPENCLAW_GATEWAY_PORT=${OPENCLAW_GATEWAY_PORT}
          - OPENCLAW_GATEWAY_TOKEN=${OPENCLAW_GATEWAY_TOKEN}
          - GOG_KEYRING_PASSWORD=${GOG_KEYRING_PASSWORD}
          - XDG_CONFIG_HOME=${XDG_CONFIG_HOME}
          - PATH=/home/linuxbrew/.linuxbrew/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
        volumes:
          - ${OPENCLAW_CONFIG_DIR}:/home/node/.openclaw
          - ${OPENCLAW_WORKSPACE_DIR}:/home/node/.openclaw/workspace
        ports:
          # Recomendado: mantenga el Gateway accesible solo mediante la interfaz de bucle invertido en la VM; acceda mediante un túnel SSH.
          # Para exponerlo públicamente, elimine el prefijo `127.0.0.1:` y configure el cortafuegos según corresponda.
          - "127.0.0.1:${OPENCLAW_GATEWAY_PORT}:18789"
        command:
          [
            "node",
            "dist/index.js",
            "gateway",
            "--bind",
            "${OPENCLAW_GATEWAY_BIND}",
            "--port",
            "${OPENCLAW_GATEWAY_PORT}",
            "--allow-unconfigured",
          ]
    ```

    `--allow-unconfigured` solo facilita el arranque inicial; no sustituye a una configuración real del Gateway. Configure igualmente la autenticación (`gateway.auth.token` o una contraseña) y un modo de enlace seguro para su despliegue.

  </Step>

  <Step title="Pasos compartidos del tiempo de ejecución de una VM con Docker">
    Siga la guía compartida del tiempo de ejecución para aplicar el flujo común de hosts Docker:

    - [Integrar los binarios necesarios en la imagen](/es/install/docker-vm-runtime#bake-required-binaries-into-the-image)
    - [Compilar e iniciar](/es/install/docker-vm-runtime#build-and-launch)
    - [Qué persiste y dónde](/es/install/docker-vm-runtime#what-persists-where)
    - [Actualizaciones](/es/install/docker-vm-runtime#updates)

  </Step>

  <Step title="Notas de inicio específicas de GCP">
    Si la compilación falla con `Killed` o `exit code 137` durante `pnpm install --frozen-lockfile`, la VM se ha quedado sin memoria. Utilice como mínimo `e2-small` o `e2-medium` para que las primeras compilaciones sean más fiables.

    Al enlazar con la LAN (`OPENCLAW_GATEWAY_BIND=lan`), configure un origen de navegador de confianza antes de continuar:

    ```bash
    docker compose run --rm openclaw-cli config set gateway.controlUi.allowedOrigins '["http://127.0.0.1:18789"]' --strict-json
    ```

    Sustituya `18789` por el puerto configurado si lo ha cambiado.

  </Step>

  <Step title="Acceder desde su portátil">
    Cree un túnel SSH para reenviar el puerto del Gateway:

    ```bash
    gcloud compute ssh openclaw-gateway --zone=us-central1-a -- -L 18789:127.0.0.1:18789
    ```

    Abra `http://127.0.0.1:18789/` en el navegador.

    Vuelva a mostrar un enlace limpio al panel:

    ```bash
    docker compose run --rm openclaw-cli dashboard --no-open
    ```

    Si la interfaz solicita autenticación mediante secreto compartido, pegue el token o la
    contraseña configurados en los ajustes de Control UI (este flujo de Docker escribe un token de forma
    predeterminada; utilice la contraseña configurada si cambió a la autenticación
    mediante contraseña).

    Si Control UI muestra `unauthorized` o `disconnected (1008): pairing required`, apruebe el dispositivo del navegador:

    ```bash
    docker compose run --rm openclaw-cli devices list
    docker compose run --rm openclaw-cli devices approve <requestId>
    ```

    Consulte [Tiempo de ejecución de una VM con Docker](/es/install/docker-vm-runtime#what-persists-where) para ver el mapa de persistencia compartido y el [flujo de actualización](/es/install/docker-vm-runtime#updates).

  </Step>
</Steps>

## Solución de problemas

**Conexión SSH rechazada**

La propagación de claves SSH puede tardar 1-2 minutos después de crear la VM. Espere y vuelva a intentarlo.

**Problemas con OS Login**

Compruebe su perfil de OS Login:

```bash
gcloud compute os-login describe-profile
```

Asegúrese de que su cuenta disponga de los permisos de IAM necesarios (Compute OS Login o Compute OS Admin Login).

**Memoria insuficiente (OOM)**

Si la compilación de Docker falla con `Killed` y `exit code 137`, el proceso de la VM se terminó por falta de memoria:

```bash
# Detenga primero la VM
gcloud compute instances stop openclaw-gateway --zone=us-central1-a

# Cambie el tipo de máquina
gcloud compute instances set-machine-type openclaw-gateway \
  --zone=us-central1-a \
  --machine-type=e2-small

# Inicie la VM
gcloud compute instances start openclaw-gateway --zone=us-central1-a
```

## Cuentas de servicio (práctica recomendada de seguridad)

Para uso personal, la cuenta de usuario predeterminada funciona correctamente. Para automatización o CI/CD, cree una cuenta de servicio específica con permisos mínimos:

```bash
gcloud iam service-accounts create openclaw-deploy \
  --display-name="OpenClaw Deployment"

gcloud projects add-iam-policy-binding my-openclaw-project \
  --member="serviceAccount:openclaw-deploy@my-openclaw-project.iam.gserviceaccount.com" \
  --role="roles/compute.instanceAdmin.v1"
```

Evite el rol Owner para la automatización; utilice el rol más restrictivo que funcione. Consulte [Descripción de los roles](https://cloud.google.com/iam/docs/understanding-roles).

## Pasos siguientes

- Configure canales de mensajería: [Canales](/es/channels)
- Vincule dispositivos locales como nodos: [Nodos](/es/nodes)
- Configure el Gateway: [Configuración del Gateway](/es/gateway/configuration)

## Contenido relacionado

- [Descripción general de la instalación](/es/install)
- [Azure](/es/install/azure)
- [Alojamiento en VPS](/es/vps)
