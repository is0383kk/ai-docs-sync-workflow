---
read_when:
    - Quiere que OpenClaw se ejecute las 24 horas del día, los 7 días de la semana, en Azure, con un grupo de seguridad de red reforzado
    - Quiere un Gateway de OpenClaw de nivel de producción y siempre activo en su propia máquina virtual Linux de Azure
    - Se desea una administración segura mediante SSH de Azure Bastion
summary: Ejecuta el Gateway de OpenClaw de forma ininterrumpida en una máquina virtual Linux de Azure con estado persistente
title: Azure
x-i18n:
    generated_at: "2026-07-26T04:40:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e8598014cdc2786a47039ffb42ddd85354da9c87fd55ea46bb6dad7714171a14
    source_path: install/azure.md
    workflow: 16
---

Configura una máquina virtual Linux de Azure con la CLI de Azure, aplica medidas de refuerzo al grupo de seguridad de red (NSG), configura Azure Bastion para el acceso mediante SSH e instala OpenClaw.

## Qué se hará

- Crear recursos de red de Azure (VNet, subredes y NSG) y de proceso con la CLI de Azure
- Aplicar reglas de NSG para que solo se permita el acceso SSH a la máquina virtual desde Azure Bastion
- Usar Azure Bastion para el acceso mediante SSH (sin IP pública en la máquina virtual)
- Instalar OpenClaw con el script de instalación
- Verificar el Gateway

## Requisitos

