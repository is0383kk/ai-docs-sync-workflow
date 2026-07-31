---
read_when:
    - Se desea ejecutar OpenClaw en un clúster de Kubernetes
    - Quieres probar OpenClaw en un entorno de Kubernetes
summary: Implementar OpenClaw Gateway en un clúster de Kubernetes con Kustomize
title: Kubernetes
x-i18n:
    generated_at: "2026-07-26T04:40:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c05eb0eb923fa1f515aca1f6dcb6073aba69af0bdf30233243027edfedd45a39
    source_path: install/kubernetes.md
    workflow: 16
---

Un punto de partida mínimo para ejecutar OpenClaw en Kubernetes, no un despliegue listo para producción. Abarca los recursos principales y está pensado para adaptarse al entorno correspondiente.

## Por qué no usar Helm

OpenClaw es un único contenedor con algunos archivos de configuración. La personalización relevante está en el contenido del agente (archivos Markdown, Skills, anulaciones de configuración), no en la creación de plantillas de infraestructura. Kustomize gestiona las superposiciones sin la sobrecarga de un chart de Helm. Se puede añadir un chart de Helm sobre estos manifiestos si el despliegue se vuelve más complejo.

## Qué se necesita

- Un clúster de Kubernetes en ejecución (AKS, EKS, GKE, k3s, kind, OpenShift, etc.)
- `kubectl` conectado al clúster
- Una clave de API para al menos un proveedor de modelos

## Inicio rápido

```bash
# Sustituir por el proveedor correspondiente: ANTHROPIC, GEMINI, OPENAI u OPENROUTER
export <PROVIDER>_API_KEY="..."
./scripts/k8s/deploy.sh

kubectl port-forward svc/openclaw 18789:18789 -n openclaw
open http://localhost:18789
```

`deploy.sh` crea de forma predeterminada la autenticación mediante token. Para obtener el token del gateway generado para la interfaz de control:

```bash
kubectl get secret openclaw-secrets -n openclaw -o jsonpath='{.data.OPENCLAW_GATEWAY_TOKEN}' | base64 -d
```

Para la depuración local, `./scripts/k8s/deploy.sh --show-token` muestra el token después del despliegue.

## Pruebas locales con Kind