- Una suscripción de Azure con permiso para crear recursos de proceso y de red
- La CLI de Azure instalada (consulta los [pasos de instalación de la CLI de Azure](https://learn.microsoft.com/cli/azure/install-azure-cli))
- Un par de claves SSH (esta guía explica cómo generarlo si es necesario)
- Entre 20 y 30 minutos aproximadamente

## Configurar la implementación

<Steps>
  <Step title="Iniciar sesión en la CLI de Azure">
    ```bash
    az login
    az extension add -n ssh
    ```

    La extensión `ssh` es necesaria para la tunelización SSH nativa de Azure Bastion.

  </Step>

  <Step title="Registrar los proveedores de recursos necesarios (una sola vez)">
    ```bash
    az provider register --namespace Microsoft.Compute
    az provider register --namespace Microsoft.Network
    ```

    Verifica el registro; espera hasta que ambos muestren `Registered`.

    ```bash
    az provider show --namespace Microsoft.Compute --query registrationState -o tsv
    az provider show --namespace Microsoft.Network --query registrationState -o tsv
    ```

  </Step>

  <Step title="Definir las variables de implementación">
    ```bash
    RG="rg-openclaw"
    LOCATION="westus2"
    VNET_NAME="vnet-openclaw"
    VNET_PREFIX="10.40.0.0/16"
    VM_SUBNET_NAME="snet-openclaw-vm"
    VM_SUBNET_PREFIX="10.40.2.0/24"
    BASTION_SUBNET_PREFIX="10.40.1.0/26"
    NSG_NAME="nsg-openclaw-vm"
    VM_NAME="vm-openclaw"
    ADMIN_USERNAME="openclaw"
    BASTION_NAME="bas-openclaw"
    BASTION_PIP_NAME="pip-openclaw-bastion"
    ```

    Ajusta los nombres y los intervalos CIDR a tu entorno. La subred de Bastion debe ser como mínimo `/26`.

  </Step>

  <Step title="Seleccionar una clave SSH">
    Usa tu clave pública existente si tienes una:

    ```bash
    SSH_PUB_KEY="$(cat ~/.ssh/id_ed25519.pub)"
    ```

    De lo contrario, genera una:

    ```bash
    ssh-keygen -t ed25519 -a 100 -f ~/.ssh/id_ed25519 -C "you@example.com"
    SSH_PUB_KEY="$(cat ~/.ssh/id_ed25519.pub)"
    ```

  </Step>

  <Step title="Seleccionar el tamaño de la máquina virtual y del disco del sistema operativo">
    ```bash
    VM_SIZE="Standard_B2as_v2"
    OS_DISK_SIZE_GB=64
    ```

    - Empieza con un tamaño menor para un uso ligero y amplíalo más adelante.
    - Usa más vCPU, RAM y disco para una automatización más exigente, más canales o mayores cargas de trabajo de modelos y herramientas.
    - Si un tamaño no está disponible en tu región o en la cuota de tu suscripción, elige la SKU disponible más parecida.

    Muestra los tamaños de máquina virtual disponibles en la región de destino:

    ```bash
    az vm list-skus --location "${LOCATION}" --resource-type virtualMachines -o table
    ```

    Comprueba el uso y la cuota actuales de vCPU y disco:

    ```bash
    az vm list-usage --location "${LOCATION}" -o table
    ```

  </Step>
</Steps>

## Implementar los recursos de Azure

<Steps>
  <Step title="Crear el grupo de recursos">
    ```bash
    az group create -n "${RG}" -l "${LOCATION}"
    ```
  </Step>

  <Step title="Crear el grupo de seguridad de red">
    Crea el NSG y añade reglas para que solo la subred de Bastion pueda acceder mediante SSH a la máquina virtual.

    ```bash
    az network nsg create \
      -g "${RG}" -n "${NSG_NAME}" -l "${LOCATION}"

    # Permitir SSH solo desde la subred de Bastion
    az network nsg rule create \
      -g "${RG}" --nsg-name "${NSG_NAME}" \
      -n AllowSshFromBastionSubnet --priority 100 \
      --access Allow --direction Inbound --protocol Tcp \
      --source-address-prefixes "${BASTION_SUBNET_PREFIX}" \
      --destination-port-ranges 22

    # Denegar SSH desde la Internet pública
    az network nsg rule create \
      -g "${RG}" --nsg-name "${NSG_NAME}" \
      -n DenyInternetSsh --priority 110 \
      --access Deny --direction Inbound --protocol Tcp \
      --source-address-prefixes Internet \
      --destination-port-ranges 22

    # Denegar SSH desde otros orígenes de la VNet
    az network nsg rule create \
      -g "${RG}" --nsg-name "${NSG_NAME}" \
      -n DenyVnetSsh --priority 120 \
      --access Deny --direction Inbound --protocol Tcp \
      --source-address-prefixes VirtualNetwork \
      --destination-port-ranges 22
    ```

    Las reglas se evalúan por prioridad, comenzando por el número más bajo: el tráfico de Bastion se permite con la prioridad 100 y, a continuación, se bloquea el resto del tráfico SSH con las prioridades 110 y 120.

  </Step>

  <Step title="Crear la red virtual y las subredes">
    Crea la VNet con la subred de la máquina virtual (con el NSG asociado) y, después, añade la subred de Bastion.

    ```bash
    az network vnet create \
      -g "${RG}" -n "${VNET_NAME}" -l "${LOCATION}" \
      --address-prefixes "${VNET_PREFIX}" \
      --subnet-name "${VM_SUBNET_NAME}" \
      --subnet-prefixes "${VM_SUBNET_PREFIX}"

    # Asociar el NSG a la subred de la máquina virtual
    az network vnet subnet update \
      -g "${RG}" --vnet-name "${VNET_NAME}" \
      -n "${VM_SUBNET_NAME}" --nsg "${NSG_NAME}"

    # AzureBastionSubnet: Azure exige este nombre exacto
    az network vnet subnet create \
      -g "${RG}" --vnet-name "${VNET_NAME}" \
      -n AzureBastionSubnet \
      --address-prefixes "${BASTION_SUBNET_PREFIX}"
    ```

  </Step>

  <Step title="Crear la máquina virtual">
    La máquina virtual no recibe ninguna IP pública. El acceso mediante SSH se realiza exclusivamente a través de Azure Bastion.

    ```bash
    az vm create \
      -g "${RG}" -n "${VM_NAME}" -l "${LOCATION}" \
      --image "Canonical:ubuntu-24_04-lts:server:latest" \
      --size "${VM_SIZE}" \
      --os-disk-size-gb "${OS_DISK_SIZE_GB}" \
      --storage-sku StandardSSD_LRS \
      --admin-username "${ADMIN_USERNAME}" \
      --ssh-key-values "${SSH_PUB_KEY}" \
      --vnet-name "${VNET_NAME}" \
      --subnet "${VM_SUBNET_NAME}" \
      --public-ip-address "" \
      --nsg ""
    ```

    `--public-ip-address ""` evita que se asigne una IP pública. `--nsg ""` omite el NSG por NIC, ya que el NSG de la subred ya se encarga de la seguridad.

    Para fijar una versión específica de la imagen de Ubuntu en lugar de `latest`, muestra primero las versiones disponibles:

    ```bash
    az vm image list \
      --publisher Canonical --offer ubuntu-24_04-lts \
      --sku server --all -o table
    ```

  </Step>

  <Step title="Crear Azure Bastion">
    Azure Bastion proporciona acceso SSH administrado sin exponer una IP pública en la máquina virtual. La SKU Standard con la tunelización habilitada es necesaria para `az network bastion ssh` mediante la CLI.

    ```bash
    az network public-ip create \
      -g "${RG}" -n "${BASTION_PIP_NAME}" -l "${LOCATION}" \
      --sku Standard --allocation-method Static

    az network bastion create \
      -g "${RG}" -n "${BASTION_NAME}" -l "${LOCATION}" \
      --vnet-name "${VNET_NAME}" \
      --public-ip-address "${BASTION_PIP_NAME}" \
      --sku Standard --enable-tunneling true
    ```

    El aprovisionamiento de Bastion suele tardar entre 5 y 10 minutos, pero puede tardar hasta entre 15 y 30 minutos en algunas regiones.

  </Step>
</Steps>

## Instalar OpenClaw

<Steps>
  <Step title="Acceder mediante SSH a la máquina virtual a través de Azure Bastion">
    ```bash
    VM_ID="$(az vm show -g "${RG}" -n "${VM_NAME}" --query id -o tsv)"

    az network bastion ssh \
      --name "${BASTION_NAME}" \
      --resource-group "${RG}" \
      --target-resource-id "${VM_ID}" \
      --auth-type ssh-key \
      --username "${ADMIN_USERNAME}" \
      --ssh-key ~/.ssh/id_ed25519
    ```

  </Step>

  <Step title="Instalar OpenClaw (en el shell de la máquina virtual)">
    ```bash
    curl -fsSL https://openclaw.ai/install.sh -o /tmp/install.sh
    bash /tmp/install.sh
    rm -f /tmp/install.sh
    ```

    El instalador instala Node y las dependencias si todavía no están presentes, instala OpenClaw e inicia la configuración inicial. Consulta [Instalación](/es/install) para obtener más información.

  </Step>

  <Step title="Verificar el Gateway">
    Después de completar la configuración inicial:

    ```bash
    openclaw gateway status
    ```

    Si tu organización ya dispone de licencias de GitHub Copilot, puedes elegir el proveedor GitHub Copilot durante la configuración inicial en lugar de usar una clave independiente de la API del modelo. Consulta [Proveedor GitHub Copilot](/es/providers/github-copilot).

  </Step>
</Steps>

## Consideraciones de costes

Costes mensuales aproximados (comprueba los precios actuales en la calculadora de precios de Azure, ya que las tarifas varían según la región y cambian con el tiempo):

- SKU Standard de Azure Bastion: aproximadamente 140 USD al mes
- Máquina virtual (`Standard_B2as_v2`): aproximadamente 55 USD al mes

Para reducir costes:

- Desasigna la máquina virtual cuando no esté en uso. Esto detiene la facturación de proceso (se siguen aplicando cargos por el disco). No se puede acceder al Gateway mientras la máquina virtual está desasignada.

  ```bash
  az vm deallocate -g "${RG}" -n "${VM_NAME}"
  az vm start -g "${RG}" -n "${VM_NAME}"   # reiniciarla más adelante
  ```

- Elimina Bastion cuando no sea necesario y vuelve a crearlo cuando necesites acceso SSH; es el componente de mayor coste y se aprovisiona en unos minutos.
- Usa la SKU Basic de Bastion (aproximadamente 38 USD al mes) si solo necesitas SSH a través del Portal y no necesitas tunelización mediante la CLI (`az network bastion ssh`).

## Limpieza

Elimina todos los recursos creados por esta guía:

```bash
az group delete -n "${RG}" --yes --no-wait
```

Esto elimina el grupo de recursos y todo lo que contiene (máquina virtual, VNet, NSG, Bastion e IP pública).

## Pasos siguientes

- Configurar canales de mensajería: [Canales](/es/channels)
- Emparejar dispositivos locales como nodos: [Nodos](/es/nodes)
- Configurar el Gateway: [Configuración del Gateway](/es/gateway/configuration)
- Más información sobre la implementación en Azure con el proveedor de modelos GitHub Copilot: [OpenClaw en Azure con GitHub Copilot](https://github.com/johnsonshi/openclaw-azure-github-copilot)

## Contenido relacionado

- [Descripción general de la instalación](/es/install)
- [GCP](/es/install/gcp)
- [DigitalOcean](/es/install/digitalocean)