Si no se dispone de un clúster, se puede crear uno localmente con [Kind](https://kind.sigs.k8s.io/):

```bash
./scripts/k8s/create-kind.sh           # detecta automáticamente docker o podman
./scripts/k8s/create-kind.sh --delete  # elimina el clúster
```

Después, se despliega de la forma habitual con `./scripts/k8s/deploy.sh`.

## Paso a paso

### 1) Desplegar

**Opción A: clave de API en el entorno (un paso)**

```bash
# Sustituir por el proveedor correspondiente: ANTHROPIC, GEMINI, OPENAI u OPENROUTER
export <PROVIDER>_API_KEY="..."
./scripts/k8s/deploy.sh
```

El script crea un Secret de Kubernetes con la clave de API y un token del gateway generado automáticamente y, a continuación, realiza el despliegue. Si el Secret ya existe, conserva el token actual del gateway y todas las claves de proveedores que no se modifiquen.

**Opción B: crear el secreto por separado**

```bash
export <PROVIDER>_API_KEY="..."
./scripts/k8s/deploy.sh --create-secret
./scripts/k8s/deploy.sh
```

Añadir `--show-token` a cualquiera de los comandos para mostrar el token en stdout durante las pruebas locales.

### 2) Acceder al gateway

```bash
kubectl port-forward svc/openclaw 18789:18789 -n openclaw
open http://localhost:18789
```

## Qué se despliega

```text
Namespace: openclaw (configurable mediante OPENCLAW_NAMESPACE)
├── Deployment/openclaw        # Pod único, contenedor de inicialización + gateway
├── Service/openclaw           # ClusterIP en el puerto 18789
├── PersistentVolumeClaim      # 10Gi para el estado y la configuración del agente
├── ConfigMap/openclaw-config  # openclaw.json + AGENTS.md
└── Secret/openclaw-secrets    # Token del gateway + claves de API
```

## Personalización

### Instrucciones del agente

Editar el `AGENTS.md` en `scripts/k8s/manifests/configmap.yaml` y volver a desplegar:

```bash
./scripts/k8s/deploy.sh
```

### Configuración del Gateway

Editar `openclaw.json` en `scripts/k8s/manifests/configmap.yaml`. Consultar [Configuración del Gateway](/es/gateway/configuration) para obtener la referencia completa.

### Añadir proveedores

Volver a ejecutar el proceso después de exportar las claves adicionales:

```bash
export ANTHROPIC_API_KEY="..."
export OPENAI_API_KEY="..."
./scripts/k8s/deploy.sh --create-secret
./scripts/k8s/deploy.sh
```

Las claves de proveedores existentes permanecen en el Secret, a menos que se sobrescriban.

También se puede aplicar un parche directamente al Secret:

```bash
kubectl patch secret openclaw-secrets -n openclaw \
  -p '{"stringData":{"<PROVIDER>_API_KEY":"..."}}'
kubectl rollout restart deployment/openclaw -n openclaw
```

### Espacio de nombres personalizado

```bash
OPENCLAW_NAMESPACE=my-namespace ./scripts/k8s/deploy.sh
```

### Imagen personalizada

Editar el campo `image` en `scripts/k8s/manifests/deployment.yaml`:

```yaml
image: ghcr.io/openclaw/openclaw:slim # principal; réplica oficial de Docker Hub: openclaw/openclaw
```

### Exponer más allá del reenvío de puertos

Los manifiestos predeterminados vinculan el gateway a la interfaz de bucle invertido dentro del pod. Esto funciona con `kubectl port-forward`, pero no con una ruta de `Service` de Kubernetes o de Ingress que necesite acceder directamente a la IP del pod.

Para exponer el gateway mediante un Ingress o un equilibrador de carga:

- Cambiar la vinculación del gateway en `scripts/k8s/manifests/configmap.yaml` de `loopback` a una vinculación que no sea de bucle invertido y que coincida con el modelo de despliegue.
- Mantener habilitada la autenticación del gateway y utilizar un punto de entrada adecuado con terminación TLS.
- Configurar la interfaz de control para el acceso remoto mediante el modelo de seguridad web compatible (por ejemplo, HTTPS/Tailscale Serve y orígenes permitidos explícitos cuando sea necesario).

## Volver a desplegar

```bash
./scripts/k8s/deploy.sh
```

Esto aplica todos los manifiestos y reinicia el pod para incorporar cualquier cambio en la configuración o los secretos.

## Desinstalación

```bash
./scripts/k8s/deploy.sh --delete
```

Esto elimina el espacio de nombres y todos sus recursos, incluido el PVC.

## Notas de arquitectura

- El gateway se vincula de forma predeterminada a la interfaz de bucle invertido dentro del pod, por lo que la configuración incluida está destinada a `kubectl port-forward`.
- No hay recursos con ámbito de clúster; todo reside en un único espacio de nombres.
- Refuerzo de seguridad: capacidades `readOnlyRootFilesystem`, `drop: ALL`, usuario no root (UID 1000).
- La configuración predeterminada mantiene la interfaz de control en la ruta más segura de acceso local: vinculación a la interfaz de bucle invertido más `kubectl port-forward` a `http://127.0.0.1:18789`.
- Si se amplía el acceso más allá de localhost, se debe utilizar el modelo remoto compatible: HTTPS/Tailscale junto con la vinculación adecuada del gateway y los ajustes de origen de la interfaz de control.
- Los secretos se generan en un directorio temporal y se aplican directamente al clúster; no se escribe ningún material secreto en el checkout del repositorio.

## Estructura de archivos

```text
scripts/k8s/
├── deploy.sh                   # Crea el espacio de nombres y el secreto; despliega mediante kustomize
├── create-kind.sh              # Clúster local de Kind (detecta automáticamente docker/podman)
└── manifests/
    ├── kustomization.yaml      # Base de Kustomize
    ├── configmap.yaml          # openclaw.json + AGENTS.md
    ├── deployment.yaml         # Especificación del pod con refuerzo de seguridad
    ├── pvc.yaml                # 10Gi de almacenamiento persistente
    └── service.yaml            # ClusterIP en 18789
```

## Contenido relacionado

- [Docker](/es/install/docker)
- [Entorno de ejecución de máquina virtual de Docker](/es/install/docker-vm-runtime)
- [Descripción general de la instalación](/es/install)
